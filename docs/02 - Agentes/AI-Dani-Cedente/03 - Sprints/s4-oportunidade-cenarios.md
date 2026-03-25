# S4 — Oportunidade e Cenários

| Campo | Valor |
|---|---|
| **Sprint** | S4 |
| **Nome** | Oportunidade e Cenários |
| **Tipo** | Módulo Fullstack (Template B) |
| **Template** | B (Módulo Fullstack: por Feature, vertical slice Banco→Backend→Frontend→Wiring→Testes) |
| **Docs Consultados** | D01 (RN-DCE-010, RN-DCE-011, RN-DCE-012), D05.1, D05.2 (RF-DCE-001 a RF-DCE-013), D12, D13, D16, D18 |
| **Cross-cutting** | D10 (Glossário) |
| **REQs cobertos** | REQ-003, REQ-004, REQ-011, REQ-012, REQ-027, REQ-028, REQ-029, REQ-030, REQ-031, REQ-032, REQ-033, REQ-034, REQ-035, REQ-036, REQ-037, REQ-038, REQ-039, REQ-040, REQ-041, REQ-042, REQ-043, REQ-044, REQ-045 |
| **Objetivo** | CRUD de oportunidades com código OPR-XXXX-XXXX, máquina de estados (6 estados, 7 transições), cenários A/B/C/D com confidencialidade, fórmula Δ, simulação de retorno líquido (`POST /simulation/return`), endpoints `GET /opportunities`, `GET /opportunities/:id`, `GET /opportunities/:id/scenarios`, bloqueio de publicação sem KYC APROVADO, retirada com confirmação modal, testes com cobertura ≥80% |

---

## Critério de Conclusão da S4

Ao final desta sprint:
- `GET /opportunities` retorna apenas as oportunidades do Cedente autenticado (RLS ativo)
- `GET /opportunities/:opportunity_id` retorna campos `opr_code`, `delta_calculated`, `scenario_chosen`, `status`, `dossier_summary`
- `GET /opportunities/:opportunity_id/scenarios` retorna cenários A/B/C/D com `scenario_type`, `repasse_value`, `net_return_estimated`, `payment_conditions` — nunca exposto ao Cessionário
- `POST /simulation/return` calcula `net_return = proposal_value − saldo_devedor` com aviso obrigatório
- Tentativa de publicar oportunidade com `kyc_status != APPROVED` retorna 422 com `DCE-OPP-4220_001`
- Código OPR-XXXX-XXXX gerado automaticamente ao criar oportunidade (único, formato exato)
- Transição `Publicada → EncerradaRetirada` requer confirmação modal + proposta ativa bloqueia retirada
- Cenários A/B/C/D retornados zero para Cessionário — RLS impede acesso cruzado
- `delta_calculated = tabela_atual − tabela_contrato`; se `delta_calculated ≤ 0` → fallback: `comissao = 20% × valor_pago_cedente`

---

## ⚙️ FEATURE 1 — Cadastro da Oportunidade

### Banco

- [x] **Verificar que migration da tabela `opportunities`** já existe no S1 com os campos: `id UUID PK`, `cedente_id UUID FK → cedente_profiles.id`, `opr_code VARCHAR(14) UNIQUE NOT NULL`, `nome_empreendimento TEXT NOT NULL`, `incorporadora TEXT NOT NULL`, `endereco TEXT NOT NULL`, `tipologia TEXT NOT NULL`, `area_m2 DECIMAL(8,2) NOT NULL`, `tabela_contrato DECIMAL(12,2) NOT NULL`, `valor_pago_cedente DECIMAL(12,2) NOT NULL`, `parcelas_restantes INT NOT NULL`, `tabela_atual DECIMAL(12,2) NOT NULL`, `saldo_devedor DECIMAL(12,2) NOT NULL`, `delta_calculated DECIMAL(12,2) NOT NULL`, `status OpportunityStatus NOT NULL DEFAULT 'DRAFT'`, `scenario_chosen ScenarioType`, `published_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ`
  - Validação: `\d opportunities` em psql mostra todos os campos acima; `opr_code` tem index `UNIQUE`; `deleted_at` existe para soft delete; todos os monetários são `DECIMAL(12,2)` — sem `FLOAT`
- [x] **Verificar que tabela `opportunity_scenarios`** já existe com campos: `id UUID PK`, `opportunity_id UUID FK → opportunities.id`, `scenario_type ScenarioType NOT NULL` (enum: A, B, C, D), `repasse_value DECIMAL(12,2) NOT NULL`, `net_return_estimated DECIMAL(12,2) NOT NULL`, `payment_conditions TEXT NOT NULL`, `created_at TIMESTAMPTZ DEFAULT now()`
  - Validação: 4 linhas por `opportunity_id` (uma por type A/B/C/D); FK com ON DELETE CASCADE; enum `ScenarioType` tem exatamente os valores 'A', 'B', 'C', 'D'
- [x] **Verificar RLS em `opportunities`**: policy `cedente_isolation` — `USING (cedente_id = current_setting('app.current_cedente_id')::uuid)` ativo para SELECT, UPDATE, DELETE; INSERT permite apenas `cedente_id` igual ao do JWT
  - Validação: query `SELECT * FROM opportunities WHERE cedente_id != current_cedente_id` retorna zero linhas; Cedente B não vê oportunidade do Cedente A em nenhum cenário
- [x] **Verificar RLS em `opportunity_scenarios`**: policy via JOIN — `USING (opportunity_id IN (SELECT id FROM opportunities WHERE cedente_id = current_setting('app.current_cedente_id')::uuid))`
  - Validação: `GET /opportunities/:id/scenarios` com token de outro Cedente retorna 403; cenários nunca aparecem em queries cruzadas

### Backend — OpportunityService

- [x] **Implementar `OpportunityService.findAll(cedenteId: string)`** em `src/modules/opportunity/opportunity.service.ts`: (1) consulta `this.prisma.opportunity.findMany({ where: { cedente_id: cedenteId, deleted_at: null } })`; (2) retorna array de oportunidades ordenado por `created_at DESC`; (3) inclui `dossier_summary` calculado como sub-query `{ total: count(docs), complete: count(approved) === count(total), documents_approved: N, documents_pending: N, documents_rejected: N }`
  - Validação: query nunca retorna oportunidades de outro `cedente_id`; campo `dossier_summary` calculado corretamente; `deleted_at IS NOT NULL` excluído
- [x] **Implementar `OpportunityService.findOne(id: string, cedenteId: string)`**: (1) busca `WHERE id = id AND cedente_id = cedenteId AND deleted_at IS NULL`; (2) inclui `dossier_summary`; (3) inclui `scenario_chosen` (campo direto no model, não os detalhes dos cenários); (4) lança `DaniNotFoundError('DCE-OPP-4040_001')` se não encontrado; (5) lança `DaniForbiddenError('DCE-OPP-4030_001')` se `cedente_id` diverge
  - Validação: ID de outro Cedente lança 403; ID inexistente lança 404; response contém `delta_calculated` calculado (não nulo)
- [x] **Implementar `OpportunityService.findScenarios(opportunityId: string, cedenteId: string)`**: (1) verifica ownership da oportunidade antes de retornar cenários; (2) busca `opportunity_scenarios WHERE opportunity_id = opportunityId`; (3) retorna array com 4 objetos `{ id, scenario_type, repasse_value, net_return_estimated, payment_conditions }`; (4) **nunca retornado para Cessionário** — verificado via role do JWT
  - Validação: token com role `cessionario` (ou qualquer role ≠ `cedente`/`admin`) retorna 403; 4 cenários retornados em ordem A→B→C→D; `repasse_value` e `net_return_estimated` são `Decimal` — sem `Number` float
- [x] **Implementar `generateOprCode()`** em `src/modules/opportunity/opportunity.service.ts`: (1) gera código no formato `OPR-XXXX-XXXX` onde X = dígito numérico aleatório; (2) verifica unicidade em banco antes de retornar (`SELECT 1 FROM opportunities WHERE opr_code = code`); (3) retry automático até 5 tentativas se colisão; (4) lança erro interno se 5 colisões consecutivas
  - Validação: 100 códigos gerados não têm colisão; formato `OPR-\d{4}-\d{4}` validado por regex; unicidade garantida com check no banco

### Backend — OpportunityController

- [x] **Implementar `GET /api/v1/opportunities`** em `OpportunityController`: (1) aplica `JwtAuthGuard` + `CedenteIsolationMiddleware`; (2) extrai `cedente_id` via `@CurrentCedente()`; (3) chama `OpportunityService.findAll(cedenteId)`; (4) retorna `{ data: [...], meta: { page, per_page, total, total_pages } }` com paginação `page`/`per_page`/`sort`/`order`; (5) suporta query params: `?status=PUBLISHED&page=1&per_page=20&sort=created_at&order=desc`
  - Validação: token válido → 200 com lista isolada; token sem oportunidades → 200 com `data: []`; sem token → 401 `DCE-AUTH-4010_003`
- [x] **Implementar `GET /api/v1/opportunities/:opportunity_id`** em `OpportunityController`: (1) valida UUID do path param; (2) chama `OpportunityService.findOne(id, cedenteId)`; (3) retorna `{ data: { id, opr_code, nome_empreendimento, incorporadora, endereco, tipologia, area_m2, tabela_contrato, tabela_atual, valor_pago_cedente, saldo_devedor, delta_calculated, scenario_chosen, status, published_at, dossier_summary, created_at, updated_at } }`
  - Validação: `cedente_id` não aparece no response (extraído do JWT); `delta_calculated` sempre presente; `scenario_chosen` pode ser `null`; outro Cedente → 403 `DCE-OPP-4030_001`
- [x] **Implementar `GET /api/v1/opportunities/:opportunity_id/scenarios`** em `OpportunityController`: (1) verifica ownership; (2) chama `OpportunityService.findScenarios(id, cedenteId)`; (3) retorna `{ data: [{ id, scenario_type, repasse_value, net_return_estimated, payment_conditions }] }` com 4 elementos
  - Validação: response tem exatamente 4 elementos; `scenario_type` values: `'A'`, `'B'`, `'C'`, `'D'`; request de Cessionário (role ≠ cedente/admin) → 403; `repasse_value` e `net_return_estimated` são decimais formatados com 2 casas

### Backend — Validação de KYC para Publicação

- [x] **Implementar validação de KYC** em `OpportunityService` antes de qualquer transição para `AWAITING_VALIDATION`: (1) busca `CedenteProfile WHERE id = cedenteId`; (2) se `kyc_status != 'APPROVED'` → lança `DaniBusinessError('DCE-OPP-4220_001', 'KYC não aprovado. Conclua a verificação de identidade para publicar sua oportunidade.')` com HTTP 422; (3) se `kyc_status == 'APPROVED'` → permite a transição
  - Validação: Cedente com `kyc_status = 'PENDING'` tenta publicar → 422 com `DCE-OPP-4220_001`; Cedente com `kyc_status = 'APPROVED'` → transição permitida; teste verifica os 3 status (PENDING, APPROVED, REJECTED)

### Backend — Máquina de Estados da Oportunidade

- [x] **Implementar `OpportunityStateMachine`** em `src/modules/opportunity/opportunity.state-machine.ts`: (1) define 6 estados válidos: `DRAFT`, `AWAITING_VALIDATION`, `PUBLISHED`, `IN_NEGOTIATION`, `CLOSED_SOLD`, `CLOSED_WITHDRAWN`; (2) define 7 transições permitidas: `DRAFT→AWAITING_VALIDATION` (dossiê + cenário escolhido), `AWAITING_VALIDATION→DRAFT` (Admin rejeita), `AWAITING_VALIDATION→PUBLISHED` (Admin aprova), `PUBLISHED→IN_NEGOTIATION` (proposta ativa recebida), `IN_NEGOTIATION→PUBLISHED` (proposta recusada/expirada), `IN_NEGOTIATION→CLOSED_SOLD` (Escrow liberado), `PUBLISHED→CLOSED_WITHDRAWN` (Cedente retira); (3) método `canTransition(from: OpportunityStatus, to: OpportunityStatus): boolean`; (4) lança `DaniBusinessError('DCE-OPP-4220_002')` se transição inválida
  - Validação: todas as 7 transições válidas retornam `true`; transição `DRAFT→PUBLISHED` direta retorna `false` e lança erro; `CLOSED_SOLD→DRAFT` retorna `false`; testes cobrem todas as transições válidas e inválidas

- [x] **Implementar lógica de retirada `withdrawOpportunity(id, cedenteId)`** em `OpportunityService`: (1) busca oportunidade e verifica ownership; (2) se `status == 'IN_NEGOTIATION'` → lança `DaniBusinessError('DCE-OPP-4220_003', 'Sua oportunidade tem uma proposta ativa no momento. Para retirá-la, você precisa primeiro recusar a proposta em andamento.')` com HTTP 422; (3) se `status == 'PUBLISHED'` → transiciona para `CLOSED_WITHDRAWN` via `OpportunityStateMachine`; (4) registra `deleted_at = now()` (soft delete)
  - Validação: retirada com proposta ativa → 422 `DCE-OPP-4220_003`; retirada sem proposta → 200 com `status: 'CLOSED_WITHDRAWN'`; `deleted_at` preenchido; oportunidade não aparece em `GET /opportunities` após retirada

---

## ⚙️ FEATURE 2 — Cálculo Δ e Simulação de Retorno

### Backend — SimulationService

- [x] **Implementar `SimulationService.calculateReturn(dto: SimulationReturnDto)`** em `src/modules/simulation/simulation.service.ts`: (1) busca `opportunity WHERE id = dto.opportunity_id AND cedente_id = cedenteId AND deleted_at IS NULL`; (2) extrai `saldo_devedor` da oportunidade; (3) calcula `net_return = dto.proposal_value − saldo_devedor`; (4) retorna `{ opportunity_id, proposal_value, saldo_devedor, net_return, disclaimer: "Este é um valor estimado com base nos dados disponíveis. Deduções adicionais (impostos, taxas notariais) podem variar. Consulte um especialista para cálculo definitivo." }`; (5) lança `DaniBusinessError('DCE-SIM-4000_001')` se `proposal_value ≤ 0`; (6) lança `DaniForbiddenError('DCE-SIM-4030_001')` se oportunidade de outro Cedente; (7) lança `DaniNotFoundError('DCE-SIM-4040_001')` se oportunidade não encontrada
  - Validação: `net_return` pode ser negativo (caso proposta < saldo devedor); `disclaimer` sempre presente no response; `proposal_value = 0` → 400 `DCE-SIM-4000_001`; todos os campos monetários são `Decimal(12,2)` — sem `Float`

- [x] **Implementar cálculo Δ em `OpportunityService.create()`**: (1) `delta_calculated = tabela_atual - tabela_contrato`; (2) se `delta_calculated > 0` → `comissao_base = delta_calculated`; (3) se `delta_calculated ≤ 0` → fallback: `comissao_base = valor_pago_cedente`; (4) `comissao_cessionario = comissao_base × 0.20`; (5) armazenar `delta_calculated` calculado no momento do cadastro; (6) método `recalculateDelta(opportunity)` chamado se `tabela_atual` for atualizada
  - Validação: `tabela_atual = 280000`, `tabela_contrato = 220000` → `delta_calculated = 60000`, `comissao = 12000`; `tabela_atual = 200000`, `tabela_contrato = 220000` → `delta_calculated = -20000`, fallback ativo; teste cobre ambos os casos (delta > 0 e delta ≤ 0)

### Backend — SimulationController

- [x] **Implementar `POST /api/v1/simulation/return`** em `SimulationController`: (1) aplica `JwtAuthGuard` + `CedenteIsolationMiddleware`; (2) valida body com `SimulationReturnDto`: `{ opportunity_id: UUID, proposal_value: Decimal > 0 }`; (3) chama `SimulationService.calculateReturn(dto, cedenteId)`; (4) retorna `{ data: { opportunity_id, proposal_value, saldo_devedor, net_return, disclaimer } }` com HTTP 200
  - Validação: body válido → 200 com `net_return` e `disclaimer`; `proposal_value = -1` → 400; `opportunity_id` de outro Cedente → 403 `DCE-SIM-4030_001`; `opportunity_id` inexistente → 404 `DCE-SIM-4040_001`

- [x] **Criar `src/modules/simulation/dto/simulation-return.dto.ts`** com: (1) `opportunity_id: string` com decorator `@IsUUID()`; (2) `proposal_value: number` com decorators `@IsPositive()` e `@IsNotEmpty()`; (3) transform via `@Type(() => Number)`; (4) validação de `Decimal` ≥ 0.01
  - Validação: body `{ "opportunity_id": "invalid", "proposal_value": -5 }` retorna 400 com campos de erro por field; corpo vazio retorna 400

---

## ⚙️ FEATURE 3 — Primeiro Uso e Boas-Vindas

### Backend — ChatSessionService (extensão S3)

- [x] **Verificar que `POST /chat/sessions` já suporta leitura de KYC e oportunidade** para retorno de `welcome_context`: (1) ao criar sessão sem `opportunity_id` (PE-1), o serviço busca `CedenteProfile.kyc_status` e `Opportunity.count(cedente_id, status != CLOSED)`; (2) retorna `welcome_context` no response com campos `{ kyc_approved: boolean, has_active_opportunity: boolean, welcome_type: 'FULL' | 'KYC_PENDING' | 'NO_OPPORTUNITY' }`; (3) `welcome_type = 'FULL'` se KYC aprovado e oportunidade ativa; `'KYC_PENDING'` se KYC != APPROVED; `'NO_OPPORTUNITY'` se sem oportunidade
  - Validação: KYC aprovado + oportunidade ativa → `welcome_type: 'FULL'`; KYC pendente → `welcome_type: 'KYC_PENDING'`; sem oportunidade → `welcome_type: 'NO_OPPORTUNITY'`; response contém exatamente 3 campos do `welcome_context`

- [x] **Verificar que PE-2 (entry_point = OPPORTUNITY) carrega contexto da oportunidade**: ao criar sessão com `entry_point = 'OPPORTUNITY'` e `opportunity_id` válido: (1) valida ownership da oportunidade antes de persistir a sessão; (2) armazena `opportunity_id` na sessão; (3) pré-carrega dados da oportunidade no contexto inicial do agente (`opr_code`, `delta_calculated`, `scenario_chosen`, `status`) sem exibir ao Cedente
  - Validação: `opportunity_id` de outro Cedente → 404 `DCE-CHAT-4040_001`; sessão criada com PE-2 tem `opportunity_id` preenchido; contexto pré-carregado nunca aparece nas mensagens visíveis ao Cedente

---

## 🖥️ FRONTEND (Widget Chat — integração de contexto)

### Tela T-02: Chat por Oportunidade (PE-2)

- [x] **Implementar carregamento de contexto de oportunidade no ChatWidget** (T-02): (1) ao receber `entry_point = 'OPPORTUNITY'` com `opportunity_id`, o widget exibe header com `opr_code` e badge de `status`; (2) header não exibe cenários ou valores — apenas código e status; (3) estado **Skeleton**: 3 linhas de loading (OPR placeholder, status placeholder, chat placeholder) enquanto sessão carrega; (4) estado **Error**: "Não foi possível carregar a oportunidade. [Tentar novamente]" com retry; (5) estado **Empty**: não aplicável — PE-2 pressupõe oportunidade existente; (6) estado **Populated**: header com `OPR-XXXX-XXXX` + `StatusBadge` + histórico de mensagens
  - Validação: `opr_code` exibido no header; `status` exibido via `StatusBadge`; nenhum valor financeiro exposto no header; estado Skeleton visível por ≥200ms em rede lenta (simulação)

- [x] **Implementar sugestões de conversa (ConversationStarters)** no widget: (1) exibir 4 botões clicáveis quando chat abre sem histórico e sem `opportunity_id` (PE-1): "Qual o retorno esperado para a minha oportunidade?", "Tenho uma proposta recebida. Vale a pena aceitar?", "O que ainda falta no meu dossiê?", "Quanto tempo demora para concluir o repasse?"; (2) ao clicar: inserir texto no campo de mensagem e enviar automaticamente; (3) sugestões desaparecem após primeiro envio de mensagem; (4) sugestões NÃO aparecem quando `opportunity_id` ou `proposal_id` estão presentes no contexto
  - Validação: 4 sugestões exibidas em PE-1 sem histórico; clique em sugestão envia mensagem com texto exato; sugestões ausentes quando PE-2 ou PE-3; sugestões ausentes após primeira mensagem enviada

### Componente OpportunityStatusBadge

- [x] **Implementar `OpportunityStatusBadge`** em `src/components/chat/OpportunityStatusBadge.tsx`: (1) mapeamento de `OpportunityStatus` → label PT-BR → cor: `DRAFT → "Rascunho" → cor neutra`, `AWAITING_VALIDATION → "Aguardando validação" → cor amarela`, `PUBLISHED → "Publicada" → cor verde`, `IN_NEGOTIATION → "Em negociação" → cor azul`, `CLOSED_SOLD → "Encerrada — Vendida" → cor verde escuro`, `CLOSED_WITHDRAWN → "Encerrada — Retirada" → cor cinza`; (2) props: `status: OpportunityStatus`; (3) usa variantes do design system
  - Validação: 6 status mapeados; labels em PT-BR exatos conforme D01/D05.2; nenhum status retorna "undefined" ou label vazio; snapshot test cobre 6 variantes

### Componente ConfirmationModal para Retirada

- [x] **Implementar modal de confirmação de retirada de oportunidade** em `src/components/chat/WithdrawOpportunityModal.tsx`: (1) exibe texto exato: "Ao retirar sua oportunidade, ela não estará mais visível para compradores. Você pode republicá-la depois. Deseja continuar?"; (2) botões: "Cancelar" (fecha modal, não faz ação) e "Retirar oportunidade" (chama API e fecha modal); (3) estado loading no botão de confirmação durante chamada à API; (4) estado de erro se API falhar: "Não foi possível retirar a oportunidade. Tente novamente."; (5) acessibilidade: `role="dialog"`, `aria-modal="true"`, focus trap ativo
  - Validação: texto exato conforme D01 RN-DCE-012; botão "Cancelar" fecha sem chamar API; botão "Retirar oportunidade" chama `DELETE /opportunities/:id` ou endpoint equivalente; loading state visível durante request; erro exibido se 422 recebido

### Componente SimulationCard

- [x] **Implementar `SimulationCard`** em `src/components/chat/SimulationCard.tsx`: (1) exibe resultado de simulação de retorno líquido com campos: `proposal_value` (valor da proposta), `saldo_devedor` (saldo devedor), `net_return` (retorno líquido — pode ser negativo); (2) se `net_return < 0`: exibe em vermelho com label "Retorno negativo"; (3) `disclaimer` sempre exibido abaixo em fonte menor: texto exato de D01 RN-DCE-015: "Este é um valor estimado com base nos dados disponíveis. Deduções adicionais (impostos, taxas notariais) podem variar. Consulte um especialista para cálculo definitivo."; (4) todos os valores monetários formatados em `R$ X.XXX,XX` (locale pt-BR); (5) props: `SimulationResult` (tipado conforme response da API)
  - Validação: `net_return = -67000` exibido em vermelho; disclaimer exato conforme D01; valores em locale pt-BR; snapshot test cobre caso positivo e negativo

---

## 🔀 Cross-Módulo

- `OpportunityStateMachine.transition(PUBLISHED → IN_NEGOTIATION)` é disparado pelo módulo de Propostas (S6) quando proposta ativa é recebida — registrar como dependência externa nesta sprint
- `OpportunityStateMachine.transition(IN_NEGOTIATION → CLOSED_SOLD)` é disparado pelo módulo de Escrow (S7) quando Escrow é liberado — registrar como dependência externa nesta sprint
- `OpportunityStateMachine.transition(IN_NEGOTIATION → PUBLISHED)` é disparado pelo módulo de Propostas (S6) quando proposta é recusada/expirada — registrar como dependência externa
- `SimulationService.calculateReturn()` também é chamado pela ferramenta `simulate-return` do agente (S3) — serviço já deve ser injetável via `SimulationModule`
- Bloqueio por KYC (`kyc_status != APPROVED`) compartilhado com S5 (Dossiê) para publicação — validação centralizada em `OpportunityService`, não duplicada no DossierService

---

## 🧪 TESTES

- [x] **Criar `src/modules/opportunity/opportunity.service.spec.ts`** com testes unitários: (1) `findAll` retorna apenas oportunidades do `cedente_id` correto; (2) `findAll` exclui `deleted_at IS NOT NULL`; (3) `findOne` com ID de outro Cedente → lança `DaniForbiddenError('DCE-OPP-4030_001')`; (4) `findOne` com ID inexistente → lança `DaniNotFoundError('DCE-OPP-4040_001')`; (5) `generateOprCode` gera formato `OPR-XXXX-XXXX` (regex `^OPR-\d{4}-\d{4}$`); (6) `generateOprCode` retenta em colisão (mock: primeira tentativa colide, segunda é única); (7) `withdrawOpportunity` com `IN_NEGOTIATION` → lança `DaniBusinessError('DCE-OPP-4220_003')`; (8) `withdrawOpportunity` com `PUBLISHED` → retorna status `CLOSED_WITHDRAWN`
  - Validação: 8 cenários cobertos; mock do Prisma sem banco real; nenhuma chamada HTTP real

- [x] **Criar `src/modules/opportunity/opportunity.state-machine.spec.ts`** com testes: (1) `canTransition(DRAFT, AWAITING_VALIDATION)` → `true`; (2) `canTransition(AWAITING_VALIDATION, DRAFT)` → `true`; (3) `canTransition(AWAITING_VALIDATION, PUBLISHED)` → `true`; (4) `canTransition(PUBLISHED, IN_NEGOTIATION)` → `true`; (5) `canTransition(IN_NEGOTIATION, PUBLISHED)` → `true`; (6) `canTransition(IN_NEGOTIATION, CLOSED_SOLD)` → `true`; (7) `canTransition(PUBLISHED, CLOSED_WITHDRAWN)` → `true`; (8) `canTransition(DRAFT, PUBLISHED)` → `false`; (9) `canTransition(CLOSED_SOLD, DRAFT)` → `false`; (10) `canTransition(CLOSED_WITHDRAWN, PUBLISHED)` → `false`
  - Validação: 10 cenários; 7 transições válidas e 3 inválidas confirmadas; nenhuma dependência de banco

- [x] **Criar `src/modules/simulation/simulation.service.spec.ts`** com testes: (1) `calculateReturn` com `tabela_atual = 280000`, `tabela_contrato = 220000` → `delta_calculated = 60000`; (2) `calculateReturn` com delta ≤ 0 → fallback `comissao = 20% × valor_pago_cedente`; (3) `calculateReturn` com `proposal_value = 158000`, `saldo_devedor = 225000` → `net_return = -67000`; (4) `calculateReturn` com `proposal_value = 0` → lança `DaniBusinessError('DCE-SIM-4000_001')`; (5) `calculateReturn` com opportunity de outro Cedente → lança `DaniForbiddenError('DCE-SIM-4030_001')`; (6) response sempre contém `disclaimer` com texto exato de D01 RN-DCE-015
  - Validação: 6 cenários; `disclaimer` verificado por igualdade de string exata; valores negativos aceitos em `net_return`

- [x] **Criar `test/opportunity.e2e-spec.ts`** com Supertest: (1) `GET /opportunities` com token válido → 200 com lista; (2) `GET /opportunities` sem token → 401 `DCE-AUTH-4010_003`; (3) `GET /opportunities/:id` de outro Cedente → 403 `DCE-OPP-4030_001`; (4) `GET /opportunities/:id/scenarios` → 200 com 4 cenários; (5) `POST /simulation/return` válido → 200 com `net_return` e `disclaimer`; (6) `POST /simulation/return` com `proposal_value ≤ 0` → 400 `DCE-SIM-4000_001`; (7) Cedente com `kyc_status = PENDING` acessa publicação → 422 `DCE-OPP-4220_001`
  - Validação: 7 cenários com banco de teste; RLS verificado nos cenários de isolamento; `disclaimer` presente em todo response de simulação

---

## 🔍 AUTO-VERIFICAÇÃO S4

| Check | Critério | Status |
|---|---|---|
| #1 Nomenclatura | `OpportunityService`, `OpportunityController`, `SimulationService`, `SimulationController`, `OpportunityStateMachine` — nomes exatos; tabelas `opportunities` e `opportunity_scenarios` conforme D12/D13 | [x] |
| #2 Valores numéricos | `delta_calculated = tabela_atual − tabela_contrato`; comissão = 20% × Δ (ou 20% × valor_pago_cedente se Δ ≤ 0); `net_return = proposal_value − saldo_devedor`; todos monetários em `Decimal(12,2)` | [x] |
| #3 Error codes | `DCE-OPP-4030_001` (permissão), `DCE-OPP-4040_001` (não encontrado), `DCE-OPP-4220_001` (KYC), `DCE-OPP-4220_002` (transição inválida), `DCE-OPP-4220_003` (proposta ativa), `DCE-SIM-4000_001` (valor ≤ 0), `DCE-SIM-4030_001` (permissão), `DCE-SIM-4040_001` (não encontrado) — todos implementados | [x] |
| #4 Estados | 6 estados: `DRAFT`, `AWAITING_VALIDATION`, `PUBLISHED`, `IN_NEGOTIATION`, `CLOSED_SOLD`, `CLOSED_WITHDRAWN`; 7 transições válidas documentadas e implementadas | [x] |
| #5 Confidencialidade | Cenários A/B/C/D nunca retornados para Cessionário; `scenario_chosen` sem detalhes de valores na listagem; RLS impede acesso cruzado | [x] |
| #6 OPR Code | Formato exato `OPR-XXXX-XXXX` (8 dígitos em 2 grupos de 4); único no banco; gerado automaticamente ao criar oportunidade | [x] |
| #7 KYC Gate | Publicação bloqueada para `kyc_status != APPROVED`; erro 422 `DCE-OPP-4220_001` com mensagem de direcionamento | [x] |
| #8 Disclaimer | Simulação sempre retorna `disclaimer` com texto exato: "Este é um valor estimado com base nos dados disponíveis. Deduções adicionais (impostos, taxas notariais) podem variar. Consulte um especialista para cálculo definitivo." | [x] |
| #9 Soft Delete | `deleted_at` presente em `opportunities`; `findAll` e `findOne` excluem `deleted_at IS NOT NULL`; `withdrawOpportunity` preenche `deleted_at` | [x] |
| #10 Isolamento | `cedente_id` sempre do JWT; RLS ativo em `opportunities` e `opportunity_scenarios`; nenhum campo `cedente_id` exposto em responses | [x] |
| #11 Anti-scaffold | `OpportunityStateMachine` com lógica real de 7 transições; `generateOprCode` com retry e verificação de unicidade; `SimulationService` com fórmula implementada; nenhum stub vazio | [x] |
| #12 Cobertura REQs | REQ-003, REQ-004, REQ-011, REQ-012, REQ-027–REQ-045 — todos com ≥1 item no checklist | [x] |
