# S8 — SLA, Dashboard e Notificações

## CRM Repasse Seguro

| **Campo**         | **Valor**                                                                                                                                                                                                                                                            |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**        | S8                                                                                                                                                                                                                                                                   |
| **Nome**          | SLA, Dashboard e Notificações                                                                                                                                                                                                                                        |
| **Template**      | B — Módulo Fullstack                                                                                                                                                                                                                                                 |
| **Docs fonte**    | 01.3, 01.4, 05.1, 06, 14, 16, 27                                                                                                                                                                                                                                     |
| **REQs cobertos** | REQ-077 a REQ-082, REQ-088 a REQ-090, REQ-109, REQ-145, REQ-165, REQ-166, REQ-223, REQ-224, REQ-243, REQ-244, REQ-255, REQ-274, REQ-292, REQ-306 a REQ-308                                                                                                           |
| **Objetivo**      | Implementar monitoramento proativo de SLA com 4 níveis de alerta, Dashboard Executivo com 7 métricas (defasagem ≤5 min), relatório semanal automatizado aos domingos 20h, notificações SSE em tempo real via Supabase Realtime, e aba Financeiro exclusiva Admin RS. |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — `sla_alerts`, `SlaCheckerWorker`, `WeeklyReportWorker`, 4 níveis de alerta exatos (`AMARELO`/`LARANJA`/`VERMELHO`/`CRITICO`), 7 métricas exatas do Dashboard Executivo
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: Valores exatos — cron 08h00 America/Fortaleza, relatório domingo 20h00, defasagem ≤5 min, 60 dias ciclo total, Realtime subscription por canal, SSE heartbeat
- [x] C4: Glossário — SLA, alerta, Dashboard Executivo, Realtime cobertos
- [x] C5: Estados de SLA Alert: AMARELO / LARANJA / VERMELHO / CRITICO — sem transições reversas (append-only)
- [x] C6: Workers: SlaCheckerWorker (cron 08h00), WeeklyReportWorker (domingo 20h00), ambos com retry e DLQ
- [x] C7: RBAC — Dashboard Executivo: Coord RS + Admin RS; aba Financeiro: apenas Admin RS; alertas visíveis por Analista RS (apenas seus Casos)
- [x] C8: T-CRM-080 Dashboard Executivo com estados loading/empty por métrica; T-CRM-081 Central de Notificações
- [x] C9: Integração com SlaCheckerWorker de S3 — worker de S3 é EXPANDIDO aqui, não duplicado
- [x] C10: Glossário — SSE vs Supabase Realtime distinção documentada
- [x] C11: Anti-scaffold — lógica de cálculo das 7 métricas com fórmulas reais, lógica de alerta por prazo de estado, lógica de funil de conversão
- [x] C12: REQ-077 a REQ-082, REQ-088 a REQ-090, REQ-109, REQ-145, REQ-165, REQ-166, REQ-223, REQ-224, REQ-243, REQ-244, REQ-255, REQ-274, REQ-292, REQ-306 a REQ-308 cobertos

---

## FEATURE 1 — SLA Monitoring (Expansão do SlaCheckerWorker)

### Banco / Migrations

- [x] **REQ-077** Verificar tabela `sla_alerts` com campos: `id UUID PK`, `case_id UUID FK cases`, `alert_level VARCHAR CHECK IN ('AMARELO','LARANJA','VERMELHO','CRITICO')`, `state VARCHAR` (estado do Caso que gerou o alerta), `days_overdue INT`, `triggered_at TIMESTAMPTZ DEFAULT NOW()`, `resolved_at TIMESTAMPTZ NULL`, `notified_users UUID[]`. Adicionar index em `sla_alerts.case_id`, `sla_alerts.alert_level`, `sla_alerts.triggered_at`
  - Validar: enums de `alert_level` completos; campos nullable corretos; indexes criados

- [x] **REQ-306** Verificar `case_status_history` com campo `sla_deadline TIMESTAMPTZ` populado ao criar entrada. Garantir que a migration seta `sla_deadline = entered_at + [prazo do estado]` via trigger ou via NestJS ao criar registro de histórico
  - Validar: entry para estado NEGOCIACAO tem `sla_deadline = entered_at + 7 days`; VERIFICACAO = 5 days; ANUENCIA = 10 days

### Backend

- [x] **REQ-078 / REQ-079** Expandir `SlaCheckerWorker` (cron `0 8 * * *` America/Fortaleza, iniciado em S3): adicionar lógica de 4 níveis de alerta por estado:
  - Para cada Caso ativo: calcular `days_in_state = NOW() - entered_at` do estado atual
  - Comparar com prazo do estado (Cadastro 1d, Simulação 2d, Verificação 5d, Publicação 1d, Match 3d, Negociação 7d, Anuência 10d, Formalização 5d)
  - Gerar alerta `AMARELO` se 50% do prazo consumido; `LARANJA` se 75%; `VERMELHO` se 100% (vencido); `CRITICO` se ciclo total ≥ 60 dias
  - Inserir em `sla_alerts` APENAS se não existir alerta do mesmo `case_id` + `alert_level` + estado não resolvido (evitar duplicatas)
  - Para alerta `CRITICO`: notificar Coordenador RS via `notification_logs` + push notification
  - Validar: Caso em NEGOCIACAO por 5 dias → gera `LARANJA`; por 8 dias → gera `VERMELHO`; ciclo 60 dias → gera `CRITICO`; alerta duplicado não reinserido

- [x] **REQ-080** Implementar `GET /sla/alerts` — Roles: `COORDENADOR_RS`, `ADMIN_RS`. Query params: `case_id`, `alert_level`, `state`, `resolved`, `from_date`, `to_date`, `page`, `per_page`. Retornar alertas com dados do Caso joinado (número RS, estado atual, responsável). Analista RS → 403
  - Validar: filtro `alert_level=CRITICO` retorna apenas alertas críticos; Analista RS → 403; paginação funciona

- [x] **REQ-081** Implementar `GET /sla/alerts/:caseId` — Roles: todos internos com escopo. Analista RS vê apenas alertas de Casos atribuídos a ele. Retornar array de `sla_alerts` do Caso ordenado por `triggered_at DESC`
  - Validar: Analista RS com Caso atribuído → 200; Analista RS com Caso de outro → 403; array ordenado corretamente

- [x] **REQ-082** Implementar `PATCH /sla/alerts/:alertId/resolve` — Roles: `COORDENADOR_RS`, `ADMIN_RS`. Setar `resolved_at = NOW()`. Criar `audit_log`. Retornar alerta atualizado
  - Validar: alerta já resolvido → 422 `CRM-038`; resolve sem `resolved_at` → 422; audit_log criado

- [x] **REQ-307** Implementar endpoint `GET /sla/summary` — Roles: `COORDENADOR_RS`, `ADMIN_RS`. Retornar contagem de Casos por nível de alerta ativo (não resolvido): `{ AMARELO: N, LARANJA: N, VERMELHO: N, CRITICO: N, sem_alerta: N }`. Cache Redis com TTL 5 minutos (chave `sla:summary`)
  - Validar: cache hit após primeira chamada; TTL 5 min correto; valores refletem `sla_alerts` não resolvidos

---

## FEATURE 2 — Notificações em Tempo Real (SSE + Supabase Realtime)

### Backend

- [x] **REQ-109** Implementar Server-Sent Events em `GET /notifications/stream` — Roles: todos internos autenticados. Headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`. Enviar heartbeat `event: ping` a cada 30 segundos para manter conexão. Fechar stream ao desconectar cliente. Eventos emitidos:
  - `sla_alert`: quando novo `sla_alert` criado para Caso do usuário
  - `case_state_change`: quando Caso atribuído ao usuário muda de estado
  - `followup_reminder`: 30 min antes de Follow-up agendado do usuário
  - `commission_update`: quando comissão do usuário muda de estado
  - Validar: heartbeat a cada 30s; cliente recebe evento `sla_alert` após inserção em `sla_alerts`; conexão fecha corretamente ao timeout

- [x] **REQ-308** Implementar Supabase Realtime subscription server-side para tabelas `sla_alerts`, `cases`, `activities` (para follow-up reminders), `commissions`. Ao receber evento Realtime, publicar no SSE stream dos usuários afetados. Canal por usuário: `notifications:user:{userId}`. Garantir que apenas eventos do próprio usuário (por `assigned_to` ou `created_by`) são enviados ao stream
  - Validar: inserção em `sla_alerts` → SSE chega ao Analista RS do Caso em <2s; mudança de estado do Caso → evento `case_state_change` no SSE; evento não vaza para usuário não relacionado

- [x] **REQ-223** Implementar `GET /notifications` — Roles: todos internos. Query params: `read`, `type`, `from_date`, `page`, `per_page`. Retornar entradas de `notification_logs` do usuário autenticado, ordenadas por `created_at DESC`
  - Validar: usuário vê apenas suas notificações; filtro `read=false` retorna apenas não lidas; paginação funciona

- [x] **REQ-224** Implementar `PATCH /notifications/:id/read` e `PATCH /notifications/read-all` — Roles: todos internos. Setar `read_at = NOW()` nas entradas de `notification_logs`. `read-all` → setar todas não lidas do usuário autenticado
  - Validar: notificação já lida → idempotente (não retorna erro); `read-all` seta todas do usuário; não afeta notificações de outros usuários

---

## FEATURE 3 — Dashboard Executivo

### Backend

- [x] **REQ-088** Implementar `GET /dashboard/metrics` — Roles: `COORDENADOR_RS`, `ADMIN_RS`. Retornar as 7 métricas calculadas em tempo real (com cache Redis TTL 5 min, chave `dashboard:metrics:{userId}`):
  1. **Total de Casos ativos** — COUNT de `cases` com state NOT IN ('CONCLUIDO','CANCELADO','ARQUIVADO')
  2. **Casos por estado** — COUNT agrupado por `state`
  3. **Taxa de conversão** — `cases CONCLUIDO` / `cases totais criados no período` × 100
  4. **SLA vencido** — COUNT de Casos com alerta `VERMELHO` ou `CRITICO` ativos
  5. **Comissão prevista** — SUM de `commissions.calculated_amount` WHERE `status = PENDENTE`
  6. **Comissão realizada** — SUM de `commissions.calculated_amount` WHERE `status = PAGO`
  7. **Tempo médio de ciclo** — AVG de `(concluded_at - created_at)` em dias para Casos `CONCLUIDO` no período
  - Query params: `from_date`, `to_date` (padrão: últimos 30 dias)
  - Validar: cache hit após primeira chamada; cada métrica tem valor numérico correto; Analista RS → 403

- [x] **REQ-089** Implementar `GET /dashboard/funnel` — Roles: `COORDENADOR_RS`, `ADMIN_RS`. Retornar funil de conversão: para cada estado, quantidade de Casos que entraram no estado no período + quantidade que avançaram + taxa de conversão entre estados. Cache Redis TTL 5 min (chave `dashboard:funnel:{period}`)
  - Validar: estados em ordem do pipeline; percentual de conversão correto entre estados adjacentes; cache funciona

- [x] **REQ-090** Implementar `GET /dashboard/financial` — Roles: **apenas `ADMIN_RS`**. Retornar dados financeiros detalhados: comissões por período (mês a mês), comissões por Analista RS, distribuição Cedente/Cessionário, ticket médio. Cache Redis TTL 5 min (chave `dashboard:financial:{period}`)
  - Validar: `COORDENADOR_RS` → 403; Admin RS → 200 com dados completos; cache correto

- [x] **REQ-145** Implementar `WeeklyReportWorker` — cron `0 20 * * 0` (domingo 20h00 America/Fortaleza). Gerar relatório semanal com:
  - Novos Casos criados na semana
  - Casos avançados por estado
  - Follow-ups vencidos e concluídos
  - Alertas SLA disparados
  - Comissões geradas
  - Enviar por e-mail para todos `COORDENADOR_RS` e `ADMIN_RS` ativos via `notification_logs` (type: `RELATORIO_SEMANAL`) + e-mail
  - Validar: cron executa domingo 20h00; e-mail enviado para todos Coord RS + Admin RS; falha de envio → DLQ; relatório com dados corretos da semana

### Frontend

- [x] **REQ-165** Implementar `app/(dashboard)/dashboard/page.tsx` — T-CRM-080 Dashboard Executivo:
  - **Layout**: grid de 4 KPI cards (linha superior) + 2 gráficos grandes (linha inferior) + 1 tabela de Casos críticos
  - **KPI cards**: Total Casos Ativos, SLA Vencido (com badge vermelho), Comissão Prevista (R$), Tempo Médio Ciclo (dias)
  - **Gráfico 1**: Funil de conversão (barras horizontais por estado, percentual de conversão entre estados)
  - **Gráfico 2**: Comissão mês a mês (área/linha, últimos 6 meses)
  - **Tabela**: Casos com alerta CRITICO ou VERMELHO — colunas: número RS, estado, dias em atraso, responsável, botão "Ver Caso"
  - **Aba Financeiro** (apenas `ADMIN_RS`): ocultar tab completamente para outros roles; exibir detalhamento financeiro com gráficos de comissão por Analista RS
  - Estado loading: skeleton em cada card/gráfico independente (não spinner global)
  - Estado empty: "Nenhum Caso ativo no período." por componente
  - **Defasagem ≤5 min**: polling via TanStack Query `refetchInterval: 300_000` (5 min); badge "Atualizado há X min" no topo
  - Validar: Analista RS → redirecionar para `/cases`; Coord RS → sem aba Financeiro; Admin RS → com aba Financeiro; loading por card independente

- [x] **REQ-166** Implementar `app/(dashboard)/notifications/page.tsx` — T-CRM-081 Central de Notificações:
  - Lista de notificações com filtros: Tipo (SLA / Follow-up / Comissão / Sistema) + Status (Lida / Não lida)
  - Badge de contagem de não lidas no ícone do sino (Topbar, componente `NotificationBell`)
  - Botão "Marcar todas como lidas"
  - Cada notificação: ícone por tipo, timestamp relativo ("há 5 minutos"), texto, link para entidade relacionada
  - SSE: conectar ao `GET /notifications/stream` via `EventSource` no `NotificationProvider` (Context React). Ao receber evento → invalidar query TanStack `['notifications']` + atualizar badge count
  - Empty state: "Nenhuma notificação. Você está em dia!"
  - Validar: badge atualiza em tempo real via SSE; click na notificação navega para entidade; "Marcar todas como lidas" zera badge

- [x] **REQ-243** Implementar `NotificationProvider` em `app/layout.tsx` — instanciar `EventSource('/api/notifications/stream')` com reconexão automática (exponential backoff: 1s → 2s → 4s → 8s → 16s → máx 30s). Expor `useNotifications()` hook. Ao receber evento `sla_alert` → exibir toast com badge vermelho + link para Caso
  - Validar: SSE reconecta após queda; toast aparece ao criar alerta CRITICO; hook `useNotifications()` retorna contagem correta

- [x] **REQ-244** Implementar componente `SlaAlertBadge` — exibir badge colorido por nível: AMARELO (#F59E0B), LARANJA (#F97316), VERMELHO (#EF4444), CRITICO (pulsante, #DC2626 com `animate-ping`). Integrar em `CaseCard` (kanban S3) e `CaseRow` (lista S3) mostrando badge do alerta mais grave do Caso
  - Validar: CRITICO exibe badge pulsante; cor correta por nível; Caso sem alerta → sem badge

---

## 🧪 Testes

### Backend — Unitário (Jest)

- [x] **REQ-255** `SlaMonitorService` — cobertura 70%:
  - `should generate AMARELO alert when 50% of state deadline consumed`
  - `should generate LARANJA alert when 75% of state deadline consumed`
  - `should generate VERMELHO alert when 100% of state deadline consumed`
  - `should generate CRITICO alert when total cycle >= 60 days`
  - `should not duplicate alert for same case_id + alert_level + state`
  - `should resolve alert correctly setting resolved_at`
  - `should calculate dashboard metrics correctly for period`
  - `should restrict financial dashboard to ADMIN_RS`

### Backend — Integração

- [x] **REQ-274** Testes de integração SLA + Dashboard:
  - `GET /dashboard/metrics` com período de 30 dias → retornar 7 métricas com valores numéricos
  - `GET /dashboard/financial` por `COORDENADOR_RS` → 403
  - `PATCH /sla/alerts/:alertId/resolve` → `resolved_at` preenchido + audit_log criado
  - `GET /notifications/stream` → receber evento SSE após inserção em `sla_alerts`
  - WeeklyReportWorker: executar manualmente → `notification_logs` criados para todos Coord RS + Admin RS

### Frontend — E2E (Playwright)

- [x] **REQ-292** E2E: Dashboard Executivo:
  - Login como `ADMIN_RS` → `/dashboard` → verificar 4 KPI cards renderizados
  - Verificar aba Financeiro visível para Admin RS
  - Login como `COORDENADOR_RS` → `/dashboard` → verificar aba Financeiro ausente
  - Login como `ANALISTA_RS` → `/dashboard` → redirecionar para `/cases`
  - Aguardar 5s → badge "Atualizado há X min" presente

---

## 🔀 Cross-Módulo

- O `SlaCheckerWorker` iniciado em S3 é expandido aqui (não duplicado). A lógica de verificação de Follow-ups vencidos adicionada em S6 e a lógica de alertas de SLA adicionada aqui coexistem no mesmo worker.
- Os alertas SLA inseridos em `sla_alerts` aqui aparecem na linha do tempo do Caso (S6, `getCaseTimeline`) como entradas `EVENTO_SISTEMA`.
- O `NotificationProvider` (SSE) aqui é o mecanismo de entrega dos avisos de follow-up 30 min antes criados em S6 (job agendado em RabbitMQ → ao disparar → insere em `notification_logs` → SSE entrega ao cliente).
- O `WeeklyReportWorker` (domingo 20h00) inclui dados de atividades e follow-ups vencidos de S6 no relatório.
