# Motion Spec — AI-Dani-Admin

## Padrões de Animação do Módulo de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Frontend Engineers / Motion Designer |
| Escopo | Tokens de motion, padrões de animação e skeleton specs para o painel Admin — módulo AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Dependências | D02 - Stacks (Framer Motion 12+), D03 - Brand Guide |

---

> **📌 TL;DR**
>
> - **Biblioteca:** Framer Motion 12+ na web. CSS transitions para micro-interações leves.
> - **Princípio dominante:** Sobriedade operacional. O painel Admin é uma ferramenta de trabalho — animações comunicam estado, não estilo. Nenhuma animação por decoração.
> - **Tokens de duração:** 5 valores (instant 100ms, fast 150ms, normal 250ms, slow 400ms, page 500ms).
> - **Easing curves:** 3 curvas padrão (ease-out para entradas, ease-in-out para transições, ease-spring para feedback de ação).
> - **Skeleton obrigatório:** Todo estado de loading usa skeleton com shimmer. Spinner proibido.
> - **prefers-reduced-motion:** Todo motion não essencial desabilitado. Estados de hover/focus mantidos via CSS sem transform ou opacity animation.
> - **Contexto crítico:** Alertas de desligamento automático e sinalizações de revisão nunca usam animações de entrada lenta — usam `instant` (100ms) ou sem animação, para que a urgência seja percebida imediatamente.

---

## 1. Princípios de Motion

**PM-01. Sobriedade operacional.** O painel Admin é uma ferramenta de supervisão. Cada animação deve comunicar um estado funcional (carregando, sucesso, erro, novo dado) — nunca existe para embelezamento.

**PM-02. Urgência é imediata.** Alertas críticos (desligamento automático, taxa de erro > 30%) nunca usam fade-in ou slide. Aparecem instantaneamente ou com `instant` (100ms) para que o Admin perceba a urgência sem delay perceptível.

**PM-03. Feedback confirma ação.** Todo clique em ação crítica (assumir conversa, salvar threshold, reativar agente) deve ter feedback visual imediato (≤150ms) com micro-interação de confirmação.

**PM-04. Continuidade sem ruptura.** Filtros na lista de interações, atualizações de métricas em tempo real e carregamento de dados usam skeleton inline — nunca substituem a tela inteira por loading spinner.

**PM-05. Acessibilidade não é opcional.** `prefers-reduced-motion: reduce` desativa todas as animações de movimento (transforms, slides, fades). Estados de hover e focus mantidos via mudança de cor/borda sem animation property.

---

## 2. Tokens Globais de Motion

### 2.1 Durações

| Token | Valor | Uso |
|---|---|---|
| `--duration-instant` | `100ms` | Alertas críticos de desligamento automático, feedback de botão de ação crítica, aparição de badges de status urgente |
| `--duration-fast` | `150ms` | Hover em botões, hover em linhas da lista de interações, micro-feedback de clique, abertura de tooltip |
| `--duration-normal` | `250ms` | Modais de confirmação de takeover, drawers de configuração, toasts de confirmação, transições de filtro |
| `--duration-slow` | `400ms` | Abertura de drawers grandes (configurações de agente), expansão de acordeão de detalhes de interação |
| `--duration-page` | `500ms` | Transição entre seções principais do painel (lista → detalhes de interação) |

### 2.2 Easing Curves

| Token | Valor | Uso |
|---|---|---|
| `--ease-out` | `cubic-bezier(0.16, 1, 0.3, 1)` | **Entradas** — elementos que aparecem na tela (modais, drawers, toasts, badges novos) |
| `--ease-in-out` | `cubic-bezier(0.65, 0, 0.35, 1)` | **Transições** — elementos que mudam de estado ou posição (filtros, accordeão, tabs) |
| `--ease-spring` | `cubic-bezier(0.34, 1.56, 0.64, 1)` | **Feedback de ação** — confirmações positivas (salvar threshold com sucesso, takeover confirmado) |

### 2.3 Escala e Transform

| Token | Valor | Uso |
|---|---|---|
| `--scale-press` | `0.97` | Feedback de press em botões de ação (assumir conversa, reativar agente) |
| `--scale-hover` | `1.02` | Hover em cards de métricas quando clicáveis |
| `--opacity-enter` | `0` | Estado inicial de elementos com fade-in |
| `--opacity-visible` | `1` | Estado final de elementos com fade-in |

---

## 3. Padrões de Animação por Tipo de Interação

### 3.1 Transições de Seção (Page Transitions)

**Contexto:** navegação entre seções do painel Admin (lista de interações → detalhes de interação → configurações).

| Propriedade | Valor |
|---|---|
| Saída (elemento saindo) | `opacity: 1 → 0`, `x: 0 → -20px`, duração: `250ms`, easing: `--ease-in-out` |
| Entrada (elemento entrando) | `opacity: 0 → 1`, `x: 20px → 0`, duração: `--duration-page` (500ms), easing: `--ease-out` |
| Biblioteca | Framer Motion `AnimatePresence` + `motion.div` |

```tsx
// Padrão para transições de seção no painel Admin
const pageVariants = {
  initial: { opacity: 0, x: 20 },
  animate: { opacity: 1, x: 0, transition: { duration: 0.5, ease: [0.16, 1, 0.3, 1] } },
  exit: { opacity: 0, x: -20, transition: { duration: 0.25, ease: [0.65, 0, 0.35, 1] } }
}
```

**prefers-reduced-motion:** Remove transform. Mantém apenas `opacity: 0 → 1` com `duration: 150ms`.

### 3.2 Modais (Confirmação de Takeover, Configuração de Threshold)

| Propriedade | Valor |
|---|---|
| Overlay | `opacity: 0 → 1`, duração: `--duration-normal` (250ms) |
| Modal container | `opacity: 0 → 1`, `scale: 0.96 → 1`, duração: `--duration-normal` (250ms), easing: `--ease-out` |
| Fechamento | `opacity: 1 → 0`, `scale: 1 → 0.96`, duração: `--duration-fast` (150ms) |

**Regra específica para modal de takeover:** Abertura em `--duration-fast` (150ms) — o Admin está em situação de urgência ao iniciar o takeover, a confirmação deve aparecer rapidamente.

**prefers-reduced-motion:** Apenas `opacity` animation. Remove `scale`.

### 3.3 Drawers (Configurações de Agente, Detalhes de Interação)

| Propriedade | Valor |
|---|---|
| Entrada (slide from right) | `x: 100% → 0`, `opacity: 0 → 1`, duração: `--duration-slow` (400ms), easing: `--ease-out` |
| Saída | `x: 0 → 100%`, `opacity: 1 → 0`, duração: `--duration-normal` (250ms) |
| Overlay | `opacity: 0 → 1`, duração: `--duration-normal` (250ms) |

**prefers-reduced-motion:** Remove `x` transform. Mantém `opacity` apenas.

### 3.4 Toasts de Confirmação

| Tipo | Animação | Duração de Exibição |
|---|---|---|
| Sucesso (threshold salvo, agente reativado) | Slide from top-right + `opacity: 0 → 1`, `--duration-normal` (250ms), `--ease-out` | 4 segundos auto-dismiss |
| Erro (threshold inválido, falha de API) | Mesmo slide, cor destrutiva | 6 segundos auto-dismiss (mais tempo para ler o erro) |
| Informativo (filtro aplicado) | `opacity: 0 → 1` apenas, `--duration-fast` (150ms) | 3 segundos auto-dismiss |

**Alerta crítico (desligamento automático):** NÃO usa toast. Usa banner fixo no topo do painel com `--duration-instant` (100ms) de aparição — persiste até ação do Admin.

### 3.5 Alertas Críticos (Banner de Desligamento Automático)

**Regra de urgência:** Alertas de desligamento automático, taxa de erro elevada e latência crítica aparecem como banner no topo do painel, não como toast dismissível.

| Propriedade | Valor |
|---|---|
| Aparição | `opacity: 0 → 1`, duração: `--duration-instant` (100ms) — urgência imediata |
| Animação de atenção | Pulso suave de borda (`box-shadow`) com `animation: pulse 2s infinite` — sinaliza que requer ação |
| Persistência | Permanece visível até o Admin tomar ação (reativar agente, investigar) |

**prefers-reduced-motion:** Remove `pulse` animation. Mantém banner estático com cor destrutiva.

### 3.6 Linha de Interação Sinalizada

**Contexto:** quando uma interação com confiança abaixo do threshold aparece na lista, ela é visualmente destacada.

| Propriedade | Valor |
|---|---|
| Aparição do badge "Sinalizada" | `opacity: 0 → 1`, duração: `--duration-fast` (150ms) |
| Background da linha | Transição de `transparent → --color-warning-subtle`, duração: `--duration-fast` (150ms) |
| Borda esquerda | Aparece sem animação (`--duration-instant`) — elemento de status, não de fluidez |

### 3.7 Separador Visual de Takeover no Chat

**Contexto:** linha com texto "Atendimento humano" que aparece no chat quando o Admin inicia o takeover (RN-DA-033).

| Propriedade | Valor |
|---|---|
| Separador | Fade-in `opacity: 0 → 1`, duração: `--duration-normal` (250ms) |
| Mensagem do Admin | Slide from bottom `y: 10px → 0` + `opacity: 0 → 1`, duração: `--duration-normal` (250ms) |
| Avatar diferenciado | Sem animação — aparece junto com a mensagem |

### 3.8 Hover em Botões de Ação

| Estado | Animação |
|---|---|
| Hover em botão primary ("Assumir conversa") | `background-color` transition, `--duration-fast` (150ms). Sem scale. |
| Press em botão primary | `scale: 1 → 0.97`, duração: `--duration-instant` (100ms) |
| Release | `scale: 0.97 → 1`, duração: `--duration-fast` (150ms), easing: `--ease-spring` |
| Hover em botão secondary | `background-color` transition, `--duration-fast` (150ms) |
| Botão desabilitado | Sem hover animation. `cursor: not-allowed`. `opacity: 0.5`. |

### 3.9 Hover em Linhas da Lista de Interações

| Estado | Animação |
|---|---|
| Hover | `background-color: transparent → --accent`, `--duration-fast` (150ms) |
| Click/Press | Sem scale. `background-color: --accent` mais intenso por `--duration-instant` (100ms) |

### 3.10 Filtros e Chips

| Interação | Animação |
|---|---|
| Adicionar chip de filtro | `opacity: 0 → 1`, `scale: 0.8 → 1`, `--duration-fast` (150ms), `--ease-spring` |
| Remover chip de filtro | `opacity: 1 → 0`, `scale: 1 → 0.8`, `--duration-fast` (150ms) |
| Lista atualizada após filtro | Skeleton loading inline substituindo os items. Não substitui a lista inteira por spinner. |

### 3.11 Dashboard de Métricas — Atualização em Tempo Real

**Contexto:** cards de métricas atualizados via Supabase Realtime ou polling.

| Interação | Animação |
|---|---|
| Valor de métrica atualizado | Flash suave: valor anterior `opacity: 1 → 0` (80ms) + novo valor `opacity: 0 → 1` (150ms) |
| Card sem dados → com dados | Skeleton dissolve-out + conteúdo fade-in, `--duration-normal` (250ms) |
| Card com dados → "Dados insuficientes" | Sem animação. Troca direta de conteúdo. |

---

## 4. Skeleton Specs

> **Regra absoluta:** Spinner é proibido. Todo estado de loading usa skeleton com shimmer animado.

### 4.1 Skeleton para Lista de Interações

| Elemento | Skeleton |
|---|---|
| Linha de interação (completa) | Retângulo de altura 56px, largura 100%, border-radius `--radius-sm` |
| Badge de status | Retângulo 60px × 20px, border-radius `--radius-sm` |
| Timestamp | Retângulo 80px × 14px |
| Nível de confiança | Retângulo 40px × 14px |
| Quantidade de linhas na carga inicial | 8 linhas skeleton |
| Quantidade de linhas no skeleton de filtro | 5 linhas (inline, sem substituir a lista) |

### 4.2 Skeleton para Cards de Métricas

| Elemento | Skeleton |
|---|---|
| Título do card | Retângulo 100px × 14px |
| Valor principal | Retângulo 80px × 28px |
| Sub-valor ou trend | Retângulo 120px × 12px |
| Ícone | Círculo 20px × 20px |

### 4.3 Skeleton para Gráfico do Dashboard

| Elemento | Skeleton |
|---|---|
| Área do gráfico | Retângulo 100% × 200px, border-radius `--radius-lg` |
| Legenda | 3 itens de 80px × 12px lado a lado |

### 4.4 Animação do Shimmer

```css
@keyframes shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}

.skeleton {
  background: linear-gradient(
    90deg,
    var(--muted) 25%,
    var(--muted-foreground, #a1a1a1) 37%, /* ajuste para luminosidade do shimmer */
    var(--muted) 63%
  );
  background-size: 400% 100%;
  animation: shimmer 1.4s ease infinite;
  border-radius: var(--radius-sm);
}

@media (prefers-reduced-motion: reduce) {
  .skeleton {
    animation: none;
    background: var(--muted);
  }
}
```

---

## 5. Acessibilidade de Motion

### 5.1 Implementação de prefers-reduced-motion

**Regra:** Toda animação implementada com Framer Motion deve verificar `useReducedMotion()`. Toda animação CSS deve usar `@media (prefers-reduced-motion: reduce)`.

```tsx
// Padrão obrigatório para componentes com Framer Motion
import { useReducedMotion } from 'framer-motion'

function SupectionModal({ isOpen }) {
  const shouldReduceMotion = useReducedMotion()

  const variants = {
    hidden: shouldReduceMotion
      ? { opacity: 0 }
      : { opacity: 0, scale: 0.96 },
    visible: shouldReduceMotion
      ? { opacity: 1 }
      : { opacity: 1, scale: 1 }
  }

  return (
    <AnimatePresence>
      {isOpen && (
        <motion.div
          variants={variants}
          initial="hidden"
          animate="visible"
          exit="hidden"
          transition={{ duration: shouldReduceMotion ? 0.1 : 0.25 }}
        >
          {/* conteúdo do modal */}
        </motion.div>
      )}
    </AnimatePresence>
  )
}
```

### 5.2 O que é preservado com prefers-reduced-motion ativo

| Preservado | Motivo |
|---|---|
| Mudanças de `color` e `background-color` | Comunicam estado sem movimento — essenciais para acessibilidade operacional |
| Mudanças de `border-color` e `box-shadow` | Indicam hover/focus sem translate ou scale |
| Fade de `opacity` com duração ≤ 100ms | Aparição/desaparição que não causa vertigem |
| Badge de status de alerta | Aparece instantaneamente — não usa animação |

| Removido | Motivo |
|---|---|
| `transform: translate`, `scale`, `rotate` | Movimento físico que pode causar desconforto |
| `AnimatePresence` com slide | Movimento horizontal/vertical perceptível |
| Shimmer do skeleton | Movimento contínuo que pode causar distração/desconforto |
| Pulse de alertas | Loop de animação |

---

## 6. Micro-interações Específicas do Módulo

### 6.1 Botão "Assumir conversa" — Sequência de Confirmação

1. Admin clica → `scale: 0.97` instantâneo (`--duration-instant` 100ms).
2. Modal de confirmação abre (`--duration-fast` 150ms com scale + opacity).
3. Admin confirma → botão exibe spinner interno por ≤ 500ms (duração da operação de takeover).
4. Sucesso → toast "Você assumiu essa conversa" + separador "Atendimento humano" no chat.

> Nota: O spinner é permitido DENTRO do botão de loading state — proibição é de spinner como substituto de skeleton para loading de conteúdo.

### 6.2 Slider de Threshold de Confiança

| Estado | Animação |
|---|---|
| Drag do thumb | `transform: scale(1.2)` no thumb durante drag, easing suave |
| Soltar thumb | `scale: 1.2 → 1`, `--duration-fast` (150ms), `--ease-spring` |
| Preview do valor | Tooltip acima do thumb com valor em tempo real, sem animação |
| Salvar com sucesso | Toast verde + thumb retorna à cor normal |
| Erro de validação (fora de 50-95%) | Shake horizontal: `x: 0 → -4px → 4px → -4px → 0`, duração total 300ms. Input borda fica destrutiva. |

### 6.3 Indicador de Agente em Tempo Real

O ponto de status do agente (ativo/desligado) exibe animação de pulsação quando ativo:

```css
@keyframes agent-pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.7; transform: scale(1.2); }
}

.agent-status-active {
  animation: agent-pulse 2s ease-in-out infinite;
}

@media (prefers-reduced-motion: reduce) {
  .agent-status-active {
    animation: none;
  }
}
```

---

## 7. Framer Motion — Configurações Globais

### 7.1 Variantes Reutilizáveis

```tsx
// src/components/ai-dani-admin/motion/variants.ts

export const fadeIn = {
  initial: { opacity: 0 },
  animate: { opacity: 1, transition: { duration: 0.25, ease: [0.16, 1, 0.3, 1] } },
  exit: { opacity: 0, transition: { duration: 0.15 } }
}

export const slideInRight = {
  initial: { opacity: 0, x: '100%' },
  animate: { opacity: 1, x: 0, transition: { duration: 0.4, ease: [0.16, 1, 0.3, 1] } },
  exit: { opacity: 0, x: '100%', transition: { duration: 0.25 } }
}

export const scaleIn = {
  initial: { opacity: 0, scale: 0.96 },
  animate: { opacity: 1, scale: 1, transition: { duration: 0.25, ease: [0.16, 1, 0.3, 1] } },
  exit: { opacity: 0, scale: 0.96, transition: { duration: 0.15 } }
}

export const instantAlert = {
  initial: { opacity: 0 },
  animate: { opacity: 1, transition: { duration: 0.1 } },
  exit: { opacity: 0, transition: { duration: 0.1 } }
}

export const chipIn = {
  initial: { opacity: 0, scale: 0.8 },
  animate: { opacity: 1, scale: 1, transition: { duration: 0.15, ease: [0.34, 1.56, 0.64, 1] } },
  exit: { opacity: 0, scale: 0.8, transition: { duration: 0.15 } }
}
```

---

## 8. Checklist de Validação de Motion

Antes de qualquer merge de componente com animações:

| Item | Critério |
|---|---|
| Token de duração | Todo `duration` usa valor de `--duration-*` definido neste doc. Nenhum valor hardcoded. |
| Token de easing | Todo `ease` usa valor de `--ease-*` definido neste doc. |
| prefers-reduced-motion | Todo componente com Framer Motion usa `useReducedMotion()`. Todo CSS animation usa `@media (prefers-reduced-motion: reduce)`. |
| Spinner | Nenhum spinner de loading de conteúdo. Apenas skeleton. Spinner apenas em loading state de botão. |
| Alerta crítico | Alertas de desligamento automático e taxa de erro elevada usam `--duration-instant` ou aparecem sem animação. |
| Separador de takeover | Aparece com `--duration-normal` (250ms). Não usa duração > 250ms. |

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Motion spec para o módulo AI-Dani-Admin — painel de supervisão operacional. |
