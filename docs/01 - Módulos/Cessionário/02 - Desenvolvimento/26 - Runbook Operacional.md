# 26 - Runbook Operacional

| Campo | Valor |
|---|---|
| **Nome do Documento** | 26 - Runbook Operacional |
| **Versão** | v1.0 |
| **Data** | 2026-03-22 (America/Fortaleza) |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Produto** | Repasse Seguro — Módulo Cessionário |

---

> 📌 **TL;DR — Resumo Operacional**
>
> - **4 severidades:** P0 (crítico/imediato), P1 (alto/30min), P2 (médio/2h), P3 (baixo/next business day).
> - **Componentes críticos:** API NestJS (Railway), PostgreSQL (Supabase), Redis (Upstash), RabbitMQ (CloudAMQP), ZapSign, idwall, Celcoin.
> - **Rollback:** `railway rollback` em < 60s; sempre verificar migrations antes.
> - **Comunicação:** Slack `#incidentes` (P0/P1) e `#alertas` (P2/P3). PagerDuty para P0.
> - **Post-mortem obrigatório** para todo incidente P0 e P1 que afete Escrow, KYC ou formalização.
> - **Contingência manual:** Admin pode confirmar depósitos Escrow manualmente (MVP — ADR-003) se Celcoin estiver indisponível.
> - **Escalation:** Tech Lead (primeiro) → responsável técnico → plataformas externas (suporte).

---

## 1. Classificação de Severidade

| Severidade | Definição | SLA Resposta | SLA Resolução | Impacto típico | Canal | Escalation |
|---|---|---|---|---|---|---|
| **P0 — Crítico** | Indisponibilidade total ou falha em fluxo financeiro/KYC em produção | 5 minutos | 1 hora | Usuários incapazes de operar; transações bloqueadas; risco regulatório | PagerDuty + Slack `#incidentes` | Tech Lead imediato |
| **P1 — Alto** | Degradação severa de funcionalidade principal; notificação crítica não entregue | 15 minutos | 2 horas | Parte dos usuários afetada; SLA de negócio em risco | Slack `#incidentes` | Tech Lead em 15min |
| **P2 — Médio** | Funcionalidade secundária degradada; latência elevada sem impacto em transações | 30 minutos | 4 horas | Experiência degradada mas operação não bloqueada | Slack `#alertas` | Tech Lead se não resolvido em 2h |
| **P3 — Baixo** | Anomalia não urgente; log anômalo; métrica levemente fora do normal | 2 horas | 1 dia útil | Sem impacto perceptível ao usuário | Slack `#alertas` ou issue | Tech Lead no próximo dia útil |

**Componentes e criticidade:**

| Componente | Criticidade | Razão |
|---|---|---|
| API backend (Railway) | P0 | Toda a operação depende |
| PostgreSQL (Supabase) | P0 | Dados da plataforma |
| Celcoin (Escrow) | P0 | Movimentação financeira real |
| ZapSign (assinatura) | P1 | Bloqueia formalização |
| idwall (KYC) | P1 | Bloqueia onboarding; fallback manual disponível |
| Redis (Upstash) | P1 | Cache e sessões; degradação sem perda de dados |
| RabbitMQ (CloudAMQP) | P1 | Notificações e KYC async |
| Resend (email) | P2 | Fallback: in-app sempre ativo |
| Expo Notifications (push) | P2 | Fallback: in-app e email |

---

## 2. Triagem Inicial

Ao receber alerta ou relato de incidente, executar nesta ordem:

```
1. CONFIRMAR IMPACTO
   - O alerta é real ou falso positivo?
   - Quantos usuários estão afetados?
   - Qual funcionalidade está impactada?

2. ISOLAR ESCOPO
   - O problema é em produção, staging ou ambos?
   - É um módulo específico ou toda a plataforma?
   - Foi precedido por um deploy recente (< 30min)?

3. VERIFICAR DEPLOY RECENTE
   - Verificar Railway deployments: https://railway.app/dashboard
   - Se há deploy recente: iniciar rollback sem aguardar diagnóstico completo (P0/P1)

4. VALIDAR DEPENDÊNCIAS EXTERNAS
   - Celcoin: https://status.celcoin.com.br
   - ZapSign: verificar painel de administração
   - Supabase: https://status.supabase.com
   - CloudAMQP: https://status.cloudamqp.com
   - Upstash: https://status.upstash.com

5. CONSULTAR DASHBOARDS
   - Sentry: https://sentry.io/{org}/repasse-seguro
   - Railway: https://railway.app/dashboard
   - Supabase Dashboard: https://app.supabase.com
   - Langfuse (se incidente de IA): https://cloud.langfuse.com

6. DECIDIR CONTENÇÃO
   - Rollback imediato? (deploy recente + P0/P1)
   - Desabilitar feature flag temporariamente?
   - Acionar contingência manual?
   - Aguardar diagnóstico? (P2/P3 sem deploy recente)

7. COMUNICAR
   - Postar no canal correto com severidade, sintoma e próximo passo
   - Nunca deixar incidente sem atualização por mais de 15min (P0) ou 30min (P1)
```

---

## 3. Catálogo de Alertas

### ALERTA-001: API Indisponível (5xx > 5% em 1min)

| Campo | Valor |
|---|---|
| **Condição** | Taxa de erros 5xx > 5% em janela de 1 minuto |
| **Severidade** | P0 |
| **Canal** | PagerDuty + Slack `#incidentes` |
| **Sintomas** | Usuários recebendo 502/503; Railway mostra instância unhealthy |

**Diagnóstico:**

```bash
# 1. Verificar status Railway
railway status --service backend-prod --environment production

# 2. Verificar logs recentes
railway logs --service backend-prod --environment production --tail 100

# 3. Verificar Sentry para stack traces
# https://sentry.io/{org}/repasse-seguro/issues/

# 4. Verificar saúde do banco
# Supabase Dashboard → Database → Performance
```

**Ação imediata:**

1. Se deploy < 30min → executar rollback imediato (ver seção 5).
2. Se sem deploy recente → verificar logs e Sentry para causa raiz.
3. Verificar se é falha de migração Prisma pendente.

**Critério de encerramento:** `/health` retornando 200; taxa de 5xx < 0.1% por 5min consecutivos.

---

### ALERTA-002: Banco de Dados Lento (p95 queries > 2s)

| Campo | Valor |
|---|---|
| **Condição** | p95 de latência de queries > 2 segundos |
| **Severidade** | P1 |
| **Canal** | Slack `#incidentes` |

**Diagnóstico:**

```sql
-- No Supabase SQL Editor ou psql:
-- Queries mais lentas em execução agora
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - pg_stat_activity.query_start > interval '1 second'
ORDER BY duration DESC;

-- Índices faltando (seq scans em tabelas grandes)
SELECT schemaname, relname, seq_scan, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan AND n_live_tup > 10000
ORDER BY seq_scan DESC;
```

**Ação imediata:**

1. Verificar se há migration longa em execução (`pg_stat_activity`).
2. Identificar query mais lenta e avaliar se é regressão de código recente.
3. Se há lock contention: identificar PID e avaliar se `pg_terminate_backend(pid)` é seguro.

**Critério de encerramento:** p95 de queries < 500ms por 5min consecutivos.

---

### ALERTA-003: Fila RabbitMQ com DLQ Acumulando (> 10 mensagens)

| Campo | Valor |
|---|---|
| **Condição** | DLQ de qualquer fila com > 10 mensagens |
| **Severidade** | P1 (notificação) / P0 (escrow, KYC) |
| **Canal** | Slack `#incidentes` |

**Diagnóstico:**

```bash
# Via CloudAMQP Dashboard: verificar quais DLQs estão acumulando
# URL: https://customer.cloudamqp.com

# Via RabbitMQ Management API (se disponível):
curl -u guest:guest https://[cloudamqp-url]/api/queues
# Verificar campos: messages_ready, messages_unacknowledged em *-dlq

# Verificar logs do worker que está falhando:
railway logs --service backend-prod | grep "NACK\|DLQ\|dead-letter"
```

**Ação imediata:**

1. Identificar qual fila está acumulando DLQ (`kyc.process.dlq`, `notification.email.dlq`, `escrow.process.dlq`).
2. Verificar logs do worker correspondente para causa raiz.
3. Se é fila de notificação: impacto é degradado mas não crítico (salvo NOT-CES-05/06).
4. Se é fila KYC ou Escrow → severidade P0, iniciar investigação imediata.
5. Após correção, reprocessar mensagens da DLQ manualmente se necessário.

**Critério de encerramento:** DLQ com 0 mensagens e worker consumindo normalmente.

---

### ALERTA-004: Redis Indisponível ou Memória Crítica (> 80%)

| Campo | Valor |
|---|---|
| **Condição** | Redis inacessível OU uso de memória > 80% |
| **Severidade** | P1 |
| **Canal** | Slack `#incidentes` |

**Diagnóstico:**

```bash
# Testar conectividade Redis (Upstash)
redis-cli -u $UPSTASH_REDIS_URL ping
# Esperado: PONG

# Verificar memória usada
redis-cli -u $UPSTASH_REDIS_URL info memory | grep used_memory_human

# Verificar chaves com maior tamanho
redis-cli -u $UPSTASH_REDIS_URL --bigkeys
```

**Ação imediata:**

1. Se Redis indisponível: sessões de usuário serão perdidas (re-login necessário); rate limiting não funciona.
2. Se memória crítica: identificar chaves sem TTL com `--bigkeys` e aplicar TTL.
3. Rate limiting de auth ficará inativo — monitorar tentativas de brute force via Sentry.

**Critério de encerramento:** `ping` respondendo; memória < 70% por 5min.

---

### ALERTA-005: NOT-CES-05/06 Não Entregues em 30min

| Campo | Valor |
|---|---|
| **Condição** | Alerta de prazo Escrow (NOT-CES-05 ou NOT-CES-06) não entregue em 30min após disparo |
| **Severidade** | P0 |
| **Canal** | PagerDuty + Slack `#incidentes` |

**Diagnóstico:**

```bash
# Verificar notification_logs para NOT-CES-05 ou NOT-CES-06 com status FAILED/QUEUED
# Via Supabase Dashboard → Table Editor → notification_logs
# Filtro: template IN ('NOT-CES-05', 'NOT-CES-06') AND status != 'DELIVERED'

# Verificar worker de email
railway logs --service backend-prod | grep "notification.email\|NOT-CES-0"

# Verificar fila de email no RabbitMQ
# CloudAMQP Dashboard → notification.email → messages
```

**Ação imediata:**

1. Verificar se Resend está operacional: https://status.resend.com
2. Verificar DLQ de `notification.email.dlq`.
3. Se Resend indisponível: acionar envio manual via painel Admin (D15 — contingência).
4. Verificar se o Cessionário tem push notifications ativas (fallback).
5. In-app está sempre ativo — confirmar que o alerta aparece no Dashboard do usuário.

**Critério de encerramento:** `notification_logs.status = 'DELIVERED'` para todas as mensagens afetadas.

---

### ALERTA-006: Latência API > 500ms (p95)

| Campo | Valor |
|---|---|
| **Condição** | p95 de latência de API > 500ms por 5min |
| **Severidade** | P2 |
| **Canal** | Slack `#alertas` |

**Diagnóstico:**

```bash
# Verificar endpoints mais lentos no Railway Metrics
# Verificar se há query N+1 nos logs do Prisma
railway logs | grep "query\|slow\|duration"

# PostHog: verificar se sessões de usuário têm interações lentas
# Langfuse: verificar se endpoint de IA está acima do SLA (5s para primeiro chunk)
```

**Ação imediata:**

1. Identificar endpoint mais lento via Railway Metrics.
2. Verificar se há deploy recente com regressão de performance.
3. Se é endpoint de IA: verificar Langfuse para latência de LLM.

**Critério de encerramento:** p95 < 400ms por 10min consecutivos.

---

### ALERTA-007: Custo Diário OpenAI > $50

| Campo | Valor |
|---|---|
| **Condição** | Custo diário OpenAI > $50 (alerta) ou > $200 (hard limit) |
| **Severidade** | P2 (alerta $50) / P1 (hard limit $200) |
| **Canal** | Slack `#alertas` (alerta) / `#incidentes` (hard limit) |

**Diagnóstico:**

```bash
# Verificar Langfuse: https://cloud.langfuse.com
# Metrics → Cost → Daily breakdown
# Identificar qual trace/usuário está gerando custo anormal
```

**Ação imediata:**

1. Verificar se há usuário abusando da API de IA (rate limit de 20 chamadas/min por usuário).
2. Se custo acima de $200: suspender temporariamente endpoint `/ai/analyze` via feature flag.
3. Verificar se há loop ou retry infinito chamando OpenAI.

**Critério de encerramento:** custo projetado do dia abaixo de $50; causa raiz identificada.

---

### ALERTA-008: Taxa de Erro KYC > 20% em 1h

| Campo | Valor |
|---|---|
| **Condição** | Taxa de erro do worker KYC > 20% em janela de 1 hora |
| **Severidade** | P1 |
| **Canal** | Slack `#incidentes` |

**Diagnóstico:**

```bash
# Verificar logs do worker KYC
railway logs --service backend-prod | grep "kyc.process\|idwall\|KYC_ERROR"

# Verificar se idwall está disponível
# Acessar painel idwall para verificar status do serviço
```

**Ação imediata:**

1. Verificar status do idwall com o suporte.
2. Se idwall indisponível: ativar fallback de revisão manual pelo Admin.
3. Novas submissões de KYC serão enfileiradas e reprocessadas quando idwall voltar.
4. Comunicar ao Admin que há revisões manuais pendentes.

**Critério de encerramento:** worker KYC consumindo com taxa de erro < 1% por 15min.

---

### ALERTA-009: ZapSign Webhook Falhou 3x

| Campo | Valor |
|---|---|
| **Condição** | Webhook ZapSign atingiu 3 tentativas sem sucesso |
| **Severidade** | P1 |
| **Canal** | Slack `#incidentes` |

**Diagnóstico:**

```bash
# Verificar logs de webhook ZapSign
railway logs --service backend-prod | grep "zapsign\|webhook\|HMAC"

# Verificar tabela de assinaturas pendentes
# Supabase → Table Editor → document_signatures WHERE status = 'PENDING' AND retry_count >= 3
```

**Ação imediata:**

1. Verificar se a URL do webhook está correta no painel ZapSign.
2. Verificar se `ZAPSIGN_WEBHOOK_SECRET` está correto (HMAC-SHA256).
3. Processar manualmente a atualização de status de assinatura via painel ZapSign + Admin.

**Critério de encerramento:** webhook sendo processado com sucesso; `document_signatures.status` atualizado.

---

### ALERTA-010: Taxa de Falha em Login > 30% (possível brute force)

| Campo | Valor |
|---|---|
| **Condição** | Taxa de falha em `POST /auth/login` > 30% em 10min |
| **Severidade** | P1 |
| **Canal** | Slack `#incidentes` |

**Diagnóstico:**

```bash
# Verificar IPs com múltiplas tentativas no Redis
redis-cli -u $UPSTASH_REDIS_URL KEYS "rs:auth:attempts:*" | head -20

# Verificar Sentry para padrão de erro AUTH-002 (credenciais inválidas)
# Verificar se há IP específico com volume anormal
```

**Ação imediata:**

1. Verificar se o rate limit está funcionando (5 tentativas → lockout 15min via Redis).
2. Se há IP específico com volume anormal: bloquear no Railway ou firewall de rede.
3. Verificar se Supabase Auth está operacional.

**Critério de encerramento:** taxa de falha de login < 10%; nenhum IP com volume anormal.

---

## 4. Diagnóstico por Camada

### 4.1 Aplicação (NestJS/Railway)

```bash
# Saúde geral
curl https://app.repasse-seguro.com.br/health
# Esperado: { "status": "ok", "version": "v1.x.x", "uptime": "..." }

# Logs recentes (últimos 100 registros)
railway logs --service backend-prod --environment production --tail 100

# Verificar instâncias ativas
railway status --service backend-prod --environment production

# Verificar métricas de CPU e memória
# Railway Dashboard → Service → Metrics
```

**Sinais de problema:**

- `status: "error"` em `/health`
- Instâncias restartando ciclicamente
- CPU > 80% sustentado por > 5min
- Memória > 90% — risco de OOM

**Próximo passo se sem solução em 10min:** rollback se há deploy recente; escalar para Tech Lead.

### 4.2 Banco de Dados (Supabase/PostgreSQL)

```sql
-- Conexões ativas
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
-- Alerta se connections > 80% do pool (max_connections via Supabase plan)

-- Tabelas com lock
SELECT pid, relation::regclass, mode, granted
FROM pg_locks l JOIN pg_stat_activity s ON l.pid = s.pid
WHERE NOT granted;

-- Transactions longas (> 30s)
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL AND now() - xact_start > interval '30 seconds';
```

**Próximo passo:** se lock contention → identificar PID bloqueador; se query lenta → `EXPLAIN ANALYZE` e avaliar índice faltante.

### 4.3 Cache (Redis/Upstash)

```bash
# Verificar conectividade
redis-cli -u $UPSTASH_REDIS_URL ping

# Estatísticas de hit/miss
redis-cli -u $UPSTASH_REDIS_URL info stats | grep keyspace_hits\|keyspace_misses

# TTL de chaves críticas
redis-cli -u $UPSTASH_REDIS_URL TTL rs:celcoin:access-token
redis-cli -u $UPSTASH_REDIS_URL TTL rs:session:{userId}

# Verificar se há chaves sem TTL
redis-cli -u $UPSTASH_REDIS_URL DEBUG sleep 0 # diagnóstico sem impacto
```

**Próximo passo:** se Redis fora: sessões serão inválidas (re-login), rate limit inativo. Monitorar brute force via Sentry.

### 4.4 Filas (RabbitMQ/CloudAMQP)

```bash
# Via CloudAMQP Dashboard (interface web):
# 1. Verificar overview: total de mensagens em fila, consumers ativos
# 2. Verificar DLQs: notification.email.dlq, kyc.process.dlq, escrow.process.dlq

# Reprocessar mensagem da DLQ manualmente (se necessário):
# CloudAMQP → Queues → [dlq-name] → Get messages → Move to queue principal
```

**Próximo passo:** identificar worker parado e verificar seus logs. Se worker crashando em loop → investigar causa raiz antes de reprocessar DLQ.

### 4.5 Integrações Externas

| Integração | Verificação | URL de Status |
|---|---|---|
| ZapSign | Painel de administração ZapSign | [DADO PENDENTE — URL do painel] |
| idwall | Contato com suporte idwall | [DADO PENDENTE — URL de status] |
| Celcoin | Contato com suporte Celcoin | https://status.celcoin.com.br |
| Resend (email) | Dashboard de entrega Resend | https://status.resend.com |
| Supabase | Status page pública | https://status.supabase.com |
| Upstash | Status page | https://status.upstash.com |
| CloudAMQP | Status page | https://status.cloudamqp.com |

**Diagnóstico de integração externa:**

```bash
# Testar endpoint de saúde de integração (exemplo: ZapSign)
curl -X GET https://api.zapsign.com.br/api/v1/health \
  -H "Authorization: Bearer $ZAPSIGN_API_TOKEN"

# Verificar logs de chamadas externas recentes
railway logs --service backend-prod | grep "EXT-\|timeout\|ECONNREFUSED"
```

### 4.6 Autenticação (Supabase Auth + JWT)

```bash
# Verificar se Supabase Auth está operacional
# https://status.supabase.com

# Verificar se refresh tokens estão sendo aceitos
curl -X POST https://app.repasse-seguro.com.br/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "[test-token]"}'

# Verificar chaves de sessão no Redis
redis-cli -u $UPSTASH_REDIS_URL KEYS "rs:session:*" | wc -l
redis-cli -u $UPSTASH_REDIS_URL KEYS "rs:auth:refresh:*" | wc -l
```

### 4.7 Infraestrutura (Railway)

```bash
# Verificar status do serviço
railway status --service backend-prod

# Listar deploys recentes
railway deployments list --service backend-prod --environment production

# Verificar uso de recursos
# Railway Dashboard → Service → Metrics → CPU, Memory, Network
```

---

## 5. Rollback

### 5.1 Critério para Acionamento

Acionar rollback imediatamente se:

- Deploy há < 30min E qualquer alerta P0/P1 ativo.
- Smoke tests falhando no pós-deploy.
- Spike de 5xx > 5% por > 2min após deploy.
- Migrations falhando no deploy.

**Quem pode acionar:** Tech Lead, on-call designado. Em emergência, qualquer membro técnico via Railway Dashboard.

### 5.2 Rollback Backend (Railway) — Passo a Passo

```bash
# 1. Verificar deploys disponíveis para rollback
railway deployments list --service backend-prod --environment production

# 2. Executar rollback para deploy anterior
railway rollback --service backend-prod --environment production
# Railway troca tráfego em ~30-60 segundos

# 3. Confirmar que nova versão está ativa
curl https://app.repasse-seguro.com.br/health
# Verificar campo "version" na resposta

# 4. Monitorar por 5min
# - Sentry: verificar que spike de erros cessou
# - Railway: confirmar que instância está healthy

# 5. Notificar canal
# Slack #incidentes: "[ROLLBACK] Backend revertido para [versão]. Motivo: [descrição]. Monitorar até [hora]."
```

**Tempo esperado:** 30-60 segundos para troca de tráfego.

**Riscos colaterais:**

- Migrations já aplicadas permanecem no banco. Se a migration foi incompatível com o código revertido, pode causar novos erros.
- Se migration é incompatível → executar migration de rollback manualmente antes do rollback de código.

**Validações obrigatórias após rollback:**

- [ ] `/health` retornando 200 com versão correta.
- [ ] Taxa de 5xx < 0.1% por 5min.
- [ ] RabbitMQ workers consumindo normalmente.
- [ ] Redis respondendo (`ping → PONG`).
- [ ] Alerta postado no canal `#incidentes`.

### 5.3 Rollback Frontend (Vercel)

```bash
# Via CLI
vercel rollback [previous-deployment-url]

# Via Dashboard Vercel:
# 1. Acessar https://vercel.com/dashboard
# 2. Selecionar o projeto
# 3. Deployments → selecionar versão anterior → "Redeploy"
# Tempo esperado: 10-20 segundos
```

### 5.4 Rollback Mobile (Expo OTA)

```bash
# Reverter OTA update
eas update:rollback --branch production --to [previous-update-group-id]
# Apenas para OTA — builds de store requerem nova submissão
```

---

## 6. Restore e Recuperação de Dados

### 6.1 Backup do Banco (Supabase)

O Supabase realiza backups automáticos diários (plano Pro). Para projetos Pro, há Point-in-Time Recovery (PITR) com granularidade de 1 minuto.

| Tipo | Frequência | Retenção | Como acessar |
|---|---|---|---|
| Daily backup | Diário 00:00 UTC | 7 dias (Pro) / 30 dias (Team) | Supabase Dashboard → Settings → Backups |
| PITR (Point-in-Time Recovery) | Contínuo | 7 dias | Supabase Dashboard → Settings → Backups → PITR |

### 6.2 Procedimento de Restore

> 🔴 **ATENÇÃO:** Restore de banco de dados apaga dados gerados após o ponto de restore. Executar apenas com autorização explícita do Tech Lead.

```
1. CONFIRMAR NECESSIDADE
   - Identificar exatamente quais dados foram corrompidos/perdidos
   - Verificar se é possível corrigir via migration sem restore completo

2. SELECIONAR PONTO DE RESTORE
   - Supabase Dashboard → Settings → Backups
   - Selecionar ponto anterior ao incidente

3. COMUNICAR ANTES DO RESTORE
   - Postar em #incidentes: "RESTORE PROGRAMADO: banco de [projeto] será restaurado para [timestamp].
     Dados criados após esse ponto serão perdidos. Janela de manutenção: [hora-hora]."
   - Colocar a plataforma em modo de manutenção (se possível)

4. EXECUTAR RESTORE
   - Supabase Dashboard → Settings → Backups → [selecionar] → Restore
   - Aguardar conclusão (pode levar de 5 a 30 minutos dependendo do volume)

5. VALIDAR INTEGRIDADE
   - Verificar contagem de registros em tabelas críticas
   - Verificar operações ativas (proposals, negotiations, escrow) — podem estar em estado inconsistente
   - Executar queries de integridade referencial

6. COMUNICAR ENCERRAMENTO
   - Postar em #incidentes: "RESTORE CONCLUÍDO. Banco restaurado para [timestamp].
     Usuários afetados: [número]. Ação necessária: [o que o usuário deve refazer, se aplicável]."
```

**Impacto esperado de restore:**

- Usuários que criaram propostas/negociações após o ponto de restore precisarão refazê-las.
- Transações Escrow confirmadas após o ponto podem precisar de reconciliação manual com Celcoin.
- Postmortem obrigatório após qualquer restore em produção.

---

## 7. Cenários de Incidente

### CENÁRIO-001: Indisponibilidade Total da API

**Sintoma:** Usuários recebendo 502/503; Railway mostra instância unhealthy.

**Triagem:**

1. Verificar se há deploy recente (< 30min) → rollback imediato.
2. Verificar logs do Railway → identificar causa (OOM, crash, migration falha).
3. Verificar se Supabase está operacional.

**Ação:**

1. Se deploy recente: `railway rollback` (ver seção 5).
2. Se OOM: aumentar memória do serviço no Railway e reanalisar memory leaks.
3. Se migration falhou: verificar estado da migration (`prisma migrate status`), reverter migration manualmente se necessário.

**Tempo esperado de resolução:** < 5 minutos com rollback; 30-60 minutos se requer correção de código.

---

### CENÁRIO-002: Falha em Banco de Dados (Supabase Down)

**Sintoma:** API retornando 503 com erro `DB-001`; Supabase status page com incidente.

**Triagem:**

1. Verificar https://status.supabase.com.
2. Verificar se é indisponibilidade parcial (somente realtime, somente auth, ou banco principal).

**Ação:**

1. Se indisponibilidade total do Supabase: colocar API em modo degradado (retornar 503 com mensagem de manutenção).
2. Comunicar usuários via Slack interno e, se prolongado, via in-app (se in-app ainda funcionar com cache).
3. Aguardar resolução do Supabase — não há fallback de banco.

**Tempo esperado de resolução:** depende do Supabase SLA (99.9% uptime).

---

### CENÁRIO-003: Fila RabbitMQ Parada

**Sintoma:** Notificações não sendo enviadas; KYC não sendo processado; DLQ acumulando.

**Triagem:**

1. Verificar CloudAMQP Dashboard — fila parou de consumir?
2. Verificar logs do worker correspondente.
3. Verificar se é falha de conexão ou falha de processamento.

**Ação:**

1. Se worker crashando: reiniciar serviço backend (`railway redeploy`).
2. Se CloudAMQP instável: verificar status CloudAMQP e aguardar resolução.
3. Reprocessar DLQ após correção (CloudAMQP Dashboard → mover mensagens de DLQ para fila principal).

**Contingência:** notificações críticas (NOT-CES-05/06) → acionar envio manual via painel Admin.

---

### CENÁRIO-004: Integração Externa Instável (ZapSign/idwall/Celcoin)

**Sintoma:** Erros `EXT-001` ou `EXT-002` em Sentry; operações de formalização/KYC/Escrow falhando.

**Triagem:**

1. Identificar qual integração está falhando (Sentry → error code `EXT-`).
2. Verificar status page da integração afetada.
3. Verificar se é timeout (latência elevada) ou erro de autenticação (token expirado).

**Ação por integração:**

| Integração | Ação imediata | Contingência |
|---|---|---|
| ZapSign | Verificar `ZAPSIGN_API_TOKEN`; aguardar resolução | Admin processa assinatura manual via painel ZapSign |
| idwall | Ativar revisão manual KYC no painel Admin | Workers reprocessam após retorno do idwall |
| Celcoin | Verificar `CELCOIN_CLIENT_ID/SECRET`; aguardar resolução | Admin confirma depósito Escrow manualmente (ADR-003 — MVP) |
| Resend | Verificar `RESEND_API_KEY`; aguardar resolução | Notificações in-app sempre ativas; push como fallback |

---

### CENÁRIO-005: Erro Massivo Após Deploy (Regressão)

**Sintoma:** Spike de erros no Sentry imediatamente após deploy; taxa de 5xx > 5%.

**Triagem:**

1. Confirmar que o spike coincide com o horário do deploy.
2. Identificar o erro mais frequente no Sentry.
3. Verificar se é migration incompatível ou regressão de lógica.

**Ação:**

1. Rollback imediato (`railway rollback`) sem aguardar diagnóstico completo se P0.
2. Após rollback: analisar causa raiz no Sentry com código da versão revertida.
3. Criar hotfix com correção mínima (ver D24, seção 14).

**Prevenção:** este cenário deve ser raro se E2E e smoke tests estão funcionando conforme D24.

---

## 8. Contingência Manual

### 8.1 Confirmação Manual de Depósito Escrow

**Quando ativar:** Celcoin indisponível ou apresentando erros ao confirmar depósito.

**Duração máxima:** até resolução da integração Celcoin (sem prazo fixo definido no MVP — ADR-003).

**Procedimento:**

1. Admin acessa painel de gestão da Repasse Seguro.
2. Localiza operação com depósito pendente de confirmação.
3. Verifica comprovante de depósito enviado pelo Cessionário.
4. Confirma manualmente via interface Admin.
5. Sistema processa o restante do fluxo automaticamente.

**Risco operacional:** dependência de disponibilidade do Admin; maior tempo de processamento. Cessionário deve ser notificado sobre o atraso.

### 8.2 Revisão Manual de KYC

**Quando ativar:** idwall indisponível.

**Procedimento:**

1. KYC jobs ficam na fila `kyc.process` durante indisponibilidade do idwall.
2. Admin acessa fila de revisão manual no painel.
3. Revisa documentos enviados pelo Cessionário.
4. Aprova ou rejeita manualmente.
5. Quando idwall voltar: workers retomam processamento da fila.

### 8.3 Disparo Manual de Notificações Críticas

**Quando ativar:** worker de email/push falhando e NOT-CES-05/06 em risco de não ser entregue.

**Procedimento:**

1. Acessar painel Admin → Notificações → Envio manual.
2. Identificar Cessionários com prazo de Escrow nas próximas 48h.
3. Disparar NOT-CES-05 ou NOT-CES-06 manualmente.
4. Registrar no log de incidente que envio foi manual.

---

## 9. Checklists Recorrentes

### 9.1 Checklist Diário (Manhã — antes das 10h BRT)

```
APLICAÇÃO
[ ] API /health retornando 200 em produção
[ ] Sentry: nenhum novo erro crítico overnight
[ ] Taxa de erros < 1% nas últimas 24h

BANCO DE DADOS
[ ] Supabase Dashboard: nenhum alerta de performance
[ ] Conexões ativas dentro do pool

FILAS
[ ] Nenhuma DLQ acumulando (CloudAMQP)
[ ] Workers de notificação, KYC e Escrow consumindo

FINANCEIRO
[ ] Custo OpenAI nas últimas 24h dentro do orçamento ($50/dia)
[ ] Nenhuma operação Escrow com depósito pendente há > 24h sem resposta

NOTIFICAÇÕES
[ ] NOT-CES-05/06: verificar se há alertas de prazo a disparar hoje
```

### 9.2 Checklist Semanal (Segunda-feira)

```
SEGURANÇA
[ ] pnpm audit --audit-level=high sem vulnerabilidades novas
[ ] Rotação de secrets programada para o mês? (ver D22)
[ ] Nenhum acesso indevido nos logs de autenticação Supabase

PERFORMANCE
[ ] p95 de latência da semana anterior < 500ms
[ ] Bundle size do frontend estável
[ ] Nenhuma query lenta nova no Supabase

INTEGRAÇÕES
[ ] ZapSign: nenhuma assinatura pendente há > 5 dias
[ ] idwall: taxa de aprovação KYC dentro do esperado
[ ] Celcoin: nenhuma operação Escrow pendente anormal

BACKUPS
[ ] Verificar que último backup do Supabase foi concluído com sucesso

PENDÊNCIAS
[ ] Revisar itens do backlog de pendências com prioridade Alta
```

### 9.3 Checklist Pós-Incidente

```
IMEDIATO (nas primeiras 2h após resolução)
[ ] Serviço totalmente restaurado e verificado
[ ] Alerta de resolução postado em #incidentes
[ ] Usuários afetados identificados
[ ] Causa raiz preliminar identificada

DENTRO DE 24H
[ ] Post-mortem redigido (ver seção 12)
[ ] Issue criada para ação corretiva
[ ] Runbook atualizado se procedimento foi inadequado
[ ] Comunicação final enviada se usuários foram impactados

DENTRO DE 1 SEMANA
[ ] Ação corretiva implementada ou com prazo definido
[ ] Testes adicionados para cobrir o cenário do incidente
[ ] Revisão do alerta que deveria ter detectado antes
```

---

## 10. URLs e Ferramentas Críticas

| Recurso | URL / Comando | Uso | Credencial necessária |
|---|---|---|---|
| Railway Dashboard | https://railway.app/dashboard | Status, logs, rollback, métricas | Railway account |
| Supabase Dashboard | https://app.supabase.com | Banco, auth, storage, realtime | Supabase account |
| Sentry | https://sentry.io/{org}/repasse-seguro | Error tracking, stack traces | Sentry account |
| PostHog Cloud | https://app.posthog.com | Analytics, session replay | PostHog account |
| Langfuse Cloud | https://cloud.langfuse.com | Observabilidade de IA | Langfuse account |
| CloudAMQP | https://customer.cloudamqp.com | RabbitMQ, DLQ, consumers | CloudAMQP account |
| Upstash Console | https://console.upstash.com | Redis stats, keys | Upstash account |
| Vercel Dashboard | https://vercel.com/dashboard | Frontend deploys, rollback | Vercel account |
| Status Supabase | https://status.supabase.com | Incidentes Supabase | Público |
| Status Celcoin | https://status.celcoin.com.br | Incidentes Celcoin | Público |
| Status Resend | https://status.resend.com | Incidentes Resend | Público |
| Status CloudAMQP | https://status.cloudamqp.com | Incidentes CloudAMQP | Público |
| Status Upstash | https://status.upstash.com | Incidentes Upstash | Público |
| Railway CLI | `railway rollback` | Rollback de backend | `RAILWAY_TOKEN` |
| Vercel CLI | `vercel rollback` | Rollback de frontend | `VERCEL_TOKEN` |
| Redis CLI | `redis-cli -u $UPSTASH_REDIS_URL` | Diagnóstico Redis | URL Redis (Upstash) |

---

## 11. Comunicação de Incidente

### 11.1 Templates por Momento

**Abertura de incidente (postar imediatamente ao identificar P0/P1):**

```
🔴 [INCIDENTE P0/P1 ABERTO] [hora BRT]
Produto: Repasse Seguro — Módulo Cessionário
Sintoma: [descrição do sintoma]
Impacto: [o que está afetado; estimativa de usuários]
Status: Investigando
Owner: @[nome]
Próxima atualização em: 15min
```

**Atualização durante incidente (a cada 15min para P0, 30min para P1):**

```
🟡 [INCIDENTE P0/P1 ATUALIZAÇÃO] [hora BRT]
Status: [Em investigação / Contenção em andamento / Resolução em andamento]
Descoberta: [o que foi identificado]
Ação atual: [o que está sendo feito]
Estimativa: [hora estimada de resolução]
```

**Encerramento:**

```
✅ [INCIDENTE RESOLVIDO] [hora BRT]
Duração: [tempo total]
Causa raiz: [descrição breve]
Solução: [o que foi feito]
Usuários afetados: [número ou estimativa]
Post-mortem: [link ou "em elaboração até [data]"]
```

### 11.2 Canais por Severidade

| Severidade | Canal | Frequência de atualização |
|---|---|---|
| P0 | `#incidentes` + PagerDuty | A cada 15 minutos |
| P1 | `#incidentes` | A cada 30 minutos |
| P2 | `#alertas` | Ao identificar e ao resolver |
| P3 | `#alertas` ou issue | Apenas ao resolver |

---

## 12. Template de Post-Mortem

```markdown
# Post-Mortem: [Título do Incidente]

## Metadados
- **Data do incidente:** YYYY-MM-DD HH:MM BRT
- **Duração:** X horas Y minutos
- **Severidade:** P0 / P1
- **Impacto:** [Número de usuários; funcionalidades afetadas]
- **Owner:** [Nome]
- **Status do post-mortem:** Rascunho / Em revisão / Finalizado

## Timeline

| Hora (BRT) | Evento |
|---|---|
| HH:MM | Primeiro alerta disparado / Primeiro relato de usuário |
| HH:MM | Incidente reconhecido por [nome] |
| HH:MM | Causa raiz identificada |
| HH:MM | Contenção iniciada |
| HH:MM | Serviço restaurado |
| HH:MM | Incidente encerrado |

## Causa Raiz

[Descrição técnica da causa raiz. Ser específico: qual código, qual query, qual configuração.]

## Por que não foi detectado antes?

[O que deveria ter alertado antes e por que não alertou?]

## Mitigação Aplicada

[O que foi feito para resolver o incidente durante a crise.]

## Impacto

- **Usuários afetados:** [número]
- **Transações impactadas:** [número e tipo: propostas, Escrow, KYC]
- **Tempo de degradação:** [X minutos de indisponibilidade / Y minutos de degradação]
- **Dados perdidos:** Sim / Não. Se sim: [descrição]

## Ações Corretivas

| Ação | Owner | Prazo | Status |
|---|---|---|---|
| [Ação preventiva 1] | [nome] | [data] | Pendente |
| [Ação preventiva 2] | [nome] | [data] | Pendente |
| [Alerta novo/melhorado] | [nome] | [data] | Pendente |
| [Teste adicionado] | [nome] | [data] | Pendente |

## O que foi bem durante o incidente?

[Aspectos positivos da resposta.]

## O que pode melhorar?

[Aspectos da resposta que podem ser melhorados.]
```

---

## 13. Backlog de Pendências

| ID | Item | Tipo | Prioridade |
|---|---|---|---|
| RUNBOOK-001 | Adicionar URLs do painel de administração ZapSign e idwall na seção de ferramentas críticas | [DADO PENDENTE] | Alta |
| RUNBOOK-002 | Configurar PagerDuty para alertas P0 (NOT-CES-05/06, API down, Escrow falha) | Ação técnica | Alta |
| RUNBOOK-003 | Criar modo de manutenção na API (retornar 503 com mensagem customizada para janelas planejadas) | Ação técnica | Média |
| RUNBOOK-004 | Definir contato de suporte técnico de cada integração (ZapSign, idwall, Celcoin) com SLA de resposta contratual | [DADO PENDENTE] | Alta |
| RUNBOOK-005 | Documentar procedimento de reconciliação financeira Celcoin após restore de banco | Ação técnica | Alta |

---

*Documento gerado pelo pipeline ShiftLabs v9.5 — Fase 2. Executor: Claude Code Desktop.*
