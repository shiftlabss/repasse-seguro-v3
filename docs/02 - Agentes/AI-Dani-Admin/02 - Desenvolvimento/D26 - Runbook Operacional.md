# Runbook Operacional — AI-Dani-Admin

## Procedimentos Operacionais e Resposta a Incidentes

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend e DevOps |
| Escopo | Procedimentos de resposta a incidentes, troubleshooting, manutenção e operações rotineiras do módulo AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D14 (Especificações Técnicas), D17 (Integrações Externas), D24 (Deploy), D25 (Observabilidade) |

---

> **📌 TL;DR**
>
> - **P0 (serviço fora do ar):** verificar health check → database → Redis → RabbitMQ → rollback se necessário.
> - **Kill switch de IA:** `webchat-enabled` flag no PostHog desabilita tudo em < 30s, sem deploy.
> - **Migrations em produção:** sempre `prisma migrate deploy` — nunca `migrate dev`. Migrations destrutivas exigem migration de rollback escrita antes.
> - **Takeover travado:** `UPDATE interactions SET status = 'RESPONDIDA_PELA_IA' WHERE id = $1` + `DELETE FROM takeovers WHERE interaction_id = $1`.
> - **DLQ acumulando:** inspecionar mensagem, corrigir causa raiz, republicar via RabbitMQ Management UI.

---

## 1. Classificação de Incidentes

| Severidade | Critério | SLA resposta | Responsável |
|---|---|---|---|
| P0 | Serviço indisponível, database down, zero requests processados | < 5 min | On-call + Tech Lead |
| P1 | Erros 5xx > 1%, latência API > 2s, consumer RabbitMQ parado | < 15 min | On-call |
| P2 | CSAT degradado, alertas não enviando, confidence baixo | < 1h | Eng. de plantão |
| P3 | Bug não-crítico, degradação leve de performance | < 24h | Time no próximo dia útil |

---

## 2. Runbook P0 — Serviço Indisponível

### Sintoma
Health check `/health` retornando 503, Sentry com spike de erros, Slack alertando `health_check_failed`.

### Diagnóstico em Ordem

```bash
# 1. Verificar status do container
docker ps | grep repasse-api
# Esperado: repasse-api Up X hours

# 2. Verificar logs recentes do container
docker logs repasse-api --tail=100 --since=15m

# 3. Verificar health check completo
curl -s https://api.repasseseguro.com.br/health | jq .
# Identificar qual dependency está "error" ou "down"

# 4. Verificar conectividade com banco (Supabase)
psql $DATABASE_URL -c "SELECT 1" 2>&1
# Se falhar: problema de banco

# 5. Verificar Redis
redis-cli -u $REDIS_URL ping
# Esperado: PONG

# 6. Verificar consumo de recursos do container
docker stats repasse-api --no-stream
# Verificar se CPU ou memória está saturado
```

### Resolução

**Container crashado ou não iniciado:**
```bash
# Reiniciar o container
docker-compose -f docker-compose.prod.yml restart api

# Verificar se subiu
docker-compose -f docker-compose.prod.yml ps
curl https://api.repasseseguro.com.br/health
```

**Database down (Supabase):**
```
1. Verificar status page do Supabase: https://status.supabase.com
2. Se problema de rede/VPC: verificar security groups / regras de firewall
3. Se Supabase está degradado: aguardar resolução ou acionar suporte Supabase
4. Comunicar status para equipe via Slack #alertas-criticos
```

**OOM (Out of Memory):**
```bash
# Verificar se o container foi morto por OOM
docker inspect repasse-api | jq '.[0].State.OOMKilled'
# Se true: aumentar limite de memória no docker-compose.prod.yml

# Após ajustar limite:
docker-compose -f docker-compose.prod.yml up -d api
```

**Migration falhou no boot:**
```bash
# Verificar logs de migration
docker logs repasse-api 2>&1 | grep -i prisma

# Aplicar migration manualmente
docker exec -it repasse-api npx prisma migrate deploy

# Se falhar — verificar migrations pendentes
docker exec -it repasse-api npx prisma migrate status
```

---

## 3. Runbook P1 — Spike de Erros 5xx

### Sintoma
Sentry mostrando > 1% de erros 5xx, alertas Slack no canal #alertas-prod.

### Diagnóstico

```bash
# 1. Identificar o endpoint com mais erros (via Sentry ou logs)
# Acessar Sentry → Issues → filtrar por environment=production, last 15min

# 2. Verificar logs do container para o erro específico
docker logs repasse-api --since=15m 2>&1 | grep '"level":50'
# level 50 = error no Pino

# 3. Verificar se é erro de banco (Prisma)
docker logs repasse-api --since=15m 2>&1 | grep -i 'prisma\|P1\|P2'

# 4. Verificar se é erro de integração externa
docker logs repasse-api --since=15m 2>&1 | grep -i 'openai\|langfuse\|sentry'
```

### Resolução por Tipo

**Erro de Prisma P1001/P1008 (database inacessível):**
```
→ Ver Runbook P0, seção database
```

**Erro de OpenAI (timeout, rate limit):**
```bash
# O FallbackAtivo deve ter sido ativado automaticamente (RF-018)
# Verificar se está respondendo com fallback
curl -X POST https://api.repasseseguro.com.br/api/v1/webchat/send \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"agentId":"...", "message":"teste"}'
# Resposta deve conter "fallback": true

# Se não está em fallback: verificar feature flag
# PostHog → Feature Flags → webchat-enabled → verificar se está ON
```

**Erro de validação em cascata (ValidationPipe):**
```
→ Verificar se uma mudança de contrato foi deployada
→ Rollback para versão anterior se necessário (ver Runbook de Rollback)
```

---

## 4. Runbook P1 — Consumer RabbitMQ Parado

### Sintoma
DLQ acumulando mensagens, alertas não sendo enviados, Sentry com erros de consumer.

### Diagnóstico

```bash
# 1. Verificar filas via RabbitMQ Management API
curl -s -u guest:guest http://rabbitmq-host:15672/api/queues | jq '.[] | {name, messages, consumers}'
# Verificar: messages crescendo, consumers = 0?

# 2. Verificar o estado do consumer no container
docker logs repasse-api --since=30m 2>&1 | grep -i 'amqp\|rabbitmq\|consumer'

# 3. Verificar se o consumer está conectado
docker logs repasse-api --since=5m 2>&1 | grep 'AlertsConsumer\|connected\|disconnected'
```

### Resolução

**Consumer desconectado (reconnect automático falhou):**
```bash
# Reiniciar apenas o serviço de API (não afeta banco/Redis)
docker-compose -f docker-compose.prod.yml restart api

# Verificar se consumer reconectou
# Aguardar 30s e verificar fila novamente
curl -s -u guest:guest http://rabbitmq-host:15672/api/queues | jq '.[] | select(.name | contains("dani-admin")) | {name, consumers}'
# Esperado: consumers = 1
```

**Mensagem envenenada (poison message) bloqueando fila:**
```bash
# 1. Acessar Management UI: http://rabbitmq-host:15672
# 2. Navegar para a fila afetada (ex: dani-admin.alerts.queue)
# 3. Clicar em "Get Messages" → inspecionar a mensagem
# 4. Se mensagem inválida: mover para DLQ manualmente (requeue=false)
# 5. Ou purgar a mensagem específica via API:
curl -X DELETE -u guest:guest \
  http://rabbitmq-host:15672/api/queues/%2F/dani-admin.alerts.queue/contents \
  -d '{"count":1,"requeue":false,"encoding":"auto"}'
```

**DLQ acumulando mensagens a processar:**
```bash
# Verificar conteúdo da DLQ
curl -s -u guest:guest \
  "http://rabbitmq-host:15672/api/queues/%2F/dani-admin.alerts.dlq" | jq '{messages, consumers}'

# 1. Corrigir a causa raiz do erro (ver Sentry para detalhes)
# 2. Após correção, republicar mensagens da DLQ:
#    Via Management UI: Queues → dani-admin.alerts.dlq → Move messages
#    Mover para: dani-admin.alerts.queue
```

---

## 5. Runbook — Takeover Travado

### Sintoma
Admin não consegue encerrar um takeover; interação fica presa com `status = EM_TAKEOVER` indefinidamente.

### Causa
Consumer de encerramento falhou ou houve inconsistência entre `interactions` e `takeovers`.

### Diagnóstico

```sql
-- Verificar takeovers ativos há mais de 2h (anômalo)
SELECT
  t.id as takeover_id,
  t.interaction_id,
  t.admin_id,
  t.started_at,
  NOW() - t.started_at as duration,
  i.status
FROM takeovers t
JOIN interactions i ON i.id = t.interaction_id
WHERE t.ended_at IS NULL
  AND t.started_at < NOW() - INTERVAL '2 hours';
```

### Resolução

```sql
-- Encerrar takeover manualmente (substituir UUIDs reais)
BEGIN;

UPDATE takeovers
SET ended_at = NOW()
WHERE interaction_id = '<interaction_id>'
  AND ended_at IS NULL;

UPDATE interactions
SET status = 'RESPONDIDA_PELA_IA',
    updated_at = NOW()
WHERE id = '<interaction_id>'
  AND status = 'EM_TAKEOVER';

-- Verificar antes de confirmar
SELECT status FROM interactions WHERE id = '<interaction_id>';
-- Deve retornar: RESPONDIDA_PELA_IA

COMMIT;
```

> **Auditoria:** Após resolução manual, criar registro em `admin_access_logs` com `action = 'MANUAL_TAKEOVER_CLOSE'` e `details` descrevendo o motivo.

---

## 6. Runbook — Kill Switch de IA

### Quando usar
Comportamento anômalo do modelo (respostas inadequadas, violação de escopo), incidente de segurança, ou custo de tokens explodindo.

### Procedimento (< 30s, sem deploy)

```
1. Acessar PostHog → Feature Flags
2. Localizar flag: webchat-enabled
3. Alterar para: Disabled (0%)
4. Salvar

Efeito imediato:
- Todos os novos requests de webchat recebem HTTP 503
- Body: { "error": { "code": "DA-AGT-001", "message": "O serviço de IA está temporariamente desabilitado." } }
- Interações em andamento: completam o ciclo atual, novas são bloqueadas
```

### Kill Switch Granular por Agente

```
Para desabilitar apenas Dani-Cessionário:
→ PostHog → dani-cessionario-enabled → Disabled

Para desabilitar apenas Dani-Cedente:
→ PostHog → dani-cedente-enabled → Disabled
```

### Reativar

```
1. Investigar e corrigir a causa raiz
2. Testar em staging com a flag habilitada
3. Reativar em produção: PostHog → flag → Enable (100%)
4. Monitorar os primeiros 5min após reativação (Sentry + Langfuse)
```

---

## 7. Runbook — Migrations em Produção

### Regras Invioláveis

1. **Nunca** rodar `prisma migrate dev` em produção — usa `prisma migrate deploy`
2. **Nunca** usar `--force` em migrations destrutivas
3. Toda migration destrutiva exige migration de rollback escrita e testada antes
4. Aplicar em staging primeiro, validar, depois produção

### Procedimento Padrão

```bash
# 1. Verificar status das migrations em produção
docker exec repasse-api-prod npx prisma migrate status

# 2. Aplicar migrations pendentes
docker exec repasse-api-prod npx prisma migrate deploy

# 3. Verificar resultado
docker exec repasse-api-prod npx prisma migrate status
# Todas devem aparecer como "Applied"

# 4. Health check pós-migration
curl https://api.repasseseguro.com.br/health
```

### Migration Destrutiva — Procedimento Seguro

```bash
# Antes do deploy:
# 1. Criar migration de rollback
npx prisma migrate dev --name rollback_v1_2_0_revert_drop_column
# Escrever SQL inverso no arquivo gerado manualmente

# 2. Testar em staging
# 3. Deploy com a migration destrutiva em produção
# 4. Manter migration de rollback pronta para uso emergencial
```

### Rollback de Migration

```bash
# Não há rollback automático no Prisma.
# Usar a migration de rollback criada previamente:
docker exec repasse-api-prod npx prisma migrate deploy
# (a migration de rollback foi incluída no prisma/migrations/)
```

---

## 8. Runbook — Rotação de Secrets

### Frequência
Toda chave de API deve ser rotacionada a cada 90 dias em produção (conforme D22).

### Procedimento

```bash
# 1. Gerar nova chave no provider (OpenAI, Langfuse, etc.)
# 2. Atualizar no GitHub Secrets (Settings → Secrets → Repository secrets)
# 3. Re-deploy do container para carregar a nova chave:
docker-compose -f docker-compose.prod.yml up -d --no-deps api

# 4. Verificar que a nova chave está funcionando
curl https://api.repasseseguro.com.br/health

# 5. Revogar a chave antiga no provider (após confirmar nova funcionando)
# 6. Registrar no log de auditoria operacional:
#    data, secret rotacionado, responsável
```

### Secrets por Ordem de Prioridade de Rotação

| Secret | Prioridade | Impacto de vazamento |
|---|---|---|
| `JWT_SECRET` | Crítica | Tokens de sessão comprometidos |
| `SUPABASE_SERVICE_ROLE_KEY` | Crítica | Acesso total ao banco |
| `OPENAI_API_KEY` | Alta | Custo e abuso do serviço |
| `DATABASE_URL` | Alta | Acesso ao banco de produção |
| `SENDGRID_API_KEY` | Alta | Envio de e-mails fraudulentos |
| `SLACK_WEBHOOK_URL` | Média | Spam no canal |
| `LANGFUSE_SECRET_KEY` | Média | Acesso a dados de observabilidade |

---

## 9. Operações de Manutenção Rotineira

### 9.1 Verificação Diária (automatizada via cron)

```bash
# Health check automatizado (cron: */5 * * * *)
curl -f https://api.repasseseguro.com.br/health || notify_slack_p0

# Verificar DLQs acumulando (cron: 0 * * * *)
CHECK_DLQ_SIZE=$(curl -s -u $RABBITMQ_USER:$RABBITMQ_PASS \
  http://$RABBITMQ_HOST:15672/api/queues/%2F/dani-admin.alerts.dlq \
  | jq '.messages')
[ "$CHECK_DLQ_SIZE" -gt 10 ] && notify_slack_p1 "DLQ com $CHECK_DLQ_SIZE mensagens"
```

### 9.2 Verificação Semanal (manual)

```sql
-- Interações com status inconsistente (em_takeover sem takeover ativo)
SELECT i.id, i.status, t.id as takeover_id
FROM interactions i
LEFT JOIN takeovers t ON t.interaction_id = i.id AND t.ended_at IS NULL
WHERE i.status = 'EM_TAKEOVER'
  AND t.id IS NULL;
-- Resultado deve ser vazio. Se não: resolver com Runbook seção 5.

-- Verificar índices de uso de queries lentas
SELECT schemaname, tablename, attname, n_distinct, correlation
FROM pg_stats
WHERE tablename IN ('interactions', 'takeovers', 'agent_configurations')
ORDER BY tablename, attname;
```

### 9.3 Manutenção de Dados

```sql
-- Arquivar interações antigas (> 2 anos) — executar em janela de manutenção
-- ATENÇÃO: Verificar requisitos de retenção LGPD antes de executar
INSERT INTO interactions_archive SELECT * FROM interactions
WHERE created_at < NOW() - INTERVAL '2 years';

DELETE FROM interactions
WHERE created_at < NOW() - INTERVAL '2 years'
  AND id IN (SELECT id FROM interactions_archive WHERE created_at < NOW() - INTERVAL '2 years');
```

---

## 10. Contatos de Escalação

| Situação | Contato Primário | Contato Secundário |
|---|---|---|
| P0 — serviço down | On-call via PagerDuty | Tech Lead |
| P0 — database Supabase | Suporte Supabase (suporte@supabase.com) | Tech Lead |
| P0 — OpenAI down | status.openai.com + Kill switch via PostHog | Tech Lead |
| P1 — bug crítico | Canal Slack #alertas-prod | Eng. responsável pelo módulo |
| Dúvida de operação | Canal Slack #ai-dani-admin | Documentação: D22, D24, D25 |

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Runbooks P0/P1, kill switch de IA, takeover travado, migrations, rotação de secrets, manutenção rotineira. |
