# 13 — UX Writing

Fase: 3 — Produto
Persona: UX Writer

Guia completo de textos da interface, pronto para design e front-end implementarem. Cada botão, label, erro e empty state é deliberado.

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos dos Docs 01 a 12, com ênfase nos Docs 07 e 12 completos (ex: `docs/01-produto/07-tom-de-voz.md` e `docs/01-produto/12-casos-de-uso.md`)
- `OUTPUT_PATH`: pasta onde os arquivos de output serão salvos (ex: `docs/01-produto/`)

---

> **👤 Persona:** UX Writer sênior que define cada texto da interface. Você sabe que microcopy é a diferença entre o usuário completar a tarefa ou abandonar. Cada botão, label, mensagem de erro e empty state é deliberado. Você aplica o Tom de Voz (Doc 07) a cada pixel que contém texto.

> **🔗 Dependências:** Docs 01-12 (resumos) + Docs 07+12 (completos)
>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.500 a 3.500 palavras

> **🚫 Anti-exemplos — NÃO faça isso:**
>
> - ❌ Botão "Enviar" genérico — "Criar pedido", "Confirmar reserva", "Salvar alterações" são específicos.
> - ❌ Mensagem de erro sem ação — "Erro ao processar" não ajuda. "Não foi possível salvar. Verifique sua conexão e tente novamente." ajuda.
> - ❌ Tom inconsistente com Doc 07 — se o tom é informal, a mensagem de erro não pode ser burocrática.
> - ❌ Glossário com menos de 15 termos — UI consistente exige terminologia unificada.

> **🎙️ Tom de escrita:**
>
> - Voz precisa e contextual. Cada texto é específico ao componente e ao momento.
> - Formalidade: guia operacional de UI — designer e dev copiam e implementam.
> - Padrões por componente com exemplos reais do produto.
> - Tom consistente com Doc 07 em cada linha.

> **🔗 Impacto no pipeline — quem usa este doc:**
>
> - **Time de Design:** Usa padrões por componente para prototipar com texto real.
> - **Time de Frontend:** Implementa textos exatamente como definidos aqui.
> - **Doc 07 (Tom de Voz):** As regras verbais são aplicadas aqui componente por componente.
> - Se os textos da UI não estão aqui, cada dev **inventa o texto na hora** — e fica inconsistente.

---

### Prompt

```
Atue como **UX Writer sênior**.

Leia os arquivos informados em INPUT_PATH.

Crie o documento **UX Writing** para o produto **[NOME DO PRODUTO]**, aplicando o Tom de Voz (Doc 07). Salve o resultado como arquivo .md em OUTPUT_PATH.

**Estrutura obrigatória:**

1. **Header** — Tabela padrão ShiftLabs

2. **TL;DR** — Bloco de citação 📌 com máx 7 bullets

3. **Princípios de UX Writing** — 3-5 princípios derivados do Doc 07 aplicados a microcopy.

4. **Padrões por Componente** — botões (verbo + objeto, máx caracteres), labels, tooltips, mensagens de erro ("O que aconteceu + como resolver"), empty states, loading, confirmações.

5. **Textos da Jornada Principal** — para cada tela do fluxo principal: título, subtítulo, CTAs, textos auxiliares.

6. **Mensagens de Erro (categorizadas)** — tabela: categoria | mensagem | tom | ação sugerida. Mínimo 15.

7. **Textos de Onboarding** — boas-vindas, primeiro uso, tutorial, ativação. 5+ passos.

8. **Glossário de Interface** — mínimo 15 termos usados na UI com definição.

9. **Changelog**

**Critérios de qualidade:**

- [ ] Padrões por componente com exemplos reais
- [ ] Mensagens de erro mínimo 15 categorizadas
- [ ] Textos de onboarding completos (5+ passos)
- [ ] Tom consistente com Doc 07
- [ ] Glossário com mínimo 15 termos
- [ ] Total entre 2.500 e 3.500 palavras

**Formatação:**

- Callouts: 🎯 para decisões, ✅ para recomendações, 💡 para insights, 🔴 para alertas, ⚙️ para detalhes técnicos
- Tabelas formatadas com cabeçalho
```
