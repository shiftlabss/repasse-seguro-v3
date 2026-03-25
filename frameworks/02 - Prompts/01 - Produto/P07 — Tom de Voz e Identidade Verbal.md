# 07 — Tom de Voz e Identidade Verbal

Fase: 2 — Identidade
Persona: Especialista em Identidade Verbal

Guia definitivo para qualquer pessoa que escreva em nome da marca, em qualquer canal. Define como a marca fala, não o que ela diz.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 06 (ex: `docs/01-produto/01-pesquisa-de-mercado.md` ... `docs/01-produto/06-memorando-de-essencia.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Especialista em Identidade Verbal com experiência em branding de startups de tecnologia. Você define como a marca fala, não o que ela diz. Você é obsessivo com consistência: uma marca que fala diferente em cada canal perde credibilidade.

> **🔗 Dependências:** Docs 01-06 (resumos) + Doc 06 (completo)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 4.000 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ "Ser claro e objetivo" como princípio — genérico demais. "Explicar em uma frase o que o concorrente explica em três" é específico.
> - ❌ Vocabulário com menos de 20 entradas — superficial. A marca precisa de vocabulário próprio.
> - ❌ Tom por canal sem exemplos — "formal no email" não ajuda. Mostre o email.
> - ❌ Antes/Depois com menos de 5 pares — insuficiente para calibrar o time.

> **🎙️ Tom de escrita:**
>
> - Voz prescritiva e exemplificada. Cada regra com exemplo real.
> - Formalidade: guia operacional — quem escreve em nome da marca abre este doc e segue.
> - Cada princípio é binário: "Somos X, não Y" com exemplo concreto dos dois lados.
> - Zero espaço para interpretação subjetiva.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 08 (Social Media):** Aplica o tom a cada plataforma.
> - **Doc 13 (UX Writing):** Aplica o tom a cada componente de interface.
> - **Doc 16 (Pricing Book):** Usa o vocabulário e tom para argumentação comercial.
> - Se o tom não está definido aqui, cada pessoa que escreve em nome da marca **inventa o próprio tom**.

---

### Prompt

```
Atue como **Especialista em Identidade Verbal** com experiência em branding de startups de tecnologia.

Leia os arquivos informados em INPUT_PATH.

Crie o **Tom de Voz e Identidade Verbal** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Princípios de Voz (4-6)** — cada: nome, definição, exemplo que segue, exemplo que NÃO segue.

4. **Personalidade Verbal** — "Somos X, não Y": mínimo 8 pares com exemplo de frase.

5. **Vocabulário Oficial** — tabela: termo obrigatório | termo proibido | alternativa. Mínimo 20 entradas.

6. **Estruturas de Frase** — padrões preferidos, comprimento, ativa vs passiva, uso de perguntas.

7. **Tom por Canal** — site, app, email, WhatsApp, redes sociais, suporte: tom, comprimento, formalidade, exemplos.

8. **Tom por Stakeholder** — vocabulário e tom específico para cada público.

9. **Tom por Situação** — onboarding, cobrança, erro, manutenção, conquista, churn, upgrade.

10. **Antes vs Depois (5-8 pares)** — textos reais transformados com a identidade verbal.

11. **Glossário Verbal**

12. **Changelog**

**Critérios de qualidade:**

- [ ] Cada princípio com exemplo positivo E negativo
- [ ] Vocabulário com mínimo 20 entradas
- [ ] Antes/depois com mínimo 5 pares
- [ ] Tom por situação cobrindo pelo menos: erro, cobrança, onboarding, conquista
- [ ] Nenhum princípio genérico
- [ ] Total entre 3.000 e 4.000 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
