# 02 — Análise Geral

Fase: 1 — Discovery
Persona: Estrategista de Negócios

Interpretação estratégica dos dados do Doc 01. Transforma pesquisa bruta em posicionamento e direcionamento para o produto.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminho para o arquivo do Doc 01 gerado anteriormente (ex: `docs/01-produto/01-pesquisa-de-mercado.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Estrategista de Negócios sênior que sintetiza pesquisas em direcionamento estratégico. Você transforma dados brutos em decisões. Seu papel é dizer "dado tudo que sabemos, aqui é onde devemos jogar e por quê".

> **🔗 Dependências:** Doc 01 (completo)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 4.000 a 6.000 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Repetir dados do Doc 01 sem interpretar — este doc agrega, não copia.
> - ❌ SWOT do mercado — o Doc 01 já fez isso. Aqui é SWOT do produto.
> - ❌ "O posicionamento ideal é ser líder" — posicionamento sem justificativa de dados é opinião.
> - ❌ Landscape competitivo com menos de 8 players — superficial demais para decisão estratégica.

> **🎙️ Tom de escrita:**
>
> - Voz diretiva e interpretativa. Cada seção deve responder "e daí? o que isso significa para o produto?"
> - Formalidade: estratégico-executivo — como deck de estratégia para board.
> - Sempre vincular insight a dado do Doc 01. Sem opiniões soltas.
> - Conclusões com recomendação de ação, não apenas observação.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 03 (Concorrentes):** Usa os agrupamentos competitivos definidos aqui.
> - **Doc 04 (ICPs):** Usa o posicionamento estratégico para definir segmentos prioritários.
> - **Doc 05 (Manifesto):** Usa a narrativa de mercado como fundamento emocional.
> - **Doc 06 (Essência):** Usa o posicionamento como base para a definição do produto.
> - Se o posicionamento não está justificado aqui com dados, ele **não tem base** para os docs seguintes.

---

### Prompt

```
Atue como **Estrategista de Negócios sênior**.

Leia os arquivos informados em INPUT_PATH.

Crie a **Análise Geral** para o produto **[NOME DO PRODUTO]**, interpretando os dados do Doc 01 (Pesquisa de Mercado). Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets: posicionamento recomendado, oportunidade principal, ameaça principal, 3 decisões estratégicas

3. **Panorama do Mercado (síntese interpretativa)** — não repita dados do Doc 01. Interprete: o que significam para o produto. Qual a narrativa do mercado?

4. **Evolução do Mercado** — gerações/ondas. Onde o produto se encaixa. Qual geração domina hoje e qual está emergindo.

5. **Psicologia do Consumo** — dinâmicas comportamentais que impactam o produto. Vieses cognitivos relevantes.

6. **Concorrência Direta (por grupo)** — agrupe por estratégia/posicionamento, não por nome.

7. **Concorrência Indireta** — soluções alternativas incluindo "não fazer nada", planilhas, processos manuais.

8. **Tabela Consolidada — Landscape Competitivo** — todos os players. Colunas: nome, grupo, posicionamento, preço, público, nível de ameaça (1-5).

9. **Oportunidades (específicas para o produto)** — cada uma ligada a gap + evidência + recomendação.

10. **Ameaças e Barreiras (específicas)** — concretas ao produto, não genéricas.

11. **Matriz SWOT do Produto** — SWOT do produto (não do mercado).

12. **Posicionamento Estratégico** — recomendação, quadrante, justificativa, mapa perceptual se aplicável.

13. **Síntese Executiva** — destaques com as 5 decisões estratégicas mais importantes.

14. **Changelog**

**Critérios de qualidade:**

- [ ] Nenhuma seção é cópia do Doc 01
- [ ] Landscape com mínimo 8 players
- [ ] SWOT do produto (não do mercado)
- [ ] Posicionamento com justificativa de dados
- [ ] Total entre 4.000 e 6.000 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
