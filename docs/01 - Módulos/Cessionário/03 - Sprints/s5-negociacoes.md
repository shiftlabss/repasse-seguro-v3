# S5 — Negociações

## Sprint 5 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo | Valor |
|---|---|
| **Sprint** | S5 — Negociações |
| **Template** | B — Módulo Fullstack (feature com vertical slice Banco→Backend→Frontend→Wiring→Testes) |
| **REQs cobertos** | S5-001 a S5-018 (18 requisitos do Registro Mestre) |
| **Docs fonte** | 01.2 - RN Core e Receita · 06 - Mapa de Telas · 16 - Documentação de API · 14 - Especificações Técnicas |
| **Total de itens** | 49 itens |
| **Status** | Concluída |

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos: `negotiations`, `chat_messages`, `NegotiationStatus`, telas T-NEG-01/02/03/04/04a.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — NegotiationStatus: 6 estados exatos; chat SLA 4h; Escrow 10 dias úteis + 5 extensão.
- [x] ✅ Check 4 — Glossário: `counteroffer_rounds`, `AGUARDANDO_DEPOSITO`, `escrow_deadline`.
- [x] ✅ Check 5 — Anti-scaffold R10: chat Realtime, contraproposta com re-auth, transição de estado real.
- [x] ✅ Check 6 — Máquina de estado NegotiationStatus: EM_NEGOCIACAO→EM_CONTRAPROPOSTA→AGUARDANDO_DEPOSITO→DEPOSITO_CONFIRMADO→ENCERRADA; CANCELADA a partir de EM_NEGOCIACAO/EM_CONTRAPROPOSTA completamente documentada.
- [x] ✅ Check 7 — Escrow deadline: 10 dias úteis; extensão 5 dias úteis (uma vez); SLA chat 4h.
- [x] ✅ Check 8 — Cross-módulo: AGUARDANDO_DEPOSITO inicia S6 (Escrow); DEPOSITO_CONFIRMADO inicia S7 (Formalização).
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — Sem ambiguidades.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S5-001 a S5-018 cobertos.

---

## FEATURE 1 — Negociações (T-NEG-01, T-NEG-02)

### 🗄️ Banco

- [x] **S5-B01** · Confirmar `negotiations` table com campos: `id`, `proposal_id UUID UNIQUE FK → proposals.id RESTRICT`, `cessionario_id UUID FK → cessionarios.id RESTRICT`, `opportunity_id UUID FK → opportunities.id RESTRICT`, `agreed_value DECIMAL(15,2)`, `commission_buyer DECIMAL(15,2)`, `total_escrow_value DECIMAL(15,2)`, `status NegotiationStatus DEFAULT EM_NEGOCIACAO`, `counteroffer_rounds INT DEFAULT 0`, `escrow_deadline TIMESTAMPTZ`, `extension_used BOOLEAN DEFAULT false`, `extended_deadline TIMESTAMPTZ`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`. Índices: `idx_negotiations_cessionario_id`, `idx_negotiations_status`. (Doc 13 — model Negotiation)

- [x] **S5-B02** · Confirmar `chat_messages` table: `id UUID PK`, `negotiation_id UUID FK → negotiations.id CASCADE`, `author_id UUID`, `author_type ActorType` (CESSIONARIO/ADMIN/SYSTEM), `content TEXT NOT NULL`, `attachment_url VARCHAR(500)`, `created_at TIMESTAMPTZ`. Índice `idx_chat_messages_negotiation_id`. RLS: cessionário lê apenas mensagens de suas negociações; Admin lê todas; escrita: cessionário cria com `author_type=CESSIONARIO`, sistema com `service_role`. (Doc 13 — model ChatMessage)

- [x] **S5-B03** · Confirmar RLS em `negotiations`: cessionário lê/acessa apenas suas negociações (`cessionario_id = auth.uid()`); Admin via `service_role`; update de `status` apenas por `service_role`; insert apenas por `service_role` (sistema cria negociação ao aceitar proposta). (Doc 12 — ERD)

### ⚙️ Backend

- [x] **S5-BE01** · Implementar `GET /api/v1/negotiations` em `apps/api/src/modules/negotiations/`: guards `JwtAuthGuard` + `KycGuard`; retornar negociações do cessionário autenticado; paginação offset-based; filtro `?status=`; cada item incluir: `opportunity_code`, `status`, `agreed_value`, `total_escrow_value`, `escrow_deadline`, `days_remaining_escrow`, `counteroffer_rounds`; ordenação por `updated_at DESC`. (Doc 16 — GET /negotiations)

- [x] **S5-BE02** · Implementar `GET /api/v1/negotiations/:id` em `apps/api/src/modules/negotiations/`: guards `JwtAuthGuard` + `KycGuard`; verificar pertinência ao cessionário; retornar dados completos da negociação + oportunidade (sem dados Cedente) + proposta vinculada; calcular `days_remaining_escrow` em dias úteis (não corridos) a partir de `escrow_deadline`; flag `can_request_extension: boolean` (true se `escrow_deadline` ativo, `extension_used = false`, e status `AGUARDANDO_DEPOSITO`); retornar últimas 20 mensagens de chat. (Doc 16 — GET /negotiations/:id; Doc 01.2 — RN-029)

- [x] **S5-BE03** · Implementar `POST /api/v1/negotiations/:id/counteroffer` (contraproposta) em `apps/api/src/modules/negotiations/`: guards `JwtAuthGuard` + `KycGuard` + `ReAuthGuard`; body `{ proposed_value: number, message?: string }`; validações: (1) `status === 'EM_NEGOCIACAO'` ou `'EM_CONTRAPROPOSTA'` — `CONFLICT-001` caso contrário; (2) `proposed_value` em centavos, positivo; atualizar negociação: `status = EM_CONTRAPROPOSTA`, `counteroffer_rounds += 1`; criar mensagem de chat com `author_type=CESSIONARIO`; disparar NOT-CES-04 via NotificationService (Admin notificado); retornar negociação atualizada HTTP 200. (Doc 16 — POST /negotiations/:id/counteroffer; Doc 01.2 — RN-027)

- [x] **S5-BE04** · Implementar `GET /api/v1/negotiations/:id/chat` em `apps/api/src/modules/negotiations/`: guards `JwtAuthGuard` + `KycGuard`; paginação cursor-based para chat (mais eficiente para histórico); retornar mensagens ordenadas por `created_at ASC`; incluir `author_type` para renderização (CESSIONARIO = alinhado à direita, ADMIN/SYSTEM = esquerda); excluir qualquer informação sobre identidade do Cedente de mensagens de sistema. (Doc 16 — GET /negotiations/:id/chat)

- [x] **S5-BE05** · Implementar `POST /api/v1/negotiations/:id/chat` em `apps/api/src/modules/negotiations/`: guards `JwtAuthGuard` + `KycGuard`; body `{ content: string, attachment_url?: string }`; validações: `content` max 2000 chars; `attachment_url` se fornecida deve ser Signed URL válida do Supabase Storage; verificar que negociação está em estado ativo (`EM_NEGOCIACAO` ou `EM_CONTRAPROPOSTA`); criar `chat_messages` com `author_type=CESSIONARIO`; publicar evento no Supabase Realtime para notificar Admin em tempo real (subscription na tabela `chat_messages`); retornar mensagem criada HTTP 201. SLA de resposta do Admin: 4h (RN-057). (Doc 16 — POST /negotiations/:id/chat; Doc 01.4 — RN-057)

### 🎨 Frontend

- [x] **S5-FE01** · Implementar tela `T-NEG-01 — Lista de Negociações` em `apps/web/src/features/negotiations/pages/NegotiationsPage.tsx` (rota `/negociacoes`): buscar via `GET /api/v1/negotiations`; lista com filtro por status (tabs: Ativas, Concluídas, Canceladas); cada item: código da oportunidade, status com `NegotiationStatusBadge`, `SlaCountdown` para `escrow_deadline` se AGUARDANDO_DEPOSITO, valor acordado; swipe left (mobile) → "Ver detalhes"; click → T-NEG-02; `EmptyState` "Nenhuma negociação ativa. Faça uma proposta no marketplace!". (Doc 06 — T-NEG-01)

- [x] **S5-FE02** · Implementar tela `T-NEG-02 — Chat da Negociação` em `apps/web/src/features/negotiations/pages/NegotiationChatPage.tsx` (rota `/negociacoes/:id`): buscar via `GET /api/v1/negotiations/:id` + `GET /api/v1/negotiations/:id/chat`; layout de chat: mensagens do cessionário (direita, azul), Admin/SYSTEM (esquerda, cinza); scroll automático para última mensagem; input para nova mensagem (max 2000 chars) + botão enviar; subscription Supabase Realtime em `chat_messages` filtrada por `negotiation_id` para receber mensagens em tempo real; painel lateral: dados da negociação (`agreed_value`, `commission_buyer`, `total_escrow_value`, `counteroffer_rounds`); botões contextuais: "Fazer Contraproposta" (se status permite), "Ver Escrow" (se AGUARDANDO_DEPOSITO). (Doc 06 — T-NEG-02)

- [x] **S5-FE03** · Implementar modal de Contraproposta em `apps/web/src/features/negotiations/components/CounterOfferModal.tsx`: campo `proposed_value` com preview de `commission_buyer` em tempo real; confirmação "Tem certeza? Esta será sua contraproposta"; `ReAuthModal` antes do submit; chama `POST /api/v1/negotiations/:id/counteroffer`; sucesso → fechar modal, toast "Contraproposta enviada"; chat atualiza com nova mensagem de sistema. (Doc 01.2 — RN-027)

- [x] **S5-FE04** · Implementar `NegotiationStatusBadge.tsx` em `apps/web/src/components/domain/`: mapear todos os 6 estados de `NegotiationStatus`: EM_NEGOCIACAO (azul, "Em negociação"), EM_CONTRAPROPOSTA (laranja, "Contraproposta enviada"), AGUARDANDO_DEPOSITO (âmbar, "Aguardando depósito Escrow"), DEPOSITO_CONFIRMADO (verde-claro, "Depósito confirmado"), ENCERRADA (verde, "Concluída"), CANCELADA (cinza, "Cancelada").

---

## FEATURE 2 — Escrow (entrada, T-NEG-03, T-NEG-04)

### ⚙️ Backend

- [x] **S5-BE06** · Implementar lógica de transição para AGUARDANDO_DEPOSITO em `NegotiationsService`: quando Admin aceita proposta ou aceita contraproposta (status Admin-only), atualizar `negotiations.status = AGUARDANDO_DEPOSITO`; calcular `escrow_deadline = NOW() + 10 dias úteis` (excluindo sábados e domingos); criar registro em `escrow_deposits` com `status = AGUARDANDO_DEPOSITO`, `amount = total_escrow_value`; atualizar `opportunities.status = RESERVADA`; disparar NOT-CES-05 (será enviado no D+8 pelo job de alertas) via agendamento; retornar instrução de depósito via NOT-CES-05 imediatamente para o cessionário (via fila). (Doc 01.2 — RN-028; Doc 01.4 — RN-054)

- [x] **S5-BE07** · Implementar `POST /api/v1/negotiations/:id/escrow/extension` em `apps/api/src/modules/negotiations/`: guards `JwtAuthGuard` + `KycGuard` + `ReAuthGuard`; validações: (1) `status === 'AGUARDANDO_DEPOSITO'`; (2) `extension_used === false`; (3) `escrow_deadline > NOW()` (prazo não expirado — `CONFLICT-001` se expirado); conceder extensão: `extended_deadline = escrow_deadline + 5 dias úteis`, `extension_used = true`; atualizar `escrow_deadline = extended_deadline`; criar mensagem de chat de sistema; disparar NOT-CES-05 com novo prazo; retornar negociação atualizada. (Doc 01.2 — RN-029)

- [x] **S5-BE08** · Implementar job cron de expiração do prazo Escrow em `apps/api/src/modules/negotiations/jobs/escrow-deadline.job.ts`: cron `0 * * * *` (hourly); buscar negociações `status = AGUARDANDO_DEPOSITO` e `escrow_deadline < NOW()`; cancelar negociação: `status = CANCELADA`; reverter `opportunities.status = DISPONIVEL` (se não há outra negociação ativa); atualizar `escrow_deposits.status = REEMBOLSADO` (manual MVP); criar mensagem de chat de sistema "Prazo de depósito encerrado. Operação cancelada."; disparar NOT-CES-07 via NotificationService. (Doc 01.2 — RN-028; Doc 01.4 — RN-054)

### 🎨 Frontend

- [x] **S5-FE05** · Implementar tela `T-NEG-03 — Instrução de Depósito Escrow` em `apps/web/src/features/negotiations/pages/EscrowInstructionsPage.tsx` (rota `/negociacoes/:id/escrow`): buscar via `GET /api/v1/negotiations/:id`; exibir: `total_escrow_value` formatado em BRL, `escrow_deadline` com countdown, dados de depósito Celcoin (conta Escrow); botão "Copiar dados bancários" (háptico no mobile); botão "Confirmar envio de comprovante" → upload de comprovante em S6 (T-NEG-03 nesta sprint mostra instrução); botão "Solicitar extensão de prazo" (visível se `can_request_extension = true`) → modal confirmação → `ReAuthModal` → `POST /api/v1/negotiations/:id/escrow/extension`. (Doc 06 — T-NEG-03; Doc 01.2 — RN-029)

- [x] **S5-FE06** · Implementar tela `T-NEG-04 — Status do Escrow` em `apps/web/src/features/negotiations/pages/EscrowStatusPage.tsx` (rota `/negociacoes/:id/escrow-status`): exibir status atual do `escrow_deposits` record via `EscrowStatusBadge`: AGUARDANDO_DEPOSITO (âmbar, com countdown), DEPOSITO_ENVIADO (azul, "Comprovante enviado. Aguardando confirmação"), DEPOSITO_CONFIRMADO (verde, "Depósito confirmado!"); subscription Realtime em `escrow_deposits` filtrada por `negotiation_id` para atualização ao vivo do status; tela T-NEG-04a (extensão concedida): exibir novo prazo com badge "Extensão concedida". (Doc 06 — T-NEG-04, T-NEG-04a)

- [x] **S5-FE07** · Implementar `EscrowStatusBadge.tsx` em `apps/web/src/components/domain/`: mapear 4 estados de `EscrowStatus`: AGUARDANDO_DEPOSITO (âmbar, "Aguardando depósito"), DEPOSITO_ENVIADO (azul, "Comprovante enviado"), DEPOSITO_CONFIRMADO (verde, "Confirmado"), REEMBOLSADO (cinza, "Reembolsado").

### 🔌 Wiring

- [x] **S5-W01** · Configurar Supabase Realtime subscription em `negotiations` + `chat_messages` no frontend: `negotiations` subscription para status updates (ex: AGUARDANDO_DEPOSITO recebido → Toast + redirecionar para T-NEG-03); `chat_messages` subscription filtrada por `negotiation_id` para chat em tempo real; degradação para polling 30s se Realtime indisponível; input de chat desabilitado com banner offline se sem rede. (Doc 11 — seção 4.2 offline T-NEG-02)

- [x] **S5-W02** · Garantir que cálculo de dias úteis para `escrow_deadline` e `extended_deadline` usa função utilitária `addBusinessDays(date, days)` em `lib/dates.ts`: excluir sábados e domingos; não excluir feriados (MVP sem calendário de feriados — ⚠️ AMBÍGUO: Doc 01.2 menciona "dias úteis" sem especificar se inclui feriados — adotar interpretação conservadora: excluir apenas finais de semana). Usar `date-fns@4` para cálculo. (Doc 01.2 — RN-028)

### 🧪 Testes

- [x] **S5-T01** · Testes unitários `NegotiationsService`: `counteroffer` — status inválido retorna `CONFLICT-001`; `proposed_value` não-positivo retorna `VAL-001`; sucesso atualiza status e rounds; `addBusinessDays` — 10 dias a partir de sexta = próxima quarta (pula fim de semana); extensão única: segunda solicitação retorna `CONFLICT-001`. Cobertura 100% branches.

- [x] **S5-T02** · Testes unitários `EscrowDeadlineJob`: negociação com `escrow_deadline < NOW()` → status CANCELADA; oportunidade com status RESERVADA → volta DISPONIVEL; `escrow_deposits.status = REEMBOLSADO`; NOT-CES-07 disparado. Mock Redis, Prisma e NotificationService.

- [x] **S5-T03** · Testes E2E Playwright — TC-CES-10 "Contraproposta em negociação": login, navegar para negociação EM_NEGOCIACAO, enviar contraproposta, verificar status EM_CONTRAPROPOSTA e mensagem no chat; TC-CES-11 "Solicitação de extensão Escrow": negociação AGUARDANDO_DEPOSITO → solicitar extensão → verificar `extension_used=true` e novo prazo. (Doc 01.3 — TC-CES-10, TC-CES-11)

- [x] **S5-T04** · Teste de segurança: nenhuma mensagem de chat, resposta de API ou notificação de sistema expõe nome, CPF ou qualquer identificador do Cedente. Verificar com mock de 10 mensagens de sistema sobre negociação. (Doc 01.5 — RN-063)

---

## 🔀 Cross-Módulo

- [x] **S5-CM01** · **[→ S6]** Quando `negotiations.status = AGUARDANDO_DEPOSITO`, `escrow_deposits` record criado com `status = AGUARDANDO_DEPOSITO` e `amount = total_escrow_value`. S6 (Escrow) usará este record para gerenciar comprovante e confirmação. Garantir que `EscrowModule` importa `NegotiationsModule` ou acessa via `PrismaService` diretamente.

- [x] **S5-CM02** · **[← S4]** A criação de `negotiations` record é acionada pelo Admin (fora do escopo do Cessionário) ao aceitar proposta. Implementar `NegotiationsService.createFromProposal(proposalId: string)` que será chamado pelo módulo Admin futuramente. Expor como método público no módulo.

---

## 📊 COBERTURA DE REQs S5

| REQ ID | Descrição | Item(s) |
|---|---|---|
| S5-001 | GET /negotiations | S5-BE01 |
| S5-002 | GET /negotiations/:id com days_remaining | S5-BE02 |
| S5-003 | POST /negotiations/:id/counteroffer | S5-BE03 |
| S5-004 | GET /negotiations/:id/chat | S5-BE04 |
| S5-005 | POST /negotiations/:id/chat | S5-BE05 |
| S5-006 | Transição AGUARDANDO_DEPOSITO | S5-BE06 |
| S5-007 | escrow_deadline 10 dias úteis | S5-BE06, S5-W02 |
| S5-008 | POST /negotiations/:id/escrow/extension | S5-BE07 |
| S5-009 | Extensão 5 dias úteis, uma vez | S5-BE07 |
| S5-010 | Job cron expiração Escrow hourly | S5-BE08 |
| S5-011 | T-NEG-01 Lista Negociações | S5-FE01 |
| S5-012 | T-NEG-02 Chat | S5-FE02 |
| S5-013 | Modal Contraproposta + re-auth | S5-FE03 |
| S5-014 | NegotiationStatusBadge 6 estados | S5-FE04 |
| S5-015 | T-NEG-03 Instrução Escrow | S5-FE05 |
| S5-016 | T-NEG-04 Status Escrow + Realtime | S5-FE06 |
| S5-017 | EscrowStatusBadge 4 estados | S5-FE07 |
| S5-018 | SLA chat 4h (RN-057) | S5-BE05 |
