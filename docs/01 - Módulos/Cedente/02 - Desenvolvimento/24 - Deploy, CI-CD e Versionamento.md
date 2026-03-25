# 24 - Deploy, CI/CD e Versionamento

## Módulo Cedente · Plataforma Repasse Seguro

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 24 - Deploy, CI/CD e Versionamento | v1.0 | 2026-03-23 (America/Fortaleza) | Claude Code Desktop | Aprovado |

| Campo | Valor |
|---|---|
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 |

---

> **TL;DR**
>
> - **Pipeline CI:** PR abre `ci.yml` → lint + type-check + testes + build. Obrigatório antes do merge.
> - **Deploy staging:** merge em `develop` → deploy automático: Railway staging + Vercel preview.
> - **Deploy produção:** tag `v*.*.*` em `main` → deploy automático: Railway prod + Vercel prod.
> - **Mobile:** EAS Build por perfil (`development`, `preview`, `production`). EAS Update para OTA.
> - **Rollback:** `railway rollback` < 60s para backend; Vercel tem rollback instantâneo no dashboard.
> - **Migrations:** `pnpm prisma migrate deploy` executa antes do deploy de backend, sempre.
> - **Migrations destrutivas:** processo obrigatório em 2 etapas (expand/contract).
> - **Versionamento:** Semantic Versioning (MAJOR.MINOR.PATCH). Tags git no `main`.

---

## 1. Ambientes

| Ambiente | Backend (Railway) | Frontend Web (Vercel) | Banco (Supabase) | Mobile |
|---|---|---|---|---|
| **Desenvolvimento local** | `http://localhost:3000` | `http://localhost:3001` | Supabase CLI local | Expo Go (dev build) |
| **Staging** | `https://api-cedente-staging.up.railway.app` | Preview Vercel por PR | Projeto Supabase staging | Preview build (EAS) |
| **Produção** | `https://api-cedente.up.railway.app` | `https://cedente.repasseseguro.com.br` | Projeto Supabase prod | App Store / Google Play |

**Regra dos 3 projetos Supabase:** desenvolvimento local, staging e produção são projetos separados. Nunca compartilhar projetos entre ambientes.

---

## 2. Pipeline de CI/CD — GitHub Actions

### 2.1 Diagrama do fluxo

```
Push para feature/* ou fix/*
        │
        ▼
  ci.yml (PR check)
  ┌─────────────────┐
  │ 1. pnpm install │
  │ 2. lint         │
  │ 3. type-check   │
  │ 4. test         │
  │ 5. build        │
  └────────┬────────┘
           │ ✅ verde
           ▼
     Merge em develop
           │
           ▼
  staging.yml (deploy staging)
  ┌─────────────────────────────────────┐
  │ 1. pnpm install                     │
  │ 2. prisma migrate deploy (staging)  │
  │ 3. Railway deploy (staging)         │
  │ 4. Vercel deploy (preview)          │
  └─────────────────────────────────────┘

     Tag v*.*.* em main
           │
           ▼
  production.yml (deploy produção)
  ┌─────────────────────────────────────┐
  │ 1. pnpm install                     │
  │ 2. prisma migrate deploy (prod)     │
  │ 3. Railway deploy (prod)            │
  │ 4. Vercel promote (prod)            │
  └─────────────────────────────────────┘
```

### 2.2 Workflow: `ci.yml` (PR check)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [develop, main]

jobs:
  ci:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Lint
        run: pnpm lint

      - name: Type-check
        run: pnpm type-check

      - name: Test
        run: pnpm test
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/repasse_test
          # Outras env vars de teste via GitHub Secrets

      - name: Build
        run: pnpm build
        env:
          NEXT_PUBLIC_API_URL: https://api-cedente-staging.up.railway.app
```

### 2.3 Workflow: `staging.yml` (deploy staging)

```yaml
# .github/workflows/staging.yml
name: Deploy Staging

on:
  push:
    branches: [develop]

jobs:
  deploy-backend-staging:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run Prisma migrations (staging)
        run: pnpm prisma migrate deploy
        working-directory: apps/api
        env:
          DATABASE_URL: ${{ secrets.SUPABASE_STAGING_DATABASE_URL }}

      - name: Deploy to Railway (staging)
        uses: bervProject/railway-deploy@main
        with:
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
          service: cedente-api-staging

  deploy-frontend-staging:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy to Vercel (preview)
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID_CEDENTE }}
          working-directory: apps/web-cedente
```

### 2.4 Workflow: `production.yml` (deploy produção)

```yaml
# .github/workflows/production.yml
name: Deploy Production

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'

jobs:
  deploy-backend-production:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    environment: production  # requer aprovação manual configurada no GitHub

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run Prisma migrations (production)
        run: pnpm prisma migrate deploy
        working-directory: apps/api
        env:
          DATABASE_URL: ${{ secrets.SUPABASE_PROD_DATABASE_URL }}

      - name: Deploy to Railway (production)
        uses: bervProject/railway-deploy@main
        with:
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
          service: cedente-api-prod

  deploy-frontend-production:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    needs: deploy-backend-production

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy to Vercel (production)
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID_CEDENTE }}
          vercel-args: '--prod'
          working-directory: apps/web-cedente
```

### 2.5 Secrets necessários no GitHub

| Secret | Descrição |
|---|---|
| `RAILWAY_TOKEN` | Token de serviço do Railway CLI |
| `SUPABASE_STAGING_DATABASE_URL` | URL de conexão do banco de dados staging |
| `SUPABASE_PROD_DATABASE_URL` | URL de conexão do banco de dados produção |
| `VERCEL_TOKEN` | Token de API do Vercel |
| `VERCEL_ORG_ID` | ID da organização no Vercel |
| `VERCEL_PROJECT_ID_CEDENTE` | ID do projeto Cedente no Vercel |
| `SENTRY_AUTH_TOKEN` | Token para upload de source maps ao Sentry |

---

## 3. Deploy por Camada

### 3.1 Backend NestJS — Railway

**Estratégia:** zero-downtime via rolling deploy. Railway cria uma nova instância, aguarda o health check passar, e então remove a instância antiga.

```bash
# Configurar health check no Railway
# Path: /health
# Timeout: 30s
# Interval: 10s
```

**Health check endpoint (obrigatório no NestJS):**
```typescript
// apps/api/src/health/health.controller.ts
@Controller('health')
export class HealthController {
  @Get()
  check(): { status: string; timestamp: string } {
    return {
      status: 'ok',
      timestamp: new Date().toISOString(),
    };
  }
}
```

**Configurações Railway obrigatórias:**
- Restart policy: `on-failure` com máximo 3 tentativas.
- Memória mínima: 512 MB.
- CPU: 0.5 vCPU mínimo.
- Região: `us-east` (mais próxima do Brasil com melhor latência).

**Deploy manual via Railway CLI:**
```bash
# Autenticar
railway login

# Vincular ao projeto
railway link

# Deploy do ambiente correto
railway up --service cedente-api-staging     # staging
railway up --service cedente-api-prod        # produção (raramente manual)
```

### 3.2 Frontend Web — Vercel

**Estratégia:** Vercel gerencia zero-downtime nativamente via Edge Network. Cada deploy cria uma URL imutável.

- **Preview por PR:** automático via integração GitHub. URL única por PR: `cedente-git-<branch>.vercel.app`.
- **Produção:** `cedente.repasseseguro.com.br` aponta para o último deploy de `main`.

**Deploy manual via Vercel CLI:**
```bash
# Autenticar
vercel login

# Preview deploy
vercel

# Deploy de produção
vercel --prod
```

**Variáveis de ambiente por ambiente:**
```bash
# Adicionar variável de produção
vercel env add NEXT_PUBLIC_API_URL production

# Listar variáveis
vercel env ls

# Remover variável
vercel env rm NEXT_PUBLIC_API_URL production
```

### 3.3 Mobile — EAS Build + EAS Update

#### 3.3.1 Perfis de build

```json
// apps/mobile-cedente/eas.json
{
  "cli": {
    "version": ">= 7.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": { "simulator": true },
      "android": { "buildType": "apk" }
    },
    "preview": {
      "distribution": "internal",
      "ios": { "simulator": false },
      "android": { "buildType": "apk" },
      "channel": "preview"
    },
    "production": {
      "distribution": "store",
      "ios": { "buildConfiguration": "Release" },
      "android": { "buildType": "app-bundle" },
      "channel": "production"
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "<apple-id>",
        "ascAppId": "<app-store-connect-app-id>",
        "appleTeamId": "<apple-team-id>"
      },
      "android": {
        "serviceAccountKeyPath": "./google-play-service-account.json",
        "track": "internal"
      }
    }
  }
}
```

#### 3.3.2 Build e submit

```bash
cd apps/mobile-cedente

# Build de desenvolvimento (simulador iOS)
eas build --profile development --platform ios

# Build de preview (device físico, distribuição interna)
eas build --profile preview --platform all

# Build de produção (App Store + Google Play)
eas build --profile production --platform all

# Submit para as stores (após build de produção)
eas submit --platform ios
eas submit --platform android
```

#### 3.3.3 OTA Updates (EAS Update)

OTA updates permitem atualizar o JavaScript do app sem passar pela store, para correções que não alteram código nativo.

```bash
# Publicar update para o canal de preview
eas update --channel preview --message "fix(cedente-propostas): corrigir timer regressivo"

# Publicar update para produção
eas update --channel production --message "fix(cedente-assinaturas): corrigir exibição de status ZapSign"
```

**Restrições de OTA:**
- Apenas alterações em JavaScript/TypeScript.
- Mudanças em dependências nativas, `app.json`, permissões ou assets nativos exigem novo build de store.

---

## 4. Versionamento Semântico

### 4.1 Formato: MAJOR.MINOR.PATCH

| Componente | Quando incrementar | Exemplo |
|---|---|---|
| **MAJOR** | Breaking change na API pública, migração destrutiva de banco, mudança de protocolo | `1.0.0` → `2.0.0` |
| **MINOR** | Nova funcionalidade compatível com versão anterior | `1.0.0` → `1.1.0` |
| **PATCH** | Correção de bug, hotfix, atualização de dependência sem breaking change | `1.0.0` → `1.0.1` |

### 4.2 Criação de release

```bash
# 1. Garantir que develop está atualizado e CI está verde
git checkout develop
git pull origin develop

# 2. Criar branch de release
git checkout -b release/1.2.0

# 3. Atualizar versão no package.json raiz e apps
# (usar sed ou manualmente)
npm version 1.2.0 --no-git-tag-version

# 4. Commit de versão
git add .
git commit -m "chore(release): versão 1.2.0"

# 5. PR para main
gh pr create --base main --title "chore(release): versão 1.2.0"

# 6. Após merge, criar tag no main
git checkout main
git pull origin main
git tag -a v1.2.0 -m "Release v1.2.0 — <descrição>"
git push origin v1.2.0

# 7. Merge de volta para develop
git checkout develop
git merge --no-ff main
git push origin develop
```

A tag `v1.2.0` dispara automaticamente o workflow `production.yml`.

### 4.3 Changelog

Manter `CHANGELOG.md` na raiz do monorepo. Formato [Keep a Changelog](https://keepachangelog.com):

```markdown
# Changelog

## [1.2.0] - 2026-03-23

### Adicionado
- Guardião do Retorno: pipeline RAG para base de conhecimento (RN-058 a RN-063)
- Fluxo de escalonamento de cenário com cooldown de 7 dias (RN-025 a RN-027)

### Corrigido
- Timer regressivo de proposta não atualizando após refresh (RN-030)

### Alterado
- Upload de documentos: mensagem de erro mais clara para MIME type inválido (RN-042)

## [1.1.0] - 2026-03-22

### Adicionado
- ...
```

---

## 5. Migrations Prisma em Produção

### 5.1 Fluxo seguro de migrations

Migrations são a operação de maior risco em produção. Seguir o processo abaixo sem exceções:

```
1. Desenvolver migration em dev local
   ↓
2. Testar em staging (pnpm prisma migrate deploy)
   ↓
3. Validar: dados íntegros, performance de queries, RLS aplicado
   ↓
4. Deploy em produção via pipeline CI (não manual)
   ↓
5. Monitorar Supabase Dashboard e Railway logs por 30 min
```

### 5.2 Migrations não-destrutivas (padrão)

Adição de coluna nullable, criação de tabela, adição de índice: podem ser deployadas diretamente.

```bash
# Criar migration
cd apps/api
pnpm prisma migrate dev --name add_guardiao_session_table

# O pipeline de CI aplica automaticamente em staging/prod:
pnpm prisma migrate deploy
```

### 5.3 Migrations destrutivas — processo obrigatório em 2 etapas (Expand/Contract)

Renomear coluna, remover coluna, mudar tipo de coluna: **nunca em um único deploy**.

**Exemplo: renomear coluna `status` para `status_caso`**

**Etapa 1 — Expand (adicionar nova coluna):**
```prisma
// Migration 1: adicionar coluna nova (manter a antiga)
model Caso {
  id          String @id
  status      String  // ← manter por enquanto
  status_caso String? // ← adicionar nullable
}
```

Deploy da Etapa 1. Código novo escreve em ambas as colunas. Código antigo lê apenas `status`.

**Etapa 2 — Contract (remover coluna antiga):**
```prisma
// Migration 2: após confirmar que nenhum código usa 'status' mais
model Caso {
  id          String @id
  // status   ← removido
  status_caso String // ← tornar required
}
```

Deploy da Etapa 2.

> **Regra:** entre as etapas 1 e 2, deve haver pelo menos um ciclo de deploy + monitoramento de 24h em produção.

### 5.4 Reverter migration em emergência

**Importante:** Prisma não tem rollback automático de migrations. Prepare sempre um script de reversão manual.

```bash
# Criar migration de reversão manualmente
cd apps/api
pnpm prisma migrate dev --name revert_add_coluna_xpto

# Aplicar em produção via pipeline (nunca manualmente)
```

---

## 6. Rollback

### 6.1 Rollback de Backend (Railway)

```bash
# Via Railway CLI — reverter para o deploy anterior
railway rollback --service cedente-api-prod

# Via Railway Dashboard
# 1. Acessar railway.app/dashboard
# 2. Selecionar projeto → service cedente-api-prod
# 3. Deployments → selecionar deploy anterior → "Rollback"
```

**Tempo esperado:** < 60 segundos para o serviço retornar online no deploy anterior.

**Atenção com migrations:** Se o rollback de código envolver reverter uma migration, é necessário:
1. Fazer rollback do código (Railway).
2. Executar manualmente a migration de reversão no banco via `psql` ou Supabase SQL editor.

### 6.2 Rollback de Frontend (Vercel)

```bash
# Via Vercel CLI
vercel rollback [deployment-url]

# Via Vercel Dashboard
# 1. Acessar vercel.com/dashboard
# 2. Projeto cedente-web → Deployments
# 3. Selecionar deploy anterior → "Promote to Production"
```

**Tempo esperado:** instantâneo — Vercel redireciona o domínio para o deploy anterior sem downtime.

### 6.3 Rollback de Mobile (EAS Update)

```bash
# Publicar update que aponta para o bundle anterior
eas update --channel production --message "revert: rollback para v1.1.0"
# (publicar o bundle da versão anterior)
```

Para rollback completo de store (mudanças nativas): requer novo build e submissão, o que pode levar horas (Apple) ou minutos (Google). Planejar com antecedência.

### 6.4 Decisão de rollback

| Situação | Ação |
|---|---|
| Deploy novo → taxa de erro > 5% em 5 min | Rollback imediato sem investigação |
| Deploy novo → latência P95 > 500ms | Rollback imediato |
| Bug não crítico introduzido | Fix forward (hotfix) — sem rollback |
| Migration destrutiva com dados corrompidos | Rollback de código + restauração de backup Supabase |

---

## 7. Checklist de Deploy em Produção

Execute este checklist antes de cada deploy em produção:

### 7.1 Pré-deploy

- [ ] CI passou (lint, type-check, testes, build) no branch `main`
- [ ] PR foi revisado e aprovado
- [ ] Migration testada em staging — dados íntegros, performance OK
- [ ] Variáveis de ambiente de produção atualizadas (se novas variáveis adicionadas)
- [ ] Sentry configurado e recebendo eventos do ambiente de produção
- [ ] Comunicação com time: deploy agendado informado no Slack `#deploys`

### 7.2 Durante o deploy

- [ ] Acompanhar Railway logs em tempo real: `railway logs --service cedente-api-prod`
- [ ] Verificar health check: `curl https://api-cedente.up.railway.app/health`
- [ ] Verificar Sentry: sem novo spike de erros
- [ ] Verificar Vercel: deploy de frontend OK

### 7.3 Pós-deploy (30 minutos de monitoramento)

- [ ] Taxa de erro Sentry < 0.1%
- [ ] Latência P95 da API < 200ms (conforme SLA)
- [ ] Dashboard Railway: CPU e memória dentro do normal
- [ ] Supabase Dashboard: queries sem degradação
- [ ] Testar manualmente os fluxos críticos: cadastro, upload, proposta, guardião

---

## 8. Referências

- Doc 02 — Stacks Tecnológicas: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/02 - Stacks.md`
- Doc 25 — Observabilidade e Logs: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/25 - Observabilidade e Logs.md`
- Doc 26 — Runbook Operacional: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/26 - Runbook Operacional.md`
- Documentação Railway: [docs.railway.app](https://docs.railway.app)
- Documentação Vercel: [vercel.com/docs](https://vercel.com/docs)
- Documentação EAS Build: [docs.expo.dev/build/introduction](https://docs.expo.dev/build/introduction)
- Prisma Migrations: [prisma.io/docs/guides/migrate](https://prisma.io/docs/guides/migrate)
- Semantic Versioning: [semver.org](https://semver.org)
