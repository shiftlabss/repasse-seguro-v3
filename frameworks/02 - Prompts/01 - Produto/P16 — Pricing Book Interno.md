# 16 — Pricing Book Interno

Fase: 4 — Negócio
Persona: Revenue Strategist / Sales Enablement Lead

Manual interno definitivo do time comercial. Quanto cobrar, de quem, como posicionar, como responder objeções, quais descontos dar.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 15, com ênfase nos Docs 04, 06, 14 e 15 completos (ex: `docs/01-produto/04-one-liner-e-icps.md`, `docs/01-produto/06-memorando-de-essencia.md`, `docs/01-produto/14-modelo-de-negocios.md`, `docs/01-produto/15-proposta-de-valor.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Revenue Strategist e Sales Enablement Lead com experiência em operações comerciais SaaS. Você cria documentos que o vendedor abre às 8h da manhã e usa o dia inteiro. Cada seção resolve uma dúvida real que aparece na ligação de vendas. Você pensa como o vendedor: "como eu explico isso em 30 segundos para o prospect sem perder a venda?"

> **🔗 Dependências:** Docs 01-15 (resumos) + Docs 04+06+14+15 (completos)
>
> **Web search:** Sim (benchmarks de pricing do setor, práticas de desconto, modelos de comissão)
>
> **Tamanho-alvo:** 3.500 a 5.500 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Pitch com mais de 2 frases — se não cabe em 30 segundos falando, não funciona na ligação.
> - ❌ ROI sem conta matemática — "grande retorno" não convence. "R$ 500/mês × 12 = R$ 6.000/ano, produto custa R$ 2.400/ano = ROI de 150%" convence.
> - ❌ Menos de 15 objeções — vendedor encontra mais que isso na primeira semana.
> - ❌ Política de descontos sem teto — sem guardrail, vendedor dá desconto até zerar margem.
> - ❌ Playbook sem falas sugeridas — "posicione o valor" não ajuda. Mostre a fala pronta.

> **🎙️ Tom de escrita:**
>
> - Voz direta e prática. Este doc é operacional, não estratégico.
> - Formalidade: manual de campo — vendedor abre no celular durante almoço com prospect.
> - Falas sugeridas no formato citação (">"). Prontas para falar, não para ler.
> - Callouts para: regra de ouro, guardrails, alertas.
> - Tudo que depender de dados específicos do produto vem do briefing ou é marcado `[DADO PENDENTE]`.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Time Comercial:** Este é o documento principal do vendedor. Abre todo dia.
> - **Doc 14 (Modelo de Negócios):** Planos e preços são consistentes com o modelo.
> - **Doc 15 (Proposta de Valor):** Argumentação e objeções são expandidas aqui.
> - **Doc 04 (ICPs):** Segmentação e qualificação derivam dos perfis definidos lá.
> - Se o Pricing Book não está aqui, o vendedor **negocia sem guardrails**.

---

### Prompt

```
Atue como **Revenue Strategist e Sales Enablement Lead** com experiência em operações comerciais SaaS.

Leia os arquivos informados em INPUT_PATH.

Crie o **Pricing Book Interno** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Objetivo deste Documento** — o que é, para quem, como usar. Destaque: fonte de verdade do time comercial.

4. **O que é o Produto (e o que NÃO é)** — definição 3-5 linhas. Lista do que NÃO vender/prometer. Alerta em destaque. Baseado no Doc 06.

5. **Estrutura de Pricing** — setup + mensalidade + comissão/variável. Tabela de planos. Tabela de setup. Se não definido: `[DADO PENDENTE]`.

6. **Segmentos-alvo e Qualificação** — ICP resumido para vendedor (do Doc 04). Quem é foco, quem NÃO, perguntas de qualificação (4-6). Checklist em destaque.

7. **Quando Vender Cada Plano** — para cada plano: quando usar, perfil ideal, pitch em 1-2 frases.

8. **Argumentação de Valor e ROI** — lógica simples, simulação modelo, conta rápida. Guardrail: nunca inventar números.

9. **Política Comercial e Descontos** — tabela de descontos por condição. Teto sem aprovação. O que exige diretoria. Regra: nunca começar oferecendo.

10. **Playbook de Venda** — 5 etapas com falas sugeridas (">"): diagnóstico → dor → plano → ROI → próximo passo.

11. **Objeções Comuns (mínimo 15)** — estrutura por objeção: validar → reframing → prova → próximo passo. Obrigatórias: "está caro", "não preciso disso", "já tenho solução", "quero desconto", "e se não funcionar?", "quanto tempo leva?". Restantes: derivar dos ICPs e setor.

12. **Expansão e Upsell** — quando e como fazer upgrade. Gatilhos. Regra: não forçar antes de resultado.

13. **FAQ Interno** — 5-10 perguntas do vendedor sobre regras internas. Pergunta → resposta curta.

14. **Como Usar Este Documento** — fonte de verdade, o que fazer fora do escopo, quando escalar.

15. **Changelog**

**Critérios de qualidade:**

- [ ] Pitchs por plano com máx 2 frases (prontos para falar)
- [ ] Simulação de ROI com conta matemática explícita
- [ ] Mínimo 15 objeções com resposta estruturada
- [ ] Política de descontos com teto e regras de aprovação
- [ ] Playbook com falas sugeridas em formato citação
- [ ] Tom direto e prático (operacional, não estratégico)
- [ ] Dados específicos do produto vindos dos arquivos de input ou marcados `[DADO PENDENTE]`
- [ ] Total entre 3.500 e 5.500 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
