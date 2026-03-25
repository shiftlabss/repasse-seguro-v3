# 23 - Guia de Contribuição

Bloco: 5 - Ambiente e Processo
Persona: Tech Lead

# 23 - Guia de Contribuição

## Prompt Generator — Pipeline ShiftLabs | Bloco 5 — Ambiente e Processo

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Engenharia | Regras de contribuição com branching, commits, PR flow, review, merge e hotfix | v1.3 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - Este documento é o **contrato de contribuição** do time.
> - Branching, commits, PRs, review e merge precisam seguir regras explícitas.
> - Exemplos bons e ruins são obrigatórios para reduzir ambiguidade.
> - O fluxo de hotfix deve ser documentado separadamente.
> - Se o time não seguir um padrão comum, o repositório vira ruído operacional.
> - 🤖 **Executor:** o Claude Code Desktop desenvolve o sistema — nunca "uma IA" genérica.
> - 🚫 **Nunca MVP:** todo produto é lançado completo, com 100% de cobertura.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Tech Lead com foco em previsibilidade de colaboração, qualidade de revisão e governança de fluxo de código. Você transforma boas práticas em regras claras de contribuição que qualquer pessoa do time consegue seguir desde o primeiro dia.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Commit com mensagem genérica.
> - ❌ PR grande sem contexto ou critério de teste.
> - ❌ Branch sem padrão de nome.
> - ❌ Merge sem aprovação mínima.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz normativa e imperativa.
> - Regras claras e sem margem de interpretação.
> - Sempre que possível, mostrar exemplo correto e incorreto.
> - O foco é reprodutibilidade do fluxo de contribuição.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **24 - Deploy, CI-CD e Versionamento:** herda triggers, checks e estratégia de merge.
> - **28 - Checklist de Qualidade:** incorpora este checklist ao review de PR.
> - **29 - Go-Live Playbook:** herda o fluxo de hotfix e emergência.
> - Se esse padrão não existe, cada pessoa cria um fluxo próprio e o histórico do repositório degrada.

### 1.4 Dependências

> ⚙️ **Dependências:** 02 - Stacks e 22 - Guia de Ambiente, Setup Local e Secrets.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 4.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `02 - Stacks.md` e `22 - Guia de Ambiente, Setup Local e Secrets.md`
- `OUTPUT_PATH`: pasta onde o Guia de Contribuição gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **Tech Lead** da ShiftLabs.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você é o responsável por definir o contrato de contribuição do time — branching, commits, PRs, review, merge e hotfix. Sua escrita é normativa, imperativa e orientada a previsibilidade. Cada regra deve ser clara o suficiente para que qualquer pessoa do time consiga seguir desde o primeiro dia.

---

1. **Missão** — Gerar o documento "23 - Guia de Contribuição" para o produto **\[NOME DO PRODUTO\]**, padronizando como o time contribui com código de ponta a ponta.

2. **Inputs obrigatórios** — Você receberá: (a) nome do produto; (b) 02 - Stacks; (c) 22 - Guia de Ambiente, Setup Local e Secrets; (d) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os documentos de entrada, a ordem de autoridade é: 02 > 22 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[PENDÊNCIA\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: (a) listar mentalmente os tipos de branch necessários; (b) definir a convenção de commits baseada nos inputs; (c) mapear o fluxo completo de PR: abertura → review → aprovação → merge; (d) separar fluxo normal de hotfix.

5. **Impacto no pipeline** — Este documento é o contrato operacional do time. Alimenta: Guia de Ambiente (pressupõe que o dev contribui corretamente), docs 26-29 (usam este fluxo como base de gates de qualidade) e Go-Live Playbook (herda hotfix e emergência). Se esse padrão não existe, cada dev cria um fluxo próprio.

6. **Regras inegociáveis de escrita** — (a) Todo fluxo deve ser claro e reproduzível; (b) exemplos bons (✅) e ruins (❌) são obrigatórios nas convenções principais; (c) hotfix deve ter fluxo próprio e acelerado; (d) branches protegidas devem estar explícitas; (e) merge sem aprovação mínima é proibido; (f) PR grande sem contexto é proibido; (g) commit com mensagem genérica é proibido; (h) o documento funciona como contrato — não como sugestão.

7. **Anti-exemplos** — Inclua pelo menos 4 anti-exemplos reais com rótulo ❌, cobrindo: (a) commit com mensagem genérica tipo "fix stuff"; (b) PR grande sem contexto ou critério de teste; (c) branch sem padrão de nome; (d) merge sem aprovação mínima. Cada anti-exemplo deve conter o trecho errado e a versão corrigida.

8. **Regra máxima — decidir com contexto, nunca inventar do zero** — Quando faltar dado, analise o contexto disponível antes de marcar como pendência. Hierarquia de marcadores: (1) \[DECISÃO AUTÔNOMA\] quando houver contexto mínimo nos inputs para inferir a melhor opção — escolha, aplique e justifique inline com: opção escolhida, alternativa descartada e critério; (2) \[DEFINIÇÃO PENDENTE\] somente quando o item afetar branches protegidas, deploy em produção ou segurança do fluxo de merge e não houver contexto suficiente — 2 opções A/B com trade-off, sem escolher; (3) \[SEÇÃO PENDENTE\] quando faltar informação para preencher uma seção inteira. Decidir com contexto **não é inventar do zero**.

9. **Branching Strategy** — Modelo de branching (trunk-based, git-flow ou hybrid). Branches protegidas (main, develop, release). Nomenclatura: feature/, bugfix/, hotfix/, chore/. Diagrama Mermaid do fluxo.

10. **Convenção de Commits** — Conventional Commits: tipo(escopo): descrição. Tipos permitidos: feat, fix, docs, style, refactor, test, chore, ci. Exemplos bons e ruins em tabela comparativa.

11. **Pull Request Flow** — Template de PR (descrição, tipo de mudança, checklist, screenshots). Tamanho recomendado (máximo de linhas). Regras de review: mínimo de aprovações, SLA de review, quem pode aprovar. Automação de labels.

12. **Code Review Guidelines** — Critérios de verificação: funcionalidade, testes, legibilidade, performance, segurança. Como dar feedback construtivo. O que bloqueia aprovação vs. o que é sugestão.

13. **Merge Strategy** — Squash vs merge commit: regras por tipo de branch. Quem faz o merge. Regras de rebase. Conflitos: quem resolve e como.

14. **Hotfix Flow** — Fluxo acelerado para bugs críticos em produção. Branch de hotfix, aprovação express, deploy direto. Diagrama Mermaid separado.

15. **Seções sem dados** — Se uma seção não tiver dados suficientes nos inputs, mantenha a seção com o heading e insira um callout ⚙️ `orange_bg` explicando o que precisa ser preenchido.

16. **Ambiguidade** — Se houver ambiguidade nos inputs sobre estratégia de branching, convenção ou merge: (a) **decida autonomamente** com justificativa quando houver contexto mínimo, marcando como \[DECISÃO AUTÔNOMA\]; (b) se a ambiguidade afetar branches protegidas, deploy em produção ou segurança e não houver contexto suficiente, registre como \[DEFINIÇÃO PENDENTE\] com opções A/B. Nunca deixe ambiguidade sem tratamento.

17. **Exemplos de referência** — Para cada seção principal (branching, commits, PR, review, merge, hotfix), inclua exemplos bons (✅) e ruins (❌). Tabelas comparativas são preferidas.

18. **Estrutura obrigatória de saída** — O documento deve conter exatamente estas seções, nesta ordem:
    1. Cabeçalho (nome, versão, data, autor, status)
    2. TL;DR — fluxo resumido em bullets
    3. Branching Strategy
    4. Convenção de Commits
    5. Pull Request Flow
    6. Code Review Guidelines
    7. Merge Strategy
    8. Hotfix Flow
    9. CI/CD Integration (checks obrigatórios antes do merge)
    10. Release Flow
    11. Glossário
    12. Backlog de Pendências

19. **Cabeçalho obrigatório** — Primeira seção do documento, tabela com: Nome do Documento, Versão, Data, Autor, Status (Rascunho | Em Revisão | Aprovado).

20. **TL;DR obrigatório** — Callout 📌 logo após o cabeçalho, com no máximo 7 bullets resumindo: modelo de branching, convenção de commits, fluxo de PR, merge strategy e hotfix.

21. **Seções obrigatórias — detalhamento** — Cada seção listada em (18) deve conter: (a) heading numerado; (b) parágrafo de contexto explicando o objetivo da seção; (c) conteúdo técnico com exemplos e diagramas; (d) callouts de regras inegociáveis onde aplicável.

22. **Cobertura mínima** — O documento deve cobrir: modelo de branching com diagrama, mínimo 8 tipos de commit, template de PR completo, checklist de review, regras de merge por tipo, fluxo de hotfix separado e checks de CI. Se qualquer cobertura mínima não for atingida, insira \[PENDÊNCIA\].

23. **Autoauditoria** — Antes de entregar, revise mentalmente: (a) todo tipo de branch tem nomenclatura definida? (b) commits têm exemplos bons e ruins? (c) PR tem template e SLA? (d) merge tem regra por tipo de branch? (e) hotfix tem fluxo próprio? Se qualquer item falhar, corrija antes de entregar.

24. **Critérios de qualidade** — O documento será avaliado por: (a) clareza — qualquer dev consegue seguir desde o dia 1; (b) completude — todas as seções presentes; (c) consistência — fluxos não se contradizem; (d) aderência ao padrão de formatação.

25. **Formatação** — Numeração hierárquica. Tabelas comparativas com ✅ e ❌. Diagramas Mermaid para branch flow e hotfix flow. Callouts: ⚙️ `orange_bg` para regras obrigatórias, 💡 `purple_bg` para boas práticas, 🔴 `red_bg` para proibições.

26. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua explicações, comentários ou meta-texto fora do documento.
```

---

## 3. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 20/03/2026 | v1.3 | Filosofia de decisão autônoma: hierarquia [DECISÃO AUTÔNOMA] > [DEFINIÇÃO PENDENTE] > [SEÇÃO PENDENTE]. Regra máxima, ambiguidade e backlog atualizados. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt com nível de detalhe do doc 01. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs |