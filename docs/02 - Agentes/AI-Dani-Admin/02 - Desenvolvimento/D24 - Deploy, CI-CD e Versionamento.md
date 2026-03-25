# Deploy, CI-CD e Versionamento — AI-Dani-Admin

## Pipeline de Deploy e Estratégia de Versionamento

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend e DevOps |
| Escopo | Pipeline CI/CD completo, estratégia de deploy, ambientes e versionamento semântico do módulo AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D02 (Stacks), D15 (Arquitetura de Pastas), D22 (Setup Local), D23 (Guia de Contribuição) |

---

> **📌 TL;DR**
>
> - **CI:** GitHub Actions. PR → lint + tests + build. Tudo passa antes de merge.
> - **CD:** Deploy automático em staging ao merge na `main`. Deploy em produção manual via tag semântica.
> - **3 ambientes:** `development` (local), `staging` (automático), `production` (manual).
> - **Migrations:** `prisma migrate deploy` executado automaticamente no início do container, antes do server start.
> - **Versionamento:** Semantic Versioning (semver). Tags `v1.0.0`, `v1.1.0`, `v2.0.0`. Breaking changes → major.
> - **Rollback:** reverter tag e re-deploy. Migrations destrutivas exigem migration de rollback manual.

---

## 1. Ambientes

| Ambiente | Branch/Trigger | URL Base | Deploy | Banco |
|---|---|---|---|---|
| Development | Local | `http://localhost:3000` | Manual (`npm run start:dev`) | PostgreSQL local (Docker) |
| Staging | `main` (auto) | `https://api-staging.repasseseguro.com.br` | Automático (GitHub Actions) | Supabase staging |
| Production | Tag `v*` (manual) | `https://api.repasseseguro.com.br` | Manual (GitHub Actions release) | Supabase production |

---

## 2. Pipeline CI — Pull Request

```yaml
# .github/workflows/ci.yml

name: CI — AI-Dani-Admin

on:
  pull_request:
    branches: [main]
    paths:
      - 'apps/api/**'
      - 'prisma/**'
      - 'packages/**'

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint --workspace=apps/api

  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: lint

    services:
      postgres:
        image: supabase/postgres:17.0.0
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: repasse_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7.4-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s

    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/repasse_test
      REDIS_URL: redis://localhost:6379
      JWT_SECRET: test-secret-min-32-chars-for-ci
      NODE_ENV: test

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Prisma migrations (test db)
        run: npx prisma migrate deploy
        working-directory: apps/api
      - name: Unit tests with coverage
        run: npm run test:cov --workspace=apps/api
      - name: Check coverage threshold (80%)
        run: |
          COVERAGE=$(cat apps/api/coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage ${COVERAGE}% is below 80% threshold"
            exit 1
          fi
      - name: E2E tests
        run: npm run test:e2e --workspace=apps/api

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npm run build --workspace=apps/api
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: api-dist
          path: apps/api/dist/
          retention-days: 7
```

---

## 3. Pipeline CD — Deploy em Staging

```yaml
# .github/workflows/cd-staging.yml

name: CD — Deploy Staging

on:
  push:
    branches: [main]
    paths:
      - 'apps/api/**'
      - 'prisma/**'

jobs:
  deploy-staging:
    name: Deploy → Staging
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci
      - run: npm run build --workspace=apps/api

      - name: Build Docker image
        run: |
          docker build -t repasse-api:${{ github.sha }} -f apps/api/Dockerfile .

      - name: Push to registry
        run: |
          echo ${{ secrets.REGISTRY_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker tag repasse-api:${{ github.sha }} ghcr.io/shiftlabs/repasse-api:staging
          docker push ghcr.io/shiftlabs/repasse-api:staging

      - name: Deploy to staging
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: deploy
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /app/repasse-api
            docker pull ghcr.io/shiftlabs/repasse-api:staging
            docker-compose -f docker-compose.staging.yml up -d --no-deps api
            # Migrations executadas automaticamente pelo entrypoint do container
            docker-compose -f docker-compose.staging.yml exec -T api npx prisma migrate deploy

      - name: Health check
        run: |
          sleep 10
          curl --fail https://api-staging.repasseseguro.com.br/health || exit 1

      - name: Notify Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
          text: "Deploy staging ${{ job.status }} — commit ${{ github.sha }}"
```

---

## 4. Pipeline CD — Deploy em Produção

```yaml
# .github/workflows/cd-production.yml

name: CD — Deploy Production

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  deploy-production:
    name: Deploy → Production
    runs-on: ubuntu-latest
    environment: production  # requer aprovação manual no GitHub

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci
      - run: npm run build --workspace=apps/api

      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Build and push Docker image
        run: |
          docker build -t repasse-api:${{ steps.version.outputs.VERSION }} -f apps/api/Dockerfile .
          docker tag repasse-api:${{ steps.version.outputs.VERSION }} ghcr.io/shiftlabs/repasse-api:${{ steps.version.outputs.VERSION }}
          docker tag repasse-api:${{ steps.version.outputs.VERSION }} ghcr.io/shiftlabs/repasse-api:latest
          echo ${{ secrets.REGISTRY_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/shiftlabs/repasse-api:${{ steps.version.outputs.VERSION }}
          docker push ghcr.io/shiftlabs/repasse-api:latest

      - name: Deploy to production
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: deploy
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            cd /app/repasse-api
            docker pull ghcr.io/shiftlabs/repasse-api:${{ steps.version.outputs.VERSION }}
            # Zero-downtime: rolling update
            docker-compose -f docker-compose.prod.yml up -d --no-deps --scale api=2 api
            docker-compose -f docker-compose.prod.yml exec -T api npx prisma migrate deploy
            docker-compose -f docker-compose.prod.yml up -d --no-deps --scale api=1 api

      - name: Health check (produção)
        run: |
          sleep 15
          curl --fail https://api.repasseseguro.com.br/health || exit 1

      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ steps.version.outputs.VERSION }}
          release_name: Release ${{ steps.version.outputs.VERSION }}
          draft: false
          prerelease: false
```

---

## 5. Dockerfile

```dockerfile
# apps/api/Dockerfile

FROM node:22-alpine AS builder

WORKDIR /app

# Copiar manifests de dependências
COPY package*.json ./
COPY apps/api/package*.json ./apps/api/
COPY packages/ ./packages/

# Instalar dependências
RUN npm ci --workspace=apps/api

# Copiar código-fonte
COPY apps/api/ ./apps/api/
COPY prisma/ ./prisma/

# Build
RUN npm run build --workspace=apps/api

# ----------------------------------------
FROM node:22-alpine AS production

WORKDIR /app

ENV NODE_ENV=production

# Copiar apenas o necessário
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/apps/api/dist ./dist
COPY --from=builder /app/prisma ./prisma

# Instalar apenas o Prisma Client (necessário para migrate deploy)
RUN npm install --no-save @prisma/client prisma

EXPOSE 3000

# Migrations + start
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main"]
```

---

## 6. Versionamento Semântico

Formato: `v{MAJOR}.{MINOR}.{PATCH}`

| Tipo | Quando incrementar | Exemplo |
|---|---|---|
| PATCH | Bugfix, correção de segurança sem breaking change | `v1.0.0` → `v1.0.1` |
| MINOR | Nova funcionalidade, adição de endpoint (retrocompatível) | `v1.0.1` → `v1.1.0` |
| MAJOR | Breaking change na API, mudança de schema incompatível, mudança de endpoint | `v1.1.0` → `v2.0.0` |

**Processo de criação de release:**

```bash
# 1. Garantir que main está estável e todos os testes passam
git checkout main
git pull origin main
npm run test --workspace=apps/api  # deve passar 100%

# 2. Criar tag semântica
git tag -a v1.1.0 -m "feat: implementar dashboard de métricas com cache Redis"

# 3. Push da tag (dispara pipeline CD production)
git push origin v1.1.0

# 4. Aprovar deploy no GitHub (environment protection requer aprovação manual)
```

**Changelog automático:** o GitHub Actions gera changelog a partir dos commits Conventional Commits entre tags.

---

## 7. Estratégia de Rollback

### 7.1 Rollback de código (sem migration)

```bash
# Re-deploy da versão anterior
git push origin v1.0.0:refs/tags/v1.0.0-rollback
# ou simplesmente re-deploy via GitHub UI selecionando tag anterior
```

### 7.2 Rollback com migration destrutiva

Migrations em Prisma não têm rollback automático. Para migrations destrutivas (DROP COLUMN, DROP TABLE):

1. **Não usar `--force`** em migrations destrutivas diretamente.
2. Criar migration de rollback manual:
   ```bash
   npx prisma migrate dev --name rollback_v1_1_0_drop_column
   # Escrever SQL inverso na migration manualmente
   ```
3. Aplicar migration de rollback em staging primeiro, validar, depois produção.
4. Re-deploy do código anterior após rollback do schema.

**Regra:** toda migration destrutiva deve ter migration de rollback escrita e testada antes do deploy em produção.

---

## 8. Secrets no CI/CD

| Secret | Onde configurar | Ambientes |
|---|---|---|
| `DATABASE_URL` | GitHub Repository Secrets | staging, production |
| `REDIS_URL` | GitHub Repository Secrets | staging, production |
| `RABBITMQ_URL` | GitHub Repository Secrets | staging, production |
| `JWT_SECRET` | GitHub Repository Secrets | staging, production |
| `OPENAI_API_KEY` | GitHub Repository Secrets | staging, production |
| `LANGFUSE_PUBLIC_KEY` | GitHub Repository Secrets | staging, production |
| `LANGFUSE_SECRET_KEY` | GitHub Repository Secrets | staging, production |
| `SENTRY_DSN` | GitHub Repository Secrets | staging, production |
| `POSTHOG_API_KEY` | GitHub Repository Secrets | staging, production |
| `SLACK_WEBHOOK_URL` | GitHub Repository Secrets | staging, production |
| `SENDGRID_API_KEY` | GitHub Repository Secrets | staging, production |
| `EXPO_ACCESS_TOKEN` | GitHub Repository Secrets | production |
| `STAGING_HOST` | GitHub Repository Secrets | staging |
| `STAGING_SSH_KEY` | GitHub Repository Secrets | staging |
| `PRODUCTION_HOST` | GitHub Repository Secrets | production |
| `PRODUCTION_SSH_KEY` | GitHub Repository Secrets | production |
| `REGISTRY_TOKEN` | GitHub Repository Secrets | staging, production |

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Pipeline CI (lint+test+build), CD staging (auto) e produção (manual via tag), Dockerfile multi-stage, estratégia de rollback com migrations. |
