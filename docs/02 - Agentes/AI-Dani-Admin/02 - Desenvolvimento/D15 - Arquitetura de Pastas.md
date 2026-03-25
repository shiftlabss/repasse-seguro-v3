# Arquitetura de Pastas — AI-Dani-Admin

## Estrutura do Monorepo, Convenções de Nomenclatura e Mapeamento de Erros/Cache

| Campo | Valor |
|---|---|
| Destinatário | Tech Lead, Arquiteto, DevOps |
| Escopo | Estrutura do monorepo, convenções de nomenclatura, padrões de módulo e mapeamento de erros/cache |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D01 (Regras de Negócio), D02 (Stacks), D13 (Schema Prisma), D14 (Especificações Técnicas) |

---

> **📌 TL;DR**
>
> - **AI-Dani-Admin é módulo backend exclusivo** — sem `apps/web/` ou `apps/mobile/` próprios. Expõe API REST consumida pelo painel Admin da plataforma Repasse Seguro.
> - **Monorepo:** `apps/api/` (NestJS), `prisma/`, `packages/` (shared-types, design-tokens, eslint-config, tsconfig).
> - **Backend:** 7 módulos NestJS por domínio + 2 módulos de suporte (`audit`, `common`). Padrão rígido: `Controller → Service → Repository → DTO`.
> - **Convenções:** `kebab-case` para arquivos e pastas; `PascalCase` para classes; `camelCase` para variáveis; `UPPER_SNAKE_CASE` para constantes e enums de banco.
> - **Redis:** 4 chaves mapeadas com TTL explícito. Prefixo `dani-admin:`.
> - **Error Prefixes:** 1 prefixo único por módulo. Formato `DA-{MÓDULO}-{CODE}`.
> - **0 pendências críticas** — estrutura completa derivável dos docs D01, D02, D13 e D14.

---

## 1. Visão Geral do Monorepo

O AI-Dani-Admin é um módulo backend sem frontend próprio. Integra ao monorepo da plataforma Repasse Seguro como um app NestJS isolado.

```
repasse-seguro/                          # Monorepo raiz da plataforma
├── apps/
│   ├── api/                             # ← AI-Dani-Admin — módulo backend NestJS
│   │   └── (detalhado na seção 2)
│   ├── web/                             # Painel Admin web (React+Vite) — FORA deste módulo
│   └── mobile/                          # App mobile (React Native) — FORA deste módulo
│
├── packages/
│   ├── shared-types/                    # Types TypeScript compartilhados entre apps
│   ├── design-tokens/                   # Tokens CSS/TS de design (D03)
│   ├── eslint-config/                   # Configuração ESLint compartilhada
│   └── tsconfig/                        # Configurações TypeScript base
│
├── prisma/                              # Schema Prisma, migrations, seed
│   └── (detalhado na seção 3)
│
├── docs/                                # Documentação técnica (este pipeline)
│   └── 02 - Agentes/AI-Dani-Admin/
│
├── .github/
│   └── workflows/                       # CI/CD pipelines
│
├── docker-compose.yml                   # PostgreSQL + Redis + RabbitMQ para dev local
├── .env.example                         # Template de variáveis de ambiente
└── turbo.json                           # Configuração Turborepo (build orchestration)
```

> **⚙️ Monorepo manager:** [DECISÃO AUTÔNOMA] Turborepo — justificativa: pipeline de build paralelo nativo, caching de outputs, configuração simples para monorepos TypeScript; alternativa descartada: Nx (mais complexo, overhead desnecessário para 2-3 apps).

---

## 2. Estrutura do Backend — `apps/api/`

```
apps/api/
├── src/
│   ├── main.ts                                  # Bootstrap NestJS — app.listen(), Helmet, Swagger, ValidationPipe global
│   ├── app.module.ts                            # Root module — imports de todos os módulos de domínio
│   │
│   ├── supervision/                             # Módulo 1 — Supervisão de Interações (RF-001 a RF-004)
│   │   ├── supervision.module.ts
│   │   ├── supervision.controller.ts            # GET /admin/interactions, GET /admin/interactions/:id
│   │   ├── supervision.service.ts               # Lógica de filtros, paginação, sinalização por threshold
│   │   ├── supervision.repository.ts            # Queries Prisma (interactions) — sempre filtra deleted_at IS NULL
│   │   ├── dto/
│   │   │   ├── get-interactions-filter.dto.ts   # Query params: agentId, userId, period, confidenceRange
│   │   │   ├── interaction-list-response.dto.ts # Shape da lista paginada
│   │   │   └── interaction-detail-response.dto.ts # Shape do detalhe individual
│   │   └── tests/
│   │       ├── supervision.controller.spec.ts
│   │       └── supervision.service.spec.ts
│   │
│   ├── takeover/                                # Módulo 2 — Takeover Manual (RF-007 a RF-011)
│   │   ├── takeover.module.ts
│   │   ├── takeover.controller.ts               # POST /admin/takeover, DELETE /admin/takeover/:id, GET /admin/takeover/:id
│   │   ├── takeover.service.ts                  # Lock otimista (INSERT ON CONFLICT), notificação usuário, log auditoria
│   │   ├── takeover.repository.ts               # Queries Prisma (takeovers, interactions)
│   │   ├── takeover.gateway.ts                  # WebSocket Gateway para chat em tempo real durante takeover
│   │   ├── dto/
│   │   │   ├── start-takeover.dto.ts            # interactionId (UUID), reason (string, opcional)
│   │   │   ├── end-takeover.dto.ts              # takeoverId (UUID)
│   │   │   └── takeover-response.dto.ts
│   │   └── tests/
│   │       ├── takeover.controller.spec.ts
│   │       ├── takeover.service.spec.ts
│   │       └── takeover.gateway.spec.ts
│   │
│   ├── metrics/                                 # Módulo 3 — Dashboard de Métricas (RF-012, RF-013)
│   │   ├── metrics.module.ts
│   │   ├── metrics.controller.ts                # GET /admin/metrics?agentId=&period=
│   │   ├── metrics.service.ts                   # Agregações, cache Redis, consulta Langfuse
│   │   ├── metrics.repository.ts                # Queries Prisma (interactions, takeovers, alert_events)
│   │   ├── dto/
│   │   │   ├── get-metrics-filter.dto.ts        # agentId (UUID, opcional), period (day|week|month)
│   │   │   └── metrics-response.dto.ts          # volume, topQuestions, refusalRate, csat, avgLatency
│   │   └── tests/
│   │       ├── metrics.controller.spec.ts
│   │       └── metrics.service.spec.ts
│   │
│   ├── agent-config/                            # Módulo 4 — Configuração do Agente (RF-014 a RF-016)
│   │   ├── agent-config.module.ts
│   │   ├── agent-config.controller.ts           # GET /admin/agent-config/:agentId, PATCH /admin/agent-config/:agentId
│   │   ├── agent-config.service.ts              # Validação threshold 50-95, upsert, log auditoria, invalidação cache
│   │   ├── agent-config.repository.ts           # Queries Prisma (agent_configurations)
│   │   ├── dto/
│   │   │   ├── update-agent-config.dto.ts       # confidenceThreshold (Int, @Min(50) @Max(95)), rateLimitPerHour (Int)
│   │   │   └── agent-config-response.dto.ts
│   │   └── tests/
│   │       ├── agent-config.controller.spec.ts
│   │       └── agent-config.service.spec.ts
│   │
│   ├── alerts/                                  # Módulo de Alertas Automáticos (RF-005, RF-006)
│   │   ├── alerts.module.ts
│   │   ├── alerts.controller.ts                 # GET /admin/alerts, PATCH /admin/alerts/:id/acknowledge
│   │   ├── alerts.service.ts                    # Detecção de condições, publicação RabbitMQ, prioridade
│   │   ├── alerts.repository.ts                 # Queries Prisma (alert_events)
│   │   ├── alerts.scheduler.ts                  # @Cron('* * * * *') — checkAlerts() a cada 1min
│   │   ├── consumers/
│   │   │   ├── slack.consumer.ts                # @RabbitSubscribe dani-admin.alerts.slack
│   │   │   ├── email.consumer.ts                # @RabbitSubscribe dani-admin.alerts.email
│   │   │   └── push.consumer.ts                 # @RabbitSubscribe dani-admin.alerts.push
│   │   ├── dto/
│   │   │   ├── alert-event-response.dto.ts
│   │   │   └── acknowledge-alert.dto.ts         # alertId (UUID)
│   │   └── tests/
│   │       ├── alerts.service.spec.ts
│   │       └── alerts.scheduler.spec.ts
│   │
│   ├── launch-readiness/                        # Módulo 5 — Checklist de Prontidão (RF-020 a RF-025)
│   │   ├── launch-readiness.module.ts
│   │   ├── launch-readiness.controller.ts       # GET /admin/launch-readiness/:agentId, POST /admin/launch-readiness/:agentId/authorize
│   │   ├── launch-readiness.service.ts          # Validação de checklist, gates de segurança, autorização
│   │   ├── launch-readiness.repository.ts       # Queries Prisma (launch_readiness_checklists, adversarial_test_results)
│   │   ├── dto/
│   │   │   ├── checklist-response.dto.ts
│   │   │   ├── create-adversarial-test.dto.ts   # question, expectedRefusal, actualResponse, wasRefused, testedAt
│   │   │   └── authorize-launch.dto.ts
│   │   └── tests/
│   │       ├── launch-readiness.controller.spec.ts
│   │       └── launch-readiness.service.spec.ts
│   │
│   ├── push-tokens/                             # Push Notification Tokens — Admin mobile (D11)
│   │   ├── push-tokens.module.ts
│   │   ├── push-tokens.controller.ts            # POST /admin/push-tokens, DELETE /admin/push-tokens/:token
│   │   ├── push-tokens.service.ts               # Registro/revogação de tokens Expo Push
│   │   ├── push-tokens.repository.ts            # Queries Prisma (push_notification_tokens)
│   │   ├── dto/
│   │   │   └── register-push-token.dto.ts       # expoPushToken (string @IsNotEmpty), platform ('ios'|'android')
│   │   └── tests/
│   │       └── push-tokens.service.spec.ts
│   │
│   ├── audit/                                   # Módulo de Auditoria — suporte transversal (RF-004, RF-026)
│   │   ├── audit.module.ts                      # @Global() — exportado globalmente
│   │   ├── audit.service.ts                     # log(actionType, details, adminId) → INSERT admin_access_logs
│   │   ├── audit.repository.ts                  # Queries Prisma (admin_access_logs)
│   │   ├── dto/
│   │   │   └── audit-log-response.dto.ts
│   │   └── tests/
│   │       └── audit.service.spec.ts
│   │
│   └── common/                                  # Infraestrutura transversal
│       ├── guards/
│       │   └── admin-auth.guard.ts              # JWT guard — verifica role ADMIN. 403 para não-Admin.
│       ├── interceptors/
│       │   └── audit-log.interceptor.ts         # Registra acesso a endpoints sensíveis automaticamente
│       ├── filters/
│       │   └── http-exception.filter.ts         # Formata todas as exceções HTTP com shape padrão
│       ├── pipes/
│       │   └── validation.pipe.ts               # ValidationPipe global com whitelist:true, forbidNonWhitelisted:true
│       ├── decorators/
│       │   ├── admin-id.decorator.ts            # @AdminId() — extrai admin_id do JWT
│       │   └── public.decorator.ts              # @Public() — marca endpoint como sem autenticação
│       ├── interfaces/
│       │   └── paginated-response.interface.ts  # PaginatedResponse<T> — shape padrão de lista
│       └── constants/
│           └── business.constants.ts            # DEFAULT_CONFIDENCE_THRESHOLD=80, MIN=50, MAX=95, etc.
│
├── test/
│   └── e2e/
│       ├── supervision.e2e-spec.ts
│       ├── takeover.e2e-spec.ts
│       ├── metrics.e2e-spec.ts
│       ├── agent-config.e2e-spec.ts
│       ├── alerts.e2e-spec.ts
│       └── launch-readiness.e2e-spec.ts
│
├── nest-cli.json
├── tsconfig.json                                # Extende packages/tsconfig/base.json
├── tsconfig.build.json
├── package.json
├── .env                                         # Variáveis de ambiente (não commitado)
└── .env.example                                 # Template de variáveis
```

---

## 3. Estrutura Prisma — `prisma/`

```
prisma/
├── schema.prisma                        # Schema consolidado da plataforma Repasse Seguro
│                                        # Inclui modelos do AI-Dani-Admin (D13) e demais módulos
├── migrations/
│   ├── 20260323000001_create_interactions/
│   │   └── migration.sql
│   ├── 20260323000002_create_takeovers/
│   │   └── migration.sql
│   ├── 20260323000003_create_agent_configurations/
│   │   └── migration.sql
│   ├── 20260323000004_create_admin_access_logs/
│   │   └── migration.sql
│   ├── 20260323000005_create_alert_events/
│   │   └── migration.sql
│   ├── 20260323000006_create_launch_readiness_checklists/
│   │   └── migration.sql
│   ├── 20260323000007_create_adversarial_test_results/
│   │   └── migration.sql
│   └── 20260323000008_create_push_notification_tokens/
│       └── migration.sql
├── seed/
│   ├── index.ts                         # Entry point do seed — chama todos os seeders
│   └── ai-dani-admin.ts                 # Seed de AgentConfiguration (threshold=80, rate_limit=30)
└── middleware/
    └── soft-delete.middleware.ts        # Middleware Prisma para filtrar deleted_at IS NULL automaticamente
```

> **⚙️ Convenção de migrations:** Nome no formato `YYYYMMDDHHMMSS_<descricao>` em `snake_case`. Uma migration por tabela na criação inicial. Cada migration altera apenas um domínio.

---

## 4. Packages Compartilhados — `packages/`

### 4.1 `packages/shared-types/`

```
packages/shared-types/
├── src/
│   ├── index.ts                         # Barrel export de todos os tipos
│   ├── interactions.types.ts            # InteractionStatus, InteractionRow, etc.
│   ├── takeover.types.ts                # TakeoverStatus, TakeoverResponse, etc.
│   ├── alerts.types.ts                  # AlertType, AlertStatus, AlertEventResponse
│   ├── metrics.types.ts                 # MetricsPeriod, MetricsResponse
│   ├── agent-config.types.ts            # AgentConfigResponse
│   └── push-notifications.types.ts     # PushNotificationPayload
├── package.json
└── tsconfig.json
```

### 4.2 `packages/design-tokens/`

```
packages/design-tokens/
├── src/
│   ├── index.ts
│   ├── colors.ts                        # Tokens de cor (D03)
│   ├── spacing.ts                       # Tokens de espaçamento
│   ├── typography.ts                    # Tokens de tipografia
│   └── radius.ts                        # Tokens de border-radius
└── package.json
```

### 4.3 `packages/eslint-config/` e `packages/tsconfig/`

```
packages/eslint-config/
├── index.js                             # Regras ESLint compartilhadas
└── package.json

packages/tsconfig/
├── base.json                            # tsconfig base: strict:true, target:ES2022
├── nestjs.json                          # Extende base + decoratorMetadata:true
└── package.json
```

---

## 5. Estrutura Interna de Módulos Backend

### 5.1 Padrão por Módulo

Todo módulo backend segue o mesmo padrão de camadas:

```
<módulo>/
├── <módulo>.module.ts          # Definição do módulo NestJS — imports, providers, controllers, exports
├── <módulo>.controller.ts      # HTTP handlers. Zero lógica de negócio. Delega para service.
├── <módulo>.service.ts         # Lógica de negócio + orquestração. Injeta repository, audit, rabbit.
├── <módulo>.repository.ts      # Acesso ao banco via Prisma. Zero lógica de negócio.
├── dto/
│   ├── <ação>-<módulo>.dto.ts  # DTOs de entrada (request body, query params)
│   └── <módulo>-response.dto.ts # DTOs de saída (response body)
└── tests/
    ├── <módulo>.controller.spec.ts
    └── <módulo>.service.spec.ts
```

### 5.2 Regras de Responsabilidade por Camada

| Camada | O que pode | O que não pode |
|---|---|---|
| **Controller** | Receber request, validar DTOs (via Pipe), chamar service, retornar response | Lógica de negócio, acesso direto ao banco, chamar outro controller |
| **Service** | Lógica de negócio, chamar repositories, chamar AuditService, publicar no RabbitMQ | Acesso direto ao Prisma, lógica HTTP (status codes), imports de controllers |
| **Repository** | Queries Prisma, transformações de resultado para tipos de domínio | Lógica de negócio, validações, chamadas a serviços externos |
| **DTO** | Shapes de entrada/saída com decorators class-validator | Lógica computada |

### 5.3 Regras de Import (Dependências entre Camadas)

✅ **Permitido:**
```typescript
// Controller importa somente seu Service local
import { SupervisionService } from './supervision.service'

// Service importa Repository local e módulos de suporte globais
import { SupervisionRepository } from './supervision.repository'
import { AuditService } from '../audit/audit.service'

// Módulos compartilhados via packages/
import { InteractionRow } from '@repasse/shared-types'
```

🔴 **Proibido:**
```typescript
// Cross-module import direto (violação de encapsulamento)
import { TakeoverService } from '../takeover/takeover.service'   // ❌ use TakeoverModule.exports

// Import direto do Prisma no Controller ou Service
import { PrismaClient } from '@prisma/client'                    // ❌ apenas no Repository

// Import de DTO de outro módulo (use shared-types)
import { AlertEventResponse } from '../alerts/dto/alert-event-response.dto'  // ❌
```

---

## 6. Convenções de Nomenclatura

| Contexto | Convenção | Exemplo |
|---|---|---|
| Pastas | `kebab-case` | `agent-config/`, `launch-readiness/`, `push-tokens/` |
| Arquivos TypeScript | `kebab-case` + sufixo funcional | `supervision.controller.ts`, `start-takeover.dto.ts` |
| Classes NestJS (Controller, Service, Repository, Module) | `PascalCase` + sufixo | `SupervisionController`, `TakeoverService`, `AuditRepository`, `MetricsModule` |
| DTOs | `PascalCase` + sufixo `Dto` | `StartTakeoverDto`, `GetInteractionsFilterDto`, `MetricsResponseDto` |
| Interfaces | `PascalCase` + prefixo `I` (opcional) ou sem prefixo | `PaginatedResponse`, `AdminJwtPayload` |
| Enums TypeScript | `PascalCase` (type), `PascalCase` (values) | `enum InteractionStatus { RespondidaPelaIa = 'RESPONDIDA_PELA_IA' }` |
| Enums no banco (Prisma) | `UPPER_SNAKE_CASE` nos valores | `RESPONDIDA_PELA_IA`, `EM_TAKEOVER`, `DESLIGAMENTO_AUTOMATICO` |
| Variáveis e propriedades | `camelCase` | `confidenceScore`, `adminId`, `startedAt` |
| Constantes TypeScript | `UPPER_SNAKE_CASE` | `DEFAULT_CONFIDENCE_THRESHOLD`, `AUDIT_LOG_RETENTION_DAYS` |
| Métodos de Service | `camelCase`, verbo + substantivo | `getInteractions()`, `startTakeover()`, `updateThreshold()` |
| Métodos de Repository | `camelCase`, verbo + substantivo | `findAll()`, `findById()`, `create()`, `updateStatus()` |
| Arquivos de teste unitário | `<arquivo>.spec.ts` | `supervision.service.spec.ts` |
| Arquivos de teste e2e | `<módulo>.e2e-spec.ts` | `takeover.e2e-spec.ts` |
| Eventos PostHog | `snake_case` | `admin_takeover_initiated`, `threshold_updated`, `alert_acknowledged` |
| Rotas de API | `kebab-case`, plural, `/api/v1/admin/` prefix | `/api/v1/admin/interactions`, `/api/v1/admin/agent-config/:agentId` |
| Variáveis de ambiente | `UPPER_SNAKE_CASE` | `DATABASE_URL`, `REDIS_URL`, `RABBITMQ_URL`, `LANGFUSE_PUBLIC_KEY` |
| Tabelas PostgreSQL | `snake_case`, plural | `interactions`, `admin_access_logs`, `push_notification_tokens` |
| Colunas PostgreSQL | `snake_case` | `confidence_score`, `created_at`, `updated_by` |

---

## 7. Cache Redis — Convenção de Chaves

Prefixo obrigatório: `dani-admin:` em toda chave. Nunca sem TTL.

| Chave (padrão) | TTL | Contexto | Invalidação |
|---|---|---|---|
| `dani-admin:metrics:{agentId}:{period}` | 60s | Métricas agregadas do dashboard (volume, CSAT, top perguntas, latência média) | Toda vez que nova interação é registrada ou threshold é alterado |
| `dani-admin:agent-config:{agentId}` | 300s (5min) | Threshold de confiança configurado para o agente | Toda vez que PATCH /admin/agent-config/:agentId é executado com sucesso |
| `dani-admin:rate-limit:webchat:{userId}` | 3600s (1h, sliding window) | Contador de mensagens por usuário no webchat. Janela deslizante de 1h. | Expiração automática por TTL (sliding window via Redis EXPIRE no INCR) |
| `dani-admin:sent:slack:{alertEventId}` | 86400s (24h) | Flag de idempotência para alertas Slack. Previne envio duplicado em retry. | Expiração automática por TTL |

**Regras gerais:**
- Nunca armazenar PII nas chaves. `userId` é UUID, não nome/CPF.
- `SET` sem `EX` é proibido — qualquer PR que adicione cache sem TTL é rejeitado na code review.
- Chaves com `agentId` são invalidadas em cascata quando o agente é reativado após desligamento automático.

---

## 8. Mapeamento Módulo → Error Prefix → Queues

| Módulo NestJS | Error Prefix | Queues RabbitMQ |
|---|---|---|
| `SupervisionModule` | `DA-SUP` | — |
| `TakeoverModule` | `DA-TAK` | `dani-admin.takeover.notification`, `dani-admin.takeover.push` |
| `MetricsModule` | `DA-MET` | — |
| `AgentConfigModule` | `DA-CFG` | — |
| `AlertsModule` | `DA-ALT` | `dani-admin.alerts.slack`, `dani-admin.alerts.email`, `dani-admin.alerts.push`, `dani-admin.alerts.panel` |
| `LaunchReadinessModule` | `DA-LCH` | — |
| `PushTokensModule` | `DA-PSH` | — |
| `AuditModule` | `DA-AUD` | — |

**Formato de código de erro:** `DA-{MÓDULO}-{CODE}` onde `CODE` é número sequencial de 3 dígitos.

Exemplos:
- `DA-TAK-001` → Takeover conflict (interação já em atendimento)
- `DA-TAK-002` → Tentativa de takeover em conversa encerrada
- `DA-CFG-001` → Threshold fora do range 50-95%
- `DA-ALT-001` → Falha na entrega de alerta crítico após 3 retries

---

## 9. Arquivos de Configuração Raiz

```
apps/api/
├── .env.example                         # Template — todos os envs necessários documentados
├── nest-cli.json                        # entryFile: "main", sourceRoot: "src"
├── tsconfig.json
│   extends: "../../packages/tsconfig/nestjs.json"
│   paths: { "@repasse/*": ["../../packages/*/src"] }
└── tsconfig.build.json
    exclude: ["node_modules", "test", "dist", "**/*.spec.ts"]
```

**Variables de ambiente obrigatórias (`.env.example`):**

```bash
# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/repasse_seguro

# Redis
REDIS_URL=redis://localhost:6379

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@localhost:5672

# Supabase
SUPABASE_URL=https://<project>.supabase.co
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Auth
JWT_SECRET=
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# Langfuse
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com

# Sentry
SENTRY_DSN=

# PostHog
POSTHOG_API_KEY=
POSTHOG_HOST=https://app.posthog.com

# Slack
SLACK_WEBHOOK_URL=

# SendGrid
SENDGRID_API_KEY=
SENDGRID_FROM_EMAIL=alertas@repasseseguro.com.br

# Expo Push
EXPO_ACCESS_TOKEN=

# App
NODE_ENV=development
PORT=3000
SEED_ADMIN_ID=
```

---

## 10. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Estrutura completa do monorepo, 7 módulos NestJS, convenções de nomenclatura, mapeamento Redis e Error Prefixes. |
