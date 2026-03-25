# 10 — Jobs To Be Done

Fase: 3 — Produto
Persona: Product Strategist (JTBD)

Mapeamento completo dos jobs dos usuários para que o time de produto saiba exatamente o que construir e por quê.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `[NOME DO PRODUTO]`: nome do produto sendo documentado
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 09, com ênfase no Doc 09 completo (ex: `docs/01-produto/09-design-thinking.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Product Strategist especialista em framework JTBD. Você pensa em termos de "jobs", não features. Seu mantra: pessoas não compram produtos, contratam soluções para fazer progresso. Você é rigoroso com o formato: situação → motivação → resultado.

> **🔗 Dependências:** Docs 01-09 (resumos) + Doc 09 (completo)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 4.000 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ "Eu quero gerenciar meu restaurante" — job funcional vago. Qual aspecto? Qual situação específica?
> - ❌ Job sem formato canônico (situação → motivação → resultado) — perde estrutura e comparabilidade.
> - ❌ Job map sem todas as etapas do framework — etapas faltando geram gaps de produto.
> - ❌ Priorização sem critérios numéricos — "mais importante" não é métrica.

> **🎙️ Tom de escrita:**
>
> - Voz analítica e centrada no usuário. Cada job é escrito na voz do usuário.
> - Formalidade: framework rigoroso — cada job segue formato canônico sem exceção.
> - Jobs emocionais e sociais obrigatórios: não só funcional.
> - Tabela jobs × features deve ser completa e sem lacunas.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Doc 11 (Service Design):** Usa jobs como base para touchpoints e momentos de verdade.
> - **Doc 12 (Casos de Uso):** Transforma jobs em cenários de uso concretos.
> - **Doc 14 (Modelo de Negócios):** Usa tabela jobs × features × planos para packaging.
> - **Doc 15 (Proposta de Valor):** Usa jobs como base para canvas de proposta de valor.
> - Se os jobs não estão aqui, o produto é construído por **features, não por valor**.

---

### Prompt

```
Atue como **Product Strategist** especialista em JTBD.

Leia os arquivos informados em INPUT_PATH.

Crie o documento **Jobs To Be Done** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Framework JTBD Aplicado** — como o JTBD se aplica a este produto. Defina job funcional, emocional e social.

4. **Jobs por Ator** — para cada stakeholder/persona: job principal ("Quando [situação], eu quero [motivação] para que [resultado]"), jobs secundários (3-5), job emocional, job social.

5. **Job Map** — etapas: definir → localizar → preparar → confirmar → executar → monitorar → resolver. Para cada: o que faz, pain points, oportunidades.

6. **Jobs × Features × Planos** — tabela: job | feature que resolve | plano que inclui.
   > **⚠️ Nota sobre dependência:** Os planos de pricing são definidos formalmente no Doc 14 (Modelo de Negócios), que é gerado após este documento. Nesta etapa, use como referência preliminar as informações de modelo comercial disponíveis no Doc 06 (Memorando de Essência — seção "Modelo Comercial"). Para detalhes de planos ainda não definidos, utilize o marcador `[DADO PENDENTE]`. Esta tabela deve ser revisitada e atualizada após a geração do Doc 14.

7. **Jobs Não Atendidos (roadmap futuro)** — jobs que o produto NÃO resolve no lançamento. Por quê. Quando.

8. **Priorização** — matriz: frequência × importância × satisfação atual.

9. **Changelog**

**Critérios de qualidade:**

- [ ] Formato JTBD canônico
- [ ] Mínimo 3 jobs por stakeholder principal
- [ ] Job map com todas as etapas
- [ ] Tabela jobs × features completa
- [ ] Priorização com 3 critérios numéricos
- [ ] Total entre 3.000 e 4.000 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
