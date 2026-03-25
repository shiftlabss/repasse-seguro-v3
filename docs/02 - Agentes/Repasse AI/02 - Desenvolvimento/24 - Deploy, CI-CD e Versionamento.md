# 24 - Deploy, CI/CD e Versionamento

## Cabeçalho

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
| --- | --- | --- | --- | --- |
| 24 - Deploy, CI/CD e Versionamento | v1.0 | 2026-03-22 | Claude Code Desktop (ShiftLabs Pipeline v9.5) | Aprovado |

---

## TL;DR

> 📌 **Pipeline de deploy do Repasse AI:**
>
> - **Plataforma:** Railway (backend NestJS) — deploy automático a cada merge em `main`; sem cold starts (ADR-004).
> - **CI:** GitHub Actions — lint + typecheck + testes unitários + testes de integração + build + security audit por PR.
> - **Ambientes:** dev (local Docker), staging (Railway — branch `staging`), produção (Railway — branch `main`).
> - **Promoção staging → produção:** merge PR `staging → main` com 2 aprovações + smoke tests passando.
> - **Rollback:** Railway "Redeploy previous deployment" via dashboard ou CLI — tempo estimado < 3 min.
> - **Versionamento:** SemVer + tags Git automatizadas por GitHub Actions no merge em `main`.
> - **Hotfix:** branch `hotfix/` → PR direto para `main` → 1 aprovação → deploy prioritário.

---

## 1. Matriz de Ambientes

| Ambiente | Objetivo | URL | Branch | Trigger de Deploy | Tipo de Dados | Responsável Promote | Critério de Uso |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `dev` | Desenvolvimento local | `http://localhost:3000` | Qualquer | Manual (`pnpm dev`) | Fake/seed | Developer | Desenvolvimento e testes manuais |
| `staging` | Validação pré-produção | `https://repasse-ai-staging.railway.app` | `staging` | Push em `staging` (CD automático) | Dados de teste mascarados | Tech Lead | QA, smoke tests, validação de integração |
| `production` | Produção | `https://api.repasseseguro.com.br` | `main` | Push em `main` (CD automático) | Dados reais | Tech Lead (aprovação manual do PR) | Após validação completa em staging |

> 🔴 **Regra inegociável:** nunca fazer push direto em `main` ou `staging`. Todo deploy parte de um PR aprovado.

---

## 2. Diagrama do Pipeline

```mermaid
flowchart TD
    A[Developer: git push feat/branch] --> B[GitHub: PR aberto]
    B --> C[GitHub Actions: CI Pipeline]
    C --> C1[Lint\npnpm lint]
    C --> C2[Typecheck\npnpm typecheck]
    C --> C3[Testes unitários\npnpm test]
    C --> C4[Testes integração\npnpm test:integration]
    C --> C5[Build\npnpm build]
    C --> C6[Security audit\npnpm audit]
    C1 & C2 & C3 & C4 & C5 & C6 --> D{Todos passaram?}
    D -->|Não| E[PR bloqueado\nDev corrige]
    E --> C
    D -->|Sim| F[Code Review\n≥ 2 aprovações para main\n≥ 1 para staging]
    F --> G[Squash merge em main ou staging]
    G -->|merge em staging| H[Railway: deploy staging automático]
    H --> I[Smoke tests automáticos\npnpm test:smoke staging]
    I --> J{Smoke passou?}
    J -->|Não| K[Rollback staging\nRailway redeploy anterior]
    J -->|Sim| L[Promoção PR: staging → main\n2 aprovações + Tech Lead]
    G -->|merge em main| M[Railway: deploy produção automático]
    M --> N[Health check automático\nGET /health ×5 em 60s]
    N --> O{Health OK?}
    O -->|Não| P[Rollback produção\nRailway redeploy anterior\n< 3 min]
    O -->|Sim| Q[Tag Git automática\nv1.x.x gerada por Actions]
    Q --> R[Changelog atualizado\nCOMMIT_HISTORY.md]
    R --> S[Notificação Slack #deploys]
```

---

## 3. Workflows de CI/CD

### 3.1 Workflow: CI (Pull Request)

**Arquivo:** `.github/workflows/ci.yml`

| Atributo | Valor |
| --- | --- |
| Trigger | `pull_request` (opened, synchronize, reopened) para `main` e `staging` |
| Runner | `ubuntu-latest` |
| Node.js | 22.x |
| Secrets consumidos | Nenhum (testes usam mocks e Docker local) |
| Cache | `pnpm store` + `node_modules` por hash de `pnpm-lock.yaml` |
| Artefatos | Coverage report (uploadado como artifact, retido 7 dias) |
| Falha | Bloqueia merge — PR permanece com status "failing" |
| Notificação | GitHub status check + Slack #dev (opcional) |

**Jobs:**
1. `lint` — `pnpm lint` — 30s
2. `typecheck` — `pnpm typecheck` — 45s
3. `test-unit` — `pnpm test --coverage` — 2 min
4. `test-integration` — `pnpm test:integration` (Docker PostgreSQL + Redis via services) — 3 min
5. `build` — `pnpm build` — 3 min
6. `audit` — `pnpm audit --audit-level high` — 30s

```yaml
# .github/workflows/ci.yml (estrutura resumida)
name: CI
on:
  pull_request:
    branches: [main, staging]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint

  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7

  test-integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: supabase/postgres:17
        env: { POSTGRES_PASSWORD: postgres, POSTGRES_DB: repasse_ai_test }
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ['54322:5432']
      redis:
        image: redis:7.4
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm prisma migrate deploy
        env: { DATABASE_URL: postgresql://postgres:postgres@localhost:54322/repasse_ai_test }
      - run: pnpm test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:54322/repasse_ai_test
          REDIS_URL: redis://localhost:6379
          NODE_ENV: test
          JWT_DEV_MODE: "true"
          JWT_DEV_SECRET: test-secret-ci
```

### 3.2 Workflow: CD (Deploy Automático)

**Arquivo:** `.github/workflows/cd.yml`

| Atributo | Valor |
| --- | --- |
| Trigger | `push` em `main` (produção) e `staging` |
| Runner | `ubuntu-latest` |
| Secrets consumidos | `RAILWAY_TOKEN`, `SLACK_WEBHOOK_URL` |
| Artefatos | Nenhum (Railway faz build direto do repositório) |
| Falha | Notificação Slack #deploys + alerta PagerDuty se produção |
| Tempo estimado | Build Railway ~3 min; deploy ~1 min |

**Jobs:**
1. `deploy` — `railway up --service repasse-ai` via Railway CLI
2. `health-check` — Verificar `GET /health` × 5 com intervalo 10s
3. `tag-release` (apenas `main`) — Criar tag SemVer + atualizar CHANGELOG
4. `notify` — Mensagem Slack #deploys

### 3.3 Workflow: Smoke Tests (pós-deploy staging)

**Arquivo:** `.github/workflows/smoke.yml`

| Atributo | Valor |
| --- | --- |
| Trigger | workflow_run após CD em staging |
| Runner | `ubuntu-latest` |
| Secrets | `STAGING_JWT_DEV_SECRET`, `STAGING_API_URL` |
| Falha | Rollback automático em staging via Railway CLI |

---

## 4. Estratégia de Deploy

O Repasse AI usa **Rolling Deploy** via Railway. O Railway substitui instâncias em execução gradualmente com zero downtime.

[DECISÃO AUTÔNOMA] Rolling deploy escolhido sobre Blue-Green: Railway suporta rolling nativamente sem configuração adicional; blue-green exigiria setup manual de load balancer. Critério: zero downtime + simplicidade operacional no Railway.

**Pré-condições obrigatórias:**
- Todos os checks CI passando no PR de origem
- Migrations Prisma executadas antes do código novo (`railway run pnpm prisma migrate deploy`)
- Variáveis de ambiente validadas no Railway antes do merge

**Validação de sucesso:**
- `GET /health` retorna 200 com `{"status":"healthy"}` ou `{"status":"degraded"}`
- `http_error_rate` < 1% nos primeiros 5 minutos

**Quando não usar rolling deploy:**
- Migrations destrutivas (drop de coluna, rename) — usar janela de manutenção de 5 min com aviso prévio
- Breaking changes na API — versionar endpoint (`/v2`) antes de remover `/v1`

---

## 5. Promoção entre Ambientes

### 5.1 Dev → Staging

| Gate | Tipo | Responsável | Critério |
| --- | --- | --- | --- |
| CI passa | Automático | GitHub Actions | 100% dos jobs verdes |
| Code review | Manual | 1 reviewer | Aprovação no PR |
| Testes unitários | Automático | GitHub Actions | Coverage ≥ thresholds |
| Nenhum segredo em código | Automático | `git-secrets` pre-push | 0 matches de padrão de segredo |

### 5.2 Staging → Produção

| Gate | Tipo | Responsável | Critério |
| --- | --- | --- | --- |
| Smoke tests em staging | Automático | GitHub Actions | 100% passing |
| CI passa | Automático | GitHub Actions | 100% dos jobs verdes |
| Testes E2E em staging | Manual (pré-go-live) / Automático (CI) | QA Lead | 8 fluxos críticos passing |
| Code review | Manual | 2 reviewers (1 Tech Lead) | 2 aprovações no PR |
| Janela de deploy | Processo | Tech Lead | Terças e Quintas, 14h-16h BRT (horário de menor tráfego) |

[DECISÃO AUTÔNOMA] Janela de deploy Terças/Quintas 14h-16h: evita segunda (backlog de incidentes pós-fim de semana) e sexta (risco de problemas sem equipe no fim de semana). Alternativa (qualquer horário) descartada por risco operacional. Critério: SRE best practice para serviço B2B.

---

## 6. Build e Artefatos

O Railway faz build direto do repositório (sem Docker Hub intermediário). O Turborepo garante build incremental.

| App | Build Command | Output | Retenção | Validação |
| --- | --- | --- | --- | --- |
| `apps/ai` | `pnpm build` → `dist/` | Bundle NestJS compilado | N/A (Railway gerencia) | `node dist/main.js` sem erro |
| `prisma/` | `pnpm prisma generate` | Prisma Client | N/A | `pnpm prisma validate` |

**Variáveis de ambiente por ambiente:**

| Variável | Dev | Staging | Produção |
| --- | --- | --- | --- |
| `NODE_ENV` | development | staging | production |
| `LOG_LEVEL` | debug | info | info |
| `JWT_DEV_MODE` | true | false | false |
| `FEATURE_WHATSAPP_MOCK` | true | false | false |
| Secrets (OpenAI, Supabase, etc.) | `.env` local | Railway Secrets (staging) | Railway Secrets (prod) |

---

## 7. Rollback

> 🔴 **Regra inegociável:** rollback é uma ação operacional, não uma emergência improvisada. Deve ser executável por qualquer membro do time em < 5 minutos.

### 7.1 Quando Acionar

- `http_error_rate > 10%` por mais de 3 minutos após deploy
- `GET /health` retornando 503 por mais de 30 segundos
- Circuit breaker abrindo imediatamente após deploy (LLM não estava quebrado antes)
- Erro crítico CRITICAL no Sentry inexistente antes do deploy

### 7.2 Quem Pode Acionar

Qualquer membro do time com acesso ao Railway dashboard ou CLI. Não requer aprovação — ação emergencial autônoma.

### 7.3 Procedimento de Rollback (Produção)

**Via Railway Dashboard:**
1. Acessar `railway.app` → projeto `repasse-ai` → serviço `ai`
2. Aba "Deployments" → localizar o deployment anterior (pré-defeito)
3. Clicar em "Redeploy" → confirmar
4. Aguardar novo deployment concluir (~3 min)
5. Verificar `GET /health` retorna 200
6. Verificar Sentry — erro deve desaparecer

**Via Railway CLI:**
```bash
railway login
railway environment production
railway service repasse-ai
railway redeploy --deployment <DEPLOYMENT_ID_ANTERIOR>
# Verificar saúde
curl https://api.repasseseguro.com.br/health
```

### 7.4 Após o Rollback

- [ ] Verificar `GET /health` → 200
- [ ] `http_error_rate` < 1% por 5 minutos
- [ ] Sentry sem novos erros CRITICAL
- [ ] Notificar #deploys no Slack: "Rollback executado para vX.X.X. Causa: [motivo]"
- [ ] Abrir post-mortem (mesmo que breve) no Notion
- [ ] Criar issue no GitHub com label `regression` referenciando o deploy revertido

### 7.5 Rollback de Migration Prisma

Migrations Prisma são aditivas por padrão (sem rollback automático). Em caso de migration com erro:
1. **Não há rollback automático** — Prisma não desfaz migrations aplicadas.
2. Criar nova migration de correção: `pnpm prisma migrate dev --name fix_migration_name`
3. Se a coluna/tabela foi criada errado: migration DROP/ALTER.
4. Se dados foram corrompidos: restaurar backup Supabase (ponto anterior ao deploy).

[DECISÃO AUTÔNOMA] Migrations aditivas obrigatórias: nunca DROP de coluna em uso em uma única migration — sempre em duas etapas (1. remover referências no código, 2. DROP na migration seguinte). Alternativa (migration atômica com DROP) descartada por risco de perda de dados. Critério: segurança de dados em produção.

---

## 8. Semantic Versioning

O Repasse AI segue **SemVer** (`MAJOR.MINOR.PATCH`):

| Incremento | Quando | Exemplo |
| --- | --- | --- |
| `PATCH` | Bugfix, hotfix, chore, docs | `1.2.3 → 1.2.4` |
| `MINOR` | Nova funcionalidade backward-compatible | `1.2.3 → 1.3.0` |
| `MAJOR` | Breaking change na API (endpoint removido, schema incompatível) | `1.2.3 → 2.0.0` |

**Version Bump Automático:** o workflow CD lê os commits Conventional Commits desde a última tag e determina o bump automaticamente:
- `feat:` → MINOR
- `fix:`, `chore:`, `docs:` → PATCH
- `feat!:` ou `BREAKING CHANGE:` no footer → MAJOR

---

## 9. Ciclo de Release

```
Feature branch
    │
    ▼
PR para staging → CI passa → 1 aprovação → merge em staging
    │
    ▼
Smoke tests em staging → QA valida fluxos críticos
    │
    ▼
PR de staging → main (na janela de deploy)
    │                → CI passa → 2 aprovações (1 Tech Lead)
    ▼
Merge em main → CD automático → Railway deploy
    │
    ▼
Health checks automáticos (5 min) → Tag v1.x.x gerada
    │
    ▼
Changelog atualizado → Release Notes publicadas → Slack notificado
    │
    ▼
Estabilização: monitorar Sentry + Railway por 30 min
```

---

## 10. Changelog

O changelog é gerado automaticamente pelo GitHub Actions a partir dos commits Conventional Commits entre tags. Arquivo: `CHANGELOG.md` na raiz do monorepo.

**Formato:**
```markdown
## [1.3.0] - 2026-03-22

### Features
- feat(ai): adicionar cache semântico Redis com threshold 0.92 (#45)
- feat(chat): adicionar SSE streaming para respostas do agente (#43)

### Bug Fixes
- fix(whatsapp): corrigir timeout de OTP após reenvio (#47)

### Chores
- chore: atualizar nestjs para 10.4.2 (#46)
```

❌ **Anti-exemplo de changelog:**
```markdown
## 1.3.0
- Várias melhorias e correções
- Atualização de dependências
```

✅ **Correto:** cada entrada rastreável a um PR/commit específico.

---

## 11. Release Notes

Template mínimo para cada release em `main`:

```markdown
## Release Notes — Repasse AI v1.3.0

**Data:** 2026-03-22
**Deploy:** Railway production — 14:30 BRT

### Resumo
Adição de cache semântico Redis e SSE streaming para respostas do agente IA.

### Escopo
- Módulo `ai`: cache semântico + streaming
- Módulo `chat`: endpoint SSE
- Módulo `whatsapp`: correção de timeout OTP

### Riscos Conhecidos
- Cache semântico pode retornar resposta ligeiramente desatualizada (TTL 24h) — comportamento esperado.

### Ações Pós-Release
- [ ] Monitorar hit rate do cache semântico no dashboard IA (target > 30%)
- [ ] Verificar latência SSE no Langfuse (target p95 < 5s)

### Impacto para Usuário
- Respostas do agente chegam ~2s mais rápido em média (cache hit)
- Streaming visível na UI — texto aparece progressivamente

### Links
- PR: #45, #43, #47, #46
- Sentry release: sentry.io/releases/1.3.0
```

---

## 12. Tagging

Tags Git são criadas automaticamente pelo workflow CD no merge em `main`:

```bash
# Tags regulares
git tag -a v1.3.0 -m "Release v1.3.0 — cache semântico + SSE"
git push origin v1.3.0

# Tags de hotfix
git tag -a v1.2.4-hotfix -m "Hotfix: circuit breaker crash on startup"
git push origin v1.2.4-hotfix

# Release candidates (staging)
git tag -a v1.3.0-rc.1 -m "Release candidate: cache semântico"
git push origin v1.3.0-rc.1
```

**Regra:** a tag é o único identificador imutável de um deployment em produção. Todo rollback referencia a tag anterior.

---

## 13. Comunicação de Release

| Momento | Canal | Quem | Template |
| --- | --- | --- | --- |
| 1h antes do deploy (produção) | Slack #repasse-ai-dev | Tech Lead | "Deploy v1.3.0 agendado para 14:30 BRT. Features: [lista]. Janela de estabilização: 14:30–15:00." |
| Deploy iniciado | Slack #deploys | GitHub Actions (automático) | "🚀 Deploy v1.3.0 iniciado em produção." |
| Deploy concluído + health OK | Slack #deploys | GitHub Actions (automático) | "✅ Deploy v1.3.0 concluído. Health: OK. Monitorando por 30 min." |
| Rollback acionado | Slack #deploys + #alerts | Responsável pelo rollback | "⚠️ Rollback para v1.2.4 acionado. Causa: [motivo]. Investigando." |

---

## 14. Hotfix Flow

O fluxo de hotfix segue o documentado no Doc 23 (Guia de Contribuição), com os seguintes complementos de CI/CD:

1. Branch `hotfix/nome` criada de `main`
2. CI completo obrigatório (sem skip de etapas) — mesmo fluxo do PR regular
3. 1 aprovação via Slack em < 1 hora
4. Merge commit (não squash) para preservar histórico
5. Railway faz deploy automático em produção
6. Health checks automáticos × 5
7. Tag `vX.X.Y-hotfix` criada automaticamente
8. Changelog atualizado com entrada `### Hotfixes`
9. Notificação #deploys com contexto do hotfix

---

## 15. Backlog de Pendências

| ID | Descrição | Prioridade | Observação |
| --- | --- | --- | --- |
| CD-001 | Configurar `RAILWAY_TOKEN` como GitHub Secret + criar `cd.yml` com deploy automático | Alta | Necessário no setup do projeto |
| CD-002 | Configurar `SLACK_WEBHOOK_URL` como GitHub Secret para notificações automáticas | Média | Necessário para comunicação de release |
| CD-003 | Configurar branch `staging` no Railway como ambiente separado de produção | Alta | Necessário antes do primeiro deploy staging |
| CD-004 | Implementar smoke tests (`pnpm test:smoke`) — scripts que chamam `/health` e 1-2 endpoints críticos pós-deploy | Alta | Parte do workflow CD |

> **Decisões Autônomas Tomadas Neste Documento:**
>
> 1. **[DECISÃO AUTÔNOMA] Rolling deploy sobre Blue-Green:** Railway suporta rolling nativamente; blue-green exige setup manual. Critério: zero downtime com menor complexidade.
> 2. **[DECISÃO AUTÔNOMA] Janela de deploy Terças/Quintas 14h-16h BRT:** evita segunda e sexta. Critério: menor risco operacional sem equipe disponível.
> 3. **[DECISÃO AUTÔNOMA] Migrations aditivas obrigatórias em duas etapas:** DROP em migration única é proibido em produção. Critério: segurança de dados.
> 4. **[DECISÃO AUTÔNOMA] Version bump automático por Conventional Commits:** alternativa (bump manual) descartada por risco de esquecimento e inconsistência. Critério: automação reduz erro humano.

---

*Próximo documento do pipeline: D26 — Runbook Operacional.*
