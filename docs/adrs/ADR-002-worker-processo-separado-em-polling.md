# ADR-002: Worker de webhooks como processo separado, em polling de 2 segundos

## Status

Aceito

## Contexto

Com o padrão outbox definido ([ADR-001](ADR-001-outbox-pattern-no-mysql.md)), é preciso decidir como e onde os eventos pendentes em `webhook_outbox` são lidos e transformados em chamadas HTTP para os clientes.

Rodar essa leitura dentro do mesmo processo da API foi descartado: "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker" ([09:11] Diego).

Também foi avaliado usar um mecanismo mais reativo (trigger de banco) em vez de polling: "Não dá pra usar trigger do banco pra ser mais reativo?" ([09:09] Bruno). Diego esclareceu que o MySQL não oferece um equivalente ao `NOTIFY/LISTEN` do Postgres, e que fazer o trigger avisar um processo externo via arquivo ou endpoint HTTP seria uma solução improvisada: "a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito" ([09:09] Diego).

## Decisão

Implementar o worker como um **processo Node separado**, com entry-point próprio `src/worker.ts` (novo arquivo, a ser criado) e script `npm run worker` (seguindo o mesmo padrão de `src/server.ts` já existente, [09:11] Larissa). O worker conecta-se ao mesmo banco (mesma `DATABASE_URL`), mas instancia seu **próprio `PrismaClient`**, já que essa classe é vinculada ao processo: Diego perguntou "o worker abre o mesmo PrismaClient ou um separado?" e Bruno respondeu "Separado. PrismaClient é por processo... instância nova porque é outro processo Node" ([09:29]-[09:30] Diego/Bruno). Essa decisão final substitui uma suposição inicial diferente feita mais cedo na reunião ([09:11] Bruno), quando ainda não havia se discutido a natureza por-processo do `PrismaClient`.

O worker opera em **loop de polling a cada 2 segundos**, buscando os eventos pendentes mais antigos em lote pequeno, processando-os e marcando-os como entregues ([09:09] Diego). Esse intervalo atende com folga o requisito de latência "abaixo de 10 segundos" acordado com os clientes ([09:02] Marcos, [09:10] Marcos/Larissa).

Fica registrado como **limitação conhecida**: a garantia de ordem de entrega dos eventos existe apenas por `order_id` e apenas enquanto o worker for único (single-worker); não há garantia de ordering global entre pedidos diferentes, e essa garantia se perde caso o sistema evolua para múltiplos workers em paralelo ([09:12] Diego explica; [09:13] Larissa: "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker"). Os clientes não pediram essa garantia global — "eles só querem saber se cada pedido deles mudou" ([09:14] Marcos). Escalar para múltiplos workers (via particionamento por `order_id` ou lock pessimista) fica fora do escopo desta decisão — "problema do futuro" ([09:13] Diego).

## Alternativas Consideradas

1. **Worker embutido no processo da API.** Descartada: reiniciar a API (deploy, crash, restart) derrubaria o worker junto, sem isolamento de falhas ([09:11] Diego).
2. **Notificação reativa via trigger de banco.** Descartada: MySQL não tem um mecanismo nativo de notificação de processo externo (ao contrário do `LISTEN/NOTIFY` do Postgres); a alternativa de o trigger escrever em arquivo ou chamar um endpoint HTTP foi considerada improvisada e desnecessária, já que o polling de 2s já atende ao SLA de 10s ([09:09] Diego).

## Consequências

**Positivas:**
- Isolamento de falhas: reiniciar a API não afeta o processamento de eventos pendentes, e vice-versa.
- Implementação simples, sem infraestrutura de mensageria adicional — reaproveita Node, Prisma e MySQL já usados no projeto.
- Latência previsível e com folga confortável em relação ao SLA de 10 segundos.

**Negativas:**
- Introduz um novo processo a ser operado e implantado separadamente da API, exigindo atenção operacional própria (a reunião não detalhou healthcheck ou política de restart — recomendação de mecanismo de liveness e restart tratada no [FDD](../FDD.md), seção 7).
- Latência mínima de entrega de até 2 segundos no pior caso, mesmo sem qualquer gargalo.
- Ordering de eventos só é garantida por pedido individual e apenas em topologia single-worker; qualquer evolução futura para múltiplos workers exige revisitar esta decisão.
