# 02 - Stacks Tecnológicas — AI-Dani-Cedente

## Stack Normativa do Agente AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Tech Lead / CTO |
| Escopo | Stack normativa do agente AI-Dani-Cedente — Repasse Seguro (perspectiva do Cedente) |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Caráter | **Normativo.** Este documento define o padrão obrigatório para o agente AI-Dani-Cedente. Desvios exigem ADR aprovado pelo responsável técnico. |
| Fonte primária | ShiftLabs Stacks v7.1 |
| Revisão | Trimestral ou a cada mudança significativa de stack |

---

> **📌 TL;DR**
>
> - **AI-Dani-Cedente é um serviço backend puro** — sem frontend web ou mobile próprio. Expõe API REST e SSE. Consumido pelo frontend da plataforma Repasse Seguro via API.
> - **LLM padrão:** GPT-4 via OpenAI SDK. Orquestração com Vercel AI SDK (streaming SSE) + LangChain.js (RAG, tools). LangGraph.js para fluxos stateful do agente Dani.
> - **RAG:** Supabase pgvector como vector store. OpenAI Embeddings. Ingestão assíncrona via RabbitMQ.
> - **Backend:** Node.js 22+ / NestJS 10+ / TypeScript strict / Prisma 6+ / Supabase PostgreSQL 17+. Redis para cache de LLM e rate limiting. RabbitMQ para filas.
> - **Observabilidade de IA:** Langfuse obrigatório para tracing, custo, latência e qualidade. Sentry para error tracking de infraestrutura.
> - **Segurança de IA:** Isolamento total de dados do Cedente (RN-DCE-001), proteção contra prompt injection, rate limiting 30 msg/h por Cedente (RN-DCE-022), PII masking obrigatório.
> - **Testes de IA:** Vitest para lógica determinística + evals com golden datasets via Langfuse Evals. Mínimo 50 exemplos no golden dataset.

---

## 1. Princípios de Governança Tecnológica

**PG-01. ShiftLabs Stacks v7.1 é a fonte primária de verdade.** Toda tecnologia deste documento herda do padrão central. Desvios são permitidos exclusivamente com justificativa técnica vinculada a uma regra de negócio (RN) do AI-Dani-Cedente, documentados via ADR.

**PG-02. Desenvolvimento 100% por agentes de IA.** Todo código do AI-Dani-Cedente é gerado por Claude Code Desktop e GPT Codex. Boilerplates, convenções de código, estrutura de pastas e testes automatizados são os contratos de qualidade.

**PG-03. AI-Dani-Cedente é um serviço backend sem frontend próprio.** O agente expõe API REST e SSE. Não possui frontend web, mobile, landing page ou dashboard. Seções de frontend e mobile do ShiftLabs Stacks v7.1 não se aplicam a este módulo, exceto como referência para consumidores do serviço.

**PG-04. Segurança de IA e isolamento de dados são requisitos de lançamento, não melhorias futuras.** Isolamento de dados do Cedente (RN-DCE-001 a RN-DCE-002), proteção contra prompt injection e rate limiting por usuário (RN-DCE-022) devem estar implementados e testados antes da ativação em produção.

**PG-05. Observabilidade de IA precede o deploy.** Nenhuma feature de IA entra em produção sem tracing Langfuse configurado. Custo, latência e qualidade devem ser mensuráveis desde o primeiro request.

---

## 2. Backend e Runtime

### 2.1 Stack Obrigatória

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Node.js** | 22.x+ (LTS) | ✅ Aprovado | Runtime padrão ShiftLabs. LTS garante estabilidade. | Usar apenas versões LTS em produção. |
| **NestJS** | 10.x+ | ✅ Aprovado | Arquitetura modular, decorators TypeScript, Swagger embutido, guards/pipes nativos. Output de IA mais consistente. | Framework HTTP único. Express/Fastify/Hono proibidos sem ADR. |
| **TypeScript** | 5.4+ | ✅ Aprovado | `strict: true` obrigatório. Tipagem forte maximiza qualidade do output gerado por IA. | Arquivos `.js`/`.jsx` proibidos em novos módulos. |
| **Prisma** | 6.x+ | ✅ Aprovado | Tipagem auto-gerada, migrations declarativas, integração nativa com Supabase. Coexiste com `$queryRaw` para pgvector. | Único ORM aprovado. TypeORM/Sequelize/Drizzle proibidos. |
| **Pino** | 9.x+ | ✅ Aprovado | Logs estruturados com request ID, timestamp, nível e contexto. | Todo serviço com logs estruturados. `console.log` proibido em produção. |
| **Helmet** | 8.x+ | ✅ Aprovado | Headers de segurança automáticos. | Middleware adicionado desde o início do projeto. |
| **class-validator + class-transformer** | latest | ✅ Aprovado | Validação de DTOs integrada aos pipes do NestJS. | Toda entrada de dados validada via pipes. |
| **@nestjs/swagger** | latest | ✅ Aprovado | Swagger/OpenAPI auto-gerado. | Todo endpoint documentado. Acessível na URL do backend. |
| **@nestjs/throttler** | latest | ✅ Aprovado | Rate limiting nativo do NestJS. | Todo endpoint público com rate limiting. Endpoints de IA com rate limiting por Cedente — 30 msg/h janela deslizante (RN-DCE-022). |

### 2.2 Banco de Dados

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **PostgreSQL (via Supabase)** | 17+ | ✅ Aprovado | PostgreSQL gerenciado + Auth + Storage + Realtime + pgvector em uma plataforma. Elimina infra operacional. | Único banco relacional aprovado. MySQL/MariaDB/SQLite/MongoDB proibidos sem ADR. |
| **Supabase Auth** | latest | ✅ Aprovado | Autenticação do Cedente herdada da sessão da plataforma Repasse Seguro (RN-DCE-001). | Complementa JWT da plataforma principal. |
| **Supabase pgvector** | latest | ✅ Aprovado | Embeddings no mesmo PostgreSQL dos dados. Queries híbridas (vetorial + SQL). Sem infra adicional. Base de conhecimento da Dani (RAG sobre contratos, regras e FAQ). | Queries via `$queryRaw` com parâmetros preparados. |
| **Supabase Realtime** | latest | 🔶 Condicional | Subscriptions de dados para notificações proativas ao Cedente (RN-DCE-020). | Usar para notificações de status em tempo real. Justificativa documentada por feature. |

**Convenções de banco:** conforme ShiftLabs Stacks v7.1 seção 1.5 — UUID v4 como PK, colunas de auditoria (`created_at`, `updated_at`) obrigatórias, soft delete como padrão para tabelas de domínio, `snake_case` para tabelas e colunas, `@db.Timestamptz` obrigatório.

### 2.3 Cache e Filas

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Redis** | 7.4+ | ✅ Aprovado | Cache de LLM (exact e semantic), cache de dados críticos, rate limiting por Cedente (janela deslizante 30 msg/h). | Todo uso com TTL definido e estratégia de invalidação documentada. Mudança de prompt invalida cache correspondente. Docker local, Upstash/Railway em produção. |
| **RabbitMQ** | 4.x+ | ✅ Aprovado | Ingestão assíncrona de documentos do dossiê para RAG. Processamento de alertas proativos (RN-DCE-020). | Toda fila com retry e dead-letter queue. Docker local, CloudAMQP em produção. |

---

## 3. IA, LLM e Agentes

### 3.1 Modelo de Linguagem

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **GPT-4 (OpenAI)** | Versão fixada em produção | ✅ Aprovado | LLM padrão ShiftLabs. Melhor equilíbrio qualidade/function calling/ecossistema. Apto para análise de propostas, cálculo de retorno líquido e orientação ao Cedente. | Fixar versão em produção (ex: `gpt-4-turbo-2024-04-09`). Alias genérico (`gpt-4`) somente em desenvolvimento. |
| **GPT-4o-mini** | Versão fixada em produção | 🔶 Condicional | Model tier leve para tarefas determinísticas de baixo custo. | Classificação de intenção, extração de dados com schema fixo, sumarização de documentos do dossiê. Temperature 0 obrigatório. Justificativa documentada por feature. |
| **OpenAI SDK** | 4.x+ | ✅ Aprovado | Cliente padrão para comunicação com a API da OpenAI. Integração nativa com TypeScript. | Toda chamada ao LLM via backend. Frontend nunca chama OpenAI diretamente. |

**Fallback de indisponibilidade (vinculado a RN-DCE-023):**
- Retry com backoff exponencial: 3 tentativas, base 1s.
- Se retry falhar: a Dani exibe mensagem de indisponibilidade ao Cedente (RN-DCE-023).
- Taxa de erro > 10% em 15 minutos: alerta automático ao Admin.
- Taxa de erro > 30% em 15 minutos: desligamento automático do agente (RN-DCE-023). Admin reativa manualmente.
- Feature flag PostHog como kill switch para desligar features de IA em produção instantaneamente.

### 3.2 Orquestração de Agentes

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Vercel AI SDK** | 4.x+ | ✅ Aprovado | Streaming SSE para o webchat do Cedente. Integração nativa com NestJS. SLA: resposta inicial ≤ 5s (RN-DCE-022). | Toda interação de chat com streaming usa Vercel AI SDK. |
| **LangChain.js** | 0.2.x+ | ✅ Aprovado | RAG sobre base de conhecimento da plataforma (contratos, regras, FAQ do Cedente). Tool calling estruturado para ações determinísticas. | RAG com pgvector via LangChain. Tool calling para consultas de status, cálculo de retorno líquido (RN-DCE-015) e análise de propostas (RN-DCE-014). |
| **LangGraph.js** | 0.0.x+ (latest stable) | ✅ Aprovado | Fluxos stateful do agente Dani-Cedente: ciclo de conversa com memória, grafo de estados, human-in-the-loop para takeover do Admin (RN-DCE-023). | Agente principal implementado como LangGraph. Estados: `idle`, `analyzing_proposal`, `escrow_monitoring`, `dossier_guidance`, `takeover`. |

### 3.3 RAG (Retrieval-Augmented Generation)

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Supabase pgvector** | latest | ✅ Aprovado | Embeddings no mesmo PostgreSQL. Sem infra adicional. | Base de conhecimento: regras da plataforma, FAQ do Cedente, modelos de cenários A/B/C/D. |
| **OpenAI Embeddings** (`text-embedding-3-small`) | API atual | ✅ Aprovado | Modelo de embedding padrão OpenAI. Custo baixo, qualidade adequada para documentos de plataforma. | Embeddings gerados no momento da ingestão. Re-embedding obrigatório ao alterar base de conhecimento. |

### 3.4 Observabilidade de IA

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Langfuse** | 3.x+ | ✅ Aprovado | Tracing de LLM: custo por conversa, latência por etapa, qualidade de respostas, evals automáticos. Obrigatório antes do deploy. | Toda chamada ao LLM com trace Langfuse. Dataset de evals mínimo: 50 exemplos por fluxo (análise de proposta, orientação de dossiê, simulação de retorno). |

### 3.5 Segurança de IA

| Controle | Status | Implementação | Vinculado a |
|---|---|---|---|
| **Isolamento de dados por Cedente** | ✅ Obrigatório | Middleware de autorização valida que o `cedente_id` do JWT corresponde ao escopo da query antes de qualquer chamada ao LLM. | RN-DCE-001 |
| **PII Masking** | ✅ Obrigatório | Dados pessoais do Cessionário são mascarados antes de entrar no contexto do LLM. O agente nunca recebe nome, CPF ou contato do Cessionário. | RN-DCE-001, RN-DCE-002 |
| **Prompt Injection Protection** | ✅ Obrigatório | Sanitização de inputs do usuário antes de inserção no prompt. System prompt separado e imutável. Instruções do usuário em tag delimitada. | RN-DCE-001 |
| **Rate Limiting por Cedente** | ✅ Obrigatório | 30 mensagens por hora por Cedente (janela deslizante via Redis). Campo de entrada desabilitado com contador regressivo ao atingir o limite. | RN-DCE-022 |
| **Content Filtering** | ✅ Obrigatório | OpenAI Moderation API para inputs do Cedente antes de processar. Respostas da Dani validadas contra regras de escopo (RN-DCE-002). | RN-DCE-002 |
| **Structured Outputs** | ✅ Obrigatório | Toda resposta determinística (cálculo de retorno líquido, status de dossiê, status de Escrow) usa Structured Outputs com schema JSON fixo. | RN-DCE-015, RN-DCE-018 |

---

## 4. Comunicação e API

### 4.1 Padrão de API

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **REST (JSON)** | — | ✅ Aprovado | Padrão ShiftLabs. Interoperabilidade com o frontend da plataforma Repasse Seguro. | Todo endpoint REST. `Content-Type: application/json`. |
| **SSE (Server-Sent Events)** | — | ✅ Aprovado | Streaming de respostas do LLM ao webchat do Cedente. Menor overhead que WebSocket para comunicação unidirecional. | Exclusivo para endpoints de chat com streaming (`/chat/stream`). |
| **WebSocket** | — | ❌ Proibido sem ADR | SSE atende o caso de uso de streaming unidirecional (LLM → cliente). WebSocket aumenta complexidade sem benefício mensurável para o agente Dani. | Requer ADR aprovado para uso. |

### 4.2 Autenticação e Autorização

| Tecnologia | Status | Regra de Uso |
|---|---|---|
| **JWT (Bearer Token)** | ✅ Obrigatório | Token JWT emitido pela plataforma Repasse Seguro. Access token curto (15 min). Refresh token longo (7 dias). Todo endpoint protegido via `JwtAuthGuard`. |
| **Supabase Auth** | ✅ Complementar | Sessão do Cedente validada via Supabase Auth em endpoints que requerem dados do banco (oportunidades, dossiê, propostas). |
| **RBAC via Guards NestJS** | ✅ Obrigatório | Guards verificam que o `cedente_id` do JWT corresponde ao recurso solicitado. Nenhuma query retorna dados de outros Cedentes. Vinculado a RN-DCE-001. |

### 4.3 Versionamento de API

- Prefixo de versão obrigatório: `/api/v1/`.
- Breaking changes requerem nova versão (`/api/v2/`). Versão anterior mantida por no mínimo 90 dias após deprecação.

---

## 5. Testes

### 5.1 Estratégia de Testes

| Camada | Ferramenta | Cobertura Mínima | Notas |
|---|---|---|---|
| **Unit (lógica de negócio)** | Vitest | 80% de linhas | Services, calculadoras (retorno líquido, Δ), guards de autorização. |
| **Integration (API)** | Vitest + Supertest | Todos os endpoints | Fluxos críticos: análise de proposta, simulação de retorno, status de Escrow. Banco de dados via Supabase local/Docker. |
| **Evals de IA (LLM)** | Langfuse Evals | Mínimo 50 exemplos por fluxo | Golden datasets por fluxo: análise de proposta (RN-DCE-014), orientação de dossiê (RN-DCE-013), simulação de retorno (RN-DCE-015). |
| **Segurança** | Testes manuais + automatizados | 100% dos controles | Isolamento de dados (RN-DCE-001): verificar que Cedente A nunca acessa dados do Cedente B. Prompt injection: testar variações de payload. |

### 5.2 Regras de Testes

- Testes de isolamento de dados são obrigatórios antes de qualquer deploy em produção (PG-04).
- Evals de IA devem ser executados a cada alteração de prompt de sistema.
- Golden datasets são mantidos em `tests/evals/` e versionados junto ao código.
- Mocks do LLM (OpenAI) obrigatórios em testes unitários e de integração — nunca chamar API real em testes automatizados.

---

## 6. Repositório e Código

### 6.1 Estrutura de Repositório

- **Monorepo com Turborepo + pnpm** conforme ShiftLabs Stacks v7.1 seção 5.1.
- O módulo `ai-dani-cedente` reside em `apps/ai-dani-cedente/` no monorepo do Repasse Seguro.
- Types compartilhados com outros módulos em `packages/types/`.

### 6.2 Convenções de Código

- **Nomenclatura de arquivos:** `kebab-case` para módulos NestJS. Exemplos: `cedente-chat.controller.ts`, `proposal-analysis.service.ts`.
- **Nomenclatura de variáveis:** `camelCase` para variáveis e funções TypeScript.
- **Nomenclatura de classes:** `PascalCase` para classes, interfaces e tipos.
- **Linting:** ESLint com `@typescript-eslint/strict`. Nenhum `any` explícito.
- **Formatting:** Prettier com configuração padrão ShiftLabs (print width: 100, single quote, semi: true).
- **Commits:** Conventional Commits (`feat:`, `fix:`, `refactor:`, `test:`, `docs:`).

### 6.3 Code Review

- Todo PR requer aprovação de pelo menos 1 reviewer.
- PR sem testes não é mergeado.
- PR que modifica prompt de sistema requer execução de evals documentada no PR.

---

## 7. Deploy e CI/CD

### 7.1 Ambientes

| Ambiente | Propósito | Branch | Regra de Promoção |
|---|---|---|---|
| **development** | Desenvolvimento local | feature/* | — |
| **staging** | Validação pré-produção | main | Todos os testes passando + evals de IA aprovados |
| **production** | Produção | tags `v*.*.*` | Deploy manual por Tech Lead após validação em staging |

### 7.2 Pipeline CI/CD

- **Ferramenta:** GitHub Actions.
- **Pipeline mínimo obrigatório:** lint → type-check → testes unitários → testes de integração → evals de IA (staging/prod only) → build → deploy.
- **Gates bloqueantes:** falha em lint, type-check ou testes unitários bloqueia merge.
- **Rollback:** automatizado via script em caso de health check failure pós-deploy.

### 7.3 Deploy

- **Backend:** Docker container em VPS ou serviço gerenciado (Railway, Fly.io). [DECISÃO AUTÔNOMA — alinhado com padrão ShiftLabs para serviços backend puro sem necessidade de edge network. Alternativa descartada: Vercel (sem suporte adequado a serviços backend stateful com WebSocket/SSE de longa duração).]
- **Variáveis de ambiente:** nunca em código-fonte. Gerenciadas via secrets do CI/CD e `.env` local (não commitado).

---

## 8. Segurança

### 8.1 Regras Críticas

> **⚙️ Regras obrigatórias de segurança:**
>
> - HTTPS obrigatório em todos os ambientes.
> - CORS configurado: apenas origens da plataforma Repasse Seguro autorizadas.
> - Rate limiting em todos os endpoints públicos. Endpoints de IA com rate limiting por Cedente (30 msg/h).
> - Sanitização de todos os inputs do usuário antes de inserção no prompt do LLM.
> - Credentials, tokens e API keys nunca no código-fonte — apenas em variáveis de ambiente.
> - Logs nunca contêm PII (nome, CPF, e-mail, telefone) do Cedente ou Cessionário.
> - Supabase Row Level Security (RLS) ativo em todas as tabelas de dados do Cedente.

### 8.2 Proibições

> **🔴 Proibido:**
>
> - Chamar a API da OpenAI diretamente do frontend — sempre via backend.
> - Armazenar tokens de acesso em `localStorage` — usar `httpOnly cookies` ou `SecureStore` (mobile).
> - Logar conteúdo de mensagens de chat com dados pessoais sem mascaramento.
> - Enviar dados do Cedente para o LLM sem validação de escopo (RN-DCE-001).
> - Secrets no código-fonte ou em arquivos commitados.

### 8.3 Compliance LGPD

- Histórico de conversas mantido por 90 dias (RN-DCE-022, parâmetro configurável pelo Admin).
- Opt-out de notificações WhatsApp imediato, sem confirmação, via resposta "PARAR" ou configurações de perfil (RN-DCE-020).
- PII do Cessionário nunca exposto ao Cedente nem inserido no contexto do LLM (RN-DCE-001).
- CSAT anonimizado antes de armazenar em analytics (RN-DCE-024).

---

## 9. Analytics e Tracking

| Tecnologia | Status | Regra de Uso |
|---|---|---|
| **PostHog** | ✅ Aprovado | Analytics de uso do agente: mensagens por sessão, ações tomadas pelo Cedente (aceitou proposta, enviou contraproposta, consultou dossiê), CSAT (RN-DCE-024). Eventos em `snake_case` via wrapper centralizado. |
| **Langfuse** | ✅ Aprovado | Observabilidade específica de IA: custo por conversa, latência, qualidade, evals. Separado do PostHog (propósitos distintos). |
| **Sentry** | ✅ Aprovado | Error tracking de infraestrutura. Stack traces, breadcrumbs, alertas. SDK: `@sentry/nestjs` 9.x+. |

**Eventos PostHog obrigatórios mínimos:**

| Evento | Quando disparar |
|---|---|
| `dani_cedente_chat_opened` | Cedente abre o chat da Dani |
| `dani_cedente_message_sent` | Cedente envia mensagem |
| `dani_cedente_proposal_analysis_viewed` | Dani apresenta análise de proposta |
| `dani_cedente_simulation_requested` | Cedente solicita simulação de retorno líquido |
| `dani_cedente_dossier_status_viewed` | Cedente visualiza status do dossiê |
| `dani_cedente_escrow_status_viewed` | Cedente visualiza status do Escrow |
| `dani_cedente_csat_submitted` | Cedente submete avaliação CSAT |
| `dani_cedente_rate_limit_reached` | Cedente atinge o limite de 30 msg/h |
| `dani_cedente_takeover_triggered` | Admin inicia takeover da conversa |

---

## 10. Tecnologias Proibidas

| Tecnologia | Motivo da Proibição | Alternativa Aprovada |
|---|---|---|
| **Express / Fastify / Hono** | Frameworks alternativos sem aprovação via ADR. NestJS é o padrão ShiftLabs. | NestJS 10.x+ |
| **TypeORM / Sequelize / Drizzle** | ORMs alternativos. Prisma é o único ORM aprovado. | Prisma 6.x+ |
| **MySQL / MariaDB / SQLite / MongoDB** | Bancos não aprovados. PostgreSQL via Supabase é o padrão. | PostgreSQL 17+ via Supabase |
| **Moment.js** | Deprecated, bundle pesado (~330kB). | date-fns 4.x+ |
| **Redux / MobX** | Gerenciadores de estado globais não aprovados (aplicável a consumidores frontend). | Zustand 5.x+ |
| **Pinecone / Weaviate / Qdrant** | Vector stores externos sem justificativa. pgvector via Supabase elimina infra adicional. | Supabase pgvector |
| **console.log em produção** | Logs não estruturados, sem context, sem request ID. | Pino 9.x+ |
| **Secrets no código-fonte** | Risco crítico de segurança. Violação de compliance. | Variáveis de ambiente + secrets do CI/CD |
| **Chamada direta à OpenAI API no frontend** | Expõe API key no cliente. Bypassa rate limiting e controles de segurança do backend. | Chamadas via backend NestJS exclusivamente |
| **Auto-increment como PK** | Expõe volume de dados, inseguro em URLs públicas. | UUID v4 (`@default(uuid())`) |

---

## 11. ADRs — Desvios do Baseline

**ADR-001: Deploy em VPS/Railway em vez de Vercel para o serviço backend**

1. **Contexto:** AI-Dani-Cedente é um serviço backend NestJS com SSE de longa duração para streaming de LLM e conexões Redis/RabbitMQ persistentes.
2. **Decisão:** Deploy em container Docker em VPS ou serviço gerenciado (Railway, Fly.io).
3. **Alternativas consideradas:** Vercel — não suporta adequadamente serviços backend stateful com SSE de longa duração e conexões de mensageria.
4. **Consequências:** maior controle operacional, necessidade de gerenciar health checks e rollback manual. Justificado pelos requisitos técnicos do serviço.

**ADR-002: LangGraph.js para fluxo stateful do agente Dani-Cedente**

1. **Contexto:** A Dani-Cedente requer fluxo stateful com memória de conversa, múltiplos estados (análise de proposta, monitoramento de Escrow, orientação de dossiê) e suporte a takeover pelo Admin (RN-DCE-023).
2. **Decisão:** LangGraph.js como orquestrador do agente, complementando LangChain.js.
3. **Alternativas consideradas:** LangChain puro — sem suporte nativo a grafos de estado e human-in-the-loop. Chain simples — insuficiente para a complexidade dos fluxos.
4. **Consequências:** maior complexidade de código, compensada pela clareza do grafo de estados e pela capacidade de takeover.

---

## 12. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — stack normativa do agente AI-Dani-Cedente derivada de ShiftLabs Stacks v7.1 e RN-DCE (D01). |

---

## 13. Backlog de Pendências

| Item Pendente | Seção | Impacto | Pergunta Objetiva | Dono | Status |
|---|---|---|---|---|---|
| [DECISÃO AUTÔNOMA] Deploy em VPS/Railway | 7.3 | P1 | Confirmar serviço gerenciado preferido (Railway vs Fly.io vs VPS) com base em contrato de infra já existente. | Tech Lead | Aberto |
| [DECISÃO AUTÔNOMA] LangGraph.js versão | 3.2 | P2 | Confirmar versão estável mais recente do LangGraph.js no momento do início do desenvolvimento. | Dev Lead | Aberto |
