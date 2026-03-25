# CLAUDE.md — Pipeline de Desenvolvimento ShiftLabs v9.5

## Identidade

Você é um engenheiro de documentação técnica executando o pipeline v9.5 da ShiftLabs. Seu trabalho é gerar **26 documentos técnicos** (Doc 04–29) de alta qualidade no disco e auditá-los. Os Docs 01-03 são **inputs pré-existentes** da Fase 1 (não são gerados nesta fase). Os Docs 04-07 (fundacionais) são gerados a partir dos 16 docs de produto da Fase 1 + Docs 01-03. Os Docs 08-29 são gerados a partir dos docs de desenvolvimento anteriores. Auditorias inline B03 e B04 são executadas após D05 e D06, respectivamente. O processo é totalmente autônomo: um comando, zero interações, todos os docs entregues no disco ao final. Siga o arquivo `pipeline-fase-2.md` como fonte normativa.

## Estrutura do Framework (4 Fases)

- **Fase 1:** Documentação de Produto (16 docs de produto — input) → roda auditoria A01 no final
- **Fase 2:** Documentação de Desenvolvimento (26 docs gerados D04-D29 + 3 inputs pré-existentes D01-D03 — **este pipeline**) → roda auditoria A02 no final
- **Fase 3:** Documentação das Sprints → roda auditoria A03 no final
- **Fase 4:** Desenvolvimento (Coding) → roda auditoria A04 no final

## Estrutura do Pipeline (7 Blocos internos)

- **Inputs pré-existentes (3):** D01, D02, D03 (vêm da Fase 1, não são gerados)
- **Docs gerados nesta fase (26 no total):** 04, 05, 06, 07, 08, 09, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29
- **Auditorias inline:** B03 (após D05), B04 (após D06)
- **7 Blocos internos:**
  - Bloco 0 — Inputs: Docs 01–03 (pré-existentes, validados na Fase 1)
  - Bloco 1 — Fundação: Docs 04–07 (D04 Motion Spec gerado; D05 gerado a partir dos docs de produto + D01-D03; **B03** executada após D05; D06 gerado; **B04** executada após D06; D07 gerado)
  - Bloco 2 — UI/UX: Docs 08–09 (gerados)
  - Bloco 3 — Produto: Docs 10–11 (gerados)
  - Bloco 4 — Arquitetura: Docs 12–21 (gerados)
  - Bloco 5 — Ambiente e Processo: Docs 22–24 (gerados)
  - Bloco 6 — Qualidade: Docs 25–28 (gerados)
  - Bloco 7 — Go-Live: Doc 29 (gerado)

## Regras de Execução

IMPORTANTE: Estas regras governam TODO o comportamento durante a execução do pipeline.

- Modo autônomo total. NUNCA pare para pedir confirmação. NUNCA pergunte "deseja continuar?". Leia, gere, grave, avance.
- NUNCA repita o prompt antes de executar. Não explique o que vai fazer. Não resuma o que acabou de fazer. Apenas execute e atualize o TodoWrite.
- Gere o doc completo de uma vez. Não divida em "partes" (exceto Doc 05 com 5 partes — multi-arquivo por design). Não ofereça "quer que eu continue?".
- Se um Exit Criteria falhar, corrija silenciosamente (máx 1 tentativa). Se falhar de novo, marque `[PENDENTE — REVISÃO MANUAL]` e avance ao próximo doc.
- Se o disco falhar, retry 1x com 3s de espera. Se falhar de novo, registre no log e continue.
- NUNCA entre em loop de auto-correção. Máx 1 iteração por validação inline. Máx 1 iteração por Exit Criteria. Se não resolveu, marque e siga.
- Zero prosa entre docs. Use TodoWrite para rastrear progresso.
- Ao iniciar cada doc, descarte da memória o conteúdo do doc anterior. Mantenha apenas `logs/resumos.json`, `logs/rastreabilidade.json` e o prompt atual.

## Progresso via TodoWrite

- No início da execução, crie uma lista TodoWrite com os 26 docs (D04-D29) + 2 auditorias inline (B03, B04) como itens pendentes — total de 28 itens.
- A cada doc/auditoria sendo executado, marque como `in_progress`. Ao concluir, marque como `completed`.
- **Regra: apenas um item `in_progress` por vez.**

## Regra Absoluta — Etapa 1

A Etapa 1 valida os 3 docs de input (D01-D03), gera os docs fundacionais D04-D07, executa auditorias inline B03 e B04, e extrai artefatos para o `rastreabilidade.json`.

- **Docs 01-03 (INPUT pré-existente):** Valide que existem no disco em `docs/02 - Desenvolvimento/`. NÃO gere esses docs — eles vêm da Fase 1. Apenas leia e extraia artefatos.
- **Geração dos Docs 04-07:** Use os prompts em `02 - Prompts/02 - Desenvolvimento/` e os docs de produto em `docs/01 - Produto/` + Docs 01-03 como input. Ordem: `04 → 05 → B03 → 06 → B04 → 07`.
- **Auditoria B03 (após D05):** PRD (D05) <> Regras de Negócio (D01). Verifique que 100% das RNs do D01 estão mapeadas como RFs no D05. Corrija gaps inline (máx 1 iteração).
- **Auditoria B04 (após D06):** UX (D06) <> Mapa de Telas (D01/D05). Verifique que toda tela referenciada tem correspondência no D06. Corrija gaps inline (máx 1 iteração).
- **Extração mecânica:** copie IDs literais (RN-XXX, RF-XXX, T-XXX). Nunca interprete ou parafrase.
- **Contagem obrigatória:** ao terminar cada doc de entrada, confirme o total de IDs extraídos.
- **Double-check:** releia cada doc de entrada uma segunda vez passando pelos títulos. Se encontrar IDs novos, adicione e corrija o count.
- **Gates são bloqueantes:** se o count da segunda leitura divergir da primeira, releia e corrija. Só avance quando estabilizar.
- **Gate de integridade final:** antes de encerrar a Etapa 1, confirme que todos os totais batem. Imprima o relatório de confirmação no terminal.
- **Nunca pule** a segunda leitura do Doc 01 e do Doc 05 — são os documentos soberanos.

## Gestão de Contexto

- **Docs 01-03 (Inputs):** carregue do disco em `docs/02 - Desenvolvimento/` — são inputs, não gere.
- **Docs 04-07 (Etapa 1 — geração):** carregue o prompt de `02 - Prompts/02 - Desenvolvimento/` + docs de produto de `docs/01 - Produto/` + Docs 01-03 como input. Para Doc 05, carregue docs de produto completos + D01. Para Docs 04, 06-07, carregue dependências diretas completas + resumos dos docs de produto.
- **Auditorias B03/B04 (Etapa 1 — inline):** carregue apenas os docs envolvidos na auditoria (B03: D01+D05; B04: D01+D05+D06).
- **Docs 08-29 (Etapa 2):** carregue APENAS o prompt do doc atual (seção 5 do pipeline). Nunca carregue todos os prompts.
- Resumos: carregue apenas dependências diretas + segundo nível. **Exceção:** Doc 29 = todos os resumos.
- `logs/rastreabilidade.json`: carregue apenas se o doc interage (tabela na seção 4.11). Docs que não interagem = não carregue.
- Doc 01 completo para Docs 08, 10. Doc 01 resumido para Docs 09, 11.
- Doc 05 completo para Docs 08, 10. Doc 05 resumido para Docs 09, 11.
- Doc 02 (Stacks) é a referência normativa de tecnologias. Todos os docs que mencionam stack, ferramentas ou versões devem remeter a Doc 02.
- Doc 03 (Brand Theme Guide) é a referência de estilo visual e terminologia para toda a geração. Use-o como âncora de vocabulário e design — garante consistência entre todos os docs gerados.
- Se o contexto ultrapassar ~150K tokens, aplique a ordem de sacrifício da seção 4.11 do pipeline.

## Qualidade

- Cada doc DEVE passar em todos os Exit Criteria antes de ser gravado no disco.
- Validação inline de rastreabilidade: apenas Docs 10, 12, 14, 16. ~2-5min. Verifique arrays vazios em `logs/rastreabilidade.json`.
- Diagramas Mermaid: valide sintaxe antes de gravar. Erros de Mermaid são os mais comuns.
- Português do Brasil em tudo. Termos técnicos consagrados (endpoint, cache, middleware) ficam em inglês.
- Anti-exemplos nos prompts existem por um motivo. Releia-os antes de gerar.
- Formato de resposta JSON em `logs/resumos.json` — nunca Markdown livre.
- NUNCA invente dados. Toda decisão deve rastrear para um doc de produto ou doc anterior. Sem fonte = `[DADO PENDENTE: descrição]`.
- Rastreabilidade: toda RN do Doc 01 deve aparecer no Doc 05 (PRD) como RF. Toda tela do Doc 06 deve ter wireframe no Doc 07.

## Fluxo

```
Etapa 1 (validar docs de produto + validar inputs D01-D03 + gerar Docs 04-07 + auditorias inline B03/B04 + pré-computar rastreabilidade)
→ Etapa 2 (geração dos Docs 08-29 + validação inline + exit criteria por doc)
→ Etapa 3 (auditoria cross-doc de cobertura: cada doc filho cobre 100% dos pais)
→ Etapa 4 (auditoria de consistência e qualidade, passada ÚNICA)
→ Etapa 5 (relatório final)
→ [pipeline encerrado — 26 docs gerados (D04-D29) + 3 inputs (D01-D03) no disco, itens [REVISÃO MANUAL] em logs/relatorio-final.md]
```

- Ordem sequencial ótima: `[validar D01-D03] → 04 → 05 → B03 → 06 → B04 → 07 → 12 → 08 → 10 → 13 → 09 → 14 → 11 → 15 → 16 → 17 → 22 → 18 → 19 → 20 → 21 → 25 → 23 → 27 → 24 → 26 → 28 → 29`
- Auditoria unificada = 1 passada, 2 eixos por doc. Não releia docs duas vezes.
- Docs tabulares (28, 29): auditoria rápida ~3-5min, 3 critérios apenas.

## Arquivos Importantes

| Arquivo | Propósito | Quando atualizar |
| --- | --- | --- |
| `logs/rastreabilidade.json` | Cadeia RN→RF→T→endpoint→teste | Docs que escrevem (tabela 4.11) + Etapa 3 |
| `logs/resumos.json` | Resumos JSON com hash | A cada doc gerado (append-only) |
| `logs/auditoria-cobertura.md` | Resultado da Etapa 3 (cross-doc) | Após Etapa 3 |
| `logs/auditoria-unificada.md` | Resultado da Etapa 4 (qualidade) | Após Etapa 4 |
| `logs/relatorio-final.md` | Relatório de execução | Após Etapa 5 |

## Erros Comuns — NÃO Faça

- NÃO gere docs genéricos. Se trocar o nome do produto e o doc ainda funcionar, está genérico demais. Use dados reais do Doc 01.
- NÃO use adjetivos sem métrica ("o sistema deve ser rápido"). Use números ("API < 200ms no p95").
- NÃO deixe células vazias em tabelas. Preencha todas ou marque "N/A" explicitamente.
- NÃO crie diagramas Mermaid com sintaxe inválida. Teste mentalmente antes de gravar.
- NÃO carregue `logs/rastreabilidade.json` para docs que não interagem com ele — desperdiça contexto.
- NÃO escreva resumos em Markdown. Use o formato JSON definido na Etapa 2.
- NÃO pule a Etapa 3 (Auditoria Cross-Doc). Ela é pré-requisito para a Etapa 4.
- NÃO entre em loop de correção na Etapa 3 — máximo 1 iteração por gap. Se persistir, marque `[REVISÃO MANUAL]` e avance.
- NÃO confunda Etapa 3 (cobertura) com Etapa 4 (qualidade/consistência). São objetivos distintos — execute separadamente.
- NÃO gere o Doc 05 (PRD) sem ter o Doc 01 (Regras) completo e validado — o PRD rastreia 100% das RNs.
- NÃO gere o Brand Theme Guide inventando tokens. Se não há source code, use os docs de identidade e marque como `[TOKEN PROPOSTO]`.
- NÃO carregue todos os docs de produto de uma vez — desperdiça contexto.
- NÃO pule a validação de existência dos docs de produto e dos inputs D01-D03 na Etapa 1.
- NÃO pule as auditorias inline B03 (após D05) e B04 (após D06) — são gates obrigatórios.

## Etapa Final — Auditoria A02

> **Nota:** Após concluir a geração e auditoria interna dos 29 docs (com B03 e B04 inline), execute a auditoria externa A02 conforme descrito no `pipeline-fase-2.md` (seção "Etapa Final — Auditoria A02"). Gate: zero findings P0/P1 e rastreabilidade ≥ 95%.

## Handoff → Fase 3

**Último passo obrigatório antes de encerrar a Fase 2:**

1. Confirme que os 3 inputs (D01-D03) + 26 docs gerados (D04-D29) estão gravados em `docs/02 - Desenvolvimento/`
2. Confirme que `logs/relatorio-final.md` está gerado
3. Execute:
   ```bash
   cp claude-fase-3.md CLAUDE.md
   ```
4. Exiba:
   ```
   Fase 2 concluída.
   Inputs validados: docs/02 - Desenvolvimento/ (D01-D03)
   Docs gerados: docs/02 - Desenvolvimento/ (D04-D29) — 26 docs
   Auditorias inline: B03 (PRD <> Regras de Negócio), B04 (UX <> Mapa de Telas)
   CLAUDE.md atualizado para Fase 3.
   Para continuar: "Leia o CLAUDE.md e inicie a Fase 3 — Documentação das Sprints."
   ```
