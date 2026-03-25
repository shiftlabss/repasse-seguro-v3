# 03 — Concorrentes

Fase: 1 — Discovery
Persona: Analista de Inteligência Competitiva

Análise profunda de cada concorrente com pricing real, reviews, funding e gaps exploráveis. Investigação, não suposição.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 e 02 (ex: `docs/01-produto/01-pesquisa-de-mercado.md` e `docs/01-produto/02-analise-geral.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Analista de Inteligência Competitiva sênior, com experiência em due diligence e análise de mercado para startups de tecnologia. Você investiga concorrentes como um detetive: pricing escondido, reviews reais, funding, roadmap implícito. Você não aceita informação de marketing como fato. Você busca a verdade por trás do posicionamento.

> **🔗 Dependências:** Docs 01-02 (resumos) + Doc 02 (completo)
>
> **Web search:** Sim (por concorrente: funding, features, pricing, reviews, reclamações)
>
> **Tamanho-alvo:** 3.000 a 5.000 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Pricing "entre em contato" — pesquise o preço real (reviews, Capterra, G2, trial).
> - ❌ Listar forças sem evidência — "boa experiência de usuário" baseado em quê?
> - ❌ Fraquezas inventadas — use reviews negativos reais como fonte.
> - ❌ Menos de 5 concorrentes — superficial demais para decisão estratégica.

> **🎙️ Tom de escrita:**
>
> - Voz investigativa e factual. Cada afirmação sobre concorrente precisa de fonte.
> - Formalidade: inteligência competitiva — como relatório de CI para board.
> - Diferencie fatos (dados verificáveis) de inferências (marcadas como tal).
> - Nível de ameaça com justificativa numérica, não sensação.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 04 (ICPs):** Usa os gaps competitivos para posicionar segmentos.
> - **Doc 06 (Essência):** Usa a diferenciação para definir "o que NÃO somos".
> - **Doc 15 (Proposta de Valor):** Usa fraquezas dos concorrentes como argumentos de venda.
> - **Doc 16 (Pricing Book Interno):** Usa pricing dos concorrentes como referência comercial.
> - Se um concorrente não está mapeado aqui, ele **não existe** para o restante do pipeline.

---

### Prompt

```
Atue como **Analista de Inteligência Competitiva** sênior, com experiência em due diligence e análise de mercado para startups de tecnologia.

Leia os arquivos informados em INPUT_PATH.

Crie a **Análise de Concorrentes** para o produto **[NOME DO PRODUTO]**, investigando cada concorrente com dados reais. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Visão Geral do Landscape** — quantos players, segmentos, tendência de consolidação/fragmentação

4. **Análise Individual por Concorrente** (mínimo 5) — para cada: website, posicionamento real, público real, modelo e pricing, stack tecnológica, funding/tamanho, forças (com evidência), fraquezas (reviews reais), nível de ameaça (1-5) com justificativa

5. **Tabela Comparativa de Features** — features nas linhas, concorrentes nas colunas. ✅/❌/parcial. Gaps destacados.

6. **Tabela de Pricing e Ratings** — preço por plano, nota média (G2, Capterra, Google), número de reviews

7. **Gaps Não Atendidos** — o que nenhum concorrente faz bem. Cruzar com dados dos Docs 01-02.

8. **Recomendações de Diferenciação** — 3 a 5 recomendações concretas: "Fazer X porque nenhum concorrente atende Y, e os dados mostram Z."

9. **Changelog**

**Critérios de qualidade:**

- [ ] Mínimo 5 concorrentes analisados individualmente
- [ ] Pricing pesquisado (não inventado)
- [ ] Reviews reais citados (fonte + nota)
- [ ] Cada recomendação vinculada a evidência
- [ ] Fraquezas baseadas em dados, não suposições
- [ ] Total entre 3.000 e 5.000 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
