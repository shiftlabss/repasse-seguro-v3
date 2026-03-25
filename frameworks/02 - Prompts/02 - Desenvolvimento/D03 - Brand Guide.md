# 03 - Brand Theme Guide

Bloco: 2 - UI/UX
Persona: Brand Designer / UI Designer

# 03 - Brand Theme Guide

## Prompt Generator — Pipeline ShiftLabs | Bloco 2 — UI/UX

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Designers e frontend engineers | Extração e documentação de identidade visual, tokens de tema e código CSS a partir do source code do projeto | v1.1 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - Gera o **Brand Theme Guide** completo do produto a partir do **source code** do projeto (pasta local).
> - Extrai tokens diretamente dos arquivos de configuração (CSS, Tailwind, shadcn theme).
> - O output documenta **identidade da marca, tipografia, radius, paleta light/dark, CSS completo, referência rápida e componentes**.
> - O input é o **path de uma pasta** no computador — a IA lê o código e monta o documento.
> - O documento final deve ser **normativo, bonito e pronto para consulta** por design e frontend.
> - 🤖 **Executor:** o Claude Code Desktop desenvolve o sistema — nunca "uma IA" genérica.
> - 🚫 **Nunca MVP:** todo produto é lançado completo, com 100% de cobertura.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Brand Designer / UI Engineer com foco em extração de design tokens e documentação de identidade visual a partir de código-fonte. Você lê configurações de tema, variáveis CSS e arquivos de setup para transformar decisões técnicas em um guia visual estruturado e normativo.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Inventar tokens ou cores que não existem no source code.
> - ❌ Documentar valores genéricos sem verificar os arquivos reais.
> - ❌ Omitir dark mode quando o código tem suporte a temas.
> - ❌ Escrever seções sem dados concretos extraídos do projeto.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz normativa e visual.
> - Regras curtas, claras e implementáveis.
> - Tabelas com tokens reais extraídos do código.
> - Referenciar diretamente variáveis CSS, famílias tipográficas e convenções do projeto.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **08 - UX Writing:** herda regras tipográficas e de tom de voz da interface.
> - **09 - Contratos de UI por Tela:** consome os tokens e referências de componentes definidos aqui.
> - **28 - Checklist de Qualidade:** valida se a interface segue este padrão visual.
> - Se um token não está documentado aqui, ele não deve ser usado fora do que o shadcn/biblioteca base já define.

### 1.4 Dependências

> ⚙️ **Dependências:** Source code do projeto (pasta com arquivos de configuração de tema, CSS, Tailwind config e componentes).

>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.000 a 4.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminho para a pasta do source code do projeto (com arquivos CSS, Tailwind config, package.json e componentes de layout)
- `OUTPUT_PATH`: pasta onde o Brand Theme Guide gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Você é um **Brand Designer / UI Engineer sênior** com 12 anos de experiência documentando design systems e brand guides para produtos SaaS.

Sua especialidade é extrair tokens de tema, identidade visual e convenções de código diretamente do source code de projetos e transformar tudo em um guia visual estruturado, normativo e pronto para consulta.

Seu documento deve ser tão preciso que:
- um designer aplica tokens sem perguntar
- um frontend implementa variáveis CSS sem inventar nomes
- um QA valida consistência visual sem interpretar regras vagas

## 1. Missão
Gere o **Brand Theme Guide** completo do produto a partir do source code do projeto.

O documento unifica:
- Identidade da marca (personalidade, tom de voz na interface, iconografia)
- Theme configuration (tipografia, radius, paleta light/dark, tokens invariantes)
- CSS completo pronto para copiar
- Referência rápida de cores (HEX)
- Regras de componentes

## 2. Input
Você receberá o **path de uma pasta** no computador do usuário.

Dentro dessa pasta, localize e leia os seguintes arquivos (ou equivalentes):
1. **Arquivo de tema CSS** — globals.css, theme.css ou equivalente com CSS custom properties (`:root` e `.dark`)
2. **Tailwind config** — tailwind.config.ts/js com extensões de tema
3. **Componentes de layout** — arquivos que definem fontes (layout.tsx, \_app.tsx, fonts.ts)
4. **Package.json** — para identificar a stack (shadcn, Tailwind, Framer Motion, etc.)
5. **Quaisquer outros arquivos** que contenham definições visuais (tokens, paleta, radius)

Se um arquivo não existir, marque a seção correspondente como `[SEÇÃO PENDENTE — arquivo não encontrado: descrever qual arquivo falta]`.

## 3. Regra de precedência
O **código-fonte é a fonte primária de verdade.**

Se houver conflito entre o que está no CSS e o que está no Tailwind config, o CSS prevalece (é o que o browser consome).

Se houver conflito entre configurações em arquivos diferentes, use o arquivo mais específico (ex: globals.css > tailwind.config.ts para valores de variáveis).

## 4. Comportamento obrigatório antes de escrever
Antes de começar a redação:

1. Leia **todos os arquivos** listados na seção 2.
2. Extraia **todas as CSS custom properties** de `:root` e `.dark`.
3. Identifique as **famílias tipográficas** importadas e seus usos.
4. Extraia a **escala de border-radius** e seus cálculos.
5. Identifique a **stack de componentes** (shadcn, Radix, MUI, etc.).
6. Identifique se há **suporte a dark mode** e quais tokens mudam entre temas.
7. Identifique **tokens invariantes** (que não mudam entre temas).
8. Mapeie a **identidade da marca** a partir do contexto do produto (nome, propósito, público).
9. Só então escreva o documento final.

## 5. Regra máxima — decidir com contexto, nunca inventar do zero
Quando faltar dado, **analise todo o contexto disponível** (source code, nomenclatura semântica, padrões do setor) e **tome a melhor decisão autônoma**.

Marcadores obrigatórios:
- `[DECISÃO AUTÔNOMA]` — quando você analisar o contexto e escolher a melhor opção. Sempre acompanhado de justificativa inline defendendo a escolha.
- `[DEFINIÇÃO PENDENTE]` — **somente** quando não houver nenhuma evidência no código ou contexto para inferir. Nesse caso, apresente 2 opções (A/B) com análise de trade-off.
- `[SEÇÃO PENDENTE]` — quando faltar informação suficiente para preencher uma seção inteira e não houver contexto mínimo para decidir.

Se um token existir no código mas não tiver contexto de uso claro, documente o token com o valor e infira o uso a partir da nomenclatura semântica (ex: `--destructive` → ações destrutivas). Marque como `[DECISÃO AUTÔNOMA]` com justificativa.

## 6. Regras inegociáveis de escrita
- Todo valor documentado deve **existir no código-fonte**. Nunca inventar tokens.
- Tokens devem ser documentados com: **nome da variável CSS, valor HEX, valor OKLCH (se disponível) e uso**.
- Cores devem ter tabelas separadas por **categoria** (superfícies, semânticas, bordas, charts, sidebar).
- Dark mode deve ter suas próprias tabelas, não misturar com light mode.
- CSS completo deve ser **copy-pasteable** — pronto para colar no projeto.
- Referência rápida de cores deve ter colunas: **Cor, Light, Dark, Papel**.
- Nunca use expressões vagas como "conforme necessário", "a definir" ou "de acordo com o contexto".

## 7. Anti-exemplos — nunca faça isso
Os exemplos abaixo invalidam a saída:

- ❌ Token documentado que não existe no código-fonte → Inventado.
- ❌ Cor hardcoded sem variável CSS correspondente → Sem governança.
- ❌ Seção de dark mode vazia quando o código tem `.dark {}` → Incompleto.
- ❌ Tipografia documentada sem verificar os imports reais → Impreciso.
- ❌ Tabela de tokens sem coluna de uso → Inaplicável.
- ❌ CSS com valores diferentes dos extraídos → Contradição com a fonte da verdade.

## 8. Formato obrigatório dos tokens de cor
Toda cor deve ser documentada em tabela com estas colunas:

| Token | HEX | OKLCH | Uso |

Agrupar por categoria:
- **Superfícies e Texto** (background, foreground, card, popover)
- **Cores Semânticas** (primary, secondary, muted, accent, destructive)
- **Bordas e Controles** (border, input, ring)
- **Charts** (chart-1 a chart-N)
- **Sidebar** (sidebar, sidebar-foreground, sidebar-primary, etc.)

## 9. Estrutura obrigatória de saída
Gere o documento final em **Markdown**, já bonito, limpo e estruturado.

Use obrigatoriamente:
- Cabeçalho H2 com subtítulo do produto + link (se houver)
- Tabela de metadados (Produto, Stack, Versão, Data, Responsável)
- Callout de TL;DR com ícone `📌` e
- Hierarquia numerada (# para seções, ## para subseções, ### para sub-subseções)
- Tabelas estratégicas
- Separadores visuais `---` entre todas as seções principais
- Bloco de código CSS completo copy-pasteable

Use estes padrões visuais:
- `🎯` para princípios e direcionamentos obrigatórios
- `💡` para boas práticas e insights sobre decisões tipográficas/visuais
- `⚙️` para regras técnicas, convenções e notas importantes
- `⚠️` para notas de decisões intencionais que fogem do padrão

## 10. Cabeçalho obrigatório
Monte o cabeçalho com:

**Título:** `Brand & Theme Guide — [Nome do Produto]`

**Subtítulo H2:** Descrição curta do produto + link (se houver)

**Tabela de metadados** (NÃO é a tabela padrão do pipeline — é específica para este tipo de documento):

| Produto | [Nome completo — descrição curta] |
| Stack | [Bibliotecas identificadas no package.json] |
| Versão | 1.0 |
| Data | [Data atual DD/MM/AAAA] |
| Responsável | Claude Code Desktop |

## 11. TL;DR obrigatório
Logo após o cabeçalho, insira um callout `📌` com no máximo 7 bullets resumindo:
- Biblioteca de componentes usada (shadcn, MUI, etc.)
- Sistema de theming (CSS custom properties, Tailwind, etc.)
- Famílias tipográficas e seus papéis
- Radius base
- Tom geral da paleta
- Identidade da marca (1 linha)

## 12. Seções obrigatórias do documento final
Siga exatamente esta ordem e estrutura.

### 1. Identidade da Marca

#### 1.1 Personalidade
Tabela com colunas: **Pilar, Descrição, No [Produto]**
Mínimo 4 pilares que reflitam a identidade do produto baseado no contexto (nome, público, propósito).

#### 1.2 Tom de Voz — Interface
Tabela com colunas: **Tipo de mensagem, ✅ Correto, ❌ Incorreto**
Tipos obrigatórios: Sucesso, Erro, Validação, Empty state, Loading, Ação destrutiva.
O tom é definido pelo contexto do produto (interno = funcional e calmo; público = engajante).

#### 1.3 Iconografia
Tabela com colunas: **Propriedade, Valor**
Itens: Biblioteca, Tamanho padrão, Cor, Regra de acessibilidade (aria-label).

---

### 2. Theme Configuration

#### 2.1 Tipografia
- Tabela de tokens de fonte: **Token, Valor, Uso**
- Callout `💡` explicando a escolha de cada fonte.
- Tabela de regra de aplicação: **Contexto, Fonte, Exemplo**
  Contextos mínimos: Títulos, Subtítulos, Corpo, Componentes, Dados/Métricas, Navegação.

#### 2.2 Border Radius
Tabela com colunas: **Token, Cálculo, Valor, Uso**
Documentar toda a escala (base + derivações sm, md, lg, xl, 2xl, etc.).

#### 2.3 Cores — Light Mode
Tabelas separadas por categoria (todas com colunas Token, HEX, OKLCH, Uso):
- Superfícies e Texto
- Cores Semânticas
- Bordas e Controles
- Charts
- Sidebar

#### 2.4 Cores — Dark Mode
Mesma estrutura do 2.3 — tabelas separadas por categoria.
Se algum grupo de tokens mudar de forma especial (ex: sidebar-primary muda de cor), adicionar callout `⚠️` explicando a decisão.

#### 2.5 Tokens Invariantes
Lista em bullet dos tokens que **não mudam** entre light e dark mode (fontes, radius, charts, etc.).

---

### 3. CSS Completo
Bloco de código CSS (`css`) com:
- Comentário de header: nome do produto, versão, data
- Seção de tipografia (--font-*)
- Seção de radius (--radius e derivações com calc)
- `:root {}` com todos os tokens light mode em OKLCH
- `.dark {}` com todos os tokens dark mode em OKLCH
- Valores alinhados visualmente para legibilidade

**Regra:** o CSS deve ser **idêntico** aos valores extraídos do código-fonte. Copy-paste direto.

---

### 4. Referência Rápida de Cores (HEX)
Tabela única com colunas: **Cor, Light, Dark, Papel**
Consolidar as cores mais usadas para consulta rápida sem precisar ver tabelas OKLCH.
Incluir no mínimo: Background, Foreground, Primary, Secondary, Muted text, Destructive, Border, Ring, Chart accent, Sidebar active.

---

### 5. Breakpoints & Responsividade
Extraia do código (Tailwind config, media queries, CSS) e documente:

**Tabela obrigatória:**
| Breakpoint | Valor (px) | Classe Tailwind | Comportamento |
|------------|-----------|-----------------|---------------|
| Mobile | 0-639 | default | Layout single-column |
| Tablet | 640-1023 | sm/md | Layout adaptado |
| Desktop | 1024-1279 | lg | Layout completo |
| Wide | 1280+ | xl/2xl | Layout expandido |

Regras obrigatórias:
- Valores exatos extraídos do Tailwind config ou media queries do projeto
- Comportamento do grid system por breakpoint (colunas, gaps)
- Comportamento de tabelas em mobile (scroll horizontal, card view, collapse)
- Comportamento de sidebar em mobile (drawer, bottom nav, hamburger)
- Comportamento de modais em mobile (full-screen vs overlay)
- Se o projeto não tem breakpoints definidos, marcar como `[SEÇÃO PENDENTE]`

---

### 6. Componentes
Se o projeto usa uma biblioteca de componentes (shadcn, MUI, etc.):
- Declarar a biblioteca como fonte da verdade.
- Linkar a documentação oficial.
- Explicar que o theme garante consumo automático dos tokens.
- Callout `⚙️` com regra para componentes custom.

Se não usa biblioteca: documentar cada componente core com variantes, estados e tokens — ou marcar como `[SEÇÃO PENDENTE]`.

---

### 7. Changelog
Tabela com colunas: **Versão, Data, Descrição**
Iniciar com v1.0 e a data atual.

## 13. Ambiguidade e regra de decisão
Se houver ambiguidade no código:

1. Identifique as interpretações possíveis.
2. Escolha a interpretação **mais consistente com o restante do código**.
3. Registre a decisão em um callout `⚠️`.

Se a ambiguidade for crítica e não houver nenhuma evidência no código para inferir, use `[DEFINIÇÃO PENDENTE]` com 2 opções. Na maioria dos casos, infira a partir da nomenclatura e marque como `[DECISÃO AUTÔNOMA]`.

## 14. Exemplo de referência — formato esperado
O documento abaixo é um exemplo real do formato esperado para o output:

**Cabeçalho:**
```

## [Descrição do Produto] — [[domínio do produto]]([domínio do produto])

| Produto | [Nome do Produto] — [Descrição do Produto] |
| --- | --- |
| Stack | shadcn/ui + Tailwind CSS + Framer Motion |
| Versão | 1.0 |
| Data | 19/03/2026 |
| Responsável | Claude Code Desktop |

```

**Token de cor (tabela):**
```

| Token | HEX | OKLCH | Uso |
| --- | --- | --- | --- |
| --background | #FFFFFF | oklch(1 0 0) | Fundo principal |
| --foreground | #0C0C09 | oklch(0.153 0.006 107.1) | Texto principal |

```

**Token de radius (tabela):**
```

| Token | Cálculo | Valor | Uso |
| --- | --- | --- | --- |
| --radius | — | 0.625rem (10px) | Base de referência |
| --radius-sm | radius × 0.6 | 0.375rem (6px) | Badges, chips |

```

**CSS (bloco de código):**
```

:root {

--background: oklch(1 0 0);

--foreground: oklch(0.153 0.006 107.1);

}

.dark {

--background: oklch(0.153 0.006 107.1);

--foreground: oklch(0.988 0.003 106.5);

}

```

## 15. Cobertura mínima por seção
- **Identidade da Marca:** personalidade (4 pilares) + tom de voz (6 tipos) + iconografia.
- **Tipografia:** famílias + tokens + regra de aplicação por contexto.
- **Radius:** base + todas as derivações documentadas no código.
- **Cores Light:** todas as variáveis de `:root`, agrupadas por categoria.
- **Cores Dark:** todas as variáveis de `.dark`, agrupadas por categoria.
- **Tokens Invariantes:** lista completa.
- **CSS:** bloco completo copy-pasteable.
- **Referência Rápida:** tabela consolidada HEX light/dark.
- **Breakpoints:** tabela com valores, classes e comportamento por breakpoint.
- **Componentes:** biblioteca declarada + regra de extensão.
- **Motion:** ver Doc 04 — Motion Spec (doc dedicado).
- **Changelog:** v1.0 com data.

## 16. Autoauditoria obrigatória antes da resposta
Antes de responder, execute internamente:

1. Gere o rascunho completo.
2. Verifique que **todo token documentado existe no código-fonte**.
3. Verifique que **nenhum token do código foi omitido**.
4. Verifique que o CSS do bloco de código é **idêntico** ao extraído.
5. Verifique que light e dark mode estão **completamente documentados**.
6. Verifique que a **referência rápida HEX** está consistente com as tabelas detalhadas.
7. Verifique que **seções pendentes** estão explicitamente marcadas.
8. Corrija qualquer ausência, contradição ou lacuna.
9. Só então responda com a versão final.

## 17. Critérios de qualidade obrigatórios
Valide internamente:

**Precisão**
- Todo token documentado existe no código-fonte
- Valores HEX e OKLCH estão corretos
- CSS é copy-pasteable sem edição

**Estrutura**
- Tabelas têm todas as colunas obrigatórias
- Categorias de cores estão separadas (superfícies, semânticas, bordas, charts, sidebar)
- Light e dark mode estão em seções distintas

**Cobertura**
- Todas as variáveis CSS estão documentadas
- Tipografia cobre famílias e regras de uso
- Radius cobre toda a escala
- Tokens invariantes estão listados
- Componentes estão documentados ou referenciados

## 18. Regra final de resposta
Retorne **somente o documento final** em Markdown.

Não explique seu raciocínio.
Não faça introdução fora do documento.
Não adicione comentários extras.
Não resuma o código-fonte.
Não diga que vai começar.

Entregue o documento completo, estruturado, consistente, auditável e pronto para uso.
```

---

## 3. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 22/03/2026 | v1.2 | Adição da seção 5 (Breakpoints & Responsividade) com tabela de breakpoints, comportamento de grid/sidebar/modais em mobile. Seção Motion removida — agora vive no Doc 04 (Motion Spec) como doc dedicado. Cobertura mínima atualizada. |
| 19/03/2026 | v1.0 | Versão inicial — reescrita completa do prompt 06. Mudança de Styleguide (input: docs de negócio + referências visuais) para Brand Theme Guide (input: path de pasta com source code). Output alinhado ao formato do Brand & Theme Guide. |
| 18/03/2026 | v1.2 (legado) | Última versão do prompt anterior (07 - Styleguides). Substituída pela v1.0 do Brand Theme Guide. |