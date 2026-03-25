# S8a — Notificações

## Repasse Seguro — Módulo Admin

| Campo | Valor |
|---|---|
| **Sprint** | S8a |
| **Nome** | Notificações |
| **Template** | B — Módulo Fullstack |
| **REQs cobertos** | REQ-130, REQ-131, REQ-132 |
| **Docs fonte** | D01.3 §3 (RN-062, RN-063, RN-064), D21 (Notificações, Templates e Implementação) |
| **Itens totais** | 38 |

---

> **Objetivo da sprint:** Implementar o sistema de notificações assíncronas completo — 4 canais (E-mail via Resend, Painel in-app via Supabase Realtime, WhatsApp via Meta Cloud API, SMS via Twilio), 22 templates cobrindo os eventos do ciclo de vida, fila RabbitMQ exchange `rs.notifications` com DLQ, retry policy por canal, fallback WhatsApp→SMS, SLA de entrega (Painel ≤30s, E-mail ≤5min — RN-062), opt-out por categoria, tracking em `notification_logs`, alerta de falha ao Coordenador (RN-064) e centro de notificações in-app com badge.

---

## FEATURE 1 — Infraestrutura de Notificações

### Banco de Dados

**[BE-01] Validar schema `notification_logs` e tabela de preferências**
- Confirmar tabela `notification_logs` com colunas: `id` (UUID PK), `correlation_id`, `template_id`, `event_type`, `channel` (ENUM `email | whatsapp | sms | in_app`), `recipient_id` (UUID FK), `recipient_contact` (hash em logs), `status` (ENUM `PENDING | QUEUED | SENT | DELIVERED | READ | FAILED | FALLBACK_SMS | BOUNCED`), `failure_reason`, `sent_at`, `delivered_at`, `read_at`, `retry_count`, `fallback_channel`, `case_id` (FK nullable), `created_at`
- Confirmar tabela `notification_preferences` com colunas: `user_id`, `channel` (ENUM `email | whatsapp | sms | in_app`), `category` (ENUM `sla_alert | case_lifecycle | financial | formalization | operational`), `enabled` (boolean), `updated_at`; PK composta `(user_id, channel, category)`
- Confirmar índice `INDEX(recipient_id, status, created_at DESC)` em `notification_logs` para queries do centro de notificações
- Confirmar índice `INDEX(status)` para monitoramento de DLQ e FAILED
- Retenção: registros com `created_at < NOW() - INTERVAL '90 days'` são apagados por job cron mensal
- Verificação: `prisma migrate deploy` sem erros; consulta `SELECT COUNT(*) FROM notification_logs` retorna 0 em ambiente limpo

### Backend — Configuração RabbitMQ

**[BE-02] Configurar exchange e queues RabbitMQ para notificações**
- Exchange: `rs.notifications`, tipo `topic`
- Dead Letter Queue: `rs.notifications.dlq` (recebe mensagens após retries esgotados)
- Queues e routing keys:
  - `notifications.email` routing key `email`; concorrência 5 workers; prefetch 10; DLQ TTL 24h
  - `notifications.whatsapp` routing key `whatsapp`; concorrência 3 workers; prefetch 5; DLQ TTL 6h
  - `notifications.sms` routing key `sms`; concorrência 3 workers; prefetch 5; DLQ TTL 6h
  - `notifications.in_app` routing key `in_app`; concorrência 10 workers; prefetch 50; DLQ TTL 1h
- Retry policy por canal:
  - `email`: 3 retries com delays `[30_000, 120_000, 600_000]` (30s, 2min, 10min)
  - `whatsapp`: 2 retries com delays `[60_000, 300_000]` (1min, 5min) → fallback SMS após 2 retries
  - `sms`: 2 retries com delays `[60_000, 300_000]` (1min, 5min) → DLQ + alerta Coordenador
  - `in_app`: 1 retry com delay `[5_000]` (5s) → DLQ (carregado no próximo login)
- Verificação: `rabbitmqadmin list exchanges` inclui `rs.notifications`; `rabbitmqadmin list queues` inclui as 4 queues; DLQ configurada

**[BE-03] Implementar `NotificationService` — core de publicação**
- Método `NotificationService.emit(event: NotificationEvent)`:
  1. `INSERT notification_logs` com `status = 'PENDING'`
  2. Checar `notification_preferences` do destinatário via Redis (cache TTL 5min); se `enabled = false` para o canal/categoria → pular publicação nesse canal
  3. `PUBLISH` para RabbitMQ exchange `rs.notifications` com routing key correspondente ao canal
  4. Atualizar `notification_logs.status = 'QUEUED'`
- **Regra inegociável:** ZERO envio síncrono durante a request principal; todo envio passa obrigatoriamente pela fila RabbitMQ
- Método `NotificationService.markAsRead(notificationId, userId)`: setar `read_at = NOW()`, `status = 'READ'`; verificar que `recipient_id = userId` (403 se diferente)
- Verificação: unit test confirma que emit() retorna antes do envio ocorrer; status em `notification_logs` muda de PENDING → QUEUED

---

## FEATURE 2 — Workers de Entrega por Canal

### Backend

**[BE-04] Implementar worker de E-mail via Resend**
- Queue: `notifications.email`; provedor: Resend API
- Rate limit: 10 e-mails/segundo global; 5 e-mails/minuto por destinatário (Redis sliding window)
- Ao consumir mensagem: `POST https://api.resend.com/emails` com payload `EmailPayload` (to, subject, template_id, variables, correlation_id, priority)
- Sucesso: `UPDATE notification_logs SET status = 'SENT', sent_at = NOW()`
- Falha (4xx/5xx): NACK → retry queue; após 3 retries esgotados → DLQ + `UPDATE status = 'FAILED', failure_reason = error.message` + emitir evento `DELIVERY_FAILED` ao Coordenador
- Webhook bounce Resend: `POST /v1/webhooks/resend` recebe bounce event → `UPDATE status = 'BOUNCED', failure_reason = bounce_reason` + alerta ao Coordenador (RN-064)
- Templates React Email: 22 arquivos em `templates/email/` (rs_caso_captado, rs_dossie_incompleto, ..., rs_mediacao_iniciada); cada template compilado com variáveis tipadas
- Verificação: enviar notificação T-09 em ambiente de staging → e-mail recebido em ≤5min; bounce simulado → status BOUNCED + alerta Coordenador

**[BE-05] Implementar worker de WhatsApp via Meta Cloud API**
- Queue: `notifications.whatsapp`; provedor: Meta Cloud API (templates HSM pré-aprovados)
- Rate limit: 250 conversas/dia por WABA (WhatsApp Business Account)
- Payload: `WhatsAppPayload` (to `+55XXXXX`, template_name, language_code `pt_BR`, components)
- Fallback após 2 retries: publicar na queue `notifications.sms` com `fallback_from = 'whatsapp'`; atualizar log para `FALLBACK_SMS`
- Opt-out: verificar `notification_preferences` antes de enviar; notificações críticas (`case_lifecycle`, `financial`, `formalization`) não podem ser bloqueadas por opt-out
- Verificação: destinatário com `enabled = false` para `whatsapp/sla_alert` → mensagem não enviada; falha após 2 retries → SMS enfileirado com `fallback_from = 'whatsapp'`

**[BE-06] Implementar worker de SMS via Twilio**
- Queue: `notifications.sms`; provedor: Twilio
- Rate limit: 1 msg/segundo por número; respeitar rate limit via Redis semaphore
- Fallback após 2 retries: `UPDATE status = 'FAILED'` + alerta ao Coordenador
- Se `fallback_from = 'whatsapp'`: registrar no `notification_logs` original como `FALLBACK_SMS`
- Verificação: mensagem SMS entregue; falha total → status `FAILED` e alerta Coordenador

**[BE-07] Implementar worker in-app via Supabase Realtime**
- Queue: `notifications.in_app`; mecanismo: INSERT em `notification_logs` com `channel = 'in_app'`
- Supabase Realtime emite evento ao INSERT → frontend recebe via WebSocket e exibe no `NotificationBell` (C-13) sem polling
- `status = 'SENT'` imediatamente após INSERT (entrega garantida se conexão ativa; mensagens perdidas carregadas no próximo login via `GET /v1/notifications?status=UNREAD`)
- SLA: exibido em até 30 segundos após evento (RN-062)
- Verificação: trigger de evento → notificação aparece no painel em ≤30s; usuário offline → mensagens carregadas no próximo login

---

## FEATURE 3 — 22 Templates

### Backend

**[BE-08] Implementar os 22 templates de notificação conforme inventário D21 §3.1**
- Implementar EXATAMENTE os 22 templates com IDs e nomes canônicos:
  - `T-01 rs_caso_captado` → canais: E-mail + Painel; prioridade: High; variáveis: `case_id, analyst_name, created_at`
  - `T-02 rs_dossie_incompleto` → canais: E-mail + Painel + WhatsApp; prioridade: High; variáveis: `case_id, cedente_name, pending_docs[], deadline`
  - `T-03 rs_caso_bloqueado` → canais: E-mail + Painel + WhatsApp; prioridade: Critical; variáveis: `case_id, cedente_name, block_reason`
  - `T-04 rs_caso_qualificado` → canais: E-mail + Painel + WhatsApp; prioridade: High; variáveis: `case_id, cedente_name, scenario`
  - `T-05 rs_nova_proposta` → canais: E-mail + Painel; prioridade: High; variáveis: `case_id, analyst_name, proposed_value`
  - `T-06 rs_contraproposta` → canais: E-mail + Painel + WhatsApp; prioridade: High; variáveis: `case_id, counter_value, expires_at`
  - `T-07 rs_negociacao_escalada` → canais: E-mail + Painel; prioridade: High; variáveis: `case_id, analyst_name, escalation_reason`
  - `T-08 rs_aceite_confirmado` → canais: E-mail + Painel + WhatsApp; prioridade: High; variáveis: `case_id, accepted_value, cedente_name, cessionario_name`
  - `T-09 rs_instrucoes_deposito` → canais: E-mail + Painel + WhatsApp; prioridade: Critical; variáveis: `case_id, cessionario_name, escrow_amount, pix_key, deadline`
  - `T-10 rs_lembrete_deposito` → canais: E-mail + Painel + WhatsApp; prioridade: High; variáveis: `case_id, cessionario_name, escrow_amount, days_remaining`
  - `T-11 rs_deposito_confirmado` → canais: E-mail + Painel; prioridade: High; variáveis: `case_id, confirmed_amount, confirmed_at`
  - `T-12 rs_envelope_zapsign` → canal: E-mail via ZapSign (gerenciado pela ZapSign); prioridade: High
  - `T-13 rs_assinaturas_concluidas` → canais: E-mail + Painel; prioridade: High; variáveis: `case_id, envelope_type, signed_at`
  - `T-14 rs_fechamento_confirmado` → canais: E-mail + Painel + WhatsApp; prioridade: Critical; variáveis: `case_id, cedente_name, cessionario_name, net_value, closed_at`
  - `T-15 rs_periodo_reversao` → canais: E-mail + Painel + WhatsApp; prioridade: Critical; variáveis: `case_id, reversal_deadline, cedente_name, cessionario_name`
  - `T-16 rs_distribuicao_realizada` → canais: E-mail + Painel + WhatsApp; prioridade: Critical; variáveis: `case_id, cedente_amount, rs_fee, distributed_at`
  - `T-17 rs_caso_cancelado` → canais: E-mail + Painel + WhatsApp; prioridade: High; variáveis: `case_id, cedente_name, cancel_reason`
  - `T-18 rs_sla_proximo` → canal: Painel (somente); prioridade: Normal; desativável: Sim; variáveis: `case_id, current_stage, sla_deadline, sla_percentage_remaining`
  - `T-19 rs_sla_estourado` → canais: E-mail + Painel; prioridade: Critical; desativável: Não; variáveis: `case_id, current_stage, overdue_since, analyst_name`
  - `T-20 rs_escalonamento_sugerido` → canais: E-mail + Painel + WhatsApp; prioridade: High; variáveis: `case_id, current_scenario, suggested_scenario, reason`
  - `T-21 rs_reversao_iniciada` → canais: E-mail + Painel + WhatsApp; prioridade: Critical; variáveis: `case_id, reversal_initiator, reversal_reason, cedente_name, cessionario_name`
  - `T-22 rs_mediacao_iniciada` → canais: E-mail + Painel + WhatsApp; prioridade: Critical; variáveis: `case_id, mediator_name, cedente_name, cessionario_name`
- Verificação: para cada template, envio em staging com variáveis de teste retorna status SENT; T-12 delegado integralmente à ZapSign

**[BE-09] Implementar disparo automático de notificações nos domain services**
- `CasesService.updateStatus()`: após cada transição de status válida, emitir `NotificationService.emit()` com template correto conforme destinatário:
  - CAPTADO → analista atribuído + coordenador: `T-01`
  - BLOQUEADO → cedente: `T-03`
  - QUALIFICADO → cedente: `T-04`
  - CANCELADO → cedente + cessionário (se EM_NEGOCIACAO ou posterior): `T-17`
- `ProposalsService.createProposal()`: emitir `T-05` para analista + cedente (sem dados do cessionário — anonimato preservado)
- `ProposalsService.counterpropose()`: emitir `T-06` para parte destinatária
- `SlaMonitorService.checkSla()`: emitir `T-18` ao atingir ≤20% do SLA máximo; emitir `T-19` ao estourar
- `FormalizationService.confirmClose()`: emitir `T-14` para cedente + cessionário + gestor financeiro; emitir `T-15` para cedente + cessionário
- `EscrowService.distribute()`: emitir `T-16` para cedente + cessionário + gestor financeiro
- Verificação: transição CAPTADO → EM_TRIAGEM emite `T-01`; transição FECHAMENTO → POS_FECHAMENTO emite `T-15`; distribuição emite `T-16`

---

## FEATURE 4 — API de Preferências e Opt-Out

### Backend

**[BE-10] Implementar API de preferências de notificação**
- `GET /v1/notifications/preferences`: role qualquer autenticado; retorna todas as preferências do usuário autenticado
- `PATCH /v1/notifications/preferences`:
  - Body: `{ channel: string, category: string, enabled: boolean }`
  - **Categorias não desativáveis:** `case_lifecycle`, `financial`, `formalization` → retornar `422 BIZ_001` com message `"Notificação crítica não pode ser desativada."` (RN inegociável)
  - **Categorias desativáveis:** `sla_alert` (exceto `rs_sla_estourado`) e `operational`
  - `rs_sla_estourado` (T-19): mesma proteção crítica, independentemente de `sla_alert` ser parcialmente desativável
  - Ao atualizar: invalidar cache Redis de preferências do usuário (TTL 5min)
  - Retornar `200 OK` com preferência atualizada
- `PUT /v1/notifications/preferences/reset`: restaurar defaults do perfil (`enabled = true` para todas as categorias)
- Verificação: PATCH com `category = 'case_lifecycle'` retorna 422; PATCH com `category = 'sla_alert'` retorna 200; GET retorna todas as preferências corretamente

---

## FEATURE 5 — Centro de Notificações In-App

### Frontend

**[FE-01] Implementar `NotificationBell` (C-13) na Topbar**
- Ícone de sino na barra superior (Topbar C-11) com badge numérico de não lidas
- Badge: número de `notification_logs` WHERE `recipient_id = currentUser.id AND status != 'READ' AND channel = 'in_app' AND created_at > NOW() - INTERVAL '90 days'`
- Badge max: exibir "9+" se ≥ 10 não lidas
- Ao clicar no sino: painel dropdown renderiza lista de notificações em ordem cronológica reversa
- Cada item do dropdown: ícone do tipo de evento, texto resumido, data/hora relativa ("há 5 min"), indicador lida/não lida (ponto azul no não lida)
- Notificações não lidas: fundo levemente mais escuro + ponto azul na lateral (DEC-010)
- Botão "Marcar todas como lidas" no topo do painel → `PATCH /v1/notifications/mark-all-read`
- Cada notificação clicável → redireciona para a tela de contexto (ex: `T-09` abre `/formalizacao/:case_id`)
- Supabase Realtime subscription: `supabase.channel('notification_logs').on('INSERT', ...)` → ao receber INSERT com `recipient_id = currentUser.id AND channel = 'in_app'` → adicionar ao topo da lista + incrementar badge + exibir toast breve
- Verificação: emitir notificação in-app → aparece em ≤30s no painel; clicar notifica → redireciona para contexto; "Marcar todas como lidas" zera badge

**[FE-02] Implementar endpoint `GET /v1/notifications` e log de notificações no backend**
- `GET /v1/notifications`: role qualquer autenticado
- Query params: `page` (default 1), `per_page` (default 20), `status` (filtro), `channel` (filtro)
- Retorna: `{ data: [NotificationLog], meta: {} }`
- `GET /v1/notifications/unread-count`: retorna `{ data: { count: number } }` para o badge
- `PATCH /v1/notifications/:id/read`: setar `read_at = NOW()`, `status = 'READ'`; verificar `recipient_id = currentUser.id`
- `PATCH /v1/notifications/mark-all-read`: setar `read_at = NOW()`, `status = 'READ'` para todas WHERE `recipient_id = currentUser.id AND status != 'READ' AND channel = 'in_app'`
- Verificação: GET retorna apenas notificações do usuário autenticado; PATCH /read de notificação de outro usuário retorna 403

---

## FEATURE 6 — Alerta de Falha de Entrega (RN-064)

### Backend

**[BE-11] Implementar alerta de falha de entrega ao Coordenador (RN-064)**
- Ao detectar `DELIVERY_FAILED` (após retries esgotados em qualquer canal):
  1. Emitir notificação in-app ao Coordenador: badge "Falhou" na notificação específica no log
  2. Registrar falha em `notification_logs` com `status = 'FAILED', failure_reason = motivo`
  3. Gerar entrada em `notification_logs` tipo `DELIVERY_ALERT` para o Coordenador com contexto: usuário destinatário, canal, evento, motivo da falha
- Coordenador pode: (a) ver motivo da falha diretamente no log, (b) clicar "Corrigir E-mail" que abre perfil do usuário (se falha por e-mail inválido/bounce), (c) clicar "Reenviar" que recoloca mensagem na fila com `retry_count = 0`
- `POST /v1/notifications/:id/resend`: role `COORDENADOR`; recolocar notificação falhada na fila RabbitMQ; retornar `202 Accepted`
- Bounce webhook Resend (`POST /v1/webhooks/resend`): validar HMAC-SHA256 do webhook; processar bounce → `status = 'BOUNCED'` + acionar alerta ao Coordenador
- Verificação: envio com e-mail inválido → bounce recebido → status BOUNCED → alerta ao Coordenador; "Reenviar" → status muda para QUEUED

**[FE-03] Implementar log de notificações no módulo Admin (visível para Coordenador)**
- Tela de log acessível via `/notificacoes/log` (sub-item no menu — visível para COORDENADOR e MASTER)
- Tabela: evento, destinatário (mascarado), canal, data/hora, status (badge colorido: verde SENT, vermelho FAILED/BOUNCED, cinza PENDING)
- Badge "Falhou" em vermelho para status FAILED ou BOUNCED
- Ação "Corrigir E-mail": visível quando `failure_reason` contém bounce/invalid → abre T-078 ou T-080 do destinatário
- Ação "Reenviar": visível para status FAILED/BOUNCED → dispara `POST /v1/notifications/:id/resend`
- Filtros: status, canal, data range, busca por evento
- Verificação: filtro por `status=FAILED` mostra apenas falhas; "Reenviar" desabilitado para status SENT

---

## FEATURE 7 — Métricas e Monitoramento

### Backend

**[BE-12] Implementar coleta de métricas de notificações (RN-062)**
- Cron `*/5 * * * *` (a cada 5 minutos): calcular métricas da janela rolling 24h:
  - Taxa de falha por canal: `COUNT(status='FAILED') / COUNT(*) * 100` por canal
  - Contagem DLQ: `COUNT(*)` na queue `rs.notifications.dlq`
  - Mediana SLA e-mail: percentil 50 de `sent_at - created_at` para canal `email` dos últimos 60 min
  - Taxa bounce e-mail (7 dias): `COUNT(status='BOUNCED') / COUNT(*) WHERE channel='email'`
  - Taxa fallback WhatsApp→SMS (24h): `COUNT(fallback_channel='sms') / COUNT(*) WHERE channel='whatsapp'`
- Alertas automáticos:
  - Taxa de falha > 5% qualquer canal → log nível ERROR + alerta Slack `#alertas-producao`
  - Mediana SLA e-mail > 5min → alerta Slack
  - Contagem DLQ > 0 → alerta imediato Slack
  - Taxa bounce e-mail > 2% (7d) → alerta ao Master via in-app
  - Taxa fallback WhatsApp→SMS > 10% (24h) → alerta de degradação WhatsApp
- Job cron `0 0 1 * *` (1º de cada mês): `DELETE FROM notification_logs WHERE created_at < NOW() - INTERVAL '90 days'`
- Verificação: acumular 10+ falhas de e-mail em 5 min → métrica > 5% → log ERROR; DLQ com 1 mensagem → alerta imediato

---

## WIRING

**[WIRE-01] Integrar `NotificationsModule` no NestJS e conectar aos domain services**
- `NotificationsModule` exporta `NotificationService`; importado em `CasesModule`, `ProposalsModule`, `FormalizationModule`, `EscrowModule`, `SlaMonitorModule`, `UsersModule`
- `NotificationService` injeta: `PrismaService` (INSERT/UPDATE `notification_logs`), `AmqpConnection` (RabbitMQ publish), `RedisService` (cache preferências)
- Workers registrados como `@RabbitSubscribe` em `EmailWorker`, `WhatsAppWorker`, `SmsWorker`, `InAppWorker`
- Webhook controller: `WebhooksController.handleResendBounce()` e `WebhooksController.handleMetaDelivery()`
- `ScheduleModule.forRoot()` presente para jobs de métricas e retenção
- Verificação: `AppModule` importa `NotificationsModule`; `CasesService` injeta `NotificationService`; teste de integração confirma emit() publica na fila

**[WIRE-02] Integrar frontend com Supabase Realtime e React Query para notificações**
- `NotificationsProvider` (React Context): subscribe ao Supabase Realtime channel `notification_logs` na montagem; ao desmontar, unsubscribe
- `useNotifications()` hook: `useQuery` para `GET /v1/notifications` com polling fallback (caso Realtime offline) a cada 60s
- `useUnreadCount()` hook: `useQuery` para `GET /v1/notifications/unread-count`; invalidado ao receber INSERT via Realtime
- `useMarkAsRead(id)` e `useMarkAllRead()` mutations: após sucesso, invalidar `useUnreadCount()`
- `NotificationBell` (C-13) consome `useUnreadCount()` e `useNotifications()`
- Verificação: desconectar WebSocket → fallback polling ativa; reconectar → Realtime retoma; marcar como lida → badge decrementa imediatamente

---

## TESTES

**[TEST-01] Testes unitários — `NotificationService`**
- `NotificationService.emit()`: INSERT em `notification_logs` com status PENDING → QUEUED; NUNCA envio síncrono (assert: provedor externo não chamado na mesma thread)
- `NotificationService.emit()` com preferência `enabled = false` para canal/categoria não crítica → publicação nesse canal não enfileirada
- `NotificationService.emit()` com categoria crítica (`case_lifecycle`) e usuário com `enabled = false` → publicação ocorre normalmente (opt-out ignorado)
- `PATCH /v1/notifications/preferences` com `category = 'case_lifecycle'` → lança `422 BIZ_001`
- `PATCH /v1/notifications/preferences` com `category = 'sla_alert'` → retorna 200

**[TEST-02] Testes de integração — workers por canal**
- Worker e-mail: sucesso → `notification_logs.status = 'SENT'`; falha 3× → `status = 'FAILED'` + alerta Coordenador em `notification_logs`
- Worker WhatsApp: falha 2× → mensagem publicada na queue `notifications.sms` com `fallback_from = 'whatsapp'`; `notification_logs.status = 'FALLBACK_SMS'`
- Worker SMS: falha 2× → `status = 'FAILED'` + alerta Coordenador
- Worker in-app: INSERT em `notification_logs` → Supabase Realtime emite evento (mock do Supabase)
- Bounce webhook Resend: payload válido com HMAC correto → `status = 'BOUNCED'` + alerta gerado; HMAC inválido → 401

**[TEST-03] Testes de componente — Frontend**
- `NotificationBell`: badge exibe contagem correta de não lidas; ao clicar, dropdown renderiza lista; "Marcar todas como lidas" envia mutation e badge zera
- `NotificationBell` Realtime: mock de INSERT em Supabase → badge incrementa; item aparece no topo da lista
- Log de notificações: filtro por FAILED retorna apenas falhas; ação "Reenviar" chama mutation; badge "Falhou" vermelho presente

**[TEST-04] Testes E2E — fluxo de notificação ponta a ponta**
- Playwright: ação de transição de status CAPTADO → EM_TRIAGEM → verificar `notification_logs` tem registro com `template_id = 'rs_caso_captado'` e `status = 'QUEUED'` (ou SENT após worker processar)
- Playwright: usuário com preferência `sla_alert/email/enabled=false` → emitir T-18 → verificar que `notification_logs` não tem registro de canal `email` para esse usuário, mas canal `in_app` existe
- Playwright: login como Coordenador → abrir log de notificações → verificar que notificações com FAILED exibem badge "Falhou"

---

## AUTO-VERIFICAÇÃO (12 CHECKS)

- [x] **Check #1 — 22 templates cobertos:** todos os 22 templates (T-01 a T-22) implementados com IDs canônicos exatos, canais corretos, prioridades e variáveis conforme D21 §3.1
- [x] **Check #2 — Nomes canônicos:** `notification_logs`, `notification_preferences`, `rs.notifications`, `rs.notifications.dlq`, `PENDING | QUEUED | SENT | DELIVERED | READ | FAILED | FALLBACK_SMS | BOUNCED`, `case_lifecycle | financial | formalization | sla_alert | operational` — zero sinônimos
- [x] **Check #3 — SLA de entrega (RN-062):** Painel ≤30s via Supabase Realtime; E-mail ≤5min via Resend; verificado em testes E2E
- [x] **Check #4 — Async obrigatório:** ZERO envio síncrono; todo envio passa pela fila RabbitMQ; unit test verifica isso
- [x] **Check #5 — Retry policy por canal:** email 3× (30s/2min/10min), whatsapp 2× (1min/5min), sms 2× (1min/5min), in_app 1× (5s) — exatamente esses valores
- [x] **Check #6 — Fallback WhatsApp→SMS:** após 2 retries WhatsApp → SMS enfileirado; após 2 retries SMS → FAILED + alerta Coordenador
- [x] **Check #7 — Categorias críticas não desativáveis:** `case_lifecycle`, `financial`, `formalization` + `rs_sla_estourado` → 422 BIZ_001 ao tentar desativar
- [x] **Check #8 — Alerta de falha ao Coordenador (RN-064):** bounce, FAILED e DLQ geram alerta ao Coordenador; badge "Falhou" no log; botão "Reenviar" ativo
- [x] **Check #9 — Retenção 90 dias:** job cron mensal apaga `notification_logs` com `created_at < NOW() - 90 days`
- [x] **Check #10 — Métricas de monitoramento:** 5 métricas calculadas a cada 5min; 5 limiares de alerta configurados; Slack `#alertas-producao` para degradação
- [x] **Check #11 — Anti-scaffold R10:** todos os itens têm sub-itens com lógica concreta, valores numéricos exatos (30s, 5min, 90 dias), condições de erro, RBAC e verificação
- [x] **Check #12 — Cobertura REQs S8a:** REQ-130 (RN-062 SLA painel ≤30s + e-mail ≤5min) ✅ REQ-131 (RN-063 tabela 22 eventos completa com canais) ✅ REQ-132 (RN-064 falhas → alerta Coordenador, badge "Falhou", reenvio manual, log completo) ✅

---

*Sprint S8a gerada por Pipeline ShiftLabs v2.3 — 2026-03-24*
