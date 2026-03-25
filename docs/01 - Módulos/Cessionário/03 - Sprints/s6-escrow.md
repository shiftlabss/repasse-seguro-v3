# S6 — Escrow

## Sprint 6 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo | Valor |
|---|---|
| **Sprint** | S6 — Escrow |
| **Template** | B — Módulo Fullstack (feature com vertical slice Banco→Backend→Frontend→Wiring→Testes) |
| **REQs cobertos** | S6-001 a S6-014 (14 requisitos do Registro Mestre) |
| **Docs fonte** | 01.2 - RN Core e Receita · 01.5 - Integrações · 06 - Mapa de Telas · 16 - Documentação de API · 17 - Integrações Externas |
| **Total de itens** | 42 itens |
| **Status** | Concluída |

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos: `escrow_deposits`, `EscrowStatus`, Celcoin, telas T-NEG-03/04.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — SLA confirmação Escrow 24h Admin; reembolso 5 dias úteis; Celcoin MVP manual.
- [x] ✅ Check 4 — Glossário: `DEPOSITO_CONFIRMADO`, `DEPOSITO_ENVIADO`, Celcoin conta escrow.
- [x] ✅ Check 5 — Anti-scaffold R10: upload de comprovante real, validação de arquivo, lógica de confirmação real.
- [x] ✅ Check 6 — Máquina de estado EscrowStatus: AGUARDANDO_DEPOSITO→DEPOSITO_ENVIADO→DEPOSITO_CONFIRMADO; REEMBOLSADO como estado terminal. Documentado completamente.
- [x] ✅ Check 7 — SLA confirmação 24h (RN-054); reembolso 5 dias úteis (RN-054).
- [x] ✅ Check 8 — Cross-módulo: DEPOSITO_CONFIRMADO → aciona S7 (Formalização). NOT-CES-07 ao expirar.
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — ADR-003 (Celcoin manual MVP) sinalizado.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S6-001 a S6-014 cobertos.

---

## FEATURE — Escrow (Depósito, Comprovante, Confirmação)

### 🗄️ Banco

- [x] **S6-B01** · Confirmar `escrow_deposits` table: `id UUID PK`, `negotiation_id UUID FK → negotiations.id RESTRICT`, `cessionario_id UUID FK → cessionarios.id RESTRICT`, `amount DECIMAL(15,2) NOT NULL`, `status EscrowStatus DEFAULT AGUARDANDO_DEPOSITO`, `receipt_url VARCHAR(500)`, `submitted_at TIMESTAMPTZ`, `confirmed_at TIMESTAMPTZ`, `confirmed_by UUID`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`. Índice `idx_escrow_deposits_negotiation_id`. RLS: cessionário lê apenas seus depósitos; update apenas por `service_role` (Admin confirma). (Doc 13 — model EscrowDeposit)

- [x] **S6-B02** · Confirmar que Supabase Storage tem bucket `escrow-receipts` privado: upload pelo cessionário via backend (service_role), Signed URLs para visualização Admin; MIME types aceitos: JPEG, PNG, PDF; tamanho máximo por arquivo: 10MB. Verificar que `receipt_url` armazena path no Storage, não a signed URL diretamente (URLs têm TTL). (Doc 17 — Supabase Storage)

### ⚙️ Backend

- [x] **S6-BE01** · Implementar `GET /api/v1/negotiations/:id/escrow` em `apps/api/src/modules/escrow/`: guards `JwtAuthGuard` + `KycGuard`; verificar pertinência ao cessionário; retornar dados completos do Escrow: `{ escrow_deposit_id, amount, status, receipt_url_signed?, submitted_at?, confirmed_at?, escrow_deadline, extension_used, extended_deadline?, days_remaining, can_request_extension, celcoin_account_data: { bank, agency, account, pix_key } }`; `celcoin_account_data` é configuração estática para MVP (ADR-003 — confirmação manual pelo Admin); `receipt_url_signed` gerada com TTL 15min se `receipt_url` existente. (Doc 16 — GET /escrow; Doc 01.2 — RN-028; ADR-003)

- [x] **S6-BE02** · Implementar `POST /api/v1/negotiations/:id/escrow/receipt` em `apps/api/src/modules/escrow/`: guards `JwtAuthGuard` + `KycGuard`; body: multipart/form-data com `file` (JPEG/PNG/PDF, max 10MB); validar MIME type e tamanho; upload para Supabase Storage `escrow-receipts/{cessionario_id}/{negotiation_id}/{timestamp}.{ext}` via `service_role`; atualizar `escrow_deposits`: `receipt_url = path`, `status = DEPOSITO_ENVIADO`, `submitted_at = NOW()`; criar mensagem de chat de sistema "Comprovante de depósito enviado. Aguardando confirmação em até 24 horas."; disparar NOT-CES-03 (adaptar para contexto Escrow) ou criar notificação interna para Admin; retornar `{ escrow_deposit_id, status: "DEPOSITO_ENVIADO", submitted_at }`. (Doc 16 — POST /escrow/receipt; Doc 01.2 — RN-030; Doc 01.4 — RN-054: SLA confirmação 24h)

- [x] **S6-BE03** · Implementar webhook / endpoint de confirmação Admin `PATCH /api/v1/negotiations/:id/escrow/confirm` (service_role only) em `apps/api/src/modules/escrow/`: guard `ServiceRoleGuard`; validar que `status = DEPOSITO_ENVIADO`; atualizar `escrow_deposits`: `status = DEPOSITO_CONFIRMADO`, `confirmed_at = NOW()`, `confirmed_by = admin_id`; atualizar `negotiations.status = DEPOSITO_CONFIRMADO`; criar registro em `financial_transactions` (type: `ESCROW_DEPOSIT`, status: PROCESSADO, amount = escrow amount); criar `formalizations` record com `status = DOCUMENTOS_DISPONIVEIS`; disparar NOT-CES-06 (depósito confirmado) via NotificationService; retornar HTTP 200. (Doc 16; Doc 01.2 — RN-030; Doc 01.4 — RN-054)

- [x] **S6-BE04** · Implementar monitoramento de SLA de confirmação (24h) em `apps/api/src/modules/escrow/jobs/escrow-confirmation-sla.job.ts`: cron `0 * * * *` (hourly); buscar `escrow_deposits` com `status = DEPOSITO_ENVIADO` e `submitted_at < NOW() - 24h`; para cada: criar alerta para Admin via notificação interna (sem expor ao Cessionário); logar no Pino com nível `warn`; não cancelar automaticamente (Admin resolve manualmente no MVP). (Doc 01.4 — RN-054: SLA confirmação 24h)

- [x] **S6-BE05** · Implementar lógica de reembolso em `EscrowService.initiateRefund()` (chamado quando negociação cancelada ou reversão iniciada): atualizar `escrow_deposits.status = REEMBOLSADO`; criar `financial_transactions` (type: `REFUND`, status: PENDENTE); criar mensagem de chat de sistema "Reembolso iniciado. Prazo: 5 dias úteis."; disparar NOT-CES-10 ou NOT-CES-11 via NotificationService; SLA: 5 dias úteis (RN-054). (Doc 01.2 — RN-038; Doc 01.4 — RN-054)

- [x] **S6-BE06** · Integração Celcoin (ADR-003 — MVP manual): em `apps/api/src/infrastructure/celcoin/`: configurar cliente HTTP com `CELCOIN_CLIENT_ID` e `CELCOIN_CLIENT_SECRET`; no MVP, a confirmação do depósito é feita manualmente pelo Admin consultando o painel Celcoin; `CelcoinService` expõe método `getEscrowAccountData()` que retorna dados bancários da conta Escrow (configuração estática); ⚠️ ADR-003: integração automática Celcoin adiada para v2 — confirmação via endpoint Admin. (Doc 17 — seção Celcoin; Doc 14 — ADR-003)

### 🎨 Frontend

- [x] **S6-FE01** · Atualizar tela `T-NEG-03` (de S5) para incluir upload de comprovante: após exibir dados bancários Celcoin, adicionar seção "Enviar Comprovante": drag-and-drop ou botão "Selecionar arquivo" (JPEG/PNG/PDF max 10MB); preview do arquivo selecionado; validação de tipo/tamanho no frontend antes do upload; botão "Enviar Comprovante" → chama `POST /api/v1/negotiations/:id/escrow/receipt` com multipart/form-data; progress bar durante upload; sucesso → toast "Comprovante enviado! Aguardando confirmação em até 24 horas." + atualizar status para DEPOSITO_ENVIADO; erro de MIME/tamanho → mensagem inline. (Doc 06 — T-NEG-03; Doc 01.2 — RN-030)

- [x] **S6-FE02** · Atualizar tela `T-NEG-04` (de S5) para exibir status completo do Escrow: ao receber evento Realtime de `escrow_deposits` UPDATE com `status = DEPOSITO_CONFIRMADO` → toast de alta prioridade "Seu depósito foi confirmado!" + animar badge; link "Ver Formalização" aparece quando DEPOSITO_CONFIRMADO (→ T-ASS-01 de S7); exibir comprovante enviado com link para download (Signed URL TTL 15min via `GET /api/v1/negotiations/:id/escrow`). (Doc 06 — T-NEG-04)

- [x] **S6-FE03** · Implementar feedback de upload de comprovante no mobile: usar `expo-image-picker` para galeria e `expo-document-picker` para PDFs; feedback háptico `NotificationFeedbackType.Success` ao upload concluído (D11 — seção 3.2); banner offline ao tentar upload sem rede: "Conecte-se à internet para enviar o comprovante" (ação bloqueada — não enfileirável). (Doc 11 — seção 4.3; Doc 01.5 — RN-067)

### 🔌 Wiring

- [x] **S6-W01** · Configurar Supabase Realtime subscription em `escrow_deposits` no frontend: filtro por `negotiation_id`; ao receber UPDATE com `status = DEPOSITO_CONFIRMADO` → invalidar TanStack Query de negotiation + escrow + dashboard; redirecionar banner para T-NEG-04 com animação de sucesso; ao receber notificação NOT-CES-06, exibir in-app banner de alta prioridade. (Doc 21 — NOT-CES-06)

- [x] **S6-W02** · Garantir que ao criar `formalizations` record (em S6-BE03), o `FormalizationsModule` (S7) já tem o record pronto para ser processado. Usar transação Prisma (`prisma.$transaction`) para criar `escrow_deposits` atualização + `formalizations` criação + `financial_transactions` inserção atomicamente — não permitir estado parcial. (Doc 01.2 — RN-030)

### 🧪 Testes

- [x] **S6-T01** · Testes unitários `EscrowService`: upload comprovante com MIME inválido → `VAL-001`; upload > 10MB → `VAL-001`; confirmar Escrow com status errado → `CONFLICT-001`; confirmar sucesso → `DEPOSITO_CONFIRMADO` + cria formalization + cria financial_transaction atomicamente. Cobertura 100% branches (EscrowService é crítico — RN igual a CommissionService).

- [x] **S6-T02** · Testes unitários `EscrowConfirmationSlaJob`: mock de `submitted_at = NOW() - 25h` com status `DEPOSITO_ENVIADO` → alerta interno gerado; sem falsos positivos (status diferente ou submitted_at recente não dispara alerta).

- [x] **S6-T03** · Testes E2E Playwright — TC-CES-12 "Upload de comprovante Escrow": login, navegar para negociação AGUARDANDO_DEPOSITO, upload de comprovante PDF, verificar status DEPOSITO_ENVIADO; TC-CES-13 "Confirmação de Escrow pelo Admin" (mock webhook): disparar confirmação → verificar DEPOSITO_CONFIRMADO + NOT-CES-06 enviada + formalization record criado. (Doc 01.3 — TC-CES-12, TC-CES-13)

---

## 🔀 Cross-Módulo

- [x] **S6-CM01** · **[→ S7]** Quando `escrow_deposits.status = DEPOSITO_CONFIRMADO`, criação atômica de `formalizations` record com `status = DOCUMENTOS_DISPONIVEIS`. S7 (Formalização) monitorará este record para acionar geração de documentos ZapSign.

- [x] **S6-CM02** · **[← S5]** Quando `negotiations.status = AGUARDANDO_DEPOSITO` (definido em S5), `EscrowService` deve ter `escrow_deposit_id` vinculado à negociação. Garantir que `negotiations` table tem FK ou que `escrow_deposits` tem `negotiation_id` (já confirmado em S6-B01).

- [x] **S6-CM03** · **[→ S8]** Quando `initiateRefund()` é chamado (em reversão S8), atualizar `escrow_deposits.status = REEMBOLSADO` + criar `REFUND` transaction. Garantir que `FinancialModule` é notificado via evento RabbitMQ `financial.refund.initiated`.

---

## 📊 COBERTURA DE REQs S6

| REQ ID | Descrição | Item(s) |
|---|---|---|
| S6-001 | GET /escrow com dados Celcoin | S6-BE01 |
| S6-002 | POST /escrow/receipt upload | S6-BE02 |
| S6-003 | DEPOSITO_ENVIADO status + submitted_at | S6-BE02 |
| S6-004 | PATCH /escrow/confirm (Admin) | S6-BE03 |
| S6-005 | DEPOSITO_CONFIRMADO + transactions + formalization | S6-BE03, S6-W02 |
| S6-006 | SLA confirmação 24h monitoramento | S6-BE04 |
| S6-007 | Reembolso REEMBOLSADO + SLA 5 dias úteis | S6-BE05 |
| S6-008 | Celcoin integração MVP manual ADR-003 | S6-BE06 |
| S6-009 | T-NEG-03 upload comprovante | S6-FE01 |
| S6-010 | T-NEG-04 status Realtime | S6-FE02 |
| S6-011 | Mobile upload (expo-image-picker, expo-document-picker) | S6-FE03 |
| S6-012 | Transação atômica Prisma | S6-W02 |
| S6-013 | NOT-CES-06 depósito confirmado | S6-BE03 |
| S6-014 | NOT-CES-07 prazo expirado | S5-BE08 (vinculado S5) |
