# S1 — Fundação

| **Sprint**         | S1 — Fundação                                                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Template**       | A — Infraestrutura (Banco → Backend → Frontend → Wiring → Testes)                                                                     |
| **REQs cobertos**  | REQ-040 a REQ-060, REQ-061 a REQ-070, REQ-153, REQ-154, REQ-155, REQ-156                                                              |
| **Docs fonte**     | D02 (Stacks), D12 (ERD), D13 (Prisma), D14 (Especificações Técnicas), D15 (Arquitetura de Pastas), D22 (Ambiente), D23 (Contribuição) |
| **Total de itens** | 52                                                                                                                                    |

---

## 🎯 Objetivo

Estabelecer a infraestrutura completa do monorepo: estrutura de pastas, banco de dados (todas as 7 tabelas `dani_*`), módulos NestJS registrados, schema Prisma migrado, ambiente local funcional e convenções de desenvolvimento (commits, hooks, CI base).

---

## 🗄️ Banco de Dados

### Feature: Monorepo e Setup Inicial

- [x] **[BANCO-001]** Inicializar monorepo Turborepo + pnpm workspaces com workspaces `apps/web`, `apps/api`, `packages/shared-types`, `packages/design-tokens`
  - Sub-item: `pnpm --version` retorna ≥9.x; `node --version` retorna v20.x.x (LTS)
  - Sub-item: `turbo.json` configurado com pipelines `build`, `test`, `lint`
  - Sub-item: `pnpm install` na raiz instala dependências de todos os workspaces sem erro
  - Sub-item: Estrutura de diretórios conforme D15 (`apps/api/src/agente/`, `apps/api/src/calculadora/`, `apps/api/src/auth/`, `apps/api/src/oportunidade/`, `apps/api/src/alerta/`, `apps/api/src/whatsapp/`, `apps/api/src/cessionario/`)

- [x] **[BANCO-002]** Configurar Docker Compose com 4 serviços conforme D22:
  - Sub-item: `postgres` — imagem `supabase/postgres:15.1.0.117`, porta 5432, DB `dani_dev`, user `postgres`, pass `postgres`; healthcheck `pg_isready -U postgres -d dani_dev`
  - Sub-item: `redis` — imagem `redis:7-alpine`, porta 6379; healthcheck `redis-cli ping`
  - Sub-item: `rabbitmq` — imagem `rabbitmq:3.12-management-alpine`, portas 5672 (AMQP) + 15672 (UI), user `guest` / pass `guest`; healthcheck `rabbitmq-diagnostics ping`
  - Sub-item: `supabase-studio` — imagem `supabase/studio:latest`, porta 54323
  - Sub-item: `docker compose up -d` sobe todos os 4 serviços healthy; `pnpm health` retorna OK para todos

- [x] **[BANCO-003]** Configurar variáveis de ambiente (28 vars conforme D22):
  - Sub-item: `.env.example` com todas as 28 variáveis (17 sensíveis, 11 não-sensíveis) documentadas
  - Sub-item: `.env` no `.gitignore` — verificado por `git-secrets` no CI
  - Sub-item: Variáveis de produção nunca em `.env.local` — verificar que `DATABASE_URL` de produção está bloqueada

### Feature: Schema Prisma e Migrações

- [x] **[BANCO-004]** Implementar schema Prisma completo conforme D13 — 8 enums:
  - Sub-item: Enum `CanalDani` com valores: `WEBCHAT`, `WHATSAPP`
  - Sub-item: Enum `StatusConversa` com valores: `ATIVA`, `ENCERRADA`, `EXPIRADA`
  - Sub-item: Enum `RemetenteMensagem` com valores: `CESSIONARIO`, `DANI`, `CALCULADORA`, `SISTEMA`
  - Sub-item: Enum `EstadoAgente` com valores: `OPERACIONAL`, `LATENCIA_ALTA`, `FALLBACK_ATIVO`, `DESLIGADO_AUTOMATICO`
  - Sub-item: Enum `TipoAlerta` com valores: `NOVA_OPORTUNIDADE`, `OPORTUNIDADE_ENCERRADA`, `NEGOCIACAO_ATUALIZADA`, `ESCROW_PRAZO`, `ZAPSIGN_PENDENTE`
  - Sub-item: Enum `StatusAlerta` com valores: `NAO_LIDO`, `LIDO`
  - Sub-item: Enum `EstadoVinculacaoWhatsapp` com valores: `NAO_VINCULADO`, `OTP_SMS_ENVIADO`, `AGUARDANDO_CONFIRMACAO_WA`, `VINCULADO`, `DESVINCULADO`
  - Sub-item: Enum `EventoRateLimitAudit` com valores: `LIMITE_ATINGIDO`, `DESBLOQUEADO`, `HARD_BLOCK_OTP`

- [x] **[BANCO-005]** Implementar model `DaniConversa` conforme D13:
  - Sub-item: Campos: `id UUID @id @default(uuid())`, `cessionario_id UUID @db.Uuid NOT NULL`, `titulo TEXT?`, `status StatusConversa @default(ATIVA)`, `canal CanalDani @default(WEBCHAT)`, `opr_id TEXT?`, `created_at DateTime @default(now())`, `updated_at DateTime @updatedAt`, `expires_at DateTime?`
  - Sub-item: Índices: `@@index([cessionario_id])`, `@@index([status])`, `@@index([expires_at])`
  - Sub-item: Relation: `mensagens DaniMensagem[]`, `contextos DaniContextoOpr[]`

- [x] **[BANCO-006]** Implementar model `DaniMensagem` conforme D13:
  - Sub-item: Campos: `id UUID @id`, `conversa_id UUID FK -> DaniConversa`, `cessionario_id UUID NOT NULL`, `remetente RemetenteMensagem`, `conteudo TEXT NOT NULL`, `metadata JSON?`, `created_at DateTime @default(now())`
  - Sub-item: Índices: `@@index([conversa_id])`, `@@index([cessionario_id])`, `@@index([created_at])`
  - Sub-item: `expires_at` derivado via join na conversa (não armazenado diretamente)

- [x] **[BANCO-007]** Implementar model `DaniSessao` conforme D13:
  - Sub-item: Campos: `id UUID @id`, `cessionario_id UUID NOT NULL`, `jwt_session_hash TEXT NOT NULL`, `estado_agente EstadoAgente @default(OPERACIONAL)`, `ip_address TEXT?`, `user_agent TEXT?`, `created_at DateTime @default(now())`, `expires_at DateTime NOT NULL`
  - Sub-item: Índices: `@@index([cessionario_id])`, `@@index([jwt_session_hash])`, `@@index([expires_at])`

- [x] **[BANCO-008]** Implementar model `DaniContextoOpr` conforme D13:
  - Sub-item: Campos: `id UUID @id`, `cessionario_id UUID NOT NULL`, `opr_id TEXT NOT NULL`, `dados_oportunidade Json NOT NULL`, `created_at DateTime @default(now())`, `expires_at DateTime?`
  - Sub-item: Índices: `@@index([cessionario_id])`, `@@index([opr_id, cessionario_id])`

- [x] **[BANCO-009]** Implementar model `DaniAlerta` conforme D13:
  - Sub-item: Campos: `id UUID @id`, `cessionario_id UUID NOT NULL`, `tipo TipoAlerta NOT NULL`, `status StatusAlerta @default(NAO_LIDO)`, `titulo TEXT NOT NULL`, `corpo TEXT NOT NULL`, `opr_id TEXT?`, `lido_em DateTime?`, `created_at DateTime @default(now())`
  - Sub-item: Índices: `@@index([cessionario_id])`, `@@index([status, cessionario_id])`

- [x] **[BANCO-010]** Implementar model `DaniVinculacaoWhatsapp` conforme D13:
  - Sub-item: Campos: `id UUID @id`, `cessionario_id UUID @unique NOT NULL`, `numero_hash TEXT?`, `numero_sufixo TEXT?` (últimos 4 dígitos), `estado EstadoVinculacaoWhatsapp @default(NAO_VINCULADO)`, `otp_hash TEXT?` (bcrypt cost 12), `tentativas_otp Int @default(0)`, `otp_expires_at DateTime?`, `created_at DateTime @default(now())`, `updated_at DateTime @updatedAt`
  - Sub-item: Índices: `@@index([cessionario_id])`, `@@index([estado])`
  - Sub-item: `numero_hash`: SHA-256 do número E.164; `numero_sufixo`: últimos 4 dígitos para exibição mascarada `(XX) •••••-XXXX`

- [x] **[BANCO-011]** Implementar model `DaniRateLimitAudit` conforme D13:
  - Sub-item: Campos: `id UUID @id`, `cessionario_id UUID`, `evento EventoRateLimitAudit NOT NULL`, `canal TEXT NOT NULL`, `ip_address TEXT?`, `created_at DateTime @default(now())`
  - Sub-item: Tabela append-only — sem `updated_at`
  - Sub-item: Índice: `@@index([cessionario_id, created_at])`

- [x] **[BANCO-012]** Executar migration inicial conforme D13:
  - Sub-item: Comando: `npx prisma migrate dev --name init_dani_module`
  - Sub-item: Migration aplica todas as 7 tabelas sem erro
  - Sub-item: `npx prisma generate` gera o Prisma Client sem erros TypeScript
  - Sub-item: Seed de dados de desenvolvimento executável via `pnpm db:seed`

---

## 🔧 Backend

### Feature: Estrutura de Módulos NestJS

- [x] **[BACK-001]** Implementar estrutura de módulos NestJS conforme D14 §3 — 7 módulos registrados no `AppModule`:
  - Sub-item: `AuthModule` — `src/auth/auth.module.ts` declarado e importado
  - Sub-item: `AgenteModule` — `src/agente/agente.module.ts` declarado e importado
  - Sub-item: `CalculadoraModule` — `src/calculadora/calculadora.module.ts` declarado e importado
  - Sub-item: `OportunidadeModule` — `src/oportunidade/oportunidade.module.ts` declarado e importado
  - Sub-item: `AlertaModule` — `src/alerta/alerta.module.ts` declarado e importado
  - Sub-item: `WhatsappModule` — `src/whatsapp/whatsapp.module.ts` declarado e importado (Fase 2 — registrado mas controllers vazios)
  - Sub-item: `CessionarioModule` — `src/cessionario/cessionario.module.ts` declarado e importado
  - Sub-item: `CalculadoraModule` não importa `AgenteModule` e vice-versa — verificado via `@nestjs/common` circular dependency check (REQ-105)

- [x] **[BACK-002]** Implementar convenção de estrutura por módulo conforme D14 §3.1:
  - Sub-item: Cada módulo tem: `<modulo>.module.ts`, `<modulo>.controller.ts`, `<modulo>.service.ts`, `<modulo>.repository.ts`
  - Sub-item: Subdiretórios: `dto/create-*.dto.ts`, `dto/response-*.dto.ts`, `guards/<modulo>.guard.ts`
  - Sub-item: `AgenteModule` tem subdiretório `prompts/dani-system-prompt.v1.ts` (placeholder versionado)
  - Sub-item: Build `pnpm --filter apps/api build` passa sem erros TypeScript

- [x] **[BACK-003]** Configurar Prisma Service global:
  - Sub-item: `PrismaService` em `src/prisma/prisma.service.ts` extende `PrismaClient`
  - Sub-item: `onModuleInit` abre conexão; `onModuleDestroy` fecha conexão
  - Sub-item: `connection_limit=10` configurado no datasource (PgBouncer — REQ-137)
  - Sub-item: Timeout de query: 30s no datasource
  - Sub-item: `PrismaModule` exportado e importado nos módulos que precisam

- [x] **[BACK-004]** Configurar Redis (ioredis) Service:
  - Sub-item: `RedisService` em `src/redis/redis.service.ts` usando `ioredis ≥5.x`
  - Sub-item: Retry: 3x com backoff 100ms/500ms/2s (REQ-138)
  - Sub-item: Timeout por comando: 100ms configurado em `commandTimeout`
  - Sub-item: `RedisModule` global, exportado para todos os módulos que precisam
  - Sub-item: Teste de conexão: `redis-cli -u $REDIS_URL ping` retorna `PONG`

- [x] **[BACK-005]** Configurar RabbitMQ via @nestjs/microservices:
  - Sub-item: Fila `dani.notificacoes` — fanout exchange, DLQ `dani.notificacoes.dlq`, retry 3x (5s/15s/30s)
  - Sub-item: Fila `dani.whatsapp` — direct exchange, DLQ `dani.whatsapp.dlq`, retry 3x (5s/15s/30s)
  - Sub-item: Fila `dani.agent_monitor` — direct exchange, DLQ `dani.agent_monitor.dlq`, retry 3x (2s/5s/10s)
  - Sub-item: Acesso ao RabbitMQ Management UI em `localhost:15672` com user `guest` / pass `guest` confirma 3 exchanges + 3 filas + 3 DLQs criadas

- [x] **[BACK-006]** Configurar Pino Logger global:
  - Sub-item: `PinoLogger` configurado em `main.ts` via `nestjs-pino`
  - Sub-item: Redact obrigatório: `req.headers.authorization`, `req.body.otp`, `req.body.phone`, `*.cpf`, `*.email`, `OPENAI_API_KEY` → `[Redacted]`
  - Sub-item: Campos obrigatórios em todo log: `timestamp`, `level`, `service: "dani-api"`, `module`, `correlation_id`
  - Sub-item: `NODE_ENV=production` → sem stack trace em logs; `NODE_ENV=development` → stack trace incluída
  - Sub-item: Retenção: debug 7 dias, info 30 dias, warn/error 90 dias (configuração de aggregator)

- [x] **[BACK-007]** Implementar `AllExceptionsFilter` global conforme D20 §3.2:
  - Sub-item: `@Catch()` aplicado globalmente em `main.ts` via `app.useGlobalFilters()`
  - Sub-item: Schema de erro padrão: `{ error: { code, category, message, user_message, details?, correlation_id, timestamp } }`
  - Sub-item: `DaniBaseException` e subclasses: `AgenteException`, `CalculadoraException`, `AuthException`, `ValidationException`, `NotFoundDaniException`, `RateLimitException`, `ExternalServiceException`, `WhatsappException`
  - Sub-item: Erro não mapeado retorna `INFRA_000` com `user_message: "Serviço temporariamente indisponível. Tente novamente em instantes."` sem stack trace em produção
  - Sub-item: Teste: `NODE_ENV=production` + erro interno → body não contém `stack`

- [x] **[BACK-008]** Configurar prefixos de erro por domínio conforme D20 §1:
  - Sub-item: Prefixos registrados: `AUTH_001–009`, `AGENTE_001–020`, `CALC_001–010`, `OPR_001–010`, `CESS_001–010`, `ALERTA_001–005`, `WA_001–010`, `INFRA_001–010`, `RATE_001–003`, `VALID_001–020`
  - Sub-item: Cada prefixo mapeado para status HTTP correto (401/403, 503, 422, 404, etc.)

- [x] **[BACK-009]** Implementar job de cleanup de conversas (`cleanup-conversas.job.ts`):
  - Sub-item: Arquivo: `src/agente/jobs/cleanup-conversas.job.ts`
  - Sub-item: Deleta `dani_conversas` onde `expires_at < NOW()` e mensagens relacionadas (cascade)
  - Sub-item: Executa diariamente via `@Cron` do `@nestjs/schedule`
  - Sub-item: Log `info` ao executar com contagem de registros deletados

---

## 🖥️ Frontend

### Feature: Setup do Workspace Web

- [x] **[FRONT-001]** Configurar workspace `apps/web` com React ≥19 + Vite ≥5.x:
  - Sub-item: `pnpm --filter apps/web build` completa sem erros TypeScript
  - Sub-item: `vite.config.ts` com alias de paths configurado (`@/` → `src/`)
  - Sub-item: Tailwind CSS ≥4.x configurado com tokens de design herdados do Repasse Seguro
  - Sub-item: shadcn/ui inicializado (`components.json` presente)

- [x] **[FRONT-002]** Configurar estrutura de features conforme D15:
  - Sub-item: `src/features/dani-chat/index.ts` — barrel export (REQ-150)
  - Sub-item: `src/features/dani-dashboard/index.ts` — barrel export (REQ-150)
  - Sub-item: `src/features/dani-whatsapp/index.ts` — barrel export (Fase 2 — arquivo presente, export vazio)
  - Sub-item: Nenhum componente importa de outro feature sem passar pelo barrel export — verificado por lint rule

- [x] **[FRONT-003]** Configurar TanStack Router ≥1.x + TanStack Query ≥5.x:
  - Sub-item: TanStack Router configurado com rotas: `/cessionario/dashboard`, `/cessionario/oportunidade/:opr_id`, `/cessionario/perfil/whatsapp`
  - Sub-item: TanStack Query `QueryClient` configurado com `staleTime` e `cacheTime` padrão
  - Sub-item: Zustand ≥4.x configurado com store `useDaniChatStore` (`isChatOpen`, `chatContext`)

- [x] **[FRONT-004]** Configurar tokens de Brand Theme conforme D03:
  - Sub-item: Tokens base herdados: `--primary: #0069A8`, `--background: #FFFFFF`, `--foreground: #0A0A0A`, `--muted: #F5F5F5`, `--destructive: #E7000B`, `--border: #E5E5E5`
  - Sub-item: Tokens de extensão Dani: `--risk-low: #16A34A`, `--risk-low-bg: #F0FDF4`, `--risk-medium: #D97706`, `--risk-medium-bg: #FFFBEB`, `--risk-high: #DC2626`, `--risk-high-bg: #FEF2F2`
  - Sub-item: Tokens de status do agente: `--agent-fallback: #0069A8` ⚠️ AMBÍGUO — usar este valor provisoriamente (REQ-126); `--agent-offline: #737373`
  - Sub-item: Tokens de status: `--status-available: #16A34A`, `--status-negotiating: #D97706`, `--status-closed: #737373`
  - Sub-item: Tipografia: `--font-sans: 'Inter Variable', sans-serif` configurado no CSS global
  - Sub-item: `--radius: 0.875rem`, `--radius-sm: 0.525rem`, `--radius-md: 0.7rem`, `--radius-lg: 0.875rem`, `--radius-xl: 1.225rem`, `--radius-2xl: 1.575rem`

---

## 🔌 Wiring

### Feature: CI/CD Base e Git Hooks

- [x] **[WIRE-001]** Configurar Husky git hooks conforme D23:
  - Sub-item: `pre-commit`: lint-staged com ESLint + Prettier
  - Sub-item: `commit-msg`: commitlint validando Conventional Commits v1.0.0
  - Sub-item: Commit `fix stuff` → rejeitado pelo hook com mensagem de erro
  - Sub-item: Commit `fix(agente): handle OpenAI timeout` → aceito pelo hook

- [ ] **[WIRE-002]** <!-- BLOCKED: requires GitHub repo access to create .github/workflows/ci-pr.yml --> Configurar workflow `ci-pr.yml` conforme D24 §3.1:
  - Sub-item: Trigger: `pull_request` em qualquer branch
  - Sub-item: Job `fast-feedback`: `pnpm lint`, `pnpm tsc --noEmit`, `gitleaks-action@v2` (secrets scan), `pnpm commitlint`
  - Sub-item: Job `unit-tests`: `pnpm test:unit --coverage`; gate de cobertura ≥80%: `[ $(echo "$COV >= 80" | bc) -eq 1 ] || exit 1`
  - Sub-item: Falha em qualquer job bloqueia merge

- [x] **[WIRE-003]** Configurar Sentry base conforme D25:
  - Sub-item: Sentry inicializado no `main.ts` (backend) e no `main.tsx` (frontend)
  - Sub-item: `Sentry.captureException()` disponível em `AllExceptionsFilter`
  - Sub-item: `environment` configurado por `NODE_ENV`
  - Sub-item: `user.id` = `sha256(cessionario_id)` — nunca UUID raw

---

## ✅ Testes

- [x] **[TEST-001]** Testes unitários da estrutura de módulos:
  - Sub-item: `AppModule` compila sem erros em test (`createTestingModule`)
  - Sub-item: `PrismaService` conecta ao banco de test sem erro
  - Sub-item: `RedisService` conecta ao Redis de test sem erro
  - Sub-item: `CalculadoraModule` não tem referência a `AgenteModule` (teste de dependências circulares)

- [x] **[TEST-002]** Testes de schema Prisma (integração com banco local):
  - Sub-item: Migration aplicada ao banco de test
  - Sub-item: Todas as 7 tabelas `dani_*` existem: `dani_conversas`, `dani_mensagens`, `dani_sessoes`, `dani_contextos_opr`, `dani_alertas`, `dani_vinculacoes_whatsapp`, `dani_rate_limits_audit`
  - Sub-item: Todos os índices em `cessionario_id` presentes
  - Sub-item: Enums `EstadoVinculacaoWhatsapp`, `TipoAlerta`, `EventoRateLimitAudit` aceitos sem erro

- [x] **[TEST-003]** Testes do `AllExceptionsFilter`:
  - Sub-item: `NODE_ENV=production` + erro interno → body não contém `stack`, contém `correlation_id`
  - Sub-item: `DaniBaseException` retorna schema correto: `{ error: { code, category, message, user_message, correlation_id, timestamp } }`
  - Sub-item: Erro não mapeado retorna `INFRA_000` com status 500

---

## 🔍 Auto-verificação S1 (12 checks)

| #   | Check                                                                                           | Status |
| --- | ----------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis (feito/não feito)                                  | ✅     |
| 2   | Nomes exatos de tabelas, módulos, paths usados (zero sinônimos)                                 | ✅     |
| 3   | Valores numéricos do doc espelhados: TTL 1800s, cost 12, limit=10, etc.                         | ✅     |
| 4   | N itens listados = N itens nos docs (8 enums, 7 tabelas, 7 módulos)                             | ✅     |
| 5   | Glossário D10 consultado: "Conversa" vs "Sessão" — distinção preservada                         | ✅     |
| 6   | Máquinas de estado documentadas: EstadoAgente (4 valores), EstadoVinculacaoWhatsapp (5 valores) | ✅     |
| 7   | Schedules/TTLs documentados: cleanup diário, TTL 1800s/3600s/300s/604800s                       | ✅     |
| 8   | Nenhum conflito entre docs não marcado com ⚠️                                                   | ✅     |
| 9   | Nenhuma suposição não documentada nos docs fonte                                                | ✅     |
| 10  | Nenhum item de scaffold vazio — todos têm sub-itens verificáveis                                | ✅     |
| 11  | Anti-scaffold: cada item tem validações, lógica, erros e testes explícitos                      | ✅     |
| 12  | REQs atribuídos a S1 têm ≥1 item de checklist cobrindo-os                                       | ✅     |

---

## REQs cobertos por esta sprint

REQ-040, REQ-041, REQ-042, REQ-043, REQ-044, REQ-045, REQ-046, REQ-047, REQ-048, REQ-049, REQ-050, REQ-051, REQ-052, REQ-053, REQ-054, REQ-055 (parcial — setup), REQ-057, REQ-058, REQ-059, REQ-060, REQ-061, REQ-062, REQ-063, REQ-064, REQ-065, REQ-066, REQ-067, REQ-068, REQ-069, REQ-070, REQ-153, REQ-154, REQ-155, REQ-156
