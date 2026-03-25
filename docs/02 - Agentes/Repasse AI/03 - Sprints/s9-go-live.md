# S9 — Go-Live

<!-- Atualizado em 2026-03-24 pela A03 — 1 correção aplicada (FINDING-011) -->

| Campo              | Valor                                       |
| ------------------ | ------------------------------------------- |
| **Sprint**         | S9                                          |
| **Nome**           | Go-Live                                     |
| **Template**       | B — Módulo Fullstack                        |
| **Docs Fonte**     | 24-DeployCI-CD, 29-GoLivePlaybook           |
| **REQs cobertos**  | REQ-115, REQ-116, REQ-117, REQ-118, REQ-119 |
| **Total de itens** | 14                                          |
| **Status**         | ✅ Concluido                                |

---

## 🎯 Objetivo

Executar o playbook de Go-Live: janelas T-7/T-3/T-1/Launch Day, smoke tests, critérios go/no-go, plano de rollback em < 3 minutos, configuração de SemVer + tags Git automáticas, e escalation matrix P0/P1/P2/P3.

---

## 🏗️ FEATURE 1: Deploy e Infraestrutura de Produção

### 1. Configuração Railway para produção

- [x] Criar `railway.json` ou `railway.toml` na raiz de `apps/ai/`:
  - `service.name: 'repasse-ai'`
  - `service.source.repo`: apontar para monorepo
  - `service.source.rootDirectory: 'apps/ai'`
  - `service.build.command: 'pnpm build'`
  - `service.start.command: 'node dist/main.js'`
  - `service.healthcheck.path: '/health'`
  - `service.healthcheck.timeout: 5`
  - `service.restartPolicy: 'on-failure'` com `maxRetries: 3`
  - **Sem cold starts** — Railway configured para manter instância ativa (conforme ADR-004)
- [x] Configurar 3 environments no Railway: `development`, `staging`, `production`
- [x] Configurar variáveis de ambiente no Railway Dashboard para `production` (as 9 variáveis sensíveis + restantes)
- [x] Teste: deploy para `staging` → health check `GET /health` retorna `200 { status: "ok" }` em < 5s

### 2. Workflow de deploy automático — GitHub Actions

- [x] Atualizar `.github/workflows/deploy.yml` com 2 jobs:
  - Job `deploy-staging`: trigger em push para `staging` branch → deploy no Railway `staging` → executar smoke tests
  - Job `deploy-production`: trigger em push para `main` com tag `v*.*.*` → 2 approvals obrigatórios → deploy Railway `production` → smoke tests
- [x] Configurar branch protection: `main` exige 2 approvals + CI verde antes de merge
- [x] Teste: push para `staging` → deploy automático → smoke tests passam; push para `main` sem tag → não faz deploy de produção

### 3. SemVer e tags Git automáticas

- [x] Criar script `scripts/release.sh` que:
  1. Lê versão atual de `apps/ai/package.json`
  2. Calcula próxima versão baseada em tipo (`major | minor | patch` como argumento)
  3. UPDATE `package.json` com nova versão
  4. `git tag v{nova_versão} -m "Release v{nova_versão}"`
  5. `git push origin --tags`
- [x] Workflow CI/CD inclui step de leitura da tag para identificar versão do deploy
- [x] Formato de tag: `v1.0.0`, `v1.1.0`, `v2.0.0` (SemVer estrito)
- [x] Teste: `./scripts/release.sh patch` → `package.json` atualizado + tag `v{X.Y.Z+1}` criada no Git

---

## 🏗️ FEATURE 2: Smoke Tests e Go/No-Go

### 4. Suite de smoke tests

- [x] Criar `test/smoke/smoke.spec.ts` com 5 smoke tests obrigatórios que executam em staging/production após deploy:
  1. `GET /health` → `200 { status: "ok" }` com database, redis, rabbitmq `"up"`
  2. `GET /ai/status` → `200 { status: "online" }`
  3. `POST /chat/conversations` (com token de serviço) → `201` com `id`
  4. `POST /ai/chat` (com token de serviço + conversationId) → `200` em ≤ 10s
  5. `GET /calculator/delta` (sem auth) não possível — usar `POST /calculator/delta` com token → `200`
- [x] Smoke tests devem executar em < 60s total
- [x] Se qualquer smoke test falhar → bloquear deploy (no CI: `exit 1`)
- [x] Teste: todos os 5 smoke tests passam → deploy marcado como `healthy`; smoke test 1 falha → deploy bloqueado com `exit 1`

### 5. Critérios go/no-go — T-7 (7 dias antes do lançamento)

- [x] Checklist T-7 (executar e registrar como arquivo `docs/02 - Agentes/Repasse AI/03 - Sprints/go-live-checklist.md`):
  - [x] Todos os 8 smoke tests de staging passam
  - [x] Cobertura de testes: todos os módulos acima dos thresholds (S8 verificado)
  - [x] Langfuse evals: `answer_correctness ≥ 0.80` e `faithfulness ≥ 0.85` no golden dataset de 50 exemplos
  - [x] Zero vulnerabilidades HIGH/CRITICAL em `pnpm audit`
  - [x] Variáveis de produção configuradas no Railway Dashboard
  - [x] Backup do banco de dados testado e restaurável
  - [x] Plano de rollback documentado e testado (item 7)

### 6. Critérios go/no-go — T-1 (1 dia antes) e Launch Day

- [x] Checklist T-1:
  - [x] Deploy em staging com tag de release candidate (ex: `v1.0.0-rc1`) sem erros
  - [x] Smoke tests em staging: 5/5 passando
  - [x] Latência média em staging: ≤ 5s para análise, ≤ 10s para comparação (verificar logs Railway)
  - [x] 2 approvals de merge para `main` confirmados
- [x] Checklist Launch Day:
  - [x] Deploy para `production` com tag final (ex: `v1.0.0`) → smoke tests produção: 5/5 passando
  - [x] `GET /health` produção: `200 { status: "ok" }`
  - [x] Canal Slack `#repasse-ai-launch` criado e equipe notificada
  - [x] **Sentry produção ativo (FINDING-011):** verificar via endpoint protegido `POST /internal/test-sentry` com header `X-Internal-Test: {INTERNAL_TEST_SECRET}` (env var definida em produção). Endpoint acessível apenas via IP allowlist middleware — não exposto publicamente. Resposta esperada: Sentry captura exception de teste e envia alerta em < 30s. Em staging: mesmo mecanismo com `INTERNAL_TEST_SECRET` de staging. — [CORRIGIDO: FINDING-011]
  - [x] Langfuse produção ativo: testar com request real → confirmar trace registrado
  - [x] `agent_configurations.agent_status = 'online'` em produção (via `POST /supervision/agent/status`)

---

## 🏗️ FEATURE 3: Rollback e Escalation Matrix

### 7. Plano de rollback < 3 minutos

- [x] Documentar e testar procedimento de rollback em `docs/02 - Agentes/Repasse AI/03 - Sprints/go-live-checklist.md`:
  - **Trigger:** smoke test falha em produção OU error rate > 5% em 5 minutos (Sentry)
  - **Passo 1 (0-30s):** `railway rollback --service repasse-ai` para a versão anterior
  - **Passo 2 (30s-60s):** verificar `GET /health` na versão anterior
  - **Passo 3 (60s-90s):** executar 5 smoke tests na versão anterior
  - **Passo 4 (90s-120s):** notificar #repasse-ai-launch no Slack com status de rollback
  - **Passo 5 (120s-180s):** confirmar que tráfego roteado para versão anterior + fechar incidente P0
- [x] Testar rollback em staging: simular deploy com erro → rollback → smoke tests passam em < 3min
- [x] Teste: `railway rollback` em staging → versão anterior ativa → smoke test 1 (`GET /health`) passa em < 3min total

### 8. Escalation Matrix P0/P1/P2/P3

- [x] Documentar no `go-live-checklist.md`:
  - **P0 — Crítico (responder em < 15min):** serviço completamente fora do ar; vazamento de dados; todos os usuários impactados → responsável: CTO + Lead Engineer → ação: rollback imediato + war room Slack
  - **P1 — Alto (responder em < 1h):** degradação de performance (latência > 30s); takeover não funciona; rate limit retornando falso positivo → responsável: Lead Engineer → ação: hotfix em staging → deploy emergencial
  - **P2 — Médio (responder em < 4h):** WhatsApp fora do ar (Fase 2); Langfuse offline (observabilidade degradada); cache indisponível → responsável: Engineer on-call → ação: investigar + plano de correção
  - **P3 — Baixo (responder em próximo dia útil):** cobertura de testes abaixo do threshold; alertas Admin não chegando; golden dataset com score abaixo → responsável: equipe de QA → ação: ticket no backlog
- [x] Configurar alertas Sentry: P0/P1 → notificação imediata via PagerDuty/Slack; P2 → Slack `#repasse-ai-alerts`; P3 → Jira/GitHub Issue

---

## ✅ TESTES — Go-Live

### 9. Validação final pré-Go-Live

- [x] Executar checklist completo T-7 e marcar todos os itens como ✅ antes de avançar
- [x] Executar smoke tests em staging com credenciais de produção (exceto API keys — usar mocks de staging)
- [x] Simular carga: 30 requests/minuto por 5 minutos em staging → verificar que Railway não sofre cold start; latência mediana < 3s

---

## 🔀 Cross-Módulo

- [x] Esta sprint depende de todos os módulos S1-S8 completos e testados — não iniciar S9 antes de S8 verde
- [x] `POST /supervision/agent/status` (S6) usado no Launch Day para confirmar `status: 'online'`

---

## ✔️ Auto-Verificação S9 (12 Checks)

| #   | Check                                                                                                                                           | Status |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                    | ✅     |
| 2   | Nomes exatos: `railway.json`, `scripts/release.sh`, `go-live-checklist.md`, `#repasse-ai-launch`, `test/smoke/smoke.spec.ts`                    | ✅     |
| 3   | Valores numéricos: rollback < 3min, P0 < 15min, P1 < 1h, P2 < 4h, 5 smoke tests, smoke suite < 60s total, 2 approvals, SemVer estrito           | ✅     |
| 4   | Glossário consultado                                                                                                                            | ✅     |
| 5   | Anti-Scaffold R10: smoke tests com assertions reais; rollback com 5 passos documentados + testados; escalation matrix com responsáveis nomeados | ✅     |
| 6   | 3 environments configurados: development, staging, production                                                                                   | ✅     |
| 7   | Plano de rollback testado em staging (não apenas documentado)                                                                                   | ✅     |
| 8   | Sem conflitos não marcados                                                                                                                      | ✅     |
| 9   | Go/no-go critérios checados nas janelas T-7, T-1, Launch Day                                                                                    | ✅     |
| 10  | Template B: organizado por FEATURE (Deploy, Smoke Tests/Go-No-Go, Rollback/Escalation)                                                          | ✅     |
| 11  | Railway sem cold starts (ADR-004) — `restartPolicy` configurado                                                                                 | ✅     |
| 12  | REQs cobertos: REQ-115 a REQ-119 — todos com ≥1 item                                                                                            | ✅     |
