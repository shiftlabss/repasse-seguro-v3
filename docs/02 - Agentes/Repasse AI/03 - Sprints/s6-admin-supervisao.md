# S6 — Admin e Supervisão

<!-- Atualizado em 2026-03-24 pela A03 — 4 correções aplicadas (FINDING-003, FINDING-009, FINDING-014, FINDING-016) -->

| Campo              | Valor                                                                                                                                                                                                                  |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S6                                                                                                                                                                                                                     |
| **Nome**           | Admin e Supervisão                                                                                                                                                                                                     |
| **Template**       | B — Módulo Fullstack                                                                                                                                                                                                   |
| **Docs Fonte**     | 01.4-RN-Admin, 14-EspecTécnicas, 16-API                                                                                                                                                                                |
| **REQs cobertos**  | REQ-059, REQ-060, REQ-061, REQ-077, REQ-078, REQ-079, REQ-080, REQ-081, REQ-082, REQ-083, REQ-084, REQ-085, REQ-086, REQ-087, REQ-088, REQ-089, REQ-090, REQ-091, REQ-098, REQ-099, REQ-100, REQ-101, REQ-102, REQ-103 |
| **Total de itens** | 30                                                                                                                                                                                                                     |
| **Status**         | ✅ Concluido                                                                                                                                                                                                           |

---

## 🎯 Objetivo

Implementar o painel de supervisão Admin: takeover manual com mutex de concorrência, 6 alertas automáticos via RabbitMQ, dashboard de 5 KPIs, configuração de threshold (50%-95%, padrão 80%), gerenciamento de status do agente, e os endpoints de supervisão.

---

## 🏗️ FEATURE 1: Takeover — Intervenção Manual Admin

### 1. `SupervisaoService.initiateTakeover` — iniciar takeover

- [x] Criar `SupervisaoService` com método `initiateTakeover(admin: AuthenticatedUser, conversationId: string, reason: TakeoverReason): TakeoverResult`:
  - Verificar que não há takeover ativo para a conversa (SELECT em `ai_takeovers` WHERE `conversation_id = conversationId AND ended_at IS NULL`) — mutex de concorrência via lock Redis: `SET ai:takeover:{conversationId} {adminId} NX EX 3600`
  - Se lock já existe → `ConflictException("Takeover já ativo para esta conversa por outro Admin")`
  - INSERT em `ai_takeovers`: `{ conversation_id, admin_id, reason, confidence_at_takeover, started_at: now() }`
  - Publicar evento `ai.commands` exchange: `{ type: 'TAKEOVER_STARTED', conversationId, adminId }` para notificar o agente
  - Retornar `{ id, conversationId, adminId, reason, startedAt }`
- [x] Enum `TakeoverReason` valores: `low_confidence`, `admin_manual`, `user_request`, `error`
- [x] Teste unitário: takeover em conversa sem lock → sucesso; takeover em conversa com lock Redis → `ConflictException`; lock Redis falha → registrar erro e criar takeover sem lock (degraded mode — documentar ⚠️)

### 2. `SupervisaoService.endTakeover` — encerrar takeover

- [x] Implementar `endTakeover(admin: AuthenticatedUser, takeoverId: string): TakeoverResult`:
  - Verificar que o takeover pertence ao `adminId` (ou qualquer ADMIN pode encerrar)
  - UPDATE `ai_takeovers SET ended_at = now()` WHERE `id = takeoverId AND ended_at IS NULL`
  - Liberar lock Redis: `DEL ai:takeover:{conversationId}`
  - Publicar evento `ai.commands`: `{ type: 'TAKEOVER_ENDED', conversationId, adminId }`
  - Se `ended_at` já preenchido → `ConflictException("Takeover já encerrado")`
  - Retornar `{ id, conversationId, adminId, reason, startedAt, endedAt }`
- [x] Teste unitário: encerrar takeover ativo → `ended_at` preenchido + lock Redis liberado; encerrar takeover já encerrado → `ConflictException`

### 3. Endpoint `POST /supervision/takeover` — iniciar takeover

- [x] Criar `SupervisionController` com endpoint `POST /supervision/takeovers` (`@Roles('ADMIN')`) — [CORRIGIDO: FINDING-016 — plural conforme Registro Mestre]
- [x] DTO `InitiateTakeoverDto`: `{ conversationId: string (uuid), reason: TakeoverReason, notes?: string }`
- [x] Resposta HTTP: `201 Created` com `TakeoverResponseDto`: `{ id, conversationId, adminId, reason, startedAt }`
- [x] Teste de integração: ADMIN inicia takeover → `201`; CESSIONARIO tenta → `403`; conversa inexistente → `404`; lock já existe → `409`

### 4. Endpoint `DELETE /supervision/takeover/:id` — encerrar takeover

- [x] Criar endpoint `DELETE /supervision/takeovers/:id` (`@Roles('ADMIN')`) — [CORRIGIDO: FINDING-016]
- [x] `SupervisaoService.endTakeover` com `takeoverId` do param
- [x] Resposta HTTP: `200 OK` com `TakeoverResponseDto` atualizado (com `endedAt`)
- [x] Teste de integração: encerrar takeover ativo → `200`; takeover já encerrado → `409`; takeover inexistente → `404`

### 5. Endpoint `GET /supervision/takeovers` — listar takeovers

- [x] Criar endpoint `GET /supervision/takeovers` (`@Roles('ADMIN')`)
- [x] Query params: `conversationId?: string`, `active?: boolean`, `page (default 1)`, `per_page (default 20)`
- [x] Retornar `{ data: TakeoverResponseDto[], meta: { page, per_page, total, total_pages } }`
- [x] Teste de integração: GET sem filtros → todos os takeovers; `active=true` → apenas takeovers com `ended_at IS NULL`

---

## 🏗️ FEATURE 2: Alertas Automáticos (6 tipos)

### 6. `AlertService` — consumidor RabbitMQ + 6 tipos de alerta

- [x] Criar `AlertService` como consumer RabbitMQ na fila `ai.metrics.queue`:
  - **Alerta 1 — Baixa confiança:** `ai_interactions.confidence_score < agent_configurations.confidence_threshold` → publicar em `ai.notifications` com `type: 'LOW_CONFIDENCE'`
  - **Alerta 2 — Erro do agente:** `ai_interactions.error IS NOT NULL` → publicar com `type: 'AGENT_ERROR'`
  - **Alerta 3 — Latência alta:** `ai_interactions.latency_ms > 10000` → publicar com `type: 'HIGH_LATENCY'`
  - **Alerta 4 — Rate limit atingido:** consumir eventos de rate limit do `RateLimitGuard` → publicar com `type: 'RATE_LIMIT_HIT'`
  - **Alerta 5 — Takeover iniciado:** consumir eventos `TAKEOVER_STARTED` do `ai.commands` → publicar com `type: 'TAKEOVER_INITIATED'`
  - **Alerta 6 — Indisponibilidade:** `agent_status` != `online` → publicar com `type: 'AGENT_UNAVAILABLE'`
  - **Desligamento automático — [CORRIGIDO: FINDING-003]:** Contador de erros via Redis: `INCR ai:error:count:{window}` (janela = minute floor) + `EXPIRE 900s`. Se soma dos últimos 15 contadores > 30% do total de requests no mesmo período → INSERT em `agent_status_logs` com `status: 'offline'`, `reason: 'auto_shutdown_error_rate_exceeded'` + publicar `ai.events` com `type: 'AGENT_AUTO_SHUTDOWN'`. Reativação exclusivamente manual via `POST /supervision/agent/status` pelo Admin.
- [x] Cada alerta: INSERT em `notification_events` com `type`, `title`, `body`, `metadata: { triggeredBy, value }`
- [x] Alertas são fire-and-forget — falha no alert NÃO bloqueia fluxo principal
- [x] Teste unitário: mock de `ai_interactions` com `confidence_score=0.5` (abaixo de threshold padrão 0.80) → publicar `LOW_CONFIDENCE`; `latency_ms=15000` → publicar `HIGH_LATENCY`; taxa erro > 30% em 15min → INSERT status 'offline' + evento AGENT_AUTO_SHUTDOWN; consumer crash → log de erro, alerta perdido (documentar ⚠️)

### 7. Endpoint `GET /supervision/alerts` — listar alertas

- [x] Criar endpoint `GET /supervision/alerts` (`@Roles('ADMIN')`)
- [x] Query params: `type?: NotificationType`, `page (default 1)`, `per_page (default 20)`, `since?: ISO date`
- [x] SELECT em `notification_events` WHERE `type IN (alertTypes)` ORDER BY `created_at DESC`
- [x] Retornar `{ data: AlertDto[], meta: { page, per_page, total, total_pages } }`
- [x] `AlertDto`: `{ id, type, title, body, metadata, sentAt, createdAt }`
- [x] Teste de integração: 3 alertas de `LOW_CONFIDENCE` + 1 `AGENT_ERROR` → GET sem filtro → 4; GET `type=LOW_CONFIDENCE` → 3

---

## 🏗️ FEATURE 3: Dashboard de KPIs Admin

### 8. `MetricsService.getDashboardMetrics` — 5 KPIs

- [x] Criar `MetricsService` com método `getDashboardMetrics(period: '24h' | '7d' | '30d'): DashboardMetrics`:
  - **KPI 1 — Total de conversas:** COUNT `chat_conversations` WHERE `created_at >= NOW() - interval`
  - **KPI 2 — Mensagens por conversa (média):** AVG(COUNT `chat_messages` GROUP BY `conversation_id`) no período
  - **KPI 3 — Taxa de takeover:** COUNT `ai_takeovers` / COUNT `ai_interactions` \* 100 no período
  - **KPI 4 — Latência média (ms):** AVG(`ai_interactions.latency_ms`) no período
  - **KPI 5 — Taxa de cache hit:** COUNT(cache_hit=true) / COUNT(_) _ 100 FROM `ai_interactions` no período
  - Retornar `{ period, totalConversations, avgMessagesPerConversation, takeoverRate, avgLatencyMs, cacheHitRate, generatedAt }`
- [x] Cachear resultado no Redis com TTL 5 minutos: chave `ai:metrics:dashboard:{period}`
- [x] Teste unitário: mock `ai_interactions` com 10 registros (3 cache hit) → `cacheHitRate = 30%`; período vazio → retornar zeros (não `null`); cache Redis hit → não reconsultar DB

### 9. Endpoint `GET /supervision/metrics` — dashboard KPIs

- [x] Criar endpoint `GET /supervision/metrics` (`@Roles('ADMIN')`)
- [x] Query param: `period: '24h' | '7d' | '30d'` (default `'24h'`)
- [x] Retornar `DashboardMetricsResponseDto` conforme item 8
- [x] Resposta HTTP: `200 OK`
- [x] Teste de integração: GET `?period=7d` → `200` com 5 KPIs; GET `?period=invalid` → `400`

---

## 🏗️ FEATURE 4: Configuração de Threshold e Status do Agente

### 10. `AgentConfigService.updateThreshold` — configurar threshold de confiança

- [x] Criar `AgentConfigService` com método `updateThreshold(admin: AuthenticatedUser, threshold: number): AgentConfigResult`:
  - Validar `threshold` entre 0.50 e 0.95 (50%-95%) — `BadRequestException` se fora do range
  - UPDATE `agent_configurations SET value = threshold WHERE key = 'confidence_threshold'` com `updated_by = admin.userId`
  - Invalidar cache Redis `ai:metrics:dashboard:*` (pattern delete)
  - Retornar `{ key: 'confidence_threshold', value: threshold, updatedBy, updatedAt }`
- [x] Threshold padrão: 0.80 (seed de S1 — verificar consistência)
- [x] Teste unitário: `threshold=0.75` → UPDATE correto; `threshold=0.49` → `BadRequestException`; `threshold=0.96` → `BadRequestException`; `threshold=0.50` → válido; `threshold=0.95` → válido (limites incluídos)

### 11. Endpoint `PUT /supervision/config/threshold` — atualizar threshold

- [x] Criar endpoint `PUT /supervision/config/threshold` (`@Roles('ADMIN')`)
- [x] DTO `UpdateThresholdDto`: `{ threshold: number (min 0.50, max 0.95) }`
- [x] Retornar `ConfigResponseDto`: `{ key, value, updatedBy, updatedAt }`
- [x] Resposta HTTP: `200 OK`
- [x] Teste de integração: PUT `{ threshold: 0.75 }` → `200`; PUT `{ threshold: 0.49 }` → `400`; CESSIONARIO tenta → `403`

### 12. `AgentConfigService.updateAgentStatus` — atualizar status do agente

- [x] Implementar `updateAgentStatus(admin: AuthenticatedUser, status: AgentStatus, reason?: string)`:
  - Validar `status` pertence ao enum `AgentStatus`: `online`, `offline`, `maintenance`, `degraded`
  - INSERT em `agent_status_logs` (append-only): `{ status, reason, triggered_by: admin.userId }`
  - Publicar evento `ai.events` exchange: `{ type: 'AGENT_STATUS_CHANGED', status, reason, adminId }`
  - Retornar `{ id, status, reason, triggeredBy, createdAt }`
- [x] Máquina de estado `AgentStatus` — todas as transições válidas: qualquer status → qualquer status (sem restrição — Admin tem controle total)
- [x] Teste unitário: `online → maintenance` → INSERT + publicar evento; INSERT falha → propagar erro (status é crítico)

### 13. Endpoint `POST /supervision/agent/status` — atualizar status agente

- [x] Criar endpoint `POST /supervision/agent/status` (`@Roles('ADMIN')`)
- [x] DTO `UpdateAgentStatusDto`: `{ status: AgentStatus, reason?: string }`
- [x] Retornar `AgentStatusResponseDto`: `{ id, status, reason, triggeredBy, createdAt }`
- [x] Resposta HTTP: `201 Created`
- [x] Teste de integração: POST `{ status: 'maintenance' }` → `201`; `status: 'invalid'` → `400`; CESSIONARIO tenta → `403`

### 14. Endpoints `GET /ai/interactions` e `GET /ai/interactions/{id}` — listagem Admin [CORRIGIDO: FINDING-014]

- [x] Criar endpoints em `AiController` (ou `SupervisionController`): `GET /ai/interactions` e `GET /ai/interactions/:id` (`@Roles('ADMIN')`)
- [x] `GET /ai/interactions`: paginação + filtros: `cessionarioId?`, `dateRangeStart?`, `dateRangeEnd?`, `cacheHit?`, `intent?` — SELECT `ai_interactions` com JOIN `chat_messages` e `chat_conversations`, filtrar por `organization_id = admin.organizationId`
- [x] `GET /ai/interactions/:id`: detalhe completo com `toolCalls` e métricas
- [x] `InteractionDetailDto`: `{ id, conversationId, model, promptTokens, completionTokens, totalTokens, latencyMs, confidenceScore, toolCalls, cacheHit, cacheType, createdAt }` — campo `cessionario` anonimizado na listagem (mostrar apenas `cessionarioId`, não nome/email)
- [x] Teste: Admin lista interações → apenas da própria organização; CESSIONARIO → `403`; interação inexistente → `404`

### 15. Endpoint `GET /supervision/conversations` — listar conversas para Admin

- [x] Criar endpoint `GET /supervision/conversations` (`@Roles('ADMIN')`)
- [x] Query params: `cessionarioId?: string`, `status?: ConversationStatus`, `hasTakeover?: boolean`, `page`, `per_page`
- [x] Retornar lista de conversas com `ai_interactions` count e último `ai_takeover` (se existir)
- [x] **Sem filtro de escopo por `cessionario_id`** — Admin vê todas as conversas
- [x] Teste de integração: `hasTakeover=true` → apenas conversas com takeover ativo; `cessionarioId=X` → apenas conversas de X; sem filtros → todas

---

## ✅ TESTES — Admin e Supervisão

### 15. Suite de testes

- [x] Criar `test/unit/supervision/supervisao.service.spec.ts` — cobrir takeover mutex, lock Redis, encerramento
- [x] Criar `test/unit/supervision/alert.service.spec.ts` — cobrir 6 tipos de alerta + fire-and-forget
- [x] Criar `test/unit/supervision/metrics.service.spec.ts` — cobrir 5 KPIs + cache Redis 5min
- [x] Criar `test/unit/supervision/agent-config.service.spec.ts` — cobrir threshold 0.50-0.95 + status machine
- [x] Criar `test/integration/supervision/supervision.controller.spec.ts` — cobrir todos os 7 endpoints
- [x] Cobertura mínima módulo `supervision/`: 75% (conforme Doc 27)

---

## 🔀 Cross-Módulo

- [x] **← S3b (Agente Core):** `ai_interactions` persistidas em S3b são fonte de dados dos KPIs e alertas; `ai.commands` events publicados aqui são consumidos pelo agente para pausar fluxo durante takeover
- [x] **← S1 (Fundação):** `agent_configurations` seed com `confidence_threshold=0.80` é utilizado pelo `AlertService` para checar limiar
- [x] **→ S7 (WhatsApp/Notificações):** `notification_events` INSERTados pelos alertas (item 6) são consumidos pelo módulo de notificações para entrega

---

## ✔️ Auto-Verificação S6 (12 Checks)

| #   | Check                                                                                                                                                                                                                                          | Status |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                                                                                   | ✅     |
| 2   | Nomes exatos: `SupervisaoService`, `AlertService`, `MetricsService`, `AgentConfigService`, `ai_takeovers`, `notification_events`, `agent_status_logs`, `agent_configurations`, `ai:takeover:{conversationId}`, `ai:metrics:dashboard:{period}` | ✅     |
| 3   | Valores numéricos: threshold padrão 0.80, range 0.50-0.95, 6 alertas, 5 KPIs, cache metrics TTL 5min, lock TTL 3600s                                                                                                                           | ✅     |
| 4   | Glossário consultado — Takeover, Score de Risco, RBAC                                                                                                                                                                                          | ✅     |
| 5   | Anti-Scaffold R10: mutex Redis documentado; 6 alertas com tipos reais; 5 KPIs com fórmulas SQL; threshold com validação de range                                                                                                               | ✅     |
| 6   | Máquina de estado `AgentStatus`: `online↔offline↔maintenance↔degraded` (transições livres para Admin)                                                                                                                                          | ✅     |
| 7   | Takeover mutex: `SET NX EX 3600` no Redis; liberação em `endTakeover`                                                                                                                                                                          | ✅     |
| 8   | Sem conflitos não marcados                                                                                                                                                                                                                     | ✅     |
| 9   | Alertas fire-and-forget — falha NÃO bloqueia fluxo principal                                                                                                                                                                                   | ✅     |
| 10  | Template B: organizado por FEATURE (Takeover, Alertas, Dashboard, Configuração)                                                                                                                                                                | ✅     |
| 11  | Apenas `@Roles('ADMIN')` em todos os endpoints desta sprint                                                                                                                                                                                    | ✅     |
| 12  | REQs cobertos: REQ-059 a REQ-061, REQ-077 a REQ-091, REQ-098 a REQ-103 — todos com ≥1 item                                                                                                                                                     | ✅     |
