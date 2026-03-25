# 09 - Contratos de UI por Tela

## Repasse Seguro — Módulo Admin

| **Atributo** | **Valor** |
|---|---|
| **Documento** | 09 - Contratos de UI por Tela |
| **Produto** | Repasse Seguro — Admin |
| **Versão** | v1.0 |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Autor** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Status** | Ativo |
| **Fase** | 2 — UI/UX |
| **Área** | Design + Frontend |
| **Referências** | 01 - Regras de Negócio · 03 - Brand Theme Guide · 06 - Mapa de Telas · 07 - Wireframes · 08 - UX Writing |

---

> 📌 **TL;DR**
>
> - **68 telas cobertas** (T-001 a T-100 com gaps conforme D06).
> - **4 estados obrigatórios** por tela: Skeleton → Empty → Error → Populated. Spinners proibidos globalmente.
> - **3 layouts base utilizados:** `AuthLayout`, `AppLayout`, `ErrorLayout`.
> - **Componentes críticos de domínio:** `CaseStatusBadge`, `KpiCard`, `SlaTimer`, `EscrowGauge`, `AiConfidenceBadge`.
> - **RBAC declarado** por tela via `useRbac()` + `RbacGuard` no router.
> - **Tokens canônicos:** `--primary`, `--destructive`, `--muted`, `--border`, `--background`, `--foreground` do namespace do D03 Brand Guide.
> - **0 telas pendentes** — cobertura total.

---

## 1. Persona

Frontend Lead / UI Designer com foco em transformar cada tela do Admin em contrato explícito de componentes shadcn/ui, estados, tokens e regras RBAC — eliminando ambiguidade de implementação na camada de interface.

---

## 2. Convenções Globais

### 2.1 Quatro Estados Obrigatórios por Tela

| Estado | Componente | Condição de Disparo |
|---|---|---|
| **Skeleton** | `<Skeleton>` shadcn/ui — **nunca `<Spinner>`** | Enquanto fetch em andamento |
| **Empty** | `<EmptyState>` custom (ilustração + CTA) | `data.length === 0` ou sem dados |
| **Error** | `<Alert variant="destructive">` + botão "Tentar novamente" | Request falhou (4xx, 5xx, timeout) |
| **Populated** | Conteúdo real com dados | Após dados carregados com sucesso |

### 2.2 Layouts Base

| Layout | Usado em | Estrutura |
|---|---|---|
| `AuthLayout` | T-001, T-002, T-003, T-004 | Centralizado, card `max-w-[400px]`, fundo `--muted` |
| `AppLayout` | Todas as telas logadas (T-010 a T-100) | Topbar `h-14` + Sidebar `w-64` (desktop) / `w-16` (tablet) + content area |
| `ErrorLayout` | T-097, T-098, T-099 | Centralizado, sem sidebar, sem topbar |

### 2.3 Componentes Críticos de Domínio

| Componente | Telas | Tokens | Faixas Semânticas |
|---|---|---|---|
| `CaseStatusBadge` | T-020, T-022, T-031, T-041, T-051, T-060 | `--primary`, `--destructive`, `--muted-foreground` | Ver mapeamento §2.5 |
| `KpiCard` | T-010 | `--card`, `--primary`, `--foreground` | Verde ≥ meta; Amarelo 70–99%; Vermelho < 70% |
| `SlaTimer` | T-031, T-041, T-070 | `--primary` (normal), amarelo (< 20%), `--destructive` (vencido) | Urgência por tempo restante |
| `EscrowGauge` | T-060, T-061 | `--primary` (100%), amarelo (50–99%), `--destructive` (< 50%) | % do valor esperado depositado |
| `AiConfidenceBadge` | T-070, T-071 | Verde ≥ 0.85, Amarelo 0.60–0.84, Vermelho < 0.60 | Score de confiança do agente IA |
| `DeltaBadge` | T-041, T-042, T-060 | `--primary` (aprovado), `--destructive` (pendente aprovação) | Delta dentro/fora do threshold |

### 2.4 Padrão DataTable

```tsx
<DataTable
  columns={columns}
  data={data}
  skeleton={<TableSkeleton rows={8} />}
  emptyState={<EmptyState icon={...} title="..." description="..." action={...} />}
  pagination={{ pageSize: 25 }}
  // Máximo 100 itens por página — paginação server-side obrigatória acima de 100
/>
```

### 2.5 StatusBadge — Mapeamentos Completos

```
CaseStatus → CaseStatusBadge:
CAPTADO           → bg-slate-100   text-slate-700   "Captado"
EM_TRIAGEM        → bg-blue-100    text-blue-700    "Em Triagem"
BLOQUEADO         → bg-red-100     text-red-700     "Bloqueado"
QUALIFICADO       → bg-green-100   text-green-700   "Qualificado"
OFERTA_ATIVA      → bg-indigo-100  text-indigo-700  "Oferta Ativa"
EM_NEGOCIACAO     → bg-violet-100  text-violet-700  "Em Negociação"
EM_FORMALIZACAO   → bg-amber-100   text-amber-700   "Em Formalização"
FECHAMENTO        → bg-emerald-100 text-emerald-700 "Fechamento"
POS_FECHAMENTO    → bg-teal-100    text-teal-700    "Pós Fechamento"
EM_REVERSAO       → bg-orange-100  text-orange-700  "Em Reversão"
EM_MEDIACAO       → bg-yellow-100  text-yellow-700  "Em Mediação"
DISPUTA_FORMAL    → bg-red-200     text-red-800     "Disputa Formal"
CONCLUIDO         → bg-green-200   text-green-800   "Concluído"
CANCELADO         → bg-gray-100    text-gray-500    "Cancelado"

UserRole → RoleBadge:
ANALISTA          → bg-blue-50     text-blue-600    "Analista"
COORDENADOR       → bg-violet-50   text-violet-600  "Coordenador"
GESTOR_FINANCEIRO → bg-amber-50    text-amber-600   "Gestor Financeiro"
MASTER            → bg-red-50      text-red-700     "Master"

UserExternalStatus → ExternalStatusBadge:
ATIVO    → bg-green-100  text-green-700  "Ativo"
SUSPENSO → bg-yellow-100 text-yellow-700 "Suspenso"
INATIVO  → bg-gray-100   text-gray-500   "Inativo"

DocumentStatus → DocStatusBadge:
PENDENTE  → bg-slate-100  text-slate-600  "Pendente"
ENVIADO   → bg-blue-100   text-blue-600   "Enviado"
APROVADO  → bg-green-100  text-green-700  "Aprovado"
REJEITADO → bg-red-100    text-red-700    "Rejeitado"

ProposalStatus → ProposalStatusBadge:
PENDENTE  → bg-amber-100   text-amber-700  "Pendente"
ACEITA    → bg-green-100   text-green-700  "Aceita"
RECUSADA  → bg-red-100     text-red-700    "Recusada"
EXPIRADA  → bg-gray-100    text-gray-500   "Expirada"
CANCELADA → bg-gray-200    text-gray-600   "Cancelada"

EnvelopeStatus → EnvelopeStatusBadge:
PENDENTE              → bg-slate-100   text-slate-600   "Pendente"
ENVIADO               → bg-blue-100    text-blue-600    "Enviado"
PARCIALMENTE_ASSINADO → bg-amber-100   text-amber-700   "Parcial"
ASSINADO              → bg-green-100   text-green-700   "Assinado"
EXPIRADO              → bg-gray-100    text-gray-500    "Expirado"
CANCELADO             → bg-red-100     text-red-600     "Cancelado"

EscrowAccountStatus → EscrowStatusBadge:
AGUARDANDO_DEPOSITO   → bg-slate-100   text-slate-600   "Aguardando"
DEPOSITO_CONFIRMADO   → bg-blue-100    text-blue-700    "Depositado"
DISTRIBUICAO_PENDENTE → bg-amber-100   text-amber-700   "Pendente"
DISTRIBUIDO           → bg-green-100   text-green-700   "Distribuído"
BLOQUEADO             → bg-red-100     text-red-700     "Bloqueado"
ESTORNADO             → bg-gray-200    text-gray-600    "Estornado"
```

---

## 3. Autenticação (T-001 a T-004)

### T-001 — Login

| Atributo | Valor |
|---|---|
| **Layout** | `AuthLayout` |
| **Role mínimo** | Público (não autenticado) |
| **RFs** | RF-001, RF-002 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, `max-w-[400px]`, shadow-sm | Container do formulário de login |
| `Input` | Default, `type="email"` | Campo e-mail |
| `Input` | Default, `type="password"`, com ícone de toggle | Campo senha |
| `Button` | `variant="default"`, size="default", `w-full` | Submit login |
| `Alert` | `variant="destructive"` | Erro de credenciais inválidas |
| `Skeleton` | `h-10 w-full` | Estado loading do form |

**Acessibilidade:**
| Requisito | Implementação | Critério |
|---|---|---|
| Focus management | Auto-focus no campo e-mail ao montar | Campo e-mail recebe foco automaticamente |
| ARIA roles | `role="form"`, `aria-label="Formulário de login"` | Screen reader anuncia formulário |
| Contraste | `--foreground` sobre `--background` | Ratio ≥ 4.5:1 |
| Motion | `prefers-reduced-motion` respeita Framer Motion | Animações desabilitadas |

**Motion:**
- Entrada: `fade-in` 200ms `ease-out`
- Card: `scale-in` 150ms ao montar
- Skeleton: pulse padrão shadcn

**Regras específicas:**
- Após 5 tentativas falhas, bloquear por 30min (RN-005) — exibir `Alert destructive` com contador regressivo
- Senha: toggle de visibilidade obrigatório
- "Esqueci minha senha" → link para T-003

---

### T-002 — 2FA — Verificação de Código

| Atributo | Valor |
|---|---|
| **Layout** | `AuthLayout` |
| **Role mínimo** | Sessão intermediária (pré-autenticado) |
| **RFs** | RF-001, RF-003 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, `max-w-[400px]` | Container 2FA |
| `InputOTP` | Default, 6 dígitos, `autoFocus` | Campo de código TOTP |
| `Button` | `variant="default"`, `w-full` | Verificar código |
| `Button` | `variant="ghost"`, `w-full` | Reenviar código / usar código de recuperação |
| `Alert` | `variant="destructive"` | Código inválido / expirado |
| `Skeleton` | `h-10 w-full` | Loading durante verificação |

**Acessibilidade:**
| Requisito | Implementação | Critério |
|---|---|---|
| Focus management | Auto-focus no primeiro dígito do OTP | Cursor posicionado no input ao montar |
| ARIA | `aria-label="Código de verificação 2FA"` | Screen reader anuncia contexto |
| Contraste | Ratio ≥ 4.5:1 | Validado via axe-core |

**Motion:** Fade-in 200ms. Shake animation no `InputOTP` quando código incorreto.

**Regras específicas:**
- Código TOTP de 6 dígitos, auto-submit ao completar
- Link "Usar código de recuperação" visível mas secundário

---

### T-003 — Recuperação de Senha — Solicitação

| Atributo | Valor |
|---|---|
| **Layout** | `AuthLayout` |
| **Role mínimo** | Público |
| **RFs** | RF-004 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, `max-w-[400px]` | Container |
| `Input` | `type="email"` | Campo e-mail para envio do link |
| `Button` | `variant="default"`, `w-full` | Enviar link de recuperação |
| `Alert` | `variant="default"` | Confirmação de envio |
| `Skeleton` | `h-10 w-full` | Loading |

**Acessibilidade:** Focus no campo e-mail ao montar. ARIA label no formulário.

**Motion:** Fade-in 200ms. Success state com `Alert` slide-down 150ms.

**Regras específicas:**
- Sempre exibir mensagem de sucesso genérica (não revelar se e-mail existe — prevenção de enumeração)
- Link "Voltar ao login" sempre visível

---

### T-004 — Recuperação de Senha — Redefinição

| Atributo | Valor |
|---|---|
| **Layout** | `AuthLayout` |
| **Role mínimo** | Token de reset válido (link por e-mail) |
| **RFs** | RF-004 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, `max-w-[400px]` | Container |
| `Input` | `type="password"` + toggle | Nova senha |
| `Input` | `type="password"` + toggle | Confirmação da nova senha |
| `Progress` | `value={strength}` cor semântica | Indicador de força da senha |
| `Button` | `variant="default"`, `w-full` | Salvar nova senha |
| `Alert` | `variant="destructive"` | Senhas não coincidem / token expirado |

**Motion:** Fade-in 200ms. `Progress` anima linearmente ao digitar.

**Regras específicas:**
- Senha mínima: 8 caracteres, 1 maiúscula, 1 número, 1 especial
- Token expirado → exibir `Alert destructive` com link para solicitar novo

---

## 4. Dashboard (T-010 a T-012)

### T-010 — Dashboard Principal

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-005, RF-006, RF-007 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `KpiCard` (custom) | Com ícone `lucide-react`, valor em negrito, delta % | 4 KPIs principais: casos ativos, fechamentos do mês, taxa de conversão, receita |
| `Card` | Default | Container de cada KPI |
| `Skeleton` | `h-[120px] w-full` (x4) | Loading dos KPIs |
| `ChartAreaInteractive` (shadcn charts) | `type="area"`, tokens `--chart-1` a `--chart-5` | Gráfico de volume de casos por status no tempo |
| `ChartBar` | tokens `--chart-1` a `--chart-3` | Conversão por estágio do funil |
| `Table` + `DataTable` | Default, 5 linhas, sem paginação | Últimos 5 casos movimentados |
| `Badge` = `CaseStatusBadge` | Mapeamento §2.5 | Status do caso na tabela |
| `Alert` | `variant="destructive"` | Erro ao carregar dashboard |

**Acessibilidade:**
| Requisito | Implementação | Critério |
|---|---|---|
| Focus management | Skip link "Ir para conteúdo principal" | Primeiro elemento focável |
| ARIA | `role="region"` em cada seção de KPI e gráfico | Screen reader anuncia seções |
| Contraste gráficos | Tokens `--chart-*` com ratio ≥ 3:1 | Validado via axe-core |
| Motion | `prefers-reduced-motion` desabilita animações de gráficos | Sem movimento quando ativo |

**Motion:** Fade-in geral 300ms. KpiCards: stagger de 50ms cada. Gráficos: animate-in draw 600ms.

**Regras específicas:**
- KPIs exibem delta percentual vs. período anterior (seta verde/vermelha)
- Timestamp "Atualizado há Xmin" no canto superior direito de cada card (DIV-002 do B04 — pendente validação produto)
- Gráficos usam tokens `--chart-1` a `--chart-5` do D03

---

### T-011 — Dashboard — Alertas e Pendências

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-006, RF-008 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Alert` | `variant="destructive"` | Alertas críticos (SLA vencido) |
| `Alert` | `variant="default"` | Alertas de atenção |
| `Badge` | `variant="outline"` | Contador de pendências por tipo |
| `ScrollArea` | `h-[400px]` | Lista scrollável de alertas |
| `Skeleton` | `h-16 w-full` (x5) | Loading |

**Motion:** Fade-in lista 200ms. `Alert` crítico aparece com border pulse 1s se `aria-live="assertive"`.

**Regras específicas:**
- SLA Timer: componente `SlaTimer` com cores por urgência (ver §2.3)
- Alertas críticos têm `aria-live="assertive"` para screen readers

---

### T-012 — Busca Global

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal sobre layout) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-009 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `CommandDialog` | Default, acionado por `Ctrl+K` | Container de busca global |
| `CommandInput` | Default, `placeholder="Buscar casos, cedentes..."` | Campo de busca |
| `CommandList` | Default | Resultados agrupados |
| `CommandGroup` | Default | Grupos: Casos, Cedentes, Cessionários |
| `CommandItem` | Default | Item de resultado com badge de tipo |
| `Skeleton` | `h-8 w-full` (x3) | Loading de resultados |

**Acessibilidade:** Focus trap no `CommandDialog`. Navegação por setas do teclado. `Escape` fecha.

**Motion:** Dialog: fade + scale 150ms. Items: stagger 30ms.

**Regras:** Busca por case ID, CPF, nome, endereço. Resultado clicável navega para tela do caso.

---

## 5. Pipeline (T-020 a T-025)

### T-020 — Pipeline — Kanban

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-010, RF-011, RF-012 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `ScrollArea` | `orientation="horizontal"` | Kanban com scroll horizontal entre colunas |
| `Card` = `PipelineCard` (custom) | `w-[280px]`, hover shadow-md | Card de caso no Kanban |
| `CaseStatusBadge` | Mapeamento §2.5 | Status no card |
| `Badge` | `variant="outline"` | Analista atribuído, prazo SLA |
| `SlaTimer` | Por card quando SLA < 80% tempo restante | Urgência de SLA |
| `Button` | `variant="ghost"`, `size="icon"` | Menu de ações rápidas no card |
| `Skeleton` | `w-[280px] h-[140px]` (x6 por coluna) | Loading das colunas |
| `EmptyState` | Ícone `PackageOpen`, "Nenhum caso neste estágio" | Coluna vazia |

**Acessibilidade:**
| Requisito | Implementação | Critério |
|---|---|---|
| DnD acessível | Alternativa de teclado via menu "Mover para..." no card | DnD não é único método de reordenação |
| ARIA | `role="list"` nas colunas, `role="listitem"` nos cards | Screen reader anuncia colunas e cards |
| Focus | Tab navega entre cards; Enter abre detalhes | Navegação sem mouse |

**Motion:** Fade-in das colunas em stagger 80ms. Card: scale 1.02 + shadow no hover. Drag: ghost com opacity 0.5.

**Regras específicas:**
- Supabase Realtime — atualização automática ≤ 5s (RN) — Pulse visual leve ao receber update
- Cards paginados em 10 por coluna com "Ver mais" quando > 10 (DEC-005 do B04)
- Drag-and-drop limita transições apenas às permitidas pelo ciclo de vida

---

### T-021 — Pipeline — Lista

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-010, RF-011, RF-013 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: ID, Cedente, Status, Analista, Criado em, SLA | Listagem tabulada de todos os casos |
| `CaseStatusBadge` | Mapeamento §2.5 | Status na tabela |
| `SlaTimer` | inline na coluna SLA | Urgência visual |
| `Input` | `type="search"`, `placeholder="Filtrar..."` | Filtro inline |
| `Select` | Default, multiple | Filtro por status, analista, cenário |
| `Button` | `variant="outline"` | Exportar CSV |
| `TableSkeleton` | 8 linhas | Loading |

**Motion:** Fade-in tabela 200ms. Row hover: `bg-muted` transition 100ms.

**Regras:** Paginação server-side, 25 itens/página. Ordenação por coluna. Filtros persistem na URL.

---

### T-022 — Pipeline — Detalhe do Caso

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-014, RF-015, RF-016 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Tabs` | Default, linha: Resumo / Histórico / Documentos / Timeline | Navegação interna do caso |
| `Card` | Default | Container de seção (dados do cedente, dados financeiros) |
| `CaseStatusBadge` | Mapeamento §2.5 | Status atual com visual proeminente |
| `Badge` | `variant="secondary"` | Cenário (A/B/C/D) |
| `Table` | Default | Histórico de mudanças de status |
| `ScrollArea` | `h-[400px]` | Timeline de atividades |
| `Button` | `variant="default"` | Ações: Avançar status, Atribuir analista |
| `Button` | `variant="destructive"` | Cancelar caso (Master/Coordenador) |
| `Skeleton` | Múltiplos por seção | Loading |
| `Alert` | `variant="destructive"` | Erro ao carregar |

**Acessibilidade:** Focus management ao trocar aba. `aria-current` na aba ativa.

**Motion:** Fade-in geral 200ms. Tab troca: fade 150ms. Skeleton em todas as seções.

**Regras:** Botão "Avançar Status" respeita máquina de estados — desabilitado se transição não permitida. Cancelamento exige `ConfirmModal md`.

---

### T-023 — Pipeline — Criar Caso

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer lateral) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-017, RF-018 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Sheet` | `side="right"`, `size="lg"` | Drawer de criação do caso |
| `Form` + `FormField` | React Hook Form + Zod | Formulário multi-campo |
| `Input` | Default | Endereço do imóvel, valores |
| `Select` | Default | Cedente (busca), Cenário (A/B/C/D), Analista |
| `Button` | `variant="default"`, `w-full` | Salvar caso |
| `Button` | `variant="ghost"` | Cancelar |
| `Alert` | `variant="destructive"` | Erro de validação / API |
| `Skeleton` | `h-10 w-full` (x5) | Loading ao carregar dados de suporte |

**Motion:** Drawer slide-in da direita 300ms. Overlay fade-in 200ms.

**Regras:** Ao salvar, cria snapshot de config (RN-111). Validação inline (Zod). Analista sugerido pelo sistema via FIFO.

---

### T-024 — Pipeline — Atribuição de Analista

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-019 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Dialog` | Default, `max-w-sm` | Modal de atribuição |
| `Select` | Default, busca de analistas ativos | Seleção do analista |
| `Button` | `variant="default"` | Confirmar atribuição |
| `Button` | `variant="ghost"` | Cancelar |

**Motion:** Dialog fade + scale 150ms.

**Regras:** Apenas analistas com `is_active = true` listados. Confirmação registra em `case_status_history`.

---

### T-025 — Pipeline — Cancelamento de Caso

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-020 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `AlertDialog` | Default | Modal destrutivo de confirmação |
| `Textarea` | Default, `required`, min 20 chars | Motivo do cancelamento |
| `Button` | `variant="destructive"` | Confirmar cancelamento |
| `Button` | `variant="ghost"` | Cancelar |

**Motion:** AlertDialog fade + scale 150ms. `--destructive` no botão de confirmação.

**Regras:** Motivo obrigatório. Cancelamento irreversível via UI (somente Master pode reverter no banco).

---

## 6. Triagem (T-030 a T-035)

### T-030 — Triagem — Fila FIFO

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-021, RF-022, RF-023 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, borda-l-4 colorida por urgência | Card de caso na fila |
| `CaseStatusBadge` | `EM_TRIAGEM` / `BLOQUEADO` | Status |
| `SlaTimer` | Por card | SLA da triagem (48h) |
| `Button` | `variant="default"` | "Iniciar Triagem" |
| `Badge` | `variant="outline"` | Posição na fila |
| `Skeleton` | `h-[100px] w-full` (x5) | Loading da fila |
| `EmptyState` | "Fila vazia — nenhum caso aguardando" | Empty state |

**Acessibilidade:** `role="list"` na fila. Cada card com `role="listitem"` e `aria-label` descritivo.

**Motion:** Fade-in stagger 60ms por card. Novo caso entra com slide-down 200ms.

**Regras:** FIFO rigoroso (RN-055). Ordenação por `created_at ASC`. Analista vê apenas casos não atribuídos ou atribuídos a si.

---

### T-031 — Triagem — Análise do Caso

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-021, RF-023, RF-024 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Dados do caso (cedente, imóvel, valores) |
| `Badge` | `variant="secondary"` | Cenário financeiro |
| `SlaTimer` | Proeminente no topo | SLA 48h restante |
| `Tabs` | Dados / Documentos / Histórico | Navegação interna |
| `Button` | `variant="default"` | "Qualificar" |
| `Button` | `variant="outline"` | "Bloquear" |
| `Button` | `variant="destructive"` | "Cancelar" |
| `Textarea` | Default, para justificativa | Motivo de bloqueio |
| `Skeleton` | Por seção | Loading |

**Acessibilidade:** Focus management ao trocar aba. Botões de ação com `aria-describedby` apontando para resumo do caso.

**Motion:** Fade-in 200ms. Ações de estado (qualificar/bloquear) abrem `AlertDialog` confirmatório.

**Regras:** Ação "Qualificar" → status `QUALIFICADO`. Ação "Bloquear" requer motivo e → status `BLOQUEADO`.

---

### T-032 — Triagem — Bloqueio de Caso

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-024 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Dialog` | Default | Modal de bloqueio |
| `Textarea` | `required`, min 20 chars | Motivo do bloqueio |
| `Select` | Default | Categoria do bloqueio (documentação pendente, dado incorreto, etc.) |
| `Button` | `variant="default"` | Confirmar bloqueio |
| `Button` | `variant="ghost"` | Cancelar |

**Motion:** Dialog fade + scale 150ms.

**Regras:** Motivo obrigatório. Registra em `case_status_history`. Caso bloqueado pode retornar à triagem.

---

### T-033 — Triagem — Reabrir Caso Bloqueado

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-025 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Dialog` | Default, `max-w-sm` | Confirmação de reabertura |
| `Textarea` | `required` | Justificativa de reabertura |
| `Button` | `variant="default"` | Reabrir |
| `Button` | `variant="ghost"` | Cancelar |

**Motion:** Dialog fade + scale 150ms.

**Regras:** Reabre caso `BLOQUEADO` → `EM_TRIAGEM`. Registra em histórico com justificativa.

---

### T-034 — Triagem — Histórico de Triagem

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-026 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: Caso, Analista, Status, Data, SLA | Histórico de casos triados |
| `CaseStatusBadge` | Mapeamento §2.5 | Status na tabela |
| `Select` | Default | Filtro por analista, status, período |
| `Skeleton` | 8 linhas | Loading |

**Motion:** Fade-in tabela 200ms.

**Regras:** Paginação 25 itens. Ordenação por data desc.

---

### T-035 — Triagem — SLA e Alertas

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-027 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, borda colorida por urgência | Card de SLA por analista |
| `Progress` | `value={pct}` cor semântica | % do SLA consumido |
| `Badge` | `variant="destructive"` | SLA vencido |
| `Skeleton` | `h-[80px] w-full` (x4) | Loading |

**Motion:** Fade-in stagger 60ms. `Progress` anima ao carregar.

**Regras:** SLA timer usa cores: verde > 50%, amarelo 20–50%, vermelho < 20%.

---

## 7. Negociação (T-040 a T-047)

### T-040 — Negociação — Lista de Casos em Negociação

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-028, RF-029 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: Caso, Cedente, Cessionário, Status, Proposta, SLA | Lista de negociações |
| `CaseStatusBadge` | `OFERTA_ATIVA` / `EM_NEGOCIACAO` | Status |
| `ProposalStatusBadge` | Mapeamento §2.5 | Status da proposta |
| `Button` | `variant="ghost"`, `size="sm"` | Abrir detalhe |
| `Skeleton` | 8 linhas | Loading |

**Motion:** Fade-in tabela 200ms.

**Regras:** Filtro por status ativo por padrão. Paginação 25 itens.

---

### T-041 — Negociação — Detalhe da Negociação

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-028, RF-030, RF-031 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Resumo financeiro: saldo devedor, Delta, cenário |
| `DeltaBadge` | Custom — ver §2.3 | Status de aprovação do Delta |
| `SlaTimer` | No topo | SLA de negociação |
| `Tabs` | Proposta Atual / Histórico / Contraproposta | Navegação |
| `Table` | Default | Histórico de propostas e contrapropostas |
| `ProposalStatusBadge` | Mapeamento §2.5 | Status da proposta |
| `Button` | `variant="default"` | Criar contraproposta |
| `Button` | `variant="outline"` | Marcar como aceita |
| `Skeleton` | Por seção | Loading |

**Acessibilidade:** Focus management ao trocar aba. `aria-live="polite"` no status da proposta.

**Motion:** Fade-in 200ms. Tabs: fade 150ms.

**Regras:** Delta ≥ threshold → exibir `DeltaBadge` vermelho e bloquear avanço sem aprovação Master/Coordenador.

---

### T-042 — Negociação — Criar Proposta

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-030 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Sheet` | `side="right"`, `size="md"` | Drawer de nova proposta |
| `Input` | `type="number"`, currency mask | Valor proposto |
| `Select` | Default | Cessionário |
| `DatePicker` | Default | Data de expiração |
| `Button` | `variant="default"`, `w-full` | Enviar proposta |
| `Alert` | `variant="destructive"` | Erro de validação |

**Motion:** Slide-in 300ms.

**Regras:** Expiração padrão = D+7. Proposta cria registro em `proposals` e notifica Cessionário via WhatsApp.

---

### T-043 — Negociação — Contraproposta

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-031 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Sheet` | `side="right"`, `size="md"` | Drawer de contraproposta |
| `Input` | `type="number"`, currency mask | Valor da contraproposta |
| `Textarea` | Default | Justificativa |
| `Button` | `variant="default"`, `w-full` | Enviar |
| `Alert` | `variant="default"` | Contexto da proposta original |

**Motion:** Slide-in 300ms.

**Regras:** Registra em `counterproposals` (append-only). Notifica via WhatsApp.

---

### T-044 — Negociação — Aprovar Delta

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-032 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Dialog` | Default | Modal de aprovação de Delta |
| `Card` | Default | Resumo: valor do Delta, threshold, impacto |
| `Textarea` | Default | Justificativa da aprovação |
| `Button` | `variant="default"` | Aprovar |
| `Button` | `variant="destructive"` | Rejeitar |
| `Button` | `variant="ghost"` | Cancelar |

**Motion:** Dialog fade + scale 150ms.

**Regras:** Aprovação registra `delta_approved = true`, `delta_approved_by`, `delta_approved_at` em `formalization_criteria`.

---

### T-045 — Negociação — Histórico de Negociações

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-033 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: Caso, Resultado, Proposta aceita, Analista, Data | Histórico |
| `ProposalStatusBadge` | Mapeamento §2.5 | Status da proposta |
| `Skeleton` | 8 linhas | Loading |

**Motion:** Fade-in tabela 200ms.

---

### T-046 — Negociação — Cadastro de Cessionário

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-034 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Sheet` | `side="right"`, `size="md"` | Drawer de cadastro |
| `Form` + `FormField` | RHF + Zod | Nome, CPF (mask), e-mail, telefone |
| `Input` | Default com máscaras | Campos do Cessionário |
| `Button` | `variant="default"`, `w-full` | Salvar |
| `Alert` | `variant="destructive"` | CPF/e-mail já cadastrado |

**Motion:** Slide-in 300ms.

---

### T-047 — Negociação — Editar Cessionário

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-034 |

Estrutura idêntica a T-046, porém com dados pré-preenchidos. Versão Optimistic Lock verificada antes de salvar.

---

## 8. Formalização (T-050 a T-058)

### T-050 — Formalização — Visão Geral do Caso

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-035, RF-036 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Status dos 4 critérios de fechamento |
| `Checkbox` | Default, read-only para Analista | Critérios: assinaturas, anuência, depósito, instrumento |
| `Badge` | `variant="outline"` verde/vermelho | Critério concluído/pendente |
| `Progress` | `value={completed/4 * 100}` | % de critérios concluídos |
| `Button` | `variant="default"` | "Avançar para Fechamento" (habilitado só com 4/4) |
| `Skeleton` | Por card | Loading |

**Acessibilidade:** `aria-checked` nos checkboxes. `role="progressbar"` no `Progress`.

**Motion:** Fade-in 200ms. `Progress` anima ao carregar. Checkmark: scale + fade 150ms.

**Regras:** Botão "Fechamento" desabilitado até 4/4 critérios (RN-022/023).

---

### T-051 — Formalização — Dossiê de Documentos

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-037, RF-038 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, por documento | Item do dossiê |
| `DocStatusBadge` | Mapeamento §2.5 | Status de cada documento |
| `Button` | `variant="outline"` | Upload de documento |
| `Button` | `variant="ghost"` | Visualizar documento |
| `Button` | `variant="default"` | Aprovar documento (Coordenador) |
| `Button` | `variant="destructive"` | Rejeitar documento |
| `Textarea` | Default | Motivo da rejeição |
| `Progress` | `value={approved/total * 100}` | % do dossiê concluído |
| `Skeleton` | `h-[80px] w-full` (por doc) | Loading |

**Acessibilidade:** `aria-label` em cada card de documento. Focus management ao aprovar/rejeitar.

**Motion:** Upload: progress bar animada. Aprovação: badge muda com fade 150ms.

**Regras:** Documento rejeitado mostra motivo. Documento nunca hard-deleted (RN-002).

---

### T-052 — Formalização — Upload de Documento

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-037 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Dialog` | Default, `max-w-md` | Modal de upload |
| `Input` | `type="file"`, `accept=".pdf,.jpg,.png"` | Seleção do arquivo |
| `Select` | Default | Tipo do documento |
| `Progress` | `value={uploadPct}` | Progresso do upload |
| `Button` | `variant="default"`, `w-full` | Confirmar upload |
| `Alert` | `variant="destructive"` | Erro (tamanho, formato) |

**Motion:** Dialog fade + scale 150ms. Progress anima linearmente.

**Regras:** Máximo 10MB por arquivo. Formatos: PDF, JPG, PNG. Arquivo criptografado no Supabase Storage.

---

### T-053 — Formalização — Envelopes ZapSign

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-039, RF-040 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Card por envelope |
| `EnvelopeStatusBadge` | Mapeamento §2.5 | Status do envelope |
| `Badge` | `variant="outline"` | Signatários pendentes vs. concluídos |
| `Button` | `variant="default"` | Criar envelope |
| `Button` | `variant="outline"` | Reenviar |
| `Button` | `variant="ghost"` | Visualizar histórico |
| `Skeleton` | `h-[100px] w-full` (x3) | Loading |

**Motion:** Fade-in stagger 80ms por card.

**Regras:** Reenvio registra `resend_count++`. Status atualizado via webhook ZapSign → Supabase Realtime.

---

### T-054 — Formalização — Criar Envelope

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-039 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Sheet` | `side="right"`, `size="md"` | Drawer de criação |
| `Select` | Default | Tipo de documento do envelope |
| `Input` | Default | Nome e e-mail de cada signatário |
| `Button` | `variant="outline"` | Adicionar signatário |
| `Button` | `variant="default"`, `w-full` | Criar e enviar |
| `Alert` | `variant="destructive"` | Erro de criação |

**Motion:** Slide-in 300ms.

---

### T-055 — Formalização — Critérios de Fechamento

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-041, RF-042 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Por critério de fechamento |
| `Switch` | Default | Toggle dos critérios editáveis (anuência) |
| `Checkbox` | Default | Critérios auto-verificados (assinaturas, depósito) |
| `Input` | Default | Protocolo de anuência, caminho do doc |
| `Badge` | Verde (ok) / Vermelho (pendente) | Status visual |
| `Button` | `variant="default"` | Marcar anuência como N/A (DEC-013) |
| `Skeleton` | Por critério | Loading |

**Motion:** Fade-in 200ms. Switch: transição 200ms. Badge muda com fade 150ms.

**Regras:** Anuência pode ser N/A quando construtora não exige (DEC-013 do B04). Depósito verificado automaticamente via Celcoin webhook.

---

### T-056 — Formalização — Reversão de Caso

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-043, RF-044 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `AlertDialog` | Default | Confirmação destrutiva de reversão |
| `Card` | Default | Prazo restante de reversão (15 dias) |
| `Textarea` | `required` | Motivo da reversão |
| `Button` | `variant="destructive"` | Confirmar reversão |
| `Button` | `variant="ghost"` | Cancelar |

**Motion:** AlertDialog fade + scale 150ms. Prazo exibido com `SlaTimer` countdown.

**Regras:** Reversão disponível apenas em `POS_FECHAMENTO` dentro de 15 dias (RN). Registra em histórico.

---

### T-057 — Formalização — Mediação

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-045 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Dados da mediação: partes, data de início |
| `Textarea` | Default | Notas da mediação |
| `Badge` | `variant="secondary"` | Status da mediação |
| `Button` | `variant="default"` | Encaminhar para disputa formal |
| `Skeleton` | Por seção | Loading |

---

### T-058 — Formalização — Disputa Formal

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | MASTER |
| **RFs** | RF-046 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Dados da disputa: partes, valor em disputa |
| `Textarea` | Default | Decisão e fundamentação |
| `Button` | `variant="default"` | Registrar resolução |
| `Button` | `variant="destructive"` | Cancelar caso (resolução negativa) |
| `Skeleton` | Por seção | Loading |

---

## 9. Financeiro (T-060 a T-067)

### T-060 — Financeiro — Dashboard Financeiro

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | GESTOR_FINANCEIRO |
| **RFs** | RF-047, RF-048 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `KpiCard` | Tokens `--primary`, `--chart-1` | KPIs: total em escrow, distribuídos, comissões, pendentes |
| `EscrowGauge` | Custom radial — ver §2.3 | % de depósito por caso |
| `DataTable` | Colunas: Caso, Status Escrow, Esperado, Depositado, Ação | Lista de contas escrow |
| `EscrowStatusBadge` | Mapeamento §2.5 | Status da conta |
| `Skeleton` | Por seção | Loading |
| `Alert` | `variant="destructive"` | Erro ao carregar |

**Acessibilidade:** `role="region"` nas seções. `aria-label` nos gauges.

**Motion:** Fade-in 300ms. Gauges animam ao carregar (arc draw 600ms). Sem animação com `prefers-reduced-motion`.

**Regras:** Supabase Realtime — atualização ao vivo do saldo escrow. Timestamp de última atualização visível (DIV-002).

---

### T-061 — Financeiro — Detalhe da Conta Escrow

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | GESTOR_FINANCEIRO |
| **RFs** | RF-048, RF-049 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Saldo atual, esperado, depositado, residual |
| `EscrowStatusBadge` | Mapeamento §2.5 | Status da conta |
| `Table` | Default | Histórico de transações |
| `EscrowGauge` | Custom | % do valor esperado |
| `Button` | `variant="default"` | Aprovar distribuição (Gestor Financeiro) |
| `Button` | `variant="destructive"` | Bloquear distribuição |
| `Skeleton` | Por seção | Loading |

**Motion:** Fade-in 200ms. Transações: scroll infinito (append-only).

---

### T-062 — Financeiro — Distribuição de Valores

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | GESTOR_FINANCEIRO |
| **RFs** | RF-050, RF-051 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Sheet` | `side="right"`, `size="lg"` | Drawer de distribuição |
| `Card` | Default | Resumo: valores por destinatário |
| `Table` | Default | Breakdown: Cedente, Cessionário, RS, Construtora |
| `Input` | `type="number"`, currency mask, read-only | Valores calculados |
| `Button` | `variant="default"`, `w-full` | Confirmar distribuição |
| `AlertDialog` | Default | Confirmação final antes de processar |
| `Alert` | `variant="destructive"` | Erro de API Celcoin |

**Motion:** Slide-in 300ms. AlertDialog fade + scale 150ms.

**Regras:** Distribuição bloqueada requer aprovação Master (RN). Breakdown calculado automaticamente com percentuais do snapshot.

---

### T-063 — Financeiro — Bloquear Distribuição

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | GESTOR_FINANCEIRO |
| **RFs** | RF-052 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Dialog` | Default | Modal de bloqueio |
| `Textarea` | `required`, min 20 chars | Motivo do bloqueio |
| `Button` | `variant="destructive"` | Confirmar bloqueio |
| `Button` | `variant="ghost"` | Cancelar |

**Regras:** Bloqueio requer aprovação do Master para ser desbloqueado (RN-100).

---

### T-064 — Financeiro — Aprovar Bloqueio (Master)

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | MASTER |
| **RFs** | RF-053 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `AlertDialog` | Default | Confirmação de aprovação de bloqueio |
| `Card` | Default | Detalhes do bloqueio solicitado |
| `Button` | `variant="default"` | Aprovar |
| `Button` | `variant="destructive"` | Recusar |

---

### T-065 — Financeiro — Comissões

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | GESTOR_FINANCEIRO |
| **RFs** | RF-054, RF-055 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: Caso, Parte, Base, %, Valor, Status, Data | Lista de faturas de comissão |
| `CommissionStatusBadge` | PENDENTE/CALCULADA/DISTRIBUIDA | Status |
| `Badge` | `variant="secondary"` | Parte: Cedente / Cessionário |
| `Skeleton` | 8 linhas | Loading |

**Motion:** Fade-in tabela 200ms.

---

### T-066 — Financeiro — Extrato de Transações

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | GESTOR_FINANCEIRO |
| **RFs** | RF-056 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: Data, Tipo, Valor, Descrição, Celcoin ID | Extrato de transações |
| `Badge` | DEPOSITO/DISTRIBUICAO/ESTORNO/AJUSTE | Tipo da transação |
| `Select` | Default | Filtro por tipo, conta, período |
| `Button` | `variant="outline"` | Exportar CSV |
| `Skeleton` | 8 linhas | Loading |

---

### T-067 — Financeiro — Relatório Financeiro

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | GESTOR_FINANCEIRO |
| **RFs** | RF-057 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | KPIs do período: total distribuído, comissões, inadimplência |
| `ChartBar` | Tokens `--chart-1` a `--chart-3` | Volume de distribuições por mês |
| `DateRangePicker` | Default | Seleção de período |
| `Button` | `variant="outline"` | Exportar PDF |
| `Skeleton` | Por seção | Loading |

---

## 10. Supervisão IA (T-070 a T-075)

### T-070 — Supervisão IA — Dashboard

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-058, RF-059 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `KpiCard` | Tokens `--primary` | Decisões hoje, taxa de aprovação automática, pendentes revisão |
| `AiConfidenceBadge` | Custom — ver §2.3 | Score de confiança por decisão |
| `DataTable` | Colunas: Caso, Agente, Tipo, Confiança, Outcome, Data | Lista de decisões do agente |
| `AiDecisionOutcomeBadge` | APROVADO_AUTO/APROVADO_OP/REJEITADO_OP/PENDENTE | Outcome visual |
| `Button` | `variant="outline"` | Revisar decisão |
| `Skeleton` | Por seção | Loading |

**Acessibilidade:** `aria-live="polite"` no contador de pendentes (refresh 10s — DIV-001 do B04).

**Motion:** Fade-in 200ms. Auto-refresh 10s: pulse leve nos KPIs ao atualizar. Sem motion com `prefers-reduced-motion`.

**Regras:** Supabase Realtime com refresh configurado em 10s. Timestamp "Atualizado há Xs" obrigatório (DIV-001).

---

### T-071 — Supervisão IA — Detalhe de Decisão

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-059, RF-060 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Dados da decisão: agente, tipo, confiança, dados analisados |
| `AiConfidenceBadge` | Custom | Score visual |
| `AiDecisionOutcomeBadge` | Mapeamento §2.5 | Outcome atual |
| `Textarea` | Default | Feedback do operador |
| `Button` | `variant="default"` | Aprovar decisão |
| `Button` | `variant="destructive"` | Rejeitar decisão |
| `ScrollArea` | `h-[300px]` | JSON de `decision_data` formatado |
| `Skeleton` | Por seção | Loading |

**Motion:** Fade-in 200ms. Botões de ação: loading state durante submit.

---

### T-072 — Supervisão IA — Configuração de Agentes

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | MASTER |
| **RFs** | RF-061 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Por agente: nome, status, threshold de confiança |
| `Switch` | Default | Habilitar/desabilitar agente |
| `Slider` | Default, 0–1 step 0.01 | Threshold de confiança |
| `Button` | `variant="default"` | Salvar configuração |
| `Alert` | `variant="destructive"` | Erro ao salvar |
| `Skeleton` | Por card | Loading |

**Motion:** Fade-in stagger 80ms por card. Slider: animação nativa.

---

### T-073 — Supervisão IA — Histórico de Decisões

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-062 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: Caso, Agente, Tipo, Confiança, Outcome, Revisor, Data | Histórico completo |
| `AiDecisionOutcomeBadge` | Mapeamento §2.5 | Outcome |
| `Select` | Default | Filtros: agente, outcome, período |
| `Button` | `variant="outline"` | Exportar CSV |
| `Skeleton` | 8 linhas | Loading |

---

### T-074 — Supervisão IA — Alertas de Anomalia

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-063 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Alert` | `variant="destructive"` | Anomalia crítica detectada |
| `Alert` | `variant="default"` | Alerta de atenção |
| `ScrollArea` | `h-[500px]` | Lista de alertas |
| `Badge` | `variant="destructive"` | Prioridade crítica |
| `Skeleton` | `h-16 w-full` (x5) | Loading |

**Acessibilidade:** Alertas críticos com `aria-live="assertive"`.

---

### T-075 — Supervisão IA — Parâmetros Globais de IA

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | MASTER |
| **RFs** | RF-064 |

Estrutura similar a T-072 mas para parâmetros globais de todos os agentes. Form com `GlobalConfig` keys relacionadas a IA.

---

## 11. Usuários (T-080 a T-085)

### T-080 — Usuários — Lista de Operadores

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-065, RF-066 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: Nome, E-mail, Role, Status, Criado em, Ações | Lista de operadores |
| `RoleBadge` | Mapeamento §2.5 | Perfil do operador |
| `Badge` | `variant="outline"` verde/cinza | Ativo / Inativo |
| `Button` | `variant="default"` | Novo operador |
| `Button` | `variant="ghost"`, `size="icon"` | Editar / Desativar |
| `Input` | `type="search"` | Filtro por nome/e-mail |
| `Skeleton` | 8 linhas | Loading |

**Motion:** Fade-in tabela 200ms.

---

### T-081 — Usuários — Criar Operador

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | MASTER |
| **RFs** | RF-065 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Sheet` | `side="right"`, `size="md"` | Drawer de criação |
| `Input` | Default | Nome, e-mail |
| `Select` | Default | Role (ANALISTA/COORDENADOR/GESTOR_FINANCEIRO/MASTER) |
| `Button` | `variant="default"`, `w-full` | Criar operador |
| `Alert` | `variant="destructive"` | E-mail já cadastrado |

**Motion:** Slide-in 300ms.

**Regras:** Senha provisória enviada por e-mail. Master só pode criar outro Master.

---

### T-082 — Usuários — Editar Operador

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | MASTER |
| **RFs** | RF-066 |

Estrutura idêntica a T-081, com dados pré-preenchidos e Optimistic Lock.

---

### T-083 — Usuários — Desativar Operador

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | MASTER |
| **RFs** | RF-067 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `AlertDialog` | Default | Confirmação de desativação |
| `Textarea` | Default | Motivo (opcional) |
| `Button` | `variant="destructive"` | Confirmar desativação |

**Regras:** Desativação é soft — `is_active = false`. Operador perde acesso imediatamente.

---

### T-084 — Usuários — Lista de Cedentes

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-068 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DataTable` | Colunas: Nome, CPF, E-mail, Status, Casos, Ações | Lista de cedentes |
| `ExternalStatusBadge` | Mapeamento §2.5 | Status do cedente |
| `Input` | `type="search"` | Filtro por nome/CPF |
| `Button` | `variant="default"` | Novo cedente |
| `Skeleton` | 8 linhas | Loading |

---

### T-085 — Usuários — Criar/Editar Cedente

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (drawer) |
| **Role mínimo** | ANALISTA |
| **RFs** | RF-068 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Sheet` | `side="right"`, `size="md"` | Drawer |
| `Input` | Default, com CPF mask | Nome, CPF, e-mail, telefone, endereço |
| `Button` | `variant="default"`, `w-full` | Salvar |
| `Alert` | `variant="destructive"` | CPF/e-mail já cadastrado |

---

## 12. Relatórios (T-090 a T-093)

### T-090 — Relatórios — Hub de Relatórios

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-069, RF-070 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, hover com shadow | Card por tipo de relatório disponível |
| `Badge` | `variant="secondary"` | Categoria do relatório |
| `Button` | `variant="default"` | Gerar relatório |
| `Skeleton` | `h-[100px] w-full` (x4) | Loading |

**Motion:** Fade-in stagger 80ms por card.

---

### T-091 — Relatórios — Relatório Operacional

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-069 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `DateRangePicker` | Default | Período do relatório |
| `Select` | Default, multiple | Filtros: analista, status, cenário |
| `Card` | Default | KPIs do período |
| `ChartBar` | Tokens `--chart-1` a `--chart-3` | Volume por status |
| `DataTable` | Default | Casos do período |
| `Button` | `variant="outline"` | Exportar CSV / PDF |
| `Skeleton` | Por seção | Loading |

---

### T-092 — Relatórios — Relatório de Desempenho de Analistas

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-070 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | KPIs por analista: casos triados, qualificados, SLA médio |
| `ChartBar` | `--chart-1` a `--chart-3` | Comparativo entre analistas |
| `DataTable` | Default | Detalhamento por analista |
| `Skeleton` | Por seção | Loading |

---

### T-093 — Relatórios — Exportação

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` (modal) |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-071 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Dialog` | Default, `max-w-sm` | Modal de exportação |
| `Select` | Default | Formato: CSV, PDF |
| `DateRangePicker` | Default | Período |
| `Checkbox` | Default | Colunas a incluir |
| `Button` | `variant="default"`, `w-full` | Exportar |
| `Progress` | Default | Progresso de geração do arquivo |

---

## 13. Configurações (T-095 a T-100)

### T-095 — Configurações — Hub

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | COORDENADOR |
| **RFs** | RF-072 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Card por grupo de configuração |
| `Badge` | `variant="secondary"` | Categoria |
| `Button` | `variant="ghost"` | Acessar grupo |
| `Skeleton` | `h-[80px] w-full` (x5) | Loading |

---

### T-096 — Configurações — Parâmetros Financeiros

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | MASTER |
| **RFs** | RF-072, RF-073 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Form` + `FormField` | RHF + Zod | Form de configurações globais |
| `Input` | `type="number"`, % mask | Percentuais de comissão (4 cenários + Cessionário) |
| `Input` | `type="number"`, currency mask | Delta threshold, Distrato % |
| `Button` | `variant="default"`, `w-full` | Salvar configurações |
| `Alert` | `variant="default"` | Aviso de impacto: alteração não retroage a casos existentes |
| `AlertDialog` | Default | Confirmação antes de salvar |
| `Skeleton` | `h-10 w-full` (x5) | Loading |

**Acessibilidade:** `aria-describedby` em cada campo apontando para nota de impacto.

**Motion:** Fade-in form 200ms. AlertDialog fade + scale ao confirmar.

**Regras:** Alteração de config não altera snapshots de casos existentes (RN-111). Aviso obrigatório visível antes de salvar.

---

### T-097 — Configurações — SLAs e Prazos

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | MASTER |
| **RFs** | RF-074 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Form` + `FormField` | RHF + Zod | SLAs: triagem, negociação, formalização, reversão |
| `Input` | `type="number"` | Valores em horas ou dias |
| `Select` | Default | Unidade: horas / dias |
| `Button` | `variant="default"` | Salvar |
| `Alert` | `variant="default"` | Impacto em casos ativos |

---

### T-098 — Configurações — Integrações

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | MASTER |
| **RFs** | RF-075 |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default | Por integração: ZapSign, Celcoin, WhatsApp, SMS |
| `Badge` | Verde/Vermelho | Status da integração (conectado/erro) |
| `Button` | `variant="outline"` | Testar conexão |
| `Button` | `variant="ghost"` | Ver logs de webhook |
| `Skeleton` | `h-[80px] w-full` (x4) | Loading |

---

### T-099, T-100 — Páginas de Erro (404, 500)

| Atributo | Valor |
|---|---|
| **Layout** | `ErrorLayout` |
| **Role mínimo** | Público |
| **RFs** | — |

| Componente shadcn/ui | Variante / Config | Propósito |
|---|---|---|
| `Card` | Default, centralizado | Container de erro |
| `Button` | `variant="default"` | "Voltar ao início" |
| `Button` | `variant="outline"` | "Tentar novamente" (500) |

**Regras:** T-099 = 404 Not Found. T-100 = 500 / Erro interno.

---

## 14. Cross-References

| Documento | Relação com D09 |
|---|---|
| D01 — Regras de Negócio | Fonte das regras de domínio que definem estados, alçadas e SLAs dos componentes |
| D03 — Brand Theme Guide | Fonte dos tokens de design (`--primary`, `--destructive`, `--chart-*`, etc.) |
| D06 — Mapa de Telas | Fonte canônica dos IDs de tela, rotas e RFs |
| D07 — Wireframes | Referência visual de layout por tela |
| D08 — UX Writing | Fonte dos textos de labels, mensagens e estados vazios |
| D11 — Mobile | Consome estes contratos para adaptação React Native |
| D15 — Arquitetura de Pastas | Usa os módulos definidos aqui para estruturar `features/<módulo>/pages/` |
| D28 — Checklist de Qualidade | Valida conformidade com os contratos de componentes e estados |

---

## 15. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — 68 telas cobertas, 4 estados por tela, 3 layouts base, 5 componentes de domínio, StatusBadge mappings completos para 8 enums. |

---

## 16. Backlog de Pendências

| Item | Tipo | Tela / Módulo | Impacto | Justificativa | Dono | Status |
|---|---|---|---|---|---|---|
| Timestamp em Dashboard KPIs | [DECISÃO AUTÔNOMA] | T-010, T-060, T-070 | P1 | Exibido como texto secundário no canto superior do card; alternativa (modal de info) descartada por custo de interação. DIV-002 do B04 — aguarda validação produto. | Product | Aberto |
| Timestamp Supervisão IA | [DECISÃO AUTÔNOMA] | T-070 | P0 | DIV-001 do B04 — indicador de atualização implementado via `aria-live="polite"` + texto "Atualizado há Xs". Alternativa (sem indicador) descartada — viola expectativa de dados em tempo real. | Product | Aberto |
| `AiDecisionOutcomeBadge` enum | [DECISÃO AUTÔNOMA] | T-070, T-071, T-073 | P1 | Enum criado como extensão do mapeamento StatusBadge padrão seguindo convenção de cores semânticas do D03. Alternativa (ícones sem texto) descartada para a11y. | Frontend Lead | Resolvido |
| `CommissionStatusBadge` enum | [DECISÃO AUTÔNOMA] | T-065 | P1 | PENDENTE/CALCULADA/DISTRIBUIDA mapeados em amarelo/azul/verde seguindo semântica de estados financeiros. | Frontend Lead | Resolvido |
