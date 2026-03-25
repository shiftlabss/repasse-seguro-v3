# S3b — Agente Core (Parte 2): Streaming, Persistência, Langfuse e API

<!-- Atualizado em 2026-03-24 pela A03 — 4 correções aplicadas (FINDING-002, FINDING-008, FINDING-017, FINDING-020) -->

| Campo              | Valor                                                                                                                                                                                                                                                                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S3b                                                                                                                                                                                                                                                                                                                                         |
| **Nome**           | Agente Core — Parte 2: Streaming, Persistência, Langfuse e API                                                                                                                                                                                                                                                                              |
| **Template**       | B — Módulo Fullstack                                                                                                                                                                                                                                                                                                                        |
| **Docs Fonte**     | 01.1-RN-Fundação, 01.3-RN-Operação, 14-EspecTécnicas, 16-API, 19-CriaçãoAgentes                                                                                                                                                                                                                                                             |
| **REQs cobertos**  | REQ-014, REQ-015, REQ-016, REQ-017, REQ-018, REQ-019, REQ-020, REQ-021, REQ-038, REQ-039, REQ-093, REQ-112, REQ-113, REQ-120, REQ-121, REQ-122, REQ-123, REQ-124, REQ-125, REQ-126, REQ-127, REQ-128, REQ-129, REQ-130, REQ-131, REQ-132, REQ-133, REQ-134, REQ-159, REQ-160, REQ-161, REQ-162, REQ-163, REQ-174, REQ-175, REQ-176, REQ-177 |
| **Total de itens** | 28                                                                                                                                                                                                                                                                                                                                          |
| **Status**         | ✅ Concluída                                                                                                                                                                                                                                                                                                                                |

---

## 🎯 Objetivo

Completar o agente de IA: nós GENERATE, STREAM (SSE via Vercel AI SDK), PERSIST e LANGFUSE do grafo LangGraph; system prompt com 3-layer isolation; endpoint `POST /ai/chat` e `GET /ai/chat/stream`; persistência de mensagens e interações; observabilidade Langfuse; e SLAs de latência (≤5s análise, ≤10s comparação).

---

## 🏗️ FEATURE 3: Nós GENERATE, STREAM, PERSIST, LANGFUSE

### 1. Nó `GENERATE` — geração de resposta GPT-4o

- [x] Implementar nó `generateNode(state: AgentState): Partial<AgentState>`:
  - Montar array de mensagens para GPT-4o: `[systemMessage, ...state.messages, ...toolResultMessages]`
  - **System prompt com 3-layer isolation** (ver item 2 abaixo)
  - Chamar GPT-4o com: `model: process.env.OPENAI_MODEL` (não hardcoded), `temperature: 0.1`, `max_tokens: 4096`, `parallel_tool_calls: false`, `stream: true`
  - Capturar `usage.prompt_tokens`, `usage.completion_tokens`, `usage.total_tokens` para persistência
  - Retornar `{ response: string, metadata: { ...state.metadata, promptTokens, completionTokens, totalTokens } }`
- [x] Se GPT-4o retornar erro 429 (rate limit) → retry 2x com backoff exponencial (1s, 2s); se persistir → retornar `{ error: { code: 'LLM_RATE_LIMIT', recoverable: true } }`
- [x] Se GPT-4o retornar erro 500+ → retornar `{ error: { code: 'LLM_UNAVAILABLE', recoverable: false } }` — acionar fallback `CalculadoraModule` para intents de análise
- [x] Teste unitário: mock OpenAI SDK → retornar resposta válida com usage; mock 429 → retry 2x → retornar erro; mock 500 → error.code = 'LLM_UNAVAILABLE' imediatamente

### 2. System prompt — isolamento em 3 camadas (Camada 3)

- [x] Criar `SystemPromptBuilder` com método `build(user: AuthenticatedUser, context: DocumentEmbedding[], conversationContext: string)`:
  - **Bloco 1 — Identidade:** "Você é o Repasse AI, Analista de Oportunidades especializado em crédito consignado para o Cessionário [nome]. Você JAMAIS revela informações de outros cessionários. Ao relatar score de risco, SEMPRE incluir: valor numérico (1-10), label de cor (Baixo=1-3 verde/Médio=4-6 amarelo/Alto=7-10 vermelho), e lista dos 3 principais fatores considerados. Formato obrigatório no JSON de resposta: campo `riskScore: { value: number, label: string, color: 'green'|'yellow'|'red', factors: string[] }`" — [CORRIGIDO: FINDING-002]
  - **Bloco 2 — Escopo de dados:** "Você SOMENTE analisa oportunidades vinculadas ao Cessionário com ID [userId]. Qualquer pergunta sobre outros cessionários deve ser respondida com 'Não tenho acesso a essas informações.'"
  - **Bloco 3 — Contexto RAG:** inserir conteúdo dos documentos recuperados em `state.context`
  - **Bloco 4 — Instruções de formato:** resposta em PT-BR, markdown permitido apenas para tabelas e listas, sem formatação HTML
- [x] Validar que `userId` e `organizationId` do payload JWT estão sempre no system prompt (não opcional)
- [x] Teste unitário: `build(userA, docs, ctx)` → system prompt contém `userId` de userA; `build(userB, ...)` → system prompt contém `userId` de userB (não de userA)

### 3. Nó `STREAM` — SSE via Vercel AI SDK

- [x] Implementar nó `streamNode(state: AgentState): Partial<AgentState>`:
  - Se `state.cacheHit === true` → emitir resposta do cache diretamente como único chunk SSE (sem chamada ao GPT)
  - Se `state.cacheHit === false` → usar `streamText()` do Vercel AI SDK 4.x+ para stream do GPT-4o
  - Emitir eventos SSE no formato: `data: {"type":"chunk","content":"...","delta":"..."}\n\n`
  - Emitir evento final: `data: {"type":"done","content":"[resposta completa]","usage":{"promptTokens":N,"completionTokens":N}}\n\n`
  - Emitir `data: [DONE]\n\n` após evento final (compatibilidade com SSE clients)
- [x] Timeout de stream em 2 fases — [CORRIGIDO: FINDING-020]: (1) se nenhum chunk recebido em 10 segundos → emitir `data: {"type":"timeout_warning","message":"Análise demorando mais que o esperado. Aguarde..."}\n\n` e continuar; (2) se nenhum chunk em 120 segundos total → emitir `data: {"type":"error","message":"Timeout"}\n\n` e encerrar via `AbortController.abort()`
- [x] Teste unitário: mock Vercel AI SDK → confirmar sequência de eventos SSE; cache hit → um único chunk SSE + done; sem chunk em 10s → evento `timeout_warning`; sem chunk em 120s → evento `error` + abort

### 4. Nó `PERSIST` — persistência de mensagens e interações

- [x] Implementar nó `persistNode(state: AgentState): Partial<AgentState>`:
  - INSERT em `chat_messages` (tabela append-only): mensagem do usuário (`role: 'user'`) + resposta do assistente (`role: 'assistant'`, `tokens_used: totalTokens`)
  - INSERT em `ai_interactions`: `{ conversation_id, message_id, model: OPENAI_MODEL, prompt_tokens, completion_tokens, total_tokens, latency_ms, confidence_score: state.confidence, tool_calls: state.toolResults, cache_hit: state.cacheHit }`
  - Se `state.cacheHit === true` → não persistir nova interação (cache hit = sem nova inferência)
  - Se persistência falhar → logar erro via Pino e continuar (não bloquear resposta ao usuário — fire-and-forget para persistência)
  - **Gravar cache LLM — [CORRIGIDO: FINDING-008]:** Após INSERT em `ai_interactions` quando `cacheHit=false`: INSERT em `llm_cache_entries` com `cache_type='exact'`, `cache_key='ai:cache:exact:{sha256(lastUserMessage)}'`, `expires_at=NOW()+24h`, `response=state.response`; também INSERT com `cache_type='semantic'`, `embedding=(embedding da mensagem)`, `expires_at=NOW()+1h`. Ambos fire-and-forget.
- [x] Atualizar `chat_conversations.updated_at` via Prisma middleware após INSERT nas mensagens
- [x] Teste unitário: mock PrismaService → INSERT chat_message + ai_interaction em sequência; cache hit → skip INSERT ai_interaction; INSERT falha → erro logado, `state.error` NÃO atualizado (persistência é non-blocking)

### 5. Nó `LANGFUSE` — observabilidade

- [x] Implementar nó `langfuseNode(state: AgentState): Partial<AgentState>`:
  - Chamar `LangfuseTracer.endTrace(state.metadata.traceId, state.response, metadata)` com campos:
    - `model`, `promptTokens`, `completionTokens`, `totalTokens`, `latencyMs`
    - `intent`, `cacheHit`, `cacheType`, `nodeHistory: state.metadata.nodeHistory`
    - `toolsUsed: state.toolResults.map(t => t.toolName)`
  - Registrar custo estimado: `LangfuseTracer.logCost(traceId, model, promptTokens, completionTokens)`
  - **Fire-and-forget:** falha no Langfuse NÃO propaga erro para o usuário
  - **PII:** não enviar conteúdo das mensagens para Langfuse — apenas metadados
- [x] Teste unitário: mock Langfuse → confirmar `endTrace` chamado uma vez; falha no Langfuse → nó retorna `{}` sem erro; verificar que conteúdo de mensagens não aparece no payload do trace

---

## 🏗️ FEATURE 4: Endpoints `AiController`

### 6. Endpoint `POST /ai/chat` — enviar mensagem ao agente

- [x] Criar `AiController` com endpoint `POST /ai/chat` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `SendMessageDto`: `{ conversationId: z.string().uuid(), message: z.string().min(1).max(4096) }`
- [x] `AiService.sendMessage(user, dto)`:
  1. Verificar que `conversationId` existe e pertence ao usuário (via `ChatService.getConversation`)
  2. Verificar `LgpdService.hasValidConsent(user.userId, 'ai_analysis')` → se false → `403`
  3. Verificar rate limit (via `RateLimitGuard` — herdado de S2)
  4. Verificar se há takeover ativo para a conversa → se sim, retornar `{ takeover: true, message: "Um atendente humano está gerenciando esta conversa." }`
  5. Montar `AgentState` inicial e executar `agent.invoke(state)`
  6. Retornar `MessageResponseDto`
- [x] `MessageResponseDto`: `{ id, conversationId, role: 'assistant', content, metadata: { intent, cacheHit, tokensUsed, latencyMs }, createdAt }`
- [x] Resposta HTTP: `200 OK` com `MessageResponseDto`
- [x] SLA: ≤5s para análise simples, ≤10s para comparação — logar aviso via Pino se ultrapassar
- [x] Teste de integração: POST mensagem válida → `200` com response; `conversationId` inexistente → `404`; sem consentimento LGPD → `403`; rate limit excedido → `429`; takeover ativo → `200` com `takeover: true`

### 7. Endpoint `GET /ai/chat/stream` — SSE streaming

- [x] Criar endpoint `GET /ai/chat/stream` (`@Roles('CESSIONARIO', 'ADMIN')`) como SSE endpoint
- [x] Query params: `conversationId: string (uuid)`, `message: string (max 4096 chars)`
- [x] Configurar headers SSE: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`, `X-Accel-Buffering: no`
- [x] Usar `@nestjs/event-emitter` ou `Observable` do RxJS para stream
- [x] Emitir eventos conforme formato definido no nó STREAM (item 3): `{"type":"chunk",...}`, `{"type":"done",...}`, `[DONE]`
- [x] Desconexão do cliente → cancelar operação em andamento no grafo (via `AbortController`)
- [x] Teste de integração: GET stream → receber chunks SSE em sequência; desconexão → sem leak de memória; `conversationId` inválido → `data: {"type":"error","message":"Conversa não encontrada"}\n\n` e fechar stream

### 8. Endpoint `GET /ai/chat/history/:conversationId` — histórico de mensagens

- [x] Criar endpoint `GET /ai/chat/history/:conversationId` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] Query params: `page` (default 1), `per_page` (default 20, max 100)
- [x] `AiService.getChatHistory(user, conversationId, page, per_page)`:
  1. Verificar escopo (conversa pertence ao usuário via `ChatService.getConversation`)
  2. SELECT em `chat_messages` com filtro `conversation_id` + paginação + `ORDER BY created_at ASC`
  3. Retornar `{ data: MessageDto[], meta: { page, per_page, total, total_pages } }`
- [x] `MessageDto`: `{ id, role, content, tokensUsed, createdAt }` — sem `updatedAt` (tabela append-only)
- [x] Teste de integração: GET history → mensagens ordenadas por `created_at ASC`; conversa de outro usuário → `404`; `per_page=101` → `400`

### 9. Endpoint `GET /ai/status` — status do agente

- [x] Criar endpoint `GET /ai/status` (`@Public()` — sem autenticação)
- [x] Retornar último registro de `agent_status_logs` por `created_at DESC LIMIT 1`
- [x] Resposta: `{ status: AgentStatus, lastCheckedAt: string, message: string | null }`
- [x] Resposta HTTP: `200 OK` sempre (não retornar 503 por status degradado — apenas informar o status)
- [x] Teste: mock DB com status `online` → `200 { status: "online" }`; mock DB com status `maintenance` → `200 { status: "maintenance" }`; DB offline → `200 { status: "unknown", message: "Status indisponível" }`

---

## 🏗️ FEATURE 5: Memória Redis + Histórico Supabase

### 10. Memória de curto prazo — Redis (sessão ativa)

- [x] Criar `ConversationMemoryService` com Redis como storage:
  - Chave: `ai:session:{conversationId}` — TTL 30 minutos (reset a cada mensagem nova)
  - Armazenar últimas 20 mensagens no formato `BaseMessage[]` (LangChain)
  - Método `getMessages(conversationId)`: GET + deserializar JSON
  - Método `addMessage(conversationId, message)`: RPUSH + LTRIM para manter max 20 + EXPIRE 30min
  - Método `clearSession(conversationId)`: DEL key
- [x] Ao iniciar `AgentState`, pré-popular `messages` com histórico Redis → se Redis miss, buscar do Supabase (`chat_messages`) últimas 20 msgs
- [x] Teste unitário: `addMessage` 20x → 20 mensagens; `addMessage` 21x → 20 mensagens (trim); `getMessages` após TTL expirar → retornar `[]`; Redis offline → fallback Supabase

### 11. Memória de longo prazo — Supabase (retenção 90 dias)

- [x] Validar que `PERSIST` node (item 4) INSERT em `chat_messages` é a fonte da memória de longo prazo
- [x] Criar `ConversationMemoryService.loadFromDatabase(conversationId, limit=20)`: SELECT em `chat_messages` ORDER BY `created_at DESC LIMIT 20` → retornar em ordem ASC (reverter array)
- [x] Retenção: `LgpdRetentionJob` (definido em S2) soft-deleta conversas com `created_at < NOW() - INTERVAL '90 days'` — confirmar que `chat_messages` com `conversation_id` de conversa deletada também são soft-deletados em cascata (via FK ou job separado)
- [x] Teste unitário: `loadFromDatabase` retorna ≤20 mensagens em ordem ASC; conversa soft-deleted → `loadFromDatabase` retorna `[]`

---

## ✅ TESTES — Agente Core Parte 2

### 12. Suite de testes Streaming + API

- [x] Criar `test/unit/ai/agent/generate.node.spec.ts` — cobrir GPT-4o config (temp 0.1, max_tokens 4096), retry 429, fallback 500
- [x] Criar `test/unit/ai/agent/stream.node.spec.ts` — cobrir sequência SSE, cache hit, timeout 30s
- [x] Criar `test/unit/ai/agent/persist.node.spec.ts` — cobrir append-only, cache hit skip, falha non-blocking
- [x] Criar `test/unit/ai/agent/langfuse.node.spec.ts` — cobrir fire-and-forget, sem PII no trace
- [x] Criar `test/unit/ai/services/system-prompt.builder.spec.ts` — cobrir isolamento por userId
- [x] Criar `test/unit/ai/services/conversation-memory.service.spec.ts` — cobrir TTL 30min, max 20 msgs, fallback Supabase
- [x] Criar `test/integration/ai/ai.controller.spec.ts` — cobrir `POST /ai/chat` e `GET /ai/chat/stream` e `GET /ai/chat/history/:id`
- [x] Criar `test/e2e/ai/full-conversation.e2e-spec.ts` — E2E flow: criar conversa → enviar mensagem → receber stream → verificar histórico (1 dos 3 fluxos E2E obrigatórios do Doc 27)
- [x] Cobertura mínima módulo `ai/`: 85% (conforme Doc 27)

---

## 🔀 Cross-Módulo

- [x] **→ S4 (Calculadora):** nó `GENERATE` lança `CalculadoraModule.fallback()` se `error.code === 'LLM_UNAVAILABLE'` para intents de cálculo — verificar que `CalculadoraModule` está disponível como provider
- [x] **→ S6 (Admin):** `ai_interactions` persistidas aqui são a fonte de dados do painel Admin; `agent_status_logs` consultados em `GET /ai/status`
- [x] **→ S2 (Auth):** `JwtAuthGuard`, `@Roles()`, `@CurrentUser()`, `LgpdService.hasValidConsent()` usados diretamente nesta sprint — confirmar que `AiModule` importa `AuthModule` e `LgpdModule`

---

## ✔️ Auto-Verificação S3b (12 Checks)

| #   | Check                                                                                                                                                                                | Status |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                         | ✅     |
| 2   | Nomes exatos: `streamText`, `AgentState`, `SystemPromptBuilder`, `ConversationMemoryService`, `ai:session:{conversationId}`, `chat_messages`, `ai_interactions`, `agent_status_logs` | ✅     |
| 3   | Valores numéricos: temperature 0.1, max_tokens 4096, TTL Redis 30min, max 20 msgs Redis, timeout stream 30s, SLA análise ≤5s, SLA comparação ≤10s, retenção 90 dias                  | ✅     |
| 4   | Glossário consultado — Takeover, Score de Risco, Δ, LGPD, Cessionário                                                                                                                | ✅     |
| 5   | Anti-Scaffold R10: system prompt com blocos de isolamento reais; SSE com eventos tipados; PERSIST non-blocking documentado                                                           | ✅     |
| 6   | Nós GENERATE, STREAM, PERSIST, LANGFUSE completam o ciclo de 11 nós iniciado em S3a                                                                                                  | ✅     |
| 7   | Fire-and-forget documentado: PERSIST e LANGFUSE não bloqueiam resposta ao usuário                                                                                                    | ✅     |
| 8   | Sem conflitos não marcados                                                                                                                                                           | ✅     |
| 9   | `parallel_tool_calls: false` e `temperature: 0.1` no nó GENERATE; `OPENAI_MODEL` via env (não hardcoded)                                                                             | ✅     |
| 10  | Template B: organizado por FEATURE (Feature 3: Nós, Feature 4: Endpoints, Feature 5: Memória)                                                                                        | ✅     |
| 11  | 3 camadas de isolamento presentes: scope (S2), context filter (S2), agent instruction (System Prompt — neste sprint)                                                                 | ✅     |
| 12  | REQs cobertos: REQ-014 a REQ-021, REQ-038, REQ-039, REQ-093, REQ-112, REQ-113, REQ-120 a REQ-134, REQ-159 a REQ-163, REQ-174 a REQ-177 — todos com ≥1 item                           | ✅     |
