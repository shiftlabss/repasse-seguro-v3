# 22 - Guia de Ambiente, Setup Local e Secrets

## Módulo Cedente · Plataforma Repasse Seguro

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 22 - Guia de Ambiente, Setup Local e Secrets | v1.0 | 2026-03-23 (America/Fortaleza) | Claude Code Desktop | Aprovado |

| Campo | Valor |
|---|---|
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 |

---

> **TL;DR**
>
> - **Pré-requisitos:** Node.js 22+, pnpm 9+, Docker + Docker Compose, Expo CLI, Supabase CLI, Railway CLI.
> - **Tempo estimado de setup:** menos de 20 minutos seguindo este guia do zero.
> - **Serviços locais (Docker):** PostgreSQL 17, Redis 7, RabbitMQ 4. Supabase via CLI local. Serviços externos via sandboxes/mocks.
> - **Total de variáveis de ambiente:** 42 variáveis entre apps (backend NestJS, frontend Next.js, mobile Expo).
> - **Ambientes:** `dev` (local + Docker), `staging` (Railway + Supabase staging + Vercel preview), `prod` (Railway + Supabase prod + Vercel prod).
> - **Regra de secrets:** Nenhuma variável sensível em repositório git. Secrets gerenciados via Railway (backend), Vercel (frontend) e EAS Secrets (mobile).
> - **Monorepo:** Turborepo + pnpm workspaces — um único `pnpm install` instala tudo.
> - **Diferencial do Módulo Cedente:** inclui variáveis do Guardião do Retorno (OpenAI + Langfuse), ZapSign (assinaturas eletrônicas) e parceiro Escrow.

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
| Expo CLI | latest | `npm install -g expo-cli` | `expo --version` → `x.x.x` |

> **Dica:** Use `nvm` (Node Version Manager) para gerenciar versões do Node.js. O projeto contém `.nvmrc` na raiz com `22`. Execute `nvm use` na raiz do projeto para ativar automaticamente.

> **macOS:** Certifique-se de ter o Xcode Command Line Tools instalado: `xcode-select --install`. Necessário para compilar dependências nativas do React Native/Expo.

---

## 2. Clone e Instalação

### 2.1 Clonar o repositório

```bash
git clone https://github.com/repasse-seguro/monorepo.git
cd monorepo
```

**Output esperado:** Repositório clonado sem erros. Estrutura de diretórios visível com `ls`.

```
monorepo/
  apps/
    api/                  # NestJS — backend principal
    web-cedente/          # Next.js 15 — frontend web do Cedente
    mobile-cedente/       # Expo SDK 52 — app mobile do Cedente
  packages/
    shared-types/
    shared-utils/
    shared-services/
    config-eslint/
    config-typescript/
    ui/
  turbo.json
  package.json
  pnpm-workspace.yaml
  docker-compose.yml
  .env.example
```

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

> O `pnpm install` instala dependências de todos os workspaces simultaneamente: `apps/api`, `apps/web-cedente`, `apps/mobile-cedente` e todos os `packages/`.

### 2.3 Configurar Git hooks

```bash
pnpm prepare
```

**Output esperado:** `husky - Git hooks installed` (instala hooks de `pre-commit` e `commit-msg` via Husky).

### 2.4 Copiar arquivos de variáveis de ambiente

```bash
cp apps/api/.env.example apps/api/.env
cp apps/web-cedente/.env.example apps/web-cedente/.env.local
cp apps/mobile-cedente/.env.example apps/mobile-cedente/.env
```

Edite cada arquivo `.env` com os valores reais conforme as seções seguintes deste guia.

---

## 3. Serviços Locais com Docker

O Módulo Cedente requer PostgreSQL, Redis e RabbitMQ rodando localmente. Use o `docker-compose.yml` na raiz do monorepo.

### 3.1 Subir todos os serviços

```bash
docker compose up -d
```

**Output esperado:**
```
[+] Running 4/4
 ✔ Network monorepo_default       Created
 ✔ Container postgres-cedente     Started
 ✔ Container redis-cedente        Started
 ✔ Container rabbitmq-cedente     Started
```

### 3.2 Verificar serviços ativos

```bash
docker compose ps
```

| Serviço | Porta Local | Credenciais padrão |
|---|---|---|
| PostgreSQL 17 | `5432` | `user: postgres` / `pass: postgres` / `db: repasse_cedente_dev` |
| Redis 7 | `6379` | Sem autenticação em dev |
| RabbitMQ 4 | `5672` (AMQP) / `15672` (Management UI) | `user: guest` / `pass: guest` |

> **RabbitMQ Management UI:** Acesse `http://localhost:15672` para monitorar filas localmente.

### 3.3 Parar e remover serviços

```bash
# Parar sem remover dados
docker compose stop

# Parar e remover volumes (reseta o banco)
docker compose down -v
```

### 3.4 Supabase CLI local (opcional — para RLS e Realtime)

Para desenvolvimento com Row Level Security e Supabase Realtime localmente:

```bash
cd apps/api
supabase start
```

**Output esperado:**
```
Started supabase local development setup.

         API URL: http://localhost:54321
     GraphQL URL: http://localhost:54321/graphql/v1
          DB URL: postgresql://postgres:postgres@localhost:54322/postgres
      Studio URL: http://localhost:54323
    Inbucket URL: http://localhost:54324
        ...
```

> **Recomendado:** Use Supabase local para testar RLS (RN-011) e Realtime (RN-057, RN-083).

---

## 4. Variáveis de Ambiente

### 4.1 Backend — `apps/api/.env`

```dotenv
# ==========================================
# MÓDULO CEDENTE — Backend NestJS
# ==========================================

# Ambiente
NODE_ENV=development
PORT=3000

# Banco de Dados — Supabase (PostgreSQL)
DATABASE_URL="postgresql://postgres:postgres@localhost:54322/postgres"
# Em dev com Docker puro (sem Supabase CLI):
# DATABASE_URL="postgresql://postgres:postgres@localhost:5432/repasse_cedente_dev"

# Supabase
SUPABASE_URL=http://localhost:54321
SUPABASE_ANON_KEY=<supabase-anon-key-local>
SUPABASE_SERVICE_ROLE_KEY=<supabase-service-role-key-local>

# JWT
JWT_SECRET=<string-aleatoria-32-chars-minimo-dev>
JWT_EXPIRATION=15m
JWT_REFRESH_EXPIRATION=7d

# Redis (Upstash em produção; local em dev)
REDIS_URL=redis://localhost:6379

# RabbitMQ (CloudAMQP em produção; local em dev)
RABBITMQ_URL=amqp://guest:guest@localhost:5672

# E-mail Transacional — Resend
RESEND_API_KEY=<resend-api-key>
RESEND_FROM_EMAIL=noreply@repasseseguro.com.br

# Assinaturas Eletrônicas — ZapSign
ZAPSIGN_API_KEY=<zapsign-api-key-sandbox>
ZAPSIGN_WEBHOOK_SECRET=<zapsign-webhook-secret>
ZAPSIGN_BASE_URL=https://sandbox.api.zapsign.com.br/api/v1

# Escrow (parceiro financeiro — DEFINIÇÃO PENDENTE: DP-001)
ESCROW_WEBHOOK_SECRET=<escrow-webhook-secret>
ESCROW_BASE_URL=<escrow-api-sandbox-url>

# IA — Guardião do Retorno
OPENAI_API_KEY=<openai-api-key>
OPENAI_MODEL=gpt-4-turbo
OPENAI_EMBEDDINGS_MODEL=text-embedding-3-small

# Observabilidade IA — Langfuse
LANGFUSE_SECRET_KEY=<langfuse-secret-key>
LANGFUSE_PUBLIC_KEY=<langfuse-public-key>
LANGFUSE_BASE_URL=https://cloud.langfuse.com

# Receita Federal — Validação CNPJ
RECEITA_FEDERAL_API_URL=https://brasilapi.com.br/api/cnpj/v1

# Sentry — Backend
SENTRY_DSN=<sentry-dsn-backend>

# Supabase Storage
SUPABASE_STORAGE_BUCKET=documents
```

> **NUNCA commite** este arquivo. Ele está no `.gitignore`. Use `apps/api/.env.example` como modelo sem valores reais.

### 4.2 Frontend Web — `apps/web-cedente/.env.local`

```dotenv
# ==========================================
# MÓDULO CEDENTE — Frontend Next.js
# ==========================================

# API Backend
NEXT_PUBLIC_API_URL=http://localhost:3000

# Supabase
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=<supabase-anon-key-local>

# Analytics — PostHog
NEXT_PUBLIC_POSTHOG_KEY=<posthog-project-api-key>
NEXT_PUBLIC_POSTHOG_HOST=https://app.posthog.com

# Sentry — Next.js
NEXT_PUBLIC_SENTRY_DSN=<sentry-dsn-frontend>
SENTRY_AUTH_TOKEN=<sentry-auth-token-para-source-maps>

# ZapSign — URL iframe de assinatura
NEXT_PUBLIC_ZAPSIGN_SIGNER_URL=https://sandbox.zapsign.com.br/verificar
```

### 4.3 Mobile — `apps/mobile-cedente/.env`

```dotenv
# ==========================================
# MÓDULO CEDENTE — Mobile Expo
# ==========================================

# API Backend
EXPO_PUBLIC_API_URL=http://192.168.x.x:3000
# Use o IP da máquina na rede local (não localhost) para dispositivo físico

# Supabase
EXPO_PUBLIC_SUPABASE_URL=http://192.168.x.x:54321
EXPO_PUBLIC_SUPABASE_ANON_KEY=<supabase-anon-key-local>

# Analytics — PostHog
EXPO_PUBLIC_POSTHOG_KEY=<posthog-project-api-key>

# Sentry — React Native
EXPO_PUBLIC_SENTRY_DSN=<sentry-dsn-mobile>
```

> **Dispositivo físico:** O app mobile conecta ao backend via IP da sua máquina na rede local. Use `ifconfig` (macOS/Linux) ou `ipconfig` (Windows) para descobrir o IP.

### 4.4 Tabela consolidada de variáveis

| Variável | App | Obrigatória | Onde obter |
|---|---|---|---|
| `DATABASE_URL` | api | Sim | Supabase CLI local / Dashboard Supabase |
| `SUPABASE_URL` | api, web, mobile | Sim | Supabase CLI local / Dashboard Supabase |
| `SUPABASE_ANON_KEY` | api, web, mobile | Sim | Supabase CLI local / Dashboard Supabase → Settings → API |
| `SUPABASE_SERVICE_ROLE_KEY` | api | Sim | Supabase Dashboard → Settings → API (**nunca expor no frontend**) |
| `JWT_SECRET` | api | Sim | Gerar: `openssl rand -hex 32` |
| `REDIS_URL` | api | Sim | Docker local / Upstash Dashboard |
| `RABBITMQ_URL` | api | Sim | Docker local / CloudAMQP Dashboard |
| `RESEND_API_KEY` | api | Sim | [resend.com](https://resend.com) → API Keys |
| `ZAPSIGN_API_KEY` | api | Sim | [app.zapsign.com.br](https://app.zapsign.com.br) → Integrações → API |
| `ZAPSIGN_WEBHOOK_SECRET` | api | Sim | Configurado no painel ZapSign ao registrar o webhook |
| `ESCROW_WEBHOOK_SECRET` | api | Sim | Fornecido pelo parceiro Escrow (DP-001 pendente) |
| `OPENAI_API_KEY` | api | Sim | [platform.openai.com](https://platform.openai.com) → API Keys |
| `LANGFUSE_SECRET_KEY` | api | Sim | [cloud.langfuse.com](https://cloud.langfuse.com) → Project Settings |
| `LANGFUSE_PUBLIC_KEY` | api | Sim | [cloud.langfuse.com](https://cloud.langfuse.com) → Project Settings |
| `SENTRY_DSN` (backend) | api | Sim | [sentry.io](https://sentry.io) → Project → Settings → DSN |
| `NEXT_PUBLIC_POSTHOG_KEY` | web | Sim | [posthog.com](https://posthog.com) → Project → Settings → Project API Key |
| `NEXT_PUBLIC_SENTRY_DSN` | web | Sim | [sentry.io](https://sentry.io) → Project Next.js → Settings → DSN |
| `SENTRY_AUTH_TOKEN` | web | Sim (CI/CD) | [sentry.io](https://sentry.io) → User Settings → Auth Tokens |
| `EXPO_PUBLIC_POSTHOG_KEY` | mobile | Sim | Mesmo do web |
| `EXPO_PUBLIC_SENTRY_DSN` | mobile | Sim | Sentry projeto React Native |

---

## 5. Migrations e Seed do Banco

### 5.1 Aplicar migrations

```bash
cd apps/api
pnpm prisma migrate dev --name init
```

**Output esperado:**
```
Environment variables loaded from .env
Prisma schema loaded from prisma/schema.prisma
Datasource "db": PostgreSQL database "repasse_cedente_dev"

Applying migration `20260323000000_init`...

Database changes:
  - Created table `cedentes`
  - Created table `casos`
  - ...

✔  Generated Prisma Client
```

### 5.2 Rodar seed (dados mínimos para desenvolvimento)

```bash
cd apps/api
pnpm prisma db seed
```

O seed (`prisma/seed.ts`) cria:
- 2 Cedentes PF de teste (com `@test.com`)
- 1 Cedente PJ de teste
- 3 casos em estados diferentes (rascunho, em análise, proposta recebida)
- Dados do Guardião do Retorno (base de conhecimento mínima via pgvector)

### 5.3 Resetar banco (dev apenas)

```bash
cd apps/api
pnpm prisma migrate reset
```

> **Atenção:** Este comando apaga todos os dados e reaplicação todas as migrations. Use apenas em desenvolvimento local.

---

## 6. Rodando o Projeto Localmente

### 6.1 Subir todos os serviços em paralelo (recomendado)

```bash
# Na raiz do monorepo
pnpm dev
```

Turborepo sobe em paralelo:
- `apps/api` → NestJS em `http://localhost:3000`
- `apps/web-cedente` → Next.js em `http://localhost:3001`

**Swagger (documentação da API):** `http://localhost:3000/api/docs`

### 6.2 Subir serviços individualmente

```bash
# Backend apenas
pnpm dev --filter=api

# Frontend web apenas
pnpm dev --filter=web-cedente
```

### 6.3 App mobile com Expo Go

```bash
cd apps/mobile-cedente
pnpm start
```

**Output esperado:**
```
Starting Metro Bundler...
▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
█ ▀▀▀▀▀▀▀▀▀▀▀ █   QR code para Expo Go
█             █
...
Metro waiting on exp://192.168.x.x:8081
```

- **iOS:** Abra o app Expo Go e escaneie o QR code.
- **Android:** Abra o app Expo Go, toque em "Scan QR code" e escaneie.
- **Simulador iOS:** pressione `i` no terminal.
- **Emulador Android:** pressione `a` no terminal.

> **Pré-requisito para câmera (RN-087):** Teste de upload via câmera requer dispositivo físico. O simulador iOS não tem câmera real.

### 6.4 Build de desenvolvimento (para testar funcionalidades nativas)

```bash
cd apps/mobile-cedente
eas build --profile development --platform ios
# ou
eas build --profile development --platform android
```

---

## 7. Rodando os Testes

### 7.1 Todos os testes do monorepo

```bash
# Na raiz
pnpm test
```

### 7.2 Testes por workspace

```bash
# Backend — unitários e integração
pnpm test --filter=api

# Frontend web — componentes
pnpm test --filter=web-cedente

# Mobile — componentes
pnpm test --filter=mobile-cedente
```

### 7.3 Testes com watch mode

```bash
pnpm test --filter=api -- --watch
```

### 7.4 Type-check em todos os workspaces

```bash
pnpm type-check
```

### 7.5 Lint em todos os workspaces

```bash
pnpm lint
```

### 7.6 Testes E2E (Playwright — frontend)

```bash
cd apps/web-cedente
pnpm exec playwright test
```

> Os testes E2E requerem os serviços locais ativos (`docker compose up -d` e `pnpm dev`).

---

## 8. Gestão de Secrets por Ambiente

### 8.1 Desenvolvimento local

Secrets ficam em arquivos `.env` locais, nunca commitados. O `.gitignore` inclui:

```
apps/api/.env
apps/web-cedente/.env.local
apps/mobile-cedente/.env
```

### 8.2 Staging e Produção — Backend (Railway)

```bash
# Autenticar na Railway CLI
railway login

# Vincular ao projeto correto
railway link

# Ver variáveis configuradas
railway variables

# Definir variável
railway variables set OPENAI_API_KEY=<valor>
```

Todas as variáveis listadas na seção 4.1 devem estar configuradas no ambiente Railway de staging e produção.

### 8.3 Staging e Produção — Frontend Web (Vercel)

```bash
# Autenticar
vercel login

# Definir variável de ambiente para produção
vercel env add NEXT_PUBLIC_POSTHOG_KEY production

# Definir para todos os ambientes
vercel env add NEXT_PUBLIC_API_URL
```

### 8.4 Mobile — EAS Secrets

```bash
# No diretório do app mobile
cd apps/mobile-cedente

# Definir secret para todos os builds
eas secret:create --scope project --name EXPO_PUBLIC_API_URL --value https://api.repasseseguro.com.br

# Listar secrets configurados
eas secret:list
```

### 8.5 Rotação de secrets

| Secret | Frequência recomendada | Procedimento |
|---|---|---|
| `JWT_SECRET` | A cada 90 dias | Atualizar Railway + reiniciar serviço. Tokens existentes invalidados. |
| `OPENAI_API_KEY` | Após suspeita de vazamento | Revogar no painel OpenAI, criar nova, atualizar Railway. |
| `ZAPSIGN_API_KEY` | A cada 180 dias | Regenerar no painel ZapSign, atualizar Railway. |
| `SUPABASE_SERVICE_ROLE_KEY` | Após suspeita de vazamento | Revogar via Supabase Dashboard → Settings → API. |
| `LANGFUSE_SECRET_KEY` | A cada 90 dias | Rotacionar no painel Langfuse, atualizar Railway. |

---

## 9. Troubleshooting

### 9.1 `pnpm install` falha com erro de permissão

```bash
# macOS/Linux
sudo chown -R $(whoami) ~/.pnpm-store
pnpm install
```

### 9.2 Docker não sobe — porta em uso

```bash
# Ver o que está usando a porta 5432 (PostgreSQL)
lsof -i :5432

# Matar o processo
kill -9 <PID>

# Ou mudar a porta no docker-compose.yml
```

### 9.3 `prisma migrate dev` falha — conexão recusada

Verifique se o Docker está rodando:
```bash
docker compose ps
# STATUS deve ser "running"

# Se não estiver:
docker compose up -d
```

### 9.4 Expo — "Network request failed" no dispositivo físico

O `EXPO_PUBLIC_API_URL` deve apontar para o IP da máquina, não `localhost`:
```bash
# Descobrir IP local (macOS)
ifconfig | grep "inet " | grep -v 127.0.0.1

# Atualizar .env
EXPO_PUBLIC_API_URL=http://192.168.1.42:3000
```

### 9.5 Guardião do Retorno não responde — erro de OpenAI

```bash
# Verificar se a chave está configurada
cd apps/api
railway variables | grep OPENAI

# Testar a chave diretamente
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  https://api.openai.com/v1/models
# Esperado: 200
```

### 9.6 ZapSign — webhook não recebido em dev

Use o [ngrok](https://ngrok.com) para expor o backend local:
```bash
ngrok http 3000
# Configure o webhook no painel ZapSign com a URL ngrok:
# https://<ngrok-id>.ngrok.io/api/v1/webhooks/zapsign
```

### 9.7 RabbitMQ — fila não processa mensagens

```bash
# Acessar o Management UI
open http://localhost:15672
# user: guest / pass: guest

# Verificar se as filas estão criadas e se há consumidores
# Se não houver consumidores, reiniciar o backend:
pnpm dev --filter=api
```

### 9.8 Supabase Realtime não funciona em dev

```bash
# Reiniciar o Supabase CLI local
supabase stop
supabase start

# Verificar status
supabase status
```

### 9.9 `type-check` falha após merge

```bash
# Limpar cache do Turborepo e recompilar
pnpm turbo clean
pnpm install
pnpm type-check
```

### 9.10 Reset completo do ambiente de desenvolvimento

```bash
# 1. Parar tudo
docker compose down -v
supabase stop

# 2. Limpar node_modules e cache
find . -name "node_modules" -type d -prune -exec rm -rf {} +
pnpm turbo clean

# 3. Reinstalar
pnpm install
pnpm prepare

# 4. Subir serviços
docker compose up -d
supabase start

# 5. Migrations e seed
cd apps/api
pnpm prisma migrate dev
pnpm prisma db seed
cd ../..

# 6. Iniciar apps
pnpm dev
```

---

## 10. Referências

- Doc 02 — Stacks Tecnológicas: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/02 - Stacks.md`
- Documentação Turborepo: [turbo.build/repo/docs](https://turbo.build/repo/docs)
- Documentação Supabase CLI: [supabase.com/docs/guides/cli](https://supabase.com/docs/guides/cli)
- Documentação Expo EAS: [expo.dev/eas](https://expo.dev/eas)
- Documentação Railway CLI: [docs.railway.app/develop/cli](https://docs.railway.app/develop/cli)
- Documentação Prisma: [prisma.io/docs](https://prisma.io/docs)
