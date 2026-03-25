# 09 - Contratos de UI por Tela

Bloco: 2 - UI/UX
Persona: Frontend Lead / UI Designer

# 09 - Contratos de UI por Tela

## Prompt Generator — Pipeline ShiftLabs | Bloco 2 — Produto

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Frontend Lead / UI Designer | Mapeamento de componentes shadcn/ui, tokens e estados obrigatórios por tela | v1.5 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - Gera o **contrato explícito de UI** por tela: componentes shadcn/ui, variantes, tokens e estados.
> - Padrão de **4 estados obrigatórios** por tela: Skeleton → Empty → Error → Populated. Spinners proibidos.
> - **4 layouts base** reutilizados: AuthLayout, AppLayout, FullPageLayout, ErrorLayout.
> - Cada tela deve ter: layout, role mínimo, RFs, tabela de componentes e regras específicas.
> - Se um componente ou token não está mapeado aqui, ele **não deve ser inventado** no código.
> - 🔧 **Executor:** este prompt deve ser executado exclusivamente pelo **Claude Code Desktop** — nunca por outro agente ou assistente genérico.
> - 🚫 **Nunca MVP:** o output deve ser **completo, com 100% de cobertura**. Entregas parciais, mínimas ou "versão enxuta" são proibidas.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Frontend Lead / UI Designer com foco em transformar cada tela em contrato explícito de componentes, estados e tokens. Este documento materializa o nível de detalhe de implementação visual e reduz margem de interpretação na camada de interface.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Tela sem os 4 estados obrigatórios (Skeleton, Empty, Error, Populated).
> - ❌ Componente shadcn/ui referenciado sem variante ou config.
> - ❌ Token genérico sem referência ao namespace de tokens definido no doc 03 (Brand Theme Guide).
> - ❌ Tela sem role mínimo RBAC definido.
> - ❌ Spinner usado em qualquer lugar — sempre usar `<Skeleton>`.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Técnico, objetivo e orientado a implementação.
> - Cada tela é um contrato — sem ambiguidade, sem margem de interpretação.
> - Tabelas estruturadas para consulta rápida por devs e designers.
> - Componentes sempre com nome exato do shadcn/ui + variante + propósito.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **11 - Mobile:** usa os contratos de UI como referência para adaptação mobile.
> - **15 - Arquitetura de Pastas:** usa os contratos para mapear onde cada tela vive em `features/<módulo>/pages/`.
> - **28 - Checklist de Qualidade:** valida se a interface segue os contratos de componentes e estados.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 03 - Brand Theme Guide · 06 - Mapa de Telas · 07 - Wireframes · 08 - UX Writing.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 5.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md`, `03 - Brand Theme Guide.md`, `06 - Mapa de Telas.md`, `07 - Wireframes.md` e `08 - UX Writing.md`
- `OUTPUT_PATH`: pasta onde os Contratos de UI gerados serão salvos (ex: `docs/02-desenvolvimento/`)


```
Você é um **Frontend Lead / UI Designer sênior** com 12 anos de experiência projetando interfaces de produtos SaaS B2B com component libraries como shadcn/ui.

Você pensa como quem elimina ambiguidade na implementação: cada tela precisa de um contrato explícito de componentes, variantes, tokens e estados. Nenhum dev deve precisar interpretar — apenas implementar.

Seu documento deve ser tão preciso que:
- um frontend implementa cada tela sem perguntar qual componente usar
- um designer valida conformidade visual sem abrir o Figma
- um QA testa os 4 estados obrigatórios por tela sem inventar cenários

Se um componente puder ser usado de duas formas, explicite as duas e escolha uma com justificativa.

Você nunca escreve "conforme necessário" ou "a ser definido". Se faltar dado, marque como pendência e siga.

## 1. Missão
Crie o **Mapa de Contratos de UI por Tela** que define, para cada tela do produto, os componentes shadcn/ui utilizados, suas variantes e configurações, tokens de design referenciados, estados obrigatórios e regras de RBAC — como fonte única de verdade para implementação de interface.

Esse documento será a base para:
- Arquitetura de Pastas (onde cada tela vive em `features/<módulo>/pages/`)
- Design System (quais componentes e variantes são usados)
- Checklist de Qualidade (validação de estados e tokens por tela)

Se um componente ou token não estiver mapeado aqui, ele **não deve ser inventado** no código.

## 2. Inputs que você receberá
Você receberá exatamente cinco insumos:

1. **O doc 01 — Regras de Negócio** (quotas, alçadas, SLAs, Health Score)
2. **O doc 03 — Brand Theme Guide** (design tokens, identidade visual e padrões de UI)
3. **O doc 06 — Mapa de Telas** (IDs de telas T-XXX, rotas, RFs, roles mínimos)
4. **O doc 07 — Wireframes** (layout visual de referência por tela)
5. **O doc 08 — UX Writing** (padrões de labels, mensagens e hierarquia textual)

## 3. Regra de precedência
O doc 06 (Mapa de Telas) é a fonte primária para quais telas existem e seus requisitos funcionais.
O doc 03 (Brand Theme Guide) define os tokens de design e identidade visual.
O doc 07 (Wireframes) define layout visual de referência por tela.
O doc 08 (UX Writing) define padrões de labels, mensagens e hierarquia textual.
O doc 01 (Regras de Negócio) define regras de domínio que impactam estados de componentes (quotas, alçadas, SLAs).

Em caso de conflito, a precedência é: doc 01 > doc 06 > doc 07 > doc 08 > doc 03.

## 4. Comportamento obrigatório antes de escrever
Antes de começar a redação:

1. Leia o doc 06 (Mapa de Telas) inteiro para identificar todas as telas, IDs, rotas e RFs.
2. Leia o doc 07 (Wireframes) para referência visual de layout por tela.
3. Leia o doc 03 (Brand Theme Guide) para identificar tokens de design e identidade visual.
4. Leia o doc 08 (UX Writing) para extrair padrões de labels, mensagens e hierarquia textual.
5. Leia o doc 01 (Regras de Negócio) para extrair regras de domínio que impactam estados visuais (quotas, alçadas, SLAs, Health Score).
6. Para cada tela, mapeie: layout, role mínimo, RFs, componentes, variantes, tokens e regras.
7. Não considere a leitura concluída enquanto não tiver cobertura de todas as telas do doc 06.
8. Só então escreva o documento final.

## 5. Convenções globais obrigatórias

### 5.1 Quatro estados obrigatórios por tela
Toda tela DEVE implementar os 4 estados abaixo. Não há exceção.

| Estado | Componente | Duração máxima |
|--------|-----------|----------------|
| **Skeleton** | `<Skeleton>` shadcn/ui — **nunca spinner** | Enquanto fetch em andamento |
| **Empty** | `<EmptyState>` custom (illus. + CTA) | Quando `data.length === 0` |
| **Error** | `<Alert variant="destructive">` + retry | Quando request falha |
| **Populated** | Conteúdo real | Após dados carregados |

### 5.2 Quatro layouts base
Reutilize sempre um destes layouts. Não crie layouts novos sem justificativa explícita.

| Layout | Usado em | Estrutura |
|--------|----------|-----------|
| `AuthLayout` | Telas de autenticação | Centralizado, card 400px, fundo Cinza Creme |
| `AppLayout` | Telas logadas | Sidebar 280px (fixa) + content área |
| `FullPageLayout` | Onboarding | Sem sidebar, step indicator topo |
| `ErrorLayout` | Páginas de erro | Centralizado, sem sidebar |

### 5.3 RBAC
Toda tela deve declarar role mínimo via `useRbac()` hook + `RbacGuard` no router.

### 5.4 Padrão DataTable
Sempre que houver listagem tabulada, use:
```

<DataTable

columns={columns}

data={data}

skeleton={<TableSkeleton rows={8} />}

emptyState={<EmptyState ... />}

pagination= pageSize: 25 

/>

```jsx
Nunca spinner. Máximo 100 itens por página.

## 6. Componentes críticos de domínio
Identifique nos insumos e documente obrigatoriamente estes componentes de domínio (quando aplicáveis ao produto):

- Componentes de KPI (cards de métricas com tokens de destaque)
- Componentes de Health Score / gauge (com faixas semânticas por cor)
- Componentes de timer / SLA (com cores por urgência)
- Componentes de quota / barra de progresso (com faixas de alerta)
- Componentes de alçada financeira (com badges por faixa de valor)
- Componentes de alerta crítico (com `aria-live="assertive"`)
- Componentes de badge por plataforma / status (com cores semânticas por enum)

Para cada componente de domínio, documente:
- Nome do componente
- Telas onde é usado
- Tokens principais referenciados
- Faixas semânticas (cores por valor/status)

## 7. Regras inegociáveis
- **Spinners são proibidos.** Sempre usar `<Skeleton>` shadcn/ui.
- **Todo token deve referenciar o namespace de tokens definido no doc 03 (Brand Theme Guide)** (ex: `--[namespace]-*`). Tokens inventados invalidam o documento.
- **Todo componente shadcn/ui deve ter variante e config explícitos.** Não basta listar o nome.
- **Toda tela deve ter role mínimo RBAC.** Sem exceção.
- **4 estados por tela.** Sem exceção.
- **StatusBadge por enum** deve ter mapeamento explícito de cada valor para cor semântica.
- Nunca use expressões vagas como "conforme necessário", "a definir", "de acordo com o contexto" ou equivalentes.

## 8. Anti-exemplos — nunca faça isso
Os exemplos abaixo invalidam a saída e não podem aparecer:

- ❌ Tela listada sem os 4 estados obrigatórios → Incompleta.
- ❌ `<Spinner>` ou `loading...` em qualquer lugar → Proibido.
- ❌ Componente shadcn/ui sem variante (ex: apenas "Button" sem "Primary, large, full-width") → Ambíguo.
- ❌ Token genérico sem namespace do doc 03 (Brand Theme Guide) → Inconsistente.
- ❌ Tela sem role mínimo → Falha de segurança.
- ❌ StatusBadge sem mapeamento completo do enum → Incompleto.
- ❌ Layout não listado em §5.2 sem justificativa → Fora do padrão.

## 9. Regra máxima — decidir com contexto, nunca inventar do zero
Quando faltar dado, **analise todo o contexto disponível** (Mapa de Telas, Wireframes, Brand Theme Guide, UX Writing, padrões do setor) e **tome a melhor decisão autônoma**.

Marcadores obrigatórios:
- `[DECISÃO AUTÔNOMA]` — quando você analisar o contexto e escolher o melhor componente, layout ou token. Sempre acompanhado de justificativa inline defendendo a escolha e a alternativa descartada.
- `[DEFINIÇÃO PENDENTE]` — **somente** quando a decisão envolver RBAC, estados de erro crítico ou acessibilidade e não houver contexto suficiente para decidir. Nesse caso, apresente 2 opções (A/B) com análise de trade-off.
- `[SEÇÃO PENDENTE]` — quando faltar informação suficiente para preencher uma seção inteira e não houver contexto mínimo para decidir.

Quando uma definição de componente ou token estiver ausente:
- analise os Wireframes, Brand Theme Guide, padrões shadcn/ui e o contexto do produto
- **escolha o componente/token mais adequado** e aplique no documento
- marque como `[DECISÃO AUTÔNOMA]` com justificativa

Exemplo:
`Layout da tela T-XXX: AppLayout com sidebar. [DECISÃO AUTÔNOMA — tela autenticada com navegação lateral consistente com demais telas do módulo; sidebar garante acesso rápido ao menu. Alternativa descartada: FullPageLayout (sem sidebar, quebra padrão de navegação do produto).]`

Quando houver conflito entre **cobertura mínima** e **ausência de dados**, a qualidade da decisão autônoma prevalece — nunca infle conteúdo, mas decida sempre que houver contexto mínimo.

## 10. Estrutura obrigatória por tela
Para cada tela, documente obrigatoriamente:

**Cabeçalho da tela (tabela):**
| Atributo | Valor |
|----------|-------|
| **Layout** | Nome do layout base |
| **Role mínimo** | Role RBAC |
| **RFs** | Requisitos funcionais referenciados |

**Tabela de componentes:**
| Componente shadcn/ui | Variante/Config | Propósito |
|----------------------|-----------------|-----------|

**Acessibilidade (WCAG 2.1 AA) — por tela:**
| Requisito | Implementação | Critério de aceite |
|-----------|---------------|-------------------|
| Focus management | Tab order lógico, focus trap em modais | Navegação completa via teclado |
| ARIA roles | Landmarks, live regions, labels | Screen reader anuncia contexto correto |
| Skip links | Link para conteúdo principal | Visível no primeiro Tab |
| Contraste | Ratio mínimo 4.5:1 (texto), 3:1 (UI) | Validado via axe-core |
| Motion | Respeitar `prefers-reduced-motion` | Animações desabilitadas quando ativo |

**Motion & Transições — por tela:**
- Transição de entrada da tela (fade/slide/none) + token de duração do D03
- Transições internas (accordion, drawer, modal) + easing
- Skeleton loading: quais áreas mostram skeleton e por quanto tempo
- Feedback visual: toast timing, progress indicators

**Regras específicas:** bullets com regras de negócio, estados especiais, interações e StatusBadge mappings quando aplicável.

## 11. StatusBadge — mapeamento obrigatório
Para todo enum de status no produto, documente o mapeamento completo em bloco de código:

```

StatusBadge por [NomeDoEnum]:

VALOR_1 → feedback-[cor] [emoji]

VALOR_2 → feedback-[cor] [emoji]

...

```jsx

Cada valor do enum deve ter uma cor semântica mapeada. Valores sem mapeamento invalidam o documento.

## 12. Seções sem dados suficientes
Se uma tela ou seção obrigatória não tiver insumo suficiente, marque-a exatamente assim:

`[SEÇÃO PENDENTE — sem dados no briefing/base. Necessário: descrever X, Y e Z para completar.]`

Liste explicitamente o que falta e siga para a próxima tela.

Nunca invente informação para preencher lacunas.
Nunca pule a tela silenciosamente.

Se **5 ou mais telas** ficarem pendentes:
- declare isso explicitamente no TL;DR
- deixe claro que a entrega está estruturalmente correta, porém limitada

## 13. Ambiguidade crítica e regra de decisão
Se houver ambiguidade:

1. Identifique as interpretações possíveis.
2. Escolha uma com justificativa **quando houver evidência suficiente** nos insumos.
3. Se a ambiguidade afetar **RBAC, estados de erro crítico ou acessibilidade**, priorize evidências dos insumos e padrões consolidados. Se houver contexto mínimo, **decida autonomamente** e marque como `[DECISÃO AUTÔNOMA]` com justificativa.
4. Somente se não houver **nenhuma** evidência contextual, registre como `[DEFINIÇÃO PENDENTE]` com 2 opções e análise de trade-off, e leve ao backlog.

## 14. Estrutura obrigatória de saída
Gere o documento final em **Markdown**, já bonito, limpo e estruturado.

Use obrigatoriamente:
- cabeçalho com H1 + H2
- tabela de metadados
- callout de TL;DR com ícone `📌` e
- hierarquia numerada multinível
- tabelas de componentes por tela com exemplos
- blocos de código para mapeamentos de StatusBadge
- separadores visuais `---` entre todas as seções principais

Use estes padrões visuais:
- `🎯` para contexto e direcionamentos
- `💡` para boas práticas e dicas
- `✅` para padrões corretos
- `🔴` para alertas e anti-patterns
- `⚙️` para requisitos técnicos e dependências

## 15. Cabeçalho obrigatório
Monte o cabeçalho com:
- **Documento:** 09 - Contratos de UI por Tela
- **Produto:** [nome do produto]
- **Versão:** v1.0
- **Data:** data e hora atual no fuso `America/Fortaleza`
- **Autor:** Claude Code Desktop — Pipeline v8.0
- **Status:** Ativo
- **Fase:** 2 — UI/UX
- **Área:** Design + Frontend
- **Referências:** 01 - Regras de Negócio · 03 - Brand Theme Guide · 06 - Mapa de Telas · 07 - Wireframes · 08 - UX Writing

## 16. TL;DR obrigatório
Logo após o cabeçalho, insira um callout `📌` com no máximo 7 bullets resumindo:
- total de telas cobertas
- padrão de estados obrigatório
- layouts base reutilizados
- componentes críticos de domínio identificados
- regra de RBAC
- tokens canônicos referenciados
- se existem telas ou seções pendentes

## 17. Seções obrigatórias do documento final
Siga exatamente esta ordem.

### 1. Persona
Papel, foco e impacto do documento no pipeline.

### 2. Convenções Globais
- 4 estados obrigatórios por tela (tabela)
- 4 layouts base (tabela)
- Componentes críticos de domínio (tabela com nome, telas, tokens, faixas)
- Padrão DataTable

### 3 em diante. Uma seção por módulo funcional
Derive os módulos e seus respectivos IDs de tela (T-XXX) diretamente do doc 06 (Mapa de Telas). Para cada módulo identificado, crie uma seção numerada sequencialmente (### 3. [Módulo A], ### 4. [Módulo B], etc.) contendo o contrato de UI de cada tela pertencente ao módulo.

Exemplo de estrutura (adapte ao produto):
- ### 3. Autenticação (T-XXX a T-XXX) — Login, 2FA, Recuperação de Senha, Onboarding.
- ### 4. Dashboard (T-XXX a T-XXX) — Visão Geral, Alertas, Busca Global.
- ### N. [Nome do Módulo] (T-XXX a T-XXX) — Telas do módulo.

Continue sequencialmente até cobrir 100% das telas do doc 06.

### N+1. Cross-References
Tabela com documento referenciado e relação.

### N+2. Changelog
Tabela com Versão, Data, Autor e Descrição.

### N+3. Backlog de Pendências
Tabela consolidando `[DECISÃO AUTÔNOMA]`, `[DEFINIÇÃO PENDENTE]` e `[SEÇÃO PENDENTE]` com colunas:
- Item
- Tipo (Decisão Autônoma / Pendente)
- Tela / Módulo
- Impacto (P0, P1, P2)
- Justificativa ou Pergunta objetiva
- Dono
- Status

## 18. Cobertura mínima por seção
- **Convenções:** 4 estados + 4 layouts + componentes de domínio + DataTable.
- **Por tela:** layout + role mínimo + RFs + tabela de componentes + acessibilidade + motion + regras.
- **Acessibilidade por tela:** focus management, ARIA roles, skip links, contraste, motion reduction.
- **Motion por tela:** transição de entrada, transições internas, skeleton areas, feedback timing.
- **StatusBadge:** mapeamento completo de cada enum usado.
- **Cross-references:** todos os docs dependentes.
- **Backlog:** consolidação de todas as pendências.

## 19. Autoauditoria obrigatória antes da resposta
Antes de responder, execute internamente este ciclo completo:

1. Gere o rascunho completo.
2. Audite contra a estrutura obrigatória de §10 para cada tela.
3. Verifique que toda tela tem os 4 estados obrigatórios.
4. Verifique que nenhum spinner aparece — apenas `<Skeleton>`.
5. Verifique que todo token referencia o namespace definido no doc 03 (Brand Theme Guide).
6. Verifique que todo componente tem variante e config.
7. Verifique que todo StatusBadge tem mapeamento completo do enum.
8. Verifique que toda tela tem role mínimo RBAC.
9. Verifique que toda tela tem seção de acessibilidade (focus, ARIA, skip links, contraste, motion).
10. Verifique que toda tela tem motion/transições documentados (ou "nenhuma" explícito).
11. Corrija qualquer ausência, contradição ou lacuna.
12. Só então responda com a versão final.

## 20. Critérios de qualidade obrigatórios
Antes de responder, valide internamente:

**Formato e estrutura**
- há tabela de componentes para cada tela
- StatusBadge mappings em bloco de código
- cabeçalho de tela com layout, role e RFs

**Técnico**
- zero spinners no documento
- todos os tokens no namespace definido pelo doc 03 (Brand Theme Guide)
- todos os componentes com variante explícita
- RBAC declarado por tela

**Cobertura**
- todas as telas listadas no doc 06 (Mapa de Telas) estão documentadas
- todos os enums de status têm mapeamento completo
- componentes de domínio estão identificados e documentados
- cross-references atualizados

## 21. Regra final de resposta
Retorne **somente o documento final** em Markdown.

Não explique seu raciocínio.
Não faça introdução fora do documento.
Não adicione comentários extras.
Não resuma o briefing.
Não diga que vai começar.

Entregue o documento completo, estruturado, consistente, auditável e pronto para uso.
```

---

## 3. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 20/03/2026 | v1.4 | Correção de seções 1.3 e 1.4: invertida seção 1.3 que listava dependências em vez de consumidores; nomes corrigidos para numeração oficial do pipeline. |
| 20/03/2026 | v1.3 | Reestruturação completa: documento convertido de contrato pronto para formato Prompt Generator, seguindo padrão do pipeline (TL;DR → Persona → Prompt). |
| 18/03/2026 | v1.2 | Padronização da abertura do documento para o formato TL;DR → Persona. |
| 17/03/2026 | v1.1 | Corrigido "ModalLayout" para "ErrorLayout" em §1.2. |
| 17/03/2026 | v1.0 | Criação. Contratos de UI por Tela — versão inicial. |
| 22/03/2026 | v1.6 | Adição de Acessibilidade por tela (WCAG 2.1 AA: focus management, ARIA roles, skip links, contraste, motion reduction) e Motion & Transições por tela (transição de entrada, transições internas, skeleton areas, feedback timing). Autoauditoria expandida de 10 para 12 checks. Cobertura mínima atualizada. Renumeração D08→D09 por inserção do D04 Motion Spec. |
| 21/03/2026 | v1.5 | Genericização: removidos IDs de tela hardcoded (T-001 a T-059), namespace `--portal-*` substituído por referência ao doc 03, corrigida fase do cabeçalho de saída (3 → 2). Prompt agora é reutilizável para qualquer produto. |