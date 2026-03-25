# Notificações, Templates e Implementação — AI-Dani-Admin

## Especificação de Notificações do Módulo de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend |
| Escopo | Todos os canais de notificação do módulo AI-Dani-Admin: push mobile, Slack, e-mail e painel web. Templates completos e implementação técnica. |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D05 (PRD — RF-005, RF-006, RF-008, RF-009, RF-018), D11 (Mobile), D17 (Integrações Externas), D20 (Error Handling) |

---

> **📌 TL;DR**
>
> - **4 canais de notificação:** Push mobile (Expo), Slack, e-mail (SendGrid) e painel web (Supabase Realtime).
> - **6 tipos de alerta automático** com prioridade P0–P5. Templates completos por canal para cada tipo.
> - **Notificação de takeover:** mensagem ao usuário final (chat) quando Admin assume/encerra conversa.
> - **Notificação de indisponibilidade:** mensagem no chat e badge no FAB quando API do agente está indisponível.
> - **Arquitetura assíncrona:** Slack, e-mail e push via RabbitMQ. Painel web via Supabase Realtime.
> - **Idempotência garantida:** chave Redis por `alertEventId` para evitar duplicatas em retry.

---

## 1. Mapa de Notificações

| Evento | Push Mobile | Slack | E-mail | Painel Web |
|---|---|---|---|---|
| Desligamento automático (P0) | ✅ | ✅ | ✅ | ✅ |
| Latência alta (P1) | ✅ | ✅ | ❌ | ✅ |
| Taxa de erro elevada (P2) | ✅ | ✅ | ✅ | ✅ |
| CSAT degradado (P3) | ❌ | ❌ | ✅ | ✅ |
| Taxa de recusa alta (P4) | ❌ | ❌ | ❌ | ✅ |
| Consumo de processamento (P5) | ❌ | ❌ | ✅ | ✅ |
| Interação sinalizada para revisão | ✅ | ❌ | ❌ | ✅ |
| Takeover iniciado (ao usuário final) | ❌ | ❌ | ❌ | ✅ (chat) |
| Takeover encerrado (ao usuário final) | ❌ | ❌ | ❌ | ✅ (chat) |
| API do agente indisponível (ao usuário) | ❌ | ❌ | ❌ | ✅ (FAB badge + chat) |

---

## 2. Templates — Push Mobile (Expo)

### 2.1 Desligamento Automático (P0)

```typescript
{
  to: adminToken,
  sound: 'default',
  title: 'Agente desligado',
  body: `${agentName} desligado. Taxa de erro: ${errorRate}% (15min)`,
  data: {
    type: 'DESLIGAMENTO_AUTOMATICO',
    agentId,
    alertId,
    navigateTo: '/admin/alertas',
  },
  priority: 'high',
  badge: unreadAlertCount,
  ttl: 3600,  // 1h — se Admin não receber em 1h, descartar
}
```

### 2.2 Latência Alta (P1)

```typescript
{
  to: adminToken,
  sound: 'default',
  title: 'Latência alta',
  body: `Tempo de resposta acima do SLA há ${minutesAboveSla} min`,
  data: {
    type: 'LATENCIA_ALTA',
    agentId,
    alertId,
    navigateTo: '/admin/supervisao',
  },
  priority: 'high',
  ttl: 1800,  // 30min
}
```

### 2.3 Taxa de Erro Elevada (P2)

```typescript
{
  to: adminToken,
  sound: 'default',
  title: 'Taxa de erro elevada',
  body: `${errorRate}% das respostas com erro (15min)`,
  data: {
    type: 'TAXA_ERRO_ELEVADA',
    agentId,
    alertId,
    navigateTo: '/admin/supervisao',
  },
  priority: 'high',
  ttl: 1800,
}
```

### 2.4 Interação Sinalizada

```typescript
{
  to: adminToken,
  sound: 'default',
  title: 'Interação aguardando',
  body: `Confiança: ${confidenceScore}% — requer revisão`,
  data: {
    type: 'INTERACTION_FLAGGED',
    interactionId,
    navigateTo: `/admin/supervisao/${interactionId}`,
  },
  priority: 'normal',
  ttl: 3600,
}
```

---

## 3. Templates — Slack

### 3.1 Desligamento Automático (P0)

```json
{
  "text": "🔴 *Agente desligado automaticamente* — {{agentName}}",
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🔴 Agente desligado automaticamente"
      }
    },
    {
      "type": "section",
      "fields": [
        { "type": "mrkdwn", "text": "*Agente:*\n{{agentName}}" },
        { "type": "mrkdwn", "text": "*Taxa de erro:*\n{{errorRate}}% (últimos 15min)" },
        { "type": "mrkdwn", "text": "*Threshold:*\n30%" },
        { "type": "mrkdwn", "text": "*Amostra:*\n{{sampleSize}} interações" },
        { "type": "mrkdwn", "text": "*Horário:*\n{{triggeredAt}} (Fortaleza)" }
      ]
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "style": "danger",
          "text": { "type": "plain_text", "text": "Ver alertas" },
          "url": "https://admin.repasseseguro.com.br/supervisao-ia/alertas"
        }
      ]
    }
  ]
}
```

### 3.2 Latência Alta (P1)

```json
{
  "text": "🟡 *Latência alta* — {{agentName}}",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "🟡 *Latência alta detectada*\n*Agente:* {{agentName}}\n*Duração:* acima do SLA há {{minutesAboveSla}} minutos\n*Horário:* {{triggeredAt}} (Fortaleza)"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "Ver interações" },
          "url": "https://admin.repasseseguro.com.br/supervisao-ia"
        }
      ]
    }
  ]
}
```

### 3.3 Taxa de Erro Elevada (P2)

```json
{
  "text": "🟠 *Taxa de erro elevada* — {{agentName}}",
  "blocks": [
    {
      "type": "section",
      "fields": [
        { "type": "mrkdwn", "text": "*Agente:*\n{{agentName}}" },
        { "type": "mrkdwn", "text": "*Taxa de erro:*\n{{errorRate}}% (15min)" },
        { "type": "mrkdwn", "text": "*Threshold:*\n10%" },
        { "type": "mrkdwn", "text": "*Horário:*\n{{triggeredAt}} (Fortaleza)" }
      ]
    }
  ]
}
```

---

## 4. Templates — E-mail (SendGrid)

### 4.1 Desligamento Automático (P0)

```typescript
const emailTemplate = {
  to: adminEmails,
  from: { email: 'alertas@repasseseguro.com.br', name: 'Repasse Seguro — Alertas' },
  subject: `[P0] Agente desligado automaticamente — ${agentName}`,
  html: `
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    body { font-family: Inter, sans-serif; color: #0A0A0A; margin: 0; padding: 0; }
    .container { max-width: 600px; margin: 0 auto; padding: 24px; }
    .header { background-color: #E7000B; color: white; padding: 16px 24px; border-radius: 8px 8px 0 0; }
    .header h1 { margin: 0; font-size: 18px; }
    .body { background: #F5F5F5; padding: 24px; border-radius: 0 0 8px 8px; }
    .metric { background: white; padding: 16px; border-radius: 8px; margin-bottom: 12px; }
    .metric-label { color: #737373; font-size: 12px; margin-bottom: 4px; }
    .metric-value { font-size: 16px; font-weight: 600; }
    .cta { background: #0069A8; color: white; padding: 12px 24px; border-radius: 8px;
           text-decoration: none; display: inline-block; margin-top: 16px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>🔴 Agente desligado automaticamente</h1>
    </div>
    <div class="body">
      <div class="metric">
        <div class="metric-label">Agente</div>
        <div class="metric-value">${agentName}</div>
      </div>
      <div class="metric">
        <div class="metric-label">Taxa de erro (últimos 15min)</div>
        <div class="metric-value" style="color: #E7000B;">${errorRate}%</div>
      </div>
      <div class="metric">
        <div class="metric-label">Threshold configurado</div>
        <div class="metric-value">30%</div>
      </div>
      <div class="metric">
        <div class="metric-label">Amostra</div>
        <div class="metric-value">${sampleSize} interações</div>
      </div>
      <div class="metric">
        <div class="metric-label">Horário do evento</div>
        <div class="metric-value">${triggeredAt} (America/Fortaleza)</div>
      </div>
      <p style="color: #737373; font-size: 14px;">
        O agente foi desligado automaticamente e precisa de reativação manual.
        Acesse o painel Admin para investigar e reativar.
      </p>
      <a href="https://admin.repasseseguro.com.br/supervisao-ia/alertas" class="cta">
        Ver no painel Admin
      </a>
    </div>
  </div>
</body>
</html>
  `,
}
```

### 4.2 CSAT Degradado (P3)

```typescript
{
  subject: `[P3] CSAT abaixo da meta — ${agentName}`,
  html: `
    <h2>CSAT abaixo da meta</h2>
    <p><strong>Agente:</strong> ${agentName}</p>
    <p><strong>CSAT médio (24h):</strong> ${csatAverage}/5 (meta: 3,5/5)</p>
    <p><strong>Amostra:</strong> ${sampleSize} avaliações</p>
    <p><strong>Período:</strong> ${periodStart} a ${periodEnd}</p>
    <a href="https://admin.repasseseguro.com.br/supervisao-ia/metricas">Ver métricas</a>
  `,
}
```

### 4.3 Consumo de Processamento (P5)

```typescript
{
  subject: `[P5] Consumo de processamento acima de 80%`,
  html: `
    <h2>Consumo de processamento elevado</h2>
    <p><strong>Consumo atual:</strong> ${consumptionPercent}% do orçamento mensal</p>
    <p><strong>Orçamento mensal:</strong> R$ ${monthlyBudget}</p>
    <p><strong>Projeção para o mês:</strong> R$ ${projectedCost}</p>
    <p><strong>Horário:</strong> ${triggeredAt}</p>
    <a href="https://admin.repasseseguro.com.br/supervisao-ia/metricas">Ver dashboard de métricas</a>
  `,
}
```

---

## 5. Templates — Painel Web (Supabase Realtime)

### 5.1 COMP-001 — Banner de Alerta Crítico

```typescript
// Evento emitido via Supabase Realtime para subscribers do painel Admin
interface AlertBannerEvent {
  type: 'ALERT_BANNER'
  payload: {
    alertId: string
    alertType: 'DESLIGAMENTO_AUTOMATICO' | 'LATENCIA_ALTA' | 'TAXA_ERRO_ELEVADA'
    agentName: string
    message: string       // ex: "Dani-Cessionário desligado. Taxa de erro: 32.5% (15min)"
    priority: 'P0' | 'P1' | 'P2'
    triggeredAt: string   // ISO 8601
    actions: Array<{
      label: string       // "Ver interações" | "Reativar agente"
      navigateTo: string
    }>
  }
}
```

### 5.2 Interação Sinalizada — Atualização em Tempo Real

```typescript
interface InteractionFlaggedEvent {
  type: 'INTERACTION_FLAGGED'
  payload: {
    interactionId: string
    agentName: string
    confidenceScore: number
    status: 'SINALIZADA_PARA_REVISAO'
    createdAt: string
  }
}
```

---

## 6. Templates — Notificações de Takeover (Chat do Usuário)

### 6.1 Mensagem de Início de Takeover (RF-008)

Enviada ao usuário final quando Admin inicia takeover:

```typescript
interface TakeoverStartMessage {
  sender: {
    type: 'HUMAN_AGENT'
    name: 'Equipe Repasse Seguro'
    avatar: 'human-agent-icon'  // ícone diferenciado de pessoa
  }
  message: 'Um analista da equipe Repasse Seguro assumiu essa conversa para ajudá-lo. Como posso ajudar?'
  separator: {
    type: 'TAKEOVER_START'
    label: 'Atendimento humano'  // COMP-002 — D08 vocabulário aprovado
  }
  timestamp: string  // ISO 8601
}
```

### 6.2 Mensagem de Encerramento de Takeover (RF-009)

Enviada ao usuário final quando Admin encerra takeover:

```typescript
interface TakeoverEndMessage {
  sender: {
    type: 'AI_AGENT'
    name: agentName  // ex: 'Dani'
    avatar: 'dani-avatar'  // avatar padrão do agente
  }
  message: `Você está novamente em atendimento com ${agentName}.`
  separator: {
    type: 'TAKEOVER_END'
    label: 'Retorno ao atendimento automatizado'  // COMP-002 — D08 vocabulário aprovado
  }
  timestamp: string
}
```

---

## 7. Templates — Indisponibilidade do Agente (RF-018)

### 7.1 Mensagem no Chat

```typescript
interface AgentUnavailableMessage {
  sender: {
    type: 'SYSTEM'
    name: 'Repasse Seguro'
    avatar: 'system-icon'
  }
  message: 'O agente está temporariamente indisponível. Os cálculos de comissão e Escrow continuam disponíveis. Tente novamente em instantes.'
  type: 'SYSTEM_MESSAGE'
  timestamp: string
}
```

### 7.2 Badge no FAB Global

```typescript
interface FabBadgeState {
  agentStatus: 'ATIVO' | 'FALLBACK_ATIVO' | 'DESLIGADO'
  // agentStatus === 'FALLBACK_ATIVO':
  //   Badge: círculo amarelo (#CA8A04) sobre o ícone do FAB
  //   Tooltip/accessibility: "Agente temporariamente indisponível"
  // agentStatus === 'ATIVO':
  //   Badge: ausente (sem indicador)
  // agentStatus === 'DESLIGADO':
  //   Badge: círculo vermelho (#E7000B)
  //   Tooltip/accessibility: "Agente desligado"
}
```

---

## 8. Implementação do NotificationsService

```typescript
// src/alerts/notifications.service.ts

@Injectable()
export class NotificationsService {
  constructor(
    private readonly rabbitMQ: AmqpConnection,
    private readonly redis: Redis,
    private readonly prisma: PrismaService,
  ) {}

  async dispatchAlertNotifications(
    alertEvent: AlertEvent,
    channels: NotificationChannel[],
  ): Promise<void> {
    const adminTokens = await this.prisma.pushNotificationToken.findMany({
      where: { is_active: true },
    })

    const adminEmails = process.env.ADMIN_ALERT_EMAILS?.split(',') ?? []

    // Determinar canais com base na prioridade do alerta
    const alertConfig = ALERT_CHANNEL_MAP[alertEvent.alertType]

    if (alertConfig.slack) {
      await this.rabbitMQ.publish(
        'dani-admin.alerts',
        'slack',
        {
          alertEventId: alertEvent.id,
          alertType: alertEvent.alertType,
          alertData: alertEvent.alertData,
          priority: alertConfig.priority,
        },
      )
    }

    if (alertConfig.email) {
      await this.rabbitMQ.publish(
        'dani-admin.alerts',
        'email',
        {
          alertEventId: alertEvent.id,
          alertType: alertEvent.alertType,
          alertData: alertEvent.alertData,
          to: adminEmails,
        },
      )
    }

    if (alertConfig.push && adminTokens.length > 0) {
      await this.rabbitMQ.publish(
        'dani-admin.alerts',
        'push',
        {
          alertEventId: alertEvent.id,
          alertType: alertEvent.alertType,
          alertData: alertEvent.alertData,
          tokens: adminTokens.map(t => t.expoPushToken),
        },
      )
    }
  }
}

// Mapa de configuração de canais por tipo de alerta
const ALERT_CHANNEL_MAP: Record<AlertType, AlertChannelConfig> = {
  DESLIGAMENTO_AUTOMATICO: { priority: 'P0', slack: true, email: true, push: true, panel: true },
  LATENCIA_ALTA:           { priority: 'P1', slack: true, email: false, push: true, panel: true },
  TAXA_ERRO_ELEVADA:       { priority: 'P2', slack: true, email: true, push: true, panel: true },
  CSAT_DEGRADADO:          { priority: 'P3', slack: false, email: true, push: false, panel: true },
  TAXA_RECUSA_ALTA:        { priority: 'P4', slack: false, email: false, push: false, panel: true },
  CONSUMO_PROCESSAMENTO:   { priority: 'P5', slack: false, email: true, push: false, panel: true },
}
```

---

## 9. Supabase Realtime — Implementação de Subscriptions

```typescript
// Exemplo de uso no frontend (Painel Admin Web)
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

// Subscrição para novos alertas
const alertsChannel = supabase
  .channel('admin-alerts')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'alert_events',
      filter: `status=eq.ATIVO`,
    },
    (payload) => {
      // Exibir COMP-001 (Banner de Alerta Crítico)
      showAlertBanner(payload.new)
    },
  )
  .subscribe()

// Subscrição para interações sinalizadas
const interactionsChannel = supabase
  .channel('admin-interactions')
  .on(
    'postgres_changes',
    {
      event: 'UPDATE',
      schema: 'public',
      table: 'interactions',
      filter: `status=eq.SINALIZADA_PARA_REVISAO`,
    },
    (payload) => {
      // Atualizar badge de contagem na sidebar
      updateFlaggedCount(payload.new)
    },
  )
  .subscribe()

// Cleanup ao sair da página
return () => {
  supabase.removeChannel(alertsChannel)
  supabase.removeChannel(interactionsChannel)
}
```

---

## 10. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Templates completos para 4 canais (push, Slack, e-mail, painel web), 6 tipos de alerta, notificações de takeover e indisponibilidade. NotificationsService com ALERT_CHANNEL_MAP. |
