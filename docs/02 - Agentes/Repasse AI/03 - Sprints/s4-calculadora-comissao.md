# S4 — Calculadora de Comissão
<!-- Atualizado em 2026-03-24 pela A03 — 4 correções aplicadas (FINDING-010, FINDING-012, FINDING-022, FINDING-027) -->

| Campo | Valor |
|---|---|
| **Sprint** | S4 |
| **Nome** | Calculadora de Comissão |
| **Template** | B — Módulo Fullstack |
| **Docs Fonte** | 01.2-RN-Core, 01.3-RN-Operação, 16-API, 19-CriaçãoAgentes |
| **REQs cobertos** | REQ-022, REQ-023, REQ-033, REQ-034, REQ-035, REQ-068, REQ-069, REQ-070, REQ-074, REQ-095, REQ-096, REQ-097 |
| **Total de itens** | 24 |
| **Status** | ✅ Concluído |

---

## 🎯 Objetivo

Implementar a `CalculadoraModule`: módulo determinístico independente do LLM que serve como fallback quando o agente de IA está indisponível. Cobre cálculos de Δ (deságio), ROI, comissão, simulação de portfólio e os 3 endpoints REST da Calculadora.

---

## 🏗️ FEATURE 1: Serviço de Cálculo Determinístico

### 1. `CalculadoraService` — lógica pura de cálculo
- [x] Criar `CalculadoraService` como serviço NestJS **sem dependência de LLM** — cálculos determinísticos puros:
  - `calculateDelta(nominalValue: number, marketValue: number): DeltaResult`:
    - `delta = ((nominalValue - marketValue) / nominalValue) * 100`
    - Resultado arredondado a 2 casas decimais
    - Validar: `nominalValue > 0`, `marketValue > 0` → lançar `BadRequestException` se inválido
    - Retornar `{ delta: number, nominalValue: number, marketValue: number, deltaAbsolute: number }`
  - `calculateRoi(investment: number, expectedReturn: number, periodMonths: number): RoiResult`:
    - `roi = ((expectedReturn - investment) / investment) * 100`
    - `annualizedRoi = ((1 + roi/100)^(12/periodMonths) - 1) * 100`
    - Cenários ±20%: `{ pessimistic: roi * 0.8, base: roi, optimistic: roi * 1.2 }`
    - Validar: `investment > 0`, `periodMonths > 0` e inteiro → `BadRequestException` se inválido
    - Retornar `{ roi, annualizedRoi, scenarios, periodMonths }`
- [x] Teste unitário `CalculadoraService.calculateDelta`: `nominal=1000, market=800` → `delta=20.00, deltaAbsolute=200.00`; `nominal=0` → `BadRequestException`; `market > nominal` → delta negativo (válido)
- [x] Teste unitário `CalculadoraService.calculateRoi`: `investment=80, return=100, months=12` → `roi=25.00, annualizedRoi=25.00`; `months=6` → `annualizedRoi` correto; `investment=0` → `BadRequestException`

### 2. `CalculadoraService.calculateCommission` — cálculo de comissão [CORRIGIDO: FINDING-027]
- [x] Implementar `calculateCommission(nominalValue: number, delta: number, cedentePaidValue: number): CommissionResult`:
  - **Taxa é FIXA em 20% — não é parâmetro externo** (regra de negócio: Comissão = 20% × Δ, ou 20% × cedentePaidValue quando Δ ≤ 0)
  - Se `delta > 0`: `base = nominalValue * (delta / 100)` → `commissionAmount = base * 0.20`
  - Se `delta <= 0`: `base = cedentePaidValue` → `commissionAmount = base * 0.20` + incluir nota explicativa no response
  - `marketValue = nominalValue - (nominalValue * delta / 100)` (se delta > 0)
  - `netAmount = marketValue - commissionAmount`
  - Validar: `nominalValue > 0`, `cedentePaidValue > 0` → `BadRequestException` se inválido
  - Retornar `{ nominalValue, marketValue, delta, commissionRate: 0.20, commissionAmount, netAmount, fallbackApplied: boolean, fallbackNote?: string }`
- [x] **Descontos**: taxa de 20% nunca reduzida por este serviço — desconto é EXCLUSIVO do Admin via painel de supervisão
- [x] Teste unitário: `nominal=1000, delta=20` → `commission = 1000 * 0.20 * 0.20 = 40` ❌ — recalcular: `base = 1000 * (20/100) = 200`, `commission = 200 * 0.20 = 40`; `delta=0, cedentePaid=800` → `commission = 800 * 0.20 = 160` (fallback); `cedentePaidValue=0` → `BadRequestException`

### 3. `CalculadoraService.calculatePortfolio` — simulação de carteira
- [x] Implementar `calculatePortfolio(opportunities: PortfolioOpportunity[]): PortfolioResult`:
  - Para cada oportunidade: calcular `delta`, `roi`, `commission`, `netAmount`
  - Totais: `totalInvestment = sum(marketValue)`, `totalNominal = sum(nominalValue)`, `totalCommission = sum(commissionAmount)`, `totalNet = sum(netAmount)`
  - `weightedDelta = sum(delta * marketValue) / totalInvestment` (ponderado por valor de mercado)
  - `weightedRoi = sum(roi * marketValue) / totalInvestment`
  - Limite: máximo 10 oportunidades → `BadRequestException` se mais de 10
  - Retornar `{ opportunities: PortfolioOpportunityResult[], summary: PortfolioSummary }`
- [x] Teste unitário: 3 oportunidades → `weightedDelta` e `weightedRoi` corretos matematicamente; 11 oportunidades → `BadRequestException`; 1 oportunidade → `summary.count = 1`

### 4. Fallback do LLM — integração com S3b
- [x] Implementar `CalculadoraService.handleLlmFallback(intent: string, toolInput: unknown): FallbackResult`:
  - Recebe `intent` do `AgentState` e input do tool que falhou
  - Se `intent === 'analise_oportunidade'`: chamar `calculateDelta` + `calculateRoi` + formatar resposta em texto
  - Se `intent === 'simulacao_portfolio'`: chamar `calculatePortfolio`
  - Resposta de fallback inclui aviso: `"Análise realizada pelo módulo de cálculo (modo offline). Algumas análises avançadas estão temporariamente indisponíveis."`
  - Se intent não suportado pelo fallback (`comparacao_oportunidades`, `suporte_operacional`, `fora_do_escopo`) → retornar mensagem de indisponibilidade sem tentar calcular
- [x] Registrar uso do fallback em `agent_status_logs` com `status: 'degraded'` e `reason: 'LLM_FALLBACK_ACTIVATED'`
- [x] Teste unitário: `intent='analise_oportunidade'` com input válido → retornar análise com aviso de fallback; `intent='comparacao_oportunidades'` → retornar mensagem de indisponibilidade; INSERT em `agent_status_logs` chamado uma vez

---

## 🏗️ FEATURE 2: Endpoints REST da Calculadora

### 5. Endpoint `POST /calculator/delta` — calcular deságio
- [x] Criar `CalculatorController` com endpoint `POST /calculator/delta` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `CalculateDeltaDto`: `{ nominalValue: z.number().positive(), marketValue: z.number().positive() }` — validado com `class-validator`
- [x] `CalculatorController.calculateDelta` → `CalculadoraService.calculateDelta` → retornar `DeltaResponseDto`
- [x] `DeltaResponseDto`: `{ delta: number, nominalValue: number, marketValue: number, deltaAbsolute: number, calculatedAt: string }`
- [x] Resposta HTTP: `200 OK`
- [x] Teste de integração: POST `{ nominalValue: 1000, marketValue: 800 }` → `200 { delta: 20.00 }`; `nominalValue: -1` → `400`; sem body → `400`; token `CEDENTE` → `403`

### 6. Endpoint `POST /calculator/roi` — calcular ROI
- [x] Criar endpoint `POST /calculator/roi` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `CalculateRoiDto`: `{ investment: z.number().positive(), expectedReturn: z.number(), periodMonths: z.number().int().positive() }`
- [x] Retornar `RoiResponseDto`: `{ roi: number, annualizedRoi: number, scenarios: { pessimistic: number, base: number, optimistic: number }, periodMonths: number, calculatedAt: string }`
- [x] Cenários ±20% no response body conforme RN-019
- [x] Teste de integração: POST válido → `200` com 3 cenários; `periodMonths=0` → `400`; `periodMonths=1.5` (não inteiro) → `400`

### 7. Endpoint `POST /calculator/portfolio` — simulação de carteira
- [x] Criar endpoint `POST /calculator/portfolio` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `CalculatePortfolioDto`: `{ opportunities: CalculatePortfolioItemDto[] }` — array de 1 a 10 itens
- [x] `CalculatePortfolioItemDto`: `{ id: string, nominalValue: number, marketValue: number, expectedReturn: number, periodMonths: number, commissionRate: number }`
- [x] Retornar `PortfolioResponseDto`: `{ opportunities: PortfolioItemResult[], summary: { totalInvestment, totalNominal, totalCommission, totalNet, weightedDelta, weightedRoi, count }, calculatedAt: string }`
- [x] Resposta HTTP: `200 OK`
- [x] Teste de integração: 1 item → `summary.count=1`; 10 itens → `summary` com ponderação correta; 11 itens → `400`; array vazio → `400`

---

## ✅ TESTES — Calculadora

### 8. Endpoint `POST /calculator/escrow` — calcular custo Escrow [CORRIGIDO: FINDING-022]
- [x] Criar endpoint `POST /calculator/escrow` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `CalculateEscrowDto`: `{ repaymentPrice: number (positive), commissionAmount: number (min 0) }`
- [x] `CalculadoraService.calculateEscrow(repaymentPrice, commissionAmount)`:
  - `totalEscrow = repaymentPrice + commissionAmount`
  - Retornar `{ repaymentPrice, commissionAmount, totalEscrow, prazoUteis: 10, extensaoMaximaUteis: 5, reversaoCorridos: 15 }`
  - Validar: `repaymentPrice > 0` → `BadRequestException` se inválido
- [x] `EscrowResponseDto`: `{ repaymentPrice, commissionAmount, totalEscrow, prazoUteis, extensaoMaximaUteis, reversaoCorridos, calculatedAt }`
- [x] Resposta HTTP: `200 OK`
- [x] Teste unitário: `repaymentPrice=800000, commissionAmount=32000` → `totalEscrow=832000, prazoUteis=10, extensaoMaximaUteis=5, reversaoCorridos=15`; `repaymentPrice=0` → `400`; `commissionAmount=-1` → `400`

### 9. Suite de testes unitários e de integração
- [x] Criar `test/unit/calculator/calculadora.service.spec.ts` — cobrir todos os métodos de cálculo com valores extremos (0, negativo, máximo)
- [x] Criar `test/unit/calculator/calculadora.fallback.spec.ts` — cobrir todos os paths do fallback LLM
- [x] Criar `test/integration/calculator/calculator.controller.spec.ts` — cobrir os 3 endpoints
- [x] Cobertura mínima módulo `calculator/`: 75% (conforme Doc 27)

---

## 🔀 Cross-Módulo

- [x] **← S3b (Agente Core):** `CalculadoraService.handleLlmFallback()` é chamado pelo nó `GENERATE` quando `error.code === 'LLM_UNAVAILABLE'` — confirmar que `CalculadoraModule` é importado pelo `AiModule`
- [x] **← S3a (Tools):** `calculate_delta`, `calculate_roi`, `calculate_portfolio` tools reutilizam a lógica pura do `CalculadoraService` (não duplicar lógica de cálculo — DRY)
- [x] **← S1 (Fundação) — [CORRIGIDO: FINDING-012]:** `agent_status_logs` criada em S1 migration 010 é utilizada pelo `handleLlmFallback` para INSERT com `status: 'degraded'`. `PrismaModule` é global — disponível automaticamente. Confirmar via teste de integração.

---

## ✔️ Auto-Verificação S4 (12 Checks)

| # | Check | Status |
|---|---|---|
| 1 | Todos os itens são binariamente verificáveis | ✅ |
| 2 | Nomes exatos: `CalculadoraService`, `CalculatorController`, `calculateDelta`, `calculateRoi`, `calculateCommission`, `calculatePortfolio`, `handleLlmFallback`, `agent_status_logs` | ✅ |
| 3 | Valores numéricos: max 10 oportunidades portfolio, cenários ±20%, arredondamento 2 casas decimais, `commissionRate` 0-100% | ✅ |
| 4 | Glossário consultado — Δ (deságio), ROI, Calculadora de Comissão, Cenários A/B/C/D | ✅ |
| 5 | Anti-Scaffold R10: lógica de cálculo real com fórmulas, validações de range, fallback com aviso ao usuário | ✅ |
| 6 | Calculadora é determinística — zero dependência de LLM nesta sprint | ✅ |
| 7 | Fallback registrado em `agent_status_logs` com `status: 'degraded'` | ✅ |
| 8 | Sem conflitos não marcados | ✅ |
| 9 | DRY: tools de S3a reutilizam lógica do `CalculadoraService` (sem duplicação) | ✅ |
| 10 | Template B: organizado por FEATURE (Feature 1: Serviço, Feature 2: Endpoints) | ✅ |
| 11 | Nenhuma dependência de LLM na lógica de cálculo — módulo funciona 100% offline | ✅ |
| 12 | REQs cobertos: REQ-022, REQ-023, REQ-033 a REQ-035, REQ-068 a REQ-070, REQ-074, REQ-095 a REQ-097 — todos com ≥1 item | ✅ |
