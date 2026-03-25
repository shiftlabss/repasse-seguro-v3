# S10 — Sprint Final: Auditoria e Correções

| Campo | Valor |
|---|---|
| **Sprint** | S10 |
| **Nome** | Sprint Final — Auditoria |
| **Tipo** | Sprint Final (obrigatória) |
| **Docs Fonte** | Todos os docs 01.1-01.5, 02, 12-14, 16-17, 19, 22, 24, 27, 29 |
| **Pré-requisito** | S1 a S9 concluídas e verificadas |
| **Status** | ⬜ Pendente |

---

## 🎯 Objetivo

Auditoria cruzada completa de cobertura (185 REQs vs. checklists S1-S9), identificação e correção de gaps P0/P1, validação adversarial dos docs fonte para itens omitidos, e geração da Matriz de Cobertura final.

---

## 📋 CHECK 1 — Cobertura 100% do Registro Mestre

### Matriz de Cobertura — S1 a S9

| Sprint | REQs atribuídos | Itens no checklist | Cobertura |
|---|---|---|---|
| S1 — Fundação | REQ-007, REQ-114 a REQ-158, REQ-168 a REQ-173, REQ-178 | 27 | ⬜ |
| S2 — Auth | REQ-004 a REQ-006, REQ-022 a REQ-023, REQ-028, REQ-071 a REQ-072, REQ-094, REQ-111, REQ-135 | 14 | ⬜ |
| S3a — Agente Core P1 | REQ-008 a REQ-013, REQ-024 a REQ-027, REQ-029 a REQ-032, REQ-036, REQ-062 a REQ-067, REQ-073, REQ-075 a REQ-076 | 14 | ⬜ |
| S3b — Agente Core P2 | REQ-014 a REQ-021, REQ-038 a REQ-039, REQ-093, REQ-112 a REQ-113, REQ-120 a REQ-134, REQ-159 a REQ-163, REQ-174 a REQ-177 | 37 | ⬜ |
| S4 — Calculadora | REQ-022 a REQ-023, REQ-033 a REQ-035, REQ-068 a REQ-070, REQ-074, REQ-095 a REQ-097 | 12 | ⬜ |
| S5 — Comparação/Simulação | REQ-040 a REQ-058 | 19 | ⬜ |
| S6 — Admin/Supervisão | REQ-059 a REQ-061, REQ-077 a REQ-091, REQ-098 a REQ-103 | 24 | ⬜ |
| S7 — WhatsApp/Notificações | REQ-001 a REQ-003, REQ-104 a REQ-110, REQ-136 a REQ-158, REQ-164 a REQ-167 | 37 | ⬜ |
| S8 — Qualidade | REQ-178 a REQ-185 | 8 | ⬜ |
| S9 — Go-Live | REQ-115 a REQ-119 | 5 | ⬜ |
| **TOTAL** | **185** | **197** | **⬜ 100%?** |

### Verificações de cobertura
- [ ] Cruzar CADA REQ-001 a REQ-185 do `registro-mestre.md` contra os checklists S1-S9: identificar REQs sem nenhum item correspondente
- [ ] Para cada REQ sem cobertura: criar item corretivo nesta sprint (seção "Correções P0/P1" abaixo)
- [ ] Para REQs com múltiplas sprints: confirmar que não há duplicação contraditória entre sprints
- [ ] Verificar cobertura das **máquinas de estado**: `conversation_status`, `whatsapp_binding_status`, `otp_status`, `agent_status` — todas as transições têm pelo menos 1 item de checklist

---

## 📋 CHECK 2 — Anti-Scaffold R10: Verificação de Itens Superficiais

### Auditoria por sprint — verificar itens que permitem scaffold vazio

- [ ] **S1:** revisar itens 1-27 — confirmar que cada migration tem: coluna por coluna + índices + testes de INSERT/SELECT; confirmar que `PrismaModule` tem middleware de soft delete testado; confirmar que `RabbitMQModule` tem 4 exchanges e 6 filas (não genérico)
- [ ] **S2:** revisar itens 1-14 — confirmar que `JwtStrategy` tem validação de role CEDENTE; confirmar que rate limit tem janela deslizante real (não apenas guard vazio); confirmar que `LgpdRetentionJob` tem query SQL com `INTERVAL '90 days'`
- [ ] **S3a:** revisar itens 1-14 — confirmar que `AgentState` tem 9 campos documentados por nome; confirmar que cada tool tem timeout específico (5s/10s/15s); confirmar que `CACHE` node tem TTL 24h/1h separados
- [ ] **S3b:** revisar itens 1-12 — confirmar que system prompt tem 4 blocos com `userId` obrigatório; confirmar que SSE tem evento `[DONE]`; confirmar que PERSIST é non-blocking
- [ ] **S4:** revisar itens 1-8 — confirmar que `calculateDelta` tem fórmula explícita; confirmar que fallback INSERT em `agent_status_logs` com `status: 'degraded'`
- [ ] **S5:** revisar itens 1-11 — confirmar que `weightedScore` tem fórmula documentada; confirmar que `simulateCounterproposal` tem `feasibility`; confirmar que SLA ≤10s está em item de teste
- [ ] **S6:** revisar itens 1-15 — confirmar que takeover tem Redis mutex `SET NX EX 3600`; confirmar que 6 alertas têm tipos enumerados; confirmar que 5 KPIs têm SQL implícito
- [ ] **S7:** revisar itens 1-15 — confirmar que OTP usa `bcrypt.hash`; confirmar que webhook tem `HMAC-SHA256`; confirmar que re-verificação tem `INTERVAL '30 days'`
- [ ] **S8:** revisar itens 1-15 — confirmar que golden dataset tem 50 exemplos por 6 categorias; confirmar que E2E tem assertions de banco; confirmar que CI gate falha com cobertura abaixo
- [ ] **S9:** revisar itens 1-9 — confirmar que rollback tem 5 passos com tempos; confirmar que smoke suite tem exatamente 5 testes em < 60s

---

## 📋 CHECK 3 — Fullstack Vertical Slice por Feature

Para cada sprint com Template B, verificar que cada feature tem vertical slice completo (Banco → Backend → Wiring → Testes):

- [ ] **S3a — Feature 1 (LangGraph):** Banco (llm_cache_entries, document_embeddings) ✅ | Backend (11 nós) ✅ | Wiring (exports) ✅ | Testes (suite por nó) ✅
- [ ] **S3a — Feature 2 (Tools):** Banco (acesso via Prisma) ✅ | Backend (5 tools) ✅ | Wiring (DI providers) ✅ | Testes (por tool) ✅
- [ ] **S3b — Feature 3 (Nós GENERATE/STREAM/PERSIST/LANGFUSE):** Backend ✅ | Wiring (AiModule imports) ✅ | Testes ✅ — verificar que PERSIST tem transação com `chat_messages` + `ai_interactions`
- [ ] **S3b — Feature 4 (Endpoints API):** Backend (3 endpoints) ✅ | Wiring (Controller registrado em AiModule) ✅ | Testes (integration) ✅
- [ ] **S4 — Feature 1 (Cálculo):** Backend (CalculadoraService) ✅ | Testes (unitários) ✅ — sem banco próprio (usa banco do agente via Prisma)
- [ ] **S5 — Feature 3 (Top 3 + Suporte):** Backend (endpoints) ✅ | Testes ✅ — verificar que `SuporteOperacionalService` não tem chamadas de escrita ao banco nesta sprint
- [ ] **S6 — Feature 2 (Alertas):** Banco (notification_events) ✅ | Backend (AlertService) ✅ | Wiring (RabbitMQ consumer) ✅ | Testes ✅
- [ ] **S7 — Feature 1 (OTP):** Banco (otp_attempts, whatsapp_bindings) ✅ | Backend (WhatsappService) ✅ | Wiring (EvolutionAPI mock) ✅ | Testes ✅ | E2E ✅

---

## 📋 CHECK 4 — Alinhamento Sprint ↔ Docs Fonte

Auditoria adversarial: reler cada doc fonte buscando ativamente gaps omitidos:

- [ ] **Doc 01.1 — Fundação e Acessos:** verificar RN-001 (identidade), RN-002 (RBAC 3 roles), RN-003 (Cedente bloqueado), RN-004 (3-layer isolation), RN-005 (onboarding — consentimento LGPD antes de usar), RN-006 (sessão), RN-007 (histórico 90 dias), RN-008 (soft delete voluntário), RN-009 (webchat 24/7), RN-010 (glossário) → confirmar que TODOS têm item de checklist
- [ ] **Doc 01.2 — Módulos Core:** verificar RN-011 (análise Δ+comissão+ROI+risco), RN-012 (tool calls sequenciais), RN-013 (max 5 comparação), RN-014 (simulação proposta/contraproposta), RN-015 (portfólio), RN-016 (cenários ±20%), RN-017 (Top 3), RN-018 (score de risco — verificar se há item específico), RN-019 (ROI anualizado), RN-020 (máquina de estados do agente), RN-021 (SLA ≤5s/≤10s) → confirmar que TODOS têm item
- [ ] **Doc 01.3 — Módulos Operação:** verificar RN-022 (KYC/Escrow/ZapSign status), RN-023 (fallback Calculadora), RN-024 (rate limit 30/h webchat), RN-025 (sliding window), RN-026 (fluxo principal de conversa), RN-027 (comportamento em latência alta — mensagem "Aguarde"), RN-028 (latência máxima — sem loop infinito), RN-029 (WhatsApp rate limit 20/h) → confirmar que TODOS têm item
- [ ] **Doc 01.4 — Admin e Config:** verificar RN-030 (painel Admin), RN-031 (6 alertas), RN-032 (takeover), RN-033 (mutex), RN-034 (dashboard 5 KPIs), RN-035 (threshold 50%-95%), RN-036 (padrão 80%), RN-037 (webchat 24/7 — critérios de launch), RN-038 (agente offline → fallback), RN-039 (launch readiness) → confirmar que TODOS têm item
- [ ] **Doc 01.5 — Integrações e LGPD:** verificar RN-040 (WhatsApp Fase 2), RN-041 (EvolutionAPI), RN-042 (OTP 2 etapas), RN-043 (SMS Etapa 1), RN-044 (WhatsApp Etapa 2), RN-045 (vinculação ativa após OTP), RN-046 (re-verificação 30 dias), RN-047 (desvincular PARAR), RN-048 (WhatsApp rate 20/h), RN-049 (notificações proativas), RN-050 (3 tipos), RN-051 (LGPD consent), RN-052 (exclusão dados), RN-053 (anonimização para métricas) → confirmar que TODOS têm item

### Gap gaps específicos a verificar:

- [ ] **RN-018 (Score de Risco):** verificar se há item de checklist cobrindo score de risco — se não há implementação prevista (score de risco não é calculado explicitamente nos tools), marcar ⚠️ AMBÍGUO com referência ao Doc 01.2
- [ ] **RN-027 (mensagem "Aguarde" em latência alta):** verificar se há item em S3b/S3a cobrindo comportamento de UX quando latência > threshold — se ausente, criar item corretivo
- [ ] **RN-038 (agente offline → fallback):** verificar se há item cobrindo comportamento quando `agent_status != 'online'` — resposta ao usuário informando indisponibilidade + fallback Calculadora
- [ ] **RN-053 (anonimização para métricas):** verificar se há item cobrindo anonimização de PII em métricas Langfuse (não enviar conteúdo de mensagens) — se ausente, criar item
- [ ] **ADR-001 a ADR-005 (Doc 14):** verificar que cada ADR tem pelo menos 1 item de checklist nas sprints relevantes: ADR-001 (Railway sem cold start → S9), ADR-002 (RabbitMQ para filas → S1), ADR-003 (Redis para cache → S1), ADR-004 (Prisma como ORM → S1), ADR-005 (LangGraph para fluxo stateful → S3a)
- [ ] **Doc 17 — 8 integrações por criticidade:** verificar que P0 (OpenAI, Supabase, Redis), P1 (EvolutionAPI, RabbitMQ, Langfuse) e P2 (Sentry, Railway) têm itens nas sprints corretas — P0 em S1/S3, P1 em S1/S7, P2 em S1/S9
- [ ] **Doc 22 — 18 variáveis de ambiente:** verificar que todas as 18 vars têm itens em S1; verificar que `FEATURE_WHATSAPP_MOCK=true` tem item específico em S1 e S7

---

## 📋 CHECK 5 — Verificação de Valores Numéricos Críticos

- [ ] Verificar que os valores abaixo aparecem EXATAMENTE como nos docs em pelo menos 1 item de checklist:
  - `temperature: 0.1` (GPT-4o — S3b item 1) ✅/❌
  - `max_tokens: 4096` (GPT-4o — S3b item 1) ✅/❌
  - `parallel_tool_calls: false` (S3a item 12 + S3b item 1) ✅/❌
  - `similarity_threshold: 0.78` (RAG — S3a item 6) ✅/❌
  - `semantic_cache_threshold: 0.92` (cache — S3a item 4) ✅/❌
  - TTL exact cache `24 hours` (S3a item 4) ✅/❌
  - TTL semantic cache `1 hour` (S3a item 4) ✅/❌
  - TTL Redis sessão `30 minutes` (S3b item 10) ✅/❌
  - Max mensagens Redis `20` (S3b item 10) ✅/❌
  - Retenção LGPD `90 days` (S2 item 2 + S3b item 11) ✅/❌
  - Rate limit webchat `30 msg/h` (S2 item 10) ✅/❌
  - Rate limit WhatsApp `20 msg/h` (S7 item 9) ✅/❌
  - OTP TTL `10 minutos` (S7 item 1) ✅/❌
  - Re-verificação WhatsApp `30 dias` (S7 item 4) ✅/❌
  - Threshold padrão `0.80` (S6 item 10 + S1 seed item 11) ✅/❌
  - Threshold range `0.50-0.95` (S6 item 10) ✅/❌
  - Max oportunidades comparação `5` (S5 item 1) ✅/❌
  - SLA análise `≤5s`, comparação `≤10s` (S3b item 6 + S5 item 2) ✅/❌
  - Rollback `<3min` (S9 item 7) ✅/❌
  - Golden dataset `50 exemplos` (S8 item 11) ✅/❌
  - Seed `9 configurações` (S1 item 11) ✅/❌
  - `13 tabelas`, `17 enums` (S1 itens 1-10 + 25) ✅/❌
  - `1536 dimensões` embeddings (S1 item 5) ✅/❌
  - Chunking `512-1024 tokens` (S3a item 6) ✅/❌

---

## 📋 CHECK 6 — Verificação de Máquinas de Estado

- [ ] **`conversation_status`** — confirmar que as 3 transições têm itens:
  - `active → archived` (S2 item 8) ✅/❌
  - `archived → active` (S2 item 8) ✅/❌
  - `* → deleted` apenas via DELETE endpoint (S2 item 9) ✅/❌
  - `deleted → *` bloqueado via PATCH (S2 item 8) ✅/❌

- [ ] **`whatsapp_binding_status`** — confirmar que as transições têm itens:
  - `pending → active` via OTP confirmado (S7 item 3) ✅/❌
  - `active → inactive` via PARAR ou job (S7 itens 4, 8) ✅/❌
  - Tentativa de vincular número já `active` → `ConflictException` (S7 item 1) ✅/❌

- [ ] **`otp_status`** — confirmar:
  - `pending → verified` (S7 itens 2, 3) ✅/❌
  - `pending → failed` (OTP incorreto — S7 item 2) ✅/❌
  - `pending → expired` (expiração automática — S7 item 2) ✅/❌

- [ ] **`agent_status`** — confirmar:
  - Transições livres para ADMIN (S6 item 12) ✅/❌
  - Status persiste em `agent_status_logs` (append-only — S6 item 12) ✅/❌

---

## 🔧 Correções P0/P1 Identificadas

*Esta seção é preenchida durante a execução da Sprint Final com base nas verificações acima.*

### P0 — Bloquear deploy (corrigir antes de encerrar Sprint Final)

*(Preencher com gaps encontrados nos Checks 1-6 acima)*

### P1 — Corrigir antes do Go-Live

*(Preencher com gaps encontrados nos Checks 1-6 acima)*

---

## ✔️ Auto-Verificação S10 (7 Checks da Sprint Final)

| # | Check | Status |
|---|---|---|
| 1 | Matriz de cobertura: 185/185 REQs com ≥1 item de checklist | ⬜ |
| 2 | Anti-Scaffold R10: zero itens superficiais encontrados em S1-S9 | ⬜ |
| 3 | Fullstack vertical slice: todas as features de S3a-S7 têm Banco+Backend+Wiring+Testes | ⬜ |
| 4 | Alinhamento Sprint↔Docs: RN-001 a RN-053 + ADRs + 18 env vars cobertos | ⬜ |
| 5 | Valores numéricos: 25 valores críticos verificados com ✅ | ⬜ |
| 6 | Máquinas de estado: 4 entidades com todas as transições documentadas | ⬜ |
| 7 | Zero P0/P1 pendentes — GATE para Fase 4 | ⬜ |
