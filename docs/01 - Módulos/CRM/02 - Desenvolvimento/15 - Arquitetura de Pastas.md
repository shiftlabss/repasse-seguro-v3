# 15 - Arquitetura de Pastas

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — Frontend e Backend |
| **Escopo** | Estrutura de pastas do monorepo Turborepo para o módulo CRM: `apps/web-crm` e `apps/api/modules/crm/` |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | Doc 02 Stacks CRM · Doc 01.1 a 01.5 Regras de Negócio CRM |

---

> **TL;DR**
>
> - **Monorepo Turborepo com pnpm workspaces.** CRM ocupa `apps/web-crm` (Next.js 15 App Router) e `apps/api/modules/crm/` (NestJS 10).
> - **Frontend:** route-group-based com App Router — `(auth)` e `(dashboard)`. Componentes por domínio: cases, pipeline, contacts, activities, communications, dossier, negotiations, commissions, sla.
> - **Backend:** módulos NestJS por domínio — cases, contacts, activities, communication, commission, dossier, team, reports, sla, config, webhooks.
> - **Packages compartilhados:** `@rs/types`, `@rs/utils`, `@rs/shared-types` (schemas Zod para frontend/backend).
> - Convenção de nomenclatura: `kebab-case` para arquivos, `PascalCase` para componentes e classes.

---

## 1. Estrutura Raiz do Monorepo

```
repasse-seguro/
├── apps/
│   ├── web-crm/                # CRM — Next.js 15 App Router (ferramenta interna)
│   ├── web/                    # Admin SPA — React 19 + Vite 7
│   ├── api/                    # Backend compartilhado — NestJS 10
│   └── mobile/                 # App Mobile — React Native + Expo SDK 52
├── packages/
│   ├── types/                  # @rs/types — Tipos TypeScript compartilhados
│   ├── utils/                  # @rs/utils — Helpers e funções utilitárias
│   ├── shared-types/           # @rs/shared-types — Schemas Zod (frontend + backend)
│   └── ui/                     # @rs/ui — Componentes base compartilhados (futuro)
├── docs/                       # Documentação do projeto
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

## 2. App Web CRM — Next.js 15 (`apps/web-crm/`)

O CRM é uma ferramenta **100% interna** para a equipe Repasse Seguro. Utiliza Next.js 15 App Router com Server Components para carregamento inicial de Casos (ADR CRM-ADR-001). Sem acesso de Cedentes ou Cessionários.

```
apps/web-crm/
├── public/
│   ├── favicon.ico
│   └── robots.txt              # noindex — ferramenta interna
│
├── app/
│   ├── layout.tsx              # Root layout — providers globais
│   ├── not-found.tsx           # 404 global
│   │
│   ├── (auth)/                 # Route group — telas de autenticação (sem sidebar)
│   │   ├── layout.tsx          # AuthLayout — centralized card
│   │   ├── login/
│   │   │   └── page.tsx        # Login por e-mail/senha (sem OTP — uso interno)
│   │   └── forgot-password/
│   │       └── page.tsx        # Recuperação de senha
│   │
│   └── (dashboard)/            # Route group — área autenticada com sidebar
│       ├── layout.tsx          # DashboardLayout — sidebar + topbar + notification bell
│       │
│       ├── page.tsx            # Painel pessoal do Analista RS (KPIs + casos atribuídos)
│       │
│       ├── pipeline/           # Pipeline de Casos — kanban + lista
│       │   ├── page.tsx        # PipelinePage — view kanban padrão
│       │   └── list/
│       │       └── page.tsx    # PipelineListPage — view tabular
│       │
│       ├── cases/
│       │   ├── new/
│       │   │   └── page.tsx    # CreateCasePage — formulário de criação manual
│       │   └── [id]/
│       │       ├── page.tsx    # CaseDetailPage — visão geral do Caso
│       │       ├── activities/
│       │       │   └── page.tsx # Agenda e histórico de atividades do Caso
│       │       ├── contacts/
│       │       │   └── page.tsx # Contatos vinculados ao Caso
│       │       ├── dossier/
│       │       │   └── page.tsx # Dossiê de documentos
│       │       ├── negotiations/
│       │       │   └── page.tsx # Propostas e contrapropostas
│       │       ├── commission/
│       │       │   └── page.tsx # Comissão estimada e confirmada
│       │       └── communication/
│       │           └── page.tsx # Histórico de mensagens WhatsApp
│       │
│       ├── contacts/           # Gestão global de Contatos
│       │   ├── page.tsx        # ContactsListPage — listagem paginada
│       │   └── [id]/
│       │       └── page.tsx    # ContactDetailPage
│       │
│       ├── activities/         # Agenda e atividades do Analista RS
│       │   └── page.tsx        # ActivitiesPage — agenda semanal + filtros
│       │
│       ├── reports/            # Relatórios e exportações
│       │   ├── page.tsx        # ReportsHubPage — índice de relatórios
│       │   ├── performance/
│       │   │   └── page.tsx    # PerformancePage — métricas por Analista
│       │   └── operational/
│       │       └── page.tsx    # OperationalPage — funil de Casos
│       │
│       ├── team/               # Gestão de equipe (Admin RS e Coordenador RS apenas)
│       │   ├── page.tsx        # TeamPage — lista de membros da equipe
│       │   ├── new/
│       │   │   └── page.tsx    # InviteUserPage
│       │   └── [id]/
│       │       └── page.tsx    # UserDetailPage — perfil e redistribuição de Casos
│       │
│       └── settings/           # Configurações do sistema (Admin RS apenas)
│           ├── page.tsx        # SettingsHubPage
│           ├── sla/
│           │   └── page.tsx    # SlaSettingsPage — prazos por estado
│           ├── commission/
│           │   └── page.tsx    # CommissionSettingsPage — faixas e percentuais
│           ├── templates/
│           │   └── page.tsx    # TemplatesPage — mensagens WhatsApp
│           └── integrations/
│               └── page.tsx    # IntegrationsPage — status das integrações
│
├── components/
│   ├── layout/
│   │   ├── DashboardLayout.tsx      # Layout principal com sidebar e topbar
│   │   ├── AuthLayout.tsx           # Layout de autenticação
│   │   ├── CrmSidebar.tsx           # Sidebar do CRM com nav por role
│   │   ├── CrmTopbar.tsx            # Topbar com busca global e notificações
│   │   └── NotificationBell.tsx     # Badge de notificações in-app
│   │
│   ├── ui/                          # shadcn/ui — gerados via CLI (não editar manualmente)
│   │   ├── button.tsx
│   │   ├── card.tsx
│   │   ├── dialog.tsx
│   │   ├── drawer.tsx
│   │   ├── table.tsx
│   │   ├── skeleton.tsx
│   │   ├── badge.tsx
│   │   ├── toast.tsx
│   │   ├── tabs.tsx
│   │   ├── select.tsx
│   │   ├── input.tsx
│   │   ├── textarea.tsx
│   │   ├── popover.tsx
│   │   ├── command.tsx              # Usado no GlobalSearch (Cmd+K)
│   │   └── ...
│   │
│   ├── shared/                      # Componentes custom globais
│   │   ├── GlobalSearch.tsx         # Busca global Cmd+K (RN-197)
│   │   ├── EmptyState.tsx
│   │   ├── ConfirmDialog.tsx
│   │   ├── DataTable.tsx            # TanStack Table wrapper
│   │   ├── PageHeader.tsx           # Título + breadcrumb + ações
│   │   └── ErrorBoundary.tsx
│   │
│   ├── cases/
│   │   ├── CaseStatusBadge.tsx      # Badge visual por estado do Caso
│   │   ├── CaseStatusHistory.tsx    # Timeline de transições de estado
│   │   ├── CaseCard.tsx             # Card do Caso no kanban
│   │   ├── CaseForm.tsx             # Formulário de criação/edição
│   │   ├── CaseAssignDrawer.tsx     # Drawer de atribuição (Coordenador RS)
│   │   └── CaseTransitionButton.tsx # Botão de avanço de estado
│   │
│   ├── pipeline/
│   │   ├── KanbanBoard.tsx          # Board kanban com colunas por estado
│   │   ├── KanbanColumn.tsx         # Coluna do kanban
│   │   ├── SlaTimer.tsx             # Timer de SLA em tempo real (Supabase Realtime)
│   │   └── PipelineFilters.tsx      # Filtros do pipeline (Zustand)
│   │
│   ├── contacts/
│   │   ├── ContactCard.tsx
│   │   ├── ContactForm.tsx
│   │   └── ContactTypeBadge.tsx     # Cedente / Cessionário / Parceiro / Incorporadora
│   │
│   ├── activities/
│   │   ├── ActivityList.tsx
│   │   ├── ActivityForm.tsx         # Registro de atividade (ligação, reunião, nota)
│   │   ├── ActivityTypeBadge.tsx
│   │   └── FollowUpCard.tsx
│   │
│   ├── communication/
│   │   ├── WhatsAppChat.tsx         # Interface de chat WhatsApp
│   │   ├── MessageBubble.tsx
│   │   ├── TemplateSelector.tsx     # Seleção de templates HSM
│   │   └── WindowStatusBadge.tsx    # Status da janela 24h
│   │
│   ├── dossier/
│   │   ├── DossierProgress.tsx      # Progresso do dossiê (documentos pendentes)
│   │   ├── DocumentCard.tsx
│   │   ├── DocumentUpload.tsx       # Upload com validação de tipo e tamanho
│   │   ├── DocumentStatusBadge.tsx  # Pendente / Aprovado / Rejeitado
│   │   └── ZapsignStatusBadge.tsx   # Status do envelope de assinatura
│   │
│   ├── negotiations/
│   │   ├── ProposalCard.tsx
│   │   ├── ProposalForm.tsx
│   │   ├── ProposalStatusBadge.tsx
│   │   └── NegotiationTimeline.tsx
│   │
│   ├── commission/
│   │   ├── CommissionSummary.tsx    # Resumo de comissão estimada vs confirmada
│   │   └── CommissionBreakdown.tsx  # Detalhamento por faixa
│   │
│   └── sla/
│       ├── SlaProgressBar.tsx       # Barra de progresso % consumida
│       ├── SlaAlertBadge.tsx        # Normal / Atenção / Crítico
│       └── SlaRulesTable.tsx        # Tabela de prazos por estado
│
├── hooks/
│   ├── useAuth.ts                   # Sessão e dados do usuário logado
│   ├── useRbac.ts                   # Verificação de permissão por role
│   ├── useRealtime.ts               # Supabase Realtime subscriptions (SLA, Casos)
│   ├── useNotifications.ts          # Notificações in-app
│   └── useGlobalSearch.ts           # Hook de busca global (RN-197)
│
├── lib/
│   ├── api-client.ts                # Fetch wrapper com interceptors JWT
│   ├── supabase.ts                  # Supabase client (@supabase/ssr)
│   ├── query-client.ts              # TanStack Query client config
│   └── motion-tokens.ts             # Tokens de animação Framer Motion
│
├── stores/
│   ├── auth.store.ts                # Zustand — sessão e role do usuário
│   ├── pipeline.store.ts            # Zustand — filtros e view mode do pipeline
│   └── ui.store.ts                  # Zustand — sidebar aberta, tema, preferências
│
├── styles/
│   ├── globals.css                  # Tailwind + CSS vars (tokens Brand Guide)
│   └── crm-theme.css                # Tokens específicos do CRM
│
└── types/
    └── index.ts                     # Re-exports de @rs/types

├── next.config.ts
├── tailwind.config.ts
├── components.json                  # shadcn/ui config
├── tsconfig.json
└── package.json
```

---

## 3. App API — Módulo CRM (`apps/api/modules/crm/`)

O backend CRM é um conjunto de módulos NestJS dentro do monorepo de API compartilhado (`apps/api/`). Cada módulo segue a estrutura padrão: controller → service → repository (quando necessário) → DTOs.

```
apps/api/
├── prisma/
│   ├── schema.prisma                # Schema Prisma compartilhado
│   ├── seed.ts
│   ├── rls/
│   │   ├── crm-policies.sql         # RLS policies para tabelas do CRM
│   │   └── indexes.sql
│   └── middleware/
│       ├── soft-delete.middleware.ts
│       └── audit.middleware.ts      # Intercepta writes → audit_log
│
├── src/
│   ├── main.ts                      # Bootstrap NestJS
│   ├── app.module.ts                # Root module — importa CrmModule
│   │
│   ├── modules/
│   │   │
│   │   ├── crm/                     # ← Módulo raiz do CRM
│   │   │   ├── crm.module.ts        # Importa todos os submódulos do CRM
│   │   │   │
│   │   │   ├── auth/                # Autenticação — login, refresh, logout
│   │   │   │   ├── crm-auth.module.ts
│   │   │   │   ├── crm-auth.controller.ts
│   │   │   │   ├── crm-auth.service.ts
│   │   │   │   ├── strategies/
│   │   │   │   │   └── crm-jwt.strategy.ts    # Valida JWT + claims de papel CRM
│   │   │   │   └── dto/
│   │   │   │       ├── crm-login.dto.ts
│   │   │   │       └── crm-auth-response.dto.ts
│   │   │   │
│   │   │   ├── team/                # Usuários internos da equipe RS
│   │   │   │   ├── team.module.ts
│   │   │   │   ├── team.controller.ts
│   │   │   │   ├── team.service.ts
│   │   │   │   └── dto/
│   │   │   │       ├── create-team-member.dto.ts
│   │   │   │       ├── update-team-member.dto.ts
│   │   │   │       ├── redistribute-cases.dto.ts
│   │   │   │       └── team-member-response.dto.ts
│   │   │   │
│   │   │   ├── cases/               # Ciclo de vida do Caso
│   │   │   │   ├── cases.module.ts
│   │   │   │   ├── cases.controller.ts
│   │   │   │   ├── cases.service.ts
│   │   │   │   ├── cases.repository.ts      # Queries Prisma complexas
│   │   │   │   ├── cases-state-machine.ts   # Máquina de estados do Caso
│   │   │   │   └── dto/
│   │   │   │       ├── create-case.dto.ts
│   │   │   │       ├── update-case.dto.ts
│   │   │   │       ├── transition-case.dto.ts
│   │   │   │       ├── assign-case.dto.ts
│   │   │   │       └── case-response.dto.ts
│   │   │   │
│   │   │   ├── contacts/            # Cedentes, Cessionários, Parceiros, Incorporadoras
│   │   │   │   ├── contacts.module.ts
│   │   │   │   ├── contacts.controller.ts
│   │   │   │   ├── contacts.service.ts
│   │   │   │   └── dto/
│   │   │   │       ├── create-contact.dto.ts
│   │   │   │       ├── update-contact.dto.ts
│   │   │   │       └── contact-response.dto.ts
│   │   │   │
│   │   │   ├── activities/          # Ligações, reuniões, WhatsApp, notas, follow-ups
│   │   │   │   ├── activities.module.ts
│   │   │   │   ├── activities.controller.ts
│   │   │   │   ├── activities.service.ts
│   │   │   │   └── dto/
│   │   │   │       ├── create-activity.dto.ts
│   │   │   │       ├── update-activity.dto.ts
│   │   │   │       └── activity-response.dto.ts
│   │   │   │
│   │   │   ├── communication/       # WhatsApp Business API
│   │   │   │   ├── communication.module.ts
│   │   │   │   ├── communication.controller.ts
│   │   │   │   ├── communication.service.ts
│   │   │   │   ├── whatsapp.client.ts       # Meta Cloud API wrapper
│   │   │   │   ├── window-monitor.service.ts # Monitor de janela 24h
│   │   │   │   └── dto/
│   │   │   │       ├── send-message.dto.ts
│   │   │   │       ├── send-template.dto.ts
│   │   │   │       └── message-response.dto.ts
│   │   │   │
│   │   │   ├── dossier/             # Documentos do Dossiê
│   │   │   │   ├── dossier.module.ts
│   │   │   │   ├── dossier.controller.ts
│   │   │   │   ├── dossier.service.ts
│   │   │   │   ├── dossier-storage.service.ts # Supabase Storage wrapper
│   │   │   │   └── dto/
│   │   │   │       ├── upload-document.dto.ts
│   │   │   │       ├── approve-document.dto.ts
│   │   │   │       ├── reject-document.dto.ts
│   │   │   │       └── document-response.dto.ts
│   │   │   │
│   │   │   ├── negotiations/        # Propostas e contrapropostas
│   │   │   │   ├── negotiations.module.ts
│   │   │   │   ├── negotiations.controller.ts
│   │   │   │   ├── negotiations.service.ts
│   │   │   │   └── dto/
│   │   │   │       ├── create-proposal.dto.ts
│   │   │   │       ├── create-counterproposal.dto.ts
│   │   │   │       └── proposal-response.dto.ts
│   │   │   │
│   │   │   ├── commission/          # Comissão estimada e confirmada
│   │   │   │   ├── commission.module.ts
│   │   │   │   ├── commission.controller.ts
│   │   │   │   ├── commission.service.ts
│   │   │   │   ├── commission-calculator.ts  # Cálculo por faixas de valor
│   │   │   │   └── dto/
│   │   │   │       ├── update-commission.dto.ts
│   │   │   │       └── commission-response.dto.ts
│   │   │   │
│   │   │   ├── sla/                 # Monitor de SLA e alertas
│   │   │   │   ├── sla.module.ts
│   │   │   │   ├── sla.controller.ts
│   │   │   │   ├── sla.service.ts
│   │   │   │   ├── sla-monitor.job.ts       # Cron job — verificação a cada 5min
│   │   │   │   └── dto/
│   │   │   │       ├── update-sla-config.dto.ts
│   │   │   │       └── sla-alert-response.dto.ts
│   │   │   │
│   │   │   ├── reports/             # Relatórios e exportações
│   │   │   │   ├── reports.module.ts
│   │   │   │   ├── reports.controller.ts
│   │   │   │   ├── reports.service.ts
│   │   │   │   ├── reports.generator.ts     # Geração de PDF/CSV assíncrona
│   │   │   │   └── dto/
│   │   │   │       ├── generate-report.dto.ts
│   │   │   │       └── report-response.dto.ts
│   │   │   │
│   │   │   ├── config/              # Parâmetros configuráveis do sistema
│   │   │   │   ├── config.module.ts
│   │   │   │   ├── config.controller.ts
│   │   │   │   ├── config.service.ts
│   │   │   │   └── dto/
│   │   │   │       ├── update-config.dto.ts
│   │   │   │       └── config-response.dto.ts
│   │   │   │
│   │   │   └── webhooks/            # Webhooks de entrada (ZapSign, Escrow, Plataforma RS)
│   │   │       ├── webhooks.module.ts
│   │   │       ├── webhooks.controller.ts   # POST /crm/webhooks/*
│   │   │       ├── zapsign-webhook.handler.ts
│   │   │       ├── escrow-webhook.handler.ts
│   │   │       └── platform-webhook.handler.ts
│   │   │
│   │   └── notifications/           # Módulo compartilhado de notificações
│   │       ├── notifications.module.ts
│   │       ├── notifications.service.ts
│   │       ├── notifications.consumer.ts    # RabbitMQ consumer
│   │       └── templates/
│   │           ├── crm/
│   │           │   ├── new-case-assigned.template.ts         # NOT-CRM-01
│   │           │   ├── sla-warning.template.ts               # NOT-CRM-02
│   │           │   ├── sla-expired.template.ts               # NOT-CRM-03
│   │           │   ├── proposal-accepted.template.ts         # NOT-CRM-04
│   │           │   ├── dossier-approved.template.ts          # NOT-CRM-05
│   │           │   ├── zapsign-signed.template.ts            # NOT-CRM-06
│   │           │   ├── weekly-report.template.ts             # NOT-CRM-07
│   │           │   └── case-redistribution.template.ts       # NOT-CRM-08
│   │           └── email/
│   │               └── resend.client.ts                      # Resend API wrapper
│   │
│   ├── common/
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts            # Valida JWT
│   │   │   └── crm-rbac.guard.ts            # RBAC para roles do CRM
│   │   ├── decorators/
│   │   │   ├── crm-roles.decorator.ts       # @CrmRoles(CrmRole.ADMIN_RS)
│   │   │   └── current-user.decorator.ts    # @CurrentUser()
│   │   ├── interceptors/
│   │   │   ├── audit.interceptor.ts         # Log de auditoria imutável (RN-194)
│   │   │   └── transform.interceptor.ts     # Wrap responses em {data, meta}
│   │   ├── filters/
│   │   │   └── crm-exception.filter.ts      # Normaliza erros → CRM-XXX codes
│   │   ├── pipes/
│   │   │   └── validation.pipe.ts
│   │   └── middleware/
│   │       └── request-context.middleware.ts
│   │
│   └── prisma/
│       └── prisma.service.ts                # PrismaClient singleton
│
├── test/
│   ├── unit/
│   │   ├── crm/
│   │   │   ├── cases/
│   │   │   │   ├── cases.service.spec.ts
│   │   │   │   └── cases-state-machine.spec.ts
│   │   │   ├── sla/
│   │   │   │   └── sla.service.spec.ts
│   │   │   └── commission/
│   │   │       └── commission-calculator.spec.ts
│   ├── integration/
│   │   └── crm/
│   │       └── cases.controller.spec.ts
│   └── e2e/
│       └── crm.e2e-spec.ts
│
├── nest-cli.json
├── tsconfig.json
├── tsconfig.build.json
└── package.json
```

---

## 4. Packages Compartilhados (Relevantes para o CRM)

### `packages/types` — `@rs/types`

```
packages/types/
├── src/
│   ├── enums/
│   │   ├── crm-case-status.enum.ts       # CADASTRO | SIMULACAO | VERIFICACAO | PUBLICACAO | MATCH | NEGOCIACAO | ANUENCIA | FORMALIZACAO | CONCLUIDO | CANCELADO
│   │   ├── crm-role.enum.ts              # ADMIN_RS | COORDENADOR_RS | ANALISTA_RS | PARCEIRO_EXTERNO
│   │   ├── contact-type.enum.ts          # CEDENTE | CESSIONARIO | PARCEIRO | INCORPORADORA
│   │   ├── activity-type.enum.ts         # LIGACAO | REUNIAO | WHATSAPP | NOTA | FOLLOW_UP
│   │   ├── document-status.enum.ts       # PENDENTE | APROVADO | REJEITADO
│   │   └── sla-status.enum.ts            # NORMAL | ATENCAO | CRITICO | VENCIDO
│   ├── models/
│   │   ├── crm-case.types.ts
│   │   ├── crm-contact.types.ts
│   │   ├── crm-activity.types.ts
│   │   ├── crm-commission.types.ts
│   │   └── crm-notification.types.ts
│   └── index.ts
└── package.json
```

### `packages/shared-types` — `@rs/shared-types`

```
packages/shared-types/
├── src/
│   ├── crm/
│   │   ├── case.schema.ts             # Zod schema compartilhado (frontend + backend)
│   │   ├── contact.schema.ts
│   │   ├── activity.schema.ts
│   │   └── proposal.schema.ts
│   └── index.ts
└── package.json
```

### `packages/utils` — `@rs/utils`

```
packages/utils/
├── src/
│   ├── formatters/
│   │   ├── currency.ts                # formatBRL(value: number): string — R$ 50.000,00
│   │   ├── cpf.ts                     # formatCPF, maskCPF (exibição mascarada RN-012)
│   │   └── date.ts                    # formatBR, formatRelative, toFortaleza (UTC → America/Fortaleza)
│   ├── validators/
│   │   ├── cpf.ts
│   │   └── cnpj.ts
│   └── index.ts
└── package.json
```

---

## 5. Turborepo — Pipeline de Tasks

```json
// turbo.json (trecho relevante para CRM)
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {},
    "type-check": {}
  }
}
```

Scripts de desenvolvimento:

```bash
# Rodar apenas o CRM (frontend + api)
pnpm turbo dev --filter=web-crm --filter=api

# Build apenas o CRM
pnpm turbo build --filter=web-crm --filter=api

# Testes do módulo CRM
pnpm turbo test --filter=api -- --testPathPattern=crm
```

---

## 6. Convenções de Nomenclatura

| Contexto | Padrão | Exemplo |
|---|---|---|
| Arquivos de componente | `PascalCase.tsx` | `CaseStatusBadge.tsx` |
| Arquivos de hook | `camelCase.ts` com prefixo `use` | `usePipelineCases.ts` |
| Arquivos de service (NestJS) | `kebab-case.service.ts` | `cases.service.ts` |
| Arquivos de controller | `kebab-case.controller.ts` | `cases.controller.ts` |
| Arquivos de módulo | `kebab-case.module.ts` | `cases.module.ts` |
| Arquivos de DTO | `kebab-case.dto.ts` | `create-case.dto.ts` |
| Arquivos de store (Zustand) | `kebab-case.store.ts` | `pipeline.store.ts` |
| Pages (Next.js App Router) | `page.tsx` | `app/(dashboard)/pipeline/page.tsx` |
| Layouts (Next.js App Router) | `layout.tsx` | `app/(dashboard)/layout.tsx` |
| Pastas de feature (NestJS) | `kebab-case` | `cases/`, `sla/`, `commission/` |
| Pastas de feature (Next.js) | `kebab-case` | `pipeline/`, `contacts/` |
| Route groups (Next.js) | `(nome)` | `(auth)/`, `(dashboard)/` |
| Variáveis e funções | `camelCase` | `caseStatus`, `getUserById` |
| Classes e interfaces | `PascalCase` | `CasesService`, `ICase` |
| Constantes | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT`, `SLA_WARNING_THRESHOLD` |
| Enums TypeScript | `PascalCase` (tipo) e `UPPER_SNAKE_CASE` (valor) | `CrmRole.ADMIN_RS` |
| Error codes | `CRM-XXX` | `CRM-001`, `CRM-042` |
| Templates de notificação | `NOT-CRM-XX` | `NOT-CRM-01` |
| CSS classes (Tailwind) | `kebab-case` | `bg-primary text-foreground` |

---

## 7. Regras de Importação

```typescript
// CORRETO — 3 grupos (auto-ordenado por ESLint/Prettier)

// 1. Externos (node_modules)
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';

// 2. Pacotes internos (@rs/*)
import { CrmCaseStatus } from '@rs/types';
import { formatBRL } from '@rs/utils';

// 3. Relativos
import { CreateCaseDto } from './dto/create-case.dto';
import { CasesRepository } from './cases.repository';
```

**Regra:** Nunca importar de `../../` mais de 2 níveis acima. Se necessário, promover para `shared/` ou `packages/`.

**Barrel exports** (`index.ts`): apenas para `packages/` e `components/shared/`. Proibido em módulos NestJS para evitar dependências circulares.

---

## 8. Variáveis de Ambiente — CRM

```bash
# apps/web-crm/.env.local
NEXT_PUBLIC_API_URL=https://api.repasseseguro.com.br/api/v1/crm
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
NEXT_PUBLIC_POSTHOG_KEY=phc_...
NEXT_PUBLIC_SENTRY_DSN=https://...@sentry.io/...

# apps/api/.env (módulo CRM)
SUPABASE_SERVICE_ROLE_KEY=eyJ...
SUPABASE_JWT_SECRET=super-secret-jwt-secret
REDIS_URL=redis://...
RABBITMQ_URL=amqp://...
ZAPSIGN_API_TOKEN=...
ZAPSIGN_WEBHOOK_SECRET=...
META_WHATSAPP_TOKEN=...
META_WHATSAPP_PHONE_ID=...
META_WEBHOOK_VERIFY_TOKEN=...
CELCOIN_WEBHOOK_SECRET=...
RESEND_API_KEY=re_...
SENTRY_DSN=https://...@sentry.io/...
```

---

## 9. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — monorepo Turborepo, `apps/web-crm` Next.js 15 App Router, `apps/api/modules/crm/` NestJS modular, packages compartilhados, convenções de nomenclatura. |
