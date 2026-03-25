# S9 — Observabilidade, Qualidade e Fallback

| Campo                | Valor                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S9                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| **Nome**             | Observabilidade, Qualidade e Fallback                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| **Tipo**             | Módulo Fullstack (Template B)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Template**         | B (Módulo Fullstack: por Feature, vertical slice Banco→Backend→Frontend→Wiring→Testes)                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| **Docs Consultados** | D01 (RN-DCE-022, RN-DCE-023, RN-DCE-024), D02 (stacks), D05.5 (RF-DCE-029 a RF-DCE-033), D17 (§2.5 Langfuse, §2.6 PostHog), D20, D27, D28                                                                                                                                                                                                                                                                                                                                                                                                     |
| **Cross-cutting**    | D10 (Glossário)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **REQs cobertos**    | REQ-110, REQ-111, REQ-112, REQ-113, REQ-114, REQ-115, REQ-116, REQ-117, REQ-118, REQ-119, REQ-120, REQ-121, REQ-122, REQ-123, REQ-124, REQ-125, REQ-126, REQ-127, REQ-128, REQ-129, REQ-130                                                                                                                                                                                                                                                                                                                                                   |
| **Objetivo**         | Langfuse 3.x+ para traces/spans/confidence_score, PostHog 9 eventos obrigatórios, Pino 9.x+ structured logs (correlation_id, `console.log` proibido em produção), Sentry `@sentry/nestjs` 9.x+ (≥500 errors), circuit breaker (10%/30% em 15min janela deslizante Redis), kill switch PostHog `dani_cedente_enabled`, `POST /admin/fallback/enable`, takeover threshold configurável (padrão 80%), latência p95 ≤ 5s, 8 edge cases (RF-DCE-033), cobertura 80%/90% gates, Langfuse evals (≥50 exemplos por fluxo), checklist de segurança D28 |

---

## Critério de Conclusão da S9

Ao final desta sprint:

- Toda chamada ao OpenAI API tem trace Langfuse com `session_id`, `cedente_id` (hash), `confidence_score`, `latency_ms`
- `confidence_score < 0.80` → sessão sinalizada para takeover no painel Admin
- Circuit breaker: janela deslizante 15min, Redis key `dani:error_count:{window}`; > 10% erros → alerta Admin; > 30% → `dani_cedente_enabled = false` automaticamente
- `POST /admin/fallback/enable` reativa o agente (role `admin` obrigatório)
- `GET /health/ready` e `GET /health/live` operacionais
- 9 eventos PostHog capturados com IDs hasheados (sem UUID/PII raw)
- `console.log` ausente em produção (lint rule + CI check)
- `correlation_id` presente em 100% dos logs estruturados Pino
- Cobertura ≥80% geral, ≥90% em `agent`, `auth`, `escrow`, `fallback`
- ≥50 exemplos por fluxo carregados no Langfuse para evals antes de Go-Live
- Todos os 8 edge cases de RF-DCE-033 cobertos por testes automatizados

---

## ⚙️ FEATURE 1 — Langfuse (LLM Observability)

### Backend — LangfuseService

- [x] **Implementar `LangfuseService`** em `src/modules/agent/langfuse.service.ts`: (1) inicializa cliente `Langfuse` com `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST` (env var opcional para self-hosted); (2) método `startTrace(sessionId: string, cedenteId: string, entryPoint: string): LangfuseTrace` — cria trace com `session_id` e `cedente_id = sha256(cedenteId)`; (3) método `logGeneration(trace, input, output, model, usage, latencyMs)` — cria span com `tokens_prompt`, `tokens_completion`, `latency_ms`, `model: 'gpt-4-turbo'`; (4) método `logToolCall(trace, toolName, input, output, latencyMs)` — span por ferramenta chamada; (5) método `logRagChunks(trace, chunks, k)` — span com `rag_chunks_used: k`, `chunks_retrieved: N`; (6) método `scoreConfidence(trace, score: number)` — `Score` com `name: 'confidence_score'`, `value: score` (0.0–1.0); (7) SDK usa batch flush a cada 500ms — nunca bloqueante; (8) se Langfuse offline: SDK fail-silent, log warning, `confidence_score` fica `null` no `ChatMessage`
  - Validação: `cedente_id` sempre enviado como SHA-256, nunca UUID raw; trace criado por mensagem processada; span por ferramenta por chamada ao agente; `confidence_score` de 0.0–1.0; falha Langfuse → log warning, fluxo de chat não interrompido

- [x] **Integrar `LangfuseService` no `DaniCedenteAgent`** (S3): (1) `startTrace()` no início de cada `processMessage()`; (2) `logGeneration()` após retorno do GPT-4; (3) `logToolCall()` em cada tool execution; (4) `logRagChunks()` após `RagService.search()`; (5) `scoreConfidence()` após resposta gerada: calcular `confidence_score` como heurística: `1.0` se `finish_reason = 'stop'`; `0.7` se `finish_reason = 'tool_calls'`; `0.4` se `finish_reason = 'length'`; `0.0` se erro; (6) atualizar `ChatMessage.confidence_score` com o valor calculado
  - Validação: toda mensagem processada tem trace no Langfuse; `confidence_score` presente no `ChatMessage` após processamento; `confidence_score = null` se Langfuse falhar; `finish_reason = 'stop'` → `confidence_score = 1.0`

- [x] **Implementar sinalização de takeover** em `DaniCedenteAgent`: (1) se `confidence_score < 0.80` → chamar `AdminAlertService.flagForTakeover(sessionId, cedenteId, confidence_score)`; (2) `AdminAlertService` persiste `{ session_id, flagged_at, confidence_score, reviewed: false }` na tabela `takeover_flags`; (3) threshold configurável via `CONFIDENCE_THRESHOLD` env var (padrão: 0.80, mínimo: 0.50, máximo: 0.95); (4) Admin consulta sessões sinalizadas via `GET /admin/sessions/flagged` (endpoint do módulo Admin — registrar como dependência)
  - Validação: `confidence_score = 0.79` → flag criada; `confidence_score = 0.80` → sem flag; threshold configurável via env var; tabela `takeover_flags` com campos corretos

- [x] **Preparar dataset de evals no Langfuse**: (1) criar dataset com nome `dani-cedente-evals` no Langfuse via `LangfuseClient.createDataset()`; (2) carregar ≥50 exemplos por fluxo crítico: análise de proposta, simulação de retorno, status do dossiê, status do Escrow, cenários A/B/C/D; (3) cada exemplo: `{ input: mensagem do Cedente, expected_output: resposta esperada, metadata: { flow_type, confidence_expected } }`; (4) script `npm run langfuse:seed-evals` em `package.json`
  - Validação: `npm run langfuse:seed-evals` executa sem erro; Langfuse UI mostra dataset com ≥50 exemplos; cada exemplo tem `input`, `expected_output`, `metadata.flow_type`

---

## ⚙️ FEATURE 2 — Circuit Breaker e Kill Switch

### Backend — FallbackService

- [x] **Implementar `FallbackService`** em `src/modules/fallback/fallback.service.ts`: (1) método `recordError(sessionId: string)`: incrementa `dani:error_count:{window}` no Redis com janela deslizante de 15 minutos (TTL = 900s); (2) método `getErrorRate(): number`: calcula `error_count / total_requests` na janela atual; (3) método `checkThresholds()`: se `error_rate > 0.10` → chamar `alertAdmin('DCE-FALLBACK-ALERT-10PCT')`; se `error_rate > 0.30` → chamar `disableAgent()`; (4) método `disableAgent()`: (a) atualiza flag PostHog `dani_cedente_enabled = false` via PostHog API; (b) persiste `dani:agent_disabled = true` no Redis (TTL sem limite — só reativado manualmente); (c) log `level: error` com `error_rate`, `action: 'circuit_open'`; (d) Sentry event `circuit_breaker_opened`; (5) método `isEnabled(): boolean`: verifica cache Redis `dani:agent_disabled`; fallback: se Redis offline → retorna `true` (fail-open)
  - Validação: 11% de erros em 15min → alerta Admin; 31% → `disableAgent()` chamado; Redis `dani:agent_disabled = true` persistido; `isEnabled()` com Redis offline → `true` (fail-open); taxa calculada corretamente

- [x] **Implementar `FallbackGuard`** em `src/modules/fallback/fallback.guard.ts`: (1) aplica-se a todos os endpoints de chat (`/api/v1/chat/*`); (2) chama `FallbackService.isEnabled()`; (3) se `false` → retorna HTTP 503 com `{ error: { code: 'DCE-CHAT-5030_001', message: 'A Dani está temporariamente indisponível. Para urgências, entre em contato com o suporte.', user_message: 'A Dani está temporariamente indisponível. Para urgências, entre em contato com o suporte.' } }`; (4) SSE endpoint: emite evento `{ type: 'unavailable' }` antes de fechar a conexão
  - Validação: agente desabilitado → 503 em todos os endpoints `/chat/*`; SSE emite `type: 'unavailable'` antes de fechar; `user_message` com texto exato conforme D01 seção 12; endpoints de `/auth/*` e `/health/*` não afetados pelo guard

- [x] **Implementar `POST /api/v1/admin/fallback/enable`** em `AdminController`: (1) requer role `admin` (RbacGuard com `@RequireRole('admin')`); (2) remove `dani:agent_disabled` do Redis; (3) atualiza flag PostHog `dani_cedente_enabled = true` via PostHog API; (4) loga reativação com `admin_id`, `timestamp`, `previous_error_rate`; (5) retorna `{ data: { enabled: true, reactivated_at, reactivated_by } }`
  - Validação: token `cedente` → 403; token `admin` → 200 com agente reativado; `dani:agent_disabled` removido do Redis; log de auditoria com `admin_id`

---

## ⚙️ FEATURE 3 — Logs Estruturados (Pino)

### Backend — LoggingInterceptor

- [x] **Verificar `LoggingInterceptor`** já scaffoldado em S1 e garantir que está completo em `src/common/interceptors/logging.interceptor.ts`: (1) todo request logar: `{ level: 'info', correlation_id, method, path, status_code, duration_ms, cedente_id: hash(cedenteId) }`; (2) todo erro ≥ 500 logar: `{ level: 'error', correlation_id, method, path, status_code, error_message, stack }`; (3) `correlation_id` gerado por request via `AsyncLocalStorage` — propagado para todos os logs do mesmo request; (4) `console.log` proibido em produção — ESLint rule `no-console` ativo em `eslint.config.js`; (5) Pino configurado com `level: process.env.LOG_LEVEL || 'info'`
  - Validação: `console.log('test')` em arquivo `.ts` falha no lint; todo log contém `correlation_id`; log de erro tem `stack`; `cedente_id` como hash (sem UUID raw); duração em `duration_ms`

- [x] **Garantir que `PiiMaskingMiddleware`** (S2) está aplicado **antes** do `LoggingInterceptor`: (1) ordem no `app.module.ts`: `PiiMaskingMiddleware` primeiro, depois `LoggingInterceptor`; (2) CPF, email, telefone, `Authorization` header → `[REDACTED]` em todos os logs; (3) `cessionario_nome` → `[Comprador]` em logs de contexto do agente
  - Validação: log com corpo `{ cpf: "12345678900" }` exibe `[REDACTED]`; `Authorization` header → `[REDACTED]`; ordem correta de middlewares verificada no `app.module.ts`

---

## ⚙️ FEATURE 4 — Sentry

### Backend — Sentry

- [x] **Configurar `@sentry/nestjs` 9.x+** em `src/main.ts`: (1) `Sentry.init({ dsn: process.env.SENTRY_DSN, environment: process.env.NODE_ENV, release: process.env.APP_VERSION })` chamado antes de qualquer import do NestJS; (2) `SentryModule.forRoot()` em `app.module.ts`; (3) `SentryGlobalFilter` aplicado globalmente como exception filter; (4) captura automática de erros HTTP ≥ 500; (5) **não captura** erros HTTP 4XX (erros de negócio esperados); (6) `beforeSend` hook: mascarar `user.email`, `request.headers.authorization`, qualquer campo com `cpf` ou `senha` antes de enviar ao Sentry
  - Validação: erro 500 → Sentry recebe evento; erro 401 → Sentry **não** recebe evento; `authorization` header mascarado no Sentry; `APP_VERSION` visível no Sentry como release

- [x] **Implementar captura manual de eventos Sentry críticos**: (1) circuit breaker open → `Sentry.captureException(new Error('circuit_breaker_opened'), { extra: { error_rate, threshold: 0.30 } })`; (2) DLQ overflow → `Sentry.captureException(new Error('notification_dlq_overflow'), { extra: { dlq_count } })`; (3) CSAT < 3.5 → `Sentry.captureMessage('csat_below_threshold', { level: 'warning', extra: { avg_csat, window: '24h' } })`; (4) upload Supabase Storage falha após 3 retries → `Sentry.captureException(error, { extra: { document_type, opportunity_id } })`
  - Validação: cada evento Sentry tem `extra` com campos corretos; nenhum evento Sentry contém PII raw; `captureMessage` para warnings, `captureException` para erros

---

## ⚙️ FEATURE 5 — PostHog (Analytics e Kill Switch)

### Backend — PostHogService

- [x] **Implementar `PostHogService`** em `src/common/services/posthog.service.ts`: (1) inicializa `PostHog` com `POSTHOG_API_KEY` e `POSTHOG_HOST` (opcional); (2) SDK retry: 3 tentativas, backoff 1s/2s/4s; timeout 3s; fail-silent; (3) método `capture(eventName: string, properties: Record<string, unknown>)`: todos IDs hasheados — `cedente_id = sha256(id)`, `session_id = sha256(id)`; nunca UUIDs raw nem PII; (4) método `isFeatureEnabled(flagName: string): boolean`: verifica `dani_cedente_enabled` com cache Redis TTL 60s; se Redis offline → fail-open (`true`); (5) implementa os 9 eventos obrigatórios
  - Validação: `capture` não bloqueia o fluxo (fire-and-forget); `isFeatureEnabled` com Redis offline → `true`; IDs hasheados — teste verifica que UUID não aparece no payload PostHog

- [x] **Capturar os 9 eventos PostHog obrigatórios** em seus respectivos módulos: (1) `chat_session_started` (S3 ChatService) → `{ entry_point, kyc_status, has_opportunity }`; (2) `message_sent` (S3 ChatService) → `{ session_id: hash, response_latency_ms }`; (3) `proposal_accepted` (S6 ProposalService) → `{ opportunity_id: hash }`; (4) `proposal_rejected` (S6 ProposalService) → `{ opportunity_id: hash }`; (5) `csat_submitted` (S8 ChatController) → `{ score, session_id: hash }`; (6) `agent_fallback_triggered` (S9 FallbackService) → `{ error_rate, threshold }`; (7) `dossier_status_viewed` (S5 DossierController — evento PostHog adicional) → `{ completion_percentage }`; (8) `scenario_viewed` (S4 OpportunityController) → `{ opportunity_id: hash }`; (9) `simulation_requested` (S4 SimulationController) → `{ opportunity_id: hash }`
  - Validação: 9 eventos listados implementados em seus módulos; nenhum evento contém UUID raw ou PII; verificar via `jest.spyOn(PostHogService.prototype, 'capture')` nos testes de cada módulo

---

## ⚙️ FEATURE 6 — Qualidade e Cobertura de Testes

### Backend — Configuração de Cobertura

- [x] **Verificar que `jest.config.ts`** já configurado no S1 tem thresholds corretos: (1) `coverageThreshold.global`: `{ branches: 80, functions: 80, lines: 80, statements: 80 }`; (2) `coverageThreshold['src/modules/agent/**']`: `{ branches: 90, functions: 90, lines: 90, statements: 90 }`; (3) `coverageThreshold['src/modules/auth/**']`: 90%; (4) `coverageThreshold['src/modules/escrow/**']`: 90%; (5) `coverageThreshold['src/modules/fallback/**']`: 90%; (6) CI pipeline falha se thresholds não atingidos
  - Validação: `npm run test:cov` falha se agente com 89% de cobertura; CI pipeline (`ci.yml`) inclui step `test:cov`; relatório de cobertura publicado como artefato do CI

- [x] **Garantir que os 8 edge cases de RF-DCE-033 têm testes automatizados**: (1) Cedente tenta ver identidade do Cessionário → agente retorna mensagem de privacidade exata de RN-DCE-002; (2) Cedente recebe proposta abaixo do esperado → agente analisa retorno e oferece contraproposta; (3) Cedente solicita alterar cenário com oportunidade publicada → agente informa que requer suporte; (4) prazo de Escrow vence sem depósito → Cedente notificado e proposta cancelada; (5) Cedente pede à Dani para aceitar proposta por ele → agente recusa com mensagem de orientação; (6) documento do dossiê rejeitado → agente informa motivo e orienta correção; (7) Cedente pergunta quanto o Cessionário pagou de comissão → agente informa valor da comissão (dado público) mas não dados pessoais; (8) Cedente quer retirar oportunidade Em negociação → agente informa que precisa recusar proposta ativa primeiro
  - Validação: 8 testes em `src/modules/agent/agent.edge-cases.spec.ts`; cada teste verifica resposta do agente com LLM mockado; mensagens de recusa verificadas por igualdade de string ou regex

- [x] **Criar checklist de segurança pré-Go-Live** em `src/docs/security-checklist.md` com verificações de D28: (1) `SUPABASE_JWT_SECRET` não exposta em nenhum arquivo versionado; (2) `OPENAI_API_KEY` não exposta; (3) `ZAPSIGN_WEBHOOK_SECRET` não exposta; (4) RLS ativo em todas as 8 tabelas de domínio; (5) `console.log` ausente no código de produção; (6) HMAC ZapSign implementado com `crypto.timingSafeEqual()`; (7) PII masking em logs e contexto LLM verificado; (8) rate limiting de chat (30/h) e notificações (30/h webchat, 10/h email) configurados; (9) `Content-Security-Policy` header configurado no chat widget; (10) dependências auditadas (`npm audit --audit-level=moderate` no CI)
  - Validação: arquivo criado com 10 itens verificáveis; CI executa `npm audit --audit-level=moderate` e falha se vulnerabilidades moderadas ou críticas encontradas

---

## ⚙️ FEATURE 7 — Health Checks

### Backend — HealthController

- [x] **Implementar `GET /api/v1/health/live`** em `HealthController`: (1) retorna HTTP 200 com `{ status: 'ok', timestamp: ISO8601 }` se o serviço está rodando; (2) sem dependências externas (apenas verifica que o processo responde); (3) usado pelo Railway para liveness probe
  - Validação: responde 200 em < 50ms; sem acesso ao banco ou Redis; formato `{ status: 'ok', timestamp: '...' }`

- [x] **Implementar `GET /api/v1/health/ready`** em `HealthController`: (1) verifica dependências obrigatórias: (a) Prisma: `this.prisma.$queryRaw('SELECT 1')` com timeout 2s; (b) Redis: `this.redis.ping()` com timeout 1s; (c) RabbitMQ: verifica conexão ativa via `amqplib.connection.serverProperties`; (2) retorna 200 se todas OK: `{ status: 'ok', checks: { database: 'ok', redis: 'ok', rabbitmq: 'ok' } }`; (3) retorna 503 se qualquer dependência falhar: `{ status: 'error', checks: { database: 'ok', redis: 'error', rabbitmq: 'ok' } }`
  - Validação: Redis offline → 503 com `checks.redis: 'error'`; todas OK → 200 em < 3s; Railway readiness probe usa este endpoint

---

## 🖥️ FRONTEND (Widget Chat — Fallback)

### Componente FallbackMessage

- [x] **Implementar exibição de mensagem de fallback** no ChatWidget quando SSE emite `{ type: 'unavailable' }`: (1) exibe banner na área de chat: "A Dani está temporariamente indisponível. Para urgências, entre em contato com o suporte."; (2) campo de input desabilitado; (3) botão "Contato com suporte" com deep link para canal de suporte da plataforma; (4) banner persiste até reconexão bem-sucedida (ou reload da página); (5) não exibe toasts de notificação enquanto em estado de fallback
  - Validação: texto exato conforme D01 seção 12 e D05.5 RF-DCE-029; input desabilitado; botão de suporte visível; estado recuperado ao reconectar

### Componente LatencyIndicator

- [x] **Implementar tratamento de latência acima do SLA** no ChatWidget: (1) se `response_latency_ms > 5000` (5s desde envio): exibe `TypingIndicator` (3 pontos pulsando — componente de S3); (2) se `response_latency_ms > 10000` (10s): substitui TypingIndicator por banner: "A Dani está demorando mais que o esperado. Você pode aguardar ou tentar novamente em instantes." com botões "Aguardar" e "Tentar novamente"; (3) "Tentar novamente" reenvia a última mensagem; (4) timer iniciado no `message_sent` event
  - Validação: timer de 5s dispara TypingIndicator; timer de 10s exibe banner com 2 botões; "Tentar novamente" reenvia a mensagem; timers limpos ao receber resposta

---

## 🔀 Cross-Módulo

- `FallbackGuard` aplicado globalmente em `app.module.ts` — impacta todos os endpoints de chat de S3; deve ser aplicado após `JwtAuthGuard` (não bloqueia endpoints de auth)
- `PostHogService.isFeatureEnabled('dani_cedente_enabled')` consultado em `FallbackGuard` — cache Redis compartilha pool com outros módulos (prefixo `dani:`)
- `LangfuseService` injetado em `DaniCedenteAgent` de S3 — `AgentModule` importa `LangfuseModule`
- `AdminAlertService.flagForTakeover()` requer tabela `takeover_flags` — criar migration nesta sprint
- Cobertura de testes aciona gates nos módulos de S3 (agent ≥90%), S2 (auth ≥90%), S7 (escrow ≥90%) e S9 (fallback ≥90%)

---

## 🧪 TESTES

- [x] **Criar `src/modules/fallback/fallback.service.spec.ts`** com testes: (1) `recordError` incrementa contador Redis; (2) `getErrorRate` retorna taxa correta; (3) 10.1% erros → `alertAdmin` chamado; (4) 30.1% erros → `disableAgent` chamado; (5) `isEnabled` com Redis offline → `true` (fail-open); (6) `isEnabled` com `dani:agent_disabled = true` no Redis → `false`
  - Validação: 6 cenários; mock do Redis; mock do PostHog API; sem chamadas reais

- [x] **Criar `src/modules/agent/agent.edge-cases.spec.ts`** com os 8 edge cases de RF-DCE-033: respostas do agente verificadas com LLM mockado (jest.fn retornando texto predefinido); cada edge case tem `// RF-DCE-033: Cenário N` como comentário
  - Validação: 8 testes passam; nenhum acessa LLM real; respostas de privacidade verificadas contra strings exatas de RN-DCE-002

- [x] **Criar `test/health.e2e-spec.ts`** com Supertest: (1) `GET /health/live` → 200; (2) `GET /health/ready` com todas dependências OK → 200; (3) `GET /health/ready` com Redis simulado offline → 503 com `checks.redis: 'error'`
  - Validação: 3 cenários; mock de Redis offline via jest; formato de response verificado

- [x] **Criar `test/fallback.e2e-spec.ts`** com Supertest: (1) `POST /chat/messages` com `dani:agent_disabled = true` no Redis → 503 `DCE-CHAT-5030_001`; (2) `POST /admin/fallback/enable` com token admin → 200 + `dani:agent_disabled` removido; (3) `POST /admin/fallback/enable` com token cedente → 403
  - Validação: 3 cenários; Redis mock com estado desabilitado; role admin verificado

---

## 🔍 AUTO-VERIFICAÇÃO S9

| Check                | Critério                                                                                                                                                                          | Status |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ------- | --- |
| #1 Nomenclatura      | `LangfuseService`, `FallbackService`, `FallbackGuard`, `PostHogService`, `HealthController` — nomes exatos; tabela `takeover_flags` criada conforme especificação                 | [x]    |
| #2 Valores numéricos | Circuit breaker: 10%/30% em 15min; janela deslizante Redis; takeover threshold: 80% padrão, min 50%, max 95%; kill switch cache Redis TTL 60s; `confidence_score` range 0.0–1.0   | [x]    |
| #3 Langfuse          | Trace por mensagem com `cedente_id = sha256(id)`; span por tool call; `confidence_score` por resposta; ≥50 exemplos no dataset de evals; fail-silent se Langfuse offline          | [x]    |
| #4 Circuit Breaker   | `dani:error_count:{window}` com TTL 900s (15min); > 10% → alerta Admin; > 30% → `disableAgent()` + PostHog flag + Redis flag; reativação APENAS via `POST /admin/fallback/enable` | [x]    |
| #5 PostHog           | 9 eventos obrigatórios capturados; IDs hasheados (SHA-256); fail-silent; `dani_cedente_enabled` flag com cache Redis TTL 60s; fail-open se Redis offline                          | [x]    |
| #6 Pino              | `console.log` proibido em produção (ESLint rule); `correlation_id` em 100% dos logs; PII mascarado antes de logs; `level: process.env.LOG_LEVEL                                   |        | 'info'` | [x] |
| #7 Sentry            | `@sentry/nestjs` 9.x+; captura ≥500 HTTP errors; não captura 4XX; PII mascarado no `beforeSend`; 4 eventos críticos com `captureException`/`captureMessage`                       | [x]    |
| #8 Health            | `GET /health/live` sem dependências (resposta < 50ms); `GET /health/ready` com 3 checks (DB/Redis/RabbitMQ); 503 se qualquer check falhar                                         | [x]    |
| #9 Cobertura         | 80% geral; 90% em `agent`, `auth`, `escrow`, `fallback`; CI falha se thresholds não atingidos; relatório publicado como artefato                                                  | [x]    |
| #10 Edge Cases       | 8 edge cases de RF-DCE-033 com testes automatizados; LLM mockado; respostas verificadas contra strings de RN-DCE-002                                                              | [x]    |
| #11 Anti-scaffold    | `FallbackService` com janela deslizante real no Redis; `LangfuseService` com integração real ao SDK; `PostHogService` com cache Redis real; sem stubs que retornam valores fixos  | [x]    |
| #12 Cobertura REQs   | REQ-110 a REQ-130 — todos com ≥1 item no checklist                                                                                                                                | [x]    |
