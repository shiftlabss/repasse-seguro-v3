# 22 - Guia de Ambiente, Setup Local e Secrets

## Módulo Cessionário · Plataforma Repasse Seguro

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 22 - Guia de Ambiente, Setup Local e Secrets | v1.0 | 2026-03-22 (America/Fortaleza) | Claude Code Desktop | Aprovado |

---

> 📌 **TL;DR**
>
> - **Pré-requisitos:** Node.js 22+, pnpm 9+, Docker + Docker Compose, Git, Supabase CLI, Railway CLI.
> - **Tempo estimado de setup:** menos de 15 minutos seguindo este guia do zero.
> - **Serviços locais (Docker):** PostgreSQL 17, Redis 7, RabbitMQ 4. Supabase e serviços externos via mocks/sandboxes.
> - **Total de variáveis de ambiente:** 36 variáveis entre apps (backend, frontend, mobile).
> - **Ambientes:** `dev` (local + Docker), `staging` (Railway + Supabase staging), `prod` (Railway + Supabase prod).
> - **Regra de secrets:** Nenhuma variável sensível em repositório git. Secrets gerenciados via Railway (backend), Vercel (frontend) e EAS Secrets (mobile).
> - **Monorepo:** Turborepo + pnpm workspaces — um único `pnpm install` instala tudo.

---

## 1. Pré-requisitos

Antes de clonar o repositório, verifique se todas as ferramentas estão instaladas e nas versões corretas.

| Ferramenta | Versão Mínima | Instalação | Verificação |
|---|---|---|---|
| Node.js | 22.x LTS | [nodejs.org/en/download](https://nodejs.org/en/download) ou `nvm install 22` | `node --version` → `v22.x.x` |
| pnpm | 9.x | `npm install -g pnpm@9` | `pnpm --version` → `9.x.x` |
| Docker Desktop | 27.x | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop) | `docker --version` → `27.x.x` |
| Docker Compose | 2.x (incluído no Docker Desktop) | Incluído no Docker Desktop | `docker compose version` → `v2.x.x` |
| Git | 2.x | [git-scm.com](https://git-scm.com) | `git --version` → `git version 2.x.x` |
| Supabase CLI | 2.x | `npm install -g supabase@2` | `supabase --version` → `2.x.x` |
| Railway CLI | latest | `npm install -g @railway/cli` | `railway --version` → `x.x.x` |

> 💡 **Dica:** Use `nvm` (Node Version Manager) para gerenciar versões de Node.js. O projeto contém `.nvmrc` na raiz com `22`. Execute `nvm use` na raiz do projeto para ativar automaticamente.

---

## 2. Clone e Instalação

### 2.1 Clonar o repositório

```bash
git clone https://github.com/repasse-seguro/cessionario.git
cd cessionario
```

**Output esperado:** Repositório clonado sem erros. Estrutura de diretórios visível com `ls`.

### 2.2 Instalar dependências

```bash
pnpm install
```

**Output esperado:**
```
Packages: +XXX
Progress: resolved XXX, reused XXX, downloaded XXX, added XXX
Done in Xs
```

> ✅ O `pnpm install` instala dependências de todos os workspaces: `apps/web`, `apps/api`, `apps/mobile`, e todos os `packages/`.

### 2.3 Configurar Git hooks

```bash
pnpm prepare
```

**Output esperado:** `husky - Git hooks installed` (instala hooks de pre-commit e commit-msg via Husky).

### 2.4 Copiar arquivos de variáveis de ambiente

```bash
cp apps/api/.env.example apps/api/.env
cp apps/web/.env.example apps/web/.env
```

**Output esperado:** Arquivos `.env` criados. Editar com valores corretos antes de subir os serviços.

---

## 3. Docker Compose

O ambiente local usa Docker Compose para prover os serviços de infraestrutura: PostgreSQL, Redis e RabbitMQ. Supabase usa o projeto de desenvolvimento remoto (não roda localmente via Docker por padrão).

### 3.1 Arquivo docker-compose.yml

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:17-alpine
    container_name: repasse_postgres
    environment:
      POSTGRES_USER: repasse
      POSTGRES_PASSWORD: repasse_dev_password
      POSTGRES_DB: repasse_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U repasse -d repasse_dev"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - repasse_net

  redis:
    image: redis:7.4-alpine
    container_name: repasse_redis
    command: redis-server --requirepass redis_dev_password --appendonly yes
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "redis_dev_password", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - repasse_net

  rabbitmq:
    image: rabbitmq:4-management-alpine
    container_name: repasse_rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: repasse
      RABBITMQ_DEFAULT_PASS: rabbitmq_dev_password
      RABBITMQ_DEFAULT_VHOST: repasse_vhost
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s
    networks:
      - repasse_net

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:

networks:
  repasse_net:
    driver: bridge
```

### 3.2 Tabela de serviços locais

| Serviço | Porta local | Health check | Interface de admin |
|---|---|---|---|
| PostgreSQL 17 | `5432` | `pg_isready -U repasse` | `psql -h localhost -U repasse -d repasse_dev` |
| Redis 7.4 | `6379` | `redis-cli ping` | `redis-cli -a redis_dev_password` |
| RabbitMQ 4 | `5672` (AMQP) | `rabbitmq-diagnostics ping` | `http://localhost:15672` (user: `repasse`, pass: `rabbitmq_dev_password`) |

> ⚙️ **Nota:** Supabase (PostgreSQL gerenciado, Auth, Storage, Realtime, pgvector) usa o projeto Supabase **dev** remoto. Não roda localmente via Docker para manter paridade com o ambiente de produção e garantir que RLS, pgvector e Realtime estejam disponíveis.

---

## 4. Variáveis de Ambiente (.env.example)

### 4.1 Backend — `apps/api/.env.example`

```dotenv
# =============================================================================
# REPASSE SEGURO — API (NestJS 10)
# Ambiente: development
# =============================================================================

# -----------------------------------------------------------------------------
# APLICAÇÃO
# -----------------------------------------------------------------------------
NODE_ENV=development
PORT=3001
APP_URL=http://localhost:3001
CORS_ORIGIN=http://localhost:5173

# -----------------------------------------------------------------------------
# BANCO DE DADOS — Supabase PostgreSQL via Prisma
# Obter em: Supabase Dashboard > Settings > Database > Connection String
# SENSÍVEL: nunca commitar valor real
# -----------------------------------------------------------------------------
DATABASE_URL=postgresql://postgres:[PASSWORD]@db.[PROJECT_REF].supabase.co:5432/postgres?schema=public

# -----------------------------------------------------------------------------
# SUPABASE
# Obter em: Supabase Dashboard > Settings > API
# -----------------------------------------------------------------------------
SUPABASE_URL=https://[PROJECT_REF].supabase.co
SUPABASE_ANON_KEY=eyJ...  # chave pública — segura para expor
SUPABASE_SERVICE_ROLE_KEY=eyJ...  # SENSÍVEL: nunca expor no frontend ou logs

# -----------------------------------------------------------------------------
# JWT — Autenticação
# Usar: openssl rand -base64 64
# SENSÍVEL
# -----------------------------------------------------------------------------
JWT_SECRET=gere_um_secret_forte_aqui_minimo_64_chars
JWT_ACCESS_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=30d

# -----------------------------------------------------------------------------
# REDIS — Upstash em produção, Docker local
# -----------------------------------------------------------------------------
REDIS_URL=redis://:redis_dev_password@localhost:6379

# -----------------------------------------------------------------------------
# RABBITMQ — CloudAMQP em produção, Docker local
# -----------------------------------------------------------------------------
RABBITMQ_URL=amqp://repasse:rabbitmq_dev_password@localhost:5672/repasse_vhost

# -----------------------------------------------------------------------------
# OPENAI — Analista de Oportunidades
# Obter em: platform.openai.com/api-keys
# SENSÍVEL
# -----------------------------------------------------------------------------
OPENAI_API_KEY=sk-...
OPENAI_ORG_ID=org-...

# -----------------------------------------------------------------------------
# ZAPSIGN — Assinatura Digital
# Obter em: dashboard.zapsign.com.br > API Keys
# SENSÍVEL
# -----------------------------------------------------------------------------
ZAPSIGN_API_TOKEN=...
ZAPSIGN_WEBHOOK_SECRET=...
ZAPSIGN_SANDBOX_URL=https://sandbox.api.zapsign.com.br/api/v1
ZAPSIGN_API_URL=https://api.zapsign.com.br/api/v1  # usar sandbox em dev

# -----------------------------------------------------------------------------
# IDWALL — KYC (OCR + Liveness)
# Obter em: plataforma idwall
# SENSÍVEL
# -----------------------------------------------------------------------------
IDWALL_API_URL=https://api.idwall.co
IDWALL_API_KEY=...
IDWALL_WEBHOOK_SECRET=...

# -----------------------------------------------------------------------------
# CELCOIN — Escrow
# SENSÍVEL
# -----------------------------------------------------------------------------
CELCOIN_API_URL=https://sandbox.celcoin.com.br  # usar sandbox em dev
CELCOIN_CLIENT_ID=...
CELCOIN_CLIENT_SECRET=...

# -----------------------------------------------------------------------------
# RESEND — E-mail Transacional
# Obter em: resend.com/api-keys
# SENSÍVEL
# -----------------------------------------------------------------------------
RESEND_API_KEY=re_...
RESEND_FROM_EMAIL=noreply@repasseseguro.com.br

# -----------------------------------------------------------------------------
# EXPO — Push Notifications
# Obter em: expo.dev > Account Settings > Access Tokens
# SENSÍVEL
# -----------------------------------------------------------------------------
EXPO_ACCESS_TOKEN=...

# -----------------------------------------------------------------------------
# LANGFUSE — Observabilidade de IA
# Obter em: cloud.langfuse.com > Settings > API Keys
# SENSÍVEL
# -----------------------------------------------------------------------------
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com

# -----------------------------------------------------------------------------
# SENTRY — Error Tracking
# Obter em: sentry.io > Settings > Projects > DSN
# Público — pode ser exposto no frontend
# -----------------------------------------------------------------------------
SENTRY_DSN_BACKEND=https://...@o...sentry.io/...
SENTRY_ENVIRONMENT=development

# -----------------------------------------------------------------------------
# POSTHOG — Analytics
# Obter em: app.posthog.com > Project Settings > API Key
# Público — pode ser exposto no frontend
# -----------------------------------------------------------------------------
POSTHOG_API_KEY=phc_...
POSTHOG_HOST=https://app.posthog.com
```

### 4.2 Frontend — `apps/web/.env.example`

```dotenv
# =============================================================================
# REPASSE SEGURO — Web (React + Vite)
# Ambiente: development
# Prefixo VITE_ obrigatório para variáveis expostas ao browser
# =============================================================================

VITE_API_URL=http://localhost:3001/api/v1
VITE_SUPABASE_URL=https://[PROJECT_REF].supabase.co
VITE_SUPABASE_ANON_KEY=eyJ...  # chave pública
VITE_SENTRY_DSN=https://...@o...sentry.io/...
VITE_POSTHOG_API_KEY=phc_...
VITE_POSTHOG_HOST=https://app.posthog.com
VITE_APP_ENV=development
```

---

## 5. Subir o Projeto

Execute os passos na ordem exata abaixo.

### Passo 1 — Subir serviços Docker

```bash
docker compose up -d
```

**Output esperado:**
```
[+] Running 3/3
 ✔ Container repasse_postgres  Started
 ✔ Container repasse_redis     Started
 ✔ Container repasse_rabbitmq  Started
```

**Validar saúde dos containers:**
```bash
docker compose ps
```

Todos os serviços devem aparecer com status `healthy`.

### Passo 2 — Gerar Prisma Client

```bash
pnpm --filter api exec prisma generate
```

**Output esperado:** `Generated Prisma Client (v6.x.x)`.

### Passo 3 — Rodar migrations

```bash
pnpm --filter api exec prisma migrate dev
```

**Output esperado:**
```
Applying migration `20260322_init`
Your database is now in sync with your schema.
```

### Passo 4 — Rodar seed (dados de desenvolvimento)

```bash
pnpm --filter api exec prisma db seed
```

**Output esperado:** `Database seeded successfully.`

### Passo 5 — Subir o backend

```bash
pnpm --filter api dev
```

**Output esperado:**
```
[Nest] LOG [NestApplication] Nest application successfully started +XXms
[Nest] LOG [NestApplication] App running on http://localhost:3001
```

### Passo 6 — Subir o frontend (nova aba do terminal)

```bash
pnpm --filter web dev
```

**Output esperado:**
```
  VITE vX.X.X  ready in XXX ms
  ➜  Local:   http://localhost:5173/
```

### Passo 7 — Acessar a aplicação

Abra `http://localhost:5173` no navegador. Você deve ver a tela de login do Repasse Seguro.

---

## 6. Health Check (Validação)

Execute estes comandos para confirmar que todos os serviços estão funcionando.

```bash
# 1. API responde
curl -s http://localhost:3001/health | jq .
# Output esperado: { "status": "ok", "uptime": X, "database": "connected", "redis": "connected", "rabbitmq": "connected" }

# 2. PostgreSQL acessível
docker exec repasse_postgres pg_isready -U repasse -d repasse_dev
# Output esperado: localhost:5432 - accepting connections

# 3. Redis acessível
docker exec repasse_redis redis-cli -a redis_dev_password ping
# Output esperado: PONG

# 4. RabbitMQ UI acessível
curl -s -u repasse:rabbitmq_dev_password http://localhost:15672/api/overview | jq .rabbitmq_version
# Output esperado: "4.x.x"

# 5. Frontend renderiza
curl -s http://localhost:5173 | grep -o "<title>.*</title>"
# Output esperado: <title>Repasse Seguro</title>
```

> ✅ Se todos os 5 health checks passam, o ambiente local está pronto para desenvolvimento.

---

## 7. Inventário de Variáveis

### 7.1 Backend (`apps/api`)

| Variável | Descrição | Obrigatória | Sensível | Padrão (dev) |
|---|---|---|---|---|
| `NODE_ENV` | Ambiente de execução | Sim | Não | `development` |
| `PORT` | Porta HTTP do backend | Sim | Não | `3001` |
| `APP_URL` | URL base da API | Sim | Não | `http://localhost:3001` |
| `CORS_ORIGIN` | Origem permitida para CORS | Sim | Não | `http://localhost:5173` |
| `DATABASE_URL` | Connection string Prisma → Supabase | Sim | **Sim** | Ver `.env.example` |
| `SUPABASE_URL` | URL do projeto Supabase | Sim | Não | `https://[REF].supabase.co` |
| `SUPABASE_ANON_KEY` | Chave pública Supabase | Sim | Não | `eyJ...` |
| `SUPABASE_SERVICE_ROLE_KEY` | Chave privada Supabase (bypass RLS) | Sim | **Sim** | `eyJ...` |
| `JWT_SECRET` | Secret para assinar JWTs | Sim | **Sim** | Gerar com `openssl rand -base64 64` |
| `JWT_ACCESS_EXPIRES_IN` | Expiração do access token | Sim | Não | `15m` |
| `JWT_REFRESH_EXPIRES_IN` | Expiração do refresh token | Sim | Não | `30d` |
| `REDIS_URL` | URL de conexão Redis | Sim | **Sim** (senha) | `redis://:redis_dev_password@localhost:6379` |
| `RABBITMQ_URL` | URL de conexão RabbitMQ | Sim | **Sim** (senha) | `amqp://repasse:...@localhost:5672/repasse_vhost` |
| `OPENAI_API_KEY` | API Key OpenAI | Sim | **Sim** | `sk-...` |
| `OPENAI_ORG_ID` | ID da organização OpenAI | Não | Não | `org-...` |
| `ZAPSIGN_API_TOKEN` | Token de API ZapSign | Sim | **Sim** | Sandbox token |
| `ZAPSIGN_WEBHOOK_SECRET` | Secret para validar webhooks ZapSign | Sim | **Sim** | Gerar aleatório |
| `ZAPSIGN_SANDBOX_URL` | URL sandbox ZapSign | Dev | Não | `https://sandbox.api.zapsign.com.br/api/v1` |
| `ZAPSIGN_API_URL` | URL produção ZapSign | Prod | Não | `https://api.zapsign.com.br/api/v1` |
| `IDWALL_API_URL` | URL API idwall | Sim | Não | `https://api.idwall.co` |
| `IDWALL_API_KEY` | API Key idwall | Sim | **Sim** | Conta de teste |
| `IDWALL_WEBHOOK_SECRET` | Secret webhooks idwall | Sim | **Sim** | Gerar aleatório |
| `CELCOIN_API_URL` | URL API Celcoin | Sim | Não | Sandbox URL |
| `CELCOIN_CLIENT_ID` | OAuth Client ID Celcoin | Sim | **Sim** | Conta sandbox |
| `CELCOIN_CLIENT_SECRET` | OAuth Client Secret Celcoin | Sim | **Sim** | Conta sandbox |
| `RESEND_API_KEY` | API Key Resend | Sim | **Sim** | `re_...` |
| `RESEND_FROM_EMAIL` | E-mail remetente | Sim | Não | `noreply@repasseseguro.com.br` |
| `EXPO_ACCESS_TOKEN` | Token acesso Expo Push API | Sim | **Sim** | `...` |
| `LANGFUSE_PUBLIC_KEY` | Chave pública Langfuse | Sim | **Sim** | `pk-lf-...` |
| `LANGFUSE_SECRET_KEY` | Chave secreta Langfuse | Sim | **Sim** | `sk-lf-...` |
| `LANGFUSE_HOST` | URL Langfuse Cloud | Sim | Não | `https://cloud.langfuse.com` |
| `SENTRY_DSN_BACKEND` | DSN Sentry backend | Sim | Não | `https://...sentry.io/...` |
| `SENTRY_ENVIRONMENT` | Ambiente para Sentry | Sim | Não | `development` |
| `POSTHOG_API_KEY` | API Key PostHog | Sim | Não | `phc_...` |
| `POSTHOG_HOST` | URL PostHog | Sim | Não | `https://app.posthog.com` |

### 7.2 Frontend (`apps/web`)

| Variável | Descrição | Obrigatória | Sensível |
|---|---|---|---|
| `VITE_API_URL` | URL base da API backend | Sim | Não |
| `VITE_SUPABASE_URL` | URL do projeto Supabase | Sim | Não |
| `VITE_SUPABASE_ANON_KEY` | Chave pública Supabase | Sim | Não |
| `VITE_SENTRY_DSN` | DSN Sentry frontend | Sim | Não |
| `VITE_POSTHOG_API_KEY` | API Key PostHog | Sim | Não |
| `VITE_POSTHOG_HOST` | URL PostHog | Sim | Não |
| `VITE_APP_ENV` | Ambiente da aplicação | Sim | Não |

---

## 8. Regras de Nomenclatura

> ⚙️ **Padrão obrigatório de nomenclatura de variáveis de ambiente.**

| Regra | Exemplo correto | Exemplo errado |
|---|---|---|
| `UPPER_SNAKE_CASE` obrigatório | `OPENAI_API_KEY` | `openai_api_key`, `openaiApiKey` |
| Prefixo `VITE_` para variáveis expostas ao browser | `VITE_API_URL` | `API_URL` (não exposto ao browser) |
| Agrupamento por serviço no `.env` | `ZAPSIGN_API_TOKEN`, `ZAPSIGN_WEBHOOK_SECRET` | Variáveis misturadas sem contexto |
| Sufixo `_KEY` para API Keys | `RESEND_API_KEY` | `RESEND_TOKEN` |
| Sufixo `_SECRET` para secrets de HMAC/OAuth | `ZAPSIGN_WEBHOOK_SECRET` | `ZAPSIGN_HOOK` |
| Sufixo `_URL` para URLs de serviços | `CELCOIN_API_URL` | `CELCOIN_ENDPOINT` |
| Sufixo `_DSN` para conexões de monitoramento | `SENTRY_DSN_BACKEND` | `SENTRY_URL_BACKEND` |

---

## 9. Secrets Management

### 9.1 Onde ficam os secrets por ambiente

| Ambiente | Backend | Frontend | Mobile |
|---|---|---|---|
| **dev** (local) | `apps/api/.env` (gitignored) | `apps/web/.env` (gitignored) | `apps/mobile/.env` (gitignored) |
| **staging** | Railway Environment Variables | Vercel Environment Variables | EAS Secrets (`eas secret:push`) |
| **prod** | Railway Environment Variables | Vercel Environment Variables | EAS Secrets (`eas secret:push`) |

> 🔴 **Regra inegociável:** Arquivos `.env` com valores reais nunca são commitados. O `.gitignore` já os exclui. Verifique antes de qualquer commit com `git status`.

### 9.2 Quem tem acesso aos secrets

| Tipo de secret | Acesso |
|---|---|
| Secrets de dev (sandboxes) | Todos os devs — compartilhados via gestor de senhas da equipe (ex: 1Password, Bitwarden) |
| Secrets de staging | Tech Lead + DevOps |
| Secrets de produção | Tech Lead + DevOps + CTO |
| `SUPABASE_SERVICE_ROLE_KEY` (prod) | Tech Lead + CTO — máximo 2 pessoas |

### 9.3 Como adicionar um novo secret

**Dev:**
1. Adicionar ao `.env.example` com comentário descritivo.
2. Adicionar ao `.env` local com o valor.
3. Documentar no inventário da seção 7 deste documento.

**Staging / Prod (Railway):**
```bash
railway variables set NOME_DA_VARIAVEL=valor --environment staging
railway variables set NOME_DA_VARIAVEL=valor --environment production
```

**Mobile (EAS):**
```bash
eas secret:push --scope project --env-file apps/mobile/.env.production
```

### 9.4 Como não fazer

> ❌ **Anti-exemplo 1 — Variável sensível no repositório:**
> ```bash
> # ERRADO — nunca commitar com valor real
> OPENAI_API_KEY=sk-proj-AbC123...realkey
> ```
> ✅ **Correto:** `OPENAI_API_KEY=your_openai_api_key_here` no `.env.example`.

> ❌ **Anti-exemplo 2 — Variável sensível exposta ao frontend:**
> ```dotenv
> # ERRADO — SUPABASE_SERVICE_ROLE_KEY nunca deve ter prefixo VITE_
> VITE_SUPABASE_SERVICE_ROLE_KEY=eyJ...
> ```
> ✅ **Correto:** `SUPABASE_SERVICE_ROLE_KEY` apenas no backend, sem prefixo `VITE_`.

---

## 10. Rotação de Secrets

### 10.1 Calendário de rotação

| Secret | Frequência | Responsável |
|---|---|---|
| `OPENAI_API_KEY` | Mensal | Tech Lead |
| `ZAPSIGN_API_TOKEN`, `IDWALL_API_KEY`, `CELCOIN_CLIENT_SECRET`, `RESEND_API_KEY` | Trimestral | Tech Lead |
| `SUPABASE_SERVICE_ROLE_KEY`, `JWT_SECRET`, `LANGFUSE_*`, `EXPO_ACCESS_TOKEN` | Semestral | Tech Lead + CTO |
| `ZAPSIGN_WEBHOOK_SECRET`, `IDWALL_WEBHOOK_SECRET` | Semestral | DevOps |
| `SUPABASE_ANON_KEY` | Anual | Tech Lead |

### 10.2 Checklist de rotação sem downtime

Para rotacionar qualquer secret sem indisponibilidade:

- [ ] 1. Gerar o novo valor do secret no provedor (ex: revogar e criar novo API Key).
- [ ] 2. Adicionar o novo valor no Railway/Vercel/EAS **sem remover o antigo ainda**.
- [ ] 3. Fazer deploy com o novo valor ativo.
- [ ] 4. Monitorar logs por 15 minutos para confirmar que não há erros de autenticação.
- [ ] 5. Remover o valor antigo do provedor.
- [ ] 6. Atualizar o gestor de senhas da equipe.
- [ ] 7. Registrar a rotação no changelog de segurança (Notion: Runbook de Segurança).

> 💡 **Webhooks (ZapSign, idwall):** A rotação do webhook secret exige atualizar o valor no painel do provedor **ao mesmo tempo** que no Railway, pois há uma janela de inconsistência. Planejar fora do horário de pico.

---

## 11. Diferenças entre Ambientes

| Configuração | dev (local) | staging | prod |
|---|---|---|---|
| PostgreSQL | Docker local (`localhost:5432`) | Supabase projeto `staging` | Supabase projeto `prod` |
| Redis | Docker local (`localhost:6379`) | Upstash serverless (staging) | Upstash serverless (prod) |
| RabbitMQ | Docker local (`localhost:5672`) | CloudAMQP (staging vhost) | CloudAMQP (prod vhost) |
| ZapSign | Sandbox URL | Sandbox URL | Produção URL |
| idwall | Conta de teste | Conta de teste | Conta produção |
| Celcoin | Sandbox | Sandbox | Produção (integração manual MVP) |
| Resend | Modo dev (e-mails vão para Resend dashboard) | Staging (e-mails reais para testers) | Produção |
| OpenAI | GPT-4o-mini (custo menor) | GPT-4 Turbo (paridade prod) | GPT-4 Turbo |
| Sentry | `environment: development` (alertas desabilitados) | `environment: staging` | `environment: production` |
| PostHog | Analytics desabilitado por padrão | Analytics habilitado para testers | Analytics habilitado |
| RLS Supabase | Habilitado (igual prod) | Habilitado | Habilitado |
| CORS_ORIGIN | `http://localhost:5173` | `https://staging.repasseseguro.com.br` | `https://app.repasseseguro.com.br` |
| Rate limiting | Relaxado (1000 req/min) | Igual prod | Conforme D16 |
| Logs (Pino) | `level: debug`, pretty print | `level: info`, JSON | `level: info`, JSON |

---

## 12. Troubleshooting

### 12.1 `pnpm install` falha com erro de lockfile

**Erro:** `ERR_PNPM_OUTDATED_LOCKFILE` ou conflito de versões.

```bash
# Solução: regenerar lockfile
pnpm install --frozen-lockfile=false
```

Se persistir: delete `node_modules` e `pnpm-lock.yaml` e reinstale:
```bash
find . -name "node_modules" -type d -prune -exec rm -rf {} \; && rm pnpm-lock.yaml && pnpm install
```

---

### 12.2 PostgreSQL Docker não sobe (porta ocupada)

**Erro:** `Error starting userland proxy: listen tcp 0.0.0.0:5432: bind: address already in use`

**Causa:** PostgreSQL local instalado no host ocupando a porta 5432.

```bash
# Verificar quem está usando a porta
lsof -ti:5432

# Opção 1: parar o PostgreSQL local
sudo service postgresql stop  # Linux
brew services stop postgresql  # macOS

# Opção 2: mudar a porta no docker-compose.yml
ports:
  - "5433:5432"  # usar porta 5433 externamente
# Atualizar DATABASE_URL: postgresql://...localhost:5433/...
```

---

### 12.3 Prisma migrate falha com erro de schema

**Erro:** `Error: P3006 Migration failed to apply cleanly to the shadow database`

```bash
# Resetar banco de dev (apaga todos os dados)
pnpm --filter api exec prisma migrate reset

# Se persistir, checar conflito de enums:
pnpm --filter api exec prisma db push --force-reset
```

> 🔴 **Atenção:** `prisma migrate reset` apaga todos os dados do banco dev. Use apenas em desenvolvimento.

---

### 12.4 RabbitMQ Management UI inacessível

**Erro:** `http://localhost:15672` retorna `ERR_CONNECTION_REFUSED`.

```bash
# Verificar se o container está running e healthy
docker compose ps rabbitmq

# Se status for "starting" aguardar até 60s (RabbitMQ é lento para iniciar)
docker compose logs rabbitmq --follow

# Se status for "unhealthy", reiniciar:
docker compose restart rabbitmq
```

---

### 12.5 Backend falha ao conectar no Redis

**Erro no log:** `Error: connect ECONNREFUSED 127.0.0.1:6379`

```bash
# Verificar se Redis está rodando
docker compose ps redis

# Testar conexão manualmente
docker exec repasse_redis redis-cli -a redis_dev_password ping
# Output esperado: PONG

# Verificar variável de ambiente
grep REDIS_URL apps/api/.env
# Deve ser: redis://:redis_dev_password@localhost:6379
```

---

### 12.6 Variável de ambiente não encontrada no backend

**Erro:** `Error: ZAPSIGN_API_TOKEN is not defined`

```bash
# Verificar se o .env existe
ls -la apps/api/.env

# Se não existir, criar a partir do exemplo:
cp apps/api/.env.example apps/api/.env

# Verificar se a variável está no arquivo
grep ZAPSIGN_API_TOKEN apps/api/.env
```

> ❌ **Anti-exemplo — instrução genérica:**
> "Verifique as variáveis de ambiente do projeto."
>
> ✅ **Correto:** Cada erro de variável ausente tem causa, comando de diagnóstico e ação corretiva específica.

---

### 12.7 Docker Compose sem health check (anti-exemplo)

> ❌ **Anti-exemplo — Docker Compose sem health check:**
> ```yaml
> services:
>   postgres:
>     image: postgres:17
>     ports: ["5432:5432"]
> ```
> **Problema:** O sistema não sabe se o Postgres está pronto. O backend pode tentar conectar antes do banco estar aceitando conexões.
>
> ✅ **Correto:** Sempre incluir `healthcheck` + `depends_on` com `condition: service_healthy` para serviços dependentes.

---

### 12.8 `.env.example` incompleto (anti-exemplo)

> ❌ **Anti-exemplo — `.env.example` sem comentários:**
> ```dotenv
> SUPABASE_SERVICE_ROLE_KEY=
> ```
> **Problema:** Dev não sabe onde obter este valor, que escopo tem, ou que é sensível.
>
> ✅ **Correto:**
> ```dotenv
> # Supabase Service Role Key — bypass de RLS. NUNCA expor no frontend.
> # Obter em: Supabase Dashboard > Settings > API > service_role
> # SENSÍVEL — rotação semestral
> SUPABASE_SERVICE_ROLE_KEY=eyJ...
> ```

---

## 13. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-22 | v1.0 | Criação do documento — setup completo, 36 variáveis, Docker Compose com health checks, secrets management, rotação, troubleshooting com 7 cenários. |

---

## 14. Backlog de Pendências

| Item | Marcador | Seção | Justificativa | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Supabase local vs remoto em dev | `[DECISÃO AUTÔNOMA]` | 3 — Docker Compose | Supabase não roda em Docker local para dev. Descartado: `supabase start` (Supabase CLI local) — falta pgvector HNSW e Realtime com paridade prod. Critério: paridade máxima com produção desde o primeiro dia de dev. | Requer conexão internet para dev; isolamento de dados dev é responsabilidade do projeto Supabase dev. | Tech Lead | Implementado |
| Porta Redis local | `[DECISÃO AUTÔNOMA]` | 3 — Docker Compose | Porta padrão 6379 mantida. Alternativa: porta customizada. Critério: padrão de mercado — simplifica onboarding sem necessidade de override. | Baixo (seção troubleshooting cobre conflito de porta) | DevOps | Implementado |
| Gestor de senhas da equipe | `[DEFINIÇÃO PENDENTE]` | 9.2 — Quem tem acesso | Gestor de senhas não definido ainda. Opção A: 1Password Teams (custo mensal, integração com CLI). Opção B: Bitwarden (open source, auto-hospedado). Impacto: processo de compartilhamento seguro de secrets dev. | Médio — sem gestor, secrets são compartilhados por canais inseguros. | CTO | Aberto |
