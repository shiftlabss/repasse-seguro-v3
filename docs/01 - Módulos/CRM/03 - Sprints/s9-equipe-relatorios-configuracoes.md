# S9 — Equipe, Relatórios e Configurações

## CRM Repasse Seguro

| **Campo**         | **Valor**                                                                                                                                                                                                                                                                                         |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**        | S9                                                                                                                                                                                                                                                                                                |
| **Nome**          | Equipe, Relatórios e Configurações                                                                                                                                                                                                                                                                |
| **Template**      | B — Módulo Fullstack                                                                                                                                                                                                                                                                              |
| **Docs fonte**    | 01.4, 05.1, 06, 16, 27                                                                                                                                                                                                                                                                            |
| **REQs cobertos** | REQ-083 a REQ-087, REQ-091 a REQ-096, REQ-167 a REQ-179, REQ-186 a REQ-190, REQ-225 a REQ-230, REQ-258, REQ-279, REQ-284                                                                                                                                                                          |
| **Objetivo**      | Implementar gestão completa do ciclo de vida da equipe (convite/ativação/suspensão/desligamento), 10 relatórios com controle RBAC e export CSV/XLSX, 14 parâmetros de sistema configuráveis (quatro-olhos para parâmetros críticos), interface de auditoria e gestão de templates de comunicação. |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — `crm_users`, `system_configs`, `message_templates`, `audit.audit_logs`, estados exatos de usuário (`ATIVO`/`SUSPENSO`/`DESLIGADO`), 10 relatórios com nomes exatos, 14 parâmetros exatos
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: Valores exatos — ban de 180 dias para reuso de e-mail, convite interno 48h, convite parceiro 7 dias, TTL token convite, quórum quatro-olhos para parâmetros críticos, formatos CSV e XLSX
- [x] C4: Glossário — Quatro-olhos, Suspensão vs Desligamento, ParâmetroSistema cobertos
- [x] C5: Estados de usuário: ATIVO → SUSPENSO → ATIVO (reativação) / DESLIGADO (irreversível); convite: PENDENTE → ATIVO / EXPIRADO
- [x] C6: Sem workers exclusivos desta sprint (WeeklyReportWorker implementado em S8); ban de 180 dias verificado no momento de convite
- [x] C7: RBAC — gestão de equipe: apenas `ADMIN_RS`; parâmetros críticos: quatro-olhos `ADMIN_RS`; relatórios financeiros: apenas `ADMIN_RS`; relatórios operacionais: `COORDENADOR_RS` + `ADMIN_RS`; auditoria: `ADMIN_RS`
- [x] C8: T-CRM-090 Gestão de Equipe, T-CRM-100 Relatórios, T-CRM-110 Configurações, T-CRM-112 Auditoria, T-CRM-113 Perfil — com estados loading/empty
- [x] C9: Templates de mensagem criados no seed (S1) são gerenciados aqui; convite interno de S2 consome endpoint de S9
- [x] C10: Glossário — distinção clara entre `system_configs` (parâmetros operacionais) e `message_templates` (templates WhatsApp)
- [x] C11: Anti-scaffold — lógica de ban 180 dias, lógica quatro-olhos para parâmetros críticos, lógica de export CSV/XLSX com filtros reais
- [x] C12: REQ-083 a REQ-087, REQ-091 a REQ-096, REQ-167 a REQ-179, REQ-186 a REQ-190, REQ-225 a REQ-230, REQ-258, REQ-279, REQ-284 cobertos

---

## FEATURE 1 — Gestão de Equipe

### Banco / Migrations

- [x] **REQ-083** Verificar `crm_users` com campo `status VARCHAR CHECK IN ('ATIVO','SUSPENSO','DESLIGADO')`, `suspended_at TIMESTAMPTZ NULL`, `terminated_at TIMESTAMPTZ NULL`, `invite_token VARCHAR NULL`, `invite_expires_at TIMESTAMPTZ NULL`, `invited_by UUID FK crm_users NULL`. Adicionar index em `crm_users.status`, `crm_users.email`, `crm_users.invite_token`
  - Validar: constraint de `status` funciona; indexes criados; campos nullable corretos

- [x] **REQ-084** Criar tabela `user_invite_log` para histórico de convites: `id UUID PK`, `user_id UUID FK crm_users`, `invited_by UUID FK crm_users`, `role VARCHAR`, `invite_token VARCHAR`, `sent_at TIMESTAMPTZ`, `expires_at TIMESTAMPTZ`, `accepted_at TIMESTAMPTZ NULL`, `status VARCHAR CHECK IN ('PENDENTE','ACEITO','EXPIRADO','CANCELADO')`. Index em `user_invite_log.user_id`, `user_invite_log.status`
  - Validar: registro criado ao enviar convite; status atualizado ao aceitar; index correto

### Backend

- [x] **REQ-085** Implementar `GET /team` — Roles: `ADMIN_RS`. Query params: `role`, `status`, `search` (nome ou e-mail), `page`, `per_page`. Retornar lista de `crm_users` com campos: `id`, `name`, `email` (mascarado para non-Admin: `j***@example.com`), `role`, `status`, `created_at`, `last_login_at`, `cases_count` (COUNT de Casos atribuídos ativos)
  - Validar: Coordenador RS → 403; filtro `status=SUSPENSO` funciona; `search=João` retorna usuários com nome ou e-mail contendo "João"; e-mail mascarado para non-Admin

- [x] **REQ-086** Implementar `POST /team/invite` — Roles: `ADMIN_RS`. Request body: `{ email, role, name }`. Validações:
  - Verificar ban de e-mail: buscar em `crm_users` + `user_invite_log` se e-mail foi usado por usuário `DESLIGADO` nos últimos 180 dias → 422 `CRM-039`: "Este e-mail não pode ser reusado por 180 dias após desligamento."
  - Verificar e-mail não duplicado em usuários `ATIVO` ou `SUSPENSO` → 422 `CRM-040`
  - Gerar `invite_token` (UUID v4), `invite_expires_at = NOW() + 48h` (interno) ou `NOW() + 7 dias` (Parceiro Externo, baseado no `role`)
  - Inserir em `crm_users` com `status = PENDENTE_CONVITE` e em `user_invite_log`
  - Enviar e-mail de convite com link `{BASE_URL}/accept-invite?token={invite_token}`
  - Criar `audit_log`
  - Validar: ban 180 dias → 422 CRM-039; e-mail duplicado ativo → 422 CRM-040; convite criado → e-mail enviado; token expirado → 422 ao tentar aceitar

- [x] **REQ-087** Implementar `POST /team/invite/accept` — público (sem auth). Request body: `{ token, password, confirm_password }`. Validações:
  - Token não encontrado → 404
  - Token expirado (`invite_expires_at < NOW()`) → 422 `CRM-041`: setar `status = EXPIRADO` no `user_invite_log`
  - Password mínimo 8 chars, 1 maiúscula, 1 número, 1 símbolo → 422 se inválido
  - Criar usuário no Supabase Auth com `role` como custom claim
  - Atualizar `crm_users.status = ATIVO`, `invite_token = NULL`, `invite_expires_at = NULL`
  - Atualizar `user_invite_log.status = ACEITO`, `accepted_at = NOW()`
  - Validar: token válido → usuário criado + status ATIVO; token expirado → 422 CRM-041; senha fraca → 422

- [x] **REQ-186** Implementar `PATCH /team/:userId/suspend` — Roles: `ADMIN_RS`. Request body: `{ reason: string (mínimo 20 chars) }`. Validações:
  - Não pode suspender próprio usuário → 422 `CRM-042`
  - Usuário já SUSPENSO → 422 `CRM-043`
  - Usuário DESLIGADO → 422 (irreversível)
  - Setar `status = SUSPENSO`, `suspended_at = NOW()`
  - Revogar sessões ativas no Supabase Auth (`supabase.auth.admin.signOut(userId)`)
  - Criar `audit_log` com `reason`
  - Validar: auto-suspensão → 422; suspensão válida → sessões revogadas + audit_log; Coordenador RS → 403

- [x] **REQ-187** Implementar `PATCH /team/:userId/reactivate` — Roles: `ADMIN_RS`. Validações:
  - Usuário não SUSPENSO → 422 `CRM-044`
  - Setar `status = ATIVO`, `suspended_at = NULL`
  - Criar `audit_log`
  - Validar: reativar ATIVO → 422; reativar SUSPENSO → ATIVO; reativar DESLIGADO → 422

- [x] **REQ-188** Implementar `PATCH /team/:userId/terminate` — Roles: `ADMIN_RS`. Request body: `{ reason: string (mínimo 20 chars) }`. Validações:
  - Não pode desligar próprio usuário → 422 `CRM-042`
  - Usuário já DESLIGADO → 422 (idempotente — 200 sem efeito)
  - Setar `status = DESLIGADO`, `terminated_at = NOW()`
  - Redistribuir Casos ativos do usuário: se nenhum Analista RS disponível → Casos ficam sem atribuição + alerta para Coord RS
  - Registrar e-mail + data em lista de ban (para verificação nos próximos 180 dias)
  - Revogar sessões ativas no Supabase Auth
  - Criar `audit_log` com `reason`
  - Validar: desligar ATIVO → DESLIGADO + redistribuição; Casos redistributos sem Analista disponível → ficam sem atribuição + notificação Coord RS; e-mail banido por 180 dias após desligamento

- [x] **REQ-189** Implementar `PATCH /team/:userId/role` — Roles: `ADMIN_RS`. Request body: `{ role: UserRole }`. Validações:
  - Não pode alterar próprio role → 422 `CRM-042`
  - Atualizar `crm_users.role` e custom claim no Supabase Auth (`supabase.auth.admin.updateUserById`)
  - Revogar sessões ativas (usuário deve fazer login novamente com novo token)
  - Criar `audit_log`
  - Validar: alteração de role → sessões revogadas; novo JWT tem role correto após re-login

- [x] **REQ-190** Implementar `GET /team/:userId` — Roles: `ADMIN_RS`. Retornar dados completos do usuário: perfil, histórico de status (suspensões/reativações/desligamentos de `audit_logs`), Casos atribuídos (COUNT por estado), último login, convites enviados
  - Validar: Coordenador RS → 403; dados completos com histórico de status

---

## FEATURE 2 — Relatórios

### Backend

- [x] **REQ-091 / REQ-225** Implementar `GET /reports` — Roles: `COORDENADOR_RS`, `ADMIN_RS`. Retornar lista de relatórios disponíveis com RBAC por role. Os 10 relatórios disponíveis:
  1. **Funil de Conversão** — disponível: Coord RS + Admin RS
  2. **SLA por Estado** — disponível: Coord RS + Admin RS
  3. **Produtividade por Analista** — disponível: Coord RS + Admin RS
  4. **Comissões por Período** — disponível: Coord RS + Admin RS
  5. **Comissões por Analista** — disponível: apenas Admin RS
  6. **Atividades por Tipo** — disponível: Coord RS + Admin RS
  7. **Follow-ups Vencidos** — disponível: Coord RS + Admin RS
  8. **Dossiê — Status de Aprovação** — disponível: Coord RS + Admin RS
  9. **Comunicações WhatsApp** — disponível: Coord RS + Admin RS
  10. **LGPD — Solicitações de Dados** — disponível: apenas Admin RS
  - Validar: Coord RS → não vê relatórios 5 e 10; Admin RS → vê todos os 10; Analista RS → 403

- [x] **REQ-092 / REQ-226** Implementar `POST /reports/:reportId/generate` — Roles: conforme RBAC do relatório. Request body: `{ from_date, to_date, filters?: {} }`. Executar query do relatório, retornar dados paginados + metadados (total_rows, period, generated_at, generated_by). Máx 10.000 linhas por geração. Cache Redis TTL 10 min por `reportId + filters hash`
  - Validar: RBAC correto por relatório; período obrigatório → 422 se ausente; cache hit em segunda chamada com mesmos filtros; >10.000 linhas → erro com instrução para exportar

- [x] **REQ-093 / REQ-227** Implementar `POST /reports/:reportId/export` — Roles: conforme RBAC do relatório. Request body: `{ from_date, to_date, filters?: {}, format: 'csv' | 'xlsx' }`. Gerar arquivo com `exceljs` (XLSX) ou string CSV codificada UTF-8 com BOM. Retornar como download com headers:
  - `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` (XLSX) ou `text/csv; charset=utf-8` (CSV)
  - `Content-Disposition: attachment; filename="relatorio-{reportId}-{YYYY-MM-DD}.xlsx"`
  - Criar `audit_log` com `action: 'REPORT_EXPORT'`, `metadata: { reportId, format, from_date, to_date }`
  - Validar: export CSV → arquivo com BOM UTF-8; export XLSX → arquivo válido com cabeçalhos; audit_log criado; formato inválido → 422

### Frontend

- [x] **REQ-167** Implementar `app/(dashboard)/reports/page.tsx` — T-CRM-100 Relatórios:
  - Grid de cards de relatórios filtrados pelo role do usuário (máx 10 cards)
  - Cada card: nome do relatório, descrição curta, botão "Gerar" → abre filtros laterais (DateRangePicker + filtros específicos do relatório)
  - Tabela de resultados com TanStack Table: ordenação, paginação, colunas configuráveis
  - Botão "Exportar CSV" e "Exportar XLSX" → download direto
  - Estado loading: skeleton de tabela durante geração
  - Estado empty: "Nenhum dado encontrado para os filtros selecionados."
  - Validar: Coord RS → não vê relatórios exclusivos Admin; click "Gerar" → tabela populated; "Exportar XLSX" → download iniciado

---

## FEATURE 3 — Configurações do Sistema

### Backend

- [x] **REQ-094 / REQ-228** Implementar `GET /system-configs` — Roles: `ADMIN_RS`. Retornar os 14 parâmetros de sistema com valores atuais e metadados (tipo, mín/máx, descrição, `is_critical`):
  1. `SLA_CADASTRO_DAYS` — padrão: 1, tipo: INT, mín: 1, máx: 30
  2. `SLA_SIMULACAO_DAYS` — padrão: 2, tipo: INT, mín: 1, máx: 30
  3. `SLA_VERIFICACAO_DAYS` — padrão: 5, tipo: INT, mín: 1, máx: 60
  4. `SLA_PUBLICACAO_DAYS` — padrão: 1, tipo: INT, mín: 1, máx: 30
  5. `SLA_MATCH_DAYS` — padrão: 3, tipo: INT, mín: 1, máx: 30
  6. `SLA_NEGOCIACAO_DAYS` — padrão: 7, tipo: INT, mín: 1, máx: 60
  7. `SLA_ANUENCIA_DAYS` — padrão: 10, tipo: INT, mín: 1, máx: 90
  8. `SLA_FORMALIZACAO_DAYS` — padrão: 5, tipo: INT, mín: 1, máx: 60
  9. `SLA_CYCLE_ALERT_DAYS` — padrão: 60, tipo: INT, mín: 30, máx: 365, `is_critical: true`
  10. `MAX_CASES_PER_ANALYST` — padrão: 30, tipo: INT, mín: 1, máx: 200, `is_critical: true`
  11. `ACTIVITY_RETROACTIVITY_DAYS` — padrão: 30, tipo: INT, mín: 1, máx: 365
  12. `PROPOSAL_EXPIRY_HOURS` — padrão: 48, tipo: INT, mín: 1, máx: 720, `is_critical: true`
  13. `DOSSIER_TABELA_VALIDADE_DAYS` — padrão: 30, tipo: INT, mín: 1, máx: 365
  14. `INVITE_REUSE_BAN_DAYS` — padrão: 180, tipo: INT, mín: 30, máx: 730, `is_critical: true`
  - Validar: Coordenador RS → 403; todos os 14 parâmetros retornados; `is_critical` correto nos 4 parâmetros críticos

- [x] **REQ-095 / REQ-229** Implementar `PATCH /system-configs/:key` — Roles: `ADMIN_RS`. Request body: `{ value: number, justification?: string }`. Validações:
  - Verificar mín/máx do parâmetro → 422 se fora do range com mensagem exata: "Valor fora do range permitido: mín {min}, máx {max}."
  - Se `is_critical: true`: exigir `justification` (mínimo 50 chars) → 422 `CRM-045` se ausente; exigir aprovação de segundo Admin RS (quatro-olhos): criar `pending_config_change` com status `AGUARDANDO_APROVACAO`; notificar outros Admins RS ativos
  - Se não crítico: aplicar imediatamente, criar `audit_log`
  - Invalida cache Redis das queries que dependem do parâmetro (ex: `sla:summary`, `dashboard:metrics:*`)
  - Validar: parâmetro crítico sem justificativa → 422; com justificativa → status AGUARDANDO_APROVACAO; parâmetro não crítico → aplicado imediatamente + audit_log; valor fora do range → 422

- [x] **REQ-096** Implementar `POST /system-configs/pending/:changeId/approve` e `POST /system-configs/pending/:changeId/reject` — Roles: `ADMIN_RS`. Validações:
  - Solicitante não pode aprovar própria mudança → 422 `CRM-046`
  - `approve`: aplicar mudança, `audit_log` com ambos Admin RS (solicitante + aprovador)
  - `reject`: cancelar mudança, notificar solicitante
  - Validar: solicitante aprovando própria mudança → 422 CRM-046; aprovação por outro Admin RS → mudança aplicada; rejeição → notificação ao solicitante

- [x] **REQ-230** Implementar gestão de `message_templates`: `GET /message-templates`, `POST /message-templates`, `PATCH /message-templates/:id`, `DELETE /message-templates/:id`. Roles: `ADMIN_RS`. Validações:
  - `category` deve ser `UTILITARIA`, `AUTENTICACAO`, ou `MARKETING`
  - `MARKETING`: campo `requires_opt_in: true` obrigatório
  - `status` do template: `ATIVO` / `INATIVO`
  - Ao criar/editar: registrar `audit_log`
  - Validar: template MARKETING sem `requires_opt_in: true` → 422; DELETE template usado em `communications` → 422 `CRM-047`; Coordenador RS → 403

### Frontend

- [x] **REQ-168 / REQ-169** Implementar `app/(dashboard)/team/page.tsx` — T-CRM-090 Gestão de Equipe:
  - Tabela de usuários com colunas: nome, e-mail (mascarado para non-Admin), role (badge colorido), status (ATIVO/SUSPENSO/DESLIGADO), último acesso, Casos ativos, ações (suspender/reativar/desligar/alterar role)
  - Filtros: role, status, busca por nome/e-mail
  - Botão "Convidar membro" → modal com campos: nome, e-mail, role
  - Modal de confirmação para suspender/desligar com campo de justificativa (mínimo 20 chars)
  - Estado loading: skeleton de tabela
  - Estado empty: "Nenhum membro encontrado."
  - Validar: suspender sem justificativa → botão bloqueado; convidar e-mail banido → toast de erro; role alterado → badge atualizado na tabela

- [x] **REQ-170 / REQ-171** Implementar `app/(dashboard)/settings/page.tsx` — T-CRM-110 Configurações do Sistema:
  - Tabs: "Parâmetros SLA", "Parâmetros Operacionais", "Templates de Mensagem", "Auditoria"
  - **Parâmetros**: form com inputs numéricos para cada parâmetro, valor atual destacado, mín/máx exibidos como hint, badge "Crítico" para os 4 parâmetros críticos
  - Parâmetros críticos: campo de justificativa obrigatório (contador de chars, mínimo 50); botão "Solicitar alteração" (não "Salvar") → toast "Aguardando aprovação de outro Admin RS"
  - Parâmetros não críticos: botão "Salvar" → aplicação imediata + toast de sucesso
  - Aprovação pendente: banner informativo "Você tem N alterações aguardando aprovação" com lista de pendências e botões "Aprovar" / "Rejeitar"
  - **Templates de Mensagem**: tabela com nome, categoria (badge), status, ações (editar/ativar/inativar)
  - Validar: parâmetro crítico com justificativa < 50 chars → botão bloqueado; aprovação própria → toast de erro; template MARKETING sem opt-in → erro no form

- [x] **REQ-172** Implementar `app/(dashboard)/settings/audit/page.tsx` — T-CRM-112 Log de Auditoria:
  - Tabela de `audit.audit_logs` com colunas: data/hora, usuário, ação, entidade, entidade ID, IP, detalhes (expandível)
  - Filtros: usuário, ação (CASE_CREATED/CASE_STATE_CHANGE/ACTIVITY_CREATED/COMMISSION_APPROVED/etc.), entidade, período
  - Export CSV e XLSX
  - Somente leitura — nenhuma modificação possível
  - Validar: filtro por ação funciona; export CSV gera arquivo válido; Coord RS → 403

- [x] **REQ-173** Implementar `app/(dashboard)/profile/page.tsx` — T-CRM-113 Perfil do Usuário:
  - Formulário: nome, e-mail (somente leitura), avatar (upload Supabase Storage, máx 2MB, JPG/PNG)
  - Seção "Alterar senha": campos senha atual, nova senha, confirmação (validação Zod em tempo real)
  - Seção "Preferências": idioma (apenas pt-BR), fuso horário (exibição apenas — America/Fortaleza fixo para regras de negócio)
  - Seção "Sessões ativas": listar sessões com IP, dispositivo, última atividade; botão "Encerrar esta sessão"
  - Validar: upload avatar > 2MB → erro; senha nova < 8 chars → erro em tempo real; encerrar sessão → revogação no Supabase Auth

---

## 🧪 Testes

### Backend — Unitário (Jest)

- [x] **REQ-258** `TeamService` — cobertura 70%:
  - `should block invite for email banned within 180 days`
  - `should block invite for duplicate active email`
  - `should expire invite after 48h (internal) or 7 days (partner)`
  - `should prevent self-suspension and self-termination`
  - `should require justification for critical config change`
  - `should prevent requester from approving own config change`
  - `should redistribute cases on termination`

### Backend — Integração

- [x] **REQ-279** Testes de integração Equipe + Configurações:
  - `POST /team/invite` com e-mail banido (desligamento < 180 dias) → 422 CRM-039
  - `PATCH /team/:userId/terminate` → Casos redistribuídos + `audit_log` criado
  - `PATCH /system-configs/SLA_CYCLE_ALERT_DAYS` sem justificativa → 422 CRM-045
  - `POST /system-configs/pending/:id/approve` por solicitante → 422 CRM-046
  - `POST /reports/5/generate` por Coordenador RS → 403

### Frontend — E2E (Playwright)

- [x] **REQ-284** E2E: Gestão de Equipe:
  - Login Admin RS → `/team` → convidar novo Analista RS → verificar e-mail de convite (mock)
  - Aceitar convite via `/accept-invite?token=...` → usuário ATIVO
  - Suspender Analista RS → justificativa obrigatória → confirmação → status SUSPENSO
  - Reativar Analista RS suspenso → status ATIVO

---

## 🔀 Cross-Módulo

- O endpoint `POST /team/invite` é o mesmo usado em S2 para convite interno (S2 implementou o fluxo de convite básico; S9 implementa o endpoint completo com ban 180 dias e quatro-olhos).
- Os 14 parâmetros em `system_configs` são lidos pelos workers de S3 (SlaCheckerWorker lê `SLA_*_DAYS`), S4 (ProposalExpirationWorker lê `PROPOSAL_EXPIRY_HOURS`), S5 (DossierService lê `DOSSIER_TABELA_VALIDADE_DAYS`), S6 (ActivitiesService lê `ACTIVITY_RETROACTIVITY_DAYS`), S9 (TeamService lê `MAX_CASES_PER_ANALYST` e `INVITE_REUSE_BAN_DAYS`). Ao alterar parâmetro → cache Redis dos serviços afetados deve ser invalidado.
- O export XLSX de relatórios usa `exceljs` — adicionar à dependência de `packages/shared` no Turborepo.
- T-CRM-113 (Perfil) referenciado em S2 como "placeholder" é implementado completamente aqui.
