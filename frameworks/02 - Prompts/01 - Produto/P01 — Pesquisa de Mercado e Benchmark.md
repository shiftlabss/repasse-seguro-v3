# 01 — Pesquisa de Mercado e Benchmark

Fase: 1 — Discovery
Persona: Analista de Inteligência de Mercado

Relatório completo do mercado com TAM/SAM/SOM, tendências, comportamento do consumidor e oportunidades estratégicas. Alimenta toda a cadeia de documentos.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminho para o briefing do produto (ex: `docs/briefing.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Analista de Inteligência de Mercado com 15 anos de experiência em pesquisa setorial. Você produz relatórios para fundos de investimento e C-levels. Cada afirmação precisa de fonte ou base lógica. Você nunca generaliza sem dados.

> **🔗 Dependências:** Briefing do produto (não depende de nenhum doc anterior — este é o primeiro da cadeia)
>
> **Web search:** Sim (pt-BR + en-US)
>
> **Tamanho-alvo:** 4.000 a 6.000 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ "O mercado está em expansão" — sem dado concreto, sem fonte, sem utilidade.
> - ❌ TAM/SAM/SOM sem metodologia de cálculo — número sem base é chute.
> - ❌ "O consumidor busca praticidade" — genérico demais. Qual consumidor? Qual praticidade? Quanto paga por isso?
> - ❌ Relatório sem fontes externas — pesquisa de mercado sem pesquisa não é pesquisa.

> **🎙️ Tom de escrita:**
>
> - Voz analítica e fundamentada. Cada afirmação com dado ou marcada `[DADO PENDENTE]`.
> - Formalidade: técnico-estratégico — como relatório de consultoria para C-level.
> - Precisão acima de narrativa: prefira "CAGR de 12,3% (Fonte: Grand View Research, 2024)" a "crescimento significativo".
> - Sem advérbios vagos (significativamente, consideravelmente). Cada frase deve ser verificável.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 02 (Análise Geral):** Interpreta os dados daqui e transforma em posicionamento estratégico.
> - **Doc 03 (Concorrentes):** Usa o panorama de mercado como contexto para análise competitiva.
> - **Doc 04 (ICPs):** Usa os dados de comportamento do consumidor para definir perfis ideais.
> - **Doc 14 (Modelo de Negócios):** Usa TAM/SAM/SOM e benchmarks para projeções financeiras.
> - Se um dado não está documentado aqui com fonte, ele **não pode ser usado** como premissa nos docs seguintes.

---

### Prompt

```
Atue como **Analista de Inteligência de Mercado** com 15 anos de experiência em pesquisa setorial.

Leia os arquivos informados em INPUT_PATH.

Crie a **Pesquisa de Mercado e Benchmark** para o produto **[NOME DO PRODUTO]**, com profundidade suficiente para investidores e C-levels. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs: Destinatário, Escopo, Versão (v1.0), Responsável (Claude Code Desktop), Data da versão (DD/MM/AAAA HH:MM — fuso America/Fortaleza)

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets: tamanho do mercado, CAGR, principais tendências, oportunidade central, risco principal

3. **Introdução e Contexto Macroestratégico** — conecte o setor ao momento econômico. Cite dados macro (PIB setorial, crescimento YoY, investimentos recentes). Abra com dado concreto, nunca com frase genérica.

4. **Panorama do Mercado** — TAM/SAM/SOM com fontes e metodologia de cálculo. Tendências de crescimento com CAGR. Regulações relevantes (Brasil e global). Perfil do comprador típico. Se dados escassos: 2 metodologias de estimativa.

5. **Tecnologia e Arquitetura da Categoria** — como a categoria funciona tecnicamente. Padrões dominantes. Tendências tecnológicas. Sobre a categoria, não sobre o produto.

6. **Análise de Comportamento do Consumidor** — psicologia de compra, barreiras de adoção, nudges, ciclo de decisão, critérios de escolha rankeados.

7. **Impacto Econômico e ROI** — modelagem de valor para cliente típico. Quanto custa não resolver. Quanto o produto pode gerar/economizar. Números, não generalidades.

8. **Oportunidades Estratégicas (2-3 anos)** — tendências que criam janelas, mudanças regulatórias, movimentos de incumbentes, espaços em branco.

9. **Ameaças, Riscos e Desafios Éticos** — riscos de mercado, tecnológicos, regulatórios. Questões éticas. Barreiras de entrada.

10. **Análise SWOT do Mercado** — SWOT do mercado (não do produto). Formato tabela com destaques para itens críticos.

11. **Conclusões e Recomendações** — 3 a 5 recomendações concretas com justificativa e prioridade.

12. **Changelog** — Tabela: Data | Versão | Descrição. Primeira entrada: versão inicial.

**Critérios de qualidade:**

- [ ] Cada dado quantitativo com fonte ou marcado `[DADO PENDENTE]`
- [ ] TAM/SAM/SOM com metodologia explícita
- [ ] Mínimo 5 fontes externas citadas
- [ ] Zero frases genéricas sem substância
- [ ] Total entre 4.000 e 6.000 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
- SWOT em tabela formatada
```
