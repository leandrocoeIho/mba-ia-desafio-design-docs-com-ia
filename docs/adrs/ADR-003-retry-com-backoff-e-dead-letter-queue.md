# ADR-003: Retry com backoff exponencial e Dead Letter Queue

## Status

Aceito

## Contexto

Clientes webhook podem estar temporariamente indisponíveis quando o worker tenta entregar um evento. É preciso definir quantas vezes tentar reentregar, com qual espaçamento entre tentativas, e o que fazer quando as tentativas se esgotam ([09:14] Larissa: "Vamos pra retry. Se o cliente tá offline, o que a gente faz?").

Mais adiante na reunião, Sofia perguntou pelo timeout do HTTP call do worker; Diego definiu 10 segundos, tratando cliente lento como falha e alimentando o mesmo fluxo de retry abaixo ([09:42] Sofia/Diego).

## Decisão

Adotar **backoff exponencial com 5 tentativas**, na progressão **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** entre tentativas sucessivas, totalizando uma janela de aproximadamente 15 horas entre a primeira falha e a última tentativa ([09:15]-[09:17] Diego/Larissa). Passado o limite de tentativas, o evento é movido para uma tabela dedicada de **Dead Letter Queue**, `webhook_dead_letter`, contendo o payload, o motivo da falha e o timestamp ([09:18] Diego).

> **Nota de ambiguidade preservada da fonte:** a transcrição não deixa explícito se as "5 tentativas" contam a partir da primeira falha (1 envio inicial + 5 retries = 6 chamadas HTTP no total) ou se o envio inicial já é a "tentativa 1" (5 chamadas HTTP no total). Larissa resume como "total 5 tentativas" ([09:48]), mas os 5 intervalos de backoff listados por Diego ([09:17]) descrevem 5 retentativas após uma primeira falha — o que aponta para a leitura de 6 chamadas no total. Esta ADR assume essa leitura (envio inicial + 5 retries = 6 chamadas); a definição exata deve ser confirmada na especificação técnica ([FDD](../FDD.md)).

Complementarmente, o timeout de resposta do worker ao chamar o endpoint do cliente foi fixado em 10 segundos: passada essa janela sem resposta, a chamada é tratada como falha e entra neste mesmo fluxo de retry ([09:42] Sofia pergunta; Diego: "10 segundos. Cliente lento que não responde em 10s a gente trata como falha e marca pra retry").

A reentrega de um evento em DLQ é **manual**, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento como pendente na outbox ([09:18] Diego). Esse endpoint exige role `ADMIN` e deve registrar em auditoria quem executou o replay ([09:35]-[09:36] Sofia/Larissa) — ver [FDD](../FDD.md) para o contrato completo do endpoint.

## Alternativas Consideradas

1. **3 tentativas.** Descartada por ser agressiva demais: "se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada" ([09:16] Diego).
2. **Retry indefinido.** Descartada: geraria eventos pendurados indefinidamente para clientes que nunca mais voltam a responder, sem sinalização clara de falha permanente ([09:15] Diego).
3. **Marcar falha permanente como campo de status na própria tabela `webhook_outbox`, sem tabela separada.** Descartada em favor de uma tabela dedicada: "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento" ([09:18] Diego).

## Consequências

**Positivas:**
- Cobre janelas de indisponibilidade realistas observadas em clientes anteriores (até algumas horas) sem abandonar o evento prematuramente.
- Falhas permanentes ficam isoladas em uma tabela própria, facilitando auditoria, debug e reprocessamento sem poluir a leitura da outbox principal pelo worker.
- Dá ao time de operação um mecanismo explícito (endpoint admin) para intervir manualmente quando um cliente volta a ficar disponível após esgotar as tentativas automáticas.

**Negativas:**
- Um evento pode levar até ~15 horas para ser definitivamente classificado como falha, deixando o cliente sem aquela notificação específica de status por um período longo — a reunião não discutiu uma mitigação formal para essa janela.
- Reentrega de eventos em DLQ depende de ação manual de um administrador — não há retry automático após o esgotamento das 5 tentativas.
- A tabela de DLQ é mais uma estrutura a ser monitorada operacionalmente (volume de eventos falhos, alertas de crescimento).
