# Referências Padronizadas — ShiftLabs Framework

Documento de referência que formaliza padrões usados em todo o pipeline ShiftLabs mas nunca definidos em um único lugar: header de documento, template de resumo, marcadores e convenções de ID.

---

## 1. Tabela Padrão ShiftLabs (Header de Documento)

Todos os documentos gerados pelo pipeline devem iniciar com uma tabela de metadados no seguinte formato. Esta é a "Tabela padrão ShiftLabs" referenciada em todos os prompts de produto e desenvolvimento.

### 1.1. Formato da Tabela

```markdown
| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| [público-alvo do documento] | [descrição curta do escopo + posição no pipeline] | v1.0 | Claude Code Desktop | DD/MM/AAAA HH:MM (America/Fortaleza) |
```

### 1.2. Descrição dos Campos

| Campo | Descrição | Exemplo |
| --- | --- | --- |
| **Destinatário** | Público-alvo primário do documento. Quem vai consumir este output. | `Equipe de Produto e Engenharia` |
| **Escopo** | Descrição curta do que o documento cobre + sua posição na cadeia de dependências. | `Documento-raiz de regras de negócio — alimenta PRD, Modelo de Dados e Plano de Testes` |
| **Versão** | Versionamento semântico simplificado. Começa em `v1.0`. Auditorias incrementam minor (v1.1, v1.2). Reescritas incrementam major (v2.0). | `v2.3` |
| **Responsável** | Sempre `Claude Code Desktop` — nunca "uma IA" genérica. | `Claude Code Desktop` |
| **Data da versão** | Data e hora no formato `DD/MM/AAAA HH:MM` com fuso `(America/Fortaleza)`. | `20/03/2026 17:15 (America/Fortaleza)` |

### 1.3. Exemplo Preenchido

```markdown
| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Equipe de Produto e Engenharia | Documento-raiz de regras de negócio — alimenta PRD, Modelo de Dados e Plano de Testes | v2.3 | Claude Code Desktop | 20/03/2026 17:15 (America/Fortaleza) |
```

### 1.4. Template Vazio (Copiar e Preencher)

```markdown
| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| [PREENCHER] | [PREENCHER] | v1.0 | Claude Code Desktop | [DATA ATUAL] (America/Fortaleza) |
```

### 1.5. Regras de Uso

- A tabela é **sempre** o primeiro elemento do documento, imediatamente após o título H1.
- Seguida por um separador `---`.
- Logo abaixo vem o bloco **TL;DR** em formato de citação com o ícone `📌`.
- A versão nunca reinicia — se o doc já existe como v1.2, a próxima iteração é v1.3 (ou v2.0 se for reescrita).

> **Exceção:** O Doc 03 — Brand Theme Guide usa uma tabela de metadados própria (não a tabela padrão do pipeline), com campos específicos como `Produto`, `Stack`, `Escopo`, `Gerado por`.

---

## 2. Template de Resumo (Para Documentos Anteriores)

Muitos prompts do pipeline requerem "resumos dos docs anteriores" como input (ex.: Doc 06 pede "Docs 01-05 (resumos)", Doc 14 pede "Docs 01-13 (resumos)"). Este é o formato padronizado para esses resumos.

### 2.1. Regras Gerais

- **Tamanho:** 300 a 500 palavras por documento resumido.
- **Idioma:** mesmo do documento original (pt-BR).
- **Objetivo:** fornecer contexto suficiente para que o prompt consumidor não precise ler o doc completo, mas consiga referenciar decisões e IDs.

### 2.2. Estrutura Obrigatória do Resumo

```markdown
## Resumo — Doc XX: [Nome do Documento]

**TL;DR (1-2 frases):**
[Síntese do documento em no máximo 2 frases. O que ele define e por que importa.]

**Decisões-chave:**
- [Decisão 1 — contexto breve e justificativa]
- [Decisão 2 — contexto breve e justificativa]
- [Decisão N — máximo 7 itens]

**IDs gerados:**
- RN-001 a RN-045 (Regras de Negócio — módulos X, Y, Z)
- RF-001 a RF-023 (Requisitos Funcionais)
- [Listar todos os ranges de IDs com descrição do escopo]

**Dados críticos:**
- [Dado quantitativo 1 — ex.: TAM estimado em R$ 2,3 bi]
- [Dado quantitativo 2 — ex.: 5 personas definidas, ICP primário = gerentes de operação]
- [Métricas, benchmarks ou números que outros docs vão referenciar]

**Dependências geradas:**
- [Qual doc downstream consome o quê deste doc]

**Marcadores pendentes:**
- [DADO PENDENTE]: [descrição — X ocorrências]
- [DECISÃO AUTÔNOMA]: [descrição — X ocorrências]
- [DEFINIÇÃO PENDENTE]: [descrição — X ocorrências]
```

### 2.3. Exemplo Preenchido

```markdown
## Resumo — Doc 01: Pesquisa de Mercado e Benchmark

**TL;DR (1-2 frases):**
Mapeamento completo do mercado de gestão de pedidos para restaurantes no Brasil, com TAM/SAM/SOM calculados por metodologia bottom-up e análise de 12 concorrentes diretos.

**Decisões-chave:**
- TAM calculado por bottom-up (estabelecimentos ativos x ticket médio mensal de software)
- Segmento prioritário: restaurantes de 1 a 5 unidades (72% do SAM)
- Modelo freemium descartado com base em benchmark — churn acima de 40% no setor

**IDs gerados:**
- Nenhum ID formal (doc de pesquisa — gera dados, não regras)

**Dados críticos:**
- TAM: R$ 2,3 bi | SAM: R$ 890 mi | SOM: R$ 45 mi (ano 1)
- CAGR do setor: 12,3% (Fonte: Grand View Research, 2024)
- NPS médio dos concorrentes: 32 (baixo — oportunidade de diferenciação)

**Dependências geradas:**
- Doc 02 usa TAM/SAM/SOM para posicionamento
- Doc 03 usa lista de concorrentes como base da análise competitiva
- Doc 04 usa dados de comportamento do consumidor para ICPs
- Doc 14 usa projeções financeiras

**Marcadores pendentes:**
- [DADO PENDENTE]: participação de mercado do concorrente E — 1 ocorrência
```

### 2.4. Dicas de Uso

- Quando o prompt pedir `Docs 01-05 (resumos)`, gere **um bloco de resumo para cada doc** (01, 02, 03, 04, 05) seguindo o template acima.
- Quando o prompt pedir `Doc 07 (completo)`, forneça o **documento inteiro** — não o resumo.
- Resumos são **inputs**, não outputs — eles alimentam o próximo prompt, não são entregues ao usuário final.

---

## 3. Marcadores Padrão

Tags inline usadas em todo o pipeline para sinalizar estados especiais. Devem ser preservadas em auditorias e rastreáveis no relatório final.

### 3.1. Marcadores de Produção (Geração de Documentos)

| Marcador | Uso | Contexto |
| --- | --- | --- |
| `[DADO PENDENTE]` | Dado quantitativo que não foi encontrado em pesquisa e precisa de validação externa. | Pesquisa de mercado, benchmarks, métricas. |
| `[DECISÃO AUTÔNOMA]` | Decisão tomada pela IA com base na análise de contexto disponível, sem input explícito do usuário. Deve sempre incluir justificativa inline + alternativa descartada. | Todos os docs de produto e desenvolvimento. |
| `[DEFINIÇÃO PENDENTE]` | Decisão de **alto impacto** (compliance, financeiro, jurídico, RBAC) onde não há contexto suficiente para decidir. Deve apresentar 2 opções (A/B) com análise de trade-off. | Regras de negócio, modelo de dados, arquitetura. |
| `[SEÇÃO PENDENTE]` | Seção inteira que não pôde ser completada por falta de insumos. | Qualquer doc onde o input não cobriu o escopo necessário. |

### 3.2. Marcadores de Auditoria (Correção e Rastreio)

| Marcador | Uso | Contexto |
| --- | --- | --- |
| `[CORRIGIDO: PROBLEMA-XXX]` | Indica que um problema identificado na auditoria foi corrigido neste trecho. O ID `PROBLEMA-XXX` vincula ao registro no relatório de auditoria. | Documentos corrigidos pós-auditoria. |
| `[CONFLITO INTERNO: CI-XXX]` | Contradição entre duas partes do mesmo documento ou entre documentos do pipeline. Registra as duas versões e aplica a mais restritiva. | Auditorias de RN, PRD, UX. |
| `[CONFLITO UX: PROBLEMA-XXX vs PROBLEMA-YYY]` | Conflito específico entre problemas de UX que se contradizem. Registra qual foi favorecido e justificativa. | Auditoria UX vs Regras de Negócio. |
| `[DECISÃO APLICADA: DEC-XXX]` | Decisão interpretativa tomada durante auditoria quando normalmente precisaria de input humano. Inclui justificativa e nível de confiança (ALTA/MÉDIA/BAIXA). | Todas as auditorias. |

### 3.3. Marcadores Específicos de UX/Wireframes

| Marcador | Uso | Contexto |
| --- | --- | --- |
| `[WIREFRAME PENDENTE]` | Wireframe que não pôde ser gerado por falta de definição de layout ou hierarquia. | Doc de Wireframes. |
| `[COMPONENTE PENDENTE]` | Componente de UI referenciado mas ainda não definido no design system. | Doc de Wireframes, Brand Theme Guide. |

### 3.4. Regras de Uso dos Marcadores

1. Marcadores **nunca são removidos** — mesmo após resolução, o histórico permanece.
2. Em auditorias, todo `[CORRIGIDO]` deve ter um `PROBLEMA-XXX` correspondente no relatório.
3. Toda `[DECISÃO APLICADA]` deve ter justificativa. Se ultrapassar 15 ocorrências em um único documento, gerar alerta no relatório final.
4. `[DECISÃO AUTÔNOMA]` é preferível a `[DEFINIÇÃO PENDENTE]` — só use pendente quando o impacto for alto E o contexto for zero.

---

## 4. Convenção de IDs

Todos os IDs seguem o formato `PREFIXO-XXX` onde XXX é numeração sequencial com 3 dígitos (001, 002, ...). A numeração **nunca reinicia** dentro de um documento, mesmo quando dividido em partes.

### 4.1. IDs de Produto e Negócio

| Prefixo | Significado | Documento de Origem |
| --- | --- | --- |
| `RN-XXX` | Regra de Negócio | 01 - Regras de Negócio (partes 01.1 a 01.5) |
| `RF-XXX` | Requisito Funcional | 02 - PRD |
| `REQ-XXX` | Requisito (genérico) | Diversos (quando não se encaixa em RF) |
| `DEC-XXX` | Decisão registrada | Auditorias e docs de decisão |

### 4.2. IDs de UX e Design

| Prefixo | Significado | Documento de Origem |
| --- | --- | --- |
| `T-XXX` | Tela | Mapa de Telas |
| `FLOW-XXX` | Fluxo de Usuário | User Flows, Service Design |
| `FEAT-XXX` | Feature | PRD, Roadmap |

### 4.3. IDs de Dados e Arquitetura

| Prefixo | Significado | Documento de Origem |
| --- | --- | --- |
| `ENT-XXX` | Entidade do Modelo de Dados | Modelo de Dados |

### 4.4. IDs de Qualidade e Auditoria

| Prefixo | Significado | Documento de Origem |
| --- | --- | --- |
| `TC-XXX` | Caso de Teste | Plano de Testes |
| `PROBLEMA-XXX` | Problema identificado em auditoria | Todas as auditorias |
| `DIV-XXX` | Divergência entre documentos | Auditorias de cobertura cruzada |
| `CI-XXX` | Conflito Interno (contradição dentro do mesmo doc) | Auditorias |

### 4.5. Regras de Numeração

1. **Continuidade entre partes:** Se o Doc 01.1 termina em RN-023, o Doc 01.2 começa em RN-024.
2. **Cross-references:** Usar formato `Conforme RN-XXX (Parte 01.Y)` para referências entre partes do mesmo documento.
3. **Nunca reutilizar IDs:** Um ID aposentado não volta. Se RN-015 foi removido, o próximo RN após RN-014 é RN-016 (mantém gap).
4. **Escopo do ID:** O prefixo define o tipo. Nunca usar RN para requisito funcional ou RF para regra de negócio.

---

## 5. Equivalência de Severidade

Classificação padronizada de severidade usada em auditorias, backlog de problemas e priorização de correções.

| Nível | Nome | Definição | Ação Esperada |
| --- | --- | --- | --- |
| **P0** | Crítico | Bloqueante — impede o uso do sistema ou fluxo principal. Perda de dados, falha de segurança, impossibilidade de completar ação core. | Correção imediata. Bloqueia deploy. |
| **P1** | Alto | Impacta fluxo principal — o usuário consegue usar o sistema, mas com degradação significativa na experiência ou funcionalidade core. | Correção antes do próximo release. |
| **P2** | Médio | Impacta fluxo secundário — funcionalidades de suporte, edge cases, ou fluxos alternativos apresentam problemas. | Correção planejada no backlog. |
| **P3** | Baixo | Cosmético ou melhoria — problemas visuais, de copy, ou oportunidades de otimização que não afetam funcionalidade. | Incluir no backlog, priorizar quando possível. |

### 5.1. Regras de Classificação

- Na dúvida entre dois níveis, **usar o mais severo**.
- Problemas de **segurança** são sempre **P0**, independentemente do fluxo afetado.
- Problemas de **acessibilidade** que impedem uso por grupo relevante de usuários são no mínimo **P1**.
- Inconsistências de **dados** (ex.: regra e implementação divergem) são no mínimo **P1**.
- Problemas **cosméticos** em fluxo principal podem ser **P2** (não P3) se afetarem percepção de qualidade.
