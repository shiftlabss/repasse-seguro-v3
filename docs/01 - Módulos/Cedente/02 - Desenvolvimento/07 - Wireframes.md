# Repasse Seguro — Módulo Cedente
## 07 - Wireframes

| **Campo** | **Valor** |
|---|---|
| **Documento** | 07 - Wireframes |
| **Produto** | Repasse Seguro — Módulo Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-22 12:00 (America/Fortaleza) |
| **Autor** | Claude Code Desktop — Pipeline v9.5 |
| **Status** | Ativo |
| **Fase** | 2 — Produto |
| **Área** | Design + Frontend |
| **Referências** | 01 - Regras de Negócio · 03 - Brand Theme Guide · 05 - PRD · 06 - Mapa de Telas |

---

> 📌 **TL;DR**
>
> - **52 wireframes gerados** (T-001 a T-052) — cobertura 100% das telas do Mapa de Telas v1.1.
> - **12 módulos cobertos**: Autenticação, Global, Dashboard, Meus Casos, Cadastro de Imóvel, Cenários/Escalonamento, Propostas, Documentos, Assinaturas, Financeiro, Assistente IA (Guardião), Perfil/Configurações.
> - **Design tokens** do Brand Theme Guide (shadcn/ui + Tailwind CSS 4+) aplicados em todas as telas: primary `#0069A8`, sidebar `#FAFAFA`, radius base `0.875rem`, tipografia Inter Variable.
> - **Padrão de layout**: sidebar fixa 240px (expandida) / 64px (colapsada) para telas autenticadas; layout centralizado max-width 480px para telas públicas.
> - **Estados obrigatórios** em todas as telas: Loading (skeleton retangulares, NUNCA spinner) → Empty → Error → Populated.
> - **0 telas pendentes** — nenhum `[WIREFRAME PENDENTE]` registrado.
> - **7 decisões autônomas** registradas no Backlog de Pendências (herdadas do B04 + decisões de composição de wireframe).

---

## 1. Persona

Este documento destina-se a **frontend engineers**, **product designers** e **QA engineers** que precisam de referência visual executável para implementar e validar as 52 telas do Módulo Cedente.

### 1.1 Como consumir este documento

- **Frontend:** use o layout ASCII + componentes-chave para implementar hierarquia e composição sem ambiguidade.
- **Designer:** use os wireframes como base para refinamento visual no Design System.
- **QA:** use os estados obrigatórios (Loading/Empty/Error/Populated) como base para casos de teste.

### 1.2 Anti-exemplos

> 🔴 **Erros que invalidam a implementação:**
>
> - ❌ Spinner como estado de loading — usar skeleton retangular obrigatoriamente.
> - ❌ Ignorar focus trap em modais — obrigatório em todos os `role="dialog"`.
> - ❌ Tela autenticada sem sidebar — sidebar é parte do layout base.
> - ❌ Componente sem nome definido — todos os componentes estão nomeados neste documento.

### 1.3 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 03 - Brand Theme Guide · 05 - PRD · 06 - Mapa de Telas

---

## 2. Design Tokens de Referência

Tokens extraídos do **03 - Brand Theme Guide** (shadcn/ui, Tailwind CSS 4+). Aplicados em todos os wireframes via CSS custom properties.

### 2.1 Superfícies e Fundo

| Token | Valor (Light) | Uso |
|---|---|---|
| `--background` | `#FFFFFF` | Fundo principal da aplicação |
| `--foreground` | `#0A0A0A` | Texto principal |
| `--card` | `#FFFFFF` | Fundo de cards |
| `--card-foreground` | `#0A0A0A` | Texto em cards |
| `--muted` | `#F5F5F5` | Backgrounds inativos, áreas recuadas |
| `--muted-foreground` | `#737373` | Texto secundário, placeholders, timestamps |

### 2.2 Cores Semânticas

| Token | Valor (Light) | Uso |
|---|---|---|
| `--primary` | `#0069A8` | CTAs, botões primários, item ativo |
| `--primary-foreground` | `#F0F9FF` | Texto sobre primary |
| `--secondary` | `#F4F4F5` | Botões secundários, backgrounds sutis |
| `--destructive` | `#E7000B` | Ações destrutivas, erros críticos |
| `--border` | `#E5E5E5` | Bordas de containers |
| `--ring` | `#A1A1A1` | Focus ring |

### 2.3 Sidebar

| Token | Valor | Uso |
|---|---|---|
| `--sidebar` | `#FAFAFA` | Fundo da sidebar |
| `--sidebar-foreground` | `#0A0A0A` | Texto da sidebar |
| `--sidebar-primary` | `#0084D1` | Item ativo da sidebar |
| `--sidebar-border` | `#E5E5E5` | Bordas e divisores da sidebar |
| **Largura expandida** | `240px` | Sidebar desktop padrão |
| **Largura colapsada** | `64px` | Sidebar colapsada (ícones apenas) |

### 2.4 Grid e Layout

| Parâmetro | Valor | Uso |
|---|---|---|
| Grid | 12 colunas | Área de conteúdo autenticada |
| Gap | `1.5rem` (24px) | Espaçamento entre colunas |
| Max-width conteúdo | `1280px` | Container máximo da área logada |
| Max-width formulários públicos | `480px` | Login, cadastro, recuperação |
| Breakpoints | `<768px` mobile · `768–1024px` tablet · `>1024px` desktop | Responsividade |

### 2.5 Tipografia e Radius

| Token | Valor | Uso |
|---|---|---|
| `--font-sans` | `'Inter Variable', sans-serif` | Toda a interface |
| `--radius` | `0.875rem` (14px) | Base |
| `--radius-md` | `0.7rem` (11px) | Botões, inputs |
| `--radius-lg` | `0.875rem` (14px) | Cards |
| `--radius-xl` | `1.225rem` (20px) | Cards destaque, modais |

### 2.6 Cores Semânticas de Estado

| Estado | Token / Cor | Uso em wireframe |
|---|---|---|
| Sucesso | `green-600` / `#16A34A` | Toast sucesso, checkmark, badge "Verificado" |
| Erro | `--destructive` / `#E7000B` | Banner erro, borda campo inválido |
| Warning / Âmbar | `--chart-1` / `#74D4FF` bg sutil · âmbar `#F59E0B` | Alertas não-críticos, expiração |
| Info | `--primary` sutil | Informativos, tooltips |

---

## 3. Índice de Módulos

| # | Módulo | Telas | IDs | RFs Cobertos |
|---|---|---|---|---|
| 1 | Autenticação | 10 | T-001 a T-010 | RF-001–RF-010 |
| 2 | Global (Shell) | 2 | T-011, T-012 | RF-016, RF-017, RF-120 |
| 3 | Dashboard | 2 | T-013, T-014 | RF-019–RF-023 |
| 4 | Meus Casos | 10 | T-015 a T-023, T-016–T-021 | RF-024–RF-027, RF-049, RF-056, RF-064, RF-071, RF-075, RF-076, RF-084, RF-085 |
| 5 | Cadastro de Imóvel (Wizard) | 7 | T-024 a T-030 | RF-028–RF-034, RF-050, RF-051 |
| 6 | Cenários e Escalonamento | 4 | T-031 a T-034 | RF-036, RF-037, RF-039-A, RF-040 |
| 7 | Propostas e Negociação | 5 | T-035 a T-039 | RF-041–RF-048, RF-052 |
| 8 | Documentos (Dossiê) | 3 | T-040 a T-042 | RF-056–RF-061, RF-120 |
| 9 | Assinaturas Eletrônicas | 2 | T-043, T-044 | RF-064–RF-068 |
| 10 | Financeiro (Conta Escrow) | 3 | T-045 a T-047 | RF-053, RF-054, RF-071–RF-074 |
| 11 | Assistente IA (Guardião) | 2 | T-048, T-049 | RF-086–RF-092 |
| 12 | Perfil e Configurações | 3 | T-050 a T-052 | RF-011–RF-015, RF-079, RF-080 |
| **Total** | **12 módulos** | **52 telas** | **T-001 a T-052** | **100% cobertura PRD** |

---

## 4. Wireframes por Módulo

---

## Módulo 1 — Autenticação

### T-001 — Login
**Rota:** `/login` · **Role:** `PUBLIC` · **RFs:** RF-007, RF-008, RF-010

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │  [Logo Repasse Seguro 120px]   │                  │
│              │                               │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ E-mail *                │  │                  │
│              │  │ [seu@email.com        ] │  │                  │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ Senha *           [👁]  │  │                  │
│              │  │ [••••••••         ]     │  │                  │
│              │  └─────────────────────────┘  │                  │
│              │  [Esqueci minha senha]         │                  │
│              │                               │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │      [Entrar]           │  │  ← --primary     │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  [Não tem conta? Criar conta] │                  │
│              └───────────────────────────────┘                  │
│                        max-width: 480px                         │
└─────────────────────────────────────────────────────────────────┘

ERRO DE CREDENCIAIS:
│  ┌─────────────────────────────────────────┐  │
│  │ ⚠ E-mail ou senha incorretos.          │  │  ← banner --destructive sutil
│  └─────────────────────────────────────────┘  │

CONTAGEM DE FALHAS (a partir da 3ª tentativa):
│  ┌─────────────────────────────────────────┐  │
│  │ ⚠ 3 de 5 tentativas. Após 5, sua      │  │  ← banner âmbar
│  │   conta será bloqueada por 15 min.     │  │
│  └─────────────────────────────────────────┘  │
```

**Componentes-chave:**
- `Input` (type="email") com validação inline on-blur
- `Input` (type="password") com toggle mostrar/ocultar (ícone 44×44px)
- `Button` variant="default" largura total (--primary)
- `Alert` variant="destructive" para erros inline
- `Alert` variant="warning" (âmbar) para contagem de falhas e e-mail não confirmado

**Estados:**
- **Loading:** Botão "Entrar" desabilitado + label substituída por skeleton 16px inline; inputs readonly
- **Empty:** Formulário em branco, botão "Entrar" habilitado (validação on-submit)
- **Error:** Banner vermelho inline "E-mail ou senha incorretos." abaixo do form; borda vermelha nos campos
- **Populated:** Campos preenchidos, sem erros, botão "Entrar" ativo

---

### T-002 — Cadastro
**Rota:** `/cadastro` · **Role:** `PUBLIC` · **RFs:** RF-001, RF-002

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│         ┌─────────────────────────────────────────┐             │
│         │  [Logo 80px]   Criar conta              │             │
│         │                                         │             │
│         │  ┌──────────────────────────────────┐   │             │
│         │  │  [Pessoa Física] [Pessoa Jurídica]│   │  ← toggle  │
│         │  └──────────────────────────────────┘   │             │
│         │                                         │             │
│         │  Nome completo *                        │             │
│         │  [Seu nome completo              ]      │             │
│         │                                         │             │
│         │  CPF *  (PF) / CNPJ * (PJ)             │             │
│         │  [000.000.000-00                 ]      │             │
│         │  [Razão Social *] ← apenas PJ           │             │
│         │                                         │             │
│         │  E-mail *                               │             │
│         │  [seu@email.com                  ]      │             │
│         │                                         │             │
│         │  Confirmar e-mail *                     │             │
│         │  [seu@email.com                  ]      │             │
│         │                                         │             │
│         │  Telefone *                             │             │
│         │  [(XX) XXXXX-XXXX               ]       │             │
│         │                                         │             │
│         │  Senha *                                │             │
│         │  [••••••••••               ] [👁]       │             │
│         │  ┌────────────────────────────────┐     │             │
│         │  │ ✓ 8+ caracteres               │     │  ← checklist│
│         │  │ ○ 1 maiúscula                 │     │    força    │
│         │  │ ○ 1 número                    │     │    senha    │
│         │  │ ○ 1 especial                  │     │             │
│         │  └────────────────────────────────┘     │             │
│         │                                         │             │
│         │  Confirmar senha *                      │             │
│         │  [••••••••••               ] [👁]       │             │
│         │                                         │             │
│         │  ┌──────────────────────────────────┐   │             │
│         │  │         [Criar conta]            │   │  ← primary  │
│         │  └──────────────────────────────────┘   │             │
│         │                                         │             │
│         │  [Já tenho conta — Entrar]              │             │
│         └─────────────────────────────────────────┘             │
│                      max-width: 480px                           │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Tabs` ou `ToggleGroup` para seleção PF/PJ (dinâmico — reorganiza campos)
- `Input` com validação inline on-blur para todos os campos
- `PasswordStrengthChecklist` (componente custom — 4 critérios visuais em tempo real)
- `Button` variant="default" largura total, desabilitado enquanto form inválido

**Estados:**
- **Loading:** Botão "Criar conta" desabilitado + label "Criando conta..."
- **Empty:** Formulário em branco, toggle em PF por default
- **Error:** Borda vermelha + mensagem específica abaixo do campo inválido; "Este e-mail já está cadastrado." inline
- **Populated:** Todos os campos válidos, checklist de senha verde, botão habilitado

---

### T-003 — E-mail de ativação pendente
**Rota:** `/ativacao/pendente` · **Role:** `PUBLIC` · **RFs:** RF-003, RF-004, RF-005

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │                               │                  │
│              │   [✉ ícone envelope 64px]     │  aria-hidden     │
│              │                               │                  │
│              │   Confirme seu e-mail         │  ← H1            │
│              │                               │                  │
│              │   Enviamos um link para       │                  │
│              │   [u***@email.com]            │                  │
│              │   Clique no link para ativar. │                  │
│              │                               │                  │
│              │  ┌───────────────────────┐    │                  │
│              │  │  [Reenviar e-mail]    │    │  ← secondary     │
│              │  │  (cooldown 60s)       │    │                  │
│              │  └───────────────────────┘    │                  │
│              │                               │                  │
│              │  COOLDOWN ATIVO:              │                  │
│              │  [Reenviar em 58s] disabled   │                  │
│              │                               │                  │
│              │  [Alterar e-mail] ← link      │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ [novo@email.com      ]  │  │ ← edição inline  │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Ícone ilustrativo envelope (aria-hidden)
- `Button` variant="outline" com countdown 60s (disabled durante cooldown)
- Campo inline de edição de e-mail (aparece ao clicar "Alterar e-mail")
- `Toast` para sucesso/erro do reenvio

**Estados:**
- **Loading:** Botão "Reenviar" desabilitado + label "Enviando..."
- **Empty:** N/A (tela sempre exibe e-mail mascarado)
- **Error:** Toast vermelho "Não foi possível reenviar. Tente novamente."; botão "Reenviar" offline desabilitado + tooltip "Sem conexão"
- **Populated:** E-mail mascarado exibido, botão reenvio habilitado (após cooldown)

---

### T-004 — Ativação de conta — sucesso
**Rota:** `/ativacao/sucesso` · **Role:** `PUBLIC` · **RFs:** RF-003

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │                               │                  │
│              │  [✓ checkmark verde 64px]     │  ← scale spring  │
│              │    animado (300ms)            │                  │
│              │                               │                  │
│              │  Conta ativada com sucesso!   │  ← H1            │
│              │                               │                  │
│              │  Você será redirecionado      │                  │
│              │  para a plataforma em         │                  │
│              │  3 segundos.                  │                  │
│              │                               │                  │
│              │  [3] → [2] → [1]             │  ← countdown     │
│              │                               │                  │
│              │  ┌───────────────────────┐    │                  │
│              │  │      [Ir agora]       │    │  ← primary       │
│              │  └───────────────────────┘    │                  │
│              │                               │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Ícone checkmark animado (scale spring, `--ease-spring`, 300ms)
- Contador regressivo visual (3→2→1)
- `Button` variant="default" ("Ir agora" — redireciona imediatamente para T-013)

**Estados:**
- **Loading:** N/A (ação instantânea pós-clique no link de ativação)
- **Empty:** N/A
- **Error:** N/A (se o link for inválido, redireciona para T-005)
- **Populated:** Tela de sucesso com countdown + botão "Ir agora"

---

### T-005 — Link de ativação expirado
**Rota:** `/ativacao/expirado` · **Role:** `PUBLIC` · **RFs:** RF-004

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │                               │                  │
│              │  [⚠ ícone aviso âmbar 64px]   │                  │
│              │                               │                  │
│              │  Link expirado                │  ← H1            │
│              │                               │                  │
│              │  O link de ativação expirou.  │                  │
│              │  Links são válidos por        │                  │
│              │  24 horas.                    │                  │
│              │                               │                  │
│              │  ┌───────────────────────┐    │                  │
│              │  │ [Solicitar novo link] │    │  ← primary       │
│              │  └───────────────────────┘    │                  │
│              │                               │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Ícone aviso âmbar (aria-hidden)
- `Button` variant="default" com estados loading/sucesso/erro

**Estados:**
- **Loading:** Botão "Solicitar novo link" desabilitado + label "Enviando..."
- **Empty:** N/A
- **Error:** Toast vermelho "Não foi possível enviar. Tente novamente."
- **Populated:** Após sucesso — toast verde "Novo link enviado para [email mascarado]." + botão volta ao estado inicial

---

### T-006 — Recuperação de senha — solicitação
**Rota:** `/recuperar-senha` · **Role:** `PUBLIC` · **RFs:** RF-006

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │                               │                  │
│              │  Recuperar senha              │  ← H1            │
│              │                               │                  │
│              │  E-mail *                     │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ [seu@email.com        ] │  │                  │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  ┌───────────────────────────┐│                  │
│              │  │ [Enviar link de recup.]  ││  ← primary       │
│              │  └───────────────────────────┘│                  │
│              │                               │                  │
│              │  MENSAGEM ÂMBAR (não-revealing)│                 │
│              │  "Se este e-mail estiver       │                  │
│              │   cadastrado, você receberá   │                  │
│              │   o link em breve."           │                  │
│              │                               │                  │
│              │  [← Voltar para o login]      │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Input` (type="email")
- `Button` variant="default" largura total
- `Alert` variant="warning" (âmbar) para mensagem não-revealing de segurança

**Estados:**
- **Loading:** Botão desabilitado + label "Enviando..."
- **Empty:** Campo e-mail vazio
- **Error:** Toast vermelho para erros genéricos; mensagem âmbar para e-mail não encontrado (não revela existência)
- **Populated:** Navega para T-007 ao submeter com sucesso

---

### T-007 — Recuperação de senha — confirmação
**Rota:** `/recuperar-senha/confirmacao` · **Role:** `PUBLIC` · **RFs:** RF-006

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │                               │                  │
│              │  [✉ ícone envelope 64px]      │                  │
│              │                               │                  │
│              │  Verifique sua caixa          │  ← H1            │
│              │  de entrada                   │                  │
│              │                               │                  │
│              │  Enviamos um link de          │                  │
│              │  recuperação para seu e-mail. │                  │
│              │  O link é válido por 1 hora.  │                  │
│              │                               │                  │
│              │  ┌───────────────────────┐    │                  │
│              │  │  [Reenviar e-mail]    │    │  ← secondary     │
│              │  │  (cooldown 60s)       │    │                  │
│              │  └───────────────────────┘    │                  │
│              │                               │                  │
│              │  [← Voltar para o login]      │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Ícone envelope (aria-hidden)
- `Button` variant="outline" com cooldown 60s (idêntico ao T-003)
- Link para T-001

**Estados:**
- **Loading:** Botão "Reenviar" desabilitado + label "Enviando..." durante reenvio
- **Empty:** N/A (tela estática)
- **Error:** Toast vermelho se reenvio falhar
- **Populated:** Instrução + cooldown ativo

---

### T-008 — Redefinição de senha
**Rota:** `/redefinir-senha?token=[token]` · **Role:** `PUBLIC` · **RFs:** RF-006

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │                               │                  │
│              │  Redefinir senha              │  ← H1            │
│              │                               │                  │
│              │  Nova senha *                 │                  │
│              │  [••••••••       ] [👁]        │                  │
│              │  ┌────────────────────────┐   │                  │
│              │  │ ○ 8+ caracteres       │   │  ← checklist     │
│              │  │ ○ 1 maiúscula         │   │    força senha   │
│              │  │ ○ 1 número            │   │                  │
│              │  │ ○ 1 especial          │   │                  │
│              │  └────────────────────────┘   │                  │
│              │                               │                  │
│              │  Confirmar nova senha *       │                  │
│              │  [••••••••       ] [👁]        │                  │
│              │                               │                  │
│              │  ┌──────────────────────────┐ │                  │
│              │  │    [Redefinir senha]     │ │  ← primary       │
│              │  └──────────────────────────┘ │                  │
│              │                               │                  │
│              └───────────────────────────────┘                  │
│                                                                  │
│  TOKEN INVÁLIDO/EXPIRADO (tela alternativa):                     │
│  "Este link de redefinição expirou ou já foi usado."             │
│  [Solicitar novo link] → navega para T-006                       │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Input` (type="password") × 2 com toggle visualizar
- `PasswordStrengthChecklist` (igual T-002)
- `Button` variant="default" habilitado apenas quando ambos os campos válidos
- Tela de token inválido (condicional — exibida se token ausente/expirado)

**Estados:**
- **Loading:** Botão "Redefinir senha" desabilitado + label "Redefinindo..."
- **Empty:** Campos em branco, botão desabilitado
- **Error:** Borda vermelha nos campos não correspondentes; tela de token inválido para link expirado
- **Populated:** Toast verde "Senha redefinida com sucesso!" + redirect para T-001 após 2s

---

### T-009 — Conta bloqueada temporariamente
**Rota:** `/conta-bloqueada` · **Role:** `PUBLIC` · **RFs:** RF-007

```javascript
┌─────────────────────────────────────────────────────────────────┐
│              FUNDO: --background (#FFFFFF)                       │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │                               │                  │
│              │  [🔒 cadeado vermelho 64px]   │                  │
│              │                               │                  │
│              │  Conta bloqueada              │  ← H1            │
│              │  temporariamente              │                  │
│              │                               │                  │
│              │  Você excedeu o número máximo │                  │
│              │  de tentativas. Tente         │                  │
│              │  novamente após:              │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │      14:58           │     │  ← Countdown     │
│              │  │  (hh:mm, 1s tick)   │     │    componente    │
│              │  └──────────────────────┘     │                  │
│              │                               │                  │
│              │  Liberado às 15:23            │  ← texto sec.    │
│              │                               │                  │
│              │  ┌───────────────────────┐    │                  │
│              │  │ [Voltar para o login] │    │  ← secondary     │
│              │  │  (disabled enquanto   │    │    desabilitado  │
│              │  │   contador ativo)     │    │    no countdown  │
│              │  └───────────────────────┘    │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Ícone cadeado vermelho (aria-hidden)
- `Countdown` component (hh:mm, atualização a cada segundo)
- `Button` variant="outline" desabilitado durante countdown, habilitado ao zerar
- Texto do horário exato de liberação (--muted-foreground)

**Estados:**
- **Loading:** N/A (tela carrega com countdown já ativo)
- **Empty:** N/A
- **Error:** N/A
- **Populated:** Countdown ativo + botão desabilitado; ao zerar: botão habilitado

---

### T-010 — Modal de expiração de sessão
**Rota:** `(overlay global)` · **Role:** `CEDENTE` · **RFs:** RF-009

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY: rgba(0,0,0,0.5) — sobre qualquer tela autenticada     │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="dialog" aria-modal=true  │                  │
│              │ focus trap ativo               │                  │
│              │                               │                  │
│              │  Sua sessão está prestes      │  ← H2            │
│              │  a expirar                    │                  │
│              │                               │                  │
│              │  Por segurança, sua sessão    │                  │
│              │  encerrará em:                │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │       04:58          │     │  ← Countdown     │
│              │  │  aria-live="off"     │     │    5min→0        │
│              │  └──────────────────────┘     │                  │
│              │                               │                  │
│              │  ┌───────────────────────┐    │                  │
│              │  │ [Continuar sessão]    │    │  ← primary       │
│              │  │  (focus inicial)      │    │    44px min      │
│              │  └───────────────────────┘    │                  │
│              │                               │                  │
│              │  ┌───────────────────────┐    │                  │
│              │  │  [Encerrar agora]     │    │  ← secondary     │
│              │  │  (ESC = este botão)   │    │    destrutivo    │
│              │  └───────────────────────┘    │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Dialog` (shadcn) com `role="dialog"`, `aria-modal="true"`, focus trap obrigatório
- `Countdown` component (mm:ss, 5min→0, `aria-live="off"`)
- `Button` variant="default" (primário, 44px mínimo) — foco inicial
- `Button` variant="outline" destrutivo — ESC aciona este botão
- `Toast` discreto "Sessão renovada." ao continuar

**Estados:**
- **Loading:** N/A (modal aparece instantaneamente)
- **Empty:** N/A
- **Error:** N/A
- **Populated:** Modal aberto com countdown ativo; ao zerar: redireciona para T-001

**Notas:** [DECISÃO APLICADA: DEC-001] ESC = "Encerrar agora" (não fecha o modal neutro — encerra sessão ativamente, pois a inação é o comportamento mais seguro).

---

## Módulo 2 — Global (Shell)

### T-011 — Shell da área logada — Sidebar
**Rota:** `/(authenticated)/[qualquer]` · **Role:** `CEDENTE` · **RFs:** RF-016, RF-017

```javascript
┌──────────────────────────────────────────────────────────────────────┐
│ LAYOUT BASE WEB — todas as telas autenticadas herdam este shell       │
│                                                                        │
│┌─────────────┬──────────────────────────────────────────────────────┐│
││  SIDEBAR    │  HEADER FIXO                                          ││
││  240px      │  [Logo]            [🔔 badge N] [Avatar ▼]           ││
││  --sidebar  │───────────────────────────────────────────────────── ││
││  (#FAFAFA)  │                                                       ││
││  ┌────────┐ │  CONTEÚDO PRINCIPAL (grid 12 colunas, max-w 1280px)  ││
││  │ Logo   │ │                                                       ││
││  └────────┘ │  [Área de conteúdo da tela específica]               ││
││             │                                                       ││
││  ● Dashboard│                                                       ││
││  ○ Meus Casos│                                                      ││
││  ○ Cad. Imóvel│                                                     ││
││  ○ Documentos│                                                      ││
││  ○ Propostas │                                                      ││
││  ○ Assinaturas                                                      ││
││  ○ Financeiro│                                                      ││
││  ○ Assist. IA│                                                      ││
││  ─────────  │                                                       ││
││  ○ Meu Perfil│                                                      ││
││             │                                                       ││
││  [Avatar]   │                                                       ││
││  Nome user  │                                                       ││
│└─────────────┴──────────────────────────────────────────────────────┘│
│                                                                        │
│ COLAPSADA (64px):                                                      │
│┌──────┬────────────────────────────────────────────────────────────┐  │
││ [≡]  │  HEADER + CONTEÚDO                                         │  │
││ [●]  │                                                             │  │
││ [○]  │                                                             │  │
││ [○]  │                                                             │  │
│└──────┴────────────────────────────────────────────────────────────┘  │
│                                                                        │
│ TABLET (drawer overlay via hamburger 44×44px no header):               │
│ Sidebar aparece como overlay da esquerda ao clicar ☰                  │
└──────────────────────────────────────────────────────────────────────┘

ITEM ATIVO: borda esquerda 3px --sidebar-primary + bg --sidebar-accent
ITEM HOVER: bg --sidebar-accent, text --sidebar-accent-foreground
```

**Componentes-chave:**
- `SidebarProvider` + `Sidebar` (shadcn) com NavigationMenu, 9 itens
- `NotificationBell` (badge numérico) no header
- `Avatar` + dropdown (Ver perfil / Sair) no header
- `Button` hamburger 44×44px (tablet) que abre drawer overlay
- Transição de colapso: `--duration-normal` (250ms), `--ease-in-out`

**Estados:**
- **Loading:** Skeleton de 3 itens na sidebar + header skeleton (logo + 2 placeholders)
- **Empty:** N/A (shell sempre presente em telas autenticadas)
- **Error:** Mensagem central "Não foi possível carregar a interface." + botão "Tentar novamente"
- **Populated:** Sidebar com itens, item ativo destacado, header com notificações e avatar

**Notas:** [DECISÃO APLICADA: DEC-002] 240px expandida / 64px colapsada conforme auditoria B04.

---

### T-012 — Shell mobile — Bottom Tab Bar
**Rota:** `/(authenticated)/[qualquer]` (mobile) · **Role:** `CEDENTE` · **RFs:** RF-120

```javascript
┌─────────────────────────────────────────────────────┐
│  LAYOUT BASE MOBILE — todas as telas autenticadas   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  CONTEÚDO PRINCIPAL (tela específica)       │   │
│  │                                             │   │
│  │                                             │   │
│  │                                             │   │
│  │                                             │   │
│  │                                             │   │
│  │                                             │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  BOTTOM TAB BAR (60px, safe-area-inset)     │   │
│  │                                             │   │
│  │  [🏠]    [📁]    [📄]    [🏷 ●]    [⋮⋮]  │   │
│  │  Dashboard Casos  Docs  Propostas  Mais     │   │
│  │  (ativo) (inativo)(inativo)(badge N)(inativo)│  │
│  │                                             │   │
│  └─────────────────────────────────────────────┘   │
│  safe-area-inset-bottom (iPhone home indicator)    │
└─────────────────────────────────────────────────────┘

BOTTOM SHEET "Mais" (ao tocar):
┌─────────────────────────────────┐
│  ████████████ (drag handle)     │
│  [🤖] Assistente IA             │
│  [✍] Assinaturas               │
│  [👤] Meu Perfil               │
│  ─────────────────────────────  │
│  [Sair]                         │
└─────────────────────────────────┘
```

**Componentes-chave:**
- `Tabs` bottom fixed com 5 itens (`role="tablist"`, cada tab `role="tab"`, `aria-selected`)
- Badge numérico vermelho em Propostas e item "Mais"
- `BottomSheet` com 4 itens adicionais (Assistente IA, Assinaturas, Meu Perfil, Sair)
- Scale press 0.95 em cada item ao toque

**Estados:**
- **Loading:** N/A (tab bar sempre visível em mobile autenticado)
- **Empty:** N/A
- **Error:** N/A (tab bar não carrega dados)
- **Populated:** Item ativo com ícone `--primary` + label bold; inativos com `--muted-foreground`

---

## Módulo 3 — Dashboard

### T-013 — Dashboard — com casos
**Rota:** `/(authenticated)/dashboard` · **Role:** `CEDENTE` · **RFs:** RF-019, RF-021, RF-022, RF-023

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Dashboard                         [Notificações 🔔] │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  CARDS DE RESUMO (grid 4 colunas desktop, 2 tablet)  │
│              │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│              │  │ Casos    │ │Pendências│ │Propostas │ │ Valor  │ │
│              │  │ Ativos   │ │          │ │ Recebidas│ │ Líq.   │ │
│              │  │   [3]    │ │  [2]     │ │   [1]    │ │R$XX.XXX│ │
│              │  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │  Casos Ativos                          [Ver todos →]  │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ [CaseCard] Rua X, 100 · Em Análise · [Próximo] │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ [CaseCard] Av Y, 200 · Formalização · [Assinar]│ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │  Próximos Passos                                      │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ 📋 Assinar Termo de Cadastro   [Ir para Assin.]│ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ 📄 Enviar documentos — Rua X   [Enviar]        │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │  Feed de Eventos Recentes                             │
│              │  · Proposta recebida — Av Y · há 2h                  │
│              │  · Documento verificado — Rua X · ontem              │
│              │  · Cadastro concluído — Rua X · há 3 dias            │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Card` (4 cards de resumo) com métricas — grid 4 colunas desktop, 2 tablet, 1 mobile
- `CaseCard` com StatusBadge, endereço, próximo passo e CTA contextual
- `FinancialSummary` (mini) no card de valor líquido
- Feed de eventos com timestamps relativos (--muted-foreground)

**Estados:**
- **Loading:** Skeleton dos 4 cards de resumo (retângulos 100% width, 80px height) + skeleton de 2 CaseCCards + skeleton de 3 itens no feed
- **Empty:** Redireciona para T-014 (sem casos)
- **Error:** Banner "Não foi possível carregar o dashboard." + botão "Tentar novamente"
- **Populated:** Cards com métricas reais, lista de casos, próximos passos e feed

---

### T-014 — Dashboard — primeiro acesso (sem casos)
**Rota:** `/(authenticated)/dashboard` (empty state) · **Role:** `CEDENTE` · **RFs:** RF-020

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Dashboard                                            │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │         ┌──────────────────────────────────────┐     │
│              │         │  Bem-vindo ao Repasse Seguro! 👋      │     │
│              │         │                                      │     │
│              │         │  Você ainda não tem imóveis          │     │
│              │         │  cadastrados. Comece agora:          │     │
│              │         │                                      │     │
│              │         │  CHECKLIST DE 3 PASSOS:              │     │
│              │         │  ○ 1. Cadastrar seu imóvel           │     │
│              │         │  ○ 2. Enviar documentos do dossiê    │     │
│              │         │  ○ 3. Assinar o Termo de Cadastro    │     │
│              │         │                                      │     │
│              │         │  ┌──────────────────────────────┐    │     │
│              │         │  │ [Cadastrar meu primeiro imóvel]│  │     │
│              │         │  └──────────────────────────────┘    │     │
│              │         │          (--primary, CTA)            │     │
│              │         └──────────────────────────────────────┘     │
│              │              card centralizado, max-w 480px          │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Card` centralizado com mensagem personalizada (nome do usuário via `--foreground`)
- Checklist 3 passos (ícones circle → check ao completar)
- `Button` variant="default" "Cadastrar meu primeiro imóvel" (navega para T-024)

**Estados:**
- **Loading:** Skeleton do card central (retângulo 100% width, 240px height)
- **Empty:** Este é o empty state do Dashboard (exibido quando não há casos)
- **Error:** Banner "Não foi possível carregar o dashboard." + retry
- **Populated:** Card com mensagem + checklist + CTA

---

## Módulo 4 — Meus Casos

### T-015 — Lista de Casos
**Rota:** `/(authenticated)/meus-casos` · **Role:** `CEDENTE` · **RFs:** RF-024, RF-025

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Meus Casos                          [+ Novo Imóvel] │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  FILTROS:                                             │
│              │  ┌──────────────────────┐ ┌─────────────────────┐   │
│              │  │ Status ▼             │ │ Cenário ▼           │   │
│              │  │ Todos / Em Análise / │ │ A / B / C / D / ... │   │
│              │  │ Formalização / etc   │ │                     │   │
│              │  └──────────────────────┘ └─────────────────────┘   │
│              │                                                       │
│              │  BUSCA:                                               │
│              │  ┌────────────────────────────────────────────────┐  │
│              │  │ [🔍 Buscar por nome ou endereço...           ] │  │
│              │  └────────────────────────────────────────────────┘  │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │  3 casos encontrados                                  │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ [CaseCard]                                      │ │
│              │  │ 📍 Rua das Flores, 100 — Apto 23               │ │
│              │  │ [Em Análise]  Cenário B  · Cadastrado em 10/03 │ │
│              │  │ → Próximo: Assinar Termo            [Ver caso →]│ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ [CaseCard]                                      │ │
│              │  │ 📍 Av. Brasil, 500 — Casa                      │ │
│              │  │ [Proposta Recebida]  Cenário C  · 15/03        │ │
│              │  │ → Proposta: R$ 120.000            [Ver caso →] │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  [← Anterior]  Página 1 de 1  [Próximo →]           │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Select` (shadcn) para filtro por status e cenário
- `Input` com ícone de busca (busca client-side por nome/endereço)
- `CaseCard` com StatusBadge, endereço, cenário, data, próximo passo e botão "Ver caso"
- `Pagination` (10 casos por página)

**Estados:**
- **Loading:** Skeleton de 3 CaseCards (retângulos 100% width, 80px height cada)
- **Empty:** "Você não tem casos cadastrados." + CTA "Cadastrar meu primeiro imóvel" (navega para T-024)
- **Error:** Banner "Não foi possível carregar seus casos." + retry
- **Populated:** Lista filtrada + paginação

---

### T-016 — Detalhe do Caso — Visão Geral
**Rota:** `/(authenticated)/meus-casos/[id]` · **Role:** `CEDENTE` · **RFs:** RF-025, RF-026, RF-027

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  ← Meus Casos                                        │
│  T-011       │  Rua das Flores, 100 — Apto 23   [Em Análise] [⋮]  │
│              │──────────────────────────────────────────────────────│
│              │  [Visão Geral] [Documentos] [Propostas] [Assinaturas]│
│              │  [Financeiro] [Histórico]                             │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│  DESKTOP (2 colunas):                                                │
│              │  ┌──────────────────────────┐ ┌───────────────────┐ │
│              │  │  ZONA 1 — Próximo Passo  │ │  ZONA 3           │ │
│              │  │  (--primary bg sutil)    │ │  Linha do Tempo   │ │
│              │  │  📋 Assinar o Termo      │ │                   │ │
│              │  │  de Cadastro             │ │  Hoje             │ │
│              │  │  [Ir para Assinaturas]   │ │  · Prop. recebida │ │
│              │  └──────────────────────────┘ │                   │ │
│              │                               │  Ontem            │ │
│              │  ┌──────────────────────────┐ │  · Doc verificado │ │
│              │  │  ZONA 2 — Financeiro     │ │                   │ │
│              │  │  FinancialSummary        │ │  10 mar           │ │
│              │  │  Cenário: B              │ │  · Cadastro conc. │ │
│              │  │  Estimado: R$ 87.000     │ │                   │ │
│              │  │  Data est.: Jun/2026     │ │  [Ver histórico   │ │
│              │  └──────────────────────────┘ │   completo →]     │ │
│              │                               └───────────────────┘ │
│              │                                                       │
│  AÇÕES CONDICIONAIS (kebab menu [⋮]):                               │
│              │  · Cancelar Caso (pré-Fechamento)                    │
│              │  · Solicitar Desistência (pós-Fechamento, 15 dias)   │
│              │  · Alterar Cenário (quando disponível)               │
└──────────────┴──────────────────────────────────────────────────────┘

MOBILE: coluna única — Zona 1 → Zona 2 → "Ver histórico completo" (colapsa Zona 3)
TABLET: coluna única
```

**Componentes-chave:**
- `Tabs` com 6 abas (Visão Geral, Documentos, Propostas, Assinaturas, Financeiro, Histórico)
- `StatusBadge` para status do caso
- `Card` Zona 1 (próximo passo com CTA contextual, --primary bg sutil)
- `FinancialSummary` Zona 2 (cenário, valor estimado, data)
- Timeline Zona 3 (eventos cronológicos, --muted-foreground)
- `DropdownMenu` (kebab ⋮) para ações condicionais

**Estados:**
- **Loading:** Skeleton das 3 zonas independentemente (retângulos proporcionais)
- **Empty:** N/A (caso sempre tem dados ao abrir)
- **Error:** Banner "Não foi possível carregar os detalhes." + retry em cada zona
- **Populated:** Layout 2 colunas desktop com todas as zonas preenchidas

**Notas:** [DECISÃO APLICADA: DEC-003] 2 colunas desktop para maximizar densidade de informação sem scroll.

---

### T-017 — Detalhe do Caso — Documentos
**Rota:** `/(authenticated)/meus-casos/[id]?tab=documentos` · **Role:** `CEDENTE` · **RFs:** RF-027, RF-056

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  ← Meus Casos · Rua das Flores, 100                  │
│  T-011       │  [Visão Geral] [Documentos ●] [Propostas] ...        │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  Documentos do Dossiê          2 de 6 enviados       │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ DocumentCard                                    │ │
│              │  │ RG ou CNH · [Pendente 🔴]     [Enviar ↑]       │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ DocumentCard                                    │ │
│              │  │ Comprovante de Residência · [Em Análise 🟡]     │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ DocumentCard                                    │ │
│              │  │ Contrato Original · [Verificado ✅]             │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ DocumentCard                                    │ │
│              │  │ Comprovante de Pgto · [Rejeitado 🔴]  [Reenviar]│ │
│              │  │ ⚠ Motivo: Arquivo ilegível                      │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  [Ver todos os documentos →] (navega para T-040)     │
│              │                                                       │
│  MOBILE: lista vertical + ícone câmera 📷 inline nos itens pendentes │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `DocumentCard` com StatusBadge por item (Pendente/Em Análise/Verificado/Rejeitado)
- Botão "Enviar" ou "Reenviar" inline por item
- Texto âmbar de motivo de rejeição abaixo do card rejeitado
- Link "Ver todos os documentos" → T-040

**Estados:**
- **Loading:** Skeleton de 6 linhas (retângulos 100% width, 48px cada)
- **Empty:** "Nenhum documento enviado ainda." + CTA "Enviar documentos" (navega para T-040)
- **Error:** Banner + retry
- **Populated:** Lista de DocumentCards com status real

---

### T-018 — Detalhe do Caso — Propostas
**Rota:** `/(authenticated)/meus-casos/[id]?tab=propostas` · **Role:** `CEDENTE` · **RFs:** RF-027, RF-049

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  ← Meus Casos · Rua das Flores, 100                  │
│  T-011       │  [Visão Geral] [Documentos] [Propostas ●] ...        │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  Propostas Ativas (1)                                 │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ ProposalCard                                    │ │
│              │  │ R$ 120.000  →  Você recebe: R$ 96.000 (verde)  │ │
│              │  │ [⏱ Expira em 2d 14h] ← Countdown              │ │
│              │  │ [Aceitar ✓] [Recusar ✗] [Contraproposta ↔]    │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │  Histórico de Propostas (2)                           │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ R$ 100.000 · [Recusada] · 12/03/2026           │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ R$ 95.000 · [Expirada] · 05/03/2026            │ │
│              │  └─────────────────────────────────────────────────┘ │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `ProposalCard` com valor proposto, valor líquido verde, Countdown, StatusBadge e ações
- Histórico como lista compacta com StatusBadge e data
- `Button` Aceitar/Recusar/Contraproposta (abrem T-037/T-038/T-039 respectivamente)

**Estados:**
- **Loading:** Skeleton de 2 cards (retângulos 100% width, 100px)
- **Empty (ativas):** "Nenhuma proposta recebida ainda. Seu imóvel está disponível para compradores qualificados."
- **Empty (histórico):** "Nenhuma proposta anterior."
- **Populated:** ProposalCards com dados reais

---

### T-019 — Detalhe do Caso — Assinaturas
**Rota:** `/(authenticated)/meus-casos/[id]?tab=assinaturas` · **Role:** `CEDENTE` · **RFs:** RF-027, RF-064

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  ← Meus Casos · Rua das Flores, 100                  │
│  T-011       │  [Visão Geral] [Documentos] [Propostas] [Assinaturas●]│
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  Assinaturas                                          │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ Termo de Cadastro                               │ │
│              │  │ Criado: 15/03/2026 · Prazo: 20/03/2026         │ │
│              │  │ [⏱ Expira em 5d] · [Aguardando assinatura 🟡] │ │
│              │  │                              [Assinar →]       │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ Instrumento de Cessão                           │ │
│              │  │ Criado: 16/03/2026 · Sem prazo                 │ │
│              │  │ [Assinado ✅] · 17/03/2026                     │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│  MOBILE (informativo — sem ZapSign):                                 │
│              │  ┌────────────────────────────────────────────────┐  │
│              │  │ ℹ Acesse via computador para assinar           │  │
│              │  └────────────────────────────────────────────────┘  │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Lista de documentos de assinatura com tipo, datas, prazo (Countdown se < 48h em âmbar), StatusBadge
- `Button` "Assinar" (web — abre T-044); mobile: mensagem "Acesse via computador"

**Estados:**
- **Loading:** Skeleton de 3 linhas (48px cada)
- **Empty:** "Nenhum documento de assinatura pendente."
- **Error:** Banner + retry
- **Populated:** Lista com status real, Countdown para prazos

---

### T-020 — Detalhe do Caso — Financeiro
**Rota:** `/(authenticated)/meus-casos/[id]?tab=financeiro` · **Role:** `CEDENTE` · **RFs:** RF-027, RF-071

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  ← Meus Casos · Rua das Flores, 100                  │
│  T-011       │  [Visão Geral] [Documentos] ... [Financeiro ●] ...   │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  PRÉ-FORMALIZAÇÃO (estado informativo):               │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ EscrowPanel                                     │ │
│              │  │ ℹ Conta Escrow: Não iniciada                   │ │
│              │  │                                                 │ │
│              │  │ "O valor será depositado após a assinatura      │ │
│              │  │  do Contrato de Cessão."                        │ │
│              │  │                                                 │ │
│              │  │ VALORES ESTIMADOS (--muted background):         │ │
│              │  │  Cenário B · Estimado: R$ 96.000               │ │
│              │  │                                                 │ │
│              │  │ [Ver simulação de cenários →]                   │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  ESCROW ATIVO (estado pós-formalização):             │
│              │  → ver T-046 (EscrowPanel ativo)                     │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `EscrowPanel` (condicional: pré-formalização vs ativo)
- Valores estimados em card `--muted` background (somente leitura)
- Link "Ver simulação" → T-026/T-031

**Estados:**
- **Loading:** Skeleton do painel EscrowPanel
- **Empty:** N/A (sempre exibe estado atual)
- **Error:** N/A (falha não bloqueia a aba — exibe mensagem e retry inline)
- **Populated:** EscrowPanel com estado correto do caso

---

### T-021 — Detalhe do Caso — Histórico de Eventos
**Rota:** `/(authenticated)/meus-casos/[id]?tab=historico` · **Role:** `CEDENTE` · **RFs:** RF-027, RF-084, RF-085

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  ← Meus Casos · Rua das Flores, 100                  │
│  T-011       │  ... [Financeiro] [Histórico ●]                       │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  Histórico de Eventos                                 │
│              │                                                       │
│              │  [Atualizado em 22/03 12:00] ← offline badge         │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ 📋 Proposta aceita                              │ │
│              │  │ 22/03/2026 10:23   · há 2 horas                │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ ✅ Documento verificado: Contrato Original      │ │
│              │  │ 21/03/2026 16:40   · ontem                      │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ 📄 Cadastro concluído                           │ │
│              │  │ 15/03/2026 09:10   · há 7 dias                 │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  [Carregar mais eventos] ← paginação (50/pág)        │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Timeline vertical (lista de eventos imutáveis) com ícone de tipo, descrição e timestamps
- Timestamps absolutos + relativos (tooltip para o absoluto no hover)
- `Badge` "Atualizado em [data/hora]" quando offline (cache)
- Paginação (50 eventos por página) ou scroll infinito

**Estados:**
- **Loading:** Skeleton de 10 linhas (32px cada)
- **Empty:** "Nenhum evento registrado ainda." (apenas por robustez — improvável)
- **Error:** Banner + retry
- **Populated:** Lista cronológica com todos os eventos do caso

---

### T-022 — Modal de Cancelamento de Caso — Simples
**Rota:** `(overlay em /(authenticated)/meus-casos/[id])` · **Role:** `CEDENTE` · **RFs:** RF-075

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="dialog" · focus trap     │                  │
│              │                               │                  │
│              │  Cancelar caso                │  ← H2            │
│              │  Rua das Flores, 100 — Apto 23│  ← subtítulo     │
│              │                               │                  │
│              │  PASSO 1 — MOTIVO:            │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ Motivo do cancelamento* │  │                  │
│              │  │ [Select ▼            ]  │  │                  │
│              │  │  Mudei de ideia         │  │                  │
│              │  │  Encontrei outra solução│  │                  │
│              │  │  Informações incorretas │  │                  │
│              │  │  Outro → [textarea]     │  │                  │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  [Continuar →]  [Cancelar]    │                  │
│              │                               │                  │
│              │  PASSO 2 — CONFIRMAÇÃO:       │                  │
│              │  "Você está prestes a         │                  │
│              │   cancelar o caso [end.].     │                  │
│              │   Esta ação é irreversível."  │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │[Confirmar cancel.]   │     │  ← --destructive │
│              │  └──────────────────────┘     │                  │
│              │  [← Voltar]                   │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
ESC: fecha modal (Passo 1) ou volta ao Passo 1 (Passo 2)
```

**Componentes-chave:**
- `Dialog` com focus trap, `role="dialog"`, `aria-modal="true"`
- `Select` com 4 opções (+ `Textarea` condicional para "Outro")
- Passo 2: resumo + botão "Confirmar cancelamento" (`--destructive`)
- `Button` "Voltar" entre passos

**Estados:**
- **Loading:** Botão "Confirmar cancelamento" desabilitado + label "Cancelando..."
- **Empty:** N/A (modal sempre abre com dados do caso)
- **Error:** Toast vermelho "Não foi possível cancelar. Tente novamente."
- **Populated:** Passo 1: dropdown de motivo; Passo 2: resumo + confirmação destrutiva

---

### T-023 — Modal de Cancelamento de Caso — Com Escrow
**Rota:** `(overlay)` · **Role:** `CEDENTE` · **RFs:** RF-076

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="dialog" · focus trap     │                  │
│              │                               │                  │
│              │  Cancelar caso                │  ← H2            │
│              │  (Conta Escrow ativa)         │                  │
│              │                               │                  │
│              │  PASSO 1 — MOTIVO:            │                  │
│              │  (idêntico ao T-022)          │                  │
│              │                               │                  │
│              │  [Continuar →]                │                  │
│              │                               │                  │
│              │  PASSO 2 — AVISO ESCROW:      │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ ⚠ ATENÇÃO              │  │  ← alert         │
│              │  │ Há uma Conta Escrow     │  │    --destructive │
│              │  │ ativa. O cancelamento   │  │    sutil         │
│              │  │ iniciará o estorno.     │  │                  │
│              │  │ Prazo: até 15 dias úteis│  │                  │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  ☐ Compreendo e aceito as     │                  │
│              │    condições do estorno.*     │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │[Confirmar cancel.]   │     │  ← destrutivo    │
│              │  │(habilitado após ☑)   │     │                  │
│              │  └──────────────────────┘     │                  │
│              │  [← Voltar]                   │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Idêntico ao T-022 + `Alert` variant="destructive" sutil no Passo 2
- `Checkbox` obrigatório para habilitar botão "Confirmar cancelamento"

**Estados:**
- **Loading / Error / Populated:** Idênticos ao T-022
- **Empty:** N/A


---

## Módulo 5 — Cadastro de Imóvel (Wizard)

### T-024 — Wizard — Etapa 1: Dados do Imóvel
**Rota:** `/(authenticated)/cadastrar-imovel/etapa-1` · **Role:** `CEDENTE` · **RFs:** RF-028, RF-029

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Cadastrar Imóvel                                     │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  STEPPER (5 etapas):                                  │
│              │  [●1 Dados] ─── [○2 Valores] ─── [○3 Simul.]        │
│              │                 ─── [○4 Cenário] ─── [○5 Revisão]   │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │                                                       │
│              │  CEP *                                                │
│              │  [00000-000          ] → preenche endereço auto      │
│              │                                                       │
│              │  Endereço *                                           │
│              │  [Rua das Flores                  ]                   │
│              │  Número *       Complemento                          │
│              │  [100       ]   [Apto 23 (opcional)]                 │
│              │                                                       │
│              │  Bairro *           Cidade *         Estado *        │
│              │  [Meireles  ]       [Fortaleza  ]    [CE]            │
│              │                                                       │
│              │  ALERTA DUPLICIDADE (condicional):                   │
│              │  ⚠ Você já tem um caso ativo para este endereço.   │
│              │  [Ver caso existente →]                              │
│              │                                                       │
│              │  Construtora *                                        │
│              │  [Nome da construtora             ]                   │
│              │                                                       │
│              │  Tipo de Imóvel *                                     │
│              │  [Select: Apartamento ▼           ]                   │
│              │                                                       │
│              │  ┌─────────────────┐    ┌───────────┐               │
│              │  │ [Próxima etapa] │    │ [Cancelar]│               │
│              │  └─────────────────┘    └───────────┘               │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Stepper` horizontal (5 etapas, etapa 1 ativa)
- `Input` CEP com autocomplete (preenche endereço ao digitar CEP completo)
- `Input` campos de endereço (rua, número, complemento, bairro, cidade, estado)
- `Alert` âmbar condicional para duplicidade de endereço
- `Select` para tipo de imóvel (5 opções)
- `Button` "Próxima etapa" (primário) + "Cancelar" (secundário)

**Estados:**
- **Loading:** Spinner inline 16px no campo CEP durante busca de endereço; validação assíncrona de duplicidade com indicador inline
- **Empty:** Campos em branco, botão "Próxima etapa" desabilitado
- **Error:** Borda vermelha + mensagem específica abaixo do campo inválido; alerta âmbar para duplicidade
- **Populated:** Todos os campos válidos, botão "Próxima etapa" habilitado; ao clicar: auto-save dos dados

**Notas:** Cancelar confirma descarte se houver dados preenchidos. Auto-save ao avançar (não ao digitar).

---

### T-025 — Wizard — Etapa 2: Valores do Contrato
**Rota:** `/(authenticated)/cadastrar-imovel/etapa-2` · **Role:** `CEDENTE` · **RFs:** RF-028

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Cadastrar Imóvel                                     │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  [✓1 Dados] ─── [●2 Valores] ─── [○3 Simul.]       │
│              │                 ─── [○4 Cenário] ─── [○5 Revisão]  │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │                                                       │
│              │  Valor original do contrato *                        │
│              │  ┌──────────────────────────────────────────────┐   │
│              │  │ R$ [150.000,00                           ]   │   │
│              │  └──────────────────────────────────────────────┘   │
│              │  "Valor total do contrato firmado com a              │
│              │   construtora, sem descontos."                       │
│              │                                                       │
│              │  Total pago até hoje *                               │
│              │  ┌──────────────────────────────────────────────┐   │
│              │  │ R$ [80.000,00                            ]   │   │
│              │  └──────────────────────────────────────────────┘   │
│              │  "Quanto você pagou ao total desde o início,         │
│              │   incluindo parcelas e entrada."                     │
│              │                                                       │
│              │  Saldo devedor *                                     │
│              │  ┌──────────────────────────────────────────────┐   │
│              │  │ R$ [70.000,00 ← calculado] [Editar]      ]  │   │
│              │  └──────────────────────────────────────────────┘   │
│              │  "Sugestão calculada: valor original − total pago." │
│              │                                                       │
│              │  ┌─────────────────┐    ┌──────────────────────┐    │
│              │  │ [Próxima etapa] │    │ [← Etapa anterior]   │    │
│              │  └─────────────────┘    └──────────────────────┘    │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Input` currency mask (R$) × 3 com validações cruzadas
- Helper text abaixo de cada campo (--muted-foreground, text-sm)
- Saldo devedor pré-calculado com opção de edição manual
- `Button` "Próxima etapa" + "Etapa anterior"

**Estados:**
- **Loading:** N/A (campos locais, sem fetch)
- **Empty:** Campos em branco, sugestão de saldo zerada
- **Error:** Saldo negativo ou superior ao valor original: borda vermelha + mensagem inline
- **Populated:** Valores preenchidos e válidos, saldo calculado automaticamente

---

### T-026 — Wizard — Etapa 3: Simulador de Cenários
**Rota:** `/(authenticated)/cadastrar-imovel/etapa-3` · **Role:** `CEDENTE` · **RFs:** RF-030, RF-050, RF-051

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Cadastrar Imóvel                                     │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  [✓1] ─── [✓2] ─── [●3 Simulação] ─── [○4] ─── [○5]│
│              │                                                       │
│              │  BARRA DE PROGRESSO (10s timer, --primary, topo):    │
│              │  ████████████░░░░░░░░░░░░ 8s restantes               │
│              │                                                       │
│              │  Simulação de Cenários                                │
│              │  Calculando seu retorno estimado...                  │
│              │                                                       │
│              │  CARDS (skeleton durante 10s, fade-in staggered):    │
│              │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│              │  │ CENÁRIO A│ │ CENÁRIO B│ │ CENÁRIO C│ │CENÁRIO D│ │
│              │  │ "Rápido" │ │"Padrão" │ │"Premium" │ │"Máx"   │ │
│              │  │          │ │          │ │          │ │         │ │
│              │  │R$ 70.000 │ │R$ 84.000 │ │R$ 91.000 │ │R$98.000│ │
│              │  │Comissão: │ │Comissão: │ │Comissão: │ │Comiss: │ │
│              │  │ R$ 0     │ │R$ 12.000 │ │R$ 14.000 │ │R$16.000│ │
│              │  │Você rec: │ │Você rec: │ │Você rec: │ │Vc rec: │ │
│              │  │R$ 70.000 │ │R$ 72.000 │ │R$ 77.000 │ │R$82.000│ │
│              │  │(verde)   │ │(verde)   │ │(verde)   │ │(verde) │ │
│              │  │[ℹ dific.]│ │[ℹ dific.]│ │[ℹ dific.]│ │[ℹ dif.]│ │
│              │  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│              │                                                       │
│              │  ┌─────────────────────────────────────┐            │
│              │  │ [Próxima etapa] ← habilitado após 10s│           │
│              │  └─────────────────────────────────────┘            │
│              │  [← Etapa anterior]                                  │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Barra de progresso fina no topo da área de cards (não no botão) — --primary, 10s
- `Card` × 4 cenários (A/B/C/D) com skeleton retangular durante os 10s de cálculo
- Fade-in staggered (100ms entre cards) após skeleton
- `Tooltip` no ícone de dificuldade de venda (ℹ)
- `Button` "Próxima etapa" bloqueado nos primeiros 10s

**Estados:**
- **Loading:** Skeleton dos 4 cards (retângulos 160px × 120px) + barra de progresso ativa
- **Empty:** N/A
- **Error:** Banner vermelho "Não foi possível calcular os cenários." + botão "Revisar valores" (volta para T-025)
- **Populated:** 4 cards com valores reais, fade-in staggered, botão habilitado após 10s

---

### T-027 — Wizard — Etapa 4: Escolha do Cenário
**Rota:** `/(authenticated)/cadastrar-imovel/etapa-4` · **Role:** `CEDENTE` · **RFs:** RF-031

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Cadastrar Imóvel                                     │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  [✓1] ─── [✓2] ─── [✓3] ─── [●4 Cenário] ─── [○5] │
│              │                                                       │
│              │  Escolha seu cenário de retorno                      │
│              │                                                       │
│              │  role="radiogroup" aria-required="true"              │
│              │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│              │  │ ○ A      │ │ ○ B      │ │ ○ C      │ │ ○ D    │ │
│              │  │ "Rápido" │ │"Padrão" │ │"Premium" │ │"Máx"   │ │
│              │  │R$ 70.000 │ │R$ 84.000 │ │R$ 91.000 │ │R$98.000│ │
│              │  │Vc rec:   │ │Vc rec:   │ │Vc rec:   │ │Vc rec: │ │
│              │  │R$ 70.000 │ │R$ 72.000 │ │R$ 77.000 │ │R$82.000│ │
│              │  │(role=    │ │(role=    │ │(role=    │ │(role=  │ │
│              │  │ radio)   │ │ radio)   │ │ radio)   │ │ radio) │ │
│              │  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
│              │                                                       │
│              │  CARD SELECIONADO: borda --primary 2px + bg sutil    │
│              │  scale spring 300ms ao selecionar                    │
│              │                                                       │
│              │  ┌────────────────────────────────────────────────┐  │
│              │  │ ☐ Confirmo que li e compreendo as condições  *│  │
│              │  │   do cenário escolhido. (pulsante 3s)          │  │
│              │  └────────────────────────────────────────────────┘  │
│              │                                                       │
│              │  ┌─────────────────────────────────────┐            │
│              │  │ [Próxima etapa] ← habilitado após   │            │
│              │  │   seleção + checkbox                │            │
│              │  └─────────────────────────────────────┘            │
│              │  [← Etapa anterior]                                  │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `RadioGroup` (4 cards como radio buttons — sem pré-seleção)
- Checkbox pulsante (animação `--ease-spring` por 3s na primeira exibição)
- `Button` "Próxima etapa" habilitado apenas após seleção + checkbox marcado

**Estados:**
- **Loading:** N/A (dados vêm do T-026, já calculados)
- **Empty:** Cards sem seleção, checkbox desmarcado, botão desabilitado
- **Error:** Tentativa de avançar sem selecionar: shake animation no RadioGroup + mensagem "Selecione um cenário para continuar."
- **Populated:** Card selecionado com borda --primary, checkbox marcado, botão habilitado

---

### T-028 — Wizard — Etapa 5: Revisão e Confirmação
**Rota:** `/(authenticated)/cadastrar-imovel/etapa-5` · **Role:** `CEDENTE` · **RFs:** RF-034

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Cadastrar Imóvel                                     │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  [✓1] ─── [✓2] ─── [✓3] ─── [✓4] ─── [●5 Revisão] │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │  CENÁRIO ESCOLHIDO (H1 visual — destaque máximo):    │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │  Cenário B — Padrão                             │ │
│              │  │  Você receberá: R$ 72.000                       │ │
│              │  │  (--primary border 2px, fonte maior, verde)     │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │  Dados do Imóvel ▼ (colapsável)      [Editar etapa 1]│
│              │  · Rua das Flores, 100 — Apto 23                     │
│              │  · Construtora: ABC · Tipo: Apartamento              │
│              │                                                       │
│              │  Valores ▼ (colapsável)               [Editar etapa 2]│
│              │  · Valor original: R$ 150.000                        │
│              │  · Total pago: R$ 80.000                             │
│              │  · Saldo devedor: R$ 70.000                          │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │                                                       │
│              │  ┌──────────────────────────────────────────────┐   │
│              │  │         [Confirmar Cadastro]                 │   │  ← primary
│              │  └──────────────────────────────────────────────┘   │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Card de destaque (cenário + valor líquido) com --primary border, fonte maior — é o H1 visual
- Seções colapsáveis (`Collapsible` shadcn) para Dados do Imóvel e Valores
- Botão "Editar" em cada seção (navega diretamente para a etapa correspondente preservando dados)
- `Button` "Confirmar Cadastro" largura total (primário)

**Estados:**
- **Loading:** Botão "Confirmar Cadastro" desabilitado + label "Criando caso..." + overlay de loading sobre o stepper
- **Empty:** N/A (dados sempre presentes ao chegar na etapa 5)
- **Error:** Toast vermelho "Não foi possível criar o caso. Tente novamente." + botão reativo
- **Populated:** Resumo completo + botão ativo → sucesso navega para T-029

---

### T-029 — Cadastro concluído — Tela de Sucesso
**Rota:** `/(authenticated)/cadastrar-imovel/sucesso` · **Role:** `CEDENTE` · **RFs:** RF-034

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │                                                       │
│  T-011       │                                                       │
│              │         ┌──────────────────────────────────────┐     │
│              │         │                                      │     │
│              │         │  [✓ checkmark verde 80px animado]    │     │
│              │         │   scale spring bouncy                │     │
│              │         │                                      │     │
│              │         │  Imóvel cadastrado                   │     │
│              │         │  com sucesso!                        │     │
│              │         │                                      │     │
│              │         │  Rua das Flores, 100 — Apto 23       │     │
│              │         │  (subtítulo com endereço)            │     │
│              │         │                                      │     │
│              │         │  Próximos passos:                    │     │
│              │         │                                      │     │
│              │         │  ┌──────────────┐ ┌───────────────┐  │     │
│              │         │  │ ✍ Assinar   │ │ 📄 Enviar     │  │     │
│              │         │  │   Termo de  │ │   documentos  │  │     │
│              │         │  │   Cadastro  │ │   do dossiê   │  │     │
│              │         │  │[Ir p/ Assin]│ │ [Ir p/ Docs]  │  │     │
│              │         │  └──────────────┘ └───────────────┘  │     │
│              │         │                                      │     │
│              │         │  [Ver meu caso →]                    │     │
│              │         └──────────────────────────────────────┘     │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Ícone checkmark verde 80px animado (scale spring bouncy)
- Dois `Card` de próximos passos lado a lado (Assinaturas → T-043; Documentos → T-040)
- Link "Ver meu caso" → T-016

**Estados:**
- **Loading:** N/A (ação já concluída ao chegar aqui)
- **Empty:** N/A
- **Error:** N/A
- **Populated:** Tela de confirmação com ações de próximos passos

---

### T-030 — Banner de Rascunho Salvo
**Rota:** `/(authenticated)/cadastrar-imovel` (banner sticky) · **Role:** `CEDENTE` · **RFs:** RF-032

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Cadastrar Imóvel                                     │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ ⚠ Você tem um rascunho salvo para              │ │
│              │  │   [Rua das Flores, 100]. Deseja continuar?      │ │
│              │  │                                                 │ │
│              │  │  [Continuar →]    [Descartar ✗]               │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  BANNER: --chart-1 bg sutil (âmbar), sticky topo     │
│              │                                                       │
│              │  LOADING AO CONTINUAR:                               │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ Carregando rascunho...                          │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  CONFIRM DESCARTAR (alert):                          │
│              │  "Descartar rascunho? Esta ação é irreversível."     │
│              │  [Sim, descartar] (--destructive)  [Cancelar]       │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Banner sticky `--chart-1` (âmbar sutil) no topo da área de conteúdo
- `Button` "Continuar" (primário) + "Descartar" (secundário)
- `AlertDialog` para confirmação de descarte com botão destrutivo

**Estados:**
- **Loading:** Banner substitui botões por "Carregando rascunho..." com spinner inline
- **Empty:** N/A (banner só aparece quando há rascunho)
- **Error:** Toast vermelho "Não foi possível carregar o rascunho." + botão "Começar do zero"
- **Populated:** Banner com endereço mascarado e ações

---

## Módulo 6 — Cenários e Escalonamento

### T-031 — Tela de Escalonamento — Simulação Comparativa
**Rota:** `/(authenticated)/meus-casos/[id]/escalonamento` · **Role:** `CEDENTE` · **RFs:** RF-036, RF-039-A

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  ← Voltar ao caso                                     │
│  T-011       │  Alterar seu cenário                                  │
│              │──────────────────────────────────────────────────────│
│              │  Você pode escalar para um cenário com maior retorno. │
│              │                                                       │
│              │  MÚLTIPLOS ESCALONAMENTOS (RF-039-A):                 │
│              │  Selecione o cenário-alvo:                            │
│              │  [○ Cenário C]  [● Cenário D]  ← seletor tabs/select │
│              │                                                       │
│              │  ─────────────────────────────────────────────────── │
│              │                                                       │
│              │  CARDS COMPARATIVOS (desktop: lado a lado):           │
│              │  ┌────────────────────┐   ┌────────────────────────┐ │
│              │  │ CENÁRIO ATUAL      │   │ NOVO CENÁRIO PROPOSTO  │ │
│              │  │ (--muted bg)       │   │ (--primary border)     │ │
│              │  │                    │   │                        │ │
│              │  │ Cenário B          │   │ Cenário D              │ │
│              │  │ Você recebe:       │   │ Você receberá:         │ │
│              │  │ R$ 72.000          │   │ R$ 82.000              │ │
│              │  │ Comissão: R$12.000 │   │ Comissão: R$16.000     │ │
│              │  └────────────────────┘   └────────────────────────┘ │
│              │                                                       │
│              │         ↑ R$ 10.000 a mais (verde, seta)             │
│              │                                                       │
│              │  ┌────────────────────────────────────┐              │
│              │  │    [Solicitar escalonamento]       │  ← primary   │
│              │  └────────────────────────────────────┘              │
│              │  [← Voltar]                                           │
│              │                                                       │
│  MOBILE: cards empilhados (coluna única), seletor no topo           │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Seletor de cenário-alvo (Tabs ou Select) para múltiplos escalonamentos disponíveis (RF-039-A)
- 2 `Card` comparativos: atual (--muted) vs proposto (--primary border)
- Componente de diferença financeira: seta verde + "R$ X a mais"
- `Button` "Solicitar escalonamento" (abre T-032)

**Estados:**
- **Loading:** Skeleton dos 2 cards comparativos
- **Empty:** N/A
- **Error:** Banner vermelho "Não foi possível calcular." + retry
- **Populated:** Cards com dados reais + diferença financeira

**Notas:** [DECISÃO APLICADA: DEC-004] Seletor de cenário-alvo adicionado para casos com múltiplos escalonamentos disponíveis simultaneamente (RF-039-A).

---

### T-032 — Modal de Confirmação de Escalonamento
**Rota:** `(overlay em T-031)` · **Role:** `CEDENTE` · **RFs:** RF-036

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="dialog" · focus trap     │                  │
│              │                               │                  │
│              │  Confirmar escalonamento      │  ← H2            │
│              │                               │                  │
│              │  Você está prestes a escalar  │                  │
│              │  para o Cenário D.            │                  │
│              │  Esta ação é irreversível.    │                  │
│              │                               │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ [Cenário B] → [Cenário D]│  │  ← card resumo  │
│              │  │ R$ 72.000  →  R$ 82.000  │  │    com seta    │
│              │  │ +R$ 10.000               │  │                  │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  ☐ Confirmo que compreendo   │                  │
│              │    as condições do escal.*   │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │    [Confirmar]       │     │  ← primary       │
│              │  │  (após checkbox)     │     │                  │
│              │  └──────────────────────┘     │                  │
│              │  [Cancelar]                   │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
ESC: cancela e fecha o modal
```

**Componentes-chave:**
- `Dialog` com focus trap
- Card resumo de escalonamento (atual → novo + diferença)
- `Checkbox` obrigatório para habilitar "Confirmar"
- `Button` "Confirmar" (primário, habilitado após checkbox)

**Estados:**
- **Loading:** Botão "Confirmar" desabilitado + label "Solicitando..."
- **Empty:** N/A
- **Error:** Toast vermelho + modal permanece aberto
- **Populated:** Resumo + checkbox + confirmação → toast "Escalonamento solicitado!" + status atualiza

---

### T-033 — Modal de Escalonamento Enfileirado
**Rota:** `(overlay informativo)` · **Role:** `CEDENTE` · **RFs:** RF-037

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="alertdialog"             │                  │
│              │                               │                  │
│              │  [⏱ ícone relógio âmbar]      │                  │
│              │                               │                  │
│              │  Pedido enfileirado            │  ← H2            │
│              │                               │                  │
│              │  Você tem uma proposta em     │                  │
│              │  análise. Seu pedido de       │                  │
│              │  escalonamento foi registrado │                  │
│              │  e será processado assim que  │                  │
│              │  a proposta for resolvida.    │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │ [Cancelar enfileiram.]│     │  ← destructive   │
│              │  │ (secundário, confirma)│     │    text-only     │
│              │  └──────────────────────┘     │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │     [Entendido]      │     │  ← primary       │
│              │  └──────────────────────┘     │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `AlertDialog` (`role="alertdialog"`) — informativo, não de ação crítica imediata
- Ícone de relógio âmbar
- `Button` "Cancelar enfileiramento" (text-only destrutivo — confirma antes de cancelar)
- `Button` "Entendido" (primário — fecha modal)

**Estados:**
- **Loading:** N/A (modal informativo)
- **Empty:** N/A
- **Error:** N/A
- **Populated:** Mensagem + 2 ações

---

### T-034 — Modal de Subida de Cenário Bloqueada
**Rota:** `(overlay informativo)` · **Role:** `CEDENTE` · **RFs:** RF-040

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="alertdialog"             │                  │
│              │                               │                  │
│              │  [🚫 ícone bloqueio vermelho]  │                  │
│              │                               │                  │
│              │  Não é possível subir         │  ← H2            │
│              │  de cenário                   │                  │
│              │                               │                  │
│              │  Este imóvel não se qualifica │                  │
│              │  para um cenário superior com │                  │
│              │  base nos valores declarados. │                  │
│              │                               │                  │
│              │  ─────────────────────────── │                  │
│              │  "Se desejar tentar com       │                  │
│              │   valores atualizados, você   │                  │
│              │   pode cancelar o caso e      │                  │
│              │   recadastrá-lo."             │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │  [Cancelar o caso]   │     │  ← destrutivo    │
│              │  │ (abre T-022 ou T-023)│     │                  │
│              │  └──────────────────────┘     │                  │
│              │  [Fechar]  ← primary           │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `AlertDialog` informativo com ícone de bloqueio vermelho
- `Button` "Cancelar o caso" (destrutivo — abre T-022 ou T-023 conforme status do Escrow)
- `Button` "Fechar" (primário)

**Estados:**
- **Loading:** N/A
- **Empty:** N/A
- **Error:** N/A
- **Populated:** Mensagem de bloqueio + ações

---

## Módulo 7 — Propostas e Negociação

### T-035 — Lista de Propostas Ativas
**Rota:** `/(authenticated)/propostas` · **Role:** `CEDENTE` · **RFs:** RF-041, RF-042, RF-043, RF-048

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Propostas Ativas (3)                                 │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│  DESKTOP: split view (T-035 + T-036 lado a lado)                    │
│              │                                                       │
│  LISTA:      │  ┌─────────────────────────────────────────────────┐ │
│              │  │ ProposalCard — URGENTE ← borda vermelha          │ │
│              │  │ Rua das Flores, 100                             │ │
│              │  │ R$ 120.000 → Você recebe: R$ 96.000 (verde)    │ │
│              │  │ [⏱ Expira em 18h] ← vermelho pulsante         │ │
│              │  │ Badge: [URGENTE 🔴]                             │ │
│              │  │ [Aceitar] [Recusar] [Contraproposta]            │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ ProposalCard                                    │ │
│              │  │ Av. Brasil, 500                                 │ │
│              │  │ R$ 95.000 → Você recebe: R$ 76.000             │ │
│              │  │ [⏱ Expira em 3d 2h]                           │ │
│              │  │ [Aceitar] [Recusar] [Contraproposta]            │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│  EMPTY STATE:│  ┌─────────────────────────────────────────────────┐ │
│              │  │ [ícone ilustrativo]                             │ │
│              │  │ Nenhuma proposta recebida ainda.                │ │
│              │  │ Seu imóvel está disponível para compradores     │ │
│              │  │ qualificados.                                   │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│  MOBILE: cards em coluna única; ações via bottom sheet ao tocar     │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `ProposalCard` com valor proposto, valor líquido verde, Countdown (vermelho pulsante nas últimas 24h), StatusBadge
- Badge "URGENTE" e borda vermelha para propostas com < 24h
- Badge numérico no item "Propostas" da sidebar
- Desktop: split view (T-035 lista + T-036 detalhe lateral) — DA-004
- Mobile: bottom sheet com ações ao tocar no card

**Estados:**
- **Loading:** Skeleton de 2 ProposalCards (100% width, 120px cada)
- **Empty:** Mensagem + ícone ilustrativo
- **Error:** Banner + retry
- **Populated:** Lista ordenada por valor decrescente

---

### T-036 — Detalhe da Proposta
**Rota:** `/(authenticated)/propostas/[id]` · **Role:** `CEDENTE` · **RFs:** RF-041, RF-052

```javascript
DESKTOP (painel lateral direito — split view com T-035):
┌──────────────────────────────────────────────────────┐
│  ← Propostas                                         │
│                                                      │
│  HIERARQUIA VISUAL:                                  │
│                                                      │
│  Você recebe:                                        │
│  R$ 96.000           ← H1 visual (verde bold)        │
│                                                      │
│  ────────────────────────────────────────────────── │
│  Valor proposto pelo comprador:  R$ 120.000          │
│                                                      │
│  Comissão da plataforma ▼ (colapsável por padrão)   │
│  ┌──────────────────────────────────────────────┐   │
│  │ 20% sobre valor proposto = R$ 24.000        │   │
│  └──────────────────────────────────────────────┘   │
│                                                      │
│  Prazo:  [⏱ Expira em 2d 18h]  ← Countdown         │
│                                                      │
│  ────────────────────────────────────────────────── │
│  Histórico de negociação:                           │
│                                                      │
│  22/03 09:00  Proposta original: R$ 110.000         │
│  22/03 10:00  Contraproposta: R$ 125.000 (você)     │
│  22/03 11:30  Contraproposta: R$ 120.000 (comprador)│
│                                                      │
│  [Aceitar ✓]  [Recusar ✗]  [Contraproposta ↔]     │
└──────────────────────────────────────────────────────┘

MOBILE: tela dedicada com back navigation → T-035
```

**Componentes-chave:**
- "Você recebe: R$ X.XXX" como H1 visual (verde bold, tamanho maior)
- Comissão colapsável (`Collapsible`) — fechado por padrão
- `Countdown` para prazo
- Histórico de negociação como timeline
- 3 botões de ação (abrem T-037/T-038/T-039)

**Estados:**
- **Loading:** Skeleton do detalhe (3 retângulos proporcionais)
- **Empty:** N/A
- **Error:** "Não foi possível carregar os detalhes da proposta." + retry
- **Populated:** Hierarquia completa com todos os dados

---

### T-037 — Modal de Aceite de Proposta
**Rota:** `(overlay)` · **Role:** `CEDENTE` · **RFs:** RF-044

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="dialog" · focus trap     │                  │
│              │                               │                  │
│              │  Aceitar proposta             │  ← H2            │
│              │                               │                  │
│              │  RESUMO FINANCEIRO:           │                  │
│              │  Valor proposto: R$ 120.000   │                  │
│              │  Comissão (20%): R$ 24.000    │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ Você receberá:          │  │                  │
│              │  │ R$ 96.000               │  │  ← verde bold    │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  ⚠ Ao aceitar, o caso entra  │                  │
│              │    em formalização. Esta ação │                  │
│              │    inicia o Contrato de Cessão│                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │  [Confirmar aceite]  │     │  ← primary       │
│              │  └──────────────────────┘     │                  │
│              │  [Voltar]                     │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
ESC: retorna ao Passo 1 (cancela)
```

**Componentes-chave:**
- `Dialog` com focus trap
- Resumo financeiro (valor proposto, comissão, valor líquido em destaque verde)
- `Alert` informativo sobre início da formalização
- `Button` "Confirmar aceite" (primário)

**Estados:**
- **Loading:** Botão desabilitado + label "Processando..." durante processamento
- **Empty:** N/A
- **Error:** Toast vermelho "Não foi possível processar o aceite." + modal permanece
- **Populated:** Resumo + confirmação → toast "Proposta aceita!" + status atualiza

---

### T-038 — Modal de Recusa de Proposta
**Rota:** `(overlay)` · **Role:** `CEDENTE` · **RFs:** RF-046

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="dialog" · focus trap     │                  │
│              │                               │                  │
│              │  Recusar proposta             │  ← H2            │
│              │                               │                  │
│              │  Confirma a recusa desta      │                  │
│              │  proposta? O comprador não    │                  │
│              │  será notificado do motivo.   │                  │
│              │                               │                  │
│              │  Motivo interno (opcional):   │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ [Select ▼             ] │  │                  │
│              │  │  Valor abaixo da expect.│  │                  │
│              │  │  Outras condições       │  │                  │
│              │  │  Prefiro aguardar       │  │                  │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │ [Confirmar recusa]   │     │  ← --destructive │
│              │  └──────────────────────┘     │                  │
│              │  [Cancelar]                   │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
ESC: cancela e fecha
```

**Componentes-chave:**
- `Dialog` com focus trap
- `Select` opcional para motivo interno (não enviado ao comprador)
- `Button` "Confirmar recusa" (`--destructive`)

**Estados:**
- **Loading:** Botão desabilitado + label "Recusando..."
- **Empty:** N/A
- **Error:** Toast vermelho + modal permanece
- **Populated:** Confirmação → toast "Proposta recusada." + card some da lista

---

### T-039 — Modal de Contraproposta
**Rota:** `(overlay)` · **Role:** `CEDENTE` · **RFs:** RF-047

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="dialog" · focus trap     │                  │
│              │                               │                  │
│              │  Enviar contraproposta        │  ← H2            │
│              │                               │                  │
│              │  Piso do cenário B: R$ 68.000 │  ← helper text   │
│              │                               │                  │
│              │  Seu valor *                  │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ R$ [0,00            ]   │  │  ← currency mask │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  VALIDAÇÃO INLINE (val < piso):│                 │
│              │  ⚠ Este valor está abaixo do  │                  │
│              │    mínimo permitido (R$68.000).│                  │
│              │    Ajuste para continuar.     │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │[Enviar contraproposta]│     │  ← primary       │
│              │  │(habilitado se ≥ piso) │     │                  │
│              │  └──────────────────────┘     │                  │
│              │  [Cancelar]                   │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
ESC: cancela e fecha
```

**Componentes-chave:**
- `Dialog` com focus trap
- `Input` currency mask (R$) com validação em tempo real contra piso do cenário
- `Alert` variant="warning" âmbar quando valor < piso
- `Button` "Enviar contraproposta" habilitado apenas quando valor ≥ piso

**Estados:**
- **Loading:** Botão desabilitado + label "Enviando..."
- **Empty:** Campo zerado
- **Error:** Toast vermelho + modal permanece; alerta âmbar inline para valor abaixo do piso
- **Populated:** Valor válido → toast "Contraproposta enviada!" + status do card atualiza

---

## Módulo 8 — Documentos (Dossiê)

### T-040 — Tela de Documentos — Checklist
**Rota:** `/(authenticated)/documentos` · **Role:** `CEDENTE` · **RFs:** RF-056, RF-057, RF-059, RF-060, RF-061

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Documentos                                           │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  Dossiê — Rua das Flores, 100                        │
│              │  [Select: Caso ▼]  3 de 6 documentos enviados        │
│              │                                                       │
│              │  BANNER CONCLUSÃO (fade-out 10s, condicional):       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ ✅ Dossiê completo! O caso avançará para       │ │
│              │  │   análise assim que o Termo for assinado.       │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ DocumentCard                                    │ │
│              │  │ aria-label="RG ou CNH — status: Pendente"      │ │
│              │  │ RG ou CNH                [Pendente 🔴] [Enviar]│ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ DocumentCard                                    │ │
│              │  │ Comprovante Residência   [Em Análise 🟡]        │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ DocumentCard                                    │ │
│              │  │ Contrato Original        [Verificado ✅]         │ │
│              │  │ (somente leitura — sem botão)                   │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ DocumentCard                                    │ │
│              │  │ Comprovante Pgto  [Rejeitado 🔴]  [Reenviar ↑] │ │
│              │  │ ⚠ Arquivo ilegível (âmbar)                      │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│  DESKTOP hover: preview do arquivo à direita                        │
│  MOBILE: botão câmera 📷 após "Enviar" em itens pendentes          │
│  OFFLINE: botão "Enviar" desabilitado + tooltip "Sem conexão"      │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `DocumentCard` × 6 (PF) ou × 8 (PJ) com StatusBadge e botão de ação contextual
- Contador de progresso "X de 6 documentos enviados"
- Banner de conclusão (10s, fade-out, `aria-live="polite"`)
- Texto de motivo de rejeição em âmbar abaixo do card rejeitado
- Update em tempo real via Supabase Realtime
- Mobile: botão câmera Expo inline em itens pendentes

**Estados:**
- **Loading:** Skeleton de 6 linhas (48px cada)
- **Empty:** "Nenhum documento enviado ainda." + CTA "Enviar documentos"
- **Error:** Banner + retry; offline: botões desabilitados + tooltip
- **Populated:** Lista com status real por documento; preview desktop ao hover

---

### T-041 — Upload de Documento — In-progress
**Rota:** `(overlay/sheet em /(authenticated)/documentos)` · **Role:** `CEDENTE` · **RFs:** RF-057, RF-058

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Documentos                          [banner topo]   │
│  T-011       │  ⚠ Não feche esta janela durante o envio.           │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  OVERLAY (sobre área de conteúdo — não modal full):  │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │                                                 │ │
│              │  │  Enviando: RG_ou_CNH_joao.pdf (…completo)      │ │
│              │  │                                                 │ │
│              │  │  ████████████████░░░░░░░░░ 64%                 │ │
│              │  │  (barra de progresso animada --primary)        │ │
│              │  │  128 KB/s                                      │ │
│              │  │                                                 │ │
│              │  │  [Cancelar ✗] ← --destructive text-only       │ │
│              │  │                                                 │ │
│              │  │  CONCLUÍDO (100%):                             │ │
│              │  │  [✓ checkmark animado] → overlay fecha 1s     │ │
│              │  │                                                 │ │
│              │  │  ERRO MIME:                                    │ │
│              │  │  "Formato não aceito. Use PDF, JPG ou PNG."    │ │
│              │  │                                                 │ │
│              │  │  ERRO TAMANHO:                                 │ │
│              │  │  "Arquivo muito grande. Máximo 10 MB."         │ │
│              │  │                                                 │ │
│              │  │  CONEXÃO INTERROMPIDA:                         │ │
│              │  │  "Conexão interrompida."                       │ │
│              │  │  [Retomar] ou [Enviar novamente]               │ │
│              │  └─────────────────────────────────────────────────┘ │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Overlay que cobre a área de conteúdo (não modal fullscreen — navegação possível com aviso)
- Barra de progresso com percentual numérico + velocidade estimada (KB/s)
- `Button` "Cancelar" (--destructive text-only)
- Banner não-intrusivo no topo: "Não feche esta janela durante o envio."
- Checkmark animado ao concluir → overlay fecha em 1s

**Estados:**
- **Loading:** Estado primário — barra de progresso ativa
- **Empty:** N/A (overlay só aparece durante upload ativo)
- **Error:** Mensagem inline vermelha no DocumentCard após fechar overlay (MIME, tamanho, conexão)
- **Populated (concluído):** Barra 100% + checkmark + fechamento automático + DocumentCard atualiza para "Em Análise"

---

### T-042 — Upload por Câmera — Mobile
**Rota:** `/(authenticated)/documentos/camera` (mobile) · **Role:** `CEDENTE` · **RFs:** RF-120

```javascript
┌─────────────────────────────────────────────────────┐
│  CÂMERA NATIVA EXPO — TELA CHEIA                    │
│                                                     │
│  [✗ Cancelar]  (canto superior esquerdo, 44px)     │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │                                             │   │
│  │  (viewfinder da câmera)                     │   │
│  │                                             │   │
│  │  ┌─────────────────────────────────────┐    │   │
│  │  │  frame de recorte sugerido (guia)   │    │   │
│  │  │  (overlay pontilhado para           │    │   │
│  │  │   posicionar o documento)           │    │   │
│  │  └─────────────────────────────────────┘    │   │
│  │                                             │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  [Galeria 📷]    [● Capturar]    (espaço)          │
│  (44px)          (64px, central)                   │
│                                                     │
│  PREVIEW PÓS-CAPTURA:                              │
│  [imagem capturada]                                │
│  [Usar esta foto ✓]   [Tirar novamente ↺]         │
│                                                     │
│  PERMISSÃO NEGADA:                                 │
│  "Permita o acesso à câmera nas configurações."    │
│  [Abrir configurações]                             │
└─────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `expo-camera` em tela cheia (aceita retrato e paisagem)
- Frame de recorte overlay (guia visual para posicionar documento)
- `Button` "Capturar" 64px centralizado
- `Button` "Usar galeria" à esquerda (acesso à galeria do dispositivo)
- `Button` "Cancelar" (X) no canto superior esquerdo (44px)
- Preview pós-captura com "Usar esta foto" + "Tirar novamente"

**Estados:**
- **Loading:** N/A (câmera abre imediatamente)
- **Empty:** Viewfinder ativo com frame de guia
- **Error:** Tela de permissão negada + link para configurações do sistema
- **Populated (pós-captura):** Preview com 2 ações; ao confirmar: inicia T-041


---

## Módulo 9 — Assinaturas Eletrônicas

### T-043 — Lista de Documentos para Assinatura
**Rota:** `/(authenticated)/assinaturas` · **Role:** `CEDENTE` · **RFs:** RF-064, RF-068

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Assinaturas                                          │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  AVISO PRAZO CRÍTICO (banner âmbar, condicional):    │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ ⚠ Atenção: 1 documento vence em menos de 48h.  │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  Rua das Flores, 100 — Apto 23                       │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ Termo de Cadastro                               │ │
│              │  │ Criado: 15/03 · Prazo: 20/03                   │ │
│              │  │ [⏱ Expira em 1d 6h] (âmbar, < 48h)            │ │
│              │  │ [Aguardando assinatura 🟡]     [Assinar →]     │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ Instrumento de Cessão                           │ │
│              │  │ Criado: 16/03 · Sem prazo                      │ │
│              │  │ [Assinado ✅ · 17/03/2026]                      │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  EMPTY STATE:                                         │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │  [✓ ícone check]                                │ │
│              │  │  Nenhum documento pendente de assinatura.       │ │
│              │  │  Você está em dia!                              │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│  MOBILE: lista somente leitura; "Assinar disponível no computador" │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Lista de documentos agrupada por caso com tipo, datas e StatusBadge
- `Countdown` para documentos com prazo (âmbar quando < 48h)
- `Button` "Assinar" (web — abre T-044); mobile: mensagem informativa
- Banner âmbar de prazo crítico (condicional — exibido quando há < 48h)

**Estados:**
- **Loading:** Skeleton de 3 linhas (48px cada)
- **Empty:** "Nenhum documento pendente de assinatura. Você está em dia!" + ícone de check
- **Error:** Banner + retry
- **Populated:** Lista com status real, Countdown para prazos próximos

---

### T-044 — Assinatura Inline — ZapSign
**Rota:** `(drawer lateral / WebView mobile em /(authenticated)/assinaturas)` · **Role:** `CEDENTE` · **RFs:** RF-065, RF-066, RF-067

```javascript
DESKTOP (drawer lateral direito 640px):
┌──────────────┬────────────────────────────────────────────────────┐
│  SIDEBAR     │  Assinaturas          [drawer desliza da direita]  │
│  T-011       │──────────────────┬─────────────────────────────── │
│              │  [Lista T-043]   │  DRAWER ZAPSIGN (640px)        │
│              │                  │  role="dialog" · focus trap    │
│              │                  │  [✗ Fechar] (ESC com confirm.) │
│              │                  │                                │
│              │                  │  LOADING (< 3s target):        │
│              │                  │  ┌──────────────────────────┐  │
│              │                  │  │  Carregando documento... │  │
│              │                  │  │  [spinner leve 16px]     │  │
│              │                  │  └──────────────────────────┘  │
│              │                  │                                │
│              │                  │  IFRAME ZAPSIGN CARREGADO:    │
│              │                  │  ┌──────────────────────────┐  │
│              │                  │  │                          │  │
│              │                  │  │   [ZapSign embed 100%]   │  │
│              │                  │  │   Documento PDF para     │  │
│              │                  │  │   assinatura             │  │
│              │                  │  │                          │  │
│              │                  │  └──────────────────────────┘  │
│              │                  │                                │
│              │                  │  ERRO (timeout 10s):           │
│              │                  │  "ZapSign temporariamente      │
│              │                  │   indisponível."               │
│              │                  │  [Tentar novamente] [Cancelar] │
│              │                  │                                │
│              │                  │  OFFLINE:                      │
│              │                  │  "Você precisa de conexão      │
│              │                  │   para assinar."               │
└──────────────┴──────────────────┴────────────────────────────────┘

MOBILE (WebView fullscreen):
┌─────────────────────────────────────────────────────┐
│  [✗ Fechar] (botão flutuante, canto sup. dir., 44px)│
│                                                     │
│  [ZapSign WebView — tela cheia]                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Sheet` (shadcn) lateral direito 640px desktop; fullscreen em tablet
- `iframe` ZapSign embedado (sem redirecionamento externo)
- Botão "Fechar" — ESC com confirmação "Fechar sem assinar?" se assinatura em andamento
- Detecção de evento `postMessage` do ZapSign para confirmar assinatura concluída
- Supabase Realtime: DocumentCard na lista atualiza para "Assinado" após conclusão
- Mobile: WebView ZapSign (expo-web-browser ou WebView nativo)

**Estados:**
- **Loading:** Spinner leve 16px + "Carregando documento para assinatura..." (< 3s target)
- **Empty:** N/A (drawer só abre com documento selecionado)
- **Error:** Timeout 10s → mensagem + botões "Tentar novamente" / "Cancelar"; offline → mensagem
- **Populated:** iframe ativo com ZapSign; ao concluir: slide-up animation + lista atualiza via Realtime

**Notas:** Cedente PF: envelope enviado ao e-mail do Cedente. Cedente PJ: envelope ao Representante Legal. Assinatura parcial preservada pelo ZapSign ao fechar sem concluir — badge "Parcialmente preenchido" no DocumentCard.

---

## Módulo 10 — Financeiro (Conta Escrow)

### T-045 — Painel Financeiro — Pré-formalização
**Rota:** `/(authenticated)/financeiro` (estado pré-formalização) · **Role:** `CEDENTE` · **RFs:** RF-071

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Financeiro                                           │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ EscrowPanel — Estado: Pré-formalização          │ │
│              │  │                                                 │ │
│              │  │  ℹ Conta Escrow: Não iniciada                  │ │
│              │  │                                                 │ │
│              │  │  "O valor será depositado na Conta Escrow       │ │
│              │  │   após a assinatura do Contrato de Cessão."     │ │
│              │  │                                                 │ │
│              │  │  ┌───────────────────────────────────────────┐  │ │
│              │  │  │ VALORES ESTIMADOS (--muted background)    │  │ │
│              │  │  │ Cenário B · Estimado: R$ 72.000           │  │ │
│              │  │  │ Comissão: R$ 14.400 · Líquido: R$ 57.600 │  │ │
│              │  │  └───────────────────────────────────────────┘  │ │
│              │  │                                                 │ │
│              │  │  [Ver simulação de cenários →] (link T-026)    │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│  OFFLINE: badge "Dados de 22/03 12:00. Sem conexão." no painel     │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `EscrowPanel` em estado informativo (somente leitura)
- Card `--muted` background com valores estimados do cenário escolhido
- Link "Ver simulação" → T-026/T-031
- Badge "Dados de [data/hora]." para modo offline

**Estados:**
- **Loading:** Skeleton do painel EscrowPanel (retângulo 100% width, 200px)
- **Empty:** N/A (sempre exibe estado atual)
- **Error:** N/A (falha exibida inline com retry)
- **Populated:** Painel informativo com valores estimados

---

### T-046 — Painel Financeiro — Escrow Aberto
**Rota:** `/(authenticated)/financeiro` (estado escrow ativo) · **Role:** `CEDENTE` · **RFs:** RF-072, RF-073, RF-074

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Financeiro                                           │
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ EscrowPanel — Estado: Ativo                     │ │
│              │  │                                                 │ │
│              │  │  [Conta Escrow: Ativa ●] ← badge verde         │ │
│              │  │                                                 │ │
│              │  │  Você receberá:                                 │ │
│              │  │  R$ 96.000              ← H1 visual (verde bold)│ │
│              │  │                                                 │ │
│              │  │  Valor em Escrow: R$ 120.000                    │ │
│              │  │                                                 │ │
│              │  │  Comissão da plataforma ▼ (colapsável)          │ │
│              │  │  ┌─────────────────────────────────────────┐    │ │
│              │  │  │ 20% · R$ 24.000                        │    │ │
│              │  │  └─────────────────────────────────────────┘    │ │
│              │  │                                                 │ │
│              │  │  Liberação em: [⏱ 12d 8h 23min] ← Countdown   │ │
│              │  │                                                 │ │
│              │  │  [Solicitar Desistência] ← text-only destrutivo│ │
│              │  │  (visível apenas nos 15 dias pós-Fechamento)   │ │
│              │  │                                                 │ │
│              │  │  PÓS-15 DIAS:                                  │ │
│              │  │  "Prazo de desistência encerrado."              │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│  OFFLINE: badge "Dados de [data/hora]." no painel                  │
│  Tela web only — mobile não exibe.                                  │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `EscrowPanel` em estado ativo
- "Você receberá: R$ X.XXX" como H1 visual (verde bold, maior)
- Badge verde "Conta Escrow: Ativa"
- `Collapsible` para breakdown de comissão
- `Countdown` para liberação (dias + horas + minutos)
- `Button` "Solicitar Desistência" (text-only, --destructive) — condicional aos 15 dias

**Estados:**
- **Loading:** Skeleton do painel
- **Empty:** N/A
- **Error:** Badge "Dados de [data/hora]." + retry inline
- **Populated:** EscrowPanel completo com countdown e ação de desistência (se aplicável)

---

### T-047 — Modal de Solicitação de Desistência
**Rota:** `(overlay em /(authenticated)/financeiro)` · **Role:** `CEDENTE` · **RFs:** RF-053, RF-054

```javascript
┌─────────────────────────────────────────────────────────────────┐
│  OVERLAY rgba(0,0,0,0.5)                                         │
│                                                                  │
│              ┌───────────────────────────────┐                  │
│              │ role="dialog" · focus trap     │                  │
│              │                               │                  │
│              │  Solicitar desistência        │  ← H2            │
│              │                               │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ ⚠ ATENÇÃO              │  │  ← alert         │
│              │  │ Esta solicitação cancela│  │    vermelho      │
│              │  │ o repasse. O valor em   │  │    proeminente   │
│              │  │ Escrow será estornado.  │  │                  │
│              │  │ Prazo: até 15 dias úteis│  │                  │
│              │  └─────────────────────────┘  │                  │
│              │                               │                  │
│              │  PASSO 1 — JUSTIFICATIVA:     │                  │
│              │  Justificativa *              │                  │
│              │  ┌─────────────────────────┐  │                  │
│              │  │ [textarea 20-500 chars] │  │                  │
│              │  │ Descreva o motivo...    │  │                  │
│              │  │                    0/500│  │  ← contador     │
│              │  └─────────────────────────┘  │                  │
│              │  [Continuar →]  [Cancelar]    │                  │
│              │                               │                  │
│              │  PASSO 2 — CONFIRMAÇÃO:       │                  │
│              │  Resumo da solicitação        │                  │
│              │  ☐ Confirmo que estou        │                  │
│              │    solicitando a desistência │                  │
│              │    e compreendo que esta     │                  │
│              │    ação é irreversível. *    │                  │
│              │                               │                  │
│              │  ┌──────────────────────┐     │                  │
│              │  │[Confirmar desistência]│     │  ← destrutivo    │
│              │  │(habilitado após ☑)   │     │                  │
│              │  └──────────────────────┘     │                  │
│              │  [← Voltar]                   │                  │
│              └───────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Dialog` com focus trap
- `Alert` variant="destructive" proeminente no topo
- `Textarea` com contador (min 20 / max 500 chars)
- `Checkbox` obrigatório no Passo 2 para habilitar botão destrutivo

**Estados:**
- **Loading:** Botão "Confirmar desistência" desabilitado + label "Enviando solicitação..."
- **Empty:** Textarea vazia, contador zerado
- **Error:** Toast vermelho + modal permanece
- **Populated:** Sucesso → modal fecha + toast "Solicitação registrada. Entraremos em contato em até 48 horas." + botão "Solicitar Desistência" some do painel

---

## Módulo 11 — Assistente IA (Guardião do Retorno)

### T-048 — Chat — Guardião do Retorno
**Rota:** `/(authenticated)/assistente` · **Role:** `CEDENTE` · **RFs:** RF-086, RF-087, RF-088, RF-092

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  HEADER DO CHAT:                                      │
│  T-011       │  [Avatar Guardião] Guardião do Retorno  [IA badge]   │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│  DESKTOP: split view                                                 │
│  ┌───────────────┬──────────────────────────────────────────────┐   │
│  │ HISTÓRICO     │ ÁREA DE CHAT                                 │   │
│  │ (280px)       │ role="log" · aria-live="polite"              │   │
│  │               │                                              │   │
│  │  Hoje         │  PRIMEIRA ABERTURA (sem histórico):          │   │
│  │  · Conversa 1 │  [Mensagem boas-vindas do Guardião]          │   │
│  │               │  ┌──────────┐ ┌──────────┐ ┌─────────────┐  │   │
│  │  Ontem        │  │Status do │ │Simular   │ │Ajuda com    │  │   │
│  │  · Conversa 2 │  │meu caso? │ │cenário   │ │documentos   │  │   │
│  │               │  └──────────┘ └──────────┘ └─────────────┘  │   │
│  │               │  (botões somem após 1ª msg)                  │   │
│  │               │                                              │   │
│  │               │  MENSAGEM GUARDIÃO (balão cinza esquerdo):   │   │
│  │               │  ┌──────────────────────────────────────┐    │   │
│  │               │  │ [Avatar] Olá! Como posso ajudar?▌   │    │   │
│  │               │  │          (▌ = cursor SSE streaming) │    │   │
│  │               │  └──────────────────────────────────────┘    │   │
│  │               │                                              │   │
│  │               │  MENSAGEM USUÁRIO (balão --primary direito): │   │
│  │               │     ┌────────────────────────────────────┐   │   │
│  │               │     │ Qual o status do meu caso?         │   │   │
│  │               │     └────────────────────────────────────┘   │   │
│  │               │                                              │   │
│  │               │  ──────────────────────────────────────────  │   │
│  │               │  ┌──────────────────────────────────────┐    │   │
│  │               │  │ [Pergunte ao Guardião...          ]  │    │   │
│  │               │  │ (textarea 1-4 linhas, max 120px)  [▶]│    │   │
│  │               │  └──────────────────────────────────────┘    │   │
│  └───────────────┴──────────────────────────────────────────────┘   │
│                                                                      │
│  MOBILE: chat tela cheia; sidebar histórico via botão "Conversas"   │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `GuardiaoChat` (componente custom) com histórico lateral (280px desktop)
- Área de mensagens: `role="log"`, `aria-live="polite"` (anuncia ao completar stream — não token a token)
- Balões: Guardião (cinza esquerdo) / Usuário (--primary direito)
- Cursor caret piscante durante SSE streaming
- 3 botões de sugestão na primeira abertura (somem após 1ª mensagem)
- `Textarea` expansível (1–4 linhas, max 120px) + botão enviar 44×44px

**Estados:**
- **Loading (SSE):** Avatar do Guardião animado + cursor caret na área de resposta
- **Empty:** Mensagem de boas-vindas + 3 botões de sugestão
- **Error:** "Não consegui processar sua mensagem. Tente novamente." + botão retry; offline: "Sem conexão. O Guardião estará disponível quando você reconectar."
- **Populated:** Histórico de mensagens + resposta do Guardião em streaming

---

### T-049 — Chat — Cadastro Assistido
**Rota:** `/(authenticated)/assistente/cadastro-assistido` · **Role:** `CEDENTE` · **RFs:** RF-090

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  [Avatar Guardião] Guardião · Modo: Cadastro Assistido│
│  T-011       │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  STEPPER (5 etapas, acima da área de chat):          │
│              │  [●1 Endereço] ─ [○2 Valores] ─ [○3 Simul.]         │
│              │               ─ [○4 Cenário] ─ [○5 Revisão]         │
│              │                                                       │
│              │  BANNER PERSISTENTE:                                  │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ 🧙 Modo: Cadastro Assistido                     │ │
│              │  │ [Cancelar cadastro assistido ✗]                 │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  ÁREA DE CHAT (idêntica ao T-048):                   │
│              │  role="log" · aria-live="polite"                     │
│              │                                                       │
│              │  [Avatar] Vamos começar! Qual o                      │
│              │  endereço do imóvel que você quer                    │
│              │  cadastrar?▌                                          │
│              │                                                       │
│              │  [Etapa 1 concluída — avança stepper automaticamente]│
│              │                                                       │
│              │  [Avatar] Coletei todas as informações!              │
│              │  Clique abaixo para revisar e confirmar              │
│              │  o cadastro.                                          │
│              │  ┌──────────────────────────────┐                   │
│              │  │  [Revisar e confirmar →]      │                   │
│              │  └──────────────────────────────┘ ← inline no chat  │
│              │                                                       │
│              │  ──────────────────────────────────────────────────  │
│              │  [Digite sua resposta...                          [▶]│
└──────────────┴──────────────────────────────────────────────────────┘
CANCELAR: confirma descarte dos dados coletados ("Dados descartados.")
```

**Componentes-chave:**
- `Stepper` horizontal simples (5 etapas) acima da área de chat — etapa atual destacada
- Banner persistente "Modo: Cadastro Assistido" com botão "Cancelar" (encerra fluxo + descarta dados)
- `GuardiaoChat` em modo LangGraph.js multi-step
- Botão inline "Revisar e confirmar" ao concluir todos os steps (abre T-028 com dados pré-preenchidos)
- Hierarquia visual: stepper > área de chat > input

**Estados:**
- **Loading:** Idêntico ao T-048 (avatar animado + cursor SSE)
- **Empty:** Primeira mensagem do Guardião solicitando endereço
- **Error:** Guardião explica o que faltou e repete a pergunta; sem retry automático
- **Populated:** Conversa em andamento com stepper avançando por etapa; ao concluir: botão "Revisar e confirmar"

**Notas:** [DECISÃO APLICADA: DEC-005] Abandono do fluxo descarta dados coletados (sem rascunho para evitar estado inconsistente entre os dois modos de cadastro).

---

## Módulo 12 — Perfil e Configurações

### T-050 — Meu Perfil — Dados Pessoais
**Rota:** `/(authenticated)/perfil` · **Role:** `CEDENTE` · **RFs:** RF-011, RF-012

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Meu Perfil                                           │
│  T-011       │──────────────────────────────────────────────────────│
│              │  [Dados Pessoais ●] [Notificações] [LGPD]            │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  PF — CAMPOS EXIBIDOS:                               │
│              │  Nome completo *                                     │
│              │  [João da Silva                  ] (editável)        │
│              │                                                       │
│              │  E-mail *                                            │
│              │  [joao@email.com] [Ativo ●] (editável)               │
│              │  ALTERAÇÃO: [novo@email.com] [Confirmar] [Atualizar]  │
│              │  → badge muda para [Pendente de confirmação]         │
│              │                                                       │
│              │  Telefone *                                          │
│              │  [(85) 99999-9999          ] (editável, masked)       │
│              │                                                       │
│              │  CPF                                                  │
│              │  [000.000.000-00] (somente leitura, aria-readonly)   │
│              │  "CPF não pode ser alterado." (helper text)          │
│              │                                                       │
│              │  PJ — CAMPOS ADICIONAIS:                             │
│              │  Razão Social * [editável]                           │
│              │  CNPJ [somente leitura]                              │
│              │  Representante Legal: nome + CPF [somente leitura]   │
│              │  Cargo [editável]                                    │
│              │                                                       │
│              │  ┌──────────────────────────────────┐               │
│              │  │      [Salvar alterações]          │               │
│              │  │  (habilitado quando há mudanças)  │               │
│              │  └──────────────────────────────────┘               │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Tabs` com 3 abas: Dados Pessoais, Notificações, LGPD
- Campos editáveis: `Input` com validação on-blur
- Campos somente leitura: `aria-readonly="true"` + helper text explicativo
- Badge de status do e-mail (Ativo / Pendente de confirmação)
- `Button` "Salvar alterações" (primário, habilitado apenas quando há mudanças)

**Estados:**
- **Loading:** Skeleton de 5 campos (48px cada)
- **Empty:** N/A (sempre exibe dados do usuário logado)
- **Error:** Toast vermelho "Não foi possível salvar." + campos revertidos ao estado anterior
- **Populated:** Dados do usuário exibidos; sucesso → toast "Dados atualizados com sucesso!"

---

### T-051 — Meu Perfil — Notificações
**Rota:** `/(authenticated)/perfil?tab=notificacoes` · **Role:** `CEDENTE` · **RFs:** RF-013, RF-014, RF-079, RF-080

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Meu Perfil                                           │
│  T-011       │  [Dados Pessoais] [Notificações ●] [LGPD]            │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  Preferências de notificação                         │
│              │                                                       │
│              │  DESATIVÁVEIS:                                        │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ Propostas                 [toggle ON ●]         │ │
│              │  │ Documentos                [toggle ON ●]         │ │
│              │  │ Assinaturas               [toggle OFF ○]        │ │
│              │  │ Financeiro                [toggle ON ●]         │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  NÃO DESATIVÁVEIS (obrigatórias):                    │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ Expiração de sessão       [toggle cinza 🔒]     │ │
│              │  │ aria-describedby="notif-obrig-msg"              │ │
│              │  │ Prazo proposta < 24h      [toggle cinza 🔒]     │ │
│              │  │ Desistência/Cancelamento  [toggle cinza 🔒]     │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │  "Esta notificação é obrigatória e não pode ser      │ │
│              │   desativada." (id="notif-obrig-msg")                 │
│              │                                                       │
│              │  ┌──────────────────────────┐                        │
│              │  │  [Salvar preferências]   │  ← primary             │
│              │  └──────────────────────────┘                        │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- `Switch` (`role="switch"`) para toggles desativáveis
- Toggles acinzentados (não interativos) para notificações obrigatórias com `aria-describedby`
- `Button` "Salvar preferências" (primário)

**Estados:**
- **Loading:** Skeleton de 7 linhas (40px cada)
- **Empty:** N/A (lista sempre exibida)
- **Error:** Toast vermelho se salvar falhar
- **Populated:** Toggles com estado atual; salvo → toast "Preferências salvas."

---

### T-052 — Meu Perfil — LGPD / Exclusão de Dados
**Rota:** `/(authenticated)/perfil?tab=lgpd` · **Role:** `CEDENTE` · **RFs:** RF-015

```javascript
┌──────────────┬──────────────────────────────────────────────────────┐
│  SIDEBAR     │  Meu Perfil                                           │
│  T-011       │  [Dados Pessoais] [Notificações] [LGPD ●]            │
│              │──────────────────────────────────────────────────────│
│              │                                                       │
│              │  BANNER EXCLUSÃO EM ANDAMENTO (persistente, âmbar):  │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ ⚠ Solicitação de exclusão em andamento.        │ │
│              │  │   Protocolo: 12345-6789                         │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  Seus dados pessoais estão protegidos conforme a     │
│              │  Lei Geral de Proteção de Dados (LGPD).              │
│              │  [Política de Privacidade ↗]                         │
│              │                                                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │  [Solicitar exclusão de dados] ← destrutivo     │ │
│              │  │      variant="outline" (--destructive border)   │ │
│              │  └─────────────────────────────────────────────────┘ │
│              │                                                       │
│              │  MODAL DE EXCLUSÃO (3 passos):                       │
│              │  ┌─────────────────────────────────────────────────┐ │
│              │  │ role="dialog" · focus trap                      │ │
│              │  │                                                 │ │
│              │  │ PASSO 1 — AVISO:                                │ │
│              │  │ "Todos os seus dados serão excluídos.           │ │
│              │  │  Casos ativos podem ser afetados.               │ │
│              │  │  Prazo: 30 dias úteis."                         │ │
│              │  │ [Continuar]  [Cancelar]                         │ │
│              │  │                                                 │ │
│              │  │ PASSO 2 — CONFIRMAÇÃO SENHA:                   │ │
│              │  │ [••••••••        ] *                            │ │
│              │  │ [Continuar]                                     │ │
│              │  │                                                 │ │
│              │  │ PASSO 3 — CHECKBOX FINAL:                      │ │
│              │  │ ☐ Compreendo e aceito as consequências. *      │ │
│              │  │ [Solicitar exclusão] ← destrutivo (após ☑)    │ │
│              │  └─────────────────────────────────────────────────┘ │
└──────────────┴──────────────────────────────────────────────────────┘
```

**Componentes-chave:**
- Banner âmbar persistente quando solicitação em andamento (com protocolo)
- `Button` variant="outline" com --destructive border
- `Dialog` com 3 passos (foco inicial no botão "Continuar")
- `Input` type="password" no Passo 2
- `Checkbox` obrigatório no Passo 3 para habilitar botão final destrutivo
- Botão "Solicitar exclusão" (`aria-label` completo — "Confirmar exclusão permanente dos dados")

**Estados:**
- **Loading:** Botão "Solicitar exclusão" desabilitado + label "Registrando..."
- **Empty:** N/A (tela sempre exibe informações)
- **Error:** Toast vermelho + modal permanece
- **Populated:** Sucesso → modal fecha + card com protocolo gerado ("Protocolo nº XXXXX-XXXX. Confirmação por e-mail em até 30 dias úteis.") + banner âmbar persistente

---

## 5. Convenções Globais de Wireframe

### 5.1 Estados Obrigatórios (toda tela)

| Estado | Implementação | Componente shadcn | Duração |
|---|---|---|---|
| **Loading** | Skeleton retangulares — **NUNCA spinner** | `Skeleton` | Enquanto fetch ativo |
| **Empty** | Ícone + título + descrição + CTA (quando aplicável) | `Card` centralizado | Quando dados ausentes |
| **Error** | `Alert` variant="destructive" + botão "Tentar novamente" | `Alert` | Quando request falha |
| **Populated** | Conteúdo real com dados do backend | — | Após carregamento |

> 💡 **Regra do skeleton:** Todo estado de loading usa `Skeleton` retangular. Spinners são permitidos **apenas** para ações inline (submit de formulário, upload inline, carregamento do iframe ZapSign). **Nunca** como substituto de tela inteira.

### 5.2 Padrão de Layout — AppLayout (Sidebar)

```javascript
DESKTOP (> 1024px):
┌──────────┬──────────────────────────────────────────────┐
│ SIDEBAR  │  HEADER FIXO (h-14, --background, border-b)  │
│ 240px    │──────────────────────────────────────────────│
│ --sidebar│  CONTEÚDO (grid 12 col, gap-6, max-w-[1280px]│
│ #FAFAFA  │  padding: px-6 py-6)                         │
│          │                                              │
│ [Logo]   │                                              │
│ NavMenu  │                                              │
│ (9 itens)│                                              │
│ UserArea │                                              │
└──────────┴──────────────────────────────────────────────┘

TABLET (768-1024px):
Sidebar colapsada → drawer overlay ao clicar ☰ no header

MOBILE (< 768px):
Sem sidebar → Bottom Tab Bar (T-012)
```

### 5.3 Padrão de Layout — AuthLayout (Público)

```javascript
┌─────────────────────────────────────────────────────────┐
│ FUNDO: --background (#FFFFFF)                           │
│                                                         │
│ Conteúdo centralizado vertical + horizontal             │
│ max-width: 480px · padding: px-4 py-8                  │
│                                                         │
│ Logo Repasse Seguro no topo do card                     │
└─────────────────────────────────────────────────────────┘
```

### 5.4 Padrão de Modais

| Atributo | Valor obrigatório |
|---|---|
| `role` | `"dialog"` (ação) ou `"alertdialog"` (informativo) |
| `aria-modal` | `"true"` |
| Focus trap | Obrigatório em todos os modais |
| Fechamento ESC | Obrigatório; comportamento específico documentado por tela |
| Foco inicial | Botão primário (ou primeiro campo de input) |
| Overlay | `rgba(0,0,0,0.5)` |

### 5.5 Padrão de Toasts

| Tipo | Componente | Duração |
|---|---|---|
| Sucesso | `Toast` variant="default" (verde) | 3s |
| Erro | `Toast` variant="destructive" | 5s (dismissível) |
| Info / Âmbar | `Toast` variant="default" (âmbar) | 4s |

### 5.6 Componentes Reutilizáveis (12 identificados)

| Componente | Telas que usam | Descrição |
|---|---|---|
| `StatusBadge` | T-015, T-016, T-017, T-035, T-036, T-043 | Badge colorido com status do caso/documento/proposta |
| `DocumentCard` | T-017, T-040 | Card de documento com status e ação contextual |
| `ProposalCard` | T-018, T-035 | Card de proposta com valor, countdown e ações |
| `Countdown` | T-009, T-010, T-026, T-033, T-035, T-036, T-043, T-046 | Contagem regressiva com atualização a cada segundo |
| `EscrowPanel` | T-020, T-045, T-046 | Painel de Conta Escrow (pré-formalização ou ativo) |
| `GuardiaoChat` | T-048, T-049 | Interface de chat com SSE streaming |
| `NotificationBell` | T-011 | Sino com badge numérico no header |
| `CaseCard` | T-013, T-015 | Card resumo de caso com status, endereço e próximo passo |
| `Stepper` | T-024–T-028, T-049 | Indicador de progresso em etapas |
| `FinancialSummary` | T-013, T-016 | Resumo financeiro (cenário, valor, data estimada) |
| `SignatureInline` | T-044 | Wrapper do iframe/WebView ZapSign |
| `UploadZone` | T-040, T-041 | Área de upload com drag-and-drop e barra de progresso |

### 5.7 Padrões Transversais de Acessibilidade

- **WCAG 2.1 AA** obrigatório em todas as telas
- Touch targets mínimos: **44×44px** (botões, links, toggles)
- `aria-label` em todos os ícones sem texto visível
- `aria-live="polite"` em áreas de atualização dinâmica (chat, realtime)
- `aria-live="off"` em contadores que atualizam frequentemente (sessão T-010)
- `aria-required="true"` em todos os campos obrigatórios
- Contraste mínimo 4.5:1 (texto normal) e 3:1 (texto grande / UI)
- Sem informação transmitida **apenas** por cor

---

## 6. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-22 | Claude Code Desktop — Pipeline v9.5 | Criação. 52 wireframes ASCII para todas as telas T-001 a T-052. 12 módulos. Cobertura 100%. |

---

## 7. Backlog de Pendências

### 7.1 Decisões Autônomas Aplicadas

| ID | Módulo / Tela | Decisão | Justificativa | Impacto | Status |
|---|---|---|---|---|---|
| DEC-001 | T-010 | ESC = "Encerrar agora" (não fecha neutro) | Inação em expiração de sessão deve ser o comportamento mais seguro — encerrar é mais correto que renovar por acidente | P1 | ✅ Aplicado |
| DEC-002 | T-011 | Sidebar 240px expandida / 64px colapsada | Padrão shadcn sidebar, alinha com B04 — ícones legíveis em 64px, largura 240px acomoda labels sem truncar | P1 | ✅ Aplicado |
| DEC-003 | T-016 | Layout 2 colunas desktop (Zona 1+2 esq, Zona 3 dir) | Maximiza densidade de informação sem scroll excessivo; Zona 3 (timeline) beneficia de coluna lateral | P2 | ✅ Aplicado |
| DEC-004 | T-031 | Seletor de cenário-alvo para múltiplos escalonamentos | RF-039-A cobre múltiplos escalonamentos simultâneos; seletor de tabs/select é o padrão UX mais claro | P2 | ✅ Aplicado |
| DEC-005 | T-049 | Dados do cadastro assistido descartados ao cancelar | Evita estado inconsistente entre modo assistido e wizard direto; fluxos independentes não devem compartilhar rascunho | P2 | ✅ Aplicado |
| DA-002 | Geral | Wizard de cadastro não disponível em mobile | Formulários de 5 etapas com validações complexas demandam teclado e tela maiores; mobile tem UX degradada | P1 | ✅ Documentado |
| DA-004 | T-035/T-036 | Split view de propostas em desktop | Padrão Gmail/Outlook: lista + detalhe lado a lado aumenta eficiência para usuários com múltiplas propostas ativas | P2 | ✅ Documentado |

### 7.2 Pendências a Investigar

| Item | Módulo / Tela | Impacto | O que falta | Dono | Status |
|---|---|---|---|---|---|
| BP-01 | T-009/T-043 (Countdown) | P2 | Verificar existência de T-106 "inadimplência" — referenciada no Mapa de Telas v1.0 como uso do componente Countdown, corrigida para T-009/T-043 em B04 (DIV-001). Confirmar se tela de inadimplência existe em outro módulo ou é funcionalidade futura. | Produto | 🔍 Verificar |

