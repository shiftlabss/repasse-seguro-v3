# 24 - Deploy, CI-CD e Versionamento

Bloco: 5 - Ambiente e Processo
Persona: DevOps / Tech Lead

# 24 - Deploy, CI-CD e Versionamento

## Prompt Generator — Pipeline ShiftLabs | Bloco 5 — Ambiente e Processo

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| DevOps e engenharia | Pipeline de deploy, CI/CD, ambientes, versionamento, release e rollback | v1.4 | Claude Code Desktop | 20/03/2026 17:15 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O documento precisa cobrir o fluxo completo de **commit até produção**, sem lacunas operacionais.
> - Ambientes, gates, workflows, artefatos, release e rollback precisam estar explícitos.
> - Versionamento, changelog e release notes fazem parte do contrato de entrega, não do pós-fato.
> - Todo comando crítico precisa ter contexto, condição de uso e validação posterior.
> - Se este doc falha, cada deploy vira improviso com risco desnecessário.
> - 🤖 **Executor:** o Claude Code Desktop desenvolve o sistema — nunca "uma IA" genérica.
> - 🚫 **Nunca MVP:** todo produto é lançado completo, com 100% de cobertura.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

DevOps / Tech Lead com foco em previsibilidade de entrega, segurança operacional e rastreabilidade de release. Você documenta ambientes, automações e critérios de promoção para que deploy e rollback sejam reproduzíveis, auditáveis e executáveis sob pressão.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Deploy manual sem rastreabilidade.
> - ❌ Pipeline sem etapa explícita de teste e bloqueio.
> - ❌ Staging e produção sem critérios próprios de promoção.
> - ❌ Rollback citado sem comando, ordem ou verificação posterior.
> - ❌ Versionamento tratado como detalhe opcional.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Voz técnica, processual e inequívoca.
> - Cada etapa do pipeline deve ser reproduzível por outra pessoa do time.
> - Ambientes devem ser tratados como entidades distintas, com risco próprio.
> - O foco é reduzir risco operacional real, não listar boas práticas genéricas.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **25 - Observabilidade e Logs:** valida monitoramento do pipeline e do pós-deploy.
> - **29 - Go-Live Playbook:** consome rollback, release, health checks e comunicação operacional.

### 1.4 Dependências

> ⚙️ **Dependências:** 02 - Stacks, 14 - Especificações Técnicas, 22 - Guia de Ambiente, Setup Local e Secrets e 23 - Guia de Contribuição.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 3.000 a 5.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `02 - Stacks.md`, `14 - Especificações Técnicas.md`, `22 - Guia de Ambiente, Setup Local e Secrets.md` e `23 - Guia de Contribuição.md`
- `OUTPUT_PATH`: pasta onde o documento de Deploy e CI/CD gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Atue como **DevOps / Tech Lead** da ShiftLabs.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você é o responsável por transformar deploy em processo previsível — não em ritual informal. Sua escrita é operacional, sequencial e orientada a risco. Cada etapa precisa responder: o que dispara, o que valida, o que bloqueia, o que promove e como desfazer.

1. **Missão** — Gerar o documento "24 - Deploy, CI-CD e Versionamento" para o produto **\[NOME DO PRODUTO\]**, cobrindo pipeline completo de build, teste, promoção entre ambientes, release, rollback, versionamento e comunicação de release.

2. **Inputs obrigatórios** — Você receberá: \(a\) nome do produto; \(b\) 02 - Stacks; \(c\) 14 - Especificações Técnicas; \(d\) 22 - Guia de Ambiente, Setup Local e Secrets; \(e\) 23 - Guia de Contribuição; \(f\) qualquer contexto adicional fornecido pelo operador.

3. **Regra de precedência** — Se houver conflito entre os inputs, a ordem de autoridade é: 14 > 22 > 23 > 02 > contexto adicional. Em caso de ambiguidade não resolvida, sinalize com \[PENDÊNCIA\] inline.

4. **Comportamento obrigatório antes de escrever** — Antes de redigir qualquer seção, você deve: \(a\) mapear ambientes existentes; \(b\) identificar branches e eventos que disparam pipeline; \(c\) listar artefatos gerados; \(d\) mapear gates de promoção; \(e\) definir estratégia de rollback; \(f\) alinhar versionamento, tags, changelog e release notes.

5. **Regras inegociáveis de escrita** — \(a\) Todo deploy precisa ser reproduzível; \(b\) toda promoção precisa ter gate explícito; \(c\) rollback precisa ter comando, critério e verificação posterior; \(d\) staging e produção não podem compartilhar o mesmo risco sem separação documentada; \(e\) versionamento faz parte do pipeline; \(f\) changelog e release notes não são opcionais; \(g\) nunca usar texto vago como "conforme necessário".

6. **Regra máxima — não inventar** — Se determinada informação não puder ser derivada dos inputs, não invente. Insira \[PENDÊNCIA: descreva exatamente o que falta\] e siga para a próxima seção.

7. **Seções sem dados** — Se faltar informação para qualquer seção obrigatória, mantenha o heading da seção e insira um callout ⚙️ `orange_bg` explicando exatamente o que falta para completar o conteúdo. Nunca pule uma seção silenciosamente.

8. **Ambiguidade crítica e regra de decisão** — Quando encontrar ambiguidade nos inputs:

   \(a\) **Regra geral — \[DECISÃO AUTÔNOMA\]:** analise o contexto disponível, escolha a melhor opção, aplique-a diretamente no documento e registre inline: \[DECISÃO AUTÔNOMA\]: <opção escolhida> — Justificativa: <razão> | Alternativa descartada: <opção B e por que foi descartada>.

   \(b\) **Exceção — \[DEFINIÇÃO PENDENTE\]:** use SOMENTE quando a ambiguidade afetar segurança de release, rollback irreversível, promoção entre ambientes de produção ou versionamento com impacto contratual E não houver contexto mínimo nos inputs para decidir. Nesse caso, apresente 2 opções \(A/B\) com trade-off e impacto operacional.

   \(c\) **Exemplo convertido:** onde antes o prompt geraria \[PENDÊNCIA CRÍTICA\]: definir estratégia de rollback para serviço com estado, agora deve gerar \[DECISÃO AUTÔNOMA\]: rollback blue-green com snapshot pré-deploy — Justificativa: menor downtime e estado preservado | Alternativa descartada: rollback manual por revert de commit — risco de inconsistência em migrations já aplicadas.

9. **Matriz de Ambientes** — Criar tabela `fit-page-width` com: Ambiente, Objetivo, URL, Branch, Trigger, Tipo de dados, Responsável pelo promote e Critério de uso. Diferenciar claramente dev, staging, preview e produção quando existirem.

10. **Diagrama do Pipeline** — Incluir Mermaid flowchart cobrindo: commit → PR → checks → merge → build → deploy em staging → smoke tests → aprovação → deploy em produção → verificação. Se houver preview environments, incluir também.

11. **Workflows de CI/CD** — Documentar cada workflow com: nome, trigger, jobs, runners, secrets consumidos, artefatos gerados, caches usados, condições de falha, notificação emitida e impacto se falhar.

12. **Build e Artefatos** — Explicar o que é buildado por app ou serviço, onde o artefato fica, como é nomeado, como é versionado, por quanto tempo é retido e qual checksum ou validação confirma integridade.

13. **Estratégia de Deploy** — Definir abordagem adotada, justificativa, riscos, pré-condições, quando não usar e qual evidência confirma que a estratégia funcionou.

14. **Promoção entre Ambientes** — Documentar gates técnicos e humanos. Incluir smoke tests, testes automáticos, aprovação mínima, janela de deploy, critério objetivo de go/no-go e evidência que libera promoção.

15. **Rollback** — Incluir: quando acionar, quem pode acionar, comandos ou passos exatos, ordem de execução, tempo esperado, riscos colaterais e validações obrigatórias após rollback.

16. **Semantic Versioning** — Definir convenção de bump \(major, minor, patch\), exemplos, relação com hotfix, release candidate e tags Git.

17. **Ciclo de Release** — Descrever fluxo completo: desenvolvimento → QA → release candidate → aprovação → produção → estabilização. Se houver freeze ou janela de corte, isso deve ficar explícito.

18. **Changelog** — Definir formato, local, responsável, granularidade e momento da atualização. Mostrar um exemplo correto e um incorreto.

19. **Release Notes** — Definir template mínimo com: resumo, escopo, riscos conhecidos, ações pós-release, impacto para usuário e links relevantes.

20. **Tagging** — Definir convenção de tags Git, relação com versão do produto e regras para hotfix, rollback tag e release candidate.

21. **Comunicação de Release** — Explicar quem precisa ser avisado, em qual canal, em qual momento e com qual template de mensagem antes, durante e após o deploy.

22. **Hotfix Flow** — Criar fluxo separado para correção urgente em produção, com branch específica, approvals reduzidos, validação mínima obrigatória e versionamento próprio.

23. **Exemplos de referência** — Inclua pelo menos: \(a\) 1 linha de exemplo de matriz de ambientes; \(b\) 1 exemplo de workflow bem descrito; \(c\) 1 exemplo de rollback corretamente documentado; \(d\) 1 mini-template de release notes.

24. **Seções obrigatórias — detalhamento** — Cada seção da saída deve conter: \(a\) um breve parágrafo explicando o objetivo; \(b\) conteúdo técnico executável; \(c\) tabela, checklist, diagrama ou bloco de código quando fizer sentido; \(d\) critério explícito de sucesso.

25. **Estrutura obrigatória de saída** — O documento final deve conter exatamente estas seções, nesta ordem:
	1. Cabeçalho \(nome, versão, data, autor, status\)
	2. TL\;DR — resumo do pipeline
	3. Matriz de Ambientes
	4. Diagrama do Pipeline
	5. Workflows de CI/CD
	6. Estratégia de Deploy
	7. Promoção entre Ambientes
	8. Build e Artefatos
	9. Rollback
	10. Semantic Versioning
	11. Ciclo de Release
	12. Changelog
	13. Release Notes
	14. Tagging
	15. Comunicação de Release
	16. Hotfix Flow
	17. Backlog de Pendências

26. **Cobertura mínima** — O documento deve cobrir: pelo menos 3 ambientes quando existirem, pelo menos 2 workflows, um fluxo Mermaid completo, critérios de promoção, rollback detalhado, convenção de versionamento com exemplos e template de release notes. Toda decisão autônoma deve conter justificativa e alternativa descartada inline. Se qualquer cobertura mínima não for atingida, insira \[PENDÊNCIA\].

27. **Autoauditoria** — Antes de entregar, revise mentalmente: \(a\) todo ambiente tem propósito e trigger? \(b\) todo workflow tem falha e artefato documentado? \(c\) rollback está reproduzível? \(d\) versionamento está alinhado com tagging e changelog? \(e\) existe diferença explícita entre fluxo normal e hotfix? Se qualquer item falhar, corrija antes de responder.

28. **Regra final** — Retorne **somente o documento final**, já estruturado e pronto para uso em Markdown. Não inclua comentários, explicações ou meta-texto fora do documento.
```

---

## 3. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 20/03/2026 | v1.4 | Replicação da filosofia de decisão autônoma: §8 reescrito com `\[DECISÃO AUTÔNOMA\]` como regra geral e `\[DEFINIÇÃO PENDENTE\]` como exceção; cobertura atualizada. |
| 18/03/2026 | v1.3 | Auditoria completa do Prompt para reduzir ambiguidade, remover margem de erro e reforçar regras de saída. |
| 18/03/2026 | v1.2 | Expansão completa da seção Prompt para o mesmo nível de detalhe dos demais docs da pipeline. |
| 18/03/2026 | v1.1 | Padronização da estrutura do documento para o formato TL;DR → Persona → Prompt. |
| 08/03/2026 | v1.0 | Versão inicial do prompt — Pipeline ShiftLabs. |