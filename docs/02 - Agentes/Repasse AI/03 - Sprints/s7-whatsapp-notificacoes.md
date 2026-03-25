# S7 — WhatsApp e Notificações

<!-- Atualizado em 2026-03-24 pela A03 — 5 correções aplicadas (FINDING-001, FINDING-018, FINDING-019, FINDING-024, FINDING-025, FINDING-026) -->

| Campo              | Valor                                                                                                                                                                                                                                                                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S7                                                                                                                                                                                                                                                                                                                                          |
| **Nome**           | WhatsApp e Notificações (Fase 2)                                                                                                                                                                                                                                                                                                            |
| **Template**       | B — Módulo Fullstack                                                                                                                                                                                                                                                                                                                        |
| **Docs Fonte**     | 01.5-RN-Integrações, 14-EspecTécnicas, 16-API, 17-IntegraçõesExternas                                                                                                                                                                                                                                                                       |
| **REQs cobertos**  | REQ-001, REQ-002, REQ-003, REQ-104, REQ-105, REQ-106, REQ-107, REQ-108, REQ-109, REQ-110, REQ-136, REQ-137, REQ-138, REQ-139, REQ-140, REQ-141, REQ-142, REQ-143, REQ-144, REQ-145, REQ-146, REQ-147, REQ-148, REQ-149, REQ-150, REQ-151, REQ-152, REQ-153, REQ-154, REQ-155, REQ-156, REQ-157, REQ-158, REQ-164, REQ-165, REQ-166, REQ-167 |
| **Total de itens** | 32                                                                                                                                                                                                                                                                                                                                          |
| **Status**         | ✅ Concluido                                                                                                                                                                                                                                                                                                                                |

---

## 🎯 Objetivo

Implementar o canal WhatsApp (Fase 2) via EvolutionAPI: fluxo OTP em 2 etapas (SMS + WhatsApp), vinculação de número, re-verificação a cada 30 dias, comando PARAR para desvincular, rate limit de 20 msg/h, e 3 tipos de notificações proativas.

---

## 🏗️ FEATURE 1: OTP e Vinculação WhatsApp

### 1. `WhatsappService.initiateBinding` — iniciar vinculação (Etapa 1 — SMS)

- [x] Criar `WhatsappService` com método `initiateBinding(user: AuthenticatedUser, phoneNumber: string): OtpInitResult`:
  - Validar formato `phoneNumber`: E.164 (+55XXXXXXXXXXX) → `BadRequestException` se inválido
  - Verificar que número não está vinculado a outro `cessionario_id` ativo → `ConflictException` se já vinculado
  - Gerar OTP: 6 dígitos numéricos gerados via `crypto.randomInt(100000, 999999)`
  - Armazenar hash do OTP: `bcrypt.hash(otp, 10)` → INSERT em `otp_attempts` com `channel: 'sms'`, `expires_at: NOW() + INTERVAL '15 minutes'`, `status: 'pending'` — [CORRIGIDO: FINDING-001/019 — TTL corrigido de 10min para 15min]
  - Limitar OTP a 3 tentativas por hora via Redis: `INCR ai:otp:attempts:{phoneNumber}` com TTL 3600s; se `INCR > 3` → `429 TooManyRequests` com `Retry-After: seconds_until_window_reset`
  - Bloquear após 5 falhas consecutivas: `SET ai:otp:blocked:{phoneNumber} 1 EX 1800 NX` (bloqueio 30 minutos); verificar bloqueio antes de processar OTP
  - Enviar OTP via SMS (integração externa — mockada com `FEATURE_WHATSAPP_MOCK=true`)
  - INSERT em `whatsapp_bindings` com `status: 'pending'`, `cessionario_id`, `phone_number`
  - Retornar `{ bindingId, phoneNumber, channel: 'sms', expiresIn: 600 }` (expiresIn em segundos)
- [x] **NUNCA retornar o OTP em texto claro** na resposta da API
- [x] Teste unitário: `phoneNumber='+5511999999999'` → OTP gerado + hash armazenado + SMS enviado (mock); número já vinculado → `ConflictException`; formato inválido `'11999999999'` → `BadRequestException`

### 2. `WhatsappService.verifyOtp` — verificar OTP (Etapa 1)

- [x] Implementar `verifyOtp(user: AuthenticatedUser, bindingId: string, otp: string): OtpVerifyResult`:
  - Buscar OTP não expirado: SELECT em `otp_attempts` WHERE `whatsapp_binding_id = bindingId AND status = 'pending' AND expires_at > NOW()`
  - Se não encontrado → `NotFoundException("OTP inválido ou expirado")`
  - Comparar OTP informado com hash: `bcrypt.compare(otp, storedHash)` → se não confere → UPDATE `otp_attempts SET status = 'failed'` + `BadRequestException("OTP incorreto")`
  - Se confere → UPDATE `otp_attempts SET status = 'verified', verified_at = now()`
  - Enviar OTP de confirmação via WhatsApp (Etapa 2): novo INSERT em `otp_attempts` com `channel: 'whatsapp'`, `expires_at: NOW() + INTERVAL '10 minutes'`
  - Retornar `{ bindingId, status: 'awaiting_whatsapp_confirmation', nextStep: 'whatsapp_otp' }`
- [x] Teste unitário: OTP correto → `status: 'awaiting_whatsapp_confirmation'`; OTP incorreto → `BadRequestException`; OTP expirado (`expires_at` no passado) → `NotFoundException`

### 3. `WhatsappService.confirmWhatsappOtp` — confirmar OTP WhatsApp (Etapa 2)

- [x] Implementar `confirmWhatsappOtp(user: AuthenticatedUser, bindingId: string, otp: string): BindingResult`:
  - Buscar OTP WhatsApp não expirado em `otp_attempts` WHERE `channel = 'whatsapp' AND status = 'pending' AND expires_at > NOW()`
  - Comparar OTP com hash → se inválido → `BadRequestException`
  - UPDATE `otp_attempts SET status = 'verified'`
  - UPDATE `whatsapp_bindings SET status = 'active', verified_at = now(), last_verified_at = now()`
  - INSERT consentimento LGPD: `LgpdService.recordConsent(userId, 'whatsapp_contact', true, ipAddress, userAgent)`
  - Retornar `{ bindingId, phoneNumber, status: 'active', verifiedAt }`
- [x] Máquina de estado `whatsapp_binding_status`: `pending → active` (apenas via confirmação OTP)
- [x] Teste unitário: OTP WhatsApp correto → binding `status: 'active'`; OTP inválido → `BadRequestException`; binding já `active` → verificar idempotência (não gerar novo binding duplicado)

### 4. Job `WhatsappReverificationJob` — re-verificação a cada 30 dias

- [x] Criar `WhatsappReverificationJob` como cron job: executar diariamente às 02:00 UTC (`@Cron('0 2 * * *')`)
- [x] SELECT em `whatsapp_bindings` WHERE `status = 'active' AND last_verified_at < NOW() - INTERVAL '30 days'`
- [x] Para cada binding vencido: UPDATE `status = 'inactive'` + publicar evento `ai.notifications` com `type: 'WHATSAPP_REVERIFICATION_REQUIRED'`
- [x] Logar resultado: `{ processed: N, deactivated: N, errors: N }`
- [x] Teste unitário: binding com `last_verified_at` há 31 dias → `status: 'inactive'`; binding com 29 dias → não alterado; falha em 1 binding → logar erro e continuar os demais

### 5. Endpoint `POST /whatsapp/binding` — iniciar vinculação

- [x] Criar `WhatsappController` com endpoint `POST /whatsapp/binding` (`@Roles('CESSIONARIO')`)
- [x] DTO `InitiateBindingDto`: `{ phoneNumber: string }` — validar formato E.164
- [x] Resposta HTTP: `201 Created` com `{ bindingId, phoneNumber, channel: 'sms', expiresIn: 600 }`
- [x] Teste de integração: POST com número válido → `201`; número já vinculado → `409`; token ADMIN → `403` (apenas CESSIONARIO pode vincular)

### 6. Endpoint `POST /whatsapp/binding/verify` — verificar OTP SMS

- [x] Criar endpoint `POST /whatsapp/binding/verify` (`@Roles('CESSIONARIO')`)
- [x] DTO `VerifyOtpDto`: `{ bindingId: string (uuid), otp: string (6 dígitos) }`
- [x] Resposta HTTP: `200 OK` com `{ bindingId, status: 'awaiting_whatsapp_confirmation' }`
- [x] Teste de integração: OTP correto → `200`; OTP errado → `400`; OTP expirado → `404`; `bindingId` inexistente → `404`

### 7. Endpoint `POST /whatsapp/binding/confirm` — confirmar OTP WhatsApp

- [x] Criar endpoint `POST /whatsapp/binding/confirm` (`@Roles('CESSIONARIO')`)
- [x] DTO `ConfirmOtpDto`: `{ bindingId: string (uuid), otp: string (6 dígitos) }`
- [x] Resposta HTTP: `200 OK` com `{ bindingId, phoneNumber, status: 'active', verifiedAt }`
- [x] Teste de integração: OTP correto → `200 { status: 'active' }`; binding já `active` → `409`

### 8. Endpoint `DELETE /whatsapp/binding` — desvincular (comando PARAR)

- [x] Criar endpoint `DELETE /whatsapp/bindings/:bindingId` (`@Roles('CESSIONARIO')`) — [CORRIGIDO: FINDING-018 — plural + ID no path]
- [x] `WhatsappService.unbindWhatsapp(user)`:
  - SELECT binding ativo do usuário → se não existe → `404`
  - Soft delete: UPDATE `whatsapp_bindings SET deleted_at = now(), status = 'inactive'`
  - INSERT `lgpd_consents` com `consentType: 'whatsapp_contact', granted: false`
  - Publicar evento `ai.notifications` com `type: 'WHATSAPP_UNLINKED'`
- [x] Máquina de estado: `active → inactive` via DELETE; `inactive` → DELETE → `404` (já desvinculado)
- [x] Teste de integração: DELETE com binding ativo → `204`; DELETE sem binding → `404`; DELETE binding já inativo → `404`

### 9. Webhook EvolutionAPI — `POST /whatsapp/webhook`

- [x] Criar endpoint `POST /whatsapp/webhook` (`@Public()` — sem JWT)
- [x] Validar assinatura HMAC-SHA256 do header `X-Evolution-Signature` com `EVOLUTION_API_KEY` → se inválido → `401 Unauthorized`
- [x] Processar evento `message.received`: extrair `from` (phoneNumber), `body` (mensagem), `timestamp`
- [x] Buscar binding ativo por `phone_number = from` → se não encontrado → ignorar (não processar mensagens de números não vinculados)
- [x] Se mensagem for `"PARAR"` (case-insensitive) → chamar `WhatsappService.unbindWhatsapp` automaticamente
- [x] Caso contrário → encaminhar para fluxo do agente via `AiService.sendMessage` com `channel: 'whatsapp'` + aplicar rate limit 20 msg/h
- [x] Resposta HTTP: `200 OK` (sempre — EvolutionAPI aguarda 200 para confirmar entrega)
- [x] Teste de integração: webhook válido com mensagem normal → `200` + mensagem processada; mensagem `"parar"` → desvincular + `200`; assinatura inválida → `401`; número não vinculado → `200` (ignorar silenciosamente)

---

## 🏗️ FEATURE 2: Notificações Proativas (3 tipos)

### 10. `NotificacaoService.sendProactiveNotification` — enviar notificação

- [x] Criar `NotificacaoService` com método `sendProactiveNotification(cessionarioId: string, type: NotificationType, payload: NotificationPayload)`:
  - Verificar `notification_preferences` do usuário → se `enabled = false` para o tipo → não enviar (retornar sem erro)
  - **Tipo 1 — `new_opportunity`:** nova oportunidade disponível no mercado — enviar via canal configurado na preferência
  - **Tipo 2 — `deal_status_change`:** mudança de status em deal em andamento
  - **Tipo 3 — `market_alert`:** alerta de mercado (variação relevante de preço/taxa)
  - INSERT em `notification_events`: `{ cessionario_id, type, title, body, metadata, sent_at: now() }`
  - Se `channel = 'webchat'`: publicar evento Supabase Realtime (condicional — verificar se habilitado)
  - Se `channel = 'whatsapp'`: publicar mensagem via EvolutionAPI (somente se binding `status = 'active'`)
  - Se `channel = 'both'`: enviar por ambos os canais
- [x] Teste unitário: preferência `new_opportunities = false` → notificação não enviada (retornar sem erro); `channel = 'whatsapp'` com binding inativo → não enviar WhatsApp mas INSERT `notification_events` com `sent_at = null`

### 11. Endpoint `GET /notification/preferences` — listar preferências

- [x] Criar `NotificationController` com endpoint `GET /notification/preferences` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] SELECT em `notification_preferences` WHERE `cessionario_id = user.userId`
- [x] Se não existe registro → retornar defaults: `{ new_opportunities: true, deal_status_changes: true, market_alerts: true, channel: 'webchat' }`
- [x] Resposta HTTP: `200 OK` com `NotificationPreferencesDto`
- [x] Teste de integração: sem registro → retornar defaults; com registro → retornar valores do banco

### 12. Endpoint `PUT /notification/preferences` — atualizar preferências

- [x] Criar endpoint `PUT /notification/preferences` (`@Roles('CESSIONARIO')`)
- [x] DTO `UpdateNotificationPreferencesDto`: `{ new_opportunities?: boolean, deal_status_changes?: boolean, market_alerts?: boolean, channel?: NotificationChannel }`
- [x] UPSERT em `notification_preferences` (INSERT ou UPDATE se já existe)
- [x] Validar `channel` pertence ao enum `NotificationChannel`: `webchat`, `whatsapp`, `both`
- [x] Se `channel = 'whatsapp'` ou `'both'`: verificar que existe `whatsapp_bindings` ativo → se não existe → `UnprocessableEntityException("Vincule seu WhatsApp antes de ativar notificações por este canal")`
- [x] Resposta HTTP: `200 OK`
- [x] Teste de integração: atualizar `channel: 'whatsapp'` sem binding ativo → `422`; com binding ativo → `200`; `channel: 'webchat'` sempre válido → `200`

### 13. Endpoint `GET /notification/events` — listar notificações recebidas

- [x] Criar endpoint `GET /notification/events` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] Query params: `type?: NotificationType`, `read?: boolean`, `page (default 1)`, `per_page (default 20)`
- [x] CESSIONARIO → `WHERE cessionario_id = user.userId`; ADMIN → todos
- [x] Retornar `{ data: NotificationEventDto[], meta: { page, per_page, total, total_pages } }`
- [x] `NotificationEventDto`: `{ id, type, title, body, metadata, sentAt, readAt, createdAt }`
- [x] Teste de integração: GET sem filtros → todas as notificações do usuário; `read=false` → não lidas (readAt IS NULL)

### 14. Endpoint `PATCH /notification/events/:id/read` — marcar como lida

- [x] Criar endpoint `PATCH /notification/events/:id/read` (`@Roles('CESSIONARIO')`)
- [x] UPDATE `notification_events SET read_at = now()` WHERE `id = id AND cessionario_id = user.userId`
- [x] Se não encontrado ou pertence a outro usuário → `404`
- [x] Resposta HTTP: `200 OK` com evento atualizado
- [x] Teste de integração: marcar como lida → `200` com `readAt` preenchido; evento de outro usuário → `404`; evento inexistente → `404`

---

## ✅ TESTES — WhatsApp e Notificações

### 15. Endpoint `GET /whatsapp/bindings` — listar binding do usuário [CORRIGIDO: FINDING-025]

- [x] Criar endpoint `GET /whatsapp/bindings` (`@Roles('CESSIONARIO')`)
- [x] SELECT em `whatsapp_bindings` WHERE `cessionario_id = user.userId AND deleted_at IS NULL ORDER BY created_at DESC LIMIT 1`
- [x] Retornar `BindingResponseDto`: `{ id, phoneNumber, status, verifiedAt, lastVerifiedAt }` — ou `404 Not Found` se sem binding ativo
- [x] Teste: sem binding → `404`; binding ativo → `200` com dados; ADMIN → `403` (apenas CESSIONARIO pode ver seu próprio binding)

### 16. Suite de testes

- [x] Criar `test/unit/whatsapp/whatsapp.service.spec.ts` — cobrir OTP (hash, expiração, comparação), re-verificação, PARAR
- [x] Criar `test/unit/whatsapp/whatsapp-reverification.job.spec.ts` — cobrir cron, 30 dias, falha parcial
- [x] Criar `test/unit/notification/notificacao.service.spec.ts` — cobrir 3 tipos, preferências, canal
- [x] Criar `test/integration/whatsapp/whatsapp.controller.spec.ts` — cobrir todos os 5 endpoints + webhook
- [x] Criar `test/integration/notification/notification.controller.spec.ts` — cobrir todos os 4 endpoints
- [x] Criar `test/e2e/whatsapp/whatsapp-flow.e2e-spec.ts` — E2E: vincular → verificar OTP SMS → confirmar OTP WhatsApp → enviar mensagem → PARAR (3º fluxo E2E obrigatório do Doc 27)
- [x] `FEATURE_WHATSAPP_MOCK=true` em testes — não fazer chamadas reais à EvolutionAPI
- [x] Cobertura mínima módulo `whatsapp/`: 80%; módulo `notification/`: 75% (conforme Doc 27)

---

## 🔀 Cross-Módulo

- [x] **← S2 (Auth):** rate limit 20 msg/h WhatsApp via `RateLimitGuard` (herdado) — verificar configuração do canal via header `X-Channel: whatsapp`
- [x] **← S6 (Admin):** `notification_events` INSERTados pelos alertas de Admin (item 6 de S6) são listados no `GET /notification/events` desta sprint
- [x] **← S3b (Agente):** webhook EvolutionAPI encaminha mensagens para `AiService.sendMessage` — confirmar que `AiModule` é acessível pelo `WhatsappModule`

---

## ✔️ Auto-Verificação S7 (12 Checks)

| #   | Check                                                                                                                                                                                                                        | Status |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                                                                 | ✅     |
| 2   | Nomes exatos: `WhatsappService`, `NotificacaoService`, `WhatsappReverificationJob`, `whatsapp_bindings`, `otp_attempts`, `notification_preferences`, `notification_events`, `FEATURE_WHATSAPP_MOCK`, `X-Evolution-Signature` | ✅     |
| 3   | Valores numéricos: OTP 6 dígitos, TTL OTP 10min, re-verificação 30 dias, rate limit 20 msg/h WhatsApp, cron 02:00 UTC, 3 tipos de notificação                                                                                | ✅     |
| 4   | Glossário consultado — OTP, LGPD, Cessionário, Takeover                                                                                                                                                                      | ✅     |
| 5   | Anti-Scaffold R10: OTP com hash bcrypt real; webhook com validação HMAC-SHA256; re-verificação com cron real                                                                                                                 | ✅     |
| 6   | Máquina de estado `whatsapp_binding_status`: `pending → active` (via OTP); `active → inactive` (via PARAR ou re-verificação); `inactive` → não retorna em queries ativas                                                     | ✅     |
| 7   | OTP **nunca** retornado em texto claro na API; hash bcrypt armazenado                                                                                                                                                        | ✅     |
| 8   | Sem conflitos não marcados                                                                                                                                                                                                   | ✅     |
| 9   | Consentimento LGPD `whatsapp_contact` inserido em vinculação (`granted: true`) e desvinculação (`granted: false`)                                                                                                            | ✅     |
| 10  | Template B: organizado por FEATURE (OTP/Vinculação, Notificações Proativas)                                                                                                                                                  | ✅     |
| 11  | `FEATURE_WHATSAPP_MOCK=true` em dev/test — sem chamadas reais à EvolutionAPI                                                                                                                                                 | ✅     |
| 12  | REQs cobertos: REQ-001 a REQ-003, REQ-104 a REQ-110, REQ-136 a REQ-158, REQ-164 a REQ-167 — todos com ≥1 item                                                                                                                | ✅     |
