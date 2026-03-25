# S6 — Dashboard de Métricas

## Metadados

| Campo         | Valor                                                                           |
| ------------- | ------------------------------------------------------------------------------- |
| Sprint        | S6                                                                              |
| Nome          | Dashboard de Métricas                                                           |
| Template      | B (Módulo Fullstack por Feature)                                                |
| Módulo        | AI-Dani-Admin                                                                   |
| Docs Fonte    | D05 (RF-017 a RF-022, RF-026), D06, D07, D08, D09, D10, D12, D13, D16, D20, D25 |
| REQs Cobertos | REQs de S6 no registro-mestre.md                                                |
| Data          | 2026-03-24                                                                      |

## Objetivo

Implementar o módulo de Dashboard de Métricas completo (fullstack): endpoints de métricas agregadas (RF-017 a RF-019), métricas por agente (RF-020), cache Redis com TTL=60s (RF-021), tela T-004 (Dashboard Principal), tela T-006 (Log de Auditoria), log de auditoria de acesso, e todos os testes associados.

---

## FEATURE 1 — Métricas do Dashboard (RF-017 a RF-019, RF-021)

### BANCO (Feature 1)

- [x] Confirmar que índice `(agent_id, created_at DESC)` existe em `interactions` para queries de métricas eficientes
- [x] Confirmar que índice `(status, created_at DESC)` existe em `interactions`
- [x] Confirmar que `admin_access_logs` tem índice `(admin_id, created_at DESC)` para queries de log

### BACKEND (Feature 1)

- [x] Criar `apps/api/src/modules/metrics/metrics.module.ts`
- [x] Criar `apps/api/src/modules/metrics/metrics.service.ts`
- [x] Criar `apps/api/src/modules/metrics/metrics.controller.ts`
- [x] Criar `apps/api/src/modules/metrics/dto/dashboard-metrics.dto.ts`:
  - `period: '1h' | '24h' | '7d' | '30d'` — obrigatório
  - `agentId?: string` — opcional, filtra por agente específico
- [x] Implementar `MetricsService.getDashboardMetrics(dto: DashboardMetricsDto)`:
  - Chave de cache: `dani-admin:metrics:{agentId|all}:{period}` — TTL=60s (DASHBOARD_METRICS_CACHE_TTL_SECONDS=60)
  - Tenta ler do cache Redis primeiro
  - Se cache miss: calcula métricas via query PostgreSQL:
    - `totalInteractions`: COUNT de `interactions` no período onde `deleted_at IS NULL`
    - `interactionsByStatus`: COUNT GROUP BY `status`
    - `avgConfidenceScore`: AVG de `confidence_score`
    - `avgLatencyMs`: AVG de `latency_ms`
    - `csatAverage`: AVG de CSAT (⚠️ AMBÍGUO: campo CSAT não está nas migrations. Verificar se está em `interactions` ou em tabela separada — [PENDENTE — REVISÃO MANUAL])
    - `refusalRate`: COUNT de interações onde agente recusou / COUNT total × 100
    - `errorRate`: COUNT de interações com `status != 'RESPONDIDA_PELA_IA'` / COUNT total × 100
    - `takeoverCount`: COUNT de `takeovers` ativos no período
    - `autoDisabledCount`: COUNT de eventos `DESLIGADO_AUTOMATICO` em `admin_access_logs` no período
    - `topQuestions`: top 10 `user_message` mais frequentes no período (GROUP BY normalizado) — REQ-040 [CORRIGIDO: FINDING-013]
  - Armazena resultado no cache Redis com TTL=60s
  - Retorna objeto `DashboardMetrics`
- [x] Implementar `GET /api/v1/admin/metrics/dashboard` no `MetricsController`:
  - Guards: `@Roles('ADMIN')`
  - Query params: `period`, `agentId`
  - HTTP 200

### FRONTEND (Feature 1)

- [x] Criar `apps/web/src/app/admin/dashboard/page.tsx` — tela T-004 (Dashboard Principal) conforme D06/D07:
  - 4 cards de KPIs no topo (conforme wireframe D07):
    - Card "Total de Interações": exibe `totalInteractions`
    - Card "Confiança Média": exibe `avgConfidenceScore` + "%" ou "—" se null (D10)
    - Card "Latência Média": exibe `avgLatencyMs / 1000` + "s" (D10: `latency_ms → latency_seconds`)
    - Card "CSAT Médio": exibe `csatAverage` ou "—" (alerta visual se < 3.5)
  - Gráfico de interações por status (donut chart — shadcn/ui `recharts`)
  - Card "Top 10 Perguntas": lista das 10 mensagens de usuário mais frequentes no período — REQ-040 [CORRIGIDO: FINDING-013]
  - Gráfico de taxa de erro ao longo do tempo (linha)
  - Seletor de período: "1h", "24h", "7 dias", "30 dias"
  - Atualização automática: a cada 60s (polling, sem Realtime para métricas — D02)
  - Loading state: skeleton cards + skeleton charts (D08)
  - Empty state quando não há dados: "Nenhuma métrica disponível para o período selecionado."
- [x] Criar `apps/web/src/app/admin/dashboard/_components/kpi-card.tsx`:
  - Props: `title: string`, `value: string | number`, `unit?: string`, `alertThreshold?: number`, `alertBelow?: boolean`
  - Se `value < alertThreshold` e `alertBelow=true`: borda/texto em vermelho (ex: CSAT < 3.5)
- [x] Criar `apps/web/src/app/admin/dashboard/_components/interactions-by-status-chart.tsx`:
  - Donut chart com 4 segmentos (um por `InteractionStatus`)
  - Legenda: usa textos da interface (D10): "Aguardando revisão", "Em atendimento humano", "Respondida pela IA", "Encerrada"
- [x] Criar `apps/web/src/app/admin/dashboard/_components/error-rate-chart.tsx`:
  - Gráfico de linha com `errorRate` ao longo do tempo
  - Linha de threshold em 30% (AUTO_DISABLE_ERROR_RATE_THRESHOLD) — linha pontilhada vermelha

### WIRING (Feature 1)

- [x] Verificar que `GET /api/v1/admin/metrics/dashboard?period=24h` retorna métricas calculadas
- [x] Verificar que segunda chamada em < 60s retorna do cache Redis (verificar via log)
- [x] Verificar que dashboard T-004 exibe os 4 KPI cards com valores reais

### TESTES (Feature 1)

- [x] Criar `metrics.service.spec.ts` com Vitest:
  - Test: `getDashboardMetrics({ period: '24h' })` retorna objeto com todas as métricas esperadas
  - Test: segunda chamada em < 60s lê do cache Redis (mock `redisService.get` para verificar chamada)
  - Test: após 60s (mock TTL), lê do banco novamente
  - Test: `period = '7d'` usa janela de 7 dias na query SQL
  - Test: `agentId` especificado → query filtra por `agent_id`

---

## FEATURE 2 — Métricas por Agente (RF-020)

### BACKEND (Feature 2)

- [x] Criar `apps/api/src/modules/metrics/dto/agent-metrics.dto.ts`:
  - `period: '1h' | '24h' | '7d' | '30d'`
- [x] Implementar `MetricsService.getAgentMetrics(period: string)`:
  - Retorna array com métricas separadas para `DANI_CESSIONARIO` e `DANI_CEDENTE`
  - Para cada agente: `totalInteractions`, `avgConfidenceScore`, `avgLatencyMs`, `errorRate`, `takeoverRate`, `status` (lido de `agent_configurations`)
  - Usa cache Redis: `dani-admin:metrics:agent:{agentType}:{period}` com TTL=60s
- [x] Implementar `GET /api/v1/admin/metrics/agents` no `MetricsController`:
  - Guards: `@Roles('ADMIN')`
  - Query params: `period`
  - HTTP 200

### FRONTEND (Feature 2)

- [x] Criar `apps/web/src/app/admin/dashboard/_components/agent-metrics-table.tsx`:
  - Tabela com 2 linhas (uma por agente)
  - Colunas: "Agente", "Status", "Interações", "Confiança Média", "Latência Média", "Taxa de Erro"
  - Status usa badges (D10): "Ativo", "Desligado automaticamente", "Modo degradado"
  - Incluído na página T-004 abaixo dos charts

### TESTES (Feature 2)

- [x] Continuar em `metrics.service.spec.ts`:
  - Test: `getAgentMetrics('24h')` retorna array com 2 agentes (DANI_CESSIONARIO e DANI_CEDENTE)
  - Test: métricas de cada agente são calculadas independentemente

---

## FEATURE 3 — Log de Auditoria (RF-022)

### BANCO (Feature 3)

- [x] Confirmar que `admin_access_logs` é append-only (sem UPDATE/DELETE)
- [x] Confirmar que retenção é 365 dias (AUDIT_LOG_RETENTION_DAYS=365) — sem cleanup automático

### BACKEND (Feature 3)

- [x] Criar `apps/api/src/modules/metrics/dto/list-audit-logs.dto.ts`:
  - `cursor?: string`
  - `limit?: number` — default 20, max 100
  - `adminId?: string` — filtro por admin
  - `action?: string` — filtro por ação
  - `startDate?: string` — ISO8601
  - `endDate?: string` — ISO8601
- [x] Implementar `MetricsService.listAuditLogs(dto: ListAuditLogsDto)`:
  - Query com cursor-based pagination em `admin_access_logs`
  - Filtra por `adminId`, `action`, `startDate`, `endDate`
  - Retorna `{ data: AdminAccessLog[], nextCursor: string | null, total: number }`
  - NÃO filtra por `deleted_at` — logs são imutáveis
- [x] Implementar `GET /api/v1/admin/audit-logs` no `MetricsController`:
  - Guards: `@Roles('ADMIN')`
  - HTTP 200

### FRONTEND (Feature 3)

- [x] Criar `apps/web/src/app/admin/audit-logs/page.tsx` — tela T-006 (Log de Auditoria) conforme D06:
  - Título: "Log de auditoria" (D10 — `admin_access_logs` → "Log de auditoria")
  - Tabela com colunas: "Admin", "Ação", "Alvo", "Data"
  - Filtros: por admin (input), por ação (select), por intervalo de data
  - Paginação cursor-based: botão "Carregar mais"
  - Loading state: skeleton
  - Empty state: "Nenhum registro de auditoria encontrado."
  - Ações mapeadas para texto legível (ex: `TAKEOVER_STARTED` → "Takeover iniciado", `CONFIG_UPDATED` → "Configuração atualizada")

### WIRING (Feature 3)

- [x] Verificar que `GET /api/v1/admin/audit-logs` retorna registros criados nas sprints S3, S4, S5
- [x] Verificar que filtro `action=TAKEOVER_STARTED` filtra corretamente

### TESTES (Feature 3)

- [x] Continuar em `metrics.service.spec.ts`:
  - Test: `listAuditLogs({})` retorna primeiros 20 logs ordenados por `created_at DESC`
  - Test: `listAuditLogs({ action: 'TAKEOVER_STARTED' })` filtra por ação
  - Test: `listAuditLogs({ adminId: '<uuid>' })` filtra por admin

---

## FEATURE 4 — Supabase Realtime para Dashboard (RF-021)

### BACKEND (Feature 4)

- [x] Confirmar que Supabase Realtime está habilitado para `agent_configurations` (status change)

### FRONTEND (Feature 4)

- [x] Criar `apps/web/src/hooks/use-agent-status-realtime.ts`:
  - Subscreve ao canal `postgres_changes` na tabela `agent_configurations`
  - Ao receber UPDATE com `status = 'DESLIGADO_AUTOMATICO'`: exibe banner de alerta "Agente desligado automaticamente" na T-004
  - Ao receber UPDATE com `status = 'FALLBACK_ATIVO'`: exibe FAB com badge "Modo degradado" (D10 — FAB global)
  - Cleanup: cancela subscription no `useEffect` cleanup
- [x] Criar `apps/web/src/components/agent-status-banner.tsx`:
  - Banner no topo da T-004 quando algum agente está `DESLIGADO_AUTOMATICO`
  - Texto: "Agente [nome] foi desligado automaticamente devido à alta taxa de erros." — D08

### WIRING (Feature 4)

- [x] Verificar que ao mudar `agent_configurations.status` para `DESLIGADO_AUTOMATICO`, T-004 exibe banner

---

## 🔀 Cross-Módulo

- [x] `MetricsService` importa `AgentConfigService` (S5) para enriquecer `getAgentMetrics` com status atual do agente
- [x] Cache de métricas `dani-admin:metrics:*` usa `RedisService` (S1) — verificar que chave não conflita com `dani-admin:agent-config:*`

---

## AUTO-VERIFICAÇÃO S6

| Check                 | Critério                                                                                                                                                    | Status |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura       | `admin_access_logs` → "Log de auditoria", `agent_configurations` → "Configurações de Supervisão", `dani-admin:metrics:{agentId}:{period}` usados exatamente | ✅     |
| #2 Verificabilidade   | Cada item binariamente verificável                                                                                                                          | ✅     |
| #3 Valores numéricos  | DASHBOARD_METRICS_CACHE_TTL_SECONDS=60, AUDIT_LOG_RETENTION_DAYS=365, AUTO_DISABLE_ERROR_RATE_THRESHOLD=30% replicados                                      | ✅     |
| #4 N itens completos  | 4 features: dashboard, métricas por agente, log de auditoria, Realtime                                                                                      | ✅     |
| #5 Máquinas de estado | `FALLBACK_ATIVO` trigger (API indisponível) documentado via Realtime                                                                                        | ✅     |
| #6 Schedules/TTLs     | Cache TTL=60s, sem cleanup de `admin_access_logs`                                                                                                           | ✅     |
| #7 Conflitos          | Nenhum conflito novo                                                                                                                                        | ✅     |
| #8 Ambiguidades       | ⚠️ AMBÍGUO: campo CSAT não localizado nas migrations — marcado como [PENDENTE — REVISÃO MANUAL]                                                             | ✅     |
| #9 Anti-scaffold      | `MetricsService.getDashboardMetrics` tem lógica real: 9 métricas calculadas, cache Redis, múltiplos períodos                                                | ✅     |
| #10 Cross-módulo      | `MetricsService` importa `AgentConfigService` para status atual                                                                                             | ✅     |
| #11 IDs de referência | RF-017 a RF-022, D06, D07, D08, D09, D10, D25 citados                                                                                                       | ✅     |
| #12 Cobertura REQs S6 | Todos os REQs de métricas no registro-mestre.md têm ≥ 1 item                                                                                                | ✅     |

---

## ⚠️ Flags Pendentes

- **CSAT [PENDENTE — REVISÃO MANUAL]**: O campo `csatAverage` das métricas pressupõe que CSAT está armazenado. As migrations de S1 não incluem campo `csat_score` em `interactions`. Se CSAT é coletado, precisa de migration adicional. Verificar D12 e adicionar campo se necessário antes de implementar.
