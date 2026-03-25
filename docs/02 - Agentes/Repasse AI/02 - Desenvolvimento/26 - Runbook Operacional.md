# 26 - Runbook Operacional

## Cabeçalho

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
| --- | --- | --- | --- | --- |
| 26 - Runbook Operacional | v1.0 | 2026-03-22 | Claude Code Desktop (ShiftLabs Pipeline v9.5) | Aprovado |

---

## TL;DR

> 📌 **Manual de operação em crise do Repasse AI:**
>
> - **4 severidades:** CRITICAL (P0/15min), HIGH (P1/1h), MEDIUM (P2/4h), LOW (P3/próximo dia útil).
> - **10 alertas catalogados** com diagnóstico, comandos e critério de encerramento.
> - **Rollback** via Railway dashboard/CLI em < 3 minutos — sem aprovação necessária em P0/P1.
> - **Restore** via backup automático Supabase — ponto de recuperação mais recente (diário).
> - **5 cenários de incidente** detalhados: LLM indisponível, banco/RLS, Redis, fila RabbitMQ, webhook WhatsApp.
> - **Contingência manual:** circuit breaker force-open, WhatsApp mock, notificações in-app apenas.
> - **Post-mortem obrigatório** para P0 e P1 — template na seção 14.

---

## 1. Classificação de Severidade

| Severidade | Definição | SLA Resposta | SLA Resolução | Impacto Típico | Canal | Escalation |
| --- | --- | --- | --- | --- | --- | --- |
| **CRITICAL (P0)** | Serviço totalmente indisponível ou perda/exposição de dados | 15 min | 1 hora | Todos os usuários afetados | PagerDuty + Slack #alerts | Tech Lead obrigatório |
| **HIGH (P1)** | Funcionalidade crítica degradada; usuários afetados com workaround possível | 1 hora | 4 horas | Subconjunto de usuários afetados | Slack #alerts + DM on-call | Tech Lead notificado |
| **MEDIUM (P2)** | Funcionalidade não-crítica degradada; impacto visível mas controlado | 4 horas | 24 horas | Experiência degradada | Slack #alerts | Equipe de engenharia |
| **LOW (P3)** | Anomalia detectada, sem impacto imediato | Próximo dia útil | 72 horas | Cosmético ou métricas fora do target | Slack #dev | Developer responsável |

---

## 2. Triagem Inicial

Ao receber qualquer alerta, executar este checklist antes de agir:

1. **Confirmar impacto:** `curl -s https://api.repasseseguro.com.br/health | jq .` — qual serviço está `down`?
2. **Verificar deploy recente:** Railway dashboard → aba Deployments → algum deploy nas últimas 2 horas?
3. **Consultar Sentry:** novos erros CRITICAL/ERROR que não existiam antes?
4. **Verificar status externo:** OpenAI status (`https://status.openai.com`), Supabase status (`https://status.supabase.com`), Upstash Redis (dashboard).
5. **Isolar escopo:** o problema afeta todos os tenants ou um tenant específico?
6. **Decidir contenção:** rollback? circuit breaker manual? escalar para Tech Lead?

> ⚙️ **Regra operacional:** nunca agir sem passar pelo checklist de triagem. Ação sem diagnóstico agrava incidentes.

---

## 3. Catálogo de Alertas

### CB-001: CIRCUIT_BREAKER_OPEN

| Atributo | Valor |
| --- | --- |
| Condição | `circuit_breaker_open_count ≥ 1` |
| Severidade | CRITICAL (P0) |
| Sintoma | Respostas 503 imediatas com `error_code: LLM_UNAVAILABLE`; sem chamadas ao OpenAI |
| Hipóteses | OpenAI com incidente; bug introduzido no AiService; rate limit OpenAI atingido |
| Diagnóstico | `curl https://status.openai.com/api/v2/summary.json` + Langfuse: trace mais recente antes do OPEN |
| Ação imediata | 1. Verificar OpenAI status; 2. Se OpenAI down: aguardar reset automático (5 min); 3. Se OpenAI up: rollback do último deploy |
| Escalation | Tech Lead se não resolver em 15 min |
| Encerramento | `circuit_breaker_open_count = 0` + teste manual de mensagem bem-sucedido |

---

### APP-001: HTTP_ERROR_RATE_5XX

| Atributo | Valor |
| --- | --- |
| Condição | `http_error_rate > 5%` em 5 min |
| Severidade | CRITICAL (P0) |
| Sintoma | Múltiplas respostas 500 para endpoints variados |
| Hipóteses | Deploy com regressão; banco indisponível; OOM no Railway |
| Diagnóstico | Sentry → últimos erros CRITICAL; Railway logs → últimos 100 logs de nível error |
| Ação imediata | 1. Verificar se deploy recente → rollback; 2. Verificar Railway CPU/memória; 3. Verificar Supabase status |
| Escalation | Tech Lead se não houver deploy recente óbvio |
| Encerramento | `http_error_rate < 1%` por 5 min consecutivos |

---

### SEC-001: TENANT_MISMATCH_DETECTED

| Atributo | Valor |
| --- | --- |
| Condição | ≥ 1 evento `TENANT_MISMATCH` |
| Severidade | CRITICAL (P0) |
| Sintoma | Log `TENANT_MISMATCH` com `user_id_hash` e `cessionario_id_hash` distintos do recurso |
| Hipóteses | Bug de isolamento introduzido; token manipulado; falha no RLS |
| Diagnóstico | Supabase Dashboard → Auth logs → usuário suspeito; Railway logs filtrados por `TENANT_MISMATCH` |
| Ação imediata | 1. Identificar usuário via `correlation_id`; 2. Revogar token se suspeito (Supabase Admin); 3. Verificar RLS em todas as tabelas de domínio |
| Escalation | Tech Lead + notificar responsável jurídico (LGPD) se houve acesso a dados |
| Encerramento | Auditoria completa dos logs da janela do incidente + confirmação de que RLS está ativo |

---

### SEC-002: WEBHOOK_HMAC_INVALID

| Atributo | Valor |
| --- | --- |
| Condição | ≥ 3 eventos HMAC inválido em 10 min |
| Severidade | HIGH (P1) |
| Sintoma | Logs `[WA] HMAC inválido` com IPs distintos |
| Hipóteses | Tentativa de injeção de webhook; IP de EvolutionAPI mudou; rotação de secret pendente |
| Diagnóstico | Railway logs → IP de origem dos requests inválidos; verificar `EVOLUTION_API_WEBHOOK_SECRET` no Railway Secrets |
| Ação imediata | 1. Verificar se IP de origem é da EvolutionAPI legítima; 2. Se desconhecido: bloquear IP no Railway; 3. Rotacionar `EVOLUTION_API_WEBHOOK_SECRET` se comprometido |
| Encerramento | Sem novos eventos HMAC inválido por 30 min após ação |

---

### NOTIF-001: DLQ_MESSAGES_CRITICAL

| Atributo | Valor |
| --- | --- |
| Condição | `rabbitmq_dlq_count ≥ 10` |
| Severidade | HIGH (P1) |
| Sintoma | Notificações não entregues; RabbitMQ Management mostra mensagens em DLQ |
| Hipóteses | Worker de notificação parado; EvolutionAPI indisponível; push subscription inválida |
| Diagnóstico | RabbitMQ Management → `notif.{canal}.dlq` → inspecionar body das mensagens |
| Ação imediata | 1. Verificar worker ativo (Railway service logs); 2. Inspecionar 3 mensagens da DLQ para padrão de erro; 3. Se EvolutionAPI down: aguardar + reprocessar DLQ manualmente |
| Reprocessamento | `rabbitmq-cli move-messages --from notif.whatsapp.dlq --to notif.whatsapp` |
| Encerramento | `rabbitmq_dlq_count = 0` + notificação de teste entregue |

---

### DB-001: DB_CONNECTION_POOL_CRITICAL

| Atributo | Valor |
| --- | --- |
| Condição | `db_connection_pool_usage > 95%` |
| Severidade | CRITICAL (P0) |
| Sintoma | Queries bloqueadas; timeout de banco; respostas 503 com `error_code: INTERNAL_ERROR` |
| Diagnóstico | `railway run pnpm prisma studio` → ver conexões ativas; Supabase dashboard → Connection Pooling |
| Ação imediata | 1. Railway restart do serviço (fecha conexões ociosas); 2. Supabase: verificar conexões abertas e matar ociosas |
| Encerramento | `db_connection_pool_usage < 70%` por 5 min |

---

### COST-001: LLM_COST_DAILY_HIGH

| Atributo | Valor |
| --- | --- |
| Condição | `llm_cost_usd_per_day > $100` |
| Severidade | HIGH (P1) |
| Sintoma | Custo OpenAI acima do esperado no Langfuse ou fatura OpenAI |
| Hipóteses | Volume anômalo de requests; bug de loop; cache semântico hit rate baixo |
| Diagnóstico | Langfuse → Cost by trace; verificar `semantic_cache_hit_rate` no dashboard |
| Ação imediata | 1. Identificar tenant com maior volume; 2. Verificar se há loop de requests; 3. Forçar rate limit mais restritivo via Redis se necessário |
| Encerramento | Custo projetado voltando ao target; causa raiz identificada |

---

### CB-002: LLM_ERROR_RATE_HIGH

| Atributo | Valor |
| --- | --- |
| Condição | `llm_error_rate > 10%` em 5 min |
| Severidade | CRITICAL (P0) |
| Sintoma | Alto volume de erros `LLM_UNAVAILABLE` no Sentry; circuit breaker prestes a abrir |
| Diagnóstico | OpenAI status page; Langfuse → traces com erro; verificar model disponível |
| Ação imediata | 1. Verificar OpenAI status; 2. Se intermitente: aumentar max_retries para 5 via Railway env var temporária; 3. Se circuit breaker abrir: seguir CB-001 |
| Encerramento | `llm_error_rate < 2%` por 10 min |

---

### RAG-001: RAG_RETRIEVAL_ERROR_RATE_HIGH

| Atributo | Valor |
| --- | --- |
| Condição | ≥ 10 eventos `RAG_RETRIEVAL_ERROR` em 5 min |
| Severidade | HIGH (P1) |
| Sintoma | Agente respondendo sem contexto RAG; logs `[AI] RAG retrieval falhou` |
| Diagnóstico | Supabase status; query direta ao pgvector: `railway run pnpm prisma db execute --stdin <<< "SELECT count(*) FROM knowledge_chunks;"` |
| Ação imediata | 1. Verificar Supabase status; 2. Se Supabase OK: verificar extensão pgvector ativa; 3. Agente continua respondendo (fallback sem RAG) — comunicar degradação |
| Encerramento | `RAG_RETRIEVAL_ERROR` ausente por 10 min + teste de retrieval bem-sucedido |

---

### APP-002: SERVICE_HEALTH_DOWN

| Atributo | Valor |
| --- | --- |
| Condição | `GET /health` retorna 503 por > 30s |
| Severidade | CRITICAL (P0) |
| Sintoma | Railway mostra serviço down; health endpoint não responde |
| Diagnóstico | Railway dashboard → Deployments → ver status + logs do último deployment |
| Ação imediata | Rollback imediato (ver seção 5) |
| Encerramento | `GET /health` retorna 200 por 2 min consecutivos |

---

## 4. Diagnóstico por Camada

### 4.1 Aplicação (NestJS)

```bash
# Ver logs em tempo real (Railway CLI)
railway logs --tail 100 --service repasse-ai

# Filtrar por error_code específico
railway logs --service repasse-ai | grep "LLM_UNAVAILABLE"

# Verificar uso de memória
railway status --service repasse-ai

# Health check manual
curl -s https://api.repasseseguro.com.br/health | jq .
```

**Sinais:** status de cada módulo no `/health`; erros CRITICAL no Sentry; `http_error_rate` no dashboard.

### 4.2 Banco de Dados (Supabase PostgreSQL)

```bash
# Verificar conexões ativas via Prisma
railway run pnpm prisma db execute --stdin <<< "SELECT count(*) FROM pg_stat_activity WHERE state = 'active';"

# Verificar RLS ativo nas tabelas principais
railway run pnpm prisma db execute --stdin <<< "SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public';"

# Query lenta — ver via Supabase Dashboard → Database → Query Performance
```

**Sinais:** `db_connection_pool_usage > 85%`; queries com `duration_ms > 800ms` nos logs; Supabase status page.

### 4.3 Redis (Upstash)

```bash
# Verificar conectividade
railway run node -e "const Redis = require('ioredis'); const r = new Redis(process.env.REDIS_URL); r.ping().then(console.log).catch(console.error);"

# Ver uso de memória no Upstash Dashboard
# Upstash Dashboard → database → memory usage
```

**Sinais:** `rate limit` não funcionando (Redis down com fail-open); cache hit rate = 0%; OTP não expirando.

### 4.4 Filas (RabbitMQ CloudAMQP)

```bash
# RabbitMQ Management UI (CloudAMQP)
# Acessar: https://app.cloudamqp.com → instância → Management UI
# Credenciais: RABBITMQ_URL no Railway Secrets

# Ver queues e DLQs
# Management → Queues tab → buscar notif.*.dlq
```

**Sinais:** `rabbitmq_queue_depth > 500`; DLQ acumulando; workers parados (Railway service logs).

### 4.5 Integrações Externas

```bash
# OpenAI — verificar status
curl https://status.openai.com/api/v2/summary.json | jq '.components[] | select(.name | contains("API"))'

# EvolutionAPI — verificar instância
curl -H "apikey: $EVOLUTION_API_KEY" https://<evolution-host>/instance/fetchInstances

# Supabase
curl https://status.supabase.com/api/v2/summary.json | jq '.components[0]'
```

---

## 5. Rollback

Ver procedimento completo no Doc 24 (seção 7). Resumo rápido:

### 5.1 Via Railway Dashboard (preferido)

1. Abrir `railway.app` → projeto `repasse-ai` → serviço `ai`
2. Aba "Deployments" → localizar deployment anterior ao problemático
3. Clicar "Redeploy"
4. Aguardar ~3 min
5. Verificar `GET /health` → 200
6. Notificar #deploys

### 5.2 Via Railway CLI

```bash
railway login
# Listar deployments
railway deployments --service repasse-ai
# Copiar DEPLOYMENT_ID do deployment anterior ao problemático
railway redeploy --deployment <DEPLOYMENT_ID>
# Aguardar
watch -n 5 curl -s https://api.repasseseguro.com.br/health
```

### 5.3 Critério de Acionamento

- `http_error_rate > 10%` por > 3 min após deploy
- `GET /health` → 503 por > 30s
- Circuit breaker abrindo nos primeiros 5 min de um novo deploy (se não estava aberto antes)

---

## 6. Restore e Recuperação de Dados

### 6.1 Backup Supabase

- **Automático:** Supabase faz backup diário (plano Pro). Retenção: 7 dias.
- **Ponto de recuperação:** último backup disponível (máximo 24h de perda de dados).
- **Acesso:** Supabase Dashboard → Settings → Backups.

### 6.2 Procedimento de Restore

> 🔴 **Atenção:** restore apaga dados escritos após o ponto de recuperação. Nunca restaurar sem aprovação do Tech Lead e comunicação prévia.

1. Identificar janela do incidente (timestamp de início do problema).
2. Supabase Dashboard → Settings → Backups → selecionar backup anterior ao incidente.
3. Clicar "Restore" → confirmar.
4. Aguardar restore (~15-30 min dependendo do tamanho do banco).
5. Verificar integridade: `railway run pnpm prisma db execute --stdin <<< "SELECT count(*) FROM conversations;"` — comparar com contagem esperada.
6. Reiniciar serviço Railway para reconectar pool de conexões.
7. Comunicar stakeholders sobre perda de dados na janela de incidente.

### 6.3 Impacto de Restore

- Mensagens enviadas entre o backup e o incidente serão perdidas.
- Notificar usuários afetados via WhatsApp (canal crítico, não pode ser desativado).
- Registrar no post-mortem o volume de dados perdidos.

---

## 7. Cenários de Incidente

### Cenário 1: LLM Indisponível (OpenAI down)

**Sintoma:** Circuit breaker em OPEN; todas as respostas do agente retornam 503; Langfuse sem traces novos.

**Triagem:** OpenAI status page → incidente ativo? Sim → aguardar.

**Contenção:**
1. Circuit breaker já está protegendo o sistema (sem chamadas ao OpenAI).
2. In-App notification para usuários ativos: "O assistente está temporariamente indisponível."
3. Não fazer rollback — o problema é externo.

**Recuperação:** OpenAI resolve incidente → circuit breaker faz reset automático em 5 min → testar mensagem manualmente → confirmar normalidade.

**Comunicação:** "Indisponibilidade do assistente IA devido a incidente externo (OpenAI). Estimativa de resolução: conforme status.openai.com. Funcionalidades de chat e WhatsApp continuam ativas."

---

### Cenário 2: Banco / RLS com Problema

**Sintoma:** Respostas 500 em múltiplos endpoints; `INTERNAL_ERROR` no Sentry com stack trace apontando para Prisma.

**Triagem:** Supabase status; `db_connection_pool_usage`; ver logs de query lenta.

**Contenção:**
1. Se Supabase down: aguardar recuperação; agente IA não funciona sem banco.
2. Se RLS bug: rollback imediato do último deploy.
3. Se pool esgotado: Railway restart do serviço.

---

### Cenário 3: Redis Indisponível

**Sintoma:** Rate limiting não funcionando; OTPs não expiram; cache semântico com 0% hit rate; possíveis erros `EXTERNAL_SERVICE_ERROR`.

**Comportamento esperado:** Redis configurado com fail-open para rate limiting → não bloqueia requests, mas perde proteção.

**Contenção:**
1. Verificar Upstash status.
2. Se Redis down: aumentar atenção ao volume de requests (rate limit desativado temporariamente).
3. Não há rollback — problema é externo. Aguardar Upstash recuperar.
4. OTPs emitidos durante a outage não expirarão automaticamente — invalida manualmente via Admin API se necessário.

---

### Cenário 4: Fila RabbitMQ Parada

**Sintoma:** Notificações não entregues; `rabbitmq_dlq_count` crescendo; Workers sem consumo de mensagens.

**Contenção:**
1. CloudAMQP dashboard → verificar instância ativa.
2. Se CloudAMQP down: mensagens acumulam nas queues até reconexão; não há perda permanente (DLQ as retém).
3. Se worker parado (bug de código): rollback ou restart do serviço Railway.
4. Após reconexão: processar DLQ na ordem de chegada.

---

### Cenário 5: Webhook WhatsApp com HMAC Inválido em Volume

**Sintoma:** Alto volume de eventos `WEBHOOK_HMAC_INVALID`; mensagens do WhatsApp não processadas.

**Contenção:**
1. Verificar IP de origem dos requests no Railway logs.
2. Se IPs desconhecidos: atividade maliciosa → bloquear no Railway → notificar #security.
3. Se IP da EvolutionAPI legítima: secret desincronizado → rotacionar `EVOLUTION_API_WEBHOOK_SECRET`.

**Rotação de secret sem downtime:**
1. Adicionar novo secret como `EVOLUTION_API_WEBHOOK_SECRET_NEW` no Railway.
2. Atualizar código para aceitar ambos temporariamente (período de transição de 5 min).
3. Atualizar EvolutionAPI com o novo secret.
4. Remover `EVOLUTION_API_WEBHOOK_SECRET_OLD` do código e Railway.

---

## 8. Contingência Manual

### 8.1 Circuit Breaker Force-Open (Admin)

Quando OpenAI está em incidente prolongado e se deseja evitar tentativas desnecessárias:

```bash
# Abrir circuit breaker manualmente via Admin API
curl -X POST https://api.repasseseguro.com.br/repasse-ai/v1/admin/circuit-breaker/open \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -H "Content-Type: application/json"
```

Usar quando OpenAI tem incidente confirmado com ETA > 1 hora. Fechar quando OpenAI recuperar:

```bash
curl -X POST https://api.repasseseguro.com.br/repasse-ai/v1/admin/circuit-breaker/close \
  -H "Authorization: Bearer $ADMIN_JWT"
```

### 8.2 WhatsApp Mock Mode

Ativar em emergência quando EvolutionAPI está com incidente e é necessário processar mensagens recebidas via webhook (modo degradado sem envio):

```bash
# Railway → Environment Variables → FEATURE_WHATSAPP_MOCK=true
# Restart do serviço
# Cuidado: mensagens não serão enviadas via WhatsApp — apenas processadas internamente
```

### 8.3 Notificações In-App Only

Quando push ou WhatsApp estão indisponíveis, o sistema automaticamente faz fallback para notificações in-app (SSE). Nenhuma ação manual necessária — comportamento definido no Doc 21.

---

## 9. Checklists Recorrentes

### 9.1 Checklist Diário (5 min, toda manhã)

- [ ] `GET /health` retorna 200 com todos os serviços `up`
- [ ] Sentry: sem novos erros CRITICAL nas últimas 24h
- [ ] `circuit_breaker_open_count = 0`
- [ ] `rabbitmq_dlq_count = 0`
- [ ] Custo OpenAI do dia anterior dentro do target (< $50)
- [ ] Último deployment bem-sucedido (Railway dashboard)

### 9.2 Checklist Semanal (15 min, toda segunda)

- [ ] Revisar alertas HIGH/CRITICAL da semana anterior
- [ ] Verificar `semantic_cache_hit_rate` > 30% (Langfuse)
- [ ] Verificar `notification_delivery_rate` > 95%
- [ ] Revisar top 5 erros do Sentry e verificar se tickets criados
- [ ] Verificar `db_query_p95_ms` — queries lentas novas?
- [ ] Revisar Railway usage: CPU e memória dentro de limites?
- [ ] Rotação de secrets vencida? (checklist de rotação no Doc 22)

### 9.3 Checklist Pós-Incidente (dentro de 24h)

- [ ] Encerramento formal do incidente notificado no Slack #deploys
- [ ] Timeline do incidente documentada no Notion (usar template de post-mortem)
- [ ] Causa raiz identificada (não "investigando")
- [ ] Ação corretiva criada como GitHub Issue com label `post-mortem` e data de entrega
- [ ] Teste de regressão adicionado para o cenário do incidente
- [ ] Runbook atualizado se o procedimento existente foi insuficiente

---

## 10. URLs e Ferramentas Críticas

| Recurso | URL / Comando | Uso | Credencial |
| --- | --- | --- | --- |
| API Health | `https://api.repasseseguro.com.br/health` | Status geral do sistema | Pública |
| Railway Dashboard | `https://railway.app` | Deploy, rollback, logs, env vars | Conta Railway (time) |
| Railway CLI | `railway logs --service repasse-ai` | Logs em tempo real | `railway login` |
| Sentry | `https://sentry.io/organizations/repasse-seguro/` | Error tracking | Conta Sentry (time) |
| Langfuse | `https://cloud.langfuse.com` | Traces LLM, custo, latência | Conta Langfuse (time) |
| Supabase Dashboard | `https://supabase.com/dashboard` | Banco, RLS, backups, auth | Conta Supabase (time) |
| Upstash Dashboard | `https://console.upstash.com` | Redis — memória, hit rate | Conta Upstash (time) |
| CloudAMQP Dashboard | `https://app.cloudamqp.com` | RabbitMQ — queues, DLQ | Conta CloudAMQP (time) |
| OpenAI Status | `https://status.openai.com` | Status do OpenAI API | Pública |
| Supabase Status | `https://status.supabase.com` | Status do Supabase | Pública |
| GitHub Actions | `https://github.com/repo/repasse-ai/actions` | CI/CD status | Conta GitHub (time) |

---

## 11. Comunicação de Incidente

### 11.1 Template de Update Interno (a cada 30 min em P0, a cada 1h em P1)

```
🚨 [P0/P1] Incidente Ativo — Repasse AI
Status: [Investigando / Mitigando / Resolvido]
Impacto: [descrição em 1 linha do que está afetado]
Última ação: [o que foi feito agora]
Próxima atualização: [HH:MM BRT]
Responsável: @nome
```

### 11.2 Mensagem de Encerramento

```
✅ Incidente encerrado — Repasse AI
Duração: X minutos (HH:MM – HH:MM BRT)
Causa raiz: [descrição breve]
Resolução: [o que foi feito]
Post-mortem: [link Notion — dentro de 24h]
```

### 11.3 Donos por Severidade

| Severidade | DRI (Directly Responsible Individual) |
| --- | --- |
| P0 | Tech Lead (obrigatório) + on-call backend |
| P1 | On-call backend; Tech Lead notificado |
| P2 | Developer responsável pelo módulo |
| P3 | Developer que detectou |

---

## 12. Template de Post-Mortem

```markdown
## Post-Mortem — [Nome do Incidente]

**Data:** YYYY-MM-DD
**Severidade:** P0 / P1
**Duração:** X min (HH:MM – HH:MM BRT)
**DRI:** @nome
**Status:** Rascunho / Revisado / Publicado

### Resumo
[1 parágrafo descrevendo o incidente, impacto e resolução]

### Timeline
| HH:MM BRT | Evento |
| --- | --- |
| HH:MM | Alerta disparado no Sentry/PagerDuty |
| HH:MM | On-call investigou |
| HH:MM | Causa raiz identificada |
| HH:MM | Rollback executado |
| HH:MM | Sistema normalizado |

### Causa Raiz
[Descrição técnica precisa — sem suavizar]

### Impacto
- Usuários afetados: [número estimado]
- Duração da degradação: X min
- Dados perdidos: [sim/não — quantos registros]
- SLO impactado: [sim/não — quanto do error budget consumido]

### O Que Funcionou Bem
- [Ex: circuit breaker protegeu contra calls excessivos ao OpenAI]

### O Que Não Funcionou
- [Ex: rollback demorou 8 min por falta de acesso ao Railway CLI]

### Ação Corretiva
| Ação | Owner | Prazo | Issue |
| --- | --- | --- | --- |
| Adicionar teste de regressão para [cenário] | @dev | YYYY-MM-DD | #123 |
| Atualizar runbook seção [X] | @tech-lead | YYYY-MM-DD | #124 |
```

---

## 13. Backlog de Pendências

| ID | Descrição | Prioridade | Observação |
| --- | --- | --- | --- |
| RB-001 | Implementar endpoint `POST /admin/circuit-breaker/open|close` documentado na seção 8.1 | Alta | Necessário para contingência manual |
| RB-002 | Configurar PagerDuty integration com Sentry e Railway alerts | Alta | Necessário antes do go-live |
| RB-003 | Documentar credenciais de acesso às ferramentas críticas em gerenciador de senhas do time | Alta | Railway, Supabase, Sentry, Langfuse |
| RB-004 | Criar procedimento de rotação de `EVOLUTION_API_WEBHOOK_SECRET` sem downtime em runbook separado | Média | Mencionado na seção 7 cenário 5 |

> **Decisões Autônomas Tomadas Neste Documento:**
>
> 1. **[DECISÃO AUTÔNOMA] SLA P0 de 15 min para resposta:** alternativa (5 min) descartada por ser inviável para time pequeno fora do horário comercial. Critério: realidade operacional de early-stage + SLA do OpenAI (99.5%).
> 2. **[DECISÃO AUTÔNOMA] Post-mortem obrigatório apenas para P0/P1:** alternativa (todos os incidentes) descartada por overhead de processo em incidentes de baixo impacto. Critério: custo de processo vs. aprendizado real.
> 3. **[DECISÃO AUTÔNOMA] Restore Supabase com perda máxima de 24h:** backup diário é o padrão do plano Supabase Pro. Alternativa (backup horário com PITR) disponível em planos superiores — avaliar quando produto escalar. Critério: custo vs. RPO aceitável para early-stage.

---

*Próximo documento do pipeline: D28 — Checklist de Qualidade.*
