# S8 — Fechamento e Financeiro

## Sprint 8 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                                     |
| ------------------ | ----------------------------------------------------------------------------------------- |
| **Sprint**         | S8 — Fechamento e Financeiro                                                              |
| **Template**       | B — Módulo Fullstack (feature com vertical slice Banco→Backend→Frontend→Wiring→Testes)    |
| **REQs cobertos**  | S8-001 a S8-016 (16 requisitos do Registro Mestre)                                        |
| **Docs fonte**     | 01.2 - RN Core e Receita · 06 - Mapa de Telas · 16 - Documentação de API · 10 - Glossário |
| **Total de itens** | 46 itens                                                                                  |
| **Status**         | Concluída                                                                                 |

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos: `financial_transactions`, `TransactionType`, `TransactionStatus`, telas T-FIN-01/02/03, 4 critérios de fechamento.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — 4 critérios de fechamento (RN-033); 15 dias calendário para reversão (RN-038); commission_buyer = 20% Δ; TransactionType: ESCROW_DEPOSIT, COMMISSION, REFUND, OPERATION_COMPLETED.
- [x] ✅ Check 4 — Glossário: `financial_transactions`, `commission_buyer`, `OPERATION_COMPLETED`.
- [x] ✅ Check 5 — Anti-scaffold R10: 4 critérios verificados em código, reversão com re-auth, filtros financeiros reais.
- [x] ✅ Check 6 — Máquina de estado TransactionStatus: PENDENTE→PROCESSADO/FALHOU. 4 TransactionTypes. Documentado.
- [x] ✅ Check 7 — Reversão: janela 15 dias calendário; reembolso 5 dias úteis.
- [x] ✅ Check 8 — Cross-módulo: fechamento vem de S7 (formalização CONCLUIDA); reversão aciona reembolso em S6.
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — Sem ambiguidades.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S8-001 a S8-016 cobertos.

---

## FEATURE 1 — Fechamento de Operação

### ⚙️ Backend

- [x] **S8-BE01** · Implementar `FinancialService.recordOperationClosure(negotiationId: string)` em `apps/api/src/modules/financial/`: chamado por `FormalizationService.completeFormalization()` (S7); validar os 4 critérios de fechamento (RN-033): (1) `formalizations.status = CONCLUIDA`; (2) `escrow_deposits.status = DEPOSITO_CONFIRMADO`; (3) `formalizations.buyer_signed_at IS NOT NULL` e `formalizations.seller_signed_at IS NOT NULL`; (4) `formalizations.anuencia_obtained_at IS NOT NULL`; se todos os 4 critérios atendidos: criar `financial_transactions` (type: `COMMISSION`, status: PROCESSADO, amount: `commission_buyer` da negociação); criar `financial_transactions` (type: `OPERATION_COMPLETED`, status: PROCESSADO, amount: `agreed_value`); atualizar `negotiations.status = ENCERRADA`; atualizar `opportunities.status = CONCLUIDA`; disparar NOT-CES-09 via NotificationService; se qualquer critério falhar: logar `error` e não prosseguir. (Doc 01.2 — RN-033, RN-034, RN-036)

- [x] **S8-BE02** · Implementar `GET /api/v1/financial/transactions` em `apps/api/src/modules/financial/`: guards `JwtAuthGuard` + `KycGuard`; retornar transações do cessionário; paginação offset-based (max 100, default 20); filtros: `?type=ESCROW_DEPOSIT|COMMISSION|REFUND|OPERATION_COMPLETED`, `?status=PENDENTE|PROCESSADO|FALHOU`, `?date_from=`, `?date_to=`; ordenação `created_at DESC`; incluir: `type`, `amount` (formatado), `status`, `description`, `negotiation_id`, `opportunity_code`; retornar `meta` com totais por tipo. (Doc 16 — GET /financial/transactions)

- [x] **S8-BE03** · Implementar `GET /api/v1/financial/transactions/:id` em `apps/api/src/modules/financial/`: guards `JwtAuthGuard` + `KycGuard`; verificar pertinência ao cessionário; retornar dados completos da transação + dados da negociação vinculada + oportunidade (sem Cedente); incluir `commission_breakdown: { delta_value, commission_rate: 0.20, commission_amount }` para transações de tipo COMMISSION. (Doc 16 — GET /financial/transactions/:id)

- [x] **S8-BE04** · Implementar `GET /api/v1/financial/summary` em `apps/api/src/modules/financial/`: guards `JwtAuthGuard` + `KycGuard`; retornar sumário financeiro: `{ total_invested: number, total_commissions: number, operations_completed: number, operations_active: number, avg_commission_rate: number, last_operation_date?: string }`; cache Redis `rs:financial:summary:{cessionario_id}` TTL 5min. (Doc 16 — GET /financial/summary)

---

## FEATURE 2 — Reversão de Operação

### ⚙️ Backend

- [x] **S8-BE05** · Implementar `POST /api/v1/negotiations/:id/reversal` (solicitação de reversão) em `apps/api/src/modules/financial/`: guards `JwtAuthGuard` + `KycGuard` + `ReAuthGuard`; validações: (1) `negotiations.status = ENCERRADA` — apenas operações concluídas podem ser revertidas; (2) `formalizations.anuencia_obtained_at > NOW() - 15 dias` (janela de 15 dias corridos — `CONFLICT-001` se expirada); body `{ reason: string }`; criar `financial_transactions` (type: `REFUND`, status: PENDENTE); atualizar `negotiations.status = CANCELADA` via campo específico de reversão; disparar NOT-CES-10 (reversão iniciada) via NotificationService; criar entrada em `audit_logs` (action: `reversal_requested`); retornar HTTP 202 `{ reversal_id, status: "PENDENTE", estimated_completion: "<5 dias úteis>" }`. (Doc 01.2 — RN-038; Doc 16 — POST /negotiations/:id/reversal)

- [x] **S8-BE06** · Implementar `GET /api/v1/negotiations/:id/reversal` em `apps/api/src/modules/financial/`: guards `JwtAuthGuard` + `KycGuard`; retornar status da reversão: `{ status: "PENDENTE"|"PROCESSADO"|"FALHOU", amount, initiated_at, estimated_completion, completed_at? }`; verificar `financial_transactions` record com type=REFUND para esta negociação. (Doc 16 — GET /negotiations/:id/reversal)

- [x] **S8-BE07** · Implementar job de alerta de janela de reversão em `apps/api/src/modules/financial/jobs/reversal-window.job.ts`: cron `0 10 * * *` (10h diariamente); buscar negociações com `status = ENCERRADA` e `anuencia_obtained_at` entre `NOW() - 15 dias` e `NOW() - 12 dias` (janela se encerra em 3 dias); disparar NOT-CES-11 via NotificationService "Você tem X dias para solicitar reversão desta operação."; não alertar se reversão já solicitada. (Doc 01.2 — RN-038; Doc 21 — NOT-CES-11)

### 🎨 Frontend

- [x] **S8-FE01** · Implementar tela `T-FIN-01 — Painel Financeiro` em `apps/web/src/features/financial/pages/FinancialPage.tsx` (rota `/financeiro`): buscar via `GET /api/v1/financial/summary` + `GET /api/v1/financial/transactions`; exibir: (1) cards de KPIs — total investido, total de comissões, operações concluídas; (2) tabela de transações com colunas: tipo, descrição, valor, status, data; filtros: por tipo (dropdown), por data (date range picker); paginação; click em linha → T-FIN-02; `EmptyState` "Nenhuma transação encontrada."; skeleton durante loading. (Doc 06 — T-FIN-01)

- [x] **S8-FE02** · Implementar tela `T-FIN-02 — Detalhe de Transação` em `apps/web/src/features/financial/pages/TransactionDetailPage.tsx` (rota `/financeiro/:id`): buscar via `GET /api/v1/financial/transactions/:id`; exibir: tipo de transação com ícone, valor em BRL, status, data, oportunidade vinculada (código), negociação vinculada; para tipo COMMISSION: exibir `commission_breakdown` (delta_value, taxa 20%, valor comissão); para tipo ESCROW_DEPOSIT: link "Ver comprovante" se disponível; para tipo REFUND: exibir status do reembolso e prazo estimado; botão "Solicitar Reversão" visível apenas se: tipo OPERATION_COMPLETED E dentro da janela de 15 dias E reversão não iniciada. (Doc 06 — T-FIN-02)

- [x] **S8-FE03** · Implementar tela `T-FIN-03 — Comprovante` em `apps/web/src/features/financial/pages/ReceiptPage.tsx` (rota `/financeiro/comprovante/:id`): exibir comprovante formatado da transação: código da operação, data, valor, tipo, status; botão "Compartilhar" (`Share.share()` no mobile); botão "Baixar PDF"; pinch-to-zoom no mobile para documentos anexados. (Doc 06 — T-FIN-03; Doc 11 — gesture pinch-to-zoom)

- [x] **S8-FE04** · Implementar modal de Reversão em `apps/web/src/features/financial/components/ReversalModal.tsx`: exibir informações: "Reversão disponível por X dias", valor a ser reembolsado, "O reembolso será processado em até 5 dias úteis após aprovação."; campo `reason` (textarea obrigatório, min 20 chars); `ReAuthModal` antes do submit; chama `POST /api/v1/negotiations/:id/reversal`; sucesso → toast "Solicitação de reversão enviada" + atualizar badge; erro `CONFLICT-001` (janela expirada) → "O prazo para reversão expirou." (Doc 01.2 — RN-038)

### 🔌 Wiring

- [x] **S8-W01** · Configurar Realtime subscription em `financial_transactions` no frontend: ao receber UPDATE com `status = PROCESSADO` em tipo COMMISSION → toast "Comissão creditada!"; ao receber REFUND PROCESSADO → toast "Reembolso processado!"; invalidar TanStack Query do summary e da lista de transações. (Doc 21 — NOT-CES-09, NOT-CES-10)

- [x] **S8-W02** · Garantir transação atômica Prisma em `FinancialService.recordOperationClosure()`: 4 critérios verificados em uma query; se qualquer write falhar, rollback completo. Usar `prisma.$transaction([...])`. Verificar com teste unitário que rollback ocorre se qualquer etapa falha. (Doc 01.2 — RN-033)

### 🧪 Testes

- [x] **S8-T01** · Testes unitários `FinancialService.recordOperationClosure()`: todos os 4 critérios atendidos → 2 transactions criadas (COMMISSION + OPERATION_COMPLETED), NOT-CES-09 disparada; critério 1 falho (formalização não CONCLUIDA) → não cria transactions; critério 4 falho (anuencia nula) → não cria. Cobertura 100% branches (critical business logic — igual a EscrowService).

- [x] **S8-T02** · Testes unitários `FinancialService` — reversão: dentro da janela 15 dias → reversão criada; janela expirada (16 dias) → `CONFLICT-001`; re-auth não verificado → `AUTH-010`; dupla solicitação → `CONFLICT-001`. Mock `audit_logs` insert.

- [x] **S8-T03** · Testes E2E Playwright — TC-CES-16 "Visualização do painel financeiro após operação concluída": login, operação com status ENCERRADA presente, navegar para /financeiro, verificar que transações COMMISSION e OPERATION_COMPLETED aparecem com valores corretos; verificar commission_breakdown com delta_value correto. (Doc 01.3 — TC-CES-16)

- [x] **S8-T04** · Teste de integração: `commission_buyer` na `financial_transactions` bate exatamente com o calculado em `calculateBuyerCommission()` para o mesmo conjunto de `currentTableValue`, `contractTableValue`, `cedentePaidValue`. Zero discrepância permitida. (Doc 01.2 — RN-031)

---

## 🔀 Cross-Módulo

- [x] **S8-CM01** · **[← S7]** `FormalizationService.completeFormalization()` chama `FinancialService.recordOperationClosure(negotiationId)`. Garantir que `FinancialModule` é importado por `FormalizationModule` ou que a comunicação ocorre via evento RabbitMQ `financial.operation.completed` para evitar dependência cíclica.

- [x] **S8-CM02** · **[→ S6]** Quando reversão iniciada via `POST /api/v1/negotiations/:id/reversal`, acionar `EscrowService.initiateRefund()` de S6 para criar `REFUND` transaction e notificar. Usar evento RabbitMQ `financial.refund.initiated` publicado por `FinancialService`.

- [x] **S8-CM03** · **[← S3]** Dashboard widget "Resumo Financeiro" deve refletir `total_invested` e `operations_count` atualizados. `FinancialService` invalida cache Redis `rs:dashboard:{cessionario_id}` ao criar novas transactions.

---

## 📊 COBERTURA DE REQs S8

| REQ ID | Descrição                                     | Item(s)                   |
| ------ | --------------------------------------------- | ------------------------- |
| S8-001 | 4 critérios de fechamento (RN-033)            | S8-BE01                   |
| S8-002 | Transactions COMMISSION + OPERATION_COMPLETED | S8-BE01                   |
| S8-003 | GET /financial/transactions                   | S8-BE02                   |
| S8-004 | GET /financial/transactions/:id               | S8-BE03                   |
| S8-005 | GET /financial/summary                        | S8-BE04                   |
| S8-006 | commission_breakdown no detalhe               | S8-BE03                   |
| S8-007 | POST /negotiations/:id/reversal               | S8-BE05                   |
| S8-008 | Janela reversão 15 dias calendário            | S8-BE05                   |
| S8-009 | GET /negotiations/:id/reversal                | S8-BE06                   |
| S8-010 | Job alerta janela reversão 3 dias             | S8-BE07                   |
| S8-011 | T-FIN-01 Painel Financeiro                    | S8-FE01                   |
| S8-012 | T-FIN-02 Detalhe Transação                    | S8-FE02                   |
| S8-013 | T-FIN-03 Comprovante                          | S8-FE03                   |
| S8-014 | Modal Reversão + re-auth                      | S8-FE04                   |
| S8-015 | Transação atômica Prisma                      | S8-W02                    |
| S8-016 | NOT-CES-09, NOT-CES-10, NOT-CES-11            | S8-BE01, S8-BE05, S8-BE07 |
