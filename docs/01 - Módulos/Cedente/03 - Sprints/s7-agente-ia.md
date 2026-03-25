# S7 — Agente IA (Guardião do Retorno)

## Módulo Cedente · Plataforma Repasse Seguro

| **Sprint** | **Tipo**         | **Template** | **Versão** | **Data**   |
| ---------- | ---------------- | ------------ | ---------- | ---------- |
| S7         | Condicional (IA) | Template B   | v1.0       | 2026-03-24 |

**REQs cobertos:** REQ-097–103, REQ-150 (SSE), REQ-186–193

**Dependências:** S1 (pgvector extension, Langfuse, OpenAI configurados), S3 (casos, comissao.calculator), S4 (documentos), S5 (escalonamento), S6 (financeiro)

**Critério de aceite da sprint:** todos os itens ✅ + 12 auto-verificações ✅ + T-035, T-036, T-037 passando + Guardião <20% taxa de escalação em staging.

---

## 🔴 FEATURE 1 — Banco de Dados e Infraestrutura IA

### 1. Banco de Dados — Guardião

- [x] **1.1** Confirmar que a extensão `pgvector` está habilitada no Supabase: `CREATE EXTENSION IF NOT EXISTS vector;`.
- [x] **1.2** Criar tabela `conversas_guardiao` conforme schema de D19 seção 5.3:
  - `id UUID PK DEFAULT gen_random_uuid()`, `cedente_auth_id UUID NOT NULL REFERENCES auth.users(id)`, `caso_id UUID REFERENCES casos(id)`.
  - `role VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant', 'tool'))`.
  - `content TEXT` (anonimizado após 90 dias), `content_hash VARCHAR(64)` (SHA-256 preservado após anonimização).
  - `model_version VARCHAR(50)` (ex: `gpt-4-turbo-2024-04-09`), `tokens_input INTEGER`, `tokens_output INTEGER`, `latency_ms INTEGER`, `confidence_score DECIMAL(3,2)`.
  - `trace_id UUID`, `escalado_para_humano BOOLEAN DEFAULT FALSE`.
  - `created_at TIMESTAMPTZ DEFAULT NOW()`, `anonimizado_em TIMESTAMPTZ`.
- [x] **1.3** Criar índice `idx_conversas_cedente ON conversas_guardiao(cedente_auth_id, created_at DESC)`.
- [x] **1.4** Criar índice `idx_conversas_caso ON conversas_guardiao(caso_id)`.
- [x] **1.5** Aplicar RLS em `conversas_guardiao`: `FOR SELECT USING (auth.uid() = cedente_auth_id)` — Cedente vê apenas suas conversas; histórico imutável (sem DELETE policy para Cedente).
- [x] **1.6** Criar tabela `guardiao_knowledge_base` conforme schema de D19 seção 6.3:
  - `id UUID PK`, `categoria VARCHAR(50) NOT NULL`, `titulo VARCHAR(200) NOT NULL`, `conteudo TEXT NOT NULL`, `conteudo_embedding VECTOR(1536)`, `metadata JSONB`, `ativo BOOLEAN DEFAULT TRUE`, `created_at`, `updated_at`.
- [x] **1.7** Criar índice HNSW: `CREATE INDEX idx_knowledge_embedding ON guardiao_knowledge_base USING hnsw (conteudo_embedding vector_cosine_ops) WITH (m = 16, ef_construction = 200)` (D19 seção 6.2).
- [x] **1.8** Criar função `match_knowledge(query_embedding VECTOR, match_threshold FLOAT, match_count INT)` que retorna os `match_count` registros com `1 - (conteudo_embedding <=> query_embedding) >= match_threshold`.
- [x] **1.9** Criar migration `0007_guardiao_ia.sql` com tabelas, índices e função acima.
- [x] **1.10** Redis — memória de sessão: chave `rs:cedente:guardiao:session:{cedente_id}` com TTL **4 horas** (D19 seção 5.2). Máximo de **20 mensagens** ou **8.000 tokens** (o que vier primeiro).

### 2. Backend — Módulo AI (Estrutura de Pastas)

- [x] **2.1** Criar pasta `apps/api/src/modules/ai/guardiao/` com os arquivos:
  - `guardiao.module.ts`
  - `guardiao.controller.ts` — `POST /ai/guardiao/message` (SSE streaming via ADR-CED-002)
  - `guardiao.service.ts` — orquestração LangChain.js
  - `guardiao.guard.ts` — auth + rate limit
  - `tools/index.ts` — 6 tools declaradas
  - `rag/config.ts` — `RAG_CONFIG` com valores exatos de D19
  - `rag/retriever.ts` — `SupabaseVectorStoreRetriever`
  - `filters/output-filter.ts` — `GuardiaoOutputFilter`
  - `filters/confidence-scorer.ts` — `calculateConfidence()`
  - `memory/session-manager.ts` — Redis + DB histórico
  - `langraph/cadastro-assistido.ts` — `LangGraph.js` para wizard

---

## 🔴 FEATURE 2 — Guardião: Core do Agente

### 3. Backend — LLM e Tool Calls

- [x] **3.1** Configurar modelo OpenAI no `GuardiaoService`: `model: 'gpt-4-turbo-2024-04-09'` — versão **fixada** (não `gpt-4-turbo-preview` ou `gpt-4-turbo`). NUNCA usar alias flutuante (D19 TL;DR).
- [x] **3.2** Configurar parâmetros do LLM: `temperature: 0.3`, `max_tokens: 1000`, `stream: true` (SSE — ADR-CED-002).
- [x] **3.3** Implementar as 6 tool calls exatamente como definidas em D19 seção 4.2:
  - `get_case_status(caso_id)` — status do caso; valida que `caso_id` pertence ao Cedente logado antes de executar.
  - `get_simulation(caso_id, cenarios[])` — simula valores usando `comissao.calculator` de S3.
  - `get_pending_documents(caso_id)` — lista documentos `PENDENTE` ou `REJEITADO` do dossiê.
  - `get_escalonamento_history(caso_id)` — histórico de escalonamentos do `eventos_caso`.
  - `get_active_deadlines(caso_id)` — prazos ativos: proposta, assinatura, reversão Escrow.
  - `escalate_to_human(motivo, caso_id?, contexto_conversa)` — cria solicitação no Admin; retorna `{ protocolo, prazo: '1 dia útil' | '4 horas (horário comercial 8h-18h SEG-SEX)' }` (RN-061 item 4).
- [x] **3.4** Todas as 6 tools são **somente leitura** — nenhuma executa mutação no banco. Validar na code review que nenhuma tool chama endpoints POST/PATCH/DELETE (D19 seção 4.1).
- [x] **3.5** Cada tool call valida que o `caso_id` pertence ao `cedente_id` da sessão antes de retornar dados (isolamento — RN-011, RN-058 item 4).

### 4. Backend — RAG Pipeline

- [x] **4.1** Implementar `SupabaseVectorStoreRetriever` com parâmetros exatos de D19 seção 6.2:
  - `chunk_size: 512` tokens, `chunk_overlap: 64` tokens.
  - `embedding_model: 'text-embedding-3-small'`, `embedding_dimensions: 1536`.
  - `top_k: 5`, `score_threshold: 0.75`.
  - `index_type: 'hnsw'`, `distance_function: 'cosine'`, `hnsw_m: 16`, `hnsw_ef_construction: 200`.
- [x] **4.2** Implementar query híbrida `retrieveContext(query, caso_id?)` (D19 seção 6.4):
  - Gerar embedding da query com `text-embedding-3-small`.
  - Buscar vetorial via `supabase.rpc('match_knowledge', { query_embedding, match_threshold: 0.75, match_count: 5 })`.
  - Se `caso_id` fornecido: buscar contexto do caso (`casoService.getStatusSummary()`).
  - Combinar ambos os contextos no prompt do LLM.
- [x] **4.3** Criar script `apps/api/src/scripts/seed-knowledge-base.ts` para popular `guardiao_knowledge_base` com os 7 conteúdos de D19 seção 6.1:
  - Processo de repasse (fluxo Cedente → Fechamento).
  - Documentos obrigatórios (PF e PJ).
  - Cenários A, B, C, D.
  - FAQ — 50+ perguntas e respostas validadas.
  - Lei 13.786 (resumo acessível).
  - Anuência da construtora.
  - Glossário (cessão, distrato, Escrow, formalização, dossiê).
- [x] **4.4** Script de seed deve gerar embeddings via `text-embedding-3-small` e inserir na `guardiao_knowledge_base` com `ativo=true`.

### 5. Backend — Rate Limiting e Guardrails

- [x] **5.1** Implementar `GuardiaoGuard` com rate limit: máximo **30 mensagens por hora** por Cedente. Chave Redis: `rs:cedente:guardiao:rate:{cedente_id}` TTL **1 hora** com `INCR`. Ao atingir 30: retornar HTTP 429 com `cedente_message: "Você atingiu o limite de mensagens. Tente novamente em [X] minutos."` (D19 seção 7.1).
- [x] **5.2** Validação de comprimento: mensagem > **1.000 caracteres** → retornar HTTP 422 com `cedente_message: "Mensagem muito longa. Por favor, seja mais conciso."` (D19 seção 7.1).
- [x] **5.3** Filtro de prompt injection: detectar padrões `"ignore previous instructions"` e variantes → log da tentativa + resposta padrão educada (D19 seção 7.1).
- [x] **5.4** Implementar `GuardiaoOutputFilter` com 4 verificações (D19 seção 7.2, 7.3):
  - Bloqueio de garantias de resultado: padrões `GUARANTEE_PATTERNS` — substituir por linguagem de estimativa (RN-060).
  - Bloqueio de dados de Cessionários: `CESSIONARIO_PATTERNS` → resposta: `"Não tenho acesso a informações sobre compradores."` (RN-012).
  - Baixa confiança (`score < 0.80`): appender oferta de escalação "Não tenho certeza suficiente. Posso encaminhar para nossa equipe. Deseja?" (RN-061).
  - Linguagem jurídica vinculante: adicionar disclaimer (RN-060).
- [x] **5.5** Implementar padrões proibidos de saída exatamente como em D19 seção 7.3:
  - `GUARANTEE_PATTERNS`: `/você (vai|irá) receber (em|dentro de|até)/i`, `/garantimos que/i`, `/certamente (receberá|virá|acontecerá)/i`, `/prazo (exato|certo|garantido)/i`, `/com certeza/i`.
  - `FOMO_PATTERNS`: `/não perca essa oportunidade/i`, `/oferta por tempo limitado/i`, `/últimas vagas/i`, `/corra/i`.

### 6. Backend — Confidence Scoring e Escalação

- [x] **6.1** Implementar `calculateConfidence(params)` exatamente como em D19 seção 8.1:
  - `llm_confidence = Math.exp(avg(llm_logprobs))`.
  - `rag_confidence = rag_top_score`.
  - `context_bonus = has_case_context ? 0.1 : 0`.
  - `tool_penalty = tool_calls_successful ? 0 : -0.2`.
  - `confidence = Math.min(1, (llm_confidence * 0.5 + rag_confidence * 0.5) + context_bonus + tool_penalty)`.
- [x] **6.2** Quando `confidence < 0.80` OU Cedente pede explicitamente para falar com humano: anexar oferta de escalação ao response.
- [x] **6.3** Ao Cedente confirmar escalação: chamar tool `escalate_to_human()` → cria solicitação no Admin com `{ cedente_id, motivo, caso_id, contexto_conversa (últimas 3 mensagens), urgencia: 'NORMAL' }`.
- [x] **6.4** Horário comercial check (8h-18h BRT, SEG-SEX): usar fuso `America/Fortaleza`; se dentro do horário → prazo "4 horas"; fora → "1 dia útil" (RN-061 item 4).
- [x] **6.5** Persistir `escalado_para_humano=true` no registro da `conversas_guardiao` quando escalação ocorrer.
- [x] **6.6** Implementar takeover pelo Admin (RN-063): endpoint `POST /admin/ai/guardiao/:cedente_id/takeover` (Admin only) — notifica Cedente via Supabase Realtime com mensagem "Um especialista da nossa equipe está agora no chat para melhor atendê-lo."

### 7. Backend — Memória e Histórico

- [x] **7.1** Implementar `GuardiaoMemoryManager` com 2 níveis:
  - Redis (sessão): chave `rs:cedente:guardiao:session:{cedente_id}` TTL **4 horas**; renovar TTL a cada nova mensagem; máximo 20 mensagens ou 8.000 tokens.
  - Supabase DB (permanente): INSERT em `conversas_guardiao` após cada troca Cedente ↔ Guardião.
- [x] **7.2** Ao iniciar conversa: buscar histórico de sessão do Redis; se TTL expirado, carregar últimas 20 mensagens do banco como warm-up.
- [x] **7.3** Implementar job LGPD `GuardiaoAnonimizacaoCron` que roda diariamente às 02:00 BRT:
  - Buscar registros com `created_at < NOW() - INTERVAL '90 days'` e `anonimizado_em IS NULL`.
  - `UPDATE conversas_guardiao SET content = '[ANONIMIZADO]', anonimizado_em = NOW()`.
  - Preservar: `content_hash`, `confidence_score`, `tokens_*`, `latency_ms`, `trace_id`.
  - NUNCA enviar `cedente_id` ao Langfuse (D19 TL;DR, LGPD).
- [x] **7.4** Historico é imutável: nenhuma política de DELETE em `conversas_guardiao` para Cedente ou sistema (RN-062 item 4).

### 8. Backend — Observabilidade (Langfuse)

- [x] **8.1** Configurar `LangfuseCallbackHandler` com `LANGFUSE_PUBLIC_KEY` e `LANGFUSE_SECRET_KEY` (nunca expor no frontend).
- [x] **8.2** Rastrear no Langfuse: cada chamada LLM com `trace_id`, `model_version`, `tokens_input`, `tokens_output`, `latency_ms`, `confidence_score`.
- [x] **8.3** NUNCA enviar `cedente_id`, `cedente_nome`, ou qualquer dado identificador para o Langfuse (D19 TL;DR LGPD). Usar `trace_id` interno para correlação.
- [x] **8.4** Configurar alertas no Langfuse: taxa de escalação > **20%** em 24h → alerta para time de IA.
- [x] **8.5** Persistir `trace_id` do Langfuse no campo `conversas_guardiao.trace_id` para correlação interna.

### 9. Backend — ADR-CED-002: SSE Streaming

- [x] **9.1** Implementar endpoint `POST /ai/guardiao/message` com resposta SSE (Server-Sent Events) conforme ADR-CED-002 (D14):
  - Headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `X-Accel-Buffering: no`.
  - Cada token do LLM enviado como `data: {"token": "..."}`.
  - Evento final: `data: {"done": true, "confidence": 0.85, "trace_id": "..."}`.
  - Evento de tool call: `data: {"tool_call": "get_case_status", "status": "executing"}`.
- [x] **9.2** Mensagem de boas-vindas (primeira abertura do chat): enviar proativamente ao conectar ao SSE channel: "Olá, [nome]! Sou o Guardião do Retorno, seu assistente 24h. Posso te ajudar com: simular cenários, explicar o processo, orientar sobre documentos. Como posso ajudar?" + 4 botões de ação rápida (D19 seção 2.1, RN-058 item 1).

### 10. Backend — LangGraph: Cadastro Assistido

- [x] **10.1** Implementar `CadastroAssistidoGraph` com LangGraph.js 0.2+ para o fluxo multi-step de cadastro assistido (RN-089):
  - Nó `coletar_etapa_1`: solicitar dados do imóvel via chat.
  - Nó `coletar_etapa_2`: solicitar dados financeiros.
  - Nó `explicar_simulador`: explicar os 4 cenários.
  - Nó `ajudar_escolha_cenario`: comparar cenários com base nos valores informados.
  - Checkpoint entre cada nó (estado salvo no Redis da sessão).
  - O Guardião NUNCA avança o wizard — apenas preenche campos para confirmação do Cedente (RN-089 item 5).
- [x] **10.2** Ao preencher campo no wizard via chat: retornar evento SSE `{ "wizard_field": { "campo": "valor_pago", "valor": 40000000, "confirmacao_necessaria": true } }` → frontend exibe pré-preenchimento para confirmar antes de salvar.

---

## 🔴 FEATURE 3 — Frontend: Chat UI

### 11. Frontend — Tela Assistente IA

- [x] **11.1** Criar `apps/web-cedente/src/app/(authenticated)/assistente/page.tsx` — chat UI usando Vercel AI SDK 4+.
- [x] **11.2** Implementar `useChat()` do Vercel AI SDK apontando para `POST /ai/guardiao/message` com streaming SSE.
- [x] **11.3** Exibir mensagem de boas-vindas com 4 botões de ação rápida clicáveis (D19 seção 2.1).
- [x] **11.4** Indicador de digitação ("...") enquanto tokens chegam via SSE.
- [x] **11.5** Quando evento `{ "tool_call": "get_case_status" }` recebido: exibir badge discreto "Consultando status do caso...".
- [x] **11.6** Ao receber evento `{ "wizard_field": {...} }`: exibir cartão de confirmação no chat com campo pré-preenchido e botão "Confirmar" e "Corrigir".
- [x] **11.7** Contador de rate limit: exibir discretamente "Mensagens restantes: [X]/30" no topo do chat; ao atingir 0 → input desabilitado com mensagem "Limite atingido. Disponível novamente em [X] min".
- [x] **11.8** Histórico de conversas: exibir conversas anteriores organizadas por data (RN-062 item 2). Scroll infinito para conversas mais antigas.
- [x] **11.9** Chat do Guardião visível no canto inferior direito em TODAS as telas do wizard (componente persistente no layout — RN-089).
- [x] **11.10** Botão de escalação: quando Guardião oferece escalação → exibir botão "Falar com especialista" no chat; ao clicar → confirmar e exibir `{ protocolo, prazo }`.

---

## 🧪 TESTES

### 12. Testes unitários (Vitest)

- [x] **12.1** `output-filter.spec.ts` — testar `GUARANTEE_PATTERNS`: "você vai receber em 5 dias" → filtrado; "provavelmente em algumas semanas" → não filtrado.
- [x] **12.2** `output-filter.spec.ts` — testar bloqueio de dados de Cessionário: resposta contendo "comprador João Silva" → `"Não tenho acesso a informações sobre compradores."`.
- [x] **12.3** `confidence-scorer.spec.ts` — testar `calculateConfidence()`: sem contexto de caso + baixo RAG score → < 0.80; com contexto + tool call OK + RAG alto → próximo de 0.90.
- [x] **12.4** `guardiao.guard.spec.ts` — testar rate limit: 30 mensagens → OK; 31ª → HTTP 429.
- [x] **12.5** `guardiao.service.spec.ts` — testar que tool `get_case_status` com `caso_id` de outro Cedente retorna HTTP 403 (isolamento).
- [x] **12.6** `guardiao-anonimizacao.cron.spec.ts` — testar que conteúdo > 90 dias é anonimizado mas `content_hash` e `confidence_score` são preservados.

### 13. Testes de integração (Supertest)

- [x] **13.1** T-035: `POST /ai/guardiao/message` com pergunta "Qual o status do meu caso?" → verificar SSE stream com tokens + evento `tool_call = 'get_case_status'`; response final contém dados do caso (mock).
- [x] **13.2** T-036: `POST /ai/guardiao/message` com "Quando vou receber uma proposta?" → verificar que response NÃO contém padrões de `GUARANTEE_PATTERNS`.
- [x] **13.3** T-037: `POST /ai/guardiao/message` com `confidence` mock < 0.80 → verificar que response contém oferta de escalação.
- [x] **13.4** Rate limit: 31 mensagens sequenciais → verificar HTTP 429 na 31ª.
- [x] **13.5** Escalação: chamar `escalate_to_human` → verificar que registro em Admin é criado + response inclui `protocolo`.

---

## ✅ AUTO-VERIFICAÇÃO (12 Checks)

| #   | Check                  | Critério                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Status |
| --- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | **Nomenclatura**       | `conversas_guardiao` (não `ai_messages`), `guardiao_knowledge_base` (não `embeddings`), `content_hash` (SHA-256), `confidence_score DECIMAL(3,2)`, `cedente_auth_id` (não `user_id`), model `gpt-4-turbo-2024-04-09` (versão fixada, não alias)                                                                                                                                                                                                                                                                     | ✅     |
| 2   | **Verificabilidade**   | Tool calls verificáveis: `get_case_status` valida `caso_id` pertence ao Cedente; `GuardiaoAnonimizacaoCron` verifica campo `anonimizado_em IS NULL` antes de anonimizar                                                                                                                                                                                                                                                                                                                                             | ✅     |
| 3   | **Valores numéricos**  | Rate limit **30 mensagens/hora**; mensagem máx **1.000 caracteres**; sessão Redis TTL **4 horas**; máx **20 mensagens** ou **8.000 tokens** na sessão; anonimização após **90 dias**; escalação threshold `confidence < 0.80`; RAG top-k **5**, score mínimo **0.75**; chunk **512 tokens**, overlap **64 tokens**                                                                                                                                                                                                  | ✅     |
| 4   | **Contagem de itens**  | 6 tool calls: `get_case_status`, `get_simulation`, `get_pending_documents`, `get_escalonamento_history`, `get_active_deadlines`, `escalate_to_human`; 4 botões de ação rápida na boas-vindas; 7 categorias de conteúdo RAG                                                                                                                                                                                                                                                                                          | ✅     |
| 5   | **Máquina de estados** | Sem estado de caso próprio do Guardião — ele apenas lê; histórico é imutável (sem DELETE); memória de sessão expira em 4h (TTL Redis); anonimização = `content → '[ANONIMIZADO]'` + `anonimizado_em = NOW()`                                                                                                                                                                                                                                                                                                        | ✅     |
| 6   | **RBAC**               | `GuardiaoGuard` verifica auth antes de processar; `cedente_id` nunca enviado ao Langfuse; todas as tools validam ownership antes de retornar dados; Admin takeover via endpoint Admin-only                                                                                                                                                                                                                                                                                                                          | ✅     |
| 7   | **Anti-scaffold**      | `GuardiaoOutputFilter` implementa padrões regex reais (não apenas retorna resposta hardcoded); `calculateConfidence()` usa logprobs reais do GPT-4; `CadastroAssistidoGraph` tem checkpoints reais no Redis                                                                                                                                                                                                                                                                                                         | ✅     |
| 8   | **ADR-CED-002**        | Endpoint `/ai/guardiao/message` usa SSE (não REST simples nem WebSocket); headers corretos: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `X-Accel-Buffering: no`                                                                                                                                                                                                                                                                                                                                   | ✅     |
| 9   | **LGPD**               | `cedente_id` NUNCA no Langfuse; `content` anonimizado após 90 dias; `content_hash` SHA-256 preservado; `anonimizado_em` preenchido; job roda às 02:00 BRT                                                                                                                                                                                                                                                                                                                                                           | ✅     |
| 10  | **Stack fixada**       | LangChain.js 0.3+, LangGraph.js 0.2+, Vercel AI SDK 4+, `gpt-4-turbo-2024-04-09` (versão fixada), `text-embedding-3-small`, pgvector HNSW — verificar `package.json` após implementação                                                                                                                                                                                                                                                                                                                             | ✅     |
| 11  | **Guardrails**         | `GUARANTEE_PATTERNS` + `FOMO_PATTERNS` + bloqueio Cessionário + disclaimer jurídico — 4 guardrails implementados; nenhuma tool executa mutação (validação de code review)                                                                                                                                                                                                                                                                                                                                           | ✅     |
| 12  | **Cobertura de REQs**  | REQ-097 (Guardião capacidades 9 itens), REQ-098 (Guardião restrições 5 itens), REQ-099 (contexto do caso), REQ-100 (tom empático), REQ-101 (proibição garantias), REQ-102 (escalação humano), REQ-103 (histórico conversas), REQ-150 (SSE ADR-CED-002), REQ-186 (rate limit 30/h), REQ-187 (stack fixada), REQ-188 (RAG chunking), REQ-189 (memória sessão 4h), REQ-190 (tool calls somente leitura), REQ-191 (LangGraph cadastro assistido), REQ-192 (Langfuse LGPD), REQ-193 (anonimização 90d) — 100% atribuídos | ✅     |

---

## 📋 PENDÊNCIAS

| ID      | Tipo       | Descrição                                                                                                                                                                                                                                                                                                |
| ------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PEND-11 | ⚠️ AMBÍGUO | D19 menciona "5 tools de leitura + 1 de escalação" no comentário de arquitetura, mas lista 5 tools de leitura (`get_case_status`, `get_simulation`, `get_pending_documents`, `get_escalonamento_history`, `get_active_deadlines`) + 1 de escalação (`escalate_to_human`) = 6 no total. Adotado: 6 tools. |
| PEND-12 | ⚠️ AMBÍGUO | `LANGFUSE_PUBLIC_KEY` está listado nas 42 vars de ambiente (D22). Verificar se `LANGFUSE_SECRET_KEY` também está — não aparecer explicitamente na lista. [REVISÃO MANUAL — confirmar vars D22 antes de implementar]                                                                                      |
