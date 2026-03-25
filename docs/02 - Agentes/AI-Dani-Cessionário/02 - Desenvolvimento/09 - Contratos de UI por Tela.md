# 09 - Contratos de UI por Tela

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Frontend Lead / UI Designer | Contratos de UI por tela do agente AI-Dani-Cessionário — componentes shadcn/ui, variantes, tokens e estados obrigatórios | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **12 telas/componentes cobertos** (T-DC-001 a T-DC-012) — 100% do inventário do doc 06.
> - **Padrão obrigatório: 4 estados por tela** — Skeleton → Empty → Error → Populated. Spinners proibidos em todo o documento.
> - **Layout base único:** `AppLayout` (módulo de chat sobreposto — sem layout autônomo). T-DC-001/FAB usa `position: fixed`, T-DC-012 herda `AppLayout` da tela Meu Perfil.
> - **Componentes críticos de domínio identificados:** `RiskScoreBadge`, `OprStatusBadge`, `AgentStatusBanner`, `RateLimitCounter`, `CommissionCard`, `ComparisonTable`, `OTPInput`.
> - **RBAC:** todas as telas exigem role `CESSIONARIO_AUTHENTICATED`. T-DC-012 adiciona guard `WhatsAppPhaseGuard`.
> - **Tokens canônicos:** `--risk-low/medium/high`, `--status-available/negotiating/closed`, `--agent-fallback/offline`, `--roi-positive/negative`, `--primary`, `--muted`, `--destructive` — todos do namespace do doc 03.
> - **Zero telas pendentes.** Nenhuma seção `[SEÇÃO PENDENTE]` neste documento.

---

## 1. Persona

Frontend Lead / UI Designer sênior com foco em contratos explícitos de implementação. Este documento elimina ambiguidade na camada de interface do AI-Dani-Cessionário: cada tela possui contrato de componentes, variantes, tokens, 4 estados obrigatórios e regras RBAC.

**Impacto no pipeline:**
- **11 - Mobile:** usa os contratos como base para adaptação mobile (breakpoint ≤ 640px).
- **15 - Arquitetura de Pastas:** mapeia cada tela em `features/dani/<componente>/`.
- **28 - Checklist de Qualidade:** valida estados, tokens e componentes conforme este documento.

---

## 2. Convenções Globais

### 2.1 Quatro Estados Obrigatórios por Tela

| Estado | Componente | Duração máxima | Observação |
|---|---|---|---|
| **Skeleton** | `<Skeleton>` shadcn/ui | Enquanto fetch em andamento | **Nunca spinner.** |
| **Empty** | `<EmptyState>` custom (ilustração + CTA) | Quando `data.length === 0` | Sempre com ação disponível |
| **Error** | `<Alert variant="destructive">` + botão de retry | Quando request falha | Mensagem específica, nunca genérica |
| **Populated** | Conteúdo real | Após dados carregados | Estado principal de uso |

### 2.2 Layouts Base

| Layout | Usado em | Estrutura |
|---|---|---|
| `AppLayout` | Telas logadas da plataforma (pai do chat) | Sidebar 280px fixa + content área |
| `OverlayLayout` | Chat da Dani (T-DC-002, T-DC-003, T-DC-004) | Panel 400px × 600px, `position: fixed`, bottom-right, sobre `AppLayout` |
| `InlineChatLayout` | Módulos inline no chat (T-DC-005 a T-DC-011) | Conteúdo dentro da área de mensagens do chat |
| `AppLayout` (seção) | T-DC-012 Vinculação WhatsApp | Subseção da tela "Meu Perfil" existente |

> [DECISÃO AUTÔNOMA] O prompt define 4 layouts canônicos (AuthLayout, AppLayout, FullPageLayout, ErrorLayout). O AI-Dani-Cessionário não possui telas autônomas — é um módulo sobreposto. Introduzidos `OverlayLayout` e `InlineChatLayout` como derivações do `AppLayout` para descrever com precisão a natureza overlay do chat. Alternativa descartada: usar `FullPageLayout` para o chat — quebraria o padrão de sobreposição documentado no D06.

### 2.3 Componentes Críticos de Domínio

| Componente | Telas | Tokens principais | Faixas semânticas |
|---|---|---|---|
| `RiskScoreBadge` | T-DC-005, T-DC-006, T-DC-009 | `--risk-low`, `--risk-medium`, `--risk-high`, `--risk-low-bg`, `--risk-medium-bg`, `--risk-high-bg` | 1–3: verde, 4–6: amarelo, 7–10: vermelho |
| `OprStatusBadge` | T-DC-005, T-DC-006, T-DC-007, T-DC-008 | `--status-available`, `--status-negotiating`, `--status-closed` | Disponível: verde, Em negociação: laranja, Encerrada: cinza |
| `AgentStatusBanner` | T-DC-010 | `--agent-fallback`, `--agent-offline` | Fallback: azul `--primary`, Offline: cinza `--muted-foreground` |
| `RateLimitCounter` | T-DC-011 | `--muted`, `--muted-foreground` | Input desabilitado + contador regressivo mm:ss |
| `CommissionCard` | T-DC-005, T-DC-007, T-DC-008 | `--primary`, `--card`, `--border` | Card com Δ, Comissão, Custo Total, ROI |
| `ComparisonTable` | T-DC-006 | `--primary`, `--accent`, `--border` | Linha destaque "Melhor opção" em `--primary` |
| `OTPInput` | T-DC-012 | `--primary`, `--destructive`, `--muted` | Normal: `--border`, Erro: `--destructive`, Sucesso: `--risk-low` |
| `ROIDeltaBadge` | T-DC-007, T-DC-008 | `--roi-positive`, `--roi-negative` | Positivo (economia): verde, Negativo (acréscimo): vermelho |

### 2.4 Padrão DataTable

Sempre que houver listagem tabulada (T-DC-006):

```jsx
<DataTable
  columns={columns}
  data={data}
  skeleton={<TableSkeleton rows={5} />}
  emptyState={<EmptyState ... />}
  pagination={{ pageSize: 5 }}
/>
```

Máximo 5 linhas (limitado a 5 OPRs por comparação — RN-DC-015). Nunca spinner.

### 2.5 StatusBadge — Mapeamento Completo por Enum

```
StatusBadge por StatusMarketplace (OprStatusBadge):
  DISPONIVEL    → --status-available   (#16A34A) 🟢
  EM_NEGOCIACAO → --status-negotiating (#D97706) 🟡
  ENCERRADA     → --status-closed      (#737373) ⚫

StatusBadge por EstadoAgente (AgentStatusBanner):
  OPERACIONAL → --agent-operational (#16A34A) — não exibido ao Cessionário
  FALLBACK    → --agent-fallback    (#0069A8) 🔵 Banner "Modo básico — sem análise da IA"
  DESLIGADO   → --agent-offline     (#737373) ⚫ Banner "Analista de Oportunidades temporariamente indisponível"

StatusBadge por ScoreDeRisco (RiskScoreBadge):
  1–3  → --risk-low    (#16A34A) bg --risk-low-bg    (#F0FDF4) 🟢 "Risco baixo"
  4–6  → --risk-medium (#D97706) bg --risk-medium-bg (#FFFBEB) 🟡 "Risco moderado"
  7–10 → --risk-high   (#DC2626) bg --risk-high-bg   (#FEF2F2) 🔴 "Risco alto"
  N/A  → --muted-foreground  bg --muted              ⚫ "Dados insuficientes"

StatusBadge por EstadoVinculacaoWhatsapp:
  NAO_VINCULADO           → --muted-foreground bg --muted     ⚫ "Não vinculado"
  OTP_SMS_ENVIADO         → --status-negotiating bg --muted   🟡 "Código enviado"
  AGUARDANDO_CONFIRMACAO_WA → --status-negotiating bg --muted 🟡 "Aguardando confirmação"
  VINCULADO               → --status-available bg --risk-low-bg 🟢 "Vinculado"
  DESVINCULADO            → --muted-foreground bg --muted     ⚫ "Desvinculado"
```

---

## 3. Módulo: FAB e Chat — Pontos de Entrada (T-DC-001 a T-DC-004)

### 3.1 T-DC-001 — FAB Global

| Atributo | Valor |
|---|---|
| **Layout** | Nenhum (componente flutuante sobre qualquer tela do AppLayout) |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` via `CessionarioOwnerGuard` |
| **RFs** | RF-DC-009, RF-DC-010 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Button` | `variant="default"`, `size="icon"`, radius `--radius-xl`, 56×56px, fundo `--primary` | Botão flutuante de abertura do chat |
| `Badge` | `variant="destructive"`, posição absolute top-right, min-w 20px, h 20px, radius `--radius-sm` | Contagem de alertas não lidos |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Não aplicável — FAB monta instantaneamente. Skeleton do `Badge` enquanto alertas carregam: `<Skeleton className="h-5 w-5 rounded-full" />` na posição do badge |
| **Empty (sem alertas)** | FAB exibido sem badge. `aria-label="Abrir chat com Dani"` |
| **Error** | FAB exibido sem badge (falha de alertas é silenciosa — não bloqueia abertura do chat) |
| **Populated (com alertas)** | Badge visível com número. `aria-label="Abrir chat com Dani — [N] alertas não lidos"` |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | `tabIndex={0}`, recebe foco via Tab | FAB alcançável via teclado em todas as telas |
| ARIA roles | `role="button"`, `aria-label` dinâmico com contagem de alertas | Screen reader anuncia corretamente |
| Skip links | Não aplicável (componente flutuante, não conteúdo principal) | — |
| Contraste | Ícone `--primary-foreground` sobre `--primary` — ratio ≥ 4.5:1 | Validado via axe-core |
| Motion | `useReducedMotion()` — desabilita scale + bounce do badge quando ativo | Animações desabilitadas em `prefers-reduced-motion` |

**Motion & Transições:**
- **Entrada:** scale `0 → 1` + fade-in, 200ms, `ease-out` (token `micro` do D04)
- **Badge aparece:** scale + bounce, 200ms, Framer Motion `spring`
- **Skeleton area:** área do badge enquanto alertas carregam
- **Feedback visual:** badge atualizado em tempo real via WebSocket/polling

**Regras específicas:**
- FAB sempre visível em todas as telas do módulo Cessionário — `z-index: 9999`
- `position: fixed; bottom: 24px; right: 24px`
- Badge oculta quando `alertas_nao_lidos === 0`
- Badge máximo exibe "99+" para contagens > 99

---

### 3.2 T-DC-002 — Chat da Dani (Sem Contexto)

| Atributo | Valor |
|---|---|
| **Layout** | `OverlayLayout` — 400px × 600px desktop, 100vw × 100dvh mobile |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-009, RF-DC-010, RF-DC-011, RF-DC-012 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `variant="default"`, radius `--radius-xl` cantos superiores, shadow `0 8px 32px oklch(0 0 0 / 16%)`, 400×600px | Container do chat window |
| `Avatar` | `size="sm"` (32px), fundo `--primary`, iniciais "DA" ou ícone Dani | Avatar da Dani no header |
| `Button` | `variant="ghost"`, `size="icon"`, `aria-label="Fechar chat"` | Botão X de fechar no header |
| `ScrollArea` | height calc(600px - 56px - 56px), `overflow-y: auto` | Área de mensagens com scroll suave |
| `Skeleton` | `className="h-12 w-3/4 rounded-lg"` × 3 linhas | Loading do histórico de mensagens |
| `Input` | `variant="default"`, radius `--radius-md`, placeholder "Escreva sua mensagem..." | Campo de texto do chat |
| `Button` | `variant="default"`, `size="icon"`, fundo `--primary`, `aria-label="Enviar mensagem"` | Botão enviar |
| Chips de ação rápida | `Button variant="outline"`, `size="sm"`, radius `--radius-md`, 4 itens | Conversation starters (primeiro acesso) |
| `Alert` | `variant="destructive"`, com ícone e botão "Tentar novamente" | Estado de erro no chat |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | 3 `<Skeleton>` de bubble na área de mensagens + `<Skeleton>` nos 4 starters. Exibido durante carregamento do histórico |
| **Empty (primeiro acesso)** | Mensagem de boas-vindas + 4 chips de conversation starters (strings do D08 seção 2.1 e 2.2) |
| **Error** | `<Alert variant="destructive">` + "Não foi possível carregar o histórico. Tente novamente." + botão "Tentar novamente" |
| **Populated** | Histórico de mensagens carregado. Input ativo. Starters ocultos |

**Estados especiais:**
| Sub-estado | Comportamento |
|---|---|
| KYC pendente | Empty com boas-vindas KYC + `<Button variant="default">Completar verificação</Button>` |
| Sem oportunidades | Empty com boas-vindas + `<Button variant="default">Ativar alertas</Button>` |
| Acesso subsequente | Populated — histórico carregado sem mensagem extra de boas-vindas |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | `useFocusTrap()` dentro do chat. Ao abrir: foco no `Input`. Ao fechar: foco retorna ao FAB | Navegação completa via teclado dentro do chat |
| ARIA roles | `role="dialog"`, `aria-modal="true"`, `aria-label="Chat com Dani"`. Área de mensagens: `role="log"`, `aria-live="polite"` | Screen reader anuncia novas mensagens automaticamente |
| Skip links | Não aplicável (chat é modal overlay) | — |
| Contraste | Bubbles Dani: `--foreground` sobre `--muted` ≥ 4.5:1. Bubbles usuário: `--primary-foreground` sobre `--primary` ≥ 4.5:1 | Validado via axe-core |
| Motion | `useReducedMotion()` — desabilita slide-up ao abrir | Animações desabilitadas em `prefers-reduced-motion` |

**Motion & Transições:**
- **Abertura:** slide-up (`translateY: 20px → 0`) + fade-in, 250ms, `ease-out` (token `component` do D04). `AnimatePresence` obrigatório
- **Fechamento:** slide-down + fade-out, 200ms, `ease-in`
- **Nova mensagem (Dani):** fade-in, 200ms
- **Nova mensagem (usuário):** nenhuma (instantâneo)
- **Skeleton areas:** toda a área de mensagens durante carregamento inicial
- **Typing indicator:** 3 pontos pulsando com `Framer Motion` enquanto Dani processa

**Regras específicas:**
- Input desabilitado quando Dani está digitando (`isTyping === true`)
- `Enter` envia mensagem. `Shift+Enter` quebra linha
- Máximo 1000 caracteres por mensagem — contador exibido a partir de 800
- Scroll automático para a última mensagem ao receber nova resposta
- Chat persistente — fechar não apaga o histórico

---

### 3.3 T-DC-003 — Chat da Dani (Com Contexto de Oportunidade)

| Atributo | Valor |
|---|---|
| **Layout** | `OverlayLayout` — idêntico a T-DC-002 |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-013, RF-DC-014, RF-DC-015 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| Todos de T-DC-002 | — | Base idêntica ao chat sem contexto |
| `Badge` (chip OPR) | `variant="secondary"`, radius `--radius-sm`, fundo `--muted`, texto `--muted-foreground` | Exibição do OPR carregado no header do chat |
| `Skeleton` | `className="h-32 w-full rounded-lg"` | Skeleton do card de análise enquanto Dani processa |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Skeleton do chat + skeleton do card de análise (32px de altura) logo após abertura |
| **Empty** | Não aplicável — contexto sempre pré-injetado. Fallback: mensagem "Oportunidade não encontrada. Informe o código OPR para continuar." |
| **Error** | `<Alert variant="destructive">` + "Não foi possível carregar os dados da oportunidade. Verifique o código OPR e tente novamente." |
| **Populated** | Chat aberto com chip OPR no header + análise automática iniciada pela Dani (T-DC-005 inline) |

**Acessibilidade (WCAG 2.1 AA):** Idêntico a T-DC-002 com adição de:

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| ARIA roles | Badge OPR: `aria-label="Analisando oportunidade [OPR-XXXX-XXXX]"` | Screen reader anuncia contexto da oportunidade |

**Motion & Transições:** Idêntico a T-DC-002.

**Regras específicas:**
- Chip OPR exibido no header imediatamente após abertura
- Dani inicia análise automaticamente (sem aguardar mensagem do usuário)
- Starters de T-DC-002 não são exibidos neste estado
- Se `opr_id` inválido: estado Empty com instrução de informar OPR

---

### 3.4 T-DC-004 — Chat da Dani (Com Contexto de Negociação)

| Atributo | Valor |
|---|---|
| **Layout** | `OverlayLayout` — idêntico a T-DC-002 |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-021, RF-DC-028 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| Todos de T-DC-002 | — | Base idêntica |
| `Badge` (chip OPR + status negociação) | `variant="secondary"` para OPR + `OprStatusBadge variant="EM_NEGOCIACAO"` | Contexto de negociação no header |
| `Skeleton` | `className="h-40 w-full rounded-lg"` | Skeleton do card de simulação de contraproposta |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Skeleton do chat + skeleton do módulo de contraproposta |
| **Empty** | Não aplicável — contexto herdado da tela de negociação |
| **Error** | `<Alert variant="destructive">` + "Não foi possível carregar os dados da negociação. Acesse a tela de negociação diretamente." + botão "Ir para negociação" |
| **Populated** | Chat com contexto de negociação. Dani sugere simulação de contraproposta (T-DC-008) |

**Acessibilidade (WCAG 2.1 AA):** Idêntico a T-DC-002.

**Motion & Transições:** Idêntico a T-DC-002.

**Regras específicas:**
- Módulo ativado automaticamente: Simulação de Contraproposta (T-DC-008)
- Valor da proposta atual pré-preenchido no módulo de simulação

---

## 4. Módulo: Análise e Comparação Inline (T-DC-005, T-DC-006)

### 4.1 T-DC-005 — Módulo Análise de Oportunidade Individual

| Atributo | Valor |
|---|---|
| **Layout** | `InlineChatLayout` — card dentro da área de mensagens |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-013, RF-DC-014, RF-DC-016, RF-DC-017, RF-DC-018, RF-DC-019, RF-DC-020 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `variant="default"`, radius `--radius-lg`, border `1px solid --border`, fundo `--card` | Container do card de análise |
| `Badge` (OPR chip) | Radius `--radius-sm`, fundo `--muted`, texto `--muted-foreground`, `size="sm"` | Identificador OPR-XXXX-XXXX |
| `OprStatusBadge` | Custom — ver mapeamento §2.5 | Status da oportunidade (verde/laranja/cinza) |
| `RiskScoreBadge` | Custom — ver mapeamento §2.5, com `aria-label` obrigatório | Score de risco 1–10 |
| `Separator` | Horizontal, cor `--border` | Divisores entre seções do card |
| `Button` | `variant="outline"`, `size="sm"`, radius `--radius-md` | Quick actions: "Simular proposta", "Comparar com similares", "Salvar para alertas" |
| `Skeleton` | `className="h-6 w-1/2 rounded"` para cada campo | Loading dos dados da análise |
| `Alert` | `variant="destructive"`, com ícone | Erro ao carregar análise |
| Disclaimer ROI | Texto `text-xs` + ícone info, fundo `--muted`, cor `--muted-foreground`, radius `--radius-sm` | Aviso obrigatório de projeção |
| Gráfico de valorização | Chart shadcn/ui (`BarChart`), cor `--chart-3` (`#0084D1`) | Comparativo regional — apenas quando dados disponíveis |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Card com skeletons em todos os campos: OPR chip, status badge, Δ, comissão, custo total, score, ROI |
| **Empty** | Não aplicável — análise é disparada com dados; sem dados = estado Error |
| **Error** | `<Alert variant="destructive">` dentro do card: "Não foi possível carregar a análise. Verifique o código OPR." + botão "Tentar novamente" |
| **Populated** | Card completo: OPR, status, Δ, comissão, custo total Escrow, score de risco, ROI 3 cenários, comparativo regional, quick actions, disclaimer |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Quick actions recebem foco em sequência lógica após card | Tab navega pelos botões de ação |
| ARIA roles | `role="article"`, `aria-label="Análise da oportunidade [OPR]"`. RiskScoreBadge: `aria-label="Score de risco: [N] de 10 — risco [baixo/moderado/alto]"` | Screen reader anuncia score com rótulo textual |
| Contraste | Risk score: cor + rótulo textual obrigatório. Ratio mínimo 4.5:1 para texto, 3:1 para UI | Validado via axe-core |
| Motion | Componentes internos sem animação extra (parte do chat stream) | — |

**Motion & Transições:**
- **Entrada do card:** fade-in, 300ms, `ease-out` (token `content` do D04) — como parte do stream SSE
- **Skeleton areas:** todos os campos do card durante loading
- **Quick actions:** sem animação de entrada (aparecem juntas com o card)
- **Feedback:** toast "Alerta ativado" após "Salvar para alertas"

**Regras específicas:**
- Disclaimer ROI obrigatório e nunca ocultável (string normativa do D08)
- `RiskScoreBadge` usa cor + rótulo textual — nunca apenas cor
- Score indisponível: exibe badge cinza com texto "Dados insuficientes" (D08 seção 2.3)
- Gráfico regional renderizado somente quando dados disponíveis; caso contrário, exibe texto "Não há dados comparativos suficientes para esta região no momento."
- Quick action "Simular proposta" navega para T-DC-007
- Quick action "Comparar com similares" navega para T-DC-006

---

### 4.2 T-DC-006 — Módulo Comparação de Oportunidades

| Atributo | Valor |
|---|---|
| **Layout** | `InlineChatLayout` — tabela dentro da área de mensagens |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-023, RF-DC-024 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Table` | `role="table"`, cabeçalhos `scope="col"`, radius `--radius-lg` externo | Container da tabela comparativa |
| `TableHeader` | Fundo `--muted`, texto `--muted-foreground text-xs font-medium uppercase` | Cabeçalho com colunas: OPR, Δ, Comissão, Custo Total Escrow, Score Risco, Localização, Tipologia |
| `TableRow` | `hover:bg-accent`, `cursor: pointer` — clique abre T-DC-005 da oportunidade | Linha de oportunidade |
| `TableRow` (melhor opção) | `background: oklch(0.5 0.134 242.749 / 6%)` + `Badge variant="default"` "Melhor opção" em `--primary` | Linha destacada da melhor oportunidade |
| `RiskScoreBadge` | Custom — mapeamento §2.5 | Score de risco por linha |
| `OprStatusBadge` | Custom — mapeamento §2.5 | Status por linha |
| `Skeleton` | `<TableSkeleton rows={3} />` | Loading da tabela |
| `Alert` | `variant="destructive"` | Erro ao carregar comparação |
| Mensagem limite | Bubble sistema com `<Alert variant="default">` | Quando > 5 OPRs solicitados |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | `<TableSkeleton rows={3} />` com colunas correspondentes |
| **Empty** | Não aplicável — tabela sempre tem ao menos 2 itens para comparar |
| **Error** | `<Alert variant="destructive">` + "Não foi possível carregar a comparação. Tente novamente." + botão retry |
| **Populated** | Tabela com 2–5 linhas, linha "Melhor opção" destacada, linhas clicáveis |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Linhas da tabela com `tabIndex={0}`, `onKeyDown` com Enter para abrir análise | Tabela navegável via teclado |
| ARIA roles | `role="table"`, `aria-label="Comparação de oportunidades"`. Cabeçalhos `scope="col"` | Screen reader lê estrutura tabular corretamente |
| Contraste | Badge "Melhor opção": `--primary-foreground` sobre `--primary` ≥ 4.5:1 | Validado via axe-core |
| Motion | Nenhuma animação de linhas individuais | — |

**Motion & Transições:**
- **Entrada da tabela:** fade-in, 300ms (token `content` do D04)
- **Skeleton:** `<TableSkeleton>` durante carregamento
- **Hover row:** transição `background-color`, 150ms
- **Feedback:** sem toast — clique na linha navega para análise inline

**Regras específicas:**
- Máximo 5 oportunidades — se `opr_ids.length > 5`: exibe bubble de sistema com mensagem de limite e prompt de refinamento (D08)
- Linha "Melhor opção" determinada pela Dani com base no menor Score de Risco + maior Δ
- Coluna OPR: chip `--muted` + `--muted-foreground`
- Valores monetários: formato `R$ 30.000,00` (D08 formatação)

---

## 5. Módulo: Simulação (T-DC-007, T-DC-008)

### 5.1 T-DC-007 — Módulo Simulação de Proposta

| Atributo | Valor |
|---|---|
| **Layout** | `InlineChatLayout` — resultado inline no chat |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-016, RF-DC-017, RF-DC-018, RF-DC-019, RF-DC-020 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `variant="default"`, radius `--radius-lg`, border `1px solid --border` | Container do resultado de simulação |
| `Input` | `variant="default"`, `type="text"`, `inputMode="decimal"`, radius `--radius-md`, placeholder "R$ 0,00" | Campo de entrada do valor de proposta |
| `Button` | `variant="default"`, fundo `--primary`, `aria-label="Simular proposta"` | Botão simular |
| `Skeleton` | `className="h-4 w-full rounded"` × 5 linhas | Loading do resultado |
| `Alert` | `variant="destructive"`, inline abaixo do input | Erro de valor inválido |
| `CommissionCard` | Custom — Card com Δ, Comissão, Custo Total Escrow, ROI 3 cenários | Resultado da simulação |
| `ROIDeltaBadge` | Custom — ver §2.3, `--roi-positive`/`--roi-negative` | Indicador de variação de ROI |
| Disclaimer ROI | Texto `text-xs`, fundo `--muted`, cor `--muted-foreground` | Aviso obrigatório e não ocultável |
| `Button` | `variant="outline"`, `size="sm"` | Quick actions: "Simular outro valor", "Ir para a oportunidade" |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | 5 `<Skeleton>` linhas dentro do `CommissionCard` enquanto cálculo processa |
| **Empty** | Input vazio + placeholder — aguardando valor do usuário |
| **Error** | `<Alert variant="destructive">` abaixo do input: mensagem específica do D08 (valor inválido, negativo ou zero) |
| **Populated** | `CommissionCard` com Δ, Comissão, Custo Total Escrow, ROI 3 cenários + disclaimer + quick actions |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Ao abrir módulo: foco no `Input`. Após simular: foco no primeiro dado do resultado | Fluxo via teclado sem mouse |
| ARIA roles | `role="form"`, `aria-label="Simulação de proposta"`. Resultado: `aria-live="polite"` | Screen reader anuncia resultado ao calcular |
| Contraste | ROI positivo/negativo: cor + seta indicativa (nunca apenas cor) | Validado via axe-core |
| Motion | `useReducedMotion()` para animação do resultado | — |

**Motion & Transições:**
- **Entrada do módulo:** fade-in, 250ms (token `component` do D04)
- **Resultado da simulação:** fade-in, 300ms após cálculo
- **Skeleton areas:** campos do `CommissionCard` durante cálculo
- **Feedback:** sem toast — resultado aparece inline

**Regras específicas:**
- Disclaimer obrigatório, nunca ocultável (string normativa D08)
- Valor inválido (zero, negativo, não-numérico): input permanece ativo + `Alert variant="destructive"` inline
- `CommissionCard` reaparece com nova simulação ao clicar "Simular outro valor"
- Botão "Ir para a oportunidade": link para tela de oportunidade (fora do módulo Dani)

---

### 5.2 T-DC-008 — Módulo Simulação de Contraproposta

| Atributo | Valor |
|---|---|
| **Layout** | `InlineChatLayout` — resultado inline no chat |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-021, RF-DC-028 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `variant="default"`, radius `--radius-lg`, border `1px solid --border` | Container do resultado |
| `Input` | `variant="default"`, `type="text"`, `inputMode="decimal"`, radius `--radius-md`, valor pré-preenchido com proposta atual | Campo da contraproposta |
| `Button` | `variant="default"`, fundo `--primary"` | Botão simular contraproposta |
| `Skeleton` | `className="h-4 w-full rounded"` × 6 linhas | Loading do resultado |
| `Alert` | `variant="destructive"` | Erro de valor inválido |
| `CommissionCard` | Custom — nova comissão, novo Escrow, diferença vs proposta anterior | Resultado da simulação |
| `ROIDeltaBadge` | Seta + valor de diferença. `--roi-positive` se menor, `--roi-negative` se maior | Diferença vs proposta anterior |
| Disclaimer ROI | Texto `text-xs`, fundo `--muted`, cor `--muted-foreground` | Aviso obrigatório |
| `Button` | `variant="default"`, `size="sm"` | "Ir para tela de negociação" |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Skeletons nos campos do resultado enquanto cálculo processa |
| **Empty** | Input pré-preenchido com valor da proposta atual + aguardando edição |
| **Error** | `<Alert variant="destructive">` inline: valor inválido ou negativo |
| **Populated** | Resultado com nova comissão, novo Escrow, ROI ajustado, diferença vs anterior com `ROIDeltaBadge` |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no `Input` ao abrir módulo | Fluxo via teclado |
| ARIA roles | `aria-live="polite"` no resultado | Screen reader anuncia resultado |
| Contraste | `ROIDeltaBadge`: seta + valor numérico (nunca apenas cor) | Validado via axe-core |
| Motion | `useReducedMotion()` | — |

**Motion & Transições:** Idêntico a T-DC-007.

**Regras específicas:**
- Diferença vs proposta anterior: sempre exibida com `ROIDeltaBadge` (seta ↓ verde = economia, ↑ vermelho = acréscimo)
- Disclaimer obrigatório e não ocultável
- Botão "Ir para tela de negociação": navegação para fora do módulo Dani

---

## 6. Módulo: Dashboard e Alertas Proativos (T-DC-009)

### 6.1 T-DC-009 — Widget Top 3 no Dashboard

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` — widget embarcado na seção "Oportunidades em Destaque" do Dashboard |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-027, RF-DC-028 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `variant="default"`, radius `--radius-lg`, border `1px solid --border` — por oportunidade × 3 | Cards de oportunidade Top 3 |
| `Badge` (OPR chip) | `variant="secondary"`, radius `--radius-sm`, fundo `--muted` | Identificador OPR de cada card |
| `OprStatusBadge` | Custom — mapeamento §2.5 | Status da oportunidade |
| `RiskScoreBadge` | Custom — mapeamento §2.5 | Score de risco |
| `Skeleton` | `className="h-32 w-full rounded-lg"` × 3 | Loading dos 3 cards |
| `Alert` | `variant="default"` (banner amarelo suave) | Banner de perfil incompleto |
| `Button` | `variant="outline"`, `size="sm"` | "Completar perfil" / "Ativar alertas" |
| `EmptyState` | Custom: ilustração + texto + CTA | Estado sem oportunidades |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | 3 `<Skeleton className="h-32 w-full rounded-lg" />` em layout de grade 3 colunas |
| **Empty** | `<EmptyState>` com texto "Nenhuma oportunidade disponível no momento." + botão `<Button variant="default">Ativar alertas</Button>` |
| **Error** | `<Alert variant="destructive">` + "Não foi possível carregar as recomendações. Tente novamente." + botão retry |
| **Populated** | 3 cards com Δ, comissão estimada, score de risco e localização. Clique abre T-DC-003 |

**Sub-estado — perfil incompleto:**
| Sub-estado | Componente | Texto |
|---|---|---|
| Perfil incompleto | `<Alert variant="default">` acima dos cards | "Recomendações baseadas em dados gerais. Complete seu perfil para recomendações personalizadas." + `<Button variant="link" size="sm">Completar perfil</Button>` |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Cards com `tabIndex={0}`, `onKeyDown` Enter abre chat | Cards acessíveis via teclado |
| ARIA roles | Seção: `role="region"`, `aria-label="Oportunidades recomendadas pela Dani"`. Cada card: `aria-label="Oportunidade [OPR], score [N], [status]"` | Screen reader anuncia cada card completamente |
| Contraste | Todos os badges com ratio ≥ 4.5:1 (cor + rótulo textual para risk score) | Validado via axe-core |
| Motion | Cards sem animação de entrada | — |

**Motion & Transições:**
- **Entrada do widget:** sem animação (parte do layout do Dashboard)
- **Skeleton:** 3 cards de skeleton durante carregamento
- **Hover card:** `box-shadow` suave, 150ms
- **Feedback:** clique abre T-DC-003 (animação da chat window)

**Regras específicas:**
- Widget atualizado por WebSocket ou polling a cada 60s
- Clique em qualquer card → abre T-DC-003 com contexto do OPR injetado
- Exatamente 3 cards — se < 3 disponíveis: exibe apenas os disponíveis + `EmptyState` no espaço restante

---

## 7. Módulo: Estados Operacionais (T-DC-010, T-DC-011)

### 7.1 T-DC-010 — Banner de Fallback da Calculadora

| Atributo | Valor |
|---|---|
| **Layout** | `InlineChatLayout` — banner contextual dentro do chat |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-031, RF-DC-032 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `AgentStatusBanner` | Custom — ver §2.3. Estado `FALLBACK`: cor `--agent-fallback` + ícone calculadora. Estado `DESLIGADO`: cor `--agent-offline` + ícone pausa | Banner de status do agente |
| `Alert` | `variant="default"` customizado com cor `--agent-fallback` | Container do banner de fallback |
| `Button` | `variant="ghost"`, `size="sm"` | "Copiar valores", "Novo cálculo" |
| `Skeleton` | Não aplicável — banner exibido após resposta da Calculadora | — |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Skeleton da resposta da Calculadora (igual skeleton de mensagem da Dani) enquanto cálculo processa |
| **Empty** | Não aplicável — banner só aparece após cálculo |
| **Error** | `<Alert variant="destructive">` + "Não foi possível executar o cálculo. Tente novamente." |
| **Populated** | Banner `AgentStatusBanner` + resultado do cálculo determinístico da Calculadora |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Banner não captura foco | Fluxo não interrompido |
| ARIA roles | `role="status"`, `aria-live="polite"`, `aria-label="Modo básico ativo — resultados da Calculadora"` | Screen reader anuncia modo fallback |
| Contraste | Texto `--primary-foreground` sobre `--agent-fallback` ≥ 4.5:1 | Validado via axe-core |
| Motion | Nenhuma animação (aparece como bubble comum) | — |

**Motion & Transições:**
- **Entrada:** fade-in, 200ms (mesmo timing de bubble de sistema)
- **Skeleton:** skeleton de bubble enquanto Calculadora processa
- **Feedback:** sem animação especial

**Regras específicas:**
- Textos normativas do D08 — não podem ser alterados sem aprovação do produto
- Fallback: "Modo básico — sem análise da IA" + ícone calculadora
- Desligado: "Analista de Oportunidades temporariamente indisponível" + ícone pausa
- Ações disponíveis mesmo em fallback: copiar valores, solicitar novo cálculo

---

### 7.2 T-DC-011 — Estado de Rate Limit do Chat

| Atributo | Valor |
|---|---|
| **Layout** | `InlineChatLayout` — estado do componente de input |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` |
| **RFs** | RF-DC-033 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Input` | `disabled`, fundo `--muted`, cursor `not-allowed`, `aria-label="Chat temporariamente bloqueado. Aguarde [mm:ss]."` | Campo de input desabilitado |
| `Button` | `disabled`, `opacity: 0.4`, `aria-disabled="true"` | Botão enviar desabilitado |
| `RateLimitCounter` | Custom — texto `mm:ss` em `--muted-foreground`, `text-xs`, atualizado a cada segundo via `setInterval` | Contador regressivo até desbloqueio |
| Bubble sistema | Bubble centralizada (mapeamento §3.3 do D03), texto `--muted-foreground` | Mensagem informativa de limite atingido |
| `Skeleton` | Não aplicável | — |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Não aplicável — rate limit é estado do input, não carregamento de dados |
| **Empty** | Não aplicável |
| **Error** | Não aplicável — rate limit é estado operacional esperado |
| **Populated (rate limit ativo)** | Input desabilitado + botão inativo + contador `mm:ss` + bubble de sistema com string normativa do D08 |
| **Populated (desbloqueio)** | Pulse 500ms via Framer Motion + input reativado + contador oculto |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco retorna ao input após desbloqueio | Usuário não precisa clicar para retomar |
| ARIA roles | Input: `aria-disabled="true"`, `aria-label` com tempo restante. Bubble sistema: `aria-live="assertive"` | Screen reader anuncia bloqueio imediatamente |
| Contraste | Contador `--muted-foreground` sobre `--muted` ≥ 4.5:1 | Validado via axe-core |
| Motion | `useReducedMotion()` — desabilita pulse de desbloqueio | — |

**Motion & Transições:**
- **Ativação do rate limit:** sem animação de entrada (estado muda instantaneamente)
- **Desbloqueio:** pulse sutil 500ms, Framer Motion `spring`
- **Contador:** atualização a cada 1s sem animação
- **Feedback:** `aria-live="assertive"` na bubble de sistema ao atingir limite

**Regras específicas:**
- Ativado após 30 mensagens/hora (RN-DC-025) — janela deslizante via Redis
- String normativa do D08: "Você atingiu o limite de mensagens por hora. Aguarde [mm:ss] para continuar."
- Pulse de desbloqueio: 500ms, uma vez — não repete
- Contador calculado client-side (baseado no timestamp de reset retornado pela API)

---

## 8. Módulo: Vinculação WhatsApp (T-DC-012)

### 8.1 T-DC-012 — Vinculação WhatsApp (Fase 2)

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` — subseção da tela "Meu Perfil" (tela existente da plataforma) |
| **Role mínimo** | `CESSIONARIO_AUTHENTICATED` + `WhatsAppPhaseGuard` (Fase 2 habilitada) |
| **RFs** | RF-DC-037, RF-DC-038, RF-DC-039, RF-DC-040, RF-DC-041, RF-DC-042 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `variant="default"`, radius `--radius-lg`, border `1px solid --border` | Container da seção WhatsApp em Meu Perfil |
| `Input` | `variant="default"`, `type="tel"`, `inputMode="numeric"`, radius `--radius-md`, placeholder "+55 (XX) XXXXX-XXXX" | Campo de número de telefone |
| `Button` | `variant="default"`, fundo `--primary"`, `aria-label="Vincular WhatsApp"` | Botão "Vincular" |
| `OTPInput` | Custom — 6 campos `Input` `maxLength={1}` em linha, auto-avança entre campos, radius `--radius-md` | Inserção do OTP de 6 dígitos |
| `Alert` | `variant="destructive"` | Erros: OTP inválido, número inválido, tentativas esgotadas |
| `Alert` | `variant="default"` (tom informativo) | Instruções de cada etapa |
| `Badge` (status) | Custom `WhatsappStatusBadge` — mapeamento §2.5 | Estado atual da vinculação |
| `Separator` | Horizontal, `--border` | Entre etapas do fluxo |
| `Button` | `variant="destructive"`, `size="sm"` | "Desvincular" (estado vinculado) |
| `Dialog` | `variant="default"`, `aria-modal="true"` | Modal de confirmação de desvinculação |
| `Skeleton` | `className="h-20 w-full rounded-lg"` | Loading do estado atual de vinculação |
| `Progress` | Indicador de etapas (1/3, 2/3, 3/3), cor `--primary` | Progresso do fluxo de vinculação |

**Estados obrigatórios:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | `<Skeleton className="h-20 w-full rounded-lg" />` enquanto carrega estado atual do banco |
| **Empty (não vinculado)** | Card com `WhatsappStatusBadge` "Não vinculado" + campo de telefone + botão "Vincular" |
| **Error** | `<Alert variant="destructive">` específico por tipo: número inválido, OTP incorreto, hard block (5 falhas) |
| **Populated (vinculado)** | Número mascarado (exibe sufixo 4 dígitos "••••-XXXX") + `WhatsappStatusBadge` "Vinculado" + botão "Desvincular" |

**Sub-estados por etapa do fluxo:**

| Etapa | Componentes | Comportamento |
|---|---|---|
| **Etapa 1/3** (informar número) | Input telefone + botão "Enviar código" + `Progress` 1/3 | Validação em tempo real do formato. Erro inline abaixo do input |
| **Etapa 2/3** (inserir OTP) | `OTPInput` 6 campos + timer "Código válido por [mm:ss]" + link "Reenviar código" + `Progress` 2/3 | Rate limit: 3 tentativas/hora. Hard block: 5 falhas = `Alert destructive` com tempo bloqueado |
| **Etapa 3/3** (confirmar WhatsApp) | Instrução de confirmar no WhatsApp + `Progress` 3/3 + loader aguardando webhook | Timeout 5 min. Após confirmação: transição para estado "Vinculado" |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Ao entrar na etapa 2: foco no primeiro campo do `OTPInput`. Auto-avança entre campos. Ao confirmar: foco na confirmação de sucesso | Fluxo via teclado sem mouse |
| ARIA roles | `OTPInput`: `role="group"`, `aria-label="Código de verificação — 6 dígitos"`. Cada campo: `aria-label="Dígito [N] de 6"` | Screen reader naviga corretamente no OTP |
| Skip links | Não aplicável (subseção de formulário) | — |
| Contraste | Estados de erro `--destructive` sobre fundo branco ≥ 4.5:1. OTP correto: borda `--risk-low` | Validado via axe-core |
| Motion | `useReducedMotion()` — desabilita animações de transição entre etapas | — |

**Motion & Transições:**
- **Transição entre etapas:** slide horizontal (etapa 1→2→3), 250ms, `ease-out` (token `component` do D04). `AnimatePresence` obrigatório
- **Skeleton:** durante carregamento do estado atual
- **Sucesso de vinculação:** fade-in do estado "Vinculado", 300ms
- **Feedback OTP errado:** shake animation no `OTPInput`, 300ms (somente se `prefers-reduced-motion` inativo)

**Regras específicas:**
- Número completo nunca armazenado em texto claro — somente hash bcrypt + sufixo 4 dígitos para exibição
- Hard block: 5 falhas consecutivas → `Alert variant="destructive"` + tempo de desbloqueio (RN-DC-041)
- OTP expirado (> 15 min): `Alert variant="destructive"` + link "Reenviar código"
- Desvinculação: `Dialog` de confirmação com texto normativo do D08
- `WhatsAppPhaseGuard`: se Fase 2 não habilitada, seção exibe banner `Alert variant="default"` informando disponibilidade futura

---

## 9. Cross-References

| Documento | Relação |
|---|---|
| **01 - Regras de Negócio** | Fonte de verdade para RNs: rate limit (RN-DC-025), score de risco (RN-DC-011/012), fallback (RN-DC-023/024), WhatsApp (RN-DC-040-044), SLA (RN-DC-029) |
| **02 - Stacks** | Stack obrigatória: shadcn/ui, Tailwind CSS v4, Framer Motion ≥11, React 19, NestJS ≥10 |
| **03 - Brand Theme Guide** | Todos os tokens de design: `--risk-*`, `--status-*`, `--agent-*`, `--roi-*`, `--primary`, `--muted`, `--destructive` |
| **06 - Mapa de Telas** | Fonte dos 12 IDs de tela (T-DC-001 a T-DC-012), rotas, tipos e RFs |
| **07 - Wireframes** | Layout visual de referência por tela — base para posicionamento e hierarquia |
| **08 - UX Writing** | Strings normativas de boas-vindas, fallback, rate limit, disclaimer ROI, OTP, desvinculação |
| **11 - Mobile** | Usa este documento para adaptação mobile (breakpoint ≤ 640px por tela) |
| **15 - Arquitetura de Pastas** | Mapeia cada tela em `features/dani/<componente>/` |
| **28 - Checklist de Qualidade** | Valida 4 estados, tokens, RBAC e acessibilidade por tela |

---

## Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 23/03/2026 | Claude Code Desktop — Pipeline v9.5 | Versão inicial. 12 telas cobertas (T-DC-001 a T-DC-012). Contratos completos com 4 estados obrigatórios, RBAC, acessibilidade WCAG 2.1 AA, motion & transições e StatusBadge mappings. Zero telas pendentes. |

---

## Backlog de Pendências

| Item | Tipo | Tela / Módulo | Impacto | Justificativa / Pergunta | Dono | Status |
|---|---|---|---|---|---|---|
| Layout `OverlayLayout` e `InlineChatLayout` | Decisão Autônoma | T-DC-002 a T-DC-011 | P2 | Layouts derivados do `AppLayout` para descrever natureza overlay do chat — não conflitam com padrão; apenas especializam. Alternativa `FullPageLayout` descartada (quebraria sobreposição do D06) | Frontend Lead | Aberto |
| Linha "Melhor opção" em T-DC-006 | Decisão Autônoma | T-DC-006 | P2 | Critério: menor Score de Risco + maior Δ. Produto deve validar se há regra de negócio diferente para o ranking | Product Owner | Aberto |
| `WhatsAppPhaseGuard` | Decisão Autônoma | T-DC-012 | P1 | Guard sugerido para bloquear acesso à seção em Fase 1. Confirmar implementação com backend | Backend Lead | Aberto |
| Ícone da Dani (avatar FAB) | Definição Pendente | T-DC-001 | P2 | Asset de ícone/avatar da Dani não definido no D03. Opção A: ícone customizado da persona. Opção B: iniciais "DA" no formato `Avatar` shadcn/ui | Design Lead | Aberto |
| Polling interval do Widget Top 3 (T-DC-009) | Decisão Autônoma | T-DC-009 | P2 | Definido 60s como intervalo razoável para evitar overhead. Confirmar com backend se WebSocket está disponível para updates em tempo real | Backend Lead | Aberto |
