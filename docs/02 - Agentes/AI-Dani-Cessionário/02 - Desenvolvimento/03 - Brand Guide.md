# 03 - Brand Guide

| **Produto** | AI-Dani-Cessionário (módulo do Repasse Seguro) |
|---|---|
| **Destinatário** | Designers e frontend engineers |
| **Escopo** | Brand Theme Guide do agente AI-Dani-Cessionário — herda o tema Repasse Seguro e define extensões específicas para os componentes do agente |
| **Stack** | shadcn/ui + Tailwind CSS v4 |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - O AI-Dani-Cessionário **herda integralmente** o tema base do Repasse Seguro (tokens shadcn, tipografia Inter Variable, radius 0.875rem, paleta azul `#0069A8`).
> - **Extensões específicas da Dani:** tokens de score de risco (verde/amarelo/vermelho), tokens de status do agente (operacional/latência/fallback/desligado), chat window, FAB e bubble de mensagem.
> - **Tokens propostos** (sem source code ainda): marcados como `[TOKEN PROPOSTO]`.
> - Sem dark mode diferenciado para os tokens de extensão — o tema do agente segue o dark mode padrão do Repasse Seguro.
> - Nenhum componente custom fora dos listados nesta seção — todo o restante usa shadcn/ui sem modificação.

---

## 1. Herança do Tema Repasse Seguro

O AI-Dani-Cessionário é um módulo da plataforma Repasse Seguro. **Todos os tokens base são herdados** sem sobrescrição. A fonte normativa dos tokens base é o Brand Guide do Repasse Seguro (Repasse AI D03).

### 1.1 Tipografia (herdada)

| Token | Valor | Uso |
|---|---|---|
| `--font-heading` | `var(--font-sans)` → `'Inter Variable', sans-serif` | Headings na interface da Dani |
| `--font-sans` | `'Inter Variable', sans-serif` | Toda a interface — body, inputs, labels, mensagens |

### 1.2 Border Radius (herdado)

| Token | Valor | Uso |
|---|---|---|
| `--radius` | `0.875rem` (14px) | Base |
| `--radius-sm` | `0.525rem` (8.4px) | Badges, chips de OPR |
| `--radius-md` | `0.7rem` (11.2px) | Botões de ação rápida, inputs do chat |
| `--radius-lg` | `0.875rem` (14px) | Cards de oportunidade, chat window |
| `--radius-xl` | `1.225rem` (19.6px) | Chat window expandida, modais de análise |
| `--radius-2xl` | `1.575rem` (25.2px) | Bubble de mensagem do usuário |

### 1.3 Paleta Base (herdada — Light Mode)

| Token | HEX | Uso |
|---|---|---|
| `--background` | `#FFFFFF` | Fundo da chat window |
| `--foreground` | `#0A0A0A` | Texto nas mensagens |
| `--primary` | `#0069A8` | Botões CTA da Dani, badge do FAB |
| `--muted` | `#F5F5F5` | Background das mensagens da Dani |
| `--muted-foreground` | `#737373` | Timestamps, metadados |
| `--border` | `#E5E5E5` | Bordas da chat window, separadores |
| `--destructive` | `#E7000B` | Alertas de risco alto, erros críticos |

### 1.4 Paleta Base (herdada — Dark Mode)

Mesmos tokens semânticos com valores dark mode do Repasse Seguro. Nenhuma sobrescrição.

---

## 2. Extensões Específicas da Dani

### 2.1 Tokens de Score de Risco

Conforme RN-DC-011 e RN-DC-012: score de risco em escala 1–10 com indicadores visuais verde/amarelo/vermelho. Contraste mínimo 4.5:1 obrigatório.

| Token | HEX (Light) | HEX (Dark) | Faixa | Uso |
|---|---|---|---|---|
| `--risk-low` | `#16A34A` [TOKEN PROPOSTO] | `#4ADE80` [TOKEN PROPOSTO] | 1–3 | Score baixo — texto e borda do indicador |
| `--risk-low-bg` | `#F0FDF4` [TOKEN PROPOSTO] | `#14532D` [TOKEN PROPOSTO] | 1–3 | Background do badge de risco baixo |
| `--risk-medium` | `#D97706` [TOKEN PROPOSTO] | `#FCD34D` [TOKEN PROPOSTO] | 4–6 | Score moderado |
| `--risk-medium-bg` | `#FFFBEB` [TOKEN PROPOSTO] | `#78350F` [TOKEN PROPOSTO] | 4–6 | Background do badge de risco moderado |
| `--risk-high` | `#DC2626` [TOKEN PROPOSTO] | `#FF6467` (herdado de `--destructive` dark) | 7–10 | Score alto |
| `--risk-high-bg` | `#FEF2F2` [TOKEN PROPOSTO] | `#450A0A` [TOKEN PROPOSTO] | 7–10 | Background do badge de risco alto |

> ⚙️ **Regra de acessibilidade:** Todo indicador de risco usa cor + rótulo textual. Nunca apenas cor. Exemplo: badge "Risco baixo (3/10)" com `aria-label="Score de risco: 3 de 10 — risco baixo"`.

### 2.2 Tokens de Status do Agente

Conforme RN-DC-023 e RN-DC-024: estados do agente Dani exibidos no banner de fallback.

| Token | HEX (Light) | HEX (Dark) | Estado | Uso |
|---|---|---|---|---|
| `--agent-operational` | `#16A34A` [TOKEN PROPOSTO] | `#4ADE80` [TOKEN PROPOSTO] | Operacional | Indicador de status na interface do Admin |
| `--agent-latency` | `#D97706` [TOKEN PROPOSTO] | `#FCD34D` [TOKEN PROPOSTO] | Latência alta | Banner de aviso de lentidão |
| `--agent-fallback` | `#0069A8` (herdado de `--primary`) | `#00A6F4` (variante clara de `--primary` para dark mode) | Modo fallback | Banner "Modo básico — sem análise da IA" |
| `--agent-offline` | `#737373` (herdado de `--muted-foreground`) | `#A1A1A1` [TOKEN PROPOSTO] | Desligado | Banner neutro de indisponibilidade |

### 2.3 Tokens de Negociação

Conforme RN-DC-011: badges de status de oportunidade.

| Token | HEX (Light) | Uso |
|---|---|---|
| `--status-available` | `#16A34A` [TOKEN PROPOSTO] | Badge "Disponível" |
| `--status-negotiating` | `#D97706` [TOKEN PROPOSTO] | Badge "Em negociação" (cor laranja — RN-DC-011) |
| `--status-closed` | `#737373` (herdado de `--muted-foreground`) | Badge "Encerrada" (cor cinza — RN-DC-011) |

### 2.4 Tokens de Comparação de ROI

Conforme RN-DC-018: diferença entre proposta e contraproposta.

| Token | HEX | Uso |
|---|---|---|
| `--roi-positive` | `#16A34A` [TOKEN PROPOSTO] | Seta para baixo em verde — economia |
| `--roi-negative` | `#DC2626` [TOKEN PROPOSTO] | Seta para cima em vermelho — acréscimo |

---

## 3. Componentes Específicos da Dani

> ⚙️ **Regra:** Todos os componentes abaixo são custom. Documentar variantes, tokens e requisitos de acessibilidade obrigatórios.

### 3.1 FAB (Floating Action Button) da Dani

| Atributo | Valor |
|---|---|
| Tamanho | 56×56px |
| Radius | `--radius-xl` (19.6px) — circular |
| Cor de fundo | `--primary` (`#0069A8` light / `#00598A` dark) |
| Ícone | Ícone da Dani (avatar estilizado) em `--primary-foreground` |
| Posição | Fixed, bottom-right, `bottom: 24px; right: 24px` |
| Z-index | `9999` |
| Badge | Número de alertas não lidos. Cor: `--destructive`. Posição: top-right do FAB |
| Animação entrada | scale 0→1 + fade-in, 200ms, `ease-out` |
| Animação badge | scale + bounce, 200ms (Framer Motion) |
| aria-label | `"Abrir chat com Dani — X alertas não lidos"` |

### 3.2 Chat Window

| Atributo | Valor |
|---|---|
| Largura | 400px (desktop), 100vw (mobile ≤ 640px) |
| Altura | 600px (desktop), 100dvh (mobile) |
| Radius | `--radius-xl` (19.6px) nos cantos superiores. Mobile: sem radius |
| Fundo | `--background` |
| Borda | `1px solid --border` |
| Shadow | `0 8px 32px oklch(0 0 0 / 16%)` |
| Animação abertura | slide-up (translateY: 20px→0) + fade-in, 250ms, `ease-out` (Framer Motion AnimatePresence) |
| Header | Altura 56px. Fundo `--primary`. Avatar da Dani + "Dani · Analista de Oportunidades" em `--primary-foreground` |
| Input bar | Altura 56px. Input `--radius-md`. Botão enviar `--primary` |

### 3.3 Bubble de Mensagem

| Tipo | Fundo | Texto | Radius | Alinhamento |
|---|---|---|---|---|
| Mensagem da Dani | `--muted` | `--foreground` | `--radius-lg` (exceto canto inferior esquerdo = 4px) | Esquerda |
| Mensagem do Cessionário | `--primary` | `--primary-foreground` | `--radius-lg` (exceto canto inferior direito = 4px) | Direita |
| Mensagem de sistema (aviso, recusa) | `oklch(0.977 0.013 236.62 / 12%)` | `--muted-foreground` | `--radius-md` | Centralizado |
| Aviso disclaimer ROI | Fundo `--muted`, texto `--muted-foreground` | Corpo de texto menor (`text-xs`) + ícone info (i) | `--radius-sm` | Esquerda |

### 3.4 Card de Oportunidade (inline no chat)

| Atributo | Valor |
|---|---|
| Radius | `--radius-lg` |
| Borda | `1px solid --border` |
| Fundo | `--card` |
| Badge OPR | Chip com radius `--radius-sm`, fundo `--muted`, texto `--muted-foreground` |
| Badge status | Conforme tokens seção 2.3 |
| Badge score de risco | Conforme tokens seção 2.1 |
| Coluna de ranqueamento ativa | `font-weight: 600` + `background: oklch(0.5 0.134 242.749 / 8%)` |

### 3.5 Tabela Comparativa de Oportunidades

| Atributo | Valor |
|---|---|
| Linha "Melhor opção" | Badge "Melhor opção" em `--primary` + `background: oklch(0.5 0.134 242.749 / 6%)` |
| Linha clicável | `cursor: pointer`, hover `--accent` |
| role | `role="table"` com cabeçalhos `scope="col"` |

### 3.6 Botões de Ação Rápida (Quick Action Chips)

| Atributo | Valor |
|---|---|
| Variante | shadcn `Button` variant `outline` |
| Radius | `--radius-md` |
| Tamanho | `size="sm"` |
| Fundo hover | `--accent` |
| Disposição | Horizontal, com wrap em mobile |

### 3.7 Banner de Fallback da Calculadora

Conforme RN-DC-023 e RN-DC-024.

| Estado | Token de cor | Texto | Ícone |
|---|---|---|---|
| Modo básico (fallback ativo) | `--agent-fallback` | "Modo básico — sem análise da IA" | Ícone de calculadora |
| Agente desligado | `--agent-offline` | "Analista de Oportunidades temporariamente indisponível" | Ícone de pausa |

### 3.8 Indicador de Digitando (Typing Indicator)

Conforme RN-DC-029.

| Atributo | Valor |
|---|---|
| Visual | Três pontos pulsando (animação `pulse`, Framer Motion) |
| Cor | `--muted-foreground` |
| Posição | Bubble esquerda (igual mensagem da Dani) |
| Duração de visibilidade | Após SLA atingido — enquanto resposta não entregue |

### 3.9 Contador Regressivo de Rate Limit

Conforme RN-DC-025.

| Atributo | Valor |
|---|---|
| Campo de entrada desabilitado | Fundo `--muted`, cursor `not-allowed` |
| Botão enviar | Desabilitado, `opacity: 0.4` |
| Contador | Formato `mm:ss`, cor `--muted-foreground`, atualizado a cada segundo |
| Reativação | `pulse` sutil 500ms via Framer Motion |

---

## 4. Tom de Voz da Interface

Conforme D01 seção 2.

| Aspecto | Regra |
|---|---|
| Mensagens da Dani | Analítico, objetivo, orientado a dados. Frases curtas e diretas |
| Labels de ação | Imperativo direto: "Simular outro valor", "Ir para a oportunidade", "Ativar alertas" |
| Mensagens de recusa | Neutras, nunca ameaçadoras. Sempre com alternativa (exceto na 3ª insistência) |
| Mensagens de erro | Específicas, com instrução clara. Nunca genéricas |
| Aviso de projeção | "Esses são valores projetados com base nos dados disponíveis. Resultados reais podem variar." — obrigatório e nunca ocultável |
| Valores monetários | Sempre com separador de milhar e 2 casas decimais: `R$ 30.000,00` |

---

## 5. CSS Completo — Extensões da Dani

Tokens adicionais a inserir no arquivo de tema do projeto (após os tokens base do Repasse Seguro):

```css
/* ============================================
   AI-Dani-Cessionário — Extensões de Tema
   v1.0 · 23/03/2026
   [TOKENS PROPOSTOS — confirmar com design]
   ============================================ */

:root {
  /* Score de Risco */
  --risk-low:              #16A34A;
  --risk-low-bg:           #F0FDF4;
  --risk-medium:           #D97706;
  --risk-medium-bg:        #FFFBEB;
  --risk-high:             #DC2626;
  --risk-high-bg:          #FEF2F2;

  /* Status do Agente */
  --agent-operational:     #16A34A;
  --agent-latency:         #D97706;
  --agent-fallback:        #0069A8; /* = --primary */
  --agent-offline:         #737373; /* = --muted-foreground */

  /* Status de Oportunidade */
  --status-available:      #16A34A;
  --status-negotiating:    #D97706;
  --status-closed:         #737373;

  /* ROI Comparativo */
  --roi-positive:          #16A34A;
  --roi-negative:          #DC2626;
}

.dark {
  /* Score de Risco */
  --risk-low:              #4ADE80;
  --risk-low-bg:           #14532D;
  --risk-medium:           #FCD34D;
  --risk-medium-bg:        #78350F;
  --risk-high:             #FF6467; /* = --destructive dark */
  --risk-high-bg:          #450A0A;

  /* Status do Agente */
  --agent-operational:     #4ADE80;
  --agent-latency:         #FCD34D;
  --agent-fallback:        #00A6F4;
  --agent-offline:         #A1A1A1;

  /* Status de Oportunidade */
  --status-available:      #4ADE80;
  --status-negotiating:    #FCD34D;
  --status-closed:         #A1A1A1;

  /* ROI Comparativo */
  --roi-positive:          #4ADE80;
  --roi-negative:          #FF6467;
}
```

---

## 6. Tokens Invariantes

- `--font-heading`, `--font-sans` — Inter Variable (herdado)
- `--radius` e derivações — herdado do Repasse Seguro
- `--chart-1` a `--chart-5` — paleta azul celeste (herdado)
- Gráfico de valorização do empreendimento usa `--chart-3` (`#0084D1`) como cor principal

---

## 7. Referência Rápida — Tokens da Dani

| Decisão visual | Token | Valor (Light) |
|---|---|---|
| CTA principal da Dani | `--primary` | `#0069A8` |
| Background mensagem da Dani | `--muted` | `#F5F5F5` |
| Score baixo (1–3) | `--risk-low` | `#16A34A` |
| Score médio (4–6) | `--risk-medium` | `#D97706` |
| Score alto (7–10) | `--risk-high` | `#DC2626` |
| Badge "Em negociação" | `--status-negotiating` | `#D97706` |
| Badge "Encerrada" | `--status-closed` | `#737373` |
| ROI positivo (economia) | `--roi-positive` | `#16A34A` |
| ROI negativo (acréscimo) | `--roi-negative` | `#DC2626` |
| Fallback mode banner | `--agent-fallback` | `#0069A8` |

---

## Changelog

| Versão | Data | Descrição |
|---|---|---|
| v1.0 | 23/03/2026 | Versão inicial. Herança completa do tema Repasse Seguro. Extensões para score de risco, status do agente, tokens de negociação, componentes custom do chat. Tokens propostos marcados — aguardam confirmação do design. |
