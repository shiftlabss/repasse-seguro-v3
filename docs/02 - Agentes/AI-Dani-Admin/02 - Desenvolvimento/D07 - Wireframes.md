# Wireframes — AI-Dani-Admin

## Módulo de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Design e Engenharia Frontend |
| Escopo | Wireframes de baixa fidelidade para todas as telas do módulo AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D06 (T-001 a T-007, COMP-001, COMP-002), D05 (RFs) |

---

> **📌 TL;DR**
>
> - Wireframes textuais para as 7 telas + 2 componentes globais do módulo AI-Dani-Admin.
> - Estrutura de layout, hierarquia de componentes e comportamento de estados por tela.
> - Todos os wireframes são para o painel Admin (acesso exclusivo) na interface web.
> - Nomenclatura de componentes segue shadcn/ui (D02/D03).
> - Toda tela herda o layout base do painel Admin da plataforma Repasse Seguro (sidebar + topbar).

---

## 1. Layout Base — Painel Admin

Todas as telas do módulo AI-Dani-Admin herdam este layout:

```
┌─────────────────────────────────────────────────────────────┐
│ TOPBAR                                                      │
│ [Logo Repasse Seguro]    [Notificações] [Avatar Admin]      │
├──────────────┬──────────────────────────────────────────────┤
│ SIDEBAR      │ CONTEÚDO PRINCIPAL                          │
│              │                                              │
│ Dashboard    │ <conteúdo da tela ativa>                    │
│ Usuários     │                                              │
│ > Supervisão IA (ativo)                                     │
│   ∟ Interações                                              │
│   ∟ Métricas                                                │
│   ∟ Configurações                                           │
│   ∟ Auditoria                                               │
│   ∟ Checklist                                               │
│ Financeiro   │                                              │
│ Configurações│                                              │
└──────────────┴──────────────────────────────────────────────┘
```

**Componentes do layout base:**
- `AppSidebar` (shadcn/ui Sidebar): navegação principal do painel Admin.
- `SidebarTrigger`: toggle de collapse da sidebar.
- `Breadcrumb` (shadcn/ui): exibe rota atual no topbar.

---

## 2. T-001 — Painel Supervisão IA (Lista de Interações)

```
┌─────────────────────────────────────────────────────────────┐
│ TOPBAR [Logo] [Breadcrumb: Supervisão IA > Interações]      │
├──────────┬──────────────────────────────────────────────────┤
│ SIDEBAR  │ HEADER DA TELA                                   │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ H1: "Supervisão IA"                          │ │
│          │ │ Badge: "3 sinalizadas para revisão"          │ │
│          │ │                       [Botão: "Filtros ▼"]   │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ BARRA DE FILTROS (quando filtros ativos)         │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ Chip[Dani-Cessionário ×] Chip[Último dia ×]  │ │
│          │ │                    [Limpar filtros]           │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ LISTA DE INTERAÇÕES                              │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ TABLE HEADER                                 │ │
│          │ │ [Usuário] [Agente] [Confiança] [Latência] [Data] [Status] │
│          │ ├──────────────────────────────────────────────┤ │
│          │ │ ROW 1 (normal)                               │ │
│          │ │ ID-****23  Dani-Cess.  94%  1.2s  12:34  [Respondida] │
│          │ ├──────────────────────────────────────────────┤ │
│          │ │ ROW 2 (sinalizada — bg warning-subtle, border-left warning) │
│          │ │ ID-****45  Dani-Cess.  61%  2.1s  12:31  [⚠ Sinalizada] │
│          │ ├──────────────────────────────────────────────┤ │
│          │ │ ROW 3 (em takeover — bg takeover-subtle, border-left takeover) │
│          │ │ ID-****67  Dani-Ced.   --   --    12:28  [Em takeover] │
│          │ ├──────────────────────────────────────────────┤ │
│          │ │ ROW 4 (encerrada — bg normal)                │ │
│          │ │ ID-****89  Dani-Cess.  88%  0.9s  12:10  [Encerrada] │
│          │ └──────────────────────────────────────────────┘ │
│          │ [Paginação: < 1 2 3 ... >]                       │
└──────────┴──────────────────────────────────────────────────┘
```

**Estado vazio:**
```
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │                                              │ │
│          │ │          [ícone ilustrativo]                 │ │
│          │ │  "Nenhuma interação registrada no período    │ │
│          │ │   selecionado."                              │ │
│          │ │  "Tente ajustar o período ou os filtros      │ │
│          │ │   aplicados."                                │ │
│          │ │                                              │ │
│          │ └──────────────────────────────────────────────┘ │
```

**Estado loading (skeleton):**
```
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ [████████████] [███████] [███] [██] [██████] │ │
│          │ │ [████████████] [███████] [███] [██] [██████] │ │
│          │ │ ... (8 linhas skeleton com shimmer)          │ │
│          │ └──────────────────────────────────────────────┘ │
```

---

## 3. T-002 — Detalhe de Interação

```
┌─────────────────────────────────────────────────────────────┐
│ TOPBAR [Logo] [Breadcrumb: Supervisão IA > Interações > #45]│
├──────────┬──────────────────────────────────────────────────┤
│ SIDEBAR  │ HEADER DA TELA                                   │
│          │ [← Voltar]  "Interação #ID-****45"  [⚠ Sinalizada] │
│          │                                                  │
│          │ GRID 2 COLUNAS                                   │
│          │ ┌─────────────────────┬────────────────────────┐ │
│          │ │ COLUNA ESQUERDA     │ COLUNA DIREITA         │ │
│          │ │ (2/3 da largura)    │ (1/3 da largura)       │ │
│          │ │                     │                        │ │
│          │ │ CHAT HISTÓRICO      │ MÉTRICAS               │ │
│          │ │ ┌─────────────────┐ │ ┌────────────────────┐ │ │
│          │ │ │ [Avatar IA]     │ │ │ Card: Confiança    │ │ │
│          │ │ │ Mensagem IA     │ │ │      61%           │ │ │
│          │ │ │                 │ │ │      ⚠ Abaixo do  │ │ │
│          │ │ │      Mensagem   │ │ │      threshold 80% │ │ │
│          │ │ │      Usuário[U] │ │ ├────────────────────┤ │ │
│          │ │ │                 │ │ │ Card: Latência     │ │ │
│          │ │ │ [Avatar IA]     │ │ │      2.1s          │ │ │
│          │ │ │ Resposta IA     │ │ ├────────────────────┤ │ │
│          │ │ └─────────────────┘ │ │ Card: Agente       │ │ │
│          │ │                     │ │   Dani-Cessionário  │ │ │
│          │ │                     │ ├────────────────────┤ │ │
│          │ │                     │ │ Card: Data/Hora    │ │ │
│          │ │                     │ │   12:31 23/03/2026 │ │ │
│          │ │                     │ └────────────────────┘ │ │
│          │ │                     │                        │ │
│          │ │                     │ AÇÃO                   │ │
│          │ │                     │ ┌────────────────────┐ │ │
│          │ │                     │ │ [Assumir conversa] │ │ │
│          │ │                     │ │  (btn primary)     │ │ │
│          │ │                     │ └────────────────────┘ │ │
│          │ └─────────────────────┴────────────────────────┘ │
└──────────┴──────────────────────────────────────────────────┘
```

**Estado: interação encerrada**
```
│          │ [← Voltar]  "Interação #ID-****89"  [Encerrada]  │
│          │ ...                                              │
│          │ [Assumir conversa — desabilitado]               │
│          │  tooltip: "Conversa encerrada"                  │
```

---

## 4. T-003 — Chat em Takeover (Admin View)

```
┌─────────────────────────────────────────────────────────────┐
│ TOPBAR [Logo] [Breadcrumb: Supervisão IA > Takeover #45]    │
├──────────┬──────────────────────────────────────────────────┤
│ SIDEBAR  │ HEADER DE TAKEOVER                               │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ 🔵 "Atendimento humano ativo"                 │ │
│          │ │ Usuário: ID-****45                            │ │
│          │ │ Agente pausado — você está respondendo        │ │
│          │ │                   [Encerrar atendimento humano] │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ ÁREA DO CHAT                                     │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ [Avatar IA Dani] "Qual é a minha comissão?"  │ │
│          │ │ [Avatar IA Dani] Resposta da IA (confiança 61%) │ │
│          │ │                                              │ │
│          │ │ ────────── Atendimento humano ──────────     │ │
│          │ │                                              │ │
│          │ │ [Avatar Pessoa] "Um analista da equipe       │ │
│          │ │ "Repasse Seguro assumiu essa conversa..."    │ │
│          │ │                                              │ │
│          │ │                   [Usuário: Obrigado!]   [U] │ │
│          │ │                                              │ │
│          │ │ [Avatar Pessoa] "Olá! Posso ajudá-lo com    │ │
│          │ │ mais detalhes sobre sua comissão."           │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ CAMPO DE DIGITAÇÃO                               │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ [Digite sua mensagem...]          [Enviar →]  │ │
│          │ └──────────────────────────────────────────────┘ │
└──────────┴──────────────────────────────────────────────────┘
```

---

## 5. T-004 — Dashboard de Métricas

```
┌─────────────────────────────────────────────────────────────┐
│ TOPBAR [Logo] [Breadcrumb: Supervisão IA > Métricas]        │
├──────────┬──────────────────────────────────────────────────┤
│ SIDEBAR  │ HEADER + FILTROS                                 │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ H1: "Dashboard de Métricas"                  │ │
│          │ │ [Tabs: Dia | Semana | Mês]  [▾ Dani-Cess.]   │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ GRID DE CARDS (5 cards)                          │
│          │ ┌────────────┐ ┌────────────┐ ┌────────────────┐ │
│          │ │ Volume     │ │ CSAT Médio │ │ Taxa de Recusa │ │
│          │ │ interações │ │            │ │                │ │
│          │ │ 📊 1.247   │ │ ⭐ 4.2/5  │ │ 🚫 8.3%        │ │
│          │ │ +12% vs sem│ │ -0.3 pts   │ │ ↑ do esperado  │ │
│          │ └────────────┘ └────────────┘ └────────────────┘ │
│          │ ┌────────────┐ ┌───────────────────────────────┐ │
│          │ │ Latência   │ │ (espaço reservado para 5º     │ │
│          │ │ Média      │ │  card — Top Perguntas link)    │ │
│          │ │ ⏱ 1.8s    │ │                               │ │
│          │ └────────────┘ └───────────────────────────────┘ │
│          │                                                  │
│          │ GRÁFICO DE VOLUME                                │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ [Gráfico de barras: Volume por dia da semana]│ │
│          │ │  Azul: Dani-Cessionário                      │ │
│          │ │  Azul escuro: Dani-Cedente                   │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ TOP 10 PERGUNTAS                                 │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ 1. "Qual é minha comissão?"        (287x)    │ │
│          │ │ 2. "Como funciona o Escrow?"       (198x)    │ │
│          │ │ ... (até 10 perguntas)                       │ │
│          │ └──────────────────────────────────────────────┘ │
└──────────┴──────────────────────────────────────────────────┘
```

**Estado de card sem dados:**
```
│          │ ┌────────────┐                                   │
│          │ │ CSAT Médio │                                   │
│          │ │            │                                   │
│          │ │ ℹ "Dados   │                                   │
│          │ │   insufici-│                                   │
│          │ │   entes"   │                                   │
│          │ └────────────┘                                   │
```

---

## 6. T-005 — Configurações de Supervisão

```
┌─────────────────────────────────────────────────────────────┐
│ TOPBAR [Logo] [Breadcrumb: Supervisão IA > Configurações]   │
├──────────┬──────────────────────────────────────────────────┤
│ SIDEBAR  │ H1: "Configurações de Supervisão"                │
│          │                                                  │
│          │ SEÇÃO: Threshold de Confiança                    │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ "Nível de supervisão automática"             │ │
│          │ │ Interações com confiança abaixo deste valor  │ │
│          │ │ serão sinalizadas para revisão.              │ │
│          │ │                                              │ │
│          │ │ Valor atual: 80%                             │ │
│          │ │ ├──────────●──────────────────────────┤     │ │
│          │ │   50%             80%                 95%   │ │
│          │ │                                              │ │
│          │ │ [  Campo numérico: 80 ]  %                   │ │
│          │ │                                              │ │
│          │ │               [Salvar configuração]          │ │
│          │ │                                              │ │
│          │ │ (estado de erro):                            │ │
│          │ │ ⚠ "O nível de supervisão precisa estar      │ │
│          │ │   entre 50% e 95%."                         │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ SEÇÃO: Rate Limit do Webchat                     │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ "Limite de mensagens por usuário (por hora)" │ │
│          │ │ [ Campo: 30 ] mensagens/hora                 │ │
│          │ │               [Salvar]                       │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ SEÇÃO: Histórico de Alterações                   │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ Data         │ Admin    │ Alteração           │ │
│          │ │ 23/03 12:31  │ Admin-01 │ Threshold: 85%→80% │ │
│          │ │ 22/03 09:14  │ Admin-01 │ Threshold: 80%→85% │ │
│          │ └──────────────────────────────────────────────┘ │
└──────────┴──────────────────────────────────────────────────┘
```

---

## 7. T-006 — Log de Auditoria

```
┌─────────────────────────────────────────────────────────────┐
│ TOPBAR [Logo] [Breadcrumb: Supervisão IA > Auditoria]       │
├──────────┬──────────────────────────────────────────────────┤
│ SIDEBAR  │ HEADER DA TELA                                   │
│          │ H1: "Log de Auditoria"                           │
│          │                                                  │
│          │ FILTROS                                          │
│          │ [▾ Tipo de ação] [📅 Período] [▾ Admin]          │
│          │                                                  │
│          │ TABELA DE EVENTOS                                │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ Data/Hora      │ Admin   │ Ação     │ Detalhes│ │
│          │ │ 23/03 12:34    │ Adm-01  │ Takeover │ Int.#45 │ │
│          │ │ 23/03 12:31    │ Adm-01  │ Threshold│ 85%→80% │ │
│          │ │ 23/03 12:28    │ Adm-01  │ Acesso   │ Lista   │ │
│          │ │ 23/03 11:15    │ Sistema │ Alerta   │ CSAT<3.5│ │
│          │ │ 23/03 11:05    │ Adm-02  │ Reativou │ Dani-C. │ │
│          │ └──────────────────────────────────────────────┘ │
│          │ [Paginação: < 1 2 3 ... >]                       │
└──────────┴──────────────────────────────────────────────────┘
```

---

## 8. T-007 — Checklist de Prontidão para Lançamento

```
┌─────────────────────────────────────────────────────────────┐
│ TOPBAR [Logo] [Breadcrumb: Supervisão IA > Checklist]       │
├──────────┬──────────────────────────────────────────────────┤
│ SIDEBAR  │ HEADER DA TELA                                   │
│          │ H1: "Checklist de Prontidão para Lançamento"     │
│          │ Agente alvo: [▾ Dani-Cessionário]                │
│          │ Status: ❌ "Bloqueado — 3 itens pendentes"       │
│          │                                                  │
│          │ GRUPO 1: Isolamento de Acesso                    │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ ✅ Filtro de escopo implementado e testado    │ │
│          │ │ ✅ Filtro de contexto implementado e testado  │ │
│          │ │ ❌ Teste de penetração: acesso cruzado 100%   │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ GRUPO 2: Cobertura de Recusa                     │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ ✅ Identidade e tom do agente definidos       │ │
│          │ │ ✅ Dados bloqueados com exemplos de recusa    │ │
│          │ │ ❌ Exemplos de perguntas recusadas validados  │ │
│          │ │ ✅ Formato de resposta com próximo passo      │ │
│          │ │                                              │ │
│          │ │ Testes adversariais: 0 / 20 aprovados        │ │
│          │ │ [Campo: Resultado dos testes adversariais]   │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ GRUPO 3: Supervisão Admin Funcional              │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ ✅ Registro de interações configurado         │ │
│          │ │ ✅ Dashboard de métricas operacional          │ │
│          │ │ ✅ Alerta automático de confiança ativo       │ │
│          │ │ ❌ Takeover manual testado com registro       │ │
│          │ └──────────────────────────────────────────────┘ │
│          │                                                  │
│          │ ┌──────────────────────────────────────────────┐ │
│          │ │ [Autorizar lançamento] ← desabilitado        │ │
│          │ │ Habilita somente quando 100% dos itens ✅    │ │
│          │ └──────────────────────────────────────────────┘ │
└──────────┴──────────────────────────────────────────────────┘
```

**Estado: pronto para lançamento**
```
│          │ Status: ✅ "Pronto para lançamento"             │
│          │ [Autorizar lançamento] ← habilitado (btn primary)│
```

---

## 9. COMP-001 — Banner de Alerta Crítico (Global)

```
┌─────────────────────────────────────────────────────────────┐
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ 🔴 [bell-ring] DESLIGAMENTO AUTOMÁTICO                   │ │
│ │ Dani-Cessionário foi desligado automaticamente.          │ │
│ │ Taxa de erro: 32% nos últimos 15 minutos.                │ │
│ │ [Reativar agente]    [Ver detalhes]          [✕]         │ │
│ └──────────────────────────────────────────────────────────┘ │
│ ... (resto da tela abaixo do banner)                        │
└─────────────────────────────────────────────────────────────┘
```

**Notas de implementação:**
- Banner posicionado no topo do conteúdo principal (abaixo do topbar, acima do conteúdo da tela).
- Cor de fundo: derivada de `--destructive` com opacidade (background sutil).
- Borda esquerda: 4px `--destructive` sólida.
- Fechar (✕) disponível somente após ação tomada OU confirmação explícita "Vou investigar depois".
- Pulso de borda desabilitado quando `prefers-reduced-motion: reduce`.

---

## 10. COMP-002 — Separador Visual de Takeover (no chat do usuário)

```
VISÃO DO CHAT DO USUÁRIO (Cessionário/Cedente):

┌──────────────────────────────────────────────────────────┐
│ [Avatar IA] Dani-Cessionário                              │
│ "Sua comissão estimada é de R$ 1.200..."                  │
│                                                          │
│ [Usuário] "Não entendi, pode explicar melhor?"           │
│                                                          │
│ ─────────────── Atendimento humano ───────────────       │
│                                                          │
│ [Avatar Pessoa] Equipe Repasse Seguro                     │
│ "Um analista da equipe Repasse Seguro assumiu essa       │
│  conversa para ajudá-lo. Como posso ajudar?"             │
│                                                          │
│ [Usuário] "Queria entender o cálculo de comissão..."     │
│                                                          │
│ [Avatar Pessoa] Equipe Repasse Seguro                     │
│ "Claro! Vou explicar passo a passo..."                   │
│                                                          │
│ ─────────── Retorno ao atendimento automatizado ──────   │
│                                                          │
│ [Avatar IA] Dani-Cessionário                              │
│ "Você está novamente em atendimento com Dani."           │
└──────────────────────────────────────────────────────────┘
```

---

## 11. Modais de Confirmação

### 11.1 Modal: Confirmação de Takeover

```
┌──────────────────────────────────────┐
│ "Assumir conversa"                   │
│                                      │
│ Você está prestes a assumir o        │
│ atendimento da interação #ID-****45. │
│ O agente será pausado imediatamente. │
│                                      │
│ Motivo (opcional):                   │
│ ┌──────────────────────────────────┐ │
│ │ [Selecionar motivo ▾]            │ │
│ └──────────────────────────────────┘ │
│                                      │
│ [Cancelar]    [Assumir conversa →]   │
└──────────────────────────────────────┘
```

### 11.2 Modal: Confirmação de Encerramento de Takeover

```
┌──────────────────────────────────────┐
│ "Encerrar atendimento humano"        │
│                                      │
│ O agente Dani-Cessionário irá        │
│ retomar o atendimento desta          │
│ conversa.                            │
│                                      │
│ [Cancelar]    [Encerrar atendimento] │
└──────────────────────────────────────┘
```

### 11.3 Modal: Reativação de Agente

```
┌──────────────────────────────────────┐
│ "Reativar agente"                    │
│                                      │
│ Você está reativando o               │
│ Dani-Cessionário após desligamento   │
│ automático por taxa de erro elevada. │
│                                      │
│ ⚠ Confirme que o problema foi        │
│ investigado antes de reativar.       │
│                                      │
│ [Cancelar]    [Reativar agente]      │
└──────────────────────────────────────┘
```

---

## 12. Rastreabilidade Wireframes → Telas → RFs

| Wireframe | Tela | RFs Cobertos |
|---|---|---|
| Seção 2 | T-001 | RF-001, RF-002, RF-003, RF-004, RF-007 |
| Seção 3 | T-002 | RF-001, RF-007, RF-008, RF-010 |
| Seção 4 | T-003 | RF-008, RF-009, RF-011 |
| Seção 5 | T-004 | RF-012, RF-013 |
| Seção 6 | T-005 | RF-014, RF-015, RF-016, RF-019 |
| Seção 7 | T-006 | RF-004, RF-005, RNF-005 |
| Seção 8 | T-007 | RF-020, RF-021, RF-022, RF-023, RF-024, RF-025 |
| Seção 9 | COMP-001 | RF-005, RF-006 |
| Seção 10 | COMP-002 | RF-008, RF-009 |
| Seção 11 | Modais | RF-008, RF-009, RF-005 |

---

## 13. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Wireframes textuais para todas as 7 telas + 2 componentes globais + 3 modais de confirmação. |
