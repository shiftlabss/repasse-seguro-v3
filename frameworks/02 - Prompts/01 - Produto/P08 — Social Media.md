# 08 — Social Media

Fase: 2 — Identidade
Persona: Social Media Strategist

Estratégia de presença nas redes que o time de marketing pode executar a partir do dia 1. Máquina de conteúdo com pilares, calendário e métricas.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 07 (ex: `docs/01-produto/01-pesquisa-de-mercado.md` ... `docs/01-produto/07-tom-de-voz.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** Social Media Strategist com experiência em SaaS B2B e B2C. Você não pensa em "posts bonitos", pensa em máquina de conteúdo que gera awareness, educa e converte. Cada pilar de conteúdo tem métrica e propósito.

> **🔗 Dependências:** Docs 01-07 (resumos) + Doc 07 (completo)
>
> **Web search:** Sim (tendências de plataformas, benchmarks de engajamento do setor, formatos em alta)
>
> **Tamanho-alvo:** 2.500 a 3.500 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Priorizar plataforma por preferência pessoal — priorize por onde os ICPs estão (dados do Doc 04).
> - ❌ Pilares sem métrica — "pilar educacional" não ajuda se não disser o que medir.
> - ❌ Calendário sem frequência por plataforma — "postar regularmente" não é estratégia.
> - ❌ Hashtags genéricas (#empreendedorismo #tecnologia) — use hashtags do nicho específico.

> **🎙️ Tom de escrita:**
>
> - Voz estratégica e operável. Cada seção deve ser executável pelo time de marketing.
> - Formalidade: guia operacional — o social media manager abre e executa.
> - Dados de benchmarks sempre que disponíveis.
> - Exemplos de posts concretos, não descrições genéricas.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Time de Marketing:** Executa o calendário editorial a partir deste doc.
> - **Doc 07 (Tom de Voz):** As regras verbais são aplicadas aqui por plataforma.
> - **Doc 04 (ICPs):** Os segmentos definem as plataformas prioritárias.
> - Se a estratégia de social não está aqui, o time de marketing **improvisa sem direção**.

---

### Prompt

```
Atue como **Social Media Strategist** com experiência em SaaS B2B e B2C.

Leia os arquivos informados em INPUT_PATH.

Crie a **Estratégia de Social Media** para o produto **[NOME DO PRODUTO]**. Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Objetivos por Plataforma** — para cada rede: objetivo primário, justificativa, métrica de sucesso.

4. **Plataformas Prioritárias** — rankeadas com justificativa baseada em onde os ICPs estão.

5. **Pilares de Conteúdo (4-6)** — cada: nome, descrição, % do calendário, exemplos de posts, formato, objetivo.

6. **Formatos por Plataforma** — o que funciona em cada rede com frequência recomendada.

7. **Calendário Editorial (modelo semanal)** — tabela: dia × plataforma × pilar × formato.

8. **Tom nas Redes (aplicação do Doc 07)** — linguagem, emojis, hashtags, CTAs por rede.

9. **Hashtags e SEO Social** — lista por pilar. Estratégia de SEO por plataforma.

10. **Métricas por Plataforma** — KPIs primários e secundários. Benchmarks do setor.

11. **O que Nunca Postar** — mínimo 5 itens com justificativa.

12. **Changelog**

**Critérios de qualidade:**

- [ ] Plataformas priorizadas com dados dos ICPs
- [ ] Calendário editorial completo para 1 semana
- [ ] Cada pilar com mínimo 3 exemplos de post
- [ ] Métricas com benchmarks quando disponíveis
- [ ] "O que nunca postar" com mínimo 5 itens
- [ ] Total entre 2.500 e 3.500 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
