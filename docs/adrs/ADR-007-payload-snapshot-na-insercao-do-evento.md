# ADR-007: Snapshot do payload no momento da inserção do evento na outbox

## Status

Aceito

## Contexto

Ao definir o padrão outbox ([ADR-001](ADR-001-outbox-pattern-no-mysql.md)), ficou uma dúvida em aberto sobre o que exatamente fica armazenado na linha da `webhook_outbox`: apenas uma referência ao pedido (`order_id`), a ser resolvida no momento do envio, ou o payload já montado no momento em que o evento é criado. Bruno levantou a questão diretamente: "o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?" ([09:51] Bruno).

Essa decisão importa porque, entre o momento em que o status muda e o momento em que o worker efetivamente processa e envia o evento (até 2 segundos depois, ou mais em caso de retry), o pedido pode ter sofrido novas alterações.

> **Nota:** esta decisão (assim como a de UUID no [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)) foi tomada em uma conversa entre Larissa, Diego e Bruno após o encerramento formal da reunião ([09:50], já com Marcos e Sofia fora da call), e por isso não consta no resumo final confirmado em [09:48]. Está registrada aqui porque é uma decisão explícita e sem ambiguidade, apenas cronologicamente posterior ao fechamento da pauta principal.

## Decisão

O payload do evento é **montado (snapshot) no momento da inserção** na outbox, refletindo o estado do pedido exatamente quando a mudança de status ocorreu — e não recalculado a partir do estado atual do pedido no momento do envio: "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou" ([09:52] Larissa, confirmado por Diego e Bruno na sequência).

## Alternativas Consideradas

1. **Guardar apenas `order_id` e renderizar o payload no momento do envio.** Descartada: "Senão tem caso esquisito" ([09:52] Larissa) — o pedido pode ter mudado de estado entre a inserção do evento e o envio (inclusive já ter avançado para um status posterior), fazendo o payload enviado não corresponder à mudança de status que originou aquele evento específico.

## Consequências

**Positivas:**
- Cada evento entregue reflete fielmente o estado do pedido no exato momento da transição de status que o originou, independente de quanto tempo o evento levou para ser efetivamente enviado (incluindo cenários de retry, [ADR-003](ADR-003-retry-com-backoff-e-dead-letter-queue.md)).
- Elimina uma classe inteira de inconsistências onde o cliente receberia um payload desatualizado ou incoerente com o `to_status` do evento.

**Negativas:**
- O payload precisa ser montado dentro da própria transação de `changeStatus`, aumentando ligeiramente o trabalho feito dentro dela.
- Cada linha da outbox carrega uma cópia dos dados do pedido no momento do evento, em vez de apenas uma referência — implicando maior consumo de espaço na tabela `webhook_outbox` em comparação a armazenar só o `order_id`. Esse consumo é limitado pelo teto de 64KB por evento ([09:24] Sofia/Diego) e pela decisão de manter o payload enxuto, sem a lista de items do pedido ([09:43]-[09:44] Diego); o custo de armazenamento em si não foi avaliado na reunião.
