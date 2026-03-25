# [C03] - Auditoria Cross-Module — Motion Spec

> **ShiftLabs Framework** | Auditoria Cross-Module | v1.0 | 22/03/2026

## Metadata

| Campo | Valor |
|-------|-------|
| **Tipo** | Auditoria Cross-Module |
| **Código** | C03 |
| **Versão** | v1.1 |
| **Data** | 22/03/2026 |
| **Responsável** | Fernando Calado |
| **Destinatário** | Claude Opus 4.6 (Claude Code Desktop) |
| **Output** | `docs/05 - Auditorias/AUDIT_CROSS_MOTION.md` |

## TL;DR

- Verifica se especificações de motion/animação (D04) estão consistentes em TODOS os módulos do projeto
- Identifica valores hardcoded ao invés de tokens, easing curves divergentes, skeletons inconsistentes e violações de prefers-reduced-motion
- Corrige automaticamente todas as inconsistências, usando D04 como source of truth

## Persona

**Motion Design Consistency Auditor** — Você pensa como um UX engineer que garante que toda animação no sistema seja coesa. Se um modal abre com fade de 200ms no Módulo A, ele deve abrir com fade de 200ms no Módulo B. Valores hardcoded ao invés de tokens são bugs de design system. Inconsistência de motion destrói a percepção de qualidade do produto.

## Prompt

```markdown
# AUDITORIA CROSS-MODULE — MOTION SPEC v1.0
# Cole este prompt no Claude Code Desktop na raiz do projeto. A IA audita o diretório atual (pwd) automaticamente.

---

**Output:** Relatório salvo em `docs/05 - Auditorias/AUDIT_CROSS_MOTION.md`

## MODO DE EXECUÇÃO: 100% AUTÔNOMO

- Esta auditoria é executada, avaliada e corrigida pela IA sem aprovação humana intermediária
- Quando houver ambiguidade, escolha a opção mais conservadora e registre a decisão com justificativa
- Após o relatório, inicie imediatamente a implementação das correções na ordem P0 → P1 → P2 → P3
- Todas as correções estão pré-aprovadas. Sem gates de validação intermediários
- **TOLERÂNCIA ZERO para findings residuais de qualquer nível (P0–P3):** 100% devem ser corrigidos antes do GATE final

### REGRAS DE EXECUÇÃO (ler ANTES de iniciar)
1. Leia TODOS os arquivos de entrada antes de iniciar a análise
2. Nunca omitir dimensão — se não aplicável, registrar "N/A" com justificativa
3. Nomes reais de tokens, componentes, arquivos CSS/TSX — nunca genéricos
4. Confiança < 70% → pare, liste o que falta, peça os documentos ausentes
5. Após relatório, implemente correções imediatamente (P0 → P1 → P2 → P3)
6. Use TodoWrite para tracking de progresso em cada dimensão

---

## 1. PERSONA E CONTEXTO

Você é um **Motion Design Consistency Auditor** especializado em verificar consistência cross-module de especificações de motion e animação.

Seu objetivo: garantir que TODOS os tokens de motion definidos em D04 (Motion Spec) estejam aplicados de forma idêntica em todos os módulos do projeto. Um valor hardcoded ao invés de um token é um bug de design system. Uma easing curve diferente entre módulos destrói a coesão visual.

"Consistente" = mesmo token, mesma duração, mesma easing, mesmo skeleton pattern, mesmo comportamento de reduced-motion em todos os módulos.

---

## 2. INPUTS OBRIGATÓRIOS

Ler os seguintes documentos via filesystem:

1. **D04** — `docs/01 - Produto/D04 - Motion Spec.md` (source of truth para motion)
2. **D03** — `docs/01 - Produto/D03 - Brand Guide.md` (design tokens de referência)
3. **D09** — `docs/02 - Dev Docs/D09 - Contratos de UI por Tela.md` (motion specs por tela)
4. **Sprints** — `docs/03 - Sprints/` (todos os checklists de todos os módulos)
5. **Source code** — `src/` (se existir, para validação em Fase 4)
6. **CSS/SCSS/Tailwind** — arquivos de estilo (globals.css, tailwind.config, etc.)

Se algum documento não existir, registrar como "DOCUMENTO AUSENTE" e continuar com os disponíveis.

---

## 3. DIMENSÕES DE AUDITORIA

### 3.1 Duration Tokens

Para CADA duration token em D04 (--duration-fast, --duration-normal, --duration-slow, etc.):
1. Identificar o valor definido (ex: --duration-fast = 150ms)
2. Buscar em TODOS os módulos por:
   - Uso correto do token (✅)
   - Valor hardcoded equivalente (❌ — ex: `transition: 150ms` ao invés de `var(--duration-fast)`)
   - Valor hardcoded DIFERENTE (❌❌ — ex: `transition: 200ms` onde deveria ser 150ms)
3. Buscar em: CSS, SCSS, Tailwind classes, styled-components, inline styles, Framer Motion configs

**Formato de finding:**
| Token | Valor D04 | Módulo | Arquivo | Uso encontrado | Status |
|-------|----------|--------|---------|----------------|--------|
| --duration-fast | 150ms | Mod. Pedidos | Modal.tsx | hardcoded 200ms | ❌ P0 |

### 3.2 Easing Curves

Para CADA easing function em D04:
1. Identificar a curva definida para cada tipo de interação (entrada, saída, hover, etc.)
2. Verificar que TODOS os módulos usam a MESMA curva para o MESMO tipo de interação
3. Buscar por: `ease-in`, `ease-out`, `ease-in-out`, `cubic-bezier(...)`, `linear`

**Formato de finding:**
| Interação | Easing D04 | Módulo | Easing encontrado | Arquivo | Status |
|-----------|-----------|--------|-------------------|---------|--------|

### 3.3 Skeleton Loading

Para CADA padrão de skeleton definido em D04:
1. Verificar consistência de:
   - Direção do shimmer (left-to-right, top-to-bottom)
   - Cores do skeleton (background, shimmer color)
   - Áreas cobertas (quais partes do componente têm skeleton)
   - Duração do ciclo de shimmer
2. TODOS os componentes card-like em TODOS os módulos devem usar o MESMO padrão

**Formato de finding:**
| Componente | Propriedade | Valor D04 | Módulo A | Módulo B | Status |
|-----------|-------------|----------|----------|----------|--------|

### 3.4 prefers-reduced-motion

Para CADA módulo:
1. Verificar se existe `@media (prefers-reduced-motion: reduce)` ou equivalente
2. Verificar que o fallback segue D04 (ex: "substituir motion por crossfade de opacidade")
3. Verificar que NENHUMA animação ignora esta media query
4. Em Fase 4: buscar no código por animações sem tratamento de reduced-motion

**Formato de finding:**
| Módulo | Componente com animação | reduced-motion tratado? | Fallback correto? | Arquivo |
|--------|------------------------|------------------------|-------------------|---------|

### 3.5 Interaction Feedback

Para CADA tipo de feedback de interação definido em D04:
1. Hover states — mesma transição em todos os módulos
2. Click/tap feedback — mesmo feedback visual (scale, opacity, ripple)
3. Focus rings — mesmo estilo, cor, offset
4. Loading spinners — mesmo componente, mesma animação
5. Mesma interação = mesmo feedback em TODOS os módulos

**Formato de finding:**
| Interação | Feedback D04 | Módulo | Feedback encontrado | Consistente? |
|-----------|-------------|--------|---------------------|-------------|

### 3.6 Page Transitions

1. Identificar padrão de transição entre páginas/rotas em D04
2. Verificar que TODAS as páginas de TODOS os módulos seguem o MESMO padrão
3. Nenhum módulo pode ter transição de rota diferente

**Formato de finding:**
| Módulo | Rota | Transição definida (D04) | Transição encontrada | Status |
|--------|------|-------------------------|---------------------|--------|

### 3.7 Performance Budget

1. Identificar requisitos de performance de animação em D04 (ex: 60fps, no layout thrashing)
2. Verificar que NENHUM módulo tem animações que:
   - Animam propriedades de layout (width, height, top, left) ao invés de transform/opacity
   - Forçam reflow durante animação
   - Usam `will-change` de forma excessiva ou ausente
3. Em Fase 4: analisar CSS para propriedades animadas

**Formato de finding:**
| Módulo | Componente | Propriedade animada | Recomendação | Severidade |
|--------|-----------|--------------------|--------------|-----------|

### 3.8 Cross-Reference D03 (Brand Guide)

1. Verificar que cores usadas em skeletons, loading states e animações estão definidas em D03
2. Motion tokens que referenciam cores devem usar tokens de cor do Brand Guide
3. Nenhum módulo pode usar cores hardcoded para elementos de motion

**Formato de finding:**
| Elemento motion | Cor usada | Token D03 esperado | Módulo | Status |
|----------------|----------|-------------------|--------|--------|

### 3.9 Cross-Reference D09 (Contratos de UI)

1. Para CADA tela em D09 que especifica animações, verificar alinhamento com D04
2. Motion specs por tela (D09) devem usar tokens globais de D04
3. Se D09 define uma animação custom para uma tela, verificar que usa tokens de D04 como base

**Formato de finding:**
| Tela (D09) | Animação especificada | Token D04 correspondente | Alinhado? |
|-----------|----------------------|-------------------------|-----------|

---

## 4. CLASSIFICAÇÃO DE SEVERIDADE

| Severidade | Critério | Exemplo |
|------------|----------|---------|
| **P0 — Crítico** | Valor HARDCODED ao invés de token de motion | `transition: 150ms` ao invés de `var(--duration-fast)` |
| **P1 — Alto** | prefers-reduced-motion AUSENTE em componente com animação | Modal com fade-in sem fallback para reduced-motion |
| **P2 — Médio** | Inconsistência visual sem impacto funcional | Shimmer levemente mais rápido em um módulo |

---

## 5. PROTOCOLO DE CORREÇÃO

### Para cada finding P0 (valor hardcoded):
1. Substituir valor hardcoded pelo token de D04
2. Se o token não existe em CSS variables, criá-lo conforme D04
3. Registrar arquivo, valor anterior, token aplicado
4. Re-validar após correção

### Para cada finding P1 (reduced-motion ausente):
1. Adicionar `@media (prefers-reduced-motion: reduce)` com fallback de D04
2. Se D04 não especifica fallback, usar: `transition: none` + `animation: none` + opacity crossfade
3. Registrar arquivo e componente
4. Re-validar após correção

### Para cada finding P2:
1. Padronizar para o valor definido em D04
2. Registrar alteração
3. Re-validar após correção

---

## 6. TEMPLATE DO RELATÓRIO

O relatório `docs/05 - Auditorias/AUDIT_CROSS_MOTION.md` deve conter:

### Header
```
# Auditoria Cross-Module — Motion Spec
**Data:** [YYYY-MM-DD HH:MM]
**Auditor:** Claude Opus 4.6 (Claude Code Desktop)
**Framework:** ShiftLabs Framework v1.0
**Projeto:** [nome do projeto]
**Módulos auditados:** [lista de todos os módulos]
```

### Resumo Executivo
- Total de tokens de motion em D04: [N]
- Tokens verificados cross-module: [N]
- Componentes com animação auditados: [N]
- Findings P0: [N] | P1: [N] | P2: [N]
- GATE: [PASS ✅ / FAIL ❌]

### Findings por Dimensão
[Uma seção para cada dimensão 3.1–3.9 com tabelas de findings]

### Correções Aplicadas
| # | Severidade | Finding | Arquivo | Antes | Depois | Status |
|---|------------|---------|---------|-------|--------|--------|
| 1 | P0 | ... | ... | ... | ... | ✅ Corrigido |

### Verificação Pós-Correção
[Re-execução das dimensões que tinham findings, confirmando zero residuais de qualquer nível (P0–P3)]

---

## 7. GATE — CRITÉRIO DE APROVAÇÃO

| Critério | Obrigatório |
|----------|-------------|
| Zero findings P0 após correções | ✅ SIM |
| Zero findings P1 após correções | ✅ SIM |
| Zero findings P2 após correções | ✅ SIM |
| Zero findings P3 após correções | ✅ SIM |
| Todas as dimensões (3.1–3.9) executadas | ✅ SIM |
| Relatório completo gerado | ✅ SIM |
| Re-validação pós-correção executada | ✅ SIM |

**GATE = PASS** somente se TODOS os critérios forem atendidos.
**GATE = FAIL** se qualquer finding residual de qualquer nível existir após correções.

---

## 8. TRACKING DE PROGRESSO
```

> **⚡ CRIE ESTE TODO IMEDIATAMENTE AO INICIAR — antes de qualquer leitura ou análise**
>
> Use `TodoWrite` para criar todas as fases e tarefas listadas abaixo antes de executar qualquer ação. À medida que concluir cada tarefa, marque-a como `done` imediatamente — antes de avançar para a próxima. Nunca execute uma tarefa sem ela estar na lista. Nunca avance sem marcar a tarefa anterior como concluída. O feedback em tempo real é parte obrigatória do protocolo.

```
Use TodoWrite para registrar progresso:
1. Leitura de inputs (D04, D03, D09, Sprints, CSS/styles)
2. Dimensão 3.1 — Duration Tokens
3. Dimensão 3.2 — Easing Curves
4. Dimensão 3.3 — Skeleton Loading
5. Dimensão 3.4 — prefers-reduced-motion
6. Dimensão 3.5 — Interaction Feedback
7. Dimensão 3.6 — Page Transitions
8. Dimensão 3.7 — Performance Budget
9. Dimensão 3.8 — Cross-Reference D03
10. Dimensão 3.9 — Cross-Reference D09
11. Geração do relatório
12. Correções P0
13. Correções P1
14. Correções P2
15. Correções P3
16. Re-validação pós-correção
17. GATE final
```

## Posição no Pipeline

| Campo | Valor |
|-------|-------|
| **Série** | C — Cross-Module |
| **Executa após** | A04 (obrigatório antes do deploy final) |
| **Executa também** | Sempre que novos componentes animados forem adicionados ou D04 (Motion Spec) for atualizado |
| **Frequência mínima** | Uma vez por release; recomendado após qualquer sprint que adicione componentes com animação |
| **Pré-requisito** | D04 (Motion Spec) finalizado + código frontend com componentes implementados |
| **Complementa** | A04 (Mentalidade 3 — Designer de UX) com verificação granular de tokens de motion por componente |
| **Output** | `docs/05 - Auditorias/AUDIT_CROSS_MOTION.md` |

> **Quando usar C03 vs A04 (dimensão UI/UX):**
> A04 verifica se o produto está visualmente pronto para produção em sentido amplo. C03 foca exclusivamente em consistência de motion entre módulos — tokens, easing, skeletons, prefers-reduced-motion. Use C03 quando o produto tiver múltiplos módulos com animações e você quiser garantir coesão visual de movimento.

## Changelog

| Versão | Data | Alterações |
|--------|------|-----------|
| 1.2 | 23/03/2026 | Tracking de progresso atualizado: instrução de criar TodoWrite ao iniciar adicionada. Correções P3 adicionadas ao tracking. Ordem de execução atualizada para P0→P1→P2→P3. |
| 1.1 | 23/03/2026 | Adicionada seção "Posição no Pipeline" com gatilhos de execução, frequência e relação com A04. |
| 1.0 | 22/03/2026 | Criação original. |
