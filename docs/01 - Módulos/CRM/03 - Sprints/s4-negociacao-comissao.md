# S4 — Negociação e Comissão

## CRM Repasse Seguro

| **Campo**         | **Valor**                                                                                                                                                                                                  |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**        | S4                                                                                                                                                                                                         |
| **Nome**          | Negociação e Comissão                                                                                                                                                                                      |
| **Template**      | B — Módulo Fullstack                                                                                                                                                                                       |
| **Docs fonte**    | 01.2, 05.2, 05.3, 06, 16, 27                                                                                                                                                                               |
| **REQs cobertos** | REQ-044 a REQ-056, REQ-136, REQ-138, REQ-149 a REQ-153, REQ-215 a REQ-222, REQ-254, REQ-256, REQ-282 (parcial)                                                                                             |
| **Objetivo**      | Analista RS registra Match, proposta, contraproposta, aceite e encerramento de Negociação. Cálculo automático de comissão estimada (Cedente + Cessionário) com regras de desconto e Fechamento definitivo. |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — `proposals`, `commissions`, `negotiations`, `NegotiationState`, `CommissionState`, enums exatos
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: Valores exatos — 48h validade proposta, alerta >5 rodadas, 20% comissão Δ, 20% comissão Cedente B/C/D, faixas desconto (0%/5%/10%/15%), mínimo 100 chars cancelamento comissão
- [x] C4: Glossário — Delta (Δ), Cenário A/B/C/D, Conta Escrow, Termo Comercial, Perda Evitada cobertos
- [x] C5: TODAS as transições de NegotiationState cobertas: Em andamento → Proposta enviada → Contraproposta recebida → Aceita / Recusada / Expirada
- [x] C6: TTL de 48h para validade de proposta / expiração automática cobertos
- [x] C7: RBAC — desconto apenas Coord+Admin; cancelamento comissão apenas Admin com quatro olhos
- [x] C8: T-CRM-030/031/032/033/034 com estados de tela
- [x] C9: Regras de fallback — Δ ≤ 0 → base de cálculo alternativa
- [x] C10: Glossário — Perda Evitada, Fechamento (3 critérios), Termo Comercial cobertos
- [x] C11: Anti-scaffold — cálculos de comissão com fórmulas exatas, validações RBAC, lógica de expiração
- [x] C12: REQ-044 a REQ-056 cobertos

---

## FEATURE 1 — Negociação (Match, Propostas, Contrapropostas)

### Banco / Migrations

- [x] **REQ-136** Verificar enum `NegotiationState` no schema Prisma: `EM_ANDAMENTO`, `PROPOSTA_ENVIADA`, `CONTRAPROPOSTA_RECEBIDA`, `ACEITA`, `RECUSADA`, `EXPIRADA`. Criar tabela `negotiations` com campos: `id UUID PK`, `case_id UUID FK → cases @unique` (1 por Caso), `state NegotiationState @default(EM_ANDAMENTO)`, `cessionario_contact_id UUID FK → contacts`, `rounds_count Int @default(0)`, `final_value Decimal(15,2)?`, `accepted_at DateTime? @db.Timestamptz`, `created_by UUID FK → crm_users`, `created_at`, `updated_at`
  - Validar: `case_id @unique` (1 Negociação ativa por Caso); `NegotiationState` enum completo; migration aplicada

- [x] **REQ-138** Verificar enum `CommissionState` no schema Prisma: `ESTIMADA`, `PENDENTE_FECHAMENTO`, `CONFIRMADA`, `CANCELADA`. Verificar tabela `commissions` criada em S1 com campos: `type (CEDENTE|CESSIONARIO)`, `state CommissionState`, `base_calculation`, `delta_value Decimal(15,2)?`, `commission_value Decimal(15,2)`, `discount_percent Decimal(5,2)?`, campos de cancelamento com aprovação dupla
  - Validar: `type` enum `CEDENTE/CESSIONARIO`; migration de `commissions` aplicada

### Backend

- [x] **REQ-215** `GET /cases/:id/negotiations` — Retornar Negociação ativa do Caso (se existir) com todas as `proposals` vinculadas ordenadas por `created_at`. Incluir: estado da Negociação, Cessionário (campos mascarados para Parceiro Externo), contagem de rodadas, valor final (se aceita)
  - Validar: Caso sem Negociação → `{ data: null }`; Caso com Negociação → dados completos; Parceiro Externo → sem dados do Cessionário

- [x] **REQ-216 / REQ-044** `POST /cases/:id/negotiations` — Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS` (apenas próprios). Request body: `{ cessionario_contact_id: UUID }`. Regras:
  - Caso deve estar no estado `PUBLICACAO` ou `MATCH`
  - Caso não pode ter Negociação com state diferente de `RECUSADA` ou `EXPIRADA`
  - Se válido: criar Negociação `state = EM_ANDAMENTO`, avançar Caso para `MATCH` (se em PUBLICACAO) / `NEGOCIACAO` (se em MATCH), registrar em `case_status_history`, criar `audit_log`
  - Registrar automaticamente: data do Match, Cessionário vinculado, Analista RS
  - Validar: Caso em estado VERIFICACAO → 422; Caso com Negociação ativa → 409 CRM-022; criação válida → Negociação criada + estado do Caso atualizado

- [x] **REQ-217 / REQ-045** `POST /cases/:id/negotiations/:negId/proposal` — Request body: `{ value: Decimal, conditions?: string, type: CESSIONARIO|CEDENTE }`. Lógica:
  - Calcular `valid_until = now() + 48 horas` (parâmetro `proposal_validity_hours` em `system_configs`, default 48)
  - Se `type = CESSIONARIO` e valor abaixo do piso do Cenário do Cedente: retornar 200 com `{ warning: "BELOW_FLOOR", floor_value: N }` (não bloqueia — Analista decide)
  - Criar `proposals` com `valid_until`, atualizar `negotiations.state = PROPOSTA_ENVIADA` ou `CONTRAPROPOSTA_RECEBIDA` conforme tipo, incrementar `rounds_count`
  - Se `rounds_count > 5`: retornar 200 com `{ warning: "MANY_ROUNDS", rounds_count: N }`
  - Validar: proposta do Cessionário abaixo do piso → resposta 200 com warning BELOW_FLOOR; 6ª proposta → warning MANY_ROUNDS; `valid_until` calculado corretamente

- [x] Implementar `ProposalExpirationWorker` (cron a cada 5 minutos): buscar `proposals` com `valid_until < now()` e `status = PENDENTE`. Marcar `proposals.status = EXPIRADA`, atualizar `negotiations.state = EXPIRADA`, notificar Analista RS via fila RabbitMQ: "A negociação do Caso [RS-XXXX] foi encerrada por expiração de proposta."
  - Validar: proposta criada com `valid_until = now() - 1h` → marcada como EXPIRADA na próxima execução do worker; Analista RS recebe notificação; Caso retorna para estado PUBLICACAO (via `POST /cases/:id/advance` com lógica reversa)

- [x] **REQ-218 / REQ-047** `POST /cases/:id/negotiations/:negId/accept` — Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS`. Request body: `{ proposal_id: UUID }`. Lógica:
  - Marcar `proposals.status = ACEITA`, `negotiations.state = ACEITA`, `negotiations.accepted_at = now()`, `negotiations.final_value = proposal.value`
  - Avançar Caso para `ANUENCIA` via `caseService.advance()`
  - Calcular comissão estimada do Cessionário (REQ-049) e do Cedente (REQ-050): criar 2 registros em `commissions` com `state = ESTIMADA`
  - Retornar resumo: `{ final_value, cessionario_commission: { base, value }, cedente_commission: { base, value } }`
  - Validar: proposta aceita → estado Negociação = ACEITA; Caso avança para ANUENCIA; 2 registros de comissão ESTIMADA criados; resumo financeiro retornado

- [x] **REQ-219 / REQ-048** `POST /cases/:id/negotiations/:negId/close` — Request body: `{ reason: RECUSADA|EXPIRADA, rejected_by?: CEDENTE|CESSIONARIO }`. Lógica:
  - Marcar `negotiations.state = RECUSADA` ou `EXPIRADA`
  - Se Cessionário recusou: adicionar flag `rejection_flag = "RECUSOU_[case_id]"` no `contacts` do Cessionário
  - Retornar Caso para estado `PUBLICACAO` (se estava em NEGOCIACAO)
  - Notificar Analista RS: "A negociação do Caso [RS-XXXX] foi encerrada. O Caso voltou ao estado Publicação."
  - Validar: encerramento → Caso volta para PUBLICACAO; flag no Contato Cessionário; notificação enviada

### Frontend

- [x] **REQ-149** Implementar `app/(dashboard)/negotiations/page.tsx` — T-CRM-030 Lista de Casos em Negociação:
  - Tabela com TanStack Table: Caso (ID + empreendimento), Estado da Negociação, Cessionário (mascarado), Rodadas, Validade da proposta atual, SLA do Caso, Analista RS
  - Empty state: "Nenhuma negociação ativa." com link para Pipeline
  - Validar: Casos com proposta expirando em < 24h destacados; ordenação por validade funciona

- [x] **REQ-150** Implementar `app/(dashboard)/negotiations/[caseId]/page.tsx` — T-CRM-031 Painel do Caso em Negociação:
  - Linha do tempo de propostas/contrapropostas com valores e datas
  - Cálculo estimado de comissão (em tempo real conforme valor da proposta atual)
  - Botões: "Registrar Proposta" (abre T-CRM-032), "Registrar Contraproposta" (abre T-CRM-033), "Aceitar Proposta" (abre T-CRM-034), "Encerrar Negociação" (abre T-CRM-034)
  - Warning se proposta abaixo do piso do Cenário
  - Warning visual se > 5 rodadas
  - Validar: comissão estimada atualiza ao alterar o valor da proposta no formulário (debounced); warning de piso visível; warning de rodadas visível após 5ª

- [x] **REQ-151/152/153** Implementar modais:
  - T-CRM-032 `NewProposalModal`: campos `value (Decimal)`, `conditions (texto)`, `valid_until (calculado, exibido como informação)`. Ao submeter → `POST /cases/:id/negotiations/:negId/proposal { type: CESSIONARIO }`
  - T-CRM-033 `CounterProposalModal`: campos `value (Decimal)`, `conditions (texto)`. Ao submeter → `POST /cases/:id/negotiations/:negId/proposal { type: CEDENTE }`
  - T-CRM-034 `AcceptOrCloseModal`: modo "Aceitar": confirmar valor final + exibir comissão estimada + botão "Confirmar aceite". Modo "Encerrar": select `reason (RECUSADA|EXPIRADA)`, `rejected_by (CEDENTE|CESSIONARIO)`
  - Validar: T-CRM-034 no modo aceitar exibe comissão estimada calculada; modo encerrar exige seleção de motivo

---

## FEATURE 2 — Comissão

### Backend

- [x] **REQ-049 / REQ-220** `GET /cases/:id/commission` — Retornar comissões do Caso (estimadas e confirmadas). Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS` (apenas próprios). Retornar: `cedente_commission { state, base_calculation, value, discount_percent }` e `cessionario_commission { state, delta, value, discount_percent }` e `summary { repasse_price, cedente_commission, cessionario_commission, net_cedente_value, perda_evitada }`
  - Validar: Caso sem comissão → `{ data: null }`; Caso com comissão ESTIMADA → dados de estimativa

- [x] **REQ-049** Implementar cálculo de comissão do Cessionário em `CommissionsService.calculateCessionarioCommission(caseId)`:
  - Se `cases.delta > 0`: `comissão = 0.20 × delta` (`20% × (current_table_value − contract_value)`)
  - Se `cases.delta ≤ 0`: `comissão = 0.20 × valor_pago_cedente` (fallback: `contract_value`)
  - Armazenar `base_calculation` como string descritiva: "20% × Δ" ou "20% × Valor Pago"
  - Verificar se há `Termo Comercial` registrado → usar percentual do Termo em vez de 20%
  - Validar: Δ = R$ 50.000 → comissão R$ 10.000; Δ = 0 → comissão 20% do valor pago; Termo Comercial de 15% → comissão com 15%

- [x] **REQ-050** Implementar cálculo de comissão do Cedente em `CommissionsService.calculateCedanteCommission(caseId)`:
  - Cenário A: `comissão = R$ 0,00`
  - Cenários B, C, D: `comissão = 0.20 × (valor_recuperado − valor_distrato_referencia)`
  - `valor_distrato_referencia` extraído de `dossier_documents` (documento "Valor de Distrato Referência")
  - Verificar `Termo Comercial` registrado → usar parâmetros do Termo
  - Validar: Cenário A → R$ 0,00; Cenário B com distrato R$ 200k e recuperado R$ 300k → comissão R$ 20k; Termo Comercial sobrescreve padrão

- [x] **REQ-051 / REQ-221** `POST /cases/:id/commission/discount` — Roles: `ADMIN_RS`, `COORDENADOR_RS`. Request body: `{ commission_type: CEDENTE|CESSIONARIO, discount_percent: Decimal, justification: string }`. Tabela de limites (RN-052):
  - Repasse ≤ R$ 200.000 → 0% (sem desconto)
  - R$ 200.001 a R$ 500.000 → máx 5%
  - R$ 500.001 a R$ 1.000.000 → máx 10%
  - Acima de R$ 1.000.000 → máx 15%
  - Se `ANALISTA_RS` tenta → 403 `CRM-023`
  - Se desconto acima do limite → 422 `CRM-024`: "O desconto solicitado supera o limite permitido para esta faixa. Contate o Admin RS para casos especiais."
  - Se dentro do limite → aplicar, registrar `discount_applied_by`, `discount_percent`, `audit_log`
  - Validar: Analista RS → 403; desconto 6% em Caso de R$ 400k → 422 CRM-024; desconto 4% em R$ 400k → 200 aplicado

- [x] **REQ-052 / REQ-053** Implementar registro do Termo Comercial via `POST /cases/:id/commission/term`:
  - Request body: `{ cedente_commission_percent?: Decimal, cessionario_commission_percent?: Decimal, floor_value?: Decimal, ceiling_value?: Decimal, pdf_url: string }`
  - Armazenar em tabela `commission_terms` (criar se não existe) vinculada ao Caso
  - Todos os cálculos subsequentes verificam existência de Termo antes de aplicar defaults
  - Validar: Caso com Termo de 15% Cessionário → cálculo usa 15%; sem Termo → usa 20% padrão

- [x] **REQ-055 / REQ-056** Implementar lógica de Fechamento em `CommissionsService.confirmFechamento(caseId)`:
  - Verificar 3 critérios (REQ-053): instrumento cessão assinado (arquivo em `dossier_documents` com `document_type = "INSTRUMENTO_CESSAO"` e `status = APROVADO`), anuência incorporadora (`document_type = "ANUENCIA"` e `status = APROVADO`), comprovante Escrow (`document_type = "COMPROVANTE_ESCROW"` e `status = APROVADO`)
  - Se critério pendente → 422 `CRM-025` com lista: `{ "pending_criteria": ["Instrumento de cessão pendente", ...] }`
  - Se todos atendidos → atualizar `commissions.state = CONFIRMADA` para ambas (Cedente + Cessionário), gerar resumo financeiro completo, registrar em `audit.audit_logs`
  - Resumo financeiro: `{ repasse_price, cedente_commission, cessionario_commission, net_cedente_value: (repasse_price - cedente_commission), perda_evitada }`
  - Validar: Fechamento sem instrumento → 422 com critério listado; Fechamento com todos os 3 → comissões CONFIRMADAS; resumo financeiro com Perda Evitada calculada

- [x] **REQ-056 / REQ-222** `POST /cases/:id/commission/cancel` — Roles: `ADMIN_RS` only. Request body: `{ justification: string (mínimo 100 chars), second_admin_id: UUID, proof_document_url: string }`. Regras:
  - Apenas Admin RS pode acionar
  - `second_admin_id` deve ser um Admin RS diferente do usuário logado (quatro olhos)
  - Justificativa mínimo 100 caracteres
  - Proof document (URL do arquivo comprobatório) obrigatório
  - Sucesso: `commissions.state = CANCELADA`; NUNCA excluir o registro; notificar Coordenador RS; registrar em `audit.audit_logs`
  - Validar: Analista RS → 403; Coordenador RS → 403; justificativa < 100 chars → 422; `second_admin_id` igual ao requestor → 422 "A aprovação deve ser de outro Admin RS"; comissão cancelada permanece no histórico com state CANCELADA

### Frontend

- [x] Implementar componente `CommissionPanel` em `components/commissions/CommissionPanel.tsx`:
  - Exibir: Δ calculado, base de cálculo, comissão estimada Cedente + Cessionário, estado atual (ESTIMADA/CONFIRMADA/CANCELADA), desconto aplicado (se houver)
  - Botão "Aplicar Desconto" (visível apenas para Coord/Admin)
  - Botão "Registrar Fechamento" (visível em estado FORMALIZACAO)
  - Indicadores visuais dos 3 critérios de Fechamento com status (✓/✗)
  - Validar: comissão CONFIRMADA mostra resumo financeiro completo com Perda Evitada; critérios de Fechamento atualizados em tempo real conforme Dossiê (via TanStack Query refetch)

---

## 🧪 Testes

### Backend — Unitário (Jest)

- [x] **REQ-254** `CommissionsService` — cobertura 85% unitário:
  - `should calculate cessionario commission with delta > 0 as 20% of delta`
  - `should calculate cessionario commission with delta <= 0 as 20% of paid value`
  - `should calculate cedente commission as 0 for scenario A`
  - `should calculate cedente commission as 20% of (recovered - distrato) for scenario B`
  - `should calculate cedente commission as 20% of (recovered - distrato) for scenario C`
  - `should calculate cedente commission as 20% of (recovered - distrato) for scenario D`
  - `should use Termo Comercial percent when present`
  - `should apply discount limit of 0% for values <= 200000`
  - `should apply discount limit of 5% for values between 200001 and 500000`
  - `should apply discount limit of 10% for values between 500001 and 1000000`
  - `should apply discount limit of 15% for values above 1000000`
  - `should block cancellation without second admin approval`

- [x] **REQ-256** `NegotiationsService` — cobertura 75% unitário:
  - `should create negotiation from MATCH state`
  - `should block creation if active negotiation exists`
  - `should set valid_until to 48 hours after creation`
  - `should warn when proposal rounds exceed 5`
  - `should transition state to ACEITA on accept and calculate commissions`
  - `should return case to PUBLICACAO on close`
  - `should mark cessionario contact with rejection flag`

### Backend — Integração

- [x] **REQ-254 / REQ-256** Testes de integração:
  - `POST /cases/:id/negotiations` → Negociação criada + Caso em MATCH
  - `POST .../proposal { value: below_floor }` → 200 com warning BELOW_FLOOR
  - `POST .../accept` → comissões ESTIMADAS criadas, Caso em ANUENCIA
  - `POST .../commission/discount` por Analista RS → 403
  - `POST .../commission/discount` com desconto acima do limite → 422 CRM-024

---

## 🔀 Cross-Módulo

- `CommissionsService.confirmFechamento()` verifica documentos do Dossiê — implementação completa do Dossiê em S5. Até S5: retorna 422 "Dossiê não implementado ainda."
- Worker de expiração de proposta (48h) integra com sistema de notificações (S8). Registrar pendência: S8 deve consumir a fila `negotiations.expired`.
- Aceite da proposta avança Caso para `ANUENCIA` via `CasesService.advance()` — lógica de gate desta transição implementada em S3.
