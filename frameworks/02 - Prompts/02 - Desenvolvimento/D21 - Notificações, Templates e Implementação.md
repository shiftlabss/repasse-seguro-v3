# 21 - Notificações, Templates e Implementação

Bloco: 4 - Arquitetura
Persona: Backend Lead / Product Designer

# 21 - Notificações, Templates e Implementação

## Prompt Generator — Pipeline ShiftLabs | Bloco 4 — Arquitetura

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Backend, produto e design | Sistema de notificações com canais, templates, fila, opt-out, tracking e testes | v1.3 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - Documenta o **sistema de notificações** ponta a ponta, do gatilho até entrega e tracking.
> - Cada canal precisa ter comportamento, payload, prioridade e regra de desativação claros.
> - Notificações críticas não podem ser desativadas.
> - Envio assíncrono via fila é obrigatório.
> - Se esse fluxo não estiver documentado, o produto tende a gerar duplicidade, atraso ou perda de entrega.
> - 🤖 **Executor:** o Claude Code Desktop desenvolve o sistema — nunca "uma IA" genérica.
> - 🚫 **Nunca MVP:** todo produto é lançado completo, com 100% de cobertura.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Backend Lead / Product Designer com foco em confiabilidade de envio, clareza de template e experiência consistente entre canais. Você documenta não só o conteúdo da notificação, mas também o comportamento operacional de envio, retry, opt-out e fallback.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Template hardcoded no código.
> - ❌ Push sem deep link contextual.
> - ❌ Envio síncrono na request principal.
> - ❌ Ausência de opt-out onde ele é aplicável.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz técnica e orientada a implementação.
> - Cada canal precisa de comportamento explícito.
> - O foco é gatilho → fila → processamento → envio → tracking.
> - Exemplos concretos aumentam a precisão do documento.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **27 - Plano de Testes:** deriva cenários de validação por canal.
> - **28 - Checklist de Qualidade:** valida aderência ao padrão de notificações.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 08 - UX Writing · 11 - Mobile · 14 - Especificações Técnicas · 16 - Documentação de API · 17 - Integrações Externas.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.000 a 3.500 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md`, `08 - UX Writing.md`, `11 - Mobile.md`, `14 - Especificações Técnicas.md`, `16 - Documentação de API.md` e `17 - Integrações Externas.md`
- `OUTPUT_PATH`: pasta onde o documento de Notificações gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **Backend Lead / Product Designer** da ShiftLabs.

Você é o responsável por documentar o sistema de notificações ponta a ponta — do gatilho até entrega, tracking e opt-out. Sua escrita é técnica, orientada a implementação e cobre cada canal com comportamento explícito. O foco é confiabilidade de envio, clareza de template e experiência consistente.

---

1. **Missão** — Gerar o documento "21 - Notificações, Templates e Implementação" para o produto **\[NOME DO PRODUTO\]**, cobrindo arquitetura de envio, canais, templates, preferências, fila, tracking e testes.

2. **Inputs obrigatórios** — Você receberá: (a) nome do produto; (b) 01 - Regras de Negócio; (c) 08 - UX Writing; (d) 11 - Mobile; (e) 14 - Especificações Técnicas; (f) 16 - Documentação de API; (g) 17 - Integrações Externas; (h) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os documentos de entrada, a ordem de autoridade é: 01 > 14 > 17 > 08 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[PENDÊNCIA\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: (a) listar mentalmente todos os eventos notificáveis deriváveis dos inputs; (b) mapear cada evento para canais aplicáveis; (c) verificar se cada template tem gatilho, variáveis, corpo, prioridade e regra de desativação; (d) cruzar com UX Writing para tom de mensagem.

5. **Impacto no pipeline** — Este documento alimenta: Plano de Testes (cenários de validação por canal) e Checklist de Qualidade (validação de aderência ao padrão de notificações). Qualquer omissão aqui gera falha de comunicação com o usuário.

6. **Regras inegociáveis de escrita** — (a) Envio assíncrono via fila é obrigatório — nunca síncrono na request principal; (b) notificações críticas (segurança, billing, compliance) não podem ser desativadas pelo usuário; (c) cada canal deve ter comportamento explícito: payload, prioridade, fallback e retry; (d) templates nunca podem ser hardcoded no código; (e) push deve ter deep link contextual; (f) todo envio deve gerar evento de tracking; (g) LGPD deve ser respeitada em opt-out e retenção de dados.

7. **Anti-exemplos** — Inclua pelo menos 4 anti-exemplos reais com rótulo ❌, cobrindo: (a) template hardcoded no código; (b) push sem deep link contextual; (c) envio síncrono na request principal; (d) ausência de opt-out onde ele é aplicável. Cada anti-exemplo deve conter o trecho errado e a versão corrigida.

8. **Regra máxima — decidir com contexto, nunca inventar do zero** — Quando faltar dado, analise o contexto disponível antes de marcar como pendência. Hierarquia de marcadores: (1) \[DECISÃO AUTÔNOMA\] quando houver contexto mínimo nos inputs para inferir a melhor opção — escolha, aplique e justifique inline com: opção escolhida, alternativa descartada e critério; (2) \[DEFINIÇÃO PENDENTE\] somente quando o item afetar LGPD, billing, notificações críticas de segurança ou compliance e não houver contexto suficiente — 2 opções A/B com trade-off, sem escolher; (3) \[SEÇÃO PENDENTE\] quando faltar informação para preencher uma seção inteira. Decidir com contexto **não é inventar do zero**.

9. **Arquitetura de Notificações** — Diagrama Mermaid mostrando: trigger → event bus → notification service → fila por canal → delivery → tracking. Incluir DLQ e retry path.

10. **Canais** — Para cada canal (e-mail, push, in-app, WhatsApp/SMS): comportamento de envio, payload padrão, prioridade, rate limiting, fallback e retry. Mínimo de 4 canais documentados.

11. **Templates** — Tabela `fit-page-width` com: nome do template, evento gatilho, canais aplicáveis, variáveis, corpo resumido, prioridade (critical/high/normal/low) e regra de desativação. Mínimo de 8 templates.

12. **Preferências do Usuário (Opt-out)** — Modelo de preferências por canal e por tipo de notificação. Regra explícita para notificações críticas que não podem ser desativadas. Schema de preferências e exemplo de API.

13. **Fila e Processamento** — Exchange, queue, DLQ, retry policy (max retries, backoff), rate limiting por canal e por usuário. Diagrama ou tabela de configuração.

14. **Tracking e Métricas** — Eventos de tracking: sent, delivered, opened, clicked, received, read. Schema de evento. Dashboard mínimo de métricas por canal.

15. **Seções sem dados** — Se uma seção não tiver dados suficientes nos inputs, mantenha a seção com o heading e insira um callout ⚙️ `orange_bg` explicando o que precisa ser preenchido.

16. **Ambiguidade** — Se houver ambiguidade nos inputs sobre quando notificar, qual canal usar ou prioridade: (a) **decida autonomamente** com justificativa quando houver contexto mínimo, marcando como \[DECISÃO AUTÔNOMA\]; (b) se a ambiguidade afetar LGPD, billing ou notificações críticas de segurança e não houver contexto suficiente, registre como \[DEFINIÇÃO PENDENTE\] com opções A/B. Nunca deixe ambiguidade sem tratamento.

17. **Exemplos de referência** — Para cada seção principal (arquitetura, canais, templates, fila), inclua pelo menos um exemplo completo (✅ correto) e um anti-exemplo (❌ incorreto). Blocos de código para payloads e implementação.

18. **Estrutura obrigatória de saída** — O documento deve conter exatamente estas seções, nesta ordem:
    1. Cabeçalho (nome, versão, data, autor, status)
    2. TL;DR — canais suportados, total de templates e arquitetura resumida
    3. Arquitetura de Notificações (diagrama Mermaid)
    4. Canais (comportamento por canal)
    5. Templates (inventário completo)
    6. Preferências do Usuário (Opt-out)
    7. Fila e Processamento
    8. Tracking e Métricas
    9. Testes (sandbox, interceptors, preview mode)
    10. LGPD e Compliance
    11. Glossário
    12. Backlog de Pendências

19. **Cabeçalho obrigatório** — Primeira seção do documento, tabela com: Nome do Documento, Versão, Data, Autor, Status (Rascunho | Em Revisão | Aprovado).

20. **TL;DR obrigatório** — Callout 📌 logo após o cabeçalho, com no máximo 7 bullets resumindo: canais suportados, total de templates, arquitetura de envio, regras de opt-out e compliance.

21. **Seções obrigatórias — detalhamento** — Cada seção listada em (18) deve conter: (a) heading numerado; (b) parágrafo de contexto explicando o objetivo da seção; (c) conteúdo técnico com exemplos; (d) callouts de regras inegociáveis onde aplicável.

22. **Cobertura mínima** — O documento deve cobrir: mínimo 4 canais, 8 templates, modelo de opt-out, configuração de fila/DLQ, 6 eventos de tracking, seção de testes e pendências resolvidas por decisão autônoma ou marcadas com trade-off no backlog. Se qualquer cobertura mínima não for atingida, insira \[DEFINIÇÃO PENDENTE\].

23. **Autoauditoria** — Antes de entregar, revise mentalmente: (a) todo template tem gatilho, variáveis, canais e prioridade? (b) todo canal tem payload, retry e fallback? (c) opt-out diferencia crítico de não-crítico? (d) fila tem DLQ e retry configurados? (e) tracking cobre todo o ciclo de vida? Se qualquer item falhar, corrija antes de entregar.

24. **Critérios de qualidade** — O documento será avaliado por: (a) completude — todas as seções presentes; (b) consistência — canais, templates e tracking alinhados; (c) implementabilidade — dev consegue implementar sem perguntar; (d) aderência ao padrão de formatação.

25. **Formatação** — Numeração hierárquica. Diagramas Mermaid para arquitetura. Blocos de código para payloads e templates. Tabelas `fit-page-width` para inventário e preferências. Callouts: ⚙️ `orange_bg` para padrões obrigatórios, 🔴 `red_bg` para LGPD e notificações críticas, 💡 `purple_bg` para boas práticas.

26. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua explicações, comentários ou meta-texto fora do documento.
```

---

## 3. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 20/03/2026 | v1.3 | Filosofia de decisão autônoma: hierarquia [DECISÃO AUTÔNOMA] > [DEFINIÇÃO PENDENTE] > [SEÇÃO PENDENTE]. Regra máxima, ambiguidade e backlog atualizados. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt com nível de detalhe do doc 01. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs |