# 15 - Arquitetura de Pastas

## Repasse Seguro — Monorepo

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — Frontend, Backend e Mobile |
| **Escopo** | Estrutura de pastas do monorepo completo: apps/web, apps/api, apps/mobile e packages compartilhados |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Dependências** | D02 Stacks · D09 Contratos de UI · D13 Schema Prisma · D14 Especificações Técnicas |

---

> 📌 **TL;DR**
>
> - **Monorepo com pnpm workspaces.** 3 apps: `web` (Admin SPA), `api` (NestJS), `mobile` (Expo).
> - **Frontend:** feature-based com colocação de arquivos — cada feature tem pages, components, hooks, stores e services juntos.
> - **Backend:** modular NestJS com um diretório por domínio — controller, service, repository, DTOs.
> - **Prisma:** centralizado em `apps/api/prisma/` com migrations, seed, RLS e middleware.
> - **Packages compartilhados:** `@rs/types` (TypeScript types), `@rs/utils` (helpers), `@rs/ui` (componentes base).
> - Convenção de nomenclatura: `kebab-case` para arquivos, `PascalCase` para componentes e classes.

---

## 1. Estrutura Raiz do Monorepo

```
repasse-seguro/
├── apps/
│   ├── web/                    # Admin SPA — React 19 + Vite 7
│   ├── api/                    # Backend — NestJS 10
│   └── mobile/                 # App Mobile — React Native + Expo SDK 52
├── packages/
│   ├── types/                  # @rs/types — Tipos TypeScript compartilhados
│   ├── utils/                  # @rs/utils — Helpers e funções utilitárias
│   └── ui/                     # @rs/ui — Componentes base compartilhados (futuro)
├── docs/                       # Documentação do projeto (este pipeline)
├── .github/
│   ├── workflows/              # CI/CD GitHub Actions
│   └── CODEOWNERS
├── .env.example                # Template de variáveis de ambiente
├── package.json                # Root — scripts de monorepo
├── pnpm-workspace.yaml
├── turbo.json                  # Turborepo task pipeline
└── tsconfig.base.json          # TypeScript base config compartilhada
```

---

## 2. App Web — Admin SPA (`apps/web/`)

```
apps/web/
├── public/
│   ├── favicon.ico
│   └── robots.txt
├── src/
│   ├── app/                    # Configuração raiz da aplicação
│   │   ├── App.tsx             # Root component
│   │   ├── providers.tsx       # QueryClientProvider, ThemeProvider, ToastProvider
│   │   └── router.tsx          # TanStack Router — definição de rotas
│   │
│   ├── features/               # Módulos de feature (colocação de arquivos)
│   │   ├── auth/
│   │   │   ├── pages/
│   │   │   │   ├── LoginPage.tsx
│   │   │   │   ├── TwoFaPage.tsx
│   │   │   │   ├── ForgotPasswordPage.tsx
│   │   │   │   └── ResetPasswordPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   └── PasswordStrengthIndicator.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useAuth.ts
│   │   │   ├── services/
│   │   │   │   └── auth.service.ts
│   │   │   └── stores/
│   │   │       └── auth.store.ts
│   │   │
│   │   ├── dashboard/
│   │   │   ├── pages/
│   │   │   │   ├── DashboardPage.tsx
│   │   │   │   └── AlertsPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── KpiCard.tsx
│   │   │   │   ├── CaseVolumeChart.tsx
│   │   │   │   └── RecentCasesTable.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useDashboardKpis.ts
│   │   │   └── services/
│   │   │       └── dashboard.service.ts
│   │   │
│   │   ├── pipeline/
│   │   │   ├── pages/
│   │   │   │   ├── PipelineKanbanPage.tsx
│   │   │   │   ├── PipelineListPage.tsx
│   │   │   │   ├── CaseDetailPage.tsx
│   │   │   │   ├── CreateCasePage.tsx     # Drawer inline
│   │   │   │   └── CaseCancelPage.tsx     # Modal inline
│   │   │   ├── components/
│   │   │   │   ├── PipelineCard.tsx
│   │   │   │   ├── KanbanColumn.tsx
│   │   │   │   ├── CaseStatusBadge.tsx
│   │   │   │   ├── SlaTimer.tsx
│   │   │   │   └── CaseStatusHistory.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── usePipelineCases.ts
│   │   │   │   └── useCaseDetail.ts
│   │   │   ├── services/
│   │   │   │   └── cases.service.ts
│   │   │   └── stores/
│   │   │       └── pipeline.store.ts
│   │   │
│   │   ├── triagem/
│   │   │   ├── pages/
│   │   │   │   ├── TriagemQueuePage.tsx
│   │   │   │   ├── TriagemAnalysisPage.tsx
│   │   │   │   ├── TriagemHistoryPage.tsx
│   │   │   │   └── TriagemSlaPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── TriagemCard.tsx
│   │   │   │   └── BlockCaseModal.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useTriagemQueue.ts
│   │   │   └── services/
│   │   │       └── triagem.service.ts
│   │   │
│   │   ├── negotiation/
│   │   │   ├── pages/
│   │   │   │   ├── NegotiationListPage.tsx
│   │   │   │   ├── NegotiationDetailPage.tsx
│   │   │   │   ├── CreateProposalPage.tsx
│   │   │   │   ├── CounterproposalPage.tsx
│   │   │   │   ├── ApproveDeltaPage.tsx
│   │   │   │   ├── NegotiationHistoryPage.tsx
│   │   │   │   ├── CreateCessionarioPage.tsx
│   │   │   │   └── EditCessionarioPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── ProposalStatusBadge.tsx
│   │   │   │   ├── DeltaBadge.tsx
│   │   │   │   └── ProposalHistory.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useNegotiation.ts
│   │   │   └── services/
│   │   │       └── negotiation.service.ts
│   │   │
│   │   ├── formalization/
│   │   │   ├── pages/
│   │   │   │   ├── FormalizationOverviewPage.tsx
│   │   │   │   ├── DossieDocumentsPage.tsx
│   │   │   │   ├── UploadDocumentPage.tsx
│   │   │   │   ├── ZapsignEnvelopesPage.tsx
│   │   │   │   ├── CreateEnvelopePage.tsx
│   │   │   │   ├── ClosingCriteriaPage.tsx
│   │   │   │   ├── ReversalPage.tsx
│   │   │   │   ├── MediationPage.tsx
│   │   │   │   └── FormalDisputePage.tsx
│   │   │   ├── components/
│   │   │   │   ├── DocStatusBadge.tsx
│   │   │   │   ├── EnvelopeStatusBadge.tsx
│   │   │   │   ├── ClosingCriteriaCard.tsx
│   │   │   │   └── DossieProgress.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useFormalization.ts
│   │   │   └── services/
│   │   │       └── formalization.service.ts
│   │   │
│   │   ├── financial/
│   │   │   ├── pages/
│   │   │   │   ├── FinancialDashboardPage.tsx
│   │   │   │   ├── EscrowDetailPage.tsx
│   │   │   │   ├── DistributionPage.tsx
│   │   │   │   ├── BlockDistributionPage.tsx
│   │   │   │   ├── ApproveBlockPage.tsx
│   │   │   │   ├── CommissionsPage.tsx
│   │   │   │   ├── TransactionExtractPage.tsx
│   │   │   │   └── FinancialReportPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── EscrowStatusBadge.tsx
│   │   │   │   ├── EscrowGauge.tsx
│   │   │   │   └── DistributionBreakdown.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useEscrow.ts
│   │   │   └── services/
│   │   │       └── financial.service.ts
│   │   │
│   │   ├── ai-supervision/
│   │   │   ├── pages/
│   │   │   │   ├── AiDashboardPage.tsx
│   │   │   │   ├── AiDecisionDetailPage.tsx
│   │   │   │   ├── AiAgentConfigPage.tsx
│   │   │   │   ├── AiHistoryPage.tsx
│   │   │   │   ├── AiAlertsPage.tsx
│   │   │   │   └── AiGlobalParamsPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── AiConfidenceBadge.tsx
│   │   │   │   └── AiDecisionOutcomeBadge.tsx
│   │   │   └── services/
│   │   │       └── ai-supervision.service.ts
│   │   │
│   │   ├── users/
│   │   │   ├── pages/
│   │   │   │   ├── OperatorsListPage.tsx
│   │   │   │   ├── CreateOperatorPage.tsx
│   │   │   │   ├── EditOperatorPage.tsx
│   │   │   │   ├── DeactivateOperatorPage.tsx
│   │   │   │   └── CedentesListPage.tsx
│   │   │   ├── components/
│   │   │   │   ├── RoleBadge.tsx
│   │   │   │   └── ExternalStatusBadge.tsx
│   │   │   └── services/
│   │   │       └── users.service.ts
│   │   │
│   │   ├── reports/
│   │   │   ├── pages/
│   │   │   │   ├── ReportsHubPage.tsx
│   │   │   │   ├── OperationalReportPage.tsx
│   │   │   │   ├── AnalystPerformancePage.tsx
│   │   │   │   └── ExportPage.tsx
│   │   │   └── services/
│   │   │       └── reports.service.ts
│   │   │
│   │   └── settings/
│   │       ├── pages/
│   │       │   ├── SettingsHubPage.tsx
│   │       │   ├── FinancialParamsPage.tsx
│   │       │   ├── SlaSettingsPage.tsx
│   │       │   └── IntegrationsPage.tsx
│   │       └── services/
│   │           └── settings.service.ts
│   │
│   ├── components/              # Componentes globais reutilizáveis
│   │   ├── layout/
│   │   │   ├── AppLayout.tsx
│   │   │   ├── AuthLayout.tsx
│   │   │   ├── ErrorLayout.tsx
│   │   │   ├── AppTopbar.tsx
│   │   │   └── AppSidebar.tsx
│   │   ├── ui/                  # shadcn/ui — gerados via CLI (não editar manualmente)
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── table.tsx
│   │   │   ├── skeleton.tsx
│   │   │   └── ...
│   │   └── shared/              # Componentes custom globais
│   │       ├── DataTable.tsx
│   │       ├── EmptyState.tsx
│   │       ├── ConfirmModal.tsx
│   │       ├── GlobalSearch.tsx
│   │       └── NotificationBadge.tsx
│   │
│   ├── hooks/                   # Hooks globais
│   │   ├── useRbac.ts           # RBAC — verifica role do usuário
│   │   ├── useRealtime.ts       # Supabase Realtime subscriptions
│   │   └── useToast.ts
│   │
│   ├── lib/                     # Utilitários e configurações de bibliotecas
│   │   ├── api-client.ts        # Axios instance com interceptors JWT
│   │   ├── supabase.ts          # Supabase client
│   │   ├── query-client.ts      # TanStack Query client config
│   │   └── router.ts            # TanStack Router instance
│   │
│   ├── stores/                  # Zustand stores globais
│   │   ├── auth.store.ts
│   │   └── ui.store.ts
│   │
│   ├── styles/
│   │   ├── globals.css          # Tailwind + CSS vars (tokens do D03)
│   │   └── themes.css
│   │
│   └── types/
│       └── index.ts             # Re-exports de @rs/types
│
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── components.json              # shadcn/ui config
├── tsconfig.json
└── package.json
```

---

## 3. App API — NestJS (`apps/api/`)

```
apps/api/
├── prisma/
│   ├── schema.prisma            # Schema Prisma (D13)
│   ├── seed.ts                  # Seed de desenvolvimento
│   ├── rls/
│   │   ├── policies.sql
│   │   └── indexes.sql
│   └── middleware/
│       ├── soft-delete.middleware.ts
│       └── audit.middleware.ts
│
├── src/
│   ├── main.ts                  # Bootstrap NestJS
│   ├── app.module.ts            # Root module
│   │
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.module.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── strategies/
│   │   │   │   ├── jwt.strategy.ts
│   │   │   │   └── refresh.strategy.ts
│   │   │   └── dto/
│   │   │       ├── login.dto.ts
│   │   │       ├── verify-2fa.dto.ts
│   │   │       └── auth-response.dto.ts
│   │   │
│   │   ├── users/
│   │   │   ├── users.module.ts
│   │   │   ├── users.controller.ts
│   │   │   ├── users.service.ts
│   │   │   └── dto/
│   │   │       ├── create-user.dto.ts
│   │   │       ├── update-user.dto.ts
│   │   │       └── user-response.dto.ts
│   │   │
│   │   ├── cedentes/
│   │   │   ├── cedentes.module.ts
│   │   │   ├── cedentes.controller.ts
│   │   │   ├── cedentes.service.ts
│   │   │   └── dto/
│   │   │
│   │   ├── cessionarios/
│   │   │   └── ... (idem cedentes)
│   │   │
│   │   ├── cases/
│   │   │   ├── cases.module.ts
│   │   │   ├── cases.controller.ts
│   │   │   ├── cases.service.ts
│   │   │   ├── cases.repository.ts  # Queries Prisma complexas
│   │   │   └── dto/
│   │   │       ├── create-case.dto.ts
│   │   │       ├── update-case-status.dto.ts
│   │   │       └── case-response.dto.ts
│   │   │
│   │   ├── triagem/
│   │   ├── negotiation/
│   │   ├── formalization/
│   │   ├── zapsign/
│   │   │   ├── zapsign.module.ts
│   │   │   ├── zapsign.controller.ts
│   │   │   ├── zapsign.service.ts
│   │   │   └── zapsign-webhook.handler.ts  # RabbitMQ consumer
│   │   │
│   │   ├── escrow/
│   │   ├── commission/
│   │   ├── notifications/
│   │   │   ├── notifications.module.ts
│   │   │   ├── notifications.service.ts
│   │   │   ├── notifications.consumer.ts   # RabbitMQ consumer
│   │   │   └── templates/
│   │   │       ├── whatsapp/
│   │   │       │   └── case-status-update.ts
│   │   │       └── sms/
│   │   │           └── verification-code.ts
│   │   │
│   │   ├── ai-supervision/
│   │   ├── reports/
│   │   ├── configs/
│   │   └── webhooks/
│   │       ├── webhooks.module.ts
│   │       ├── webhooks.controller.ts  # POST /webhooks/zapsign, /webhooks/celcoin
│   │       ├── zapsign-webhook.handler.ts
│   │       └── celcoin-webhook.handler.ts
│   │
│   ├── common/
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   └── rbac.guard.ts
│   │   ├── decorators/
│   │   │   ├── roles.decorator.ts        # @Roles(UserRole.MASTER)
│   │   │   └── current-user.decorator.ts # @CurrentUser()
│   │   ├── interceptors/
│   │   │   ├── audit.interceptor.ts
│   │   │   └── transform.interceptor.ts  # Wrap responses em {data, meta}
│   │   ├── filters/
│   │   │   └── global-exception.filter.ts # Normaliza erros para padrão RFC 7807
│   │   ├── pipes/
│   │   │   └── validation.pipe.ts
│   │   └── middleware/
│   │       └── request-context.middleware.ts
│   │
│   └── prisma/
│       └── prisma.service.ts             # PrismaClient singleton
│
├── test/
│   ├── unit/                             # Jest — testes unitários por módulo
│   │   └── cases/
│   │       └── cases.service.spec.ts
│   ├── integration/                      # Jest — testes de integração
│   │   └── auth.controller.spec.ts
│   └── e2e/                             # Supertest — testes E2E
│       └── app.e2e-spec.ts
│
├── nest-cli.json
├── tsconfig.json
├── tsconfig.build.json
└── package.json
```

---

## 4. App Mobile — Expo (`apps/mobile/`)

```
apps/mobile/
├── app/                         # expo-router — file-based routing
│   ├── (auth)/
│   │   ├── _layout.tsx          # Stack Navigator de auth
│   │   ├── login.tsx            # M-001
│   │   ├── verify-otp.tsx       # M-002
│   │   └── forgot-password.tsx  # M-003
│   │
│   ├── (tabs)/
│   │   ├── _layout.tsx          # Bottom Tab Navigator
│   │   ├── index.tsx            # M-010 — Home
│   │   ├── documents.tsx        # M-020 — Documentos
│   │   ├── notifications.tsx    # M-030 — Notificações
│   │   └── profile.tsx          # M-040 — Perfil
│   │
│   ├── cases/
│   │   ├── [id].tsx             # M-011 — Detalhe do Caso
│   │   └── [id]/
│   │       ├── timeline.tsx     # M-012
│   │       ├── documents.tsx    # M-021
│   │       └── sign.tsx         # M-050 — ZapSign WebView
│   │
│   ├── _layout.tsx              # Root layout — providers
│   └── +not-found.tsx           # M-099
│
├── src/
│   ├── components/
│   │   ├── shared/
│   │   │   ├── CaseStatusBadge.tsx
│   │   │   ├── DocStatusBadge.tsx
│   │   │   ├── NetworkStatusBanner.tsx
│   │   │   ├── SkeletonPlaceholder.tsx
│   │   │   ├── InAppNotificationBanner.tsx
│   │   │   └── EmptyState.tsx
│   │   └── ui/
│   │       ├── Button.tsx
│   │       ├── Input.tsx
│   │       └── Card.tsx
│   │
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useNetwork.ts        # NetInfo — status de conexão
│   │   ├── useBiometrics.ts
│   │   └── usePushNotifications.ts
│   │
│   ├── services/
│   │   ├── api.service.ts       # Axios instance
│   │   ├── auth.service.ts
│   │   ├── cases.service.ts
│   │   ├── documents.service.ts
│   │   └── notifications.service.ts
│   │
│   ├── stores/
│   │   ├── auth.store.ts        # Zustand — sessão e token
│   │   └── ui.store.ts
│   │
│   └── types/
│       └── index.ts
│
├── assets/
│   ├── fonts/
│   ├── images/
│   └── icons/
│
├── app.config.ts                # Expo config (scheme, deep links, plugins)
├── babel.config.js
├── metro.config.js
├── tsconfig.json
└── package.json
```

---

## 5. Packages Compartilhados

### `packages/types` — `@rs/types`

```
packages/types/
├── src/
│   ├── enums/
│   │   ├── case-status.enum.ts
│   │   ├── user-role.enum.ts
│   │   └── ...
│   ├── models/
│   │   ├── case.types.ts
│   │   ├── user.types.ts
│   │   └── ...
│   └── index.ts
└── package.json
```

### `packages/utils` — `@rs/utils`

```
packages/utils/
├── src/
│   ├── formatters/
│   │   ├── currency.ts          # formatBRL(value: number): string
│   │   ├── cpf.ts               # formatCPF, validateCPF
│   │   └── date.ts              # formatRelative, formatBR
│   ├── validators/
│   │   └── cpf.ts
│   └── index.ts
└── package.json
```

---

## 6. Convenções de Nomenclatura

| Contexto | Padrão | Exemplo |
|---|---|---|
| Arquivos de componente | `PascalCase.tsx` | `CaseStatusBadge.tsx` |
| Arquivos de hook | `camelCase.ts` com prefixo `use` | `usePipelineCases.ts` |
| Arquivos de service | `kebab-case.service.ts` | `cases.service.ts` |
| Arquivos de store | `kebab-case.store.ts` | `auth.store.ts` |
| Arquivos de DTO | `kebab-case.dto.ts` | `create-case.dto.ts` |
| Pastas de feature | `kebab-case` | `ai-supervision/` |
| Variáveis e funções | `camelCase` | `caseStatus`, `getUserById` |
| Classes e interfaces | `PascalCase` | `CaseService`, `ICase` |
| Constantes | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Enums (TypeScript) | `PascalCase` (chave e valor) | `CaseStatus.CAPTADO` |
| CSS classes (Tailwind) | `kebab-case` | `bg-primary text-foreground` |

---

## 7. Regras de Importação

```typescript
// CORRETO: importações organizadas em 3 grupos (auto-ordenado por ESLint)
// 1. Externos (node_modules)
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

// 2. Pacotes internos (@rs/*)
import { CaseStatus } from '@rs/types';
import { formatBRL } from '@rs/utils';

// 3. Relativos
import { CreateCaseDto } from './dto/create-case.dto';
```

**Regra:** Nunca importar de `../../` mais de 2 níveis acima. Se necessário, promover para `shared/` ou `packages/`.

**Barrel exports** (`index.ts`): apenas para `packages/` e `src/components/`. Proibido em `features/` para evitar circular dependencies.

---

## 8. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — monorepo pnpm, 3 apps + 2 packages, estrutura feature-based no web, modular NestJS no api, expo-router no mobile. |
