# 24 - Deploy, CI-CD e Versionamento

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — DevOps, Backend, Frontend |
| **Escopo** | Pipeline de CI/CD, ambientes, deploy da API, migrations, rollback e feature flags |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | 02 Stacks CRM · 23 Guia de Contribuição |

---

> **TL;DR**
>
> - **Pipeline GitHub Actions:** lint → typecheck → test → build → deploy. Todo PR em `develop` passa pelo pipeline completo.
> - **3 ambientes:** `development` (local), `staging` (Vercel preview + Railway staging), `production` (Vercel + Railway main).
> - **Deploy frontend:** Vercel — automático em todo merge para `develop` (staging) e `main` (produção).
> - **Deploy API:** Railway — zero-downtime com rolling deploy.
> - **Migrations:** `prisma migrate deploy` executado automaticamente no pipeline antes do deploy da API.
> - **Rollback:** procedimento documentado para reverter deploy em ≤ 10 min.
> - **Feature flags:** PostHog para rollout gradual de novas features críticas.

---

## 1. Ambientes

| Ambiente | Frontend (Next.js) | API (NestJS) | Banco | Redis | RabbitMQ |
|---|---|---|---|---|---|
| `development` | `localhost:3000` | `localhost:3001` | Supabase dev | Docker local | Docker local |
| `staging` | Vercel preview (URL por PR) | Railway staging | Supabase staging | Upstash staging | CloudAMQP staging |
| `production` | `crm.repasseseguro.com.br` | `api.crm.repasseseguro.com.br` | Supabase produção | Upstash produção | CloudAMQP produção |

---

## 2. Pipeline GitHub Actions

### 2.1 Estrutura do Pipeline

```yaml
# .github/workflows/crm-ci.yml
name: CRM — CI/CD Pipeline

on:
  pull_request:
    branches: [develop, main]
    paths:
      - 'apps/api/modules/crm/**'
      - 'apps/web-crm/**'
      - 'packages/**'
  push:
    branches: [develop, main]

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint

  typecheck:
    name: Typecheck
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck

  test:
    name: Test (unit + integration)
    runs-on: ubuntu-latest
    needs: typecheck
    services:
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
      rabbitmq:
        image: rabbitmq:4-alpine
        ports: ['5672:5672']
    env:
      DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
      DIRECT_URL: ${{ secrets.STAGING_DIRECT_URL }}
      SUPABASE_URL: ${{ secrets.STAGING_SUPABASE_URL }}
      SUPABASE_SERVICE_ROLE_KEY: ${{ secrets.STAGING_SUPABASE_SERVICE_ROLE_KEY }}
      REDIS_URL: redis://localhost:6379
      RABBITMQ_URL: amqp://localhost:5672
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter api test --coverage
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          flags: crm-api

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter web-crm build
      - run: pnpm --filter api build

  deploy-staging:
    name: Deploy → Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy API → Railway staging
        run: railway up --environment staging --service crm-api
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
      # Frontend Vercel: automático via Vercel GitHub integration

  deploy-production:
    name: Deploy → Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Apply Prisma migrations
        run: pnpm --filter api prisma migrate deploy
        env:
          DIRECT_URL: ${{ secrets.PRODUCTION_DIRECT_URL }}
      - name: Deploy API → Railway production
        run: railway up --environment production --service crm-api
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
      # Frontend Vercel: automático via Vercel GitHub integration
```

### 2.2 Tempo Esperado por Etapa

| Etapa | Tempo médio | Bloqueia pipeline? |
|---|---|---|
| Lint | ~1 min | Sim |
| Typecheck | ~2 min | Sim |
| Test (unit + integration) | ~5 min | Sim |
| Build | ~3 min | Sim |
| Deploy staging | ~3 min | Sim (para staging) |
| Deploy production | ~5 min (inclui migration) | Sim (para produção) |

**Tempo total para produção:** ~19 min desde o merge em `main`.

---

## 3. Deploy da API — Railway

### 3.1 Zero-Downtime com Rolling Deploy

O Railway executa rolling deploys por padrão. O processo para o módulo CRM:

1. Nova instância iniciada em paralelo com a atual
2. Health check (`GET /health`) valida que a nova instância está saudável
3. Tráfego migrado gradualmente (Railway Load Balancer)
4. Instância antiga removida após 30s de drenagem de conexões

```bash
# Verificar status do deploy
railway status --environment production

# Ver logs do deploy
railway logs --environment production --tail 100
```

### 3.2 Configuração de Health Check no Railway

```yaml
# railway.json (raiz de apps/api)
{
  "deploy": {
    "healthcheckPath": "/health",
    "healthcheckTimeout": 30,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 3
  }
}
```

O endpoint `/health` retorna:
```json
{
  "status": "ok",
  "database": "ok",
  "redis": "ok",
  "rabbitmq": "ok",
  "timestamp": "2026-03-23T15:30:00.000Z"
}
```

Se qualquer dependência retornar `"degraded"`, o deploy é pausado e o Railway notifica via webhook de Slack.

---

## 4. Migrations com Prisma

### 4.1 Fluxo de Migration

```
Desenvolvimento:
  pnpm --filter api prisma migrate dev --name descricao_da_mudanca
  → Cria arquivo em prisma/migrations/YYYYMMDDHHMMSS_descricao_da_mudanca/migration.sql
  → Commitar o arquivo de migration junto com o código

Staging (automático via CI):
  pnpm --filter api prisma migrate deploy
  → Aplica migrations pendentes no banco staging

Produção (automático via CI — apenas em merge para main):
  pnpm --filter api prisma migrate deploy
  → Aplica migrations pendentes no banco produção
  → Executado ANTES do deploy da nova imagem da API
```

### 4.2 Regras de Migration

- **Migrations são sempre aditivas na v1.0.** Nenhum `DROP COLUMN`, `DROP TABLE` ou `ALTER COLUMN` que quebre a versão anterior.
- Colunas removidas: adicionar `@deprecated` no schema e remover em 2 releases.
- Colunas `NOT NULL` novas: sempre com valor `DEFAULT` definido.
- Toda migration passa por revisão de código antes de merge.
- Em produção, `DIRECT_URL` é usada para migrations (contorna o connection pooler do Supabase).

### 4.3 Migration de Emergência

Se uma migration precisar ser aplicada fora do pipeline:

```bash
# Conectar ao Railway production via CLI
railway run --environment production -- pnpm --filter api prisma migrate deploy

# Verificar status das migrations
railway run --environment production -- pnpm --filter api prisma migrate status
```

---

## 5. Rollback

### 5.1 Rollback de Deploy (API — Railway)

O Railway mantém histórico dos últimos 10 deployments. Rollback em ≤ 5 min:

```bash
# 1. Listar deployments recentes
railway deployments --environment production

# 2. Identificar o deployment estável anterior (pelo ID ou timestamp)

# 3. Reverter para o deployment anterior via dashboard Railway:
#    Railway Dashboard → CRM API Service → Deployments → [deployment anterior] → Redeploy

# 4. Verificar health check
curl https://api.crm.repasseseguro.com.br/health
# Esperado: { "status": "ok" }

# 5. Notificar equipe no Slack #crm-incidents
```

Tempo esperado para rollback da API: **3–5 min**.

### 5.2 Rollback de Deploy (Frontend — Vercel)

```bash
# Via Vercel CLI:
vercel rollback --scope repasse-seguro

# Ou via Vercel Dashboard:
# Vercel → web-crm → Deployments → [deployment anterior] → Promote to Production
```

Tempo esperado para rollback do frontend: **1–2 min**.

### 5.3 Rollback de Migration

Migrations no Prisma **não têm rollback automático**. Procedimento manual:

```bash
# 1. Identificar a migration a reverter (em prisma/migrations/)

# 2. Criar uma nova migration reversa (nunca deletar a migration aplicada):
pnpm --filter api prisma migrate dev --name revert_nome_da_migration_anterior

# 3. Escrever manualmente o SQL de reversão no arquivo gerado

# 4. Aplicar via pipeline normal ou migration de emergência
```

> **Nota:** A melhor prevenção é garantir que todas as migrations são aditivas. Reverter migrations destrutivas requer restauração de backup do Supabase.

### 5.4 Critérios para Acionar Rollback Automático

| Métrica | Threshold | Ação |
|---|---|---|
| Taxa de erros 5xx | > 2% em janela de 5 min | Rollback imediato |
| Latência p95 | > 2s por > 5 min | Investigar; rollback se persistir |
| Health check falhando | 3 checks consecutivos falhando | Rollback imediato |
| DLQ de RabbitMQ | > 10 mensagens em < 5 min | Investigar; rollback se causa for deploy |

---

## 6. Feature Flags com PostHog

### 6.1 Uso de Feature Flags no CRM

Feature flags são usadas para rollout gradual de features críticas — especialmente aquelas que alteram fluxos de trabalho da equipe interna.

```typescript
// Exemplo de uso no frontend (apps/web-crm)
import { useFeatureFlag } from 'posthog-js/react';

function PipelineKanban() {
  const newKanbanEnabled = useFeatureFlag('crm-new-kanban-v2');

  if (newKanbanEnabled) {
    return <NewKanbanBoard />;
  }
  return <LegacyKanbanBoard />;
}
```

```typescript
// Exemplo de uso no backend (apps/api)
import { PostHogClient } from '../observability/posthog.service';

async function processCase(caseId: string, userId: string) {
  const posthog = new PostHogClient();
  const newFlowEnabled = await posthog.isFeatureEnabled('crm-new-case-flow', userId);

  if (newFlowEnabled) {
    return this.newCaseFlow(caseId);
  }
  return this.legacyCaseFlow(caseId);
}
```

### 6.2 Estratégia de Rollout

| Fase | Percentual de usuários | Duração sugerida |
|---|---|---|
| Canary (equipe de testes) | 5% — usuários específicos por e-mail | 2–3 dias |
| Beta interno | 25% — aleatório | 3–5 dias |
| Rollout progressivo | 50% → 75% → 100% | 2 dias por etapa |
| GA (Generally Available) | 100% — remover flag | Após validação |

### 6.3 Flags Ativas (CRM v1.0)

| Flag | Descrição | Status inicial |
|---|---|---|
| `crm-realtime-sla-alerts` | Alertas de SLA via Supabase Realtime | `false` (fallback polling) |
| `crm-dani-admin-ia` | Agente Dani-Admin (Fase 2) | `false` — não disponível na v1.0 |
| `crm-bulk-case-operations` | Operações em lote no pipeline | `false` (pós go-live) |

---

## 7. Versionamento Semântico

O CRM segue **SemVer 2.0.0** gerenciado automaticamente pelo `semantic-release`.

### 7.1 Configuração

```json
// .releaserc.json (raiz do monorepo — seção CRM)
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", {
      "changelogFile": "CHANGELOG.md"
    }],
    "@semantic-release/github",
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.version} [skip ci]"
    }]
  ]
}
```

### 7.2 Mapeamento de Commits para Versão

| Tipo de commit | Incremento de versão |
|---|---|
| `fix:`, `perf:` | PATCH (`1.0.0 → 1.0.1`) |
| `feat:` | MINOR (`1.0.1 → 1.1.0`) |
| `feat!:` ou `BREAKING CHANGE:` | MAJOR (`1.1.0 → 2.0.0`) |
| `chore:`, `docs:`, `test:`, `ci:` | Nenhum (sem release) |

---

## 8. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — pipeline GitHub Actions completo, 3 ambientes, deploy Railway com zero-downtime, migrations Prisma no pipeline, rollback em ≤5 min, feature flags PostHog, SemVer com semantic-release. |
