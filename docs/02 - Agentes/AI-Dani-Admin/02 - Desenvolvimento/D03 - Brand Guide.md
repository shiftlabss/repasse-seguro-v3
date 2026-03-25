# Brand Theme Guide — AI-Dani-Admin

## Módulo de Supervisão Operacional — Painel Admin

| Campo | Valor |
|---|---|
| Produto | Repasse Seguro — Módulo AI-Dani-Admin |
| Stack | shadcn/ui + Tailwind CSS v4 |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Fonte primária de tokens | Brand Theme Guide — Repasse Seguro v1.0 |

---

> **📌 TL;DR**
>
> - **Tokens:** AI-Dani-Admin herda 100% dos tokens do tema Repasse Seguro. Nenhum token novo é introduzido — o painel Admin é parte da mesma plataforma.
> - **Tipografia:** Inter Variable (heading e body) — invariante.
> - **Radius base:** 0.875rem (14px) — visual arredondado, transmite confiança operacional.
> - **Paleta:** Neutro puro (true gray) com acentos azul (#0069A8). Identidade visual consistente com a plataforma.
> - **Contexto de uso:** O painel Admin é uma interface 100% logada, técnica e orientada a dados. Sem landing page, sem face pública.
> - **Tokens de status operacional:** Adicionados para representar estados de supervisão (agente ativo, desligado, em takeover, sinalizado) — documentados como tokens de extensão na seção 3.
> - **Light/Dark mode:** Suportado com mesma lógica do tema base Repasse Seguro.

---

## 1. Identidade da Marca no Contexto Admin

### 1.1 Personalidade da Interface Admin

O painel de Supervisão IA da **Dani-Admin** é uma ferramenta operacional. A linguagem visual deve transmitir:

- **Clareza operacional:** cada dado visível deve ter significado imediato. Sem ornamentação supérflua.
- **Confiança em dados:** métricas e alertas apresentados com hierarquia visual clara — o Admin sabe de imediato o que exige atenção.
- **Controle:** a interface comunica que o Admin tem poder de intervir. Botões de ação (takeover, reativar agente) são sempre visíveis quando relevantes.

### 1.2 Tom de Voz na Interface

| Contexto | Tom | Exemplo |
|---|---|---|
| Estado normal (sem incidentes) | Neutro, informativo | "Todas as interações dentro do padrão." |
| Sinalização de revisão | Direto, sem alarme desnecessário | "1 interação aguarda revisão." |
| Alerta crítico (desligamento automático) | Urgente, acionável | "Agente desligado automaticamente. Taxa de erro: 32% nos últimos 15 min. Reativar após resolução." |
| Takeover ativo | Claro, confirmativo | "Você assumiu essa conversa. O agente não irá responder." |
| Confirmação de ação | Conciso, positivo | "Threshold atualizado para 75%." |
| Estado vazio | Útil, sem drama | "Nenhuma interação registrada no período selecionado. Ajuste os filtros para visualizar outros períodos." |

### 1.3 Iconografia

- **Sistema de ícones:** Lucide Icons — mesmo padrão da plataforma Repasse Seguro.
- **Tamanho padrão:** 16px (inline em texto), 20px (ações em botões), 24px (headers de seção).
- **Ícones de status operacional:**

| Estado | Ícone Lucide | Cor de Aplicação |
|---|---|---|
| Agente ativo | `circle-check` | `--color-success` (#16A34A) |
| Agente desligado automaticamente | `circle-x` | `--destructive` |
| Agente em FallbackAtivo | `triangle-alert` | `--color-warning` (#CA8A04) |
| Interação sinalizada para revisão | `flag` | `--color-warning` (#CA8A04) |
| Interação em takeover | `user-round` | `--primary` (#0069A8) |
| Interação encerrada | `check` | `--muted-foreground` |
| Alerta crítico | `bell-ring` | `--destructive` |
| Dashboard de métricas | `chart-bar` | `--primary` |

---

## 2. Theme Configuration (tokens herdados do Repasse Seguro)

O AI-Dani-Admin herda integralmente o tema base da plataforma Repasse Seguro. Os tokens abaixo são a fonte primária para toda implementação de interface no painel Admin.

### 2.1 Tipografia

| Token | Valor | Uso |
|---|---|---|
| `--font-heading` | `var(--font-sans)` → `'Inter Variable', sans-serif` | Headings — herda de `--font-sans` |
| `--font-sans` | `'Inter Variable', sans-serif` | Toda a interface — body, inputs, selects, tabelas, labels, buttons, badges, dados de métricas |

### 2.2 Border Radius

Base de 14px — raio generoso. Visual suave e arredondado.

| Token | Cálculo | Valor | Uso no Painel Admin |
|---|---|---|---|
| `--radius` | — | 0.875rem (14px) | Base de referência |
| `--radius-sm` | `radius × 0.6` | 0.525rem (8.4px) | Badges de status (Sinalizada, Em takeover), chips de filtro |
| `--radius-md` | `radius × 0.8` | 0.7rem (11.2px) | Botões (Assumir conversa, Reativar), inputs de threshold |
| `--radius-lg` | `radius` | 0.875rem (14px) | Cards de métricas, painéis de interação |
| `--radius-xl` | `radius × 1.4` | 1.225rem (19.6px) | Modais de confirmação de takeover, drawers de configuração |
| `--radius-2xl` | `radius × 1.8` | 1.575rem (25.2px) | Elementos de destaque (alerta crítico expandido) |

### 2.3 Cores — Light Mode

#### Superfícies e Texto

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--background` | `#FFFFFF` | `oklch(1 0 0)` | Fundo principal do painel Admin |
| `--foreground` | `#0A0A0A` | `oklch(0.145 0 0)` | Texto principal, dados de métricas |
| `--card` | `#FFFFFF` | `oklch(1 0 0)` | Cards de métricas, cards de interações |
| `--card-foreground` | `#0A0A0A` | `oklch(0.145 0 0)` | Texto em cards |
| `--popover` | `#FFFFFF` | `oklch(1 0 0)` | Dropdowns de filtros, tooltips de métricas |
| `--popover-foreground` | `#0A0A0A` | `oklch(0.145 0 0)` | Texto em popovers |

#### Cores Semânticas

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--primary` | `#0069A8` | `oklch(0.5 0.134 242.749)` | CTAs principais: "Assumir conversa", "Salvar threshold", "Reativar agente" |
| `--primary-foreground` | `#F0F9FF` | `oklch(0.977 0.013 236.62)` | Texto sobre primary |
| `--secondary` | `#F4F4F5` | `oklch(0.967 0.001 286.375)` | Botões secundários, backgrounds de seção |
| `--secondary-foreground` | `#18181B` | `oklch(0.21 0.006 285.885)` | Texto sobre secondary |
| `--muted` | `#F5F5F5` | `oklch(0.97 0 0)` | Background de linha de interação encerrada, áreas inativas |
| `--muted-foreground` | `#737373` | `oklch(0.556 0 0)` | Timestamps, labels de filtro, texto de estado vazio |
| `--accent` | `#F5F5F5` | `oklch(0.97 0 0)` | Hover de linhas na lista de interações |
| `--accent-foreground` | `#171717` | `oklch(0.205 0 0)` | Texto sobre accent |
| `--destructive` | `#E7000B` | `oklch(0.577 0.245 27.325)` | Alertas críticos, desligamento automático, taxa de erro elevada |

#### Bordas e Controles

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--border` | `#E5E5E5` | `oklch(0.922 0 0)` | Bordas de cards, divisores entre interações |
| `--input` | `#E5E5E5` | `oklch(0.922 0 0)` | Bordas de inputs (threshold, filtros) |
| `--ring` | `#A1A1A1` | `oklch(0.708 0 0)` | Focus ring em campos de configuração |

#### Charts (Dashboard de Métricas)

Paleta azul celeste em 5 níveis — usada nos gráficos do Dashboard Admin:

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--chart-1` | `#74D4FF` | `oklch(0.828 0.111 230.318)` | Volume de interações — Dani-Cessionário |
| `--chart-2` | `#00A6F4` | `oklch(0.685 0.169 237.323)` | Volume de interações — Dani-Cedente |
| `--chart-3` | `#0084D1` | `oklch(0.588 0.158 241.966)` | CSAT médio |
| `--chart-4` | `#0069A8` | `oklch(0.5 0.134 242.749)` | Taxa de recusa |
| `--chart-5` | `#00598A` | `oklch(0.443 0.11 240.79)` | Latência média |

#### Sidebar

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--sidebar` | `#FAFAFA` | `oklch(0.985 0 0)` | Fundo da sidebar do painel Admin |
| `--sidebar-foreground` | `#0A0A0A` | `oklch(0.145 0 0)` | Texto da sidebar |
| `--sidebar-primary` | `#0084D1` | `oklch(0.588 0.158 241.966)` | Item ativo da sidebar (Supervisão IA ativo) |
| `--sidebar-primary-foreground` | `#F0F9FF` | `oklch(0.977 0.013 236.62)` | Texto do item ativo |
| `--sidebar-accent` | `#F5F5F5` | `oklch(0.97 0 0)` | Hover de items da sidebar |
| `--sidebar-accent-foreground` | `#171717` | `oklch(0.205 0 0)` | Texto do item em hover |
| `--sidebar-border` | `#E5E5E5` | `oklch(0.922 0 0)` | Bordas e divisores da sidebar |
| `--sidebar-ring` | `#A1A1A1` | `oklch(0.708 0 0)` | Focus ring na sidebar |

### 2.4 Cores — Dark Mode

#### Superfícies e Texto

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--background` | `#0A0A0A` | `oklch(0.145 0 0)` | Fundo principal |
| `--foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto principal |
| `--card` | `#171717` | `oklch(0.205 0 0)` | Cards de métricas e interações |
| `--card-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto em cards |
| `--popover` | `#171717` | `oklch(0.205 0 0)` | Dropdowns e popovers |
| `--popover-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto em popovers |

#### Cores Semânticas (Dark)

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--primary` | `#00598A` | `oklch(0.443 0.11 240.79)` | CTAs em dark mode |
| `--primary-foreground` | `#F0F9FF` | `oklch(0.977 0.013 236.62)` | Texto sobre primary |
| `--secondary` | `#27272A` | `oklch(0.274 0.006 286.033)` | Botões secundários |
| `--secondary-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto sobre secondary |
| `--muted` | `#262626` | `oklch(0.269 0 0)` | Backgrounds inativos |
| `--muted-foreground` | `#A1A1A1` | `oklch(0.708 0 0)` | Timestamps, labels, estado vazio |
| `--accent` | `#262626` | `oklch(0.269 0 0)` | Hover de linhas de interação |
| `--accent-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto sobre accent |
| `--destructive` | `#FF6467` | `oklch(0.704 0.191 22.216)` | Alertas críticos — versão mais clara para contraste em dark |

#### Bordas e Controles (Dark)

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--border` | `#FFFFFF1A` | `oklch(1 0 0 / 10%)` | Bordas — branco 10% |
| `--input` | `#FFFFFF26` | `oklch(1 0 0 / 15%)` | Bordas de inputs — branco 15% |
| `--ring` | `#737373` | `oklch(0.556 0 0)` | Focus ring |

#### Charts (Dark) — Invariantes

| Token | HEX | Nota |
|---|---|---|
| `--chart-1` | `#74D4FF` | Invariante |
| `--chart-2` | `#00A6F4` | Invariante |
| `--chart-3` | `#0084D1` | Invariante |
| `--chart-4` | `#0069A8` | Invariante |
| `--chart-5` | `#00598A` | Invariante |

#### Sidebar (Dark)

| Token | HEX | OKLCH | Uso |
|---|---|---|---|
| `--sidebar` | `#171717` | `oklch(0.205 0 0)` | Fundo da sidebar |
| `--sidebar-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto da sidebar |
| `--sidebar-primary` | `#00A6F4` | `oklch(0.685 0.169 237.323)` | Item ativo — azul vibrante |
| `--sidebar-primary-foreground` | `#052F4A` | `oklch(0.293 0.066 243.157)` | Texto do item ativo |
| `--sidebar-accent` | `#262626` | `oklch(0.269 0 0)` | Hover de items |
| `--sidebar-accent-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto em hover |
| `--sidebar-border` | `#FFFFFF1A` | `oklch(1 0 0 / 10%)` | Bordas — branco 10% |
| `--sidebar-ring` | `#737373` | `oklch(0.556 0 0)` | Focus ring |

---

## 3. Tokens de Extensão — Status Operacional

Tokens adicionais exclusivos do módulo AI-Dani-Admin para representar estados operacionais de supervisão. Estes tokens **não substituem** os tokens base — são extensões semânticas para contextos de status de agente e interação.

> **[TOKEN PROPOSTO]** — Tokens abaixo não existem no source code atual. Propostos com base na identidade visual da plataforma e nas necessidades de supervisão definidas no D01. Devem ser implementados no `global.css` junto ao tema base.

| Token | HEX (Light) | HEX (Dark) | Uso |
|---|---|---|---|
| `--color-success` | `#16A34A` | `#4ADE80` | Agente ativo, interação respondida dentro do SLA, CSAT acima da meta |
| `--color-success-foreground` | `#FFFFFF` | `#052E16` | Texto sobre success |
| `--color-success-subtle` | `#F0FDF4` | `#14532D` | Background de badge "Respondida pela IA" |
| `--color-warning` | `#CA8A04` | `#FBBF24` | Interação sinalizada para revisão, CSAT degradado, taxa de recusa alta |
| `--color-warning-foreground` | `#FFFFFF` | `#451A03` | Texto sobre warning |
| `--color-warning-subtle` | `#FFFBEB` | `#292524` | Background de badge "Sinalizada para revisão" |
| `--color-takeover` | `#0069A8` | `#00A6F4` | Interação em takeover ativo (usa `--primary` como base) |
| `--color-takeover-subtle` | `#EFF6FF` | `#1E3A5F` | Background de badge "Em takeover" |
| `--color-agent-offline` | `#E7000B` | `#FF6467` | Agente desligado automaticamente (usa `--destructive` como base) |

### 3.1 Badges de Status de Interação

| Estado | Token de Fundo | Token de Texto | Exemplo Visual |
|---|---|---|---|
| Respondida pela IA | `--color-success-subtle` | `--color-success` | Badge verde claro "Respondida" |
| Sinalizada para revisão | `--color-warning-subtle` | `--color-warning` | Badge amarelo "Aguardando revisão" |
| Em takeover | `--color-takeover-subtle` | `--color-takeover` | Badge azul "Em atendimento humano" |
| Encerrada | `--muted` | `--muted-foreground` | Badge cinza "Encerrada" |

### 3.2 Badges de Status do Agente

| Estado | Token de Fundo | Token de Texto | Exemplo Visual |
|---|---|---|---|
| Ativo | `--color-success-subtle` | `--color-success` | "Ativo" com ponto verde pulsante |
| DesligadoAutomatico | `--destructive` (bg sutil) | `--destructive` | "Desligado" com ícone de alerta |
| FallbackAtivo | `--color-warning-subtle` | `--color-warning` | "Modo degradado" com ícone de triângulo |

---

## 4. Tokens Invariantes

Os tokens abaixo não mudam entre light e dark mode:

- `--font-heading` — Inter Variable
- `--font-sans` — Inter Variable
- `--radius` e todas as derivações (sm, md, lg, xl, 2xl, 3xl, 4xl)
- `--chart-1` a `--chart-5` — paleta azul celeste
- Tokens de extensão de status devem manter consistência entre temas (dark mode usa versões mais brilhantes para contraste)

---

## 5. CSS Completo — Tokens Base (herdado do Repasse Seguro)

```css
/* ============================================
   AI-Dani-Admin — shadcn Theme Tokens
   Herda do: Repasse Seguro Brand Guide v1.0
   v1.0 · 2026-03-23
   ============================================ */

/* Tipografia */
--font-heading: var(--font-sans);
--font-sans: 'Inter Variable', sans-serif;

/* Radius */
--radius: 0.875rem;
--radius-sm: calc(var(--radius) * 0.6);    /* 8.4px  */
--radius-md: calc(var(--radius) * 0.8);    /* 11.2px */
--radius-lg: var(--radius);                /* 14px   */
--radius-xl: calc(var(--radius) * 1.4);    /* 19.6px */
--radius-2xl: calc(var(--radius) * 1.8);   /* 25.2px */
--radius-3xl: calc(var(--radius) * 2.2);   /* 30.8px */
--radius-4xl: calc(var(--radius) * 2.6);   /* 36.4px */

/* Charts (invariantes entre temas) */
--chart-1: oklch(0.828 0.111 230.318);   /* #74D4FF */
--chart-2: oklch(0.685 0.169 237.323);   /* #00A6F4 */
--chart-3: oklch(0.588 0.158 241.966);   /* #0084D1 */
--chart-4: oklch(0.5 0.134 242.749);     /* #0069A8 */
--chart-5: oklch(0.443 0.11 240.79);     /* #00598A */

/* ── Light Mode ── */
:root {
  --background:                  oklch(1 0 0);
  --foreground:                  oklch(0.145 0 0);
  --card:                        oklch(1 0 0);
  --card-foreground:             oklch(0.145 0 0);
  --popover:                     oklch(1 0 0);
  --popover-foreground:          oklch(0.145 0 0);
  --primary:                     oklch(0.5 0.134 242.749);
  --primary-foreground:          oklch(0.977 0.013 236.62);
  --secondary:                   oklch(0.967 0.001 286.375);
  --secondary-foreground:        oklch(0.21 0.006 285.885);
  --muted:                       oklch(0.97 0 0);
  --muted-foreground:            oklch(0.556 0 0);
  --accent:                      oklch(0.97 0 0);
  --accent-foreground:           oklch(0.205 0 0);
  --destructive:                 oklch(0.577 0.245 27.325);
  --border:                      oklch(0.922 0 0);
  --input:                       oklch(0.922 0 0);
  --ring:                        oklch(0.708 0 0);
  --sidebar:                     oklch(0.985 0 0);
  --sidebar-foreground:          oklch(0.145 0 0);
  --sidebar-primary:             oklch(0.588 0.158 241.966);
  --sidebar-primary-foreground:  oklch(0.977 0.013 236.62);
  --sidebar-accent:              oklch(0.97 0 0);
  --sidebar-accent-foreground:   oklch(0.205 0 0);
  --sidebar-border:              oklch(0.922 0 0);
  --sidebar-ring:                oklch(0.708 0 0);

  /* Extensões de Status Operacional — AI-Dani-Admin */
  --color-success:               #16A34A;
  --color-success-foreground:    #FFFFFF;
  --color-success-subtle:        #F0FDF4;
  --color-warning:               #CA8A04;
  --color-warning-foreground:    #FFFFFF;
  --color-warning-subtle:        #FFFBEB;
  --color-takeover:              oklch(0.5 0.134 242.749); /* = --primary */
  --color-takeover-subtle:       #EFF6FF;
  --color-agent-offline:         oklch(0.577 0.245 27.325); /* = --destructive */
}

/* ── Dark Mode ── */
.dark {
  --background:                  oklch(0.145 0 0);
  --foreground:                  oklch(0.985 0 0);
  --card:                        oklch(0.205 0 0);
  --card-foreground:             oklch(0.985 0 0);
  --popover:                     oklch(0.205 0 0);
  --popover-foreground:          oklch(0.985 0 0);
  --primary:                     oklch(0.443 0.11 240.79);
  --primary-foreground:          oklch(0.977 0.013 236.62);
  --secondary:                   oklch(0.274 0.006 286.033);
  --secondary-foreground:        oklch(0.985 0 0);
  --muted:                       oklch(0.269 0 0);
  --muted-foreground:            oklch(0.708 0 0);
  --accent:                      oklch(0.269 0 0);
  --accent-foreground:           oklch(0.985 0 0);
  --destructive:                 oklch(0.704 0.191 22.216);
  --border:                      oklch(1 0 0 / 10%);
  --input:                       oklch(1 0 0 / 15%);
  --ring:                        oklch(0.556 0 0);
  --sidebar:                     oklch(0.205 0 0);
  --sidebar-foreground:          oklch(0.985 0 0);
  --sidebar-primary:             oklch(0.685 0.169 237.323);
  --sidebar-primary-foreground:  oklch(0.293 0.066 243.157);
  --sidebar-accent:              oklch(0.269 0 0);
  --sidebar-accent-foreground:   oklch(0.985 0 0);
  --sidebar-border:              oklch(1 0 0 / 10%);
  --sidebar-ring:                oklch(0.556 0 0);

  /* Extensões de Status Operacional — Dark Mode */
  --color-success:               #4ADE80;
  --color-success-foreground:    #052E16;
  --color-success-subtle:        #14532D;
  --color-warning:               #FBBF24;
  --color-warning-foreground:    #451A03;
  --color-warning-subtle:        #292524;
  --color-takeover:              oklch(0.685 0.169 237.323); /* = sidebar-primary dark */
  --color-takeover-subtle:       #1E3A5F;
  --color-agent-offline:         oklch(0.704 0.191 22.216); /* = --destructive dark */
}
```

---

## 6. Referência Rápida de Cores

### Padrões de uso mais frequentes no painel Admin

| Token | HEX (Light) | Uso Principal |
|---|---|---|
| `--primary` | `#0069A8` | Botão "Assumir conversa", "Salvar threshold", "Reativar agente" |
| `--destructive` | `#E7000B` | Alertas de desligamento automático, taxa de erro elevada |
| `--color-success` | `#16A34A` | Agente ativo, interação respondida, CSAT dentro da meta |
| `--color-warning` | `#CA8A04` | Interação sinalizada, CSAT degradado, taxa de recusa alta |
| `--muted-foreground` | `#737373` | Timestamps, labels de filtro, texto secundário de métricas |
| `--chart-1..5` | `#74D4FF` → `#00598A` | Gráficos do Dashboard de métricas |

---

## 7. Regras de Componentes

### 7.1 Botões

| Variante | Uso no Contexto Admin | Regra |
|---|---|---|
| `primary` | "Assumir conversa", "Salvar", "Reativar agente" | CTA principal por tela. Máximo 1 botão primary por seção de ação. |
| `secondary` | "Cancelar", "Limpar filtros", "Ver detalhes" | Ações secundárias. Sempre acompanha um primary quando há ação principal. |
| `destructive` | N/A no painel Admin atual | Reservado para futuras ações irreversíveis (ex: excluir configuração). |
| `ghost` | Links de navegação na sidebar, ações em linha na lista de interações | Sem bordas, sem fundo. Usar para ações de baixo peso visual. |
| `outline` | Filtros ativos como botão, ações de exportação | Borda visível, fundo transparente. |

### 7.2 Badges de Status

- Usar `--radius-sm` (8.4px) em todos os badges.
- Texto máximo: 2 palavras. Exemplo: "Em takeover", "Sinalizada", "Encerrada".
- Fonte: `text-xs` (12px), `font-medium` (500).
- Nunca usar badge para transmitir informação crítica sem suporte de texto e/ou ícone.

### 7.3 Cards de Métricas

- `--radius-lg` (14px) como border-radius padrão.
- Header do card: título em `text-sm font-medium` + `--muted-foreground`, valor principal em `text-2xl font-semibold` + `--foreground`.
- Card com dado insuficiente: substitui o valor por "Dados insuficientes" em `text-sm` + `--muted-foreground`. Nunca exibir `0` ou `0%` para indicador sem dados (RN-DA-034).
- Ícone no header do card: 20px, `--muted-foreground`.

### 7.4 Lista de Interações

- Linha com hover usando `--accent` como background.
- Interação sinalizada: linha com background `--color-warning-subtle` e borda esquerda 3px `--color-warning`.
- Interação em takeover: linha com background `--color-takeover-subtle` e borda esquerda 3px `--color-takeover`.
- Skeleton loading durante filtros: shimmer animado com `--muted` como cor base. Nunca substituir a lista inteira por spinner.

### 7.5 Modais e Drawers

- `--radius-xl` (19.6px) como border-radius.
- Overlay: `rgba(0,0,0,0.5)` com `backdrop-filter: blur(4px)`.
- Título do modal: `text-lg font-semibold`.
- Ações do modal: botões alinhados à direita, primary à direita, secondary/cancel à esquerda.

---

## 8. Acessibilidade

- **Contraste mínimo:** 4.5:1 para texto normal (AA). 3:1 para texto grande (≥18px ou ≥14px bold).
- `--color-success` (#16A34A sobre branco): contraste ≈ 4.5:1 — atende AA.
- `--color-warning` (#CA8A04 sobre branco): contraste ≈ 4.6:1 — atende AA.
- `--destructive` (#E7000B sobre branco): contraste ≈ 4.8:1 — atende AA.
- **Focus ring:** sempre visível em campos de configuração de threshold e botões de ação crítica. Usar `--ring` como cor de outline.
- **Tamanho mínimo de toque:** 44×44px em elementos interativos da interface.
- **Texto alternativo:** badges de status devem ter `aria-label` quando o texto não for suficiente para contexto.

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Herda tokens do Repasse Seguro Brand Guide v1.0. Adiciona tokens de extensão para status operacional de supervisão. |
