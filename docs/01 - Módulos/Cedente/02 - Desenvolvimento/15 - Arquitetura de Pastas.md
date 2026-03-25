# 15 - Arquitetura de Pastas

## Módulo Cedente · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Tech Lead, Arquiteto, DevOps |
| **Escopo** | Estrutura do monorepo, convenções de nomenclatura, padrões de módulo e mapeamento de erros/cache |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Data da versão** | 2026-03-23 (America/Fortaleza) |
| **Status** | Ativo |
| **Referências** | 01.1 a 01.5 - Regras de Negócio · 02 - Stacks · 13 - Schema Prisma · 14 - Especificações Técnicas |

---

> **TL;DR**
>
> - **Monorepo Turborepo + pnpm workspaces:** 3 apps (`web-cedente`, `api`, `mobile-cedente`) + 4 packages (`shared-types`, `design-tokens`, `eslint-config`, `tsconfig`) + `prisma/` centralizado + `docs/`.
> - **Frontend Web (Next.js 15 App Router) feature-first:** `apps/web-cedente/src/app/(public)/` para rotas públicas e `apps/web-cedente/src/app/(authenticated)/` para área logada. Features em `src/features/<módulo>/` com barrel exports obrigatórios.
> - **Mobile (Expo SDK 52) feature-first:** `apps/mobile-cedente/src/screens/<módulo>/` com navegação stack + tabs via expo-router 4.
> - **Backend NestJS módulo-por-módulo:** `apps/api/src/modules/<módulo>/` com padrão rígido `Controller → Service → Repository → DTO → Entity`. 11 módulos de domínio + `common/` + `infrastructure/`.
> - **Convenções:** pastas em `kebab-case`, componentes React em `PascalCase`, hooks em `camelCase` com prefixo `use`, classes NestJS em `PascalCase`, arquivos NestJS em `kebab-case.tipo.ts`.
> - **Cache Redis:** prefixo `rs:cedente:` em todas as chaves. 8 recursos cacheados com TTL explícito.
> - **Error Prefixes:** 11 módulos mapeados (AUTH, CED, CAS, DOC, PRP, ASS, ESC, NOT, AI, ANU, COM).

---

## 1. Visão Geral do Monorepo

```
repasse-seguro/                              # Raiz do monorepo Turborepo + pnpm
├── apps/
│   ├── web-cedente/                         # Frontend Next.js 15 App Router — Módulo Cedente
│   ├── web-cessionario/                     # Frontend React 19 + Vite 7 — Módulo Cessionário
│   ├── api/                                 # Backend NestJS 10 — todos os módulos (API unificada)
│   └── mobile-cedente/                      # Mobile Expo SDK 52 + React Native 0.76 — Cedente
├── packages/
│   ├── shared-types/                        # Types TypeScript compartilhados entre apps
│   ├── design-tokens/                       # Tokens CSS + JSON (shadcn/ui + Tailwind 4)
│   ├── eslint-config/                       # Config ESLint base (ShiftLabs v7.0)
│   └── tsconfig/                            # Config TypeScript base (strict: true)
├── prisma/                                  # Schema Prisma centralizado
│   ├── schema.prisma                        # Schema unificado de todos os módulos
│   ├── migrations/                          # Migrations geradas pelo Prisma
│   ├── seed/
│   │   ├── cedente.seed.ts                  # Seed do módulo Cedente
│   │   └── index.ts                         # Seed entry point
│   └── middleware/
│       └── soft-delete.middleware.ts        # Middleware de soft delete
├── docs/                                    # Documentação técnica do pipeline
│   ├── Cedente/
│   └── Cessionário/
├── .github/
│   └── workflows/                           # CI/CD GitHub Actions
│       ├── ci.yml                           # Lint + type-check + tests
│       ├── deploy-api.yml                   # Deploy Railway (NestJS)
│       ├── deploy-web-cedente.yml           # Deploy Vercel (Next.js)
│       └── deploy-mobile.yml               # Build EAS (Expo)
├── turbo.json                               # Config Turborepo (pipeline tasks)
├── pnpm-workspace.yaml                      # Workspaces pnpm
├── package.json                             # Root package.json (devDeps compartilhadas)
├── .env.example                             # Template de variáveis de ambiente
└── README.md
```

**Regra de workspaces:** cada `apps/*` e `packages/*` tem seu próprio `package.json`. Dependências compartilhadas são elevadas ao root somente se usadas em 3+ workspaces. Packages locais são referenciados via `"@repasse/shared-types": "workspace:*"`.

---

## 2. Frontend Web — `apps/web-cedente/`

### 2.1 Estrutura Completa

```
apps/web-cedente/
├── src/
│   ├── app/                                 # App Router — file-based routing
│   │   ├── (public)/                        # Route group: rotas públicas (sem auth)
│   │   │   ├── layout.tsx                   # Layout público (sem sidebar)
│   │   │   ├── page.tsx                     # Landing page (SSR — SEO)
│   │   │   ├── cadastro/
│   │   │   │   └── page.tsx                 # Página de cadastro
│   │   │   ├── login/
│   │   │   │   └── page.tsx                 # Página de login
│   │   │   ├── ativar-conta/
│   │   │   │   └── page.tsx                 # Ativação de conta (token por query param)
│   │   │   └── recuperar-senha/
│   │   │       ├── page.tsx                 # Solicitar recuperação
│   │   │       └── [token]/page.tsx         # Redefinir senha
│   │   ├── (authenticated)/                 # Route group: área logada (requer auth)
│   │   │   ├── layout.tsx                   # Layout logado (AppLayout com sidebar)
│   │   │   ├── dashboard/
│   │   │   │   └── page.tsx                 # Dashboard com resumo de casos
│   │   │   ├── casos/
│   │   │   │   ├── page.tsx                 # Lista de casos (Meus Casos)
│   │   │   │   ├── novo/
│   │   │   │   │   └── page.tsx             # Wizard de cadastro (5 etapas)
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx             # Detalhe do caso
│   │   │   │       ├── documentos/
│   │   │   │       │   └── page.tsx         # Checklist do dossiê
│   │   │   │       ├── propostas/
│   │   │   │       │   ├── page.tsx         # Lista de propostas
│   │   │   │       │   └── [proposta_id]/page.tsx
│   │   │   │       ├── assinaturas/
│   │   │   │       │   └── page.tsx         # Envelopes de assinatura
│   │   │   │       └── financeiro/
│   │   │   │           └── page.tsx         # Painel financeiro (Conta Escrow)
│   │   │   ├── guardiao/
│   │   │   │   └── page.tsx                 # Chat com o Guardião do Retorno (SSE)
│   │   │   └── perfil/
│   │   │       └── page.tsx                 # Dados pessoais + segurança + notificações
│   │   ├── api/
│   │   │   └── guardiao/
│   │   │       └── stream/route.ts          # API Route para SSE do Guardião
│   │   ├── layout.tsx                       # Root layout (providers, fonts, meta)
│   │   ├── error.tsx                        # Error boundary global
│   │   ├── not-found.tsx                    # Página 404
│   │   └── globals.css                      # Design tokens + Tailwind 4 base
│   ├── features/                            # Módulos por domínio (feature-first)
│   │   ├── auth/                            # Autenticação e conta
│   │   │   ├── components/
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   ├── RegisterForm.tsx
│   │   │   │   ├── ForgotPasswordForm.tsx
│   │   │   │   └── ActivateAccountBanner.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useLogin.ts
│   │   │   │   ├── useRegister.ts
│   │   │   │   └── useSession.ts
│   │   │   ├── services/
│   │   │   │   └── auth.service.ts          # Chamadas à API de auth
│   │   │   ├── store/
│   │   │   │   └── auth.store.ts            # Zustand — estado de sessão
│   │   │   └── index.ts                     # Barrel export obrigatório
│   │   ├── dashboard/                       # Dashboard (resumo de casos)
│   │   │   ├── components/
│   │   │   │   ├── DashboardSummary.tsx
│   │   │   │   ├── CasoStatusCard.tsx
│   │   │   │   ├── ProximosPassosBanner.tsx
│   │   │   │   └── AlertasPendentes.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useDashboard.ts
│   │   │   ├── services/
│   │   │   │   └── dashboard.service.ts
│   │   │   └── index.ts
│   │   ├── casos/                           # Gestão de casos e wizard de cadastro
│   │   │   ├── components/
│   │   │   │   ├── CasoCard.tsx
│   │   │   │   ├── CasoStatusBadge.tsx
│   │   │   │   ├── CasoTimeline.tsx
│   │   │   │   ├── wizard/
│   │   │   │   │   ├── WizardContainer.tsx
│   │   │   │   │   ├── WizardProgress.tsx
│   │   │   │   │   ├── Etapa1DadosImovel.tsx
│   │   │   │   │   ├── Etapa2DadosFinanceiros.tsx
│   │   │   │   │   ├── Etapa3Simulador.tsx    # Simulador com timer 10s (RN-021)
│   │   │   │   │   ├── Etapa4EscolhaCenario.tsx # Escolha ativa (RN-022)
│   │   │   │   │   └── Etapa5Confirmacao.tsx
│   │   │   │   └── escalonamento/
│   │   │   │       ├── EscalonamentoModal.tsx
│   │   │   │       └── SimulacaoComparativa.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useCasos.ts
│   │   │   │   ├── useCaso.ts
│   │   │   │   ├── useWizard.ts             # Estado multi-step do wizard (Zustand)
│   │   │   │   └── useEscalonamento.ts
│   │   │   ├── services/
│   │   │   │   └── casos.service.ts
│   │   │   ├── store/
│   │   │   │   └── wizard.store.ts          # Rascunho do wizard (RN-023)
│   │   │   └── index.ts
│   │   ├── documentos/                      # Dossiê e upload de documentos
│   │   │   ├── components/
│   │   │   │   ├── DossieChecklist.tsx
│   │   │   │   ├── DocumentoItem.tsx
│   │   │   │   ├── DocumentoStatusIcon.tsx
│   │   │   │   ├── UploadZone.tsx           # Drag-and-drop + câmera mobile
│   │   │   │   └── UploadProgress.tsx       # Barra de progresso (RN-042)
│   │   │   ├── hooks/
│   │   │   │   ├── useDossie.ts
│   │   │   │   └── useUploadDocumento.ts
│   │   │   ├── services/
│   │   │   │   └── documentos.service.ts
│   │   │   └── index.ts
│   │   ├── propostas/                       # Propostas recebidas
│   │   │   ├── components/
│   │   │   │   ├── PropostaCard.tsx
│   │   │   │   ├── PropostaSimulacao.tsx    # Simulação de valores líquidos
│   │   │   │   ├── PropostaTimerBadge.tsx   # Timer regressivo 5 d.u. (RN-031)
│   │   │   │   ├── AceitarPropostaModal.tsx # Dupla confirmação (RN-032)
│   │   │   │   └── ContrapropostaForm.tsx   # Contraproposta (RN-035)
│   │   │   ├── hooks/
│   │   │   │   ├── usePropostas.ts
│   │   │   │   └── useProposta.ts
│   │   │   ├── services/
│   │   │   │   └── propostas.service.ts
│   │   │   └── index.ts
│   │   ├── assinaturas/                     # Envelopes de assinatura ZapSign
│   │   │   ├── components/
│   │   │   │   ├── EnvelopeList.tsx
│   │   │   │   ├── EnvelopeItem.tsx
│   │   │   │   ├── ZapSignViewer.tsx        # iframe inline (RN-047, RN-080)
│   │   │   │   └── DocumentoAssinadoDownload.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useAssinaturas.ts
│   │   │   │   └── useZapSign.ts
│   │   │   ├── services/
│   │   │   │   └── assinaturas.service.ts
│   │   │   └── index.ts
│   │   ├── financeiro/                      # Painel financeiro (somente leitura)
│   │   │   ├── components/
│   │   │   │   ├── FinanceiroPanel.tsx
│   │   │   │   ├── ValorLiquidoDestaque.tsx  # Maior destaque visual (RN-052)
│   │   │   │   ├── EscrowStatusCard.tsx
│   │   │   │   ├── PeriodoReversaoCountdown.tsx # Countdown 15 dias (RN-053)
│   │   │   │   └── ComissaoBreakdown.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useFinanceiro.ts
│   │   │   ├── services/
│   │   │   │   └── financeiro.service.ts
│   │   │   └── index.ts
│   │   ├── guardiao/                        # Chat IA — Guardião do Retorno
│   │   │   ├── components/
│   │   │   │   ├── GuardiaoChat.tsx         # Container do chat
│   │   │   │   ├── MessageBubble.tsx
│   │   │   │   ├── TypingIndicator.tsx      # Streaming SSE visual
│   │   │   │   ├── EscalacaoButton.tsx      # Escalar para humano (RN-061)
│   │   │   │   └── GuardiaoChatInput.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useGuardiao.ts           # Vercel AI SDK useChat
│   │   │   ├── services/
│   │   │   │   └── guardiao.service.ts
│   │   │   └── index.ts
│   │   ├── notificacoes/                    # Notificações in-app
│   │   │   ├── components/
│   │   │   │   ├── NotificacaoList.tsx
│   │   │   │   ├── NotificacaoItem.tsx
│   │   │   │   └── NotificacaoBadge.tsx     # Badge de não lidas (RN-057)
│   │   │   ├── hooks/
│   │   │   │   └── useNotificacoes.ts       # Supabase Realtime subscription
│   │   │   ├── services/
│   │   │   │   └── notificacoes.service.ts
│   │   │   └── index.ts
│   │   └── perfil/                          # Dados pessoais e configurações
│   │       ├── components/
│   │       │   ├── DadosPessoaisForm.tsx
│   │       │   ├── AlterarSenhaForm.tsx
│   │       │   ├── PreferenciasNotificacao.tsx
│   │       │   ├── ConsentimentosLgpd.tsx   # Gerenciar consentimentos (RN-010)
│   │       │   └── SolicitarExclusaoModal.tsx
│   │       ├── hooks/
│   │       │   └── usePerfil.ts
│   │       ├── services/
│   │       │   └── perfil.service.ts
│   │       └── index.ts
│   ├── components/                          # Componentes globais reutilizáveis
│   │   ├── ui/                              # shadcn/ui re-exports + customizações
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── toast.tsx
│   │   │   └── ...                          # demais componentes shadcn/ui
│   │   ├── layout/
│   │   │   ├── AppLayout.tsx                # Layout logado (sidebar + header + main)
│   │   │   ├── Sidebar.tsx                  # Menu fixo lateral (RN-015)
│   │   │   ├── SidebarItem.tsx
│   │   │   ├── AuthLayout.tsx               # Layout público (logo + form)
│   │   │   └── ErrorLayout.tsx
│   │   ├── domain/                          # Componentes de domínio compartilhados
│   │   │   ├── CenarioRetornoBadge.tsx      # Badge A/B/C/D
│   │   │   ├── StatusCasoBadge.tsx
│   │   │   ├── SimuladorCenarios.tsx        # Simulador reutilizável (wizard + escalonamento)
│   │   │   ├── ValorMonetario.tsx           # Formatação de valores (R$ X.XXX,XX)
│   │   │   └── CountdownTimer.tsx           # Timer reutilizável (propostas, reversão)
│   │   └── feedback/
│   │       ├── LoadingSpinner.tsx
│   │       ├── EmptyState.tsx
│   │       └── ErrorBoundary.tsx
│   ├── hooks/                               # Hooks globais
│   │   ├── useAuth.ts                       # Sessão JWT global
│   │   ├── useToast.ts
│   │   ├── useMediaQuery.ts                 # Responsividade (mobile/tablet/desktop)
│   │   └── useSupabaseRealtime.ts           # Wrapper de Supabase Realtime
│   ├── lib/                                 # Utilitários e configurações
│   │   ├── api-client.ts                    # TanStack Query client + interceptors JWT
│   │   ├── supabase.ts                      # Supabase browser client
│   │   ├── validations/
│   │   │   ├── cpf.ts                       # Validação de CPF (dígito verificador)
│   │   │   ├── cnpj.ts                      # Validação de CNPJ
│   │   │   └── documento.ts                 # Validação de MIME type + tamanho
│   │   ├── formatters/
│   │   │   ├── currency.ts                  # R$ X.XXX,XX
│   │   │   ├── date.ts                      # date-fns + pt-BR locale
│   │   │   └── cpf-cnpj.ts                  # Mascaramento XXX.XXX.XXX-XX
│   │   └── constants/
│   │       ├── cenarios.ts                  # Dados dos 4 cenários de retorno
│   │       ├── status-caso.ts               # Labels dos 13 status visíveis
│   │       └── documentos.ts                # Lista dos 6/8 documentos obrigatórios
│   ├── services/
│   │   ├── analytics.ts                     # Wrapper PostHog (eventos em snake_case)
│   │   └── sentry.ts                        # Wrapper Sentry (error tracking)
│   ├── store/                               # Zustand stores globais
│   │   └── notification.store.ts            # Contagem de notificações não lidas
│   ├── types/                               # TypeScript types locais
│   │   ├── api.types.ts                     # Tipos dos responses da API
│   │   └── domain.types.ts                  # Tipos de domínio (Caso, Proposta, etc.)
│   └── middleware.ts                        # Next.js middleware (auth guard + redirect)
├── public/
│   ├── icons/
│   └── images/
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json                            # Extends @repasse/tsconfig/nextjs.json
├── .env.local                               # Variáveis de ambiente locais (não commitadas)
└── package.json
```

---

## 3. Mobile — `apps/mobile-cedente/`

### 3.1 Estrutura Completa

```
apps/mobile-cedente/
├── src/
│   ├── app/                                 # expo-router 4 — file-based routing
│   │   ├── (public)/                        # Telas sem autenticação
│   │   │   ├── _layout.tsx                  # Stack layout público
│   │   │   ├── index.tsx                    # Tela de boas-vindas
│   │   │   ├── login.tsx
│   │   │   └── cadastro.tsx
│   │   ├── (authenticated)/                 # Telas com autenticação
│   │   │   ├── _layout.tsx                  # Tabs + Stack layout logado
│   │   │   ├── (tabs)/
│   │   │   │   ├── _layout.tsx              # Bottom tabs: Casos / Notificações / Perfil
│   │   │   │   ├── casos/
│   │   │   │   │   ├── index.tsx            # Lista de casos
│   │   │   │   │   ├── [id]/index.tsx       # Detalhe do caso
│   │   │   │   │   ├── [id]/documentos.tsx  # Upload de documentos (câmera — RN-087)
│   │   │   │   │   ├── [id]/propostas.tsx   # Propostas (aceitar/recusar/contrapropor)
│   │   │   │   │   └── [id]/assinaturas.tsx # Assinaturas ZapSign touch (RN-087)
│   │   │   │   ├── notificacoes/
│   │   │   │   │   └── index.tsx            # Lista de notificações
│   │   │   │   └── perfil/
│   │   │   │       └── index.tsx            # Perfil e configurações
│   │   │   └── guardiao/
│   │   │       └── index.tsx                # Chat Guardião (Stack, fora das tabs)
│   │   └── +not-found.tsx
│   ├── features/                            # Funcionalidades por domínio (espelha web-cedente)
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   └── RegisterForm.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useLogin.ts
│   │   │   └── index.ts
│   │   ├── casos/
│   │   │   ├── components/
│   │   │   │   ├── CasoCard.tsx
│   │   │   │   └── CasoStatusBadge.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useCasos.ts
│   │   │   └── index.ts
│   │   ├── documentos/                      # Funcionalidade crítica mobile (RN-087)
│   │   │   ├── components/
│   │   │   │   ├── DocumentoChecklist.tsx
│   │   │   │   ├── CameraUploadButton.tsx   # Upload por câmera (expo-camera)
│   │   │   │   └── UploadProgressBar.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useUploadDocumento.ts
│   │   │   └── index.ts
│   │   ├── propostas/                       # Funcionalidade crítica mobile (RN-087)
│   │   │   ├── components/
│   │   │   │   ├── PropostaCard.tsx
│   │   │   │   ├── AceitarPropostaBottomSheet.tsx # Dupla confirmação touch
│   │   │   │   └── ContrapropostaInput.tsx
│   │   │   ├── hooks/
│   │   │   │   └── usePropostas.ts
│   │   │   └── index.ts
│   │   ├── assinaturas/                     # Funcionalidade crítica mobile (RN-087)
│   │   │   ├── components/
│   │   │   │   ├── EnvelopeCard.tsx
│   │   │   │   └── ZapSignWebView.tsx       # ZapSign via WebView touch-friendly
│   │   │   ├── hooks/
│   │   │   │   └── useAssinaturas.ts
│   │   │   └── index.ts
│   │   ├── notificacoes/
│   │   │   ├── components/
│   │   │   │   └── NotificacaoItem.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useNotificacoes.ts
│   │   │   └── index.ts
│   │   └── guardiao/
│   │       ├── components/
│   │       │   ├── GuardiaoChat.tsx
│   │       │   └── MessageBubble.tsx
│   │       ├── hooks/
│   │       │   └── useGuardiao.ts
│   │       └── index.ts
│   ├── components/                          # Componentes de UI reutilizáveis (React Native)
│   │   ├── ui/
│   │   │   ├── Button.tsx                   # Altura mínima 44px (RN-087)
│   │   │   ├── Input.tsx
│   │   │   ├── Badge.tsx
│   │   │   └── Skeleton.tsx
│   │   ├── layout/
│   │   │   ├── SafeArea.tsx
│   │   │   └── KeyboardAware.tsx
│   │   └── domain/
│   │       ├── CenarioRetornoBadge.tsx
│   │       └── StatusCasoBadge.tsx
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   └── useSupabaseRealtime.ts
│   ├── lib/
│   │   ├── api-client.ts                    # Axios + interceptors JWT
│   │   ├── supabase.ts                      # Supabase React Native client
│   │   └── formatters/
│   │       ├── currency.ts
│   │       └── date.ts
│   ├── store/
│   │   └── auth.store.ts                    # Zustand — sessão JWT no SecureStore
│   └── types/
│       ├── api.types.ts
│       └── domain.types.ts
├── assets/
│   ├── icons/
│   └── images/
├── app.json                                 # Config Expo
├── eas.json                                 # Config EAS Build/Submit
├── babel.config.js
├── tsconfig.json
└── package.json
```

---

## 4. Backend API — `apps/api/src/modules/` (Módulo Cedente)

### 4.1 Estrutura dos Módulos do Cedente no NestJS

```
apps/api/
├── src/
│   ├── modules/
│   │   ├── auth/                            # Autenticação — Cedente e outros módulos
│   │   │   ├── auth.module.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── strategies/
│   │   │   │   └── jwt.strategy.ts          # Passport JWT
│   │   │   ├── guards/
│   │   │   │   ├── jwt-auth.guard.ts
│   │   │   │   └── roles.guard.ts
│   │   │   └── dto/
│   │   │       ├── register-cedente.dto.ts
│   │   │       ├── login.dto.ts
│   │   │       └── reset-password.dto.ts
│   │   ├── cedentes/                        # Perfil e conta do Cedente
│   │   │   ├── cedentes.module.ts
│   │   │   ├── cedentes.controller.ts
│   │   │   ├── cedentes.service.ts
│   │   │   ├── cedentes.repository.ts       # Prisma queries isoladas
│   │   │   ├── dto/
│   │   │   │   ├── update-cedente.dto.ts
│   │   │   │   └── update-notification-preferences.dto.ts
│   │   │   └── entities/
│   │   │       └── cedente.entity.ts        # Tipagem de retorno da API
│   │   ├── casos/                           # Ciclo de vida do caso
│   │   │   ├── casos.module.ts
│   │   │   ├── casos.controller.ts
│   │   │   ├── casos.service.ts
│   │   │   ├── casos.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-caso-draft.dto.ts
│   │   │   │   ├── update-caso.dto.ts
│   │   │   │   ├── confirmar-caso.dto.ts
│   │   │   │   ├── set-cenario.dto.ts
│   │   │   │   └── cancelar-caso.dto.ts
│   │   │   └── entities/
│   │   │       └── caso.entity.ts
│   │   ├── dossie/                          # Gestão do dossiê e upload de documentos
│   │   │   ├── dossie.module.ts
│   │   │   ├── dossie.controller.ts
│   │   │   ├── dossie.service.ts
│   │   │   ├── dossie.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── request-upload-url.dto.ts
│   │   │   │   ├── confirm-upload.dto.ts
│   │   │   │   └── reenviar-documento.dto.ts
│   │   │   └── entities/
│   │   │       └── documento.entity.ts
│   │   ├── propostas/                       # Propostas recebidas pelo Cedente
│   │   │   ├── propostas.module.ts
│   │   │   ├── propostas.controller.ts
│   │   │   ├── propostas.service.ts
│   │   │   ├── propostas.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── aceitar-proposta.dto.ts
│   │   │   │   ├── recusar-proposta.dto.ts
│   │   │   │   └── contraproposta.dto.ts
│   │   │   └── entities/
│   │   │       └── proposta.entity.ts
│   │   ├── assinaturas/                     # Envelopes ZapSign
│   │   │   ├── assinaturas.module.ts
│   │   │   ├── assinaturas.controller.ts
│   │   │   ├── assinaturas.service.ts
│   │   │   ├── assinaturas.repository.ts
│   │   │   ├── webhooks/
│   │   │   │   └── zapsign-webhook.handler.ts # POST /webhooks/zapsign
│   │   │   ├── dto/
│   │   │   │   └── zapsign-webhook.dto.ts
│   │   │   └── entities/
│   │   │       └── envelope.entity.ts
│   │   ├── escrow/                          # Conta Escrow (somente leitura para Cedente)
│   │   │   ├── escrow.module.ts
│   │   │   ├── escrow.controller.ts
│   │   │   ├── escrow.service.ts
│   │   │   ├── escrow.repository.ts
│   │   │   ├── webhooks/
│   │   │   │   └── escrow-webhook.handler.ts  # POST /webhooks/escrow
│   │   │   └── entities/
│   │   │       └── conta-escrow.entity.ts
│   │   ├── notificacoes/                    # Notificações ao Cedente
│   │   │   ├── notificacoes.module.ts
│   │   │   ├── notificacoes.controller.ts
│   │   │   ├── notificacoes.service.ts
│   │   │   ├── notificacoes.repository.ts
│   │   │   ├── workers/
│   │   │   │   ├── email.worker.ts          # Consome fila cedente.email.queue
│   │   │   │   └── push.worker.ts           # Consome fila cedente.push.queue
│   │   │   └── entities/
│   │   │       └── notificacao.entity.ts
│   │   ├── guardiao/                        # Guardião do Retorno (IA)
│   │   │   ├── guardiao.module.ts
│   │   │   ├── guardiao.controller.ts       # SSE endpoint
│   │   │   ├── guardiao.service.ts
│   │   │   ├── guardiao.repository.ts
│   │   │   ├── prompts/
│   │   │   │   ├── system.prompt.ts         # System prompt — proibido revelar Cessionário
│   │   │   │   ├── user.prompt.ts
│   │   │   │   └── tools.ts                 # Function calling tools
│   │   │   ├── chains/
│   │   │   │   ├── guardiao.chain.ts        # LangChain pipeline
│   │   │   │   └── rag.chain.ts             # RAG com pgvector
│   │   │   └── entities/
│   │   │       └── ai-session.entity.ts
│   │   └── anuencia/                        # Anuência da construtora
│   │       ├── anuencia.module.ts
│   │       ├── anuencia.controller.ts
│   │       ├── anuencia.service.ts
│   │       ├── anuencia.repository.ts
│   │       └── dto/
│   │           └── registrar-anuencia.dto.ts
│   ├── common/                              # Utilitários e middleware transversais
│   │   ├── filters/
│   │   │   └── http-exception.filter.ts     # Formata todos os erros no padrão {error: {code, message}}
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts       # Pino — log de request/response com requestId
│   │   │   ├── transform.interceptor.ts     # Wrap de responses em { data: ... }
│   │   │   └── timeout.interceptor.ts       # Timeout de 30s por request
│   │   ├── pipes/
│   │   │   └── validation.pipe.ts           # ValidationPipe global (class-validator)
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts    # @CurrentUser() — injeta JWT payload
│   │   │   └── public.decorator.ts          # @Public() — pula JwtAuthGuard
│   │   └── utils/
│   │       ├── cpf.util.ts                  # Validação de CPF
│   │       ├── cnpj.util.ts                 # Validação de CNPJ + Receita Federal
│   │       ├── mime.util.ts                 # Validação de MIME type real
│   │       └── pagination.util.ts           # Helpers de paginação offset-based
│   ├── infrastructure/                      # Serviços de infraestrutura
│   │   ├── prisma/
│   │   │   ├── prisma.module.ts
│   │   │   └── prisma.service.ts            # PrismaClient com soft delete middleware
│   │   ├── redis/
│   │   │   ├── redis.module.ts
│   │   │   └── redis.service.ts             # Upstash Redis com prefixo rs:cedente:
│   │   ├── rabbitmq/
│   │   │   ├── rabbitmq.module.ts
│   │   │   └── rabbitmq.service.ts          # CloudAMQP — publish + consume
│   │   ├── supabase/
│   │   │   ├── supabase.module.ts
│   │   │   ├── supabase-auth.service.ts     # createUser, verifyEmail, resetPassword
│   │   │   └── supabase-storage.service.ts  # Signed URLs, validação MIME
│   │   └── resend/
│   │       ├── resend.module.ts
│   │       └── resend.service.ts            # Templates React Email
│   ├── app.module.ts                        # Root module — importa todos os módulos
│   ├── app.controller.ts                    # GET /health
│   └── main.ts                              # Bootstrap NestJS (Helmet, CORS, Swagger)
├── test/
│   ├── unit/                                # Testes unitários por módulo
│   │   ├── casos/
│   │   ├── propostas/
│   │   └── dossie/
│   └── e2e/                                 # Testes E2E (supertest)
│       ├── auth.e2e-spec.ts
│       ├── casos.e2e-spec.ts
│       └── propostas.e2e-spec.ts
├── nest-cli.json
├── tsconfig.json
├── tsconfig.build.json
└── package.json
```

---

## 5. Convenções de Nomenclatura

### 5.1 Pastas e Arquivos

| **Elemento** | **Convenção** | **Exemplos** |
|---|---|---|
| Pastas (todos os contextos) | `kebab-case` | `web-cedente/`, `casos/`, `shared-types/` |
| Componentes React | `PascalCase.tsx` | `CasoCard.tsx`, `PropostaTimerBadge.tsx` |
| Hooks React | `camelCase.ts` com prefixo `use` | `useCasos.ts`, `useUploadDocumento.ts` |
| Services (frontend) | `camelCase.service.ts` | `casos.service.ts` |
| Store Zustand | `camelCase.store.ts` | `wizard.store.ts`, `auth.store.ts` |
| Classes NestJS | `PascalCase` | `CasosController`, `PropostasService` |
| Arquivos NestJS | `kebab-case.tipo.ts` | `casos.controller.ts`, `propostas.service.ts` |
| DTOs NestJS | `kebab-case.dto.ts` | `create-caso-draft.dto.ts` |
| Entities | `kebab-case.entity.ts` | `caso.entity.ts` |
| Types compartilhados | `PascalCase` | `CasoEntity`, `PropostaEntity` |

### 5.2 Barrel Exports Obrigatórios

Todo `feature/` deve ter um `index.ts` com export de:
- Todos os componentes públicos
- Todos os hooks públicos
- Todos os types públicos

**Regra:** imports cross-feature só são permitidos via barrel (`import { CasoCard } from '@/features/casos'`). Imports diretos de arquivos internos de outra feature são proibidos.

```typescript
// features/casos/index.ts — exemplo
export { CasoCard } from './components/CasoCard';
export { CasoStatusBadge } from './components/CasoStatusBadge';
export { useCasos } from './hooks/useCasos';
export { useCaso } from './hooks/useCaso';
export type { CasoListItem, CasoDetalhe } from './types';
```

### 5.3 Aliases de Importação

Configurados no `tsconfig.json` de cada app:

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@repasse/shared-types": ["../../packages/shared-types/src"],
      "@repasse/design-tokens": ["../../packages/design-tokens/src"]
    }
  }
}
```

---

## 6. Mapeamento de Error Codes por Módulo

| **Prefixo** | **Módulo** | **Range** | **Exemplos** |
|---|---|---|---|
| `AUTH` | AuthModule | AUTH-001 a AUTH-099 | AUTH-001: Token inválido, AUTH-005: Conta bloqueada, AUTH-010: Rate limit |
| `CED` | CedentesModule | CED-001 a CED-099 | CED-001: CPF duplicado, CED-002: E-mail duplicado, CED-010: CNPJ irregular |
| `CAS` | CasosModule | CAS-001 a CAS-099 | CAS-010: Imóvel duplicado, CAS-027: Cooldown escalonamento, CAS-029: Subida de cenário |
| `DOC` | DossieModule | DOC-001 a DOC-099 | DOC-001: Formato inválido, DOC-002: Arquivo muito grande, DOC-003: Imutável |
| `PRP` | PropostasModule | PRP-001 a PRP-099 | PRP-001: Proposta expirada, PRP-010: Valor abaixo do piso, PRP-020: Máx. contrapropostas |
| `ASS` | AssinaturasModule | ASS-001 a ASS-099 | ASS-001: ZapSign indisponível, ASS-010: Token inválido, ASS-020: Doc imutável |
| `ESC` | EscrowModule | ESC-001 a ESC-099 | ESC-001: Parceiro indisponível, ESC-010: Estorno já processado |
| `NOT` | NotificacoesModule | NOT-001 a NOT-099 | NOT-001: Notificação não encontrada |
| `AI` | GuardiaoModule | AI-001 a AI-099 | AI-001: OpenAI indisponível, AI-010: Rate limit, AI-020: Sessão inválida |
| `ANU` | AnuenciaModule | ANU-001 a ANU-099 | ANU-001: Anuência negada, ANU-010: Construtora não responde |
| `COM` | CommonModule | COM-001 a COM-099 | COM-001: Validação falhou, COM-002: Recurso não encontrado (404), COM-003: Proibido (403) |

---

## 7. Variáveis de Ambiente

### 7.1 Backend (`apps/api/.env`)

```bash
# Database
DATABASE_URL="postgresql://user:pass@host:5432/repasse_seguro?schema=public"
DIRECT_URL="postgresql://user:pass@host:5432/repasse_seguro"

# Supabase
SUPABASE_URL="https://xxx.supabase.co"
SUPABASE_SERVICE_KEY="service_role_key_here"

# Redis (Upstash)
REDIS_URL="rediss://user:pass@host:6379"

# RabbitMQ (CloudAMQP)
RABBITMQ_URL="amqps://user:pass@host/vhost"

# JWT
JWT_SECRET="secret_here"
JWT_EXPIRES_IN="15m"
JWT_REFRESH_EXPIRES_IN="30d"

# ZapSign
ZAPSIGN_API_TOKEN="token_here"
ZAPSIGN_WEBHOOK_SECRET="hmac_secret_here"

# Escrow (definir após escolha do parceiro — DP-001)
ESCROW_API_URL="[DEFINICAO PENDENTE — DP-001]"
ESCROW_API_KEY="[DEFINICAO PENDENTE — DP-001]"
ESCROW_WEBHOOK_SECRET="[DEFINICAO PENDENTE — DP-001]"

# Receita Federal
RECEITA_FEDERAL_API_URL="https://publica.cnpj.ws/cnpj"

# Resend
RESEND_API_KEY="re_xxx"
RESEND_FROM_EMAIL="noreply@repasseseguro.com.br"

# OpenAI
OPENAI_API_KEY="sk-xxx"
OPENAI_MODEL="gpt-4-turbo"

# Langfuse
LANGFUSE_SECRET_KEY="sk-lf-xxx"
LANGFUSE_PUBLIC_KEY="pk-lf-xxx"
LANGFUSE_HOST="https://cloud.langfuse.com"

# Sentry
SENTRY_DSN="https://xxx@sentry.io/xxx"

# PostHog
POSTHOG_API_KEY="phc_xxx"
```

### 7.2 Frontend Web (`apps/web-cedente/.env.local`)

```bash
# API
NEXT_PUBLIC_API_URL="http://localhost:3001/api/v1"

# Supabase (client-side — apenas chaves públicas)
NEXT_PUBLIC_SUPABASE_URL="https://xxx.supabase.co"
NEXT_PUBLIC_SUPABASE_ANON_KEY="anon_key_here"

# Analytics (client-side)
NEXT_PUBLIC_POSTHOG_KEY="phc_xxx"
NEXT_PUBLIC_POSTHOG_HOST="https://app.posthog.com"
NEXT_PUBLIC_SENTRY_DSN="https://xxx@sentry.io/xxx"
```

---

## 8. Regras de Governança de Código

| **Regra** | **Descrição** | **Violação** |
|---|---|---|
| Barrel exports obrigatórios | Todo `feature/` tem `index.ts` com todos os exports públicos | PR bloqueado pelo lint |
| Imports cross-feature via barrel | `import { X } from '@/features/y'` — nunca `import { X } from '@/features/y/components/X'` | Erro de lint (import/no-internal-modules) |
| Sem `any` TypeScript | `@typescript-eslint/no-explicit-any` configurado como error | PR bloqueado pelo type-check |
| Componentes funcionais apenas | Class components React são proibidos | Code review |
| Sem `fetch` direto em `useEffect` | Usar sempre TanStack Query com retry e cache | Code review |
| Arquivos `.js`/`.jsx` proibidos | TypeScript exclusivo em todos os apps e packages | PR bloqueado pelo lint |
| Imports de `@prisma/client` apenas no backend | Frontend nunca importa Prisma Client diretamente | Lint + type-check |
| Valores monetários em centavos na API | Frontend converte para display; backend trabalha em inteiros | Code review + testes |
| Sem `cessionario_id` em endpoints do Cedente | Anonimato estrutural (RN-085) | Code review + security review |
