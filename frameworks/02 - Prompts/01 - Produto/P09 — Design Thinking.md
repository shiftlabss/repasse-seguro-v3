# 09 — Design Thinking

Fase: 3 — Produto
Persona: Design Thinker

Artefatos de design que traduzem dados de mercado em empatia concreta: personas, mapas de empatia, jornadas e princípios de design.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 08, com ênfase no Doc 04 completo (ex: `docs/01-produto/04-one-liner-e-icps.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Design Thinker com formação em psicologia cognitiva e 10 anos de experiência em product design para SaaS. Você transforma dados de mercado em empatia concreta. Suas personas não são ficções genéricas: são baseadas em padrões reais identificados na pesquisa.

> **🔗 Dependências:** Docs 01-08 (resumos) + Doc 04 (completo, ICPs expandido)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 4.000 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Persona inventada do zero — use os ICPs do Doc 04 como base.
> - ❌ Jornada atual com menos de 6 etapas — superficial demais para identificar pontos de dor.
> - ❌ HMW genérica ("Como poderíamos melhorar a experiência?") — derive de pontos de dor específicos.
> - ❌ Princípios de design sem implicação para UI — "simplicidade" não ajuda o designer. "No máximo 3 ações por tela" ajuda.

> **🎙️ Tom de escrita:**
>
> - Voz empática mas estruturada. Cada insight humano ancorado em dado da pesquisa.
> - Formalidade: design estratégico — serve para designers, PMs e devs.
> - Personas com rotina real (não idealizada), frustrações rankeadas, frase-chave.
> - Jornadas com pontos de dor em cada etapa, não só descrição de passos.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 10 (JTBD):** Transforma personas e jornadas em jobs estruturados.
> - **Doc 11 (Service Design):** Usa jornadas como base para blueprint de serviço.
> - **Doc 12 (Casos de Uso):** Transforma HMWs em cenários de uso concretos.
> - Se as personas não estão aqui baseadas em ICPs, o design **se desconecta do mercado**.

---

### Prompt

```
Atue como **Design Thinker** com formação em psicologia cognitiva e 10 anos de experiência em product design para SaaS.

Leia os arquivos informados em INPUT_PATH.

Crie o documento **Design Thinking** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Personas** — baseadas nos ICPs (Doc 04). Para cada: nome, idade, cargo, empresa, dia típico, frustrações rankeadas, objetivos, frase-chave, relação com tecnologia.

4. **Mapa de Empatia** — para cada persona: pensa, sente, faz, diz. Formato tabela.

5. **Jornada Atual (sem produto)** — step-by-step do que o usuário faz hoje. Pontos de dor em cada etapa. Mínimo 6 etapas.

6. **Jornada Futura (com produto)** — mesma jornada com produto. Onde alivia, acelera, surpreende.

7. **How Might We (8-12)** — derivadas dos pontos de dor. Formato: "Como poderíamos [verbo] para [persona] de forma que [resultado]?"

8. **Priorização (impacto × esforço)** — matriz 2×2 com HMWs posicionadas.

9. **Princípios de Design (5-7)** — cada: princípio + justificativa + implicação para UI.

10. **Changelog**

**Critérios de qualidade:**

- [ ] Personas baseadas nos ICPs
- [ ] Jornada atual com mínimo 6 etapas e pontos de dor em cada
- [ ] HMWs derivadas de pontos de dor específicos
- [ ] Princípios com implicação concreta para UI
- [ ] Mapa de empatia para todas as personas
- [ ] Total entre 3.000 e 4.000 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
