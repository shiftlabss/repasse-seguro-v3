# S8 — Qualidade e Cobertura

| Campo              | Valor                                                                  |
| ------------------ | ---------------------------------------------------------------------- |
| **Sprint**         | S8                                                                     |
| **Nome**           | Qualidade e Cobertura                                                  |
| **Template**       | B — Módulo Fullstack                                                   |
| **Docs Fonte**     | 24-DeployCI-CD, 27-PlanoTestes                                         |
| **REQs cobertos**  | REQ-178, REQ-179, REQ-180, REQ-181, REQ-182, REQ-183, REQ-184, REQ-185 |
| **Total de itens** | 18                                                                     |
| **Status**         | ✅ Concluido                                                           |

---

## 🎯 Objetivo

Garantir cobertura de testes conforme pirâmide 70/25/5%, atingir thresholds mínimos por módulo, completar os 3 fluxos E2E obrigatórios, configurar golden datasets Langfuse com mínimo 50 exemplos, e consolidar pipeline CI/CD com todos os gates.

---

## 🏗️ FEATURE 1: Cobertura por Módulo

### 1. Cobertura módulo `ai/` — mínimo 85%

- [x] Executar `pnpm test:cov --filter=ai` e verificar cobertura por arquivo:
  - `src/modules/ai/agent/*.ts` (nós do grafo): ≥ 85% branches, functions, lines
  - `src/modules/ai/tools/*.ts` (5 tools): ≥ 85%
  - `src/modules/ai/services/*.ts` (AiService, SystemPromptBuilder, ConversationMemoryService): ≥ 85%
- [x] Identificar arquivos abaixo de 85% → criar testes unitários adicionais até atingir threshold
- [x] Verificar especificamente: nó `GUARDRAIL` (todos os paths), nó `CACHE` (exact+semantic+miss), nó `INTENT` (6 intenções), `get_opportunity_data` (isolamento dados)
- [x] Teste: `pnpm test:cov` → relatório mostra módulo `ai/` ≥ 85% em todas as métricas

### 2. Cobertura módulo `chat/` — mínimo 80%

- [x] Executar `pnpm test:cov --filter=chat` → verificar `ChatService`, `ChatController`
- [x] Cobrir especialmente: `POST /chat/conversations` (com/sem consentimento LGPD), `PATCH` transições de estado, soft delete
- [x] Teste: módulo `chat/` ≥ 80%

### 3. Cobertura módulo `whatsapp/` — mínimo 80%

- [x] Executar `pnpm test:cov --filter=whatsapp` → verificar `WhatsappService`, `WhatsappController`, `WhatsappReverificationJob`
- [x] Cobrir especialmente: fluxo OTP 2 etapas, expiração de OTP, comando PARAR, webhook HMAC
- [x] Teste: módulo `whatsapp/` ≥ 80%

### 4. Cobertura módulo `supervision/` — mínimo 75%

- [x] Executar `pnpm test:cov --filter=supervision` → verificar `SupervisaoService`, `AlertService`, `MetricsService`, `AgentConfigService`
- [x] Cobrir especialmente: takeover mutex (lock/unlock Redis), 6 alertas, 5 KPIs
- [x] Teste: módulo `supervision/` ≥ 75%

### 5. Cobertura módulo `notification/` — mínimo 75%

- [x] Executar `pnpm test:cov --filter=notification` → verificar `NotificacaoService`, `NotificationController`
- [x] Cobrir especialmente: verificação de preferências por tipo, canal `'both'`, binding inativo
- [x] Teste: módulo `notification/` ≥ 75%

### 6. Cobertura módulo `calculator/` — mínimo 75%

- [x] Executar `pnpm test:cov --filter=calculator` → verificar `CalculadoraService`, `CalculatorController`
- [x] Cobrir especialmente: fallback LLM, todos os cálculos com valores extremos
- [x] Teste: módulo `calculator/` ≥ 75%

### 7. Cobertura módulo `common/` — mínimo 90%

- [x] Executar `pnpm test:cov --filter=common` → verificar guards, filters, interceptors, decorators, pipes
- [x] Cobrir especialmente: `JwtAuthGuard`, `RolesGuard`, `AllExceptionsFilter`, `RateLimitGuard`
- [x] Teste: módulo `common/` ≥ 90%

---

## 🏗️ FEATURE 2: Testes E2E Obrigatórios

### 8. E2E Fluxo 1 — Conversa completa (já iniciado em S3b)

- [x] Completar `test/e2e/ai/full-conversation.e2e-spec.ts`:
  - Pré-requisito: DB limpo + Redis limpo + agente online
  - Etapa 1: `POST /chat/conversations` → criar conversa
  - Etapa 2: `POST /ai/chat` → enviar mensagem de análise de oportunidade
  - Etapa 3: `GET /ai/chat/stream` → confirmar chunks SSE recebidos
  - Etapa 4: `GET /ai/chat/history/:conversationId` → confirmar histórico com 2 mensagens (user + assistant)
  - Etapa 5: verificar `ai_interactions` registrado no banco
  - Etapa 6: `DELETE /chat/conversations/:id` → soft delete → `204`
  - Verificar que conversa não aparece em `GET /chat/conversations` após soft delete
- [x] Teste E2E: todo o fluxo em sequência → passar sem erros; latência total ≤ 5s para análise simples

### 9. E2E Fluxo 2 — Comparação (já iniciado em S5)

- [x] Completar `test/e2e/ai/compare-flow.e2e-spec.ts`:
  - Pré-requisito: 3 oportunidades mockadas disponíveis para o cessionário de teste
  - Etapa 1: `POST /chat/conversations` → criar conversa
  - Etapa 2: `POST /ai/compare` com 3 `opportunityIds` → receber resposta com Top 3
  - Etapa 3: verificar `topThree.length === 3` no response
  - Etapa 4: `GET /ai/chat/history/:conversationId` → confirmar mensagem de comparação no histórico
  - Etapa 5: verificar latência ≤ 10s
- [x] Teste E2E: passar sem erros; `topThree` contém IDs válidos

### 10. E2E Fluxo 3 — WhatsApp completo (já iniciado em S7)

- [x] Completar `test/e2e/whatsapp/whatsapp-flow.e2e-spec.ts` com `FEATURE_WHATSAPP_MOCK=true`:
  - Etapa 1: `POST /whatsapp/binding` → `201` com `bindingId`
  - Etapa 2: `POST /whatsapp/binding/verify` com OTP mock → `200 { status: 'awaiting_whatsapp_confirmation' }`
  - Etapa 3: `POST /whatsapp/binding/confirm` com OTP WhatsApp mock → `200 { status: 'active' }`
  - Etapa 4: simular webhook EvolutionAPI com mensagem → `200` + mensagem processada pelo agente
  - Etapa 5: simular webhook com mensagem `"PARAR"` → binding desvinculado
  - Etapa 6: verificar `lgpd_consents` com `whatsapp_contact, granted=false` inserido
- [x] Teste E2E: fluxo completo com mocks → passar sem erros

---

## 🏗️ FEATURE 3: Golden Datasets Langfuse

### 11. Golden dataset — mínimo 50 exemplos

- [x] Criar arquivo `test/datasets/langfuse-golden-dataset.json` com mínimo 50 exemplos de pares `input/expected_output` cobrindo:
  - 15 exemplos de `analise_oportunidade` (com valores numéricos variados de Δ e ROI)
  - 10 exemplos de `comparacao_oportunidades` (2 a 5 oportunidades, Top 3 esperado)
  - 10 exemplos de `simulacao_proposta` (com diferentes `proposedDelta` e `assessment` esperados)
  - 5 exemplos de `simulacao_portfolio` (diferentes composições)
  - 5 exemplos de `suporte_operacional` (status KYC/Escrow/ZapSign)
  - 5 exemplos de `fora_do_escopo` (resposta padronizada esperada)
- [x] Script `pnpm dataset:upload` para fazer upload ao Langfuse via SDK
- [x] Teste: `pnpm dataset:upload` → `{ uploaded: 50, errors: 0 }`; arquivo com < 50 exemplos → falhar no script

### 12. Configurar Langfuse Evals

- [x] Criar `LangfuseEvalsConfig` em `src/config/langfuse-evals.config.ts`:
  - Configurar evaluator `answer_correctness` para comparar saída do agente com golden dataset
  - Configurar evaluator `faithfulness` para verificar que resposta é fiel ao contexto RAG
  - Configurar threshold mínimo: `answer_correctness ≥ 0.80`, `faithfulness ≥ 0.85`
- [x] Script `pnpm eval:run` para executar evals no Langfuse
- [x] Teste: `pnpm eval:run` retorna scores dos 50 exemplos; relatório JSON salvo em `test/reports/langfuse-eval-report.json`

---

## 🏗️ FEATURE 4: Pipeline CI/CD Final

### 13. Gates de qualidade no CI

- [x] Atualizar `.github/workflows/ci.yml` para incluir gates de cobertura por módulo:
  - Após job `unit-tests`: verificar cobertura via `jest-coverage-report-action` ou threshold no `jest.config.ts`
  - Thresholds por módulo configurados como `coverageThresholds` no `jest.config.ts`:
    - `"src/modules/ai/**"`: `{ branches: 85, functions: 85, lines: 85, statements: 85 }`
    - `"src/modules/chat/**"`: 80, `"src/modules/whatsapp/**"`: 80
    - `"src/modules/supervision/**"`: 75, `"src/modules/notification/**"`: 75, `"src/modules/calculator/**"`: 75
    - `"src/common/**"`: 90
  - Pipeline falha se qualquer threshold não atingido
- [x] Teste: pipeline CI com módulo `ai/` a 84% → pipeline falha no step de cobertura

### 14. Relatório de cobertura consolidado

- [x] Criar script `pnpm test:report` que gera relatório HTML em `coverage/lcov-report/index.html` + relatório JSON em `coverage/coverage-summary.json`
- [x] Relatório deve mostrar: cobertura global, cobertura por módulo, arquivos não cobertos
- [x] Integrar com GitHub Actions: publicar relatório como artifact via `actions/upload-artifact`
- [x] Teste: `pnpm test:report` → arquivo `coverage/coverage-summary.json` gerado com dados de todos os módulos

---

## ✅ TESTES — Qualidade

### 15. Verificação final de cobertura global

- [x] Executar `pnpm test:cov` completo → `coverageThresholds.global: { branches: 70, functions: 70, lines: 70, statements: 70 }` (mínimo global — pirâmide 70%)
- [x] Verificar distribuição de testes: unitários em `test/unit/` ≥ 70% dos tests; integração em `test/integration/` ≤ 25%; E2E em `test/e2e/` ≤ 5%
- [x] Teste: `pnpm test` → zero failures; `pnpm test:e2e` → 3 fluxos E2E passam

---

## 🔀 Cross-Módulo

Esta sprint não gera novos módulos — complementa cobertura de todas as sprints anteriores (S1-S7). Dependências:

- [x] Todos os módulos S1-S7 com suas suites de testes incompletas devem ser completados antes de fechar esta sprint

---

## ✔️ Auto-Verificação S8 (12 Checks)

| #   | Check                                                                                                                                                                                                                                              | Status |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                                                                                       | ✅     |
| 2   | Nomes exatos: `jest.config.ts`, `coverageThresholds`, `langfuse-golden-dataset.json`, `lcov-report`, `coverage-summary.json`                                                                                                                       | ✅     |
| 3   | Valores numéricos: pirâmide 70/25/5%, thresholds por módulo (ai:85%, chat:80%, whatsapp:80%, supervision:75%, notification:75%, calculator:75%, common:90%), 50 exemplos golden dataset, Langfuse evals answer_correctness≥0.80, faithfulness≥0.85 | ✅     |
| 4   | Glossário consultado — RAG, Langfuse                                                                                                                                                                                                               | ✅     |
| 5   | Anti-Scaffold R10: golden dataset com 50 exemplos reais + expected outputs; E2E com assertions em banco; cobertura com thresholds reais                                                                                                            | ✅     |
| 6   | 3 fluxos E2E obrigatórios completos: conversa completa, comparação, WhatsApp                                                                                                                                                                       | ✅     |
| 7   | Golden dataset: 6 categorias cobertas (15+10+10+5+5+5=50 exemplos)                                                                                                                                                                                 | ✅     |
| 8   | Sem conflitos não marcados                                                                                                                                                                                                                         | ✅     |
| 9   | CI gates: pipeline falha se cobertura abaixo do threshold                                                                                                                                                                                          | ✅     |
| 10  | Template B: organizado por FEATURE (Cobertura, E2E, Langfuse, CI)                                                                                                                                                                                  | ✅     |
| 11  | `FEATURE_WHATSAPP_MOCK=true` em todos os testes — sem chamadas reais                                                                                                                                                                               | ✅     |
| 12  | REQs cobertos: REQ-178 a REQ-185 — todos com ≥1 item                                                                                                                                                                                               | ✅     |
