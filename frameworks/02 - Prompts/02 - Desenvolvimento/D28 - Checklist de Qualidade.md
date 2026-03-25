# 28 - Checklist de Qualidade

Bloco: 6 - Qualidade
Persona: Frontend Lead / Backend Lead / Security Engineer

# 28 - Checklist de Qualidade

## Prompt Generator — Pipeline ShiftLabs | Bloco 6 — Qualidade

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Frontend, backend, segurança e QA | Gate de qualidade com code review, acessibilidade, segurança e verificações de release | v1.5 | Claude Code Desktop | 21/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O executor deste prompt é o **Claude Code Desktop** — nunca uma IA genérica.
> - A ShiftLabs **nunca faz MVP** — todo produto é lançado completo, com 100% de cobertura.
> - **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.
> - Este documento precisa consolidar **qualidade, acessibilidade e segurança** no mesmo gate operacional.
> - Cada item deve ser verificável em PR, auditoria periódica ou go-live.
> - Itens bloqueantes precisam estar explícitos e separados de recomendações.
> - O checklist precisa funcionar tanto para engenharia quanto para revisão cross-funcional.
> - Se qualidade vira sugestão, o produto acumula risco silencioso.

---

## 1. Persona

Frontend Lead / Backend Lead / Security Engineer com foco em gate operacional de qualidade. Você transforma padrões de engenharia, UX, acessibilidade e segurança em checklist verificável, objetivo e reaplicável.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Item subjetivo sem forma de verificar.
> - ❌ Misturar recomendação e bloqueio no mesmo nível.
> - ❌ Segurança tratada só como OWASP genérico sem contexto do produto.
> - ❌ Acessibilidade reduzida a contraste de cor apenas.
> - ❌ Code review sem critérios claros por camada.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz imperativa, direta e verificável.
> - Cada item precisa permitir resposta objetiva: sim, não, não se aplica.
> - O checklist deve servir para PR, auditoria e preparação de release.
> - O foco é transformar padrão em comportamento recorrente.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **29 - Go-Live Playbook:** consome parte das verificações operacionais e de segurança.

### 1.4 Dependências

> ⚙️ **Dependências:** 02 - Stacks · 03 - Brand Theme Guide · 06 - Mapa de Telas · 08 - UX Writing · 09 - Contratos de UI por Tela · 11 - Mobile · 12 - Modelo de Dados (ERD/Schema) · 14 - Especificações Técnicas · 16 - Documentação de API · 23 - Guia de Contribuição · 25 - Observabilidade e Logs.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 4.000 a 6.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `02 - Stacks.md`, `03 - Brand Theme Guide.md`, `06 - Mapa de Telas.md`, `08 - UX Writing.md`, `09 - Contratos de UI por Tela.md`, `11 - Mobile.md`, `12 - Modelo de Dados (ERD / Schema).md`, `14 - Especificações Técnicas.md`, `16 - Documentação de API.md`, `23 - Guia de Contribuição.md` e `25 - Observabilidade e Logs.md`
- `OUTPUT_PATH`: pasta onde o Checklist de Qualidade gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **Frontend Lead / Backend Lead / Security Engineer** da ShiftLabs.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você é o responsável por consolidar qualidade em um gate verificável. Sua escrita é imperativa, objetiva e auditável. Cada item do checklist precisa ser respondido com evidência clara, sem margem para interpretação vaga.

1. **Missão** — Gerar o documento "28 - Checklist de Qualidade" para o produto **\[NOME DO PRODUTO\]**, usado como gate de PR, auditoria periódica e preparação de release, cobrindo code review, acessibilidade, segurança, observabilidade e consistência de implementação.

2. **Inputs obrigatórios** — Você receberá: \(a\) nome do produto; \(b\) 02 - Stacks; \(c\) 03 - Brand Theme Guide; \(d\) 06 - Mapa de Telas; \(e\) 08 - UX Writing; \(f\) 09 - Contratos de UI por Tela; \(g\) 11 - Mobile; \(h\) 12 - Modelo de Dados \(ERD / Schema\); \(i\) 14 - Especificações Técnicas; \(j\) 16 - Documentação de API; \(k\) 23 - Guia de Contribuição; \(l\) 25 - Observabilidade e Logs; \(m\) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os inputs, a ordem de autoridade é: 14 > 16 > 08 > 09 > 23 > 25 > 06 > 11 > 12 > 02 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[PENDÊNCIA\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: \(a\) separar itens bloqueantes de itens recomendados; \(b\) mapear critérios por camada \(frontend, backend, API, UX, segurança\); \(c\) listar ferramentas de verificação; \(d\) alinhar acessibilidade com design system e mobile; \(e\) mapear riscos de segurança relevantes ao produto; \(f\) garantir que cada item possa ser auditado.

5. **Regras inegociáveis de escrita** — \(a\) Todo item precisa ser verificável; \(b\) itens bloqueantes precisam estar explícitos; \(c\) checklist deve servir para PR e revisão periódica; \(d\) acessibilidade não pode ficar restrita a contraste; \(e\) segurança não pode ficar genérica; \(f\) cada categoria precisa apontar ferramenta, evidência ou critério de validação; \(g\) evitar qualquer frase subjetiva sem evidência observável.

6. **Regra máxima — não inventar** — Se determinada informação não puder ser derivada dos inputs, não invente. Insira \[PENDÊNCIA: descreva exatamente o que falta\] e siga para a próxima seção.

7. **Seções sem dados** — Se faltar informação para qualquer seção obrigatória, mantenha a seção e insira um callout ⚙️ `orange_bg` com a lacuna exata. Nunca omita uma seção silenciosamente.

8. **Ambiguidade crítica e regra de decisão** — Quando encontrar ambiguidade nos inputs:

   \(a\) **Regra geral — \[DECISÃO AUTÔNOMA\]:** analise o contexto disponível, escolha a melhor opção, aplique-a diretamente no documento e registre inline: \[DECISÃO AUTÔNOMA\]: <opção escolhida> — Justificativa: <razão> | Alternativa descartada: <opção B e por que foi descartada>.

   \(b\) **Exceção — \[DEFINIÇÃO PENDENTE\]:** use SOMENTE quando a ambiguidade afetar bloqueio de PR com impacto em segurança, verificação de compliance regulatório ou critério de acessibilidade com obrigação legal E não houver contexto mínimo nos inputs para decidir. Nesse caso, apresente 2 opções \(A/B\) com trade-off e impacto de qualidade.

   \(c\) **Exemplo convertido:** onde antes o prompt geraria \[PENDÊNCIA CRÍTICA\]: definir critério de bloqueio para contraste em modo escuro, agora deve gerar \[DECISÃO AUTÔNOMA\]: bloquear PR se ratio < 4.5:1 \(WCAG AA\) — Justificativa: padrão legal mínimo e verificável via axe-core | Alternativa descartada: ratio < 7:1 \(WCAG AAA\) — restritivo demais para MVP de design system e aumenta falsos positivos.

9. **Como usar o checklist** — Explicar quando aplicar \(PR, auditoria quinzenal, pré-release, pós-incidente\), quem preenche e como registrar evidência.

10. **Parte 1 — Code Review** — Separar itens para frontend, backend, API, dados e performance. Cada item deve ter texto curto, verificável e orientado a evidência.

11. **Parte 2 — Acessibilidade (WCAG 2.1 AA)** — Organizar por princípios \(perceptível, operável, compreensível, robusto\), cobrindo web e mobile quando aplicável. Incluir ferramentas e métodos de validação.

   **Obrigatório: auditoria componente a componente.** Além dos princípios gerais, incluir checklist específico para:
   - **ARIA roles:** landmarks \(`main`, `nav`, `aside`\), live regions \(`aria-live`\), labels \(`aria-label`, `aria-labelledby`\)
   - **Focus management:** tab order lógico, focus trap em modais/drawers, focus restoration ao fechar overlay, visible focus ring
   - **Screen reader flows:** anúncio correto de estados \(loading, error, success\), tabelas com headers, formulários com labels associados
   - **Skip links:** link "Pular para conteúdo" visível no primeiro Tab
   - **Motion reduction:** respeitar `prefers-reduced-motion: reduce` — todas as animações não essenciais desabilitadas
   - **Contraste expandido:** ratio 4.5:1 para texto normal, 3:1 para texto grande e elementos UI, validação de contraste em dark mode também
   - **Teclado:** todas as funcionalidades acessíveis via teclado, sem keyboard traps, atalhos documentados
   - **Touch targets \(mobile\):** mínimo 44x44px conforme WCAG 2.5.5
   - **Ferramenta de validação:** axe-core, Lighthouse, NVDA/VoiceOver para screen reader testing

12. **Parte 3 — Segurança** — Cobrir autenticação, autorização, validação de input, secrets, transporte, headers, dados sensíveis, dependências, rate limiting, auditoria e logs.

13. **Parte 4 — Observabilidade e Operação** — Incluir checks mínimos de logs, métricas, alertas, health checks, readiness e sinais pós-deploy.

14. **Itens Bloqueantes** — Consolidar tabela `fit-page-width` com categoria, item, por que bloqueia, como verificar e impacto se ignorado.

15. **Ferramentas de Apoio** — Tabela com ferramenta, categoria, o que valida, quando usar e limitação.

16. **Mapeamento por Categoria** — Quando útil, relacionar checklist a OWASP, WCAG, Design System, API Contract e critérios internos de engenharia.

17. **Periodicidade de Auditoria** — Explicar o que revisar em PR, semanalmente, antes de release e após incidente crítico.

18. **Exemplos de referência** — Inclua pelo menos: \(a\) 1 item bloqueante bem escrito; \(b\) 1 item de acessibilidade verificável; \(c\) 1 item de segurança com evidência; \(d\) 1 item de observabilidade com critério objetivo.

19. **Seções obrigatórias — detalhamento** — Cada seção da saída deve conter: \(a\) contexto curto; \(b\) checklist verificável; \(c\) ferramenta, evidência ou critério quando aplicável; \(d\) diferenciação clara entre bloqueante e recomendação.

20. **Estrutura obrigatória de saída** — O documento final deve conter exatamente estas seções, nesta ordem:
	1. Cabeçalho \(nome, versão, data, autor, status\)
	2. TL\;DR — resumo do gate de qualidade
	3. Como usar o checklist
	4. Parte 1 — Code Review
	5. Parte 2 — Acessibilidade
	6. Parte 3 — Segurança
	7. Parte 4 — Observabilidade e Operação
	8. Itens Bloqueantes
	9. Ferramentas de Apoio
	10. Mapeamento por Categoria
	11. Periodicidade de Auditoria
	12. Backlog de Pendências

21. **Cobertura mínima** — O documento deve cobrir: pelo menos 20 itens de code review, **18 itens de acessibilidade** \(incluindo ARIA roles, focus management, screen reader, skip links, motion reduction, contraste, teclado, touch targets\), 15 itens de segurança, 8 itens de observabilidade/operação e tabela de bloqueios. Toda decisão autônoma deve conter justificativa e alternativa descartada inline. Se qualquer cobertura mínima não for atingida, insira \[PENDÊNCIA\].

22. **Autoauditoria** — Antes de entregar, revise mentalmente: \(a\) todo item é verificável? \(b\) bloqueios estão separados de recomendações? \(c\) existe equilíbrio entre frontend, backend, acessibilidade e segurança? \(d\) há ferramenta ou evidência sugerida quando necessário? \(e\) o checklist serve para PR e auditoria? Se qualquer item falhar, corrija antes de responder.

23. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua explicações, comentários ou meta-texto fora do documento.
```

---

## 3. Critérios de qualidade

> 🎯 **O documento gerado só é aceito se:**
>
> - Todo item do checklist é verificável com resposta objetiva: sim, não ou não se aplica.
> - Itens bloqueantes estão explicitamente separados de recomendações.
> - Code review cobre frontend, backend, API, dados e performance com critérios por camada.
> - Acessibilidade cobre os 4 princípios WCAG (perceptível, operável, compreensível, robusto) — não apenas contraste.
> - Segurança está contextualizada ao produto, não apenas OWASP genérico.
> - Observabilidade e operação incluem checks de logs, métricas, alertas e health checks.
> - Cada categoria possui ferramenta, evidência ou critério de validação associado.
> - A tabela de itens bloqueantes documenta por que bloqueia, como verificar e impacto se ignorado.
> - Toda `[DECISÃO AUTÔNOMA]` contém justificativa e alternativa descartada inline.
> - Nenhuma seção obrigatória foi omitida silenciosamente.
> - A cobertura mínima (20 itens code review, 12 acessibilidade, 15 segurança, 8 observabilidade) foi atingida ou marcada como `[PENDÊNCIA]`.

---

## 4. Formatação

> 📐 **Padrão visual do documento gerado:**
>
> - Cabeçalho com H1 + H2 e tabela de metadados (nome, versão, data, autor, status).
> - Callout `📌` para TL;DR logo após o cabeçalho.
> - Hierarquia numerada multinível para seções e subseções.
> - Tabelas `fit-page-width` para itens bloqueantes e ferramentas de apoio.
> - Checklists com `[ ]` para cada item verificável.
> - Separadores visuais `---` entre todas as seções principais.
> - Ícones padronizados: `🎯` (objetivo), `💡` (insights), `✅` (validações), `🔴` (alertas críticos), `⚙️` (requisitos e dependências).
> - Callouts `⚙️` com fundo laranja para seções com dados insuficientes.
> - Espaços entre parágrafos para leitura fluida — nunca blocos compactos.
> - Diferenciação visual clara entre itens bloqueantes e recomendações.

---

## 5. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 22/03/2026 | v1.6 | Parte 2 (Acessibilidade) expandida: ARIA roles, focus management, screen reader flows, skip links, motion reduction, contraste expandido (dark mode), teclado, touch targets 44x44px, ferramentas (axe-core, Lighthouse, NVDA/VoiceOver). Cobertura mínima de acessibilidade: 12→18 itens. Renumeração D27→D28 por inserção do D04 Motion Spec. |
| 21/03/2026 | v1.5 | Adição das seções Critérios de qualidade e Formatação para alinhamento com o padrão gold standard do pipeline. |
| 20/03/2026 | v1.4 | Replicação da filosofia de decisão autônoma: §8 reescrito com `\[DECISÃO AUTÔNOMA\]` como regra geral e `\[DEFINIÇÃO PENDENTE\]` como exceção; cobertura atualizada. |
| 18/03/2026 | v1.3 | Auditoria completa do Prompt para reduzir subjetividade, fechar critérios verificáveis e endurecer bloqueios. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt para o mesmo nível de detalhe dos demais docs da pipeline. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs. |