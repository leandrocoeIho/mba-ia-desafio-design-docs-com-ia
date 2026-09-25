# Sistema de Webhooks de Notificação de Pedidos — Design Docs

Pacote de design docs produzido para o desafio de MBA em Engenharia de Software "Da Reunião ao Documento: Design Docs Gerados por IA". O enunciado original completo está preservado no histórico do Git (commit inicial deste fork) e no repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia). Este README documenta o processo de produção, não o enunciado.

## Sobre o desafio

A tarefa era transformar a transcrição literal de uma reunião técnica (`TRANSCRICAO.md`) e o código de um Order Management System (OMS) já em produção em um pacote completo de design docs — PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade — para uma nova feature de webhooks de notificação de pedidos. A reunião já tinha fechado as decisões técnicas; nada disso estava documentado além da gravação da call.

A restrição central do desafio não é escrever bonito, é não inventar: toda frase registrada nos documentos precisa apontar para um timestamp real da transcrição ou para um caminho de arquivo real do código. O papel esperado de quem faz o desafio é o de maestro — usar a IA para ler, estruturar e redigir, mas revisar criticamente cada afirmação antes de aceitá-la, especialmente porque é fácil para um modelo de linguagem "arredondar" uma frase incerta da reunião em algo que soa como fato decidido.

## Ferramentas de IA utilizadas

- **Claude Code (Sonnet 5)** — orquestrador principal da sessão inteira. Leu o código-fonte do OMS e a transcrição completa, construiu o inventário classificado de decisões, redigiu os 7 ADRs, o RFC, o FDD, o PRD e o Tracker, aplicou todas as correções apontadas pelas rodadas de verificação, e conduziu a conversa de definição de escopo/processo com o autor.
- **Claude Opus**, via subagentes disparados pelo Claude Code — usado exclusivamente como **verificador cético independente**, nunca para redigir. Cada documento, depois de escrito, foi relido por um agente Opus separado que comparava frase a frase contra a transcrição, o código e os demais documentos já produzidos, procurando citações distorcidas, alternativas inventadas, contradições entre documentos e afirmações sobre o código que não conferiam com o código real. Essa separação entre "quem escreve" e "quem audita" foi uma decisão deliberada, discutida e ajustada com o autor no meio do processo (ver seção de Workflow).

## Workflow adotado

O processo não seguiu geração única. Foi estruturado em duas fases por documento — **rascunho** e **verificação crítica** — repetidas para cada um dos cinco artefatos, e fechado com uma **auditoria final holística** sobre o pacote inteiro.

1. **Contextualização**: leitura completa do código relevante do OMS (módulo de pedidos, classes de erro, middlewares de autenticação/log/validação, schema Prisma) e da transcrição inteira, sem resumir.
2. **Inventário classificado da transcrição** (etapa que não estava no roteiro sugerido pelo curso, mas foi proposta pelo autor depois de uma discussão sobre o risco real de alucinação — ver Iterações): antes de escrever qualquer documento, cada um dos ~40 pontos de discussão da reunião foi listado, citado literalmente e classificado como decisão arquitetural, requisito, detalhe técnico, descartado ou adiado. Essa tabela intermediária (fora de `docs/`, não faz parte do entregável) virou a base de todos os documentos seguintes, reduzindo a chance de a IA "preencher lacunas" na hora de redigir.
3. **ADRs primeiro** — as 6 decisões centrais da reunião (outbox, worker, retry/DLQ, HMAC, at-least-once, reuso de padrões) mais uma decisão complementar (payload snapshot), cada uma virando um ADR MADR completo.
4. **RFC**, consolidando os ADRs em nível de arquitetura, com alternativas descartadas e questões em aberto.
5. **FDD**, o documento mais técnico, com fluxos, contratos HTTP, matriz de erros e a seção obrigatória de integração com o código.
6. **PRD**, produzido por último entre os documentos grandes, consolidando RFC + FDD + ADRs em linguagem de produto.
7. **Tracker**, montado ao final, varrendo os quatro documentos anteriores.
8. **Auditoria final**: um agente Opus separado revisou o pacote inteiro de uma vez, verificando literalmente cada checkbox dos critérios de aceite do enunciado e procurando contradições cruzadas entre documentos que uma verificação documento-por-documento não pegaria.
9. Este README, por último.

Cada rascunho foi seguido imediatamente por uma chamada a um agente de verificação antes de seguir para o próximo documento — nunca dois documentos foram escritos sem o anterior estar auditado.

## Prompts customizados

Dois exemplos representativos dos prompts usados para acionar os agentes de verificação (adaptados/resumidos aqui; os prompts reais incluíam o texto completo dos documentos e trechos maiores de contexto).

**1. Verificação de um documento recém-escrito contra a fonte** (usado, com variações, para os 7 ADRs, o RFC, o FDD e o PRD):

```
Contexto: estamos produzindo um pacote de design docs para uma feature de
Webhooks de Notificação de Pedidos, a partir da transcrição de uma reunião
técnica e do código de um OMS existente. Requisito central: NADA pode ser
inventado — toda afirmação precisa ser rastreável à transcrição (com
timestamp) ou ao código (com caminho de arquivo real).

Acabei de escrever [DOCUMENTO]. Sua tarefa é ser o verificador cético final:
reler a transcrição inteira e o documento, e apontar qualquer afirmação sem
lastro real na fonte.

Leia por completo (não use resumos):
1. TRANSCRICAO.md
2. [documento a verificar]
3. [documentos relacionados já produzidos, para checar consistência cruzada]
4. [arquivos de código citados no documento, para conferir que existem e que
   a descrição do que fazem é factualmente correta]

Para CADA timestamp/citação/caminho de código no documento, verifique:
- o timestamp existe e a fala sustenta o que foi escrito;
- a atribuição ao falante está correta;
- toda "alternativa considerada" foi realmente discutida e descartada, não
  inventada para preencher a seção;
- nenhuma "consequência" é uma inferência sem relação com o que foi decidido;
- o caminho de arquivo existe e o comportamento descrito bate com o código real.

IMPORTANTE: você é só o verificador. NÃO edite nenhum arquivo. Apenas relate
os problemas encontrados, um por um, com a correção sugerida.
```

**2. Auditoria final holística contra os critérios de aceite literais do enunciado:**

```
Sua tarefa agora é ser o auditor final e holístico da entrega inteira. As
verificações anteriores focaram documento por documento; você deve focar em:
1. Conformidade literal com os critérios de aceite do enunciado (colados
   abaixo, verbatim) — item por item, ATENDE / NÃO ATENDE / PARCIAL.
2. Consistência cruzada entre os documentos — uma decisão em um ADR que
   contradiga o RFC ou o FDD; um número (SLA, timeout, contagem de tentativas)
   que apareça diferente em documentos diferentes.
3. Estrutura de arquivos e nomenclatura exigida pelo enunciado.
4. Qualidade geral: redundância desnecessária entre RFC e FDD, seções
   genéricas demais.

--- CRITÉRIOS DE ACEITE DO ENUNCIADO (verbatim) ---
[checklist completo colado aqui, seção por seção]
--- FIM DOS CRITÉRIOS ---

Formato do relatório: uma tabela marcando cada checkbox como ATENDE / NÃO
ATENDE / PARCIAL com evidência; uma seção de inconsistências cruzadas com os
IDs exatos de cada lado da contradição; um veredito final com pendências em
ordem de prioridade.
```

O segundo prompt foi o que rendeu o achado mais importante de todo o processo: na leitura *literal* do critério "Localização = `[hh:mm] Nome`", o Tracker tinha só 41,5% das linhas nesse formato exato — abaixo do mínimo de 70% — mesmo com 83% das linhas vindo da transcrição. O conteúdo estava certo; o formato, não. Sem pedir explicitamente essa checagem literal, esse problema teria passado.

## Iterações e ajustes

O processo levou bem mais que uma geração por documento. Por artefato: 1 rascunho + pelo menos 1 rodada de verificação + correções, e ao final uma rodada extra de auditoria cruzada — no total, mais de 10 ciclos de geração/crítica/correção ao longo do pacote inteiro. Os momentos mais relevantes em que a IA errou e precisou de correção:

- **Afirmação factualmente falsa sobre o próprio código**: o primeiro rascunho do ADR de reuso de padrões afirmou que "todas as tabelas do schema usam UUID". Falso — uma tabela (`OrderNumberSequence`) usa um inteiro fixo por ser um contador singleton. A verificação pegou isso comparando a frase contra o `schema.prisma` real.
- **Alternativas inventadas para preencher seção**: mais de um ADR listava uma "alternativa considerada" que a própria verificação apontou como nunca discutida na reunião (ex.: "rotação de secret sem grace period" no ADR de HMAC) — sintoma clássico de a IA preencher uma seção obrigatória com algo plausível em vez de admitir que só havia uma alternativa real.
- **Métrica de objetivo inventada e dois prazos confundidos**: o primeiro rascunho do PRD media sucesso por "webhook ativo cadastrado" — algo nunca discutido — e tratava a ameaça de churn da Atlas ("fim do trimestre") como sinônimo do prazo real pedido depois na mesma reunião ("fim de novembro"). Duas datas diferentes, ditas em momentos diferentes, por motivos diferentes.
- **Contradição interna dentro do próprio FDD**: uma seção dizia que um evento com falha "volta a `FAILED`"; outra seção do mesmo documento dizia que ele "volta a `PENDING`" — e a consulta do worker só lia `PENDING`. Um evento em falha nunca seria reprocessado. Essa contradição sobreviveu a três rodadas de verificação documento-por-documento e só foi pega na auditoria final, que lê o documento inteiro de uma vez em busca de inconsistência interna, não só de fidelidade a citações pontuais.
- **Mecanismo de erro que o código real nunca produziria**: o FDD atribuía o código `WEBHOOK_INVALID_URL` a uma validação que, pelo próprio desenho do FDD, é tratada pelo Zod e sempre vira o código genérico `VALIDATION_ERROR` — ou seja, um código de erro que a implementação, seguindo a própria especificação, jamais emitiria.
- **Formato do Tracker "conteudisticamente certo, formalmente errado"**: como descrito acima, a coluna Localização usava intervalos de tempo e múltiplos falantes (`[09:31]-[09:33] Marcos/Bruno`) em vez do formato estrito de um timestamp e um falante exigido pelo enunciado — corrigido normalizando cada uma das ~170 linhas de origem TRANSCRICAO para o formato exato, movendo nuances de atribuição para a coluna de conteúdo.

## Como navegar a entrega

Ordem sugerida de leitura (do mais alto nível para o mais detalhado):

1. [`docs/PRD.md`](docs/PRD.md) — o quê e por quê: problema, público, objetivos, escopo.
2. [`docs/RFC.md`](docs/RFC.md) — a proposta técnica em nível de arquitetura, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — as 7 decisões arquiteturais isoladas (ver [índice](docs/adrs/README.md)), cada uma com contexto, alternativas e consequências.
4. [`docs/FDD.md`](docs/FDD.md) — a especificação de implementação: fluxos, contratos HTTP, matriz de erros, integração com o código existente.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — a rastreabilidade de cada item acima até a transcrição ou o código; use como referência cruzada sempre que quiser confirmar de onde veio uma afirmação.
6. [`TRANSCRICAO.md`](TRANSCRICAO.md) — a fonte primária, para quem quiser conferir qualquer citação contra o original.
