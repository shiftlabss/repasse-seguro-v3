# S9 — Supervisão IA

## Repasse Seguro — Módulo Admin

| Campo | Valor |
|---|---|
| **Sprint** | S9 |
| **Nome** | Supervisão IA |
| **Template** | B — Módulo Fullstack |
| **REQs cobertos** | REQ-012, REQ-013, REQ-021, REQ-144, REQ-145, REQ-146, REQ-147, REQ-148, REQ-149, REQ-150, REQ-151, REQ-152, REQ-153, REQ-154, REQ-155, REQ-156, REQ-157, REQ-225, REQ-226, REQ-227 |
| **Docs fonte** | D01.4 §2 (RN-093 a RN-100), D06 §2.8 (T-070 a T-072), D19 (Criação de Agentes de IA) |
| **Itens totais** | 46 |

---

> **Objetivo da sprint:** Implementar os 2 agentes de IA (Guardião do Retorno + Analista de Oportunidades), o painel de Supervisão IA (T-070 a T-072), os thresholds de confiança configuráveis, o takeover manual/automático, o Human Gate de aprovação, os 6 gatilhos de falha crítica (RN-096), o modo Supervisão Total (RN-099), a revisão diária obrigatória (RN-097), e o log de decisões com polling 10s via Supabase Realtime (RN-093).

---

## FEATURE 1 — Infraestrutura Base dos Agentes

### Banco de Dados

**[BE-01] Validar schema `ai_agent_decisions` e estados de agente**
- Confirmar tabela `ai_agent_decisions` com colunas: `id` (UUID PK), `agent_name` (ENUM `GUARDIAO_DO_RETORNO | ANALISTA_DE_OPORTUNIDADES`), `case_id` (UUID FK nullable), `decision_type`, `decision_data` (JSONB), `confidence_score` (DECIMAL 0–100), `outcome` (ENUM `AGUARDANDO_CONFIRMACAO | APROVADO_OPERADOR | REJEITADO_OPERADOR | ACAO_EXECUTADA | FALHA_CRITICA | LOG_ONLY`), `operator_feedback`, `reviewed_by` (UUID FK nullable), `reviewed_at`, `prompt_version`, `latency_ms`, `estimated_cost_brl`, `created_at`
- Confirmar índices: `INDEX(case_id, created_at DESC)`, `INDEX(agent_name, outcome)`, `INDEX(outcome)` para queries de alertas
- Confirmar chaves Redis para estado de agente: `rs:agent:takeover:{agent_name}:{case_id}` (JSON `{ operator_id, reason, started_at }`, sem TTL automático) e `rs:agent:takeover_count:{operator_id}` (INTEGER, sem TTL automático)
- Confirmar Redis para circuit breaker: `rs:agent:circuit:{tool_name}` (TTL 60s)
- Verificação: `prisma migrate deploy` sem erros; `rs:agent:takeover_count` começa em 0 para todos os operadores

### Backend — Core

**[BE-02] Implementar `AgentBaseService` — ciclo de execução e componentes obrigatórios**
- Ciclo de estados: `IDLE → CONTEXT_LOADING → PROMPT_BUILD → LLM_CALL → SCHEMA_VALIDATION → CONFIDENCE_CHECK → [HUMAN_GATE?] → TOOL_EXECUTION → AUDIT → IDLE`
- **Context Loader:** `getCaseContext(case_id)` — SELECT case + `case_config_snapshot` + `case_status_history` últimas 5 entradas (Prisma); leitura de `global_configs` via Redis cache (`rs:configs:all`, TTL 5min)
- **LLM Client:** Claude 3.5 Haiku via API Anthropic; temperatura `0.1`; max tokens: Guardião 512 / Analista 1024; output mode `json_object`; timeout 10s; retry 1× após 2s
- **Schema Validator:** Zod parse do output LLM; falha de parse → `DADOS_CORROMPIDOS` → takeover automático
- **Confidence Evaluator:** comparar `confidence_score` com thresholds carregados de `global_configs`
- **Human Gate:** `INSERT ai_agent_decisions` com `outcome = 'AGUARDANDO_CONFIRMACAO'`; Supabase Realtime emite evento; se 4h sem resposta → alerta ao Coordenador
- **Audit Logger (Pino):** registrar em cada etapa: `{ agent, case_id, step, decision_type, confidence, outcome, latency_ms, estimated_cost_brl, prompt_version }`
- **Failure Handler:** detectar os 6 eventos e acionar takeover automático (BE-04)
- Verificação: executar agente mock em teste → log contém todas as etapas do ciclo; falha de schema → takeover ativado

**[BE-03] Configurar exchange RabbitMQ `ai` e jobs de execução periódica**
- Exchange: `rs.ai`, tipo `topic`
- Eventos que disparam agentes: `ai.case.status_changed` (ambos os agentes), `ai.cessionario.new_profile` (Analista de Oportunidades)
- Job periódico Guardião do Retorno: cron `*/15 * * * *` (a cada 15 minutos) para casos com status `OFERTA_ATIVA` ou `EM_NEGOCIACAO`
- Concorrência máxima: 20 execuções simultâneas por agente; extras aguardam slot via RabbitMQ queue
- Verificação: transição de status publica em `rs.ai` com routing key `ai.case.status_changed`; job cron executa a cada 15min confirmado em logs

---

## FEATURE 2 — Guardião do Retorno

### Backend

**[BE-04] Implementar `GuardiaoDoRetornoAgent`**
- Gatilhos: evento `ai.case.status_changed` OR cron `*/15 * * * *` para casos em `OFERTA_ATIVA`/`EM_NEGOCIACAO`
- **Fronteiras de autonomia (RN-095.a REQ-149):**
  - NUNCA autônomo: escalonamento de cenário (D→C/C→B/B→A), bloqueio por inadimplência, alteração de valores financeiros
  - Confiança ≥ 90%: atualizar status do caso (ações não restritas) autonomamente + registrar
  - Confiança 70–89%: age + notifica Analista; Analista confirma em até 4h ou reverte
  - Confiança 50–69%: sugere + enfileira para decisão humana (Human Gate)
  - Confiança < 50%: apenas log, sem notificação
- Schema de saída Zod `GuardiaoOutputSchema`: `{ action: enum('SUGGEST_ESCALATION'|'SEND_ALERT'|'NO_ACTION'|'WAIT_HUMAN'), confidence_score: number(0-100), reasoning: string(min:10, max:500), suggested_new_scenario: enum('A'|'B'|'C'|'D').optional(), urgency: enum('LOW'|'MEDIUM'|'HIGH'|'CRITICAL'), requires_human_approval: boolean }`
- Prompts armazenados em `apps/api/src/modules/ai-supervision/prompts/guardiao.v1.prompt.ts`; versão ativa via `GlobalConfig` chave `AI_GUARDIAO_PROMPT_VERSION`; cada `AiAgentDecision` registra `prompt_version`
- Verificação: caso em OFERTA_ATIVA há 29 dias → Guardião sugere escalonamento com `requires_human_approval = true`; `action = 'SUGGEST_ESCALATION'` nunca gera atualização autônoma

**[BE-05] Implementar os 6 gatilhos de takeover automático (RN-096 REQ-153)**
- Detectar e acionar `triggerAutoTakeover(case_id, agent_name, failure_type)` para:
  1. `LOOP_DETECTADO`: mesma ação executada 3+ vezes em 60s no mesmo caso (verificar Redis counter `rs:agent:loop:{agent}:{case_id}` TTL 60s)
  2. `TIMEOUT_ACAO`: LLM não respondeu em 10s após 1 retry (2s intervalo)
  3. `ERRO_SISTEMA`: API LLM retornou erro 5xx
  4. `DADOS_CORROMPIDOS`: output fora do schema Zod (campos ausentes ou `confidence_score` fora de 0–100)
  5. `CONFLITO_VERSAO`: Prisma `P2034` ao atualizar caso
  6. `VIOLACAO_REGRA`: ação proposta contradiz regra de negócio explícita (ex: tentar escalonar autonomamente)
- Ao disparar: setar `rs:agent:takeover:{agent_name}:{case_id}` (JSON + sem TTL); registrar `outcome = 'FALHA_CRITICA'` em `ai_agent_decisions`; notificar Coordenador e Master via `NotificationService.emit()`; registrar em `audit.audit_logs`
- Agente não se reativa sozinho — apenas operador humano pode liberar via `PATCH /v1/ai/takeover/:id/release`
- Verificação: injetar LOOP_DETECTADO artificialmente → Redis counter chega a 3 → takeover ativo; agente não executa mais ações para o caso até liberação manual

---

## FEATURE 3 — Analista de Oportunidades

### Backend

**[BE-06] Implementar `AnalistaDeOportunidadesAgent`**
- Gatilhos: caso avança para `OFERTA_ATIVA` (evento `ai.case.status_changed`) OR novo Cessionário cadastrado com perfil compatível (evento `ai.cessionario.new_profile`)
- **Fronteiras de autonomia (RN-095.a REQ-150):**
  - NUNCA: contatar Cessionário diretamente, confirmar aceite de proposta, enviar notificação sem aprovação do Analista
  - Confiança ≥ 85%: `action = 'SUGGEST_MATCH'` → envia sugestão de match ao Analista (aprovação obrigatória)
  - Confiança 60–84%: registra oportunidade como "Em análise" → Analista avalia em até 24h
  - Confiança < 60%: `action = 'LOG_ONLY'` → registra sem notificar
- Tool `getCessionarioProfiles(case_id, scenario)`: retorna até 5 perfis de Cessionários ordenados por score
- Schema de saída Zod `AnalistaOutputSchema`: `{ action: enum('SUGGEST_MATCH'|'LOG_ONLY'|'NO_MATCH'), confidence_score: number(0-100), matches: array({ cessionario_id: UUID, match_score: number(0-100), reasoning: string(min:10, max:300) }).max(5), requires_human_approval: literal(true) }`
- `requires_human_approval` é sempre `true` (agente nunca age diretamente)
- Prompts em `apps/api/src/modules/ai-supervision/prompts/analista.v1.prompt.ts`; versão via `AI_ANALISTA_PROMPT_VERSION`
- Verificação: caso avança para OFERTA_ATIVA → Analista de Oportunidades executa → `ai_agent_decisions` registra resultado com `requires_human_approval = true`

---

## FEATURE 4 — Thresholds, Takeover Manual e Modo Supervisão Total

### Backend

**[BE-07] Implementar API de gestão de agentes**
- `GET /v1/ai/status`: role `COORDENADOR`; retorna status dos 2 agentes: `{ agent_name, status (ATIVO|PAUSADO_MANUAL|PAUSADO_AUTO|AGUARDANDO_CONFIRMACAO|SUPERVISAO_TOTAL), active_takeovers_count, metrics_24h: { decisions_count, avg_confidence, success_rate } }`; dados de Redis + `ai_agent_decisions` últimas 24h; atualização Supabase Realtime → polling ≤10s no frontend (RN-093)
- `POST /v1/ai/takeover`: role `MASTER`; body: `{ agent_name, case_id?, justification: string (min 20 chars) }`;
  - Verificar `rs:agent:takeover_count:{masterId}` < 5; se ≥ 5 → `422` com message `"Você já tem 5 takeovers ativos."` (RN-095 REQ-146)
  - Setar Redis `rs:agent:takeover:{agent_name}:{case_id}` (sem TTL); incrementar contador; registrar em `audit.audit_logs`
- `PATCH /v1/ai/takeover/:id/release`: role `MASTER`; libera takeover; decrementar contador Redis; agente retorna a `ATIVO`
- `PATCH /v1/ai/decisions/:id/review`: role `COORDENADOR`; body: `{ outcome: 'APROVADO_OPERADOR'|'REJEITADO_OPERADOR', feedback: string }`; `reviewed_by`, `reviewed_at` preenchidos; se APROVADO_OPERADOR → tool executor executa ação enfileirada
- Verificação: criar 5 takeovers com MASTER → 6° retorna 422; liberar 1 → 6° agora aceito

**[BE-08] Implementar thresholds configuráveis (RN-095.a REQ-151) e Modo Supervisão Total (RN-099 REQ-156)**
- Thresholds configuráveis via `global_configs`: chaves `AI_GUARDIAO_HIGH_THRESHOLD` (default 90), `AI_GUARDIAO_MEDIUM_THRESHOLD` (default 70), `AI_ANALISTA_HIGH_THRESHOLD` (default 85), `AI_ANALISTA_MEDIUM_THRESHOLD` (default 60); faixa válida 50–95%
- Alteração de threshold: role `MASTER`; registrar em `audit.audit_logs` com `{ old_value, new_value, master_id, justificativa }` — histórico imutável (REQ-157 RN-100)
- Modo Supervisão Total: chave `AI_SUPERVISION_TOTAL_MODE` (boolean, default false); quando `true`: TODAS as ações de ambos os agentes vão para `AGUARDANDO_CONFIRMACAO` antes de executar
  - 4h sem confirmação → alerta ao Coordenador
  - 8h → escala ao Master
  - 12h → descarta ação + flag `"Ação pendente manual"` no `ai_agent_decisions`
- Cache Redis de `global_configs` invalida ao alterar (`rs:configs:all`); novos thresholds entram em vigor imediatamente
- Verificação: alterar `AI_GUARDIAO_HIGH_THRESHOLD` para 80 → agente passa a agir autonomamente com confiança 80+; histórico registra old=90, new=80; Modo Supervisão Total ON → ação com confiança 95% aguarda confirmação

**[BE-09] Implementar alertas de governança (RN-097 REQ-154, RN-098 REQ-155, RN-152 REQ-152)**
- **Revisão diária obrigatória (RN-097):** cron `0 9 * * *` (9h diariamente); verificar se algum Coordenador ou Master acessou `/supervisao-ia` nas últimas 24h (via `audit.audit_logs` `action = 'AI_PANEL_VIEWED'`); se nenhum → alerta ao Master; 48h sem acesso → escalação crítica; 72h sem acesso → incidente de governança (log separado `audit.incidents`)
- **Escalação automática 48h (RN-098):** alertas em `ai_agent_decisions` com `outcome = 'AGUARDANDO_CONFIRMACAO'` há > 48h → emitir notificação com badge "Escalado automaticamente" ao Master
- **Alerta de degradação (RN-152):** cron `0 8 * * *` (8h diariamente); calcular taxa de reversão de decisões nos últimos 7 dias; se > 30% → notificar Master via in-app e e-mail
- Verificação: simular 48h sem acesso ao painel → alerta ao Master gerado; taxa de reversão > 30% → notificação enviada

---

## FEATURE 5 — Circuit Breaker e Limites de Uso LLM

### Backend

**[BE-10] Implementar circuit breaker para tools e limites de uso LLM**
- Circuit breaker por tool: após 5 falhas em 60s na mesma tool → abrir circuito por 300s; Redis key `rs:agent:circuit:{tool_name}` (TTL 60s); ao abrir: `logger.warn` + `audit.audit_logs` `action = 'CIRCUIT_BREAKER_OPEN'`
- Tratamento de erros por tool (D19 §4.2):
  - Timeout > 5s banco: retry com backoff 100ms, 500ms, 2s → falha crítica tipo `TIMEOUT_ACAO` após 3 tentativas
  - `P2034` Prisma: `CONFLITO_VERSAO` → takeover automático
  - Output fora do domínio: `DADOS_CORROMPIDOS` → takeover automático
  - API externa indisponível (Celcoin, ZapSign): não bloqueia agente; registra tentativa como pendente
- Limites de uso:
  - Tokens input por execução: max 4.096 tokens; truncar histórico de ações mais antigas se exceder
  - Custo estimado por execução: alert em log quando execução exceder 3× a média (limiar ~R$0,15)
  - Execuções simultâneas: max 20 por agente (RabbitMQ)
  - Timeout LLM: 10s; retry 1× após 2s; após 2 falhas → `TIMEOUT_ACAO` → takeover
- Verificação: forçar 5 falhas na tool `getCaseContext` em 60s → circuit breaker abre; tool não chamada por 300s

---

## FEATURE 6 — Versionamento de Prompts e Testes de Regressão

### Backend

**[BE-11] Implementar versionamento de prompts e testes de regressão**
- Estrutura de arquivos: `apps/api/src/modules/ai-supervision/prompts/guardiao.v1.prompt.ts` e `analista.v1.prompt.ts`
- Toda alteração de prompt gera nova versão de arquivo (ex: `guardiao.v2.prompt.ts`) — NUNCA sobrescrever arquivo existente
- Versão ativa via `GlobalConfig`: `AI_GUARDIAO_PROMPT_VERSION` e `AI_ANALISTA_PROMPT_VERSION`
- Estrutura obrigatória de cada prompt: 4 blocos `[SYSTEM]`, `[CONTEXT]`, `[INSTRUCTIONS]`, `[OUTPUT FORMAT]` com campos: `agent_name`, `prompt_version`, `current_date`, `objective`, `allowed_actions`, `forbidden_actions`, `high_threshold`, `medium_threshold`, `case_id`, `current_status`, `scenario`, `recent_history` (últimas 5 ações), `financial_params`, `output_schema_json`
- Testes de regressão: 20 casos fixos por agente (10 cada) em `apps/api/src/modules/ai-supervision/prompts/__tests__/`; critério: saída correta em ≥ 90% dos casos; executar via `pnpm --filter api test:ai-prompts`; obrigatório antes de qualquer deploy que altere `prompt_version`
- Verificação: `pnpm --filter api test:ai-prompts` passa com ≥ 18/20 casos corretos por agente

---

## FEATURE 7 — Frontend Supervisão IA (T-070, T-071, T-072)

### Frontend

**[FE-01] Implementar T-070 — Supervisão IA Painel Principal (`/supervisao-ia`)**
- Rota: `/supervisao-ia`; módulo visível na sidebar apenas para COORDENADOR e MASTER (oculto para ANALISTA e GESTOR_FINANCEIRO — DEC-002)
- Registrar `audit.audit_logs` `action = 'AI_PANEL_VIEWED'` ao acessar a rota (para RN-097 revisão diária)
- Layout: grid 2 colunas (1 card por agente); abaixo: tabela de log de decisões recentes (últimas 20 entradas unificadas)
- **Card por agente** (2 cards: Guardião do Retorno e Analista de Oportunidades):
  - Nome do agente + badge de status: verde "Ativo" / vermelho "Takeover Manual" (com nome do operador) / laranja "Pausado Auto"
  - Métricas 24h: decisões executadas, taxa de acerto (%), confiança média (%)
  - Indicadores de confiança (RN-093): ≥80% verde, 60–79% amarelo, <60% vermelho
  - Ação "Ver log completo" → T-071; ação "Takeover manual" → T-072 (apenas Master)
- Refresh automático: Supabase Realtime subscription em `ai_agent_decisions` → polling ≤10s no painel (RN-093)
- Responsividade: Desktop 2 colunas; Mobile 1 coluna empilhada
- Estado loading: skeleton nos cards durante carregamento inicial
- Verificação: painel atualiza automaticamente em ≤10s após nova `ai_agent_decisions`; card com takeover ativo exibe badge vermelho + nome do operador

**[FE-02] Implementar T-071 — Log de Decisões do Agente (`/supervisao-ia/:agente/log`)**
- Rota: `/supervisao-ia/:agente/log` onde `:agente` = `guardiao-do-retorno` ou `analista-de-oportunidades`
- Tabela paginada (25 por página) com colunas: timestamp, tipo de decisão (`decision_type`), caso relacionado (link clicável para case drawer), confiança (% com badge colorido conforme indicador ≥80/60-79/<60), resultado (`outcome` badge), feedback do operador (quando houver)
- Filtros: período (date range picker), tipo de decisão, faixa de confiança (select: Alto ≥80% / Médio 60-79% / Baixo <60%)
- Ação por linha: "Revisar" → drawer lateral com detalhes completos: `decision_data` (JSONB formatado), `reasoning` do LLM, `prompt_version`, `latency_ms`, `estimated_cost_brl`, histórico de revisões do operador
- Botão "Exportar CSV": `GET /v1/ai/decisions/export?agent_name=...&filters=...`
- Componente `TimelineLog` (C-05) reutilizado para histórico de revisões no drawer
- Verificação: filtro por confiança < 60% mostra apenas decisões de baixa confiança; "Exportar CSV" gera download; drawer exibe `decision_data` e `reasoning` completos

**[FE-03] Implementar T-072 — Modal Takeover Manual de Agente**
- Trigger: botão "Takeover manual" em T-070 (apenas Master)
- Modal exibe: nome do agente, lista de casos em que o agente está ativo atualmente (com status de cada caso), contador de takeovers ativos do operador ("X de 5 slots utilizados")
- Campo justificativa obrigatório (mínimo 20 chars) com contador de caracteres
- Toggle: "Pausar agente para este caso" (se `case_id` selecionado) OU "Pausar agente globalmente"
- Botão "Confirmar takeover": `--destructive`; desabilitado até justificativa ≥ 20 chars; ao confirmar → `POST /v1/ai/takeover`
- Se retornar 422 ("5 slots cheios"): exibir erro inline "Você já tem 5 takeovers ativos. Libere um antes de continuar."
- Após confirmação: badge do agente em T-070 atualiza para "Takeover Manual" + nome do operador; toast "Takeover ativado."
- Componente `ConfirmModal` (C-07) com variante `--destructive`
- Verificação: justificativa < 20 chars → botão desabilitado; 5 takeovers ativos → 6° exibe erro inline

**[FE-04] Implementar aba de alertas e revisão de decisões pendentes**
- Aba "Alertas" no painel `/supervisao-ia` com lista de `ai_agent_decisions` com `outcome = 'AGUARDANDO_CONFIRMACAO'`
- Cada alerta exibe: agente, caso (link), tipo de decisão, confidence_score (badge colorido), tempo aguardando, ação proposta
- Badge "Escalado automaticamente" (vermelho) para decisões aguardando > 48h (RN-098)
- Ações por alerta: "Aprovar" e "Rejeitar" (ambas exigem justificativa ≥ 10 chars); disparam `PATCH /v1/ai/decisions/:id/review`
- Após aprovar/rejeitar: item sai da lista imediatamente; toast de feedback
- Contagem de alertas pendentes: badge numérico na aba "Alertas" + badge no menu "Supervisão IA" da sidebar
- Verificação: aprovar decisão → `outcome` muda para `APROVADO_OPERADOR`; badge de alertas decrementa; item sai da lista

---

## WIRING

**[WIRE-01] Integrar `AiSupervisionModule` no NestJS**
- `AiSupervisionModule` exporta `GuardiaoDoRetornoAgent`, `AnalistaDeOportunidadesAgent`, `AgentBaseService`
- Importado em `CasesModule` (injetar `GuardiaoDoRetornoAgent` no listener de status change) e `UsersModule` (injetar `AnalistaDeOportunidadesAgent` no listener de novo Cessionário)
- `AiDecisionsController` expõe: `GET /v1/ai/status`, `POST /v1/ai/takeover`, `PATCH /v1/ai/takeover/:id/release`, `PATCH /v1/ai/decisions/:id/review`, `GET /v1/ai/decisions/export`
- `ScheduleModule.forRoot()` presente para jobs periódicos (cron 15min, diário 9h, diário 8h)
- Verificação: `AppModule` importa `AiSupervisionModule`; transição de status em `CasesService` dispara `GuardiaoDoRetornoAgent.execute()`

**[WIRE-02] Integrar frontend com React Query e Supabase Realtime para Supervisão IA**
- `useAiStatus()`: `useQuery` para `GET /v1/ai/status` com polling fallback 10s + Supabase Realtime subscription
- `useAiDecisions(agentName)`: `useQuery` para listagem paginada com filtros
- `useTakeover()` mutation: após sucesso → invalidar `useAiStatus()` e mostrar toast
- `useReviewDecision(id)` mutation: após sucesso → invalidar `useAiDecisions()` e badge de alertas
- Sidebar "Supervisão IA" oculto para ANALISTA e GESTOR_FINANCEIRO (verificar role no `useAuth()`)
- Verificação: loop Supabase → React Query invalida cache; badge de alertas atualiza ao receber nova decisão pendente

---

## TESTES

**[TEST-01] Testes unitários — `AgentBaseService` e agentes**
- `GuardiaoDoRetornoAgent.execute()`: mock LLM retorna `action = 'SUGGEST_ESCALATION'` com `confidence = 95%` → `requires_human_approval = true` → Human Gate ativado (NUNCA escalonamento autônomo — REQ-149)
- `GuardiaoDoRetornoAgent.execute()`: mock LLM com `action = 'SEND_ALERT'` e `confidence = 91%` → ação executada autonomamente + audit log registrado
- `AnalistaDeOportunidadesAgent.execute()`: sempre `requires_human_approval = true`; `action = 'SUGGEST_MATCH'` com confiança 86% → Human Gate ativado
- Schema inválido (campo ausente): `DADOS_CORROMPIDOS` → takeover automático; Redis key setada; Coordenador notificado
- Loop detectado (3 ações em 60s): `LOOP_DETECTADO` → takeover automático
- Limite 5 takeovers: `POST /v1/ai/takeover` com `rs:agent:takeover_count = 5` → 422

**[TEST-02] Testes de integração — API de Supervisão IA**
- `GET /v1/ai/status`: ANALISTA → 403; COORDENADOR → 200 com status dos 2 agentes
- `POST /v1/ai/takeover`: COORDENADOR → 403 (apenas Master); MASTER + justificativa válida → 200 + Redis key setada; MASTER + 5 takeovers ativos → 422
- `PATCH /v1/ai/decisions/:id/review`: ANALISTA → 403; COORDENADOR + `outcome = 'APROVADO_OPERADOR'` → 200 + ação executada
- Modo Supervisão Total ON: ação com confiança 95% → `outcome = 'AGUARDANDO_CONFIRMACAO'` (não executada autonomamente)
- Alerta 48h: simular `ai_agent_decisions` com `created_at = NOW() - 49h` e `outcome = 'AGUARDANDO_CONFIRMACAO'` → badge "Escalado automaticamente"

**[TEST-03] Testes de componente — Frontend**
- `T-070 Painel`: cards exibem status correto; polling 10s atualiza métricas; takeover ativo exibe badge vermelho
- `T-072 Modal Takeover`: justificativa < 20 chars → botão desabilitado; 5 slots cheios → erro inline; submit válido → mutation chamada
- `T-071 Log`: filtro por confiança < 60% filtra corretamente; drawer exibe `reasoning` e `decision_data`

**[TEST-04] Testes E2E — fluxo de decisão e takeover**
- Playwright: transição de status de caso para OFERTA_ATIVA → verificar `ai_agent_decisions` criado com `agent_name = 'GUARDIAO_DO_RETORNO'`
- Playwright: login como MASTER → `/supervisao-ia` → "Takeover manual" do Guardião → modal abre → preencher justificativa "Caso crítico requer supervisão manual imediata." → confirmar → badge "Takeover Manual" aparece no card
- Playwright: acessar `/supervisao-ia` → registra `AI_PANEL_VIEWED` em `audit.audit_logs` (para RN-097)

---

## AUTO-VERIFICAÇÃO (12 CHECKS)

- [x] **Check #1 — RBAC completo:** `/supervisao-ia` oculto para ANALISTA e GESTOR_FINANCEIRO; takeover MASTER only; review COORDENADOR+; todos os endpoints têm role mínimo declarado
- [x] **Check #2 — Nomes canônicos:** `ai_agent_decisions`, `GUARDIAO_DO_RETORNO`, `ANALISTA_DE_OPORTUNIDADES`, `AGUARDANDO_CONFIRMACAO | APROVADO_OPERADOR | REJEITADO_OPERADOR | ACAO_EXECUTADA | FALHA_CRITICA | LOG_ONLY`, `LOOP_DETECTADO | TIMEOUT_ACAO | ERRO_SISTEMA | DADOS_CORROMPIDOS | CONFLITO_VERSAO | VIOLACAO_REGRA` — zero sinônimos
- [x] **Check #3 — Fronteiras de autonomia:** Guardião NUNCA escalonamento autônomo (REQ-149); Analista NUNCA contata Cessionário (REQ-150); `requires_human_approval = literal(true)` no schema Analista
- [x] **Check #4 — 6 gatilhos de falha crítica (RN-096 REQ-153):** LOOP_DETECTADO, TIMEOUT_ACAO, ERRO_SISTEMA, DADOS_CORROMPIDOS, CONFLITO_VERSAO, VIOLACAO_REGRA — todos implementados e testados
- [x] **Check #5 — Thresholds por agente (RN-095.a REQ-147/148):** Guardião: ≥90% autônomo, 70–89% age+notifica, 50–69% aguarda, <50% apenas log; Analista: ≥85% sugere, 60–84% análise, <60% log
- [x] **Check #6 — Limite 5 takeovers (RN-095 REQ-146):** Redis counter; 422 no 6°; decremento ao liberar
- [x] **Check #7 — Polling ≤10s (RN-093 REQ-144):** Supabase Realtime + fallback polling 10s; log atualiza em ≤10s
- [x] **Check #8 — Modo Supervisão Total (RN-099 REQ-156):** 4h→alerta Coordenador, 8h→escala Master, 12h→descarta+flag
- [x] **Check #9 — Revisão diária obrigatória (RN-097 REQ-154):** cron 9h; 24h→alerta Master; 48h→crítico; 72h→incidente
- [x] **Check #10 — Versionamento de prompts:** arquivos `{agent}.v{N}.prompt.ts`; nunca sobrescrever; 20 casos de teste ≥90% aprovação; `pnpm --filter api test:ai-prompts`
- [x] **Check #11 — Anti-scaffold R10:** todos os itens têm sub-itens com thresholds numéricos exatos, casos de erro, guardrails, verificação e estado Redis
- [x] **Check #12 — Cobertura REQs S9:** REQ-012 ✅ REQ-013 ✅ REQ-021 ✅ REQ-144 ✅ REQ-145 ✅ REQ-146 ✅ REQ-147 ✅ REQ-148 ✅ REQ-149 ✅ REQ-150 ✅ REQ-151 ✅ REQ-152 ✅ REQ-153 ✅ REQ-154 ✅ REQ-155 ✅ REQ-156 ✅ REQ-157 ✅ REQ-225 ✅ REQ-226 ✅ REQ-227 ✅

---

*Sprint S9 gerada por Pipeline ShiftLabs v2.3 — 2026-03-24*
