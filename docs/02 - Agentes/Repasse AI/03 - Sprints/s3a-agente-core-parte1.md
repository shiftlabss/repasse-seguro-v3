# S3a — Agente Core (Parte 1): Fluxo LangGraph + RAG + Tools

<!-- Atualizado em 2026-03-24 pela A03 — 2 correções aplicadas (FINDING-005, FINDING-028) -->

| Campo              | Valor                                                                                                                                                                                                                  |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S3a                                                                                                                                                                                                                    |
| **Nome**           | Agente Core — Parte 1: Fluxo LangGraph + RAG + Tools                                                                                                                                                                   |
| **Template**       | B — Módulo Fullstack                                                                                                                                                                                                   |
| **Docs Fonte**     | 01.2-RN-Core, 01.3-RN-Operação, 14-EspecTécnicas, 16-API, 19-CriaçãoAgentes                                                                                                                                            |
| **REQs cobertos**  | REQ-008, REQ-009, REQ-010, REQ-011, REQ-012, REQ-013, REQ-024, REQ-025, REQ-026, REQ-027, REQ-029, REQ-030, REQ-031, REQ-032, REQ-036, REQ-062, REQ-063, REQ-064, REQ-065, REQ-066, REQ-067, REQ-073, REQ-075, REQ-076 |
| **Total de itens** | 30                                                                                                                                                                                                                     |
| **Status**         | ✅ Concluída                                                                                                                                                                                                           |

---

## 🎯 Objetivo

Implementar a infraestrutura core do agente de IA: ciclo LangGraph.js completo (11 nós), `AgentState` com 9 campos, configuração GPT-4o, cache LLM (exact + semantic), RAG com pgvector, e os 5 tools com timeouts e retries. Esta é a Parte 1 — a Parte 2 (S3b) cobre streaming SSE, persistência, Langfuse, endpoint `/ai/chat`, e system prompt.

---

## 🏗️ FEATURE 1: Ciclo LangGraph.js — Grafo e Estado

### 1. `AgentState` — schema completo com 9 campos

- [x] Criar interface `AgentState` com exatamente 9 campos conforme Doc 19:
  1. `messages: BaseMessage[]` — histórico de mensagens (LangChain format)
  2. `intent: string | null` — intenção classificada pelo nó INTENT
  3. `context: DocumentEmbedding[]` — documentos recuperados pelo RAG
  4. `toolResults: ToolResult[]` — resultados dos tools executados
  5. `cacheHit: boolean` — se a resposta veio do cache
  6. `cacheType: 'exact' | 'semantic' | null` — tipo de cache hit
  7. `response: string | null` — resposta final gerada
  8. `error: AgentError | null` — erro capturado em qualquer nó
  9. `metadata: AgentMetadata` — dados de rastreabilidade: `{ traceId, userId, conversationId, startedAt, nodeHistory: string[] }`
- [x] `AgentState` deve ser imutável por convenção — cada nó retorna um novo objeto parcial (spread)
- [x] Criar `AgentMetadata` interface e `AgentError` interface com campos: `{ code: string, message: string, node: string, recoverable: boolean }`
- [x] Teste unitário: criar `AgentState` inicial → verificar todos os 9 campos presentes; simular nó retornando estado parcial → verificar spread correto sem mutação

### 2. Grafo LangGraph.js — 11 nós em sequência

- [x] Criar `AgentGraph` usando `StateGraph<AgentState>` do LangGraph.js 0.2.x+
- [x] Declarar exatamente 11 nós na ordem: `START → GUARDRAIL → CACHE → INTENT → RAG → TOOL_SELECT → TOOL_EXEC → GENERATE → STREAM → PERSIST → LANGFUSE → END`
- [x] Definir arestas condicionais:
  - `GUARDRAIL` → `CACHE` (pass) ou `END` (bloqueado — ex: conteúdo proibido)
  - `CACHE` → `STREAM` (cache hit) ou `INTENT` (cache miss)
  - `TOOL_SELECT` → `TOOL_EXEC` (tools necessários) ou `GENERATE` (sem tools)
  - `GENERATE` → `STREAM`
  - Todos os nós → `END` em caso de `state.error.recoverable === false`
- [x] Compilar o grafo: `const agent = graph.compile()` — sem erros em TypeScript strict
- [x] Teste unitário: verificar que `agent.nodes` contém os 11 nós; verificar que arestas condicionais existem; mock de cada nó → confirmar fluxo correto para cada caminho

### 3. Nó `GUARDRAIL` — validação de entrada

- [x] Implementar nó `guardrailNode(state: AgentState): Partial<AgentState>`:
  - Verificar que `messages` tem pelo menos 1 mensagem com `role === 'user'`
  - Verificar comprimento máximo da última mensagem: ≤ 4096 caracteres → se maior, truncar e logar aviso
  - Verificar que `metadata.userId` e `metadata.conversationId` estão presentes → se ausente, retornar `{ error: { code: 'MISSING_CONTEXT', recoverable: false } }`
  - Verificar conteúdo com lista de termos proibidos (configurável via `agent_configurations`) → se detectado, retornar `{ error: { code: 'CONTENT_POLICY', recoverable: false } }`
  - **Contador de insistência em dados bloqueados — [CORRIGIDO: FINDING-005]:** `INCR ai:blocked:insistence:{conversationId}` com TTL 3600s. Se `contador === 1`: adicionar ao state `insistenceLevel: 1` (agente responde com recusa + alternativa via nó GENERATE). Se `contador === 2`: `insistenceLevel: 2` (recusa + "Posso ajudá-lo com análises dentro do seu perfil. Veja algumas opções:"). Se `contador >= 3`: `insistenceLevel: 3` (recusa sem alternativas — sem loop). Estado reseta ao criar nova conversa. Verificar contador apenas para mensagens com intent `'dados_bloqueados'` (detectado pelo nó GUARDRAIL via matching de padrões de solicitação de dados proibidos).
- [x] Logar entrada do guardrail via Pino (sem PII — apenas `userId`, `conversationId`, comprimento da mensagem)
- [x] Teste unitário: mensagem válida → pass; mensagem > 4096 chars → truncar; `userId` ausente → `error.recoverable = false`; termo proibido → `error.code = 'CONTENT_POLICY'`

### 4. Nó `CACHE` — verificação exact + semantic

- [x] Implementar nó `cacheNode(state: AgentState): Partial<AgentState>`:
  - **Cache exact:** gerar hash SHA-256 da última mensagem do usuário → buscar em `llm_cache_entries` por `cache_key = 'ai:cache:exact:{hash}'`; se encontrado E `expires_at > NOW()` → retornar `{ cacheHit: true, cacheType: 'exact', response: entry.response }`
  - **Cache semantic (se exact miss):** gerar embedding da mensagem via `text-embedding-3-small` → buscar em `llm_cache_entries` via `<=>` (cosine) com threshold `0.92` — usar valor de `agent_configurations.semantic_cache_threshold`; se encontrado → retornar `{ cacheHit: true, cacheType: 'semantic', response: entry.response }`
  - Cache miss → retornar `{ cacheHit: false, cacheType: null }`
- [x] TTL exact cache: 24 horas (`expires_at = NOW() + INTERVAL '24 hours'`) — ao gravar cache novo
- [x] TTL semantic cache: 1 hora (`expires_at = NOW() + INTERVAL '1 hour'`) — ao gravar cache novo
- [x] Incrementar `hit_count` no registro ao usar cache (fire-and-forget — não bloquear fluxo)
- [x] Teste unitário: mock Redis com key `ai:cache:exact:{hash}` existente → retornar cache hit; threshold 0.92 → mensagem similar (score 0.95) → hit; mensagem similar (score 0.88) → miss; `expires_at` no passado → miss (não usar cache expirado)

### 5. Nó `INTENT` — classificação de intenção

- [x] Implementar nó `intentNode(state: AgentState): Partial<AgentState>`:
  - Chamar GPT-4o com temperature 0 e system prompt de classificação
  - Classificar em 6 intenções: `analise_oportunidade`, `comparacao_oportunidades`, `simulacao_proposta`, `simulacao_portfolio`, `suporte_operacional`, `fora_do_escopo`
  - Usar `Structured Outputs` via zod schema: `z.object({ intent: z.enum([...]), confidence: z.number().min(0).max(1) })`
  - Intenção `fora_do_escopo` → retornar `{ response: "Sou especializado em análise de oportunidades de crédito. Posso ajudá-lo com análise, comparação, simulação ou status de operações." }` e encerrar o fluxo
- [x] Logar intenção classificada + confidence via Langfuse (trace)
- [x] Teste unitário: mock GPT → retornar `{ intent: 'analise_oportunidade', confidence: 0.95 }`; intenção `fora_do_escopo` → retornar resposta padronizada sem chamar RAG; schema inválido → retornar `error.code = 'INTENT_PARSE_ERROR'`

### 6. Nó `RAG` — recuperação semântica com pgvector

- [x] Implementar nó `ragNode(state: AgentState): Partial<AgentState>`:
  - Gerar embedding da última mensagem via `text-embedding-3-small` (1536 dims)
  - Buscar em `document_embeddings` via `SELECT ... ORDER BY embedding <=> $1 LIMIT 5` com threshold `similarity_threshold = 0.78` (via `agent_configurations`)
  - Retornar top-5 documentos por similaridade coseno acima do threshold
  - Chunking: documentos divididos em chunks de 512-1024 tokens (parâmetros respeitados no momento da indexação — verificar que `metadata.chunk_index` está presente)
  - Se 0 documentos acima do threshold → retornar `{ context: [] }` (sem erro — RAG pode não retornar nada)
- [x] Não aplicar filtro por `cessionario_id` em `document_embeddings` — isolamento feito pelo system prompt (Camada 3)
- [x] Teste unitário: mock pgvector com 5 docs acima de 0.78 → retornar 5; mock com 2 acima e 3 abaixo → retornar 2; mock com todos abaixo → retornar `[]`; embedding falha → `error.code = 'RAG_EMBEDDING_ERROR'`

---

## 🏗️ FEATURE 2: Tools — 5 Ferramentas com Timeouts e Retries

### 7. Tool `calculate_delta` — cálculo de Δ (deságio)

- [x] Criar `CalculateDeltaTool` implementando interface `Tool` do LangChain.js:
  - `name: 'calculate_delta'`
  - `description: 'Calcula o deságio (Δ) de uma oportunidade de crédito: diferença percentual entre valor nominal e valor de mercado'`
  - Input schema (zod): `{ nominal_value: z.number().positive(), market_value: z.number().positive() }`
  - Lógica: `delta = ((nominal_value - market_value) / nominal_value) * 100` — resultado em percentual com 2 casas decimais
  - Timeout: 5s; sem retry (operação determinística — falha é erro de input)
  - Retornar `{ delta: number, nominal_value: number, market_value: number, calculation: string }` com `calculation` sendo a expressão completa
- [x] Validar que `market_value > 0` e `nominal_value > 0` → input inválido lança `ToolInputError`
- [x] Teste unitário: `nominal=100, market=80` → `delta=20.00`; `nominal=100, market=100` → `delta=0.00`; `market_value=0` → `ToolInputError`; `market_value > nominal_value` → delta negativo (válido, reflete ágio)

### 8. Tool `calculate_roi` — ROI e cenários

- [x] Criar `CalculateRoiTool`:
  - Input schema: `{ investment: z.number().positive(), expected_return: z.number(), period_months: z.number().int().positive() }`
  - Lógica ROI: `roi = ((expected_return - investment) / investment) * 100`
  - Cenários obrigatórios (±20% conforme RN-019): `{ pessimistic: roi * 0.8, base: roi, optimistic: roi * 1.2 }`
  - Calcular também `annualized_roi`: `((1 + roi/100)^(12/period_months) - 1) * 100`
  - Timeout: 5s; sem retry
  - Retornar `{ roi, annualized_roi, scenarios: { pessimistic, base, optimistic }, period_months }`
- [x] Teste unitário: `investment=80, expected_return=100, period_months=12` → `roi=25.00`; verificar os 3 cenários ±20%; `period_months=0` → `ToolInputError`

### 9. Tool `calculate_portfolio` — simulação de carteira

- [x] Criar `CalculatePortfolioTool`:
  - Input schema: `{ opportunities: z.array(z.object({ id: z.string(), nominal_value: z.number().positive(), market_value: z.number().positive(), expected_return: z.number(), period_months: z.number().int().positive() })).min(1).max(10) }`
  - Calcular por oportunidade: `delta`, `roi`, `roi_annualized`
  - Calcular totais do portfólio: `total_investment`, `total_nominal`, `weighted_roi` (ROI ponderado por `market_value`), `weighted_delta`
  - Timeout: 10s; retry 2x (pode envolver múltiplas operações)
  - Retornar `{ opportunities: [...], portfolio_summary: { total_investment, total_nominal, weighted_roi, weighted_delta, count } }`
- [x] Máximo 10 oportunidades — input com 11+ → `ToolInputError`
- [x] Teste unitário: 1 oportunidade → totals corretos; 5 oportunidades → weighted ROI correto; 11 oportunidades → `ToolInputError`

### 10. Tool `get_market_data` — dados de mercado

- [x] Criar `GetMarketDataTool`:
  - Input schema: `{ segment: z.string().optional(), asset_type: z.string().optional(), reference_date: z.string().date().optional() }`
  - Buscar dados de mercado do banco (tabela ou source externo conforme configuração)
  - Timeout: 15s; retry 3x com backoff exponencial (500ms, 1000ms, 2000ms)
  - Retornar `{ market_data: MarketDataEntry[], source: string, retrieved_at: string }`
  - Fallback se market data indisponível: retornar `{ market_data: [], source: 'unavailable', retrieved_at: string }` sem lançar erro (fluxo continua)
- [x] Teste unitário: mock DB com dados → retornar dados; DB timeout após 15s → retry 3x → fallback `market_data: []`; DB disponível na 2ª tentativa → sucesso sem falha

### 11. Tool `get_opportunity_data` — dados de oportunidade específica

- [x] Criar `GetOpportunityDataTool`:
  - Input schema: `{ opportunity_id: z.union([z.string().uuid(), z.string().regex(/^OPR-[A-Z0-9]{4}-[A-Z0-9]{4}$/)]) }` — aceitar UUID ou código OPR-XXXX-XXXX — [CORRIGIDO: FINDING-028]
  - Buscar dados da oportunidade na plataforma (via API interna ou banco)
  - Aplicar **isolamento de dados**: verificar que a oportunidade pertence ao `cessionario_id` do state — injetar `userId` do `AgentState.metadata` na query
  - Timeout: 10s; retry 2x
  - Retornar `{ opportunity: OpportunityData }` com campos: `id, nominal_value, market_value, segment, status, cedente_id, created_at`
  - Oportunidade não encontrada ou sem permissão → retornar `{ opportunity: null }` (não lançar erro — o agente decide o que responder)
- [x] Teste unitário: `opportunity_id` válido pertencente ao usuário → retornar dados; `opportunity_id` de outro usuário → retornar `{ opportunity: null }`; UUID inválido → `ToolInputError`

### 12. Nó `TOOL_SELECT` — seleção de tools

- [x] Implementar nó `toolSelectNode(state: AgentState): Partial<AgentState>`:
  - Usar `parallel_tool_calls: false` na chamada GPT-4o (conforme Doc 19 — tools executados em sequência)
  - GPT-4o decide quais tools chamar com base em `state.intent` + últimas mensagens
  - Retornar lista de `ToolCall[]` a serem executados pelo próximo nó
  - Máximo de 3 tools por turn (evitar loops de tools)
  - Se intent é `suporte_operacional` → não chamar tools de cálculo; chamar apenas `get_opportunity_data`
- [x] Teste unitário: intent `analise_oportunidade` → seleciona `calculate_delta` + `calculate_roi` + `get_opportunity_data`; intent `suporte_operacional` → seleciona apenas `get_opportunity_data`; intent `fora_do_escopo` → `toolResults: []`

### 13. Nó `TOOL_EXEC` — execução sequencial

- [x] Implementar nó `toolExecNode(state: AgentState): Partial<AgentState>`:
  - Executar cada tool da lista `ToolCall[]` em sequência (não paralelo — `parallel_tool_calls: false`)
  - Para cada tool: `try { result = await tool.invoke(input) } catch (e) { result = { error: e.message } }`
  - Erros em tools individuais NÃO abortam o fluxo — registrar no `toolResult.error` e continuar
  - Após todos os tools: acumular em `state.toolResults`
- [x] Timeout global por tool: conforme declarado em cada tool (5s, 10s, 15s)
- [x] Teste unitário: 2 tools, primeiro falha → segundo ainda executa; todos os tools timeout → `toolResults` com `error` em todos, fluxo continua para GENERATE; tool retorna resultado válido → acumulado corretamente

---

## ✅ TESTES — Agente Core Parte 1

### 14. Suite de testes LangGraph + Tools

- [x] Criar `test/unit/ai/agent/agent-state.spec.ts` — verificar imutabilidade e estrutura dos 9 campos
- [x] Criar `test/unit/ai/agent/guardrail.node.spec.ts` — cobrir todos os paths do nó GUARDRAIL
- [x] Criar `test/unit/ai/agent/cache.node.spec.ts` — cobrir exact cache, semantic cache (threshold 0.92), TTL (24h exact, 1h semantic)
- [x] Criar `test/unit/ai/agent/intent.node.spec.ts` — cobrir 6 intenções + Structured Outputs
- [x] Criar `test/unit/ai/agent/rag.node.spec.ts` — cobrir threshold 0.78, chunking 512-1024, 0 resultados
- [x] Criar `test/unit/ai/tools/calculate-delta.tool.spec.ts` — cobrir lógica Δ + edge cases
- [x] Criar `test/unit/ai/tools/calculate-roi.tool.spec.ts` — cobrir ROI + 3 cenários ±20%
- [x] Criar `test/unit/ai/tools/calculate-portfolio.tool.spec.ts` — cobrir max 10 oportunidades
- [x] Criar `test/unit/ai/tools/get-market-data.tool.spec.ts` — cobrir retry 3x + fallback
- [x] Criar `test/unit/ai/tools/get-opportunity-data.tool.spec.ts` — cobrir isolamento de dados
- [x] Criar `test/unit/ai/tools/tool-exec.node.spec.ts` — cobrir falha parcial + sequência
- [x] Cobertura mínima módulo `ai/`: 85% (conforme Doc 27)

---

## 🔀 Cross-Módulo

- [x] **→ S4 (Calculadora):** `calculate_delta`, `calculate_roi`, `calculate_portfolio` serão reutilizados como fallback determinístico pela `CalculadoraModule` — garantir que os tools exportem lógica pura separada da integração LangChain para reuso
- [x] **→ S3b (Agente Core Parte 2):** nós `GENERATE`, `STREAM`, `PERSIST`, `LANGFUSE` dependem do grafo compilado nesta sprint — confirmar que `AgentGraph` é exportado como provider NestJS

---

## ✔️ Auto-Verificação S3a (12 Checks)

| #   | Check                                                                                                                                                                                                                                                                       | Status |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                                                                                                                | ✅     |
| 2   | Nomes exatos: `AgentState`, `calculate_delta`, `calculate_roi`, `calculate_portfolio`, `get_market_data`, `get_opportunity_data`, `llm_cache_entries`, `document_embeddings`, `text-embedding-3-small`                                                                      | ✅     |
| 3   | Valores numéricos: 11 nós, 9 campos AgentState, 5 tools, threshold RAG 0.78, threshold semantic cache 0.92, TTL exact 24h, TTL semantic 1h, 1536 dims, chunking 512-1024 tokens, ±20% cenários, max 10 oportunidades portfolio, max 3 tools/turn, parallel_tool_calls=false | ✅     |
| 4   | Glossário consultado — Δ (deságio), ROI, Score de Risco, Calculadora de Comissão, RAG                                                                                                                                                                                       | ✅     |
| 5   | Anti-Scaffold R10: cada tool tem lógica real (não placeholder); cada nó tem edge cases testados                                                                                                                                                                             | ✅     |
| 6   | Fluxo do grafo: todas as arestas condicionais documentadas (GUARDRAIL→END, CACHE→STREAM, TOOL_SELECT→GENERATE)                                                                                                                                                              | ✅     |
| 7   | Timeouts por tool: calculate_delta=5s, calculate_roi=5s, calculate_portfolio=10s, get_market_data=15s, get_opportunity_data=10s                                                                                                                                             | ✅     |
| 8   | Sem conflitos não marcados                                                                                                                                                                                                                                                  | ✅     |
| 9   | Isolamento de dados em `get_opportunity_data`: `userId` do AgentState.metadata injetado na query                                                                                                                                                                            | ✅     |
| 10  | Template B aplicado: organizado por FEATURE (Feature 1: LangGraph, Feature 2: Tools)                                                                                                                                                                                        | ✅     |
| 11  | `parallel_tool_calls: false` declarado explicitamente                                                                                                                                                                                                                       | ✅     |
| 12  | REQs cobertos: REQ-008 a REQ-013, REQ-024 a REQ-027, REQ-029 a REQ-032, REQ-036, REQ-062 a REQ-067, REQ-073, REQ-075, REQ-076 — todos com ≥1 item                                                                                                                           | ✅     |
