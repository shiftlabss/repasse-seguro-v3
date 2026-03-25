# 04 - Motion Spec

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Frontend engineers e designers | Especificação de motion e animações para o agente AI-Dani-Cessionário — Framer Motion na web | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - Stack de motion: **Framer Motion ≥ 11.x** na web. Tokens de duração e easing padronizados.
> - Princípio central: **motion funcional** — animações comunicam estado, não decoram.
> - Todos os tokens derivam do Brand Guide (D03) e das regras de negócio (D01).
> - Animações de acessibilidade: `prefers-reduced-motion` obrigatório em toda animação.
> - **6 componentes animados obrigatórios:** FAB, Chat Window, Bubble de mensagem, Typing indicator, Rate limit (reativação), Badge do FAB.
> - Mobile (Fase 2 — WhatsApp): sem motion — canal textual puro via EvolutionAPI.

---

## 1. Princípios de Motion

| Princípio | Definição |
|---|---|
| **Funcional** | Toda animação comunica um estado ou uma transição de dados. Animação sem função comunicativa é removida |
| **Instantâneo ou perceptível** | Duração ≤ 200ms para micro-interações. Duração ≤ 350ms para transições de componente. Nunca mais lento |
| **Consistente** | Mesmos tokens de easing e duração para o mesmo tipo de evento em todo o agente |
| **Acessível** | `prefers-reduced-motion: reduce` desativa ou substitui toda animação por fade opacity simples (100ms) |
| **Non-blocking** | Animações nunca bloqueiam interação do usuário. Layout não muda durante animação |

---

## 2. Tokens de Motion

### 2.1 Duração

| Token | Valor | Uso |
|---|---|---|
| `--motion-instant` | `100ms` | Hover states, focus rings |
| `--motion-micro` | `200ms` | Micro-interações: badge, botão, chip |
| `--motion-component` | `250ms` | Transições de componente: chat window open/close |
| `--motion-content` | `300ms` | Aparecimento de conteúdo: mensagem de boas-vindas, fade-in de cards |
| `--motion-feedback` | `500ms` | Feedback de estado: reativação do campo após rate limit |

### 2.2 Easing

| Token | Valor CSS | Uso |
|---|---|---|
| `--ease-out` | `cubic-bezier(0, 0, 0.2, 1)` | Entradas — elementos aparecem com desaceleração |
| `--ease-in` | `cubic-bezier(0.4, 0, 1, 1)` | Saídas — elementos desaparecem com aceleração |
| `--ease-in-out` | `cubic-bezier(0.4, 0, 0.2, 1)` | Transições simétricas |
| `--ease-spring` | `spring(stiffness: 300, damping: 25)` | Badge do FAB, bounce |

### 2.3 CSS Custom Properties (Framer Motion via variáveis)

```css
/* Motion Tokens — AI-Dani-Cessionário */
:root {
  --motion-instant:    100ms;
  --motion-micro:      200ms;
  --motion-component:  250ms;
  --motion-content:    300ms;
  --motion-feedback:   500ms;
  --ease-out:          cubic-bezier(0, 0, 0.2, 1);
  --ease-in:           cubic-bezier(0.4, 0, 1, 1);
  --ease-in-out:       cubic-bezier(0.4, 0, 0.2, 1);
}

@media (prefers-reduced-motion: reduce) {
  :root {
    --motion-instant:    0ms;
    --motion-micro:      0ms;
    --motion-component:  100ms;
    --motion-content:    100ms;
    --motion-feedback:   0ms;
  }
}
```

---

## 3. Catálogo de Animações por Componente

### 3.1 FAB — Floating Action Button

**Evento:** Montagem inicial (page load)

```typescript
// Framer Motion
const fabVariants = {
  hidden: { scale: 0, opacity: 0 },
  visible: {
    scale: 1,
    opacity: 1,
    transition: { duration: 0.2, ease: [0, 0, 0.2, 1] }
  }
}
```

| Propriedade | Valor |
|---|---|
| Duração | 200ms (`--motion-micro`) |
| Easing | `--ease-out` |
| Propriedades animadas | `scale` (0 → 1), `opacity` (0 → 1) |
| Trigger | `AnimatePresence` ao montar o componente |
| Reduced motion | `opacity` 0→1, 100ms |

---

**Evento:** Badge de alerta (novo alerta chega)

```typescript
const badgeVariants = {
  initial: { scale: 0.5, opacity: 0 },
  animate: {
    scale: [0.5, 1.2, 1],
    opacity: 1,
    transition: {
      duration: 0.2,
      times: [0, 0.6, 1],
      type: 'spring',
      stiffness: 300,
      damping: 25
    }
  }
}
```

| Propriedade | Valor |
|---|---|
| Duração | 200ms (`--motion-micro`) |
| Easing | Spring (stiffness: 300, damping: 25) |
| Propriedades animadas | `scale` (0.5 → 1.2 → 1), `opacity` (0 → 1) |
| Trigger | Badge count aumenta |
| Reduced motion | Aparecer instantaneamente |

---

### 3.2 Chat Window — Abertura

```typescript
const chatWindowVariants = {
  hidden: {
    opacity: 0,
    y: 20,
    scale: 0.98
  },
  visible: {
    opacity: 1,
    y: 0,
    scale: 1,
    transition: { duration: 0.25, ease: [0, 0, 0.2, 1] }
  },
  exit: {
    opacity: 0,
    y: 16,
    scale: 0.98,
    transition: { duration: 0.2, ease: [0.4, 0, 1, 1] }
  }
}
```

| Evento | Duração | Easing | Propriedades |
|---|---|---|---|
| Abertura | 250ms (`--motion-component`) | `--ease-out` | `opacity` (0→1), `y` (20px→0), `scale` (0.98→1) |
| Fechamento | 200ms (`--motion-micro`) | `--ease-in` | `opacity` (1→0), `y` (0→16px), `scale` (1→0.98) |
| Reduced motion | 100ms | — | Apenas `opacity` |

---

### 3.3 Mensagem de Boas-Vindas (RN-DC-005)

```typescript
const welcomeVariants = {
  hidden: { opacity: 0, y: 8 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.3, ease: [0, 0, 0.2, 1], delay: 0.1 }
  }
}
```

| Propriedade | Valor |
|---|---|
| Duração | 300ms (`--motion-content`) |
| Delay | 100ms após chat window estar visível |
| Easing | `--ease-out` |
| Propriedades animadas | `opacity` (0→1), `y` (8px→0) |
| Reduced motion | `opacity` 0→1, 100ms, sem delay |

---

### 3.4 Bubble de Mensagem — Entrada

Toda mensagem nova entra com animação de deslizamento.

```typescript
const bubbleVariants = {
  hidden: { opacity: 0, y: 12, scale: 0.97 },
  visible: {
    opacity: 1,
    y: 0,
    scale: 1,
    transition: { duration: 0.2, ease: [0, 0, 0.2, 1] }
  }
}
```

| Tipo de mensagem | Origin X | Origin Y | Duração |
|---|---|---|---|
| Mensagem da Dani | `left` | `bottom` | 200ms |
| Mensagem do Cessionário | `right` | `bottom` | 200ms |
| Mensagem de sistema | `center` | `bottom` | 150ms |

---

### 3.5 Typing Indicator (RN-DC-029)

Três pontos pulsando enquanto a Dani processa.

```typescript
// Animação de cada ponto individualmente, com stagger
const dotVariants = {
  animate: {
    y: [0, -6, 0],
    transition: {
      duration: 0.6,
      repeat: Infinity,
      ease: 'easeInOut',
      delay: index * 0.15 // stagger de 150ms entre pontos
    }
  }
}
```

| Propriedade | Valor |
|---|---|
| Loop | Infinito enquanto typing indicator visível |
| Stagger | 150ms entre cada ponto |
| Duração por ciclo | 600ms |
| Amplitude | 6px (para cima) |
| Easing | `ease-in-out` |
| Reduced motion | Substituir por `opacity` pulsando (0.3→1→0.3, 1s loop) sem translação |
| Visibilidade | Aparece após SLA ser atingido; desaparece quando resposta chega |

---

### 3.6 Reativação do Campo de Chat (RN-DC-025)

Quando o rate limit expira e o campo de chat é reativado.

```typescript
const inputReactivateVariants = {
  pulse: {
    boxShadow: [
      '0 0 0 0px rgba(0, 105, 168, 0)',
      '0 0 0 4px rgba(0, 105, 168, 0.2)',
      '0 0 0 0px rgba(0, 105, 168, 0)'
    ],
    transition: { duration: 0.5, ease: 'easeInOut' }
  }
}
```

| Propriedade | Valor |
|---|---|
| Duração | 500ms (`--motion-feedback`) |
| Easing | `ease-in-out` |
| Propriedade animada | `box-shadow` (pulse sutil com cor primary) |
| Executa | Uma vez, ao reativar |
| Reduced motion | Sem animação — apenas reativar visualmente |

---

### 3.7 Cards de Oportunidade — Entrada (lista inline no chat)

Quando a Dani retorna múltiplos cards de oportunidade.

```typescript
// Stagger container
const containerVariants = {
  hidden: {},
  visible: {
    transition: { staggerChildren: 0.08 }
  }
}

// Card individual
const cardVariants = {
  hidden: { opacity: 0, y: 10 },
  visible: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.2, ease: [0, 0, 0.2, 1] }
  }
}
```

| Propriedade | Valor |
|---|---|
| Stagger | 80ms entre cards |
| Duração por card | 200ms |
| Reduced motion | Todos aparecem simultaneamente, `opacity` 0→1, 100ms |

---

### 3.8 Contador Regressivo de Rate Limit

Sem animação de transição — o número atualiza a cada segundo de forma direta (sem tweening). Apenas o estado final de reativação usa animation (seção 3.6).

---

## 4. Regras de Acessibilidade

| Regra | Implementação |
|---|---|
| `prefers-reduced-motion: reduce` | Todo componente Framer Motion usa o hook `useReducedMotion()`. Quando `true`, substituir por fade opacity simples (100ms) ou sem animação |
| Nenhuma animação bloqueia input | `pointer-events: auto` mantido durante toda animação |
| Typing indicator com ARIA | `role="status"` + `aria-label="Dani está digitando"`. Remover do DOM quando resposta chegar |
| Animações de loop | Pausar quando componente sair do viewport (`useInView`) |

---

## 5. Padrão de Implementação Framer Motion

### 5.1 Hook obrigatório

```typescript
import { useReducedMotion } from 'framer-motion'

function DaniChatWindow() {
  const shouldReduceMotion = useReducedMotion()

  const variants = shouldReduceMotion
    ? { hidden: { opacity: 0 }, visible: { opacity: 1, transition: { duration: 0.1 } } }
    : chatWindowVariants

  return (
    <AnimatePresence>
      {isOpen && (
        <motion.div
          variants={variants}
          initial="hidden"
          animate="visible"
          exit="exit"
        >
          {/* ... */}
        </motion.div>
      )}
    </AnimatePresence>
  )
}
```

### 5.2 Regras de implementação

- Usar `AnimatePresence` para todo componente que monta/desmonta (FAB, chat window, bubbles, typing indicator).
- Usar `layoutId` para shared element transitions entre FAB e chat window (expansão).
- Nunca usar `transition: all` via CSS — usar exclusivamente Framer Motion para componentes animados.
- Variantes centralizadas em `src/agente/motion/variants.ts` — nunca inline no componente.
- Nomenclatura de variantes: `{componenteVariants}` (ex: `fabVariants`, `chatWindowVariants`).

---

## 6. Inventário de Animações (Resumo)

| Componente | Evento | Duração | Propriedades | Reduzido |
|---|---|---|---|---|
| FAB | Montagem | 200ms | scale, opacity | opacity |
| FAB badge | Novo alerta | 200ms spring | scale bounce, opacity | instantâneo |
| Chat window | Abertura | 250ms | opacity, y, scale | opacity 100ms |
| Chat window | Fechamento | 200ms | opacity, y, scale | opacity 100ms |
| Boas-vindas | Montagem | 300ms + 100ms delay | opacity, y | opacity 100ms |
| Bubble de mensagem | Entrada | 200ms | opacity, y, scale | opacity |
| Typing indicator | Loop | 600ms/ciclo | y (bounce 3 pontos) | opacity pulse |
| Reativação input | Rate limit expira | 500ms | box-shadow pulse | nenhuma |
| Cards de oportunidade | Entrada stagger | 200ms + 80ms stagger | opacity, y | opacity simultâneo |

---

## Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. Motion tokens, catálogo de animações por componente, regras de acessibilidade, padrões de implementação Framer Motion. |
