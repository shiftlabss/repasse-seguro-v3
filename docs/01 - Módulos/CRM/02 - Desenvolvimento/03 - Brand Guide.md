# Brand Guide — Módulo CRM

## Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Designers e frontend engineers |
| **Escopo** | Brand Guide do módulo CRM — interface interna profissional |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 2026-03-23 (America/Fortaleza) |
| **Fonte primária** | ShiftLabs Stacks v7.0 · Regras de Negócio CRM v1.0 |
| **Caráter** | **Normativo.** Desvios exigem ADR aprovado. |

---

> **TL;DR**
>
> - **Filosofia visual:** Interface profissional de alta densidade. Ferramenta de trabalho — não produto consumer. Clareza e eficiência acima de decoração.
> - **Paleta:** Fundo neutro (cinza-50), azul-petróleo como cor de ação principal, paleta de status obrigatória (verde/âmbar/vermelho) para SLA e estados.
> - **Tipografia:** Inter para toda a interface. Monospace (JetBrains Mono) exclusivamente para IDs de Caso (RS-XXXX) e valores de código.
> - **Componentes:** shadcn/ui com customizações específicas do CRM. Alta densidade de informação — padding reduzido vs. produtos consumer.
> - **Ícones:** Lucide React. Tamanho padrão 16px em contexto de tabela/lista, 20px em ações primárias.
> - **Modo escuro:** Não suportado na v1.0. Light mode apenas.
> - **Acessibilidade:** Contraste mínimo 4.5:1 para todos os textos e indicadores de status (RN-199).

---

## 1. Filosofia de Design

O CRM é uma **ferramenta profissional interna** usada por Analistas RS, Coordenadores RS e Admins RS para gerenciar casos de cessão imobiliária. Os princípios que governam cada decisão visual:

| Princípio | Descrição | Aplicação |
|---|---|---|
| **Densidade informacional** | Mais informação por tela, menos cliques | Tabelas compactas, cards densos, abas em vez de páginas separadas |
| **Hierarquia de urgência** | SLA e alertas devem ser imediatamente visíveis | Cores de status nunca decorativas — carregam significado semântico |
| **Sobriedade funcional** | Animações e cores a serviço da tarefa | Sem gradientes decorativos, sem ilustrações de marketing |
| **Consistência preditiva** | Cada padrão comporta-se sempre igual | Botão de avanço de estado sempre no mesmo lugar, mesmo estilo |
| **Acessibilidade obrigatória** | Contraste mínimo 4.5:1, rótulos textuais em indicadores visuais | Indicadores de SLA têm texto + cor + ícone (não apenas cor) |

---

## 2. Paleta de Cores

### 2.1 Tokens de cor base

Todos os tokens usam variáveis CSS via `@theme` do Tailwind CSS 4.

```css
@theme {
  /* ── Neutros (base da interface) ── */
  --color-neutral-50:  #F8FAFC;   /* fundo da página */
  --color-neutral-100: #F1F5F9;   /* fundo de cards, painéis laterais */
  --color-neutral-200: #E2E8F0;   /* bordas, divisores */
  --color-neutral-300: #CBD5E1;   /* bordas de input desabilitado */
  --color-neutral-400: #94A3B8;   /* placeholder, ícones secundários */
  --color-neutral-500: #64748B;   /* texto auxiliar, labels secundários */
  --color-neutral-600: #475569;   /* texto de metadados */
  --color-neutral-700: #334155;   /* texto de corpo */
  --color-neutral-800: #1E293B;   /* texto de cabeçalhos de seção */
  --color-neutral-900: #0F172A;   /* texto de título principal */

  /* ── Azul-petróleo (ação principal) ── */
  --color-primary-50:  #EFF6FF;
  --color-primary-100: #DBEAFE;
  --color-primary-200: #BFDBFE;
  --color-primary-400: #60A5FA;
  --color-primary-500: #3B82F6;   /* botão primário, links ativos */
  --color-primary-600: #2563EB;   /* hover de botão primário */
  --color-primary-700: #1D4ED8;   /* active state */
  --color-primary-900: #1E3A5F;   /* sidebar ativa, badges primários */

  /* ── Status: Sucesso (Concluído, SLA dentro do prazo) ── */
  --color-success-50:  #F0FDF4;
  --color-success-100: #DCFCE7;
  --color-success-500: #22C55E;
  --color-success-600: #16A34A;   /* ícone e texto de status Concluído */
  --color-success-700: #15803D;

  /* ── Status: Atenção (SLA em 80%, Follow-up próximo) ── */
  --color-warning-50:  #FFFBEB;
  --color-warning-100: #FEF3C7;
  --color-warning-400: #FBBF24;
  --color-warning-500: #F59E0B;   /* badge de SLA em risco */
  --color-warning-600: #D97706;   /* hover */
  --color-warning-700: #B45309;   /* texto sobre fundo âmbar claro */

  /* ── Status: Urgente (SLA vencido, erros, cancelamentos) ── */
  --color-danger-50:   #FFF1F2;
  --color-danger-100:  #FFE4E6;
  --color-danger-400:  #F87171;
  --color-danger-500:  #EF4444;   /* badge de SLA vencido, ações destrutivas */
  --color-danger-600:  #DC2626;
  --color-danger-700:  #B91C1C;

  /* ── Status: Informação (notificações neutras) ── */
  --color-info-50:   #F0F9FF;
  --color-info-500:  #0EA5E9;
  --color-info-600:  #0284C7;

  /* ── Superfícies ── */
  --color-surface-page:   var(--color-neutral-50);
  --color-surface-card:   #FFFFFF;
  --color-surface-panel:  var(--color-neutral-100);
  --color-surface-hover:  var(--color-neutral-100);
  --color-surface-active: var(--color-primary-50);
}
```

### 2.2 Semântica de cores de status (obrigatória)

| Contexto | Cor | Token | Uso semântico |
|---|---|---|---|
| **SLA dentro do prazo** | Verde | `--color-success-600` | Badge verde, indicador positivo |
| **SLA em risco (≥80%)** | Âmbar | `--color-warning-500` | Badge âmbar, alerta de atenção |
| **SLA vencido (100%+)** | Vermelho | `--color-danger-500` | Badge vermelho, urgência |
| **Caso Concluído** | Verde | `--color-success-600` | Estado terminal positivo |
| **Caso Cancelado** | Cinza | `--color-neutral-500` | Estado terminal neutro |
| **Ação primária** | Azul | `--color-primary-500` | Botões CTA, links ativos |
| **Ação destrutiva** | Vermelho | `--color-danger-600` | Cancelar Caso, desligar usuário |

> **Regra inviolável:** nunca usar cor de status de forma decorativa. Âmbar = atenção real. Vermelho = urgência real. Quem vê vermelho precisa agir.

### 2.3 Estados do Caso — mapeamento visual

| Estado | Cor de badge | Fundo do badge | Texto |
|---|---|---|---|
| Cadastro | `neutral-600` | `neutral-100` | `neutral-700` |
| Simulação | `primary-600` | `primary-50` | `primary-700` |
| Verificação | `warning-700` | `warning-50` | `warning-700` |
| Publicação | `info-600` | `info-50` | `info-600` |
| Match | `primary-700` | `primary-100` | `primary-800` |
| Negociação | `warning-600` | `warning-100` | `warning-700` |
| Anuência | `info-600` | `info-50` | `info-600` |
| Formalização | `primary-600` | `primary-50` | `primary-700` |
| Concluído | `success-700` | `success-50` | `success-700` |
| Cancelado | `neutral-500` | `neutral-100` | `neutral-600` |

---

## 3. Tipografia

### 3.1 Família tipográfica

| Uso | Família | Peso | Observação |
|---|---|---|---|
| **Interface geral** | Inter | 400, 500, 600, 700 | Carregada via `next/font/google`. Variable font. |
| **IDs de Caso (RS-XXXX)** | JetBrains Mono | 400, 500 | Monospace para identificadores. Clareza em listagens densas. |
| **Valores monetários em tabelas** | JetBrains Mono | 400 | Alinhamento tabular de dígitos. |
| **Código / endpoints** | JetBrains Mono | 400 | Snippets de erro, logs técnicos. |

```typescript
// app/layout.tsx
import { Inter, JetBrains_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-inter',
  display: 'swap',
});

const jetbrainsMono = JetBrains_Mono({
  subsets: ['latin'],
  variable: '--font-mono',
  display: 'swap',
});
```

### 3.2 Escala tipográfica

| Token | Tamanho | Line-height | Peso | Uso |
|---|---|---|---|---|
| `text-xs` | 11px | 16px | 400 | Metadados, timestamps, labels de campo |
| `text-sm` | 13px | 20px | 400/500 | Texto de corpo em tabelas, descrições |
| `text-base` | 15px | 24px | 400 | Texto de corpo padrão |
| `text-md` | 16px | 24px | 500/600 | Labels de seção, headings de card |
| `text-lg` | 18px | 28px | 600 | Título de página secundária, header de modal |
| `text-xl` | 20px | 30px | 700 | Título de página principal |
| `text-2xl` | 24px | 32px | 700 | Valores de KPI no Dashboard |
| `text-3xl` | 30px | 36px | 700 | Métricas de destaque no Dashboard Executivo |

> **Nota de densidade:** O CRM usa `text-sm` (13px) como padrão de tabelas e listas. Produtos consumer usam `text-base` (16px). A diferença é intencional — mais informação por tela.

### 3.3 Regras tipográficas

- IDs de Caso (`RS-2026-0001`) sempre em `font-mono`, `text-sm`, `tracking-wide`.
- Valores monetários em tabelas usam `font-mono` para alinhamento tabular de dígitos.
- Nomes de Cedentes e Cessionários em listagens usam mascaramento (RN-012): `text-sm font-medium` com truncamento após 20 caracteres.
- Datas e timestamps usam `text-xs text-neutral-500`.

---

## 4. Espaçamento e Layout

### 4.1 Grid e containers

| Contexto | Configuração |
|---|---|
| **Layout principal** | Sidebar fixa 240px + área de conteúdo fluida |
| **Área de conteúdo** | `max-width: 1440px`, padding horizontal `24px` |
| **Gutter padrão** | `gap-4` (16px) entre cards e seções |
| **Padding de card** | `p-4` (16px) — compacto para alta densidade |
| **Padding de modal** | `p-6` (24px) |
| **Padding de tabela (célula)** | `px-3 py-2` — reduzido vs. padrão shadcn |

### 4.2 Breakpoints do CRM

| Breakpoint | Largura | Comportamento |
|---|---|---|
| `tablet` | 768px | Sidebar colapsa para ícones. Pipeline mostra 3 colunas kanban. |
| `desktop-sm` | 1280px | Layout completo com sidebar expandida. |
| `desktop-lg` | 1440px+ | Pipeline kanban mostra todas as colunas sem scroll. |

---

## 5. Componentes shadcn/ui Customizados

### 5.1 Badge de Estado do Caso

Componente `<CaseStateBadge state={CaseState} />` — usa mapeamento da seção 2.3.

```typescript
// components/cases/case-state-badge.tsx
// Renderiza badge com cor semântica + ícone + rótulo textual
// Nunca apenas cor — sempre texto + cor (RN-199, acessibilidade)
```

- Tamanho: `text-xs font-medium px-2 py-0.5 rounded-full`
- Sempre inclui rótulo textual (não apenas ponto colorido)
- `aria-label` obrigatório: `aria-label="Estado: Negociação"`

### 5.2 Badge de SLA

Componente `<SlaBadge status="on_track" | "at_risk" | "overdue" daysRemaining={N} />`

| Status | Visual | Ícone Lucide |
|---|---|---|
| `on_track` | Verde, `success-50` bg | `CheckCircle` 14px |
| `at_risk` | Âmbar, `warning-50` bg | `AlertTriangle` 14px |
| `overdue` | Vermelho, `danger-50` bg | `XCircle` 14px |

- Sempre exibe dias restantes ou dias em atraso em texto: "+3 dias" ou "-2 dias"
- `role="status"` e `aria-live="polite"` para atualizações em tempo real

### 5.3 Card de Caso (Kanban)

Dimensões fixas: `width: 260px`, `min-height: 120px`, `p-3`.

Estrutura do card:
```
┌─────────────────────────────────┐
│ RS-2026-0001              [SLA] │  ← ID (mono) + badge SLA
│ João S. · Vila das Flores       │  ← Cedente mascarado + empreendimento
│ R$ 280.000                      │  ← Valor do contrato
│ ─────────────────────────────── │
│ [Avatar] Ana · 2 ativ. vencidas │  ← Analista RS + alertas
└─────────────────────────────────┘
```

- Drag handle: ícone `GripVertical` visível no hover do card
- Cursor: `cursor-grab` no hover, `cursor-grabbing` durante drag
- Shadow durante drag: `shadow-lg ring-2 ring-primary-300`

### 5.4 Linha do Tempo do Caso

Componente `<CaseTimeline events={TimelineEvent[]} />` — scroll vertical infinito.

| Tipo de evento | Ícone | Cor da linha |
|---|---|---|
| Avanço de estado | `ArrowRight` | `primary-500` |
| Retrocesso de estado | `ArrowLeft` | `warning-500` |
| Atividade registrada | `MessageSquare` | `neutral-300` |
| Documento enviado | `FileText` | `neutral-300` |
| Alerta de SLA | `AlertTriangle` | `warning-500` |
| Alerta SLA vencido | `XCircle` | `danger-500` |
| Sistema | `Zap` | `neutral-200` |

### 5.5 Formulário de Atividade

Campos com densidade reduzida. Label acima do campo, `text-xs` uppercase para labels.

```
TIPO DE ATIVIDADE *
[Select: Ligação ▾]

RESUMO DA INTERAÇÃO *
[Textarea: mínimo 20 caracteres]

RESULTADO
[Select: Sem resposta / Contatado — follow-up necessário / ...]

[Cancelar]  [Registrar Atividade →]
```

### 5.6 Modal de Avanço de Estado

Exibe checklist de condições de saída do estado atual (RN-007). Cada item com ícone `CheckCircle` (verde) ou `Circle` (cinza). Botão "Avançar" habilitado apenas quando todos os itens estão marcados como atendidos.

---

## 6. Ícones

### 6.1 Biblioteca e tamanhos

Exclusivamente **Lucide React**. Nenhuma outra biblioteca de ícones.

| Contexto | Tamanho | `strokeWidth` |
|---|---|---|
| Em texto (inline) | 14px | 1.5 |
| Em listas e tabelas | 16px | 1.5 |
| Em botões de ação | 16px | 2 |
| Em navegação sidebar | 20px | 1.5 |
| Em ações primárias CTA | 20px | 2 |
| Em estados vazios (empty state) | 48px | 1 |

### 6.2 Ícones semânticos obrigatórios

| Elemento | Ícone Lucide |
|---|---|
| Novo Caso | `Plus` |
| Avançar estado | `ArrowRight` |
| Retroceder estado | `ArrowLeft` |
| Cancelar Caso | `X` |
| Dossiê / Documentos | `FolderOpen` |
| Upload de documento | `Upload` |
| Atividade / Histórico | `Clock` |
| Follow-up agendado | `CalendarClock` |
| Follow-up vencido | `AlertCircle` |
| WhatsApp | `MessageCircle` |
| E-mail | `Mail` |
| Redistribuir Caso | `Shuffle` |
| SLA ok | `CheckCircle` |
| SLA em risco | `AlertTriangle` |
| SLA vencido | `XCircle` |
| Concluído | `CheckCircle2` |
| Cancelado | `Ban` |
| Comissão / Financeiro | `DollarSign` |
| Negociação | `Handshake` |
| Busca global | `Search` |
| Configurações | `Settings` |
| Equipe | `Users` |
| Dashboard | `LayoutDashboard` |
| Relatórios | `BarChart2` |
| Log de auditoria | `Shield` |
| Notificação | `Bell` |
| Perfil / Usuário | `User` |
| Parceiro Externo | `UserCheck` |
| Cedente | `UserMinus` |
| Cessionário | `UserPlus` |

---

## 7. Componentes de Estado

### 7.1 Empty State (estado vazio)

Estrutura padronizada para qualquer tela sem dados:

```
[Ícone 48px, neutral-300]
Nenhum caso encontrado
Abra um novo caso para começar.
[Botão primário: Novo Caso]
```

- Ícone sempre de Lucide, `strokeWidth: 1`, `neutral-300`
- Título: `text-base font-semibold text-neutral-700`
- Descrição: `text-sm text-neutral-500`
- Sem ilustrações decorativas

### 7.2 Loading State (carregamento)

**Spinners globais de página são proibidos.** Usar skeleton screens em todo estado de carregamento.

| Componente | Skeleton |
|---|---|
| Card de Caso | Retângulo animado 260×120px |
| Linha de tabela | 3-4 retângulos com larguras variadas |
| Dados do Dashboard | Cards com retângulos de métrica |
| Linha do tempo | Círculos + retângulos alternados |

```css
/* Animação de shimmer */
@keyframes shimmer {
  from { background-position: -200% 0; }
  to   { background-position:  200% 0; }
}
.skeleton {
  background: linear-gradient(
    90deg,
    var(--color-neutral-200) 25%,
    var(--color-neutral-100) 50%,
    var(--color-neutral-200) 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s ease-in-out infinite;
  border-radius: 4px;
}
```

### 7.3 Error State (erro)

```
[AlertTriangle 48px, danger-400]
Erro ao carregar os casos
Verifique sua conexão e tente novamente.
[Tentar novamente]
```

### 7.4 Notificações (Toast)

Usando `sonner` via shadcn/ui. Posição: canto superior direito. Duração padrão: 4 segundos.

| Tipo | Ícone | Cor esquerda |
|---|---|---|
| Sucesso | `CheckCircle` | `success-500` |
| Erro | `XCircle` | `danger-500` |
| Atenção | `AlertTriangle` | `warning-500` |
| Info | `Info` | `info-500` |

---

## 8. Navegação

### 8.1 Sidebar (navegação principal)

Largura: 240px expandida, 56px colapsada (ícones apenas).

```
┌──────────────────────────────────────┐
│  ◈ Repasse Seguro — CRM              │  ← Logo + nome
│  ─────────────────────────────────── │
│  [Avatar] Ana Oliveira               │  ← Nome + papel do usuário
│  Analista RS                         │
│  ─────────────────────────────────── │
│  ⌘K  Busca global...                 │  ← Busca (RN-197)
│  ─────────────────────────────────── │
│  🏠  Painel                          │
│  📋  Pipeline                        │  ← Kanban / Lista de Casos
│  👥  Contatos                        │
│  📅  Atividades                      │
│  💬  Comunicação                     │
│  ─────────────────────────────────── │
│  [somente Admin/Coordenador]         │
│  📊  Dashboard Executivo             │
│  📤  Relatórios                      │
│  ─────────────────────────────────── │
│  [somente Admin]                     │
│  👤  Equipe                          │
│  ⚙️  Configurações                   │
│  ─────────────────────────────────── │
│  🔔  [N] Notificações                │  ← Badge com contagem
│  ❓  Ajuda                           │
└──────────────────────────────────────┘
```

- Seções restritas por papel são ocultas (não apenas desabilitadas) para papéis sem permissão.
- Item ativo: fundo `primary-50`, borda esquerda `2px solid primary-500`, texto `primary-700`.

### 8.2 Breadcrumb e títulos de página

Padrão: `Pipeline / RS-2026-0001 / Dossiê`

- `text-sm text-neutral-500` com separador `/`
- Último item: `text-sm font-medium text-neutral-800`

---

## 9. Formulários

### 9.1 Anatomia de campo

```
Label (text-xs uppercase tracking-wide text-neutral-600) *
[Input / Select / Textarea]
Texto auxiliar ou mensagem de erro (text-xs text-neutral-500 / danger-600)
```

### 9.2 Estados de input

| Estado | Borda | Sombra |
|---|---|---|
| Default | `neutral-300` | nenhuma |
| Hover | `neutral-400` | nenhuma |
| Focus | `primary-500` | `ring-2 ring-primary-200` |
| Preenchido | `neutral-300` | nenhuma |
| Erro | `danger-500` | `ring-2 ring-danger-200` |
| Desabilitado | `neutral-200` | nenhuma, fundo `neutral-50` |

### 9.3 Botões

| Variante | Uso | Aparência |
|---|---|---|
| `primary` | Ação principal (Criar Caso, Registrar Atividade) | `bg-primary-500 text-white hover:bg-primary-600` |
| `secondary` | Ação secundária | `bg-white border border-neutral-300 text-neutral-700` |
| `ghost` | Ações terciárias, links de ação em tabelas | `bg-transparent text-neutral-600 hover:bg-neutral-100` |
| `destructive` | Cancelar Caso, Desligar Usuário | `bg-danger-500 text-white hover:bg-danger-600` |
| `outline-primary` | Ação alternativa importante | `border border-primary-500 text-primary-600` |

- Spinner dentro de botão: permitido (apenas para feedback de submissão de form)
- Loading state de botão: `opacity-70 cursor-not-allowed` com spinner inline 14px
- Tamanho padrão: `h-9 px-4 text-sm` (compacto para alta densidade)

---

## 10. Tabelas

Padrão de alta densidade para listagens de Casos, Contatos e Relatórios.

| Propriedade | Valor |
|---|---|
| Altura da linha | 40px (padrão) |
| Padding de célula | `px-3 py-2` |
| Fonte da célula | `text-sm` |
| Fonte de cabeçalho | `text-xs uppercase tracking-wide text-neutral-500` |
| Hover de linha | `bg-neutral-50` |
| Linha selecionada | `bg-primary-50` |
| Borda de separação | `border-b border-neutral-200` |

- Colunas numéricas (valores R$) alinhadas à direita, `font-mono`
- Colunas de data alinhadas à esquerda, `text-xs text-neutral-500`
- IDs de Caso (RS-XXXX): `font-mono text-sm text-primary-600 underline cursor-pointer`

---

## 11. Regras de Acessibilidade

Todas as regras derivam de RN-199.

| Requisito | Implementação |
|---|---|
| **Contraste mínimo 4.5:1** | Todos os pares de cor testados via `color-contrast()`. Sem exceções. |
| **Indicadores não apenas por cor** | Todo indicador de status tem texto + ícone além da cor |
| **Focus visível** | `ring-2 ring-primary-500 ring-offset-2` em todos os elementos interativos |
| **Rótulos em inputs** | Nunca usar apenas placeholder como label. Label sempre presente. |
| **ARIA em componentes dinâmicos** | `aria-live="polite"` em atualizações de SLA e status em tempo real |
| **Teclado** | Todos os fluxos críticos navegáveis por teclado. `Tab`, `Enter`, `Escape` para modais. |
| **Screen readers** | `aria-label` em ícones sem texto. `role="status"` em badges de SLA. |

---

## 12. Modo Escuro

**Não suportado na v1.0.** Light mode apenas.

O sistema operacional pode estar em dark mode — a aplicação permanece em light mode via `color-scheme: light` forçado no `html`.

```css
html {
  color-scheme: light;
}
```

Versão futura (v2.0) avaliará dark mode conforme demanda da equipe operacional.
