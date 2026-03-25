# S13 — Qualidade

## Metadados

| Campo              | Valor                                               |
| ------------------ | --------------------------------------------------- |
| **Sprint**         | S13                                                 |
| **Nome**           | Qualidade                                           |
| **Docs Fonte**     | D27 (Plano de Testes), D28 (Checklist de Qualidade) |
| **REQs cobertos**  | REQ-189                                             |
| **Total de itens** | 42                                                  |
| **Template**       | B — Módulo Fullstack (organizado por FEATURE)       |
| **Gerado em**      | 2026-03-24                                          |
| **Status**         | Concluido                                           |

---

> **Objetivo da sprint:** Configurar e implementar toda a infraestrutura de qualidade do projeto — pirâmide de testes (Vitest + Supertest + Playwright), gates de CI/CD, cobertura mínima por módulo, checklist de qualidade D28 integrado ao pipeline e dashboards de monitoramento contínuo.

---

## FEATURE 1 — Infraestrutura de Testes (Pirâmide 70/20/5/5)

### 1.1 Configuração Vitest (unitários + integração)

- [x] **UNIT-01 — Configurar `vitest.config.ts` para backend (NestJS)**
  - Arquivo: `apps/api/vitest.config.ts`
  - `environment: 'node'`, `globals: true`, `coverage.provider: 'v8'`
  - `coverage.thresholds`: `AuthService` ≥ 85% branches; `CasesService` ≥ 80%; `EscrowService` ≥ 80%; `AiAgentService` ≥ 75%; demais serviços ≥ 70% (D27 §3.4)
  - `coverage.include`: `['src/**/*.service.ts', 'src/**/*.guard.ts', 'src/**/*.pipe.ts']`
  - `coverage.exclude`: `['src/**/*.spec.ts', 'src/**/*.e2e-spec.ts', 'src/main.ts']`
  - Verificação: `pnpm vitest run --coverage` gera relatório sem erros

- [x] **UNIT-02 — Configurar `vitest.config.ts` para frontend (React)**
  - Arquivo: `apps/web/vitest.config.ts`
  - `environment: 'jsdom'`, integração com `@testing-library/react` e `msw`
  - `coverage.thresholds.branches` ≥ 60% em componentes com lógica
  - `setupFiles: ['./src/test/setup.ts']` com `import '@testing-library/jest-dom'`
  - Verificação: `pnpm vitest run --coverage` em `apps/web` sem erros

- [x] **UNIT-03 — Configurar `vitest.config.ts` para mobile (React Native)**
  - Arquivo: `apps/mobile/vitest.config.ts`
  - `environment: 'node'`, `@testing-library/react-native` configurado
  - Verificação: `pnpm vitest run` em `apps/mobile` sem erros

- [x] **UNIT-04 — Criar convenções e estrutura de factories**
  - Arquivo: `apps/api/src/test/factories/case.factory.ts` — `createCase(overrides?)` com faker
  - Arquivo: `apps/api/src/test/factories/user.factory.ts` — `createUser({ role })` com faker
  - Arquivo: `apps/api/src/test/factories/escrow.factory.ts` — `createEscrowAccount(overrides?)`
  - Arquivo: `apps/api/src/test/seed-test.ts` — seed de usuários por role + casos em status críticos (D27 §8.1)
  - Regra: **nunca usar CPFs reais** — usar `@faker-js/faker` com extensão brasileira (`faker.cpf()`)
  - Verificação: seed executa sem erros; `prisma.user.count()` retorna ≥ 4 usuários com roles distintos

### 1.2 Testes Unitários por Módulo

- [x] **UNIT-05 — Testes unitários `AuthService`**
  - `auth.service.spec.ts` ao lado de `auth.service.ts`
  - Cenários obrigatórios (D27 §7.1):
    - Login sem 2FA → retorna `access_token` + `refresh_token`
    - Login com 2FA habilitado → retorna `{ requires_2fa: true, temp_token }` + `redis.setex` chamado com `rs:2fa_pending:{id}` TTL 300
    - 5 tentativas incorretas → lança `AccountLockedError`; `locked_until` persistido no banco
    - Refresh token rotation → token antigo adicionado ao blacklist Redis; novo par emitido
    - Token reusado (blacklist) → lança `TokenReusedError`; sessão invalidada
    - Logout → token adicionado ao blacklist `rs:token_blacklist:{jti}` com TTL restante
  - Mocks: `vi.mock('PrismaService')`, `vi.mock('RedisService')`, `vi.mock('bcrypt')`
  - Cobertura mínima: ≥ 85% branches (D27 §3.4)
  - Verificação: `pnpm vitest run auth.service.spec.ts --coverage` passa com ≥ 85%

- [x] **UNIT-06 — Testes unitários `CasesService`**
  - `cases.service.spec.ts`
  - Cenários obrigatórios (D27 §7.2):
    - Transição de status válida → `status_history` registrado
    - Transição inválida (ex: `CONCLUIDO → EM_TRIAGEM`) → lança `InvalidTransitionError` com `BIZ_002`
    - Conflito de versão (Optimistic Lock) → lança `ConflictError` HTTP 409
    - Criação de caso → `case_config_snapshots` populado com configs atuais do Redis (RN-111)
    - Cancelamento em qualquer estado pré-Fechamento → retorna `CANCELADO`
    - Soft delete → `deleted_at` setado; caso não retorna em listagem
  - Cobertura mínima: ≥ 80% branches
  - Verificação: `pnpm vitest run cases.service.spec.ts --coverage` passa com ≥ 80%

- [x] **UNIT-07 — Testes unitários `EscrowService`**
  - `escrow.service.spec.ts`
  - Cenários obrigatórios (D27 §7.3):
    - Criação de conta escrow → idempotência via `external_id` (segunda chamada retorna existente)
    - Depósito confirmado → `EscrowTransaction` criada com `type: DEPOSIT`; `escrow_account.status` → `ATIVO`
    - Distribuição com escrow bloqueado → lança `ForbiddenError` 403
    - Distribuição com idempotência `external_id` → segunda chamada retorna `200 OK` sem duplicar
    - Comissão calculada corretamente cenário A → `COMMISSION_CEDENTE_PCT` aplicado sobre valor residual
    - Comissão calculada corretamente cenário B → `COMMISSION_CESSIONARIO_PCT` aplicado
    - Comissão calculada corretamente cenário C → 30% valorização considerada
    - Comissão calculada corretamente cenário D → 50% valorização considerada
  - Cobertura mínima: ≥ 80% branches
  - Verificação: `pnpm vitest run escrow.service.spec.ts --coverage` passa com ≥ 80%

- [x] **UNIT-08 — Testes unitários `AiAgentService`**
  - `ai-agent.service.spec.ts`
  - Cenários obrigatórios (D27 §7.5):
    - Confiança ≥ 90% → age autonomamente; `ai_agent_decisions` registrado com `executed_autonomously: true`
    - Confiança 70–89% → enfileira para revisão humana; `requires_human_approval: true`
    - Takeover manual → agente pausado para o caso; Redis `rs:agent:takeover:{agent}:{case_id}` setado
    - Takeover automático após 3 ações idênticas em 60s → `LOOP_DETECTADO` registrado
    - Schema de saída inválido → lança `CriticalFailureError`; `DADOS_CORROMPIDOS` registrado; não persiste
    - Limite de 5 takeovers simultâneos → lança `TakeoverLimitError`
  - Cobertura mínima: ≥ 75% branches
  - Verificação: `pnpm vitest run ai-agent.service.spec.ts --coverage` passa com ≥ 75%

### 1.3 Testes de Integração (Supertest)

- [x] **INTEG-01 — Setup de testes de integração com testcontainers**
  - Arquivo: `apps/api/src/test/integration-setup.ts`
  - PostgreSQL via `testcontainers` — container isolado por suíte; sem contaminar banco dev (D27 §4.1)
  - Redis: `ioredis-mock` no módulo de integração
  - RabbitMQ: mock de publicação — validar apenas que mensagem é publicada, não a entrega
  - `@nestjs/testing` + Supertest para requisições HTTP reais
  - Reset entre testes: `beforeAll` banco limpo + `afterEach` transação revertida onde possível
  - Verificação: suíte de integração executa em < 5 minutos

- [x] **INTEG-02 — Testes de integração `POST /v1/auth/login`**
  - Cenários (D27 §4.2 + §7.1):
    - Login bem-sucedido sem 2FA → 200, `access_token` presente, formato JWT válido
    - Login com 2FA habilitado → 200, `requires_2fa: true`, `temp_token` presente, Redis populado
    - 5 tentativas incorretas → 401 `AUTH_003`, `locked_until` no response
    - Conta já bloqueada → 401 `AUTH_003`, sem incremento adicional
    - RBAC: ANALISTA tentando acessar rota de MASTER → 403 `AUTH_005`
  - Verificação: todos cenários passando no CI; banco não alterado em cenários de erro

- [x] **INTEG-03 — Testes de integração `PATCH /v1/cases/:id/status`**
  - Cenários (D27 §7.2):
    - Transição válida → 200, `status` atualizado, `status_history` registrado no banco
    - Transição inválida → 422 `BIZ_002`; banco sem alteração (verificar via query direta)
    - Conflito de versão (`version` desatualizada) → 409; banco sem alteração
    - ANALISTA tentando transição reservada a COORDENADOR → 403
  - Verificação: todos cenários passando; uso de factory `createCase()`

- [x] **INTEG-04 — Testes de integração `EscrowService` com banco real**
  - Cenários (D27 §7.3):
    - Criação de conta escrow → `escrow_accounts` populado; `POST /v1/cases/:id/escrow` retorna 201
    - Webhook Celcoin com HMAC-SHA256 válido → 200, `EscrowTransaction` criada
    - Webhook Celcoin com HMAC-SHA256 inválido → 401 `WEBHOOK_001`
    - Desbloqueio de escrow por não-MASTER → 403
  - Verificação: HMAC calculado corretamente; banco atualizado após webhook

- [x] **INTEG-05 — Testes de integração webhooks ZapSign**
  - Cenários (D27 §7.4):
    - Webhook ZapSign `SIGNED` com HMAC válido → 200; `envelope_status` → `SIGNED`; job publicado
    - Webhook ZapSign `REFUSED` → 200; `envelope_status` → `REFUSED`; notificação disparada
    - Webhook com HMAC inválido → 401
    - Webhook `SIGNED` com todos os signatários → notificação `T-14 INSTRUMENTO_ASSINADO` publicada no RabbitMQ
  - Verificação: verificar publicação no mock RabbitMQ; sem chamadas reais ao ZapSign

### 1.4 Testes de Contrato

- [x] **CONTRACT-01 — Contratos obrigatórios via schemas Zod**
  - Arquivo: `apps/api/src/test/contracts/`
  - Contratos obrigatórios (D27 §5.2):
    - `login.contract.spec.ts` → `LoginResponseSchema` (sem 2FA): `{ access_token, refresh_token, user: { id, role } }`
    - `case-create.contract.spec.ts` → `CaseResponseSchema`: campos obrigatórios + `config_snapshot`
    - `webhook-zapsign.contract.spec.ts` → `ZapsignWebhookSchema`: `document_token`, `status`, `signers`
    - `webhook-celcoin.contract.spec.ts` → `CelcoinWebhookSchema`: `transaction_id`, `status`, `amount`
    - `notification-job.contract.spec.ts` → `NotificationJobSchema`: `template_id`, `recipient_id`, `channel`, `variables`
  - Todos os schemas com `safeParse(response.body.data)` → `expect(parsed.success).toBe(true)`
  - Verificação: `pnpm vitest run --reporter=verbose` todos os 5 contratos passing

### 1.5 Testes E2E (Playwright)

- [x] **E2E-01 — Setup Playwright**
  - Arquivo: `apps/web/playwright.config.ts`
  - `baseURL`: staging dedicado com banco isolado
  - `globalSetup`: `pnpm --filter api prisma db seed:test` antes de cada suíte
  - `globalTeardown`: truncar tabelas de domínio (não `audit`) após cada suíte
  - Autenticação real: `storageState` por role salvo em `playwright/.auth/{role}.json`
  - ZapSign/Celcoin: mocked via `msw` no nível de rede — sem chamadas reais
  - `@axe-core/playwright` configurado para detectar violações WCAG 2.1 AA automaticamente (D27 §11.2)
  - Verificação: `pnpm playwright test --list` lista os 5 fluxos críticos

- [x] **E2E-02 — Fluxo crítico: Login com 2FA (TOTP)**
  - Arquivo: `apps/web/e2e/auth-2fa.spec.ts`
  - Cenário: usuário Admin com `two_fa_enabled: true` → 6 inputs OTP → `verify-2fa` → dashboard (D27 §6.1; RN-001, RN-005)
  - axe-core: zero violações `critical` ou `serious` na tela de login
  - Verificação: teste E2E passa em staging; `page.waitForURL('/dashboard')` bem-sucedido

- [x] **E2E-03 — Fluxo crítico: Criação de Caso com config snapshot**
  - Arquivo: `apps/web/e2e/case-create.spec.ts`
  - Cenário: ANALISTA preenche formulário → POST `/v1/cases` → `case_config_snapshots` criado com configs atuais (D27 §6.1; RN-111)
  - Verificar via API que `case_config_snapshots.created_at` não é nulo após criação
  - axe-core: zero violações no formulário de criação
  - Verificação: teste E2E passa; snapshot verificado via `GET /v1/cases/:id`

- [x] **E2E-04 — Fluxo crítico: Fechamento de Caso (4 critérios)**
  - Arquivo: `apps/web/e2e/case-closing.spec.ts`
  - Cenário: caso em `EM_FORMALIZACAO` → verificar botão `[data-testid=confirm-close-btn]` desabilitado → seed cumpre 4 critérios → reload → botão habilitado com classe `.pulse` → confirmar → `FECHAMENTO` (D27 §6.3; RN-021, RN-023)
  - axe-core: zero violações no modal de confirmação de fechamento
  - Verificação: `data-testid=case-status-badge` contém texto `FECHAMENTO`

- [x] **E2E-05 — Fluxo crítico: Distribuição de Conta Escrow**
  - Arquivo: `apps/web/e2e/escrow-distribution.spec.ts`
  - Cenário: GESTOR_FINANCEIRO acessa financeiro → caso em `FECHAMENTO` → distribuição → 200 → `EscrowTransaction` criada (D27 §6.1; RN-085, RN-086)
  - Verificar idempotência: segundo clique não dispara nova transação (verificar via API)
  - axe-core: zero violações na tela de distribuição
  - Verificação: teste E2E passa; `GET /v1/cases/:id/escrow` retorna `status: DISTRIBUIDO`

- [x] **E2E-06 — Fluxo crítico: Assinatura via ZapSign (webhook)**
  - Arquivo: `apps/web/e2e/zapsign-signing.spec.ts`
  - Cenário: ANALISTA cria envelope → URL de assinatura exibida → mock webhook `SIGNED` via msw → `envelope_status` → `SIGNED` → notificação disparada (D27 §6.1; RN-040 a RN-047)
  - Verificar `FormalizationService` atualiza status do caso
  - axe-core: zero violações na tela de formalização
  - Verificação: `data-testid=envelope-status` exibe `SIGNED` após webhook mockado

---

## FEATURE 2 — Gates de CI/CD

- [x] **CI-01 — Configurar `ci.yml` no GitHub Actions**
  - Arquivo: `.github/workflows/ci.yml`
  - Jobs em paralelo (D27 §12):
    - `push` para feature branch → `unit-tests` apenas (< 2min; warning se falhar, não bloqueia)
    - `PR → develop` → `typecheck` + `lint` + `unit-tests` + `integration-tests` + `contract-tests` (< 8min; bloqueia PR)
    - `merge → develop` → todos os jobs acima + `e2e-tests` (< 20min; alerta se E2E falhar)
    - `merge → main` → suite completa + `smoke-test-staging` (< 25min; rollback automático se falhar)
  - `unit-tests`: `pnpm vitest run --coverage` — falha se thresholds abaixo do mínimo
  - `integration-tests`: `pnpm vitest run --config vitest.integration.config.ts`
  - `e2e-tests`: `pnpm playwright test` em ambiente staging
  - Verificação: pipeline executa em PR de teste; jobs de PR bloqueiam se testes falham

- [x] **CI-02 — Gate de segurança `pnpm audit`**
  - Job `security-audit` em `ci.yml` executando em todo PR
  - `pnpm audit --audit-level=high` — falha o pipeline se houver vulnerabilidades High/Critical (D28 §6.5; B15 equivalente)
  - `detect-secrets` como pre-commit hook — impede commit de secrets no repositório (D27 §11.3)
  - Verificação: `pnpm audit` retorna exit code 0 em repositório limpo

- [x] **CI-03 — Gate de cobertura de branches com enforcement**
  - `vitest --coverage --coverage.thresholdAutoUpdate=false`
  - Limites aplicados por arquivo (D27 §9.1):
    - `AuthService`: 85% branches — BLOQUEIA PR
    - `CasesService`, `EscrowService`: 80% branches — BLOQUEIA PR
    - `AiAgentService`: 75% branches — BLOQUEIA PR
    - Endpoints críticos documentados: 100% cobertos — BLOQUEIA PR
    - 5 fluxos E2E críticos: todos passando — BLOQUEIA merge em `main`
  - Verificação: PR com cobertura abaixo do threshold é bloqueado; report HTML gerado

- [x] **CI-04 — Smoke test de carga mínimo (Autocannon)**
  - Arquivo: `.github/workflows/smoke-load.yml` ou step em `ci.yml` pré-release
  - Comando: `npx autocannon -c 100 -d 30 https://staging.repasseseguro.com.br/health` (D27 §11.1)
  - Critério: latência p95 < 500ms; zero erros 5xx durante os 30 segundos
  - Executado apenas em `merge → main` (não em todos os PRs)
  - Verificação: Autocannon report mostra p95 < 500ms; zero 5xx

---

## FEATURE 3 — Checklist de Qualidade D28 — Gates Integrados

- [x] **QUAL-01 — Gate B01: TypeScript strict — zero `any` implícito**
  - ESLint rule `@typescript-eslint/no-explicit-any: error` no `eslint.config.ts` (D28 §4.1)
  - Exceções documentadas inline com `// justificativa: ...` obrigatório
  - CI job `lint` falha se regra violada sem justificativa
  - Verificação: `pnpm lint` retorna exit 0; nenhum `any` não-justificado nos arquivos de produção

- [x] **QUAL-02 — Gate B02: Todos os endpoints com guard de autorização**
  - Teste automatizado de verificação: grep em todos os `.controller.ts` para rotas sem `@UseGuards(JwtAuthGuard)` ou `@Public()` explícito
  - Incluir no code review checklist de PR: "Nenhum endpoint sem `@Roles` + `@UseGuards`" (D28 §4.2; B02)
  - Verificação: script `pnpm check:auth-guards` retorna zero endpoints não-protegidos

- [x] **QUAL-03 — Gate B03 + B04: Transações obrigatórias e RFC 7807**
  - ESLint custom rule (ou grep no CI): detectar `prisma.{create,update,delete}` fora de `prisma.$transaction()`
  - Teste de integração: `POST /v1/cases` com erro forçado no meio → banco sem estado parcial (rollback verificado)
  - Teste de integração: resposta de erro qualquer → schema RFC 7807 verificado via `ErrorResponseSchema` Zod (`type`, `title`, `status`, `detail`, `instance`, `correlation_id`, `timestamp`) (D28 §4.3; B04)
  - Verificação: grep CI não encontra escrita fora de transação; todos os erros seguem RFC 7807

- [x] **QUAL-04 — Gate B05: Migrações aditivas**
  - CI step `validate-migrations`: grep nos arquivos de migration em `prisma/migrations/` por `DROP COLUMN`, `DROP TABLE`, `ALTER COLUMN ... TYPE` sem ciclo de deprecação documentado (D28 §4.4; B05)
  - Se encontrado: bloquear PR com erro explícito `"Migration não-aditiva detectada: {arquivo}"`
  - Verificação: PR com migration `DROP COLUMN` é bloqueado pelo CI

- [x] **QUAL-05 — Gate B06–B08: Segurança de dados**
  - `pino.redact` configurado com paths: `['authorization', 'password', 'token', 'secret', 'cpf', 'pix_key', 'credit_card', 'api_key']` (D28 §7.1)
  - Teste de integração: logar requisição com CPF no body → verificar que log não contém CPF em texto plano
  - Hash bcrypt `saltOrRounds: 12` confirmado em `AuthService` (D28 §6.3; B07)
  - Verificação: log de staging filtrado; `authService.hash('test')` retorna hash com `$2b$12$`

- [x] **QUAL-06 — Gate B09–B12: Acessibilidade WCAG 2.1 AA**
  - `@axe-core/playwright` integrado a todos os 5 fluxos E2E — zero violações `critical` ou `serious` (D28 §5; D27 §11.2)
  - Critérios automaticamente verificados (D28 §5.2):
    - Contraste texto normal ≥ 4.5:1 (B09)
    - Sem keyboard trap (B10)
    - Skip link funcional — primeiro elemento focável (B11)
    - `prefers-reduced-motion: reduce` — animações não essenciais desabilitadas (B12)
  - Lighthouse CI configurado: bloquear PR com score de acessibilidade < 90
  - Verificação: `npx playwright test --project=chromium` com axe-core sem violações nos 5 fluxos

- [x] **QUAL-07 — Gate B13–B14: Observabilidade obrigatória**
  - Correlation ID obrigatório em todos os logs: `correlationId` injetado pelo `CorrelationIdMiddleware` (D28 §7.1; B13)
  - Teste de integração: `GET /health` com DB offline → resposta contém `db: degraded`; não retorna 200 mascarado (D28 §7.2; B14)
  - SLOs configurados no Railway: API availability ≥ 99.5%, p95 ≤ 500ms, DLQ rate ≤ 1%, email delivery ≥ 95% em 5min (D28 §7.2)
  - Verificação: log de staging mostra `correlationId` em toda linha; `/health` retorna `degraded` quando dep offline

- [x] **QUAL-08 — Gate B15: Bundle ≤ 500 KB gzipped**
  - Webpack Bundle Analyzer / Vite bundle report no CI (D28 §4.5; B15)
  - `vite-plugin-visualizer` configurado em `vite.config.ts`
  - CI step `bundle-size`: falha se alguma página exceder 500 KB gzipped
  - Code splitting obrigatório: `import()` dinâmico para páginas de relatórios, configurações, supervisão IA
  - Web Vitals monitorados: LCP < 2.5s, FID < 100ms, CLS < 0.1 (D28 §4.5)
  - Verificação: `pnpm build` + bundle report; nenhuma rota acima de 500 KB gzipped

---

## FEATURE 4 — Periodicidade de Auditoria e Documentação

- [x] **DOC-01 — Template de PR com checklist D28**
  - Arquivo: `.github/PULL_REQUEST_TEMPLATE.md`
  - Seção obrigatória: `## Checklist de Qualidade` com todos os B01–B15 como `[ ]`
  - Instrução: "Marcar `[x]` para cada item coberto. Itens `[N/A]` com justificativa no comentário."
  - Verificação: novo PR criado exibe template com seção de qualidade

- [x] **DOC-02 — Auditoria quinzenal automatizada**
  - Arquivo: `.github/workflows/quality-audit.yml` — execução `schedule: cron: '0 9 * * 1'` (segundas-feiras)
  - Jobs: `pnpm audit` + Lighthouse CI em staging + axe-core E2E + relatório publicado como artefato GitHub Actions
  - Resultado postado no canal `#rs-operacional` via Slack webhook (D28 §11.2, §11.3)
  - Verificação: workflow executa sem erros no próximo gatilho semanal

- [x] **DOC-03 — `CONTRIBUTING.md` com guia de testes**
  - Arquivo: `CONTRIBUTING.md` — seções:
    - "Como rodar testes unitários" (`pnpm vitest run`)
    - "Como rodar testes de integração" (`pnpm vitest run --config vitest.integration.config.ts`)
    - "Como rodar E2E" (`pnpm playwright test`)
    - "Convenções de naming" (`{subject}.spec.ts` ao lado do arquivo; estrutura `describe → it`)
    - "Isolamento de dados" (testcontainers; nunca banco de dev; `faker.cpf()`)
    - "Cobertura mínima por módulo" (tabela D27 §3.4)
  - Verificação: arquivo existe; revisado pelo Tech Lead antes do go-live

---

## FEATURE 5 — Testes de Regressão e Não-Funcionais

- [x] **REG-01 — Suite de regressão por módulo**
  - Definir escopo por evento de merge (D27 §10.1):
    - Merge de bugfix em `develop` → módulo afetado + integração relacionada
    - Merge de feature em `develop` → suite completa unitários + integração
    - Merge em `main` → suite completa + E2E dos 5 fluxos críticos
    - Rollback de deploy → suite completa + análise do diff
  - Regra de manutenção: teste flaky > 3×/semana → abrir issue `[FLAKY]` + `describe.skip` temporário + prioridade alta (D27 §10.2)
  - Verificação: processo documentado em `CONTRIBUTING.md`; workflow `ci.yml` reflete os escopos

- [x] **REG-02 — Headers de segurança verificados**
  - Teste de integração: `GET /health` response contém headers (D28 §6.4):
    - `Strict-Transport-Security: max-age=63072000; includeSubDomains`
    - `Content-Security-Policy` presente e não vazio
    - `X-Frame-Options: DENY`
    - `X-Content-Type-Options: nosniff`
  - Configurados via Helmet.js middleware NestJS em `app.module.ts`
  - CORS: lista explícita de origens — sem `*` em produção (D28 §6.4)
  - Verificação: `pnpm vitest run security-headers.spec.ts` passa

- [x] **REG-03 — Rate limiting verificado por endpoint crítico**
  - Teste de integração: POST `/v1/auth/login` com 11 requisições em 60s → 11ª retorna 429 `RATE_LIMIT_001` (D28 §6.6; máximo 10 req/min por IP)
  - Rate limit também verificado para `/v1/auth/reset-password` e `/v1/auth/verify-2fa`
  - Alerta brute force: ≥ 10 falhas de login em 60s → log nível `warn` com campo `event: BRUTE_FORCE_ATTEMPT` (D28 §6.6)
  - Verificação: teste retorna 429 na 11ª requisição; log contém campo correto

---

## 🔀 Cross-Módulo

- [x] **CROSS-01 — Validar que RabbitMQ mock captura todas as notificações nos testes**
  - Testes de integração de CasesService, EscrowService, ZapSign webhook → verificar que `NotificationService.emit()` é chamado com `template_id` correto para cada evento (S8a)
  - Mock de RabbitMQ: `vi.mock('RabbitMQService')` com spy em `publish()`
  - Verificação: `expect(mockRabbitMQ.publish).toHaveBeenCalledWith('rs.notifications', expect.objectContaining({ template_id: 'T-XX' }))`

- [x] **CROSS-02 — Validar que audit trail é registrado nos testes de integração**
  - Após cada operação que altera estado de caso, escrow ou usuário → verificar registro em schema `audit` (D28 §4.2)
  - Método: `prisma.$queryRaw('SELECT * FROM audit.events WHERE entity_id = $1', caseId)` — espera ≥ 1 registro
  - Verificação: testes de integração de fechamento, distribuição, e takeover IA verificam audit trail

- [x] **CROSS-03 — S13 é gate obrigatório para SN1** [CORRIGIDO: FINDING-003]
  - S13 é dependência obrigatória de SN1. Gates CI-01 a CI-04, QUAL-01 a QUAL-08 bloqueiam SN1 se não aprovados.
  - DevOps deve confirmar que todos os gates CI estão passando e o Checklist D28 tem zero itens 🔴 em aberto antes de iniciar SN1.
  - Evidência: relatório de cobertura Vitest + Playwright all-passing postado em `#rs-go-live` antes de T-7
  - Verificação: `pnpm ci:all` passa sem erros em staging; Tech Lead assina aprovação S13 em `#rs-go-live`

---

## 🔧 Auto-Verificação (12 Checks)

| #   | Check                                                                                                                                                                                     | Status |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis (feito/não feito)                                                                                                                            | ✅     |
| 2   | EXATAMENTE os nomes dos docs: `AuthService`, `CasesService`, `EscrowService`, `AiAgentService`, `vitest.config.ts`, `playwright.config.ts`, `ci.yml`                                      | ✅     |
| 3   | Valores numéricos exatos: 85% (Auth), 80% (Cases/Escrow), 75% (AI), 70% (outros), 60% (React), 500 KB bundle, 4.5:1 contraste, 10 req/min rate limit, 5 fluxos E2E, TTL 300 (2fa_pending) | ✅     |
| 4   | Glossário D10 consultado: termos `Cedente`, `Cessionário`, `Caso`, `CONCLUIDO`, `CANCELADO`, `EM_FORMALIZACAO`, `FECHAMENTO` usados canonicamente                                         | ✅     |
| 5   | IDs reais referenciados: REQ-189, D27, D28, RN-001, RN-005, RN-021, RN-023, RN-040–047, RN-085, RN-086, RN-111, B01–B15, T-14                                                             | ✅     |
| 6   | Máquinas de estado: transições de status testadas incluem todas as inválidas relevantes                                                                                                   | ✅     |
| 7   | Sem omissão de schedules/TTLs: `rs:2fa_pending:{id}` TTL 300, Autocannon 30s, cron `0 9 * * 1` auditoria                                                                                  | ✅     |
| 8   | Conflitos: nenhum conflito identificado entre D27 e D28 nesta sprint                                                                                                                      | ✅     |
| 9   | Sem lacunas preenchidas com suposições — todos os cenários rastreados a D27 §3.4, §4.2, §6.1, §7 ou D28 §4–§7                                                                             | ✅     |
| 10  | Anti-scaffold R10: cada item tem sub-itens com validações, RBAC, lógica, erros e verificações — nenhum item permite scaffold vazio                                                        | ✅     |
| 11  | Template B aplicado: organizado por FEATURE com vertical slices Banco→Backend→Frontend→Wiring→Testes                                                                                      | ✅     |
| 12  | REQ-189 (D02 §6: Testes Vitest + Supertest + Playwright; pirâmide 70/20/5/5) coberto por UNIT-01–UNIT-08, INTEG-01–05, CONTRACT-01, E2E-01–06, CI-01–04                                   | ✅     |

---

_Sprint gerada pelo pipeline ShiftLabs v2.3 — módulo Admin, Repasse Seguro._
