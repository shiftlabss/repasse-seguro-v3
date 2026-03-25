# S1 — Fundação

<!-- Atualizado em 2026-03-24 pela A03 — 2 correções aplicadas (FINDING-007, FINDING-026) -->

| Campo              | Valor                                                                                                                       |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S1                                                                                                                          |
| **Nome**           | Fundação                                                                                                                    |
| **Template**       | A — Infraestrutura                                                                                                          |
| **Docs Fonte**     | 02-Stacks, 12-ModeloDados, 13-SchemaPrisma, 14-EspecificaçõesTécnicas, 17-IntegraçõesExternas, 22-AmbienteSetupLocalSecrets |
| **REQs cobertos**  | REQ-007, REQ-114 a REQ-158, REQ-168 a REQ-173, REQ-178                                                                      |
| **Total de itens** | 42                                                                                                                          |
| **Status**         | ✅ Concluída                                                                                                                |

---

## 🎯 Objetivo

Criar a infraestrutura completa do serviço `apps/ai/` no monorepo: estrutura de pastas, todas as migrations + seed do banco, configuração de variáveis de ambiente, integrações base (OpenAI, Redis, RabbitMQ, Langfuse, Sentry), error handling global, logger Pino, health check, e pipeline CI/CD. Nenhum código de negócio — apenas fundação verificável.

---

## 📦 BANCO DE DADOS — Migrations e Schema

### 1. Migration 001 — Tabela `chat_conversations`

- [x] Criar migration `001_create_chat_conversations` com colunas: `id` UUID PK DEFAULT gen_random_uuid(), `cessionario_id` UUID NOT NULL, `title` TEXT, `status` `conversation_status` NOT NULL DEFAULT 'active', `context` JSONB DEFAULT '{}', `metadata` JSONB DEFAULT '{}', `created_at` TIMESTAMPTZ DEFAULT now(), `updated_at` TIMESTAMPTZ DEFAULT now(), `deleted_at` TIMESTAMPTZ NULL
- [x] Criar enum `conversation_status` com valores: `active`, `archived`, `deleted`
- [x] Criar índices: `idx_chat_conversations_cessionario_id`, `idx_chat_conversations_status`, `idx_chat_conversations_deleted_at`
- [x] Validar que `deleted_at` NULL = soft delete padrão; registro NUNCA é deletado fisicamente
- [x] Teste: INSERT + SELECT + soft delete via UPDATE `deleted_at = now()` + confirmar que registro persiste

### 2. Migration 002 — Tabela `chat_messages`

- [x] Criar migration `002_create_chat_messages` com colunas: `id` UUID PK DEFAULT gen_random_uuid(), `conversation_id` UUID NOT NULL REFERENCES `chat_conversations(id)`, `role` `message_role` NOT NULL, `content` TEXT NOT NULL, `metadata` JSONB DEFAULT '{}', `tokens_used` INTEGER, `created_at` TIMESTAMPTZ DEFAULT now(), `deleted_at` TIMESTAMPTZ NULL
- [x] Criar enum `message_role` com valores: `user`, `assistant`, `system`, `tool`
- [x] **Sem coluna `updated_at`** — tabela append-only (imutável após criação)
- [x] Criar índices: `idx_chat_messages_conversation_id`, `idx_chat_messages_role`, `idx_chat_messages_created_at`
- [x] Teste: INSERT message → confirmar ausência de `updated_at` no schema; tentar UPDATE → deve ser bloqueado por política ou documentado como proibido

### 3. Migration 003 — Tabela `ai_interactions`

- [x] Criar migration `003_create_ai_interactions` com colunas: `id` UUID PK, `conversation_id` UUID REFERENCES `chat_conversations(id)`, `message_id` UUID REFERENCES `chat_messages(id)`, `model` TEXT NOT NULL, `prompt_tokens` INTEGER, `completion_tokens` INTEGER, `total_tokens` INTEGER, `latency_ms` INTEGER, `confidence_score` DECIMAL(5,4), `tool_calls` JSONB DEFAULT '[]', `cache_hit` BOOLEAN DEFAULT false, `created_at` TIMESTAMPTZ DEFAULT now(), `deleted_at` TIMESTAMPTZ NULL
- [x] Criar índices: `idx_ai_interactions_conversation_id`, `idx_ai_interactions_created_at`, `idx_ai_interactions_cache_hit`
- [x] Teste: INSERT interaction → SELECT com JOIN em `ai_interactions` + `chat_messages` → confirmar integridade referencial

### 4. Migration 004 — Tabela `llm_cache_entries`

- [x] Criar migration `004_create_llm_cache_entries` com colunas: `id` UUID PK, `cache_type` `cache_type` NOT NULL, `cache_key` TEXT NOT NULL UNIQUE, `prompt_hash` TEXT NOT NULL, `response` TEXT NOT NULL, `model` TEXT NOT NULL, `tokens_used` INTEGER, `hit_count` INTEGER DEFAULT 0, `embedding` vector(1536), `created_at` TIMESTAMPTZ DEFAULT now(), `expires_at` TIMESTAMPTZ NOT NULL
- [x] Criar enum `cache_type` com valores: `exact`, `semantic`
- [x] **Sem coluna `deleted_at`** — não tem soft delete
- [x] Criar índice IVFFlat via raw SQL: `CREATE INDEX ON llm_cache_entries USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100)` — executar via Prisma `$executeRawUnsafe`
- [x] Criar índices: `idx_llm_cache_entries_cache_key`, `idx_llm_cache_entries_expires_at`
- [x] Teste: INSERT exact cache + semantic cache → confirmar TTL `expires_at` correto; query por `cache_key` → hit único

### 5. Migration 005 — Tabela `document_embeddings`

- [x] Criar migration `005_create_document_embeddings` com colunas: `id` UUID PK, `content` TEXT NOT NULL, `embedding` `vector(1536)` NOT NULL (via `Unsupported("vector(1536)")`), `metadata` JSONB DEFAULT '{}', `source` TEXT, `chunk_index` INTEGER, `created_at` TIMESTAMPTZ DEFAULT now()
- [x] **Sem FK para usuários** (isolamento por instrução do agente)
- [x] **Sem soft delete** (`deleted_at` não existe nesta tabela)
- [x] Criar índice IVFFlat: `CREATE INDEX ON document_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100)`
- [x] Teste: INSERT embedding com vetor 1536 dims → similarity search via `<=>` → confirmar retorno ordenado por distância coseno

### 6. Migration 006 — Tabela `whatsapp_bindings`

- [x] Criar migration `006_create_whatsapp_bindings` com colunas: `id` UUID PK, `cessionario_id` UUID NOT NULL UNIQUE, `phone_number` TEXT NOT NULL, `verified_at` TIMESTAMPTZ, `last_verified_at` TIMESTAMPTZ, `status` `whatsapp_binding_status` NOT NULL DEFAULT 'pending', `created_at` TIMESTAMPTZ DEFAULT now(), `updated_at` TIMESTAMPTZ DEFAULT now(), `deleted_at` TIMESTAMPTZ NULL
- [x] Criar enum `whatsapp_binding_status` com valores: `pending`, `active`, `inactive`, `blocked`
- [x] Criar índices: `idx_whatsapp_bindings_cessionario_id`, `idx_whatsapp_bindings_phone_number`, `idx_whatsapp_bindings_status`
- [x] Teste: INSERT binding → soft delete → confirmar que `cessionario_id` UNIQUE constraint funciona

### 7. Migration 007 — Tabela `otp_attempts`

- [x] Criar migration `007_create_otp_attempts` com colunas: `id` UUID PK, `whatsapp_binding_id` UUID REFERENCES `whatsapp_bindings(id)`, `phone_number` TEXT NOT NULL, `otp_hash` TEXT NOT NULL, `channel` `otp_channel` NOT NULL, `status` `otp_status` NOT NULL DEFAULT 'pending', `expires_at` TIMESTAMPTZ NOT NULL, `verified_at` TIMESTAMPTZ, `created_at` TIMESTAMPTZ DEFAULT now()
- [x] Criar enum `otp_channel` com valores: `sms`, `whatsapp`
- [x] Criar enum `otp_status` com valores: `pending`, `verified`, `expired`, `failed`
- [x] **Sem `updated_at`** — tabela append-only
- [x] Criar índices: `idx_otp_attempts_phone_number`, `idx_otp_attempts_status`, `idx_otp_attempts_expires_at`
- [x] Teste: INSERT OTP → confirmar ausência de `updated_at`; checar constraint `expires_at` no futuro

### 8. Migration 008 — Tabelas `notification_preferences` e `notification_events`

- [x] Criar migration `008_create_notification_tables`:
  - `notification_preferences`: `id` UUID PK, `cessionario_id` UUID NOT NULL UNIQUE, `new_opportunities` BOOLEAN DEFAULT true, `deal_status_changes` BOOLEAN DEFAULT true, `market_alerts` BOOLEAN DEFAULT true, `channel` `notification_channel` NOT NULL DEFAULT 'webchat', `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
  - `notification_events`: `id` UUID PK, `cessionario_id` UUID NOT NULL, `type` `notification_type` NOT NULL, `title` TEXT NOT NULL, `body` TEXT NOT NULL, `metadata` JSONB DEFAULT '{}', `sent_at` TIMESTAMPTZ, `read_at` TIMESTAMPTZ, `created_at` TIMESTAMPTZ
- [x] Criar enums: `notification_channel` (`webchat`, `whatsapp`, `both`), `notification_type` (`new_opportunity`, `deal_status_change`, `market_alert`)
- [x] Teste: INSERT preference + INSERT event → confirmar relação por `cessionario_id`

### 9. Migration 009 — Tabelas `ai_takeovers` e `agent_configurations`

- [x] Criar migration `009_create_admin_tables`:
  - `ai_takeovers`: `id` UUID PK, `conversation_id` UUID REFERENCES `chat_conversations(id)`, `admin_id` UUID NOT NULL, `reason` `takeover_reason` NOT NULL, `confidence_at_takeover` DECIMAL(5,4), `notes` TEXT NULL, `started_at` TIMESTAMPTZ DEFAULT now(), `ended_at` TIMESTAMPTZ, `created_at` TIMESTAMPTZ DEFAULT now() — [CORRIGIDO: FINDING-007 — adicionado campo `notes TEXT NULL`]
  - `agent_configurations`: `id` UUID PK, `key` TEXT NOT NULL UNIQUE, `value` TEXT NOT NULL, `description` TEXT, `updated_by` UUID, `created_at` TIMESTAMPTZ, `updated_at` TIMESTAMPTZ
- [x] Criar enum `takeover_reason` com valores: `low_confidence`, `admin_manual`, `user_request`, `error`
- [x] Criar índice em `ai_takeovers.conversation_id` e `ai_takeovers.started_at`
- [x] Teste: INSERT takeover com `confidence_at_takeover` fora do range 0-1 → rejeitar; INSERT agent_configuration com chave duplicada → rejeitar com UNIQUE violation

### 10. Migration 010 — Tabela `agent_status_logs` e `lgpd_consents`

- [x] Criar migration `010_create_supplementary_tables`:
  - `agent_status_logs`: `id` UUID PK, `status` `agent_status` NOT NULL, `reason` TEXT, `triggered_by` UUID, `created_at` TIMESTAMPTZ DEFAULT now() — **sem `updated_at`**, append-only
  - `lgpd_consents`: `id` UUID PK, `cessionario_id` UUID NOT NULL, `consent_type` `lgpd_consent_type` NOT NULL, `granted` BOOLEAN NOT NULL, `ip_address` INET, `user_agent` TEXT, `created_at` TIMESTAMPTZ DEFAULT now()
- [x] Criar enums: `agent_status` (`online`, `offline`, `maintenance`, `degraded`), `lgpd_consent_type` (`data_processing`, `ai_analysis`, `whatsapp_contact`, `marketing`)
- [x] Teste: INSERT consent → confirmar imutabilidade (sem UPDATE); INSERT status log → confirmar append-only

### 11. Seed obrigatório — 9 `agent_configurations`

- [x] Criar seed com exatamente 9 registros obrigatórios em `agent_configurations`:
  1. `confidence_threshold` = `"0.80"` — limiar padrão de takeover
  2. `max_messages_per_session` = `"20"` — Redis TTL contexto
  3. `redis_session_ttl_minutes` = `"30"` — TTL sessão Redis
  4. `webchat_rate_limit_per_hour` = `"30"` — limite msgs webchat
  5. `whatsapp_rate_limit_per_hour` = `"20"` — limite msgs WhatsApp
  6. `message_retention_days` = `"90"` — retenção LGPD
  7. `similarity_threshold` = `"0.78"` — RAG threshold
  8. `semantic_cache_threshold` = `"0.92"` — semantic cache threshold
  9. `agent_status` = `"online"` — status inicial
- [x] Teste: executar seed → SELECT COUNT(\*) FROM agent_configurations → retornar 9; tentar inserir chave duplicada → rejeitar

---

## 🏗️ BACKEND — Estrutura, Config e Infraestrutura

### 12. Estrutura de pastas `apps/ai/`

- [x] Criar estrutura conforme Doc 02 (PG-02 — pasta primeiro):
  ```
  apps/ai/
  ├── src/
  │   ├── modules/
  │   │   ├── chat/
  │   │   ├── ai/
  │   │   ├── calculator/
  │   │   ├── notification/
  │   │   ├── supervision/
  │   │   └── whatsapp/
  │   ├── common/
  │   │   ├── guards/
  │   │   ├── interceptors/
  │   │   ├── filters/
  │   │   ├── pipes/
  │   │   └── decorators/
  │   ├── config/
  │   ├── prisma/
  │   └── main.ts
  ├── prisma/
  │   ├── schema.prisma
  │   ├── migrations/
  │   └── seed.ts
  ├── test/
  │   ├── unit/
  │   ├── integration/
  │   └── e2e/
  └── package.json
  ```
- [x] Cada módulo com arquivos: `*.module.ts`, `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `dto/`, `entities/`
- [x] Teste: `tsc --noEmit` sem erros; `pnpm lint` sem erros

### 13. `AppModule` + ConfigModule global

- [x] Configurar `ConfigModule.forRoot({ isGlobal: true, validate: (config) => validateEnvSchema(config) })` — validação com `class-validator` ao startup
- [x] 18 variáveis de ambiente mapeadas (doc 22): `DATABASE_URL`, `REDIS_URL`, `RABBITMQ_URL`, `OPENAI_API_KEY`, `OPENAI_MODEL`, `OPENAI_EMBEDDING_MODEL`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`, `SENTRY_DSN`, `JWT_SECRET`, `EVOLUTION_API_URL`, `EVOLUTION_API_KEY`, `FEATURE_WHATSAPP_MOCK`, `NODE_ENV`
- [x] 9 variáveis marcadas como sensíveis no env schema: `DATABASE_URL`, `OPENAI_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `JWT_SECRET`, `EVOLUTION_API_KEY`, `LANGFUSE_SECRET_KEY`, `SENTRY_DSN`, `REDIS_URL`, `RABBITMQ_URL`
- [x] `FEATURE_WHATSAPP_MOCK=true` por padrão em `.env.development`
- [x] Teste: startup com variável obrigatória ausente → `ConfigModule` lança `Error` descritivo antes de servir requests; startup com todas as variáveis presentes → servidor sobe sem erros

### 14. Logger Pino global

- [x] Instalar e configurar `nestjs-pino` 4.x+ com `pino-pretty` apenas em `NODE_ENV !== 'production'`
- [x] `console.log` proibido em produção — configurar regra ESLint `no-console: error` para arquivos em `src/`
- [x] Configurar `PinoLogger` com campos obrigatórios em cada log: `service: 'repasse-ai'`, `env: NODE_ENV`, `requestId` (correlation ID via `AsyncLocalStorage`)
- [x] Configurar `LoggerModule.forRoot` com `pinoHttp` para logar todas as requests HTTP (method, url, statusCode, responseTime, requestId)
- [x] Teste: request HTTP → log JSON contém campos `service`, `requestId`, `statusCode`, `responseTime`; `NODE_ENV=production` → output JSON puro (sem pino-pretty)

### 15. `PrismaModule` global + soft-delete middleware

- [x] Criar `PrismaModule` com `PrismaService extends PrismaClient` — exportado como global
- [x] Implementar middleware Prisma de soft delete: interceptar `findMany`, `findFirst`, `findUnique` → adicionar `WHERE deleted_at IS NULL` automaticamente em tabelas com soft delete (`chat_conversations`, `chat_messages`, `ai_interactions`, `whatsapp_bindings`)
- [x] Implementar middleware de `updated_at`: auto-set em `UPDATE` para tabelas com essa coluna
- [x] Bloquear `UPDATE` e `DELETE` em tabelas append-only (`chat_messages`, `otp_attempts`, `agent_status_logs`, `lgpd_consents`) via middleware — lançar `ForbiddenException` com mensagem `"Tabela append-only: operação não permitida"`
- [x] Teste: `prisma.chatMessages.update({...})` → lançar `ForbiddenException`; `prisma.chatConversations.findMany()` com `deleted_at` preenchido → não retornar o registro

### 16. `RedisModule` global

- [x] Instalar `ioredis` 5.x+ e criar `RedisModule` global com provider `REDIS_CLIENT`
- [x] Configurar conexão com `REDIS_URL` do env — Upstash em produção
- [x] Implementar `RedisHealthIndicator` para uso no health check
- [x] Padrão de chaves: `ai:cache:{tipo}:{hash}` — documentar no módulo
- [x] Teste: `ping` ao Redis → PONG; SET/GET/DEL básico; conexão com URL inválida → erro descritivo no startup

### 17. `RabbitMQModule` global

- [x] Instalar `@golevelup/nestjs-rabbitmq` e criar `RabbitMQModule` global
- [x] Configurar 4 exchanges conforme Doc 14: `ai.events` (topic), `ai.commands` (direct), `ai.notifications` (fanout), `ai.dlx` (direct — Dead Letter Exchange)
- [x] Criar 6 filas conforme Doc 14: `ai.analysis.queue`, `ai.stream.queue`, `ai.notification.queue`, `ai.whatsapp.queue`, `ai.metrics.queue`, `ai.dlq` (Dead Letter Queue)
- [x] Binding: cada fila de negócio → `x-dead-letter-exchange: ai.dlx` com `x-dead-letter-routing-key: dlq`
- [x] Teste: startup → confirmar 4 exchanges e 6 filas criados; publish em `ai.analysis.queue` → consume; mensagem com erro → roteada para `ai.dlq`

### 18. `HttpExceptionFilter` global

- [x] Criar `AllExceptionsFilter` implementando `ExceptionFilter` — capturar todos os erros não tratados
- [x] Formato de resposta de erro padronizado: `{ statusCode, message, error, requestId, timestamp }`
- [x] `HttpException` → status code original + message do erro
- [x] Erros não-HTTP (runtime errors) → `500 Internal Server Error` com `message: "Erro interno do servidor"` (sem vazar stack trace em produção)
- [x] Integração com Sentry: chamar `Sentry.captureException(exception)` para status >= 500
- [x] Teste unitário: mock `HttpException(403, "Forbidden")` → response `{ statusCode: 403, message: "Forbidden" }`; mock `new Error("crash")` em `NODE_ENV=production` → response `{ statusCode: 500, message: "Erro interno do servidor" }` sem stack trace

### 19. `JwtAuthGuard` global base

- [x] Instalar `@nestjs/passport` + `passport-jwt`
- [x] Criar `JwtStrategy` que valida Bearer token do header `Authorization`
- [x] Extrair payload: `{ sub: userId, role: 'CESSIONARIO' | 'ADMIN' | 'CEDENTE', organizationId }`
- [x] Criar guard `JwtAuthGuard` registrado globalmente — aplica em todos os endpoints exceto `@Public()` decorator
- [x] Criar decorator `@Public()` para marcar endpoints sem autenticação (health check, webhook EvolutionAPI)
- [x] Criar decorator `@Roles(...roles)` e `RolesGuard` — verificar `role` do payload JWT contra roles permitidas
- [x] Teste: request sem token → `401 Unauthorized`; token inválido → `401 Unauthorized`; token com role `CEDENTE` em endpoint `CESSIONARIO`-only → `403 Forbidden`; endpoint `@Public()` sem token → `200 OK`

### 20. Health Check endpoint

- [x] Instalar `@nestjs/terminus`
- [x] Criar `GET /health` (rota `@Public()`) com checks: `PrismaHealthIndicator` (DB), `RedisHealthIndicator`, `RabbitMQHealthIndicator`
- [x] Resposta healthy: `{ status: "ok", info: { database: { status: "up" }, redis: { status: "up" }, rabbitmq: { status: "up" } } }`
- [x] Resposta unhealthy: HTTP 503 com `{ status: "error", error: { [service]: { status: "down", message: "..." } } }`
- [x] Teste: todos os serviços up → `200 { status: "ok" }`; mock Redis down → `503 { status: "error", error: { redis: { status: "down" } } }`

### 21. Sentry global

- [x] Instalar `@sentry/nestjs` e inicializar em `main.ts` antes de criar o app: `Sentry.init({ dsn: SENTRY_DSN, environment: NODE_ENV, tracesSampleRate: NODE_ENV === 'production' ? 0.1 : 1.0 })`
- [x] Integrar com `AllExceptionsFilter` — `Sentry.captureException` para erros >= 500
- [x] `SENTRY_DSN` ausente em dev → Sentry desabilitado silenciosamente (não lançar erro)
- [x] Teste: mock erro 500 → `Sentry.captureException` chamado uma vez; `SENTRY_DSN` undefined → Sentry.init não chamado (ou chamado com DSN vazio sem crash)

### 22. Docker Compose para desenvolvimento local

- [x] Criar `docker-compose.yml` na raiz de `apps/ai/` com 3 serviços conforme Doc 22:
  1. `postgres`: `postgres:17-alpine`, porta `5432`, volume persistente, env `POSTGRES_DB=repasse_ai`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
  2. `redis`: `redis:7.4-alpine`, porta `6379`
  3. `rabbitmq`: `rabbitmq:4-management-alpine`, portas `5672` e `15672` (management UI)
- [x] Criar `.env.example` com as 18 variáveis (sem valores sensíveis reais — apenas placeholders)
- [x] Criar `.env.development` com valores locais + `FEATURE_WHATSAPP_MOCK=true`
- [x] Teste: `docker compose up -d` → todos os 3 containers `healthy`; `prisma migrate deploy` → sem erros; `prisma db seed` → 9 registros em `agent_configurations`

---

## 🔌 WIRING — Integrações Base

### 23. Cliente OpenAI base

- [x] Instalar `openai` SDK 4.x+ e criar `OpenAIProvider` em `src/config/`
- [x] Configurar com `OPENAI_API_KEY` + timeout 30s + retry 3x com backoff exponencial
- [x] Fixar versão do modelo via env `OPENAI_MODEL` — não hardcodar `"gpt-4o"` no código
- [x] Criar `OpenAIHealthIndicator` — chamar `openai.models.list()` com timeout 5s para health check
- [x] Teste: mock OpenAI SDK → `openai.chat.completions.create()` chamável; API key inválida → erro tipado `AuthenticationError` (não genérico)

### 24. Langfuse base

- [x] Instalar `langfuse` SDK e criar `LangfuseProvider` em `src/config/`
- [x] Configurar com `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`
- [x] Criar wrapper `LangfuseTracer` com métodos: `startTrace(name, input)`, `endTrace(traceId, output, metadata)`, `logCost(traceId, model, promptTokens, completionTokens)`
- [x] Teste: mock Langfuse → `startTrace` retorna `traceId` válido; `endTrace` com `traceId` inexistente → não lançar erro (fire-and-forget)

### 25. Prisma schema completo (`schema.prisma`)

- [x] Criar `schema.prisma` com `datasource db { provider = "postgresql", url = env("DATABASE_URL") }` e `generator client { provider = "prisma-client-js", previewFeatures = ["postgresqlExtensions"] }`
- [x] Habilitar extensão `pgvector` via `extensions = [pgvector(map: "vector")]`
- [x] Declarar os 13 modelos conforme Doc 13: `ChatConversation`, `ChatMessage`, `AiInteraction`, `LlmCacheEntry`, `DocumentEmbedding`, `WhatsappBinding`, `OtpAttempt`, `NotificationPreference`, `NotificationEvent`, `AiTakeover`, `AgentConfiguration`, `AgentStatusLog`, `LgpdConsent`
- [x] Declarar os 17 enums conforme Doc 13: `ConversationStatus`, `MessageRole`, `CacheType`, `WhatsappBindingStatus`, `OtpChannel`, `OtpStatus`, `NotificationChannel`, `NotificationType`, `TakeoverReason`, `AgentStatus`, `LgpdConsentType` + enums de suporte
- [x] Rodar `prisma generate` → sem erros; rodar `prisma validate` → schema válido
- [x] Teste: `prisma migrate dev --name init` em banco limpo → sem erros; `prisma db push` → schema aplicado

---

## ✅ TESTES — Fundação

### 26. Configuração Jest

- [x] Criar `jest.config.ts` na raiz de `apps/ai/` com:
  - `testEnvironment: 'node'`
  - `transform: { '^.+\\.tsx?$': ['ts-jest', { tsconfig: 'tsconfig.json' }] }`
  - `collectCoverageFrom: ['src/**/*.ts', '!src/**/*.module.ts', '!src/main.ts']`
  - `coverageThresholds: { global: { branches: 70, functions: 70, lines: 70, statements: 70 } }` — mínimo base para S1; módulos específicos sobem nas sprints de feature
  - `testPathPattern` para separar `unit/`, `integration/`, `e2e/`
- [x] Criar `test/unit/common/filters/all-exceptions.filter.spec.ts` — testes unitários do `AllExceptionsFilter` conforme item 18
- [x] Criar `test/unit/common/guards/jwt-auth.guard.spec.ts` — testes unitários do `JwtAuthGuard` conforme item 19
- [x] Teste: `pnpm test` → todos os testes passam; `pnpm test:cov` → cobertura >= 70% nos arquivos de `common/`

---

## 🔀 Cross-Módulo

_(Sem cross-módulo nesta sprint — fundação não dispara ações em outros módulos de negócio)_

---

## 🤖 CI/CD — Pipeline GitHub Actions

### 27. Workflow CI base

- [x] Criar `.github/workflows/ci.yml` conforme Doc 24:
  - Trigger: `push` em qualquer branch + `pull_request` para `main` e `staging`
  - Jobs em sequência: `lint` → `typecheck` → `unit-tests` → `integration-tests` → `build` → `security-audit`
  - Job `lint`: `pnpm lint` — ESLint com regra `no-console: error`
  - Job `typecheck`: `tsc --noEmit`
  - Job `unit-tests`: `pnpm test:unit --coverage`
  - Job `integration-tests`: sobe serviços via `services:` (postgres, redis, rabbitmq) + `pnpm test:integration`
  - Job `build`: `pnpm build` → artefato gerado sem erros
  - Job `security-audit`: `pnpm audit --audit-level=high` — falhar se vulnerabilidade HIGH ou CRITICAL
- [x] Configurar 2 approvals obrigatórios para merge em `main` (Branch Protection Rules documentadas)
- [x] Teste: criar PR → pipeline CI executa todos os 6 jobs; merge sem aprovações → bloqueado

---

## ✔️ Auto-Verificação S1 (12 Checks)

| #   | Check                                                                                                                                  | Status |
| --- | -------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis (feito/não feito)                                                                         | ✅     |
| 2   | Nomes de tabelas, colunas, enums e arquivos exatamente como nos docs                                                                   | ✅     |
| 3   | Valores numéricos idênticos aos docs (90 dias retenção, 30min TTL Redis, 9 configs seed, 18 env vars, 1536 dims, 13 tabelas, 17 enums) | ✅     |
| 4   | Glossário consultado — termos do domínio corretos                                                                                      | ✅     |
| 5   | Nenhum item permite scaffold vazio — cada item tem sub-itens com lógica, validações e testes (Anti-Scaffold R10)                       | ✅     |
| 6   | Máquinas de estado: `conversation_status`, `whatsapp_binding_status`, `otp_status`, `agent_status` documentadas com todos os valores   | ✅     |
| 7   | TTLs documentados: Redis TTL 30min, cache exact 24h, seed `confidence_threshold=0.80`                                                  | ✅     |
| 8   | Sem conflitos entre docs não marcados                                                                                                  | ✅     |
| 9   | Sem ambiguidades não resolvidas                                                                                                        | ✅     |
| 10  | Template A aplicado: Banco→Backend→Frontend(N/A)→Wiring→Testes                                                                         | ✅     |
| 11  | Sem itens de negócio — apenas infraestrutura                                                                                           | ✅     |
| 12  | REQs cobertos: REQ-007, REQ-114 a REQ-158, REQ-168 a REQ-173, REQ-178 — todos com ≥1 item no checklist                                 | ✅     |
