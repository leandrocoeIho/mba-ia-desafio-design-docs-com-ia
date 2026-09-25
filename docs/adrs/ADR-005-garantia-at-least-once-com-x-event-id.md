# ADR-005: Garantia at-least-once com deduplicação via X-Event-Id

## Status

Aceito

## Contexto

Dado o modelo de retry e o timeout de 10 segundos do worker ([ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md)), é possível que um cliente receba o mesmo evento de webhook mais de uma vez — por exemplo, se o cliente processar a chamada mas não responder dentro dos 10 segundos, o worker trata isso como falha e reenvia o mesmo evento em uma tentativa posterior, mesmo que o processamento original tenha sido bem-sucedido. Diego levantou isso diretamente: "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado" ([09:24] Diego).

Era preciso decidir como o cliente diferencia uma entrega duplicada de um evento novo, e de quem é a responsabilidade por essa deduplicação.

## Decisão

Garantir semântica **at-least-once** (nunca menos que uma entrega, podendo haver duplicatas) e enviar um identificador único por evento no header **`X-Event-Id`**, um UUID gerado no momento em que o evento entra na outbox ([09:25] Diego). O cliente é responsável por deduplicar do lado dele usando esse identificador: "se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado dele" ([09:25] Diego).

Essa responsabilidade é documentada de forma destacada no portal de desenvolvedor para os clientes ([09:26] Marcos), e segue prática já validada por outras plataformas de mercado (Stripe, GitHub, citadas em [09:25] Diego).

## Alternativas Consideradas

1. **Garantia exactly-once.** Descartada: "exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" ([09:25] Diego). Sofia observou que essa escolha "joga responsabilidade pro cliente" ([09:25] Sofia), trade-off aceito conscientemente pelo grupo em troca de simplicidade de implementação.

## Consequências

**Positivas:**
- Implementação simples do lado do OMS: não é necessário nenhum protocolo de confirmação distribuída ou two-phase commit com o cliente.
- Alinhado com práticas já consolidadas no mercado (Stripe, GitHub), reduzindo a curva de integração para clientes que já lidam com webhooks de outras plataformas.
- `X-Event-Id` sendo um UUID é consistente com o padrão de identificadores já usado no restante do projeto (ver [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)); a transcrição não define explicitamente se esse UUID é também a chave primária da linha na `webhook_outbox` — trata-se de uma inferência razoável, não de uma decisão registrada.

**Negativas:**
- Transfere para o cliente a responsabilidade de implementar deduplicação — um cliente que não implementar isso corretamente pode processar o mesmo evento (ex.: mesma mudança de status) mais de uma vez.
- Exige documentação clara e destacada no portal de desenvolvedor, sob risco de suporte reativo caso clientes não leiam essa exigência.
