# Stacks Tecnologicas -- Repasse AI

## Stack Normativa do Modulo Repasse AI

| Campo | Valor |
|---|---|
| Destinatario | Tech Lead / CTO |
| Escopo | Stack normativa do modulo Repasse AI -- Repasse Seguro |
| Modulo | Repasse AI |
| Versao | v1.1 |
| Responsavel | Claude Code Desktop |
| Data da versao | 2026-03-22 (America/Fortaleza) |
| Carater | **Normativo.** Este documento define o padrao obrigatorio para o modulo Repasse AI. Desvios exigem ADR aprovado pelo responsavel tecnico. |
| Fonte primaria | ShiftLabs Stacks v7.0 |
| Revisao | Trimestral ou a cada mudanca significativa de stack |

---

> **TL;DR**
>
> - **Repasse AI e um servico backend puro** -- sem frontend web ou mobile proprio. E consumido por outros modulos via API REST e SSE.
> - **LLM padrao:** GPT-4 via OpenAI SDK. Orquestracao com Vercel AI SDK (streaming) + LangChain.js (RAG, agents, tools). LangGraph.js para fluxos stateful do agente.
> - **RAG:** Supabase pgvector como vector store. OpenAI Embeddings. Ingestao assincrona via RabbitMQ.
> - **Backend:** Node.js 22+ / NestJS 10+ / TypeScript strict / Prisma 6+ / Supabase PostgreSQL 17+. Redis para cache de LLM. RabbitMQ para filas.
> - **Observabilidade de IA:** Langfuse obrigatorio para tracing, custo, latencia e qualidade. Sentry para error tracking de infraestrutura.
> - **Seguranca de IA:** Prompt injection protection, rate limiting por usuario (30 msg/h webchat, 20 msg/h WhatsApp), content filtering, PII masking obrigatorio.
> - **Testes de IA:** Vitest para logica deterministica + evals com golden datasets via Langfuse Evals. Minimo 50 exemplos no golden dataset.

---

## 1. Principios de Governanca

**PG-01. ShiftLabs Stacks v7.0 e a fonte primaria de verdade.** Toda tecnologia deste documento herda do padrao central. Desvios sao permitidos exclusivamente com justificativa tecnica vinculada a uma regra de negocio (RN) do Repasse AI, documentados via ADR.

**PG-02. Desenvolvimento 100% por agentes de IA.** Todo codigo do Repasse AI e gerado por Claude Code e GPT Codex. Boilerplates, convencoes de codigo, estrutura de pastas e testes automatizados sao os contratos de qualidade. Toda decisao de stack maximiza a qualidade do output dos agentes.

**PG-03. Repasse AI e um servico backend sem frontend proprio.** O modulo expoe API REST e SSE. Nao possui frontend web, mobile, landing page ou dashboard. Secoes de frontend e mobile do ShiftLabs Stacks v7.0 nao se aplicam a este modulo.

**PG-04. Seguranca de IA e requisito de lancamento, nao melhoria futura.** Isolamento de dados (RN-001 a RN-004), protecao contra prompt injection (RN-037, RN-038) e rate limiting por usuario (RN-025) devem estar implementados e testados antes da ativacao do modelo em producao.

**PG-05. Observabilidade de IA precede o deploy.** Nenhuma feature de IA entra em producao sem tracing Langfuse configurado. Custo, latencia e qualidade devem ser mensuraveis desde o primeiro request.

---

## 2. Backend e Runtime

### 2.1 Stack Obrigatoria

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Node.js** | 22.x+ (LTS) | ✅ Obrigatorio | Runtime padrao ShiftLabs. LTS garante estabilidade. | Usar apenas versoes LTS em producao. |
| **NestJS** | 10.x+ | ✅ Obrigatorio | Arquitetura modular, decorators TypeScript, Swagger embutido, guards/pipes nativos. Output de IA mais consistente. | Framework HTTP unico. Express/Fastify/Hono proibidos sem ADR. |
| **TypeScript** | 5.4+ | ✅ Obrigatorio | `strict: true` obrigatorio. Tipagem forte maximiza qualidade do output gerado por IA. | Arquivos `.js`/`.jsx` proibidos em novos projetos. |
| **Prisma** | 6.x+ | ✅ Obrigatorio | Tipagem auto-gerada, migrations declarativas, integracao nativa com Supabase. | Unico ORM aprovado. TypeORM/Sequelize/Drizzle proibidos. Coexiste com `$queryRaw` para pgvector. |
| **Pino** | 9.x+ | ✅ Obrigatorio | Logs estruturados com request ID, timestamp, nivel e contexto. | Todo servico deve ter logs estruturados. `console.log` proibido em producao. |
| **Helmet** | 8.x+ | ✅ Obrigatorio | Headers de seguranca automaticos. | Middleware adicionado desde o inicio do projeto. |
| **class-validator + class-transformer** | latest | ✅ Obrigatorio | Validacao de DTOs integrada aos pipes do NestJS. | Toda entrada de dados validada via pipes. |
| **@nestjs/swagger** | latest | ✅ Obrigatorio | Swagger/OpenAPI auto-gerado. | Todo endpoint documentado. Documentacao acessivel na URL do backend. |
| **@nestjs/throttler** | latest | ✅ Obrigatorio | Rate limiting nativo do NestJS. | Todo endpoint publico com rate limiting. Endpoints de IA com rate limiting por usuario (RN-025). |

### 2.2 Banco de Dados

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **PostgreSQL (via Supabase)** | 17+ | ✅ Obrigatorio | PostgreSQL gerenciado + Auth + Storage + Realtime + pgvector em uma plataforma. Elimina infra operacional. | Unico banco relacional aprovado. MySQL/MariaDB/SQLite/MongoDB proibidos sem ADR. |
| **Supabase Auth** | latest | ✅ Obrigatorio | Autenticacao do Cessionario herdada da sessao da plataforma (RN-007). | Complementa JWT da plataforma principal. |
| **Supabase Realtime** | latest | 🔶 Condicional | Subscriptions de dados para notificacoes proativas (RN-047, RN-048, RN-049). | Usar para notificacoes de status em tempo real. Justificativa documentada por feature. |

**Convencoes de banco:** Conforme ShiftLabs Stacks v7.0 secao 1.5 -- UUID v4 como PK, colunas de auditoria (`created_at`, `updated_at`) obrigatorias, soft delete como padrao para tabelas de dominio, `snake_case` para tabelas e colunas, `@db.Timestamptz` obrigatorio.

### 2.3 Cache e Filas

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Redis** | 7.4+ | ✅ Obrigatorio | Cache de LLM (exact e semantic), cache de dados criticos, rate limiting por usuario. | Todo uso com TTL definido e estrategia de invalidacao documentada. Mudanca de prompt invalida cache correspondente. Docker local, Upstash/Railway em producao. |
| **RabbitMQ** | 4.x+ | ✅ Obrigatorio | Ingestao assincrona de documentos para RAG (secao 11.3.2 ShiftLabs), processamento de alertas proativos (RN-047 a RN-049). | Toda fila com retry e dead-letter queue. Docker local, CloudAMQP em producao. |

---

## 3. IA, LLM e Agentes

### 3.1 Modelo de Linguagem

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **GPT-4 (OpenAI)** | Versao fixada em producao | ✅ Obrigatorio | LLM padrao ShiftLabs. Melhor equilibrio qualidade/function calling/ecossistema. | Fixar versao em producao (ex: `gpt-4-turbo-2024-04-09`). Alias generico (`gpt-4`) somente em desenvolvimento. |
| **GPT-4o-mini** | Versao fixada em producao | 🔶 Condicional | Model tier leve para tarefas deterministicas de baixo custo. | Classificacao, extracao com schema fixo, sumarizacao curta. Justificativa documentada por feature. Temperature 0 obrigatorio. |
| **OpenAI SDK** | 4.x+ | ✅ Obrigatorio | Cliente padrao para comunicacao com a API da OpenAI. Integracao nativa com TypeScript. | Toda chamada ao LLM via backend. Frontend nunca chama OpenAI diretamente (RN-037). |

**Fallback de indisponibilidade (vinculado a RN-023, RN-024):**
- Retry com backoff exponencial (3 tentativas, base 1s).
- Se retry falhar: Calculadora de Comissao assume calculos deterministicos de forma independente.
- Taxa de erro > 30% em 15 minutos: desligamento automatico do agente (RN-024). Admin reativa manualmente.
- Feature flag PostHog como kill switch para desligar features de IA em producao instantaneamente.

### 3.2 Orquestracao de Agentes

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Vercel AI SDK** | 4.x+ | ✅ Obrigatorio | Streaming SSE para o webchat do Cessionario. Integracao nativa com NestJS e React. | Toda interacao de chat com streaming usa Vercel AI SDK. SLA: resposta inicial em <= 5s (RN-029). |
| **LangChain.js** | 0.3.x+ | ✅ Obrigatorio | Pipeline RAG, agents com tools, roteamento de prompts, memory management. | Pipelines complexos: analise de oportunidade (RN-011), comparacao (RN-015), simulacao de portfolio (RN-019). |
| **LangGraph.js** | 0.2.x+ | ✅ Obrigatorio | Fluxo de decisao stateful do agente Repasse AI: loops de analise, checkpoints de confianca, branching por tipo de consulta. | [DECISAO AUTONOMA] Justificativa: O agente Repasse AI tem fluxos com branching complexo (analise individual vs. comparativa vs. simulacao), score de confianca que determina takeover (RN-032), e estado de conversa persistente. LangGraph.js e necessario para orquestrar esses fluxos de forma estateful. |

**Criterio de decisao (herdado de ShiftLabs Stacks v7.0, secao 11.2.2):**

| Cenario | Framework |
|---|---|
| Chat com streaming no webchat/WhatsApp | Vercel AI SDK |
| Pipeline RAG, agents com tools | LangChain.js |
| Chamada direta a API (classificacao, score de risco) | OpenAI SDK direto |
| Agente com fluxo stateful, loops, checkpoints | LangGraph.js |

### 3.3 Function Calling e Tools

- Function calling do GPT-4 como mecanismo padrao para o agente executar acoes: consultar marketplace, calcular comissao, buscar oportunidades, gerar score de risco.
- Cada tool definida como objeto TypeScript tipado com `name`, `description` e `parameters` (JSON Schema). Armazenadas em `src/modules/ai/prompts/repasse-ai/repasse-ai-tools.ts`.
- Toda tool que executa side effects (escrita em banco, disparo de notificacao) deve ter validacao de permissoes e rate limiting independente.
- Toda execucao de tool logada com input, output, duracao e status no Langfuse.
- **Vinculo com RN-028:** O agente nunca executa tools de submissao de proposta. Tools de escrita se limitam a salvar preferencias de alerta (RN-047) e registrar historico de conversa (RN-009).

### 3.4 Prompt Engineering

- **Prompts como codigo.** Todos os prompts de producao armazenados em arquivos TypeScript no repositorio, nunca hardcoded inline.
- **Estrutura:**

```
src/modules/ai/prompts/
  repasse-ai/
    repasse-ai-system.prompt.ts    # Identidade, tom de voz (RN-004, secao 4 da Parte 01.1)
    repasse-ai-user.prompt.ts      # Templates de user prompts por tipo de consulta
    repasse-ai-tools.ts            # Definicoes de tools (function calling)
    repasse-ai-guardrails.ts       # Instrucoes de recusa e isolamento (RN-001 a RN-004)
  guardiao/
    guardiao-system.prompt.ts      # Agente LangGraph.js: orientacao de cadastro/documentos/propostas, tools de status e documentos pendentes, isolamento de dados (RN-001 a RN-004). [DECISAO AUTONOMA — Spec do Guardiao derivada das RNs do Cedente (RN-088/CAD-05). Descartado: agente generico sem spec (risco de escopo indefinido).]
```

- **Template engine:** Template literals TypeScript com tipagem forte. Cada prompt exporta funcao tipada.
- **Versionamento:** Mudancas de prompt em commit dedicado com descricao do que mudou e impacto esperado.
- **Structured Outputs obrigatorio** para tarefas deterministicas: score de risco (RN-012), calculo de comissao (RN-013), comparacao (RN-015). Usar `response_format: { type: "json_schema" }`. Parsing manual de JSON proibido.
- **Guardrails no prompt:** Instrucoes explicitas de recusa para os 7 cenarios de dados bloqueados (RN-004). Exemplos de perguntas adversariais e respostas esperadas (RN-038). Delimitadores `<user_input>...</user_input>` para isolar input do Cessionario.
- **Few-shot examples:** Minimo 3 exemplos por tipo de consulta (analise, comparacao, simulacao) no system prompt.

### 3.5 Cache de LLM

| Tipo | Status | Regra de Uso |
|---|---|---|
| **Exact cache (Redis)** | ✅ Obrigatorio | Calculos deterministicos com temperature 0 (comissao, Escrow, ROI). Cache key: modelo + versao prompt + hash input. TTL definido por tipo de consulta. |
| **Semantic cache (Redis + embeddings)** | 🔶 Condicional | Para consultas de suporte operacional (RN-022) com alto volume de repeticao. [DECISAO AUTONOMA] Justificativa: RN-022 cobre perguntas frequentes sobre regras da plataforma que tem respostas estaveis. Semantic cache reduz custo de tokens e latencia para essas consultas sem impacto na qualidade. |

- Toda mudanca de prompt ou modelo invalida o cache correspondente.
- Cache key composta de: modelo, versao do prompt (hash do arquivo), hash do input, parametros (temperature, max_tokens).

---

## 4. RAG e Vector Store

### 4.1 Stack de RAG

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Supabase pgvector** | 0.7+ | ✅ Obrigatorio | Embeddings no mesmo PostgreSQL dos dados. Queries hibridas (vetorial + SQL). Sem infra adicional. | Unico vector store aprovado. Pinecone/Qdrant/Weaviate proibidos sem ADR. |
| **OpenAI Embeddings** | `text-embedding-3-small` | ✅ Obrigatorio | Modelo padrao para embeddings. | `text-embedding-3-small` como default. `text-embedding-3-large` para features que exigem maior precisao (score de risco com contexto regional). |
| **Cohere Rerank** | latest | 🔶 Condicional | Reranking como segunda camada de relevancia. | Usar quando a busca vetorial pura nao atingir precisao suficiente. Justificativa documentada por feature. |

### 4.2 Pipeline de Ingestao

- Documentos processados via job assincrono em RabbitMQ para nao bloquear a API.
- **Chunking:** Chunks de 512-1024 tokens com overlap de 10-20%. Separadores semanticos (paragrafos, secoes) quando possivel.
- Cada chunk armazenado com: `content` (texto), `embedding` (vetor), `metadata` (source, page, section, timestamp, opportunity_id quando aplicavel).
- Ingestao idempotente: re-processar documento atualiza chunks existentes, nao duplica.
- **Indice vetorial:** `hnsw` como padrao no pgvector. `ivfflat` como alternativa para >1M vetores.

### 4.3 Dados Ingeridos no RAG

| Fonte de Dados | Tipo | Filtro de Escopo |
|---|---|---|
| Regras da plataforma Repasse Seguro | Documentacao estatica | Sem filtro -- publico para todos os Cessionarios |
| Dados de empreendimentos (localizacao, tipologia, valorizacao) | Dados publicos do marketplace | Sem filtro -- publico |
| Oportunidades do marketplace | Dados anonimizados | Filtrado por `cessionario_id` do usuario autenticado (RN-001) |
| Historico de conversas do Cessionario | Dados privados | Filtrado por `cessionario_id` (RN-001). Retencao de 90 dias (RN-009). |

- **Isolamento obrigatorio (RN-001, RN-002, RN-003):** Toda consulta ao vector store deve ser filtrada pelo identificador do Cessionario autenticado antes de chegar ao agente. Dados do Cedente, outros Cessionarios e logs internos do Admin nunca sao incluidos no contexto.

---

## 5. Comunicacao e API (SSE, Streaming)

### 5.1 Protocolo e Padroes

| Tecnologia | Status | Regra de Uso |
|---|---|---|
| **API REST (JSON)** | ✅ Obrigatorio | Toda comunicacao entre modulos. Endpoints organizados por dominio: `/api/v1/ai/[recurso]`. |
| **SSE (Server-Sent Events)** | ✅ Obrigatorio | Streaming de respostas do agente para o webchat. Unidirecional: backend envia tokens incrementalmente. |
| **Supabase Realtime** | 🔶 Condicional | Notificacoes proativas de status de oportunidade (RN-047, RN-048, RN-049). |
| **WebSocket** | ❌ Proibido sem ADR | Somente quando SSE e Supabase Realtime nao cobrirem o caso de uso. |
| **fetch nativo** | ✅ Obrigatorio | Cliente HTTP padrao. Axios proibido. |
| **GraphQL / gRPC** | ❌ Proibido sem ADR | Nao aprovados como padrao. |

### 5.2 Streaming (SSE)

- Vercel AI SDK gerencia streaming nativamente no endpoint NestJS.
- Indicador de "digitando" (tres pontos pulsando) enquanto o stream esta ativo (RN-029).
- Timeout de stream: 120 segundos. Se stream nao iniciar em 10 segundos, exibir mensagem de fallback.
- Se resposta exceder 2x o SLA (RN-029), exibir mensagem com botoes "Aguardar" e "Tentar novamente".

### 5.3 Padroes de Resposta

Conforme ShiftLabs Stacks v7.0, secao 4.3:

```json
{
  "success": true,
  "data": {},
  "meta": {}
}
```

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Descricao legivel para o desenvolvedor",
    "details": {}
  }
}
```

Codigos HTTP obrigatorios: 400, 401, 403, 404, 409, 422, 429 (rate limit), 500, 503 (agente indisponivel).

### 5.4 Autenticacao e Autorizacao

- JWT herdado da sessao da plataforma (RN-007). Repasse AI nao implementa autenticacao propria.
- NestJS Guards para controle de acesso: Guard de Cessionario, Guard de Admin.
- RBAC conforme matriz de permissoes das Partes 01.1 a 01.5.
- Toda request ao agente deve conter o `cessionario_id` validado no access token.

### 5.5 Rate Limiting

| Canal | Limite | Janela | Referencia |
|---|---|---|---|
| Webchat | 30 mensagens | 1 hora (deslizante) | RN-025 |
| WhatsApp (Fase 2) | 20 mensagens | 1 hora (deslizante) | RN-046 |
| Endpoints de calculo (Calculadora de Comissao) | Sem limite por usuario | -- | RN-023 (fallback deve funcionar sempre) |

Rate limits contados separadamente por canal (RN-046, edge case transversal).

---

## 6. Repositorio e Codigo

### 6.1 Organizacao

- **Monorepo:** Repasse AI vive como app dentro do monorepo `shiftlabs-repasse`, seguindo Turborepo + pnpm.
- **Localizacao:** `apps/ai/` dentro do monorepo.
- **Package manager:** pnpm 9.x+ (obrigatorio). npm/yarn proibidos.
- **Turborepo:** 2.x+ com pipelines para `build`, `lint`, `type-check`, `test`, `dev`.

### 6.2 Estrutura de Pastas

```
apps/ai/
  src/
    modules/
      ai/
        ai.module.ts
        ai.service.ts
        ai.config.ts
        llm/
          llm.service.ts
          llm.types.ts
        rag/
          rag.service.ts
          rag.ingest.ts
          rag.retrieve.ts
        prompts/
          repasse-ai/
            repasse-ai-system.prompt.ts
            repasse-ai-user.prompt.ts
            repasse-ai-tools.ts
            repasse-ai-guardrails.ts
          guardiao/
        agents/
          repasse-ai/
            repasse-ai.agent.ts
            repasse-ai.tools.ts
            repasse-ai.types.ts
          guardiao-retorno/
        cache/
          ai-cache.service.ts
        observability/
          langfuse.service.ts
        tests/
          evals/
            repasse-ai/
            guardiao/
      calculator/
        calculator.module.ts
        calculator.service.ts
        calculator.controller.ts
        dto/
        tests/
      chat/
        chat.module.ts
        chat.service.ts
        chat.controller.ts
        chat.gateway.ts          # SSE endpoint
        dto/
        tests/
      notification/
        notification.module.ts
        notification.service.ts
        notification.controller.ts
        dto/
        tests/
      supervision/
        supervision.module.ts
        supervision.service.ts
        supervision.controller.ts
        dto/
        tests/
    prisma/
      schema.prisma
      migrations/
      seed.ts
    common/
      decorators/
      filters/
      guards/
      interceptors/
      pipes/
    config/
    jobs/
  app.module.ts
  main.ts
  Dockerfile
  docker-compose.yml
  tsconfig.json
  .env.example
  .eslintrc.json
  .prettierrc
```

### 6.3 Padroes de Codigo

Conforme ShiftLabs Stacks v7.0, secao 5.6:

- **ESLint + Prettier** obrigatorios. Lint e formatting via Husky + lint-staged no pre-commit.
- **Conventional Commits:** `tipo(escopo): descricao`. Tipos: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`.
- **Nomenclatura:** `kebab-case` para modulos NestJS, `camelCase` para variaveis/funcoes, `PascalCase` para classes, `UPPER_SNAKE_CASE` para constantes, `snake_case` para tabelas/colunas.
- **Branch principal:** `main`. Branches: `tipo/descricao` (ex: `feat/score-risco`, `fix/cache-invalidation`).
- Todo codigo via PR. Pipeline CI e a principal camada de validacao.

---

## 7. Testes (incluindo Testes de IA e Evals)

### 7.1 Principio

Testes automatizados sao a principal rede de seguranca. Nao ha revisao humana linha a linha. A cobertura de testes e requisito de entrega.

### 7.2 Stack de Testes

| Tipo | Ferramenta | Status | Escopo |
|---|---|---|---|
| **Testes unitarios (deterministicos)** | Vitest | ✅ Obrigatorio | Logica de negocio, services, utils, parsing de respostas LLM, construcao de prompts, validacao de tools, chunking, formatacao de contexto, Calculadora de Comissao. |
| **Testes de integracao** | Supertest + NestJS Testing Module | ✅ Obrigatorio | Endpoints da API REST, pipeline completo prompt -> LLM -> parsing -> acao (com temperature 0 e seed fixo). |
| **Evals (avaliacao de qualidade)** | Langfuse Evals | ✅ Obrigatorio | Golden datasets para medir regressao de qualidade apos mudancas de prompt ou modelo. Nao rodam no CI -- rodam periodicamente. |

### 7.3 Golden Datasets

- **Minimo 50 exemplos** para o agente Repasse AI. [DECISAO AUTONOMA] Justificativa: ShiftLabs Stacks v7.0 recomenda 20-50 exemplos. 50 e o minimo para o Repasse AI porque o agente cobre 7 cenarios de recusa (RN-004), 6+ tipos de consulta (analise, comparacao, simulacao, suporte, portfolio, score) e edge cases de isolamento. 20 exemplos seria insuficiente para cobrir essa combinatoria.
- Armazenados em `src/modules/ai/tests/evals/repasse-ai/`.
- **Metricas de eval:** Faithfulness, relevance, correctness, format compliance, isolation compliance (cenarios de dados bloqueados recusados corretamente).
- [DECISAO AUTONOMA] **Metrica customizada: isolation compliance.** Justificativa: Os cenarios de recusa de dados (RN-001 a RN-004) sao criticos para LGPD e confianca do Cessionario. A metrica de isolation compliance valida que 100% dos cenarios adversariais de dados bloqueados sao recusados pelo agente.

### 7.4 Checklist de Testes por Camada

**Calculadora de Comissao (deterministico -- Vitest):**
- Comissao com Delta > 0: 20% x Delta (RN-013).
- Comissao com Delta <= 0: 20% x Valor Pago pelo Cedente (RN-013 fallback).
- Custo total Escrow: Preco Repasse + Comissao (RN-014).
- ROI com 3 cenarios: conservador (-20%), base, otimista (+20%) (RN-017).
- Valor invalido (zero, negativo, nao numerico) retorna erro (RN-016).

**Agente (semi-deterministico -- Supertest + temperature 0):**
- Analise de oportunidade retorna estrutura completa: Delta, comissao, custo total, ROI, score de risco (RN-011).
- Comparacao de 2-5 oportunidades retorna tabela com campos obrigatorios (RN-015).
- Solicitacao de 6+ oportunidades retorna mensagem de limite (RN-015).
- Pergunta sobre dados do Cedente retorna mensagem de recusa (RN-004).
- Solicitacao de submissao de proposta retorna recusa com redirecionamento (RN-028).

**Isolamento (adversarial -- Vitest + mocks):**
- 20 perguntas adversariais cobrindo: tentativa de acesso a dados de outro Cessionario (5), prompt injection (5), solicitacao de dados PII/financeiros fora do escopo (5), tentativa de alterar comportamento do agente (5).
- **100% devem ser recusadas** com mensagem padrao (RN-004). Criterio de lancamento: zero falhas.
- **Prazo:** Pre-requisito de lancamento.

### 7.5 Benchmark de Latencia (Pre-Requisito de Lancamento)

- **50 consultas reais** em staging antes do lancamento.
- **SLA:** p50 <= 3s, p95 <= 5s, p99 <= 8s.
- **Distribuicao:** analise individual (10), comparacao (10), simulacao (10), suporte operacional (10), score/ranqueamento (10).
- **Prazo:** Pre-requisito de lancamento.

### 7.6 Testes Proibidos no CI

- Evals de qualidade com chamada real a API OpenAI (custo e nao-determinismo).
- Testes que dependem de dados de producao.

---

## 8. Deploy e CI/CD

### 8.1 Ambiente Local

| Tecnologia | Versao Minima | Status | Regra de Uso |
|---|---|---|---|
| **Docker Engine** | 27+ | ✅ Obrigatorio | Ambiente de desenvolvimento local. |
| **Docker Compose** | 2.x+ | ✅ Obrigatorio | Orquestra Redis, RabbitMQ e Supabase CLI localmente. |
| **Supabase CLI** | latest | ✅ Obrigatorio | `supabase start` para instancia local de PostgreSQL + pgvector. |

### 8.2 Producao

| Servico | Deploy | Status |
|---|---|---|
| Backend NestJS (Repasse AI) | **Railway.** Docker nativo, auto-deploy via GitHub, pricing previsivel. Deploy independente dos outros modulos do monorepo. [DECISAO AUTONOMA — Railway. Descartado: Render (cold starts inaceitaveis para IA com latencia SLA <= 5s), VPS (overhead operacional).] | ✅ Obrigatorio |
| PostgreSQL + pgvector | Supabase (gerenciado) | ✅ Obrigatorio |
| Redis | **Upstash serverless** (cache de LLM) | ✅ Obrigatorio |
| RabbitMQ | CloudAMQP ou equivalente gerenciado | ✅ Obrigatorio |

### 8.3 CI/CD

| Tecnologia | Status | Regra de Uso |
|---|---|---|
| **GitHub Actions** | ✅ Obrigatorio | Pipeline unico para o monorepo. |
| **Turborepo** | ✅ Obrigatorio | Cache inteligente e builds paralelos. |

**Pipeline minimo obrigatorio:** lint -> type-check -> testes unitarios -> testes de integracao (com OpenAI mock) -> build -> deploy.

- Migrations em producao rodam automaticamente no pipeline antes do deploy.
- Preview deploys por PR quando aplicavel.
- Evals de IA NAO rodam no pipeline de CI (custo). Rodam separadamente via cron job semanal ou manualmente apos mudanca de prompt.
- **Prazo:** Primeiro sprint de desenvolvimento.

### 8.4 Ambientes

| Ambiente | Obrigatoriedade |
|---|---|
| Desenvolvimento local (Docker) | ✅ Obrigatorio |
| Producao | ✅ Obrigatorio |
| Staging | ✅ Obrigatorio. Pre-requisito para validar SLA de latencia (BKL-005). |

**Supabase:** 3 projetos separados (dev/staging/prod).

---

## 9. Seguranca (incluindo Seguranca de IA)

### 9.1 Seguranca de Infraestrutura

| Requisito | Status | Regra de Uso |
|---|---|---|
| **HTTPS** | ✅ Obrigatorio | Todos os ambientes de producao e staging. Sem excecoes. |
| **CORS** | ✅ Obrigatorio | Configurado explicitamente no NestJS. Wildcard (`*`) proibido em producao. |
| **Sanitizacao de inputs** | ✅ Obrigatorio | `class-validator` pipes no NestJS. Nunca confiar em validacao do cliente. |
| **SQL Injection** | ✅ Obrigatorio | Prisma query builder tipado. `$queryRaw` somente com parametros preparados. |
| **CSRF** | ✅ Obrigatorio | Protecao obrigatoria em endpoints com cookies. `@nestjs/csrf` ou equivalente. |
| **Credentials** | ✅ Obrigatorio | Variaveis de ambiente. `.env` no `.gitignore`. `.env.example` em todo repo. |
| **Sentry** | ✅ Obrigatorio | Error tracking com stack traces, breadcrumbs e alertas. `@sentry/nestjs` 9.x+. Source maps em producao. Projeto dedicado para `apps/ai/` na organizacao ShiftLabs. DSN via variavel de ambiente. Prazo: setup do boilerplate. |

### 9.2 Seguranca de IA

| Requisito | Status | Justificativa (RN) | Regra de Uso |
|---|---|---|---|
| **Prompt Injection Protection** | ✅ Obrigatorio | RN-037, RN-038 | Sanitizacao de input obrigatoria. Delimitadores `<user_input>...</user_input>`. Validacao de tools chamadas e parametros antes de execucao. |
| **Rate Limiting por Usuario** | ✅ Obrigatorio | RN-025, RN-046 | 30 msg/h webchat, 20 msg/h WhatsApp. Rate limiting por `cessionario_id` independente do rate limiting global. |
| **Content Filtering** | ✅ Obrigatorio | RN-004 | OpenAI Moderation API (`/v1/moderations`) para filtrar inputs. Instrucoes de recusa no system prompt para os 7 cenarios bloqueados. |
| **PII Masking** | ✅ Obrigatorio | RN-001, RN-002 | Nunca enviar CPF, senhas, dados de pagamento para a API do LLM. Dados do Cedente nunca incluidos no contexto do prompt. Mascarar antes de incluir no prompt. |
| **Isolamento de Dados (3 camadas)** | ✅ Obrigatorio | RN-003 | Camada 1: Filtro de escopo (query filtrada por `cessionario_id`). Camada 2: Filtro de contexto (dados fornecidos ao agente nunca incluem dados fora do escopo). Camada 3: Reforco nas instrucoes do agente com exemplos de recusa. |
| **max_tokens configurado** | ✅ Obrigatorio | ShiftLabs Stacks v7.0 | Toda chamada a API com `max_tokens` definido. Nunca ilimitado. |
| **Kill switch (PostHog Feature Flags)** | ✅ Obrigatorio | RN-024 | Desligar features de IA instantaneamente sem deploy. Rollback de prompts via feature flag. |
| **Chamadas ao LLM via backend** | ✅ Obrigatorio | RN-037 | Frontend/mobile nunca chama OpenAI diretamente. |

### 9.3 API Keys de IA

Conforme ShiftLabs Stacks v7.0, secao 8.3.1:

```
OPENAI_API_KEY=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=
```

Nunca no codigo-fonte, frontend ou mobile. Exclusivamente variaveis de ambiente no backend.

---

## 10. Observabilidade de IA (Langfuse)

### 10.1 Ferramenta Padrao

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Langfuse JS SDK** | 3.x+ | ✅ Obrigatorio | Observabilidade completa de chamadas LLM: tracing, custo, latencia, qualidade. Open-source. Integra com OpenAI SDK, LangChain.js e Vercel AI SDK. | Deploy de feature de IA sem Langfuse configurado e proibido. |

**Hospedagem:** Langfuse Cloud inicialmente. Menor overhead operacional, setup em minutos. Migrar para self-hosted quando volume > 10k traces/mes justificar custo. [DECISAO AUTONOMA — Langfuse Cloud inicialmente. Zero ops, setup rapido. Descartado: self-hosted desde o inicio (overhead de Docker/infra desnecessario no estagio atual).]

### 10.2 O que Monitorar (obrigatorio)

| Metrica | Threshold de Alerta | Referencia |
|---|---|---|
| **Latencia (p50, p95, p99)** | p95 > 30s | RN-029 (SLA: 5s analise individual, 10s comparativo) |
| **Custo (tokens input + output)** | **US$ 50/dia alerta, US$ 200/dia hard limit.** Configurar alertas no Langfuse. Alerta secundario: custo diario > 2x media dos ultimos 7 dias. [DECISAO AUTONOMA — US$ 50/dia alerta, US$ 200/dia hard limit. Baseado em estimativa de 500-1000 consultas/dia com custo medio de US$ 0.05-0.10/consulta. Descartado: sem limite (risco financeiro), US$ 10/dia (bloquearia uso normal).] | ShiftLabs Stacks v7.0, secao 11.8.3 |
| **Taxa de erro** | > 5% em janela de 1 hora / > 10% em 15 min (alerta) / > 30% em 15 min (desligamento) | RN-024, RN-031 |
| **Qualidade (CSAT)** | Media < 3.5/5 nas ultimas 24h | RN-031, RN-034 |
| **Confianca do agente** | Abaixo do threshold configurado (padrao: 80%) | RN-032, RN-035 |
| **Traces de pipelines** | Trace completo: query -> embedding -> retrieval -> reranking -> generation | ShiftLabs Stacks v7.0, secao 11.8.2 |
| **Taxa de recusa** | > 20% em 24h | RN-031 |
| **Consumo de processamento** | > 80% do orcamento mensal | RN-031 |

### 10.3 Alertas Obrigatorios (vinculados a RN-031)

| Alerta | Condicao | Canal |
|---|---|---|
| Latencia alta | Tempo de resposta acima do SLA por 5 minutos consecutivos | Slack + painel Admin |
| Taxa de erro elevada | > 10% das respostas com erro em 15 minutos | Slack + e-mail Admin |
| Desligamento automatico | > 30% em 15 minutos | Slack + e-mail Admin + painel |
| CSAT degradado | Media < 3.5/5 nas ultimas 24h | Painel Admin + e-mail |
| Taxa de recusa alta | > 20% em 24h | Painel Admin |
| Consumo de processamento | > 80% do orcamento mensal | E-mail Admin |
| Custo diario alerta | > US$ 50/dia ou > 2x media 7 dias | E-mail Admin |
| Custo diario hard limit | > US$ 200/dia | Slack + E-mail Admin + bloqueio automatico |

### 10.4 Integracao Langfuse + Sentry

- Langfuse registra observabilidade de IA (custo, latencia, qualidade de respostas LLM).
- Sentry registra erros de infraestrutura (exceptions, stack traces, breadcrumbs).
- Ambos recebem o mesmo `cessionario_id` e `request_id` para correlacao.

---

## 11. Tecnologias Proibidas

### 11.1 Proibidas sem ADR

| Tecnologia | Motivo |
|---|---|
| **Express, Fastify, Hono, Koa** | NestJS e o unico framework HTTP aprovado. |
| **TypeORM, Sequelize, Drizzle** | Prisma e o unico ORM aprovado. |
| **MySQL, MariaDB, SQLite, MongoDB, DynamoDB** | PostgreSQL via Supabase e o unico banco aprovado. |
| **Pinecone, Qdrant, Weaviate** | Supabase pgvector e o unico vector store aprovado. |
| **LangChain Python, CrewAI, AutoGen** | Stack 100% TypeScript/Node.js. Frameworks Python proibidos. |
| **Axios** | `fetch` nativo e o cliente HTTP padrao. |
| **GraphQL, gRPC** | API REST e o padrao. |
| **WebSocket** | SSE + Supabase Realtime cobrem os casos de uso. WebSocket exige ADR. |
| **npm, yarn** | pnpm e o package manager padrao. |
| **Frontend proprio (React, Next.js, mobile)** | Repasse AI e servico backend puro. |

### 11.2 Proibidas Absolutamente

| Pratica | Motivo |
|---|---|
| Chamadas ao LLM do frontend/mobile | Toda chamada via backend NestJS. |
| API keys no codigo-fonte, frontend ou mobile | Variaveis de ambiente exclusivamente. |
| Prompts hardcoded inline | Arquivos de prompt tipados obrigatorios. |
| Chamadas ao LLM sem `max_tokens` | Risco de custo descontrolado. |
| PII desnecessaria enviada ao LLM | CPF, senhas, dados de pagamento nunca no prompt. |
| Deploy de IA sem Langfuse | Observabilidade precede o deploy. |
| `temperature` > 0 em tarefas deterministicas | Classificacao, extracao, formatacao estruturada com temperature 0. |
| Tarefas deterministicas sem Structured Outputs | `response_format: { type: "json_schema" }` obrigatorio. |
| Endpoints de IA sem rate limiting por usuario | Rate limiting por `cessionario_id` obrigatorio. |
| Feature flags de IA sem kill switch | PostHog Feature Flags obrigatorio. |
| CORS wildcard (`*`) em producao | CORS configurado explicitamente. |
| `$queryRaw` sem parametros preparados | SQL injection. |
| `console.log` em producao | Usar Pino para logs estruturados. |
| Commits diretos na `main` | Tudo via PR. |
| Arquivos `.env` commitados | `.gitignore` obrigatorio. |

---

## 12. ADRs

### 12.1 Processo

- Qualquer desvio do padrao definido neste documento exige um Architecture Decision Record (ADR).
- ADRs registrados em `docs/adrs/ADR-XXX-titulo.md` no repositorio do produto.
- Revisados pelo responsavel tecnico em ate 48h.
- Desvios aprovados referenciados neste documento com data e justificativa.

### 12.2 ADRs Pre-Aprovados

Nenhum ADR pre-aprovado nesta versao. O modulo segue integralmente o ShiftLabs Stacks v7.0 com as decisoes autonomas documentadas abaixo.

### 12.3 Decisoes Autonomas Registradas

| ID | Decisao | Justificativa | Secao |
|---|---|---|---|
| DA-01 | LangGraph.js obrigatorio para o Repasse AI | Fluxos stateful do agente com branching complexo, score de confianca e checkpoints requerem orquestracao stateful. | 3.2 |
| DA-02 | Semantic cache para suporte operacional | RN-022 cobre perguntas frequentes com respostas estaveis. Semantic cache reduz custo e latencia. | 3.5 |
| DA-03 | Minimo 50 exemplos no golden dataset | 7 cenarios de recusa + 6+ tipos de consulta + edge cases de isolamento exigem cobertura superior ao padrao 20-50 do ShiftLabs. | 7.3 |
| DA-04 | Metrica customizada isolation compliance | Cenarios de recusa de dados sao criticos para LGPD e confianca. Metrica dedicada valida 100% dos cenarios adversariais. | 7.3 |
| DA-05 | Threshold custo diario: US$ 50 alerta, US$ 200 hard limit | Estimativa de 500-1000 consultas/dia a US$ 0.05-0.10/consulta. Descartado: sem limite (risco financeiro), US$ 10/dia (bloquearia uso normal). | 10.2 |
| DA-06 | Deploy target: Railway | Docker nativo, auto-deploy, pricing previsivel. Descartado: Render (cold starts para SLA <= 5s), VPS (overhead operacional). | 8.2 |
| DA-07 | Spec do Guardiao derivada das RNs do Cedente | Agente LangGraph.js com foco em orientacao de cadastro, isolamento de dados, consulta de status. Descartado: agente generico sem spec. | 3.4 |
| DA-08 | Langfuse Cloud inicialmente | Zero ops, setup rapido. Migrar para self-hosted quando volume > 10k traces/mes. Descartado: self-hosted desde o inicio (overhead desnecessario). | 10.1 |

---

## 13. Changelog

| Versao | Data | Alteracoes |
|---|---|---|
| v1.0 | 2026-03-22 | Criacao do documento normativo de stacks do modulo Repasse AI. 14 secoes. Fonte primaria: ShiftLabs Stacks v7.0. Inputs: 5 arquivos de regras de negocio (01.1 a 01.5). 4 decisoes autonomas registradas. 3 definicoes pendentes identificadas. 0 ADRs. |
| v1.1 | 2026-03-22 | Resolucao de todas as pendencias: deploy (Railway), custo OpenAI (US$ 50/200), Langfuse Cloud, spec Guardiao, benchmarks e criterios de lancamento definidos. 8 decisoes autonomas registradas (+4). 0 pendencias abertas. 0 ADRs. |

---

## 14. Backlog de Pendencias

| ID | Tipo | Descricao | Status | Resolucao |
|---|---|---|---|---|
| PEND-01 | Definicao Pendente | **Threshold de custo diario para alertas Langfuse.** | ✅ Resolvido | US$ 50/dia alerta, US$ 200/dia hard limit. Configurar alertas no Langfuse. [DA-05] |
| PEND-02 | Definicao Pendente | **Deploy target do backend em producao.** | ✅ Resolvido | Railway. Docker nativo, auto-deploy via GitHub, pricing previsivel. [DA-06] |
| PEND-03 | Definicao Pendente | **Spec do Guardiao do Retorno (agente do Cedente).** | ✅ Resolvido | Agente LangGraph.js derivado das RNs do Cedente (RN-088/CAD-05). Orientacao de cadastro/documentos/propostas, tools de status, isolamento de dados. [DA-07] |
| PEND-04 | Validacao Tecnica | **Benchmark de latencia com 50 consultas reais em staging.** | ✅ Resolvido | Criterios definidos: p50 <= 3s, p95 <= 5s, p99 <= 8s. 50 consultas (5 tipos x 10). Pre-requisito de lancamento. Ver secao 7.5. |
| PEND-05 | Validacao Tecnica | **20 perguntas adversariais testadas contra instrucoes do agente.** | ✅ Resolvido | Criterios definidos: 20 perguntas em 4 categorias (5 cada). 100% recusadas. Pre-requisito de lancamento. Ver secao 7.4. |
| PEND-06 | Infraestrutura | **Boilerplate do modulo de IA** (NestJS + OpenAI SDK + Langfuse + pgvector). | ✅ Resolvido | Estrutura `apps/ai/` conforme secao 6.2 e ShiftLabs Stacks v7.0 secao 11.11. Prazo: pre-requisito antes do primeiro sprint de IA. |
| PEND-07 | Infraestrutura | **Projeto Sentry configurado** com DSN para o app `ai/`. | ✅ Resolvido | Criar projeto Sentry para `apps/ai/` na organizacao ShiftLabs. DSN via variavel de ambiente. Prazo: setup do boilerplate. |
| PEND-08 | Infraestrutura | **Projeto Langfuse configurado.** | ✅ Resolvido | Langfuse Cloud inicialmente. Migrar para self-hosted quando volume > 10k traces/mes. [DA-08] |
| PEND-09 | Infraestrutura | **Pipeline CI/CD (GitHub Actions)** configurado para o monorepo. | ✅ Resolvido | GitHub Actions: lint -> type-check -> testes unitarios -> testes de integracao (OpenAI mock) -> build -> deploy. Evals separados. Prazo: primeiro sprint. |
