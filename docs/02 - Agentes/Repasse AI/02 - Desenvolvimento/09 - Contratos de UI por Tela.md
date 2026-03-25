# Repasse AI
## 09 — Contratos de UI por Tela

| Campo | Valor |
|---|---|
| **Destinatário** | Frontend Lead / UI Designer |
| **Escopo** | Mapeamento de componentes shadcn/ui, tokens e estados obrigatórios por tela |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 22/03/2026 00:00 (America/Fortaleza) |
| **Status** | Ativo |
| **Fase** | 2 — UI/UX |
| **Área** | Design + Frontend |
| **Referências** | 01 - Regras de Negócio · 03 - Brand Theme Guide · 06 - Mapa de Telas · 07 - Wireframes · 08 - UX Writing |

---

> 📌 **TL;DR**
>
> - **22 telas** cobertas com contratos completos (T-001 a T-022), sem pendências.
> - **4 estados obrigatórios** por tela: Skeleton → Empty → Error → Populated. Spinners proibidos em todo o documento.
> - **3 layouts base reutilizados:** `AppLayout` (sidebar fixa — Admin T-020/T-022), `OverlayLayout` (modais e overlays — T-002/T-016/T-017), `ChatPanelLayout` (painel lateral/full-screen — T-003 a T-014). [DECISÃO AUTÔNOMA — layout "ChatPanelLayout" não é um dos 4 base do template genérico, mas é a arquitetura de renderização do chat que o Repasse AI usa como módulo embutido no frontend do Cessionário; alternativa descartada: FullPageLayout (o chat não ocupa página inteira exceto em mobile).]
> - **Componentes críticos de domínio identificados:** ScoreIndicador (risco), ConfidenceBadge (Admin), OportunidadeCard, ChatBubble (7 variantes), StepProgress (OTP), MetricCard (dashboard Admin).
> - **RBAC:** `cessionario` (T-001 a T-019), `admin` (T-020 a T-022) — via `useRbac()` + `RbacGuard`.
> - **Tokens canônicos:** `--primary` (#0069A8), `--destructive` (#E7000B light / #FF6467 dark), `--muted-foreground`, `--card`, `--border`, `--chart-1` a `--chart-5`.
> - **Telas ou seções pendentes:** 0.

---

## 1. Persona

Frontend Lead / UI Designer responsável por transformar cada tela do Repasse AI em contrato explícito de componentes, estados e tokens. O documento materializa o nível de detalhe de implementação visual — um dev aplica cada tela sem ambiguidade.

**Impacto no pipeline:**
- **11 - Mobile:** usa os contratos como referência de adaptação mobile por tela.
- **15 - Arquitetura de Pastas:** mapeia onde cada tela vive em `features/<módulo>/pages/`.
- **28 - Checklist de Qualidade:** valida conformidade de componentes, estados e tokens por tela.

---

## 2. Convenções Globais

### 2.1 Quatro Estados Obrigatórios por Tela

| Estado | Componente | Trigger |
|--------|-----------|---------|
| **Skeleton** | `<Skeleton>` shadcn/ui — **nunca `<Spinner>`** | Enquanto fetch em andamento |
| **Empty** | Componente custom `<EmptyState>` (ilus. + CTA opcional) | `data.length === 0` ou estado não iniciado |
| **Error** | `<Alert variant="destructive">` + botão retry | Request falha ou erro de rede |
| **Populated** | Conteúdo real renderizado | Dados carregados com sucesso |

> 🔴 **Spinners são proibidos em todo o produto.** Qualquer estado de carregamento usa `<Skeleton>` shadcn/ui.

### 2.2 Layouts Base

| Layout | Usado em | Estrutura |
|--------|----------|-----------|
| `ChatPanelLayout` | T-001 a T-014 | FAB flutuante + painel lateral 420px (desktop), drawer 400px (tablet), full-screen 100dvh (mobile < 768px) |
| `ProfileSectionLayout` | T-015 a T-019 | Subseção dentro de "Meu Perfil" da plataforma; sem sidebar própria |
| `OverlayLayout` | T-002, T-016, T-017 | Modal/overlay sobre o conteúdo com focus trap; `role="dialog"` obrigatório |
| `AppLayout` | T-020 a T-022 | Sidebar fixa 280px + content area (Admin); sidebar recolhível em tablet |

### 2.3 Componentes Críticos de Domínio

| Componente | Telas | Tokens Principais | Faixas Semânticas |
|-----------|-------|-------------------|-------------------|
| **ScoreIndicador** | T-004, T-008 | `--chart-1` (verde custom), `--chart-3` (amarelo custom), `--destructive` | Score 1–3 → verde; 4–6 → amarelo; 7–10 → vermelho |
| **ConfidenceBadge** | T-020, T-021 | `--chart-2` (≥80%), `--chart-3` (50–79%), `--destructive` (<50%) | Badge colorida com % numérico |
| **OportunidadeCard** | T-004, T-005, T-006, T-007, T-012 | `--card`, `--border`, `--primary` | Análise completa / Compacto / Mini chip |
| **ChatBubble** | T-003 a T-014 | `--card` (agente), `--primary` (user), `--muted` (sistema) | 7 variantes (ver 2.4) |
| **StepProgress** | T-016 | `--primary`, `--muted`, `--muted-foreground` | Steps 1–3 com estado ativo/concluído/pendente |
| **OTPInput** | T-016 | `--input`, `--destructive`, `--ring` | Normal / Erro / Bloqueado |
| **MetricCard** | T-022 | `--card`, `--foreground`, `--muted-foreground` | Volume (primário), CSAT (destaque), restantes (secondary) |
| **AlertaBadge** | T-001, T-012–T-014, T-020 | `--destructive`, `--primary` | Numérica (FAB) / Cor semântica (confiança) |

### 2.4 ChatBubble — 7 Variantes

```
ChatBubble variant="user"       → alinhada direita, bg --primary, texto --primary-foreground
ChatBubble variant="agent"      → alinhada esquerda, bg --card, texto --foreground
ChatBubble variant="system"     → centralizada, bg --muted, texto --muted-foreground, italic
ChatBubble variant="analysis"   → alinhada esquerda, card estruturado com subcomponentes
ChatBubble variant="alert"      → alinhada esquerda, badge "Novo alerta" --primary no topo
ChatBubble variant="error"      → alinhada esquerda, border-left --destructive, 2px solid
ChatBubble variant="blocked"    → alinhada esquerda, texto --foreground, sem borda de erro
```

### 2.5 Padrão DataTable (Admin)

```tsx
<DataTable
  columns={columns}
  data={data}
  skeleton={<TableSkeleton rows={10} />}
  emptyState={<EmptyState icon="inbox" message="Nenhuma interação registrada ainda." />}
  pagination={{ pageSize: 20 }}
/>
```

> Nunca spinner. Máximo 100 itens por página. Colunas ordenáveis com `aria-sort`.

### 2.6 StatusBadge — Mapeamentos de Enums

```
StatusBadge — WhatsappBindingStatusEnum:
nao_vinculado          → bg --muted, text --muted-foreground, "Não vinculado"
aguardando_otp         → bg --chart-3 10%, text --chart-3, "Aguardando código"
aguardando_confirmacao → bg --chart-3 10%, text --chart-3, "Aguardando WhatsApp"
ativo                  → bg --chart-1 10%, text green-700 (custom), "Vinculado ✓"
aguardando_reverificacao → bg --chart-3 10%, text --chart-3, "Reverificação pendente"
suspenso               → bg --destructive 10%, text --destructive, "Suspenso"
desvinculado           → bg --muted, text --muted-foreground, "Desvinculado"

StatusBadge — ConversationStatusEnum:
active  → bg --chart-1 10%, text green-700, "Ativa"
closed  → bg --muted, text --muted-foreground, "Encerrada"

StatusBadge — InteractionStatusEnum (Admin T-020/T-021):
completed → bg --chart-1 10%, text green-700, "Concluída"
error     → bg --destructive 10%, text --destructive, "Erro"
timeout   → bg --chart-3 10%, text amber-700 (custom), "Timeout"
takeover  → bg --primary 10%, text --primary, "Takeover"

StatusBadge — NotificationStatusEnum (Admin T-022):
pending    → bg --muted, text --muted-foreground, "Pendente"
delivered  → bg --chart-1 10%, text green-700, "Entregue"
failed     → bg --destructive 10%, text --destructive, "Falhou"
suppressed → bg --muted, text --muted-foreground, "Suprimido"

ConfidenceBadge (T-020, T-021):
≥ 80%    → bg --chart-2 10%, text --chart-2, "[valor]% Alta"
50–79%   → bg --chart-3 10%, text amber-700, "[valor]% Média"
< 50%    → bg --destructive 10%, text --destructive, "[valor]% Baixa"
```

---

## 3. Módulo A — Webchat (T-001 a T-011)

### T-001 — FAB do Chat (Botão Flutuante)

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — FAB posicionado `position: fixed; bottom: 24px; right: 24px` (desktop); `bottom: 16px; right: 16px; width: 56px; height: 56px` (mobile) |
| **Role mínimo** | `cessionario` (via `RbacGuard`) |
| **RFs** | RF-040 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Button` | `size="icon"`, `rounded-full`, bg `--primary`, shadow `shadow-lg` | Botão flutuante de acesso ao chat |
| `Badge` | `variant="destructive"`, `rounded-full`, 20px × 20px, texto 11px, posição `absolute top-0 right-0` | Badge numérica de alertas não lidos (máx "9+") |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | Não aplicável — FAB renderiza com estado local imediato |
| **Empty** | FAB sem Badge; ícone `MessageCircle` (lucide-react) em `--primary-foreground` |
| **Error** | Não aplicável — erros ocorrem dentro do chat |
| **Populated** | FAB com `<Badge variant="destructive">` exibindo contagem; máx "9+" |

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Tab order natural; foco visível com `--ring` | FAB focável via teclado |
| ARIA roles | `aria-label="Abrir Analista de Oportunidades"` | Screen reader anuncia corretamente |
| Skip links | Não aplicável (FAB é elemento de ação, não navegação) | — |
| Contraste | Ícone branco `--primary-foreground` sobre `--primary` #0069A8 → ratio 4.7:1 | WCAG AA aprovado |
| Motion | Badge anima com `scale-in` (150ms); `prefers-reduced-motion` → sem animação | Validado via CSS media query |

**Motion & Transições:**
- Badge aparece: `scale-in` 150ms `ease-out`; desaparece: `scale-out` 150ms
- `prefers-reduced-motion: reduce` → sem animação de scale; troca direta de estado
- Offline: FAB com `opacity-50`, tooltip `<TooltipContent>"Sem conexão"</TooltipContent>`

**Regras específicas:**
- FAB sempre visível em todas as telas do Cessionário — `position: fixed`, `z-index: 50`
- Badge exibe número real até 9; exibe "9+" para 10 ou mais
- Badge com `aria-live="polite"` para anunciar novas notificações sem interromper o leitor

---

### T-002 — Banner de Consentimento LGPD

| Atributo | Valor |
|----------|-------|
| **Layout** | `OverlayLayout` — overlay semi-opaco sobre toda a tela; banner centralizado `max-w-lg` (desktop), full-screen com scroll (mobile) |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-101 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Card` | `radius-xl` (19.6px), `padding: 32px`, bg `--card`, shadow `shadow-xl` | Container do banner |
| `CardHeader` | Default | Título e subtítulo LGPD |
| `CardContent` | Default | Texto da política com `<ScrollArea max-h-64>` |
| `Button` | `variant="default"`, `size="lg"`, `full-width`, bg `--primary` | CTA "Aceitar" (ação primária) |
| `Button` | `variant="ghost"`, `size="sm"` | Link "Ver política de privacidade" (abre nova aba) |
| `Alert` | `variant="destructive"` | Exibido em caso de falha ao registrar consentimento |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | Não aplicável — banner renderiza sincronicamente |
| **Empty** | Não aplicável |
| **Error** | `<Alert variant="destructive">` + botão "Tentar novamente" |
| **Populated** | Banner com texto, link para política, botões "Aceitar" e "Ver política" |

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | `role="dialog"`, `aria-modal="true"`, focus trap ativo; ao montar, foco no botão "Aceitar" | Navegação por Tab dentro do modal apenas |
| ARIA roles | `aria-labelledby` aponta para título do banner | Screen reader anuncia contexto de consentimento |
| Skip links | Não aplicável (overlay bloqueia interação externa) | — |
| Contraste | `--foreground` #0A0A0A sobre `--card` #FFFFFF → 19.5:1 | WCAG AAA |
| Motion | Aceite → banner `slide-up` 300ms `ease-in-out`; chat `fade-in` 200ms | `prefers-reduced-motion` → sem animação |

**Motion & Transições:**
- Entrada: `fade-in` 200ms `ease-out`
- Aceite: banner `slide-up` 300ms + chat `fade-in` 200ms em sequência
- `prefers-reduced-motion: reduce` → troca direta de estado

**Regras específicas:**
- Input do chat bloqueado (`pointer-events: none`, `aria-disabled="true"`) enquanto consentimento não registrado
- Offline: botão "Aceitar" `disabled` com `<Tooltip>"Necessário ter conexão para aceitar."</Tooltip>`
- Em caso de erro no registro: banner permanece visível; toast no topo com `"Não foi possível registrar seu consentimento. Tente novamente."`

---

### T-003 — Janela do Chat — Estado Ativo

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — painel lateral fixo 420px (desktop > 1024px); drawer 400px overlay (tablet 768–1024px); full-screen `100dvh` (mobile < 768px) |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-038, RF-042, RF-046 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `ScrollArea` | `flex-1`, overflow-y auto, padding `16px` | Área de histórico de mensagens |
| `Skeleton` | 3 linhas `h-12 w-full`, `w-2/3`, `w-3/4`, gap `8px` | Loading do histórico |
| `Input` | `variant="default"`, `radius-md`, `placeholder="Escreva sua mensagem..."` | Campo de entrada do usuário |
| `Button` | `variant="default"`, `size="icon"`, `aria-label="Enviar mensagem"` | Botão de envio (ícone `Send`) |
| `Badge` | `variant="secondary"`, chip clicável, `radius-sm` | Sugestões de conversa (empty state) |
| `Button` | `variant="ghost"`, `size="sm"`, texto "Carregar mais" | Paginação do histórico (partial state) |
| `Alert` | `variant="default"` | Banner offline / histórico indisponível |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | 3 `<Skeleton>` de linhas de mensagem com larguras variadas; duração máx 500ms antes de fallback |
| **Empty** | Mensagem de boas-vindas contextualizada + chips de sugestão clicáveis (`<Badge variant="secondary">`) em `--muted-foreground` |
| **Error** | `<Toast>` no topo da janela: "Não foi possível carregar o histórico." — chat continua funcional |
| **Populated** | Histórico com ChatBubbles; timestamps em `--muted-foreground`; histórico rola acima do input ancorado |

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Ao abrir: foco no `<Input>`; Tab order: Input → Botão Envio → Sugestões | Navegação completa via teclado |
| ARIA roles | `role="log"` + `aria-live="polite"` na área de histórico; `aria-label` no Input | Screen reader anuncia novas mensagens |
| Skip links | Não aplicável (painel é componente, não página) | — |
| Contraste | Todos os elementos em paleta Brand Guide → ratio ≥ 4.5:1 | Validado via axe-core |
| Motion | Indicador "digitando": 3 pontos pulsando 800ms; `prefers-reduced-motion` → texto estático "Analisando..." | Sem layout shift durante streaming |

**Motion & Transições:**
- Indicador SSE "digitando": 3 pontos pulsando (800ms ciclo, `opacity` 0.3 → 1 → 0.3)
- Stream SSE: tokens renderizados progressivamente sem layout shift (CSS `word-break: break-word`)
- Edge case — stream interrompido: bolha com texto recebido + `"Resposta incompleta"` em `--muted-foreground` italic + botão `"Tentar novamente"` (variant ghost)
- `prefers-reduced-motion: reduce` → indicador de digitação como texto "Analisando..." sem animação

**Regras específicas:**
- Input ancorado no bottom (`position: sticky; bottom: 0`); histórico rola livremente acima
- Últimas 20 mensagens carregadas; botão "Carregar mais" acima da primeira mensagem visível
- Offline: `<Alert variant="default">` "Você está offline. As mensagens serão enviadas quando a conexão voltar." — histórico em cache exibido normalmente
- Indicador de mensagens restantes: `aria-describedby` no Input apontando para contador `"[N] de 30 mensagens disponíveis"`

---

### T-004 — Chat — Bolha de Análise de Oportunidade

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="analysis"` dentro do histórico |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-001, RF-002, RF-008, RF-009, RF-010 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Card` | `radius-lg`, border `--border`, shadow `shadow-sm` | Container da bolha de análise |
| `Skeleton` | inline, altura do card completo | Loading durante streaming |
| **`ScoreIndicador`** (custom) | Baixo/Moderado/Alto com cor semântica | Score de risco 1–10 |
| `Tabs` | `variant="default"`, 3 tabs: Conservador/Base/Otimista | ROI por cenário |
| `TabsContent` | Default | Conteúdo de cada cenário de ROI |
| `Badge` | `variant="secondary"` laranja, custom | Badge "Em negociação" |
| `Badge` | `variant="secondary"` cinza, custom | Badge "Encerrada" |
| `Badge` | `variant="secondary"` + cor dinâmica | Status da oportunidade |
| `Button` | `variant="outline"`, `size="sm"` | "Me avisar quando disponível" (Em Negociação) |
| `Collapsible` | Default | "Ver mais detalhes" — comparativo regional e histórico |
| `CollapsibleContent` | Default | Comparativo regional + gráfico/tabela histórico |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | `<Skeleton>` inline cobrindo toda a área do card durante streaming |
| **Empty** | Não aplicável — bolha só renderiza quando há resposta |
| **Error** | OPR não encontrado: ícone `Search` (lucide) + mensagem de oportunidade não encontrada (variant blocked) |
| **Populated** | Card completo: Δ (destaque bold large), Comissão, Custo Escrow, ROI em Tabs, ScoreIndicador, Collapsible para comparativo |

**Hierarquia visual obrigatória:**
1. **Δ (Delta)** — maior tamanho de fonte (`text-2xl font-bold`), `--foreground`; elemento primário de destaque
2. **Comissão + Custo Escrow** — tamanho médio (`text-lg`), na mesma linha ou logo abaixo
3. **ScoreIndicador** — badge colorida semântica; elemento de atenção secundário
4. **ROI (Tabs)** — ao lado do ScoreIndicador; 3 cenários em tabs
5. **Comparativo Regional + Histórico** — em `<Collapsible>` "Ver mais detalhes"; não visível por padrão

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Tabs com `role="tablist"` + `role="tab"` + `aria-selected` | Navegação de tabs via teclado (Arrow keys) |
| ARIA roles | ScoreIndicador: `aria-label="Score de risco: [valor] — [Baixo/Moderado/Alto]"` + `role="status"` | Screen reader anuncia score |
| Skip links | Não aplicável | — |
| Contraste | Todos os tokens semânticos respeitam ratio ≥ 4.5:1 | Validado |
| Motion | Badge de status: `scale-in` ao finalizar stream; `prefers-reduced-motion` → sem animação | OK |

**Motion & Transições:**
- Streaming: tokens renderizados progressivamente
- ScoreIndicador badge: `scale-in` 200ms `ease-out` ao finalizar o stream
- Cards "Em Negociação"/"Encerrada": `fade-in` 150ms ao renderizar estado
- `prefers-reduced-motion` → sem animações; renderização direta

**Regras específicas:**
- Modo "Em Negociação": badge laranja + botão "Me avisar quando disponível" (`variant="outline"`, `size="sm"`)
- Modo "Encerrada": badge cinza + chips de até 3 oportunidades semelhantes (`<Badge variant="outline">`)
- Gráfico de histórico: usando `--chart-1` a `--chart-5` (paleta azul); substituído por tabela textual no canal WhatsApp
- Score omitido (dados insuficientes): mensagem `"Score indisponível — dados insuficientes"` em `--muted-foreground`, sem score parcial

---

### T-005 — Chat — Bolha de Comparação

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="analysis"` com tabela responsiva |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-014 a RF-018 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Table` | `overflow-x-auto`, `border-collapse`, col de destaque com `border --primary 2px` | Tabela comparativa de oportunidades |
| `TableHeader` | Fixo (`sticky top-0`), bg `--card` | Headers das colunas (OPRs) |
| `TableCell` | Default; coluna destaque com `bg --primary/5` | Células comparativas |
| `Skeleton` | `TableSkeleton` — header + 3 linhas placeholder | Loading de tabela |
| `Badge` | `variant="outline"`, border `--primary` | Indicador da coluna de destaque (Maior Δ / Menor Risco) |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | `<TableSkeleton rows={3}>` — header placeholder + 3 linhas com Skeleton |
| **Empty** | Não aplicável — bolha só aparece quando há resposta |
| **Error** | OPR faltante: mensagem inline `"Oportunidade [OPR] não encontrada"` + colunas restantes preenchidas com "-" |
| **Populated** | Tabela com até 5 colunas; coluna destaque com borda `--primary` e `<Badge>` |

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Cells focáveis via Tab | Navegação completa por teclado |
| ARIA roles | `role="table"`, `scope="col"` em headers | Screen reader lê corretamente |
| Skip links | Não aplicável | — |
| Contraste | Ratio ≥ 4.5:1 em todas as células | OK |
| Motion | Scroll hint: chevron right com pulse 2 ciclos × 1s; `prefers-reduced-motion` → sem animação | OK |

**Motion & Transições:**
- Tabela renderiza coluna por coluna durante streaming (stagger 80ms entre colunas)
- Scroll hint mobile: ícone `ChevronRight` com `animate-pulse` 2 ciclos × 1s; desaparece após primeiro scroll ou 3s
- `prefers-reduced-motion` → sem stagger, sem pulse; tabela renderiza diretamente

**Regras específicas:**
- Desktop: 5 colunas visíveis; tablet: 4 colunas; mobile: 3 colunas + scroll horizontal
- Colunas faltantes preenchidas com "-" (dado indisponível)
- Cabeçalho fixo (`sticky`) durante scroll horizontal em mobile

---

### T-006 — Chat — Bolha de Simulação (Proposta / Contraproposta)

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="analysis"` card único |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-019 a RF-024 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Card` | `radius-lg`, border `--border` | Container da simulação |
| `Skeleton` | Card único | Loading |
| `Separator` | Default (`--border`) | Divisor entre valor proposto e ROI |
| `p` / `span` | `text-sm italic text-muted-foreground` | Nota de recusa de submissão automática |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | `<Skeleton>` cobrindo o card completo |
| **Empty** | Não aplicável |
| **Error** | Valor inválido ou OPR não encontrado: `ChatBubble variant="error"` com mensagem inline |
| **Populated** | Card: valor proposto, comissão calculada, custo total Escrow, ROI ajustado (3 cenários) |

**Motion & Transições:**
- Streaming: ROI renderiza cenário por cenário (Conservador → Base → Otimista), stagger 100ms
- `prefers-reduced-motion` → renderização direta

**Regras específicas:**
- Nota de recusa de submissão automática em `--muted-foreground italic text-sm`: _"O Analista de Oportunidades não submete propostas automaticamente. Esta é uma simulação para sua análise."_
- Cálculos são determinísticos — simulação não usa o agente de IA

---

### T-007 — Chat — Bolha de Portfólio

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="analysis"` com cards empilhados |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-026 a RF-029 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Card` | `radius-lg`, por oportunidade | Card individual de cada oportunidade do portfólio |
| `Card` | `radius-lg`, bg `--primary/5`, border `--primary` | Card de resumo do portfólio (destaque) |
| `Skeleton` | 3 cards empilhados | Loading |
| `Tabs` | 3 tabs: Conservador/Base/Otimista | ROI total do portfólio |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | 3 `<Skeleton>` de cards empilhados |
| **Empty** | Não aplicável |
| **Error** | Oportunidade inválida: indicação inline na posição do card `"[OPR] não encontrado"` |
| **Populated** | Cards por oportunidade + card de resumo do portfólio com capital comprometido total e ROI em Tabs |

**Motion & Transições:**
- Cards aparecem sequencialmente durante streaming: stagger 100ms entre cards
- `prefers-reduced-motion` → todos os cards renderizam simultaneamente

---

### T-008 — Chat — Bolha de Suporte Operacional

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="agent"` |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-047, RF-048 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `ChatBubble` | `variant="agent"` | Container da resposta de suporte |
| `Skeleton` | Parágrafo único | Loading |
| `a` | `text-primary underline`, `target="_blank"` | Links para documentação (quando disponíveis) |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | `<Skeleton className="h-20 w-full">` simulando parágrafo |
| **Empty** | Não aplicável |
| **Error** | Consulta fora do escopo: mensagem de fallback em `ChatBubble variant="agent"` com sugestão de redirecionamento |
| **Populated** | Texto formatado: cabeçalho da regra, parágrafos de detalhe, links quando disponíveis |

**Regras específicas:**
- Resposta parcial: indicação `"Dados adicionais podem ser consultados em [link da plataforma]"` em `--muted-foreground`

---

### T-009 — Chat — Bolha de Erro / Degradação

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="error"` |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-063, RF-064, RF-065 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `ChatBubble` | `variant="error"`, border-left `--destructive 2px` | Container do erro |
| `Button` | `variant="outline"`, `size="sm"` | "Aguardar" — mantém loading ativo |
| `Button` | `variant="default"`, `size="sm"` | "Tentar novamente" |
| `Button` | `variant="secondary"`, `size="sm"` | "Calcular comissão apenas" (quando fallback disponível) |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | Não aplicável |
| **Empty** | Não aplicável |
| **Error** | Esta bolha **é** o estado de erro — aparece após timeout (> 10s) ou agente indisponível |
| **Populated** | Mensagem de erro + 2 ou 3 botões de ação |

**Motion & Transições:**
- Bolha aparece com `fade-in` 200ms após timeout
- `prefers-reduced-motion` → sem animação; renderização direta

**Regras específicas:**
- Exibida quando: (a) LLM não responde em ≤ 10s, (b) agente desligado por taxa de erro > 30%
- Botão "Calcular comissão apenas" exibido somente quando Calculadora de Comissão disponível como fallback

---

### T-010 — Chat — Bolha de Dado Bloqueado

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="blocked"` |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-034, RF-035 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `ChatBubble` | `variant="blocked"` — texto `--foreground`, sem borda de erro | Mensagem informativa (não é erro técnico) |
| `Badge` | `variant="secondary"`, chip clicável, segunda insistência | Sugestões de conversa alternativa |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | Não aplicável |
| **Empty** | Não aplicável |
| **Error** | Não aplicável — este é um estado de negócio, não técnico |
| **Populated** | Mensagem informativa sem ícone de alerta; na 2ª insistência: + chips de sugestão |

**Regras específicas:**
- Sem ícone de alerta vermelho — a mensagem é informativa, não alarmante [DECISÃO AUTÔNOMA — ícone de alerta criaria ansiedade desnecessária; alternativa descartada: ícone de cadeado (sugere dado trancado, não inacessível)]
- 1ª insistência: mensagem padrão
- 2ª insistência: mesma mensagem + chips de sugestões de conversa
- 3ª insistência: mesma mensagem sem chips (evita repetição visual)

---

### T-011 — Chat — Bolha de Rate Limit Atingido

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="system"` |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-061, RF-062 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `ChatBubble` | `variant="system"` — centralizada, bg `--muted` | Mensagem de rate limit |
| `Badge` | `variant="secondary"`, atualizado em tempo real | Contador de tempo restante `hh:mm` |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | Não aplicável |
| **Empty** | Não aplicável |
| **Error** | Não aplicável |
| **Populated** | Mensagem com contador regressivo; input desabilitado (`opacity` transition 200ms) |

**Motion & Transições:**
- Input desabilita: `opacity 0.5` transition 200ms ao atingir limite
- Reabilita: `opacity 1.0` transition 200ms quando limite liberado
- `prefers-reduced-motion` → troca direta de opacity

**Regras específicas:**
- Contador atualizado a cada minuto (não tempo real para evitar re-renders)
- Input reabilitado automaticamente — sem ação do usuário necessária

---

## 4. Módulo B — Notificações Proativas (T-012 a T-014)

### T-012 — Bolha de Alerta Proativo — Nova Oportunidade
### T-013 — Bolha de Alerta Proativo — Prazo de Escrow
### T-014 — Bolha de Alerta Proativo — Mudança de Status

| Atributo | Valor |
|----------|-------|
| **Layout** | `ChatPanelLayout` — `ChatBubble variant="alert"` |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-097 (T-012), RF-098 (T-013), RF-099 (T-014) |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `ChatBubble` | `variant="alert"` — badge "Novo alerta" `--primary` no topo | Container do alerta proativo |
| `OportunidadeCard` | `variant="mini"` | Card compacto com dados da oportunidade (T-012) |
| `Badge` | `variant="default"`, bg `--primary` | "Novo alerta" no topo da bolha |
| `Button` | `variant="default"`, `size="sm"` | CTA de ação ("Ver oportunidade", "Ver negociação", "Ver status") |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | Não aplicável — bolhas inseridas assincronicamente |
| **Empty** | Não aplicável |
| **Error** | Não exibido ao usuário — sistema retenta via RabbitMQ; silencioso para o Cessionário |
| **Populated** | Card compacto + badge "Novo alerta" + CTA |

**Motion & Transições:**
- Bolha aparece com `slide-down` 250ms quando Cessionário abre o chat
- `prefers-reduced-motion` → `fade-in` 200ms

**Regras específicas (T-014):**
- Mudança de status: texto em linguagem clara (não código de status técnico) + próximo passo recomendado
- Status específicos de proposta: [DEFINIÇÃO PENDENTE — sem especificação dos strings de status do módulo Cessionário. Opção A: importar strings do módulo Cessionário. Opção B: categorias genéricas (Aceite, Recusa, Contraproposta). Responsável: Produto (BKL-PRD-006).]

---

## 5. Módulo C — Configurações do Usuário (T-015 a T-019)

### T-015 — Meu Perfil — Seção WhatsApp

| Atributo | Valor |
|----------|-------|
| **Layout** | `ProfileSectionLayout` — subseção dentro de "Meu Perfil" da plataforma |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-086, RF-092 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Skeleton` | `h-10 w-full` | Loading do status de vinculação |
| `Input` | `type="tel"`, `placeholder="(XX) XXXXX-XXXX"`, máscara automática | Campo de número de WhatsApp |
| `Button` | `variant="default"`, `size="default"` | "Vincular WhatsApp" (NaoVinculado) |
| `Button` | `variant="outline"`, `size="sm"`, `className="text-destructive"` | "Desvincular" (Ativo) |
| `Button` | `variant="outline"`, `size="sm"` | "Re-verificar agora" (Suspenso) |
| **`StatusBadge`** | Conforme mapeamento WhatsappBindingStatusEnum | Status atual da vinculação |
| `Alert` | `variant="destructive"`, inline abaixo do input | Erro de validação do número |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | `<Skeleton className="h-10 w-full">` para o bloco de status |
| **Empty** | Estado `nao_vinculado`: campo de número + botão "Vincular WhatsApp" |
| **Error** | Erro de validação: `<Alert variant="destructive">` inline abaixo do campo |
| **Populated** | Número mascarado `(XX) XXXXX-XXXX` + `StatusBadge` + botão contextual |

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Foco no Input ao renderizar estado NaoVinculado | Input focável via Tab |
| ARIA roles | Input com `aria-describedby` apontando para descrição de formato | Screen reader anuncia formato esperado |
| Skip links | Não aplicável | — |
| Contraste | StatusBadge respeita ratio ≥ 4.5:1 conforme mapeamento | OK |
| Motion | Status atualiza com `fade-in` 200ms; `prefers-reduced-motion` → sem animação | OK |

**Regras específicas:**
- Máscara automática no Input: `(XX) XXXXX-XXXX` aplicada progressivamente enquanto digita
- Validação de formato: em tempo real após perda de foco (blur event), não apenas no submit
- Offline: botão "Vincular" `disabled` com `<Tooltip>"Sem conexão"</Tooltip>`
- Número no estado Ativo exibido mascarado: `(XX) XXXXX-XXXX`

---

### T-016 — Modal — Fluxo de Vinculação WhatsApp

| Atributo | Valor |
|----------|-------|
| **Layout** | `OverlayLayout` — modal centrado 480px (desktop/tablet); full-screen bottom sheet (mobile < 768px) |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-086, RF-087, RF-088, RF-089 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Dialog` | `role="dialog"`, `aria-modal="true"` | Container do modal |
| `DialogHeader` | Default | Título do step ativo (`aria-labelledby`) |
| **`StepProgress`** (custom) | 3 steps, step ativo em `--primary`, concluído com ✓ | Indicador de progresso |
| `Input` | `type="tel"`, com máscara | Step 1: campo de número |
| `Button` | `variant="default"`, `size="lg"`, `full-width` | "Enviar código" (Step 1) |
| **`OTPInput`** (custom) | 6 campos individuais, `autocomplete="one-time-code"` | Step 2: entrada do OTP |
| `Badge` | `variant="secondary"` | Contador de tentativas restantes |
| `Button` | `variant="ghost"`, `size="sm"`, `disabled` com countdown | "Reenviar código" (Step 2) |
| `Alert` | `variant="destructive"` | Bloqueio por rate limit |
| `div` | `aria-live="assertive"` | Contador regressivo mm:ss durante bloqueio |
| `div` | `aria-live="polite"` | Contador de expiração OTP |
| `Spinner` de WhatsApp | Ícone animado WhatsApp | Step 3: aguardando confirmação |

> 💡 O ícone animado do WhatsApp no Step 3 é exceção ao padrão "sem spinners" — é um ícone de branding/comunicação, não um loading de dados. [DECISÃO AUTÔNOMA — uso do ícone animado do WhatsApp para comunicar visualmente que a ação ocorre no WhatsApp, diferenciando do loading técnico; alternativa descartada: Skeleton (sem sentido semântico para aguardar resposta no WhatsApp).]

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | Não aplicável — modal renderiza sincronicamente ao abrir |
| **Empty** | Step 1 inicial: Input de número vazio |
| **Error** | Número inválido / OTP incorreto: `<Alert variant="destructive">` inline; Bloqueio: overlay com countdown |
| **Populated** | Step ativo com inputs e ações contextuais |

**Transição entre steps:**
- Step 1 → 2 → 3: `slide-horizontal` (300ms, `ease-in-out`); conteúdo anterior `slide-out-left`, novo `slide-in-right`
- `prefers-reduced-motion` → `fade-in/out` 200ms

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Focus trap ativo; ao abrir: foco no primeiro input do step; ao fechar: foco retorna ao botão "Vincular WhatsApp" em T-015 | Navegação completa por teclado dentro do modal |
| ARIA roles | `role="dialog"`, `aria-modal="true"`, `aria-labelledby` → título do step ativo | Screen reader anuncia contexto correto |
| Skip links | Não aplicável (modal bloqueia foco externo) | — |
| Contraste | Todos os tokens respeitam ratio ≥ 4.5:1 | OK |
| Motion | Transição entre steps; `prefers-reduced-motion` → fade | OK |

**Regras específicas:**
- OTPInput: `aria-label="Dígito [N] de 6 do código de verificação"` em cada campo; `autocomplete="one-time-code"`
- Countdown bloqueio: `aria-live="assertive"` para anunciar estado urgente
- Countdown expiração OTP: `aria-live="polite"` (menos urgente)
- Edge case — fechar modal no Step 2 ou 3: `<AlertDialog>` de confirmação: "Tem certeza? O código enviado expirará e você precisará reiniciar o processo." — botões "Cancelar" e "Sair"

---

### T-017 — Modal — Confirmação de Desvinculação WhatsApp

| Atributo | Valor |
|----------|-------|
| **Layout** | `OverlayLayout` — modal centrado `max-w-sm` |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-092 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `AlertDialog` | Default | Container do modal de confirmação destrutiva |
| `AlertDialogTitle` | Default | "Desvincular WhatsApp" |
| `AlertDialogDescription` | Default, `--muted-foreground` | Aviso sobre perda de alertas |
| `AlertDialogCancel` | `variant="outline"` | "Cancelar" |
| `AlertDialogAction` | `variant="destructive"` | "Desvincular" |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | Não aplicável |
| **Empty** | Não aplicável |
| **Error** | Falha na desvinculação: `<Toast>` de erro + modal permanece aberto |
| **Populated** | Modal com aviso e botões de ação |

**Motion & Transições:**
- Modal fecha com `fade-out` 150ms
- Status em T-015 atualiza para `nao_vinculado` com `fade-in` 200ms após fechamento

---

### T-018 — Meu Perfil — Seção Notificações

| Atributo | Valor |
|----------|-------|
| **Layout** | `ProfileSectionLayout` |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-100 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Skeleton` | `h-12 w-full` × 3 | Loading dos 3 toggles |
| `Switch` | `checked/unchecked`, bg `--primary` (ativo) | Toggle de cada tipo de notificação |
| `Label` | `text-sm font-medium` | Nome da notificação |
| `p` | `text-sm text-muted-foreground` | Descrição inline de cada tipo |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | 3 `<Skeleton className="h-12 w-full">` |
| **Empty** | Não aplicável — sempre há 3 toggles |
| **Error** | Falha ao salvar: `<Toast variant="destructive">` "Não foi possível salvar a preferência." |
| **Populated** | 3 linhas: `<Switch>` + `<Label>` + `<p>` descritiva |

**Motion & Transições:**
- `<Switch>`: animação spring nativa do shadcn/ui (150ms)
- `prefers-reduced-motion` → sem animação de spring; toggle direta

**Regras específicas:**
- Offline: Switches `disabled` com `<Tooltip>"Sem conexão"</Tooltip>`
- Default: todos os 3 toggles ativos (`checked=true`) — opt-out explícito requerido

---

### T-019 — Meu Perfil — Seção Histórico de Chat

| Atributo | Valor |
|----------|-------|
| **Layout** | `ProfileSectionLayout` |
| **Role mínimo** | `cessionario` |
| **RFs** | RF-102 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Skeleton` | `h-16 w-full` × 5 | Loading da lista de conversas |
| `Card` | `radius-lg`, hover `bg --accent` | Linha de conversa na lista |
| `CardContent` | `flex justify-between`, truncate | Data + primeira mensagem truncada |
| `Button` | `variant="outline"`, `className="text-destructive border-destructive"`, ao final da lista | "Apagar tudo" |
| `AlertDialog` | Default | Modal de confirmação de exclusão |
| `AlertDialogAction` | `variant="destructive"` | "Confirmar exclusão" |
| `Toast` | `variant="default"` | Confirmação de solicitação enviada (5s) |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | 5 `<Skeleton className="h-16 w-full">` |
| **Empty** | `<EmptyState icon="message-square" message="Nenhuma conversa registrada ainda." cta={{ label: "Abrir chat", onClick: openChat }}>` |
| **Error** | `<Alert variant="destructive">` "Não foi possível carregar o histórico." |
| **Populated** | Lista de Cards de conversas + botão "Apagar tudo" no final |

**Motion & Transições:**
- Lista some com `fade-out` após confirmação de exclusão
- Toast "Exclusão solicitada. Você receberá confirmação em até 48 horas." persiste 5s
- `prefers-reduced-motion` → sem fade; troca direta de estado

**Regras específicas:**
- Primeira mensagem de cada conversa truncada em 60 caracteres com `...`
- Modal de confirmação: "Ação irreversível. O histórico será excluído em até 48 horas." — `AlertDialogDescription`

---

## 6. Módulo D — Admin — Supervisão IA (T-020 a T-022)

### T-020 — Painel de Supervisão IA — Listagem

| Atributo | Valor |
|----------|-------|
| **Layout** | `AppLayout` — sidebar fixa 280px + content area |
| **Role mínimo** | `admin` |
| **RFs** | RF-066, RF-067, RF-068, RF-069 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `DataTable` | Padrão §2.5; `TableSkeleton rows={10}` | Listagem de interações |
| `Skeleton` | `TableSkeleton` — 10 linhas | Loading |
| `EmptyState` | `icon="inbox"`, sem CTA | Empty state passivo (Admin) |
| `Alert` | `variant="destructive"` + retry | Erro ao carregar |
| `Select` | `variant="default"` | Filtros: período, canal, faixa de confiança |
| `DatePickerWithRange` | shadcn/ui | Filtro de período customizado |
| **`ConfidenceBadge`** | Conforme mapeamento (verde/amarelo/vermelho) | Badge de nível de confiança por interação |
| `Badge` | `variant="outline"` | Canal (webchat / whatsapp) |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | `<TableSkeleton rows={10}>` |
| **Empty** | `<EmptyState icon="inbox" message="Nenhuma interação registrada ainda.">` sem CTA |
| **Error** | `<Alert variant="destructive">` + botão "Tentar novamente" |
| **Populated** | Tabela com filtros + paginação 20 itens |

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Tab order: filtros → tabela → paginação | Navegação completa por teclado |
| ARIA roles | `role="table"`, `scope="col"` em headers, `aria-sort` em colunas ordenáveis | Screen reader lê corretamente |
| Skip links | `<a href="#main-content">` visível no primeiro Tab | Disponível |
| Contraste | `--foreground` sobre `--background` → 19.5:1 | OK |
| Motion | Nova interação: highlight 2s em `--accent`; `prefers-reduced-motion` → sem highlight | OK |

**Motion & Transições:**
- Nova interação no topo da lista: `animate-highlight` (bg `--accent`) 2s → `ease-out` ao desaparecer
- `prefers-reduced-motion` → sem highlight; inserção direta

**Regras específicas:**
- `ConfidenceBadge`: `aria-label="Confiança: [valor]% — [Baixa/Média/Alta]"`
- Paginação: 20 itens por página; setas de navegação
- Filtros persistem durante a sessão (localStorage ou estado do componente)
- Mobile: aviso "Use o painel Admin em telas maiores." + redirect sugerido para desktop [DECISÃO AUTÔNOMA]

---

### T-021 — Supervisão IA — Detalhe da Interação + Takeover

| Atributo | Valor |
|----------|-------|
| **Layout** | `AppLayout` — painel de detalhe full-width na content area |
| **Role mínimo** | `admin` |
| **RFs** | RF-071, RF-072, RF-073, RF-074 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Skeleton` | Painel de detalhe completo | Loading |
| `Card` | `radius-lg` | Container dos dados da interação |
| `Button` | `variant="default"`, `size="lg"`, bg `--primary` | "Assumir conversa" (quando elegível) |
| `Button` | `variant="outline"`, `size="sm"`, `disabled`, tooltip | "Assumir conversa" desabilitado (não elegível) |
| `Textarea` | `placeholder="Descreva o motivo do takeover..."`, obrigatório | Campo de motivo (aparece ao clicar Assumir) |
| `Select` | Opções: `TakeoverReasonEnum` | Categoria do motivo |
| `Alert` | `variant="default"`, bg `--destructive`, texto `--destructive-foreground`, `sticky top-0` | Banner "Você está controlando esta conversa" |
| `ScrollArea` | Histórico de mensagens lado esquerdo | Painel de histórico durante takeover |
| `Textarea` | Lado direito, input do Admin | Campo de resposta durante takeover |
| `Button` | `variant="destructive"`, `size="lg"` | "Encerrar takeover" |
| `Toast` | `variant="destructive"` | Erro no takeover / takeover concorrente |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | `<Skeleton>` cobrindo o painel de detalhe |
| **Empty** | Não aplicável — tela só é acessada via click em T-020 |
| **Error** | `<Toast variant="destructive">` "Não foi possível assumir a conversa." |
| **Populated** | Dados da interação + botão contextual (com ou sem takeover disponível) |

**Hierarquia visual durante takeover obrigatória:**
1. **Banner "Você está controlando esta conversa"** — `sticky top-0`, bg `--destructive`, texto `--destructive-foreground`, atenção máxima
2. **Histórico de mensagens** — painel esquerdo, `ScrollArea`, `flex-1`
3. **Input do Admin** — painel direito, posição primária (bottom-right, maior largura)
4. **Botão "Encerrar takeover"** — `variant="destructive"`, destaque visual, ação primária destrutiva

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Ao iniciar takeover: foco no `Textarea` do Admin | Pronto para digitar imediatamente |
| ARIA roles | Botão Assumir: `aria-describedby` → tooltip de elegibilidade | Screen reader descreve por que está disponível ou não |
| Skip links | `<a href="#main-content">` | Disponível |
| Contraste | Banner destructive: texto `--destructive-foreground` sobre `--destructive` → ratio verificado | OK |
| Motion | Takeover ativa: `slide-in-right` 300ms do painel de input; `prefers-reduced-motion` → fade | OK |

**Regras específicas:**
- Botão "Assumir" desabilitado se confiança ≥ threshold: `<Tooltip>"Confiança acima do threshold configurado"</Tooltip>`
- Campo de motivo (`Textarea`) + `Select` de categoria obrigatórios antes de confirmar takeover
- Takeover concorrente: `<Toast>` "Esta conversa já está sendo controlada por outro membro da equipe." → botão permanece `disabled`

---

### T-022 — Dashboard Admin — Métricas do Repasse AI

| Atributo | Valor |
|----------|-------|
| **Layout** | `AppLayout` — grid de métricas na content area |
| **Role mínimo** | `admin` |
| **RFs** | RF-076, RF-077 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|
| `Skeleton` | `h-32 w-full` × 5 + `h-64 w-full` | Loading de 5 cards + gráfico |
| **`MetricCard`** (custom) | Volume (lg, destaque), CSAT (md, destaque), Takeover/Recusa (sm), Latência (sm, informativo) | Cards de KPI |
| `AreaChart` / `LineChart` | shadcn/ui charts, usando `--chart-1` a `--chart-5` | Volume de interações por dia |
| `Tabs` | `variant="default"` | Filtro de período: 7d, 30d, 90d, Custom |
| `DatePickerWithRange` | Default | Período customizado |
| `EmptyState` | Dois estados distintos (ver abaixo) | Empty por período e First-use |

**Estados obrigatórios:**

| Estado | Implementação |
|--------|--------------|
| **Skeleton** | 5 `<Skeleton className="h-32">` + 1 `<Skeleton className="h-64">` |
| **Empty (sem dados no período)** | Cards com `"--"` + `<Tooltip>"Sem dados para o período selecionado."</Tooltip>` |
| **Empty (first-use)** | Cards com `"--"` + `<EmptyState>` no gráfico: `"Ainda não há interações registradas."` + CTA opcional `"Ver documentação"` |
| **Error** | Banner `<Alert variant="destructive">` + cards em estado degradado |
| **Populated** | 5 MetricCards + AreaChart |

> 💡 Os estados Empty (período) e Empty (first-use) são distintos: first-use inclui mensagem orientativa para o Admin entender o que esperar. [DECISÃO AUTÔNOMA — contexto diferente exige comunicação diferente; alternativa descartada: mesmo estado (confunde Admin sobre status do produto).]

**Hierarquia visual dos MetricCards obrigatória:**
1. **Volume de interações** — card primário: maior tamanho (`col-span-2`), posição top-left
2. **CSAT** — card de destaque: tamanho médio, posição top-right
3. **Taxa de takeover + Taxa de recusa** — cards secundários: tamanho padrão
4. **Latência média** — card informativo: menor destaque, texto `--muted-foreground`
5. **AreaChart** — ocupa metade inferior do painel

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Tab order: Tabs de período → DatePicker → MetricCards → Chart | Navegação completa |
| ARIA roles | Tabs com `role="tablist"`, `role="tab"`, `aria-selected` | Screen reader navega tabs |
| Skip links | `<a href="#main-content">` | Disponível |
| Contraste | Todos os tokens `--chart-*` com ratio ≥ 3:1 (elementos gráficos) | OK |
| Motion | MetricCards: `number-flip` 300ms ao trocar período; `prefers-reduced-motion` → troca direta | OK |

**Motion & Transições:**
- MetricCards: animação `number-flip` (counter animation) 300ms ao trocar filtro de período
- Chart re-render: `fade-in` 200ms do novo dataset
- `prefers-reduced-motion` → todos os valores trocam diretamente sem animação

---

## 7. Cross-References

| Documento | Relação com este documento |
|-----------|---------------------------|
| **01 - Regras de Negócio** | RNs de domínio que definem estados visuais: quotas (T-011), takeover threshold (T-020/T-021), OTP rate limit (T-016), retenção 90 dias (T-019) |
| **03 - Brand Theme Guide** | Fonte única de tokens CSS: `--primary`, `--destructive`, `--muted`, `--chart-1` a `--chart-5`, `--radius-*` |
| **06 - Mapa de Telas** | IDs T-001 a T-022, rotas, RFs, roles mínimos e estados detalhados por tela |
| **07 - Wireframes** | Layout visual de referência ASCII para hierarquia de componentes por tela |
| **08 - UX Writing** | Labels, placeholders, mensagens de erro, CTAs e vocabulário controlado aplicados em todos os componentes |
| **11 - Mobile** | Consome os contratos como referência de adaptação mobile por tela |
| **15 - Arquitetura de Pastas** | Mapeia onde cada tela vive em `features/<módulo>/pages/` |
| **28 - Checklist de Qualidade** | Valida conformidade de componentes, estados e tokens por tela |

---

## 8. Changelog

| Versão | Data | Autor | Descrição |
|--------|------|-------|-----------|
| v1.0 | 22/03/2026 | Claude Code Desktop — Pipeline ShiftLabs v9.5 | Criação. 22 telas cobertas (T-001 a T-022), 4 módulos, 7 variantes de ChatBubble, StatusBadge mappings completos para 4 enums, componentes críticos de domínio documentados. |

---

## 9. Backlog de Pendências

| Item | Tipo | Tela / Módulo | Impacto | Justificativa | Dono | Status |
|------|------|---------------|---------|---------------|------|--------|
| Layout ChatPanelLayout não padrão genérico | [DECISÃO AUTÔNOMA] | T-001 a T-014 | P1 | Repasse AI é módulo embutido no frontend Cessionário — layout de painel lateral é a arquitetura correta; alternativa FullPageLayout descartada | Frontend | Aplicado |
| Ícone animado WhatsApp no Step 3 (T-016) | [DECISÃO AUTÔNOMA] | T-016 | P2 | Exceção ao padrão "sem spinners" — ícone de branding/comunicação, não loading técnico; alternativa Skeleton descartada | Frontend / Design | Aplicado |
| Empty first-use vs. empty por período (T-022) | [DECISÃO AUTÔNOMA] | T-022 | P1 | Dois estados distintos com comunicação diferente; alternativa único estado descartada | Frontend | Aplicado |
| Admin não otimizado para mobile (T-020, T-021, T-022) | [DECISÃO AUTÔNOMA] | T-020–T-022 | P2 | Tabelas densas incompatíveis com mobile; aviso + redirect sugerido; alternativa layout colapsado descartada | Frontend | Aplicado |
| Status específicos de proposta em T-014 (StatusBadge de mudança de status) | [DEFINIÇÃO PENDENTE] | T-014 | P1 | Opção A: strings do módulo Cessionário; Opção B: categorias genéricas (Aceite/Recusa/Contraproposta). Bloqueado: Produto (BKL-PRD-006) | Produto | Pendente |

---

*Contratos de UI v1.0 concluído. Cobertura: 22 telas, 4 módulos, 0 pendências críticas. Status: APROVADO. Próximo documento do pipeline: D14 — Especificações Técnicas.*
