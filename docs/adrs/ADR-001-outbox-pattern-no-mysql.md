# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

Aceito

## Contexto

O OMS precisa notificar sistemas de clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) quando o status de um pedido muda, com latência abaixo de 10 segundos ([09:00]-[09:02] Marcos). A mudança de status hoje já é uma transação pesada em `OrderService.changeStatus()` ([src/modules/orders/order.service.ts:126](../../src/modules/orders/order.service.ts)): atualiza `orders`, insere em `order_status_history` e debita/repõe `stock_quantity` dos produtos.

Disparar a notificação de forma síncrona dentro dessa transação foi descartado: "a transação de mudança de status hoje já é pesada... se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos" ([09:04] Bruno). Além disso, não há como fazer rollback de uma mudança de status já efetivada só porque o cliente webhook está fora do ar ([09:04] Bruno).

A alternativa de introduzir uma fila dedicada (ex.: Redis Streams) foi levantada por Larissa e descartada por Diego: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" ([09:07] Diego). A decisão foi fechada por Larissa logo em seguida: "Tá decidido então: outbox em MySQL" ([09:08] Larissa).

## Decisão

Adotar o padrão **Transactional Outbox** sobre o MySQL já existente: ao mudar o status de um pedido, dentro da **mesma transação SQL** que atualiza `orders` e `order_status_history`, inserir também uma linha em uma nova tabela `webhook_outbox` com o evento a ser publicado ([09:06] Diego). Um worker separado (ver [ADR-002](ADR-002-worker-processo-separado-em-polling.md)) lê essa tabela de forma assíncrona e dispara as chamadas HTTP.

Se a transação principal fizer commit, o evento foi garantidamente registrado; se sofrer rollback, o evento desaparece junto — "não tem inconsistência possível" ([09:06] Diego). Essa garantia é reforçada explicitamente para a integração com `changeStatus`: "Se ficar fora da transação, perde a garantia toda" ([09:41] Diego); ver seção "Integração com o sistema existente" do [FDD](../FDD.md) para o ponto exato de inserção.

A tabela `webhook_outbox` terá índice nos campos de status (pendente/processando/falhou/entregue) e `created_at`, para o worker ler apenas os pendentes mais antigos em lotes pequenos ([09:08] Diego).

## Alternativas Consideradas

1. **Disparo síncrono dentro de `changeStatus`.** Descartada: acopla a latência/disponibilidade de sistemas externos à transação crítica de negócio, arriscando travar outras mudanças de status e sem estratégia de rollback viável ([09:04] Bruno).
2. **Fila dedicada (Redis Streams ou similar).** Descartada: exigiria subir e operar infraestrutura nova (ex.: Redis Cluster) para um time pequeno, sem necessidade real — o MySQL já existente resolve o problema sem custo operacional adicional ([09:07] Larissa propõe; [09:07] Diego descarta; [09:08] Larissa confirma a decisão).

## Consequências

**Positivas:**
- Garantia atômica entre a mudança de estado do pedido e o registro do evento a ser notificado — nunca existe status mudado sem evento correspondente, nem evento sem status efetivamente mudado.
- Nenhuma infraestrutura nova é introduzida; reaproveita o MySQL já usado pelo restante do sistema (o worker usa uma instância própria de `PrismaClient`, ver [ADR-002](ADR-002-worker-processo-separado-em-polling.md)).
- Desacopla a latência de entrega do webhook da transação de negócio.

**Negativas:**
- Introduz uma tabela e um processo de leitura assíncrona (worker) que não existiam antes, com sua própria operação (monitoramento de fila, arquivamento de eventos antigos — explicitamente fora do escopo desta feature, [09:08] Diego).
- A latência mínima de entrega passa a depender do ciclo de leitura do worker (ver [ADR-002](ADR-002-worker-processo-separado-em-polling.md)), não é mais "imediata" no sentido estrito.
- Acoplamento de esquema: qualquer mudança na tabela `webhook_outbox` precisa considerar o impacto na transação de `changeStatus`, que passa a ser ainda mais crítica.
