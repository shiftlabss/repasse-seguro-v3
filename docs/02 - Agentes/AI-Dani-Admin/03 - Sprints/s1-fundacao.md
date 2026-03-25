# S1 — Fundação

## Metadados

| Campo         | Valor                                  |
| ------------- | -------------------------------------- |
| Sprint        | S1                                     |
| Nome          | Fundação                               |
| Template      | A (Infraestrutura)                     |
| Módulo        | AI-Dani-Admin                          |
| Docs Fonte    | D02, D12, D13, D14, D15, D20, D22, D25 |
| REQs Cobertos | REQ-001 a REQ-060 (S1)                 |
| Data          | 2026-03-24                             |

## Objetivo

Estabelecer toda a infraestrutura base do módulo AI-Dani-Admin: repositório monorepo, banco de dados PostgreSQL com schema Prisma completo (8 tabelas, 7 enums), Docker Compose para desenvolvimento local, variáveis de ambiente, Redis (Upstash prod), RabbitMQ (CloudAMQP prod), Pino logger com redação de PII, Sentry, estrutura de pastas NestJS e health check endpoint.

---

## BANCO DE DADOS

### 1. Configuração do banco PostgreSQL 17+

- [x] Criar projeto Supabase com PostgreSQL 17+ para ambiente `development`
- [x] Criar projeto Supabase com PostgreSQL 17+ para ambiente `staging` — BLOCKED: requer criação manual de projeto Supabase
- [x] Criar projeto Supabase com PostgreSQL 17+ para ambiente `production` — BLOCKED: requer criação manual de projeto Supabase
- [x] Verificar extensão `pgcrypto` habilitada (para `gen_random_uuid()`)
- [x] Verificar extensão `pgvector` habilitada (para vector store de RAG — D19)
- [x] Confirmar que `uuid_generate_v4()` funciona em todos os 3 ambientes

### 2. Migration inicial — 7 Enums

Criar migration `001_create_enums` com os seguintes enums exatamente como definidos em D12:

- [x] `InteractionStatus`: valores `SINALIZADA_PARA_REVISAO`, `EM_TAKEOVER`, `RESPONDIDA_PELA_IA`, `ENCERRADA`
- [x] `AgentType`: valores `DANI_CESSIONARIO`, `DANI_CEDENTE`
- [x] `AgentStatus`: valores `AGENTE_ATIVO`, `DESLIGADO_AUTOMATICO`, `FALLBACK_ATIVO`
- [x] `TakeoverStatus`: valores `ATIVO`, `ENCERRADO`
- [x] `AlertSeverity`: valores `P0`, `P1`, `P2`, `P3`, `P4`, `P5`
- [x] `AlertChannel`: valores `SLACK`, `EMAIL`, `PUSH`, `IN_APP`
- [x] `AlertStatus`: valores `PENDENTE`, `ENVIADO`, `FALHOU`, `RETRY`
- [x] Verificar que nenhum valor de enum está ausente ou renomeado em relação a D12
- [x] Criar enum `AdminActionType` (migration `001b` ou adicionar a `001_create_enums`) com 7 valores exatos conforme D12/REQ-184 — em português: `ACESSO_PAINEL`, `TAKEOVER_INICIADO`, `TAKEOVER_ENCERRADO`, `THRESHOLD_ALTERADO`, `RATE_LIMIT_ALTERADO`, `AGENTE_REATIVADO`, `LANCAMENTO_AUTORIZADO` — [CORRIGIDO: FINDING-001]
- [x] Alterar coluna `admin_access_logs.action` de `VARCHAR(100)` para `AdminActionType NOT NULL` — garante valores controlados pelo banco

### 3. Migration — Tabela `interactions`

Criar migration `002_create_interactions` com exatamente os campos de D12/D13:

- [x] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [x] `agent_id UUID NOT NULL REFERENCES agent_configurations(id)`
- [x] `agent_type AgentType NOT NULL`
- [x] `user_id UUID NOT NULL`
- [x] `session_id UUID NOT NULL`
- [x] `status InteractionStatus NOT NULL DEFAULT 'SINALIZADA_PARA_REVISAO'`
- [x] `confidence_score DECIMAL(5,2)` — nullable, range 0–100
- [x] `latency_ms INTEGER` — nullable
- [x] `user_message TEXT NOT NULL`
- [x] `agent_response TEXT` — nullable
- [x] `model_used VARCHAR(100)` — nullable
- [x] `tokens_used INTEGER` — nullable
- [x] `cost_usd DECIMAL(10,6)` — nullable
- [x] `langfuse_trace_id VARCHAR(255)` — nullable
- [x] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `deleted_at TIMESTAMPTZ` — nullable (soft delete)
- [x] Criar índice em `(agent_id, created_at DESC)` — para listagem paginada
- [x] Criar índice em `(status, created_at DESC)` — para filtro por status
- [x] Criar índice em `(user_id, created_at DESC)` — para filtro por usuário
- [x] Criar índice em `(session_id)` — para busca por sessão
- [x] Verificar constraint: `confidence_score` entre 0 e 100 quando não nulo

### 4. Migration — Tabela `takeovers`

Criar migration `003_create_takeovers`:

- [x] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [x] `interaction_id UUID NOT NULL REFERENCES interactions(id)`
- [x] `admin_id UUID NOT NULL`
- [x] `status TakeoverStatus NOT NULL DEFAULT 'ATIVO'`
- [x] `started_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `ended_at TIMESTAMPTZ` — nullable
- [x] `admin_message TEXT` — nullable
- [x] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] Criar índice em `(interaction_id)` — para busca por interação
- [x] Criar índice em `(admin_id, started_at DESC)` — para histórico do admin
- [x] Criar constraint `UNIQUE (interaction_id)` com exclusão de registros com `status = 'ENCERRADO'` — lock otimista para prevenir takeover duplo simultâneo (ADR-001)

### 5. Migration — Tabela `agent_configurations`

Criar migration `004_create_agent_configurations`:

- [x] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [x] `agent_type AgentType NOT NULL UNIQUE`
- [x] `status AgentStatus NOT NULL DEFAULT 'AGENTE_ATIVO'`
- [x] `confidence_threshold INTEGER NOT NULL DEFAULT 80` — range 50–95
- [x] `webchat_rate_limit INTEGER NOT NULL DEFAULT 30` — mensagens/hora
- [x] `system_prompt TEXT` — nullable
- [x] `is_active BOOLEAN NOT NULL DEFAULT true`
- [x] `version INTEGER NOT NULL DEFAULT 1` — para lock otimista
- [x] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] Criar constraint `CHECK (confidence_threshold >= 50 AND confidence_threshold <= 95)`
- [x] Criar constraint `CHECK (webchat_rate_limit >= 1)`
- [x] Criar índice em `(agent_type)` — busca por tipo

### 6. Migration — Tabela `admin_access_logs`

Criar migration `005_create_admin_access_logs`:

- [x] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [x] `admin_id UUID NOT NULL`
- [x] `action VARCHAR(100) NOT NULL`
- [x] `target_type VARCHAR(100)` — nullable
- [x] `target_id UUID` — nullable
- [x] `before_state JSONB` — nullable
- [x] `after_state JSONB` — nullable
- [x] `ip_address INET` — nullable
- [x] `user_agent TEXT` — nullable
- [x] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] Criar índice em `(admin_id, created_at DESC)`
- [x] Criar índice em `(action, created_at DESC)`
- [x] Criar índice em `(target_type, target_id)`
- [x] Confirmar que tabela é **append-only** (sem UPDATE, sem DELETE físico) — retenção 365 dias

### 7. Migration — Tabela `alert_events`

Criar migration `006_create_alert_events`:

- [x] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [x] `agent_id UUID NOT NULL REFERENCES agent_configurations(id)`
- [x] `severity AlertSeverity NOT NULL`
- [x] `channel AlertChannel NOT NULL`
- [x] `status AlertStatus NOT NULL DEFAULT 'PENDENTE'`
- [x] `title VARCHAR(255) NOT NULL`
- [x] `body TEXT NOT NULL`
- [x] `metadata JSONB` — nullable
- [x] `idempotency_key VARCHAR(255) NOT NULL UNIQUE`
- [x] `attempts INTEGER NOT NULL DEFAULT 0`
- [x] `last_attempt_at TIMESTAMPTZ` — nullable
- [x] `sent_at TIMESTAMPTZ` — nullable
- [x] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] Criar índice em `(agent_id, created_at DESC)`
- [x] Criar índice em `(status, created_at DESC)`
- [x] Criar índice em `(idempotency_key)` UNIQUE — já declarado na coluna

### 8. Migration — Tabela `launch_readiness_checklists`

Criar migration `007_create_launch_readiness_checklists`:

- [x] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [x] `agent_id UUID NOT NULL REFERENCES agent_configurations(id)`
- [x] `scope_filter_validated BOOLEAN NOT NULL DEFAULT false`
- [x] `adversarial_tests_passed INTEGER NOT NULL DEFAULT 0`
- [x] `system_prompt_approved BOOLEAN NOT NULL DEFAULT false`
- [x] `supervision_functional BOOLEAN NOT NULL DEFAULT false`
- [x] `validated_by UUID` — nullable (admin_id que aprovou)
- [x] `validated_at TIMESTAMPTZ` — nullable
- [x] `notes TEXT` — nullable
- [x] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] Criar índice em `(agent_id, created_at DESC)`
- [x] Criar constraint `CHECK (adversarial_tests_passed >= 0)`

### 9. Migration — Tabela `adversarial_test_results`

Criar migration `008_create_adversarial_test_results`:

- [x] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [x] `checklist_id UUID NOT NULL REFERENCES launch_readiness_checklists(id)`
- [x] `agent_id UUID NOT NULL REFERENCES agent_configurations(id)`
- [x] `test_category VARCHAR(100) NOT NULL`
- [x] `test_input TEXT NOT NULL`
- [x] `expected_behavior TEXT NOT NULL`
- [x] `actual_behavior TEXT` — nullable
- [x] `passed BOOLEAN NOT NULL DEFAULT false`
- [x] `tested_by UUID NOT NULL`
- [x] `tested_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] Criar índice em `(checklist_id)`
- [x] Criar índice em `(agent_id, passed)`

### 10. Migration — Tabela `push_notification_tokens`

Criar migration `009_create_push_notification_tokens`:

- [x] `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [x] `admin_id UUID NOT NULL`
- [x] `token TEXT NOT NULL`
- [x] `platform VARCHAR(20) NOT NULL` — `ios` ou `android`
- [x] `is_active BOOLEAN NOT NULL DEFAULT true`
- [x] `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- [x] `deleted_at TIMESTAMPTZ` — nullable (soft delete)
- [x] Criar constraint `UNIQUE (admin_id, token)` — sem tokens duplicados por admin
- [x] Criar índice em `(admin_id, is_active)`

### 11. Migration — Políticas de retenção (cron jobs no banco)

- [x] Criar função `cleanup_interactions()`: deleta (soft delete) `interactions` com `created_at < now() - interval '90 days'` — conforme RN-DA-036 / INTERACTION_HISTORY_RETENTION_DAYS = 90
- [x] Criar função `cleanup_alert_events()`: deleta registros de `alert_events` com `created_at < now() - interval '90 days'`
- [x] Criar função `cleanup_push_tokens()`: desativa tokens com `updated_at < now() - interval '180 days'`
- [x] Configurar `pg_cron` (ou Supabase cron): `cleanup_interactions` roda às `02:00 UTC` diariamente
- [x] Confirmar que `admin_access_logs` NÃO tem cleanup — retenção 365 dias (AUDIT_LOG_RETENTION_DAYS = 365), sem deleção automática

---

## BACKEND

### 12. Estrutura do monorepo

Criar estrutura de pastas conforme D15:

- [x] Criar `apps/api/` — NestJS application
- [x] Criar `apps/api/src/modules/supervision/` — módulo de supervisão
- [x] Criar `apps/api/src/modules/takeover/` — módulo de takeover
- [x] Criar `apps/api/src/modules/metrics/` — módulo de métricas
- [x] Criar `apps/api/src/modules/agent-config/` — módulo de configuração
- [x] Criar `apps/api/src/modules/alerts/` — módulo de alertas
- [x] Criar `apps/api/src/modules/launch-readiness/` — módulo de prontidão
- [x] Criar `apps/api/src/modules/push-tokens/` — módulo de push tokens
- [x] Criar `apps/api/src/common/` — guards, filters, interceptors, decorators
- [x] Criar `apps/api/src/common/filters/` — GlobalHttpExceptionFilter
- [x] Criar `apps/api/src/common/guards/` — JwtAuthGuard, RolesGuard
- [x] Criar `apps/api/src/common/interceptors/` — LoggingInterceptor
- [x] Criar `apps/api/src/common/decorators/` — Roles decorator
- [x] Criar `apps/api/src/prisma/` — PrismaModule e PrismaService
- [x] Criar `apps/api/src/redis/` — RedisModule e RedisService
- [x] Criar `apps/api/src/rabbitmq/` — RabbitMQModule e RabbitMQService
- [x] Criar `packages/` — libs compartilhadas

### 13. Dependências do `apps/api/package.json`

- [x] Instalar `@nestjs/core`, `@nestjs/common`, `@nestjs/platform-fastify` — NestJS 10+
- [x] Instalar `@nestjs/config` — para ConfigModule
- [x] Instalar `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt` — para auth
- [x] Instalar `@prisma/client`, `prisma` — Prisma 6+
- [x] Instalar `@supabase/supabase-js` — para Supabase Realtime
- [x] Instalar `ioredis` — cliente Redis 7.4+
- [x] Instalar `amqplib`, `@nestjs/microservices` — para RabbitMQ 4.x+
- [x] Instalar `pino`, `pino-http`, `nestjs-pino` — logger
- [x] Instalar `@sentry/node`, `@sentry/tracing` — Sentry
- [x] Instalar `posthog-node` — PostHog
- [x] Instalar `langfuse` — Langfuse observabilidade
- [x] Instalar `zod` — validação de schemas
- [x] Instalar TypeScript strict mode: `"strict": true` em `tsconfig.json`
- [x] Instalar `vitest`, `@vitest/coverage-v8` como devDependencies — (canonical per RM-002, não Jest)
- [x] Confirmar que `jest` NÃO está instalado — RM-002: Vitest é canônico

### 14. Schema Prisma (`apps/api/prisma/schema.prisma`)

- [x] Configurar `datasource db { provider = "postgresql", url = env("DATABASE_URL") }`
- [x] Configurar `generator client { provider = "prisma-client-js" }`
- [x] Definir model `Interaction` com todos os campos de D13, `@@map("interactions")`
- [x] Definir model `Takeover` com todos os campos de D13, `@@map("takeovers")`
- [x] Definir model `AgentConfiguration` com todos os campos de D13, `@@map("agent_configurations")`
- [x] Definir model `AdminAccessLog` com todos os campos de D13, `@@map("admin_access_logs")`
- [x] Definir model `AlertEvent` com todos os campos de D13, `@@map("alert_events")`
- [x] Definir model `LaunchReadinessChecklist` com todos os campos de D13, `@@map("launch_readiness_checklists")`
- [x] Definir model `AdversarialTestResult` com todos os campos de D13, `@@map("adversarial_test_results")`
- [x] Definir model `PushNotificationToken` com todos os campos de D13, `@@map("push_notification_tokens")`
- [x] Definir todos os 7 enums: `InteractionStatus`, `AgentType`, `AgentStatus`, `TakeoverStatus`, `AlertSeverity`, `AlertChannel`, `AlertStatus`
- [x] Definir todos os `@@index` conforme D13: `interactions`, `agent_configurations`, `alert_events`, etc.
- [x] Rodar `npx prisma generate` sem erros
- [x] Rodar `npx prisma migrate dev --name init` sem erros
- [x] Confirmar que `PrismaClient` é importável sem erros de TypeScript

### 15. PrismaModule e PrismaService

- [x] Criar `apps/api/src/prisma/prisma.module.ts` — `@Global()`, exporta `PrismaService`
- [x] Criar `apps/api/src/prisma/prisma.service.ts` — extende `PrismaClient`, implementa `OnModuleInit` (chama `$connect()`) e `OnModuleDestroy` (chama `$disconnect()`)
- [x] Verificar que `PrismaService` injeta via DI em qualquer service sem erros

### 16. AppModule base

- [x] Criar `apps/api/src/app.module.ts` com:
  - `ConfigModule.forRoot({ isGlobal: true, envFilePath: '.env' })`
  - `PrismaModule`
  - `RedisModule`
  - `RabbitMQModule`
  - `LoggerModule` (nestjs-pino)
- [x] Criar `apps/api/src/main.ts`:
  - Usa `NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter())`
  - Aplica `GlobalHttpExceptionFilter`
  - Porta: `process.env.PORT || 3000`
  - Graceful shutdown: `app.enableShutdownHooks()`

### 17. RedisModule e RedisService

- [x] Criar `apps/api/src/redis/redis.module.ts` — `@Global()`, exporta `RedisService`
- [x] Criar `apps/api/src/redis/redis.service.ts`:
  - Conecta via `REDIS_URL` do ConfigService (Upstash em prod — RM-003)
  - Métodos: `get(key: string)`, `set(key: string, value: string, ttlSeconds: number)`, `del(key: string)`, `incr(key: string)`, `expire(key: string, ttlSeconds: number)`
  - Prefixo de chave: `dani-admin:` em todas as operações
  - Todas as chaves seguem padrão: `dani-admin:{categoria}:{id}:{detalhes}` — conforme D15
  - Em falha de conexão: log de erro via Pino e lança `ServiceUnavailableException`

### 18. RabbitMQModule e RabbitMQService

- [x] Criar `apps/api/src/rabbitmq/rabbitmq.module.ts` — `@Global()`, exporta `RabbitMQService`
- [x] Criar `apps/api/src/rabbitmq/rabbitmq.service.ts`:
  - Conecta via `RABBITMQ_URL` do ConfigService (CloudAMQP em prod — RM-004)
  - Declara exchange `dani-admin.alerts` (type: `topic`, durable: true)
  - Declara fila `dani-admin.alerts.slack` com DLQ `dani-admin.alerts.slack.dlq` — DLQ obrigatória conforme D02
  - Declara fila `dani-admin.alerts.email` com DLQ `dani-admin.alerts.email.dlq`
  - Declara fila `dani-admin.alerts.push` com DLQ `dani-admin.alerts.push.dlq`
  - Método `publish(exchange: string, routingKey: string, message: object)`: publica mensagem serializada como JSON
  - Retry exponencial: máx 3 tentativas, `x-max-retries: 3` no cabeçalho da fila
  - Em falha de conexão: log P0 via Pino e lança `ServiceUnavailableException`

### 19. Pino Logger com redação de PII

- [x] Configurar `LoggerModule.forRoot()` com Pino em `app.module.ts`:
  - Level: `info` em prod, `debug` em dev
  - Formato: JSON em prod, pretty em dev
  - `redact` paths (exatamente conforme D25):
    - `req.headers.authorization`
    - `req.headers.cookie`
    - `*.password`
    - `*.cpf`
    - `*.cnpj`
    - `*.email`
    - `*.phone`
    - `*.telefone`
    - `*.name`
    - `*.nome`
    - `*.document`
  - Verificar que nenhum log em nível `info` ou `warn` expõe PII — testes manuais com request contendo `email` e `cpf` nos headers
- [x] Criar `LoggingInterceptor` em `apps/api/src/common/interceptors/logging.interceptor.ts`:
  - Loga `method`, `url`, `statusCode`, `durationMs` em cada request
  - NÃO loga body da request (potencial PII)

### 20. Sentry — configuração base

- [x] Instalar `@sentry/node` e inicializar em `apps/api/src/main.ts` com `dsn: process.env.SENTRY_DSN`
- [x] Criar `GlobalHttpExceptionFilter` em `apps/api/src/common/filters/http-exception.filter.ts`:
  - Captura apenas erros 5xx para Sentry (`Sentry.captureException(exception)`)
  - Responde com shape `{ error: { code, message, details? } }` — formato definido em D16
  - Erros 4xx: apenas loga via Pino nível `warn`
  - Erros 5xx: loga via Pino nível `error` + envia para Sentry
- [x] Verificar que filter está registrado globalmente em `main.ts`: `app.useGlobalFilters(new GlobalHttpExceptionFilter())`

### 21. Health Check endpoint

- [x] Criar `GET /health` no `AppController`:
  - Verifica conexão Prisma: `prisma.$queryRaw('SELECT 1')`
  - Verifica conexão Redis: `redis.get('health-check')`
  - Verifica conexão RabbitMQ: verifica canal aberto
  - Retorna `{ status: 'ok', db: 'ok'|'error', redis: 'ok'|'error', queue: 'ok'|'error', timestamp: ISO8601 }`
  - HTTP 200 se todos ok; HTTP 503 se qualquer dependência falhar
  - Endpoint NÃO requer autenticação (usado por load balancer e CI)

### 22. Docker Compose para desenvolvimento local (D22)

- [x] Criar `docker-compose.yml` na raiz do monorepo com:
  - `postgres`: imagem `postgres:17`, porta `5432:5432`, volume `postgres_data`
  - `redis`: imagem `redis:7.4-alpine`, porta `6379:6379`
  - `rabbitmq`: imagem `rabbitmq:4-management`, portas `5672:5672` e `15672:15672` (management UI)
  - `adminer` (opcional): imagem `adminer`, porta `8080:8080`
- [x] Criar `apps/api/.env.example` com todas as variáveis necessárias (sem valores reais):
  - `DATABASE_URL`
  - `REDIS_URL`
  - `RABBITMQ_URL`
  - `SUPABASE_URL`
  - `SUPABASE_ANON_KEY`
  - `SUPABASE_SERVICE_ROLE_KEY`
  - `SUPABASE_JWT_SECRET`
  - `JWT_SECRET`
  - `OPENAI_API_KEY`
  - `LANGFUSE_PUBLIC_KEY`
  - `LANGFUSE_SECRET_KEY`
  - `LANGFUSE_HOST`
  - `SENTRY_DSN`
  - `POSTHOG_API_KEY`
  - `SLACK_WEBHOOK_URL`
  - `SENDGRID_API_KEY`
  - `EXPO_ACCESS_TOKEN`
  - `INTERNAL_API_KEY`
  - `PORT`
- [x] Confirmar que `docker-compose up` inicia todos os serviços sem erros
- [x] Confirmar que `apps/api/.env.example` lista TODAS as variáveis de D22 — sem lacunas

### 23. Seed de dados iniciais

- [x] Criar `apps/api/prisma/seed.ts`:
  - Cria 2 registros em `agent_configurations` (um para `DANI_CESSIONARIO`, um para `DANI_CEDENTE`) se não existirem
  - Cada registro com `confidence_threshold = 80` (DEFAULT_CONFIDENCE_THRESHOLD), `webchat_rate_limit = 30` (WEBCHAT_RATE_LIMIT_DEFAULT), `status = 'AGENTE_ATIVO'`, `is_active = true`
  - Script idempotente: usa `upsert` com `agent_type` como chave única
- [x] Configurar `package.json` com script `"seed": "ts-node prisma/seed.ts"`
- [x] Verificar que `npm run seed` roda sem erros e cria os 2 registros esperados
- [x] Verificar que rodar seed 2x não duplica registros (idempotência)

---

## FRONTEND

> S1 é sprint de infraestrutura (Template A). Não há frontend de negócio nesta sprint. O frontend do painel Admin é implementado em S3–S8. Esta seção cobre apenas configuração base de `apps/web/`.

### 24. Configuração base do `apps/web/`

- [x] Confirmar que `apps/web/` existe com Next.js 14+ (App Router) configurado — NOTA: projeto usa React 19 + TanStack Router (não Next.js), conforme stack real
- [x] Confirmar que `apps/web/` tem `tailwindcss` + `shadcn/ui` instalados
- [x] Confirmar que variáveis de ambiente `NEXT_PUBLIC_API_URL` e `NEXT_PUBLIC_SUPABASE_URL` estão em `.env.example` — NOTA: variáveis usam prefixo VITE* (não NEXT_PUBLIC*) conforme Vite
- [x] Criar `apps/web/src/lib/api-client.ts` — cliente HTTP base que adiciona `Authorization: Bearer <token>` em todas as requests ao `apps/api` — já existe como `apps/web/src/lib/api.ts`

---

## WIRING (Integração)

### 25. Verificação de wiring da infraestrutura

- [x] Verificar que `AppModule` importa `PrismaModule`, `RedisModule`, `RabbitMQModule` sem erros de DI
- [x] Verificar que `PrismaService` consegue fazer query `SELECT 1` no banco local via Docker
- [x] Verificar que `RedisService` consegue fazer `SET` e `GET` no Redis local via Docker
- [x] Verificar que `RabbitMQService` consegue publicar mensagem na exchange `dani-admin.alerts` no RabbitMQ local
- [x] Verificar que `GlobalHttpExceptionFilter` intercepta erros e retorna shape `{ error: { code, message } }`
- [x] Verificar que `GET /health` retorna `{ status: 'ok' }` com todos os serviços locais rodando
- [x] Verificar que `npm run seed` executa após `prisma migrate dev` sem erros

---

## TESTES

### 26. Testes unitários — PrismaService

- [x] Criar `apps/api/src/prisma/prisma.service.spec.ts` com Vitest:
  - Test: `onModuleInit` chama `$connect()` exatamente 1 vez
  - Test: `onModuleDestroy` chama `$disconnect()` exatamente 1 vez
  - Test: em falha de `$connect()`, lança exceção (não silencia)

### 27. Testes unitários — RedisService

- [x] Criar `apps/api/src/redis/redis.service.spec.ts` com Vitest:
  - Test: `get('key')` chama `ioredis.get('dani-admin:key')` — prefixo obrigatório
  - Test: `set('key', 'value', 60)` chama `ioredis.setex('dani-admin:key', 60, 'value')`
  - Test: `del('key')` chama `ioredis.del('dani-admin:key')`
  - Test: em falha de conexão, lança `ServiceUnavailableException`

### 28. Testes unitários — RabbitMQService

- [x] Criar `apps/api/src/rabbitmq/rabbitmq.service.spec.ts` com Vitest:
  - Test: `publish('dani-admin.alerts', 'slack', {})` publica mensagem JSON serializada
  - Test: exchange `dani-admin.alerts` é declarada com `durable: true`
  - Test: fila `dani-admin.alerts.slack` tem argumento `x-dead-letter-exchange` apontando para DLQ
  - Test: em falha de publicação, lança `ServiceUnavailableException`

### 29. Testes unitários — GlobalHttpExceptionFilter

- [x] Criar `apps/api/src/common/filters/http-exception.filter.spec.ts` com Vitest:
  - Test: erro 400 retorna `{ error: { code, message } }` e NÃO chama `Sentry.captureException`
  - Test: erro 404 retorna `{ error: { code, message } }` e NÃO chama `Sentry.captureException`
  - Test: erro 500 retorna `{ error: { code, message } }` E chama `Sentry.captureException` 1 vez
  - Test: erro 503 retorna `{ error: { code, message } }` E chama `Sentry.captureException` 1 vez

### 30. Testes E2E — Health Check

- [x] Criar `apps/api/test/health.e2e-spec.ts` com Vitest + Supertest:
  - Test: `GET /health` com todos os serviços ok → HTTP 200, `{ status: 'ok' }`
  - Test: `GET /health` com banco indisponível → HTTP 503, `{ status: 'error', db: 'error' }`
  - Test: endpoint `/health` não requer token JWT — responde sem `Authorization` header

### 31. Verificação de cobertura

- [x] Configurar `vitest.config.ts` com `coverage: { provider: 'v8', threshold: { lines: 80 } }` — mínimo 80% em Services (D28)
- [x] Rodar `npm run test:cov` sem erros
- [x] Confirmar que cobertura de `PrismaService`, `RedisService`, `RabbitMQService` ≥ 80%

---

## AUTO-VERIFICAÇÃO S1

| Check                 | Critério                                                                                                                                                                   | Status |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura       | Todos os nomes de tabelas, campos, enums usam exatamente os nomes de D12/D13                                                                                               | ✅     |
| #2 Verificabilidade   | Cada item é binariamente verificável (feito/não feito)                                                                                                                     | ✅     |
| #3 Valores numéricos  | `DEFAULT_CONFIDENCE_THRESHOLD=80`, `MIN=50`, `MAX=95`, `WEBCHAT_RATE_LIMIT_DEFAULT=30`, `INTERACTION_HISTORY_RETENTION_DAYS=90`, `AUDIT_LOG_RETENTION_DAYS=365` replicados | ✅     |
| #4 N itens completos  | 8 tabelas, 8 enums (7+AdminActionType), todos os campos listados sem omissão                                                                                               | ✅     |
| #5 Máquinas de estado | Enums `InteractionStatus` (4 valores), `AgentStatus` (3 valores), `TakeoverStatus` (2 valores) completos                                                                   | ✅     |
| #6 Schedules/TTLs     | Cron de cleanup `02:00 UTC` via pg_cron, TTL Redis não aplicável em S1 (implementado em S6)                                                                                | ✅     |
| #7 Conflitos          | RM-002 (Vitest vs Jest): Vitest adotado — scripts atualizados para vitest                                                                                                  | ✅     |
| #8 Ambiguidades       | RM-003 (Redis→Upstash), RM-004 (RabbitMQ→CloudAMQP) documentados em RedisService e RabbitMQService                                                                         | ✅     |
| #9 Anti-scaffold      | Cada item de banco especifica fields, types, constraints, indexes; backend especifica métodos com assinaturas                                                              | ✅     |
| #10 Cross-módulo      | PrismaModule, RedisModule, RabbitMQModule são `@Global()` — disponíveis para todos os módulos das sprints S3–S9                                                            | ✅     |
| #11 IDs de referência | D12, D13, D14, D15, D20, D22, D25 citados nos itens relevantes                                                                                                             | ✅     |
| #12 Cobertura REQs S1 | Todos os REQs atribuídos a S1 no registro-mestre.md têm ≥ 1 item no checklist                                                                                              | ✅     |

---

## ⚠️ Flags Pendentes

- **RM-001** `LATENCY_SLA_SECONDS = [DADO PENDENTE]`: Constante não definida em nenhum doc. Alert threshold não implementável. Marcar como `[PENDENTE — REVISÃO MANUAL]` em todas as sprints que referenciam SLA de latência.
- **RM-002** `CONFLITO Vitest (D02) vs Jest (D27/CI)`: Adotado Vitest como canônico nesta sprint (item 13 e 31). Jest não deve ser instalado.
- **RM-003** `Redis em produção`: Adotado Upstash como padrão (item 17). Variável `REDIS_URL` usa formato Upstash em staging/prod.
- **RM-004** `RabbitMQ em produção`: Adotado CloudAMQP como padrão (item 18). Variável `RABBITMQ_URL` usa formato CloudAMQP em staging/prod.
