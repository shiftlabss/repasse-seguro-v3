# 05 — Manifesto da Marca

Fase: 2 — Identidade
Persona: Brand Strategist

Documento fundacional que define o "porquê" da marca. Referência para comunicação, contratação e produto.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 04 (ex: `docs/01-produto/01-pesquisa-de-mercado.md` ... `docs/01-produto/04-one-liner-e-icps.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Brand Strategist com formação em filosofia e experiência em startups de impacto. Você escreve manifestos que são lidos em voz alta e fazem as pessoas sentirem algo. Mas você não é superficial: cada frase emocional é fundamentada em uma verdade do mercado ou do usuário.

> **🔗 Dependências:** Docs 01-04 (resumos) + Doc 04 (completo)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.000 a 3.000 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Manifesto que poderia ser de qualquer empresa — teste: substitua o nome do produto por outro. Se funcionar, reescreva.
> - ❌ "Acreditamos na inovação" — genérico. Todo mundo "acredita na inovação". Qual inovação específica?
> - ❌ Crenças sem consequência prática — se a crença não muda uma decisão de produto, não é crença, é slogan.
> - ❌ Anti-manifesto ausente — definir o que a marca NÃO é é tão importante quanto definir o que ela é.

> **🎙️ Tom de escrita:**
>
> - Voz narrativa, emocional mas fundamentada. Cada frase emocional ancorada em verdade do mercado.
> - Formalidade: autoral — este doc é lido em voz alta, não é relatório.
> - O manifesto deve provocar sensação de "é exatamente isso" em quem lê.
> - Crenças e princípios: tom direto e pragmático.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 06 (Essência):** Usa crenças e princípios como base para a definição do produto.
> - **Doc 07 (Tom de Voz):** Deriva o tom da personalidade definida no manifesto.
> - **Doc 08 (Social Media):** Usa os pilares narrativos do manifesto para conteúdo.
> - **Doc 16 (Pricing Book Interno):** Usa o posicionamento emocional para argumentação de valor.
> - Se o manifesto não ressoa com o time, os docs de identidade inteiros **perdem fundamento**.

---

### Prompt

```
Atue como **Brand Strategist** com formação em filosofia e experiência em startups de impacto.

Leia os arquivos informados em INPUT_PATH.

Crie o **Manifesto da Marca** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Manifesto** — texto narrativo, emocional, fundamentado. Estrutura: contexto do problema → visão de mundo → o que acreditamos → o que estamos construindo → convite. Específico a este produto e mercado. 800-1.200 palavras.

4. **Crenças Fundamentais (5-7)** — cada: frase + parágrafo explicando impacto nas decisões do produto. Formato: "Acreditamos que [crença]. Por isso, [consequência prática]."

5. **Princípios de Marca (5-7)** — cada: nome, descrição, exemplo prático, contra-exemplo (o que NÃO fazer).

6. **Promessas (por stakeholder)** — uma promessa concreta por stakeholder. Formato: "Para [stakeholder], prometemos [ação verificável]."

7. **Anti-Manifesto** — o que a marca NÃO é, NÃO faz, NÃO promete. Mínimo 5 itens.

8. **Como Usar Este Documento** — instruções práticas: reuniões de produto, entrevistas, briefings, decisões de feature.

9. **Changelog**

**Critérios de qualidade:**

- [ ] Manifesto lê como texto autoral, não template preenchido
- [ ] Zero frases intercambiáveis com outra empresa
- [ ] Cada crença com consequência prática
- [ ] Anti-manifesto com mínimo 5 itens
- [ ] Princípios com exemplos E contra-exemplos
- [ ] Total entre 2.000 e 3.000 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
