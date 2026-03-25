# 25 - Observabilidade e Logs

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 25 - Observabilidade e Logs | v1.0 | 23/03/2026 | Claude Code Desktop | Aprovado |

---

> 📌 **TL;DR**
>
> - **Stack de observabilidade:** Pino (logs estruturados JSON) + Langfuse (tracing LLM) + Sentry (error tracking) + RabbitMQ metrics
> - **Correlation ID:** gerado por request, propagado em todos os logs, traces e filas
> - **Dados proibidos em logs:** CPF, email, telefone, OTP, JWT, cessionario_id raw (usar SHA-256 hash)
> - **10 eventos críticos mapeados** por módulo com nível, campos e ação operacional
> - **12 métricas** cobertas (aplicação, LLM, infra, negócio)
> - **8 alertas** com threshold, severidade, canal e ação imediata
> - **Retenção:** logs debug 7 dias; info 30 dias; warn/error 90 dias; Langfuse traces 90 dias
> - **SLO:** latência p95 ≤ 5s (individual), disponibilidade ≥ 99.5%

---

## 1. Arquitetura de Observabilidade

```
Request ──→ NestJS API
              │
              ├── Pino Logger ──────────────────────→ stdout (dev) / log aggregator (prod)
              │   [correlation_id, cessionario_id_hash, module, level, message, metadata]
              │
              ├── Langfuse ─────────────────────────→ cloud.langfuse.com
              │   [traces LLM: tokens, latency, tool calls, cost, session_id, user_hash]
              │
              ├── Sentry ────────────────────────────→ sentry.io
              │   [erros 4xx/5xx, exceptions, performance]
              │
              └── RabbitMQ metrics ─────────────────→ RabbitMQ Management API (porta 15672)
                  [queue depth, DLQ size, consumer lag]
```

| Ferramenta | Papel | Quando usar |
|---|---|---|
| **Pino** | Logs estruturados JSON — correlação, auditoria, debugging | Toda camada backend (NestJS) |
| **Langfuse** | Tracing de execuções LLM — tokens, latência, tool calls, custo | Todo trace do agente Dani |
| **Sentry** | Error tracking — exceções, performance, alertas P0/P1 | Backend + Frontend |
| **RabbitMQ Management** | Métricas de fila — profundidade, DLQ, consumer lag | Monitoramento de notificações |

---

## 2. Níveis de Log

| Nível | Quando usar | Campos obrigatórios adicionais | Retenção | Quem consome |
|---|---|---|---|---|
| `debug` | Execução interna detalhada — apenas em dev | `step`, `data` (sanitizado) | 7 dias | Dev local |
| `info` | Eventos de negócio normais — requests, tool calls, sessões | `event`, `module`, `latency_ms` | 30 dias | Backend Lead, monitoramento |
| `warn` | Condições anormais recuperáveis — rate limit, OTP incorreto, retry | `code`, `retry_count`, `recovery` | 90 dias | Backend Lead, DevOps |
| `error` | Falhas não recuperáveis — 5xx, exceções, DLQ | `error_code`, `stack` (dev apenas) | 90 dias | DevOps, Sentry |
| `fatal` | Sistema inoperável — desligamento Dani, banco inacessível | `error_code`, `service`, `impact` | 90 dias | DevOps, alerta imediato |

---

## 3. Formato de Log Estruturado

Schema JSON obrigatório em toda mensagem de log:

```json
{
  "timestamp": "2026-03-23T12:00:00.123Z",
  "level": "info",
  "service": "dani-api",
  "module": "AgenteModule",
  "message": "Agent response generated",
  "correlation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "request_id": "req_abc123",
  "session_id": "sess_xyz789",
  "cessionario_id": "a3f1e8b2...",
  "environment": "production",
  "event": "agent_response",
  "metadata": {
    "latency_ms": 2340,
    "tokens_in": 512,
    "tokens_out": 287,
    "model": "gpt-4o",
    "tools_called": ["buscarOportunidades"],
    "rag_chunks": 4
  }
}
```

**Exemplo de dado mascarado:**
```json
{
  "cessionario_id": "a3f1e8b2c4d6e8f0a1b2c3d4...",  // SHA-256 — NUNCA UUID raw
  "phone": "[Redacted]",                              // Pino redact
  "otp": "[Redacted]",                               // Pino redact
  "authorization": "[Redacted]"                       // Pino redact
}
```

---

## 4. O que Logar

| Evento | Módulo | Nível | Campos obrigatórios | Risco coberto | Ação operacional |
|---|---|---|---|---|---|
| Request recebido | todos | info | `correlation_id`, `method`, `path`, `cessionario_id_hash` | rastreabilidade end-to-end | baseline para investigação |
| Resposta LLM gerada | AgenteModule | info | `latency_ms`, `tokens_in`, `tokens_out`, `model`, `tools_called`, `session_id` | SLA e custo LLM | monitorar no Langfuse |
| Tool call executada | AgenteModule | info | `tool_name`, `latency_ms`, `success`, `cessionario_id_hash` | isolamento + performance | verificar se `cessionario_id` injetado |
| Fallback LLM ativado | AgenteModule | warn | `reason`, `retry_count`, `fallback_service` | disponibilidade Dani | checar status OpenAI |
| Rate limit atingido | AgenteModule | warn | `limit`, `window_s`, `cessionario_id_hash` | RN-DC-025 | normal — sem ação |
| Isolamento violado | AgenteModule | error | `error_code: AGENTE_004`, `cessionario_id_token`, `cessionario_id_resource` | segurança P0 | incidente imediato |
| OTP falha | WhatsappModule | warn | `phone_hash`, `attempt`, `remaining`, `block_ttl?` | brute-force | contagem de tentativas |
| OTP hard block ativado | WhatsappModule | warn | `phone_hash`, `block_ttl_s`, `reason: "5_consecutive_fails"` | brute-force | normal — TTL 1800s |
| Erro não mapeado (500) | AllExceptionsFilter | error | `correlation_id`, `error_code: INFRA_000`, `path` | bugs | investigar via Sentry |
| Dani desligada automaticamente | AgenteModule | fatal | `reason: "error_rate_30pct"`, `taxa_erro`, `session_count` | RN-DC-024 | reativar manualmente (Admin) |
| DLQ message recebida | NotificacaoService | error | `template_id`, `canal`, `retry_count` | confiabilidade notificações | reprocessar ou descartar manualmente |
| Calculo determinístico executado | CalculadoraModule | info | `opr_id`, `formula_aplicada`, `latency_ms` | auditoria cálculo | sem ação em caso normal |

---

## 5. O que Não Logar

| Dado proibido | Risco | Regra de masking |
|---|---|---|
| `cessionario_id` (UUID raw) | LGPD — dado pessoal associável | substituir por `sha256(cessionario_id)` |
| Número de telefone | LGPD | substituir por `sha256(phone)` |
| OTP em plain text | Segurança | `[Redacted]` via Pino redact |
| JWT (access ou refresh token) | Sequestro de sessão | `[Redacted]` via Pino redact |
| CPF / documento de identidade | LGPD — dado sensível | `[Redacted]` |
| Chaves de API (OpenAI, Supabase, etc.) | Segurança P0 | nunca em logs — apenas em `.env` |
| Stack traces completos | Exposição de lógica interna | apenas em `NODE_ENV=development`; em produção: apenas `error_code` e `message` |
| Dados do Cedente | RN-DC-002 — isolamento | nunca devem chegar ao logger |

**Configuração Pino redact obrigatória:**
```typescript
const logger = pino({
  redact: {
    paths: [
      'req.headers.authorization',
      'req.body.otp',
      'req.body.phone',
      '*.cpf',
      '*.email',
      'OPENAI_API_KEY'
    ],
    censor: '[Redacted]'
  }
})
```

---

## 6. Correlation ID e Rastreabilidade

### 6.1 Geração e Propagação

```typescript
// Gerado no início de cada request HTTP
// Middleware NestJS — executado antes de qualquer handler
@Injectable()
export class CorrelationIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction): void {
    const correlationId = req.headers['x-correlation-id'] as string ?? uuid()
    req.correlationId = correlationId
    res.setHeader('x-correlation-id', correlationId)  // retorna ao cliente
    next()
  }
}
```

### 6.2 Propagação em Todos os Pontos

| Ponto | Como propagar |
|---|---|
| Logs Pino | campo `correlation_id` em todos os logs |
| Langfuse trace | `metadata.correlation_id` no trace |
| Sentry error | `extra.correlation_id` no `captureException` |
| RabbitMQ message | campo `correlationId` no header da mensagem AMQP |
| Response de erro | campo `correlation_id` no body do error response |
| Tool calls externas | header `X-Correlation-ID` nas chamadas ao OpenAI (se suportado) |

> ⚙️ **Regra:** toda log line deve conter `correlation_id`. Log sem `correlation_id` em produção é bug de observabilidade.

---

## 7. Métricas

### 7.1 Métricas de Aplicação

| Métrica | Objetivo | Unidade | Target | Threshold alerta | Reação |
|---|---|---|---|---|---|
| `dani_request_latency_p95` | SLA de resposta individual | ms | ≤ 5000ms | > 5000ms = warn; > 10000ms = critical | checar OpenAI latência; verificar cache Redis |
| `dani_request_latency_comparison_p95` | SLA de comparação | ms | ≤ 10000ms | > 10000ms = warn | checar n° de oportunidades na comparação |
| `dani_error_rate` | Taxa de erro das respostas | % | ≤ 5% | > 10% = warn; > 30% = critical (desligamento) | investigar logs AgenteModule; checar OpenAI status |
| `dani_fallback_rate` | Frequência de ativação do fallback | % | ≤ 5% | > 5% = warn | checar conectividade OpenAI |
| `dani_rate_limit_hits` | Cessionários atingindo rate limit | count/hora | — | > 20%/sessão = warn | normal se uso elevado; anomalia se em massa |
| `dani_agent_status` | Estado operacional da Dani | enum | ACTIVE | FALLBACK/DESLIGADO = critical | reativar manualmente (Admin) |

### 7.2 Métricas de LLM (Langfuse)

| Métrica | Objetivo | Unidade | Frequência |
|---|---|---|---|
| `llm_tokens_per_session` | Custo por sessão | tokens | por sessão |
| `llm_cost_daily` | Custo diário OpenAI | USD | diário |
| `llm_tool_call_success_rate` | Eficácia das tools | % | por hora |
| `llm_confidence_score_avg` | Qualidade média das respostas | 0–1 | diário |

### 7.3 Métricas de Infraestrutura

| Métrica | Objetivo | Threshold alerta |
|---|---|---|
| `redis_memory_usage` | Saúde do Redis | > 80% = warn |
| `rabbitmq_queue_depth` | Backlog de notificações | `dani.notificacoes` > 100 = warn |
| `rabbitmq_dlq_depth` | Mensagens mortas | qualquer > 0 = error |
| `supabase_connection_pool` | Saturação do pool Prisma | > 8/10 = warn |

### 7.4 Métricas de Negócio

| Métrica | Objetivo |
|---|---|
| `sessions_per_day` | Volume de uso diário da Dani |
| `oportunidades_analisadas_per_session` | Profundidade de uso |
| `calculadora_calls_per_session` | Uso da Calculadora como standalone ou integrada |
| `whatsapp_vinculacoes_ativas` | Adoção do canal WhatsApp (Fase 2) |

---

## 8. SLI, SLO e Error Budget

| SLI | SLO | Medição | Error Budget (30 dias) |
|---|---|---|---|
| Latência p95 resposta individual | ≤ 5s em 99% das requests | `dani_request_latency_p95` | 0.1% das requests podem ultrapassar (≈ 43 min em 30 dias) |
| Latência p95 comparação | ≤ 10s em 99% das requests | `dani_request_latency_comparison_p95` | idem |
| Disponibilidade API | ≥ 99.5% dos minutos em 30 dias | uptime checks Sentry | 3h36min de downtime permitido |
| Taxa de erro IA | ≤ 5% das respostas | `dani_error_rate` | 5% de respostas com erro são aceitáveis |
| Entrega de notificações críticas | 100% das notificações críticas entregues | `alerta_tracking.failed` WHERE `prioridade='critico'` | zero falhas permitidas |

---

## 9. Alertas

| Alerta | Condição de disparo | Severidade | Canal | Responsável | Ação imediata |
|---|---|---|---|---|---|
| `DANI_AGENT_DOWN` | `dani_agent_status = DESLIGADO` (taxa erro > 30%) | Critical P0 | Sentry + Slack #alertas-criticos | DevOps + Tech Lead | Verificar logs AgenteModule; checar OpenAI status; se necessário reativar via Admin |
| `DANI_HIGH_ERROR_RATE` | `dani_error_rate > 10%` por 5 min | High P1 | Sentry | Backend Lead | Checar AgenteModule logs; possível degradação OpenAI |
| `DATA_ISOLATION_VIOLATED` | `error_code = AGENTE_004` em qualquer request | Critical P0 | Sentry + alerta imediato | Tech Lead + DevOps | Tratar como incidente de segurança; isolar instância se necessário |
| `DLQ_MESSAGE_RECEIVED` | Qualquer mensagem em `dani.notificacoes.dlq` | High P1 | Sentry | Backend Lead | Verificar logs do consumer; reprocessar ou descartar manualmente |
| `REDIS_UNAVAILABLE` | `redis_connection_error` em 3 tentativas consecutivas | High P1 | Sentry | DevOps | Verificar Upstash dashboard; fail open para rate limit (exceto OTP block) |
| `SUPABASE_POOL_SATURATED` | `supabase_connection_pool > 8/10` por 5 min | Warning P2 | Sentry | Backend Lead | Checar queries lentas; aumentar pool limit se necessário |
| `RABBITMQ_DLQ_CRITICAL_NOTIFICATION` | DLQ com `prioridade=critico` e `count > 0` | Critical P0 | Sentry + Slack #alertas-criticos | Backend Lead | Reprocessar manualmente imediatamente (ZapSign, Escrow) |
| `HIGH_LATENCY_SUSTAINED` | `dani_request_latency_p95 > 5s` por 5 min consecutivos | Warning P2 | Sentry | Backend Lead | Checar OpenAI latência; checar Redis; revisar logs de tool calls |

---

## 10. Dashboards

### Dashboard 1: Saúde Operacional da Dani

**Pergunta guia:** "A Dani está respondendo dentro do SLA e com baixa taxa de erro?"

| Widget | Métrica | Tipo |
|---|---|---|
| Status atual da Dani | `dani_agent_status` | Badge colorido (verde/amarelo/vermelho) |
| Latência p95 (última 1h) | `dani_request_latency_p95` | Gráfico linha com threshold |
| Taxa de erro (última 1h) | `dani_error_rate` | Gráfico linha com threshold 5% e 30% |
| Fallback rate | `dani_fallback_rate` | Número com tendência |
| Requests por minuto | `dani_requests_per_min` | Gráfico linha |

### Dashboard 2: LLM e Custo

**Pergunta guia:** "Quanto a Dani está custando e qual a qualidade das respostas?"

| Widget | Métrica | Tipo |
|---|---|---|
| Custo diário OpenAI | `llm_cost_daily` | Número com tendência |
| Tokens por sessão (média) | `llm_tokens_per_session` | Histograma |
| Tool calls success rate | `llm_tool_call_success_rate` | Gauge (%) |
| Confidence score médio | `llm_confidence_score_avg` | Gauge (0–1) |

### Dashboard 3: Infraestrutura

**Pergunta guia:** "Redis, RabbitMQ e Supabase estão saudáveis?"

| Widget | Métrica | Tipo |
|---|---|---|
| Redis memory | `redis_memory_usage` | Gauge com threshold 80% |
| RabbitMQ queue depth | `rabbitmq_queue_depth` por fila | Barras por fila |
| DLQ depth | `rabbitmq_dlq_depth` por DLQ | Número (alerta se > 0) |
| Supabase pool | `supabase_connection_pool` | Gauge 0–10 |

### Dashboard 4: Notificações e Alertas

**Pergunta guia:** "As notificações estão sendo entregues? Há mensagens mortas?"

| Widget | Métrica | Tipo |
|---|---|---|
| Taxa de entrega por canal | `alerta_tracking` (`sent` / `delivered`) | Gauge por canal |
| DLQ crítica | `dani.notificacoes.dlq` + `prioridade=critico` | Contador alarme |
| Notificações enviadas hoje | `alerta_tracking.sent` count | Número |
| Taxa de leitura in-app | `alerta_tracking.read/delivered` | % |

---

## 11. Retenção e Custos

| Tipo de dado | Ambiente | Retenção | Justificativa |
|---|---|---|---|
| Logs `debug` | dev | 7 dias | efêmero, sem valor operacional |
| Logs `info` | prod/staging | 30 dias | investigação operacional normal |
| Logs `warn` / `error` | prod/staging | 90 dias | janela de investigação de incidentes |
| Langfuse traces | prod | 90 dias | alinhado ao histórico de conversas (D01) |
| `alerta_tracking` (banco) | prod | 90 dias | LGPD + auditoria de notificações |
| Sentry events | prod | 30 dias (plano free) | suficiente para bugs ativos; escalar com plano se necessário |
| Audit log (acesso isolamento) | prod | 1 ano | [DECISÃO AUTÔNOMA] 1 ano para cobertura de potenciais investigações jurídicas. Alternativa descartada: 90 dias — insuficiente para auditoria de compliance com LGPD. Critério: risco jurídico de violação de isolamento de dados. |

> ⚙️ **Custo de retenção:** logs em stdout são gerenciados pelo serviço de deploy (Railway/Vercel). Langfuse tem plano gratuito com 50k eventos/mês — escalar para plano pago se volume superar. Sentry plano Developer gratuito é suficiente para Fase 1.

---

## 12. Health Checks e Pós-Deploy

Após qualquer deploy ou go-live, execute os seguintes checks antes de marcar como saudável:

```bash
# 1. API responde com status ok
curl https://api.dani.repasseseguro.com.br/api/v1/health
# Esperado: {"status":"ok","services":{"database":"ok","redis":"ok","rabbitmq":"ok"}}

# 2. Latência da Dani dentro do SLA
curl -w "%{time_total}" -o /dev/null https://api.dani.../api/v1/health
# Esperado: < 1.0s

# 3. Logs fluindo (verificar no aggregator)
# Esperado: logs info com correlation_id presentes nos últimos 5 min

# 4. Langfuse recebendo traces
# Verificar: dashboard Langfuse → última trace < 5 min

# 5. Redis funcional
redis-cli -h $REDIS_HOST -a $REDIS_AUTH ping
# Esperado: PONG

# 6. RabbitMQ sem DLQ messages
curl -u guest:guest http://localhost:15672/api/queues/%2F/dani.notificacoes.dlq
# Esperado: {"messages":0,...}
```

| Sinal | Threshold | Interpretação de falha |
|---|---|---|
| API `/health` → 200 | obrigatório | serviço não subiu ou banco inacessível — rollback |
| Latência health < 1s | obrigatório | possível deadlock ou conexão lenta ao banco |
| Langfuse trace presente | desejável | observabilidade LLM não configurada — investigar env vars |
| DLQ = 0 | obrigatório | mensagem morta no DLQ = problema no consumer — investigar |

---

## 13. Integração com Runbook

| Alerta | Procedimento no Runbook |
|---|---|
| `DANI_AGENT_DOWN` | Runbook §3: Reativação Manual da Dani |
| `DATA_ISOLATION_VIOLATED` | Runbook §1: Resposta a Incidente de Segurança |
| `RABBITMQ_DLQ_CRITICAL_NOTIFICATION` | Runbook §5: Reprocessamento de Mensagens Mortas |
| `REDIS_UNAVAILABLE` | Runbook §4: Recuperação do Redis |
| `SUPABASE_POOL_SATURATED` | Runbook §6: Escalabilidade do Pool de Banco |

> 💡 O Runbook (D26) deve referenciar este documento para obter os comandos de diagnóstico e thresholds de cada alerta.

---

## 14. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Retenção audit log 1 ano | [DECISÃO AUTÔNOMA] 1 ano. Alternativa descartada: 90 dias — insuficiente para investigação de compliance LGPD. Critério: risco jurídico de violação de isolamento de dados (AGENTE_004). | §11 | Compliance | P1 | DevOps | Concluído |
| Log aggregator em produção | [DEFINIÇÃO PENDENTE] Opção A: Railway built-in log shipping (simples, sem custo extra). Opção B: Datadog (mais poderoso, custo ~$15/host/mês). Trade-off: A é suficiente para Fase 1; B tem alertas e dashboards nativos. | §1 | Operação | P2 | DevOps | Pendente |
| D24 (Deploy, CI-CD) não gerado ainda | [DECISÃO AUTÔNOMA] Referências a deploy/CI-CD assumem Railway + GitHub Actions (inferido de D02 Stacks e D22). Quando D24 for gerado, revisar esta seção para consistência. | §12 | Consistência | P2 | Backend Lead | Pendente revisão após D24 |
