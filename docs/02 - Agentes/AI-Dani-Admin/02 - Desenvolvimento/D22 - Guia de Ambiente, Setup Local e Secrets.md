# Guia de Ambiente, Setup Local e Secrets — AI-Dani-Admin

## Configuração de Ambiente de Desenvolvimento

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend e DevOps |
| Escopo | Setup completo do ambiente local, variáveis de ambiente, secrets e pré-requisitos para rodar o AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D02 (Stacks), D15 (Arquitetura de Pastas), D17 (Integrações Externas) |

---

> **📌 TL;DR**
>
> - **Pré-requisitos:** Node.js 22+ LTS, Docker Desktop, acesso ao Supabase do projeto, credenciais das integrações externas.
> - **Serviços locais via Docker:** PostgreSQL 17+ + Redis 7.4+ + RabbitMQ 4.x+ — um único `docker-compose up` sobe tudo.
> - **Banco:** Supabase local (via Supabase CLI) ou projeto Supabase cloud de staging.
> - **Setup em 5 passos:** clone → install → `.env` → `docker-compose up` → `prisma migrate dev`.
> - **Secrets:** nunca commitados. `.env` no `.gitignore`. Produção usa Vault/Doppler.
> - **Seed:** `npm run seed` popula configurações padrão (threshold=80, rate_limit=30) para desenvolvimento.

---

## 1. Pré-requisitos

| Ferramenta | Versão Mínima | Verificação |
|---|---|---|
| Node.js | 22.x LTS | `node --version` |
| npm | 10.x+ | `npm --version` |
| Docker Desktop | 4.x+ | `docker --version` |
| Git | 2.x+ | `git --version` |
| Supabase CLI | 1.x+ | `supabase --version` (opcional, para dev local) |

---

## 2. Setup em 5 Passos

### Passo 1 — Clone e Instalação

```bash
# Clone do repositório
git clone https://github.com/shiftlabs/repasse-seguro.git
cd repasse-seguro

# Instalar dependências do monorepo (Turborepo)
npm install

# Navegar para o módulo AI-Dani-Admin
cd apps/api
```

### Passo 2 — Variáveis de Ambiente

```bash
# Copiar o template
cp .env.example .env

# Editar com suas credenciais
# (ver seção 3 para descrição de cada variável)
```

### Passo 3 — Subir Serviços Locais

```bash
# Na raiz do monorepo
docker-compose up -d

# Verificar se os serviços estão rodando
docker-compose ps
# Esperado:
# repasse-postgres   Up   0.0.0.0:5432->5432/tcp
# repasse-redis      Up   0.0.0.0:6379->6379/tcp
# repasse-rabbitmq   Up   0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp
```

### Passo 4 — Migrations e Seed

```bash
# Dentro de apps/api/
npx prisma migrate dev
# ou
npm run db:migrate

# Seed de dados de desenvolvimento
npm run seed
# Cria AgentConfiguration padrão (threshold=80, rate_limit=30) para agentes cadastrados
```

### Passo 5 — Iniciar o Servidor

```bash
# Modo desenvolvimento (hot reload)
npm run start:dev

# Verificar
curl http://localhost:3000/health
# Esperado: {"status":"ok","timestamp":"2026-03-23T..."}
```

---

## 3. Variáveis de Ambiente

### 3.1 Arquivo `.env.example`

```bash
# ============================================================
# AI-Dani-Admin — Variáveis de Ambiente
# ============================================================
# NUNCA commitar o .env. Este arquivo é apenas o template.
# Produção usa Vault/Doppler para gerenciar secrets.
# ============================================================

# App
NODE_ENV=development          # development | staging | production
PORT=3000
LOG_LEVEL=debug               # debug | info | warn | error

# ============================================================
# Database (PostgreSQL via Supabase)
# ============================================================
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/repasse_seguro_dev
# Em staging/produção: usar connection pooling via Supabase pooler
# DATABASE_URL=postgresql://postgres.[project]:[password]@aws-0-sa-east-1.pooler.supabase.com:6543/postgres

# ============================================================
# Supabase
# ============================================================
SUPABASE_URL=http://localhost:54321             # Supabase local
# SUPABASE_URL=https://<project>.supabase.co   # Supabase cloud

SUPABASE_ANON_KEY=eyJ...                       # Chave pública (pode ir para o frontend)
SUPABASE_SERVICE_ROLE_KEY=eyJ...               # Chave privada (NUNCA no frontend)

# ============================================================
# Cache (Redis)
# ============================================================
REDIS_URL=redis://localhost:6379
# Em produção: REDIS_URL=rediss://default:<password>@<host>.upstash.io:6380

# ============================================================
# Filas (RabbitMQ)
# ============================================================
RABBITMQ_URL=amqp://guest:guest@localhost:5672
# Em produção: RABBITMQ_URL=amqps://<user>:<password>@<host>.cloudamqp.com/<vhost>

# ============================================================
# Auth (JWT)
# ============================================================
JWT_SECRET=your-super-secret-jwt-key-min-32-chars
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# ============================================================
# OpenAI
# ============================================================
OPENAI_API_KEY=sk-...
OPENAI_ORG_ID=org-...   # opcional

# ============================================================
# Langfuse (Observabilidade de LLM)
# ============================================================
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com

# ============================================================
# Sentry (Error Tracking)
# ============================================================
SENTRY_DSN=https://...@o....ingest.sentry.io/...
SENTRY_TRACES_SAMPLE_RATE=1.0   # 1.0 em dev, 0.1 em produção

# ============================================================
# PostHog (Analytics + Feature Flags)
# ============================================================
POSTHOG_API_KEY=phc_...
POSTHOG_HOST=https://app.posthog.com

# ============================================================
# Slack (Alertas)
# ============================================================
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/T.../B.../...
# Em dev: usar canal #alertas-dev ou webhook de teste

# ============================================================
# SendGrid (E-mail)
# ============================================================
SENDGRID_API_KEY=SG....
SENDGRID_FROM_EMAIL=alertas@repasseseguro.com.br
ADMIN_ALERT_EMAILS=dev@repasseseguro.com.br   # vírgula-separado para múltiplos

# ============================================================
# Expo Push Notifications
# ============================================================
EXPO_ACCESS_TOKEN=...

# ============================================================
# Seeds
# ============================================================
SEED_ADMIN_ID=00000000-0000-0000-0000-000000000001   # UUID fixo para dev

# ============================================================
# Thresholds Operacionais (RN-DA-031)
# ============================================================
LATENCY_SLA_MS=5000          # Latência máxima aceitável em ms (5s). Alerta disparado se ≥ este valor por 5 min consecutivos.
ERROR_RATE_ALERT_PCT=10      # % de erros em 15 min que dispara alerta
ERROR_RATE_SHUTDOWN_PCT=30   # % de erros em 15 min que dispara desligamento automático
CSAT_ALERT_THRESHOLD=3.5     # Média CSAT abaixo deste valor em 24h dispara alerta
```

### 3.2 Tabela de Classificação de Secrets

| Variável | Sensibilidade | Pode ir ao frontend | Pode ficar em logs |
|---|---|---|---|
| `DATABASE_URL` | Alta | Não | Não |
| `SUPABASE_ANON_KEY` | Baixa | Sim | Não (tem token embutido) |
| `SUPABASE_SERVICE_ROLE_KEY` | Crítica | Nunca | Nunca |
| `REDIS_URL` | Alta | Não | Não |
| `RABBITMQ_URL` | Alta | Não | Não |
| `JWT_SECRET` | Crítica | Nunca | Nunca |
| `OPENAI_API_KEY` | Alta | Não | Não |
| `LANGFUSE_SECRET_KEY` | Alta | Não | Não |
| `LANGFUSE_PUBLIC_KEY` | Baixa | Sim | Sim |
| `SENTRY_DSN` | Média | Sim (frontend usa DSN próprio) | Não |
| `SLACK_WEBHOOK_URL` | Alta | Não | Não |
| `SENDGRID_API_KEY` | Alta | Não | Não |
| `EXPO_ACCESS_TOKEN` | Alta | Não | Não |
| `LATENCY_SLA_MS` | Baixa | Não | Sim |
| `ERROR_RATE_ALERT_PCT` | Baixa | Não | Sim |
| `ERROR_RATE_SHUTDOWN_PCT` | Baixa | Não | Sim |
| `CSAT_ALERT_THRESHOLD` | Baixa | Não | Sim |

---

## 4. Docker Compose

```yaml
# docker-compose.yml — na raiz do monorepo

version: '3.9'

services:
  postgres:
    image: supabase/postgres:17.0.0
    container_name: repasse-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: repasse_seguro_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7.4-alpine
    container_name: repasse-redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:4-management-alpine
    container_name: repasse-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
      RABBITMQ_DEFAULT_VHOST: /
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI: http://localhost:15672
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
```

---

## 5. Scripts NPM

```json
// apps/api/package.json
{
  "scripts": {
    "start:dev": "nest start --watch",
    "start:prod": "node dist/main",
    "build": "nest build",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "db:migrate": "prisma migrate dev",
    "db:migrate:prod": "prisma migrate deploy",
    "db:reset": "prisma migrate reset --force",
    "db:studio": "prisma studio",
    "seed": "ts-node prisma/seed/index.ts",
    "lint": "eslint src --ext .ts",
    "format": "prettier --write src"
  }
}
```

---

## 6. Gestão de Secrets por Ambiente

| Ambiente | Método | Ferramenta |
|---|---|---|
| Desenvolvimento local | `.env` (não commitado) | Editor de texto |
| CI/CD (GitHub Actions) | Repository secrets | GitHub Secrets |
| Staging | Gerenciador de secrets | Doppler ou Vault |
| Produção | Gerenciador de secrets | Doppler ou Vault |

**Regra absoluta:** `JWT_SECRET`, `SUPABASE_SERVICE_ROLE_KEY` e `OPENAI_API_KEY` nunca aparecem em:
- Código-fonte (hardcoded)
- Logs de aplicação
- Output de CI/CD
- Pull requests
- Issues ou comentários

**Rotação de secrets:** toda chave de API deve ser rotacionada a cada 90 dias em produção. Evento de rotação registrado no log de auditoria operacional (não no `admin_access_logs` do produto, mas no log de operações da infra).

---

## 7. Acesso ao RabbitMQ Management UI

Em desenvolvimento, a UI de gerenciamento do RabbitMQ está disponível em:

```
http://localhost:15672
Usuário: guest
Senha: guest
```

Útil para:
- Verificar mensagens nas filas durante desenvolvimento.
- Inspecionar DLQs após falhas de consumer.
- Republicar mensagens da DLQ manualmente.

---

## 8. Verificação de Saúde do Ambiente

```bash
# Verificar todos os serviços
docker-compose ps

# Testar conexão com PostgreSQL
psql postgresql://postgres:postgres@localhost:5432/repasse_seguro_dev -c "SELECT 1"

# Testar Redis
redis-cli ping
# Esperado: PONG

# Testar RabbitMQ
curl -s http://guest:guest@localhost:15672/api/healthchecks/node

# Testar API NestJS
curl http://localhost:3000/health
# Esperado: {"status":"ok"}
```

---

## 9. Troubleshooting Comum

| Problema | Causa | Solução |
|---|---|---|
| `ECONNREFUSED 5432` | PostgreSQL não está rodando | `docker-compose up -d postgres` |
| `ECONNREFUSED 6379` | Redis não está rodando | `docker-compose up -d redis` |
| Migrations falhando | Schema divergente | `npm run db:reset` (destrói dados locais) |
| `PrismaClientInitializationError` | `DATABASE_URL` incorreta | Verificar `.env` e formato da URL |
| RabbitMQ consumer não conecta | `RABBITMQ_URL` incorreta | Verificar formato `amqp://user:pass@host:port` |
| `Invalid API key` (OpenAI) | `OPENAI_API_KEY` expirada ou incorreta | Renovar chave no dashboard OpenAI |
| JWT inválido em testes | `JWT_SECRET` diferente entre serviços | Usar o mesmo `JWT_SECRET` em todos os serviços |

---

## 10. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Setup completo em 5 passos, docker-compose, tabela de secrets, scripts NPM, troubleshooting. |
