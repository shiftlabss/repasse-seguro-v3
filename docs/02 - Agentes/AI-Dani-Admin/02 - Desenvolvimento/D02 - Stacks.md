# Stacks Tecnológicas — AI-Dani-Admin

## Stack Normativa do Módulo de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Tech Lead / CTO |
| Escopo | Stack normativa do módulo AI-Dani-Admin — Supervisão Operacional da Repasse Seguro |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Caráter | **Normativo.** Este documento define o padrão obrigatório para o módulo AI-Dani-Admin. Desvios exigem ADR aprovado pelo responsável técnico. |
| Fonte primária | ShiftLabs Stacks v7.1 |
| Revisão | Trimestral ou a cada mudança significativa de stack |

---

> **📌 TL;DR**
>
> - **AI-Dani-Admin é um módulo de supervisão backend** — sem frontend web ou mobile próprio. Expõe API REST interna para o painel Admin da plataforma Repasse Seguro.
> - **Escopo:** monitoramento de interações, takeover manual, métricas de agentes, alertas automáticos, configuração de threshold, critérios de prontidão para lançamento.
> - **Backend:** Node.js 22+ / NestJS 10+ / TypeScript strict / Prisma 6+ / Supabase PostgreSQL 17+. Redis para cache de métricas. RabbitMQ para filas de alertas.
> - **Observabilidade de IA:** Langfuse obrigatório para tracing de interações supervisionadas. Sentry para error tracking de infraestrutura.
> - **Segurança:** Acesso restrito ao perfil Admin. Isolamento de dados obrigatório antes de qualquer ativação em produção (RN-DA-037). Rate limit: 30 msg/h por usuário no webchat (RN-DA-036).
> - **Alertas:** Slack + e-mail via RabbitMQ para condições críticas (desligamento automático, latência alta, taxa de erro elevada) — RN-DA-031.
> - **0 desvios do baseline ShiftLabs v7.1 nesta versão.** Todos os ADRs registrados na seção 12.

---

## 1. Princípios de Governança

**PG-01. ShiftLabs Stacks v7.1 é a fonte primária de verdade.** Toda tecnologia deste documento herda do padrão central. Desvios são permitidos exclusivamente com justificativa técnica vinculada a uma regra de negócio (RN-DA-XXX), documentados via ADR.

**PG-02. Desenvolvimento 100% por agentes de IA.** Todo código do AI-Dani-Admin é gerado por Claude Code e GPT Codex. Boilerplates, convenções de código, estrutura de pastas e testes automatizados são os contratos de qualidade.

**PG-03. AI-Dani-Admin é um módulo backend sem frontend próprio.** O módulo expõe API REST interna consumida pelo painel Admin da plataforma. Não possui frontend web, mobile ou landing page próprios. As seções de frontend e mobile do ShiftLabs Stacks v7.1 não se aplicam a este módulo.

**PG-04. Segurança de acesso é requisito de lançamento, não melhoria futura.** Isolamento de dados (RN-DA-037), filtro de escopo e filtro de contexto devem estar implementados e testados antes da ativação em produção. Lançamento bloqueado se qualquer item de segurança falhar na verificação.

**PG-05. Supervisão Admin deve ser operacional antes do deploy.** Registro de interações, dashboard de métricas, alertas automáticos e takeover manual devem estar implementados e testados antes do lançamento de qualquer agente (RN-DA-039).

---

## 2. Backend e Runtime

### 2.1 Stack Obrigatória

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Node.js** | 22.x+ (LTS) | ✅ Aprovado | Runtime padrão ShiftLabs. LTS garante estabilidade em produção. | Usar apenas versões LTS em produção. |
| **NestJS** | 10.x+ | ✅ Aprovado | Arquitetura modular, decorators TypeScript, Swagger embutido, guards/pipes nativos. Output de IA mais consistente. | Framework HTTP único. Express/Fastify/Hono proibidos sem ADR. |
| **TypeScript** | 5.4+ | ✅ Aprovado | `strict: true` obrigatório. Tipagem forte maximiza qualidade do output gerado por IA e reduz erros em operações críticas de supervisão. | Arquivos `.js`/`.jsx` proibidos em novos módulos. |
| **Prisma** | 6.x+ | ✅ Aprovado | Tipagem auto-gerada, migrations declarativas, integração nativa com Supabase. | Único ORM aprovado. TypeORM/Sequelize/Drizzle proibidos. |
| **Pino** | 9.x+ | ✅ Aprovado | Logs estruturados com request ID, timestamp, nível e contexto. Essencial para auditoria de ações Admin (RN-DA-033, RN-DA-035). | `console.log` proibido em produção. Todo log de auditoria Admin via Pino. |
| **Helmet** | 8.x+ | ✅ Aprovado | Headers de segurança automáticos. | Middleware adicionado desde o início do projeto. |
| **class-validator + class-transformer** | latest | ✅ Aprovado | Validação de DTOs integrada aos pipes do NestJS. | Toda entrada de dados validada via pipes, incluindo thresholds de confiança (50%–95%). |
| **@nestjs/swagger** | latest | ✅ Aprovado | Swagger/OpenAPI auto-gerado. | Todo endpoint documentado. Acessível na URL interna do backend. |
| **@nestjs/throttler** | latest | ✅ Aprovado | Rate limiting nativo do NestJS. | Endpoints do webchat com rate limiting por usuário: 30 msg/h (RN-DA-036). |

### 2.2 Banco de Dados

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **PostgreSQL (via Supabase)** | 17+ | ✅ Aprovado | PostgreSQL gerenciado + Auth + Storage + Realtime em uma plataforma. Elimina infra operacional. | Único banco relacional aprovado. MySQL/MariaDB/SQLite/MongoDB proibidos sem ADR. |
| **Supabase Auth** | latest | ✅ Aprovado | Autenticação do Admin herdada da sessão da plataforma Repasse Seguro. | Complementa JWT da plataforma principal. |
| **Supabase Realtime** | latest | 🔶 Condicional | Subscriptions em tempo real para atualização do painel Admin sem polling (dashboard de métricas RN-DA-034, alertas RN-DA-031). | Usar para subscriptions de status de interações e métricas. Justificativa documentada por feature. |

> **⚙️ Convenções de banco:** Conforme ShiftLabs Stacks v7.1 seção 1.5 — UUID v4 como PK, colunas de auditoria (`created_at`, `updated_at`) obrigatórias, soft delete como padrão para tabelas de domínio, `snake_case` para tabelas e colunas, `@db.Timestamptz` obrigatório. Tabelas de log de auditoria de ações Admin (`admin_access_logs`, `alert_events`) permitem hard delete após **365 dias** (1 ano — conforme RNF-004, D12 seção 4 e constante `AUDIT_LOG_RETENTION_DAYS = 365` do D10). Histórico de interações (`interactions`, `takeovers`): soft delete após **90 dias** (RN-DA-036). [CORRIGIDO: PROBLEMA-004]

### 2.3 Cache e Filas

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Redis** | 7.4+ | ✅ Aprovado | Cache de métricas agregadas do dashboard Admin (volume de interações, CSAT médio, top perguntas). Reduz carga no PostgreSQL para dashboards em tempo real. | Todo uso com TTL definido e estratégia de invalidação documentada. TTL padrão para métricas: 60s. Docker local, Upstash/Railway em produção. |
| **RabbitMQ** | 4.x+ | ✅ Aprovado | Filas de disparo de alertas automáticos (Slack, e-mail) para condições críticas: desligamento automático, latência alta, CSAT degradado (RN-DA-031). Processamento assíncrono de notificações de takeover. | Toda fila com retry e dead-letter queue. Docker local, CloudAMQP em produção. |

---

## 3. IA, LLM e Supervisão de Agentes

### 3.1 Modelo de Linguagem (consumido pelo módulo de supervisão)

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **GPT-4 (OpenAI)** | Versão fixada em produção | ✅ Aprovado | LLM padrão ShiftLabs para os agentes supervisionados (Dani-Cessionário, Dani-Cedente). | Fixar versão em produção. Alias genérico (`gpt-4`) somente em desenvolvimento. |
| **OpenAI SDK** | 4.x+ | ✅ Aprovado | Cliente padrão para comunicação com a API da OpenAI pelos agentes supervisionados. | Toda chamada ao LLM via backend. Frontend nunca chama OpenAI diretamente. |

**Fallback de indisponibilidade (vinculado a RN-DA-036, RN-DA-031):**
- Retry com backoff exponencial (3 tentativas, base 1s) para chamadas à API do modelo.
- Se API indisponível: webchat exibe mensagem de degradação; Calculadora de Comissão permanece ativa independentemente.
- Taxa de erro > 30% em 15 minutos: desligamento automático do agente + alerta Admin via Slack e e-mail (RN-DA-031). Admin reativa manualmente via painel.
- Feature flag PostHog como kill switch para desligar features de IA instantaneamente em produção.

### 3.2 Orquestração e Streaming

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Vercel AI SDK** | 4.x+ | ✅ Aprovado | Streaming SSE para o webchat. Integração nativa com NestJS. | Toda interação de chat com streaming usa Vercel AI SDK. SLA: resposta inicial ≤ 5s. |
| **LangChain.js** | 0.3.x+ | ✅ Aprovado | Orquestração de agentes com RAG, tools e pipelines de LLM para Dani-Cessionário e Dani-Cedente. | Pipelines de agente com RAG ou tools obrigatoriamente via LangChain.js. |

### 3.3 Observabilidade de IA

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Langfuse** | latest | ✅ Aprovado | Tracing de interações dos agentes: custo, latência, qualidade, nível de confiança. Fornece dados para dashboard Admin (RN-DA-034). | Obrigatório em produção. Nenhuma feature de IA entra sem tracing configurado. Todo trace inclui: user_id anonimizado, agent_id, confidence_score, latência. |

### 3.4 Segurança de IA

| Controle | Requisito | Regra de Negócio |
|---|---|---|
| **Filtro de escopo** | Toda consulta de dados valida que o recurso pertence ao usuário autenticado antes de qualquer processamento | RN-DA-037 |
| **Filtro de contexto** | Informações fornecidas ao agente contêm apenas dados autorizados para o perfil do usuário | RN-DA-037 |
| **Teste de penetração pré-lançamento** | Cenário de acesso cruzado bloqueado em 100% dos casos testados | RN-DA-037 |
| **Rate limiting por usuário** | 30 msg/h por usuário no webchat (janela deslizante) | RN-DA-036 |
| **Cobertura de cenários de recusa** | Mínimo 20 perguntas adversariais testadas antes do lançamento | RN-DA-038 |

---

## 4. Comunicação e API

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **REST (JSON)** | — | ✅ Aprovado | Padrão ShiftLabs. Simplicidade e compatibilidade com todos os consumidores do painel Admin. | Todo endpoint expõe JSON. Content-Type: `application/json`. |
| **SSE (Server-Sent Events)** | — | ✅ Aprovado | Streaming de respostas do agente para o webchat (Vercel AI SDK). | Exclusivo para endpoints de streaming de chat. |
| **JWT** | — | ✅ Aprovado | Autenticação stateless, compatível com web e mobile. Access token curto + refresh token longo. | Todo endpoint protegido requer JWT válido. Endpoints Admin validam role `ADMIN`. |
| **Supabase Realtime (WebSocket)** | latest | 🔶 Condicional | Subscriptions em tempo real para alertas e atualizações de métricas no painel Admin. | Usar para subscriptions do dashboard. Justificativa documentada por feature. |

**Versionamento de API:** Prefixo `/api/v1/` em todos os endpoints. Mudanças breaking criam nova versão (`/api/v2/`). Versão anterior mantida por mínimo 3 meses.

**Autenticação e autorização:**
- Endpoints do módulo Admin exigem role `ADMIN` no JWT.
- Endpoints de interação de chat exigem role correspondente ao agente (`CESSIONARIO` ou `CEDENTE`).
- Takeover (RN-DA-033): endpoint exclusivo Admin, com lock otimista para prevenir takeover simultâneo.

---

## 5. Repositório e Código

- **Estrutura:** Monorepo com Turborepo + pnpm. AI-Dani-Admin integrado como módulo NestJS dentro do serviço principal de IA da plataforma Repasse Seguro.
- **Nomenclatura de arquivos:** `kebab-case` para arquivos e pastas. Módulos NestJS em `PascalCase`.
- **Nomenclatura de variáveis:** `camelCase` para variáveis TypeScript. `UPPER_SNAKE_CASE` para constantes e enums.
- **Linting:** ESLint + Prettier obrigatórios. Configuração centralizada no monorepo.
- **Code review:** Obrigatório para todo PR que afete lógica de supervisão, alertas ou permissões Admin.

---

## 6. Testes

| Camada | Ferramenta | Cobertura Mínima | Estratégia |
|---|---|---|---|
| **Unitário** | Vitest | 80% para lógica determinística (validação de threshold, cálculo de métricas, condições de alerta) | Mocks via `vi.mock()`. Fixtures de interações pré-definidas. |
| **Integração** | Vitest + Supertest | Endpoints críticos: takeover, configuração de threshold, disparo de alertas | Banco de teste isolado via Supabase Branch. |
| **E2E de supervisão** | Vitest + Playwright | Fluxos completos: Admin vê interação → sinalização → takeover → encerramento | Ambiente staging dedicado. |
| **Evals de IA** | Langfuse Evals | Golden dataset mínimo: 50 exemplos de interações com confidence_score esperado | Executados em CI antes de qualquer merge em produção. |
| **Adversariais** | Manual + Vitest | Mínimo 20 perguntas adversariais (RN-DA-038) | Executados antes de cada lançamento de agente. |

---

## 7. Deploy e CI/CD

| Item | Definição |
|---|---|
| **CI/CD** | GitHub Actions |
| **Ambientes** | `dev` (local Docker), `staging` (Supabase Branch + Railway), `prod` (Supabase + Railway/Vercel) |
| **Pipeline mínimo obrigatório** | lint → type-check → unit tests → integration tests → build → deploy |
| **Gate de lançamento de agente** | Checklist RN-DA-037, RN-DA-038, RN-DA-039 aprovados antes de promote para prod |
| **Requisito de promoção staging → prod** | Zero falhas no pipeline + aprovação manual do Tech Lead + checklist de prontidão completo |

---

## 8. Segurança

> **⚙️ Regras críticas obrigatórias:**
>
> - HTTPS obrigatório em todos os ambientes (exceto localhost).
> - CORS configurado para domínios explícitos. Wildcard proibido em produção.
> - Rate limiting em todos os endpoints públicos e de chat (RN-DA-036: 30 msg/h/usuário).
> - Sanitização de inputs obrigatória antes de qualquer processamento pelo agente.
> - Secrets, tokens e API keys exclusivamente em variáveis de ambiente. Proibido no código-fonte.
> - Logs nunca devem conter PII (nomes, CPFs, e-mails de usuários). Identificadores anonimizados obrigatórios em logs de interação (RN-DA-030).
> - Toda ação Admin (takeover, alteração de threshold, reativação de agente) registrada em log de auditoria com timestamp e identificação do Admin.

> **🔴 Proibições de segurança:**
>
> - Credentials ou secrets no código-fonte ou em arquivos versionados.
> - Logs com dados pessoais identificáveis (PII).
> - Ativação do modelo de IA sem validação completa dos itens de isolamento (RN-DA-037).
> - Lançamento de agente sem supervisão Admin funcional e testada (RN-DA-039).

**LGPD:** Dados de interação armazenados com identificadores anonimizados na listagem pública do painel. Dados pessoais apenas na visualização individual, acessível somente por Admin autenticado. Retenção máxima de histórico: 90 dias (RN-DA-036). Deleção automatizada após janela de retenção.

---

## 9. Analytics e Tracking

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **PostHog** | 1.x+ | ✅ Aprovado | Analytics padrão ShiftLabs. Feature flags para kill switch de IA. Session replay para investigação de incidentes. | Obrigatório em todo produto com usuários. Eventos em `snake_case` via wrapper centralizado. |

**Eventos obrigatórios do módulo AI-Dani-Admin:**

| Evento | Descrição |
|---|---|
| `admin_supervision_panel_viewed` | Admin acessa painel de Supervisão IA |
| `admin_takeover_initiated` | Admin inicia takeover de conversa |
| `admin_takeover_ended` | Admin encerra takeover |
| `admin_threshold_updated` | Admin altera threshold de confiança |
| `agent_auto_disabled` | Agente desligado automaticamente por taxa de erro > 30% |
| `agent_manually_reactivated` | Admin reativa agente após desligamento automático |
| `alert_triggered` | Alerta automático disparado (latência, erro, CSAT, recusa) |
| `interaction_flagged_for_review` | Interação sinalizada por confiança abaixo do threshold |

**Política de dados PostHog:** Sem PII nos eventos. Usar IDs anonimizados para usuários. Propriedades de evento em `snake_case`.

---

## 10. Observabilidade de Infraestrutura

| Tecnologia | Versão Mínima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| **Sentry** | 9.x+ (`@sentry/nestjs`) | ✅ Aprovado | Error tracking com stack traces, breadcrumbs e alertas de infraestrutura. | Obrigatório em produção. Alertas configurados para P0 (erros críticos de supervisão e takeover). |

---

## 11. Tecnologias Proibidas

| Tecnologia | Motivo | Alternativa Aprovada |
|---|---|---|
| **Express puro / Fastify / Hono** | Sem arquitetura modular nativa, sem guards/pipes/interceptors — aumenta superfície de erro em módulos críticos de supervisão | NestJS 10.x+ |
| **TypeORM / Sequelize / Drizzle** | Prisma é o único ORM aprovado. Consistência cross-produto na Shift Labs. | Prisma 6.x+ |
| **MongoDB / DynamoDB / MySQL / MariaDB / SQLite** | Supabase PostgreSQL é o banco padrão. NoSQL sem justificativa cria divergência arquitetural. | Supabase PostgreSQL 17+ |
| **Redux / MobX / Context API como state manager global** | Zustand é o padrão. Redux aumenta boilerplate sem benefício no contexto de agentes de IA. | Zustand 5.x+ |
| **MUI / Chakra / AntD / Bootstrap** | N/A para este módulo (sem frontend próprio). shadcn/ui é o padrão do painel Admin da plataforma. | shadcn/ui (no painel Admin) |
| **jQuery** | Redundante e incompatível com React. | React 19 + hooks nativos |
| **Moment.js** | Deprecated. Bundle pesado (330kB). | date-fns 4.x+ |
| **console.log em produção** | Logs não estruturados, sem rastreabilidade de auditoria. | Pino 9.x+ |
| **`fetch` direto em useEffect sem cache** | N/A para este módulo (sem frontend próprio). Padrão do painel Admin usa TanStack Query. | TanStack Query 5.x+ |

---

## 12. ADRs — Desvios do Baseline

> Nenhum desvio do baseline ShiftLabs Stacks v7.1 nesta versão inicial do documento.

*ADRs serão registrados aqui quando desvios forem necessários. Numeração contínua: ADR-001, ADR-002, etc.*

---

## 13. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial do documento. Stack normativa para o módulo AI-Dani-Admin — Supervisão Operacional. |

---

## 14. Backlog de Pendências

| Item Pendente | Seção | Impacto | Pergunta Objetiva | Dono | Status |
|---|---|---|---|---|---|
| Definir serviço gerenciado de Redis em produção (Upstash vs Railway) | 2.3 | P1 | Qual fornecedor de Redis gerenciado usar em produção? Upstash (serverless) ou Railway (container gerenciado)? | Tech Lead | Aberto |
| Definir fornecedor de RabbitMQ em produção (CloudAMQP vs Railway) | 2.3 | P1 | Qual fornecedor de RabbitMQ gerenciado usar em produção? CloudAMQP ou Railway? | Tech Lead | Aberto |
| Definir TTL de cache Redis para cada métrica do dashboard | 2.3 | P2 | TTL para volume de interações, CSAT médio e top perguntas: 60s é adequado ou deve ser configurável pelo Admin? | Produto | Aberto |
| Definir canal Slack para alertas de produção | 3.4 | P1 | Qual canal Slack recebe alertas automáticos de incidente (desligamento automático, latência alta)? | Admin / Operações | Aberto |
