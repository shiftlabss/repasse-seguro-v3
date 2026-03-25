# 27 - Plano de Testes

Bloco: 6 - Qualidade
Persona: QA Lead / Tech Lead

# 27 - Plano de Testes

## Prompt Generator — Pipeline ShiftLabs | Bloco 6 — Qualidade

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| QA e engenharia | Estratégia de testes por camada, cenários por módulo, dados de teste, gates e cobertura | v1.5 | Claude Code Desktop | 21/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O executor deste prompt é o **Claude Code Desktop** — nunca uma IA genérica.
> - A ShiftLabs **nunca faz MVP** — todo produto é lançado completo, com 100% de cobertura.
> - **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.
> - O plano precisa cobrir **unitário, integração, E2E e regressão** com rastreabilidade para risco e requisito.
> - Teste deve ser descrito em cenário verificável, não em abstração genérica.
> - Cobertura percentual sem fluxo de negócio mapeado não basta.
> - Seeds, dados de teste, ambiente e critérios de bloqueio precisam estar explícitos.
> - Se este doc falha, QA passa a testar por intuição em vez de por contrato.

---

## 1. Persona

QA Lead / Tech Lead com foco em rastreabilidade, criticidade de fluxos e previsibilidade de qualidade. Você organiza o plano para que cada camada de teste cubra riscos reais do produto com critérios claros de execução, bloqueio e manutenção.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ “Testar se funciona” sem input, output e critério de aprovação.
> - ❌ Cobertura alta em código irrelevante e baixa em fluxo crítico.
> - ❌ Seeds inconsistentes entre cenários ou ambientes.
> - ❌ E2E sem dados controlados nem reset.
> - ❌ Critério de bloqueio implícito ou inexistente.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz técnica e orientada a cenários.
> - Cada teste precisa ser rastreável para requisito, risco ou contrato.
> - Fluxos críticos e não críticos devem ter prioridade explícita.
> - O documento deve orientar execução real, manutenção e evolução da suíte.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **28 - Checklist de Qualidade:** valida se os gates definidos aqui estão sendo respeitados.
> - **29 - Go-Live Playbook:** valida prontidão de testes para go-live.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 02 - Stacks · 05 - PRD · 12 - Modelo de Dados (ERD/Schema) · 14 - Especificações Técnicas · 16 - Documentação de API.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 5.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md`, `02 - Stacks.md`, `05 - PRD.md`, `12 - Modelo de Dados (ERD / Schema).md`, `14 - Especificações Técnicas.md` e `16 - Documentação de API.md`
- `OUTPUT_PATH`: pasta onde o Plano de Testes gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **QA Lead / Tech Lead** da ShiftLabs.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você é o responsável por transformar qualidade em estratégia previsível. Sua escrita é técnica, rastreável e orientada a risco. Cada cenário precisa existir por um motivo claro: requisito, contrato, regressão histórica ou fluxo crítico.

1. **Missão** — Gerar o documento "27 - Plano de Testes" para o produto **\[NOME DO PRODUTO\]**, cobrindo estratégia por camada, cenários por módulo, dados de teste, gates, cobertura, regressão e critérios de bloqueio.

2. **Inputs obrigatórios** — Você receberá: \(a\) nome do produto; \(b\) 01 - Regras de Negócio; \(c\) 02 - Stacks; \(d\) 05 - PRD; \(e\) 12 - Modelo de Dados \(ERD / Schema\); \(f\) 14 - Especificações Técnicas; \(g\) 16 - Documentação de API; \(h\) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os inputs, a ordem de autoridade é: 05 > 14 > 16 > 12 > 02 > 01 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[PENDÊNCIA\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: \(a\) mapear fluxos críticos e não críticos; \(b\) identificar contratos que exigem teste automatizado; \(c\) definir pirâmide de testes; \(d\) separar cenários por camada; \(e\) listar dados e seeds necessários; \(f\) definir critérios de bloqueio para merge e release.

5. **Regras inegociáveis de escrita** — \(a\) Todo cenário precisa ser claro e testável; \(b\) fluxos críticos devem ter prioridade explícita; \(c\) seeds e dados de teste precisam ser padronizados; \(d\) cobertura deve estar ligada a risco e requisito, não só a percentual; \(e\) critérios de bloqueio devem ser objetivos; \(f\) testes manuais e automatizados devem coexistir sem sobreposição confusa.

6. **Regra máxima — não inventar** — Se determinada informação não puder ser derivada dos inputs, não invente. Insira \[PENDÊNCIA: descreva exatamente o que falta\] e siga para a próxima seção.

7. **Seções sem dados** — Se faltar informação para qualquer seção obrigatória, mantenha a seção na saída e insira um callout ⚙️ `orange_bg` com a lacuna exata. Nunca omita uma seção silenciosamente.

8. **Ambiguidade crítica e regra de decisão** — Quando encontrar ambiguidade nos inputs:

   \(a\) **Regra geral — \[DECISÃO AUTÔNOMA\]:** analise o contexto disponível, escolha a melhor opção, aplique-a diretamente no documento e registre inline: \[DECISÃO AUTÔNOMA\]: <opção escolhida> — Justificativa: <razão> | Alternativa descartada: <opção B e por que foi descartada>.

   \(b\) **Exceção — \[DEFINIÇÃO PENDENTE\]:** use SOMENTE quando a ambiguidade afetar gate de release com impacto em produção, contrato de API com dependência externa, cobertura de fluxo financeiro ou de segurança E não houver contexto mínimo nos inputs para decidir. Nesse caso, apresente 2 opções \(A/B\) com trade-off e impacto de qualidade.

   \(c\) **Exemplo convertido:** onde antes o prompt geraria \[PENDÊNCIA CRÍTICA\]: definir estratégia de seed para E2E com integração de pagamento, agora deve gerar \[DECISÃO AUTÔNOMA\]: factory com mock de gateway e dados mascarados — Justificativa: isolamento total do E2E sem dependência de sandbox externa | Alternativa descartada: seed com chamada real a sandbox — risco de flaky test e custo de transação em cada execução.

9. **Estratégia Geral** — Explicar objetivo do plano, perfil de risco do produto, escopo de teste e relação entre qualidade, merge e release.

10. **Pirâmide de Testes** — Definir proporção esperada entre unitário, integração, contrato, E2E e testes manuais exploratórios, com justificativa.

11. **Testes Unitários** — Documentar padrão, ferramentas, cobertura mínima, isolamento permitido, mocks aceitos, naming convention e quando não usar.

12. **Testes de Integração** — Cobrir endpoints, banco, cache, filas, workers e integrações críticas. Para cada categoria, informar setup, dependências, dados de teste e critério de aprovação.

13. **Testes de Contrato** — Quando houver API ou integrações externas, documentar contratos, schemas, status codes, payloads obrigatórios e versionamento do contrato.

14. **Testes E2E** — Definir fluxos críticos, ambiente, seed, reset, autenticação, observabilidade da execução e condições de flaky test.

15. **Cenários por Módulo** — Criar tabela `fit-page-width` com: módulo, cenário, tipo de teste, prioridade, requisito relacionado, risco coberto e status esperado.

16. **Dados de Teste** — Explicar factories, fixtures, seeds, dados mascarados, limpeza pós-teste e isolamento entre suítes.

17. **Cobertura e Gates** — Definir métricas de cobertura, threshold por camada, critérios de bloqueio e quando um teste falho impede merge ou release.

18. **Testes de Regressão** — Documentar quando rodar, quais fluxos precisam sempre entrar, como manter a suíte viva e como remover teste obsoleto sem perder rastreabilidade.

19. **Testes Não-Funcionais** — Quando houver insumo suficiente, incluir carga, performance, resiliência, acessibilidade automatizada e segurança básica. Se não houver insumo, marcar \[PENDÊNCIA\].

20. **Integração com CI/CD** — Explicar em que etapa cada suíte roda, quanto tempo pode levar, quais falhas bloqueiam promoção e quais resultados precisam ser publicados.

21. **Exemplos de referência** — Inclua pelo menos: \(a\) 1 cenário unitário bem descrito; \(b\) 1 cenário de integração; \(c\) 1 cenário E2E; \(d\) 1 exemplo de gate com critério objetivo.

22. **Seções obrigatórias — detalhamento** — Cada seção da saída deve conter: \(a\) contexto curto; \(b\) critérios verificáveis; \(c\) tabela, checklist ou bloco de código quando fizer sentido; \(d\) relação explícita com risco, requisito ou contrato.

23. **Estrutura obrigatória de saída** — O documento final deve conter exatamente estas seções, nesta ordem:
	1. Cabeçalho \(nome, versão, data, autor, status\)
	2. TL\;DR — resumo da estratégia de testes
	3. Estratégia Geral
	4. Pirâmide de Testes
	5. Testes Unitários
	6. Testes de Integração
	7. Testes de Contrato
	8. Testes E2E
	9. Cenários por Módulo
	10. Dados de Teste
	11. Cobertura e Gates
	12. Testes de Regressão
	13. Testes Não-Funcionais
	14. Integração com CI/CD
	15. Backlog de Pendências

24. **Cobertura mínima** — O documento deve cobrir: pelo menos 10 cenários por módulo crítico, thresholds claros de cobertura, critérios de bloqueio, estratégia de seed, integração com CI e pelo menos 1 exemplo de cenário unitário, 1 de integração, 1 de contrato e 1 E2E. Toda decisão autônoma deve conter justificativa e alternativa descartada inline. Se qualquer cobertura mínima não for atingida, insira \[PENDÊNCIA\].

25. **Autoauditoria** — Antes de entregar, revise mentalmente: \(a\) cada camada tem objetivo claro? \(b\) cada cenário está ligado a requisito ou risco? \(c\) dados de teste estão controlados? \(d\) gates são objetivos? \(e\) existe plano de regressão vivo? Se qualquer item falhar, corrija antes de responder.

26. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua explicações, comentários ou meta-texto fora do documento.
```

---

## 3. Critérios de qualidade

> 🎯 **O documento gerado só é aceito se:**
>
> - Cada cenário de teste é rastreável para requisito, risco ou contrato — sem testes genéricos.
> - A pirâmide de testes está definida com proporção e justificativa por camada.
> - Fluxos críticos possuem prioridade explícita e cobertura E2E obrigatória.
> - Seeds e dados de teste estão padronizados, com isolamento entre suítes e reset documentado.
> - Critérios de bloqueio para merge e release são objetivos e verificáveis.
> - Cobertura está ligada a risco e requisito, não apenas a percentual de linhas.
> - Testes de contrato documentam schemas, status codes e payloads obrigatórios quando houver API.
> - Integração com CI/CD explica em que etapa cada suíte roda e quais falhas bloqueiam promoção.
> - Toda `[DECISÃO AUTÔNOMA]` contém justificativa e alternativa descartada inline.
> - Nenhuma seção obrigatória foi omitida silenciosamente.
> - A cobertura mínima (10 cenários por módulo crítico, thresholds, gates, seed, CI, exemplos por camada) foi atingida ou marcada como `[PENDÊNCIA]`.

---

## 4. Formatação

> 📐 **Padrão visual do documento gerado:**
>
> - Cabeçalho com H1 + H2 e tabela de metadados (nome, versão, data, autor, status).
> - Callout `📌` para TL;DR logo após o cabeçalho.
> - Hierarquia numerada multinível para seções e subseções.
> - Tabelas `fit-page-width` para cenários por módulo, cobertura e gates.
> - Blocos de código para exemplos de testes unitários, integração e E2E.
> - Separadores visuais `---` entre todas as seções principais.
> - Ícones padronizados: `🎯` (objetivo), `💡` (insights), `✅` (validações), `🔴` (alertas críticos), `⚙️` (requisitos e dependências).
> - Callouts `⚙️` com fundo laranja para seções com dados insuficientes.
> - Espaços entre parágrafos para leitura fluida — nunca blocos compactos.

---

## 5. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 21/03/2026 | v1.5 | Adição das seções Critérios de qualidade e Formatação para alinhamento com o padrão gold standard do pipeline. |
| 20/03/2026 | v1.4 | Replicação da filosofia de decisão autônoma: §8 reescrito com `\[DECISÃO AUTÔNOMA\]` como regra geral e `\[DEFINIÇÃO PENDENTE\]` como exceção; cobertura atualizada. |
| 18/03/2026 | v1.3 | Auditoria completa do Prompt para reduzir ambiguidade, fechar critérios de teste e endurecer regras de cobertura e gates. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt para o mesmo nível de detalhe dos demais docs da pipeline. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs. |