# S3 — Dashboard e Perfil

## Sprint 3 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S3 — Dashboard e Perfil                                                                                                                           |
| **Template**       | B — Módulo Fullstack (feature com vertical slice Banco→Backend→Frontend→Wiring→Testes)                                                            |
| **REQs cobertos**  | S3-001 a S3-018 (18 requisitos do Registro Mestre)                                                                                                |
| **Docs fonte**     | 01.3 - RN Operação e Suporte · 01.4 - RN Administração · 06 - Mapa de Telas · 09 - Contratos de UI · 16 - Documentação de API · 21 - Notificações |
| **Total de itens** | 46 itens                                                                                                                                          |
| **Status**         | Concluída                                                                                                                                         |

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos de widgets, endpoints, telas e campos dos docs.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — 5 widgets do Dashboard, 5 seções do Perfil, 17 notificações NOT-CES-01 a NOT-CES-17 exatos.
- [x] ✅ Check 4 — Glossário: `CessionarioStatus`, `in-app notification`, `Realtime`, `Skeleton Screen` usados corretamente.
- [x] ✅ Check 5 — Anti-scaffold R10: componentes com lógica real, não apenas estrutura HTML.
- [x] ✅ Check 6 — Máquinas de estado de notificação (lida/não lida) e preferências de canais documentadas.
- [x] ✅ Check 7 — Cache `rs:cessionario:{id}:perfil` TTL 15min aplicado.
- [x] ✅ Check 8 — Cross-módulo: Dashboard consome dados de proposals, negotiations, notifications.
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — Sem ambiguidades.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S3-001 a S3-018 cobertos.

---

## FEATURE 1 — Dashboard (T-DASH-01)

### 🗄️ Banco

- [x] **S3-B01** · Verificar que `notifications` table existe com campos `id`, `cessionario_id`, `type VARCHAR(50)`, `title VARCHAR(200)`, `body TEXT`, `metadata JSONB`, `read BOOLEAN DEFAULT false`, `read_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ`. Índice parcial `idx_notifications_cessionario_unread WHERE read = false`. Confirmar RLS: apenas o próprio cessionário lê suas notificações. (Doc 13 — model Notification)

### ⚙️ Backend

- [x] **S3-BE01** · Implementar `GET /api/v1/dashboard` em `apps/api/src/modules/dashboard/`: guard `JwtAuthGuard` + `KycGuard`; retornar objeto com 5 widgets: (1) `proposals_summary: { sent: number, under_analysis: number, accepted: number, rejected: number, expired: number }` — contagem de propostas do cessionário por status; (2) `active_negotiations: { count: number, items: [{ negotiation_id, opportunity_code, status, escrow_deadline? }] }` — negociações ativas; (3) `pending_escrow: { count: number, items: [{ negotiation_id, amount, deadline, days_remaining }] }` — depósitos pendentes; (4) `recent_notifications: [{ id, type, title, body, read, created_at }]` — últimas 5 notificações; (5) `financial_summary: { total_invested: number, operations_count: number, avg_commission: number }` — resumo financeiro; cache Redis `rs:dashboard:{cessionario_id}` TTL 2min (invalidar ao mudar proposal/negotiation status). (Doc 01.3 — RN-042; Doc 16)

- [x] **S3-BE02** · Implementar `GET /api/v1/notifications` em `apps/api/src/modules/notifications/`: guard `JwtAuthGuard`; paginação offset-based (`page`, `per_page` max 100 default 20) com objeto `meta: { total, page, per_page, total_pages }`; filtro opcional `?read=false` para não lidas; retornar array de notificações ordenadas por `created_at DESC`. (Doc 16 — GET /notifications)

- [x] **S3-BE03** · Implementar `PATCH /api/v1/notifications/:id/read` em `apps/api/src/modules/notifications/`: guard `JwtAuthGuard`; verificar que notificação pertence ao cessionário autenticado (isolamento); atualizar `read = true`, `read_at = NOW()`; retornar HTTP 200 `{ id, read: true, read_at }`; erro `NOT-FOUND-001` se notificação não existe; erro `PERM-001` se notificação pertence a outro cessionário. (Doc 16 — PATCH /notifications/:id/read)

- [x] **S3-BE04** · Implementar `NotificationService` em `apps/api/src/modules/notifications/notification.service.ts` (lógica central): função `notify(event: NotificationEvent)` — buscar preferências `cessionarios.notification_preferences`; determinar canais elegíveis (email sempre ativo — RN-069; push se `prefs.push = true`; in-app sempre); publicar em `notification.exchange` para cada canal via RabbitMQ; criar registro em `notifications` table (in-app always on); função `getEligibleChannels` — NOT-CES-05 e NOT-CES-06 são `critical` e não podem ser desabilitadas; para outros templates, respeitar preferências do usuário. (Doc 21 — seção 1.1)

- [x] **S3-BE05** · Implementar `NotificationWorkers` em `apps/api/src/modules/notifications/workers/`: `EmailWorker` consome `notification.email` — renderiza template React Email, envia via Resend SDK, retry 3x (30s→60s→120s), DLQ após 3 falhas; `PushWorker` consome `notification.push` — envia via Expo Push API, retry 2x (30s→60s), trata `DeviceNotRegistered` removendo token da tabela `notification_tokens`; `InAppWorker` consome `notification.inapp` — atualiza registro na `notifications` table, dispara Supabase Realtime para subscription do cessionário. (Doc 21 — seção 2)

- [x] **S3-BE06** · Implementar todos os 17 templates de notificação NOT-CES-01 a NOT-CES-17 em `apps/api/src/modules/notifications/templates/`: cada template com título, corpo e variáveis exatamente conforme Doc 01.4 (RN-056, tabela de 17 notificações) e Doc 21 (tabela de inventário NOT-CES-01 a NOT-CES-11 + restantes). Verificar: NOT-CES-05 e NOT-CES-06 são `critical` (não desabilitáveis, enviados via todos os canais independente de preferências). (Doc 01.4 — RN-056; Doc 21 — seção 3)

### 🎨 Frontend

- [x] **S3-FE01** · Implementar tela `T-DASH-01 — Dashboard` em `apps/web/src/features/dashboard/pages/DashboardPage.tsx` (rota `/dashboard`): buscar dados via `GET /api/v1/dashboard` com TanStack Query (staleTime 2min); exibir 5 widgets com `KpiCard` para cada: (1) Resumo de Propostas — contadores por status com badges coloridos; (2) Negociações Ativas — lista com status e prazo Escrow (countdown se `days_remaining ≤ 2`); (3) Depósito Escrow Pendente — alerta visual com valor e prazo; (4) Notificações Recentes — últimas 5 com badge não lida; (5) Resumo Financeiro — total investido e nº de operações; skeleton screens durante loading (não spinner genérico); estado vazio com `EmptyState` para cada widget conforme RN-043. (Doc 06 — T-DASH-01; Doc 01.3 — RN-042)

- [x] **S3-FE02** · Implementar widget de Notificações Recentes no Dashboard e sidebar: subscription Supabase Realtime na tabela `notifications` filtrada por `cessionario_id` (RLS ativo); ao receber evento INSERT, incrementar `notifications.store.unreadCount` e adicionar ao topo da lista; in-app banner para notificações recebidas em foreground (componente customizado 4s no topo — não via sistema OS); ao clicar: marcar como lida via `PATCH /api/v1/notifications/:id/read`, navegar para deep link se houver. (Doc 01.3 — RN-043; Doc 21 — seção 2.3)

- [x] **S3-FE03** · Implementar empty states obrigatórios para T-DASH-01 (RN-043): quando `proposals_summary` todo zero → ilustração + "Nenhuma proposta enviada ainda. Explore o marketplace!"; quando `active_negotiations` zero → "Você não tem negociações ativas."; quando `pending_escrow` zero → nenhum alerta exibido; quando `recent_notifications` vazio → "Nenhuma notificação."; todos com ação sugerida (botão ou link). (Doc 01.3 — RN-043)

- [x] **S3-FE04** · Implementar `SlaCountdown.tsx` em `apps/web/src/components/domain/`: recebe `deadline: Date`; exibe tempo restante em formato "X dias Y horas" ou "X horas Y minutos" se < 24h; cor verde se > 3 dias, âmbar se 1-3 dias, vermelho se < 1 dia; atualização a cada 60s; usado no widget de Escrow e negociações no Dashboard. (Doc 01.3 — RN-042)

---

## FEATURE 2 — Perfil (T-PRF-01, T-PRF-04, T-PRF-05)

### ⚙️ Backend

- [x] **S3-BE07** · Implementar `GET /api/v1/profile` em `apps/api/src/modules/cessionarios/`: guard `JwtAuthGuard`; retornar `{ user: { id, name, email, provider }, cessionario: { id, status, phone, phone_verified_at, bank_account: { masked }, bank_account_verified_at, notification_preferences, ai_consent, ai_consent_at, investment_preferences, created_at } }`; nunca retornar dados bancários completos — mascarar agência e conta (últimos 4 dígitos visíveis); cache Redis `rs:cessionario:{id}:perfil` TTL 15min. (Doc 16 — GET /profile)

- [x] **S3-BE08** · Implementar `PATCH /api/v1/profile` em `apps/api/src/modules/cessionarios/`: guard `JwtAuthGuard`; campos editáveis: `name`, `phone`, `investment_preferences`; NOT editável via este endpoint: `email` (exige fluxo separado de verificação), `bank_account` (exige endpoint específico), `status` (controlado pelo sistema); validações: `name` 2-100 chars, `phone` formato brasileiro; retornar perfil atualizado; invalidar cache `rs:cessionario:{id}:perfil`. (Doc 16 — PATCH /profile)

- [x] **S3-BE09** · Implementar `PATCH /api/v1/profile/notifications` em `apps/api/src/modules/cessionarios/`: guard `JwtAuthGuard`; body `{ push: boolean, sms: boolean }`; email NUNCA desabilitável (RN-069 — canal mínimo obrigatório); atualizar `cessionarios.notification_preferences`; sincronia com push token: se `push=false`, remover push token da tabela `notification_tokens`; retornar preferências atualizadas; invalidar cache do perfil. (Doc 16 — PATCH /profile/notifications; Doc 01.4 — RN-069 via Doc 01.5)

- [x] **S3-BE10** · Implementar `PATCH /api/v1/profile/ai-consent` em `apps/api/src/modules/cessionarios/`: guard `JwtAuthGuard`; body `{ ai_consent: boolean }`; se `false`, registrar `ai_consent_at = NOW()` com valor false (auditoria LGPD); se Analista de IA já está em execução para este cessionário, encerrar sessão ativa; retornar `{ ai_consent, ai_consent_at }`; log de auditoria em `audit_logs` com `action: "ai_consent_changed"`. (Doc 16 — PATCH /profile/ai-consent; Doc 01.1 — RN-015)

- [x] **S3-BE11** · Implementar `GET /api/v1/profile/data-export` (LGPD) em `apps/api/src/modules/cessionarios/`: guard `JwtAuthGuard`; retornar todos os dados do cessionário (proposta, negociações, transações financeiras, sessões IA) em formato JSON estruturado; operação assíncrona — retornar HTTP 202 com `{ job_id, estimated_time: "5 minutos" }`; enviar arquivo por e-mail quando pronto; registrar em `audit_logs` com `action: "data_export_requested"`. (Doc 01.1 — RN-016; Doc 16)

### 🎨 Frontend

- [x] **S3-FE05** · Implementar tela `T-PRF-01 — Perfil` em `apps/web/src/features/profile/pages/ProfilePage.tsx` (rota `/perfil`): buscar via `GET /api/v1/profile`; exibir: nome, e-mail (readonly), `CessionarioStatusBadge`, status KYC com botão "Completar KYC" se `status !== KYC_APROVADO`; dados bancários mascarados com botão "Atualizar dados bancários"; preferências de notificação (toggles); consentimento IA (toggle com texto informativo); botão "Editar perfil" → T-PRF-04; botão "Configurações" → T-PRF-05; botão "Exportar meus dados" (LGPD) → chama `GET /api/v1/profile/data-export` + toast "Você receberá os dados por e-mail em até 5 minutos". (Doc 06 — T-PRF-01)

- [x] **S3-FE06** · Implementar tela `T-PRF-04 — Editar Perfil` em `apps/web/src/features/profile/pages/EditProfilePage.tsx` (rota `/perfil/editar`): formulário com campos `nome` e `telefone`; validação inline; submit chama `PATCH /api/v1/profile`; sucesso → volta para T-PRF-01 com toast "Perfil atualizado"; erro de validação → mensagens por campo (`aria-describedby`); input de telefone com máscara BR `(XX) XXXXX-XXXX`. (Doc 06 — T-PRF-04)

- [x] **S3-FE07** · Implementar tela `T-PRF-05 — Configurações` em `apps/web/src/features/profile/pages/SettingsPage.tsx` (rota `/perfil/configuracoes`): seção "Notificações" com toggles: Push (desabilitável), E-mail (label "Sempre ativo" — toggle desabilitado e sempre true, tooltip "Canal mínimo garantido"), SMS (label "Em breve"); ao mudar push, chamar `PATCH /api/v1/profile/notifications`; seção "Privacidade": toggle "Consentimento para Analista de IA" com texto explicativo sobre coleta de dados, chamar `PATCH /api/v1/profile/ai-consent`; seção "Conta": botão "Sair" (logout), link "Excluir conta" (desabilitado no MVP com tooltip). (Doc 06 — T-PRF-05; Doc 01.1 — RN-015)

---

## FEATURE 3 — Notificações (Centro de Notificações)

### ⚙️ Backend

- [x] **S3-BE12** · Implementar job cron para alertas de prazo Escrow (NOT-CES-05 e NOT-CES-06) em `apps/api/src/modules/notifications/jobs/escrow-alert.job.ts`: cron `0 9 * * *` (9h diariamente); buscar negociações com `status = AGUARDANDO_DEPOSITO` e `escrow_deadline = HOJE + 2 dias úteis` → disparar NOT-CES-05; negociações com `escrow_deadline = AMANHÃ` → disparar NOT-CES-06; estas notificações são `critical` (não podem ser desabilitadas — RN-069). (Doc 01.4 — RN-055; Doc 21 — NOT-CES-05, NOT-CES-06)

### 🎨 Frontend

- [x] **S3-FE08** · Implementar centro de notificações (drawer/página) em `apps/web/src/features/notifications/`: listar notificações via `GET /api/v1/notifications?per_page=20`; infinite scroll com TanStack Query `useInfiniteQuery`; cada item: ícone por tipo, título, corpo, timestamp relativo (date-fns `formatDistanceToNow`), indicador visual de não lida (ponto azul); ao clicar: `PATCH /api/v1/notifications/:id/read` + navegar para deep link se houver; botão "Marcar todas como lidas" (chamada batch se disponível); badge no header sidebar mostra `unreadCount` do `notifications.store`.

### 🔌 Wiring

- [x] **S3-W01** · Configurar Supabase Realtime subscription para notificações no frontend: subscription em tabela `notifications` com filtro `cessionario_id=eq.{id}` (RLS garante isolamento); ao receber INSERT: adicionar ao store, incrementar badge, exibir in-app banner se foreground; ao receber UPDATE (read): atualizar store; degradação para polling TanStack Query a cada 30s se Realtime indisponível; banner "Atualizações em tempo real temporariamente indisponíveis" com fundo âmbar. (Doc 17 — Supabase Realtime; Doc 21 — seção 2.3)

### 🧪 Testes

- [x] **S3-T01** · Testes unitários `DashboardService`: aggregate retorna 5 widgets com dados corretos; cache Redis invalidado ao mudar status de proposta/negociação; cenário sem dados retorna zeros e arrays vazios (não null/undefined). Cobertura branches 100%.

- [x] **S3-T02** · Testes unitários `NotificationService`: `getEligibleChannels` — NOT-CES-05/06 sempre retorna todos os canais independente de preferências; templates com push=false não incluem canal push; email nunca removido. Mock RabbitMQ e Redis.

- [x] **S3-T03** · Testes unitários `NotificationWorkers`: `EmailWorker` — retry 3x em falha Resend, DLQ após 3 falhas; `PushWorker` — remove token se `DeviceNotRegistered`; `InAppWorker` — insere na tabela notifications e dispara Realtime.

- [x] **S3-T04** · Teste E2E Playwright — TC-CES-06 "Recebimento de notificação in-app": simular webhook de KYC aprovado → verificar que badge no header incrementa e notificação aparece no centro de notificações sem reload de página. (Doc 01.3 — TC-CES-06)

- [x] **S3-T05** · Teste E2E Playwright — Dashboard com dados reais: login com cessionário com propostas ativas, verificar que 5 widgets rendem corretamente; verificar skeleton durante loading; verificar empty state em widget sem dados; verificar countdown de Escrow.

---

## 🔀 Cross-Módulo

- [x] **S3-CM01** · **[← S4, S5, S6, S7, S8]** `NotificationService.notify()` será chamado por todos os módulos ao mudar estado: proposals (NOT-CES-03), negotiations (NOT-CES-04), escrow (NOT-CES-05, NOT-CES-06, NOT-CES-07), formalization (NOT-CES-08), financial (NOT-CES-09, NOT-CES-10). Garantir que `NotificationService` é exportado de `NotificationsModule` e importado pelos módulos de domínio.

- [x] **S3-CM02** · **[← S2]** Dashboard widget "Negociações Ativas" usa dados de `negotiations` table. Garantir que `NegotiationsRepository` é acessível ou que dashboard query agrega diretamente via Prisma com `include: { opportunity: { select: { code: true } } }` — sem N+1.

---

## 📊 COBERTURA DE REQs S3

| REQ ID | Descrição                              | Item(s)                   |
| ------ | -------------------------------------- | ------------------------- |
| S3-001 | Dashboard 5 widgets                    | S3-BE01, S3-FE01          |
| S3-002 | Widget propostas por status            | S3-BE01, S3-FE01          |
| S3-003 | Widget negociações ativas              | S3-BE01, S3-FE01          |
| S3-004 | Widget escrow pendente                 | S3-BE01, S3-FE01          |
| S3-005 | Widget notificações recentes           | S3-BE01, S3-FE01, S3-FE02 |
| S3-006 | Widget resumo financeiro               | S3-BE01, S3-FE01          |
| S3-007 | Empty states 8 telas                   | S3-FE03                   |
| S3-008 | SlaCountdown componente                | S3-FE04                   |
| S3-009 | GET /profile                           | S3-BE07                   |
| S3-010 | PATCH /profile                         | S3-BE08                   |
| S3-011 | PATCH /profile/notifications + RN-069  | S3-BE09                   |
| S3-012 | PATCH /profile/ai-consent LGPD         | S3-BE10                   |
| S3-013 | GET /profile/data-export LGPD          | S3-BE11                   |
| S3-014 | T-PRF-01, T-PRF-04, T-PRF-05           | S3-FE05, S3-FE06, S3-FE07 |
| S3-015 | 17 templates NOT-CES-01 a NOT-CES-17   | S3-BE06                   |
| S3-016 | Job cron alertas escrow NOT-CES-05/06  | S3-BE12                   |
| S3-017 | Centro notificações + Realtime         | S3-FE08, S3-W01           |
| S3-018 | Preferências notificação sincronizadas | S3-BE09, S3-FE07          |
