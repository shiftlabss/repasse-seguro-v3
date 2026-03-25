# S5 — Comparação e Simulação
<!-- Atualizado em 2026-03-24 pela A03 — 2 correções aplicadas (FINDING-013, FINDING-029) -->

| Campo | Valor |
|---|---|
| **Sprint** | S5 |
| **Nome** | Comparação e Simulação |
| **Template** | B — Módulo Fullstack |
| **Docs Fonte** | 01.2-RN-Core, 14-EspecTécnicas, 16-API, 19-CriaçãoAgentes |
| **REQs cobertos** | REQ-040, REQ-041, REQ-042, REQ-043, REQ-044, REQ-045, REQ-046, REQ-047, REQ-048, REQ-049, REQ-050, REQ-051, REQ-052, REQ-053, REQ-054, REQ-055, REQ-056, REQ-057, REQ-058 |
| **Total de itens** | 26 |
| **Status** | ✅ Concluído |

---

## 🎯 Objetivo

Implementar os módulos de Comparação de Oportunidades (máx. 5) e Simulação de Propostas/Contrapropostas, incluindo os endpoints REST e a lógica de Top 3 Recomendação, SLA ≤10s para comparação.

---

## 🏗️ FEATURE 1: Comparação de Oportunidades

### 1. `ComparacaoService.compareOpportunities` — comparação de até 5 oportunidades
- [x] Criar `ComparacaoService` com método `compareOpportunities(user: AuthenticatedUser, opportunityIds: string[]): ComparisonResult`:
  - Validar `opportunityIds.length` entre 2 e 5 → `BadRequestException` se < 2 ou > 5 (RN-013 — máx. 5)
  - Para cada `opportunityId`: chamar `get_opportunity_data` tool para buscar dados (com isolamento de dados — só oportunidades do `cessionario_id`)
  - Calcular para cada oportunidade: `delta`, `roi`, `annualizedRoi`, `commissionAmount`, `netAmount` via `CalculadoraService`
  - Identificar Top 3 oportunidades por `weightedScore = (roi * 0.5) + ((1/delta) * 0.3) + (netAmount/totalNominal * 0.2)` — documentar fórmula exata conforme Doc 01.2 RN-017
  - Retornar `{ opportunities: ComparedOpportunity[], topThree: string[], comparison: ComparisonMatrix }`
- [x] `ComparisonMatrix`: tabela comparativa com campos `delta`, `roi`, `annualizedRoi`, `netAmount`, `rank` para cada oportunidade
- [x] Teste unitário: 2 oportunidades → comparação válida; 5 oportunidades → Top 3 correto; 6 oportunidades → `BadRequestException`; 1 oportunidade → `BadRequestException`; oportunidade de outro usuário → excluída silenciosamente (retornar comparação com N-1 oportunidades se N-1 ≥ 2, senão `404`)

### 2. Endpoint `POST /ai/compare` — comparar oportunidades
- [x] Criar endpoint `POST /ai/compare` em `AiController` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `CompareOpportunitiesDto`: `{ conversationId: string (uuid), opportunityIds: string[] (min 2, max 5) }`
- [x] `AiService.compare(user, dto)`:
  1. Verificar conversa existe e pertence ao usuário
  2. Verificar consentimento LGPD `ai_analysis`
  3. Chamar `ComparacaoService.compareOpportunities`
  4. Gerar narrativa via GPT-4o baseada na comparação (não recalcular — usar resultado determinístico)
  5. Persistir mensagens no histórico de conversa (user + assistant)
- [x] SLA: ≤10s total (busca dados + cálculo + geração narrativa) — logar aviso via Pino se ultrapassar
- [x] `ComparisonResponseDto`: `{ conversationId, message: { role: 'assistant', content: string }, comparison: ComparisonResult, topThree: string[], calculatedAt: string }`
- [x] Resposta HTTP: `200 OK`
- [x] Teste de integração: 3 oportunidades válidas → `200` com `topThree.length = 3`; 1 oportunidade → `400`; 6 oportunidades → `400`; sem consentimento LGPD → `403`

### 3. Endpoint `GET /ai/compare/stream` — comparação com streaming
- [x] Criar endpoint `GET /ai/compare/stream` (`@Roles('CESSIONARIO', 'ADMIN')`) como SSE
- [x] Query params: `conversationId`, `opportunityIds` (comma-separated, ex: `id1,id2,id3`)
- [x] Emitir evento intermediário: `data: {"type":"calculating","message":"Calculando comparação..."}\n\n`
- [x] Emitir chunks da narrativa GPT-4o via `streamText()`
- [x] Emitir evento final: `data: {"type":"done","comparison":{...},"topThree":[...]}\n\n`
- [x] Teste de integração: GET stream → sequência: `calculating` → chunks → `done` com `comparison` em JSON; `opportunityIds` com 6 ids → `data: {"type":"error",...}` + fechar stream

---

## 🏗️ FEATURE 2: Simulação de Propostas

### 4. `SimulacaoService.simulateProposal` — simulação de proposta
- [x] Criar `SimulacaoService` com método `simulateProposal(user: AuthenticatedUser, opportunityId: string, proposedDelta: number): SimulationResult`:
  - Buscar dados da oportunidade via `get_opportunity_data` (com isolamento)
  - Calcular impacto da proposta: `proposedMarketValue = nominalValue * (1 - proposedDelta/100)`
  - Calcular `deltaFromCurrent = ((proposedMarketValue - currentMarketValue) / currentMarketValue) * 100`
  - Calcular ROI com o novo valor: `proposedRoi`, `proposedAnnualizedRoi`
  - Gerar avaliação: se `proposedDelta > currentDelta * 1.1` → `assessment: 'desfavorável'`; se dentro de ±10% → `assessment: 'neutro'`; se `proposedDelta < currentDelta * 0.9` → `assessment: 'favorável'`
  - Retornar `{ opportunityId, currentDelta, proposedDelta, proposedMarketValue, deltaFromCurrent, proposedRoi, proposedAnnualizedRoi, assessment }`
- [x] Teste unitário: `proposedDelta=15` com `currentDelta=20` → `assessment: 'favorável'`; `proposedDelta=20` → `assessment: 'neutro'`; `proposedDelta=25` → `assessment: 'desfavorável'`; `proposedDelta<0` → `BadRequestException`

### 5. `SimulacaoService.simulateCounterproposal` — contraproposta
- [x] Implementar `simulateCounterproposal(user: AuthenticatedUser, opportunityId: string, targetRoi: number): CounterproposalResult`:
  - Calcular `requiredMarketValue = nominalValue / (1 + targetRoi/100)` (inverso do ROI)
  - Calcular `requiredDelta = ((nominalValue - requiredMarketValue) / nominalValue) * 100`
  - Validar que `requiredDelta` está em range razoável (0-50%) → se fora, marcar `feasibility: 'inviável'`
  - Retornar `{ opportunityId, targetRoi, requiredMarketValue, requiredDelta, feasibility: 'viável' | 'inviável', currentMarketValue, gap }`
- [x] Teste unitário: `targetRoi=25, investment=100` → `requiredMarketValue=80`; `targetRoi=200` → `feasibility: 'inviável'`; `targetRoi=-10` → `BadRequestException`

### 6. Endpoint `POST /ai/simulate` — simular proposta
- [x] Criar endpoint `POST /ai/simulate` em `AiController` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `SimulateProposalDto`: `{ conversationId: string (uuid), opportunityId: string (uuid), proposedDelta: number (min 0, max 100), simulationType: 'proposal' | 'counterproposal', targetRoi?: number }`
- [x] Se `simulationType === 'proposal'` → `SimulacaoService.simulateProposal`; se `'counterproposal'` → `SimulacaoService.simulateCounterproposal`
- [x] Gerar narrativa via GPT-4o baseada no resultado determinístico
- [x] Persistir no histórico de conversa
- [x] `SimulationResponseDto`: `{ conversationId, message: { role: 'assistant', content: string }, simulation: SimulationResult | CounterproposalResult, simulationType, calculatedAt }`
- [x] Resposta HTTP: `200 OK`
- [x] Teste de integração: `simulationType='proposal'` com `proposedDelta=15` → `200` com `assessment`; `simulationType='counterproposal'` sem `targetRoi` → `400`; `proposedDelta=101` → `400`

### 7. Endpoint `POST /ai/simulate/portfolio` — simulação de portfólio completo
- [x] Criar endpoint `POST /ai/simulate/portfolio` em `AiController` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `SimulatePortfolioDto`: `{ conversationId: string (uuid), opportunityIds: string[] (min 1, max 10) }`
- [x] `AiService.simulatePortfolio(user, dto)`:
  1. Buscar dados de todas as oportunidades (com isolamento)
  2. Chamar `CalculadoraService.calculatePortfolio` com dados reais
  3. Gerar narrativa de análise de portfólio via GPT-4o
  4. Persistir no histórico
- [x] Retornar `PortfolioSimulationResponseDto`: `{ conversationId, message, portfolio: PortfolioResult, calculatedAt }`
- [x] Teste de integração: 5 oportunidades válidas → `200` com `portfolio.summary`; oportunidade inexistente → incluída com `error: 'not_found'` no array mas não abortar

---

## 🏗️ FEATURE 3: Top 3 Recomendação e Suporte Operacional

### 8. `ComparacaoService.getTopThreeRecommendation` — Top 3 explícito
- [x] Implementar `getTopThreeRecommendation(user: AuthenticatedUser, opportunityIds: string[]): TopThreeResult`:
  - Executar `compareOpportunities` e selecionar Top 3 por `weightedScore`
  - Para cada uma das Top 3: gerar justificativa textual dos pontos fortes (delta competitivo, ROI atrativo, prazo favorável)
  - Retornar `{ topThree: [{ rank: 1|2|3, opportunityId, score, justification, metrics: { delta, roi, netAmount } }], generatedAt }`
- [x] Deve funcionar com 3 a 10 oportunidades como input (máx. 5 na comparação padrão, mas para Top 3 até 10 para maior pool)
- [x] Teste unitário: 10 oportunidades → retornar exatamente 3; empate no score → critério de desempate por `netAmount DESC`; < 3 oportunidades elegíveis → retornar as disponíveis (1 ou 2)

### 9. Endpoint `GET /ai/opportunities/top` — endpoint Top 3
- [x] Criar endpoint `GET /ai/opportunities/top` em `AiController` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] Query params: `conversationId (uuid, obrigatório)`, `opportunityIds (comma-separated, max 10)`
- [x] Retornar `TopThreeResponseDto`: `{ conversationId, topThree: TopThreeItem[], generatedAt }`
- [x] SLA: ≤5s — este endpoint usa apenas cálculo determinístico (sem narrativa GPT-4o)
- [x] Teste de integração: 10 oportunidades → `200` com `topThree.length = 3`; 2 oportunidades → `200` com `topThree.length = 2`; 0 oportunidades → `400`

### 10. Suporte operacional — status de KYC/Escrow/ZapSign
- [x] Criar `SuporteOperacionalService` com método `getOperationalStatus(user: AuthenticatedUser, opportunityId: string): OperationalStatus`:
  - **Fonte de dados (FINDING-013):** chamadas à API interna da plataforma Repasse Seguro via client HTTP interno (fetch nativo, não Axios). Endpoints: `GET /internal/opportunities/{id}/kyc-status`, `GET /internal/opportunities/{id}/escrow-status`, `GET /internal/opportunities/{id}/zapsign-status`. Se API interna indisponível: retornar `status: 'unknown'` sem lançar erro — documentado no response. — [CORRIGIDO: FINDING-013]
  - Verificar status de KYC da oportunidade via `GET /internal/opportunities/{id}/kyc-status`
  - Verificar status de Escrow via `GET /internal/opportunities/{id}/escrow-status`
  - Verificar status de assinatura ZapSign via `GET /internal/opportunities/{id}/zapsign-status`
  - **Campos do dossiê (FINDING-029):** campo `dossie` no response com: `{ documentosObrigatorios: ['id_frente', 'id_verso', 'selfie_vivacidade', 'comprovante_endereco_90d'], documentosEnviados: string[], documentosPendentes: string[], status: 'completo' | 'pendente' | 'reprovado' }`. Fonte: módulo Cessionário da plataforma principal via `GET /internal/cessionarios/{id}/dossie`. — [CORRIGIDO: FINDING-029]
  - Retornar `{ opportunityId, kyc: { status, updatedAt }, escrow: { status, updatedAt }, zapSign: { status, updatedAt }, dossie: { documentosObrigatorios, documentosEnviados, documentosPendentes, status }, pendingActions: string[] }`
  - `pendingActions`: lista de ações pendentes derivada dos status (ex: "KYC aguardando validação", "Escrow pendente de fundos", "Dossiê pendente: selfie_vivacidade")
- [x] Dados de status são apenas leitura — sem mutações nesta sprint
- [x] Teste unitário: todos os status `completed` + dossie `completo` → `pendingActions: []`; KYC `pending` → `pendingActions` inclui item de KYC; API interna indisponível → `status: 'unknown'` sem erro; dossiê com `documentosPendentes: ['selfie_vivacidade']` → `pendingActions` inclui "Dossiê pendente: selfie_vivacidade"

---

## ✅ TESTES — Comparação e Simulação

### 11. Suite de testes
- [x] Criar `test/unit/ai/services/comparacao.service.spec.ts` — cobrir max 5, Top 3, isolamento dados
- [x] Criar `test/unit/ai/services/simulacao.service.spec.ts` — cobrir proposta, contraproposta, `feasibility`
- [x] Criar `test/integration/ai/compare.spec.ts` — cobrir `POST /ai/compare` e `GET /ai/compare/stream`
- [x] Criar `test/integration/ai/simulate.spec.ts` — cobrir `POST /ai/simulate` e `POST /ai/simulate/portfolio`
- [x] Criar `test/integration/ai/top.spec.ts` — cobrir `GET /ai/opportunities/top`
- [x] Criar `test/e2e/ai/compare-flow.e2e-spec.ts` — E2E: comparar 3 oportunidades → receber Top 3 (2º fluxo E2E obrigatório do Doc 27)
- [x] Cobertura mínima módulo `ai/`: 85% (conforme Doc 27)

---

## 🔀 Cross-Módulo

- [x] **← S4 (Calculadora):** `ComparacaoService` e `SimulacaoService` chamam `CalculadoraService` para todos os cálculos determinísticos — confirmar import do `CalculadoraModule`
- [x] **← S3a (Tools):** `get_opportunity_data` tool usado para busca de dados com isolamento — confirmar disponibilidade do tool via DI
- [x] **→ S6 (Admin):** comparações e simulações registradas em `ai_interactions` alimentam métricas do painel Admin (sem dependência nova — apenas persistência via `PERSIST` node)

---

## ✔️ Auto-Verificação S5 (12 Checks)

| # | Check | Status |
|---|---|---|
| 1 | Todos os itens são binariamente verificáveis | ✅ |
| 2 | Nomes exatos: `ComparacaoService`, `SimulacaoService`, `SuporteOperacionalService`, `compareOpportunities`, `simulateProposal`, `simulateCounterproposal`, `getTopThreeRecommendation` | ✅ |
| 3 | Valores numéricos: máx. 5 oportunidades comparação, máx. 10 portfólio, Top 3, SLA ≤5s e ≤10s, cenários ±20%, `assessment` thresholds ±10% | ✅ |
| 4 | Glossário consultado — Δ, ROI, Score de Risco, KYC, Escrow, ZapSign | ✅ |
| 5 | Anti-Scaffold R10: fórmula de `weightedScore` documentada; `feasibility` com lógica real; `pendingActions` derivadas dos status | ✅ |
| 6 | Isolamento de dados: `get_opportunity_data` filtra por `cessionario_id` em todas as operações | ✅ |
| 7 | SLA ≤10s para comparação; SLA ≤5s para Top 3 (sem GPT-4o) | ✅ |
| 8 | Sem conflitos não marcados | ✅ |
| 9 | Cálculos determinísticos (Calculadora) separados da narrativa GPT-4o | ✅ |
| 10 | Template B: organizado por FEATURE (Feature 1: Comparação, Feature 2: Simulação, Feature 3: Top 3 + Suporte) | ✅ |
| 11 | `compareOpportunities` com 1 ou 6+ oportunidades → `BadRequestException` | ✅ |
| 12 | REQs cobertos: REQ-040 a REQ-058 — todos com ≥1 item | ✅ |
