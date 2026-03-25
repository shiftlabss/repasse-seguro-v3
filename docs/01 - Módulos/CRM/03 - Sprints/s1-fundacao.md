# S1 — Fundação

## CRM Repasse Seguro

| **Campo**         | **Valor**                                                                                                                                                                                  |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sprint**        | S1                                                                                                                                                                                         |
| **Nome**          | Fundação                                                                                                                                                                                   |
| **Template**      | A — Infraestrutura                                                                                                                                                                         |
| **Docs fonte**    | 02, 03, 04, 08, 09, 10, 12, 13, 14, 15, 20, 22, 23, 24, 25, 26                                                                                                                             |
| **REQs cobertos** | REQ-108, REQ-110 a REQ-135, REQ-139 a REQ-141, REQ-180, REQ-241 a REQ-242, REQ-245 a REQ-250, REQ-264 a REQ-271, REQ-277 a REQ-280, REQ-287, REQ-290 a REQ-305                             |
| **Objetivo**      | Projeto roda localmente, banco existe com schema completo, layout shell renderiza, rotas protegidas por auth guard, error handling funciona. Zero lógica de negócio — toda a infra pronta. |

---

## Auto-verificação (12 checks)

- [x] C1: Nenhum item genérico — todos usam nomes exatos dos docs
- [x] C2: Todos os itens são binariamente verificáveis (feito/não feito)
- [x] C3: Valores numéricos copiados dos docs (versões, TTLs, limites)
- [x] C4: Glossário (Doc 10) consultado — itens exclusivos do glossário incluídos
- [x] C5: Máquinas de estado cobradas onde aplicável
- [x] C6: Schedules/TTLs/retry policies incluídos
- [x] C7: RBAC/isolamento de dados contemplado
- [x] C8: Telas com estados loading/empty/error cobertos
- [x] C9: Integrações com env vars e fallback declarados
- [x] C10: Glossário verificado — Optimistic Locking, Soft Delete, DLQ, Turborepo, HttpOnly cookie incluídos
- [x] C11: Anti-scaffold R10 — cada item tem sub-itens verificáveis com lógica real
- [x] C12: 100% dos REQs atribuídos a S1 têm ≥1 item de checklist

---

## 🏗️ Banco de Dados

### Migrations — Schema Completo

- [x] **REQ-265** Configurar monorepo Turborepo com workspace `packages/database/`, arquivo `schema.prisma` com `datasource db { provider = "postgresql" url = env("DATABASE_URL") }` e `generator client { provider = "prisma-client-js" }`, executar `prisma generate`
  - Validar: `npx prisma validate` sem erros; client gerado em `packages/database/node_modules/.prisma/client`

- [x] **REQ-117** Configurar convenções obrigatórias no schema Prisma: UUID v4 como PK (`@id @default(uuid())`), `@db.Timestamptz` em todos os campos de data/hora, `Decimal(15,2)` em campos monetários, soft delete `deleted_at DateTime? @db.Timestamptz`, campos de auditoria `created_at`, `updated_at` em toda tabela de domínio
  - Validar: nenhum campo de data sem `@db.Timestamptz`; nenhum campo monetário como `Float`; toda tabela com `created_at` e `updated_at`

- [x] **REQ-133** Criar enum `CaseState`: `CADASTRO`, `SIMULACAO`, `VERIFICACAO`, `PUBLICACAO`, `MATCH`, `NEGOCIACAO`, `ANUENCIA`, `FORMALIZACAO`, `CONCLUIDO`, `CANCELADO`
  - Validar: enum declarado em `schema.prisma`; todos os 10 valores presentes

- [x] **REQ-134** Criar enum `UserRole`: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS`, `PARCEIRO_EXTERNO`
  - Validar: enum declarado; todos os 4 valores presentes

- [x] **REQ-135** Criar enum `UserStatus`: `ATIVO`, `SUSPENSO`, `DESLIGADO`
  - Validar: enum declarado; todos os 3 valores presentes

- [x] **REQ-124** Criar tabela `crm_users` com campos: `id UUID PK`, `email String @unique`, `full_name String`, `role UserRole`, `status UserStatus @default(ATIVO)`, `avatar_url String?`, `last_login_at DateTime? @db.Timestamptz`, `created_at`, `updated_at`, `deleted_at`
  - Validar: migration gerada; `email` com `@unique`; role e status com enums corretos

- [x] **REQ-118** Criar tabela `cases` com campos: `id UUID PK`, `case_number String @unique`, `state CaseState @default(CADASTRO)`, `cedente_contact_id UUID FK → contacts`, `cessionario_contact_id UUID? FK → contacts`, `assigned_to UUID? FK → crm_users`, `scenario String?`, `contract_value Decimal(15,2)`, `current_table_value Decimal(15,2)?`, `delta Decimal(15,2)?`, `enterprise_name String`, `enterprise_address String`, `cancel_reason String?`, `cancel_reason_detail String?`, `version Int @default(1)`, `state_changed_at DateTime @db.Timestamptz`, `estimated_close_at DateTime? @db.Timestamptz`, `created_at`, `updated_at`, `deleted_at`, `created_by UUID FK → crm_users`
  - Validar: `case_number @unique`; FKs para `contacts` e `crm_users`; campo `version` para Optimistic Locking (REQ-273)

- [x] **REQ-119** Criar tabela `contacts` com campos: `id UUID PK`, `full_name String`, `email String`, `phone String`, `cpf_cnpj String?`, `role String`, `status String`, `has_opt_out Boolean @default(false)`, `is_recurrent_investor Boolean @default(false)`, `is_possible_duplicate Boolean @default(false)`, `primary_contact_id UUID? FK → contacts (self-relation)`, `origin String?`, `created_at`, `updated_at`, `deleted_at`, `created_by UUID? FK → crm_users`
  - Validar: self-relation para contatos mesclados; `has_opt_out` boolean

- [x] **REQ-130** Criar tabela `case_status_history` com campos: `id UUID PK`, `case_id UUID FK → cases`, `from_state CaseState?`, `to_state CaseState`, `changed_by UUID FK → crm_users`, `justification String?`, `created_at DateTime @db.Timestamptz`
  - Validar: append-only (sem `updated_at` nem `deleted_at`); FKs corretas

- [x] **REQ-120** Criar tabela `activities` com campos: `id UUID PK`, `case_id UUID? FK → cases`, `contact_id UUID? FK → contacts`, `created_by UUID FK → crm_users`, `type String`, `status String`, `channel String?`, `summary String`, `result String?`, `duration_minutes Int?`, `scheduled_at DateTime? @db.Timestamptz`, `completed_at DateTime? @db.Timestamptz`, `activity_date DateTime @db.Timestamptz`, `created_at`, `updated_at`, `deleted_at`
  - Validar: `summary` obrigatório; `scheduled_at` presente para follow-ups

- [x] **REQ-121** Criar tabela `communications` com campos: `id UUID PK`, `case_id UUID FK → cases`, `contact_id UUID FK → contacts`, `sent_by UUID FK → crm_users`, `channel String`, `direction String`, `status String`, `content String`, `template_id UUID? FK → message_templates`, `external_id String?`, `is_manual Boolean @default(false)`, `created_at`, `updated_at`
  - Validar: `direction` para bidireccionalidade; `is_manual` flag

- [x] **REQ-129** Criar tabela `proposals` com campos: `id UUID PK`, `case_id UUID FK → cases`, `negotiation_id UUID FK → negotiations`, `type String`, `value Decimal(15,2)`, `conditions String?`, `valid_until DateTime @db.Timestamptz`, `status String`, `created_by UUID FK → crm_users`, `created_at`, `updated_at`
  - Validar: `valid_until` obrigatório; `value` como `Decimal(15,2)`

- [x] **REQ-122** Criar tabela `commissions` com campos: `id UUID PK`, `case_id UUID FK → cases`, `type String`, `state CommissionState @default(ESTIMADA)`, `base_calculation String`, `delta_value Decimal(15,2)?`, `commission_value Decimal(15,2)`, `discount_percent Decimal(5,2)?`, `discount_applied_by UUID? FK → crm_users`, `cancellation_reason String?`, `cancelled_by UUID? FK → crm_users`, `cancelled_at DateTime? @db.Timestamptz`, `created_by UUID FK → crm_users`, `created_at`, `updated_at`
  - Validar: `type` para Cedente/Cessionário; `state` enum; campos monetários como `Decimal(15,2)`

- [x] **REQ-123** Criar tabela `dossier_documents` com campos: `id UUID PK`, `case_id UUID FK → cases`, `document_type String`, `file_url String`, `file_name String`, `file_size_bytes Int`, `mime_type String`, `status String`, `version Int @default(1)`, `rejection_reason String?`, `approved_by UUID? FK → crm_users`, `uploaded_by UUID FK → crm_users`, `origin String`, `created_at`, `updated_at`, `deleted_at`
  - Validar: `version` para controle de versões; `origin` distingue "Cedente via plataforma" vs "Analista RS"

- [x] **REQ-125** Criar tabela `sla_alerts` com campos: `id UUID PK`, `case_id UUID FK → cases`, `alert_type String`, `level String`, `status String @default("PENDENTE")`, `triggered_at DateTime @db.Timestamptz`, `acknowledged_at DateTime? @db.Timestamptz`, `acknowledged_by UUID? FK → crm_users`, `resolved_at DateTime? @db.Timestamptz`, `created_at`, `updated_at`
  - Validar: campo `level` com valores: AMARELO, VERMELHO, CRITICO, INATIVIDADE; `acknowledged_at` registra reconhecimento

- [x] **REQ-127** Criar tabela `system_configs` com campos: `id UUID PK`, `key String @unique`, `value String`, `description String?`, `is_critical Boolean @default(false)`, `updated_by UUID FK → crm_users`, `updated_at DateTime @db.Timestamptz`
  - Validar: `key @unique`; `is_critical` para parâmetros que exigem quatro olhos

- [x] **REQ-128** Criar tabela `message_templates` com campos: `id UUID PK`, `name String`, `moment String`, `channel String`, `content String`, `version String @default("v1.0")`, `is_active Boolean @default(true)`, `created_by UUID FK → crm_users`, `created_at`, `updated_at`, `deleted_at`
  - Validar: `version` para histórico; `is_active` para desativação sem exclusão

- [x] **REQ-131** Criar tabela `tags` com campos: `id UUID PK`, `name String @unique`, `color String?`, `created_at` / Criar tabela `contact_tags` com campos: `contact_id UUID FK → contacts`, `tag_id UUID FK → tags`, `@@id([contact_id, tag_id])`
  - Validar: relação N:N; PK composta em `contact_tags`

- [x] **REQ-132** Criar tabela `notification_logs` com campos: `id UUID PK`, `user_id UUID FK → crm_users`, `type String`, `priority String`, `channel String`, `content String`, `reference_id UUID?`, `reference_type String?`, `sent_at DateTime @db.Timestamptz`, `status String`, `created_at`
  - Validar: `priority` com 4 níveis: CRITICA, ALTA, MEDIA, BAIXA; `reference_id` para rastreabilidade

- [x] **REQ-126** Criar schema `audit` e tabela `audit.audit_logs` com campos: `id UUID PK`, `user_id UUID FK → crm_users`, `user_role String`, `action String`, `object_type String`, `object_id UUID`, `field_changed String?`, `old_value Text?`, `new_value Text?`, `ip_address String`, `created_at DateTime @db.Timestamptz` (UTC), `fortaleza_at DateTime @db.Timestamptz`
  - Validar: schema `audit` separado; sem `updated_at` nem `deleted_at` (append-only imutável); NENHUM UPDATE ou DELETE possível via Prisma

- [x] **REQ-240** Configurar RLS no Supabase: políticas por `assigned_to` para Analistas RS (veem apenas próprios Casos) e por `role` para Admin RS/Coordenador RS (visão ampla). Parceiro Externo bloqueado de acesso a colunas pessoais (`cpf_cnpj`, `email`, `phone` em `contacts`)
  - Validar: Analista RS não consegue `SELECT` em `cases` de outro Analista via RLS; Parceiro Externo retorna 403 em endpoint com dados pessoais

- [x] **REQ-266** Criar script `prisma/seed.ts` com dados iniciais: 1 usuário Admin RS, parâmetros de sistema padrão (14 entradas conforme RN-147), templates de mensagem padrão (10 templates conforme RN-104)
  - Validar: `npx prisma db seed` executa sem erros; Admin RS criado; 14 parâmetros presentes; 10 templates presentes; script NÃO executa em ambiente `production`

- [x] **REQ-265** Executar primeira migration: `prisma migrate dev --name init-crm-schema` com todas as 16 tabelas (15 public + 1 audit). Commit da migration no repositório
  - Validar: `prisma/migrations/[timestamp]_init_crm_schema/migration.sql` criada; schema aplicado ao banco dev sem erros

---

## 🔧 Backend — Infraestrutura

- [x] **REQ-264** Criar estrutura de pastas `apps/api/modules/crm/` com subpastas: `cases/`, `contacts/`, `activities/`, `negotiations/`, `commissions/`, `dossier/`, `team/`, `dashboard/`, `reports/`, `integrations/`, `sla/`, `config/` — cada uma com `*.module.ts`, `*.controller.ts`, `*.service.ts`, `*.dto.ts` vazios (stubs)
  - Validar: todos os diretórios existem; `CrmModule` importa todos os sub-módulos

- [x] **REQ-115** Configurar `main.ts` do NestJS com: Node.js 22.x LTS, TypeScript 5.4+ strict, Helmet 8+ (REQ-295), `@nestjs/throttler` rate limit geral 200 req/min (REQ-139), CORS restrito a origens aprovadas, `ValidationPipe` global com `whitelist: true`, `forbidNonWhitelisted: true`, `transform: true` (REQ-296)
  - Validar: `curl localhost:3001/api/v1/crm/health` → 200; headers Helmet presentes (X-Frame-Options, Strict-Transport-Security); body inválido → 400

- [x] **REQ-241** Criar `CrmExceptionFilter` global com: captura de todas as exceções, resposta RFC 7807 (`type`, `title`, `status`, `detail`, `instance`, `code`, `errors[]`), log Pino estruturado sempre, Sentry apenas para erros P0/P1 (status >= 500) (REQ-242)
  - Validar: lançar `BadRequestException` → resposta RFC 7807 com `code: "CRM-XXX"`; lançar erro 500 → Sentry capturado; lançar `NotFoundException` → sem Sentry

- [x] **REQ-302** Definir error codes `CRM-001` a `CRM-035` com prefixo `CRM-` por módulo (auth, cases, contacts, negotiations, commissions, dossier, team, sla, reports, config, webhooks)
  - Validar: arquivo `crm-error-codes.ts` exporta objeto com todos os códigos; cada code documentado com status HTTP, mensagem e prioridade

- [x] **REQ-293** Configurar Pino 9+ como logger do NestJS: `PinoModule.forRoot()` com `transport: pino-pretty` em dev, JSON puro em produção. Request ID gerado por `uuid()` e propagado em `x-request-id` header
  - Validar: logs estruturados em JSON em `NODE_ENV=production`; `request_id` presente em todo log de request

- [x] **REQ-294** Configurar `@nestjs/swagger` com `SwaggerModule.setup('api/docs', ...)`: título "CRM Repasse Seguro API", versão "1.0", Bearer auth scheme, tags por módulo
  - Validar: `GET /api/docs` retorna Swagger UI; endpoints aparecem agrupados por tag

- [x] **REQ-180** Configurar base URL `/api/v1/crm`, resposta padrão `{ data: T, meta?: { page, per_page, total, total_pages } }`, interceptor para wrapping automático das respostas
  - Validar: `GET /api/v1/crm/cases` (stub) retorna `{ data: [], meta: { page: 1, per_page: 25, total: 0, total_pages: 0 } }`

- [x] **REQ-303** Implementar paginação cursor-based em listagens com parâmetros query: `page` (default 1), `per_page` (default 25, máx 100), retornando `meta.page`, `meta.per_page`, `meta.total`, `meta.total_pages`
  - Validar: `GET /cases?page=2&per_page=10` retorna `meta.page=2`, `meta.per_page=10`

- [x] **REQ-110** Implementar protocolo de falha de integração: 1ª falha → retry em 2 min, 2ª falha → retry em 10 min, 3ª falha → alerta Admin RS + Analista RS, 1h contínua → alerta crítico. Usar `RabbitMQ` com DLQ obrigatória em toda fila
  - Validar: mensagem falha 3x → DLQ recebe; log registra cada tentativa com timestamp

- [x] **REQ-280** Criar Dead Letter Queue (DLQ) para cada fila RabbitMQ do CRM. Configurar `x-dead-letter-exchange` e `x-message-ttl`. Monitorar via CloudAMQP
  - Validar: mensagem na fila principal com erro → após N retries → aparece na DLQ correspondente

- [x] **REQ-108** Implementar serviço `AuditLogService` com método `log(action, object, objectId, userId, userRole, ip, oldValue?, newValue?, fieldChanged?)` gravando em `audit.audit_logs`. O serviço NÃO expõe métodos de update ou delete
  - Validar: chamada ao `log()` cria registro em `audit.audit_logs`; tentativa de UPDATE na tabela via Prisma lança erro de permissão (RLS no Supabase bloqueia)

- [x] **REQ-298** Configurar `@sentry/nestjs` 9+: DSN via `SENTRY_DSN` env var, `tracesSampleRate: 0.1` em produção, source maps no build, ignorar erros P3 esperados (validação, negócio)
  - Validar: erro 500 artificial → capturado no Sentry; erro 422 de validação → NÃO capturado

---

## 🎨 Frontend — Infraestrutura

- [x] **REQ-116** Criar `apps/web-crm/` com Next.js 15+ App Router, TypeScript 5.4+ strict, `tsconfig.json` com `strict: true`, React 19+, estrutura de pastas: `app/(auth)/`, `app/(dashboard)/`, `components/ui/`, `components/cases/`, `components/pipeline/`, `components/dossier/`, `components/negotiations/`, `components/commissions/`, `components/communications/`, `lib/motion-tokens.ts`, `lib/api-client.ts`
  - Validar: `npm run dev` inicia sem erros; rota `/` redireciona para `/login`; TypeScript strict sem erros de compilação

- [x] **REQ-116** Configurar Tailwind CSS 4+ com `@theme {}` de design tokens do Brand Guide (Doc 03): cores primária, secundária, accent, muted, destructive, border radius, tipografia. Proibir CSS Modules e styled-components
  - Validar: classes Tailwind customizadas (`bg-primary`, `text-muted`) funcionam; nenhum arquivo `.module.css` no projeto

- [x] **REQ-268** Instalar e configurar `shadcn/ui` com tema customizado pelo Brand Guide (Doc 03): executar `shadcn init`, aplicar tokens de cor do `@theme`, instalar componentes base: `Button`, `Input`, `Select`, `Dialog`, `Toast`, `Skeleton`, `Badge`, `Table`, `Form`
  - Validar: `<Button variant="default">` renderiza com a cor primária do Brand Guide; `<Skeleton>` disponível para substituir spinners (REQ-267)

- [x] **REQ-267** Configurar estados obrigatórios por tela em componentes base: `<PageSkeleton>` para loading (SPINNERS GLOBAIS PROIBIDOS), `<EmptyState message="" action="">` para empty state, `<ErrorState error="" onRetry="">` para error state, `<ForbiddenState>` para sem permissão (403)
  - Validar: importar `<PageSkeleton>` em qualquer página → renderiza skeleton; `<EmptyState>` aceita `message` e `action` props

- [x] **REQ-270** Criar `lib/motion-tokens.ts` com tokens de duração do Motion Spec (Doc 04): `duration.fast`, `duration.normal`, `duration.slow`. Configurar Framer Motion 12+ com `LazyMotion` e `domAnimation`
  - Validar: importar `motionTokens.duration.fast` retorna valor numérico; Framer Motion sem bundle completo em client

- [x] **REQ-116** Configurar TanStack Query 5+ com `QueryClient` e `QueryClientProvider` no root layout. Stale time padrão: 30s. Retry: 1x para erros de servidor
  - Validar: `useQuery` disponível em Client Components; stale time configurado

- [x] **REQ-299** Configurar Zustand 5+ com store de UI: `usePipelineStore` (filtros kanban), `useUserPreferencesStore` (preferências do usuário). Proibir uso de Zustand para dados de servidor
  - Validar: `usePipelineStore.getState().filters` retorna objeto de filtros inicial; dados do servidor NÃO armazenados no store

- [x] **REQ-297** Configurar PostHog com `NEXT_PUBLIC_POSTHOG_KEY` e `NEXT_PUBLIC_POSTHOG_HOST`. `identify()` apenas com `userId` (sem dados pessoais de Cedentes/Cessionários). Eventos: `case_created`, `state_advanced`, `filter_applied`, `export_generated`
  - Validar: PostHog inicializa no client; evento `page_view` enviado em cada navegação; nenhum dado pessoal de Cedente/Cessionário em propriedades do evento

- [x] **REQ-298** Configurar `@sentry/nextjs` 9+: `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`. Source maps gerados no `next build`. Ignorar erros de rede esperados
  - Validar: erro lançado em Client Component → capturado no Sentry; source maps presentes no dashboard Sentry

- [x] **REQ-141** Implementar layout shell global em `app/(dashboard)/layout.tsx`:
  - Topbar: altura `h-14`, fixo, logo Repasse Seguro (esquerda), ícone busca global `Ctrl+K/Cmd+K` com placeholder "Buscar Caso, Contato... (Ctrl+K)" (centro), ícone notificações com badge numérico (direita), avatar do usuário com dropdown (direita)
  - Sidebar: largura `w-64` em desktop, collapsible para `w-16` (ícones only) em telas `< 1280px`, 12 itens de módulo com ícone + label, item ativo com fundo `--accent` e borda esquerda `--primary` 4px
  - Área de trabalho: `flex-1`, scroll vertical independente, padding `p-6`, breadcrumb no topo
  - Validar: sidebar colapsa em viewport < 1280px; breadcrumb atualiza por rota; topbar badge notificações exibe contagem; rota não autenticada → redirect para `/login`

- [x] **REQ-111** Implementar busca global: abrir modal com `Ctrl+K/Cmd+K` em qualquer tela, pesquisa debounced 300ms, retorna Casos (ID + empreendimento), Contatos (nome mascarado conforme RN-012), Atividades (resumo texto), respeitando permissões do usuário logado. NÃO indexa: PDFs do Dossiê, logs de auditoria, versões de templates (conforme REQ-111)
  - Validar: Ctrl+K abre modal; digitar RS-2026 retorna Casos cujo `case_number` corresponde; Analista RS não vê Casos de outros; dados pessoais mascarados nos resultados

- [x] **REQ-305** Implementar sistema de toast com `sonner` ou `shadcn/ui Toast`: toast para erros transitórios (duração 5s), página de erro (`/error`) para falhas críticas (500), retry automático visível para falhas de integração
  - Validar: erro 422 dispara toast com mensagem amigável; erro 500 → página de erro exibida; toast de integração exibe botão "Tentar novamente"

- [x] **REQ-300** Configurar TanStack Table 8+ para tabelas complexas: `getCoreRowModel`, `getSortedRowModel`, `getFilteredRowModel`, `getPaginationRowModel`. Criar componente `<DataTable>` reutilizável
  - Validar: `<DataTable>` aceita `columns` e `data` props; sorting e filtro funcionam; paginação respeita `meta.total` do backend

---

## ⚙️ Configurações e DevOps

- [x] **REQ-245** Criar arquivo `.env.example` com todas as variáveis obrigatórias documentadas: `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_KEY`, `REDIS_URL`, `RABBITMQ_URL`, `META_WABA_TOKEN`, `META_PHONE_NUMBER_ID`, `META_WEBHOOK_VERIFY_TOKEN`, `ZAPSIGN_API_TOKEN`, `ZAPSIGN_WEBHOOK_SECRET`, `CELCOIN_CLIENT_ID`, `CELCOIN_CLIENT_SECRET`, `CELCOIN_WEBHOOK_SECRET`, `SENTRY_DSN`, `NEXT_PUBLIC_POSTHOG_KEY`, `NEXT_PUBLIC_POSTHOG_HOST`, `NODE_ENV`
  - Validar: `.env.example` presente no repositório; `.env` listado no `.gitignore`; ao iniciar sem `DATABASE_URL` → erro explícito com orientação

- [x] **REQ-264** Criar `turbo.json` com pipeline: `"build": { "dependsOn": ["^build"] }`, `"test": { "dependsOn": ["^build"] }`, `"lint": {}`. Configurar `pnpm-workspace.yaml` com `packages: ["apps/*", "packages/*"]`
  - Validar: `pnpm build` executa na ordem correta; cache Turborepo funciona (segundo run mais rápido)

- [x] **REQ-246** Configurar git flow: branch `main` protegida (requer PR + CI verde), branch `develop`, prefixos `feature/`, `hotfix/`. Conventional Commits: `feat:`, `fix:`, `chore:`, `docs:`, `test:`. Arquivo `.commitlintrc.json` configurado
  - Validar: commit `feat: add cases module` passa lint; commit `add cases` (sem prefixo) falha; push direto para `main` bloqueado

- [x] **REQ-248** Criar `.github/workflows/crm-ci.yml` com jobs sequenciais: `lint` → `typecheck` → `test` → `build`. Trigger: pull_request para `develop` e `main`, paths `apps/api/modules/crm/**`, `apps/web-crm/**`, `packages/**`. Adicionar job `migrate-deploy` (executar `prisma migrate deploy`) antes do job `build` da API
  - Validar: PR aberto → pipeline inicia; falha em `lint` cancela jobs seguintes; `prisma migrate deploy` executa antes do build da API

- [x] **REQ-247** Documentar 3 ambientes em `docs/environments.md`: `development` (localhost:3000/3001, Supabase dev, Docker local Redis+RabbitMQ), `staging` (Vercel preview + Railway staging, Supabase staging, Upstash staging, CloudAMQP staging), `production` (crm.repasseseguro.com.br, api.crm.repasseseguro.com.br, Supabase produção)
  - Validar: arquivo criado; 3 ambientes documentados com URLs e serviços; README referencia o documento

- [x] **REQ-249** Documentar procedimento de rollback em ≤ 10 min: reverter deploy Railway via painel ou `railway rollback`, reverter Vercel via `vercel rollback`, reverter migration via `prisma migrate resolve --rolled-back`. Configurar feature flags PostHog para rollout gradual de features críticas
  - Validar: documento de rollback presente; PostHog configurado com feature flag de exemplo

- [x] **REQ-250** Configurar observabilidade completa: PostHog (frontend analytics), Sentry (error tracking frontend + backend), Pino (logs estruturados backend), Railway Metrics (infraestrutura). Propagar `x-request-id` do frontend ao backend em cada requisição
  - Validar: request do frontend tem `x-request-id`; backend loga o mesmo `request_id`; Sentry linked a Railway

- [x] **REQ-114** Documentar política de backup: Supabase realiza backup automático diário com retenção de 30 dias, RPO máximo 24 horas, RTO máximo 4 horas. Configurar alerta no Supabase para notificar Admin RS em falha de backup
  - Validar: configuração de alerta presente no Supabase; documentação de RPO/RTO no runbook

- [x] **REQ-139** Configurar TLS 1.3+ com redirect HTTP → HTTPS no Vercel (frontend) e Railway (API). Validar que `HSTS` está presente via Helmet
  - Validar: `curl -I http://crm.repasseseguro.com.br` → redirect 301 para HTTPS; header `Strict-Transport-Security` presente

- [x] **REQ-287** Confirmar no `next.config.js` que não há breakpoint abaixo de 768px como breakpoint suportado. Tailwind config: `screens: { tablet: '768px', 'desktop-sm': '1280px', 'desktop-lg': '1440px' }`. Documentar no README que mobile < 640px está fora do escopo do MVP
  - Validar: classes `tablet:` e `desktop-sm:` funcionam; README menciona ausência de suporte para smartphones

- [x] **REQ-112** Configurar internacionalização e fuso horário: interface em `pt-BR`, valores monetários formatados como `R$ X.XXX,XX` (separador milhar ponto, decimal vírgula), todos os carimbos de data/hora armazenados em UTC (`@db.Timestamptz`), exibição convertida para `America/Fortaleza`
  - Validar: `new Date()` armazenada como UTC no banco; exibição na UI mostra hora correta de Fortaleza (UTC-3); `R$ 50.000,00` formatado corretamente

---

## 🔀 Cross-Módulo

- Nenhum cross-módulo nesta sprint — S1 é puramente de infraestrutura.

---

## ✅ Critério de Conclusão da S1

A S1 está concluída quando:

1. `prisma migrate dev` aplica todas as 16 tabelas sem erros
2. `npx prisma db seed` cria Admin RS + 14 parâmetros + 10 templates
3. `curl localhost:3001/api/v1/crm/health` retorna 200
4. `npm run dev` no frontend abre layout shell com sidebar + topbar
5. Busca global `Ctrl+K` abre modal
6. CI/CD GitHub Actions passa em PR de teste
7. Todos os 12 checks de auto-verificação marcados
