# Repasse AI
## 15 — Arquitetura de Pastas

| Campo | Valor |
|---|---|
| **Destinatário** | Tech Lead, Arquiteto, DevOps |
| **Escopo** | Estrutura do monorepo, convenções de nomenclatura, padrões de módulo e mapeamento de erros/cache |
| **Módulo** | Repasse AI |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Repasse AI é um serviço backend puro (PG-03).** Não há `apps/web` neste módulo. A estrutura do monorepo contém `apps/ai/` (NestJS) + `prisma/` + `packages/` + `docs/`.
> - **Backend:** monolito modular NestJS 10+. Estrutura `apps/ai/src/modules/{módulo}/` com padrão `Controller → Service → Repository → DTO → Entity` por módulo.
> - **6 módulos de domínio:** `ai`, `chat`, `calculator`, `notification`, `supervision`, `whatsapp`.
> - **Convenções rígidas:** `kebab-case` para arquivos/pastas; `PascalCase` para classes; `camelCase` para variáveis/funções; `SCREAMING_SNAKE_CASE` para constantes e enums.
> - **Cache Redis:** 8 chaves mapeadas com prefixo `repasse-ai:{cessionario_id}:{domínio}:` e TTL explícito por recurso.
> - **Error Prefixes:** 1 prefixo único por módulo. Total de 6 prefixos + 3 prefixos de infraestrutura.
> - **Seções pendentes:** 0. 2 decisões autônomas aplicadas (marcadas inline).

---

## 1. Persona e Contexto

> 🎯 **Objetivo:** Este documento é a fonte única de verdade para organização do código do módulo Repasse AI. Toda estrutura de pasta, arquivo e convenção documentada aqui é normativa — qualquer desvio exige ADR aprovado.

**Quem usa este documento:**
- Desenvolvedores: criam features sem questionar onde colocar cada arquivo.
- Tech Leads: auditam a estrutura sem interpretar regras vagas.
- Claude Code / Codex: geram código respeitando a árvore sem desvios.
- DevOps: configuram pipelines de CI/CD com base na estrutura definida.

**Nota arquitetural:** O Repasse AI é um módulo dentro do monorepo maior da plataforma Repasse Seguro. Este documento cobre exclusivamente o escopo do módulo Repasse AI — o monorepo da plataforma pode ter outros apps (frontend Cessionário, frontend Admin, etc.) que não são responsabilidade deste documento.

---

## 2. Visão Geral do Monorepo (Escopo Repasse AI)

```
repasse-ai/                          # Raiz do módulo Repasse AI
├── apps/
│   └── ai/                          # Serviço NestJS — backend puro (PG-03)
│       ├── src/
│       │   ├── main.ts              # Bootstrap do NestJS + Swagger + Helmet
│       │   ├── app.module.ts        # AppModule raiz (importa todos os módulos)
│       │   ├── app.controller.ts    # Health check endpoint
│       │   ├── app.service.ts       # Health check service
│       │   ├── common/              # Guards, interceptors, filters globais
│       │   ├── config/              # Configuração por ambiente (ConfigModule)
│       │   ├── modules/             # 6 módulos de domínio
│       │   │   ├── ai/              # Orquestração do agente LLM
│       │   │   ├── chat/            # Conversas + SSE streaming
│       │   │   ├── calculator/      # Cálculo financeiro (Δ, ROI, simulação)
│       │   │   ├── notification/    # Notificações proativas
│       │   │   ├── supervision/     # Supervisão Admin + Takeover
│       │   │   └── whatsapp/        # WhatsApp binding + OTP
│       │   └── database/            # PrismaModule + PrismaService + withTenant
│       ├── test/                    # Testes e2e
│       ├── tsconfig.json
│       ├── tsconfig.build.json
│       ├── nest-cli.json
│       └── package.json
├── prisma/                          # Prisma schema, migrations, seed e RLS
│   ├── schema.prisma
│   ├── migrations/                  # Migrations geradas pelo Prisma Migrate
│   ├── seed.ts                      # Seed de 9 AgentConfiguration obrigatórias
│   ├── middleware/
│   │   ├── soft-delete.middleware.ts
│   │   └── tenant.middleware.ts
│   └── rls/
│       ├── policies.sql             # 22 políticas RLS
│       └── indexes.sql              # Índices adicionais (IVFFlat, partial unique)
├── packages/
│   ├── shared-types/                # Types TypeScript compartilhados
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── ai.types.ts
│   │   │   ├── chat.types.ts
│   │   │   ├── notification.types.ts
│   │   │   └── whatsapp.types.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   ├── eslint-config/               # Configuração ESLint compartilhada
│   │   ├── index.js
│   │   └── package.json
│   └── tsconfig/                    # tsconfig base compartilhada
│       ├── base.json
│       └── package.json
├── docs/                            # Documentação técnica do módulo
│   └── Repasse AI/
│       └── 02 - Desenvolvimento/    # Docs de desenvolvimento (este arquivo incluso)
├── .github/
│   └── workflows/
│       ├── ci.yml                   # Lint + testes em PR
│       ├── deploy-staging.yml       # Deploy automático para staging
│       └── deploy-production.yml    # Deploy manual para produção
├── docker-compose.yml               # PostgreSQL + Redis + RabbitMQ local
├── .env.example                     # Template de variáveis de ambiente
├── turbo.json                       # Turborepo: pipeline de build/test/lint
└── package.json                     # package.json raiz (workspaces)
```

> ⚙️ **Turborepo:** O monorepo usa Turborepo para orquestrar builds, testes e lint. [DECISÃO AUTÔNOMA] — Turborepo foi escolhido por ter integração nativa com Vercel e suporte a cache de builds incremental. Alternativa descartada: Nx (mais complexo para equipe pequena desenvolvida por agentes de IA).

---

## 3. Backend — apps/ai/

### 3.1 Estrutura Raiz do Serviço

```
apps/ai/src/
├── main.ts                          # Bootstrap: NestJS + Fastify + Swagger + Helmet + Pino
├── app.module.ts                    # AppModule: importa ConfigModule, DatabaseModule, todos os módulos de domínio
├── app.controller.ts                # GET /health — health check
├── app.service.ts                   # HealthService: verifica DB, Redis, RabbitMQ
├── common/
│   ├── guards/
│   │   ├── jwt-auth.guard.ts        # Valida JWT da plataforma Cessionário
│   │   ├── cessionario.guard.ts     # Extrai e valida cessionario_id do JWT
│   │   └── admin.guard.ts           # Valida role ADMIN
│   ├── interceptors/
│   │   ├── transform.interceptor.ts # Normaliza resposta: { data, meta, timestamp }
│   │   └── logging.interceptor.ts   # Log estruturado por request via Pino
│   ├── filters/
│   │   └── http-exception.filter.ts # Normaliza erros HTTP para formato padrão
│   ├── decorators/
│   │   ├── current-user.decorator.ts # @CurrentUser() extrai payload do JWT
│   │   └── cessionario-id.decorator.ts # @CessionarioId() extrai cessionario_id
│   ├── pipes/
│   │   └── validation.pipe.ts       # ValidationPipe global com class-validator
│   └── constants/
│       ├── error-prefixes.ts        # Centraliza todos os Error Prefixes
│       └── redis-keys.ts            # Centraliza todos os padrões de chaves Redis
├── config/
│   ├── app.config.ts                # Porta, ambiente, CORS
│   ├── database.config.ts           # DATABASE_URL, pool config
│   ├── redis.config.ts              # REDIS_URL, TTL defaults
│   ├── rabbitmq.config.ts           # AMQP_URL, exchange/queue names
│   ├── openai.config.ts             # OPENAI_API_KEY, modelo, temperatura
│   ├── langfuse.config.ts           # LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY
│   └── evolution.config.ts          # EVOLUTION_API_URL, EVOLUTION_API_KEY
├── modules/
│   ├── ai/                          # Módulo: Orquestração do Agente LLM
│   ├── chat/                        # Módulo: Conversas + SSE Streaming
│   ├── calculator/                  # Módulo: Cálculo Financeiro
│   ├── notification/                # Módulo: Notificações Proativas
│   ├── supervision/                 # Módulo: Supervisão Admin + Takeover
│   └── whatsapp/                    # Módulo: WhatsApp Binding + OTP
└── database/
    ├── database.module.ts            # DatabaseModule: providers PrismaService
    ├── prisma.service.ts             # PrismaService: $connect, withTenant(), soft-delete middleware, tenant middleware
    └── database.config.ts           # DATABASE_URL, connection pool
```

### 3.2 Estrutura Padrão por Módulo

Todos os 6 módulos de domínio seguem **exatamente** este padrão:

```
modules/{módulo}/
├── {módulo}.module.ts               # Definição do módulo NestJS (imports, controllers, providers, exports)
├── {módulo}.controller.ts           # Endpoints HTTP + @ApiTags + @ApiOperation + guards
├── services/
│   └── {módulo}.service.ts          # Regra de negócio + Prisma + Redis + RabbitMQ
├── repositories/
│   └── {módulo}.repository.ts       # Queries Prisma isoladas (sem lógica de negócio)
├── dto/
│   ├── create-{módulo}.dto.ts       # DTO de criação com @IsString, @IsUUID, etc.
│   ├── update-{módulo}.dto.ts       # DTO de atualização (PartialType do Create)
│   └── filter-{módulo}.dto.ts       # DTO de filtros/paginação para listagens
├── entities/
│   └── {módulo}.entity.ts           # Tipo de saída da API (baseado no model Prisma)
└── tests/
    ├── {módulo}.controller.spec.ts  # Testes unitários do Controller (mock do Service)
    ├── {módulo}.service.spec.ts     # Testes unitários do Service (mock do Repository)
    └── {módulo}.repository.spec.ts  # Testes unitários do Repository (mock do PrismaService)
```

> 🔴 **Regras de import — proibições:**
> - Service **nunca** importa outro Service diretamente → use injeção de dependência via módulo.
> - Repository **nunca** contém lógica de negócio → apenas queries Prisma.
> - Controller **nunca** acessa PrismaService diretamente → sempre via Service.
> - Imports cross-módulo **apenas** via `exports` no módulo de origem e `imports` no módulo de destino.

### 3.3 Mapeamento Completo de Módulos

#### Módulo: `ai`

```
modules/ai/
├── ai.module.ts
├── ai.controller.ts                 # POST /ai/analyze, POST /ai/compare, POST /ai/simulate
├── services/
│   ├── ai.service.ts                # Orquestração LangGraph.js + LLM
│   ├── llm.service.ts               # Wrapper OpenAI SDK + Langfuse tracing
│   ├── rag.service.ts               # RAG: embeddings + pgvector similarity search
│   └── cache.service.ts             # Cache exato + semântico (Redis)
├── repositories/
│   ├── ai.repository.ts             # Queries: ai_interactions, ai_analysis_cache
│   └── embedding.repository.ts      # Queries: document_embeddings ($queryRaw)
├── dto/
│   ├── analyze-request.dto.ts       # { message: string, conversation_id: string }
│   ├── analyze-response.dto.ts      # { content: string, confidence: number, sources: string[] }
│   └── filter-interaction.dto.ts    # Filtros para supervisão Admin
├── entities/
│   ├── ai-interaction.entity.ts     # Entidade de saída: ai_interactions
│   └── ai-analysis-cache.entity.ts  # Entidade de saída: ai_analysis_cache
└── tests/
    ├── ai.controller.spec.ts
    ├── ai.service.spec.ts
    ├── llm.service.spec.ts
    ├── rag.service.spec.ts
    └── cache.service.spec.ts
```

#### Módulo: `chat`

```
modules/chat/
├── chat.module.ts
├── chat.controller.ts               # POST /chat/messages, GET /chat/stream (SSE), GET /chat/history
├── services/
│   └── chat.service.ts              # Gerencia conversas + streaming SSE via Vercel AI SDK
├── repositories/
│   ├── conversation.repository.ts   # Queries: conversations
│   └── message.repository.ts        # Queries: messages
├── dto/
│   ├── create-message.dto.ts        # { content: string, conversation_id?: string }
│   ├── create-conversation.dto.ts   # { cessionario_id: string }
│   └── filter-conversation.dto.ts   # Filtros de histórico
├── entities/
│   ├── conversation.entity.ts       # Entidade: conversations
│   └── message.entity.ts            # Entidade: messages
└── tests/
    ├── chat.controller.spec.ts
    ├── chat.service.spec.ts
    ├── conversation.repository.spec.ts
    └── message.repository.spec.ts
```

#### Módulo: `calculator`

```
modules/calculator/
├── calculator.module.ts
├── calculator.controller.ts         # POST /calculator/delta, POST /calculator/roi, POST /calculator/portfolio
├── services/
│   └── calculator.service.ts        # Cálculos financeiros: Δ, comissão, Escrow, ROI (3 cenários)
├── repositories/
│   └── calculator.repository.ts     # Sem queries próprias — usa dados do chat/ai
├── dto/
│   ├── calculate-delta.dto.ts       # { valor_face: Decimal, valor_proposto: Decimal }
│   ├── calculate-roi.dto.ts         # { valor_investido: Decimal, cenarios: enum }
│   └── calculate-portfolio.dto.ts   # { oportunidades: CalculateDeltaDto[] }
├── entities/
│   └── calculation-result.entity.ts # Resultado com Δ, comissão, custo_escrow, roi
└── tests/
    ├── calculator.controller.spec.ts
    └── calculator.service.spec.ts
```

#### Módulo: `notification`

```
modules/notification/
├── notification.module.ts
├── notification.controller.ts       # GET /notifications, PATCH /notifications/preferences
├── services/
│   ├── notification.service.ts      # Lógica de envio e preferências
│   └── notification-dispatch.service.ts # Worker RabbitMQ: consome fila repasse.ai.notification
├── repositories/
│   └── notification.repository.ts   # Queries: notifications, notification_preferences
├── dto/
│   ├── update-preferences.dto.ts    # { type: NotificationTypeEnum, enabled: boolean }
│   └── filter-notification.dto.ts   # Filtros por tipo e status
├── entities/
│   ├── notification.entity.ts       # Entidade: notifications
│   └── notification-preference.entity.ts # Entidade: notification_preferences
└── tests/
    ├── notification.controller.spec.ts
    ├── notification.service.spec.ts
    └── notification.repository.spec.ts
```

#### Módulo: `supervision`

```
modules/supervision/
├── supervision.module.ts
├── supervision.controller.ts        # GET /supervision/interactions, GET /supervision/interactions/:id, POST /supervision/takeover, DELETE /supervision/takeover/:id
├── services/
│   └── supervision.service.ts       # Takeover mutex (SELECT FOR UPDATE), métricas
├── repositories/
│   ├── supervision.repository.ts    # Queries: ai_interactions (Admin view)
│   └── takeover.repository.ts       # Queries: ai_takeovers (SELECT FOR UPDATE)
├── dto/
│   ├── create-takeover.dto.ts       # { interaction_id: string, reason: string }
│   ├── filter-interaction.dto.ts    # Filtros: data, confiança, canal
│   └── metrics-query.dto.ts         # Período de métricas para dashboard
├── entities/
│   ├── interaction-detail.entity.ts # Entidade detalhada de ai_interactions
│   └── takeover.entity.ts           # Entidade: ai_takeovers
└── tests/
    ├── supervision.controller.spec.ts
    ├── supervision.service.spec.ts
    └── takeover.repository.spec.ts
```

#### Módulo: `whatsapp`

```
modules/whatsapp/
├── whatsapp.module.ts
├── whatsapp.controller.ts           # POST /whatsapp/bind, POST /whatsapp/verify-otp, DELETE /whatsapp/bind
├── services/
│   ├── whatsapp.service.ts          # Binding, OTP, state machine
│   └── whatsapp-webhook.service.ts  # Processa webhooks do EvolutionAPI
├── repositories/
│   └── whatsapp.repository.ts       # Queries: whatsapp_bindings
├── dto/
│   ├── bind-whatsapp.dto.ts         # { phone_number: string (BR format) }
│   ├── verify-otp.dto.ts            # { phone_number: string, otp_code: string }
│   └── webhook-event.dto.ts         # Payload do webhook EvolutionAPI
├── entities/
│   └── whatsapp-binding.entity.ts   # Entidade: whatsapp_bindings
└── tests/
    ├── whatsapp.controller.spec.ts
    ├── whatsapp.service.spec.ts
    └── whatsapp.repository.spec.ts
```

### 3.4 Estrutura do `database/`

```
database/
├── database.module.ts               # @Global() DatabaseModule — providers: [PrismaService]; exports: [PrismaService]
└── prisma.service.ts                # extends PrismaClient; onModuleInit(), onModuleDestroy(), withTenant(cessionarioId, fn)
```

**`PrismaService.withTenant()` pattern:**
```typescript
// apps/ai/src/database/prisma.service.ts
async withTenant<T>(cessionarioId: string, fn: (prisma: PrismaClient) => Promise<T>): Promise<T> {
  return this.$transaction(async (tx) => {
    await tx.$executeRaw`SET LOCAL app.cessionario_id = ${cessionarioId}`;
    return fn(tx);
  });
}
```

### 3.5 Estrutura de Testes e2e

```
apps/ai/test/
├── app.e2e-spec.ts                  # Health check e2e
├── chat.e2e-spec.ts                 # Fluxo completo de chat + streaming
├── ai-analysis.e2e-spec.ts          # Análise de oportunidade end-to-end
├── whatsapp-binding.e2e-spec.ts     # Fluxo OTP completo
├── supervision-takeover.e2e-spec.ts # Takeover mutex com concorrência
└── jest-e2e.json                    # Config Jest para e2e (testEnvironment: node)
```

---

## 4. Packages

### 4.1 `packages/shared-types/`

```
packages/shared-types/
├── src/
│   ├── index.ts                     # Barrel: re-exporta todos os types
│   ├── ai.types.ts                  # AnalysisResult, InteractionStatus, ConfidenceLevel
│   ├── chat.types.ts                # Message, Conversation, ConversationStatus, MessageRole
│   ├── notification.types.ts        # Notification, NotificationType, NotificationStatus
│   └── whatsapp.types.ts            # WhatsappBinding, WhatsappBindingStatus, OtpState
├── tsconfig.json
└── package.json
```

> ⚙️ **Regra de uso:** `packages/shared-types` é o único ponto de compartilhamento de tipos entre o backend (`apps/ai`) e potenciais consumidores externos (frontend do Cessionário). Nunca importar tipos diretamente do modelo Prisma em código externo ao serviço.

### 4.2 `packages/eslint-config/`

```
packages/eslint-config/
├── index.js                         # ESLint: extends @nestjs/eslint-config + regras customizadas
├── nestjs.js                        # Config específica para NestJS
└── package.json
```

### 4.3 `packages/tsconfig/`

```
packages/tsconfig/
├── base.json                        # strict: true, target: ES2022, module: NodeNext
├── nestjs.json                      # extends base + decoratorMetadata: true + emitDecoratorMetadata: true
└── package.json
```

---

## 5. Convenções de Nomenclatura

### 5.1 Tabela Completa

| Contexto | Convenção | Exemplo |
|---|---|---|
| **Pastas** | `kebab-case` | `modules/chat/`, `modules/ai/`, `dto/`, `repositories/` |
| **Arquivos TypeScript** | `kebab-case.tipo.ts` | `chat.service.ts`, `create-message.dto.ts`, `jwt-auth.guard.ts` |
| **Arquivos de teste** | `kebab-case.spec.ts` ou `kebab-case.e2e-spec.ts` | `chat.service.spec.ts`, `chat.e2e-spec.ts` |
| **Classes NestJS (Module, Service, Controller, Repository, Guard, Filter, Interceptor, Pipe)** | `PascalCase` | `ChatService`, `ConversationRepository`, `JwtAuthGuard` |
| **DTOs** | `PascalCase` + sufixo `Dto` | `CreateMessageDto`, `FilterInteractionDto` |
| **Entidades** | `PascalCase` + sufixo `Entity` | `MessageEntity`, `AiInteractionEntity` |
| **Interfaces** | `PascalCase` + prefixo `I` opcional | `IAnalysisResult` ou `AnalysisResult` (preferir sem prefixo) |
| **Enums** | `PascalCase` + sufixo `Enum` | `ConversationStatusEnum`, `WhatsappBindingStatusEnum` |
| **Constantes** | `SCREAMING_SNAKE_CASE` | `MAX_MESSAGES_PER_HOUR`, `DEFAULT_LLM_MODEL` |
| **Variáveis e parâmetros** | `camelCase` | `cessionarioId`, `conversationId`, `analysisResult` |
| **Funções e métodos** | `camelCase` | `findByConversationId()`, `withTenant()`, `calculateDelta()` |
| **Decorators customizados** | `PascalCase` | `@CurrentUser()`, `@CessionarioId()` |
| **Variáveis de ambiente** | `SCREAMING_SNAKE_CASE` | `DATABASE_URL`, `OPENAI_API_KEY`, `REDIS_URL` |
| **Chaves Redis** | `kebab-case` com `:` como separador | `repasse-ai:{uuid}:chat:history` |
| **Exchanges RabbitMQ** | `kebab-case` com `.` como separador | `repasse.ai.rag`, `repasse.ai.notification` |
| **Queues RabbitMQ** | `kebab-case` com `.` como separador | `repasse.ai.rag.ingest`, `repasse.ai.notification.send` |
| **Tabelas Prisma (PostgreSQL)** | `snake_case` plural | `conversations`, `ai_interactions`, `whatsapp_bindings` |
| **Colunas Prisma (PostgreSQL)** | `snake_case` | `created_at`, `cessionario_id`, `confidence_score` |
| **Models Prisma** | `PascalCase` singular | `Conversation`, `AiInteraction`, `WhatsappBinding` |

### 5.2 Regras Adicionais de Nomenclatura

| Regra | Detalhe |
|---|---|
| Arquivos de módulo NestJS | Sempre `{módulo}.module.ts`, `{módulo}.controller.ts`, `{módulo}.service.ts` — nunca abreviados |
| DTOs de criação vs. atualização | `Create{Entidade}Dto` (todos campos obrigatórios) e `Update{Entidade}Dto` (todos opcionais via `PartialType`) |
| Nomes de arquivos = nomes de classes | `chat.service.ts` exporta `ChatService`; `create-message.dto.ts` exporta `CreateMessageDto` |
| Nunca usar `index.ts` para barrel de módulos backend | Cada arquivo backend é importado diretamente pelo nome; barrels são reservados para `packages/` |
| Teste espelha o arquivo testado | `chat.service.spec.ts` testa `ChatService`; arquivo na mesma pasta (`tests/`) |

---

## 6. Cache Redis — Convenção de Chaves

> ⚙️ **Padrão:** `repasse-ai:{cessionario_id}:{domínio}:{recurso}[:{id}]`
>
> Todas as chaves usam `repasse-ai:` como prefixo de produto. Tenant isolado por `{cessionario_id}` (UUID). TTL explícito — nunca sem TTL. Mudança de versão de prompt invalida cache correspondente do domínio `ai`.

| Chave (padrão) | TTL | Contexto | Invalidação |
|---|---|---|---|
| `repasse-ai:{cessionario_id}:ai:cache:exact:{sha256}` | 86400s (24h) | Cache exato de LLM — hash SHA-256 de `{model}+{prompt_version}+{input}` | Ao atualizar versão de prompt ou modelo |
| `repasse-ai:{cessionario_id}:ai:cache:semantic:{embedding_hash}` | 3600s (1h) | Cache semântico — embedding da query + threshold 0.92 cosine similarity | TTL; sem invalidação manual |
| `repasse-ai:{cessionario_id}:chat:history:{conversation_id}` | 1800s (30min) | Histórico recente da conversa para contexto do LLM | Ao iniciar nova conversa; ao apagar histórico |
| `repasse-ai:{cessionario_id}:chat:rate-limit` | 3600s (1h) | Contador de mensagens por hora (webchat: 30 msg/h) | Janela deslizante de 1h |
| `repasse-ai:{cessionario_id}:whatsapp:rate-limit` | 3600s (1h) | Contador de mensagens por hora (WhatsApp: 20 msg/h) | Janela deslizante de 1h |
| `repasse-ai:{cessionario_id}:whatsapp:otp:{phone_hash}` | 300s (5min) | OTP de vinculação WhatsApp — hash SHA-256 do número | Ao verificar OTP com sucesso; ao expirar |
| `repasse-ai:{cessionario_id}:whatsapp:otp:attempts:{phone_hash}` | 900s (15min) | Contador de tentativas de OTP (máx: 3) | Ao atingir 3 tentativas (usuário bloqueado); ao expirar |
| `repasse-ai:global:supervision:metrics:{period}` | 300s (5min) | Métricas agregadas para Dashboard Admin — sem isolamento por tenant | Ao atualizar período; expiração natural |

> 💡 **Centralização:** Todos os padrões de chave Redis são definidos em `apps/ai/src/common/constants/redis-keys.ts` como constantes tipadas. Nenhuma string de chave Redis pode ser escrita inline em Services ou Repositories.

```typescript
// apps/ai/src/common/constants/redis-keys.ts
export const REDIS_KEYS = {
  AI_CACHE_EXACT: (cessionarioId: string, hash: string) =>
    `repasse-ai:${cessionarioId}:ai:cache:exact:${hash}`,
  AI_CACHE_SEMANTIC: (cessionarioId: string, embeddingHash: string) =>
    `repasse-ai:${cessionarioId}:ai:cache:semantic:${embeddingHash}`,
  CHAT_HISTORY: (cessionarioId: string, conversationId: string) =>
    `repasse-ai:${cessionarioId}:chat:history:${conversationId}`,
  CHAT_RATE_LIMIT: (cessionarioId: string) =>
    `repasse-ai:${cessionarioId}:chat:rate-limit`,
  WHATSAPP_RATE_LIMIT: (cessionarioId: string) =>
    `repasse-ai:${cessionarioId}:whatsapp:rate-limit`,
  WHATSAPP_OTP: (cessionarioId: string, phoneHash: string) =>
    `repasse-ai:${cessionarioId}:whatsapp:otp:${phoneHash}`,
  WHATSAPP_OTP_ATTEMPTS: (cessionarioId: string, phoneHash: string) =>
    `repasse-ai:${cessionarioId}:whatsapp:otp:attempts:${phoneHash}`,
  SUPERVISION_METRICS: (period: string) =>
    `repasse-ai:global:supervision:metrics:${period}`,
} as const;
```

---

## 7. Mapeamento Módulo → Error Prefix

> ⚙️ **Padrão:** `[PREFIXO] Mensagem de erro descritiva`. Prefixo único por módulo. Facilita grep de logs e Sentry grouping.

| Módulo NestJS | Error Prefix | Queue(s) RabbitMQ |
|---|---|---|
| `AiModule` | `[AI]` | `repasse.ai.rag.ingest`, `repasse.ai.rag.ingest.dlq` |
| `ChatModule` | `[CHAT]` | — |
| `CalculatorModule` | `[CALC]` | — |
| `NotificationModule` | `[NOTIF]` | `repasse.ai.notification.send`, `repasse.ai.notification.send.dlq`, `repasse.ai.notification.whatsapp`, `repasse.ai.notification.whatsapp.dlq` |
| `SupervisionModule` | `[SUPER]` | — |
| `WhatsappModule` | `[WA]` | `repasse.ai.whatsapp.webhook`, `repasse.ai.whatsapp.webhook.dlq` |
| `DatabaseModule` | `[DB]` | — |
| `AppModule` (health) | `[APP]` | — |
| `Global / HTTP Filter` | `[HTTP]` | — |

> 💡 **Centralização de Error Prefixes:**

```typescript
// apps/ai/src/common/constants/error-prefixes.ts
export const ERROR_PREFIXES = {
  AI: '[AI]',
  CHAT: '[CHAT]',
  CALCULATOR: '[CALC]',
  NOTIFICATION: '[NOTIF]',
  SUPERVISION: '[SUPER]',
  WHATSAPP: '[WA]',
  DATABASE: '[DB]',
  APP: '[APP]',
  HTTP: '[HTTP]',
} as const;
```

**Exemplo de uso em Service:**
```typescript
// apps/ai/src/modules/chat/services/chat.service.ts
import { ERROR_PREFIXES } from '../../../common/constants/error-prefixes';

throw new InternalServerErrorException(
  `${ERROR_PREFIXES.CHAT} Failed to create conversation: ${error.message}`
);
```

---

## 8. Regras de Import e Dependências Entre Camadas

### 8.1 Hierarquia de Dependências (Backend)

```
Controller
    └── Service
            ├── Repository
            │       └── PrismaService (via DatabaseModule)
            ├── RedisService (via @nestjs-modules/ioredis ou ioredis direto)
            └── RabbitMQService (via @nestjs/microservices)
```

### 8.2 Regras de Import — Exemplos

✅ **Correto:**
```typescript
// chat.service.ts importa ConversationRepository via injeção
@Injectable()
export class ChatService {
  constructor(
    private readonly conversationRepo: ConversationRepository,
    private readonly aiService: AiService, // via import do AiModule
  ) {}
}
```

🔴 **Incorreto — import direto cross-módulo sem DI:**
```typescript
// PROIBIDO: chat.service.ts importando PrismaService diretamente
import { PrismaService } from '../../database/prisma.service';
// Use ConversationRepository como intermediário
```

🔴 **Incorreto — controller acessando Prisma:**
```typescript
// PROIBIDO: controller acessando PrismaService ou Repository diretamente
@Controller('chat')
export class ChatController {
  constructor(private prisma: PrismaService) {} // PROIBIDO
}
```

✅ **Correto — module declara exports para compartilhamento:**
```typescript
// ai.module.ts
@Module({
  providers: [AiService, LlmService, RagService, CacheService, AiRepository, EmbeddingRepository],
  controllers: [AiController],
  exports: [AiService], // ChatModule pode importar AiModule e injetar AiService
})
export class AiModule {}
```

### 8.3 Dependências entre Módulos

| Módulo que importa | Módulo importado | Motivo |
|---|---|---|
| `ChatModule` | `AiModule` | Envia mensagem → aciona orquestração do agente |
| `NotificationModule` | `ChatModule` | Entrega alertas proativos via conversa no chat |
| `SupervisionModule` | `ChatModule`, `AiModule` | Acessa interações + assume conversa em takeover |
| `WhatsappModule` | `NotificationModule` | Entrega notificações via WhatsApp |

---

## 9. Prisma — Estrutura da Camada `prisma/`

```
prisma/
├── schema.prisma                    # 13 models, 17 enums, datasource Supabase
├── migrations/
│   ├── migration_lock.toml
│   └── {timestamp}_{descricao}/
│       └── migration.sql            # SQL gerado pelo Prisma Migrate
├── seed.ts                          # 9 AgentConfiguration com upsert + skipDuplicates
├── middleware/
│   ├── soft-delete.middleware.ts    # Intercepta delete → update(deleted_at: now())
│   └── tenant.middleware.ts         # Logging de contexto; SET LOCAL via withTenant()
└── rls/
    ├── policies.sql                 # 22 RLS policies (SELECT + INSERT por tabela)
    └── indexes.sql                  # IVFFlat pgvector, partial unique WhatsApp, partial index takeovers
```

> ⚙️ **Convenção de migrations:** nome obrigatório no formato `{timestamp}_{verbo}_{entidade}_{descrição}`.
> Ex: `20260322000000_create_conversations_and_messages`
> Ex: `20260325120000_add_confidence_score_to_ai_interactions`

---

## 10. Estrutura de CI/CD

```
.github/workflows/
├── ci.yml                           # Trigger: PR → main ou staging
│                                    # Jobs: lint (ESLint + Prettier) → type-check (tsc --noEmit) → test:unit → test:e2e
├── deploy-staging.yml               # Trigger: push → staging
│                                    # Jobs: build → prisma migrate deploy → deploy Railway (staging)
└── deploy-production.yml            # Trigger: manual (workflow_dispatch) + tag semver
│                                    # Jobs: build → prisma migrate deploy → deploy Railway (production)
```

```
turbo.json                           # Pipeline:
                                     # build: depends on ^build; outputs: dist/**
                                     # test: depends on build
                                     # lint: no dependencies
                                     # type-check: no dependencies
```

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 22/03/2026 | v1.0 | Versão inicial. Monorepo completo para Repasse AI: apps/ai (NestJS), prisma/, packages/. 6 módulos de domínio. Convenções de nomenclatura completas. 8 chaves Redis com TTL. 9 Error Prefixes. CI/CD mapeado. |

---

## 12. Backlog de Pendências

| Item | Marcador | Seção / Módulo | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Turborepo como orquestrador de monorepo | [DECISÃO AUTÔNOMA] | 2. Visão Geral | Turborepo: integração nativa Vercel, cache incremental de builds, curva de aprendizado menor para equipes de agentes IA. Alternativa descartada: Nx (mais complexo, overhead desnecessário para estágio atual). | P1 | Tech Lead | Decidido |
| `apps/ai` como nome do diretório do serviço NestJS | [DECISÃO AUTÔNOMA] | 2. Visão Geral | O serviço principal do Repasse AI é o backend NestJS. `apps/ai/` é o nome semântico correto dado que o módulo é centrado em IA. `apps/api/` seria mais genérico e poderia colidir com o naming de outros módulos da plataforma. Alternativa descartada: `apps/api/` (ambíguo no contexto do monorepo maior do Repasse Seguro). | P2 | Tech Lead | Decidido |

---

*Próximo documento do pipeline: D16 — Documentação de API.*
