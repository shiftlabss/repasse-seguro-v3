# S1 — Fundação

## Sprint 1 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                                                                                                                   |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S1 — Fundação                                                                                                                                                           |
| **Template**       | A — Infraestrutura (Banco → Backend → Frontend → Wiring → Testes)                                                                                                       |
| **REQs cobertos**  | S1-001 a S1-037 (37 requisitos do Registro Mestre)                                                                                                                      |
| **Docs fonte**     | 02 - Stacks · 12 - Modelo de Dados · 13 - Schema Prisma · 14 - Especificações Técnicas · 15 - Arquitetura de Pastas · 20 - Error Handling · 25 - Observabilidade e Logs |
| **Total de itens** | 55 itens                                                                                                                                                                |
| **Status**         | Em Progresso                                                                                                                                                            |

---

## Auto-Verificação (12 checks)

- [ ] ✅ Check 1 — Nenhum item usa termo genérico. Todos os nomes espelham exatamente os docs (tabelas, enums, campos, endpoints).
- [ ] ✅ Check 2 — Cada item é binariamente verificável (feito / não feito).
- [ ] ✅ Check 3 — Valores numéricos idênticos aos docs (TTLs, limites, contagens).
- [ ] ✅ Check 4 — Glossário (D10) consultado: termos como `service_role`, `anon key`, `RLS`, `soft delete`, `UUID v4` usados corretamente.
- [ ] ✅ Check 5 — Nenhum item permite scaffold vazio (R10): cada item inclui validações, lógica real e sub-itens concretos.
- [ ] ✅ Check 6 — Máquinas de estado: se aplicável, todas as transições documentadas.
- [ ] ✅ Check 7 — TTLs e retry policies presentes para todos os recursos de cache e filas.
- [ ] ✅ Check 8 — Cross-módulo: ações que impactam outros módulos sinalizadas em seção dedicada.
- [ ] ✅ Check 9 — Sem conflitos não sinalizados entre docs.
- [ ] ✅ Check 10 — Sem ambiguidades não sinalizadas.
- [ ] ✅ Check 11 — Nenhum item depende de contexto perdido.
- [ ] ✅ Check 12 — 100% dos REQs S1-001 a S1-037 têm ≥1 item no checklist.

---

## 🗄️ BANCO — Migrations e Schema

- [x] **S1-B01** · Habilitar extensões PostgreSQL via migration SQL: `CREATE EXTENSION IF NOT EXISTS "pgcrypto";` e `CREATE EXTENSION IF NOT EXISTS "vector";` (Doc 13 — seção 2.3). Verificar que a migration é executada antes de qualquer outra migration de schema.

- [x] **S1-B02** · Criar migration Prisma para tabela `users`: campos `id UUID PK gen_random_uuid()`, `email VARCHAR(255) UNIQUE NOT NULL`, `name VARCHAR(255) NOT NULL`, `provider AuthProvider DEFAULT EMAIL`, `email_verified_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ DEFAULT NOW()`, `updated_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ`. Verificar que `@@map("users")` é aplicado e todos os campos têm `@db.Timestamptz` (nunca sem timezone). (Doc 13 — model User)

- [x] **S1-B03** · Criar migration Prisma para tabela `cessionarios`: campos `id UUID PK`, `user_id UUID UNIQUE FK → users.id RESTRICT`, `status CessionarioStatus DEFAULT CADASTRADA`, `phone VARCHAR(20)`, `phone_verified_at TIMESTAMPTZ`, `bank_account JSONB`, `bank_account_verified_at TIMESTAMPTZ`, `notification_preferences JSONB DEFAULT '{"email":true,"push":true,"sms":true}'`, `ai_consent BOOLEAN DEFAULT true`, `ai_consent_at TIMESTAMPTZ`, `investment_preferences JSONB`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`. Índice: `idx_cessionarios_status` em `status`. (Doc 13 — model Cessionario)

- [x] **S1-B04** · Criar migration Prisma para tabela `kyc_documents`: campos `id UUID PK`, `cessionario_id UUID FK → cessionarios.id RESTRICT`, `status KycStatus DEFAULT PENDENTE`, `identity_doc_front_url VARCHAR(500)`, `identity_doc_back_url VARCHAR(500)`, `address_doc_url VARCHAR(500)`, `selfie_url VARCHAR(500)`, `identity_status KycStatus DEFAULT PENDENTE`, `address_status KycStatus DEFAULT PENDENTE`, `selfie_status KycStatus DEFAULT PENDENTE`, `rejection_reason TEXT`, `idwall_session_id VARCHAR(255)`, `attempt_count INT DEFAULT 0`, `blocked_until TIMESTAMPTZ`, `reviewer_id UUID`, `reviewed_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`. Índice: `idx_kyc_documents_cessionario_id`. (Doc 13 — model KycDocument)

- [x] **S1-B05** · Criar migration Prisma para tabela `opportunities`: campos `id UUID PK`, `code VARCHAR(20) UNIQUE`, `status OpportunityStatus DEFAULT DISPONIVEL`, `city VARCHAR(100)`, `neighborhood VARCHAR(100)`, `state VARCHAR(2)`, `development_name VARCHAR(200)`, `bedrooms INT`, `area_sqm DECIMAL(8,2)`, `has_garage BOOLEAN`, `current_table_value DECIMAL(15,2)`, `contract_table_value DECIMAL(15,2)`, `cedente_paid_percentage DECIMAL(5,2)`, `cedente_paid_value DECIMAL(15,2)`, `ai_risk_score DECIMAL(4,2)`, `appreciation_data JSONB`, `published_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ`. Índices: `idx_opportunities_status`, `idx_opportunities_city_state`. Sem coluna `cedente_id` — anonimato estrutural (RN-063, Doc 12). (Doc 13 — model Opportunity)

- [x] **S1-B06** · Criar migration SQL adicional para coluna gerada `delta_value` em `opportunities`: `ALTER TABLE opportunities ADD COLUMN delta_value DECIMAL(15,2) GENERATED ALWAYS AS (current_table_value - contract_table_value) STORED;` e índice `CREATE INDEX idx_opportunities_delta_value ON opportunities (delta_value);`. (Doc 13 — seção 2.1)

- [x] **S1-B07** · Criar migration SQL adicional para coluna `embedding vector(1536)` em `opportunities` com índice HNSW: `ALTER TABLE opportunities ADD COLUMN embedding vector(1536);` e `CREATE INDEX idx_opportunities_embedding ON opportunities USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);`. (Doc 13 — seção 2.2)

- [x] **S1-B08** · Criar migrations Prisma para tabelas `proposals`, `negotiations`, `chat_messages`, `escrow_deposits`, `formalizations`, `financial_transactions`, `notifications`, `ai_sessions`, `audit_logs` com todos os campos, tipos, constraints, FKs, índices exatamente conforme Doc 13 (models completos). Verificar: `Decimal(15,2)` para todos os valores monetários; `UUID v4` para todos os PKs; `@db.Timestamptz` em todos os timestamps; soft delete `deleted_at` nas tabelas de domínio.

- [x] **S1-B09** · Criar e habilitar todos os 8 enums Prisma: `CessionarioStatus` (valores: CADASTRADA, KYC_EM_ANALISE, KYC_APROVADO, KYC_REPROVADO, BLOQUEADA_TEMPORARIAMENTE, ENCERRADA), `KycStatus` (PENDENTE, EM_ANALISE, APROVADO, REPROVADO), `OpportunityStatus` (DISPONIVEL, COM_PROPOSTA, EM_NEGOCIACAO, RESERVADA, CONCLUIDA, CANCELADA), `ProposalStatus` (ENVIADA, EM_ANALISE, ACEITA, RECUSADA, EXPIRADA, CANCELADA), `NegotiationStatus` (EM_NEGOCIACAO, EM_CONTRAPROPOSTA, AGUARDANDO_DEPOSITO, DEPOSITO_CONFIRMADO, ENCERRADA, CANCELADA), `EscrowStatus` (AGUARDANDO_DEPOSITO, DEPOSITO_ENVIADO, DEPOSITO_CONFIRMADO, REEMBOLSADO), `FormalizationStatus` (DOCUMENTOS_DISPONIVEIS, ASSINATURA_PENDENTE_CESSIONARIO, ASSINATURA_PENDENTE_CEDENTE, AGUARDANDO_ANUENCIA, CONCLUIDA, CANCELADA), `TransactionType`, `TransactionStatus`, `AuthProvider`, `ActorType`. (Doc 13 — seção de enums)

- [x] **S1-B10** · Configurar políticas de RLS (Row Level Security) no Supabase para todas as 12 tabelas: habilitar RLS via `ALTER TABLE <tabela> ENABLE ROW LEVEL SECURITY;`; policy de leitura para `cessionario_id = auth.uid()` nas tabelas `cessionarios`, `kyc_documents`, `proposals`, `negotiations`, `escrow_deposits`, `formalizations`, `financial_transactions`, `notifications`, `ai_sessions`; tabelas `opportunities` e `audit_logs` com policies específicas de leitura pública e escrita somente por `service_role`. Verificar: `SUPABASE_SERVICE_ROLE_KEY` nunca exposta no frontend. (Doc 17 — seção Supabase; Doc 12 — ERD)

- [ ] **S1-B11** · Criar seed Prisma em `prisma/seed/` com dados sintéticos para dev: 1 usuário admin, 5 cessionários com status variados, 20 oportunidades com status DISPONIVEL/COM_PROPOSTA, 10 propostas, 5 negociações. Verificar: seed executável via `pnpm prisma db seed` sem erros.

---

## ⚙️ BACKEND — Infraestrutura NestJS

- [x] **S1-BE01** · Inicializar monorepo Turborepo + pnpm workspaces com estrutura exata: `apps/web/`, `apps/api/`, `apps/mobile/`, `packages/shared-types/`, `packages/design-tokens/`, `packages/eslint-config/`, `packages/tsconfig/`, `prisma/` centralizado (schema.prisma, migrations/, seed/, middleware/), `turbo.json`, `pnpm-workspace.yaml`. Verificar: `pnpm install` funciona no root; `turbo run build` compila todos os workspaces. (Doc 15 — seção 2)

- [x] **S1-BE02** · Configurar `apps/api/` NestJS 10 com TypeScript 5.4 `strict: true`: instalar dependências `@nestjs/core`, `@nestjs/common`, `@nestjs/platform-express`, `@nestjs/config`, `@nestjs/throttler`, `@nestjs/swagger`, `class-validator`, `class-transformer`, `@prisma/client`, `prisma`, `pino`, `pino-http`, `helmet`, `@sentry/nestjs`. Verificar: `pnpm type-check` sem erros; `tsc --noEmit` limpo.

- [x] **S1-BE03** · Configurar Prisma 6 em `apps/api/`: `DATABASE_URL` e `DIRECT_URL` como env vars; `previewFeatures = ["postgresqlExtensions"]`; extensões `pgcrypto` e `pgvector`. Verificar: `npx prisma generate` sem erros; `npx prisma migrate deploy` executa todas as migrations na ordem correta.

- [x] **S1-BE04** · Configurar módulo de infraestrutura `apps/api/src/infrastructure/`: `RedisModule` com Upstash via `ioredis` (env var: `UPSTASH_REDIS_URL`); TTL padrão por recurso: `rs:oportunidades:lista` 5min, `rs:oportunidade:{id}` 10min, `rs:cessionario:{id}:perfil` 15min, `rs:kyc:{id}:status` 5min, `rs:proposta:quota:{id}` 1min, `rs:ai:sessao:{id}` 30min (conforme Doc 14 — seção Redis). Verificar: conexão Redis estabelecida no startup; `PING` retorna `PONG`.

- [x] **S1-BE05** · Configurar RabbitMQ em `apps/api/src/infrastructure/rabbitmq/`: 4 exchanges (`notification.exchange` direct, `kyc.exchange` topic, `ai.exchange` direct, `system.exchange` fanout) com 7 filas (`notification.email`, `notification.push`, `notification.inapp`, `kyc.process`, `ai.rag-index`, `ai.rag-query`, `system.cleanup`) cada uma com DLQ correspondente (`*.dead-letter`). Retry policy por fila: `notification.email` — 3 tentativas, backoff 30s→60s→120s; `notification.push` — 2 tentativas; `kyc.process` — 3 tentativas, backoff 2s→4s→8s. Conexão via `CLOUDAMQP_URL`. (Doc 14 — seção RabbitMQ)

- [x] **S1-BE06** · Implementar `GlobalExceptionFilter` em `apps/api/src/common/filters/`: capturar todas as exceções e retornar schema padrão `{ "error": { "code", "message", "user_message", "correlation_id", "timestamp", "metadata?" } }` (Doc 20 — seção 2); nunca incluir stack trace no response; gerar `correlation_id` UUID v4 se não presente no request header; mapear todas as 13 classes de erro (`AUTH-001`, `AUTH-003`, `AUTH-008`, `AUTH-010`, `VAL-001`, `NOT-FOUND-001`, `CONFLICT-001`, `RATE-001`, `PERM-001`, `EXT-001`, `EXT-002`, `DB-001`, `INT-001`) para status HTTP correto.

- [x] **S1-BE07** · Implementar `LoggerService` em `apps/api/src/common/logger/` com Pino 9: log JSON estruturado com campos obrigatórios `timestamp` (ISO 8601 UTC), `level`, `service`, `module`, `action`, `message`, `correlation_id`, `environment`; `pino.redact` para sanitizar `password`, `token`, `jwt`, `cpf`, `bank_account`, `refresh_token` — nunca aparecem em logs; nível `debug` apenas em dev; `console.log` proibido em produção (ESLint rule `no-console`). Retenção: `error/warn` 30 dias, `info` 7 dias, `debug` não persistido. (Doc 25 — seções 2 e 3)

- [x] **S1-BE08** · Configurar `CorrelationIdMiddleware` em `apps/api/src/common/middleware/`: extrair `X-Correlation-ID` do header ou gerar UUID v4; injetar no contexto da request; propagar em todos os logs Pino e em todas as respostas de erro. Configurar `helmet` para headers de segurança. Verificar: toda requisição tem `correlation_id` propagado end-to-end.

- [x] **S1-BE09** · Configurar Sentry 9 em `apps/api/`: capturar exceções não tratadas, rastrear performance de endpoints críticos; configurar `dsn` via `SENTRY_DSN`; `environment` dinâmico (`development`/`staging`/`production`); dados sensíveis (CPF, JWT) excluídos via `beforeSend`. Configurar alertas: novos erros P0 → Slack `#incidentes`; taxa de erros > 2% em 5min → PagerDuty. (Doc 25 — seção Sentry; Doc 29 — seção dashboards)

- [x] **S1-BE10** · Configurar `@nestjs/throttler` em `AppModule`: 100 req/min geral; 10 req/15min para rotas de auth (`/api/v1/auth/*`); 5 req/10min para upload KYC (`/api/v1/kyc/documents`); 20 req/min para endpoints LLM (`/api/v1/ai/*`). Resposta de rate limit: `429` com código `RATE-001` e header `Retry-After`. (Doc 16 — seção de rate limiting)

- [x] **S1-BE11** · Configurar `@nestjs/swagger` em `apps/api/src/main.ts`: título `Repasse Seguro API`, versão `1.0`, base URL `/api/v1`, Bearer auth JWT; todos os DTOs com `@ApiProperty`; todos os endpoints com `@ApiOperation` e `@ApiResponse`. Verificar: `/api-docs` acessível com schema completo.

- [x] **S1-BE12** · Configurar health check endpoint `GET /health`: retornar `{ "status": "ok", "timestamp": "<ISO 8601>" }` em HTTP 200; verificar conexão PostgreSQL (query `SELECT 1`), conexão Redis (`PING`) e conexão RabbitMQ. Falha em qualquer dependência retorna `{ "status": "degraded", "checks": { ... } }` com HTTP 503. (Doc 29 — seção 2.1)

- [x] **S1-BE13** · Implementar hierarquia de classes de erro em `apps/api/src/common/errors/`: `AppError` (abstrata) com propriedades `statusCode`, `errorCode`, `loggingLevel`, `isRetryable`; subclasses concretas: `ValidationError` (422, VAL-001), `AuthError` (401, AUTH-001), `ForbiddenError` (403, PERM-001), `NotFoundError` (404, NOT-FOUND-001), `ConflictError` (409, CONFLICT-001), `RateLimitError` (429, RATE-001, isRetryable=true), `ExternalServiceError` (502, EXT-001, isRetryable=true), `ExternalServiceTimeoutError` (504, EXT-002, isRetryable=true), `InternalError` (500, INT-001). (Doc 20 — seção 3)

- [x] **S1-BE14** · Configurar estrutura de módulos NestJS em `apps/api/src/modules/`: criar diretórios para 10 módulos de domínio (`auth/`, `cessionarios/`, `kyc/`, `opportunities/`, `proposals/`, `negotiations/`, `escrow/`, `formalization/`, `financial/`, `ai/`) + `common/` + `infrastructure/`, cada um com padrão `controller.ts`, `service.ts`, `repository.ts`, `dto/`, `entities/` (barrel exports `index.ts` em cada pasta). (Doc 15 — seção backend)

- [x] **S1-BE15** · Configurar integração Resend em `apps/api/src/infrastructure/resend/`: SDK `resend` npm; remetente `noreply@repasseseguro.com.br` com display `"Repasse Seguro"`; retry 3x backoff exponencial (30s→60s→120s); DLQ após 3 falhas em `notification.email.dead-letter`; rate limit monitorado: 50.000 e-mails/mês (plano Resend Pro, 100 req/s). Configuração via `RESEND_API_KEY`. (Doc 17 — seção Resend; Doc 21 — seção 2.1)

---

## 🎨 FRONTEND — Setup React + Design System

- [x] **S1-FE01** · Inicializar `apps/web/` com Vite 7 + React 19 + TypeScript 5.4 `strict: true`: instalar dependências `react`, `react-dom`, `@tanstack/react-router`, `@tanstack/react-query`, `zustand`, `tailwindcss@4`, `@shadcn/ui`, `framer-motion@12`, `date-fns@4`, `date-fns-tz@3`. Verificar: `pnpm dev` sobe na porta 3000; `pnpm type-check` limpo.

- [x] **S1-FE02** · Configurar estrutura de pastas `apps/web/src/` exatamente conforme Doc 15: `features/` (10 subpastas: auth, dashboard, marketplace, proposals, negotiations, formalizations, financial, ai, profile, notifications), `components/ui/`, `components/layout/` (AppLayout, AuthLayout, FullPageLayout, ErrorLayout), `components/domain/` (KpiCard, SlaCountdown, QuotaBar, OpportunityCard, ProposalStatusBadge, NegotiationStatusBadge, EscrowStatusBadge, FormalizationStatusBadge, CessionarioStatusBadge, AiRiskScoreBadge), `components/shared/` (EmptyState, DataTable, TableSkeleton, OfflineBanner), `hooks/`, `stores/`, `services/`, `lib/`, `types/`, `router/`. Barrel exports `index.ts` em cada pasta. Imports cross-feature via barrel proibidos.

- [x] **S1-FE03** · Configurar Tailwind CSS 4 e shadcn/ui: `globals.css` com variáveis CSS de design tokens (cores, espaçamento, tipografia) conforme `packages/design-tokens/`; inicializar shadcn/ui com comando `npx shadcn@latest init`; instalar componentes base: `Button`, `Input`, `Label`, `Badge`, `Card`, `Dialog`, `Sheet`, `Skeleton`, `Toast`, `Table`, `Select`, `Checkbox`. Verificar: hot reload funciona; sem erros de CSS.

- [x] **S1-FE04** · Configurar TanStack Router em `apps/web/src/router/`: `RootRoute` com `RouterProvider`; rotas iniciais `/login`, `/cadastro`, `/recuperar-senha`, `/nova-senha` (sem auth); rotas protegidas com `AuthGuard`, `RbacGuard`, `KycGuard` em `router/guards/`; constantes de rotas tipadas em `routes.ts` (sem strings mágicas nos componentes). Verificar: navegação entre rotas funciona; guards redirecionam corretamente.

- [x] **S1-FE05** · Implementar `services/api.ts`: fetch wrapper com `JWT interceptor` (adiciona `Authorization: Bearer <token>` no header); refresh automático via `POST /api/v1/auth/refresh` quando 401 recebido; se refresh falha, redirecionar para `/login`; `correlation_id` UUID v4 gerado por request e adicionado ao header `X-Correlation-ID`; timeout de 30s por request; tipos de resposta com `{ data, error }` discriminated union.

- [x] **S1-FE06** · Implementar `services/supabase.ts`: cliente Supabase JS com `anon key` (`VITE_SUPABASE_ANON_KEY`) — nunca `service_role`; configuração de Realtime para subscriptions; subscriptions filtradas por `cessionario_id` via RLS. Implementar `useRealtimeSubscription` hook em `hooks/` para re-uso.

- [x] **S1-FE07** · Implementar stores Zustand em `stores/`: `auth.store.ts` (estado: `user`, `cessionario`, `kycStatus`, `isAuthenticated`, `pendingDeepLink`); `notifications.store.ts` (estado: `unreadCount`, `notifications[]`, `markAsRead`); `ui.store.ts` (estado: `sidebarOpen`, `theme`, `preferredView`). Verificar: stores persistem entre reloads via `zustand/middleware` `persist` onde necessário.

- [x] **S1-FE08** · Implementar `services/analytics.ts`: wrapper PostHog centralizado; nenhum componente chama PostHog diretamente; eventos mapeados por nome exato (ex: `proposal_created`, `kyc_submitted`, `opportunity_viewed`); dados sensíveis (CPF, e-mail) nunca incluídos em eventos PostHog. Configuração via `VITE_POSTHOG_KEY`. (Doc 28 — seção 2.1)

- [x] **S1-FE09** · Implementar `lib/commission.ts`: função `calculateBuyerCommission(currentTableValue: number, contractTableValue: number, cedentePaidValue: number): number` — lógica: se `Δ = currentTableValue - contractTableValue > 0` então `comissão = 0.20 × Δ`; se `Δ ≤ 0` então `comissão = 0.20 × cedentePaidValue`; resultado em centavos (inteiro); sem floating point (usar inteiros ou Decimal.js). Testes unitários: 100% de cobertura desta função. (Doc 01.2 — RN-031)

- [x] **S1-FE10** · Implementar `lib/currency.ts` (formatação BRL), `lib/dates.ts` (date-fns + timezone `America/Fortaleza`), `lib/validators.ts` (CPF, telefone, e-mail). Implementar `Error Boundary` global em nível de rota e global, com fallback UI adequado. (Doc 28 — seção 2.1)

- [x] **S1-FE11** · Configurar Sentry SDK no frontend `apps/web/`: capturar exceções JS não tratadas, erros de fetch, performance de rotas críticas; `dsn` via `VITE_SENTRY_DSN`; `environment` dinâmico; dados sensíveis excluídos via `beforeSend`. Configurar no mobile `apps/mobile/` da mesma forma. (Doc 25)

---

## 🔌 WIRING — Integração e Configuração de Ambiente

- [x] **S1-W01** · Criar `.env.example` na raiz do monorepo com todas as variáveis de ambiente necessárias (sem valores reais): `DATABASE_URL`, `DIRECT_URL`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `UPSTASH_REDIS_URL`, `CLOUDAMQP_URL`, `ZAPSIGN_API_TOKEN`, `ZAPSIGN_WEBHOOK_SECRET`, `IDWALL_API_KEY`, `IDWALL_WEBHOOK_SECRET`, `RESEND_API_KEY`, `OPENAI_API_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_PUBLIC_KEY`, `SENTRY_DSN`, `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `VITE_POSTHOG_KEY`, `VITE_SENTRY_DSN`. Verificar: nenhum valor real commitado; `.env` no `.gitignore`.

- [ ] **S1-W02** · Configurar CI/CD via GitHub Actions: criar `.github/workflows/ci.yml` com jobs `validate` (lint + type-check + commitlint + testes unitários/integração) e `e2e` (apenas para PRs em `main`, usa Playwright); timeout 20min para `validate`, 30min para `e2e`. Configurar `.github/workflows/deploy-staging.yml` (trigger: push em `develop`, build Railway + `prisma migrate deploy`). (Doc 24 — seção 3)

- [x] **S1-W03** · Configurar `turbo.json` com pipeline de tasks: `build` (depende de `^build`), `test` (depende de `^build`), `lint` (sem dependências), `type-check` (sem dependências), `dev` (persistente). Configurar `pnpm-workspace.yaml` com workspaces `apps/*` e `packages/*`. Verificar: `turbo run test` executa testes em todos os workspaces em paralelo.

- [ ] **S1-W04** · Configurar 3 projetos Supabase separados (dev/staging/prod) com variáveis de ambiente por ambiente. Verificar: `SUPABASE_SERVICE_ROLE_KEY` disponível apenas no backend (Railway secrets); `SUPABASE_ANON_KEY` disponível no frontend (Vercel env vars). Configurar URLs de projeto distintas por ambiente.

- [ ] **S1-W05** · Configurar PostHog Cloud: projeto criado, `VITE_POSTHOG_KEY` configurada, captura de eventos básicos (`pageview`, `session_start`) funcionando em staging. Feature flags configuradas para funcionalidades condicionais. (Doc 29 — seção T-7)

---

## 🧪 TESTES — Infraestrutura de Testes

- [x] **S1-T01** · Configurar Vitest em `apps/api/` e `apps/web/`: instalar `vitest`, `@vitest/coverage-v8`, `@nestjs/testing`, `@testing-library/react`, `@testing-library/jest-dom`; configurar thresholds de cobertura: geral 80%, `CommissionService` e `EscrowService` 100%; scripts `pnpm test` e `pnpm test:coverage`. Verificar: `pnpm test` executa sem erros no monorepo vazio.

- [ ] **S1-T02** · Configurar Playwright para E2E em `apps/web/`: instalar `@playwright/test`; configurar `playwright.config.ts` com `baseURL` de staging; browsers: Chromium, Firefox, WebKit; timeout 30s por teste; screenshots e vídeos em falha retidos 7 dias. Verificar: `pnpm test:e2e` executa contra staging sem erros de configuração.

- [x] **S1-T03** · Implementar teste unitário da função `calculateBuyerCommission`: caso 1 — Δ positivo (currentTableValue=500000, contractTableValue=400000 → comissão=20000, ou seja 20% × 100000); caso 2 — Δ zero (comissão=20% × cedentePaidValue); caso 3 — Δ negativo (comissão=20% × cedentePaidValue); caso 4 — valores em centavos (sem floating point); verificar 100% de cobertura de branches. (Doc 01.2 — RN-031)

- [ ] **S1-T04** · Implementar testes de integração para migrations Prisma: verificar que `npx prisma migrate deploy` em banco PostgreSQL de teste executa todas as migrations sem erro; verificar que todos os índices existem via `SELECT indexname FROM pg_indexes WHERE tablename = '<tabela>'`; verificar RLS habilitado em todas as tabelas via `SELECT tablename, rowsecurity FROM pg_tables`.

- [x] **S1-T05** · Verificar configuração de ESLint com rule `no-console` (bloqueia `console.log` em produção), `@typescript-eslint/no-explicit-any` (proíbe `any` implícito), `import/no-cycle` (proíbe imports circulares entre features). Configurar commitlint com conventional commits (`feat`, `fix`, `chore`, `docs`, `test`, `refactor`). (Doc 28 — seções 2.1 e 2.2)

---

## 📊 COBERTURA DE REQs S1

| REQ ID | Descrição                                  | Item(s) do checklist |
| ------ | ------------------------------------------ | -------------------- |
| S1-001 | Turborepo monorepo com 3 workspaces        | S1-BE01, S1-W03      |
| S1-002 | PostgreSQL 17 + Prisma 6                   | S1-BE03, S1-B01—B11  |
| S1-003 | 12 tabelas com schema exato                | S1-B02—B08           |
| S1-004 | 8 enums Prisma                             | S1-B09               |
| S1-005 | RLS em todas as tabelas                    | S1-B10               |
| S1-006 | Soft delete nas tabelas de domínio         | S1-B02—B08           |
| S1-007 | UUID v4 em todos os PKs                    | S1-B02—B08           |
| S1-008 | Redis Upstash com 6 TTLs                   | S1-BE04              |
| S1-009 | RabbitMQ com 4 exchanges e 7 filas + DLQs  | S1-BE05              |
| S1-010 | GlobalExceptionFilter schema único         | S1-BE06              |
| S1-011 | 13 classes de erro                         | S1-BE13              |
| S1-012 | CorrelationId obrigatório                  | S1-BE08              |
| S1-013 | LoggerService Pino + redact sensíveis      | S1-BE07              |
| S1-014 | Sentry backend + frontend + mobile         | S1-BE09, S1-FE11     |
| S1-015 | Rate limiting 4 regras                     | S1-BE10              |
| S1-016 | Swagger configurado                        | S1-BE11              |
| S1-017 | Health check endpoint                      | S1-BE12              |
| S1-018 | Integração Resend                          | S1-BE15              |
| S1-019 | Coluna gerada delta_value                  | S1-B06               |
| S1-020 | pgvector HNSW index embedding              | S1-B07               |
| S1-021 | NestJS 10 TypeScript strict                | S1-BE02              |
| S1-022 | React 19 + Vite 7 + TypeScript strict      | S1-FE01              |
| S1-023 | Estrutura de pastas frontend feature-first | S1-FE02              |
| S1-024 | Tailwind CSS 4 + shadcn/ui                 | S1-FE03              |
| S1-025 | TanStack Router com guards                 | S1-FE04              |
| S1-026 | Fetch wrapper com JWT interceptor          | S1-FE05              |
| S1-027 | Supabase cliente + Realtime                | S1-FE06              |
| S1-028 | Stores Zustand (auth, notifications, ui)   | S1-FE07              |
| S1-029 | PostHog wrapper centralizado               | S1-FE08              |
| S1-030 | lib/commission.ts cálculo comissão         | S1-FE09              |
| S1-031 | lib/currency, lib/dates, lib/validators    | S1-FE10              |
| S1-032 | Error Boundary global                      | S1-FE10              |
| S1-033 | .env.example completo                      | S1-W01               |
| S1-034 | GitHub Actions CI/CD                       | S1-W02               |
| S1-035 | Vitest + Playwright configurados           | S1-T01, S1-T02       |
| S1-036 | Testes calculateBuyerCommission 100%       | S1-T03               |
| S1-037 | ESLint + commitlint                        | S1-T05               |
