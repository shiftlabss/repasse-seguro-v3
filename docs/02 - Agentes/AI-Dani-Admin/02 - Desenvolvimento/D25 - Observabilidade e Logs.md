# Observabilidade e Logs — AI-Dani-Admin

## Estratégia de Observabilidade, Logging e Monitoramento

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend e DevOps |
| Escopo | Configuração de Pino, Langfuse, Sentry, PostHog e dashboards operacionais do módulo AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D02 (Stacks), D14 (Especificações Técnicas), D17 (Integrações Externas), D20 (Error Handling) |

---

> **📌 TL;DR**
>
> - **Logs estruturados:** Pino (JSON) com redact de PII. Nível `debug` em dev, `info` em staging/prod.
> - **LLM observabilidade:** Langfuse — trace por request, métricas de tokens, latência e custo. Fire-and-forget.
> - **Erros de infra:** Sentry — captura automática via `GlobalHttpExceptionFilter` (só 5xx). Sem PII.
> - **Analytics e feature flags:** PostHog — eventos de negócio (takeover_started, alert_dispatched, config_updated).
> - **Métricas de produto:** banco de dados (volume, CSAT, top_questions). Nunca dependente de Langfuse para negócio crítico.
> - **Alertas operacionais:** Slack + e-mail para P0/P1 automáticos. SLA: P0 < 5min, P1 < 15min.

---

## 1. Pilha de Observabilidade

| Camada | Ferramenta | Propósito | Criticidade |
|---|---|---|---|
| Logs estruturados | Pino (NestJS) | Rastreamento de requests, eventos de negócio, erros | Alta |
| LLM observabilidade | Langfuse | Traces de IA, tokens, custo, latência de modelos | Média |
| Erros de infra | Sentry | Captura de exceções 5xx, alertas críticos de prod | Alta |
| Analytics/Feature flags | PostHog | Eventos de uso, kill switches, funnels | Média |
| Métricas de produto | PostgreSQL | Volume, CSAT, top questions — fonte de verdade | Alta |
| Health check | `/health` (NestJS) | Liveness + readiness do container | Alta |

---

## 2. Pino — Logs Estruturados

### 2.1 Configuração

```typescript
// apps/api/src/common/logger/pino.config.ts

import { PinoLogger } from 'nestjs-pino'

export const pinoConfig = {
  pinoHttp: {
    level: process.env.LOG_LEVEL ?? 'info',
    transport:
      process.env.NODE_ENV !== 'production'
        ? { target: 'pino-pretty', options: { colorize: true } }
        : undefined,

    // Redact de PII — nunca logar dados pessoais
    redact: {
      paths: [
        'req.headers.authorization',
        'req.body.password',
        'req.body.email',
        'req.body.cpf',
        'req.body.phone',
        'res.headers["set-cookie"]',
        '*.email',
        '*.cpf',
        '*.phone',
      ],
      censor: '[REDACTED]',
    },

    // Serializers padrão
    serializers: {
      req: (req) => ({
        method: req.method,
        url: req.url,
        correlationId: req.headers['x-correlation-id'],
      }),
      res: (res) => ({
        statusCode: res.statusCode,
      }),
    },

    // Campos base em todos os logs
    base: {
      module: 'ai-dani-admin',
      env: process.env.NODE_ENV,
    },

    // Customizar o campo de nível
    messageKey: 'msg',
    timestamp: () => `,"time":"${new Date().toISOString()}"`,
  },
}
```

### 2.2 Níveis de Log por Ambiente

| Ambiente | Nível | Comportamento |
|---|---|---|
| `development` | `debug` | Todos os logs incluindo debug de queries Prisma |
| `staging` | `info` | Info, warn, error. Sem debug. |
| `production` | `info` | Info, warn, error. Sem debug. |

### 2.3 Campos Obrigatórios em Logs de Negócio

Todo log de evento de negócio deve incluir:

```typescript
// ✅ Estrutura correta para log de evento de negócio
this.logger.log({
  event: 'TAKEOVER_STARTED',       // nome do evento (UPPER_SNAKE_CASE)
  adminId: admin.id,                // UUID — nunca e-mail ou nome
  interactionId: interaction.id,    // UUID
  agentId: interaction.agentId,     // UUID
  durationMs: Date.now() - start,  // latência da operação
  // Nunca: email, cpf, nome, telefone
})
```

### 2.4 Campos Proibidos em Logs

```typescript
// ❌ PROIBIDO — dados pessoais identificáveis (PII)
this.logger.log(`Admin ${adminEmail} iniciou takeover`)      // e-mail
this.logger.log(`CPF do usuário: ${user.cpf}`)               // CPF
this.logger.log(`Mensagem: ${interaction.userMessage}`)      // conteúdo de conversa

// ✅ CORRETO — usar IDs opacos
this.logger.log({ event: 'TAKEOVER_STARTED', adminId, interactionId })
```

### 2.5 Correlation ID

Cada request recebe um `correlationId` via header `X-Correlation-Id`. Gerado automaticamente se não fornecido:

```typescript
// apps/api/src/common/middleware/correlation-id.middleware.ts

import { Injectable, NestMiddleware } from '@nestjs/common'
import { randomUUID } from 'crypto'

@Injectable()
export class CorrelationIdMiddleware implements NestMiddleware {
  use(req: any, res: any, next: () => void) {
    req.correlationId = req.headers['x-correlation-id'] ?? randomUUID()
    res.setHeader('X-Correlation-Id', req.correlationId)
    next()
  }
}
```

---

## 3. Langfuse — Observabilidade de LLM

### 3.1 Integração

```typescript
// apps/api/src/common/langfuse/langfuse.service.ts

import { Injectable } from '@nestjs/common'
import { Langfuse } from 'langfuse'

@Injectable()
export class LangfuseService {
  private readonly client: Langfuse

  constructor() {
    this.client = new Langfuse({
      publicKey: process.env.LANGFUSE_PUBLIC_KEY,
      secretKey: process.env.LANGFUSE_SECRET_KEY,
      baseUrl: process.env.LANGFUSE_HOST ?? 'https://cloud.langfuse.com',
      flushAt: 10,    // flush a cada 10 eventos
      flushInterval: 5000,  // ou a cada 5s
    })
  }

  // Criar trace por request de IA
  createTrace(params: {
    name: string
    userId: string
    sessionId: string
    agentType: 'DANI_CESSIONARIO' | 'DANI_CEDENTE'
    metadata?: Record<string, unknown>
  }) {
    return this.client.trace({
      name: params.name,
      userId: params.userId,
      sessionId: params.sessionId,
      tags: [params.agentType],
      metadata: params.metadata,
    })
  }

  // Registrar uma geração (chamada ao modelo)
  logGeneration(trace: any, params: {
    name: string
    model: string
    promptTokens: number
    completionTokens: number
    latencyMs: number
    confidence: number
    input: string
    output: string
  }) {
    // Fire-and-forget — não bloqueia a resposta ao usuário
    trace.generation({
      name: params.name,
      model: params.model,
      usage: {
        input: params.promptTokens,
        output: params.completionTokens,
        total: params.promptTokens + params.completionTokens,
      },
      latency: params.latencyMs,
      input: params.input,   // prompt enviado (sem PII do usuário)
      output: params.output, // resposta do modelo
      metadata: {
        confidence: params.confidence,
      },
    })
  }

  async shutdown() {
    await this.client.shutdownAsync()
  }
}
```

### 3.2 Uso no AgentService

```typescript
// Dentro de AgentService.chat()

const trace = this.langfuse.createTrace({
  name: 'dani-chat',
  userId: adminId,
  sessionId: interactionId,
  agentType: agent.type,
})

const start = Date.now()
const response = await this.openai.chat.completions.create({ ... })

// Fire-and-forget — não bloqueia resposta
this.langfuse.logGeneration(trace, {
  name: 'openai-completion',
  model: 'gpt-4o',
  promptTokens: response.usage.prompt_tokens,
  completionTokens: response.usage.completion_tokens,
  latencyMs: Date.now() - start,
  confidence: calculatedConfidence,
  input: systemPrompt,  // não inclui mensagem do usuário (PII)
  output: response.choices[0].message.content,
})
```

### 3.3 Métricas Disponíveis no Langfuse

| Métrica | Tipo | Uso |
|---|---|---|
| `usage.input` | tokens | Custo de entrada por request |
| `usage.output` | tokens | Custo de saída por request |
| `latency` | ms | Latência de geração do modelo |
| `metadata.confidence` | float 0-1 | Score de confiança calculado |
| Tags por `agentType` | string | Separar métricas por agente |

> **Importante:** Langfuse é fonte de métricas de LLM (tokens, custo, latência de modelo). Métricas de negócio (volume, CSAT, top questions) são armazenadas no PostgreSQL via `interactions` table — nunca dependentes de disponibilidade do Langfuse.

---

## 4. Sentry — Rastreamento de Erros

### 4.1 Configuração

```typescript
// apps/api/src/main.ts

import * as Sentry from '@sentry/node'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  integrations: [
    Sentry.prismaIntegration(),    // queries lentas do Prisma
    Sentry.httpIntegration(),      // requests HTTP
  ],
  // Nunca capturar PII
  beforeSend(event) {
    // Remover dados pessoais do payload
    if (event.request?.data) {
      delete event.request.data.email
      delete event.request.data.cpf
      delete event.request.data.password
    }
    return event
  },
})
```

### 4.2 Regras de Captura

| Tipo de erro | Capturar no Sentry | Nível |
|---|---|---|
| `5xx` — erros de infra | Sim | `error` |
| `4xx` — erros do cliente | Não | `warn` (só Pino) |
| Erros de consumer RabbitMQ | Sim | `error` |
| Erros de scheduler (AlertsScheduler) | Sim | `error` |
| Falha de health check dependency | Sim | `fatal` |

```typescript
// No GlobalHttpExceptionFilter (D20) — só captura 5xx
if (status >= 500) {
  Sentry.captureException(exception, {
    extra: {
      correlationId: request.correlationId,
      path: request.url,
      adminId: request.user?.sub,  // UUID apenas, nunca e-mail
    },
  })
}
```

### 4.3 Contexto nos Erros

```typescript
// Em consumers RabbitMQ — captura com contexto completo
try {
  await this.alertsService.dispatch(message)
} catch (error) {
  Sentry.withScope((scope) => {
    scope.setTag('queue', 'dani-admin.alerts')
    scope.setExtra('messageId', message.id)
    scope.setExtra('alertType', message.type)
    // Nunca setExtra com PII
    Sentry.captureException(error)
  })
  throw error  // relança para DLQ
}
```

---

## 5. PostHog — Analytics e Feature Flags

### 5.1 Eventos de Negócio

```typescript
// apps/api/src/common/analytics/posthog.service.ts

import { Injectable } from '@nestjs/common'
import { PostHog } from 'posthog-node'

@Injectable()
export class PostHogService {
  private readonly client: PostHog

  constructor() {
    this.client = new PostHog(process.env.POSTHOG_API_KEY, {
      host: process.env.POSTHOG_HOST ?? 'https://app.posthog.com',
      flushAt: 20,
      flushInterval: 10000,
    })
  }

  capture(event: {
    distinctId: string      // adminId (UUID)
    event: PostHogEvent
    properties?: Record<string, unknown>
  }) {
    // Fire-and-forget
    this.client.capture({
      distinctId: event.distinctId,
      event: event.event,
      properties: {
        ...event.properties,
        $lib: 'dani-admin-api',
        env: process.env.NODE_ENV,
      },
    })
  }

  async isFeatureEnabled(flag: FeatureFlag, adminId: string): Promise<boolean> {
    return this.client.isFeatureEnabled(flag, adminId) ?? false
  }
}

export type PostHogEvent =
  | 'takeover_started'
  | 'takeover_ended'
  | 'alert_dispatched'
  | 'agent_config_updated'
  | 'launch_readiness_authorized'
  | 'launch_readiness_revoked'
  | 'interaction_flagged'
  | 'dashboard_accessed'

export type FeatureFlag =
  | 'webchat-enabled'
  | 'dani-cessionario-enabled'
  | 'dani-cedente-enabled'
```

### 5.2 Tabela de Eventos

| Evento | Trigger | Propriedades |
|---|---|---|
| `takeover_started` | Admin inicia takeover | `interactionId`, `agentId`, `agentType` |
| `takeover_ended` | Admin encerra takeover | `interactionId`, `durationMs`, `agentId` |
| `alert_dispatched` | Alerta enviado com sucesso | `alertType`, `channels`, `agentId` |
| `agent_config_updated` | Threshold ou rate limit alterado | `agentId`, `field`, `oldValue`, `newValue` |
| `launch_readiness_authorized` | Agente autorizado para go-live | `agentId`, `agentType` |
| `launch_readiness_revoked` | Autorização revogada | `agentId`, `reason` |
| `interaction_flagged` | Interação sinalizada para revisão | `interactionId`, `agentId` |
| `dashboard_accessed` | Dashboard carregado | `adminId`, `filters` aplicados |

### 5.3 Feature Flags como Kill Switches

```typescript
// No AgentService — verificar antes de processar request de IA
const isEnabled = await this.posthog.isFeatureEnabled(
  'webchat-enabled',
  adminId,
)

if (!isEnabled) {
  throw new ServiceUnavailableException({
    error: {
      code: 'DA-AGT-001',
      message: 'O serviço de IA está temporariamente desabilitado.',
    },
  })
}
```

| Flag | Desabilita | Quando usar |
|---|---|---|
| `webchat-enabled` | Todo o webchat (ambos agentes) | Incidente crítico de IA |
| `dani-cessionario-enabled` | Apenas Dani-Cessionário | Problema isolado no agente de cessionário |
| `dani-cedente-enabled` | Apenas Dani-Cedente | Problema isolado no agente de cedente |

---

## 6. Health Check

### 6.1 Endpoint

```
GET /health
```

Resposta esperada (HTTP 200):

```json
{
  "status": "ok",
  "timestamp": "2026-03-23T10:00:00.000Z",
  "dependencies": {
    "database": "ok",
    "redis": "ok",
    "rabbitmq": "ok"
  }
}
```

Resposta degradada (HTTP 503):

```json
{
  "status": "degraded",
  "timestamp": "2026-03-23T10:00:00.000Z",
  "dependencies": {
    "database": "ok",
    "redis": "error",
    "rabbitmq": "ok"
  }
}
```

### 6.2 Implementação

```typescript
// apps/api/src/health/health.controller.ts

import { Controller, Get } from '@nestjs/common'
import {
  HealthCheck,
  HealthCheckService,
  PrismaHealthIndicator,
} from '@nestjs/terminus'
import { InjectRedis } from '@nestjs-modules/ioredis'
import Redis from 'ioredis'

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private prisma: PrismaHealthIndicator,
    @InjectRedis() private redis: Redis,
  ) {}

  @Get()
  @HealthCheck()
  async check() {
    return this.health.check([
      () => this.prisma.pingCheck('database'),
      async () => {
        const pong = await this.redis.ping()
        return {
          redis: { status: pong === 'PONG' ? 'up' : 'down' },
        }
      },
    ])
  }
}
```

---

## 7. Métricas de Produto (PostgreSQL)

As métricas de negócio são armazenadas diretamente no PostgreSQL via tabela `interactions`. O endpoint `GET /api/v1/admin/metrics` agrega estas métricas em tempo real (com cache Redis TTL 60s).

### 7.1 Métricas Disponíveis

| Métrica | Fonte | Atualização |
|---|---|---|
| `totalInteractions` | `COUNT(interactions)` | Real-time |
| `resolvedByAI` | `COUNT WHERE status = RESPONDIDA_PELA_IA` | Real-time |
| `flaggedForReview` | `COUNT WHERE status = SINALIZADA_PARA_REVISAO` | Real-time |
| `takeoverCount` | `COUNT(takeovers)` | Real-time |
| `avgConfidenceScore` | `AVG(interactions.confidence_score)` | Real-time |
| `avgResponseTimeMs` | `AVG(interactions.response_time_ms)` | Real-time |
| `csatScore` | `AVG(interactions.csat_score) WHERE csat_score IS NOT NULL` | Real-time |
| `topQuestions` | `GROUP BY normalized_question ORDER BY count DESC LIMIT 10` | Cache 5min |

### 7.2 Separação de Responsabilidades

```
PostgreSQL (métricas de negócio)
├── volume de interações
├── taxas de resolução/sinalização/takeover
├── CSAT e NPS
└── top perguntas

Langfuse (métricas de LLM)
├── tokens consumidos por modelo
├── custo por request
├── latência de geração (p50, p95, p99)
└── distribuição de confidence score

Sentry (métricas de infra)
├── taxa de erros 5xx
├── requests lentos (tracing)
└── alertas de anomalias

PostHog (métricas de produto/uso)
├── funnels de takeover
├── feature flag adoption
└── padrões de uso por admin
```

---

## 8. Dashboards e Alertas Operacionais

### 8.1 Dashboard de Saúde (recomendado: Grafana)

Conectar ao PostgreSQL e exportar métricas para Grafana ou ferramenta similar.

| Painel | Fonte | Alerta |
|---|---|---|
| Taxa de erros 5xx | Sentry | > 1% por 5min → Slack P1 |
| Latência API p95 | Pino (response time) | > 2s por 5min → Slack P1 |
| Consumer RabbitMQ lag | RabbitMQ Management API | > 100 msgs → Slack P1 |
| Health check falho | `/health` endpoint | Qualquer falha → Slack P0 |
| Taxa de CSAT < 3.0 | PostgreSQL | Média 24h < 3.0 → Slack P1 |
| Confidence score < threshold | PostgreSQL | Média 1h < 50% → Slack P1 |

### 8.2 SLAs de Alerta Operacional

| Severidade | Critério | Canal | Tempo de Resposta |
|---|---|---|---|
| P0 | Serviço fora do ar, database down | Slack #alertas-criticos + e-mail | < 5 min |
| P1 | Erros 5xx > 1%, latência > 2s, consumer lag | Slack #alertas-prod | < 15 min |
| P2 | CSAT degradado, confidence baixo | Slack #alertas-prod | < 1h |

### 8.3 Consultas Operacionais Úteis

```sql
-- Taxa de erros nas últimas 1h (via tabela de logs se aplicável)
-- Monitorar via Sentry dashboard diretamente

-- Interações por status nas últimas 24h
SELECT status, COUNT(*) as total
FROM interactions
WHERE created_at > NOW() - INTERVAL '24 hours'
GROUP BY status;

-- Confidence score médio por agente (últimas 6h)
SELECT
  a.type as agent_type,
  ROUND(AVG(i.confidence_score)::numeric, 3) as avg_confidence,
  COUNT(*) as total_interactions
FROM interactions i
JOIN agents a ON a.id = i.agent_id
WHERE i.created_at > NOW() - INTERVAL '6 hours'
GROUP BY a.type;

-- Top 10 interações com menor confidence (últimas 24h)
SELECT id, agent_id, confidence_score, created_at
FROM interactions
WHERE created_at > NOW() - INTERVAL '24 hours'
ORDER BY confidence_score ASC
LIMIT 10;

-- Consumer lag no RabbitMQ (via Management API)
-- GET http://guest:guest@localhost:15672/api/queues/%2F/dani-admin.alerts.queue
```

---

## 9. Estratégia de Retenção de Logs

| Dados | Retenção | Onde |
|---|---|---|
| Logs de aplicação (Pino) | 30 dias | CloudWatch / Loki |
| Traces Langfuse | 90 dias | Langfuse cloud |
| Erros Sentry | 90 dias | Sentry cloud |
| Eventos PostHog | 1 ano | PostHog cloud |
| Métricas de produto (banco) | Indefinido | PostgreSQL |
| Logs de auditoria (`admin_access_logs`) | Conforme LGPD (mínimo 5 anos) | PostgreSQL |

> **LGPD:** Logs de auditoria de ações administrativas são retidos conforme exigência regulatória. Logs técnicos de infra (Pino, Sentry) têm retenção menor e não contêm PII.

---

## 10. Checklist de Observabilidade por Deploy

Antes de qualquer deploy em produção:

- [ ] `LANGFUSE_PUBLIC_KEY` e `LANGFUSE_SECRET_KEY` configurados nos secrets do ambiente
- [ ] `SENTRY_DSN` configurado — testar com `Sentry.captureMessage('test')` em staging
- [ ] `POSTHOG_API_KEY` configurado — verificar events chegando no dashboard PostHog
- [ ] `LOG_LEVEL=info` em staging e production
- [ ] `SENTRY_TRACES_SAMPLE_RATE=0.1` em production (não 1.0)
- [ ] `/health` retornando 200 com todas as dependencies `ok`
- [ ] Alertas Slack configurados para P0/P1 no canal correto
- [ ] Pino redact cobrindo todos os campos PII definidos na seção 2.4

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Pino + Langfuse + Sentry + PostHog + health check + métricas de produto + dashboards. |
