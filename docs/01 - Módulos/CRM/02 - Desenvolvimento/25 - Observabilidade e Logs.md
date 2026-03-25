# 25 - Observabilidade e Logs

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — Backend, DevOps |
| **Escopo** | Stack de observabilidade, logs estruturados, alertas, métricas de SLA e conformidade LGPD |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | 02 Stacks CRM · 24 Deploy CI-CD |

---

> **TL;DR**
>
> - **Stack:** Sentry (errors) + PostHog (analytics + session replay) + Pino (logs estruturados JSON) + Railway Metrics (infra).
> - **Logs:** JSON estruturado no Railway. Retenção 30 dias em produção. NUNCA logar PII (CPF, nome, e-mail).
> - **Alertas:** Sentry para P0/P1 com notificação Slack + e-mail imediata (SLA de 15 min para P0).
> - **Health check:** `/health` valida banco, Redis e RabbitMQ.
> - **Langfuse (Fase 2):** rastreamento de chamadas ao agente Dani-Admin — não ativo na v1.0.

---

## 1. Stack de Observabilidade

| Ferramenta | Camada | Propósito no CRM |
|---|---|---|
| **Sentry** | Frontend + Backend | Error tracking com stack traces, breadcrumbs de sessão, alertas automáticos |
| **PostHog** | Frontend | Analytics de produto: uso do pipeline, abertura de Casos, avanço de estágios, session replay |
| **Pino** | Backend | Logs estruturados JSON com request ID propagado, timestamp UTC, nível e contexto |
| **Railway Metrics** | Infraestrutura | CPU, memória, latência de rede e restarts do serviço CRM API |
| **Supabase Dashboard** | Banco de dados | Slow queries, conexões ativas, uso de storage do Dossiê |
| **CloudAMQP Console** | Filas | Profundidade de filas, taxa de mensagens, dead-letter queue |
| **Upstash Console** | Cache | Hit rate, comandos/s, uso de memória |
| **Langfuse** | IA (Fase 2) | Rastreamento de chamadas LLM do agente Dani-Admin — não ativo na v1.0 |

---

## 2. Logs Estruturados (Pino)

### 2.1 Configuração

```typescript
// apps/api/src/common/logger/pino.config.ts
import { LoggerModule } from 'nestjs-pino';

export const PinoLoggerConfig = LoggerModule.forRoot({
  pinoHttp: {
    level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
    transport: process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty', options: { colorize: true } }
      : undefined,  // JSON puro em produção (Railway coleta)
    serializers: {
      req: (req) => ({
        method: req.method,
        url: req.url,
        requestId: req.id,
        // NUNCA incluir: headers de autenticação, body completo, query params com PII
      }),
      res: (res) => ({
        statusCode: res.statusCode,
      }),
    },
    customProps: (req) => ({
      requestId: req.headers['x-request-id'] ?? req.id,
      module: 'crm',
    }),
    redact: {
      paths: [
        'req.headers.authorization',
        'req.headers.cookie',
        'req.body.password',
        'req.body.cpf',
        'req.body.email',
        'req.body.nome',
        'req.body.name',
      ],
      censor: '[REDACTED]',
    },
  },
});
```

### 2.2 Níveis de Log

| Nível | Quando usar | Exemplo no CRM |
|---|---|---|
| `ERROR` | Exceção não tratada, falha de integração crítica, dado não persistido | ZapSign retornou erro 500; migration falhou; RabbitMQ conexão perdida |
| `WARN` | Situação anormal mas recuperável; degradação de serviço | Retry #2 de notificação; fila com alta profundidade; cache miss acima do esperado |
| `INFO` | Eventos de negócio importantes; lifecycle de requisição | Caso avançado para MATCH; relatório semanal gerado; usuário autenticado |
| `DEBUG` | Informação detalhada para diagnóstico — desativado em produção | Query Prisma com parâmetros; payload de webhook recebido; decisão de rota |

### 2.3 Formato de Log em Produção

```json
{
  "level": "info",
  "time": "2026-03-23T18:45:22.312Z",
  "pid": 1,
  "hostname": "crm-api-railway-abc123",
  "requestId": "req-uuid-v4-aqui",
  "module": "crm",
  "context": "CasesService",
  "msg": "Caso avançado para estágio MATCH",
  "caseId": "case-uuid-aqui",
  "fromStatus": "PUBLICACAO",
  "toStatus": "MATCH",
  "userId": "user-uuid-aqui",
  "durationMs": 42
}
```

> **Regra absoluta:** nunca incluir nos logs: CPF, e-mail, nome completo, endereço, telefone, dados bancários de Cedentes/Cessionários. Use apenas UUIDs para identificação.

### 2.4 Padrão de Log nos Serviços NestJS

```typescript
// Injetar PinoLogger em todo service que precisar de log
@Injectable()
export class CasesService {
  constructor(private readonly logger: PinoLogger) {
    this.logger.setContext(CasesService.name);
  }

  async advanceStatus(caseId: string, toStatus: CaseStatus, userId: string): Promise<Case> {
    this.logger.info({ caseId, toStatus, userId }, 'Iniciando transição de status de Caso');

    try {
      const result = await this.performTransition(caseId, toStatus, userId);
      this.logger.info({ caseId, fromStatus: result.previousStatus, toStatus, durationMs: result.duration }, 'Caso avançado com sucesso');
      return result.case;
    } catch (error) {
      this.logger.error({ caseId, toStatus, error: error.message }, 'Falha na transição de status do Caso');
      throw error;
    }
  }
}
```

---

## 3. Sentry — Error Tracking

### 3.1 Configuração Backend (NestJS)

```typescript
// apps/api/src/main.ts
import * as Sentry from '@sentry/nestjs';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  integrations: [nodeProfilingIntegration()],
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  profilesSampleRate: 0.1,
  beforeSend(event) {
    // Remover PII antes de enviar ao Sentry
    if (event.user) {
      delete event.user.email;
      delete event.user.username;
      delete event.user.ip_address;
    }
    return event;
  },
});
```

### 3.2 Configuração Frontend (Next.js)

```typescript
// apps/web-crm/sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.05,  // 5% de sessões gravadas
  replaysOnErrorSampleRate: 1.0,   // 100% de sessões com erro gravadas
  integrations: [
    Sentry.replayIntegration({
      maskAllText: true,       // Mascara todo texto (PII protection)
      blockAllMedia: false,
    }),
  ],
});
```

### 3.3 Classificação de Erros

| Nível Sentry | Impacto | Exemplo no CRM | SLA de Resposta |
|---|---|---|---|
| **Fatal** | Sistema indisponível | API em crash loop; banco inacessível | 15 min |
| **Error (P0)** | Feature crítica quebrada | ZapSign não cria envelope; pagamento escrow falha; login impossível | 15 min |
| **Error (P1)** | Feature importante degradada | Relatório semanal não gerado; alerta SLA não disparado | 1 hora |
| **Warning (P2)** | Degradação parcial | Retry de notificação; timeout de integração não crítica | 4 horas |
| **Info** | Evento de diagnóstico | Fallback ativado; cache miss esperado | 24 horas |

### 3.4 Alertas Sentry → Slack

```
Configuração no Sentry:
  Projeto: crm-api + crm-frontend
  Regra: "Send alert when new issue is created with level = error OR fatal"
  Destino: Slack #crm-incidents (integração Sentry → Slack)
  E-mail: tech-lead@repasseseguro.com.br (P0/P1 apenas)

Formato da notificação Slack:
  🔴 [P0] CRM API — NullReferenceException in CasesService.advanceStatus
  Ambiente: production | Ocorrências: 3 | Usuários afetados: 2
  [Ver no Sentry] [Silenciar por 1h]
```

---

## 4. PostHog — Analytics

### 4.1 Eventos Rastreados

| Evento | Quando disparar | Propriedades (sem PII) |
|---|---|---|
| `case_created` | Caso criado no CRM | `case_id`, `initial_status`, `created_by_role` |
| `case_status_advanced` | Caso avança de estágio | `case_id`, `from_status`, `to_status`, `duration_in_stage_days` |
| `case_closed` | Caso concluído | `case_id`, `total_duration_days`, `close_reason` |
| `pipeline_filter_applied` | Usuário aplica filtro no pipeline | `filter_type`, `filter_value` (sem dados pessoais) |
| `activity_registered` | Atividade registrada (ligação, reunião, WhatsApp) | `activity_type`, `case_id` |
| `report_generated` | Relatório gerado manualmente ou via agendamento | `report_type`, `period`, `generated_by_role` |
| `sla_alert_viewed` | Usuário visualiza alerta de SLA | `alert_id`, `severity`, `days_overdue` |
| `dossier_document_uploaded` | Documento enviado ao Dossiê | `document_type`, `case_id` |

### 4.2 Identificação de Usuário

```typescript
// Identificação por papel — sem dados pessoais
posthog.identify(userId, {
  role: user.role,           // ADMIN_RS, COORDENADOR_RS, etc.
  team_id: user.team_id,    // Para segmentação por equipe
  // NUNCA incluir: email, nome, CPF, telefone
});
```

### 4.3 Session Replay

Session replay está configurado no `@sentry/nextjs` (não PostHog) para o CRM. Regra: `maskAllText: true` — todo texto é mascarado por padrão para proteção de PII de Cedentes/Cessionários que possam aparecer na tela.

---

## 5. Health Check

### 5.1 Endpoint `/health`

```typescript
// apps/api/src/health/health.controller.ts
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: PrismaHealthIndicator,
    private readonly redis: RedisHealthIndicator,
    private readonly rabbitmq: RabbitMQHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.redis.pingCheck('redis'),
      () => this.rabbitmq.pingCheck('rabbitmq'),
    ]);
  }
}
```

**Resposta saudável:**
```json
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "redis": { "status": "up" },
    "rabbitmq": { "status": "up" }
  },
  "error": {},
  "details": {
    "database": { "status": "up" },
    "redis": { "status": "up" },
    "rabbitmq": { "status": "up" }
  }
}
```

**Resposta degradada (HTTP 503):**
```json
{
  "status": "error",
  "error": {
    "redis": { "status": "down", "message": "ECONNREFUSED" }
  }
}
```

### 5.2 Métricas de SLA

| Métrica | SLO | Medição | Alerta |
|---|---|---|---|
| Disponibilidade da API | ≥ 99.5% (mensal) | Railway health check a cada 30s | Sentry + Slack se < 99% |
| Latência p95 | < 300ms | Railway Metrics | Sentry Warning se > 300ms; Error se > 1s |
| Taxa de erro (5xx) | < 0.5% | Sentry + Railway | Sentry Error se > 1%; Fatal se > 2% |
| Tempo de geração de relatório | < 30s | Log personalizado `report_generation_ms` | WARN no log se > 30s |
| Tempo de processamento de fila | < 5s por mensagem | Pino log + CloudAMQP | WARN se > 10s; escalação se DLQ > 0 |

---

## 6. Dashboard Operacional

### 6.1 Componentes do Dashboard

| Painel | Ferramenta | O que monitorar |
|---|---|---|
| Uptime e latência | Railway Metrics | Disponibilidade e tempo de resposta da API por endpoint |
| Erros em tempo real | Sentry | Volume de erros por nível, novos erros, tendência |
| Uso do produto | PostHog | Casos criados/dia, avanços por estágio, usuários ativos |
| Filas | CloudAMQP | Profundidade das filas CRM, DLQ, throughput |
| Cache | Upstash | Hit rate Redis, memória utilizada |
| Banco | Supabase Dashboard | Conexões ativas, slow queries (> 1s), storage do Dossiê |

### 6.2 Filas RabbitMQ do CRM

| Fila | Propósito | DLQ |
|---|---|---|
| `crm.notifications` | Notificações para usuários internos | `crm.notifications.dlq` |
| `crm.sla_alerts` | Alertas de SLA gerados pelo monitor | `crm.sla_alerts.dlq` |
| `crm.weekly_reports` | Geração do relatório semanal | `crm.weekly_reports.dlq` |
| `crm.integrations.retry` | Retentativas de integração (ZapSign, Celcoin, Meta) | `crm.integrations.dlq` |

Alerta automático: DLQ > 0 mensagens → Sentry Warning + notificação Slack #crm-alertas.

---

## 7. Retenção de Logs

| Ambiente | Retenção | Ferramenta |
|---|---|---|
| `development` | Sem retenção — logs apenas no terminal | N/A |
| `staging` | 7 dias | Railway Logs (retido automaticamente) |
| `production` | 30 dias | Railway Logs (retido automaticamente) |
| `audit_log` (banco) | 10 anos | Supabase PostgreSQL (tabela append-only, RN-194) |

> **Nota:** Os logs do Railway são logs de aplicação. O `audit_log` do banco (tabela separada, append-only, imutável) tem retenção de 10 anos conforme RN-194 — é independente dos logs do Railway.

---

## 8. Conformidade LGPD nos Logs

### 8.1 Regras de PII nos Logs de Aplicação

**Nunca logar nos logs de aplicação (Pino + Railway):**

| Dado | Categoria LGPD | Alternativa nos logs |
|---|---|---|
| CPF de Cedente/Cessionário | Dado pessoal sensível | UUID do contato (`contact_id`) |
| Nome completo | Dado pessoal | UUID do usuário (`user_id`) |
| E-mail | Dado pessoal | UUID (`user_id`) |
| Telefone | Dado pessoal | UUID (`contact_id`) |
| Endereço do imóvel | Dado pessoal | UUID do Caso (`case_id`) |
| Valor de proposta | Dado financeiro (contexto) | Apenas em logs DEBUG — desativado em produção |

### 8.2 Redaction Automático

O Pino já está configurado com `redact` para remover campos sensíveis dos logs de request/response (ver Seção 2.1). Para dados nos bodies de operações, os serviços NestJS devem explicitamente não incluir PII nas chamadas de log.

### 8.3 Sentry — Mascaramento

- `beforeSend`: e-mail e username removidos do evento antes do envio (ver Seção 3.1).
- Session replay: `maskAllText: true` — todo texto na tela é mascarado por padrão.
- Nenhum dado de Cedente/Cessionário deve aparecer nos breadcrumbs do Sentry.

---

## 9. Langfuse (Fase 2 — Dani-Admin IA)

> **Status: NÃO ATIVO na v1.0.** Reservado para Fase 2 com integração do agente Dani-Admin (LangChain.js + GPT-4o).

Quando ativo, o Langfuse rastreará:
- Cada chamada ao GPT-4o com prompt, resposta e tokens usados
- Decisões autônomas vs. encaminhamentos para aprovação humana
- Latência de resposta do agente por tipo de ação
- Custo de tokens por sessão de atendimento

Variável de ambiente reservada: `LANGFUSE_SECRET_KEY`, `LANGFUSE_PUBLIC_KEY` (não configurar na v1.0).

---

## 10. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — stack de observabilidade (Sentry + PostHog + Pino + Railway), logs estruturados com redaction de PII, alertas P0/P1, health check, métricas de SLA, dashboard operacional, retenção de logs, conformidade LGPD, Langfuse reservado para Fase 2. |
