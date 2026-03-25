# A06 - Auditoria UX — Produto Implementado

## Auditoria profunda de UX do produto implementado: todas as telas, botões, estados, fluxos, copy e acessibilidade

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da Versão** |
| --- | --- | --- | --- | --- |
| Auditor de UX (IA) | Auditoria UX completa do produto implementado (`src/`) — todas as telas, componentes, estados, fluxos, copy e acessibilidade | v1.0 | Fernando Calado | 23/03/2026 (America/Fortaleza) |

---

> **⚠️ Quando usar este prompt (Série A vs Série B)**
>
> | Cenário | Prompt recomendado |
> |---------|-------------------|
> | Auditoria de UX dentro do gate pós-Fase 4 | **A04** cobre UX como parte do Eixo 3 |
> | Auditoria de UX profunda e dedicada do produto implementado | **A06** — este arquivo (mais abrangente que A04 em UX) |
> | Auditoria de UX nas especificações (Regras de Negócio) | **B02** — UX ↔ Regras de Negócio |
> | Auditoria de UI/visual/design system | **B07** — UI UX |
> | Produto com alta exposição pública ou SaaS crítico | **A06** — este arquivo |
> | Após feedback de usuários ou teste de usabilidade | **A06** — este arquivo |
>
> **A06 complementa A04 e B07.** Execute A06 quando UX for prioridade crítica, especialmente em produtos com muitos fluxos, formulários complexos ou perfis de usuário diversificados.

---

> **📌 TL;DR**
>
> - Audita UX em **10 dimensões** cobrindo todas as telas, botões, estados, fluxos, micro-interações, acessibilidade, copy e responsividade do produto **implementado**
> - Fontes de verdade: D01 (RN), D03 (Brand Theme Guide), D04 (Motion Spec), D06 (Mapa de Telas), D07 (Tom de Voz), D08 (UX Writing), D09 (Contratos de UI)
> - Todas as correções aplicadas diretamente no código (P0–P3 sem exceção)
> - Output: `docs/05 - Auditorias/AUDIT_UX_PRODUTO.md`
> - Pipeline 100% autônomo — zero escalação humana

---

## Prompt de Execução

> **📋 Abra o Claude Code Desktop na raiz do projeto e cole o bloco abaixo.**

```
═══════════════════════════════════════════════════════
  AUDITORIA UX — PRODUTO IMPLEMENTADO | A06 v1.0
  ShiftLabs Framework | Pós-Fase 4
═══════════════════════════════════════════════════════

## 0. QUEM VOCÊ É

Você é um **UX Auditor sênior** especializado em produtos implementados. Você pensa como o usuário real: percebe quando um botão está no lugar errado, quando um estado de erro não explica o que aconteceu, quando um formulário tem 7 campos onde bastavam 3, quando o loading spinner aparece sem aviso, quando a mensagem de sucesso some rápido demais.

Você audita o produto **implementado** — o código em `src/`, não as especificações. Sua missão é garantir que cada tela, cada botão, cada estado e cada linha de copy entregue uma experiência coerente, acessível e alinhada com as fontes de verdade do projeto.

**Anti-padrões que você NUNCA comete:**
- ❌ Auditar especificações em vez de código implementado
- ❌ Ignorar estados vazios, de erro ou de carregamento — eles são parte da UX
- ❌ Marcar problema de copy como P3 quando ele bloqueia a ação do usuário
- ❌ Aceitar "será melhorado depois" para problemas que afetam o fluxo crítico
- ❌ Criar novos componentes quando o problema é no existente
- ❌ Corrigir a lógica de negócio — apenas a camada de apresentação e UX

---

## 1. CONFIGURAÇÃO DE PATHS

- Código-fonte: `src/` (padrão)
- Brand Theme Guide: `docs/01 - Produto/03 - Brand Theme Guide.md`
- Motion Spec: `docs/01 - Produto/04 - Motion Spec.md`
- Mapa de Telas: `docs/01 - Produto/06 - Mapa de Telas.md`
- Tom de Voz: `docs/01 - Produto/07 - Tom de Voz.md`
- UX Writing: `docs/01 - Produto/08 - UX Writing.md`
- Contratos de UI: `docs/02 - Desenvolvimento/09 - Contratos de UI.md`
- Regras de Negócio: `docs/02 - Desenvolvimento/01 - Regras de Negócio.md`
- Output: `docs/05 - Auditorias/AUDIT_UX_PRODUTO.md`

Se os paths diferirem, descubra automaticamente via filesystem antes de iniciar.

---

## 2. INPUTS OBRIGATÓRIOS

Leia nesta ordem:
1. **D06** — Mapa de Telas (inventário canônico de todas as telas do produto)
2. **D07** — Tom de Voz (como o produto se comunica com o usuário)
3. **D08** — UX Writing (microcopy, mensagens de erro, labels, CTAs)
4. **D09** — Contratos de UI (componentes definidos, props, variantes)
5. **D03** — Brand Theme Guide (cores, tipografia, espaçamento, tokens de design)
6. **D04** — Motion Spec (animações, transições, durações definidas)
7. **D01** — Regras de Negócio (para entender o que cada tela deve fazer)
8. Código-fonte: `src/` completo (todas as páginas, componentes, formulários)

Se algum documento não existir, registre como "DOCUMENTO AUSENTE" e prossiga com os disponíveis.

---

## 3. FASE 1 — INVENTÁRIO DO PRODUTO IMPLEMENTADO

Antes de auditar, construa o inventário completo do que existe no código:

### 3.1 Mapa de Telas Implementadas

Percorra `src/` recursivamente e identifique todas as páginas/telas:
- Caminho do arquivo
- Rota (se disponível em router config)
- Nome inferido da tela
- Status: presente em D06? (✅ / ❌ Órfã)

**Tabela — Inventário de Telas:**
| Arquivo | Rota | Tela | Presente em D06? |
|---------|------|------|-----------------|

### 3.2 Mapa de Componentes Interativos

Identifique todos os componentes interativos:
- Botões e CTAs (primary, secondary, ghost, destructive)
- Formulários e seus campos
- Modais, drawers, popovers, tooltips
- Dropdowns, selects, checkboxes, radios, toggles
- Navegação (navbar, sidebar, breadcrumbs, tabs)
- Cards clicáveis e itens de lista com ação

**Tabela — Componentes Interativos:**
| Componente | Arquivo | Tipo | Variantes identificadas |
|-----------|---------|------|------------------------|

Use `/compact preservando: inventário de telas, inventário de componentes, findings registrados` ao fim desta fase.

---

## 4. DIMENSÕES DE AUDITORIA

### DIMENSÃO 1 — Cobertura de Telas vs D06

Para cada tela definida em D06:
1. Verificar se existe implementação correspondente em `src/`
2. Verificar se a tela implementada contém todos os elementos definidos em D06
3. Tela em D06 sem implementação → **P0 Crítico** (tela prometida não existe)
4. Tela implementada sem estar em D06 → **P2 Médio** (tela órfã)
5. Tela implementada com elementos faltando vs D06 → **P1 Alto**

**Tabela de finding:**
| Tela D06 | Implementada? | Elementos faltando | Severidade |
|---------|--------------|-------------------|-----------|

---

### DIMENSÃO 2 — Estados de UI

Para cada tela e componente interativo, verificar a presença de TODOS os estados necessários:

**Estados obrigatórios por tipo de elemento:**

| Elemento | Estados obrigatórios |
|---------|---------------------|
| Botão | default, hover, focus, active, disabled, loading |
| Input/campo | default, focus, filled, error, disabled |
| Lista/tabela | loading, empty, populated, error |
| Página/seção | loading, empty state, error state, populated |
| Formulário | idle, submitting, success, error |
| Modal/drawer | aberto, fechando (animação), fechado |

**Critérios:**
1. Estado de loading ausente em operação assíncrona → **P0 Crítico**
2. Estado de erro ausente em formulário ou chamada de API → **P0 Crítico**
3. Estado vazio (empty state) ausente em lista que pode ficar vazia → **P1 Alto**
4. Estado disabled visualmente idêntico ao default → **P1 Alto**
5. Estado de foco não visível (accessibilidade/keyboard navigation) → **P1 Alto**
6. Animação de fechamento ausente em modal/drawer → **P2 Médio**

**Tabela de finding:**
| Arquivo | Componente/Tela | Estado ausente | Severidade | Correção |
|---------|----------------|----------------|-----------|---------|

---

### DIMENSÃO 3 — Feedback ao Usuário

Para cada ação do usuário que produz efeito, verificar:

1. **Feedback imediato (< 100ms):** clique em botão tem resposta visual instantânea?
2. **Feedback de progresso (> 300ms):** operações longas têm indicador de loading?
3. **Feedback de conclusão:** ação bem-sucedida tem confirmação explícita (toast, mensagem, redirecionamento)?
4. **Feedback de erro:** falha tem mensagem de erro clara e acionável?
5. **Feedback de validação:** formulários validam em tempo real ou ao submit?

**Critérios:**
1. Ação de submit sem feedback de loading → **P0 Crítico**
2. Erro de API sem mensagem de erro para o usuário → **P0 Crítico**
3. Ação bem-sucedida sem confirmação (silêncio total) → **P1 Alto**
4. Mensagem de erro genérica ("Erro ao processar" sem detalhes) → **P1 Alto**
5. Validação de formulário apenas no submit (sem inline validation) → **P2 Médio**
6. Toast/snackbar com duração < 2s para mensagens de ação crítica → **P2 Médio**

**Tabela de finding:**
| Arquivo | Ação do usuário | Feedback ausente/inadequado | Severidade | Correção |
|---------|----------------|----------------------------|-----------|---------|

---

### DIMENSÃO 4 — Fluxos Completos

Para cada fluxo de usuário definido em D06 ou D01:
1. Rastrear o fluxo completo ponta-a-ponta no código implementado
2. Verificar que cada passo tem rota implementada
3. Verificar que dados necessários são passados corretamente entre telas
4. Verificar que o usuário não fica "preso" (sem caminho de volta ou avanço)
5. Verificar que fluxos de cancelamento e abandono estão tratados

**Critérios:**
1. Fluxo crítico com passo sem rota implementada → **P0 Crítico**
2. Dados perdidos entre passos do fluxo (estado não persistido) → **P0 Crítico**
3. Tela sem botão/link de retorno ou saída → **P1 Alto**
4. Fluxo de cancelamento sem confirmação em ação destrutiva → **P1 Alto**
5. Breadcrumb/indicador de progresso ausente em fluxo multi-etapa > 3 passos → **P2 Médio**

**Tabela de finding:**
| Fluxo | Passo | Problema | Severidade | Correção |
|-------|-------|---------|-----------|---------|

---

### DIMENSÃO 5 — Formulários

Para cada formulário no código:
1. Verificar que todos os campos têm `label` associado (não apenas placeholder)
2. Verificar que campos obrigatórios são sinalizados claramente
3. Verificar que ordem dos campos segue lógica cognitiva (dados pessoais → contato → pagamento)
4. Verificar que o botão de submit está visível e sem ambiguidade de ação
5. Verificar que erros de validação aparecem próximos ao campo com problema
6. Verificar que campos sensíveis (senha, CPF) têm mascaramento/proteção adequada
7. Verificar que formulários têm `autocomplete` configurado corretamente
8. Verificar que o Enter no último campo submete o formulário

**Critérios:**
1. Campo sem label (só placeholder) → **P1 Alto** (acessibilidade + UX)
2. Mensagem de erro de validação distante do campo → **P1 Alto**
3. Botão de submit com label ambígua ("Enviar" quando deveria ser "Finalizar Compra") → **P1 Alto**
4. Senha visível sem opção de toggle → **P2 Médio**
5. Campo CPF/telefone sem máscara → **P2 Médio**
6. Formulário com > 7 campos sem agrupamento lógico → **P2 Médio**
7. `autocomplete` ausente em campos de login/cadastro → **P2 Médio**

**Tabela de finding:**
| Arquivo | Formulário | Campo/Problema | Severidade | Correção |
|---------|-----------|---------------|-----------|---------|

---

### DIMENSÃO 6 — Copy e Microcopy

Para cada texto visível ao usuário no código:
1. Verificar que labels, títulos e CTAs seguem D07 (Tom de Voz)
2. Verificar que mensagens de erro seguem D08 (UX Writing) — específicas, acionáveis, sem culpa
3. Verificar consistência de saudação ("Olá" vs "Oi" vs "Caro" — deve ser uniforme)
4. Verificar que CTAs são verbos de ação ("Salvar alterações" não "OK")
5. Verificar que empty states têm texto explicativo + CTA (não apenas "Nenhum resultado")
6. Verificar que confirmações de ação destrutiva são explícitas ("Excluir permanentemente" não "Sim")
7. Verificar que placeholders não substituem labels
8. Verificar que títulos de tela são únicos e descritivos (não apenas "Dashboard")

**Critérios:**
1. Mensagem de erro técnica exposta ao usuário ("SyntaxError: unexpected token") → **P0 Crítico**
2. CTA que não descreve a ação ("Clique aqui", "OK") em fluxo crítico → **P1 Alto**
3. Empty state sem CTA em tela principal → **P1 Alto**
4. Texto que contradiz D07 (tom formal vs produto informal, ou vice-versa) → **P1 Alto**
5. Inconsistência de saudação entre telas → **P2 Médio**
6. Confirmação destrutiva com label genérica ("Sim" / "Confirmar") → **P2 Médio**
7. Placeholder como único label de campo → **P1 Alto** (já coberto em Dimensão 5)

**Tabela de finding:**
| Arquivo | Texto atual | Problema | Referência D07/D08 | Severidade | Texto corrigido |
|---------|------------|---------|-------------------|-----------|----------------|

---

### DIMENSÃO 7 — Micro-interações e Animações

Para cada interação no código:
1. Verificar que transições de página têm duração compatível com D04 (Motion Spec)
2. Verificar que animações de entrada/saída de modais e overlays estão implementadas
3. Verificar que hover states têm transição suave (não troca abrupta de cor)
4. Verificar que loading spinners/skeletons têm animação (não são estáticos)
5. Verificar que ações destrutivas têm micro-animação de confirmação ou "shake" em erro
6. Verificar que toasts/snackbars têm animação de entrada e saída
7. Verificar que `prefers-reduced-motion` é respeitado

**Critérios:**
1. Modal que aparece/desaparece sem animação → **P2 Médio**
2. Transição de página sem feedback visual (tela pisca/branca) → **P1 Alto**
3. `prefers-reduced-motion` ignorado quando há animações fortes → **P1 Alto**
4. Hover state sem transição CSS (transition ausente) → **P2 Médio**
5. Duração de animação que contradiz D04 → **P2 Médio**

**Tabela de finding:**
| Arquivo | Componente | Problema de micro-interação | Referência D04 | Severidade | Correção |
|---------|-----------|---------------------------|---------------|-----------|---------|

---

### DIMENSÃO 8 — Hierarquia Visual e Layout

Para cada tela:
1. Verificar que há um elemento de destaque visual claro (H1/CTA primário)
2. Verificar que a hierarquia tipográfica segue D03 (H1 > H2 > H3 > body)
3. Verificar que espaçamento entre elementos segue o sistema de tokens de D03
4. Verificar que elementos primários têm contraste visual claro vs secundários
5. Verificar que a área de conteúdo principal é identificável imediatamente
6. Verificar que ações destrutivas são visualmente distintas das construtivas (cor vermelha/warning)
7. Verificar que CTAs primários são únicos por tela (não 3 botões igualmente proeminentes)

**Critérios:**
1. Tela sem hierarquia visual clara (tudo igual em tamanho/peso) → **P1 Alto**
2. Ação destrutiva com mesmo estilo visual de ação construtiva → **P1 Alto**
3. Múltiplos CTAs primários com mesma proeminência em fluxo crítico → **P1 Alto**
4. Espaçamento inconsistente (valores fora do sistema de tokens) → **P2 Médio**
5. Densidade excessiva sem espaço de respiração → **P2 Médio**

**Tabela de finding:**
| Arquivo | Tela/Componente | Problema de hierarquia | Referência D03 | Severidade | Correção |
|---------|----------------|----------------------|---------------|-----------|---------|

---

### DIMENSÃO 9 — Acessibilidade

Para cada tela e componente interativo:
1. Verificar que imagens têm `alt` descritivo (ou `alt=""` para decorativas)
2. Verificar que todos os elementos interativos são alcançáveis por teclado (Tab order)
3. Verificar que foco é visível em todos os elementos interativos (`:focus-visible`)
4. Verificar que botões de ícone têm `aria-label`
5. Verificar que formulários têm `htmlFor`/`aria-labelledby` associando labels a campos
6. Verificar que modais/dialogs têm `role="dialog"`, `aria-modal="true"` e focus trap
7. Verificar que mensagens de erro são associadas ao campo via `aria-describedby`
8. Verificar que estados de loading são comunicados via `aria-live` ou `aria-busy`
9. Verificar contraste de cores (mínimo 4.5:1 para texto normal, 3:1 para texto grande) — análise estática

**Critérios:**
1. Imagem sem `alt` → **P1 Alto** (WCAG 2.1 AA obrigatório)
2. Elemento interativo não alcançável por teclado → **P1 Alto**
3. Modal sem focus trap (foco sai do modal pelo Tab) → **P1 Alto**
4. Botão de ícone sem `aria-label` → **P1 Alto**
5. Campo de formulário sem label associado via `htmlFor` → **P1 Alto** (duplica D5, reforça)
6. Estado de loading sem `aria-live` → **P2 Médio**
7. Foco visível ausente ou insuficiente → **P1 Alto**

**Tabela de finding:**
| Arquivo | Elemento | Problema de acessibilidade | WCAG | Severidade | Correção |
|---------|---------|--------------------------|------|-----------|---------|

---

### DIMENSÃO 10 — Responsividade e Adaptação

Para cada tela e componente:
1. Verificar que o layout adapta corretamente para os breakpoints definidos em D03
2. Verificar que botões e áreas tocáveis têm mínimo de 44×44px no mobile
3. Verificar que textos não transbordam nem ficam ilegíveis em telas pequenas
4. Verificar que tabelas têm estratégia de responsividade (scroll horizontal, cards, etc.)
5. Verificar que formulários são usáveis em mobile (campos não cobertos pelo teclado virtual)
6. Verificar que navegação mobile tem estratégia (hamburger menu, bottom nav, etc.)
7. Verificar que imagens têm `srcset` ou equivalente para diferentes densidades

**Critérios:**
1. Conteúdo transbordando horizontalmente em mobile → **P0 Crítico**
2. Botão/área tocável < 44×44px → **P1 Alto** (WCAG 2.5.5)
3. Tabela sem estratégia de responsividade → **P1 Alto**
4. Texto ilegível (< 12px) em qualquer breakpoint → **P1 Alto**
5. Navegação ausente ou quebrada em mobile → **P1 Alto**
6. Formulário coberto pelo teclado virtual sem scroll → **P2 Médio**

**Tabela de finding:**
| Arquivo | Componente | Breakpoint | Problema | Severidade | Correção |
|---------|-----------|-----------|---------|-----------|---------|

---

## 5. CLASSIFICAÇÃO DE SEVERIDADE

| Nível | Código | Critério | Ação |
|-------|--------|---------|------|
| Crítico | P0 | Tela crítica não implementada, estado de loading/erro ausente em fluxo crítico, conteúdo transbordando, erro técnico exposto ao usuário | Correção imediata. Deploy bloqueado. |
| Alto | P1 | Empty state ausente, campo sem label, foco invisível, botão de ícone sem aria-label, feedback de sucesso ausente, CTA ambíguo em fluxo crítico | Correção imediata. Deploy bloqueado. |
| Médio | P2 | Inconsistência de copy, micro-animação ausente, espaçamento fora do sistema, validação apenas no submit | Corrigido diretamente. |
| Baixo | P3 | Melhoria estética, otimização de densidade, srcset ausente em imagens não-críticas | Corrigido diretamente. |

---

## 6. PROTOCOLO DE CORREÇÃO

1. **Todos os findings (P0–P3) são corrigidos diretamente no código** — sem exceção
2. Cada correção marcada com comentário: `{/* [CORRIGIDO: UX-XXX] */}` (JSX) ou `// [CORRIGIDO: UX-XXX]` (JS/TS)
3. **NUNCA corrigir lógica de negócio** — apenas camada de apresentação e UX
4. **NUNCA criar novos componentes** quando o problema é no existente — corrigir no lugar
5. Se a correção exige asset externo (ícone, imagem) que não existe → marcar `[PENDENTE: asset UX-XXX necessário]`
6. Decisões não óbvias → registrar `[DECISÃO APLICADA: DEC-XXX — motivo]`
7. Ao atingir 15 decisões autônomas não-óbvias → emitir alerta e continuar (não parar)
8. Após cada dimensão com > 10 findings → use `/compact preservando: inventário de telas, findings registrados, correções aplicadas`

**Tipos de correção por dimensão:**

| Dimensão | Exemplos de correção |
|----------|---------------------|
| D1 — Cobertura | Criar rota e componente básico para tela ausente |
| D2 — Estados | Adicionar skeleton, empty state, tratamento de erro |
| D3 — Feedback | Adicionar toast, loading state, mensagem de sucesso |
| D4 — Fluxos | Adicionar botão de volta, link de continuação |
| D5 — Formulários | Adicionar `<label>`, mover mensagem de erro para perto do campo |
| D6 — Copy | Reescrever CTA, mensagem de erro, empty state copy |
| D7 — Micro-interações | Adicionar `transition`, `duration`, animação CSS |
| D8 — Hierarquia | Ajustar tamanho tipográfico, cor de botão destrutivo |
| D9 — Acessibilidade | Adicionar `alt`, `aria-label`, `htmlFor`, focus trap |
| D10 — Responsividade | Adicionar breakpoint CSS, ajustar área tocável |

---

## 7. RELATÓRIO FINAL

Salve em `docs/05 - Auditorias/AUDIT_UX_PRODUTO.md`:

```markdown
# Auditoria UX — Produto Implementado
**Data:** [YYYY-MM-DD HH:MM]
**Auditor:** Claude Code Desktop (UX Auditor)
**Framework:** ShiftLabs Framework A06 v1.0
**Projeto:** [nome do projeto]
**Telas auditadas:** [N] (todas identificadas em src/)

## Resumo Executivo
- Dimensões auditadas: 10
- Telas mapeadas: [N]
- Componentes interativos mapeados: [N]
- Findings P0: [N] | P1: [N] | P2: [N] | P3: [N]
- Correções aplicadas: [N]
- GATE: [APROVADO ✅ / BLOQUEADO ❌]

## Inventário do Produto Implementado

### Telas
[tabela da Fase 1 — Inventário de Telas]

### Componentes Interativos
[tabela da Fase 1 — Componentes Interativos]

## Findings por Dimensão

### Dimensão 1 — Cobertura de Telas vs D06
[tabela de findings]

### Dimensão 2 — Estados de UI
[tabela de findings]

### Dimensão 3 — Feedback ao Usuário
[tabela de findings]

### Dimensão 4 — Fluxos Completos
[tabela de findings]

### Dimensão 5 — Formulários
[tabela de findings]

### Dimensão 6 — Copy e Microcopy
[tabela de findings]

### Dimensão 7 — Micro-interações e Animações
[tabela de findings]

### Dimensão 8 — Hierarquia Visual e Layout
[tabela de findings]

### Dimensão 9 — Acessibilidade
[tabela de findings]

### Dimensão 10 — Responsividade e Adaptação
[tabela de findings]

## Correções Aplicadas
| # | UX-ID | Dimensão | Arquivo | Antes | Depois | Status |
|---|-------|---------|---------|-------|--------|--------|
| 1 | UX-001 | D2 | src/... | ... | ... | ✅ Corrigido |

## Decisões Autônomas Registradas
| # | DEC-ID | Dimensão | Decisão | Justificativa |
|---|--------|---------|---------|--------------|

## GATE DE APROVAÇÃO
| Critério | Status |
|----------|--------|
| Zero P0 após correções | PASS/FAIL |
| Zero P1 após correções | PASS/FAIL |
| Zero P2 após correções | PASS/FAIL |
| Zero P3 após correções | PASS/FAIL |
| Todas as 10 dimensões executadas | PASS/FAIL |
| Relatório completo gerado | PASS/FAIL |
| Re-validação pós-correção executada | PASS/FAIL |

RESULTADO: APROVADO ✅ / BLOQUEADO ❌
```

---

## 8. GATE — CRITÉRIO DE APROVAÇÃO

| Critério | Obrigatório |
|----------|-------------|
| Zero findings P0 após correções | ✅ SIM |
| Zero findings P1 após correções | ✅ SIM |
| Zero findings P2 após correções | ✅ SIM |
| Zero findings P3 após correções | ✅ SIM |
| Todas as 10 dimensões executadas | ✅ SIM |
| Inventário de telas completo | ✅ SIM |
| Relatório completo gerado | ✅ SIM |
| Re-validação pós-correção executada | ✅ SIM |

**GATE = APROVADO** somente se TODOS os critérios forem atendidos.
**GATE = BLOQUEADO** se qualquer finding residual de qualquer nível existir após correções.

---

## 9. TRACKING DE PROGRESSO

> **⚡ CRIE ESTE TODO IMEDIATAMENTE AO INICIAR — antes de qualquer leitura ou análise**
>
> Use `TodoWrite` para criar todas as fases e tarefas listadas abaixo antes de executar qualquer ação. À medida que concluir cada tarefa, marque-a como `done` imediatamente — antes de avançar para a próxima. Nunca execute uma tarefa sem ela estar na lista. Nunca avance sem marcar a tarefa anterior como concluída. O feedback em tempo real é parte obrigatória do protocolo.

Use TodoWrite:
1. Leitura de inputs (D01, D03, D04, D06, D07, D08, D09)
2. Fase 1 — Inventário de telas implementadas
3. Fase 1 — Inventário de componentes interativos
4. Dimensão 1 — Cobertura de Telas vs D06
5. Dimensão 2 — Estados de UI
6. Dimensão 3 — Feedback ao Usuário
7. Dimensão 4 — Fluxos Completos
8. Dimensão 5 — Formulários
9. Dimensão 6 — Copy e Microcopy
10. Dimensão 7 — Micro-interações e Animações
11. Dimensão 8 — Hierarquia Visual e Layout
12. Dimensão 9 — Acessibilidade
13. Dimensão 10 — Responsividade e Adaptação
14. Correções P0
15. Correções P1
16. Correções P2
17. Geração do relatório
18. Re-validação pós-correção
19. GATE final
```

---

## Changelog

| Versão | Data | Alterações |
|--------|------|-----------|
| 1.0 | 23/03/2026 | Criação. Auditoria UX profunda do produto implementado cobrindo 10 dimensões: cobertura de telas, estados de UI, feedback ao usuário, fluxos completos, formulários, copy/microcopy, micro-interações, hierarquia visual, acessibilidade e responsividade. Complementa A04 Eixo 3 e B07. Fontes de verdade: D01, D03, D04, D06, D07, D08, D09. |
