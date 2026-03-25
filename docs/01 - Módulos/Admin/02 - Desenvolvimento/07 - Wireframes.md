# 07 - Wireframes

## Repasse Seguro — Módulo Admin

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Design, Frontend e QA |
| **Escopo** | Wireframes de baixa fidelidade das telas prioritárias, anotações de layout e comportamento, especificações de componentes e fluxos de navegação |
| **Módulo** | Admin |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-22 (America/Fortaleza) |
| **Dependências** | 06 - Mapa de Telas · 01.1–01.5 Regras de Negócio · 03 Brand Guide · 05.1–05.5 PRD |

---

> 📌 **TL;DR**
>
> - Wireframes de **21 telas prioritárias** organizados por bloco funcional.
> - Representação em ASCII art + anotações descritivas. Ferramenta de design (Figma) deve usar este documento como blueprint.
> - Fidelidade: baixa (sem cores, sem tipografia final, sem imagens reais).
> - Foco: estrutura, hierarquia de informação, posicionamento de componentes e anotações de comportamento.
> - Telas prioritárias selecionadas por impacto operacional: autenticação, dashboard, pipeline, triagem, negociação, formalização, financeiro, supervisão IA, configurações.

---

## 1. Convenções de Wireframe

### 1.1 Notação

```
┌─────────┐  = Container / Card / Modal
│         │  = Área de conteúdo
└─────────┘

[BOTÃO]      = Botão de ação
[input]      = Campo de entrada
[□ checkbox] = Checkbox
[▼ select]   = Dropdown/Select
[tab]        = Aba de navegação

████████     = Imagem / Avatar placeholder
░░░░░░░░     = Skeleton loading / área cinza
▓▓▓▓▓▓▓▓     = Área preenchida / destaque
═══════      = Linha divisória / separador
→            = Navegação para outra tela
⚠            = Alerta
ℹ            = Tooltip / informação
```

### 1.2 Anotações

Anotações seguem o formato:
```
[A1] Texto da anotação
```
Referenciadas no wireframe com `[A1]`, `[A2]`, etc.

---

## 2. Wireframes por Bloco

---

### 2.1 Bloco: Autenticação

#### WF-T001 — Login

```
┌──────────────────────────────────────────────────┐
│                                                  │
│              ████████████                        │
│           [Logo Repasse Seguro]                  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │                                            │  │
│  │  ┌──────────────────────────────────────┐  │  │
│  │  │ [A1] E-mail                          │  │  │
│  │  │ [input: email]                       │  │  │
│  │  └──────────────────────────────────────┘  │  │
│  │                                            │  │
│  │  ┌──────────────────────────────────────┐  │  │
│  │  │ [A2] Senha                           │  │  │
│  │  │ [input: password]          [👁 show] │  │  │
│  │  └──────────────────────────────────────┘  │  │
│  │                                            │  │
│  │  [A3]                                      │  │
│  │  ┌──────────────────────────────────────┐  │  │
│  │  │          [ENTRAR]                    │  │  │  ← primário, full-width
│  │  └──────────────────────────────────────┘  │  │
│  │                                            │  │
│  │        [Esqueci minha senha] →T-003        │  │
│  │                                            │  │
│  └────────────────────────────────────────────┘  │
│                                                  │
└──────────────────────────────────────────────────┘
Estado erro credencial: [A4]
┌────────────────────────────────────┐
│ ⚠ E-mail ou senha incorretos.     │  ← abaixo do campo senha, text destructive
└────────────────────────────────────┘

Estado bloqueio 5 tentativas: [A5]
┌────────────────────────────────────────────────┐
│ ⚠ Conta bloqueada. Tente em 30 minutos.       │  ← banner topo card, destructive
└────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Input email: placeholder "seu@email.com". aria-label="E-mail". Validação: formato de e-mail.
- [A2] Input senha: placeholder "Senha". Toggle show/hide. aria-label="Senha".
- [A3] Botão "Entrar": ao clicar → loading state (spinner + "Entrando..."). Previne duplo clique.
- [A4] Shake animation no card (200ms) ao erro de credencial.
- [A5] Formulário desabilitado + countdown visível quando bloqueio ativo.

---

#### WF-T002 — 2FA

```
┌──────────────────────────────────────────────────┐
│                                                  │
│              ████████████                        │
│          [Logo Repasse Seguro]                   │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │                                            │  │
│  │  Verificação em 2 etapas                  │  │
│  │  Digite o código do seu app autenticador  │  │
│  │                                            │  │
│  │  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐    │  │  ← [A1] 6 inputs OTP
│  │  │   │ │   │ │   │ │   │ │   │ │   │    │  │
│  │  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘    │  │
│  │                                            │  │
│  │  ┌──────────────────────────────────────┐  │  │
│  │  │          [VERIFICAR]                 │  │  │
│  │  └──────────────────────────────────────┘  │  │
│  │                                            │  │
│  │      [Usar código de recuperação]          │  │
│  │                                            │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] 6 inputs individuais. Foco avança automaticamente. Paste detecta 6 dígitos e distribui.

---

### 2.2 Bloco: Dashboard

#### WF-T010 — Dashboard Principal (perfil Coordenador)

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                               │
│  ████ Repasse Seguro  |  Dashboard             🔔(3)  ████ Coord. ▼ │
├─────────────┬────────────────────────────────────────────────────────┤
│  SIDEBAR    │  Dashboard                      [Hoje ▼] [Este mês ▼] │
│  ─────────  │                                                        │
│  ▓ Dashboard│  [A1] ══ BANNER ALERTA SLA (se há SLAs estourados) ══  │
│  ○ Pipeline │                                                        │
│  ○ Triagem  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  ○ Negoc.   │  │ Casos ativos │ │ SLAs vencidos│ │ Analistas    │ │ Sem atrib.   │
│  ○ Formal.  │  │              │ │              │ │              │ │              │
│  ○ Financ.  │  │   ████  42   │ │   ████   7   │ │   ████   5   │ │   ████   3   │
│  ○ Superv.  │  │              │ │ ▲ 2 vs ant.  │ │              │ │              │
│  ○ Usuários │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
│  ○ Relatór. │                                                        │
│  ○ Config.  │  ┌─────────────────────────────────────┐ ┌──────────────────────────┐
│             │  │  Funil de Casos  [A2]               │ │  Ranking Analistas [A3]  │
│             │  │                                     │ │                          │
│             │  │  ░░░░░░░░░░░░░░ Em Triagem  (14)    │ │  1. Ana S.    ████ 12    │
│             │  │  ░░░░░░░ Oferta Ativa         (8)   │ │  2. João P.   ████  9    │
│             │  │  ░░░░░ Em Negociação           (6)  │ │  3. Maria L.  ████  8    │
│             │  │  ░░░ Em Formalização           (3)  │ │  4. Carlos M. ████  7    │
│             │  │  ░ Concluídos este mês         (11) │ │  5. Beatriz C.████  5    │
│             │  │                                     │ │                          │
│             │  └─────────────────────────────────────┘ └──────────────────────────┘
│             │                                                        │
│             │  ┌─────────────────────────────────────┐ ┌──────────────────────────┐
│             │  │  Atividades Pendentes [A4]          │ │  Alertas SLA [A5]        │
│             │  │                                     │ │                          │
│             │  │  Case #001  Triagem  Há 2h          │ │  ⚠ #045 - Em Negoc. 1d  │
│             │  │  Case #002  Negoc.   Há 4h          │ │  ⚠ #012 - Formal. 2h    │
│             │  │  Case #003  Formal.  Há 6h          │ │  ⚠ #089 - Triagem venc. │
│             │  │  [Ver todas as atividades →]        │ │                          │
│             │  └─────────────────────────────────────┘ └──────────────────────────┘
└─────────────┴────────────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Banner SLA: aparece apenas se há SLAs estourados. Fundo destructive/10. Dismissível (×).
- [A2] Gráfico funil: barras horizontais empilhadas horizontalmente. aria-label com dados textuais. Clicável: filtra Pipeline pelo estado correspondente.
- [A3] Ranking: avatar 24px + nome + barra de progresso + número de casos concluídos. Clicável: vai para perfil do analista.
- [A4] Lista de atividades: as 5 mais recentes. Cada item clicável → detalhe do caso.
- [A5] Alertas: lista dos casos com SLA próximo ou estourado. Ícone ⚠ amarelo (próximo) ou vermelho (estourado).

---

### 2.3 Bloco: Pipeline

#### WF-T020 — Kanban

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                                           │
├─────────────┬────────────────────────────────────────────────────────────────────┤
│  SIDEBAR    │  Pipeline                [Analista ▼] [Cenário ▼] [SLA ▼]  [☰][▤] │
│             │  ═══════════════════════════════════════════════════════════════════│
│             │  [A1] ←scroll horizontal→                                          │
│             │                                                                    │
│             │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ... (14 col)│
│             │  │ Captado  │ │Em Triagem│ │ Bloqueado│ │Qualific. │              │
│             │  │   (3)    │ │  (14)    │ │   (5)    │ │   (4)    │              │
│             │  │──────────│ │──────────│ │──────────│ │──────────│              │
│             │  │          │ │          │ │          │ │          │              │
│             │  │ ┌──────┐ │ │ ┌──────┐ │ │ ┌──────┐ │ │ ┌──────┐ │              │
│             │  │ │ #001 │ │ │ │ #002 │ │ │ │ #010 │ │ │ │ #018 │ │              │
│             │  │ │ End. │ │ │ │ End. │ │ │ │ End. │ │ │ │ End. │ │              │
│             │  │ │ [B]  │ │ │ │ [A]  │ │ │ │ [C]  │ │ │ │ [D]  │ │              │
│             │  │ │ ████ │ │ │ │ ████ │ │ │ │ ████ │ │ │ │ ████ │ │              │
│             │  │ │ 2d🟡 │ │ │ │ 5d🔴 │ │ │ │ 1d🟢 │ │ │ │ 3d🟡 │ │              │
│             │  │ └──────┘ │ │ └──────┘ │ │ └──────┘ │ │ └──────┘ │              │
│             │  │          │ │          │ │          │ │          │              │
│             │  │ ┌──────┐ │ │ ┌──────┐ │ │ (vazio)  │ │          │              │
│             │  │ │ #003 │ │ │ │ #004 │ │ │          │ │          │              │
│             │  │ │ ...  │ │ │ │ ...  │ │ │          │ │          │              │
│             │  │ └──────┘ │ │ └──────┘ │ │          │ │          │              │
│             │  │          │ │          │ │          │ │          │              │
│             │  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │
└─────────────┴────────────────────────────────────────────────────────────────────┘

Card de caso (detalhado): [A2]
┌────────────────────┐
│ #001 · [A cenário] │  ← ID + badge cenário
│ Rua das Flores, 42 │  ← endereço, max 2 linhas, truncate
│ ████ Ana S.        │  ← avatar 24px + nome analista
│ Há 2 dias · 🟡 SLA │  ← tempo no estado + indicador SLA
│ R$ 320.000         │  ← valor estimado
└────────────────────┘
```

**Anotações:**
- [A1] Scroll horizontal nativo. Sidebar deve estar colapsada (w-16) para maximizar área do board.
- [A2] Card clicável → T-022 (drawer). Drag-and-drop habilitado para Coordenador/Master. Coluna de destino inválida: opacidade 0.4. Alternativa teclado via Space/Enter → menu contextual.

---

#### WF-T022 — Drawer Detalhe do Caso

```
┌─────────────────────────────────────────────────────────────────────┐
│ ÁREA PRINCIPAL (blur/overlay semitransparente)    ┌────────────────┐│
│                                                   │ [×] Fechar     ││
│                                                   │                ││
│                                                   │ Caso #001      ││
│                                                   │ [Em Triagem]   ││
│                                                   │ Cenário [B]    ││
│                                                   │                ││
│                                                   │ ══════════════ ││
│                                                   │                ││
│                                                   │[Resumo][Dossiê]││
│                                                   │[Histórico][Ações]│
│                                                   │                ││
│                                                   │ ░░░░░░░░░░░░░  ││  ← [A1]
│                                                   │ ░░░░░░░░░░░░░  ││
│                                                   │ ░░░░░░░░░░░░░  ││
│                                                   │                ││
│                                                   │ ══════════════ ││
│                                                   │                ││
│                                                   │  [AÇÃO PRIMÁRIA] │  ← [A2]
│                                                   │  [ação sec.]   ││
│                                                   └────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
Mobile: Drawer = Sheet fullscreen slide-up from bottom
```

**Anotações:**
- [A1] Conteúdo da tab ativa. Loading: skeleton.
- [A2] Ações variam por perfil e estado do caso. Ver T-031 (Triagem) para ações específicas.

---

### 2.4 Bloco: Triagem

#### WF-T030/T031 — Triagem (layout master-detail)

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                               │
├─────────────┬────────────────────────────────────────────────────────┤
│  SIDEBAR    │  Triagem                                               │
│             │  ─────────────────────────────────────────────────────│
│             │  ┌─────────────────────┬──────────────────────────────┐│
│             │  │  FILA DE TRIAGEM    │  PAINEL DO CASO #002  [←Fila]││
│             │  │                     │                              ││
│             │  │ ┌───────────────────┐│ ⚠ Este caso não é o próximo ││  [A1]
│             │  │ │▓#001 PRÓXIMO FILA │ │ da fila. Case #001 aguarda  ││
│             │  │ │ Rua A, Há 3d  4/6 │ │ há mais tempo. [Ir →#001]  ││
│             │  │ └───────────────────┘│                              ││
│             │  │ ┌───────────────────┐│ ═══════════════════════════  ││
│             │  │ │ #002 Rua B  1d 2/6││                              ││
│             │  │ │ [selecionado ▓]   ││ [A2] TABS:                   ││
│             │  │ └───────────────────┘│ [Dossiê] [Adimplência]       ││
│             │  │ ┌───────────────────┐│                              ││
│             │  │ │ #003 Rua C  2d 6/6││ ── ABA DOSSIÊ ──────────────││
│             │  │ └───────────────────┘│ Progresso: ████████░░  4/6  ││
│             │  │ ┌───────────────────┐│                              ││
│             │  │ │ #004 Rua D  Bloq. ││ Doc 1: Contrato ✅ [Ver]    ││
│             │  │ │ [badge BLOQUEADO] ││ Doc 2: Comprov.  ✅ [Ver]   ││
│             │  │ └───────────────────┘│ Doc 3: Tabela    ✅ [Ver]   ││
│             │  │                     │ Doc 4: Instrumento✅ [Ver]   ││
│             │  │                     │ Doc 5: Declaração ⏳ [A3]   ││
│             │  │                     │ Doc 6: RG/CPF    ❌ Rejeitado││
│             │  │                     │                              ││
│             │  │                     │  [+ Adicionar documento]     ││
│             │  │                     │                              ││
│             │  │                     │ ─────────────────────────── ││
│             │  │                     │                              ││
│             │  │                     │  [A4] AÇÕES SIDEBAR:         ││
│             │  │                     │  [BLOQUEAR CASO]  (outline)  ││
│             │  │                     │  [QUALIFICAR CASO] (disabled)││
│             │  │                     │  [Solicitar documentos]      ││
│             │  └─────────────────────┴──────────────────────────────┘│
└─────────────┴────────────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Banner amarelo de alerta fora-de-ordem FIFO. Apenas informativo — não bloqueia ação.
- [A2] Tab Dossiê ativa por padrão. Tab Adimplência: checklist de parcelas.
- [A3] Documento pendente: aguarda envio pelo Cedente. Botão "Solicitar reenvio" (após rejeição).
- [A4] "Qualificar Caso" habilitado apenas quando 6/6 documentos ✅ + adimplência confirmada.

---

### 2.5 Bloco: Negociação

#### WF-T041/T042 — Negociação Painel + Timeline

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                               │
├─────────────┬────────────────────────────────────────────────────────┤
│  SIDEBAR    │  Negociação  /  Caso #045                [←Lista]      │
│             │  ─────────────────────────────────────────────────────│
│             │  ┌──────────────────────────────┬─────────────────────┐│
│             │  │  TIMELINE DE PROPOSTAS       │  INFO DO CASO [A1] ││
│             │  │                              │                     ││
│             │  │  [Ir para proposta ativa →]  │ Imóvel: Rua C, 100  ││
│             │  │  (skip link para a11y)        │ Cedente: João Silva ││
│             │  │                              │ Cenário: [C]        ││
│             │  │  ┌──────────────────────────┐│ SLA: 🟡 2 dias     ││
│             │  │  │ 12/01 · Proposta          │                     ││
│             │  │  │ Cessionário: Maria L.     │ AÇÕES:              ││
│             │  │  │ Valor: R$ 480.000         │                     ││
│             │  │  │ Status: [Expirada]        │ [ACEITAR PROPOSTA]  ││
│             │  │  └──────────────────────────┘│ (primário)          ││
│             │  │                              │                     ││
│             │  │  ┌──────────────────────────┐│ [Recusar/Contraprop]││
│             │  │  │ 15/01 · Contraproposta    │ (secondary)         ││
│             │  │  │ Analista: Ana S.           │                     ││
│             │  │  │ Valor: R$ 510.000         │ [Escalar Coordenador]││
│             │  │  │ Status: [Expirada]        │ (outline)           ││
│             │  │  └──────────────────────────┘│                     ││
│             │  │                              │ 3 propostas ativas  ││
│             │  │  ┌──────────────────────────┐│ [A2]                ││
│             │  │  │ 20/01 · Proposta ← [A3]  │                     ││
│             │  │  │ Cessionário: Carlos B.    │                     ││
│             │  │  │ Valor: R$ 520.000         │                     ││
│             │  │  │ Status: [▓ ATIVA ▓]       │                     ││
│             │  │  │ Prazo: 2d 14h restantes   │                     ││
│             │  │  └──────────────────────────┘│                     ││
│             │  └──────────────────────────────┴─────────────────────┘│
└─────────────┴────────────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Sidebar info: leitura, não ação. Ações principais em botões abaixo das informações.
- [A2] Badge de contagem de propostas ativas (de DEC-010).
- [A3] Proposta ativa: destaque com borda primária, fundo accent/20. Foco automático ao abrir (a11y).

---

#### WF-T043 — Modal Aceitar Proposta

```
                    ┌────────────────────────────────────────┐
                    │  Confirmar aceite da proposta    [×]   │
                    │  ════════════════════════════════════   │
                    │                                        │
                    │  Cessionário: Carlos Barbosa           │
                    │  Valor da proposta: R$ 520.000         │
                    │  Cenário do Cedente: C                 │
                    │                                        │
                    │  ── Comissões calculadas ──────────── │
                    │  Comissão Cedente:  R$ 36.000 (20%)   │
                    │  Comissão Cessionário: R$ 4.000 (20%) │
                    │  Valor líquido Cedente: R$ 484.000     │
                    │                                        │
                    │  [□ Confirmo os valores calculados]   │ ← [A1]
                    │                                        │
                    │  [CONFIRMAR ACEITE]  [Cancelar]       │
                    └────────────────────────────────────────┘
```

**Anotações:**
- [A1] Checkbox obrigatório para habilitar botão de confirmação.

---

### 2.6 Bloco: Formalização

#### WF-T051 — Painel de Formalização (4 critérios)

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                               │
├─────────────┬────────────────────────────────────────────────────────┤
│  SIDEBAR    │  Formalização  /  Caso #018                  [←Lista]  │
│             │  ─────────────────────────────────────────────────────│
│             │  [Banner contingência ZapSign se ativo] [A1]           │
│             │                                                        │
│             │  ┌─────────────────────────┬──────────────────────────┐│
│             │  │  4 CRITÉRIOS            │  INFO + AÇÕES            ││
│             │  │                         │                          ││
│             │  │ ┌──────────┬──────────┐ │  Caso: #018              ││
│             │  │ │ASSINATURAS│ ANUÊNCIA │ │  Imóvel: Av. Brasil, 500 ││
│             │  │ │          │          │ │  Cedente: Pedro M.        ││
│             │  │ │ 2/3 ✅  │  N/A ──  │ │  Valor: R$ 610.000       ││
│             │  │ │[A2]     │[A3]      │ │                          ││
│             │  │ │[Reenviar]│Não exigido│ │  ─────────────────────  ││
│             │  │ └──────────┴──────────┘ │                          ││
│             │  │                         │  Critérios: 3/3 ✅ [A4]  ││
│             │  │ ┌──────────┬──────────┐ │                          ││
│             │  │ │DEPÓSITO  │INSTRUMENTO│ │  [CONFIRMAR FECHAMENTO] ││
│             │  │ │  ESCROW  │          │ │  (desabilitado se <3/3)  ││
│             │  │ │          │          │ │                          ││
│             │  │ │ ✅ Conf. │ ✅ Assina│ │  [Prorrogar prazo]      ││
│             │  │ │ R$610k   │ ZapSign  │ │  (Coordenador)          ││
│             │  │ └──────────┴──────────┘ │                          ││
│             │  └─────────────────────────┴──────────────────────────┘│
└─────────────┴────────────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Banner contingência: amarelo, fixo no topo da área de trabalho (abaixo topbar).
- [A2] Card Assinaturas: barra de progresso fracionária 2/3. Badge por signatário.
- [A3] Card Anuência: badge cinza "N/A — Não exigido". Check cinza. Não bloqueia fechamento.
- [A4] Contador "3/3" (quando anuência N/A, conta apenas critérios aplicáveis).

---

#### WF-T055 — Modal Confirmar Fechamento

```
                    ┌────────────────────────────────────────────┐
                    │  Confirmar Fechamento             [×]      │
                    │  ════════════════════════════════════════   │
                    │                                            │
                    │  ✅ Assinaturas concluídas (3/3)          │
                    │  ✅ Anuência: N/A (não exigida)           │
                    │  ✅ Depósito confirmado: R$ 610.000       │
                    │  ✅ Instrumento assinado                   │
                    │                                            │
                    │  ── Distribuição prevista ─────────────── │
                    │  Cedente:    R$ 542.000                   │
                    │  Repasse Seguro: R$ 68.000 (comissão)     │
                    │  Data/hora:  22/03/2026 às 14:30          │
                    │                                            │
                    │  [A1] Para confirmar, digite: FECHAR       │
                    │  ┌──────────────────────────────────────┐ │
                    │  │ [input text: confirmar]              │ │
                    │  └──────────────────────────────────────┘ │
                    │                                            │
                    │  [CONFIRMAR FECHAMENTO]  [Cancelar]       │
                    │  (habilitado apenas com "FECHAR" digitado) │
                    └────────────────────────────────────────────┘
Estado Delta alto (aguardando Coordenador): [A2]
                    ┌────────────────────────────────────────────┐
                    │  Aprovação pendente do Coordenador        │
                    │  Delta ≥ R$ 100.000 — aprovação obrigatória│
                    │                                            │
                    │  Coordenador notificado: Ana Lima          │
                    │  Notificado às: 14:30                      │
                    │  SLA de resposta: 8 horas (22h30)         │
                    │  ████████████░░░░ 3h restantes            │ ← countdown
                    │                                            │
                    │  [OK, entendi]                             │
                    └────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Input de confirmação tipada. Botão habilitado ao digitar "FECHAR" (case insensitive).
- [A2] Estado de aguardo pós-submit quando Delta alto. Modal não fecha — transita para este estado.

---

### 2.7 Bloco: Financeiro

#### WF-T060 — Contas Escrow

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                               │
├─────────────┬────────────────────────────────────────────────────────┤
│  SIDEBAR    │  Financeiro  /  Contas Escrow           ● Ao vivo      │
│             │  ─────────────────────────────────────────────────────│
│             │  [Aberta] [Dep.Confirm.] [Em Distrib.] [Distribuída]   │
│             │  [Congelada] [Estornada]                               │
│             │  ─────────────────────────────────────────────────────│
│             │                                                        │
│             │  ┌──────────────────────────────────────────────────┐ │
│             │  │ ID    │ Status    │ V.Esperado│ V.Dep. │ D.Distr.│A│ │
│             │  │──────────────────────────────────────────────────│ │
│             │  │ #018  │ ✅ Dep.Conf│ R$610k    │ R$610k │ 01/04  │⋮│ │
│             │  │ #023  │ ⚠Prazo pr.│ R$480k    │ -      │ vence!  │⋮│ │ [A1]
│             │  │ #031  │ 🔴Prazo est│ R$320k    │ -      │ 3d atr. │⋮│ │ [A2]
│             │  │ #045  │ Aberta    │ R$200k    │ -      │ 15/04   │⋮│ │
│             │  └──────────────────────────────────────────────────┘ │
│             │                                                        │
│             │  [← 1/5 →]  25 por página                              │
└─────────────┴────────────────────────────────────────────────────────┘
Drawer ao clicar em linha: [A3]
┌────────────────────────────────┐
│ Conta Escrow — Caso #023  [×] │
│ ════════════════════════════   │
│ Imóvel: Rua B, 200             │
│ Valor esperado: R$ 480.000     │
│                                │
│ ─ Distribuição ──────────────  │
│ Cedente  ████████████ 80%      │
│ R$ 384.000                     │
│ Repasse S.████ 20%             │
│ R$ 96.000                      │
│                                │
│ ─ Histórico ─────────────────  │
│ 01/03 Conta aberta             │
│ 10/03 Prazo: 22/03             │
│ [ver mais]                     │
│                                │
│ [BLOQUEAR DISTRIBUIÇÃO]        │
│ [INICIAR REVERSÃO]             │
└────────────────────────────────┘
```

**Anotações:**
- [A1] "Prazo próximo" (≤3 dias): badge amarelo + tooltip com dias restantes.
- [A2] "Prazo estourado": badge vermelho + tooltip com dias em atraso.
- [A3] Drawer 480px desktop. Barras empilhadas visuais da distribuição.

---

### 2.8 Bloco: Supervisão IA

#### WF-T070 — Supervisão IA Painel

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                               │
├─────────────┬────────────────────────────────────────────────────────┤
│  SIDEBAR    │  Supervisão IA               Atualizado: 5s atrás [A1] │
│             │  ─────────────────────────────────────────────────────│
│             │                                                        │
│             │  ┌─────────────────────────────────────────────────┐  │
│             │  │  ┌──────────────────────┬──────────────────────┐ │  │
│             │  │  │ 🤖 Guardião Retorno  │ 🤖 Analista Opor.  │ │  │
│             │  │  │ [● Ativo]            │ [⚠ Falha técnica]  │ │  │ [A2]
│             │  │  │                      │                     │ │  │
│             │  │  │ Decisões 24h: 47     │ Erro: Sem resp.5min │ │  │
│             │  │  │ Taxa acerto: 94%      │ Últ.decisão: 2h     │ │  │
│             │  │  │ Últ.ação: 3min        │                     │ │  │
│             │  │  │                      │ [Reiniciar]         │ │  │
│             │  │  │ [Ver log]            │ [Takeover manual]   │ │  │
│             │  │  └──────────────────────┴──────────────────────┘ │  │
│             │  └─────────────────────────────────────────────────┘  │
│             │                                                        │
│             │  Decisões recentes (unificado)                         │
│             │  ┌──────────────────────────────────────────────────┐ │
│             │  │ Hora  │ Agente       │ Caso │ Decisão │ Confiança │ │
│             │  │ 14:25 │ Guardião     │ #045 │ Validado│  96%      │ │
│             │  │ 14:18 │ An.Opor.     │ #019 │ Match!  │  88%      │ │
│             │  │ 14:10 │ Guardião     │ #031 │ Alerta  │  72%      │ │
│             │  └──────────────────────────────────────────────────┘ │
└─────────────┴────────────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Timestamp de última atualização: resolução da DIV-001. Exibido no cabeçalho da seção.
- [A2] Estado falha técnica: badge vermelho, descrição do erro, botões de ação. Alerta enviado ao Master.

---

### 2.9 Bloco: Configurações

#### WF-T096 — Configurações — Comissões

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                               │
├─────────────┬────────────────────────────────────────────────────────┤
│  SIDEBAR    │  Configurações  /  Comissões e Parâmetros              │
│             │  ─────────────────────────────────────────────────────│
│             │  ⚠ Alterações afetam toda a operação. Registre o motivo│
│             │  ─────────────────────────────────────────────────────│
│             │                                                        │
│             │  Comissão do Cedente (%)      Comissão do Cessionário (%)│
│             │  ┌─────────────────────┐      ┌─────────────────────┐  │
│             │  │ [input: 20] %       │      │ [input: 20] %       │  │
│             │  │ Min: 5% | Max: 50%  │      │ Min: 5% | Max: 50%  │  │
│             │  └─────────────────────┘      └─────────────────────┘  │
│             │  ℹ Aplica-se aos casos nos cenários B, C e D.          │
│             │                                                        │
│             │  Limiar de revisão do Delta (R$)                       │
│             │  ┌─────────────────────┐                              │
│             │  │ [input: 100000]     │                              │
│             │  └─────────────────────┘                              │
│             │  ℹ Delta acima deste valor requer aprovação do Coord.  │
│             │                                                        │
│             │  Valor Distrato Referência (%)                         │
│             │  ┌─────────────────────┐                              │
│             │  │ [input: 50] %       │                              │
│             │  └─────────────────────┘                              │
│             │                                                        │
│             │  ═══════════════════════════════════════════════════   │
│             │  Motivo da alteração (obrigatório):                    │
│             │  ┌──────────────────────────────────────────────────┐ │
│             │  │ [textarea: descreva o motivo...]                 │ │
│             │  │                               mín. 20 caracteres │ │
│             │  └──────────────────────────────────────────────────┘ │
│             │                                                        │
│             │  [SALVAR CONFIGURAÇÕES]   [Restaurar padrões]         │
└─────────────┴────────────────────────────────────────────────────────┘
```

---

## 3. Fluxos de Navegação Detalhados

### FLOW-001: Autenticação completa
```
T-001 (Login)
  → credenciais corretas + 2FA ativo → T-002 (2FA)
  → código correto → T-010 (Dashboard)
  → credenciais incorretas → T-001 (erro inline)
  → 5 falhas → T-001 (bloqueio 30min)
  → "Esqueci senha" → T-003 → e-mail enviado → T-004 → T-001
```

### FLOW-002: Triagem completa
```
T-010 (Dashboard) → [Ir para Triagem]
  → T-030 (Fila) → seleciona caso
  → T-031 (Painel) → T-032 (Dossiê)
  → verificar 6 docs → T-033 (Adimplência)
  → confirmar adimplência → "Qualificar" habilitado
  → clicar "Qualificar" → confirmação inline → caso: Qualificado
  → alerta: "aguarda publicação pelo Coordenador"

  OU: parcela em atraso → clicar "Bloquear Caso" → T-034 (Modal)
  → confirmar → caso: Bloqueado → toast "Cedente notificado"
```

### FLOW-003: Formalização → Fechamento
```
T-050 (Lista) → seleciona caso
  → T-051 (Painel) → preenche 4 cards
  → T-052 (enviar para assinatura) → aguarda webhook ZapSign
  → T-053 (confirmar anuência / N/A)
  → T-054 (aguarda depósito Celcoin)
  → 3/3 ou 4/4 critérios ✅ → botão habilitado
  → T-055 (Modal) → digita "FECHAR" → confirma
  → caso: Pós-Fechamento → toast + redirect T-050
```

---

## 4. Especificações de Componentes Globais

### C-11 — Topbar

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ████ Repasse Seguro  |  Dashboard  /  Triagem  /  Caso #001    🔔(3)  ████ A.S.▼│
│  [logo 32px]          [breadcrumb — separado por "/" ]          [sino] [avatar]  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

Dropdown avatar: "Meu perfil" · "Alterar senha" · "═══" · "Sair"
Dropdown notificações: lista das 5 últimas, "Ver todas →"

### C-12 — Sidebar (estados)

```
Expandida (w-64):          Colapsada (w-16):
┌──────────────────┐       ┌────┐
│ ▓ Dashboard      │       │ ▓ 🏠│
│ ○ Pipeline       │       │ ○ ◉│
│ ○ Triagem     (4)│       │ ○(4│
│ ○ Negociação     │       │ ○  │
│ ...              │       │... │
└──────────────────┘       └────┘
```
Badge numérico (vermelho) em itens com itens pendentes de atenção.

### C-07 — ConfirmModal (tamanhos)

```
sm (max-w-sm):           md (max-w-md):           lg (max-w-lg):
┌────────────────────┐   ┌────────────────────────┐   ┌──────────────────────────────┐
│ Título             │   │ Título                 │   │ Título                       │
│                    │   │                        │   │                              │
│ Mensagem curta     │   │ Mensagem com detalhes  │   │ Conteúdo extenso com tabelas │
│                    │   │ e campos adicionais    │   │ e resumos financeiros        │
│ [OK] [Cancelar]   │   │ [Confirmar] [Cancelar] │   │ [Confirmar com texto] [Canc.]│
└────────────────────┘   └────────────────────────┘   └──────────────────────────────┘
```

---

## 5. Estados de Interface Globais

### Sessão Expirada
```
┌────────────────────────────────────┐
│  Sua sessão expirou               │
│  Faça login novamente para        │
│  continuar.                       │
│                                   │
│  [FAZER LOGIN]                    │
└────────────────────────────────────┘
(Modal sobre toda a interface, não dismissível)
```

### Offline Banner
```
┌──────────────────────────────────────────────────────────────────────┐
│  📶 Sem conexão — dados podem estar desatualizados.   [×] Ignorar   │
└──────────────────────────────────────────────────────────────────────┘
(Banner fixo abaixo do Topbar, amarelo, persistente até restaurar conexão)
```

### Erro 500
```
┌──────────────────────────────────────────────────────────────────────┐
│  Algo deu errado                                                     │
│  Erro #ERR-20260322-001                                              │
│  Nosso time foi notificado automaticamente.                          │
│                                                                      │
│  [VOLTAR AO INÍCIO]    [Contatar suporte]                            │
└──────────────────────────────────────────────────────────────────────┘
```

---

*Documento gerado por Claude Code Desktop — Pipeline ShiftLabs v9.5 — 2026-03-22*
