# 07 - Wireframes

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Design, Frontend e QA |
| **Escopo** | Wireframes de baixa fidelidade das telas prioritárias, anotações de layout e comportamento, especificações de componentes e fluxos de navegação |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 (America/Fortaleza) |
| **Dependências** | 06 - Mapa de Telas · 01.1–01.5 Regras de Negócio · 05.1–05.5 PRD |

---

> **TL;DR**
>
> - Wireframes de **8 telas prioritárias** organizados por bloco funcional.
> - Representação ASCII-style com seções estruturadas + anotações descritivas.
> - Fidelidade: baixa (sem cores finais, sem tipografia final, sem imagens reais).
> - Foco: estrutura, hierarquia de informação, posicionamento de componentes, estados e anotações de comportamento.
> - Telas priorizadas: Login, Pipeline (Kanban), Detalhes do Caso, Negociação, Dossiê, Contatos, SLA Monitor, Comunicação (WhatsApp).

---

## 1. Convenções de Wireframe

### 1.1 Notação

```
┌─────────┐  = Container / Card / Modal
│         │  = Área de conteúdo
└─────────┘

[ BOTÃO ]    = Botão de ação primária
[ botão ]    = Botão de ação secundária
[ campo ]    = Campo de entrada de texto
[▼ select]   = Dropdown / Select
[□ item]     = Checkbox ou item selecionável
[● radio]    = Radio button

████████     = Imagem / Avatar placeholder
░░░░░░░░     = Skeleton loading / área cinza
───────────  = Linha divisória / separador
→            = Navegação para outra tela
⚠            = Alerta / aviso
ℹ            = Tooltip / informação
🔴           = Indicador vermelho (SLA vencido, erro)
🟡           = Indicador amarelo (SLA em risco, aviso)
🟢           = Indicador verde (dentro do prazo, ok)
```

### 1.2 Anotações

Anotações seguem o formato:
```
[A1] Texto da anotação
```
Referenciadas no wireframe com `[A1]`, `[A2]`, etc.

### 1.3 Layout Shell (presente em todas as telas logadas)

```
┌──────────────────────────────────────────────────────────────────────┐
│ TOPBAR                                                               │
│ ████  Repasse Seguro · CRM     [ Buscar... Ctrl+K ]   🔔 3  ████▼  │
├────────────┬─────────────────────────────────────────────────────────┤
│ SIDEBAR    │ ÁREA DE TRABALHO                                        │
│            │                                                         │
│ Dashboard  │                                                         │
│ Pipeline   │                                                         │
│ Negociação │                                                         │
│ Dossiê     │                                                         │
│ Contatos   │                                                         │
│ Atividades │                                                         │
│ Comunic.   │                                                         │
│ SLA        │                                                         │
│ ─────────  │                                                         │
│ Equipe     │                                                         │
│ Relatórios │                                                         │
│ Config.    │                                                         │
└────────────┴─────────────────────────────────────────────────────────┘
```

[A1] Topbar: altura h-14. Logo + busca global central + notificações + avatar.
[A2] Sidebar: w-64 desktop, colapsável para w-16 (ícones) abaixo de 1280px.
[A3] Item ativo: fundo --accent, borda esquerda 4px --primary.
[A4] Área de trabalho: flex-1, scroll vertical, padding p-6.

---

## 2. Wireframes por Bloco

---

### 2.1 WF-T001 — Login / Autenticação

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│                    ████████████████                      │
│                 [Logo Repasse Seguro]                    │
│                   CRM — Ferramenta Interna               │
│                                                          │
│   ┌──────────────────────────────────────────────────┐   │
│   │                                                  │   │
│   │   E-mail                                         │   │
│   │   ┌──────────────────────────────────────────┐   │   │
│   │   │ [ email@repasse.com.br              ]    │   │   │  [A1]
│   │   └──────────────────────────────────────────┘   │   │
│   │                                                  │   │
│   │   Senha                                          │   │
│   │   ┌──────────────────────────────────────────┐   │   │
│   │   │ [ ••••••••••••••••••••        ] [👁]     │   │   │  [A2]
│   │   └──────────────────────────────────────────┘   │   │
│   │                                                  │   │
│   │   ┌──────────────────────────────────────────┐   │   │
│   │   │              [ ENTRAR ]                  │   │   │  [A3]
│   │   └──────────────────────────────────────────┘   │   │
│   │                                                  │   │
│   │              [ Esqueci minha senha ] →T-002      │   │
│   │                                                  │   │
│   └──────────────────────────────────────────────────┘   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Estado: erro de credencial**
```
   │ Senha                                                │
   │ ┌─────────────────────────────────────────────────┐ │
   │ │ [ ••••••••••••••         ]              [👁]    │ │
   │ └─────────────────────────────────────────────────┘ │
   │ ⚠ E-mail ou senha incorretos.                       │  [A4]
```

**Estado: usuário suspenso**
```
   ┌──────────────────────────────────────────────────────┐
   │ ⚠  Seu acesso foi temporariamente suspenso.          │
   │    Entre em contato com seu gestor.                  │
   └──────────────────────────────────────────────────────┘
```

**Estado: sessão expirada**
```
   ┌──────────────────────────────────────────────────────┐
   │ ℹ  Sua sessão expirou. Faça login novamente.         │
   └──────────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Campo e-mail: type="email", autocomplete="email", autofocus.
- [A2] Botão 👁 alterna entre password/text. Sem mostrar senha por padrão.
- [A3] Botão "ENTRAR": primário, full-width, desabilitado enquanto campos vazios.
- [A4] Mensagem de erro: abaixo do campo de senha, texto destructive, sem indicar qual campo está errado (segurança).

---

### 2.2 WF-T020 — Pipeline — Visão Kanban

```
┌────────────┬─────────────────────────────────────────────────────────┐
│  SIDEBAR   │  Pipeline                                               │
│            │  ─────────────────────────────────────────────────────  │
│            │  [ Kanban ] [ Lista ]    [ + Novo Caso ]  [ Filtros ▼]  │
│            │                                                         │
│            │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│            │  │ CADASTRO │ │VERIFICAÇÃO│ │PUBLICAÇÃO│ │NEGOCIAÇÃO│  │  [A1]
│            │  │    4     │ │    7     │ │    3     │ │    2     │  │
│            │  ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤  │
│            │  │┌────────┐│ │┌────────┐│ │┌────────┐│ │┌────────┐│  │
│            │  ││RS-0042 ││ ││RS-0031 ││ ││RS-0028 ││ ││RS-0019 ││  │
│            │  ││Silva F.││ ││Pereira ││ ││Cardoso ││ ││Alves J.││  │
│            │  ││Lumina  ││ ││Park RS ││ ││View RS ││ ││Lumina  ││  │
│            │  ││🟢 D+1  ││ ││🔴 D+8  ││ ││🟢 D+2  ││ ││🟡 D+6  ││  │  [A2]
│            │  │└────────┘│ │└────────┘│ │└────────┘│ │└────────┘│  │
│            │  │┌────────┐│ │┌────────┐│ │          │ │          │  │
│            │  ││RS-0043 ││ ││RS-0033 ││ │  ...     │ │          │  │
│            │  ││Costa M.││ ││Lima T. ││ │          │ │          │  │
│            │  ││View RS ││ ││Alpha   ││ │          │ │          │  │
│            │  ││🟢 D+0  ││ ││🟡 D+5  ││ │          │ │          │  │
│            │  │└────────┘│ │└────────┘│ │          │ │          │  │
│            │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│            │  ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│            │  │ANUÊNCIA  │ │FORMALI-  │ │CONCLUÍDO │              │
│            │  │    1     │ │ZAÇÃO  2  │ │    8     │              │
│            │  └──────────┘ └──────────┘ └──────────┘              │
└────────────┴─────────────────────────────────────────────────────────┘
```

**Estado: filtro ativo**
```
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Filtro ativo: SLA Vencido · Analista: João Silva    [✕ Limpar] │  │
│  └──────────────────────────────────────────────────────────────┘   │
```

**Estado: Pipeline vazio (Analista RS)**
```
│                                                                      │
│              ┌─────────────────────────────────┐                    │
│              │     Nenhum Caso ativo por aqui.  │                    │
│              │   [ + Criar primeiro Caso ]      │                    │
│              └─────────────────────────────────┘                    │
```

**Anotações:**
- [A1] Colunas do kanban: Cadastro · Verificação · Publicação · Match · Negociação · Anuência · Formalização · Concluído · Cancelado (oculto por padrão).
- [A2] Badge de SLA: 🟢 D+N (dias no estado, dentro do prazo) · 🟡 D+N (>80% SLA) · 🔴 D+N (SLA vencido). Badge especial "⚠ Risco" para >150% SLA.
- [A3] Card do kanban: ID do Caso (RS-XXXX) · nome do Cedente mascarado · empreendimento · badge de SLA. Click → T-CRM-022.
- [A4] Analista RS: vê apenas Casos próprios por padrão. Toggle "Ver equipe" disponível.
- [A5] Botão "+ Novo Caso": disponível para todos os perfis logados.

---

### 2.3 WF-T022 — Detalhes do Caso

```
┌────────────┬─────────────────────────────────────────────────────────┐
│  SIDEBAR   │  Pipeline › RS-2026-0042                                │
│            │  ─────────────────────────────────────────────────────  │
│            │  🟡 NEGOCIAÇÃO  │  Analista: João Silva  │  🟡 D+6 SLA │
│            │                                                         │
│            │  Silva F. — Lumina Park (Bloco A, Ap. 42)               │
│            │  Tabela Contrato: R$ 380.000  |  Cenário: B             │
│            │                                                         │
│            │  [ AVANÇAR PARA ANUÊNCIA ]   [ ... ▼ Mais ações ]       │  [A1]
│            │  ─────────────────────────────────────────────────────  │
│            │  [ Resumo ] [ Linha do Tempo ] [ Dossiê ] [Negociação]  │  [A2]
│            │  [ Comunicação ] [ Atividades ]                         │
│            │  ─────────────────────────────────────────────────────  │
│            │                                                         │
│            │  ABA: RESUMO (padrão)                                   │
│            │  ┌──────────────────────┐  ┌───────────────────────┐   │
│            │  │ CEDENTE              │  │ CASO                  │   │
│            │  │ Nome: Silva F.***    │  │ ID: RS-2026-0042      │   │
│            │  │ E-mail: si***@g.com  │  │ Criado: 10/03/2026    │   │
│            │  │ Fone: (85) *****-42  │  │ Origem: Plataforma    │   │
│            │  │ [Ver perfil completo]│  │ SLA corrente: 🟡 80%  │   │
│            │  └──────────────────────┘  └───────────────────────┘   │
│            │                                                         │
│            │  ┌─────────────────────────────────────────────────┐   │
│            │  │ DOSSIÊ                           [ Ver Dossiê ] │   │
│            │  │ ● 8/11 documentos presentes · 6/11 aprovados    │   │
│            │  │ ⚠ Tabela Atual vence em 5 dias                  │   │
│            │  └─────────────────────────────────────────────────┘   │
│            │                                                         │
│            │  ┌─────────────────────────────────────────────────┐   │
│            │  │ ATIVIDADE RECENTE               [ + Registrar ] │   │
│            │  │ Ontem 14:30 — WhatsApp — "Proposta enviada ao   │   │
│            │  │ Cessionário para análise..."                     │   │
│            │  └─────────────────────────────────────────────────┘   │
└────────────┴─────────────────────────────────────────────────────────┘
```

**Estado: "Mais ações" expandido (Coordenador RS / Admin RS)**
```
│  [ AVANÇAR PARA ANUÊNCIA ]   [ ... ▼ Mais ações ]       │
│                              ┌─────────────────────┐    │
│                              │ Retroceder estado   │    │
│                              │ Redistribuir Caso   │    │
│                              │ Duplicar Caso       │    │
│                              │ Cancelar Caso       │    │
│                              └─────────────────────┘    │
```

**Estado: pré-requisitos não atendidos para avanço**
```
│  [ AVANÇAR PARA ANUÊNCIA — desabilitado ]                │
│  ⚠ Para avançar, resolva as pendências:                  │
│     • Dossiê não aprovado pelo Coordenador               │
│     • Tabela Atual com mais de 30 dias                   │
```

**Anotações:**
- [A1] Botão "Avançar" primário: desabilitado se pré-requisitos não atendidos. Exibe lista de pendências abaixo.
- [A2] Abas: linha de abas com indicadores de badge quando há pendências (ex: "Dossiê 🔴" indica documentos rejeitados).
- [A3] Dados pessoais mascarados: seguem RF-CRM-012. CPF, e-mail e telefone mascarados por padrão.
- [A4] Ação "Retroceder estado" disponível apenas para Coordenador RS e Admin RS.

---

### 2.4 WF-T031 — Negociação — Painel do Caso

```
┌────────────┬─────────────────────────────────────────────────────────┐
│  SIDEBAR   │  Negociação › RS-2026-0042                              │
│            │  ─────────────────────────────────────────────────────  │
│            │  MATCH  |  Cessionário: Car***  |  Data: 15/03/2026    │
│            │                                                         │
│            │  HISTÓRICO DE PROPOSTAS                                 │
│            │  ─────────────────────────────────────────────────────  │
│            │  🔵 Rodada 1 — 15/03/2026 14:00                        │
│            │     Proposta Cessionário: R$ 350.000                    │
│            │     Cond.: À vista · Validade: 48h · Status: Expirada   │
│            │                                                         │
│            │  🟠 Rodada 2 — 16/03/2026 10:00                        │
│            │     Contraproposta Cedente: R$ 370.000                  │
│            │     Status: Aguardando resposta do Cessionário          │
│            │     ⏱ Expira em: 18/03/2026 10:00 (32h restantes)      │  [A1]
│            │                                                         │
│            │  ─────────────────────────────────────────────────────  │
│            │  COMISSÃO ESTIMADA (após aceite)                        │
│            │  [ESTIMATIVA — sujeita a Fechamento]                    │  [A2]
│            │  Δ: R$ 45.000  |  Comissão Cess.: R$ 9.000             │
│            │  Comissão Ced.: R$ 3.200 (Cenário B)                   │
│            │                                                         │
│            │  ─────────────────────────────────────────────────────  │
│            │  [ + Registrar Contraproposta ]  [ Aceitar Proposta ]  │
│            │  [ Encerrar Negociação ]                                │  [A3]
│            │                                                         │
└────────────┴─────────────────────────────────────────────────────────┘
```

**Estado: alerta de múltiplas rodadas (≥6)**
```
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 🟡 Esta negociação está em 6 rodadas. Considere mediar uma   │   │
│  │    conversa direta para destravar o impasse.                 │   │
│  └──────────────────────────────────────────────────────────────┘   │
```

**Anotações:**
- [A1] Contador de prazo em tempo real. Quando <6h: cor vermelho.
- [A2] Badge "[ESTIMATIVA]" sempre visível enquanto o Caso não está em "Concluído".
- [A3] "Aceitar Proposta" disponível apenas quando há proposta ativa com status "Aguardando". "Encerrar" disponível sempre.

---

### 2.5 WF-T040 — Dossiê — Checklist de Documentos

```
┌────────────┬─────────────────────────────────────────────────────────┐
│  SIDEBAR   │  Dossiê › RS-2026-0042                                  │
│            │  ─────────────────────────────────────────────────────  │
│            │  Status: Em montagem  │  6/11 aprovados                 │
│            │                                                         │
│            │  [ Solicitar Aprovação — desabilitado ]                 │  [A1]
│            │  ⚠ 2 documentos obrigatórios pendentes                  │
│            │  ─────────────────────────────────────────────────────  │
│            │                                                         │
│            │  DOCUMENTOS OBRIGATÓRIOS — TODOS OS ESTADOS             │
│            │  ┌────────────────────────────────────────────────────┐ │
│            │  │ ✅  Tabela Atual (fonte oficial)     [Histórico]  │ │  [A2]
│            │  │ ✅  Tabela do Contrato                            │ │
│            │  │ ✅  Extrato do saldo devedor                      │ │
│            │  │ ✅  Comprovante de pagamento das parcelas         │ │
│            │  │ ✅  Documento de identidade do Cedente            │ │
│            │  │ ✅  Selfie com verificação de vivacidade          │ │
│            │  │ ⏳  Comprovante de endereço (≤90 dias)  [Upload] │ │  [A3]
│            │  │ ❌  Valor de Distrato Referência        [Upload] │ │  [A4]
│            │  │     ↳ Rejeitado: "Documento ilegível"            │ │
│            │  └────────────────────────────────────────────────────┘ │
│            │                                                         │
│            │  DOCUMENTOS — ESTADO FORMALIZAÇÃO                       │
│            │  ┌────────────────────────────────────────────────────┐ │
│            │  │ ⏳  Instrumento de cessão assinado      [Upload]  │ │
│            │  │ ⏳  Anuência da incorporadora            [Upload]  │ │
│            │  │ ⏳  Comprovante de depósito Escrow       [Upload]  │ │
│            │  └────────────────────────────────────────────────────┘ │
│            │                                                         │
│            │  ⚠ Tabela Atual emitida em 05/02/2026 — 46 dias        │  [A5]
│            │    (>30 dias). Reenvie antes de publicar o Caso.        │
└────────────┴─────────────────────────────────────────────────────────┘
```

**Estado: Dossiê aprovado**
```
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ✅ Dossiê aprovado pelo Coord. Ana Lima em 20/03/2026 10:30  │   │
│  └──────────────────────────────────────────────────────────────┘   │
```

**Anotações:**
- [A1] Botão "Solicitar Aprovação": habilitado somente quando todos os itens obrigatórios estão em ✅ ou 🔵.
- [A2] Botão [Histórico]: abre drawer lateral com versões anteriores do documento.
- [A3] ⏳ Pendente: aguardando envio — botão [Upload] visível.
- [A4] ❌ Rejeitado: exibe motivo da rejeição + botão [Reenviar].
- [A5] Alerta de validade da Tabela Atual: banner âmbar quando data de emissão >30 dias.

---

### 2.6 WF-T051 — Contatos — Perfil do Contato

```
┌────────────┬─────────────────────────────────────────────────────────┐
│  SIDEBAR   │  Contatos › Carlos Alves Mendes                        │
│            │  ─────────────────────────────────────────────────────  │
│            │  ████  Carlos A. M.                    [ Editar ]      │
│            │        Cessionário · Investidor Recorrente 🏆           │  [A1]
│            │                                                         │
│            │  ─────────────────────────────────────────────────────  │
│            │  INFORMAÇÕES DE CONTATO                                 │
│            │  E-mail:   ca***@gmail.com                              │  [A2]
│            │  Telefone: (85) *****-1234                              │
│            │  CPF:      ***.456.***-**                               │
│            │                                                         │
│            │  PREFERÊNCIAS (Cessionário)                             │
│            │  Região: Fortaleza — Aldeota, Meireles                  │
│            │  Faixa valor: R$ 200k – R$ 500k                        │
│            │  Tipologia: Residencial                                  │
│            │                                                         │
│            │  ─────────────────────────────────────────────────────  │
│            │  CASOS VINCULADOS (3)                                   │
│            │  RS-2026-0019 · Concluído · 12/02/2026                  │
│            │  RS-2026-0031 · Negociação · Em andamento               │
│            │  RS-2025-0087 · Concluído · 08/11/2025                  │
│            │                                                         │
│            │  ─────────────────────────────────────────────────────  │
│            │  ATIVIDADES RECENTES                                    │
│            │  20/03 · WhatsApp · "Interesse confirmado em RS-0031..."│
│            │  18/03 · Ligação · "Follow-up sobre proposta..."        │
│            │  [ Ver todas as atividades ]                            │
│            │                                                         │
│            │  ─────────────────────────────────────────────────────  │
│            │  [ + Registrar Atividade ]   [ Enviar WhatsApp ]        │
│            │  [ Enviar E-mail ]                                       │
└────────────┴─────────────────────────────────────────────────────────┘
```

**Estado: Contato com opt-out ativo**
```
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 🔴 OPT-OUT ATIVO — Comunicações bloqueadas (LGPD)            │   │
│  │    Registrado em 15/03/2026 por João Silva                   │   │
│  │    [ Reverter opt-out — requer confirmação ] →               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  [ + Registrar Nota Interna ]                                        │  [A3]
│  (botões "Enviar WhatsApp" e "Enviar E-mail" desabilitados)          │
```

**Estado: Contato arquivado**
```
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ℹ ARQUIVADO em 01/01/2026 — Inativo há 12 meses             │   │
│  │  [ Reativar vinculando a novo Caso ]                         │   │
│  └──────────────────────────────────────────────────────────────┘   │
```

**Anotações:**
- [A1] Badge "Investidor Recorrente" 🏆: visível apenas em Cessionários com flag ativa.
- [A2] Dados mascarados por padrão. Admin RS e Coordenador RS podem ver dados completos clicando em "Mostrar dados completos" (com registro no log de auditoria).
- [A3] Com opt-out: botões de comunicação desabilitados. Apenas "Registrar Nota Interna" disponível.

---

### 2.7 WF-T080 — SLA Monitor — Dashboard de SLA

```
┌────────────┬─────────────────────────────────────────────────────────┐
│  SIDEBAR   │  SLA Monitor                                            │
│            │  ─────────────────────────────────────────────────────  │
│            │  Atualizado: há 2 min  [ 🔄 Atualizar ]                │
│            │                                                         │
│            │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│            │  │ DENTRO   │ │ EM RISCO │ │  SLA     │ │ CICLO    │  │
│            │  │  DO      │ │  (>80%)  │ │ VENCIDO  │ │ CRÍTICO  │  │
│            │  │ PRAZO    │ │          │ │ (100%)   │ │ (>60 d)  │  │
│            │  │   12     │ │    5     │ │    3     │ │    1     │  │
│            │  │  🟢      │ │  🟡      │ │  🔴      │ │  ⚠      │  │
│            │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│            │  ─────────────────────────────────────────────────────  │
│            │                                                         │
│            │  ALERTAS ATIVOS (8)         [▼ Ordenar por: Prioridade] │
│            │  ┌────────────────────────────────────────────────────┐ │
│            │  │ 🔴 RS-2026-0019 · João · NEGOCIAÇÃO · 108% SLA    │ │  [A1]
│            │  │    ⏱ Venceu há 2 dias · Não reconhecido           │ │
│            │  │    [ Reconhecer ]  [ Ver Caso ]                    │ │
│            │  ├────────────────────────────────────────────────────┤ │
│            │  │ 🔴 RS-2026-0031 · Ana · VERIFICAÇÃO · 100% SLA   │ │
│            │  │    ⏱ Venceu hoje · Reconhecido por Coord. 09:15   │ │
│            │  │    [ Ver Caso ]                                    │ │
│            │  ├────────────────────────────────────────────────────┤ │
│            │  │ 🟡 RS-2026-0042 · João · NEGOCIAÇÃO · 82% SLA    │ │
│            │  │    ⏱ Falta 1,5 dias para vencer                   │ │
│            │  │    [ Reconhecer ]  [ Ver Caso ]                    │ │
│            │  └────────────────────────────────────────────────────┘ │
│            │                                                         │
│            │  RANKING ANALISTAS — MÊS CORRENTE                      │
│            │  ┌────────────────────────────────────────────────────┐ │
│            │  │ 1. Ana Lima        ████████████ 94% cumprimento    │ │
│            │  │ 2. João Silva      █████████░░░ 82% cumprimento    │ │
│            │  │ 3. Pedro Mota      ███████░░░░░ 71% cumprimento    │ │
│            │  └────────────────────────────────────────────────────┘ │
└────────────┴─────────────────────────────────────────────────────────┘
```

**Anotações:**
- [A1] Alerta não reconhecido após 24h: badge "⚠ Escalado para Admin RS" na linha.
- [A2] Alerta com 150% de SLA: badge especial "Caso em risco de cancelamento" em vermelho intenso.
- [A3] Botão "Reconhecer": registra usuário + timestamp. Badge muda para "Reconhecido por [Nome] às [hora]".
- [A4] Ranking: dados do mês corrente por padrão, toggle para "Últimos 3 meses".

---

### 2.8 WF-T070 — Comunicação — Caixa de Mensagens do Caso

```
┌────────────┬─────────────────────────────────────────────────────────┐
│  SIDEBAR   │  Comunicação › RS-2026-0042 › Cedente: Silva F.        │
│            │  ─────────────────────────────────────────────────────  │
│            │  WhatsApp: (85) 9****-4242  · 🟢 Janela aberta (18h)  │  [A1]
│            │                                                         │
│            │  ─────────────────────────────────────────────────────  │
│            │                                                         │
│            │      20/03/2026                                         │
│            │  ─── ─────────────────────────────────────── ───       │
│            │                                                         │
│            │      ┌────────────────────────────────────────┐        │
│            │      │ Olá, Silva! Seu Caso RS-0042 teve uma  │        │
│            │      │ proposta recebida. Vou entrar em        │        │
│            │      │ contato em breve com os detalhes.       │        │
│            │      │                              ✅ Lida    │        │  [A2]
│            │      └────────────────────────────────────────┘        │
│            │                         Enviado via template · 14:30   │
│            │                                                         │
│            │  ┌──────────────────────────────────────────┐          │
│            │  │ Oi! Pode me passar mais informações?     │          │
│            │  │ Obrigado.                    14:45       │          │
│            │  └──────────────────────────────────────────┘          │
│            │  Cedente                                                │
│            │                                                         │
│            │  ─────────────────────────────────────────────────────  │
│            │  ┌────────────────────────────────────────────────────┐ │
│            │  │ [ Escreva uma mensagem...                        ] │ │  [A3]
│            │  └────────────────────────────────────────────────────┘ │
│            │  [ 📎 Anexar ]  [ 📋 Templates ]  [ ENVIAR ]           │
│            │                                                         │
└────────────┴─────────────────────────────────────────────────────────┘
```

**Estado: Janela de 24h encerrada**
```
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ⚠ Janela de atendimento encerrada (última msg. >24h).        │   │
│  │   Apenas templates aprovados podem ser enviados.             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  [ área de input desabilitada ]                                      │
│  [ 📋 Usar Template Aprovado ]                                       │
```

**Estado: Falha de entrega**
```
│      ┌──────────────────────────────────────────────┐             │
│      │ Mensagem não entregue ao destinatário.       │             │
│      │ ⚠ Tente outro canal: ligue ou envie e-mail.  │             │
│      │                                   ❌ Falhou  │             │
│      └──────────────────────────────────────────────┘             │
```

**Estado: Opt-out ativo**
```
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 🔴 OPT-OUT ATIVO — Comunicação bloqueada por solicitação     │   │
│  │    do Contato (LGPD). Edite o perfil do Contato para         │   │
│  │    reverter mediante nova solicitação expressa.              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  [ área de input desabilitada ]                                      │
```

**Anotações:**
- [A1] Indicador de janela: 🟢 Janela aberta (Xh restantes) · 🔴 Janela encerrada.
- [A2] Status de entrega: ✅ Lida · ✔✔ Entregue · ✔ Enviada · ❌ Falhou.
- [A3] Campo de input: autofocus. Máximo 4096 caracteres. Contador visível ao digitar.
- [A4] Botão "Templates": abre modal T-CRM-071 com lista de templates disponíveis para o momento do Caso.
- [A5] Mensagem registrada manualmente exibe badge "📝 Registrado manualmente" na thread.

---

## 3. Componentes Compartilhados

### 3.1 Modal de Confirmação de Ação Destrutiva

```
┌──────────────────────────────────────────────────────────┐
│  ⚠ Confirmar: [Título da ação]                           │
│  ─────────────────────────────────────────────────────── │
│  [Descrição da consequência da ação. Não pode ser        │
│   desfeita.]                                             │
│                                                          │
│  [ campo de justificativa — mín. 20 caracteres ]         │  [A1]
│                                                          │
│  [ Cancelar ]              [ CONFIRMAR — ação ]          │
└──────────────────────────────────────────────────────────┘
```

[A1] Campo de justificativa: obrigatório para: retroceder estado, cancelar Caso, desligar usuário, excluir Atividade (Admin RS). Não obrigatório para ações não-destrutivas.

---

### 3.2 Drawer de Linha do Tempo do Caso

```
┌───────────────────────────────────────────────┐
│  Linha do Tempo · RS-2026-0042      [✕ Fechar]│
│  ─────────────────────────────────────────── │
│                                               │
│  Hoje 14:30                                   │
│  💬 WhatsApp (João Silva)                     │
│     "Proposta enviada ao Cessionário..."      │
│                                               │
│  ─────────────────────────────────────────── │
│  Hoje 10:15                                   │
│  📊 Estado → NEGOCIAÇÃO (João Silva)          │
│                                               │
│  ─────────────────────────────────────────── │
│  Ontem 16:00                                  │
│  📄 Documento enviado — Tabela Atual          │
│     (Cedente via plataforma)                  │
│                                               │
│  ─────────────────────────────────────────── │
│  15/03 09:00                                  │
│  📋 Match registrado — Cessionário: Car***    │
│                                               │
│  ─────────────────────────────────────────── │
│  10/03 08:30                                  │
│  ⚙ Caso criado (Origem: Plataforma)           │
│                                               │
│  [ Carregar mais ]                            │
└───────────────────────────────────────────────┘
```

[A1] Drawer somente leitura para todos os perfis. Parceiro Externo não acessa este drawer — vê apenas mudanças de estado no portal externo.

---

### 3.3 Componente de Notificação (Sino / Badge)

```
Topbar → 🔔 3
         ┌─────────────────────────────────────────────┐
         │ NOTIFICAÇÕES (3 não lidas)          [Todas] │
         │ ─────────────────────────────────────────── │
         │ 🔴 [CRÍTICA] SLA vencido — RS-0019          │
         │    Há 2h · João Silva · Negociação          │
         │ ─────────────────────────────────────────── │
         │ 🟡 [ALTA] Follow-up vencido — RS-0031       │
         │    Há 45min · Ana Lima                      │
         │ ─────────────────────────────────────────── │
         │ 🔵 Novo interesse — RS-0042 (plataforma)    │
         │    Há 3min                                  │
         │ ─────────────────────────────────────────── │
         │              [ Marcar todas como lidas ]    │
         └─────────────────────────────────────────────┘
```

[A1] Notificações Críticas e de Alta prioridade não podem ser desativadas pelo usuário.
[A2] Click em qualquer notificação navega para o objeto referenciado (Caso, Atividade, etc.).

---

### 3.4 Busca Global (Ctrl+K / Cmd+K)

```
┌──────────────────────────────────────────────────────────┐
│  [ 🔍 Buscar Caso, Contato, Atividade...         ] [✕]   │
│  ─────────────────────────────────────────────────────── │
│  RESULTADOS — "RS-2026"                                   │
│                                                           │
│  CASOS (4)                                                │
│  📋 RS-2026-0042 · Lumina Park · Negociação              │
│  📋 RS-2026-0031 · Park RS · Verificação                 │
│  📋 RS-2026-0019 · Lumina · Concluído                    │
│  [ + 1 resultado ]                                        │
│  ─────────────────────────────────────────────────────── │
│  CONTATOS (0)                                             │
│  ─────────────────────────────────────────────────────── │
│  ATIVIDADES (2)                                           │
│  📝 "...Caso RS-2026 necessita de seguimento..." (RS-0042)│
│  📝 "...proposta RS-2026 enviada ao cedente..." (RS-0031) │
└──────────────────────────────────────────────────────────┘
```

[A1] Resultados filtrados por permissão: Analista RS vê apenas Casos e Contatos próprios.
[A2] Dados pessoais mascarados nos resultados.
[A3] Não retorna documentos PDF, logs de auditoria nem versões de templates.

---

## 4. Estados de Loading e Erro

### 4.1 Skeleton Loading (carregamento inicial)

```
┌──────────────────────────────────────────────────────────┐
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░                  │
│  ░░░░░░░░░░░░░░░░░░░░░░░                                  │
│                                                          │
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░                  │
└──────────────────────────────────────────────────────────┘
```

[A1] Skeleton exibido por máximo 3s. Se dados não carregaram em 3s: exibir estado de erro.

### 4.2 Estado de Erro de Carregamento

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│              ⚠ Não foi possível carregar                 │
│                os dados desta tela.                      │
│                                                          │
│              [ Tentar novamente ]                        │
│                                                          │
│         Se o problema persistir, contate o suporte.      │
└──────────────────────────────────────────────────────────┘
```

### 4.3 Estado Vazio (sem dados)

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│              [ícone de contexto]                         │
│                                                          │
│              [Mensagem contextual de estado vazio]       │
│              [Instrução de ação, se aplicável]           │
│                                                          │
│              [ Ação primária ]                           │
└──────────────────────────────────────────────────────────┘
```

Exemplos por tela:
- Pipeline vazio: "Nenhum Caso ativo." + [ + Criar Caso ]
- Contatos vazio: "Nenhum contato cadastrado." + [ + Novo Contato ]
- Atividades vazio: "Nenhum follow-up agendado para hoje."
- SLA Monitor sem alertas: "Todos os SLAs estão dentro do prazo. Bom trabalho!"

---

## 5. Resumo das Telas Cobertos pelos Wireframes

| WF | Tela | Prioridade |
|---|---|---|
| WF-T001 | Login / Autenticação | P0 — crítico para acesso |
| WF-T020 | Pipeline — Visão Kanban | P0 — tela principal de trabalho |
| WF-T022 | Detalhes do Caso | P0 — hub central de operação |
| WF-T031 | Negociação — Painel do Caso | P1 — módulo de receita |
| WF-T040 | Dossiê — Checklist | P1 — gate obrigatório do ciclo |
| WF-T051 | Contatos — Perfil | P1 — entidade central do CRM |
| WF-T080 | SLA Monitor — Dashboard | P1 — visibilidade operacional |
| WF-T070 | Comunicação — WhatsApp | P1 — canal principal |

> **Telas não wireframadas neste documento** (P2/P3 — baixa complexidade estrutural ou uso esporádico): T-CRM-002/003 (recuperação de senha), T-CRM-021 (Pipeline lista — variante tabular de T-020), T-CRM-032/033/034 (modais de proposta — estrutura simples de formulário), T-CRM-042 (modal de aprovação de dossiê), T-CRM-050 (lista de Contatos — tabela padrão), T-CRM-052 (formulário de Contato), T-CRM-053 (modal de mesclagem), T-CRM-060/061 (agenda e modal de atividade), T-CRM-071 (modal de template), T-CRM-090–093 (Equipe), T-CRM-100–104 (Relatórios), T-CRM-110–113 (Configurações).
