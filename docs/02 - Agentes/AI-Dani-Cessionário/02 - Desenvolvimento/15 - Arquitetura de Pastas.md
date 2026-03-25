# 15 - Arquitetura de Pastas

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Tech Lead, Arquiteto, DevOps | Estrutura do monorepo do AI-Dani-Cessionário — convenções de nomenclatura, padrões de módulo e mapeamento de erros/cache | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Monorepo:** `apps/web` (React 19 + Vite) + `apps/api` (NestJS) + `packages/` (shared) + `prisma/` + `docs/`. Sem `apps/ai/` separado — IA está dentro de `apps/api/src/modules/agente/`.
> - **Frontend feature-first:** cada feature em `features/<módulo>/` com barrel export obrigatório. Features da Dani: `dani-chat`, `dani-dashboard`, `dani-whatsapp`.
> - **Backend módulo-por-módulo:** 7 módulos NestJS — `agente`, `calculadora`, `oportunidade`, `cessionario`, `alerta`, `whatsapp`, `auth`. Padrão rígido: Controller → Service → Repository → DTO → Entity.
> - **Nomenclatura:** `kebab-case` em pastas/arquivos, `PascalCase` em componentes/classes, `camelCase` em variáveis/funções, `SCREAMING_SNAKE_CASE` em constantes/enums.
> - **Cache Redis:** 5 chaves com prefixo `dani:` + TTL explícito. Zero chaves sem TTL.
> - **Error Prefixes:** 7 módulos mapeados — `AGENTE_`, `CALC_`, `OPR_`, `CESS_`, `ALERTA_`, `WA_`, `AUTH_`.
> - **Zero seções pendentes.**

---

## 1. Persona

Arquiteto de Software sênior responsável por organizar o monorepo do AI-Dani-Cessionário para previsibilidade, rastreabilidade e coerência entre camadas. Este documento é o contrato de estrutura para o time de engenharia e para o Claude Code ao gerar código.

---

## 2. Visão Geral do Monorepo

```
repasse-seguro-dani/               # Raiz do módulo AI-Dani-Cessionário
├── apps/
│   ├── web/                       # Frontend SPA — React 19 + Vite
│   └── api/                       # Backend — NestJS 10 + Node.js 20
├── packages/
│   ├── shared-types/              # Types compartilhados entre web e api
│   ├── design-tokens/             # Tokens CSS da Dani (extensões do D03)
│   ├── eslint-config/             # Config ESLint base do monorepo
│   └── tsconfig/                  # Config TypeScript base
├── prisma/
│   ├── schema.prisma              # Schema único (fonte de verdade do D13)
│   ├── migrations/                # Migrations geradas pelo Prisma
│   ├── seed/
│   │   └── seed.ts                # Seed de dados de desenvolvimento
│   └── middleware/
│       └── rbac.middleware.ts     # Middleware de injeção de cessionario_id
├── docs/                          # Documentação técnica (output pipeline D04-D29)
├── .github/
│   └── workflows/                 # CI/CD (ver D24)
├── .env.example                   # Template de variáveis de ambiente
├── package.json                   # Workspace root (pnpm workspaces)
├── pnpm-workspace.yaml            # Configuração de workspaces
├── turbo.json                     # Turborepo — build cache e pipeline
└── README.md
```

> ⚙️ **Gerenciador de pacotes:** `pnpm` com workspaces. `turbo` para cache de build e execução paralela. [DECISÃO AUTÔNOMA — pnpm + turbo é o padrão ShiftLabs para monorepos TypeScript; alternativa descartada: npm workspaces (sem cache de build nativo).]

---

## 3. Frontend — `apps/web/`

### 3.1 Estrutura Completa

```
apps/web/
├── src/
│   ├── features/                  # Feature-first — cada domínio isolado
│   │   ├── dani-chat/             # Chat da Dani (T-DC-001 a T-DC-011)
│   │   ├── dani-dashboard/        # Widget Top 3 no Dashboard (T-DC-009)
│   │   └── dani-whatsapp/         # Vinculação WhatsApp — Fase 2 (T-DC-012)
│   ├── components/                # Componentes globais reutilizáveis (sem domínio)
│   │   ├── ui/                    # Componentes shadcn/ui instalados
│   │   └── shared/                # Componentes custom compartilhados (ex: EmptyState)
│   ├── hooks/                     # Hooks globais (ex: useNetworkStatus, useReducedMotion)
│   ├── stores/                    # Zustand stores globais
│   │   └── chat.store.ts          # Estado global do chat (isChatOpen, chatContext)
│   ├── services/                  # Clientes de API globais (fetch wrappers)
│   │   └── api.client.ts          # Instância base com JWT header
│   ├── lib/                       # Utilitários e helpers (formatação monetária, etc.)
│   │   ├── format.ts              # R$ XX.XXX,XX formatter
│   │   └── cn.ts                  # classnames/clsx helper (shadcn padrão)
│   ├── types/                     # Types globais (não-específicos de feature)
│   ├── router/                    # TanStack Router — definição de rotas
│   │   ├── routes.tsx             # Árvore de rotas com lazy loading
│   │   └── guards/                # Route guards (autenticação, Fase 2)
│   ├── styles/
│   │   └── globals.css            # Tokens CSS base + extensões da Dani (D03)
│   └── main.tsx                   # Entry point
├── public/                        # Assets estáticos (favicon, ícone Dani)
├── index.html
├── vite.config.ts
├── tailwind.config.ts             # Tailwind v4 config com tokens da Dani
├── tsconfig.json                  # Extends packages/tsconfig/web.json
└── package.json
```

### 3.2 Estrutura Interna de cada Feature

```
features/dani-chat/
├── index.ts                       # Barrel export — API pública do módulo
├── pages/
│   └── DaniChatPage.tsx           # Entry point da feature (se tiver rota própria)
├── components/
│   ├── DaniFab.tsx                # T-DC-001 — FAB global
│   ├── DaniChatWindow.tsx         # T-DC-002/003/004 — Chat window
│   ├── DaniMessageBubble.tsx      # Bubble de mensagem
│   ├── DaniConversationStarters.tsx # 4 chips de conversation starters
│   ├── DaniTypingIndicator.tsx    # Typing indicator (3 pontos)
│   ├── DaniRateLimitBanner.tsx    # T-DC-011 — State de rate limit
│   ├── DaniAgentStatusBanner.tsx  # T-DC-010 — Banner de fallback
│   ├── DaniAnalysisCard.tsx       # T-DC-005 — Card de análise inline
│   ├── DaniComparisonTable.tsx    # T-DC-006 — Tabela comparativa
│   ├── DaniSimulationCard.tsx     # T-DC-007/008 — Card de simulação
│   ├── RiskScoreBadge.tsx         # Badge de score de risco (domínio)
│   ├── OprStatusBadge.tsx         # Badge de status da oportunidade
│   └── ROIDeltaBadge.tsx          # Badge de variação de ROI
├── hooks/
│   ├── useDaniChat.ts             # Hook principal do chat (open/close/send)
│   ├── useDaniStream.ts           # Hook para SSE streaming (Vercel AI SDK)
│   ├── useDaniAlerts.ts           # Hook para contagem de alertas não lidos
│   └── useRateLimitTimer.ts       # Hook para contador regressivo do rate limit
├── services/
│   ├── dani-chat.service.ts       # POST /api/v1/dani/chat
│   ├── dani-stream.service.ts     # GET /api/v1/dani/stream (SSE)
│   └── dani-alertas.service.ts    # GET /api/v1/alertas/count
├── stores/
│   └── dani-chat.store.ts         # Zustand: contexto do chat, mensagens em cache
├── types/
│   ├── dani-chat.types.ts         # Types locais da feature
│   └── dani-message.types.ts      # Types de mensagem e metadata
└── tests/
    ├── DaniFab.test.tsx
    ├── DaniChatWindow.test.tsx
    └── useDaniChat.test.ts
```

```
features/dani-dashboard/
├── index.ts
├── components/
│   └── DaniTop3Widget.tsx         # T-DC-009 — Widget Top 3
├── hooks/
│   └── useDaniTop3.ts             # Hook para Top 3 recomendações
├── services/
│   └── dani-dashboard.service.ts  # GET /api/v1/dani/top3
├── types/
│   └── dani-dashboard.types.ts
└── tests/
    └── DaniTop3Widget.test.tsx
```

```
features/dani-whatsapp/            # Fase 2
├── index.ts
├── components/
│   ├── WhatsappVinculacaoSection.tsx # T-DC-012 — Seção em Meu Perfil
│   ├── WhatsappOTPInput.tsx          # Input de 6 dígitos
│   └── WhatsappStatusBadge.tsx       # Badge de estado de vinculação
├── hooks/
│   └── useDaniWhatsapp.ts
├── services/
│   └── dani-whatsapp.service.ts
├── types/
│   └── dani-whatsapp.types.ts
└── tests/
    └── WhatsappVinculacaoSection.test.tsx
```

### 3.3 Regras de Import

✅ **Correto — via barrel export:**
```typescript
import { DaniFab, useDaniChat, RiskScoreBadge } from '@/features/dani-chat'
```

🔴 **Proibido — import direto cross-feature:**
```typescript
// PROIBIDO: import direto bypassa a API pública da feature
import DaniFab from '@/features/dani-chat/components/DaniFab'
```

✅ **Correto — componente global:**
```typescript
import { EmptyState } from '@/components/shared'
```

🔴 **Proibido — feature importando outra feature diretamente:**
```typescript
// PROIBIDO: dani-dashboard não pode importar internals de dani-chat
import { DaniChatWindow } from '@/features/dani-chat/components/DaniChatWindow'
// CORRETO: usa barrel
import { DaniChatWindow } from '@/features/dani-chat'
```

**Alias de paths obrigatórios em `vite.config.ts` e `tsconfig.json`:**
```
@/ → src/
@features/ → src/features/
@components/ → src/components/
@lib/ → src/lib/
@types/ → src/types/
```

---

## 4. Backend — `apps/api/`

### 4.1 Estrutura Completa

```
apps/api/
├── src/
│   ├── common/                    # Código transversal a todos os módulos
│   │   ├── decorators/
│   │   │   └── current-cessionario.decorator.ts  # @CurrentCessionario()
│   │   ├── filters/
│   │   │   └── http-exception.filter.ts          # Filtro global de exceções
│   │   ├── guards/
│   │   │   ├── cessionario-owner.guard.ts        # CessionarioOwnerGuard
│   │   │   └── jwt-auth.guard.ts                 # JwtAuthGuard
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts            # Log de request/response (Pino)
│   │   │   └── timeout.interceptor.ts            # Timeout global (30s)
│   │   └── pipes/
│   │       └── validation.pipe.ts                # Global ValidationPipe
│   ├── config/                    # Configurações carregadas de variáveis de ambiente
│   │   ├── database.config.ts     # DATABASE_URL
│   │   ├── redis.config.ts        # REDIS_URL
│   │   ├── rabbitmq.config.ts     # RABBITMQ_URL
│   │   ├── openai.config.ts       # OPENAI_API_KEY
│   │   ├── langfuse.config.ts     # LANGFUSE_SECRET_KEY
│   │   └── app.config.ts          # PORT, NODE_ENV, CORS
│   ├── modules/
│   │   ├── auth/                  # AuthModule
│   │   ├── agente/                # AgenteModule
│   │   ├── calculadora/           # CalculadoraModule
│   │   ├── oportunidade/          # OportunidadeModule
│   │   ├── cessionario/           # CessionarioModule
│   │   ├── alerta/                # AlertaModule
│   │   └── whatsapp/              # WhatsappModule (Fase 2)
│   ├── database/
│   │   ├── prisma.service.ts      # PrismaService — singleton injetável
│   │   └── prisma.module.ts       # PrismaModule global
│   ├── jobs/
│   │   └── cleanup-conversas.job.ts  # Job de limpeza 90 dias (D13 seção 4.3)
│   ├── app.module.ts              # Módulo raiz — importa todos os módulos
│   └── main.ts                    # Bootstrap NestJS
├── test/
│   └── app.e2e-spec.ts            # Testes e2e
├── tsconfig.json                  # Extends packages/tsconfig/api.json
├── tsconfig.build.json
└── package.json
```

### 4.2 Estrutura Interna de cada Módulo

Padrão rígido para todos os 7 módulos:

```
modules/agente/
├── agente.module.ts               # Declaração do módulo + DI
├── agente.controller.ts           # Endpoints HTTP: POST /dani/chat, GET /dani/stream
├── services/
│   ├── agente.service.ts          # Orquestração principal do agente
│   ├── agente-context.service.ts  # Montagem de contexto (RAG + histórico)
│   └── agente-monitor.service.ts  # Monitoramento de SLA e estado
├── repositories/
│   └── agente.repository.ts       # Queries Prisma: DaniConversa, DaniMensagem, DaniSessao
├── dto/
│   ├── create-chat.dto.ts         # { mensagem: string, conversa_id?: string }
│   ├── chat-response.dto.ts       # Resposta serializada
│   └── context-opr.dto.ts         # Contexto de oportunidade injetado
├── entities/
│   └── conversa.entity.ts         # Tipo derivado do Prisma (re-exportado com métodos)
├── prompts/
│   └── dani-system-prompt.v1.ts  # System prompt tipado — nunca hardcoded
└── tests/
    ├── agente.service.spec.ts
    ├── agente.controller.spec.ts
    └── agente.repository.spec.ts
```

```
modules/calculadora/
├── calculadora.module.ts
├── calculadora.controller.ts      # POST /calculadora/calcular (exposição opcional)
├── services/
│   └── calculadora.service.ts     # Fórmulas determinísticas (D01 RN-DC-013 a RN-DC-020)
├── repositories/
│   └── calculadora.repository.ts  # Read-only: tabelas de oportunidades
├── dto/
│   ├── calcular-comissao.dto.ts   # { opr_id, valor_proposta }
│   └── calculo-result.dto.ts      # { delta, comissao, custo_total, roi_* }
├── entities/
│   └── calculo.entity.ts
└── tests/
    └── calculadora.service.spec.ts
```

```
modules/alerta/
├── alerta.module.ts
├── alerta.controller.ts           # GET /alertas, PATCH /alertas/:id/lido
├── services/
│   └── alerta.service.ts          # CRUD de alertas + consumer da fila
├── repositories/
│   └── alerta.repository.ts       # Queries em dani_alertas
├── dto/
│   ├── create-alerta.dto.ts
│   └── alerta-response.dto.ts
├── entities/
│   └── alerta.entity.ts
├── consumers/
│   └── notificacoes.consumer.ts   # Consumer RabbitMQ dani.notificacoes
└── tests/
    └── alerta.service.spec.ts
```

```
modules/whatsapp/                  # Fase 2
├── whatsapp.module.ts
├── whatsapp.controller.ts         # POST /whatsapp/vincular, POST /whatsapp/verificar-otp, POST /whatsapp/webhook
├── services/
│   ├── whatsapp.service.ts        # Lógica de vinculação + envio
│   └── evolution-api.service.ts   # HTTP client para EvolutionAPI
├── repositories/
│   └── whatsapp.repository.ts     # Queries em dani_vinculacoes_whatsapp
├── dto/
│   ├── iniciar-vinculacao.dto.ts
│   ├── verificar-otp.dto.ts
│   └── webhook-payload.dto.ts
├── entities/
│   └── vinculacao.entity.ts
├── consumers/
│   └── whatsapp-saida.consumer.ts # Consumer RabbitMQ dani.whatsapp
└── tests/
    └── whatsapp.service.spec.ts
```

> 💡 Os módulos `auth/`, `oportunidade/` e `cessionario/` seguem o mesmo padrão sem consumers (não usam filas).

---

## 5. Convenções de Nomenclatura

| Contexto | Convenção | Exemplo |
|---|---|---|
| Pastas | `kebab-case` | `dani-chat/`, `dani-whatsapp/` |
| Arquivos de componente React | `PascalCase.tsx` | `DaniFab.tsx`, `RiskScoreBadge.tsx` |
| Arquivos de hook | `camelCase` prefixo `use` `.ts` | `useDaniChat.ts`, `useRateLimitTimer.ts` |
| Arquivos de store Zustand | `kebab-case.store.ts` | `dani-chat.store.ts` |
| Arquivos de service (frontend) | `kebab-case.service.ts` | `dani-chat.service.ts` |
| Arquivos de types | `kebab-case.types.ts` | `dani-message.types.ts` |
| Barrel export | `index.ts` | `features/dani-chat/index.ts` |
| Classes NestJS (Controller) | `PascalCase` sufixo `Controller` | `AgenteController` |
| Classes NestJS (Service) | `PascalCase` sufixo `Service` | `CalculadoraService` |
| Classes NestJS (Repository) | `PascalCase` sufixo `Repository` | `AlertaRepository` |
| DTOs | `PascalCase` sufixo `Dto` | `CreateChatDto`, `AlculoResultDto` |
| Entities | `PascalCase` sufixo `Entity` | `ConversaEntity` |
| Módulos NestJS | `PascalCase` sufixo `Module` | `AgenteModule` |
| Arquivos NestJS | `kebab-case.<tipo>.ts` | `agente.controller.ts`, `agente.service.ts` |
| Testes unitários | `*.spec.ts` (co-located) | `agente.service.spec.ts` |
| Testes e2e | `*.e2e-spec.ts` em `/test/` | `app.e2e-spec.ts` |
| Enums | `PascalCase` (Prisma) / `SCREAMING_SNAKE_CASE` (valores) | `enum CanalDani { WEBCHAT, WHATSAPP }` |
| Constantes | `SCREAMING_SNAKE_CASE` | `MAX_MENSAGENS_POR_HORA = 30` |
| Variáveis | `camelCase` | `cessionarioId`, `conversaAtiva` |
| Funções | `camelCase` verbo + substantivo | `calcularComissao()`, `buscarOportunidade()` |
| Interfaces TypeScript | `PascalCase` prefixo `I` (opcional) ou sem prefixo | `ChatContext` ou `IChatContext` [DECISÃO AUTÔNOMA — sem prefixo I: padrão mais moderno, consistente com Prisma types gerados; alternativa descartada: prefixo I (convenção legada, desnecessária com TS moderno)] |
| CSS custom properties (tokens) | `--kebab-case` com namespace | `--risk-low`, `--agent-fallback` |
| Variáveis de ambiente | `SCREAMING_SNAKE_CASE` | `OPENAI_API_KEY`, `DATABASE_URL` |

---

## 6. Cache Redis — Convenção de Chaves

> ⚙️ **Regra absoluta:** Toda chave Redis começa com prefixo `dani:`. TTL explícito obrigatório — nunca usar `SET` sem `EX`. Nunca armazenar dados pessoais em texto claro.

| Chave (padrão) | TTL | Módulo | Contexto |
|---|---|---|---|
| `dani:rate:webchat:{cessionario_id}` | 3.600s | AgenteModule | Rate limit de 30 msgs/hora por janela deslizante (RN-DC-025) |
| `dani:rate:otp:{phone_hash}` | 3.600s | WhatsappModule | Rate limit de 3 tentativas OTP/hora (RN-DC-041) |
| `dani:block:otp:{phone_hash}` | 1.800s | WhatsappModule | Hard block após 5 falhas consecutivas (RN-DC-041) |
| `dani:cache:calc:{opr_id}:{valor_hash}` | 300s | CalculadoraModule | Cache de resultado determinístico da Calculadora |
| `dani:status:agent` | 60s | AgenteModule | Status operacional do agente (OPERACIONAL/FALLBACK/DESLIGADO) |

> 💡 `{phone_hash}` = SHA-256 do número completo (nunca armazenar número em texto claro na chave). `{valor_hash}` = SHA-256 do valor da proposta (para evitar key explosion com floats).

---

## 7. Mapeamento Módulo → Error Prefix

| Módulo NestJS | Error Prefix | Queue(s) RabbitMQ |
|---|---|---|
| `AuthModule` | `AUTH_` | — |
| `AgenteModule` | `AGENTE_` | `dani.agent_monitor` |
| `CalculadoraModule` | `CALC_` | — |
| `OportunidadeModule` | `OPR_` | — |
| `CessionarioModule` | `CESS_` | — |
| `AlertaModule` | `ALERTA_` | `dani.notificacoes` (producer + consumer) |
| `WhatsappModule` | `WA_` | `dani.whatsapp` (producer + consumer) |

**Padrão de código de erro:**

```typescript
// Formato: {ERROR_PREFIX}{DOMÍNIO}_{TIPO}
// Exemplos:
'AGENTE_STREAM_TIMEOUT'      // AgenteModule — timeout do streaming
'CALC_INVALID_VALUE'         // CalculadoraModule — valor inválido
'WA_OTP_HARD_BLOCK'          // WhatsappModule — hard block OTP
'AUTH_JWT_EXPIRED'           // AuthModule — JWT expirado
'OPR_NOT_FOUND'              // OportunidadeModule — OPR não existe
```

**Exceções tipadas (padrão por módulo):**

```typescript
// Cada módulo exporta suas próprias exceções de /common/exceptions/
export class AgenteStreamTimeoutException extends HttpException {
  constructor() {
    super({ code: 'AGENTE_STREAM_TIMEOUT', message: 'Timeout no streaming da Dani' }, 504)
  }
}
```

---

## Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. Monorepo completo com apps/web (3 features da Dani), apps/api (7 módulos NestJS), packages, prisma. Convenções de nomenclatura, chaves Redis com prefixo `dani:` e TTL, error prefixes por módulo. |

---

## Backlog de Pendências

| Item | Marcador | Seção/Módulo | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Interfaces TypeScript sem prefixo `I` | Decisão Autônoma | §5 Nomenclatura | Padrão moderno sem prefixo `I` — consistente com Prisma types. Alternativa `I`-prefix descartada (legada) | P3 | Tech Lead | Aberto |
| pnpm + Turborepo como gerenciador | Decisão Autônoma | §2 Visão Geral | Padrão ShiftLabs para monorepos TS. Alternativa npm workspaces descartada (sem cache de build nativo) | P2 | DevOps | Aberto |
| SHA-256 nas chaves Redis de telefone e valor | Decisão Autônoma | §6 Cache Redis | Evita armazenamento de dados pessoais em texto claro nas chaves + previne key explosion. Alternativa UUID descartada (não determinístico — impossibilitaria lookups) | P1 | Backend Lead | Aberto |
| `apps/ai/` separado | Decisão Autônoma | §2 Visão Geral | IA dentro de `apps/api/src/modules/agente/` — sem microserviço separado. Volume atual não justifica overhead operacional. Alternativa microserviço Python descartada (complexidade prematura) | P2 | Tech Lead | Aberto |
