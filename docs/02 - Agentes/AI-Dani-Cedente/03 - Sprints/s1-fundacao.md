# S1 — Fundação

| Campo                | Valor                                                                                                                                                                                           |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S1                                                                                                                                                                                              |
| **Nome**             | Fundação                                                                                                                                                                                        |
| **Tipo**             | Fixa de Infraestrutura                                                                                                                                                                          |
| **Template**         | A (Infraestrutura)                                                                                                                                                                              |
| **Docs Consultados** | D02, D12, D13, D14, D15, D16, D20, D22, D23                                                                                                                                                     |
| **Cross-cutting**    | D10 (Glossário), D02 (Stacks)                                                                                                                                                                   |
| **REQs cobertos**    | REQ-085 a REQ-100, REQ-116, REQ-119, REQ-124 a REQ-127, REQ-130 a REQ-155, REQ-163, REQ-166, REQ-183, REQ-184                                                                                   |
| **Objetivo**         | Projeto roda localmente, banco existe com esquema completo (todas as migrations aplicadas), 12 módulos NestJS scaffolded, filas e cache configurados, error handling global, Swagger, RLS ativo |

---

## Critério de Conclusão da S1

Ao final desta sprint:

- `docker compose up` sobe PostgreSQL 17 (Supabase), Redis 7.4 e RabbitMQ 4 sem erros
- `GET /api/v1/health` retorna HTTP 200 com status de todos os serviços
- `npx prisma migrate deploy` aplica todas as migrations sem erro
- `npx prisma db execute --file prisma/rls/policies.sql` aplica RLS sem erro
- `npx prisma db execute --file prisma/rls/indexes.sql` aplica indexes (incluindo IVFFlat pgvector) sem erro
- Swagger acessível em `/api/v1/docs` com todos os endpoints iniciais listados
- Zero lógica de negócio implementada — apenas infraestrutura

---

## 🗄️ BANCO

### Migrations e Schema Prisma

- [x] **Configurar `prisma/schema.prisma`** com `generator client { provider = "prisma-client-js"; previewFeatures = ["postgresqlExtensions"] }` e `datasource db { provider = "postgresql"; url = env("DATABASE_URL"); directUrl = env("DIRECT_URL"); extensions = [pgcrypto, pg_trgm] }` — sem declarar pgvector nativo (gerenciado via SQL raw)
  - Validação: `pnpm prisma validate` sem erros; extensões `pgcrypto` e `pg_trgm` declaradas
- [x] **Criar enum `KycStatus`** com valores: `PENDENTE`, `APROVADO`, `REPROVADO`
  - Validação: enum presente no schema com os 3 valores exatos
- [x] **Criar enum `OpportunityStatus`** com valores: `RASCUNHO`, `PUBLICADA`, `EM_NEGOCIACAO`, `PAUSADA`, `EXPIRADA`, `CANCELADA`
  - Validação: enum presente com os 6 valores exatos
- [x] **Criar enum `ScenarioType`** com valores: `A`, `B`, `C`, `D`
  - Validação: enum presente com os 4 valores exatos
- [x] **Criar enum `ProposalStatus`** com valores: `RECEBIDA`, `EM_ANALISE`, `ACEITA`, `RECUSADA`, `CONTRAPROPOSTA`, `CANCELADA`, `EXPIRADA`
  - Validação: enum presente com os 7 valores exatos
- [x] **Criar enum `ProposalType`** com valores: `INICIAL`, `CONTRAPROPOSTA`
  - Validação: enum presente com os 2 valores exatos
- [x] **Criar enum `DossierDocumentType`** com valores: `RG_CNH`, `CPF`, `COMPROVANTE_RESIDENCIA`, `CONTRATO_FINANCIAMENTO`, `EXTRATO_FINANCIAMENTO`, `DECLARACAO_QUITACAO`
  - Validação: enum presente com os 6 valores exatos
- [x] **Criar enum `DossierDocumentStatus`** com valores: `AGUARDANDO_ENVIO`, `EM_ANALISE`, `APROVADO`, `REJEITADO`
  - Validação: enum presente com os 4 valores exatos
- [x] **Criar enum `EscrowStatus`** com valores: `AGUARDANDO_DEPOSITO`, `DEPOSITO_CONFIRMADO`, `LIBERADO`, `REVERTIDO`, `EXPIRADO`
  - Validação: enum presente com os 5 valores exatos
- [x] **Criar enum `MessageRole`** com valores: `USER`, `ASSISTANT`, `SYSTEM`
  - Validação: enum presente com os 3 valores exatos
- [x] **Criar enum `ChatSessionStatus`** com valores: `ATIVA`, `ENCERRADA`
  - Validação: enum presente com os 2 valores exatos
- [x] **Criar model `CedenteProfile`** com campos: `id String @id @default(uuid())`, `user_id String @unique`, `full_name String`, `cpf String @unique`, `kyc_status KycStatus @default(PENDENTE)`, `kyc_verified_at DateTime?`, `created_at DateTime @default(now())`, `updated_at DateTime @updatedAt`, `deleted_at DateTime?`, `created_by String @default("system")`, `updated_by String?`; relações: `opportunities Opportunity[]`, `dossier_documents DossierDocument[]`, `chat_sessions ChatSession[]`; `@@map("cedente_profiles")`
  - Validação: `pnpm prisma validate` sem erros; migration gerada com `CREATE TABLE cedente_profiles`
- [x] **Criar model `Opportunity`** com campos: `id(UUID PK)`, `cedente_id(FK CedenteProfile)`, `status OpportunityStatus @default(RASCUNHO)`, `financing_contract_number String`, `asking_price Decimal @db.Decimal(12,2)`, `outstanding_balance Decimal @db.Decimal(12,2)`, `amount_paid Decimal @db.Decimal(12,2)`, `delta Decimal @db.Decimal(12,2)`, `published_at DateTime?`, `withdrawn_at DateTime?`, `version Int @default(1)` (concorrência otimista), `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`; relações: `cedente @relation(..., onDelete: Restrict)`, `scenarios OpportunityScenario[]`, `proposals Proposal[]`; `@@map("opportunities")`
  - Validação: campo `version` presente; todos os campos Decimal com `@db.Decimal(12,2)`; FK para `cedente_profiles`
- [x] **Criar model `OpportunityScenario`** com campos: `id(UUID PK)`, `opportunity_id(FK)`, `cedente_id String` (redundante para RLS), `scenario_type ScenarioType`, `conditions Json`, `is_active Boolean @default(true)`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`; relação: `opportunity Opportunity @relation(..., onDelete: Cascade)`; `@@unique([opportunity_id, scenario_type])`; `@@map("opportunity_scenarios")`
  - Validação: `@@unique` presente; `cedente_id` redundante para RLS; `conditions Json`
- [x] **Criar model `Proposal`** com campos: `id(UUID PK)`, `opportunity_id(FK)`, `cedente_id String` (redundante para RLS), `proposal_value Decimal @db.Decimal(12,2)`, `status ProposalStatus @default(RECEBIDA)`, `type ProposalType`, `expires_at DateTime`, `responded_at DateTime?`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`; relações; `@@map("proposals")`
  - Validação: `proposal_value` com `@db.Decimal(12,2)`; `expires_at` obrigatório
- [x] **Criar model `DossierDocument`** com campos: `id(UUID PK)`, `opportunity_id(FK)`, `cedente_id String`, `document_type DossierDocumentType`, `status DossierDocumentStatus @default(AGUARDANDO_ENVIO)`, `rejection_reason String?`, `storage_path String?`, `expires_at DateTime?`, `approved_at DateTime?`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`; `@@map("dossier_documents")`
  - Validação: `rejection_reason` nullable; `approved_at` nullable; storage_path nullable
- [x] **Criar model `EscrowTransaction`** com campos: `id(UUID PK)`, `proposal_id(FK Proposal, unique)`, `cedente_id String`, `amount Decimal @db.Decimal(12,2)`, `status EscrowStatus @default(AGUARDANDO_DEPOSITO)`, `deposit_deadline DateTime`, `deposited_at DateTime?`, `released_at DateTime?`, `reversed_at DateTime?`, `extension_requested Boolean @default(false)`, `extension_approved Boolean?`, `extension_requested_at DateTime?`, `extension_approved_at DateTime?`, `created_at`, `updated_at`; `@@map("escrow_transactions")`
  - Validação: `extension_requested_at` e `extension_approved_at` presentes (rastreabilidade); `proposal_id` com `@unique`
- [x] **Criar model `ChatSession`** com campos: `id(UUID PK)`, `cedente_id(FK CedenteProfile)`, `opportunity_id String?` (FK nullable), `proposal_id String?` (FK nullable), `entry_point String`, `status ChatSessionStatus @default(ATIVA)`, `message_count Int @default(0)`, `csat_score Int?`, `last_message_at DateTime?`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`; relação: `messages ChatMessage[]`; `@@map("chat_sessions")`
  - Validação: `opportunity_id` e `proposal_id` nullable; `csat_score` nullable (escala 1–5)
- [x] **Criar model `ChatMessage`** com campos: `id(UUID PK)`, `session_id(FK ChatSession)`, `role MessageRole`, `content String @db.Text`, `langfuse_trace_id String?`, `confidence_score Float?`, `admin_takeover Boolean @default(false)`, `created_at DateTime @default(now())`; **sem** `updated_at` nem `deleted_at` (append-only); `@@map("chat_messages")`
  - Validação: ausência de `updated_at` e `deleted_at` (append-only); `content @db.Text`
- [x] **Criar model `KnowledgeEmbedding`** com campos: `id(UUID PK)`, `title String`, `content String @db.Text`, `category String`, `created_at DateTime @default(now())`, `updated_at DateTime @updatedAt`; **sem** `cedente_id` (conhecimento compartilhado); **sem** campo vector no Prisma (gerenciado via SQL raw); `@@map("knowledge_embeddings")`
  - Validação: ausência de `cedente_id`; nota no arquivo sobre pgvector via SQL raw

### RLS e Indexes

- [x] **Criar `prisma/rls/policies.sql`** com `ALTER TABLE cedente_profiles ENABLE ROW LEVEL SECURITY` e policy por `user_id = auth.uid()` — repetir para: `opportunities`, `opportunity_scenarios`, `proposals`, `dossier_documents`, `escrow_transactions`, `chat_sessions`, `chat_messages`
  - Validação: todas as 8 tabelas de domínio com RLS habilitado; `knowledge_embeddings` sem RLS (compartilhado)
- [x] **Criar `prisma/rls/indexes.sql`** com: `CREATE EXTENSION IF NOT EXISTS vector;`, coluna `embedding vector(1536)` na tabela `knowledge_embeddings`, `CREATE INDEX CONCURRENTLY idx_knowledge_embeddings_embedding ON knowledge_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);`, indexes adicionais em `cedente_id` para tabelas frequentemente filtradas
  - Validação: extensão `vector` criada; IVFFlat com `lists=100` e `vector_cosine_ops`; dimensão `1536`
- [x] **Criar `prisma/middleware/cedente-isolation.middleware.ts`** com lógica de injeção do `cedente_id` do JWT no request context via AsyncLocalStorage
  - Validação: `cedente_id` nunca vem de `req.body` — apenas do JWT deconstruído
- [x] **Criar `prisma/middleware/soft-delete.middleware.ts`** com interceptação de `delete` → converte para `update { deleted_at: new Date() }`
  - Validação: nenhuma deleção física em tabelas de domínio; `findMany` e `findUnique` filtram `deleted_at: null` automaticamente
- [x] **Criar `prisma/seed.ts`** com dados de desenvolvimento: 2 CedenteProfiles de teste, 2 oportunidades (status RASCUNHO e PUBLICADA), 1 proposta de teste, embeddings de exemplo para knowledge_embeddings (mínimo 1 por categoria)
  - Validação: `pnpm prisma db seed` executa sem erros em ambiente local; ao menos 1 embedding por categoria inserido (necessário para IVFFlat)

---

## ⚙️ BACKEND

### Setup do Projeto NestJS

- [x] **Criar `src/main.ts`** com bootstrap: `NestFactory.create(AppModule)`, `app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }))`, `app.use(helmet())`, `app.setGlobalPrefix('api/v1')`, `DocumentBuilder` para Swagger com `SwaggerModule.setup('api/v1/docs', app, document)`, listen na porta `PORT` env var
  - Validação: `GET /api/v1/docs` retorna 200 com Swagger UI; `GET /api/v1/health` retorna 200
- [x] **Criar `src/app.module.ts`** importando: `ConfigModule.forRoot({ isGlobal: true, validate: ... })`, `PrismaModule`, `RedisModule`, `RabbitMQModule`, e todos os 12 módulos de domínio (auth, chat, agent, rag, opportunity, proposal, dossier, escrow, notification, simulation, fallback, admin)
  - Validação: aplicação inicia sem erros de injeção de dependência; todos os módulos carregados
- [x] **Criar `src/config/configuration.ts`** com validação Zod de todas as 22 variáveis de ambiente obrigatórias (conforme D22): DATABASE_URL, DIRECT_URL, REDIS_URL, RABBITMQ_URL, OPENAI_API_KEY, SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY, SUPABASE_JWT_SECRET, LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY, POSTHOG_API_KEY, EMAIL_PROVIDER, EMAIL_FROM, ZAPSIGN_API_KEY, ZAPSIGN_WEBHOOK_SECRET, PORT, NODE_ENV, LOG_LEVEL, CORS_ORIGINS, REPASSE_API_URL, REPASSE_API_KEY
  - Validação: aplicação falha na inicialização se qualquer var obrigatória estiver ausente; erro descritivo no log
- [x] **Criar `src/infrastructure/prisma/prisma.service.ts`** implementando `PrismaClient` com hooks `onModuleInit`/`onModuleDestroy`, aplicando middleware de soft delete e cedente isolation
  - Validação: `PrismaService` injetável em qualquer módulo; soft delete e isolamento ativos
- [x] **Criar `src/infrastructure/redis/redis.service.ts`** com cliente Redis 7.4+, métodos `get`, `set`, `del`, `incr`, `expire`, health check via `PING`
  - Validação: `GET /api/v1/health` inclui status do Redis; TTL obrigatório em todo `set`
- [x] **Criar `src/infrastructure/rabbitmq/rabbitmq.service.ts`** com configuração das 3 filas: `rag.ingest`, `notification.send`, `escrow.events`; cada fila com DLQ (`rag.ingest.dlq`, `notification.dlq`, `escrow.events.dlq`) e retry policy (max 3 tentativas, backoff exponencial 1s/2s/4s)
  - Validação: `GET /api/v1/health` inclui status do RabbitMQ; filas e DLQs criadas no broker

### Estrutura de Módulos (scaffolding)

- [x] **Criar estrutura de pastas para todos os 12 módulos** seguindo o padrão: `src/modules/<dominio>/<dominio>.module.ts`, `<dominio>.controller.ts`, `<dominio>.service.ts`, `<dominio>.repository.ts`, `dto/`, `entities/`
  - Módulos: auth, chat, agent, rag, opportunity, proposal, dossier, escrow, notification, simulation, fallback, admin
  - Validação: 12 pastas criadas com os 4 arquivos base cada; aplicação compila sem erros
- [x] **Criar `src/common/filters/global-exception.filter.ts`** com `@Catch()` capturando todas as exceções não tratadas, formatando a resposta com o schema `{ error: { code: "DCE-MODULE-HTTPSTATUS_SEQ", category, message, user_message, correlation_id, timestamp, details } }` e logging via Pino
  - Validação: qualquer erro não tratado retorna o schema correto; stack trace nunca exposto ao cliente; correlation_id presente
- [x] **Criar `src/common/interceptors/logging.interceptor.ts`** com logging Pino de todo request/response: método HTTP, path, status, duration_ms, correlation_id — sem CPF, e-mail, telefone, Authorization no log
  - Validação: logs em JSON estruturado; PII ausente; `correlation_id` presente em 100% dos logs
- [x] **Criar `src/common/middleware/pii-masking.middleware.ts`** com mascaramento de `cpf`, `email`, `telefone`, `Authorization` antes de qualquer operação de log
  - Validação: logs de integração nunca contêm CPF real; testes unitários verificam mascaramento
- [x] **Criar `src/common/decorators/@CurrentCedente()`** que extrai `cedente_id` do request context (AsyncLocalStorage) — nunca de `req.body`
  - Validação: uso do decorator em qualquer controller extrai `cedente_id` correto; teste unitário confirma
- [x] **Criar `src/common/decorators/@RequireRole(role)`** que aplica `RbacGuard` na rota com o role especificado
  - Validação: rota com `@RequireRole('admin')` bloqueia request com token de `cedente`; retorna 403

### Error Handling Global

- [x] **Criar hierarquia de erros `DaniBaseError`** com subclasses: `DaniValidationError` (400), `DaniAuthError` (401), `DaniForbiddenError` (403), `DaniNotFoundError` (404), `DaniConflictError` (409), `DaniBusinessRuleError` (422), `DaniRateLimitError` (429), `DaniInternalError` (500), `DaniExternalServiceError` (502/503), `DaniAgentUnavailableError` (503)
  - Validação: todos os erros herdam de `DaniBaseError`; cada subclasse tem code pattern `DCE-{MÓDULO}-{HTTP_STATUS}{SEQ}`; nenhuma string "genérica" como mensagem

### Infraestrutura Adicional

- [x] **Criar `src/infrastructure/openai/openai.service.ts`** com cliente `openai` npm 4.x+, configurado com `OPENAI_API_KEY`, métodos `streamText()` (Vercel AI SDK) e `embedText()` (text-embedding-3-small) — sem inicializar a lógica de negócio do agente (apenas o cliente)
  - Validação: serviço injetável; `OPENAI_API_KEY` nunca hardcoded; nenhuma chamada real em testes
- [x] **Criar `src/infrastructure/langfuse/langfuse.service.ts`** com SDK `langfuse` 3.x+, métodos `createTrace()`, `createSpan()`, `createScore()` — obrigatório antes de qualquer deploy
  - Validação: serviço injetável; `LANGFUSE_PUBLIC_KEY` e `LANGFUSE_SECRET_KEY` via env

### Health Check

- [x] **Criar endpoint `GET /api/v1/health`** retornando `{ status: "ok", services: { postgres: "ok"|"error", redis: "ok"|"error", rabbitmq: "ok"|"error" }, timestamp }` com HTTP 200 se todos OK ou HTTP 503 se algum falhar
  - Validação: após `docker compose up`, retorna 200; após derrubar Redis container, retorna 503 com `redis: "error"`
- [x] **Criar endpoint `GET /api/v1/health/ready`** e `GET /api/v1/health/live` conforme D29 para Railway health check automático (verifica a cada 30s e 15s respectivamente)
  - Validação: `/health/ready` verifica todas as dependências; `/health/live` apenas verifica se o processo está vivo

---

## 🐳 DOCKER COMPOSE E AMBIENTE LOCAL

- [x] **Criar `docker-compose.yml`** com 3 serviços: `postgres` (supabase/postgres:15.1.0.117, porta 5432, healthcheck pg_isready), `redis` (redis:7.4-alpine, porta 6379), `rabbitmq` (rabbitmq:4-management, porta 5672 e 15672), com healthchecks, volumes e network `dani_network`
  - Validação: `docker compose up -d` sobe os 3 serviços sem erros; health checks passam em ≤60s
- [x] **Criar `.env.example`** com todas as 22 variáveis de ambiente documentadas conforme D22 (valores placeholder, não credenciais reais)
  - Validação: `.env.example` commitado no repositório; `.env.local` e `.env` no `.gitignore`
- [x] **Verificar `.gitignore`** incluindo: `.env`, `.env.local`, `.env.production`, `node_modules/`, `dist/`, `coverage/`
  - Validação: `git status` não mostra nenhum arquivo `.env.*` como tracked

---

## 📋 GIT FLOW E CONTRIBUIÇÃO

- [x] **Configurar Husky** com hook `pre-commit`: executa `pnpm lint-staged` (lint + type-check em arquivos staged)
  - Validação: `pnpm prepare` instala hooks; commit com erro TypeScript é bloqueado pelo hook
- [x] **Criar `.github/workflows/ci.yml`** com jobs: `lint-typecheck` (ESLint + `tsc --noEmit`), `unit-tests` (Jest), `integration-tests` (Jest + supertest); trigger: `pull_request` para `develop` e `main`; runner: `ubuntu-latest`; tempo máximo: 15 minutos
  - Validação: workflow executável; lint bloqueia PR com erros; coverage report gerado como artefato
- [x] **Criar `.github/workflows/deploy.yml`** com deploy automático para Railway em `push develop` (staging) e `tag v*` (produção)
  - Validação: workflow presente; deploy manual funciona via `railway up`
- [x] **Criar template de PR `.github/pull_request_template.md`** com seções: Descrição, Tipo de mudança, Checklist de segurança (cedenteId do contexto, sem console.log, sem credenciais hardcoded), Evidências de teste
  - Validação: template aparece ao abrir PR no GitHub

---

## 🧪 TESTES

- [x] **Configurar Jest** com `jest.config.ts`: cobertura mínima 80% geral (configurado em `coverageThreshold`), 90% para módulos críticos (agent, auth, escrow, fallback); `ts-jest` como transformer; `testEnvironment: "node"
  - Validação: `pnpm test --coverage` gera relatório; build falha se cobertura < 80% global
- [x] **Criar teste unitário `src/common/filters/global-exception.filter.spec.ts`** verificando: (1) erro DaniValidationError retorna HTTP 400 com schema correto; (2) stack trace nunca no response; (3) correlation_id presente no response
  - Validação: `pnpm test global-exception.filter` verde
- [x] **Criar teste unitário `src/common/middleware/pii-masking.middleware.spec.ts`** verificando: (1) CPF é mascarado no log; (2) e-mail é mascarado; (3) Authorization header é mascarado; (4) dados sem PII passam intactos
  - Validação: `pnpm test pii-masking` verde

---

## 🔍 AUTO-VERIFICAÇÃO S1

| Check                | Critério                                                                                                                                    | Status |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura      | Todos os nomes usam os nomes exatos dos docs (tabelas em snake_case, enums em PascalCase, módulos em kebab-case)                            | [x]    |
| #2 Valores numéricos | Decimal(12,2) em todos os campos monetários; TTL 1800s Redis; listas IVFFlat 100                                                            | [x]    |
| #3 Enums completos   | 11 enums com TODOS os valores listados — nenhum valor omitido                                                                               | [x]    |
| #4 Soft delete       | `deleted_at DateTime?` em todas as entidades de negócio (exceto ChatMessage e KnowledgeEmbedding)                                           | [x]    |
| #5 RLS               | 8 tabelas de domínio com RLS habilitado; knowledge_embeddings sem RLS                                                                       | [x]    |
| #6 Filas             | 3 queues: rag.ingest, notification.send, escrow.events — cada uma com DLQ e retry                                                           | [x]    |
| #7 Redis             | 3 chaves com TTL: agent_state (1800s), rate_limit, cache LLM — TTL explícito obrigatório                                                    | [x]    |
| #8 Anti-scaffold     | GlobalExceptionFilter com schema DCE; soft-delete middleware; PII masking — não apenas stubs                                                | [x]    |
| #9 Env vars          | 22 variáveis validadas via Zod; aplicação falha se ausente                                                                                  | [x]    |
| #10 Glossário        | ChatMessage é append-only (sem updated_at/deleted_at); KnowledgeEmbedding sem cedente_id; version para concorrência otimista em Opportunity | [x]    |
| #11 Cobertura REQs   | REQ-085 a REQ-100, REQ-116–127, REQ-130–155, REQ-163–166, REQ-183–184 todos cobertos                                                        | [x]    |
| #12 Testes           | Testes unitários de GlobalExceptionFilter e PII masking passando                                                                            | [x]    |
