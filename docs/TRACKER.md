# TRACKER: Rastreabilidade de Itens

Mapeia cada item relevante registrado em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/*.md` à sua origem — a transcrição da reunião (`TRANSCRICAO.md`) ou o código-fonte da aplicação. Itens marcados nos próprios documentos como "decisão de implementação não discutida na reunião" ou "recomendação" foram deliberadamente excluídos deste tracker quando não têm nenhuma citação de apoio real (nem transcrição, nem código) — se não é rastreável, não entra aqui, conforme o princípio do próprio desafio.

**Tipos usados na coluna "Tipo":** Contexto, Objetivo, Escopo, Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, Dependência, Risco, Critério de Aceite, Alternativa, Questão em Aberto, Contrato Técnico, Código de Erro, Estratégia de Teste.

**Convenção da coluna Localização:** para Fonte = TRANSCRICAO, um único `[hh:mm] Nome` — o momento mais decisivo da citação (quando a discussão se estendeu por um intervalo com várias falas, escolhe-se a fala que fecha a decisão ou que melhor sustenta o conteúdo da linha). Nuances adicionais (quem levantou o ponto, falas complementares) ficam registradas na coluna Conteúdo, não na Localização. Para Fonte = CODIGO, o caminho do arquivo (com linha, quando relevante).

---

## docs/PRD.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Polling em `GET /orders` deixa a integração lenta e cara para os clientes | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Atlas Comercial sugere migrar para concorrente se a entrega não sair até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Contexto | OMS não tem hoje nenhum mecanismo de eventos/filas/notificação externa | CODIGO | package.json |
| PRD-CTX-04 | docs/PRD.md | Requisito Não Funcional | "Tempo real" definido como abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-PUB-01 | docs/PRD.md | Contexto | Público-alvo: clientes B2B integrados (Atlas, MaxDistribuição, Nova Cargo) | TRANSCRICAO | [09:00] Marcos |
| PRD-PUB-02 | docs/PRD.md | Contexto | Público-alvo: administradores internos (role ADMIN) para replay de DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-SCN-01 | docs/PRD.md | Escopo | Cenário: cliente cadastra webhook e recebe secret | TRANSCRICAO | [09:31] Marcos |
| PRD-SCN-02 | docs/PRD.md | Escopo | Cenário: cliente rotaciona secret após suspeita de vazamento | TRANSCRICAO | [09:22] Diego |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Eliminar dependência de polling (métrica proposta; prazo real de referência é fim de novembro, não "fim do trimestre") | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Latência de entrega abaixo de 10s no caminho feliz | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Zero mudanças de status sem evento correspondente (para status assinados) | TRANSCRICAO | [09:06] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Implementação em até 3 sprints, incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-ESC-IN-01 | docs/PRD.md | Escopo | CRUD de configuração de webhook (cadastro, edição, remoção, listagem) | TRANSCRICAO | [09:33] Bruno |
| PRD-ESC-IN-02 | docs/PRD.md | Escopo | Filtro de eventos por status, por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-ESC-IN-03 | docs/PRD.md | Escopo | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-ESC-IN-04 | docs/PRD.md | Escopo | Entrega via Outbox + worker + retry + DLQ | TRANSCRICAO | [09:06] Diego |
| PRD-ESC-IN-05 | docs/PRD.md | Escopo | Autenticação HMAC-SHA256 por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-ESC-IN-06 | docs/PRD.md | Escopo | Garantia at-least-once com dedup via X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-ESC-IN-07 | docs/PRD.md | Escopo | Histórico de entregas consultável pelo cliente | TRANSCRICAO | [09:34] Marcos |
| PRD-ESC-IN-08 | docs/PRD.md | Escopo | Endpoint admin de replay manual de DLQ | TRANSCRICAO | [09:18] Diego |
| PRD-ESC-IN-09 | docs/PRD.md | Escopo | Webhook é outbound-only (OMS envia, não recebe) | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-OUT-01 | docs/PRD.md | Escopo | E-mail em falhas recorrentes — fora de escopo, adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-OUT-02 | docs/PRD.md | Escopo | Rate limiting de envio — adiado, ponto em aberto | TRANSCRICAO | [09:39] Diego |
| PRD-ESC-OUT-03 | docs/PRD.md | Escopo | Dashboard visual para o cliente — descartado deste projeto | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-OUT-04 | docs/PRD.md | Escopo | Múltiplos workers em paralelo — adiado | TRANSCRICAO | [09:13] Diego |
| PRD-ESC-OUT-05 | docs/PRD.md | Escopo | Arquivamento de eventos entregues — fora de escopo | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-LIM-01 | docs/PRD.md | Restrição | Sem garantia de ordering global entre pedidos (limitação aceita, não adiada) | TRANSCRICAO | [09:13] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cliente cadastra webhook (URL + status a acompanhar) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | OMS gera e devolve a secret na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Cliente edita e remove webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Cliente lista webhooks de um customerId (acesso restrito ao próprio cliente não foi definido na reunião) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Cliente rotaciona secret via API, com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Sistema notifica automaticamente nos status assinados | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Sistema assina cada evento com HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Sistema reentrega eventos com falha, com backoff | TRANSCRICAO | [09:14] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Cliente consulta histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Admin reprocessa manualmente evento em DLQ (restrição a ADMIN decidida em [09:36] Sofia) | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Sistema audita quem executou o replay | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Coleta em ~2s pelo worker; SLA ponta a ponta abaixo de 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | URLs de webhook devem ser HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Secret isolada por endpoint, sem secret global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB, com erro | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa de chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once, não exactly-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Reuso de padrões existentes, sem novo logger | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Worker roda em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | CRUD de webhook aberto a qualquer role autenticada (só o replay exige ADMIN) | TRANSCRICAO | [09:37] Sofia |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox transacional, não síncrono nem fila dedicada | TRANSCRICAO | [09:06] Diego |
| PRD-DEC-02 | docs/PRD.md | Decisão | Worker single em polling, não trigger de banco | TRANSCRICAO | [09:09] Diego |
| PRD-DEC-03 | docs/PRD.md | Decisão | At-least-once com dedup no cliente, não exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-04 | docs/PRD.md | Decisão | Secret por endpoint com rotação, não secret global | TRANSCRICAO | [09:21] Sofia |
| PRD-DEC-05 | docs/PRD.md | Decisão | Retry com backoff + DLQ manual | TRANSCRICAO | [09:15] Diego |
| PRD-DEC-06 | docs/PRD.md | Decisão | Payload como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| PRD-TO-01 | docs/PRD.md | Trade-off | Consistência eventual (outbox+worker) vs. notificação instantânea, para nunca acoplar disponibilidade do cliente à transação de negócio | TRANSCRICAO | [09:04] Bruno |
| PRD-TO-02 | docs/PRD.md | Trade-off | At-least-once transfere responsabilidade de dedup ao cliente, em troca de simplicidade | TRANSCRICAO | [09:25] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Módulo de pedidos existente (`OrderService.changeStatus`) | CODIGO | src/modules/orders/order.service.ts |
| PRD-DEP-02 | docs/PRD.md | Dependência | Infraestrutura MySQL/Prisma já existente | CODIGO | prisma/schema.prisma |
| PRD-DEP-03 | docs/PRD.md | Dependência | Sistema de autenticação/autorização (JWT + requireRole) | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-DEP-04 | docs/PRD.md | Dependência | Revisão de segurança dedicada da Sofia (2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-05 | docs/PRD.md | Dependência | Confirmação do prazo com a Atlas pelo PM | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-06 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor sobre dedup | TRANSCRICAO | [09:26] Marcos |
| PRD-RISK-01 | docs/PRD.md | Risco | Atraso compromete prazo da Atlas / risco de churn | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret, repetindo incidente anterior | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Ambiguidade na contagem de tentativas de retry | TRANSCRICAO | [09:17] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Ausência de rate limiting gera rajadas de chamadas | TRANSCRICAO | [09:39] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Worker sem healthcheck/restart — sem script/infra de worker ainda no projeto | CODIGO | package.json |
| PRD-AC-01 | docs/PRD.md | Critério de Aceite | Cliente cadastra/edita/lista/remove webhooks autenticado | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de Aceite | Notificação de mudança de status dentro do SLA | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-03 | docs/PRD.md | Critério de Aceite | Endpoint indisponível não impede outros pedidos | TRANSCRICAO | [09:04] Bruno |
| PRD-AC-04 | docs/PRD.md | Critério de Aceite | Falha repetida vai para DLQ, reprocessável por ADMIN com auditoria | TRANSCRICAO | [09:18] Diego |
| PRD-AC-05 | docs/PRD.md | Critério de Aceite | URL não-HTTPS é rejeitada no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de Aceite | Status não assinado não gera notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-07 | docs/PRD.md | Critério de Aceite | Rotação mantém secret anterior válida por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-08 | docs/PRD.md | Critério de Aceite | Revisão de segurança realizada antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-AC-09 | docs/PRD.md | Critério de Aceite | Cliente valida autenticidade do evento usando a secret | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-10 | docs/PRD.md | Critério de Aceite | Cliente consulta histórico de entregas do webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-11 | docs/PRD.md | Critério de Aceite | Nenhum item fora de escopo (seção 5) foi implementado | TRANSCRICAO | [09:37] Larissa |
| PRD-TEST-01 | docs/PRD.md | Estratégia de Teste | Testes automatizados no padrão Vitest+supertest do projeto | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Estratégia de Teste | Testes de integração ponta a ponta reservados na estimativa | TRANSCRICAO | [09:46] Larissa |
| PRD-TEST-03 | docs/PRD.md | Estratégia de Teste | Revisão de código de segurança (HMAC/secret) pela Sofia | TRANSCRICAO | [09:46] Sofia |

## docs/RFC.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| RFC-META-01 | docs/RFC.md | Contexto | Larissa se compromete a abrir o doc de design e marcar revisão | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | `OrderService.changeStatus()` como operação transacional isolada, ponto central de integração | CODIGO | src/modules/orders/order.service.ts:126 |
| RFC-PROP-01 | docs/RFC.md | Decisão | Padrão Outbox no MySQL | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker separado em polling de 2s | TRANSCRICAO | [09:09] Diego |
| RFC-PROP-03 | docs/RFC.md | Decisão | Retry com backoff + DLQ | TRANSCRICAO | [09:15] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | HMAC-SHA256 por endpoint | TRANSCRICAO | [09:20] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:24] Diego |
| RFC-PROP-06 | docs/RFC.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| RFC-PROP-07 | docs/RFC.md | Decisão | Payload snapshot na inserção (complementar) | TRANSCRICAO | [09:52] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Disparo síncrono — descartado | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Fila dedicada (Redis Streams) — descartada | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Exactly-once — descartada | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio | TRANSCRICAO | [09:39] Diego |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Assinatura do OMS durante grace period de rotação | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Contagem exata de tentativas de retry | TRANSCRICAO | [09:17] Diego |
| RFC-OPEN-04 | docs/RFC.md | Questão em Aberto | `customerId` no corpo ou no path | TRANSCRICAO | [09:32] Larissa |
| RFC-OPEN-05 | docs/RFC.md | Questão em Aberto | Endurecimento futuro das roles do CRUD de configuração | TRANSCRICAO | [09:37] Sofia |
| RFC-RISK-01 | docs/RFC.md | Risco | Impacto na transação crítica de `changeStatus` | TRANSCRICAO | [09:41] Diego |
| RFC-RISK-02 | docs/RFC.md | Restrição | Ordering apenas por order_id, limitação aceita | TRANSCRICAO | [09:13] Larissa |
| RFC-RISK-03 | docs/RFC.md | Risco | Novo processo (worker) — sem script/infra de worker ainda no projeto | CODIGO | package.json |
| RFC-RISK-04 | docs/RFC.md | Risco | Vazamento de secret já ocorrido no passado | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-05 | docs/RFC.md | Risco | Risco de prazo (3 sprints vs. fim de novembro) | TRANSCRICAO | [09:45] Marcos |

## docs/FDD.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| FDD-OBJ-01 | docs/FDD.md | Objetivo | Registro atômico do evento com a mudança de status | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo | Entrega abaixo de 10s, polling de 2s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-03 | docs/FDD.md | Objetivo | At-least-once com dedup via X-Event-Id | TRANSCRICAO | [09:24] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo | Autenticar cada evento via HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo | Reutilizar padrões existentes sem dependências novas | TRANSCRICAO | [09:30] Larissa |
| FDD-ESC-01 | docs/FDD.md | Escopo | CRUD de webhook incluso | TRANSCRICAO | [09:31] Marcos |
| FDD-ESC-02 | docs/FDD.md | Escopo | E-mail de falha — fora de escopo | TRANSCRICAO | [09:37] Larissa |
| FDD-ESC-03 | docs/FDD.md | Escopo | Múltiplos workers — adiado (distinto de ordering, que é limitação aceita) | TRANSCRICAO | [09:13] Diego |
| FDD-ESC-04 | docs/FDD.md | Escopo | Rotação de secret incluída | TRANSCRICAO | [09:21] Sofia |
| FDD-ESC-05 | docs/FDD.md | Escopo | Histórico de entregas incluído | TRANSCRICAO | [09:34] Marcos |
| FDD-ESC-06 | docs/FDD.md | Escopo | Replay administrativo de DLQ incluído | TRANSCRICAO | [09:18] Diego |
| FDD-ESC-07 | docs/FDD.md | Escopo | Rate limiting de envio — fora de escopo | TRANSCRICAO | [09:39] Diego |
| FDD-ESC-08 | docs/FDD.md | Escopo | Dashboard visual — fora de escopo | TRANSCRICAO | [09:40] Larissa |
| FDD-ESC-09 | docs/FDD.md | Escopo | Arquivamento de eventos entregues — fora de escopo | TRANSCRICAO | [09:08] Diego |
| FDD-FLOW-01 | docs/FDD.md | Decisão | `changeStatus` chama `publishWebhookEvent(tx, ...)` na mesma transação (ponto exato de integração em FDD-INTEG-01) | TRANSCRICAO | [09:41] Diego |
| FDD-FLOW-02 | docs/FDD.md | Restrição | Filtro por status assinado acontece na inserção, não no envio | TRANSCRICAO | [09:34] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Decisão | Rollback total se a inserção do evento falhar | TRANSCRICAO | [09:41] Diego |
| FDD-FLOW-04 | docs/FDD.md | Restrição | Ponto de verificação do limite de 64KB não definido na reunião | TRANSCRICAO | [09:24] Larissa |
| FDD-FLOW-05 | docs/FDD.md | Decisão | Worker com PrismaClient próprio, processo separado | TRANSCRICAO | [09:30] Bruno |
| FDD-FLOW-06 | docs/FDD.md | Restrição | 4 estados do evento (pendente/processando/falhou/entregue) | TRANSCRICAO | [09:08] Diego |
| FDD-FLOW-07 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s por chamada do worker | TRANSCRICAO | [09:42] Diego |
| FDD-FLOW-08 | docs/FDD.md | Restrição | Premissa da reunião: ordering só por order_id, só em single-worker (a quebra sob retry, seção 12, é análise própria do FDD, sem fonte direta na transcrição) | TRANSCRICAO | [09:13] Larissa |
| FDD-FLOW-09 | docs/FDD.md | Decisão | Backoff exponencial 1m/5m/30m/2h/12h | TRANSCRICAO | [09:15] Diego |
| FDD-FLOW-10 | docs/FDD.md | Restrição | Ambiguidade sobre contagem de "5 tentativas" (risco correspondente em FDD §12) | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-11 | docs/FDD.md | Decisão | DLQ em tabela separada `webhook_dead_letter` | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-12 | docs/FDD.md | Requisito Funcional | Replay exige ADMIN e gera log de auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-FLOW-13 | docs/FDD.md | Decisão | Worker como entry-point `src/worker.ts` / script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| FDD-DEC-PREFIX | docs/FDD.md | Decisão | Prefixo `WEBHOOK_` para todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato Técnico | `POST /webhooks` — cadastro, secret devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato Técnico | `GET /webhooks` — listagem por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato Técnico | `PATCH /webhooks/:id` — edição | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato Técnico | `DELETE /webhooks/:id` — remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato Técnico | Endpoint de rotação de secret (rota exata `/rotate-secret` é definição do FDD, não da reunião) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato Técnico | `GET /webhooks/:id/deliveries` — histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato Técnico | `POST /admin/webhooks/dead-letter/:id/replay` — replay | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato Técnico | Headers X-Event-Id/X-Signature/X-Timestamp/X-Webhook-Id + payload de saída | TRANSCRICAO | [09:44] Diego |
| FDD-ERRO-01 | docs/FDD.md | Código de Erro | `WEBHOOK_NOT_FOUND` — nova subclasse de AppError | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-02 | docs/FDD.md | Código de Erro | `WEBHOOK_CUSTOMER_NOT_FOUND` — análogo à checagem em OrderService.create | CODIGO | src/modules/orders/order.service.ts:60 |
| FDD-ERRO-03 | docs/FDD.md | Código de Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` (nome do código é convenção do FDD; endpoint vem da reunião) | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-04 | docs/FDD.md | Código de Erro | Limite de 64KB com erro (nome do código `WEBHOOK_PAYLOAD_TOO_LARGE` e checagem no envio são recomendação do FDD, não definição da reunião) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-05 | docs/FDD.md | Código de Erro | `VALIDATION_ERROR` reutilizado para URL/status inválidos | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-ERRO-06 | docs/FDD.md | Código de Erro | `FORBIDDEN` reutilizado no replay restrito a ADMIN | CODIGO | src/middlewares/auth.middleware.ts:49-57 |
| FDD-ERRO-07 | docs/FDD.md | Código de Erro | `WEBHOOK_SECRET_REQUIRED` e `WEBHOOK_INVALID_URL` deliberadamente excluídos (secret sempre gerada pelo OMS; URL vira VALIDATION_ERROR genérico) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-08 | docs/FDD.md | Código de Erro | `UNAUTHORIZED` reutilizado (token ausente/inválido, qualquer endpoint autenticado) | CODIGO | src/middlewares/auth.middleware.ts:27 |
| FDD-ERRO-09 | docs/FDD.md | Código de Erro | `CONFLICT` reutilizado (violação de unicidade do Prisma) | CODIGO | src/middlewares/error.middleware.ts |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Fallback: sem e-mail automático nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-RESIL-02 | docs/FDD.md | Requisito Não Funcional | Isolamento de falhas entre API e worker | TRANSCRICAO | [09:11] Diego |
| FDD-RESIL-03 | docs/FDD.md | Restrição | Healthcheck/restart do worker — sem script/infra de worker ainda no projeto | CODIGO | package.json |
| FDD-OBS-01 | docs/FDD.md | Decisão | Logs estruturados via Pino, padrão já existente | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-02 | docs/FDD.md | Decisão | Correlação via event_id, análogo ao X-Request-Id | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-03 | docs/FDD.md | Restrição | Sem biblioteca de métricas/tracing no projeto hoje | CODIGO | package.json |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node >=20, fetch nativo para chamadas HTTP | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL via Prisma 5.22.0 já em uso | CODIGO | package.json |
| FDD-DEP-03 | docs/FDD.md | Dependência | Sem novas dependências de runtime (zod, pino, jsonwebtoken já existentes) | CODIGO | package.json |
| FDD-INTEG-01 | docs/FDD.md | Decisão | Integração em `changeStatus()` | CODIGO | src/modules/orders/order.service.ts:126 |
| FDD-INTEG-02 | docs/FDD.md | Decisão | Novas subclasses de erro seguindo padrão existente | CODIGO | src/shared/errors/http-errors.ts:45-63 |
| FDD-INTEG-03 | docs/FDD.md | Decisão | Middleware de erro sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-04 | docs/FDD.md | Decisão | `requireRole('ADMIN')` reutilizado no replay | CODIGO | src/middlewares/auth.middleware.ts:49 |
| FDD-INTEG-05 | docs/FDD.md | Decisão | Logger Pino reutilizado sem alteração estrutural (gap de `redactPaths` detalhado em FDD-RISK-03) | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-06 | docs/FDD.md | Decisão | Novo router `/webhooks` no padrão de composição existente | CODIGO | src/routes/index.ts |
| FDD-INTEG-07 | docs/FDD.md | Decisão | Novos modelos Prisma seguindo convenção de UUID | CODIGO | prisma/schema.prisma |
| FDD-INTEG-08 | docs/FDD.md | Decisão | Novo script `worker` espelhando `dev`/`start` | CODIGO | package.json |
| FDD-INTEG-09 | docs/FDD.md | Decisão | `requireRole('ADMIN')` já usado hoje em outro endpoint (precedente real) | CODIGO | src/modules/users/user.routes.ts:15 |
| FDD-AC-01 | docs/FDD.md | Critério de Aceite | Falha na inserção reverte também a mudança de status | TRANSCRICAO | [09:41] Diego |
| FDD-AC-02 | docs/FDD.md | Critério de Aceite | Status não assinado não gera linha na outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério de Aceite | Endpoint com erro/timeout reagenda conforme backoff | TRANSCRICAO | [09:15] Diego |
| FDD-AC-04 | docs/FDD.md | Critério de Aceite | Esgotadas tentativas, evento vai para DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-AC-05 | docs/FDD.md | Critério de Aceite | Replay 403 não-ADMIN / 200 ADMIN com auditoria | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-06 | docs/FDD.md | Critério de Aceite | X-Signature validável com a secret devolvida | TRANSCRICAO | [09:20] Sofia |
| FDD-AC-07 | docs/FDD.md | Critério de Aceite | URL não-HTTPS rejeitada | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-08 | docs/FDD.md | Critério de Aceite | Mesmo X-Event-Id em todas as tentativas | TRANSCRICAO | [09:24] Diego |
| FDD-AC-09 | docs/FDD.md | Critério de Aceite | Rotação mantém secret válida por 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-10 | docs/FDD.md | Critério de Aceite | Worker entrega evento pendente em ~2s no caminho feliz | TRANSCRICAO | [09:09] Diego |
| FDD-AC-11 | docs/FDD.md | Critério de Aceite | `GET /webhooks/:id/deliveries` retorna sucesso e falha com tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-AC-12 | docs/FDD.md | Critério de Aceite | Chamada de saída inclui os 5 headers do contrato | TRANSCRICAO | [09:44] Diego |
| FDD-RISK-01 | docs/FDD.md | Risco | Ambiguidade da contagem de tentativas (mesmo ponto de FDD-FLOW-10) | TRANSCRICAO | [09:17] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Assinatura do OMS durante grace period não definida | TRANSCRICAO | [09:21] Sofia |
| FDD-RISK-03 | docs/FDD.md | Risco | `redactPaths` não cobre campo `secret` | CODIGO | src/shared/logger/index.ts |
| FDD-RISK-04 | docs/FDD.md | Risco | Ordering quebra sob retry (análise própria do FDD sobre a premissa de FDD-FLOW-08; sem fonte direta na transcrição para a conclusão) | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-05 | docs/FDD.md | Risco | Assinar webhook a `PENDING` nunca dispararia evento | CODIGO | src/modules/orders/order.status.ts |

## docs/adrs/

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| ADR-001-DEC | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Padrão Outbox no MySQL, mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Alternativa | Disparo síncrono — descartado | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Alternativa | Fila dedicada (Redis Streams) — descartada | TRANSCRICAO | [09:07] Diego |
| ADR-001-COD | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Ponto de integração citado | CODIGO | src/modules/orders/order.service.ts:126 |
| ADR-002-DEC | docs/adrs/ADR-002-worker-processo-separado-em-polling.md | Decisão | Worker separado, processo próprio, polling de 2s | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-processo-separado-em-polling.md | Alternativa | Worker embutido na API — descartado | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-processo-separado-em-polling.md | Alternativa | Trigger de banco — descartado | TRANSCRICAO | [09:09] Diego |
| ADR-002-REST | docs/adrs/ADR-002-worker-processo-separado-em-polling.md | Restrição | Ordering só por order_id, só em single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-003-DEC | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Decisão | Backoff exponencial, 5 tentativas, DLQ | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Alternativa | 3 tentativas — descartada | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Alternativa | Retry indefinido — descartado | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Alternativa | Status na própria outbox, sem tabela separada — descartado (pergunta de Larissa em [09:17]) | TRANSCRICAO | [09:18] Diego |
| ADR-003-REPLAY | docs/adrs/ADR-003-retry-com-backoff-e-dead-letter-queue.md | Requisito Funcional | Endpoint admin de replay manual | TRANSCRICAO | [09:18] Diego |
| ADR-004-DEC-01 | docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md | Decisão | HMAC-SHA256 sobre o payload | TRANSCRICAO | [09:20] Sofia |
| ADR-004-DEC-02 | docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md | Decisão | Secret única por endpoint, rotação com grace period 24h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT | docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md | Alternativa | Secret global — descartada | TRANSCRICAO | [09:21] Sofia |
| ADR-004-OPEN | docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md | Questão em Aberto | Assinatura do OMS durante grace period | TRANSCRICAO | [09:21] Sofia |
| ADR-004-FIELDS | docs/adrs/ADR-004-autenticacao-hmac-sha256-por-endpoint.md | Decisão | Tabela de configuração armazena url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-005-DEC | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | At-least-once, dedup via X-Event-Id | TRANSCRICAO | [09:24] Diego |
| ADR-005-ALT | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Alternativa | Exactly-once — descartada | TRANSCRICAO | [09:25] Diego |
| ADR-006-DEC | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo de padrões existentes (módulo, erros, logger, UUID) | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa | ID auto-incremental na outbox — descartado | TRANSCRICAO | [09:51] Larissa |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Alternativa | Novo logger — descartado | TRANSCRICAO | [09:29] Bruno |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Padrão de módulo (controller/service/repository/routes/schemas) citado | CODIGO | src/modules/orders/ |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Padrão de erro citado | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | `requireRole` citado | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Padrão UUID nas tabelas de entidade | CODIGO | prisma/schema.prisma |
| ADR-007-DEC | docs/adrs/ADR-007-payload-snapshot-na-insercao-do-evento.md | Decisão | Payload como snapshot no momento da inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT | docs/adrs/ADR-007-payload-snapshot-na-insercao-do-evento.md | Alternativa | Guardar só order_id, renderizar no envio — descartada | TRANSCRICAO | [09:52] Larissa |

## Resumo de Cobertura

- **Total de itens rastreados:** 211 (contagem verificada por script, não estimada).
- **Fonte TRANSCRICAO:** 174 linhas (~82% do total) — **todas** no formato estrito `[hh:mm] Nome` (um único timestamp, um único falante), verificado por script. Muito acima do mínimo de 70% exigido pelo desafio. Nuances de atribuição (quem levantou o ponto vs. quem decidiu) ficam registradas na coluna Conteúdo, não na Localização.
- **Fonte CODIGO:** 37 linhas (~18% do total) — acima do mínimo de 5 linhas exigido, cobrindo `order.service.ts`, `order.status.ts`, `http-errors.ts`, `error.middleware.ts`, `auth.middleware.ts`, `user.routes.ts`, `logger/index.ts`, `validate.middleware.ts`, `request-logger.middleware.ts`, `routes/index.ts`, `prisma/schema.prisma`, `package.json` e `tests/orders.test.ts`.
- **Itens excluídos deliberadamente:** recomendações de implementação sem nenhuma citação de apoio (ex.: "validação de contrato" e "cliente piloto" no PRD, seção 12) — mantidas nos documentos como recomendação explícita, mas fora deste tracker por não terem origem rastreável, conforme a diretriz do próprio desafio.

Comando usado para verificar a contagem (linhas de dados, TRANSCRICAO, CODIGO, e formato estrito `[hh:mm] Nome`):
```
grep -cE '^\| [A-Z]+-' TRACKER.md
grep -E '^\| [A-Z]+-' TRACKER.md | grep -c 'TRANSCRICAO'
grep -E '^\| [A-Z]+-' TRACKER.md | grep -c '| CODIGO |'
grep -E '^\| [A-Z]+-.*TRANSCRICAO' TRACKER.md | grep -cE '\[[0-9]{2}:[0-9]{2}\] [A-ZÀ-Ú][a-zà-ú]+ \|$'
```
