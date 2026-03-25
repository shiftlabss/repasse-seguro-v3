# 21 - Notificações, Templates e Implementação

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend Lead, Frontend Lead, QA |
| **Escopo** | Sistema de notificações do CRM — canais, 8 templates, arquitetura RabbitMQ assíncrona, regras de prioridade e controle de preferências |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | Doc 01.3 RN-107–RN-112 · Doc 01.5 RN-195 · Doc 02 Stacks CRM · Doc 17 Integrações · Doc 20 Error Handling |

---

> **TL;DR**
>
> - **Destinatários:** equipe interna (Analistas RS, Coordenadores RS, Admin RS). NÃO Cedentes/Cessionários.
> - **2 canais:** in-app (dashboard badge + painel de alertas) e e-mail (Resend — alertas críticos e relatório semanal).
> - **8 templates:** NOT-CRM-01 a NOT-CRM-08 documentados com payload, destinatário e prioridade.
> - **Arquitetura:** RabbitMQ assíncrono — fila `crm.notifications` com dead-letter queue.
> - **Regra invariante:** notificações críticas de SLA (NOT-CRM-03) nunca podem ser desabilitadas (RN-195).

---

## 1. Visão Geral do Sistema

### 1.1 Canais de Notificação

| Canal | Tecnologia | Uso |
|---|---|---|
| **In-app** | Supabase Realtime (WebSocket) + badge + painel | Todas as notificações em tempo real — badge no sino, lista de alertas |
| **E-mail** | Resend API | Alertas críticos (SLA P0/P1) e relatório semanal |

> **Canais fora do escopo do CRM:** Push notification mobile (sem app nativo na v1.0), WhatsApp (canal de comunicação com Cedentes/Cessionários — não equipe interna), SMS (não aprovado no baseline).

### 1.2 Hierarquia de Prioridade (RN-195)

| Prioridade | Tipo | Canais obrigatórios | Pode ser desabilitada? |
|---|---|---|---|
| **Crítica** | SLA 100% vencido · Integração falhando > 1h · Caso em risco de cancelamento | In-app + E-mail | **Nunca** |
| **Alta** | SLA 80% consumido · Follow-up vencido | In-app + E-mail | **Nunca** |
| **Média** | Novo Match · Documento recebido · Proposta expirada | In-app ou E-mail | Sim (usuário desabilita e-mail) |
| **Baixa** | Relatório semanal · Contato inativo · Arquivamento automático | E-mail apenas | Sim |

### 1.3 Arquitetura Geral

```
[Evento de domínio ocorre no CRM]
        │
        ▼
[Service publica na fila crm.notifications]
        │
        ▼
[NotificationsConsumer consome e roteia]
        │
        ├──► In-app: INSERT em notifications_table
        │          │
        │          └──► Supabase Realtime broadcast → frontend
        │
        └──► E-mail: Resend API (para Crítico, Alto, e relatórios)
```

---

## 2. Arquitetura RabbitMQ

### 2.1 Configuração da Fila

```typescript
// notifications.module.ts
RabbitMQModule.forRoot({
  exchanges: [
    { name: 'crm.direct', type: 'direct' },
    { name: 'crm.dlx', type: 'direct' },
  ],
  queues: [
    {
      name: 'crm.notifications',
      options: {
        durable: true,
        arguments: {
          'x-dead-letter-exchange': 'crm.dlx',
          'x-dead-letter-routing-key': 'crm.notifications.dlq',
          'x-message-ttl': 86400000, // 24h TTL
        },
      },
    },
    {
      name: 'crm.notifications.dlq',
      options: { durable: true },
    },
  ],
});
```

### 2.2 Payload Padrão de Notificação

```typescript
// crm-notification.types.ts
interface CrmNotificationPayload {
  id: string;                          // UUID — idempotência
  template: CrmNotificationTemplate;  // NOT-CRM-01 a NOT-CRM-08
  priority: 'CRITICA' | 'ALTA' | 'MEDIA' | 'BAIXA';
  recipient: {
    user_id: string;
    role: CrmRole;
    email: string;
  };
  channels: ('IN_APP' | 'EMAIL')[];
  context: Record<string, string | number>;  // Variáveis do template
  case_id?: string;
  created_at: string;                  // ISO8601
}
```

### 2.3 Publicação de Notificação

```typescript
// Como qualquer service publica uma notificação
@Injectable()
export class CasesService {
  constructor(
    @InjectAmqpConnection() private readonly amqp: AmqpConnection,
  ) {}

  async assignCase(caseId: string, analystId: string): Promise<void> {
    // ... lógica de atribuição ...

    await this.amqp.publish('crm.direct', 'crm.notifications', {
      id: randomUUID(),
      template: CrmNotificationTemplate.NEW_CASE_ASSIGNED,
      priority: 'MEDIA',
      recipient: {
        user_id: analyst.id,
        role: analyst.role,
        email: analyst.email,
      },
      channels: ['IN_APP', 'EMAIL'],
      context: {
        case_identifier: 'RS-0042',
        analyst_name: analyst.name,
        empreendimento: 'Residencial Park 2',
        case_url: `https://crm.repasseseguro.com.br/cases/${caseId}`,
      },
      case_id: caseId,
      created_at: new Date().toISOString(),
    } satisfies CrmNotificationPayload);
  }
}
```

### 2.4 Consumer de Notificações

```typescript
// notifications.consumer.ts
@Injectable()
export class NotificationsConsumer {
  constructor(
    private readonly inAppService: InAppNotificationsService,
    private readonly emailService: EmailService,
    private readonly logger: PinoLogger,
  ) {}

  @RabbitSubscribe({
    exchange: 'crm.direct',
    routingKey: 'crm.notifications',
    queue: 'crm.notifications',
  })
  async handle(payload: CrmNotificationPayload): Promise<Nack | undefined> {
    try {
      // Idempotência — verificar se notificação já foi processada
      const alreadyProcessed = await this.inAppService.exists(payload.id);
      if (alreadyProcessed) return;

      const promises: Promise<void>[] = [];

      if (payload.channels.includes('IN_APP')) {
        promises.push(this.inAppService.create(payload));
      }

      if (payload.channels.includes('EMAIL')) {
        // Verificar preferências do usuário (exceto CRITICA e ALTA)
        if (payload.priority !== 'CRITICA' && payload.priority !== 'ALTA') {
          const prefs = await this.inAppService.getUserPreferences(payload.recipient.user_id);
          if (!prefs.email_enabled) {
            promises.splice(promises.indexOf(this.emailService.send(payload) as any), 1);
          } else {
            promises.push(this.emailService.send(payload));
          }
        } else {
          promises.push(this.emailService.send(payload));
        }
      }

      await Promise.allSettled(promises);
    } catch (error) {
      this.logger.error({ type: 'notification_consumer_error', payload_id: payload.id, error });
      return new Nack(false); // → DLQ
    }
  }
}
```

---

## 3. Canal In-App

### 3.1 Schema de Banco de Dados

```sql
-- Tabela de notificações in-app (Supabase)
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  notification_uuid UUID UNIQUE NOT NULL,  -- idempotência
  user_id UUID NOT NULL REFERENCES users(id),
  template VARCHAR(20) NOT NULL,           -- NOT-CRM-XX
  priority VARCHAR(10) NOT NULL,
  title VARCHAR(200) NOT NULL,
  body TEXT NOT NULL,
  action_url TEXT,
  case_id UUID REFERENCES cases(id),
  read_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_unread
  ON notifications(user_id, read_at)
  WHERE read_at IS NULL;
```

### 3.2 Realtime — Badge de Notificação

```typescript
// hooks/useNotifications.ts — subscribe a notificações do usuário logado
export function useNotifications() {
  const { user } = useAuth();
  const [unreadCount, setUnreadCount] = useState(0);
  const [notifications, setNotifications] = useState<Notification[]>([]);

  useEffect(() => {
    if (!user) return;

    // Carregar notificações existentes
    fetchUnreadNotifications(user.id).then(setNotifications);

    // Subscribe a novas notificações via Supabase Realtime
    const channel = supabase
      .channel(`notifications:${user.id}`)
      .on(
        'postgres_changes',
        {
          event: 'INSERT',
          schema: 'public',
          table: 'notifications',
          filter: `user_id=eq.${user.id}`,
        },
        (payload) => {
          setNotifications((prev) => [payload.new as Notification, ...prev]);
          setUnreadCount((c) => c + 1);

          // Toast para notificações críticas e altas
          if (['CRITICA', 'ALTA'].includes(payload.new.priority)) {
            toast.warning(payload.new.title, { description: payload.new.body });
          }
        }
      )
      .subscribe();

    return () => { supabase.removeChannel(channel); };
  }, [user?.id]);

  const markAsRead = async (notificationId: string) => {
    await supabase
      .from('notifications')
      .update({ read_at: new Date().toISOString() })
      .eq('id', notificationId);
    setUnreadCount((c) => Math.max(0, c - 1));
  };

  return { notifications, unreadCount, markAsRead };
}
```

### 3.3 Painel de Alertas

O componente `NotificationBell.tsx` na topbar exibe:
- Badge numérico vermelho com contagem de não lidas
- Dropdown com lista de notificações recentes (últimas 20)
- Clique na notificação → navega para `action_url` + marca como lida
- Link "Ver todas" → painel completo de alertas

---

## 4. Canal E-mail (Resend)

### 4.1 Configuração

```typescript
// email.service.ts
@Injectable()
export class EmailService {
  private readonly client: Resend;
  private readonly FROM = 'CRM Repasse Seguro <crm@repasseseguro.com.br>';

  constructor() {
    this.client = new Resend(process.env.RESEND_API_KEY);
  }

  async send(payload: CrmNotificationPayload): Promise<void> {
    const template = this.renderTemplate(payload);

    await this.client.emails.send({
      from: this.FROM,
      to: [payload.recipient.email],
      subject: template.subject,
      html: template.html,
      tags: [
        { name: 'module', value: 'crm' },
        { name: 'template', value: payload.template },
        { name: 'priority', value: payload.priority },
      ],
    });
  }
}
```

### 4.2 Limitações do Canal E-mail

- Apenas alertas de prioridade **Crítica** e **Alta** são enviados por e-mail obrigatoriamente.
- Alertas de prioridade **Média** e **Baixa** só enviam e-mail se o usuário não desabilitou nas preferências.
- Notificações **NOT-CRM-07** (relatório semanal) são enviadas sempre por e-mail, independente de preferências (é o único entregável do canal baixa prioridade não desabilitável).

---

## 5. Templates de Notificação

### NOT-CRM-01 — Novo Caso Atribuído

| Campo | Valor |
|---|---|
| **ID** | NOT-CRM-01 |
| **Gatilho** | Caso atribuído/reatribuído a um Analista RS |
| **Destinatário** | Analista RS que recebeu o Caso |
| **Prioridade** | Média |
| **Canais** | In-app + E-mail |
| **Pode desabilitar** | Sim (e-mail) |

**Payload de contexto:**
```json
{
  "analyst_name": "Ana Lima",
  "case_identifier": "RS-0042",
  "empreendimento": "Residencial Park 2",
  "scenario": "B",
  "assigned_by": "Carlos Souza (Coordenador RS)",
  "case_url": "https://crm.repasseseguro.com.br/cases/uuid"
}
```

**In-app:**
- Título: "Novo Caso atribuído: RS-0042"
- Corpo: "O Caso RS-0042 — Residencial Park 2 (Cenário B) foi atribuído a você por Carlos Souza."

**E-mail:**
- Assunto: "CRM — Novo Caso atribuído a você: RS-0042"
- Corpo: Template HTML com resumo do Caso e botão "Abrir Caso".

---

### NOT-CRM-02 — SLA Próximo do Vencimento (Atenção)

| Campo | Valor |
|---|---|
| **ID** | NOT-CRM-02 |
| **Gatilho** | SLA do Caso atinge 80% consumido (RN-107) — estado âmbar |
| **Destinatário** | Analista RS responsável + Coordenador RS |
| **Prioridade** | Alta |
| **Canais** | In-app + E-mail |
| **Pode desabilitar** | **Nunca** (RN-195) |

**Payload de contexto:**
```json
{
  "case_identifier": "RS-0042",
  "current_state": "VERIFICACAO",
  "sla_percent": 80,
  "deadline": "2026-03-25T18:00:00-03:00",
  "hours_remaining": 14,
  "analyst_name": "Ana Lima",
  "case_url": "https://crm.repasseseguro.com.br/cases/uuid"
}
```

**In-app:**
- Título: "⚠️ SLA em atenção: RS-0042"
- Corpo: "O Caso RS-0042 consumiu 80% do SLA no estado VERIFICACAO. Prazo em 14h."

**E-mail:**
- Assunto: "⚠️ ATENÇÃO — SLA do Caso RS-0042 a 80%"

---

### NOT-CRM-03 — SLA Vencido (Crítico)

| Campo | Valor |
|---|---|
| **ID** | NOT-CRM-03 |
| **Gatilho** | SLA do Caso atinge 100% consumido (RN-107) — estado vermelho |
| **Destinatário** | Analista RS responsável + Coordenador RS + Admin RS |
| **Prioridade** | Crítica |
| **Canais** | In-app + E-mail |
| **Pode desabilitar** | **Nunca** (RN-195) |

**Payload de contexto:**
```json
{
  "case_identifier": "RS-0042",
  "current_state": "VERIFICACAO",
  "sla_percent": 100,
  "deadline": "2026-03-25T18:00:00-03:00",
  "hours_overdue": 3,
  "analyst_name": "Ana Lima",
  "case_url": "https://crm.repasseseguro.com.br/cases/uuid"
}
```

**In-app:**
- Título: "🔴 SLA VENCIDO: RS-0042"
- Corpo: "O Caso RS-0042 está com SLA vencido há 3 horas no estado VERIFICACAO. Ação imediata necessária."

**E-mail:**
- Assunto: "🔴 CRÍTICO — SLA VENCIDO no Caso RS-0042"

---

### NOT-CRM-04 — Proposta Aceita pelo Cessionário

| Campo | Valor |
|---|---|
| **ID** | NOT-CRM-04 |
| **Gatilho** | Proposta aceita por um Cessionário (match confirmado) — requer ação do Analista |
| **Destinatário** | Analista RS responsável pelo Caso |
| **Prioridade** | Alta |
| **Canais** | In-app + E-mail |
| **Pode desabilitar** | Sim (e-mail) |

**Payload de contexto:**
```json
{
  "case_identifier": "RS-0042",
  "empreendimento": "Residencial Park 2",
  "accepted_value": "280000.00",
  "cessionario_name_masked": "Ma**** S****",
  "next_action": "Registre o Match e avance o Caso para Negociação",
  "case_url": "https://crm.repasseseguro.com.br/cases/uuid/negotiations"
}
```

**In-app:**
- Título: "Proposta aceita no Caso RS-0042"
- Corpo: "Cessionário aceitou a proposta de R$ 280.000,00. Registre o Match para avançar o Caso."

**E-mail:**
- Assunto: "CRM — Proposta aceita no Caso RS-0042 — Ação necessária"

---

### NOT-CRM-05 — Dossiê Aprovado — Próximo Passo

| Campo | Valor |
|---|---|
| **ID** | NOT-CRM-05 |
| **Gatilho** | Todos os documentos obrigatórios do Dossiê aprovados (incluindo comprovante de depósito Escrow) |
| **Destinatário** | Analista RS responsável pelo Caso |
| **Prioridade** | Alta |
| **Canais** | In-app + E-mail |
| **Pode desabilitar** | Sim (e-mail) |

**Payload de contexto:**
```json
{
  "case_identifier": "RS-0042",
  "empreendimento": "Residencial Park 2",
  "approved_by": "Carlos Souza (Coordenador RS)",
  "next_action": "Avance o Caso para Fechamento",
  "case_url": "https://crm.repasseseguro.com.br/cases/uuid/dossier"
}
```

**In-app:**
- Título: "Dossiê aprovado: RS-0042"
- Corpo: "Todos os documentos do Dossiê foram aprovados. Avance o Caso RS-0042 para Fechamento."

---

### NOT-CRM-06 — ZapSign Totalmente Assinado — Próximo Passo

| Campo | Valor |
|---|---|
| **ID** | NOT-CRM-06 |
| **Gatilho** | Todos os signatários assinaram o envelope ZapSign (RN-189) |
| **Destinatário** | Analista RS responsável pelo Caso |
| **Prioridade** | Alta |
| **Canais** | In-app + E-mail |
| **Pode desabilitar** | Sim (e-mail) |

**Payload de contexto:**
```json
{
  "case_identifier": "RS-0042",
  "empreendimento": "Residencial Park 2",
  "signed_at": "2026-03-23T14:32:00-03:00",
  "pdf_url": "https://crm.repasseseguro.com.br/cases/uuid/dossier",
  "next_action": "PDF assinado adicionado ao Dossiê. Verifique os critérios de Fechamento.",
  "case_url": "https://crm.repasseseguro.com.br/cases/uuid/dossier"
}
```

**In-app:**
- Título: "Contrato assinado: RS-0042"
- Corpo: "Todas as partes assinaram o contrato do Caso RS-0042. PDF adicionado ao Dossiê."

---

### NOT-CRM-07 — Relatório Semanal Gerado

| Campo | Valor |
|---|---|
| **ID** | NOT-CRM-07 |
| **Gatilho** | Cron job semanal (toda segunda-feira, 08:00 America/Fortaleza) |
| **Destinatário** | Coordenador RS + Admin RS |
| **Prioridade** | Baixa |
| **Canais** | E-mail apenas |
| **Pode desabilitar** | Não (é o entregável principal do relatório semanal — RN-150) |

**Payload de contexto:**
```json
{
  "period_from": "2026-03-16",
  "period_to": "2026-03-22",
  "total_cases_active": 134,
  "cases_concluded_period": 8,
  "sla_compliance_percent": "72.4",
  "top_bottleneck_state": "VERIFICACAO",
  "report_url": "https://crm.repasseseguro.com.br/reports/weekly/uuid",
  "download_url": "https://crm.repasseseguro.com.br/reports/export/job-id"
}
```

**E-mail:**
- Assunto: "CRM Repasse Seguro — Relatório Semanal (16 a 22 de março de 2026)"
- Corpo: Template HTML com KPIs principais, link para relatório completo e botão de download.

---

### NOT-CRM-08 — Redistribuição de Casos

| Campo | Valor |
|---|---|
| **ID** | NOT-CRM-08 |
| **Gatilho** | Redistribuição de Casos entre Analistas (por Coordenador RS ou Admin RS) |
| **Destinatário** | Analista RS que recebeu Casos + Analista RS que perdeu Casos (se houver) |
| **Prioridade** | Média |
| **Canais** | In-app + E-mail |
| **Pode desabilitar** | Sim (e-mail) |

**Payload de contexto — Analista que recebe:**
```json
{
  "analyst_name": "Ana Lima",
  "cases_received": 5,
  "redistributed_by": "Carlos Souza (Coordenador RS)",
  "case_identifiers": ["RS-0038", "RS-0039", "RS-0040", "RS-0041", "RS-0042"],
  "pipeline_url": "https://crm.repasseseguro.com.br/pipeline"
}
```

**Payload de contexto — Analista que perde:**
```json
{
  "analyst_name": "Pedro Costa",
  "cases_removed": 5,
  "redistributed_by": "Carlos Souza (Coordenador RS)",
  "reason": "Redistribuição de carga de trabalho"
}
```

**In-app (recebeu):**
- Título: "5 Casos atribuídos a você"
- Corpo: "Carlos Souza redistribuiu 5 Casos para você. RS-0038, RS-0039, RS-0040, RS-0041, RS-0042."

**In-app (perdeu):**
- Título: "5 Casos redistribuídos"
- Corpo: "5 Casos foram redistribuídos da sua carteira por Carlos Souza."

---

## 6. Tabela Resumo de Templates

| Template | Gatilho | Destinatário | Prioridade | In-app | E-mail | Desabilitável |
|---|---|---|---|---|---|---|
| NOT-CRM-01 | Caso atribuído | Analista RS | Média | ✅ | ✅ | Sim (e-mail) |
| NOT-CRM-02 | SLA 80% — âmbar | Analista RS + Coord RS | Alta | ✅ | ✅ | **Nunca** |
| NOT-CRM-03 | SLA 100% — vencido | Analista RS + Coord RS + Admin RS | Crítica | ✅ | ✅ | **Nunca** |
| NOT-CRM-04 | Proposta aceita | Analista RS | Alta | ✅ | ✅ | Sim (e-mail) |
| NOT-CRM-05 | Dossiê aprovado | Analista RS | Alta | ✅ | ✅ | Sim (e-mail) |
| NOT-CRM-06 | ZapSign assinado | Analista RS | Alta | ✅ | ✅ | Sim (e-mail) |
| NOT-CRM-07 | Relatório semanal | Coord RS + Admin RS | Baixa | — | ✅ | Não |
| NOT-CRM-08 | Redistribuição | Analista RS (recebeu/perdeu) | Média | ✅ | ✅ | Sim (e-mail) |

---

## 7. Preferências de Notificação do Usuário

```typescript
// Endpoint para atualizar preferências
// PATCH /api/v1/crm/team/me/notification-preferences

interface NotificationPreferences {
  email_media_enabled: boolean;      // NOT-CRM-01, NOT-CRM-04, NOT-CRM-05, NOT-CRM-06, NOT-CRM-08
  email_baixa_enabled: boolean;      // NOT-CRM-07 (não afeta — sempre enviado)
  // Prioridade Crítica e Alta: NUNCA desabilitáveis (RN-195)
}

// Regra de negócio no service
function canDisable(template: CrmNotificationTemplate, channel: 'EMAIL'): boolean {
  const nonDisablable = [
    CrmNotificationTemplate.SLA_WARNING,     // NOT-CRM-02
    CrmNotificationTemplate.SLA_EXPIRED,     // NOT-CRM-03
    CrmNotificationTemplate.WEEKLY_REPORT,   // NOT-CRM-07 (sempre por e-mail)
  ];
  return !nonDisablable.includes(template);
}
```

---

## 8. Cron Jobs de Notificação

| Job | Schedule | Descrição |
|---|---|---|
| `sla-monitor` | A cada 5 minutos | Verifica Casos com SLA em atenção (80%) ou vencido (100%). Publica NOT-CRM-02 e NOT-CRM-03. |
| `weekly-report` | Segunda-feira 08:00 America/Fortaleza | Gera relatório semanal e publica NOT-CRM-07. |
| `follow-up-reminder` | Diariamente 07:00 America/Fortaleza | Verifica follow-ups vencidos e publica alerta de prioridade Alta. |
| `escrow-fallback` | A cada 1 hora | Verifica Casos em FORMALIZACAO com depósito esperado mas não confirmado há > 24h. |

```typescript
// sla-monitor.job.ts
@Cron('*/5 * * * *', { timeZone: 'America/Fortaleza' })
async checkSlaAlerts(): Promise<void> {
  // Buscar Casos com SLA >= 80% não notificados recentemente
  const cases = await this.casesRepository.findCasesWithSlaThreshold({
    threshold: 80,
    notNotifiedInLastMinutes: 60,  // Evitar spam de alertas
  });

  for (const caso of cases) {
    const percent = caso.sla.percent_consumed;
    const priority = percent >= 100 ? 'CRITICA' : 'ALTA';
    const template = percent >= 100
      ? CrmNotificationTemplate.SLA_EXPIRED
      : CrmNotificationTemplate.SLA_WARNING;

    // Notificar Analista RS
    await this.publishNotification({ template, priority, recipient: caso.analyst, ... });

    // Notificar Coordenador RS e Admin RS para SLA vencido
    if (percent >= 100) {
      await this.publishNotification({ template, priority, recipient: caso.coordinator, ... });
      await this.publishNotification({ template, priority, recipient: caso.admin, ... });
    }
  }
}
```

---

## 9. Implementação — InAppNotificationsService

```typescript
// in-app-notifications.service.ts
@Injectable()
export class InAppNotificationsService {
  constructor(private readonly prisma: PrismaService) {}

  async create(payload: CrmNotificationPayload): Promise<void> {
    const rendered = this.render(payload);

    await this.prisma.notification.create({
      data: {
        notification_uuid: payload.id,
        user_id: payload.recipient.user_id,
        template: payload.template,
        priority: payload.priority,
        title: rendered.title,
        body: rendered.body,
        action_url: rendered.action_url,
        case_id: payload.case_id ?? null,
      },
    });
    // Supabase Realtime transmite o INSERT automaticamente via postgres_changes
  }

  async exists(notificationUuid: string): Promise<boolean> {
    const count = await this.prisma.notification.count({
      where: { notification_uuid: notificationUuid },
    });
    return count > 0;
  }

  async getUserPreferences(userId: string): Promise<NotificationPreferences> {
    const user = await this.prisma.user.findUnique({
      where: { id: userId },
      select: { notification_preferences: true },
    });
    return user?.notification_preferences ?? { email_media_enabled: true, email_baixa_enabled: true };
  }

  private render(payload: CrmNotificationPayload): { title: string; body: string; action_url?: string } {
    const templates = {
      [CrmNotificationTemplate.NEW_CASE_ASSIGNED]: {
        title: `Novo Caso atribuído: ${payload.context.case_identifier}`,
        body: `O Caso ${payload.context.case_identifier} — ${payload.context.empreendimento} foi atribuído a você por ${payload.context.assigned_by}.`,
        action_url: payload.context.case_url,
      },
      // ... demais templates
    };
    return templates[payload.template];
  }
}
```

---

## 10. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 8 templates NOT-CRM-01 a NOT-CRM-08, arquitetura RabbitMQ assíncrona, Supabase Realtime in-app, Resend para e-mail, cron jobs, preferências de usuário e regras de prioridade (RN-195). |
