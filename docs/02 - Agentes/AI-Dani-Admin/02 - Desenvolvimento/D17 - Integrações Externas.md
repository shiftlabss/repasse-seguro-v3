# Integrações Externas — AI-Dani-Admin

## Guia de Integrações com Serviços Terceiros

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend e DevOps |
| Escopo | Especificação completa de todas as integrações externas do módulo AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D02 (Stacks), D05 (PRD), D14 (Especificações Técnicas), D16 (Documentação de API) |

---

> **📌 TL;DR**
>
> - **7 integrações externas:** OpenAI (LLM), Langfuse (observabilidade), Slack (alertas), SendGrid (e-mail), Sentry (error tracking), PostHog (analytics + feature flags), Expo Push (notificações mobile Admin).
> - **Padrão de falha:** toda integração tem comportamento explícito em caso de indisponibilidade. Nenhuma integração externa bloqueia o fluxo crítico de registro de interações.
> - **Secrets:** nunca hardcoded. Todas as credenciais via variáveis de ambiente (`.env` dev, Vault/Doppler prod).
> - **Retry:** toda chamada HTTP externa tem retry exponencial (máx 3 tentativas) + timeout definido.
> - **Alertas críticos (Slack, SendGrid):** via RabbitMQ — assíncronos, com DLQ.

---

## 1. OpenAI API

### 1.1 Propósito

Geração de respostas LLM pelos agentes Dani-Cessionário e Dani-Cedente. O AI-Dani-Admin não chama OpenAI diretamente — os agentes chamam. O módulo Admin recebe o resultado já processado (resposta + confidence_score + latency_ms) via `POST /api/v1/internal/interactions`.

### 1.2 Dependência

| Item | Valor |
|---|---|
| SDK | `openai` npm (versão 4.x+) |
| Modelo principal | GPT-4o (padrão) |
| Modelo de embedding | `text-embedding-3-small` |
| Endpoint | `https://api.openai.com/v1` |
| Auth | `OPENAI_API_KEY` no header `Authorization: Bearer` |

### 1.3 Comportamento em caso de indisponibilidade (FallbackAtivo)

Quando a OpenAI API estiver indisponível:
1. Agente passa para status `FALLBACK_ATIVO`.
2. Calculadora de Comissão (módulo determinístico) assume os cálculos.
3. FAB global exibe badge amarelo ao usuário.
4. Alerta `DESLIGAMENTO_AUTOMATICO` é disparado se erro persistir por 15 minutos.
5. AI-Dani-Admin registra a interação com `status: SINALIZADA_PARA_REVISAO`.

### 1.4 Variáveis de ambiente

```bash
OPENAI_API_KEY=sk-...
OPENAI_ORG_ID=org-... # opcional
```

---

## 2. Langfuse

### 2.1 Propósito

Observabilidade de LLM: tracing completo de cada interação dos agentes (custo por request, latência do modelo, tokens consumidos, confidence score, qualidade das respostas). Alimenta parte dos dados do dashboard de métricas (D14 ADR-003).

### 2.2 Dependência

| Item | Valor |
|---|---|
| SDK | `langfuse` npm (versão 3.x+) |
| Endpoint | `https://cloud.langfuse.com` (SaaS) ou instância self-hosted |
| Auth | `LANGFUSE_PUBLIC_KEY` + `LANGFUSE_SECRET_KEY` |
| Modo | Fire-and-forget (async) — não bloqueia o fluxo principal |

### 2.3 O que é registrado

Cada interação envia um trace para o Langfuse com:

```typescript
langfuse.trace({
  id: interaction.id,              // UUID da interação
  name: `dani-${agentName}-interaction`,
  userId: interaction.userId,      // anonimizado
  metadata: {
    agentId: interaction.agentId,
    confidenceScore: interaction.confidenceScore,
    latencyMs: interaction.latencyMs,
    tokensUsed: dataUsed.contextWindowTokens,
    toolsUsed: dataUsed.toolsUsed,
  },
  input: { userMessage },
  output: { agentResponse },
})
```

### 2.4 Dados consumidos do Langfuse pelo Dashboard

O `MetricsService` consulta o Langfuse SDK para:
- Custo estimado de tokens por agente por período.
- Distribuição de latência do modelo (p50, p95).
- Taxa de erros de LLM (timeouts, rate limits).

### 2.5 Comportamento em caso de indisponibilidade

Langfuse indisponível → fire-and-forget falha silenciosamente → log de warning no Pino → Sentry captura a exceção com prioridade P3 (não crítico). O fluxo de registro de interação continua normalmente. Métricas do dashboard que dependem do Langfuse exibem `"status": "INSUFFICIENT_DATA"`.

### 2.6 Variáveis de ambiente

```bash
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com
```

---

## 3. Slack

### 3.1 Propósito

Canal de alertas críticos para a equipe Admin (RF-005). Alertas de prioridade P0, P1 e P2 são enviados ao Slack.

### 3.2 Dependência

| Item | Valor |
|---|---|
| Método | Incoming Webhook (não Slack Bot API) |
| Endpoint | URL do webhook configurada no canal `#alertas-dani-admin` |
| Auth | URL do webhook contém token embutido |
| Protocolo | HTTPS POST |
| Payload | JSON com `text`, `blocks` (Block Kit opcional) |

**Justificativa do Incoming Webhook (D14 ADR-001):** Notificações unidirecionais sem necessidade de ler mensagens do canal. Menor complexidade vs. Slack Bot OAuth.

### 3.3 Payload de alerta

```json
POST <SLACK_WEBHOOK_URL>
Content-Type: application/json

{
  "text": "🔴 *Agente desligado automaticamente* — Dani-Cessionário",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Taxa de erro:* 32.5% (últimos 15min)\n*Threshold:* 30%\n*Amostra:* 48 interações\n*Horário:* 23/03/2026 15:10 (Fortaleza)"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "Ver no painel" },
          "url": "https://admin.repasseseguro.com.br/supervisao-ia/alertas"
        }
      ]
    }
  ]
}
```

### 3.4 Mapeamento de alertas → Slack

| Tipo de alerta | Prioridade | Mensagem Slack |
|---|---|---|
| `DESLIGAMENTO_AUTOMATICO` | P0 | "Agente desligado. Taxa de erro: [X]% (15min)" |
| `LATENCIA_ALTA` | P1 | "Latência alta há [X] minutos. SLA excedido." |
| `TAXA_ERRO_ELEVADA` | P2 | "[X]% das respostas com erro (15min)" |

Alertas P3, P4, P5 não são enviados ao Slack.

### 3.5 Retry e DLQ

Publicado na fila `dani-admin.alerts.slack` via RabbitMQ. Consumer executa retry exponencial:
- Tentativa 1: imediata
- Tentativa 2: 1s depois
- Tentativa 3: 2s depois
- Após 3 falhas: mensagem vai para `dani-admin.alerts.slack.dlq` + Sentry captura P1

### 3.6 Idempotência

Chave Redis `dani-admin:sent:slack:{alertEventId}` com TTL 24h. Consumer verifica antes de enviar — previne duplicatas em retry.

### 3.7 Variáveis de ambiente

```bash
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/T.../B.../...
```

---

## 4. SendGrid

### 4.1 Propósito

Envio de e-mails de alerta para administradores (RF-005). Alertas de prioridade P0, P2 e P5.

### 4.2 Dependência

| Item | Valor |
|---|---|
| SDK | `@sendgrid/mail` npm (versão 8.x+) |
| Endpoint | `https://api.sendgrid.com/v3/mail/send` |
| Auth | `SENDGRID_API_KEY` no header `Authorization: Bearer` |
| From | `alertas@repasseseguro.com.br` |
| To | Lista de e-mails Admin (enviada no payload do job RabbitMQ) |

### 4.3 Payload de e-mail

```typescript
await sgMail.send({
  to: ['admin1@repasseseguro.com.br', 'admin2@repasseseguro.com.br'],
  from: { email: 'alertas@repasseseguro.com.br', name: 'Repasse Seguro — Alertas' },
  subject: `[P0] Agente desligado automaticamente — Dani-Cessionário`,
  html: `
    <h2>Agente desligado automaticamente</h2>
    <p><strong>Agente:</strong> Dani-Cessionário</p>
    <p><strong>Taxa de erro:</strong> 32.5% (últimos 15 minutos)</p>
    <p><strong>Threshold configurado:</strong> 30%</p>
    <p><strong>Horário:</strong> 23/03/2026 15:10 (America/Fortaleza)</p>
    <a href="https://admin.repasseseguro.com.br/supervisao-ia/alertas">Ver no painel Admin</a>
  `,
})
```

### 4.4 Mapeamento de alertas → E-mail

| Tipo de alerta | Prioridade | Subject |
|---|---|---|
| `DESLIGAMENTO_AUTOMATICO` | P0 | `[P0] Agente desligado automaticamente — {agentName}` |
| `TAXA_ERRO_ELEVADA` | P2 | `[P2] Taxa de erro elevada — {agentName}` |
| `CSAT_DEGRADADO` | P3 | `[P3] CSAT abaixo da meta — {agentName}` |
| `CONSUMO_PROCESSAMENTO` | P5 | `[P5] Consumo de processamento acima de 80%` |

### 4.5 Retry e DLQ

Idêntico ao Slack: fila `dani-admin.alerts.email`, retry 3x exponencial, DLQ `dani-admin.alerts.email.dlq`.

### 4.6 Variáveis de ambiente

```bash
SENDGRID_API_KEY=SG....
SENDGRID_FROM_EMAIL=alertas@repasseseguro.com.br
ADMIN_ALERT_EMAILS=admin1@repasseseguro.com.br,admin2@repasseseguro.com.br
```

---

## 5. Sentry

### 5.1 Propósito

Error tracking de exceções de infraestrutura (não erros de LLM — esses ficam no Langfuse). Captura: exceções não tratadas, timeouts, falhas de conexão com Redis/RabbitMQ/DB, alertas críticos não entregues.

### 5.2 Dependência

| Item | Valor |
|---|---|
| SDK | `@sentry/nestjs` npm (versão 8.x+) |
| Endpoint | `https://sentry.io` |
| Auth | `SENTRY_DSN` |
| Inicialização | `Sentry.init()` antes do bootstrap NestJS |

### 5.3 Configuração

```typescript
// main.ts — antes do app.listen()
import * as Sentry from '@sentry/nestjs'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Express({ app }),
  ],
  beforeSend(event) {
    // Remover PII de eventos antes de enviar
    if (event.user) {
      delete event.user.email
      delete event.user.username
    }
    return event
  },
})
```

### 5.4 Eventos capturados

| Evento | Prioridade Sentry | Contexto |
|---|---|---|
| DB connection timeout | P1 | `{module: 'db', timeout_ms: 5000}` |
| Redis indisponível | P2 | `{module: 'redis', operation: 'GET'}` |
| RabbitMQ consumer falhou após 3 retries | P1 | `{queue: 'dani-admin.alerts.slack', alert_event_id}` |
| Langfuse trace falhou | P3 | `{module: 'langfuse', interaction_id}` |
| Alerta crítico não entregue (DLQ) | P0 | `{queue, message_count}` |

### 5.5 Comportamento em caso de indisponibilidade

Sentry indisponível → SDK falha silenciosamente → exceção continua sendo tratada normalmente pelo `HttpExceptionFilter`. Nenhum impacto no fluxo do produto.

### 5.6 Variáveis de ambiente

```bash
SENTRY_DSN=https://...@o....ingest.sentry.io/...
```

---

## 6. PostHog

### 6.1 Propósito

Analytics de ações do Admin no painel + feature flags como kill switch para features de IA (RF-017 — disponibilidade do webchat). PostHog captura eventos de uso sem PII.

### 6.2 Dependência

| Item | Valor |
|---|---|
| SDK | `posthog-node` npm (versão 4.x+) |
| Endpoint | `https://app.posthog.com` |
| Auth | `POSTHOG_API_KEY` |
| Modo | Async (fire-and-forget para eventos) |

### 6.3 Eventos registrados

| Evento | Trigger | Propriedades |
|---|---|---|
| `admin_takeover_initiated` | POST /admin/takeover | `{agent_id, interaction_id, has_reason}` |
| `admin_takeover_ended` | DELETE /admin/takeover/:id | `{takeover_duration_ms}` |
| `threshold_updated` | PATCH /admin/agent-config | `{agent_id, previous_value, new_value}` |
| `agent_reactivated` | POST /admin/agents/:id/reactivate | `{agent_id, downtime_ms}` |
| `alert_acknowledged` | PATCH /admin/alerts/:id/acknowledge | `{alert_type, time_to_acknowledge_ms}` |
| `launch_authorized` | POST /admin/launch-readiness/:id/authorize | `{agent_id, adversarial_tests_count}` |
| `agent_auto_disabled` | AlertsScheduler detecta erro_rate > 30% | `{agent_id, error_rate, window_minutes: 15}` |
| `cache_miss_high` | MetricsService: hit rate < 60% | `{agent_id, period, hit_rate}` |
| `alert_delivery_failed` | Consumer após 3 retries | `{queue, alert_type}` |

### 6.4 Feature Flags — Kill Switch

```typescript
// Verificar se o webchat está habilitado (RF-017)
const isWebchatEnabled = await posthog.isFeatureEnabled('webchat-enabled', 'global')

if (!isWebchatEnabled) {
  // FAB exibe badge de indisponibilidade
  // Chat retorna mensagem RF-018
}
```

Feature flags relevantes:
- `webchat-enabled` — habilita/desabilita o webchat globalmente (kill switch de emergência)
- `dani-cessionario-enabled` — habilita/desabilita o agente Cessionário
- `dani-cedente-enabled` — habilita/desabilita o agente Cedente
- `launch-readiness-enabled` — habilita/desabilita o checklist de prontidão

### 6.5 Comportamento em caso de indisponibilidade

PostHog indisponível → eventos não são capturados (fire-and-forget) → feature flags retornam valor padrão configurado localmente → log de warning no Pino. Nenhum impacto no fluxo do produto.

**Valor padrão de feature flags em caso de falha:** `true` (habilitado) — para evitar que falha do PostHog desabilite features em produção.

### 6.6 PII e privacidade

- `adminId` é UUID — nunca e-mail ou nome.
- `interactionId` e `userId` são UUIDs — dados anonimizados.
- PostHog configurado com `person_profiles: 'never'` — sem criação de perfis individuais de Admin.

### 6.7 Variáveis de ambiente

```bash
POSTHOG_API_KEY=phc_...
POSTHOG_HOST=https://app.posthog.com
```

---

## 7. Expo Push Notifications

### 7.1 Propósito

Envio de push notifications para o app mobile do Admin (D11). Alertas críticos (P0, P1) + notificações de nova interação sinalizada.

### 7.2 Dependência

| Item | Valor |
|---|---|
| SDK | `expo-server-sdk` npm (versão 3.x+) |
| Endpoint | `https://exp.host/--/api/v2/push/send` |
| Auth | `EXPO_ACCESS_TOKEN` (Expo Application Services) |
| Tokens | Armazenados em `push_notification_tokens` (D13) |

### 7.3 Envio de notificação

```typescript
import Expo from 'expo-server-sdk'

const expo = new Expo({ accessToken: process.env.EXPO_ACCESS_TOKEN })

const messages = adminTokens
  .filter(token => Expo.isExpoPushToken(token.expoPushToken))
  .map(token => ({
    to: token.expoPushToken,
    sound: 'default',
    title: 'Agente desligado',
    body: `Dani-Cessionário desligado. Taxa de erro: 32.5% (15min)`,
    data: {
      type: 'DESLIGAMENTO_AUTOMATICO',
      agentId: 'agent-uuid',
      alertId: 'alert-uuid',
      navigateTo: '/admin/alertas',
    },
    priority: 'high',
    badge: unreadAlertCount,
  }))

const chunks = expo.chunkPushNotifications(messages)
for (const chunk of chunks) {
  const ticketChunk = await expo.sendPushNotificationsAsync(chunk)
  // Processar tickets — verificar erros de token inválido
  for (const ticket of ticketChunk) {
    if (ticket.status === 'error' && ticket.details?.error === 'DeviceNotRegistered') {
      // Token inválido → marcar como is_active: false no banco
      await pushTokensService.deactivateToken(ticket.expoPushToken)
    }
  }
}
```

### 7.4 Mapeamento de alertas → Push

| Tipo | Título | Corpo | Ação ao tap |
|---|---|---|---|
| `DESLIGAMENTO_AUTOMATICO` | "Agente desligado" | "Dani-[X] desligado. Taxa de erro: [X]% (15min)" | `/admin/alertas` |
| `LATENCIA_ALTA` | "Latência alta" | "Tempo de resposta acima do SLA há [X] min" | `/admin/supervisao` |
| `TAXA_ERRO_ELEVADA` | "Taxa de erro elevada" | "[X]% das respostas com erro (15min)" | `/admin/supervisao` |
| `CSAT_DEGRADADO` | "CSAT abaixo da meta" | "Média: [X]/5 nas últimas 24h" | `/admin/supervisao` |
| Nova interação sinalizada | "Interação aguardando" | "Confiança: [X]% — requer revisão" | `/admin/supervisao/{id}` |

### 7.5 Tratamento de tokens inválidos

Quando Expo retorna `DeviceNotRegistered`:
1. `PushTokensService.deactivateToken(expoPushToken)` — atualiza `is_active: false` no banco.
2. Token continua na tabela (para histórico), mas não recebe mais notificações.
3. Próximo login do Admin no app mobile registra novo token via `POST /api/v1/admin/push-tokens`.

### 7.6 Retry e DLQ

Publicado na fila `dani-admin.alerts.push` via RabbitMQ. Consumer executa retry 3x. Após falha: DLQ + log Sentry.

### 7.7 Variáveis de ambiente

```bash
EXPO_ACCESS_TOKEN=...
```

---

## 8. Tabela de Todas as Integrações

| Serviço | Protocolo | Modo | Crítico para fluxo principal | Retry | Fallback |
|---|---|---|---|---|---|
| OpenAI | HTTPS REST | Síncrono (via agentes) | Não (Admin não chama diretamente) | Sim (agentes) | FallbackAtivo — Calculadora de Comissão |
| Langfuse | HTTPS REST | Fire-and-forget (async) | Não | Não | Falha silenciosa — métricas exibem INSUFFICIENT_DATA |
| Slack | HTTPS Webhook | Assíncrono (RabbitMQ) | Não | 3x exponencial + DLQ | DLQ + Sentry P1 |
| SendGrid | HTTPS REST | Assíncrono (RabbitMQ) | Não | 3x exponencial + DLQ | DLQ + Sentry P1 |
| Sentry | HTTPS SDK | Fire-and-forget | Não | SDK nativo | Falha silenciosa |
| PostHog | HTTPS SDK | Fire-and-forget | Não (exceto feature flags) | Não | Feature flags retornam `true` (habilitado) |
| Expo Push | HTTPS REST | Assíncrono (RabbitMQ) | Não | 3x exponencial + DLQ | DLQ — Admin recebe alerta no painel web |

---

## 9. Checklist de Configuração por Ambiente

| Variável | Dev (local) | Staging | Produção |
|---|---|---|---|
| `OPENAI_API_KEY` | Conta de dev | Conta de dev | Conta de produção |
| `LANGFUSE_PUBLIC_KEY` / `SECRET_KEY` | Projeto dev | Projeto staging | Projeto produção |
| `SLACK_WEBHOOK_URL` | Canal `#alertas-dev` | Canal `#alertas-staging` | Canal `#alertas-dani-admin` |
| `SENDGRID_API_KEY` | Sandbox mode | Sandbox mode | Conta de produção |
| `ADMIN_ALERT_EMAILS` | e-mail dev | e-mails do time | e-mails operacionais |
| `SENTRY_DSN` | Projeto dev | Projeto staging | Projeto produção |
| `POSTHOG_API_KEY` | Projeto dev | Projeto staging | Projeto produção |
| `EXPO_ACCESS_TOKEN` | Token dev | Token staging | Token produção |

---

## 10. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. 7 integrações externas documentadas com SDKs, payloads, retry, DLQ, fallbacks e variáveis de ambiente. |
