# S5 — Alertas e Configuração de Agentes

## Metadados

| Campo | Valor |
|---|---|
| Sprint | S5 |
| Nome | Alertas e Configuração de Agentes |
| Template | B (Módulo Fullstack por Feature) |
| Módulo | AI-Dani-Admin |
| Docs Fonte | D05 (RF-011 a RF-016, RF-023, RF-026), D06, D07, D08, D09, D10, D12, D13, D16, D17, D20, D21, D25 |
| REQs Cobertos | REQs de S5 no registro-mestre.md |
| Data | 2026-03-24 |

## Objetivo

Implementar o módulo de Alertas (RF-011 a RF-013) e o módulo de Configuração de Agentes (RF-014 a RF-016, RF-023): criação e disparo de alertas via RabbitMQ para Slack/e-mail/push/in-app, idempotência por Redis, configuração de `confidence_threshold` e `webchat_rate_limit` pelo Admin, tela T-005 (Configurações de Supervisão), e tela T-003 (alertas in-app). Inclui também a resolução do TODO cross-módulo de S4 (publicação de evento de takeover em fila).

---

## FEATURE 1 — Configuração de Agentes (RF-014, RF-015, RF-016)

### BANCO (Feature 1)

- [x] Confirmar que `agent_configurations` tem campo `version INTEGER NOT NULL DEFAULT 1` — para lock otimista de configuração
- [x] Confirmar que constraint `CHECK (confidence_threshold >= 50 AND confidence_threshold <= 95)` existe
- [x] Confirmar que `agent_configurations` foi populado pelo seed com 2 registros (DANI_CESSIONARIO, DANI_CEDENTE) com `confidence_threshold=80`, `webchat_rate_limit=30`

### BACKEND (Feature 1)

- [x] Criar `apps/api/src/modules/agent-config/agent-config.module.ts`
- [x] Criar `apps/api/src/modules/agent-config/agent-config.service.ts`
- [x] Criar `apps/api/src/modules/agent-config/agent-config.controller.ts`
- [x] Criar `apps/api/src/modules/agent-config/dto/update-agent-config.dto.ts`:
  - `confidenceThreshold?: number` — opcional, 50–95 (MIN_CONFIDENCE_THRESHOLD=50, MAX_CONFIDENCE_THRESHOLD=95)
  - `webchatRateLimit?: number` — opcional, min 1
  - `systemPrompt?: string` — opcional
  - `isActive?: boolean` — opcional
  - `version: number` — obrigatório (lock otimista)
- [x] Implementar `AgentConfigService.listConfigurations()`:
  - Retorna todos os registros de `agent_configurations` (2 registros: DANI_CESSIONARIO e DANI_CEDENTE)
  - Inclui todos os campos: `id`, `agentType`, `status`, `confidenceThreshold`, `webchatRateLimit`, `systemPrompt`, `isActive`, `version`
- [x] Implementar `AgentConfigService.updateConfiguration(id: string, dto: UpdateAgentConfigDto, adminId: string)`:
  - Verifica que `id` existe — senão 404 `DA-CFG-001`
  - Lock otimista: `UPDATE agent_configurations SET ... WHERE id = :id AND version = :dto.version` — se nenhuma row atualizada → 409 `DA-CFG-002` (`'Configuração foi alterada por outro administrador. Recarregue e tente novamente.'`)
  - Se atualizado: incrementa `version += 1`
  - Invalida cache Redis `dani-admin:agent-config:{id}` — TTL = `AGENT_CONFIG_CACHE_TTL_SECONDS = 300`
  - Registra em `admin_access_logs`: `{ action: 'CONFIG_UPDATED', targetType: 'agent_configuration', targetId: id, beforeState, afterState }`
  - Retorna configuração atualizada
- [x] Implementar `AgentConfigService.getConfigurationById(id: string)`:
  - Lê do cache Redis `dani-admin:agent-config:{id}` primeiro (TTL=300s) — RM-003 + D10 `AGENT_CONFIG_CACHE_TTL_SECONDS=300`
  - Se não está no cache: busca no banco e popula o cache
  - Este método é usado por `SupervisionService.createInteraction` (S3 — resolve TODO)
- [x] Implementar `GET /api/v1/admin/agent-config` no `AgentConfigController`:
  - Guards: `@Roles('ADMIN')`
  - HTTP 200
- [x] Implementar `PATCH /api/v1/admin/agent-config/:id` no `AgentConfigController`:
  - Guards: `@Roles('ADMIN')`
  - HTTP 200

### FRONTEND (Feature 1)

- [x] Criar `apps/web/src/app/admin/config/page.tsx` — tela T-005 (Configurações de Supervisão) conforme D06:
  - Título: "Configurações de Supervisão" (D10 — `agent_configurations` → "Configurações de Supervisão")
  - Duas seções: uma para cada agente (DANI_CESSIONARIO, DANI_CEDENTE)
  - Para cada agente, exibir:
    - Campo "Nível de supervisão" (D10 — `confidence_threshold` → "Nível de supervisão"): Slider de 50–95 com passo 1; valor padrão 80
    - Campo "Rate limit do webchat" (D10): input numérico, min 1; label "Mensagens por hora"
    - Campo "Status do agente": Select com opções "Ativo", "Desligado automaticamente", "Modo degradado"
    - Campo "System prompt": Textarea, opcional
    - Badge de status do agente: "Ativo" / "Desligado automaticamente" / "Modo degradado" (D10)
  - Botão "Salvar configurações" — ao clicar, chama `PATCH /api/v1/admin/agent-config/:id`
  - Em sucesso: toast "Configurações salvas com sucesso." (D08)
  - Em erro 409 `DA-CFG-002`: toast de erro "Configuração foi alterada por outro administrador. Recarregue e tente novamente." (D08)
  - Loading state: skeleton form (D08)
- [x] Criar `apps/web/src/app/admin/config/_components/agent-config-form.tsx`:
  - Usa `react-hook-form` + `zod` para validação
  - Valida `confidenceThreshold` entre 50 e 95 client-side
  - Inclui campo oculto `version` para lock otimista

### WIRING (Feature 1)

- [x] Verificar que `PATCH /api/v1/admin/agent-config/:id` com `version=1` correto → atualiza e retorna `version=2`
- [x] Verificar que `PATCH` com `version=0` (desatualizado) → 409 `DA-CFG-002`
- [x] Verificar que cache Redis `dani-admin:agent-config:{id}` é invalidado após PATCH
- [x] Verificar que `SupervisionService.createInteraction` agora usa `AgentConfigService.getConfigurationById` em vez do hardcoded `80` (resolve TODO de S3)
- [x] Verificar que log `CONFIG_UPDATED` é criado em `admin_access_logs`

### TESTES (Feature 1)

- [x] Criar `agent-config.service.spec.ts` com Vitest:
  - Test: `listConfigurations()` retorna 2 registros (DANI_CESSIONARIO, DANI_CEDENTE)
  - Test: `updateConfiguration` com `version` correto → sucesso, `version` incrementado
  - Test: `updateConfiguration` com `version` desatualizado → 409 `DA-CFG-002`
  - Test: `updateConfiguration` com `id` inexistente → 404 `DA-CFG-001`
  - Test: `updateConfiguration` invalida cache Redis `dani-admin:agent-config:{id}`
  - Test: `getConfigurationById` lê do cache Redis antes de consultar banco
  - Test: `getConfigurationById` popula cache com TTL=300s quando cache está vazio
  - Test: log de auditoria criado com `action: 'CONFIG_UPDATED'`

---

## FEATURE 2 — Alertas — Disparo e Canais (RF-011, RF-012)

### BANCO (Feature 2)

- [x] Confirmar que `alert_events` tem `idempotency_key VARCHAR(255) UNIQUE NOT NULL`
- [x] Confirmar que campos `severity`, `channel`, `status`, `attempts`, `last_attempt_at`, `sent_at` existem
- [x] Confirmar que `AlertSeverity` enum tem valores `P0`, `P1`, `P2`, `P3`, `P4`, `P5`
- [x] Confirmar que `AlertChannel` enum tem valores `SLACK`, `EMAIL`, `PUSH`, `IN_APP`
- [x] Confirmar que `AlertStatus` enum tem valores `PENDENTE`, `ENVIADO`, `FALHOU`, `RETRY`

### BACKEND (Feature 2)

- [x] Criar `apps/api/src/modules/alerts/alerts.module.ts`
- [x] Criar `apps/api/src/modules/alerts/alerts.service.ts`
- [x] Criar `apps/api/src/modules/alerts/alerts.controller.ts`
- [x] Criar `apps/api/src/modules/alerts/alerts.consumer.ts` — consumer RabbitMQ
- [x] Criar `apps/api/src/modules/alerts/dto/create-alert.dto.ts`:
  - `agentId: string` — UUID
  - `severity: AlertSeverity`
  - `channel: AlertChannel`
  - `title: string`
  - `body: string`
  - `metadata?: object`
  - `idempotencyKey: string` — UUID ou hash gerado pelo chamador
- [x] Implementar `AlertsService.createAlert(dto: CreateAlertDto)`:
  - Verifica idempotência: `SELECT FROM alert_events WHERE idempotency_key = :key`
  - Se já existe com `status = 'ENVIADO'` → retorna registro existente sem criar novo (idempotência via Redis `dani-admin:alert:idempotency:{key}` com TTL=3600s — D21)
  - Se não existe: cria registro em `alert_events` com `status = 'PENDENTE'`
  - Publica mensagem na fila RabbitMQ `dani-admin.alerts` com routing key baseado no canal:
    - `SLACK` → `dani-admin.alerts.slack`
    - `EMAIL` → `dani-admin.alerts.email`
    - `PUSH` → `dani-admin.alerts.push`
    - `IN_APP` → processado diretamente (sem fila)
  - Retorna `{ alertId, status: 'PENDENTE' }`
- [x] Definir shape do payload RabbitMQ para alertas (todos os consumers) — [CORRIGIDO: FINDING-003]:
  - `{ alertEventId: string, agentId: string, severity: AlertSeverity, title: string, body: string, metadata?: object, idempotencyKey: string, attempt: number }`
  - Após processar com sucesso: `channel.ack(msg)`
  - Após falha com `attempts < 3`: `channel.nack(msg, false, true)` (requeue) com delay exponencial
  - Após 3 falhas (`attempts >= 3`): `channel.nack(msg, false, false)` (sem requeue → DLQ automaticamente via `x-dead-letter-exchange`)
- [x] Implementar `AlertsConsumer.handleSlackAlert(message)` — consumer da fila `dani-admin.alerts.slack`:
  - Chama `SlackService.sendWebhook(url, message)` — `SLACK_WEBHOOK_URL` de `process.env`
  - Em sucesso: atualiza `alert_events.status = 'ENVIADO'`, `sent_at = now()`
  - Em falha: incrementa `attempts`, atualiza `last_attempt_at`, define `status = 'RETRY'`
  - Após 3 falhas (`attempts >= 3`): `status = 'FALHOU'`, mensagem vai para DLQ `dani-admin.alerts.slack.dlq`
  - Retry exponencial: 1ª tentativa imediata, 2ª após 30s, 3ª após 120s
- [x] Implementar `AlertsConsumer.handleEmailAlert(message)` — consumer da fila `dani-admin.alerts.email`:
  - Chama `SendGridService.sendEmail(to, subject, body)` — `SENDGRID_API_KEY` de `process.env`
  - Mesma lógica de retry: máx 3 tentativas, DLQ após falha
- [x] Implementar `AlertsConsumer.handlePushAlert(message)` — consumer da fila `dani-admin.alerts.push`:
  - Chama `ExpoService.sendPushNotification(tokens, title, body)` — `EXPO_ACCESS_TOKEN` de `process.env`
  - Busca tokens ativos de `push_notification_tokens` pelo `admin_id`
  - Mesma lógica de retry e DLQ
- [x] Criar `apps/api/src/modules/alerts/slack.service.ts`:
  - `sendWebhook(url: string, payload: SlackWebhookPayload)`: POST para `SLACK_WEBHOOK_URL` com payload
  - Canal: `#alertas-dani-admin` (D17)
  - Em falha HTTP → lança `ServiceUnavailableException`
- [x] Criar `apps/api/src/modules/alerts/sendgrid.service.ts`:
  - `sendEmail(to: string, subject: string, htmlBody: string)`: POST para SendGrid API v3
  - From: configurado via `SENDGRID_FROM_EMAIL` env var
  - Em falha → lança `ServiceUnavailableException`
- [x] Criar `apps/api/src/modules/alerts/expo.service.ts`:
  - `sendPushNotification(tokens: string[], title: string, body: string)`: usa Expo Push API
  - Filtra tokens inválidos; em falha → lança `ServiceUnavailableException`

### FRONTEND (Feature 2)

- [x] Criar `apps/web/src/app/admin/alerts/page.tsx` — Tela de Alertas (sem número T — conforme D06, T-003 = Chat em Takeover implementado em S4) — [CORRIGIDO: FINDING-006]:
  - Lista de alertas `IN_APP` paginada (cursor-based)
  - Coluna "Severidade": badge colorido (P0=vermelho, P1=laranja, P2=amarelo, P3-P5=cinza)
  - Coluna "Título", "Data", "Status"
  - Loading state: skeleton
  - Empty state: "Nenhum alerta registrado."

### WIRING (Feature 2)

- [x] Verificar que `POST /api/v1/admin/alerts/test` (RF-013) cria alerta e publica na fila
- [x] Verificar que consumer Slack processa mensagem da fila e chama webhook
- [x] Verificar que 2ª chamada com mesmo `idempotencyKey` retorna alerta existente sem criar novo
- [x] Verificar que após 3 falhas, mensagem vai para DLQ

### TESTES (Feature 2)

- [x] Criar `alerts.service.spec.ts` com Vitest:
  - Test: `createAlert(dto)` com novo `idempotencyKey` → cria registro e publica na fila
  - Test: `createAlert(dto)` com `idempotencyKey` já `ENVIADO` → retorna existente sem criar novo
  - Test: `createAlert(dto)` para canal `SLACK` → publica em `dani-admin.alerts.slack`
  - Test: `createAlert(dto)` para canal `IN_APP` → não publica na fila
- [x] Criar `alerts.consumer.spec.ts`:
  - Test: `handleSlackAlert` em sucesso → `status = 'ENVIADO'`, `sent_at` preenchido
  - Test: `handleSlackAlert` em falha → `attempts` incrementado, `status = 'RETRY'`
  - Test: após 3 falhas → `status = 'FALHOU'`, mensagem rejeitada para DLQ

---

## FEATURE 3 — Teste Manual de Alertas (RF-013)

### BACKEND (Feature 3)

- [x] Implementar `PATCH /api/v1/admin/alerts/:id/acknowledge` no `AlertsController` — [CORRIGIDO: FINDING-009]:
  - Guards: `@Roles('ADMIN')`
  - Atualiza `alert_events.status = 'RECONHECIDO'` (valor D12/REQ-187: ATIVO, RECONHECIDO, RESOLVIDO)
  - Registra em `admin_access_logs`: `{ action: AdminActionType.ACESSO_PAINEL }`
  - HTTP 200
- [x] Implementar `POST /api/v1/admin/alerts/test` no `AlertsController`:
  - Guards: `@Roles('ADMIN')`
  - Body: `{ agentId: string, channel: AlertChannel, message?: string }`
  - Cria alerta de teste com `severity = P5` (baixa prioridade), `title = 'Teste de alerta'`, `body = message || 'Teste enviado pelo Admin'`
  - Gera `idempotencyKey` único para cada teste (UUID v4 gerado no handler)
  - Retorna `{ alertId, status: 'PENDENTE', channel }`

### TESTES (Feature 3)

- [x] Criar `alerts.controller.spec.ts`:
  - Test: `POST /api/v1/admin/alerts/test` com body válido → HTTP 201, retorna `alertId`
  - Test: sem role `ADMIN` → 403

---

## FEATURE 4 — Monitoramento de Estado do Agente (RF-011, RN-DA-031)

### BACKEND (Feature 4)

- [x] Criar `apps/api/src/modules/agent-config/agent-monitor.service.ts`:
  - Cron job `@Cron('*/1 * * * *')` (a cada 1 minuto): verifica métricas de erro por agente nos últimos 15 minutos
  - Lê métricas de `interactions`: conta `status != 'RESPONDIDA_PELA_IA'` como erro
  - Se taxa de erro > `AUTO_DISABLE_ERROR_RATE_THRESHOLD = 30%` em `AUTO_DISABLE_WINDOW_MINUTES = 15` minutos:
    - Atualiza `agent_configurations.status = 'DESLIGADO_AUTOMATICO'`
    - Cria alerta com `severity = P0`, `channel = SLACK`, `title = 'Agente desligado automaticamente'`
  - Se taxa de erro > `HIGH_ERROR_RATE_THRESHOLD = 10%` em 15 minutos:
    - Cria alerta com `severity = P1`, `channel = SLACK`, `title = 'Alta taxa de erros no agente'`
  - Se CSAT médio das últimas 24h < `DEGRADED_CSAT_THRESHOLD = 3.5`:
    - Cria alerta `severity = P2`, `title = 'CSAT abaixo do limite'`
  - Se taxa de recusa > `HIGH_REFUSAL_RATE_THRESHOLD = 20%` em 24h:
    - Cria alerta `severity = P2`, `title = 'Alta taxa de recusa'`
- [x] Verificar que cron job está registrado no `AgentConfigModule` com `@nestjs/schedule`

### TESTES (Feature 4)

- [x] Criar `agent-monitor.service.spec.ts`:
  - Test: taxa de erro = 35% em 15min → `status = 'DESLIGADO_AUTOMATICO'`, alerta P0 criado
  - Test: taxa de erro = 15% em 15min → alerta P1 criado, status NÃO muda
  - Test: taxa de erro = 5% → nenhum alerta criado
  - Test: CSAT < 3.5 → alerta P2 criado
  - Test: taxa de recusa = 25% → alerta P2 criado

---

## Cross-Módulo

- [x] Resolver TODO de S3: `SupervisionService.createInteraction` agora usa `AgentConfigService.getConfigurationById` para ler `confidence_threshold` real (substituir hardcoded `80`)
- [x] Resolver TODO de S4: `TakeoverService.startTakeover` agora publica evento em `AlertsService.createAlert` com `channel = 'IN_APP'` e `severity = P3`
- [x] `AgentConfigService` exporta `getConfigurationById` para uso em S7 (AgentService de IA)

---

## AUTO-VERIFICAÇÃO S5

| Check | Critério | Status |
|---|---|---|
| #1 Nomenclatura | `agent_configurations`, `alert_events`, `AlertSeverity` P0–P5, `AlertChannel` SLACK/EMAIL/PUSH/IN_APP, `AlertStatus` PENDENTE/ENVIADO/FALHOU/RETRY, `DA-CFG-001/002`, `dani-admin.alerts.*` usados exatamente | PASS |
| #2 Verificabilidade | Cada item binariamente verificável | PASS |
| #3 Valores numéricos | MIN_CONFIDENCE=50, MAX=95, DEFAULT=80, WEBCHAT_RATE=30, AUTO_DISABLE=30%, HIGH_ERROR=10%, CSAT=3.5, REFUSAL=20%, AGENT_CONFIG_CACHE=300s, retry máx 3 | PASS |
| #4 N itens completos | 4 features: config, alertas+canais, teste manual, monitor automático | PASS |
| #5 Máquinas de estado | `AgentStatus`: AGENTE_ATIVO → DESLIGADO_AUTOMATICO (trigger: erro >30%) | PASS |
| #6 Schedules/TTLs | Cron `*/1 * * * *`, cache Redis TTL=300s, idempotência Redis TTL=3600s, retry 0s/30s/120s | PASS |
| #7 Conflitos | Nenhum conflito novo | PASS |
| #8 Ambiguidades | Nenhuma ambiguidade nova | PASS |
| #9 Anti-scaffold | `AlertsConsumer` tem lógica real: retry exponencial, DLQ, status update | PASS |
| #10 Cross-módulo | TODOs de S3 e S4 resolvidos nesta sprint | PASS |
| #11 IDs de referência | RF-011 a RF-016, RF-023, RN-DA-031, D17, D21 citados | PASS |
| #12 Cobertura REQs S5 | Todos os REQs de alertas e config no registro-mestre.md têm ≥ 1 item | PASS |
