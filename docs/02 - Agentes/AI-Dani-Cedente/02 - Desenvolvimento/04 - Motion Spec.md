# 04 - Motion Spec — AI-Dani-Cedente

## Especificação de Motion e Animações do Chat

| Campo | Valor |
|---|---|
| Destinatário | Time de Frontend / Design |
| Escopo | Especificação completa de motion, animações e transições do agente AI-Dani-Cedente — chat do Cedente |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D02 (Stacks — Framer Motion 12.x+), D03 (Brand Guide — tokens de design) |

---

> **📌 TL;DR**
>
> - **Framework de motion:** Framer Motion 12.x+ na interface web. React Native Reanimated 3.x+ no mobile.
> - **Princípio central:** motion serve ao Cedente — toda animação guia a atenção, confirma ação ou reduz ansiedade. Nunca decora.
> - **Tokens de duração:** fast (150ms), normal (250ms), slow (400ms). Base: `ease-out` para entradas, `ease-in` para saídas.
> - **Componentes animados:** bolhas de mensagem (entrada deslizante), indicador de digitando (bounce), sugestões (fade+slide), cards de análise (expand), modais (scale+fade), notificações proativas (slide-in).
> - **Redução de movimento:** `prefers-reduced-motion` obrigatório — todas as animações substituídas por fade simples 150ms.
> - **Performance:** animações exclusivamente em `transform` e `opacity` — nunca em `layout` properties que causem reflow.

---

## 1. Princípios de Motion

### 1.1 Hierarquia de Propósito

Toda animação no chat da Dani-Cedente deve atender a um destes propósitos, em ordem de prioridade:

1. **Confirmar ação:** feedback imediato de que o sistema recebeu o input do Cedente (ex: bolha aparece ao enviar mensagem).
2. **Guiar atenção:** direcionar o olhar para o conteúdo que importa agora (ex: card de análise de proposta expande ao surgir).
3. **Reduzir ansiedade:** indicar que o sistema está trabalhando (ex: indicador de digitando da Dani).
4. **Contextualizar transição:** mostrar de onde veio e para onde vai (ex: modal abre de onde o botão estava).

Animação sem um destes propósitos explícitos é removida.

### 1.2 Valores de Duração

| Token | Duração | Uso |
|---|---|---|
| `motion-fast` | 150ms | Micro-interações: hover, focus, toggle de checkbox, fade de tooltip |
| `motion-normal` | 250ms | Entrada de bolhas de mensagem, sugestões, badges de status |
| `motion-slow` | 400ms | Modais, cards de análise (expand), transições de tela |
| `motion-spring` | spring (stiffness: 300, damping: 30) | Bolhas de mensagem, elementos "vivos" |

### 1.3 Curvas de Easing

| Curva | Valor CSS | Uso |
|---|---|---|
| `ease-out` | `cubic-bezier(0, 0, 0.2, 1)` | Entradas — elemento desacelera ao chegar na posição final |
| `ease-in` | `cubic-bezier(0.4, 0, 1, 1)` | Saídas — elemento acelera ao sair |
| `ease-in-out` | `cubic-bezier(0.4, 0, 0.2, 1)` | Transições entre estados do mesmo elemento |
| `spring` | Framer Motion `spring` config | Bolhas e elementos físicos que "chegam com vida" |

### 1.4 Regra de Performance

> **⚙️ Regra absoluta de performance:**
>
> - Animar **apenas** `transform` (translate, scale, rotate) e `opacity`.
> - **Nunca** animar `width`, `height`, `top`, `left`, `padding`, `margin` ou qualquer propriedade que cause reflow de layout.
> - Para expandir/colapsar conteúdo (ex: card de análise): usar `scaleY` + `opacity` com `transform-origin: top`.
> - `will-change: transform` permitido para elementos animados frequentemente (bolhas recorrentes).

---

## 2. Tokens de Motion — Implementação

### 2.1 Framer Motion — Variantes Globais

```typescript
// src/lib/motion-tokens.ts — AI-Dani-Cedente

export const motionTokens = {
  // Durações
  fast: 0.15,
  normal: 0.25,
  slow: 0.4,

  // Easings
  easeOut: [0, 0, 0.2, 1] as const,
  easeIn: [0.4, 0, 1, 1] as const,
  easeInOut: [0.4, 0, 0.2, 1] as const,

  // Spring para bolhas
  spring: {
    type: 'spring' as const,
    stiffness: 300,
    damping: 30,
    mass: 0.8,
  },
} as const;

// Variante: entrada de bolha da Dani (da esquerda)
export const daniMessageVariant = {
  hidden: { opacity: 0, x: -12, scale: 0.96 },
  visible: {
    opacity: 1,
    x: 0,
    scale: 1,
    transition: motionTokens.spring,
  },
};

// Variante: entrada de bolha do Cedente (da direita)
export const cedenteMessageVariant = {
  hidden: { opacity: 0, x: 12, scale: 0.96 },
  visible: {
    opacity: 1,
    x: 0,
    scale: 1,
    transition: motionTokens.spring,
  },
};

// Variante: sugestões de conversa (fade + slide up)
export const suggestionVariant = {
  hidden: { opacity: 0, y: 8 },
  visible: (i: number) => ({
    opacity: 1,
    y: 0,
    transition: {
      duration: motionTokens.normal,
      ease: motionTokens.easeOut,
      delay: i * 0.05, // stagger 50ms entre chips
    },
  }),
};

// Variante: card de análise (expand de cima para baixo)
export const analysisCardVariant = {
  hidden: { opacity: 0, scaleY: 0.92, transformOrigin: 'top' },
  visible: {
    opacity: 1,
    scaleY: 1,
    transition: {
      duration: motionTokens.slow,
      ease: motionTokens.easeOut,
    },
  },
};

// Variante: modal (scale + fade)
export const modalVariant = {
  hidden: { opacity: 0, scale: 0.95 },
  visible: {
    opacity: 1,
    scale: 1,
    transition: {
      duration: motionTokens.slow,
      ease: motionTokens.easeOut,
    },
  },
  exit: {
    opacity: 0,
    scale: 0.95,
    transition: {
      duration: motionTokens.fast,
      ease: motionTokens.easeIn,
    },
  },
};

// Variante: notificação proativa (slide da direita)
export const notificationVariant = {
  hidden: { opacity: 0, x: 40 },
  visible: {
    opacity: 1,
    x: 0,
    transition: {
      duration: motionTokens.normal,
      ease: motionTokens.easeOut,
    },
  },
  exit: {
    opacity: 0,
    x: 40,
    transition: {
      duration: motionTokens.fast,
      ease: motionTokens.easeIn,
    },
  },
};
```

---

## 3. Animações por Componente

### 3.1 Bolha de Mensagem — Dani

| Propriedade | Valor |
|---|---|
| **Entrada** | Slide da esquerda: `x: -12 → 0` + `scale: 0.96 → 1` + `opacity: 0 → 1` |
| **Tipo** | Spring (stiffness: 300, damping: 30, mass: 0.8) |
| **Trigger** | Nova mensagem da Dani adicionada ao array de mensagens |
| **Sequência** | Se a Dani envia múltiplas bolhas seguidas: stagger 80ms entre cada bolha |
| **Scroll** | Container scrolls suavemente para a última mensagem após entrada (behavior: 'smooth') |

### 3.2 Bolha de Mensagem — Cedente

| Propriedade | Valor |
|---|---|
| **Entrada** | Slide da direita: `x: 12 → 0` + `scale: 0.96 → 1` + `opacity: 0 → 1` |
| **Tipo** | Spring (stiffness: 300, damping: 30) |
| **Trigger** | Cedente envia mensagem (após confirmação do envio) |
| **Estado "enviando"** | `opacity: 0.6` na bolha enquanto aguarda confirmação do servidor |

### 3.3 Indicador de Digitando (Typing Indicator)

| Propriedade | Valor |
|---|---|
| **Aparece** | Fade-in 150ms após a Dani receber o request do Cedente |
| **Animação dos pontos** | Bounce: `y: 0 → -4 → 0`, duração 1.4s, loop infinito |
| **Stagger** | 0.2s entre ponto 1, 2 e 3 |
| **Desaparece** | Fade-out 150ms quando a primeira bolha da Dani aparece |
| **Posição** | Mesma posição que a próxima bolha da Dani ocupará |

```typescript
// Animação dos pontos do typing indicator
export const typingDotVariant = {
  animate: {
    y: [0, -4, 0],
    transition: {
      duration: 1.4,
      repeat: Infinity,
      ease: 'easeInOut',
    },
  },
};
// Aplicar com delay: dot1 = 0s, dot2 = 0.2s, dot3 = 0.4s
```

### 3.4 Sugestões de Conversa (Conversation Starters)

| Propriedade | Valor |
|---|---|
| **Aparece** | Chat abre sem contexto (RN-DCE-008) |
| **Entrada** | Fade + slide up: `y: 8 → 0` + `opacity: 0 → 1`, 250ms ease-out |
| **Stagger** | 50ms entre cada chip (chips 1→4 aparecem em cascata) |
| **Hover** | Background muda para `--accent` com transição 150ms |
| **Clique** | Scale `1 → 0.96 → 1` em 100ms (feedback tátil), depois chip desaparece com fade-out 150ms |
| **Desaparece** | Fade-out coletivo 150ms quando Cedente envia primeira mensagem |

### 3.5 Card de Análise Inline

Usado em respostas que contêm análise de proposta, simulação de retorno, status de dossiê ou Escrow.

| Propriedade | Valor |
|---|---|
| **Entrada** | `scaleY: 0.92 → 1` + `opacity: 0 → 1`, 400ms ease-out, transform-origin: top |
| **Aparece depois da bolha** | Delay de 100ms após a bolha de texto da Dani aparecer |
| **Valores numéricos** | Counter animation: valor sobe de 0 até o valor final em 600ms (ease-out) para valores monetários (retorno líquido, valor de proposta) |
| **Badges de status** | Fade-in stagger 50ms entre cada badge (para lista de documentos do dossiê) |

### 3.6 Modal de Confirmação

| Propriedade | Valor |
|---|---|
| **Overlay** | Fade-in `opacity: 0 → 0.4` (backdrop), 250ms ease-out |
| **Modal** | Scale + fade: `scale: 0.95 → 1` + `opacity: 0 → 1`, 400ms ease-out |
| **Origem do scale** | Centro da tela (transform-origin: center) |
| **Saída** | Scale + fade: `scale: 1 → 0.95` + `opacity: 1 → 0`, 150ms ease-in |
| **AnimatePresence** | Obrigatório para animar saída (`mode="wait"`) |

### 3.7 Notificação Proativa

Disparada pelos eventos de RN-DCE-020 (nova proposta, prazo de Escrow, etc.).

| Propriedade | Valor |
|---|---|
| **Posição** | Toast no canto superior direito (ou inferior direito em mobile) |
| **Entrada** | Slide da direita: `x: 40 → 0` + `opacity: 0 → 1`, 250ms ease-out |
| **Saída** | Slide para direita: `x: 0 → 40` + `opacity: 1 → 0`, 150ms ease-in |
| **Auto-dismiss** | 5 segundos para notificações informativas. Alertas críticos (prazo vencendo, Escrow perto do fim) não têm auto-dismiss — requerem ação do Cedente. |
| **Stacking** | Se houver múltiplas notificações: stagger 100ms entre entradas, empilhadas verticalmente com `gap: 8px` |

### 3.8 Badge de Status (Dossiê e Escrow)

| Propriedade | Valor |
|---|---|
| **Mudança de status** | Cross-fade: badge atual faz fade-out 150ms enquanto novo badge faz fade-in 150ms simultaneamente |
| **Aparece pela primeira vez** | Fade-in 150ms |

### 3.9 Campo de Mensagem — Estado Desabilitado (Rate Limit)

| Propriedade | Valor |
|---|---|
| **Transição para desabilitado** | `opacity: 1 → 0.5` em 250ms ease-in-out |
| **Contador regressivo** | Aparece com fade-in 250ms quando campo é desabilitado |
| **Transição para habilitado** | `opacity: 0.5 → 1` em 250ms ease-in-out + pequeno pulse: `scale: 1 → 1.02 → 1` em 300ms |

### 3.10 Scroll Automático para Última Mensagem

| Propriedade | Valor |
|---|---|
| **Trigger** | Nova mensagem da Dani ou do Cedente adicionada |
| **Comportamento** | `scrollIntoView({ behavior: 'smooth', block: 'end' })` |
| **Exceção** | Se o Cedente scrollou manualmente para cima (lendo histórico), o scroll automático é suspendido. Um botão "Nova mensagem ↓" aparece no canto inferior — ao clicar, retoma o scroll automático. |
| **Botão "Nova mensagem"** | Slide-in da direita: `x: 20 → 0` + `opacity: 0 → 1`, 250ms |

---

## 4. Mobile — React Native Reanimated

Para o chat da Dani acessado via aplicativo mobile (se implementado via React Native + Expo), as animações são adaptadas para o ambiente mobile.

### 4.1 Equivalências Web → Mobile

| Animação Web (Framer Motion) | Animação Mobile (Reanimated) |
|---|---|
| Bolha da Dani (spring) | `withSpring(0, { stiffness: 300, damping: 30 })` para translateX |
| Sugestões (fade + slide) | `withTiming(1, { duration: 250, easing: Easing.out(Easing.cubic) })` para opacity + translateY |
| Modal (scale + fade) | `withTiming` + `withSpring` combinados |
| Typing indicator (bounce) | `withRepeat(withSequence(withTiming(-4), withTiming(0)), -1)` |

### 4.2 Regras Específicas Mobile

- Usar **UI thread** para todas as animações via `useAnimatedStyle` — nunca JS thread para animações de UI.
- Gestos de swipe para dismissar notificações: `react-native-gesture-handler` + `withSpring`.
- Haptic feedback em ações críticas (aceitar proposta, enviar contraproposta): `expo-haptics` com `ImpactFeedbackStyle.Medium`.

---

## 5. Redução de Movimento (prefers-reduced-motion)

> **🔴 Obrigatório:** Toda animação deve respeitar `prefers-reduced-motion: reduce`.

### 5.1 Implementação

```typescript
// src/hooks/use-reduced-motion.ts
import { useReducedMotion } from 'framer-motion';

export function useDaniMotion() {
  const shouldReduceMotion = useReducedMotion();

  return {
    // Se reduced motion, apenas fade simples de 150ms
    messageTransition: shouldReduceMotion
      ? { opacity: [0, 1], transition: { duration: 0.15 } }
      : undefined, // usa a variante spring padrão

    // Desabilita bounce do typing indicator
    typingBounce: shouldReduceMotion ? { y: 0 } : undefined,

    // Modal apenas fade
    modalScale: shouldReduceMotion ? { scale: 1 } : undefined,
  };
}
```

### 5.2 Fallbacks por Componente

| Componente | Animação padrão | Com reduced-motion |
|---|---|---|
| Bolha de mensagem | Spring slide + scale | Fade simples 150ms |
| Typing indicator | Bounce 1.4s loop | Fade pulsante: `opacity: 0.4 → 1`, 800ms loop |
| Sugestões | Fade + slide stagger | Fade simples 150ms (simultâneo, sem stagger) |
| Modal | Scale + fade | Fade simples 250ms |
| Card de análise | ScaleY expand | Fade simples 250ms |
| Notificação | Slide da direita | Fade simples 150ms |

---

## 6. Checklist de Implementação

Antes de mergear qualquer componente animado do chat, verificar:

- [ ] Animação usa exclusivamente `transform` e `opacity` (sem reflow)
- [ ] `prefers-reduced-motion` implementado com fallback
- [ ] `AnimatePresence` envolvendo componentes com animação de saída
- [ ] Duração respeita os tokens (`motion-fast`, `motion-normal`, `motion-slow`)
- [ ] Easing corresponde ao tipo de transição (entrada: ease-out, saída: ease-in)
- [ ] Stagger aplicado quando há múltiplos itens aparecendo (max: 50ms por item)
- [ ] Performance validada no DevTools (sem janks, sem layout thrashing)
- [ ] Mobile: animações via UI thread (useAnimatedStyle)
- [ ] Haptic feedback em ações críticas (mobile)

---

## 7. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — Motion Spec do chat AI-Dani-Cedente. Tokens de duração, curvas de easing, variantes Framer Motion, equivalências mobile (Reanimated), redução de movimento. |
