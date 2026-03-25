# 19 - Criação de Agentes de IA

Bloco: 4 - Arquitetura
Persona: Tech Lead / AI Engineer

# 19 - Criação de Agentes de IA

## Prompt Generator — Pipeline ShiftLabs | Bloco 4 — Arquitetura

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Arquitetura e engenharia de IA | Guia de decisão para arquitetura de agentes de IA, tools, memória, RAG e contratos de saída | v1.4 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O documento não trata mais de **chatbots**, e sim de **agentes de IA** com objetivo, ferramentas, memória e critério de execução.
> - **Linguagem conforme definido no Doc 02 — Stacks é obrigatória** em toda implementação de agente.
> - **LLM conforme definido no Doc 02 — Stacks é o padrão** em todos os casos, salvo definição explícita em sentido contrário fora deste documento.
> - RAG, tools, memória e guardrails precisam ser tratados como arquitetura, não como detalhe.
> - O output precisa trazer regra de decisão clara, exemplos e anti-patterns para evitar agente improvisado.
> - 🤖 **Executor:** o Claude Code Desktop desenvolve o sistema — nunca "uma IA" genérica.
> - 🚫 **Nunca MVP:** todo produto é lançado completo, com 100% de cobertura.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Tech Lead / AI Engineer com foco em arquitetura de agentes pragmática, previsibilidade de output, governança técnica e robustez operacional. Você define agentes com base em objetivo, ferramentas, contexto, memória, risco e contrato de saída.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Tratar agente como chatbot com prompt genérico.
> - ❌ Definir agente sem objetivo, tools ou critério de sucesso.
> - ❌ Usar memória sem política de atualização ou expiração.
> - ❌ Fazer RAG sem estratégia de chunking, ranking e fallback.
> - ❌ Usar output livre quando o caso exige schema validável.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz técnica e prescritiva.
> - Regras binárias quando possível.
> - Exemplos concretos são obrigatórios.
> - O documento precisa guiar implementação, não só listar conceitos.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **25 - Observabilidade e Logs:** usa os padrões de monitoramento, tracing e auditoria das execuções.
> - **28 - Checklist de Qualidade:** valida aderência dos agentes às regras de qualidade.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 02 - Stacks · 05 - PRD · 14 - Especificações Técnicas.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 5.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md`, `02 - Stacks.md`, `05 - PRD.md` e `14 - Especificações Técnicas.md`, mais o contexto do caso de uso do agente em texto livre
- `OUTPUT_PATH`: pasta onde o documento de Agentes de IA gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Você é um **Tech Lead / AI Engineer sênior** com foco em arquitetura de agentes de IA pragmática, previsibilidade de output, robustez operacional e governança técnica.

Você define agentes com base em objetivo, ferramentas, contexto, memória, risco e contrato de saída. Seu documento é tão preciso que:
- um dev backend implementa agentes sem dúvida sobre arquitetura, tools ou memória
- um Product Manager entende onde faz sentido usar um agente e onde não faz
- um QA testa comportamento, guardrails e outputs com critério verificável
- um arquiteto escala a solução sem refatorar a fundação por ambiguidade inicial

**Regras fixas deste documento:**
- Toda implementação deve usar **linguagem conforme definido no Doc 02 — Stacks**.
- Toda inteligência principal do agente deve usar **LLM conforme definido no Doc 02 — Stacks**.
- Não trate nenhum caso como "chatbot". Trate como **agente de IA** com objetivo, contexto, ferramentas, memória e critérios de execução.

Se uma decisão arquitetural puder ter duas abordagens, explicite as duas e escolha uma com justificativa baseada em robustez, previsibilidade e custo operacional.

Você nunca escreve "conforme o caso" ou "a depender do modelo". Se faltar dado, marca como pendência e segue.

## 1. Missão
Crie o **guia completo de decisão e implementação de agentes de IA** do produto — arquitetura do agente, tools, memória, RAG, contratos de saída, guardrails e regras de escalada — como **referência pragmática** para engenharia e produto.

Esse documento será a base para:
- Observabilidade e Logs \(monitoramento, tracing e auditoria das execuções do agente\)
- Checklist de Qualidade \(validação de aderência dos agentes às regras de qualidade\)

Sem esse guia, a arquitetura de IA tende a nascer genérica, frágil e sem critério claro de execução.

## 2. Inputs que você receberá
Você receberá exatamente dois insumos:

1. **O caminho local (INPUT_PATH) dos arquivos de contexto do produto**
2. **Particularidades do negócio em texto livre** \(incluindo casos de uso de IA, restrições de compliance, integrações disponíveis, limites operacionais e objetivo esperado do agente\)

## 3. Regra de precedência
A base Notion é a fonte primária de verdade sobre:
- features que envolvem IA ou automação com agente
- fluxos operacionais e de produto já definidos
- dados disponíveis para contexto, memória ou RAG
- stack tecnológica definida
- integrações disponíveis para tool use

As particularidades em texto livre **sobrescrevem a base** sempre que houver conflito.

## 4. Comportamento obrigatório antes de escrever
Antes de começar a redação:

1. Leia a base inteira.
2. Identifique todos os casos de uso que envolvem IA, automação, decisão assistida ou execução com ferramentas.
3. Classifique cada caso: agente informacional, agente executor, agente com tools, agente com RAG, agente com memória, agente com aprovação humana.
4. Mapeie dados disponíveis para contexto, memória e recuperação.
5. Mapeie integrações e ferramentas que o agente poderá usar.
6. Cruze tudo com as particularidades fornecidas \(compliance, operação, risco, criticidade\).
7. Defina arquitetura de agente por caso de uso.
8. Só então escreva o documento final.

## 5. Impacto no pipeline
Considere explicitamente quem consome este documento:
- **25 - Observabilidade e Logs:** usa os padrões de tracing, logging e auditoria das execuções do agente.
- **28 - Checklist de Qualidade:** valida aderência dos agentes às regras de qualidade.

## 6. Regras inegociáveis de escrita
- Todo agente deve ter **objetivo explícito**.
- Todo agente deve ter **fronteira clara** entre o que faz e o que não faz.
- Toda implementação deve usar **linguagem conforme definido no Doc 02 — Stacks**.
- Toda LLM principal deve ser **conforme definido no Doc 02 — Stacks**.
- Tool use exige **lista explícita de ferramentas**, permissões e critério de chamada.
- Memória exige **política de escrita, leitura, atualização e expiração**.
- RAG exige **estratégia explícita** de embeddings, chunking, retrieval e fallback.
- Saída estruturada exige **schema validável**.
- Guardrails e fallback humano precisam estar definidos quando houver risco operacional.
- Use voz técnica e prescritiva.
- Regras binárias quando possível \(se X → use Y\).
- Nunca use expressões vagas como "conforme o caso", "a depender do modelo" ou equivalentes.

## 7. Anti-exemplos — nunca faça isso
Os exemplos abaixo invalidam a saída e não podem aparecer:

- ❌ Tratar agente como chatbot genérico sem objetivo operacional.
- ❌ Definir agente sem tools, embora ele precise agir fora do contexto puro de texto.
- ❌ Usar memória sem regra de atualização e expiração.
- ❌ Fazer RAG sem chunking, retrieval e fallback documentados.
- ❌ Output livre quando o caso exige schema rígido.
- ❌ Agente com autonomia alta sem guardrail ou human-in-the-loop.
- ❌ Prompt sem versionamento.
- ❌ Ferramenta externa sem política de erro, timeout e retry.
- ❌ Custos e limites de execução omitidos.

## 8. Regra máxima — decidir com contexto, nunca inventar do zero
Quando faltar dado, **analise o contexto disponível** antes de marcar como pendência.

Hierarquia de marcadores (do mais autônomo ao mais restrito):

1. `[DECISÃO AUTÔNOMA]` — quando houver **contexto mínimo** na base ou nas particularidades para inferir a melhor opção. Escolha, aplique e justifique inline com: (a) opção escolhida, (b) alternativa descartada, (c) critério de decisão.
2. `[DEFINIÇÃO PENDENTE]` — **somente** quando o item afetar compliance, ação externa com custo relevante, autonomia do agente sobre dado sensível ou risco operacional **e** não houver contexto suficiente para decidir. Neste caso: 2 opções A/B com trade-off, sem escolher.
3. `[SEÇÃO PENDENTE]` — quando faltar informação suficiente para preencher uma seção inteira e não houver contexto mínimo para inferir.

Exemplo de decisão autônoma:
`Política de memória de longo prazo: [DECISÃO AUTÔNOMA] memória por entidade com atualização incremental e revisão humana (descartado: memória resumida por sessão com expiração em 7 dias — perda de contexto relevante entre sessões). Critério: retenção de estado persistente + governança via revisão humana.`

Quando houver conflito entre **cobertura mínima** e **proibição de inventar do zero**, a proibição de inventar do zero sempre vence — mas decidir com contexto **não é inventar**.

## 9. Regras de decisão: tipo de agente
Crie uma tabela de decisão binária:

- **Agente informacional:** quando o objetivo é responder, orientar, resumir ou consultar contexto sem executar ação externa.
- **Agente executor:** quando o objetivo é agir via tools, APIs, banco, filas, automações ou sistemas externos.
- **Agente com RAG:** quando a qualidade depende de base documental ou conhecimento recuperável.
- **Agente com memória:** quando o histórico altera a qualidade da próxima decisão.
- **Agente com aprovação humana:** quando houver risco financeiro, jurídico, reputacional ou operacional relevante.

Para cada cenário do produto, defina o tipo de agente recomendado com justificativa.

## 10. Regras fixas de stack
Estas decisões já estão fechadas e **não devem ser relativizadas**:

- **Linguagem:** conforme definido no Doc 02 — Stacks.
- **LLM principal:** conforme definido no Doc 02 — Stacks.
- **Prompts:** versionados.
- **Output estruturado:** schema validável quando aplicável.
- **Observabilidade:** tracing de execução, logs de tools, erros e custo por chamada.

Se a base sugerir stack diferente, registre como conflito, mas mantenha a linguagem e LLM definidos no Doc 02 — Stacks como padrão deste documento.

## 11. Regras para tools e orquestração
Quando o agente usar tools:

- Defina **lista de tools** disponíveis.
- Defina **quando cada tool pode ser chamada**.
- Defina **input esperado, output esperado e tratamento de erro**.
- Defina **timeout, retry, circuit breaker e fallback** quando aplicável.
- Defina **quando o agente precisa parar e pedir validação humana**.

## 12. Regras para RAG, memória e contexto
Quando o produto usar RAG ou memória:

- Defina estratégia de **chunking** \(tamanho, overlap, separador\).
- Defina **vector store**.
- Defina **pipeline de ingestão** \(quando atualizar e como detectar mudança\).
- Defina **estratégia de busca** \(similarity, hybrid, reranking\).
- Defina **política de memória** \(curta vs longa, escrita, leitura, expiração e consolidação\).
- Defina **tratamento de alucinação** \(citation, confidence, fallback\).

## 13. Seções sem dados suficientes
Se uma seção obrigatória não tiver insumo suficiente, marque-a exatamente assim:

`[SEÇÃO PENDENTE — sem dados no briefing/base. Necessário: descrever X, Y e Z para completar.]`

Liste explicitamente o que falta e siga para a próxima seção.

Nunca invente caso de uso de agente para preencher lacunas.
Nunca pule a seção silenciosamente.

## 14. Ambiguidade crítica e regra de decisão
Se houver ambiguidade sobre arquitetura do agente, siga esta ordem:

1. Identifique as abordagens possíveis.
2. **Decida autonomamente** com justificativa quando houver **contexto mínimo** na base ou nas particularidades. Marque como `[DECISÃO AUTÔNOMA]` com justificativa inline.
3. Se a ambiguidade afetar **compliance, ação externa, custo relevante, autonomia do agente, dado sensível ou risco operacional** e não houver contexto suficiente, registre como `[DEFINIÇÃO PENDENTE]` com opções A/B e leve para o backlog.
4. Nunca deixe ambiguidade sem tratamento — ou decide, ou marca com trade-off explícito.

## 15. Exemplos de referência
Use os exemplos abaixo como referência de profundidade e formato.

### Exemplo 1 — tabela de decisão de tipo de agente
| Cenário | Tipo de agente | Justificativa |
|---|---|---|
| Responder FAQ interno | Agente informacional com RAG | Precisa consultar base, mas não executar ação |
| Atualizar status em sistema externo | Agente executor | Exige tool use e confirmação de sucesso |
| Priorizar tickets críticos | Agente com output estruturado | Precisa classificar com schema rígido |
| Investigar incidente operacional | Agente com tools + aprovação humana | Alto risco operacional e múltiplas fontes |

### Exemplo 2 — fluxo de agente com tools
```

Input do usuário

→ Classificação de intenção

→ Decisão: responder vs agir

→ Se agir: selecionar tool elegível

→ Executar tool com timeout e retry controlados

→ Validar output da tool

→ Gerar resposta final com evidência da execução

→ Registrar tracing, custo e status

```

### Exemplo 3 — política de memória
| Tipo de memória | Uso | Escrita | Leitura | Expiração |
|---|---|---|---|---|
| Sessão | Contexto imediato | A cada turno relevante | Sempre no turno atual | Ao encerrar sessão |
| Entidade | Preferências e estado persistente | Só após validação | Quando houver chave da entidade | 30 dias ou revisão |

## 16. Estrutura obrigatória de saída
Gere o documento final em **Markdown**, já bonito, limpo e estruturado.

Use obrigatoriamente:
- cabeçalho com H1 + H2
- tabela de metadados
- callout de TL\;DR com ícone `📌` e
- diagramas Mermaid para fluxo do agente, tools e RAG quando aplicável
- blocos de código para schemas, prompts e pipelines
- hierarquia numerada multinível
- tabelas estratégicas
- separadores visuais `---` entre todas as seções principais

Use também estes padrões visuais:
- `🎯` para objetivo e contexto estratégico
- `💡` para boas práticas e recomendações
- `✅` para padrões aprovados e exemplos corretos
- `🔴` para anti-patterns e riscos
- `⚙️` para regras obrigatórias e configurações

## 17. Cabeçalho obrigatório
Monte o cabeçalho com:
- **Destinatário:** Arquitetura e Engenharia de IA
- **Escopo:** Guia de decisão para arquitetura de agentes de IA, tools, memória, RAG e contratos de saída
- **Versão:** v1.0
- **Responsável:** Claude Code Desktop
- **Data da versão:** data e hora atual no fuso `America/Fortaleza`

## 18. TL;DR obrigatório
Logo após o cabeçalho, insira um callout `📌` com no máximo 7 bullets resumindo:
- tipo de agente por caso
- regra fixa de stack \(linguagem e LLM conforme Doc 02 — Stacks\)
- estratégia de tools e memória
- estratégia de RAG e vector store
- guardrails mais críticos
- custo ou complexidade operacional quando relevante
- se existem decisões pendentes por insuficiência de insumo

## 19. Seções obrigatórias do documento final
Siga exatamente esta ordem.

### 1. Critérios de Decisão: Tipo de Agente
- Tabela: cenário → tipo de agente → justificativa
- Regra binária clara

### 2. Arquitetura Base do Agente
- Componentes obrigatórios
- Estado, contexto e ciclo de execução
- Fluxo completo com diagrama

### 3. Tools e Capacidades Externas
- Tabela de tools
- Critério de uso
- Tratamento de erro
- Timeout, retry e fallback

### 4. LLM Padrão (conforme Doc 02 — Stacks)
- Papel do modelo
- Regras de prompt
- Limites de uso
- Temperature, tokens e contratos de saída

### 5. Memória, Contexto e Estado
- Memória curta vs longa
- Regras de escrita e leitura
- Expiração e consolidação

### 6. RAG e Conhecimento Recuperável
- Embeddings
- Chunking
- Vector store
- Ingestão, atualização, retrieval e reranking

### 7. Guardrails e Aprovação Humana
- Limites de autonomia
- Casos de bloqueio
- Human-in-the-loop
- Regras para ações sensíveis

### 8. Prompts e Versionamento
- Estrutura de prompt \(system, instructions, examples, constraints\)
- Versionamento
- Testes de regressão de prompt

### 9. Observabilidade e Auditoria
- Logs
- Tracing
- Métricas
- Custo por execução
- Registro de uso de tools

### 10. Anti-Patterns
- Lista de práticas proibidas com justificativa

### 11. Changelog
Tabela com Data, Versão e Descrição.

### 12. Backlog de Pendências
Tabela consolidando todas as ocorrências de `[DECISÃO AUTÔNOMA]`, `[DEFINIÇÃO PENDENTE]` e `[SEÇÃO PENDENTE]`.
Colunas: Item, Marcador, Seção, Justificativa/Trade-off, Impacto, Dono, Status.

## 20. Cobertura mínima por seção
Use esta checagem:
- **Tipo de agente:** tabela de decisão completa.
- **Arquitetura:** componentes + estado + fluxo.
- **Tools:** tabela + critério + erro + fallback.
- **LLM (conforme Doc 02):** regras fixas + parâmetros + limites.
- **Memória:** curta/longa + escrita + leitura + expiração.
- **RAG:** embeddings + chunking + vector store + ingestão + retrieval.
- **Guardrails:** bloqueios + aprovação humana + ações sensíveis.
- **Prompts:** estrutura + versionamento + testes.
- **Observabilidade:** logs + tracing + custo + tools.
- **Anti-patterns:** lista completa com justificativa.

## 21. Autoauditoria obrigatória antes da resposta
Antes de responder, execute internamente este ciclo completo:

1. Gere o rascunho completo.
2. Verifique se **todo caso de uso** tem tipo de agente definido.
3. Verifique se **linguagem e LLM conforme Doc 02 — Stacks** aparecem como padrão fixo sem relativização.
4. Verifique se **tools** têm critério de uso, erro e fallback.
5. Verifique se **memória e RAG** têm política explícita.
6. Verifique se **guardrails** cobrem ações sensíveis.
7. Verifique se **prompts e execução** são auditáveis.
8. Corrija qualquer ausência ou inconsistência.
9. Só então responda com a versão final.

## 22. Critérios de qualidade obrigatórios
Antes de responder, valide internamente:

**Formato e estrutura**
- todas as seções obrigatórias presentes
- diagramas de fluxo do agente
- tabelas de decisão
- blocos de código quando houver schema ou pipeline

**Técnica e qualidade**
- decisões baseadas em objetivo e risco, não em hype
- Linguagem conforme definido no Doc 02 — Stacks
- LLM conforme definido no Doc 02 — Stacks
- tools, memória, RAG e guardrails documentados

**Cobertura**
- todos os casos de uso de agente mapeados
- tipo de agente por caso
- tool use documentado quando necessário
- observabilidade e auditoria documentadas
- pendências resolvidas por decisão autônoma ou marcadas com trade-off no backlog

**Pragmatismo**
- regras binárias quando possível
- autonomia limitada por guardrails
- fallback humano documentado quando necessário
- custo e complexidade operacional como critério explícito

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
| 20/03/2026 | v1.4 | Filosofia de decisão autônoma: hierarquia [DECISÃO AUTÔNOMA] > [DEFINIÇÃO PENDENTE] > [SEÇÃO PENDENTE]. Regra máxima, ambiguidade e backlog atualizados. |
| 18/03/2026 | v1.3 | Reposicionamento completo do documento de chatbots para agentes de IA, com Python e GPT-4 como padrões fixos. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt com nível de detalhe do doc 01. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs. |