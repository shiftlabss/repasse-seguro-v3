# 09 - Contratos de UI por Tela — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Frontend Lead / UI Designer |
| Escopo | Mapeamento de componentes shadcn/ui, tokens e estados obrigatórios por tela |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Fase | 2 — UI/UX |
| Área | Design + Frontend |
| Referências | D01 · D03 · D06 · D07 · D08 |

---

> **📌 TL;DR**
>
> - **7 telas cobertas:** T-01 a T-07 (chat widget flutuante 400×608px desktop / fullscreen mobile).
> - **4 estados obrigatórios por tela:** Skeleton → Empty → Error → Populated. Spinners proibidos.
> - **Layout base único:** `ChatWidgetLayout` (não é AppLayout/sidebar — é widget flutuante embutido na plataforma Repasse Seguro).
> - **Componentes de domínio:** `AnalysisCard` (4 variantes), `ProactiveToast`, `ConfirmationModal`, `TypingIndicator`, `CsatRating`.
> - **Role mínimo:** `role: 'cedente'` (JWT claim) em todas as telas — nunca exposto a Cessionário ou usuário não autenticado.
> - **Tokens:** namespace `--dani-*` herda 100% de `--rs-*` (Repasse Seguro) via shadcn/ui conforme D03.
> - **Zero seções pendentes** — cobertura 100% das 7 telas do D06.

---

## 1. Persona

Frontend Lead responsável por materializar cada tela do chat AI-Dani-Cedente em contrato explícito de componentes, estados e tokens. Foco em eliminar ambiguidade: nenhum dev interpreta — apenas implementa.

---

## 2. Convenções Globais

### 2.1 Quatro Estados Obrigatórios por Tela

| Estado | Componente | Duração máxima |
|---|---|---|
| **Skeleton** | `<Skeleton>` shadcn/ui — nunca `<Spinner>` | Enquanto fetch/streaming em andamento |
| **Empty** | `<EmptyState>` custom (ilustração + título + descrição + CTA opcional) | Quando não há dados/histórico |
| **Error** | `<Alert variant="destructive">` + botão retry | Quando request ou API LLM falha |
| **Populated** | Conteúdo real renderizado | Após dados carregados |

### 2.2 Layout Base do Chat

> **Nota arquitetural:** o AI-Dani-Cedente não tem próprio frontend independente. O chat widget é embutido como componente flutuante na plataforma Repasse Seguro. Não usa `AppLayout` com sidebar.

| Layout | Contexto | Estrutura |
|---|---|---|
| `ChatWidgetLayout` | Desktop — widget flutuante | 400px × 608px, fixed bottom-right, z-index 9999, shadow-lg |
| `ChatFullscreenLayout` | Mobile — tela cheia | 100vw × 100dvh, sem border-radius, sem sombra |

### 2.3 Componentes Críticos de Domínio

| Componente | Telas | Tokens principais | Faixas semânticas |
|---|---|---|---|
| `AnalysisCard` (4 variantes) | T-05 | `--rs-primary`, `--rs-muted`, `--rs-border` | Proposta / Simulação / Dossiê / Escrow (cada um com cor de header diferente) |
| `ProactiveToast` | Transversal (10 eventos) | `--rs-primary`, `--rs-warning`, `--rs-destructive` | Info / Alerta / Urgente |
| `ConfirmationModal` | T-06 | `--rs-destructive`, `--rs-primary` | Confirmação destrutiva / Confirmação padrão |
| `TypingIndicator` | T-04, T-07 | `--rs-muted-foreground` | Animação de 3 pontos bounce |
| `CsatRating` | T-07 (pós-conversa) | `--rs-primary`, `--rs-muted` | 1–5 estrelas |
| `StatusBadge` | T-02, T-03, T-05 | `--rs-success`, `--rs-warning`, `--rs-destructive`, `--rs-muted` | Por enum (ver mapeamentos abaixo) |
| `RateLimitCounter` | T-07 | `--rs-warning`, `--rs-destructive` | 0–29 mensagens: normal / 25–30: warning / 30: blocked |

### 2.4 StatusBadge — Mapeamentos por Enum

```
StatusBadge por OpportunityStatus:
RASCUNHO      → muted (cinza)          🔘
PUBLICADA     → success (verde)        ✅
EM_NEGOCIACAO → primary (azul)         🔵
PAUSADA       → warning (amarelo)      ⚠️
EXPIRADA      → destructive (vermelho) ❌
CANCELADA     → destructive (vermelho) ❌

StatusBadge por ProposalStatus:
RECEBIDA        → primary (azul)         🔵
EM_ANALISE      → warning (amarelo)      👁
ACEITA          → success (verde)        ✅
RECUSADA        → destructive (vermelho) ❌
CONTRAPROPOSTA  → secondary (roxo/índigo)↩️
CANCELADA       → muted (cinza)          🔘
EXPIRADA        → destructive (vermelho) ❌

StatusBadge por DossierDocumentStatus:
AGUARDANDO_ENVIO → muted (cinza)          📄
EM_ANALISE       → warning (amarelo)      🔍
APROVADO         → success (verde)        ✅
REJEITADO        → destructive (vermelho) ❌

StatusBadge por EscrowStatus:
AGUARDANDO_DEPOSITO  → warning (amarelo)      ⏳
DEPOSITO_CONFIRMADO  → success (verde)        💰
LIBERADO             → success (verde escuro) 🎉
REVERTIDO            → destructive (vermelho) ↩️
EXPIRADO             → destructive (vermelho) ❌
```

---

## 3. Módulo Chat — Telas T-01 a T-07

### T-01 — Chat Inicial (Boas-vindas / Primeiro Acesso)

| Atributo | Valor |
|---|---|
| **Layout** | `ChatWidgetLayout` (desktop) / `ChatFullscreenLayout` (mobile) |
| **Role mínimo** | `role: 'cedente'` — validado via JWT claim no AuthGuard |
| **RFs** | RF-DCE-005 (boas-vindas), RF-DCE-006 (pontos de entrada), RF-DCE-007 (histórico) |

**Tabela de Componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Avatar` | size: 40px, src: Dani avatar, fallback: "D" | Avatar da Dani no header |
| `Badge` | variant: "outline", size: "sm" | Status online da Dani ("Online") |
| `Button` | variant: "ghost", size: "icon" | Botão fechar (X) no header |
| `Skeleton` | width: 100%, height: 48px, rounded | Skeleton da mensagem de boas-vindas enquanto carrega |
| `Alert` | variant: "destructive" | Error state: falha ao carregar contexto do Cedente |
| Mensagem de boas-vindas (`ChatBubble` primary) | variant: "assistant", max-width: 85% | Primeira mensagem da Dani |
| `Button` x3 | variant: "outline", size: "sm", full-width | Sugestões de conversa iniciais |
| `Input` + `Button` enviar | variant: "default" para input, variant: "default" para send | Campo de entrada de mensagem |

**Estados da T-01:**

| Estado | Trigger | Comportamento |
|---|---|---|
| **Skeleton** | Carregando perfil do Cedente (KYC status, oportunidade) | Skeleton na área de mensagens + skeleton nas sugestões |
| **Empty (KYC Pendente)** | `kyc_status === 'PENDENTE'` | Boas-vindas com CTA "Iniciar verificação" — input desabilitado |
| **Empty (sem oportunidade)** | KYC aprovado, sem oportunidade | Boas-vindas contextualizada — "Vou te avisar quando surgir uma oportunidade" |
| **Error** | Falha de autenticação ou JWT inválido | `<Alert variant="destructive">` "Não foi possível carregar o chat. Tente novamente." + retry |
| **Populated** | KYC aprovado + oportunidade ativa | Boas-vindas com 3 sugestões de conversa contextualizadas |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Ao abrir o widget, focus vai para o campo de input | Tab order: header → mensagens → input → botão enviar |
| ARIA roles | `role="dialog"` no widget, `aria-label="Chat com a Dani"` | Screen reader anuncia "Chat com a Dani, janela de diálogo" |
| Skip links | Não aplicável (widget flutuante) | — |
| Contraste | Texto primário: `--rs-foreground` sobre `--rs-background` (≥ 4.5:1) | Validado via axe-core |
| Motion | Transição de entrada respeita `prefers-reduced-motion` | `@media (prefers-reduced-motion: reduce)` desativa slide/fade |

**Motion e Transições:**

- Transição de entrada: `slideUp` + `fadeIn`, 250ms, `ease-out` (D04 token: `normal`)
- Sugestões de conversa: `staggerChildren` 50ms, `fadeIn` 150ms cada (D04 token: `fast`)
- Skeleton: pulse de opacidade 0.5↔1.0, 1.5s loop
- Sem animação se `prefers-reduced-motion: reduce`

**Regras específicas:**

- Exibir exatamente 3 sugestões de conversa — nunca mais, nunca menos
- Sugestões mudam conforme contexto (KYC, oportunidade, proposta)
- Input desabilitado se `kyc_status === 'PENDENTE'`
- CTA "Iniciar verificação" abre modal externo da plataforma (não gerenciado pela Dani)

---

### T-02 — Contexto de Oportunidade (Ponto de Entrada PE-2)

| Atributo | Valor |
|---|---|
| **Layout** | `ChatWidgetLayout` / `ChatFullscreenLayout` |
| **Role mínimo** | `role: 'cedente'` com `opportunity.status !== null` |
| **RFs** | RF-DCE-010 (cadastro), RF-DCE-011 (cenários A/B/C/D), RF-DCE-012 (estados), RF-DCE-013 (retirada) |

**Tabela de Componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | padding: 12px, border: `--rs-border` | Card de contexto da oportunidade no topo do chat |
| `StatusBadge` (custom) | Conforme mapeamento `OpportunityStatus` | Status atual da oportunidade |
| `Skeleton` | Dentro do card de contexto | Skeleton enquanto carrega dados da oportunidade |
| `Alert` | variant: "destructive" | Error ao carregar oportunidade |
| `ChatBubble` (assistant) | max-width: 85%, bg: `--rs-primary/10` | Mensagem da Dani com contexto da oportunidade |
| `Button` | variant: "outline", size: "sm" | Sugestões contextuais (ex: "Ver cenários", "Publicar no marketplace") |

**Estados da T-02:**

| Estado | Trigger | Comportamento |
|---|---|---|
| **Skeleton** | Carregando dados da oportunidade | Card de contexto com skeleton |
| **Empty** | Nenhum cenário calculado ainda | Dani orienta sobre cadastro de dados para calcular cenário |
| **Error** | Falha ao carregar oportunidade | Alert + retry |
| **Populated** | Oportunidade carregada com status e Δ | Card de contexto com `StatusBadge` + mensagem contextualizada |

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Focus no card de contexto ao abrir T-02 | Screen reader lê o status da oportunidade |
| ARIA roles | `aria-label` no card: "Sua oportunidade: [status]" | Anuncia corretamente |
| Contraste | `StatusBadge` cores validadas | Ratio ≥ 4.5:1 |
| Motion | Card de contexto: `fadeIn` 150ms | Respeitado `prefers-reduced-motion` |

**Motion:** Card de contexto desliza do topo (`slideDown` 250ms, `ease-out`). StatusBadge sem animação.

**Regras específicas:**

- Cenários A/B/C/D nunca são exibidos ao Cessionário — RLS garante, mas UI também oculta
- Botão "Retirar do marketplace" só exibido se `status === 'PUBLICADA'` ou `'EM_NEGOCIACAO'`
- Se `status === 'EM_NEGOCIACAO'`, Dani informa que é necessário recusar a proposta primeiro antes de retirar

---

### T-03 — Contexto de Proposta (Ponto de Entrada PE-3)

| Atributo | Valor |
|---|---|
| **Layout** | `ChatWidgetLayout` / `ChatFullscreenLayout` |
| **Role mínimo** | `role: 'cedente'` com `proposal.status !== null` |
| **RFs** | RF-DCE-016 (estados proposta), RF-DCE-017 (análise), RF-DCE-018 (simulação), RF-DCE-019 (contraproposta), RF-DCE-020 (aceite), RF-DCE-021 (recusa) |

**Tabela de Componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | padding: 12px, border: `--rs-border` | Card de contexto da proposta no topo |
| `StatusBadge` (custom) | Conforme mapeamento `ProposalStatus` | Status atual da proposta |
| `Skeleton` | Card de contexto | Skeleton durante carregamento |
| `Alert` | variant: "destructive" | Error ao carregar proposta |
| `ChatBubble` (assistant) | max-width: 85% | Mensagem da Dani com análise da proposta |
| `AnalysisCard` variant="proposta" | Ver T-05 | Card de análise inline exibido pela Dani |

**Estados da T-03:**

| Estado | Trigger | Comportamento |
|---|---|---|
| **Skeleton** | Carregando dados da proposta | Card de contexto com skeleton |
| **Empty** | Sem propostas recebidas | "Nenhuma proposta ainda — sua oportunidade está visível no marketplace." |
| **Error** | Falha ao carregar proposta | Alert + retry |
| **Populated** | Proposta carregada | Card com `StatusBadge` + mensagem de análise da Dani |

**Acessibilidade:** idêntico ao padrão T-02 com `aria-label="Proposta recebida: [status]"`.

**Motion:** igual T-02 — card desliza do topo.

**Regras específicas:**

- Dados do Cessionário nunca exibidos (nome, CPF, contato) — apenas valores financeiros da proposta
- Múltiplas propostas: Dani exibe tabela comparativa no `AnalysisCard variant="proposta"`
- `StatusBadge` no card muda em tempo real conforme webhook de status da proposta

---

### T-04 — Conversa Ativa

| Atributo | Valor |
|---|---|
| **Layout** | `ChatWidgetLayout` / `ChatFullscreenLayout` |
| **Role mínimo** | `role: 'cedente'` |
| **RFs** | RF-DCE-001 (isolamento), RF-DCE-007 (histórico + rate limit), RF-DCE-008 (sugestões), RF-DCE-009 (SLA) |

**Tabela de Componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `ScrollArea` | height: calc(608px - 120px - 64px), overflowY: auto | Área de scroll das mensagens |
| `ChatBubble` (assistant) | bg: `--rs-primary/10`, border-radius-tl: 2px, max-width: 85% | Mensagem da Dani |
| `ChatBubble` (user) | bg: `--rs-primary`, color: white, border-radius-tr: 2px, max-width: 75% | Mensagem do Cedente |
| `Avatar` | size: 32px, Dani | Avatar da Dani na bolha (primeira mensagem de cada bloco) |
| `TypingIndicator` | 3 dots bounce, `--rs-muted-foreground` | Indicador de digitação da Dani |
| `Skeleton` x3 | height: 40px/60px/48px, width: 60%/85%/45% | Skeleton do histórico de mensagens |
| `Alert` | variant: "destructive", with retry | Error de carregamento do histórico |
| `Button` x3 | variant: "outline", size: "sm", full-width | Sugestões de resposta rápida (contextual) |
| `Textarea` / `Input` | placeholder: "Digite uma mensagem…", maxLength: sem limite UI | Campo de entrada de mensagem |
| `Button` send | variant: "default", size: "icon", `aria-label="Enviar mensagem"` | Botão de envio |

**Estados da T-04:**

| Estado | Trigger | Comportamento |
|---|---|---|
| **Skeleton** | Carregando histórico da sessão | 3 bolhas skeleton intercaladas (2 assistant + 1 user) |
| **Empty** | Histórico vazio, primeira mensagem | "Nenhuma conversa ainda — pode me perguntar qualquer coisa." |
| **Error** | Falha de conexão ou API LLM | Alert "Não foi possível enviar sua mensagem. Verifique sua conexão." + retry |
| **Populated** | Histórico carregado | Mensagens renderizadas, auto-scroll para a mais recente |

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no textarea após cada envio | Mantém foco no input durante a conversa |
| ARIA roles | `role="log"`, `aria-live="polite"` na área de mensagens | Screen reader anuncia novas mensagens |
| `TypingIndicator` | `aria-label="A Dani está digitando"`, `aria-live="polite"` | Anunciado ao screen reader |
| Contraste | Bolha assistant: `--rs-primary/10` bg com `--rs-foreground` text | ≥ 4.5:1 |
| Motion | Bolhas: spring slide conforme D04 | `prefers-reduced-motion` desativa spring |

**Motion:**

- Bolha nova (assistant): `slideInLeft` spring (stiffness: 300, damping: 30) — D04 `daniMessageVariant`
- Bolha nova (user): `slideInRight` spring — D04 `cedenteMessageVariant`
- Sugestões: `staggerChildren` 50ms, `fadeIn` 150ms — D04 `suggestionVariant`
- Typing indicator: bounce loop 500ms/dot — D04 `typingBounceVariant`
- Sem animação se `prefers-reduced-motion: reduce`

**Regras específicas:**

- Auto-scroll para a mensagem mais recente após cada nova mensagem
- Sugestões de resposta rápida: 3 opções máximo, contextuais ao estado atual
- Rate limit: campo de input desabilitado ao atingir 30 mensagens/hora (exibir T-07 rate-limit)
- Streaming de resposta: tokens exibidos progressivamente via SSE (não espera resposta completa)

---

### T-05 — Cards de Análise Inline

| Atributo | Valor |
|---|---|
| **Layout** | Embutido dentro de T-04 (não é tela independente) |
| **Role mínimo** | `role: 'cedente'` (herda de T-04) |
| **RFs** | RF-DCE-017 (análise proposta), RF-DCE-018 (simulação), RF-DCE-015 (dossiê), RF-DCE-022 (Escrow) |

**4 Variantes do AnalysisCard:**

#### AnalysisCard variant="proposta"

| Componente | Config | Propósito |
|---|---|---|
| `Card` header | bg: `--rs-primary/10`, height: 40px | Header do card com título "Análise de proposta" |
| Tabela comparativa (múltiplas propostas) | `Table`, `TableRow`, `TableCell` | Comparação de valor, retorno líquido, status |
| `Badge` por `ProposalStatus` | Conforme mapeamento | Status de cada proposta |
| `Button` "Aceitar proposta" | variant: "default", size: "sm" | CTA primário |
| `Button` "Enviar contraproposta" | variant: "outline", size: "sm" | CTA secundário |

#### AnalysisCard variant="simulacao"

| Componente | Config | Propósito |
|---|---|---|
| `Card` header | bg: `--rs-success/10`, height: 40px | Header "Simulação de retorno" |
| Valor do repasse | `p` bold, `--rs-foreground` | Valor da proposta |
| Saldo devedor | `p` muted, `--rs-muted-foreground` | Valor a ser deduzido |
| Retorno líquido | `p` bold large, `--rs-success` | Resultado da simulação |
| Disclaimer | `p` small muted, italic | "Estimativa — consulte corretor para valor definitivo" |

#### AnalysisCard variant="dossie"

| Componente | Config | Propósito |
|---|---|---|
| `Card` header | bg: `--rs-warning/10`, height: 40px | Header "Status do dossiê" |
| Lista de documentos | `ul`, `li` com `StatusBadge` | Cada documento com status |
| Progresso | `Progress` value: (aprovados/total × 100) | Barra de progresso do dossiê |
| `Button` "Ver documentos necessários" | variant: "outline", size: "sm" | CTA contextual |

#### AnalysisCard variant="escrow"

| Componente | Config | Propósito |
|---|---|---|
| `Card` header | bg: `--rs-primary/10`, height: 40px | Header "Status do Escrow" |
| `StatusBadge` EscrowStatus | Conforme mapeamento | Status atual |
| Prazo de depósito | `p` com countdown | Data limite + dias restantes |
| `Alert` variant: "warning" | Exibido quando ≤ 2 dias úteis | "O prazo vence em X dias" |
| `Button` "Confirmar extensão" | variant: "outline", size: "sm" | Exibido quando extensão solicitada |

**Estados da T-05 (para cada variante de card):**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Card com skeleton de altura definida (120px para proposta, 100px para simulação) |
| **Empty** | Card sem dados — mensagem contextualizada pela Dani (sem CTA no card) |
| **Error** | Ícone de erro + "Não foi possível carregar. Tente novamente." |
| **Populated** | Card completo com dados reais |

**Motion:**

- Expansão do card: `scaleY` de 0 a 1, 400ms `ease-out` — D04 `analysisCardVariant`
- `prefers-reduced-motion`: sem animação de expansão

---

### T-06 — Modais de Confirmação

| Atributo | Valor |
|---|---|
| **Layout** | Overlay sobre `ChatWidgetLayout` — z-index: 10000 |
| **Role mínimo** | `role: 'cedente'` (herda da sessão ativa) |
| **RFs** | RF-DCE-020 (aceite proposta), RF-DCE-021 (recusa), RF-DCE-013 (retirada marketplace), RF-DCE-024 (extensão Escrow) |

**Tabela de Componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Dialog` | modal: true, overlay bg: `rgba(0,0,0,0.5)` | Estrutura do modal |
| `DialogHeader` | — | Título do modal |
| `DialogTitle` | size: "lg", sentence case | Pergunta de confirmação |
| `DialogDescription` | color: `--rs-muted-foreground` | Consequência da ação |
| `Button` (primário) | variant: "default" ou "destructive" | CTA de confirmação (contextual) |
| `Button` (secundário) | variant: "outline" | CTA de cancelamento |

**4 Variantes de Modal:**

#### Modal aceitar proposta

- Título: "Aceitar proposta?"
- Descrição: "Ao aceitar, o Cessionário terá 10 dias úteis para depositar o valor no Escrow."
- CTA primário: `<Button variant="default">Aceitar proposta</Button>`
- CTA secundário: `<Button variant="outline">Cancelar</Button>`

#### Modal recusar proposta

- Título: "Recusar proposta?"
- Descrição: "A proposta será recusada e o Cessionário será notificado."
- CTA primário: `<Button variant="destructive">Recusar proposta</Button>`
- CTA secundário: `<Button variant="outline">Cancelar</Button>`

#### Modal retirar do marketplace

- Título: "Retirar oportunidade do marketplace?"
- Descrição: "Sua oportunidade ficará invisível para Cessionários. Você poderá republicá-la quando quiser."
- CTA primário: `<Button variant="destructive">Retirar oportunidade</Button>`
- CTA secundário: `<Button variant="outline">Cancelar</Button>`

#### Modal confirmar extensão de Escrow

- Título: "Confirmar extensão de prazo?"
- Descrição: "O prazo do Escrow será estendido em +5 dias úteis. Silêncio em 24h equivale a aprovação."
- CTA primário: `<Button variant="default">Confirmar extensão</Button>`
- CTA secundário: `<Button variant="outline">Recusar extensão</Button>`

**Estados da T-06:**

| Estado | Comportamento |
|---|---|
| **Skeleton** | Não aplicável — modal só abre com dados já carregados |
| **Empty** | Não aplicável |
| **Error** | Se ação falhar após confirmação: `<Alert variant="destructive">` dentro do modal (modal permanece aberto) |
| **Populated** | Modal aberto com dados da ação |

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Focus trap dentro do modal. Focus no primeiro botão ao abrir. | Tab circula apenas dentro do modal |
| ARIA roles | `role="alertdialog"` para confirmações destrutivas, `role="dialog"` para confirmações padrão | Screen reader anuncia "diálogo de alerta" para ações irreversíveis |
| Tecla Escape | Fecha o modal (equivale a "Cancelar") | Testado via keyboard |
| Contraste | CTA destructive: `--rs-destructive` bg com branco texto | ≥ 4.5:1 |

**Motion:**

- Entrada: `scale` 0.95→1.0 + `fadeIn`, 250ms, `ease-out` — D04 `modalVariant`
- Saída: `scale` 1.0→0.95 + `fadeOut`, 150ms, `ease-in`
- Overlay: `fadeIn` 200ms
- `prefers-reduced-motion`: sem escala, apenas `fadeIn/Out`

---

### T-07 — Estados do Sistema

| Atributo | Valor |
|---|---|
| **Layout** | Inline dentro do `ChatWidgetLayout` (não substitui o chat) |
| **Role mínimo** | `role: 'cedente'` |
| **RFs** | RF-DCE-009 (SLA), RF-DCE-029 (fallback), RF-DCE-031 (latência), RF-DCE-032 (CSAT) |

**4 Sub-estados do Sistema:**

#### T-07a — Digitando (Typing Indicator)

| Componente | Config | Propósito |
|---|---|---|
| `TypingIndicator` | 3 dots, bounce loop 500ms/dot, `--rs-muted-foreground` | Indica que Dani está processando (após envio, antes de primeira token) |

Exibido: imediatamente após envio da mensagem do Cedente. Substituído pela bolha de resposta assim que primeira token chega via SSE.

#### T-07b — Resposta Lenta (SLA > 5s)

| Componente | Config | Propósito |
|---|---|---|
| `TypingIndicator` | Mantido | Indicador de digitando continua |
| Mensagem inline | `p` small, `--rs-muted-foreground` | "A Dani está preparando a resposta…" — exibida abaixo do typing indicator após 5s |

#### T-07c — Rate Limit Atingido (30 mensagens/hora)

| Componente | Config | Propósito |
|---|---|---|
| `Alert` | variant: "warning" | "Você atingiu o limite de 30 mensagens por hora." |
| Countdown | `span` com timer regressivo | Tempo restante até liberar |
| `Input` (textarea) | `disabled: true` | Campo desabilitado |
| `Button` send | `disabled: true` | Botão desabilitado |
| `RateLimitCounter` | progress bar 0–30 | Visual de uso do rate limit |

#### T-07d — Dani Indisponível (Fallback)

| Componente | Config | Propósito |
|---|---|---|
| `Alert` | variant: "destructive" | "A Dani está temporariamente indisponível. Para urgências, entre em contato com o suporte." |
| `Button` "Falar com suporte" | variant: "outline", size: "sm" | Link para suporte da plataforma |
| `Input` (textarea) | `disabled: true` | Campo desabilitado |

#### T-07e — CSAT (pós-conversa)

Exibido ao encerrar a sessão (Cedente fecha o chat ou após 5min de inatividade):

| Componente | Config | Propósito |
|---|---|---|
| `CsatRating` | 5 estrelas, `aria-label` por estrela | Avaliação de satisfação |
| `Button` "Enviar avaliação" | variant: "default", size: "sm" | CTA primário |
| `Button` "Pular" | variant: "ghost", size: "sm" | CTA terciário — CSAT é opcional |

**Acessibilidade de T-07:**

| Requisito | Implementação |
|---|---|
| Rate limit countdown | `aria-live="polite"` — atualiza screen reader a cada minuto |
| Dani indisponível | `aria-live="assertive"` — anuncia imediatamente ao screen reader |
| CSAT stars | Cada estrela tem `aria-label="Avaliar com N estrela(s)"` + `role="radio"` |

**Motion:** Nenhuma animação especial — estados do sistema não têm transição decorativa para não gerar distração durante falhas.

---

## 4. Componente Transversal — ProactiveToast

Não é uma tela — é um componente overlay exibido em qualquer tela do chat quando um evento proativo ocorre.

| Componente | Config | Propósito |
|---|---|---|
| `Toast` shadcn/ui | slide-da-direita, 5s auto-dismiss, max-width: 360px | Container do toast proativo |
| Ícone semântico | Conforme tipo do evento | Contexto visual |
| Título do toast | bold, max: 80 chars | Evento resumido |
| `Button` "Ver detalhes" | variant: "ghost", size: "sm" | Abre contexto relacionado no chat |
| `Button` fechar (X) | variant: "ghost", size: "icon" | Dispensa o toast |

**10 Eventos com mapeamento de tipo:**

| Evento | Tipo | Ícone | Exemplo de texto |
|---|---|---|---|
| Nova proposta recebida | `primary` | 📨 | "Nova proposta de R$ 280.000,00 recebida." |
| Proposta aceita (confirmação) | `success` | ✅ | "Proposta aceita! Aguardando depósito no Escrow." |
| Proposta recusada | `muted` | — | "Proposta recusada. Sua oportunidade continua no marketplace." |
| Escrow: depósito confirmado | `success` | 💰 | "Depósito confirmado no Escrow." |
| Escrow: alerta de prazo (2 dias) | `warning` | ⏰ | "O prazo do Escrow vence em 2 dias." |
| Escrow: extensão solicitada | `primary` | 🔔 | "O Cessionário solicitou extensão de prazo (+5 dias)." |
| Escrow: expirado | `destructive` | ❌ | "Prazo do Escrow encerrou. Proposta cancelada." |
| Documento aprovado | `success` | 📄 | "Documento [nome] aprovado pelo Admin." |
| Documento rejeitado | `destructive` | ❌ | "Documento [nome] rejeitado. Veja o motivo." |
| ZapSign: lembrete de assinatura | `warning` | ✍️ | "Seu contrato aguarda assinatura." |

**Motion:** slide-da-direita + fadeIn, 300ms `ease-out` — D04 `notificationVariant`. `prefers-reduced-motion`: apenas fadeIn.

---

## 5. Cross-References

| Documento | Relação |
|---|---|
| D01 — Regras de Negócio | Fonte dos fluxos de status e regras que impactam estados visuais |
| D03 — Brand Guide | Fonte dos tokens `--rs-*` usados em todos os componentes |
| D04 — Motion Spec | Fonte dos tokens e variantes de animação (spring, fast, normal, slow) |
| D06 — Mapa de Telas | Fonte das 7 telas (T-01 a T-07) e dos 10 eventos proativos |
| D07 — Wireframes | Referência visual de layout e dimensões (400×608px, etc.) |
| D08 — UX Writing | Fonte dos textos de empty state, error, CTAs e labels |
| D10 — Glossário Técnico | Nomenclatura canônica de componentes (CedenteProfile, Proposal, etc.) |
| D11 — Mobile | Usa contratos de UI como referência para adaptação mobile |
| D15 — Arquitetura de Pastas | Usa contratos para mapear `features/chat/components/` e `features/chat/pages/` |
| D28 — Checklist de Qualidade | Valida cobertura de 4 estados, tokens e RBAC por tela |

---

## 6. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 23/03/2026 | Claude Code Desktop | Versão inicial. 7 telas (T-01 a T-07), componente transversal ProactiveToast, 4 variantes de AnalysisCard, 4 variantes de ConfirmationModal. StatusBadge mappings completos para 5 enums. Acessibilidade WCAG 2.1 AA e Motion D04 por tela. |

---

## 7. Backlog de Pendências

| Item | Tipo | Tela / Módulo | Impacto | Justificativa | Dono | Status |
|---|---|---|---|---|---|---|
| Componente `CsatRating` — validar se usar `RadioGroup` shadcn/ui com ícone de estrela ou componente custom | DECISÃO AUTÔNOMA | T-07e | P2 | Adotado `RadioGroup` com ícone estrela SVG preenchido/vazio. Alternativa descartada: component custom separado (aumenta bundle sem necessidade). | Frontend | Validar |
| `AnalysisCard` variant="proposta" — tabela comparativa para 1 proposta vs. múltiplas propostas simultâneas | DECISÃO AUTÔNOMA | T-05 | P1 | Adotado layout de tabela para N propostas (mesmo com 1, evita retrabalho). Alternativa descartada: layout de card único para 1 proposta (quebra consistência ao adicionar segunda proposta). | Frontend / Produto | Validar |
| Focus trap no `ChatWidgetLayout` em desktop — confirmar se o focus deve sair do widget ao pressionar Tab no último elemento | DECISÃO AUTÔNOMA | T-01/T-04 | P2 | Adotado focus trap: Tab circula dentro do widget. Alternativa: permitir Tab sair (melhor para power users). [DECISÃO AUTÔNOMA — WCAG recomenda focus trap em dialogs; widget tem `role="dialog"`.] | Acessibilidade / Frontend | Validar com QA |
