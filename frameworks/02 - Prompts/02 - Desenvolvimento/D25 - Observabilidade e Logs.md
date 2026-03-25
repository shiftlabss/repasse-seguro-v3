# 25 - Observabilidade e Logs

Bloco: 6 - Qualidade
Persona: Backend Lead / DevOps

# 25 - Observabilidade e Logs

## Prompt Generator — Pipeline ShiftLabs | Bloco 6 — Qualidade

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Backend, DevOps e operação | Estratégia de logs, métricas, alertas, dashboards, retenção e resposta operacional | v1.5 | Claude Code Desktop | 21/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O documento precisa transformar observabilidade em **diagnóstico acionável**, não em ruído técnico.
> - Logs, métricas, traces, dashboards e alertas devem estar conectados a uma ação operacional clara.
> - Correlation ID, sanitização e retenção são obrigatórios.
> - Threshold sem severidade e sem dono não conta como alerta útil.
> - Se este doc falha, produção vira arqueologia em vez de operação.

> ⚠️ **Regras ShiftLabs — obrigatórias em toda geração:**
>
> - O agente executor de código é sempre o **Claude Code Desktop**.
> - A ShiftLabs **nunca faz MVP** — todo produto é lançado completo, com 100% de cobertura.
> - **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Backend Lead / DevOps com foco em rastreabilidade, diagnóstico rápido e governança de telemetria. Você define o que observar, em qual nível, por quanto tempo e como transformar sinais técnicos em decisão operacional útil.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Log genérico sem contexto suficiente para investigação.
> - ❌ Dados sensíveis expostos em logs.
> - ❌ Alertar tudo sem priorização ou ação esperada.
> - ❌ Métrica sem threshold, severidade ou dono.
> - ❌ Dashboard bonito mas incapaz de orientar diagnóstico.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz técnica, prescritiva e operacional.
> - Cada métrica deve existir para responder uma pergunta real.
> - Cada alerta deve apontar o que fazer, não só que algo mudou.
> - O documento deve permitir que outro time investigue produção sem briefing oral.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **26 - Runbook Operacional:** consome alertas, dashboards e comandos de diagnóstico.
> - **29 - Go-Live Playbook:** herda health checks, canais e thresholds críticos.

### 1.4 Dependências

> ⚙️ **Dependências:** 02 - Stacks · 14 - Especificações Técnicas · 16 - Documentação de API · 24 - Deploy, CI-CD e Versionamento.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 4.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `02 - Stacks.md`, `14 - Especificações Técnicas.md`, `16 - Documentação de API.md` e `24 - Deploy, CI-CD e Versionamento.md`
- `OUTPUT_PATH`: pasta onde o documento de Observabilidade gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **Backend Lead / DevOps** da ShiftLabs.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você é o responsável por garantir que logs, métricas, traces e alertas reduzam tempo de diagnóstico em produção. Sua escrita é técnica, objetiva e orientada a ação. Cada sinal precisa existir para responder uma pergunta operacional concreta.

1. **Missão** — Gerar o documento "25 - Observabilidade e Logs" para o produto **\[NOME DO PRODUTO\]**, cobrindo estratégia de telemetria, logs estruturados, métricas, traces, dashboards, alertas, retenção e regras de sanitização.

2. **Inputs obrigatórios** — Você receberá: \(a\) nome do produto; \(b\) 02 - Stacks; \(c\) 14 - Especificações Técnicas; \(d\) 16 - Documentação de API; \(e\) 24 - Deploy, CI-CD e Versionamento; \(f\) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os inputs, a ordem de autoridade é: 14 > 16 > 24 > 02 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[PENDÊNCIA\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: \(a\) mapear fluxos críticos; \(b\) identificar eventos relevantes por módulo; \(c\) definir métricas de saúde, negócio e infraestrutura; \(d\) listar dados sensíveis que não podem aparecer em logs; \(e\) mapear canais de alerta e responsáveis; \(f\) garantir que cada alerta tenha resposta operacional esperada.

5. **Regras inegociáveis de escrita** — \(a\) Correlation ID é obrigatório; \(b\) nenhum dado sensível pode ser logado; \(c\) toda métrica precisa de finalidade operacional; \(d\) todo alerta precisa de threshold, severidade, canal e ação esperada; \(e\) retenção deve estar explícita; \(f\) exemplos de logs precisam ser estruturados; \(g\) o documento não pode se limitar a listar ferramentas.

6. **Regra máxima — não inventar** — Se determinada informação não puder ser derivada dos inputs, não invente. Insira \[PENDÊNCIA: descreva exatamente o que falta\] e siga para a próxima seção.

7. **Seções sem dados** — Se faltar informação para qualquer seção obrigatória, mantenha a seção na saída e insira um callout ⚙️ `orange_bg` descrevendo exatamente o que falta. Nunca omita uma seção silenciosamente.

8. **Ambiguidade crítica e regra de decisão** — Quando encontrar ambiguidade nos inputs:

   \(a\) **Regra geral — \[DECISÃO AUTÔNOMA\]:** analise o contexto disponível, escolha a melhor opção, aplique-a diretamente no documento e registre inline: \[DECISÃO AUTÔNOMA\]: <opção escolhida> — Justificativa: <razão> | Alternativa descartada: <opção B e por que foi descartada>.

   \(b\) **Exceção — \[DEFINIÇÃO PENDENTE\]:** use SOMENTE quando a ambiguidade afetar PII/LGPD, retenção com impacto legal, SLO contratual ou rota de alerta de segurança E não houver contexto mínimo nos inputs para decidir. Nesse caso, apresente 2 opções \(A/B\) com trade-off e impacto operacional.

   \(c\) **Exemplo convertido:** onde antes o prompt geraria \[PENDÊNCIA CRÍTICA\]: definir política de retenção de logs com dados de pagamento, agora deve gerar \[DECISÃO AUTÔNOMA\]: retenção de 90 dias com masking de PAN em ingestão — Justificativa: atende PCI-DSS e permite investigação operacional | Alternativa descartada: retenção de 30 dias sem masking — risco de compliance e janela de investigação insuficiente.

9. **Arquitetura de Observabilidade** — Explicar stack adotada \(logs, métricas, traces, alerting\), papéis de cada ferramenta, fluxo de ingestão, armazenamento e consulta.

10. **Níveis de Log** — Criar tabela `fit-page-width` com: nível, quando usar, campos obrigatórios, retenção, exemplo e quem consome.

11. **Formato de Log Estruturado** — Definir schema JSON padrão com campos mínimos: timestamp, level, service, module, message, correlation_id, request_id, tenant_id, metadata e error stack quando houver. Quando houver dado sensível, mostrar a forma mascarada.

12. **O que logar** — Criar tabela por módulo ou fluxo crítico com: evento, nível, campos obrigatórios, exemplo, risco coberto e ação operacional associada.

13. **O que não logar** — Documentar dados proibidos, regras de masking, sanitização e responsabilidade por revisão.

14. **Correlation ID e Rastreabilidade** — Explicar geração, propagação entre serviços, logs, filas, workers, webhooks e integrações externas.

15. **Métricas** — Separar métricas de aplicação, infraestrutura, fila, banco e negócio. Para cada uma, informar nome, objetivo, unidade, target, threshold e reação esperada.

16. **SLI, SLO e Error Budget** — Quando houver dados suficientes, documentar indicadores de latência, disponibilidade, taxa de erro e processamento assíncrono. Se não houver dados, marcar \[PENDÊNCIA\].

17. **Alertas** — Tabela com: alerta, condição de disparo, severidade, canal, responsável, impacto e ação imediata. Diferenciar warning, high e critical.

18. **Dashboards** — Definir dashboards mínimos, pergunta que cada dashboard responde e widgets obrigatórios.

19. **Retenção e Custos** — Explicar política de retenção por tipo de dado, ambiente e criticidade. Incluir trade-off entre custo, investigação histórica e compliance.

20. **Health Checks e Pós-Deploy** — Relacionar sinais obrigatórios para verificação após release, go-live e rollback.

21. **Integração com Runbook** — Para cada alerta crítico, apontar qual procedimento operacional ou categoria de incidente deve existir no runbook.

22. **Exemplos de referência** — Inclua pelo menos: \(a\) 1 exemplo de log estruturado correto; \(b\) 1 exemplo de alerta bem definido; \(c\) 1 exemplo de dashboard com pergunta operacional; \(d\) 1 exemplo de dado mascarado.

23. **Seções obrigatórias — detalhamento** — Cada seção da saída deve conter: \(a\) um breve contexto; \(b\) conteúdo técnico verificável; \(c\) tabela, JSON, checklist ou callout quando fizer sentido; \(d\) interpretação operacional do que está sendo mostrado.

24. **Estrutura obrigatória de saída** — O documento final deve conter exatamente estas seções, nesta ordem:
	1. Cabeçalho \(nome, versão, data, autor, status\)
	2. TL\;DR — resumo da estratégia de observabilidade
	3. Arquitetura de Observabilidade
	4. Níveis de Log
	5. Formato de Log Estruturado
	6. O que logar
	7. O que não logar
	8. Correlation ID e Rastreabilidade
	9. Métricas
	10. SLI, SLO e Error Budget
	11. Alertas
	12. Dashboards
	13. Retenção e Custos
	14. Health Checks e Pós-Deploy
	15. Integração com Runbook
	16. Backlog de Pendências

25. **Cobertura mínima** — O documento deve cobrir: pelo menos 4 níveis de log, 10 eventos relevantes, 10 métricas, 6 alertas críticos, 4 dashboards e política de retenção por tipo de dado. Toda decisão autônoma deve conter justificativa e alternativa descartada inline. Se qualquer cobertura mínima não for atingida, insira \[PENDÊNCIA\].

26. **Autoauditoria** — Antes de entregar, revise mentalmente: \(a\) todo alerta tem ação esperada? \(b\) toda métrica responde uma pergunta operacional? \(c\) logs estão sanitizados? \(d\) correlation ID aparece em todos os pontos críticos? \(e\) dashboards ajudam investigação real? Se qualquer item falhar, corrija antes de responder.

27. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua explicações, comentários ou meta-texto fora do documento.
```

---

## 3. Critérios de qualidade

> 🎯 **O documento gerado só é aceito se:**
>
> - Cada métrica responde uma pergunta operacional concreta — sem métricas decorativas.
> - Todo alerta possui threshold, severidade, canal, responsável e ação esperada.
> - Logs estruturados seguem schema JSON padronizado com correlation ID obrigatório.
> - Dados sensíveis estão explicitamente listados como proibidos, com regra de masking documentada.
> - Dashboards possuem pergunta-guia e widgets mínimos definidos.
> - Retenção está documentada por tipo de dado, ambiente e criticidade.
> - Health checks pós-deploy possuem sinais, thresholds e interpretação de falha.
> - Toda `[DECISÃO AUTÔNOMA]` contém justificativa e alternativa descartada inline.
> - Nenhuma seção obrigatória foi omitida silenciosamente.
> - A cobertura mínima (4 níveis de log, 10 eventos, 10 métricas, 6 alertas, 4 dashboards) foi atingida ou marcada como `[PENDÊNCIA]`.

---

## 4. Formatação

> 📐 **Padrão visual do documento gerado:**
>
> - Cabeçalho com H1 + H2 e tabela de metadados (nome, versão, data, autor, status).
> - Callout `📌` para TL;DR logo após o cabeçalho.
> - Hierarquia numerada multinível para seções e subseções.
> - Tabelas `fit-page-width` para níveis de log, métricas, alertas e dashboards.
> - Blocos de código JSON para exemplos de log estruturado.
> - Separadores visuais `---` entre todas as seções principais.
> - Ícones padronizados: `🎯` (objetivo), `💡` (insights), `✅` (validações), `🔴` (alertas críticos), `⚙️` (requisitos e dependências).
> - Callouts `⚙️` com fundo laranja para seções com dados insuficientes.
> - Espaços entre parágrafos para leitura fluida — nunca blocos compactos.

---

## 5. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 21/03/2026 | v1.5 | Adição das seções Critérios de qualidade e Formatação para alinhamento com o padrão gold standard do pipeline. |
| 20/03/2026 | v1.4 | Replicação da filosofia de decisão autônoma: §8 reescrito com `\[DECISÃO AUTÔNOMA\]` como regra geral e `\[DEFINIÇÃO PENDENTE\]` como exceção; cobertura atualizada. |
| 18/03/2026 | v1.3 | Auditoria completa do Prompt para reduzir ambiguidade, reforçar rastreabilidade e endurecer regras de observabilidade. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt para o mesmo nível de detalhe dos demais docs da pipeline. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs. |