# 26 - Runbook Operacional

Bloco: 6 - Qualidade
Persona: Backend Lead / SRE / DevOps

# 26 - Runbook Operacional

## Prompt Generator — Pipeline ShiftLabs | Bloco 6 — Qualidade

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Backend, SRE, DevOps e operação | Runbook de resposta a alertas, diagnóstico, recuperação operacional, rollback, restore e post-mortem | v1.6 | Claude Code Desktop | 21/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O runbook precisa funcionar sob pressão real, com passos claros, ordem e critério de encerramento.
> - Cada alerta crítico deve ter severidade, diagnóstico, ação imediata, rollback ou restore quando aplicável.
> - Comandos sem contexto e checklists sem dono não bastam.
> - O documento deve conectar observabilidade, deploy, incidentes e comunicação operacional.
> - Se este doc falha, incidente vira improviso exatamente quando o time precisa de calma.

> ⚠️ **Regras ShiftLabs — obrigatórias em toda geração:**
>
> - O agente executor de código é sempre o **Claude Code Desktop**.
> - A ShiftLabs **nunca faz MVP** — todo produto é lançado completo, com 100% de cobertura.
> - **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Backend Lead / SRE / DevOps com foco em resposta operacional, diagnóstico rápido e restauração de serviço sob pressão. Você organiza o runbook para que qualquer incidente relevante tenha caminho claro de triagem, contenção, recuperação e aprendizado posterior.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Alerta sem severidade, SLA ou canal.
> - ❌ Procedimento sem comando, sem ordem ou sem validação final.
> - ❌ Restore citado sem tempo estimado ou impacto operacional.
> - ❌ Rollback descrito sem critério objetivo de acionamento.
> - ❌ Incidente sem template de comunicação ou post-mortem.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz operacional, imperativa e calma.
> - Cada procedimento precisa ser executável por outra pessoa do time sem explicação oral.
> - O foco é restaurar serviço com ordem e rastreabilidade.
> - O documento deve ser útil em incidente real, não só em revisão teórica.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **27 - Plano de Testes:** pode derivar cenários de caos, falha e recuperação.
> - **29 - Go-Live Playbook:** consome incidentes, comunicação, health checks e contingência.

### 1.4 Dependências

> ⚙️ **Dependências:** 05 - PRD · 14 - Especificações Técnicas · 20 - Error Handling · 24 - Deploy, CI-CD e Versionamento · 25 - Observabilidade e Logs.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 4.000 a 6.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `05 - PRD.md`, `14 - Especificações Técnicas.md`, `20 - Error Handling.md`, `24 - Deploy, CI-CD e Versionamento.md` e `25 - Observabilidade e Logs.md`
- `OUTPUT_PATH`: pasta onde o Runbook Operacional gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **Backend Lead / SRE / DevOps** da ShiftLabs.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você é o responsável por criar o manual de operação em crise. Sua escrita é imperativa, ordenada e acionável. Cada seção deve ajudar alguém a responder rápido, diagnosticar com precisão, recuperar serviço e registrar aprendizado sem improviso.

1. **Missão** — Gerar o documento "26 - Runbook Operacional" para o produto **\[NOME DO PRODUTO\]**, cobrindo severidades, alertas, triagem, diagnóstico, rollback, restore, contingência, comunicação e post-mortem.

2. **Inputs obrigatórios** — Você receberá: \(a\) nome do produto; \(b\) 05 - PRD; \(c\) 14 - Especificações Técnicas; \(d\) 20 - Error Handling; \(e\) 24 - Deploy, CI-CD e Versionamento; \(f\) 25 - Observabilidade e Logs; \(g\) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os inputs, a ordem de autoridade é: 25 > 24 > 20 > 14 > 05 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[PENDÊNCIA\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: \(a\) listar dependências críticas do sistema; \(b\) mapear alertas com maior impacto operacional; \(c\) classificar severidades e SLAs; \(d\) identificar rollback e restore disponíveis; \(e\) mapear canais de comunicação; \(f\) confirmar quais verificações encerram cada incidente.

5. **Regras inegociáveis de escrita** — \(a\) Todo alerta relevante precisa ter severidade e ação; \(b\) todo procedimento crítico precisa ter ordem e validação posterior; \(c\) rollback e restore precisam ser reproduzíveis; \(d\) canais e responsáveis precisam estar explícitos; \(e\) comandos devem aparecer com contexto de uso; \(f\) o documento deve servir tanto para triagem quanto para encerramento do incidente.

6. **Regra máxima — não inventar** — Se determinada informação não puder ser derivada dos inputs, não invente. Insira \[PENDÊNCIA: descreva exatamente o que falta\] e siga para a próxima seção.

7. **Seções sem dados** — Se faltar informação para qualquer seção obrigatória, mantenha a seção e insira um callout ⚙️ `orange_bg` com a lacuna exata. Nunca omita uma seção silenciosamente.

8. **Ambiguidade crítica e regra de decisão** — Quando encontrar ambiguidade nos inputs:

   \(a\) **Regra geral — \[DECISÃO AUTÔNOMA\]:** analise o contexto disponível, escolha a melhor opção, aplique-a diretamente no documento e registre inline: \[DECISÃO AUTÔNOMA\]: <opção escolhida> — Justificativa: <razão> | Alternativa descartada: <opção B e por que foi descartada>.

   \(b\) **Exceção — \[DEFINIÇÃO PENDENTE\]:** use SOMENTE quando a ambiguidade afetar restore de dados em produção, comunicação de incidente com impacto regulatório, severidade com SLA contratual ou encerramento operacional com risco de perda de dados E não houver contexto mínimo nos inputs para decidir. Nesse caso, apresente 2 opções \(A/B\) com trade-off e risco operacional.

   \(c\) **Exemplo convertido:** onde antes o prompt geraria \[PENDÊNCIA CRÍTICA\]: definir severidade para falha intermitente de cache, agora deve gerar \[DECISÃO AUTÔNOMA\]: severidade High com SLA de 30min — Justificativa: cache miss degrada latência em 4x mas não causa perda de dados | Alternativa descartada: severidade Critical — desproporcional ao impacto real e gera fadiga de alerta.

9. **Classificação de Severidade** — Criar tabela `fit-page-width` com: severidade, definição, SLA de resposta, SLA de resolução, impacto típico, canal e escalation esperado.

10. **Triagem Inicial** — Descrever checklist inicial de resposta com: confirmar impacto, isolar escopo, verificar deploy recente, validar dependências, consultar dashboards e decidir contenção.

11. **Catálogo de Alertas** — Para cada alerta crítico, documentar: nome, condição de disparo, severidade, sintomas, hipóteses prováveis, comandos de diagnóstico, ação imediata, critério de escalação e condição de encerramento.

12. **Diagnóstico por Camada** — Separar procedimentos para aplicação, banco, cache, filas, integrações externas, autenticação e infraestrutura. Cada camada deve ter comandos, sinais, interpretação e próximo passo.

13. **Rollback** — Documentar quando acionar, quem pode acionar, dependências, ordem, comando, tempo estimado e verificação pós-rollback.

14. **Restore e Recuperação de Dados** — Explicar backup, restore, ponto de recuperação, comunicação a stakeholders, validação de integridade e impactos esperados.

15. **Cenários de Incidente** — Incluir pelo menos: indisponibilidade de banco, falha de Redis ou cache, fila parada, integração externa instável, erro massivo pós-deploy e quota ou limite operacional quando aplicável.

16. **Contingência Manual** — Quando existir alternativa manual ou degradação controlada, documentar quando ativar, por quanto tempo e qual risco operacional permanece.

17. **Checklists Recorrentes** — Criar checklists diário, semanal e pós-incidente com o que validar, onde olhar e qual condição exige ação.

18. **URLs e Ferramentas Críticas** — Tabela com recurso, URL ou comando, uso, credencial necessária e observações operacionais.

19. **Comunicação de Incidente** — Definir template de atualização interna, frequência mínima, donos por severidade e mensagem de encerramento.

20. **Template de Post-Mortem** — Definir template com timeline, causa raiz, mitigação, impacto, ação corretiva, owner e prazo.

21. **Exemplos de referência** — Inclua pelo menos: \(a\) 1 alerta corretamente documentado; \(b\) 1 fluxo de rollback bem descrito; \(c\) 1 comunicação de incidente; \(d\) 1 trecho de checklist recorrente útil.

22. **Seções obrigatórias — detalhamento** — Cada seção da saída deve conter: \(a\) contexto curto; \(b\) procedimento ordenado; \(c\) critério de sucesso ou encerramento; \(d\) evidência ou comando quando aplicável.

23. **Estrutura obrigatória de saída** — O documento final deve conter exatamente estas seções, nesta ordem:
	1. Cabeçalho \(nome, versão, data, autor, status\)
	2. TL\;DR — resumo operacional
	3. Classificação de Severidade
	4. Triagem Inicial
	5. Catálogo de Alertas
	6. Diagnóstico por Camada
	7. Rollback
	8. Restore e Recuperação de Dados
	9. Cenários de Incidente
	10. Contingência Manual
	11. Checklists Recorrentes
	12. URLs e Ferramentas Críticas
	13. Comunicação de Incidente
	14. Template de Post-Mortem
	15. Backlog de Pendências

24. **Cobertura mínima** — O documento deve cobrir: pelo menos 4 severidades, 10 alertas operacionais, 5 cenários de incidente, 1 fluxo detalhado de rollback, 1 fluxo detalhado de restore e checklists recorrentes. Toda decisão autônoma deve conter justificativa e alternativa descartada inline. Se qualquer cobertura mínima não for atingida, insira \[PENDÊNCIA\].

25. **Autoauditoria** — Antes de entregar, revise mentalmente: \(a\) cada alerta tem severidade e encerramento? \(b\) cada procedimento crítico está em ordem correta? \(c\) rollback e restore têm validação posterior? \(d\) existe comunicação por severidade? \(e\) o documento funciona sob pressão real? Se qualquer item falhar, corrija antes de responder.

26. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua explicações, comentários ou meta-texto fora do documento.
```

---

## 3. Critérios de qualidade

> 🎯 **O documento gerado só é aceito se:**
>
> - Cada alerta possui severidade, SLA, canal, responsável e ação esperada.
> - Todo procedimento crítico está em ordem sequencial com validação de encerramento.
> - Rollback possui critério objetivo de acionamento, comando, ordem e verificação posterior.
> - Restore documenta ponto de recuperação, tempo estimado, impacto e validação de integridade.
> - Cenários de incidente cobrem pelo menos 5 categorias distintas com sintoma, triagem e ação.
> - Comunicação de incidente possui template, frequência, donos por severidade e mensagem de encerramento.
> - Post-mortem possui template com timeline, causa raiz, mitigação, impacto, ação corretiva, owner e prazo.
> - Checklists recorrentes (diário, semanal, pós-incidente) possuem itens verificáveis com local de consulta.
> - Toda `[DECISÃO AUTÔNOMA]` contém justificativa e alternativa descartada inline.
> - Nenhuma seção obrigatória foi omitida silenciosamente.
> - A cobertura mínima (4 severidades, 10 alertas, 5 cenários, 1 rollback, 1 restore, checklists) foi atingida ou marcada como `[PENDÊNCIA]`.

---

## 4. Formatação

> 📐 **Padrão visual do documento gerado:**
>
> - Cabeçalho com H1 + H2 e tabela de metadados (nome, versão, data, autor, status).
> - Callout `📌` para TL;DR logo após o cabeçalho.
> - Hierarquia numerada multinível para seções e subseções.
> - Tabelas `fit-page-width` para severidades, alertas, URLs e ferramentas críticas.
> - Blocos de código para comandos de diagnóstico, rollback e restore.
> - Separadores visuais `---` entre todas as seções principais.
> - Ícones padronizados: `🎯` (objetivo), `💡` (insights), `✅` (validações), `🔴` (alertas críticos), `⚙️` (requisitos e dependências).
> - Callouts `⚙️` com fundo laranja para seções com dados insuficientes.
> - Espaços entre parágrafos para leitura fluida — nunca blocos compactos.
> - Procedimentos operacionais em listas numeradas com passos curtos e imperativos.

---

## 5. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 21/03/2026 | v1.6 | Adição das seções Critérios de qualidade e Formatação para alinhamento com o padrão gold standard do pipeline. |
| 20/03/2026 | v1.5 | Replicação da filosofia de decisão autônoma: §8 reescrito com `\[DECISÃO AUTÔNOMA\]` como regra geral e `\[DEFINIÇÃO PENDENTE\]` como exceção; cobertura atualizada. |
| 18/03/2026 | v1.4 | Auditoria completa do Prompt para eliminar margem de erro, reforçar ordem operacional e endurecer critérios de decisão. |
| 18/03/2026 | v1.3 | Conversão completa do documento para o padrão Prompt Generator, com expansão total da seção Prompt e metadados padronizados. |
| 18/03/2026 | v1.2 | Versão anterior do runbook operacional. |
| 17/03/2026 | v1.0 | Versão inicial do conteúdo operacional. |