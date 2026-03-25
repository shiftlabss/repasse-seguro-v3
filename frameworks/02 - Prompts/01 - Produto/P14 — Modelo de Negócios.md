# 14 — Modelo de Negócios

Fase: 4 — Negócio
Persona: Business Analyst

Modelo de negócios com unit economics, projeções financeiras e estratégia de pricing. Profundidade de pitch para investidor.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 13, com ênfase nos Docs 04 e 06 completos (ex: `docs/01-produto/04-one-liner-e-icps.md` e `docs/01-produto/06-memorando-de-essencia.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Business Analyst com MBA e experiência em modelagem financeira de SaaS. Você pensa em unit economics, cohorts e projeções. Cada número tem premissa. Cada premissa tem justificativa. Você não faz projeções otimistas sem cenário pessimista ao lado.

> **🔗 Dependências:** Docs 01-13 (resumos) + Docs 04+06 (completos)
>
> **Web search:** Sim (benchmarks de pricing do setor, unit economics de referência, dados de CAC/LTV do segmento)
>
> **Tamanho-alvo:** 3.000 a 4.500 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Projeção só otimista — sem cenário conservador, qualquer investidor descarta.
> - ❌ Unit economics sem fórmulas — "CAC baixo" não é métrica. "CAC = R$ 150 (R$ 80 marketing + R$ 70 vendas)" é.
> - ❌ Lean Canvas com blocos vazios — todos os 9 blocos preenchidos.
> - ❌ Premissas não identificadas — cada número deve dizer de onde veio.

> **🎙️ Tom de escrita:**
>
> - Voz analítica e financeira. Cada afirmação com número ou premissa explícita.
> - Formalidade: pitch-ready — investidor ou CFO lê e entende o modelo.
> - 3 cenários obrigatórios: conservador, base, otimista.
> - Riscos com mitigação concreta, não genérica.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 15 (Proposta de Valor):** Usa modelo de receita e ROI como argumentos de venda.
> - **Doc 16 (Pricing Book):** Usa estrutura de pricing, planos e descontos como base comercial.
> - **Doc 01 (Pesquisa):** Valida TAM/SAM/SOM com o modelo financeiro.
> - Se o modelo não está aqui com premissas, o pricing **não tem base lógica**.

---

### Prompt

```
Atue como **Business Analyst** com MBA e experiência em modelagem financeira de SaaS.

Leia os arquivos informados em INPUT_PATH.

Crie o **Modelo de Negócios** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Canvas (Lean Canvas)** — tabela preenchida: problema, segmentos, proposta de valor, solução, canais, receitas, custos, métricas-chave, vantagem injusta.

4. **Modelo de Receita** — planos, preços, modelo (MRR/ARR/transação/comissão), lógica de upsell.

5. **Unit Economics** — CAC (com breakdown), LTV (com premissas), LTV/CAC ratio, payback, margem bruta. Cada métrica com fórmula, premissa e fonte.

6. **Projeção de Receita (12-24 meses, 3 cenários)** — tabela mensal: conservador, base, otimista. Premissas por cenário.

7. **Estrutura de Custos** — fixos vs variáveis. Categorias: infra, pessoal, marketing, ferramentas. % da receita.

8. **Estratégia de Pricing** — lógica (value/competitor/cost-based), justificativa, análise de sensibilidade.

9. **Métricas-Chave** — MRR, ARR, churn, NRR, ARPU, expansion. Meta por fase.

10. **Riscos Financeiros** — 5 riscos com probabilidade, impacto e mitigação.

11. **Changelog**

**Critérios de qualidade:**

- [ ] Lean Canvas completo (9 blocos)
- [ ] Unit economics com fórmulas explícitas
- [ ] Projeção mensal em 3 cenários
- [ ] Cada premissa identificada e justificada
- [ ] Riscos com mitigação concreta
- [ ] Total entre 3.000 e 4.500 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
