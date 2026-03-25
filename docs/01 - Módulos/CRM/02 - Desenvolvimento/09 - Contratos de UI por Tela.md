# 09 - Contratos de UI por Tela

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Frontend, Design e QA |
| **Escopo** | Especificações de componentes shadcn/ui por tela: componentes, props obrigatórias, estados, comportamentos interativos e critérios de aceite |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 (America/Fortaleza) |
| **Dependências** | 06 - Mapa de Telas · 07 - Wireframes · 08 - UX Writing · 01.1–01.5 Regras de Negócio |

---

> **TL;DR**
>
> - **14 telas documentadas** (T-CRM-001, 010, 020, 022, 031, 040, 050, 051, 060, 061, 070, 080, 090, 110).
> - **Biblioteca de componentes:** shadcn/ui (Radix UI primitives + Tailwind CSS).
> - **Padrão:** cada tela lista componentes, props críticas, estados (loading / empty / error / success) e comportamentos interativos.
> - **Breakpoints:** desktop ≥1280px (primário), tablet 768–1279px (secundário). Mobile <768px: fora do escopo MVP.

---

## 1. Convenções Globais

### 1.1 Componentes Base Reutilizados em Todas as Telas

| **Componente shadcn/ui** | **Uso** |
|---|---|
| `Toaster` + `toast()` | Feedback de ações (sucesso, erro, informativo) — canto inferior direito |
| `Skeleton` | Estado de carregamento de qualquer bloco de dados |
| `Alert` | Banners persistentes de contexto (SLA, opt-out, janela WhatsApp) |
| `Badge` | Labels de status, SLA, opt-out, prioridade |
| `Tooltip` | Informações auxiliares em ícones e elementos truncados |
| `Separator` | Divisores visuais entre seções |
| `Breadcrumb` | Localização hierárquica no topo de cada tela interna |

### 1.2 Props Padrão de Formulários

- Todos os `Input` e `Textarea` com `aria-describedby` apontando para o elemento de erro inline.
- Todos os `Button` de submissão com `disabled` durante loading e com `aria-busy="true"`.
- `Form` (React Hook Form + Zod) em todos os formulários — sem formulários não validados.

### 1.3 Breakpoints

| **Breakpoint** | **Tailwind** | **Comportamento** |
|---|---|---|
| Desktop | `lg:` (≥1024px) | Layout completo, sidebar expandida |
| Tablet | `md:` (≥768px) | Sidebar colapsada (ícones), tabelas scrolláveis |
| Mobile | `sm:` (<768px) | Fora do escopo MVP — sem contrato definido |

---

## 2. T-CRM-001 — Login / Autenticação

### 2.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Card` + `CardContent` | Container central do formulário | `className="w-full max-w-md mx-auto"` |
| `Form` (RHF) | Wrapper do formulário com validação | `onSubmit`, `resolver` (Zod) |
| `FormField` + `Input` | Campo e-mail | `type="email"`, `autoComplete="email"`, `autoFocus` |
| `FormField` + `Input` | Campo senha | `type="password"`, `autoComplete="current-password"` |
| `Button` | Botão primário "Entrar" | `type="submit"`, `disabled={isSubmitting}` |
| `Alert` (variant destructive) | Erros de credencial / bloqueio / rede | `role="alert"` |

### 2.2 Estados

| **Estado** | **Componente ativo** | **Comportamento** |
|---|---|---|
| **Padrão** | Formulário limpo | `autoFocus` no campo e-mail |
| **Loading** | `Button` com spinner | Texto "Entrando...", `disabled=true`, campos desabilitados |
| **Erro de credencial** | `Alert` abaixo do formulário | Texto UX-CRM-008; campo senha limpo via `resetField("password")` |
| **Conta bloqueada** | `Alert` variant destructive | Texto UX-CRM-009; botão "Entrar" desabilitado |
| **Sessão expirada** | `Alert` variant default (azul) | Banner UX-CRM-011 exibido antes do formulário |

### 2.3 Comportamentos Interativos

- `Enter` no campo senha submete o formulário.
- Ícone de olho no campo senha alterna `type` entre `password` e `text` (aria label UX-CRM-013).
- Link "Esqueci minha senha" navega para T-CRM-002 via `<Link>` (Next.js App Router).

---

## 3. T-CRM-010 — Dashboard

### 3.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Card` | Cada bloco de KPI | `className="flex-1"` |
| `CardHeader` + `CardTitle` | Título do bloco | — |
| `CardContent` | Valor do KPI | Tipografia `text-3xl font-bold` |
| `Badge` | Status de SLA nos blocos | `variant` conforme nível (green/yellow/red) |
| `Skeleton` | Estado de carregamento dos KPIs | Mínimo 4 skeletons durante fetch |
| `Button` variant outline | Atalho "Novo caso" | `asChild` com `<Link>` |
| `Separator` | Entre blocos de Admin RS e Analista RS | — |

### 3.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | `Skeleton` em todos os cards de KPI (pulse animation) |
| **Analista RS** | Exibe: Meus casos hoje, Follow-ups pendentes, Alertas de SLA próprios |
| **Coordenador RS** | Exibe: visão da equipe, carga de Analistas, SLA geral |
| **Admin RS** | Exibe: tudo do Coordenador + bloco Financeiro (receita, breakeven) |
| **Empty** | UX-CRM-030 com atalho para criar caso ou registrar atividade |

### 3.3 Comportamentos Interativos

- Cards de KPI clicáveis navegam para a tela correspondente (Pipeline, Agenda, SLA Monitor).
- Bloco financeiro oculto para Coordenador RS e Analista RS via verificação de `role` no server component.

---

## 4. T-CRM-020 — Pipeline Kanban

### 4.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Tabs` | Toggle Kanban / Lista | `defaultValue="kanban"` |
| `TabsList` + `TabsTrigger` | Opções de visão | Labels UX-CRM-036 / UX-CRM-037 |
| `ScrollArea` | Container das colunas Kanban | `orientation="horizontal"` |
| `Card` (column) | Cada coluna do Kanban (1 por estado) | Largura fixa `min-w-[280px]` |
| `Card` (case card) | Card de cada caso | `draggable` (futuro), `onClick` navega para T-CRM-022 |
| `Badge` | SLA status no card | `variant` "default" / "secondary" / "destructive" |
| `Input` | Busca e filtro | `placeholder` UX-CRM-038 |
| `Button` variant ghost | Limpar filtros | Exibido apenas quando filtro ativo |
| `Skeleton` | Carregamento das colunas | 3 skeleton cards por coluna |

### 4.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton cards em todas as colunas |
| **Padrão** | 9 colunas (estados do Caso) — Cancelado omitido da visão padrão |
| **Filtro ativo** | Banner inline UX-CRM-039 com botão UX-CRM-040 |
| **Pipeline vazio (Analista RS)** | Empty state UX-CRM-041 centralizado na área de trabalho |
| **SLA em risco** | Badge amarelo UX-CRM-042 no card |
| **SLA vencido** | Badge vermelho UX-CRM-043 no card |

### 4.3 Comportamentos Interativos

- Click no card do caso abre T-CRM-022.
- Hover no badge de SLA exibe `Tooltip` com UX-CRM-045.
- Toggle de visão persiste em `localStorage` (`crm_pipeline_view`).
- Busca com debounce de 300ms.

---

## 5. T-CRM-022 — Detalhes do Caso

### 5.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Tabs` | Abas do caso (Resumo, Linha do Tempo, Dossiê, Negociação, Comunicação, Atividades) | `defaultValue="resumo"` |
| `TabsList` + `TabsTrigger` | Cada aba | Labels UX-CRM-060 a UX-CRM-065 |
| `Button` variant default | Avançar estado | `disabled={!canAdvance}` — UX-CRM-046/047 |
| `Button` variant outline | Retroceder estado (Coordenador/Admin) | `disabled={!canRetrocede}` |
| `Button` variant destructive | Cancelar caso | Abre `AlertDialog` de confirmação |
| `AlertDialog` | Confirmação de cancelamento | Título UX-CRM-051, motivo obrigatório |
| `Select` | Motivo de cancelamento | `required`, opções UX-CRM-053 a 057 |
| `Badge` | Estado atual do caso | Cor por estado |
| `DropdownMenu` | Menu "..." (redistribuir, retroceder) | `Coordenador RS` e `Admin RS` apenas |
| `Tooltip` | Botão de avanço desabilitado | Texto UX-CRM-048 |
| `Alert` | Banner opt-out na aba Comunicação | Quando contato tem opt-out |

### 5.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | `Skeleton` para header e abas |
| **Pode avançar** | Botão "Avançar" habilitado, verde |
| **Não pode avançar** | Botão desabilitado + `Tooltip` UX-CRM-048 + lista de pendências |
| **Caso cancelado** | Banner destructive, abas em somente leitura, botão de avanço oculto |
| **Caso concluído** | Banner success, abas em somente leitura |

### 5.3 Comportamentos Interativos

- Breadcrumb atualizado: "Pipeline → [RS-ID] → [Estado atual]".
- Modal de cancelamento exige seleção de motivo antes de habilitar botão de confirmar.
- Retrocesso abre `Dialog` com campo de justificativa obrigatório (mín. 20 caracteres).

---

## 6. T-CRM-031 — Negociação — Painel do Caso

### 6.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Card` | Seções: Match, Histórico de propostas, Comissão estimada | — |
| `ScrollArea` | Histórico de propostas (timeline) | `className="h-64"` |
| `Badge` | Status da proposta (ativa/expirada/aceita/recusada) | `variant` conforme status |
| `Alert` | Banner ≥6 rodadas | `variant="warning"` (customizado) |
| `Alert` | Banner proposta expirada | `variant="destructive"` |
| `Button` | Registrar proposta | Abre modal T-CRM-032 |
| `Button` | Registrar contraproposta | Abre modal T-CRM-033 |
| `Button` variant default | Aceitar proposta | Abre modal T-CRM-034 |
| `HoverCard` | Preview rápido do Cessionário | — |

### 6.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton nas 3 seções |
| **Sem proposta** | Empty state na seção de histórico |
| **Com proposta ativa** | Botão "Aceitar" habilitado, botão "Contraproposta" habilitado |
| **Proposta expirada** | Banner vermelho UX-CRM-072, botão "Aceitar" desabilitado |
| **Negociação em ≥6 rodadas** | Banner âmbar UX-CRM-071 |
| **Proposta aceita** | Timeline mostra evento de aceite, seção de comissão estimada exibida |

---

## 7. T-CRM-040 — Dossiê — Checklist de Documentos

### 7.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Table` | Lista de documentos do checklist | `className="w-full"` |
| `TableHeader` + `TableRow` + `TableCell` | Colunas: Documento, Status, Data de envio, Ações | — |
| `Badge` | Status por item (pendente, enviado, aprovado, rejeitado) | `variant` por status |
| `Button` variant outline | Upload de documento | `accept=".pdf,.jpg,.png"` via `Input type="file"` oculto |
| `Button` | Solicitar aprovação | `disabled={!allRequiredUploaded}`, tooltip UX-CRM-086 |
| `Alert` | Banner dossiê aprovado | `variant="success"` (customizado) |
| `Alert` | Alerta tabela atual vencida | `variant="warning"` |
| `Dialog` | Visualizador de documento (PDF/imagem) | — |
| `Progress` | Progresso do checklist | `value={approvedCount / totalRequired * 100}` |

### 7.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton da tabela |
| **Pendente** | Todos os itens com badge "Aguardando envio" |
| **Em andamento** | Mix de pendentes, enviados e aprovados |
| **Aprovação solicitada** | Botão "Solicitar aprovação" desabilitado, tooltip indica aguardando |
| **Dossiê aprovado** | Banner verde UX-CRM-087, todos os itens com badge "Aprovado" |
| **Item rejeitado** | Badge vermelho, motivo exibido inline, botão de re-upload habilitado |

---

## 8. T-CRM-050 — Contatos — Lista

### 8.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `DataTable` (custom com TanStack Table) | Listagem de contatos | Colunas: Nome, Papel, Status, Casos ativos, Ações |
| `Input` | Busca por nome, e-mail ou CPF | `placeholder="Buscar contato..."`, debounce 300ms |
| `Select` | Filtro de papel (Cedente/Cessionário/Parceiro) | `onValueChange` atualiza query |
| `Select` | Filtro de status (Ativo/Arquivado/Opt-out) | — |
| `Badge` | Status do contato | `variant` conforme status |
| `Badge` variant destructive | Opt-out ativo | Label UX-CRM-100 |
| `Badge` | Investidor recorrente | Label "Investidor recorrente" |
| `Button` | Novo contato | Navega para T-CRM-052 |
| `DropdownMenu` | Ações por linha (Ver perfil, Editar, Mesclar) | — |

### 8.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton de tabela (10 linhas) |
| **Padrão** | Contatos ativos, sem arquivados por padrão |
| **Filtro aplicado** | Badge de filtro ativo na barra de busca |
| **Empty** | UX-CRM-105 centralizado |
| **Opt-out** | Badge vermelho na linha, ações de comunicação ocultas |

---

## 9. T-CRM-051 — Contatos — Perfil do Contato

### 9.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Avatar` + `AvatarFallback` | Foto/iniciais do contato | Iniciais de 2 letras |
| `Card` | Seção de dados pessoais | Mascaramento de CPF (RN-012) |
| `Card` | Seção de casos vinculados | Lista com links para T-CRM-022 |
| `Card` | Seção de histórico de atividades | Timeline com `ScrollArea` |
| `Alert` variant destructive | Banner opt-out | UX-CRM-101 |
| `Badge` | Status do contato | — |
| `Badge` | "Mesclado" se registro secundário | Somente leitura |
| `Button` | Editar contato | Navega para T-CRM-052 |
| `Button` | Registrar atividade | Abre modal T-CRM-061 |
| `Button` variant outline | Mesclar contatos | Visível apenas se duplicata sinalizada (Coordenador RS) |
| `Button` | Enviar WhatsApp | `disabled={hasOptOut}`, tooltip UX-CRM-101 |
| `Button` | Enviar e-mail | `disabled={hasOptOut}` |

### 9.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton de perfil (avatar + 3 cards) |
| **Padrão** | Todos os dados visíveis conforme papel do usuário |
| **Opt-out ativo** | Banner vermelho no topo, botões de comunicação desabilitados |
| **Mesclado** | Banner informativo, campos em somente leitura |
| **Sem casos vinculados** | Seção de casos com empty state "Nenhum caso vinculado" |

---

## 10. T-CRM-060 — Atividades — Agenda de Follow-ups

### 10.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Tabs` | Toggle visão diária / semanal | `defaultValue="today"` |
| `TabsTrigger` | Hoje / Esta semana | Labels UX-CRM-108/109 |
| `Card` | Cada follow-up na agenda | `onClick` navega para o caso vinculado |
| `Badge` variant destructive | Follow-up vencido | Label UX-CRM-110 |
| `Badge` variant secondary | Alta prioridade | Label UX-CRM-111 |
| `Button` | Agendar follow-up | Abre modal T-CRM-061 |
| `ScrollArea` | Container dos cards | `className="h-full"` |

### 10.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton de 5 cards |
| **Vazio** | Empty state UX-CRM-112 com atalho para agendar |
| **Com vencidos** | Cards vencidos no topo, badge vermelho, ordenados por data de vencimento |
| **Visão semanal** | Agrupamento por dia da semana |

---

## 11. T-CRM-061 — Modal — Registrar Atividade / Follow-up

### 11.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Dialog` + `DialogContent` | Container do modal | `onOpenChange` fecha corretamente |
| `DialogHeader` + `DialogTitle` | Título UX-CRM-113 | — |
| `Tabs` | Toggle modo: Atividade passada / Agendar follow-up | `defaultValue="activity"` |
| `Form` (RHF + Zod) | Validação do formulário | Schema Zod diferente por modo |
| `Select` | Tipo de atividade | Opções UX-CRM-115, `required` |
| `DateTimePicker` (custom) | Data/hora da atividade ou do follow-up | Restrição: passado (atividade) / futuro (follow-up) |
| `Textarea` | Resumo da atividade | `minLength={20}`, label UX-CRM-116 |
| `Select` | Resultado (para atividade passada) | Opções: Positivo / Neutro / Sem resposta / Negativo |
| `Combobox` | Contato-alvo (para follow-up) | Busca por nome |
| `Button` type="submit" | Salvar | `disabled={isSubmitting}` |

### 11.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Spinner no botão de salvar |
| **Erro de data retroativa** | Inline error UX-CRM-118 abaixo do campo de data |
| **Sucesso (atividade)** | Toast UX-CRM-119, modal fecha |
| **Sucesso (follow-up)** | Toast UX-CRM-120, modal fecha |
| **Erro de validação** | Inline errors em campos obrigatórios |

---

## 12. T-CRM-070 — Comunicação — Caixa de Mensagens

### 12.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `ScrollArea` | Thread de mensagens | `ref` para scroll automático na última mensagem |
| `Card` (mensagem enviada) | Mensagem do CRM | Alinhamento direita, fundo `--primary` |
| `Card` (mensagem recebida) | Mensagem do contato | Alinhamento esquerda, fundo `--muted` |
| `Badge` | Status de entrega (UX-CRM-122 a 125) | Ícone + texto inline na mensagem |
| `Badge` variant secondary | Mensagem registrada manualmente | Label UX-CRM-126 |
| `Alert` | Banner janela 24h encerrada | UX-CRM-127 + botão UX-CRM-128 |
| `Alert` variant destructive | Banner opt-out | UX-CRM-129 |
| `Alert` variant default | Sem WhatsApp cadastrado | UX-CRM-130 |
| `Textarea` | Campo de mensagem | `placeholder` UX-CRM-131, `disabled` se bloqueado |
| `Button` | Enviar mensagem | `disabled={isBlocked || !message.trim()}` |
| `Button` variant outline | Selecionar template | Abre modal T-CRM-071 |

### 12.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton da thread |
| **Padrão** | Thread + input habilitado |
| **Janela 24h encerrada** | Input desabilitado, banner âmbar, botão de template |
| **Opt-out ativo** | Input desabilitado, banner vermelho, template bloqueado |
| **Sem WhatsApp** | Input desabilitado, banner âmbar |
| **Mensagem enviando** | `Button` com spinner, `disabled` |

---

## 13. T-CRM-080 — SLA Monitor — Dashboard

### 13.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Card` | Bloco "Visão Geral de SLA" | 3 contadores: dentro do prazo / em risco / vencido |
| `Card` | Ranking de Analistas RS | Lista ordenada por taxa de cumprimento |
| `Card` | Histograma | Chart via `recharts` (BarChart) |
| `Card` | Previsão de fechamentos | Lista com data estimada por caso |
| `DataTable` | Lista de alertas ativos | Colunas: Caso, Analista, Estado, % SLA, Ação |
| `Badge` variant destructive | Alerta vencido | — |
| `Badge` variant secondary | Alerta em risco | — |
| `Button` variant outline | Reconhecer alerta | `onClick` chama API PATCH /alerts/:id/acknowledge |
| `Skeleton` | Carregamento de todos os blocos | — |

### 13.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton em todos os 4 cards |
| **Sem alertas** | Tabela com empty state "Nenhum alerta ativo no momento" |
| **Com alertas** | Ordenação por prioridade (vencido primeiro) |
| **Alerta reconhecido** | Toast UX-CRM-142, linha removida da tabela |

---

## 14. T-CRM-090 — Equipe — Lista de Usuários

### 14.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `DataTable` | Listagem de usuários | Colunas: Nome, Papel, Status, Casos ativos, Ações |
| `Tabs` | Filtro por status | Ativo / Suspenso / Desligado |
| `Badge` | Status do usuário | Verde (ativo), âmbar (suspenso), cinza (desligado) |
| `Avatar` | Foto/iniciais do usuário | — |
| `Button` | Convidar membro | Abre modal T-CRM-093 modo "Convidar" |
| `DropdownMenu` | Ações por linha (Ver perfil, Suspender, Desligar, Alterar papel) | Visível apenas para Admin RS |
| `AlertDialog` | Confirmação de suspensão/desligamento | Destrutivo — `variant="destructive"` |

### 14.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton de 5 linhas |
| **Padrão** | Aba "Ativo" selecionada por default |
| **Desligar com casos ativos** | AlertDialog com aviso UX-CRM-153, botão de confirmar desabilitado até redistribuição |
| **Convite pendente** | Linha com badge "Convite pendente" e botão "Reenviar" |

---

## 15. T-CRM-110 — Configurações — Parâmetros do Sistema

### 15.1 Componentes

| **Componente** | **Uso** | **Props obrigatórias** |
|---|---|---|
| `Card` | Cada grupo de parâmetros | — |
| `Label` + `Input` / `Switch` / `Select` | Campo de parâmetro | `disabled={requiresDoubleApproval && !isApproved}` |
| `Badge` | "Requer aprovação dupla" | Label UX-CRM-166, ícone de cadeado |
| `Badge` variant secondary | "Aprovação pendente" | Label UX-CRM-167 |
| `Tooltip` | Explicação da aprovação dupla | Texto UX-CRM-168 |
| `Button` | Salvar parâmetro | `disabled={!isDirty}` |
| `Button` variant outline | Ver histórico de alterações | Link para T-CRM-112 com filtro |
| `AlertDialog` | Confirmação de alteração de parâmetro crítico | Texto UX-CRM-187 |

### 15.2 Estados

| **Estado** | **Comportamento** |
|---|---|
| **Loading** | Skeleton dos grupos de parâmetros |
| **Padrão** | Campos editáveis para Admin RS |
| **Parâmetro crítico** | Campo bloqueado até ter 2 aprovações |
| **Aprovação pendente** | Badge âmbar, campo em somente leitura |
| **Salvo com sucesso** | Toast UX-CRM-176 |

---

## 16. Tabela de Cobertura

| **Tela** | **ID** | **Componentes** | **Estados documentados** |
|---|---|---|---|
| Login | T-CRM-001 | 6 | 5 |
| Dashboard | T-CRM-010 | 7 | 5 |
| Pipeline Kanban | T-CRM-020 | 9 | 6 |
| Detalhes do Caso | T-CRM-022 | 10 | 5 |
| Negociação Painel | T-CRM-031 | 9 | 6 |
| Dossiê Checklist | T-CRM-040 | 9 | 6 |
| Contatos Lista | T-CRM-050 | 9 | 5 |
| Contatos Perfil | T-CRM-051 | 10 | 5 |
| Agenda Follow-ups | T-CRM-060 | 7 | 4 |
| Modal Atividade | T-CRM-061 | 10 | 5 |
| Comunicação | T-CRM-070 | 11 | 6 |
| SLA Monitor | T-CRM-080 | 8 | 4 |
| Equipe Lista | T-CRM-090 | 8 | 4 |
| Configurações | T-CRM-110 | 8 | 5 |

---

## 17. Controle de Versão

| **Versão** | **Data** | **Responsável** | **Alteração** |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 14 telas documentadas |
