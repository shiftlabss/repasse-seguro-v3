# 12 — Casos de Uso

Fase: 3 — Produto
Persona: Product Manager

Todos os cenários de uso do produto com detalhe suficiente para alimentar especificações técnicas. Cada caso é implementável sem perguntar.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 11, com ênfase nos Docs 10 e 11 completos (ex: `docs/01-produto/10-jobs-to-be-done.md` e `docs/01-produto/11-service-design.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Product Manager sênior com experiência em especificação funcional de produtos SaaS. Você documenta casos de uso para o time de desenvolvimento. Cada caso precisa ser tão claro que um desenvolvedor consegue implementar sem perguntar nada. Você pensa em cenários reais, incluindo edge cases.

> **🔗 Dependências:** Docs 01-11 (resumos) + Docs 10+11 (completos)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 4.500 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Caso de uso sem pré-condições — o dev não sabe o estado inicial do sistema.
> - ❌ Fluxo principal sem exceções — só happy path não prepara para o mundo real.
> - ❌ Menos de 10 casos — um SaaS tem mais de 10 cenários de uso relevantes.
> - ❌ Tabela resumo com lacunas — cada caso deve estar mapeado a ator, feature e plano.

> **🎙️ Tom de escrita:**
>
> - Voz técnica e cenário-driven. Cada caso é um cenário real com contexto.
> - Formalidade: técnico-funcional — serve para dev, QA e PM.
> - Step-by-step com numeração. Exceções tratadas explicitamente.
> - Edge cases com mínimo 3 cenários.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 13 (UX Writing):** Usa cenários para definir textos contextuais.
> - **Doc 15 (Proposta de Valor):** Usa cenários como antes/depois para argumentação.
> - Se um cenário de uso não está aqui, ele **não será implementado** corretamente.

---

### Prompt

```
Atue como **Product Manager sênior** com experiência em especificação funcional de SaaS.

Leia os arquivos informados em INPUT_PATH.

Crie o documento **Casos de Uso** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Casos por Propósito/Pilar** — agrupados por funcionalidade. Para cada: nome, cenário real, atores, pré-condições, fluxo principal step-by-step, resultado esperado, exceções.

4. **Casos por ICP** — mesmos casos reorganizados por perfil de cliente.

5. **Casos Edge** — cenários incomuns mas possíveis. Limites do sistema. Mínimo 3.

6. **Tabela Resumo: Caso × Ator × Feature × Plano** — visão consolidada.

7. **Changelog**

**Critérios de qualidade:**

- [ ] Mínimo 10 casos documentados
- [ ] Cada caso com pré-condições e exceções
- [ ] Casos edge com mínimo 3 cenários
- [ ] Tabela resumo completa e sem lacunas
- [ ] Total entre 3.000 e 4.500 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
