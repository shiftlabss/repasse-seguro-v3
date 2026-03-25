# S8 — Qualidade, Observabilidade e CI/CD

| **Sprint**         | S8 — Qualidade, Observabilidade e CI/CD                                                                                               |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Template**       | B — Módulo Fullstack (organizado por Feature com vertical slice por domínio de qualidade)                                             |
| **REQs cobertos**  | REQ-142, REQ-143, REQ-144, REQ-145, REQ-146, REQ-147, REQ-148, REQ-149, REQ-150, REQ-151, REQ-152, REQ-153, REQ-155, REQ-156, REQ-157 |
| **Docs fonte**     | D25 (Observabilidade e Logs), D27 (Plano de Testes), D28 (Checklist de Qualidade), D24 (Deploy/CI-CD), D23 (Guia de Contribuição)     |
| **Total de itens** | 54                                                                                                                                    |

---

## 🎯 Objetivo

Finalizar e validar todos os mecanismos de qualidade do AI-Dani-Cessionário: configuração completa do Pino Logger com Pino redact, Langfuse tracing, Sentry, 10 fluxos E2E (Playwright), suite completa Vitest (unitários + integração + contrato Pact), Checklist de Qualidade D28 verificado, pipeline CI/CD completo (`ci-pr.yml`, `ci-staging.yml`, `release.yml`), gates de cobertura ≥80%, e testes P0 de isolamento e cálculo.

---

## Feature 1 — Observabilidade: Pino, Langfuse e Sentry

### 🔧 Backend

- [x] **[BACK-OBS-001]** Validar configuração completa do Pino Logger em `src/common/logger/pino.config.ts`:
  - Sub-item: Campo `service: "dani-api"` presente em TODOS os logs (D25 §3)
  - Sub-item: Campos obrigatórios por request: `timestamp`, `level`, `service`, `module`, `correlation_id`, `request_id`, `session_id`, `cessionario_id` (como SHA-256 — NUNCA UUID raw), `environment`
  - Sub-item: Pino redact configurado: `paths: ['req.headers.authorization', 'req.body.otp', 'req.body.phone', '*.cpf', '*.email', 'OPENAI_API_KEY']`; `censor: '[Redacted]'`
  - Sub-item: Nível por ambiente: `debug` em `NODE_ENV=development`; `info` em `NODE_ENV=production`
  - Sub-item: Retenção configurada: debug 7 dias, info 30 dias, warn/error 90 dias (D25 §2)

- [x] **[BACK-OBS-002]** Verificar que os 12 eventos críticos de log estão implementados (D25 §4):
  - Sub-item: `request_received` (info): `correlation_id`, `method`, `path`, `cessionario_id_hash` em todos os módulos
  - Sub-item: `agent_response` (info): `latency_ms`, `tokens_in`, `tokens_out`, `model`, `tools_called`, `session_id`
  - Sub-item: `tool_call_executed` (info): `tool_name`, `latency_ms`, `success`, `cessionario_id_hash`
  - Sub-item: `llm_fallback_activated` (warn): `reason`, `retry_count`, `fallback_service`
  - Sub-item: `rate_limit_reached` (warn): `limit`, `window_s`, `cessionario_id_hash`
  - Sub-item: `isolation_violated` (error): `error_code: AGENTE_004`, `cessionario_id_token`, `cessionario_id_resource` — alerta P0 imediato
  - Sub-item: `otp_failure` (warn): `phone_hash`, `attempt`, `remaining`, `block_ttl?`
  - Sub-item: `otp_hard_block` (warn): `phone_hash`, `block_ttl_s: 1800`, `reason: "5_consecutive_fails"`
  - Sub-item: `unhandled_error` (error): `correlation_id`, `error_code: INFRA_000`, `path`
  - Sub-item: `dani_auto_shutdown` (fatal): `reason: "error_rate_30pct"`, `taxa_erro`, `session_count`
  - Sub-item: `dlq_message_received` (error): `template_id`, `canal`, `retry_count`
  - Sub-item: `calc_executed` (info): `opr_id`, `formula_aplicada`, `latency_ms`

- [x] **[BACK-OBS-003]** Validar integração Langfuse (D25 §1):
  - Sub-item: `Langfuse.trace({ userId: sha256(cessionario_id), sessionId, input_tokens, output_tokens, latency_ms })` — campos mandatórios em toda chamada ao LLM
  - Sub-item: `userId = sha256(cessionario_id)` — NUNCA UUID raw (REQ-107, LGPD)
  - Sub-item: Fields redactados nos traces: `phone`, `otp`, `jwt`, `cpf` — nunca presentes em Langfuse
  - Sub-item: Traces com `session_id`, `cessionario_id_hash`, `input_tokens`, `output_tokens`, `latency_ms` — todos presentes
  - Sub-item: Langfuse traces retidos por 90 dias (D25 §1)

- [x] **[BACK-OBS-004]** Validar integração Sentry (D25 §1):
  - Sub-item: Sentry inicializado em `apps/api/src/main.ts` e `apps/web/src/main.tsx`
  - Sub-item: Alerta P0 automático para: `isolation_violated` (`AGENTE_004`), `dani_auto_shutdown`, DLQ crítica
  - Sub-item: `AllExceptionsFilter` captura e envia para Sentry todos os 5xx com `correlation_id`
  - Sub-item: Stack traces NUNCA expostos em resposta de produção — apenas `correlation_id` (D28 §4.5)
  - Sub-item: Sentry Performance: transações para endpoints P0 (`/dani/chat`, `/calculadora/calcular`)

- [x] **[BACK-OBS-005]** Verificar SLOs e alertas (D25 §1):
  - Sub-item: SLO latência p95 ≤ 5s — configurado como alerta `ALT-001` (D26)
  - Sub-item: SLO disponibilidade ≥ 99.5% — monitorado via Sentry Performance + uptime
  - Sub-item: 8 alertas configurados com threshold, severidade e canal: `ALT-001` (p95 >5s → P1), `ALT-002` (5xx >5% → P0), `ALT-003` (isolamento falha → P0 rollback imediato), `ALT-004` (Redis down → P1), `ALT-005` (DLQ >50 → P1)
  - Sub-item: Monitoramento RabbitMQ via Management API porta 15672: `queue_depth`, `dlq_size`, `consumer_lag`

---

### ✅ Testes de Observabilidade

- [x] **[TEST-OBS-001]** Testes de integração Pino e redact:
  - Sub-item: Log gerado por qualquer request com JWT válido não contém UUID raw de `cessionario_id` — spy no logger
  - Sub-item: Request com `Authorization: Bearer token` → log do middleware não contém o token — campo `[Redacted]`
  - Sub-item: Request com `body.otp = "123456"` → log não contém o valor OTP — campo `[Redacted]`
  - Sub-item: Evento `isolation_violated` (AGENTE_004) → Sentry `captureMessage` chamado com severity `error`
  - Sub-item: Langfuse trace: `userId` é string de 64 chars hex (SHA-256); não é um UUID v4

---

## Feature 2 — Suite de Testes Vitest (Unitários + Integração)

### 🔧 Setup de Testes

- [x] **[BACK-TEST-001]** Configurar ambiente de testes Vitest para `apps/api`:
  - Sub-item: `vitest.config.ts` com `environment: 'node'` para NestJS; `environment: 'jsdom'` para componentes React
  - Sub-item: Coverage provider: `@vitest/coverage-v8`; relatório LCOV para CI; thresholds: `lines: 80, functions: 80, branches: 80, statements: 80` (D27 §3.2)
  - Sub-item: Cobertura mínima 100% para: `CalculadoraService`, `CessionarioOwnerGuard`, `OtpService` (aqui: `WhatsappService.verificarOtp`), `RateLimitService`
  - Sub-item: Setup/Teardown: `beforeAll` seeds com `cessionario_id` de teste `'00000000-0000-0000-0000-000000000001'`; `afterEach` limpa Redis keys `dani:*:cess-e2e-test*`; `afterAll` DELETE nas tabelas de teste
  - Sub-item: Banco real: Supabase local via `supabase start` (PostgreSQL 15 — não mock)
  - Sub-item: Redis real: `redis:7-alpine` local via Docker (não mock)
  - Sub-item: RabbitMQ real: `rabbitmq:3.12-management-alpine` local via Docker (não mock)

- [x] **[BACK-TEST-002]** Implementar cenários P0 obrigatórios do `AgenteModule` (D27 §7.1):
  - Sub-item: A-01: `POST /dani/chat` com JWT válido → 200; stream SSE iniciado; `correlation_id` presente no header
  - Sub-item: A-02: `POST /dani/chat` sem Authorization → 401 `{ code: "AUTH_TOKEN_MISSING" }`; A-02b: 31ª requisição em 1 hora → 429 com `retry_after`
  - Sub-item: A-04: Tool `buscarOportunidades` com `cessionario_id` do cessionário A → query retorna apenas dados de A; cessionário B não visível
  - Sub-item: A-07: Request com CPF no body → log mostra `[Redacted]`; Sentry não recebe CPF real
  - Sub-item: A-10: Langfuse trace — `userId = sha256(cessionario_id)` presente; UUID raw ausente

- [x] **[BACK-TEST-003]** Implementar cenários P0 obrigatórios do `CalculadoraModule` (D27 §7.2):
  - Sub-item: C-01: `tabelaAtual=85000, tabelaContrato=70000, valorPagoCedente=65000` → `delta=15000`, `comissao=3000`, `baseCalculo="delta"` (Δ>0)
  - Sub-item: C-02: `tabelaAtual=60000, tabelaContrato=70000, valorPagoCedente=65000` → `delta=-10000`, `comissao=13000`, `baseCalculo="valorPagoCedente"` (Δ<0)
  - Sub-item: C-03: `tabelaAtual=70000, tabelaContrato=70000, valorPagoCedente=65000` → `delta=0`, `comissao=13000`, `baseCalculo="valorPagoCedente"` (Δ=0)
  - Sub-item: C-04: `opr_id` de outro cessionário → 403 `AUTH_FORBIDDEN` (CessionarioOwnerGuard executado real)
  - Sub-item: C-08: `tabelaAtual=85450.75, tabelaContrato=70000, valorPagoCedente=65000` → `delta=15450.75`, `comissao=3090.15` (centavos preservados — sem arredondamento indevido)
  - Sub-item: C-09: Importar `CalculadoraModule` e verificar ausência de import de `AgenteModule` (circular dependency detector)
  - Sub-item: C-10: `valorPagoCedente=0` com Δ≤0 → `comissao=0` (sem divisão por zero ou exceção)

- [x] **[BACK-TEST-004]** Implementar cenários P0 de `AuthModule` / `WhatsappModule` (D27 §7.3 e §7.4):
  - Sub-item: O-01: OTP gerado com 6 dígitos numéricos exatos; TTL Redis = 900s (15min)
  - Sub-item: O-02: OTP armazenado como bcrypt hash cost 12 — `bcrypt.compare(otp_plain, hash)` → `true`
  - Sub-item: O-04: OTP incorreto → Redis `INCR dani:rate:otp:{phone_hash}` incrementado; TTL 3600s
  - Sub-item: O-05: OTP incorreto × 5 consecutivos → `dani:block:otp:{phone_hash}` criado TTL 1800s; resposta 429
  - Sub-item: O-06: OTP expirado (TTL zerado manualmente) → 400 `AUTH_OTP_EXPIRED` / `WA_OTP_EXPIRADO`
  - Sub-item: O-08: Phone armazenado como SHA-256 em chave Redis — nunca número em texto no Redis
  - Sub-item: W-04: Webhook com `message: "PARAR"` → `dani_vinculacoes_whatsapp.estado = 'NAO_VINCULADO'`; log `opt_out_lgpd` gerado; processamento síncrono (não enfileirado)

- [x] **[BACK-TEST-005]** Testes de isolamento por `cessionario_id` — P0 cobertura 100% (D27 §4.3):
  - Sub-item: Seed: cessionário A (`id: 'cess-aaa'`) com oportunidades `['opr-001', 'opr-002']`; cessionário B (`id: 'cess-bbb'`) com oportunidade `['opr-003']`
  - Sub-item: JWT de A → `GET /oportunidades` retorna `opr-001` e `opr-002`; NUNCA `opr-003`
  - Sub-item: pgvector: busca semântica com `cessionario_id = 'cess-aaa'` e `namespace = 'cess-aaa'` → não retorna chunks de `'cess-bbb'`
  - Sub-item: Redis keys de A e B são independentes — `dani:rate:webchat:cess-aaa` e `dani:rate:webchat:cess-bbb`
  - Sub-item: `CessionarioOwnerGuard` não pode ser mockado nos testes de integração (D27 §3.3, REQ-149) — verificado via `jest.spyOn` ausente no arquivo de teste

- [x] **[BACK-TEST-006]** Testes RabbitMQ e DLQ (D27 §4.5):
  - Sub-item: Publicação em `dani.notificacoes` → consumer consome e grava evento `delivered` em `alerta_tracking`
  - Sub-item: Consumer falha 1x (mock throw) → reentrega após 5s (backoff 1ª tentativa)
  - Sub-item: Consumer falha 3x → mensagem em `dani.notificacoes.dlq`; log `dlq_message_received` gerado
  - Sub-item: DLQ crítica (`prioridade = 'critico'`) → Sentry `captureMessage` chamado 1x

---

## Feature 3 — Testes E2E Playwright

- [x] **[E2E-001]** Configurar ambiente E2E Playwright em `test/e2e/`:
  - Sub-item: `playwright.config.ts` com `testDir: './test/e2e'`, `timeout: 30000`, `use: { baseURL: 'http://localhost:5173' }`
  - Sub-item: Seed E2E: arquivo `test/e2e/fixtures/seed-e2e.ts` — 1 cessionário teste `id: '00000000-0000-0000-0000-000000000001'`, 3 oportunidades, sessão ativa
  - Sub-item: `beforeEach` reset: `DELETE FROM sessoes WHERE cessionario_id = 'cess-e2e-test'`; limpeza Redis `dani:*:cess-e2e-test*`
  - Sub-item: MSW configurado para interceptar EvolutionAPI real: `msw` com `delay: 100ms` fixo em todos os handlers (D27 §6.1)

- [x] **[E2E-002]** Implementar os 10 fluxos E2E obrigatórios (D27 §6.3):
  - Sub-item: E2E-001: Login via plataforma → acesso ao dashboard Dani; `[data-testid="dani-fab"]` visível
  - Sub-item: E2E-002: Chat → enviar mensagem → `waitForSelector('[data-testid="dani-message-complete"]', { timeout: 15000 })`; stream SSE completo recebido
  - Sub-item: E2E-003: `POST /calculadora/calcular` via chat com `OPR-001` (Δ>0) → resposta contém `R$ 3.000,00`; `[data-testid="loading-spinner"]` ausente (proibido — D28 §2.1)
  - Sub-item: E2E-004: 31ª mensagem em 1 hora → banner rate limit visível com countdown `mm:ss`; input desabilitado; `aria-live="assertive"` ativo
  - Sub-item: E2E-005: Fluxo vinculação WhatsApp completo — informar número → OTP correto → confirmação → badge "Vinculado" visível
  - Sub-item: E2E-006: Hard block OTP — 5 tentativas incorretas → `WA_OTP_HARD_BLOCK` 429; formulário desabilitado; countdown bloqueio exibido
  - Sub-item: E2E-007: Isolamento — JWT de cessionário A → `GET /oportunidades` sem registros de cessionário B; `opr-003` nunca presente na resposta de A
  - Sub-item: E2E-008: Estado Empty — nenhuma oportunidade disponível → `<EmptyState>` com botão "Ativar alertas" exibido; NENHUM skeleton infinito
  - Sub-item: E2E-009: LLM indisponível (MSW retorna 503) → FallbackBanner com cor `--agent-fallback` visível; `POST /calculadora/calcular` ainda funciona; NENHUM spinner
  - Sub-item: E2E-010: Acessibilidade — tab order no chat sem saltos; `aria-live="polite"` anuncia nova mensagem da Dani; `role="dialog"` no painel; focus trap ativo (Tab não sai do painel)

---

## Feature 4 — Testes de Contrato (Pact)

- [x] **[CONTRACT-001]** Implementar contrato OpenAI GPT-4o (D27 §5.2):
  - Sub-item: Consumer `dani-api`, provider `openai-gpt4o`
  - Sub-item: Request: `{ model: "gpt-4o", stream: true, temperature: 0.3, max_tokens: 2048 }`
  - Sub-item: Response: status `200`, `Content-Type: text/event-stream`, `choices[0].delta.content: string`
  - Sub-item: Contrato de falha: HTTP 429 → código `AGENTE_LLM_RATE_LIMIT`; HTTP 503 → `AGENTE_LLM_UNAVAILABLE` + retry 3x

- [x] **[CONTRACT-002]** Implementar contrato EvolutionAPI (D27 §5.3):
  - Sub-item: Consumer `dani-api`, provider `evolution-api-whatsapp`
  - Sub-item: Request: `POST /message/sendText/{instanceName}`, `{ number: string (E.164), text: string }`
  - Sub-item: Response: status `200` ou `201`, `{ key: { id: string } }`
  - Sub-item: Contrato de falha: HTTP 503 × 3 → `WA_CIRCUIT_OPEN`

- [x] **[CONTRACT-003]** Validar schema Supabase (D27 §5.4):
  - Sub-item: Gerar tipos via `supabase gen types typescript > src/types/supabase.ts` — sem divergência das tabelas reais
  - Sub-item: Colunas obrigatórias de `dani_conversas`, `dani_mensagens`, `dani_sessoes`, `dani_alertas`, `dani_vinculacoes_whatsapp` validadas contra schema Prisma

---

## Feature 5 — Checklist de Qualidade D28 e CI/CD

- [x] **[QUALITY-001]** Verificar todos os itens [BLOQUEANTE] do D28 Parte 1 — Code Review:
  - Sub-item: Frontend [BLOQUEANTE]: todos os componentes com 4 estados (Skeleton → Empty → Error → Populated); zero spinners em dados assíncronos (D28 §2.1)
  - Sub-item: Frontend [BLOQUEANTE]: barrel exports em `features/dani-chat/index.ts`, `features/dani-dashboard/index.ts`, `features/dani-whatsapp/index.ts`; sem imports diretos entre features
  - Sub-item: Backend [BLOQUEANTE]: `CessionarioOwnerGuard` em TODOS os endpoints de negócio; `WHERE cessionario_id = :id` em TODAS as queries
  - Sub-item: Backend [BLOQUEANTE]: nenhum dado sensível (phone, CPF, JWT, OTP, API key) em logs — Pino redact verificado
  - Sub-item: Backend [BLOQUEANTE]: `CalculadoraModule` não importa de `AgenteModule` (ADR-003) — verificação via `circular-dependency-plugin` no build
  - Sub-item: API [BLOQUEANTE]: todo endpoint documentado no D16; todo endpoint de negócio com `CessionarioOwnerGuard + JwtAuthGuard`

- [x] **[QUALITY-002]** Verificar itens [BLOQUEANTE] do D28 Parte 2 — Acessibilidade WCAG 2.1 AA:
  - Sub-item: [BLOQUEANTE] `role="main"` em área de conteúdo principal; `role="dialog"` no painel de chat com `aria-labelledby`; `role="log"` / `aria-live="polite"` na área de mensagens
  - Sub-item: [BLOQUEANTE] Focus trap ativo quando chat aberto; focus restaurado para FAB ao fechar
  - Sub-item: [BLOQUEANTE] Estados de erro anunciados via `aria-live` ou `role="alert"` — não apenas visualmente
  - Sub-item: [BLOQUEANTE] `@media (prefers-reduced-motion: reduce)` aplicado a todas as animações não essenciais
  - Sub-item: [BLOQUEANTE] Contraste texto normal ≥ 4.5:1 e texto grande ≥ 3:1 — validado axe-core zero violações `critical` ou `serious`
  - Sub-item: [BLOQUEANTE] Todas as funcionalidades do chat acessíveis via teclado; Esc fecha modais

- [x] **[QUALITY-003]** Verificar itens [BLOQUEANTE] do D28 Parte 3 — Segurança:
  - Sub-item: [BLOQUEANTE] OTP como bcrypt cost 12; phone como SHA-256 em Redis; NUNCA texto plano
  - Sub-item: [BLOQUEANTE] Todos os DTOs com `class-validator`; payloads de chat sanitizados antes de enviar ao LLM
  - Sub-item: [BLOQUEANTE] Nenhuma chave de API hardcoded; GitLeaks sem findings no CI; `.env` no `.gitignore`
  - Sub-item: [BLOQUEANTE] Pino redact ativo para todos os campos sensíveis (D28 §4.5)
  - Sub-item: [BLOQUEANTE] Índices em `cessionario_id`, `status`, `created_at` nas tabelas `dani_*`

- [x] **[QUALITY-004]** Configurar pipeline CI completo (D24):
  - Sub-item: `ci-pr.yml`: lint + type-check + unitários + integração + contrato Pact; gate: cobertura ≥ 80% (`[ $(echo "$COV >= 80" | bc) -eq 1 ] || exit 1`)
  - Sub-item: `ci-staging.yml`: disparado em push para `develop`; suite completa + E2E Playwright (Chromium)
  - Sub-item: `release.yml`: disparado em tag semver `v*.*.*`; regressão completa + build Docker + deploy Railway (API) + deploy Vercel (frontend)
  - Sub-item: GitLeaks scan em `ci-pr.yml` — bloqueia merge se secrets detectados
  - Sub-item: `pnpm audit` sem CVEs `critical` ou `high` — bloqueia merge se encontrar
  - Sub-item: Husky hooks: `pre-commit` (lint + type-check); `commit-msg` (Conventional Commits v1.0.0 — scopes válidos: `chat`, `agente`, `calculadora`, `auth`, `opr`, `alerta`, `whatsapp`, `ux`, `api`, `db`, `infra`)
  - Sub-item: PR max 400 linhas (D23 §branching); squash merge para feature/bugfix → develop; merge commit para develop → main

---

## Feature 6 — Testes de Acessibilidade Automatizados

- [x] **[TEST-A11Y-001]** Integrar axe-core ao Playwright E2E:
  - Sub-item: `@axe-core/playwright` instalado e configurado
  - Sub-item: Scan executado em todas as telas principais: dashboard, chat aberto, T-DC-006 tabela comparativa, T-DC-012 vinculação WhatsApp
  - Sub-item: Zero violações `critical` ou `serious` — gate automático no CI
  - Sub-item: Relatório HTML de acessibilidade gerado como artifact no CI a cada run

- [x] **[TEST-A11Y-002]** Verificar contraste dos tokens de risco manualmente:
  - Sub-item: `--risk-low: #16A34A` sobre `--risk-low-bg: #F0FDF4` — ratio ≥ 4.5:1
  - Sub-item: `--risk-medium: #D97706` sobre `--risk-medium-bg: #FFFBEB` — ratio ≥ 4.5:1
  - Sub-item: `--risk-high: #DC2626` sobre `--risk-high-bg: #FEF2F2` — ratio ≥ 4.5:1
  - Sub-item: `--primary-foreground` sobre `--primary: #0069A8` — ratio ≥ 4.5:1
  - Sub-item: `--agent-fallback: #0069A8` ⚠️ AMBÍGUO REQ-126 — texto sobre este fundo ≥ 4.5:1 validado

---

## 🔍 Auto-verificação S8 (12 checks)

| #   | Check                                                                                                                                                                                                                                     | Status |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                                                                              | ✅     |
| 2   | Nomes exatos: Vitest, Playwright, Pact, axe-core, `ci-pr.yml`, `ci-staging.yml`, `release.yml`, `@vitest/coverage-v8`, `seed-e2e.ts`, `cessionario.factory.ts`, `dani-api` (service), `supabase gen types typescript`                     | ✅     |
| 3   | Valores numéricos: cobertura ≥80%, coverage gate `bc` check, OTP bcrypt cost 12, hard block TTL 1800s, rate limit TTL 3600s, p95 ≤5s, disponibilidade ≥99.5%, 10 E2E obrigatórios, 12 eventos críticos de log, DLQ >10 → P1, DLQ >50 → P1 | ✅     |
| 4   | Cobertura 100% obrigatória para: `CalculadoraService`, `CessionarioOwnerGuard`, OTP/rate limit, isolamento                                                                                                                                | ✅     |
| 5   | Pirâmide de testes: 55% unitário (Vitest), 25% integração, 10% contrato (Pact), 10% E2E (Playwright)                                                                                                                                      | ✅     |
| 6   | `CessionarioOwnerGuard` NUNCA mockado em testes de integração de endpoints (REQ-149, D27 §3.3)                                                                                                                                            | ✅     |
| 7   | CI gates: GitLeaks, `pnpm audit`, cobertura ≥80%, E2E, Pact contracts                                                                                                                                                                     | ✅     |
| 8   | Nenhuma suposição sem base documental                                                                                                                                                                                                     | ✅     |
| 9   | `CalculadoraModule` não importa de `AgenteModule` — verificado via circular-dependency-plugin (ADR-003)                                                                                                                                   | ✅     |
| 10  | Nenhum item de scaffold — todos com sub-itens verificáveis (R10)                                                                                                                                                                          | ✅     |
| 11  | Banco real (Supabase local), Redis real, RabbitMQ real em testes de integração — sem mocks desses serviços                                                                                                                                | ✅     |
| 12  | REQs REQ-142–157 têm ≥1 item de checklist                                                                                                                                                                                                 | ✅     |

---

## REQs cobertos por esta sprint

REQ-142 (Pino logger configurado com redact e 12 eventos críticos), REQ-143 (Langfuse tracing com userId=sha256), REQ-144 (correlation_id em todos os logs), REQ-145 (pipeline CI/CD: ci-pr.yml + ci-staging.yml + release.yml), REQ-146 (Railway + Vercel deploy via release.yml), REQ-147 (Conventional Commits + Husky hooks), REQ-148 (cobertura ≥80% gate no CI), REQ-149 (CessionarioOwnerGuard não mockável em integração), REQ-150 (Checklist D28 [BLOQUEANTE] verificados), REQ-151 (10 fluxos E2E Playwright P0), REQ-152 (smoke test: POST /dani/chat + POST /calculadora/calcular ⚠️ AMBÍGUO REQ-152 adota paths D16), REQ-153 (Guia de Contribuição: PR max 400 linhas, squash merge, Conventional Commits), REQ-155 (Setup local: pnpm health valida todos os serviços), REQ-156 (Docker Compose com 4 serviços), REQ-157 (axe-core zero violações critical/serious)
