# 06 — Memorando de Essência

Fase: 2 — Identidade
Persona: Brand Planner

Documento de referência definitivo. Se alguém perguntar "o que é esse produto?", a resposta completa está aqui.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 05 (ex: `docs/01-produto/01-pesquisa-de-mercado.md` ... `docs/01-produto/05-manifesto-da-marca.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Brand Planner sênior que trabalha na interseção entre estratégia de negócios e construção de marca. Você é o guardião da consistência: tudo que a empresa faz, diz e vende deve passar pelo crivo deste documento.

> **🔗 Dependências:** Docs 01-05 (resumos) + Doc 05 (completo)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 4.000 a 6.000 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Frase-essência com mais de 10 palavras — se não é memorável em uma respiração, não funciona.
> - ❌ "O que NÃO é" sem teste ácido — 3 perguntas binárias que detectam desvio de escopo.
> - ❌ Scripts de uso genéricos ("adapte ao contexto") — cada script deve ser prontão para usar.
> - ❌ Glossário com menos de 10 termos — se o produto tem vocabulário próprio, documente.

> **🎙️ Tom de escrita:**
>
> - Voz definitiva e enciclopedista. Este doc é a fonte de verdade sobre o produto.
> - Formalidade: técnico-estratégico — serve para C-level, vendedor e dev.
> - Cada seção deve ser auto-contida: quem ler só uma seção já extrai valor.
> - Referências cruzadas obrigatórias para docs anteriores.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 07 (Tom de Voz):** Usa personalidade e valores como base para identidade verbal.
> - **Doc 09 (Design Thinking):** Usa "o que faz" e "o que NÃO faz" como guardrails de design.
> - **Doc 14 (Modelo de Negócios):** Usa modelo comercial e planos como base financeira.
> - **Doc 15 (Proposta de Valor):** Usa frases-âncora e diferenciação como argumentos.
> - **Doc 16 (Pricing Book Interno):** Usa definição de produto e escopo como base comercial.
> - Este doc é o **hub central**. Se estiver fraco, todos os docs seguintes ficam desalinhados.

---

### Prompt

```
Atue como **Brand Planner sênior**.

Leia os arquivos informados em INPUT_PATH.

Crie o **Memorando de Essência** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Definição do Produto** — posicionamento por geração/onda do mercado (referência Doc 02).

4. **Essência** — personalidade (3-5 traços), frase-essência (10 palavras max), propósito, valores (5 max), KPIs-âncora (3-5 métricas permanentes).

5. **Modelo Operacional Inegociável** — regras que nunca mudam, independente de pivot.

6. **O que Faz (capacidades por stakeholder)** — lista concreta por stakeholder.

7. **O que NÃO é (+ teste ácido)** — lista + 3 perguntas binárias de desvio.

8. **Posição na Operação do Cliente** — diagrama mermaid: antes, durante, depois.

9. **Diferenciação vs Concorrentes** — tabela features/capacidades × produto + concorrentes.

10. **ICPs + Anti-ICP (resumo)** — síntese do Doc 04 com referência cruzada.

11. **Modelo Comercial** — planos, preços, modelo de receita.

12. **One-Liners Oficiais** — do Doc 04 com referência cruzada.

13. **Frases-Âncora** — 10-15 frases para uso diário (vendas, suporte, marketing, produto).

14. **Como Usar** — scripts: pitch 30s/2min/5min, onboarding, objeção, post social. Exercício para novos funcionários.

15. **Referências Cruzadas + Mapa do Ecossistema** — caminhos dos arquivos + mermaid mostrando conexões.

16. **Glossário** — mínimo 10 termos.

17. **Changelog**

**Critérios de qualidade:**

- [ ] Frase-essência com máx 10 palavras
- [ ] Teste ácido com exatamente 3 perguntas binárias
- [ ] Diagrama mermaid funcional
- [ ] Scripts com exemplos completos
- [ ] Glossário com mínimo 10 termos
- [ ] Total entre 4.000 e 6.000 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
