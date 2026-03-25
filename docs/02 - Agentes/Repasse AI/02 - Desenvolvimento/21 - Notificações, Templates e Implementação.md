# 21 - Notificações, Templates e Implementação

## Cabeçalho

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
| --- | --- | --- | --- | --- |
| 21 - Notificações, Templates e Implementação | v1.0 | 2026-03-22 | Claude Code Desktop (ShiftLabs Pipeline v9.5) | Aprovado |

---

## TL;DR

> 📌 **Sistema de notificações do Repasse AI:**
>
> - **4 canais:** Web Push (PWA), WhatsApp (EvolutionAPI), In-App (SSE Webchat), Email (futuro — P2).
> - **12 templates** mapeados, cobrindo conversas, takeover, OTP, análise de oportunidade e inatividade.
> - **Envio assíncrono obrigatório** via RabbitMQ — exchange `notifications`, 3 queues por canal com DLQ.
> - **Notificações críticas** (OTP WhatsApp, alerta de segurança) não podem ser desativadas por opt-out.
> - **Deep link contextual** em todo push — nenhum push genérico sem destino definido.
> - **6 eventos de tracking:** sent, delivered, opened, clicked, failed, opted_out.
> - **LGPD:** retenção de logs de tracking por 90 dias; opt-out persistido na tabela `NotificationPreference`.

---

## 1. Arquitetura de Notificações

O sistema de notificações do Repasse AI é processado inteiramente de forma assíncrona. Nenhuma notificação é enviada de forma síncrona na request principal. O `NotificationService` publica eventos no RabbitMQ; workers por canal processam e entregam independentemente.

```mermaid
flowchart TD
    A[Evento de Domínio\ne.g. AgentResponse, TakeoverRequest] --> B[NotificationService\npublica para exchange]
    B --> C[RabbitMQ Exchange: notifications\ndirect + topic]

    C --> D[Queue: notif.push\nWeb Push - Upstash]
    C --> E[Queue: notif.whatsapp\nEvolutionAPI]
    C --> F[Queue: notif.inapp\nSSE Webchat]

    D --> G{Preferência ativa?\n+ Opt-out check}
    E --> G
    F --> G

    G -->|Sim| H[TemplateService\ninterpolação de variáveis]
    G -->|Não - não crítico| I[Ignorar + log opted_out]
    G -->|Não - crítico| H

    H --> J[DeliveryService\ncanal específico]
    J --> K{Entregue?}
    K -->|Sim| L[Tracking: sent + delivered]
    K -->|Não - retryable| M[Retry: backoff 2x]
    M --> K
    K -->|Falha final| N[DLQ: notif.{canal}.dlq\n+ tracking: failed]
    N --> O[Alerta Sentry/Slack]

    L --> P[NotificationLog\nPersistência 90 dias]
```

---

## 2. Canais

### 2.1 Web Push (PWA)

O canal principal para usuários que acessam o Webchat via browser (iOS 16.4+ com PWA instalado; Android Chrome sem instalação necessária). Entregue via Upstash Web Push ou VAPID direto.

| Atributo | Valor |
| --- | --- |
| Biblioteca | Web Push API (VAPID) |
| Trigger | Worker `notif.push` |
| Rate limit | 10 pushes/usuário/hora |
| Retry | 2x, backoff 30s → 60s |
| Fallback | In-app na próxima sessão |
| Opt-out | Configurável (exceto críticos) |

**Payload padrão:**
```json
{
  "title": "Repasse Seguro",
  "body": "Nova oportunidade analisada: Imóvel Ref #4521",
  "icon": "/icons/icon-192x192.png",
  "badge": "/icons/badge-72x72.png",
  "data": {
    "deep_link": "/chat/conversations/conv_abc123",
    "notification_id": "notif_01HX4M3P9Q",
    "type": "agent_response_ready",
    "correlation_id": "req_01HX4M3P9QKJZ8B2V"
  },
  "actions": [
    { "action": "open", "title": "Ver análise" },
    { "action": "dismiss", "title": "Dispensar" }
  ]
}
```

### 2.2 WhatsApp (EvolutionAPI)

Canal P1. Usado para notificações proativas ao lead vinculado e para o fluxo OTP de binding. Mensagens enviadas via EvolutionAPI usando o número vinculado ao tenant.

| Atributo | Valor |
| --- | --- |
| Biblioteca | EvolutionAPI REST (HTTP client interno) |
| Trigger | Worker `notif.whatsapp` |
| Rate limit | 20 mensagens/dia/número (WhatsApp Business Policy) |
| Retry | 2x, backoff 2s → 4s + DLQ |
| Fallback | DLQ → alerta supervisor |
| Opt-out | Configurável (exceto OTP e alertas críticos) |

**Payload padrão:**
```json
{
  "number": "5585999991234",
  "text": "Olá! Sua análise de repasse está pronta. Acesse: https://app.repasseseguro.com.br/chat/conv_abc123",
  "options": {
    "delay": 1000,
    "presence": "composing"
  }
}
```

### 2.3 In-App (SSE Webchat)

Canal de menor latência — entregue diretamente no Webchat via SSE se o usuário estiver conectado. Implementado no endpoint `GET /repasse-ai/v1/supervision/takeover-stream` e em eventos de chat.

| Atributo | Valor |
| --- | --- |
| Protocolo | Server-Sent Events (SSE) |
| Trigger | Worker `notif.inapp` |
| Rate limit | Sem limite explícito (bounded pela sessão SSE) |
| Retry | N/A — se SSE fechada, fallback para push |
| Fallback | Web Push se SSE não ativa |
| Opt-out | Não aplicável — in-app é canal de UI |

**Payload SSE:**
```
data: {"type":"notification","notification_id":"notif_01HX4M3","notif_type":"takeover_requested","title":"Supervisor solicitou assumir conversa","data":{"conversation_id":"conv_abc123","supervisor_id":"usr_xyz"}}\n\n
```

### 2.4 Email (P2 — Futuro)

Canal previsto para notificações de relatório semanal e alertas de billing/compliance. Não implementado no MVP. [DECISÃO AUTÔNOMA] Excluído do P0/P1 por PG-03 (backend puro) e ausência de configuração de SMTP/SES nas integrações externas (Doc 17). Alternativa (incluir no P1 com Resend.com) descartada pela baixa urgência no caso de uso de conversa em tempo real. Será implementado quando houver relatórios periódicos ou demanda de billing.

> ⚙️ **Padrão obrigatório:** quando email for implementado, seguir o mesmo contrato de fila/template/tracking desta seção.

---

## 3. Templates

Inventário completo de templates do Repasse AI. Todos os templates são armazenados no banco via seed `AgentConfiguration` e interpolados em runtime pelo `TemplateService`. Nenhum template hardcoded no código.

> 🔴 **Regra inegociável:** templates são dados gerenciados, não código. Qualquer alteração de texto passa pelo banco, não por deploy.

| ID | Nome | Evento Gatilho | Canais | Variáveis | Prioridade | Opt-out? |
| --- | --- | --- | --- | --- | --- | --- |
| NT-001 | agent_response_ready | AgentService.processMessage concluído | Push, In-App | `{{conversation_id}}`, `{{opportunity_ref}}`, `{{deep_link}}` | high | Sim |
| NT-002 | whatsapp_otp | WhatsApp binding iniciado | WhatsApp | `{{otp_code}}`, `{{expires_min}}` | critical | Não |
| NT-003 | takeover_requested | SupervisionService.requestTakeover | Push, In-App | `{{conversation_id}}`, `{{supervisor_name}}`, `{{deep_link}}` | high | Não |
| NT-004 | takeover_granted | SupervisionService.grantTakeover | In-App | `{{conversation_id}}`, `{{cessionario_name}}` | high | Não |
| NT-005 | takeover_released | SupervisionService.releaseTakeover | In-App, Push | `{{conversation_id}}`, `{{deep_link}}` | normal | Sim |
| NT-006 | inactivity_warning | Conversation sem resposta há 24h | Push, WhatsApp | `{{conversation_id}}`, `{{lead_name}}`, `{{deep_link}}` | normal | Sim |
| NT-007 | opportunity_analysis_complete | CalculatorService.analyzePortfolio concluído | Push, In-App | `{{portfolio_id}}`, `{{total_opportunities}}`, `{{deep_link}}` | high | Sim |
| NT-008 | whatsapp_binding_success | WhatsApp OTP validado com sucesso | WhatsApp, In-App | `{{phone_number_masked}}` | high | Não |
| NT-009 | whatsapp_binding_failed | OTP expirado ou máximo de tentativas | In-App | `{{reason}}` | high | Não |
| NT-010 | agent_circuit_breaker_open | CircuitBreaker abriu (5 falhas) | In-App (Admin) | `{{opened_at}}`, `{{failure_count}}` | critical | Não |
| NT-011 | conversation_assigned | Nova conversa atribuída ao agente IA | In-App | `{{conversation_id}}`, `{{lead_name}}` | normal | Sim |
| NT-012 | message_received_whatsapp | Mensagem recebida via WhatsApp | In-App | `{{conversation_id}}`, `{{lead_name}}`, `{{preview}}` | high | Não |

### 3.1 Exemplo de Template Renderizado (NT-001)

**Push:**
```
Título: "Análise pronta"
Corpo: "Sua análise do imóvel Ref #4521 está disponível."
Deep link: /chat/conversations/conv_abc123
```

**In-App (SSE):**
```json
{
  "type": "notification",
  "notif_type": "agent_response_ready",
  "title": "Análise pronta",
  "body": "Sua análise do imóvel Ref #4521 está disponível.",
  "data": { "conversation_id": "conv_abc123", "deep_link": "/chat/conversations/conv_abc123" }
}
```

---

## 4. Preferências do Usuário (Opt-out)

### 4.1 Modelo de Preferências

Persistido na tabela `NotificationPreference` (Doc 12 — Modelo de Dados). Um registro por `(cessionario_id, notification_type, channel)`.

```typescript
// Modelo de preferência
interface NotificationPreference {
  cessionario_id: string;
  notification_type: string;  // e.g. "agent_response_ready"
  channel: 'push' | 'whatsapp' | 'inapp' | 'email';
  enabled: boolean;
  updated_at: Date;
}
```

### 4.2 Regras de Opt-out

> 🔴 **Regra inegociável:** notificações marcadas como `critical` na tabela de templates (NT-002, NT-003, NT-004, NT-008, NT-009, NT-010) não podem ser desativadas pelo usuário. O `NotificationService` ignora a preferência para esses tipos.

| Tipo | Crítico? | Pode desativar? | Canais forçados |
| --- | --- | --- | --- |
| `whatsapp_otp` | Sim | Não | WhatsApp |
| `takeover_requested` | Sim | Não | In-App (Push também se SSE inativa) |
| `takeover_granted` | Sim | Não | In-App |
| `agent_circuit_breaker_open` | Sim | Não | In-App (Admin) |
| `whatsapp_binding_success` | Sim | Não | WhatsApp + In-App |
| `whatsapp_binding_failed` | Sim | Não | In-App |
| `agent_response_ready` | Não | Sim | Push, In-App |
| `inactivity_warning` | Não | Sim | Push, WhatsApp |
| `opportunity_analysis_complete` | Não | Sim | Push, In-App |
| `takeover_released` | Não | Sim | In-App, Push |
| `conversation_assigned` | Não | Sim | In-App |
| `message_received_whatsapp` | Não | Não | In-App |

### 4.3 API de Preferências

```
GET  /repasse-ai/v1/notification/preferences
PUT  /repasse-ai/v1/notification/preferences
```

**Exemplo de body PUT:**
```json
{
  "preferences": [
    { "notification_type": "agent_response_ready", "channel": "push", "enabled": false },
    { "notification_type": "inactivity_warning", "channel": "whatsapp", "enabled": false }
  ]
}
```

O backend rejeita silenciosamente qualquer tentativa de desativar notificações críticas — retorna 200 mas não persiste a alteração, logando o evento como `CRITICAL_OPTOUT_ATTEMPT`.

---

## 5. Fila e Processamento

### 5.1 Configuração RabbitMQ

Exchange principal: `notifications` (type: `direct`). Routing keys por canal.

| Queue | Routing Key | DLQ | Max Retries | Backoff | TTL Mensagem |
| --- | --- | --- | --- | --- | --- |
| `notif.push` | `push` | `notif.push.dlq` | 3 | 30s → 60s → 120s | 5 min |
| `notif.whatsapp` | `whatsapp` | `notif.whatsapp.dlq` | 2 | 2s → 4s | 24h (WhatsApp policy) |
| `notif.inapp` | `inapp` | `notif.inapp.dlq` | 1 | 5s | 30 min |

### 5.2 Payload de Mensagem na Fila

```json
{
  "notification_id": "notif_01HX4M3P9QKJZ8B2VCWF5NTD7",
  "template_id": "NT-001",
  "notification_type": "agent_response_ready",
  "channel": "push",
  "cessionario_id": "ces_abc123",
  "recipient": {
    "user_id": "usr_xyz",
    "push_subscription": { "endpoint": "https://fcm...", "keys": { "auth": "...", "p256dh": "..." } }
  },
  "variables": {
    "conversation_id": "conv_abc123",
    "opportunity_ref": "4521",
    "deep_link": "/chat/conversations/conv_abc123"
  },
  "priority": "high",
  "correlation_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD9",
  "enqueued_at": "2026-03-22T14:30:00.000Z"
}
```

### 5.3 Publicação na Fila (NotificationService)

```typescript
// apps/ai/src/notification/services/notification.service.ts
async sendNotification(dto: SendNotificationDto): Promise<void> {
  const template = await this.templateService.resolve(dto.templateId, dto.variables);
  const preferences = await this.preferenceService.get(dto.cessionarioId, dto.notificationType);

  for (const channel of template.channels) {
    const isCritical = template.priority === 'critical';
    const channelEnabled = isCritical || preferences[channel] !== false;

    if (!channelEnabled) {
      await this.trackingService.track(dto.notificationId, channel, 'opted_out');
      continue;
    }

    await this.rabbitMQService.publish('notifications', channel, {
      notification_id: dto.notificationId,
      template_id: dto.templateId,
      notification_type: dto.notificationType,
      channel,
      cessionario_id: dto.cessionarioId,
      recipient: dto.recipient,
      variables: dto.variables,
      priority: template.priority,
      correlation_id: dto.correlationId,
      enqueued_at: new Date().toISOString(),
    });

    await this.trackingService.track(dto.notificationId, channel, 'sent');
  }
}
```

### 5.4 Anti-Exemplos de Fila

❌ **Anti-exemplo 1 — Envio síncrono na request principal:**
```typescript
// PROIBIDO — bloqueia a request e aumenta latência
async processMessage(dto: CreateMessageDto) {
  const response = await this.llmService.invoke(dto);
  await this.pushService.send(response.userId, 'Análise pronta'); // síncrono!
  return response;
}
```
✅ **Correto:** publicar evento no RabbitMQ e retornar — o worker entrega assincronamente.

---

❌ **Anti-exemplo 2 — Template hardcoded:**
```typescript
// PROIBIDO — texto no código, deploy necessário para qualquer alteração
const message = `Olá ${name}, sua análise está pronta!`;
await this.whatsappService.send(phone, message);
```
✅ **Correto:** `this.templateService.resolve('NT-001', { lead_name: name })` — texto vem do banco.

---

## 6. Tracking e Métricas

### 6.1 Eventos de Tracking

Todo ciclo de vida de uma notificação gera eventos persistidos na tabela `NotificationLog`.

| Evento | Quando | Canal | Obrigatório |
| --- | --- | --- | --- |
| `sent` | Publicado na fila | Todos | Sim |
| `delivered` | Confirmação de entrega pelo canal | Push, WhatsApp | Sim |
| `opened` | Usuário abriu a notificação | Push | Quando disponível |
| `clicked` | Usuário clicou no deep link | Push | Quando disponível |
| `failed` | DLQ atingida após retries esgotados | Todos | Sim |
| `opted_out` | Preferência desativada — notificação ignorada | Todos | Sim |

### 6.2 Schema do Evento de Tracking

```typescript
interface NotificationTrackingEvent {
  notification_id: string;
  channel: 'push' | 'whatsapp' | 'inapp' | 'email';
  event: 'sent' | 'delivered' | 'opened' | 'clicked' | 'failed' | 'opted_out';
  cessionario_id_hash: string;  // SHA-256 — nunca em claro
  notification_type: string;
  template_id: string;
  timestamp: string;            // ISO 8601 UTC
  correlation_id: string;
  metadata?: Record<string, unknown>;
}
```

### 6.3 Métricas Mínimas por Canal

| Métrica | Canal | Frequência |
| --- | --- | --- |
| Taxa de entrega (delivered/sent) | Push, WhatsApp | Por hora |
| Taxa de abertura (opened/delivered) | Push | Por dia |
| Taxa de falha (failed/sent) | Todos | Por hora |
| Taxa de opt-out acumulado | Todos | Por semana |
| Volume total por tipo | Todos | Por dia |

---

## 7. Testes (Sandbox, Interceptors, Preview Mode)

### 7.1 Ambiente de Teste

Em desenvolvimento (`NODE_ENV=development`), o `NotificationInterceptor` intercepta todos os envios antes de chegar aos canais reais:

```typescript
// apps/ai/src/notification/interceptors/notification-mock.interceptor.ts
// Ativo quando NODE_ENV=development OU FEATURE_NOTIFICATIONS_MOCK=true
// Armazena todas as notificações em memória + Redis para inspeção
// Endpoint de inspeção: GET /repasse-ai/v1/internal/notifications/mock-inbox
```

### 7.2 Variável de Controle

```env
# .env.example
FEATURE_NOTIFICATIONS_MOCK=true    # dev: intercepta e armazena sem enviar
FEATURE_NOTIFICATIONS_MOCK=false   # staging/prod: envio real
```

### 7.3 Testes Unitários Obrigatórios

```typescript
// apps/ai/src/notification/services/__tests__/notification.service.spec.ts
describe('NotificationService', () => {
  it('deve ignorar canal com opt-out para notificação não-crítica', async () => {
    mockPreferences({ inactivity_warning: { push: false } });
    await notificationService.sendNotification({ templateId: 'NT-006', ... });
    expect(rabbitMQService.publish).not.toHaveBeenCalledWith('notifications', 'push', expect.anything());
    expect(trackingService.track).toHaveBeenCalledWith(expect.any(String), 'push', 'opted_out');
  });

  it('deve enviar notificação crítica mesmo com opt-out configurado', async () => {
    mockPreferences({ whatsapp_otp: { whatsapp: false } }); // tentativa de bloquear crítico
    await notificationService.sendNotification({ templateId: 'NT-002', ... });
    expect(rabbitMQService.publish).toHaveBeenCalledWith('notifications', 'whatsapp', expect.anything());
  });

  it('não deve hardcodar texto — deve chamar templateService.resolve', async () => {
    const spy = jest.spyOn(templateService, 'resolve');
    await notificationService.sendNotification({ templateId: 'NT-001', variables: { ... } });
    expect(spy).toHaveBeenCalledWith('NT-001', expect.any(Object));
  });
});
```

### 7.4 Anti-Exemplos de Push

❌ **Anti-exemplo 3 — Push sem deep link:**
```json
{
  "title": "Nova mensagem",
  "body": "Você tem uma nova mensagem.",
  "data": {}
}
```
✅ **Correto:** `"data": { "deep_link": "/chat/conversations/conv_abc123", "notification_id": "notif_..." }`.

---

❌ **Anti-exemplo 4 — Ausência de opt-out para notificação não-crítica:**
```typescript
// PROIBIDO — envia inactivity_warning ignorando preferência
await pushService.send(userId, template.render(vars));
// sem verificar preferências do usuário
```
✅ **Correto:** verificar `preferenceService.get(cessionarioId, 'inactivity_warning')` antes de publicar na fila.

---

## 8. LGPD e Compliance

> 🔴 **Regras inegociáveis de compliance:**

- **Retenção de logs de tracking:** 90 dias — alinhado com política de retenção do Modelo de Dados (Doc 12).
- **Conteúdo de mensagem:** não persistido no `NotificationLog` — apenas metadados (`template_id`, `variables` com PII mascarada).
- **Número de telefone:** nunca logado em claro — apenas `phone_hash` SHA-256.
- **Opt-out:** solicitação de opt-out atendida em até 48h (automático via API — imediato).
- **Exclusão LGPD:** ao deletar cessionário (soft delete), todas as `NotificationPreference` e `NotificationLog` são anonimizadas — `cessionario_id` substituído por `DELETED_{hash}`.
- **Push subscription:** endpoint VAPID armazenado criptografado; descartado ao revogar permissão.

| Dado | Tratamento em Logs | Retenção |
| --- | --- | --- |
| `phone_number` | Substituído por `phone_hash` | 90 dias |
| `user_id` | Substituído por `user_id_hash` | 90 dias |
| `message_body` | Não persistido | N/A |
| `push_subscription.endpoint` | Não persistido em log | Até revogação |
| `notification_type` | Persistido | 90 dias |
| `template_id` | Persistido | 90 dias |

---

## 9. Glossário

| Termo | Definição |
| --- | --- |
| `notification_id` | ID único de uma instância de notificação (ulid/nanoid) |
| `template_id` | Referência ao template `NT-XXX` no inventário |
| `notification_type` | Slug do tipo (e.g. `agent_response_ready`) |
| `channel` | Canal de entrega: `push`, `whatsapp`, `inapp`, `email` |
| `priority` | `critical` / `high` / `normal` / `low` |
| `deep_link` | URL relativa do Webchat para navegação contextual |
| `opted_out` | Evento de tracking quando canal está desativado |
| `DLQ` | Dead Letter Queue — fila de mensagens após retries esgotados |
| `VAPID` | Voluntary Application Server Identification — protocolo de push |
| `TemplateService` | Serviço que resolve variáveis nos templates do banco |
| `NotificationLog` | Tabela de tracking com retenção 90 dias |
| `NotificationPreference` | Tabela de preferências de canal por usuário |

---

## 10. Backlog de Pendências

| ID | Descrição | Prioridade | Doc Dependente |
| --- | --- | --- | --- |
| NT-P001 | Canal email (Resend.com ou AWS SES) para relatórios periódicos e billing | Baixa — P2 | 17 - Integrações Externas (atualizar) |
| NT-P002 | UI de preferências de notificação no Webchat (painel de configurações) | Média | 09 - Contratos de UI |
| NT-P003 | Definir política de rate limiting global para WhatsApp quando múltiplos tenants compartilham o mesmo número (cenário edge) | Média | 17 - Integrações Externas |

> **Decisões Autônomas Tomadas Neste Documento:**
>
> 1. **[DECISÃO AUTÔNOMA] Email excluído do P0/P1:** alternativa (incluir com Resend.com no P1) descartada por ausência de demanda de relatórios periódicos no MVP e nenhuma configuração de email nas integrações externas (Doc 17). Critério: YAGNI + PG-03 (backend puro focado em chat e IA).
> 2. **[DECISÃO AUTÔNOMA] `message_received_whatsapp` não tem opt-out:** qualquer mensagem recebida via WhatsApp deve gerar notificação in-app, pois o cessionário precisa saber que o lead enviou mensagem. Alternativa (permitir opt-out) descartada por comprometer a operação do negócio. Critério: regra de negócio — lead não pode ser ignorado silenciosamente.
> 3. **[DECISÃO AUTÔNOMA] TTL de mensagem na fila `notif.inapp` é 30 min:** se SSE não está ativa em 30 minutos, a notificação in-app perde relevância e o fallback push já foi acionado. Alternativa (24h) descartada por gerar notificações descontextualizadas ao reconectar. Critério: experiência de usuário.
> 4. **[DECISÃO AUTÔNOMA] `CRITICAL_OPTOUT_ATTEMPT` silencioso:** retornar 200 ao tentar desativar críticos para não revelar quais notificações são indesativáveis (menos superfície de ataque). Alternativa (retornar 403 com mensagem explicativa) descartada por facilitar engenharia reversa dos controles. Critério: security by obscurity mínima para feature de compliance.

---

*Próximo documento do pipeline: D25 — Observabilidade e Logs.*
