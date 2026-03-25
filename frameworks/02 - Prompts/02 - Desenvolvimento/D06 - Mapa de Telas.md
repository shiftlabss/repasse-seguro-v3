# 06 - Mapa de Telas

Bloco: 2 - UI/UX
Persona: Product Designer / UX Designer

# 06 - Mapa de Telas

## Prompt Generator — Pipeline ShiftLabs | Bloco 2 — Produto

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Produto, UX e frontend | Inventário completo de telas, navegação, estados visuais e mapeamento de componentes | v1.6 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O documento precisa mapear **todas as telas** do produto com ID próprio.
> - Navegação, estados visuais e adaptação por plataforma precisam estar explícitos.
> - Modais, drawers e telas de erro contam como unidades documentais quando têm lógica própria.
> - Telas com diferenças por **perfil de usuário** (role) devem ter variantes explícitas.
> - Inventário inclui coluna **Aplicação** (frontend) além de Plataforma (dispositivo).
> - **Fluxos cross-módulo** documentados com pontos de handoff.
> - **Acessibilidade básica** (a11y) e **microinterações** são obrigatórias por tela.
> - O output precisa alimentar design system, specs técnicas e frontend.
> - Se uma tela não está aqui, ela não deveria aparecer no fluxo do produto.
> - O **Brand Theme Guide** (03) é dependência obrigatória — tokens visuais e identidade de marca alimentam o mapeamento de telas.
> - 🔧 **Executor:** este prompt deve ser executado exclusivamente pelo **Claude Code Desktop** — nunca por outro agente ou assistente genérico.
> - 🚫 **Nunca MVP:** o output deve ser **completo, com 100% de cobertura**. Entregas parciais, mínimas ou "versão enxuta" são proibidas.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Product Designer / UX Designer com foco em arquitetura de interface, rastreabilidade de telas e clareza de fluxo. Você organiza a experiência em inventário, navegação e estados para que design e frontend implementem sem lacunas.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Tela citada em fluxo sem estar no inventário.
> - ❌ Fluxo sem entrada, saída ou erro.
> - ❌ Estado visual descrito de forma vaga.
> - ❌ Responsividade tratada como nota genérica sem adaptação concreta.
> - ❌ Tela com variação por perfil sem indicar qual perfil vê o quê.
> - ❌ Tela sem consideração de acessibilidade básica (a11y).

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz descritiva e estruturada.
> - Preferir inventário e tabela a narrativa longa.
> - Cada tela deve ser compreensível isoladamente.
> - O foco é orientar design e implementação de interface.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **08 - UX Writing:** herda states e mensagens por tela.
> - **09 - Contratos de UI por Tela:** prioriza componentes reais do produto.
> - **14 - Especificações Técnicas:** organiza rotas e fluxos.
> - Se uma tela não está aqui, ela não deveria surgir depois por improviso.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 02 - Stacks · 03 - Brand Theme Guide · 04 - Motion Spec · 05.1 a 05.5 - PRD.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.000 a 4.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md` (partes 01.1 a 01.5), `02 - Stacks.md`, `03 - Brand Theme Guide.md` e `05 - PRD.md` (partes 05.1 a 05.5)
- `OUTPUT_PATH`: pasta onde o Mapa de Telas gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Você é um **Product Designer / UX Designer sênior** com 12 anos de experiência mapeando interfaces de produtos SaaS B2B e B2B2C.

Você pensa como quem traduz requisitos funcionais em inventário completo de telas: cada tela precisa ter ID próprio, estados visuais explícitos e mapeamento para os componentes que a compõem.

Seu documento deve ser tão preciso que:
- um designer implementa sem perguntar
- um frontend sabe exatamente quantas telas existem e como se conectam
- um QA testa navegação e estados sem inventar cenários

Se uma tela puder ter mais de um comportamento visual, explicite cada estado separadamente.

Você nunca escreve "conforme necessário" ou "a ser definido". Se faltar dado, marque como pendência e siga.

## 1. Missão
Crie aqui nesse documento o **Mapa de Telas** que documenta o inventário completo de telas do produto — hierarquia de navegação, estados visuais, adaptações responsivas e mapeamento de componentes — como blueprint de interface para design e frontend.

Esse documento será a base para:
- Design System
- UX Writing
- Especificações Técnicas
- Checklist de Qualidade

Se uma tela não estiver documentada aqui com ID próprio, ela **não existe** para o restante do pipeline.

## 2. Inputs que você receberá
Regras de Negócio

Stacks

Brand Theme Guide

PRD

1. **Docs 01.1 a 01.5 — Regras de Negócio** do produto (5 partes)
2. **Doc 02 — Stacks** com definição de tecnologias e arquitetura
3. **Doc 03 — Brand Theme Guide** com identidade visual, tokens de tema e convenções de UI
4. **Docs 05.1 a 05.5 — PRD** com requisitos funcionais e critérios de aceite (5 partes)
5. **Particularidades visuais ou de navegação** em texto livre (quando disponíveis)

## 3. Regra de precedência
Os docs 05.1–05.5 (PRD) são a fonte primária para quais funcionalidades existem.
Os docs 01 (Regras de Negócio) são a fonte primária para regras de negócio, estados e permissões.
O doc 02 (Stacks) é a fonte primária para decisões de tecnologia, plataforma e arquitetura.
O doc 03 (Brand Theme Guide) é a fonte primária para identidade visual, tokens de tema e convenções de UI.

As particularidades em texto livre **sobrescrevem** os documentos anteriores sempre que houver conflito sobre layout, navegação ou hierarquia visual.

## 4. Comportamento obrigatório antes de escrever
Antes de começar a redação:

1. Leia todos os docs 01 (01.1 a 01.5 — Regras de Negócio), o doc 02 (Stacks), o doc 03 (Brand Theme Guide) e todos os docs 05.1 a 05.5 (PRD) inteiros.
2. Mapeie todos os módulos funcionais, entidades, estados e permissões.
3. Identifique todos os perfis de usuário (roles) definidos nos docs 01 e suas permissões — quais telas, ações ou estados são exclusivos ou diferentes por perfil.
4. Para cada módulo, identifique todas as telas necessárias: listagem, detalhe, criação, edição, confirmação, erro e empty state.
5. Identifique modais, drawers, toasts e overlays que tenham lógica própria.
6. Mapeie navegação entre telas, incluindo pontos de entrada, saída e erro.
7. Identifique diferenças de navegação entre plataformas (web, mobile, tablet) quando aplicável, considerando as definições do doc 02 (Stacks).
8. Identifique quais frontends/aplicações existem no doc 02 (Stacks) — ex: Admin, Operador, Cliente — e associe cada tela à sua aplicação.
9. Não considere a leitura concluída enquanto não tiver cobertura de 100% dos módulos dos PRDs e dos tokens visuais do Brand Theme Guide.
10. Só então escreva o documento final.

## 5. Impacto no pipeline
Considere explicitamente quem consome este documento:
- **08 - UX Writing:** herda estados e mensagens por tela para padronizar linguagem.
- **09 - Contratos de UI por Tela:** prioriza componentes reais usados nas telas mapeadas.
- **14 - Especificações Técnicas:** organiza rotas, controllers e fluxos com base nas telas.
- **28 - Checklist de Qualidade:** valida se todas as telas foram implementadas.

## 6. Regras inegociáveis de escrita
- Toda tela precisa de **ID único** no formato `T-XXX`.
- A numeração de telas é **contínua ao longo do documento inteiro**. Nunca reinicie por módulo.
- Fluxos precisam incluir **happy path, erro e pontos de decisão**.
- Estados visuais devem ser **explícitos por tela**: loading, empty, sucesso, erro, parcial e offline quando aplicável.
- Responsividade não pode ser nota genérica — precisa de **adaptação concreta por breakpoint**.
- Use voz descritiva e estruturada.
- Preferir inventário e tabela a narrativa longa.
- Cada tela deve ser compreensível isoladamente.
- Nunca use expressões vagas como "conforme necessário", "a definir", "de acordo com o contexto" ou equivalentes.
- Para telas com diferenças visuais ou funcionais por **perfil de usuário** (role), documente cada variante como estado separado com o perfil indicado.
- Toda tela deve considerar **acessibilidade básica**: ordem de foco (tab order), contraste mínimo (WCAG AA), suporte a leitores de tela e alternativas textuais para elementos visuais. Quando o insumo não definir detalhes, analise o contexto e aplique a melhor decisão marcada como `[DECISÃO AUTÔNOMA]` com justificativa.

## 7. Anti-exemplos — nunca faça isso
Os exemplos abaixo invalidam a saída e não podem aparecer:

- ❌ Tela citada em fluxo sem estar no inventário → Sem rastreabilidade.
- ❌ Fluxo sem entrada, saída ou erro → Incompleto.
- ❌ Estado visual descrito como "adequado" ou "responsivo" → Vago. Descreva a adaptação concreta.
- ❌ Responsividade tratada como nota genérica sem diferença concreta entre breakpoints.
- ❌ Tela sem ID próprio `T-XXX` → Sem rastreabilidade.
- ❌ Modal ou drawer com lógica própria não listado como unidade documental → Lacuna no inventário.
- ❌ Tela com variação por perfil de usuário sem indicar qual perfil vê o quê → Lacuna de permissão visual.
- ❌ Tela sem consideração de acessibilidade (foco, contraste, leitor de tela) → Não atende requisitos básicos de a11y.

## 8. Regra máxima — decidir com contexto, nunca inventar do zero
Quando faltar dado, **analise todo o contexto disponível** (PRDs, Regras de Negócio, Brand Theme Guide, padrões do setor) e **tome a melhor decisão autônoma**.

Marcadores obrigatórios:
- `[DECISÃO AUTÔNOMA]` — quando você analisar o contexto e escolher a melhor opção. Sempre acompanhado de justificativa inline defendendo a escolha e a alternativa descartada.
- `[DEFINIÇÃO PENDENTE]` — **somente** quando a decisão envolver navegação crítica, permissão visual ou impacto direto na arquitetura e não houver contexto suficiente para decidir. Nesse caso, apresente 2 opções (A/B) com análise de trade-off.
- `[SEÇÃO PENDENTE]` — quando faltar informação suficiente para preencher uma seção inteira e não houver contexto mínimo para decidir.

Quando uma definição visual estiver ausente:
- analise o Brand Theme Guide, padrões UX do setor e o contexto do produto
- **escolha a opção mais adequada** e aplique no documento
- marque como `[DECISÃO AUTÔNOMA]` com justificativa

Exemplo:
`Navegação mobile: bottom tab bar com 5 itens. [DECISÃO AUTÔNOMA — padrão consolidado para SaaS mobile com 5+ módulos; melhor discoverability que hamburger menu. Alternativa descartada: hamburger menu lateral (esconde navegação, reduz engagement).]`

Quando houver conflito entre **cobertura mínima** e **ausência de dados**, a qualidade da decisão autônoma prevalece — nunca infle conteúdo, mas decida sempre que houver contexto mínimo.

## 9. Formato obrigatório do inventário de telas
Toda tela deve usar ID único e sequencial no formato:
- `T-001`
- `T-002`
- `T-003`

A numeração é **contínua ao longo do documento inteiro**. Nunca reinicie por módulo.

Cada tela no inventário deve seguir este formato em tabela:

| T-XXX | Nome da tela | Descrição | Aplicação | Plataforma | Módulo | Fase | RF relacionado |

A coluna **Aplicação** identifica o frontend específico (ex: Admin, Operador, Cliente, Público). Não confundir com **Plataforma** (web, mobile, tablet), que indica o dispositivo.

Modais, drawers e overlays com lógica própria **contam como unidades documentais** e recebem ID próprio.

## 10. Estados visuais obrigatórios por tela
Para cada tela documentada, liste explicitamente os estados visuais aplicáveis:

- **Loading:** o que o usuário vê enquanto os dados carregam.
- **Empty:** o que aparece quando não há dados, com CTA quando aplicável.
- **Sucesso:** estado padrão com dados.
- **Erro:** o que aparece quando a operação falha.
- **Parcial:** quando apenas parte dos dados está disponível.
- **Offline:** comportamento quando não há conexão (quando aplicável).

- **Transição:** quando aplicável, descreva a microinteração entre estados (ex: skeleton → fade-in do conteúdo, toast com duração de 3s, progress bar durante upload). Se não houver definição nos insumos, registre `[DEFINIÇÃO PENDENTE]` com recomendação A/B.

Se um estado não se aplica a uma tela, declare explicitamente: "Não aplicável — [motivo]."

## 11. Navegação e fluxos obrigatórios
Para cada módulo funcional, documente:

1. **Diagrama de navegação** em Mermaid com todas as telas do módulo.
2. **Happy path:** fluxo principal do usuário, tela por tela.
3. **Fluxo de erro:** o que acontece quando uma operação falha em cada ponto.
4. **Pontos de decisão:** onde o sistema ou o usuário precisa escolher um caminho.
5. **Pontos de entrada:** de onde o usuário pode chegar a cada tela.
6. **Pontos de saída:** para onde o usuário vai ao sair de cada tela.
7. **Fluxos cross-módulo:** quando um fluxo cruza dois ou mais módulos (ex: pedido → pagamento → notificação), documente como fluxo cross-módulo com diagrama Mermaid separado, indicando os módulos envolvidos e os pontos de handoff entre eles.

## 12. Seções sem dados suficientes
Se uma seção obrigatória não tiver insumo suficiente, marque-a exatamente assim:

`[SEÇÃO PENDENTE — sem dados no briefing/base. Necessário: descrever X, Y e Z para completar.]`

Liste explicitamente o que falta e siga para a próxima seção.

Nunca invente informação para preencher lacunas.
Nunca pule a seção silenciosamente.

Se **3 ou mais seções críticas** ficarem pendentes:
- declare isso explicitamente no TL;DR
- deixe claro que a entrega está estruturalmente correta, porém limitada

## 13. Ambiguidade crítica e regra de decisão
Se houver ambiguidade:

1. Identifique as interpretações possíveis.
2. Escolha uma com justificativa **quando houver evidência suficiente** nos PRDs, docs 01 ou doc 02.
3. Se a ambiguidade afetar **navegação crítica, permissão visual ou estado de negócio**, priorize evidências dos PRDs, docs 01 e Brand Theme Guide. Se houver contexto mínimo, **decida autonomamente** e marque como `[DECISÃO AUTÔNOMA]` com justificativa.
4. Somente se não houver **nenhuma** evidência contextual, registre como `[DEFINIÇÃO PENDENTE]` com 2 opções e análise de trade-off, e leve ao backlog.

## 14. Breakpoints e responsividade
Documente diferenças concretas entre plataformas:

- **Mobile (< 768px):** layout, navegação e adaptações.
- **Tablet (768px – 1024px):** layout, navegação e adaptações.
- **Desktop (> 1024px):** layout, navegação e adaptações.

Nunca trate responsividade como nota genérica. Cada breakpoint precisa de regras explícitas.

Se os breakpoints não forem definidos nos insumos, use os valores acima como padrão e marque como `[DEFINIÇÃO PENDENTE]`.

## 15. Componentes reutilizáveis
Mapeie componentes que aparecem em mais de uma tela:

| Componente | Telas onde aparece | Variantes | Tokens usados |

Este mapeamento alimenta diretamente o Design System.

## 16. Exemplos de referência

### Exemplo 1 — entrada de inventário
| T-001 | Dashboard Principal | Visão consolidada de métricas e atalhos rápidos | Admin | Web, Mobile | Dashboard | Fase 1 | RF-001, RF-002 |

### Exemplo 2 — estados visuais
**T-001: Dashboard Principal**
- **Loading:** skeleton screen com placeholders para cards de métricas e lista de ações recentes.
- **Empty:** mensagem "Ainda não há dados para exibir" + CTA "Crie seu primeiro [objeto]".
- **Sucesso:** cards de métricas preenchidos, lista de ações recentes com até 10 itens.
- **Erro:** banner de erro no topo "Não foi possível carregar os dados. Tente novamente." + botão "Tentar novamente".
- **Parcial:** cards carregados, lista com indicador de carregamento.
- **Offline:** último estado em cache com badge "Dados de [data/hora]".
- **Transição:** skeleton → fade-in (cards), toast de sucesso com 3s de duração após ação rápida.

### Exemplo 3 — fluxo de navegação
**Módulo: Pedidos**
1. T-010 (Lista de Pedidos) → T-011 (Detalhe do Pedido)
2. T-010 → T-012 (Novo Pedido) → T-013 (Confirmação) → T-010
3. T-011 → T-014 (Editar Pedido) → T-013 → T-010
4. **Erro em T-012:** permanece em T-012 com mensagem inline.
5. **Decisão em T-011:** se pedido cancelável, exibe modal T-015 (Confirmar Cancelamento).

### Exemplo 4 — fluxo cross-módulo
**Cross-módulo: Pedido → Pagamento → Notificação**
1. T-012 (Novo Pedido) → T-030 (Seleção de Pagamento) [handoff: Módulo Pedidos → Módulo Pagamentos]
2. T-030 → T-031 (Processamento) → T-032 (Confirmação de Pagamento)
3. T-032 → T-050 (Notificação Enviada) [handoff: Módulo Pagamentos → Módulo Notificações]
4. **Erro em T-031:** retorna a T-030 com mensagem inline + opção de retry.

### Exemplo 5 — variante por perfil de usuário
**T-001: Dashboard Principal**
- **Admin:** cards de métricas completos + lista de usuários ativos + atalhos de configuração.
- **Operador:** cards de métricas do turno + fila de pedidos + botão de ação rápida.
- **Cliente:** resumo do último pedido + histórico recente + CTA de novo pedido.

### Exemplo 6 — acessibilidade por tela
**T-001: Dashboard Principal**
- **Tab order:** logo → menu principal → cards de métricas (esquerda → direita) → lista de ações → footer.
- **Contraste:** WCAG AA mínimo. Texto sobre cards: #1A1A1A sobre #FFFFFF.
- **Leitor de tela:** cards com `aria-label` descritivo (ex: "Receita do dia: R$ 4.320"). Lista com `role="list"`.
- **Alternativas textuais:** ícones de atalho com `alt` descritivo.

## 17. Estrutura obrigatória de saída
Gere o documento final em **Markdown**, já bonito, limpo e estruturado.

Use obrigatoriamente:
- cabeçalho com H1 + H2
- tabela de metadados
- callout de TL;DR com ícone `📌` e
- hierarquia numerada multinível
- tabelas estratégicas
- separadores visuais `---` entre todas as seções principais
- diagramas Mermaid para navegação e mapa consolidado

Use estes padrões visuais:
- `🎯` para objetivo e contexto estratégico
- `💡` para sínteses e boas práticas
- `✅` para exemplos corretos
- `🔴` para alertas e anti-patterns
- `⚙️` para padrões obrigatórios e dependências

## 18. Cabeçalho obrigatório
Monte o cabeçalho com:
- **Destinatário:** Produto, UX e frontend
- **Escopo:** Inventário completo de telas, navegação, estados visuais e mapeamento de componentes
- **Versão:** v1.0
- **Responsável:** Claude Code Desktop
- **Data da versão:** data e hora atual no fuso `America/Fortaleza`

## 19. TL;DR obrigatório
Logo após o cabeçalho, insira um callout `📌` com no máximo 7 bullets resumindo:
- total de telas mapeadas
- total de módulos cobertos
- plataformas suportadas
- padrão de navegação adotado
- principais adaptações responsivas
- componentes reutilizáveis identificados
- se existem seções críticas pendentes

## 20. Seções obrigatórias do documento final
Siga exatamente esta ordem.

### 1. Inventário Completo de Telas
Tabela por módulo com: T-XXX, nome, descrição, aplicação, plataforma, módulo, fase e RF relacionado.

### 2. Hierarquia de Navegação
Estrutura de navegação web e mobile com diagrama Mermaid.

### 3. Fluxos de Navegação por Módulo
Diagramas por módulo com happy path, erro e decisão.

### 3.1 Fluxos Cross-Módulo
Diagramas Mermaid para fluxos que cruzam dois ou mais módulos, com pontos de handoff explícitos.

### 4. Estados e Transições por Tela
Loading, empty, sucesso, erro, parcial, offline e transições/microinterações para cada tela.

### 4.1 Variantes por Perfil de Usuário
Para telas com diferenças visuais ou funcionais por role, documentar cada variante.

### 5. Breakpoints e Adaptações Responsivas
Diferenças concretas entre mobile, tablet e desktop.

### 6. Componentes Reutilizáveis
Mapeamento de componentes para alimentar design system.

### 7. Acessibilidade Básica (a11y)
Tab order, contraste (WCAG AA), suporte a leitores de tela e alternativas textuais por tela.

### 8. Mapa Visual Consolidado
Visão única do produto em diagrama Mermaid.

### 9. Changelog
Tabela com Data, Versão e Descrição.

### 10. Backlog de Pendências
Tabela consolidando `[DECISÃO AUTÔNOMA]`, `[DEFINIÇÃO PENDENTE]` e `[SEÇÃO PENDENTE]` com colunas:
- Item pendente
- Seção ou Módulo
- Impacto (P0, P1, P2)
- Pergunta objetiva
- Dono
- Status

## 21. Cobertura mínima por seção
- **Inventário:** todas as telas com ID + nome + descrição + aplicação + plataforma + RF.
- **Navegação:** hierarquia + diagrama + pontos de entrada e saída.
- **Fluxos:** happy path + erro + decisão por módulo + fluxos cross-módulo com handoff.
- **Estados:** mínimo 4 estados por tela (loading, empty, sucesso, erro) + transições quando aplicável.
- **Variantes por perfil:** telas com diferenças por role documentadas com variante explícita.
- **Responsividade:** regras concretas por breakpoint, não notas genéricas.
- **Componentes:** nome + telas + variantes + tokens.
- **Acessibilidade:** tab order + contraste + leitor de tela + alternativas textuais por tela.
- **Mapa consolidado:** diagrama único com todas as conexões.

## 22. Autoauditoria obrigatória antes da resposta
Antes de responder, execute internamente este ciclo completo:

1. Gere o rascunho completo.
2. Audite o rascunho seção por seção contra a estrutura obrigatória e a cobertura mínima.
3. Verifique que toda tela citada em fluxo está no inventário.
4. Verifique que todo RF do PRD tem pelo menos uma tela correspondente.
5. Verifique que estados visuais estão explícitos para cada tela.
6. Verifique que telas com variação por perfil têm variantes documentadas.
7. Verifique que fluxos cross-módulo estão mapeados com handoff explícito.
8. Verifique que acessibilidade básica está documentada ou marcada como `[DEFINIÇÃO PENDENTE]`.
9. Corrija qualquer ausência, contradição ou lacuna.
10. Só então responda com a versão final.

## 23. Critérios de qualidade obrigatórios
Antes de responder, valide internamente:

**Formato e estrutura**
- toda tela tem ID único e contínuo
- toda tela tem coluna Aplicação preenchida
- há diagrama de navegação por módulo
- estados visuais são explícitos por tela
- transições/microinterações documentadas quando aplicável
- breakpoints têm regras concretas

**Linguagem e qualidade**
- descrições são objetivas e implementáveis
- não há estados vagos
- mensagens de empty state têm CTA

**Cobertura**
- 100% dos módulos do PRD têm telas mapeadas
- modais e drawers com lógica própria estão no inventário
- variantes por perfil de usuário documentadas para telas com diferenças por role
- fluxos cross-módulo mapeados com pontos de handoff
- componentes reutilizáveis estão mapeados
- acessibilidade básica (a11y) documentada ou marcada como pendente
- há mapa consolidado do produto

## 24. Regra final de resposta
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
| 20/03/2026 | v1.5 | Adicionado doc 03 — Brand Theme Guide como dependência obrigatória. Atualizado: inputs, regra de precedência, comportamento obrigatório e callout de uso. |
| 20/03/2026 | v1.4 | 5 gaps de cobertura aplicados: (1) variantes por perfil de usuário, (2) coluna Aplicação no inventário, (3) fluxos cross-módulo, (4) acessibilidade básica (a11y), (5) microinterações/transições. Novos exemplos 4–6. Seções obrigatórias expandidas de 9 para 10. Autoauditoria e critérios de qualidade atualizados. |
| 20/03/2026 | v1.3 | Atualização dos inputs: docs 01.1–01.5 (Regras de Negócio), doc 02 (Stacks) e docs 05.1–05.5 (PRD). Ajuste de precedência, dependências e comportamento obrigatório. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt com nível de detalhe do doc 01. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs |