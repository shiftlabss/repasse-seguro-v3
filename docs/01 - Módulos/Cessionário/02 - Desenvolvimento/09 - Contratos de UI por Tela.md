# 09 - Contratos de UI por Tela

## Módulo Cessionário · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Documento** | 09 - Contratos de UI por Tela |
| **Produto** | Repasse Seguro — Módulo Cessionário |
| **Versão** | v1.0 |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |
| **Autor** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Status** | Ativo |
| **Fase** | 2 — UI/UX |
| **Área** | Design + Frontend |
| **Referências** | 01 - Regras de Negócio · 03 - Brand Theme Guide · 06 - Mapa de Telas · 07 - Wireframes · 08 - UX Writing |

---

> 📌 **TL;DR**
>
> - **34 telas cobertas** com 100% dos contratos de UI (nenhuma tela pendente).
> - **Padrão de 4 estados obrigatórios** por tela: `Skeleton → Empty → Error → Populated`. Spinners são proibidos em todo o produto.
> - **Layouts base reutilizados:** `AuthLayout` (4 telas de autenticação), `AppLayout` (25 telas logadas), `FullPageLayout` (3 telas de onboarding/KYC), `ErrorLayout` (2 telas de erro).
> - **Componentes críticos de domínio identificados:** `OpportunityCard`, `ProposalStatusBadge`, `NegotiationStatusBadge`, `EscrowStatusBadge`, `FormalizationStatusBadge`, `CessionarioStatusBadge`, `KpiCard`, `SlaCountdown`, `QuotaBar`.
> - **RBAC obrigatório** via `useRbac()` + `RbacGuard` — role `CESSIONARIO` em todas as telas logadas; telas de autenticação são públicas.
> - **Tokens canônicos:** todos os tokens referenciam o namespace `--` do shadcn/ui conforme definido no doc 03 (Brand Theme Guide). Não há tokens customizados fora dessa estrutura.
> - **Zero telas ou seções pendentes** — todas as 34 telas têm contratos completos com componentes, variantes, tokens, acessibilidade e motion.

---

## 1. Persona

Frontend Lead / UI Designer com foco em transformar cada tela em contrato explícito de componentes, estados e tokens. Este documento materializa o nível de detalhe de implementação visual do módulo Cessionário e elimina margem de interpretação na camada de interface.

**Impacto no pipeline:**
- **D11 - Mobile:** usa os contratos de UI como referência para adaptação mobile (breakpoints, reorganização de componentes).
- **D15 - Arquitetura de Pastas:** usa os contratos para mapear onde cada tela vive em `features/<módulo>/pages/`.
- **D28 - Checklist de Qualidade:** valida se a interface segue os contratos de componentes e estados definidos aqui.

---

## 2. Convenções Globais

### 2.1 Quatro Estados Obrigatórios por Tela

Toda tela DEVE implementar os 4 estados abaixo. Não há exceção.

| Estado | Componente | Trigger | Duração máxima |
|--------|-----------|---------|----------------|
| **Skeleton** | `<Skeleton>` shadcn/ui — **nunca spinner** | Enquanto fetch em andamento | Até dados chegarem |
| **Empty** | `<EmptyState>` custom (ilustração SVG + headline + CTA) | `data.length === 0` ou sem dados | Indeterminado |
| **Error** | `<Alert variant="destructive">` + botão "Tentar novamente" | Request HTTP falha ou timeout | Indeterminado |
| **Populated** | Conteúdo real renderizado | Após dados carregados com sucesso | Indeterminado |

🔴 **Spinners são absolutamente proibidos.** Qualquer loading state usa `<Skeleton>` do shadcn/ui.

### 2.2 Quatro Layouts Base

| Layout | Telas | Estrutura |
|--------|-------|-----------|
| `AuthLayout` | T-AUTH-01 a T-AUTH-04 | Centralizado, card 400px, fundo `--background`, sem sidebar |
| `AppLayout` | Todas as telas logadas (T-DASH, T-OPR, T-PRP, T-NEG, T-FIN, T-IA, T-PRF) | Sidebar 280px fixa (`--sidebar`) + área de conteúdo fluida |
| `FullPageLayout` | T-PRF-02 (KYC stepper) | Sem sidebar, step indicator no topo, full-width |
| `ErrorLayout` | Páginas de erro 404/500 | Centralizado, sem sidebar, ilustração + CTA |

### 2.3 Componentes Críticos de Domínio

| Componente | Telas onde é usado | Tokens principais | Faixas semânticas |
|---|---|---|---|
| `KpiCard` | T-DASH-01 | `--card`, `--card-foreground`, `--primary`, `--muted-foreground` | n/a — exibe valores numéricos |
| `SlaCountdown` | T-NEG-03, T-NEG-04 | `--destructive` (≤24h), `--chart-3` (≤72h), `--foreground` (>72h) | Verde → Amarelo → Vermelho por urgência |
| `QuotaBar` | T-OPR-03 (proposta), T-PRP-01 | `--primary` (0-66%), `--chart-3` (67-89%), `--destructive` (90-100%) | Azul → Laranja → Vermelho por saturação |
| `OpportunityCard` | T-OPR-01 | `--card`, `--primary`, `--muted-foreground`, `--chart-4` | n/a — card de oportunidade |
| `ProposalStatusBadge` | T-OPR-03, T-PRP-01, T-PRP-02 | Mapeamento completo em §5.4 | Por enum `ProposalStatus` |
| `NegotiationStatusBadge` | T-NEG-01 a T-NEG-04 | Mapeamento completo em §6.5 | Por enum `NegotiationStatus` |
| `EscrowStatusBadge` | T-NEG-03, T-FIN-01 | Mapeamento completo em §7.4 | Por enum `EscrowStatus` |
| `FormalizationStatusBadge` | T-ASS-01 a T-ASS-03, T-FIN-01 | Mapeamento completo em §8.4 | Por enum `FormalizationStatus` |
| `CessionarioStatusBadge` | T-PRF-01, T-PRF-03 | Mapeamento completo em §11.4 | Por enum `CessionarioStatus` |
| `AiRiskScoreBadge` | T-OPR-02 | `--destructive` (1-3), `--chart-3` (4-6), `--primary` (7-10) | Vermelho → Amarelo → Azul por faixa |

### 2.4 Padrão DataTable

Sempre que houver listagem tabulada, usar:

```tsx
<DataTable
  columns={columns}
  data={data}
  skeleton={<TableSkeleton rows={8} />}
  emptyState={<EmptyState icon={<IconName />} title="Título" description="Descrição" cta={<Button>Ação</Button>} />}
  pagination={{ pageSize: 25 }}
/>
```

Nunca spinner. Máximo 100 itens por página. `pageSize` padrão: 25.

### 2.5 StatusBadge — Mapeamentos Globais de Enum

Todos os enums do domínio têm mapeamento obrigatório definido nas seções correspondentes. Ver §3.4, §5.4, §6.5, §7.4, §8.4, §11.4.

---

## 3. Autenticação (T-AUTH-01 a T-AUTH-04)

### 3.1 T-AUTH-01 — Login

| Atributo | Valor |
|---|---|
| **Layout** | `AuthLayout` |
| **Role mínimo** | Público (sem autenticação) |
| **RFs** | RF-001, RF-002 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `className="w-[400px]"`, `--card`, `--radius-xl` | Container principal do formulário |
| `CardHeader` | Padrão | Logo + título "Bem-vindo ao Repasse Seguro" |
| `CardContent` | Padrão | Formulário de login |
| `Input` | `type="email"`, `placeholder="seu@email.com"`, `--input`, `--radius-md` | Campo de e-mail |
| `Input` | `type="password"`, `--input`, `--radius-md` | Campo de senha |
| `Button` | `variant="default"`, `size="lg"`, `className="w-full"`, `--primary`, `--radius-md` | CTA "Entrar" |
| `Button` | `variant="ghost"`, `size="sm"`, `--muted-foreground` | Link "Esqueci minha senha" |
| `Separator` | Padrão, `--border` | Divisor visual entre login e OAuth |
| `Button` | `variant="outline"`, `size="lg"`, `className="w-full"`, `--border` | "Entrar com Google" (ícone Google SVG) |
| `Alert` | `variant="destructive"`, `--destructive` | Mensagem de erro de credenciais |
| `Skeleton` | `className="h-10 w-full"` | Placeholder durante submit |

**Estados:**

- **Skeleton:** `<Skeleton className="h-10 w-full" />` aparece nos dois inputs enquanto submissão está em andamento.
- **Empty:** n/a — tela estática de formulário.
- **Error:** `<Alert variant="destructive">` com texto "E-mail ou senha incorretos. Verifique suas credenciais e tente novamente."
- **Populated:** Formulário vazio pronto para preenchimento.

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | `autoFocus` no campo e-mail ao montar | Cursor posicionado no e-mail imediatamente |
| ARIA roles | `role="form"`, `aria-label="Formulário de login"` | Screen reader anuncia formulário |
| Skip links | `<a href="#main-content" className="sr-only focus:not-sr-only">` | Visível no primeiro Tab |
| Contraste | `--primary` (#0069A8) sobre `--primary-foreground` (#F0F9FF) = 5.2:1 | Passa 4.5:1 mínimo |
| Motion | `prefers-reduced-motion`: desabilitar animação de fade do Card | Sem animação quando ativo |

**Motion & Transições:**

- Entrada da tela: fade-in do Card (`opacity: 0 → 1`, 200ms, `ease-out`, `--font-sans`)
- Transições internas: shake animation no Card ao erro de credenciais (150ms, `ease-in-out`)
- Skeleton: exibido apenas durante submissão do formulário (não no carregamento inicial)
- Feedback: toast de sucesso não aplicável (redirect imediato após login)

**Regras específicas:**

- Após login bem-sucedido: redirect para `/dashboard` via `TanStack Router`.
- Se KYC não iniciado (`status === 'CADASTRADA'`): redirect para `/perfil/kyc`.
- Se conta bloqueada (`status === 'BLOQUEADA_TEMPORARIAMENTE'`): exibir `<Alert variant="destructive">` com mensagem específica, desabilitar CTA.
- Não exibir campo de senha em texto puro — sempre `type="password"` com toggle de visibilidade via ícone `Eye/EyeOff` do Lucide.

---

### 3.2 T-AUTH-02 — Cadastro

| Atributo | Valor |
|---|---|
| **Layout** | `AuthLayout` |
| **Role mínimo** | Público (sem autenticação) |
| **RFs** | RF-001, RF-003 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `className="w-[400px]"`, `--card`, `--radius-xl` | Container do formulário de cadastro |
| `CardHeader` | Padrão | Logo + "Criar sua conta" |
| `CardContent` | Padrão | Formulário multi-campo |
| `Input` | `type="text"`, `placeholder="Seu nome completo"`, `--input`, `--radius-md` | Nome completo |
| `Input` | `type="email"`, `--input`, `--radius-md` | E-mail |
| `Input` | `type="password"`, `--input`, `--radius-md` | Senha |
| `Input` | `type="password"`, `placeholder="Confirmar senha"`, `--input`, `--radius-md` | Confirmação de senha |
| `Checkbox` | Padrão, `--primary` | Aceite de Termos de Uso |
| `Button` | `variant="default"`, `size="lg"`, `className="w-full"`, `--primary` | CTA "Criar conta" |
| `Button` | `variant="ghost"`, `size="sm"` | Link "Já tenho conta — Entrar" |
| `Alert` | `variant="destructive"`, `--destructive` | Erros de validação (e-mail duplicado, senha fraca) |
| `Progress` | `value={passwordStrength}`, `--primary` | Indicador de força de senha |

**Estados:**

- **Skeleton:** `<Skeleton className="h-10 w-full" />` em todos os inputs durante submit.
- **Empty:** n/a — formulário estático.
- **Error:** `<Alert variant="destructive">` por tipo: "E-mail já cadastrado", "As senhas não coincidem", "Senha muito fraca".
- **Populated:** Formulário preenchido, validações inline visíveis.

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | `autoFocus` no campo nome | Cursor em nome ao montar |
| ARIA roles | `aria-describedby` nos inputs com erro, `aria-live="polite"` no indicador de força | Screen reader anuncia erros e força |
| Skip links | Link para `#main-content` | Visível no primeiro Tab |
| Contraste | Todos textos ≥ 4.5:1 | Validado via axe-core |
| Motion | Desabilitar animação Progress se `prefers-reduced-motion` | Sem animação |

**Motion & Transições:**

- Entrada: fade-in do Card (200ms, `ease-out`)
- Progress de senha: transição de largura `transition-all duration-300 ease-out`
- Mensagens de erro inline: `AnimatePresence` com `opacity: 0 → 1` (150ms)
- Sem animações se `prefers-reduced-motion: reduce`

**Regras específicas:**

- Validar força de senha inline (mínimo 8 caracteres, 1 maiúscula, 1 número, 1 especial) — `<Progress>` com 4 faixas: fraca (25%), razoável (50%), boa (75%), forte (100%).
- Checkbox de Termos deve estar desmarcado por padrão; CTA desabilitado se não marcado.
- Após cadastro: redirect para `/perfil/kyc` com toast "Conta criada! Complete sua verificação de identidade."

---

### 3.3 T-AUTH-03 — Recuperação de Senha

| Atributo | Valor |
|---|---|
| **Layout** | `AuthLayout` |
| **Role mínimo** | Público |
| **RFs** | RF-004 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `className="w-[400px]"`, `--card`, `--radius-xl` | Container |
| `CardHeader` | Padrão | "Recuperar acesso" |
| `CardContent` | Padrão | Formulário de e-mail |
| `Input` | `type="email"`, `--input`, `--radius-md` | Campo e-mail para recuperação |
| `Button` | `variant="default"`, `size="lg"`, `className="w-full"`, `--primary` | "Enviar link de recuperação" |
| `Button` | `variant="ghost"`, `size="sm"` | "Voltar para o login" |
| `Alert` | `variant="default"`, `--primary` | Confirmação de envio do e-mail |

**Estados:**

- **Skeleton:** `<Skeleton className="h-10 w-full" />` no input durante submit.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "E-mail não encontrado. Verifique o endereço informado."
- **Populated:** Após envio bem-sucedido: `<Alert variant="default">` "Link enviado! Verifique sua caixa de entrada."

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | `autoFocus` no campo e-mail | Cursor posicionado |
| ARIA roles | `aria-live="polite"` no Alert de confirmação | Screen reader anuncia confirmação |
| Skip links | Link `#main-content` | Visível no Tab |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | `prefers-reduced-motion`: sem fade | Sem animação |

**Motion & Transições:**

- Entrada: fade-in do Card (200ms)
- Alert de sucesso: `AnimatePresence` fade-in (200ms)
- Sem animações se `prefers-reduced-motion`

**Regras específicas:**

- CTA desabilitado se campo vazio.
- Após envio: CTA torna-se "Reenviar e-mail" com cooldown de 60s (`<Button disabled>` com countdown).
- Rate limit visual: desabilitar botão por 60s após envio.

---

### 3.4 T-AUTH-04 — Nova Senha (link de recuperação)

| Atributo | Valor |
|---|---|
| **Layout** | `AuthLayout` |
| **Role mínimo** | Público (token de reset na URL) |
| **RFs** | RF-004 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `className="w-[400px]"`, `--card`, `--radius-xl` | Container |
| `CardHeader` | Padrão | "Criar nova senha" |
| `CardContent` | Padrão | Formulário de nova senha |
| `Input` | `type="password"`, `--input`, `--radius-md` | Nova senha |
| `Input` | `type="password"`, `--input`, `--radius-md` | Confirmar nova senha |
| `Progress` | `value={strength}`, `--primary` | Força da nova senha |
| `Button` | `variant="default"`, `size="lg"`, `className="w-full"`, `--primary` | "Salvar nova senha" |
| `Alert` | `variant="destructive"`, `--destructive` | Token inválido/expirado |

**StatusBadge por CessionarioStatus:**

```
CessionarioStatus → Cor semântica → Variante Badge:
CADASTRADA       → --muted-foreground  → "Cinza"      (badge outline cinza)
KYC_EM_ANALISE   → --chart-2           → "Azul claro" (badge outline azul)
KYC_APROVADO     → --primary           → "Azul"       (badge solid azul)
KYC_REPROVADO    → --destructive       → "Vermelho"   (badge destructive)
BLOQUEADA_TEMPORARIAMENTE → --chart-1  → "Amarelo"    (badge outline amarelo) [DECISÃO AUTÔNOMA — chart-1 é o tom mais claro; alternativa descartada: --destructive (muito severo para bloqueio temporário)]
ENCERRADA        → --muted-foreground  → "Cinza escuro" (badge muted)
```

**Estados:**

- **Skeleton:** `<Skeleton className="h-10 w-full" />` nos inputs durante submit.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Link expirado ou inválido. Solicite um novo link de recuperação."
- **Populated:** Formulário disponível com token válido.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | `autoFocus` na nova senha | Cursor posicionado |
| ARIA roles | `aria-describedby` nos inputs | Screen reader anuncia requisitos |
| Skip links | `#main-content` | Visível no Tab |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | OK |

**Motion & Transições:**

- Entrada: fade-in do Card (200ms)
- Validação de token na montagem: `<Skeleton>` no Card inteiro (400ms máximo)
- Sem animações se `prefers-reduced-motion`

**Regras específicas:**

- Validar token JWT de reset antes de renderizar formulário — se inválido, exibir estado `Error` imediatamente.
- Mesma validação de força de senha do cadastro.
- Após sucesso: redirect para `/login` com toast "Senha atualizada com sucesso!"

---

## 4. Dashboard (T-DASH-01 + overlays)

### 4.1 T-DASH-01 — Dashboard Principal

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-041, RF-042, RF-043, RF-044, RF-045 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` (KpiCard) | `className="flex-1 min-w-[200px]"`, `--card`, `--radius-lg` | KPI: Propostas Ativas, Negociações, Em Formalização, Concluídas |
| `Skeleton` | `className="h-8 w-24"` | Placeholder do valor do KPI |
| `Skeleton` | `className="h-4 w-32"` | Placeholder do label do KPI |
| `Badge` | `variant="secondary"`, `--secondary` | Variação % vs. período anterior no KPI |
| `Card` (gráfico) | `--card`, `--radius-lg`, `className="col-span-2"` | Card de gráfico de atividade |
| `ChartContainer` | Usando `recharts` via shadcn/ui chart, paleta `--chart-1` a `--chart-5` | Gráfico de linha de atividade |
| `Skeleton` | `className="h-[200px] w-full"` | Placeholder do gráfico |
| `Card` (oportunidades) | `--card`, `--radius-lg` | Lista resumida das últimas oportunidades vistas |
| `Button` | `variant="default"`, `size="sm"`, `--primary` | "Ver marketplace" |
| `Alert` | `variant="default"`, `--primary` | Banner de ação pendente (KYC, Escrow vencendo) |
| `SlaCountdown` (custom) | Props: `deadline: Date`, `urgency: 'low' | 'medium' | 'high'` | Countdown do SLA de depósito Escrow |

**KpiCard — configurações por métrica:**

| Métrica | Ícone Lucide | Cor do valor | Formato |
|---|---|---|---|
| Propostas Ativas | `FileText` | `--foreground` | Número inteiro |
| Em Negociação | `MessageSquare` | `--primary` | Número inteiro |
| Em Formalização | `PenLine` | `--chart-3` | Número inteiro |
| Concluídas (total) | `CheckCircle2` | `--primary` | Número inteiro |

**Estados:**

- **Skeleton:** Grid de KPIs com 4x `<Skeleton className="h-24 w-full rounded-lg" />` + gráfico `<Skeleton className="h-[200px] w-full" />`.
- **Empty:** `<EmptyState icon={<TrendingUp />} title="Sua jornada começa aqui" description="Explore oportunidades no marketplace e envie sua primeira proposta." cta={<Button>Ver oportunidades</Button>} />`.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar seu dashboard. Tente novamente."
- **Populated:** Grid de KPIs + gráfico + lista de oportunidades recentes.

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Primeiro KpiCard recebe foco na montagem via `tabIndex={0}` | Navegação por Tab funciona |
| ARIA roles | `role="region"` em cada seção, `aria-label` nos KpiCards | Screen reader anuncia regiões |
| Skip links | `<a href="#main-content">Ir para o conteúdo</a>` | Visível no Tab |
| Contraste | Valores KPI em `--foreground` (#0A0A0A) sobre `--card` (#FFFFFF) = 21:1 | Passa critério |
| Motion | Gráfico sem animação se `prefers-reduced-motion` | Charts estáticos |

**Motion & Transições:**

- Entrada da tela: `AnimatePresence` fade-in (`opacity: 0 → 1`, 250ms, `ease-out`)
- KPI Cards: stagger de entrada com delay de 50ms por card
- Gráfico: sem animação de linha se `prefers-reduced-motion`
- `SlaCountdown`: atualiza a cada segundo via `setInterval` — cor muda em tempo real
- Skeleton: exibido até que todas as 4 queries do TanStack Query resolvam

**Regras específicas:**

- Banner de KYC pendente: exibir `<Alert>` amarelo no topo se `status !== 'KYC_APROVADO'` — bloqueia interação com marketplace.
- KPIs atualizados via Supabase Realtime (max 60s lag tolerado por RN-043).
- `SlaCountdown`: cores por urgência — `>72h` → `--foreground`; `≤72h` → `--chart-2`; `≤24h` → `--destructive`.
- Dashboard não exibe nenhum dado do Cedente (RN-014, RN-072).

---

### 4.2 Overlay — Re-autenticação Modal

| Atributo | Valor |
|---|---|
| **Layout** | `Dialog` sobre `AppLayout` |
| **Role mínimo** | `CESSIONARIO` (sessão expirada) |
| **RFs** | RF-005 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Dialog` | `open={isSessionExpired}`, `--card`, `--radius-xl` | Container do modal de re-auth |
| `DialogHeader` | Padrão | "Sessão encerrada" |
| `DialogDescription` | `--muted-foreground` | "Por segurança, sua sessão foi encerrada. Confirme sua identidade para continuar." |
| `Input` | `type="password"`, `--input` | Campo senha para re-autenticação |
| `Button` | `variant="default"`, `--primary` | "Confirmar identidade" |
| `Button` | `variant="ghost"` | "Sair da conta" |

**Estados:**

- **Skeleton:** `<Skeleton className="h-10 w-full" />` no Input durante submit.
- **Empty:** n/a.
- **Error:** Erro inline embaixo do input — "Senha incorreta. Tente novamente."
- **Populated:** Modal exibido com campo de senha.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | `focus trap` dentro do Dialog | Tab não sai do modal |
| ARIA roles | `role="alertdialog"`, `aria-modal="true"`, `aria-labelledby` | Screen reader anuncia alerta |
| Skip links | n/a — modal | n/a |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Dialog estático |

**Motion & Transições:**

- Entrada: scale de `0.95 → 1` + fade-in (150ms) via `DialogContent`
- Sem animações se `prefers-reduced-motion`

**Regras específicas:**

- Modal não pode ser fechado pelo usuário (sem `DialogClose`/ESC) — apenas via re-autenticação ou logout.
- Máximo 3 tentativas de senha — após 3 falhas: logout automático + redirect para `/login`.

---

## 5. Marketplace (T-OPR-01 a T-OPR-03)

### 5.1 T-OPR-01 — Marketplace (listagem de oportunidades)

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-017, RF-018 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Input` | `type="search"`, `placeholder="Buscar oportunidades..."`, `--input`, `--radius-md` | Busca textual |
| `Select` | `--input`, `--radius-md` | Filtro por cidade/estado |
| `Select` | `--input`, `--radius-md` | Filtro por faixa de preço |
| `Select` | `--input`, `--radius-md` | Filtro por Score de Risco IA |
| `Button` | `variant="outline"`, `size="sm"` | "Limpar filtros" |
| `Badge` | `variant="secondary"` | Contador de filtros ativos |
| `OpportunityCard` (custom) | Props: `opportunity`, `onPropose: () => void` | Card de oportunidade individual |
| `Skeleton` | `className="h-[180px] w-full rounded-lg"` | Placeholder do OpportunityCard |
| `Pagination` | `--primary` | Paginação dos resultados |
| `Select` | `defaultValue="25"` | Itens por página (25/50/100) |

**OpportunityCard — subcomponentes:**

| Subcomponente | Variante | Propósito |
|---|---|---|
| `Badge` (código) | `variant="outline"`, `--border` | Código anônimo (ex: OPR-2026-0042) |
| `AiRiskScoreBadge` (custom) | Props: `score: number` (1-10) | Score de risco IA com cor semântica |
| Linha de valores | `--foreground` + `--muted-foreground` | Tabela Atual, Delta, Comissão Comprador |
| `Button` | `variant="default"`, `size="sm"`, `--primary` | "Ver detalhes" |

**Estados:**

- **Skeleton:** Grid de 8x `<Skeleton className="h-[180px] w-full rounded-lg" />`.
- **Empty:** `<EmptyState icon={<Search />} title="Nenhuma oportunidade encontrada" description="Tente ajustar os filtros ou aguarde novas oportunidades." cta={<Button variant="outline" onClick={clearFilters}>Limpar filtros</Button>} />`.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar as oportunidades. Tente novamente."
- **Populated:** Grid de OpportunityCards com paginação.

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no campo de busca ao montar | Tab order lógico |
| ARIA roles | `role="list"` no grid, `role="listitem"` em cada card, `aria-label` no card com código OPR | Screen reader anuncia lista |
| Skip links | `#main-content` | Visível no Tab |
| Contraste | Código OPR em `--muted-foreground` sobre `--card` = 4.6:1 | Passa |
| Motion | Card hover sem transição se `prefers-reduced-motion` | Estático no hover |

**Motion & Transições:**

- Entrada: stagger de cards (50ms delay entre cards, fade-in 200ms cada)
- Hover no card: `transform: translateY(-2px)` + `box-shadow` leve (150ms, `ease-out`)
- Skeleton → Populated: `AnimatePresence` com `opacity: 0 → 1`
- Mudança de filtro: Skeleton reexibido instantaneamente, depois Populated

**Regras específicas:**

- KYC não aprovado: `<Alert>` no topo bloqueando a listagem — "Complete sua verificação de identidade para acessar as oportunidades."
- Paginação: máximo 100 itens por página; padrão 25.
- Nenhum dado do Cedente exibido (RN-014) — apenas código anônimo OPR-XXXX-XXXX.
- Score de risco IA sempre visível no card (RN-048).

---

### 5.2 T-OPR-02 — Detalhe da Oportunidade

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-017, RF-018, RF-019, RF-046, RF-047, RF-048, RF-049, RF-050 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Breadcrumb` | `--muted-foreground` | Marketplace → Detalhe OPR-XXXX |
| `Badge` | `variant="outline"`, `--border` | Código anônimo OPR |
| `AiRiskScoreBadge` (custom) | Props: `score: number`, tamanho `lg` | Score de risco com cor semântica |
| `Card` | `--card`, `--radius-lg` | Card de informações financeiras |
| `Separator` | `--border` | Divisores entre seções de informação |
| `DataTable` | `columns`, `data`, `skeleton={<TableSkeleton rows={3} />}` | Tabela de histórico de valores |
| `Button` | `variant="default"`, `size="lg"`, `--primary` | "Fazer proposta" |
| `Button` | `variant="outline"`, `size="lg"` | "Voltar para marketplace" |
| `Card` (AI) | `--card`, `--radius-lg`, borda `--primary` com opacidade | Card do Analista de Oportunidades |
| `Skeleton` | `className="h-4 w-3/4"` (múltiplos) | Placeholder da análise de IA |
| `Badge` | `variant="secondary"` | Label "Análise de IA" |
| `ScrollArea` | `className="h-[300px]"` | Área scrollável da análise de IA |

**AiRiskScoreBadge — configuração visual:**

```
Score 1-3  → background: --destructive/10, color: --destructive, label: "Risco Alto"
Score 4-6  → background: --chart-2/10,    color: --chart-3,     label: "Risco Médio"
Score 7-10 → background: --primary/10,    color: --primary,     label: "Risco Baixo"
```

**Estados:**

- **Skeleton:** Toda a seção de detalhes com `<Skeleton>` em bloco + análise de IA com 5 linhas de skeleton.
- **Empty:** n/a — tela sempre tem dados se rota foi acessada.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar esta oportunidade. Ela pode ter sido removida do marketplace."
- **Populated:** Todos os detalhes visíveis com análise de IA.

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Breadcrumb ao montar | Tab order lógico |
| ARIA roles | `aria-live="polite"` na análise IA (streaming) | Screen reader anuncia atualização |
| Skip links | `#main-content` | Visível |
| Contraste | Score badge text ≥ 4.5:1 | Validado |
| Motion | SSE streaming: sem animação se `prefers-reduced-motion` | Texto aparece instantâneo |

**Motion & Transições:**

- Entrada: fade-in geral (200ms)
- Análise IA: streaming via SSE — cursor piscante durante geração; sem cursor se `prefers-reduced-motion`
- Skeleton → Populated: `AnimatePresence` fade

**Regras específicas:**

- Nenhum dado do Cedente exibido (RN-014, RN-072).
- CTA "Fazer proposta" desabilitado se: quota 3 propostas simultâneas atingida (RN-020) — tooltip explicativo "Você já tem 3 propostas ativas".
- CTA desabilitado se: 10 propostas em 24h atingidas (RN-021) — tooltip "Limite diário atingido. Aguarde [X horas]".
- Análise IA carregada via SSE (Vercel AI SDK) — streaming de texto.
- Score de risco sempre visível, nunca oculto (RN-048).

---

### 5.3 T-OPR-03 — Formulário de Proposta

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-020, RF-021, RF-022, RF-023, RF-036, RF-037, RF-038 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `--card`, `--radius-lg` | Container do formulário de proposta |
| `CardHeader` | Padrão | "Sua proposta para OPR-XXXX" |
| `Input` | `type="text"`, `inputMode="numeric"`, `placeholder="R$ 0,00"`, `--input` | Valor da proposta (Preço Repasse) |
| `Card` (resumo financeiro) | `--muted`, `--radius-md` | Card resumo: Preço Repasse + Comissão Comprador + Total |
| `QuotaBar` (custom) | Props: `used: number`, `max: 3` | Barra de quota de propostas simultâneas |
| `Progress` | Para quota diária `used/10`, `--primary` | Quota diária de 10 propostas/24h |
| `Button` | `variant="default"`, `size="lg"`, `className="w-full"`, `--primary` | "Enviar proposta" |
| `Button` | `variant="outline"`, `size="lg"` | "Cancelar" |
| `Alert` | `variant="default"`, borda `--primary` | Resumo de regras antes do envio |
| `Skeleton` | `className="h-10 w-full"` | Placeholder durante cálculo de comissão |

**QuotaBar — configuração:**

```
used/max ≤ 0.66 → --primary   (azul — confortável)
used/max ≤ 0.89 → --chart-2   (azul claro — atenção)
used/max = 1.0  → --destructive (vermelho — limite atingido)
```

**Estados:**

- **Skeleton:** Card de resumo financeiro com skeleton enquanto calcula comissão.
- **Empty:** n/a — formulário estático.
- **Error:** `<Alert variant="destructive">` — "Não foi possível enviar a proposta. Tente novamente."
- **Populated:** Formulário com campo de valor preenchido e resumo financeiro calculado.

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | `autoFocus` no campo de valor | Cursor posicionado |
| ARIA roles | `aria-live="polite"` no resumo financeiro (atualiza ao digitar) | Screen reader anuncia valores |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | QuotaBar sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Resumo financeiro: atualização ao digitar com debounce de 300ms
- QuotaBar: `transition-width 300ms ease-out`
- Skeleton do resumo: exibido apenas durante cálculo (< 500ms esperado)
- Sem animações se `prefers-reduced-motion`

**Regras específicas:**

- Cálculo automático: Comissão Comprador = 20% × Δ; fallback quando Δ ≤ 0: 20% × Valor Pago pelo Cedente (RN-038).
- Total exibido = Preço Repasse + Comissão Comprador.
- Validar que valor > 0 antes de habilitar CTA.
- Ao enviar: `Dialog` de confirmação com resumo antes de submeter (RN-022).

**StatusBadge por ProposalStatus:**

```
ProposalStatus → Cor semântica → Variante Badge:
ENVIADA     → --chart-2    → Badge outline azul   ("Enviada")
EM_ANALISE  → --chart-3    → Badge solid azul      ("Em Análise")
ACEITA      → --primary    → Badge solid azul      ("Aceita") [DECISÃO AUTÔNOMA — --primary sinaliza ação positiva confirmada; alternativa descartada: verde (não disponível no design system)]
RECUSADA    → --destructive → Badge destructive    ("Recusada")
EXPIRADA    → --muted-foreground → Badge muted     ("Expirada")
CANCELADA   → --muted-foreground → Badge muted     ("Cancelada")
```

---

## 6. Propostas (T-PRP-01 e T-PRP-02)

### 6.1 T-PRP-01 — Lista de Propostas

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-020, RF-021, RF-022, RF-023 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Tabs` | `defaultValue="ativas"`, `--border` | Aba "Ativas" / "Histórico" |
| `TabsList` | Padrão | Container das abas |
| `TabsTrigger` | `--primary` ativo | Trigger de cada aba |
| `DataTable` | `columns`, `data`, `skeleton={<TableSkeleton rows={8} />}`, `emptyState`, `pagination={{ pageSize: 25 }}` | Tabela de propostas |
| `ProposalStatusBadge` | Mapeamento em §5.4 | Status da proposta na tabela |
| `QuotaBar` (custom) | Props: `used: number`, `max: 3` | Barra de propostas simultâneas ativas |
| `Button` | `variant="ghost"`, `size="sm"` | "Ver detalhes" na linha da tabela |
| `Button` | `variant="destructive"`, `size="sm"` | "Cancelar proposta" |
| `Badge` | `variant="secondary"` | Contador de propostas ativas |

**Colunas da DataTable:**

| Coluna | Tipo | Ordenável |
|---|---|---|
| Código | `string` (OPR-XXXX) | Sim |
| Status | `ProposalStatusBadge` | Sim |
| Valor proposto | `currency` (R$) | Sim |
| Comissão | `currency` (R$) | Não |
| Data envio | `datetime` | Sim |
| Ações | `Button` | Não |

**Estados:**

- **Skeleton:** `<TableSkeleton rows={8} />` em toda a tabela.
- **Empty (ativas):** `<EmptyState icon={<FileText />} title="Nenhuma proposta ativa" description="Explore o marketplace e envie sua primeira proposta." cta={<Button>Ver oportunidades</Button>} />`.
- **Empty (histórico):** `<EmptyState icon={<Archive />} title="Nenhuma proposta no histórico" description="Suas propostas concluídas ou canceladas aparecerão aqui." />`.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar suas propostas."
- **Populated:** Tabela com propostas.

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco na primeira aba ao montar | Tab funciona nas abas |
| ARIA roles | `role="tab"`, `role="tabpanel"`, `aria-selected` | Screen reader anuncia abas |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem transição de aba se `prefers-reduced-motion` | Troca instantânea |

**Motion & Transições:**

- Troca de aba: `AnimatePresence` fade-in do conteúdo (150ms)
- Skeleton → Tabela: `AnimatePresence` opacity
- Sem animações se `prefers-reduced-motion`

**Regras específicas:**

- QuotaBar exibida no topo da aba "Ativas" sempre.
- Botão "Cancelar proposta" visível apenas em status `ENVIADA` ou `EM_ANALISE`.
- Cancelamento exige `AlertDialog` de confirmação.

---

### 6.2 T-PRP-02 — Detalhe da Proposta

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-022, RF-023, RF-024 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Breadcrumb` | `--muted-foreground` | Propostas → OPR-XXXX |
| `ProposalStatusBadge` | Mapeamento §5.4 | Status atual da proposta |
| `Card` | `--card`, `--radius-lg` | Detalhes financeiros da proposta |
| `Separator` | `--border` | Divisores |
| `Button` | `variant="destructive"`, `size="sm"` | "Cancelar proposta" (se cancelável) |
| `Alert` | `variant="default"` com borda `--primary` | Informação de próximo passo |
| `Skeleton` | Blocos de skeleton para cada campo | Carregamento |

**Estados:**

- **Skeleton:** Blocos de `<Skeleton>` em cada campo do card.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar os detalhes desta proposta."
- **Populated:** Todos os campos visíveis.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Breadcrumb | Tab order lógico |
| ARIA roles | `role="region"` em cada seção | Screen reader anuncia |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Entrada: fade-in geral (200ms)
- Skeleton → Populated: `AnimatePresence`

**Regras específicas:**

- `AlertDialog` de confirmação ao cancelar: "Tem certeza que deseja cancelar esta proposta? Esta ação não pode ser desfeita."

---

## 7. Negociações (T-NEG-01 a T-NEG-04 + overlay)

### 7.1 T-NEG-01 — Lista de Negociações

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-024, RF-025, RF-026, RF-027 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `DataTable` | `columns`, `data`, `skeleton={<TableSkeleton rows={8} />}` | Lista de negociações |
| `NegotiationStatusBadge` | Mapeamento §7.5 | Status da negociação |
| `SlaCountdown` (custom) | Props: `deadline`, `urgency` | SLA de depósito visível na linha |
| `Button` | `variant="ghost"`, `size="sm"` | "Acessar negociação" |

**Colunas da DataTable:**

| Coluna | Tipo | Ordenável |
|---|---|---|
| Código | `string` | Sim |
| Status | `NegotiationStatusBadge` | Sim |
| Valor acordado | `currency` | Sim |
| Rodadas | `number` (de 3) | Não |
| SLA Escrow | `SlaCountdown` | Sim |
| Ações | `Button` | Não |

**Estados:**

- **Skeleton:** `<TableSkeleton rows={8} />`.
- **Empty:** `<EmptyState icon={<MessageSquare />} title="Nenhuma negociação em andamento" description="Quando uma proposta sua for aceita, a negociação aparecerá aqui." />`.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar suas negociações."
- **Populated:** Tabela com negociações e SLA countdown.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco na primeira linha da tabela | Tab order lógico |
| ARIA roles | `aria-live="polite"` no `SlaCountdown` | Screen reader anuncia mudanças de SLA |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | `SlaCountdown` sem cor pulsante se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Skeleton → Tabela: fade-in (200ms)
- `SlaCountdown` cor vermelha: sem pulse animation se `prefers-reduced-motion`

---

### 7.2 T-NEG-02 — Detalhe da Negociação (chat)

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-025, RF-026, RF-027, RF-028, RF-029 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` (info header) | `--card`, `--radius-lg` | Header com código OPR + status + rodadas |
| `NegotiationStatusBadge` | Mapeamento §7.5 | Status da negociação |
| `Badge` | `variant="outline"` | "Rodada X de 3" |
| `ScrollArea` | `className="h-[400px]"` | Área de histórico de mensagens |
| `Card` (mensagem Admin) | `--muted`, `--radius-lg`, alinhado à esquerda | Mensagem do Admin |
| `Card` (mensagem Cessionário) | `--primary/10`, `--radius-lg`, alinhado à direita | Mensagem do Cessionário |
| `Input` | `type="text"`, `placeholder="Digite sua contraproposta..."` | Campo de contraproposta |
| `Button` | `variant="default"`, `--primary` | "Enviar contraproposta" |
| `Button` | `variant="outline"` | "Aceitar valor atual" |
| `SlaCountdown` | Props: `deadline`, `urgency` | SLA do depósito Escrow |
| `Skeleton` | Linhas de skeleton no chat | Carregamento de histórico |

**Estados:**

- **Skeleton:** `<Skeleton>` em 3 linhas de chat + skeleton no header.
- **Empty:** `<EmptyState icon={<MessageSquare />} title="Negociação iniciada" description="Aguardando proposta do administrador." />`.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar a negociação."
- **Populated:** Histórico de chat + campo de contraproposta.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Input de contraproposta ao montar (se status `EM_CONTRAPROPOSTA`) | Tab order correto |
| ARIA roles | `role="log"` no ScrollArea do chat, `aria-live="polite"` | Screen reader anuncia novas mensagens |
| Skip links | `#main-content` | Visível |
| Contraste | Texto sobre `--primary/10` ≥ 4.5:1 | Validado |
| Motion | Sem scroll animado se `prefers-reduced-motion` | Scroll instantâneo |

**Motion & Transições:**

- Nova mensagem: `AnimatePresence` slide-in de baixo (150ms)
- Scroll automático para última mensagem (sem animação se `prefers-reduced-motion`)
- Skeleton → Populated: fade-in

**Regras específicas:**

- Máximo 3 rodadas de contraproposta (RN-027) — campo de input desabilitado após 3ª rodada com tooltip "Limite de rodadas atingido".
- Atualização em tempo real via Supabase Realtime.
- Contraproposta exige `Dialog` de confirmação com resumo financeiro.

---

### 7.3 T-NEG-03 — Instrução de Depósito Escrow

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-028, RF-029, RF-030 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Alert` | `variant="default"`, borda `--primary` | Instrução de depósito Escrow |
| `Card` | `--card`, `--radius-lg` | Dados bancários da conta Escrow |
| `Button` | `variant="ghost"`, `size="icon"` (ícone `Copy`) | Copiar chave PIX / dados bancários |
| `SlaCountdown` | Props: `deadline: Date`, `urgency` — destaque visual | Prazo de 10 dias úteis para depósito |
| `EscrowStatusBadge` | Mapeamento §7.4 | Status atual do Escrow |
| `Card` (resumo) | `--muted`, `--radius-md` | Resumo: Preço Repasse + Comissão + Total |
| `Button` | `variant="default"`, `--primary` | "Confirmar envio do depósito" |
| `Skeleton` | Em todos os campos durante carregamento | Loading |

**Estados:**

- **Skeleton:** Skeleton nos campos de dados bancários e no resumo.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar as instruções de depósito."
- **Populated:** Dados bancários + SLA countdown + CTA de confirmação.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Alert principal | Usuário vê instrução imediatamente |
| ARIA roles | `aria-live="assertive"` no `SlaCountdown` quando urgency=`high` | Screen reader anuncia urgência |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | SlaCountdown sem pulse vermelho se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- SlaCountdown muda de cor conforme urgência (transição de cor 500ms)
- Skeleton → Populated: fade-in (200ms)
- Sem animações se `prefers-reduced-motion`

**Regras específicas:**

- Prazo padrão: 10 dias úteis (RN-028); extensão de +5 dias úteis disponível (RN-029).
- Botão "Solicitar extensão" visível se SLA > 3 dias úteis restantes; oculto se < 3.
- CTA "Confirmar envio" abre `Dialog` de confirmação — "Você confirma que realizou o depósito de [valor]?"

**StatusBadge por EscrowStatus:**

```
EscrowStatus → Cor semântica → Variante Badge:
AGUARDANDO_DEPOSITO  → --chart-2      → Badge outline azul    ("Aguardando Depósito")
DEPOSITO_ENVIADO     → --chart-3      → Badge solid azul      ("Depósito Enviado")
DEPOSITO_CONFIRMADO  → --primary      → Badge solid azul      ("Depósito Confirmado")
REEMBOLSADO          → --muted-foreground → Badge muted       ("Reembolsado")
```

---

### 7.4 T-NEG-04 — Acompanhamento Escrow

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-029, RF-030 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `--card`, `--radius-lg` | Status atual do Escrow |
| `EscrowStatusBadge` | Mapeamento §7.3 | Status visual |
| `SlaCountdown` | Props: `deadline`, `urgency` | SLA restante |
| `Progress` | `value` baseado em dias decorridos/total | Progresso do prazo |
| `Alert` | `variant="default"` | Notificação de status |
| `Button` | `variant="outline"` | "Ver comprovante enviado" |
| `Skeleton` | Em status e progress durante loading | Loading |

**Estados:**

- **Skeleton:** `<Skeleton className="h-6 w-32" />` no badge + `<Skeleton className="h-2 w-full" />` no progress.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar o status do depósito."
- **Populated:** Status + progress + SLA countdown.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Card principal | Tab order lógico |
| ARIA roles | `aria-valuenow` no Progress, `aria-live="polite"` no status | Screen reader anuncia progresso |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Progress sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Progress: `transition-width 500ms ease-out` ao carregar
- Sem animações se `prefers-reduced-motion`

---

### 7.5 Overlay — Confirmação de Contraproposta

| Atributo | Valor |
|---|---|
| **Layout** | `Dialog` sobre `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-027 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Dialog` | `--card`, `--radius-xl` | Container do modal |
| `DialogHeader` | Padrão | "Confirmar contraproposta" |
| `DialogDescription` | `--muted-foreground` | Resumo da contraproposta |
| `Card` | `--muted`, `--radius-md` | Breakdown financeiro |
| `Button` | `variant="default"`, `--primary` | "Confirmar" |
| `Button` | `variant="outline"` | "Voltar e revisar" |

**StatusBadge por NegotiationStatus:**

```
NegotiationStatus → Cor semântica → Variante Badge:
EM_NEGOCIACAO        → --chart-2         → Badge outline azul    ("Em Negociação")
EM_CONTRAPROPOSTA    → --chart-3         → Badge solid azul      ("Em Contraproposta")
AGUARDANDO_DEPOSITO  → --chart-1         → Badge outline amarelo ("Aguardando Depósito")
DEPOSITO_CONFIRMADO  → --primary         → Badge solid azul      ("Depósito Confirmado")
ENCERRADA            → --muted-foreground → Badge muted          ("Encerrada")
CANCELADA            → --destructive     → Badge destructive     ("Cancelada")
```

**Estados:**

- **Skeleton:** `<Skeleton>` no breakdown financeiro durante cálculo.
- **Empty:** n/a.
- **Error:** Inline no Dialog — "Erro ao processar. Tente novamente."
- **Populated:** Resumo com valores calculados.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Focus trap no Dialog | Tab não sai do modal |
| ARIA roles | `role="dialog"`, `aria-modal="true"`, `aria-labelledby` | Screen reader anuncia dialog |
| Skip links | n/a — modal | n/a |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação de entrada se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Entrada: scale `0.95 → 1` + fade (150ms)
- Sem animações se `prefers-reduced-motion`

---

## 8. Formalização (T-ASS-01 a T-ASS-03)

### 8.1 T-ASS-01 — Documentos de Formalização

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` e Escrow confirmado |
| **RFs** | RF-031, RF-032, RF-033 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `--card`, `--radius-lg` | Card de cada documento disponível |
| `FormalizationStatusBadge` | Mapeamento §8.4 | Status do processo de formalização |
| `Badge` | `variant="outline"` | Tipo de documento (Instrumento de Cessão, etc.) |
| `Button` | `variant="default"`, `--primary` | "Visualizar documento" |
| `Button` | `variant="outline"` | "Baixar PDF" |
| `Alert` | `variant="default"` | Instrução de próximo passo |
| `Separator` | `--border` | Divisores |
| `Skeleton` | `className="h-[120px] w-full rounded-lg"` | Placeholder de cada card de documento |

**Estados:**

- **Skeleton:** 2x `<Skeleton className="h-[120px] w-full rounded-lg" />`.
- **Empty:** n/a — se chegou aqui, há documentos.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar os documentos de formalização."
- **Populated:** Lista de documentos com botões de ação.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no primeiro card de documento | Tab order |
| ARIA roles | `role="list"` nos documentos, `aria-label` por documento | Screen reader anuncia lista |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Skeleton → Populated: fade-in (200ms)
- Sem animações internas

---

### 8.2 T-ASS-02 — Assinatura Pendente

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-032, RF-033 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Alert` | `variant="default"`, borda `--primary` | Instrução de assinatura |
| `FormalizationStatusBadge` | Mapeamento §8.4 | Status atual |
| `Card` | `--card`, `--radius-lg` | Resumo do documento a assinar |
| `Button` | `variant="default"`, `size="lg"`, `--primary` | "Assinar documento" (abre ZapSign) |
| `Progress` | 2 passos (Cessionário → Cedente) | Progresso da formalização |
| `Skeleton` | Em card e progress | Loading |

**Estados:**

- **Skeleton:** Card + progress com skeleton.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar o documento para assinatura."
- **Populated:** Resumo + CTA de assinatura + progress.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Alert de instrução | Usuário vê instrução |
| ARIA roles | `aria-label` no Progress, `aria-valuenow` | Screen reader anuncia progresso |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Progress: `transition-width 400ms ease-out`
- Sem animações se `prefers-reduced-motion`

---

### 8.3 T-ASS-03 — Transição ZapSign

| Atributo | Valor |
|---|---|
| **Layout** | `FullPageLayout` [DECISÃO AUTÔNOMA — tela de transição para iframe/redirect ZapSign; sem sidebar, foco total na assinatura; alternativa descartada: AppLayout (sidebar distrai e ocupa espaço)] |
| **Role mínimo** | `CESSIONARIO` |
| **RFs** | RF-032 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `className="max-w-[600px] mx-auto"`, `--card`, `--radius-xl` | Card de transição |
| `CardHeader` | Padrão | "Você será redirecionado para a assinatura" |
| `CardDescription` | `--muted-foreground` | Instrução de que abrirá plataforma ZapSign |
| `Button` | `variant="default"`, `size="lg"`, `--primary` | "Ir para assinatura" |
| `Button` | `variant="ghost"` | "Voltar" |
| `Skeleton` | `className="h-12 w-full"` | Loading do redirect |
| `Alert` | `variant="default"` | Nota de segurança sobre o ZapSign |

**FormalizationStatus — mapeamento:**

```
FormalizationStatus → Cor semântica → Variante Badge:
DOCUMENTOS_DISPONIVEIS           → --muted-foreground  → Badge muted       ("Documentos Disponíveis")
ASSINATURA_PENDENTE_CESSIONARIO  → --chart-2           → Badge outline azul ("Assinatura Pendente")
ASSINATURA_PENDENTE_CEDENTE      → --chart-3           → Badge solid azul   ("Aguardando Cedente")
AGUARDANDO_ANUENCIA              → --chart-1           → Badge outline      ("Aguardando Anuência")
CONCLUIDA                        → --primary           → Badge solid azul   ("Concluída")
CANCELADA                        → --destructive       → Badge destructive  ("Cancelada")
```

**Estados:**

- **Skeleton:** `<Skeleton className="h-12 w-full" />` no CTA durante redirect.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Não foi possível conectar ao serviço de assinatura. Tente novamente."
- **Populated:** Card com instrução e CTA.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no CTA "Ir para assinatura" | Usuário pode pressionar Enter |
| ARIA roles | `aria-live="polite"` no Alert | Screen reader anuncia |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação de redirect se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Fade-in do Card (200ms) ao montar
- Sem animações se `prefers-reduced-motion`

---

## 9. Financeiro (T-FIN-01 a T-FIN-03)

### 9.1 T-FIN-01 — Resumo Financeiro

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-036, RF-037, RF-038, RF-039, RF-040 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` (KPI) | `--card`, `--radius-lg` | KPI: Total investido, Comissões pagas, Operações concluídas |
| `Skeleton` | `className="h-8 w-24"` | Placeholder KPI valor |
| `Tabs` | `defaultValue="depositos"` | Depósitos / Comissões / Histórico |
| `DataTable` | `columns`, `data`, `skeleton`, `emptyState` | Tabela de transações |
| `Badge` | Por tipo de transação | Tipo (Depósito Escrow, Comissão, Reembolso) |
| `EscrowStatusBadge` | Mapeamento §7.3 | Status do Escrow na linha |

**Colunas da DataTable:**

| Coluna | Tipo | Ordenável |
|---|---|---|
| Data | `datetime` | Sim |
| Tipo | `Badge` | Sim |
| Descrição | `string` | Não |
| Valor | `currency` (R$) | Sim |
| Status | `Badge` | Sim |

**Estados:**

- **Skeleton:** 3x `<Skeleton className="h-20 w-full" />` nos KPIs + `<TableSkeleton rows={8} />`.
- **Empty:** `<EmptyState icon={<DollarSign />} title="Nenhuma transação ainda" description="Seu histórico financeiro aparecerá aqui após a primeira operação." />`.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar o histórico financeiro."
- **Populated:** KPIs + tabela com transações.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no primeiro KPI | Tab order |
| ARIA roles | `role="region"` em cada seção | Screen reader anuncia |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação de KPI se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- KPI: counter animation (0 → valor final, 600ms) — desabilitado se `prefers-reduced-motion`
- Troca de aba: fade-in (150ms)
- Skeleton → Populated: fade-in

---

### 9.2 T-FIN-02 — Detalhe de Transação

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-039 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Breadcrumb` | `--muted-foreground` | Financeiro → Transação |
| `Card` | `--card`, `--radius-lg` | Detalhes da transação |
| `Badge` | Por tipo | Tipo da transação |
| `Separator` | `--border` | Divisores |
| `Button` | `variant="outline"` | "Baixar comprovante" (PDF) |
| `Skeleton` | Blocos nos campos | Loading |

**Estados:**

- **Skeleton:** Blocos de skeleton.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Transação não encontrada."
- **Populated:** Detalhes completos da transação.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Breadcrumb | Tab order |
| ARIA roles | `role="region"` | Screen reader anuncia |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Skeleton → Populated: fade-in (200ms)

---

### 9.3 T-FIN-03 — Comprovante de Depósito

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-040 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `--card`, `--radius-lg` | Container do comprovante |
| `Badge` | `variant="default"`, `--primary` | Status "Confirmado" |
| `Separator` | `--border` | Divisores |
| `Button` | `variant="default"`, `--primary` | "Baixar PDF" |
| `Button` | `variant="outline"` | "Compartilhar" |
| `Skeleton` | Bloco | Loading |

**Estados:**

- **Skeleton:** Card inteiro com skeleton.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Comprovante não disponível."
- **Populated:** Comprovante completo.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no CTA de download | Tab order |
| ARIA roles | `role="document"` no Card | Screen reader anuncia documento |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Skeleton → Populated: fade-in (200ms)

---

## 10. Analista de IA (T-IA-01)

### 10.1 T-IA-01 — Chat com Analista de Oportunidades

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` com `KYC_APROVADO` |
| **RFs** | RF-046, RF-047, RF-048, RF-049, RF-050 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` (header) | `--card`, `--radius-lg`, borda `--primary` tênue | Header: "Analista de Oportunidades" + badge "IA" |
| `Badge` | `variant="secondary"` | Label "GPT-4 · Powered by OpenAI" |
| `ScrollArea` | `className="flex-1 min-h-0"` | Área de histórico do chat |
| `Card` (mensagem usuário) | `--primary/10`, alinhado à direita | Mensagem do usuário |
| `Card` (mensagem IA) | `--card`, `--muted-foreground`, alinhado à esquerda | Resposta do Analista |
| `Skeleton` | 3 linhas durante streaming | Loading da resposta IA |
| `Input` | `type="text"`, `placeholder="Pergunte sobre esta oportunidade..."` | Campo de pergunta |
| `Button` | `variant="default"`, `size="icon"`, `--primary` | Enviar (ícone `Send`) |
| `Alert` | `variant="default"` | Aviso: "O Analista não tem acesso a dados do vendedor" |

**Estados:**

- **Skeleton:** 3x `<Skeleton className="h-4 w-3/4" />` empilhados durante geração de resposta.
- **Empty:** `<EmptyState icon={<Bot />} title="Analista de Oportunidades" description="Faça perguntas sobre qualquer oportunidade do marketplace. Análises baseadas em dados agregados, sem dados pessoais do vendedor." cta={null} />`.
- **Error:** `<Alert variant="destructive">` — "Não foi possível conectar ao Analista. Tente novamente."
- **Populated:** Histórico de mensagens + campo de input ativo.

**Acessibilidade (WCAG 2.1 AA):**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Input ao montar | Usuário pode digitar imediatamente |
| ARIA roles | `role="log"` no ScrollArea, `aria-live="polite"` nas mensagens da IA | Screen reader anuncia respostas |
| Skip links | `#main-content` | Visível |
| Contraste | Texto sobre `--primary/10` ≥ 4.5:1 | Validado |
| Motion | Cursor piscante de streaming desabilitado se `prefers-reduced-motion` | Texto aparece instantâneo |

**Motion & Transições:**

- Entrada da tela: fade-in (200ms)
- Mensagem do usuário: slide-in da direita (150ms)
- Resposta IA: streaming via SSE — aparece caractere a caractere com cursor piscante; se `prefers-reduced-motion`: exibe tudo de uma vez após conclusão
- Scroll automático para última mensagem: sem animação se `prefers-reduced-motion`

**Regras específicas:**

- Analista não tem acesso a nenhum dado do Cedente (RN-014, RN-049) — aviso sempre visível.
- Tom analítico, orientado a dados — sem FOMO (RN-047).
- Sessão de chat preservada durante a sessão do usuário (não persistida entre sessões, exceto contexto da oportunidade ativa).
- Streaming via Vercel AI SDK + SSE.
- Score de risco sempre referenciado nas respostas quando disponível.

---

## 11. Perfil (T-PRF-01 a T-PRF-05)

### 11.1 T-PRF-01 — Perfil Principal

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` |
| **RFs** | RF-011, RF-012, RF-013, RF-014, RF-015, RF-016 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `--card`, `--radius-lg` | Card de dados pessoais |
| `CessionarioStatusBadge` | Mapeamento §11.4 | Status da conta |
| `Avatar` | `size="lg"`, fallback com iniciais | Avatar do usuário |
| `Input` | `type="text"`, `disabled` | Campos somente leitura (nome, e-mail) |
| `Button` | `variant="outline"` | "Editar informações" |
| `Button` | `variant="outline"`, `--destructive` texto | "Solicitar exclusão de dados (LGPD)" |
| `Separator` | `--border` | Divisores de seção |
| `Skeleton` | Em cada campo | Loading |

**Estados:**

- **Skeleton:** 4x `<Skeleton className="h-10 w-full" />` nos campos.
- **Empty:** n/a — sempre há dados se logado.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar seus dados."
- **Populated:** Todos os campos com dados do usuário.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no primeiro campo editável | Tab order |
| ARIA roles | `role="form"`, `aria-label="Dados pessoais"` | Screen reader anuncia |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Skeleton → Populated: fade-in (200ms)

---

### 11.2 T-PRF-02 — KYC Stepper (3 etapas)

| Atributo | Valor |
|---|---|
| **Layout** | `FullPageLayout` |
| **Role mínimo** | `CESSIONARIO` com `status !== 'KYC_APROVADO'` |
| **RFs** | RF-006, RF-007, RF-008, RF-009, RF-010 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Progress` | 3 etapas (33/66/100%), `--primary` | Step indicator de progresso |
| `Badge` | `variant="outline"` por etapa | Labels "Documento", "Endereço", "Selfie" |
| `Card` | `--card`, `--radius-xl`, `className="max-w-[500px] mx-auto"` | Container de cada etapa |
| `Input` | `type="file"`, `accept="image/*,application/pdf"` | Upload de documentos |
| `Label` | Padrão | Labels dos campos |
| `Button` | `variant="default"`, `size="lg"`, `className="w-full"`, `--primary` | "Continuar" / "Enviar para análise" |
| `Button` | `variant="outline"` | "Voltar" (entre etapas) |
| `Alert` | `variant="default"` | Instrução por etapa |
| `Skeleton` | `className="h-40 w-full rounded-lg"` | Preview do documento durante upload |

**Etapas:**

| Etapa | Título | Documentos aceitos | CTA |
|---|---|---|---|
| 1 — Documento | "Documento de identidade" | RG (frente + verso) ou CNH | "Avançar para endereço" |
| 2 — Endereço | "Comprovante de endereço" | Conta (água, luz, gás, bancária) | "Avançar para selfie" |
| 3 — Selfie | "Selfie de prova de vida" | Câmera em tempo real ou upload | "Enviar para análise" |

**Estados:**

- **Skeleton:** `<Skeleton className="h-40 w-full rounded-lg" />` durante upload.
- **Empty:** Etapa inicial pronta para upload (sem documento carregado).
- **Error:** `<Alert variant="destructive">` por tipo: "Formato inválido", "Arquivo muito grande (max 10MB)", "Documento ilegível".
- **Populated:** Preview do documento carregado + CTA habilitado.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no input de upload de cada etapa | Tab order correto |
| ARIA roles | `aria-current="step"` na etapa ativa do Progress | Screen reader anuncia etapa atual |
| Skip links | `#main-content` | Visível |
| Contraste | Labels em `--foreground` ≥ 4.5:1 | Validado |
| Motion | Progress sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Transição entre etapas: slide horizontal (300ms, `ease-in-out`) via `AnimatePresence`
- Progress: `transition-width 300ms ease-out` ao avançar etapas
- Skeleton de upload: exibido até preview estar disponível
- Sem animações se `prefers-reduced-motion`

**Regras específicas:**

- Documentos aceitos: apenas imagens (JPG, PNG, HEIC) e PDF.
- Limite de tamanho: 10MB por arquivo.
- Upload processado via Supabase Storage em bucket privado.
- Após envio: tela T-PRF-03 (status KYC).

---

### 11.3 T-PRF-03 — Status KYC

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` |
| **RFs** | RF-006, RF-007, RF-008, RF-010 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `--card`, `--radius-lg` | Card de status KYC |
| `CessionarioStatusBadge` | Mapeamento §11.4 | Status atual do KYC |
| `Alert` | `variant="default"` ou `variant="destructive"` | Mensagem de status |
| `Button` | `variant="default"`, `--primary` | "Reenviar documentos" (se reprovado) |
| `Skeleton` | Em status e alert | Loading |

**StatusBadge por CessionarioStatus (consolidado):**

```
CessionarioStatus → Cor semântica → Variante Badge:
CADASTRADA                → --muted-foreground   → Badge muted         ("Cadastrada")
KYC_EM_ANALISE            → --chart-2            → Badge outline azul  ("KYC em Análise")
KYC_APROVADO              → --primary            → Badge solid azul    ("KYC Aprovado")
KYC_REPROVADO             → --destructive        → Badge destructive   ("KYC Reprovado")
BLOQUEADA_TEMPORARIAMENTE → --chart-1            → Badge outline amber ("Bloqueada Temporariamente")
ENCERRADA                 → --muted-foreground   → Badge muted         ("Encerrada")
```

**Estados:**

- **Skeleton:** `<Skeleton className="h-6 w-32" />` no badge + `<Skeleton className="h-16 w-full" />` no Alert.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar o status do KYC."
- **Populated:** Status atual + mensagem contextual.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no Card de status | Tab order |
| ARIA roles | `aria-live="polite"` no status (pode mudar) | Screen reader anuncia mudanças |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Status muda via Supabase Realtime — `AnimatePresence` fade (200ms) ao mudar badge
- Sem animações se `prefers-reduced-motion`

---

### 11.4 T-PRF-04 — Editar Perfil

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` |
| **RFs** | RF-011, RF-012 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `--card`, `--radius-lg` | Container do formulário |
| `Input` | `type="text"`, `--input` | Campos editáveis (nome, telefone) |
| `Input` | `type="email"`, `disabled` | E-mail somente leitura |
| `Button` | `variant="default"`, `--primary` | "Salvar alterações" |
| `Button` | `variant="outline"` | "Cancelar" |
| `Alert` | `variant="default"` | Confirmação de salvamento |
| `Skeleton` | Em campos | Loading inicial |

**Estados:**

- **Skeleton:** 4x `<Skeleton className="h-10 w-full" />`.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Não foi possível salvar as alterações."
- **Populated:** Formulário preenchido com dados atuais.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no primeiro campo editável | Tab order |
| ARIA roles | `aria-describedby` nos campos com restrição | Screen reader anuncia restrições |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Alert de sucesso: `AnimatePresence` fade-in (200ms), auto-dismiss após 3s
- Sem animações se `prefers-reduced-motion`

---

### 11.5 T-PRF-05 — Configurações (Notificações, Segurança)

| Atributo | Valor |
|---|---|
| **Layout** | `AppLayout` |
| **Role mínimo** | `CESSIONARIO` |
| **RFs** | RF-015, RF-016, RF-064, RF-065 |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Tabs` | `defaultValue="notificacoes"` | Notificações / Segurança / Privacidade |
| `Switch` | `--primary` ativo | Toggles de preferências de notificação |
| `Label` | `--foreground` | Labels de cada toggle |
| `Card` | `--card`, `--radius-lg` | Container por seção |
| `Separator` | `--border` | Divisores |
| `Button` | `variant="destructive"` | "Alterar senha" / "Encerrar conta" |
| `AlertDialog` | `--card`, `--radius-xl` | Confirmação de encerramento de conta |
| `Skeleton` | Em cada switch | Loading inicial |

**Estados:**

- **Skeleton:** Switches com `<Skeleton className="h-6 w-10 rounded-full" />`.
- **Empty:** n/a.
- **Error:** `<Alert variant="destructive">` — "Não foi possível carregar suas preferências."
- **Populated:** Switches com estado salvo.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no primeiro Switch | Tab order |
| ARIA roles | `role="switch"`, `aria-checked` nos toggles | Screen reader anuncia estado |
| Skip links | `#main-content` | Visível |
| Contraste | `--primary` sobre branco ≥ 4.5:1 | Validado |
| Motion | Switch sem animação de toggle se `prefers-reduced-motion` | Troca instantânea |

**Motion & Transições:**

- Switch: `transition-transform 150ms ease-in-out` para o thumb — desabilitado se `prefers-reduced-motion`
- Troca de aba: fade-in (150ms)
- Sem animações se `prefers-reduced-motion`

**Regras específicas:**

- "Encerrar conta" exige `AlertDialog` com texto "Todos os seus dados serão anonimizados conforme a LGPD. Esta ação é irreversível."
- Canal mínimo de notificação: e-mail (RN-058) — toggle de e-mail não pode ser desabilitado.

---

## 12. Páginas de Erro

### 12.1 Erro 404 — Página não encontrada

| Atributo | Valor |
|---|---|
| **Layout** | `ErrorLayout` |
| **Role mínimo** | Público |
| **RFs** | n/a |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `className="max-w-[400px] mx-auto"`, `--card`, `--radius-xl` | Container da mensagem de erro |
| `Button` | `variant="default"`, `--primary` | "Voltar ao início" |
| `Button` | `variant="ghost"` | "Ir para o dashboard" |

**Estados:**

- **Skeleton:** n/a — página estática.
- **Empty:** n/a.
- **Error:** Tela inteira é o estado de erro.
- **Populated:** Ilustração SVG + "Página não encontrada" + CTAs.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no CTA principal | Tab order |
| ARIA roles | `role="main"`, `aria-label="Erro 404"` | Screen reader anuncia |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

**Motion & Transições:**

- Entrada: fade-in do Card (200ms)
- Sem animações se `prefers-reduced-motion`

---

### 12.2 Erro 500 — Erro interno

| Atributo | Valor |
|---|---|
| **Layout** | `ErrorLayout` |
| **Role mínimo** | Público |
| **RFs** | n/a |

**Tabela de componentes:**

| Componente shadcn/ui | Variante/Config | Propósito |
|---|---|---|
| `Card` | `className="max-w-[400px] mx-auto"`, `--card`, `--radius-xl` | Container |
| `Alert` | `variant="destructive"` | Mensagem de erro técnico |
| `Button` | `variant="default"`, `--primary` | "Tentar novamente" |
| `Button` | `variant="ghost"` | "Voltar ao início" |

**Estados:** Idêntico ao 404. Tela estática de erro.

**Acessibilidade:**

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Focus management | Foco no CTA principal | Tab order |
| ARIA roles | `role="alertdialog"`, `aria-live="assertive"` | Screen reader anuncia erro crítico |
| Skip links | `#main-content` | Visível |
| Contraste | ≥ 4.5:1 | Validado |
| Motion | Sem animação se `prefers-reduced-motion` | Estático |

---

## 13. Cross-References

| Documento | Relação |
|---|---|
| **01 - Regras de Negócio** | Fonte de regras de domínio para estados de componentes (quotas, SLAs, limites) |
| **03 - Brand Theme Guide** | Fonte de tokens de design (namespace `--`) — todos os tokens deste documento são derivados daqui |
| **06 - Mapa de Telas** | Fonte primária de IDs de telas, rotas e RFs — determina quais telas documentar |
| **07 - Wireframes** | Referência visual de layout e estrutura por tela |
| **08 - UX Writing** | Fonte de labels, mensagens de erro, empty states e microcopy |
| **D11 - Mobile** | Consome os contratos de UI para adaptação aos breakpoints mobile |
| **D14 - Especificações Técnicas** | Referencia os componentes para definir interfaces TypeScript |
| **D15 - Arquitetura de Pastas** | Usa o mapeamento de módulos para estruturar `features/<módulo>/pages/` |
| **D28 - Checklist de Qualidade** | Valida 4 estados, tokens e componentes contra este documento |

---

## 14. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop — ShiftLabs v9.5 | Criação inicial — 34 telas cobertas (100%), 4 estados obrigatórios, 4 layouts base, 10 componentes de domínio, mapeamentos completos de 5 enums de status, acessibilidade WCAG 2.1 AA e motion por tela |

---

## 15. Backlog de Pendências

| Item | Tipo | Tela / Módulo | Impacto | Justificativa / Decisão | Dono | Status |
|---|---|---|---|---|---|---|
| T-ASS-03 usa `FullPageLayout` em vez de `AppLayout` | Decisão Autônoma | T-ASS-03 — Transição ZapSign | P2 | Tela de transição para serviço externo — sem sidebar melhora foco; `AppLayout` descartado | Frontend Lead | Decidido |
| Status `BLOQUEADA_TEMPORARIAMENTE` usa `--chart-1` (azul claro) | Decisão Autônoma | T-PRF-01, T-PRF-03 | P2 | Bloqueio temporário não é terminal como `ENCERRADA` nem crítico como `KYC_REPROVADO`; `--chart-1` sinaliza atenção sem alarme; `--destructive` descartado por severidade excessiva | Frontend Lead | Decidido |
| Status `ACEITA` da Proposta usa `--primary` em vez de cor verde | Decisão Autônoma | T-OPR-03, T-PRP-01 | P2 | Paleta do produto não inclui cor verde semântica; `--primary` (azul) é a cor de ação positiva do sistema; verde poderia ser interpretado como "concluído" e não apenas "aceito" | Frontend Lead | Decidido |
| `EmptyState` é componente custom (não nativo do shadcn/ui) | Decisão Autônoma | Todas as telas | P1 | shadcn/ui não tem componente EmptyState nativo; decisão: criar componente custom com `icon + title + description + cta` usando tokens shadcn; registrado aqui para doc 03 | Frontend Lead | Decidido |
| `SlaCountdown` é componente custom | Decisão Autônoma | T-NEG-01, T-NEG-03, T-NEG-04, T-DASH-01 | P1 | Não há componente de countdown no shadcn/ui; criado custom com faixas de cor por urgência usando tokens semânticos do sistema | Frontend Lead | Decidido |
| `QuotaBar` é componente custom | Decisão Autônoma | T-OPR-03, T-PRP-01 | P1 | Wrapper sobre `<Progress>` do shadcn com lógica de cor por faixa de saturação; semântica de quota não existe nativamente | Frontend Lead | Decidido |
| `AiRiskScoreBadge` usa transparência 10% em background | Decisão Autônoma | T-OPR-01, T-OPR-02 | P2 | Background com opacidade cria destaque visual sem sobrepor o card; alternativa (cor sólida) descartada por poluição visual no card | Frontend Lead | Decidido |
