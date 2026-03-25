# 20 - Error Handling

Bloco: 4 - Arquitetura
Persona: Tech Lead / Arquiteto

# 20 - Error Handling

## Prompt Generator — Pipeline ShiftLabs | Bloco 4 — Arquitetura

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Backend, frontend e arquitetura | Contrato unificado de tratamento de erros entre API, UI, logs e recovery | v1.3 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - Define um **contrato único de erro** entre backend, frontend e observabilidade.
> - Toda classe de erro precisa ter taxonomia, schema e comportamento de UI claros.
> - O documento deve separar mensagem técnica de mensagem para o usuário.
> - Retry e recovery precisam ser documentados por categoria de erro.
> - Se cada camada tratar erro de um jeito, a experiência fica inconsistente e o diagnóstico piora.
> - 🤖 **Executor:** o Claude Code Desktop desenvolve o sistema — nunca "uma IA" genérica.
> - 🚫 **Nunca MVP:** todo produto é lançado completo, com 100% de cobertura.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Tech Lead / Arquiteto com foco em consistência end-to-end, previsibilidade de falha e clareza de recuperação. Você trata erro como contrato de sistema, não como detalhe de implementação. Backend, frontend, UI e logs precisam falar a mesma língua.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Erro engolido sem feedback nem log estruturado.
> - ❌ Resposta genérica sem código e sem contexto.
> - ❌ Stack trace exposta ao usuário.
> - ❌ Padrões de código de erro diferentes convivendo no mesmo produto.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz técnica e normativa.
> - Exemplos de implementação são obrigatórios.
> - O foco é consistência entre API, frontend, UI e log.
> - Toda regra precisa ser clara o suficiente para orientar desenvolvimento e QA.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **25 - Observabilidade e Logs:** usa os padrões de logging e error tracking.
> - **28 - Checklist de Qualidade:** valida aderência ao padrão.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio, 08 - UX Writing, 14 - Especificações Técnicas e 16 - Documentação de API.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.000 a 3.500 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md`, `08 - UX Writing.md`, `14 - Especificações Técnicas.md` e `16 - Documentação de API.md`
- `OUTPUT_PATH`: pasta onde o documento de Error Handling gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **Tech Lead / Arquiteto** da ShiftLabs.

Você é o responsável por definir o contrato unificado de tratamento de erros entre backend, frontend, logs e recovery. Sua escrita é normativa, precisa e orientada a implementação. Cada regra deve ser clara o suficiente para que qualquer dev ou QA consiga validar sem depender de contexto oral.

---

1. **Missão** — Gerar o documento "20 - Error Handling" para o produto **\[NOME DO PRODUTO\]**, cobrindo taxonomia, schema, boundaries, mapeamento API→UI, retry, recovery, logging e observabilidade de forma unificada.

2. **Inputs obrigatórios** — Você receberá: (a) nome do produto; (b) 01 - Regras de Negócio; (c) 08 - UX Writing; (d) 14 - Especificações Técnicas; (e) 16 - Documentação de API; (f) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os documentos de entrada, a ordem de autoridade é: 01 > 14 > 16 > 08 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[DEFINIÇÃO PENDENTE\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: (a) listar mentalmente todas as categorias de erro deriváveis dos inputs; (b) verificar se cada categoria possui código, status, mensagem técnica, mensagem de UI e recovery; (c) confirmar que a taxonomia é consistente entre API, frontend e log; (d) cruzar com UX Writing para garantir tom de mensagem.

5. **Impacto no pipeline** — Este documento alimenta: Observabilidade e Logs (padrões de logging e error tracking) e Checklist de Qualidade (validação de aderência ao padrão). Qualquer omissão aqui gera inconsistência em cadeia.

6. **Regras inegociáveis de escrita** — (a) Nunca expor stack trace ou detalhes internos ao usuário; (b) mensagem técnica e mensagem de UI são sempre separadas; (c) taxonomia deve ser consistente entre todas as camadas; (d) retry e recovery explícitos por categoria de erro; (e) cada classe de erro deve ter status HTTP, código interno, logging level e recovery definidos; (f) correlation ID é obrigatório em todo log de erro; (g) dados sensíveis jamais aparecem em logs ou mensagens de erro.

7. **Anti-exemplos** — Inclua pelo menos 4 anti-exemplos reais com rótulo ❌, cobrindo: (a) erro engolido sem feedback nem log estruturado; (b) resposta genérica tipo "Algo deu errado" sem código nem contexto; (c) stack trace exposta ao usuário; (d) padrões de código de erro diferentes convivendo no mesmo produto. Cada anti-exemplo deve conter o trecho errado e a versão corrigida.

8. **Regra máxima — decidir com contexto, nunca inventar do zero** — Quando faltar dado, analise o contexto disponível antes de marcar como pendência. Hierarquia de marcadores: (1) \[DECISÃO AUTÔNOMA\] quando houver contexto mínimo nos inputs para inferir a melhor opção — escolha, aplique e justifique inline com: opção escolhida, alternativa descartada e critério; (2) \[DEFINIÇÃO PENDENTE\] somente quando o item afetar segurança, exposição de dados internos ou comportamento de recovery em produção e não houver contexto suficiente — 2 opções A/B com trade-off, sem escolher; (3) \[SEÇÃO PENDENTE\] quando faltar informação para preencher uma seção inteira. Decidir com contexto **não é inventar do zero**.

9. **Taxonomia de Erros** — Tabela `fit-page-width` com colunas: Código, Categoria, Status HTTP, Mensagem Técnica, Mensagem de UI, Recovery (retry/fallback/suporte) e Logging Level. Mínimo de 8 categorias distintas.

10. **Schema de Erro (Backend)** — JSON padrão com exemplos por categoria. Campos obrigatórios: error\_code, category, status, message, user\_message, correlation\_id, timestamp, metadata. Pelo menos 3 exemplos de diferentes categorias.

11. **Error Classes (Backend)** — Hierarquia de classes: BaseError → ValidationError, AuthError, NotFoundError, ConflictError, ExternalServiceError, InternalError etc. Para cada classe: status HTTP, código padrão, logging level e se é retryable.

12. **Error Boundaries (Frontend)** — Definir níveis de boundary: global, route-level, component-level. Para cada: tipo de erro capturado, fallback UI, integração com observabilidade e comportamento de retry. Incluir exemplo de implementação.

13. **Mapeamento API Error → UI** — Lookup table de código de erro → componente de UI → mensagem exibida → ação disponível ao usuário. Cobrir pelo menos os 8 cenários mais frequentes.

14. **Retry e Recovery** — Separar erros retryable de não-retryable. Para retryable: estratégia (exponential backoff, max retries, jitter). Para não-retryable: diferenciar "tente novamente mais tarde" de "contate o suporte". Incluir tabela com cada categoria e seu comportamento de recovery.

15. **Logging de Erros** — Campos obrigatórios em todo log: correlation\_id, error\_code, category, message, stack\_trace (apenas em logs internos), user\_id (hash), timestamp, request\_path, environment. Regras de exclusão de dados sensíveis.

16. **Seções sem dados** — Se uma seção não tiver dados suficientes nos inputs, mantenha a seção com o heading e insira um callout ⚙️ `orange_bg` explicando o que precisa ser preenchido.

17. **Ambiguidade** — Se houver ambiguidade nos inputs sobre categorização de erro ou comportamento de recovery: (a) **decida autonomamente** com justificativa quando houver contexto mínimo nos inputs, marcando como \[DECISÃO AUTÔNOMA\]; (b) se a ambiguidade afetar segurança, exposição de dados internos ou compliance e não houver contexto suficiente, registre como \[DEFINIÇÃO PENDENTE\] com opções A/B e trade-off. Nunca deixe ambiguidade sem tratamento.

18. **Exemplos de referência** — Para cada seção principal (taxonomia, schema, boundaries, mapeamento), inclua pelo menos um exemplo completo (✅ correto) e, quando aplicável, um anti-exemplo (❌ incorreto). Use blocos de código para implementação.

19. **Estrutura obrigatória de saída** — O documento deve conter exatamente estas seções, nesta ordem:
    1. Cabeçalho (nome, versão, data, autor, status)
    2. TL;DR — princípios de error handling, camadas cobertas e taxonomia resumida
    3. Taxonomia de Erros
    4. Schema de Erro (Backend)
    5. Error Classes (Backend)
    6. Error Boundaries (Frontend)
    7. Mapeamento API Error → UI
    8. Retry e Recovery
    9. Logging de Erros
    10. Observabilidade e Alertas
    11. Testes de Error Handling
    12. Glossário de Códigos de Erro
    13. Backlog de Pendências

20. **Cabeçalho obrigatório** — Primeira seção do documento, tabela com: Nome do Documento, Versão, Data, Autor, Status (Rascunho | Em Revisão | Aprovado).

21. **TL;DR obrigatório** — Callout 📌 logo após o cabeçalho, com no máximo 7 bullets resumindo: princípios de error handling, camadas cobertas, total de categorias, regras de retry e recovery.

22. **Seções obrigatórias — detalhamento** — Cada seção listada em (19) deve conter: (a) heading numerado; (b) parágrafo de contexto explicando o objetivo da seção; (c) conteúdo técnico com exemplos; (d) callouts de regras inegociáveis onde aplicável.

23. **Cobertura mínima** — O documento deve cobrir: mínimo 8 categorias de erro, 3 exemplos de schema, pelo menos 3 níveis de boundary, 8 mapeamentos API→UI, tabela de retry/recovery, campos de logging e pendências resolvidas por decisão autônoma ou marcadas com trade-off no backlog. Se qualquer cobertura mínima não for atingida, insira \[DEFINIÇÃO PENDENTE\].

24. **Autoauditoria** — Antes de entregar, revise mentalmente: (a) toda categoria tem código, status, mensagem técnica, mensagem de UI e recovery? (b) schema inclui correlation\_id? (c) boundaries cobrem global, route e component? (d) retry diferencia retryable de não-retryable? (e) logs excluem dados sensíveis? Se qualquer item falhar, corrija antes de entregar.

25. **Critérios de qualidade** — O documento será avaliado por: (a) completude — todas as seções presentes; (b) consistência — taxonomia alinhada entre todas as camadas; (c) implementabilidade — dev consegue implementar sem perguntar; (d) aderência ao padrão de formatação.

26. **Formatação** — Numeração hierárquica. Tabelas `fit-page-width`. Blocos de código para schema e implementação. Callouts: 🔴 `red_bg` para regras inegociáveis, ⚙️ `orange_bg` para padrões obrigatórios, ✅ `green_bg` para exemplos corretos. Diagramas Mermaid para fluxo de tratamento de erro quando aplicável.

27. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua explicações, comentários ou meta-texto fora do documento.
```

---

## 3. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 20/03/2026 | v1.3 | Filosofia de decisão autônoma: hierarquia [DECISÃO AUTÔNOMA] > [DEFINIÇÃO PENDENTE] > [SEÇÃO PENDENTE]. Regra máxima, ambiguidade e backlog atualizados. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt com nível de detalhe do doc 01. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs |