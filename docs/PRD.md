# PRD: Sistema de Webhooks de Notificação de Pedidos

> Documento de produto: o "por quê" e o "o quê". Para a proposta técnica, ver o [RFC](RFC.md); para o "como construir", ver o [FDD](FDD.md); para cada decisão de arquitetura isolada, ver os [ADRs](adrs/).

## 1. Resumo e Contexto da Feature

O Order Management System (OMS) hoje não notifica ninguém quando o status de um pedido muda — clientes integrados descobrem mudanças fazendo polling periódico em `GET /orders`. Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram formalmente notificação em tempo real, com a Atlas Comercial chegando a sugerir migração para um concorrente se isso não for entregue até o fim do trimestre ([09:00] Marcos).

Esta feature adiciona um sistema de **webhooks outbound**: o OMS passa a notificar proativamente os sistemas dos clientes sempre que o status de um pedido muda, com garantias de entrega, segurança e auditabilidade. A decisão técnica foi definida em reunião entre Tech Lead, PM, engenharia e segurança, registrada em `TRANSCRICAO.md`, e detalhada tecnicamente no [RFC](RFC.md), no [FDD](FDD.md) e nos [ADRs](adrs/).

## 2. Problema e Motivação

- **Integração cara e lenta para o cliente:** "isso tá deixando a integração lenta e cara pra eles" — os clientes precisam fazer polling constante em vez de serem avisados ([09:00] Marcos).
- **Risco de perda de cliente:** a Atlas Comercial "chegou a sugerir" migrar para um concorrente se a notificação em tempo real não for entregue até o fim do trimestre ([09:00] Marcos).
- **Ausência total de mecanismo de eventos:** o OMS não tem hoje nenhuma infraestrutura de eventos, filas ou notificação externa (confirmado no código: sem dependências de mensageria em `package.json` nem módulo correspondente em `src/`) — toda mudança de status é uma operação puramente interna.
- **Requisito de latência explícito:** os clientes definiram "tempo real" como qualquer coisa abaixo de 10 segundos ([09:02] Marcos) — um requisito mensurável, não apenas uma aspiração vaga.

## 3. Público-Alvo e Cenários de Uso

**Público-alvo:**
- **Clientes B2B integrados** (perfil representado por Atlas Comercial, MaxDistribuição, Nova Cargo): sistemas externos que consomem a API do OMS e precisam saber quando os pedidos deles mudam de status.
- **Administradores internos do OMS** (usuários com role `ADMIN`): responsáveis por operar o sistema, incluindo reprocessar manualmente entregas que falharam permanentemente.

**Cenários de uso:**
1. Um cliente cadastra um endpoint de webhook via API (autenticado com JWT do OMS, [09:32] Marcos/Larissa), informando URL e quais status de pedido quer acompanhar; recebe uma secret gerada pelo OMS na resposta ([09:31] Marcos).
2. Um pedido daquele cliente muda de status (ex.: de `PROCESSING` para `SHIPPED`); o cliente recebe uma notificação HTTP assinada em poucos segundos, sem precisar fazer polling.
3. O endpoint do cliente está temporariamente fora do ar; o OMS tenta novamente algumas vezes, espaçando as tentativas, antes de desistir daquele evento especificamente.
4. Um administrador, de posse do id de um evento em falha permanente, reprocessa-o manualmente depois que o cliente volta a responder ([09:18] Diego) — a reunião não definiu como o administrador descobre esse id (nenhum endpoint de consulta/listagem da DLQ foi discutido).
5. Um cliente consulta o histórico de entregas do seu webhook para auditar o que foi enviado e diagnosticar problemas de integração do lado dele ([09:34] Marcos).
6. Um cliente rotaciona a secret do seu webhook (por exemplo, após suspeita de vazamento, [09:22] Diego) e tem 24 horas para migrar seus sistemas, período em que a secret antiga continua válida ([09:21] Sofia) — o comportamento de assinatura do próprio OMS durante essa janela é uma questão em aberto (ver [RFC](RFC.md)).

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica de sucesso |
|---|---|
| **Eliminar a dependência de polling** dos clientes B2B integrados | *(métrica proposta, não discutida na reunião)* Atlas, MaxDistribuição e Nova Cargo com pelo menos um webhook ativo cadastrado até **o fim de novembro** — prazo pedido pela Atlas ([09:45] Marcos), a ser confirmado pelo PM junto ao cliente ([09:47] Marcos). Note-se que "fim do trimestre" ([09:00] Marcos) foi a ameaça de churn relatada pela Atlas, não necessariamente a mesma data que o prazo de entrega pedido depois ([09:45]) — os dois não devem ser tratados como sinônimos. |
| **Notificar mudanças de status dentro do SLA acordado** | Latência de entrega **abaixo de 10 segundos** no caminho feliz — cliente respondendo prontamente à chamada do worker ([09:02] Marcos; polling de 2s confirmado em [09:09]-[09:10] Diego/Marcos/Larissa) |
| **Garantir consistência entre pedido e notificação** | **Zero** casos de mudança de status, para status assinados por ao menos um webhook ativo, sem o evento de webhook correspondente registrado (garantia de atomicidade do padrão Outbox, [09:06], [09:40]-[09:41] Bruno/Diego; [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md)) |
| **Entregar dentro da janela de engenharia estimada** | Implementação concluída em até **3 sprints** (estimativa de Larissa, "eu chuto três sprints", [09:46] Larissa), incluindo a revisão de segurança dedicada da Sofia ([09:45]-[09:47] Larissa/Sofia) |

## 5. Escopo

### Incluso
- Cadastro, edição, remoção e listagem de webhooks por cliente (CRUD de configuração) ([09:31]-[09:33] Marcos/Bruno).
- Filtro de eventos por status do pedido, por webhook ([09:33]-[09:34] Marcos/Bruno/Diego).
- Rotação de secret com grace period de 24h ([09:21] Sofia).
- Entrega de eventos via padrão Outbox + worker assíncrono, com retry e Dead Letter Queue ([ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md), [ADR-002](adrs/ADR-002-worker-processo-separado-em-polling.md), [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)).
- Autenticação de cada evento via HMAC-SHA256 com secret por endpoint ([ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md)).
- Garantia at-least-once com deduplicação do lado do cliente via `X-Event-Id` ([ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).
- Histórico de entregas por webhook, consultável pelo cliente ([09:34] Marcos).
- Endpoint administrativo de replay manual de eventos em falha permanente, restrito a `ADMIN` ([09:18]-[09:19], [09:35]-[09:36]).

### Fora de Escopo
- **Notificação por e-mail em caso de falhas recorrentes de entrega.** Explicitamente descartada para esta fase: "Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" ([09:37] Larissa).
- **Rate limiting de envio ao cliente.** Discutido e deliberadamente adiado, não resolvido nesta feature: "Eu acho que não [faz parte do escopo]. A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto" ([09:38]-[09:39] Diego).
- **Dashboard visual para o cliente acompanhar seus webhooks.** Descartado deste projeto: "Não, agora não... Painel é projeto separado do time de frontend" ([09:39] Marcos pergunta; [09:40] Larissa responde).
- **Suporte a múltiplos workers em paralelo.** Adiado explicitamente: "isso é problema do futuro, não agora" ([09:13] Diego).
- **Arquivamento de eventos já entregues na outbox.** Fora do escopo desta feature ([09:08] Diego).

**Limitação conhecida e aceita (não é item adiado, é uma não-garantia assumida deliberadamente):** não há garantia de ordering global entre pedidos diferentes — apenas por `order_id`, e só enquanto o worker for único: "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker" ([09:13] Larissa). Os clientes nunca pediram essa garantia: "eles só querem saber se cada pedido deles mudou" ([09:14] Marcos).

## 6. Requisitos Funcionais

1. O cliente deve conseguir cadastrar um webhook informando URL de destino e a lista de status de pedido que deseja acompanhar ([09:31]-[09:33] Marcos).
2. O OMS deve gerar a secret do webhook automaticamente e devolvê-la na resposta de criação — o cliente nunca informa a própria secret ([09:31] Marcos).
3. O cliente deve conseguir editar (PATCH) e remover (DELETE) um webhook cadastrado ([09:33] Bruno).
4. O cliente deve conseguir listar os webhooks cadastrados de um `customerId` ([09:33] Bruno) — a reunião não definiu se o acesso é restrito ao próprio cliente ou aberto a qualquer usuário autenticado (o CRUD, de modo geral, aceita "qualquer role autenticada", [09:36]-[09:37] Marcos/Sofia).
5. O cliente deve conseguir rotacionar a secret do seu webhook via API, com a secret anterior permanecendo válida por 24 horas em paralelo ([09:21] Sofia).
6. O sistema deve notificar automaticamente o(s) webhook(s) inscritos sempre que um pedido daquele cliente mudar para um dos status assinados ([09:33]-[09:34] Marcos/Bruno/Diego).
7. O sistema deve assinar cada evento enviado com HMAC-SHA256, permitindo ao cliente validar autenticidade e integridade ([09:19]-[09:20] Sofia).
8. O sistema deve reentregar automaticamente eventos que falharem, com backoff crescente, antes de desistir de uma entrega ([09:14]-[09:17] Larissa/Diego).
9. O cliente deve conseguir consultar o histórico das últimas entregas de um webhook, incluindo sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos).
10. Um administrador (`ADMIN`) deve conseguir reprocessar manualmente um evento que esgotou as tentativas automáticas de entrega ([09:18] Diego; restrição a `ADMIN` decidida em [09:35]-[09:36] Sofia/Larissa).
11. O sistema deve registrar em auditoria quem executou um replay manual de evento em falha ([09:35]-[09:36] Sofia).

## 7. Requisitos Não Funcionais

- **Latência:** o worker coleta um evento pendente em até ~2 segundos após o commit da transação (intervalo de polling, [09:09] Diego; [09:10] Larissa) — isso **não** é o tempo total de entrega, que ainda inclui a chamada HTTP ao cliente (até 10s de timeout, seção abaixo). O SLA de ponta a ponta combinado com os clientes é entrega **abaixo de 10 segundos** no caminho feliz ([09:02] Marcos).
- **Segurança de transporte:** URLs de webhook devem ser HTTPS; URLs `http://` são rejeitadas no cadastro ([09:23] Sofia).
- **Isolamento de credenciais:** cada webhook tem sua própria secret — não existe secret compartilhada entre clientes ([09:21] Sofia).
- **Limite de payload:** eventos que ultrapassem 64KB geram erro ([09:23]-[09:24] Sofia/Diego/Larissa: "64KB de limite, erro caso ultrapasse") — o ponto exato de verificação desse limite ainda está em aberto (ver [FDD](FDD.md), seção 4.1).
- **Timeout de entrega:** 10 segundos por tentativa de chamada ao cliente ([09:42] Sofia/Diego).
- **Garantia de entrega:** at-least-once (o cliente pode receber duplicatas e deve saber deduplicar) — não exactly-once ([09:24]-[09:26] Diego).
- **Reuso de padrões existentes:** a feature deve seguir as convenções de módulo, erro, log e autenticação já usadas no OMS ("reuso máximo do que já existe", [09:30] Larissa), sem introduzir um novo logger ("Não vamos botar nada novo", [09:29] Bruno). A ausência de qualquer dependência de runtime nova é uma extensão desse princípio, feita no [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md), não uma frase literal da reunião.
- **Isolamento operacional:** o processo que entrega os webhooks deve rodar separado da API, para que um problema em um não derrube o outro ([09:11] Diego).

## 8. Decisões e Trade-offs Principais

Cada decisão abaixo tem um ADR próprio com o racional completo, alternativas e consequências:

- **Outbox transacional no MySQL, não disparo síncrono nem fila dedicada** — evita acoplar a disponibilidade de clientes externos à transação de mudança de status, sem exigir infraestrutura nova ([ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md)).
- **Worker single em polling de 2s, não trigger de banco** — simplicidade operacional em troca de uma latência mínima de até 2s e ausência de ordering global entre pedidos diferentes ([ADR-002](adrs/ADR-002-worker-processo-separado-em-polling.md)).
- **At-least-once com dedup no cliente, não exactly-once** — reduz complexidade de implementação do lado do OMS, transferindo a responsabilidade de deduplicação ao cliente ([ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).
- **Secret por endpoint com rotação, não secret global** — isola o impacto de um vazamento de credencial a um único cliente, ao custo de mais superfície de gestão de segredo ([ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md)).
- **Retry com backoff + DLQ manual, não retry indefinido nem poucas tentativas** — um evento pode levar até ~15 horas para ser definitivamente classificado como falha, e a reentrega depois disso depende de ação manual de um administrador, não de retry automático ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md)).
- **Payload como snapshot no momento da mudança de status, não recalculado no envio** — garante que o evento sempre reflita o estado do pedido no instante da transição, mesmo que o pedido mude de novo antes da entrega efetiva ([ADR-007](adrs/ADR-007-payload-snapshot-na-insercao-do-evento.md)).

O trade-off central da proposta como um todo: entrega **eventualmente consistente** (via outbox + worker assíncrono) em vez de notificação instantânea, aceitando uma latência mínima de alguns segundos em troca de nunca acoplar a disponibilidade de um cliente externo à operação crítica de mudança de status do pedido.

## 9. Dependências

- **Módulo de pedidos existente** (`src/modules/orders/`) — a integração central acontece dentro de `OrderService.changeStatus()`.
- **Infraestrutura de banco já existente** (MySQL via Prisma) — o Outbox e a DLQ são novas tabelas no mesmo banco, sem infraestrutura nova.
- **Sistema de autenticação/autorização já existente** (JWT + `requireRole`) — reaproveitado para autenticar o CRUD de webhooks e restringir o replay de DLQ a `ADMIN`.
- **Revisão de segurança dedicada da Sofia** antes do deploy, com pelo menos 2 dias úteis reservados para revisar HMAC e geração de secret ([09:46] Sofia) — é uma dependência de cronograma, não apenas técnica.
- **Confirmação do prazo com a Atlas** pelo PM, a ser feita após a reunião ([09:47] Marcos: "Eu confirmo prazo com eles"; [09:49] Marcos: "Eu atualizo os clientes hoje à tarde") — a Atlas pediu entrega até o fim de novembro ([09:45] Marcos).
- **Documentação no portal do desenvolvedor** sobre a responsabilidade do cliente de deduplicar eventos por `X-Event-Id` e sobre como integrar com a nova API — comprometida por Marcos: "Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema" ([09:26] Marcos, reforçado em [09:40] Marcos). A garantia de at-least-once do produto depende dessa comunicação chegar ao cliente.

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Atraso na entrega compromete o prazo pedido pela Atlas (fim de novembro, [09:45] Marcos), com risco de churn do cliente ([09:00] Marcos) | Média | Alto (perda de cliente B2B) | Estimativa de 3 sprints já inclui a revisão de segurança da Sofia no fim ([09:45]-[09:47] Larissa) — *acompanhamento de progresso por sprint é recomendação, não discutida na reunião* |
| Vazamento de secret de um cliente, repetindo incidente já ocorrido no passado ([09:22] Diego) | Média (já ocorreu antes com outro cliente) | Alto (compromete integridade das notificações daquele cliente) | Secret por endpoint (isola o impacto a um único cliente) + rotação com grace period de 24h ([ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md)) + revisão de segurança dedicada antes do deploy. Risco residual: durante as 24h de grace period, a secret vazada continua válida em paralelo. |
| Ambiguidade sobre a contagem exata de tentativas de retry ("5 tentativas") gera implementação divergente da intenção original (ver [RFC](RFC.md), Questões em Aberto) | Média | Baixo-Médio (comportamento de retry ligeiramente diferente do esperado) | Confirmar com Larissa/Diego antes da implementação (documentado como ação pendente no [ADR-003](adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md) e no [FDD](FDD.md)) |
| Ausência de rate limiting de saída gera rajadas de chamadas a um cliente com muitos pedidos mudando de status ao mesmo tempo — cenário hipotético levantado por Diego, não observado nem medido ([09:38]-[09:39] Diego) | Baixa (hipotético) | Médio (pode sobrecarregar o endpoint do cliente) | Nenhuma mitigação nesta fase — decisão deliberada de "observar e decidir depois"; requer monitoramento pós-lançamento |
| O worker não tem healthcheck nem política de restart definidos ([RFC](RFC.md); [FDD](FDD.md), seção 12) — uma falha silenciosa do processo interromperia toda a entrega de webhooks sem alarme automático | Baixa | Alto (toda notificação para todos os clientes para) | Nenhuma definida na reunião; definir mecanismo de liveness/restart antes do deploy em produção |

*Nota: a garantia de ordem por `order_id` pode ser quebrada quando um evento entra em retry (ver [FDD](FDD.md), seção 12) — risco técnico já registrado lá, não duplicado aqui por ser de impacto predominantemente técnico, não de produto.*

## 11. Critérios de Aceitação

- [ ] Um cliente consegue cadastrar, editar, listar e remover webhooks via API autenticada.
- [ ] Uma mudança de status de pedido gera uma notificação para todo webhook do cliente inscrito naquele status, entregue em condições normais dentro do SLA de 10 segundos.
- [ ] Um cliente consegue validar a autenticidade de um evento recebido usando a secret retornada no cadastro.
- [ ] Um endpoint de cliente indisponível não impede mudanças de status de outros pedidos.
- [ ] Um evento que falha repetidamente é reentregue com backoff crescente e, esgotadas as tentativas, fica disponível para reprocessamento manual por um `ADMIN`, com o replay registrado em auditoria.
- [ ] Um cliente consegue consultar o histórico de entregas do seu webhook.
- [ ] Cadastrar um webhook com URL não-HTTPS é rejeitado.
- [ ] Uma mudança de status para um valor que nenhum webhook do cliente assina não gera notificação alguma.
- [ ] Rotacionar a secret de um webhook mantém a secret anterior válida por 24 horas.
- [ ] A revisão de segurança da Sofia foi realizada antes do deploy em produção (o critério de "aprovação" formal não foi discutido na reunião).
- [ ] Nenhum requisito marcado como "fora de escopo" (seção 5) foi implementado nesta entrega.

## 12. Estratégia de Testes e Validação

- **Testes automatizados (Vitest + supertest):** seguindo o padrão já usado no projeto em `tests/orders.test.ts` (testes de integração sobre o app real e o banco), cobrindo as regras de negócio do módulo de webhooks (validação de URL HTTPS, filtro de eventos por status, geração/validação de HMAC). A reunião não detalhou a separação entre testes unitários e de integração — fica como recomendação de implementação.
- **Testes de integração ponta a ponta:** o fluxo completo outbox → worker → entrega, incluindo os caminhos de sucesso, falha com retry, e falha permanente com DLQ; e a integração real dentro de `OrderService.changeStatus()` (garantindo que uma falha na inserção do evento reverte a mudança de status, conforme [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md)). A própria reunião reservou tempo de estimativa para isso: "Integração no order.service e testes ponta a ponta é mais meio [sprint]" ([09:46] Larissa).
- **Revisão de código de segurança:** a Sofia revisa especificamente HMAC e geração de secret antes do deploy, com pelo menos 2 dias úteis reservados ([09:46] Sofia: "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança... HMAC e geração de secret eu quero olhar com calma") — é revisão de código, não teste manual funcional.
- **Validação de contrato (recomendação, não discutida na reunião):** conferência dos payloads, headers e status codes de cada endpoint contra o [FDD](FDD.md), seção 5.
- **Validação com cliente piloto (recomendação, não discutida na reunião):** antes do rollout completo, validar a integração ponta a ponta com pelo menos um dos três clientes que solicitaram a feature (Atlas, MaxDistribuição ou Nova Cargo) — dado o prazo que a Atlas pediu ([09:45] Marcos), ainda a ser confirmado pelo PM ([09:47] Marcos).
