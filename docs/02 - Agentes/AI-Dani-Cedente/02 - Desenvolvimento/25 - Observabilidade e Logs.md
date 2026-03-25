# 25 - Observabilidade e Logs — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| **Nome do Documento** | Observabilidade e Logs — AI-Dani-Cedente |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Bloco** | 6 — Qualidade |
| **Dependências** | D02, D14, D16, D20 |

---

> **📌 TL;DR**
>
> - **Stack de observabilidade:** Pino (logs estruturados JSON) + Sentry (erros ≥500 + LLM) + Langfuse (traces de agente IA) + PostHog (métricas de produto) + Prometheus/Grafana (infraestrutura).
> - **Correlation ID:** obrigatório em 100% dos logs, respostas de erro, traces Langfuse e mensagens de fila.
> - **Dados proibidos em logs:** CPF, e-mail, telefone, nome completo, conteúdo de mensagem do Cedente, tokens JWT.
> - **SLO:** Latência chat ≤5s p95 | Disponibilidade ≥99.5% | Taxa de erro LLM ≤2% em 15min.
> - **6 alertas críticos** com threshold, severidade, canal e ação operacional definidos.
> - **4 dashboards** operacionais: saúde geral, agente IA, notificações, infraestrutura.
> - **Retenção:** logs de aplicação 30 dias (prod) / 7 dias (dev); traces Langfuse 90 dias; métricas Prometheus 15 dias.

---

## 1. Arquitetura de Observabilidade

A estratégia de observabilidade da AI-Dani-Cedente usa ferramentas especializadas por tipo de sinal, evitando concentração em uma única plataforma. Cada ferramenta tem papel claro e responde a uma classe específica de perguntas operacionais.

```mermaid
graph TD
    A[NestJS App] --> B[Pino Logger\nlogs estruturados JSON]
    A --> C[Sentry SDK\nerros ≥500 + LLM crítico]
    A --> D[Langfuse SDK\ntraces do agente IA]
    A --> E[PostHog SDK\neventos de produto]
    A --> F[/metrics endpoint\nPrometheus scrape]

    B --> G[stdout/stderr\nDocker logs]
    G --> H[Railway Logs\nConsulta operacional]
    G --> I[Loki / Datadog\nFase 2 — planejado]

    C --> J[Sentry Dashboard\nErros + alertas]
    D --> K[Langfuse Dashboard\nTraces LLM + custo]
    E --> L[PostHog Dashboard\nMétricas produto]
    F --> M[Prometheus\nScrape 15s]
    M --> N[Grafana Dashboard\nInfraestrutura]
    N --> O[AlertManager\nAlertas infra]

    J --> P[PagerDuty / Slack\n#ops-incidents]
    O --> P
```

| Ferramenta | Papel | Pergunta que responde |
|---|---|---|
| **Pino** | Logs estruturados JSON com correlation ID | O que aconteceu neste request? |
| **Sentry** | Erros de aplicação ≥500 + erros LLM | Qual erro ocorreu em produção e onde? |
| **Langfuse** | Traces completos do ciclo LLM (prompt, completion, tools, custo) | Por que a Dani respondeu X? Qual foi o custo desta sessão? |
| **PostHog** | Eventos de produto: CSAT, taxa resolução, erros 4xx de negócio | O produto está cumprindo suas métricas? |
| **Prometheus + Grafana** | Métricas de infraestrutura: CPU, memória, fila, banco | A infra está saudável? Há gargalos? |

---

## 2. Níveis de Log

| Nível | Quando Usar | Campos Obrigatórios | Retenção (Prod) | Exemplo de Uso | Consumidor |
|---|---|---|---|---|---|
| **ERROR** | Exceção não recuperável, erro ≥500, falha de integração P0/P1 | timestamp, level, service, module, message, correlation_id, error.code, error.stack | 30 dias | `GlobalExceptionFilter`, falha OpenAI, DLQ RabbitMQ | Sentry + Ops |
| **WARN** | Degradação esperada, retry, rate limit atingido, circuit breaker aberto | timestamp, level, service, module, message, correlation_id, metadata | 30 dias | 3ª tentativa de retry, threshold 10% erros LLM | Grafana Alert |
| **INFO** | Eventos de negócio relevantes: request recebido, notificação enviada, agente mudou de estado | timestamp, level, service, module, message, correlation_id, duration_ms | 30 dias | Proposta aceita, ZapSign enviado, sessão iniciada | Operação |
| **DEBUG** | Dados de diagnóstico: payload de fila, decisão do agente, cache hit/miss | timestamp, level, service, module, message, correlation_id, metadata | 7 dias (dev) / desligado prod | Cache miss no Redis, routing key escolhida | Dev local |
| **TRACE** | Fluxo interno detalhado: middleware chain, guards, cada step do LangGraph | timestamp, level, service, module, message, correlation_id | Desligado prod | Transições de estado do agente | Dev local |

> ⚙️ **Produção:** Nível mínimo em produção é `INFO`. DEBUG e TRACE são desligados via `LOG_LEVEL=info` (env var). Nunca habilitar TRACE em produção — gera volume excessivo e risco de vazar dados.

---

## 3. Formato de Log Estruturado

Todos os logs seguem o schema JSON padrão via Pino. Campos adicionais vão em `metadata`. Nunca usar `console.log` — sempre o logger injetado.

**Schema obrigatório:**
```json
{
  "timestamp": "2026-03-23T14:30:00.123Z",
  "level": "info",
  "service": "ai-dani-cedente",
  "version": "1.2.3",
  "module": "chat",
  "message": "Mensagem recebida e enfileirada para processamento",
  "correlation_id": "corr_01HX4K8MZPQ3R7VB9N2X",
  "request_id": "req_01HX4K8...",
  "cedente_id": "ced_01HX...",
  "session_id": "sess_01HX...",
  "duration_ms": 45,
  "metadata": {
    "agent_state": "analyzing_proposal",
    "tokens_used": 312
  }
}
```

**Log de erro (com stack):**
```json
{
  "timestamp": "2026-03-23T14:30:05.456Z",
  "level": "error",
  "service": "ai-dani-cedente",
  "version": "1.2.3",
  "module": "agent",
  "message": "Falha ao chamar ferramenta get-proposal",
  "correlation_id": "corr_01HX4K8MZPQ3R7VB9N2X",
  "error": {
    "code": "DCE-AGENT-5030_001",
    "type": "ExternalServiceError",
    "message": "Timeout na chamada ao módulo de propostas",
    "stack": "ExternalServiceError: Timeout...\n    at AgentService.callTool (/app/src/modules/agent/agent.service.ts:142:11)"
  },
  "metadata": {
    "tool": "get-proposal",
    "attempt": 2,
    "timeout_ms": 5000
  }
}
```

**Dado mascarado (PII):**
```json
{
  "timestamp": "2026-03-23T14:30:00Z",
  "level": "info",
  "module": "auth",
  "message": "JWT validado com sucesso",
  "correlation_id": "corr_01HX...",
  "metadata": {
    "cedente_id": "ced_01HX...",
    "email": "jo***@gm***.com",   // ← mascarado por PiiMaskingMiddleware
    "cpf": "***.***.***-**"        // ← mascarado — nunca o valor real
  }
}
```

### 3.1 Implementação no NestJS

```typescript
// src/common/logger/logger.service.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: {
    service: 'ai-dani-cedente',
    version: process.env.APP_VERSION || 'unknown',
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: ['*.email', '*.cpf', '*.phone', '*.token', '*.password', '*.jwt'],
    censor: '[REDACTED]',
  },
});
```

---

## 4. O que Logar

| Evento | Nível | Módulo | Campos Adicionais | Risco Coberto | Ação Operacional |
|---|---|---|---|---|---|
| Request HTTP recebido | INFO | http | method, path, status, duration_ms | Rastreabilidade de chamadas | Diagnóstico de latência por endpoint |
| JWT validado / rejeitado | INFO / WARN | auth | cedente_id (se válido), reject_reason | Tentativas de acesso inválido | Alerta se rejeições > 5/min do mesmo IP |
| Agente mudou de estado | INFO | agent | from_state, to_state, trigger | Diagnóstico de fluxo da Dani | Verificar transições anômalas |
| Tool LangChain chamada | INFO | agent | tool_name, duration_ms, success | Performance por ferramenta | Ferramenta com p95 > 2s → otimizar |
| LLM call (GPT-4) | INFO | agent | tokens_input, tokens_output, duration_ms, model | Custo e latência LLM | Budget mensal e p95 > 5s |
| Confirmação de ação financeira | INFO | chat | action_type, cedente_id, confirmed | Auditoria de ações críticas | Rastrear aceites/recusas de proposta |
| Rate limit atingido | WARN | chat | cedente_id, limit, window_ms | Uso abusivo ou bug de cliente | Investigar se cedente_id legítimo |
| Notificação publicada na fila | INFO | notification | event_type, channel, notification_id | Rastreabilidade de entrega | Cruzar com tracking no `notification_log` |
| Notificação falhou (DLQ) | ERROR | notification | notification_id, event_type, failure_reason, retry_count | Perda de entrega crítica | Ver Runbook — Falha de Notificação |
| Circuit breaker aberto | ERROR | fallback | threshold_pct, window_ms, trigger | Degradação do serviço | Desabilitar Dani via PostHog flag |
| Webhook ZapSign recebido | INFO | escrow | event_type, document_id, hmac_valid | Integridade de integração | HMAC inválido → bloquear + alertar |
| RAG ingestão concluída | INFO | rag | chunks_indexed, duration_ms, source | Atualidade da base de conhecimento | Erro de ingestão → conhecimento desatualizado |
| Sessão expirada / removida do Redis | DEBUG | agent | session_id, ttl_remaining | Gestão de memória de sessão | Expiração prematura → rever TTL |

---

## 5. O que Não Logar

> 🔴 **Dados proibidos em qualquer nível de log:**

| Dado Proibido | Motivo | Substituto |
|---|---|---|
| CPF do Cedente | PII sensível — LGPD | Hash irreversível ou `[REDACTED]` |
| E-mail do Cedente | PII sensível — LGPD | `jo***@gm***.com` (mascarado pelo middleware) |
| Telefone / WhatsApp | PII sensível — LGPD | `[REDACTED]` |
| Nome completo | PII sensível — LGPD | `cedente_id` (UUID) |
| Conteúdo de mensagem do chat | Privacidade + risco de vazar dados de negócio | Logar apenas `session_id` + `message_type` |
| Token JWT completo | Credencial de acesso | `token_prefix: "eyJh..."` (apenas prefixo) |
| Chave de API (OpenAI, Langfuse, etc.) | Credencial crítica | `[REDACTED]` — usar `redact` do Pino |
| Valor da proposta no contexto livre | Dado de negócio sensível | Apenas em traces Langfuse (com masking) |
| Senha ou hash de senha | Credencial | `[REDACTED]` |
| Número de conta bancária | Dado financeiro | `[REDACTED]` |

**Regra de sanitização:**

O `PiiMaskingMiddleware` (D18) atua ANTES do log em todas as respostas. O Pino está configurado com `redact` para remover automaticamente campos conhecidos. Qualquer campo novo que chegue via `metadata` deve ser auditado antes de entrar em produção.

**Responsabilidade por revisão:** A cada PR que adicionar campos ao logger, o revisor deve verificar se há risco de PII. O checklist de PR (D28) inclui esta verificação.

---

## 6. Correlation ID e Rastreabilidade

O `correlation_id` é o fio condutor entre todos os sistemas. Permite reconstruir o caminho completo de um request sem acesso ao chat do usuário.

### 6.1 Geração e Propagação

```typescript
// src/common/middleware/correlation-id.middleware.ts
@Injectable()
export class CorrelationIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const correlationId = req.headers['x-correlation-id']
      || `corr_${generateULID()}`;

    req.correlationId = correlationId;
    res.setHeader('x-correlation-id', correlationId);

    // Propagar para AsyncLocalStorage (usado por toda a cadeia)
    correlationStorage.run({ correlationId }, next);
  }
}
```

### 6.2 Propagação por Componente

| Componente | Como Recebe | Como Propaga |
|---|---|---|
| **HTTP Request** | Header `x-correlation-id` ou gerado | Resposta + AsyncLocalStorage |
| **Pino Logger** | AsyncLocalStorage | Campo `correlation_id` em todos os logs |
| **RabbitMQ Producer** | AsyncLocalStorage | Header `x-correlation-id` na mensagem |
| **RabbitMQ Worker** | Header da mensagem | AsyncLocalStorage + logs |
| **Langfuse trace** | Injetado na criação do trace | Campo `correlation_id` no trace root |
| **Sentry error** | Automaticamente via Pino-Sentry bridge | `tags.correlation_id` no evento |
| **Webhook ZapSign** | Campo na fila ao receber | Propagado na cadeia de processamento |
| **Respostas de erro** | AsyncLocalStorage | Campo `correlation_id` no body da resposta |

### 6.3 Exemplo de Rastreamento Completo

```
Request: POST /api/v1/chat/sessions/sess_001/messages
  correlation_id: corr_01HX4K8MZPQ3R7VB9N2X

  → [INFO] chat: Mensagem recebida                 | corr_01HX4K8...
  → [INFO] agent: Estado idle → analyzing_proposal  | corr_01HX4K8...
  → [INFO] agent: Tool get-proposal chamada          | corr_01HX4K8...
  → [INFO] agent: LLM call GPT-4, 312 tokens        | corr_01HX4K8...
  → [INFO] chat: Resposta SSE enviada               | corr_01HX4K8...
  → [INFO] notification: Evento publicado na fila   | corr_01HX4K8...

  Langfuse trace: dani_01HX4K8... → correlation_id: corr_01HX4K8...
```

---

## 7. Métricas

### 7.1 Métricas de Aplicação

| Métrica | Objetivo | Unidade | Target | Threshold Alerta | Reação |
|---|---|---|---|---|---|
| `dani_chat_message_duration_ms` | Latência end-to-end de mensagem | ms | p95 ≤5.000ms | p95 > 5.000ms | Investigar LLM / ferramentas |
| `dani_chat_messages_total` | Volume de mensagens por estado (success/error) | count/min | — | — | Dashboard de saúde |
| `dani_llm_call_duration_ms` | Latência da chamada GPT-4 | ms | p95 ≤4.000ms | p95 > 4.000ms | Verificar OpenAI status |
| `dani_llm_error_rate` | Taxa de erro em chamadas LLM (15min) | % | < 2% | > 10% → WARN / > 30% → CRITICAL | Circuit breaker / fallback |
| `dani_agent_state_transitions_total` | Transições de estado do agente por tipo | count | — | — | Detectar estados anômalos |
| `dani_tool_call_duration_ms{tool}` | Latência por ferramenta LangChain | ms | p95 ≤2.000ms | p95 > 2.000ms | Ferramenta lenta → otimizar |
| `dani_rate_limit_triggered_total` | Rate limit acionado por canal | count/h | — | > 50/h por cedente | Investigar uso abusivo |
| `dani_circuit_breaker_state` | Estado do circuit breaker (0=closed, 1=open) | boolean | 0 (fechado) | = 1 | Página Admin + Runbook |
| `dani_rag_search_duration_ms` | Latência de busca semântica pgvector | ms | p95 ≤500ms | p95 > 1.000ms | Reavaliar index pgvector |
| `dani_session_active_total` | Sessões ativas no Redis | count | — | > 500 simultâneas | Verificar TTL e escala |

### 7.2 Métricas de Infraestrutura

| Métrica | Objetivo | Target | Threshold Alerta |
|---|---|---|---|
| `process_cpu_seconds_total` | Uso de CPU do processo NestJS | < 70% média 5min | > 80% → WARN, > 90% → CRITICAL |
| `process_resident_memory_bytes` | Uso de memória | < 512MB | > 700MB → WARN |
| `redis_connected_clients` | Conexões Redis ativas | < 50 | > 80 → WARN |
| `rabbitmq_queue_messages{queue}` | Mensagens pendentes por fila | < 100 em `notification.send` | > 500 → WARN, > 1000 → CRITICAL |
| `rabbitmq_queue_messages{queue=dlq}` | Mensagens na DLQ | 0 | > 10 em 1h → CRITICAL |
| `pg_stat_activity_count` | Conexões ativas no PostgreSQL | < 20 | > 40 → WARN |

### 7.3 Métricas de Negócio

| Métrica | Objetivo | Target | Threshold Alerta |
|---|---|---|---|
| `dani_csat_score_avg` | CSAT médio das sessões avaliadas | ≥ 4.0/5.0 | < 3.5 em 24h → alerta produto |
| `dani_resolution_rate` | Taxa de resolução sem takeover | ≥ 80% | < 70% em 24h → revisar prompts |
| `dani_takeover_rate` | Taxa de takeover para Admin | < 20% | > 30% → investigar |
| `dani_notification_delivery_rate` | Taxa de entrega de notificações | ≥ 95% | < 90% → verificar canal |

---

## 8. SLI, SLO e Error Budget

| Indicador | SLI (Medição) | SLO (Target) | Error Budget (30 dias) |
|---|---|---|---|
| **Latência de chat** | p95 de `dani_chat_message_duration_ms` | p95 ≤5.000ms | 5% dos requests pode ultrapassar |
| **Disponibilidade do serviço** | Uptime (health check `/health` ≥ 200) | ≥99.5% | 3h36min de downtime/mês |
| **Taxa de erro LLM** | `dani_llm_error_rate` por janela 15min | ≤2% em steady state | Circuit breaker abre em > 10% por 15min |
| **Entrega de notificações críticas** | `dani_notification_delivery_rate` para prioridade critical | ≥99% | 1% de falha tolerada com DLQ |

**Cálculo de error budget:**
```
Error Budget = 1 - SLO
Disponibilidade: 1 - 0.995 = 0.005 = 0.5% do mês = ~3h36min
Se budget < 20%: congelar deploys não-críticos
Se budget = 0%: post-mortem obrigatório
```

---

## 9. Alertas

| # | Alerta | Condição de Disparo | Severidade | Canal | Responsável | Impacto | Ação Imediata |
|---|---|---|---|---|---|---|---|
| A-001 | **Circuit Breaker Aberto** | `dani_circuit_breaker_state = 1` | CRITICAL | Slack #ops-incidents + PagerDuty | On-call | Dani indisponível para todos os Cedentes | Verificar taxa de erro LLM. Se > 30%: manter desligado. Se falso positivo: reativar via `POST /admin/fallback/enable` |
| A-002 | **Latência chat p95 > 5s** | p95 `dani_chat_message_duration_ms` > 5.000ms por 5min | HIGH | Slack #ops-alerts | Backend Lead | Experiência degradada para Cedentes | Verificar Langfuse — qual step está lento? LLM? Tool? RAG? |
| A-003 | **Taxa de erro LLM > 10%** | `dani_llm_error_rate` > 10% em janela 15min | HIGH | Slack #ops-alerts | Backend Lead | Respostas de erro para Cedentes | Checar status OpenAI. Se API down: circuit breaker já deve ter aberto. Se não: revisar threshold. |
| A-004 | **DLQ com > 10 mensagens** | `rabbitmq_queue_messages{queue="notification.dlq"}` > 10 em 1h | HIGH | Slack #ops-alerts | Backend Lead | Notificações críticas não entregues | Inspecionar DLQ. Reprocessar mensagens ou investigar falha do worker. |
| A-005 | **CSAT médio < 3.5** | `dani_csat_score_avg` < 3.5 por 24h | WARN | Slack #product-metrics | Product Lead | Satisfação abaixo do threshold | Analisar sessões com CSAT < 3. Identificar padrão de falha de entendimento. |
| A-006 | **Webhook ZapSign com HMAC inválido** | Mais de 3 requisições com HMAC inválido em 10min | CRITICAL | Slack #ops-security | Tech Lead | Possível ataque ou misconfiguration | Bloquear IP temporariamente. Verificar `ZAPSIGN_WEBHOOK_SECRET` na integração. |
| A-007 | **Memória > 700MB** | `process_resident_memory_bytes` > 700MB | WARN | Slack #ops-alerts | DevOps | Risco de OOM em pico | Verificar vazamentos de memória. Reiniciar instância se crítico. |
| A-008 | **Taxa de takeover > 30%** | `dani_takeover_rate` > 30% em 24h | WARN | Slack #product-metrics | Product Lead | Dani não consegue resolver sem intervenção humana | Revisar system prompt e base RAG. |

---

## 10. Dashboards

### Dashboard 1 — Saúde Geral da Dani

**Pergunta guia:** A Dani está respondendo bem agora?

| Widget | Tipo | Dado |
|---|---|---|
| Status do Circuit Breaker | Badge (verde/vermelho) | `dani_circuit_breaker_state` |
| Latência p50/p95/p99 (últimas 1h) | Gráfico de linha | `dani_chat_message_duration_ms` percentis |
| Taxa de erro (últimas 15min) | Gauge | `dani_llm_error_rate` |
| Mensagens processadas/min | Contador | `dani_chat_messages_total{status="success"}` |
| Sessões ativas | Número | `dani_session_active_total` |
| CSAT médio (24h) | Gauge | `dani_csat_score_avg` |

### Dashboard 2 — Agente IA (LangGraph + LLM)

**Pergunta guia:** Onde o agente está gastando tempo e dinheiro?

| Widget | Tipo | Dado |
|---|---|---|
| Distribuição de estados do agente | Pie chart | `dani_agent_state_transitions_total` por estado |
| Latência por ferramenta (p95) | Bar chart | `dani_tool_call_duration_ms{tool}` |
| Tokens usados por hora (input/output) | Gráfico de área empilhada | Langfuse API — tokens por hora |
| Custo estimado por dia (USD) | Número | Langfuse — cost tracking |
| Taxa de takeover | Gauge | `dani_takeover_rate` |
| RAG: latência de busca p95 | Número | `dani_rag_search_duration_ms` p95 |

### Dashboard 3 — Notificações

**Pergunta guia:** As notificações estão chegando ao Cedente?

| Widget | Tipo | Dado |
|---|---|---|
| Taxa de entrega por canal (webchat/email) | Gauge | `dani_notification_delivery_rate{channel}` |
| Mensagens na fila por canal | Contador | `rabbitmq_queue_messages{queue}` |
| Tamanho da DLQ | Contador (alarme se > 0) | `rabbitmq_queue_messages{queue="dlq"}` |
| Notificações enviadas/h por tipo | Gráfico de linha | `notification_log.status = SENT` agrupado por event_type |
| Taxa de opt-out nas últimas 24h | % | `notification_log.status = UNSUBSCRIBED / SENT` |

### Dashboard 4 — Infraestrutura

**Pergunta guia:** A infraestrutura está com capacidade para a carga atual?

| Widget | Tipo | Dado |
|---|---|---|
| CPU do processo NestJS (5min avg) | Gauge | `process_cpu_seconds_total` |
| Memória RSS | Gauge | `process_resident_memory_bytes` |
| Conexões Redis | Número | `redis_connected_clients` |
| Conexões PostgreSQL ativas | Número | `pg_stat_activity_count` |
| Latência p95 do banco (queries) | Número | `pg_query_duration_ms` p95 |
| Uptime do serviço (30 dias) | % | Calculado de `up` metric |

---

## 11. Retenção e Custos

| Tipo de Dado | Ambiente | Retenção | Justificativa | Custo Estimado |
|---|---|---|---|---|
| Logs de aplicação (Pino → stdout) | Produção | 30 dias | Janela de investigação de incidentes | Railway Logs incluído |
| Logs de aplicação | Desenvolvimento | 7 dias | Diagnóstico local — sem retenção longa | N/A |
| Traces Langfuse (LLM) | Produção | 90 dias | Alinhado com política de chat (LGPD D01). Necessário para auditoria de qualidade da IA | Langfuse Cloud — tier free até 50k traces/mês |
| Erros Sentry | Produção | 90 dias | Investigação retroativa de bugs | Sentry free tier — 5k erros/mês |
| Métricas Prometheus | Produção | 15 dias | Tendências operacionais recentes — custo alto para mais | Railway addon ou self-hosted |
| `notification_log` (PostgreSQL) | Produção | 90 dias | Rastreabilidade de entrega + LGPD opt-out | Incluído no Supabase |
| Eventos PostHog (produto) | Produção | 1 ano | Análise de produto e tendências CSAT | PostHog free tier — 1M eventos/mês |

> [DECISÃO AUTÔNOMA]: Retenção de logs em 30 dias para produção — Justificativa: cobre janela de investigação de incidentes (típico SLA de 90% dos incidentes resolvidos em < 7 dias) com margem, sem custo de armazenamento excessivo | Alternativa descartada: 90 dias — necessário apenas para compliance de dados financeiros, não para logs operacionais; o custo seria 3x maior sem benefício operacional claro.

---

## 12. Health Checks e Pós-Deploy

### 12.1 Endpoints de Health

```
GET /health          → liveness check (processo vivo)
GET /health/ready    → readiness check (banco + Redis + RabbitMQ conectados)
GET /health/live     → K8s liveness probe (apenas processo)
```

**Resposta `GET /health/ready`:**
```json
{
  "status": "ok",
  "checks": {
    "database": { "status": "ok", "latency_ms": 12 },
    "redis": { "status": "ok", "latency_ms": 3 },
    "rabbitmq": { "status": "ok", "latency_ms": 8 },
    "openai": { "status": "ok", "latency_ms": 145 }
  },
  "version": "1.2.3",
  "timestamp": "2026-03-23T14:30:00Z"
}
```

### 12.2 Sinais Obrigatórios Pós-Deploy

| Sinal | Threshold de Aprovação | Interpretação de Falha |
|---|---|---|
| `/health/ready` retorna 200 | Dentro de 30s após start | Serviço não inicializou — rollback |
| Taxa de erro (5min após deploy) | < 1% | Novo código introduziu regressão — rollback |
| Latência p95 chat (5min após deploy) | ≤5.000ms | Degradação de performance — investigar antes de continuar |
| Circuit breaker estado | Fechado (= 0) | Se abrir nos primeiros 5min → rollback imediato |
| Nenhuma mensagem na DLQ | 0 novas mensagens em 5min | Falha de worker — verificar logs |
| Primeiro request de chat bem-sucedido | Status 200 ou SSE stream aberto | Smoke test manual obrigatório |

### 12.3 Checklist de Rollback

1. `railway rollback --deployment <previous-deployment-id>`
2. Verificar `/health/ready` após rollback
3. Confirmar circuit breaker fechado
4. Comunicar no #ops-incidents com correlation_id do incidente

---

## 13. Integração com Runbook

| Alerta | Seção do Runbook (D26) |
|---|---|
| A-001 — Circuit Breaker Aberto | `RB-001: Circuit Breaker — Procedimento de Reativação` |
| A-002 — Latência p95 > 5s | `RB-002: Latência Elevada — Diagnóstico LLM e Ferramentas` |
| A-003 — Taxa de erro LLM > 10% | `RB-003: Degradação OpenAI — Contingência e Fallback` |
| A-004 — DLQ > 10 mensagens | `RB-004: Falha de Notificação — Reprocessamento de DLQ` |
| A-006 — HMAC ZapSign inválido | `RB-005: Webhook ZapSign Suspeito — Bloqueio e Investigação` |
| Rollback pós-deploy | `RB-006: Rollback de Deploy — Procedimento Padrão` |

> 💡 **Regra:** Todo alerta de severidade HIGH ou CRITICAL deve ter um procedimento correspondente no Runbook (D26). Alertas sem procedimento geram pânico operacional — não são úteis.

---

## 14. Backlog de Pendências

| # | Item | Tipo | Prioridade |
|---|---|---|---|
| P-OBS-001 | Definir ferramenta de agregação de logs centralizada (Loki vs Datadog vs Railway Logs nativo) para Fase 2 | [DECISÃO AUTÔNOMA: Railway Logs para Fase 1] confirmar para Fase 2 | Baixa (Fase 2) |
| P-OBS-002 | Configurar AlertManager no Grafana com integração PagerDuty para alertas CRITICAL fora do horário comercial | [SEÇÃO PENDENTE] — requer configuração de on-call schedule | Média |
| P-OBS-003 | Definir error budget policy formal: ações obrigatórias quando budget < 20% (congelar deploys, post-mortem) | [DECISÃO AUTÔNOMA: congelar deploys não-críticos] — formalizar com o time | Média |
| P-OBS-004 | Implementar tracing distribuído OpenTelemetry se arquitetura evoluir para multi-serviço | [SEÇÃO PENDENTE] — não aplicável na arquitetura atual (serviço único) | Baixa |

---

## 15. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — stack completa (Pino, Sentry, Langfuse, PostHog, Prometheus/Grafana). 5 níveis de log, schema JSON padrão, 14 eventos documentados, 14 métricas (aplicação + infra + negócio). SLOs definidos. 8 alertas com ação operacional. 4 dashboards. Retenção por tipo/ambiente. Health checks pós-deploy. Integração com Runbook D26. |
