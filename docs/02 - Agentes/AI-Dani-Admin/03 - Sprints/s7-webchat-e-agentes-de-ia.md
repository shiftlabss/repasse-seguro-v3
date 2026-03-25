# S7 — Webchat e Agentes de IA

## Metadados

| Campo         | Valor                                                                                    |
| ------------- | ---------------------------------------------------------------------------------------- |
| Sprint        | S7                                                                                       |
| Nome          | Webchat e Agentes de IA                                                                  |
| Template      | B (Módulo Fullstack por Feature) — Sprint Condicional (IA)                               |
| Módulo        | AI-Dani-Admin                                                                            |
| Docs Fonte    | D05 (RF-024, RF-025, RNF-001 a RNF-003, RNF-009), D02, D10, D14, D17, D19, D20, D25, D26 |
| REQs Cobertos | REQs de S7 no registro-mestre.md                                                         |
| Data          | 2026-03-24                                                                               |

## Objetivo

Implementar o módulo de Agentes de IA: pipeline RAG de 7 passos (D19), SSE streaming, rate limiting por usuário (RN-DA-036), kill switch via PostHog, modo degradado (`FALLBACK_ATIVO`) quando OpenAI indisponível, Langfuse observabilidade, FAB global com badge de status degradado, e integração dos eventos PostHog (8 obrigatórios).

---

## FEATURE 1 — Pipeline do Agente de IA (RF-024, D19)

### BANCO (Feature 1)

- [x] Confirmar que extensão `pgvector` está habilitada (S1 — requisito de RAG)
- [x] Criar migration `010_create_vector_store` — tabela `documents` para vector store RAG:
  - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  - `agent_type AgentType NOT NULL`
  - `title TEXT NOT NULL`
  - `content TEXT NOT NULL`
  - `embedding vector(1536)` — dimensão do modelo `text-embedding-3-small` (OpenAI)
  - `metadata JSONB`
  - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  - `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  - Criar índice HNSW: `CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)` — busca por similaridade
- [x] Verificar que índice HNSW funciona com query `embedding <=> :vector ORDER BY embedding <=> :vector LIMIT 5`

### BACKEND (Feature 1)

- [x] Criar `apps/api/src/modules/agents/agents.module.ts`
- [x] Criar `apps/api/src/modules/agents/agents.service.ts`
- [x] Criar `apps/api/src/modules/agents/rag.service.ts`
- [x] Criar `apps/api/src/modules/agents/langfuse.service.ts`
- [x] Criar `apps/api/src/modules/agents/posthog.service.ts`
- [x] Implementar `RagService.retrieveDocuments(query: string, agentType: AgentType, topK: number = 5)`:
  - Gera embedding do `query` via OpenAI Embeddings API (`text-embedding-3-small`)
  - Busca top-5 documentos mais similares em `documents` via `embedding <=> :vector`
  - Filtra por `agent_type` correspondente
  - Retorna array de `{ title, content, metadata }`
- [x] Implementar `AgentsService.generateResponse(dto: GenerateResponseDto)` — pipeline de 7 passos (D19):
  - **Passo 1 — Validar scope**: verifica que `agentId` pertence ao `userId` autenticado (filtro de escopo — RN-DA-037)
  - **Passo 2 — Verificar kill switch**: checa feature flag `webchat-enabled` via PostHog; se desabilitado → retorna `{ fallback: true, message: 'Serviço temporariamente indisponível.' }`
  - **Passo 3 — Verificar rate limit**: `dani-admin:rate-limit:{userId}:{hour}` no Redis (janela deslizante 1h, máx 30 — WEBCHAT_RATE_LIMIT_DEFAULT); se excedido → HTTP 429 com erro `DA-AGT-001`
  - **Passo 4 — Recuperar contexto (RAG)**: chama `RagService.retrieveDocuments(userMessage, agentType)`
  - **Passo 5 — Construir prompt**: monta system prompt do agente (lido de `agent_configurations.system_prompt`) + documentos RAG + histórico de conversa (últimas 10 mensagens da sessão)
  - **Passo 6 — Gerar resposta via LLM**: chama OpenAI GPT-4o via Vercel AI SDK 4.x+ com streaming SSE; calcula `confidenceScore` (0–100) baseado em probabilidades do modelo ou heurística; mede `latencyMs`
  - **Passo 7 — Persistir e observar**: chama `SupervisionService.createInteraction(dto)` para criar interação; envia trace para Langfuse (fire-and-forget); rastreia evento PostHog `interaction_completed`
  - Em falha da OpenAI API → ativa `FALLBACK_ATIVO`: atualiza `agent_configurations.status = 'FALLBACK_ATIVO'`, retorna resposta da Calculadora de Comissão (módulo determinístico)
- [x] Implementar `POST /api/v1/agents/:agentType/chat` (endpoint público para webchat):
  - Guards: JWT do usuário (não apenas ADMIN — usuários Cessionário/Cedente acessam)
  - Body: `{ message: string, sessionId: string }`
  - Resposta: SSE stream com tokens parciais e evento final `{ done: true, confidenceScore, interactionId }`
  - Header `Content-Type: text/event-stream`, `Cache-Control: no-cache`

### FRONTEND (Feature 1)

> Webchat é implementado nos apps de usuário (apps/web para Cessionários/Cedentes), não no painel Admin. Esta seção cobre a integração do status do agente no painel Admin (FAB global e badge de degradação).

- [x] Criar `apps/web/src/components/fab-chat.tsx` — FAB global (D10 — "FAB global"):
  - Ícone flutuante fixo em todas as telas do módulo do usuário
  - Posição: `fixed bottom-4 right-4`
  - Badge de status degradado quando `agent_configurations.status = 'FALLBACK_ATIVO'` — texto "Modo degradado" (D10)
  - Badge verde quando `AGENTE_ATIVO`
  - Badge vermelho quando `DESLIGADO_AUTOMATICO` — texto "Desligado automaticamente"
- [x] Criar `apps/web/src/components/chat-widget.tsx` — janela de chat SSE:
  - Conecta via `EventSource` ao `POST /api/v1/agents/:agentType/chat`
  - Renderiza tokens parciais em tempo real (streaming)
  - Exibe spinner durante streaming
  - Exibe indicador de confiança quando `confidenceScore` disponível

### WIRING (Feature 1)

- [x] Verificar que `POST /api/v1/agents/DANI_CESSIONARIO/chat` com JWT válido retorna stream SSE
- [x] Verificar que rate limit bloqueia após 30 mensagens na mesma hora
- [x] Verificar que kill switch `webchat-enabled = false` retorna fallback imediatamente
- [x] Verificar que falha da OpenAI ativa `FALLBACK_ATIVO` e FAB exibe badge "Modo degradado"

### TESTES (Feature 1)

- [x] Criar `agents.service.spec.ts` com Vitest:
  - Test: pipeline passo 3 — após 30 mensagens no Redis, lança HTTP 429 `DA-AGT-001`
  - Test: pipeline passo 2 — kill switch `webchat-enabled = false` retorna `{ fallback: true }`
  - Test: pipeline passo 1 — `agentId` pertencendo a outro `userId` lança 403 (filtro de escopo RN-DA-037)
  - Test: pipeline passo 6 — em falha OpenAI, `status = 'FALLBACK_ATIVO'` atualizado
  - Test: pipeline passo 7 — `createInteraction` chamado com dados corretos
- [x] Criar `rag.service.spec.ts`:
  - Test: `retrieveDocuments` retorna top-5 documentos ordenados por similaridade
  - Test: filtro por `agent_type` aplicado na query pgvector

---

## FEATURE 2 — Langfuse Observabilidade (D25, RF-025)

### BACKEND (Feature 2)

- [x] Implementar `LangfuseService.traceInteraction(dto: LangfuseTraceDto)` — fire-and-forget:
  - Cria trace no Langfuse com: `name`, `input` (user_message), `output` (agent_response), `model` (gpt-4o), `usage` (tokens), `metadata` (agentType, confidenceScore, latencyMs)
  - Chamada assíncrona — não bloqueia resposta ao usuário; erro não lança exceção
  - `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST` via env vars
- [x] Verificar que `langfuse_trace_id` é armazenado em `interactions.langfuse_trace_id` para rastreabilidade
- [x] Verificar que Langfuse alimenta métricas de custo, latência e qualidade visíveis no dashboard Admin (link via `langfuse_trace_id`)

### TESTES (Feature 2)

- [x] Criar `langfuse.service.spec.ts`:
  - Test: `traceInteraction` chama SDK Langfuse com parâmetros corretos
  - Test: falha do Langfuse SDK NÃO lança exceção (fire-and-forget)
  - Test: `langfuse_trace_id` retornado e armazenado na interação

---

## FEATURE 3 — PostHog Events e Kill Switch (D25, RN-DA-036)

### BACKEND (Feature 3)

- [x] Implementar `PostHogService.trackEvent(event: string, properties: object)`:
  - Usa `posthog-node` SDK com `POSTHOG_API_KEY`
  - Fire-and-forget — não bloqueia
- [x] Implementar `PostHogService.isFeatureEnabled(flag: string): Promise<boolean>`:
  - Verifica feature flag via PostHog API
  - Cache local de 60s para evitar latência em cada request
- [x] Registrar os 8 eventos PostHog obrigatórios (D25) nos services corretos:
  - `admin_takeover_initiated` — em `TakeoverService.startTakeover` (S4) — resolver TODO
  - `admin_takeover_ended` — em `TakeoverService.endTakeover` (S4) — resolver TODO
  - `agent_auto_disabled` — em `AgentMonitorService` (S5) — resolver TODO
  - `agent_config_updated` — em `AgentConfigService.updateConfiguration` (S5) — resolver TODO
  - `interaction_completed` — em `AgentsService.generateResponse` (S7)
  - `interaction_flagged_for_review` — em `SupervisionService.createInteraction` quando `status = SINALIZADA_PARA_REVISAO` (S3) — resolver TODO
  - `alert_sent` — em `AlertsConsumer` após envio bem-sucedido (S5) — resolver TODO
  - `launch_readiness_approved` — em `LaunchReadinessService.approveChecklist` (S8)
- [x] Verificar que `webchat-enabled` feature flag está registrado no PostHog como kill switch

### FRONTEND (Feature 3)

- [x] Instalar `posthog-js` no `apps/web/`
- [x] Inicializar PostHog em `apps/web/src/app/layout.tsx` com `POSTHOG_API_KEY`
- [x] Rastrear eventos de UI via PostHog: `admin_page_viewed`, `filter_applied`, `export_clicked`

### TESTES (Feature 3)

- [x] Criar `posthog.service.spec.ts`:
  - Test: `isFeatureEnabled('webchat-enabled')` retorna `true` quando flag habilitada
  - Test: `isFeatureEnabled('webchat-enabled')` retorna `false` quando flag desabilitada
  - Test: falha do PostHog SDK NÃO lança exceção

---

## FEATURE 4 — Rate Limiting do Webchat (RN-DA-036)

### BACKEND (Feature 4)

- [x] Implementar `RateLimitService.checkWebchatRateLimit(userId: string, agentId: string)`:
  - Chave Redis: `dani-admin:rate-limit:{userId}:{agentId}:{hour}` — janela deslizante de 1 hora
  - Incrementa contador via `INCR` + `EXPIRE`
  - Se contador > `webchat_rate_limit` (lido de `agent_configurations.webchat_rate_limit`) → retorna `{ allowed: false, retryAfterSeconds: X }`
  - Padrão: 30 mensagens/hora (WEBCHAT_RATE_LIMIT_DEFAULT=30)
- [x] Verificar que rate limit usa o `webchat_rate_limit` configurado por agente (não hardcoded 30)

### TESTES (Feature 4)

- [x] Criar `rate-limit.service.spec.ts`:
  - Test: 30ª mensagem → permitida
  - Test: 31ª mensagem → bloqueada, retorna `{ allowed: false }`
  - Test: nova hora (chave Redis diferente) → contador reset, mensagem permitida

---

## 🔀 Cross-Módulo

- [x] Resolver TODOs de S4 (TakeoverService): registrar `admin_takeover_initiated` e `admin_takeover_ended` via `PostHogService`
- [x] Resolver TODOs de S5 (AgentMonitorService, AgentConfigService, AlertsConsumer): registrar eventos PostHog via `PostHogService`
- [x] Resolver TODO de S3 (SupervisionService.createInteraction): registrar `interaction_flagged_for_review` via `PostHogService`
- [x] `AgentsService` importa `AgentConfigService` (S5) para ler `system_prompt` e `confidence_threshold`
- [x] `AgentsService` importa `SupervisionService` (S3) para `createInteraction`

---

## AUTO-VERIFICAÇÃO S7

| Check                 | Critério                                                                                                                                          | Status |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura       | `documents` (vector store), `text-embedding-3-small`, `gpt-4o`, `webchat-enabled` (kill switch), `DA-AGT-001`, `FALLBACK_ATIVO` usados exatamente | ✅     |
| #2 Verificabilidade   | Cada item binariamente verificável                                                                                                                | ✅     |
| #3 Valores numéricos  | WEBCHAT_RATE_LIMIT_DEFAULT=30, topK=5 documentos RAG, histórico 10 mensagens, PostHog cache 60s                                                   | ✅     |
| #4 N itens completos  | 4 features: pipeline IA, Langfuse, PostHog, rate limit                                                                                            | ✅     |
| #5 Máquinas de estado | `AGENTE_ATIVO` → `FALLBACK_ATIVO` (trigger: OpenAI indisponível); `FALLBACK_ATIVO` → `AGENTE_ATIVO` (trigger: OpenAI recupera)                    | ✅     |
| #6 Schedules/TTLs     | Rate limit janela 1h; PostHog flag cache 60s; fire-and-forget para Langfuse/PostHog                                                               | ✅     |
| #7 Conflitos          | RM-002 resolvido (Vitest usado); RM-001 `LATENCY_SLA_SECONDS` pendente                                                                            | ✅     |
| #8 Ambiguidades       | Nenhuma nova                                                                                                                                      | ✅     |
| #9 Anti-scaffold      | Pipeline de 7 passos com lógica real em cada passo; rate limit com janela deslizante real                                                         | ✅     |
| #10 Cross-módulo      | 6 TODOs resolvidos (PostHog events) + importação de AgentConfigService e SupervisionService                                                       | ✅     |
| #11 IDs de referência | D19, D25, RN-DA-036, RN-DA-037, RF-024, RF-025, ADR-001 (para filtro de escopo) citados                                                           | ✅     |
| #12 Cobertura REQs S7 | Todos os REQs de IA no registro-mestre.md têm ≥ 1 item                                                                                            | ✅     |

---

## ⚠️ Flags Pendentes

- **RM-001** `LATENCY_SLA_SECONDS = [DADO PENDENTE]`: SLA de latência não definido. Não é possível implementar alerta automático por latência. Marcar no `AgentMonitorService` como `[PENDENTE — REVISÃO MANUAL]`.
