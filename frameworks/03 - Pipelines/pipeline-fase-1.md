# Pipeline de Documentação de Produto — ShiftLabs v10.7

## Framework de Documentação de Produto com IA — Claude Code Desktop Edition

---

| Campo | Valor |
| --- | --- |
| **Destinatário** | Claude Code Desktop (execução automatizada) + Time de produto da ShiftLabs (consumo dos docs) |
| **Escopo** | Orquestração completa de 17 documentos de produto via Claude Code Desktop (1 briefing + 16 gerados) |
| **Caráter** | **Normativo.** Este pipeline define a ordem, as dependências, os prompts e os critérios de qualidade para cada documento. |
| **Versão** | v10.7 |
| **Responsável** | Fernando Calado |
| **Data** | 19/03/2026 — America/Fortaleza |
| **Revisão** | A cada novo produto ou mudança significativa no framework |
| **Ambiente de execução** | Claude Code Desktop + sistema de arquivos local |

---

## TL;DR

- **17 documentos** no total: Doc 00 (Briefing, preenchido pelo usuário) + 16 gerados organizados em **4 etapas** sequenciais (1 — Discovery → 2 — Identidade → 3 — Produto → 4 — Negócio), mantidos no disco.
- **Um comando só:** você diz "execute o pipeline de produtos para [produto]" e o Claude Code lê este arquivo, lê o `docs/01 - Produto/00 - Briefing.md` local, gera tudo no disco e audita.
- **100% local:** zero dependência de Notion ou MCP. Briefing em `docs/01 - Produto/00 - Briefing.md`, docs finais em `docs/01 - Produto/`.
- **Estado no disco, não na memória:** progresso em `logs/progresso-pipeline-produtos.json`, resumos JSON estruturados em `logs/resumos.json`. Se a sessão cair, retoma automaticamente.
- **Contexto leve:** carrega apenas o prompt do doc atual + resumos seletivos (dependências diretas + 2º nível) + dependências completas. Nunca os 17 arquivos ao mesmo tempo.
- **Auditoria unificada** (passada única): Consistência + qualidade avaliados simultaneamente por doc. Correções no disco = zero operações externas.
- **Exit Criteria** em cada prompt: checklist binário verificável em <30s antes de gravar cada doc.
- **Web search bilíngue** para docs que exigem pesquisa (Docs 01, 03, 08, 14, 15, 16): pt-BR primeiro, en-US depois.
- **Fluxo simplificado:** Etapa 1 → Etapa 2 (geração no disco + validação + exit criteria) → Etapa 3 (auditoria unificada) → Etapa 4 (relatório).
- **Tamanho total estimado**: 49.000 a 71.000 palavras. **Tempo estimado: ~10-14 horas**. Economia por: pipeline 100% local (sem rate limiting), auditoria unificada (~1.5h vs. 3 passadas), exit criteria (menos retrabalho ~1h).

---

## Tabela Consolidada — Docs, Propriedades e Dependências

| # | Nome | Etapa | Área | Depende de | Web Search | Tamanho-alvo |
| --- | --- | --- | --- | --- | --- | --- |
| — | **00 — Briefing** | — | — | **Preenchido pelo usuário (input)** | — | — |
| 1 | 01 — Pesquisa de Mercado e Benchmark | 1 — Discovery | Mercado | Doc 00 (Briefing) | Sim | 4.000-6.000 |
| 2 | 02 — Análise Geral | 1 — Discovery | Estratégia | Doc 01 completo | Não | 4.000-6.000 |
| 3 | 03 — Concorrentes | 1 — Discovery | Mercado | Docs 01-02 resumos + Doc 02 completo | Sim | 3.000-5.000 |
| 4 | 04 — One-Liner e ICPs | 1 — Discovery | Marketing | Docs 01-03 resumos + Doc 03 completo | Não | 3.000-4.000 |
| 5 | 05 — Manifesto da Marca | 2 — Identidade | Branding | Docs 01-04 resumos + Doc 04 completo | Não | 2.000-3.000 |
| 6 | 06 — Memorando de Essência | 2 — Identidade | Branding | Docs 01-05 resumos + Doc 05 completo | Não | 4.000-6.000 |
| 7 | 07 — Tom de Voz e Identidade Verbal | 2 — Identidade | Branding | Docs 01-06 resumos + Doc 06 completo | Não | 3.000-4.000 |
| 8 | 08 — Social Media | 2 — Identidade | Marketing | Docs 01-07 resumos + Doc 07 completo | Sim | 2.500-3.500 |
| 9 | 09 — Design Thinking | 3 — Produto | Produto | Docs 01-08 resumos + Doc 04 completo | Não | 3.000-4.000 |
| 10 | 10 — Jobs To Be Done | 3 — Produto | Produto | Docs 01-09 resumos + Doc 09 completo | Não | 3.000-4.000 |
| 11 | 11 — Service Design | 3 — Produto | Produto | Docs 01-10 resumos + Doc 10 completo | Não | 2.500-3.500 |
| 12 | 12 — Casos de Uso | 3 — Produto | Produto | Docs 01-11 resumos + Docs 10+11 completos | Não | 3.000-4.500 |
| 13 | 13 — UX Writing | 3 — Produto | UI/UX | Docs 01-12 resumos + Docs 07+12 completos | Não | 2.500-3.500 |
| 14 | 14 — Modelo de Negócios | 4 — Negócio | Estratégia | Docs 01-13 resumos + Docs 04+06 completos | Sim | 3.000-4.500 |
| 15 | 15 — Proposta de Valor | 4 — Negócio | Comercial | Docs 01-14 resumos + Docs 06+14 completos | Sim | 3.000-4.500 |
| 16 | 16 — Pricing Book Interno | 4 — Negócio | Comercial | Docs 01-15 resumos + Docs 04+06+14+15 completos | Sim | 3.500-5.500 |

**Total estimado:** 49.000 a 71.000 palavras (~16 documentos gerados, Doc 00 = Briefing preexistente)

**Tempo estimado por fase:** Discovery (Docs 01-04) ≈ 4-6h · Identidade (Docs 05-08) ≈ 3-5h · Produto (Docs 09-13) ≈ 4-6h · Negócio (Docs 14-16) ≈ 3-5h · Auditorias + Índice ≈ 2-4h

---

# SUB-PROMPTS — Persona e Instruções por Documento

Ao gerar cada documento, **assuma a persona indicada**. Isso define o ângulo, o vocabulário e os critérios de profundidade. Após gerar, **volte à persona de orquestrador** para publicar e registrar o progresso.

---

---

## Formatação — Padrão ShiftLabs

### Header (obrigatório em todos os docs)

```
<table fit-page-width="true">
<tr>
<td>**Destinatário**</td>
<td>[do briefing]</td>
</tr>
<tr>
<td>**Escopo**</td>
<td>[descrição curta do doc]</td>
</tr>
<tr>
<td>**Versão**</td>
<td>v1.0</td>
</tr>
<tr>
<td>**Responsável**</td>
<td>[do briefing]</td>
</tr>
<tr>
<td>**Data da versão**</td>
<td>[DD/MM/AAAA HH:MM — fuso America/Fortaleza]</td>
</tr>
</table>
```

### TL;DR (obrigatório em todos os docs)

```
::: callout {icon="📌" color="gray_bg"}
	**TL;DR**
	- [máximo 7 bullets com as decisões/conclusões mais importantes]
:::
```

### Callouts por tipo

- 🎯 `blue_bg`: Decisão, posicionamento
- ✅ `green_bg`: Resultado positivo, recomendação
- 💡 `purple_bg`: Insight, observação estratégica
- 🔴 `red_bg`: Alerta, risco
- ⚙️ `orange_bg`: Detalhe técnico

### Tabelas

```
<table fit-page-width="true" header-row="true">
```

### Referências cruzadas (a partir do Doc 03)

```
<mention-page url="{{URL_REAL_DO_DOC}}"/>
```

### Changelog (obrigatório no final de cada doc)

```
## Changelog
<table fit-page-width="true" header-row="true">
<tr>
<td>**Data**</td>
<td>**Versão**</td>
<td>**Descrição**</td>
</tr>
<tr>
<td>[DD/MM/AAAA]</td>
<td>v1.0</td>
<td>Versão inicial gerada pelo Pipeline ShiftLabs v10.6</td>
</tr>
</table>
```

## Títulos das Páginas

Formato obrigatório: `XX — Nome do Doc`. Exemplo: `01 — Pesquisa de Mercado e Benchmark`. Sem emoji, sem prefixo extra. Fase e Área ficam nas propriedades.

---

## Ordem do Conteúdo (obrigatória em todos os docs)

Cada página segue esta ordem, sem exceção:

1. **Header** (tabela de metadados)
2. **TL;DR** (callout cinza com máx 7 bullets)
3. **Seções obrigatórias** (conforme sub-prompt do doc)
4. **Changelog** (tabela de versão)

---

---

## Orquestração — Claude Code Desktop

Este pipeline é executado pelo Claude Code Desktop. Você não cola prompts. O Claude Code lê este arquivo do disco, interpreta as instruções, e executa passo a passo.

### Idioma

Todos os documentos, resumos, mensagens de progresso, a página-índice e qualquer interação devem ser em **Português do Brasil**. Sem exceção. Termos técnicos consagrados (ex: stakeholder, upsell, churn) e nomes próprios podem permanecer em inglês.

### Como Executar

**Comando único:** O usuário diz algo como:

```
Execute o pipeline de produtos para ShiftLabs.
Briefing: <Link>
```

O Claude Code Desktop deve:

1. Localizar este arquivo no disco (o pipeline md)
2. Ler as instruções de execução
3. Executar a Etapa 1 (preparação), Etapa 2 (geração no disco), Etapa 3 (auditoria unificada no disco), Etapa 4 (relatório final)
4. Gravar progresso no disco a cada doc gerado

**Modo fixo:** Completo + Autonomia Total. O pipeline sempre gera todos os 16 docs (Doc 01 a Doc 16) do início ao fim, sem parar para confirmação em nenhum momento.

### Estrutura de Arquivos no Disco

O Claude Code Desktop deve criar e manter estes arquivos no diretório do projeto:

```
projeto/
├── pipeline-fase-1.md                               ← este arquivo (read-only)
├── docs/
│   ├── 01 - Produto/                               ← output da Fase 1: Documentação de Produto (este pipeline)
│   │   ├── 00 - Briefing.md                         ← briefing do produto (input, preenchido pelo usuário)
│   │   ├── 01 - Pesquisa de Mercado e Benchmark.md
│   │   └── ...
│   ├── 02 - Desenvolvimento/                       ← output da Fase 2: Documentação de Desenvolvimento
│   ├── 03 - Sprints/                               ← output da Fase 3: Documentação das Sprints
│   └── 05 - Auditorias/                             ← relatórios de auditoria (A01-A04)
├── src/                                             ← output da Fase 4: código implementado
└── logs/
    ├── progresso-pipeline-produtos.json             ← estado da execução (criado pelo Claude Code)
    ├── resumos.json                                 ← resumos JSON estruturados com hash
    ├── glossario-global.md                          ← glossário unificado cross-doc
    └── fase-1/
        ├── auditoria-unificada.md                   ← resultado da auditoria (consistência + qualidade)
        └── relatorio-final.md                       ← relatório final
```

### Arquivo de Progresso — progresso-pipeline-produtos.json

Criado na Etapa 1. Atualizado após cada doc publicado. Lido no início de cada sessão para retomada automática.

```json
{
  "produto": "Nome do Produto",
  "empresa": "Nome da Empresa",
  "classificacao": {
    "stakeholders": 3,
    "modelo": "B2B2C",
    "planos": "multiplos",
    "marketplace": false
  },
  "docs": {
    "00": { "status": "preexistente", "arquivo_local": "docs/01 - Produto/00 - Briefing.md", "titulo": "00 — Briefing" },
    "01": { "status": "gerado", "arquivo_local": "docs/01 - Produto/01 - Pesquisa de Mercado e Benchmark.md", "titulo": "01 — Pesquisa de Mercado e Benchmark" },
    "02": { "status": "pendente", "arquivo_local": null, "titulo": "02 — Análise Geral" }
  },
  "etapa_atual": "doc-02",
  "auditorias": {
    "unificada": "pendente"
  },
  "ultima_atualizacao": "2026-02-27T15:30:00-03:00"
}
```

**Backup:** Copie o JSON para `logs/progresso-pipeline-produtos.backup.json` apenas em transição de etapa (Discovery→Identidade→Produto→Negócio→Auditoria→Relatório). Não a cada doc — reduz I/O.

**Regra de retomada:** Ao iniciar, o Claude Code SEMPRE verifica se `logs/progresso-pipeline-produtos.json` existe. Se existe, lê e continua de `etapa_atual`. Se não existe, executa a Etapa 1 do zero.

**Re-execução do pipeline (mesmo produto):** Se o JSON já existe com `etapa_atual: "concluido"`:

1. Incremente a versão de todos os docs: vX.0 → v(X+1).0.
2. Regenere todos os docs no disco (Etapa 1 do zero). Os arquivos locais serão sobrescritos.
3. Atualize o Changelog de cada doc com a nova versão e motivo da re-geração.
4. Resete `etapa_atual` para `"doc-01"` e `auditorias.unificada` para `"pendente"` no JSON.
5. Preserve os resumos anteriores em `logs/resumos.json` (append-only com separador entre execuções).

### Etapa 1 — Leitura, Validação e Preparação

### 1.1 Leia a spec de formatação

Use a REFERÊNCIA DE SINTAXE incluída neste arquivo (seção abaixo).

### 1.2 Leia o briefing

Leia o arquivo `docs/01 - Produto/00 - Briefing.md`. Extraia e valide:

**Campos obrigatórios (se faltar, marque `[CAMPO OBRIGATÓRIO AUSENTE]` e prossiga):**

- Nome do produto (ou nome de trabalho)
- Nome da empresa
- Descrição do produto (o que é, para quem, que problema resolve)
- Setor/mercado
- Problema que resolve
- Público-alvo inicial
- Plataformas-alvo (Web, Mobile ou ambos)

**Threshold mínimo:** Se o briefing não tiver ao menos **nome do produto** + **descrição** + **público-alvo**, interrompa e solicite ao usuário: `Briefing insuficiente. Necessário ao menos: nome do produto, descrição e público-alvo inicial.`

**Campos recomendados (prossiga marcando `[DEFINIÇÃO PENDENTE]` se faltar):**

- Modelo de negócio pretendido (SaaS, marketplace, comissão, etc.)
- Concorrentes conhecidos
- Diferenciais pretendidos
- Restrições (orçamento, prazo, regulações, limitações técnicas)
- Responsável e destinatário

**Checklist de validação do briefing (execute antes de avançar para 1.3):**

| Campo | Presente? | Impacto se ausente |
| --- | --- | --- |
| Nome do produto | Sim / Não | **BLOQUEANTE** — stop se ausente |
| Descrição do produto | Sim / Não | **BLOQUEANTE** — stop se ausente |
| Público-alvo inicial | Sim / Não | **BLOQUEANTE** — stop se ausente |
| Nome da empresa | Sim / Não | `[DEFINIÇÃO PENDENTE]` em todos os headers |
| Setor / mercado | Sim / Não | Doc 01 marcado `[DEFINIÇÃO PENDENTE]` |
| Problema central que resolve | Sim / Não | Docs 01, 02, 04 parcialmente prejudicados |
| Modelo de negócio | Sim / Não | Docs 14, 15, 16 marcados `[DEFINIÇÃO PENDENTE]` |
| Concorrentes conhecidos | Sim / Não | Doc 03 com `[SEÇÃO COM DADOS LIMITADOS]` |
| Diferenciais pretendidos | Sim / Não | Docs 04, 05, 15 com profundidade reduzida |
| Plataformas-alvo | Sim / Não | `[DEFINIÇÃO PENDENTE]` em Docs 09, 11, 12 |

**Saída obrigatória da Etapa 1.2:** Imprima no terminal:
```
Briefing validado — [Nome do Produto]
Bloqueantes: ✅ OK (ou ❌ STOP se ausente)
Campos recomendados ausentes: [lista ou "nenhum"]
```
Se qualquer BLOQUEANTE estiver ausente, interrompa imediatamente. Não prossiga para 1.3.

### 1.3 Analise o tipo de produto

Com base no briefing, classifique:

- **Número de stakeholders:** 2 atores ou 3+ atores?
- **Modelo de interação:** B2B, B2C, B2B2C, marketplace?
- **Planos:** plano único ou múltiplos?
- **Marketplace:** sim/não?

Grave a classificação no `logs/progresso-pipeline-produtos.json`.

### 1.4 Crie a estrutura de arquivos

Crie o diretório `docs/01 - Produto/` e `logs/fase-1/` (se não existirem) e os arquivos:

- `logs/progresso-pipeline-produtos.json` com todos os dados coletados nos passos 1.2 e 1.3
- `logs/resumos.json` vazio com estrutura:

```json
{
  "produto": "[Nome do Produto]",
  "gerado_em": "[data]",
  "resumos": []
}
```

- `logs/glossario-global.md` com header:

```markdown
# Glossário Global — [Nome do Produto]

Termos unificados extraídos de todos os docs do pipeline.
Atualizado automaticamente após cada doc gerado.
Gerado em: [data]

| Termo | Definição | Origem (Doc) | Docs que usam |
|-------|-----------|-------------|---------------|
```

- Diretório `docs/` (para fallback local)
- Diretório `logs/`

### 1.5 Informe o início no terminal

Grave o resumo no terminal e inicie o Doc 01 imediatamente:

```
Pipeline de Produtos — [Nome do Produto]

Empresa: [empresa]
Setor: [setor]
Tipo: [B2B/B2C/B2B2C] com [X] stakeholders
Marketplace: [Sim/Não]
Planos: [único/múltiplos]

Total de docs: 17 (00 — Briefing preexistente + 16 gerados: Doc 01 a Doc 16)
Iniciando Doc 01...
```

### Gestão de Contexto — Leitura por Doc

**Regra fundamental:** O Claude Code NÃO carrega os 17 arquivos na memória ao mesmo tempo. Para cada doc, ele:

1. Lê `logs/progresso-pipeline-produtos.json` do disco (estado atual)
2. Lê de `logs/resumos.json` **apenas os resumos das dependências diretas e de segundo nível** (não todos). Exemplo: Doc 12 depende de 10+11 → carrega resumos de 10, 11 + seus pais (01-09). **Exceção:** Doc 16 carrega todos os resumos.
3. Lê deste arquivo APENAS o prompt do doc atual
4. Lê do disco (`docs/01 - Produto/`) os docs de dependência completos. O briefing está sempre disponível em `docs/01 - Produto/00 - Briefing.md`.
5. Gera o conteúdo
6. **Verifica Exit Criteria** antes de gravar
7. **Grava o doc no disco** em `docs/01 - Produto/`
8. Atualiza `logs/progresso-pipeline-produtos.json` e `logs/resumos.json` no disco

**O que carregar por fase:**

**Docs 01-03 (Discovery):** Briefing completo (`docs/01 - Produto/00 - Briefing.md`) + resumos do disco + doc anterior completo do disco. Web search ativo.

**Docs 04-08 (Identidade):** Briefing resumido (500 palavras) + resumos do disco + dependências completas do disco.

**Docs 09-13 (Produto):** Briefing resumido + resumos do disco + dependências completas do disco.

**Docs 14-16 (Negócio):** Briefing resumido + resumos do disco + dependências completas do disco. Web search ativo para Docs 14, 15 e 16.

**Ordem de sacrifício se contexto apertar:**

1. Conteúdo completo de docs antigos (já estão nos resumos)
2. Detalhes do briefing (usar resumido)
3. NUNCA sacrificar: resumos do disco + dependências completas do disco

**Threshold prático:** Se o contexto estimado ultrapassar ~150k tokens (~6 docs completos + resumos + prompt), ative a divisão em blocos descrita na seção 2.2. Referência: 1 doc completo ≈ 5.000–8.000 tokens; resumos acumulados ≈ 5.000–10.000 tokens; prompt do doc atual ≈ 3.000–5.000 tokens.

### Web Search — Estratégia Bilíngue

Quando o doc exigir web search (Docs 01, 03, 08, 14, 15, 16):

**Rodada 1 (pt-BR):** dados locais, regulações brasileiras, concorrentes locais, benchmarks regionais.

**Rodada 2 (en-US):** benchmarks globais, estudos de caso, frameworks e metodologias, dados de mercado.

**Prioridade de fontes:** Relatórios de mercado (Gartner, Statista, CB Insights) > Documentação oficial > Estudos de caso > Artigos especializados > Blog posts.

**Fontes obrigatórias:** Toda métrica deve ter fonte inline: `X% [Fonte: Gartner, 2024]`. Sem fonte verificável = `[DADO PENDENTE: descrição do dado faltante]`. Zero exceções — nunca escreva um número de mercado sem fonte ou sem marcação.

**Se dados insuficientes:** marque `[SEÇÃO COM DADOS LIMITADOS]`. Nunca invente dados de mercado ou métricas.

### Checklist de Qualidade (validar antes de gravar cada doc no disco)

- [ ]  Português do Brasil?
- [ ]  Ordem correta: Header > TL;DR > Seções > Changelog?
- [ ]  Header padronizado?
- [ ]  TL;DR com máx 7 bullets?
- [ ]  Seções obrigatórias do prompt presentes?
- [ ]  Tamanho-alvo atingido? Se fora do range: expanda seções fracas (abaixo do mín) ou remova redundâncias (acima do máx) e revalide.
- [ ]  Callouts corretos por tipo?
- [ ]  Tabelas com fit-page-width e header-row?
- [ ]  Terminologia consistente com docs anteriores?
- [ ]  Todo dado quantitativo tem fonte inline no formato `[Fonte: Nome, AAAA]`? Se não tiver fonte verificável, está marcado `[DADO PENDENTE: descrição do dado faltante]`? Zero métricas sem fonte e sem marcação.
- [ ]  Diagramas Mermaid válidos? (nós com texto especial entre aspas duplas, `-->` sem escape, `[*]` sem escape em stateDiagram)
- [ ]  Changelog presente?
- [ ]  Cobertura mínima: cada seção obrigatória tem ≥ 5% do total de palavras do doc (nenhuma seção vazia ou simbólica)?
- [ ]  Arquivo `logs/resumos.json` atualizado no disco? (`tem_pendencias` e `pendencias_count` preenchidos corretamente?)
- [ ]  Arquivo `logs/progresso-pipeline-produtos.json` atualizado no disco?


### Tratamento de Erros

### Web search insuficiente

1. Queries alternativas (mais específicas, em inglês, nomes de concorrentes).
2. Se insuficiente: gere com o que tem, marque `[SEÇÃO COM DADOS LIMITADOS]`.
3. Nunca invente dados de mercado. Toda métrica sem fonte verificável = `[DADO PENDENTE: descrição do dado faltante]`.

### Falha na leitura de dependências

1. Retente 1x com delay de 5 segundos.
2. Se falhar: use o resumo do doc de dependência em `logs/resumos.json`.
3. Se o resumo for insuficiente: marque `[DEPENDÊNCIA PARCIAL: Doc XX]` nas seções que usariam dados desse doc.
4. Continue a geração com o contexto disponível.
5. Registre a falha no `logs/progresso-pipeline-produtos.json` para revisão na auditoria.

### Web search indisponível

1. Se a ferramenta de web search não estiver disponível no ambiente, marque `[WEB SEARCH INDISPONÍVEL]` no doc.
2. Gere o conteúdo com dados do briefing + resumos + conhecimento geral.
3. Registre no `logs/progresso-pipeline-produtos.json` para revisão na auditoria.
4. Na auditoria, priorize esses docs para verificação e complementação manual de dados.

### Briefing incompleto

- Campos obrigatórios: marque `[CAMPO OBRIGATÓRIO AUSENTE: nome]` e prossiga.
- Campos recomendados: prossiga marcando `[DEFINIÇÃO PENDENTE]`.

### Sessão interrompida

Se a sessão do Claude Code Desktop cair, fechar, ou perder conexão:

1. Na próxima execução, o Claude Code lê `logs/progresso-pipeline-produtos.json` do disco.
2. Identifica `etapa_atual` e último doc publicado.
3. Retoma exatamente de onde parou, sem regenerar docs já gerados.
4. Informa ao usuário: "Retomando de [etapa_atual]. Docs 01 a [XX] já gerados."

### Perda de contexto

1. Leia `logs/resumos.json` do disco.
2. Se insuficiente, releia os docs completos de `docs/01 - Produto/`.
3. Reconstrua o contexto e continue.

---

### Referência de Sintaxe — Notion Markdown

Use esta referência para formatar todos os documentos gerados pelo pipeline. Os docs são gerados localmente mas seguem a sintaxe do Notion Markdown para importação direta.

**Nota importante:** Dentro de blocos de código, NÃO escape caracteres especiais. Escreva `[`, `]`, `>` literalmente. Escapes (`\[`, `\]`, `--\>`) só são necessários fora de blocos de código.

---

#### Header (H1 + H2 + Metadata)

Todo documento começa com H1 (título do produto/projeto) + H2 (título do doc) + bloco de metadados em tabela:

```
# Nome do Produto
## Doc XX — Nome do Documento

| Campo | Valor |
| --- | --- |
| **Destinatário** | [Quem vai usar este doc] |
| **Escopo** | [O que este doc cobre] |
| **Versão** | vX.X |
| **Responsável** | [Nome] |
| **Data** | DD/MM/AAAA HH:MM — America/Fortaleza |
```

---

#### TL;DR

Callout `📌 gray_bg` com 3–5 bullets de decisões críticas e números. **Opcional** para docs puramente técnicos (listas de endpoints, glossários):

```
::: callout {icon="📌" color="gray_bg"}
	- **Decisão 1:** [resumo em 1 linha]
	- **Decisão 2:** [resumo em 1 linha]
	- **Número chave:** [métrica com unidade]
:::
```

---

#### Hierarquia Numerada

Use numeração multinível. Nunca `1.a` — sempre `1.1`:

```
# 1 — Título Principal        (H1 — fonte grande)
## 1.1 Subtítulo              (H2 — fonte média)
### 1.1.1 Subseção            (H3 — fonte pequena)
#### 1.1.1.1 Detalhe          (H4 — tamanho normal, bold)
```

---

#### Callouts Temáticos

| Callout | Ícone | Cor | Uso |
| --- | --- | --- | --- |
| Objetivo | 🎯 | blue_bg | Metas, propósitos, north stars |
| Insight | 💡 | purple_bg | Descobertas, hipóteses, interpretações |
| Resultado | ✅ | green_bg | Conclusões, validações, entregas |
| Alerta crítico | 🔴 | red_bg | Riscos, bloqueadores, requisitos obrigatórios |
| Requisito técnico | ⚙️ | orange_bg | Dependências de sistema, performance, integrações |
| TL;DR | 📌 | gray_bg | Resumo executivo, decisões-chave |

Sintaxe:

```
::: callout {icon="🎯" color="blue_bg"}
	Conteúdo do callout. Use tab para indentar.
	- Bullets dentro do callout também indentados com tab.
:::
```

---

#### Tabelas

```
<table fit-page-width="true" header-row="true">
<tr>
<td>**Coluna 1**</td>
<td>**Coluna 2**</td>
</tr>
<tr>
<td>Dado 1</td>
<td>Dado 2</td>
</tr>
</table>
```

---

#### Blockquotes (perguntas de validação)

```
> Pergunta de validação ou hipótese a confirmar com o usuário.
```

---

#### Divider

Separe todas as seções principais com `---`:

```
---
```

---

#### Mermaid (flowchart)

```
graph LR

A["Início"] --> B["Etapa 1"]

B --> C["Etapa 2"]

C --> D["Fim"]
```

---

#### Mermaid (stateDiagram — usado nos Docs 06 e 12)

```
stateDiagram-v2

[*] --> Inativo

Inativo --> Ativo: Cadastro

Ativo --> Pausado: Inatividade

Pausado --> Ativo: Reativação

Ativo --> Cancelado: Cancelamento

Cancelado --> [*]
```

---

#### Toggles

```
<details>
<summary>Título do toggle</summary>
	Conteúdo interno (indentado com tab).
</details>
```

---

#### Referências cruzadas

Link local (usar durante geração — antes da importação no Notion):

```
[Nome do Doc](./XX - Nome do Arquivo.md)
```

Mention-page (preencher após importação no Notion):

```
<mention-page url="https://www.notion.so/ID_DA_PAGINA"/>
```

---

#### Elementos Tipográficos

```
**negrito** e *itálico*

- Item 1
- Item 2
	- Sub-item (indentado com tab)

1. Item 1
2. Item 2
	1. Sub-item (indentado com tab)

Texto com **destaque**{color="blue"}

## Título da Seção {color="blue"}
```

---

#### Regras Gerais

- **Espaçamento:** sempre uma linha em branco entre parágrafos. Nunca blocos de texto compactos.
- **Hierarquia:** use `1.1`, nunca `1.a`.
- **TL;DR:** obrigatório em todos os docs, exceto puramente técnicos (listas, glossários).
- **`⚙️ orange_bg`:** use para dependências de sistema, requisitos de performance, integrações técnicas.
- **Células vazias em tabelas:** preencha com "N/A" — nunca deixe vazio.
- **Separadores `---`:** entre todas as seções principais do documento.
---

---

# SUB-PROMPTS — Referência por Documento

Ao gerar cada documento, leia o prompt completo do arquivo correspondente em `02 - Prompts/01 - Produto/`. Assuma a persona indicada no prompt. Após gerar, volte à persona de orquestrador para registrar o progresso.

| Doc | Arquivo do Prompt | Web Search | Tamanho-alvo |
| --- | --- | --- | --- |
| 01 | `02 - Prompts/01 - Produto/P01 — Pesquisa de Mercado e Benchmark.md` | Sim | 4.000–6.000 |
| 02 | `02 - Prompts/01 - Produto/P02 — Análise Geral.md` | Não | 4.000–6.000 |
| 03 | `02 - Prompts/01 - Produto/P03 — Concorrentes.md` | Sim | 3.000–5.000 |
| 04 | `02 - Prompts/01 - Produto/P04 — One-Liner e ICPs.md` | Não | 3.000–4.000 |
| 05 | `02 - Prompts/01 - Produto/P05 — Manifesto da Marca.md` | Não | 2.000–3.000 |
| 06 | `02 - Prompts/01 - Produto/P06 — Memorando de Essência.md` | Não | 4.000–6.000 |
| 07 | `02 - Prompts/01 - Produto/P07 — Tom de Voz e Identidade Verbal.md` | Não | 3.000–4.000 |
| 08 | `02 - Prompts/01 - Produto/P08 — Social Media.md` | Sim | 2.500–3.500 |
| 09 | `02 - Prompts/01 - Produto/P09 — Design Thinking.md` | Não | 3.000–4.000 |
| 10 | `02 - Prompts/01 - Produto/P10 — Jobs To Be Done.md` | Não | 3.000–4.000 |
| 11 | `02 - Prompts/01 - Produto/P11 — Service Design.md` | Não | 2.500–3.500 |
| 12 | `02 - Prompts/01 - Produto/P12 — Casos de Uso.md` | Não | 3.000–4.500 |
| 13 | `02 - Prompts/01 - Produto/P13 — UX Writing.md` | Não | 2.500–3.500 |
| 14 | `02 - Prompts/01 - Produto/P14 — Modelo de Negócios.md` | Sim | 3.000–4.500 |
| 15 | `02 - Prompts/01 - Produto/P15 — Proposta de Valor.md` | Sim | 3.000–4.500 |
| 16 | `02 - Prompts/01 - Produto/P16 — Pricing Book Interno.md` | Sim | 3.500–5.500 |

**Instrução de execução:** Para cada doc, o Claude Code deve:
1. Ler o arquivo de prompt correspondente do disco
2. Assumir a persona indicada no prompt
3. Gerar o conteúdo seguindo as seções obrigatórias do prompt
4. Verificar os Exit Criteria do prompt antes de gravar
5. Voltar à persona de orquestrador

---

## Processo de Execução

### Etapa 2 — Geração dos 16 docs no disco

> **Disk-first:** Todos os docs são gerados e gravados no disco local (`docs/01 - Produto/`). Não há publicação externa — os arquivos `.md` são o produto final do pipeline.

> **Paralelização opcional:** Pares paralelizáveis: Doc 05 ∥ Doc 09 (ambos dependem do Doc 04); Doc 13 ∥ Doc 14 (dependências distintas). Caso contrário, seguir ordem sequencial ótima: `01 → 02 → 03 → 04 → 05 → 09 → 06 → 10 → 07 → 11 → 08 → 12 → 13 → 14 → 15 → 16` (intercala branches para desbloquear dependentes mais cedo).

Para CADA doc, na ordem de dependência:

**2.1.** Leia `logs/progresso-pipeline-produtos.json` do disco. Verifique `etapa_atual`.

> **Proteção contra sobrescrita:** Antes de gerar qualquer doc, verifique `docs[XX].status` no JSON.
> - Se `status == "gerado"` → **pule este doc automaticamente** e avance para o próximo. Não regere nem sobrescreva.
> - Se `status == "pendente"` ou o campo não existir → prossiga normalmente.
> - **Exceção:** Se o usuário incluir o termo `--forçar` ou `--force` na instrução original, ignore esta proteção e regere todos os docs (incrementando a versão: vX.0 → v(X+1).0).

**2.2.** Leia de `logs/resumos.json` **apenas os resumos das dependências diretas e de segundo nível** (não todos). Exceção: Doc 16 carrega todos.

**2.3.** Leia do disco o arquivo de prompt correspondente ao doc atual (ver tabela em "SUB-PROMPTS — Referência por Documento"). Não carregue prompts de outros docs.

**2.4.** Leia do disco (`docs/01 - Produto/`) os docs de dependência completos. O briefing está sempre disponível em `docs/01 - Produto/00 - Briefing.md`.

> **Verificação de pendências em cascata:** Antes de gerar, verifique em `logs/resumos.json` se alguma dependência direta tem `"tem_pendencias": true`. Se sim, insira no topo do doc gerado (logo após o header):
>
> ```
> ::: callout {icon="⚠️" color="red_bg"}
> **Atenção — Dependências com pendências:** Este doc depende de [Doc XX — Nome] que possui [N] pendência(s) não resolvidas. Revise e atualize após correção manual dos docs anteriores.
> :::
> ```

**2.5.** Assuma a PERSONA do sub-prompt correspondente. Gere o conteúdo completo seguindo o prompt, a persona, os anti-exemplos e o tom de escrita.

**2.6.** Valide contra o checklist de qualidade.

**2.7.** **Verifique Exit Criteria** do arquivo de prompt deste doc (lido no passo 2.3). Cada item deve ser ✅. Se algum falhar, corrija e revalide. Máx 1 iteração. Se não resolver, marque `[PENDENTE — REVISÃO MANUAL]` e prossiga.

**2.8.** **Grave o doc no disco** em `docs/01 - Produto/[XX] - [Nome do Arquivo].md`. Exemplo: `01 - Pesquisa de Mercado e Benchmark.md`. Formatação Notion Markdown (callouts, tabelas, toggles, mermaid). Só grava após passar nos Exit Criteria.

**2.9.** Atualize `logs/progresso-pipeline-produtos.json` no disco:

- `docs[XX].status` = "gerado"
- `docs[XX].arquivo_local` = caminho do arquivo no disco
- `etapa_atual` = próximo doc
- `ultima_atualizacao` = agora

**2.10.** Acrescente (append-only — nunca sobrescreva resumos anteriores) em `logs/resumos.json` o resumo do doc em formato JSON estruturado:

```json
{
  "doc": "XX",
  "nome": "[Nome]",
  "hash": "[MD5 do conteúdo]",
  "decisoes_chave": ["...", "..."],
  "dados_quantitativos": ["TAM: R$ X [Fonte: Statista, 2024]", "CAGR: Y% [Fonte: CB Insights, 2023]"],
  "termos_definidos": ["termo1", "termo2"],
  "icps_personas": ["ICP 1", "ICP 2"],
  "features_capacidades": ["feature1", "feature2"],
  "pendencias": ["[DADO PENDENTE: seção X — métrica Y sem fonte]"],
  "tem_pendencias": false,
  "pendencias_count": 0,
  "conexoes_proximos": ["Doc XX usa Y deste doc"]
}
```

**Regras do campo `tem_pendencias`:**
- `true` se o doc tiver 1 ou mais ocorrências de `[DADO PENDENTE: ...]` ou `[SEÇÃO COM DADOS LIMITADOS]` ou `[DEPENDÊNCIA PARCIAL]`
- `false` somente se o doc não tiver nenhuma pendência
- `pendencias_count` = número total de marcações de pendência no doc

**2.10b.** Atualize `logs/glossario-global.md`: para cada termo novo ou redefinido neste doc, adicione ou atualize a entrada na tabela. Formato por linha:

```markdown
| [Termo] | [Definição concisa] | Doc XX | Docs XX, YY, ZZ |
```

**Regras do glossário:**

- Se o termo já existe com definição diferente, atualize a definição e registre o doc de origem mais recente.
- Coluna "Docs que usam" é cumulativa: adicione o doc atual se ainda não está listado.
- Termos técnicos em inglês mantêm a grafia original com tradução entre parênteses na definição.
- Mínimo esperado ao final do pipeline: 30 termos.

**2.11.** Atualize o progresso via **TodoWrite**:

- No início da Etapa 2 (antes do primeiro doc), crie um TodoWrite com todos os 16 docs a gerar. Exemplo de items: `"Gerar Doc 01 — Pesquisa de Mercado e Benchmark"`, `"Gerar Doc 02 — Análise Geral"`, etc.
- Ao iniciar cada doc, marque-o como `in_progress` (apenas um item `in_progress` por vez).
- Ao concluir a gravação no disco + atualização dos logs, marque como `completed`.
- Não use barras de progresso, logs no terminal, nem JSON de progresso visual. O TodoWrite é o único mecanismo de feedback de progresso.

**2.12.** Passe para o próximo doc. Repita 2.1 a 2.11.

### Etapa 3 — Auditoria Unificada (ANTES do Relatório Final)

> **Fluxo:** A auditoria roda após a Etapa 2 em uma **única passada** por doc. Todas as correções são aplicadas diretamente nos arquivos locais.

Após a Etapa 2 (16 docs gerados no disco), execute nesta ordem:

1. **3.1** Auditoria Unificada (passada única: consistência + qualidade)
2. **3.2** Página-índice (preparar no disco)

Depois, siga para a **Etapa 4** (Relatório Final).

### 3.1 Auditoria Unificada — Passada Única

Releia todos os 16 documentos gerados **do disco** (`docs/01 - Produto/`), Docs 01 a 16. **Uma passada, dois eixos por doc.** Não releia o doc duas vezes — ao abrir cada doc, avalie ambos os eixos simultaneamente.

**Para cada doc, verificar:**

**Eixo A — Consistência Factual:**

1. **Dados:** Números, métricas, nomes de concorrentes, TAM/SAM/SOM, ICPs — idênticos entre docs.
2. **Terminologia:** Mesmos conceitos usam mesmos termos (validar contra `logs/glossario-global.md`).
3. **Referências cruzadas:** Dados citados existem exatamente como descritos nos docs-fonte.
4. **Tom e identidade:** Tom de voz (Doc 07) consistente com UX Writing (Doc 13), Manifesto (Doc 05), Social Media (Doc 08).
5. **ICPs e personas:** Definições do Doc 04 idênticas em JTBD (Doc 10), Service Design (Doc 11), Casos de Uso (Doc 12).
6. **Modelo de negócio:** Doc 14 consistente com Pricing (Doc 16) e Proposta de Valor (Doc 15).
7. **Gaps:** ICP sem caso de uso? Job sem solução? Feature sem pricing?

**Eixo B — Qualidade:**

1. **Precisão cirúrgica:** Zero ambiguidade remanescente.
2. **Profundidade analítica:** Análise ou só enumeração? Decisões justificadas?
3. **Exemplos concretos:** Suficientes? Anti-exemplos do pipeline respeitados?
4. **Acionabilidade:** PM, designer, fundador, vendedor consegue agir sem perguntar?
5. **Diferenciação:** Específico ao produto (se trocar o nome e ainda funciona, está genérico demais).
6. **Dados e evidências:** Claims fundamentadas, métricas com números.
7. **Experiência de leitura:** Callouts, tabelas e diagramas Mermaid efetivos.

**Mentalidade de leitura:** Cada doc deve funcionar como se fosse apresentado a um investidor série A, avaliado por uma consultoria estratégica e usado por 20 pessoas no dia 1.

**Ação:** Corrija TUDO nos arquivos locais em `docs/01 - Produto/`. Melhore apenas onde identificar ganho real — não adicione prosa por obrigação.

**Scoring — Rubrica Explícita:**

Cada doc recebe score combinado (0–100) calculado assim:

| Critério | Eixo | Peso | 0 pts | 1 pt | 2 pts |
| --- | --- | --- | --- | --- | --- |
| Dados consistentes entre docs | A | 15 | ≥1 inconsistência grave | 1 discrepância menor | Zero inconsistências |
| Terminologia alinhada ao glossário | A | 10 | ≥2 desvios | 1 desvio | Alinhamento total |
| Referências cruzadas corretas | A | 10 | Referência incorreta | Imprecisão menor | Todas corretas |
| Tom de voz consistente | A | 5 | Fora do tom | Parcialmente | Dentro do tom |
| ICPs/personas idênticos ao Doc 03 | A | 10 | Divergência material | Divergência menor | Idênticos |
| Modelo de negócio consistente | A | 10 | Contradiz Docs 14/16 | Ambiguidade | Consistente |
| Precisão — zero ambiguidade | B | 10 | ≥2 ambiguidades | 1 ambiguidade | Zero ambiguidade |
| Profundidade analítica | B | 10 | Só enumeração | Análise parcial | Análise justificada |
| Exemplos concretos | B | 5 | Ausentes | Poucos/genéricos | Suficientes e específicos |
| Acionabilidade | B | 10 | Não acionável | Parcialmente | Totalmente acionável |
| Diferenciação (específico ao produto) | B | 10 | Genérico demais | Parcialmente específico | 100% específico |
| Dados com fontes / marcações | B | 5 | ≥1 métrica sem fonte/marcação | Maioria com fonte | Todas com fonte ou marcadas |

**Score do doc** = Σ (pontos obtidos × peso) / 200 × 100

**Meta: ≥ 95% global** (média dos 16 docs gerados). Docs com score < 80% exigem revisão imediata antes de avançar. Docs entre 80-94% recebem nota `⚠️ Revisão Recomendada` no relatório final.

Exemplo: doc com todos os critérios em 2 pts = 200/200 = 100%. Doc com todos em 1 pt = 100/200 = 50%.

**Gestão de contexto:** Se o contexto apertar, divida em Bloco A (Docs 01-08) e Bloco B (Docs 09-16). Audite cada bloco separadamente.

Grave resultado em `logs/fase-1/auditoria-unificada.md` usando o template abaixo:

```markdown
# Auditoria Unificada — [Nome do Produto]

**Gerada em:** [DD/MM/AAAA HH:MM — America/Fortaleza]
**Pipeline versão:** v10.5
**Docs auditados:** [X]/16
**Score global:** [X]% ([status: ✅ ≥95% | ⚠️ 80-94% | ❌ <80%])

---

## Resultado por Documento

| Doc | Nome | Score Eixo A | Score Eixo B | Score Final | Status | Ações tomadas |
| --- | --- | --- | --- | --- | --- | --- |
| 01 | Pesquisa de Mercado | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 02 | Análise Geral | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 03 | Concorrentes | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 04 | One-Liner e ICPs | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 05 | Manifesto da Marca | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 06 | Memorando de Essência | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 07 | Tom de Voz | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 08 | Social Media | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 09 | Design Thinking | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 10 | Jobs To Be Done | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 11 | Service Design | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 12 | Casos de Uso | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 13 | UX Writing | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 14 | Modelo de Negócios | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 15 | Proposta de Valor | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |
| 16 | Pricing Book | [X]% | [X]% | [X]% | ✅/⚠️/❌ | [correções ou "Nenhuma"] |

---

## Inconsistências Encontradas

> Liste apenas as inconsistências identificadas durante o Eixo A. Se nenhuma, escreva "Nenhuma inconsistência encontrada."

- [Inconsistência 1: doc X vs doc Y — descrição]
- [Inconsistência 2: ...]

## Pendências Globais

> Total de `[DADO PENDENTE: ...]` e `[SEÇÃO COM DADOS LIMITADOS]` encontrados após a auditoria.

| Doc | Tipo de pendência | Descrição |
| --- | --- | --- |
| [XX] | DADO PENDENTE / DADOS LIMITADOS | [descrição] |

**Total:** [N] pendências em [M] docs.

## Ações de Correção Aplicadas

> Liste todas as correções feitas nos arquivos locais durante a auditoria.

- Doc [XX]: [descrição da correção]

## Próximos Passos Manuais

> Itens que requerem intervenção humana antes ou após o import para Notion.

- [ ] [Ação necessária 1]
- [ ] [Ação necessária 2]
```

Atualize `logs/progresso-pipeline-produtos.json`: `auditorias.unificada` = "concluida", `etapa_atual` = "relatorio-final".

### 3.2 Página-índice (preparar no disco)

Prepare o conteúdo da página-índice e grave em `docs/01 - Produto/indice.md`.

Título: "Índice — [Nome do Produto]". Sem propriedades Fase/Área (é página avulsa, não item do database).

Conteúdo:

```
<table fit-page-width="true">
<tr><td>**Produto**</td><td>[nome]</td></tr>
<tr><td>**Empresa**</td><td>[empresa]</td></tr>
<tr><td>**Gerado em**</td><td>[data]</td></tr>
<tr><td>**Total de docs**</td><td>[X]/17 (00 = Briefing + 16 gerados)</td></tr>
<tr><td>**Pendências**</td><td>[Y] DADO PENDENTE | [Z] DADOS LIMITADOS</td></tr>
<tr><td>**Status**</td><td>🟢 Completo | ⚠️ [N] docs com pendências</td></tr>
</table>

## Entrada
- [00 — Briefing](./00%20-%20Briefing.md) (Input — preenchido pelo usuário)

## 1 — Discovery
1. [01 — Pesquisa de Mercado e Benchmark](./01%20-%20Pesquisa%20de%20Mercado%20e%20Benchmark.md) (Mercado)
2. [02 — Análise Geral](./02%20-%20Análise%20Geral.md) (Estratégia)
3. [03 — Concorrentes](./03%20-%20Concorrentes.md) (Mercado)
4. [04 — One-Liner e ICPs](./04%20-%20One-Liner%20e%20ICPs.md) (Marketing)

## 2 — Identidade
5. [05 — Manifesto da Marca](./05%20-%20Manifesto%20da%20Marca.md) (Branding)
6. [06 — Memorando de Essência](./06%20-%20Memorando%20de%20Essência.md) (Branding)
7. [07 — Tom de Voz e Identidade Verbal](./07%20-%20Tom%20de%20Voz%20e%20Identidade%20Verbal.md) (Branding)
8. [08 — Social Media](./08%20-%20Social%20Media.md) (Marketing)

## 3 — Produto
9. [09 — Design Thinking](./09%20-%20Design%20Thinking.md) (Produto)
10. [10 — Jobs To Be Done](./10%20-%20Jobs%20To%20Be%20Done.md) (Produto)
11. [11 — Service Design](./11%20-%20Service%20Design.md) (Produto)
12. [12 — Casos de Uso](./12%20-%20Casos%20de%20Uso.md) (Produto)
13. [13 — UX Writing](./13%20-%20UX%20Writing.md) (UI/UX)

## 4 — Negócio
14. [14 — Modelo de Negócios](./14%20-%20Modelo%20de%20Negócios.md) (Estratégia)
15. [15 — Proposta de Valor](./15%20-%20Proposta%20de%20Valor.md) (Comercial)
16. [16 — Pricing Book Interno](./16%20-%20Pricing%20Book%20Interno.md) (Comercial)

## Mapa de Dependências

```

graph LR

B["Doc 00 — Briefing"] --> D1["Doc 01"]

D1 --> D2["Doc 02"]

D2 --> D3["Doc 03"]

D3 --> D4["Doc 04"]

D4 --> D5["Doc 05"]

D5 --> D6["Doc 06"]

D6 --> D7["Doc 07"]

D4 --> D9["Doc 09"]

D7 --> D8["Doc 08"]

D9 --> D10["Doc 10"]

D10 --> D11["Doc 11"]

D11 --> D12["Doc 12"]

D7 --> D13["Doc 13"]

D12 --> D13

D4 --> D14["Doc 14"]

D6 --> D14

D6 --> D15["Doc 15"]

D14 --> D15

D4 --> D16["Doc 16"]

D6 --> D16

D14 --> D16

D15 --> D16

```

```

### Relatório Final (Etapa 4)

Grave em `logs/fase-1/relatorio-final.md` e apresente ao usuário no terminal:

```
Pipeline de Produtos — [Nome do Produto] — Completo ✅

Resumo:
- Docs gerados: [X]/16 (Doc 01 a Doc 16)
- Doc 00 (Briefing): preexistente (input do usuário)
- Docs com pendências: [lista e motivo]
- Tempo total de execução: [X]h

Auditoria Unificada:
- Correções aplicadas: [X]
- Score global: [X]%
- Top 3 docs com mais correções: [listar]

Arquivos gerados em docs/01 - Produto/:
- 00 - Briefing.md (preexistente)
- 01 - Pesquisa de Mercado e Benchmark.md
- 02 - Análise Geral.md
- ... (16 docs gerados + indice.md)

Arquivos de controle:
- logs/progresso-pipeline-produtos.json
- logs/resumos.json
- logs/glossario-global.md
- logs/auditoria-unificada.md
- logs/relatorio-final.md
```

Atualize `logs/progresso-pipeline-produtos.json`: `etapa_atual` = "concluido".

---

## Adaptações por Tipo de Produto

Com base na classificação da Etapa 1.3:

**2 stakeholders:** Remova referências a "terceiro stakeholder" nos Docs 04 (ICPs), 05 (Promessas), 06 (Capacidades por stakeholder), 09 (Personas), 10 (Jobs por Ator), 11 (Blueprint), 12 (Casos por ICP), 15 (Canvas e Argumento por Stakeholder). Adapte promessas, propostas de valor e casos de uso para 2 atores.

**B2C puro:** Doc 04 (ICPs) usa perfis demográficos em vez de empresariais. Doc 08 (Social Media) prioriza canais B2C (redes sociais, influencers). Docs 14-16 (Negócio) removem tier "enterprise" e simplificam modelo para pessoa física.

**Plano único:** Docs 10 (Jobs × Features × Planos), 12 (Caso × Ator × Feature × Plano), 14 (Modelo de Receita) e 15 (Tabela de Planos) removem colunas "por plano". Doc 16 (Pricing Book) simplifica para plano único sem upsell entre tiers.

**Marketplace:** Docs 04 (ICPs), 10 (JTBD), 11 (Service Design) e 14 (Modelo de Negócios) modelam dois lados (oferta e demanda) separadamente. Docs 01 (Pesquisa) e 02 (Análise) incluem dinâmicas de efeitos de rede. Doc 14 modela unit economics por lado do marketplace.

---

## Pipeline Completo — 4 Fases

```
Fase 1: Documentação de Produto        ← VOCÊ ESTÁ AQUI → roda auditoria A01 no final
Fase 2: Documentação de Desenvolvimento → roda auditoria A02 no final
Fase 3: Documentação das Sprints        → roda auditoria A03 no final
Fase 4: Desenvolvimento (Coding)        → roda auditoria A04 no final
```

## Etapa Final — Auditoria A01

Após gerar e auditar internamente todos os 16 docs, execute a auditoria externa A01:

1. Leia o prompt completo em `02 - Prompts/03 - Auditorias/A01 - Auditoria Fase 1 - Doc Produto.md`
2. Execute os 3 eixos de auditoria: Qualidade Individual + Consistência Cross-Doc + Cobertura RN
3. Aplique todas as correções diretamente nos docs
4. Salve relatório em `docs/05 - Auditorias/AUDIT_FASE1_PRODUTO.md`

**GATE:** Score global ≥ 95% e zero findings P0/P1 → aprovado para Fase 2.
Se reprovado → corrigir e re-auditar até aprovação.

## Handoff → Fase 2

Ao concluir todas as etapas (incluindo relatório final), execute o procedimento de handoff:

1. Confirme que todos os docs estão gravados em `docs/01 - Produto/`
2. Confirme que `logs/relatorio-final.md` está gerado
3. Execute:
   ```bash
   cp claude-fase-2.md CLAUDE.md
   ```
4. Exiba a mensagem de encerramento:
   ```
   ✅ Fase 1 concluída.
   Docs gerados: docs/01 - Produto/
   CLAUDE.md atualizado para Fase 2.
   Para continuar: "Leia o CLAUDE.md e inicie a Fase 2 — Documentação de Desenvolvimento."
   ```

---

## Changelog

| Data | Versão | Descrição |
| --- | --- | --- |
| 22/03/2026 | v10.7 | **Reestruturação para 4 fases.** (1) Framework reestruturado de 6 fases para 4 fases: Fase 1 (Produto), Fase 2 (Desenvolvimento), Fase 3 (Sprints), Fase 4 (Desenvolvimento/Coding). Fases antigas 4 (Auditoria Docs), 5 (Desenvolvimento) e 6 (Auditoria Dev) eliminadas — conteúdo absorvido nas novas fases. (2) Seção "Etapa Final — Auditoria A01" adicionada antes do handoff: executa auditoria externa A01 com gate de aprovação (≥ 95%, zero P0/P1). (3) Diagrama de fases atualizado para 4 fases com auditorias integradas. |
| 22/03/2026 | v10.6 | **Correções de consistência.** (1) H1 redundante "02 - Documentação de Produto" removido. (2) Template de changelog corrigido: v9.0→v10.6. (3) Estrutura de pastas alinhada com o framework: removidas pastas inexistentes (04 - Auditoria, 05 - Código, 06 - Auditoria Dev), adicionadas pastas reais (docs/05 - Auditorias/, src/). (4) URL malformada em mention-page corrigida. (5) Changelog reordenado cronologicamente (descendente). |
| 19/03/2026 | v10.5 | **Reestruturação de etapas e numeração de docs + 4 correções.** (1) Briefing movido de `00 - Briefing.md` na raiz para `docs/01 - Produto/00 - Briefing.md` — agora é o Doc 00 dentro da mesma pasta dos demais docs. (2) Etapas renumeradas: 0→1, 1→2, 1.5→3, 3→4. Sub-passos atualizados (0.X→1.X, 1.X→2.X, 1.5.X→3.X). (3) Docs renumerados: 00-16. Total = 17 docs (1 preexistente + 16 gerados). Referências cruzadas atualizadas. (4) Bug de lógica corrigido: passos 2.7 e 2.8 invertidos — Exit Criteria agora vem ANTES de gravar. (5) Título da Etapa 3 corrigido: "ANTES da Publicação" → "ANTES do Relatório Final". (6) Backup note corrigida: "Auditoria→Publicação" → "Auditoria→Relatório". (7) Doc 00 (Briefing) adicionado ao `logs/progresso-pipeline-produtos.json` com `status: "preexistente"`. |
| 09/03/2026 | v10.4 | **Alinhamento com Fase 4.** Output path atualizado de `pipeline-produtos/docs/` para `docs/01 - Produto/`. Logs movidos para `logs/fase-1/`. Diagrama de estrutura de arquivos reescrito para refletir layout unificado `docs/`. |
| 08/03/2026 | v10.3 | **5 melhorias para 10/10 (exceto template de briefing).** (1) Passo 0.2: checklist de validação estruturado com campos bloqueantes vs. recomendados, impacto por campo ausente e saída obrigatória no terminal. (2) Passo 1.1: proteção contra sobrescrita — docs com `status: "gerado"` são pulados; `--force` necessário para reger. (3) Passo 1.5: rubrica de scoring explícita com 12 critérios, pesos e escala 0-2 pts. Meta ≥95%; docs <80% exigem revisão imediata. (4) `[DADO PENDENTE]` padronizado para `[DADO PENDENTE: descrição do dado faltante]` em todos os pontos do pipeline. (5) Template completo para `logs/auditoria-unificada.md`: score por doc, inconsistências, pendências globais, correções aplicadas e próximos passos manuais. |
| 08/03/2026 | v10.2 | **Rastreamento de pendências em cascata + fontes obrigatórias.** (1) Campos `tem_pendencias` e `pendencias_count` adicionados ao schema de `logs/resumos.json`. Step 1.4 verifica dependências com pendências e insere callout de aviso no topo do doc afetado. Checklist atualizado para validar esses campos. (2) Toda métrica quantitativa agora exige fonte inline `[Fonte: X, AAAA]` — sem fonte verificável = `[DADO PENDENTE]` obrigatório. Regra reforçada na seção Web Search, no tratamento de erros e no `claude-fase-1.md`. |
| 08/03/2026 | v10.1 | **Briefing 100% local.** Passo 0.2 agora lê `00 - Briefing.md` da raiz do projeto em vez do Notion MCP. Removidos: seção "Notion MCP indisponível", referências "do Notion" na gestão de contexto, `briefing_url` do JSON de progresso. Sintaxe renomeada: "Referência Notion Markdown" → "Referência Markdown ShiftLabs". `mention-page` substituído por links markdown locais. Pipeline agora opera com zero dependência de Notion ou MCP. |
| 08/03/2026 | v10.0 | **Remoção da publicação no Notion.** Passo 2 (publicação batch) eliminado. Pipeline agora é 100% local: docs ficam em `docs/01 - Produto/`. Removidos: Passos 0.4 (leitura do database), 0.5 (duplicatas), tratamento de rate limiting/timeout Notion, seção de falha de publicação. Atualizados: JSON de progresso (sem `database_url`, `data_source_id`, `mapeamento_fase/area`), página-índice (links markdown locais), `etapa_atual` pós-auditoria ("relatorio-final"). |
| 04/03/2026 | v9.0 | **Otimização de velocidade e performance.** (1) Disk-first, Publish-batch: geração e auditoria no disco, Notion MCP apenas para briefing e publicação batch. (2) Auditoria unificada: 3 passadas → 1 passada única (consistência + qualidade simultâneos). (3) Resumos em JSON estruturado (`logs/resumos.json`). (4) Exit Criteria binários em cada prompt. (5) Ordem sequencial ótima explícita. (6) Resumos seletivos (dependências diretas + 2º nível). Tempo estimado: ~10-14h (vs. 16-24h na v8.4). |
| 28/02/2026 | v8.4 | 10 melhorias para 100/100. Estrutural: regras de versionamento para re-execução do pipeline; read-back check (Passo 1.7b) para validar formatação pós-publicação; callout de paralelização com pares Doc 5∥9 e Doc 13∥14 (~30% ganho); fix encoding de mention-page. Robustez: re-verificação em cascata nas auditorias (corrigir Doc X → re-verificar dependentes); critério de completude por seção (≥5%) + sistema de scoring quantificado (5 critérios, 0-2, máx 10); fallback completo para Notion MCP indisponível (gerar .md local, status "gerado-local", batch publish). Polish: template de resumo estruturado (campos: dados quantitativos, termos definidos, ICPs/personas, features/capacidades, pendências); glossário global cross-doc (`logs/glossario-global.md` com regras de atualização cumulativa, mín 30 termos); diretório `docs/` para fallback local. |
| 28/02/2026 | v8.3 | 12 melhorias para 10/10. Estrutura: Passo 2 reestruturado (2.1–2.5 numerados sequencialmente); fix web search fase Docs 14-16. Robustez: threshold de contexto (~150k tokens); tratamento de rate limiting/timeout do Notion MCP; fallback para web search indisponível; backup automático do JSON de progresso; threshold mínimo de briefing. Polish: nota de encoding para code blocks; tempo estimado por fase no TL;DR; status visual na página-índice; validação Mermaid detalhada no checklist. |
| 28/02/2026 | v8.2 | Auditoria completa do script. Correções: lista de docs com web search (TL;DR e corpo) → Docs 1, 3, 8, 14, 15, 16; estimativa de palavras → 49.000–71.000; cores de Área no Passo 0.4; template de Changelog atualizado para HTML; lógica de duplicatas no Passo 1.7 (update-page se existe, create-pages se novo); tratamento de falha na leitura de dependências; lembretes de gestão de contexto nas 3 auditorias; ação corretiva na checklist de tamanho; resumos como append-only; clarificação da página-índice (sem Fase/Área); adaptações por tipo com referências a docs e seções específicas. |
| 27/02/2026 | v8.0–v8.1 | Adaptação para Claude Code Desktop + modo fixo Completo + Autonomia Total. |
| 27/02/2026 | v7.0 | Versão estável com orquestração Notion MCP, web search bilíngue, três auditorias, modo autônomo/interativo, adaptações por tipo de produto, gestão de contexto progressivo. |
| 27/02/2026 | v1.0–v6.0 | Versões anteriores. Evolução incremental do pipeline de 16 documentos de produto. |
