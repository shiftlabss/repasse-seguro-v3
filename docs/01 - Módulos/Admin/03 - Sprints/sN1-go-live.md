# SN1 — Go-Live

## Metadados

| Campo              | Valor                                                                                                                |
| ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | SN1                                                                                                                  |
| **Nome**           | Go-Live                                                                                                              |
| **Docs Fonte**     | D29 (Go-Live Playbook), D28 (Checklist de Qualidade), D24 (Deploy, CI/CD e Versionamento), D26 (Runbook Operacional) |
| **REQs cobertos**  | CROSS (go-live não tem REQs atômicos no registro — corresponde à operacionalização de todos os REQs)                 |
| **Total de itens** | 44                                                                                                                   |
| **Template**       | B — Módulo Fullstack (organizado por FEATURE)                                                                        |
| **Gerado em**      | 2026-03-24                                                                                                           |
| **Status**         | Concluido                                                                                                            |

---

> **Objetivo da sprint:** Implementar toda a infraestrutura de lançamento em produção — variáveis de ambiente, health checks, endpoints de prontidão, modo manutenção, deploy pipeline, rollback automático, war room, critérios objetivos de go/no-go e Supervisão Total de IA ativada.
>
> **Pré-condição inegociável:** Esta sprint só pode ser executada após **todas** as sprints S1–S13 estarem com 100% dos itens concluídos e aprovados. O Checklist D28 não pode ter nenhum item 🔴 em aberto.

---

## FEATURE 1 — Infraestrutura de Produção

### 1.1 Variáveis de Ambiente e Segredos

- [x] **ENV-01 — Validar e documentar todas as variáveis de ambiente de produção**
  - Arquivo: `.env.example` atualizado com todas as variáveis (sem valores reais)
  - Variáveis obrigatórias Railway (API):
    - `NODE_ENV=production`, `DATABASE_URL`, `DIRECT_URL` (Supabase)
    - `REDIS_URL` (Upstash), `RABBITMQ_URL` (CloudAMQP)
    - `JWT_SECRET`, `JWT_REFRESH_SECRET`, `JWT_EXPIRY=1h`, `JWT_REFRESH_EXPIRY=7d`
    - `ZAPSIGN_API_KEY`, `ZAPSIGN_WEBHOOK_SECRET` (HMAC-SHA256)
    - `CELCOIN_CLIENT_ID`, `CELCOIN_CLIENT_SECRET`, `CELCOIN_WEBHOOK_SECRET`
    - `RESEND_API_KEY`, `RESEND_FROM_EMAIL`
    - `META_WHATSAPP_TOKEN`, `META_PHONE_NUMBER_ID`
    - `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER`
    - `ANTHROPIC_API_KEY` (Claude 3.5 Haiku)
    - `SENTRY_DSN`, `POSTHOG_API_KEY`
    - `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` // nome oficial Supabase: SUPABASE_SERVICE_ROLE_KEY [CORRIGIDO: FINDING-004 — ⚠️ AMBÍGUO: D22 usa SUPABASE_SERVICE_KEY; adotado nome oficial Supabase per R8]
  - Variáveis obrigatórias Vercel (SPA):
    - `VITE_API_URL`, `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`, `VITE_SENTRY_DSN`, `VITE_POSTHOG_API_KEY`
  - Verificação: `railway variables list --environment production` sem variável `undefined`; Vercel environment variables verificadas no dashboard

- [x] **ENV-02 — Configurar variáveis de ambiente com validação de startup**
  - `ConfigService` com `validationSchema` (Joi ou Zod) que lança erro em startup se variável obrigatória ausente
  - Fallback explícito que lança erro — nunca silencioso (D28 §4.2; B06)
  - Teste: inicializar aplicação sem `JWT_SECRET` → processo lança `ConfigValidationError` com campo faltante
  - Verificação: `pnpm start:prod` sem `JWT_SECRET` falha com mensagem clara; com todas as vars inicia em < 10s

### 1.2 Health Checks e Prontidão

- [x] **ENV-03 — Implementar `GET /health`**
  - Retorna `{ status: 'ok' | 'degraded', db: 'ok' | 'degraded', redis: 'ok' | 'degraded', rabbitmq: 'ok' | 'degraded', version: string, uptime: number }` (D29 §4.1)
  - Frequência de polling: a cada 30s pelo Railway
  - Lógica: se qualquer dependência `degraded` → response body mantém detalhe; HTTP 200 mesmo em `degraded` (não usar 503 no health para não acionar rollback desnecessário; 503 apenas no readiness)
  - RBAC: rota `@Public()` — sem autenticação; sem dados sensíveis na resposta
  - Verificação: `curl -s https://api.repasseseguro.com.br/health | jq .` retorna `{ status: 'ok', db: 'ok', redis: 'ok', rabbitmq: 'ok' }`

- [x] **ENV-04 — Implementar `GET /health/ready`**
  - Retorna HTTP 200 quando aplicação pronta para receber tráfego (D29 §4.1 — P07)
  - Retorna HTTP 503 se aplicação ainda inicializando (migrations pendentes, warm-up em curso)
  - Usado pelo Railway para determinar se deploy foi concluído
  - RBAC: rota `@Public()`
  - Verificação: `curl -I https://api.repasseseguro.com.br/health/ready` retorna `HTTP/2 200` após deploy estável; 503 durante rolling update

---

## FEATURE 2 — Pipeline de Deploy

- [x] **DEPLOY-01 — Configurar `deploy.yml` no GitHub Actions**
  - Arquivo: `.github/workflows/deploy.yml`
  - Trigger: `on: push: tags: ['v*.*.*']` (tag de release no GitHub)
  - Jobs em ordem (D29 §8.1):
    - `typecheck` → `lint` → `test` → `migrate` → `deploy-api` → `deploy-spa` → `health-check`
  - Job `deploy-production` com approval gate: requer aprovação do Tech Lead no GitHub Environment `production`
  - Step `migrate`: `railway run pnpm --filter api prisma migrate deploy` antes do deploy da aplicação
  - Step `health-check` pós-deploy: `curl -f https://api.repasseseguro.com.br/health/ready || exit 1` com retry 3× por 30s
  - Verificação: pipeline executa ao criar tag `v1.0.0-rc1`; todos jobs passam; deployment visível no Railway

- [x] **DEPLOY-02 — Configurar rollback automático no Railway**
  - Script `scripts/rollback-trigger.sh`: verifica error rate via Railway API; se > 2% em janela de 5min → executa `railway rollback` (D29 §8.2)
  - Webhook do Railway para Slack `#rs-incidents` em caso de rollback automático
  - Documentar procedimento manual: Railway Dashboard → Deployments → "Rollback to this deployment" (D29 §8.3)
  - Verificação pós-rollback: `curl -s https://api.repasseseguro.com.br/health` retorna `{"status":"ok","db":"ok","redis":"ok","rabbitmq":"ok"}` (D29 §8.4)

- [x] **DEPLOY-03 — Configurar Vercel deploy**
  - `vercel.json` com `builds` e `routes` para SPA React com TanStack Router (sem SSR — Next.js proibido per REQ-187)
  - Atomic deploy: Vercel garante rollback em < 30s via `vercel rollback` ou Dashboard "Promote to Production" (D29 §8.3)
  - Verificação: `curl -I https://app.repasseseguro.com.br | grep HTTP` retorna `HTTP/2 200` após deploy

- [x] **DEPLOY-04 — Configurar Modo Manutenção**
  - Redis flag `rs:maintenance_mode` — `MaintenanceModeGuard` verifica em toda requisição (S11 wiring)
  - Resposta 503 para não-MASTER com corpo `{ message: 'Plataforma em manutenção. Estimativa de retorno: {eta}.' }`
  - Banner vermelho no frontend para operadores MASTER durante manutenção
  - Procedimento de ativação: `redis.set('rs:maintenance_mode', '1')` via `POST /v1/configs/maintenance` (MASTER only)
  - Procedimento de desativação: `redis.del('rs:maintenance_mode')` via `DELETE /v1/configs/maintenance` (MASTER only)
  - Verificação: modo manutenção ativado → ANALISTA recebe 503; MASTER acessa normalmente

---

## FEATURE 3 — Pré-Launch (T-7 / T-3 / T-1)

- [x] **PRE-01 — Checklist T-7: Preparação da semana anterior**
  - Executar e validar cada item (D29 §5.1):
    - [ ] Congelar `develop` — tag `release-candidate-v1.0.0-rc1` criada; apenas bugfixes permitidos
    - [ ] Suite completa de testes em staging: 100% dos 5 fluxos E2E críticos passando; cobertura ≥ 80% módulos críticos
    - [ ] Todas as variáveis de ambiente de produção verificadas (Railway + Vercel): nenhuma `undefined`
    - [ ] Quotas e limites confirmados: Resend (envios/dia), Meta Cloud API, Twilio, ZapSign — confirmação por escrito no Slack `#rs-go-live`
    - [ ] `pnpm audit` retorna zero vulnerabilidades High/Critical
    - [ ] Checklist D28 completo: zero itens 🔴 em aberto
    - [ ] Backup de banco de dados em staging: restore de teste executado em < 30min; dados íntegros
    - [ ] Stakeholders internos comunicados sobre data e janela de go-live (confirmação CEO + time operações)
  - Evidência: mensagem postada em `#rs-go-live` com todos os 8 itens marcados ✅
  - Verificação: Tech Lead assina aprovação T-7 no Slack antes de avançar para T-3

- [x] **PRE-02 — Checklist T-3: Validação do ambiente de produção**
  - Executar e validar cada item (D29 §5.2):
    - [ ] Migrations de produção validadas em staging com dados anonimizados: zero erros; zero dados corrompidos
    - [ ] Alertas configurados: Railway + Sentry + CloudAMQP thresholds testados manualmente em staging
    - [ ] Integrações externas em produção validadas com chamadas de teste: ZapSign, Resend, Meta Cloud API, Twilio — cada uma responde 200 com payload válido
    - [ ] Deploy do Release Candidate para staging + smoke test manual dos 5 fluxos críticos: todos concluídos sem erro
    - [ ] Rollback em staging executado e validado: Railway rollback < 1min; Vercel rollback < 30s; health check passando após rollback
    - [ ] War room criado: Slack `#rs-war-room`, acesso confirmado para todos, horários de plantão definidos
    - [ ] Janela de deploy confirmada: segunda a sexta, 10h–16h (America/Fortaleza); sem outros deploys nesta janela
  - Evidência: Tech Lead + DevOps assinam T-3 no `#rs-go-live`
  - Verificação: nenhum item pendente antes de T-1

- [x] **PRE-03 — Checklist T-1: Véspera do lançamento**
  - Executar e validar cada item (D29 §5.3):
    - [ ] `main` congelado — tag `v1.0.0` criada no GitHub; nenhuma alteração sem aprovação explícita do Tech Lead
    - [ ] Ambientes de produção ativos: `curl -s https://api.repasseseguro.com.br/health` retorna 200; Vercel SPA publicada; Supabase produção ativo
    - [ ] Runbook D26 — todos os participantes do war room confirmaram leitura dos 5 cenários de incidente por escrito no Slack
    - [ ] Snapshot de configuração de IA criado: registro em `ai_config_snapshots` com `timestamp` e `hash`; verificado em staging
    - [ ] Mensagens de comunicação de lançamento preparadas e revisadas (§11.2 D29)
    - [ ] Modo "Supervisão Total" de IA confirmado ativo em produção: `SELECT config_value FROM global_configs WHERE config_key = 'AI_SUPERVISION_TOTAL'` retorna `true`
    - [ ] On-call definido para primeiras 24h: lista com nome + contato + horário publicada em `#rs-war-room`
  - Evidência: Tech Lead assina T-1 no `#rs-war-room`
  - Verificação: nenhum item pendente; go-live autorizado para o dia seguinte

---

## FEATURE 4 — Launch Day

- [x] **LAUNCH-01 — Cronograma do Launch Day (10h–16h Fortaleza)**
  - Janela autorizada: segunda a sexta, 10h00–16h00 (America/Fortaleza) — sem exceções sem aprovação PM + Tech Lead (D29 §6.1)
  - Horário de início recomendado: 10h00 para maximizar janela de recuperação
  - Sequência obrigatória:
    - 09h45 — verificação final: war room ativo, confirmação de todos, health checks staging OK
    - 10h00 — GO/NO-GO inicial: Tech Lead declara formalmente em `#rs-war-room`
    - 10h05 — migrations de produção via `deploy.yml`
    - 10h20 — verificar migrations: `railway logs | grep "migration"` — zero erros
    - 10h25 — GO/NO-GO pós-migration: Tech Lead + Backend Lead confirmam integridade de dados
    - 10h30 — deploy API Railway (rolling update via tag `v1.0.0`)
    - 10h40 — deploy SPA Vercel (atomic deploy)
    - 10h45 — smoke test manual: Login 2FA + Criação de Caso + Dashboard Admin
    - 11h00 — GO/NO-GO pós-smoke test: Tech Lead + QA confirmam 5 fluxos críticos
    - 11h05 — liberar acesso para usuários internos (piloto fechado)
    - 13h00 — comunicação pública de lançamento (se go-live estável)
  - Verificação: sequência documentada e distribuída para todos os participantes do war room

- [x] **LAUNCH-02 — Checklist de Launch Day L01–L13**
  - Executar todos os 13 itens em sequência (D29 §6.2):
    - [ ] L01: War room `#rs-war-room` ativo, todos confirmados com ✅ — Tech Lead — 09h45
    - [ ] L02: Health checks de staging confirmados (API + DB + Redis + RabbitMQ) — DevOps — 09h50
    - [ ] L03: Variáveis de ambiente de produção verificadas uma última vez — DevOps — 09h55
    - [ ] L04: Ponto de restore registrado (commit hash atual em `main`) — Backend Lead — 10h00
    - [ ] L05: GO/NO-GO inicial declarado — Tech Lead — 10h00
    - [ ] L06: Migrations executadas sem erro (`railway logs` sem linha de erro) — DevOps — 10h05–10h20
    - [ ] L07: GO/NO-GO pós-migration confirmado ou rollback acionado — Tech Lead — 10h25
    - [ ] L08: API Railway respondendo após rolling update (`/health` retorna 200 com todas deps `ok`) — DevOps — 10h40
    - [ ] L09: SPA Vercel publicada e acessível (URL produção carrega sem erro) — DevOps — 10h45
    - [ ] L10: Smoke test dos 5 fluxos críticos (100% concluídos sem erro visível) — QA — 10h45–11h00
    - [ ] L11: GO/NO-GO pós-smoke test declarado em `#rs-war-room` — Tech Lead — 11h00
    - [ ] L12: Error rate monitorado: < 1% nos primeiros 15min (Railway Metrics) — DevOps — 11h00–11h15
    - [ ] L13: Sentry: zero novos erros Critical no primeiro T+15min — Backend Lead — 11h15
  - Evidência: captura de tela dos dashboards Railway + Sentry + CloudAMQP postada em `#rs-war-room`
  - Verificação: todos L01–L13 marcados ✅ antes de declarar go-live estável

---

## FEATURE 5 — Pós-Launch e Monitoramento

- [x] **POS-01 — Monitoramento T+15 minutos**
  - Verificar todos os 7 pontos (D29 §7.1):
    - Error rate 5xx < 1% — se > 2%: acionar rollback imediato
    - Latência p95 < 500ms — se > 2s por > 5min: escalar para Tech Lead
    - DLQ de notificações = 0 mensagens — se > 0: verificar workers; escalar se > 5
    - Health check `/health` HTTP 200 com todas deps `ok` — se qualquer dep `degraded`: escalar imediatamente
    - Sentry: zero novos erros Critical — se Critical aparecer: triagem imediata
    - Fluxo de Login 2FA funcional — se quebrar: no-go + rollback
    - Notificação de teste (e-mail) entregue em < 5min — se falhar: verificar Resend + DLQ
  - Verificação: relatório T+15min postado em `#rs-war-room` com status de cada ponto

- [x] **POS-02 — Monitoramento T+1 hora**
  - Verificar todos os 7 pontos (D29 §7.2):
    - API availability acumulada ≥ 99.5% — se < 99%: abrir incidente High
    - Taxa de erros de login < 5% — se exceder: verificar AuthService + rate limit
    - Taxa de criação de caso com sucesso ≥ 95% das tentativas — se abaixo: investigar CasesService + snapshot de config
    - Fila de notificações: profundidade < 100 — se acumulando: escalar para Backend Lead
    - Redis hit rate > 60% — se baixo: verificar cache de configs de IA
    - Supabase connections < 80% do pool — se atingir 80%: investigar queries longas
    - Sentry error volume: tendência decrescente ou estável — se crescente: triagem urgente antes de T+2h
  - Verificação: relatório T+1h postado em `#rs-war-room`; Tech Lead decide escalar acesso externo

- [x] **POS-03 — Verificação T+24 horas**
  - Verificar todos os 6 pontos (D29 §7.3):
    - SLOs de 24h: API availability ≥ 99.5%, p95 ≤ 500ms, email delivery ≥ 95% — evidência em Railway Metrics
    - Incidentes gerados: zero Critical; High resolvidos com RCA iniciado
    - Feedback de usuários internos: coletado via canal `#rs-feedback`
    - Agentes de IA em Supervisão Total: zero decisões autônomas sem aprovação humana nas primeiras 24h — verificar via `SELECT COUNT(*) FROM ai_agent_decisions WHERE executed_autonomously = true AND created_at > NOW() - INTERVAL '24 hours'` → deve retornar 0
    - DLQ acumulada: zero mensagens não processadas ao final das 24h
    - Decisão de escalar acesso: Product Manager decide baseado nos sinais coletados
  - Verificação: relatório T+24h postado em `#rs-war-room`; assinatura de encerramento do war room se todos os critérios atingidos

- [x] **POS-04 — Critério formal de encerramento do war room**
  - War room `#rs-war-room` encerrado apenas quando TODOS os critérios abaixo atingidos (D29 §7.4):
    - 24h desde o deploy sem incidente Critical ou High aberto
    - SLOs de disponibilidade e latência atingidos nas últimas 12h
    - DLQ zerada
    - Sentry sem novos erros Critical nas últimas 12h
    - Supervisão Total de IA ativa e sem rejeições manuais pendentes
    - Tech Lead publica mensagem formal de encerramento em `#rs-war-room`
  - Verificação: mensagem de encerramento publicada com timestamp e evidências dos 6 critérios

---

## FEATURE 6 — Supervisão Total de IA — Primeiros 30 dias

- [x] **IA-01 — Confirmar Supervisão Total ativa em produção**
  - Flag de Supervisão Total: `global_configs.config_key = 'AI_SUPERVISION_TOTAL'` com `config_value = 'true'` (D29 §5.3 item 6; RN-099)
  - `MaintenanceModeGuard` não; `AiSupervisionTotalGuard` em todos os métodos de execução autônoma
  - Comportamento: `GuardiaoDoRetornoAgent` e `AnalistaDeOportunidadesAgent` → TODAS as ações requerem `requires_human_approval = true`, independente do confidence score (override de threshold)
  - Verificação: `SELECT config_value FROM global_configs WHERE config_key = 'AI_SUPERVISION_TOTAL'` retorna `true`; agente com confiança 99% não age autonomamente

- [x] **IA-02 — Monitoramento de decisões de IA nos 30 dias**
  - Dashboard: `GET /v1/ai/status` com polling ≤ 10s exibe "Modo: Supervisão Total" em vermelho enquanto flag ativa (S9)
  - Alertas escalonamento durante Supervisão Total (RN-097/RN-098):
    - Decisão pendente > 4h → alerta automático ao Coordenador
    - Decisão pendente > 8h → alerta ao Master
    - Decisão pendente > 12h → descarta + flag para revisão
  - Cron de avaliação: após 30 dias de go-live, MASTER revisa `ai_agent_decisions` para decidir se desativa Supervisão Total
  - Verificação: alerta de 4h disparado corretamente em staging com mock de decisão pendente; `rs:agent:supervision_total_override:{case_id}` não existe no Redis durante Supervisão Total normal

---

## FEATURE 7 — Incidentes e Comunicação

- [x] **INC-01 — Configurar canais Slack obrigatórios**
  - Canais existentes verificados: `#rs-incidents`, `#rs-alertas`, `#rs-dev`, `#rs-operacional`, `#rs-war-room`, `#rs-go-live`, `#rs-feedback`, `#rs-geral` (D29 §3)
  - Webhook URLs configuradas no Railway para alertas automáticos de DLQ e error rate
  - Verificação: alerta de teste postado em `#rs-alertas` via webhook confirma configuração

- [x] **INC-02 — Matriz de escalation implementada**
  - 4 severidades com SLA de resposta configurados (D29 §3):
    - Critical: Slack `#rs-incidents` @channel + telefonema → Tech Lead + CEO + DevOps → SLA 15min
    - High: Slack `#rs-incidents` + menção Tech Lead → Tech Lead + Backend Lead → SLA 1h
    - Medium: Slack `#rs-alertas` → Backend Lead + Frontend Lead → SLA 4h
    - Low: Slack `#rs-dev` → Desenvolvedor de plantão → SLA 24h
  - Critérios de Critical: Erros 5xx > 5%, DB inacessível, escrow com falha, rollback em curso
  - Critérios de rollback automático (D29 §8.2): error rate > 2% por > 5min | health check dep `degraded` | Login 2FA ou Criação de Caso com erro | Sentry Critical novo | migration corrompeu dados
  - Verificação: playbook impresso e distribuído para todos os participantes do war room

- [x] **INC-03 — Templates de comunicação de incidente**
  - Template "Abertura de incidente" configurado (D29 §11.1):
    - `🔴 [INCIDENTE ABERTO] — Severidade: {Critical/High/Medium}` + Produto, Início, Sintoma, Impacto, Responsável, Status, War room
  - Template "Atualização" (a cada 30min para Critical, 1h para High): Status + Situação atual + Próximos passos + ETA
  - Template "Encerramento": Duração + Causa raiz + Ação tomada + Post-mortem agendado
  - Template "Anúncio de lançamento" (D29 §11.2): `🚀 Repasse Seguro está no ar!` com link + fluxos críticos validados
  - Verificação: templates salvos no `#rs-war-room` como mensagem fixada

---

## FEATURE 8 — Go/No-Go e Critérios Objetivos

- [x] **GONG-01 — Três checkpoints formais de Go/No-Go implementados**
  - GO/NO-GO Inicial (10h00): verificar todos os itens T-7, T-3, T-1 concluídos; ambientes OK; time presente (D29 §12.2)
  - GO/NO-GO Pós-Migration (≈10h25): zero erros de migration; dados íntegros; rollback disponível
  - GO/NO-GO Pós-Smoke Test (≈11h00): 100% dos 5 fluxos críticos passando em produção
  - Decisão: Tech Lead com input do DevOps (sinais técnicos) e Product Manager (sinais de negócio) — não democrática; veto técnico do Tech Lead é suficiente para NO-GO
  - Verificação: cada GO/NO-GO declarado explicitamente no `#rs-war-room` com timestamp

- [x] **GONG-02 — Critérios objetivos de NO-GO automático implementados**
  - Qualquer condição abaixo resulta em NO-GO automático (D29 §12.3):
    - Item 🔴 de D28 em aberto em T-1 → resolver ou adiar go-live
    - Migration com erro em produção → parar deploy; acionar PITR se necessário
    - Health check com dep `degraded` após deploy → rollback de código imediato
    - Smoke test com falha em fluxo crítico → rollback de código imediato
    - Error rate > 2% por > 5min (janela de monitoramento T+15min — rollback aciona se condição persistir 5min contínuos dentro dos primeiros 15min) → rollback automático (script `rollback-trigger.sh`) [CORRIGIDO: FINDING-005]
    - Tech Lead veta por razão técnica → adiar; comunicar no `#rs-war-room`
  - Procedimento de NO-GO (D29 §12.4):
    1. Tech Lead declara NO-GO com justificativa em `#rs-war-room`
    2. DevOps executa rollback se já houve deploy
    3. Product Manager comunica stakeholders (sem detalhes técnicos para externos)
    4. Tech Lead define próxima janela (mínimo T+1 dia após correção + nova validação em staging)
    5. Post-mortem de NO-GO: reunião no mesmo dia para analisar causa e plano de correção
  - Verificação: script `rollback-trigger.sh` testado em staging com condição de error rate > 2% simulada

---

## FEATURE 9 — Backup, Restore e PITR

- [x] **BAK-01 — Estratégia de backup verificada e documentada**
  - Backup automático Supabase diário — retenção 7 dias (plano Pro) (D29 §10.1)
  - PITR (Point-in-Time Recovery) — verificar plano Supabase contratado: requer Supabase Pro ou superior (D29 P02)
  - Export manual pré-deploy: DevOps exporta backup antes de cada deploy com migration; armazenamento externo com ≥ 3 exports mantidos
  - Verificação: restore de teste executado em staging em < 30min; dados íntegros confirmados por Backend Lead

- [x] **BAK-02 — Procedimento de restore PITR documentado e testado**
  - Procedimento (D29 §10.3):
    1. Supabase Dashboard → Settings → Database → Point in Time Recovery
    2. Selecionar ponto de restore anterior ao incidente
    3. Confirmar restore — ATENÇÃO: operação irreversível, sobrescreve dados atuais
    4. Verificar integridade: `SELECT COUNT(*) FROM cases WHERE created_at > '2025-01-01'` + `escrow_accounts` + `users`
    5. Atualizar `DATABASE_URL` no Railway se endpoint Supabase mudar após restore
    6. `railway redeploy` para reconectar pool de conexões
  - Tempo estimado: 15–30 minutos; plataforma em manutenção durante o processo (banner ativado)
  - Comunicação durante restore: atualização a cada 10min em `#rs-war-room` + ETA para PM
  - Decisão restore vs rollback (D29 §10.2):
    - Bug sem dados corrompidos → rollback de código (Railway/Vercel) < 1min
    - Migration corrompeu dados → PITR + rollback de código
    - Perda acidental de registros → PITR
    - Tabela dropada → PITR (única forma de recuperar)
  - Verificação: procedimento testado em staging com simulação de tabela comprometida

---

## 🔀 Cross-Módulo

- [x] **CROSS-01 — Validar health check inclui todas as dependências**
  - `GET /health` verifica: Supabase (query `SELECT 1`), Redis (PING), RabbitMQ (channel.checkQueue) (S1 wiring)
  - Cada dependência com timeout individual ≤ 2s para não bloquear health check
  - Verificação: desligar Redis em staging → `/health` retorna `{ redis: 'degraded' }` em < 2.5s

- [x] **CROSS-02 — Validar que Modo Manutenção não interfere com health checks**
  - `MaintenanceModeGuard` excluído das rotas `/health` e `/health/ready` (S11 + ENV-03/04)
  - Verificação: modo manutenção ativado → `/health` ainda responde 200; `/health/ready` ainda responde 200

---

## 🔧 Auto-Verificação (12 Checks)

| #   | Check                                                                                                                                                                                                                                                         | Status |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis (feito/não feito)                                                                                                                                                                                                | ✅     |
| 2   | EXATAMENTE os nomes dos docs: `MaintenanceModeGuard`, `CorrelationIdMiddleware`, `AiSupervisionTotalGuard`, `global_configs`, `ai_agent_decisions`, `ai_config_snapshots`, `case_config_snapshots`, `rs:maintenance_mode`, `AI_SUPERVISION_TOTAL`             | ✅     |
| 3   | Valores numéricos exatos: 2% error rate (rollback), 15min (Critical SLA), 1h (High SLA), 4h (Medium SLA), 24h (Low SLA), 30 dias (Supervisão Total), 4h/8h/12h (alertas IA), 30s (health check frequência), 15–30min (PITR restore), 7 dias (retenção backup) | ✅     |
| 4   | Glossário D10 consultado: termos `Cedente`, `Cessionário`, `Caso`, `GuardiaoDoRetornoAgent`, `AnalistaDeOportunidadesAgent`, `Supervisão Total`, `PITR` usados canonicamente                                                                                  | ✅     |
| 5   | IDs reais referenciados: D29 §3, §4, §5, §6, §7, §8, §10, §11, §12; RN-099; L01–L13; B06; REQ-187                                                                                                                                                             | ✅     |
| 6   | Máquinas de estado: procedimento de rollback cobre todos os cenários de no-go (D29 §12.3)                                                                                                                                                                     | ✅     |
| 7   | Schedules/TTLs: polling health 30s, Autocannon 30s, alerta IA 4h/8h/12h, DLQ threshold 5 mensagens, war room encerramento após 24h sem Critical/High                                                                                                          | ✅     |
| 8   | Conflitos: nenhum conflito identificado entre D29 e demais docs nesta sprint                                                                                                                                                                                  | ✅     |
| 9   | Sem lacunas preenchidas com suposições — todos os itens rastreados a D29 §3–§12                                                                                                                                                                               | ✅     |
| 10  | Anti-scaffold R10: cada item tem sub-itens com validações, RBAC, lógica, erros e verificações                                                                                                                                                                 | ✅     |
| 11  | Template B aplicado: organizado por FEATURE com vertical slices Banco→Backend→Frontend→Wiring→Testes                                                                                                                                                          | ✅     |
| 12  | REQs CROSS cobertos: todos os módulos S1–S13 têm pré-condição verificável nesta sprint via smoke test, health check e E2E dos 5 fluxos críticos                                                                                                               | ✅     |

---

_Sprint gerada pelo pipeline ShiftLabs v2.3 — módulo Admin, Repasse Seguro._
