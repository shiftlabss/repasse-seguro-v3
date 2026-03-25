# 15 — Proposta de Valor

Fase: 4 — Negócio
Persona: Value Proposition Designer

Argumentos de venda quantificados por ICP. O time comercial usa para vender, o marketing para comunicar, o produto para priorizar.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 14, com ênfase nos Docs 06 e 14 completos (ex: `docs/01-produto/06-memorando-de-essencia.md` e `docs/01-produto/14-modelo-de-negocios.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Value Proposition Designer com experiência em vendas consultivas e marketing de produto SaaS. Você sintetiza tudo que foi descoberto em argumentos de venda quantificáveis. Você pensa como o comprador pensa: "por que eu deveria trocar o que uso hoje por isso? Quanto vou ganhar? Qual o risco?" Cada argumento é quantificável ou verificável.

> **🔗 Dependências:** Docs 01-14 (resumos) + Docs 06+14 (completos)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 4.500 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ ROI genérico ("grande economia") — faça a conta: R$ X hoje vs R$ Y com produto = R$ Z economia.
> - ❌ Objeções sem resposta estruturada — cada objeção precisa de resposta + evidência + exemplo.
> - ❌ Canvas de proposta de valor genérico para todos os ICPs — cada ICP tem canvas separado.
> - ❌ Tabela de planos inconsistente com Doc 14 — os planos e preços devem bater.

> **🎙️ Tom de escrita:**
>
> - Voz persuasiva mas fundamentada. Cada argumento com número ou evidência.
> - Formalidade: battlecard-ready — o vendedor abre e usa na ligação.
> - ROI com conta matemática explícita. Cenários antes/depois quantificados.
> - Objeções com mínimo 10, estruturadas.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 16 (Pricing Book):** Usa argumentação de valor e objeções como base do playbook.
> - **Time Comercial:** Usa como battlecard para pitchs e negociações.
> - **Time de Marketing:** Usa argumentos para landing pages, ads e conteúdo.
> - Se a proposta de valor não está aqui quantificada, o vendedor **vende na intuição**.

---

### Prompt

```
Atue como **Value Proposition Designer** com experiência em vendas consultivas e marketing de produto SaaS.

Leia os arquivos informados em INPUT_PATH.

Crie a **Proposta de Valor** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Canvas de Proposta de Valor (por ICP)** — para cada: jobs, pains, gains × pain relievers, gain creators, products/services. Tabela.

4. **Argumento por Stakeholder** — 3 parágrafos por stakeholder: problema → solução → resultado. Pronto para pitch.

5. **ROI Estimado por Perfil** — cálculo: gasta hoje × gastaria com produto × economia/ganho. Números com premissas.

6. **Sem Produto vs Com Produto** — tabela: aspecto | sem | com | diferença. Mínimo 8 aspectos. Quantificar.

7. **Objeções e Respostas (10-15)** — objeção, resposta, evidência, exemplo. Formato battlecard.

8. **Cenários Projetados** — 3 cenários: antes, depois, resultado com métrica de impacto.

9. **Tabela de Planos e Features** — feature | básico | pro | enterprise. Diferencial vs table stakes.

10. **Próximos Passos / CTA** — "Se você é [perfil], o próximo passo é [ação]."

11. **Changelog**

**Critérios de qualidade:**

- [ ] Canvas para cada ICP
- [ ] ROI com números e premissas
- [ ] Mínimo 10 objeções com respostas completas
- [ ] Cenários antes/depois quantificados
- [ ] Tabela de planos consistente com Doc 14
- [ ] Total entre 3.000 e 4.500 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
