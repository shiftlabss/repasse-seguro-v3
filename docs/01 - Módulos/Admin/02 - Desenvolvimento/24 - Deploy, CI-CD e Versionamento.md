# 24 - Deploy, CI-CD e Versionamento

## Repasse Seguro — Módulo Admin

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | DevOps e Engenharia |
| **Escopo** | Pipeline de deploy, CI/CD, ambientes, versionamento, release e rollback |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Status** | Aprovado |
| **Dependências** | D02 Stacks · D14 Especificações Técnicas · D22 Guia de Ambiente · D23 Guia de Contribuição |

---

> 📌 **TL;DR**
>
> - **3 ambientes:** Development (local), Staging (Railway preview / Vercel preview), Production (Railway main / Vercel main).
> - **Pipeline GitHub Actions:** 2 workflows principais — `ci.yml` (PR) e `deploy.yml` (merge para `develop`/`main`).
> - **Backend:** Railway (deploy automático por branch); **Frontend:** Vercel (deploy automático por branch); **Mobile:** Expo EAS.
> - **Deploy strategy:** rolling update no Railway (sem downtime, zero blue-green por ora); atomic deploy no Vercel.
> - **Rollback:** Railway → botão "Rollback" no dashboard (< 1min); Vercel → redeploy da versão anterior (< 30s).
> - **Versionamento:** Semver com tags Git — `v{major}.{minor}.{patch}`.
> - **Promoção para produção:** merge em `main` + tag + E2E passando + aprovação manual do Tech Lead.

---

## 1. Matriz de Ambientes

| Ambiente | Objetivo | URL API | URL SPA | Branch | Trigger | Dados | Responsável promoção |
|---|---|---|---|---|---|---|---|
| **Development** | Desenvolvimento local | `localhost:3001` | `localhost:5173` | Qualquer | Manual (`pnpm dev`) | Docker local (fake) | Dev |
| **Staging** | Validação pré-release, QA, testes E2E | `staging-api.repasseseguro.com.br` | `staging.repasseseguro.com.br` | `develop` | Merge em `develop` | Supabase staging (dados de teste) | Tech Lead |
| **Production** | Produção real | `api.repasseseguro.com.br` | `app.repasseseguro.com.br` | `main` | Tag `v*.*.*` em `main` | Supabase produção | Tech Lead (aprovação manual) |

> ⚙️ **Regra inegociável:** Staging e produção usam projetos Supabase **diferentes**. Secrets de produção **nunca** são usados em staging. Sem exceção.

---

## 2. Diagrama do Pipeline

```mermaid
flowchart TD
    A[Commit em feature branch] --> B[PR aberto para develop]
    B --> C{CI: typecheck + lint + test}
    C -->|Falhou| D[Bloqueia merge\nDev corrige]
    C -->|Passou| E[Review aprovado]
    E --> F[Squash merge → develop]
    F --> G[GitHub Actions: deploy.yml]
    G --> H[Build API + SPA]
    H --> I[Deploy Staging\nRailway + Vercel]
    I --> J[Smoke tests automáticos]
    J -->|Falhou| K[Alerta Slack\nRollback automático staging]
    J -->|Passou| L[E2E tests em staging]
    L -->|Falhou| K
    L -->|Passou| M{Aprovação Tech Lead?}
    M -->|Não aprovado| N[Aguardar]
    M -->|Aprovado| O[PR develop → main\nCriar tag v*.*.*]
    O --> P[GitHub Actions: deploy.yml - production]
    P --> Q[Migrate banco produção\nprisma migrate deploy]
    Q --> R[Deploy Produção\nRailway + Vercel]
    R --> S[Health check /health]
    S -->|Falhou| T[Rollback automático\nAlerta PagerDuty]
    S -->|Passou| U[Release nota publicada\nComunicação enviada]
```

---

## 3. Workflows de CI/CD

### 3.1 Workflow: `ci.yml` (Pull Requests)

**Trigger:** `pull_request` para `develop` ou `main`

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [develop, main]

jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17-alpine
        env: { POSTGRES_USER: repasse, POSTGRES_PASSWORD: test, POSTGRES_DB: test }
        ports: ['5432:5432']
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter api prisma migrate deploy
        env: { DATABASE_URL: postgresql://repasse:test@localhost:5432/test }
      - run: pnpm test
        env:
          DATABASE_URL: postgresql://repasse:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret-32-chars-minimum-ok
          REFRESH_TOKEN_SECRET: test-refresh-secret-32-chars-ok

  prisma-validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter api prisma validate
```

**Impacto se falhar:** merge bloqueado. PR não pode avançar até todos os jobs passarem.

---

### 3.2 Workflow: `deploy.yml` (Deploy por Branch)

**Trigger:** `push` para `develop` ou criação de tag `v*.*.*`

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [develop]
    tags: ['v*.*.*']

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy API (Railway Staging)
        uses: railway-app/deploy-action@v1
        with:
          service: api
          environment: staging
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
      - name: Deploy SPA (Vercel Staging)
        run: vercel --prod=false --token=${{ secrets.VERCEL_TOKEN }}
      - name: Run smoke tests
        run: |
          sleep 30  # aguardar deploy
          curl -f https://staging-api.repasseseguro.com.br/health
      - name: Run E2E (staging)
        run: pnpm --filter web e2e
        env:
          PLAYWRIGHT_BASE_URL: https://staging.repasseseguro.com.br

  deploy-production:
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.repasseseguro.com.br
    steps:
      - uses: actions/checkout@v4
      - name: Run migrations (production)
        run: pnpm --filter api prisma migrate deploy
        env: { DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }} }
      - name: Deploy API (Railway Production)
        uses: railway-app/deploy-action@v1
        with:
          service: api
          environment: production
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
      - name: Deploy SPA (Vercel Production)
        run: vercel --prod --token=${{ secrets.VERCEL_TOKEN }}
      - name: Health check
        run: curl -f https://api.repasseseguro.com.br/health
      - name: Notify Slack
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text":"✅ Deploy v${{ github.ref_name }} em produção concluído."}'
```

**Impacto se falhar:** alerta Slack `#alertas-producao` + rollback automático ativado pelo Railway.

---

## 4. Estratégia de Deploy

**`[DECISÃO AUTÔNOMA]`** — Rolling update no Railway para a API; atomic deploy no Vercel para o SPA.

| Componente | Estratégia | Justificativa | Risco |
|---|---|---|---|
| API (Railway) | Rolling update | Railway gerencia automaticamente — sem downtime, nova versão substitui gradualmente. Operação simples. | Versão antiga e nova da API podem coexistir por ~30s durante o rolling |
| SPA (Vercel) | Atomic deploy | Vercel substitui atomicamente — zero momento intermediário com versão mista | Nenhum — Vercel garante atomicidade |
| Mobile (Expo EAS) | OTA (Over-the-Air) | Atualizações JS sem publicar nova versão na store | Limitado a mudanças JS — mudanças nativas requerem nova build |

**Alternativa descartada:** Blue-green deployment no Railway — adiciona complexidade operacional sem ganho proporcional para o volume atual. Reavaliar quando escalar.

**Pré-condição obrigatória antes do deploy em produção:**
1. Migrations testadas em staging sem erro
2. Smoke tests de staging passando
3. E2E dos 5 fluxos críticos passando
4. Aprovação manual do Tech Lead no GitHub Actions environment `production`

---

## 5. Promoção entre Ambientes

### 5.1 Development → Staging

**Gate:** merge do PR para `develop` (automático — sem aprovação humana adicional)
- CI passando (typecheck + lint + testes + prisma validate)
- 1 aprovação de reviewer

**Validação pós-deploy staging:**
- `GET /health` retorna `200 { status: 'ok' }`
- Login com usuário de teste funcional
- E2E dos 5 fluxos críticos passando

### 5.2 Staging → Production

**Gate:** criação de tag `v*.*.*` em `main` + aprovação manual no GitHub Actions

| Critério | Verificação |
|---|---|
| E2E staging passando | CI artefato de teste com 0 falhas |
| Staging estável por ≥ 24h | Sem alerta crítico em staging nas últimas 24h |
| Migrations validadas | `prisma migrate status` sem pending em staging |
| Aprovação Tech Lead | Botão "Approve" no GitHub Actions environment |
| Janela de deploy | Seg–Sex entre 10h e 16h (America/Fortaleza) |

> ⚙️ **Janela de deploy:** deploys em produção são proibidos entre 17h e 9h do dia seguinte, e aos fins de semana — exceto hotfix crítico.

---

## 6. Build e Artefatos

| App | Comando de build | Artefato | Nomeação | Retenção |
|---|---|---|---|---|
| API (NestJS) | `pnpm --filter api build` | Imagem Docker (Railway build automático) | `api:v{tag}` | Últimas 10 versões |
| SPA (Vite) | `pnpm --filter web build` | `dist/` estático (Vercel deploy) | Snapshot por deploy ID | 90 dias no Vercel |
| Mobile (Expo EAS) | `eas build --platform all` | `.apk` + `.ipa` + OTA bundle | `{version}+{buildNumber}` | Indefinido (App Store / EAS) |

**Integridade:** Railway e Vercel calculam hash do artefato automaticamente. Não há checksum manual necessário.

---

## 7. Rollback

### 7.1 Quando Acionar

Acionar rollback imediatamente quando:
- `GET /health` retorna status != `200 ok` por mais de 2 minutos após deploy
- Taxa de erros 5xx > 5% nos primeiros 10 minutos após deploy
- Alerta PagerDuty disparado por falha crítica pós-deploy

### 7.2 Quem Pode Acionar

- Tech Lead ou membro de engenharia on-call
- Em emergência: qualquer engenheiro com acesso ao Railway e Vercel

### 7.3 Procedimento de Rollback

**API (Railway):**
```bash
# 1. Acessar Railway dashboard
# 2. Ir em Deployments → selecionar deploy anterior
# 3. Clicar "Rollback to this deployment"
# Tempo esperado: < 1 minuto

# Verificação obrigatória após rollback:
curl https://api.repasseseguro.com.br/health
# Esperado: { "status": "ok", "version": "{versão anterior}" }
```

**SPA (Vercel):**
```bash
# Via Vercel CLI:
vercel rollback [deployment-url] --token=$VERCEL_TOKEN

# Ou via dashboard Vercel:
# Project → Deployments → selecionar versão anterior → "Promote to Production"
# Tempo esperado: < 30 segundos
```

**Migrations de banco (atenção):**
> 🔴 **Atenção crítica:** migrations de banco são **irreversíveis por padrão**. Se o rollback envolve desfazer uma migration, acionar suporte do Supabase para snapshot pré-deploy. **Regra:** toda migration em produção deve ser aditiva (adicionar coluna/tabela) — nunca remover coluna em uso sem ciclo de deprecation.

### 7.4 Validações Obrigatórias Pós-Rollback

1. `GET /health` retorna `{ status: "ok" }` com versão anterior
2. Login com usuário de teste funcional
3. Nenhum alerta novo disparado por 5 minutos
4. Notificar Slack `#alertas-producao`: "🔄 Rollback executado para v{versão-anterior}. Motivo: {motivo}. Status: estável."

---

## 8. Semantic Versioning

| Tipo | Bump | Exemplo | Quando |
|---|---|---|---|
| Nova funcionalidade | `MINOR` | `v1.0.0 → v1.1.0` | Feature release normal |
| Correção de bug | `PATCH` | `v1.0.0 → v1.0.1` | Bugfix em staging ou produção |
| Breaking change (API) | `MAJOR` | `v1.0.0 → v2.0.0` | Mudança incompatível de contrato |
| Hotfix crítico | `PATCH` | `v1.0.0 → v1.0.1` | Correção urgente em produção |

**Formato de tag Git:** `v{major}.{minor}.{patch}` — ex: `v1.2.3`

```bash
# Criar tag de release
git tag v1.1.0 -m "release: v1.1.0 - Add SLA badge and escrow monitoring"
git push origin v1.1.0
```

---

## 9. Ciclo de Release

```
Feature branches → develop (staging) → QA/E2E → Tech Lead approval → main (tag) → produção
```

| Fase | Duração típica | Responsável | Gate de saída |
|---|---|---|---|
| Desenvolvimento | 1–3 dias | Dev | PR aprovado + CI passando |
| Staging | 1 dia | QA + Dev | E2E passando + smoke tests ok |
| Release candidate | Opcional (features grandes) | Tech Lead | QA manual + stakeholder sign-off |
| Produção | — | Tech Lead | Health check + monitoramento por 1h |
| Estabilização | 24–48h | Dev + DevOps | Sem alertas críticos |

---

## 10. Changelog

**Localização:** `CHANGELOG.md` na raiz do monorepo.

**Responsável:** autor do PR de release (`develop → main`).

**Quando atualizar:** no mesmo PR que abre a release (antes do merge em `main`).

**Formato (Keep a Changelog):**

```markdown
## [1.1.0] - 2026-03-22

### Adicionado
- Badge de SLA com timer regressivo no card de caso
- Monitoramento de Conta Escrow em tempo real (Supabase Realtime)

### Corrigido
- Status do agente IA não atualizava após takeover manual

### Alterado
- Threshold de confiança padrão do Guardião alterado de 80% para 85%
```

❌ **Errado:**
```markdown
## v1.1.0 - Bug fixes and improvements
- Various fixes
- Performance improvements
```

---

## 11. Release Notes

**Template mínimo obrigatório:**

```markdown
# Release Notes — v{versão}

**Data de deploy:** {data}
**Ambientes afetados:** Staging + Produção
**Tipo:** Feature / Bugfix / Hotfix

## Resumo
{1–3 bullets do que foi entregue}

## O que muda para o operador
{Mudanças visíveis ao usuário do Admin — novos botões, comportamentos alterados}

## Riscos conhecidos
{Comportamentos esperados que podem parecer bug, ou limitações conhecidas}

## Ações pós-deploy requeridas
{Configurações manuais, migrações de dados, atualizações de configuração pelo Master}

## Links
- PR de release: {link}
- Jira/Issue: {links}
- Sentry release: {link}
```

---

## 12. Tagging

| Tipo de tag | Formato | Exemplo | Quem cria |
|---|---|---|---|
| Release | `v{major}.{minor}.{patch}` | `v1.2.0` | Tech Lead (manual após aprovação) |
| Hotfix | `v{major}.{minor}.{patch+1}` | `v1.2.1` | Tech Lead (acelerado) |
| Release Candidate | `v{major}.{minor}.{patch}-rc.{N}` | `v2.0.0-rc.1` | Tech Lead (features grandes) |
| Rollback marker | `rollback-{data}-{motivo}` | `rollback-20260322-auth-bug` | Automático pelo workflow |

---

## 13. Comunicação de Release

### Antes do deploy (D-1):
- Slack `#eng-geral`: "🚀 Deploy de v{versão} amanhã entre 10h–12h. Staging estável. Detalhes: {link Release Notes}"

### Durante o deploy:
- Slack `#eng-geral`: "⏳ Deploy v{versão} em produção iniciado. Acompanhar em #alertas-producao"

### Após o deploy (sucesso):
- Slack `#eng-geral`: "✅ Deploy v{versão} em produção concluído. Health check ok."

### Após o deploy (rollback):
- Slack `#eng-geral`: "⚠️ Rollback de v{versão}. Sistema em v{versão-anterior}. Investigação em andamento."

---

## 14. Hotfix Flow

Hotfix é para bugs críticos em produção **apenas**. Ver critérios em D23 — Guia de Contribuição.

```bash
# 1. Criar branch a partir de main
git checkout main
git checkout -b hotfix/125-critical-auth-bypass

# 2. Implementar correção + teste de regressão
# ... commits ...

# 3. Abrir PR para main com título [HOTFIX]
# 4. Aprovação express: Tech Lead (SLA 2h)

# 5. Após aprovação: merge para main + criar tag patch
git checkout main
git merge --no-ff hotfix/125-critical-auth-bypass
git tag v1.0.1 -m "hotfix: v1.0.1 - Fix auth bypass"
git push origin main v1.0.1

# 6. GitHub Actions deploy.yml dispara para produção
# 7. Após deploy estável: merge hotfix para develop
git checkout develop
git merge --no-ff hotfix/125-critical-auth-bypass
git push origin develop
git branch -d hotfix/125-critical-auth-bypass
```

---

## 15. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — 3 ambientes, 2 workflows GitHub Actions, rolling update Railway, atomic deploy Vercel, rollback < 1min, Semver, ciclo de release, template de release notes, comunicação de deploy. |

---

## 16. Backlog de Pendências

| Item | Marcador | Seção | Justificativa | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Rolling update vs Blue-Green | `[DECISÃO AUTÔNOMA]` | 4 | Rolling update Railway escolhido por simplicidade operacional. Blue-green descartado por overhead desnecessário no volume atual. Reavaliar pós escala. | Disponibilidade durante deploy | DevOps | Decidido |
| Janela de deploy 10h–16h | `[DECISÃO AUTÔNOMA]` | 5.2 | Janela com maior suporte técnico disponível e menor impacto em usuários (horário comercial Fortaleza). Alternativa: anytime — maior flexibilidade mas risco operacional sem suporte. | Risco operacional | Tech Lead | Decidido |
