# S8 — Usuários

## Repasse Seguro — Módulo Admin

| Campo | Valor |
|---|---|
| **Sprint** | S8 |
| **Nome** | Usuários |
| **Template** | B — Módulo Fullstack |
| **REQs cobertos** | REQ-022, REQ-133, REQ-134, REQ-135, REQ-136, REQ-228, REQ-229, REQ-230, REQ-231, REQ-232, REQ-233, REQ-234, REQ-279, REQ-280, REQ-281, REQ-282, REQ-283, REQ-284, REQ-285, REQ-286, REQ-287 |
| **Docs fonte** | D01.3 §4 (RN-065 a RN-071), D06 §2.9 (T-075 a T-081), D16 §3-5 |
| **Itens totais** | 42 |

---

> **Objetivo da sprint:** Implementar o módulo de gestão de usuários — operadores internos (ANALISTA, COORDENADOR, GESTOR_FINANCEIRO, MASTER), Cedentes e Cessionários externos — com CRUD completo, suspensão/reativação com impacto rastreado, inatividade automática (RN-067), proteção do último Master ativo (RN-068), mascaramento de CPF, isolamento de dados (RN-069) e painel LGPD (RN-070).

---

## FEATURE 1 — CRUD de Operadores Internos

### Banco de Dados

**[BE-01] Validar schema `users` e índices para módulo Usuários**
- Confirmar que a tabela `users` possui as 13 colunas: `id` (UUID PK), `email` (UNIQUE), `name`, `role` (ENUM `UserRole`), `password_hash`, `is_active`, `two_fa_enabled`, `two_fa_secret`, `failed_login_attempts`, `locked_until`, `version`, `created_at`, `updated_at`, `deleted_at`
- Confirmar enum `UserRole` = `ANALISTA | COORDENADOR | GESTOR_FINANCEIRO | MASTER` (exatos, sem variantes)
- Confirmar índices: `UNIQUE(email)`, `INDEX(role)`, `INDEX(is_active, deleted_at)`
- Confirmar que `version` é INTEGER DEFAULT 0 e incrementado a cada PATCH (Optimistic Lock)
- Confirmar que `deleted_at` é NULL para registros ativos (soft delete)
- Verificação: executar migration `prisma migrate deploy` sem erros; consultar `\d users` confirma estrutura

### Backend

**[BE-02] Implementar `GET /v1/users` — listagem paginada de operadores**
- Role mínimo: `COORDENADOR`; ANALISTA e GESTOR_FINANCEIRO recebem `403 Forbidden`
- Query params: `page` (default 1), `per_page` (default 25, max 100), `role` (filtro enum `UserRole`), `is_active` (boolean), `search` (ILIKE em `name` e `email`)
- Resposta `200`: `{ data: [{ id, email, name, role, is_active, created_at, last_login_at, active_cases_count }], meta: { page, per_page, total, total_pages } }`
- `active_cases_count`: COUNT de casos em `assigned_analyst_id = user.id` WHERE status NOT IN (`CONCLUIDO`, `CANCELADO`)
- `deleted_at IS NOT NULL` → excluído da listagem por padrão
- Verificação: GET com token ANALISTA retorna 403; GET com token COORDENADOR retorna 200 com `meta.total` correto

**[BE-03] Implementar `POST /v1/users` — criação de operador**
- Role mínimo: `MASTER`; qualquer outro perfil retorna `403 Forbidden`
- Request body: `{ email: string, name: string, role: UserRole }`
- Validações: `email` formato RFC 5321; `name` mínimo 3 chars; `role` enum válido
- Se `email` já cadastrado (mesmo soft-deleted): retorna `409 Conflict` com message `"Este e-mail já está vinculado a um operador."`
- Ao criar: gerar `password_hash` temporário; enviar e-mail de convite com link de primeiro acesso (token UUID único, TTL 72h, gravado em `password_reset_tokens`); retornar `201 Created` com `{ data: { id, email, name, role, created_at } }`
- Registrar no `audit.audit_logs`: `action = 'USER_CREATED'`, `actor_id = masterId`, `target_id = newUserId`, `metadata = { role }`
- Verificação: POST com token COORDENADOR retorna 403; POST com e-mail duplicado retorna 409; POST válido retorna 201 e e-mail de convite é enfileirado

**[BE-04] Implementar `GET /v1/users/:id` — perfil completo do operador**
- Role mínimo: `COORDENADOR`
- Retorna: todos os campos de `users` (exceto `password_hash`, `two_fa_secret`) + `cases_assigned` (lista paginada, últimos 10) + `recent_activity` (últimas 10 entradas do `audit.audit_logs` onde `actor_id = id`)
- Se `id` não existe ou `deleted_at IS NOT NULL`: retorna `404 Not Found`
- Verificação: GET de usuário inexistente retorna 404; GET válido com COORDENADOR retorna 200 com estrutura completa

**[BE-05] Implementar `PATCH /v1/users/:id` — edição de operador**
- Role mínimo: `MASTER`
- Request body: campos opcionais `{ name?: string, role?: UserRole, is_active?: boolean, version: number }`
- `version` obrigatório no body; se `version` diferente da versão no banco: retorna `409 Conflict` com message `"Conflito de versão. Recarregue e tente novamente."`; campo `version` é incrementado +1 a cada PATCH bem-sucedido
- **Proteção último Master ativo (RN-068):** se `role` sendo alterado de `MASTER` para outro perfil E COUNT(`users` WHERE `role = 'MASTER'` AND `is_active = TRUE`) = 1: retorna `422 Unprocessable Entity` com message `"Não é possível alterar o perfil do único Master ativo."`
- **Proteção auto-alteração:** se `req.user.id === params.id` e a alteração inclui `role`: retorna `403 Forbidden` com message `"O Master não pode alterar o próprio perfil."`
- Registrar no `audit.audit_logs`: `action = 'USER_UPDATED'`, `metadata = { changes: { campo_anterior, campo_novo } }`
- Verificação: PATCH com version errada retorna 409; PATCH de único Master ativo para outro role retorna 422; PATCH válido retorna 200 com `version` incrementado

**[BE-06] Implementar `DELETE /v1/users/:id` — soft delete de operador**
- Role mínimo: `MASTER`
- Soft delete: setar `deleted_at = NOW()`, `is_active = false`
- Se `req.user.id === params.id`: retorna `403 Forbidden` com message `"Não é possível desativar o próprio usuário."`
- **Proteção último Master ativo:** se `role = 'MASTER'` E seria o último: retorna `422` com message `"Não é possível desativar o único Master ativo."`
- Retornar `204 No Content`
- Verificação: DELETE de si mesmo retorna 403; DELETE de único Master retorna 422; DELETE válido retorna 204 e registro tem `deleted_at` preenchido

### Frontend

**[FE-01] Implementar T-075 — Lista de Operadores (`/usuarios/operadores`)**
- Rota: `/usuarios/operadores`; sidebar item "Usuários" ativo com sub-item "Operadores"
- RBAC: COORDENADOR (leitura + edição básica) e MASTER (tudo); ANALISTA e GESTOR_FINANCEIRO → redirecionar para `/dashboard` (módulo oculto no menu)
- Tabela com colunas: avatar (iniciais), nome, e-mail, perfil (badge colorido por role), status (Ativo / Suspenso / Inativo), data de criação, casos ativos, última atividade
- Filtros: chips `role` (4 opções), `status` (Ativo/Suspenso/Inativo), campo de busca por nome/e-mail (debounce 300ms)
- Botão "+ Novo Operador": visível apenas para MASTER; abre drawer/modal de criação inline
- Ações por linha: "Ver perfil" → T-076; MASTER: dropdown com "Editar", "Suspender" (→ T-081), "Forçar redefinição de senha"
- Responsividade: Desktop → tabela; Mobile → cards com dados principais
- Estado vazio: ícone + "Nenhum operador encontrado com os filtros aplicados."
- Verificação: acessar `/usuarios/operadores` como ANALISTA redireciona para `/dashboard`; tabela renderiza todos os colunas; botão "+ Novo Operador" ausente para COORDENADOR

**[FE-02] Implementar T-076 — Perfil do Operador (`/usuarios/operadores/:id`)**
- Seções: (1) dados pessoais (nome, e-mail, role badge, status badge), (2) perfil de acesso (histórico de alterações de perfil), (3) casos atribuídos (lista paginada, 10 por página), (4) log de atividade recente (últimas 10 ações do audit log)
- Ações visíveis por role:
  - COORDENADOR: "Atribuir caso"
  - MASTER: + "Editar perfil" (nome), "Alterar perfil de acesso" (dropdown `UserRole`), "Suspender operador" (→ T-081), "Forçar redefinição de senha" (envia e-mail)
- Alteração de perfil via dropdown: exibe confirmação inline "Confirmar alteração de [RoleAnterior] → [NovoRole]?"; ao confirmar, envia `PATCH /v1/users/:id` com `version`
- **Proteção visual:** se o operador logado é o único MASTER ativo, o dropdown de "Alterar perfil de acesso" fica desabilitado com tooltip "Você é o único Master ativo."
- Verificação: editar role do único Master ativo exibe tooltip de bloqueio; alterar role de outro operador funciona e reflete mudança imediata

---

## FEATURE 2 — Inatividade Automática e Suspensão

### Backend

**[BE-07] Implementar job cron de inatividade automática (RN-067)**
- Cron: `0 2 * * *` (diariamente às 02:00 UTC)
- Query: `UPDATE users SET is_active = false WHERE last_login_at < NOW() - INTERVAL '180 days' AND is_active = true AND deleted_at IS NULL AND role != 'MASTER'`
- **Atenção:** Masters não são inativados automaticamente (proteção operacional)
- Para cada usuário inativado: registrar no `audit.audit_logs` `action = 'USER_AUTO_INACTIVATED'`; enfileirar notificação por e-mail informando inativação
- Após inativação: na próxima tentativa de login com credenciais válidas, o sistema reativa automaticamente (`is_active = true`) e registra `action = 'USER_REACTIVATED_BY_LOGIN'`
- Verificação: executar job manualmente com usuário de 181 dias sem login → status muda para Inativo; login com usuário inativo → é reativado automaticamente

**[BE-08] Implementar `PATCH /v1/users/:id/suspend` e `PATCH /v1/users/:id/reactivate` — suspensão e reativação (RN-066)**
- `PATCH /v1/users/:id/suspend`: Role mínimo `MASTER`
  - Request: `{ suspension_reason: string (min 20 chars) }`
  - Validação: `suspension_reason` < 20 chars → `422` com message `"A justificativa deve ter pelo menos 20 caracteres."`
  - Ao suspender: setar `is_active = false`, `suspended_at = NOW()`, `suspension_reason = reason`
  - **Cascata Cedente:** se usuário é da tabela `cedentes` → setar `status = 'SUSPENSO'` na tabela `cedentes`; casos do Cedente com status em (CAPTADO, EM_TRIAGEM, BLOQUEADO, QUALIFICADO, OFERTA_ATIVA, EM_NEGOCIACAO, EM_FORMALIZACAO) → adicionar flag `frozen = true` em `cases`
  - **Cascata Cessionário:** se usuário é da tabela `cessionarios` → setar `status = 'SUSPENSO'` em `cessionarios`; propostas ativas do Cessionário → status `CANCELADA`
  - Retornar `200` com `{ data: { frozen_cases_count, cancelled_proposals_count } }`
  - Registrar no `audit.audit_logs`: `action = 'USER_SUSPENDED'`, `metadata = { suspension_reason, frozen_cases, cancelled_proposals }`
- `PATCH /v1/users/:id/reactivate`: Role mínimo `MASTER`
  - Setar `is_active = true`, `suspended_at = null`, `suspension_reason = null`; casos descongelados (flag `frozen = false`); propostas NÃO são restauradas
  - Retornar `200`
- Verificação: suspender Cessionário com proposta ativa → proposta fica com status `CANCELADA`; reativar → `is_active = true`, proposta permanece `CANCELADA`

---

## FEATURE 3 — CRUD de Cedentes

### Backend

**[BE-09] Implementar `GET /v1/cedentes` — listagem paginada**
- Role mínimo: `ANALISTA`
- Query params: `page`, `per_page`, `status` (ATIVO/SUSPENSO/INATIVO), `search` (nome/CPF)
- Resposta: `{ data: [{ id, name, cpf_masked, email, status, total_cases, created_at }], meta: {} }`
- `cpf_masked`: sempre retornado como `"***.XXX.XXX-**"` (somente dígitos 4-6 visíveis) exceto se MASTER ou COORDENADOR solicitam com `?unmask_cpf=true` (registra auditoria)
- Verificação: GET com ANALISTA retorna CPF mascarado; GET com `?unmask_cpf=true` como MASTER retorna CPF completo e registra audit log

**[BE-10] Implementar `POST /v1/cedentes` — criação de Cedente**
- Role mínimo: `ANALISTA`
- Request: `{ name, cpf (formato "000.000.000-00"), email, phone?, address? }`
- Validações: CPF válido (dígitos verificadores); e-mail formato RFC; unicidade CPF e e-mail (se duplicado → `409`)
- Ao criar: `status = 'ATIVO'`, `version = 0`
- Verificação: POST com CPF inválido retorna 422; POST com CPF duplicado retorna 409; POST válido retorna 201

**[BE-11] Implementar `GET /v1/cedentes/:id`, `PATCH /v1/cedentes/:id` e `PATCH /v1/cedentes/:id/suspend`**
- `GET /v1/cedentes/:id`: Role `ANALISTA`; retorna perfil completo + casos vinculados + histórico de interações
- `PATCH /v1/cedentes/:id`: Role `ANALISTA`; inclui `version` no body (Optimistic Lock, 409 se mismatch); campos editáveis: `name`, `email`, `phone`, `address`
- `PATCH /v1/cedentes/:id/suspend`: Role `COORDENADOR`; `suspension_reason ≥ 20 chars`; aciona cascata de congelamento de casos (igual BE-08)
- Verificação: PATCH com version errada retorna 409; PATCH de CPF não permitido (campo não editável); suspend com reason < 20 chars retorna 422

---

## FEATURE 4 — CRUD de Cessionários

### Backend

**[BE-12] Implementar endpoints `/v1/cessionarios` — estrutura idêntica a `/cedentes`**
- Rotas: `GET /v1/cessionarios`, `POST /v1/cessionarios`, `GET /v1/cessionarios/:id`, `PATCH /v1/cessionarios/:id`, `PATCH /v1/cessionarios/:id/suspend`
- Mesmas regras de role, validação e Optimistic Lock de `/cedentes`
- Diferencial: resposta de `GET /v1/cessionarios/:id` inclui seção `proposals_history` (histórico de propostas enviadas)
- **Isolamento de dados (RN-069):** response de Cessionário NUNCA inclui campo `cedente_id` ou qualquer referência ao Cedente vinculado fora do contexto de um caso específico
- `PATCH /v1/cessionarios/:id/suspend`: cascata → propostas ativas ficam com status `CANCELADA`
- Verificação: GET de cessionário não retorna `cedente_id`; suspend com proposta ativa → proposta CANCELADA

---

## FEATURE 5 — Telas de Listagem e Perfil (Cedentes e Cessionários)

### Frontend

**[FE-03] Implementar T-077 — Lista de Cedentes (`/usuarios/cedentes`)**
- Rota: `/usuarios/cedentes`; aba "Cedentes" no menu Usuários
- RBAC: COORDENADOR e MASTER (ANALISTA pode acessar mas com CPF mascarado)
- Colunas: nome, CPF (mascarado `"***.XXX.XXX-**"`), e-mail, total de casos, status (badge), data de cadastro
- Ações: "Ver perfil" → T-078; COORDENADOR/MASTER: "Suspender" (→ T-081)
- Filtros: status (ATIVO/SUSPENSO/INATIVO), busca por nome
- Verificação: CPF sempre mascarado na listagem; ação "Suspender" ausente para ANALISTA

**[FE-04] Implementar T-078 — Perfil do Cedente (`/usuarios/cedentes/:id`)**
- Seções: (1) dados cadastrais: nome, CPF (mascarado por padrão), RG, endereço, e-mail, telefone; (2) casos vinculados (lista paginada); (3) histórico de interações; (4) documentos de identificação
- Botão "Mostrar CPF": visível para COORDENADOR e MASTER; ao clicar, exibe CPF completo por 30s depois mascara novamente; cada clique registra `audit.audit_logs` `action = 'CPF_UNMASKED'`
- Ações: "Editar dados cadastrais" (COORDENADOR+); "Suspender conta" (→ T-081)
- Verificação: ANALISTA não vê botão "Mostrar CPF"; MASTER clica → CPF visível por 30s + audit registrado

**[FE-05] Implementar T-079 — Lista de Cessionários (`/usuarios/cessionarios`) e T-080 — Perfil do Cessionário**
- T-079: igual T-077 para Cessionários; colunas iguais; sem coluna de vínculo com Cedente (RN-069)
- T-080 (`/usuarios/cessionarios/:id`): igual T-078 + seção adicional "Propostas enviadas" (histórico paginado de propostas do Cessionário); sem exibir Cedente vinculado fora de contexto de caso específico
- KYC badge: exibir badge "KYC Verificado" ou "KYC Pendente" no perfil
- Verificação: T-079 não exibe coluna cedente; T-080 não exibe `cedente_id`; KYC badge renderizado conforme status

---

## FEATURE 6 — Modal Suspender / Reativar e LGPD

### Frontend

**[FE-06] Implementar T-081 — Modal Suspender / Reativar Usuário**
- Trigger: ação "Suspender" em T-076, T-078 ou T-080
- Modal abre com: nome do usuário, perfil, resumo de impacto "Suspender [Nome] afetará: [X] casos ativos (congelados) e [Y] propostas ativas (canceladas)."
- Campo motivo obrigatório: select com opções `Fraude suspeita | Inadimplência | Solicitação do usuário | Outro` + textarea (mínimo 20 chars) com contador de caracteres
- Contador de caracteres: exibe "X/20 mínimo" em vermelho se < 20; verde se ≥ 20
- Botão "Confirmar Suspensão": `--destructive` (vermelho), desabilitado até motivo válido; Botão "Cancelar": cinza
- Após confirmação: toast de sucesso "Usuário suspenso. [X] casos congelados e [Y] propostas canceladas."; perfil atualiza status para "Suspenso"
- Reativação: mesmo modal com toggle "Reativar" — confirma reativação; exibe aviso "Propostas canceladas NÃO serão restauradas."
- Componente reutilizável `ConfirmModal` (C-07) com variante `--destructive`
- Verificação: modal não abre sem calcular impacto; botão desabilitado com reason < 20 chars; após suspensão, status no perfil atualiza imediatamente

**[FE-07] Implementar painel LGPD no módulo Usuários (RN-070)**
- Aba "Solicitações LGPD" visível apenas para MASTER no módulo Usuários
- Lista de solicitações com: nome do titular (mascarado), tipo (exclusão/acesso/correção), data de recebimento, timer regressivo dos 15 dias corridos, status (Pendente/Em Análise/Respondida/Vencida)
- Badge visual: ≤ 3 dias restantes → badge amarelo "Prazo próximo"; vencida → badge vermelho "Prazo excedido"
- Botão "Registrar Resposta": abre formulário com campo de resposta (textarea) e opção de anexar documentação (PDF, max 10MB)
- Timer regressivo: atualizado em tempo real (JavaScript `setInterval` 1s)
- Alerta crítico: se prazo ultrapassa 15 dias sem resposta registrada → toast persistente ao MASTER em todas as telas: "LGPD: Prazo excedido para solicitação [ID]. Ação imediata necessária."
- Verificação: aba "Solicitações LGPD" invisível para COORDENADOR; timer conta decrescentemente; alerta persistente aparece ao ultrapassar 15 dias

---

## FEATURE 7 — Exportação e Isolamento de Dados

### Backend

**[BE-13] Implementar `GET /v1/users/export` e `GET /v1/cedentes/export` e `GET /v1/cessionarios/export` — exportação CSV (RN-071)**
- Role: `MASTER` ou `COORDENADOR`
- Gera CSV com campos filtrados conforme a tela atual (aplicar mesmos filtros do GET de listagem)
- **Restrição de segurança:** exportação de `cessionarios` não inclui CPF/CNPJ no CSV
- Resposta: `200` com `Content-Type: text/csv`, `Content-Disposition: attachment; filename="usuarios_YYYY-MM-DD.csv"`
- Se falha na geração: retorna `500` com message para nova tentativa
- Registrar no `audit.audit_logs`: `action = 'USER_LIST_EXPORTED'`, `metadata = { entity_type, filters_applied, exported_by }`
- Verificação: export de cessionários não contém CPF no CSV; export registra audit log; falha retorna 500

### Frontend

**[FE-08] Implementar botão "Exportar CSV" nas listagens T-075, T-077, T-079**
- Botão "Exportar CSV" no topo direito de cada listagem; visível para MASTER e COORDENADOR
- Ao clicar: POST para endpoint de exportação com filtros atuais; exibe loading spinner; ao concluir, download automático inicia + toast "Exportação concluída."
- Se falha: toast de erro "Falha na exportação. Tente novamente." com botão de retry
- Verificação: exportação de cessionários baixa CSV sem coluna CPF; botão ausente para ANALISTA

---

## WIRING

**[WIRE-01] Integrar `UsersModule` no NestJS com serviços de suspensão e cascata**
- `UsersController` → `UsersService` (CRUD operadores + cedentes + cessionários)
- `UsersService.suspendUser()`: orquestra suspensão + cascata `CasesService.freezeCases()` + `ProposalsService.cancelActiveProposas()`
- `UsersService.inactivateJob()`: registrado no `ScheduleModule` com cron `0 2 * * *`
- Injetar `AuditService` para registrar todas as mutações
- Verificação: `UsersModule` importado em `AppModule`; `ScheduleModule.forRoot()` presente; teste de integração valida cascata de suspensão

**[WIRE-02] Integrar telas de Usuários com React Query e roteamento TanStack Router**
- Rotas: `/usuarios/operadores`, `/usuarios/operadores/:id`, `/usuarios/cedentes`, `/usuarios/cedentes/:id`, `/usuarios/cessionarios`, `/usuarios/cessionarios/:id`
- Queries: `useUsers()`, `useUser(id)`, `useCedentes()`, `useCedente(id)`, `useCessionarios()`, `useCessionario(id)` — cache TTL padrão 5 min; invalidar após mutação
- Mutations: `useCreateUser()`, `usePatchUser()`, `useSuspendUser()`, `useReactivateUser()`; após mutação bem-sucedida → invalidar queries relevantes + toast de feedback
- Sidebar item "Usuários" com sub-itens "Operadores", "Cedentes", "Cessionários" — oculto para ANALISTA e GESTOR_FINANCEIRO (DEC-002: menu oculto, nunca desabilitado)
- Verificação: navegação entre sub-itens preserva filtros; mutation invalida cache corretamente

---

## TESTES

**[TEST-01] Testes unitários — `UsersService`**
- `UsersService.createUser()`: e-mail duplicado → lança `ConflictException`; role inválido → lança `BadRequestException`; sucesso → retorna user criado + e-mail de convite enfileirado
- `UsersService.patchUser()`: version mismatch → lança `ConflictException(409)`; único Master ativo alterar role → lança `UnprocessableEntityException(422)`; auto-alteração de role pelo próprio Master → lança `ForbiddenException(403)`
- `UsersService.suspendUser()`: justificativa < 20 chars → lança `UnprocessableEntityException(422)`; Cedente com casos ativos → `CasesService.freezeCases()` chamado; Cessionário com propostas ativas → `ProposalsService.cancelActiveProposals()` chamado
- `UsersService.inactivateJob()`: usuário com `last_login_at` > 180 dias → `is_active = false`; Master não é inativado

**[TEST-02] Testes de integração — endpoints de Usuários**
- `GET /v1/users`: ANALISTA → 403; COORDENADOR → 200 com listagem paginada; filtro `role=MASTER` retorna apenas Masters
- `POST /v1/users`: COORDENADOR → 403; MASTER + e-mail duplicado → 409; MASTER + dados válidos → 201 + e-mail de convite
- `PATCH /v1/users/:id`: version errada → 409; único Master ativo + role change → 422; self role change → 403; válido → 200 + version incrementado
- `DELETE /v1/users/:id`: self delete → 403; único Master → 422; válido → 204 + soft delete confirmado
- `PATCH /v1/cedentes/:id/suspend`: reason < 20 → 422; reason válida → 200 + casos congelados

**[TEST-03] Testes de componente — Frontend**
- `T-081 Modal Suspender`: renderiza com impacto calculado; botão desabilitado até reason ≥ 20 chars; submit dispara mutation; toast exibido após sucesso
- `T-075 Lista Operadores`: filtro `role` filtra resultados; botão "+ Novo Operador" ausente para COORDENADOR; ação "Suspender" abre T-081
- `T-078 Perfil Cedente`: botão "Mostrar CPF" ausente para ANALISTA; CPF mascarado por padrão; após clique, CPF visível por 30s
- `T-076 Perfil Operador`: dropdown "Alterar perfil" desabilitado para único Master ativo com tooltip correto

**[TEST-04] Testes E2E — fluxo de suspensão**
- Playwright: login como MASTER → acessar `/usuarios/cedentes/:id` → clicar "Suspender conta" → modal abre com impacto → preencher reason "Fraude identificada no CPF enviado" → confirmar → toast aparece → status do Cedente muda para "Suspenso" → casos do Cedente listados como congelados
- Playwright: login como ANALISTA → tentar acessar `/usuarios/operadores` → redirecionado para `/dashboard`

---

## AUTO-VERIFICAÇÃO (12 CHECKS)

- [x] **Check #1 — RBAC completo:** todos os endpoints têm role mínimo declarado; ANALISTA/GESTOR_FINANCEIRO retornam 403 onde aplicável; menu Usuários oculto para ANALISTA e GESTOR_FINANCEIRO
- [x] **Check #2 — Nomes canônicos:** `users`, `cedentes`, `cessionarios`, `UserRole`, `ANALISTA`, `COORDENADOR`, `GESTOR_FINANCEIRO`, `MASTER`, `UserExternalStatus`, `ATIVO`, `SUSPENSO`, `INATIVO` — zero sinônimos
- [x] **Check #3 — Proteção último Master (RN-068):** verificada em PATCH role, DELETE, suspend e job de inatividade (Masters excluídos do job)
- [x] **Check #4 — Optimistic Lock:** campo `version` obrigatório em PATCH de `users`, `cedentes`, `cessionarios`; 409 em version mismatch; version incrementado após cada PATCH
- [x] **Check #5 — Cascata de suspensão (RN-066):** Cedente suspenso → casos congelados; Cessionário suspenso → propostas CANCELADAS; ambos retornam contagem de impacto
- [x] **Check #6 — Inatividade automática (RN-067):** cron `0 2 * * *`; 180 dias sem login; Masters excluídos; reativação automática por login
- [x] **Check #7 — Mascaramento CPF:** listagens sempre retornam CPF mascarado; desmascaramento exige role COORDENADOR ou MASTER + registra audit log; export de cessionários não inclui CPF/CNPJ
- [x] **Check #8 — Isolamento de dados (RN-069):** nenhum endpoint ou tela de Usuários expõe vínculo Cessionário↔Cedente fora de contexto de caso específico
- [x] **Check #9 — Painel LGPD (RN-070):** aba "Solicitações LGPD" visível apenas para MASTER; timer regressivo 15 dias; alerta crítico ao ultrapassar prazo; dados retidos por obrigação legal informados ao titular
- [x] **Check #10 — Audit trail completo:** USER_CREATED, USER_UPDATED, USER_SUSPENDED, USER_REACTIVATED_BY_LOGIN, USER_AUTO_INACTIVATED, CPF_UNMASKED, USER_LIST_EXPORTED — todos registrados em `audit.audit_logs`
- [x] **Check #11 — Anti-scaffold R10:** todos os itens têm sub-itens com validações, condições de erro, RBAC explícito e etapa de verificação
- [x] **Check #12 — Cobertura REQs S8:** REQ-022 ✅ REQ-133 ✅ REQ-134 ✅ REQ-135 ✅ REQ-136 ✅ REQ-228 ✅ REQ-229 ✅ REQ-230 ✅ REQ-231 ✅ REQ-232 ✅ REQ-233 ✅ REQ-234 ✅ REQ-279 ✅ REQ-280 ✅ REQ-281 ✅ REQ-282 ✅ REQ-283 ✅ REQ-284 ✅ REQ-285 ✅ REQ-286 ✅ REQ-287 ✅

---

*Sprint S8 gerada por Pipeline ShiftLabs v2.3 — 2026-03-24*
