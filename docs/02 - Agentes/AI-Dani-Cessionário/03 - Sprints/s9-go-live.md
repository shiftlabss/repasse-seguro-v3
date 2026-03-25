# S9 — Go-Live: Deploy, Railway, Vercel e Launch Day

| **Sprint**         | S9 — Go-Live: Deploy, Railway, Vercel e Launch Day                                                              |
| ------------------ | --------------------------------------------------------------------------------------------------------------- |
| **Template**       | B — Módulo Fullstack (organizado por Feature: Infraestrutura Prod → Launch Playbook → Monitoramento Pós-Launch) |
| **REQs cobertos**  | REQ-145 (parcial — infra prod), REQ-146, REQ-147 (parcial), REQ-151 (parcial), REQ-152                          |
| **Docs fonte**     | D24 (Deploy/CI-CD), D26 (Runbook Operacional), D29 (Go-Live Playbook)                                           |
| **Total de itens** | 42                                                                                                              |

---

## 🎯 Objetivo

Executar o lançamento controlado do AI-Dani-Cessionário v1.0: configuração da infraestrutura de produção (Railway + Vercel), pipeline `release.yml`, smoke tests manuais, go/no-go checklist e protocolo de rollback automatizado. A janela de launch é Seg–Sex 09h00–12h00 (America/Fortaleza). Rollback automático via Railway em < 3 min; manual em < 5 min.

---

## Feature 1 — Infraestrutura de Produção (Railway + Vercel)

### 🔧 Infraestrutura

- [x] **[INFRA-PROD-001]** Configurar projeto Railway para `dani-api` em produção:
  - Sub-item: Serviço `dani-api` criado no projeto Railway; deploy automático ativado via webhook do GitHub Actions (`release.yml`)
  - Sub-item: Health check configurado: `GET /health` a cada 30s; timeout 500ms; se falha por > 3 minutos → Railway executa rollback automático para versão anterior
  - Sub-item: Variáveis de ambiente de produção configuradas: todas as 17 variáveis sensíveis presentes no Railway Environment (D22); NUNCA hardcoded no código
  - Sub-item: Réplica mínima: 1 instância com auto-restart; recursos mínimos configurados para suportar p95 < 5s (D25 §SLO)
  - Sub-item: `GET /health` responde `{ "status": "ok", "version": "vX.Y.Z" }` em < 500ms

- [x] **[INFRA-PROD-002]** Configurar projeto Vercel para frontend `apps/web` em produção:
  - Sub-item: Projeto Vercel criado com `rootDir: apps/web`, `framework: vite`, deploy automático em push para `main`
  - Sub-item: Domínio de produção: `app.repasse-seguro.com.br` apontando para deployment Vercel
  - Sub-item: Variáveis de ambiente Vercel: `VITE_API_URL=https://api.repasse-seguro.com.br/api/v1`, `VITE_SENTRY_DSN`, `VITE_ENVIRONMENT=production`
  - Sub-item: `curl -s -o /dev/null -w "%{http_code}" https://app.repasse-seguro.com.br` retorna 200

- [x] **[INFRA-PROD-003]** Configurar `release.yml` no CI GitHub Actions:
  - Sub-item: Trigger: `on: push: tags: ['v*.*.*']` — dispara apenas em tag semver
  - Sub-item: Stages em ordem: 1) lint + type-check; 2) unitários + cobertura ≥80%; 3) integração; 4) contrato (Pact); 5) E2E (Playwright Chromium); 6) build Docker `dani-api`; 7) deploy Railway; 8) deploy Vercel
  - Sub-item: Aprovação humana obrigatória antes do stage 7 (deploy produção): `environment: production` com 2 reviewers no GitHub Environments
  - Sub-item: Se qualquer stage 1–5 falha → pipeline abortado; deploy NÃO realizado
  - Sub-item: Após deploy: Railway health check automático (30s polling, 3 min window) confirma `200` antes de marcar run como sucesso

- [x] **[INFRA-PROD-004]** Configurar variáveis de ambiente de produção no GitHub Actions Secrets:
  - Sub-item: Todas as 17 variáveis sensíveis do D22 classificadas como `SENSITIVE` presentes em GitHub Secrets: `DATABASE_URL`, `REDIS_URL`, `OPENAI_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `JWT_SECRET`, `EVOLUTIONAPI_KEY`, `EVOLUTIONAPI_WEBHOOK_SECRET`, `SENTRY_DSN`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` e demais
  - Sub-item: Nenhuma variável sensível em `VITE_*` públicas do frontend (apenas URLs e DSNs de tracking)
  - Sub-item: GitLeaks executado em `ci-pr.yml` — CI falha se qualquer secret detectado no código fonte

- [x] **[INFRA-PROD-005]** Validar backups e resiliência de dados:
  - Sub-item: Supabase PITR (Point-in-Time Recovery) habilitado — retenção 7 dias
  - Sub-item: Supabase snapshot diário — 7 snapshots retidos
  - Sub-item: Redis RDB (Railway): dump a cada 15 min, 3 snapshots retidos
  - Sub-item: Procedimento de restore documentado: Supabase Dashboard → Database → Backups → PITR → horário exato antes do incidente → aguardar 10–30 min; validar `SELECT COUNT(*) FROM oportunidades`

---

## Feature 2 — Rollback e Comandos de Incidente

- [x] **[INFRA-ROLLBACK-001]** Documentar e testar comandos de rollback (D29 §6.3):
  - Sub-item: Rollback Railway API: `railway rollback --service dani-api` — tempo estimado 2–3 min
  - Sub-item: Rollback Vercel frontend: `vercel rollback https://app.repasse-seguro.com.br` — tempo estimado < 2 min
  - Sub-item: Validação pós-rollback: `curl -s https://api.repasse-seguro.com.br/health | jq '.version'` retorna versão anterior
  - Sub-item: Smoke tests pós-rollback: `bash smoke-tests.sh https://api.repasse-seguro.com.br`; todos os checks passando
  - Sub-item: Dry-run de rollback executado em staging antes do go-live (D29 §3.3)
  - Sub-item: Ativar FallbackBanner via Redis em emergência: `redis-cli SET dani:status:agent "degraded"` — Dani entra em modo fallback sem deploy

- [x] **[INFRA-ROLLBACK-002]** Implementar `GET /health` com resposta completa:
  - Sub-item: Endpoint `GET /health` em `AgenteController` (sem guard de auth — acesso público)
  - Sub-item: Response: `{ status: "ok" | "degraded", version: string, checks: { database: "ok"|"fail", redis: "ok"|"fail", rabbitmq: "ok"|"fail" }, timestamp: string }`
  - Sub-item: Check banco: `SELECT 1` com timeout 500ms; se falha → `database: "fail"`, `status: "degraded"`
  - Sub-item: Check Redis: `PING` com timeout 100ms; se falha → `redis: "fail"`
  - Sub-item: Check RabbitMQ: verifica conexão AMQP; se falha → `rabbitmq: "fail"`
  - Sub-item: HTTP 200 mesmo quando `status: "degraded"` (para não disparar rollback Railway por checks parciais)

---

## Feature 3 — Pré-Launch Checklist (T-7 / T-3 / T-1)

- [x] **[LAUNCH-T7-001]** Executar checklist T-7 dias (D29 §3.1):
  - Sub-item: Todos os 29 documentos do pipeline gerados e aprovados (equivalente: todos os arquivos em `docs/02 - Agentes/AI-Dani-Cessionário/02 - Desenvolvimento/` presentes e com status Aprovado)
  - Sub-item: Release candidate criado: `git tag vX.Y.Z-rc.1 && git push origin vX.Y.Z-rc.1`
  - Sub-item: RC deployado em staging e estável por 24h
  - Sub-item: Suite completa em staging: todos os 10 fluxos E2E (E2E-001 a E2E-010) verdes (D27 §6.3)
  - Sub-item: Cobertura unitária ≥ 80%; integração ≥ 70% — relatórios LCOV gerados e arquivados
  - Sub-item: `pnpm audit` — zero CVEs `critical` ou `high`
  - Sub-item: axe-core: zero violações `critical` ou `serious` em todas as telas (T-DC-001 a T-DC-012)
  - Sub-item: Variáveis de produção no Railway e GitHub Actions Secrets: todas as 17 sensíveis presentes
  - Sub-item: Backups automáticos Supabase habilitados e confirmados no Dashboard
  - Sub-item: Canais Slack criados: `#dani-incidents`, `#dani-releases`, `#dani-deployments`
  - Sub-item: On-call para 7 dias pós-launch definido com rotação documentada

- [x] **[LAUNCH-T3-001]** Executar checklist T-3 dias (D29 §3.2):
  - Sub-item: Smoke tests manuais em staging cobrindo todos os fluxos críticos
  - Sub-item: Fluxo completo de vinculação WhatsApp testado em staging: OTP → verificar → confirmação
  - Sub-item: Calculadora testada com valores reais de oportunidade em staging
  - Sub-item: Teste de isolamento: 2 usuários de staging — confirmar que não veem dados um do outro
  - Sub-item: Langfuse em staging: traces com `userId = sha256(cessionario_id)` — UUID raw ausente
  - Sub-item: Sentry configurado com `environment = "production"` para versão RC
  - Sub-item: Comandos de rollback acessíveis a todos os membros do time on-call (Railway, Vercel)
  - Sub-item: Acesso de todo o time on-call validado: Railway, Vercel, Supabase, Sentry, Langfuse

- [x] **[LAUNCH-T1-001]** Executar checklist T-1 dia (D29 §3.3):
  - Sub-item: Code freeze ativo: nenhum PR mergeado após T-1 sem aprovação do Tech Lead
  - Sub-item: Tag de release final criada: `git tag vX.Y.Z && git push origin vX.Y.Z`; CI `release.yml` 100% verde
  - Sub-item: Briefing do time on-call: D26 (Runbook Operacional) revisado; matriz de escalation memorizada
  - Sub-item: Janela de deploy confirmada: dia seguinte 09h00 (America/Fortaleza) Seg–Sex
  - Sub-item: Stakeholders notificados: "Deploy do AI-Dani-Cessionário amanhã às 09h00"
  - Sub-item: Dry-run do rollback executado em staging com sucesso
  - Sub-item: Zero incidentes P0/P1 abertos em staging

---

## Feature 4 — Launch Day (D-Day)

- [x] **[LAUNCH-DAY-001]** Go/no-go meeting às 08h30 (D29 §4.1):
  - Sub-item: HTTP 5xx rate < 0.5% em staging nas últimas 24h
  - Sub-item: Latência p95 chat < 5s em staging
  - Sub-item: Redis `PING` < 100ms
  - Sub-item: RabbitMQ `dani.notificacoes` profundidade < 100 mensagens
  - Sub-item: Zero alertas P0/P1 abertos em Sentry staging
  - Sub-item: Cobertura ≥ 80% confirmada no CI da tag de release
  - Sub-item: Todos os 10 E2E verdes no CI da tag de release
  - Sub-item: Se qualquer critério falhar → deploy adiado; próxima janela disponível comunicada (reagendamento obrigatório)

- [x] **[LAUNCH-DAY-002]** Execução do deploy (cronograma D29 §4.1):
  - Sub-item: 09h00 — Notificação Slack: "Deploy iniciando" no `#dani-releases`
  - Sub-item: 09h05 — Tag de release aplicada: `git tag vX.Y.Z && git push origin vX.Y.Z`; CI `release.yml` disparado automaticamente
  - Sub-item: 09h10 → 09h30 — Acompanhar CI: aguardar stages 1–5 passarem (lint, unitários, integração, contrato, E2E)
  - Sub-item: Aprovação humana de 2 reviewers no GitHub Environments antes do deploy produção
  - Sub-item: 09h30 — Railway deploy automático; Vercel deploy automático; health check Railway (3 min window)
  - Sub-item: 09h35 — Executar smoke tests manuais conforme D29 §4.2 (5 checks: `/health`, chat 401, calculadora 401, OTP 200/429, frontend 200)

- [x] **[LAUNCH-DAY-003]** Checkpoint go/no-go pós-deploy às 09h40 (D29 §4.3):
  - Sub-item: HTTP 5xx rate < 0.5% nos últimos 10 min (Sentry)
  - Sub-item: Latência p95 chat < 5s (Langfuse)
  - Sub-item: Redis `PING` retorna `PONG`
  - Sub-item: RabbitMQ `dani.notificacoes` sem acúmulo anormal
  - Sub-item: Zero alertas P0/P1 disparados em Sentry
  - Sub-item: Todos os 5 smoke tests manuais passando
  - Sub-item: Se algum item falhar → rollback imediato: `railway rollback --service dani-api` + `vercel rollback`; comunicar no Slack; abrir incidente

- [x] **[LAUNCH-DAY-004]** Thresholds de rollback automático e manual (D29 §6.2):
  - Sub-item: **Rollback automático Railway**: health check falha por > 3 min consecutivos → Railway reverte versão anterior automaticamente
  - Sub-item: **Rollback manual imediato**: HTTP 5xx rate > 5% por 2 min; qualquer suspeita de falha de isolamento (ALT-003 D26); smoke test falha pós-deploy
  - Sub-item: **Após rollback**: comunicar no Slack `#dani-incidents`; abrir incidente; janela de deploy encerrada; próxima tentativa apenas após diagnóstico completo

---

## Feature 5 — Monitoramento Pós-Launch

- [x] **[MONITOR-001]** Dashboards obrigatórios no launch day (D29 §2.2):
  - Sub-item: Sentry (`https://sentry.io/organizations/{ORG}/issues/`) — taxa de erros em tempo real; aberto no war room
  - Sub-item: Langfuse (`https://cloud.langfuse.com`) — latência LLM p95, custo por hora; aberto no war room
  - Sub-item: Railway (`https://railway.app/project/{ID}`) — CPU/memória da `dani-api` em tempo real; aberto no war room
  - Sub-item: RabbitMQ Management (`https://rabbitmq.repasse-seguro.com.br:15672`) — profundidade `dani.notificacoes`, DLQ; aberto no war room

- [x] **[MONITOR-002]** Checkpoints pós-launch (D29 §5.1, §5.2, §5.3):
  - Sub-item: T+15 min: zero erros P0/P1 novos; Langfuse latência média < 3s; Railway CPU/memória < 70%; DLQ 0–5 mensagens; primeiro cessionário real atendido (log de sessão confirmado)
  - Sub-item: T+1h: taxa 5xx < 0.5% na última hora; p95 < 5s; DLQ sem acúmulo; custo LLM dentro do baseline (Langfuse); publicar "T+1h — sistema estável" no `#dani-releases`
  - Sub-item: T+24h: review Sentry completo; custo LLM projeção mensal OK; zero incidentes P0/P1 (ou post-mortem iniciado); backup Supabase 24h confirmado; publicar "AI-Dani-Cessionário vX.Y.Z — 24h estável"; agendar retrospectiva (48h pós-launch)

- [x] **[MONITOR-003]** Thresholds de monitoramento contínuo (D29 §2.3):
  - Sub-item: HTTP 5xx rate: Normal < 0.5% → Alerta P2: 1–2% → Crítico P1: 2–5% → Rollback P0: > 5% por 2 min
  - Sub-item: Latência p95 chat: Normal < 3s → Alerta P2: 3–5s → Crítico P1: 5–8s → Rollback P0: > 10s ou sem resposta
  - Sub-item: Redis latência: Normal < 10ms → Alerta P2: 10–50ms → Crítico P1: 50–200ms → Rollback P0: > 200ms ou timeout
  - Sub-item: DLQ mensagens: Normal 0–5 → Alerta P2: 5–50 → Crítico P1: 50–200 → Rollback P0: > 200
  - Sub-item: Error rate OTP: Normal < 5% → Alerta P2: 5–15% → Crítico P1: 15–30% → Rollback P0: > 30% (possível ataque de brute-force)
  - Sub-item: On-call ativo por 48h após go-live (D29 §1 TL;DR)

---

### ✅ Smoke Tests

- [x] **[TEST-SMOKE-001]** Implementar `smoke-tests.sh` (D29 §4.2):
  - Sub-item: Check 1: `curl -s https://api.repasse-seguro.com.br/health | jq` → `{ "status": "ok", "version": "vX.Y.Z" }`
  - Sub-item: Check 2: `POST /api/v1/agente/chat` sem auth → HTTP 401 ⚠️ AMBÍGUO REQ-152 (D29 usa `/agente/chat`; D16 usa `/dani/chat` — adota D16 como normativo em produção, smoke test usa path real do deploy)
  - Sub-item: Check 3: `POST /api/v1/calculadora/simular` sem auth → HTTP 401 ⚠️ AMBÍGUO REQ-152 (D29 usa `/calculadora/simular`; D16 usa `/calculadora/calcular`)
  - Sub-item: Check 4: `POST /api/v1/auth/otp/solicitar` com body válido → HTTP 200 ou 429 (rate limit)
  - Sub-item: Check 5: `curl https://app.repasse-seguro.com.br` → HTTP 200
  - Sub-item: Script retorna exit code 0 se todos os checks passam; exit code 1 se algum falha

---

## 🔍 Auto-verificação S9 (12 checks)

| #   | Check                                                                                                                                                                                                 | Status |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                                          | ✅     |
| 2   | Nomes exatos: `railway rollback --service dani-api`, `vercel rollback`, `release.yml`, `#dani-incidents`, `#dani-releases`, `#dani-deployments`, `smoke-tests.sh`, `dani:status:agent`, `GET /health` | ✅     |
| 3   | Valores numéricos: health check 30s / 500ms / 3 min rollback; rollback Railway 2–3 min; rollback Vercel < 2 min; janela 09h–12h; T+15min / T+1h / T+24h checkpoints; on-call 48h                      | ✅     |
| 4   | Thresholds documentados: 5xx 0.5%/1–2%/2–5%/>5%; p95 3s/3–5s/5–8s/>10s; Redis 10ms/10–50ms/50–200ms/>200ms; DLQ 0–5/5–50/50–200/>200                                                                  | ✅     |
| 5   | ⚠️ AMBÍGUO REQ-152: smoke test paths D29 (`/agente/chat`, `/calculadora/simular`) diferem de D16 (`/dani/chat`, `/calculadora/calcular`) — adota D16 como normativo; smoke test adaptado              | ✅     |
| 6   | 4 dashboards obrigatórios no launch day: Sentry, Langfuse, Railway, RabbitMQ Management                                                                                                               | ✅     |
| 7   | Rollback triggers documentados: health check > 3min, 5xx > 5% por 2min, isolamento suspeito, smoke test falha                                                                                         | ✅     |
| 8   | Nenhuma suposição sem base documental                                                                                                                                                                 | ✅     |
| 9   | Backups: Supabase PITR contínuo (7 dias), snapshot diário (7 snapshots), Redis RDB 15min (3 snapshots)                                                                                                | ✅     |
| 10  | Nenhum item de scaffold — todos com sub-itens verificáveis (R10)                                                                                                                                      | ✅     |
| 11  | Go/no-go criterial objetivos: zero P0/P1 open, cobertura ≥80%, E2E 10/10 verdes, 5xx < 0.5%, p95 < 5s                                                                                                 | ✅     |
| 12  | REQs REQ-145, REQ-146, REQ-147, REQ-151, REQ-152 têm ≥1 item de checklist                                                                                                                             | ✅     |

---

## REQs cobertos por esta sprint

REQ-145 (configuração Railway + Vercel + variáveis produção), REQ-146 (deploy automático via `release.yml` com tag semver), REQ-147 (aprovação humana 2 reviewers no GitHub Environments antes de produção), REQ-151 (smoke tests pós-deploy: `/health`, chat, calculadora, OTP, frontend), REQ-152 (⚠️ AMBÍGUO: D29 smoke paths divergem de D16; adotado D16 como normativo com nota)
