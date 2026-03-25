# Pipeline Fase 4 -- Desenvolvimento — ShiftLabs v1.0

## Framework de Execucao de Sprints via Claude Code Desktop

---

| Campo | Valor |
| --- | --- |
| **Destinatario** | Claude Code Desktop (aba Code) — execucao de sprint checklists |
| **Escopo** | Prompt autossuficiente para execucao de sprints tarefa por tarefa no Claude Code Desktop, com `CLAUDE.md` persistente, Skill reutilizavel e gestao de contexto |
| **Carater** | **Normativo.** Este pipeline define os templates, prompts, configuracoes e estrategias para execucao de sprints no Claude Code Desktop. |
| **Versao** | v1.0 |
| **Responsavel** | Fernando Calado |
| **Data** | 22/03/2026 (America/Fortaleza) |
| **Pre-requisitos** | Fase 3 (Sprints) concluida — sprint checklists gerados e auditados (A03 aprovada) |
| **Ambiente de execucao** | Claude Code Desktop (aba Code) + repositorio Git local |

---

## Fluxo do Pipeline

```
Fase 1           Fase 2              Fase 3         Fase 4
Produto   ->     Desenvolvimento ->  Sprints   ->   Desenvolvimento/Coding
                 (Docs)                              VOCE ESTA AQUI
```

---

## TL;DR

- **Fase 4 do pipeline.** Prompt autossuficiente otimizado para o **Claude Code Desktop** (aba Code). Instrui o Claude Code a executar sprint checklists gerados pela Fase 3 (`pipeline-fase-3.md`).
- O agente **le toda a documentacao na ordem** (`docs/01 - Produto/` -> `docs/02 - Desenvolvimento/` -> `docs/03 - Sprints/`), absorve o contexto completo, e implementa tarefa por tarefa usando o checklist como fonte da verdade.
- **`CLAUDE.md` como instrucao persistente** — lido automaticamente no inicio de cada sessao, sem precisar colar no prompt.
- **Skill `/execute-sprint`** — workflow reutilizavel com 5 etapas internas: leitura -> planejamento -> execucao -> criterios de conclusao -> relatorio.
- Marca `[x]` nas tarefas concluidas e documenta com `BLOCKED` as que nao conseguir realizar.
- **Progresso via TodoWrite** — use a ferramenta TodoWrite para registrar e acompanhar o progresso de cada tarefa/bloco durante a execucao da sprint.
- **Fluxo completo do pipeline: Fase 1 (Produto) -> Fase 2 (Desenvolvimento Docs) -> Fase 3 (Sprints) -> Fase 4 (Desenvolvimento/Coding — este doc).**

---

## Quick Start

1. Copie o template `CLAUDE.md` -> raiz do projeto
2. Copie `.claude/settings.json` -> pasta `.claude/`
3. Copie a Skill -> `.claude/skills/execute-sprint/SKILL.md`
4. Abra o **Claude Code Desktop** -> aba **Code** -> selecione a pasta do projeto
5. Cole o **Prompt Principal** e rode

---

## Diferencas vs. Abordagem Generica

> Este prompt foi **redesenhado do zero** para explorar as capacidades nativas do **Claude Code Desktop**:
>
> - **Sessoes locais com worktrees** — cada sessao recebe uma copia isolada do projeto via Git worktrees (`~/.claude-worktrees/`), permitindo trabalho paralelo sem conflito
> - **Modos de permissao nativos** — Ask (revisao de cada mudanca), Code (auto-aceitar edits), Plan (read-only) e Act (autonomia total)
> - **Diff view integrado** — revisao visual de mudancas linha a linha, com comentarios inline antes de commitar
> - **`CLAUDE.md` como instrucao persistente** — o Claude Code le automaticamente no inicio de cada sessao, sem precisar colar no prompt
> - **Connectors nativos** — GitHub, Slack, Linear, Notion e outros via MCP (configurar via "..." -> "Connectors" antes de iniciar a sessao)
> - **Skills e Subagentes** — workflows reutilizaveis em `.claude/skills/` e agentes especializados em `.claude/agents/`
> - **Hooks automaticos** — scripts que rodam em resposta a eventos (SessionStart, apos edits, antes de commits)
> - **Checkpoints e Rewind** — cada edit cria um snapshot reversivel; `Esc+Esc` abre menu de rewind
> - **Sessoes remotas** — para tarefas longas, rodar na nuvem Anthropic e acompanhar pelo app
> - **Extended Thinking** — habilitado por padrao, melhora raciocinio em tarefas complexas

---

## Pre-requisitos

> **Setup obrigatorio antes da primeira sessao:**
>
> 1. **Claude Code Desktop** instalado e autenticado (aba Code)
> 2. **Repositorio Git local** com a estrutura de pastas abaixo
> 3. **`CLAUDE.md`** na raiz do projeto (template fornecido neste doc)
> 4. **Variaveis de ambiente** configuradas no shell profile (`~/.zshrc` ou `~/.bashrc`) — o Desktop herda do shell
> 5. **Git** instalado (obrigatorio para isolamento de sessoes via worktrees)
> 6. **(Opcional)** Arquivo `.worktreeinclude` na raiz do repo listando patterns de arquivos em `.gitignore` a copiar para worktrees (ex: `.env`, `.env.local`)

### Estrutura obrigatoria de pastas

```
projeto/
├── CLAUDE.md                        <- instrucoes persistentes (template abaixo)
├── .claude/
│   ├── settings.json                <- permissoes e hooks
│   ├── skills/                      <- skills reutilizaveis
│   │   └── execute-sprint/
│   │       └── SKILL.md
│   └── agents/                      <- subagentes especializados (opcional)
│       └── code-reviewer.md
├── docs/
│   ├── 01 - Produto/                <- docs de produto (.md) — leitura 1a
│   │   ├── doc-01-pesquisa-de-mercado-e-benchmark.md
│   │   ├── doc-02-analise-geral.md
│   │   └── ... (16 docs)
│   ├── 02 - Desenvolvimento/        <- docs tecnicos (.md) — leitura 2a
│   │   ├── doc-01-regras-de-negocio.md
│   │   ├── 02 - Stacks.md
│   │   └── ... (29 docs)
│   └── 03 - Sprints/                <- sprint checklists (.md) — leitura 3a + execucao
│       ├── s1-fundacao.md
│       ├── s2-auth.md
│       ├── s3-dashboard.md           <- sprints organizadas por MODULO (fullstack), nao por camada tecnica
│       └── ...
├── src/
└── ...
```

---

## `CLAUDE.md` — Template para a Raiz do Projeto

> O `CLAUDE.md` e lido **automaticamente** pelo Claude Code Desktop no inicio de cada sessao. Coloque-o na raiz do repositorio e faca commit para compartilhar com o time. Este arquivo **substitui** a necessidade de colar a persona e regras no prompt — elas ficam persistentes.
>
> **Hierarquia:** O Claude Code tambem le `~/.claude/CLAUDE.md` (nivel usuario). Instrucoes globais vao la; instrucoes de projeto ficam na raiz do repo. Em caso de conflito, o project-level prevalece.

**IMPORTANTE:** Use o arquivo `claude-fase-4.md` como base para o `CLAUDE.md` do projeto. Copie-o para a raiz do projeto com `cp claude-fase-4.md CLAUDE.md`.

---

## `.claude/settings.json` — Configuracao Recomendada

```json
{
  "permissions": {
    "defaultMode": "code"
  },
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/scripts/setup-dev.sh"
          }
        ]
      }
    ]
  }
}
```

> **Modo `code` como default**: auto-aceita edits de arquivos mas pede confirmacao para comandos shell. Ideal para execucao de sprints — velocidade nos edits, seguranca nos comandos.
>
> **SessionStart Hook**: roda `scripts/setup-dev.sh` ao iniciar qualquer sessao, garantindo que dependencias estejam instaladas.

---

## Skill: Execute Sprint

Crie o arquivo `.claude/skills/execute-sprint/SKILL.md`:

```markdown
---
name: execute-sprint
description: Executa uma sprint completa lendo docs na ordem e implementando tarefa por tarefa
---

Execute a sprint especificada em $ARGUMENTS seguindo este fluxo:

## Etapa 1 — Leitura da Documentacao (obrigatoria)

1. Leia TODOS os arquivos .md de @docs/01 - Produto/
2. Leia TODOS os arquivos .md de @docs/02 - Desenvolvimento/
3. Leia o arquivo .md da sprint em @docs/03 - Sprints/ por COMPLETO
4. Identifique todas as secoes tematicas e seus itens de checklist
5. Leia os Pre-requisitos e Criterios de Conclusao

NAO comece a implementar ate ter lido TUDO.

## Etapa 2 — Planejamento da Execucao

Antes de codar, crie um plano de execucao seguindo esta prioridade:

1. **Infraestrutura** — setup, configs, env vars, Docker
2. **Banco de dados** — migrations, seeds, enums
3. **Backend** — models, services, controllers, middlewares
4. **Rotas/API** — endpoints, validacoes, error handling
5. **Frontend** — componentes, telas, estados, navegacao
6. **Integracao** — conectar frontend <-> backend <-> servicos externos
7. **Testes** — unitarios, integracao, E2E
8. **Qualidade** — a11y, seguranca, observabilidade, documentacao

Mapeie dependencias entre tarefas e agrupe as que podem ser feitas juntas.

**Use TodoWrite para registrar o plano de execucao com todas as tarefas/blocos identificados.**

> **Nota:** As sprints sao organizadas por MODULO (fullstack). Cada sprint
> implementa banco + backend + frontend + testes de um modulo de negocio.
> Organize a execucao por FEATURE dentro do modulo, nao por camada tecnica.

## Etapa 3 — Execucao Tarefa por Tarefa

Para CADA tarefa do checklist:

### 3.1 — Implementar
- Implemente exatamente o que a tarefa descreve
- Use os nomes exatos do dominio (tabelas, campos, endpoints)
- Siga a stack definida nos docs de referencia
- Nao adicione features, libs ou comportamentos nao especificados
- Se a tarefa tem sub-itens, implemente TODOS

### 3.2 — Verificar (fullstack end-to-end)
Antes de marcar, verifique TODAS as camadas:

**Compilacao:**
- O codigo compila/transpila sem erros? (`tsc --noEmit`)
- Lint passa? (`eslint .`)

**Backend:**
- Endpoint: status code e payload corretos?
- Validacoes: input invalido e REJEITADO com erro correto?
- RBAC: role sem permissao recebe 403?
- Regra de negocio: valores numericos exatos?

**Banco:**
- Migration: roda sem erros?
- Seed: dados iniciais inseridos?
- Queries: retornam dados reais (nao hardcoded)?

**Frontend — WIRING (mais importante):**
- Botao: onClick EXECUTA acao real (nao console.log, nao vazio)?
- Form: onSubmit ENVIA dados para a API e RECEBE resposta?
- Dialog: abre quando o trigger e clicado E fecha ao concluir?
- Listagem: FAZ FETCH da API e renderiza dados REAIS?
- Filtro: ENVIA query para API e ATUALIZA lista?
- Paginacao: muda page e REFAZ fetch?
- Loading: APARECE durante fetch (skeleton, nao spinner)?
- Empty state: APARECE quando data.length === 0?
- Error state: APARECE quando API falha?
- Toast: MOSTRA mensagem real ao concluir acao?
- Navegacao: REDIRECIONA para rota que EXISTE?

**Conexao entre camadas:**
- Frontend chama URL que EXISTE no backend?
- Backend retorna EXATAMENTE o shape que o frontend espera?
- Variaveis de ambiente existem no .env?

**Testes:**
- Testes passam? (`npm test`)
- Assertions verificam logica REAL (nao mock)?

**Sub-itens:** TODOS implementados?

**Teste mental de fluxo:** Para cada feature, percorra:
```
Usuario clica → UI reage → API chamada → Backend processa →
Banco persiste → Response volta → UI atualiza → Feedback aparece
```
Se qualquer elo esta quebrado → a feature NAO esta pronta.

### 3.3 — Marcar Conclusao
Se implementada e verificada:
- Marque com [x] no arquivo .md da sprint
- Marque TODOS os sub-itens com [x]
- Salve o arquivo apos cada marcacao
- **Atualize o TodoWrite marcando a tarefa como completed**

### 3.4 — Documentar Falha
Se NAO pode ser implementada, NAO marque [x]. Adicione abaixo da tarefa:

BLOCKED — [Titulo curto do impedimento]
- **Motivo**: [Descricao tecnica precisa]
- **Dependencia**: [O que precisaria existir/mudar]
- **Impacto**: [Quais outras tarefas sao afetadas]
- **Sugestao**: [Acao recomendada]

Categorias de bloqueio:
- DEPS — Dependencia de tarefa anterior nao concluida
- AMBIGUO — Spec insuficiente para implementar com confianca
- INFRA — Falta recurso externo (API key, servico, permissao)
- CONFLITO — Spec conflita com outra spec ou codigo existente
- LIMITACAO — Limitacao tecnica real

### 3.5 — Commitar
Apos cada bloco logico de tarefas concluidas:
- Commit com conventional commits: `tipo(escopo): descricao`
- Commits atomicos: 1 commit = 1 bloco logico coeso
- Inclua o .md atualizado no commit

## Etapa 4 — Criterios de Conclusao da Sprint

Apos processar TODAS as tarefas:

1. Releia os Criterios de Conclusao da sprint
2. Verifique cada criterio — se falha, identifique qual tarefa falta
3. Rode o projeto localmente e confirme funcionamento end-to-end
4. Verifique que TODAS as tarefas estao [x] OU BLOCKED
5. **Verificacao fullstack obrigatoria:**
   - ZERO funcoes com TODO, return null ou placeholder
   - ZERO botoes sem handler funcional
   - ZERO forms sem submit real
   - ZERO listagens com dados hardcoded
   - ZERO dialogs que nao abrem/fecham
   - ZERO filtros que nao filtram
   - ZERO rotas que nao levam a lugar nenhum
   - Toda feature funciona END-TO-END: UI → API → Banco → Response → UI

## Etapa 5 — Relatorio de Execucao

Adicione no FINAL do arquivo .md da sprint:

---
## Relatorio de Execucao
- **Data de execucao**: [data/hora]
- **Tasks concluidas**: [N] de [TOTAL] ([percentual]%)
- **Tarefas bloqueadas**: [N] — [lista com titulos curtos]
- **Commits realizados**: [N]
- **Observacoes**: [Info relevante nao capturada acima]

Se existirem tarefas bloqueadas:

## Resumo de Bloqueios
| # | Tarefa | Categoria | Motivo (resumo) | Impacto |
|---|--------|-----------|-----------------|------------|
| 1 | [Tarefa] | DEPS/AMBIGUO/INFRA/CONFLITO/LIMITACAO | [Resumo] | [Tarefas afetadas] |
```

---

## Prompt Principal — Colar na Sessao

> **Como usar**: Abra o Claude Code Desktop -> aba **Code** -> selecione a pasta do projeto -> modo **Code** (ou **Ask** se quiser revisar cada mudanca) -> cole o prompt abaixo.
>
> O `CLAUDE.md` ja contem a persona, regras e convencoes. O prompt abaixo e **apenas o gatilho de execucao** — enxuto e direto.

### Executar uma sprint especifica

```
Leia toda a documentacao do projeto na ordem antes de comecar:

1. Leia TODOS os arquivos de @docs/01 - Produto/
2. Leia TODOS os arquivos de @docs/02 - Desenvolvimento/
3. Leia o sprint completo em @docs/03 - Sprints/s[N]-[nome].md

Apos absorver TODO o contexto, execute a sprint tarefa por tarefa seguindo
a skill /execute-sprint docs/03 - Sprints/s[N]-[nome].md.

Use TodoWrite para registrar e acompanhar o progresso de cada tarefa/bloco.

REGRAS ABSOLUTAS (reforco):
- O checklist e o contrato — implemente EXATAMENTE o que esta escrito
- Nunca pule tarefas — processe TODAS
- Falhar e melhor que inventar — documente como BLOCKED
- Marque [x] e salve o .md apos CADA tarefa concluida
- Commits atomicos com conventional commits apos cada bloco logico
- Nao modifique texto, ordem ou estrutura das tarefas no checklist
- Atualize TodoWrite a cada tarefa concluida ou bloqueada
```

### Executar multiplas sprints em sequencia

```
Leia toda a documentacao do projeto na ordem antes de comecar:

1. Leia TODOS os arquivos de @docs/01 - Produto/
2. Leia TODOS os arquivos de @docs/02 - Desenvolvimento/

Execute as sprints na ordem, uma por vez, usando /execute-sprint.

Sprints:
- S1: docs/03 - Sprints/s1-fundacao.md
- S2: docs/03 - Sprints/s2-auth.md
- S3: docs/03 - Sprints/s3-[modulo].md  (nome do modulo vem da Fase 3)

Apos concluir cada sprint:
1. Gere o Relatorio de Execucao no .md
2. Confirme que os Criterios de Conclusao foram atendidos
3. NUNCA avance para a proxima sprint com tarefas BLOCKED
4. Use /clear entre sprints para limpar o contexto
5. So entao abra a proxima sprint e repita o ciclo
```

### Retomar uma sprint parcialmente executada

```
Leia toda a documentacao do projeto na ordem:

1. Leia TODOS os arquivos de @docs/01 - Produto/
2. Leia TODOS os arquivos de @docs/02 - Desenvolvimento/
3. Leia o sprint em @docs/03 - Sprints/s[N]-[nome].md

A sprint foi parcialmente executada. Tarefas marcadas com [x] ja foram
implementadas — NAO reimplemente. Continue a partir da primeira tarefa
nao marcada e sem BLOCKED.

Use /execute-sprint docs/03 - Sprints/s[N]-[nome].md para continuar.
```

---

## Estrategias Avancadas para o Desktop

### Gestao de contexto (critico para sprints longas)

> **O contexto e o recurso mais importante do Claude Code.** Performance degrada conforme o contexto enche. Sprints longas exigem gestao ativa.

1. **Use `/clear` entre sprints** — nunca acumule contexto de sprints diferentes na mesma sessao
2. **Use `/compact focus on sprint tarefas` se o contexto encher** — preserva o progresso e descarta lixo
3. **Delegue investigacoes para subagentes** — eles rodam em contexto isolado e retornam resumo:

```
Use um subagente para investigar como o modulo de auth funciona
antes de implementar a tarefa de JWT.
```

4. **Trabalhe secao por secao** — se a sprint e longa, trate cada secao tematica como uma unidade
5. **Apos completar uma secao, peca para compactar** antes de avancar:

```
/compact Preserve as tarefas marcadas com [x] e pendentes. Foque no que falta implementar.
```

### Sessoes paralelas para sprints independentes

O Claude Code Desktop cria **worktrees automaticos** para cada sessao em repos Git:

1. Clique **+ New session** no sidebar
2. Selecione a mesma pasta do projeto
3. Cada sessao recebe uma copia isolada — mudancas nao interferem
4. Ideal para rodar Sprint 3 (Modulo Dashboard) e Sprint 4 (Modulo Clientes) ao mesmo tempo

### Plan Mode para sprints complexas

Antes de executar, use **Plan Mode** para validar a abordagem:

```
Mude para Plan Mode (botao ao lado do send).

Leia toda a documentacao do projeto e a sprint s[N]-[nome].md.
Crie um plano detalhado de execucao com:
- Ordem de implementacao por dependencias
- Tasks que podem ser agrupadas
- Riscos e possiveis bloqueios antecipados
- Estimativa de complexidade por secao
```

Revise o plano, ajuste via comentarios, depois mude para **Code Mode** e execute.

### Diff view para review de qualidade

Apos cada bloco de implementacao:

1. Clique no indicador de diff (ex: `+45 -12`) no toolbar
2. Revise mudancas arquivo por arquivo
3. Clique em linhas especificas para adicionar comentarios
4. O Claude le os comentarios e ajusta automaticamente

### Sessoes remotas para sprints longas

Para sprints que vao demorar horas (ex: fundacao de projeto):

1. Selecione **Remote** ao inves de **Local** ao criar a sessao
2. A sprint roda na nuvem Anthropic mesmo se voce fechar o app
3. Acompanhe o progresso pelo app ou pelo celular (Claude iOS)
4. Ao finalizar, revise o diff e crie o PR

---

## Dicas de Reforco

### Se o Claude inventar features nao especificadas

```
LEMBRE-SE: Implemente APENAS o que esta no checklist. Se o item nao esta
na sprint, ele nao existe. Nao adicione features, campos, endpoints ou
comportamentos que nao estejam explicitamente escritos no checklist.
```

### Se o Claude marcar [x] sem implementar de verdade

```
REGRA: Antes de marcar [x], PROVE que a tarefa foi implementada.
Para endpoints: rode o request e mostre que funciona.
Para migrations: mostre que a tabela foi criada com os campos corretos.
Para telas: mostre que o componente renderiza com os estados corretos.
Se nao consegue provar, nao marque [x].
```

### Se o Claude pular tarefas "dificeis"

```
REGRA ABSOLUTA: Voce DEVE processar TODAS as tarefas. Se nao consegue
implementar, documente como BLOCKED com o motivo. Pular
silenciosamente e PROIBIDO. Cada tarefa deve ter [x] ou BLOCKED.
```

### Se o Claude perder contexto no meio da sprint

```
/compact Preserve a lista de tarefas com status [x] e pendentes.

Releia o checklist atualizado da sprint. Tarefas com [x] ja foram
implementadas — ignore-as. Continue da primeira tarefa sem [x]
e sem BLOCKED.
```

---

## Relacao com os Outros Pipelines

| Pipeline | Arquivo | Papel | Output |
| --- | --- | --- | --- |
| **Fase 1 — Produto** | `pipeline-fase-1.md` | Define o que construir (visao, regras, fluxos) | Docs em `docs/01 - Produto/` |
| **Fase 2 — Desenvolvimento Docs** | `pipeline-fase-2.md` | Define como construir (arquitetura, stack, padroes) | Docs em `docs/02 - Desenvolvimento/` |
| **Fase 3 — Sprints** | `pipeline-fase-3.md` | Transforma specs em checklists executaveis | Checklists em `docs/03 - Sprints/` |
| **Fase 4 — Desenvolvimento (este)** | `pipeline-fase-4.md` | Executa os checklists no Claude Code Desktop | Codigo implementado + `.md` atualizado + commits |

O pipeline da Fase 4 **nunca modifica** a spec nem os docs das pastas `01 - Produto` e `02 - Desenvolvimento`. Se encontrar algo errado no checklist, documenta como `BLOCKED` e segue.

---

## Troubleshooting

| Sintoma | Causa provavel | Solucao |
| --- | --- | --- |
| Claude nao le o `CLAUDE.md` | Arquivo nao esta na raiz ou nao foi commitado | Mova para a raiz do repo e faca `git add CLAUDE.md && git commit` |
| Worktree nao copia `.env` | `.worktreeinclude` ausente ou pattern errado | Crie `.worktreeinclude` na raiz com o pattern `.env*` |
| Contexto estourou no meio da sprint | Sprint longa sem compactacao | `/compact Preserve tarefas [x] e pendentes` — depois continue da primeira tarefa sem [x] |
| Skill `/execute-sprint` nao encontrada | Arquivo fora do path esperado | Verifique: `.claude/skills/execute-sprint/SKILL.md` (case-sensitive) |
| Sessao paralela conflitando | Repo sem Git inicializado | `git init` — worktrees dependem do Git para isolamento |
| Hook `SessionStart` nao roda | `settings.json` fora da pasta `.claude/` | Verifique: `.claude/settings.json` e que o script tem permissao de execucao (`chmod +x`) |

---

## Como Usar

1. **Conclua as Fases 1 a 3** — os docs devem estar em `docs/01 - Produto/`, `docs/02 - Desenvolvimento/` e `docs/03 - Sprints/`, todos auditados e validados
2. **Copie os templates** deste doc para o projeto: `CLAUDE.md`, `.claude/settings.json`, `.claude/skills/execute-sprint/SKILL.md`
3. **Abra o Claude Code Desktop** -> aba Code -> selecione a pasta do projeto
4. **Cole o Prompt Principal** (secao acima) ajustando o nome da sprint
5. O Claude Code le o `CLAUDE.md` automaticamente, executa a Skill, e implementa tarefa por tarefa

---

## Etapa Final — Auditoria A04

Apos implementar TODAS as sprints, execute a auditoria final A04:

1. Leia o prompt completo em `02 - Prompts/03 - Auditorias/A04 - Auditoria Fase 4 - Desenvolvimento.md`
2. Execute os 4 eixos: Codigo↔Docs + Fullstack Tecnica (13 dimensoes) + UI/UX + Registro Mestre vs Implementacao
3. Aplique todas as correcoes no codigo (NUNCA nos docs)
4. Salve relatorio em `docs/05 - Auditorias/AUDIT_FASE4_CODIGO.md`

**GATE FINAL:** Zero findings P0/P1 → PIPELINE SHIFTLABS CONCLUIDO.

Ao aprovar, exiba:

```
==========================================================
  PIPELINE SHIFTLABS CONCLUIDO
==========================================================

  Todas as 4 fases executadas e auditadas:
    Fase 1: Produto (+ Auditoria A01)
    Fase 2: Desenvolvimento Docs (+ Auditoria A02)
    Fase 3: Sprints (+ Auditoria A03)
    Fase 4: Desenvolvimento Codigo (+ Auditoria A04)

  Proximos passos:
    - Revisao manual pelo responsavel
    - Deploy

==========================================================
```

---

## Changelog

| Versao | Data | Alteracoes |
| --- | --- | --- |
| 1.0 | 22/03/2026 | **Reestruturado de Fase 5 para Fase 4** no novo pipeline de 4 fases. Pipeline consolidado: Fase 1 (Produto), Fase 2 (Desenvolvimento Docs), Fase 3 (Sprints), Fase 4 (Desenvolvimento Codigo). Fases de auditoria separadas (antigas Fase 4 e Fase 6) eliminadas — auditorias agora integradas em cada fase (A01-A04). Adicionada Etapa Final com Auditoria A04 e gate de conclusao do pipeline. Handoff para Fase 6 removido. Pre-requisito atualizado para Fase 3 (A03 aprovada). Tabela de pipelines reduzida para 4 fases. Baseado no conteudo original de pipeline-fase-5.md v1.4. |
