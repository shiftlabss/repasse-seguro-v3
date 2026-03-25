# Motion Spec — Módulo CRM

## Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Designers e frontend engineers |
| **Escopo** | Especificações de animação e transição do CRM — interface interna profissional |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 2026-03-23 (America/Fortaleza) |
| **Fonte primária** | ShiftLabs Stacks v7.0 seções 2.4 e 3.6 · Motion Spec Unificado Admin v2.0 |
| **Fonte secundária** | Brand Guide CRM v1.0 (Doc 03) · Regras de Negócio CRM v1.0 |
| **Caráter** | **Normativo.** Desvios exigem ADR aprovado. |

---

> **TL;DR**
>
> - **Filosofia:** Funcional first. O CRM é ferramenta de trabalho — animações existem para orientar, não entreter.
> - **Biblioteca:** Framer Motion 12+ obrigatório. CSS transitions para micro-interações leves (hover, focus).
> - **Spinners:** PROIBIDOS em nível de página. Skeleton screens com shimmer em todo loading state.
> - **Spinners em botões:** Permitidos exclusivamente para feedback de submissão de formulário.
> - **Duração máxima:** 300ms para qualquer transição de UI. Exceção: highlight de status em tempo real (500ms).
> - **Acessibilidade:** `prefers-reduced-motion` obrigatório em toda animação. Com `reduced-motion`, todas as animações são substituídas por `opacity` 0→1 em 100ms.
> - **Regra de ouro:** Se remover a animação não prejudica a compreensão da interface, ela não deveria existir.

---

## 1. Filosofia

O CRM é usado por Analistas RS em ciclos de trabalho intensos (múltiplos Casos simultâneos, SLAs monitorados, decisões operacionais rápidas). Animações devem:

| Princípio | Descrição |
|---|---|
| **Orientar, não distrair** | Animações comunicam mudança de estado, não criam espetáculo |
| **Imperceptíveis quando corretas** | O usuário deve sentir fluidez, nunca esperar por animações |
| **Informativas em contexto crítico** | Highlight de SLA vencido deve chamar atenção — é uma exceção deliberada |
| **Consistentes** | Mesmo padrão de entrada/saída em todos os modais, drawers e toasts |
| **Sem loop** | Nenhuma animação em loop além do skeleton shimmer e loading de botão |

---

## 2. Tokens de Duração

Herdados do ShiftLabs Stacks v7.0, seção 2.4.3. O CRM usa apenas os tokens relevantes para ferramenta profissional — sem tokens de `slower` (700ms) e `crawl` (1000ms) que são exclusivos de páginas de marketing consumer.

### 2.1 Variáveis CSS

```css
@theme {
  --duration-instant: 100ms;   /* hover, focus ring, toggle, estado ativo de botão */
  --duration-fast:    150ms;   /* dropdown, tooltip, popover, badge update */
  --duration-normal:  200ms;   /* modal open/close, drawer, accordion */
  --duration-slow:    300ms;   /* kanban drag settle, skeleton shimmer cycle */
  --duration-status:  500ms;   /* highlight de status em tempo real (âmbar → neutro) */
}
```

### 2.2 Constantes TypeScript — Framer Motion

```typescript
// apps/web-crm/src/lib/motion-tokens.ts
export const duration = {
  instant: 0.1,   // hover, focus
  fast:    0.15,  // dropdown, tooltip
  normal:  0.2,   // modal, drawer
  slow:    0.3,   // drag-and-drop, skeleton
  status:  0.5,   // highlight de status em tempo real
} as const;

export const ease = {
  default:   [0.4, 0, 0.2, 1],  // material design standard — entrada e saída de elementos
  enter:     [0, 0, 0.2, 1],    // decelerate — elementos que entram
  exit:      [0.4, 0, 1, 1],    // accelerate — elementos que saem
  spring:    { type: 'spring', stiffness: 400, damping: 30 },  // drag-and-drop
} as const;
```

---

## 3. Transições de Página e Rota

### 3.1 Navegação entre rotas

Duração: `fast` (150ms). Tipo: `opacity` fade.

Não usar slide ou scale nas transições de rota — o CRM tem layouts densos onde movimento lateral causa desorientação.

```typescript
// components/layout/page-transition.tsx
const pageVariants = {
  initial: { opacity: 0 },
  animate: { opacity: 1, transition: { duration: duration.fast, ease: ease.enter } },
  exit:    { opacity: 0, transition: { duration: duration.fast, ease: ease.exit  } },
};
```

### 3.2 Troca de aba (detalhe do Caso)

O detalhe do Caso tem abas horizontais (Visão Geral, Dossiê, Proposta/Negociação, Comissão, Comunicação, Histórico). A troca de aba não anima o conteúdo — apenas muda o indicador ativo com `duration-fast`.

```css
/* Tab indicator — borda inferior */
.tab-indicator {
  transition: left var(--duration-fast) var(--ease-default),
              width var(--duration-fast) var(--ease-default);
}
```

---

## 4. Modais e Drawers

### 4.1 Modal

Entrada: `scale(0.97) + opacity(0)` → `scale(1) + opacity(1)`. Duração: `normal` (200ms).

Saída: inverso, duração: `fast` (150ms). Saída é sempre mais rápida que entrada.

```typescript
const modalVariants = {
  initial: { opacity: 0, scale: 0.97 },
  animate: {
    opacity: 1, scale: 1,
    transition: { duration: duration.normal, ease: ease.enter },
  },
  exit: {
    opacity: 0, scale: 0.97,
    transition: { duration: duration.fast, ease: ease.exit },
  },
};

// Overlay (backdrop)
const overlayVariants = {
  initial: { opacity: 0 },
  animate: { opacity: 1, transition: { duration: duration.fast } },
  exit:    { opacity: 0, transition: { duration: duration.fast } },
};
```

**Modais do CRM que usam este padrão:**
- Modal de Novo Caso
- Modal de Avanço de Estado (checklist de condições de saída)
- Modal de Cancelamento de Caso (seleção de motivo)
- Modal de Redistribuição de Caso
- Modal de Retrocesso de Estado (campo de justificativa)
- Modal de Confirmação de Desligamento de Usuário
- Modal de Registro de Fechamento (critérios cumulativos)
- Modal de Cancelamento de Comissão (quatro olhos)

### 4.2 Drawer (painel lateral)

Usado para: perfil detalhado de Contato, histórico de versões de documento do Dossiê.

Entrada: `x(320px) + opacity(0)` → `x(0) + opacity(1)`. Duração: `normal` (200ms).

```typescript
const drawerVariants = {
  initial: { x: 320, opacity: 0 },
  animate: {
    x: 0, opacity: 1,
    transition: { duration: duration.normal, ease: ease.enter },
  },
  exit: {
    x: 320, opacity: 0,
    transition: { duration: duration.fast, ease: ease.exit },
  },
};
```

---

## 5. Kanban — Pipeline de Casos

### 5.1 Drag-and-drop de cards

O kanban do Pipeline (RN-039) é o componente de maior interatividade do CRM. As animações devem dar feedback físico claro sem latência perceptível.

| Evento | Animação | Duração |
|---|---|---|
| **Hover no card** | Sombra: `shadow-sm` → `shadow-md` | `instant` (100ms) |
| **Início do drag** | Scale `1.02`, sombra `shadow-lg`, `ring-2 ring-primary-300`, `cursor-grabbing` | `fast` (150ms) |
| **Durante drag** | Card flutuante segue o cursor com offset visual. Slot de destino: fundo `primary-50`, borda `dashed border-primary-300` | — |
| **Drop bem-sucedido** | Scale `1.02` → `1.0`, sombra reduz para `shadow-sm` | `slow` via spring (300ms) |
| **Drop inválido (estado proibido)** | Shake horizontal: `x: [0, -4, 4, -4, 4, 0]` em `slow` (300ms), cor da borda `danger-500` | 300ms |

```typescript
// Física do drop
const dropSettle = {
  type: 'spring',
  stiffness: 400,
  damping: 30,
};

// Shake de drop inválido
const invalidDropShake = {
  x: [0, -4, 4, -4, 4, 0],
  transition: { duration: 0.3 },
};
```

### 5.2 Reordenação de lista (modo lista)

Quando o usuário ordena a lista de Casos, os itens deslizam para a nova posição com `layout` do Framer Motion.

```typescript
<motion.tr layout transition={{ duration: duration.normal, ease: ease.default }}>
```

---

## 6. Atualizações em Tempo Real

### 6.1 Highlight de atualização de status

Quando um Caso é atualizado em tempo real via Supabase Realtime (RN-183, RN-186), o card ou linha recebe highlight visual para indicar a mudança.

Animação: fundo do card/linha vai de `warning-100` (âmbar claro) para `transparent` (neutro) em 500ms.

```typescript
// Hook de highlight
const useStatusHighlight = () => {
  const [highlighted, setHighlighted] = useState(false);

  const triggerHighlight = () => {
    setHighlighted(true);
    setTimeout(() => setHighlighted(false), 500);
  };

  return { highlighted, triggerHighlight };
};

// Variante de animação
const highlightVariants = {
  initial:     { backgroundColor: 'transparent' },
  highlighted: { backgroundColor: 'var(--color-warning-100)' },
  fadeOut:     {
    backgroundColor: 'transparent',
    transition: { duration: duration.status, ease: ease.default },
  },
};
```

### 6.2 Badge de alerta de SLA (nova aparição)

Quando um badge de SLA muda de `on_track` → `at_risk` → `overdue`, ele pulsa uma vez para chamar atenção:

```typescript
const slaAlertAppear = {
  scale: [1, 1.15, 1],
  transition: { duration: 0.3, ease: ease.default },
};
```

Não há animação contínua de pulsação — o badge se estabelece no novo estado sem loop.

### 6.3 Contador de notificações (badge no sino)

Quando uma nova notificação chega, o badge do sino (número de notificações não lidas) faz scale `1` → `1.3` → `1` em `fast` (150ms).

---

## 7. Loading States

### 7.1 Skeleton screens (obrigatório)

**Spinners globais de página são proibidos.** Todo estado de carregamento usa skeleton.

```typescript
// components/ui/skeleton.tsx
// Implementação via shadcn/ui + animação CSS personalizada
```

| Componente | Skeleton |
|---|---|
| Card kanban | `260px × 120px` retângulo com shimmer |
| Linha de tabela | 4 retângulos de larguras variadas (`w-16`, `w-32`, `w-48`, `w-20`) |
| Métricas do Dashboard | Card com retângulo grande + retângulo pequeno |
| Linha do tempo | Círculo 8px + retângulo, repetido 5× |
| Header do Caso | Retângulos para ID, nome, badges |

Animação shimmer: ciclo de 1.5s, `ease-in-out infinite` (conforme Brand Guide Doc 03).

### 7.2 Spinner em botão (único caso permitido)

Exclusivamente durante submissão de formulário — enquanto a requisição está em andamento.

Tamanho: 14px. Cor: herdada do texto do botão. Não substituir o texto — exibir spinner à esquerda do texto com `gap-2`.

```typescript
<Button disabled={isSubmitting}>
  {isSubmitting && <Spinner size={14} className="animate-spin" />}
  {isSubmitting ? 'Salvando...' : 'Registrar Atividade'}
</Button>
```

---

## 8. Formulários e Inputs

### 8.1 Feedback de validação

| Evento | Animação |
|---|---|
| Campo com erro (ao sair do foco) | Border `neutral-300` → `danger-500` em `instant` (100ms) |
| Mensagem de erro aparece | `opacity(0) + y(4px)` → `opacity(1) + y(0)` em `fast` (150ms) |
| Campo correto após erro | Border `danger-500` → `neutral-300`, mensagem some em `fast` |
| Focus em input | `ring-2 ring-primary-200` em `instant` (100ms) via CSS transition |

### 8.2 Checklist de condições de saída (Modal de Avanço de Estado)

Quando o Analista RS marca uma condição como atendida, o ícone muda de `Circle` (cinza) para `CheckCircle` (verde):

```typescript
const checkmarkVariants = {
  unchecked: { scale: 1, color: 'var(--color-neutral-400)' },
  checked: {
    scale: [1, 1.2, 1],
    color: 'var(--color-success-600)',
    transition: { duration: duration.fast },
  },
};
```

---

## 9. Notificações e Toasts

### 9.1 Toast (usando Sonner via shadcn/ui)

Entrada: `y(-16px) + opacity(0)` → `y(0) + opacity(1)`. Duração: `fast` (150ms). Posição: canto superior direito.

Saída (auto-dismiss após 4s): `opacity(1)` → `opacity(0)`. Duração: `fast` (150ms).

Saída por swipe: `x(120px)` em `fast`.

### 9.2 Notificação push (sino)

Painel de notificações: entrada via `y(-8px) + opacity(0)` → `y(0) + opacity(1)` com stagger de 50ms entre itens.

---

## 10. Linha do Tempo do Caso

### 10.1 Carregamento de novos eventos

Quando um novo evento é adicionado à linha do tempo (via Realtime), ele aparece no topo com:

```typescript
const timelineItemVariants = {
  initial: { opacity: 0, y: -12, height: 0 },
  animate: {
    opacity: 1, y: 0, height: 'auto',
    transition: { duration: duration.normal, ease: ease.enter },
  },
};
```

### 10.2 Scroll suave

A linha do tempo usa `scroll-behavior: smooth` para rolagem programática até novos eventos.

---

## 11. Acessibilidade — `prefers-reduced-motion`

**Obrigatório em toda animação implementada com Framer Motion.**

```typescript
// hooks/use-reduced-motion.ts
import { useReducedMotion } from 'framer-motion';

export const useCrmMotion = () => {
  const shouldReduceMotion = useReducedMotion();

  return {
    // Se reduced-motion ativo: apenas opacity fade em 100ms
    pageTransition: shouldReduceMotion
      ? { opacity: [0, 1], transition: { duration: 0.1 } }
      : pageVariants,

    modalTransition: shouldReduceMotion
      ? { opacity: [0, 1], transition: { duration: 0.1 } }
      : modalVariants,

    // Highlight de status: apenas mudança de cor, sem scale
    statusHighlight: shouldReduceMotion
      ? { backgroundColor: ['var(--color-warning-100)', 'transparent'] }
      : highlightVariants,
  };
};
```

Com `prefers-reduced-motion: reduce`:
- Todas as animações são substituídas por `opacity` 0→1 em 100ms
- Drag-and-drop: sem animação de levitação, sem scale
- Checklist: mudança direta de cor sem scale
- Skeleton: animação shimmer desabilitada (fundo estático `neutral-200`)

---

## 12. Animações Proibidas no CRM

As seguintes animações estão **explicitamente proibidas** por conflitarem com a filosofia de ferramenta de trabalho:

| Proibição | Motivo |
|---|---|
| Spinner global de página | Substitua por skeleton screen sempre |
| Animações em loop decorativas | Distraem em ambiente de trabalho |
| Parallax e scroll reveals | Exclusivos de páginas consumer/marketing |
| Confetti ou animações de celebração | Inadequado para contexto profissional |
| Transições de rota com slide/flip | Causam desorientação em layouts densos |
| Hover scale em cards de tabela | Conflita com densidade de informação |
| Animação de entrada stagger em listas longas | Atraso perceptível em listas com +20 itens |
| Bounce excessivo em spring | Use `damping: 30` mínimo para comportamento sóbrio |
| `duration > 500ms` em qualquer transição de UI | Exceto skeleton shimmer (ciclo de 1.5s) |

---

## 13. Implementação por Componente — Referência Rápida

| Componente | Duração | Tipo | Notas |
|---|---|---|---|
| Fade de rota | 150ms | opacity | Framer Motion layout |
| Modal open | 200ms | scale 0.97→1 + opacity | ease: enter |
| Modal close | 150ms | scale 1→0.97 + opacity | ease: exit |
| Drawer open | 200ms | x 320→0 + opacity | ease: enter |
| Dropdown open | 150ms | opacity + y 4→0 | CSS transition |
| Tooltip | 100ms | opacity | CSS transition |
| Badge de alerta SLA | 300ms | scale pulse 1→1.15→1 | Framer Motion |
| Highlight de atualização | 500ms | background âmbar→neutro | Framer Motion |
| Kanban drag lift | 150ms | scale 1→1.02 + shadow | Framer Motion |
| Kanban drop settle | spring | scale 1.02→1.0 | spring stiffness 400 |
| Kanban drop inválido | 300ms | shake x [-4,4,-4,4,0] | Framer Motion |
| Checkbox/checklist | 150ms | scale 1→1.2→1 + color | Framer Motion |
| Toast enter | 150ms | y -16→0 + opacity | Sonner |
| Toast exit | 150ms | opacity 1→0 | Sonner |
| Error message | 150ms | y 4→0 + opacity | Framer Motion |
| Notificação badge | 150ms | scale 1→1.3→1 | Framer Motion |
| Timeline item novo | 200ms | y -12→0 + height auto | Framer Motion |
| Skeleton shimmer | 1500ms | background gradient loop | CSS animation |
| Spinner de botão | contínuo | rotate 360° | CSS animation |
