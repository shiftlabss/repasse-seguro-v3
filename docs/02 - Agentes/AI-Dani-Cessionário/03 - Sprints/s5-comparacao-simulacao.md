# S5 — Comparação de Oportunidades, Simulação e Portfólio

| **Sprint** | S5 — Comparação de Oportunidades, Simulação e Portfólio |
|---|---|
| **Template** | B — Módulo Fullstack (organizado por Feature com vertical slice Banco→Backend→Frontend→Wiring→Testes) |
| **REQs cobertos** | REQ-018, REQ-019, REQ-020, REQ-024, REQ-025, REQ-026, REQ-122, REQ-123, REQ-124 |
| **Docs fonte** | D01 (RN-DC-015, RN-DC-016, RN-DC-017, RN-DC-018, RN-DC-019, RN-DC-020), D09 (T-DC-003, T-DC-006, T-DC-007, T-DC-008), D16 (§7.1, §7.2) |
| **Total de itens** | 46 |

---

## 🎯 Objetivo

Implementar o módulo fullstack de comparação de até 5 oportunidades (T-DC-006), simulação de proposta individual (T-DC-007), simulação de contraproposta em negociação ativa (T-DC-008), análise de portfólio com capital distribuído (RN-DC-019) e simulação de impacto de variação de valorização (RN-DC-020). Todas as telas seguem o contrato de D09: 4 estados obrigatórios, 0 spinners, RBAC `CESSIONARIO_AUTHENTICATED`, acessibilidade WCAG 2.1 AA.

---

## Feature 1 — Comparação de Oportunidades (T-DC-006 + RN-DC-015)

### 🗄️ Banco de Dados

- [x] **[BANCO-COMP-001]** Verificar que `dani_contextos_opr` suporta array de OPR IDs para comparação:
  - Sub-item: Campo `opr_ids_contexto JSONB` em `dani_contextos_opr` aceita array de até 5 strings `OPR-XXXX-XXXX`
  - Sub-item: Constraint `CHECK (jsonb_array_length(opr_ids_contexto) <= 5)` presente ou validação em nível de aplicação documentada
  - Sub-item: Query `SELECT opr_ids_contexto FROM dani_contextos_opr WHERE conversa_id = :id AND cessionario_id = :cid` executa sem full scan (explain confirma uso de índice em `conversa_id + cessionario_id`)

---

### 🔧 Backend

- [x] **[BACK-COMP-001]** Implementar `OportunidadeService.compararOportunidades(opr_ids: string[], cessionario_id: string)` em `src/oportunidade/oportunidade.service.ts`:
  - Sub-item: Valida `opr_ids.length >= 2 && opr_ids.length <= 5`; se `length > 5` → lança `AgentException` código `AGENTE_010`, status 422, `user_message: "Consigo comparar até 5 oportunidades de uma vez. Qual grupo você gostaria de analisar primeiro?"`
  - Sub-item: Para cada OPR: chama `buscarPorOpr(opr_id, cessionario_id)` (isolamento REQ-003); se OPR não encontrado → remove da lista e registra no campo `opr_removidos[]` do response
  - Sub-item: Se após filtragem `opr_ids.length < 2` → lança `AgentException` código `AGENTE_011`, status 422, `user_message: "Não encontrei oportunidades suficientes disponíveis para comparar. Verifique os códigos OPR."`
  - Sub-item: Para cada OPR disponível: chama `CalculadoraService.calcularComissao(opr_id, tabela_atual)` para obter `delta`, `comissao`, `custo_total_escrow`, `roi`
  - Sub-item: Ordena resultado pelo critério recebido no parâmetro opcional `criterio`: `MELHOR_RETORNO` (maior ROI base), `MENOR_RISCO` (menor score_risco), `MELHOR_RELACAO` (default — menor score + maior delta combinados); informa critério utilizado no campo `criterio_aplicado` do response
  - Sub-item: Marca primeira posição com `is_melhor_opcao: true`; em desempate por mesmo score → maior Δ vence (RN-DC-015)
  - Sub-item: Retorna `CompararOportunidadesResponseDto` com campos: `oportunidades[]`, `criterio_aplicado`, `opr_removidos[]`, `total_comparadas`

- [x] **[BACK-COMP-002]** Implementar endpoint `POST /dani/comparar` em `AgenteController`:
  - Sub-item: DTO `CompararOportunidadesDto`: `opr_ids: string[]` (array 2–5 itens, cada item match regex `OPR-\d{4}-\d{4}`); `criterio?: 'MELHOR_RETORNO' | 'MENOR_RISCO' | 'MELHOR_RELACAO'`; `conversa_id?: string` (UUID)
  - Sub-item: Guards: `@UseGuards(JwtAuthGuard, CessionarioOwnerGuard)` — `CessionarioOwnerGuard` obrigatório (REQ-150)
  - Sub-item: `conversa_id` fornecido → persiste contexto de comparação em `dani_contextos_opr` (para histórico de conversa)
  - Sub-item: Response `200`: `{ data: CompararOportunidadesResponseDto }`; response `422`: `{ error: { code, message } }` para > 5 OPRs
  - Sub-item: ⚠️ AMBÍGUO REQ-089: adota path `/dani/comparar` conforme padrão D16 (prefixo `/dani` para todos os endpoints do domínio Agente)

---

### 🖥️ Frontend

- [x] **[FRONT-COMP-001]** Implementar componente `T-DC-006 — Módulo Comparação de Oportunidades` em `src/features/dani/comparacao/ComparacaoTable.tsx`:
  - Sub-item: **Estado Skeleton** — `<TableSkeleton rows={3} />` com colunas correspondentes (OPR, Δ, Comissão, Custo Total Escrow, Score Risco, Localização, Tipologia); não exibe enquanto `isLoading === true`
  - Sub-item: **Estado Error** — `<Alert variant="destructive">` + "Não foi possível carregar a comparação. Tente novamente." + `<Button>Tentar novamente</Button>` com `onClick={() => refetch()}`
  - Sub-item: **Estado Populated** — `<Table role="table" aria-label="Comparação de oportunidades">`; cabeçalhos `<TableHeader>` com `scope="col"`, fundo `--muted`, texto `--muted-foreground text-xs font-medium uppercase`; colunas: OPR, Δ, Comissão, Custo Total Escrow, Score Risco, Localização, Tipologia
  - Sub-item: Linha "Melhor opção" — `background: oklch(0.5 0.134 242.749 / 6%)` + `<Badge variant="default">Melhor opção</Badge>` em `--primary`; determinada pelo campo `is_melhor_opcao: true` do backend
  - Sub-item: Cada `<TableRow>` — `tabIndex={0}`, `hover:bg-accent`, `cursor: pointer`, `onKeyDown` com Enter chama `onRowClick(opr_id)` que abre `T-DC-005` da oportunidade correspondente
  - Sub-item: `<RiskScoreBadge>` por linha — cores exatas: 1–3 `#16A34A`, 4–6 `#D97706`, 7–10 `#DC2626`; `aria-label="Score de risco: [N] de 10 — risco [baixo/moderado/alto]"` (nunca apenas cor)
  - Sub-item: `<OprStatusBadge>` por linha — `DISPONIVEL` verde, `EM_NEGOCIACAO` laranja, `ENCERRADA` cinza
  - Sub-item: Mensagem limite: quando `opr_ids.length > 5` — bubble de sistema com `<Alert variant="default">` + string de D08: "Consigo comparar até 5 oportunidades de uma vez. Qual grupo você gostaria de analisar primeiro?"
  - Sub-item: Valores monetários formato `R$ XX.XXX,XX` (D08 formatação)
  - Sub-item: **Animação**: fade-in 300ms token `content` (D04); hover row transição `background-color` 150ms; sem animação de linhas individuais

- [x] **[FRONT-COMP-002]** Implementar acessibilidade WCAG 2.1 AA em `ComparacaoTable`:
  - Sub-item: `role="table"`, `aria-label="Comparação de oportunidades"` no elemento `<Table>`
  - Sub-item: Cabeçalhos `<th scope="col">` para cada coluna
  - Sub-item: Badge "Melhor opção": contraste `--primary-foreground` sobre `--primary` ≥ 4.5:1 — validado via axe-core
  - Sub-item: Linhas navegáveis via teclado: `tabIndex={0}` + `onKeyDown` Enter; screen reader anuncia estrutura tabular corretamente

---

### 🔌 Wiring

- [x] **[WIRE-COMP-001]** Conectar `ComparacaoTable` ao hook `useComparacaoOportunidades`:
  - Sub-item: Hook usa `useMutation` do TanStack Query para `POST /api/v1/dani/comparar`
  - Sub-item: `onSuccess`: atualiza cache de conversas se `conversa_id` presente; chama `onRowClick(opr_id)` para navegar para `T-DC-005`
  - Sub-item: `onError`: exibe estado Error com retry
  - Sub-item: `queryKey: ['dani', 'comparacao', opr_ids.join(',')]` — cache 300s (igual ao da Calculadora)

---

### ✅ Testes

- [x] **[TEST-COMP-001]** Testes unitários de `OportunidadeService.compararOportunidades`:
  - Sub-item: `opr_ids.length === 2` → retorna resultado com 2 linhas e `is_melhor_opcao: true` na primeira
  - Sub-item: `opr_ids.length === 5` → aceita sem erro; processa e retorna 5 linhas
  - Sub-item: `opr_ids.length === 6` → lança `AgentException AGENTE_010`, status 422
  - Sub-item: 1 OPR inexistente em lista de 3 → retorna 2 linhas + `opr_removidos: [opr_id_removido]`
  - Sub-item: 2 OPRs inexistentes em lista de 2 → lança `AgentException AGENTE_011`, status 422
  - Sub-item: Desempate: dois OPRs com mesmo score de risco → maior Δ vence (posição `is_melhor_opcao: true`)

- [x] **[TEST-COMP-002]** Testes de integração do endpoint `POST /dani/comparar`:
  - Sub-item: JWT válido + `opr_ids: [2 OPRs válidos]` → 200 com `total_comparadas: 2` e `is_melhor_opcao: true` em uma linha
  - Sub-item: JWT inválido → 401 `AUTH_001`
  - Sub-item: JWT de outro Cessionário → 403 `AUTH_009` (CessionarioOwnerGuard não mockado — REQ-149)
  - Sub-item: `opr_ids: [6 OPRs]` → 422 `AGENTE_010`

---

## Feature 2 — Simulação de Proposta (T-DC-007 + RN-DC-016, RN-DC-017)

### 🔧 Backend

- [x] **[BACK-SIM-001]** Implementar `AgenteService.simularProposta(opr_id, valor_proposta, cessionario_id)` em `src/agente/agente.service.ts`:
  - Sub-item: Valida `valor_proposta > 0` e é número; se inválido → lança `CalculadoraException` código `CALC_001`, status 400, `user_message: "O valor informado não é válido. Por favor, informe um valor em reais maior que zero para a simulação."`
  - Sub-item: Chama `CalculadoraService.calcularComissao(opr_id, valor_proposta)` — ADR-003: `AgenteModule` pode chamar `CalculadoraService` via injeção, mas `CalculadoraModule` NUNCA importa `AgenteModule`
  - Sub-item: Monta `SimulacaoPropostaResponseDto`: `opr_id`, `valor_proposta`, `delta`, `comissao`, `formula_aplicada` (`20_PERCENT_DELTA` | `20_PERCENT_VALOR_PAGO`), `custo_total_escrow`, `roi: { conservador, base, otimista }` (cenários ×0.80 / ×1.00 / ×1.20), `disclaimer` (string normativa REQ-017)
  - Sub-item: `disclaimer` obrigatório: "Esses são valores projetados com base nos dados disponíveis. Resultados reais podem variar conforme condições de mercado." — nunca omitido (RN-DC-017)
  - Sub-item: Persiste simulação no histórico da conversa se `conversa_id` presente

- [x] **[BACK-SIM-002]** Reutilizar endpoint `POST /calculadora/calcular` (já implementado em S4) como base determinística — `simularProposta` é wrapper com persistência e contexto de conversa:
  - Sub-item: Cache Redis `dani:cache:calc:{opr_id}:{valor_hash}` TTL 300s verificado antes de chamar `CalculadoraService` diretamente
  - Sub-item: `source` do response: `"cache"` se resultado veio do Redis; `"computed"` se calculado agora
  - Sub-item: `nota_fallback` incluída no response quando `fallback_ativo: true` (agente indisponível)

---

### 🖥️ Frontend

- [x] **[FRONT-SIM-001]** Implementar componente `T-DC-007 — Módulo Simulação de Proposta` em `src/features/dani/simulacao/SimulacaoProposta.tsx`:
  - Sub-item: **Estado Empty** — `<Input variant="default" type="text" inputMode="decimal" placeholder="R$ 0,00" aria-label="Valor da proposta" />` + `<Button variant="default" aria-label="Simular proposta">Simular</Button>`; foco automático no `Input` ao abrir módulo
  - Sub-item: **Estado Skeleton** — 5 `<Skeleton className="h-4 w-full rounded" />` dentro do `CommissionCard` enquanto cálculo processa (isLoading === true)
  - Sub-item: **Estado Error** — `<Alert variant="destructive">` abaixo do input: mensagem específica por tipo (valor inválido, zero, negativo); input permanece ativo para nova tentativa
  - Sub-item: **Estado Populated** — `<CommissionCard>` com Δ, Comissão (fórmula usada `formula_aplicada`), Custo Total Escrow, ROI 3 cenários (Conservador ×0.80 / Base / Otimista ×1.20) + disclaimer + quick actions
  - Sub-item: `<ROIDeltaBadge>` — seta + valor de variação; cor `--roi-positive` se ROI > 0, cor `--roi-negative` se ROI < 0; `aria-label` textual obrigatório (nunca apenas cor — WCAG)
  - Sub-item: Disclaimer ROI obrigatório em `text-xs`, fundo `--muted`, cor `--muted-foreground`, ícone info — nunca ocultável (RN-DC-017, REQ-017)
  - Sub-item: `nota_fallback` — exibe `AgentStatusBanner` estado `FALLBACK` (`--agent-fallback #0069A8`) quando `fallback_ativo: true` ⚠️ AMBÍGUO REQ-126
  - Sub-item: Quick actions: `<Button variant="outline" size="sm">Simular outro valor</Button>` (reseta para Empty + foca Input) + `<Button variant="outline" size="sm">Ir para a oportunidade</Button>` (navega para fora do módulo Dani)
  - Sub-item: `Enter` no Input dispara simulação; máximo 1000 chars por mensagem (REQ-027); `Shift+Enter` não aplicável aqui (campo numérico)
  - Sub-item: **Animação**: fade-in 250ms token `component` (D04); resultado da simulação fade-in 300ms após cálculo; `useReducedMotion()` desabilita animação do resultado

- [x] **[FRONT-SIM-002]** Acessibilidade WCAG 2.1 AA em `SimulacaoProposta`:
  - Sub-item: `role="form"`, `aria-label="Simulação de proposta"` no container
  - Sub-item: `aria-live="polite"` no container do resultado — screen reader anuncia resultado ao calcular
  - Sub-item: ROI positivo/negativo: cor + seta indicativa (nunca apenas cor)
  - Sub-item: Após simular: foco move para o primeiro dado do resultado (Δ ou comissão)

---

### 🔌 Wiring

- [x] **[WIRE-SIM-001]** Conectar `SimulacaoProposta` ao hook `useSimulacaoProposta`:
  - Sub-item: Hook usa `useMutation` do TanStack Query para `POST /api/v1/calculadora/calcular`
  - Sub-item: Props: `opr_id: string`, `conversa_id?: string`, `onSuccess?: (result) => void`
  - Sub-item: `mutateAsync(valor_proposta)` — chama endpoint; loading state controla skeleton do `CommissionCard`
  - Sub-item: Quick action "Simular outro valor": chama `reset()` do `useMutation` e foca o `Input` via `useRef`
  - Sub-item: Navegação "Ir para a oportunidade": `navigate('/oportunidades/:opr_id')` via TanStack Router

---

### ✅ Testes

- [x] **[TEST-SIM-001]** Testes unitários de `AgenteService.simularProposta`:
  - Sub-item: `valor_proposta = 350000, delta = 80000` → `comissao = 16000` (20% × 80000), `formula_aplicada = "20_PERCENT_DELTA"`, disclaimer presente
  - Sub-item: `valor_proposta = 300000, delta = -10000, valor_pago_cedente = 250000` → `comissao = 50000` (20% × 250000), `formula_aplicada = "20_PERCENT_VALOR_PAGO"`, nota explicativa presente
  - Sub-item: `valor_proposta = 0` → lança `CalculadoraException CALC_001`, status 400
  - Sub-item: `valor_proposta = -1` → lança `CalculadoraException CALC_001`, status 400
  - Sub-item: `valor_proposta = "abc"` (string não-numérica) → lança `CalculadoraException CALC_001`, status 400
  - Sub-item: ROI cenários: `valor_proposta = 300000, tabela_atual = 430000, custo_total = 316000` → conservador = `(430000×0.80 − 316000) / 316000 × 100`, base = `(430000 − 316000) / 316000 × 100`, otimista = `(430000×1.20 − 316000) / 316000 × 100`

- [x] **[TEST-SIM-002]** Testes de componente `SimulacaoProposta`:
  - Sub-item: Render inicial → estado Empty; Input com foco; botão "Simular" presente
  - Sub-item: Input com valor inválido "0" → estado Error; `Alert variant="destructive"` visível; input permanece ativo
  - Sub-item: Input com valor válido "350000" + submit → estado Skeleton durante loading; estado Populated com CommissionCard após response
  - Sub-item: Disclaimer obrigatório visível em Populated — teste falha se string normativa estiver ausente
  - Sub-item: Clique em "Simular outro valor" → estado Empty com foco no Input

---

## Feature 3 — Simulação de Contraproposta (T-DC-008 + RN-DC-018)

### 🔧 Backend

- [x] **[BACK-CONTRA-001]** Implementar `AgenteService.simularContraproposta(opr_id, valor_contraproposta, valor_proposta_anterior, cessionario_id)`:
  - Sub-item: Valida existência de negociação ativa para `opr_id` do `cessionario_id` — se não existe → lança `AgentException` código `AGENTE_012`, status 422, `user_message: "Não encontrei uma negociação ativa para simular a contraproposta. Quer que eu simule uma proposta inicial para alguma oportunidade disponível?"`
  - Sub-item: Valida `valor_contraproposta > 0`; se inválido → `CALC_001`, status 400
  - Sub-item: Calcula nova comissão, novo custo total Escrow e ROI 3 cenários para `valor_contraproposta`
  - Sub-item: Calcula `diferenca_absoluta = valor_proposta_anterior - valor_contraproposta`; `diferenca_percentual = diferenca_absoluta / valor_proposta_anterior × 100`
  - Sub-item: `ROIDeltaBadge direction`: `"down"` se `diferenca_absoluta > 0` (economia = seta verde), `"up"` se `diferenca_absoluta < 0` (acréscimo = seta vermelha)
  - Sub-item: Retorna `SimulacaoContrapropostaResponseDto`: todos os campos de `SimulacaoPropostaResponseDto` + `diferenca_absoluta`, `diferenca_percentual`, `roi_delta_direction`, `disclaimer`, `proximo_passo: "Quando decidir o valor, acesse a tela de negociação na plataforma para submeter a contraproposta."` (string normativa D01 RN-DC-018)

---

### 🖥️ Frontend

- [x] **[FRONT-CONTRA-001]** Implementar componente `T-DC-008 — Módulo Simulação de Contraproposta` em `src/features/dani/simulacao/SimulacaoContraproposta.tsx`:
  - Sub-item: **Estado Empty** — `<Input>` pré-preenchido com `valor_proposta_atual` (prop recebida de T-DC-004); foco no Input ao abrir módulo; botão "Simular contraproposta"
  - Sub-item: **Estado Skeleton** — 6 `<Skeleton className="h-4 w-full rounded" />` durante cálculo
  - Sub-item: **Estado Error** — `<Alert variant="destructive">` inline para valor inválido ou negativo
  - Sub-item: **Estado Populated** — `<CommissionCard>` com nova comissão, novo Escrow, ROI ajustado; `<ROIDeltaBadge>` mostrando diferença vs proposta anterior: seta ↓ verde `--roi-positive` se economia, seta ↑ vermelho `--roi-negative` se acréscimo; valor absoluto + percentual
  - Sub-item: Próximo passo obrigatório: texto "Quando decidir o valor, acesse a tela de negociação na plataforma para submeter a contraproposta." + `<Button variant="default" size="sm">Ir para tela de negociação</Button>`
  - Sub-item: Disclaimer ROI obrigatório e não ocultável (mesmo texto de T-DC-007)
  - Sub-item: `ROIDeltaBadge` — seta + valor numérico absoluto + percentual (nunca apenas cor — WCAG); `aria-label="Diferença: [valor] ([percentual]%) — [economia/acréscimo] em relação à proposta anterior"`
  - Sub-item: **Animação**: idêntico ao T-DC-007; `useReducedMotion()` obrigatório

- [x] **[FRONT-CONTRA-002]** Acessibilidade WCAG 2.1 AA em `SimulacaoContraproposta`:
  - Sub-item: `aria-live="polite"` no resultado; screen reader anuncia resultado ao calcular
  - Sub-item: `ROIDeltaBadge`: seta + valor numérico (nunca apenas cor); contraste validado via axe-core
  - Sub-item: Foco no Input ao montar componente; após simular → foco no resultado

---

### 🔌 Wiring

- [x] **[WIRE-CONTRA-001]** Integrar T-DC-008 ao T-DC-004 (Chat com Contexto de Negociação):
  - Sub-item: `SimulacaoContraproposta` é montado automaticamente quando `T-DC-004` abre — prop `valor_proposta_atual` vem do contexto da negociação
  - Sub-item: Botão "Ir para tela de negociação" — `navigate('/negociacoes/:negotiation_id')` via TanStack Router
  - Sub-item: `conversa_id` propagado para persistência no histórico de conversa

---

### ✅ Testes

- [x] **[TEST-CONTRA-001]** Testes unitários de `AgenteService.simularContraproposta`:
  - Sub-item: Contraproposta menor que proposta anterior → `diferenca_absoluta > 0`, `roi_delta_direction = "down"` (economia/verde)
  - Sub-item: Contraproposta maior que proposta anterior → `diferenca_absoluta < 0`, `roi_delta_direction = "up"` (acréscimo/vermelho)
  - Sub-item: Sem negociação ativa para o `opr_id` → lança `AgentException AGENTE_012`, status 422
  - Sub-item: `valor_contraproposta = 0` → lança `CalculadoraException CALC_001`, status 400
  - Sub-item: `proximo_passo` presente no response (string normativa de D01 RN-DC-018)

---

## Feature 4 — Portfólio e Variação de Valorização (RN-DC-019, RN-DC-020)

### 🔧 Backend

- [x] **[BACK-PORT-001]** Implementar `AgenteService.analisarPortfolio(opr_ids: string[], capital_disponivel: number, cessionario_id: string)`:
  - Sub-item: Para cada OPR: chama `CalculadoraService.calcularComissao` para obter `custo_total_escrow` individual
  - Sub-item: Soma `capital_total_necessario = sum(custo_total_escrow)` de todas as OPRs disponíveis
  - Sub-item: Compara com `capital_disponivel`: se suficiente → `status: "SUFICIENTE"`; se insuficiente → `status: "INSUFICIENTE"` com `deficit = capital_total_necessario - capital_disponivel` (destacado no response); se excedente → `status: "EXCEDENTE"` com `excedente = capital_disponivel - capital_total_necessario`
  - Sub-item: Calcula ROI projetado para o portfólio completo (soma dos ROIs base) e ROI individual por OPR
  - Sub-item: Se alguma OPR não está mais disponível: remove, recalcula com as restantes, inclui `opr_removidos[]` no response (RN-DC-019)
  - Sub-item: Quick action `"RANQUEAR_POR_PRIORIDADE"` sugerida quando `status === "INSUFICIENTE"` — instrução para o frontend exibir o botão correspondente
  - Sub-item: Retorna `AnalisePortfolioResponseDto`: `oportunidades[]`, `capital_total_necessario`, `capital_disponivel`, `status`, `deficit?`, `excedente?`, `roi_portfolio_base`, `roi_individual[]`, `opr_removidos[]`, `acao_sugerida?`

- [x] **[BACK-PORT-002]** Implementar `AgenteService.simularVariacaoValorizacao(opr_id, percentual_variacao, cessionario_id)`:
  - Sub-item: Valida `percentual_variacao` é número (pode ser negativo para desvalorização)
  - Sub-item: Calcula novo ROI aplicando `tabela_atual × (1 + percentual_variacao / 100)` como nova `tabela_atual_ajustada`
  - Sub-item: Se resultado for negativo → inclui `roi_negativo: true` e `user_message: "Com uma desvalorização de [X]%, o retorno desta oportunidade seria negativo. O investimento resultaria em uma perda estimada de R$ [valor]."` (string normativa RN-DC-020)
  - Sub-item: Disclaimer obrigatório: "Esta é uma projeção baseada nos dados disponíveis. Resultados reais podem variar." (RN-DC-020)

---

### 🖥️ Frontend

- [x] **[FRONT-PORT-001]** Implementar componente `PortfolioAnalysis` em `src/features/dani/portfolio/PortfolioAnalysis.tsx`:
  - Sub-item: **Estado Skeleton** — skeletons para `capital_total_necessario`, `status`, ROI total + 3 linhas de ROI individual
  - Sub-item: **Estado Error** — `<Alert variant="destructive">` + "Não foi possível calcular a análise de portfólio." + retry
  - Sub-item: **Estado Populated** — `capital_total_necessario` em destaque; badge de `status`: verde `SUFICIENTE`, vermelho `INSUFICIENTE` com `deficit` em negrito + cor de alerta, azul `EXCEDENTE`; ROI do portfólio completo + tabela de ROI individual por OPR
  - Sub-item: `status === "INSUFICIENTE"` → `<Button variant="default" size="sm">Ranquear por prioridade</Button>` exibido proativamente (RN-DC-019)
  - Sub-item: `opr_removidos.length > 0` → alerta inline: "A seguinte oportunidade foi removida por não estar mais disponível: [OPR-XXXX-XXXX]. Recálculo realizado com as demais."
  - Sub-item: Disclaimer obrigatório exibido no rodapé do componente

- [x] **[FRONT-PORT-002]** Implementar componente `VariacaoValorizacao` em `src/features/dani/portfolio/VariacaoValorizacao.tsx`:
  - Sub-item: **Estado Empty** — input percentual + botão "Simular variação"; placeholder "Ex: 10 (positivo) ou -5 (negativo)"
  - Sub-item: **Estado Populated** — novo ROI com `ROIDeltaBadge`; se `roi_negativo: true` → exibe `user_message` normativa em `<Alert variant="destructive">` com valor da perda estimada
  - Sub-item: Disclaimer obrigatório e não ocultável (RN-DC-020)

---

### 🔌 Wiring

- [x] **[WIRE-PORT-001]** Integrar `PortfolioAnalysis` ao sistema de chat:
  - Sub-item: Montado como card inline quando `AgenteService` detecta intenção de análise de portfólio (intent classification no sistema de prompt)
  - Sub-item: Props: `opr_ids: string[]`, `capital_disponivel: number`, `conversa_id: string`
  - Sub-item: Botão "Ranquear por prioridade": chama `compararOportunidades(opr_ids, criterio: 'MELHOR_RELACAO')` e exibe `T-DC-006`

---

### ✅ Testes

- [x] **[TEST-PORT-001]** Testes unitários de `AgenteService.analisarPortfolio`:
  - Sub-item: `capital_disponivel = 800000`, `custo_total_3_oprs = 700000` → `status: "SUFICIENTE"`, `excedente: 100000`
  - Sub-item: `capital_disponivel = 500000`, `custo_total_3_oprs = 700000` → `status: "INSUFICIENTE"`, `deficit: 200000`, `acao_sugerida: "RANQUEAR_POR_PRIORIDADE"`
  - Sub-item: 1 OPR removida por estar encerrada → `opr_removidos: [opr_id]`; recálculo com OPRs restantes retorna valores corretos
  - Sub-item: `roi_portfolio_base` = média ponderada dos ROI base individuais

- [x] **[TEST-PORT-002]** Testes unitários de `AgenteService.simularVariacaoValorizacao`:
  - Sub-item: `percentual_variacao = 10` → ROI positivo calculado corretamente; `roi_negativo: false`
  - Sub-item: `percentual_variacao = -50` e custo > tabela_atual ajustada → `roi_negativo: true`; `user_message` normativa presente com valor de perda estimada
  - Sub-item: Disclaimer obrigatório presente no response em todos os cenários

---

## 🔍 Auto-verificação S5 (12 checks)

| # | Check | Status |
|---|---|---|
| 1 | Todos os itens são binariamente verificáveis | ✅ |
| 2 | Nomes exatos: `ComparacaoTable`, `SimulacaoProposta`, `SimulacaoContraproposta`, `PortfolioAnalysis`, `VariacaoValorizacao`, `compararOportunidades`, `simularProposta`, `simularContraproposta`, `analisarPortfolio`, `simularVariacaoValorizacao` | ✅ |
| 3 | Valores numéricos: máx 5 OPRs, TTL cache 300s, ROI ×0.80/×1.00/×1.20, rate limit 30 msgs/hora, 1000 chars máx | ✅ |
| 4 | Disclaimer ROI obrigatório em T-DC-007, T-DC-008, portfólio e variação — nunca ocultável (RN-DC-017, RN-DC-020) | ✅ |
| 5 | Glossário D10: Δ, Escrow, Comissão Comprador, Score de Risco, ROI — todos os termos com definição exata | ✅ |
| 6 | Máquina de estado de simulação documentada em D01 §8 (SimulacaoIniciada→CalculoRealizado→ResultadoExibido→PropostaSubmetida|SimulacaoEncerrada) mapeada nos contratos de UI | ✅ |
| 7 | ADR-003 verificado: `AgenteModule` chama `CalculadoraService` via injeção; `CalculadoraModule` nunca importa `AgenteModule` | ✅ |
| 8 | ⚠️ AMBÍGUO REQ-089 (path `/dani/comparar`) e REQ-126 (`--agent-fallback #0069A8`) documentados e adotados valor normativo D16/D09 | ✅ |
| 9 | Nenhuma suposição sem base documental | ✅ |
| 10 | Nenhum item de scaffold — todos com sub-itens verificáveis (R10) | ✅ |
| 11 | `CessionarioOwnerGuard` não pode ser mockado em testes de integração de endpoints (REQ-149) | ✅ |
| 12 | REQs REQ-018–020, REQ-024–026, REQ-122–124 têm ≥1 item de checklist | ✅ |

---

## REQs cobertos por esta sprint

REQ-018 (comparação até 5 OPRs + desempate + OPR removida), REQ-019 (portfólio multi-OPR + déficit + ranking), REQ-020 (variação de valorização + ROI negativo), REQ-024 (simulação de proposta + ROI 3 cenários), REQ-025 (simulação de contraproposta + diferença vs anterior), REQ-026 (portfólio + capital insuficiente + ranking por prioridade), REQ-122 (T-DC-006 tabela comparativa 4 estados WCAG), REQ-123 (T-DC-007 simulação proposta inline 4 estados), REQ-124 (T-DC-008 simulação contraproposta inline 4 estados)
