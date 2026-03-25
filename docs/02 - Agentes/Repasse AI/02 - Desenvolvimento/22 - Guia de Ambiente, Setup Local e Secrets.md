# Repasse AI
## 22 — Guia de Ambiente, Setup Local e Secrets

| Campo | Valor |
|---|---|
| **Destinatário** | DevOps e Engenharia |
| **Escopo** | Setup local completo, variáveis de ambiente, secrets e troubleshooting operacional |
| **Módulo** | Repasse AI |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |
| **Status** | Aprovado |

---

> 📌 **TL;DR**
>
> - **Pré-requisitos:** Node.js 22+, Docker 24+, Docker Compose 2.24+, Git, pnpm 9+.
> - **Tempo estimado de setup:** < 15 minutos em máquina com dependências instaladas.
> - **Serviços locais via Docker:** 3 (PostgreSQL 17, Redis 7.4, RabbitMQ 4).
> - **Total de variáveis de ambiente:** 18 variáveis (9 sensíveis / 9 não sensíveis).
> - **Serviços externos em dev:** apenas Langfuse e OpenAI requerem credenciais reais — sem mock local disponível.
> - **Regra de secrets:** nunca em código-fonte; nunca em `.env` commitado; sempre via Railway Secrets em produção.
> - **EvolutionAPI em dev:** opcional — vinculação WhatsApp pode ser mockada via flag `FEATURE_WHATSAPP_MOCK=true`.

---

## 1. Pré-requisitos

Antes de iniciar o setup, garanta que todas as ferramentas abaixo estão instaladas e nas versões corretas.

| Ferramenta | Versão Mínima | Instalação | Verificação |
|---|---|---|---|
| **Node.js** | 22.x LTS | [nodejs.org/download](https://nodejs.org/en/download) ou `nvm install 22` | `node --version` → `v22.x.x` |
| **pnpm** | 9.x | `npm install -g pnpm@9` | `pnpm --version` → `9.x.x` |
| **Docker** | 24.x | [docs.docker.com/get-docker](https://docs.docker.com/get-docker/) | `docker --version` → `Docker version 24.x.x` |
| **Docker Compose** | 2.24+ | Incluído no Docker Desktop | `docker compose version` → `Docker Compose version v2.24.x` |
| **Git** | 2.40+ | [git-scm.com](https://git-scm.com/downloads) | `git --version` → `git version 2.4x.x` |
| **Turbo CLI** | latest | `pnpm add -g turbo` | `turbo --version` → `x.x.x` |

> 💡 **Recomendação:** use `nvm` (Node Version Manager) para gerenciar versões do Node.js. O repositório inclui `.nvmrc` com a versão correta — basta rodar `nvm use` na raiz.

---

## 2. Clone e Instalação

### 2.1 Clonar o Repositório

```bash
git clone https://github.com/repasse-seguro/repasse-ai.git
cd repasse-ai
```

**Output esperado:**
```
Cloning into 'repasse-ai'...
remote: Counting objects: ...
```

### 2.2 Instalar Dependências

```bash
pnpm install
```

**Output esperado:**
```
Packages: +XXX
Progress: resolved XXX, reused XXX, downloaded X, added XXX, done
```

> 🔴 **Erro comum:** `ERR_PNPM_PEER_DEP_ISSUES` — instale as dependências com `pnpm install --shamefully-hoist`. Se persistir, verifique se o Node.js é v22+ via `node --version`.

### 2.3 Configurar Git Hooks (Husky)

```bash
pnpm exec husky install
```

**Output esperado:**
```
husky - Git hooks installed
```

### 2.4 Copiar o `.env.example`

```bash
cp apps/ai/.env.example apps/ai/.env
```

Edite o arquivo `apps/ai/.env` com as credenciais necessárias (ver Seção 6).

---

## 3. Docker Compose

### 3.1 Serviços Locais

O Docker Compose sobe PostgreSQL, Redis e RabbitMQ localmente. **Serviços externos** (OpenAI, Langfuse, EvolutionAPI) são acessados com credenciais reais — não são mockados por padrão.

```yaml
# docker-compose.yml (raiz do repositório)
version: '3.9'

services:
  postgres:
    image: supabase/postgres:15.8.1.038
    container_name: repasse-ai-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: repasse_ai_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "54322:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./prisma/rls/policies.sql:/docker-entrypoint-initdb.d/01-policies.sql
      - ./prisma/rls/indexes.sql:/docker-entrypoint-initdb.d/02-indexes.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d repasse_ai_dev"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7.4-alpine
    container_name: repasse-ai-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --requirepass repasse_ai_dev_password
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "repasse_ai_dev_password", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  rabbitmq:
    image: rabbitmq:4-management-alpine
    container_name: repasse-ai-rabbitmq
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: repasse_ai
      RABBITMQ_DEFAULT_PASS: repasse_ai_dev
      RABBITMQ_DEFAULT_VHOST: repasse_ai
    ports:
      - "5672:5672"       # AMQP
      - "15672:15672"     # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s

volumes:
  postgres_data:
  rabbitmq_data:
```

### 3.2 Tabela de Serviços

| Serviço | Imagem | Porta Local | Porta Container | Health Check | Management UI |
|---|---|---|---|---|---|
| PostgreSQL 17 | `supabase/postgres:15.8.1.038` | `54322` | `5432` | `pg_isready` | — |
| Redis 7.4 | `redis:7.4-alpine` | `6379` | `6379` | `redis-cli ping` | — |
| RabbitMQ 4 | `rabbitmq:4-management-alpine` | `5672` (AMQP) | `5672` | `check_port_connectivity` | `http://localhost:15672` |

> ⚙️ **Nota:** A imagem `supabase/postgres` é usada para ter `pgvector` pré-instalado e habilitar extensões compatíveis com Supabase em produção. A versão local não inclui Auth/Realtime do Supabase — somente o PostgreSQL com pgvector.

> 💡 **RabbitMQ Management UI:** acesse `http://localhost:15672` com `repasse_ai` / `repasse_ai_dev` para visualizar filas, exchanges e mensagens em dev.

> 🔴 **Anti-exemplo — Docker Compose sem health check:**
> ```yaml
> # ❌ ERRADO — sem health check
> postgres:
>   image: postgres:17
>   ports:
>     - "5432:5432"
> ```
> ```yaml
> # ✅ CORRETO — com health check
> postgres:
>   image: supabase/postgres:15.8.1.038
>   healthcheck:
>     test: ["CMD-SHELL", "pg_isready -U postgres -d repasse_ai_dev"]
>     interval: 10s
>     retries: 5
> ```

---

## 4. Subir os Serviços Docker

```bash
docker compose up -d
```

**Output esperado:**
```
[+] Running 3/3
 ✔ Container repasse-ai-postgres   Started
 ✔ Container repasse-ai-redis      Started
 ✔ Container repasse-ai-rabbitmq   Started
```

Aguardar health checks ficarem `healthy`:

```bash
docker compose ps
```

**Output esperado:**
```
NAME                      STATUS                    PORTS
repasse-ai-postgres       Up X seconds (healthy)    0.0.0.0:54322->5432/tcp
repasse-ai-redis          Up X seconds (healthy)    0.0.0.0:6379->6379/tcp
repasse-ai-rabbitmq       Up X seconds (healthy)    0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp
```

> 🔴 **Erro comum:** `(health: starting)` por mais de 60s — execute `docker compose logs postgres` para ver o erro. Causa mais comum: porta `54322` já em uso por outro processo PostgreSQL.

---

## 5. Configurar o Banco de Dados

### 5.1 Rodar Migrations Prisma

```bash
cd apps/ai
pnpm prisma migrate dev --name init
```

**Output esperado:**
```
The following migration(s) have been applied:
  migrations/
    └─ 20260322000000_init/
         └─ migration.sql

Your database is now in sync with your schema.
```

### 5.2 Rodar o Seed

```bash
pnpm prisma db seed
```

**Output esperado:**
```
🌱 Seeding AgentConfiguration...
✅ 9 AgentConfiguration records inserted/updated.
Seeding complete.
```

### 5.3 Verificar o Banco

```bash
pnpm prisma studio
```

Acesse `http://localhost:5555`. Confirme que as 9 entradas de `AgentConfiguration` estão presentes.

---

## 6. Variáveis de Ambiente (`.env.example`)

```bash
# apps/ai/.env.example
# ============================================================
# REPASSE AI — Variáveis de Ambiente
# Copie este arquivo para apps/ai/.env e preencha os valores.
# NUNCA commite o arquivo .env com valores reais.
# ============================================================

# ------------------------------------------------------------
# APP
# ------------------------------------------------------------
NODE_ENV=development
PORT=3000
API_PREFIX=repasse-ai/v1

# Feature flags (true/false)
FEATURE_LLM_ENABLED=true
FEATURE_WHATSAPP_MOCK=false   # true = mock EvolutionAPI em dev

# ------------------------------------------------------------
# DATABASE — PostgreSQL via Supabase (local: Docker; prod: Supabase)
# ------------------------------------------------------------
# Local dev: postgresql://postgres:postgres@localhost:54322/repasse_ai_dev
# Prod: string de conexão do Supabase (connection pooler)
DATABASE_URL=postgresql://postgres:postgres@localhost:54322/repasse_ai_dev

# ------------------------------------------------------------
# AUTH — Supabase (apenas validação JWT em dev com mock)
# ------------------------------------------------------------
SUPABASE_URL=https://your-project-ref.supabase.co
SUPABASE_ANON_KEY=your_supabase_anon_key_here
SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_key_here

# Para dev local sem Supabase: use JWT_DEV_MODE=true (assina JWT com JWT_DEV_SECRET)
JWT_DEV_MODE=true
JWT_DEV_SECRET=dev_secret_32_chars_minimum_here!!

# ------------------------------------------------------------
# REDIS — Cache LLM + Rate Limiting (local: Docker; prod: Upstash)
# ------------------------------------------------------------
# Local dev: redis://:repasse_ai_dev_password@localhost:6379
# Prod: redis://default:password@host.upstash.io:6379 (TLS obrigatório)
REDIS_URL=redis://:repasse_ai_dev_password@localhost:6379

# ------------------------------------------------------------
# RABBITMQ — Filas assíncronas (local: Docker; prod: CloudAMQP)
# ------------------------------------------------------------
# Local dev: amqp://repasse_ai:repasse_ai_dev@localhost:5672/repasse_ai
# Prod: amqps://user:pass@host.cloudamqp.com/vhost (TLS obrigatório)
AMQP_URL=amqp://repasse_ai:repasse_ai_dev@localhost:5672/repasse_ai

# ------------------------------------------------------------
# OPENAI — LLM + Embeddings (SENSÍVEL — requer conta OpenAI)
# ------------------------------------------------------------
OPENAI_API_KEY=sk-proj-your_openai_api_key_here
OPENAI_MODEL=gpt-4o
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
OPENAI_MAX_TOKENS=4096
OPENAI_TEMPERATURE=0.1

# ------------------------------------------------------------
# LANGFUSE — Observabilidade de IA (SENSÍVEL — requer conta Langfuse)
# ------------------------------------------------------------
LANGFUSE_PUBLIC_KEY=pk-lf-your_langfuse_public_key_here
LANGFUSE_SECRET_KEY=sk-lf-your_langfuse_secret_key_here
LANGFUSE_BASE_URL=https://cloud.langfuse.com

# ------------------------------------------------------------
# EVOLUTIONAPI — WhatsApp Business (SENSÍVEL — opcional em dev)
# ------------------------------------------------------------
# Deixe em branco se FEATURE_WHATSAPP_MOCK=true
EVOLUTION_API_URL=https://your-evolution-api-host.com
EVOLUTION_API_KEY=your_evolution_api_key_here
EVOLUTION_INSTANCE=repasse-ai-dev
EVOLUTION_WEBHOOK_SECRET=your_webhook_secret_32_chars_min_here

# ------------------------------------------------------------
# SENTRY — Error Tracking
# ------------------------------------------------------------
SENTRY_DSN=https://your_sentry_dsn_here@sentry.io/project_id
```

> 🔴 **Anti-exemplo — `.env.example` incompleto:**
> ```bash
> # ❌ ERRADO — sem comentários e variáveis faltando
> DATABASE_URL=
> OPENAI_API_KEY=
> ```
> ```bash
> # ✅ CORRETO — comentado, com valores de dev e descrição de cada variável
> # DATABASE_URL — String de conexão PostgreSQL (local: Docker; prod: Supabase Pooler)
> DATABASE_URL=postgresql://postgres:postgres@localhost:54322/repasse_ai_dev
> ```

---

## 7. Subir o Projeto

### 7.1 Modo Desenvolvimento (com hot reload)

```bash
cd apps/ai
pnpm start:dev
```

**Output esperado:**
```
[NestApplication] Application is running on: http://localhost:3000/repasse-ai/v1
[NestApplication] Swagger UI: http://localhost:3000/docs
```

### 7.2 Modo Produção (local)

```bash
pnpm build
pnpm start:prod
```

---

## 8. Health Check (Validação)

### 8.1 Verificar Health do Serviço

```bash
curl http://localhost:3000/repasse-ai/v1/health
```

**Output esperado (todos saudáveis):**
```json
{
  "data": {
    "status": "healthy",
    "version": "1.0.0",
    "timestamp": "2026-03-22T00:00:00-03:00",
    "dependencies": {
      "database": "healthy",
      "redis": "healthy",
      "rabbitmq": "healthy",
      "llm": "healthy"
    }
  }
}
```

**Output esperado (LLM degradado — sem `OPENAI_API_KEY` válida):**
```json
{
  "data": {
    "status": "degraded",
    "dependencies": {
      "database": "healthy",
      "redis": "healthy",
      "rabbitmq": "healthy",
      "llm": "degraded"
    }
  }
}
```

### 8.2 Verificar Swagger UI

Acesse `http://localhost:3000/docs` no browser. Deve exibir a documentação interativa de todos os 22 endpoints.

### 8.3 Verificar Prisma

```bash
cd apps/ai && pnpm prisma studio
```

Acesse `http://localhost:5555` e confirme: tabelas criadas, 9 registros de `AgentConfiguration` presentes.

---

## 9. Inventário de Variáveis

| Variável | Seção | Tipo | Obrigatório | Sensível | Valor Dev (Exemplo) |
|---|---|---|---|---|---|
| `NODE_ENV` | App | string | Sim | Não | `development` |
| `PORT` | App | integer | Sim | Não | `3000` |
| `API_PREFIX` | App | string | Sim | Não | `repasse-ai/v1` |
| `FEATURE_LLM_ENABLED` | Feature Flag | boolean | Sim | Não | `true` |
| `FEATURE_WHATSAPP_MOCK` | Feature Flag | boolean | Sim | Não | `false` |
| `DATABASE_URL` | Database | string (URL) | Sim | **Sim** | `postgresql://postgres:postgres@localhost:54322/repasse_ai_dev` |
| `SUPABASE_URL` | Auth | string (URL) | Prod | **Sim** | `https://ref.supabase.co` |
| `SUPABASE_ANON_KEY` | Auth | string | Prod | **Sim** | `eyJ...` |
| `SUPABASE_SERVICE_ROLE_KEY` | Auth | string | Prod | **Sim** | `eyJ...` |
| `JWT_DEV_MODE` | Auth (dev) | boolean | Dev | Não | `true` |
| `JWT_DEV_SECRET` | Auth (dev) | string (≥32 chars) | Dev | **Sim** | `dev_secret_32_chars_minimum_here!!` |
| `REDIS_URL` | Cache | string (URL) | Sim | **Sim** | `redis://:password@localhost:6379` |
| `AMQP_URL` | Queue | string (URL) | Sim | **Sim** | `amqp://user:pass@localhost:5672/vhost` |
| `OPENAI_API_KEY` | OpenAI | string | Sim | **Sim** | `sk-proj-...` |
| `OPENAI_MODEL` | OpenAI | string | Sim | Não | `gpt-4o` |
| `OPENAI_EMBEDDING_MODEL` | OpenAI | string | Sim | Não | `text-embedding-3-small` |
| `LANGFUSE_PUBLIC_KEY` | Langfuse | string | Sim | **Sim** | `pk-lf-...` |
| `LANGFUSE_SECRET_KEY` | Langfuse | string | Sim | **Sim** | `sk-lf-...` |
| `EVOLUTION_API_URL` | WhatsApp | string (URL) | Condicional* | Não | `https://evolution.example.com` |
| `EVOLUTION_API_KEY` | WhatsApp | string | Condicional* | **Sim** | `your_api_key` |
| `EVOLUTION_WEBHOOK_SECRET` | WhatsApp | string (≥32 chars) | Condicional* | **Sim** | `secret_32_chars_minimum` |
| `SENTRY_DSN` | Monitoring | string (URL) | Não | Não | `https://...@sentry.io/...` |

*Condicional: obrigatório se `FEATURE_WHATSAPP_MOCK=false`.

---

## 10. Regras de Nomenclatura

| Regra | Padrão | Exemplo |
|---|---|---|
| Formato | `SCREAMING_SNAKE_CASE` | `OPENAI_API_KEY`, `DATABASE_URL` |
| Prefixo por serviço externo | `{SERVIÇO}_` | `OPENAI_`, `REDIS_`, `AMQP_`, `EVOLUTION_` |
| Feature flags | `FEATURE_` | `FEATURE_LLM_ENABLED`, `FEATURE_WHATSAPP_MOCK` |
| Secrets com `KEY` ou `SECRET` no nome | Para facilitar grep de segurança | `OPENAI_API_KEY`, `EVOLUTION_WEBHOOK_SECRET` |
| Sem espaços | Nunca usar espaços | ❌ `OPEN AI_KEY` → ✅ `OPENAI_API_KEY` |
| Sem aspas no valor do `.env` | Valores sem aspas (exceto com espaços) | `DATABASE_URL=postgresql://...` (sem aspas) |

---

## 11. Secrets Management

### 11.1 Onde Ficam os Secrets

| Ambiente | Localização | Acesso |
|---|---|---|
| **Local Dev** | `apps/ai/.env` (não commitado) | Dev local apenas — arquivo no `.gitignore` |
| **Staging** | Railway Secrets (painel Railway) | Tech Lead + DevOps |
| **Produção** | Railway Secrets (painel Railway) | Tech Lead + DevOps (acesso restrito) |
| **CI/CD** | GitHub Secrets | `RAILWAY_TOKEN` apenas — sem credenciais de app |

### 11.2 Quem Pode Acessar Secrets de Produção

- `OPENAI_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, credenciais de banco: Tech Lead e DevOps apenas.
- Nunca conceder acesso a secrets de produção para devs fora do core team sem aprovação.

### 11.3 Como Adicionar Novo Secret

1. Adicionar ao `apps/ai/.env.example` com comentário e sem valor real.
2. Atualizar o **Inventário de Variáveis** (Seção 9 deste documento).
3. Adicionar ao Railway Secrets nos ambientes de staging e produção.
4. Comunicar ao time via PR description ou Slack `#engenharia`.

### 11.4 Verificar se `.env` está no `.gitignore`

```bash
cat .gitignore | grep .env
```

**Output esperado:**
```
apps/ai/.env
.env
.env.local
.env.*.local
```

> 🔴 **Se `.env` não estiver no `.gitignore`, adicione ANTES de qualquer commit:**
> ```bash
> echo "apps/ai/.env" >> .gitignore
> git rm --cached apps/ai/.env 2>/dev/null || true
> ```

---

## 12. Rotação de Secrets

### 12.1 Checklist de Rotação

Use este checklist sempre que rotacionar um secret:

- [ ] Gerar novo valor no painel do provedor (OpenAI, Supabase, Upstash, etc.)
- [ ] Atualizar Railway Secrets no ambiente correspondente (staging ou prod)
- [ ] **Não** remover o secret antigo até confirmar que o novo está funcionando
- [ ] Verificar health check após atualização: `curl {base_url}/health`
- [ ] Revogar o secret antigo no painel do provedor
- [ ] Atualizar o `apps/ai/.env` local (se aplicável)
- [ ] Registrar a rotação no log de operações do time

### 12.2 Rotação sem Downtime (Produção)

1. Adicionar novo secret no Railway como `{NOME}_NEW` (ex: `OPENAI_API_KEY_NEW`).
2. Atualizar o código para ler `{NOME}_NEW` em paralelo ao `{NOME}` atual.
3. Deploy e validação em staging.
4. Deploy em produção e validação via health check.
5. Remover `{NOME}` antigo e renomear `{NOME}_NEW` para `{NOME}`.
6. Revogar chave antiga no provedor.

> ⚙️ **Para OpenAI especificamente:** rotacionar `OPENAI_API_KEY` invalida o cache de LLM (exact cache usa hash de `{model}+{prompt_version}+{input}`). O cache semântico não é invalidado. Latência temporariamente mais alta nas primeiras horas após rotação.

---

## 13. Diferenças entre Ambientes

| Configuração | Development | Staging | Production |
|---|---|---|---|
| PostgreSQL | Docker local (porta 54322) | Supabase (projeto staging) | Supabase (projeto prod) |
| Redis | Docker local (porta 6379, sem TLS) | Upstash Redis (TLS) | Upstash Redis (TLS) |
| RabbitMQ | Docker local (porta 5672, sem TLS) | CloudAMQP (TLS, `amqps://`) | CloudAMQP (TLS, `amqps://`) |
| OpenAI API | Credenciais reais (conta dev) | Credenciais reais (conta staging) | Credenciais reais (conta prod) |
| Langfuse | Credenciais reais (projeto dev) | Credenciais reais (projeto staging) | Credenciais reais (projeto prod) |
| EvolutionAPI | Mock (`FEATURE_WHATSAPP_MOCK=true`) ou instância dev | Instância staging real | Instância prod real |
| Supabase Auth | `JWT_DEV_MODE=true` com `JWT_DEV_SECRET` | JWT real do Supabase staging | JWT real do Supabase prod |
| CORS | `localhost:3001` | Domínio de staging | Domínio de produção |
| Pino log level | `debug` | `info` | `warn` |
| Sentry | Desabilitado (sem DSN) | DSN de staging | DSN de produção |
| `NODE_ENV` | `development` | `staging` | `production` |
| `FEATURE_LLM_ENABLED` | `true` | `true` | `true` |
| `FEATURE_WHATSAPP_MOCK` | `true` (recomendado) | `false` | `false` |

---

## 14. Troubleshooting

### 14.1 `ECONNREFUSED 127.0.0.1:54322` — PostgreSQL não responde

**Causa:** Container Docker do PostgreSQL não subiu ou não ficou `healthy`.

**Diagnóstico:**
```bash
docker compose ps
docker compose logs postgres
```

**Resolução:**
```bash
docker compose down
docker compose up -d postgres
docker compose ps  # aguardar status (healthy)
```

---

### 14.2 `PrismaClientInitializationError: Can't reach database server`

**Causa:** `DATABASE_URL` incorreta ou PostgreSQL não está rodando.

**Diagnóstico:**
```bash
# Testar conexão direta
psql postgresql://postgres:postgres@localhost:54322/repasse_ai_dev -c "SELECT 1"
```

**Output esperado:** `?column? \n 1`

**Resolução:** verificar se `DATABASE_URL` no `.env` aponta para a porta `54322` (não `5432`).

---

### 14.3 `ReplyError: WRONGPASS invalid username-password pair` — Redis

**Causa:** `REDIS_URL` não inclui a senha configurada no Docker Compose.

**Resolução:** verificar se `REDIS_URL=redis://:repasse_ai_dev_password@localhost:6379` (dois-pontos antes da senha é obrigatório).

```bash
# Testar conexão Redis
redis-cli -a repasse_ai_dev_password ping
```

**Output esperado:** `PONG`

---

### 14.4 `AuthError: Invalid JWT` em endpoints que requerem autenticação

**Causa:** `JWT_DEV_MODE` não configurado ou `JWT_DEV_SECRET` não bate com o token gerado.

**Diagnóstico:**
```bash
# Verificar se JWT_DEV_MODE está ativo
grep JWT_DEV apps/ai/.env
```

**Resolução para testes locais:** gerar token de teste com:
```bash
node -e "
const jwt = require('jsonwebtoken');
const token = jwt.sign(
  { sub: 'uuid-dev', cessionario_id: 'uuid-dev-cess', role: 'CESSIONARIO' },
  process.env.JWT_DEV_SECRET || 'dev_secret_32_chars_minimum_here!!',
  { expiresIn: '1h' }
);
console.log('Bearer', token);
"
```

---

### 14.5 `connect ECONNREFUSED 127.0.0.1:5672` — RabbitMQ não conecta

**Causa:** Container RabbitMQ não ficou `healthy` ainda (pode levar até 30s).

**Diagnóstico:**
```bash
docker compose ps rabbitmq
docker compose logs rabbitmq
```

**Resolução:** aguardar o container ficar `(healthy)`. Se demorar > 60s:
```bash
docker compose restart rabbitmq
```

---

### 14.6 `OpenAI API Key is invalid` — LLM degradado

**Causa:** `OPENAI_API_KEY` inválida, expirada ou sem crédito.

**Diagnóstico:**
```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY" | jq '.error // "OK"'
```

**Output esperado:** `"OK"` (ou lista de modelos). Se retornar erro, gerar nova chave em `platform.openai.com`.

---

### 14.7 `pnpm: command not found`

**Resolução:**
```bash
npm install -g pnpm@9
# Ou via corepack (Node.js 22+):
corepack enable
corepack prepare pnpm@9 --activate
```

---

### 14.8 Migrations Prisma falham com `relation "AgentConfiguration" already exists`

**Causa:** banco com dados de migrations anteriores.

**Resolução (dev only — apaga todos os dados):**
```bash
cd apps/ai
pnpm prisma migrate reset --force
pnpm prisma db seed
```

> 🔴 **NUNCA rodar `migrate reset` em staging ou produção.**

---

## 15. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| `JWT_DEV_MODE` para autenticação local | [DECISÃO AUTÔNOMA] | 6 / 13 | Em dev local sem Supabase, a validação JWT seria impossível sem conta configurada. `JWT_DEV_MODE=true` assina JWTs com `JWT_DEV_SECRET` para desenvolvimento. Alternativa descartada: mock completo do Supabase Auth (overhead desnecessário). Desabilitado em staging e produção. | P1 | Backend Lead | Decidido |
| `FEATURE_WHATSAPP_MOCK=true` como padrão em dev | [DECISÃO AUTÔNOMA] | 6 / 13 | EvolutionAPI requer instância WhatsApp Business configurada — setup complexo para dev local. Flag `FEATURE_WHATSAPP_MOCK=true` substitui chamadas reais por respostas mockadas. Alternativa descartada: exigir EvolutionAPI para todos os devs (bloqueia onboarding). | P1 | Backend Lead | Decidido |

---

*Próximo documento do pipeline: D18 — Fluxos de Autenticação OAuth.*
