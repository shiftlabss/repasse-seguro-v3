# S3 — Agente Core (LangGraph + RAG + Tools + Chat)

| Campo                | Valor                                                                                                                                                                                                                                                                              |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S3                                                                                                                                                                                                                                                                                 |
| **Nome**             | Agente Core                                                                                                                                                                                                                                                                        |
| **Tipo**             | Dinâmica — Módulo de negócio fullstack                                                                                                                                                                                                                                             |
| **Template**         | B (Módulo Fullstack)                                                                                                                                                                                                                                                               |
| **Docs Consultados** | D01, D02, D05.1–05.5, D06, D09, D14, D16, D17, D19                                                                                                                                                                                                                                 |
| **Cross-cutting**    | D10 (Glossário)                                                                                                                                                                                                                                                                    |
| **REQs cobertos**    | REQ-017 a REQ-026, REQ-079–082, REQ-096, REQ-101–108, REQ-114–117, REQ-128, REQ-158–162, REQ-168, REQ-177–182                                                                                                                                                                      |
| **Objetivo**         | DaniCedenteAgent funcional com LangGraph state machine (5 estados), 5 tools LangChain, streaming SSE, RAG com pgvector, memória de sessão Redis, system prompt v1.0, endpoints de chat (POST/GET/PATCH sessions + messages), telas T-01 a T-07 do chat widget com todos os estados |

---

## Critério de Conclusão da S3

Ao final desta sprint:

- Cedente envia mensagem via `POST /api/v1/chat/sessions/:id/messages` e recebe streaming SSE com resposta da Dani
- Agente entra em estado `analyzing_proposal` ao receber pergunta sobre proposta
- Tool `get-opportunity` retorna dados APENAS do Cedente autenticado (nenhum dado de outro Cedente)
- RAG retorna chunks relevantes da base de conhecimento para consultas sobre regras da plataforma
- Chat widget (T-01 a T-07) renderiza com estados Skeleton/Empty/Error/Populated
- Rate limit de 30 msg/h ativo no endpoint de chat
- `confidence_score` salvo em cada `ChatMessage` via Langfuse

---

## 🗄️ BANCO

- [x] **Criar seed de `KnowledgeEmbedding`** com 6 categorias: `PLATFORM_RULES` (~50 chunks), `FAQ_CEDENTE` (~30), `ESCROW_PROCESS` (~20), `KYC_PROCESS` (~15), `DOSSIER_RULES` (~15), `SCENARIOS_GUIDE` (~10) — embeddings gerados via `text-embedding-3-small`
  - Validação: seed insere ≥130 chunks; categorias correspondem exatamente aos 6 nomes; busca semântica retorna resultados relevantes em testes

---

## ⚙️ BACKEND — FEATURE: Chat Sessions

- [x] **Implementar `POST /api/v1/chat/sessions`** em `ChatController`: (1) cria `ChatSession` com `cedente_id` do JWT, `entry_point` (PAINEL/OPORTUNIDADE/NEGOCIACAO — conforme ponto de entrada), `opportunity_id?` e `proposal_id?` opcionais, `status: ATIVA`; (2) retorna `{ data: { id, cedente_id, entry_point, status, created_at } }`; (3) error: 401 sem token; 422 `DCE-CHAT-4220_001` se `opportunity_id` inválido
  - Validação: sessão criada com `cedente_id` correto; `entry_point` registrado; testes: criar sessão PE-1 (sem contexto), PE-2 (com opportunity_id), PE-3 (com proposal_id)
- [x] **Implementar `GET /api/v1/chat/sessions`** paginado: (1) lista sessions do Cedente autenticado (WHERE cedente_id = req.user.cedenteId AND deleted_at IS NULL); (2) paginação: page, per_page(max 100), sort(default: last_message_at), order(desc); (3) objeto meta com page, per_page, total, total_pages
  - Validação: retorna apenas sessions do Cedente autenticado; outro Cedente não vê sessions alheias; meta preenchido corretamente
- [x] **Implementar `GET /api/v1/chat/sessions/:session_id`** com histórico de mensagens: (1) valida que `session.cedente_id === req.user.cedenteId` (se não → 404 `DCE-CHAT-4040_001`); (2) retorna session + `messages: ChatMessage[]` (últimas N mensagens, append-only)
  - Validação: session de outro Cedente retorna 404 (não 403) — não revelando existência do recurso
- [x] **Implementar `PATCH /api/v1/chat/sessions/:session_id/close`** (1) valida pertencimento; (2) atualiza `status: ENCERRADA`, `last_message_at: now()`; (3) dispara evento para coleta de CSAT (hook para S8); (4) retorna 204 No Content
  - Validação: session encerrada muda status para ENCERRADA; CSAT pode ser coletado após encerramento

### FEATURE: Chat Messages com Streaming SSE

- [x] **Implementar `POST /api/v1/chat/sessions/:session_id/messages`** (endpoint principal do chat): (1) valida JwtAuthGuard + pertencimento da session ao cedente_id; (2) verifica rate limit 30 msg/h via Redis `rate_limit:{cedente_id}` — retorna 429 `DCE-CHAT-4290_001` se excedido, com `retry_after` em segundos e headers `X-RateLimit-Limit/Remaining/Reset`; (3) insere `ChatMessage { role: USER, content: sanitized_content }` no banco; (4) invoca `DaniCedenteAgent.invoke({ sessionId, cedenteId, message })`; (5) abre SSE stream e transmite tokens ao cliente conforme chegam do GPT-4; (6) ao final: insere `ChatMessage { role: ASSISTANT, content, langfuse_trace_id, confidence_score }`; (7) atualiza `ChatSession.message_count++` e `last_message_at`
  - Validação: tokens chegam ao cliente em streaming SSE (Content-Type: text/event-stream); mensagem USER e ASSISTANT salvas no banco; rate limit 31ª mensagem → 429; `confidence_score` salvo
- [x] **Implementar `GET /api/v1/chat/sessions/:session_id/messages`** com paginação (histórico de mensagens): retorna `{ data: ChatMessage[], meta: { page, per_page, total, total_pages } }`; ordenado por `created_at ASC`
  - Validação: paginação funciona; apenas mensagens da session do Cedente autenticado
- [x] **Implementar evento SSE de tipos corretos**: `{ type: "token", content: string }` para cada token; `{ type: "done", message_id: string, confidence_score: number }` ao finalizar; `{ type: "error", code: string, message: string }` em caso de erro; `{ type: "unavailable", message: "A Dani está temporariamente indisponível..." }` se circuit breaker aberto
  - Validação: frontend pode distinguir os 4 tipos de evento SSE; teste de streaming com mock do OpenAI

---

## ⚙️ BACKEND — FEATURE: DaniCedenteAgent (LangGraph)

- [x] **Implementar `DaniCedenteAgent`** em `src/modules/agent/dani-cedente.agent.ts` como LangGraph state machine com tipo `AgentState = { sessionId, cedenteId, currentState, context: { opportunityId?, proposalId?, escrowId? }, messages: ChatMessage[], ragContext: string[], pendingConfirmation?: 'accept_proposal'|'reject_proposal'|'approve_extension'|'reject_extension' }`
  - Validação: tipo `AgentState` implementado com todos os campos; `pendingConfirmation` como union type exato
- [x] **Implementar as 5 transições de estado do LangGraph**: (1) `[*] → idle`: sessão iniciada; (2) `idle → analyzing_proposal`: Cedente pergunta sobre proposta; (3) `idle → dossier_guidance`: Cedente pergunta sobre dossiê; (4) `idle → escrow_monitoring`: notificação de evento Escrow; (5) `analyzing_proposal → idle`: ação concluída; (6) `dossier_guidance → idle`: orientação concluída; (7) `escrow_monitoring → idle`: status atualizado; (8) `idle → takeover`: Admin inicia takeover; (9) `analyzing_proposal → takeover`: Admin inicia takeover; (10) `takeover → idle`: Admin encerra takeover
  - Validação: todas as 10 transições implementadas; estado inválido não é alcançável; testes de cada transição
- [x] **Implementar tool `get-opportunity`**: `{ name: 'get-opportunity', description: '...', parameters: { opportunity_id: z.string().uuid() }, execute: async ({ opportunity_id }, { cedenteId }) => opportunityRepository.findByCedenteId(opportunity_id, cedenteId), timeout: 5000, retries: 2, fallback: 'Não consegui carregar os dados da sua oportunidade.' }`
  - Validação: `cedenteId` NUNCA vem do LLM — sempre injetado do context; `opportunityRepository.findByCedenteId` enforça WHERE `cedente_id = cedenteId`; retorna 404 se opportunity pertence a outro Cedente
- [x] **Implementar tool `get-proposal`**: `{ name: 'get-proposal', parameters: { proposal_id: z.string().uuid() }, timeout: 5000, retries: 2, fallback: 'Não consegui carregar os dados da proposta.' }` — enforça `cedente_id` via RLS; retorna `Proposal + simulated_return`
  - Validação: `cedente_id` enforçado via WHERE; proposta de outro Cedente retorna erro sem vazar existência do recurso
- [x] **Implementar tool `get-dossier`**: `{ name: 'get-dossier', parameters: { opportunity_id: z.string().uuid() }, timeout: 5000, retries: 2, fallback: 'Não consegui carregar o status do seu dossiê.' }` — retorna `DossierDocument[]` com status de cada documento
  - Validação: retorna apenas documentos do `opportunity_id` do Cedente autenticado; testes com dossiê completo e incompleto
- [x] **Implementar tool `get-escrow`**: `{ name: 'get-escrow', parameters: { proposal_id: z.string().uuid() }, timeout: 5000, retries: 2, fallback: 'Não consegui carregar o status do Escrow.' }` — retorna `EscrowTransaction`
  - Validação: enforça `cedente_id` via JOIN com proposals; testes com os 4 status de Escrow
- [x] **Implementar tool `simulate-return`**: `{ name: 'simulate-return', parameters: { opportunity_id: z.string().uuid(), proposal_value: z.number().positive() }, execute: async ({ opportunity_id, proposal_value }, { cedenteId }) => { net_return = proposal_value - opportunity.outstanding_balance; return { net_return, proposal_value, saldo_devedor: opportunity.outstanding_balance, calculation: 'R$ X - R$ Y = R$ Z', label: net_return >= 0 ? 'Retorno positivo.' : 'Retorno negativo...' } }, timeout: 3000, retries: 1 }`
  - Validação: fórmula exata `net_return = proposal_value - outstanding_balance`; `label` distingue positivo de negativo; aviso de estimativa emitido pela Dani na resposta final
- [x] **Implementar tratamento de erro em tools**: timeout → retry 1x → mensagem fallback; 404 (isolamento) → sem retry → mensagem de bloqueio; 500 → retry 2x com backoff 1s/2s → log Sentry + continuar sem tool result
  - Validação: testes com mock de cada cenário de erro; `cedenteId` NUNCA passado pelo LLM

### FEATURE: System Prompt v1.0

- [x] **Criar `src/modules/agent/prompts/system-prompt.v1.0.ts`** com template: "Você é a Dani, assistente da plataforma Repasse Seguro. Sua missão é acompanhar o Cedente {cedente_name} no ciclo completo do repasse." + 5 PRINCÍPIOS OBRIGATÓRIOS (isolamento, próximo passo, sem ação financeira sem confirmação, mensagem de bloqueio fora do escopo, tom acolhedor/direto) + CONTEXTO DA SESSÃO (opportunity_summary, opportunity_status, pending_proposals_count) + BASE DE CONHECIMENTO (rag_chunks placeholder)
  - Validação: system prompt em arquivo separado; nunca contém dados do Cedente hardcoded; variáveis injetadas em runtime
- [x] **Configurar GPT-4 com parâmetros**: `model: 'gpt-4-turbo'`, `temperature: 0.2`, `max_tokens: 1024`, `stream: true`, `tool_choice: 'auto'`
  - Validação: parâmetros exatos conforme D19 seção 4.1; versão de modelo fixada em produção via env var
- [x] **Implementar versionamento de prompts**: nome de arquivo `system-prompt.v{MAJOR}.{MINOR}.ts`; critério para MAJOR: mudança de comportamento fundamental; critério para MINOR: ajuste de instrução; suite de 20 cenários de regressão em `test/agent/prompt-regression.spec.ts`
  - Validação: arquivo v1.0 presente; suite de 20 cenários criada (smoke tests básicos)

### FEATURE: RAG Pipeline

- [x] **Implementar `RagService`** em `src/modules/rag/rag.service.ts` com método `semanticSearch(query: string, k: number = 5): Promise<string[]>`: (1) gera embedding via OpenAI `text-embedding-3-small`; (2) query SQL raw via pgvector: `SELECT content FROM knowledge_embeddings ORDER BY embedding <=> $1::vector LIMIT $2`; (3) retorna os top-k chunks por cosine similarity
  - Validação: `k=5` por padrão; dimensão embedding = 1536; consulta com `vector_cosine_ops`; testes com mock do OpenAI Embeddings
- [x] **Implementar `RAGIngestWorker`** em `src/modules/rag/rag-ingest.worker.ts` que consome fila `rag.ingest`: (1) chunking com `RecursiveCharacterTextSplitter({ chunkSize: 800, chunkOverlap: 100, separators: ["\n\n", "\n", ". ", " "] })`; (2) gera embedding para cada chunk via OpenAI; (3) upsert em `knowledge_embeddings`; (4) em caso de erro OpenAI: NACK e republica (máx 3 tentativas) → DLQ `rag.ingest.dlq` após 3 falhas
  - Validação: chunk size ~800 tokens; overlap 100 tokens; separadores exatos; retry 3x antes de DLQ
- [x] **Implementar `POST /api/v1/admin/knowledge`** (Admin only, `@RequireRole('admin')`): (1) recebe `{ content: string, source: string, metadata: object }`; (2) publica em `rag.ingest`; (3) retorna `202 Accepted { data: { jobId: string } }`
  - Validação: apenas Admin consegue chamar; payload enviado para fila; jobId rastreável

### FEATURE: Memória de Sessão Redis

- [x] **Implementar política de estado no Redis** com chave `dani:agent_state:{session_id}`: (1) estrutura `AgentStateCache { sessionId, cedenteId, currentState, recentMessages (últimas 10), context: { opportunityId?, proposalId? }, pendingConfirmation?, updatedAt }`; (2) escrita após cada turno completo; (3) leitura no início de cada `invoke()`; (4) TTL 1800s reset a cada escrita; (5) cache miss → recria estado inicial sem erro
  - Validação: TTL 1800s confirmado; recentMessages limitado a 10; cache miss não causa erro; testes de expiração
- [x] **Implementar window de contexto no LLM**: (1) histórico de mensagens enviado ao LLM: últimas 10 mensagens (truncar as mais antigas, manter sempre o system prompt); (2) RAG: 5 chunks (k=5); (3) total input máximo: 6.000 tokens; (4) estratégia de truncamento: histórico primeiro, depois RAG se necessário
  - Validação: nunca enviadas mais de 10 mensagens de histórico; total tokens estimado ≤6.000; testes com histórico longo

---

## 🖥️ FRONTEND — Chat Widget

> O AI-Dani-Cedente não tem frontend próprio — o chat widget é embutido como componente flutuante na plataforma Repasse Seguro. Esta seção especifica os contratos de UI que o frontend da plataforma deve implementar ao integrar o widget.

### FEATURE: Layout Base

- [x] **Implementar `ChatWidgetLayout`**: desktop: `width: 400px`, `height: 608px`, `position: fixed`, `bottom: 24px`, `right: 24px`, `z-index: 9999`, `box-shadow: shadow-lg`; mobile: `ChatFullscreenLayout` — `100vw × 100dvh`, sem border-radius, sem sombra
  - Validação: widget visível em desktop sem sobrepor conteúdo principal; mobile ocupa tela inteira; breakpoint de troca definido
- [x] **Implementar 4 estados obrigatórios em cada tela**: `Skeleton` (`<Skeleton>` shadcn/ui — nunca `<Spinner>`), `Empty` (`<EmptyState>` com ilustração + título + descrição + CTA opcional), `Error` (`<Alert variant="destructive">` + botão retry), `Populated` (conteúdo real)
  - Validação: spinners proibidos; Skeleton exibido durante streaming; EmptyState ao abrir chat sem histórico

### FEATURE: T-01 — Chat Inicial (Primeiro Acesso)

- [x] **Implementar T-01** (RF-DCE-005, RN-DCE-005): (1) Header: avatar da Dani, nome "Dani", badge "Guardiã do Retorno", indicador de status disponível; (2) Área de mensagens vazia com fundo `--background`; (3) Mensagem de boas-vindas por estado: KYC aprovado+oportunidade ativa → "Olá! Sou a Dani, sua Guardiã do Retorno..." + 4 sugestões; KYC pendente → boas-vindas + CTA link clicável para verificação de identidade; sem oportunidade → "Você ainda não tem uma oportunidade publicada. Quer começar?"; (4) Campo de mensagem placeholder "Digite sua mensagem..."; (5) Botão de envio
  - Validação: 3 estados do Cedente renderizando a mensagem correta; link KYC clicável; campo de mensagem funcional
- [x] **Implementar 4 sugestões de conversa** (RF-DCE-008, RN-DCE-008) — apenas se KYC aprovado + oportunidade ativa: chips clicáveis com textos exatos: "Qual o retorno esperado para a minha oportunidade?", "Tenho uma proposta recebida. Vale a pena aceitar?", "O que ainda falta no meu dossiê?", "Quanto tempo demora para concluir o repasse?"
  - Validação: 4 chips com textos exatos; click dispara envio da mensagem; ocultos se KYC pendente ou sem oportunidade

### FEATURE: T-02 e T-03 — Chat com Contexto

- [x] **Implementar T-02** (PE-2, RF-DCE-006): banner de contexto "Você está consultando sobre a oportunidade [OPR-XXXX-XXXX]" removido após primeira interação; contexto `{ opportunity_id, opportunity_code, opportunity_state, delta, scenario_chosen }` injetado no system message (não visível ao Cedente)
  - Validação: banner exibe código OPR correto; desaparece após primeira mensagem; contexto não vazado na UI
- [x] **Implementar T-03** (PE-3, RF-DCE-006): chat abre a partir da tela de negociação com `proposal_id` pré-carregado; contexto da proposta no system message
  - Validação: `proposal_id` passado ao criar ChatSession; contexto da proposta disponível para o agente

### FEATURE: T-04 — Chat em Conversa

- [x] **Implementar T-04** com `ChatBubble` (mensagem individual): variante primária (Dani — alinhado à esquerda, fundo `--rs-muted`) e secundária (Cedente — alinhado à direita, fundo `--rs-primary`); timestamps; `TypingIndicator` (`TypingIndicator` com animação de 3 pontos bounce em `--rs-muted-foreground`) exibido durante streaming SSE
  - Validação: bolhas visuais distintas; TypingIndicator ativo durante streaming; desaparece após `type: "done"`
- [x] **Implementar `RateLimitCounter`**: 0–24 mensagens → `--rs-primary` (normal); 25–29 mensagens → `--rs-warning` (warning); 30 mensagens → `--rs-destructive` (blocked) + campo de entrada desabilitado com contador regressivo até liberar quota
  - Validação: contador atualiza a cada mensagem enviada; campo desabilitado na 31ª; contador regressivo exibido

### FEATURE: T-05 — Chat com AnalysisCard Inline

- [x] **Implementar `AnalysisCard`** com 4 variantes — cor de header diferente por variante: (1) `Proposta` (análise de proposta — RN-DCE-014); (2) `Simulação` (retorno líquido — RN-DCE-015); (3) `Dossiê` (status dos documentos — RN-DCE-013); (4) `Escrow` (status do Escrow — RN-DCE-018); cada variante com `StatusBadge` correspondente
  - Validação: 4 variantes implementadas; `StatusBadge` mapeado corretamente para cada enum de status
- [x] **Implementar `StatusBadge`** com mapeamentos exatos: OpportunityStatus (RASCUNHO → muted, PUBLICADA → success, EM_NEGOCIACAO → primary, PAUSADA → warning, EXPIRADA/CANCELADA → destructive); ProposalStatus (RECEBIDA → primary, EM_ANALISE → warning); EscrowStatus (AGUARDANDO_DEPOSITO → warning, DEPOSITO_CONFIRMADO → success, LIBERADO → success, REVERTIDO/EXPIRADO → destructive); DossierDocumentStatus (APROVADO → success, EM_ANALISE → warning, REJEITADO → destructive, AGUARDANDO_ENVIO → muted)
  - Validação: todos os valores de enum mapeados; sem valor não mapeado causando UI quebrada

### FEATURE: T-06 — Modal de Confirmação

- [x] **Implementar `ConfirmationModal`** para ações críticas: (1) aceite de proposta — exibe: valor da proposta, retorno líquido estimado, próximos passos (depósito Escrow); (2) retirada de oportunidade do marketplace; (3) extensão de prazo de Escrow; botões: "Cancelar" e ação confirmatória com texto específico
  - Validação: modal aparece ANTES de qualquer ação financeira; sem modal → ação não pode ser executada; testes de regressão verificam que Dani nunca executa ação financeira sem modal confirmado

### FEATURE: T-07 — Estados de Sistema

- [x] **Implementar T-07** com estados: (1) TypingIndicator quando resposta > 5s; (2) mensagem de timeout "A Dani está demorando mais que o esperado..." + botões `[Aguardar] [Tentar novamente]` quando > 10s (2× SLA); (3) estado de rate limit com contador regressivo; (4) estado de indisponibilidade "A Dani está temporariamente indisponível. Para urgências, entre em contato com o suporte."; (5) `CsatRating` (1–5 estrelas) após encerramento de sessão
  - Validação: todos os 5 estados de sistema implementados; CSAT é opcional (Cedente pode fechar sem responder)

---

## 🔀 Cross-Módulo

- Tools do agente consomem serviços de `opportunity`, `proposal`, `dossier`, `escrow` — módulos que serão implementados em S4–S7. Na S3: implementar as interfaces (contratos TypeScript) que as tools esperam; na S4–S7: implementar as queries reais
- `NotificationService.publishNotification()` chamado ao encerrar sessão (para CSAT) — implementado em S8; na S3: apenas o hook de evento

---

## 🧪 TESTES

- [x] **Criar `src/modules/agent/dani-cedente.agent.spec.ts`** com testes unitários: (1) transição idle → analyzing_proposal ao receber mensagem sobre proposta; (2) estado pendingConfirmation criado ao processar aceite de proposta (sem executar ação); (3) tool `simulate-return` com proposal_value=85000 e outstanding_balance=60000 → net_return=25000; (4) tool `get-opportunity` com cedenteId de outro Cedente → ToolError sem vazar dados; (5) cache miss no Redis → estado inicial criado sem erro
  - Validação: 5 cenários com mock LangGraph, mock Redis e mock repositórios; nenhuma chamada real ao LLM
- [x] **Criar `src/modules/rag/rag.service.spec.ts`** com testes: (1) `semanticSearch("escrow")` retorna 5 chunks; (2) embedding gerado com `text-embedding-3-small`; (3) dimensão = 1536; (4) query SQL raw com `vector_cosine_ops`
  - Validação: mocks do OpenAI Embeddings; 4 cenários cobertos
- [x] **Criar `test/chat.e2e-spec.ts`** com testes de integração: (1) POST /chat/sessions → 201 com session_id; (2) POST /chat/sessions/:id/messages com mensagem → SSE stream recebido; (3) 31ª mensagem → 429 com retry_after; (4) GET /chat/sessions de outro Cedente → lista vazia (isolamento)
  - Validação: 4 cenários E2E com banco de teste; mock do OpenAI para evitar custos

---

## 🔍 AUTO-VERIFICAÇÃO S3

| Check                | Critério                                                                                                                                                                                                                                   | Status |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------ |
| #1 Nomenclatura      | `DaniCedenteAgent`, `get-opportunity`/`get-proposal`/`get-dossier`/`get-escrow`/`simulate-return`, `ChatWidgetLayout`/`ChatFullscreenLayout`, `AnalysisCard`, `TypingIndicator`, `ConfirmationModal`, `CsatRating` — nomes exatos dos docs | [x]    |
| #2 Valores numéricos | temperature: 0.2; max_tokens: 1024; k=5 RAG; TTL 1800s Redis; 10 msgs histórico; 6.000 tokens max input; chunk 800 tokens overlap 100; rate limit 30 msg/h; ≤5s SLA; ≤10s timeout com mensagem                                             | [x]    |
| #3 Estados LangGraph | 5 estados: idle, analyzing_proposal, escrow_monitoring, dossier_guidance, takeover; todas as 10 transições implementadas                                                                                                                   | [x]    |
| #4 Tools             | 5 tools: timeout/retries/fallback definidos; cedenteId nunca vem do LLM — sempre do context                                                                                                                                                | [x]    |
| #5 System prompt     | v1.0; 5 princípios obrigatórios; variáveis em runtime; arquivo separado imutável                                                                                                                                                           | [x]    |
| #6 SSE eventos       | 4 tipos: token, done, error, unavailable — tratados no frontend                                                                                                                                                                            | [x]    |
| #7 Telas             | T-01 a T-07 implementadas; 4 estados (Skeleton/Empty/Error/Populated) em cada; spinners proibidos                                                                                                                                          | [x]    |
| #8 Anti-scaffold     | DaniCedenteAgent com state machine real; tools com lógica de isolamento; RAG com pgvector real                                                                                                                                             | [x]    |
| #9 ConfirmationModal | Modal obrigatório ANTES de aceite de proposta, retirada e extensão Escrow; Dani nunca age sem confirmação                                                                                                                                  | [x]    |
| #10 Glossário        | `confidence_score` salvo em `ChatMessage`; `pendingConfirmation` como union type; `ProactiveToast` previsto para S8                                                                                                                        | [x]    |
| #11 Cobertura REQs   | REQ-017 a REQ-026, REQ-079–082, REQ-096, REQ-101–108, REQ-114–117, REQ-128, REQ-158–162, REQ-168, REQ-177–182 cobertos                                                                                                                     | [x]    |
| #12 Testes           | Agent spec (5 cenários), RAG spec (4 cenários), Chat E2E (4 cenários) passando; nenhuma chamada real ao LLM                                                                                                                                | [x]    |
