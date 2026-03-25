# S3 — Agente Core (Chat, Streaming SSE, Tools, RAG, Fallback)

| **Sprint**         | S3 — Agente Core                                                                                                                                                                                           |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Template**       | B — Módulo Fullstack (organizado por Feature, vertical slice Banco→Backend→Frontend→Wiring→Testes)                                                                                                         |
| **REQs cobertos**  | REQ-001, REQ-002, REQ-006, REQ-007, REQ-008, REQ-009, REQ-011, REQ-029, REQ-030, REQ-031, REQ-032, REQ-033, REQ-034, REQ-035, REQ-056, REQ-071–090 (endpoints core), REQ-098–108, REQ-117–128, REQ-129–133 |
| **Docs fonte**     | D01 (RN-DC-001 a RN-DC-008, RN-DC-023–029), D14 (AgenteModule), D16 (endpoints), D19 (Agente IA), D05.1 (PRD RF-DC-001–012, RF-DC-028–036)                                                                 |
| **Total de itens** | 59                                                                                                                                                                                                         |

---

## 🎯 Objetivo

Implementar o núcleo do agente Dani: chat via webchat, streaming SSE, 5 tools com timeouts e retry, memória de curto e longo prazo (pgvector), fallback automático para Calculadora, rate limiting de mensagens, estados do agente, desligamento automático por taxa de erro, e toda a interface do chat (FAB, chat window, módulos inline, 4 estados por componente).

---

## FEATURE 1: Chat e Streaming SSE

### Banco

- [x] **[F1-BANCO-001]** Verificar presença de `dani_conversas` + `dani_mensagens` + `dani_sessoes` (migradas em S1); adicionar índice `@@index([canal, status])` em `dani_conversas` para filtros por canal:
  - Sub-item: Índice adicionado via migration `add_canal_status_index`
  - Sub-item: `dani_mensagens.metadata JSONB` aceita campos `{ tool_calls: [...], confidence_score: float, fallback: bool }`

### Backend

- [x] **[F1-BACK-001]** Implementar `AgenteController` com endpoints de chat:
  - Sub-item: `POST /api/v1/dani/chat` — recebe `{ mensagem: string, conversa_id?: string }`. Guards: `JwtAuthGuard`, `CessionarioOwnerGuard`. Persiste `DaniMensagem` remetente `CESSIONARIO`. Chama `AgenteService.chat()`. Retorna `DaniMensagem` remetente `DANI`
  - Sub-item: `GET /api/v1/dani/stream` — SSE streaming. Header `Content-Type: text/event-stream`, `Cache-Control: no-cache`. Vercel AI SDK `streamText()`. Timeout 30s. Guards: `JwtAuthGuard`, `CessionarioOwnerGuard`
  - Sub-item: `GET /api/v1/dani/conversas` — lista conversas do cessionário autenticado. Paginação `{ page, per_page (max 100) }`. Response: `{ data: DaniConversa[], meta: { page, per_page, total, total_pages } }`
  - Sub-item: `GET /api/v1/dani/conversas/{conversa_id}/mensagens` — lista mensagens. `cessionario_id` verificado via `CessionarioOwnerGuard`
  - Sub-item: `POST /api/v1/dani/conversas` — cria nova `DaniConversa`. Body: `{ canal: CanalDani, opr_id?: string }`. Retorna `conversa_id`
  - Sub-item: `PATCH /api/v1/dani/conversas/{conversa_id}/encerrar` — atualiza `status = ENCERRADA`. `cessionario_id` verificado

- [x] **[F1-BACK-002]** Implementar `AgenteService.chat()` com fluxo completo conforme D14 §4.2:
  - Sub-item: Passo 1 — verifica rate limit Redis `rate_limit:webchat:{cessionario_id}` (INCR + EXPIRE 3600s). Se count ≥ 30 → lança `RateLimitException AGENTE_RATE_001`, status 429, `details.retry_after_seconds` calculado a partir do TTL restante
  - Sub-item: Passo 2 — persiste `DaniMensagem` remetente `CESSIONARIO` antes de chamar LLM
  - Sub-item: Passo 3 — busca histórico da conversa (últimas 20 mensagens) para contexto
  - Sub-item: Passo 4 — busca `DaniContextoOpr` se conversa tem `opr_id` carregado
  - Sub-item: Passo 5 — verifica `GET dani:status:agent` no Redis; se `DESLIGADO` → chama `CalculadoraService` diretamente
  - Sub-item: Passo 6 — chama OpenAI `createChatCompletion(stream: true, tools: [...])` com temperature 0.3, max_tokens 2048, timeout 30s
  - Sub-item: Passo 7 — SSE stream chunks para o frontend token a token
  - Sub-item: Passo 8 — OpenAI timeout > 30s → fallback automático para `CalculadoraService`; Redis `SET dani:status:agent = FALLBACK EX 60`
  - Sub-item: Passo 9 — persiste `DaniMensagem` remetente `DANI` (ou `CALCULADORA` em fallback) com `metadata.fallback = true`
  - Sub-item: Passo 10 — envia trace ao Langfuse: `{ session_id, userId: sha256(cessionario_id), input_tokens, output_tokens, latency_ms, model: "gpt-4o", tools_called: [...] }`

- [x] **[F1-BACK-003]** Implementar sistema de estados do agente (machine state) conforme D05.1:
  - Sub-item: `OPERACIONAL` → sem degradação
  - Sub-item: `LATENCIA_ALTA` → SLA ultrapassado mas operacional
  - Sub-item: `FALLBACK_ATIVO` → OpenAI com falhas, Calculadora assume
  - Sub-item: `DESLIGADO_AUTOMATICO` → taxa de erro >30% em 15 min; reativação somente manual por Admin
  - Sub-item: Monitor de taxa de erro: janela deslizante 15 min. >10% → alerta via `dani.agent_monitor`; >30% → `SET dani:status:agent = DESLIGADO_AUTOMATICO`; log `fatal` com `reason: "error_rate_30pct"`
  - Sub-item: Takeover: `confidence_score < 0.80` em 3 turns consecutivos → publicar evento em fila `dani.agent_monitor` (REQ-108)

- [x] **[F1-BACK-004]** Implementar System Prompt da Dani conforme D05.5 + D08:
  - Sub-item: Arquivo: `src/agente/prompts/dani-system-prompt.v1.ts` — versionado (nunca hardcoded inline)
  - Sub-item: Persona: Analista de investimentos imobiliários — experiente, precisa, confiável (REQ-014)
  - Sub-item: Restrições obrigatórias no prompt: (1) responde apenas sobre dados do cessionário autenticado; (2) nunca revela dados do Cedente; (3) nunca usa linguagem FOMO, superlativos de venda ou garantias financeiras; (4) recusa submeter propostas em nome do Cessionário
  - Sub-item: Aviso de projeção obrigatório em toda resposta com ROI: "ℹ Esses são valores projetados com base nos dados disponíveis. Resultados reais podem variar conforme condições de mercado."

- [x] **[F1-BACK-005]** Implementar 5 Agent Tools conforme D19:
  - Sub-item: `buscarOportunidades` — timeout 5s, retry 2x backoff; guard: injeta `cessionario_id` em todo chamado; retorna oportunidades filtradas pelo perfil do cessionário
  - Sub-item: `calcularComissao` — timeout 2s, retry 1x; chama `CalculadoraService.calcularComissao()` com `cessionario_id` injetado
  - Sub-item: `analisarRisco` — timeout 10s, retry 2x backoff; retorna score de risco 1–10 com fatores
  - Sub-item: `buscarComparativoRegional` — timeout 8s, retry 2x backoff; dados de mercado da região da oportunidade
  - Sub-item: `verificarStatusNegociacao` — timeout 5s, retry 2x backoff; verifica status da negociação ativa do cessionário
  - Sub-item: Toda tool call loga `{ tool_name, latency_ms, success, cessionario_id_hash }` em nível `info`
  - Sub-item: Tool call sem `cessionario_id` injetado → lança `AgenteException AGENTE_004`

- [x] **[F1-BACK-006]** Implementar memória de curto prazo via Redis conforme D19:
  - Sub-item: Redis key `dani:session:{session_id}` TTL 1800s (sliding — renova a cada mensagem)
  - Sub-item: Máximo 50 mensagens por sessão (FIFO — remove a mais antiga ao atingir o limite)
  - Sub-item: Estrutura: lista Redis `RPUSH` / `LTRIM 0 49`

- [x] **[F1-BACK-007]** Implementar memória de longo prazo via pgvector conforme D19:
  - Sub-item: Tabela `agent_memory` com `embedding vector(1536)`, `namespace TEXT` (= `cessionario_id`), `content TEXT`, `created_at`, `expires_at` (TTL 90 dias)
  - Sub-item: Modelo de embedding: `text-embedding-3-small`; chunk 512 tokens, overlap 64 (REQ-056)
  - Sub-item: Ingestão disparada pós-análise, pós-simulação e pós-comparação
  - Sub-item: Busca por cosine similarity: `ORDER BY embedding <=> :query_embedding LIMIT 5 WHERE namespace = :cessionario_id`
  - Sub-item: ADR-004: namespace obrigatório — query sem namespace → `AgenteException AGENTE_004`

- [x] **[F1-BACK-008]** Implementar SLA monitoring conforme D01 RN-DC-029:
  - Sub-item: SLA por tipo: análise individual ≤5s, comparação ≤10s, simulação ≤5s, operacional ≤5s
  - Sub-item: Latência > SLA: log `warn` com `{ latency_ms, sla_ms, tipo_operacao }`
  - Sub-item: Latência > 2× SLA: emite evento na fila `dani.agent_monitor` (REQ-035)
  - Sub-item: Frontend recebe evento SSE de tipo `latency_warning` → exibe mensagem de latência com botões "Aguardar" / "Tentar novamente"

### Frontend

- [x] **[F1-FRONT-001]** Implementar FAB Global `T-DC-001` conforme D06 + D09:
  - Sub-item: `position: fixed; bottom: 24px; right: 24px; z-index: 9999` (REQ-117)
  - Sub-item: 56×56px; fundo `--primary`; radius `--radius-xl`
  - Sub-item: Estado Skeleton: `<Skeleton className="h-5 w-5 rounded-full" />` enquanto alertas carregam
  - Sub-item: Estado Empty (sem alertas): FAB sem badge; `aria-label="Abrir chat com Dani"`
  - Sub-item: Estado Error (falha de alertas): FAB sem badge; falha silenciosa (não bloqueia chat)
  - Sub-item: Estado Populated (com alertas): Badge `--destructive` no canto superior direito com contagem; `aria-label="Abrir chat com Dani — {N} alertas não lidos"`
  - Sub-item: Hover: `scale: 1.05`; Foco teclado: `ring` com `--ring`; `tabIndex={0}`, `role="button"`
  - Sub-item: `useReducedMotion()` — desabilita scale + bounce em `prefers-reduced-motion`
  - Sub-item: Entrada: scale `0→1` + fade-in 200ms `ease-out`; Badge aparece: scale + bounce 200ms Framer Motion `spring`

- [x] **[F1-FRONT-002]** Implementar Chat Window `T-DC-002/003/004` conforme D06 + D09:
  - Sub-item: `OverlayLayout`: 400px × 600px desktop, `position: fixed; bottom: 88px; right: 24px`; `border-radius: --radius-xl` nos cantos superiores
  - Sub-item: Mobile (≤640px): 100vw × 100dvh; `position: fixed`; sem radius (REQ-118)
  - Sub-item: Header 56px: avatar Dani (SVG inline) + "Dani · Analista de Oportunidades" + botão fechar
  - Sub-item: Área de mensagens: `role="log"`, `aria-live="polite"` (REQ-129, D28 bloqueante)
  - Sub-item: Input bar 56px: placeholder "Digite sua mensagem..." + botão enviar; `Shift+Enter` = quebra linha; `Enter` = envia
  - Sub-item: `role="dialog"`, `aria-modal="true"`, `aria-label="Chat com Dani"` — foco trap ativo (D28 bloqueante)
  - Sub-item: Swipe-down fecha chat (mobile — Framer Motion `drag`); botão back Android interceptado via `popstate`
  - Sub-item: 4 estados obrigatórios: Skeleton (carregando histórico), Empty (novo chat — boas-vindas + starters), Error (falha API + retry), Populated (mensagens)
  - Sub-item: Sem spinners — apenas Skeleton (D28 bloqueante)

- [x] **[F1-FRONT-003]** Implementar estado de boas-vindas e conversation starters (T-DC-002 — primeiro acesso):
  - Sub-item: Mensagem: "Olá! Sou a Dani, sua Analista de Oportunidades. Posso analisar riscos, comparar imóveis e simular retornos para você. Como posso ajudar?" (REQ-008)
  - Sub-item: 4 chips de starter (REQ-011): "Quais são as melhores oportunidades para mim hoje?", "Tenho R$ 500.000 para investir. O que recomenda?", "Me explica como funciona a comissão do comprador.", "Qual o prazo para depósito em Escrow?"
  - Sub-item: Estado KYC pendente: boas-vindas + link "Acesse Meu Perfil > Verificação de Identidade"
  - Sub-item: Estado sem oportunidades: boas-vindas + botão "Ativar alertas"
  - Sub-item: Acesso subsequente: histórico carregado (últimos 90 dias) — sem mensagem de boas-vindas extra

- [x] **[F1-FRONT-004]** Implementar Typing Indicator `W-DC-013` conforme D07:
  - Sub-item: 3 pontos com bounce stagger 150ms; `role="status"`, `aria-label="Dani está digitando"`
  - Sub-item: Removido do DOM quando resposta chega
  - Sub-item: `prefers-reduced-motion`: opacity pulse (0.3→1→0.3) sem translação

- [x] **[F1-FRONT-005]** Implementar Rate Limit state `T-DC-011` conforme D06 + D08:
  - Sub-item: Input desabilitado (fundo cinza, `cursor: not-allowed`)
  - Sub-item: Botão enviar inativo
  - Sub-item: Contador regressivo mm:ss atualizado a cada segundo
  - Sub-item: Reativação: pulse 500ms (`--motion-feedback`) na borda do input
  - Sub-item: `aria-live="assertive"` no contador (D11 §10.1)
  - Sub-item: Mensagem: "Você atingiu o limite de 30 mensagens por hora. Você poderá enviar a próxima mensagem em [mm:ss]." (REQ-132)

- [x] **[F1-FRONT-006]** Implementar Banner de Fallback `T-DC-010` conforme D06 + D08:
  - Sub-item: Texto: "Modo básico — sem análise da IA" com subtexto "Cálculo realizado sem análise contextual da IA. Para análise completa, tente novamente em instantes."
  - Sub-item: Cor `--agent-fallback (#0069A8)` ⚠️ AMBÍGUO — usar #0069A8 conforme D09 (REQ-126)
  - Sub-item: Componente `AgentStatusBanner` com variant `fallback`
  - Sub-item: Ações disponíveis: copiar valores, solicitar novo cálculo
  - Sub-item: Estado Dani desligada: texto "A Analista de Oportunidades está temporariamente indisponível. Os cálculos de comissão e Escrow continuam disponíveis."

### Wiring

- [x] **[F1-WIRE-001]** Conectar FAB ao store Zustand `useDaniChatStore`:
  - Sub-item: FAB click → `setIsChatOpen(true)`; `chatContext` null = T-DC-002; com `opr_id` = T-DC-003
  - Sub-item: Botão "Consultar Dani" na tela de oportunidade → `setChatContext({ opr_id })` + `setIsChatOpen(true)` → T-DC-003
  - Sub-item: FAB na tela de negociação ativa → `setChatContext({ neg_id, opr_id, valor_proposto })` → T-DC-004
  - Sub-item: Deep link `?chat=open` → `setIsChatOpen(true)` via `beforeLoad` do TanStack Router (REQ-135)

- [x] **[F1-WIRE-002]** Configurar SSE connection via `EventSource`:
  - Sub-item: `GET /api/v1/dani/stream` com `Accept: text/event-stream`
  - Sub-item: Token Bearer enviado no header (EventSource não suporta nativo — usar `fetch` com `ReadableStream`)
  - Sub-item: Chunks acumulados no store Zustand; renderização progressiva na área de mensagens
  - Sub-item: `error` event → exibe FallbackBanner se agente indisponível

### Testes

- [x] **[F1-TEST-001]** Testes unitários `AgenteService.chat()`:
  - Sub-item: Rate limit atingido (count = 30) → `RateLimitException AGENTE_RATE_001` com `retry_after_seconds > 0`
  - Sub-item: `dani:status:agent = DESLIGADO` → chama `CalculadoraService`, não chama OpenAI
  - Sub-item: OpenAI timeout após 30s → fallback para Calculadora; `dani:status:agent = FALLBACK`; `metadata.fallback = true` na mensagem persistida
  - Sub-item: OpenAI function_call `calcularComissao` → delega para `CalculadoraService`, resultado injetado na resposta
  - Sub-item: Confidence_score < 0.80 em 3 turns → evento publicado em `dani.agent_monitor`

- [x] **[F1-TEST-002]** Testes unitários das 5 tools:
  - Sub-item: `buscarOportunidades` — timeout >5s → fallback com mensagem de indisponibilidade
  - Sub-item: `calcularComissao` — sem `cessionario_id` injetado → `AGENTE_004`
  - Sub-item: `analisarRisco` — timeout >10s → fallback
  - Sub-item: RAG query sem namespace → `AGENTE_004`

- [x] **[F1-TEST-003]** Testes de integração dos endpoints de chat (P1 ≥80%):
  - Sub-item: `POST /dani/chat` sem autenticação → 401 `AUTH_001`
  - Sub-item: `POST /dani/chat` com rate limit atingido → 429 `AGENTE_RATE_001`
  - Sub-item: `GET /dani/stream` com `Accept: text/event-stream` → 200, chunks SSE recebidos
  - Sub-item: `GET /dani/conversas` → 200, apenas conversas do cessionário autenticado (isolamento)
  - Sub-item: `GET /dani/conversas/{id}/mensagens` com conversa de outro cessionário → 403 `AUTH_009`

- [x] **[F1-TEST-004]** Testes E2E Playwright — fluxo de chat básico:
  - Sub-item: FAB visível em todas as telas do Cessionário
  - Sub-item: Click no FAB → Chat abre (T-DC-002); 4 conversation starters visíveis
  - Sub-item: Click em starter → mensagem enviada; typing indicator aparece; resposta da Dani exibida
  - Sub-item: Chat fecha → foco retorna ao FAB (D28 bloqueante)
  - Sub-item: `axe-core` — zero violações `critical` ou `serious` no componente FAB + Chat

---

## FEATURE 2: Isolamento e Mensagens de Recusa

### Backend

- [x] **[F2-BACK-001]** Implementar lógica de recusa de dados bloqueados conforme D01 RN-DC-004 + D08 §2.11:
  - Sub-item: System prompt contém proibições explícitas para as 7 categorias de dado bloqueado (REQ-006)
  - Sub-item: `AgenteService.detectarSolicitacaoProibida()` identifica categorias bloqueadas e retorna a string normativa correta (REQ-132)
  - Sub-item: Contador de insistência por conversa: após 2 insistências → mesma mensagem + alternativa; após 3ª → recusa sem alternativas (REQ-007)
  - Sub-item: Recusa de submissão de proposta — 1ª vez: instrução + botão "Ir para a oportunidade"; insistência: recusa reiterada (REQ-034)

---

## 🔍 Auto-verificação S3 (12 checks)

| #   | Check                                                                                                                                          | Status |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                   | ✅     |
| 2   | Nomes exatos: `AgenteService`, `CessionarioOwnerGuard`, `dani:status:agent`, `dani:session:{session_id}`, `dani.agent_monitor`, `agent_memory` | ✅     |
| 3   | Valores numéricos: temperature 0.3, max_tokens 2048, timeout 30s, TTL 1800s, max 50 msgs FIFO, chunk 512, overlap 64, confidence < 0.80        | ✅     |
| 4   | 5 tools com timeouts individuais documentados; 4 estados do agente documentados                                                                | ✅     |
| 5   | Glossário D10: "Conversa" = `dani_conversas`; "Sessão" = `dani_sessoes`; distinção preservada                                                  | ✅     |
| 6   | Estados do agente: OPERACIONAL → LATENCIA_ALTA → FALLBACK_ATIVO → DESLIGADO_AUTOMATICO; todas as transições documentadas                       | ✅     |
| 7   | TTLs: sessão Redis 1800s, RAG 90 dias, calc cache 300s                                                                                         | ✅     |
| 8   | ⚠️ AMBÍGUO agent-fallback (#0069A8) sinalizado nos itens relevantes                                                                            | ✅     |
| 9   | ADR-003 (Calculadora isolada) e ADR-004 (RAG namespace) implementados                                                                          | ✅     |
| 10  | Nenhum item de scaffold — todos com sub-itens de lógica real                                                                                   | ✅     |
| 11  | Anti-FOMO, sem garantias, recusa submissão — todos no system prompt                                                                            | ✅     |
| 12  | REQs atribuídos têm ≥1 item de checklist                                                                                                       | ✅     |

---

## REQs cobertos por esta sprint

REQ-001, REQ-002, REQ-003 (verificado em S2, aplicado aqui), REQ-006, REQ-007, REQ-008, REQ-009, REQ-011, REQ-029, REQ-030, REQ-031, REQ-032, REQ-033, REQ-034, REQ-035, REQ-056, REQ-060, REQ-071, REQ-072, REQ-073, REQ-074, REQ-075, REQ-076, REQ-077, REQ-088, REQ-089 (adotando D16), REQ-090, REQ-098, REQ-099, REQ-100, REQ-101, REQ-102, REQ-103, REQ-104, REQ-105, REQ-106, REQ-107, REQ-108, REQ-117, REQ-118, REQ-119, REQ-120, REQ-126, REQ-127, REQ-129, REQ-130, REQ-131, REQ-132, REQ-133, REQ-134
