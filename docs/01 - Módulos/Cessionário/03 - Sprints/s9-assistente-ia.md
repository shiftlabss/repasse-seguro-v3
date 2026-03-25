# S9 — Assistente de IA (Analista de Oportunidades)

## Sprint 9 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S9 — Assistente de IA                                                                                            |
| **Template**       | B — Módulo Fullstack (feature com vertical slice Banco→Backend→Frontend→Wiring→Testes)                           |
| **REQs cobertos**  | S9-001 a S9-018 (18 requisitos do Registro Mestre)                                                               |
| **Docs fonte**     | 19 - Criação de Agentes de IA · 01.3 - RN Operação · 06 - Mapa de Telas · 16 - Documentação de API · 02 - Stacks |
| **Total de itens** | 52 itens                                                                                                         |
| **Status**         | Concluída                                                                                                        |

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos: `ai_sessions`, `AiSession`, modelos `gpt-4-turbo-2024-04-09`, `gpt-4o-mini-2024-07-18`, `text-embedding-3-small`, tools: `getOpportunity`, `listOpportunities`, `getCessionarioPortfolio`, `calculateCommission`, `semanticSearch`.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — SLA 5s individual/10s comparativo; retry 30s; temperature 0.1; chunking 512 tokens/overlap 64; Redis TTL 30min; rate limit 20 req/min.
- [x] ✅ Check 4 — Glossário: RAG, pgvector, HNSW, SSE, Structured Output, Langfuse, `InputSanitizerService`, `OutputFilterService`.
- [x] ✅ Check 5 — Anti-scaffold R10: 5 tools implementadas com lógica real, guardrails reais, streaming SSE real.
- [x] ✅ Check 6 — Estado da sessão IA: criada/ativa/encerrada (TTL Redis); sem persistência indefinida.
- [x] ✅ Check 7 — Redis session TTL 30min; embeddings indexados; chunking 512/64.
- [x] ✅ Check 8 — Cross-módulo: `getCessionarioPortfolio` acessa proposals/negotiations; `calculateCommission` usa mesma lógica de S4.
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — ai_consent obrigatório verificado antes de qualquer uso.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S9-001 a S9-018 cobertos.

---

## FEATURE — Analista de Oportunidades (T-IA-01)

### 🗄️ Banco

- [x] **S9-B01** · Confirmar `ai_sessions` table: `id UUID PK`, `cessionario_id UUID FK → cessionarios.id CASCADE`, `messages JSONB DEFAULT '[]'`, `context JSONB`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`. Índice `idx_ai_sessions_cessionario_id`. Confirmar que `opportunities.embedding vector(1536)` e índice HNSW `idx_opportunities_embedding` (m=16, ef_construction=64) criados em S1. (Doc 13 — model AiSession; Doc 13 — seção 2.2)

- [x] **S9-B02** · Criar migration SQL para função de busca semântica em PostgreSQL: `CREATE OR REPLACE FUNCTION match_opportunities(query_embedding vector(1536), match_threshold float, match_count int) RETURNS TABLE (id UUID, code VARCHAR, similarity float) AS $$ SELECT id, code, 1 - (embedding <=> query_embedding) AS similarity FROM opportunities WHERE deleted_at IS NULL AND status IN ('DISPONIVEL', 'COM_PROPOSTA') AND 1 - (embedding <=> query_embedding) > match_threshold ORDER BY similarity DESC LIMIT match_count; $$ LANGUAGE sql;`. Verificar que a função é acessível via `prisma.$queryRaw`. (Doc 19 — RAG; Doc 14 — pgvector)

### ⚙️ Backend

- [x] **S9-BE01** · Implementar `AiModule` em `apps/api/src/modules/ai/`: instalar dependências `langchain@0.3+`, `@langchain/openai`, `@langchain/core`, `ai` (Vercel AI SDK 4+), `langfuse@3+`; configurar `OpenAI` client com `OPENAI_API_KEY`; configurar Langfuse com `LANGFUSE_SECRET_KEY` + `LANGFUSE_PUBLIC_KEY` + `LANGFUSE_HOST`; todos os traces LLM rastreados no Langfuse com `traceId = correlation_id` da request. (Doc 02 — Stacks IA; Doc 17 — Langfuse)

- [x] **S9-BE02** · Implementar `InputSanitizerService` em `apps/api/src/modules/ai/services/input-sanitizer.service.ts`: sanitizar inputs do usuário antes de enviar ao LLM; remover possíveis tentativas de prompt injection (padrões: "ignore previous instructions", "system:", "assistant:", repetição anormal de tokens); validar tamanho máximo de mensagem (2000 chars); remover PII detectável (CPF pattern `\d{3}\.\d{3}\.\d{3}-\d{2}`, e-mail patterns, telefone patterns); retornar input sanitizado ou lançar `VAL-001` se malicioso. (Doc 19 — InputSanitizerService)

- [x] **S9-BE03** · Implementar `OutputFilterService` em `apps/api/src/modules/ai/services/output-filter.service.ts`: filtrar outputs do LLM antes de retornar ao usuário; verificar que nenhuma informação de identidade do Cedente foi incluída (nome, CPF, contato); verificar que valores financeiros mencionados são consistentes com dados reais (não alucinação); remover instruções de sistema que possam ter "vazado" no output; se output suspeito: substituir com mensagem de fallback "Não posso responder a esta pergunta. Por favor, reformule." e logar `warn` no Pino. (Doc 19 — OutputFilterService)

- [x] **S9-BE04** · Implementar as 5 tools do Analista de Oportunidades em `apps/api/src/modules/ai/tools/`: (1) `getOpportunity(opportunity_id: string)` — busca oportunidade por ID via `OpportunitiesService.findById()`, sanitizando dados do Cedente; (2) `listOpportunities(filters: {...})` — lista oportunidades com filtros, máx 10 por chamada; (3) `getCessionarioPortfolio(cessionario_id: string)` — busca propostas e negociações ativas/históricas do cessionário; (4) `calculateCommission(current_table_value, contract_table_value, cedente_paid_value)` — usa exatamente a mesma lógica de `lib/commission.ts` em centavos; (5) `semanticSearch(query: string, threshold: float = 0.7, limit: int = 5)` — gera embedding via `text-embedding-3-small`, executa função `match_opportunities` via `prisma.$queryRaw`. (Doc 19 — seção tools)

- [x] **S9-BE05** · Implementar `RagPipelineService` em `apps/api/src/modules/ai/services/rag-pipeline.service.ts`: method `indexOpportunity(opportunity_id: string)` — buscar texto representativo da oportunidade (city, neighborhood, developmentName, delta_value, bedrooms); chunking: 512 tokens com overlap 64 usando `RecursiveCharacterTextSplitter`; gerar embedding via `text-embedding-3-small`; armazenar em `opportunities.embedding` via `prisma.$executeRaw`; job para re-indexar oportunidades modificadas; method `queryEmbedding(text: string): number[]` — gerar embedding para query do usuário. (Doc 19 — RAG; chunking 512/64)

- [x] **S9-BE06** · Implementar `AiSessionService` em `apps/api/src/modules/ai/services/ai-session.service.ts`: criar/buscar sessão ativa do cessionário (`ai_sessions` record); histórico de mensagens armazenado em `messages JSONB` (formato `[{ role, content, timestamp }]`); memória de contexto no Redis `rs:ai:sessao:{cessionario_id}` TTL 30min (mais rápido que banco para messages ativas); ao encerrar sessão (TTL expirar), persistir `messages` no banco; máximo de 50 mensagens por sessão (circular buffer — remover mais antigas). (Doc 19 — Redis session; Doc 14 — TTL 30min)

- [x] **S9-BE07** · Implementar endpoint `POST /api/v1/ai/chat` (streaming SSE) em `apps/api/src/modules/ai/`: guards `JwtAuthGuard` + `KycGuard`; verificar `cessionarios.ai_consent = true` antes de processar (`PERM-001` se false); rate limit 20 req/min; body `{ message: string, opportunity_id?: string, session_id?: string }`; pipeline: (1) `InputSanitizerService.sanitize(message)`; (2) construir prompt com contexto do cessionário + oportunidade se fornecida; (3) chamar modelo `gpt-4-turbo-2024-04-09` (temperature: 0.1, structured output) com tools disponíveis; (4) streaming via Vercel AI SDK usando Server-Sent Events; (5) `OutputFilterService.filter(response)`; (6) salvar mensagem em `AiSessionService`; (7) rastrear trace no Langfuse; resposta via SSE com eventos `data:`, `error:`, `done:`. SLA: 5s para resposta inicial (retry button disponível após 30s se sem resposta). (Doc 19 — modelo, SLA; Doc 16 — POST /ai/chat; Doc 01.3 — RN-044, RN-045)

- [x] **S9-BE08** · Implementar endpoint `POST /api/v1/ai/compare` (análise comparativa) em `apps/api/src/modules/ai/`: guards `JwtAuthGuard` + `KycGuard` + ai_consent; body `{ opportunity_ids: string[] }` (máx 5 oportunidades); usar `gpt-4o-mini-2024-07-18` para análise comparativa (mais econômico para comparações); SLA: 10s; retornar análise estruturada JSON com ranking e justificativas; streaming SSE; rastrear no Langfuse. (Doc 19 — gpt-4o-mini; Doc 01.3 — RN-044)

- [x] **S9-BE09** · Implementar `GET /api/v1/ai/sessions` e `GET /api/v1/ai/sessions/:id` em `apps/api/src/modules/ai/`: listar histórico de sessões do cessionário com resumo (n° de mensagens, created_at, updated_at); buscar sessão específica com histórico completo de mensagens; paginação; verificar ai_consent. (Doc 16 — GET /ai/sessions, GET /ai/sessions/:id)

### 🎨 Frontend

- [x] **S9-FE01** · Implementar tela `T-IA-01 — Analista de IA` em `apps/web/src/features/ai/pages/AiAssistantPage.tsx` (rota `/analista`, modal full-screen): layout de chat com histórico; input de mensagem (max 2000 chars); botão enviar; se `ai_consent = false`, exibir tela de consentimento com botão "Ativar Analista de IA" → `PATCH /api/v1/profile/ai-consent`; conectar ao endpoint `POST /api/v1/ai/chat` via SSE (`EventSource` ou `fetch` com `stream: true`); renderizar resposta progressivamente conforme chunks chegam; spinner de "digitando..." durante geração; botão "Analisar oportunidade" pré-preenchido se `?opportunity_id=` na URL; FAB (Floating Action Button) no Dashboard abre esta tela. (Doc 06 — T-IA-01; Doc 01.3 — RN-044)

- [x] **S9-FE02** · Implementar streaming SSE no frontend em `apps/web/src/features/ai/hooks/useAiStream.ts`: usar `fetch` com `ReadableStream` para consumir SSE; parser de eventos `data:`, `error:`, `done:`; ao receber `error:`, exibir mensagem de erro inline; ao receber `done:`, finalizar streaming e habilitar input; botão "Tentar novamente" (retry) aparece se 30s sem resposta inicial — chama o mesmo endpoint novamente (sem duplicar mensagem no histórico); animação de cursor piscando durante streaming. (Doc 01.3 — RN-045; Doc 19 — SLA 5s/retry 30s)

- [x] **S9-FE03** · Implementar componentes de contexto IA no T-IA-01: (1) `OpportunityContext` — card compacto da oportunidade em análise (código, delta_value, score de risco) exibido no topo do chat; (2) `ToolCallIndicator` — exibir "Consultando oportunidades..." ou "Calculando comissão..." enquanto tool está sendo executada; (3) `AiDisclaimer` — texto fixo "As análises do Assistente são orientativas e não constituem recomendação de investimento."; (4) botão "Comparar oportunidades" — abre modal para selecionar até 5 oportunidades → `POST /api/v1/ai/compare`. (Doc 01.3 — RN-044)

### 🔌 Wiring

- [x] **S9-W01** · Configurar Langfuse no backend para rastreabilidade completa do Analista: cada request ao `POST /api/v1/ai/chat` cria uma trace com: `traceId = correlation_id`, `userId = hash(cessionario_id)`, `input = message sanitizado`, `output = response filtrada`, `metadata: { opportunity_id, session_id, model, temperature, tools_used: [], latency_ms, tokens: { prompt, completion, total } }`; configurar alertas Langfuse: p95 latência > 5s → Slack `#ops`; custo diário > threshold → Slack `#ops`. (Doc 17 — Langfuse; Doc 25 — alertas)

- [x] **S9-W02** · Implementar job de indexação RAG em `apps/api/src/modules/ai/jobs/rag-index.job.ts`: cron `0 2 * * *` (2h diariamente); buscar oportunidades com `embedding IS NULL` ou `updated_at > last_indexed_at`; para cada: chamar `RagPipelineService.indexOpportunity()`; processar em batches de 10; publicar na fila `ai.rag-index` para processamento assíncrono pelo `RagIndexWorker`; log de progresso no Pino. (Doc 19 — RAG pipeline; Doc 14 — fila ai.rag-index)

- [x] **S9-W03** · Configurar proxy CORS para SSE no frontend: `EventSource` tem limitações de CORS e autenticação; usar `fetch` com `ReadableStream` para SSE autenticado (JWT no header `Authorization: Bearer <token>`); garantir que servidor responde com `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`. (Doc 14 — SSE; Doc 19)

### 🧪 Testes

- [x] **S9-T01** · Testes unitários `InputSanitizerService`: prompt injection "ignore previous instructions" → sanitizado; CPF `123.456.789-09` → removido; mensagem limpa → passa intacta; mensagem > 2000 chars → truncada ou `VAL-001`. Cobertura 100%.

- [x] **S9-T02** · Testes unitários `OutputFilterService`: output com nome fictício de Cedente → substituído por fallback; output com CPF pattern → substituído; output limpo → passa intacto. Cobertura 100%.

- [x] **S9-T03** · Testes unitários das 5 tools: `calculateCommission` usa exatamente mesma fórmula de `lib/commission.ts`; `semanticSearch` chama `prisma.$queryRaw` com função `match_opportunities`; `getCessionarioPortfolio` não expõe dados do Cedente; `getOpportunity` chama `sanitizeOpportunity`. Cobertura 100%.

- [x] **S9-T04** · Testes unitários `AiSessionService`: mensagem 51 → remove mensagem mais antiga (circular buffer, max 50); TTL Redis 30min configurado; persistência no banco ao expirar sessão. Mock Redis e Prisma.

- [x] **S9-T05** · Testes E2E Playwright — T-IA-01: login com ai_consent=true, abrir Analista, enviar mensagem "Analise a oportunidade OPR-XXXX", verificar que resposta é gerada progressivamente via streaming, verificar `ToolCallIndicator` aparece durante tool execution, verificar que nenhum dado de Cedente aparece na resposta.

---

## 🔀 Cross-Módulo

- [x] **S9-CM01** · **[← S4]** `calculateCommission` tool usa exatamente a mesma lógica de `lib/commission.ts` (S4/S1). Importar de `packages/shared-types` ou garantir que `ProposalsService.calculateCommission()` é acessível via DI. Nunca duplicar a lógica.

- [x] **S9-CM02** · **[← S3]** Verificar `ai_consent = true` em `cessionarios` antes de qualquer operação IA. Se `ai_consent` mudado para false (PATCH /profile/ai-consent em S3), invalidar sessão ativa Redis `rs:ai:sessao:{cessionario_id}` imediatamente.

---

## 📊 COBERTURA DE REQs S9

| REQ ID | Descrição                              | Item(s)          |
| ------ | -------------------------------------- | ---------------- |
| S9-001 | AiModule com LangChain + Vercel AI SDK | S9-BE01          |
| S9-002 | InputSanitizerService                  | S9-BE02          |
| S9-003 | OutputFilterService                    | S9-BE03          |
| S9-004 | 5 tools implementadas                  | S9-BE04          |
| S9-005 | RAG chunking 512/64 + pgvector         | S9-BE05          |
| S9-006 | AiSessionService Redis TTL 30min       | S9-BE06          |
| S9-007 | POST /ai/chat streaming SSE            | S9-BE07          |
| S9-008 | Modelo gpt-4-turbo-2024-04-09          | S9-BE07          |
| S9-009 | temperature 0.1 structured output      | S9-BE07          |
| S9-010 | POST /ai/compare gpt-4o-mini           | S9-BE08          |
| S9-011 | GET /ai/sessions                       | S9-BE09          |
| S9-012 | SLA 5s/10s, retry 30s                  | S9-BE07, S9-FE02 |
| S9-013 | Rate limit 20 req/min                  | S9-BE07          |
| S9-014 | T-IA-01 chat modal                     | S9-FE01          |
| S9-015 | SSE streaming progressivo              | S9-FE02          |
| S9-016 | Langfuse observabilidade               | S9-W01           |
| S9-017 | Job RAG indexação 2h cron              | S9-W02           |
| S9-018 | ai_consent verificado + PERM-001       | S9-BE07, S9-CM02 |
