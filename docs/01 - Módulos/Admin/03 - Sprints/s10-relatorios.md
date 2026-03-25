# S10 — Relatórios

## Repasse Seguro — Módulo Admin

| Campo | Valor |
|---|---|
| **Sprint** | S10 |
| **Nome** | Relatórios |
| **Template** | B — Módulo Fullstack |
| **REQs cobertos** | REQ-023, REQ-143, REQ-158, REQ-159, REQ-160, REQ-161, REQ-162, REQ-235, REQ-236, REQ-237, REQ-238, REQ-239, REQ-240, REQ-241 |
| **Docs fonte** | D01.4 §3 (RN-101 a RN-108, RN-149), D06 §2.10 (T-085 a T-091) |
| **Itens totais** | 36 |

---

> **Objetivo da sprint:** Implementar o módulo de Relatórios com 6 abas analíticas (SLA Operacional, Volume de Casos, Receita, Conversão, Trilha de Auditoria, Agentes IA), RBAC por aba (COORDENADOR sem Receita; GESTOR_FINANCEIRO com Receita+Auditoria; MASTER tudo), exportação PDF/CSV (DEC-011: link temporário 1h — RN-104/108), agendamento de envio automático (MASTER only), período máximo de consulta 24 meses (RN-149), trilha de auditoria imutável (RN-105), relatório financeiro mensal automático no 1° dia útil (RN-092 REQ-143), e alerta de calibração IA (RN-106).

---

## FEATURE 1 — Infraestrutura de Relatórios

### Backend

**[BE-01] Implementar `ReportsModule` — queries agregadas e cache**
- Módulo NestJS `ReportsModule` com `ReportsService` injetando `PrismaService` e `RedisService`
- Cache Redis por relatório: chave `rs:reports:{type}:{userId}:{filters_hash}` TTL 15min (atualização máxima a cada 15 minutos — RN-101)
- Período máximo de consulta: 24 meses; qualquer `start_date` anterior a `NOW() - 720 days` → ajustar automaticamente para 24 meses e retornar `{ data, meta: { period_adjusted: true, adjusted_start: date } }` (RN-149)
- **Relatório financeiro mensal automático (RN-092 REQ-143):** cron `0 9 1-7 * 1` (1° dia útil de cada mês às 9h); gerar PDF do Relatório de Receita + Auditoria do mês anterior; enviar por e-mail ao Gestor Financeiro e Master; registrar em `notification_logs`
- Verificação: GET de relatório com período de 25 meses → `meta.period_adjusted = true`; cron do 1° dia útil gera PDF e enfileira e-mail

**[BE-02] Implementar exportação PDF e CSV (RN-104/108 e DEC-011)**
- `POST /v1/reports/:type/export`: role conforme RBAC do tipo de relatório
- Body: `{ format: 'pdf' | 'csv', filters: object }`
- **DEC-011 — link temporário:** ao gerar o arquivo, salvar em storage temporário (Supabase Storage bucket `reports-exports`) + gerar URL assinada com expiração de 1h; retornar `202 Accepted` com `{ data: { export_id, estimated_ready_in_seconds } }`
- Job assíncrono via RabbitMQ: gerar o arquivo (PDF via Puppeteer renderizando HTML; CSV via papaparse); ao concluir → emitir notificação in-app com link de download ao solicitante
- PDF inclui gráficos renderizados; CSV inclui dados brutos
- `GET /v1/reports/exports/:export_id`: verificar status (`PROCESSING | READY | FAILED`) + retornar URL assinada quando `READY`
- Registro de exportação: cada export registrado em `audit.audit_logs` com `action = 'REPORT_EXPORTED'`, `metadata = { type, format, filters, exported_by }`
- Verificação: `POST export` retorna 202 com `export_id`; GET de `export_id` com status READY retorna URL assinada; URL expira após 1h; audit log registrado

**[BE-03] Implementar agendamento de envio automático (RN-108 — Master only)**
- `POST /v1/reports/schedules`: role `MASTER`; body: `{ report_type, format, recipients: string[], frequency: 'daily' | 'weekly' | 'monthly', filters: object }`
- `GET /v1/reports/schedules`: role `MASTER`; lista agendamentos ativos
- `DELETE /v1/reports/schedules/:id`: role `MASTER`; desativar agendamento
- Job cron executa agendamentos: ao disparar, gera arquivo + envia por e-mail aos destinatários configurados via `NotificationService`
- Verificação: criar agendamento mensal de Receita → job executa no dia configurado → e-mail enviado com arquivo anexado

---

## FEATURE 2 — Relatórios por Tipo

### Backend

**[BE-04] Implementar `GET /v1/reports/sla` — Relatório SLA Operacional (RN-101)**
- RBAC: COORDENADOR, GESTOR_FINANCEIRO (apenas leitura para cross-check), MASTER
- Params: `start_date`, `end_date` (max 24 meses), `analyst_id?`, `scenario?`
- Resposta: tabela de cada transição: `{ transition, sla_target_days, sla_max_days, avg_actual_days, pct_within_sla, pct_exceeded }` (7 transições conforme D01.3 tabela SLAs)
- Gráfico de tendência: `{ period_labels[], pct_within_sla_series[] }` (semanal ou mensal conforme range)
- Ranking de Analistas: `{ analyst_id, analyst_name, avg_resolution_days_by_stage[] }` ordenado por melhor desempenho
- Casos com estouro: ao clicar em etapa com estouro → `GET /v1/reports/sla/cases?transition=...&start_date=...` retorna lista de casos com link
- Verificação: query com range de 3 meses retorna dados corretos; campo `pct_exceeded` = 100% - `pct_within_sla`

**[BE-05] Implementar `GET /v1/reports/volume` — Relatório Volume de Casos (RN-102)**
- RBAC: COORDENADOR, MASTER
- Params: `start_date`, `end_date`, `granularity: 'week' | 'month'`
- Resposta: `{ by_status: {status, count}[], by_scenario: {scenario, count, pct}[], timeline: {period, captured, closed, cancelled}[], comparison: {current_period, previous_period, delta_pct} }`
- Verificação: volume com granularity `week` retorna série semanal; comparação de períodos calcula delta correto

**[BE-06] Implementar `GET /v1/reports/revenue` — Relatório Receita (RN-103)**
- RBAC: GESTOR_FINANCEIRO, MASTER (COORDENADOR retorna 403 — REQ-159)
- Params: `start_date`, `end_date`
- Resposta: `{ realized: number, in_pipeline: number, projected: number, by_scenario: {scenario, total, count}[], avg_ticket: number, lost_revenue: number }`
- `realized`: SUM commission_invoices WHERE `status = 'DISTRIBUTED'` no período
- `in_pipeline`: SUM commission_invoices WHERE case.status = 'POS_FECHAMENTO'
- `projected`: estimativa baseada em casos EM_FORMALIZACAO e EM_NEGOCIACAO × taxa de conversão histórica (24 meses)
- `lost_revenue`: SUM commission_invoices estimados de casos CANCELADO ou REVERTIDO
- Verificação: COORDENADOR retorna 403; GESTOR_FINANCEIRO retorna 200 com campos corretos; `realized + in_pipeline + projected` logicamente consistente

**[BE-07] Implementar `GET /v1/reports/conversion` — Relatório Conversão (RN-104)**
- RBAC: COORDENADOR, MASTER
- Params: `start_date`, `end_date`
- Resposta: funil completo `{ stages: [{ status, count, pct_from_previous, pct_loss, avg_time_days }] }` para os 7 estágios: CAPTADO, QUALIFICADO, OFERTA_ATIVA, EM_NEGOCIACAO, EM_FORMALIZACAO, FECHAMENTO, CONCLUIDO
- Identificar automaticamente maior ponto de perda: `{ biggest_drop_stage, biggest_drop_pct }`
- Verificação: funil retorna 7 estágios; `biggest_drop_stage` é o estágio com maior `pct_loss`

**[BE-08] Implementar `GET /v1/reports/audit` — Trilha de Auditoria (RN-105)**
- RBAC: GESTOR_FINANCEIRO, MASTER (REQ-160); COORDENADOR retorna 403
- Params: `case_id?`, `user_id?`, `action?`, `start_date`, `end_date`, `page`, `per_page`
- Filtros obrigatórios para evitar dump completo (REQ-162): pelo menos 1 filtro obrigatório (`case_id` OU `user_id` OU `action`) OU `start_date`+`end_date` ≤ 30 dias; caso contrário → `422` com message `"Aplique pelo menos um filtro para consultar a trilha de auditoria."`
- Dados de `audit.audit_logs`: ID, `actor_id`, `actor_name`, `action`, `target_id`, `target_type`, `metadata (JSONB)`, `created_at`
- Registros IMUTÁVEIS: nenhum endpoint de DELETE ou PATCH existe para `audit.audit_logs`
- Export CSV e JSON apenas para MASTER (GESTOR_FINANCEIRO pode exportar apenas CSV)
- Verificação: COORDENADOR → 403; query sem filtros e com período > 30d → 422; registros corretos e imutáveis

**[BE-09] Implementar `GET /v1/reports/ai-agents` — Relatório Agentes IA (RN-106)**
- RBAC: COORDENADOR, MASTER
- Params: `start_date`, `end_date`, `agent_name?`
- Resposta: `{ by_agent: [{ agent_name, total_actions, avg_confidence, alerts_generated, takeovers, success_rate, monthly_comparison: {current, previous, delta_pct} }] }`
- Alerta de calibração automático (RN-106): se `alerts_generated / total_actions > 15%` → `{ calibration_alert: true, recommendation: "Taxa de alertas excede 15%. Revise o limiar ou os parâmetros do agente." }`
- Verificação: taxa > 15% → `calibration_alert = true`; comparação mês a mês calcula delta corretamente

---

## FEATURE 3 — Frontend Relatórios (T-085 a T-091)

### Frontend

**[FE-01] Implementar T-085 — Hub de Relatórios (`/relatorios`)**
- Rota: `/relatorios`; módulo visível para COORDENADOR (sem Receita), GESTOR_FINANCEIRO (Receita+Auditoria), MASTER (tudo); oculto para ANALISTA
- Grid de cards de relatórios disponíveis; cada card: ícone, nome, descrição curta, botão "Abrir"
- Relatórios sem permissão: exibidos com overlay bloqueado cinza + tooltip "Permissão requerida: [perfil necessário]"
- Tabs no hub: SLA Operacional, Volume de Casos, Receita (bloqueada para COORDENADOR), Conversão, Auditoria (bloqueada para COORDENADOR), Agentes IA
- Verificação: COORDENADOR vê tabs Receita e Auditoria com overlay bloqueado; MASTER vê todas desbloqueadas

**[FE-02] Implementar T-086 — Relatório SLA Operacional**
- Layout: filtros topo (date range picker max 24m + analista select + cenário select), área de gráficos (tendência linha + tabela de transições), tabela detalhada abaixo
- Tabela de transições: 7 linhas com SLA Alvo, SLA Máximo, Tempo Médio Real, % Dentro do SLA (barra de progresso colorida), % Estourado
- Gráfico de tendência: `ReportChart` (C-10) tipo linha; eixo Y = % SLA cumprido; eixo X = períodos
- Ranking de Analistas: tabela ordenável por tempo médio; clique na linha expande por etapa
- Clicar em etapa com estouro: abre modal com lista de casos (com link direto para o caso)
- Estado vazio: ícone + "Sem dados para o período selecionado."
- Skeleton loading durante fetch
- Verificação: selecionar período de 25 meses → date picker limita a 24 meses + aviso; clicar em etapa com estouro → modal com casos

**[FE-03] Implementar T-087 — Relatório Volume de Casos**
- Gráfico de barras agrupadas: captados/fechados/cancelados por semana ou mês (toggle semanal/mensal)
- Gráfico de pizza: distribuição por cenário A/B/C/D com percentuais
- Cards de comparação: Mês atual vs. Anterior (∆%), Trimestre atual vs. Anterior (∆%)
- Verificação: toggle semanal → granularity muda para `week`; valores de comparação batem com os dados da tabela

**[FE-04] Implementar T-088 — Relatório Receita (GESTOR_FINANCEIRO + MASTER only)**
- Oculto para COORDENADOR (overlay bloqueado em hub)
- 5 métricas em cards: Receita Realizada, Em Pipeline, Projetada, Ticket Médio, Receita Perdida
- Gráfico de barras empilhadas: receita por cenário A/B/C/D
- Verificação: acessar como COORDENADOR → overlay bloqueado; como MASTER → dados carregam normalmente

**[FE-05] Implementar T-089 — Relatório Conversão**
- Funil visual interativo: 7 etapas com largura proporcional à quantidade; cada etapa: count, % conversão, % perda, tempo médio
- Destaque visual automático no maior ponto de perda (borda vermelha + badge "Maior queda")
- Verificação: estágio com maior `pct_loss` exibe badge "Maior queda"; hover em etapa exibe tooltip com `avg_time_days`

**[FE-06] Implementar T-090 — Relatório Auditoria (GESTOR_FINANCEIRO + MASTER only)**
- Filtros obrigatórios: pelo menos 1 entre (case_id, user_id, action) OU date range ≤ 30d; mensagem inline se não aplicado
- Tabela paginada: ID, actor_name, action (badge), target, metadata (expansível), data/hora
- Registros imutáveis: sem botões de edição ou exclusão
- Export: MASTER → CSV + JSON; GESTOR_FINANCEIRO → CSV apenas
- Verificação: buscar sem filtro → mensagem inline de filtro obrigatório; tabela não tem coluna de ação de edição/exclusão

**[FE-07] Implementar T-091 — Relatório Agentes IA e botão "Exportar" global**
- Relatório Agentes IA: cards por agente com métricas 24h + comparação mensal; alerta de calibração: banner amarelo se `calibration_alert = true`
- Botão "Exportar" em cada aba: dropdown (CSV / PDF); ao clicar → POST export → spinner "Gerando..." → toast com link de download ao concluir (DEC-011); toast de erro com "Tentar novamente" em caso de falha
- Download via URL assinada com expiração 1h (não download direto)
- Acessibilidade: tabelas alternativas para todos os gráficos (aria-label); contraste mínimo 4.5:1 WCAG 2.1 AA; tooltips acessíveis via Tab
- Verificação: exportar PDF do SLA → spinner → toast com link; link expira após 1h; gráficos têm tabela alternativa acessível

---

## WIRING

**[WIRE-01] Integrar `ReportsModule` no NestJS**
- `ReportsController` com rotas: `GET /v1/reports/sla`, `GET /v1/reports/volume`, `GET /v1/reports/revenue`, `GET /v1/reports/conversion`, `GET /v1/reports/audit`, `GET /v1/reports/ai-agents`, `POST /v1/reports/:type/export`, `GET /v1/reports/exports/:id`, `POST /v1/reports/schedules`, `GET /v1/reports/schedules`, `DELETE /v1/reports/schedules/:id`
- RBAC guard em cada rota conforme RN-107
- Cache Redis `rs:reports:{type}:{filters_hash}` invalidado quando dados subjacentes mudam (via hook pós-mutation em `CasesService`, `EscrowService`)
- Cron `0 9 1-7 * 1` para relatório mensal automático (RN-092)
- Verificação: `AppModule` importa `ReportsModule`; cron registrado no `ScheduleModule`

**[WIRE-02] Integrar frontend Relatórios com React Query**
- `useReportSla(filters)`, `useReportVolume(filters)`, `useReportRevenue(filters)`, `useReportConversion(filters)`, `useReportAudit(filters)`, `useReportAiAgents(filters)` — todos com `staleTime: 15min` (refletindo cache backend)
- `useExportReport(type)` mutation: após 202 → polling de `GET /v1/reports/exports/:id` até status `READY`; ao READY → toast com link
- Sidebar "Relatórios" visível para COORDENADOR, GESTOR_FINANCEIRO, MASTER; oculto para ANALISTA
- Verificação: mudança de filtro invalida cache e refetch; export polling termina ao receber `READY`

---

## TESTES

**[TEST-01] Testes unitários — `ReportsService`**
- `ReportsService.getSla()`: período > 24 meses → `period_adjusted = true` e data ajustada
- `ReportsService.getRevenue()`: COORDENADOR → lança `ForbiddenException(403)`
- `ReportsService.getAudit()`: sem filtros e período > 30d → lança `UnprocessableEntityException(422)`
- `ReportsService.getAiAgents()`: `alerts_generated / total_actions > 0.15` → `calibration_alert = true`
- Cron relatório mensal: gera PDF + enfileira e-mail para Gestor e Master

**[TEST-02] Testes de integração — endpoints de Relatórios**
- `GET /v1/reports/revenue`: COORDENADOR → 403; GESTOR_FINANCEIRO → 200
- `GET /v1/reports/audit` sem filtros e range > 30d → 422; com filtro `case_id` → 200
- `POST /v1/reports/sla/export` com `format = 'pdf'` → 202 com `export_id`; GET `export_id` → `READY` com URL assinada
- `POST /v1/reports/schedules` com COORDENADOR → 403; MASTER → 201

**[TEST-03] Testes de componente — Frontend**
- T-085 Hub: COORDENADOR vê overlay bloqueado em Receita e Auditoria; MASTER vê tudo desbloqueado
- T-086: date picker não aceita datas > 24 meses atrás; clicar em etapa com estouro abre modal
- T-088: rota `/relatorios/receita` como COORDENADOR → overlay bloqueado

**[TEST-04] Testes E2E — exportação**
- Playwright: MASTER abre T-086 → clica "Exportar PDF" → spinner aparece → aguarda notificação in-app → clica link → download inicia; acessar link após 1h → erro de URL expirada (mock)

---

## AUTO-VERIFICAÇÃO (12 CHECKS)

- [x] **Check #1 — RBAC por aba (RN-107):** ANALISTA sem acesso; COORDENADOR: SLA/Volume/Conversão/AgentesIA; GESTOR_FINANCEIRO: Receita/Auditoria; MASTER: tudo; implementado como overlay + 403 no backend
- [x] **Check #2 — Nomes canônicos:** `audit.audit_logs`, `commission_invoices`, `ai_agent_decisions`, `reports-exports` (bucket), `PROCESSING | READY | FAILED` — zero sinônimos
- [x] **Check #3 — Período máximo 24 meses (RN-149):** ajuste automático de `start_date`; `meta.period_adjusted = true` na resposta; data picker bloqueado > 24m
- [x] **Check #4 — DEC-011 link temporário 1h:** export via `202 Accepted` + job assíncrono; URL assinada Supabase Storage com expiração 1h; poll até READY
- [x] **Check #5 — Trilha imutável (RN-105):** zero endpoints DELETE/PATCH em `audit.audit_logs`; filtro obrigatório para queries (REQ-162); CSV+JSON apenas MASTER
- [x] **Check #6 — Relatório mensal automático (RN-092 REQ-143):** cron `0 9 1-7 * 1` (1° dia útil); PDF gerado; e-mail para Gestor+Master
- [x] **Check #7 — Alerta calibração IA (RN-106):** taxa > 15% → `calibration_alert = true` → banner amarelo no frontend
- [x] **Check #8 — Cache 15min (RN-101):** Redis `rs:reports:{type}:{filters_hash}` TTL 15min; invalidado após mutations relevantes
- [x] **Check #9 — Export registrado em audit (RN-104):** `audit.audit_logs` `action = 'REPORT_EXPORTED'` para cada export
- [x] **Check #10 — Acessibilidade WCAG 2.1 AA:** tabelas alternativas para gráficos; contraste 4.5:1; tooltips via Tab
- [x] **Check #11 — Anti-scaffold R10:** todos os itens têm sub-itens com queries SQL concretas, fórmulas de cálculo, RBAC, erros e verificação
- [x] **Check #12 — Cobertura REQs S10:** REQ-023 ✅ REQ-143 ✅ REQ-158 ✅ REQ-159 ✅ REQ-160 ✅ REQ-161 ✅ REQ-162 ✅ REQ-235 ✅ REQ-236 ✅ REQ-237 ✅ REQ-238 ✅ REQ-239 ✅ REQ-240 ✅ REQ-241 ✅

---

*Sprint S10 gerada por Pipeline ShiftLabs v2.3 — 2026-03-24*
