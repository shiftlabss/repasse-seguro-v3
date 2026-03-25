# 26 - Runbook Operacional — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| **Nome do Documento** | Runbook Operacional — AI-Dani-Cedente |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Bloco** | 6 — Qualidade |
| **Dependências** | D05, D14, D20, D24, D25 |

---

> **📌 TL;DR**
>
> - **4 severidades:** P0 (crítico, SLA 15min) → P1 (alto, 1h) → P2 (médio, 4h) → P3 (baixo, 24h).
> - **Rollback:** `railway rollback --deployment <id>` — < 2 minutos. Critério: taxa de erro > 5% ou health check falhando.
> - **Circuit breaker:** abre em > 10% erros/15min. Reativação: `POST /admin/fallback/enable` após confirmar causa raiz.
> - **Canal de comunicação:** #ops-incidents para P0/P1, #ops-alerts para P2, atualizar a cada 30min em P0.
> - **Post-mortem:** obrigatório para todo P0 e P1 com downtime > 15min. Entrega em 48h.
> - **Checklist diário:** health check + DLQ + CSAT + circuit breaker status.

---

## 1. Classificação de Severidade

| Severidade | Definição | SLA Resposta | SLA Resolução | Impacto Típico | Canal | Escalation |
|---|---|---|---|---|---|---|
| **P0 — Crítico** | Serviço indisponível ou dados de Cedente em risco | 15 min | 2h | Todos os Cedentes impactados | PagerDuty + #ops-incidents | Tech Lead imediato |
| **P1 — Alto** | Degradação severa de funcionalidade crítica | 1h | 4h | Funcionalidade crítica parcialmente indisponível | Slack #ops-incidents | Tech Lead em 30min |
| **P2 — Médio** | Degradação de funcionalidade não-crítica | 4h | 24h | Experiência degradada, workaround disponível | Slack #ops-alerts | Tech Lead em 2h |
| **P3 — Baixo** | Anomalia monitorada, sem impacto direto ao usuário | 24h | 72h | Métrica fora do threshold, sem impacto imediato | Slack #ops-alerts | Ticket para próxima sprint |

**Canal de on-call:** [PENDÊNCIA: definir on-call schedule e contatos de escalation para fora do horário comercial]

---

## 2. Triagem Inicial

Execute este checklist SEMPRE que receber um alerta ou relato de incidente:

1. **Confirmar impacto:** `GET https://dani-cedente.railway.app/health/ready` — retornou 200?
2. **Verificar deploy recente:** há deploy nas últimas 2h? Consultar Railway deploy history.
3. **Isolar escopo:** afeta todos os Cedentes ou apenas um subconjunto? Checar correlation_id se disponível.
4. **Verificar circuit breaker:** `GET /admin/fallback/status` — aberto ou fechado?
5. **Consultar dashboards D25:**
   - Dashboard "Saúde Geral" — taxa de erro e latência p95.
   - Dashboard "Agente IA" — taxa de erro LLM.
   - Dashboard "Infraestrutura" — CPU, memória, DLQ.
6. **Decidir contenção:**
   - Regressão por deploy? → Executar rollback (seção 5).
   - Falha de integração externa? → Verificar status das dependências (seção 4.4).
   - Problema de dados? → Verificar banco e logs com correlation_id.

---

## 3. Catálogo de Alertas

### RB-001 — Circuit Breaker Aberto

| Atributo | Valor |
|---|---|
| **Condição** | `dani_circuit_breaker_state = 1` |
| **Severidade** | P0 |
| **Canal** | PagerDuty + #ops-incidents |
| **Sintomas** | Dani responde com mensagem de indisponibilidade. Cedentes não conseguem chat. |
| **Hipóteses** | (1) Taxa de erro OpenAI > 30% em 15min; (2) Bug pós-deploy; (3) OpenAI API down. |

**Diagnóstico:**
```bash
# 1. Verificar status do circuit breaker
curl -H "Authorization: Bearer $ADMIN_JWT" https://dani-cedente.railway.app/api/v1/admin/fallback/status

# 2. Verificar taxa de erro LLM (Grafana ou logs)
# Dashboard "Saúde Geral" → gauge "Taxa de erro LLM"

# 3. Verificar status da OpenAI API
curl https://status.openai.com/api/v2/status.json | jq '.status.description'

# 4. Verificar logs do serviço (últimos 100 erros)
railway logs --service dani-cedente-prod --tail 100 | grep '"level":"error"'
```

**Ação imediata:**
1. Se OpenAI API down → aguardar restauração da OpenAI. Circuit breaker fechará automaticamente ao atingir threshold de recuperação.
2. Se bug pós-deploy → executar rollback (seção 5).
3. Confirmar causa raiz antes de reativar manualmente.

**Reativação manual (somente após causa raiz confirmada):**
```bash
curl -X POST \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -H "Content-Type: application/json" \
  https://dani-cedente.railway.app/api/v1/admin/fallback/enable
```

**Encerramento:** `dani_circuit_breaker_state = 0` + taxa de erro < 5% por 5min + health check OK.

---

### RB-002 — Latência Chat p95 > 5s

| Atributo | Valor |
|---|---|
| **Condição** | p95 `dani_chat_message_duration_ms` > 5.000ms por 5min |
| **Severidade** | P1 |
| **Canal** | #ops-incidents |
| **Sintomas** | Cedentes reportam resposta lenta. SSE demora > 5s para iniciar. |
| **Hipóteses** | (1) LLM lento (OpenAI degradado); (2) ferramenta LangChain com timeout; (3) RAG lento (pgvector). |

**Diagnóstico:**
```bash
# 1. Verificar latência por step no Langfuse
# Langfuse Dashboard → trace de mensagem recente → qual step demora mais?

# 2. Verificar latência de ferramenta
# Grafana → Dashboard "Agente IA" → widget "Latência por ferramenta p95"

# 3. Verificar latência RAG
# Grafana → `dani_rag_search_duration_ms` p95

# 4. Verificar latência banco
railway logs --service dani-cedente-prod | grep '"module":"prisma"' | tail 50
```

**Ação imediata:**
- Se LLM: verificar status OpenAI, aguardar ou acionar circuit breaker manual.
- Se ferramenta: identificar ferramenta lenta e reduzir timeout temporariamente via env var.
- Se RAG: verificar index pgvector — `SELECT * FROM pg_stat_activity WHERE query LIKE '%knowledge_embeddings%'`.

**Encerramento:** p95 ≤ 5.000ms por 10min consecutivos.

---

### RB-003 — Taxa de Erro LLM > 10%

| Atributo | Valor |
|---|---|
| **Condição** | `dani_llm_error_rate` > 10% em janela de 15min |
| **Severidade** | P1 |
| **Canal** | #ops-incidents |
| **Sintomas** | Respostas de erro DCE-AGENT-503 para Cedentes. Logs com `ExternalServiceError`. |

**Diagnóstico:**
```bash
# Verificar tipo de erro LLM
railway logs --service dani-cedente-prod | grep '"module":"agent"' | grep '"level":"error"' | tail 50

# Verificar status OpenAI
curl https://status.openai.com/api/v2/status.json

# Verificar uso de quota OpenAI (via Langfuse ou OpenAI dashboard)
```

**Ação imediata:**
- < 30%: monitorar. Circuit breaker ainda não abre.
- > 30%: circuit breaker abre automaticamente (RB-001 será acionado).
- Se erro de quota: verificar billing OpenAI — limite mensal atingido?

**Encerramento:** `dani_llm_error_rate` < 2% por 10min.

---

### RB-004 — DLQ com > 10 Mensagens

| Atributo | Valor |
|---|---|
| **Condição** | `rabbitmq_queue_messages{queue="notification.dlq"}` > 10 em 1h |
| **Severidade** | P1 |
| **Canal** | #ops-incidents |
| **Sintomas** | Notificações críticas não chegando ao Cedente. Tracking com status FAILED. |

**Diagnóstico:**
```bash
# Inspecionar mensagens na DLQ (via RabbitMQ Management UI)
# URL: http://localhost:15672 (local) / Railway addon (staging/prod)
# Fila: notification.dlq
# Verificar: failure_reason no header da mensagem

# Verificar logs do NotificationWorker
railway logs --service dani-cedente-prod | grep '"module":"notification"' | grep '"level":"error"'
```

**Ação imediata:**
1. Identificar `failure_reason` nas mensagens da DLQ.
2. Se falha de provedor de e-mail (SMTP down): aguardar restauração + reprocessar.
3. Se bug no worker: fazer hotfix + reprocessar manualmente.
4. **Reprocessamento manual:**
```bash
# Mover mensagens da DLQ de volta para a fila principal
curl -u guest:guest -X POST http://localhost:15672/api/queues/%2F/notification.dlq/get \
  -H "Content-Type: application/json" \
  -d '{"count": 50, "ackmode": "ack_requeue_true", "encoding": "auto"}'
```

**Encerramento:** DLQ = 0 + taxa de entrega ≥ 95% por 30min.

---

### RB-005 — Webhook ZapSign com HMAC Inválido

| Atributo | Valor |
|---|---|
| **Condição** | > 3 requisições com HMAC inválido em 10min |
| **Severidade** | P1 (segurança) |
| **Canal** | #ops-incidents + #ops-security |
| **Sintomas** | Logs com `hmac_valid: false`. Possível tentativa de forjar eventos de Escrow. |

**Diagnóstico:**
```bash
# Verificar IPs das requisições com HMAC inválido
railway logs --service dani-cedente-prod | grep '"hmac_valid":false' | tail 20

# Verificar se é misconfiguration da ZapSign (chave rotacionada?)
# Consultar ZapSign Dashboard → configurações de webhook
```

**Ação imediata:**
1. Se IP suspeito não-ZapSign: bloquear IP temporariamente (Railway WAF ou Cloudflare).
2. Se chave ZapSign rotacionada: atualizar `ZAPSIGN_WEBHOOK_SECRET` via Railway Secrets.
3. Comunicar no #ops-security com correlation_id das requisições suspeitas.

**Encerramento:** Nenhuma requisição com HMAC inválido nos últimos 30min.

---

### RB-006 — Rollback de Deploy

**Ver seção 5 — Rollback.**

---

### RB-007 — CSAT Médio < 3.5

| Atributo | Valor |
|---|---|
| **Condição** | `dani_csat_score_avg` < 3.5 por 24h |
| **Severidade** | P2 |
| **Canal** | #ops-alerts |
| **Sintomas** | Avaliações baixas de Cedentes. Taxa de resolução < 70%. |

**Diagnóstico:**
- Acessar PostHog → filtrar sessões com CSAT < 3 nas últimas 24h.
- Identificar padrão: qual tipo de pergunta está com baixa resolução?
- Verificar se houve mudança no system prompt recentemente.

**Ação imediata:**
- Revisar últimas 10 sessões com CSAT < 3 no Langfuse (traces LLM completos).
- Identificar se é problema de entendimento, de knowledge base RAG ou de tom de resposta.
- Criar ticket P2 para revisão de system prompt ou atualização da base RAG.

**Encerramento:** CSAT médio ≥ 4.0 por 24h.

---

### RB-008 — Redis Indisponível

| Atributo | Valor |
|---|---|
| **Condição** | `redis_connected_clients` = 0 ou timeout de conexão |
| **Severidade** | P0 |
| **Canal** | #ops-incidents |
| **Sintomas** | Sessões de agente não persistidas. Rate limiting não funcionando. Circuit breaker estado perdido. |

**Diagnóstico:**
```bash
# Verificar status Redis
redis-cli -u $REDIS_URL ping

# Verificar logs de conexão
railway logs --service dani-cedente-prod | grep '"module":"redis"' | tail 20
```

**Ação imediata:**
1. Verificar se Railway Redis addon está online.
2. Se downtime do Railway Redis: abrir ticket com Railway Support.
3. Servico continua funcionando sem rate limiting e sem sessão persistida (degradação).
4. Comunicar Cedentes se downtime > 15min.

**Encerramento:** `redis-cli ping` retorna PONG + sessões novas criando corretamente.

---

### RB-009 — Memória > 700MB

| Atributo | Valor |
|---|---|
| **Condição** | `process_resident_memory_bytes` > 700MB |
| **Severidade** | P2 |
| **Canal** | #ops-alerts |
| **Sintomas** | Uso de memória crescente sem queda. Risco de OOM. |

**Diagnóstico:**
```bash
# Verificar tendência de memória (Grafana)
# Se crescimento linear → leak de memória no processo Node.js

# Reiniciar instância como mitigação temporária
railway restart --service dani-cedente-prod
```

**Encerramento:** Memória < 400MB após restart. Abrir ticket P2 para investigação de leak.

---

### RB-010 — Banco de Dados Lento

| Atributo | Valor |
|---|---|
| **Condição** | `pg_stat_activity_count` > 40 ou queries com duração > 5s |
| **Severidade** | P1 |
| **Canal** | #ops-incidents |
| **Sintomas** | Latência de API elevada. Timeouts de Prisma. |

**Diagnóstico:**
```sql
-- Queries lentas em execução
SELECT pid, now() - query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - query_start > interval '2 seconds'
ORDER BY duration DESC;

-- Locks
SELECT * FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE granted = false;
```

**Ação imediata:**
1. Identificar query lenta e matar se necessário: `SELECT pg_terminate_backend(<pid>)`.
2. Verificar se há migration em execução (pode gerar lock).
3. Se problema estrutural: escalar para Railway Supabase support.

---

## 4. Diagnóstico por Camada

### 4.1 Aplicação NestJS

```bash
# Health check completo
curl https://dani-cedente.railway.app/health/ready | jq

# Últimos erros (últimos 5 minutos)
railway logs --service dani-cedente-prod --tail 200 \
  | grep '"level":"error"' \
  | jq '{timestamp, module, message, error_code: .error.code, correlation_id}'

# Verificar versão deployada
curl https://dani-cedente.railway.app/health/ready | jq '.version'
```

**Interpretação:**
- `status: "ok"` em todos os checks → aplicação saudável.
- `status: "error"` em `database` → problema de banco.
- `status: "error"` em `redis` → problema de cache.
- `status: "error"` em `rabbitmq` → problema de fila.

### 4.2 Banco de Dados (Supabase PostgreSQL)

```bash
# Conexões ativas
psql $DATABASE_URL -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# Verificar RLS ativa nas tabelas críticas
psql $DATABASE_URL -c "SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname='public';"

# Verificar tamanho das tabelas principais
psql $DATABASE_URL -c "
SELECT relname, pg_size_pretty(pg_total_relation_size(oid))
FROM pg_class WHERE relkind='r' AND relname IN
('chat_sessions','chat_messages','knowledge_embeddings','notification_log')
ORDER BY pg_total_relation_size(oid) DESC;"
```

### 4.3 Cache Redis

```bash
# Verificar chaves de sessão ativas
redis-cli -u $REDIS_URL KEYS "dani:agent_state:*" | wc -l

# Verificar rate limit de um Cedente específico
redis-cli -u $REDIS_URL GET "dani:chat_rate:ced_001-..."

# Verificar estado do circuit breaker
redis-cli -u $REDIS_URL GET "dani:circuit_breaker:state"

# Memória usada
redis-cli -u $REDIS_URL INFO memory | grep used_memory_human
```

### 4.4 Integrações Externas

| Integração | Comando de Verificação | Status Esperado |
|---|---|---|
| OpenAI API | `curl https://status.openai.com/api/v2/status.json \| jq '.status.description'` | `"All Systems Operational"` |
| Supabase | `curl $SUPABASE_URL/rest/v1/ -H "apikey: $SUPABASE_ANON_KEY"` | HTTP 200 |
| ZapSign | Consultar https://status.zapsign.com.br | N/A |
| Langfuse | `curl https://cloud.langfuse.com/api/public/health` | `{"status":"ok"}` |

### 4.5 RabbitMQ

```bash
# Via RabbitMQ Management API (local ou addon staging)
curl -u guest:guest http://localhost:15672/api/queues/%2F

# Tamanho das filas principais
curl -u $RABBITMQ_USER:$RABBITMQ_PASS \
  $RABBITMQ_MANAGEMENT_URL/api/queues/%2F/notification.email | jq '.messages'

# DLQ
curl -u $RABBITMQ_USER:$RABBITMQ_PASS \
  $RABBITMQ_MANAGEMENT_URL/api/queues/%2F/notification.dlq | jq '.messages'
```

---

## 5. Rollback

**Quando acionar:**
- Health check `/health/ready` falhando após deploy.
- Taxa de erro > 5% nos primeiros 10 minutos pós-deploy.
- Circuit breaker abrindo nos primeiros 5 minutos pós-deploy.
- Funcionalidade crítica quebrada (isolamento de Cedente, confirmação financeira).

**Quem pode acionar:** Tech Lead ou qualquer membro de on-call (sem aprovação adicional em emergência).

**Passos:**
```bash
# 1. Listar deployments recentes
railway deployments list --service dani-cedente-prod

# 2. Identificar deployment anterior saudável
# Procurar o deployment com status "success" antes do atual

# 3. Executar rollback
railway rollback --deployment <deployment-id-anterior>

# 4. Aguardar e verificar
sleep 30
curl -f https://dani-cedente.railway.app/health/ready && echo "ROLLBACK OK"

# 5. Verificar taxa de erro (aguardar 5min)
# Grafana → Dashboard "Saúde Geral" → gauge "Taxa de erro"

# 6. Comunicar encerramento
# Template no #ops-incidents (ver seção 11)
```

**Tempo esperado:** < 2 minutos.

**Verificações pós-rollback:**
- `GET /health/ready` retorna 200.
- Circuit breaker fechado.
- Taxa de erro < 1% por 5min.
- Nenhuma mensagem nova na DLQ.
- Smoke test manual de chat funcionando.

---

## 6. Restore e Recuperação de Dados

### 6.1 Backup

[DECISÃO AUTÔNOMA]: Backups gerenciados pelo Supabase (Point-in-Time Recovery — PITR) — Justificativa: Supabase oferece PITR nativo com retenção de 7 dias nos planos Pro+, sem necessidade de configuração adicional | Alternativa descartada: backup manual via pg_dump — mais trabalhoso e sujeito a erro humano na restauração.

| Tipo | Frequência | Retenção | Localização |
|---|---|---|---|
| PITR (Point-in-Time Recovery) | Contínuo (Supabase) | 7 dias | Supabase Dashboard |
| Snapshot diário | 00:00 UTC | 30 dias | Supabase Dashboard |
| Pre-migration backup | Antes de cada migration | 7 dias | Supabase Dashboard |

### 6.2 Restore de Dados

> 🔴 **Restore em produção é uma operação irreversível.** Requer aprovação explícita do Tech Lead e comunicação a stakeholders antes de executar.

**Pré-condições:**
1. Confirmar escopo do problema (quais tabelas, qual janela de tempo).
2. Estimar impacto: quantos Cedentes afetados, quais dados corrompidos.
3. Comunicar Tech Lead + stakeholders antes de iniciar.

**Passos:**
1. Acessar Supabase Dashboard → Project Settings → Backups.
2. Identificar ponto de restore (PITR ou snapshot).
3. Iniciar restore pelo dashboard (Supabase faz fork do banco em um novo endpoint).
4. Validar integridade dos dados no banco restaurado ANTES de apontar a aplicação.
5. Atualizar `DATABASE_URL` no Railway para o banco restaurado.
6. Executar `pnpm prisma migrate deploy` se necessário.
7. Reiniciar o serviço.

**Tempo estimado:** 30–60 minutos (depende do tamanho do banco).

**Validação de integridade:**
```sql
-- Verificar contagem de registros chave
SELECT 'cedente_profiles' as table_name, count(*) FROM cedente_profiles
UNION ALL SELECT 'opportunities', count(*) FROM opportunities
UNION ALL SELECT 'proposals', count(*) FROM proposals
UNION ALL SELECT 'chat_sessions', count(*) FROM chat_sessions;
```

---

## 7. Cenários de Incidente

### Cenário 1 — Indisponibilidade Total do Serviço

**Sintoma:** `GET /health/ready` retorna 5xx ou timeout. Todos os Cedentes afetados.

**Triagem:**
1. Verificar Railway Dashboard — pod rodando?
2. Verificar últimos logs: `railway logs --tail 50`.
3. Deploy recente? → Rollback imediato.
4. Sem deploy recente → verificar dependências (banco, Redis).

**Contenção:** Rollback se causado por deploy. Se dependência externa, aguardar restauração.

**Escalation:** P0 — Tech Lead + PagerDuty imediatamente.

---

### Cenário 2 — Falha de Redis

**Sintoma:** `GET /health/ready` retorna `redis: error`. Rate limiting inoperante. Sessões perdidas.

**Contenção:** O serviço continua operando em modo degradado (sem rate limiting, sessões reiniciam). Comunicar Cedentes se > 15min.

**Resolução:** Verificar Railway Redis addon. Se necessário, recriar addon e atualizar `REDIS_URL`.

---

### Cenário 3 — Fila RabbitMQ Parada

**Sintoma:** Workers não consumindo. Notificações acumulando. DLQ crescendo.

**Contenção:** Notificações em fila são consumidas quando RabbitMQ voltar. Nenhuma perda de mensagem (durable queues).

**Resolução:** Reiniciar RabbitMQ addon no Railway. Verificar se filas e exchanges estão recriados (são durable, mas confirmar).

---

### Cenário 4 — OpenAI API Instável

**Sintoma:** Erros DCE-AGENT-503 em respostas de chat. Taxa de erro LLM > 10%.

**Contenção:** Circuit breaker abrirá automaticamente em > 30% de erros em 15min. Dani exibirá mensagem de indisponibilidade temporária.

**Resolução:** Aguardar estabilização da OpenAI. Reativar circuit breaker manualmente após confirmação de estabilidade.

---

### Cenário 5 — Erro Massivo Pós-Deploy

**Sintoma:** Taxa de erro > 5% nos primeiros 10min após deploy. Alertas em cascata.

**Contenção:** Rollback imediato (ver seção 5).

**Resolução:** Investigar causa raiz nos logs com correlation_id das requisições com erro. Hotfix conforme D23 seção 6.

---

### Cenário 6 — Vazamento de Dados de Cedente (Segurança)

**Sintoma:** Logs indicando acesso cruzado (404 em recursos de outro Cedente retornando 200, ou dados de Cedente A aparecendo em sessão de Cedente B).

**Contenção:**
1. **Imediato:** desligar serviço via `POST /admin/fallback/disable`.
2. Isolar escopo: quantos Cedentes afetados, quais dados expostos.
3. Notificar DPO (LGPD — notificação à ANPD em até 72h se dados sensíveis expostos).

**Escalation:** P0 imediato + notificação ao DPO.

---

## 8. Contingência Manual

| Situação | Contingência | Duração Máxima | Risco Residual |
|---|---|---|---|
| OpenAI API down | Dani exibe mensagem de indisponibilidade. Cedente redirecionado para suporte humano. | Até OpenAI restaurar | Sem assistência da IA durante o período |
| Redis down | Rate limiting desativado. Sessões não persistidas. | < 30min | Cedentes podem exceder rate limit |
| RabbitMQ down | Notificações acumuladas, entregues após restauração. | < 1h | Atraso em notificações críticas |
| ZapSign instável | Admin notificado para acompanhar manualmente assinaturas pendentes | Por release da ZapSign | Atraso no processo de assinatura |

---

## 9. Checklists Recorrentes

### Checklist Diário (5 min — toda manhã)

- [ ] `GET /health/ready` retorna 200 em produção
- [ ] Circuit breaker fechado: `GET /admin/fallback/status`
- [ ] DLQ = 0: verificar `notification.dlq` no RabbitMQ
- [ ] CSAT últimas 24h ≥ 4.0: verificar PostHog dashboard
- [ ] Taxa de erro < 1%: verificar Grafana "Saúde Geral"
- [ ] Sem alertas novos no #ops-alerts ou #ops-incidents desde ontem

### Checklist Semanal (15 min — toda segunda)

- [ ] Verificar uso de quota OpenAI (custo semanal dentro do budget)
- [ ] Verificar tamanho do banco (crescimento normal?)
- [ ] Verificar memória RSS — crescimento estável ou há tendência de aumento?
- [ ] Revisar mensagens de DLQ da semana — padrão identificado?
- [ ] Verificar logs de HMAC inválido do ZapSign — algum IP suspeito?
- [ ] Confirmar CSAT médio da semana ≥ 4.0
- [ ] Verificar se há dependências com update disponível (Dependabot alerts)

### Checklist Pós-Incidente (30 min — após qualquer P0 ou P1)

- [ ] Rollback ou fix aplicado?
- [ ] Health check verde?
- [ ] Taxa de erro < 1% por 30min?
- [ ] DLQ = 0?
- [ ] Cedentes comunicados (se impacto visível)?
- [ ] Correlation_id do incidente registrado?
- [ ] Post-mortem agendado (obrigatório para P0 e P1 com > 15min)?
- [ ] Ticket de ação corretiva criado?

---

## 10. URLs e Ferramentas Críticas

| Recurso | URL / Comando | Uso | Credencial |
|---|---|---|---|
| Health Check Produção | `https://dani-cedente.railway.app/health/ready` | Verificar status do serviço | Pública |
| Admin API | `https://dani-cedente.railway.app/api/v1/admin/*` | Fallback, CSAT, takeover | `ADMIN_JWT` |
| Railway Dashboard | `https://railway.app` | Deploy, logs, restart, rollback | Conta Railway |
| Railway Logs Produção | `railway logs --service dani-cedente-prod` | Logs em tempo real | Railway CLI autenticado |
| Langfuse Dashboard | `https://cloud.langfuse.com` | Traces LLM, custo, debugging | `LANGFUSE_*` env vars |
| PostHog Dashboard | `https://app.posthog.com` | CSAT, métricas produto, feature flags | Conta PostHog |
| Grafana Dashboard | [PENDÊNCIA: URL do Grafana a configurar] | Métricas infra, alertas | Conta Grafana |
| Sentry Dashboard | `https://sentry.io` | Erros de aplicação | Conta Sentry |
| Supabase Dashboard | `https://supabase.com/dashboard` | Banco de dados, backups, PITR | Conta Supabase |
| OpenAI Status | `https://status.openai.com` | Status da API OpenAI | Pública |
| ZapSign Status | `https://status.zapsign.com.br` | Status da API ZapSign | Pública |

---

## 11. Comunicação de Incidente

### Template de Atualização Interna (Slack)

**Abertura:**
```
[INCIDENTE ABERTO] P{SEVERIDADE} — {Título}
🕐 Início: {timestamp}
📍 Escopo: {descrição do impacto}
🔍 Investigando: {hipótese inicial}
👤 Responsável: @{nome}
Correlation ID: {corr_id se disponível}
```

**Atualização (a cada 30min em P0, a cada 1h em P1):**
```
[UPDATE {N}] P{SEVERIDADE} — {Título}
🕐 {timestamp}
📊 Status: {Em investigação | Contido | Em recuperação}
🔍 Root cause: {se identificado}
⏱ ETA resolução: {estimativa}
```

**Encerramento:**
```
[INCIDENTE ENCERRADO] P{SEVERIDADE} — {Título}
🕐 Duração: {tempo total}
✅ Resolução: {descrição da solução}
📈 Impacto: {número de Cedentes, janela de tempo}
🔄 Ações: {link do post-mortem se P0/P1 com > 15min}
```

**Frequência mínima de atualização:**
- P0: a cada 30 minutos.
- P1: a cada 1 hora.
- P2/P3: ao fechar o ticket.

---

## 12. Template de Post-Mortem

```markdown
# Post-Mortem: {Título do Incidente}

**Data:** {data}
**Severidade:** P{N}
**Duração:** {início} → {fim} ({duração total})
**Owner:** @{nome}
**Status:** {Em elaboração | Revisado | Finalizado}

## Timeline

| Horário | Evento |
|---|---|
| {HH:MM} | Alerta disparado: {descrição} |
| {HH:MM} | Investigação iniciada por @{nome} |
| {HH:MM} | Root cause identificado: {descrição} |
| {HH:MM} | Contenção aplicada: {ação} |
| {HH:MM} | Serviço restaurado |
| {HH:MM} | Incidente encerrado |

## Causa Raiz

{Descrição técnica detalhada da causa raiz. Sem atribuição de culpa — foco no sistema.}

## Mitigação Aplicada

{O que foi feito para resolver o incidente no momento: rollback, hotfix, restart, etc.}

## Impacto

- **Cedentes afetados:** {número estimado}
- **Janela de impacto:** {duração}
- **Funcionalidades afetadas:** {lista}
- **Notificações perdidas:** {número, se aplicável}

## O que Funcionou Bem

- {O que ajudou na resposta: alertas rápidos, documentação clara, etc.}

## O que Pode Melhorar

- {O que poderia ter acelerado a resposta ou prevenido o incidente}

## Ações Corretivas

| Ação | Owner | Prazo | Status |
|---|---|---|---|
| {Descrição da ação corretiva} | @{nome} | {data} | Pendente |

## Lições Aprendidas

{Insight para o time evitar recorrência.}
```

---

## 13. Backlog de Pendências

| # | Item | Tipo | Prioridade |
|---|---|---|---|
| P-RBK-001 | Definir on-call schedule e contatos de escalation (PagerDuty ou Slack OnCall) | [SEÇÃO PENDENTE] — requer definição do time | Alta |
| P-RBK-002 | Configurar Grafana com URL de acesso e credenciais para todos os membros do time | [SEÇÃO PENDENTE] — depende do setup de infra (D24) | Alta |
| P-RBK-003 | Criar runbook de restore Supabase com steps detalhados para o ambiente de produção específico | [SEÇÃO PENDENTE] — requer acesso ao Supabase Dashboard de prod | Média |
| P-RBK-004 | Implementar testes de caos (chaos engineering) para validar comportamentos documentados neste runbook | [DECISÃO AUTÔNOMA: adiar para pós go-live] — prioridade após primeiro ciclo de operação | Baixa |

---

## 14. Changelog do Documento

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — 4 severidades, 10 procedimentos de alerta (RB-001 a RB-010), diagnóstico por 5 camadas, rollback em < 2min, 6 cenários de incidente, contingência manual por integração, 3 checklists recorrentes, tabela de ferramentas críticas, template de comunicação e post-mortem. |
