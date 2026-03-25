# S6 — Propostas e Negociação

| Campo                | Valor                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S6                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| **Nome**             | Propostas e Negociação                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| **Tipo**             | Módulo Fullstack (Template B)                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Template**         | B (Módulo Fullstack: por Feature, vertical slice Banco→Backend→Frontend→Wiring→Testes)                                                                                                                                                                                                                                                                                                                                                                         |
| **Docs Consultados** | D01 (RN-DCE-014, RN-DCE-015, RN-DCE-016, RN-DCE-017), D05.3 (RF-DCE-016 a RF-DCE-021), D12, D13, D16 (domínio Proposal)                                                                                                                                                                                                                                                                                                                                        |
| **Cross-cutting**    | D10 (Glossário)                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **REQs cobertos**    | REQ-061, REQ-062, REQ-063, REQ-064, REQ-065, REQ-066, REQ-067, REQ-068, REQ-069, REQ-070, REQ-071, REQ-072, REQ-073, REQ-074, REQ-075                                                                                                                                                                                                                                                                                                                          |
| **Objetivo**         | Máquina de estados da proposta (6 estados, 5 transições), `GET /proposals`, `GET /proposals/:id`, `POST /proposals/:id/accept` (cria EscrowTransaction + inicia ZapSign), `POST /proposals/:id/reject`, `POST /proposals/:id/counter`, identidade do Cessionário nunca exposta, tabela comparativa de múltiplas propostas, confirmação `{ confirmation: true }` obrigatória no aceite, retorno líquido calculado com aviso obrigatório, testes unitários e E2E |

---

## Critério de Conclusão da S6

Ao final desta sprint:

- `GET /opportunities/:id/proposals` retorna propostas sem `cessionario_id`, apenas `proposal_value`, `counter_value`, `status`, `expires_at`
- `POST /proposals/:id/accept` com `{ "confirmation": true }` retorna 200 com `escrow.deposit_deadline = now() + 10 dias úteis` e `status: 'AWAITING_DEPOSIT'`
- `POST /proposals/:id/accept` com `confirmation != true` retorna 400 `DCE-PROP-4000_001`
- `POST /proposals/:id/accept` de proposta expirada retorna 422 `DCE-PROP-4220_001`
- `POST /proposals/:id/accept` de proposta já respondida retorna 409 `DCE-PROP-4090_001`
- `POST /proposals/:id/reject` transiciona proposta para `REJECTED` e oportunidade volta para `PUBLISHED`
- `POST /proposals/:id/counter` com `counter_value ≤ 0` retorna 400 `DCE-PROP-4000_002`
- `POST /proposals/:id/counter` transiciona proposta para `COUNTER_SENT`
- Aceite com dossiê incompleto retorna 422 `DCE-PROP-4220_002`
- Identidade do Cessionário (`cessionario_id`, nome, CPF) nunca presente em nenhum response do domínio Proposal

---

## ⚙️ FEATURE 1 — Máquina de Estados da Proposta

### Banco

- [x] **Verificar que migration da tabela `proposals`** já existe no S1 com campos: `id UUID PK`, `opportunity_id UUID FK → opportunities.id`, `cessionario_id UUID NOT NULL` (nunca exposto em response do Cedente), `proposal_value DECIMAL(12,2) NOT NULL`, `counter_value DECIMAL(12,2)`, `status ProposalStatus NOT NULL DEFAULT 'RECEIVED'`, `reason TEXT`, `expires_at TIMESTAMPTZ NOT NULL`, `responded_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ`
  - Validação: `\d proposals` mostra todos os campos; `proposal_value` e `counter_value` são `DECIMAL(12,2)` — sem Float; `cessionario_id` presente no banco mas **nunca selecionado** em queries do Cedente; `expires_at` NOT NULL (toda proposta tem prazo)
- [x] **Verificar enum `ProposalStatus`** já definido no S1 com exatamente os 6 valores: `RECEIVED`, `IN_ANALYSIS`, `ACCEPTED`, `REJECTED`, `COUNTER_SENT`, `EXPIRED`
  - Validação: `SELECT enum_range(NULL::ProposalStatus)` retorna os 6 valores exatos; nenhum valor extra; enum mapeado corretamente no Prisma schema
- [x] **Verificar RLS em `proposals`**: policy via JOIN — `USING (opportunity_id IN (SELECT id FROM opportunities WHERE cedente_id = current_setting('app.current_cedente_id')::uuid AND deleted_at IS NULL))`
  - Validação: Cedente A não vê propostas da oportunidade do Cedente B; query cruzada retorna zero linhas; `cessionario_id` nunca retornado por queries com RLS do Cedente

### Backend — ProposalStateMachine

- [x] **Implementar `ProposalStateMachine`** em `src/modules/proposal/proposal.state-machine.ts`: (1) define 6 estados: `RECEIVED`, `IN_ANALYSIS`, `ACCEPTED`, `REJECTED`, `COUNTER_SENT`, `EXPIRED`; (2) define transições permitidas: `RECEIVED → IN_ANALYSIS` (Cedente abre proposta), `IN_ANALYSIS → ACCEPTED` (Cedente aceita), `IN_ANALYSIS → REJECTED` (Cedente recusa), `IN_ANALYSIS → COUNTER_SENT` (Cedente envia contraproposta), `RECEIVED/IN_ANALYSIS → EXPIRED` (prazo expirado via job); (3) método `canTransition(from, to): boolean`; (4) lança `DaniBusinessError('DCE-PROP-4090_001')` se transição inválida (409); (5) `isRespondable(status): boolean` → retorna `true` apenas para `RECEIVED` e `IN_ANALYSIS`
  - Validação: `canTransition(IN_ANALYSIS, ACCEPTED)` → `true`; `canTransition(ACCEPTED, REJECTED)` → `false` (409); `canTransition(EXPIRED, ACCEPTED)` → `false`; `isRespondable(RECEIVED)` → `true`; `isRespondable(ACCEPTED)` → `false`; testes cobrem todas as 5 transições válidas e 3+ inválidas

---

## ⚙️ FEATURE 2 — CRUD de Propostas

### Backend — ProposalService

- [x] **Implementar `ProposalService.findAll(opportunityId: string, cedenteId: string)`** em `src/modules/proposal/proposal.service.ts`: (1) verifica ownership da oportunidade; (2) busca `proposals WHERE opportunity_id = opportunityId AND deleted_at IS NULL`; (3) **campos selecionados**: `id`, `proposal_value`, `counter_value`, `status`, `expires_at`, `responded_at`, `created_at` — **`cessionario_id` excluído explicitamente do SELECT**; (4) retorna array ordenado por `created_at DESC`; (5) calcula `net_return_estimated` para cada proposta: `proposal_value - saldo_devedor` da oportunidade
  - Validação: response nunca contém `cessionario_id`, nome ou qualquer PII do Cessionário; `net_return_estimated` presente para cada proposta; oportunidade de outro Cedente → 403; `deleted_at IS NOT NULL` excluído

- [x] **Implementar `ProposalService.findOne(proposalId: string, opportunityId: string, cedenteId: string)`**: (1) verifica ownership; (2) busca `proposal WHERE id = proposalId AND opportunity_id = opportunityId AND deleted_at IS NULL`; (3) lança `DaniNotFoundError('DCE-PROP-4040_001')` se não encontrado; (4) lança `DaniForbiddenError('DCE-PROP-4030_001')` se oportunidade de outro Cedente; (5) retorna proposta sem `cessionario_id`; (6) inclui `net_return_estimated = proposal_value - oportunidade.saldo_devedor`
  - Validação: response não tem `cessionario_id`; `net_return_estimated` calculado; proposta expirada ainda retornada (Cedente pode consultar histórico)

- [x] **Implementar `ProposalService.acceptProposal(proposalId, opportunityId, cedenteId, dto: AcceptProposalDto)`**: (1) verifica `dto.confirmation === true` → senão lança `DaniBusinessError('DCE-PROP-4000_001')` com HTTP 400; (2) verifica ownership; (3) busca proposta; (4) verifica `ProposalStateMachine.isRespondable(status)` → senão 409; (5) verifica `expires_at > now()` → senão 422 `DCE-PROP-4220_001`; (6) verifica dossiê completo (`DossierService.isDossierComplete(opportunityId)`) → senão 422 `DCE-PROP-4220_002`; (7) transiciona proposta para `ACCEPTED`; (8) define `responded_at = now()`; (9) **cria `EscrowTransaction`** com: `status = 'AWAITING_DEPOSIT'`, `amount = proposal_value`, `deposit_deadline = now() + 10 dias úteis`; (10) transiciona oportunidade para `IN_NEGOTIATION` via `OpportunityStateMachine`; (11) retorna proposta com `escrow` embutido
  - Validação: `confirmation = false` → 400 `DCE-PROP-4000_001`; proposta expirada → 422 `DCE-PROP-4220_001`; dossiê incompleto → 422 `DCE-PROP-4220_002`; aceite bem-sucedido → proposta `ACCEPTED` + escrow `AWAITING_DEPOSIT` + oportunidade `IN_NEGOTIATION`; `deposit_deadline` = `now() + 10 business days`

- [x] **Implementar cálculo de `deposit_deadline` (10 dias úteis)** em `src/modules/proposal/proposal.service.ts`: (1) método `addBusinessDays(date: Date, days: number): Date`; (2) exclui sábados (getDay() = 6) e domingos (getDay() = 0); (3) não considera feriados (documentado como limitação — feriados são de responsabilidade do Admin); (4) retorna timestamp final com hora 23:59:59Z do dia calculado
  - Validação: `addBusinessDays(2026-03-23_Monday, 10)` → `2026-04-07_Tuesday` (excluindo 2 finais de semana); sábado e domingo não contados; resultado com horário `23:59:59Z`

- [x] **Implementar `ProposalService.rejectProposal(proposalId, opportunityId, cedenteId, dto: RejectProposalDto)`**: (1) verifica ownership; (2) verifica `isRespondable(status)` → senão 409 `DCE-PROP-4090_001`; (3) verifica `expires_at > now()` → senão 422 `DCE-PROP-4220_001`; (4) transiciona proposta para `REJECTED`; (5) persiste `reason` (até 500 chars) se fornecido; (6) define `responded_at = now()`; (7) transiciona oportunidade de volta para `PUBLISHED` via `OpportunityStateMachine.transition(IN_NEGOTIATION, PUBLISHED)`; (8) retorna `{ proposal_id, status: 'REJECTED', responded_at }`
  - Validação: recusa → proposta `REJECTED` + oportunidade `PUBLISHED`; proposta já `ACCEPTED` → 409; `reason` até 500 chars aceito; `reason` não obrigatório

- [x] **Implementar `ProposalService.sendCounter(proposalId, opportunityId, cedenteId, dto: CounterProposalDto)`**: (1) verifica ownership; (2) verifica `isRespondable(status)` → senão 409; (3) verifica `expires_at > now()` → senão 422; (4) valida `dto.counter_value > 0` → senão 400 `DCE-PROP-4000_002`; (5) transiciona proposta para `COUNTER_SENT`; (6) persiste `counter_value` e `message` (até 1000 chars); (7) define `responded_at = now()`; (8) retorna `{ proposal_id, counter_value, status: 'COUNTER_SENT', responded_at }`
  - Validação: `counter_value = 0` → 400 `DCE-PROP-4000_002`; `counter_value = -1` → 400; contraproposta válida → `COUNTER_SENT`; oportunidade permanece `IN_NEGOTIATION` (não transiciona); `message` até 1000 chars aceito; `message` opcional

### Backend — ProposalController

- [x] **Implementar `GET /api/v1/opportunities/:opportunity_id/proposals`** em `ProposalController`: (1) aplica `JwtAuthGuard` + `CedenteIsolationMiddleware`; (2) chama `ProposalService.findAll(opportunityId, cedenteId)`; (3) retorna `{ data: [...], meta: { page, per_page, total, total_pages } }` com suporte a `?status=IN_ANALYSIS&page=1&per_page=20`; (4) campo `cessionario_id` **proibido no response** — interceptor verifica ausência
  - Validação: response sem `cessionario_id` em nenhum objeto da lista; com múltiplas propostas → `net_return_estimated` calculado por proposta; sem token → 401

- [x] **Implementar `GET /api/v1/opportunities/:opportunity_id/proposals/:proposal_id`** em `ProposalController`
  - Validação: response inclui `net_return_estimated`; sem `cessionario_id`; `status = 'EXPIRED'` retornado corretamente (consulta histórica)

- [x] **Implementar `POST /api/v1/opportunities/:opportunity_id/proposals/:proposal_id/accept`** em `ProposalController`: (1) valida body `{ confirmation: boolean }`; (2) chama `ProposalService.acceptProposal()`; (3) retorna HTTP 200 com `{ data: { proposal_id, status, responded_at, escrow: { id, amount, status, deposit_deadline } } }`
  - Validação: body `{ confirmation: true }` → 200 com escrow; body `{}` ou `{ confirmation: false }` → 400 `DCE-PROP-4000_001`

- [x] **Implementar `POST /api/v1/opportunities/:opportunity_id/proposals/:proposal_id/reject`** em `ProposalController`: (1) valida body `{ reason?: string (max 500) }`; (2) chama `ProposalService.rejectProposal()`; (3) retorna HTTP 200 com `{ data: { proposal_id, status, responded_at } }`
  - Validação: body com `reason > 500 chars` → 400; sem `reason` → aceito

- [x] **Implementar `POST /api/v1/opportunities/:opportunity_id/proposals/:proposal_id/counter`** em `ProposalController`: (1) valida body `{ counter_value: number > 0, message?: string (max 1000) }`; (2) chama `ProposalService.sendCounter()`; (3) retorna HTTP 200 com `{ data: { proposal_id, counter_value, status, responded_at } }`
  - Validação: `counter_value = -50` → 400 `DCE-PROP-4000_002`; `message > 1000 chars` → 400; aceito sem `message`

- [x] **Criar DTOs** em `src/modules/proposal/dto/`: (1) `accept-proposal.dto.ts` com `confirmation: boolean` + `@IsBoolean()` + `@IsNotEmpty()`; (2) `reject-proposal.dto.ts` com `reason?: string` + `@IsOptional()` + `@MaxLength(500)`; (3) `counter-proposal.dto.ts` com `counter_value: number` + `@IsPositive()` + `@IsNotEmpty()` + `message?: string` + `@IsOptional()` + `@MaxLength(1000)` + `@Type(() => Number)`
  - Validação: `confirmation = "true"` (string) → ValidationError (deve ser boolean); `counter_value = "165000"` string → convertido via `@Type(() => Number)` para 165000 number; `reason = null` → aceito (opcional)

---

## ⚙️ FEATURE 3 — Tabela Comparativa e Análise de Propostas

### Backend — ProposalComparisonService

- [x] **Implementar `ProposalService.getComparisonTable(opportunityId: string, cedenteId: string)`** em `proposal.service.ts`: (1) busca todas as propostas `status IN ('RECEIVED', 'IN_ANALYSIS')` da oportunidade; (2) para cada proposta calcula `net_return_estimated = proposal_value - oportunidade.saldo_devedor`; (3) calcula `variance_pct` em relação ao `repasse_value` do cenário escolhido: `((proposal_value - scenario_repasse_value) / scenario_repasse_value) × 100`; (4) calcula `expires_in_hours = (expires_at - now()) / 3600`; (5) retorna array ordenado por `net_return_estimated DESC` (melhor retorno primeiro); (6) se `proposals.length < 2` → retorna array com 1 elemento (sem tabela comparativa renderizada no frontend, mas dados disponíveis)
  - Validação: 2 propostas → array com 2 objetos ordenados por `net_return_estimated DESC`; `variance_pct` calculado corretamente; `expires_in_hours` negativo para propostas expiradas; nenhum campo de identidade do Cessionário no output

---

## 🖥️ FRONTEND (Widget Chat — Propostas)

### Componente ProposalCard

- [x] **Implementar `ProposalCard`** em `src/components/chat/ProposalCard.tsx`: (1) exibe proposta com campos: `proposal_value` (formatado em `R$ X.XXX,XX`), `net_return_estimated` (formatado, vermelho se negativo), `status` via `ProposalStatusBadge`, `expires_at` formatado em `dd/mm/aaaa HH:mm`; (2) botões de ação condicionais por status: se `RECEIVED` ou `IN_ANALYSIS` → exibe "Aceitar", "Recusar", "Contrapropor"; se outros status → sem botões de ação; (3) estado **Skeleton**: 4 linhas de loading; (4) estado **Empty**: "Nenhuma proposta recebida para esta oportunidade."; (5) estado **Error**: "Não foi possível carregar as propostas. [Tentar novamente]"; (6) estado **Populated**: card com dados e botões
  - Validação: `net_return_estimated = -30000` exibido em vermelho; botões ausentes quando `status = 'ACCEPTED'`; 4 estados implementados; valores em locale pt-BR

### Componente ProposalStatusBadge

- [x] **Implementar `ProposalStatusBadge`** em `src/components/chat/ProposalStatusBadge.tsx`: (1) mapeamento de `ProposalStatus` → label PT-BR → cor: `RECEIVED → "Recebida" → azul claro`, `IN_ANALYSIS → "Em análise" → amarelo`, `ACCEPTED → "Aceita" → verde`, `REJECTED → "Recusada" → vermelho`, `COUNTER_SENT → "Contraproposta enviada" → azul`, `EXPIRED → "Expirada" → cinza`; (2) 6 status mapeados; nenhum retorna label vazio
  - Validação: 6 variantes com labels PT-BR exatos conforme D01/D05.3; snapshot test cobre 6 variantes

### Componente ProposalComparisonTable

- [x] **Implementar `ProposalComparisonTable`** em `src/components/chat/ProposalComparisonTable.tsx`: (1) renderizada quando `proposals.length >= 2`; (2) colunas: Proposta (número sequencial), Valor Proposto (`R$ X.XXX,XX`), Retorno Líquido Estimado (formatado, verde/vermelho), Variação (% em relação ao tabela), Prazo de Resposta (`expires_at`); (3) linha com maior `net_return_estimated` destacada (background diferenciado); (4) identidade do Cessionário **nunca exibida** — sem coluna de nome ou ID; (5) `disclaimer` exibido abaixo da tabela: texto exato de D01 RN-DCE-015
  - Validação: tabela visível apenas com 2+ propostas; linha de maior retorno destacada; sem coluna de identidade; `disclaimer` exibido; valores em locale pt-BR

### Modal de Confirmação de Aceite

- [x] **Implementar `AcceptProposalModal`** em `src/components/chat/AcceptProposalModal.tsx`: (1) exibe: valor da proposta aceita (`R$ X.XXX,XX`), retorno líquido estimado, próximos passos: "O comprador tem 10 dias úteis para depositar o Escrow. Você receberá uma notificação quando o depósito for confirmado."; (2) botões: "Cancelar" (fecha modal) e "Aceitar proposta" (envia `{ confirmation: true }` e fecha); (3) loading state durante request; (4) erro se API retornar 4XX: exibir mensagem correspondente (`DCE-PROP-4220_001` → "Proposta expirada. Não é possível aceitá-la.", `DCE-PROP-4220_002` → "Dossiê incompleto. Envie todos os documentos antes de aceitar."); (5) acessibilidade: `role="dialog"`, `aria-modal="true"`, focus trap
  - Validação: texto "10 dias úteis" exato conforme D01 RN-DCE-017; `disclaimer` de retorno estimado presente; erro 422 `DCE-PROP-4220_002` exibe mensagem de dossiê; botão "Cancelar" não chama API

---

## 🔀 Cross-Módulo

- `ProposalService.acceptProposal()` cria `EscrowTransaction` com `status = 'AWAITING_DEPOSIT'` — `EscrowModule` (S7) deve existir e ser injetável; registrar dependência: S6 **cria** o escrow, S7 **gerencia** o ciclo de vida
- `ProposalService.rejectProposal()` chama `OpportunityStateMachine.transition(IN_NEGOTIATION, PUBLISHED)` de S4 — importar `OpportunityModule` em `ProposalModule`
- `ProposalService.acceptProposal()` chama `OpportunityStateMachine.transition(PUBLISHED, IN_NEGOTIATION)` de S4 — mesma dependência
- Notificação "Nova proposta recebida" disparada quando proposta é criada (pelo Cessionário, fora do escopo desta sprint) — S8 (Notificações) consome evento `proposal.created` do RabbitMQ
- Notificação "Proposta próxima do vencimento (24h)" disparada por job agendado — registrar como pendência em S8
- `DossierService.isDossierComplete()` chamado em S6 → importar `DossierModule` em `ProposalModule`

---

## 🧪 TESTES

- [x] **Criar `src/modules/proposal/proposal.state-machine.spec.ts`**: (1) `canTransition(IN_ANALYSIS, ACCEPTED)` → `true`; (2) `canTransition(IN_ANALYSIS, REJECTED)` → `true`; (3) `canTransition(IN_ANALYSIS, COUNTER_SENT)` → `true`; (4) `canTransition(RECEIVED, IN_ANALYSIS)` → `true`; (5) `canTransition(RECEIVED, EXPIRED)` → `true`; (6) `canTransition(ACCEPTED, REJECTED)` → `false` (409); (7) `canTransition(EXPIRED, ACCEPTED)` → `false`; (8) `isRespondable(RECEIVED)` → `true`; (9) `isRespondable(IN_ANALYSIS)` → `true`; (10) `isRespondable(ACCEPTED)` → `false`
  - Validação: 10 cenários; nenhuma dependência de banco; estado machine puro

- [x] **Criar `src/modules/proposal/proposal.service.spec.ts`** com testes unitários: (1) `acceptProposal` com `confirmation = false` → lança `DCE-PROP-4000_001`; (2) `acceptProposal` com proposta expirada (`expires_at < now()`) → lança `DCE-PROP-4220_001`; (3) `acceptProposal` com dossiê incompleto → lança `DCE-PROP-4220_002`; (4) `acceptProposal` válido → proposta `ACCEPTED` + `EscrowTransaction` criada com `deposit_deadline = now() + 10 dias úteis`; (5) `rejectProposal` → proposta `REJECTED` + oportunidade `PUBLISHED`; (6) `sendCounter` com `counter_value = 0` → lança `DCE-PROP-4000_002`; (7) `sendCounter` válido → proposta `COUNTER_SENT`; (8) `findAll` nunca retorna `cessionario_id`; (9) `addBusinessDays(Monday, 10)` → data correta excluindo fins de semana
  - Validação: 9 cenários; mocks do Prisma, `OpportunityStateMachine`, `DossierService`; verificação explícita de ausência de `cessionario_id` no output

- [x] **Criar `test/proposal.e2e-spec.ts`** com Supertest: (1) `GET /proposals` com token válido → 200 sem `cessionario_id`; (2) `POST /accept` com `{ confirmation: true }` → 200 com escrow; (3) `POST /accept` com `{ confirmation: false }` → 400 `DCE-PROP-4000_001`; (4) `POST /accept` de proposta expirada → 422 `DCE-PROP-4220_001`; (5) `POST /accept` com dossiê incompleto → 422 `DCE-PROP-4220_002`; (6) `POST /reject` → 200 + oportunidade voltou para PUBLISHED; (7) `POST /counter` com `counter_value = 0` → 400 `DCE-PROP-4000_002`
  - Validação: 7 cenários E2E; resposta de `cessionario_id` verificada por ausência de campo no JSON; estados de oportunidade verificados após reject

---

## 🔍 AUTO-VERIFICAÇÃO S6

| Check                  | Critério                                                                                                                                                                                                                                                                           | Status |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura        | `ProposalService`, `ProposalController`, `ProposalStateMachine` — nomes exatos; tabela `proposals` conforme D12/D13; enum `ProposalStatus` conforme D13                                                                                                                            | [x]    |
| #2 Valores numéricos   | `deposit_deadline = now() + 10 dias úteis`; `reason` max 500 chars; `message` max 1000 chars; `counter_value` e `proposal_value` em `Decimal(12,2)`                                                                                                                                | [x]    |
| #3 Error codes         | `DCE-PROP-4000_001` (confirmation), `DCE-PROP-4000_002` (counter_value), `DCE-PROP-4030_001` (permissão), `DCE-PROP-4040_001` (não encontrado), `DCE-PROP-4090_001` (já respondida), `DCE-PROP-4220_001` (expirada), `DCE-PROP-4220_002` (dossiê incompleto) — todos implementados | [x]    |
| #4 Estados             | 6 estados: `RECEIVED`, `IN_ANALYSIS`, `ACCEPTED`, `REJECTED`, `COUNTER_SENT`, `EXPIRED`; 5 transições válidas implementadas                                                                                                                                                        | [x]    |
| #5 Privacidade         | `cessionario_id`, nome, CPF do Cessionário **nunca** presentes em nenhum response do Cedente; SELECT explícito sem `cessionario_id`                                                                                                                                                | [x]    |
| #6 Confirmação         | `POST /accept` exige `{ "confirmation": true }` (booleano explícito); `false` ou ausente → 400                                                                                                                                                                                     | [x]    |
| #7 Escrow automático   | Aceite cria `EscrowTransaction` com `AWAITING_DEPOSIT` + `deposit_deadline = now() + 10 dias úteis` na mesma transação                                                                                                                                                             | [x]    |
| #8 Retorno líquido     | `net_return_estimated = proposal_value - saldo_devedor`; presente em `findAll` e `findOne`; disclaimer obrigatório nas simulações                                                                                                                                                  | [x]    |
| #9 Anti-scaffold       | `addBusinessDays` com lógica real (sem library, exclui sábado/domingo); `ProposalStateMachine` com 6 estados e 5 transições reais; sem stubs                                                                                                                                       | [x]    |
| #10 Efeitos colaterais | Aceite → oportunidade `IN_NEGOTIATION`; Recusa → oportunidade `PUBLISHED`; Contraproposta → oportunidade permanece `IN_NEGOTIATION`                                                                                                                                                | [x]    |
| #11 Dossiê gate        | Aceite bloqueado se `DossierService.isDossierComplete()` retorna `false`; erro 422 `DCE-PROP-4220_002`                                                                                                                                                                             | [x]    |
| #12 Cobertura REQs     | REQ-061 a REQ-075 — todos com ≥1 item no checklist                                                                                                                                                                                                                                 | [x]    |
