# S4 — Calculadora de Comissão e Análise Individual

| **Sprint** | S4 — Calculadora de Comissão e Análise Individual |
|---|---|
| **Template** | B — Módulo Fullstack (organizado por Feature) |
| **REQs cobertos** | REQ-012, REQ-013, REQ-014, REQ-015, REQ-016, REQ-017, REQ-021, REQ-022, REQ-023, REQ-028, REQ-078, REQ-087, REQ-121, REQ-122 (parcial) |
| **Docs fonte** | D01 (RN-DC-011–014, RN-DC-016, RN-DC-017, RN-DC-022), D05.2 (RF-DC-013–022), D05.5 (RF-DC-043–044), D14 (CalculadoraModule, OportunidadeModule), D20 (CALC_, OPR_) |
| **Total de itens** | 46 |

---

## 🎯 Objetivo

Implementar o `CalculadoraModule` determinístico (independente do agente IA) com cálculos de Comissão, Custo Total de Escrow e ROI em 3 cenários. Implementar o `OportunidadeModule` (read-only). Implementar o módulo de análise individual (T-DC-005 + componentes `CommissionCard`, `RiskScoreBadge`, `OprStatusBadge`). Cobrir todos os edge cases (Δ=0, Δ negativo, proposta acima da Tabela Atual, score indisponível).

---

## FEATURE 1: Calculadora de Comissão (CalculadoraModule)

### Banco

- [x] **[F1-BANCO-001]** Configurar cache Redis para resultados da Calculadora:
  - Sub-item: Redis key: `dani:cache:calc:{opr_id}:{val_hash}` TTL 300s (5 min)
  - Sub-item: `val_hash` = SHA-256 dos parâmetros `{ opr_id, valor_proposta }` como string JSON
  - Sub-item: `SETEX dani:cache:calc:{opr_id}:{val_hash} 300 {resultado_json}`
  - Sub-item: Cache miss: computar e armazenar; cache hit: retornar sem consultar banco

### Backend

- [x] **[F1-BACK-001]** Implementar `CalculadoraService.calcularComissao()` — cobertura 100% obrigatória (REQ-149):
  - Sub-item: Parâmetros de entrada: `{ tabela_atual: number, tabela_contrato: number, valor_pago_cedente: number, valor_proposta: number }`
  - Sub-item: Passo 1 — validação: `valor_proposta ≤ 0` ou `!isFinite()` → lança `CalculadoraException CALC_001` status 422 antes de consultar Redis/banco
  - Sub-item: Passo 2 — calcular `delta = tabela_atual - tabela_contrato`
  - Sub-item: Passo 3 — se `delta > 0`: `comissao = 0.20 * delta`, `base_calculo = "delta"` (REQ-015)
  - Sub-item: Passo 4 — se `delta <= 0` (incluindo `delta = 0`): `comissao = 0.20 * valor_pago_cedente`, `base_calculo = "valor_pago_cedente"`, campo `nota_fallback = true` (REQ-015, REQ-016)
  - Sub-item: Passo 5 — `custo_total = valor_proposta + comissao` (REQ-017)
  - Sub-item: Passo 6 — calcular ROI 3 cenários (REQ-023):
    - Conservador: `tabela_atual_conservador = tabela_atual * 0.80`; `roi_conservador = (tabela_atual_conservador - custo_total) / custo_total * 100`
    - Base: `roi_base = (tabela_atual - custo_total) / custo_total * 100`
    - Otimista: `tabela_atual_otimista = tabela_atual * 1.20`; `roi_otimista = (tabela_atual_otimista - custo_total) / custo_total * 100`
  - Sub-item: Passo 7 — se `roi_base < 0`: campo `roi_negativo = true`, `perda_estimada = abs(roi_base * custo_total / 100)` (REQ-022)
  - Sub-item: Retorno: `{ delta, comissao, base_calculo, custo_total, roi: { conservador, base, otimista }, roi_negativo?, perda_estimada?, nota_fallback? }`
  - Sub-item: Log `info`: `{ opr_id, formula_aplicada: "delta"|"valor_pago_cedente", latency_ms }` (D25)

- [x] **[F1-BACK-002]** Implementar endpoint `POST /api/v1/calculadora/calcular` conforme D16:
  - Sub-item: Guards: `JwtAuthGuard`, `CessionarioOwnerGuard`
  - Sub-item: Body DTO: `{ opr_id: string, valor_proposta: number, tabela_atual: number, tabela_contrato: number, valor_pago_cedente: number }`
  - Sub-item: Passo 1 — verifica cache Redis `dani:cache:calc:{opr_id}:{val_hash}`; hit → retorna com `source: "cache"`
  - Sub-item: Passo 2 — cache miss → chama `CalculadoraService.calcularComissao()`
  - Sub-item: Passo 3 — armazena resultado no cache (`SETEX ... 300`)
  - Sub-item: Retorna 200 com resultado + `source: "computed" | "cache"`
  - Sub-item: `CalculadoraModule` não importa de `AgenteModule` — verificado em build (ADR-003, REQ-105)

- [x] **[F1-BACK-003]** Implementar `OportunidadeService.buscarPorOpr()` e capturarSnapshot():
  - Sub-item: `buscarPorOpr(opr_id, cessionario_id)` — query read-only em tabela `oportunidades` da plataforma principal
  - Sub-item: Retorna `{ tabela_atual, tabela_contrato, valor_pago_cedente, localizacao, tipologia, status }` ou `OprException OPR_001` status 404
  - Sub-item: `capturarSnapshot(opr_id, cessionario_id, dados)` — persiste em `dani_contextos_opr.dados_oportunidade JSONB`; TTL `expires_at` = NOW() + 7 dias
  - Sub-item: Endpoint `GET /api/v1/oportunidades/{opr_id}` retorna snapshot se cache em `dani_contextos_opr` presente (< 7 dias), senão consulta plataforma
  - Sub-item: Oportunidade indisponível (status `ENCERRADA` ou inexistente): retorna `OprException OPR_001` status 404

### Frontend

- [x] **[F1-FRONT-001]** Implementar `CommissionCard` conforme D09:
  - Sub-item: Exibe: `Δ (Delta): R$ X.XXX,XX`, `Comissão: R$ X.XXX,XX`, `Custo Total Escrow: R$ X.XXX,XX`
  - Sub-item: Quando `nota_fallback = true`: exibe nota "Como a Tabela Atual não é superior à Tabela Contrato, a comissão é calculada sobre o Valor Pago pelo Cedente." (REQ-016, D08 §2.3)
  - Sub-item: Quando `roi_negativo = true`: exibe "Com esse valor, o retorno desta oportunidade seria negativo. O investimento resultaria em uma perda estimada de R$ X.XXX,XX." + aviso de projeção (REQ-022, D08 §2.4)
  - Sub-item: Formatação monetária: `R$ X.XXX,XX` (separador de milhar + 2 casas decimais — REQ-133)
  - Sub-item: 4 estados: Skeleton → Empty → Error → Populated (REQ-129)

- [x] **[F1-FRONT-002]** Implementar ROI card com 3 cenários e aviso de projeção obrigatório:
  - Sub-item: Layout: `🛡 Conservador: [X]%`, `🎯 Base: [X]% ← referência`, `📈 Otimista: [X]%`
  - Sub-item: Aviso obrigatório (não pode ser omitido — REQ-132): "ℹ Esses são valores projetados com base nos dados disponíveis. Resultados reais podem variar conforme condições de mercado."
  - Sub-item: Aviso em `text-xs`, cor `--muted-foreground`
  - Sub-item: ROI negativo: texto em `--destructive`

### Testes

- [x] **[F1-TEST-001]** Testes unitários `CalculadoraService` — cobertura 100% (P0 crítico REQ-149):
  - Sub-item: `delta > 0`: `calcularComissao({ tabela_atual: 85000, tabela_contrato: 70000, valor_pago_cedente: 65000, valor_proposta: 300000 })` → `delta=15000`, `comissao=3000`, `base_calculo="delta"`
  - Sub-item: `delta < 0`: `tabela_atual=60000, tabela_contrato=70000, valor_pago_cedente=65000` → `delta=-10000`, `comissao=13000`, `base_calculo="valor_pago_cedente"`, `nota_fallback=true`
  - Sub-item: `delta = 0` (edge case REQ-016): `tabela_atual=70000, tabela_contrato=70000, valor_pago_cedente=65000` → `comissao=13000`, `base_calculo="valor_pago_cedente"`, `nota_fallback=true`
  - Sub-item: `valor_proposta` inválido (`0`, `-100`, `NaN`) → lança `CALC_001` status 422
  - Sub-item: ROI base negativo (REQ-022): `tabela_atual=100000, custo_total=150000` → `roi_negativo=true`, `perda_estimada` calculado corretamente
  - Sub-item: ROI conservador = `(tabela_atual * 0.80 - custo_total) / custo_total * 100`; otimista = `(tabela_atual * 1.20 - custo_total) / custo_total * 100` (REQ-023)

- [x] **[F1-TEST-002]** Testes de integração `POST /calculadora/calcular`:
  - Sub-item: Cache hit → retorna com `source: "cache"` sem consultar banco
  - Sub-item: Cache miss → calcula, armazena, retorna com `source: "computed"`. Próxima call idêntica → `source: "cache"`
  - Sub-item: Redis indisponível → cache miss → calcula sem cache (fail open); log `warn` gerado

---

## FEATURE 2: Análise de Oportunidade Individual (T-DC-005)

### Backend

- [x] **[F2-BACK-001]** Implementar `AgenteService.analisarOportunidade()` que compõe análise completa conforme D01 RN-DC-011 + D05.2 RF-DC-013:
  - Sub-item: Dado `opr_id` e `cessionario_id`: busca snapshot via `OportunidadeService.buscarPorOpr()`
  - Sub-item: Calcula `delta`, `comissao`, `custo_total`, 3 cenários ROI via `CalculadoraService`
  - Sub-item: Chama tool `analisarRisco` → score 1–10 com fatores
  - Sub-item: Chama tool `buscarComparativoRegional` → dados de mercado da região
  - Sub-item: Verifica status da oportunidade: `DISPONIVEL`, `EM_NEGOCIACAO`, `ENCERRADA`
  - Sub-item: Se oportunidade sai do marketplace durante simulação (status mudou para `ENCERRADA`): na próxima mensagem informa "A oportunidade [OPR-XXXX] não está mais disponível." + sugere até 3 oportunidades semelhantes como chips (REQ-013)
  - Sub-item: Latência da análise individual deve ser ≤5s (REQ-035); loga latência

- [x] **[F2-BACK-002]** Implementar score de risco conforme D01 RN-DC-012 + D05.2 RF-DC-015:
  - Sub-item: Escala 1–10: 1–3 = baixo, 4–6 = moderado, 7–10 = alto
  - Sub-item: Se dados insuficientes para calcular score → retorna `score_disponivel = false`; mensagem: "Os dados disponíveis desta oportunidade não são suficientes para calcular um score de risco preciso. Recomendo solicitar o dossiê completo antes de decidir." (D08 §2.3)

### Frontend

- [x] **[F2-FRONT-001]** Implementar `RiskScoreBadge` conforme D09 §2.3:
  - Sub-item: 1–3: `--risk-low (#16A34A)`, bg `--risk-low-bg (#F0FDF4)`, texto "RISCO BAIXO"
  - Sub-item: 4–6: `--risk-medium (#D97706)`, bg `--risk-medium-bg (#FFFBEB)`, texto "RISCO MODERADO"
  - Sub-item: 7–10: `--risk-high (#DC2626)`, bg `--risk-high-bg (#FEF2F2)`, texto "RISCO ALTO"
  - Sub-item: Score indisponível: `--muted-foreground`, bg `--muted`, texto "DADOS INSUFICIENTES"
  - Sub-item: `aria-label="Score de risco: {N} de 10 — risco {baixo/moderado/alto}"` (D07 W-DC-006, REQ-014)
  - Sub-item: Contraste mínimo 4.5:1 obrigatório; cor + rótulo textual (nunca apenas cor — REQ-014)

- [x] **[F2-FRONT-002]** Implementar `OprStatusBadge` conforme D09 §2.5:
  - Sub-item: `DISPONIVEL` → `--status-available (#16A34A)`, texto "Disponível"
  - Sub-item: `EM_NEGOCIACAO` → `--status-negotiating (#D97706)`, texto "Em negociação"
  - Sub-item: `ENCERRADA` → `--status-closed (#737373)`, texto "Encerrada"

- [x] **[F2-FRONT-003]** Implementar módulo de Análise Individual `T-DC-005` inline no chat:
  - Sub-item: Card estruturado com: `OprStatusBadge`, código OPR, localização, tipologia, `CommissionCard` (Δ, Comissão, Custo Total), `RiskScoreBadge` com fatores listados, ROI 3 cenários com aviso obrigatório
  - Sub-item: Ações rápidas como chips: "Simular proposta" / "Comparar com similares" / "Salvar para alertas" (REQ-121)
  - Sub-item: 4 estados: Skeleton (enquanto carrega análise), Empty (impossível neste contexto), Error (OPR não encontrado — chips de oportunidades semelhantes), Populated (análise completa)
  - Sub-item: Oportunidade encerrada: badge cinza + mensagem + 3 chips de oportunidades semelhantes (REQ-013)

- [x] **[F2-FRONT-004]** Implementar Widget Top 3 no Dashboard `T-DC-009` conforme D06 + D08:
  - Sub-item: Seção "Oportunidades em Destaque" no Dashboard (REQ-125)
  - Sub-item: 3 cards: `OPR-XXXX`, Δ, comissão estimada, `RiskScoreBadge`, localização
  - Sub-item: Botão "Analisar →" em cada card → abre T-DC-003 com contexto OPR
  - Sub-item: Estado perfil incompleto: "Recomendações baseadas em dados gerais de mercado. Complete seu perfil para resultados personalizados." + link "Ir para Meu Perfil > Preferências" (REQ-027, D08 §2.8)
  - Sub-item: Estado sem oportunidades: "No momento não há oportunidades disponíveis." + botão "Ativar alertas"
  - Sub-item: 4 estados: Skeleton → Empty → Error → Populated

### Testes

- [x] **[F2-TEST-001]** Testes unitários de composição da análise:
  - Sub-item: Análise com delta > 0 → `base_calculo = "delta"`, `comissao = 20% × delta`
  - Sub-item: Análise com delta = 0 (edge case REQ-016) → `base_calculo = "valor_pago_cedente"`, `nota_fallback = true`
  - Sub-item: OPR com status `ENCERRADA` → retorna `opr_disponivel = false` + lista de 3 similares
  - Sub-item: Score indisponível → `score_disponivel = false`, mensagem de dados insuficientes

- [x] **[F2-TEST-002]** Testes E2E — análise individual via chat:
  - Sub-item: Abre T-DC-003 (via botão "Consultar Dani" na tela de oportunidade) → Dani inicia análise automaticamente
  - Sub-item: Card de análise exibe: Δ, Comissão, Custo Total, RiskScoreBadge com rótulo correto, ROI 3 cenários, aviso de projeção
  - Sub-item: Click em "Simular proposta" → exibe input de valor

### 🔀 Cross-Módulo

- [x] **[CROSS-001]** `AgenteModule` chama `CalculadoraService` via injeção de dependência (não import circular):
  - Sub-item: `CalculadoraModule` exporta `CalculadoraService`; `AgenteModule` importa `CalculadoraModule`
  - Sub-item: `CalculadoraModule` NÃO importa `AgenteModule` em nenhum arquivo
  - Sub-item: Teste: `CalculadoraService` pode ser instanciado independentemente sem `AgenteModule` presente

---

## 🔍 Auto-verificação S4 (12 checks)

| # | Check | Status |
|---|---|---|
| 1 | Todos os itens são binariamente verificáveis | ✅ |
| 2 | Nomes exatos: `CalculadoraService`, `dani:cache:calc:{opr_id}:{val_hash}`, `dani_contextos_opr`, `CommissionCard`, `RiskScoreBadge`, `OprStatusBadge` | ✅ |
| 3 | Valores numéricos: 20% comissão, ±20% ROI cenários, 0.80/1.20 fatores, TTL 300s cache, TTL 7 dias contexto OPR | ✅ |
| 4 | Fórmulas exatas: `delta = tabela_atual - tabela_contrato`; `comissao = 0.20 × delta (ou 0.20 × valor_pago_cedente)`; `custo_total = valor_proposta + comissao`; `roi = (tabela - custo_total) / custo_total × 100` | ✅ |
| 5 | Edge cases documentados: delta=0, delta<0, valor inválido, ROI negativo, score indisponível, OPR encerrado durante simulação | ✅ |
| 6 | Cores exatas dos tokens: risk-low #16A34A, risk-medium #D97706, risk-high #DC2626, status-available #16A34A, status-negotiating #D97706, status-closed #737373 | ✅ |
| 7 | ADR-003: `CalculadoraModule` sem import de `AgenteModule` — verificado em build + teste | ✅ |
| 8 | Aviso de projeção obrigatório: presente em todos os contextos de ROI | ✅ |
| 9 | Strings normativas D08 espelhadas literalmente (nota_fallback, aviso projeção, ROI negativo) | ✅ |
| 10 | Nenhum item scaffold — todos com sub-itens de lógica real | ✅ |
| 11 | Anti-scaffold: cobertura 100% unitária exigida em `CalculadoraService` (P0 crítico) | ✅ |
| 12 | REQs REQ-015 a REQ-017, REQ-021 a REQ-023 têm ≥1 item de checklist | ✅ |

---

## REQs cobertos por esta sprint

REQ-012, REQ-013, REQ-014, REQ-015, REQ-016, REQ-017, REQ-021, REQ-022, REQ-023, REQ-027, REQ-028 (parcial — Escrow e ZapSign em S6), REQ-078, REQ-087, REQ-121, REQ-125
