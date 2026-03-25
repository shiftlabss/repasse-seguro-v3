# 22 - Guia de Ambiente, Setup Local e Secrets — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Nome do Documento | Guia de Ambiente, Setup Local e Secrets |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Status | Aprovado |
| Dependências | D02 · D12 · D13 · D17 |

---

> **📌 TL;DR**
>
> - **Pré-requisitos:** Node.js 22+, Docker 25+, Docker Compose v2, Git, pnpm 9+.
> - **Tempo estimado:** < 15 minutos para ambiente funcional do zero.
> - **Serviços locais (Docker):** 3 — PostgreSQL 17 (Supabase), Redis 7.4, RabbitMQ 4.
> - **Total de variáveis de ambiente:** 22 (8 obrigatórias sensíveis, 14 obrigatórias não-sensíveis).
> - **Secrets:** gerenciados via `.env.local` (dev) e Railway Secrets / Vault (staging/prod). Nunca em `.env` commitado.
> - **Health check:** `GET /api/v1/health` retorna `200` com todos os serviços ok.
> - **Integrações externas em dev:** OpenAI real (usa créditos), Langfuse sandbox, PostHog dev project. ZapSign usa webhook simulado.

---

## 1. Pré-requisitos

| Ferramenta | Versão mínima | Instalação | Verificação |
|---|---|---|---|
| **Node.js** | 22.0.0 | `https://nodejs.org` ou `nvm install 22` | `node -v` → `v22.x.x` |
| **pnpm** | 9.0.0 | `npm install -g pnpm@9` | `pnpm -v` → `9.x.x` |
| **Docker** | 25.0.0 | `https://docs.docker.com/get-docker` | `docker -v` → `25.x.x` |
| **Docker Compose** | v2.0.0 | Incluído no Docker Desktop | `docker compose version` → `v2.x.x` |
| **Git** | 2.40.0 | `https://git-scm.com` | `git --version` → `2.40.x` |
| **Supabase CLI** | 1.x | `npm install -g supabase` | `supabase --version` → `1.x.x` |

> ⚙️ **Obrigatório:** use `pnpm` — não `npm` nem `yarn`. O projeto usa `pnpm-lock.yaml` e workspaces pnpm. Usar `npm install` quebrará o lockfile e causará conflitos de dependência.

---

## 2. Clone e Instalação

```bash
# 1. Clonar o repositório
git clone https://github.com/repasse-seguro/ai-dani-cedente.git
cd ai-dani-cedente

# 2. Verificar a branch principal
git status
# Saída esperada: "On branch main, nothing to commit"

# 3. Instalar dependências
pnpm install
# Saída esperada: "Lockfile is up to date, resolution step is skipped"
# Dependências instaladas em node_modules/

# 4. Configurar Git hooks (Husky)
pnpm prepare
# Saída esperada: "husky - Git hooks installed"

# 5. Copiar .env de desenvolvimento
cp .env.example .env.local
# Editar .env.local com as credenciais de desenvolvimento (ver Seção 6)
```

> 💡 **Dica:** se `pnpm install` falhar com erro de peer dependencies, execute `pnpm install --no-frozen-lockfile` e abra uma issue descrevendo as dependências conflitantes.

---

## 3. Docker Compose

O ambiente local usa 3 serviços Docker: PostgreSQL (simulando Supabase), Redis e RabbitMQ. **OpenAI, Langfuse e PostHog são reais** — configure as credenciais em `.env.local`.

### 3.1 docker-compose.yml

```yaml
version: "3.9"

services:
  postgres:
    image: supabase/postgres:15.1.0.117
    container_name: dani_postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: dani_cedente_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres_dev_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./prisma/rls:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d dani_cedente_dev"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - dani_network

  redis:
    image: redis:7.4-alpine
    container_name: dani_redis
    ports:
      - "6379:6379"
    command: redis-server --requirepass redis_dev_password --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redis_dev_password", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - dani_network

  rabbitmq:
    image: rabbitmq:4-management-alpine
    container_name: dani_rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: rabbitmq_user
      RABBITMQ_DEFAULT_PASS: rabbitmq_dev_password
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 15s
      timeout: 10s
      retries: 5
    networks:
      - dani_network

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:

networks:
  dani_network:
    driver: bridge
```

### 3.2 Tabela de Serviços Locais

| Serviço | Imagem | Porta local | Health check endpoint | Credenciais padrão |
|---|---|---|---|---|
| PostgreSQL | `supabase/postgres:15.1.0.117` | `5432` | `pg_isready -U postgres` | `postgres / postgres_dev_password` |
| Redis | `redis:7.4-alpine` | `6379` | `redis-cli ping` | password: `redis_dev_password` |
| RabbitMQ | `rabbitmq:4-management` | `5672` (AMQP) / `15672` (UI) | `rabbitmq-diagnostics check_running` | `rabbitmq_user / rabbitmq_dev_password` |

> 💡 **RabbitMQ Management UI:** acesse `http://localhost:15672` com as credenciais acima para visualizar filas, consumers e mensagens.

---

## 4. Clone e Instalação (Anti-exemplos)

> 🔴 **Anti-exemplos — nunca faça isso:**

**❌ Instrução genérica sem comando:**
```
Instale as dependências do projeto.
```
✅ **Correto:**
```bash
pnpm install
# Saída esperada: "Lockfile is up to date, resolution step is skipped"
```

**❌ `.env.example` incompleto:**
```
OPENAI_API_KEY=
DATABASE_URL=
```
✅ **Correto:** ver Seção 6 — `.env.example` completo com comentários por seção.

**❌ Docker Compose sem health check:**
```yaml
services:
  postgres:
    image: postgres:15
    ports:
      - "5432:5432"
```
✅ **Correto:** ver Seção 3.1 — cada serviço tem `healthcheck` com `test`, `interval`, `timeout` e `retries`.

**❌ Troubleshooting vago:**
```
Se o banco não conectar, verifique os logs.
```
✅ **Correto:** ver Seção 14 — cada cenário tem erro esperado, causa provável e ação recomendada.

---

## 5. Subir o Projeto

```bash
# 1. Subir serviços Docker
docker compose up -d
# Saída esperada: 3 containers "Started" (postgres, redis, rabbitmq)

# 2. Aguardar health checks (aguarda todos os serviços ficarem healthy)
docker compose ps
# Saída esperada: todos os STATUS = "healthy"

# 3. Rodar migrations Prisma
pnpm prisma migrate dev
# Saída esperada: "All migrations have been applied."

# 4. Aplicar RLS policies e pgvector index
pnpm prisma db execute --file ./prisma/rls/policies.sql
pnpm prisma db execute --file ./prisma/rls/indexes.sql
# Saída esperada: nenhum erro

# 5. Rodar seed de desenvolvimento (opcional)
pnpm prisma db seed
# Saída esperada: "Seed completed. 2 cedentes, 1 oportunidade, 1 proposta criados."

# 6. Iniciar o servidor NestJS em modo watch
pnpm start:dev
# Saída esperada:
# [NestFactory] Starting Nest application...
# [RoutesResolver] AuthController {/api/v1/auth}
# [RoutesResolver] ChatController {/api/v1/chat}
# ...
# Application is running on: http://localhost:3001/api/v1
```

> ⚙️ **Porta padrão:** `3001`. Configurável via `PORT` no `.env.local`.

---

## 6. Variáveis de Ambiente (.env.example)

```bash
# =============================================================================
# AI-Dani-Cedente — .env.example
# Copie para .env.local e preencha com as credenciais de desenvolvimento.
# NUNCA commite .env.local — está no .gitignore.
# =============================================================================

# --- APP ---
NODE_ENV=development
PORT=3001
API_PREFIX=api/v1
LOG_LEVEL=debug
# LOG_LEVEL: error | warn | info | debug | verbose

# --- DATABASE (Supabase PostgreSQL) ---
DATABASE_URL=postgresql://postgres:postgres_dev_password@localhost:5432/dani_cedente_dev
# Para staging/prod: use a URL de conexão do Supabase project
# Formato: postgresql://postgres:[PASSWORD]@db.[PROJECT_REF].supabase.co:5432/postgres

# --- SUPABASE AUTH ---
SUPABASE_URL=https://[PROJECT_REF].supabase.co
# Em dev local, pode apontar para Supabase local CLI (http://localhost:54321)
SUPABASE_JWT_SECRET=super-secret-jwt-key-for-dev-only
# Chave pública RSA do Supabase para validação JWT local (RS256)
# Em dev: use o jwt_secret do supabase start

SUPABASE_SERVICE_ROLE_KEY=dev-service-role-key
# Service role key para operações de storage

# --- REDIS ---
REDIS_URL=redis://:redis_dev_password@localhost:6379
# Formato: redis://:[PASSWORD]@[HOST]:[PORT]

# --- RABBITMQ ---
RABBITMQ_URL=amqp://rabbitmq_user:rabbitmq_dev_password@localhost:5672
# Formato: amqp://[USER]:[PASSWORD]@[HOST]:[PORT]

# --- OPENAI ---
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# Obtida em: https://platform.openai.com/api-keys
# ATENÇÃO: chave de dev consome créditos reais. Use projeto separado de dev.

OPENAI_CHAT_MODEL=gpt-4-turbo
# Modelos aceitos: gpt-4-turbo | gpt-4o | gpt-4-turbo-preview
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
# Modelo de embeddings: sempre text-embedding-3-small (dimensão 1536)

# --- LANGFUSE ---
LANGFUSE_PUBLIC_KEY=pk-lf-xxxxxxxxxxxxxxxx
# Obtida em: https://cloud.langfuse.com/project/settings
LANGFUSE_SECRET_KEY=sk-lf-xxxxxxxxxxxxxxxx
LANGFUSE_HOST=https://cloud.langfuse.com
# Para self-hosted: URL da instância

# --- POSTHOG ---
POSTHOG_API_KEY=phc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# Obtida em: https://app.posthog.com/project/settings
POSTHOG_HOST=https://app.posthog.com
# Para instância EU: https://eu.posthog.com

# --- ZAPSIGN ---
ZAPSIGN_API_TOKEN=dev-zapsign-token
# Token de API do ZapSign (ambiente de sandbox)
ZAPSIGN_WEBHOOK_SECRET=dev-webhook-secret-32chars-minimum
# Segredo HMAC para validação de webhooks — mínimo 32 caracteres

# --- FALLBACK / CIRCUIT BREAKER ---
FALLBACK_ERROR_THRESHOLD_WARNING=0.10
# 10% de erros em 15min → alerta ao Admin
FALLBACK_ERROR_THRESHOLD_SHUTDOWN=0.30
# 30% de erros em 15min → desligamento automático do agente
FALLBACK_WINDOW_MINUTES=15
# Janela de cálculo do threshold em minutos

# --- RATE LIMITING ---
RATE_LIMIT_CHAT_PER_HOUR=30
# Máximo de mensagens por Cedente por hora no Chat
RATE_LIMIT_API_PER_MINUTE=100
# Máximo de requisições por IP por minuto nos demais endpoints

# --- SENTRY (opcional em dev) ---
SENTRY_DSN=
# Deixe vazio em dev para desabilitar o Sentry
SENTRY_ENVIRONMENT=development
```

---

## 7. Health Check (Validação)

Após subir o projeto, valide com:

```bash
# 1. Health check da API
curl -s http://localhost:3001/api/v1/health | jq .
# Saída esperada:
# {
#   "status": "ok",
#   "services": {
#     "database": "ok",
#     "redis": "ok",
#     "rabbitmq": "ok"
#   },
#   "version": "1.0.0",
#   "timestamp": "2026-03-23T10:00:00Z"
# }

# 2. Verificar containers saudáveis
docker compose ps
# Saída esperada: Status = "healthy" para postgres, redis, rabbitmq

# 3. Verificar migrations aplicadas
pnpm prisma migrate status
# Saída esperada: "All migrations have been applied."

# 4. Verificar filas RabbitMQ criadas
curl -s -u rabbitmq_user:rabbitmq_dev_password \
  http://localhost:15672/api/queues | jq '.[].name'
# Saída esperada:
# "rag.ingest"
# "rag.ingest.dlq"
# "notification.send"
# "notification.send.dlq"
# "escrow.events"
# "escrow.events.dlq"
```

---

## 8. Inventário de Variáveis

| Variável | Descrição | Obrigatória | Sensível | Dev default | Escopo |
|---|---|---|---|---|---|
| `NODE_ENV` | Ambiente de execução | Sim | Não | `development` | App |
| `PORT` | Porta HTTP do servidor | Sim | Não | `3001` | App |
| `API_PREFIX` | Prefixo base da API | Sim | Não | `api/v1` | App |
| `LOG_LEVEL` | Nível de log | Sim | Não | `debug` | App |
| `DATABASE_URL` | Connection string PostgreSQL | Sim | Sim | Ver `.env.example` | Database |
| `SUPABASE_URL` | URL do projeto Supabase | Sim | Não | Local CLI URL | Supabase |
| `SUPABASE_JWT_SECRET` | Segredo JWT para validação RS256 | Sim | Sim | Dev-only key | Auth |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key para Storage | Sim | Sim | Dev-only key | Storage |
| `REDIS_URL` | Connection string Redis | Sim | Sim | Ver `.env.example` | Cache |
| `RABBITMQ_URL` | AMQP connection string | Sim | Sim | Ver `.env.example` | Queue |
| `OPENAI_API_KEY` | API key OpenAI (GPT-4 + embeddings) | Sim | Sim | Chave de dev | OpenAI |
| `OPENAI_CHAT_MODEL` | Modelo de chat | Sim | Não | `gpt-4-turbo` | OpenAI |
| `OPENAI_EMBEDDING_MODEL` | Modelo de embeddings | Sim | Não | `text-embedding-3-small` | OpenAI |
| `LANGFUSE_PUBLIC_KEY` | Chave pública Langfuse | Sim | Não | Dev project key | Langfuse |
| `LANGFUSE_SECRET_KEY` | Chave secreta Langfuse | Sim | Sim | Dev project key | Langfuse |
| `LANGFUSE_HOST` | Host Langfuse | Sim | Não | `https://cloud.langfuse.com` | Langfuse |
| `POSTHOG_API_KEY` | Project API key PostHog | Sim | Sim | Dev project key | PostHog |
| `POSTHOG_HOST` | Host PostHog | Sim | Não | `https://app.posthog.com` | PostHog |
| `ZAPSIGN_API_TOKEN` | Token API ZapSign | Sim | Sim | Sandbox token | ZapSign |
| `ZAPSIGN_WEBHOOK_SECRET` | Segredo HMAC webhook | Sim | Sim | Dev secret | ZapSign |
| `FALLBACK_ERROR_THRESHOLD_WARNING` | Threshold de alerta (10%) | Sim | Não | `0.10` | Fallback |
| `FALLBACK_ERROR_THRESHOLD_SHUTDOWN` | Threshold de desligamento (30%) | Sim | Não | `0.30` | Fallback |
| `RATE_LIMIT_CHAT_PER_HOUR` | Rate limit chat (msgs/h) | Sim | Não | `30` | Rate Limit |
| `SENTRY_DSN` | DSN Sentry (vazio = desabilitado) | Não | Sim | (vazio) | Monitoring |

**Total: 24 variáveis** — 9 sensíveis, 15 não-sensíveis.

---

## 9. Regras de Nomenclatura

| Regra | Exemplo correto | Exemplo incorreto |
|---|---|---|
| UPPER_SNAKE_CASE | `OPENAI_API_KEY` | `openaiApiKey`, `openai-api-key` |
| Prefixo por serviço | `REDIS_URL`, `RABBITMQ_URL` | `CACHE_URL` (genérico) |
| Sufixo `_URL` para connection strings | `DATABASE_URL`, `REDIS_URL` | `DATABASE_HOST` + `DATABASE_PORT` separados |
| Sufixo `_KEY` para API keys | `OPENAI_API_KEY`, `LANGFUSE_SECRET_KEY` | `OPENAI_TOKEN` |
| Sufixo `_SECRET` para segredos HMAC | `ZAPSIGN_WEBHOOK_SECRET` | `ZAPSIGN_WEBHOOK_KEY` |
| Booleanos como `true`/`false` string | `FEATURE_FLAG=true` | `FEATURE_FLAG=1` |
| Decimais com ponto | `FALLBACK_THRESHOLD=0.30` | `FALLBACK_THRESHOLD=30` (ambíguo) |

---

## 10. Secrets Management

### 10.1 Por Ambiente

| Ambiente | Onde ficam os secrets | Quem acessa | Formato |
|---|---|---|---|
| **Desenvolvimento** | `.env.local` (não commitado) | Dev individual | Arquivo local |
| **Staging** | Railway Secrets (project settings) | Backend team | Variáveis de ambiente Railway |
| **Produção** | Railway Secrets + Vault (para secrets críticos) | Backend lead + DevOps | Variáveis de ambiente Railway + Vault API |

### 10.2 Regras de Acesso

- `DATABASE_URL` prod: apenas Backend Lead e DevOps.
- `OPENAI_API_KEY` prod: apenas Backend Lead.
- `SUPABASE_SERVICE_ROLE_KEY` prod: apenas Backend Lead e DevOps.
- `ZAPSIGN_API_TOKEN` prod: Backend Lead.
- Secrets de staging: Backend team (todos).

### 10.3 Como Adicionar Novo Secret

```bash
# Desenvolvimento — adicionar ao .env.local
echo "NOVO_SECRET=valor_dev" >> .env.local

# Staging/Prod — via Railway CLI
railway variables set NOVO_SECRET=valor_staging --environment staging
railway variables set NOVO_SECRET=valor_prod --environment production

# Validar que a variável foi adicionada ao ConfigurationService
# Arquivo: src/config/configuration.ts
# Adicionar à validação Joi/zod e ao export
```

---

## 11. Rotação de Secrets

### 11.1 Frequência Recomendada

| Secret | Frequência | Motivo |
|---|---|---|
| `OPENAI_API_KEY` | A cada 90 dias | Best practice OpenAI |
| `SUPABASE_SERVICE_ROLE_KEY` | A cada 90 dias | Acesso amplo ao Storage |
| `ZAPSIGN_API_TOKEN` | A cada 90 dias | Best practice ZapSign |
| `ZAPSIGN_WEBHOOK_SECRET` | A cada 90 dias | Valida integridade de webhooks |
| `LANGFUSE_SECRET_KEY` | A cada 90 dias | — |
| `POSTHOG_API_KEY` | A cada 90 dias | — |
| `SUPABASE_JWT_SECRET` | Somente em comprometimento | Rotação requer reemissão de todos os JWTs ativos |
| `DATABASE_URL` | Somente em comprometimento | Requer coordenação com Supabase |

### 11.2 Checklist de Rotação Sem Downtime

```
[ ] 1. Gerar novo valor no painel do provedor (ex: nova API key OpenAI)
[ ] 2. Adicionar nova chave ao Railway Secrets com nome temporário (ex: OPENAI_API_KEY_NEW)
[ ] 3. Atualizar o código para ler a nova chave (se necessário)
[ ] 4. Deploy em staging com nova chave — validar health check e integrações
[ ] 5. Promover para produção — deploy zero-downtime via Railway
[ ] 6. Verificar logs por 15 minutos — confirmar ausência de erros 401
[ ] 7. Revogar chave antiga no painel do provedor
[ ] 8. Renomear variável temporária para o nome definitivo
[ ] 9. Registrar rotação no log de segurança (Notion ou documento interno)
```

> 🔴 **Atenção:** nunca revogar a chave antiga antes de confirmar que o deploy com a nova chave está estável em produção.

---

## 12. Diferenças entre Ambientes

| Configuração | Desenvolvimento | Staging | Produção |
|---|---|---|---|
| `NODE_ENV` | `development` | `staging` | `production` |
| `LOG_LEVEL` | `debug` | `info` | `warn` |
| PostgreSQL | Docker local (`localhost:5432`) | Supabase staging project | Supabase prod project |
| Redis | Docker local (`localhost:6379`) | Railway Redis staging | Railway Redis prod |
| RabbitMQ | Docker local (`localhost:5672`) | Railway RabbitMQ staging | Railway RabbitMQ prod |
| OpenAI API key | Projeto de dev (créditos limitados) | Projeto de staging | Projeto de produção |
| ZapSign | Ambiente sandbox | Ambiente sandbox | Ambiente produção |
| Langfuse | Projeto dev | Projeto staging | Projeto produção |
| PostHog feature flags | `dani_cedente_enabled=true` (fixo) | Configurável | Configurável |
| Sentry DSN | Vazio (desabilitado) | DSN staging | DSN produção |
| Rate limit chat | 30 msg/h | 30 msg/h | 30 msg/h |
| `FALLBACK_ERROR_THRESHOLD_SHUTDOWN` | `0.30` | `0.30` | `0.30` |
| Hot reload | Sim (`pnpm start:dev`) | Não | Não |
| Seed de dados | Disponível (`pnpm prisma db seed`) | Não | Não |
| Migrations automáticas | Sim (`migrate dev`) | Manual (`migrate deploy`) | Manual (`migrate deploy`) |

---

## 13. Troubleshooting

### Cenário 1: `pnpm install` falha com "peer dependencies conflict"

**Erro esperado:**
```
ERR_PNPM_PEER_DEP_ISSUES Unmet peer dependencies
```
**Causa:** versão de pacote incompatível com peer dependency declarada.

**Ação:**
```bash
pnpm install --no-strict-peer-dependencies
# Se persistir, verifique a versão do Node.js: node -v (deve ser 22+)
nvm use 22
pnpm install
```

---

### Cenário 2: PostgreSQL container não sobe (porta 5432 em uso)

**Erro esperado:**
```
Error response from daemon: Ports are not available: exposing port TCP 0.0.0.0:5432 -> 0.0.0.0:0: listen tcp4 0.0.0.0:5432: bind: address already in use
```
**Causa:** outro serviço (ex: PostgreSQL local nativo) está usando a porta 5432.

**Ação:**
```bash
# Identificar o processo na porta
lsof -i :5432
# Parar o PostgreSQL nativo (macOS)
brew services stop postgresql
# Ou alterar a porta no docker-compose.yml para "5433:5432"
# E atualizar DATABASE_URL para porta 5433
```

---

### Cenário 3: `pnpm prisma migrate dev` falha com "Can't reach database server"

**Erro esperado:**
```
Error: P1001: Can't reach database server at `localhost:5432`
```
**Causa:** container PostgreSQL não está healthy ou `.env.local` com `DATABASE_URL` incorreta.

**Ação:**
```bash
# 1. Verificar se o container está healthy
docker compose ps
# Se status != "healthy", aguardar e tentar novamente:
docker compose logs postgres

# 2. Verificar DATABASE_URL no .env.local
grep DATABASE_URL .env.local
# Deve ser: postgresql://postgres:postgres_dev_password@localhost:5432/dani_cedente_dev
```

---

### Cenário 4: Redis retorna "NOAUTH Authentication required"

**Erro esperado:**
```
ReplyError: NOAUTH Authentication required.
```
**Causa:** `REDIS_URL` sem senha ou senha incorreta.

**Ação:**
```bash
# Verificar REDIS_URL
grep REDIS_URL .env.local
# Deve ser: redis://:redis_dev_password@localhost:6379
# Nota: os dois pontos (:) antes da senha são obrigatórios quando não há usuário
```

---

### Cenário 5: Swagger / OpenAPI não abre em `localhost:3001/api/docs`

**Erro esperado:** `404 Not Found` ao acessar `/api/docs`.

**Causa:** Swagger só está habilitado em `NODE_ENV !== production`.

**Ação:**
```bash
# Verificar NODE_ENV no .env.local
grep NODE_ENV .env.local
# Deve ser: NODE_ENV=development
# Reiniciar o servidor após alteração:
pnpm start:dev
```

---

### Cenário 6: `ZAPSIGN_WEBHOOK_SECRET` muito curto

**Erro esperado:**
```
Error: ZAPSIGN_WEBHOOK_SECRET must be at least 32 characters
```
**Causa:** validação Joi/zod no `ConfigurationService`.

**Ação:**
```bash
# Gerar secret seguro com 32+ caracteres
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
# Copiar o output para ZAPSIGN_WEBHOOK_SECRET no .env.local
```

---

### Cenário 7: RabbitMQ filas não criadas após subir o projeto

**Sintoma:** GET `http://localhost:15672/api/queues` retorna array vazio.

**Causa:** as filas são criadas ao conectar os consumers NestJS — o servidor precisa estar rodando.

**Ação:**
```bash
# 1. Garantir que o servidor NestJS está rodando
pnpm start:dev

# 2. Aguardar a mensagem nos logs:
# [RabbitMQModule] Connected to amqp://localhost:5672
# [RabbitMQModule] Queues created: rag.ingest, notification.send, escrow.events

# 3. Verificar novamente
curl -s -u rabbitmq_user:rabbitmq_dev_password \
  http://localhost:15672/api/queues | jq '.[].name'
```

---

## 14. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Vault para secrets críticos em prod | `[DECISÃO AUTÔNOMA]` | 10.1 | Railway Secrets suficiente para Fase 1. HashiCorp Vault adicionado como opção para prod mas não obrigatório no lançamento. Alternativa descartada: apenas `.env` no servidor — sem auditoria. | Médio | DevOps | Fase 2 |
| Supabase CLI local para dev | `[DECISÃO AUTÔNOMA]` | 3 | Docker supabase/postgres usado em vez de supabase CLI completo — reduz complexidade de setup local. Supabase Auth em dev aponta para projeto real. Alternativa descartada: supabase start completo — overhead de 4+ containers extras. | Baixo | Backend | Decidido |
| ZapSign sandbox URL | `[DECISÃO AUTÔNOMA]` | 6 | Assumido que ZapSign tem ambiente sandbox separado do produção. Confirmar URL de sandbox com equipe ZapSign antes do Go-Live. | Médio | Backend | Pendente confirmação |
