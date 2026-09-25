# ADR-004: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

O sistema passa a expor dados de pedidos (incluindo valores financeiros) para endpoints fora da infraestrutura da empresa. Sofia levantou a necessidade de o cliente conseguir validar que a requisição realmente veio do OMS e que o payload não foi adulterado em trânsito: "o cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio" ([09:19] Sofia).

Diego reforçou a importância dessa exigência citando um incidente anterior: "a gente já teve cliente que vazou secret em log de aplicação dele uma vez" ([09:22] Diego), depois que Sofia já havia proposto a rotação com grace period abaixo.

## Decisão

Assinar o corpo de cada requisição de webhook com **HMAC-SHA256**, usando uma secret compartilhada entre o OMS e o cliente, e enviar a assinatura resultante no header `X-Signature` ([09:20] Sofia). SHA-256 foi escolhido por ser "o padrão de mercado, todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).

Cada **endpoint de webhook cadastrado tem sua própria secret**, gerada pelo OMS e devolvida na criação do cadastro ([09:21] Sofia, [09:31] Marcos) — não existe uma secret global da plataforma, evitando que o vazamento de uma credencial comprometa todos os clientes ([09:21] Sofia).

A secret é **rotacionável** via API. Ao rotacionar, a secret antiga permanece válida em paralelo por um **grace period de 24 horas**, dando tempo ao cliente de migrar seus sistemas antes de a secret antiga ser desativada: "Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele" ([09:21] Sofia).

> **Ponto em aberto:** a transcrição define apenas que a secret antiga "fica válida por 24 horas em paralelo" ([09:21] Sofia), sem especificar o que "válida" significa do lado do OMS — com qual secret o serviço assina os eventos enviados durante essa janela (a antiga, a nova, ou ambas), nem como o cliente identificaria qual delas foi usada em cada evento recebido. Ver "Questões em aberto" no [RFC](../RFC.md).

A tabela de configuração de webhook armazena, no mínimo, `url`, `secret`, `customer_id` e um estado ativo/inativo ([09:21] Bruno/Sofia).

## Alternativas Consideradas

1. **Secret única/global para todos os clientes da plataforma.** Descartada explicitamente: "se vaza uma, vaza tudo" ([09:21] Sofia).

## Consequências

**Positivas:**
- Cliente consegue validar autenticidade e integridade de cada evento recebido sem exigir nenhuma infraestrutura de segurança adicional além de verificar a assinatura HMAC do lado dele ([09:20] Sofia: "Cliente verifica do lado dele").
- Isolamento de blast radius: o comprometimento da secret de um cliente não afeta os demais.
- Rotação com grace period permite resposta a vazamentos sem downtime de integração para o cliente afetado.

**Negativas:**
- Exige geração, armazenamento seguro e gestão de ciclo de vida (rotação, expiração) de uma secret por endpoint cadastrado — mais superfície de gestão de segredo do que uma chave única.
- Durante o grace period de 24h, duas secrets ficam simultaneamente válidas para o mesmo endpoint — o mecanismo exato de assinatura/verificação do lado do OMS nesse intervalo é um ponto em aberto (ver nota acima).
- A revisão de segurança da Sofia sobre HMAC e geração de secret é um passo obrigatório antes do deploy: "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma" ([09:46] Sofia), o que é uma dependência explícita no cronograma.
