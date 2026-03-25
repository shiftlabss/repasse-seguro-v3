# Repasse AI
## 07 — Wireframes

| Campo | Valor |
|---|---|
| **Documento** | 07 - Wireframes |
| **Produto** | Repasse AI |
| **Versão** | v1.0 |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |
| **Autor** | Claude Code Desktop — Pipeline v6.1 |
| **Status** | Ativo |
| **Fase** | 2 — Produto |
| **Área** | Design + Frontend |
| **Referências** | 01 - Regras de Negócio · 03 - Brand Theme Guide · 05 - PRD · 06 - Mapa de Telas |

---

> 📌 **TL;DR**
>
> - **22 wireframes gerados** — cobertura completa de T-001 a T-022.
> - **4 módulos cobertos:** A (Webchat), B (Notificações Proativas), C (Configurações do Usuário), D (Admin Supervisão IA).
> - **Design tokens aplicados:** shadcn/ui tokens do Brand Theme Guide v1.0 — paleta neutra com acento `#0069A8`, Inter Variable, radius base 14px.
> - **Padrão de layout:** FAB flutuante para Cessionário (Webchat overlay); sidebar fixa + grid 12 col para Admin; modais centrados; full-screen no mobile.
> - **Estados obrigatórios:** Loading (skeleton retangular), Empty, Error, Populated — implementados em todas as 22 telas.
> - **Telas pendentes:** 0 — cobertura 100%.
> - **Nota arquitetural:** Repasse AI é serviço backend puro (PG-03). As telas são superfícies no frontend do Cessionário e do Admin que consomem a API do Repasse AI.

---

## 1. Persona

**Product Designer / UX Designer / Frontend Lead** — consome este documento para traduzir o Mapa de Telas em implementação. Um desenvolvedor frontend implementa layout e componentes sem precisar de reunião. Um QA valida estados e responsividade com base nos wireframes. Um UX Writer extrai empty states, labels e CTAs para o documento D08.

**Anti-exemplos que invalidam este documento:**
- ❌ Wireframe sem estados obrigatórios (Loading/Empty/Error/Populated).
- ❌ Tela sem role RBAC mínimo definido.
- ❌ Layout que ignora o padrão de layout de cada tipo de tela.
- ❌ Tela do Mapa de Telas sem wireframe correspondente.
- ❌ Wireframe sem RFs relacionados.
- ❌ Spinner como estado de loading — proibido; usar skeleton.

**Dependências:** 01 - Regras de Negócio · 03 - Brand Theme Guide · 05 - PRD · 06 - Mapa de Telas.

---

## 2. Design Tokens de Referência

Tokens extraídos do Brand Theme Guide v1.0 — shadcn/ui com CSS custom properties.

### 2.1 Superfícies e Background

| Token | Light | Dark | Uso |
|---|---|---|---|
| `--background` | `#FFFFFF` | `#0A0A0A` | Fundo principal |
| `--card` | `#FFFFFF` | `#171717` | Fundo de cards e bolhas |
| `--muted` | `#F5F5F5` | `#262626` | Backgrounds inativos |
| `--popover` | `#FFFFFF` | `#171717` | Dropdowns, modais |

### 2.2 Cores Primárias e Semânticas

| Token | Light | Dark | Uso |
|---|---|---|---|
| `--primary` | `#0069A8` | `#00598A` | CTAs, botões primários, FAB |
| `--primary-foreground` | `#F0F9FF` | `#F0F9FF` | Texto sobre primary |
| `--destructive` | `#E7000B` | `#FF6467` | Erros, ações destrutivas, takeover banner |
| `--foreground` | `#0A0A0A` | `#FAFAFA` | Texto principal |
| `--muted-foreground` | `#737373` | `#A1A1A1` | Timestamps, labels secundários |
| `--secondary` | `#F4F4F5` | `#27272A` | Botões secundários |
| `--border` | `#E5E5E5` | `#FFFFFF1A` | Bordas, divisores |
| `--ring` | `#A1A1A1` | `#737373` | Focus ring |

### 2.3 Sidebar

| Token | Light | Dark | Uso |
|---|---|---|---|
| `--sidebar` | `#FAFAFA` | `#171717` | Fundo da sidebar |
| `--sidebar-primary` | `#0084D1` | `#00A6F4` | Item ativo da sidebar |
| `--sidebar-accent` | `#F5F5F5` | `#262626` | Hover de items |
| `--sidebar-border` | `#E5E5E5` | `#FFFFFF1A` | Bordas da sidebar |

### 2.4 Charts

| Token | HEX | Uso |
|---|---|---|
| `--chart-1` | `#74D4FF` | Série mais clara / score baixo |
| `--chart-2` | `#00A6F4` | Série 2 |
| `--chart-3` | `#0084D1` | Série 3 / chart accent |
| `--chart-4` | `#0069A8` | Série 4 / score médio |
| `--chart-5` | `#00598A` | Série mais escura / score alto |

### 2.5 Grid e Sidebar Layout

| Dimensão | Valor | Nota |
|---|---|---|
| Colunas | 12 col | Grid 12 colunas — padrão Tailwind |
| Gap | 24px (gap-6) | Espaçamento entre colunas |
| Max-width | 1280px (max-w-screen-xl) | Container principal |
| Sidebar expanded | 256px (w-64) | Desktop |
| Sidebar collapsed | 48px (w-12) | Tablet — somente ícones |
| Sidebar mobile | Drawer full-height | Mobile — overlay |

### 2.6 Border Radius

| Token | Valor | Uso |
|---|---|---|
| `--radius-sm` | 8.4px | Badges, chips |
| `--radius-md` | 11.2px | Botões, inputs |
| `--radius-lg` | 14px | Cards, containers |
| `--radius-xl` | 19.6px | Modais, cards de destaque |

### 2.7 Tipografia

| Token | Valor |
|---|---|
| `--font-sans` | `'Inter Variable', sans-serif` |
| `--font-heading` | `var(--font-sans)` |

---

## 3. Índice de Módulos

| # | Módulo | ID das Telas | RFs Cobertos |
|---|---|---|---|
| A | Webchat (Interface principal do Repasse AI) | T-001 a T-011 | RF-001, RF-002, RF-008 a RF-010, RF-014 a RF-024, RF-026 a RF-029, RF-034, RF-035, RF-038, RF-040, RF-042, RF-046 a RF-048, RF-061 a RF-065, RF-101 |
| B | Notificações Proativas | T-012 a T-014 | RF-097, RF-098, RF-099 |
| C | Configurações do Usuário | T-015 a T-019 | RF-086, RF-087, RF-088, RF-089, RF-092, RF-100, RF-101, RF-102 |
| D | Admin — Supervisão IA | T-020 a T-022 | RF-066 a RF-069, RF-071 a RF-074, RF-076, RF-077 |

**Total de telas:** 22 | **Telas pendentes:** 0 | **Cobertura:** 100%

---

## 4. Wireframes por Módulo

---

## Módulo A — Webchat

### T-001 — FAB do Chat (Botão Flutuante)

**Rota:** overlay em todas as telas do Cessionário · **Role:** `CESSIONARIO` · **RFs:** RF-040

```javascript
// T-001 — FAB do Chat
// Posição: fixed bottom-right em todas as telas do Cessionário
// Tamanho: 56×56px (mobile) / 56×56px (desktop)

┌─────────────────────────────────────────────────────────┐
│  [QUALQUER TELA DO CESSIONÁRIO — conteúdo da página]    │
│                                                          │
│                                                          │
│                                          ┌──────────┐   │
│                                          │  [BADGE] │   │  ← Badge "9+" se alertas
│                                          │ ╔══════╗ │   │    cor: --destructive
│                                          │ ║  💬  ║ │   │  ← Ícone chat
│                                          │ ╚══════╝ │   │    bg: --primary
│                                          └──────────┘   │
│                              bottom: 24px, right: 24px  │
└─────────────────────────────────────────────────────────┘

ESTADO SEM BADGE (Empty):
  [●] — círculo --primary, ícone chat branco, sem badge

ESTADO COM BADGE (Populated):
  [●] — círculo --primary, ícone chat branco
  [3] — badge circular no canto superior direito, --destructive, texto branco
  "9+" — quando > 9 alertas não lidos

ESTADO OFFLINE:
  [●] — círculo --primary, opacity: 0.5
  tooltip ao hover: "Sem conexão"
```

**Componentes-chave:**
- `Button` (shadcn) — variante ghost com ícone, tamanho 56×56px, `border-radius: 50%`, bg `--primary`
- `Badge` — numérica, bg `--destructive`, posição absolute top-0 right-0
- `aria-label="Abrir Analista de Oportunidades"` no botão
- `aria-live="polite"` na badge

**Estados:**
- **Loading:** não aplicável — FAB renderiza imediatamente com estado local.
- **Empty:** FAB sem badge. Ícone chat, bg `--primary`.
- **Error:** não aplicável — FAB nunca falha; erros ocorrem dentro do chat.
- **Populated:** FAB com badge `--destructive` numérica; "9+" quando acima de 9.

**Notas:** Animação de aparição da badge: `scale-in` 150ms ease-out. Remoção: `scale-out` 150ms. `position: fixed`. Em mobile: `bottom: 16px, right: 16px`. Não renderiza no painel Admin.

---

### T-002 — Banner de Consentimento LGPD

**Rota:** overlay em T-003 (primeira abertura) · **Role:** `CESSIONARIO` · **RFs:** RF-101

```javascript
// T-002 — Banner de Consentimento LGPD
// Tipo: overlay/modal sobre o chat — bloqueia input até aceite

┌─────────────────────────────────────────────────────────┐
│  FUNDO: scrim overlay semi-transparente sobre T-003      │
│                                                          │
│         ┌─────────────────────────────────────┐         │
│         │  ┌───────────────────────────────┐  │         │
│         │  │  Repasse AI — Privacidade dos │  │         │
│         │  │  seus dados                   │  │         │
│         │  │  (H3, --foreground bold)      │  │         │
│         │  └───────────────────────────────┘  │         │
│         │                                      │         │
│         │  Para melhorar sua experiência, o    │         │
│         │  Repasse AI armazena suas conversas  │         │
│         │  por até 90 dias. Você pode apagar   │         │
│         │  seu histórico a qualquer momento    │         │
│         │  em Meu Perfil > Histórico de Chat.  │         │
│         │  (body, --foreground, 14px)           │         │
│         │                                      │         │
│         │  [Ver política de privacidade →]     │         │
│         │  (link --primary, 13px)              │         │
│         │                                      │         │
│         │  ┌──────────────┐  ┌─────────────┐  │         │
│         │  │  Ver política │  │   Aceitar   │  │         │
│         │  │  (outline)   │  │  (primary)  │  │         │
│         │  └──────────────┘  └─────────────┘  │         │
│         └─────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────┘

MOBILE (< 768px):
┌─────────────────────────┐
│  [Título consentimento] │
│                         │
│  [Texto descritivo]     │
│                         │
│  [Ver política →]       │
│                         │
│  ┌─────────────────────┐│
│  │  Ver política        ││  ← botão full-width
│  └─────────────────────┘│
│  ┌─────────────────────┐│
│  │      Aceitar         ││  ← botão full-width --primary
│  └─────────────────────┘│
└─────────────────────────┘
(full-screen com scroll vertical)
```

**Componentes-chave:**
- `Dialog` (shadcn) — `role="dialog"`, `aria-modal="true"`, `aria-labelledby` apontando para título
- `Button` — variante `outline` (Ver política) e `default` (Aceitar)
- `Link` — inline para política de privacidade

**Estados:**
- **Loading:** não aplicável — banner renderiza sincronicamente.
- **Empty:** não aplicável.
- **Error:** se falha ao registrar consentimento no backend, banner permanece com toast de erro: "Não foi possível registrar seu consentimento. Tente novamente." + botão "Tentar novamente". Botão "Aceitar" volta a ficar ativo.
- **Populated:** banner visível com texto, link e botões.
- **Offline:** botão "Aceitar" desabilitado, tooltip "Necessário ter conexão para aceitar."

**Notas:** Ao aceitar: slide-up 300ms ease-in-out; fade-in da mensagem de boas-vindas em T-003 (200ms). Tab order: link "Ver política" → botão "Aceitar".

---

### T-003 — Janela do Chat — Estado Ativo

**Rota:** overlay/painel lateral em telas do Cessionário · **Role:** `CESSIONARIO` · **RFs:** RF-038, RF-042, RF-046

```javascript
// T-003 — Janela do Chat — Estado Ativo
// Desktop: painel lateral fixo 420px | Mobile: full-screen 100dvh

// ── DESKTOP ──
┌──────────────────────────────────────────────────────┬──────────────────────────────┐
│  [CONTEÚDO DA PLATAFORMA]                            │  Analista de Oportunidades ✕ │
│                                                      │  (Header, --card, 56px h)    │
│                                                      │──────────────────────────────│
│                                                      │  [ÁREA DE HISTÓRICO]         │
│                                                      │  role="log" aria-live=polite │
│                                                      │                              │
│                                                      │  ╔═══════════════════════╗   │
│                                                      │  ║ Olá! Sou seu Analista ║   │ ← agente, esquerda
│                                                      │  ║ de Oportunidades...   ║   │   bg: --card
│                                                      │  ╚═══════════════════════╝   │
│                                                      │  09:42 (--muted-foreground)  │
│                                                      │                              │
│                                                      │       ╔════════════════╗     │
│                                                      │       ║ OPR-2024-001   ║     │ ← usuário, direita
│                                                      │       ╚════════════════╝     │   bg: --primary
│                                                      │              09:43           │
│                                                      │                              │
│                                                      │  ···  [digitando]            │ ← 3 pontos pulsando
│                                                      │──────────────────────────────│
│                                                      │  [CHIPS DE SUGESTÃO]         │
│                                                      │  [Analisar OPR] [Comparar]   │ ← --muted-foreground
│                                                      │──────────────────────────────│
│                                                      │  ┌──────────────────────┐ ▶  │
│                                                      │  │ Escreva sua mensagem │    │ ← input + botão envio
│                                                      │  └──────────────────────┘    │
│                                                      │  12 msgs restantes (today)   │ ← contador rate limit
└──────────────────────────────────────────────────────┴──────────────────────────────┘

// ── MOBILE ──
┌─────────────────────────┐
│ ← Voltar  Analista...  ⋮│  ← header fixo, 56px
│─────────────────────────│
│                         │
│ ╔═════════════════════╗ │
│ ║ Olá! Sou seu...     ║ │  ← agente, esquerda, --card
│ ╚═════════════════════╝ │
│ 09:42                   │
│                         │
│        ╔══════════════╗ │
│        ║ OPR-2024-001 ║ │  ← usuário, direita, --primary
│        ╚══════════════╝ │
│                  09:43  │
│                         │
│─────────────────────────│
│ [Analisar] [Comparar]   │  ← chips sugestão
│─────────────────────────│
│ ┌───────────────────┐ ▶ │  ← input fixo bottom + safe-area-inset
│ │ Escreva aqui      │   │
│ └───────────────────┘   │
└─────────────────────────┘

// ── ESTADO LOADING (skeleton) ──
┌───────────────────────────────┐
│  [████████████████████]       │  ← skeleton msg agente
│  [████████████]               │  ← skeleton msg curta
│           [█████████████████] │  ← skeleton msg usuário
└───────────────────────────────┘
(3 linhas de skeleton, max 500ms)

// ── ESTADO STREAM INTERROMPIDO (edge case) ──
│  ╔═══════════════════════════╗ │
│  ║ A análise da OPR-2024...  ║ │ ← texto recebido preservado
│  ║  ⚠ Resposta incompleta   ║ │ ← --muted-foreground, 12px
│  ║  [Tentar novamente]       ║ │ ← botão ghost, ao lado
│  ╚═══════════════════════════╝ │
```

**Componentes-chave:**
- `Sheet` (shadcn) — painel lateral no desktop, `role="complementary"`
- `ScrollArea` — área de histórico com `role="log"`, `aria-live="polite"`
- `Input` — `aria-label="Escreva sua mensagem para o Analista de Oportunidades"`, `aria-describedby` → contador
- `Button` — envio, variante `default`
- `Badge` — chips de sugestão, variante `outline`, cor `--muted-foreground`
- `Skeleton` — 3 linhas retangulares no loading

**Estados:**
- **Loading:** skeleton 3 linhas enquanto histórico carrega (max 500ms).
- **Empty:** mensagem de boas-vindas + chips de sugestão clicáveis.
- **Error:** toast no topo "Não foi possível carregar o histórico." — chat continua funcional.
- **Populated:** histórico com bubbles; usuário direita `--primary`; agente esquerda `--card`.
- **Offline:** input desabilitado, banner "Você está offline." no topo do chat.

**Notas:** Indicador digitando: 3 pontos pulsando, ciclo 800ms. Streaming SSE renderiza tokens progressivamente sem layout shift. Timestamps em `--muted-foreground`. Últimas 20 mensagens + botão "Carregar mais".

---

### T-004 — Chat — Bolha de Análise de Oportunidade

**Rota:** inline em T-003 · **Role:** `CESSIONARIO` · **RFs:** RF-001, RF-002, RF-008, RF-009, RF-010

```javascript
// T-004 — Bolha de Análise de Oportunidade
// Renderizada como ChatBubble do agente (esquerda) em T-003

┌──────────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border, radius-xl      │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  OPR-2024-001 · São Paulo, SP                           │ │  ← cabeçalho, --muted-foreground 12px
│  │─────────────────────────────────────────────────────────│ │
│  │                                                          │ │
│  │  Δ (Delta)                                               │ │  ← DESTAQUE PRIMÁRIO
│  │  ┌────────────────────────┐                             │ │
│  │  │  R$ 48.500             │ ← --foreground bold, 28px   │ │
│  │  │  +12.5% sobre o valor  │ ← --muted-foreground 12px   │ │
│  │  └────────────────────────┘                             │ │
│  │                                                          │ │
│  │  ┌────────────────┐  ┌─────────────────────┐           │ │
│  │  │ Comissão       │  │ Custo Escrow         │           │ │  ← SECUNDÁRIO
│  │  │ R$ 9.700 (2%)  │  │ R$ 3.100 (0.64%)    │           │ │
│  │  └────────────────┘  └─────────────────────┘           │ │
│  │                                                          │ │
│  │  [Conservador] [Base ✓] [Otimista]  ← tabs ROI         │ │  ← SECUNDÁRIO (TabComparativo)
│  │  ┌────────────────────────────────┐                     │ │
│  │  │  ROI (Base): 18.4% em 24 meses │                     │ │
│  │  └────────────────────────────────┘                     │ │
│  │                                                          │ │
│  │  Score de Risco: [● MODERADO]  ← badge amarela          │ │  ← ATENÇÃO (ScoreIndicador)
│  │  "Precificação regional elevada, histórico de 18m"      │ │
│  │                                                          │ │
│  │  ▼ Ver mais detalhes                                     │ │  ← TERCIÁRIO (colapsável)
│  │  ┌────────────────────────────────────────────────────┐ │ │
│  │  │  Comparativo Regional: Top 3 do segmento em SP     │ │ │
│  │  │  Histórico de Valorização: ████ ████ ████ (24m)    │ │ │
│  │  └────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘

// ── STATUS: EM NEGOCIAÇÃO ──
│  [🟠 Em negociação]  ← badge --warning (#F59E0B)            │
│  Esta oportunidade está em negociação.                       │
│  [Me avisar quando disponível]  ← botão outline --primary   │

// ── STATUS: ENCERRADA ──
│  [⬜ Encerrada]  ← badge --muted                             │
│  Esta oportunidade foi encerrada.                            │
│  Oportunidades semelhantes: [OPR-2024-015] [OPR-2024-022] [OPR-2024-031]  ← chips

// ── STATUS: NÃO ENCONTRADA ──
│  🔍 Oportunidade OPR-XXXX não encontrada.                   │
│  Verifique o código e tente novamente.                       │

// ── LOADING (streaming ativo) ──
│  ┌────────────────────────────┐  │
│  │  [████████████████████]   │  │  ← skeleton --muted, animate-pulse
│  │  [████████████]           │  │
│  │  [████████████████]       │  │
│  └────────────────────────────┘  │
```

**Componentes-chave:**
- `ChatBubble` — variante agente (esquerda), bg `--card`
- `OportunidadeCard` — variante análise completa
- `TabComparativo` — 3 abas ROI (Conservador/Base/Otimista), `role="tablist"`
- `ScoreIndicador` — Baixo (verde `--chart-1`), Moderado (amarelo custom), Alto (`--destructive`)
- `Collapsible` (shadcn) — seção "Ver mais detalhes"
- `Badge` — status da oportunidade
- `Skeleton` — loading inline

**Estados:**
- **Loading:** skeleton inline na bolha durante streaming.
- **Empty:** não aplicável — bolha só aparece após resposta.
- **Error:** mensagem "OPR não encontrado" com ícone lupa.
- **Populated:** card estruturado com Δ, Comissão, Escrow, ROI tabs, Score, seção colapsável.

**Notas:** Hierarquia visual: Δ primário (28px bold) → Comissão/Escrow secundários (16px) → Score badge atenção → ROI tabs → Comparativo/Histórico terciários (colapsável). Badge score anima com scale-in ao finalizar stream.

---

### T-005 — Chat — Bolha de Comparação

**Rota:** inline em T-003 · **Role:** `CESSIONARIO` · **RFs:** RF-014 a RF-018

```javascript
// T-005 — Bolha de Comparação (máx 5 oportunidades)
// Tabela responsiva como ChatBubble do agente

┌────────────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border, radius-xl        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Comparando 3 oportunidades  (14px --muted-foreground)   │  │
│  │                                                           │  │
│  │  ┌──────────────┬───────────────┬───────────────────┐   │  │
│  │  │ Campo        │ OPR-001 ★     │ OPR-002  │ OPR-003 │  │  │  ← ★ = destaque (maior Δ)
│  │  │              │ border:primary│          │         │  │  │    borda --primary na col
│  │  ├──────────────┼───────────────┼──────────┼─────────┤  │  │
│  │  │ Δ (Delta)    │ R$ 48.500     │ R$ 32.100│ R$ 41.200│ │  │
│  │  │ Comissão     │ R$ 9.700      │ R$ 6.420 │ R$ 8.240 │ │  │
│  │  │ Custo Escrow │ R$ 3.100      │ R$ 2.050 │ R$ 2.630 │ │  │
│  │  │ ROI Base     │ 18.4%         │ 11.2%    │ 15.8%    │ │  │
│  │  │ Score Risco  │ [🟡 MOD]      │ [🟢 BAIXO] │ [🟡 MOD]│ │  │
│  │  └──────────────┴───────────────┴──────────┴─────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘

// ── MOBILE — scroll horizontal ──
┌───────────────────────────────────┐
│  Comparando 3 oportunidades       │
│  ┌────────────┬──────────────┐    │
│  │ Campo      │ OPR-001 ★   │ ▶  │  ← chevron + pulse na borda direita
│  ├────────────┼──────────────┤    │     2 ciclos 1s, desaparece após scroll
│  │ Δ (Delta)  │ R$ 48.500   │    │
│  │ Comissão   │ R$ 9.700    │    │
│  │ Escrow     │ R$ 3.100    │    │
│  │ ROI Base   │ 18.4%       │    │
│  │ Score      │ [🟡 MOD]    │    │
│  └────────────┴──────────────┘    │
│  ← header fixo, scroll horizontal │
└───────────────────────────────────┘

// ── LOADING ──
│  ┌───────────────────────────────────┐  │
│  │  [████] [████████] [████] [████]  │  │  ← skeleton tabela: header + 3 linhas
│  │  [████] [████████] [████] [████]  │  │
│  │  [████] [████████] [████] [████]  │  │
│  └───────────────────────────────────┘  │
```

**Componentes-chave:**
- `ChatBubble` — variante agente
- `Table` (shadcn) — responsiva, scroll horizontal mobile, header fixo
- `TabComparativo` — variante N colunas
- `ScoreIndicador` — inline na tabela
- `Badge` — indicador de destaque (★ maior Δ)
- `Skeleton` — header + 3 linhas de placeholder

**Estados:**
- **Loading:** skeleton tabela (header + 3 linhas).
- **Empty:** não aplicável.
- **Error:** mensagem inline se OPR não encontrado, com código faltante indicado.
- **Populated:** tabela com coluna de destaque (borda `--primary`). Colunas ausentes com "-".

**Notas:** Scroll horizontal no mobile com indicador chevron + pulse (2 ciclos 1s, desaparece após scroll ou 3s). Máximo 5 colunas desktop; 3 visíveis sem scroll no mobile.

---

### T-006 — Chat — Bolha de Simulação

**Rota:** inline em T-003 · **Role:** `CESSIONARIO` · **RFs:** RF-019 a RF-024

```javascript
// T-006 — Bolha de Simulação (Proposta / Contraproposta)

┌──────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border, radius-xl  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Simulação para OPR-2024-001                         │ │  ← 12px --muted-foreground
│  │  Valor proposto: R$ 380.000                          │ │
│  │──────────────────────────────────────────────────────│ │
│  │                                                       │ │
│  │  ┌──────────────┐  ┌──────────────────────────────┐  │ │
│  │  │ Δ           │  │ Comissão (2.0%)              │  │ │
│  │  │ R$ 52.000   │  │ R$ 7.600                     │  │ │
│  │  └──────────────┘  └──────────────────────────────┘  │ │
│  │                                                       │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │ Custo Total Escrow: R$ 2.432 (0.64%)             │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                                                       │ │
│  │  [Conservador] [Base ✓] [Otimista]                   │ │  ← tabs ROI ajustado
│  │  ROI (Base): 21.3% em 24 meses                       │ │
│  │                                                       │ │
│  │  ⚠ Este agente não submete propostas automaticamente.│ │  ← italic --muted-foreground
│  │    Para enviar, use o botão "Fazer proposta" na       │ │
│  │    página da oportunidade.                            │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘

// ── LOADING ──
│  ┌─────────────────────────────┐  │
│  │  [████████████████████████] │  │  ← skeleton card único animate-pulse
│  └─────────────────────────────┘  │
```

**Componentes-chave:**
- `ChatBubble` — variante agente
- `OportunidadeCard` — variante simulação
- `TabComparativo` — 3 abas ROI
- `Skeleton` — card único

**Estados:**
- **Loading:** skeleton card único.
- **Empty:** não aplicável.
- **Error:** mensagem se valor inválido ou OPR não encontrado.
- **Populated:** card com valor proposto, comissão, Escrow, ROI tabs, nota de recusa de submissão.

**Notas:** Nota de recusa de submissão em `--muted-foreground` italic — obrigatória em todas as simulações. ROI renderiza por cenário durante streaming (Conservador → Base → Otimista).

---

### T-007 — Chat — Bolha de Portfólio

**Rota:** inline em T-003 · **Role:** `CESSIONARIO` · **RFs:** RF-026 a RF-029

```javascript
// T-007 — Bolha de Portfólio (multi-oportunidade)

┌──────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border, radius-xl  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Portfólio simulado — 3 oportunidades                │ │  ← header
│  │──────────────────────────────────────────────────────│ │
│  │                                                       │ │
│  │  ┌────────────────────────────────────────────────┐  │ │
│  │  │ OPR-2024-001 · Δ R$ 48.500 · Score: [🟡 MOD]  │  │ │  ← OportunidadeCard mini
│  │  └────────────────────────────────────────────────┘  │ │
│  │  ┌────────────────────────────────────────────────┐  │ │
│  │  │ OPR-2024-015 · Δ R$ 32.100 · Score: [🟢 BAI]  │  │ │
│  │  └────────────────────────────────────────────────┘  │ │
│  │  ┌────────────────────────────────────────────────┐  │ │
│  │  │ OPR-2024-022 · Δ R$ 41.200 · Score: [🟡 MOD]  │  │ │
│  │  └────────────────────────────────────────────────┘  │ │
│  │                                                       │ │
│  │  ┌────────────────────────────────────────────────┐  │ │
│  │  │  RESUMO DO PORTFÓLIO                           │  │ │  ← card de resumo, destaque
│  │  │  Capital comprometido: R$ 1.245.000            │  │ │    bold 20px
│  │  │  Δ Total: R$ 121.800                           │  │ │
│  │  │  [Conservador] [Base ✓] [Otimista]             │  │ │
│  │  │  ROI Total (Base): 16.8% · 24 meses            │  │ │
│  │  └────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘

// ── LOADING ──
│  [████████████████████████████] │  ← skeleton card 1
│  [████████████████████████████] │  ← skeleton card 2
│  [████████████████████████████] │  ← skeleton card 3 (stagger 100ms)
```

**Componentes-chave:**
- `ChatBubble` — variante agente
- `OportunidadeCard` — variante mini (compacto)
- `TabComparativo` — 3 abas ROI total
- `Skeleton` — 3 cards com stagger 100ms

**Estados:**
- **Loading:** skeleton 3 cards, stagger 100ms entre eles.
- **Empty:** não aplicável.
- **Error:** mensagem se alguma OPR do portfólio não encontrada.
- **Populated:** cards empilhados + card resumo com capital e ROI total.

**Notas:** Cards aparecem sequencialmente (stagger 100ms) durante streaming. OPRs inválidas marcadas com badge `--destructive` no card.

---

### T-008 — Chat — Bolha de Suporte Operacional

**Rota:** inline em T-003 · **Role:** `CESSIONARIO` · **RFs:** RF-047, RF-048

```javascript
// T-008 — Bolha de Suporte Operacional

┌──────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border, radius-xl  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Sobre o processo de KYC                             │ │  ← H4 --foreground bold
│  │──────────────────────────────────────────────────────│ │
│  │                                                       │ │
│  │  O KYC (Know Your Customer) é obrigatório para todos │ │  ← body texto formatado
│  │  os Cessionários antes de fazer propostas na         │ │
│  │  plataforma. O processo inclui:                      │ │
│  │                                                       │ │
│  │  • Validação de identidade (documento + selfie)      │ │
│  │  • Verificação de endereço                           │ │
│  │  • Análise de capacidade financeira                  │ │
│  │                                                       │ │
│  │  Prazo: até 2 dias úteis após envio dos documentos.  │ │
│  │                                                       │ │
│  │  [Ver documentação completa →]  ← link --primary 13px│ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘

// ── LOADING ──
│  [████████████████████████████████████████] │  ← skeleton parágrafo
│  [████████████████████████]                 │
│  [████████████████████████████████]         │

// ── ERRO (fora do escopo) ──
│  Desculpe, não tenho informações sobre esse assunto.    │
│  Para suporte especializado, entre em contato via       │
│  Central de Atendimento.                                │
│  [Abrir suporte →]  ← link --primary                   │
```

**Componentes-chave:**
- `ChatBubble` — variante agente
- `Skeleton` — parágrafo (1 bloco)
- Links externos formatados com `--primary`

**Estados:**
- **Loading:** skeleton parágrafo.
- **Empty:** não aplicável.
- **Error:** fallback se consulta excede escopo, com link para suporte.
- **Populated:** texto formatado com cabeçalho da regra, detalhes e links.

---

### T-009 — Chat — Bolha de Erro / Degradação

**Rota:** inline em T-003 · **Role:** `CESSIONARIO` · **RFs:** RF-063, RF-064, RF-065

```javascript
// T-009 — Bolha de Erro / Degradação
// Exibida: (a) timeout > 10s, (b) taxa erro agente > 30%

┌──────────────────────────────────────────────────────────┐
│  BOLHA SISTEMA — centrada, bg: --muted, border: --border │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  ⚠ O Analista está demorando mais que o esperado.   │ │  ← ícone warning --destructive
│  │  Você pode aguardar ou tentar novamente.             │ │
│  │                                                       │ │
│  │  ┌──────────────────┐  ┌────────────────────────┐   │ │
│  │  │   Aguardar       │  │   Tentar novamente     │   │ │  ← botões lado a lado
│  │  │   (outline)      │  │   (default --primary)  │   │ │
│  │  └──────────────────┘  └────────────────────────┘   │ │
│  │                                                       │ │
│  │  ┌────────────────────────────────────────────────┐  │ │
│  │  │  [Calcular comissão apenas]  ← botão ghost     │  │ │  ← se Calculadora disponível
│  │  └────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
(fade-in 200ms após timeout)
```

**Componentes-chave:**
- `ChatBubble` — variante sistema (centrado)
- `Alert` (shadcn) — `variant="destructive"` (ícone warning)
- `Button` — "Aguardar" outline, "Tentar novamente" default, "Calcular comissão" ghost

**Estados:**
- **Loading:** não aplicável — bolha de erro É o estado de falha.
- **Empty:** não aplicável.
- **Error:** este wireframe representa o estado de erro.
- **Populated:** mensagem de erro com duas (ou três) ações disponíveis.

---

### T-010 — Chat — Bolha de Dado Bloqueado

**Rota:** inline em T-003 · **Role:** `CESSIONARIO` · **RFs:** RF-034, RF-035

```javascript
// T-010 — Bolha de Dado Bloqueado (restrição de negócio, não erro técnico)

┌──────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border, radius-xl  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Não tenho acesso a informações sobre outros         │ │  ← texto informativo, sem ícone
│  │  Cessionários. Minha análise cobre apenas            │ │    cor: --foreground (não --destructive)
│  │  oportunidades disponíveis no marketplace e          │ │
│  │  dados da sua conta.                                 │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘

// ── 2ª INSISTÊNCIA (mesma mensagem + chips) ──
│  [não tenho acesso...]                                   │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Você pode me perguntar sobre:                        │ │
│  │  [Analisar OPR]  [Comparar OPRs]  [Simular proposta] │ │  ← chips sugestão
│  └──────────────────────────────────────────────────────┘ │

// ── 3ª INSISTÊNCIA (mesma mensagem sem chips) ──
│  [não tenho acesso...]  sem chips adicionais             │
```

**Componentes-chave:**
- `ChatBubble` — variante agente
- `Badge` — chips de sugestão (2ª insistência), variante outline

**Estados:**
- **Loading:** não aplicável — resposta determinística.
- **Empty:** não aplicável.
- **Error:** não aplicável — esta é uma restrição de negócio, não erro.
- **Populated:** mensagem informativa sem ícone de erro. Gradação por número de insistências.

**Notas:** Sem ícone de alerta vermelho — a mensagem é informativa, não alarmante. Texto em `--foreground`, não em `--destructive`.

---

### T-011 — Chat — Bolha de Rate Limit Atingido

**Rota:** inline em T-003 · **Role:** `CESSIONARIO` · **RFs:** RF-061, RF-062

```javascript
// T-011 — Bolha de Rate Limit Atingido

┌──────────────────────────────────────────────────────────┐
│  BOLHA SISTEMA — centrada, bg: --muted                   │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  🚫 Limite de mensagens atingido.                    │ │  ← ícone proibido
│  │  Você pode enviar novas mensagens em:                │ │
│  │                                                       │ │
│  │              00:47:23                                 │ │  ← contador hh:mm:ss
│  │         (--foreground bold 24px)                     │ │    atualizado a cada minuto
│  │                                                       │ │
│  │  [Input desabilitado — opacity 0.4]                  │ │  ← input em T-003 desabilitado
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
(input reabilitado automaticamente quando contador chega a 00:00:00)
```

**Componentes-chave:**
- `ChatBubble` — variante sistema
- Contador atualizado em tempo real
- `Input` em T-003 com `disabled` + opacity transition 200ms

**Estados:**
- **Loading:** não aplicável.
- **Empty:** não aplicável.
- **Error:** não aplicável.
- **Populated:** mensagem com contador regressivo. Input desabilitado automaticamente.

---

## Módulo B — Notificações Proativas

### T-012 — Bolha de Alerta Proativo — Nova Oportunidade

**Rota:** inline em T-003 (ao abrir chat) · **Role:** `CESSIONARIO` · **RFs:** RF-097

```javascript
// T-012 — Bolha de Alerta Proativo — Nova Oportunidade
// Slide-down 250ms ao abrir o chat

┌──────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border, radius-xl  │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  [● Novo alerta]  ← badge --primary texto branco     │ │
│  │                                                       │ │
│  │  Nova oportunidade compatível com seu perfil!        │ │  ← título, --foreground bold
│  │                                                       │ │
│  │  ┌────────────────────────────────────────────────┐  │ │
│  │  │  OPR-2024-043 · São Paulo, SP                  │  │ │  ← OportunidadeCard compacto
│  │  │  OPR: R$ 485.000 · Δ estimado: R$ 52.000       │  │ │
│  │  │  Score: [🟢 BAIXO]                              │  │ │
│  │  └────────────────────────────────────────────────┘  │ │
│  │                                                       │ │
│  │  [Ver oportunidade]  ← botão --primary               │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `ChatBubble` — variante agente
- `AlertaBadge` — textual, bg `--primary`
- `OportunidadeCard` — variante compacto
- `Button` — "Ver oportunidade", variante default

**Estados:**
- **Loading:** não aplicável — bolha inserida de forma assíncrona.
- **Empty:** não aplicável.
- **Error:** não exibido ao usuário; sistema retenta via fila RabbitMQ.
- **Populated:** card compacto com dados da oportunidade e CTA.

---

### T-013 — Bolha de Alerta Proativo — Prazo de Escrow

**Rota:** inline em T-003 (ao abrir chat) · **Role:** `CESSIONARIO` · **RFs:** RF-098

```javascript
// T-013 — Bolha de Alerta Proativo — Prazo de Escrow

┌──────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  [⏰ Prazo urgente]  ← badge --destructive           │ │
│  │                                                       │ │
│  │  Prazo de depósito se aproximando!                   │ │  ← bold --foreground
│  │                                                       │ │
│  │  Negociação: NEG-2024-0089                           │ │
│  │  Valor: R$ 380.000                                   │ │
│  │  Prazo: 2 dias úteis restantes                       │ │  ← --destructive se < 2 dias
│  │                                                       │ │
│  │  [Ver negociação]  ← botão --destructive outline     │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `ChatBubble` — variante agente
- `AlertaBadge` — textual, bg `--destructive`
- `Button` — variante `outline` com `--destructive`

**Estados:**
- **Loading:** não aplicável.
- **Empty:** não aplicável.
- **Error:** não exibido ao usuário; reprocessado via RabbitMQ.
- **Populated:** alerta com dados da negociação e CTA.

---

### T-014 — Bolha de Alerta Proativo — Mudança de Status

**Rota:** inline em T-003 (ao abrir chat) · **Role:** `CESSIONARIO` · **RFs:** RF-099

```javascript
// T-014 — Bolha de Alerta Proativo — Mudança de Status

┌──────────────────────────────────────────────────────────┐
│  BOLHA AGENTE — bg: --card, border: --border             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  [📋 Atualização]  ← badge --primary                 │ │
│  │                                                       │ │
│  │  Sua proposta foi atualizada!                        │ │  ← bold --foreground
│  │                                                       │ │
│  │  Proposta: PRO-2024-0127                             │ │
│  │  Novo status: [✅ Aceita pelo Cedente]               │ │  ← badge verde
│  │                                                       │ │
│  │  O que isso significa: O Cedente aceitou sua        │ │  ← explicação em linguagem clara
│  │  proposta. O próximo passo é o depósito em Escrow.  │ │
│  │                                                       │ │
│  │  [Ver proposta]  ← botão --primary                   │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `ChatBubble` — variante agente
- `AlertaBadge` — textual, bg `--primary`
- `Badge` — status da proposta com cor semântica
- `Button` — variante default

**Estados:**
- **Loading:** não aplicável.
- **Empty:** não aplicável.
- **Error:** não exibido ao usuário; reprocessado via RabbitMQ.
- **Populated:** alerta com status, explicação em linguagem clara e CTA.

---

## Módulo C — Configurações do Usuário

### T-015 — Meu Perfil — Seção WhatsApp

**Rota:** `/perfil/whatsapp` (subseção de Meu Perfil) · **Role:** `CESSIONARIO` · **RFs:** RF-086, RF-092

```javascript
// T-015 — Meu Perfil — Seção WhatsApp
// Layout: sidebar (AppLayout) + conteúdo principal

┌──────────┬───────────────────────────────────────────────────────────┐
│ SIDEBAR  │  Meu Perfil                                  [Salvar]     │
│ #FAFAFA  │───────────────────────────────────────────────────────────│
│          │  [Dados Pessoais]  [WhatsApp ●]  [Notificações]  [Chat]  │  ← tabs de seção
│ Logo     │───────────────────────────────────────────────────────────│
│          │                                                            │
│ Nav      │  WhatsApp para Notificações                               │  ← H3 --foreground
│ items    │  Vincule seu WhatsApp para receber alertas proativos       │  ← 14px --muted-foreground
│          │  sobre oportunidades, prazos e status de propostas.       │
│          │                                                            │
│ User     │  ┌────────────────────────────────────────────────────┐   │
│ Avatar   │  │  ESTADO: NaoVinculado                               │   │
│          │  │  ┌──────────────────────────────────────────────┐  │   │
└──────────┤  │  │  Número do WhatsApp                          │  │   │
           │  │  │  +55 (__)  _____-____  ← input com máscara   │  │   │
           │  │  └──────────────────────────────────────────────┘  │   │
           │  │  [Vincular WhatsApp]  ← botão --primary full-width  │   │
           │  └────────────────────────────────────────────────────┘   │
           │                                                            │
           │  ┌────────────────────────────────────────────────────┐   │
           │  │  ESTADO: Ativo                                      │   │
           │  │  ✅ WhatsApp vinculado                              │   │  ← ícone verde
           │  │  (11) 98765-4321  ← número mascarado               │   │
           │  │  [Desvincular]  ← botão outline --destructive      │   │
           │  └────────────────────────────────────────────────────┘   │
           │                                                            │
           │  ┌────────────────────────────────────────────────────┐   │
           │  │  ESTADO: Suspenso                                   │   │
           │  │  ⚠ Verificação pendente                            │   │  ← ícone warning
           │  │  (11) 98765-4321                                    │   │
           │  │  [Re-verificar agora]  ← botão --primary           │   │
           │  └────────────────────────────────────────────────────┘   │
           └───────────────────────────────────────────────────────────┘

// ── LOADING ──
┌─────────────────────────────────────────────────────────┐
│  [████████████████████████████████████]  ← skeleton     │  ← campo número
│  [██████████████████]  ← skeleton botão                 │
└─────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- AppLayout (sidebar + conteúdo)
- `Tabs` (shadcn) — seções de Meu Perfil
- `Input` — número com máscara `+55 (DDD) NNNNN-NNNN`
- `Button` — "Vincular" primary, "Desvincular" destructive outline, "Re-verificar" primary
- `Badge` — status vinculação com ícone semântico
- `Skeleton` — campo + botão

**Estados:**
- **Loading:** skeleton do campo de número enquanto status carrega.
- **Empty (NaoVinculado):** campo de número + botão "Vincular WhatsApp".
- **Error:** mensagem inline abaixo do campo se número inválido.
- **Populated (Ativo):** número mascarado + botão "Desvincular".
- **Populated (Suspenso):** aviso pendente + botão "Re-verificar agora".
- **Offline:** botão desabilitado com tooltip "Sem conexão".

**Notas:** Status atualizado com fade-in 200ms após mudança de estado.

---

### T-016 — Modal — Fluxo de Vinculação WhatsApp

**Rota:** modal em `/perfil/whatsapp` · **Role:** `CESSIONARIO` · **RFs:** RF-086, RF-087, RF-088, RF-089

```javascript
// T-016 — Modal de Vinculação WhatsApp (3 steps)
// Desktop: modal centrado 480px | Mobile: bottom sheet full-width

// ── STEP 1 — Inserir número ──
┌──────────────────────────────────────────────────┐
│ ✕  Vincular WhatsApp                             │  ← título + botão fechar
│──────────────────────────────────────────────────│
│                                                   │
│  [1 ●]──────[2 ○]──────[3 ○]  ← StepProgress    │
│  Inserir nº  Código SMS  Confirmar WA             │
│                                                   │
│  Insira o número do WhatsApp que receberá         │
│  as notificações do Repasse AI.                   │
│                                                   │
│  ┌────────────────────────────────────────────┐  │
│  │  +55 (__)  _____-____                      │  │  ← input máscara
│  └────────────────────────────────────────────┘  │
│  [mensagem de erro inline se inválido]            │
│                                                   │
│  [Enviar código]  ← botão --primary full-width    │
│  [Cancelar]  ← botão ghost, sem confirmação no step 1
└──────────────────────────────────────────────────┘

// ── STEP 2 — Inserir OTP SMS ──
┌──────────────────────────────────────────────────┐
│ ✕  Vincular WhatsApp                             │
│──────────────────────────────────────────────────│
│  [1 ✓]──────[2 ●]──────[3 ○]  ← StepProgress   │
│                                                   │
│  Digite o código enviado para +55 (11) 9****-1   │
│                                                   │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐           │  ← OTPInput 6 dígitos
│  │ _ │ │ _ │ │ _ │ │ _ │ │ _ │ │ _ │           │    autocomplete="one-time-code"
│  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘           │
│                                                   │
│  Expira em: 14:32  ← contador regressivo          │
│  2 tentativas restantes                          │
│                                                   │
│  [Reenviar código] ← desabilitado 60s após envio  │
│                                                   │
│  [Verificar]  ← botão --primary full-width        │
│  [Cancelar → confirmação obrigatória]  ← ghost    │
└──────────────────────────────────────────────────┘

// ── STEP 2 — BLOQUEIO ──
┌──────────────────────────────────────────────────┐
│  🔒 Muitas tentativas incorretas.                │
│  Aguarde para tentar novamente:                  │
│                                                   │
│              29:47  ← contador mm:ss             │
│                                                   │
│  [todos os inputs desabilitados]                 │
└──────────────────────────────────────────────────┘

// ── STEP 3 — Aguardar confirmação WhatsApp ──
┌──────────────────────────────────────────────────┐
│  [1 ✓]──────[2 ✓]──────[3 ●]  ← StepProgress   │
│                                                   │
│       [ícone WhatsApp animado — pulse]            │
│                                                   │
│  Aguardando confirmação pelo WhatsApp             │
│  Enviamos uma mensagem para +55 (11) 9****-1.    │
│  Toque no link recebido para confirmar.           │
│                                                   │
│  Expira em: 23h 58m                              │
│                                                   │
│  [Cancelar → confirmação obrigatória]             │
└──────────────────────────────────────────────────┘

// ── CONFIRMAÇÃO DE FECHAMENTO (steps 2 e 3) ──
┌──────────────────────────────────────────────────┐
│  Tem certeza?                                    │
│  O código enviado expirará e você precisará      │
│  reiniciar o processo de vinculação.             │
│                                                   │
│  [Cancelar]     [Sair]  ← botão --destructive    │
└──────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Dialog` (shadcn) — `role="dialog"`, `aria-modal="true"`, `aria-labelledby` → título do step ativo
- `StepProgress` — 3 steps, tokens `--primary`, `--muted`, `--muted-foreground`
- `OTPInput` — 6 campos, `autocomplete="one-time-code"`, variantes Normal/Erro/Bloqueado
- `Input` — máscara telefone no step 1
- `Button` — primary, ghost, destructive
- Focus trap obrigatório — ao abrir: foco no primeiro input; ao fechar: foco retorna ao botão de origem em T-015

**Estados:**
- **Loading:** spinner inline no botão durante envio do OTP.
- **Empty:** step 1 com campo vazio focado.
- **Error:** erro inline abaixo de campos; toast de bloqueio.
- **Populated:** steps progressivos com dados preenchidos; confirmação de fechamento nos steps 2 e 3.

**Notas:** Transição entre steps: slide horizontal 300ms ease-in-out. `aria-live="assertive"` nos bloqueios; `aria-live="polite"` na expiração normal. Mobile: bottom sheet full-width.

---

### T-017 — Modal — Confirmação de Desvinculação WhatsApp

**Rota:** modal em `/perfil/whatsapp` · **Role:** `CESSIONARIO` · **RFs:** RF-092

```javascript
// T-017 — Modal de Confirmação de Desvinculação

┌──────────────────────────────────────────────────┐
│  Desvincular WhatsApp?                           │  ← H4 --foreground bold
│──────────────────────────────────────────────────│
│                                                   │
│  Ao desvincular, você não receberá mais alertas   │
│  via WhatsApp sobre oportunidades, prazos e       │
│  status de propostas.                             │
│                                                   │
│  Esta ação pode ser desfeita vinculando           │
│  novamente o seu número.                          │
│                                                   │
│  ┌────────────────┐  ┌─────────────────────────┐ │
│  │   Cancelar     │  │      Desvincular         │ │  ← Cancelar: outline | Desv: --destructive
│  │   (outline)    │  │   (--destructive)        │ │
│  └────────────────┘  └─────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `AlertDialog` (shadcn) — confirmação destrutiva
- `Button` — "Cancelar" outline, "Desvincular" destructive

**Estados:**
- **Loading:** botão "Desvincular" com spinner inline durante processamento.
- **Empty:** não aplicável.
- **Error:** toast "Não foi possível desvincular. Tente novamente."
- **Populated:** modal com aviso e botões.

**Notas:** Ao confirmar: fade-out 150ms; status em T-015 atualiza para NaoVinculado com fade-in.

---

### T-018 — Meu Perfil — Seção Notificações

**Rota:** `/perfil/notificacoes` · **Role:** `CESSIONARIO` · **RFs:** RF-100

```javascript
// T-018 — Seção Notificações

┌──────────┬───────────────────────────────────────────────────────────┐
│ SIDEBAR  │  Meu Perfil                                               │
│          │───────────────────────────────────────────────────────────│
│          │  [Dados Pessoais]  [WhatsApp]  [Notificações ●]  [Chat]  │
│          │───────────────────────────────────────────────────────────│
│          │                                                            │
│          │  Preferências de Notificação                              │  ← H3
│          │  Escolha quais alertas quer receber no chat e WhatsApp.   │  ← --muted-foreground 14px
│          │                                                            │
│          │  ┌────────────────────────────────────────────────────┐   │
│          │  │ 🔔 Nova Oportunidade                    [ ● ]  ON  │   │  ← ToggleNotificacao ON
│          │  │    Alertas quando uma nova oportunidade             │   │
│          │  │    compatível com seu perfil for publicada.         │   │  ← desc. 12px --muted-fgd
│          │  └────────────────────────────────────────────────────┘   │
│          │  ┌────────────────────────────────────────────────────┐   │
│          │  │ ⏰ Prazo de Escrow                      [ ● ]  ON  │   │  ← ToggleNotificacao ON
│          │  │    Alertas sobre prazos de depósito em Escrow       │   │
│          │  │    com 2 dias úteis de antecedência.                │   │
│          │  └────────────────────────────────────────────────────┘   │
│          │  ┌────────────────────────────────────────────────────┐   │
│          │  │ 📋 Mudança de Status                    [   ]  OFF │   │  ← ToggleNotificacao OFF
│          │  │    Alertas sobre mudanças no status das suas        │   │
│          │  │    propostas enviadas.                              │   │
│          │  └────────────────────────────────────────────────────┘   │
└──────────┴───────────────────────────────────────────────────────────┘

// ── LOADING ──
│  [████████████████████████████████████  ○ ]  │  ← skeleton toggle linha 1
│  [████████████████████████████████████  ○ ]  │  ← skeleton toggle linha 2
│  [████████████████████████████████████  ○ ]  │  ← skeleton toggle linha 3
```

**Componentes-chave:**
- AppLayout (sidebar + conteúdo)
- `Switch` (shadcn) — variantes Ativo/Inativo/Desabilitado; animação spring nativa 150ms
- `Skeleton` — 3 linhas de toggle

**Estados:**
- **Loading:** skeleton 3 toggles.
- **Empty:** não aplicável — sempre há 3 tipos de notificação.
- **Error:** toast "Não foi possível salvar sua preferência." + toggle reverte estado.
- **Populated:** 3 linhas com toggle, label, descrição e estado atual.
- **Offline:** toggles desabilitados com tooltip "Sem conexão".

---

### T-019 — Meu Perfil — Seção Histórico de Chat

**Rota:** `/perfil/historico` · **Role:** `CESSIONARIO` · **RFs:** RF-102

```javascript
// T-019 — Seção Histórico de Chat

┌──────────┬───────────────────────────────────────────────────────────┐
│ SIDEBAR  │  Meu Perfil                                               │
│          │───────────────────────────────────────────────────────────│
│          │  [Dados Pessoais]  [WhatsApp]  [Notificações]  [Chat ●]  │
│          │───────────────────────────────────────────────────────────│
│          │                                                            │
│          │  Histórico de Conversas                                   │  ← H3
│          │  Suas conversas com o Repasse AI ficam salvas por 90      │  ← 14px --muted-foreground
│          │  dias. Você pode apagá-las a qualquer momento.            │
│          │                                                            │
│          │  ┌────────────────────────────────────────────────────┐   │
│          │  │  22/03/2026  09:42 — "OPR-2024-001 — análise..."  │   │  ← item conversa, 14px
│          │  ├────────────────────────────────────────────────────┤   │
│          │  │  21/03/2026  14:15 — "Comparar OPR-2024-015 e..." │   │
│          │  ├────────────────────────────────────────────────────┤   │
│          │  │  20/03/2026  11:30 — "Simular proposta para..."    │   │
│          │  └────────────────────────────────────────────────────┘   │
│          │                                                            │
│          │  [Apagar tudo]  ← botão outline --destructive             │
│          └───────────────────────────────────────────────────────────┘

// ── EMPTY STATE ──
│  (ícone chat vazio)                               │
│  Nenhuma conversa registrada ainda.               │  ← --muted-foreground
│  [Abrir chat]  ← botão --primary                 │

// ── CONFIRMAÇÃO DE EXCLUSÃO (inline) ──
│  ┌────────────────────────────────────────────┐  │
│  │  ⚠ Ação irreversível.                      │  │  ← Alert variant destructive
│  │  Histórico será excluído em até 48 horas.  │  │
│  │  ┌───────────────┐  ┌─────────────────┐   │  │
│  │  │   Cancelar    │  │  Confirmar excl.│   │  │
│  │  └───────────────┘  └─────────────────┘   │  │
│  └────────────────────────────────────────────┘  │

// ── LOADING ──
│  [████████████████████████████████████]  │  ← skeleton lista de conversas
│  [████████████████████████████]          │
│  [██████████████████████████████████]    │
```

**Componentes-chave:**
- AppLayout (sidebar + conteúdo)
- `ScrollArea` — lista de conversas
- `Button` — "Apagar tudo" outline destructive, "Abrir chat" primary, "Confirmar exclusão" destructive
- `Alert` (shadcn) — aviso de ação irreversível
- `Skeleton` — lista de conversas

**Estados:**
- **Loading:** skeleton lista de conversas.
- **Empty:** ícone + mensagem + CTA "Abrir chat".
- **Error:** toast "Não foi possível carregar o histórico."
- **Populated:** lista com data, hora e preview truncado; botão "Apagar tudo".

**Notas:** Lista some com fade-out após confirmação. Toast "Exclusão solicitada. Você receberá confirmação em até 48 horas." persiste 5s.

---

## Módulo D — Admin — Supervisão IA

### T-020 — Painel de Supervisão IA — Listagem

**Rota:** `/admin/supervisao` · **Role:** `ADMIN` · **RFs:** RF-066, RF-067, RF-068, RF-069

```javascript
// T-020 — Painel de Supervisão IA — Listagem
// Layout: sidebar Admin + conteúdo full-width

┌──────────────────┬──────────────────────────────────────────────────────────┐
│ SIDEBAR ADMIN    │  Supervisão IA                                           │
│ #FAFAFA          │──────────────────────────────────────────────────────────│
│ w: 256px         │  Filtros: [Período ▾]  [Canal ▾]  [Confiança ▾]  [Limpar]│
│                  │──────────────────────────────────────────────────────────│
│ Dashboard        │                                                           │
│ Supervisão IA ● │  ┌──┬────────┬──────────────┬──────────────┬────┬──────┐  │
│ Métricas         │  │ID│ Data   │ Pergunta     │ Confiança    │ Lat│Canal │  │  ← tabela
│                  │  ├──┼────────┼──────────────┼──────────────┼────┼──────┤  │
│ Admin name       │  │01│22/03 09│ "Analisar OPR│ [● 92%] 🟢  │1.2s│ Web  │  │  ← verde ≥80%
│ Avatar           │  │02│22/03 08│ "Comparar OP │ [● 67%] 🟡  │2.1s│ Web  │  │  ← amarelo 50-79%
└──────────────────┤  │03│22/03 07│ "Simular pro │ [● 38%] 🔴  │4.3s│ WA   │  │  ← vermelho <50%
                   │  └──┴────────┴──────────────┴──────────────┴────┴──────┘  │
                   │  [← Anterior]  Página 1 de 5  [Próxima →]                 │
                   └──────────────────────────────────────────────────────────┘

// ── LOADING ──
┌────────────────────────────────────────────────────────────────┐
│  [████] [████████] [████████████████] [████████] [████] [████]│  ← skeleton linha 1
│  [████] [████████] [████████████████] [████████] [████] [████]│  ← skeleton linha 2
│  ... × 10 linhas                                               │
└────────────────────────────────────────────────────────────────┘

// ── EMPTY STATE ──
│  Nenhuma interação registrada ainda.  (--muted-foreground)     │
│  [sem CTA — painel passivo]                                     │

// ── NOVA INTERAÇÃO (highlight) ──
│  [linha aparece no topo com bg --accent highlight 2s]          │
```

**Componentes-chave:**
- AppLayout Admin (sidebar 256px + conteúdo)
- `Table` (shadcn) — `role="table"`, `scope="col"` em headers, `aria-sort` em colunas ordenáveis
- `Select` — filtros de período, canal, confiança
- `AlertaBadge` — confiança com cor semântica (verde/amarelo/vermelho), `aria-label="Confiança: X% — Alta/Média/Baixa"`
- `Pagination` (shadcn) — 20 itens/página desktop, 10 em tablet
- `Skeleton` — 10 linhas de tabela

**Estados:**
- **Loading:** skeleton 10 linhas de tabela.
- **Empty:** "Nenhuma interação registrada ainda." sem CTA.
- **Error:** banner "Não foi possível carregar as interações. [Tentar novamente]."
- **Populated:** tabela com badge de confiança colorida, paginação.

**Notas:** Nova interação aparece no topo com highlight 2s em `--accent`. Tablet: sidebar recolhível 48px, 10 itens/página. Admin não otimizado para mobile — aviso para usar tela maior.

---

### T-021 — Supervisão IA — Detalhe da Interação + Takeover

**Rota:** `/admin/supervisao/:id` · **Role:** `ADMIN` · **RFs:** RF-071, RF-072, RF-073, RF-074

```javascript
// T-021 — Detalhe da Interação + Takeover

// ── SEM TAKEOVER DISPONÍVEL ──
┌──────────────────┬──────────────────────────────────────────────────────────┐
│ SIDEBAR ADMIN    │  ← Supervisão IA  /  Interação #003                     │
│                  │──────────────────────────────────────────────────────────│
│                  │  ┌─────────────────────┐  ┌────────────────────────┐    │
│                  │  │  DADOS DA INTERAÇÃO  │  │  CONFIANÇA: 92%        │    │
│                  │  │  ID: #003            │  │  [████████████] 🟢     │    │  ← progress bar
│                  │  │  Data: 22/03 09:42   │  │  Latência: 1.2s        │    │
│                  │  │  Canal: Web          │  │  Dados usados: 4 docs  │    │
│                  │  └─────────────────────┘  └────────────────────────┘    │
│                  │                                                           │
│                  │  CONVERSA                                                │
│                  │  ┌──────────────────────────────────────────────────┐   │
│                  │  │  [Cessionário]: Analisar OPR-2024-001             │   │
│                  │  │  [Agente]: [Bolha análise completa — T-004]       │   │
│                  │  └──────────────────────────────────────────────────┘   │
│                  │                                                           │
│                  │  [Assumir conversa]  ← DESABILITADO, tooltip:            │
│                  │   "Confiança acima do threshold (80%)"                   │
└──────────────────┴──────────────────────────────────────────────────────────┘

// ── COM TAKEOVER DISPONÍVEL (confiança < threshold) ──
│  Confiança: 38%  🔴                                               │
│  [Assumir conversa]  ← ATIVO, --primary                           │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  Motivo do takeover: (obrigatório)                            │ │  ← aparece ao clicar
│  │  ┌────────────────────────────────────────────────────────┐  │ │
│  │  │  Selecione ou descreva o motivo...  ▾                  │  │ │  ← Select + textarea
│  │  └────────────────────────────────────────────────────────┘  │ │
│  │  [Confirmar takeover]  ← --primary  [Cancelar]  ← ghost      │ │
│  └──────────────────────────────────────────────────────────────┘ │

// ── DURANTE TAKEOVER ──
┌──────────────────┬──────────────────────┬─────────────────────────────────┐
│ SIDEBAR ADMIN    │  HISTÓRICO           │  ADMIN RESPONDE                  │
│                  │                      │                                  │
│                  │  [Cessionário]:      │  ╔═══════════════════════════╗   │
│                  │  Analisar OPR-001    │  ║ [Cessionário]: preciso...  ║  │
│                  │                      │  ╚═══════════════════════════╝   │
│                  │  [Agente]:           │                                  │
│                  │  [análise T-004]     │  ┌────────────────────────────┐  │
│                  │                      │  │ Escreva sua resposta...    │  │  ← input Admin
│                  │                      │  └────────────────────────────┘  │
│                  │                      │  [Enviar]  [Encerrar takeover]   │
└──────────────────┴──────────────────────┴─────────────────────────────────┘

// BANNER TAKEOVER (topo do painel, persistente):
┌──────────────────────────────────────────────────────────────────────────┐
│  !! VOCÊ ESTÁ CONTROLANDO ESTA CONVERSA  —  [Encerrar takeover]  !!     │
│  bg: --destructive  |  texto: --destructive-foreground                    │
└──────────────────────────────────────────────────────────────────────────┘

// ── TAKEOVER CONCORRENTE (segundo Admin) ──
│  Toast: "Esta conversa já está sendo controlada por outro membro da equipe."  │
│  Botão "Assumir conversa" fica DESABILITADO durante takeover ativo.           │
```

**Componentes-chave:**
- AppLayout Admin
- `Progress` (shadcn) — nível de confiança
- `ScrollArea` — histórico de conversa
- `Textarea` + `Select` — motivo de takeover
- `Button` — "Assumir" primary, "Confirmar" primary, "Encerrar takeover" destructive
- `Toast` — feedback concorrência
- Banner de status takeover em `--destructive`
- `aria-describedby` no botão de takeover → tooltip elegibilidade

**Estados:**
- **Loading:** skeleton do painel de detalhe.
- **Empty:** não aplicável — detalhe requer interação selecionada.
- **Error:** toast "Não foi possível assumir a conversa."
- **Populated (sem takeover):** dados completos, botão desabilitado com tooltip.
- **Populated (takeover disponível):** botão ativo + campo de motivo ao clicar.
- **Populated (durante takeover):** painel dividido + banner persistente `--destructive`.

**Notas:** Takeover concorrente: primeiro Admin vence; segundo recebe toast imediato e botão desabilitado. Banner de takeover: posição top sticky, `--destructive` fundo, não pode ser fechado. Slide-in do painel de input: 300ms, right.

---

### T-022 — Dashboard Admin — Métricas do Repasse AI

**Rota:** `/admin/metricas` · **Role:** `ADMIN` · **RFs:** RF-076, RF-077

```javascript
// T-022 — Dashboard Admin — Métricas do Repasse AI

┌──────────────────┬──────────────────────────────────────────────────────────┐
│ SIDEBAR ADMIN    │  Métricas do Repasse AI                    [Exportar]    │
│                  │──────────────────────────────────────────────────────────│
│ Dashboard        │  Período: [7d] [30d ✓] [90d] [Custom ▾]                 │  ← TabComparativo
│ Supervisão IA    │──────────────────────────────────────────────────────────│
│ Métricas ●       │                                                           │
│                  │  ┌───────────────────────────────────────────────────┐   │
│                  │  │  VOLUME DE INTERAÇÕES  ← card primário (maior)    │   │  ← col-span-2
│                  │  │                                                    │   │
│                  │  │  2.847                                             │   │  ← 40px bold
│                  │  │  ▲ +12% vs período anterior                       │   │  ← 12px verde
│                  │  └───────────────────────────────────────────────────┘   │
│                  │                                                           │
│                  │  ┌────────────────────┐  ┌────────────────────────────┐  │
│                  │  │  CSAT              │  │  TAXA DE TAKEOVER           │  │  ← cards secundários
│                  │  │  4.2 / 5.0         │  │  3.2%                      │  │
│                  │  │  ▲ +0.3 vs ant.    │  │  ▼ -0.8% vs ant.           │  │
│                  │  └────────────────────┘  └────────────────────────────┘  │
│                  │                                                           │
│                  │  ┌────────────────────┐  ┌────────────────────────────┐  │
│                  │  │  TAXA DE RECUSA    │  │  LATÊNCIA MÉDIA             │  │
│                  │  │  8.7%             │  │  1.8s                      │  │
│                  │  └────────────────────┘  └────────────────────────────┘  │
│                  │                                                           │
│                  │  ┌───────────────────────────────────────────────────┐   │
│                  │  │  Interações por dia  (gráfico de linha)           │   │  ← metade inferior
│                  │  │  ████████████████████████████████████████        │   │
│                  │  │  ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔   │   │
│                  │  └───────────────────────────────────────────────────┘   │
└──────────────────┴──────────────────────────────────────────────────────────┘

// ── EMPTY (sem dados no período) ──
│  [card]  --  ← valor "--" com tooltip "Sem dados para o período selecionado"  │

// ── EMPTY FIRST-USE (zero interações históricas) ──
│  [card]  --                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │  (linha do tempo sem dados)                                               │ │
│  │                                                                           │ │
│  │  Ainda não há interações registradas.                                     │ │  ← center
│  │  As métricas aparecerão aqui após os primeiros usos do Repasse AI.       │ │
│  │                                                                           │ │
│  │  [Ver documentação]  ← botão outline --primary                           │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │

// ── LOADING ──
│  [████████████████████████████████████████████████]  ← skeleton card primário  │
│  [████████████████] [████████████████]                ← skeleton cards sec.    │
│  [████████████████] [████████████████]                ← skeleton cards sec.    │
│  [████████████████████████████████████████████████]  ← skeleton gráfico       │
```

**Componentes-chave:**
- AppLayout Admin
- `Card` (shadcn) — 5 cards de métrica, grid 12 col (volume col-span-2)
- `TabComparativo` — filtro de período (7d/30d/90d/Custom)
- Gráfico de linha — `--chart-3` (#0084D1) como cor principal
- `Skeleton` — 5 cards + gráfico
- `Button` — "Exportar" outline, "Ver documentação" outline primary (first-use)

**Estados:**
- **Loading:** skeleton 5 cards + gráfico.
- **Empty (período sem dados):** cards com "--" e tooltip.
- **Empty (first-use):** cards "--" + mensagem orientativa centralizada + CTA documentação.
- **Error:** banner de erro + cards em estado degradado.
- **Populated:** 5 cards + gráfico de linha com animação number-flip 300ms ao trocar período.

**Notas:** Hierarquia visual: Volume (primário, maior, col-span-2, top-left) → CSAT (segundo destaque) → Taxa takeover/recusa (secundários) → Latência (informativo menor). Estado first-use é distinto do empty de período — mensagem e contexto diferentes.

---

## 5. Convenções Globais de Wireframe

### 5.1 Estados Obrigatórios (toda tela)

| Estado | Implementação | Quando |
|---|---|---|
| **Loading** | Skeleton retangulares `animate-pulse` — **NUNCA spinner** | Enquanto fetch/carregamento ativo |
| **Empty** | Ícone + título + descrição + CTA (quando cabível) | Dados ausentes ou primeiro acesso |
| **Error** | `Alert variant="destructive"` + botão retry | Falha de request/rede |
| **Populated** | Conteúdo real com dados | Após carregamento bem-sucedido |

### 5.2 Padrão de Sidebar (AppLayout)

```javascript
// AppLayout — padrão para todas as telas autenticadas com navegação própria

Sidebar (w: 256px expanded / 48px collapsed):
  ├── Logo (topo, 48px altura)
  ├── NavigationMenu — itens conforme Mapa de Telas
  │     ├── Ativo: bg --sidebar-primary, texto --sidebar-primary-foreground
  │     └── Hover: bg --sidebar-accent, texto --sidebar-accent-foreground
  ├── QuickActions — atalhos contextuais (quando aplicável)
  └── UserAvatar + nome + role (fundo, 64px altura)

Tokens de sidebar:
  bg:           --sidebar (#FAFAFA light / #171717 dark)
  borda:        --sidebar-border (right-side, 1px)
  item ativo:   --sidebar-primary (#0084D1 light / #00A6F4 dark)
  hover:        --sidebar-accent
```

**Nota:** As telas do Webchat (T-001 a T-014) não usam sidebar própria do Repasse AI — são sobrepostas ao layout da plataforma Cessionário como FAB/overlay/painel lateral. Sidebar é aplicada em T-015 a T-022 (Meu Perfil e Admin).

### 5.3 Padrão de Layout por Tipo de Tela

| Tipo | Telas | Layout |
|---|---|---|
| FAB/overlay no Cessionário | T-001 a T-014 | FAB fixed + overlay/painel lateral/bolha inline |
| Meu Perfil (Cessionário) | T-015 a T-019 | AppLayout com sidebar + tabs de seção |
| Admin | T-020 a T-022 | AppLayout Admin com sidebar 256px + grid 12 col |
| Modais | T-016, T-017 | Modal centrado Desktop / Bottom sheet Mobile |

### 5.4 Padrões Transversais

| Padrão | Especificação |
|---|---|
| Tipografia | Inter Variable — headings e body |
| Radius cards | `--radius-xl` (19.6px) para modais; `--radius-lg` (14px) para cards |
| Animações | scale-in/out 150ms (badges); fade-in/out 200ms (overlays); slide 300ms (modais/drawers); stagger 100ms (listas) |
| Tooltips | Sempre em `Tooltip` shadcn — em `disabled` buttons para explicar por que estão desabilitados |
| Focus ring | `--ring` (`#A1A1A1` light / `#737373` dark) |
| Toasts | Posição: top-right, duração: 4s (padrão) / 5s (confirmações) |
| Spinners | **Proibidos** como estado de loading — usar skeleton sempre |
| Timestamps | `--muted-foreground`, 2px menor que o texto de mensagem |
| CTAs primários | `--primary` (#0069A8 light / #00598A dark) |
| Ações destrutivas | `--destructive` (#E7000B light / #FF6467 dark) |
| Textos de aviso | `--muted-foreground`, italic quando nota complementar |
| Breakpoints | Mobile < 768px / Tablet 768–1024px / Desktop > 1024px |

---

## 6. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop — Pipeline v6.1 | Criação. 22 wireframes ASCII para T-001 a T-022. Cobertura 100%. 4 módulos. Design tokens do Brand Theme Guide v1.0 aplicados. Estados obrigatórios em todas as telas. |

---

## 7. Backlog de Pendências

| Item | Módulo / Tela | Impacto | O que falta | Dono | Status |
|---|---|---|---|---|---|
| [DECISÃO AUTÔNOMA] Grid sidebar Admin — sem especificação de `max-width` do content area no Brand Guide | Módulo D (T-020 a T-022) | P2 | Brand Guide não define max-width explícito para o content area do Admin. Decisão aplicada: `max-w-screen-xl` (1280px) alinhado ao padrão Tailwind. Justificativa: consistente com os padrões mais comuns de dashboards SaaS em shadcn/ui. | Design | Aplicado |
| [DECISÃO AUTÔNOMA] Ícone do FAB — sem especificação do ícone exato no MT ou PRD | Módulo A (T-001) | P2 | Mapa de Telas menciona "ícone de chat" mas não define qual ícone do Lucide. Decisão aplicada: `MessageCircle` (Lucide) — ícone de chat mais reconhecível e com variante filled disponível. Alternativa descartada: `Bot` (menos familiar para o público alvo). | Design | Aplicado |
| [DECISÃO AUTÔNOMA] Ícone do WhatsApp em T-016 Step 3 — sem ativo de marca no Brand Guide | Módulo C (T-016) | P2 | Brand Guide não inclui ícone do WhatsApp. Decisão aplicada: usar SVG oficial do WhatsApp (verde #25D366) conforme brand guidelines do Meta. Alternativa descartada: ícone genérico de telefone (perde reconhecimento da marca). | Design | Aplicado |
| [DECISÃO AUTÔNOMA] Gráfico de linha em T-022 — biblioteca não especificada | Módulo D (T-022) | P2 | PRD e MT não especificam biblioteca de charts. Decisão aplicada: Recharts (padrão do shadcn/ui para charts) com token `--chart-3`. Alternativa descartada: Chart.js (não nativo do ecossistema shadcn). | Dev | Aplicado |

**Telas pendentes:** 0 de 22 — cobertura 100%.

---

*Wireframes v1.0 concluídos. Cobertura: 22/22 telas. Status: APROVADO. Próximo documento do pipeline: D12 — Modelo de Dados (ERD Schema).*
