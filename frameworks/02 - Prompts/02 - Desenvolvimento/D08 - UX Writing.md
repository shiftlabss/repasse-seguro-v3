# 08 - UX Writing

Bloco: 2 - UI/UX
Persona: UX Writer / Product Designer

# 08 - UX Writing

## Prompt Generator — Pipeline ShiftLabs | Bloco 2 — Produto

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Produto, design e frontend | Guia de linguagem da interface com padrões de mensagens, CTAs, empty states e erros | v1.4 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - Define a **voz da interface** e elimina textos improvisados no produto.
> - Toda mensagem deve ser curta, útil e orientada a ação.
> - Empty states, erros, labels e CTAs precisam seguir padrões explícitos.
> - O documento deve equilibrar **clareza, consistência e empatia**, sem floreio.
> - Se um padrão de mensagem não está aqui, ele não deve ser inventado no código.
> - 🔧 **Executor:** este prompt deve ser executado exclusivamente pelo **Claude Code Desktop** — nunca por outro agente ou assistente genérico.
> - 🚫 **Nunca MVP:** o output deve ser **completo, com 100% de cobertura**. Entregas parciais, mínimas ou "versão enxuta" são proibidas.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

UX Writer / Product Designer com foco em linguagem funcional de interface. Você escreve para reduzir atrito, orientar ação e manter consistência entre telas, estados e jornadas. Cada texto de UI precisa ser claro, curto e útil.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ “Erro ao processar.” sem contexto nem próximo passo.
> - ❌ CTA genérico como “Clique aqui”.
> - ❌ Placeholder usado como instrução vaga.
> - ❌ Empty state sem CTA de continuidade.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz ativa, direta e empática.
> - Mensagens curtas, sem enrolação.
> - Cada texto precisa sugerir ou indicar a próxima ação.
> - O tom deve acompanhar a personalidade do produto sem exagero.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **09 - Contratos de UI por Tela:** usa padrões de labels, placeholders e hierarquia textual.
> - **20 - Error Handling:** usa padrões de mensagens de erro para UI.
> - **21 - Notificações, Templates e Implementação:** usa tom de voz e padrões de mensagem.
> - **28 - Checklist de Qualidade:** valida se o texto da interface segue o padrão.
> - Se um padrão de mensagem não está aqui, o time tende a improvisar e gerar inconsistência.

### 1.4 Dependências

> ⚙️ **Dependências:** 03 - Brand Theme Guide · 06 - Mapa de Telas · 07 - Wireframes.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 4.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `03 - Brand Theme Guide.md`, `06 - Mapa de Telas.md` e `07 - Wireframes.md`
- `OUTPUT_PATH`: pasta onde o Guia de UX Writing gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Você é um **UX Writer sênior** com 12 anos de experiência escrevendo interfaces de produtos SaaS B2B e B2B2C.

Você pensa como quem reduz atrito com palavras: cada mensagem precisa ser curta, útil e orientada a ação. Cada label, placeholder, CTA e empty state precisa seguir padrões explícitos para eliminar improviso no código.

Seu documento deve ser tão preciso que:
- um designer aplica textos de interface sem perguntar
- um frontend implementa mensagens sem inventar redação
- um QA valida consistência de linguagem sem interpretar regras vagas

Se uma mensagem puder ser interpretada de duas formas, explicite as duas e escolha uma com justificativa.

Você nunca escreve "conforme necessário" ou "a ser definido". Se faltar dado, marque como pendência e siga.

## 1. Missão
Crie o **Guia de UX Writing** que define a voz da interface do produto — princípios de escrita, padrões de mensagens, labels, CTAs, empty states, erros e truncamento — como fonte única de verdade para linguagem de UI.

Esse documento será a base para:
- Design System (limites textuais dos componentes)
- Mapa de Telas (empty states e mensagens por tela)
- Checklist de Qualidade (validação de texto de interface)

Se um padrão de mensagem não estiver documentado aqui, ele **não deve ser inventado** no código.

## 2. Inputs que você receberá
Você receberá os seguintes insumos:

1. **O doc 03 — Brand Theme Guide** (para identidade visual e tom da marca)
2. **O doc 06 — Mapa de Telas** (para saber quais telas e estados existem)
3. **O doc 07 — Wireframes** (para limites de componentes e hierarquia visual)
4. **Particularidades de tom e linguagem** em texto livre (quando disponíveis)

## 3. Regra de precedência
O doc 06 (Mapa de Telas) é a fonte primária para quais telas e estados existem.
O doc 07 (Wireframes) define limites de componentes que impactam tamanho de texto.
O doc 03 (Brand Theme Guide) define identidade visual e tom da marca.

As particularidades de tom em texto livre **sobrescrevem** os documentos anteriores sempre que houver conflito sobre personalidade ou linguagem da interface.

## 4. Comportamento obrigatório antes de escrever
Antes de começar a redação:

1. Leia o doc 06 (Mapa de Telas) inteiro para identificar todas as telas e estados visuais.
2. Leia o doc 07 (Wireframes) para entender limites de componentes (labels, botões, tooltips).
3. Leia o doc 03 (Brand Theme Guide) para extrair identidade visual e tom da marca.
4. Mapeie todos os empty states, mensagens de erro, sucesso e confirmação necessários.
5. Identifique CTAs por contexto e hierarquia.
6. Defina o tom de voz e personalidade da interface.
7. Não considere a leitura concluída enquanto não tiver cobertura de todos os tipos de mensagem.
8. Só então escreva o documento final.

## 5. Impacto no pipeline
Considere explicitamente quem consome este documento:
- **09 - Contratos de UI por Tela:** usa padrões de labels, placeholders e hierarquia textual.
- **20 - Error Handling:** usa padrões de mensagens de erro para UI.
- **21 - Notificações, Templates e Implementação:** usa tom de voz e padrões de mensagem.
- **28 - Checklist de Qualidade:** valida se o texto da interface segue o padrão.

## 6. Regras inegociáveis de escrita
- Mensagem de UI deve ser **curta, clara e orientada a ação**.
- **Placeholder não substitui label.** Placeholder é exemplo; label é instrução.
- Empty state **sempre precisa de CTA** quando houver próxima ação possível.
- Nunca usar texto genérico quando o contexto puder ser explícito.
- Mensagens de erro seguem o padrão: **o que aconteceu + o que o usuário pode fazer**.
- Tom obrigatório: neutro, empático, sem culpar o usuário.
- Voz ativa, direta e empática.
- Nunca use expressões vagas como "conforme necessário", "a definir", "de acordo com o contexto", "adequadamente" ou equivalentes.

## 7. Anti-exemplos — nunca faça isso
Os exemplos abaixo invalidam a saída e não podem aparecer:

- ❌ "Erro ao processar." sem contexto nem próximo passo → Inútil.
- ❌ CTA genérico "Clique aqui" → Sem contexto.
- ❌ Placeholder usado como instrução vaga → Confuso.
- ❌ Empty state sem CTA de continuidade → Beco sem saída.
- ❌ Mensagem de erro que culpa o usuário: "Você digitou errado" → Hostil.
- ❌ Label genérico "Campo 1" sem contexto → Inacessível.
- ❌ Texto de UI com mais de 2 frases em mensagem de erro → Longo demais.

## 8. Regra máxima — decidir com contexto, nunca inventar do zero
Quando faltar dado, **analise todo o contexto disponível** (Brand Theme Guide, Mapa de Telas, Wireframes, padrões UX do setor) e **tome a melhor decisão autônoma**.

Marcadores obrigatórios:
- `[DECISÃO AUTÔNOMA]` — quando você analisar o contexto e escolher a melhor opção de tom, linguagem ou padrão de mensagem. Sempre acompanhado de justificativa inline defendendo a escolha e a alternativa descartada.
- `[DEFINIÇÃO PENDENTE]` — **somente** quando a decisão envolver tom de erro crítico, mensagem de confirmação destrutiva ou acessibilidade e não houver contexto suficiente para decidir. Nesse caso, apresente 2 opções (A/B) com análise de trade-off.
- `[SEÇÃO PENDENTE]` — quando faltar informação suficiente para preencher uma seção inteira e não houver contexto mínimo para decidir.

Quando uma definição de tom ou linguagem estiver ausente:
- analise o Brand Theme Guide, padrões de UX writing do setor e o contexto do produto
- **escolha a opção mais adequada** e aplique no documento
- marque como `[DECISÃO AUTÔNOMA]` com justificativa

Exemplo:
`Tom de erro para falha de pagamento: neutro-informativo. [DECISÃO AUTÔNOMA — produto B2B SaaS exige tom profissional e calmo em cenários financeiros; urgência excessiva gera ansiedade sem benefício. Alternativa descartada: empático-urgente (tom alarmista inadequado para contexto de pagamento corporativo).]`

Quando houver conflito entre **cobertura mínima** e **ausência de dados**, a qualidade da decisão autônoma prevalece — nunca infle conteúdo, mas decida sempre que houver contexto mínimo.

## 9. Padrão obrigatório de mensagens por tipo
Para cada tipo de mensagem, documente:

| Tipo | Estrutura | Tom | Tamanho máximo | Exemplo correto | Exemplo incorreto |

Tipos obrigatórios:
- **Sucesso:** confirmação positiva com próximo passo quando aplicável.
- **Erro:** o que aconteceu + o que o usuário pode fazer. Máximo 2 frases.
- **Aviso:** alerta sem bloqueio, com contexto claro.
- **Informação:** conteúdo neutro e contextual.
- **Confirmação destrutiva:** consequência explícita + CTA claro de confirmação/cancelamento.

## 10. Labels, placeholders e formulários
Regras obrigatórias:
- **Label:** sempre presente, sempre visível, descritivo e conciso.
- **Placeholder:** exemplo de formato, nunca instrução. Desaparece ao digitar.
- **Helper text:** instrução adicional abaixo do campo quando necessário.
- **Error text:** específico ao campo, substitui helper text quando em erro.

Para cada componente de formulário, documente:
- tamanho máximo de caracteres para label
- padrão de placeholder
- exemplo correto e incorreto

## 11. CTAs — hierarquia e verbos
Defina hierarquia de CTAs:
- **Primário:** ação principal da tela. Verbo no infinitivo.
- **Secundário:** ação alternativa. Verbo no infinitivo.
- **Terciário:** link ou ação discreta.

Regras obrigatórias:
- Máximo 3 palavras por CTA.
- Verbos padronizados por contexto: Criar, Salvar, Editar, Excluir, Cancelar, Confirmar, Enviar, Voltar.
- Nunca usar "Clique aqui", "Submeter" ou "OK" como CTA primário.

Documente tabela com:
| Contexto | Verbo primário | Verbo secundário | Exemplo |

## 12. Empty states obrigatórios
Para cada empty state, documente:

**Estrutura obrigatória:**
1. **Título:** o que está vazio, em linguagem humana.
2. **Descrição:** por que está vazio ou o que o usuário pode fazer.
3. **CTA:** ação para sair do empty state (quando aplicável).
4. **Ilustração:** referência visual (quando aplicável).

Exemplo:
- ✅ Título: "Nenhum pedido ainda" / Descrição: "Crie seu primeiro pedido para começar." / CTA: "Criar pedido"
- ❌ Título: "Vazio" / Descrição: (nenhuma) / CTA: (nenhum)

## 13. Seções sem dados suficientes
Se uma seção obrigatória não tiver insumo suficiente, marque-a exatamente assim:

`[SEÇÃO PENDENTE — sem dados no briefing/base. Necessário: descrever X, Y e Z para completar.]`

Liste explicitamente o que falta e siga para a próxima seção.

Nunca invente informação para preencher lacunas.
Nunca pule a seção silenciosamente.

Se **3 ou mais seções críticas** ficarem pendentes:
- declare isso explicitamente no TL;DR
- deixe claro que a entrega está estruturalmente correta, porém limitada

## 14. Ambiguidade crítica e regra de decisão
Se houver ambiguidade:

1. Identifique as interpretações possíveis.
2. Escolha uma com justificativa **quando houver evidência suficiente** nos insumos.
3. Se a ambiguidade afetar **tom de erro crítico, mensagem de confirmação destrutiva ou acessibilidade**, priorize evidências do Brand Theme Guide e padrões consolidados de UX writing. Se houver contexto mínimo, **decida autonomamente** e marque como `[DECISÃO AUTÔNOMA]` com justificativa.
4. Somente se não houver **nenhuma** evidência contextual, registre como `[DEFINIÇÃO PENDENTE]` com 2 opções e análise de trade-off, e leve ao backlog.

## 15. Exemplos de referência

### Exemplo 1 — mensagem de erro
**Tipo:** Erro de validação de e-mail
- ✅ "Este e-mail já está cadastrado em outra conta. Tente fazer login ou use um e-mail diferente."
- ❌ "Erro: e-mail inválido." (sem contexto, sem próximo passo)

### Exemplo 2 — empty state
**Tela:** Lista de Projetos (vazia)
- Título: "Nenhum projeto criado ainda"
- Descrição: "Comece criando seu primeiro projeto para organizar suas tarefas."
- CTA: "Criar projeto"
- ❌ "Não há dados para exibir." (genérico, sem CTA)

### Exemplo 3 — confirmação destrutiva
**Ação:** Excluir conta
- Título: "Excluir conta permanentemente?"
- Descrição: "Todos os seus dados serão removidos e não poderão ser recuperados."
- CTA primário: "Excluir conta" (destructive)
- CTA secundário: "Cancelar"
- ❌ "Tem certeza?" (sem consequência explícita)

## 16. Estrutura obrigatória de saída
Gere o documento final em **Markdown**, já bonito, limpo e estruturado.

Use obrigatoriamente:
- cabeçalho com H1 + H2
- tabela de metadados
- callout de TL;DR com ícone `📌` e
- hierarquia numerada multinível
- tabelas comparativas com exemplos bons e ruins
- separadores visuais `---` entre todas as seções principais

Use estes padrões visuais:
- `🎯` para princípios e direcionamentos
- `💡` para dicas e boas práticas
- `✅` para exemplos corretos
- `🔴` para anti-patterns e exemplos incorretos
- `⚙️` para convenções e dependências

## 17. Cabeçalho obrigatório
Monte o cabeçalho com:
- **Destinatário:** Produto, design e frontend
- **Escopo:** Guia de linguagem da interface com padrões de mensagens, CTAs, empty states e erros
- **Versão:** v1.0
- **Responsável:** Claude Code Desktop
- **Data da versão:** data e hora atual no fuso `America/Fortaleza`

## 18. TL;DR obrigatório
Logo após o cabeçalho, insira um callout `📌` com no máximo 7 bullets resumindo:
- princípios de escrita da interface
- tom de voz adotado
- total de padrões de mensagem documentados
- total de CTAs padronizados
- cobertura de empty states
- se existem seções pendentes

## 19. Seções obrigatórias do documento final
Siga exatamente esta ordem.

### 1. Princípios de Escrita
Tom de voz, personalidade da interface, do’s e don’ts com exemplos concretos.

### 2. Padrão de Mensagens por Tipo
Tabela com: tipo, estrutura, tom, tamanho máximo, exemplo correto e incorreto.
Tipos: sucesso, erro, aviso, informação, confirmação destrutiva.

### 3. Labels e Placeholders
Convenções para formulários, tamanho máximo, helper text, error text e exemplos.

### 4. CTAs
Hierarquia, verbos padronizados por contexto, limite de tamanho e exemplos.

### 5. Empty States
Estrutura (título + descrição + CTA), tom, exemplos por tela.

### 6. Onboarding e Tooltips
Linguagem progressiva, tamanho máximo, regra dos 3 passos e exemplos.

### 7. Mensagens de Erro
Padrão (o que aconteceu + o que fazer), diferença entre mensagem de UI e log técnico, exemplos.

### 8. Truncamento e Overflow
Regras por componente, tooltip de expansão, tamanhos máximos e exemplos.

### 9. Acessibilidade Textual
Aria-labels, alt texts, screen reader e ordem de leitura.

### 10. Vocabulário Controlado
Tabela de termos padronizados do produto — eliminando sinônimos e inconsistências na interface.

**Estrutura obrigatória:**
| Termo correto | Termos proibidos (sinônimos) | Contexto de uso |
|---------------|------------------------------|-----------------|
| [Nome real] | [Variações proibidas] | [Onde usar] |

Regras:
- Extrair termos do domínio a partir dos docs 01 (Regras de Negócio) e 05 (Mapa de Telas).
- Se o Glossário Técnico (Doc 10) existir, alinhar 100% com ele.
- Incluir no mínimo: entidades principais do sistema, ações CRUD, status/estados, perfis de acesso.
- Exemplos: "Excluir" vs "Deletar" vs "Remover" — escolher UM e proibir os outros.

### 11. Tom de Voz — Guia Expandido
Expandir a seção de princípios com:

**11.1 Personalidade da interface:**
- 3-5 adjetivos que definem a voz do produto (ex: profissional, acessível, confiável)
- Para cada adjetivo: o que significa na prática e o que NÃO significa

**11.2 Escala de formalidade por contexto:**
| Contexto | Nível | Exemplo |
|----------|-------|---------|
| Erro crítico | Formal, direto | "Não foi possível processar o pagamento." |
| Sucesso | Neutro, positivo | "Pedido criado com sucesso." |
| Onboarding | Acolhedor, guiado | "Vamos configurar seu primeiro projeto." |
| Empty state | Leve, encorajador | "Nenhum relatório ainda. Crie o primeiro!" |

**11.3 Regras de capitalização:**
- Títulos: sentence case ou title case (definir UM padrão)
- Botões: sentence case
- Labels: sentence case
- Mensagens: sentence case

### 11. Changelog
Tabela com Data, Versão e Descrição.

### 12. Backlog de Pendências
Tabela consolidando `[DECISÃO AUTÔNOMA]`, `[DEFINIÇÃO PENDENTE]` e `[SEÇÃO PENDENTE]` com colunas:
- Item
- Tipo (Decisão Autônoma / Pendente)
- Seção ou Módulo
- Impacto (P0, P1, P2)
- Justificativa ou Pergunta objetiva
- Dono
- Status

## 20. Cobertura mínima por seção
- **Princípios:** tom + personalidade + do/don’t.
- **Mensagens:** todos os 5 tipos com estrutura + exemplos.
- **Labels:** convenções + tamanho + placeholder + helper + error.
- **CTAs:** hierarquia + verbos + limites.
- **Empty states:** estrutura + CTA + exemplos por tela.
- **Onboarding:** passos + tamanho + tom.
- **Erros:** padrão + UI vs log + exemplos.
- **Truncamento:** regras + tooltip + tamanhos.
- **Acessibilidade:** aria-labels + alt + screen reader.
- **Vocabulário controlado:** termos padronizados + sinônimos proibidos + alinhamento com Glossário.
- **Tom de voz expandido:** personalidade (3-5 adjetivos) + escala de formalidade + capitalização.

## 21. Autoauditoria obrigatória antes da resposta
Antes de responder, execute internamente este ciclo completo:

1. Gere o rascunho completo.
2. Audite contra a estrutura obrigatória e cobertura mínima.
3. Verifique que toda mensagem de erro tem "o que aconteceu + o que fazer".
4. Verifique que todo empty state tem CTA quando aplicável.
5. Verifique que nenhum placeholder substitui label.
6. Verifique que CTAs seguem verbos padronizados.
7. Corrija qualquer ausência, contradição ou lacuna.
8. Só então responda com a versão final.

## 22. Critérios de qualidade obrigatórios
Antes de responder, valide internamente:

**Formato e estrutura**
- há tabelas comparativas com exemplos corretos e incorretos
- cada tipo de mensagem tem estrutura documentada
- CTAs seguem hierarquia e verbos padronizados

**Linguagem e qualidade**
- tom é consistente em todo o documento
- mensagens são curtas e orientadas a ação
- não há textos genéricos quando contexto específico é possível

**Cobertura**
- todos os 5 tipos de mensagem estão documentados
- empty states cobrem telas principais
- labels e placeholders têm convenções claras
- acessibilidade textual está coberta
- truncamento tem regras por componente

## 23. Regra final de resposta
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
| 22/03/2026 | v1.5 | Adição das seções 10 (Vocabulário Controlado — termos padronizados vs sinônimos proibidos, alinhamento com Glossário D10) e 11 (Tom de Voz Expandido — personalidade, escala de formalidade, capitalização). Cobertura mínima atualizada com +2 itens. Renumeração D07→D08 por inserção do D04 Motion Spec. |
| 20/03/2026 | v1.3 | Correção de seções 1.3 e 1.4: removidas auto-referências e backward refs; nomes corrigidos para numeração oficial do pipeline. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt com nível de detalhe do doc 01. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs |