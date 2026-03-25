# S10 — Go-Live

| Campo                | Valor                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S10                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Nome**             | Go-Live                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Tipo**             | Infraestrutura de Deploy (Template A)                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Template**         | A (Infraestrutura)                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **Docs Consultados** | D02 (stacks Railway), D24, D25, D26, D29                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Cross-cutting**    | D10 (Glossário)                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **REQs cobertos**    | REQ-131, REQ-132, REQ-133, REQ-134, REQ-135, REQ-136, REQ-137, REQ-138, REQ-139, REQ-140, REQ-141, REQ-142, REQ-143, REQ-144, REQ-145, REQ-146, REQ-147                                                                                                                                                                                                                                                                                  |
| **Objetivo**         | GitHub Actions `ci.yml` + `deploy.yml`, deploy Railway zero-downtime com rolling update, 3 ambientes (development/staging/production), `GET /health/ready` + `GET /health/live`, smoke test manual (6 etapas), checklist T-7/T-3/T-1, critérios Go/No-Go, rollback via `railway rollback` em < 2 minutos, runbook RB-001 (circuit breaker), logs operacionais, snapshot pré-Go-Live, Semantic Versioning MAJOR.MINOR.PATCH com tags `v*` |

---

## Critério de Conclusão da S10

Ao final desta sprint:

- `ci.yml` executa em todo PR: lint, testes, `npm audit --audit-level=moderate`, cobertura
- `deploy.yml` disparado por tag `v*` em `main`: build Docker → push Railway → health check → smoke test
- `GET /health/ready` retorna 200 com `{ checks: { database, redis, rabbitmq } }` ou 503 se qualquer check falhar
- `GET /health/live` retorna 200 em < 50ms sem dependências externas
- Railway configurado com 3 environments: `development`, `staging`, `production`
- Rollback via `railway rollback --deployment <id>` executável em < 2 minutos
- Smoke test de 6 etapas documentado e executável por qualquer membro do time
- Critérios Go/No-Go documentados: taxa de erro < 2%, p95 ≤ 5s, circuit breaker fechado
- Critério de rollback automático: `/health/ready` 5xx OU taxa > 2% por 5min OU circuit breaker abrindo nos primeiros 15min

---

## ⚙️ FEATURE 1 — GitHub Actions CI/CD

### Backend — CI Pipeline

- [x] **Criar `.github/workflows/ci.yml`** com jobs executados em todo PR: (1) job `lint`: `npm run lint` (ESLint + `no-console` rule); (2) job `type-check`: `npm run build` (TypeScript strict, `tsc --noEmit`); (3) job `test`: `npm run test:cov` com thresholds 80%/90% para módulos críticos; (4) job `security`: `npm audit --audit-level=moderate` — falha se vulnerabilidades moderate ou críticas; (5) jobs paralelos onde possível (lint + type-check paralelos; test após ambos); (6) cache de `node_modules` via `actions/cache@v4` com key baseada em `package-lock.json`
  - Validação: PR com `console.log` → CI falha em `lint`; PR com 79% de cobertura em auth → CI falha em `test`; PR com vulnerabilidade moderate → CI falha em `security`; todos os jobs concluem em < 5 minutos no total

- [x] **Criar `.github/workflows/deploy.yml`** disparado por push de tag `v*` em `main`: (1) job `build`: `docker build -t dani-cedente:{tag} .`; (2) job `deploy-staging` (apenas para tags `v*-rc*`): deploy no Railway environment `staging`; aguarda health check; (3) job `deploy-production` (apenas para tags `v[0-9]*.[0-9]*.[0-9]*` sem rc): deploy no Railway environment `production`; aguarda health check; executa smoke test; (4) ambos os jobs usam `RAILWAY_TOKEN` secret configurado no repositório; (5) notifica `#ops-incidents` via Slack webhook ao iniciar e concluir deploy
  - Validação: tag `v1.2.3-rc.1` → deploy staging; tag `v1.2.3` → deploy production; health check falhando após deploy → job falha (não silencioso); Slack notificação enviada

### Backend — Dockerfile

- [x] **Criar `Dockerfile`** multi-stage em root do projeto: (1) **stage `builder`**: `FROM node:22-alpine AS builder`; instala dependências com `npm ci --omit=dev`; executa `npm run build` (output em `dist/`); (2) **stage `production`**: `FROM node:22-alpine`; copia apenas `dist/`, `node_modules/` (produção), `package.json`, `prisma/schema.prisma`; usuário não-root (`USER node`); `EXPOSE 3001`; `CMD ["node", "dist/main.js"]`; (3) `.dockerignore` exclui: `.env*`, `*.spec.ts`, `test/`, `coverage/`, `node_modules/.cache`
  - Validação: `docker build -t dani-cedente:test .` conclui sem erro; imagem final < 300MB; usuário não-root (segurança); `dist/main.js` presente na imagem; `.env` nunca na imagem

---

## ⚙️ FEATURE 2 — Railway Deploy

### Backend — Configuração Railway

- [x] **Configurar `railway.json`** (ou `railway.toml`) em root: (1) service name: `dani-cedente`; (2) healthcheck: `{ "path": "/api/v1/health/ready", "timeout": 10, "interval": 30 }`; (3) `startCommand: "node dist/main.js"`; (4) `restartPolicyType: "on_failure"`, `restartPolicyMaxRetries: 3`; (5) região: `us-east` ou conforme preferência do time; (6) `numReplicas: 1` (Fase 1); escala horizontal prevista para Fase 2
  - Validação: `railway validate` sem erros; health check configurado para `/api/v1/health/ready`; restart policy ativa

- [x] **Documentar variáveis de ambiente obrigatórias para produção** em `src/docs/env-production.md`: (1) lista das 22+ variáveis de `src/docs/env.example.md` (de S1); (2) separadas por ambiente: `development` (localhost), `staging` (railway.app staging), `production` (railway.app prod); (3) marcadas como `REQUIRED` ou `OPTIONAL`; (4) instrução: nenhuma variável de staging/test deve estar em produção; (5) variável `NOTIFICATION_MODE=production` obrigatória em produção (habilita envio real de e-mails)
  - Validação: documento criado; `NOTIFICATION_MODE` listada; instrução de verificação: `railway vars list --environment production` antes do Go-Live

- [x] **Configurar 3 environments no Railway**: (1) `development` — conecta a `localhost:5432` (docker-compose local), `localhost:6379`, `localhost:5672`; (2) `staging` — Railway PostgreSQL + Railway Redis + Railway RabbitMQ addons; `NODE_ENV=staging`; (3) `production` — Supabase PostgreSQL, Railway Redis, Railway RabbitMQ; `NODE_ENV=production`; `LOG_LEVEL=warn`
  - Validação: 3 environments no Railway dashboard; variáveis distintas por environment; `LOG_LEVEL=debug` apenas em development/staging; `NODE_ENV=production` no production environment

- [x] **Configurar rolling deploy zero-downtime** no Railway: (1) Railway executa rolling update por padrão — validar que health check `GET /health/ready` está configurado para esperar que nova instância responda antes de desligar a antiga; (2) `startupTimeout: 60` segundos na config Railway; (3) Railway health check interval: 30s, threshold de falha: 3 checks consecutivos → rollback automático
  - Validação: deploy de nova versão com health check demora > 30s → Railway aguarda; deploy com health check falhando 3× → Railway reverte automaticamente; zero downtime verificado nos logs durante deploy

---

## ⚙️ FEATURE 3 — Runbooks Operacionais

### Runbook RB-001 — Circuit Breaker

- [x] **Criar runbook `src/docs/runbooks/RB-001-circuit-breaker.md`**: (1) título: "RB-001 — Circuit Breaker Aberto"; (2) sintoma: agente retorna 503 para todos os requests de chat; (3) diagnóstico: `curl -H "Authorization: Bearer {admin_token}" https://dani.repasseseguro.com.br/api/v1/admin/fallback/status` → espera `{ "enabled": false }`; (4) verificar logs: `railway logs --tail 100 --grep "circuit_open"`; (5) verificar Sentry: evento `circuit_breaker_opened` com `error_rate`; (6) reativação: `curl -X POST -H "Authorization: Bearer {admin_token}" https://dani.repasseseguro.com.br/api/v1/admin/fallback/enable` → espera `{ "enabled": true }`; (7) verificar estabilização: taxa de erro < 10% por 5 minutos consecutivos antes de reativar
  - Validação: runbook criado com todos os 7 passos; comandos `curl` com placeholders documentados; critério de estabilização antes de reativar documentado

- [x] **Criar runbook `src/docs/runbooks/RB-002-rollback.md`**: (1) título: "RB-002 — Rollback de Deploy"; (2) gatilhos para rollback imediato: `/health/ready` 5xx; taxa de erro > 2% por > 5 minutos; circuit breaker abrindo nos primeiros 15min; smoke test falhando; dados cruzados de Cedente detectados; (3) execução: `railway deployments list --service dani-cedente-prod`; `railway rollback --deployment <deployment-id-anterior>`; `sleep 30`; `curl -f https://dani.repasseseguro.com.br/api/v1/health/ready && echo "ROLLBACK OK"`; (4) comunicar no `#ops-incidents`: `"[ROLLBACK INICIADO] Motivo: {motivo}"` e `"[ROLLBACK CONCLUÍDO] Revertido para vX.Y.Z-1"`; (5) pós-rollback: snapshot manual do banco; post-mortem agendado para 24h; (6) **decisão do Tech Lead — sem consenso necessário em emergência**
  - Validação: runbook criado com comandos Railway exatos; critérios de gatilho exatos conforme D29; tempo esperado `< 2 minutos` documentado; instrução de comunicação nos templates exatos de D29

### Checklist T-7 / T-3 / T-1

- [x] **Criar `src/docs/go-live-checklist.md`** com os 3 checklists de D29**: (1) **T-7**: 7 itens (documentos técnicos revisados, testes E2E passando, integrações testadas em staging, checklist D28 auditado, secrets de produção configurados, backup staging validado, runbook revisado pelo time); (2) **T-3**: 7 itens (deploy staging com versão exata, smoke test 6/6, alertas testados, Cedentes piloto cadastrados, comunicação interna, rollback testado em staging, plano comunicação aprovado); (3) **T-1\*\*: 7 itens (freeze de deploys, PR com 2 aprovações, health check prod manual, dashboard Grafana monitorado 1h, comunicação final, on-call confirmado, `NOTIFICATION_MODE=production` configurado)
  - Validação: 21 itens no total (7×3); itens exatos conforme D29 seção 3; todos verificáveis (sim/não); pendências marcadas com `[PENDÊNCIA: ...]`

---

## ⚙️ FEATURE 4 — Smoke Test e Go/No-Go

### Smoke Test Manual (6 etapas)

- [x] **Criar `src/docs/smoke-test.md`** com as 6 etapas de D29 §2.3\*\*: (1) "Abrir chat widget como Cedente de teste → mensagem de saudação recebida?"; (2) "Perguntar sobre oportunidade cadastrada → Dani responde com dados corretos?"; (3) "Simular início de análise de proposta → Dani entra no estado correto?"; (4) "Verificar que outro Cedente não vê dados do primeiro → 404 retornado?"; (5) "Enviar 30 mensagens rápidas → rate limit ativo na 31ª?"; (6) "Verificar notificação de teste → toast exibido no widget?"; para cada etapa: pré-condições, passos exatos, resultado esperado, critério de aprovação (✅/❌)
  - Validação: 6 etapas com pré-condições documentadas; etapa 4 verifica isolamento de dados (Cedente A não acessa Cedente B); etapa 5 verifica rate limit exato (30 msg/h); etapa 6 verifica SSE push funcional

### Critérios Go/No-Go

- [x] **Criar `src/docs/go-no-go-criteria.md`** com critérios exatos de D29**: (1) **Go**: `GET /health/ready` → 200; taxa de erro < 2%; p95 ≤ 5s; circuit breaker `CLOSED`; DLQ = 0; smoke test 6/6; (2) **No-Go / Rollback automático\*\*: `GET /health/ready` retorna 5xx (qualquer momento nos primeiros 30min); taxa de erro > 2% por > 5 minutos; circuit breaker abrindo nos primeiros 15min; smoke test falhando após deploy; dados de Cedente vazando (qualquer momento → P0 + DPO); (3) dashboards obrigatórios: Saúde Geral (Grafana), Agente IA (Grafana/Langfuse), Notificações (Grafana), PostHog CSAT com thresholds exatos conforme D29 §2.2
  - Validação: critérios Go e No-Go em tabela; thresholds numéricos exatos; instrução: "Tech Lead decide — sem consenso necessário em emergência" para rollback

---

## ⚙️ FEATURE 5 — Testes de Deploy

- [x] **Criar script de smoke test automatizado** em `scripts/smoke-test.sh`: (1) recebe `BASE_URL` e `ADMIN_TOKEN` como argumentos; (2) executa as 6 etapas com `curl`; (3) retorna exit code 0 se todos passam, 1 se algum falha; (4) usado pelo `deploy.yml` como step pós-deploy; (5) timeout de 30s por request
  - Validação: `bash scripts/smoke-test.sh https://staging.dani.app admin_token` retorna exit 0 em ambiente staging; exit 1 se health check falhar; script executa em < 2 minutos

- [x] **Criar `test/deploy.e2e-spec.ts`** com verificações de integridade pós-deploy\*\*: (1) `GET /health/ready` → 200 com `checks.database: 'ok'`; (2) `GET /health/live` → 200 com `{ status: 'ok' }`; (3) `GET /auth/me` sem token → 401 `DCE-AUTH-4010_003`; (4) `POST /admin/fallback/enable` com token cedente → 403; (5) rate limit de chat: 30 requisições consecutivas passam, 31ª retorna 429
  - Validação: 5 verificações de sanidade pós-deploy; executável em ambiente staging antes do Go-Live; todos os testes passam em < 1 minuto

---

## 🔍 AUTO-VERIFICAÇÃO S10

| Check              | Critério                                                                                                                                                    | Status |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 CI/CD           | `ci.yml` com jobs lint + type-check + test (cobertura gates) + security; `deploy.yml` disparado por tag `v*`; ambos em `.github/workflows/`                 | [x]    |
| #2 Deploy          | Dockerfile multi-stage com usuário não-root; Railway health check em `/api/v1/health/ready`; 3 environments (development, staging, production) configurados | [x]    |
| #3 Health Checks   | `GET /health/live` sem dependências (< 50ms); `GET /health/ready` verifica DB + Redis + RabbitMQ; 503 se qualquer falhar; Railway probe configurada         | [x]    |
| #4 Rollback        | `railway rollback` executável em < 2 minutos; RB-002 documentado com comandos exatos; critérios de gatilho documentados conforme D29                        | [x]    |
| #5 Smoke Test      | 6 etapas documentadas conforme D29 §2.3; script `smoke-test.sh` automatizado; inclui verificação de isolamento (etapa 4) e rate limit (etapa 5)             | [x]    |
| #6 Go/No-Go        | Critérios exatos: erro < 2%, p95 ≤ 5s, circuit breaker fechado, DLQ = 0, smoke test 6/6; critérios de No-Go com thresholds de D29                           | [x]    |
| #7 Runbooks        | RB-001 (circuit breaker) e RB-002 (rollback) com comandos exatos; RB-002 com instrução "Tech Lead decide sem consenso em emergência"                        | [x]    |
| #8 Checklists      | T-7 (7 itens), T-3 (7 itens), T-1 (7 itens) em `go-live-checklist.md`; itens exatos conforme D29 seção 3                                                    | [x]    |
| #9 Versionamento   | Tags `v*` disparam deploy production; tags `v*-rc*` disparam deploy staging; Semantic Versioning MAJOR.MINOR.PATCH                                          | [x]    |
| #10 Secrets        | Nenhuma credencial real no código; `NOTIFICATION_MODE=production` obrigatório; instrução de verificar `railway vars list` antes do Go-Live                  | [x]    |
| #11 Zero Downtime  | Rolling deploy configurado; `startupTimeout: 60s`; health check threshold 3 falhas consecutivas → rollback Railway automático                               | [x]    |
| #12 Cobertura REQs | REQ-131 a REQ-147 — todos com ≥1 item no checklist                                                                                                          | [x]    |
