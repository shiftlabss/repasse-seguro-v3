# S4 — Pipeline e Casos

## Módulo Admin — Repasse Seguro

| Campo                | Valor                                                                                                                                                                                                                  |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S4                                                                                                                                                                                                                     |
| **Nome**             | Pipeline e Casos                                                                                                                                                                                                       |
| **Tipo**             | Dinâmica — Módulo Fullstack                                                                                                                                                                                            |
| **Template**         | B                                                                                                                                                                                                                      |
| **Docs Consultados** | D01.1, D01.3, D06, D16                                                                                                                                                                                                 |
| **Cross-cutting**    | D10 (Glossário), D02 (Stacks)                                                                                                                                                                                          |
| **REQs cobertos**    | REQ-005, REQ-016, REQ-030, REQ-037, REQ-038, REQ-042, REQ-055, REQ-063, REQ-064, REQ-083, REQ-111, REQ-126, REQ-127, REQ-166, REQ-198, REQ-199, REQ-200, REQ-288, REQ-289, REQ-290, REQ-291, REQ-292, REQ-293, REQ-295 |
| **Total de itens**   | 55                                                                                                                                                                                                                     |

---

> **Critério de conclusão de S4:** CRUD completo de casos com state machine validada; kanban com 14 colunas e polling 15s; visão lista paginada; drawer de detalhe T-022 com 4 tabs; ANALISTA só vê seus casos; drag-and-drop disponível para COORDENADOR/MASTER; SLA badges verde/amarelo/vermelho em todos os cards; `case_config_snapshot` congelado na criação; cancelamento disponível em estados pré-Fechamento; lock otimista com `version` em todas as mutações.

---

## ⚙️ BACKEND

### Feature: CRUD de Casos

- [x] **[BE-01]** Implementar `GET /v1/cases` (ANALISTA/COORDENADOR/MASTER/GESTOR_FINANCEIRO): ANALISTA filtra por `assigned_analyst_id = userId` automaticamente (RN-013); COORDENADOR/MASTER veem todos; GESTOR_FINANCEIRO: somente leitura (mesmo endpoint, sem ações); parâmetros: `page`, `per_page=25`, `status` (enum `CaseStatus`), `scenario` (A/B/C/D), `assigned_analyst_id`, `sort` (campo), `order` (asc/desc), `search` (endereço ou ID); resposta inclui `case_status_badge`, `cedente.name`, `assigned_analyst.name`, `sla_status` (GREEN/YELLOW/RED), `sla_days_remaining`; paginação padrão `{ page, per_page, total, total_pages }`. Verificar: token ANALISTA não retorna casos de outros analistas; parâmetro `status=EM_TRIAGEM` filtra corretamente.

- [x] **[BE-02]** Implementar `POST /v1/cases` (ANALISTA): cria caso com campos `cedente_id`, `scenario` (A/B/C/D — REQ-005), `property_address`, `contract_table_value`, `paid_amount`, `assigned_analyst_id?`; ao criar: (1) verificar RN-003 — se já existe caso ativo para o mesmo imóvel+Cedente → 422 com `"Um caso ativo já existe para este imóvel e Cedente"` (REQ-063); (2) se Cedentes diferentes para o mesmo imóvel → criar e gerar alerta automático ao Coordenador (REQ-064); (3) criar automaticamente `case_config_snapshot` (snapshot das configs globais no momento da criação — RN-111/REQ-166); (4) criar `formalization_criteria` com 4 critérios como `false`; (5) criar `escrow_account` com status `Aberta`; retorna 201 com caso completo. Verificar: segundo caso para mesmo imóvel+Cedente retorna 422; `case_config_snapshot` criado com valores corretos das configurações atuais.

- [x] **[BE-03]** Implementar `GET /v1/cases/:id` (ANALISTA): retorna caso completo com `config_snapshot`, `formalization_criteria`, `escrow_account` summary; ANALISTA: apenas se `assigned_analyst_id = userId` ou se COORDENADOR/MASTER. Verificar: ANALISTA não consegue buscar caso de outro analista (404 ou 403).

- [x] **[BE-04]** Implementar `PATCH /v1/cases/:id` (ANALISTA): campos editáveis: `property_address`, `current_table_value`; **obrigatório** incluir `version` no request; se `version` != DB version → 409 `{ title: "Conflict", detail: "Caso atualizado por outro operador. Recarregue." }` (lock otimista — RN-013); ao salvar → incrementar `version`. Verificar: requisição com `version` desatualizado retorna 409; requisição com `version` correto salva e incrementa.

- [x] **[BE-05]** Implementar `PATCH /v1/cases/:id/status` (ANALISTA para avanços simples / COORDENADOR para cancelamento e bloqueio): campos `to_status` (CaseStatus), `reason?`, `version`; validar transição contra state machine (14 estados — D01.1 §5); se transição inválida → 422 `{ detail: "Transição não permitida de {status_atual} para {to_status}. Consulte o fluxo de estados permitidos." }`; lock otimista com `version`; registrar em `audit.audit_logs` campos: `case_id`, `from_status`, `to_status`, `operator_id`, `timestamp`, `reason`; iniciar/reiniciar timer SLA para a nova etapa (REQ-037/REQ-126). Regras de cancelamento (REQ-055): disponível em CAPTADO, BLOQUEADO, EM_TRIAGEM, QUALIFICADO, OFERTA_ATIVA, EM_NEGOCIACAO, EM_FORMALIZACAO; apenas COORDENADOR/MASTER podem cancelar. Verificar: transição QUALIFICADO → FECHAMENTO retorna 422; transição CAPTADO → EM_TRIAGEM salva e registra em audit.

- [x] **[BE-06]** Implementar `PATCH /v1/cases/:id/assign` (COORDENADOR): campos `analyst_id`, `version`; valida que `analyst_id` é usuário com `role = ANALISTA` e `is_active = true`; lock otimista; registrar em audit. Verificar: atribuição a usuário inativo retorna 422; atribuição correta salva e registra.

- [x] **[BE-07]** Implementar `GET /v1/cases/:id/status-history` (ANALISTA): retorna histórico cronológico de todas as transições do caso; cada item: `from_status`, `to_status`, `operator.name`, `timestamp`, `reason`; ordenado por `timestamp DESC`. Verificar: histórico completo de um caso com 3 transições retorna 3 itens na ordem correta.

- [x] **[BE-08]** Implementar `GET /v1/cases/:id/config-snapshot` (ANALISTA): retorna `case_config_snapshot` do caso; usado pelo frontend para calcular comissões com os parâmetros vigentes na época da criação do caso (RN-111). Verificar: snapshot retorna parâmetros da data de criação mesmo após alteração das configurações globais.

### Feature: SLA Monitor

- [x] **[BE-09]** Implementar `SlaMonitorService` — job cron a cada 5 minutos (_/5 _ \* \* \*): para cada caso ativo, calcular `sla_days_remaining` e `sla_status` (GREEN/YELLOW/RED) conforme tabela RN-059: Captado→EmTriagem: SLA máximo 24h; EmTriagem→Qualificado: SLA alvo 3 dias úteis / máximo 5 dias úteis; Qualificado→OfertaAtiva: SLA alvo 2 dias úteis / máximo 3 dias úteis; OfertaAtiva→EmNegociacao: SLA alvo 15 dias corridos / máximo 30 dias corridos; EmNegociacao→EmFormalizacao: SLA alvo 10 dias úteis / máximo 20 dias úteis; EmFormalizacao→Fechamento: SLA alvo 10 dias úteis / máximo 20 dias úteis; PosFechamento→Concluido: 15 dias corridos fixo. YELLOW quando `sla_days_remaining <= 20% do SLA máximo`; RED quando `sla_days_remaining <= 0`. Verificar: caso em EmTriagem há 5 dias úteis tem `sla_status = RED`; caso em EmTriagem há 2 dias tem `sla_status = GREEN`.

- [x] **[BE-10]** Implementar alertas automáticos de SLA (RN-059/REQ-127): quando `sla_status` muda para YELLOW → criar alerta no painel do Analista e Coordenador (sem e-mail); quando `sla_status` muda para RED → criar alerta por e-mail E painel para Coordenador e Master; mensagem e-mail: "URGENTE: o caso {case_id} estourou o SLA da etapa {etapa}. Ação imediata necessária."; mensagem painel YELLOW: "Atenção: o caso {case_id} está a {X}% do SLA máximo da etapa {etapa}. {N} dia(s) restante(s).". Verificar: mudança de GREEN para YELLOW cria alerta de painel sem e-mail; mudança para RED cria alerta de painel E dispara job de e-mail.

- [x] **[BE-11]** Implementar endpoint `GET /v1/cases` com suporte a Supabase Realtime: ao mutação de `cases` → publicar evento no canal `cases:{userId}` para o Analista atribuído e `cases:all` para COORDENADOR/MASTER; payload mínimo: `{ case_id, new_status, updated_at }`. Verificar: PATCH /v1/cases/:id/status gera evento Realtime no canal correto.

---

## 🖥️ FRONTEND

### Feature: T-020 — Pipeline Visão Kanban

- [x] **[FE-01]** Implementar `PipelineKanbanPage` (T-020, rota `/pipeline?view=kanban`): board kanban com **14 colunas**, uma por estado (CAPTADO, EM_TRIAGEM, BLOQUEADO, QUALIFICADO, OFERTA_ATIVA, EM_NEGOCIACAO, EM_FORMALIZACAO, FECHAMENTO, POS_FECHAMENTO, EM_REVERSAO, EM_MEDIACAO, DISPUTA_FORMAL, CONCLUIDO, CANCELADO); header por coluna: nome do estado + contagem de cards; cards ordenados por data de entrada no estado (mais antigos no topo); scroll horizontal do board em desktop ≥1280px; em mobile `<768px` redireciona automaticamente para T-021. RBAC: ANALISTA vê apenas seus casos; GESTOR_FINANCEIRO: somente leitura (sem ações nos cards). Verificar: 14 colunas presentes; ANALISTA vê apenas seus casos; mobile redireciona para lista.

- [x] **[FE-02]** Implementar `CaseCard` component para o kanban: exibir ID do caso (badge), endereço do imóvel (truncate 2 linhas), Analista responsável (avatar 24px + nome), cenário (A/B/C/D badge colorido), tempo no estado atual (badge relativo: "Há 3 dias"), `SlaIndicator` (badge verde/amarelo/vermelho com ícone diferenciado: ✓ verde / relógio amarelo / ! vermelho + contador regressivo "-2d" para estourados — RN-059 acessibilidade), valor estimado; ao clicar → abrir T-022 como drawer lateral direito (480px). Verificar: SLA badge varia cor e ícone corretamente; clique abre drawer sem navegar.

- [x] **[FE-03]** Implementar drag-and-drop no kanban (COORDENADOR/MASTER apenas — RN-013): arrastar card entre colunas; coluna de destino inválida → `opacity-40`; coluna válida → borda `--primary`; ao soltar → chamar `PATCH /v1/cases/:id/status`; se 422 (transição inválida) → snap card de volta + toast "Transição não permitida de {status_atual} para {status_destino}"; se 409 (conflict) → snap back + toast "Este caso foi atualizado por outro operador. Recarregue."; alternativa acessível via menu de contexto "Mover para..." (botão em cada card). ANALISTA e GESTOR_FINANCEIRO: drag-and-drop desabilitado. Verificar: ANALISTA não consegue arrastar cards; transição inválida snaps back com toast correto.

- [x] **[FE-04]** Implementar filtros do kanban: barra de filtros acima do board: Analista (multi-select — apenas COORDENADOR/MASTER), Cenário (A/B/C/D checkboxes), SLA (Vencendo hoje / Vencido / Em dia); toggle "Kanban / Lista" no canto superior direito → T-021; filtros passam como query params para `GET /v1/cases`. Verificar: filtro Cenário=A remove cards de outros cenários; toggle troca view mantendo filtros.

- [x] **[FE-05]** Implementar estados do kanban: `loading` → skeleton cards por coluna; `vazio por coluna` → mensagem "Nenhum caso neste estado" (sem CTA); `vazio total` → mensagem "Nenhum caso cadastrado. Aguardando cadastro pelo Cedente."; `atualização em tempo real` → Supabase Realtime move card entre colunas com animação slide horizontal 300ms ease-in-out + badge "Atualizado agora" por 3s no card. Verificar: evento Realtime move card com animação; badge some após 3s.

### Feature: T-021 — Pipeline Visão Lista

- [x] **[FE-06]** Implementar `PipelineListPage` (T-021, rota `/pipeline?view=list`): tabela paginada 25/página; colunas: ID, Imóvel (truncado), Estado atual (badge colorido), Cenário (A/B/C/D), Analista (avatar+nome), Tempo no estado, SLA (ícone colorido), Valor estimado, Ações; RBAC: ANALISTA filtra seus casos; GESTOR_FINANCEIRO somente leitura; ações por linha: "Ver detalhe" → T-022; COORDENADOR/MASTER: dropdown "Reatribuir analista" + "Cancelar caso"; busca por endereço/ID + mesmos filtros do kanban. Responsividade: desktop = tabela completa; tablet = colunas "Valor" e "Analista" ocultadas (acessíveis via expansão da linha); mobile = lista de cards (ID + endereço + estado + SLA). Acessibilidade: `role="table"`, `scope="col"`, `aria-sort` em colunas ordenáveis. Verificar: paginação funciona; mobile exibe cards; COORDENADOR vê dropdown com ações.

### Feature: T-022 — Pipeline Detalhe do Caso (Drawer)

- [x] **[FE-07]** Implementar `CaseDetailDrawer` (T-022): drawer lateral direito 480px em desktop, fullscreen slide-up em mobile; cabeçalho: ID + endereço + estado atual (badge) + cenário; 4 tabs: Resumo · Dossiê · Histórico · Ações; focus trap dentro do drawer; `aria-modal="true"`; fechamento via botão "×", Escape key ou clique no overlay semitransparente; `aria-labelledby` no título; estados: `loading` = skeleton nas tabs, `erro ao carregar` = mensagem + "Tentar novamente". Verificar: focus trap impede tab para fora do drawer; Escape fecha; clique no overlay fecha.

- [x] **[FE-08]** Implementar tab **Resumo** no `CaseDetailDrawer`: dados do Cedente (nome, CPF mascarado para ANALISTA, CPF completo para COORDENADOR/MASTER — RN-131), Cessionário (quando houver; anonimizado para ANALISTA em fase de negociação), valores calculados (`contract_table_value`, `paid_amount`, `current_table_value`), Analista responsável (avatar + nome), timeline simplificada do ciclo de vida (ícones de estado com check/pending/locked). Verificar: ANALISTA vê CPF mascarado `***.***.***/***-**`; COORDENADOR vê CPF completo.

- [x] **[FE-09]** Implementar tab **Histórico** no `CaseDetailDrawer`: log cronológico de todas as ações do caso; cada item: `from_status` → `to_status`, nome do operador, data/hora relativa, `reason` quando presente; dados de `GET /v1/cases/:id/status-history`; mais recente no topo. Verificar: histórico de 3 transições aparece na ordem correta.

- [x] **[FE-10]** Implementar tab **Ações** no `CaseDetailDrawer` com RBAC dinâmico: **ANALISTA:** botão "Iniciar Triagem" (se CAPTADO); **COORDENADOR/MASTER:** botões "Reatribuir analista" (abre modal com select) + "Cancelar caso" (abre modal de confirmação com textarea obrigatório de motivo); todos os botões com `ActionButton` (spinner + desabilitado durante request — S1); cancelamento via `PATCH /v1/cases/:id/status { to_status: "CANCELADO", reason, version }`. Verificar: ANALISTA não vê botão "Cancelar caso"; cancelamento sem reason retorna 422; cancelamento em FECHAMENTO retorna 422.

---

## 🔗 WIRING

- [x] **[WIRE-01]** Implementar `cases.service.ts` no frontend: `getCases(filters)` → `GET /v1/cases`; `getCase(id)` → `GET /v1/cases/:id`; `createCase(dto)` → `POST /v1/cases`; `updateCase(id, dto)` → `PATCH /v1/cases/:id`; `updateCaseStatus(id, dto)` → `PATCH /v1/cases/:id/status`; `assignCase(id, dto)` → `PATCH /v1/cases/:id/assign`; `getCaseHistory(id)` → `GET /v1/cases/:id/status-history`; todas com header `X-Idempotency-Key` gerado por `crypto.randomUUID()` nas mutações. Verificar: `createCase` com `version` inválido lança erro com mensagem "Este caso foi atualizado por outro operador. Recarregue."

- [x] **[WIRE-02]** Implementar listener Supabase Realtime no Pipeline: assinar canal `cases:{userId}` (ANALISTA) ou `cases:all` (COORDENADOR/MASTER); ao receber evento → `queryClient.invalidateQueries(['cases'])` + mover card com animação; polling fallback de 15 segundos se Realtime não disponível (RN-132). Verificar: evento de mudança de status move o card no kanban em tempo real com animação.

---

## ✅ TESTES

- [x] **[TEST-01]** Testes unitários `CasesService` (backend): `POST /v1/cases` com mesmo imóvel+Cedente retorna 422; `case_config_snapshot` criado corretamente na criação do caso; lock otimista 409 quando `version` desatualizado; transição de status inválida retorna 422 com mensagem correta; ANALISTA não acessa casos de outros analistas (403/404). Cobertura: 80% de branches.

- [x] **[TEST-02]** Testes unitários `SlaMonitorService`: caso em EmTriagem há exatamente 4 dias úteis = YELLOW (80% de 5 dias); caso em EmTriagem há 5 dias úteis = RED; alerta YELLOW cria evento de painel sem job de e-mail; alerta RED cria evento de painel E job de e-mail.

- [x] **[TEST-03]** Testes de integração: `POST /v1/cases` cria `case_config_snapshot`, `formalization_criteria` (4 critérios false), `escrow_account` (status Aberta); `PATCH /v1/cases/:id/status` com transição válida registra em `audit.audit_logs`; `PATCH /v1/cases/:id/status` com transição inválida retorna 422 e não registra em audit.

- [ ] **[TEST-04]** Testes E2E (Playwright): fluxo Pipeline — login COORDENADOR → `/pipeline?view=kanban` → arrastar card de CAPTADO para EM_TRIAGEM → verificar card move de coluna; login ANALISTA → tentar arrastar card → verificar que drag-and-drop está desabilitado; filtro Cenário=A → verificar que apenas cards A permanecem visíveis.

---

## 🔍 AUTO-VERIFICAÇÃO S4 (12 checks)

- [x] **[CHECK-01]** `GET /v1/cases` com token ANALISTA retorna apenas casos com `assigned_analyst_id = userId`
- [x] **[CHECK-02]** `POST /v1/cases` com mesmo imóvel+Cedente em caso ativo retorna 422 com mensagem correta (RN-003)
- [x] **[CHECK-03]** `case_config_snapshot` criado automaticamente ao criar caso com valores das configurações atuais (RN-111)
- [x] **[CHECK-04]** Lock otimista: `PATCH /v1/cases/:id` com `version` desatualizado retorna 409
- [x] **[CHECK-05]** Transição de status inválida (ex: QUALIFICADO → FECHAMENTO) retorna 422
- [x] **[CHECK-06]** SLA badge: caso em EmTriagem há 5 dias úteis mostra badge VERMELHO com ícone "!" no kanban
- [x] **[CHECK-07]** Kanban: 14 colunas presentes (incluindo CANCELADO e CONCLUIDO)
- [x] **[CHECK-08]** Drag-and-drop: ANALISTA não consegue arrastar cards; COORDENADOR consegue
- [x] **[CHECK-09]** Mobile (<768px): `/pipeline?view=kanban` redireciona automaticamente para T-021 (lista)
- [x] **[CHECK-10]** Drawer T-022: ANALISTA vê CPF mascarado do Cedente; COORDENADOR vê CPF completo
- [x] **[CHECK-11]** Cancelamento de caso em FECHAMENTO retorna 422 (estado pós-Fechamento não cancelável)
- [x] **[CHECK-12]** Todos os REQs de S4 (REQ-005, 016, 030, 037, 038, 042, 055, 063, 064, 083, 111, 126, 127, 166, 198–200, 288–295) cobertos
