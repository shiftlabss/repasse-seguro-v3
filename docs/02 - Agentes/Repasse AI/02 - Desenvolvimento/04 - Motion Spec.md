# Motion Spec Unificado — Repasse Seguro

## Todos os Modulos

| Campo | Valor |
|---|---|
| **Destinatario** | Designers e frontend engineers |
| **Escopo** | Motion Spec unificado — Repasse Seguro (todos os modulos) |
| **Versao** | v2.0 |
| **Responsavel** | Claude Code Desktop |
| **Data da versao** | 2026-03-22 (America/Fortaleza) |
| **Fonte primaria** | ShiftLabs Stacks v7.0 — secoes 2.4 e 3.6 (Fernando Calado) |
| **Fonte secundaria** | Brand Guide v1.0 · Regras de Negocio v1.2 (01.1 a 01.5) |
| **Carater** | **Normativo.** Desvios exigem ADR aprovado. |

---

> **TL;DR**
>
> Este e o Motion Spec unificado aplicavel a **Admin, Cedente, Cessionario e Repasse AI**. Todos os modulos compartilham os mesmos tokens, padroes e regras de acessibilidade.
>
> - **Web:** Framer Motion 12+ como biblioteca obrigatoria. CSS transitions para micro-interacoes leves (hover, focus).
> - **Mobile:** React Native Reanimated 3+ na UI thread. Gesture Handler 2+ para feedback tatil. `[Cedente]` `[Cessionario]`
> - **Repasse AI:** Servico backend puro — este documento define os **contratos de motion** que Admin, Cedente e Cessionario implementam ao renderizar respostas da IA.
> - **Spinners:** PROIBIDOS. Skeleton screens com shimmer em todo loading state, sem excecao.
> - **Acessibilidade:** `prefers-reduced-motion` obrigatorio em toda animacao web. `accessibilityReduceMotionEnabled` no mobile. `aria-live` para streaming de IA.
> - **Tokens:** 7 duracoes, 5 easings CSS (+ spring physics), 6 escalas, 2 opacidades — herdados do ShiftLabs Stacks v7.0 secao 2.4.3 com extensoes documentadas.
> - **Principio:** Sobriedade — animacoes sao percebidas como fluidez, nunca como distracao.

---

## 1. Tokens de Duracao

Herdados do ShiftLabs Stacks v7.0, secao 2.4.3, com extensao de 2 niveis adicionais (`slower`, `crawl`) para cobrir scroll animations, onboarding e parallax.

### 1.1 Variaveis CSS (Web)

| Token | Valor | Variavel CSS | Uso |
|---|---|---|---|
| `instant` | `100ms` | `--duration-instant` | Feedback imediato: hover, focus ring, toggle, estado ativo de botao |
| `fast` | `150ms` | `--duration-fast` | Micro-interacoes: dropdown abrir/fechar, tooltip, popover, badge update |
| `normal` | `250ms` | `--duration-normal` | Transicoes padrao: modal, drawer, accordion, collapse, toast entrada |
| `slow` | `400ms` | `--duration-slow` | Transicoes complexas: stagger de listas, drag-and-drop settle, skeleton shimmer cycle, chart animate-in |
| `page` | `500ms` | `--duration-page` | Transicoes de rota, wizard step transitions, landing scroll reveals `[Cedente]` |
| `slower` | `700ms` | `--duration-slower` | Stagger lists completas, onboarding illustration entrance `[Cedente]` |
| `crawl` | `1000ms` | `--duration-crawl` | Landing page parallax segments, progress bar fill `[Cedente]` |

```css
@theme {
  --duration-instant: 100ms;
  --duration-fast: 150ms;
  --duration-normal: 250ms;
  --duration-slow: 400ms;
  --duration-page: 500ms;
  --duration-slower: 700ms;
  --duration-crawl: 1000ms;
}
```

### 1.2 Constantes TypeScript — Framer Motion (Web)

```typescript
// src/lib/motion-tokens.ts
export const duration = {
  instant: 0.1,
  fast: 0.15,
  normal: 0.25,
  slow: 0.4,
  page: 0.5,
  slower: 0.7,
  crawl: 1.0,
} as const;
```

### 1.3 Constantes TypeScript — Reanimated (Mobile)

```typescript
// packages/shared-utils/tokens/motion.ts
export const DURATION = {
  instant: 100,
  fast: 150,
  normal: 250,
  slow: 400,
  page: 500,
  slower: 700,
  crawl: 1000,
} as const;
```

---

## 2. Easing Curves

### 2.1 Cubic-Bezier (CSS)

| Token | Valor | Variavel CSS | Uso |
|---|---|---|---|
| `ease-out` | `cubic-bezier(0.16, 1, 0.3, 1)` | `--ease-out` | Elementos que entram em tela: modais, drawers, toasts, dropdowns, popovers. Desaceleracao rapida. |
| `ease-in-out` | `cubic-bezier(0.65, 0, 0.35, 1)` | `--ease-in-out` | Elementos que se movem dentro da tela: page transitions, accordions, collapse, resize. Simetria no inicio e fim. |
| `ease-spring` | `cubic-bezier(0.34, 1.56, 0.64, 1)` | `--ease-spring` | Elementos com personalidade fisica: drag-and-drop settle, bounce de confirmacao, badges. Overshooting sutil. |
| `ease-enter` | `cubic-bezier(0.0, 0.0, 0.2, 1)` | `--ease-enter` | Entrada de overlays (modal, drawer, sheet). Desaceleracao suave. `[Cedente]` |
| `ease-exit` | `cubic-bezier(0.4, 0.0, 1, 1)` | `--ease-exit` | Saida de overlays. Aceleracao para sair rapido. `[Cedente]` |

```css
@theme {
  --ease-out: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-in-out: cubic-bezier(0.65, 0, 0.35, 1);
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
  --ease-enter: cubic-bezier(0.0, 0.0, 0.2, 1);
  --ease-exit: cubic-bezier(0.4, 0.0, 1, 1);
}
```

### 2.2 Constantes TypeScript — Framer Motion (Web)

```typescript
// src/lib/motion-tokens.ts
export const easing = {
  easeOut: [0.16, 1, 0.3, 1] as const,
  easeInOut: [0.65, 0, 0.35, 1] as const,
  easeSpring: [0.34, 1.56, 0.64, 1] as const,
  easeEnter: [0.0, 0.0, 0.2, 1] as const,
  easeExit: [0.4, 0.0, 1, 1] as const,
};
```

### 2.3 Spring Physics — Framer Motion (Web)

| Preset | stiffness | damping | mass | Uso |
|---|---|---|---|---|
| `spring-default` | 300 | 25 | 1 | Drag-and-drop settle `[Admin]`, layout animations, card expand, list reorder |
| `spring-snappy` | 500 | 30 | 0.8 | Resposta rapida: reordenacao de listas, snap de elementos, press states, drag-to-dismiss `[Cessionario]` |
| `spring-gentle` | 200 | 20 | 1.2 | Entrada suave: drawer lateral, modal de detalhes, mensagens de chat IA `[Cedente]` `[Cessionario]` |
| `spring-bouncy` | 400 | 15 | 0.8 | Feedback positivo: botao desbloqueado `[Cedente]`, success checkmark, badge bounce `[Cessionario]` |

```typescript
// src/lib/motion-tokens.ts
export const spring = {
  default: { type: 'spring' as const, stiffness: 300, damping: 25, mass: 1 },
  snappy: { type: 'spring' as const, stiffness: 500, damping: 30, mass: 0.8 },
  gentle: { type: 'spring' as const, stiffness: 200, damping: 20, mass: 1.2 },
  bouncy: { type: 'spring' as const, stiffness: 400, damping: 15, mass: 0.8 },
} as const;
```

### 2.4 Spring Physics — Reanimated (Mobile)

```typescript
// packages/shared-utils/tokens/motion.ts
export const SPRING = {
  default: { damping: 20, stiffness: 300, mass: 1 },
  bouncy: { damping: 10, stiffness: 400, mass: 0.8 },
  gentle: { damping: 18, stiffness: 200, mass: 1 },
  snappy: { damping: 25, stiffness: 500, mass: 0.6 },
} as const;
```

---

## 3. Scale Tokens

| Token | Valor | Variavel CSS | Plataforma | Uso |
|---|---|---|---|---|
| `scale-press` | `0.97` | `--scale-press` | Web + Mobile | Botoes e cards ao serem pressionados |
| `scale-hover` | `1.02` | `--scale-hover` | Web only | Cards e links ao hover |
| `scale-active` | `0.95` | `--scale-active` | Mobile only | Press state em mobile (mais pronunciado que web) `[Cedente]` `[Cessionario]` |
| `scale-badge-pulse` | `1.15` | `--scale-badge-pulse` | Web + Mobile | Badge de notificacao e timer pulsante `[Cedente]` |
| `scale-icon-tap` | `0.90` | `--scale-icon-tap` | Mobile only | Icones de acao ao toque `[Cedente]` |
| `scale-checkbox` | `1.10` | `--scale-checkbox` | Web + Mobile | Checkbox de confirmacao ao ser marcado `[Cedente]` |

```css
@theme {
  --scale-press: 0.97;
  --scale-hover: 1.02;
  --scale-active: 0.95;
  --scale-badge-pulse: 1.15;
  --scale-icon-tap: 0.90;
  --scale-checkbox: 1.10;
}
```

```typescript
// packages/shared-utils/tokens/motion.ts (Mobile)
export const SCALE = {
  press: 0.97,
  active: 0.95,
  badgePulse: 1.15,
  iconTap: 0.90,
  checkbox: 1.10,
} as const;
```

---

## 4. Tokens de Opacidade

| Token | Valor | Variavel CSS | Uso |
|---|---|---|---|
| `opacity-enter` | `0` | `--opacity-enter` | Estado inicial de elementos que entram em tela (fade in) |
| `opacity-visible` | `1` | `--opacity-visible` | Estado final de elementos visiveis |

```css
@theme {
  --opacity-enter: 0;
  --opacity-visible: 1;
}
```

```typescript
export const OPACITY = {
  enter: 0,
  visible: 1,
  muted: 0.6,
  disabled: 0.4,
  placeholder: 0.3,
} as const;
```

---

## 5. Tokens de Streaming — Repasse AI

Tokens exclusivos do contexto de IA. Implementados como variaveis CSS adicionais nos modulos consumidores (Admin, Cedente, Cessionario).

| Token | Valor | Uso |
|---|---|---|
| `--stream-token-delay` | `30ms` | Delay entre cada token renderizado no DOM. Simula velocidade de digitacao natural (~33 tokens/segundo). |
| `--stream-paragraph-pause` | `120ms` | Pausa adicional apos quebra de paragrafo (`\n\n`). Cria ritmo de leitura natural. |
| `--stream-token-fade` | `var(--duration-fast)` (150ms) | Duracao do fade-in de cada token individual. |
| `--stream-batch-size` | `3` (tokens) | Tokens acumulados antes de cada re-render do DOM. Reduz re-renders em ~66%. |

### 5.1 Cursor/Caret de Streaming

| Propriedade | Valor |
|---|---|
| Formato | Barra vertical (`\|`) com 2px de largura |
| Cor (light) | `var(--foreground)` (#0A0A0A) |
| Cor (dark) | `var(--foreground)` (#FAFAFA) |
| Animacao | `opacity` alternando entre 1 e 0 |
| Duracao do ciclo | `1000ms` (500ms visivel + 500ms invisivel) |
| Easing | `steps(1)` |
| Remocao | Fade-out de `--duration-fast` (150ms) quando streaming termina |

```css
@keyframes blink-caret {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}

.ai-stream-caret {
  display: inline-block;
  width: 2px;
  height: 1.2em;
  background: var(--foreground);
  animation: blink-caret 1000ms steps(1) infinite;
  vertical-align: text-bottom;
  margin-left: 1px;
}
```

### 5.2 Regras de Streaming

1. Tokens sao renderizados em batches de `--stream-batch-size` (3 tokens).
2. Cada batch faz fade-in com `--stream-token-fade` (150ms) e easing `--ease-out`.
3. Apos `\n\n` (quebra de paragrafo), pausa adicional de `--stream-paragraph-pause` (120ms).
4. Cursor pisca durante todo o streaming e desaparece com fade-out de 150ms ao final.
5. Scroll automatico: o container de chat faz scroll suave (`scroll-behavior: smooth`) para acompanhar o ultimo token renderizado.

### 5.3 `prefers-reduced-motion` para Streaming

Quando `prefers-reduced-motion: reduce` esta ativo:

- `--stream-token-delay` passa para `0ms` (texto aparece instantaneamente).
- `--stream-token-fade` passa para `0ms` (sem fade-in por token).
- `--stream-paragraph-pause` passa para `0ms`.
- Cursor caret e removido.
- Texto completo exibido como bloco unico com fade-in de `--duration-normal` (250ms).

---

## 6. Skeleton Loading

Spinners sao **PROIBIDOS** em todos os modulos (ShiftLabs Stacks v7.0 secao 2.4.4). Todo estado de carregamento usa skeleton screens com shimmer.

### 6.1 Shimmer Animation (CSS — Web)

```css
@keyframes shimmer {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}

[data-slot="skeleton"] {
  position: relative;
  overflow: hidden;
  background-color: var(--muted);
  border-radius: var(--radius-md);
}

[data-slot="skeleton"]::after {
  content: '';
  position: absolute;
  inset: 0;
  transform: translateX(-100%);
  background: linear-gradient(
    90deg,
    transparent,
    oklch(1 0 0 / 8%),
    transparent
  );
  animation: shimmer 1.8s ease-in-out infinite;
}

.dark [data-slot="skeleton"]::after {
  background: linear-gradient(
    90deg,
    transparent,
    oklch(1 0 0 / 5%),
    transparent
  );
}
```

### 6.2 Shimmer Animation (Reanimated — Mobile)

```typescript
const shimmerValue = useSharedValue(0);

useEffect(() => {
  shimmerValue.value = withRepeat(
    withTiming(1, { duration: DURATION.crawl, easing: Easing.inOut(Easing.ease) }),
    -1,
    true
  );
}, []);
```

### 6.3 Propriedades do Shimmer

| Propriedade | Valor |
|---|---|
| Direcao | Left-to-right |
| Duracao do ciclo | `1.8s` |
| Timing function | `ease-in-out` |
| Iteracao | `infinite` |
| Cor base | `var(--muted)` |
| Cor highlight (light) | `oklch(1 0 0 / 8%)` |
| Cor highlight (dark) | `oklch(1 0 0 / 5%)` |
| Border radius | `var(--radius-md)` |

### 6.4 Skeleton Transition (Saida)

| Propriedade | Valor |
|---|---|
| Trigger | Dados carregados (TanStack Query `isSuccess`) |
| Animacao de saida do skeleton | Fade out: `opacity 1 -> 0`, duracao `--duration-fast` (150ms), easing `--ease-out` |
| Animacao de entrada do conteudo | Fade in: `opacity 0 -> 1`, duracao `--duration-normal` (250ms), easing `--ease-out` |

```tsx
<AnimatePresence mode="wait">
  {isLoading ? (
    <motion.div
      key="skeleton"
      exit={{ opacity: 0 }}
      transition={{ duration: 0.15, ease: [0.16, 1, 0.3, 1] }}
    >
      <Skeleton />
    </motion.div>
  ) : (
    <motion.div
      key="content"
      initial={{ opacity: 0 }}
      animate={{ opacity: 1 }}
      transition={{ duration: 0.25, ease: [0.16, 1, 0.3, 1] }}
    >
      <Content />
    </motion.div>
  )}
</AnimatePresence>
```

### 6.5 Contextos por Modulo

| Contexto | Admin | Cedente | Cessionario | Repasse AI |
|---|---|---|---|---|
| Dashboard — KPIs / widgets | Skeleton por card | Skeleton 4 cards | Skeleton widgets independentes | — |
| Pipeline — cards de caso | Skeleton por card | — | — | — |
| Tabelas com filtros | Skeleton por linha (min 5 linhas) | Skeleton 5 rows x 4 cols (Financeiro) | Skeleton linhas (5-8) | — |
| Drawer / detalhe de caso | Skeleton por secao | Skeleton tab bar + 5 linhas | Skeleton full-page | — |
| Modal de confirmacao | Placeholder estatico | Placeholder estatico | Placeholder estatico | — |
| Graficos / Relatorios | Skeleton area do grafico | — | Skeleton chart placeholder | — |
| Accordion (conteudo expandido) | Skeleton por conteudo | Skeleton por conteudo | Skeleton por conteudo | — |
| Sidebar / menu lateral | Nenhum (estado local) | Nenhum | Nenhum | — |
| Toast / Notificacao | Nenhum | Nenhum | Nenhum | — |
| Simulador de cenarios | — | 4 cards skeleton + texto apos 5s | — | — |
| Lista de propostas | — | 3 rows avatar + linhas + badge | Skeleton cards | — |
| Chat IA | — | 3 message bubbles | 3-5 baloes skeleton | 3 linhas skeleton (texto), cards+tabela (analise) |
| Dossie de documentos | — | Lista 4 items + icone + status | — | — |
| Timeline de caso | — | 5 nodes verticais | Blocos timeline skeleton | — |
| Marketplace (oportunidades) | — | — | Grid 3-6 cards skeleton | — |
| Score / Analise financeira IA | — | — | — | Header + 4 cards + tabela 6x3 + score badge |

### 6.6 Regra de Skeleton em Tabelas

1. **Header da tabela:** renderizado imediatamente (colunas conhecidas).
2. **Corpo da tabela:** renderiza entre 5 e 10 linhas skeleton (min 5).
3. **Cada celula skeleton:** respeita a largura da coluna e usa `border-radius: var(--radius-sm)`.
4. **Altura do skeleton:** `h-4` para texto, `h-8` para badges/chips, `h-10` para avatar + nome.

### 6.7 Timeout de Skeleton

| Momento | Acao |
|---|---|
| 0s - 5s | Skeleton visivel com shimmer |
| 5s+ | `[Cedente]` Texto "Estamos calculando seus cenarios..." com fade-in 250ms |
| 10s+ | `[Repasse AI]` Texto "Isso esta demorando mais que o esperado..." com fade-in 500ms |

### 6.8 `prefers-reduced-motion` para Skeleton

Shimmer animation desabilitado. Skeleton exibe cor solida `var(--muted)` sem animacao.

```css
@media (prefers-reduced-motion: reduce) {
  [data-slot="skeleton"]::after {
    animation: none;
    display: none;
  }
}
```

---

## 7. Padroes de Animacao

### 7.1 Page Transitions

| Propriedade | Valor |
|---|---|
| **Trigger** | Mudanca de rota (TanStack Router, Next.js App Router, expo-router) |
| **Duracao** | `--duration-page` (500ms) — `[Admin]` usa `--duration-slow` (400ms) na saida |
| **Easing** | `--ease-in-out` |
| **Propriedades animadas** | `opacity`, `transform: translateY` |
| **Enter** | `opacity: 0 -> 1`, `y: 8px -> 0` |
| **Exit** | `opacity: 1 -> 0`, `y: 0 -> -8px` |
| **`prefers-reduced-motion`** | Apenas cross-fade `opacity` sem deslocamento. Duracao reduzida para `150ms`. |

```tsx
<AnimatePresence mode="wait">
  <motion.div
    key={routeKey}
    initial={{ opacity: 0, y: 8 }}
    animate={{ opacity: 1, y: 0 }}
    exit={{ opacity: 0, y: -8 }}
    transition={{ duration: 0.5, ease: [0.65, 0, 0.35, 1] }}
  >
    {children}
  </motion.div>
</AnimatePresence>
```

**Mobile (Reanimated + expo-router):** `[Cedente]` `[Cessionario]`

| Propriedade | Valor |
|---|---|
| Trigger | Navegacao entre screens via expo-router |
| Duracao | `400ms` (`DURATION.slow`) |
| Enter (push) | `translateX: screenWidth -> 0`, `opacity: 0.8 -> 1` |
| Exit (push) | `translateX: 0 -> -screenWidth * 0.3` |
| `accessibilityReduceMotion` | Cross-fade sem deslocamento. Duracao `150ms`. |

---

### 7.2 Modal / Dialog

| Propriedade | Valor |
|---|---|
| **Trigger** | Acao do usuario (clique em botao CTA, confirmacao) |
| **Duracao entrada** | `--duration-normal` (250ms) |
| **Duracao saida** | `--duration-fast` (150ms) |
| **Easing entrada** | `--ease-out` |
| **Easing saida** | `--ease-in-out` |
| **Overlay** | `opacity: 0 -> 1` com `backdrop-filter: blur(4px)` |
| **Content enter** | `opacity: 0 -> 1`, `scale: 0.95 -> 1`, `y: 10px -> 0` |
| **Content exit** | `opacity: 1 -> 0`, `scale: 1 -> 0.95` |
| **`prefers-reduced-motion`** | Apenas `opacity`. Sem scale ou y. Duracao `100ms`. |

```tsx
const modalVariants = {
  initial: { opacity: 0, scale: 0.95, y: 10 },
  animate: {
    opacity: 1, scale: 1, y: 0,
    transition: { duration: duration.normal, ease: easing.easeOut },
  },
  exit: {
    opacity: 0, scale: 0.95,
    transition: { duration: duration.fast, ease: easing.easeInOut },
  },
};
```

**Mobile:** `[Cedente]` `[Cessionario]` Content slide from bottom com `SPRING.default`. Haptics `impactAsync(Medium)` ao abrir.

---

### 7.3 Drawer / Sheet

| Propriedade | Valor |
|---|---|
| **Trigger** | Clique em "Ver detalhes", botao de filtro, swipe up (mobile) |
| **Duracao entrada** | `--duration-normal` (250ms) |
| **Duracao saida** | `--duration-fast` (150ms) |
| **Easing entrada** | `--ease-out` |
| **Easing saida** | `--ease-in-out` |
| **Drawer lateral (web)** | `translateX: 100% -> 0` (slide da direita) |
| **Bottom sheet (mobile)** | `translateY: 100% -> 0` (slide de baixo) |
| **Drag-to-dismiss** | Habilitado. Spring: `spring-snappy`. Dismiss threshold: 30% da altura. |
| **`prefers-reduced-motion`** | Apenas fade. Sem slide. Duracao `100ms`. |

**Mobile:** `[Cedente]` `[Cessionario]` Snap points: `[0.5, 0.85]` do screenHeight. Haptics `impactAsync(Light)` em cada snap.

---

### 7.4 Toast / Notification

| Propriedade | Valor |
|---|---|
| **Trigger** | Evento do sistema (alerta de SLA, deposito confirmado, nova proposta, erro) |
| **Duracao entrada** | `--duration-normal` (250ms) |
| **Duracao saida** | `--duration-fast` (150ms) |
| **Easing entrada** | `--ease-out` |
| **Posicao** | Top-right (web desktop), bottom-center (mobile web), top-center (mobile nativo) |
| **Enter** | `opacity: 0 -> 1`, `translateY: -16px -> 0` |
| **Exit** | `opacity: 1 -> 0`, `translateX: 0 -> 100%` (swipe right dismiss) |
| **Auto-dismiss** | 5 segundos (sucesso), 8 segundos (erro `[Cedente]`), persistent (acao requerida) |
| **Stacking** | Maximo 3 toasts visiveis. Novos empurram anteriores com `spring-gentle`. |
| **`prefers-reduced-motion`** | Apenas opacity. Sem translate ou scale. |

**Toasts criticos:** `[Cessionario]` Borda esquerda colorida (amarelo aviso, vermelho urgente) + pulse sutil na borda: 2 ciclos de `opacity 0.7 -> 1` com 300ms cada.

**Mobile:** `[Cedente]` `[Cessionario]` Haptics `notificationAsync(Success)` para sucesso, `notificationAsync(Error)` para erro.

---

### 7.5 Accordion / Collapsible

| Propriedade | Valor |
|---|---|
| **Trigger** | Clique no header do accordion |
| **Duracao** | `--duration-normal` (250ms) |
| **Easing** | `--ease-in-out` |
| **Expand** | `height: 0 -> auto` + `opacity: 0 -> 1` |
| **Collapse** | `height: auto -> 0` + `opacity: 1 -> 0` |
| **Chevron icon** | `rotate: 0deg -> 180deg` sincronizado |
| **`prefers-reduced-motion`** | Conteudo aparece/desaparece instantaneamente. Chevron sem animacao. |

```css
[data-state='open'] > [data-slot='accordion-content'] {
  animation: accordion-down var(--duration-normal) var(--ease-in-out);
}

[data-state='closed'] > [data-slot='accordion-content'] {
  animation: accordion-up var(--duration-normal) var(--ease-in-out);
}

@keyframes accordion-down {
  from { height: 0; opacity: 0; }
  to { height: var(--radix-accordion-content-height); opacity: 1; }
}

@keyframes accordion-up {
  from { height: var(--radix-accordion-content-height); opacity: 1; }
  to { height: 0; opacity: 0; }
}
```

**Mobile:** `[Cedente]` `[Cessionario]` Spring `SPRING.gentle`. Haptics `selectionAsync()` ao toggle.

---

### 7.6 Dropdown / Popover

| Propriedade | Valor |
|---|---|
| **Trigger** | Clique em botao de filtro, menu de contexto, select |
| **Duracao entrada** | `--duration-fast` (150ms) |
| **Duracao saida** | `--duration-instant` (100ms) |
| **Easing entrada** | `--ease-out` |
| **Enter** | `opacity: 0 -> 1`, `scale: 0.95 -> 1`, `y: -4px -> 0` |
| **Exit** | `opacity: 1 -> 0`, `scale: 0.95` |
| **Transform origin** | Calculado dinamicamente com base no posicionamento (Radix Primitives) |
| **`prefers-reduced-motion`** | Apenas opacity. Duracao `50ms`. |

**Mobile:** `[Cedente]` `[Cessionario]` Bottom sheet ao inves de popover flutuante.

**Tooltip (botao desabilitado):** `[Cessionario]` Delay de entrada 300ms, delay de saida 100ms, fade in 150ms.

---

### 7.7 Hover States (Web Only)

| Elemento | Hover | Duracao | Easing |
|---|---|---|---|
| Card interativo | `scale(1.02)` + `box-shadow` elevacao + `border-color` sutil | `100ms` | `--ease-spring` |
| Botao Primary | `background-color` shift (escurece 5%) | `100ms` | `--ease-out` |
| Botao Secondary | `background-color` -> `var(--accent)` | `100ms` | `--ease-out` |
| Link de Sidebar | `background-color` -> `var(--sidebar-accent)` | `100ms` | `--ease-out` |
| Chip de filtro | `background-color` shift + `scale(1.02)` | `100ms` | `--ease-out` |
| Row de tabela | `background-color` -> `var(--muted)` a 50% | `100ms` | `--ease-out` |
| Icone de acao | `color: var(--primary)` + `scale: 1.10` | `100ms` | `--ease-spring` |

```css
.hover-lift {
  transition: transform var(--duration-instant) var(--ease-out),
              box-shadow var(--duration-instant) var(--ease-out);
}

.hover-lift:hover {
  transform: scale(var(--scale-hover));
  box-shadow: 0 4px 12px oklch(0 0 0 / 8%);
}

.hover-lift:active {
  transform: scale(var(--scale-press));
}
```

**`prefers-reduced-motion`:** Hover states de cor sao mantidos. Scale transitions sao removidos.

---

### 7.8 Press States (Mobile Only) `[Cedente]` `[Cessionario]`

| Elemento | Visual | Haptics | Duracao |
|---|---|---|---|
| Botao primario | `scale: 0.97` | `impactAsync(Light)` | Spring `SPRING.snappy` |
| Botao secundario | `scale: 0.97` + `opacity: 0.8` | `impactAsync(Light)` | Spring `SPRING.snappy` |
| Card (caso, proposta) | `scale: 0.98` + `opacity: 0.95` | `selectionAsync()` | Spring `SPRING.snappy` |
| Item de lista | `backgroundColor` shift para muted | `selectionAsync()` | `100ms` timing |
| Icone de acao | `scale: 0.90` | `impactAsync(Medium)` | Spring `SPRING.snappy` |
| Toggle / Switch | Spring snap | `impactAsync(Light)` | Spring `SPRING.bouncy` |
| Tab bar item | `scale: 0.95` + icon color shift | `selectionAsync()` | Spring `SPRING.snappy` |

**`accessibilityReduceMotion`:** Scale removido. Apenas opacity e color feedback. Haptics mantidos.

---

### 7.9 Focus States

| Elemento | Focus Ring | Duracao | Easing |
|---|---|---|---|
| Todos os interativos (web) | `box-shadow: 0 0 0 2px var(--background), 0 0 0 4px var(--ring)` | `100ms` | `--ease-out` |
| Links | `outline: 2px solid var(--ring)` + `outline-offset: 2px` | `100ms` | `--ease-out` |

```css
*:focus-visible {
  outline: none;
  box-shadow: 0 0 0 2px var(--background), 0 0 0 4px var(--ring);
  transition: box-shadow var(--duration-instant) var(--ease-out);
}
```

**`prefers-reduced-motion`:** Focus states sao mantidos integralmente — obrigatorios para acessibilidade.

**Mobile:** Focus nao se aplica. Acessibilidade via `accessibilityLabel`, `accessibilityRole` e highlight nativo VoiceOver/TalkBack.

---

### 7.10 List Items / Stagger

| Propriedade | Valor |
|---|---|
| **Trigger** | Dados carregados (apos skeleton) ou filtro aplicado |
| **Duracao por item** | `--duration-normal` (250ms) |
| **Delay entre items** | `50ms` (stagger) |
| **Maximo de items animados** | 10 items. Apos o 10o, items aparecem sem delay. |
| **Easing** | `--ease-out` |
| **Enter** | `opacity: 0 -> 1`, `y: 12px -> 0` |
| **`prefers-reduced-motion`** | Todos os items aparecem simultaneamente com fade `100ms`. |

```tsx
<motion.div
  variants={{ hidden: {}, show: { transition: { staggerChildren: 0.05 } } }}
  initial="hidden"
  animate="show"
>
  {items.map((item) => (
    <motion.div
      key={item.id}
      variants={{
        hidden: { opacity: 0, y: 12 },
        show: { opacity: 1, y: 0, transition: { duration: 0.25, ease: [0.16, 1, 0.3, 1] } },
      }}
    />
  ))}
</motion.div>
```

**Mobile:** `[Cedente]` `[Cessionario]` Stagger `30ms` entre items. `FlatList` com `entering` prop do Reanimated. Spring `SPRING.gentle`.

---

### 7.11 Drag-and-Drop — Pipeline `[Admin]`

| Fase | Comportamento | Duracao |
|---|---|---|
| **Lift** | Card escala para `1.03`, sombra aumenta, z-index elevado. Placeholder translucido (opacity 0.3). | `--duration-fast` (150ms) |
| **Drag** | Card segue o cursor. Coluna de destino recebe highlight de borda. Outros cards reordenam com spring. | Em tempo real |
| **Drop (sucesso)** | Card solta na nova posicao com `spring-default`. | Spring settle ~300ms |
| **Drop (cancelado)** | Card retorna com `spring-snappy`. | Spring settle ~200ms |

```tsx
<Reorder.Group axis="y" values={cards} onReorder={setCards}>
  {cards.map((card) => (
    <Reorder.Item
      key={card.id}
      value={card}
      whileDrag={{
        scale: 1.03,
        boxShadow: '0 8px 24px oklch(0 0 0 / 15%)',
        zIndex: 50,
        cursor: 'grabbing',
      }}
      layout
      transition={spring.default}
    >
      <PipelineCard data={card} />
    </Reorder.Item>
  ))}
</Reorder.Group>
```

**`prefers-reduced-motion`:** Card muda para visual `grabbed` sem animacao. Reorder visual mantido (essencial). Settle com snap instantaneo (sem spring).

---

### 7.12 Onboarding Wizard (Step Transitions) `[Cedente]`

| Propriedade | Valor (Web) | Valor (Mobile) |
|---|---|---|
| Trigger | Click em "Avancar" / "Voltar" ou swipe (mobile) |
| Duracao | `500ms` (`--duration-page`) | `500ms` (`DURATION.page`) |
| Easing | `--ease-in-out` | `Easing.inOut(Easing.ease)` |
| Avancar (enter) | `opacity: 0 -> 1`, `x: 40px -> 0` | `translateX: screenWidth -> 0` |
| Avancar (exit) | `opacity: 1 -> 0`, `x: 0 -> -40px` | `translateX: 0 -> -screenWidth` |
| `prefers-reduced-motion` | Cross-fade sem deslocamento. Duracao `150ms`. | Idem. |

**Barra de Progresso:** Width transition de `(step-1)/5 * 100%` para `step/5 * 100%`. Duracao `400ms`, easing `--ease-spring`. Mobile: `withSpring(targetWidth, SPRING.bouncy)`.

**Timer de 10 Segundos (Etapa 3):** Barra de progresso fina (3px) com `width: 0% -> 100%` em 10s linear. Ao completar, botao habilita com `spring-bouncy`. Mobile: Haptics `notificationAsync(Success)`.

**Expansao de Card de Cenario (Etapa 4):** `layoutId` transition com borda `2px solid var(--primary)`, outros cards `opacity -> 0.6`. Checkbox com pulse 3 ciclos.

---

### 7.13 Scroll Animations (Landing Page) `[Cedente]`

| Elemento | Trigger | Enter | Duracao | Easing |
|---|---|---|---|---|
| Hero headline | Viewport load | `opacity: 0 -> 1`, `y: 24px -> 0` | `700ms` | `--ease-out` |
| Hero CTA | Load + 200ms delay | `opacity: 0 -> 1`, `y: 16px -> 0`, `scale: 0.95 -> 1` | `500ms` | `--ease-spring` |
| Feature cards (3) | `whileInView` (30%) | Stagger: `opacity: 0 -> 1`, `y: 32px -> 0`, delay `100ms` | `500ms` | `--ease-out` |
| Testimonials | `whileInView` (20%) | `opacity: 0 -> 1`, `y: 24px -> 0` | `700ms` | `--ease-in-out` |
| Stats counters | `whileInView` (50%) | Number count-up de 0 ao valor | `1000ms` | `--ease-out` |
| Final CTA | `whileInView` (30%) | `opacity: 0 -> 1`, `scale: 0.97 -> 1` | `500ms` | `--ease-out` |

`viewport.once: true` — animacao roda apenas na primeira vez.

**`prefers-reduced-motion`:** Todos os elementos visiveis desde a montagem. `whileInView` desabilitado.

---

### 7.14 Real-Time Updates `[Cessionario]`

**Nova Oportunidade na Lista:**

| Propriedade | Valor |
|---|---|
| Trigger | Evento Supabase Realtime (`INSERT`) |
| Animacao de entrada | `{ opacity: 0, scale: 0.95, y: -20 }` -> `{ opacity: 1, scale: 1, y: 0 }` |
| Duracao | `--duration-slow` (400ms) |
| Highlight temporario | Background `var(--primary)` com 5% opacity por 3 segundos |
| Deslocamento dos demais | `spring-gentle` |

**Atualizacao de Status:** Flash no badge: `opacity 0.5 -> 1` (2 ciclos, 200ms cada) + highlight temporario 2s no card/row.

**Badge de Notificacao:** Bounce `spring-bouncy` no `scale`: `1 -> 1.3 -> 1`.

---

### 7.15 Chat / IA — Typing + Streaming `[Cedente]` `[Cessionario]`

**Typing Indicator (Guardiao do Retorno + Analista de Oportunidades):**

| Propriedade | Valor |
|---|---|
| Trigger | SSE stream inicia (antes do primeiro token) |
| Elemento | 3 dots pulsando sequencialmente |
| Tamanho de cada dot | 8px de diametro |
| Cor | `var(--muted-foreground)` |
| Animacao | `scale` de 0.4 a 1.0 + `opacity` de 0.4 a 1.0, stagger 150ms |
| Duracao do ciclo | `1200ms` |
| Easing | `--ease-in-out` |
| Espacamento entre dots | `6px` |
| `prefers-reduced-motion` | Dots estaticos visiveis + texto "Processando..." |

```css
@keyframes dot-pulse {
  0%, 100% { transform: scale(0.4); opacity: 0.4; }
  50% { transform: scale(1); opacity: 1; }
}

.ai-typing-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--muted-foreground);
  animation: dot-pulse 1200ms var(--ease-in-out) infinite;
}

.ai-typing-dot:nth-child(2) { animation-delay: 150ms; }
.ai-typing-dot:nth-child(3) { animation-delay: 300ms; }
```

**Message Enter:**

| Propriedade | Valor |
|---|---|
| Mensagem do usuario | `opacity: 0 -> 1`, `scale: 0.9 -> 1`, `x: 12px -> 0`. Duracao `250ms`. |
| Mensagem do agente | `opacity: 0 -> 1`, `x: -12px -> 0`. Typing indicator faz fade-out. |
| Streaming | Texto token-a-token conforme contratos da secao 5. Cursor piscante. |
| Pos-streaming | Stagger fade-in de elementos ricos (tabelas, badges, botoes) com delay 80ms, duracao 400ms. |
| Auto-scroll | `scroll-behavior: smooth` acompanhando ultimo token. |

---

### 7.16 Data Visualization `[Cessionario]`

| Tipo | Animacao | Duracao | Easing |
|---|---|---|---|
| Bar chart | Barras crescem de `height: 0`. Stagger 80ms entre barras. | `400ms` | `--ease-out` |
| Line chart | Path draw com `strokeDasharray` / `strokeDashoffset` | `800ms` | `--ease-in-out` |
| Donut/Pie chart | `strokeDashoffset` animado | `600ms` | `--ease-out` |
| Number counter | Contagem de 0 ao valor final. `Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' })` | `600ms` | `--ease-out` |
| Metricas Dashboard | Counter 0 -> valor + fade in label. Stagger 100ms entre widgets. | `600ms` | `--ease-out` |

**`prefers-reduced-motion`:** Valores finais renderizados instantaneamente. Sem contagem ou draw animation.

---

## 8. Estados do Agente IA `[Repasse AI]`

Maquina de estados: `Idle -> Thinking -> Streaming -> Complete` / `Error` / `Timeout`

### 8.1 Idle

Nenhum indicador visual. Campo de input ativo. Fade-out do indicador anterior com `--duration-fast` (150ms).

### 8.2 Thinking / Processing

Typing indicator (3 dots, ciclo 1200ms) dentro do balao do agente. `aria-label="O analista esta processando sua pergunta"`. Fade-in `--duration-normal` (250ms). Tempo maximo: 10 segundos.

### 8.3 Streaming

Tokens renderizados em batches de 3, fade-in 150ms cada. Cursor piscando 1s. Typing indicator faz fade-out `--duration-fast` (150ms). `aria-live="polite"`.

### 8.4 Complete

Cursor fade-out 150ms. Elementos ricos (tabelas, badges, botoes de acao) com stagger fade-in: delay 80ms entre elementos, duracao `--duration-slow` (400ms), easing `--ease-out`, `translateY(8px) -> 0`.

### 8.5 Error

Shake horizontal sutil no balao: `translateX` alternando -4px, 4px, -2px, 2px, 0. Duracao `400ms`, easing `--ease-out`, 1 repeticao. Fade-in da mensagem de erro 250ms. `role="alert"`, `aria-live="assertive"`.

```css
@keyframes shake-error {
  0% { transform: translateX(0); }
  20% { transform: translateX(-4px); }
  40% { transform: translateX(4px); }
  60% { transform: translateX(-2px); }
  80% { transform: translateX(2px); }
  100% { transform: translateX(0); }
}

.ai-error-shake {
  animation: shake-error 400ms var(--ease-out) 1;
}
```

### 8.6 Timeout

Progressive disclosure:

| Momento | Acao visual |
|---|---|
| 0s - SLA (5s ou 10s) | Typing indicator pulsando normalmente |
| 2x SLA | Mensagem "O Analista esta demorando mais que o esperado." + botoes "Aguardar" e "Tentar novamente". Fade-in `--duration-page` (500ms). |
| 120s (timeout absoluto) | Mensagem "Nao foi possivel obter a resposta. Tente novamente." |

### 8.7 `prefers-reduced-motion` para Estados

| Estado | Comportamento padrao | Comportamento reduzido |
|---|---|---|
| Thinking | Dots pulsando | Dots estaticos + texto "Processando..." |
| Streaming | Token-by-token com fade-in | Texto completo com fade-in unico 250ms |
| Complete | Stagger fade-in | Todos aparecem simultaneamente |
| Error | Shake + fade-in | Sem shake; apenas fade-in |
| Timeout | Fade-in progressivo | Mensagem aparece instantaneamente |

---

## 9. Confidence / Score Visualization `[Repasse AI]`

### 9.1 Score de Confianca — Admin (Painel Supervisao IA)

| Propriedade | Valor |
|---|---|
| Formato | Percentual numerico + barra de progresso horizontal |
| Animacao do numero | Number counting de 0 ate o valor final |
| Duracao | `--duration-slow` (400ms) |
| Easing | `--ease-out` |
| Animacao da barra | Width de 0% ate o valor final, sincronizada |

**Faixas de cor:**

| Faixa | Cor (light) | Cor (dark) |
|---|---|---|
| 80-100% (alta) | `#16A34A` (green-600) | `#22C55E` (green-500) |
| 50-79% (media) | `#CA8A04` (yellow-600) | `#EAB308` (yellow-500) |
| 0-49% (baixa) | `var(--destructive)` | `var(--destructive)` |

Barra e numero fazem transicao de cor com `--duration-fast` (150ms) ao atingir cada threshold.

### 9.2 Score de Risco — Cessionario (Chat)

| Propriedade | Valor |
|---|---|
| Formato | Badge circular com numero (1-10) + label ("Risco baixo/moderado/alto") |
| Animacao de entrada | Scale de 0.6 a 1.0 + fade-in |
| Duracao | `--duration-slow` (400ms) |
| Easing | `--ease-spring` |

**Faixas de cor:**

| Faixa | Cor (light) | Cor (dark) | Label |
|---|---|---|---|
| 1-3 (baixo) | `#16A34A` | `#22C55E` | "Risco baixo" |
| 4-6 (moderado) | `#CA8A04` | `#EAB308` | "Risco moderado" |
| 7-10 (alto) | `var(--destructive)` | `var(--destructive)` | "Risco alto" |

### 9.3 Number Counting para Valores Financeiros

Valores monetarios (Delta, comissao, Escrow, ROI) usam contagem de R$ 0,00 ate o valor final. Duracao `--duration-slow` (400ms), easing `--ease-out`. Formato `Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' })`. Atualizacao a cada 16ms (~60fps via `requestAnimationFrame`).

### 9.4 `prefers-reduced-motion` para Scores

- Number counting desativado. Valor final instantaneo.
- Score badge aparece sem scale, apenas fade-in `--duration-normal` (250ms).
- Transicoes de cor instantaneas.

---

## 10. Categorizacao de Animacoes

### 10.1 Definicao das Categorias

| Categoria | Definicao | Politica com `reduced-motion` |
|---|---|---|
| **Essencial** | Transmite informacao de estado indispensavel. Remove-la impediria o usuario de entender o que aconteceu. | Mantida com duracao reduzida (100ms maximo) |
| **Funcional** | Melhora a compreensao do fluxo, mas a informacao pode ser deduzida sem ela. | Reduzida a transicao instantanea (1ms) ou eliminada, com fallback visual estatico |
| **Decorativa** | Puramente estetica. Nao transmite informacao. | Removida completamente |

### 10.2 Tabela Unificada

| Animacao | Categoria | Modulos | Comportamento com `reduced-motion` |
|---|---|---|---|
| Skeleton loading | **Essencial** | Todos | Fundo solido muted, sem shimmer |
| Focus ring transition | **Essencial** | Todos (web) | Mantida — obrigatoria para acessibilidade |
| Typing indicator | **Essencial** | Cedente, Cessionario | Dots estaticos + texto "Processando..." |
| Timer de 10s (progress bar) | **Essencial** | Cedente | Barra preenche sem animacao |
| Accordion expand/collapse | **Essencial** | Todos | Mantida com duracao reduzida (100ms) |
| Drag-and-drop reorder (spring) | **Essencial** | Admin | Mantida com duracao reduzida |
| Page transitions (fade + translate) | **Funcional** | Todos | Cross-fade instantaneo |
| Modal / Dialog (scale + fade) | **Funcional** | Todos | Aparece/desaparece instantaneamente |
| Drawer / Sheet (slide) | **Funcional** | Todos | Aparece/desaparece instantaneamente |
| Toast entrada (translate + fade) | **Funcional** | Todos | Aparece instantaneamente |
| Dropdown / Popover (scale + fade) | **Funcional** | Todos | Aparece/desaparece instantaneamente |
| Skeleton shimmer | **Funcional** | Todos | Skeleton estatico |
| Wizard step transitions | **Funcional** | Cedente | Cross-fade sem slide |
| Chat message enter | **Funcional** | Cedente, Cessionario | Fade sem deslocamento |
| Real-time item enter | **Funcional** | Cessionario | Item aparece instantaneamente |
| Drag-and-drop settle | **Funcional** | Admin | Snap instantaneo (sem spring) |
| Drag-and-drop lift/shadow | **Decorativa** | Admin | Card visual `grabbed` sem animacao |
| Hover scale (cards, botoes) | **Decorativa** | Todos (web) | Apenas `background-color` muda |
| Hover box-shadow | **Decorativa** | Todos (web) | Removida |
| Stagger de lista | **Decorativa** | Todos | Items aparecem simultaneamente |
| Scroll reveal (landing) | **Decorativa** | Cedente | Conteudo visivel desde montagem |
| Hero entrance | **Decorativa** | Cedente | Conteudo visivel desde montagem |
| Badge pulse | **Decorativa** | Cedente, Cessionario | Badge estatico |
| Stats count-up | **Decorativa** | Cedente, Cessionario | Numero final direto |
| Press scale (mobile) | **Decorativa** | Cedente, Cessionario | Apenas opacity + haptics |
| Chart animate-in | **Decorativa** | Cessionario | Valores finais diretos |
| Number counting (scores) | **Decorativa** | Repasse AI | Valor final instantaneo |
| Streaming token-by-token | **Funcional** | Repasse AI | Texto completo com fade-in unico |
| Error shake | **Funcional** | Repasse AI | Sem shake, apenas fade-in |

---

## 11. Acessibilidade

### 11.1 `prefers-reduced-motion` — Web (Obrigatorio)

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }

  /* Excecao: animacoes essenciais mantidas com duracao minima */
  [data-motion='essential'],
  [data-motion='essential'] *,
  [data-motion='essential']::before,
  [data-motion='essential']::after {
    animation-duration: 100ms !important;
    transition-duration: 100ms !important;
  }
}
```

### 11.2 Framer Motion Global

```typescript
import { MotionConfig, LazyMotion, domAnimation } from 'framer-motion';

export function MotionProvider({ children }: { children: React.ReactNode }) {
  return (
    <MotionConfig reducedMotion="user">
      <LazyMotion features={domAnimation}>
        {children}
      </LazyMotion>
    </MotionConfig>
  );
}
```

### 11.3 Hook Utilitario (Web)

```typescript
import { useReducedMotion } from 'framer-motion';

export function useMotionSafe() {
  const shouldReduce = useReducedMotion();
  return {
    shouldAnimate: !shouldReduce,
    duration: shouldReduce ? 0 : undefined,
    transition: shouldReduce ? { duration: 0 } : undefined,
  };
}
```

### 11.4 Mobile — `accessibilityReduceMotionEnabled` `[Cedente]` `[Cessionario]`

```typescript
import { useReducedMotion } from 'react-native-reanimated';

export function useMotionSafe() {
  const reanimatedReduced = useReducedMotion();
  return {
    shouldAnimate: !reanimatedReduced,
    duration: reanimatedReduced ? 0 : undefined,
  };
}
```

### 11.5 Dynamic Type (Mobile) `[Cedente]` `[Cessionario]`

Quando Dynamic Type esta ativo com fontes maiores, animacoes de layout (expansao de cards, accordions, chat bubbles) devem acomodar alturas maiores sem truncamento. Implementar via `onLayout` para medir altura real antes de animar.

### 11.6 ARIA para Streaming de IA `[Repasse AI]`

| Requisito | Implementacao |
|---|---|
| Container de streaming | `aria-live="polite"` + `aria-atomic="false"` |
| Typing indicator | `aria-label="O analista esta processando sua pergunta"` + `role="status"` |
| Fim do streaming | `<span class="sr-only">Resposta completa.</span>` ao final |
| Mensagem de erro | `role="alert"` + `aria-live="assertive"` |
| Mensagem de timeout | `role="status"` + `aria-live="polite"` |
| Score de risco | `aria-label="Score de risco: [N] de 10. [Label da faixa]."` |
| Barra de confianca | `role="meter"` + `aria-valuenow` + `aria-valuemin="0"` + `aria-valuemax="100"` + `aria-label="Confianca da resposta: [N]%"` |
| ARIA live para toasts | `role="status"` + `aria-live="polite"`. Toasts de erro: `aria-live="assertive"`. |
| ARIA live para real-time | Novas oportunidades e atualizacoes anunciadas via `aria-live="polite"`. `[Cessionario]` |

### 11.7 Contraste

- Texto sobre cor: ratio minimo 4.5:1 (WCAG 2.1 AA).
- Texto grande (18px+ ou 14px+ bold): ratio minimo 3:1.
- Indicadores de cor devem sempre ter rotulo textual acessivel — nunca depender apenas de cor.
- Focus rings: contraste minimo 3:1 contra o fundo.
- Skeleton shimmer: highlight a 8% opacity para nao interferir com placeholders.

### 11.8 Como Testar

| Metodo | Instrucoes |
|---|---|
| Chrome DevTools | Rendering tab -> Emulate `prefers-reduced-motion` -> `reduce` |
| macOS | System Settings -> Accessibility -> Display -> Reduce Motion |
| Windows | Settings -> Accessibility -> Visual Effects -> Animation effects -> desativar |
| iOS | Settings -> Accessibility -> Motion -> Reduce Motion |
| Android | Settings -> Accessibility -> Remove animations |
| Automated (Playwright) | `await page.emulateMedia({ reducedMotion: 'reduce' })` |

---

## 12. Performance

### 12.1 Budget de Animacoes Simultaneas

| Constraint | Valor |
|---|---|
| Maximo de animacoes simultaneas por viewport (web) | **8** |
| Maximo de spring animations simultaneas | **4** |
| Maximo de animacoes simultaneas no chat IA | **3** (streaming + cursor + scroll) |

### 12.2 Propriedades Otimizadas

| Propriedade | Permitida | Justificativa |
|---|---|---|
| `transform` | Sim | Compositor-only. Escala, translacao e rotacao sem reflow. |
| `opacity` | Sim | Compositor-only. Fade sem repaint. |
| `box-shadow` | Com cautela | Causa repaint em area contida. Apenas hover lift e drag. |
| `background-color` | Com cautela | Causa repaint. Aceitavel para hover states. |
| `height` | Apenas accordion | Causa reflow. Justificado apenas para accordion expand/collapse. |
| `width`, `margin`, `padding`, `top`, `left` | **Proibido** | Causam reflow pesado. Usar `transform: translate`. |

### 12.3 `will-change`

Aplicar apenas quando a animacao e previsivel e iminente. Nunca estaticamente.

| Quando usar | Quando remover |
|---|---|
| `onMouseEnter` em cards com hover lift | `onMouseLeave` |
| `onDragStart` em cards do Pipeline | `onDragEnd` |
| Container de streaming IA ativo | Quando streaming termina |

### 12.4 Debounce de Re-renders durante Streaming `[Repasse AI]`

| Regra | Valor |
|---|---|
| Batch de tokens antes do re-render | 3 tokens (`--stream-batch-size`) |
| Metodo de rendering | `requestAnimationFrame` |
| Debounce de scroll | `100ms` com `requestAnimationFrame` |

### 12.5 Web — Framer Motion

| Otimizacao | Implementacao |
|---|---|
| `LazyMotion` + `domAnimation` | Reduz bundle ~50%. Importar `m` ao inves de `motion` para tree-shaking. |
| `layout` prop | Apenas em elementos que realmente mudam de posicao (Pipeline cards `[Admin]`). |
| `AnimatePresence mode="wait"` | Para page transitions. Evitar `mode="sync"` em rotas. |
| `AnimatePresence mode="popLayout"` | Para listas com reorder `[Admin]`. |
| CSS vs Framer Motion | CSS para hover/focus/color. Framer Motion para layout animations, AnimatePresence, springs e gestos. |
| Scroll animations | `whileInView` com `viewport={{ once: true }}`. `[Cedente]` |

### 12.6 Mobile — Reanimated `[Cedente]` `[Cessionario]`

| Diretriz | Implementacao |
|---|---|
| UI thread | Toda animacao via Reanimated `useSharedValue` + `useAnimatedStyle`. Nunca `setState` ou `Animated` API legacy. |
| FlatList | Animacao de enter apenas nos items visiveis. Limitar stagger a 6-10 items. `getItemLayout` para scroll performante. |
| Gesture Handler | Gestos processados nativamente, sem bridge JavaScript. |
| Memory | Cancelar animacoes ao desmontar (`cancelAnimation(sharedValue)` no cleanup). |
| Haptics | `expo-haptics` para feedback tatil. Nunca substitui feedback visual — sao complementares. |

### 12.7 Skeleton via CSS

Shimmer implementado via `background` animado com `@keyframes` (GPU composited automaticamente). Maximo 1 skeleton por container de chat. `display: none` apos fade-out (libera espaco no layout e remove do accessibility tree).

### 12.8 Regra de Ouro

Se uma animacao causa frame drops (< 60fps), ela deve ser simplificada ou removida. Performance sempre tem prioridade sobre polimento visual.

---

## 13. Mobile-Specific `[Cedente]` `[Cessionario]`

### 13.1 Gesture Feedback

| Gesto | Contexto | Feedback Visual | Haptics |
|---|---|---|---|
| Tap | Botoes, cards, list items | Scale `SPRING.snappy` | `impactAsync(Light)` |
| Long press | Item de caso (acao contextual) | Scale `0.95` + overlay backgroundColor | `impactAsync(Medium)` |
| Swipe right | Notificacao dismiss / acao rapida | `translateX: 0 -> screenWidth`, fade out | `impactAsync(Light)` |
| Swipe left | Proposta — ver detalhes / cancelar | Revela acao (drawer peek) | `selectionAsync()` |
| Pan vertical | Bottom sheet drag | Sheet segue o dedo. Snap nos pontos. | `impactAsync(Light)` em cada snap |
| Pinch | Documento (PDF preview) | Zoom in/out limites `[0.5, 3.0]` | Nenhum |
| Pull to refresh | Todas as listas | Icone rotativo customizado + skeleton shimmer | `impactAsync(Light)` ao disparar |
| Drag-to-dismiss | Bottom sheet | Spring `spring-snappy`. Threshold: 30%. Velocity: 500px/s. | `impactAsync(Light)` |

### 13.2 Haptics Guide

| Tipo | Funcao Expo | Quando usar |
|---|---|---|
| `impactAsync(Light)` | `Haptics.impactAsync(ImpactFeedbackStyle.Light)` | Tap em botao, swipe feedback, snap de sheet |
| `impactAsync(Medium)` | `Haptics.impactAsync(ImpactFeedbackStyle.Medium)` | Abertura de modal, long press, acao importante |
| `impactAsync(Heavy)` | `Haptics.impactAsync(ImpactFeedbackStyle.Heavy)` | Erro critico, acao destrutiva confirmada |
| `notificationAsync(Success)` | `Haptics.notificationAsync(NotificationFeedbackType.Success)` | Upload concluido, proposta aceita, timer desbloqueou |
| `notificationAsync(Warning)` | `Haptics.notificationAsync(NotificationFeedbackType.Warning)` | Timer expirando, documento pendente |
| `notificationAsync(Error)` | `Haptics.notificationAsync(NotificationFeedbackType.Error)` | Erro de validacao, upload falhou |
| `selectionAsync()` | `Haptics.selectionAsync()` | Toggle de accordion, troca de tab, selecao |

**Regra:** Haptics nunca substituem feedback visual. Sao complementares.

### 13.3 Navigation Transitions (expo-router)

| Tipo de navegacao | Animacao | Duracao |
|---|---|---|
| Push (avancar) | Slide from right | `400ms` |
| Pop (voltar) | Slide to right | `400ms` |
| Modal (bottom sheet) | Slide from bottom | `250ms` |
| Tab switch | Cross-fade | `150ms` |
| Replace (wizard step) | Cross-fade com slide horizontal | `500ms` |

```typescript
<Stack
  screenOptions={{
    animation: "slide_from_right",
    animationDuration: DURATION.slow,
    gestureEnabled: true,
    fullScreenGestureEnabled: true,
  }}
/>
```

### 13.4 Safe Areas e Keyboard

| Cenario | Comportamento |
|---|---|
| Keyboard aparece | Conteudo desliza para cima com `withTiming(DURATION.normal)` |
| Keyboard desaparece | Conteudo retorna com `withTiming(DURATION.normal)` |
| Bottom sheet + keyboard | Sheet ajusta altura com `spring-snappy` |

---

## 14. Contratos do Repasse AI para Modulos Consumidores

### 14.1 Modulo Cessionario (Webchat)

| Componente | Estado IA | Motion esperado |
|---|---|---|
| Balao de mensagem do agente | Thinking | Typing indicator (3 dots, 1200ms). `aria-label`. |
| Balao de mensagem do agente | Streaming | Tokens em batches de 3, fade-in 150ms, cursor 1s. `aria-live="polite"`. |
| Balao de mensagem do agente | Complete | Cursor fade-out 150ms. Elementos ricos com stagger 80ms delay, 400ms duracao. |
| Balao de mensagem do agente | Error | Shake 400ms + fade-in erro 250ms. `role="alert"`. |
| Balao de mensagem do agente | Timeout | Progressive disclosure. Fade-in 500ms. |
| Score de risco | Complete | Scale spring 0.6->1.0, fade-in 400ms. Cor por faixa. `aria-label`. |
| Valores financeiros | Complete | Number counting 0->valor em 400ms. Formato R$ XX.XXX,XX. |
| Tabela comparativa | Complete | Skeleton shimmer 1500ms ate dados. Fade-in 400ms da tabela. |
| Cenarios ROI | Complete | Stagger fade-in dos 3 cards com delay 80ms. |
| Skeleton de resposta | Thinking | 3 linhas skeleton com shimmer 1500ms. Fade-out 150ms quando primeiro token chega. |
| Campo de input (rate limit) | Rate limit | Campo desabilitado. Pulse sutil 500ms ao desbloquear. |
| Banner fallback | Calculadora ativa | Fade-in 250ms "Modo basico — sem analise da IA". |

### 14.2 Modulo Admin (Painel Supervisao IA)

| Componente | Estado IA | Motion esperado |
|---|---|---|
| Barra de confianca | Carregamento | Width 0->valor em 400ms. Cor por faixa. `role="meter"`. |
| Percentual de confianca | Carregamento | Number counting 0->valor em 400ms. Transicao de cor ao cruzar thresholds. |
| Badge de estado | Mudanca de estado | Fade-in 250ms. |
| Lista de interacoes (filtro) | Filtragem | Skeleton inline shimmer 1500ms. |
| Alerta de threshold | Confianca < 80% | Pulse sutil ciclo 2000ms no badge. |
| Separador de takeover | Inicio takeover | Fade-in 500ms da linha separadora. |
| Avatar de transicao | Takeover ativo | Crossfade 250ms do avatar IA para humano. |

### 14.3 Modulo Cedente (Guardiao do Retorno)

| Componente | Estado IA | Motion esperado |
|---|---|---|
| Balao do Guardiao | Thinking | Typing indicator identico ao Cessionario (3 dots, 1200ms). |
| Balao do Guardiao | Streaming | Tokens em batches de 3, fade-in 150ms, cursor piscando. |
| Balao do Guardiao | Complete | Cursor fade-out. Elementos ricos com stagger fade-in. |
| Balao do Guardiao | Error | Shake + mensagem de erro. |

---

## 15. Variaveis CSS Consolidadas (global.css)

Bloco completo de motion tokens para inclusao no `@theme` do `global.css` de cada modulo:

```css
@theme {
  /* -- Motion: Duracoes -- */
  --duration-instant: 100ms;
  --duration-fast: 150ms;
  --duration-normal: 250ms;
  --duration-slow: 400ms;
  --duration-page: 500ms;
  --duration-slower: 700ms;
  --duration-crawl: 1000ms;

  /* -- Motion: Easing -- */
  --ease-out: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-in-out: cubic-bezier(0.65, 0, 0.35, 1);
  --ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1);
  --ease-enter: cubic-bezier(0.0, 0.0, 0.2, 1);
  --ease-exit: cubic-bezier(0.4, 0.0, 1, 1);

  /* -- Motion: Scale -- */
  --scale-press: 0.97;
  --scale-hover: 1.02;
  --scale-active: 0.95;
  --scale-badge-pulse: 1.15;
  --scale-icon-tap: 0.90;
  --scale-checkbox: 1.10;

  /* -- Motion: Opacidade -- */
  --opacity-enter: 0;
  --opacity-visible: 1;

  /* -- Motion: Streaming IA -- */
  --stream-token-delay: 30ms;
  --stream-paragraph-pause: 120ms;
  --stream-token-fade: var(--duration-fast);
  --stream-batch-size: 3;
}
```

---

## 16. Changelog

| Versao | Data | Descricao |
|---|---|---|
| v1.0 | 2026-03-22 | Versao inicial (por modulo). |
| v2.0 | 2026-03-22 | Unificacao cross-modulo. Tokens, padroes e categorizacao consolidados. 7 tokens de duracao (escala completa com `slower` e `crawl`). 5 easing curves CSS + 4 spring presets (Framer Motion e Reanimated). 6 scale tokens cross-plataforma. Skeleton loading unificado com contextos por modulo. 16 padroes de animacao cobrindo Admin, Cedente, Cessionario e Repasse AI. Categorizacao unificada (Essencial/Funcional/Decorativa). Contratos do Repasse AI para modulos consumidores. Acessibilidade web + mobile + streaming IA. Performance web + mobile + streaming. Mobile-specific com gestos, haptics e transitions. |

---

*Documento normativo — Motion Spec unificado — Repasse Seguro — v2.0*
