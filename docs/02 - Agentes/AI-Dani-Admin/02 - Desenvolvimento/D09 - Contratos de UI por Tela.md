# Contratos de UI por Tela — AI-Dani-Admin

## Especificações de Componentes por Tela

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Frontend |
| Escopo | Contratos de UI detalhados por tela — componentes, tokens, estados, props e comportamento |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D03 (Brand Guide — tokens), D04 (Motion Spec), D06 (Telas), D07 (Wireframes), D08 (UX Writing) |

---

> **📌 TL;DR**
>
> - Contratos de UI para 7 telas + 2 componentes globais do módulo AI-Dani-Admin.
> - Componentes base: shadcn/ui + Lucide Icons. Animações: Framer Motion (D04).
> - Tokens de cor, spacing e radius do D03 (Brand Guide).
> - Cada tela define: estrutura de layout, componentes por região, estados e props críticas.
> - Nenhum componente customizado sem justificativa — shadcn/ui cobre 90% dos casos.

---

## 1. Convenções Globais

### 1.1 Componentes Base

| Componente | Origem | Uso |
|---|---|---|
| `Table`, `TableHeader`, `TableRow`, `TableCell` | shadcn/ui | Listagem de interações, log de auditoria |
| `Card`, `CardHeader`, `CardContent` | shadcn/ui | Cards de métricas, cards de detalhes |
| `Badge` | shadcn/ui | Status de interação e agente |
| `Button` | shadcn/ui | Todas as ações (primary, secondary, ghost, destructive) |
| `Dialog` / `AlertDialog` | shadcn/ui | Modais de confirmação |
| `Sheet` | shadcn/ui | Drawers (configurações, detalhes) |
| `Toast` / `Sonner` | shadcn/ui | Toasts de confirmação e erro |
| `Slider` | shadcn/ui | Slider de threshold |
| `Input` | shadcn/ui | Campos de texto (threshold, rate limit) |
| `Select` | shadcn/ui | Dropdowns de filtros |
| `Tabs` | shadcn/ui | Seleção de período (dia/semana/mês) |
| `Separator` | shadcn/ui | Separador visual no chat de takeover |
| `Skeleton` | shadcn/ui + custom shimmer | Loading states |
| `Tooltip` | shadcn/ui | Tooltips em botões desabilitados, timestamps |
| `Alert` / Custom Banner | Custom (baseia-se em shadcn/ui Alert) | COMP-001 Banner de alerta crítico |
| Ícones | Lucide Icons | Toda iconografia |

### 1.2 Tokens de Referência

Todos os tokens de cor, radius e spacing são do D03 - Brand Guide. Tokens de motion são do D04 - Motion Spec.

### 1.3 Regras de Layout Base

- Toda tela herda o layout do painel Admin: `AppSidebar` (256px) + área de conteúdo principal.
- Padding do conteúdo principal: `p-6` (24px em todos os lados).
- Max-width do conteúdo: `max-w-7xl` (1280px) centralizado.
- Header de cada tela: `flex items-center justify-between mb-6`.

---

## 2. T-001 — Painel Supervisão IA (Lista de Interações)

### 2.1 Estrutura de Layout

```
<main className="p-6 max-w-7xl mx-auto">
  <PageHeader />         {/* H1 + badge de sinalizadas + botão filtros */}
  <FilterBar />          {/* chips de filtros ativos + limpar */}
  <InteractionTable />   {/* tabela de interações */}
  <Pagination />         {/* paginação */}
</main>
```

### 2.2 Componentes

**PageHeader**
```typescript
interface PageHeaderProps {
  flaggedCount: number  // número de interações sinalizadas
}
// H1: text-2xl font-semibold
// Badge (flaggedCount > 0): variant="warning", text: "[X] aguardando revisão"
// Botão filtros: Button variant="outline" + Filter icon
```

**FilterBar** (condicional — exibir apenas quando há filtros ativos)
```typescript
interface FilterBarProps {
  activeFilters: FilterChip[]
  onRemoveFilter: (filterId: string) => void
  onClearAll: () => void
}
// Chips: Badge com botão × interno, variant="secondary"
// Botão "Limpar filtros": Button variant="ghost" size="sm"
// Animação dos chips: chipIn variant do D04
```

**InteractionTable**
```typescript
interface InteractionRow {
  id: string
  userIdAnon: string       // exibido como "ID-****XX"
  agentName: string        // "Dani-Cessionário" | "Dani-Cedente"
  confidenceScore: number | null  // 0-100, null se em takeover
  latencySeconds: number | null
  timestamp: Date
  status: 'RESPONDIDA_PELA_IA' | 'SINALIZADA_PARA_REVISAO' | 'EM_TAKEOVER' | 'ENCERRADA'
}

// Colunas: Usuário | Agente | Confiança | Latência | Data/Hora | Status
// Clique na linha: navega para T-002
// Loading state: <Skeleton /> 8 linhas, altura 56px
```

**Row styles por status:**
```typescript
// RESPONDIDA_PELA_IA: bg padrão, hover bg-accent
// SINALIZADA_PARA_REVISAO: bg-[--color-warning-subtle], border-l-2 border-[--color-warning]
// EM_TAKEOVER: bg-[--color-takeover-subtle], border-l-2 border-[--color-takeover]
// ENCERRADA: bg-[--muted/30], text-[--muted-foreground]
```

**BadgeStatus (componente interno)**
```typescript
const statusConfig = {
  RESPONDIDA_PELA_IA: { label: "Respondida pela IA", variant: "success" },
  SINALIZADA_PARA_REVISAO: { label: "Aguardando revisão", variant: "warning" },
  EM_TAKEOVER: { label: "Em atendimento humano", variant: "takeover" },
  ENCERRADA: { label: "Encerrada", variant: "secondary" }
}
```

**Confidence Display:**
```typescript
// >= threshold: "[X]%" em foreground padrão
// < threshold: "[X]% ⚠" com TriangleAlert icon 12px, cor warning
// null: "—" (traço)
```

**Latency Display:**
```typescript
// <= 2s: "[X.X]s" foreground padrão
// > 2s e <= SLA: "[X.X]s" cor warning
// > SLA: "[X.X]s ⚠" com TriangleAlert, cor warning
// null: "—"
```

**Empty State:**
```typescript
// Ícone: MessageSquareOff Lucide, 48px, cor muted-foreground
// Título: "Nenhuma interação registrada no período selecionado."
// Subtítulo: "Tente ajustar o período ou os filtros aplicados."
// Container: flex flex-col items-center gap-2 py-16
```

---

## 3. T-002 — Detalhe de Interação

### 3.1 Estrutura de Layout

```
<main className="p-6 max-w-7xl mx-auto">
  <DetailHeader />      {/* botão voltar + ID + badge status */}
  <div className="grid grid-cols-3 gap-6">
    <ChatHistory className="col-span-2" />   {/* histórico do chat */}
    <InteractionSidebar className="col-span-1" />  {/* métricas + ação */}
  </div>
</main>
```

### 3.2 Componentes

**DetailHeader**
```typescript
interface DetailHeaderProps {
  interactionId: string
  status: InteractionStatus
}
// Botão voltar: Button variant="ghost" size="sm" + ArrowLeft icon
// ID: text-lg font-semibold
// Badge: BadgeStatus component
```

**ChatHistory**
```typescript
// Container: Card com overflow-y-auto max-h-[600px]
// Cada mensagem do agente: avatar IA + nome agente + texto
// Cada mensagem do usuário: alinhada à direita + avatar placeholder
// Separador de takeover: Separator com texto "Atendimento humano"
//   - texto: text-xs text-muted-foreground
//   - label: "Atendimento humano"
// Separador de retorno: "Retorno ao atendimento automatizado"
```

**InteractionSidebar**
```typescript
// 4 Cards empilhados:
// 1. Confiança: valor + ícone de aviso se abaixo do threshold
// 2. Latência: valor em segundos
// 3. Agente: nome do agente
// 4. Data/Hora: timestamp formatado

// CTA Card (sticky ao bottom da sidebar):
interface ActionCardProps {
  status: InteractionStatus
  onTakeover: () => void
  onEndTakeover: () => void
}
// status != ENCERRADA e != EM_TAKEOVER:
//   Button primary "Assumir conversa"
// status == EM_TAKEOVER:
//   Button destructive outline "Encerrar atendimento humano"
// status == ENCERRADA:
//   Button disabled "Assumir conversa" + Tooltip "Conversa encerrada"
```

---

## 4. T-003 — Chat em Takeover (Admin View)

### 4.1 Estrutura de Layout

```
<main className="flex flex-col h-[calc(100vh-64px)]">
  <TakeoverHeader />     {/* banner de status + encerrar */}
  <ChatArea className="flex-1 overflow-y-auto" />   {/* histórico + mensagens */}
  <MessageInput />       {/* campo de digitação */}
</main>
```

### 4.2 Componentes

**TakeoverHeader**
```typescript
// Alert com variant="info" (azul)
// Ícone: UserRound Lucide
// Texto: "Atendimento humano ativo — Usuário: [ID anonimizado]"
// Sub-texto: "Agente pausado — você está respondendo."
// Botão encerrar: Button variant="outline" size="sm" "Encerrar atendimento humano"
```

**ChatArea**
```typescript
// Idêntico ao ChatHistory de T-002
// Mensagens do Admin: avatar de pessoa + "Equipe Repasse Seguro" + texto
// Atualização em tempo real via Supabase Realtime subscription
```

**MessageInput**
```typescript
interface MessageInputProps {
  onSend: (message: string) => Promise<void>
  isLoading: boolean
}
// Textarea com resize: none, minHeight 52px
// Enter para enviar (Shift+Enter para nova linha)
// Botão enviar: Button primary + Send icon
// Loading state: botão com spinner interno
// Erro de envio: texto inline abaixo do campo + botão "Tentar novamente"
```

---

## 5. T-004 — Dashboard de Métricas

### 5.1 Estrutura de Layout

```
<main className="p-6 max-w-7xl mx-auto">
  <DashboardHeader />    {/* H1 + seletores */}
  <MetricsGrid />        {/* 5 cards de métricas */}
  <VolumeChart />        {/* gráfico de volume */}
  <TopQuestions />       {/* top 10 perguntas */}
</main>
```

### 5.2 Componentes

**DashboardHeader**
```typescript
// Tabs (shadcn/ui): "Dia" | "Semana" | "Mês"
// Select (shadcn/ui): "Todos os agentes" | "Dani-Cessionário" | "Dani-Cedente"
```

**MetricCard**
```typescript
interface MetricCardProps {
  title: string
  value: string | null   // null = sem dados
  trend?: string         // ex: "+12% vs semana anterior"
  icon: LucideIcon
  isLoading?: boolean
}
// isLoading: Skeleton do card inteiro
// value == null: "Dados insuficientes para o período selecionado" + Info icon
//               text-sm text-muted-foreground + ℹ️ icon
// Nunca: value="0" quando null. Nunca: value="0%" quando null.
// value != null: text-2xl font-semibold
// trend: text-sm text-muted-foreground
// Animação de atualização: flash de opacity (D04 — seção 3.11)
```

**VolumeChart**
```typescript
// Gráfico de barras usando Recharts (padrão shadcn/ui charts)
// 2 séries: Dani-Cessionário (--chart-1) + Dani-Cedente (--chart-2)
// Skeleton: retângulo 100% × 200px durante loading
// Sem dados: EmptyState interno ao card do gráfico
```

**TopQuestions**
```typescript
interface TopQuestion {
  rank: number
  question: string
  count: number
}
// Lista ordenada, max 10 itens
// Rank: text-sm font-mono text-muted-foreground
// Pergunta: text-sm
// Count: Badge variant="secondary" ex: "287x"
```

---

## 6. T-005 — Configurações de Supervisão

### 6.1 Estrutura de Layout

```
<main className="p-6 max-w-2xl">
  <PageHeader />           {/* H1 */}
  <ThresholdCard />        {/* configuração de threshold */}
  <RateLimitCard />        {/* configuração de rate limit */}
  <AuditHistoryTable />    {/* histórico de alterações */}
</main>
```

### 6.2 Componentes

**ThresholdCard**
```typescript
interface ThresholdCardProps {
  currentValue: number      // 50-95
  onSave: (value: number) => Promise<void>
  isLoading: boolean
  error: string | null
}

// Card com:
// - Label: "Nível de supervisão (%)"
// - Descrição: "Interações com confiança abaixo deste valor serão sinalizadas para revisão."
// - Slider (shadcn/ui): min=50, max=95, step=1
// - Input numérico sincronizado com o slider
// - Range display: "50%" | "95%"
// - Botão: "Salvar configuração" (primary, full-width)
// - Estado loading: botão com spinner, input desabilitado
// - Erro: Alert variant="destructive" abaixo do campo
//   Texto: "O nível de supervisão precisa estar entre 50% e 95%..."
//   Shake animation: D04 seção 6.2
```

**AuditHistoryTable**
```typescript
interface AuditEntry {
  timestamp: Date
  adminId: string
  previousValue: number
  newValue: number
}
// Table com colunas: Data | Admin | Alteração
// "Alteração": "Threshold: [anterior]% → [novo]%"
// Max 10 linhas, paginação se necessário
```

---

## 7. T-006 — Log de Auditoria

### 7.1 Componentes

**AuditLogTable**
```typescript
interface AuditEvent {
  timestamp: Date
  adminId: string
  eventType: 'ACESSO' | 'TAKEOVER_INICIADO' | 'TAKEOVER_ENCERRADO' | 'THRESHOLD_ALTERADO' | 'AGENTE_REATIVADO' | 'ALERTA_DISPARADO'
  details: Record<string, unknown>
}
// Ícones por tipo de evento:
// ACESSO: Eye
// TAKEOVER_INICIADO: UserRound
// TAKEOVER_ENCERRADO: UserRoundX
// THRESHOLD_ALTERADO: SlidersHorizontal
// AGENTE_REATIVADO: RotateCcw
// ALERTA_DISPARADO: BellRing
```

---

## 8. T-007 — Checklist de Prontidão para Lançamento

### 8.1 Componentes

**ChecklistGroup**
```typescript
interface ChecklistItem {
  id: string
  label: string
  status: 'PENDING' | 'APPROVED' | 'FAILED'
  verifiedBy?: string
  verifiedAt?: Date
}

interface ChecklistGroupProps {
  title: string
  items: ChecklistItem[]
  onUpdateItem: (id: string, status: ChecklistItem['status']) => void
}

// Item styles:
// PENDING: ○ círculo vazio + texto text-foreground
// APPROVED: ✅ CheckCircle2 Lucide (verde) + texto text-foreground
// FAILED: ❌ XCircle Lucide (destrutivo) + texto text-destructive
```

**LaunchButton**
```typescript
interface LaunchButtonProps {
  isReady: boolean   // true somente quando 100% dos itens APPROVED
  agentName: string
  onLaunch: () => void
}
// isReady=false: Button disabled, variant="secondary"
// isReady=true: Button primary, "Autorizar lançamento"
// Clique: AlertDialog de confirmação (modal 11.4 do D08)
```

---

## 9. COMP-001 — Banner de Alerta Crítico

### 9.1 Componente

```typescript
interface CriticalAlertBannerProps {
  alert: {
    type: 'DESLIGAMENTO_AUTOMATICO' | 'LATENCIA_ALTA' | 'TAXA_ERRO_ELEVADA' | 'CSAT_DEGRADADO' | 'TAXA_RECUSA_ALTA' | 'CONSUMO_PROCESSAMENTO'
    title: string
    description: string
    metricValue?: string    // ex: "32%", "5 minutos"
    ctaLabel: string
    onCta: () => void
  }
  onDismiss?: () => void    // null = não pode ser fechado sem ação
}

// Position: sticky top-0 z-50 (dentro da área de conteúdo, abaixo do topbar)
// Animação: instantAlert variant do D04 (100ms)
// prefers-reduced-motion: sem animação, aparece diretamente
// Border-left: 4px --destructive
// Background: --destructive/10 (10% de opacidade)
// Pulse animation para borda: desabilitado com prefers-reduced-motion
```

---

## 10. COMP-002 — Separador Visual de Takeover

### 10.1 Componente

```typescript
interface TakeoverSeparatorProps {
  type: 'START' | 'END'
}

// type="START":
//   <div className="flex items-center gap-2 my-4">
//     <Separator />
//     <span className="text-xs text-muted-foreground whitespace-nowrap">
//       Atendimento humano
//     </span>
//     <Separator />
//   </div>
//   Animação: fadeIn 250ms (D04 seção 3.7)

// type="END":
//   Mesmo padrão com texto "Retorno ao atendimento automatizado"
```

---

## 11. Modais de Confirmação

### 11.1 Modal de Confirmação de Takeover

```typescript
// AlertDialog (shadcn/ui)
// Título: "Assumir conversa"
// Descrição: (copy do D08 seção 11.1)
// Campo de motivo: Select (shadcn/ui) — opcional
// CTA: AlertDialogAction "Assumir conversa →" (primary)
// Cancel: AlertDialogCancel "Cancelar"
// Animação: scaleIn 150ms (D04 — rápido por ser situação urgente)
```

### 11.2 Modal de Encerramento de Takeover

```typescript
// AlertDialog
// Título: "Encerrar atendimento humano"
// Descrição: (copy do D08 seção 11.2)
// CTA: AlertDialogAction "Encerrar atendimento" (outline)
// Cancel: "Cancelar"
```

### 11.3 Modal de Reativação de Agente

```typescript
// AlertDialog com ícone de aviso
// Título: "Reativar agente"
// Descrição: (copy do D08 seção 11.3) com aviso de verificação prévia
// CTA: AlertDialogAction "Reativar agente" (primary)
// Cancel: "Cancelar"
```

---

## 12. Regras de Acessibilidade nos Componentes

| Componente | Requisito de acessibilidade |
|---|---|
| Tabela de interações | `<table>` semântica com `<thead>`, `<th scope="col">` e `<td>` |
| Badges de status | `aria-label="Status: [texto do status]"` |
| Botão "Assumir conversa" desabilitado | `aria-disabled="true"` + `title="Conversa encerrada"` |
| Banner de alerta crítico | `role="alert"` para anúncio automático por leitor de tela |
| Campo de threshold | `aria-describedby` apontando para o elemento de erro |
| Slider de threshold | `aria-label="Nível de supervisão"`, `aria-valuemin="50"`, `aria-valuemax="95"`, `aria-valuenow="[value]"` |
| Modal de confirmação | Foco automático no botão de cancelar ao abrir (ação segura) |
| Toast de notificação | `role="status"` para mensagens não urgentes, `role="alert"` para erros |

---

## 13. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Contratos de UI para 7 telas + 2 componentes globais + 3 modais. |
