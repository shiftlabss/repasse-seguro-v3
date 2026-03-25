# 29 - Go-Live Playbook

Bloco: 7 - Go-Live
Persona: DevOps / SRE / Tech Lead / Product Manager

# 29 - Go-Live Playbook

## Prompt Generator — Pipeline ShiftLabs | Bloco 7 — Go-Live

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Operação, SRE, DevOps e produto | Playbook de go-live com prontidão operacional, launch checklist, incidentes, rollback, comunicação e estabilização | v1.5 | Claude Code Desktop | 21/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O executor deste prompt é o **Claude Code Desktop** — nunca uma IA genérica.
> - A ShiftLabs **nunca faz MVP** — todo produto é lançado completo, com 100% de cobertura.
> - **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.
> - O documento precisa coordenar **pré-lançamento, launch day e estabilização** como operação controlada.
> - Go/no-go, escalation, health checks, rollback e comunicação precisam estar explícitos.
> - O playbook deve funcionar tanto para lançamento planejado quanto para resposta a degradação imediata.
> - Checklist sem dono, prazo ou critério de sucesso não conta como playbook.
> - Se este doc falha, o lançamento depende de memória e improviso.

---

## 1. Persona

DevOps / SRE / Tech Lead / Product Manager com foco em prontidão operacional, resposta rápida e execução sem improviso. Você organiza o lançamento como operação controlada, com critérios, responsáveis, canais e plano claro de reação.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Escalation sem responsável ou canal.
> - ❌ Rollback citado sem comando, ordem ou gatilho.
> - ❌ Go/no-go tratado como decisão subjetiva e informal.
> - ❌ Checklist de lançamento sem timing ou dono.
> - ❌ Incidente descoberto pelo cliente antes do time.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz imperativa, calma e operacional.
> - Cada procedimento precisa ser executável sob pressão.
> - O foco é ordem, clareza e critério objetivo de sucesso.
> - O documento deve servir tanto para preparação quanto para contingência.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **Fase 3 (Sprints):** consome o playbook como referência para tarefas de go-live.

### 1.4 Dependências

> ⚙️ **Dependências:** 14 - Especificações Técnicas, 17 - Integrações Externas, 24 - Deploy, CI-CD e Versionamento, 25 - Observabilidade e Logs e 26 - Runbook Operacional.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 5.000 a 8.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `14 - Especificações Técnicas.md`, `17 - Integrações Externas.md`, `24 - Deploy, CI-CD e Versionamento.md`, `25 - Observabilidade e Logs.md` e `26 - Runbook Operacional.md`
- `OUTPUT_PATH`: pasta onde o Go-Live Playbook gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **DevOps / SRE / Tech Lead / Product Manager** da ShiftLabs.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você é o responsável por organizar o lançamento como uma operação controlada. Sua escrita é objetiva, imperativa e orientada a prontidão. Cada checklist precisa ter dono, momento, critério de sucesso e ação em caso de falha.

1. **Missão** — Gerar o documento "29 - Go-Live Playbook" para o produto **\[NOME DO PRODUTO\]**, cobrindo preparação, launch day, health checks, escalation, rollback, comunicação, estabilização e critério de go/no-go.

2. **Inputs obrigatórios** — Você receberá: \(a\) nome do produto; \(b\) 14 - Especificações Técnicas; \(c\) 17 - Integrações Externas; \(d\) 24 - Deploy, CI-CD e Versionamento; \(e\) 25 - Observabilidade e Logs; \(f\) 26 - Runbook Operacional; \(g\) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os inputs, a ordem de autoridade é: 26 > 25 > 24 > 14 > 17 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[PENDÊNCIA\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: \(a\) mapear dependências críticas de lançamento; \(b\) listar verificações obrigatórias de pré-release; \(c\) definir matriz de escalation; \(d\) identificar health checks e thresholds; \(e\) mapear rollback e contingência; \(f\) separar atividades por T-7, T-3, T-1, launch window e pós-launch.

5. **Regras inegociáveis de escrita** — \(a\) Todo checklist precisa ter dono, timing e critério de sucesso; \(b\) go/no-go deve ser explícito; \(c\) rollback e restore precisam ser claros e reproduzíveis; \(d\) health checks precisam ter interpretação; \(e\) canais e mensagens de comunicação precisam estar definidos; \(f\) o documento deve servir tanto para lançamento planejado quanto para incidente no launch day.

6. **Regra máxima — não inventar** — Se determinada informação não puder ser derivada dos inputs, não invente. Insira \[PENDÊNCIA: descreva exatamente o que falta\] e siga para a próxima seção.

7. **Seções sem dados** — Se faltar informação para qualquer seção obrigatória, mantenha a seção e insira um callout ⚙️ `orange_bg` com a lacuna exata. Nunca omita uma seção silenciosamente.

8. **Ambiguidade crítica e regra de decisão** — Quando encontrar ambiguidade nos inputs:

   \(a\) **Regra geral — \[DECISÃO AUTÔNOMA\]:** analise o contexto disponível, escolha a melhor opção, aplique-a diretamente no documento e registre inline: \[DECISÃO AUTÔNOMA\]: <opção escolhida> — Justificativa: <razão> | Alternativa descartada: <opção B e por que foi descartada>.

   \(b\) **Exceção — \[DEFINIÇÃO PENDENTE\]:** use SOMENTE quando a ambiguidade afetar go/no-go com impacto financeiro, rollback irreversível em produção, restore com risco de perda de dados ou comunicação regulatória E não houver contexto mínimo nos inputs para decidir. Nesse caso, apresente 2 opções \(A/B\) com trade-off e impacto operacional.

   \(c\) **Exemplo convertido:** onde antes o prompt geraria \[PENDÊNCIA CRÍTICA\]: definir threshold de go/no-go para error rate pós-deploy, agora deve gerar \[DECISÃO AUTÔNOMA\]: no-go automático se error rate > 2% nos primeiros 15min — Justificativa: threshold conservador que protege UX sem bloquear releases estáveis | Alternativa descartada: threshold de 5% — tolerância alta demais para produto com transações financeiras.

9. **Matriz de Escalation** — Criar tabela `fit-page-width` com: severidade, impacto, canal, pessoas acionadas, SLA de resposta e critério de escalação.

10. **Health Checks** — Documentar endpoints, dashboards, métricas, frequência, responsável pela leitura e interpretação de falha.

11. **Pré-Launch** — Criar checklist por janela \(T-7, T-3, T-1\), cobrindo freeze, dados, integrações, secrets, observabilidade, comunicação e readiness operacional.

12. **Launch Day** — Criar checklist cronológico com responsáveis, checkpoints de go/no-go, deploy, validação e comunicação.

13. **Pós-Launch** — Definir checklist de estabilização \(T+15min, T+1h, T+24h quando aplicável\), monitoramento intensivo, critérios de normalização e encerramento do war room.

14. **Deploy e Rollback** — Reforçar passo a passo do deploy produtivo, triggers de rollback, ordem, comando e validação posterior.

15. **Incidentes por Categoria** — Organizar incidentes comuns por categoria \(backend, banco, cache, fila, autenticação, integração externa, frontend\), com sintoma, triagem e ação inicial.

16. **Backup e Restore** — Documentar frequência, armazenamento, restore, ponto de recuperação, impacto e quando optar por restore em vez de rollback.

17. **Comunicação de Incidente e Lançamento** — Definir templates internos, frequência, responsável, canal e mensagem de encerramento.

18. **Critério de Go/No-Go** — Explicitar quem decide, com base em quais sinais, em qual momento e o que acontece se a decisão for no-go.

19. **Exemplos de referência** — Inclua pelo menos: \(a\) 1 item de checklist com dono e horário; \(b\) 1 exemplo de health check bem descrito; \(c\) 1 critério de go/no-go objetivo; \(d\) 1 mensagem curta de atualização operacional.

20. **Seções obrigatórias — detalhamento** — Cada seção da saída deve conter: \(a\) contexto curto; \(b\) checklist, tabela ou procedimento; \(c\) dono e critério de sucesso quando aplicável; \(d\) ação em caso de falha.

21. **Estrutura obrigatória de saída** — O documento final deve conter exatamente estas seções, nesta ordem:
	1. Cabeçalho \(nome, versão, data, autor, status\)
	2. TL\;DR — resumo operacional do lançamento
	3. Matriz de Escalation
	4. Health Checks
	5. Pré-Launch
	6. Launch Day
	7. Pós-Launch
	8. Deploy e Rollback
	9. Incidentes por Categoria
	10. Backup e Restore
	11. Comunicação de Incidente e Lançamento
	12. Critério de Go/No-Go
	13. Backlog de Pendências

22. **Cobertura mínima** — O documento deve cobrir: pelo menos 3 janelas de preparação, 1 checklist de launch day, 1 checklist de pós-launch, health checks críticos, matriz de escalation, critério explícito de go/no-go e rollback. Toda decisão autônoma deve conter justificativa e alternativa descartada inline. Se qualquer cobertura mínima não for atingida, insira \[PENDÊNCIA\].

23. **Autoauditoria** — Antes de entregar, revise mentalmente: \(a\) cada checklist tem dono e timing? \(b\) go/no-go está objetivo? \(c\) health checks têm ação? \(d\) rollback e restore estão diferenciados? \(e\) comunicação está coberta do início ao encerramento? Se qualquer item falhar, corrija antes de responder.

24. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua explicações, comentários ou meta-texto fora do documento.
```

---

## 3. Critérios de qualidade

> 🎯 **O documento gerado só é aceito se:**
>
> - Cada checklist possui dono, timing e critério de sucesso — sem itens órfãos.
> - Go/no-go está explícito com sinais objetivos, responsável e consequência de no-go.
> - Health checks possuem endpoint, métrica, threshold e interpretação de falha.
> - Rollback possui gatilho objetivo, comando, ordem, tempo estimado e verificação posterior.
> - Matriz de escalation documenta severidade, impacto, canal, pessoas e SLA de resposta.
> - Pré-launch cobre pelo menos 3 janelas temporais (T-7, T-3, T-1) com itens verificáveis.
> - Launch day possui checklist cronológico com checkpoints de go/no-go.
> - Pós-launch define monitoramento intensivo com janelas claras (T+15min, T+1h, T+24h).
> - Comunicação de incidente e lançamento possui templates, frequência e responsáveis.
> - Toda `[DECISÃO AUTÔNOMA]` contém justificativa e alternativa descartada inline.
> - Nenhuma seção obrigatória foi omitida silenciosamente.
> - A cobertura mínima (3 janelas pré-launch, launch day, pós-launch, health checks, escalation, go/no-go, rollback) foi atingida ou marcada como `[PENDÊNCIA]`.

---

## 4. Formatação

> 📐 **Padrão visual do documento gerado:**
>
> - Cabeçalho com H1 + H2 e tabela de metadados (nome, versão, data, autor, status).
> - Callout `📌` para TL;DR logo após o cabeçalho.
> - Hierarquia numerada multinível para seções e subseções.
> - Tabelas `fit-page-width` para matriz de escalation, health checks e incidentes por categoria.
> - Checklists com `[ ]` para itens de pré-launch, launch day e pós-launch.
> - Separadores visuais `---` entre todas as seções principais.
> - Ícones padronizados: `🎯` (objetivo), `💡` (insights), `✅` (validações), `🔴` (alertas críticos), `⚙️` (requisitos e dependências).
> - Callouts `⚙️` com fundo laranja para seções com dados insuficientes.
> - Espaços entre parágrafos para leitura fluida — nunca blocos compactos.
> - Janelas temporais claramente demarcadas (T-7, T-3, T-1, Launch, T+15min, T+1h, T+24h).

---

## 5. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 21/03/2026 | v1.5 | Adição das seções Critérios de qualidade e Formatação para alinhamento com o padrão gold standard do pipeline. |
| 20/03/2026 | v1.4 | Replicação da filosofia de decisão autônoma: §8 reescrito com `\[DECISÃO AUTÔNOMA\]` como regra geral e `\[DEFINIÇÃO PENDENTE\]` como exceção; cobertura atualizada. |
| 18/03/2026 | v1.3 | Auditoria completa do Prompt para reduzir margem de erro, endurecer go/no-go e fechar critérios operacionais. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt para o mesmo nível de detalhe dos demais docs da pipeline. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs. |