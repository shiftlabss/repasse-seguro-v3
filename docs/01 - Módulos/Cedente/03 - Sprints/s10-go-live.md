# S10 — Go-Live

## Módulo Cedente · Repasse Seguro · Sprint 10

| Campo             | Valor                                                                                    |
| ----------------- | ---------------------------------------------------------------------------------------- |
| **Sprint**        | S10 — Go-Live                                                                            |
| **Tipo**          | Template B — Módulo Fullstack por Feature                                                |
| **Docs Fonte**    | D29 (Go-Live Playbook), D24 (CI/CD), D25 (Observabilidade), D28 (Checklist de Qualidade) |
| **REQs cobertos** | REQ-212, REQ-213, REQ-214, REQ-215                                                       |
| **Dependências**  | S1–S9 todos aprovados; nenhum gate de release pendente                                   |
| **Status**        | Pendente                                                                                 |

---

## Contexto

Esta sprint prepara e executa o lançamento em produção do módulo Cedente. Não adiciona lógica de negócio. Foco em: validação Go/No-Go, infraestrutura de produção, smoke tests, monitoramento pós-launch, feature flag e plano de rollback. Janela de deploy: terça a quinta, 10h–14h BRT.

---

## FEATURE 1 — Infraestrutura de Produção

### Backend e Infraestrutura

- [x] **3 projetos Supabase separados** (D29 seção 3.1):
  - `repasse-seguro-dev` (desenvolvimento local)
  - `repasse-seguro-staging` (staging)
  - `repasse-seguro-prod` (produção)
  - Nunca compartilhar projetos entre ambientes
  - Validar: `supabase projects list` → 3 projetos listados; cada um com URL e anon key distintos

- [x] **RLS em todas as tabelas de Cedentes em produção** (D29 seção 3.1):
  - Executar: `SELECT tablename, policyname FROM pg_policies WHERE tablename IN ('users','cedentes','casos','dossies','documentos_dossie','propostas','envelopes_assinatura','contas_escrow','notificacoes','eventos_caso','ai_sessions','push_tokens')`
  - Resultado esperado: 12 tabelas com pelo menos 1 policy cada
  - Validar: query em staging e prod retorna 12 tabelas com policies; qualquer tabela faltando → bloqueante de go-live

- [x] **Railway service `cedente-api-prod`** configurado (D29 seção 3.1):
  - Health check: path `/health`, timeout 30s, interval 10s
  - Restart policy: `on-failure`, máx 3 tentativas
  - Memória mínima: 512 MB; CPU: 0.5 vCPU
  - Região: `us-east` (melhor latência para Brasil)
  - Validar: `railway status --service cedente-api-prod` → healthy; `/health` responde < 500ms

- [x] **Vercel project `cedente-web`** configurado (D29 seção 3.1):
  - Domínio: `cedente.repasseseguro.com.br`
  - Environment variables de produção configuradas no Vercel (não commitadas)
  - Validar: `vercel env ls --environment production` → variáveis listadas sem valores visíveis

- [x] **Upstash Redis prod** criado e conectado (D29 seção 3.1):
  - `REDIS_URL` configurado no Railway backend prod
  - Validar: `redis-cli ping` → PONG; memória < 70%; operações/s dentro do esperado

- [x] **Secrets de produção configurados** — 14 secrets obrigatórios (D29 seção 3.1):
  - Railway: `ZAPSIGN_API_KEY`, `ZAPSIGN_WEBHOOK_SECRET`, `ESCROW_API_KEY`, `ESCROW_WEBHOOK_SECRET`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `JWT_SECRET`, `REDIS_URL`, `OPENAI_API_KEY`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `RESEND_API_KEY`, `SENTRY_DSN`
  - Vercel: `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_SENTRY_DSN`, `NEXT_PUBLIC_POSTHOG_KEY`, `POSTHOG_API_KEY`
  - EAS Secrets: `EXPO_TOKEN`, `SENTRY_DSN`
  - Nenhum secret commitado no código ou `.env`
  - `.env.example` com todas as 42 variáveis (REQ-201) sem valores
  - Validar: `pnpm audit --audit-level=high` → sem vulnerabilidades críticas; GitHub Secret Scanning → sem alertas

- [x] **Feature flag `CEDENTE_MODULE_ENABLED`** (D29 seção 7.2):
  - Via variável de ambiente Railway OU PostHog feature flag
  - Quando `false`: rotas `/cedente/*` retornam HTTP 503 com mensagem "Módulo temporariamente indisponível"
  - Sidebar web não exibe itens do Módulo Cedente quando flag = false
  - App mobile exibe banner "Módulo temporariamente indisponível" quando flag = false
  - Outros módulos (Cessionário, Admin) continuam operando normalmente
  - Validar: `CEDENTE_MODULE_ENABLED=false` → `GET /cedentes/casos` → HTTP 503; outros endpoints (Auth, Admin) → normais

---

## FEATURE 2 — Health Checks e Smoke Tests (REQ-213)

### Health Checks

- [x] **`GET /health`** — já implementado em S1; validar conformidade final (REQ-213):
  - Response: `{ "status": "ok", "timestamp": string }` HTTP 200 em < 500ms
  - Verifica: conexão Postgres, conexão Redis, ping ZapSign, ping Escrow (D28 seção 5.5)
  - Response com componente degradado: `{ "status": "degraded", "components": { "postgres": "ok", "redis": "ok", "zapsign": "error", "escrow": "ok" } }` HTTP 503
  - Railway health check: reinicializar após 3 falhas consecutivas (configurado em S9)
  - Validar: `curl https://api-cedente.up.railway.app/health` → HTTP 200 em < 500ms; Postgres down simulado → HTTP 503

- [x] **4 health check endpoints** — verificar a cada 5min durante launch window (D29 seção 2.1):
  - `GET /health` → HTTP 200, `{ "status": "ok" }` em < 500ms
  - `GET /api/v1/cedente/casos` (com Bearer token válido) → HTTP 200, latência < 500ms
  - `POST /api/v1/auth/login` (payload de teste) → HTTP 200 ou 401 (nunca 5xx)
  - `GET /api/v1/cedente/simulador` (com Bearer token válido) → HTTP 200, latência < 300ms
  - Validar: os 4 endpoints respondem conforme especificado em staging antes do go-live

- [x] **`pnpm test:smoke:prod`** — script de smoke tests automatizados (D29 seção 2.3):
  - Executa imediatamente após deploy em produção via CI
  - Testes:
    1. `GET /health` → HTTP 200 em < 500ms
    2. `GET /api/v1/cedente/casos` (sem token) → HTTP 401 (não 5xx)
    3. `POST /api/v1/auth/login` (payload inválido) → HTTP 400 (não 5xx)
    4. `GET /api/v1/cedente/simulador` (sem token) → HTTP 401 (não 5xx)
    5. `POST /api/v1/cedente/guardiao/chat` (sem token) → HTTP 401 (não 5xx)
  - Falha em qualquer teste → `exit code 1` → rollback automático via CI
  - Validar: script presente em `scripts/smoke-test-prod.sh`; execução em staging simula todos os 5 casos

### Dashboards Críticos

- [x] **5 dashboards configurados e monitorados** durante launch window (D29 seção 2.2):
  - Railway Metrics: CPU, memória, instâncias ativas — threshold: CPU > 80%, memória > 85%
  - Sentry: taxa de erros, novos issues — threshold: novos P0 OU taxa > 1% em 5min
  - Supabase Dashboard: conexões, latência de queries — threshold: conexões > 80% do pool
  - Langfuse: latência Guardião, % escalamentos — threshold: p95 > 5s, escalamentos > 20%
  - Upstash Redis: memória, operações/s — threshold: memória > 70%, ping sem resposta
  - Validar: links dos 5 dashboards documentados em `docs/dashboards-prod.md`; alertas configurados para todos os thresholds

---

## FEATURE 3 — Pré-Launch: T-7, T-3, T-1

### T-7 (7 dias antes do go-live)

- [x] **Testes e qualidade — T-7** (D29 seção 3.1):
  - Suíte E2E completa (E2E-001 a E2E-007 de S9) passando em staging sem falhas intermitentes
  - `pnpm audit --audit-level=high` → sem vulnerabilidades críticas
  - D28 (Checklist de Qualidade S9) revisado e aprovado pelo Tech Lead
  - Testes de segurança SEC-001 a SEC-005 (S9) passando

- [x] **Integrações — T-7** (D29 seção 3.1):
  - ZapSign: webhook URL de staging configurada no painel ZapSign; teste com payload real de Termo de Cadastro
  - ZapSign: INT-020 (HMAC-SHA256) testado com payload válido e inválido
  - Escrow: credenciais sandbox configuradas; fluxo depósito + distribuição testado no sandbox
  - Escrow: webhook sandbox testado com payload de depósito confirmado, distribuição e estorno
  - Resend: domínio de produção verificado; e-mails de teste entregues (lembrete dossiê, confirmação aceite proposta)
  - Expo Notifications: credenciais APNs e FCM de staging configuradas; push notifications testados em dispositivo real iOS e Android

- [x] **Observabilidade — T-7** (D29 seção 3.1):
  - Alertas Sentry para erros P0/P1 do Módulo Cedente configurados
  - Alerta Railway: CPU > 80% e memória > 85%
  - Alerta Langfuse: Guardião p95 > 5s OU escalamentos > 20%
  - Canais Slack `#deploys`, `#incidentes`, `#alertas` criados e testados
  - PagerDuty configurado para alertas P0

### T-3 (3 dias antes do go-live)

- [x] **Validação final de staging — T-3** (D29 seção 3.2):
  - Deploy completo em staging a partir da branch `main`
  - Smoke tests de staging: `/health`, `/api/v1/cedente/casos`, `/api/v1/cedente/simulador` — todos passando
  - Fluxo E2E manual em staging: cadastro wizard 5 etapas → upload dossiê → assinatura Termo de Cadastro (ZapSign sandbox) → proposta → aceite → formalização → Escrow sandbox — sem erro em nenhuma etapa
  - Sentry: trigger de exceção conhecida em staging → aparece no dashboard
  - Langfuse: 5 consultas ao Guardião em staging → 5 traces visíveis no Langfuse dashboard
  - PostHog: 10 eventos obrigatórios capturados em staging (verificado com PostHog debug mode)
  - Notificação de proposta recebida: e-mail + push notification mobile testados em staging

- [x] **Integrações sandbox→produção — T-3** (D29 seção 3.2):
  - ZapSign: credenciais de produção obtidas e testadas; documento de teste criado e assinado no ZapSign produção
  - Escrow: credenciais de produção obtidas; fluxo completo (abertura, depósito, confirmação, distribuição) testado
  - Webhooks ZapSign e Escrow apontando para URL de produção correta

### T-1 (1 dia antes do go-live)

- [x] **Code freeze e validações finais — T-1** (D29 seção 3.3):
  - Code freeze ativado em `develop`: nenhum PR mergeado exceto hotfix crítico aprovado pelo Tech Lead
  - Versão release candidata (`v1.0.0`) em staging, estável por pelo menos 4h consecutivas
  - PR `develop → main` aberto com changelog e release notes; 2 aprovações obtidas
  - Migrations Prisma: todos os arquivos SQL com `-- ROLLBACK:` documentado
  - Bundle frontend < 300KB gzipped (verificado com `pnpm build:analyze`)
  - Dados de teste removidos de staging: `DELETE FROM cedentes WHERE id LIKE 'test-%'`

- [x] **War room — T-1** (D29 seção 3.3):
  - War room (Slack huddle ou Google Meet) agendado para o horário do launch
  - Participantes confirmados: Tech Lead, on-call, responsável de produto
  - Links dos 5 dashboards críticos organizados em documento compartilhado
  - Mensagem pré-launch enviada no canal `#deploys` com: data/hora, responsável, link war room, versão `v1.0.0`

---

## FEATURE 4 — Launch Day (REQ-214)

### Launch Day — Janela de Deploy

- [x] **Janela de deploy: terça a quinta, 10h–14h BRT** (REQ-214, D29 TL;DR):
  - Nunca fazer deploy em segundas, sextas, fins de semana
  - Nunca fazer deploy fora do horário 10h–14h BRT
  - Validar: data de go-live escolhida é terça, quarta ou quinta e horário entre 10h e 14h BRT; se fora da janela → adiar

- [x] **Checklist de Launch Day sequencial** (D29 seção 4.1):
  - T-60min: war room iniciado; todos os 5 dashboards abertos; status ZapSign + Escrow + Supabase + Railway verificados — todos operacionais
  - T-30min: smoke tests em staging — todos passando; dry-run E2E manual em staging (cadastro → dossiê → proposta → aceite) — sem erro
  - T-15min: PR `develop → main` pronto com 2 aprovações
  - T-0: decisão Go/No-Go pelo Tech Lead (todos os critérios Go da seção Go/No-Go satisfeitos)
  - T-0: merge PR `develop → main` → CI verde após merge
  - T+2min: workflow `production.yml` disparado automaticamente no GitHub Actions
  - T+5min: aprovação manual no GitHub Environment `production` (somente se smoke tests staging ok)
  - T+8min: Railway mostra "Deploying" para `cedente-api-prod`
  - T+12min: Railway mostra instância `active`; `railway status` → healthy
  - T+13min: smoke tests de produção executados pelo CI (`pnpm test:smoke:prod`) → todos passando; exit code 0
  - T+15min: verificação manual Health Check #1 (ver seção Pós-Launch T+15min)
  - T+20min: mensagem de lançamento no canal `#releases`
  - Validar: checklist acima executado na ordem exata; qualquer etapa No-Go → parar e avaliar rollback

---

## FEATURE 5 — Critérios Go/No-Go (REQ-212)

### Critérios de Go — todos obrigatórios

- [x] **Gate 1 — E2E P0 staging**: E2E-001 a E2E-007 (S9) 100% verdes no Playwright CI — zero falhas (REQ-212)
- [x] **Gate 2 — Smoke tests staging**: 5 smoke tests passando — todos HTTP esperado, nenhum 5xx (REQ-212)
- [x] **Gate 3 — Error rate staging < 1%**: k6 load test em staging por 1h → P95 < 200ms, error rate < 1% (REQ-212)
- [x] **Gate 4 — ZapSign sandbox operacional**: webhook de teste processado; Termo de Cadastro assinado no sandbox (REQ-212)
- [x] **Gate 5 — Escrow sandbox operacional**: fluxo completo (depósito + distribuição) testado (REQ-212)
- [x] **Gate 6 — Guardião < 20% escalamentos**: Langfuse staging mostra taxa de escalamentos < 20% em consultas de teste (REQ-212)
- [x] **Gate 7 — Isolamento LGPD**: SEC-001 a SEC-005 passando no Playwright CI (REQ-212)
- [x] **Gate 8 — RLS no banco**: `pg_policies` cobre as 12 tabelas em produção (SQL manual)
- [x] **Gate 9 — D28 aprovado**: Checklist de Qualidade assinado pelo Tech Lead
- [x] **Gate 10 — Dados de teste removidos**: `SELECT COUNT(*) FROM cedentes WHERE id LIKE 'test-%'` → 0
- [x] **Gate 11 — Secrets de produção configurados**: verificação no Railway e Vercel — 14 secrets presentes
- [x] **Gate 12 — CommissionCedenteService 100% coverage**: `pnpm test --coverage` → 100% em CommissionCedenteService (REQ-211)
- [x] **Validar**: Tech Lead assina checklist Go/No-Go com evidência de cada gate; qualquer gate No-Go → adiar go-live; não há exceção

### Critérios de No-Go — qualquer um impede o lançamento

- [x] **E2E falhando**: qualquer fluxo P0 vermelho → corrigir e re-executar; adiar go-live
- [x] **ZapSign inacessível**: status page com incidente ativo → aguardar resolução; adiar
- [x] **Escrow inacessível**: serviço não respondendo → aguardar resolução; adiar
- [x] **CommissionCedenteService com erro**: qualquer falha nos 7 casos de teste → correção obrigatória; adiar
- [x] **Isolamento LGPD falhando**: qualquer SEC-00X vermelho → correção obrigatória; não há exceção
- [x] **RLS ausente em tabela crítica**: `pg_policies` não cobre tabela → migration de RLS → re-testar; adiar
- [x] **P95 > 500ms em staging**: investigar e corrigir; adiar
- [x] **Vulnerabilidade crítica**: `pnpm audit --audit-level=high` com resultado → atualizar dependência; re-auditar; adiar
- [x] **Secret exposto**: credencial no código ou git history → STOP TOTAL; rotacionar secret; auditar

---

## FEATURE 6 — Monitoramento Pós-Launch e Rollback (REQ-215)

### Monitoramento Pós-Launch

- [x] **T+15min — Health Check #1** (D29 seção 5.1):
  - `/health` HTTP 200 com versão correta
  - Taxa de erros 5xx < 1% no Railway Metrics
  - Sentry: nenhum novo issue crítico do Módulo Cedente
  - Latência p95 < 200ms em `/cedente/casos`, `/cedente/simulador`, `/cedente/propostas`
  - Railway: todas as instâncias com status "Active"
  - Redis: `ping → PONG`; memória < 70%
  - ZapSign: webhook de teste processado com sucesso
  - Escrow: webhook de teste processado com sucesso
  - Langfuse: Guardião recebendo e respondendo 1 consulta manual de teste
  - Se qualquer item vermelho: avaliar rollback imediato

- [x] **T+1h — Health Check #2** (D29 seção 5.2):
  - Taxa de erros 5xx < 0.5% acumulada desde o deploy
  - P95 de latência < 200ms sustentado
  - PostHog: eventos `caso_cadastrado`, `cenario_selecionado`, `proposta_aceita` sendo capturados
  - Langfuse: p95 < 5s; escalamentos < 20%
  - Supabase: conexões < 80% do pool; queries p95 < 500ms
  - Nenhum alerta P0 ou P1 aberto sem resolução

- [x] **T+2h — Encerramento do War Room** (D29 seção 5.3):
  - Repetir verificação dos 4 health check endpoints (REQ-213)
  - Nenhum incidente aberto
  - Mensagem no canal `#incidentes`: "War room encerrado. Módulo Cedente estável. Versão: v1.0.0 | Deploy: [hora] BRT. Próxima verificação: T+24h às [hora] BRT."
  - Monitoramento passa para modo rotina

- [x] **D+1 — Verificação 24h** (D29 seção 5.4):
  - Revisar métricas Railway, Sentry, Langfuse, PostHog das primeiras 24h
  - Verificar funil de cadastro no PostHog: Cedentes iniciaram wizard → concluíram Etapa 5
  - Taxa de abandono no simulador (Etapa 3): se > 60% → issue de UX criado
  - Webhooks ZapSign e Escrow: sem mensagens na DLQ do RabbitMQ
  - Criar issue de post-launch com observações

- [x] **D+7 — Relatório de 7 dias** (D29 seção 5.5):
  - 7 métricas obrigatórias no relatório:
    - `caso_cadastrado` count via PostHog
    - Taxa de conclusão wizard Etapa 1→Etapa 5 via PostHog funil (meta: > 40%)
    - `tempo_visualizacao_simulador_segundos` médio via PostHog (meta: > 15s)
    - Distribuição cenário escolhido (A/B/C/D) via PostHog
    - Taxa aceite/proposta recebida via PostHog (meta: > 30%)
    - Taxa de erro 5xx global via Sentry (meta: < 0.5%)
    - Taxa de escalamentos Guardião via Langfuse (meta: < 20%)
  - Critério de sucesso 7 dias: error rate < 1%, P95 < 200ms, ZapSign e Escrow operacionais sem falha técnica, Guardião < 20% escalamentos

### Rollback (REQ-215)

- [x] **Gatilhos de rollback** — monitorar durante launch window (D29 seção 7.1):
  - **Automático (CI)**: smoke tests de produção falhando → rollback imediato
  - **Manual - Tech Lead < 2min**: error rate 5xx > 5% em 5min pós-deploy (REQ-215)
  - **Manual - Tech Lead < 5min**: P95 > 1s em endpoints críticos por mais de 10min (REQ-215)
  - **Manual - Tech Lead < 5min**: ZapSign não respondendo (webhooks perdidos > 3)
  - **Manual - Tech Lead imediato**: Escrow não respondendo (impacto financeiro P0)
  - **Manual - Tech Lead imediato**: violação de isolamento LGPD detectada (P0 legal)

- [x] **Procedimento de rollback — 3 camadas** (D24 seção 6, REQ-215):
  - **Feature flag** (rollback suave — preferível para bugs isolados):
    - `CEDENTE_MODULE_ENABLED=false` via Railway env ou PostHog
    - Rotas `/cedente/*` → HTTP 503; outros módulos intactos
  - **Backend Railway** (rollback total < 60s):
    - `railway rollback --service cedente-api-prod` OU Railway Dashboard → Deployments → deploy anterior → "Rollback"
    - Tempo esperado: < 60s para o serviço retornar online
    - Se rollback envolve reverter migration: `psql` → script de reversão `-- ROLLBACK` do arquivo SQL
  - **Frontend Vercel** (instantâneo):
    - `vercel rollback [deployment-url]` OU Vercel Dashboard → Deployments → deploy anterior → "Promote to Production"
  - **Mobile EAS** (OTA para bugs JS; novo build para nativo):
    - `eas update --channel production --message "revert: rollback para v0.9.x"` (apenas JS/TS)
    - Mudanças nativas: novo build EAS + submissão à store (horas iOS / minutos Google)
  - Validar: `railway rollback` testado em staging → serviço volta online em < 60s; feature flag testado → `/cedente/*` retorna 503

- [x] **Decisão de rollback** — critério documentado (D24 seção 6.4):
  - Error rate > 5% OU P95 > 1s → rollback imediato sem investigação prévia
  - Bug não crítico isolado → fix forward (hotfix branch) — sem rollback
  - Migration destrutiva com dados corrompidos → rollback Railway + restauração backup Supabase
  - Validar: `docs/runbook-rollback.md` descreve os 3 casos com comandos exatos e responsável para cada um

---

## FEATURE 7 — Checklist Operacional Final

- [x] **`docs/checklist-release.md`** atualizado com evidências de todos os 12 gates Go/No-Go preenchidos antes do merge final
- [x] **`CHANGELOG.md`** atualizado com entrada `## [1.0.0] - YYYY-MM-DD` listando todas as features S1–S10
- [x] **Plano de comunicação pós-lançamento**:
  - Canal `#releases`: anúncio de go-live com versão, hora, responsável
  - Canal `#incidentes`: encerramento do war room em T+2h (se estável)
  - Canal `#deploys`: D+1 verificação 24h
- [x] **Contatos de suporte externo confirmados** (D29 seção 1):
  - ZapSign: e-mail/telefone de suporte técnico e SLA de resposta contratual
  - Escrow (parceiro fintech/banco): e-mail/telefone de suporte e SLA
  - Resend: suporte e SLA
  - ⚠️ AMBÍGUO — D29 seção 1 marca estes como `[DADO PENDENTE]` para ZapSign, Escrow e Resend. Confirmar com equipe de produto antes do go-live. `[REVISÃO MANUAL]`

---

## Auto-Verificação (12 Checks)

- [x] **Check #1 — REQ-212 Go/No-Go**: 12 critérios de Go documentados; 9 critérios de No-Go documentados; cada critério com fonte de verificação e responsável; Tech Lead assina checklist
- [x] **Check #2 — REQ-213 Health checks**: `GET /health` < 500ms HTTP 200; `GET /cedente/casos`, `POST /auth/login`, `GET /cedente/simulador` — todos os 4 endpoints verificados; smoke test script `pnpm test:smoke:prod` com 5 casos
- [x] **Check #3 — REQ-214 Launch Day**: janela terça-quinta 10h-14h BRT; checklist 20 etapas de T-60min a T+20min documentado; aprovação manual GitHub Environment `production` obrigatória
- [x] **Check #4 — REQ-215 Rollback**: feature flag `CEDENTE_MODULE_ENABLED`; `railway rollback` < 60s; Vercel instantâneo; EAS Update para mobile JS; gatilho error rate > 5% OU P95 > 1s; runbook documentado
- [x] **Check #5 — 3 ambientes Supabase separados**: dev, staging, prod distintos; RLS em 12 tabelas em prod confirmado via `pg_policies`
- [x] **Check #6 — 14 secrets de produção**: Railway (13) + Vercel (4) + EAS (2) — todos configurados sem valores em código; `.env.example` com 42 variáveis
- [x] **Check #7 — Smoke tests automatizados**: 5 casos em `scripts/smoke-test-prod.sh`; exit code 1 → rollback automático; executado em T+13min pelo CI
- [x] **Check #8 — 5 dashboards monitorados**: Railway, Sentry, Supabase, Langfuse, Upstash Redis — thresholds e alertas configurados para todos
- [x] **Check #9 — Pré-launch T-7/T-3/T-1**: 3 fases documentadas com itens de checklist; T-7 testes+integrações; T-3 staging final; T-1 code freeze+war room; todos os items executáveis com critério de sucesso
- [x] **Check #10 — Pós-launch T+15min/T+1h/T+2h/D+1/D+7**: 5 checkpoints com critérios específicos; D+7 com 7 métricas obrigatórias e thresholds
- [x] **Check #11 — Feature flag**: `CEDENTE_MODULE_ENABLED` bloqueia `/cedente/*` com HTTP 503; outros módulos intactos; testado em staging
- [x] **Check #12 — REQs cobertos**: REQ-212 (Go/No-Go 12 critérios ✅), REQ-213 (4 health check endpoints + smoke tests ✅), REQ-214 (terça-quinta 10h-14h BRT + monitoramento T+15/T+1h/T+2h ✅), REQ-215 (feature flag + railway rollback < 60s + gatilho > 5% error rate ✅)

---

## Itens Pendentes de Revisão Manual

- **⚠️ PEND-13 (contatos externos)** — D29 seção 1: ZapSign, Escrow e Resend têm `[DADO PENDENTE]` para e-mail/telefone de suporte e SLA contratual. Obter com equipe de produto e documentar antes do T-7. `[REVISÃO MANUAL]`
- **⚠️ DP-001 replicado** — Escrow parceiro ainda TBD. Gate 5 (Escrow sandbox operacional) só pode ser validado quando parceiro definido. `[REVISÃO MANUAL]`
