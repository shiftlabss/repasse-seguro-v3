# 22 - Guia de Ambiente, Setup Local e Secrets

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — Backend, Frontend |
| **Escopo** | Setup local do módulo CRM no monorepo, variáveis de ambiente e gestão de secrets |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | 02 Stacks CRM |

---

> **TL;DR**
>
> - **Pré-requisitos:** Node.js 22+, pnpm 9+, Docker (para Redis e RabbitMQ locais), Git.
> - **Setup completo em < 20 min** seguindo este guia.
> - **Supabase:** usar projeto de desenvolvimento separado do de produção (Supabase Cloud gratuito é suficiente para dev).
> - **Secrets:** nunca commitados. Usar `.env.local` na raiz do monorepo (ignorado pelo `.gitignore`).
> - **3 ambientes:** `development` (local), `staging` (Vercel preview + Railway staging), `production` (Vercel + Railway main).
> - **CRM é ferramenta interna** — equipe Repasse Seguro (Admin RS, Coordenador RS, Analista RS, Parceiro Externo).

---

## 1. Pré-requisitos

| Ferramenta | Versão mínima | Verificar |
|---|---|---|
| Node.js | 22.x LTS | `node --version` |
| pnpm | 9.x | `pnpm --version` |
| Docker + Docker Compose | 24.x | `docker --version` |
| Git | 2.40+ | `git --version` |

**Instalação do pnpm (se necessário):**
```bash
corepack enable
corepack prepare pnpm@latest --activate
```

**Nota sobre Supabase:** O módulo CRM usa Supabase para banco de dados, autenticação e Realtime. É necessário ter um projeto Supabase de desenvolvimento criado em [supabase.com](https://supabase.com). Supabase Realtime não é emulado localmente — aponte sempre para um projeto real de dev.

---

## 2. Clone e Setup Inicial

```bash
# 1. Clone o monorepo
git clone https://github.com/repasse-seguro/repasse-seguro.git
cd repasse-seguro

# 2. Instalar dependências de todos os packages (Turborepo)
pnpm install

# 3. Copiar template de variáveis de ambiente
cp .env.example .env.local
# Editar .env.local com os valores reais (ver Seção 4)

# 4. Iniciar serviços de infraestrutura local via Docker
docker compose up -d redis rabbitmq

# 5. Executar migrations Prisma para o módulo CRM
pnpm --filter api prisma migrate dev

# 6. Executar seed de desenvolvimento
pnpm --filter api prisma db seed

# 7. Iniciar o CRM em modo desenvolvimento
pnpm --filter web-crm dev
pnpm --filter api dev
```

**O que cada comando inicia:**
- `apps/api` → NestJS API na porta `3001` (módulo CRM em `apps/api/modules/crm/`)
- `apps/web-crm` → Next.js 15 App Router na porta `3000`

---

## 3. Docker Compose — Serviços Locais

O módulo CRM usa Redis (cache de KPIs e blacklist de tokens) e RabbitMQ (filas assíncronas de notificações, alertas de SLA e relatório semanal). PostgreSQL não é rodado localmente — use o Supabase de desenvolvimento.

```yaml
# docker-compose.yml (raiz do monorepo — seção CRM)
services:
  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    command: redis-server --save ""  # Sem persistência em dev

  rabbitmq:
    image: rabbitmq:4-management-alpine
    ports:
      - '5672:5672'    # AMQP
      - '15672:15672'  # Management UI (http://localhost:15672)
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
```

> **Acesso ao RabbitMQ Management UI:** `http://localhost:15672` — usuário `guest`, senha `guest`. Útil para inspecionar filas e dead-letter queues durante o desenvolvimento.

---

## 4. Variáveis de Ambiente

### 4.1 `.env.example` — Template Completo

```bash
# ─── AMBIENTE ──────────────────────────────────────────────────────────────
NODE_ENV=development
PORT=3001
FRONTEND_CRM_URL=http://localhost:3000

# ─── BANCO DE DADOS — SUPABASE ──────────────────────────────────────────────
# Prisma — Connection Pooler (porta 6543) para transações, Direct para migrations
DATABASE_URL=postgresql://postgres.[DEV_PROJECT_ID]:[PASSWORD]@aws-0-sa-east-1.pooler.supabase.com:6543/postgres?pgbouncer=true
DIRECT_URL=postgresql://postgres.[DEV_PROJECT_ID]:[PASSWORD]@aws-0-sa-east-1.pooler.supabase.com:5432/postgres

# Supabase — para Auth, Realtime e Storage
SUPABASE_URL=https://[DEV_PROJECT_ID].supabase.co
SUPABASE_ANON_KEY=[DEV_ANON_KEY]
SUPABASE_SERVICE_ROLE_KEY=[DEV_SERVICE_ROLE_KEY]
# Encontrar no painel Supabase: Settings → API

# ─── REDIS ──────────────────────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379
# Em staging/prod — Upstash (TLS):
# REDIS_URL=rediss://:[TOKEN]@[HOST].upstash.io:6380

# ─── RABBITMQ ───────────────────────────────────────────────────────────────
RABBITMQ_URL=amqp://guest:guest@localhost:5672
# Em staging/prod — CloudAMQP:
# RABBITMQ_URL=amqps://[USER]:[PASS]@[HOST].cloudamqp.com/[VHOST]

# ─── RESEND (E-mail transacional) ───────────────────────────────────────────
RESEND_API_KEY=re_[OBTIDO_EM_resend.com]
RESEND_FROM_EMAIL=noreply@repasseseguro.com.br

# ─── ZAPSIGN (Assinatura eletrônica) ────────────────────────────────────────
ZAPSIGN_API_TOKEN=[OBTIDO_NO_PAINEL_ZAPSIGN]
ZAPSIGN_WEBHOOK_SECRET=[CONFIGURADO_NO_PAINEL_ZAPSIGN]
ZAPSIGN_BASE_URL=https://sandbox.zapsign.com.br/api/v1
# Em prod:
# ZAPSIGN_BASE_URL=https://api.zapsign.com.br/api/v1

# ─── META CLOUD API (WhatsApp Business) ─────────────────────────────────────
META_API_TOKEN=[OBTIDO_NO_META_DEVELOPERS]
META_PHONE_NUMBER_ID=[ID_DO_NUMERO_WHATSAPP_BUSINESS]
META_WABA_ID=[ID_DA_CONTA_WHATSAPP_BUSINESS]
META_WEBHOOK_VERIFY_TOKEN=[TOKEN_CUSTOMIZADO_PARA_VERIFICACAO]
# Webhook de recebimento de mensagens — URL configurada no Meta Developers

# ─── CELCOIN (Conta Escrow) ─────────────────────────────────────────────────
CELCOIN_CLIENT_ID=[OBTIDO_NO_PAINEL_CELCOIN]
CELCOIN_CLIENT_SECRET=[OBTIDO_NO_PAINEL_CELCOIN]
CELCOIN_WEBHOOK_SECRET=[CONFIGURADO_NO_PAINEL_CELCOIN]
CELCOIN_BASE_URL=https://sandbox.openfinance.celcoin.dev
# Em prod:
# CELCOIN_BASE_URL=https://openfinance.celcoin.com.br

# ─── SENTRY (Error tracking) ────────────────────────────────────────────────
SENTRY_DSN=https://[KEY]@o[ORG_ID].ingest.sentry.io/[PROJECT_ID]
# Encontrar em: Sentry → Projeto CRM-API → Settings → Client Keys (DSN)

# ─── POSTHOG (Analytics de produto) ────────────────────────────────────────
# Backend (identificação de servidor)
POSTHOG_API_KEY=phc_[OBTIDO_NO_PAINEL_POSTHOG]
POSTHOG_HOST=https://app.posthog.com

# ─── OPENAI (Fase 2 — Dani-Admin IA) ────────────────────────────────────────
# Não implementar na v1.0. Variável reservada para Fase 2.
# OPENAI_API_KEY=sk-[OBTIDO_EM_platform.openai.com]

# ─── SEGURANÇA ───────────────────────────────────────────────────────────────
# Usado internamente para assinar tokens de contexto e webhooks internos
INTERNAL_HMAC_SECRET=[GERAR_COM: openssl rand -hex 32]

# ─── FRONTEND — Next.js 15 (apps/web-crm) ────────────────────────────────────
# Prefixo NEXT_PUBLIC_ para expor ao browser
NEXT_PUBLIC_API_BASE_URL=http://localhost:3001/v1
NEXT_PUBLIC_SUPABASE_URL=https://[DEV_PROJECT_ID].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[DEV_ANON_KEY]
NEXT_PUBLIC_POSTHOG_KEY=phc_[OBTIDO_NO_PAINEL_POSTHOG]
NEXT_PUBLIC_POSTHOG_HOST=https://app.posthog.com
NEXT_PUBLIC_SENTRY_DSN=https://[KEY]@o[ORG_ID].ingest.sentry.io/[PROJECT_CRM_FRONTEND_ID]
# Em produção — a URL pública do CRM:
# NEXT_PUBLIC_API_BASE_URL=https://api.crm.repasseseguro.com.br/v1
```

### 4.2 Tabela de Secrets por Ambiente

| Secret | Dev (local) | Staging | Produção |
|---|---|---|---|
| `DATABASE_URL` | Supabase dev | Supabase staging | Supabase produção |
| `DIRECT_URL` | Supabase dev | Supabase staging | Supabase produção |
| `REDIS_URL` | Redis local (Docker) | Upstash staging | Upstash produção |
| `RABBITMQ_URL` | RabbitMQ local (Docker) | CloudAMQP staging | CloudAMQP produção |
| `RESEND_API_KEY` | Conta dev Resend | Conta dev Resend | Conta produção Resend |
| `ZAPSIGN_*` | Sandbox ZapSign | Sandbox ZapSign | Produção ZapSign |
| `META_*` | Conta de teste Meta | Conta de produção | Conta de produção |
| `CELCOIN_*` | Sandbox Celcoin | Sandbox Celcoin | Produção Celcoin |
| `SENTRY_DSN` | Projeto dev Sentry | Projeto staging Sentry | Projeto produção Sentry |
| `POSTHOG_API_KEY` | Projeto dev PostHog | Projeto dev PostHog | Projeto produção PostHog |
| `INTERNAL_HMAC_SECRET` | Gerado localmente | Railway secrets | Railway secrets |
| `OPENAI_API_KEY` | N/A (Fase 2) | N/A (Fase 2) | N/A (Fase 2) |

---

## 5. Setup do Banco com Prisma

```bash
# Criar nova migration (após alterar schema.prisma)
pnpm --filter api prisma migrate dev --name nome_da_migration

# Aplicar migrations existentes (staging/prod — via CI/CD)
pnpm --filter api prisma migrate deploy

# Abrir Prisma Studio (GUI de banco)
pnpm --filter api prisma studio

# Regenerar Prisma Client após mudança no schema
pnpm --filter api prisma generate
```

### 5.1 Seed de Dados para Desenvolvimento

O seed cria usuários de teste por papel e dados de exemplo para o CRM:

```bash
# Executar seed completo
pnpm --filter api prisma db seed
```

**Usuários criados pelo seed:**

| E-mail | Senha | Papel |
|---|---|---|
| `admin@crm.dev` | `CrmAdmin@123` | ADMIN_RS |
| `coordenador@crm.dev` | `CrmCoord@123` | COORDENADOR_RS |
| `analista@crm.dev` | `CrmAnalista@123` | ANALISTA_RS |
| `parceiro@crm.dev` | `CrmParceiro@123` | PARCEIRO_EXTERNO |

**Dados de exemplo criados pelo seed:**
- 10 Casos em diferentes estágios do pipeline (CADASTRO → CONCLUIDO)
- 20 Contatos (Cedentes, Cessionários, Parceiros, Incorporadoras)
- Atividades e histórico vinculados aos Casos de exemplo
- Parâmetros do sistema com valores padrão (`system_parameters`)
- Templates de mensagem aprovados (`message_templates`)

---

## 6. Comandos de Desenvolvimento

```bash
# Instalar todas as dependências do monorepo
pnpm install

# Iniciar API (NestJS) e web-crm (Next.js) simultaneamente via Turborepo
pnpm dev

# Iniciar somente o CRM frontend
pnpm --filter web-crm dev

# Iniciar somente a API
pnpm --filter api dev

# Build de produção (todos os apps)
pnpm build

# Build somente do CRM
pnpm --filter web-crm build
pnpm --filter api build

# Rodar testes (todos os packages)
pnpm test

# Rodar testes apenas do módulo CRM (API)
pnpm --filter api test

# Lint de todos os packages
pnpm lint

# Type check
pnpm typecheck

# Adicionar componente shadcn/ui ao web-crm
pnpm --filter web-crm dlx shadcn@latest add [component-name]
```

---

## 7. Gestão de Secrets em Staging e Produção

### 7.1 Railway (API Backend)

```bash
# Instalar Railway CLI
npm install -g @railway/cli

# Autenticar
railway login

# Vincular ao projeto
railway link

# Adicionar secret em staging
railway variables set ZAPSIGN_API_TOKEN=[TOKEN] --environment staging

# Adicionar secret em produção
railway variables set ZAPSIGN_API_TOKEN=[TOKEN] --environment production

# Listar secrets configurados (sem exibir valores)
railway variables
```

### 7.2 Vercel (Frontend web-crm)

```bash
# Instalar Vercel CLI
npm install -g vercel

# Adicionar secret em produção
vercel env add NEXT_PUBLIC_API_BASE_URL production

# Adicionar secret em preview (staging)
vercel env add NEXT_PUBLIC_API_BASE_URL preview

# Listar variáveis
vercel env ls
```

### 7.3 Regras de Segurança de Secrets

- **Nunca** commitar `.env.local` ou qualquer arquivo com valores reais.
- **Nunca** usar secrets de produção em desenvolvimento — especialmente `ZAPSIGN_BASE_URL` de produção.
- **Rotacionar** secrets a cada 90 dias (`INTERNAL_HMAC_SECRET`, tokens de API de integrações).
- **Revogar imediatamente** qualquer secret acidentalmente exposto em commits (use `git filter-branch` ou `git-secrets`).
- **`.gitignore`** deve incluir: `.env`, `.env.local`, `.env.*.local`, `*.key`, `*.pem`.
- **Webhooks externos** (ZapSign, Celcoin, Meta): validar sempre com HMAC-SHA256 usando o `*_WEBHOOK_SECRET` correspondente.

---

## 8. Configuração do VSCode

### 8.1 Extensões Recomendadas (`.vscode/extensions.json`)

```json
{
  "recommendations": [
    "bradlc.vscode-tailwindcss",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "prisma.prisma",
    "ms-vscode.vscode-typescript-next",
    "ms-playwright.playwright"
  ]
}
```

### 8.2 Settings do Workspace (`.vscode/settings.json`)

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "typescript.tsdk": "node_modules/typescript/lib",
  "tailwindCSS.experimental.classRegex": [
    ["cva\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"]
  ]
}
```

---

## 9. Verificação do Setup

Após o setup, verificar se tudo está funcionando:

```bash
# 1. API respondendo
curl http://localhost:3001/health
# Esperado: { "status": "ok", "database": "ok", "redis": "ok", "rabbitmq": "ok" }

# 2. CRM frontend carregando
# Abrir http://localhost:3000 no browser
# Deve exibir a tela de login do CRM

# 3. Login com usuário seed
# Usar admin@crm.dev / CrmAdmin@123
# Deve redirecionar para o painel do CRM

# 4. Verificar Prisma Studio
pnpm --filter api prisma studio
# Abre em http://localhost:5555 — verificar tabelas populadas pelo seed
```

---

## 10. Troubleshooting

| Problema | Causa | Solução |
|---|---|---|
| `ECONNREFUSED 6379` | Redis não iniciado | `docker compose up -d redis` |
| `ECONNREFUSED 5672` | RabbitMQ não iniciado | `docker compose up -d rabbitmq` |
| `P1001: Can't reach database server` | `DATABASE_URL` ou `DIRECT_URL` incorretos | Verificar credenciais no painel Supabase → Settings → Database |
| `Invalid API key` no Supabase | `SUPABASE_SERVICE_ROLE_KEY` incorreta | Verificar em Supabase → Settings → API → service_role key |
| `Cannot find module '@rs/types'` | Package compartilhado não buildado | `pnpm build --filter @rs/types` |
| Prisma Client desatualizado | Schema mudou sem `generate` | `pnpm --filter api prisma generate` |
| `JWT invalid or expired` no frontend | `NEXT_PUBLIC_SUPABASE_ANON_KEY` incorreta | Verificar anon key no painel Supabase |
| Seed falha com constraint error | Banco com dados residuais | `pnpm --filter api prisma migrate reset` (apaga tudo) e refaz seed |
| Porta 3000 ocupada | Outro processo usando a porta | `lsof -ti:3000 | xargs kill -9` ou alterar `PORT` em `.env.local` |
| Realtime não conecta | `SUPABASE_URL` aponta para projeto errado | Confirmar que `NEXT_PUBLIC_SUPABASE_URL` aponta para o projeto dev real |

---

## 11. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — pré-requisitos, Docker Compose (Redis + RabbitMQ), `.env.example` completo com todas as variáveis do CRM, seed de desenvolvimento, comandos de desenvolvimento, gestão de secrets Railway/Vercel, troubleshooting dos 10 problemas mais comuns. |
