# 15 - Arquitetura de Pastas

## Módulo Cessionário · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Tech Lead, Arquiteto, DevOps |
| **Escopo** | Estrutura do monorepo, convenções de nomenclatura, padrões de módulo e mapeamento de erros/cache |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Data da versão** | 22/03/2026 00:00 (America/Fortaleza) |
| **Status** | Ativo |
| **Referências** | 01 - Regras de Negócio · 02 - Stacks · 09 - Contratos de UI por Tela · 13 - Schema Prisma · 14 - Especificações Técnicas |

---

> 📌 **TL;DR**
>
> - **Monorepo Turborepo + pnpm workspaces:** 3 apps (`web`, `api`, `mobile`) + 4 packages (`shared-types`, `design-tokens`, `eslint-config`, `tsconfig`) + `prisma/` centralizado + `docs/`.
> - **Frontend feature-first:** `apps/web/src/features/<módulo>/` com barrel exports obrigatórios. 10 features mapeadas por domínio do D09 (Contratos de UI). Imports cross-feature via barrel proibidos.
> - **Backend módulo-por-módulo:** `apps/api/src/modules/<módulo>/` com padrão rígido `Controller → Service → Repository → DTO → Entity`. 10 módulos de domínio + `common/` + `infrastructure/`.
> - **Convenções:** pastas em `kebab-case`, componentes React em `PascalCase`, hooks em `camelCase` com prefixo `use`, classes NestJS em `PascalCase`, arquivos NestJS em `kebab-case.tipo.ts`.
> - **Cache Redis:** prefixo `rs:` em todas as chaves. 8 recursos cacheados com TTL explícito.
> - **Error Prefixes:** 10 módulos mapeados (AUTH-001, USR-001, CES-001, OPR-001, PRP-001, NEG-001, FRM-001, FIN-001, NOT-001, AI-001).
> - **Zero seções pendentes** — cobertura completa de todos os apps, packages e módulos.

---

## 1. Persona

Tech Lead / Arquiteto responsável pela organização do monorepo Repasse Seguro. Este documento é a fonte única de verdade para estrutura de código, convenções de nomenclatura, padrões de módulo, cache e mapeamento de erros. Qualquer arquivo criado por devs humanos ou agentes de IA (Claude Code) deve seguir estritamente as convenções aqui definidas.

---

## 2. Visão Geral do Monorepo

```
repasse-seguro/                          # Raiz do monorepo
├── apps/
│   ├── web/                             # Frontend SPA React 19 + Vite 7
│   ├── api/                             # Backend NestJS 10 + Node 22
│   └── mobile/                          # Expo 52 + React Native 0.76
├── packages/
│   ├── shared-types/                    # Types compartilhados (TS)
│   ├── design-tokens/                   # Tokens CSS + JSON
│   ├── eslint-config/                   # Config ESLint base
│   └── tsconfig/                        # Config TypeScript base
├── prisma/                              # Schema Prisma centralizado
│   ├── schema.prisma
│   ├── migrations/
│   ├── seed/
│   └── middleware/
├── docs/                                # Documentação técnica (pipeline)
│   └── Cessionário/
├── .github/
│   └── workflows/                       # CI/CD GitHub Actions
├── turbo.json                           # Config Turborepo
├── pnpm-workspace.yaml                  # Workspaces pnpm
├── package.json                         # Root package.json
├── .env.example                         # Variáveis de ambiente (template)
└── README.md
```

⚙️ **Regra de workspaces:** cada `apps/*` e `packages/*` tem seu próprio `package.json`. Dependências compartilhadas são elevadas ao root somente se usadas em 3+ workspaces. Packages locais são referenciados via `"@repasse/shared-types": "workspace:*"`.

---

## 3. Frontend — apps/web/

### 3.1 Estrutura Completa

```
apps/web/
├── src/
│   ├── features/                        # Módulos por domínio (feature-first)
│   │   ├── auth/                        # Autenticação (T-AUTH-*)
│   │   ├── dashboard/                   # Dashboard (T-DASH-*)
│   │   ├── marketplace/                 # Marketplace (T-OPR-01 a 03)
│   │   ├── proposals/                   # Propostas (T-PRP-*)
│   │   ├── negotiations/                # Negociações (T-NEG-*)
│   │   ├── formalizations/              # Formalização (T-ASS-*)
│   │   ├── financial/                   # Financeiro (T-FIN-*)
│   │   ├── ai/                          # Analista de IA (T-IA-01)
│   │   ├── profile/                     # Perfil + KYC (T-PRF-*)
│   │   └── notifications/               # Notificações in-app
│   ├── components/                      # Componentes globais reutilizáveis
│   │   ├── ui/                          # shadcn/ui re-exports + customizações
│   │   │   ├── button.tsx
│   │   │   ├── skeleton.tsx
│   │   │   └── ...
│   │   ├── layout/
│   │   │   ├── AppLayout.tsx            # Layout logado (sidebar + header)
│   │   │   ├── AuthLayout.tsx           # Layout de autenticação
│   │   │   ├── FullPageLayout.tsx       # Layout full-page (KYC, ZapSign)
│   │   │   └── ErrorLayout.tsx          # Layout de erro
│   │   ├── domain/                      # Componentes de domínio compartilhados
│   │   │   ├── KpiCard.tsx
│   │   │   ├── SlaCountdown.tsx
│   │   │   ├── QuotaBar.tsx
│   │   │   ├── OpportunityCard.tsx
│   │   │   ├── ProposalStatusBadge.tsx
│   │   │   ├── NegotiationStatusBadge.tsx
│   │   │   ├── EscrowStatusBadge.tsx
│   │   │   ├── FormalizationStatusBadge.tsx
│   │   │   ├── CessionarioStatusBadge.tsx
│   │   │   └── AiRiskScoreBadge.tsx
│   │   └── shared/                      # Componentes utilitários
│   │       ├── EmptyState.tsx
│   │       ├── DataTable.tsx
│   │       ├── TableSkeleton.tsx
│   │       └── OfflineBanner.tsx
│   ├── hooks/                           # Hooks globais
│   │   ├── useRbac.ts                   # RBAC + KYC guard
│   │   ├── useRealtimeSubscription.ts   # Supabase Realtime
│   │   ├── useSession.ts                # Sessão e auto-refresh
│   │   └── useToast.ts                  # Toast notifications
│   ├── stores/                          # Stores Zustand globais
│   │   ├── auth.store.ts                # Sessão, perfil, KYC status
│   │   ├── notifications.store.ts       # Notificações in-app
│   │   └── ui.store.ts                  # Preferências de UI (tema, sidebar)
│   ├── services/                        # Camada de API global
│   │   ├── api.ts                       # fetch wrapper + JWT interceptor
│   │   ├── supabase.ts                  # Supabase JS client
│   │   └── analytics.ts                 # PostHog wrapper centralizado
│   ├── lib/                             # Utilitários puros
│   │   ├── currency.ts                  # Formatação BRL
│   │   ├── dates.ts                     # date-fns + timezone
│   │   ├── commission.ts                # Cálculo Comissão Comprador
│   │   └── validators.ts                # Validadores comuns
│   ├── types/                           # Types globais (re-exporta de packages)
│   │   └── index.ts
│   ├── router/                          # TanStack Router
│   │   ├── index.tsx                    # Router definition + RootRoute
│   │   ├── guards/
│   │   │   ├── AuthGuard.tsx            # Redireciona para /login se não autenticado
│   │   │   ├── RbacGuard.tsx            # Verifica role + KYC
│   │   │   └── KycGuard.tsx             # Bloqueia se KYC não aprovado
│   │   └── routes.ts                    # Constantes de rotas tipadas
│   ├── styles/
│   │   └── globals.css                  # Tailwind + shadcn tokens CSS
│   └── main.tsx                         # Entry point
├── public/
│   └── assets/
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

### 3.2 Estrutura Interna de Feature

Cada feature segue este padrão obrigatório:

```
features/marketplace/
├── index.ts                             # ← Barrel export (API pública)
├── pages/
│   ├── MarketplacePage.tsx              # T-OPR-01
│   ├── OpportunityDetailPage.tsx        # T-OPR-02
│   └── ProposalFormPage.tsx             # T-OPR-03
├── components/
│   ├── OpportunityFilters.tsx
│   ├── OpportunityGrid.tsx
│   └── FinancialSummaryCard.tsx
├── hooks/
│   ├── useOpportunities.ts              # TanStack Query — listagem
│   ├── useOpportunityDetail.ts          # TanStack Query — detalhe
│   └── useCreateProposal.ts             # TanStack Query mutation
├── services/
│   └── marketplace.service.ts           # fetch calls para /api/v1/opportunities
├── stores/
│   └── marketplace.store.ts             # Estado de filtros, paginação
├── types/
│   └── marketplace.types.ts             # Interfaces locais da feature
└── __tests__/
    ├── OpportunityFilters.test.tsx
    └── useCreateProposal.test.ts
```

💡 **Barrel export obrigatório:** `features/marketplace/index.ts` exporta somente o que outras features/camadas precisam consumir:

```typescript
// features/marketplace/index.ts
export { MarketplacePage } from './pages/MarketplacePage';
export { OpportunityDetailPage } from './pages/OpportunityDetailPage';
export { ProposalFormPage } from './pages/ProposalFormPage';
// Hooks e types que podem ser reutilizados:
export type { Opportunity, OpportunityFilters } from './types/marketplace.types';
```

### 3.3 Features Mapeadas por Domínio

| Feature | Diretório | Telas cobertas | RFs principais |
|---|---|---|---|
| `auth` | `features/auth/` | T-AUTH-01 a T-AUTH-04 | RF-001 a RF-005 |
| `dashboard` | `features/dashboard/` | T-DASH-01 + overlay re-auth | RF-041 a RF-045 |
| `marketplace` | `features/marketplace/` | T-OPR-01 a T-OPR-03 | RF-017 a RF-023 |
| `proposals` | `features/proposals/` | T-PRP-01, T-PRP-02 | RF-020 a RF-023 |
| `negotiations` | `features/negotiations/` | T-NEG-01 a T-NEG-04 + overlay | RF-024 a RF-030 |
| `formalizations` | `features/formalizations/` | T-ASS-01 a T-ASS-03 | RF-031 a RF-033 |
| `financial` | `features/financial/` | T-FIN-01 a T-FIN-03 | RF-036 a RF-040 |
| `ai` | `features/ai/` | T-IA-01 | RF-046 a RF-050 |
| `profile` | `features/profile/` | T-PRF-01 a T-PRF-05 | RF-006 a RF-016 |
| `notifications` | `features/notifications/` | (widget Dashboard + centro) | RF-064, RF-065 |

### 3.4 Regras de Import

✅ **Correto — via barrel export:**
```typescript
// Em negotiations/pages/NegotiationPage.tsx
import { Opportunity } from '@/features/marketplace'; // barrel
import { ProposalStatusBadge } from '@/components/domain';
import { formatCurrency } from '@/lib/currency';
```

🔴 **Incorreto — import direto cross-feature:**
```typescript
// PROIBIDO — acesso direto a internos de outra feature
import { OpportunityCard } from '@/features/marketplace/components/OpportunityCard';
import { useOpportunities } from '@/features/marketplace/hooks/useOpportunities';
```

🔴 **Incorreto — import de serviço global dentro de feature sem passar por hook:**
```typescript
// PROIBIDO — fetch direto sem TanStack Query
import { api } from '@/services/api';
useEffect(() => { api.get('/opportunities') }, []);
```

⚙️ **Alias `@/`** configurado em `vite.config.ts` apontando para `src/`. Nunca usar `../../../`.

---

## 4. Backend — apps/api/

### 4.1 Estrutura Completa

```
apps/api/
├── src/
│   ├── modules/                         # Módulos de domínio
│   │   ├── auth/
│   │   ├── users/
│   │   ├── cessionarios/
│   │   ├── opportunities/
│   │   ├── proposals/
│   │   ├── negotiations/
│   │   ├── formalizations/
│   │   ├── financial/
│   │   ├── notifications/
│   │   └── ai/
│   ├── common/                          # Transversais do framework
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   ├── rbac.guard.ts
│   │   │   └── kyc.guard.ts
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts
│   │   │   ├── sentry.interceptor.ts
│   │   │   └── transform-response.interceptor.ts
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts
│   │   │   ├── roles.decorator.ts
│   │   │   └── kyc-required.decorator.ts
│   │   ├── pipes/
│   │   │   ├── validation.pipe.ts
│   │   │   └── parse-uuid.pipe.ts
│   │   ├── filters/
│   │   │   └── global-exception.filter.ts
│   │   └── dto/
│   │       ├── pagination.dto.ts
│   │       └── api-response.dto.ts
│   ├── infrastructure/                  # Integrações de infraestrutura
│   │   ├── database/
│   │   │   ├── prisma.service.ts
│   │   │   └── prisma.module.ts
│   │   ├── redis/
│   │   │   ├── redis.service.ts
│   │   │   └── redis.module.ts
│   │   ├── rabbitmq/
│   │   │   ├── rabbitmq.service.ts
│   │   │   ├── rabbitmq.module.ts
│   │   │   └── consumers/
│   │   │       ├── kyc.consumer.ts
│   │   │       ├── notifications.consumer.ts
│   │   │       ├── rag-embed.consumer.ts
│   │   │       └── zapsign.consumer.ts
│   │   ├── storage/
│   │   │   ├── storage.service.ts       # Supabase Storage
│   │   │   └── storage.module.ts
│   │   └── realtime/
│   │       ├── realtime.service.ts      # Supabase Realtime broadcast
│   │       └── realtime.module.ts
│   ├── config/
│   │   ├── app.config.ts                # Configurações gerais (porta, CORS)
│   │   ├── database.config.ts
│   │   ├── redis.config.ts
│   │   ├── rabbitmq.config.ts
│   │   └── supabase.config.ts
│   └── main.ts                          # Bootstrap NestJS
├── test/
│   └── e2e/                             # Testes E2E (*.e2e-spec.ts)
├── tsconfig.json
├── tsconfig.build.json
└── package.json
```

### 4.2 Estrutura Interna de Módulo

Cada módulo de domínio segue este padrão obrigatório:

```
modules/proposals/
├── proposals.module.ts                  # NestJS Module
├── proposals.controller.ts              # HTTP handlers
├── services/
│   └── proposals.service.ts             # Lógica de negócio
├── repositories/
│   └── proposals.repository.ts          # Acesso Prisma
├── dto/
│   ├── create-proposal.dto.ts
│   ├── update-proposal.dto.ts
│   ├── filter-proposals.dto.ts
│   └── proposal-response.dto.ts
├── entities/
│   └── proposal.entity.ts               # Tipo de retorno da entidade
├── types/
│   └── proposals.types.ts               # Interfaces internas
└── __tests__/
    ├── proposals.service.spec.ts
    └── proposals.controller.spec.ts
```

### 4.3 Módulos Backend Mapeados

| Módulo | Diretório | Prefix da rota | RNs principais |
|---|---|---|---|
| `AuthModule` | `modules/auth/` | `/api/v1/auth` | RN-001 a RN-005 |
| `UsersModule` | `modules/users/` | `/api/v1/users` | RN-006 a RN-016 |
| `CessionariosModule` | `modules/cessionarios/` | `/api/v1/cessionarios` | RN-006 a RN-016 |
| `OpportunitiesModule` | `modules/opportunities/` | `/api/v1/opportunities` | RN-017 a RN-019 |
| `ProposalsModule` | `modules/proposals/` | `/api/v1/proposals` | RN-020 a RN-023 |
| `NegotiationsModule` | `modules/negotiations/` | `/api/v1/negotiations` | RN-024 a RN-031 |
| `FormalizationsModule` | `modules/formalizations/` | `/api/v1/formalizations` | RN-032 a RN-034 |
| `FinancialModule` | `modules/financial/` | `/api/v1/financial` | RN-037 a RN-041 |
| `NotificationsModule` | `modules/notifications/` | `/api/v1/notifications` | RN-061 a RN-066 |
| `AiModule` | `modules/ai/` | `/api/v1/ai` | RN-047 a RN-051 |

### 4.4 Mobile — apps/mobile/

```
apps/mobile/
├── app/                                 # expo-router file-based routing
│   ├── _layout.tsx                      # RootLayout
│   ├── (auth)/
│   │   ├── _layout.tsx
│   │   ├── login.tsx
│   │   ├── cadastro.tsx
│   │   ├── recuperar-senha.tsx
│   │   └── nova-senha.tsx
│   ├── (tabs)/
│   │   ├── _layout.tsx
│   │   ├── dashboard/
│   │   ├── marketplace/
│   │   ├── operacoes/
│   │   ├── financeiro/
│   │   └── perfil/
│   └── analista.tsx
├── src/
│   ├── components/                      # Componentes compartilhados mobile
│   │   ├── ui/                          # Base components (Button, Input, etc.)
│   │   ├── domain/                      # Componentes de domínio
│   │   └── layout/                      # Layouts mobile
│   ├── hooks/                           # Hooks mobile
│   │   ├── useSession.ts
│   │   ├── useOfflineSync.ts
│   │   └── useNetworkStatus.ts
│   ├── stores/                          # Zustand stores (compartilhados via packages)
│   ├── services/
│   │   ├── api.ts                       # fetch wrapper mobile
│   │   └── analytics.ts
│   ├── lib/                             # Utilitários
│   └── types/
├── assets/
├── app.json                             # Expo config
├── babel.config.js
├── tsconfig.json
└── package.json
```

---

## 5. Convenções de Nomenclatura

| Contexto | Convenção | Exemplo |
|---|---|---|
| **Pastas** (todos os apps) | `kebab-case` | `features/marketplace/`, `shared-types/` |
| **Arquivos React** (componentes) | `PascalCase.tsx` | `OpportunityCard.tsx`, `MarketplacePage.tsx` |
| **Arquivos React** (hooks) | `camelCase.ts` | `useOpportunities.ts`, `useCreateProposal.ts` |
| **Arquivos React** (stores, services, lib) | `kebab-case.tipo.ts` | `marketplace.store.ts`, `api.service.ts` |
| **Arquivos NestJS** (todos) | `kebab-case.tipo.ts` | `proposals.controller.ts`, `proposals.service.ts` |
| **Componentes React** | `PascalCase` (função) | `export function OpportunityCard() {}` |
| **Hooks React** | `camelCase` com prefixo `use` | `export function useOpportunities() {}` |
| **Stores Zustand** | `camelCase` com sufixo `Store` | `useMarketplaceStore`, `useAuthStore` |
| **Classes NestJS** | `PascalCase` com sufixo do tipo | `ProposalsController`, `ProposalsService`, `ProposalsRepository` |
| **DTOs** | `PascalCase` — `Create/Update/Filter/Response` + DTO | `CreateProposalDto`, `ProposalResponseDto` |
| **Interfaces TypeScript** | `PascalCase` sem prefixo `I` | `Proposal`, `OpportunityFilters`, `JwtPayload` |
| **Types utilitários** | `PascalCase` com sufixo descritivo | `ProposalStatus`, `EscrowStatusMap` |
| **Enums** | `PascalCase` (nome), `UPPER_SNAKE_CASE` (valores) | `enum ProposalStatus { ENVIADA = 'ENVIADA' }` |
| **Constantes** | `UPPER_SNAKE_CASE` | `MAX_SIMULTANEOUS_PROPOSALS = 3` |
| **Testes unitários** | `kebab-case.spec.ts` | `proposals.service.spec.ts` |
| **Testes de componente** | `PascalCase.test.tsx` | `OpportunityCard.test.tsx` |
| **Testes E2E** | `kebab-case.e2e-spec.ts` | `proposals.e2e-spec.ts` |
| **Variáveis de ambiente** | `UPPER_SNAKE_CASE` | `DATABASE_URL`, `OPENAI_API_KEY` |
| **Rotas API** | `kebab-case` plural | `/api/v1/opportunities`, `/api/v1/proposals` |
| **Migrations Prisma** | `timestamp_descricao_snake_case` | `20260322000000_create_proposals_table` |

⚙️ **Regra de idioma:** todos os nomes de arquivos, pastas, variáveis e funções em **inglês**. Comentários e docstrings podem ser em português. Mensagens de erro para usuário final sempre em português.

---

## 6. Cache Redis — Convenção de Chaves

Prefixo global: `rs:` (Repasse Seguro). Padrão: `rs:{domínio}:{tipo}:{id}`.

| Chave (padrão) | TTL | Módulo | Contexto |
|---|---|---|---|
| `rs:session:{user_id}` | 86400s (24h) | Auth | Payload da sessão JWT validada |
| `rs:quota:sim:{cessionario_id}` | 86400s (24h) | Proposals | Contador de propostas simultâneas (máx 3) |
| `rs:quota:daily:{cessionario_id}` | Até midnight UTC | Proposals | Contador diário de propostas (máx 10) |
| `rs:ratelimit:auth:{ip}` | 900s (15min) | Auth | Falhas de autenticação por IP |
| `rs:ratelimit:llm:{cessionario_id}` | 60s | AI | Chamadas LLM por usuário (máx 20/min) |
| `rs:ai:cache:{opp_id}:{msg_hash}` | 300s (5min) | AI | Resultado de análise IA por mensagem |
| `rs:ai:score:{opportunity_id}` | 3600s (1h) | AI | Score de risco IA da oportunidade |
| `rs:opp:list:{filters_hash}:{page}` | 60s | Opportunities | Cache de listagem com filtros |

⚙️ **Regras obrigatórias:**
- Todo `SET` deve incluir `EX {ttl}` — nunca sem TTL.
- Chaves de quota: usar `INCR` + `EXPIRE` em operações atômicas via `MULTI/EXEC`.
- Chaves de ratelimit: usar `INCR` + `EXPIRE`; o primeiro `INCR` define o TTL.
- Em invalidação proativa: `DEL rs:opp:list:*` via `SCAN` + `DEL` (nunca `FLUSHDB`).
- Prefixo `rs:` é inegociável — facilita `SCAN rs:*` para debug e limpeza.

---

## 7. Mapeamento Módulo → Error Prefix

| Módulo NestJS | Error Prefix | Queue(s) RabbitMQ |
|---|---|---|
| `AuthModule` | `AUTH` | — |
| `UsersModule` | `USR` | — |
| `CessionariosModule` | `CES` | `kyc.process` (publish) |
| `OpportunitiesModule` | `OPR` | `ai.embed` (publish) |
| `ProposalsModule` | `PRP` | `notifications.send` (publish) |
| `NegotiationsModule` | `NEG` | `notifications.send` (publish) |
| `FormalizationsModule` | `FRM` | `zapsign.webhook` (consume), `notifications.send` (publish) |
| `FinancialModule` | `FIN` | — |
| `NotificationsModule` | `NOT` | `notifications.email` (consume), `notifications.push` (consume) |
| `AiModule` | `AI` | `ai.embed` (consume) |
| `PrismaService` | `DB` | — |
| `RedisService` | `CACHE` | — |
| `StorageService` | `STG` | — |

**Padrão de código de erro:**
```
{PREFIX}-{3 dígitos}
Exemplos:
  AUTH-001: Credenciais inválidas
  PRP-001: Quota simultânea atingida
  PRP-002: Quota diária atingida
  NEG-001: Máximo de rodadas de contraproposta atingido
  AI-001: Rate limit LLM atingido
  CES-001: KYC não aprovado
```

Todos os erros são lançados via `GlobalExceptionFilter` com estrutura padronizada:
```typescript
{
  statusCode: number,
  errorCode: string,   // ex: "PRP-001"
  message: string,     // mensagem human-readable
  timestamp: string,   // ISO 8601
  path: string         // URL da requisição
}
```

---

## 8. Packages

### 8.1 packages/shared-types/

```
packages/shared-types/
├── src/
│   ├── entities/                        # Interfaces espelho das entidades Prisma
│   │   ├── cessionario.types.ts
│   │   ├── opportunity.types.ts
│   │   ├── proposal.types.ts
│   │   ├── negotiation.types.ts
│   │   ├── escrow.types.ts
│   │   ├── formalization.types.ts
│   │   └── financial.types.ts
│   ├── enums/                           # Enums compartilhados (espelho dos Prisma enums)
│   │   └── index.ts
│   ├── api/                             # Request/Response types de API
│   │   └── index.ts
│   └── index.ts                         # Barrel export
├── tsconfig.json
└── package.json                         # name: "@repasse/shared-types"
```

### 8.2 packages/design-tokens/

```
packages/design-tokens/
├── src/
│   ├── tokens.css                       # CSS custom properties (shadcn theme)
│   ├── tokens.ts                        # TypeScript constants
│   └── index.ts
├── tsconfig.json
└── package.json                         # name: "@repasse/design-tokens"
```

### 8.3 packages/eslint-config/

```
packages/eslint-config/
├── index.js                             # Base config
├── react.js                             # React + TSX rules
├── node.js                              # Node.js rules
└── package.json                         # name: "@repasse/eslint-config"
```

### 8.4 packages/tsconfig/

```
packages/tsconfig/
├── base.json                            # TypeScript strict base
├── react.json                           # React + DOM
├── node.json                            # Node.js
└── package.json                         # name: "@repasse/tsconfig"
```

---

## 9. Prisma (centralizado)

```
prisma/
├── schema.prisma                        # Schema principal (gerado em D13)
├── migrations/
│   └── YYYYMMDDHHMMSS_descricao/
│       └── migration.sql
├── seed/
│   ├── index.ts                         # Entry point do seed
│   ├── seeds/
│   │   ├── opportunities.seed.ts        # Oportunidades de teste
│   │   └── cessionarios.seed.ts         # Usuários de teste
│   └── utils/
│       └── faker.ts                     # Faker.js helpers
└── middleware/
    └── soft-delete.middleware.ts        # Middleware Prisma para soft delete
```

⚙️ **Regra de migrations:** migrations geradas apenas via `prisma migrate dev`. Nunca editar arquivos em `migrations/` manualmente. Colunas especiais (generated column `delta_value`, HNSW index pgvector) criadas via `migration.sql` customizado.

---

## 10. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 22/03/2026 | v1.0 | Criação inicial — Pipeline ShiftLabs v9.5. Monorepo Turborepo completo: 3 apps, 4 packages, prisma centralizado. 10 features web, 10 módulos backend. Convenções de nomenclatura completas. 8 chaves Redis com TTL. 13 error prefixes mapeados. |

---

## 11. Backlog de Pendências

| Item | Tipo | Seção | Impacto | Justificativa / Decisão | Dono | Status |
|---|---|---|---|---|---|---|
| apps/mobile usa app/ (expo-router) + src/ (componentes) — estrutura dual | Decisão Autônoma | §4.4 Mobile | P1 | expo-router exige `app/` para file-based routing; lógica de componentes em `src/` garante separação de responsabilidades; alternativa descartada: tudo em `app/` (mistura rota e componente) | Tech Lead | Decidido |
| Prefixo global Redis `rs:` (não `repasse:`) | Decisão Autônoma | §6 Cache | P2 | Prefixo curto reduz tamanho das chaves e overhead em operações de SCAN; `rs:` é inequívoco no contexto do produto; alternativa descartada: `repasse-seguro:` (muito verbose) | Tech Lead | Decidido |
| Error codes no formato `PREFIX-3dígitos` | Decisão Autônoma | §7 Error Prefix | P1 | 3 dígitos permite 999 erros por módulo — mais que suficiente; padrão adotado em APIs REST maduras (Stripe, Twilio); alternativa descartada: UUID (ilegível em logs) | Tech Lead | Decidido |
| packages/shared-types com espelhos manuais dos tipos Prisma | Decisão Autônoma | §8.1 | P2 | Prisma gera tipos apenas para o banco; shared-types expõe interfaces para frontend e mobile sem acoplamento direto ao Prisma client; alternativa descartada: importar `@prisma/client` no frontend (vazamento de schema) | Tech Lead | Decidido |
