# 09 - Contratos de UI por Tela

## Módulo Cedente · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Documento** | 09 - Contratos de UI por Tela |
| **Produto** | Repasse Seguro — Módulo Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 (America/Fortaleza) |
| **Autor** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Status** | Ativo |
| **Fase** | 2 — UI/UX |
| **Área** | Design + Frontend |
| **Referências** | 01.1/01.2 - Regras de Negócio · 03 - Brand Theme Guide · 06 - Mapa de Telas · 07 - Wireframes · 08 - UX Writing |

---

> 📌 **TL;DR**
>
> - **52 telas cobertas** (T-001 a T-052) com contratos de UI completos.
> - **Padrão de 4 estados obrigatórios** por tela: `Skeleton → Empty → Error → Populated`. Spinners são proibidos em todo o produto.
> - **Layouts base reutilizados:** `AuthLayout` (10 telas de autenticação), `AppLayout` (40 telas logadas), `ErrorLayout` (2 telas de erro).
> - **Componentes críticos de domínio:** `CaseStatusBadge`, `DocumentCard`, `ProposalCard`, `EscrowPanel`, `GuardiaoChat`, `CaseCard`, `ScenarioCard`, `FinancialSummary`, `SlaCountdown`, `UploadZone`, `Stepper`, `SignatureInline`.
> - **RBAC obrigatório** via `useRbac()` + `RbacGuard` — role `CEDENTE` em todas as telas logadas; telas de autenticação são públicas.
> - **Tokens canônicos:** todos os tokens referenciam o namespace `--` do shadcn/ui conforme definido no doc 03 (Brand Theme Guide). Não há tokens customizados fora dessa estrutura.
> - **Responsividade:** mobile-first. Breakpoints: `< 768px` (mobile), `768px–1024px` (tablet), `> 1024px` (desktop). Sidebar colapsa em drawer em tablet/mobile.

---

## 1. Convenções Globais

### 1.1 Quatro Estados Obrigatórios por Tela

Toda tela DEVE implementar os 4 estados abaixo. Não há exceção.

| **Estado** | **Componente** | **Trigger** | **Duração máxima** |
|---|---|---|---|
| **Skeleton** | `<Skeleton>` shadcn/ui — **nunca spinner** | Enquanto fetch em andamento | Até dados chegarem |
| **Empty** | `<EmptyState>` custom (ilustração SVG + headline + CTA opcional) | `data.length === 0` ou sem dados | Indeterminado |
| **Error** | `<Alert variant="destructive">` + botão "Tentar novamente" | Request HTTP falha ou timeout | Indeterminado |
| **Populated** | Conteúdo real renderizado | Após dados carregados com sucesso | Indeterminado |

🔴 **Spinners são absolutamente proibidos.** Qualquer loading state usa `<Skeleton>` do shadcn/ui. A única exceção é o botão de submit durante POST/PUT: botão desabilitado + label textual de contexto (ex: "Enviando...").

### 1.2 Três Layouts Base

| **Layout** | **Telas** | **Estrutura** |
|---|---|---|
| `AuthLayout` | T-001 a T-010 | Centralizado, card 400px, fundo `--background`, sem sidebar |
| `AppLayout` | T-011 a T-052 (área logada) | Sidebar 240px fixa (`--sidebar`) + área de conteúdo fluida. Em tablet: sidebar como drawer. Em mobile: sem sidebar (bottom tab bar T-012) |
| `ErrorLayout` | Páginas de erro 404/500 | Centralizado, sem sidebar, ilustração + CTA |

### 1.3 Componentes Críticos de Domínio

| **Componente** | **Telas principais** | **Tokens principais** | **Faixas semânticas** |
|---|---|---|---|
| `CaseStatusBadge` | T-015, T-016, T-013 | Mapeamento completo em §2.1 | Por enum `CaseStatus` |
| `DocumentCard` | T-040, T-017 | `--card`, `--muted-foreground`, `--destructive`, `--chart-2` | Por status do documento |
| `ProposalCard` | T-035, T-018 | `--primary`, `--chart-2`, `--destructive` | Valor líquido em destaque |
| `EscrowPanel` | T-045, T-046, T-020 | `--chart-2` (verde), `--chart-3` (âmbar), `--destructive` | Por estado da conta Escrow |
| `GuardiaoChat` | T-048, T-049 | `--primary`, `--muted`, `--card` | N/A — interface conversacional |
| `CaseCard` | T-015 | `--card`, `--primary`, `--muted-foreground` | N/A — card de resumo de caso |
| `ScenarioCard` | T-026, T-027, T-031 | `--card`, `--primary`, `--chart-2` | A/B/C/D sem pré-seleção |
| `FinancialSummary` | T-016, T-013 | `--chart-2`, `--primary`, `--muted` | Valor líquido em destaque |
| `SlaCountdown` | T-035, T-043, T-046 | `--destructive` (≤24h), `--chart-3` (≤72h), `--foreground` (>72h) | Verde → Âmbar → Vermelho |
| `Stepper` | T-024 a T-028 | `--primary` (ativo), `--chart-2` (concluído), `--muted` (pendente) | Horizontal no wizard |
| `UploadZone` | T-040, T-041 | `--primary`, `--muted`, `--destructive` | Estados de upload |
| `SignatureInline` | T-044 | `--card`, `--primary` | Iframe ZapSign embedado |

### 1.4 RBAC — Guards e Roles

```typescript
// Telas públicas — sem guard
// T-001 a T-010

// Telas logadas — role CEDENTE obrigatório
// T-011 a T-052
<RbacGuard role="CEDENTE">
  <ComponenteDaTela />
</RbacGuard>

// Ações condicionais por tipo de pessoa
const { tiposPessoa } = useRbac()
// tiposPessoa: 'PF' | 'PJ'

// Ações bloqueadas para Cedente PJ sem representante legal autenticado
// RN-002: cancelamento, aceite de proposta, assinaturas
```

### 1.5 Motion — Padrões Globais

| **Animação** | **Duração** | **Easing** | **Uso** |
|---|---|---|---|
| Fade in de dados | `--duration-normal` (250ms) | `--ease-in-out` | Conteúdo após skeleton |
| Scale spring (cards, badges) | 300ms | `--ease-spring` | Cards de cenário, badges de status |
| Toast entrada/saída | 200ms | `--ease-in-out` | Toasts de sucesso/erro |
| Modal open/close | 150ms | `--ease-in-out` | Todos os modais |
| Sidebar collapse | `--duration-normal` (250ms) | `--ease-in-out` | Collapse da sidebar em tablet |
| Checkmark animado | 300ms | `--ease-spring` bouncy | T-004, T-029 |
| Stagger de cards | 100ms entre cards | `--ease-in-out` | ScenarioCards em T-026 |
| Streaming de texto | N/A | N/A | Guardião — cursor piscante nativo |

---

## 2. Contratos de Componentes de Domínio

### 2.1 CaseStatusBadge

```typescript
type CaseStatus =
  | 'CAPTADO'
  | 'EM_TRIAGEM'
  | 'BLOQUEADO'
  | 'QUALIFICADO'
  | 'OFERTA_ATIVA'
  | 'EM_NEGOCIACAO'
  | 'EM_FORMALIZACAO'
  | 'FECHAMENTO'
  | 'POS_FECHAMENTO'
  | 'EM_REVERSAO'
  | 'EM_MEDIACAO'
  | 'CONCLUIDO'
  | 'CANCELADO'

interface CaseStatusBadgeProps {
  status: CaseStatus
  size?: 'sm' | 'md'          // md default
  pulse?: boolean              // true para EM_NEGOCIACAO
}
```

| **Status** | **Label** | **Token de cor** | **Pulse** |
|---|---|---|---|
| `CAPTADO` | "Cadastro realizado" | `--muted-foreground` | Não |
| `EM_TRIAGEM` | "Em análise" | `--chart-3` | Não |
| `BLOQUEADO` | "Pendência identificada" | `--destructive` | Não |
| `QUALIFICADO` | "Aprovado para oferta" | `--chart-2` | Não |
| `OFERTA_ATIVA` | "Disponível para compradores" | `--primary` | Não |
| `EM_NEGOCIACAO` | "Proposta recebida" | `--primary` | Sim |
| `EM_FORMALIZACAO` | "Em formalização" | `--chart-3` | Não |
| `FECHAMENTO` | "Negócio fechado" | `--chart-2` | Não |
| `POS_FECHAMENTO` | "Aguardando liberação" | `--chart-2` | Não |
| `EM_REVERSAO` | "Desistência em análise" | `--chart-3` | Não |
| `EM_MEDIACAO` | "Mediação em andamento" | `--chart-3` | Não |
| `CONCLUIDO` | "Concluído" | `--chart-2` | Não |
| `CANCELADO` | "Cancelado" | `--muted-foreground` | Não |

### 2.2 DocumentCard

```typescript
type DocumentStatus = 'PENDENTE' | 'EM_ANALISE' | 'VERIFICADO' | 'REJEITADO'

interface DocumentCardProps {
  id: string
  nome: string
  status: DocumentStatus
  motivoRejeicao?: string      // obrigatório quando status === 'REJEITADO'
  arquivoUrl?: string          // quando status !== 'PENDENTE'
  onUpload: () => void         // callback para abrir UploadZone
  onReenviar?: () => void      // quando status === 'REJEITADO'
  loading?: boolean
}
```

**Estados visuais:**
- `PENDENTE`: borda `--muted`, botão "Enviar documento" primário
- `EM_ANALISE`: borda `--chart-3`, badge âmbar, sem ação (somente leitura)
- `VERIFICADO`: borda `--chart-2`, badge verde com ícone check, sem botão
- `REJEITADO`: borda `--destructive`, badge vermelho, texto âmbar com `motivoRejeicao`, botão "Reenviar"

**Acessibilidade:** `aria-label="[nome do documento] — status: [label do status]"`

### 2.3 ProposalCard

```typescript
type ProposalStatus =
  | 'AGUARDANDO_RESPOSTA'
  | 'CONTRAPROPOSTA_ENVIADA'
  | 'ACEITA'
  | 'RECUSADA'
  | 'EXPIRADA'
  | 'SUPERADA'

interface ProposalCardProps {
  id: string
  valorProposto: number
  valorLiquido: number          // destaque máximo — "Você recebe"
  comissao: number
  prazoExpiracao: Date
  rodada: number
  status: ProposalStatus
  urgente: boolean              // true quando < 24h para expirar
  onAceitar: () => void
  onRecusar: () => void
  onContrapropor: () => void
}
```

**Hierarquia visual obrigatória:**
1. "Você recebe: R$ [valorLiquido]" — `--chart-2`, fonte H2, posição superior
2. "Valor proposto: R$ [valorProposto]" — `--foreground`, fonte body
3. "Comissão: R$ [comissao]" — `--muted-foreground`, colapsável

**Urgente:** borda `--destructive` + badge "Urgente" + `SlaCountdown` em vermelho pulsante

### 2.4 EscrowPanel

```typescript
type EscrowStatus =
  | 'NAO_INICIADO'
  | 'AGUARDANDO_DEPOSITO'
  | 'DEPOSITO_CONFIRMADO'
  | 'POS_FECHAMENTO'
  | 'DISTRIBUIDA'
  | 'EM_REVERSAO'
  | 'ESTORNADA'

interface EscrowPanelProps {
  status: EscrowStatus
  valorLiquido?: number
  valorEscrow?: number
  comissao?: number
  dataLiberacao?: Date           // para POS_FECHAMENTO
  diasRestantes?: number         // para POS_FECHAMENTO
  onSolicitarDesistencia?: () => void  // visível apenas em POS_FECHAMENTO dentro de 15 dias
  loading?: boolean
}
```

**Estados por status:**
- `NAO_INICIADO`: informativo somente leitura, valores estimados em `--muted`
- `AGUARDANDO_DEPOSITO`: badge âmbar, aguardando ação do Cessionário
- `DEPOSITO_CONFIRMADO`: badge verde, Fechamento em andamento
- `POS_FECHAMENTO`: "Você receberá R$ [X]" em `--chart-2` bold H1, `SlaCountdown` dos 15 dias, botão "Solicitar Desistência" (text-only, `--destructive`)
- `DISTRIBUIDA`: badge verde, "R$ [X] liberado para sua conta"
- `EM_REVERSAO`: badge âmbar, mensagem de estorno em andamento
- `ESTORNADA`: badge neutro, confirmação de estorno

### 2.5 ScenarioCard

```typescript
type Cenario = 'A' | 'B' | 'C' | 'D'

interface ScenarioCardProps {
  cenario: Cenario
  valorRetornoEstimado: number
  comissao: number
  valorLiquido: number           // destaque máximo
  dificuldadeLabel: string
  selected: boolean
  disabled?: boolean
  onSelect: (cenario: Cenario) => void
}
```

**Props de acessibilidade:** `role="radio"`, `aria-checked={selected}`, parte de `role="radiogroup"`

**Sem pré-seleção:** `selected` inicia `false` para todos os cards (RN-022).

**Estilo quando selecionado:** borda `--primary` 2px + fundo `--primary/5` + scale spring 300ms.

### 2.6 SlaCountdown

```typescript
interface SlaCountdownProps {
  deadline: Date
  onExpired?: () => void
}
```

**Faixas de cor:**
- `> 72h`: `--foreground` (neutro)
- `≤ 72h e > 24h`: `--chart-3` (âmbar)
- `≤ 24h`: `--destructive` (vermelho) + pulse animation

**Formato de exibição:**
- `> 7 dias`: "X dias"
- `> 24h e ≤ 7 dias`: "X dias X horas"
- `> 1h e ≤ 24h`: "X horas X minutos"
- `≤ 1h`: "X minutos X segundos"

**Acessibilidade:** `aria-live="polite"` — anuncia quando muda de faixa de cor (não a cada segundo).

---

## 3. Contratos por Tela — Autenticação (T-001 a T-010)

### T-001 — Login

| **Atributo** | **Valor** |
|---|---|
| Layout | `AuthLayout` |
| Role | Público |
| Estados obrigatórios | Populated (sempre) |

**Props / Estado:**
```typescript
interface LoginFormState {
  email: string
  senha: string
  loading: boolean
  erroCredenciais: boolean
  erroEmailNaoConfirmado: boolean
  tentativas: number             // exibe contador a partir de 3
  contaBloqueada: boolean
}
```

**Validações:**
- E-mail: formato válido (regex `^[^\s@]+@[^\s@]+\.[^\s@]+$`), validado on blur
- Senha: não vazia, validada on submit (nunca on blur — evita mensagens prematuras)

**Eventos:**
- `onSubmit`: POST `/auth/login` → sucesso: redirect T-013 | 401: `erroCredenciais = true` | 403 email não confirmado: `erroEmailNaoConfirmado = true` | 429 conta bloqueada: redirect T-009
- `onReenviarConfirmacao`: link visível apenas quando `erroEmailNaoConfirmado = true`

**Acessibilidade:**
- `<form aria-label="Formulário de login">`
- Campo senha: `aria-describedby="toggle-senha-hint"`
- Erro inline: `role="alert"`, associado via `aria-describedby`

### T-002 — Cadastro

| **Atributo** | **Valor** |
|---|---|
| Layout | `AuthLayout` — card 480px (PJ tem mais campos) |
| Role | Público |
| Estados obrigatórios | Populated (sempre) |

**Props / Estado:**
```typescript
interface CadastroFormState {
  tipoPessoa: 'PF' | 'PJ'       // PF default
  nome: string
  documento: string              // CPF (PF) ou CNPJ (PJ)
  razaoSocial?: string           // apenas PJ
  email: string
  confirmarEmail: string
  telefone: string
  senha: string
  confirmarSenha: string
  loading: boolean
  erroEmailDuplicado: boolean
  erroDocumentoDuplicado: boolean
}
```

**Checklist de força da senha (em tempo real):**
```
☐ Mínimo 8 caracteres
☐ Uma letra maiúscula
☐ Um número
☐ Um caractere especial
```

**Responsividade:**
- Mobile: campos em coluna única, sem split view
- Tablet/Desktop: campos em coluna única (card centralizado)

**Acessibilidade:**
- `<fieldset>` com `<legend>` para agrupamentos PF/PJ
- Toggle de tipo: `role="radiogroup"`, `aria-label="Tipo de pessoa"`
- Checklist de senha: `aria-live="polite"` para anunciar critérios atendidos

### T-003 — E-mail de Ativação Pendente

**Componentes:** ícone envelope SVG `aria-hidden="true"`, `SlaCountdown` (48h), botão "Reenviar e-mail" com cooldown 60s.

**Estados do botão de reenvio:**

| **Estado** | **Label** | **Habilitado** |
|---|---|---|
| Default | "Reenviar e-mail" | Sim |
| Cooldown | "Reenviar em [X]s" | Não |
| Loading | "Enviando..." | Não |

### T-009 — Conta Bloqueada Temporariamente

**Componente especial:** `SlaCountdown` configurado com `deadline = agora + 15 minutos`. Botão "Voltar para o login" habilitado apenas após `onExpired`.

---

## 4. Contratos por Tela — Área Logada Global (T-011, T-012)

### T-011 — Shell da Área Logada (Sidebar Web)

| **Atributo** | **Valor** |
|---|---|
| Layout | Componente de shell (não usa `AppLayout` — É o próprio layout) |
| Role | `CEDENTE` obrigatório |
| Persistente | Sim — renderizado em todas as rotas logadas |

**Props:**
```typescript
interface SidebarProps {
  activeRoute: string
  notificacoesNaoLidas: number
  nomeUsuario: string
  avatarUrl?: string
  collapsed: boolean             // false = 240px, true = 64px
  onCollapse: () => void
}
```

**Itens da sidebar (9 itens fixos):**

| **Item** | **Ícone** | **Route** | **Badge** |
|---|---|---|---|
| Dashboard | `LayoutDashboard` | `/dashboard` | Nenhum |
| Meus Casos | `FolderOpen` | `/casos` | Nenhum |
| Cadastrar Imóvel | `Home` | `/cadastrar` | Nenhum |
| Documentos | `FileText` | `/documentos` | Número de pendentes |
| Propostas | `Tag` | `/propostas` | Número de propostas ativas |
| Assinaturas | `PenLine` | `/assinaturas` | Número de pendentes |
| Financeiro | `DollarSign` | `/financeiro` | Nenhum |
| Assistente IA | `Bot` | `/guardiao` | Nenhum |
| Meu Perfil | `User` | `/perfil` | Nenhum |

**Acessibilidade:** `<nav role="navigation" aria-label="Menu principal">`, cada link com `aria-current="page"` quando ativo.

### T-012 — Shell Mobile (Bottom Tab Bar)

**5 tabs:**

| **Tab** | **Ícone** | **Route** | **Badge** |
|---|---|---|---|
| Dashboard | `LayoutDashboard` | `/dashboard` | Nenhum |
| Meus Casos | `FolderOpen` | `/casos` | Nenhum |
| Documentos | `FileText` | `/documentos` | Pendentes |
| Propostas | `Tag` | `/propostas` | Ativas |
| Mais | `Grid2x2` | (bottom sheet) | Total de pendências |

**"Mais" — bottom sheet:** Assistente IA · Assinaturas · Meu Perfil · [divider] · Sair

**Acessibilidade:** `role="tablist"`, cada tab `role="tab"` + `aria-selected`.

---

## 5. Contratos por Tela — Dashboard (T-013, T-014)

### T-013 — Dashboard com Casos

**Componentes:** `KpiCard` × 4, `CaseCard[]` (lista de casos ativos), `NextStepCard`, feed de eventos.

```typescript
interface DashboardProps {
  kpis: {
    casosAtivos: number
    pendencias: number
    propostasRecebidas: number
    valorEstimadoTotal: number
  }
  casosAtivos: Case[]
  proximosPassos: NextStep[]
  eventosRecentes: Event[]
  loading: boolean
}
```

**Skeleton:** 4 `KpiCard` skeletons (100% width, 80px height) + 2 `CaseCard` skeletons + 3 linhas de eventos.

**Layout desktop:** grid 4 colunas (KPIs) + grid 2/3 (casos) + 1/3 (próximos passos) + feed full-width.

**Layout tablet:** grid 2 colunas (KPIs) + coluna única (casos) + próximos passos + feed.

**Layout mobile:** coluna única, KPIs em scroll horizontal, restante em stack vertical.

### T-014 — Dashboard Primeiro Acesso

**Componente único:** `OnboardingChecklist` com 3 itens, ilustração SVG e CTA.

**Não exibe skeleton** — é tela estática, dados do usuário já estão no contexto de auth.

---

## 6. Contratos por Tela — Meus Casos (T-015 a T-023)

### T-015 — Lista de Casos

```typescript
interface ListaCasosProps {
  casos: Case[]
  filtros: {
    status?: CaseStatus[]
    cenario?: Cenario[]
    busca?: string
  }
  paginacao: {
    pagina: number
    total: number
    porPagina: 10
  }
  loading: boolean
}
```

**Skeleton:** 3 `CaseCard` skeletons (altura 120px cada).

**Empty state (sem casos):** ilustração + "Nenhum caso cadastrado" + botão "Cadastrar imóvel".

**Empty state (com filtros ativos):** "Nenhum caso com esses filtros" + botão "Limpar filtros".

**CaseCard — campos obrigatórios:** endereço, `CaseStatusBadge`, cenário escolhido, valor líquido estimado, data do último evento.

### T-016 — Detalhe do Caso — Visão Geral

**3 zonas de conteúdo (independentes em loading):**

| **Zona** | **Componente** | **Skeleton** |
|---|---|---|
| Zona 1 — Próximo passo | `NextStepCard` com CTA contextual | 1 card, 80px |
| Zona 2 — Resumo financeiro | `FinancialSummary` | 3 linhas de dados |
| Zona 3 — Linha do tempo | `EventTimeline` scroll infinito | 5 linhas de eventos |

**Ações condicionais (kebab menu):**

| **Ação** | **Visível quando** |
|---|---|
| Cancelar Caso | `status` não é `FECHAMENTO`, `POS_FECHAMENTO`, `CONCLUIDO`, `CANCELADO` |
| Solicitar Desistência | `status === 'POS_FECHAMENTO'` dentro de 15 dias |
| Alterar Cenário | `status` é `OFERTA_ATIVA` ou `EM_NEGOCIACAO` + cenário atual ≠ A + cooldown expirado |

### T-017 — Detalhe do Caso — Documentos (aba)

Checklist resumida (máx 8 itens) com `DocumentCard`. Botão "Ver todos" navega para T-040.

**Mobile:** botão câmera inline em cada `DocumentCard` com status `PENDENTE` ou `REJEITADO`.

### T-018 — Detalhe do Caso — Propostas (aba)

Seção "Ativas" com `ProposalCard[]` + seção "Histórico" com lista simplificada.

**Empty state seção ativas (caso disponível mas sem proposta):** "Nenhuma proposta recebida ainda. Seu imóvel está disponível para compradores qualificados."

### T-019 — Detalhe do Caso — Assinaturas (aba)

**Web only para assinar.** Mobile exibe lista somente leitura + instrução "Acesse via computador para assinar documentos."

**Campos por item:** tipo do documento, data de criação, `SlaCountdown` (quando há prazo), status (`DocumentSignatureStatus`), botão "Assinar" (web) ou instrução (mobile).

### T-020 — Detalhe do Caso — Financeiro (aba)

Exibe `EscrowPanel` adaptado para o caso. **Web only** para ações de desistência.

### T-021 — Detalhe do Caso — Histórico de Eventos (aba)

Lista imutável. Paginação: 50 eventos/página. **Sem ações** — somente leitura.

**Campos por evento:** ícone de tipo, descrição textual, data/hora absoluta + relativa (tooltip no hover).

### T-022 e T-023 — Modais de Cancelamento

**T-022 (sem Escrow):** 2 passos. Passo 1: dropdown de motivo. Passo 2: confirmação com aviso de irreversibilidade.

**T-023 (com Escrow ativo):** 2 passos. Passo 2: alerta vermelho proeminente + checkbox obrigatório antes do botão de confirmação.

**Props compartilhadas:**
```typescript
interface CancelamentoModalProps {
  open: boolean
  enderecoImóvel: string
  temEscrowAtivo: boolean
  onConfirmar: (motivo: string) => Promise<void>
  onFechar: () => void
}
```

---

## 7. Contratos por Tela — Cadastro de Imóvel (T-024 a T-030)

### Stepper Global do Wizard (T-024 a T-028)

```typescript
type WizardStep = 1 | 2 | 3 | 4 | 5

interface WizardStepperProps {
  currentStep: WizardStep
  completedSteps: WizardStep[]
}
```

**Estilos por estado:**
- Ativo: círculo `--primary` + label bold
- Concluído: círculo `--chart-2` + ícone check
- Pendente: círculo `--muted` + label regular

**Mobile:** stepper horizontal reduzido (apenas ícones + numeração, labels ocultos).

### T-024 — Wizard Etapa 1: Dados do Imóvel

```typescript
interface Etapa1State {
  cep: string
  logradouro: string            // preenchido automaticamente via CEP
  numero: string
  complemento?: string
  bairro: string
  cidade: string
  estado: string
  construtora: string
  tipoImovel: 'APARTAMENTO' | 'CASA' | 'COBERTURA' | 'TERRENO' | 'COMERCIAL'
  verificandoDuplicidade: boolean
  duplicidadeDetectada: boolean
  loading: boolean
}
```

**Validação assíncrona de duplicidade:** ao `onBlur` no campo `numero` (após CEP preenchido), faz GET `/cases/check-duplicate?endereco=...`. Durante verificação: spinner inline 16px no campo. Se duplicado: alerta âmbar + botão "Ver caso existente" (link para T-016 do caso duplicado).

### T-025 — Wizard Etapa 2: Valores do Contrato

**Campos com currency mask (R$):**
- `valorOriginalContrato` — min R$ 50.000
- `totalPagoAteHoje` — ≤ `valorOriginalContrato`
- `saldoDevedor` — calculado automaticamente: sugestão = `valorOriginalContrato - totalPagoAteHoje`. Editável manualmente. Não pode ser negativo nem > `valorOriginalContrato`.

**Validação de negócio inline (on blur):**
- `totalPagoAteHoje > valorOriginalContrato`: erro "O total pago não pode ser superior ao valor original do contrato."
- `saldoDevedor < 0`: erro "O saldo devedor não pode ser negativo."

### T-026 — Wizard Etapa 3: Simulador de Cenários

**Comportamento do bloqueio de 10s:**
```typescript
interface SimuladorState {
  loading: boolean               // skeleton nos cards durante cálculo
  tempoVisualizacao: number      // segundos desde montagem
  botaoDesbloqueado: boolean     // true após 10s
  cenarios: CenarioSimulado[]
  erro: boolean
}
```

Barra de progresso fino `--primary` no topo da área de cards (não no botão). Stagger de fade-in nos cards após skeleton (100ms entre cada).

**Error state:** banner `<Alert variant="destructive">` + botão "Revisar valores" que navega para Etapa 2 preservando dados.

### T-027 — Wizard Etapa 4: Escolha do Cenário

**RadioGroup sem pré-seleção:**
```typescript
// NUNCA: defaultValue="A"
// SEMPRE:
<RadioGroup value={cenarioSelecionado ?? undefined}>
  {cenarios.map(c => <ScenarioCard key={c.cenario} ... />)}
</RadioGroup>
```

**Checkbox de confirmação:** aparece apenas após seleção, com pulse animation 3s (`--ease-spring`). Botão "Próxima etapa" habilitado somente após seleção + checkbox marcado.

**Erro de navegação sem seleção:** shake animation no `RadioGroup` + mensagem inline.

### T-028 — Wizard Etapa 5: Revisão e Confirmação

Seções colapsáveis. Destaque máximo: `ScenarioCard` do cenário escolhido com "Você receberá R$ [X]" em `--chart-2` bold.

**Botão "Editar" por seção:** navega diretamente para a etapa correspondente preservando todos os dados.

### T-029 — Cadastro Concluído

Dois `NextStepCard` horizontais (desktop) / empilhados (mobile):
1. "Assinar o Termo de Cadastro" → botão "Ir para Assinaturas" → T-043
2. "Enviar documentos do dossiê" → botão "Ir para Documentos" → T-040

Checkmark animado (scale spring bouncy, 300ms) como feedback visual de conclusão.

### T-030 — Banner de Rascunho Salvo

Banner sticky âmbar abaixo do header, visível apenas na rota `/cadastrar` quando há rascunho ativo.

```typescript
interface RascunhoBannerProps {
  enderecoRascunho?: string
  ultimaEdicao: Date
  onContinuar: () => void
  onDescartar: () => void
  loading: boolean
}
```

---

## 8. Contratos por Tela — Cenários e Escalonamento (T-031 a T-034)

### T-031 — Simulação Comparativa de Escalonamento

```typescript
interface EscalonamentoSimulacaoProps {
  cenarioAtual: Cenario
  cenarioPropostoOptions: Cenario[]   // apenas cenário imediatamente inferior
  cenarioSelecionado: Cenario | null  // para múltiplos escalonamentos
  simulacaoAtual: CenarioSimulado
  simulacaoProposta: CenarioSimulado | null
  loading: boolean
  onSolicitar: () => void
  onVoltar: () => void
}
```

**Layout dos cards:**
- Desktop: 2 cards lado a lado (300px cada) + área de diferença financeira entre eles
- Mobile: 2 cards empilhados, diferença abaixo

**Diferença financeira:** seta verde ↑ + "R$ X a mais" OR seta vermelha ↓ + "R$ X a menos" dependendo do cenário comparado.

### T-032 — Modal de Confirmação de Escalonamento

Checkbox de confirmação obrigatório antes do botão de confirmar. Botão "Confirmar" em `--primary` (não destrutivo — escalonamento é solicitado pelo Cedente voluntariamente).

### T-033 — Modal de Escalonamento Enfileirado

`role="alertdialog"` (informativo, não ação destrutiva). Ícone relógio em `--chart-3`.

Botão "Cancelar o enfileiramento" abre confirmação inline antes de executar.

### T-034 — Modal de Subida de Cenário Bloqueada

`role="alertdialog"`. Ícone de bloqueio em `--destructive`. Botão "Cancelar o caso" é destrutivo — abre T-022 ou T-023 conforme contexto.

---

## 9. Contratos por Tela — Propostas (T-035 a T-039)

### T-035 — Lista de Propostas Ativas

**Split view (desktop):** lista à esquerda (320px) + painel de detalhe T-036 à direita (fluido). Click em card seleciona e exibe detalhe sem navegação.

**Mobile:** lista em coluna única. Toque em card navega para T-036.

```typescript
interface ListaPropostasProps {
  propostas: Proposal[]
  propostaSelecionada: string | null   // id — split view desktop
  loading: boolean
}
```

**Ordenação padrão:** valor líquido decrescente.

**Propostas urgentes (< 24h):** `ProposalCard` com borda `--destructive` + badge "Urgente" + `SlaCountdown` em vermelho pulsante.

### T-036 — Detalhe da Proposta

**Hierarquia visual obrigatória (RN-030):**
1. "Você recebe: R$ [valorLiquido]" — `--chart-2`, H1, posição superior
2. "Valor proposto: R$ [valorProposto]" — `--foreground`, H3
3. "Comissão: R$ [comissao]" — `--muted-foreground`, colapsável com chevron

**Histórico de negociação:** lista de rodadas com valores e datas (colapsada por padrão).

### T-037 — Modal de Aceite de Proposta

**2 passos:**
1. Resumo financeiro completo + aviso sobre irreversibilidade
2. Loading state (não há passo 2 visual — o passo 2 é o estado de loading)

```typescript
interface AceitePropostaModalProps {
  open: boolean
  proposta: Proposal
  onConfirmar: () => Promise<void>
  onFechar: () => void
}
```

**Tratamento de erros (RN-032):** após 3 falhas consecutivas → exibe mensagem de suporte + pausa o prazo via API.

### T-038 — Modal de Recusa de Proposta

Dropdown opcional de motivo (não enviado ao Cessionário). Botão de confirmação em `--destructive`.

### T-039 — Modal de Contraproposta

**Validação em tempo real:**
```typescript
// on change do campo de valor
const pisoViolado = valorContraproposta < pisoCenarioAtivo
// exibe alerta âmbar inline + bloqueia botão quando pisoViolado === true
```

Exibe piso do cenário como helper text: "Piso do Cenário [X]: R$ [piso]."

---

## 10. Contratos por Tela — Documentos (T-040 a T-042)

### T-040 — Checklist de Documentos

```typescript
interface ChecklistDocumentosProps {
  caseId: string
  tipoPessoa: 'PF' | 'PJ'
  documentos: Document[]        // 6 para PF, 8 para PJ
  loading: boolean
  // realtime via Supabase — subscribe ao canal 'documents:{caseId}'
}
```

**Contador de progresso:** "X de [6 ou 8] documentos enviados" — barra de progresso `--primary` (0-66%) / `--chart-3` (67-89%) / `--chart-2` (100%).

**Banner de conclusão (auto-dismiss 10s):** fade-out automático após 10s, descartável manualmente.

**Offline:** botão "Enviar" desabilitado + tooltip `aria-describedby="offline-hint"` "Você precisa de conexão para enviar documentos."

**Realtime:** subscribe ao canal `documents:{caseId}` via Supabase Realtime — atualiza `DocumentCard` sem reload.

### T-041 — Upload de Documento

**UploadZone — props:**
```typescript
interface UploadZoneProps {
  documentoId: string
  accept: string                // 'application/pdf,image/jpeg,image/png'
  maxSizeBytes: 10_485_760      // 10 MB
  onUploadComplete: (url: string) => void
  onUploadError: (error: UploadError) => void
}
```

**Estados sequenciais:**
1. `idle`: zona de drop com instrução "Arraste o arquivo ou clique para selecionar"
2. `uploading`: barra de progresso (0–100%) + velocidade estimada + botão "Cancelar"
3. `success`: checkmark animado → fecha após 1s + DocumentCard atualiza para "Em Análise"
4. `error`: mensagem de erro específica + botão "Tentar novamente"

**Banner não-intrusivo:** "Não feche esta janela durante o envio." — `aria-live="polite"`.

### T-042 — Upload por Câmera (Mobile)

**Permissão de câmera:**
- Não solicitada: pede permissão com `expo-camera` antes de abrir
- Negada: tela de bloqueio com instrução + botão "Abrir configurações do dispositivo"

**Preview após captura:** "Usar esta foto" (primário) + "Tirar novamente" (secundário). Confirmar inicia `UploadZone` com imagem capturada.

**Orientação:** aceita retrato e paisagem. Layout adapta automaticamente.

---

## 11. Contratos por Tela — Assinaturas (T-043, T-044)

### T-043 — Lista de Documentos para Assinatura

```typescript
interface ListaAssinaturasProps {
  envelopes: SignatureEnvelope[]
  loading: boolean
}

interface SignatureEnvelope {
  id: string
  tipo: 'TERMO_CADASTRO' | 'INSTRUMENTO_CESSAO' | 'TERMO_ESCALONAMENTO' | 'TERMO_COMERCIAL'
  caseId: string
  enderecoImovel: string
  criadoEm: Date
  prazo?: Date
  status: 'AGUARDANDO' | 'ASSINADO' | 'EXPIRADO'
}
```

**Aviso de prazo crítico:** banner âmbar persistente quando há envelope com `prazo` < 48h.

**Mobile — ação por envelope:**
- `AGUARDANDO`: label "Assinatura disponível pelo computador" (sem botão) — exceto envelopes com suporte a WebView ZapSign
- `ASSINADO`: badge verde
- `EXPIRADO`: badge cinza + opção de solicitar novo envelope (via suporte)

### T-044 — Assinatura Inline (ZapSign)

**Web — drawer lateral:**
```typescript
interface ZapSignDrawerProps {
  envelopeId: string
  zapSignUrl: string
  open: boolean
  onAssinado: (envelopeId: string) => void
  onFechar: () => void
}
```

**Eventos ZapSign (postMessage):**
- `'zapsign:signed'`: fecha drawer com slide-up + toast "Documento assinado com sucesso!" + lista atualiza via Supabase Realtime
- `'zapsign:error'`: toast vermelho + drawer permanece aberto

**Timeout:** se ZapSign não carrega em 10s → mensagem de indisponibilidade + botões "Tentar novamente" / "Cancelar".

**ESC com assinatura em andamento:** confirmação "Fechar sem assinar? Você poderá retornar depois." (`role="alertdialog"`).

---

## 12. Contratos por Tela — Financeiro (T-045 a T-047)

### T-045 — Painel Financeiro Pré-formalização

`EscrowPanel` com `status="NAO_INICIADO"`. Somente leitura. CTA "Ver simulação de cenários" navega para T-026 ou T-031 conforme status do caso.

**Offline:** exibe dados em cache com badge "Dados de [data/hora]. Sem conexão."

### T-046 — Painel Financeiro com Escrow Aberto

`EscrowPanel` com `status="POS_FECHAMENTO"` (ou `DEPOSITO_CONFIRMADO`).

**Hierarquia obrigatória (RN):**
1. "Você receberá: R$ [valorLiquido]" — `--chart-2`, H1 (DESTAQUE MÁXIMO)
2. "Em Escrow: R$ [valorEscrow]" — segundo nível
3. Breakdown de comissão colapsável
4. `SlaCountdown` dos 15 dias de período de segurança
5. Botão "Solicitar Desistência" (text-only, `--destructive`) — visível apenas dentro dos 15 dias

**Após expiração dos 15 dias:** botão some, label "Prazo de desistência encerrado."

### T-047 — Modal de Solicitação de Desistência

**2 passos com aviso vermelho proeminente:**

Passo 1: alerta `<Alert variant="destructive">` no topo + textarea de justificativa com contador de caracteres.

Passo 2: resumo da solicitação + checkbox de ciência + botão "Confirmar desistência" (habilitado após checkbox).

```typescript
interface DesistenciaModalProps {
  open: boolean
  valorEscrow: number
  onConfirmar: (justificativa: string) => Promise<void>
  onFechar: () => void
}
```

---

## 13. Contratos por Tela — Guardião do Retorno (T-048, T-049)

### T-048 — Chat com o Guardião

```typescript
interface GuardiaoChatProps {
  conversaId: string
  mensagens: Mensagem[]
  loading: boolean              // streaming em andamento
  onEnviar: (texto: string) => void
  onEscalar: () => void
}

interface Mensagem {
  id: string
  role: 'CEDENTE' | 'GUARDIAO'
  conteudo: string
  criadaEm: Date
  streaming?: boolean           // cursor caret quando true
}
```

**Primeira abertura (sem histórico):** mensagem proativa do Guardião + 3 botões de sugestão. Botões somem após primeira mensagem enviada.

**Layout desktop:** sidebar de conversas 280px + área de chat fluida.

**Layout mobile:** tela cheia, sidebar acessível via botão "Conversas" no header.

**SSE streaming:** `aria-live="polite"` anuncia mensagem completa (não token a token) para leitores de tela.

**Realtime para novas mensagens enquanto não está na tela:** badge numérico no item "Assistente IA" da sidebar/bottom tab.

### T-049 — Cadastro Assistido (Guardião)

**Diferenças em relação ao T-048:**
1. Banner persistente "Modo: Cadastro Assistido" com botão "Cancelar"
2. `Stepper` horizontal simples (5 etapas) acima da área de chat
3. Ao concluir: botão inline "Revisar e confirmar" no último balão do Guardião → navega para T-028

**Dados coletados no LangGraph:** descartados ao cancelar. Sem criação de rascunho automaticamente.

---

## 14. Contratos por Tela — Perfil (T-050 a T-052)

### T-050 — Meu Perfil — Dados Pessoais

**Tab navigation:** "Dados" (ativa) | "Notificações" | "LGPD"

**Campos por tipo de Cedente:**

| **Campo** | **PF** | **PJ** | **Editável** |
|---|---|---|---|
| Nome completo | Sim | N/A | Sim |
| Razão social | N/A | Sim | Sim |
| CPF | Sim | N/A | Não (`aria-readonly`) |
| CNPJ | N/A | Sim | Não (`aria-readonly`) |
| E-mail | Sim | Sim | Sim (com dupla confirmação) |
| Telefone | Sim | Sim | Sim |
| Nome do representante | N/A | Sim | Não |
| CPF do representante | N/A | Sim | Não |
| Cargo do representante | N/A | Sim | Sim |

**Botão "Salvar alterações":** habilitado apenas quando `formDirty === true`.

**Alteração de e-mail:** campo de novo e-mail + confirmação + botão "Atualizar e-mail" separado do "Salvar alterações".

### T-051 — Meu Perfil — Notificações

**Toggles por categoria:**

| **Categoria** | **Desativável** | **Default** |
|---|---|---|
| Propostas | Sim | Ativo |
| Documentos | Sim | Ativo |
| Assinaturas | Sim | Ativo |
| Financeiro | Sim | Ativo |
| Expiração de sessão | **Não** | Sempre ativo |
| Prazo de proposta < 24h | **Não** | Sempre ativo |
| Desistência / Cancelamento | **Não** | Sempre ativo |

Toggles não desativáveis: `aria-disabled="true"` + tooltip `aria-describedby` "Esta notificação é obrigatória e não pode ser desativada."

`role="switch"` em cada toggle desativável.

### T-052 — Meu Perfil — LGPD / Exclusão de Dados

**Modal de exclusão — 3 etapas sequenciais:**
1. Aviso de consequências (texto) + botão "Continuar"
2. Campo senha (type="password") + validação
3. Checkbox de ciência + botão "Solicitar exclusão" (destrutivo, habilitado após senha válida + checkbox)

**Após sucesso:** protocolo em card com `--muted` background + banner âmbar persistente no topo da página.

---

## 15. Responsividade Global

### 15.1 Breakpoints

| **Breakpoint** | **Range** | **Comportamento** |
|---|---|---|
| Mobile | `< 768px` | Sem sidebar, bottom tab bar T-012, colunas únicas |
| Tablet | `768px–1024px` | Sidebar como drawer overlay (hamburger), 2 colunas em listagens |
| Desktop | `> 1024px` | Sidebar fixa 240px, split views em T-035, layouts 2/3 colunas |

### 15.2 Componentes com Comportamento Diferente por Breakpoint

| **Componente** | **Desktop** | **Tablet** | **Mobile** |
|---|---|---|---|
| `Sidebar` (T-011) | Fixa 240px | Drawer overlay | Ausente (T-012) |
| `ProposalCard` no T-035 | Split view com painel lateral | Lista sem split | Bottom sheet ao toque |
| `ScenarioCard` no T-026/027 | 4 cards em grid | 2 cards × 2 linhas | 1 card por linha, scroll |
| `Stepper` no wizard | Horizontal com labels | Horizontal com labels | Horizontal sem labels |
| `EscrowPanel` | 2 colunas (valor + breakdown) | Coluna única | Coluna única |
| `EventTimeline` (T-016 Zona 3) | Painel lateral 1/3 | Abaixo das zonas 1/2 | Colapsada por padrão |
| `ZapSign` (T-044) | Drawer lateral 640px | Fullscreen | WebView fullscreen |
| `GuardiaoChat` (T-048) | Split: sidebar conversas + área de chat | Área de chat sem sidebar | Tela cheia |
