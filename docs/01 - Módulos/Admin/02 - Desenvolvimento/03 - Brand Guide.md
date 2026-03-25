# Theme Guide — Repasse Seguro

## Repasse Seguro

| **Produto** | Repasse Seguro |
| --- | --- |
| **Stack** | shadcn/ui + Tailwind CSS |
| **Versão** | 1.0 |
| **Data** | 20/03/2026 |
| **Responsável** | @Lucas Cartaxo |

---

<aside>
📌

**TL;DR**

- **Componentes:** shadcn/ui como biblioteca única.
- **Theming:** CSS custom properties do shadcn com suporte a light/dark mode.
- **Tipografia:** Inter Variable (heading e body).
- **Radius base:** 0.875rem (14px) — raio generoso, visual amigável e arredondado.
- **Paleta:** Neutro puro (true gray) com acentos azul (#0069A8). Charts em escala azul celeste.
- **Destructive:** Adaptado por tema (light: #E7000B / dark: #FF6467).
</aside>

---

# 1. Theme Configuration (shadcn)

O Repasse Seguro utiliza o sistema de theming nativo do shadcn/ui. Todos os componentes consomem as variáveis abaixo automaticamente. **Não há tokens customizados fora dessa estrutura.**

---

## 1.1 Tipografia

| **Token** | **Valor** | **Uso** |
| --- | --- | --- |
| `--font-heading` | `var(--font-sans)` → `'Inter Variable', sans-serif` | Headings — herda de `--font-sans` |
| `--font-sans` | `'Inter Variable', sans-serif` | Toda a interface — body, inputs, selects, tabelas, labels, buttons, badges |

## 1.2 Border Radius

Base de 14px — raio muito generoso. Visual suave e arredondado que transmite confiança.

| **Token** | **Cálculo** | **Valor** | **Uso** |
| --- | --- | --- | --- |
| `--radius` | — | 0.875rem (14px) | Base de referência |
| `--radius-sm` | `radius × 0.6` | 0.525rem (8.4px) | Badges, chips, barras de gráfico |
| `--radius-md` | `radius × 0.8` | 0.7rem (11.2px) | Botões, inputs, selects |
| `--radius-lg` | `radius` | 0.875rem (14px) | Cards, containers |
| `--radius-xl` | `radius × 1.4` | 1.225rem (19.6px) | Cards de destaque, modais |
| `--radius-2xl` | `radius × 1.8` | 1.575rem (25.2px) | Elementos hero |
| `--radius-3xl` | `radius × 2.2` | 1.925rem (30.8px) | Reserved |
| `--radius-4xl` | `radius × 2.6` | 2.275rem (36.4px) | Reserved |

---

## 1.3 Cores — Light Mode

Paleta neutra pura (true gray, sem subtom). Acentos em azul profundo.

### Superfícies e Texto

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--background` | `#FFFFFF` | `oklch(1 0 0)` | Fundo principal |
| `--foreground` | `#0A0A0A` | `oklch(0.145 0 0)` | Texto principal |
| `--card` | `#FFFFFF` | `oklch(1 0 0)` | Fundo de cards |
| `--card-foreground` | `#0A0A0A` | `oklch(0.145 0 0)` | Texto em cards |
| `--popover` | `#FFFFFF` | `oklch(1 0 0)` | Fundo de popovers/dropdowns |
| `--popover-foreground` | `#0A0A0A` | `oklch(0.145 0 0)` | Texto em popovers |

### Cores Semânticas

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--primary` | `#0069A8` | `oklch(0.5 0.134 242.749)` | CTAs, botões primários, ações principais |
| `--primary-foreground` | `#F0F9FF` | `oklch(0.977 0.013 236.62)` | Texto sobre primary |
| `--secondary` | `#F4F4F5` | `oklch(0.967 0.001 286.375)` | Botões secundários, backgrounds sutis |
| `--secondary-foreground` | `#18181B` | `oklch(0.21 0.006 285.885)` | Texto sobre secondary |
| `--muted` | `#F5F5F5` | `oklch(0.97 0 0)` | Backgrounds inativos, áreas recuadas |
| `--muted-foreground` | `#737373` | `oklch(0.556 0 0)` | Texto secundário, placeholders, timestamps |
| `--accent` | `#F5F5F5` | `oklch(0.97 0 0)` | Hover de items de menu, destaques sutis |
| `--accent-foreground` | `#171717` | `oklch(0.205 0 0)` | Texto sobre accent |
| `--destructive` | `#E7000B` | `oklch(0.577 0.245 27.325)` | Ações destrutivas, erros, alertas críticos |

### Bordas e Controles

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--border` | `#E5E5E5` | `oklch(0.922 0 0)` | Bordas de containers, divisores |
| `--input` | `#E5E5E5` | `oklch(0.922 0 0)` | Bordas de inputs e selects |
| `--ring` | `#A1A1A1` | `oklch(0.708 0 0)` | Focus ring |

### Charts

Paleta azul celeste em 5 níveis de luminosidade:

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--chart-1` | `#74D4FF` | `oklch(0.828 0.111 230.318)` | Série 1 — mais claro |
| `--chart-2` | `#00A6F4` | `oklch(0.685 0.169 237.323)` | Série 2 |
| `--chart-3` | `#0084D1` | `oklch(0.588 0.158 241.966)` | Série 3 |
| `--chart-4` | `#0069A8` | `oklch(0.5 0.134 242.749)` | Série 4 |
| `--chart-5` | `#00598A` | `oklch(0.443 0.11 240.79)` | Série 5 — mais escuro |

### Sidebar

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--sidebar` | `#FAFAFA` | `oklch(0.985 0 0)` | Fundo da sidebar |
| `--sidebar-foreground` | `#0A0A0A` | `oklch(0.145 0 0)` | Texto da sidebar |
| `--sidebar-primary` | `#0084D1` | `oklch(0.588 0.158 241.966)` | Item ativo da sidebar |
| `--sidebar-primary-foreground` | `#F0F9FF` | `oklch(0.977 0.013 236.62)` | Texto do item ativo |
| `--sidebar-accent` | `#F5F5F5` | `oklch(0.97 0 0)` | Hover de items da sidebar |
| `--sidebar-accent-foreground` | `#171717` | `oklch(0.205 0 0)` | Texto do item em hover |
| `--sidebar-border` | `#E5E5E5` | `oklch(0.922 0 0)` | Bordas e divisores da sidebar |
| `--sidebar-ring` | `#A1A1A1` | `oklch(0.708 0 0)` | Focus ring na sidebar |

---

## 1.4 Cores — Dark Mode

Inversão semântica: fundos escuros neutros puros, bordas em branco com opacidade.

### Superfícies e Texto

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--background` | `#0A0A0A` | `oklch(0.145 0 0)` | Fundo principal |
| `--foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto principal |
| `--card` | `#171717` | `oklch(0.205 0 0)` | Fundo de cards |
| `--card-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto em cards |
| `--popover` | `#171717` | `oklch(0.205 0 0)` | Fundo de popovers |
| `--popover-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto em popovers |

### Cores Semânticas

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--primary` | `#00598A` | `oklch(0.443 0.11 240.79)` | CTAs, botões primários |
| `--primary-foreground` | `#F0F9FF` | `oklch(0.977 0.013 236.62)` | Texto sobre primary |
| `--secondary` | `#27272A` | `oklch(0.274 0.006 286.033)` | Botões secundários |
| `--secondary-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto sobre secondary |
| `--muted` | `#262626` | `oklch(0.269 0 0)` | Backgrounds inativos |
| `--muted-foreground` | `#A1A1A1` | `oklch(0.708 0 0)` | Texto secundário, placeholders |
| `--accent` | `#262626` | `oklch(0.269 0 0)` | Hover de items de menu |
| `--accent-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto sobre accent |
| `--destructive` | `#FF6467` | `oklch(0.704 0.191 22.216)` | Ações destrutivas (versão mais clara para contraste em dark) |

### Bordas e Controles

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--border` | `#FFFFFF1A` | `oklch(1 0 0 / 10%)` | Bordas — branco 10% |
| `--input` | `#FFFFFF26` | `oklch(1 0 0 / 15%)` | Bordas de inputs — branco 15% |
| `--ring` | `#737373` | `oklch(0.556 0 0)` | Focus ring |

### Charts (Dark)

Mesma paleta azul do light mode — invariante entre temas:

| **Token** | **HEX** | **Nota** |
| --- | --- | --- |
| `--chart-1` | `#74D4FF` | Invariante |
| `--chart-2` | `#00A6F4` | Invariante |
| `--chart-3` | `#0084D1` | Invariante |
| `--chart-4` | `#0069A8` | Invariante |
| `--chart-5` | `#00598A` | Invariante |

### Sidebar (Dark)

| **Token** | **HEX** | **OKLCH** | **Uso** |
| --- | --- | --- | --- |
| `--sidebar` | `#171717` | `oklch(0.205 0 0)` | Fundo da sidebar |
| `--sidebar-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto da sidebar |
| `--sidebar-primary` | `#00A6F4` | `oklch(0.685 0.169 237.323)` | Item ativo — azul vibrante |
| `--sidebar-primary-foreground` | `#052F4A` | `oklch(0.293 0.066 243.157)` | Texto do item ativo |
| `--sidebar-accent` | `#262626` | `oklch(0.269 0 0)` | Hover de items |
| `--sidebar-accent-foreground` | `#FAFAFA` | `oklch(0.985 0 0)` | Texto do item em hover |
| `--sidebar-border` | `#FFFFFF1A` | `oklch(1 0 0 / 10%)` | Bordas — branco 10% |
| `--sidebar-ring` | `#737373` | `oklch(0.556 0 0)` | Focus ring |

---

## 1.5 Tokens Invariantes (não mudam entre temas)

- `--font-heading` — Inter Variable (herda de `--font-sans`)
- `--font-sans` — Inter Variable
- `--radius` e todas as derivações (sm, md, lg, xl, 2xl, 3xl, 4xl)
- `--chart-1` a `--chart-5` — paleta azul celeste

---

# 2. CSS Completo

Pronto para copiar no arquivo de theme do projeto:

```css
/* ============================================
   Repasse Seguro — shadcn Theme Tokens
   v1.0 · 20/03/2026
   ============================================ */

/* Tipografia */
--font-heading: var(--font-sans);
--font-sans: 'Inter Variable', sans-serif;

/* Radius */
--radius: 0.875rem;
--radius-sm: calc(var(--radius) * 0.6);    /* 8.4px  */
--radius-md: calc(var(--radius) * 0.8);    /* 11.2px */
--radius-lg: var(--radius);                /* 14px   */
--radius-xl: calc(var(--radius) * 1.4);    /* 19.6px */
--radius-2xl: calc(var(--radius) * 1.8);   /* 25.2px */
--radius-3xl: calc(var(--radius) * 2.2);   /* 30.8px */
--radius-4xl: calc(var(--radius) * 2.6);   /* 36.4px */

/* ── Light Mode ── */
:root {
  --background:                  oklch(1 0 0);
  --foreground:                  oklch(0.145 0 0);
  --card:                        oklch(1 0 0);
  --card-foreground:             oklch(0.145 0 0);
  --popover:                     oklch(1 0 0);
  --popover-foreground:          oklch(0.145 0 0);
  --primary:                     oklch(0.5 0.134 242.749);
  --primary-foreground:          oklch(0.977 0.013 236.62);
  --secondary:                   oklch(0.967 0.001 286.375);
  --secondary-foreground:        oklch(0.21 0.006 285.885);
  --muted:                       oklch(0.97 0 0);
  --muted-foreground:            oklch(0.556 0 0);
  --accent:                      oklch(0.97 0 0);
  --accent-foreground:           oklch(0.205 0 0);
  --destructive:                 oklch(0.577 0.245 27.325);
  --border:                      oklch(0.922 0 0);
  --input:                       oklch(0.922 0 0);
  --ring:                        oklch(0.708 0 0);
  --chart-1:                     oklch(0.828 0.111 230.318);
  --chart-2:                     oklch(0.685 0.169 237.323);
  --chart-3:                     oklch(0.588 0.158 241.966);
  --chart-4:                     oklch(0.5 0.134 242.749);
  --chart-5:                     oklch(0.443 0.11 240.79);
  --sidebar:                     oklch(0.985 0 0);
  --sidebar-foreground:          oklch(0.145 0 0);
  --sidebar-primary:             oklch(0.588 0.158 241.966);
  --sidebar-primary-foreground:  oklch(0.977 0.013 236.62);
  --sidebar-accent:              oklch(0.97 0 0);
  --sidebar-accent-foreground:   oklch(0.205 0 0);
  --sidebar-border:              oklch(0.922 0 0);
  --sidebar-ring:                oklch(0.708 0 0);
}

/* ── Dark Mode ── */
.dark {
  --background:                  oklch(0.145 0 0);
  --foreground:                  oklch(0.985 0 0);
  --card:                        oklch(0.205 0 0);
  --card-foreground:             oklch(0.985 0 0);
  --popover:                     oklch(0.205 0 0);
  --popover-foreground:          oklch(0.985 0 0);
  --primary:                     oklch(0.443 0.11 240.79);
  --primary-foreground:          oklch(0.977 0.013 236.62);
  --secondary:                   oklch(0.274 0.006 286.033);
  --secondary-foreground:        oklch(0.985 0 0);
  --muted:                       oklch(0.269 0 0);
  --muted-foreground:            oklch(0.708 0 0);
  --accent:                      oklch(0.269 0 0);
  --accent-foreground:           oklch(0.985 0 0);
  --destructive:                 oklch(0.704 0.191 22.216);
  --border:                      oklch(1 0 0 / 10%);
  --input:                       oklch(1 0 0 / 15%);
  --ring:                        oklch(0.556 0 0);
  --chart-1:                     oklch(0.828 0.111 230.318);
  --chart-2:                     oklch(0.685 0.169 237.323);
  --chart-3:                     oklch(0.588 0.158 241.966);
  --chart-4:                     oklch(0.5 0.134 242.749);
  --chart-5:                     oklch(0.443 0.11 240.79);
  --sidebar:                     oklch(0.205 0 0);
  --sidebar-foreground:          oklch(0.985 0 0);
  --sidebar-primary:             oklch(0.685 0.169 237.323);
  --sidebar-primary-foreground:  oklch(0.293 0.066 243.157);
  --sidebar-accent:              oklch(0.269 0 0);
  --sidebar-accent-foreground:   oklch(0.985 0 0);
  --sidebar-border:              oklch(1 0 0 / 10%);
  --sidebar-ring:                oklch(0.556 0 0);
}
```

---

# 3. Referência Rápida de Cores (HEX)

| **Cor** | **Light** | **Dark** | **Papel** |
| --- | --- | --- | --- |
| Background | `#FFFFFF` | `#0A0A0A` | Fundo principal |
| Foreground | `#0A0A0A` | `#FAFAFA` | Texto principal |
| Primary | `#0069A8` | `#00598A` | CTAs, ações primárias |
| Secondary | `#F4F4F5` | `#27272A` | Ações secundárias |
| Muted text | `#737373` | `#A1A1A1` | Labels, placeholders |
| Destructive | `#E7000B` | `#FF6467` | Erros, exclusões |
| Border | `#E5E5E5` | `#FFFFFF1A` | Divisores, bordas |
| Ring | `#A1A1A1` | `#737373` | Focus states |
| Chart accent | `#0084D1` | `#0084D1` | Cor principal de gráficos (chart-3) |
| Sidebar active (dark) | `#0084D1` | `#00A6F4` | Item ativo da navegação |

---

# 4. Componentes

Todos os componentes vêm do **shadcn/ui** sem customização estrutural. A fonte da verdade para anatomia, estados, variantes e acessibilidade (ARIA) é a documentação oficial do shadcn:

🔗 [ui.shadcn.com/docs/components](http://ui.shadcn.com/docs/components)

<aside>
⚙️

**Regra:** Se um componente do shadcn não atender a uma necessidade específica, documentar o componente custom aqui nesta seção com: variantes, tokens consumidos e requisitos de acessibilidade.

</aside>

---

# 5. Changelog

| **Versão** | **Data** | **Descrição** |
| --- | --- | --- |
| 1.0 | 20/03/2026 | Versão inicial — Theme Configuration. Inter Variable, radius 14px (generoso), paleta neutro puro com charts azul celeste. |