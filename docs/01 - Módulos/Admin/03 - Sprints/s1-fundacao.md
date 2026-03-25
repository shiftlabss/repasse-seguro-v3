# S1 — Fundação

## Módulo Admin — Repasse Seguro

| Campo                | Valor                                                                      |
| -------------------- | -------------------------------------------------------------------------- |
| **Sprint**           | S1                                                                         |
| **Nome**             | Fundação                                                                   |
| **Tipo**             | Fixa — Infraestrutura                                                      |
| **Template**         | A                                                                          |
| **Docs Consultados** | D02, D12, D13, D14, D15, D17, D20, D22, D23, D01.5                         |
| **Cross-cutting**    | D10 (Glossário), D02 (Stacks)                                              |
| **REQs cobertos**    | REQ-178 a REQ-192, REQ-250 a REQ-272, REQ-183 a REQ-187, REQ-190 a REQ-192 |
| **Total de itens**   | 58                                                                         |

---

> **Critério de conclusão de S1:** monorepo rodando localmente com `pnpm dev`; banco existe com esquema completo (20 tabelas + schema audit); layout shell renderiza com sidebar/header; rotas protegidas por auth guard redirecionam para /login; error handling global funcional; zero lógica de negócio.

---

## 🗄️ BANCO

### Migrations — Schema `public`

- [x] **[BANCO-01]** Criar migration `001_create_enums`: enums `UserRole` (ANALISTA, COORDENADOR, GESTOR_FINANCEIRO, MASTER), `UserExternalStatus` (ATIVO, SUSPENSO, INATIVO), `CaseStatus` (CAPTADO, EM_TRIAGEM, BLOQUEADO, QUALIFICADO, OFERTA_ATIVA, EM_NEGOCIACAO, EM_FORMALIZACAO, FECHAMENTO, POS_FECHAMENTO, EM_REVERSAO, EM_MEDIACAO, DISPUTA_FORMAL, CONCLUIDO, CANCELADO), `CaseScenario` (A, B, C, D). Verificar: `SELECT enumlabel FROM pg_enum` lista todos os 4 enums com todos os valores.

- [x] **[BANCO-02]** Criar migration `002_create_users`: tabela `users` com colunas `id UUID PK gen_random_uuid()`, `email VARCHAR(255) NOT NULL UNIQUE`, `name VARCHAR(255) NOT NULL`, `role UserRole NOT NULL`, `password_hash VARCHAR(255) NOT NULL`, `is_active BOOLEAN NOT NULL DEFAULT true`, `two_fa_enabled BOOLEAN NOT NULL DEFAULT false`, `two_fa_secret VARCHAR(255) NULL`, `failed_login_attempts INTEGER NOT NULL DEFAULT 0`, `locked_until TIMESTAMPTZ NULL`, `version INTEGER NOT NULL DEFAULT 1`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `deleted_at TIMESTAMPTZ NULL`. Índices: `UNIQUE(email)`, `INDEX(role)`, `INDEX(is_active, deleted_at)`. Verificar: `\d users` no psql mostra todas as 15 colunas com tipos corretos.

- [x] **[BANCO-03]** Criar migration `003_create_cedentes_cessionarios`: tabelas `cedentes` e `cessionarios`, cada uma com colunas `id UUID PK`, `name VARCHAR(255) NOT NULL`, `cpf VARCHAR(14) NOT NULL UNIQUE`, `email VARCHAR(255) NOT NULL UNIQUE`, `phone VARCHAR(20) NULL`, `address TEXT NULL`, `status UserExternalStatus NOT NULL DEFAULT 'ATIVO'`, `version INTEGER NOT NULL DEFAULT 1`, `created_at/updated_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ NULL`. Verificar: `\d cedentes` e `\d cessionarios` exibem estrutura idêntica.

- [x] **[BANCO-04]** Criar migration `004_create_cases`: tabela `cases` com colunas `id UUID PK`, `cedente_id UUID NOT NULL FK→cedentes(id)`, `assigned_analyst_id UUID NULL FK→users(id)`, `status CaseStatus NOT NULL DEFAULT 'CAPTADO'`, `scenario CaseScenario NOT NULL`, `property_address TEXT NOT NULL`, `contract_table_value DECIMAL(15,2) NOT NULL`, `current_table_value DECIMAL(15,2) NULL`, `current_table_source VARCHAR(100) NULL`, `paid_amount DECIMAL(15,2) NOT NULL`, `delta DECIMAL(15,2) NULL`, `estimated_recovered_value DECIMAL(15,2) NULL`, `distrato_reference_value DECIMAL(15,2) NULL`, `cancellation_reason TEXT NULL`, `version INTEGER NOT NULL DEFAULT 1`, `created_by UUID FK→users(id)`, `updated_by UUID FK→users(id)`, `created_at/updated_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ NULL`. Verificar: FK constraints ativas; `DELETE FROM cases` com referência bloqueia com erro de FK.

- [x] **[BANCO-05]** Criar migration `005_create_case_config_snapshots`: tabela `case_config_snapshots` com colunas `id UUID PK`, `case_id UUID NOT NULL FK→cases(id) UNIQUE`, `commission_cedente_pct DECIMAL(5,2) NOT NULL`, `commission_cessionario_pct DECIMAL(5,2) NOT NULL`, `delta_review_threshold DECIMAL(15,2) NOT NULL`, `distrato_reference_pct DECIMAL(5,2) NOT NULL`, `snapshot_at TIMESTAMPTZ NOT NULL`. Verificar: UNIQUE em `case_id` garante 1 snapshot por caso.

- [x] **[BANCO-06]** Criar migration `006_create_case_status_history`: tabela `case_status_history` com colunas `id UUID PK`, `case_id UUID NOT NULL FK→cases(id)`, `from_status CaseStatus NULL`, `to_status CaseStatus NOT NULL`, `changed_by UUID FK→users(id)`, `reason TEXT NULL`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`. Verificar: inserção com `from_status=NULL` (estado inicial) funciona.

- [x] **[BANCO-07]** Criar migration `007_create_dossie_documents`: tabela `dossie_documents` com colunas `id UUID PK`, `case_id UUID NOT NULL FK→cases(id)`, `document_type VARCHAR(50) NOT NULL`, `storage_path TEXT NOT NULL`, `status VARCHAR(20) NOT NULL DEFAULT 'PENDENTE'` (PENDENTE, VERIFICADO, REJEITADO), `rejection_reason TEXT NULL`, `verified_by UUID NULL FK→users(id)`, `verified_at TIMESTAMPTZ NULL`, `version INTEGER NOT NULL DEFAULT 1`, `created_by UUID FK→users(id)`, `created_at/updated_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ NULL`. Verificar: INDEX(case_id) criado para performance.

- [x] **[BANCO-08]** Criar migration `008_create_proposals_counterproposals`: tabelas `proposals` (id, case_id FK, cessionario_id FK→cessionarios, proposed_value DECIMAL, status VARCHAR, expires_at TIMESTAMPTZ, version INT, created_at/updated_at, deleted_at) e `counterproposals` (id, proposal_id FK→proposals, case_id FK, counter_value DECIMAL, reason TEXT NULL, created_by FK→users, created_at). Verificar: FK de `counterproposals.proposal_id → proposals.id` ativa.

- [x] **[BANCO-09]** Criar migration `009_create_zapsign_envelopes`: tabela `zapsign_envelopes` com colunas `id UUID PK`, `case_id UUID FK→cases(id)`, `document_type VARCHAR(50) NOT NULL`, `zapsign_document_token VARCHAR(255) NULL`, `status VARCHAR(30) NOT NULL DEFAULT 'PENDENTE'`, `signatories JSONB NOT NULL DEFAULT '[]'`, `expires_at TIMESTAMPTZ NULL`, `resend_count INTEGER NOT NULL DEFAULT 0`, `created_by UUID FK→users(id)`, `created_at/updated_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ NULL`. Verificar: coluna `signatories` aceita JSONB array.

- [x] **[BANCO-10]** Criar migration `010_create_formalization_criteria`: tabela `formalization_criteria` com colunas `id UUID PK`, `case_id UUID NOT NULL FK→cases(id) UNIQUE`, `signatures_ok BOOLEAN NOT NULL DEFAULT false`, `annuence_ok BOOLEAN NOT NULL DEFAULT false`, `annuence_na BOOLEAN NOT NULL DEFAULT false`, `annuence_protocol TEXT NULL`, `annuence_document_path TEXT NULL`, `deposit_ok BOOLEAN NOT NULL DEFAULT false`, `instrument_ok BOOLEAN NOT NULL DEFAULT false`, `delta_approved BOOLEAN NOT NULL DEFAULT false`, `delta_approved_by UUID NULL FK→users(id)`, `delta_approved_at TIMESTAMPTZ NULL`, `created_at/updated_at TIMESTAMPTZ`. Verificar: UNIQUE(case_id) garante 1 registro por caso.

- [x] **[BANCO-11]** Criar migration `011_create_escrow_accounts_transactions`: tabelas `escrow_accounts` (id, case_id FK UNIQUE, celcoin_account_id VARCHAR NULL, status VARCHAR, expected_amount DECIMAL, deposited_amount DECIMAL DEFAULT 0, residual_credit DECIMAL DEFAULT 0, deposit_confirmed_at, distribution_due_at, distributed_at, distribution_blocked BOOLEAN DEFAULT false, block_reason TEXT NULL, blocked_by FK→users NULL, block_approved_by FK→users NULL, version INT, created_at/updated_at) e `escrow_transactions` (id, escrow_account_id FK, transaction_type VARCHAR, amount DECIMAL, description TEXT NULL, celcoin_transaction_id VARCHAR NULL, created_by FK→users, created_at). Verificar: UNIQUE(case_id) em `escrow_accounts`.

- [x] **[BANCO-12]** Criar migration `012_create_commission_distributions`: tabelas `commission_invoices` (id, case_id FK, party_type VARCHAR (CEDENTE/CESSIONARIO), commission_amount DECIMAL, base_amount DECIMAL, commission_pct DECIMAL, status VARCHAR, distributed_at NULL, created_at/updated_at) e `distributions` (id, case_id FK, escrow_account_id FK, recipient_type VARCHAR, recipient_id UUID, amount DECIMAL, status VARCHAR, celcoin_transfer_id VARCHAR NULL, distributed_at NULL, created_at). Verificar: INDEX(case_id) em ambas as tabelas.

- [x] **[BANCO-13]** Criar migration `013_create_notification_logs`: tabela `notification_logs` com colunas `id UUID PK`, `case_id UUID NULL FK→cases(id)`, `recipient_id UUID NOT NULL`, `recipient_type VARCHAR(30) NOT NULL`, `channel VARCHAR(20) NOT NULL` (EMAIL, PAINEL, WHATSAPP, SMS), `event_type VARCHAR(100) NOT NULL`, `status VARCHAR(20) NOT NULL` (ENVIADO, FALHOU), `payload JSONB NULL`, `error_message TEXT NULL`, `created_at TIMESTAMPTZ NOT NULL`. Verificar: INDEX(case_id), INDEX(recipient_id, created_at).

- [x] **[BANCO-14]** Criar migration `014_create_ai_agent_decisions`: tabela `ai_agent_decisions` com colunas `id UUID PK`, `case_id UUID NOT NULL FK→cases(id)`, `agent_name VARCHAR(50) NOT NULL` (GUARDIAO_DO_RETORNO, ANALISTA_DE_OPORTUNIDADES), `decision_type VARCHAR(100) NOT NULL`, `decision_data JSONB NOT NULL`, `confidence_score DECIMAL(5,2) NOT NULL`, `outcome VARCHAR(30) NOT NULL`, `operator_feedback TEXT NULL`, `reviewed_by UUID NULL FK→users(id)`, `created_at TIMESTAMPTZ NOT NULL`. Verificar: INDEX(case_id, agent_name, created_at).

- [x] **[BANCO-15]** Criar migration `015_create_global_configs`: tabela `global_configs` com colunas `id UUID PK`, `config_key VARCHAR(100) NOT NULL UNIQUE`, `config_value TEXT NOT NULL`, `data_type VARCHAR(20) NOT NULL` (STRING, NUMBER, BOOLEAN, JSON), `description TEXT NULL`, `updated_by UUID NULL FK→users(id)`, `update_reason TEXT NULL`, `created_at/updated_at TIMESTAMPTZ`. Verificar: UNIQUE(config_key); inserir seed com valores padrão das configurações.

- [x] **[BANCO-16]** Criar migration `016_create_audit_schema`: schema separado `audit`; tabela `audit.audit_logs` com 13 colunas obrigatórias: `id UUID PK`, `timestamp TIMESTAMPTZ NOT NULL`, `event_type VARCHAR(100) NOT NULL`, `actor_id UUID NOT NULL`, `actor_profile UserRole NOT NULL`, `case_id UUID NULL`, `entity_type VARCHAR(50) NOT NULL`, `entity_id UUID NOT NULL`, `previous_value JSONB NULL`, `new_value JSONB NULL`, `ip_address VARCHAR(45) NOT NULL`, `user_agent TEXT NOT NULL`, `correlation_id UUID NOT NULL`. Sem soft delete; sem UPDATE; sem DELETE via aplicação. Verificar: `INSERT` funciona; `UPDATE` e `DELETE` bloqueados por política de RLS ou trigger.

- [x] **[BANCO-17]** Criar seed de `global_configs` com valores padrão: `commission_cedente_pct = 20`, `commission_cessionario_pct = 20`, `distrato_reference_pct = 50`, `delta_review_threshold = 100000`, `ai_confidence_threshold_guardiao = 80`, `ai_confidence_threshold_analista = 80`, `password_reset_expiry_hours = 1`, `session_timeout_admin_hours = 8`, `session_timeout_cedente_hours = 24`, `session_timeout_cessionario_minutes = 30`. Verificar: `SELECT config_key, config_value FROM global_configs` retorna ≥10 registros.

---

## ⚙️ BACKEND

### Monorepo e Configuração Base

- [x] **[BE-01]** Inicializar monorepo pnpm workspaces com estrutura: `apps/web/`, `apps/api/`, `apps/mobile/`, `packages/types/`, `packages/utils/`. Verificar: `pnpm install` na raiz instala dependências de todos os workspaces sem erros.

- [x] **[BE-02]** Configurar `apps/api/` como aplicação NestJS 10+ com TypeScript strict (`"strict": true` no tsconfig). Instalar: `@nestjs/core`, `@nestjs/common`, `@nestjs/platform-express`, `@nestjs/config`, `@nestjs/swagger`, `@nestjs/throttler`, `helmet` 8+, `class-validator`, `class-transformer`, `pino`, `pino-nestjs`, `@sentry/nestjs` 9+. Verificar: `pnpm run dev --filter api` inicia sem erros de compilação TypeScript.

- [x] **[BE-03]** Configurar Prisma 6+ em `apps/api/prisma/`: schema.prisma com datasource PostgreSQL 17+ (Supabase), client generator, todos os models das 20 tabelas + schema audit. Executar `prisma migrate deploy` em staging. Verificar: `prisma db pull` retorna schema idêntico ao definido.

- [x] **[BE-04]** Configurar Redis (Upstash em produção, Docker local): `ioredis` client wrapper com helper `setWithTTL` que rejeita chamadas sem TTL definido. Verificar: `redis.set('test', 'val', 'EX', 60)` funciona; `redis.set('test', 'val')` sem TTL lança exceção customizada `MissingTtlError`.

- [x] **[BE-05]** Configurar RabbitMQ 4+ (CloudAMQP em produção, Docker local): exchanges `notifications`, `zapsign`, `celcoin`, `ai`, `jobs`. Cada exchange com dead-letter queue (`{exchange}.dlq`) e retry exponencial (3 tentativas, backoff 1s/5s/30s). Verificar: `amqp-connection-manager` reconecta automaticamente após `docker stop rabbitmq && docker start rabbitmq`.

- [x] **[BE-06]** Configurar `apps/api/.env.example` com todas as variáveis obrigatórias: `DATABASE_URL`, `REDIS_URL`, `RABBITMQ_URL`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `ZAPSIGN_API_KEY`, `ZAPSIGN_WEBHOOK_SECRET`, `CELCOIN_API_KEY`, `CELCOIN_WEBHOOK_SECRET`, `META_WHATSAPP_TOKEN`, `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `SENTRY_DSN`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`. Verificar: `.env.example` lista ≥15 variáveis; `apps/api/.env` adicionado ao `.gitignore`.

- [x] **[BE-07]** Implementar middleware global no `main.ts`: `helmet()` (headers de segurança), `ThrottlerGuard` (rate limiting global), `ValidationPipe` com `whitelist: true, forbidNonWhitelisted: true, transform: true`. Verificar: requisição sem Content-Type JSON retorna 400; campo extra no body é rejeitado.

- [x] **[BE-08]** Implementar lock otimista via Prisma middleware: intercepta `UPDATE` nas tabelas com `version`; se `version` do payload ≠ `version` do banco → lança `OptimisticLockException` que o controller mapeia para HTTP 409. Verificar: dois updates concorrentes na mesma entidade com a mesma `version` — o segundo retorna 409.

- [x] **[BE-09]** Implementar `AuditService` injetável: método `log(event: AuditEvent)` grava em `audit.audit_logs` com todos os 13 campos; extrai `ip_address`, `user_agent`, `correlation_id` do request context via `AsyncLocalStorage`. Verificar: após qualquer mutação de caso, registro aparece em `SELECT * FROM audit.audit_logs ORDER BY timestamp DESC LIMIT 1`.

- [x] **[BE-10]** Implementar proteção contra duplo submit via `idempotency_key` (UUID no header `X-Idempotency-Key`): Interceptor NestJS verifica no Redis se a chave já foi processada (TTL 24h); se sim, retorna resultado original sem reprocessar. Verificar: POST com mesmo `X-Idempotency-Key` enviado duas vezes retorna o mesmo resultado sem criar dois registros.

---

## 🖥️ FRONTEND

### Setup do SPA Admin

- [x] **[FE-01]** Inicializar `apps/web/` com Vite 7+ e React 19+, TypeScript strict. Instalar: `@tanstack/react-router` 1+, `@tanstack/react-query` 5+, `zustand` 5+, `tailwindcss` 4, `shadcn/ui`, `framer-motion` 12+, `axios`. Verificar: `pnpm dev --filter web` abre `localhost:5173` sem erros no console.

- [x] **[FE-02]** Implementar layout shell (`AppShell.tsx`) com: `AppTopbar` (h-14 fixo: logo esquerda, breadcrumb centro-esquerda, notificações + avatar direita), `AppSidebar` (w-64 desktop, collapsible para w-16 em telas <1280px, 10 itens de menu com ícone+label, item ativo com fundo `--accent` e borda esquerda `--primary` 4px, hover `--accent`), área de trabalho `flex-1` com padding `p-6`. Verificar: sidebar colapsa automaticamente em viewport <1280px; `pnpm run build` sem erros de TypeScript.

- [x] **[FE-03]** Configurar TanStack Router com rotas base: `/login`, `/login/2fa`, `/recuperar-senha`, `/redefinir-senha`, `/dashboard`, `/pipeline`, `/triagem`, `/negociacao`, `/formalizacao`, `/financeiro`, `/supervisao-ia`, `/usuarios`, `/relatorios`, `/configuracoes`. AuthGuard: rotas autenticadas redirecionam para `/login` se token ausente. Verificar: acessar `/dashboard` sem token redireciona para `/login`; acessar `/login` autenticado redireciona para `/dashboard`.

- [x] **[FE-04]** Implementar `ThemeProvider` com tokens CSS do Brand Guide: variáveis `--primary`, `--accent`, `--muted`, `--card`, `--destructive`, `--radius-xl`, `--radius-lg`. Implementar `QueryClientProvider` com retry 3x e staleTime padrão 30s. Implementar `ToastProvider` (sonner ou shadcn toast) com variantes success/error/warning. Verificar: toast de sucesso e erro renderizam com cores corretas.

- [x] **[FE-05]** Implementar `ErrorBoundary` React envolvendo cada rota de módulo: captura erros de render; exibe tela de fallback com ícone, mensagem "Algo deu errado nesta seção" e botão "Tentar novamente". Implementar interceptor Axios global: 401 → limpa token e redireciona para `/login`; 403 → toast "Sem permissão"; 500 → toast "Erro interno, tente novamente"; 502 → toast "Serviço externo indisponível". Verificar: simular throw em componente dispara ErrorBoundary; 401 da API redireciona para /login.

- [x] **[FE-06]** Implementar sidebar com RBAC do lado frontend: menus sem permissão para o perfil logado são **ocultados** (não desabilitados — RN-004/DEC-002). Lógica baseada no `role` do token JWT. Verificar: usuário com role ANALISTA não vê menus: Financeiro, Supervisão IA, Usuários, Relatórios, Configurações; usuário GESTOR_FINANCEIRO não vê: Triagem, Negociação, Formalização.

- [x] **[FE-07]** Implementar proteção contra duplo submit em todos os botões de ação mutativa: ao clicar → botão desabilitado + spinner; após 5s → exibir "Processando..."; após 30s → reabilitar + toast "A operação demorou mais que o esperado. Tente novamente." Criar componente `<ActionButton>` reutilizável com este comportamento. Verificar: clicar duas vezes rapidamente em qualquer botão não dispara duas requisições.

---

## 🔗 WIRING (Integração Frontend ↔ Backend)

- [x] **[WIRE-01]** Configurar `packages/types/` com tipos TypeScript compartilhados: `UserRole`, `CaseStatus`, `CaseScenario`, exportados de `@rs/types`. Configurar path alias `@rs/*` no tsconfig.base.json. Verificar: `import { UserRole } from '@rs/types'` funciona em `apps/web` e `apps/api`.

- [x] **[WIRE-02]** Implementar healthcheck da API: `GET /health` retorna `{ status: 'ok', db: 'ok', redis: 'ok', mq: 'ok' }` verificando conexões ativas. Configurar Railway para healthcheck em `GET /health/ready` a cada 30s. Verificar: `curl localhost:3001/health` retorna 200 com todos os serviços `ok`; se banco desconectado, retorna `db: 'degraded'`.

---

## ✅ TESTES

- [x] **[TEST-01]** Criar testes unitários para `AuditService.log()`: mock de `PrismaService`; verificar que todos os 13 campos obrigatórios são gravados; verificar que `UPDATE` e `DELETE` em `audit_logs` lançam `ImmutableAuditError`. Cobertura mínima: 85%.

- [x] **[TEST-02]** Criar teste de migração: executar `prisma migrate deploy` em banco de teste limpo; verificar que todas as 20 tabelas + schema audit existem; verificar todos os enums (UserRole, UserExternalStatus, CaseStatus, CaseScenario) estão criados. Verificar: `pnpm test --filter api` passa sem erros.

- [x] **[TEST-03]** Criar teste do lock otimista: criar entidade com `version=1`; atualizar com `version=1` → sucesso, `version` vira 2; atualizar novamente com `version=1` → HTTP 409. Verificar: teste passa.

- [x] **[TEST-04]** Criar teste de idempotência: POST com mesmo `X-Idempotency-Key` duas vezes → segundo retorna resultado do primeiro sem novo registro no banco. Verificar: `SELECT COUNT(*)` em tabela alvo retorna 1 após dois POSTs com mesmo idempotency_key.

---

## 🔍 AUTO-VERIFICAÇÃO S1 (12 checks)

- [x] **[CHECK-01]** Todos os enums criados com valores corretos: UserRole(4), UserExternalStatus(3), CaseStatus(14), CaseScenario(4)
- [x] **[CHECK-02]** Todas as 20 tabelas + audit.audit_logs existem no banco
- [x] **[CHECK-03]** Zero arquivo `.js` ou `.jsx` na codebase — apenas `.ts` e `.tsx`
- [x] **[CHECK-04]** `pnpm dev` inicia API na porta 3001 e SPA na porta 5173 sem erros
- [x] **[CHECK-05]** Layout shell (sidebar + topbar) renderiza; menus ocultos para perfis sem permissão
- [x] **[CHECK-06]** AuthGuard redireciona `/dashboard` sem token para `/login`
- [x] **[CHECK-07]** Redis: helper `setWithTTL` funciona; `set` sem TTL lança erro
- [x] **[CHECK-08]** RabbitMQ: 5 exchanges com DLQ configurados
- [x] **[CHECK-09]** `audit.audit_logs`: INSERT funciona; UPDATE/DELETE bloqueados
- [x] **[CHECK-10]** Glossário consultado: nomes canônicos usados (cases, cedentes, cessionarios, proposals, etc.)
- [x] **[CHECK-11]** Todos os REQs de S1 (REQ-178, 179, 182–192, 250–272) cobertos por ≥1 item
- [x] **[CHECK-12]** `pnpm run typecheck` e `pnpm run lint` passam sem erros
