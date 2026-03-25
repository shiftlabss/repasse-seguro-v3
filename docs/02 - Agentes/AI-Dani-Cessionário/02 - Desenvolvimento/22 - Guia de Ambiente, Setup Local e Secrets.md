# 22 - Guia de Ambiente, Setup Local e Secrets

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 22 - Guia de Ambiente, Setup Local e Secrets | v1.0 | 23/03/2026 | Claude Code Desktop | Aprovado |

---

> 📌 **TL;DR**
>
> - **Pré-requisitos:** Node.js ≥ 20 LTS, pnpm ≥ 9, Docker Desktop ≥ 25, Git ≥ 2.40, Supabase CLI ≥ 1.x
> - **Tempo estimado:** < 15 minutos do clone ao `pnpm dev` com todos os serviços rodando
> - **Serviços:** 4 (PostgreSQL via Supabase local, Redis, RabbitMQ, API NestJS + Web Vite)
> - **Total de variáveis:** 28 (17 sensíveis, 11 não-sensíveis)
> - **Regra de secrets:** Nenhum secret real no repositório. Arquivo `.env` no `.gitignore`. Secrets de produção apenas no Vault do CI/CD (GitHub Actions Secrets).
> - **Isolamento obrigatório:** toda variável de produção difere de dev. `DATABASE_URL` de produção nunca em `.env.local`.
> - **Validação:** ao final do setup, execute `pnpm health` — todos os serviços devem retornar `OK`.

---

## 1. Pré-requisitos

Esta seção lista todas as ferramentas necessárias para rodar o projeto localmente. Instale na ordem indicada.

| Ferramenta | Versão Mínima | Instalação | Verificação |
|---|---|---|---|
| Node.js | ≥ 20 LTS | https://nodejs.org ou `nvm install 20` | `node --version` → `v20.x.x` |
| pnpm | ≥ 9.x | `npm install -g pnpm@latest` | `pnpm --version` → `9.x.x` |
| Docker Desktop | ≥ 25.x | https://www.docker.com/products/docker-desktop | `docker --version` → `Docker version 25.x.x` |
| Docker Compose | ≥ 2.x (incluso no Docker Desktop) | Incluso no Docker Desktop ≥ 20 | `docker compose version` → `Docker Compose version v2.x.x` |
| Git | ≥ 2.40 | https://git-scm.com ou via Homebrew `brew install git` | `git --version` → `git version 2.40.x` |
| Supabase CLI | ≥ 1.x | `npm install -g supabase` | `supabase --version` → `1.x.x` |
| curl ou httpie | qualquer | Pré-instalado no macOS/Linux. Windows: WSL | `curl --version` |

> ⚙️ **Atenção:** Node.js versões ímpares (19, 21, 23) são proibidas em dev e produção conforme D02. Use exclusivamente versões LTS.

**Anti-exemplo:**
```
❌ Instale o Node.js e as dependências necessárias.
```
```
✅ Execute:
   nvm install 20
   nvm use 20
   node --version   # deve exibir v20.x.x
```

---

## 2. Clone e Instalação

Esta seção cobre o clone do repositório, instalação de dependências e configuração dos git hooks.

### 2.1 Clone

```bash
git clone https://github.com/repasse-seguro/dani-cessionario.git
cd dani-cessionario
```

Output esperado:
```
Cloning into 'dani-cessionario'...
remote: Enumerating objects: 1234, done.
...
Resolving deltas: 100% (xxx/xxx), done.
```

### 2.2 Instalação de Dependências

```bash
pnpm install
```

Output esperado:
```
Lockfile is up to date, resolution step is skipped
Packages: +XXX
Progress: resolved XXX, reused XXX, downloaded 0, added XXX, done
```

> 💡 O monorepo usa pnpm workspaces + Turborepo. O comando `pnpm install` na raiz instala dependências de todos os workspaces: `apps/web`, `apps/api`, `packages/shared-types`, `packages/design-tokens`.

### 2.3 Configuração dos Git Hooks (Husky)

```bash
pnpm prepare
```

Output esperado:
```
husky - Git hooks installed
```

Hooks instalados:
- `pre-commit`: lint-staged (ESLint + Prettier)
- `commit-msg`: validação de Conventional Commits

### 2.4 Cópia do .env

```bash
cp .env.example .env
```

> ⚙️ **Obrigatório:** edite o `.env` com os valores locais antes de subir os serviços. Veja seção 6 para referência completa de variáveis.

---

## 3. Docker Compose

Esta seção define os serviços de infraestrutura local. O Docker Compose sobe PostgreSQL (via Supabase local), Redis e RabbitMQ. A API NestJS e o frontend Vite rodam diretamente via `pnpm dev` (fora do Docker em dev).

### 3.1 Tabela de Serviços

| Serviço | Imagem | Porta Local | Health Check Endpoint | Credenciais Padrão (dev) |
|---|---|---|---|---|
| postgres | `supabase/postgres:15.x` | 5432 | `pg_isready -U postgres` | user: `postgres` / pass: `postgres` |
| redis | `redis:7-alpine` | 6379 | `redis-cli ping` → `PONG` | sem auth em dev |
| rabbitmq | `rabbitmq:3.12-management-alpine` | 5672 (AMQP) / 15672 (UI) | `rabbitmq-diagnostics ping` | user: `guest` / pass: `guest` |
| supabase-studio | `supabase/studio:latest` | 54323 | HTTP GET `/` → 200 | — |

### 3.2 docker-compose.yml

```yaml
version: "3.9"

services:
  postgres:
    image: supabase/postgres:15.1.0.117
    container_name: dani_postgres
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: dani_dev
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d dani_dev"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7-alpine
    container_name: dani_redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: dani_rabbitmq
    restart: unless-stopped
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
      RABBITMQ_DEFAULT_VHOST: /
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s

  supabase-studio:
    image: supabase/studio:latest
    container_name: dani_supabase_studio
    restart: unless-stopped
    ports:
      - "54323:3000"
    environment:
      SUPABASE_URL: http://localhost:54321
      STUDIO_PG_META_URL: http://localhost:54321/pg
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
```

**Anti-exemplo:**
```yaml
❌ services:
     redis:
       image: redis
       ports:
         - "6379:6379"
```
```yaml
✅ services:
     redis:
       image: redis:7-alpine
       ports:
         - "6379:6379"
       healthcheck:
         test: ["CMD", "redis-cli", "ping"]
         interval: 10s
         timeout: 3s
         retries: 5
```

---

## 4. Clone e Setup Supabase Local

O Supabase CLI provê PostgreSQL + Auth + Storage localmente.

```bash
# Inicializa o projeto Supabase local (apenas na primeira vez)
supabase init

# Sobe os serviços Supabase locais
supabase start
```

Output esperado após `supabase start`:
```
Started supabase local development setup.

         API URL: http://localhost:54321
          DB URL: postgresql://postgres:postgres@localhost:5432/postgres
      Studio URL: http://localhost:54323
    Inbucket URL: http://localhost:54324
      anon key: eyJ...
  service_role key: eyJ...
```

> ⚙️ **Copie as chaves exibidas** para o `.env` — `SUPABASE_ANON_KEY` e `SUPABASE_SERVICE_ROLE_KEY`. As chaves locais são diferentes das de produção.

### 4.1 Aplicar Migrations

```bash
# Aplica todas as migrations via Prisma
cd apps/api
pnpm prisma migrate dev --name init
```

Output esperado:
```
Applying migration `20260323000001_init`
Database changes applied ✔
Generated Prisma Client (v5.x.x) to ./node_modules/@prisma/client
```

### 4.2 Executar Seed

```bash
pnpm prisma db seed
```

Output esperado:
```
Seeding database...
✓ 3 cessionários criados
✓ 12 oportunidades criadas
✓ 5 alertas criados
Seed completed.
```

---

## 5. Variáveis de Ambiente (.env.example)

Abaixo o arquivo `.env.example` completo com comentários. **Nenhum valor real deve ser commitado.**

```dotenv
# ============================================================
# DANI CESSIONÁRIO — .env.example
# Copie para .env e preencha com valores locais de desenvolvimento.
# NUNCA commit o arquivo .env com valores reais.
# ============================================================

# ------------------------------------------------------------
# APP
# ------------------------------------------------------------
# Ambiente de execução: development | staging | production
NODE_ENV=development

# Porta da API NestJS
API_PORT=3001

# URL base da API (usada pelo frontend)
API_BASE_URL=http://localhost:3001

# Porta do servidor Vite (frontend)
VITE_PORT=5173

# URL do frontend (usada pelo backend para CORS)
WEB_BASE_URL=http://localhost:5173

# Log level: debug | info | warn | error
LOG_LEVEL=debug

# ------------------------------------------------------------
# DATABASE — Supabase / PostgreSQL
# Sensível: sim (contém credenciais de banco)
# ------------------------------------------------------------
# URL de conexão Prisma
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/dani_dev?schema=public

# URL direta (para migrations e seeds — bypassa connection pooler)
DIRECT_URL=postgresql://postgres:postgres@localhost:5432/dani_dev?schema=public

# ------------------------------------------------------------
# SUPABASE
# Sensível: SERVICE_ROLE_KEY = sim | ANON_KEY = baixo risco
# ------------------------------------------------------------
SUPABASE_URL=http://localhost:54321
SUPABASE_ANON_KEY=<cole a anon key exibida por supabase start>
SUPABASE_SERVICE_ROLE_KEY=<cole a service_role key exibida por supabase start>

# ------------------------------------------------------------
# REDIS
# Sensível: sim (se autenticado; em dev sem senha)
# ------------------------------------------------------------
REDIS_URL=redis://localhost:6379
# Em produção use: rediss://:SENHA@host:6380
REDIS_TTL_DEFAULT=300

# ------------------------------------------------------------
# RABBITMQ
# Sensível: sim (credenciais do broker)
# ------------------------------------------------------------
RABBITMQ_URL=amqp://guest:guest@localhost:5672
# Filas obrigatórias (não alterar nomes)
RABBITMQ_QUEUE_NOTIFICACOES=dani.notificacoes
RABBITMQ_QUEUE_WHATSAPP=dani.whatsapp
RABBITMQ_QUEUE_AGENT_MONITOR=dani.agent_monitor

# ------------------------------------------------------------
# AUTENTICAÇÃO JWT
# Sensível: sim — nunca expor em frontend
# ------------------------------------------------------------
# Secret do access token (mín. 256 bits = 32 chars)
JWT_ACCESS_SECRET=<gere com: openssl rand -base64 32>
# Expiração do access token
JWT_ACCESS_EXPIRY=15m
# Secret do refresh token
JWT_REFRESH_SECRET=<gere com: openssl rand -base64 32>
# Expiração do refresh token
JWT_REFRESH_EXPIRY=7d

# ------------------------------------------------------------
# OPENAI
# Sensível: sim — chave de API paga
# ------------------------------------------------------------
OPENAI_API_KEY=<sua chave de API OpenAI — sk-...>
# Modelo padrão (D02 Stacks: gpt-4o)
OPENAI_MODEL=gpt-4o
# Timeout da chamada ao modelo (ms)
OPENAI_TIMEOUT_MS=30000
# Máximo de retentativas
OPENAI_MAX_RETRIES=3

# ------------------------------------------------------------
# LANGFUSE — Observabilidade de LLM
# Sensível: sim
# ------------------------------------------------------------
LANGFUSE_SECRET_KEY=<sua secret key Langfuse>
LANGFUSE_PUBLIC_KEY=<sua public key Langfuse>
LANGFUSE_BASEURL=https://cloud.langfuse.com
# Em dev pode ser vazio para desabilitar o tracing
# LANGFUSE_SECRET_KEY=
# LANGFUSE_PUBLIC_KEY=

# ------------------------------------------------------------
# SENTRY — Error Tracking
# Sensível: DSN não é segredo, mas recomendado em variável
# ------------------------------------------------------------
SENTRY_DSN=<seu DSN do projeto Sentry>
# Desabilitar em dev se não quiser enviar erros reais
# SENTRY_DSN=

# ------------------------------------------------------------
# EVOLUTIONAPI — WhatsApp Gateway (Fase 2)
# Sensível: sim
# ------------------------------------------------------------
# URL da instância EvolutionAPI (Fase 2)
EVOLUTION_API_URL=http://localhost:8080
EVOLUTION_API_KEY=<sua API key EvolutionAPI>
EVOLUTION_INSTANCE_NAME=dani-whatsapp

# ------------------------------------------------------------
# RATE LIMITING
# Não sensível — configuração operacional
# ------------------------------------------------------------
# Mensagens por hora por cessionario (janela deslizante)
RATE_LIMIT_WEBCHAT_MAX=30
RATE_LIMIT_WEBCHAT_WINDOW_S=3600
# OTP: tentativas por hora
RATE_LIMIT_OTP_MAX=3
RATE_LIMIT_OTP_WINDOW_S=3600
# OTP: duração do hard block (s)
RATE_LIMIT_OTP_BLOCK_S=1800
```

**Anti-exemplo:**
```dotenv
❌ # Variáveis de ambiente
DATABASE_URL=postgresql://...
OPENAI_API_KEY=sk-real-key-here-1234567890
JWT_SECRET=minhasenhassecreta
```
```dotenv
✅ # Sensível: sim — nunca expor ou commitar
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/dani_dev?schema=public
OPENAI_API_KEY=<sua chave de API OpenAI — sk-...>
JWT_ACCESS_SECRET=<gere com: openssl rand -base64 32>
```

---

## 6. Subir o Projeto (Passo a Passo)

Execute nesta sequência exata:

```bash
# 1. Sobe infraestrutura Docker (postgres, redis, rabbitmq)
docker compose up -d

# Aguarda serviços ficarem healthy (~30s)
docker compose ps

# Output esperado: todos os serviços com "healthy"

# 2. Verifica conectividade Redis
docker exec dani_redis redis-cli ping
# Output esperado: PONG

# 3. Verifica conectividade RabbitMQ
docker exec dani_rabbitmq rabbitmq-diagnostics ping
# Output esperado: Ping succeeded if [node name] is running

# 4. Aplica migrations Prisma
cd apps/api
pnpm prisma migrate dev
# Output: migrations aplicadas com sucesso

# 5. Executa seed de desenvolvimento
pnpm prisma db seed
# Output: Seed completed

# 6. Sobe todos os serviços em modo dev (raiz do monorepo)
cd ../..
pnpm dev
```

Output esperado do `pnpm dev`:
```
 DANI:API  ▸ NestJS running on http://localhost:3001
 DANI:WEB  ▸ Vite dev server running at http://localhost:5173
 TURBO     ▸ Tasks: 2 successful, 2 total
```

---

## 7. Health Check (Validação)

Após subir todos os serviços, execute as validações abaixo. Todos devem retornar `OK` ou `200`.

```bash
# API NestJS — health endpoint
curl http://localhost:3001/api/v1/health
# Output esperado: {"status":"ok","services":{"database":"ok","redis":"ok","rabbitmq":"ok"}}

# Frontend Vite
curl -o /dev/null -s -w "%{http_code}" http://localhost:5173
# Output esperado: 200

# RabbitMQ Management UI
curl -o /dev/null -s -w "%{http_code}" http://localhost:15672
# Output esperado: 200

# Supabase Studio
curl -o /dev/null -s -w "%{http_code}" http://localhost:54323
# Output esperado: 200

# Redis ping direto
redis-cli -h localhost -p 6379 ping
# Output esperado: PONG
```

> ⚙️ **Atalho:** execute `pnpm health` na raiz para rodar todos os checks acima automaticamente. Saída: `✅ API ok | ✅ Web ok | ✅ Redis ok | ✅ RabbitMQ ok`

---

## 8. Inventário de Variáveis

Tabela completa de todas as variáveis de ambiente com classificação de sensibilidade.

| Variável | Seção | Tipo | Valor Padrão (dev) | Obrigatória | Sensível | Dono |
|---|---|---|---|---|---|---|
| `NODE_ENV` | App | string | `development` | Sim | Não | Dev |
| `API_PORT` | App | number | `3001` | Sim | Não | Dev |
| `API_BASE_URL` | App | URL | `http://localhost:3001` | Sim | Não | Dev |
| `VITE_PORT` | App | number | `5173` | Não | Não | Dev |
| `WEB_BASE_URL` | App | URL | `http://localhost:5173` | Sim | Não | Dev |
| `LOG_LEVEL` | App | enum | `debug` | Não | Não | Dev |
| `DATABASE_URL` | Database | connection string | `postgresql://postgres:postgres@localhost:5432/dani_dev?schema=public` | Sim | **Sim** | DevOps |
| `DIRECT_URL` | Database | connection string | igual `DATABASE_URL` em dev | Não | **Sim** | DevOps |
| `SUPABASE_URL` | Supabase | URL | `http://localhost:54321` | Sim | Não | Dev |
| `SUPABASE_ANON_KEY` | Supabase | JWT | gerado por `supabase start` | Sim | Não | Dev |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase | JWT | gerado por `supabase start` | Sim | **Sim** | DevOps |
| `REDIS_URL` | Redis | connection string | `redis://localhost:6379` | Sim | **Sim** (prod) | DevOps |
| `REDIS_TTL_DEFAULT` | Redis | number | `300` | Não | Não | Dev |
| `RABBITMQ_URL` | RabbitMQ | connection string | `amqp://guest:guest@localhost:5672` | Sim | **Sim** | DevOps |
| `RABBITMQ_QUEUE_NOTIFICACOES` | RabbitMQ | string | `dani.notificacoes` | Sim | Não | Dev |
| `RABBITMQ_QUEUE_WHATSAPP` | RabbitMQ | string | `dani.whatsapp` | Sim (Fase 2) | Não | Dev |
| `RABBITMQ_QUEUE_AGENT_MONITOR` | RabbitMQ | string | `dani.agent_monitor` | Sim | Não | Dev |
| `JWT_ACCESS_SECRET` | Auth | string | — | Sim | **Sim** | DevOps |
| `JWT_ACCESS_EXPIRY` | Auth | duration | `15m` | Sim | Não | Dev |
| `JWT_REFRESH_SECRET` | Auth | string | — | Sim | **Sim** | DevOps |
| `JWT_REFRESH_EXPIRY` | Auth | duration | `7d` | Sim | Não | Dev |
| `OPENAI_API_KEY` | OpenAI | string | — | Sim | **Sim** | DevOps/Billing |
| `OPENAI_MODEL` | OpenAI | string | `gpt-4o` | Não | Não | Dev |
| `OPENAI_TIMEOUT_MS` | OpenAI | number | `30000` | Não | Não | Dev |
| `OPENAI_MAX_RETRIES` | OpenAI | number | `3` | Não | Não | Dev |
| `LANGFUSE_SECRET_KEY` | Langfuse | string | — | Não | **Sim** | DevOps |
| `LANGFUSE_PUBLIC_KEY` | Langfuse | string | — | Não | Não | Dev |
| `LANGFUSE_BASEURL` | Langfuse | URL | `https://cloud.langfuse.com` | Não | Não | Dev |
| `SENTRY_DSN` | Sentry | URL | — | Não | Não | Dev |
| `EVOLUTION_API_URL` | EvolutionAPI | URL | `http://localhost:8080` | Não (Fase 2) | Não | Dev |
| `EVOLUTION_API_KEY` | EvolutionAPI | string | — | Não (Fase 2) | **Sim** | DevOps |
| `EVOLUTION_INSTANCE_NAME` | EvolutionAPI | string | `dani-whatsapp` | Não (Fase 2) | Não | Dev |
| `RATE_LIMIT_WEBCHAT_MAX` | Rate Limit | number | `30` | Não | Não | Dev |
| `RATE_LIMIT_WEBCHAT_WINDOW_S` | Rate Limit | number | `3600` | Não | Não | Dev |
| `RATE_LIMIT_OTP_MAX` | Rate Limit | number | `3` | Não | Não | Dev |
| `RATE_LIMIT_OTP_WINDOW_S` | Rate Limit | number | `3600` | Não | Não | Dev |
| `RATE_LIMIT_OTP_BLOCK_S` | Rate Limit | number | `1800` | Não | Não | Dev |

**Total:** 37 variáveis documentadas (17 sensíveis, 20 não-sensíveis).

---

## 9. Regras de Nomenclatura

Esta seção define as convenções obrigatórias para nomear variáveis de ambiente no projeto.

| Regra | Padrão | Exemplo |
|---|---|---|
| Case | `UPPER_SNAKE_CASE` | `OPENAI_API_KEY` |
| Prefixo por serviço | `SERVICO_NOME` | `REDIS_URL`, `RABBITMQ_URL` |
| Variáveis de frontend (Vite) | prefixo `VITE_` | `VITE_PORT`, `VITE_API_URL` |
| Variáveis booleanas | `_ENABLED` ou `_DISABLED` | `LANGFUSE_ENABLED=true` |
| TTL e timeouts | sufixo `_S` (segundos) ou `_MS` (milissegundos) | `REDIS_TTL_DEFAULT`, `OPENAI_TIMEOUT_MS` |
| Chaves de API externas | sufixo `_KEY` | `OPENAI_API_KEY`, `LANGFUSE_SECRET_KEY` |
| URLs de serviços | sufixo `_URL` | `SUPABASE_URL`, `REDIS_URL` |
| Segredos JWT | prefixo `JWT_` | `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET` |

> ⚙️ **Regra obrigatória:** variáveis expostas ao browser via Vite DEVEM ter prefixo `VITE_`. Variáveis sem esse prefixo nunca chegam ao frontend (tree-shaking do Vite).

> ⚙️ **Variáveis sensíveis** (`DATABASE_URL`, `JWT_*`, `OPENAI_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `REDIS_URL` com senha, `RABBITMQ_URL` com senha, `EVOLUTION_API_KEY`, `LANGFUSE_SECRET_KEY`) jamais devem ter prefixo `VITE_`. Isso resultaria em exposição ao navegador.

---

## 10. Secrets Management

Esta seção define onde ficam os secrets por ambiente, quem acessa e como gerenciá-los.

### 10.1 Onde Ficam os Secrets

| Ambiente | Localização | Mecanismo | Acesso |
|---|---|---|---|
| **Development** | `.env` local (não commitado) | Arquivo local + `.gitignore` | Dev individual |
| **Staging** | GitHub Actions Secrets + Vercel/Railway | CI/CD injeta em runtime | Tech Lead + DevOps |
| **Produção** | GitHub Actions Secrets (Environment: production) + serviço de deploy | CI/CD injeta em runtime | DevOps (acesso restrito) |

### 10.2 Roles e Acesso

| Role | Secrets Acessíveis | Como Acessar |
|---|---|---|
| **Developer** | Chaves dev (Supabase local, Redis local, credenciais de test) | `.env` local fornecido pelo Tech Lead no onboarding |
| **Tech Lead** | Todos os secrets de staging | GitHub Actions Secrets via UI do repositório |
| **DevOps** | Todos os secrets (incluindo produção) | GitHub Actions Secrets → Environment `production` |

### 10.3 Como Adicionar um Novo Secret

```bash
# 1. Documente a variável no .env.example com comentário e sensibilidade
# 2. Adicione ao inventário da seção 9
# 3. Configure no GitHub Actions:
#    Settings → Secrets and variables → Actions → New repository secret
# 4. Para staging/produção: Settings → Environments → production → Add secret
# 5. Atualize o arquivo de deploy (ex: railway.toml ou Vercel env settings)
# 6. Comunique ao time via canal #devops no Slack
```

> ⚙️ **Nunca** compartilhe secrets por e-mail, Slack, Notion ou qualquer canal não-criptografado. Use o mecanismo de Secrets do CI/CD.

### 10.4 Geração de Secrets

```bash
# JWT secrets (32 bytes, base64)
openssl rand -base64 32

# API keys internas (UUID v4)
node -e "console.log(require('crypto').randomUUID())"

# Verificação: nunca use secrets com menos de 32 caracteres para JWT
echo -n "$JWT_ACCESS_SECRET" | wc -c   # deve ser >= 32
```

---

## 11. Rotação de Secrets

Esta seção define o processo para rotacionar secrets sem downtime.

### 11.1 Checklist de Rotação

```markdown
## Rotação de Secret — Checklist

- [ ] Identificar o secret a ser rotacionado e seus consumidores
- [ ] Gerar novo valor (nunca reutilizar valores anteriores)
- [ ] Atualizar no GitHub Actions Secrets (staging → esperar CI verde → produção)
- [ ] Para JWT secrets: redeploy da API com novo secret ANTES de invalidar o antigo
      (período de overlap de 5 minutos para tokens em vôo)
- [ ] Para OPENAI_API_KEY: revogar a chave antiga no dashboard OpenAI APÓS confirmar
      que a nova está funcionando
- [ ] Para DATABASE_URL (prod): coordenar com Supabase → connection string nova antes
      de remover a antiga
- [ ] Executar smoke tests pós-rotação:
      - [ ] Login funcionando
      - [ ] Chat da Dani respondendo
      - [ ] Calculadora retornando resultado
      - [ ] Health check verde
- [ ] Registrar rotação no log de auditoria (data, motivo, responsável)
- [ ] Comunicar ao time via #devops que rotação foi concluída
```

### 11.2 Rotação Sem Downtime — JWT

Para rotacionar `JWT_ACCESS_SECRET` sem forçar re-login de todos os usuários:

1. Adicione `JWT_ACCESS_SECRET_OLD` com o valor atual
2. Atualize `JWT_ACCESS_SECRET` com o novo valor
3. Configure o guard para aceitar tokens assinados por ambos os secrets por 15 minutos
4. Após 15 minutos (expiração do access token), remova `JWT_ACCESS_SECRET_OLD`

> [DECISÃO AUTÔNOMA] Período de overlap de 15 minutos = expiração máxima do access token (JWT_ACCESS_EXPIRY=15m). Alternativa descartada: 30 min de overlap — desnecessário pois tokens expiram em 15m. Critério: mínimo overlap necessário para não forçar logout.

---

## 12. Diferenças entre Ambientes

Esta tabela consolida as diferenças operacionais entre desenvolvimento, staging e produção.

| Configuração | Development | Staging | Produção |
|---|---|---|---|
| `NODE_ENV` | `development` | `staging` | `production` |
| `LOG_LEVEL` | `debug` | `info` | `warn` |
| PostgreSQL | Docker local (porta 5432) | Supabase projeto `dani-staging` | Supabase projeto `dani-prod` |
| Redis | Docker local sem senha | Upstash Redis (TLS) | Upstash Redis (TLS, dedicated) |
| RabbitMQ | Docker local `guest:guest` | CloudAMQP (plan free/launch) | CloudAMQP (plan rabbit/pelican) |
| OpenAI | chave de dev com limite baixo | chave de staging | chave de produção com billing alert |
| Langfuse | desabilitado (variável vazia) | habilitado, projeto `dani-staging` | habilitado, projeto `dani-prod` |
| Sentry | desabilitado ou projeto local | habilitado, environment `staging` | habilitado, environment `production` |
| EvolutionAPI | não configurado (Fase 2) | instância de teste | instância de produção |
| CORS | `http://localhost:5173` | URL do deploy staging | URL de produção |
| Rate limit webchat | 30 msgs/hora (idem prod) | 30 msgs/hora | 30 msgs/hora |
| Swagger UI | habilitado | habilitado | **desabilitado** |
| Prisma Studio | habilitado (`pnpm prisma studio`) | desabilitado | desabilitado |
| Seed automático | sim (`pnpm prisma db seed`) | sim (dados de teste) | não (dados reais) |
| JWT expiry access | 15m (idem prod) | 15m | 15m |
| JWT expiry refresh | 7d (idem prod) | 7d | 7d |

> ⚙️ **Regra crítica:** `SUPABASE_SERVICE_ROLE_KEY` de produção NUNCA deve aparecer em `.env.local` ou qualquer arquivo fora do CI/CD vault. Violação desta regra é incidente de segurança nível P0.

---

## 13. Troubleshooting

Esta seção cobre os erros mais comuns com causa raiz, erro exato e ação de resolução.

**Anti-exemplo:**
```
❌ Se der erro, verifique os logs dos containers.
```
```
✅ Execute `docker compose logs postgres --tail=50` para ver os últimos 50 logs do
   container postgres. Se aparecer "FATAL: password authentication failed", verifique
   que DATABASE_URL no .env usa "postgres:postgres" como credenciais (dev).
```

### Cenário 1 — Porta já em uso

**Erro:**
```
Error response from daemon: driver failed programming external connectivity:
Bind for 0.0.0.0:5432 failed: port is already allocated
```

**Causa:** PostgreSQL local (ou outro container) já está usando a porta 5432.

**Resolução:**
```bash
# Identifica o processo usando a porta
lsof -i :5432
# Para o processo ou muda a porta no docker-compose.yml:
# "5433:5432" → acesse em localhost:5433 e ajuste DATABASE_URL
```

---

### Cenário 2 — Prisma falha na migration

**Erro:**
```
Error: P1001: Can't reach database server at `localhost:5432`
```

**Causa:** Container postgres não está rodando ou não está healthy.

**Resolução:**
```bash
docker compose ps    # verifica status
docker compose up -d postgres   # sobe apenas o postgres
docker compose ps    # aguarda status "healthy" (pode levar 30s)
pnpm prisma migrate dev   # tenta novamente
```

---

### Cenário 3 — `supabase start` falha

**Erro:**
```
Error: supabase/postgres failed to start on port 5432: port is busy
```

**Causa:** Conflito com o container Docker do postgres já rodando.

**Resolução:**
```bash
# Opção A: pare o container Docker postgres e use Supabase CLI
docker compose stop postgres
supabase start

# Opção B: use apenas o Docker Compose (sem Supabase CLI)
# Ajuste SUPABASE_URL no .env para apontar para localhost:54321 apenas
# se for usar a CLI. Para dev simples, o Docker Compose é suficiente.
```

---

### Cenário 4 — Redis connection refused

**Erro:**
```
Error: connect ECONNREFUSED 127.0.0.1:6379
```

**Causa:** Container Redis não está rodando.

**Resolução:**
```bash
docker compose up -d redis
docker exec dani_redis redis-cli ping
# Output esperado: PONG
```

---

### Cenário 5 — JWT secret inválido

**Erro:**
```
JsonWebTokenError: invalid signature
```

**Causa:** `JWT_ACCESS_SECRET` no `.env` não corresponde ao secret usado para gerar o token.

**Resolução:**
```bash
# Gere um novo secret
openssl rand -base64 32
# Atualize JWT_ACCESS_SECRET no .env
# Reinicie a API
pnpm --filter apps/api dev
# Em dev, todos os tokens existentes serão invalidados (normal)
```

---

### Cenário 6 — OpenAI 401 Unauthorized

**Erro:**
```
OpenAIError: 401 Unauthorized - Invalid API key
```

**Causa:** `OPENAI_API_KEY` ausente, incorreto ou revogado.

**Resolução:**
```bash
# Verifique se a variável está definida
echo $OPENAI_API_KEY   # deve exibir sk-...
# Se vazio, abra o .env e adicione a chave
# Se presente mas inválido: acesse platform.openai.com e regenere a chave
```

---

### Cenário 7 — pnpm install falha com peer deps

**Erro:**
```
ERR_PNPM_PEER_DEP_ISSUES  Unmet peer dependencies
```

**Causa:** Conflito de versão entre pacotes do monorepo.

**Resolução:**
```bash
# Aceita peer deps automaticamente (comportamento padrão do monorepo)
pnpm install --shamefully-hoist
# Se persistir, verifique a versão do Node:
node --version   # deve ser v20.x.x (LTS)
nvm use 20       # se necessário
pnpm install
```

---

### Cenário 8 — RabbitMQ connection refused

**Erro:**
```
Error: connect ECONNREFUSED 127.0.0.1:5672
```

**Causa:** Container RabbitMQ não está rodando ou ainda está em startup.

**Resolução:**
```bash
docker compose up -d rabbitmq
# Aguarda 30s (RabbitMQ demora mais para iniciar)
docker compose ps     # verifica "healthy"
# Se não ficar healthy em 60s:
docker compose logs rabbitmq --tail=50
```

---

## 14. Backlog de Pendências

| ID | Item | Status | Tipo | Observação |
|---|---|---|---|---|
| P22-01 | Vault centralizado (HashiCorp Vault ou AWS Secrets Manager) para secrets de produção | Pendente | [DEFINIÇÃO PENDENTE] — Opção A: GitHub Actions Secrets (simples, sem infra adicional). Opção B: HashiCorp Vault self-hosted (mais controle, mais custo). Trade-off: custo vs controle. | Para o momento, GitHub Actions Secrets é suficiente. |
| P22-02 | Configuração de `REDIS_URL` com TLS e autenticação para staging/prod | [DECISÃO AUTÔNOMA] Upstash Redis com TLS habilitado por padrão. Alternativa descartada: Redis self-hosted — Upstash elimina gestão de infra. Critério: menor overhead operacional. | Concluído | Doc 17 especifica Upstash como opção aprovada |
| P22-03 | Rotação automática de JWT secrets com período de overlap | [DECISÃO AUTÔNOMA] Overlap de 15 minutos (= JWT_ACCESS_EXPIRY). Alternativa descartada: 30min. Critério: mínimo overlap para não forçar logout. | Concluído | Seção 11.2 |
| P22-04 | Integração com Doppler ou similar para sync de `.env` entre devs do time | Pendente | [DEFINIÇÃO PENDENTE] — Opção A: Doppler (SaaS, fácil onboarding). Opção B: 1Password Secrets Automation. Trade-off: custo vs integração com ferramentas existentes. | Avaliar na Fase 3 |
