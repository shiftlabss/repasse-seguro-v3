# 25 - Observabilidade e Logs

## Cabeçalho

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
| --- | --- | --- | --- | --- |
| 25 - Observabilidade e Logs | v1.0 | 2026-03-22 | Claude Code Desktop (ShiftLabs Pipeline v9.5) | Aprovado |

---

## TL;DR

> 📌 **Estratégia de observabilidade do Repasse AI:**
>
> - **Stack:** Pino (logs estruturados) + Langfuse (traces LLM/agente) + Sentry (error tracking) + Railway Metrics (infra) + Upstash Redis CLI (cache).
> - **Correlation ID obrigatório** em todo log, trace, fila e resposta HTTP — gerado no ingresso do request e propagado via `AsyncLocalStorage`.
> - **5 níveis de log** (fatal, error, warn, info, debug) com retenção diferenciada por nível e ambiente.
> - **15 métricas** cobrindo aplicação, LLM, fila, banco e negócio — cada uma com threshold e ação esperada.
> - **8 alertas críticos** com canal (PagerDuty/Slack), responsável e runbook associado.
> - **4 dashboards operacionais:** Saúde Geral, Agente IA, Filas/Async, Segurança.
> - **Dados sensíveis nunca logados:** JWT, OTP, CPF, telefone, conteúdo de mensagem — mascarados por `PiiMaskingInterceptor` antes de qualquer escrita.

---

## 1. Arquitetura de Observabilidade

O Repasse AI adota uma estratégia de observabilidade em três pilares: **logs** (o quê aconteceu), **traces** (como aconteceu no pipeline LLM), **métricas** (indicadores de saúde em tempo real). Cada pilar tem ferramenta dedicada e nenhum sinal é compartilhado sem correlation ID.

```
Request → NestJS App
   │
   ├─ Pino Logger (JSON estruturado) ──→ Railway Log Stream ──→ retenção 30d
   │       │
   │       └─ Sentry SDK ──→ Sentry Cloud (erros 4xx/5xx) ──→ retenção 90d
   │
   ├─ Langfuse SDK (traces IA) ──→ Langfuse Cloud ──→ retenção 90d
   │       (LangGraph.js steps, tool calls, latência, custo, evals)
   │
   ├─ RabbitMQ Management ──→ métricas de fila via Railway env
   │
   └─ Railway Metrics ──→ CPU, memória, latência, uptime
```

**Ferramentas e responsabilidades:**

| Ferramenta | Responsabilidade | Ambiente |
| --- | --- | --- |
| Pino | Logs estruturados JSON — todos os módulos | Todos |
| Sentry | Error tracking — erros 4xx/5xx não esperados | Staging + Prod |
| Langfuse | Traces LLM — prompts, tools, latência, custo, evals | Staging + Prod |
| Railway Metrics | CPU, memória, uptime, deploy events | Staging + Prod |
| RabbitMQ Management | Queue depth, consumer lag, DLQ count | Todos |
| Upstash Dashboard | Redis hits/misses, latência, memory | Staging + Prod |

---

## 2. Níveis de Log

| Nível | Quando Usar | Campos Obrigatórios | Retenção (Prod) | Exemplo de Uso | Consumidor |
| --- | --- | --- | --- | --- | --- |
| `fatal` | Processo vai encerrar; estado irrecuperável | correlation_id, error_code, stack | 90 dias | OOM, falha na inicialização do módulo | On-call + PagerDuty imediato |
| `error` | Erro tratado com impacto funcional; requer investigação | correlation_id, error_code, module, stack | 90 dias | LLM_UNAVAILABLE, INTERNAL_ERROR, circuit breaker aberto | Sentry + Slack #alerts |
| `warn` | Comportamento anômalo sem impacto imediato | correlation_id, error_code, module | 30 dias | RATE_LIMIT_EXCEEDED, retry attempt, OTP expirado | Slack #alerts (threshold) |
| `info` | Fluxo normal de operação — eventos de negócio | correlation_id, module, event | 14 dias | Mensagem processada, notificação enviada, takeover liberado | Dashboard + auditoria |
| `debug` | Diagnóstico detalhado — desabilitado em produção | correlation_id, module, dados de estado | 3 dias (dev) | Parâmetros de chamada, estado do AgentState, Redis cache hit/miss | Dev/staging apenas |

> 🔴 **Regra inegociável:** nível `debug` é proibido em produção (`LOG_LEVEL=info` em prod). Qualquer tentativa de habilitar debug em prod requer aprovação explícita e janela de tempo máxima de 30 minutos.

---

## 3. Formato de Log Estruturado

Todo log do Repasse AI segue este schema JSON. O Pino é configurado para emitir JSON puro — sem formatação colorida em produção.

### 3.1 Schema Padrão

```json
{
  "timestamp": "2026-03-22T14:30:00.000Z",
  "level": "error",
  "service": "repasse-ai",
  "module": "ai",
  "message": "OpenAI API indisponível após 3 tentativas.",
  "correlation_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD9",
  "request_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD9",
  "tenant_id_hash": "b7e4f9a0",
  "user_id_hash": "a3f8c2d1",
  "error_code": "LLM_UNAVAILABLE",
  "environment": "production",
  "metadata": {
    "provider": "openai",
    "model": "gpt-4o",
    "attempts": 3,
    "circuit_breaker_status": "OPEN",
    "duration_ms": 12350
  },
  "stack": "LlmUnavailableError: OpenAI API indisponível...\n    at AiService.processMessage..."
}
```

> ⚙️ **Campo `stack`:** presente apenas em logs internos (Railway logs). Nunca incluído em respostas HTTP ao cliente.

### 3.2 Exemplo de Dado Mascarado

```json
{
  "timestamp": "2026-03-22T14:31:00.000Z",
  "level": "info",
  "service": "repasse-ai",
  "module": "whatsapp",
  "message": "OTP enviado com sucesso.",
  "correlation_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD8",
  "metadata": {
    "phone_masked": "+55859****1234",
    "otp_sent": true,
    "expires_in_seconds": 300
  }
}
```
Nenhum OTP, JWT ou número completo de telefone aparece no log.

### 3.3 Configuração Pino (NestJS)

```typescript
// apps/ai/src/common/logger/pino.config.ts
import pino from 'pino';

export const pinoLogger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: {
    service: 'repasse-ai',
    environment: process.env.NODE_ENV,
  },
  redact: {
    paths: [
      'req.headers.authorization',
      '*.otp',
      '*.password',
      '*.token',
      '*.jwt',
      '*.cpf',
      '*.cnpj',
    ],
    censor: '[REDACTED]',
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});
```

---

## 4. O Que Logar

Tabela de eventos obrigatórios por módulo com nível, campos e ação operacional associada.

| Módulo | Evento | Nível | Campos Obrigatórios | Risco Coberto | Ação Operacional |
| --- | --- | --- | --- | --- | --- |
| `ai` | LLM invocado | `info` | correlation_id, model, conversation_id, latency_ms | Latência LLM alta | Verificar Langfuse + OpenAI status |
| `ai` | LLM falhou | `error` | correlation_id, error_code, attempts, circuit_breaker | Indisponibilidade IA | Verificar circuit breaker + OpenAI |
| `ai` | Circuit breaker abriu | `error` | correlation_id, failure_count, opened_at | IA fora do ar | Alertar on-call — runbook CB-001 |
| `ai` | Cache semântico hit | `debug` | correlation_id, similarity_score, cache_key_hash | Custo OpenAI | Monitorar taxa de hit no Langfuse |
| `ai` | RAG retrieval falhou | `error` | correlation_id, error_code, threshold_used | Base de conhecimento indisponível | Verificar pgvector Supabase |
| `chat` | Mensagem criada | `info` | correlation_id, conversation_id, channel | Volume de conversas | Dashboard de negócio |
| `chat` | SSE conectado | `info` | correlation_id, conversation_id, user_id_hash | Sessões ativas | Monitorar conexões simultâneas |
| `whatsapp` | OTP enviado | `info` | correlation_id, phone_masked, expires_in | Auditoria de OTP | — |
| `whatsapp` | Webhook recebido | `info` | correlation_id, event_type, evolution_instance | Integridade webhook | Verificar HMAC se warn gerado |
| `whatsapp` | HMAC inválido | `warn` | correlation_id, expected_hmac_prefix, received_hmac_prefix | Segurança webhook | Alerta imediato — runbook SEC-001 |
| `supervision` | Takeover solicitado | `info` | correlation_id, conversation_id, supervisor_id_hash | Auditoria de takeover | — |
| `supervision` | Mutex de takeover conflito | `warn` | correlation_id, conversation_id, current_holder_hash | Condição de corrida | Verificar lógica SELECT FOR UPDATE |
| `notification` | Notificação enviada | `info` | correlation_id, notification_id, channel, template_id | Rastreabilidade de envio | — |
| `notification` | DLQ atingida | `error` | correlation_id, notification_id, channel, attempts | Falha de entrega | Processar DLQ — runbook NOTIF-001 |
| `db` | Query lenta (>500ms) | `warn` | correlation_id, query_hash, duration_ms, module | Performance de banco | Verificar indexes — runbook DB-001 |

---

## 5. O Que Não Logar

> 🔴 **Proibição absoluta — dados nunca logados:**

| Dado | Tratamento | Responsável |
| --- | --- | --- |
| JWT / Bearer token | Redact via Pino config (`req.headers.authorization`) | `PiiMaskingInterceptor` |
| OTP (código numérico) | Redact — logar apenas `otp_sent: true` | `PiiMaskingInterceptor` |
| Número de telefone completo | Mascarar: `+55859****1234` | `PiiMaskingInterceptor` |
| CPF / CNPJ | Redact — nunca logar | `PiiMaskingInterceptor` |
| Conteúdo de mensagem do usuário | Truncar a 100 chars em WARN/ERROR; omitir em INFO | `MessageTruncationInterceptor` |
| Email completo | Mascarar: `us**@do***.com` | `PiiMaskingInterceptor` |
| Variáveis de ambiente | Nunca logar — usar `process.env.VAR_NAME` sem valor | Revisão de código |
| Chaves de API (OpenAI, etc.) | Redact via Pino — qualquer campo `*key*`, `*secret*` | `PiiMaskingInterceptor` |
| IDs de usuário em claro | Substituir por `SHA-256(id)[:8]` | `PiiMaskingInterceptor` |

**Revisão de compliance:** auditoria trimestral dos logs por amostragem (10 registros aleatórios por módulo) para verificar ausência de PII.

---

## 6. Correlation ID e Rastreabilidade

### 6.1 Geração

O `correlation_id` é gerado no `CorrelationIdMiddleware` — primeiro ponto de entrada do request. Formato: `req_` + nanoid(21). [DECISÃO AUTÔNOMA] nanoid escolhido sobre UUID v4 por menor colisão e tamanho, e sobre ULID por simplicidade; UUID v7 descartado por dependência de biblioteca extra.

```typescript
// apps/ai/src/common/middleware/correlation-id.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { nanoid } from 'nanoid';
import { Request, Response, NextFunction } from 'express';
import { AsyncLocalStorage } from 'node:async_hooks';

export const correlationStorage = new AsyncLocalStorage<string>();

@Injectable()
export class CorrelationIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const correlationId = (req.headers['x-correlation-id'] as string) ?? `req_${nanoid()}`;
    res.setHeader('x-correlation-id', correlationId);
    correlationStorage.run(correlationId, next);
  }
}
```

### 6.2 Propagação

O `correlation_id` é propagado automaticamente para:

| Camada | Mecanismo |
| --- | --- |
| Logs Pino | `AsyncLocalStorage` — injetado em todo log via `correlationStorage.getStore()` |
| Resposta HTTP | Header `x-correlation-id` em toda resposta |
| Erros (BaseError) | Campo `correlationId` da classe de erro |
| RabbitMQ messages | Campo `correlation_id` no payload da mensagem |
| Langfuse traces | `metadata.correlation_id` em todo span |
| Sentry | `scope.setExtra('correlation_id', ...)` |
| SSE events | Campo `correlation_id` no payload SSE |

### 6.3 Rastreamento End-to-End

```
1. Request POST /chat/conversations/conv_abc/messages
   → correlation_id = req_01HX4M3P9Q
2. ChatService → AiService (correlation_id propagado)
3. AiService → LangGraph.js (Langfuse trace: correlation_id = req_01HX4M3P9Q)
4. AiService → RabbitMQ publish (correlation_id no payload)
5. NotificationWorker processa (correlation_id do payload → log)
6. Resposta HTTP com header x-correlation-id: req_01HX4M3P9Q
```

---

## 7. Métricas

### 7.1 Métricas de Aplicação

| Métrica | Objetivo | Unidade | Target | Threshold | Reação |
| --- | --- | --- | --- | --- | --- |
| `http_request_duration_ms` | Latência de endpoints | ms | p95 < 500ms | p95 > 1000ms = WARN; > 2000ms = CRITICAL | Verificar slow queries + cache |
| `http_error_rate` | Taxa de erros 5xx | % | < 1% | > 2% = WARN; > 5% = CRITICAL | Verificar Sentry + logs recentes |
| `active_sse_connections` | Sessões SSE ativas | count | — | > 500 simultâneos = WARN | Verificar Railway CPU |
| `agent_invocations_per_minute` | Volume de invocações do agente | req/min | — | > 100/min = WARN (rate limit) | Verificar distribuição por tenant |

### 7.2 Métricas de LLM / Agente IA

| Métrica | Objetivo | Unidade | Target | Threshold | Reação |
| --- | --- | --- | --- | --- | --- |
| `llm_latency_p95_ms` | Latência p95 das chamadas OpenAI | ms | < 5000ms | > 8000ms = WARN; > 15000ms = CRITICAL | Verificar OpenAI status page |
| `llm_error_rate` | Taxa de erros do LLM | % | < 2% | > 5% = WARN; > 10% = CRITICAL | Verificar circuit breaker status |
| `circuit_breaker_open_count` | Vezes que o circuit breaker abriu | count/hora | 0 | ≥ 1 = CRITICAL | Runbook CB-001 |
| `semantic_cache_hit_rate` | Taxa de cache semântico Redis | % | > 30% | < 10% = WARN | Verificar qualidade dos embeddings |
| `llm_cost_usd_per_day` | Custo diário OpenAI | USD | — | > $50/dia = WARN; > $100/dia = CRITICAL | Verificar volume + cache hit |
| `rag_retrieval_latency_ms` | Latência do RAG pgvector | ms | < 300ms | > 800ms = WARN | Verificar pgvector index + Supabase |

### 7.3 Métricas de Fila (RabbitMQ)

| Métrica | Objetivo | Unidade | Target | Threshold | Reação |
| --- | --- | --- | --- | --- | --- |
| `rabbitmq_queue_depth` | Mensagens pendentes por fila | count | < 100 | > 500 = WARN; > 2000 = CRITICAL | Escalar workers ou verificar DLQ |
| `rabbitmq_dlq_count` | Mensagens na DLQ | count | 0 | ≥ 1 = WARN; ≥ 10 = CRITICAL | Runbook NOTIF-001 ou processar DLQ |
| `notification_delivery_rate` | Taxa de entrega de notificações | % | > 95% | < 90% = WARN; < 80% = CRITICAL | Verificar canal de entrega |

### 7.4 Métricas de Banco

| Métrica | Objetivo | Unidade | Target | Threshold | Reação |
| --- | --- | --- | --- | --- | --- |
| `db_query_p95_ms` | Latência p95 de queries | ms | < 100ms | > 300ms = WARN; > 800ms = CRITICAL | Verificar indexes + EXPLAIN |
| `db_connection_pool_usage` | Uso do pool de conexões Supabase | % | < 70% | > 85% = WARN; > 95% = CRITICAL | Aumentar pool ou otimizar queries |

---

## 8. SLI, SLO e Error Budget

[DECISÃO AUTÔNOMA] SLOs definidos com base no perfil de uso (chat B2B em horário comercial) e nos thresholds da seção 7. Alternativa (SLO 99.9%) descartada por ser excessivamente agressiva para backend com dependência de OpenAI (SLA 99.5%).

| SLI | SLO | Janela | Error Budget (mensal) |
| --- | --- | --- | --- |
| Disponibilidade (HTTP 2xx+3xx / total) | ≥ 99.5% | 30 dias | 3h 39min de downtime |
| Latência p95 do agente IA (processMessage) | < 8000ms em 95% das req | 7 dias | 5% das req podem exceder |
| Taxa de erro 5xx | < 2% das req | 24 horas | — |
| Taxa de entrega de notificações | ≥ 95% | 24 horas | — |
| Processamento de fila (lag < 5 min) | ≥ 99% das mensagens | 1 hora | — |

**Monitoramento de error budget:** alerta quando > 50% do budget mensal for consumido antes do dia 20 do mês.

---

## 9. Alertas

> 🔴 **Regra inegociável:** todo alerta tem threshold, severidade, canal e ação esperada. Alertas sem ação são ruído e serão removidos.

| Alerta | Condição | Severidade | Canal | Responsável | Impacto | Ação Imediata |
| --- | --- | --- | --- | --- | --- | --- |
| `CIRCUIT_BREAKER_OPEN` | `circuit_breaker_open_count ≥ 1` | CRITICAL | PagerDuty + Slack #alerts | On-call backend | IA fora do ar | Runbook CB-001: verificar OpenAI status, resetar manualmente via Admin API se necessário |
| `LLM_ERROR_RATE_HIGH` | `llm_error_rate > 10%` em 5 min | CRITICAL | PagerDuty + Slack #alerts | On-call backend | Degradação severa do agente | Verificar OpenAI status page + circuit breaker |
| `HTTP_ERROR_RATE_5XX` | `http_error_rate > 5%` em 5 min | CRITICAL | PagerDuty + Slack #alerts | On-call backend | Produto indisponível | Verificar Sentry + Railway logs |
| `TENANT_MISMATCH_DETECTED` | ≥ 1 evento `TENANT_MISMATCH` | CRITICAL | PagerDuty + Slack #security | Tech Lead | Potencial violação de dados | Runbook SEC-001: isolar tenant + auditoria imediata |
| `WEBHOOK_HMAC_INVALID` | ≥ 3 eventos HMAC inválido em 10 min | HIGH | Slack #security | On-call backend | Tentativa de injeção webhook | Runbook SEC-002: verificar source IP + bloquear |
| `DLQ_MESSAGES_CRITICAL` | `rabbitmq_dlq_count ≥ 10` | HIGH | Slack #alerts | On-call backend | Notificações perdidas | Runbook NOTIF-001: inspecionar DLQ + reprocessar |
| `COST_LLM_DAILY_HIGH` | `llm_cost_usd_per_day > $100` | HIGH | Slack #alerts | Tech Lead | Overspend OpenAI | Verificar volume anômalo + cache hit rate |
| `DB_CONNECTION_POOL_CRITICAL` | `db_connection_pool_usage > 95%` | CRITICAL | PagerDuty + Slack #alerts | On-call backend | Queries bloqueadas | Runbook DB-001: verificar conexões abertas + restart se necessário |

---

## 10. Dashboards

### Dashboard 1: Saúde Geral do Sistema

**Pergunta operacional:** "O sistema está saudável agora?"

Widgets obrigatórios:
- `http_error_rate` (linha temporal 24h)
- `http_request_duration_ms` p50/p95/p99 (linha temporal)
- Uptime do serviço Railway (%) últimas 24h
- `active_sse_connections` (gauge em tempo real)
- Últimos 10 erros CRITICAL do Sentry

### Dashboard 2: Agente IA e LLM

**Pergunta operacional:** "A IA está funcionando bem e dentro do custo?"

Widgets obrigatórios:
- `llm_latency_p95_ms` (linha temporal 7d)
- `llm_error_rate` (linha temporal 24h)
- `circuit_breaker_open_count` (contador — deve ser 0)
- `semantic_cache_hit_rate` (gauge %)
- `llm_cost_usd_per_day` (linha temporal 30d + projeção mensal)
- `rag_retrieval_latency_ms` p95 (linha temporal)
- Top 5 templates com maior latência (Langfuse)

### Dashboard 3: Filas e Notificações

**Pergunta operacional:** "As mensagens assíncronas estão sendo processadas?"

Widgets obrigatórios:
- `rabbitmq_queue_depth` por fila (barras em tempo real)
- `rabbitmq_dlq_count` por fila (deve ser 0)
- `notification_delivery_rate` por canal (gauge %)
- Lag médio de processamento (ms) — enqueued_at → delivered_at
- Últimas 5 mensagens em DLQ (tabela)

### Dashboard 4: Segurança e Auditoria

**Pergunta operacional:** "Há anomalias de segurança ou acessos suspeitos?"

Widgets obrigatórios:
- Contagem de `AUTH_INVALID_TOKEN` últimas 24h (deve ser baixo)
- Contagem de `TENANT_MISMATCH` últimas 24h (deve ser 0)
- Contagem de HMAC inválido por hora (deve ser 0)
- Top 10 tenants por volume de requisições (anomalia de uso)
- `CRITICAL_OPTOUT_ATTEMPT` por hora

---

## 11. Retenção e Custos

> 🔴 **Regra LGPD:** logs com potencial de conter dados pessoais (mesmo mascarados) têm retenção máxima de 90 dias.

| Tipo de Dado | Ambiente | Retenção | Custo Estimado | Justificativa |
| --- | --- | --- | --- | --- |
| Logs `fatal`/`error` | Produção | 90 dias | Incluído no Railway | LGPD + investigação de incidentes |
| Logs `warn` | Produção | 30 dias | Incluído no Railway | Diagnóstico operacional |
| Logs `info` | Produção | 14 dias | Incluído no Railway | Rastreabilidade de negócio |
| Logs `debug` | Produção | Proibido | N/A | Ver regra §2 |
| Logs todos os níveis | Staging | 7 dias | Incluído no Railway | Depuração de deploy |
| Logs todos os níveis | Dev | 3 dias | Local | Desenvolvimento |
| Traces Langfuse | Produção | 90 dias | Plano Langfuse Cloud | LGPD + análise de custo LLM |
| Erros Sentry | Produção | 90 dias | Plano Sentry | Investigação de bugs |
| `NotificationLog` (banco) | Produção | 90 dias | Incluído no Supabase | LGPD (Doc 12 e Doc 21) |

[DECISÃO AUTÔNOMA] Retenção de logs de erro em 90 dias (não 365 dias): alinhado com política de retenção do modelo de dados LGPD (Doc 12) e suficiente para investigação de incidentes. Alternativa (365 dias) descartada por custo e por exceder janela de investigação prática de qualquer incidente documentado na literatura de SRE.

---

## 12. Health Checks e Pós-Deploy

### 12.1 Sinais de Health Check Pós-Deploy

Após cada deploy (Railway auto-deploy ou manual), verificar sequencialmente:

| Sinal | Threshold de Sucesso | Timeout | Falha indica |
| --- | --- | --- | --- |
| `GET /health` → 200 | Status `healthy` ou `degraded` (não `down`) | 10s | Serviço não inicializou |
| `http_error_rate` | < 1% nos primeiros 5 min pós-deploy | 5 min | Regressão introduzida |
| `llm_error_rate` | < 5% nos primeiros 10 min | 10 min | Problema de integração OpenAI |
| `rabbitmq_queue_depth` | Não cresce > 50 mensagens em 2 min | 2 min | Worker parado |
| `circuit_breaker_open_count` | 0 | Imediato | LLM indisponível |
| Sentry — novos erros CRITICAL | 0 erros CRITICAL novos | 10 min | Bug crítico introduzido |

### 12.2 Decisão de Rollback

Iniciar rollback automático (Railway rollback para versão anterior) se:
- `http_error_rate > 10%` por mais de 3 minutos após deploy
- `circuit_breaker_open_count ≥ 1` nos primeiros 5 minutos
- `GET /health` retorna 503 por mais de 30 segundos

### 12.3 Checklist Manual Pós-Deploy (Go-Live)

- [ ] `GET /health` retorna 200 com todos os serviços UP
- [ ] Enviar uma mensagem de teste no Webchat — agente responde em < 10s
- [ ] Verificar Langfuse — trace criado sem erro
- [ ] Verificar Sentry — sem novos erros CRITICAL
- [ ] Verificar Railway — CPU < 50%, memória < 70%
- [ ] Verificar RabbitMQ Management — filas vazias, DLQs com 0 mensagens

---

## 13. Integração com Runbook

Para cada alerta crítico desta seção, o Runbook Operacional (Doc 26) deve conter o procedimento correspondente:

| Alerta | Runbook ID | Categoria |
| --- | --- | --- |
| `CIRCUIT_BREAKER_OPEN` | CB-001 | Degradação de IA |
| `LLM_ERROR_RATE_HIGH` | CB-002 | Degradação de IA |
| `HTTP_ERROR_RATE_5XX` | APP-001 | Saúde geral |
| `TENANT_MISMATCH_DETECTED` | SEC-001 | Segurança |
| `WEBHOOK_HMAC_INVALID` | SEC-002 | Segurança |
| `DLQ_MESSAGES_CRITICAL` | NOTIF-001 | Filas |
| `COST_LLM_DAILY_HIGH` | COST-001 | Custo |
| `DB_CONNECTION_POOL_CRITICAL` | DB-001 | Banco de dados |

---

## 14. Backlog de Pendências

| ID | Descrição | Prioridade | Observação |
| --- | --- | --- | --- |
| OBS-001 | Definir plataforma de dashboard (Grafana Cloud, Datadog ou Railway nativo) para hospedar os 4 dashboards documentados | Alta | Railway não oferece dashboards customizados — necessário avaliar custo |
| OBS-002 | Configurar alertas automáticos no Sentry (webhook para Slack/PagerDuty) — requer configuração manual pós-deploy | Alta | Documentar no Runbook |
| OBS-003 | Avaliar se `llm_cost_usd_per_day` deve ser calculado via Langfuse API ou via parsing de fatura OpenAI | Média | Langfuse tem cost tracking nativo — preferível |

> **Decisões Autônomas Tomadas Neste Documento:**
>
> 1. **[DECISÃO AUTÔNOMA] `correlation_id` via nanoid:** alternativa ULID descartada por dependência de biblioteca extra. UUID v4 descartado por tamanho maior e menor densidade de informação. Critério: praticidade e zero dependências além do `nanoid` já presente.
> 2. **[DECISÃO AUTÔNOMA] Retenção de logs de erro 90 dias:** alternativa 365 dias descartada por custo e excesso de janela para investigação. Critério: alinhamento com LGPD do modelo de dados.
> 3. **[DECISÃO AUTÔNOMA] SLO de disponibilidade 99.5%:** alternativa 99.9% descartada por ser incompatível com SLA do OpenAI (99.5%). Critério: SLO do produto não pode exceder SLO do componente mais frágil.

---

*Próximo documento do pipeline: D23 — Guia de Contribuição.*
