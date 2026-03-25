# 29 - Go-Live Playbook

| Campo | Valor |
|---|---|
| **Nome do Documento** | 29 - Go-Live Playbook |
| **Versão** | v1.0 |
| **Data** | 2026-03-22 (America/Fortaleza) |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Produto** | Repasse Seguro — Módulo Cessionário |

---

> 📌 **TL;DR — Resumo Operacional do Lançamento**
>
> - **Preparação:** 3 janelas — T-7 (testes e integrações), T-3 (staging final), T-1 (war room e checklist).
> - **Go/No-Go:** decisão do Tech Lead com base em: E2E verde, smoke tests passando, integrações externas operacionais, error rate < 2% em staging, SLO de 99.5% atingível.
> - **Launch Day:** terça a quinta, 10h-14h BRT. Deploy automático + aprovação manual no GitHub.
> - **Monitoramento pós-launch:** intensivo por 2 horas (T+15min, T+1h, T+2h). War room encerra após T+2h se estável.
> - **Rollback:** `railway rollback` em < 60s. Gatilho automático: error rate > 2% em 5min. Gatilho manual: qualquer P0 identificado.
> - **Escalation:** Tech Lead (P0: 5min, P1: 15min) → responsável técnico → suporte plataformas externas.
> - **Comunicação:** templates definidos para abertura, atualização (a cada 15min P0, 30min P1) e encerramento.

---

## 1. Matriz de Escalation

| Severidade | Impacto | Canal | Pessoas acionadas | SLA de resposta | Critério de escalação |
|---|---|---|---|---|---|
| **P0 — Crítico** | API down, Escrow falhou, KYC bloqueado, dados expostos | PagerDuty + Slack `#incidentes` | Tech Lead + on-call | 5 minutos | Qualquer falha em fluxo financeiro ou de segurança |
| **P1 — Alto** | Funcionalidade principal degradada, NOT-CES-05/06 não entregue | Slack `#incidentes` | Tech Lead | 15 minutos | P0 não resolvido em 15min → escalar para responsável técnico |
| **P2 — Médio** | Latência elevada sem bloqueio operacional, integração secundária instável | Slack `#alertas` | On-call técnico | 30 minutos | P1 não resolvido em 1h → Tech Lead |
| **P3 — Baixo** | Anomalia não urgente, métricas levemente fora | Slack `#alertas` ou issue | On-call técnico | 2 horas | — |

**Contatos de suporte externo (a confirmar antes do go-live):**

| Serviço | Canal de suporte | SLA de resposta contratual |
|---|---|---|
| Celcoin | [DADO PENDENTE — email/telefone de suporte técnico Celcoin] | [DADO PENDENTE] |
| ZapSign | [DADO PENDENTE — email/telefone de suporte técnico ZapSign] | [DADO PENDENTE] |
| idwall | [DADO PENDENTE — email/telefone de suporte técnico idwall] | [DADO PENDENTE] |
| Supabase | https://supabase.com/support + Discord | Plano Pro: 1 dia útil |
| Railway | https://railway.app/help | Plano Pro: email |
| CloudAMQP | https://customer.cloudamqp.com/support | Depende do plano |

---

## 2. Health Checks

### 2.1 Endpoints de Health (Verificar a cada 5min durante launch window)

| Endpoint | Método | Owner | Threshold de sucesso | Interpretação de falha |
|---|---|---|---|---|
| `https://app.repasse-seguro.com.br/health` | GET | Automático (CI smoke test) | HTTP 200, `{ "status": "ok" }` | API offline ou instância não-saudável → P0 imediato |
| `https://api.repasse-seguro.com.br/api/v1/opportunities` | GET + Bearer token | Tech Lead | HTTP 200, latência < 500ms | Banco ou auth com problema → P1 |
| `https://api.repasse-seguro.com.br/api/v1/auth/login` | POST (payload de teste) | Tech Lead | HTTP 200 ou 401 (sem 5xx) | Auth degradado → P1 |

### 2.2 Dashboards Críticos (Monitorar continuamente durante launch window)

| Dashboard | URL | O que observar | Threshold de alerta |
|---|---|---|---|
| Railway Metrics | https://railway.app/dashboard | CPU, memória, instâncias ativas | CPU > 80%, memória > 85% |
| Sentry | https://sentry.io/{org}/repasse-seguro | Taxa de erros, novos issues | Novos P0 ou taxa > 2% em 5min |
| Supabase Dashboard | https://app.supabase.com | Conexões, latência de queries | Conexões > 80% do pool |
| Langfuse | https://cloud.langfuse.com | Latência LLM, taxa de erro | p95 > 5s ou taxa de erro > 5% |
| CloudAMQP | https://customer.cloudamqp.com | DLQ, consumers, mensagens pendentes | DLQ > 5 mensagens |

### 2.3 Smoke Tests Automatizados (Executar imediatamente após deploy)

```bash
# Executado automaticamente pelo CI após deploy em produção
pnpm test:smoke
# Verifica: /health, /auth/login (400 esperado sem payload), /opportunities (401 esperado sem token)
# Falha: retorna exit code 1 → rollback automático via CI
```

---

## 3. Pré-Launch

### 3.1 T-7 (7 dias antes do go-live)

**Owner: Tech Lead**

**Testes e qualidade:**
- [ ] Suíte E2E completa passando no staging sem falhas intermitentes.
- [ ] Todos os fluxos E2E P0 (E2E-001 a E2E-015 conforme D27) verdes.
- [ ] Cobertura de testes ≥ 80% em módulos críticos; 100% em CommissionService e EscrowService.
- [ ] `pnpm audit --audit-level=high` sem vulnerabilidades críticas.
- [ ] D28 (Checklist de Qualidade) revisado e aprovado pelo Tech Lead.

**Integrações:**
- [ ] ZapSign: webhook URL de produção configurado no painel ZapSign. Teste de webhook com payload real.
- [ ] idwall: credenciais de produção configuradas. Teste de submissão KYC real.
- [ ] Celcoin: credenciais de produção configuradas. Conta Escrow de teste criada e validada.
- [ ] Resend: domínio de produção verificado e emails de teste entregues.
- [ ] Expo Notifications: credenciais APNs e FCM de produção configuradas.

**Infraestrutura:**
- [ ] 3 projetos Supabase criados e configurados: dev, staging, production.
- [ ] RLS habilitado em todas as tabelas de produção. Verificação: `SELECT * FROM pg_policies`.
- [ ] Railway services criados: backend-prod com health check configurado.
- [ ] Vercel project configurado para frontend com environment variables corretas.
- [ ] Upstash Redis prod criado e conectado.
- [ ] CloudAMQP prod criado com filas e DLQs configuradas.

**Secrets:**
- [ ] Todos os secrets de produção adicionados ao Railway (backend) e Vercel (frontend).
- [ ] EAS Secrets configurados para o app mobile.
- [ ] Nenhum secret no código ou `.env` commitado.
- [ ] `.env.example` atualizado com todas as variáveis (sem valores).

---

### 3.2 T-3 (3 dias antes do go-live)

**Owner: Tech Lead**

**Validação de staging (simula produção):**
- [ ] Deploy completo em staging executado a partir da branch `main` (mesma que produção).
- [ ] Smoke tests de staging passando: `/health`, `/opportunities`, `/auth/login`.
- [ ] Fluxo completo E2E executado manualmente em staging: cadastro → KYC → proposta → Escrow → formalização.
- [ ] Sentry configurado e capturando erros no staging (trigger manual de teste).
- [ ] Langfuse recebendo traces do Analista de Oportunidades em staging.
- [ ] PostHog capturando eventos obrigatórios em staging.
- [ ] Notificações NOT-CES-05/06 testadas manualmente em staging.

**Comunicação:**
- [ ] Canais Slack `#deploys`, `#incidentes` e `#alertas` criados e testados.
- [ ] PagerDuty configurado para alertas P0.
- [ ] Lista de contatos de escalation confirmada e atualizada.
- [ ] Janela de deploy aprovada: terça-quinta, 10h-14h BRT (decisão da semana do go-live).

**Observabilidade:**
- [ ] Alertas do D25 configurados no Sentry, Railway e Langfuse.
- [ ] Dashboard de "System Health" operacional (ver D25, seção dashboards).
- [ ] Runbook D26 acessível para todos os membros do time de operação.

---

### 3.3 T-1 (1 dia antes do go-live)

**Owner: Tech Lead + QA**

**Freeze:**
- [ ] Code freeze ativado em `develop`. Nenhum novo PR mergeado exceto hotfix crítico.
- [ ] Versão de release candidata (`v1.0.0`) em staging, estável por pelo menos 4h.

**Verificações finais:**
- [ ] PR `develop → main` aberto com changelog e release notes.
- [ ] 2 aprovações obtidas no PR de release.
- [ ] Migrations Prisma revisadas e retrocompatíveis.
- [ ] Rollback de cada migration documentado no arquivo SQL.
- [ ] Bundle size frontend verificado: < 300KB gzipped.

**War room:**
- [ ] War room (Slack huddle ou Google Meet) agendado para o horário do launch.
- [ ] Participantes confirmados: Tech Lead, on-call, responsável de produto (se aplicável).
- [ ] Runbook D26 aberto e acessível durante o war room.
- [ ] Links de todos os dashboards críticos organizados em documento compartilhado.

**Comunicação pré-launch:**
- [ ] Mensagem pré-launch enviada no canal `#deploys`:
  ```
  🚀 Go-Live agendado para amanhã [data] às [hora] BRT.
  Responsável: @[nome] | War room: [link]
  Monitorar: [link dashboards]
  ```

---

## 4. Launch Day

**Janela de deploy:** terça a quinta, 10h-14h BRT.

[DECISÃO AUTÔNOMA — janela 10h-14h BRT, terça a quinta. Justificativa: evita segundas (ajuste pós-fim de semana), sextas (time reduzido para resposta), manhã cedo (equipe não totalmente disponível) e fim de tarde (pouco tempo para estabilização). Alternativa descartada: qualquer horário — sem janela definida o risco de deploy fora do horário ideal é alto.]

### 4.1 Checklist de Launch Day

| Horário | Atividade | Owner | Critério de sucesso | Se falhar |
|---|---|---|---|---|
| T-60min | War room iniciado. Dashboards abertos. | Tech Lead | Todos os participantes presentes | Reagendar se Tech Lead indisponível |
| T-60min | Verificar status de integrações externas (Celcoin, ZapSign, idwall, Supabase) | Tech Lead | Todas operacionais | No-go se P0 ou P1 em integração crítica |
| T-30min | Executar smoke tests em staging pela última vez | QA | Todos passando | No-go se qualquer smoke test falhar |
| T-15min | Verificar que `release-please` PR está pronto para merge | Tech Lead | PR aprovado com 2 revisores | Obter aprovações restantes |
| T-0 | **GO/NO-GO decision** (ver seção 12) | Tech Lead | Todos os critérios de go satisfeitos | No-go → reagendar |
| T-0 | Merge PR `develop → main` | Tech Lead | CI verde após merge | Investigar falha de CI antes de continuar |
| T+2min | CI disparou workflow `deploy-production.yml` | Automático | Workflow iniciado no GitHub Actions | Verificar trigger manual |
| T+5min | **Aprovação manual no GitHub Environment** `production` | Tech Lead | Workflow aguarda aprovação | Aprovar somente se smoke tests staging ok |
| T+8min | Deploy Rolling iniciado no Railway | Automático | Railway mostra "Deploying" | Verificar Railway Dashboard |
| T+12min | Deploy concluído — Railway mostra instância active | Automático | `railway status` → healthy | Rollback imediato se unhealthy |
| T+13min | Smoke tests de produção executados pelo CI | Automático | Todos passando | Rollback automático se falha |
| T+15min | **Verificação manual — Health Check #1** | Tech Lead | `/health` retorna 200; Sentry sem novos erros | Rollback se error rate > 2% |
| T+20min | Mensagem de lançamento no canal `#releases` | Tech Lead | — | — |

---

## 5. Pós-Launch

### 5.1 T+15 minutos

**Owner: Tech Lead (em war room)**

- [ ] `/health` retornando 200 com versão correta.
- [ ] Taxa de erros 5xx < 1% no Railway Metrics.
- [ ] Sentry: nenhum novo issue crítico.
- [ ] Latência p95 < 500ms.
- [ ] Railway: todas as instâncias com status "Active".
- [ ] RabbitMQ: workers consumindo; DLQ = 0.
- [ ] Redis: `ping → PONG`; memória < 70%.

**Critério de go:** todos os itens verdes → continuar monitoramento.
**Critério de no-go:** qualquer item vermelho → avaliar rollback (ver seção 8).

---

### 5.2 T+1 hora

**Owner: Tech Lead (em war room)**

- [ ] Taxa de erros 5xx < 0.5% acumulada desde o deploy.
- [ ] p95 de latência < 400ms sustentado.
- [ ] Nenhuma DLQ acumulando.
- [ ] Langfuse: latência de LLM estável (p95 < 5s).
- [ ] PostHog: eventos de produto sendo capturados normalmente.
- [ ] Custo OpenAI dentro do esperado para 1h de operação.
- [ ] Nenhum alerta P0 ou P1 aberto sem resolução.

**Critério de normalização:** todos os itens verdes → escalar monitoramento para rotina (não war room).

---

### 5.3 T+2 horas — Encerramento do War Room

**Owner: Tech Lead**

- [ ] Verificação final de todos os health checks.
- [ ] Nenhum incidente aberto.
- [ ] Mensagem de encerramento do war room no canal `#incidentes`:
  ```
  ✅ War room encerrado. Lançamento estável.
  Versão: v1.0.0 | Deploy: [hora] BRT
  Próxima verificação: rotina (D26 checklist diário amanhã às 10h).
  ```
- [ ] Monitoramento passa para modo de rotina (D26 checklist diário).

---

### 5.4 T+24 horas

**Owner: Tech Lead**

- [ ] Revisar métricas do dia anterior (Railway, Sentry, Langfuse, PostHog).
- [ ] Verificar se há notificações NOT-CES-05/06 não entregues (D26 — ALERTA-005).
- [ ] Verificar DLQs acumuladas overnight.
- [ ] Revisar custo OpenAI acumulado nas primeiras 24h.
- [ ] Criar issue de post-launch com observações para próximo release (se aplicável).

---

## 6. Deploy e Rollback

### 6.1 Deploy em Produção (Passo a Passo)

```bash
# 1. Garantir que develop está em staging, estável e com smoke tests verdes

# 2. Fazer merge do PR develop → main (via GitHub UI, 2 aprovações)
# CI dispara deploy-production.yml automaticamente

# 3. Acessar GitHub Actions → deploy-production.yml
# Aguardar chegada no step "Aguardando aprovação"
# Verificar smoke tests de staging uma última vez antes de aprovar

# 4. Aprovar o deployment no GitHub UI
# GitHub → Actions → deploy-production.yml → Review deployments → Approve

# 5. Acompanhar Railway Dashboard durante o Rolling Deploy
# https://railway.app/dashboard → backend-prod → Deployments

# 6. Aguardar CI executar smoke tests de produção (automático após deploy)
# CI falha → rollback automático executado pelo workflow

# 7. Verificar manualmente
curl https://app.repasse-seguro.com.br/health
# Esperado: {"status":"ok","version":"v1.0.0"}
```

### 6.2 Rollback — Gatilhos

| Gatilho | Tipo | Tempo de resposta |
|---|---|---|
| Smoke tests de produção falhando | Automático (CI) | Imediato |
| Taxa de erros 5xx > 2% em 5min pós-deploy | Manual (Tech Lead) | < 2 minutos |
| Instância Railway não-saudável | Manual (Tech Lead) | < 2 minutos |
| Spike de erros P0 no Sentry | Manual (Tech Lead) | < 5 minutos |
| Qualquer P0 identificado na launch window | Manual (Tech Lead) | Imediato |

[DECISÃO AUTÔNOMA — no-go automático se error rate > 2% nos primeiros 15min. Justificativa: threshold conservador protege UX sem bloquear releases estáveis. Alternativa descartada: threshold de 5% — tolerância alta demais para produto com transações financeiras.]

### 6.3 Rollback — Passo a Passo

```bash
# 1. Confirmar que rollback é necessário (error rate > 2% ou P0 ativo)

# 2. Executar rollback Railway
railway rollback --service backend-prod --environment production
# Troca de tráfego em ~30-60 segundos

# 3. Verificar que versão anterior está ativa
curl https://app.repasse-seguro.com.br/health
# Verificar campo "version" — deve ser versão anterior

# 4. Verificar Sentry — confirmar que spike de erros cessou

# 5. Notificar canal #incidentes
# "[ROLLBACK] v1.0.0 revertida para [versão anterior]. Motivo: [causa]. [hora] BRT"

# 6. Abrir post-mortem (D26 template)
# 7. Criar hotfix para o problema identificado (D24 — Hotfix Flow)
```

**Rollback frontend (Vercel):**

```bash
vercel rollback [previous-deployment-url]
# Ou via Vercel Dashboard → Deployments → Redeploy versão anterior
```

---

## 7. Incidentes por Categoria

### Backend (NestJS/Railway)

| Sintoma | Triagem | Ação inicial |
|---|---|---|
| 502/503 em todas as rotas | Deploy recente? Instância unhealthy? | Rollback imediato se deploy < 30min; verificar logs Railway |
| Erros 500 em endpoint específico | Stack trace no Sentry; migration recente? | Corrigir via hotfix; rollback se não tiver correção rápida |
| Timeout em endpoints | Banco lento? Redis indisponível? | Verificar p95 no Railway; consultar Supabase Dashboard |

### Banco de Dados (Supabase)

| Sintoma | Triagem | Ação inicial |
|---|---|---|
| Queries lentas (p95 > 2s) | Migration longa? N+1 queries? Lock? | `pg_stat_activity`; identificar query problemática |
| Conexões esgotadas | Pool saturado? Worker em loop? | Verificar contagem de conexões; reiniciar worker suspeito |
| Supabase down | Status page | Aguardar resolução; modo de manutenção na API |

### Cache (Redis/Upstash)

| Sintoma | Triagem | Ação inicial |
|---|---|---|
| `redis-cli ping` sem resposta | Upstash instável? | Verificar status Upstash; sessões serão perdidas (re-login) |
| Rate limit inativo | Redis indisponível | Monitorar tentativas de brute force via Sentry |

### Filas (RabbitMQ/CloudAMQP)

| Sintoma | Triagem | Ação inicial |
|---|---|---|
| Notificações não chegando | DLQ? Worker parado? | Verificar DLQ CloudAMQP; verificar logs do worker |
| KYC não processando | Worker KYC parado? idwall down? | Ativar revisão manual KYC no painel Admin |

### Autenticação

| Sintoma | Triagem | Ação inicial |
|---|---|---|
| Login retornando 500 | Supabase Auth down? | Verificar status Supabase; verificar logs do AuthService |
| Refresh token inválido para todos | JWT_SECRET alterado? | Verificar secret no Railway; usuários precisarão fazer login novamente |

### Integrações Externas

| Integração | Sintoma | Ação imediata | Contingência |
|---|---|---|---|
| ZapSign | Webhook não processado | Verificar HMAC secret; verificar URL no painel ZapSign | Admin processa assinatura manualmente |
| idwall | KYC falhando com EXT-001 | Verificar status idwall | Ativar revisão manual no painel Admin |
| Celcoin | Depósito Escrow não confirmado | Verificar credenciais e status Celcoin | Admin confirma depósito manualmente (ADR-003) |
| Resend | Email não entregue | Verificar status Resend; verificar RESEND_API_KEY | In-app sempre ativo; push como fallback |

### Frontend (React/Vercel)

| Sintoma | Triagem | Ação inicial |
|---|---|---|
| 404 em assets | Build incorreto? Deploy incompleto? | Verificar Vercel Dashboard; redeploy |
| Tela em branco | JavaScript error? | Verificar Sentry (Error Boundary deve capturar); rollback Vercel |

---

## 8. Backup e Restore

### 8.1 Frequência e Armazenamento

| Tipo | Frequência | Retenção | Armazenamento |
|---|---|---|---|
| Daily backup (Supabase Pro) | Diário 00:00 UTC | 7 dias | Supabase managed |
| PITR (Point-in-Time Recovery) | Contínuo | 7 dias | Supabase managed |
| Deploys anteriores | Contínuo | 5 últimas builds | Railway managed |

### 8.2 Quando Optar por Restore vs. Rollback

| Situação | Decisão | Justificativa |
|---|---|---|
| Regressão de código, sem dados corrompidos | **Rollback** | Mais rápido (30-60s), sem risco de perda de dados |
| Dados corrompidos por bug em migration | **Restore de banco + rollback de código** | Rollback de código não reverte dados já escritos |
| Migration destrutiva mal planejada | **Restore + hotfix** | Necessário recuperar dados apagados |
| Terceiros não estão respondendo (ZapSign/Celcoin) | **Nem rollback nem restore** — contingência manual | Problema externo, não de código ou dados |

### 8.3 Restore — Procedimento Resumido

```
1. Confirmar necessidade com Tech Lead (restore apaga dados após o ponto selecionado)
2. Identificar ponto de restore no Supabase Dashboard → Settings → Backups
3. Comunicar em #incidentes: "RESTORE PROGRAMADO para [timestamp]. Impacto: dados após [hora] perdidos."
4. Colocar API em modo de manutenção (503 com mensagem)
5. Executar restore via Supabase Dashboard (5-30 minutos)
6. Validar integridade: contagem de registros, operações ativas
7. Reiniciar serviços backend
8. Verificar smoke tests
9. Comunicar encerramento: "RESTORE CONCLUÍDO. [impacto]."
10. Iniciar post-mortem obrigatório
```

---

## 9. Comunicação de Incidente e Lançamento

### 9.1 Templates de Comunicação de Lançamento

**Anúncio pré-launch (T-1h):**

```
🚀 [PRÉ-LAUNCH] Repasse Seguro — Módulo Cessionário
Deploy iniciando em 1h — [hora] BRT
Versão: v1.0.0
Owner: @[nome]
War room: [link]
Status page (interno): [link dashboards]
```

**Confirmação de deploy iniciado:**

```
⏳ [DEPLOY] v1.0.0 em andamento — [hora] BRT
Railway Rolling Deploy iniciado. Acompanhar: [link Railway]
```

**Lançamento bem-sucedido:**

```
✅ [GO-LIVE] Repasse Seguro v1.0.0 em produção — [hora] BRT
Smoke tests: ✅ | Sentry: ✅ | Latência: < 300ms
Monitoramento intensivo por 2h. Próxima atualização: T+1h.
```

### 9.2 Templates de Comunicação de Incidente

**Abertura (P0/P1):**

```
🔴 [INCIDENTE P0] [hora BRT]
Produto: Repasse Seguro — Módulo Cessionário
Sintoma: [descrição]
Impacto: [o que está afetado]
Owner: @[nome]
Próxima atualização: 15min
```

**Atualização (a cada 15min para P0, 30min para P1):**

```
🟡 [INCIDENTE P0 ATUALIZAÇÃO] [hora BRT]
Status: Investigando / Contenção / Recuperação
Descoberta: [o que foi identificado]
Ação atual: [o que está sendo feito]
ETA: [estimativa]
```

**Encerramento:**

```
✅ [INCIDENTE RESOLVIDO] [hora BRT]
Duração: [X minutos]
Causa: [resumo]
Solução: [o que foi feito]
Post-mortem: [link ou "em elaboração até [data]"]
```

### 9.3 Frequência de Atualização por Severidade

| Severidade | Frequência mínima de atualização | Canal |
|---|---|---|
| P0 | A cada 15 minutos | `#incidentes` + PagerDuty |
| P1 | A cada 30 minutos | `#incidentes` |
| P2 | Ao abrir e ao resolver | `#alertas` |
| P3 | Apenas ao resolver | `#alertas` ou issue |

---

## 10. Critério de Go/No-Go

### 10.1 Quem Decide

**Decisor:** Tech Lead. Em ausência, responsável técnico designado.

**Momento da decisão:** T-0 (conforme checklist de Launch Day, linha "GO/NO-GO decision").

### 10.2 Critérios de GO (todos devem ser verdadeiros)

- [ ] E2E P0 (fluxos críticos D27) passando no staging.
- [ ] Smoke tests de staging passando.
- [ ] Todas as integrações externas operacionais (Celcoin, ZapSign, idwall, Supabase, Resend).
- [ ] `pnpm audit` sem vulnerabilidades critical/high.
- [ ] D28 (Checklist de Qualidade) aprovado.
- [ ] Migrations Prisma retrocompatíveis e com rollback documentado.
- [ ] Error rate em staging < 0.5% nas últimas 2h.
- [ ] Latência p95 em staging < 400ms.
- [ ] Sentry sem issues P0 abertos em staging.
- [ ] Secrets de produção confirmados nos vaults corretos (Railway, Vercel, EAS).
- [ ] Rollback testado e documentado (D24 + seção 6.3 deste documento).
- [ ] War room ativo com participantes confirmados.
- [ ] Janela de deploy dentro do horário permitido (terça-quinta, 10h-14h BRT).

### 10.3 Critérios de NO-GO (qualquer um é suficiente para bloquear)

- [ ] Qualquer E2E P0 falhando.
- [ ] Integração crítica (Celcoin, ZapSign, KYC) com incidente ativo.
- [ ] Migration sem rollback documentado.
- [ ] Secret de produção não configurado.
- [ ] Error rate em staging > 1%.
- [ ] Tech Lead indisponível ou war room sem cobertura.
- [ ] Horário fora da janela permitida (sexta-feira após 16h, véspera de feriado).
- [ ] Hotfix urgente não testado incluído na release.

### 10.4 O que Acontece se a Decisão for NO-GO

1. Tech Lead anuncia no canal `#deploys`: "NO-GO para [data/hora]. Motivo: [critério não atendido]."
2. Deploy não é iniciado.
3. Issue criada para resolver o critério bloqueante.
4. Nova janela de go-live agendada assim que critério for resolvido.
5. Code freeze mantido até nova tentativa.

---

## 11. Backlog de Pendências

| ID | Item | Tipo | Prioridade |
|---|---|---|---|
| GOLIVE-001 | Obter e documentar contatos de suporte técnico de Celcoin, ZapSign e idwall | [DADO PENDENTE] | Alta |
| GOLIVE-002 | Criar documento compartilhado de war room (links de dashboards, contatos) para o dia do launch | Ação técnica | Alta |
| GOLIVE-003 | Definir data e janela do primeiro go-live real | [DADO PENDENTE — depende de cronograma de produto] | Alta |
| GOLIVE-004 | Configurar PagerDuty com políticas de escalation para P0 | Ação técnica | Alta |
| GOLIVE-005 | Criar usuários de teste em produção para smoke tests pós-deploy (sem dados reais) | Ação técnica | Alta |
| GOLIVE-006 | Definir status page pública para usuários finais em caso de incidente prolongado | [DEFINIÇÃO PENDENTE] — Opção A: Atlassian Statuspage (custo mensal, profissional); Opção B: página estática no Vercel (gratuito, menos recursos). Impacto: transparência com usuários durante incidentes. | Média |

---

*Documento gerado pelo pipeline ShiftLabs v9.5 — Fase 2. Executor: Claude Code Desktop.*
