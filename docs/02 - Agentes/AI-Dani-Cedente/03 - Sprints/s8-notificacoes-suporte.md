# S8 — Notificações e Suporte

| Campo                | Valor                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S8                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| **Nome**             | Notificações e Suporte                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Tipo**             | Módulo Fullstack (Template B)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **Template**         | B (Módulo Fullstack: por Feature, vertical slice Banco→Backend→Frontend→Wiring→Testes)                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Docs Consultados** | D01 (RN-DCE-020, RN-DCE-021, RN-DCE-024), D05.4, D14 (§4), D21                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Cross-cutting**    | D10 (Glossário)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **REQs cobertos**    | REQ-090, REQ-091, REQ-092, REQ-093, REQ-094, REQ-095, REQ-096, REQ-097, REQ-098, REQ-099, REQ-100, REQ-101, REQ-102, REQ-103, REQ-104, REQ-105, REQ-106, REQ-107, REQ-108, REQ-109                                                                                                                                                                                                                                                                                                                              |
| **Objetivo**         | 12 templates de notificação (T-NTF-001 a T-NTF-012), exchange RabbitMQ `notifications.topic`, 2 workers (WebchatWorker SSE e EmailWorker), CSAT `POST /chat/sessions/:id/csat`, SSE stream `GET /notifications/stream`, preferências `GET/PATCH /notifications/preferences`, opt-out One-Click, DLQ + retry (3× com backoff 1s/5s/30s), notificações críticas imunes a opt-out, rate limiting (30/h webchat, 10/h email), expiração de não lidas em 7 dias, tabela `notification_log` com 6 eventos de tracking |

---

## Critério de Conclusão da S8

Ao final desta sprint:

- `NotificationService.publishNotification()` é o único ponto de entrada — todas as notificações passam por ele, nunca direto ao SendGrid/SSE
- Exchange `notifications.topic` roteia para 3 queues: `notification.webchat` (priority 10), `notification.email` (priority 5), `notification.whatsapp` (Fase 2, definida mas não processada)
- 12 templates (T-NTF-001 a T-NTF-012) em arquivos `.hbs` em `src/modules/notification/templates/` — sem strings hardcoded no código
- Notificações críticas (10 tipos) ignoram opt-out do Cedente
- Apenas `DOCUMENTO_DOSSIE_REJEITADO` e `OPORTUNIDADE_PUBLICADA` são configuráveis pelo Cedente
- `POST /chat/sessions/:id/csat` aceita `{ score: 1–5 }` e persiste em `ChatSession.csat_score`
- CSAT < 3.5 em 24h dispara alerta ao Admin via Sentry/Slack
- Notificações não lidas expiram em 7 dias
- E-mail do Cedente nunca trafega no payload da fila — resolvido pelo `EmailWorker` via `cedente_id → CedenteProfile.email`
- DLQ com > 10 mensagens em 1h → alerta Admin

---

## ⚙️ FEATURE 1 — Infraestrutura de Notificações

### Banco

- [x] **Criar migration da tabela `notification_log`** em `prisma/migrations/`: `id UUID PK DEFAULT gen_random_uuid()`, `cedente_id UUID NOT NULL FK → cedente_profiles.id`, `notification_id VARCHAR(50) NOT NULL UNIQUE`, `event_type VARCHAR(60) NOT NULL`, `channel VARCHAR(20) NOT NULL`, `template_id VARCHAR(60) NOT NULL`, `status VARCHAR(20) NOT NULL DEFAULT 'PENDING'` (valores: `PENDING`, `SENT`, `DELIVERED`, `OPENED`, `CLICKED`, `FAILED`, `BOUNCED`, `UNSUBSCRIBED`), `retry_count SMALLINT NOT NULL DEFAULT 0`, `sent_at TIMESTAMPTZ`, `delivered_at TIMESTAMPTZ`, `opened_at TIMESTAMPTZ`, `clicked_at TIMESTAMPTZ`, `failed_at TIMESTAMPTZ`, `failure_reason TEXT`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`
  - Validação: índice `idx_notification_log_cedente` em `(cedente_id, created_at DESC)`; índice `idx_notification_log_status` em `(status, created_at DESC)`; `notification_id` UNIQUE (idempotência); nenhum campo PII (sem email, sem CPF)

- [x] **Criar migration da tabela `notification_preferences`** em `prisma/migrations/`: `id UUID PK`, `cedente_id UUID NOT NULL UNIQUE FK → cedente_profiles.id`, `email_enabled BOOLEAN NOT NULL DEFAULT true`, `webchat_enabled BOOLEAN NOT NULL DEFAULT true`, `whatsapp_enabled BOOLEAN NOT NULL DEFAULT false`, `preferences JSONB NOT NULL DEFAULT '{}'::jsonb`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`
  - Validação: `cedente_id` UNIQUE (1 registro por Cedente); `preferences` JSONB com estrutura por template type; `whatsapp_enabled DEFAULT false` (Fase 2 desabilitado por padrão)

### Backend — RabbitMQ Exchange e Queues

- [x] **Configurar exchange e queues RabbitMQ** em `src/modules/notification/notification.module.ts`: (1) exchange `notifications.topic` (type: `topic`, durable: true, auto-delete: false); (2) queue `notification.webchat` (durable: true, `x-max-priority: 10`, routing key `notification.webchat.*`); (3) queue `notification.email` (durable: true, `x-max-priority: 5`, routing key `notification.email.*`); (4) queue `notification.whatsapp` (durable: true, routing key `notification.whatsapp.*` — consumidor não ativo na Fase 1); (5) Dead Letter Exchange `notifications.dlx` (type: direct, durable: true); (6) Dead Letter Queue `notification.dlq` (durable: true, binding key `notification.failed`)
  - Validação: `rabbitmq-management` UI mostra 6 recursos criados; exchange `notifications.topic` existe; 3 queues ligadas ao exchange via routing keys corretas; `notification.dlq` ligada ao `notifications.dlx`

### Backend — NotificationService

- [x] **Implementar `NotificationService.publishNotification(dto: PublishNotificationDto)`** em `src/modules/notification/notification.service.ts`: (1) recebe `{ cedente_id, event_type, variables, priority, channels? }`; (2) verifica preferências do Cedente via `NotificationPreferencesService.isEnabled(cedente_id, event_type, channel)`; (3) se `priority = 'critical'` → ignora preferências e envia sempre; (4) gera `notification_id = ulid()` único por notificação; (5) publica no exchange `notifications.topic` com routing key `notification.{channel}.{event_type}`; (6) cria registro em `notification_log` com `status = 'PENDING'`; (7) **NUNCA** inclui email do Cedente no payload — apenas `cedente_id`
  - Validação: notificação crítica enviada mesmo com `email_enabled = false`; `notification.webchat.*` e `notification.email.*` publicados para eventos não-críticos com opt-in; e-mail nunca no payload da fila (verificado em teste unitário); `notification_id` é ULID (não UUID)

- [x] **Implementar `NotificationPreferencesService`** em `src/modules/notification/notification-preferences.service.ts`: (1) `isEnabled(cedenteId, eventType, channel): boolean` — busca `notification_preferences` do Cedente; (2) se `event_type IN ('NOVA_PROPOSTA', 'PROPOSTA_VENCENDO', 'EXTENSAO_ESCROW_SOLICITADA', 'DEPOSITO_ESCROW_CONFIRMADO', 'ESCROW_VENCENDO', 'ZAPSIGN_ENVIADO', 'ZAPSIGN_LEMBRETE_D2', 'ZAPSIGN_LEMBRETE_D4_URGENTE', 'NEGOCIACAO_CONCLUIDA', 'EXTENSAO_ESCROW_APROVADA_AUTO')` → retorna `true` sempre (crítico); (3) para `DOCUMENTO_DOSSIE_REJEITADO` e `OPORTUNIDADE_PUBLICADA` → verifica preferência do Cedente; (4) rate limit: verifica `dani:notif_rate:{cedente_id}:{channel}` no Redis; webchat: 30/h; email: 10/h; notificações `critical` ignoram rate limit
  - Validação: `isEnabled(id, 'NOVA_PROPOSTA', 'email')` → `true` mesmo com `email_enabled = false`; `isEnabled(id, 'OPORTUNIDADE_PUBLICADA', 'email')` com `email_enabled = false` → `false`; 31ª notificação webchat não-crítica → `false` (rate limit)

---

## ⚙️ FEATURE 2 — Workers de Envio

### Backend — WebchatWorker

- [x] **Implementar `WebchatWorker`** em `src/modules/notification/workers/webchat.worker.ts`: (1) consome queue `notification.webchat`; (2) para cada mensagem: resolve template via `TemplateService.render(template_id, variables)`; (3) envia evento SSE ao cliente conectado via `NotificationSseService.push(cedente_id, payload)`; (4) payload SSE: `{ type: 'notification', id: notification_id, event_type, priority, title, body, action_label, action_url, created_at, expires_at: created_at + 7 dias }`; (5) se Cedente offline → persiste `notification_log.status = 'PENDING'` (entregue ao reconectar); (6) se enviado → atualiza `status = 'SENT'`; (7) retry: `x-retry-count` header; max 3; backoff 1s/5s/30s; após 3 falhas → DLQ
  - Validação: Cedente conectado recebe evento SSE imediatamente; Cedente offline → notificação fica `PENDING` e é entregue ao reconectar via `GET /notifications/stream`; após 3 falhas → mensagem vai para `notification.dlq`; `expires_at = created_at + 7 dias` calculado corretamente

- [x] **Implementar `NotificationSseService`** em `src/modules/notification/notification-sse.service.ts`: (1) mantém Map de conexões SSE ativas: `activeConnections: Map<string, Response>` (cedente_id → Express Response); (2) método `registerConnection(cedenteId, res)` — registra conexão, configura headers SSE (`Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`), envia heartbeat `data: ping\n\n` a cada 30s; (3) método `push(cedenteId, payload)` — se Cedente conectado: escreve `data: {JSON}\n\n`; (4) método `removeConnection(cedenteId)` — remove ao desconectar; (5) cleanup automático de conexões mortas (on `res.on('close')`)
  - Validação: heartbeat enviado a cada 30s; conexão removida ao fechar; múltiplos Cedentes com conexões simultâneas não interferem; `cedente_id` como chave garante isolamento de SSE

- [x] **Implementar `GET /api/v1/notifications/stream`** em `NotificationController`: (1) aplica `JwtAuthGuard`; (2) chama `NotificationSseService.registerConnection(cedenteId, res)`; (3) retorna headers SSE corretos; (4) ao conectar: entrega notificações `PENDING` (não lidas, `created_at > now() - 7 dias`) como eventos SSE em sequência; (5) após entrega das pendentes: conexão permanece aberta para novas notificações
  - Validação: request SSE retorna `Content-Type: text/event-stream`; notificações pendentes entregues ao conectar; heartbeat visível a cada 30s; desconexão limpa a entrada do Map

### Backend — EmailWorker

- [x] **Implementar `EmailWorker`** em `src/modules/notification/workers/email.worker.ts`: (1) consome queue `notification.email`; (2) para cada mensagem: resolve `to` via `CedenteProfileRepository.getEmail(cedente_id)` (nunca lê email do payload da fila); (3) renderiza template via `TemplateService.render(template_id, variables)` para HTML e texto plano; (4) envia via `EmailProviderService.send({ to, subject, html, text, unsubscribe_url? })`; (5) templates não-críticos recebem `unsubscribe_url` via One-Click Unsubscribe (RFC 8058); (6) templates críticos **não** têm link de unsubscribe; (7) retry: max 3, backoff 1s/5s/30s; bounce → `status = 'BOUNCED'` + alerta Admin se bounce rate > 5%
  - Validação: e-mail do Cedente resolvido no worker, nunca no payload; template crítico não tem unsubscribe_url; bounce → `notification_log.status = 'BOUNCED'`; após 3 falhas → DLQ

- [x] **Implementar `EmailProviderService`** em `src/modules/notification/email-provider.service.ts`: (1) lê `EMAIL_PROVIDER` env var: `sendgrid` ou `smtp`; (2) se `sendgrid`: usa `@sendgrid/mail` SDK com `SENDGRID_API_KEY`; (3) se `smtp`: usa `nodemailer` com `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`; (4) método `send(options)` abstrato para ambos providers; (5) timeout: 15s por request
  - Validação: `EMAIL_PROVIDER = 'smtp'` usa nodemailer; `EMAIL_PROVIDER = 'sendgrid'` usa SendGrid SDK; credenciais lidas de env vars — nunca hardcoded; timeout de 15s

### Backend — TemplateService

- [x] **Implementar `TemplateService.render(templateId: string, variables: Record<string, string>)`** em `src/modules/notification/template.service.ts`: (1) carrega template `.hbs` de `src/modules/notification/templates/{templateId}.hbs`; (2) compila via `Handlebars.compile(source)(variables)`; (3) se template não encontrado → lança `DaniNotFoundError` com log de warning; (4) sanitiza variáveis antes de inserir no template (sem HTML injection); (5) suporta variantes: `{templateId}.webchat.hbs` (texto simples), `{templateId}.email.html.hbs` (HTML), `{templateId}.email.text.hbs` (texto plano)
  - Validação: 12 arquivos `.hbs` criados (T-NTF-001 a T-NTF-012 — webchat + email HTML + email text); template `NOVA_PROPOSTA.webchat.hbs` renderiza com `proposta_valor` e `oportunidade_id`; template inexistente → log warning + erro; variáveis com `<script>` tags escapadas

- [x] **Criar os 12 templates `.hbs`** em `src/modules/notification/templates/`: (1) `T-NTF-001.NOVA_PROPOSTA.*`; (2) `T-NTF-002.PROPOSTA_VENCENDO.*`; (3) `T-NTF-003.EXTENSAO_ESCROW_SOLICITADA.*`; (4) `T-NTF-004.DEPOSITO_ESCROW_CONFIRMADO.*`; (5) `T-NTF-005.ESCROW_VENCENDO.*`; (6) `T-NTF-006.ZAPSIGN_ENVIADO.*`; (7) `T-NTF-007.ZAPSIGN_LEMBRETE_D2.*`; (8) `T-NTF-008.ZAPSIGN_LEMBRETE_D4_URGENTE.*`; (9) `T-NTF-009.DOCUMENTO_DOSSIE_REJEITADO.*`; (10) `T-NTF-010.OPORTUNIDADE_PUBLICADA.*`; (11) `T-NTF-011.NEGOCIACAO_CONCLUIDA.*`; (12) `T-NTF-012.EXTENSAO_ESCROW_APROVADA_AUTO.*`; cada template tem variante webchat + email HTML + email text; templates críticos (T-NTF-001 a T-NTF-008, T-NTF-011, T-NTF-012) sem `unsubscribe_url`
  - Validação: 12 × 3 = 36 arquivos `.hbs` criados; variáveis conforme coluna "Variáveis" da tabela D21 §3; templates críticos sem `{{unsubscribe_url}}`; conteúdo conforme exemplos do D21 §3.1

---

## ⚙️ FEATURE 3 — Preferências e Opt-out

### Backend — NotificationController (Preferências)

- [x] **Implementar `GET /api/v1/notifications/preferences`** em `NotificationController`: (1) aplica `JwtAuthGuard`; (2) retorna preferências do Cedente autenticado; (3) se não existir → cria default (tudo enabled para configuráveis); (4) response: `{ data: { email_enabled, webchat_enabled, preferences: { DOCUMENTO_DOSSIE_REJEITADO: { email, webchat }, OPORTUNIDADE_PUBLICADA: { email, webchat } } } }`
  - Validação: sem registro existente → cria default e retorna; preferências de tipos críticos **não incluídas** no response (não são configuráveis); Cedente B não acessa preferências do Cedente A

- [x] **Implementar `PATCH /api/v1/notifications/preferences`** em `NotificationController`: (1) aceita body `{ preferences: { DOCUMENTO_DOSSIE_REJEITADO?: { email?, webchat? }, OPORTUNIDADE_PUBLICADA?: { email?, webchat? } } }`; (2) se body contém tipo crítico (ex: `NOVA_PROPOSTA`) → silenciosamente ignora e retorna `ignored_critical: [...]`; (3) atualiza apenas os 2 tipos configuráveis; (4) retorna `{ data: { updated: [...], ignored_critical: [...], updated_at } }`
  - Validação: `PATCH { preferences: { NOVA_PROPOSTA: { email: false } } }` → retorna `ignored_critical: ['NOVA_PROPOSTA']` com HTTP 200; `PATCH { preferences: { DOCUMENTO_DOSSIE_REJEITADO: { email: false } } }` → retorna `updated: ['DOCUMENTO_DOSSIE_REJEITADO']`; `updated_at` atualizado

- [x] **Implementar `POST /api/v1/notifications/unsubscribe`** em `NotificationController`: (1) aceita token One-Click Unsubscribe do link no e-mail (`{ token: string }`); (2) valida token (JWT de curta duração com `cedente_id` e `event_type`); (3) se válido → desativa a preferência correspondente; (4) se inválido/expirado → 400; (5) retorna `{ data: { unsubscribed: true, event_type, updated_at } }`; (6) rastreamento: atualiza `notification_log.status = 'UNSUBSCRIBED'` para notificações futuras daquele tipo
  - Validação: token válido → preferência desativada; token expirado → 400; token de tipo crítico → 400 (críticos não podem ser unsubscribed por token); operação idempotente

---

## ⚙️ FEATURE 4 — CSAT

### Backend — CsatService

- [x] **Implementar `POST /api/v1/chat/sessions/:id/csat`** em `ChatController` (extensão do módulo de Chat de S3): (1) aplica `JwtAuthGuard` + `CedenteIsolationMiddleware`; (2) valida body `{ score: number }` com `@IsInt()`, `@Min(1)`, `@Max(5)`; (3) verifica que `session.cedente_id == cedenteId` (isolamento); (4) verifica que `session.status == 'CLOSED'` → senão 422 `DCE-CHAT-4220_001` ("CSAT só disponível para sessões encerradas."); (5) verifica que `session.csat_score IS NULL` → senão 409 `DCE-CHAT-4090_001` ("CSAT já registrado para esta sessão."); (6) atualiza `ChatSession.csat_score = score`; (7) captura evento PostHog `csat_submitted` com `{ score, session_id: hash(session_id) }`; (8) retorna `{ data: { session_id, csat_score, submitted_at: now() } }` com HTTP 200
  - Validação: `score = 0` → 400; `score = 6` → 400; `score = 3` (inteiro) → 200; sessão ativa (`ACTIVE`) → 422; CSAT duplicado → 409; `session_id` hasheado no PostHog (não UUID raw)

- [x] **Implementar job de alerta CSAT** em `src/modules/notification/csat.scheduler.ts`: (1) `@Cron('0 * * * *')` (a cada hora); (2) busca `ChatMessage` com `csat_score IS NOT NULL AND created_at > now() - INTERVAL '24 hours'`; (3) calcula média `AVG(csat_score)`; (4) se `AVG < 3.5` → dispara alerta ao Admin: (a) log estruturado `level: error` com `avg_csat`, `sample_size`, `window: '24h'`; (b) Sentry event `csat_below_threshold`; (c) evento PostHog `csat_alert_triggered` com `{ avg_score, threshold: 3.5 }`; (5) alerta disparado no máximo 1x por hora (sem spam — flag Redis `dani:csat_alert:{window}` TTL 1h)
  - Validação: CSAT médio de 3.4 em 24h → alerta disparado; CSAT médio de 3.5 → sem alerta; alerta não repetido se Redis flag ativa; `sample_size >= 1` (não alerta para janela sem dados)

---

## ⚙️ FEATURE 5 — DLQ e Monitoramento

### Backend — DLQ Monitor

- [x] **Implementar `DlqMonitorService`** em `src/modules/notification/dlq-monitor.service.ts`: (1) `@Cron('*/5 * * * *')` (a cada 5 minutos); (2) conta mensagens na queue `notification.dlq` via RabbitMQ Management API ou `amqplib.assertQueue({ passive: true })`; (3) se `dlq_count > 10` em janela de 1 hora → dispara alerta: (a) Sentry error `notification_dlq_overflow`; (b) log `level: error` com `dlq_count`; (c) flag Redis `dani:dlq_alert` TTL 1h (sem spam); (4) atualiza `notification_log.status = 'FAILED'` + `failure_reason` para mensagens no DLQ
  - Validação: DLQ com 11 mensagens → alerta disparado; DLQ com 10 → sem alerta; alerta não repetido na mesma hora; `notification_log` atualizado com `FAILED`

---

## 🖥️ FRONTEND (Widget Chat — Notificações)

### Componente ProactiveToast

- [x] **Implementar `ProactiveToast`** em `src/components/chat/ProactiveToast.tsx`: (1) renderiza notificação SSE recebida como toast sobreposto ao widget; (2) campos exibidos: `title` (negrito), `body` (texto), botão `action_label` (abre `action_url`); (3) auto-dismiss em 8s para `priority = 'high'`; (4) **não** auto-dismiss para `priority = 'critical'` (requer clique); (5) máximo 3 toasts simultâneos — fila FIFO; (6) acessibilidade: `role="alert"`, `aria-live="assertive"` para críticos; `aria-live="polite"` para high/normal; (7) animação de entrada (slide from right) e saída (fade out)
  - Validação: toast crítico persiste até clique; toast high desaparece em 8s; máximo 3 simultâneos (4º fica na fila); botão de ação redireciona para `action_url`; `aria-live` correto por prioridade

### Componente CsatRating

- [x] **Implementar `CsatRating`** em `src/components/chat/CsatRating.tsx`: (1) exibido após `ChatSession.status = 'CLOSED'` e `csat_score IS NULL`; (2) escala de 1 a 5 com ícones estrela; (3) ao clicar → envia `POST /chat/sessions/:id/csat` com score selecionado; (4) após envio bem-sucedido: exibe "Obrigado pelo seu feedback!" e oculta o componente; (5) estado **loading** durante request (estrelas desabilitadas); (6) estado **error**: "Não foi possível registrar sua avaliação. Tente novamente."; (7) CSAT já registrado (`csat_score != null`) → componente não exibido
  - Validação: 5 estrelas clicáveis; seleção envia score correto (1–5); feedback confirmado após sucesso; componente oculto se `csat_score` já existe; acessibilidade: cada estrela com `aria-label="Nota X de 5"`

### Componente NotificationBadge

- [x] **Implementar `NotificationBadge`** em `src/components/chat/NotificationBadge.tsx`: (1) exibe contador de notificações não lidas (`PENDING` + não expiradas); (2) ao clicar no widget (ícone da Dani) → conecta ao `GET /notifications/stream` e limpa badge; (3) atualiza em tempo real via SSE (evento `notification` incrementa contador, evento `tracking.delivered` decrementa); (4) `aria-label="N notificações não lidas"`; (5) contador zerado após todas as notificações pendentes serem entregues via SSE
  - Validação: 3 notificações pendentes → badge exibe "3"; ao reconectar → badge zerado após entrega; SSE incrementa contador em tempo real

---

## 🔀 Cross-Módulo

- Rate limiting de notificações (30/h webchat) compartilha o prefixo Redis com rate limit de chat de S2 — usar chave distinta: `dani:notif_rate:{cedente_id}:webchat` vs `rate_limit:{cedente_id}` (chat)
- Eventos de domínio que disparam notificações (todos publicados pelo módulo de origem via `NotificationService`): S4 → `opportunity.published` (T-NTF-010); S5 → `dossier.document_rejected` (T-NTF-009); S6 → `proposal.received` (T-NTF-001), `proposal.expiring_24h` (T-NTF-002); S7 → `escrow.extension_requested` (T-NTF-003), `escrow.deposit_confirmed` (T-NTF-004), `escrow.expiring_2d` (T-NTF-005), `zapsign.contract_sent` (T-NTF-006), `zapsign.reminder_d2` (T-NTF-007), `zapsign.reminder_d4` (T-NTF-008), `escrow.released` (T-NTF-011), `escrow.extension_auto_approved` (T-NTF-012)
- CSAT (`POST /chat/sessions/:id/csat`) é extensão do `ChatModule` de S3 — implementar no `ChatController` já existente, não criar novo controller
- Alerta CSAT < 3.5 e alerta DLQ usam Sentry e PostHog — dependem da configuração de observabilidade de S9

---

## 🧪 TESTES

- [x] **Criar `src/modules/notification/notification.service.spec.ts`** com testes unitários: (1) `publishNotification` com tipo crítico e Cedente com opt-out → publicado na fila (opt-out ignorado); (2) `publishNotification` com tipo configurável e `email_enabled = false` → não publicado no canal email; (3) `publishNotification` não inclui e-mail do Cedente no payload da fila; (4) rate limit: 30ª notificação webchat passa; 31ª não-crítica é bloqueada; notificação crítica passa no 31º; (5) `notification_id` gerado como ULID (formato verificado)
  - Validação: 5 cenários; mock do RabbitMQ; mock do Redis (rate limit); e-mail ausente no payload verificado por `expect(payload).not.toHaveProperty('to')`

- [x] **Criar `src/modules/notification/template.service.spec.ts`** com testes: (1) renderiza template `NOVA_PROPOSTA.webchat.hbs` com variáveis corretas; (2) template inexistente → lança `DaniNotFoundError`; (3) variável com `<script>alert()` → escapada no output; (4) todos os 12 templates renderizam sem erro (smoke test)
  - Validação: 4 cenários; templates carregados de arquivo real (não mock); output HTML validado

- [x] **Criar `src/modules/notification/csat.service.spec.ts`** com testes: (1) CSAT `score = 5` → 200; (2) CSAT `score = 0` → 400; (3) CSAT `score = 6` → 400; (4) CSAT em sessão ativa → 422 `DCE-CHAT-4220_001`; (5) CSAT duplicado → 409 `DCE-CHAT-4090_001`; (6) média CSAT = 3.4 → alerta disparado; (7) média CSAT = 3.5 → sem alerta
  - Validação: 7 cenários; mock do Prisma; mock do PostHog; mock do Sentry

- [x] **Criar `test/notification.e2e-spec.ts`** com Supertest: (1) `GET /notifications/stream` → retorna SSE stream com headers corretos; (2) `GET /notifications/preferences` → 200 com preferences; (3) `PATCH /notifications/preferences` com tipo crítico → 200 com `ignored_critical`; (4) `PATCH /notifications/preferences` com tipo configurável → 200 com `updated`; (5) `POST /chat/sessions/:id/csat` com score válido → 200; (6) `POST /csat` com sessão ativa → 422
  - Validação: 6 cenários E2E; SSE testado com `EventSource` ou `supertest` com stream response; banco de teste real

---

## 🔍 AUTO-VERIFICAÇÃO S8

| Check              | Critério                                                                                                                                                                                                             | Status |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura    | `NotificationService`, `WebchatWorker`, `EmailWorker`, `TemplateService`, `NotificationPreferencesService`, `DlqMonitorService` — nomes exatos; tabelas `notification_log` e `notification_preferences` conforme D21 | [x]    |
| #2 12 templates    | T-NTF-001 a T-NTF-012 criados em arquivos `.hbs`; nomes exatos conforme D21 §3; variáveis corretas por template; 10 críticos sem unsubscribe_url; 2 configuráveis com unsubscribe_url                                | [x]    |
| #3 Críticos imunes | 10 tipos críticos ignoram opt-out E rate limit; 2 tipos configuráveis (`DOCUMENTO_DOSSIE_REJEITADO`, `OPORTUNIDADE_PUBLICADA`) respeitam preferências                                                                | [x]    |
| #4 Assíncrono      | Todo envio via `NotificationService.publishNotification()` → RabbitMQ; zero chamadas diretas ao SendGrid/SSE no ciclo request-response                                                                               | [x]    |
| #5 PII na fila     | E-mail do Cedente nunca no payload do RabbitMQ; resolvido pelo `EmailWorker` via `cedente_id → CedenteProfile.email`; sem CPF ou nome completo no payload                                                            | [x]    |
| #6 Rate limits     | Webchat: 30/h; Email: 10/h; chaves Redis `dani:notif_rate:{cedente_id}:{channel}`; notificações `critical` ignoram rate limit                                                                                        | [x]    |
| #7 Retry e DLQ     | Max 3 tentativas; backoff 1s/5s/30s; após 3 falhas → `notification.dlq`; DLQ > 10 em 1h → alerta Admin                                                                                                               | [x]    |
| #8 Expiração       | Notificações não lidas expiram em 7 dias (`expires_at = created_at + 7 dias`); não entregues após expiração                                                                                                          | [x]    |
| #9 CSAT            | Escala 1–5; sessão `CLOSED` obrigatória; CSAT duplicado → 409; CSAT médio < 3.5/5 em 24h → alerta (Sentry + PostHog)                                                                                                 | [x]    |
| #10 SSE Stream     | `GET /notifications/stream` com headers corretos; heartbeat a cada 30s; notificações PENDING entregues ao conectar; isolamento por `cedente_id`                                                                      | [x]    |
| #11 Anti-scaffold  | `TemplateService` com render real via Handlebars; `WebchatWorker` com lógica real de SSE push; `EmailWorker` com lógica real de SMTP/SendGrid; sem stubs que retornam dados fixos                                    | [x]    |
| #12 Cobertura REQs | REQ-090 a REQ-109 — todos com ≥1 item no checklist                                                                                                                                                                   | [x]    |
