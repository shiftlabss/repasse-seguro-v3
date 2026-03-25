# 22 - Guia de Ambiente, Setup Local e Secrets

## Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — Dev, Backend, Frontend, Mobile |
| **Escopo** | Setup local do monorepo, variáveis de ambiente e gestão de secrets |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Dependências** | D02 Stacks · D13 Schema Prisma · D15 Arquitetura de Pastas |

---

> 📌 **TL;DR**
>
> - **Pré-requisitos:** Node.js 22+, pnpm 9+, Docker (para PostgreSQL local), Git.
> - **Setup completo em < 15min** seguindo este guia.
> - **Secrets:** nunca commitados. Usar `.env.local` (ignorado pelo `.gitignore`).
> - **3 ambientes:** `development` (local), `staging` (Railway preview), `production` (Railway main).
> - **Supabase:** usar projeto de desenvolvimento separado do produção.
> - **Seed** executado automaticamente no primeiro `pnpm dev`.

---

## 1. Pré-requisitos

| Ferramenta | Versão mínima | Verificar |
|---|---|---|
| Node.js | 22.x LTS | `node --version` |
| pnpm | 9.x | `pnpm --version` |
| Docker + Docker Compose | 24.x | `docker --version` |
| Git | 2.40+ | `git --version` |
| Expo CLI | SDK 52 | `npx expo --version` |

**Instalação do pnpm (se necessário):**
```bash
corepack enable
corepack prepare pnpm@latest --activate
```

---

## 2. Clone e Setup Inicial

```bash
# 1. Clone o repositório
git clone https://github.com/repasse-seguro/repasse-seguro.git
cd repasse-seguro

# 2. Instalar dependências de todos os packages
pnpm install

# 3. Copiar template de variáveis de ambiente
cp .env.example .env.local

# 4. Preencher variáveis no .env.local (ver Seção 4)
# Editar com editor de preferência

# 5. Iniciar banco de dados PostgreSQL local via Docker
docker compose up -d postgres redis

# 6. Executar migrations e seed
pnpm --filter api prisma migrate dev
pnpm --filter api prisma db seed

# 7. Iniciar todos os apps em modo desenvolvimento
pnpm dev
```

**O que o `pnpm dev` inicia:**
- `apps/api` → NestJS na porta `3001`
- `apps/web` → Vite SPA na porta `5173`
- `apps/mobile` → Expo Dev Server na porta `8081`

---

## 3. Docker Compose — Serviços Locais

```yaml
# docker-compose.yml (raiz do monorepo)
version: '3.9'
services:
  postgres:
    image: postgres:17-alpine
    ports:
      - '5432:5432'
    environment:
      POSTGRES_USER: repasse
      POSTGRES_PASSWORD: repasse_local
      POSTGRES_DB: repasse_seguro_dev
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    command: redis-server --save ""  # Sem persistência em dev

volumes:
  postgres_data:
```

> ⚙️ **Nota:** Supabase Realtime não é emulado localmente. Em desenvolvimento, usar polling de 5s como fallback ou apontar `SUPABASE_URL` para um projeto Supabase de desenvolvimento real.

---

## 4. Variáveis de Ambiente

### 4.1 `.env.example` — Template Completo

```bash
# ─── AMBIENTE ──────────────────────────────────────────────
NODE_ENV=development
PORT=3001
FRONTEND_URL=http://localhost:5173
MOBILE_DEEP_LINK_SCHEME=repasseseguro

# ─── BANCO DE DADOS ────────────────────────────────────────
# Prisma — banco local
DATABASE_URL=postgresql://repasse:repasse_local@localhost:5432/repasse_seguro_dev
DIRECT_URL=postgresql://repasse:repasse_local@localhost:5432/repasse_seguro_dev

# Supabase (desenvolvimento) — para Realtime e Storage
SUPABASE_URL=https://[DEV_PROJECT_ID].supabase.co
SUPABASE_ANON_KEY=[DEV_ANON_KEY]
SUPABASE_SERVICE_ROLE_KEY=[DEV_SERVICE_ROLE_KEY]

# ─── REDIS ──────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379
# Em staging/prod — Upstash:
# REDIS_URL=rediss://:[TOKEN]@[HOST].upstash.io:6380

# ─── JWT ────────────────────────────────────────────────────
JWT_SECRET=[GERAR_COM: openssl rand -hex 32]
JWT_EXPIRES_IN=1h
REFRESH_TOKEN_SECRET=[GERAR_COM: openssl rand -hex 32]
REFRESH_TOKEN_EXPIRES_IN=7d

# ─── RABBITMQ ───────────────────────────────────────────────
RABBITMQ_URL=amqp://guest:guest@localhost:5672
# Em staging/prod — CloudAMQP:
# RABBITMQ_URL=amqps://[USER]:[PASS]@[HOST].cloudamqp.com/[VHOST]

# ─── ZAPSIGN ────────────────────────────────────────────────
ZAPSIGN_API_TOKEN=[OBTIDO_NO_PAINEL_ZAPSIGN]
ZAPSIGN_WEBHOOK_SECRET=[CONFIGURADO_NO_PAINEL_ZAPSIGN]
ZAPSIGN_BASE_URL=https://sandbox.zapsign.com.br/api/v1
# Em prod:
# ZAPSIGN_BASE_URL=https://api.zapsign.com.br/api/v1

# ─── CELCOIN ─────────────────────────────────────────────────
CELCOIN_CLIENT_ID=[OBTIDO_NO_PAINEL_CELCOIN]
CELCOIN_CLIENT_SECRET=[OBTIDO_NO_PAINEL_CELCOIN]
CELCOIN_WEBHOOK_SECRET=[CONFIGURADO_NO_PAINEL_CELCOIN]
CELCOIN_BASE_URL=https://sandbox.openfinance.celcoin.dev
# Em prod:
# CELCOIN_BASE_URL=https://openfinance.celcoin.com.br

# ─── META (WHATSAPP) ─────────────────────────────────────────
META_API_TOKEN=[OBTIDO_NO_META_DEVELOPERS]
META_PHONE_NUMBER_ID=[ID_DO_NUMERO_WHATSAPP_BUSINESS]
META_WABA_ID=[ID_DA_CONTA_WHATSAPP_BUSINESS]

# ─── TWILIO ──────────────────────────────────────────────────
TWILIO_ACCOUNT_SID=[OBTIDO_NO_PAINEL_TWILIO]
TWILIO_AUTH_TOKEN=[OBTIDO_NO_PAINEL_TWILIO]
TWILIO_PHONE_NUMBER=+55119...

# ─── FRONTEND (Vite) ─────────────────────────────────────────
# Prefixo VITE_ para expor ao browser
VITE_API_BASE_URL=http://localhost:3001/v1
VITE_SUPABASE_URL=https://[DEV_PROJECT_ID].supabase.co
VITE_SUPABASE_ANON_KEY=[DEV_ANON_KEY]
```

### 4.2 Tabela de Secrets por Ambiente

| Secret | Dev (local) | Staging | Produção |
|---|---|---|---|
| `DATABASE_URL` | Docker local | Supabase staging | Supabase produção |
| `REDIS_URL` | Redis local | Upstash staging | Upstash produção |
| `RABBITMQ_URL` | RabbitMQ local (Docker) | CloudAMQP | CloudAMQP |
| `ZAPSIGN_*` | Sandbox ZapSign | Sandbox ZapSign | Produção ZapSign |
| `CELCOIN_*` | Sandbox Celcoin | Sandbox Celcoin | Produção Celcoin |
| `META_*` | Conta de teste Meta | Conta de produção | Conta de produção |
| `TWILIO_*` | Conta trial Twilio | Conta trial Twilio | Conta produção Twilio |
| `JWT_SECRET` | Gerado localmente | Railway secrets | Railway secrets |

---

## 5. Gestão de Secrets em Staging e Produção

### 5.1 Railway (Backend e ambiente)

```bash
# Instalar Railway CLI
npm install -g @railway/cli

# Autenticar
railway login

# Adicionar secret em staging
railway variables set JWT_SECRET=$(openssl rand -hex 32) --environment staging

# Listar secrets (sem exibir valores)
railway variables
```

### 5.2 Vercel (Frontend)

```bash
# Instalar Vercel CLI
npm install -g vercel

# Adicionar secret em produção
vercel env add VITE_API_BASE_URL production

# Adicionar secret em preview (staging)
vercel env add VITE_API_BASE_URL preview
```

### 5.3 Regras de Segurança

- **Nunca** commitar `.env.local` ou qualquer arquivo com valores reais.
- **Nunca** usar secrets de produção em desenvolvimento.
- **Rotacionar** secrets a cada 90 dias (JWT_SECRET, tokens de API).
- **Revogar imediatamente** qualquer secret acidentalmente exposto em commits.
- **`.gitignore`** deve incluir: `.env`, `.env.local`, `.env.*.local`, `*.key`, `*.pem`.

---

## 6. Comandos de Desenvolvimento

```bash
# Instalar todas as dependências
pnpm install

# Iniciar todos os apps (web + api + mobile)
pnpm dev

# Iniciar app específico
pnpm --filter web dev
pnpm --filter api dev
pnpm --filter mobile start

# Build de produção
pnpm build

# Rodar testes (todos os packages)
pnpm test

# Rodar testes de um package específico
pnpm --filter api test

# Lint de todos os packages
pnpm lint

# Type check
pnpm typecheck

# Migrations
pnpm --filter api prisma migrate dev        # Nova migration
pnpm --filter api prisma migrate deploy     # Aplicar migrations em staging/prod
pnpm --filter api prisma studio             # Abre Prisma Studio (GUI)
pnpm --filter api prisma db seed            # Executar seed

# Gerar Prisma Client após mudança no schema
pnpm --filter api prisma generate

# Adicionar componente shadcn/ui
pnpm --filter web dlx shadcn@latest add button
```

---

## 7. Configuração do VSCode

### 7.1 Extensões Recomendadas (`.vscode/extensions.json`)

```json
{
  "recommendations": [
    "bradlc.vscode-tailwindcss",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "prisma.prisma",
    "ms-vscode.vscode-typescript-next",
    "expo.vscode-expo-tools"
  ]
}
```

### 7.2 Settings do Workspace (`.vscode/settings.json`)

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

## 8. Verificação do Setup

Após o setup, verificar se tudo está funcionando:

```bash
# 1. API respondendo
curl http://localhost:3001/health
# Esperado: { "status": "ok", "database": "ok", "redis": "ok", "rabbitmq": "ok" }

# 2. SPA carregando
# Abrir http://localhost:5173 no browser

# 3. Seed executado (usuário Master criado)
# Login com: master@repasseseguro.com.br / Change@Me123!
```

---

## 9. Troubleshooting

| Problema | Causa | Solução |
|---|---|---|
| `ECONNREFUSED 5432` | PostgreSQL não iniciado | `docker compose up -d postgres` |
| `ECONNREFUSED 6379` | Redis não iniciado | `docker compose up -d redis` |
| `P1001: Can't reach database server` | DATABASE_URL incorreta | Verificar `DATABASE_URL` no `.env.local` |
| `Cannot find module '@rs/types'` | packages não buildado | `pnpm build --filter @rs/types` |
| Prisma Client desatualizado | Schema mudou | `pnpm --filter api prisma generate` |
| Expo: `Unable to find expo in this project` | Dependências não instaladas | `pnpm install` na raiz |
| `JWT_SECRET not defined` | `.env.local` não copiado | `cp .env.example .env.local` |

---

## 10. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — pré-requisitos, Docker Compose, .env.example completo, comandos dev, VSCode setup, troubleshooting. |
