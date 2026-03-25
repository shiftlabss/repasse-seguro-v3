# S7 — Escrow e Assinatura

| Campo                | Valor                                                                                                                                                                                                                                                                                                                                                                                                                               |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S7                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Nome**             | Escrow e Assinatura                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Tipo**             | Módulo Fullstack (Template B)                                                                                                                                                                                                                                                                                                                                                                                                       |
| **Template**         | B (Módulo Fullstack: por Feature, vertical slice Banco→Backend→Frontend→Wiring→Testes)                                                                                                                                                                                                                                                                                                                                              |
| **Docs Consultados** | D01 (RN-DCE-018, RN-DCE-019), D05.4 (RF-DCE-022 a RF-DCE-025), D12, D13, D16 (domínio Escrow), D17 (ZapSign §2.4)                                                                                                                                                                                                                                                                                                                   |
| **Cross-cutting**    | D10 (Glossário)                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **REQs cobertos**    | REQ-076, REQ-077, REQ-078, REQ-079, REQ-080, REQ-081, REQ-082, REQ-083, REQ-084, REQ-085, REQ-086, REQ-087, REQ-088, REQ-089                                                                                                                                                                                                                                                                                                        |
| **Objetivo**         | EscrowTransaction CRUD, máquina de estados (4 estados), prazo de depósito 10 dias úteis, extensão +5 dias úteis (aprovação/rejeição pelo Cedente, silêncio = aprovação automática em 24h), reversão em 15 dias corridos, webhook ZapSign com validação HMAC-SHA256, régua de notificações D+0/D+2/D+4/D+5, `GET /proposals/:id/escrow`, `POST /escrow/extension/approve`, `POST /escrow/extension/reject`, `POST /webhooks/zapsign` |

---

## Critério de Conclusão da S7

Ao final desta sprint:

- `GET /proposals/:id/escrow` retorna status com `deposit_deadline`, `extension_deadline`, `extension_requested`, `extension_approved`
- `POST /escrow/extension/approve` com `{ "confirmation": true }` → `extension_deadline = deposit_deadline + 5 dias úteis`
- `POST /escrow/extension/approve` sem extensão solicitada → 409 `DCE-ESCR-4090_001`
- `POST /escrow/extension/approve` com extensão já respondida → 409 `DCE-ESCR-4090_002`
- `POST /webhooks/zapsign` valida HMAC-SHA256 com `ZAPSIGN_WEBHOOK_SECRET` antes de processar — falha de validação → 401
- Webhook `document.signed` transiciona escrow para `RELEASED` e oportunidade para `CLOSED_SOLD`
- Job de auto-aprovação de extensão: se Cedente não responde em 24h → `extension_approved = true` automaticamente
- Régua ZapSign: D+0, D+2, D+4, D+5 — notificações disparadas no prazo exato
- Reversão de Escrow só possível dentro de 15 dias corridos após `deposited_at`

---

## ⚙️ FEATURE 1 — Ciclo de Vida do Escrow

### Banco

- [x] **Verificar que migration da tabela `escrow_transactions`** já existe no S1 com todos os campos: `id UUID PK`, `proposal_id UUID FK → proposals.id UNIQUE`, `amount DECIMAL(12,2) NOT NULL`, `status EscrowStatus NOT NULL DEFAULT 'AWAITING_DEPOSIT'`, `deposit_deadline TIMESTAMPTZ NOT NULL`, `extension_deadline TIMESTAMPTZ`, `deposited_at TIMESTAMPTZ`, `released_at TIMESTAMPTZ`, `reversed_at TIMESTAMPTZ`, `extension_requested BOOLEAN NOT NULL DEFAULT false`, `extension_approved BOOLEAN`, `extension_responded_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ`
  - Validação: `\d escrow_transactions` mostra todos os campos; `proposal_id` é UNIQUE (1 escrow por proposta); `amount` é `DECIMAL(12,2)` — sem Float; `extension_requested DEFAULT false`; `extension_approved` pode ser NULL (não respondido)
- [x] **Verificar enum `EscrowStatus`** já definido no S1 com exatamente os 4 valores: `AWAITING_DEPOSIT`, `DEPOSITED`, `RELEASED`, `REVERSED`
  - Validação: `SELECT enum_range(NULL::EscrowStatus)` retorna 4 valores exatos conforme D16; nenhum valor extra; Prisma schema alinhado
- [x] **Verificar RLS em `escrow_transactions`** via JOIN com proposals e opportunities: `USING (proposal_id IN (SELECT p.id FROM proposals p JOIN opportunities o ON p.opportunity_id = o.id WHERE o.cedente_id = current_setting('app.current_cedente_id')::uuid))`
  - Validação: Cedente A não acessa escrow da proposta da oportunidade do Cedente B; query cruzada retorna zero linhas

### Backend — EscrowService

- [x] **Implementar `EscrowService.findByProposal(proposalId: string, cedenteId: string)`** em `src/modules/escrow/escrow.service.ts`: (1) verifica que `proposal.opportunity.cedente_id == cedenteId`; (2) busca `escrow_transaction WHERE proposal_id = proposalId`; (3) lança `DaniNotFoundError('DCE-ESCR-4040_001', 'Escrow não encontrado para esta proposta.')` se não encontrado; (4) retorna escrow com todos os campos exceto dados PII internos
  - Validação: proposta sem escrow (status RECEIVED) → 404; proposta de outro Cedente → 403; proposta aceita → 200 com todos os campos de deadline

- [x] **Implementar `EscrowService.approveExtension(proposalId: string, cedenteId: string, dto: ApproveExtensionDto)`**: (1) verifica ownership via proposta; (2) busca escrow; (3) verifica `escrow.extension_requested == true` → senão lança `DaniConflictError('DCE-ESCR-4090_001')` com 409; (4) verifica `escrow.extension_approved === null` → senão lança `DaniConflictError('DCE-ESCR-4090_002')` com 409; (5) valida `dto.confirmation === true` → senão 400 `DCE-ESCR-4000_001`; (6) calcula `extension_deadline = deposit_deadline + 5 dias úteis` usando `addBusinessDays(deposit_deadline, 5)`; (7) atualiza `extension_approved = true`, `extension_deadline`, `extension_responded_at = now()`; (8) retorna `{ escrow_id, extension_approved: true, extension_deadline, extension_responded_at }`
  - Validação: `confirmation = false` → 400; sem extensão solicitada → 409 `DCE-ESCR-4090_001`; extensão já aprovada → 409 `DCE-ESCR-4090_002`; aprovação válida → `extension_deadline = deposit_deadline + 5 business days`

- [x] **Implementar `EscrowService.rejectExtension(proposalId: string, cedenteId: string)`**: (1) verifica ownership e valida estado igual ao `approveExtension`; (2) atualiza `extension_approved = false`, `extension_responded_at = now()`; (3) retorna `{ escrow_id, extension_approved: false, extension_responded_at }`; (4) dispara notificação ao Admin (via RabbitMQ `notification.send` queue): "O Cedente recusou a extensão de prazo. O prazo original permanece. Ação pode ser necessária."
  - Validação: rejeição bem-sucedida → `extension_approved = false`; `extension_deadline` permanece NULL (não houve extensão); notificação ao Admin disparada via RabbitMQ

- [x] **Implementar job de auto-aprovação de extensão por silêncio** em `src/modules/escrow/escrow.scheduler.ts`: (1) job agendado com `@Cron('*/5 * * * *')` (a cada 5 minutos); (2) busca escrows com `extension_requested = true AND extension_approved IS NULL AND (created_at_extension_request + 24h) <= now()`; (3) para cada escrow: atualiza `extension_approved = true`, `extension_deadline = deposit_deadline + 5 dias úteis`, `extension_responded_at = now()`; (4) dispara notificação ao Cedente: "O prazo foi automaticamente estendido em 5 dias úteis, pois você não respondeu dentro de 24 horas."; (5) loga cada auto-aprovação com `correlation_id` via Pino
  - Validação: escrow com extensão solicitada há 25h sem resposta → `extension_approved = true` após próxima execução do cron; texto da notificação exato conforme D01 RN-DCE-018; log estruturado com `escrow_id` e `auto_approved: true`

- [x] **Implementar `EscrowService.processDeposit(escrowId: string)`** (chamado via webhook interno ou Admin): (1) transiciona `AWAITING_DEPOSIT → DEPOSITED`; (2) preenche `deposited_at = now()`; (3) dispara notificação ao Cedente: "O depósito em Escrow foi confirmado. Próximo passo: assinatura do contrato."; (4) dispara evento `escrow.deposited` no RabbitMQ para o módulo de notificações
  - Validação: transição inválida (ex: `DEPOSITED → DEPOSITED`) → lança erro; `deposited_at` preenchido; notificação disparada

- [x] **Implementar alerta proativo de prazo (2 dias úteis restantes)** em `EscrowScheduler`: (1) job `@Cron('0 9 * * *')` (diário às 9h); (2) busca escrows `status = 'AWAITING_DEPOSIT'`; (3) calcula dias úteis restantes para `deposit_deadline`; (4) se restam exatamente 2 dias úteis → dispara notificação ao Cedente: "O prazo para o depósito em Escrow vence em 2 dias úteis. Se o depósito não for realizado, a proposta será cancelada automaticamente e sua oportunidade voltará ao marketplace."; (5) texto exato conforme D01 RN-DCE-018
  - Validação: escrow com 2 dias úteis restantes → notificação enviada; escrow com 3 dias restantes → sem notificação; texto da mensagem exato conforme D01

### Backend — EscrowController

- [x] **Implementar `GET /api/v1/proposals/:proposal_id/escrow`** em `EscrowController`: (1) aplica `JwtAuthGuard` + `CedenteIsolationMiddleware`; (2) chama `EscrowService.findByProposal(proposalId, cedenteId)`; (3) retorna `{ data: { id, proposal_id, amount, status, deposit_deadline, extension_deadline, deposited_at, released_at, reversed_at, extension_requested, extension_approved, created_at } }`
  - Validação: proposta sem escrow → 404 `DCE-ESCR-4040_001`; token inválido → 401; proposta de outro Cedente → 403

- [x] **Implementar `POST /api/v1/proposals/:proposal_id/escrow/extension/approve`** em `EscrowController`: (1) valida body `{ confirmation: boolean }`; (2) chama `EscrowService.approveExtension()`; (3) retorna HTTP 200 com `{ data: { escrow_id, extension_approved, extension_deadline, extension_responded_at } }`
  - Validação: `confirmation = false` → 400 `DCE-ESCR-4000_001`; body vazio → 400; sem extensão solicitada → 409

- [x] **Implementar `POST /api/v1/proposals/:proposal_id/escrow/extension/reject`** em `EscrowController`: (1) sem body obrigatório; (2) chama `EscrowService.rejectExtension()`; (3) retorna HTTP 200 com `{ data: { escrow_id, extension_approved, extension_responded_at } }`
  - Validação: rejeição duplicada → 409 `DCE-ESCR-4090_002`; sem extensão solicitada → 409 `DCE-ESCR-4090_001`

---

## ⚙️ FEATURE 2 — Webhook ZapSign e Régua de Assinatura

### Backend — ZapSign Webhook

- [x] **Implementar `POST /api/v1/webhooks/zapsign`** em `ZapSignController` (`src/modules/escrow/zapsign.controller.ts`): (1) **antes de qualquer processamento**: valida assinatura HMAC-SHA256 do header `X-ZapSign-Signature` contra `raw_body` usando `ZAPSIGN_WEBHOOK_SECRET`; (2) se assinatura inválida → retorna 401 imediatamente (sem log do body); (3) parse do body para extrair `event_type` e `document_id`; (4) roteamento por `event_type`: `document.signed` → `ZapSignService.handleSigned(document_id)`, `document.refused` → `ZapSignService.handleRefused(document_id)`, `document.expired` → `ZapSignService.handleExpired(document_id)`, `document.viewed` → log de auditoria apenas; (5) responde HTTP 200 em até 5s (ZapSign retenta se não receber 200 dentro de 5s); (6) processamento assíncrono via `setImmediate` para não bloquear a resposta de 200
  - Validação: request sem header `X-ZapSign-Signature` → 401; HMAC inválido → 401; HMAC válido → 200 imediato; `document.signed` dispara `handleSigned`; `document.viewed` apenas log; resposta 200 em < 5s garantida

- [x] **Implementar `ZapSignService.handleSigned(documentId: string)`** em `src/modules/escrow/zapsign.service.ts`: (1) busca proposta vinculada ao `documentId` (campo `zapsign_document_id` na tabela `proposals`); (2) se não encontrado: log de warning com `documentId` e retorna silenciosamente; (3) transiciona escrow para `RELEASED`: `status = 'RELEASED'`, `released_at = now()`; (4) transiciona oportunidade para `CLOSED_SOLD` via `OpportunityStateMachine.transition(IN_NEGOTIATION, CLOSED_SOLD)`; (5) dispara notificação ao Cedente: "Ótima notícia! O contrato foi assinado por todas as partes. O valor do Escrow será liberado para você em breve."
  - Validação: `documentId` desconhecido → log warning, sem erro; escrow transicionado para `RELEASED`; oportunidade para `CLOSED_SOLD`; notificação com texto exato de D01 RN-DCE-019

- [x] **Implementar `ZapSignService.handleRefused(documentId: string)`**: (1) busca proposta vinculada; (2) log de auditoria com `document_id`, `proposal_id`, `cedente_id` (hash), `timestamp`; (3) dispara alerta ao Admin via RabbitMQ `notification.send`: "Contrato recusado. Ação do Admin necessária. Document ID: {id}"; (4) não altera estado do escrow automaticamente — ação manual do Admin necessária
  - Validação: log de auditoria gerado; alerta ao Admin disparado; escrow não alterado (Admin decide)

- [x] **Implementar `ZapSignService.handleExpired(documentId: string)`**: (1) busca proposta vinculada; (2) dispara alerta ao Admin via RabbitMQ; (3) dispara notificação ao Cedente: "O prazo de assinatura expirou. O Admin foi notificado para avaliar os próximos passos."; (4) log de auditoria
  - Validação: notificação ao Cedente com texto exato de D01 RN-DCE-019; alerta ao Admin disparado; log de auditoria com campos obrigatórios

- [x] **Implementar `ZapSignWebhookGuard`** em `src/modules/escrow/guards/zapsign-webhook.guard.ts`: (1) extrai `X-ZapSign-Signature` do header; (2) lê raw body do request (usar `RawBodyMiddleware` para preservar buffer antes de parse JSON); (3) calcula `HMAC-SHA256(rawBody, ZAPSIGN_WEBHOOK_SECRET)` via `crypto.createHmac('sha256', secret).update(rawBody).digest('hex')`; (4) compara com header usando `crypto.timingSafeEqual()` (evitar timing attacks); (5) se inválido → lança `DaniUnauthorizedError` com HTTP 401; (6) aplica `@SkipAuth()` no endpoint (webhook não usa JWT — usa HMAC)
  - Validação: HMAC correto → guard passa; HMAC incorreto → 401; header ausente → 401; timing-safe comparison implementada (não usar `===` diretamente)

### Backend — Régua ZapSign (D+0/D+2/D+4/D+5)

- [x] **Implementar job de régua ZapSign** em `EscrowScheduler`: (1) `@Cron('0 8 * * *')` (diário às 8h UTC); (2) busca propostas com `status = 'ACCEPTED'` e `zapsign_sent_at IS NOT NULL`; (3) para cada proposta, calcula dias desde `zapsign_sent_at`; (4) **D+2**: dias_corridos == 2 → dispara notificação ao Cedente: "Seu contrato ainda aguarda assinatura. Você tem até [data] para assinar."; (5) **D+4**: dias_corridos == 4 → dispara notificação urgente ao Cedente + alerta automático ao Admin; (6) **D+5**: dias_corridos == 5 → dispara notificação ao Cedente: "O prazo de assinatura expirou. O Admin foi notificado para avaliar os próximos passos." + alerta Admin; (7) **D+0**: disparado uma única vez imediatamente ao criar o evento ZapSign: "O contrato de cessão foi enviado para sua assinatura eletrônica via ZapSign. Você receberá um e-mail com o link para assinar. O prazo para assinatura é de 5 dias úteis."
  - Validação: proposta em D+2 → lembrete enviado uma única vez; proposta em D+4 → alerta urgente + admin notificado; proposta em D+5 → notificação de expiração + admin; D+0 disparado no momento da criação do envelope; textos exatos conforme D01 RN-DCE-019

---

## ⚙️ FEATURE 3 — Reversão de Escrow

### Backend — Reversão

- [x] **Implementar `EscrowService.reverseEscrow(escrowId: string)` (chamado apenas por Admin)**: (1) verifica `status = 'DEPOSITED'` → senão lança `DaniBusinessError('DCE-ESCR-4220_001', 'Reversão só permitida para Escrow com depósito confirmado.')` com 422; (2) verifica `reversed_at IS NULL` → senão 409 (já revertido); (3) verifica que `deposited_at + 15 dias corridos >= now()` → senão lança `DaniBusinessError('DCE-ESCR-4220_002', 'Prazo de 15 dias corridos para reversão expirou.')` com 422; (4) transiciona `DEPOSITED → REVERSED`; (5) preenche `reversed_at = now()`; (6) dispara notificação ao Cedente: "O Escrow foi revertido. A negociação não foi concluída e sua oportunidade voltará ao marketplace."; (7) transiciona oportunidade de volta para `PUBLISHED`
  - Validação: reversão de escrow `AWAITING_DEPOSIT` → 422 `DCE-ESCR-4220_001`; reversão após 16 dias corridos → 422 `DCE-ESCR-4220_002`; reversão válida → escrow `REVERSED` + oportunidade `PUBLISHED`; notificação ao Cedente com texto

---

## 🖥️ FRONTEND (Widget Chat — Escrow)

### Componente EscrowStatusCard

- [x] **Implementar `EscrowStatusCard`** em `src/components/chat/EscrowStatusCard.tsx`: (1) mapeamento de `EscrowStatus` → label PT-BR → cor: `AWAITING_DEPOSIT → "Aguardando depósito" → amarelo`, `DEPOSITED → "Depositado" → verde`, `RELEASED → "Liberado ao Cedente" → verde escuro`, `REVERSED → "Revertido" → cinza`; (2) exibe `deposit_deadline` formatado: "Prazo: {dd/mm/aaaa}"; (3) se `extension_requested = true AND extension_approved IS NULL`: exibe banner amarelo "O Cessionário solicitou extensão de 5 dias. [Aprovar] [Rejeitar]"; (4) se `extension_approved = true`: exibe "Prazo estendido até {extension_deadline}"; (5) se `extension_approved = false`: exibe "Extensão recusada. Prazo original mantido."; (6) 4 estados obrigatórios: Skeleton, Empty, Error, Populated
  - Validação: banner de extensão visível apenas quando `extension_requested AND extension_approved IS NULL`; botões "Aprovar" e "Rejeitar" presentes no banner; 4 labels PT-BR exatos conforme D01 seção 8.2; 4 estados implementados

### Componente ZapSignStatusCard

- [x] **Implementar `ZapSignStatusCard`** em `src/components/chat/ZapSignStatusCard.tsx`: (1) exibe status da assinatura ZapSign: "Aguardando assinatura" (amarelo) / "Assinado" (verde) / "Expirado" (vermelho) / "Recusado" (cinza); (2) exibe prazo máximo de assinatura: "Assine até {data}" (D+5 desde o envio); (3) estado **Error**: se `status = 'EXPIRED'` → "O prazo de assinatura expirou. O Admin foi notificado."; (4) link de ação: "Acessar contrato ZapSign →" (abre URL do ZapSign em nova aba); (5) 4 estados obrigatórios: Skeleton, Empty (sem ZapSign ativo), Error, Populated
  - Validação: link externo para ZapSign abre em nova aba; prazo exibido como `dd/mm/aaaa`; 4 estados implementados; status `EXPIRED` exibe mensagem de Admin notificado

---

## 🔀 Cross-Módulo

- `EscrowService.processDeposit()` é disparado pelo módulo de notificações/webhook quando Cessionário confirma depósito — registrar como dependência do sistema externo (fora do escopo do agente AI-Dani-Cedente conforme D17)
- `ZapSignService.handleSigned()` chama `OpportunityStateMachine.transition(IN_NEGOTIATION, CLOSED_SOLD)` de S4 — importar `OpportunityModule`
- `EscrowService.reverseEscrow()` chama `OpportunityStateMachine.transition(IN_NEGOTIATION, PUBLISHED)` de S4
- Notificações proativas (D+0/D+2/D+4/D+5, alerta 2 dias úteis) são disparadas via RabbitMQ `notification.send` queue — módulo de notificações (S8) consome essas mensagens; registrar templates como pendência em S8
- Job scheduler de extensão automática usa `addBusinessDays()` — reaproveitar implementação de S6 via serviço compartilhado `BusinessDaysService` (criar em `src/common/services/business-days.service.ts`)

---

## 🧪 TESTES

- [x] **Criar `src/modules/escrow/escrow.service.spec.ts`** com testes unitários: (1) `findByProposal` de proposta sem escrow → lança `DCE-ESCR-4040_001`; (2) `approveExtension` sem `extension_requested` → lança `DCE-ESCR-4090_001`; (3) `approveExtension` já respondida → lança `DCE-ESCR-4090_002`; (4) `approveExtension` com `confirmation = false` → lança `DCE-ESCR-4000_001`; (5) `approveExtension` válida → `extension_deadline = deposit_deadline + 5 business days`; (6) `rejectExtension` válida → `extension_approved = false`; (7) `reverseEscrow` com `AWAITING_DEPOSIT` → lança `DCE-ESCR-4220_001`; (8) `reverseEscrow` após 16 dias corridos → lança `DCE-ESCR-4220_002`; (9) job auto-aprovação: escrow com extensão há 25h → `extension_approved = true` automaticamente
  - Validação: 9 cenários; mock do Prisma; mock do RabbitMQ (não envia mensagens reais)

- [x] **Criar `src/modules/escrow/zapsign.guard.spec.ts`** com testes do webhook guard: (1) HMAC válido com secret correto → guard passa; (2) HMAC inválido → 401; (3) header `X-ZapSign-Signature` ausente → 401; (4) timing-safe comparison: dois HMACs com tamanho diferente não causam erro (buffer comparison segura)
  - Validação: 4 cenários; sem chamada real ao ZapSign; `crypto.timingSafeEqual` testado com mocks

- [x] **Criar `src/modules/escrow/zapsign.service.spec.ts`** com testes: (1) `handleSigned` com `documentId` válido → escrow `RELEASED` + oportunidade `CLOSED_SOLD`; (2) `handleSigned` com `documentId` desconhecido → log warning, sem erro lançado; (3) `handleRefused` → log de auditoria + alerta Admin; (4) `handleExpired` → notificação ao Cedente + alerta Admin; (5) régua D+2 → lembrete enviado; (6) régua D+4 → alerta urgente + Admin; (7) régua D+5 → notificação expiração
  - Validação: 7 cenários; mock do `OpportunityStateMachine`; textos de notificação verificados por igualdade de string exata

- [x] **Criar `test/escrow.e2e-spec.ts`** com Supertest: (1) `GET /proposals/:id/escrow` com token válido → 200 com todos os campos; (2) `GET /escrow` de proposta de outro Cedente → 403; (3) `POST /extension/approve` válido → 200 com `extension_deadline`; (4) `POST /extension/approve` sem extensão solicitada → 409 `DCE-ESCR-4090_001`; (5) `POST /webhooks/zapsign` com HMAC válido → 200; (6) `POST /webhooks/zapsign` com HMAC inválido → 401
  - Validação: 6 cenários E2E; webhook testado com HMAC gerado corretamente; banco de teste real

---

## 🔍 AUTO-VERIFICAÇÃO S7

| Check                | Critério                                                                                                                                                                                                                                                               | Status |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura      | `EscrowService`, `EscrowController`, `ZapSignService`, `ZapSignController`, `ZapSignWebhookGuard`, `EscrowScheduler` — nomes exatos; tabela `escrow_transactions`; enum `EscrowStatus` conforme D12/D13/D16                                                            | [x]    |
| #2 Valores numéricos | Prazo depósito: 10 dias úteis; extensão: +5 dias úteis; auto-aprovação silêncio: 24h; reversão: 15 dias corridos; régua ZapSign: D+0/D+2/D+4/D+5; alerta proativo: 2 dias úteis restantes                                                                              | [x]    |
| #3 Error codes       | `DCE-ESCR-4000_001` (confirmation), `DCE-ESCR-4040_001` (não encontrado), `DCE-ESCR-4090_001` (sem extensão), `DCE-ESCR-4090_002` (já respondida), `DCE-ESCR-4220_001` (reversão estado inválido), `DCE-ESCR-4220_002` (reversão prazo expirado) — todos implementados | [x]    |
| #4 Estados           | 4 estados Escrow: `AWAITING_DEPOSIT`, `DEPOSITED`, `RELEASED`, `REVERSED`; transições implementadas com guards de estado                                                                                                                                               | [x]    |
| #5 HMAC ZapSign      | `ZapSignWebhookGuard` valida `X-ZapSign-Signature` via HMAC-SHA256 com `ZAPSIGN_WEBHOOK_SECRET`; `crypto.timingSafeEqual()` para comparison; header ausente → 401                                                                                                      | [x]    |
| #6 Auto-aprovação    | Job cron a cada 5min verifica extensões com 24h sem resposta; auto-aprovação com notificação ao Cedente com texto exato conforme D01 RN-DCE-018                                                                                                                        | [x]    |
| #7 Régua ZapSign     | D+0 (envio imediato), D+2 (lembrete), D+4 (urgente + admin), D+5 (expirado + admin) — textos exatos conforme D01 RN-DCE-019; job diário às 8h UTC                                                                                                                      | [x]    |
| #8 Reversão          | Reversão apenas para `DEPOSITED`; prazo máximo 15 dias corridos após `deposited_at`; oportunidade volta para `PUBLISHED`                                                                                                                                               | [x]    |
| #9 Webhook response  | `POST /webhooks/zapsign` responde HTTP 200 em < 5s; processamento assíncrono via `setImmediate`; `document.viewed` → apenas log, sem efeitos                                                                                                                           | [x]    |
| #10 Notificações     | D+0/D+2/D+4/D+5 disparadas via RabbitMQ `notification.send`; alerta 2 dias úteis via job diário; rejeição de extensão → notificação ao Admin                                                                                                                           | [x]    |
| #11 Anti-scaffold    | `ZapSignWebhookGuard` com HMAC real; `EscrowScheduler` com cron real; `reverseEscrow` com verificação de 15 dias corridos real; sem stubs                                                                                                                              | [x]    |
| #12 Cobertura REQs   | REQ-076 a REQ-089 — todos com ≥1 item no checklist                                                                                                                                                                                                                     | [x]    |
