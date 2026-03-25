# 07 - Wireframes — AI-Dani-Cedente

## Wireframes do Chat da Dani (Perspectiva do Cedente)

| Campo | Valor |
|---|---|
| Destinatário | Time de Design e Frontend |
| Escopo | Wireframes detalhados de todas as telas do chat AI-Dani-Cedente |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D01 (Regras de Negócio), D03 (Brand Guide), D05.1–D05.5 (PRD), D06 (Mapa de Telas) |

---

> **📌 TL;DR**
>
> - Wireframes detalhados para as 7 telas do Mapa de Telas (T-01 a T-07) + notificação toast.
> - Notação ASCII + especificações de layout, dimensões e comportamento.
> - Cada wireframe referencia os tokens do Brand Guide (D03) e os RFs do PRD (D05.x).
> - Breakpoints: desktop 1280px (primário), tablet 768px, mobile 375px.
> - Todos os wireframes partem da perspectiva do Cedente — jamais exibem dados do Cessionário.

---

## 1. Breakpoints e Grid

| Breakpoint | Largura | Chat Width | Chat Height |
|---|---|---|---|
| **Desktop** | ≥ 1280px | 400px (flutuante lateral direita) | 600px |
| **Tablet** | 768–1279px | 380px (flutuante ou drawer) | 580px |
| **Mobile** | < 768px | 100vw | 100dvh |

**Posicionamento do chat (desktop):**
- Flutuante fixo no canto inferior direito: `bottom: 24px; right: 24px`
- Z-index: 1000 (acima do conteúdo da página, abaixo de modais do sistema)
- Botão de acionamento (FAB): círculo 56px, `--primary`, ícone da Dani, `bottom: 24px; right: 24px`

---

## 2. Wireframe: T-01 — Chat Inicial

**Estado:** KYC aprovado + oportunidade ativa

```
┌─────────────────────────────────── 400px ────────┐
│  [Ⓓ] Dani                              [×]       │  ← Header 64px
│       Guardiã do Retorno  ● Disponível            │
├──────────────────────────────────────────────────┤
│                                                  │  ← Área de mensagens
│                                                  │    min-height: 400px
│  ┌────────────────────────────────────────┐      │    overflow-y: auto
│  │ Olá! Sou a Dani, sua Guardiã do        │      │
│  │ Retorno. Estou aqui para ajudá-lo a    │      │  ← Bolha Dani
│  │ acompanhar sua oportunidade,           │      │    bg: --primary
│  │ entender as propostas e garantir que  │      │    max-w: 75%
│  │ o repasse seja tranquilo.              │      │
│  │ Como posso ajudar?                     │      │
│  │                                   agora│      │  ← timestamp
│  └────────────────────────────────────────┘      │
│                                                  │
├──────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────┐│  ← Sugestões 80px
│  │[Retorno esperado?][Proposta vale a pena?]    ││
│  │[O que falta no dossiê?][Quanto tempo?]       ││
│  └──────────────────────────────────────────────┘│
├──────────────────────────────────────────────────┤
│  ┌────────────────────────────────┐ [Enviar →]  │  ← Footer 64px
│  │ Digite sua mensagem...         │             │
│  └────────────────────────────────┘             │
└──────────────────────────────────────────────────┘

Dimensões:
- Container total: 400px × 608px
- Header: 400px × 64px (padding: 16px)
- Área mensagens: 400px × 400px (flex: 1, overflow auto)
- Área sugestões: 400px × 80px (padding: 12px)
- Footer: 400px × 64px (padding: 12px)
```

---

## 3. Wireframe: T-04 — Chat em Conversa (Estado Principal)

```
┌──────────────────────────────────── 400px ───────┐
│  [Ⓓ] Dani                              [×]       │  Header 64px
│       Guardiã do Retorno  ● Disponível            │
├──────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────┐         │
│  │ Olá! Como posso ajudar hoje?        │ 09:30   │  Dani (--primary)
│  └─────────────────────────────────────┘         │
│                                                  │
│                   ┌─────────────────────────────┐│
│                   │ Quero analisar a proposta    ││  Cedente (--secondary)
│                   │ que recebi ontem             ││  align: right
│                   │                      09:31  ││
│                   └─────────────────────────────┘│
│                                                  │
│  ● ● ●  (digitando...)                           │  Typing indicator
│                                                  │
├──────────────────────────────────────────────────┤
│  ┌────────────────────────────────┐ [Enviar →]  │
│  │ Digite sua mensagem...         │             │
│  └────────────────────────────────┘             │
└──────────────────────────────────────────────────┘

Especificações de bolha:
- Bolha Dani: max-w 75%, bg --primary, color --primary-foreground
  border-radius: radius-xl radius-xl radius-xl radius-sm (canto inf. esq. menor)
  padding: 12px, margin-bottom: 4px (consecutivas), 16px (troca de remetente)

- Bolha Cedente: max-w 75%, bg --secondary, color --secondary-foreground
  border-radius: radius-xl radius-xl radius-sm radius-xl (canto inf. dir. menor)
  padding: 12px, margin-left: auto

- Timestamp: text-xs, --muted-foreground, padding-top: 4px
```

---

## 4. Wireframe: T-05 — Card de Análise de Proposta

```
┌──────────────────────────────────── 400px ───────┐
│  [Ⓓ] Dani                              [×]       │
├──────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────┐         │
│  │ Aqui está a análise da proposta:    │         │  Bolha texto Dani
│  └─────────────────────────────────────┘         │
│  ┌─────────────────────────────────────┐         │  Card análise
│  │ 📋 Análise da Proposta              │         │  bg: --card
│  │ ─────────────────────────────────  │         │  border: 1px --border
│  │ Valor proposto                      │         │  border-radius: radius-lg
│  │ R$ 320.000          −5% da tabela  │         │  padding: 16px
│  │                                    │         │  shadow-sm
│  │ Retorno líquido estimado           │         │
│  │ R$ 85.000           ↑ destacado   │         │  ← valor em --primary
│  │ (Cenário B)                        │         │     font-weight: 700
│  │                                    │         │     text-lg
│  │ Prazo para resposta                │         │
│  │ 3 dias úteis                       │         │
│  │ ─────────────────────────────────  │         │
│  │ [Aceitar]  [Recusar]  [Contrapr.]  │         │  Botões de ação
│  └─────────────────────────────────────┘         │
│                                     09:35        │
├──────────────────────────────────────────────────┤
│  ┌────────────────────────────────┐ [Enviar →]  │
│  │ Digite sua mensagem...         │             │
└──────────────────────────────────────────────────┘

Dimensões do card:
- Largura: 100% da área de mensagens menos padding (calc(100% - 32px))
- Altura: auto (min 140px)
- Botões: altura 36px, padding 8px 16px, gap 8px
- [Aceitar]: --primary
- [Recusar]: --destructive
- [Contrapr.]: --secondary
```

---

## 5. Wireframe: T-05 — Card de Status do Dossiê

```
│  ┌─────────────────────────────────────┐         │
│  │ 📁 Status do Dossiê — 83% completo  │         │
│  │ [=============================    ] │         │  ← Progress bar
│  │ ─────────────────────────────────  │         │
│  │ ✅ Contrato original                │         │  Badge verde
│  │ ✅ Matrícula do imóvel              │         │
│  │ ✅ Certidão negativa de ônus reais  │         │
│  │ ✅ Comprovante de pagamentos        │         │
│  │ ❌ Certidão negativa de débitos     │         │  Badge vermelho
│  │    Motivo: documento com rasura     │         │  texto xs --destructive
│  │    [Como corrigir →]                │         │  link --primary
│  │ 📎 Procuração (se aplicável)        │         │  Badge cinza
│  └─────────────────────────────────────┘         │

Badges:
- ✅ Aprovado: bg green-100 text-green-800, radius-sm, px-2 py-0.5
- ❌ Rejeitado: bg red-100 text-red-800, radius-sm, px-2 py-0.5
- ⏳ Em análise: bg yellow-100 text-yellow-800
- 📎 Pendente: bg gray-100 text-gray-600
- Progress bar: bg --primary, height 6px, radius full
```

---

## 6. Wireframe: T-06 — Modal de Confirmação (Aceitar Proposta)

```
┌──────────────────────────────────── 400px ───────┐
│  [Ⓓ] Dani  (opacidade reduzida)       [×]       │  Chat com overlay
├──────────────────────────────────────────────────┤
│  [fundo escurecido opacity 0.4]                  │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Confirmar aceite da proposta             │   │  Modal
│  │                                          │   │  bg: --card
│  │ Valor da proposta                        │   │  border-radius: radius-xl
│  │ R$ 320.000                               │   │  padding: 24px
│  │                                          │   │  shadow-xl
│  │ Retorno líquido estimado                 │   │  max-w: 360px
│  │ R$ 85.000                                │   │  margin: auto
│  │                                          │   │
│  │ Próximos passos:                         │   │
│  │ • Depósito Escrow (10 dias úteis)        │   │
│  │ • Assinatura contrato via ZapSign        │   │
│  │                                          │   │
│  │  [Cancelar]       [Aceitar proposta →]   │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
└──────────────────────────────────────────────────┘

Botões no modal:
- [Cancelar]: --secondary, border 1px --border, radius-md
- [Aceitar proposta →]: --primary, radius-md, font-weight 600
- Gap entre botões: 12px
- Botão width: calc(50% - 6px)
```

---

## 7. Wireframe: T-07 — Estados de Sistema

### T-07a: Digitando

```
│  ● ● ●                                           │
│  ↑                                               │
│  3 pontos na cor --muted-foreground               │
│  dentro de bolha --muted, radius-xl               │
│  Bounce animation 1.4s loop                       │
│  Posição: alinhado à esquerda (posição da Dani)   │
```

### T-07b: Resposta Lenta

```
│  ┌─────────────────────────────────────┐         │
│  │ ⏳ A Dani está demorando mais que    │         │
│  │ o esperado. Você pode aguardar ou   │         │
│  │ tentar novamente em instantes.      │         │
│  │                                    │         │
│  │  [Aguardar]   [Tentar novamente]   │         │
│  └─────────────────────────────────────┘         │

bg: --muted, border: 1px --border, radius-lg, padding: 12px
Botões: variant outline, radius-md, height 36px
```

### T-07c: Rate Limit

```
┌──────────────────────────────────────────────────┐
│  (chat normal acima)                             │
├──────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────┐  │
│  │ 🚫 Limite de mensagens atingido             │  │  Banner informativo
│  │ Disponível novamente em 47:32               │  │  bg: amber-50
│  └────────────────────────────────────────────┘  │  border: amber-200
│  ┌──────────────────────────┐ [Enviar →]       │
│  │ (input desabilitado)     │ (desabilitado)   │
│  └──────────────────────────┘                   │
│  opacity: 0.5 nos elementos de input             │
└──────────────────────────────────────────────────┘
```

### T-07d: Dani Indisponível

```
┌──────────────────────────────────── 400px ───────┐
│  [Ⓓ] Dani                              [×]       │
│       Guardiã do Retorno  ● Indisponível          │  status: vermelho
├──────────────────────────────────────────────────┤
│                                                  │
│  ┌─────────────────────────────────────┐         │
│  │ 😔 A Dani está temporariamente      │         │
│  │ indisponível. Para urgências,       │         │
│  │ entre em contato com o suporte.     │         │
│  │                                    │         │
│  │        [Contatar suporte →]         │         │
│  └─────────────────────────────────────┘         │
│                                                  │
├──────────────────────────────────────────────────┤
│  ┌────────────────────────────────┐ [Enviar →]  │
│  │ (desabilitado)                 │ (disabled)  │
│  └────────────────────────────────┘             │
└──────────────────────────────────────────────────┘
```

---

## 8. Wireframe: Notificação Toast Proativa

```
Desktop (canto superior direito):
┌──────────────────────────────────────────┐
│ 📄 Nova proposta recebida                │  width: 320px
│ R$ 320.000 para OPR-0042-2026           │  bg: --card
│                                          │  border: 1px --border
│             [Ver detalhes →] [×]         │  border-radius: radius-xl
└──────────────────────────────────────────┘  shadow-lg
                                              padding: 12px 16px

Posição: fixed, top: 20px, right: 20px, z-index: 9999
Stacking: gap 8px para múltiplas notificações
Auto-dismiss (informativas): 5 segundos com progress bar

Mobile (canto inferior):
┌──────────────────────────────────────────┐
│ 📄 Nova proposta: R$ 320.000             │  width: calc(100vw - 32px)
│             [Ver detalhes →]             │  bottom: 16px, centered
└──────────────────────────────────────────┘
```

---

## 9. Mobile Adaptations (375px)

### Chat Fullscreen (Mobile)

```
┌─── 375px ─────────────────────────────┐
│  ←  Dani            ●               │  Header 56px
│     Guardiã do Retorno               │  back button para fechar
├───────────────────────────────────────┤
│                                       │
│  (mensagens)                          │  flex: 1
│                                       │  overflow-y: auto
│                                       │  padding-bottom: safe-area
├───────────────────────────────────────┤
│  [Sugestões — horizontal scroll]      │  72px se presentes
├───────────────────────────────────────┤
│  ┌─────────────────────┐ [→]         │  Footer 56px
│  │ Mensagem...         │             │  padding-bottom: safe-area
│  └─────────────────────┘             │
└───────────────────────────────────────┘

Diferenças mobile vs desktop:
- Chat ocupa 100vw × 100dvh (fullscreen)
- Botão "←" no header fecha o chat (volta à tela anterior)
- FAB de acionamento: 48px × 48px (menor que desktop)
- Teclado virtual: bottom padding dinâmico via CSS env(keyboard-inset-height)
- Sugestões: scroll horizontal ao invés de wrap
```

---

## 10. Anotações de Acessibilidade

| Elemento | ARIA | Notas |
|---|---|---|
| Container do chat | `role="dialog"` `aria-label="Chat com a Dani"` | Para leitores de tela |
| Área de mensagens | `role="log"` `aria-live="polite"` `aria-label="Histórico de conversas"` | Anuncia novas mensagens |
| Bolha da Dani | `aria-label="Dani: [conteúdo]"` | Identifica o remetente |
| Bolha do Cedente | `aria-label="Você: [conteúdo]"` | Identifica o remetente |
| Typing indicator | `aria-label="Dani está digitando"` `aria-live="polite"` | Anuncia quando aparece |
| Campo de mensagem | `aria-label="Digite sua mensagem para a Dani"` | — |
| Botão enviar | `aria-label="Enviar mensagem"` | — |
| Modal | `role="dialog"` `aria-modal="true"` `aria-labelledby="modal-title"` | Focus trap obrigatório |
| Botão fechar (×) | `aria-label="Fechar chat"` | — |

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — Wireframes completos de T-01 a T-07 + toast proativo + mobile adaptations + anotações de acessibilidade. |
