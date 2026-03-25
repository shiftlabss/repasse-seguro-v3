# S3 — Pipeline de Casos

## CRM Repasse Seguro

| **Campo**         | **Valor**                                                                                                                                                                               |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**        | S3                                                                                                                                                                                      |
| **Nome**          | Pipeline de Casos                                                                                                                                                                       |
| **Template**      | B — Módulo Fullstack                                                                                                                                                                    |
| **Docs fonte**    | 01.1, 01.2, 05.1, 05.2, 06, 16, 27                                                                                                                                                      |
| **REQs cobertos** | REQ-006 a REQ-009, REQ-016 a REQ-043, REQ-146 a REQ-148, REQ-191 a REQ-199, REQ-253, REQ-273, REQ-281                                                                                   |
| **Objetivo**      | Analista RS consegue criar Caso, visualizar pipeline em kanban e lista, avançar/retroceder estados, redistribuir, cancelar e filtrar. Máquina de estados completa com SLAs monitorados. |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — `cases`, `case_status_history`, `CaseState`, `RS-YYYY-NNNN`, 9 estados exatos
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: SLAs exatos: Cadastro 1d, Simulação 2d, Verificação 5d, Publicação 1d, Match 3d, Negociação 7d, Anuência 10d, Formalização 5d; limites: 30 Casos/Analista, 60 dias ciclo total, 90 dias arquivamento, 20 chars justificativa
- [x] C4: Glossário — Optimistic Locking (REQ-273), Condição de Saída, Ciclo de Vida do Caso cobertos
- [x] C5: TODAS as 13 transições de estado cobertas (9 sequenciais + 3 retrocessos + 1 cancelamento múltiplos)
- [x] C6: Cron 08h00 America/Fortaleza para SLA / arquivamento mensal cobertos
- [x] C7: RBAC: criar Caso (Admin+Coord+Analista), retroceder (Coord+Admin), redistribuir (Coord+Admin)
- [x] C8: T-CRM-020/021/022 com estados loading/empty/error
- [x] C9: Integração com `sla_alerts`, `case_status_history` append-only
- [x] C10: Glossário — Caso como unidade operacional, SLA do Caso, Condição de Saída
- [x] C11: Anti-scaffold — lógica de transição, condições de saída, log auditoria em todo item
- [x] C12: REQ-006 a REQ-043 cobertos

---

## FEATURE 1 — Criação e Edição de Casos

### Banco / Migrations (já criado em S1 — verificar campos)

- [x] **REQ-006 / REQ-118** Verificar que a tabela `cases` tem o campo `case_number` com formato `RS-YYYY-NNNN` gerado automaticamente via sequence ou trigger PostgreSQL. Criar função SQL `generate_case_number(year INT) RETURNS TEXT` que gera `RS-{year}-{NNNN com zero-padding}` com contador atomicamente incrementado por ano
  - Validar: inserção de 2 Casos no mesmo ano → `RS-2026-0001` e `RS-2026-0002`; inserção em anos diferentes → contadores independentes

### Backend

- [x] **REQ-192** `POST /cases` — Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS`. Request body (validado via `CreateCaseDto`): `cedente_name`, `cedente_phone`, `cedente_email`, `enterprise_name`, `enterprise_address`, `contract_value (Decimal)`, `scenario (enum: A|B|C|D)` — todos obrigatórios. Lógica:
  - Criar `contacts` (Cedente) se e-mail + CPF não existem → criar novo Contato
  - Se e-mail ou CPF já existe → retornar `CRM-014` 409 com dados do Contato existente para decisão do Analista
  - Criar `cases` com `state = CADASTRO`, `assigned_to = userId`, `case_number = generate_case_number(year)`, `state_changed_at = now()`, `version = 1`
  - Criar registro em `case_status_history`: `from_state = null`, `to_state = CADASTRO`, `changed_by = userId`
  - Criar entrada em `audit.audit_logs`
  - Retornar 201 com Caso criado
  - Validar: campo ausente → 422 `CRM-015` com lista de campos; e-mail duplicado → 409 com dados Contato; Caso criado com state CADASTRO e ID correto; log de auditoria criado

- [x] **REQ-194** `PATCH /cases/:id` — Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS` (apenas próprios para Analista). Request body: campos editáveis (`enterprise_name`, `enterprise_address`, `contract_value`, `cedente_email`, `cedente_phone`). Verificar `version` do Caso (Optimistic Locking — REQ-273): se `version` do body ≠ `version` do banco → retornar `CRM-016` 409 "Conflito de edição: o Caso foi modificado por outro usuário." Incrementar `version` ao salvar. Registrar em `audit.audit_logs`
  - Validar: Analista RS não consegue editar Caso de outro Analista (403); edição concorrente com version stale → 409 CRM-016; version incrementado após cada PATCH bem-sucedido

- [x] **REQ-193** `GET /cases/:id` — Roles: todos autenticados. Retornar Caso com: dados do Cedente (mascarados se role ≠ `ADMIN_RS`/`COORDENADOR_RS`/Analista responsável), estado atual, histórico de estados (`case_status_history`), SLA atual (calculado: dias no estado atual vs SLA esperado), estimativa de fechamento, Analista RS atribuído
  - Validar: `PARCEIRO_EXTERNO` não acessa dados pessoais (campos retornados: apenas `state`, `estimated_close_at`, `enterprise_name`); `ANALISTA_RS` não responsável → 403

### Frontend

- [x] **REQ-006** Implementar formulário "Novo Caso" em `components/cases/NewCaseForm.tsx`:
  - Campos obrigatórios com React Hook Form + Zod: `cedente_name`, `cedente_phone` (formato WhatsApp), `cedente_email`, `enterprise_name`, `enterprise_address`, `contract_value` (formatado R$ X.XXX,XX), `scenario (radio: A|B|C|D)`
  - Submit: chamar `POST /cases`; campo ausente → marcar em vermelho + "Preencha todos os campos obrigatórios para continuar."
  - Sucesso: redirecionar para `/cases/{id}` com toast "Caso RS-2026-NNNN criado com sucesso."
  - Resposta 409 (contato duplicado): exibir modal "Este contato já está cadastrado." com opções "Vincular ao existente" | "Criar novo"
  - Validar: submit com campo vazio → campo marcado em vermelho; `contract_value` aceita apenas números > 0; scenario obrigatório; duplicata → modal com os 2 botões

---

## FEATURE 2 — Máquina de Estados: Avanço e Retrocesso

### Backend

- [x] **REQ-195** `POST /cases/:id/advance` — Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS` (apenas próprios). Lógica de validação de condição de saída por estado:
  - `CADASTRO → SIMULACAO`: verificar `scenario` preenchido + `cedente_contact_id` existente
  - `SIMULACAO → VERIFICACAO`: verificar que Cedente confirmou cenário (flag `scenario_confirmed = true` em `cases`)
  - `VERIFICACAO → PUBLICACAO`: verificar `dossier_documents` com status `APROVADO` (gate: REQ-027)
  - `PUBLICACAO → MATCH`: verificar que `cessionario_contact_id` preenchido
  - `MATCH → NEGOCIACAO`: verificar que Negociação ativa criada para o Caso
  - `NEGOCIACAO → ANUENCIA`: verificar que Negociação tem status `ACEITA`
  - `ANUENCIA → FORMALIZACAO`: verificar documento de anuência no Dossiê com status `APROVADO`
  - `FORMALIZACAO → CONCLUIDO`: verificar 3 critérios de Fechamento (REQ-053): instrumento assinado + anuência + comprovante Escrow
  - Se condição NÃO atendida: retornar 422 `CRM-017` com lista de pendências: `{ "pending": ["Dossiê não aprovado", ...] }`
  - Se condição atendida: atualizar `cases.state`, `state_changed_at = now()`, incrementar `version`, criar `case_status_history`, criar `audit_log`
  - Validar: Caso em VERIFICACAO sem Dossiê aprovado → 422 com pendências; transição válida → novo estado retornado; history criado; auditoria criada

- [x] **REQ-197** `POST /cases/:id/rollback` — Roles: `ADMIN_RS`, `COORDENADOR_RS`. Request body: `{ justification: string (mínimo 20 chars) }`. Regras:
  - `CONCLUIDO` → NÃO pode retroceder: retornar `CRM-018` 422 "Casos concluídos não podem ser reabertos. Se necessário, abra um novo Caso e contate o suporte."
  - `CANCELADO` → NÃO pode retroceder: retornar `CRM-019` 422 "Casos cancelados não podem ser reabertos. Abra um novo Caso."
  - `ANALISTA_RS` → 403 (não tem permissão para retroceder)
  - Justificativa < 20 chars → 422 `CRM-020`
  - Sucesso: retroceder para estado anterior (conforme `case_status_history`), criar nova entrada em `case_status_history`, criar `audit_log`
  - Validar: Caso CONCLUIDO → CRM-018; Analista RS → 403; justificativa curta → CRM-020; retrocesso registrado no histórico com justificativa

- [x] **REQ-196** `POST /cases/:id/cancel` — Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS` (apenas próprios). Request body: `{ cancel_reason: enum, cancel_reason_detail?: string }`. Motivos obrigatórios: `CEDENTE_DESISTIU | DOCUMENTACAO_INVALIDA | NENHUM_CESSIONARIO | ANUENCIA_NEGADA | OUTROS`. Se `OUTROS`: `cancel_reason_detail` obrigatório. Caso em `CONCLUIDO` → 422 não cancelável. Sucesso: state → `CANCELADO`, registrar em `cases.cancel_reason`, `case_status_history`, `audit.audit_logs`. Nenhuma comissão registrada
  - Validar: cancelar sem motivo → 422 CRM-021; cancelar com OUTROS sem detalhe → 422; cancelar Caso CONCLUIDO → 422; cancelamento com motivo → state CANCELADO + log

- [x] **REQ-199** `POST /cases/:id/assign` — Roles: `ADMIN_RS`, `COORDENADOR_RS`. Request body: `{ new_analyst_id: UUID }`. Verificar contagem de Casos abertos do Analista destino:
  - Se < 30 Casos: redistribuir imediatamente, notificar novo Analista por e-mail (fila RabbitMQ), registrar em `audit.audit_logs`. Analista anterior perde acesso (exceto histórico de atividades próprias)
  - Se ≥ 30 Casos: retornar 200 com `{ data: { warning: "ANALYST_OVERLOADED", current_cases: N, case_assigned: true } }` (redistribui mas avisa — RN-041 permite confirmar mesmo assim)
  - Validar: redistribuição → Analista anterior não vê Caso em `GET /cases` (filtro RLS); Analista destino recebe notificação; alerta quando ≥ 30 Casos

- [x] **REQ-198** `POST /cases/:id/duplicate` — Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS`. Apenas para Casos com state `CANCELADO` ou `CONCLUIDO`. Cria novo Caso copiando: `cedente_contact_id`, `enterprise_name`, `enterprise_address`, `contract_value`. NÃO copia: `scenario`, atividades, comissões, dossiê, histórico. Novo ID `RS-YYYY-NNNN`, state `CADASTRO`, `assigned_to = userId`
  - Validar: duplicar Caso ATIVO → 422; Caso duplicado não tem atividades do original; scenario vazio no novo Caso; novo case_number gerado corretamente

---

## FEATURE 3 — Visualização do Pipeline

### Backend

- [x] **REQ-191** `GET /cases` — Roles: todos autenticados (com escopo por role via `RlsDataScopeMiddleware`). Query params: `state`, `assigned_to`, `enterprise_name`, `value_min`, `value_max`, `created_from`, `created_to`, `sla_status (ON_TRACK|AT_RISK|OVERDUE)`, `cancel_reason`, `search (texto livre)`, `page`, `per_page`. Para `search`: full-text search em `case_number`, `enterprise_name`, contatos vinculados (nome mascarado). Retornar com metadados de SLA calculados por Caso (percentual do SLA consumido)
  - Validar: `ANALISTA_RS` → apenas Casos próprios; filtro `sla_status=OVERDUE` → apenas Casos acima de 100% SLA; `search=RS-2026` → retorna Casos com `case_number` contendo "RS-2026"; paginação funciona

- [x] **REQ-009** Implementar cron job `SlaCheckerWorker` executando todo dia às **08h00 America/Fortaleza** via `@nestjs/schedule`:
  - Para cada Caso ativo: calcular dias úteis no estado atual vs SLA esperado (conforme tabela RN-009/REQ-016 a REQ-024)
  - 80% consumido → criar/atualizar `sla_alerts` com `level = AMARELO`; notificar Analista RS + Coordenador RS via fila RabbitMQ
  - 100% consumido → criar `sla_alerts` com `level = VERMELHO`; notificar Analista RS + Coordenador RS com urgência
  - Ciclo total > 60 dias corridos → `level = CRITICO`; notificar Coordenador RS + Admin RS
  - Inatividade > 7 dias sem Atividade → `level = INATIVIDADE`; notificar Analista RS + Coordenador RS
  - Validar: cron executa às 08h00 Fortaleza (verificar com `TZ=America/Fortaleza`); Caso com 80% SLA → sla_alert nível AMARELO; Caso com 100% SLA → badge vermelho disponível para frontend; logs Pino registram execução

- [x] **REQ-043** Implementar cron job `CaseArchivingWorker` executando **mensalmente** (1º dia do mês, 00h00): buscar Casos `CANCELADO` com `last_activity_at < now() - 90 days` → atualizar `cases.state` para `ARQUIVADO` (novo value no enum CaseState) → notificar Analista RS e Coordenador RS
  - Validar: cron mensal configurado; Caso cancelado há 90 dias → arquivado; Caso cancelado há 89 dias → não arquivado; notificações enviadas

### Frontend

- [x] **REQ-146** Implementar `app/(dashboard)/pipeline/page.tsx` com alternância Kanban / Lista:
  - **T-CRM-020 — Kanban**: colunas = estados do `CaseState` (exceto `CANCELADO`, `ARQUIVADO`). Cada card: `case_number`, nome empreendimento, nome Cedente (mascarado), Analista RS, badge SLA (verde/amarelo/vermelho). Drag-and-drop de cards entre colunas SÓ se transição é válida (validar no backend). Loading: skeleton de cards por coluna
  - **T-CRM-021 — Lista**: TanStack Table com colunas: ID, Estado, Empreendimento, Cedente (mascarado), Analista RS, Data criação, SLA status, Ações. Ordenação por qualquer coluna. Paginação server-side
  - Empty state (sem Casos): mensagem de encorajamento + botão "Novo Caso"
  - Validar: kanban exibe colunas na ordem correta dos 9 estados; lista suporta ordenação por SLA; filtros da barra de busca atualizam a URL (`?state=VERIFICACAO`)

- [x] **REQ-039** Implementar barra de filtros avançados no Pipeline:
  - 7 filtros: `estado` (multi-select), `assigned_to` (select Analista RS — apenas Coord/Admin), `empreendimento` (texto), `faixa de valor` (min/max), `data de abertura` (range), `SLA` (radio: Dentro do prazo / Em risco / Vencido), `motivo de cancelamento` (select)
  - Busca por texto livre: pesquisa em `case_number`, nome Cedente (mascarado), nome empreendimento
  - Filtros ativos exibidos como chips removíveis
  - Filtros persistidos na URL via query params
  - Validar: filtro `SLA=Em risco` → apenas Casos com SLA > 80%; filtro `estado=VERIFICACAO` + `SLA=Vencido` combinados funcionam; chips de filtros ativos com botão X

- [x] **REQ-148** Implementar `app/(dashboard)/cases/[id]/page.tsx` — T-CRM-022 Detalhes do Caso:
  - Seções: dados do Caso (empreendimento, valores, estado atual, SLA, Analista RS), dados do Cedente (mascarados em listagem, completos para Analista responsável/Coord/Admin), botão "Avançar estado" com nome do próximo estado, botão "Cancelar Caso" (exceto CONCLUIDO), botão "Redistribuir" (apenas Coord/Admin), botão "Retroceder estado" (apenas Coord/Admin), linha do tempo (será completada em S6), tabs: Dossiê (S5), Negociação (S4), Atividades (S6), Comunicação (S7)
  - Validar: botão "Avançar estado" desabilitado se pendências (tooltip lista pendências); botão "Retroceder" não visível para Analista RS; dados pessoais do Cedente exibidos completos para Analista responsável; Caso CANCELADO não tem botão "Avançar"

- [x] **REQ-007** Implementar fluxo de avanço de estado na UI:
  - Clique em "Avançar estado": chamar `POST /cases/:id/advance`
  - Resposta 200: toast "Caso avançado para [novo estado]." + atualizar badge de estado na tela
  - Resposta 422: modal "Pendências para avançar:" com lista de pendências itemizada (cada pendência como checkbox vermelho)
  - Validar: avanço sem pendências → estado atualizado em tempo real (Supabase Realtime); avanço com pendências → modal com lista; modal "Cancelar Caso" exige seleção de motivo antes de habilitar botão de confirmação

- [x] **REQ-041** Implementar modal de redistribuição de Caso (Coord/Admin only):
  - Listar Analistas RS ativos com contagem de Casos abertos ao lado do nome
  - Selecionar Analista → chamar `POST /cases/:id/assign`
  - Se `ANALYST_OVERLOADED`: exibir aviso "Este Analista já tem N Casos abertos. Redistribuir pode comprometer o acompanhamento. Confirmar mesmo assim?"
  - Validar: Analistas com < 30 Casos sem aviso; Analista com ≥ 30 Casos com aviso e confirmação extra; após redistribuição → toast "Caso redistribuído para [Nome]"

---

## 🧪 Testes

### Backend — Unitário (Jest)

- [x] **REQ-253** `CasesService` — cobertura 80% unitário:
  - `should create case with correct state CADASTRO`
  - `should block advance if exit condition not met for each state`
  - `should block rollback for CONCLUIDO state`
  - `should block rollback for CANCELADO state`
  - `should validate justification minimum 20 chars for rollback`
  - `should generate unique case_number per year`
  - `should apply optimistic locking (version check)`
  - `should not duplicate scenario when duplicating case`

### Backend — Integração (Supertest)

- [x] **REQ-253** Testes de integração `CasesService`:
  - `POST /cases` com payload válido → 201, state CADASTRO, case_number no formato RS-YYYY-NNNN
  - `POST /cases/:id/advance` em VERIFICACAO sem Dossiê aprovado → 422 com pendências
  - `POST /cases/:id/cancel` sem motivo → 422; com motivo → CANCELADO
  - `POST /cases/:id/rollback` por Analista RS → 403
  - `POST /cases/:id/assign` com Analista ≥ 30 Casos → 200 com warning ANALYST_OVERLOADED

### E2E (Playwright)

- [x] Fluxo E2E "Criar e avançar Caso":
  - Login como Analista RS → clicar "Novo Caso" → preencher formulário → submeter → verificar estado CADASTRO
  - Clicar "Avançar estado" sem condição → verificar modal de pendências
  - Preencher condição → avançar → verificar novo estado no kanban
  - Validar: 100% deste fluxo E2E passa sem intervenção manual

---

## 🔀 Cross-Módulo

- O worker `SlaCheckerWorker` (FEATURE 2) dispara notificações consumidas pelo módulo SLA (S8). Registrar pendência: sprint S8 deve subscrever à fila `sla-alerts.queue`.
- `POST /cases/:id/advance` para `VERIFICACAO → PUBLICACAO` depende de Dossiê aprovado — implementação completa em S5. Este gate retorna 422 com mensagem clara até S5 estar implementado.
- `POST /cases/:id/advance` para `FORMALIZACAO → CONCLUIDO` depende dos 3 critérios de Fechamento (Comissão S4 + Dossiê S5 + Integração Escrow S7). Gate implementado aqui com placeholder; lógica completa em S4/S5/S7.
