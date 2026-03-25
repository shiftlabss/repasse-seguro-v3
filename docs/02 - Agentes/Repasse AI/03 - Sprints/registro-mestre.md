# Registro Mestre de Requisitos — Repasse AI

| Campo | Valor |
|---|---|
| **Versão** | v1.0 |
| **Data** | 2026-03-24 |
| **Total de requisitos** | 185 |
| **Cobertura** | 100% atribuídos após Etapa 3 |

---

## Inventário 2.1 — Identidade do Projeto

- **Nome:** Repasse AI
- **Tipo:** Backend puro (sem frontend próprio) — serviço de agente de IA
- **Classificação:** B2B2C · Web + Mobile (via webchat/WhatsApp) · Com agente de IA (LangGraph.js + LangChain.js)
- **Público-alvo:** Cessionário (externo) + Admin (interno)
- **Fase 1:** Webchat integrado à plataforma
- **Fase 2:** WhatsApp Business via EvolutionAPI

## Inventário 2.2 — Stack Tecnológica

**Backend:**
- Node.js 22.x LTS · NestJS 10.x+ · TypeScript 5.4+ (strict: true)
- Prisma 6.x+ (único ORM — TypeORM/Sequelize/Drizzle proibidos)
- Supabase PostgreSQL 17+ (pgvector 0.7+)
- Redis 7.4+ (Upstash em produção) — cache LLM + rate limiting
- RabbitMQ 4.x+ (CloudAMQP em produção) — filas assíncronas
- Pino 9.x+ — logs estruturados (`console.log` proibido em produção)
- Helmet 8.x+ · class-validator + class-transformer · @nestjs/swagger · @nestjs/throttler

**IA / LLM:**
- GPT-4 / GPT-4o via OpenAI SDK 4.x+ (versão fixada em produção)
- GPT-4o-mini (condicional — tarefas determinísticas baixo custo, temperature 0 obrigatório)
- Vercel AI SDK 4.x+ — streaming SSE
- LangChain.js 0.3.x+ — RAG, agents, tools
- LangGraph.js 0.2.x+ — fluxo stateful do agente
- Langfuse SDK — observabilidade de IA (tracing, custo, latência, qualidade)
- Supabase pgvector — vector store (Pinecone/Qdrant/Weaviate proibidos sem ADR)
- OpenAI Embeddings `text-embedding-3-small` (1536 dimensões)
- Cohere Rerank (condicional — segunda camada de relevância)

**Comunicação:**
- API REST (JSON) + SSE (Server-Sent Events) — obrigatórios
- Supabase Realtime (condicional — notificações proativas)
- WebSocket: proibido sem ADR
- fetch nativo (Axios proibido) · GraphQL/gRPC proibidos sem ADR

**Testes:**
- Jest 29+ + ts-jest (unitários e integração)
- Supertest (integração + E2E)
- Pirâmide: 70% unitários / 25% integração / 5% E2E
- Sem Playwright/Cypress (backend puro, PG-03)
- Langfuse Evals (golden datasets — mínimo 50 exemplos)

**Infraestrutura:**
- Railway (deploy NestJS — ADR-004) — sem cold starts
- GitHub Actions (CI/CD)
- Sentry (error tracking)
- pnpm 9.x+ · Turborepo 2.x+ · Husky + lint-staged

**Libs proibidas sem ADR:** Axios, TypeORM, Sequelize, Drizzle, Pinecone, Qdrant, Weaviate, WebSocket, GraphQL, gRPC, MySQL, SQLite, MongoDB, Express, Fastify, Hono, npm, yarn

## Inventário 2.3 — Banco de Dados

**13 tabelas em 6 domínios:**

| Tabela | Domínio | Soft Delete | Observação |
|---|---|---|---|
| `chat_conversations` | Conversas | Sim (`deleted_at`) | PII — retenção 90 dias |
| `chat_messages` | Conversas | Sim (`deleted_at`) | PII — imutável após criação, sem `updated_at` |
| `ai_interactions` | Interações | Sim (`deleted_at`) | Fonte painel Admin |
| `llm_cache_entries` | Análise/Cache | Não | Cache exact + semantic |
| `document_embeddings` | Análise/Cache | Não | pgvector — sem FK para usuários |
| `whatsapp_bindings` | WhatsApp | Sim (`deleted_at`) | PII |
| `otp_attempts` | WhatsApp | Não | Append-only, sem `updated_at` |
| `notification_preferences` | Notificações | Não | — |
| `notification_events` | Notificações | Não | — |
| `ai_takeovers` | Admin/Config | Não | — |
| `agent_configurations` | Admin/Config | Não | — |
| `agent_status_logs` | Admin/Config | Não | Append-only, sem `updated_at` |
| `lgpd_consents` | Admin/Config | Não | Append-only, sem `updated_at` |

**17 enums PostgreSQL** (nomenclatura `{domínio}_status_enum`)

**Convenções obrigatórias:**
- UUID v4 como PK (`gen_random_uuid()`)
- `snake_case` tabelas (plural) e colunas (singular)
- `TIMESTAMPTZ` obrigatório em todos os timestamps
- `created_at` + `updated_at` em todas as tabelas de domínio (exceto append-only)
- Soft delete global via middleware Prisma

**Ordem de aplicação em produção:** `prisma migrate deploy` → `policies.sql` (RLS) → `indexes.sql` → `npx prisma db seed` (9 configurações obrigatórias)

## Inventário 2.4 — API / Endpoints

**Base URL:** `https://api.repasseseguro.com.br/repasse-ai/v1`
**Total:** 22 endpoints + 1 health check + 1 webhook EvolutionAPI

| Domínio | Endpoints |
|---|---|
| Health | `GET /health` |
| Chat | `POST /chat/conversations` · `GET /chat/conversations` · `GET /chat/conversations/{id}/messages` · `POST /chat/conversations/{id}/messages` (SSE) · `DELETE /chat/history` |
| AI | `GET /ai/interactions` · `GET /ai/interactions/{id}` |
| Calculator | `POST /calculator/commission` · `POST /calculator/escrow` · `POST /calculator/roi` |
| Notification | `GET /notifications/preferences` · `PATCH /notifications/preferences` · `GET /notifications/events` |
| Supervision (Admin) | `GET /supervision/interactions` · `GET /supervision/interactions/{id}` · `POST /supervision/takeovers` · `GET /supervision/takeovers/{id}` · `PATCH /supervision/takeovers/{id}/release` · `GET /supervision/metrics` · `GET /supervision/alerts` · `GET /supervision/agent-config` · `PATCH /supervision/agent-config` |
| WhatsApp | `POST /whatsapp/bindings` · `GET /whatsapp/bindings` · `DELETE /whatsapp/bindings/{id}` · `POST /whatsapp/webhook` (EvolutionAPI) |

**Padrão de erro:** `{ "error": { "code": "ERROR_CODE", "message": "...", "details": {} } }`
**Paginação:** `page` (default: 1) + `per_page` (default: 20, máx: 100) + objeto `meta`
**HTTP codes obrigatórios:** 200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500, 503

## Inventário 2.5 — Perfis de Acesso (RBAC)

| Perfil | Acesso principal |
|---|---|
| `CESSIONARIO` | Próprias conversas, oportunidades do marketplace, cálculos, simulações, notificações próprias, exclusão de histórico próprio |
| `ADMIN` | Todas as interações (supervisão), takeover, métricas, configuração do agente, reativação após desligamento |
| `CEDENTE` | Nenhum acesso ao Repasse AI |

**Regras de isolamento:**
- Cessionário nunca vê dados de outros Cessionários (filtro por `cessionario_id`)
- Cessionário nunca vê dados do Cedente (nome, CPF, contato, negociações, histórico)
- Cessionário nunca vê Cenário A/B/C/D escolhido pelo Cedente
- Três camadas de isolamento: filtro de escopo → filtro de contexto → reforço nas instruções do agente
- JWT herdado da sessão da plataforma — Guard de Cessionário + Guard de Admin no NestJS
- RLS (Row Level Security) habilitado no Supabase

## Inventário 2.6 — Módulos do Sistema

| Módulo | Descrição | RNs Principais |
|---|---|---|
| Fundação/Isolamento | Identidade agente, RBAC, isolamento de dados (3 camadas) | RN-001 a RN-004 |
| Onboarding/Sessão | Primeiro uso, pontos de entrada, autenticação por herança, sugestões | RN-005 a RN-008 |
| Histórico | Retenção 90 dias, exclusão voluntária, soft delete | RN-009, RN-010 |
| Análise de Oportunidade | Análise individual, score de risco | RN-011, RN-012 |
| Cálculo de Comissão e Escrow | Calculadora determinística independente | RN-013, RN-014 |
| Comparação | Até 5 oportunidades | RN-015 |
| Simulação | Propostas, contrapropostas, portfólio, variação | RN-016 a RN-020 |
| Recomendação Proativa | Top 3 oportunidades | RN-021 |
| Suporte Operacional | Regras, prazos, status | RN-022, RN-022.a, RN-022.b |
| Fallback/Calculadora | Fallback determinístico quando IA indisponível | RN-023, RN-024 |
| Rate Limit Webchat | 30 msg/hora por Cessionário, janela deslizante | RN-025 |
| Fluxos de Conversação | Fluxos principais documentados | RN-026 a RN-028 |
| SLA e Disponibilidade | SLAs por tipo de interação, latência | RN-029 |
| Supervisão Admin | Monitoramento, alertas automáticos | RN-030, RN-031 |
| Takeover | Takeover manual Admin, mutex concorrência | RN-032, RN-033 |
| Métricas Admin | Dashboard de métricas | RN-034 |
| Configuração do Agente | Threshold de confiança (50%-95%, padrão 80%) | RN-035 |
| Canal Webchat Config | Parâmetros 24/7, disponibilidade API | RN-036 |
| Prontidão para Lançamento | Isolamento, cobertura recusas, supervisão | RN-037 a RN-039 |
| Canal WhatsApp (Fase 2) | Vinculação, OTP, re-verificação, desvinculação | RN-040 a RN-046 |
| Notificações Proativas | Alertas de oportunidade, Escrow, status | RN-047 a RN-050 |
| LGPD/Privacidade | Consentimento, exclusão, anonimização | RN-051 a RN-053 |

## Inventário 2.7 — Regras de Negócio com Valores Numéricos

| Regra | Valor |
|---|---|
| Comissão padrão | 20% × Δ (quando Δ > 0) |
| Comissão fallback | 20% × Valor Pago pelo Cedente (quando Δ ≤ 0) |
| Custo total Escrow | Preço Repasse + Comissão |
| ROI projetado | (Tabela Atual − Custo Total) ÷ Custo Total × 100 |
| Cenário conservador ROI | Valorização 20% abaixo da estimativa base |
| Cenário otimista ROI | Valorização 20% acima da estimativa base |
| Score de risco | Escala 1 a 10 (verde 1-3, amarelo 4-6, vermelho 7-10) |
| Comparação simultânea | Máximo 5 oportunidades |
| Rate limit webchat | 30 mensagens / 1 hora (janela deslizante) por Cessionário |
| Rate limit WhatsApp | 20 mensagens / 1 hora por Cessionário |
| Prazo Escrow padrão | 10 dias úteis |
| Extensão Escrow | +5 dias úteis (aprovação Admin) |
| Reversão Escrow | 15 dias corridos se negociação não concluída |
| Retenção histórico | 90 dias (LGPD) |
| Exclusão histórico (conta encerrada) | Até 48 horas |
| Taxa de erro → alerta Admin | > 10% em 15 minutos |
| Taxa de erro → desligamento automático | > 30% em 15 minutos |
| Threshold de confiança padrão | 80% |
| Threshold mínimo configurável | 50% |
| Threshold máximo configurável | 95% |
| SLA análise individual | ≤ 5 segundos |
| SLA comparação até 5 | ≤ 10 segundos |
| SLA simulação | ≤ 5 segundos |
| SLA suporte operacional | ≤ 5 segundos |
| Tolerância latência | 2× o SLA antes de exibir fallback |
| Latência alta (alerta) | Acima do SLA por 5 minutos consecutivos |
| OTP SMS validade | 15 minutos |
| OTP tentativas por hora | 3 tentativas |
| OTP bloqueio por excesso | 30 minutos |
| OTP falhas consecutivas → bloqueio | 5 falhas → 30 minutos |
| Re-verificação WhatsApp | A cada 30 dias |
| Prazo re-verificação | 48 horas para responder |
| Cooldown reenvio OTP | 60 segundos |
| OTP re-verificação expiração | 7 dias em suspenso → desvinculação automática |
| Histórico lazy-load | 20 mensagens mais recentes, paginação para anteriores |
| Janela histórico skeleton | Máximo 3 segundos |
| Fade-in mensagem boas-vindas | 300ms |
| Animação liberação rate limit | Pulse 500ms |
| Alerta Escrow (prazo) | 2 dias úteis antes do vencimento |
| Utilização semanal meta Fase 1 | ≥ 30% Cessionários ativos |
| CSAT meta Fase 1 | ≥ 4,2 / 5 |
| CSAT gate Fase 2 | ≥ 4,0 / 5 |
| Redução chamados Admin meta Fase 1 | −30% |
| CSAT degradado (alerta) | Média < 3,5 de 5 em 24 horas |
| Taxa de recusa alta (alerta) | > 20% respostas com recusa em 24 horas |
| Consumo processamento (alerta) | > 80% orçamento mensal |
| Retry LLM backoff | 3 tentativas, base 1s exponencial |
| Memória sessão | 20 mensagens (Redis TTL 30 min) |
| Chunking RAG | 512-1024 tokens, overlap 10-20% |
| Similarity threshold RAG | 0.78 |
| Cache semântico threshold | 0.92 |
| Cache exact TTL | 24h |
| Cache semântico TTL | 1h |
| Golden dataset mínimo evals | 50 exemplos |
| Temperatura LLM padrão | 0.1 |
| Max tokens response | 4096 |
| Timeout ferramenta calculate_* | 5s (retry 2x, 500ms backoff) |
| Timeout ferramenta get_* | 5s (retry 3x, exponencial) |
| Timeout calculate_portfolio | 10s (retry 2x, 1s backoff) |
| Timeout stream SSE | 120 segundos |
| Stream início timeout | 10 segundos (então fallback) |
| Cobertura mínima módulo `ai` | 85% |
| Cobertura mínima módulo `chat` | 80% |
| Cobertura mínima módulo `whatsapp` | 80% |
| Cobertura mínima módulo `supervision` | 75% |
| Cobertura mínima módulo `notification` | 75% |
| Cobertura mínima módulo `calculator` | 75% |
| Cobertura mínima módulo `common` | 90% |
| Acesso token JWT expiração | 1 hora |
| Versão API via path | `/repasse-ai/v1` |
| Deprecação versão antiga | 90 dias após deprecação |

**Máquinas de Estado:**

| Entidade | Transições |
|---|---|
| `chat_conversations` status | `active` → `closed` (Cessionário fecha) · `closed` → `active` (reabre) |
| Sessão de chat | `Nova` → `Ativa` → `Encerrada` → `Ativa` · `Nova` → `BloqueadaPorKYC` → `Ativa` |
| Escopo de dados | `Escopo autorizado` ↔ `Escopo bloqueado` |
| Serviço de IA | `Operacional` → `LatenciaAlta` ↔ `Operacional` · `LatenciaAlta` → `FallbackAtivo` · `Operacional` → `FallbackAtivo` · `FallbackAtivo` → `Operacional` · `FallbackAtivo` → `DesligadoAutomatico` → `Operacional` (manual) |
| Oportunidade (visível ao agente) | `Disponivel` → `EmNegociacao` → `Disponivel` ou `Encerrada` · `Disponivel` → `Encerrada` |
| Simulação | `SimulacaoIniciada` → `CalculoRealizado` → `ResultadoExibido` → `SimulacaoIniciada` ou `PropostaSubmetida` ou `SimulacaoEncerrada` |
| Vinculação WhatsApp | `NaoVinculado` → `AguardandoOTP` → `NaoVinculado` (expirado) ou `AguardandoConfirmacao` → `Ativo` → `AguardandoReVerificacao` → `Ativo` ou `Suspenso` → `Desvinculado` · `Ativo` → `Desvinculado` |
| Interação | `RespondidaPelaIA` → `Encerrada` · `SinalizadaParaRevisao` → `EmTakeover` → `Encerrada` · `SinalizadaParaRevisao` → `RespondidaPelaIA` |
| LGPD consentimento | `NaoConsentida` → `Ativa` |
| Histórico | `Ativo` → `Expirado` (90 dias) · `Ativo` → `Excluído` (solicitação) · `Ativo` → `Anonimizado` |

## Inventário 2.8 — Integrações Externas

| Integração | Criticidade | Env var principal | Fallback |
|---|---|---|---|
| OpenAI API (GPT-4o + embeddings) | P0 | `OPENAI_API_KEY` | Calculadora de Comissão (cálculos determinísticos) |
| Supabase PostgreSQL 17+ (pgvector + Auth) | P0 | `DATABASE_URL` | Nenhum — crítico |
| Upstash Redis 7.4+ | P0 | `REDIS_URL` | Nenhum — crítico |
| EvolutionAPI (WhatsApp Business) | P1 (Fase 2) | `EVOLUTION_API_URL` · `EVOLUTION_API_KEY` · `EVOLUTION_API_WEBHOOK_SECRET` | Mock via `FEATURE_WHATSAPP_MOCK=true` |
| CloudAMQP (RabbitMQ 4+) | P1 | `RABBITMQ_URL` | DLQ com retry exponencial (max 3) |
| Langfuse (AI Observability) | P1 | `LANGFUSE_PUBLIC_KEY` · `LANGFUSE_SECRET_KEY` | Nenhum — observabilidade |
| Sentry (Error Tracking) | P2 | `SENTRY_DSN` | Nenhum — degradação tolerada |
| Railway (Deploy) | P2 | N/A (infra) | Rollback Railway < 3 min |

**Total env vars:** 18 (9 sensíveis / 9 não sensíveis)
**Outras:** `JWT_SECRET` · `JWT_DEV_MODE` · `LOG_LEVEL` · `FEATURE_WHATSAPP_MOCK` · `NODE_ENV`

**Filas RabbitMQ:** 4 exchanges, 6 queues, todas com DLQ e retry exponencial (max 3 tentativas)

## Inventário 2.9 — Glossário / Terminologia

**Termos críticos (Doc 01.1 — Glossário de Domínio):**

| Termo | Detalhe crítico de implementação |
|---|---|
| **Δ (Delta)** | Tabela Atual − Tabela Contrato. Se Δ ≤ 0, fallback: 20% × Valor Pago pelo Cedente |
| **Calculadora de Comissão** | Módulo determinístico independente do agente — funciona sem IA |
| **Cedente** | Dados pessoais NUNCA expostos ao Cessionário |
| **Cenários A/B/C/D** | Dado confidencial — NUNCA revelado ao Cessionário nem ao agente |
| **Cessionário** | Usuário-alvo — isolamento total de dados |
| **Comissão Comprador** | 20% × Δ (geral) / 20% × Valor Pago pelo Cedente (fallback Δ ≤ 0). Descontos APENAS pelo Admin |
| **Dossiê** | Documentos obrigatórios de validação |
| **Envelope ZapSign** | Assinatura eletrônica para contrato de cessão |
| **Escrow** | Conta garantia: prazo padrão 10 dias úteis / extensão +5 dias (Admin) |
| **Guardião do Retorno** | Agente do Cedente — compartilha infraestrutura de supervisão, identificado por filtro de nome |
| **KYC** | Documento ID (frente + verso) + selfie vivacidade + comprovante endereço (até 90 dias) |
| **OPR-XXXX-XXXX** | Código identificador único de oportunidade no marketplace |
| **OTP** | 6 dígitos, 3 tentativas/hora, bloqueio 30 min após 5 falhas consecutivas |
| **RBAC** | Cada Cessionário acessa apenas próprios dados |
| **Score de Risco** | Escala 1–10 calculada pelo agente |
| **Tabela Atual** | Preço vigente do imóvel conforme incorporadora |
| **Tabela Contrato** | Preço na data do contrato original |
| **Takeover** | Intervenção manual Admin quando confiança < threshold (padrão 80%) |

## Inventário 2.10 — Qualidade e Go-Live

- **Pirâmide:** 70% unitários / 25% integração / 5% E2E (10-15 fluxos)
- **E2E obrigatórios:** Webchat → agente → SSE · WhatsApp OTP → binding · Takeover Admin
- **Cobertura mínima por módulo:** (ver seção 2.7 valores numéricos)
- **Gates de CI:** lint + typecheck + unitários + integração + build + security audit
- **Deploy:** Railway automático em merge para `main` / `staging`
- **Rollback:** Railway "Redeploy previous deployment" < 3 min
- **Smoke tests:** `pnpm test:smoke staging` após deploy staging
- **Go/no-go:** `http_error_rate < 2%` + `GET /health` 200 + circuit breaker CLOSED + sem CRITICAL Sentry
- **T-7:** infraestrutura + integrações staging · T-3: QA + E2E · T-1: freeze + smoke + war room
- **Launch Day:** Terça ou Quinta, 14h BRT
- **War room:** Slack #repasse-ai-launch (T-1 até T+1h estável)
- **Observabilidade:** Langfuse (tracing IA) + Sentry (infraestrutura) + Railway Metrics
- **Pré-requisitos de lançamento (RN-037/038/039):** filtro de escopo validado + 20 perguntas adversariais + supervisão funcional

## Inventário 2.11 — UX e Estados de Tela

**Estados obrigatórios por feature:**

| Feature | Loading | Empty State | Error State | Sem Permissão |
|---|---|---|---|---|
| Chat (abertura) | Skeleton loading balões (máx 3s) | Sugestões de conversa (chips) | "Serviço temporariamente indisponível" + ícone alerta amarelo | Banner consentimento + campo desabilitado |
| Histórico paginado | Skeleton loading | — | — | — |
| Widget Top 3 Dashboard | — | "Não foi possível carregar recomendações" + botão "Tentar novamente" | — | — |
| Contexto oportunidade (load 3s) | "Carregando dados da oportunidade..." | — | "Não consegui carregar dados automaticamente. Informe o código OPR" | — |
| Supervisão Admin (lista) | Skeleton inline | "Nenhuma interação registrada..." + sugestão filtros | — | 403 |
| Métricas Admin (card) | — | "Dados insuficientes para o período" (nunca 0) | — | — |

**Feedbacks visuais obrigatórios:**
- FAB global: badge numérica para alertas não lidos
- FAB: badge amarela quando serviço degradado
- Score de risco: verde (1-3) / amarelo (4-6) / vermelho (7-10) + rótulo acessível
- Contraste mínimo 4.5:1 (WCAG 2.1 AA)
- Área de toque mínima: 48×48px (FAB) / 44×44px (chips)
- Chips navegáveis por teclado (Tab + Enter) + rótulo screen reader
- Indicador "digitando" (3 pontos pulsando) durante stream SSE
- Counter regressivo rate limit (mm:ss em tempo real)
- Toast confirmação: 5 segundos visível
- Modal exclusão histórico: botão destrutivo vermelho sem atalho Enter
- Fade-in mensagem boas-vindas: 300ms
- Separador visual transição IA ↔ humano no chat (takeover)
- Banner "Modo básico" (cinza/azul claro) em fallback calculadora
- Diferença proposta: seta verde (economia) / seta vermelha (acréscimo) + valor absoluto e percentual
- Tabela comparativa: scroll horizontal mobile, coluna critério ativa destacada, linhas clicáveis
- Aviso ROI: sempre visível, não pode ser ocultado pelo Cessionário

**Sprints condicionais:**
- Agentes de IA: **SIM** — Doc 19 confirma (LangGraph.js + LangChain.js + tools)
- Mobile: **NÃO** — Doc 02 confirma que Repasse AI é backend puro (PG-03), sem frontend próprio. Doc 11 não se aplica a este módulo.

---

## Tabela de Requisitos Atômicos

| ID | Doc Fonte | Seção | Tipo | Descrição | Valor/Detalhe | Sprint |
|----|-----------|-------|------|-----------|---------------|--------|
| REQ-001 | D01.1 | Glossário | Termo | Δ (Delta) | Tabela Atual − Tabela Contrato; se Δ ≤ 0 → fallback 20% × Valor Pago pelo Cedente | CROSS |
| REQ-002 | D01.1 | Glossário | Termo | Calculadora de Comissão | Módulo determinístico independente do agente de IA | S4 |
| REQ-003 | D01.1 | Glossário | Regra | OTP | 6 dígitos; 3 tentativas/hora; bloqueio 30 min após 5 falhas consecutivas | S7 |
| REQ-004 | D01.1 | 3.1 | RBAC | Perfil Cessionário | Acessa apenas próprias conversas, oportunidades marketplace, cálculos, simulações, notificações | S2 |
| REQ-005 | D01.1 | 3.1 | RBAC | Perfil Admin | Todas as interações, takeover, métricas, configuração agente, reativação | S2 |
| REQ-006 | D01.1 | 3.2 | RBAC | Matriz permissões agente | Cessionário: próprios dados; Admin: todas; Cedente: nenhum | S2 |
| REQ-007 | D01.1 | 4.1 | Identidade | Nome exibido na interface | "Analista de Oportunidades" | S1 |
| REQ-008 | D01.1 | 4.2/4.3 | Identidade | Tom de voz | Analítico, objetivo, dados concretos; sem FOMO, sem superlativos, sem garantias financeiras, sem jurídico | S3 |
| REQ-009 | D01.1 | RN-001 | Regra de negócio | Escopo de dados acessíveis ao agente | Oportunidades marketplace (anonimizado), propostas/negociações/Escrow/comissões próprias, dados públicos empreendimento, histórico conversas próprio | S3 |
| REQ-010 | D01.1 | RN-001 | Regra de negócio | Recusa de escopo em ≤ 2 segundos | Sem indicador de carregamento prolongado | S3 |
| REQ-011 | D01.1 | RN-002 | Regra de negócio | Dados que o agente nunca acessa | Dados pessoais/financeiros Cedente, cenário A/B/C/D, propostas/negociações outros Cessionários, logs internos Admin | S3 |
| REQ-012 | D01.1 | RN-003 | Regra de negócio | Três camadas de isolamento | Filtro de escopo + filtro de contexto + reforço nas instruções do agente | S3 |
| REQ-013 | D01.1 | RN-003 | Regra de negócio | Falha em camada de isolamento → recusa total | "O serviço de análise está temporariamente indisponível." + ícone alerta amarelo + campo ativo | S3 |
| REQ-014 | D01.1 | RN-004 | UX Writing | 7 mensagens padrão para dados bloqueados | Dados Cedente, qtd outros Cessionários, cenário Cedente, negociações outros, garantia financeira, conselho jurídico/fiscal, alteração perfil/KYC | S3 |
| REQ-015 | D01.1 | RN-004 | Regra de negócio | Insistência Cessionário em dados bloqueados | 1ª: recusa + alternativa; 2ª: recusa + "Posso ajudá-lo com análises dentro do seu perfil. Veja algumas opções:"; 3ª: recusa sem alternativas (sem loop) | S3 |
| REQ-016 | D01.1 | RN-005 | UX | Mensagem boas-vindas primeiro acesso | "Olá! Sou o Analista de Oportunidades. Posso analisar riscos, comparar imóveis e simular retornos para você. Como posso ajudar?" + fade-in 300ms + avatar | S3 |
| REQ-017 | D01.1 | RN-005 | UX | KYC pendente no primeiro acesso | Mensagem boas-vindas + orientação link clicável "Meu Perfil > Verificação de Identidade" | S3 |
| REQ-018 | D01.1 | RN-005 | UX | Sem oportunidades no marketplace | Informa + botão ação rápida "Ativar alertas" → confirmação inline | S3 |
| REQ-019 | D01.1 | RN-006 | UX | Ponto de entrada: Tela de Oportunidade | Carrega contexto OPR automático; loading "Carregando dados da oportunidade..." (máx 3s); fallback se falhar | S3 |
| REQ-020 | D01.1 | RN-006 | UX | Ponto de entrada: Dashboard (widget) | Exibe Top 3 recomendações; empty state + "Tentar novamente" se falhar | S3 |
| REQ-021 | D01.1 | RN-006 | UX | Ponto de entrada: FAB global | Abre sem contexto específico; badge numérica alertas não lidos; mobile: canto inferior direito, margem 16px, toque mínimo 48×48px | S3 |
| REQ-022 | D01.1 | RN-007 | Regra de negócio | Autenticação por herança de sessão | Sem novo login se sessão ativa; redireciona login se expirada; preserva ponto de entrada após re-login | S2 |
| REQ-023 | D01.1 | RN-007 | UX | Sessão expirada durante conversa | Banner temporário topo + botão "Fazer login" + campo desabilitado | S2 |
| REQ-024 | D01.1 | RN-008 | UX | Sugestões de conversa sem oportunidade pré-carregada | 4 chips: "Quais são as melhores oportunidades para mim hoje?", "Tenho R$ 500.000 para investir. O que recomenda?", "Me explica como funciona a comissão do comprador.", "Qual o prazo para depósito em Escrow?" | S3 |
| REQ-025 | D01.1 | RN-008 | UX | Sugestões de conversa com oportunidade pré-carregada | 4 chips contextualizados: "Analise essa oportunidade em detalhes.", "Compare com as 3 melhores da mesma região.", "Quanto preciso depositar no total se propor R$ 300.000?", "Qual o score de risco dessa oportunidade?" | S3 |
| REQ-026 | D01.1 | RN-008 | Acessibilidade | Chips de sugestão | Área toque mínima 44×44px; navegáveis Tab+Enter; rótulo acessível screen readers | S3 |
| REQ-027 | D01.1 | RN-009 | Regra de negócio | Retenção histórico 90 dias | Soft delete após 90 dias; skeleton loading (máx 3s); lazy-load 20 mensagens + "Carregar mensagens anteriores" + scroll infinito | S3 |
| REQ-028 | D01.1 | RN-009 | LGPD | Consentimento explícito no primeiro uso | Antes de qualquer processamento de mensagem | S2 |
| REQ-029 | D01.1 | RN-010 | UX | Modal exclusão histórico | Título "Apagar histórico de conversas"; corpo com aviso irreversível; botões "Cancelar" (esquerda) + "Apagar histórico" (destrutivo, vermelho, direita); botão destrutivo sem atalho Enter | S3 |
| REQ-030 | D01.1 | RN-010 | UX | Exclusão confirmada | Toast de sucesso (barra verde, 5 segundos) + chat recarregado vazio + sugestões RN-008 | S3 |
| REQ-031 | D01.1 | RN-010 | Regra de negócio | Encerramento de conta | Histórico conversas excluído em até 48 horas; dados financeiros retidos [PENDENTE JURÍDICO] | S3 |
| REQ-032 | D01.2 | RN-011 | Análise | Análise de oportunidade individual | Δ + comissão + custo total + ROI (3 cenários) + score de risco + comparativo regional + gráfico valorização | S3 |
| REQ-033 | D01.2 | RN-011 | UX | Status "Em negociação" | Badge laranja + oferta de notificação se disponível novamente | S3 |
| REQ-034 | D01.2 | RN-011 | UX | Status "Encerrada" | Badge cinza + até 3 sugestões chips com OPR | S3 |
| REQ-035 | D01.2 | RN-011 | UX | Oportunidade não encontrada | Mensagem + ícone lupa | S3 |
| REQ-036 | D01.2 | RN-011 | UX | Comparativo regional sem dados | Informa explicitamente "Não há dados suficientes..." — não omite silenciosamente | S3 |
| REQ-037 | D01.2 | RN-011 | UX | Gráfico valorização em WhatsApp | Substituir por tabela textual; se não disponível: informa explicitamente | S5 |
| REQ-038 | D01.2 | RN-012 | Análise | Score de risco | Escala 1-10; verde 1-3 / amarelo 4-6 / vermelho 7-10; contraste ≥ 4.5:1; rótulos acessíveis; listagem fatores considerados | S3 |
| REQ-039 | D01.2 | RN-012 | Análise | Score de risco insuficiente | Omissão total (sem score parcial/estimado) + mensagem explícita | S3 |
| REQ-040 | D01.2 | RN-013 | Cálculo | Comissão padrão | 20% × Δ quando Δ > 0 | S4 |
| REQ-041 | D01.2 | RN-013 | Cálculo | Comissão fallback | 20% × Valor Pago pelo Cedente quando Δ ≤ 0 | S4 |
| REQ-042 | D01.2 | RN-013 | Cálculo | Desconto comissão | Exclusivamente pelo Admin — agente não aplica | S4 |
| REQ-043 | D01.2 | RN-013 | UX | Formatação monetária | Separador milhar (ponto) + duas casas decimais (ex: R$ 30.000,00) | S4 |
| REQ-044 | D01.2 | RN-013 | UX | Nota explicativa fallback | "Como a Tabela Atual não é superior à Tabela Contrato, a comissão é calculada sobre o Valor Pago pelo Cedente." | S4 |
| REQ-045 | D01.2 | RN-014 | Cálculo | Custo total Escrow | Preço Repasse + Comissão | S4 |
| REQ-046 | D01.2 | RN-015 | Comparação | Comparação até 5 oportunidades | Campos: Δ, Comissão, Custo total Escrow, Score risco, Localização, Tipologia | S5 |
| REQ-047 | D01.2 | RN-015 | UX | Tabela comparativa | Scroll horizontal mobile; coluna critério ativa destacada; linhas clicáveis → análise RN-011; badge "Melhor opção" primeiro colocado | S5 |
| REQ-048 | D01.2 | RN-015 | Regra de negócio | Limite 5 oportunidades | Mensagem de limite se > 5 solicitadas | S5 |
| REQ-049 | D01.2 | RN-015 | Regra de negócio | Desempate comparação | Maior Δ como critério de desempate quando mesmo score de risco | S5 |
| REQ-050 | D01.2 | RN-016 | Simulação | Simulação de custos para proposta | Comissão + Escrow + ROI 3 cenários sobre valor informado | S5 |
| REQ-051 | D01.2 | RN-016 | UX | Valor inválido em simulação | Mensagem erro + ícone triângulo amarelo + campo ativo para nova tentativa | S5 |
| REQ-052 | D01.2 | RN-016 | UX | Próximos passos simulação | Botões ação: "Simular outro valor" + "Ir para a oportunidade" | S5 |
| REQ-053 | D01.2 | RN-017 | Cálculo | ROI com 3 cenários | Fórmula: (Tabela Atual − Custo Total) ÷ Custo Total × 100; conservador: −20%; base: estimativa; otimista: +20% | S5 |
| REQ-054 | D01.2 | RN-017 | UX | Aviso obrigatório ROI | "Esses são valores projetados... Resultados reais podem variar." — não pode ser ocultado | S5 |
| REQ-055 | D01.2 | RN-017 | UX | Visual 3 cenários | Conservador (escudo, cor neutra) / base (alvo, padrão, destacado) / otimista (tendência ascendente, destaque) | S5 |
| REQ-056 | D01.2 | RN-018 | Simulação | Simulação de contraproposta | Nova comissão + novo Escrow + diferença (seta verde economia / seta vermelha acréscimo + abs + %) + ROI ajustado 3 cenários | S5 |
| REQ-057 | D01.2 | RN-018 | Regra de negócio | Iterações simulação | Sem limite de iterações na mesma sessão | S5 |
| REQ-058 | D01.2 | RN-019 | Simulação | Portfólio multi-oportunidade | Capital total necessário + suficiência/insuficiência/excedente + ROI portfólio + ROI individual | S5 |
| REQ-059 | D01.2 | RN-019 | UX | Capital insuficiente | Déficit em destaque (negrito, cor alerta) + sugestão ranqueamento + botão "Ranquear por prioridade" | S5 |
| REQ-060 | D01.2 | RN-020 | Simulação | Impacto de variação de valorização | ROI ajustado pela variação; se negativo: informa claramente "resultado negativo... perda estimada de R$..." | S5 |
| REQ-061 | D01.2 | RN-020 | Regra de negócio | Aviso obrigatório variação | "Esta é uma projeção baseada nos dados disponíveis. Resultados reais podem variar." | S5 |
| REQ-062 | D01.2 | RN-021 | Recomendação | Top 3 oportunidades em destaque | Melhor relação retorno/risco compatível com perfil; campos: Δ, comissão estimada, score de risco, localização | S3 |
| REQ-063 | D01.2 | RN-021 | UX | Perfil incompleto no Top 3 | Banner informativo "Recomendações baseadas em dados gerais" + link clicável "Meu Perfil > Preferências" | S3 |
| REQ-064 | D01.3 | RN-022 | Suporte | Regras plataforma (KYC, Escrow, ZapSign, fechamento, status) | Resposta objetiva com prazos e próximo passo | S3 |
| REQ-065 | D01.3 | RN-022 | UX | Redirecionamento fora do escopo | Link clicável para canal de suporte quando aplicável | S3 |
| REQ-066 | D01.3 | RN-022.a | Suporte | Prazos SLA | Escrow: 10 dias úteis; extensão: +5 dias úteis (Admin); reversão: 15 dias corridos; KYC: [PENDENTE DEFINIÇÃO] | S3 |
| REQ-067 | D01.3 | RN-022.b | Suporte | Status de proposta | Status definidos nas RNs do módulo Cessionário (referência cruzada) — [PENDENTE BKL-003] | S3 |
| REQ-068 | D01.3 | RN-023 | Fallback | Calculadora de Comissão independente | Fórmulas: comissão + custo Escrow + ROI determinísticos sem IA | S4 |
| REQ-069 | D01.3 | RN-023 | UX | Banner fallback | Banner "Modo básico — sem análise da IA" (cinza/azul claro); Cessionário pode copiar valores e solicitar novo cálculo | S4 |
| REQ-070 | D01.3 | RN-024 | Operação | Desligamento automático | Taxa erro > 30% em 15 min → desligamento automático → Admin reativa manualmente | S6 |
| REQ-071 | D01.3 | RN-025 | Rate limit | Rate limit webchat | 30 mensagens / 1 hora (janela deslizante) → bloqueio + contador regressivo mm:ss + campo desabilitado | S2 |
| REQ-072 | D01.3 | RN-025 | UX | Liberação rate limit | Campo retorna normal + contador desaparece + micro-animação pulse 500ms | S2 |
| REQ-073 | D01.3 | RN-026 | Fluxo | Fluxo análise oportunidade individual | Contexto auto-carregado → análise completa → comparação sugerida → simulação → próximo passo | S3 |
| REQ-074 | D01.3 | RN-027 | Fluxo | Fluxo simulação contraproposta | Negociação ativa identificada → valor informado → cálculo → submissão via plataforma | S5 |
| REQ-075 | D01.3 | RN-028 | Regra de negócio | Recusa submissão de proposta | Agente nunca submete proposta; botão "Ir para a oportunidade"; se insistir: "...por segurança, apenas você pode confirmar..." | S3 |
| REQ-076 | D01.3 | RN-029 | SLA | Comportamento em latência acima do SLA | Indicador "digitando" (3 pontos pulsando, streaming); se 2× SLA: "Analista demorando mais..." + botões "Aguardar" / "Tentar novamente" | S3 |
| REQ-077 | D01.3 | RN-029 | SLA | Alerta Admin por latência | Se > 5 min consecutivos → alerta automático Admin (RN-031) | S6 |
| REQ-078 | D01.4 | RN-030 | Admin | Monitoramento de interações | Lista: Cessionário (anonimizado lista, detalhado individual) + data/hora + pergunta + resposta + confiança (%) + dados utilizados + latência (ms) | S6 |
| REQ-079 | D01.4 | RN-030 | UX Admin | Empty state supervisão | Ícone ilustrativo + "Nenhuma interação registrada no período selecionado." + sugestão ajuste filtros | S6 |
| REQ-080 | D01.4 | RN-030 | UX Admin | Filtros supervisão | Por data, Cessionário, nível de confiança; chips removíveis; skeleton inline durante aplicação | S6 |
| REQ-081 | D01.4 | RN-031 | Admin | 6 alertas automáticos | Latência alta (Slack + painel) · Taxa erro elevada (Slack + e-mail) · Desligamento automático (Slack + e-mail + painel) · CSAT degradado (painel + e-mail) · Taxa recusa alta (painel) · Consumo processamento (e-mail) | S6 |
| REQ-082 | D01.4 | RN-031 | Admin | Prioridade alertas simultâneos | Desligamento automático primeiro; latência alta segundo | S6 |
| REQ-083 | D01.4 | RN-032 | Admin | Elegibilidade takeover | Confiança < threshold → sinaliza + habilita botão takeover; Admin pode iniciar a qualquer momento | S6 |
| REQ-084 | D01.4 | RN-033 | Admin | Execução takeover | Registra timestamp + motivo + Admin ID; Cessionário recebe "Um analista da equipe Repasse Seguro assumiu..." + avatar pessoa + "Equipe Repasse Seguro" + separador "Atendimento humano" | S6 |
| REQ-085 | D01.4 | RN-033 | Admin | IA silenciada durante takeover | Nenhuma resposta IA gerada enquanto takeover ativo; campo Cessionário permanece ativo | S6 |
| REQ-086 | D01.4 | RN-033 | Admin | Encerramento takeover | "Você está novamente em atendimento com o Analista de Oportunidades." + separador "Analista de Oportunidades" + avatar agente | S6 |
| REQ-087 | D01.4 | RN-033 | Admin | Concorrência takeover | Primeiro Admin que confirma assume; segundo recebe "Esta conversa já está em atendimento por outro analista" | S6 |
| REQ-088 | D01.4 | RN-034 | Admin | 5 métricas no Dashboard Admin | Volume interações (dia/semana) + Top 10 perguntas + Taxa respostas recusa (%) + CSAT médio (1-5) + Tempo médio resposta (por tipo) | S6 |
| REQ-089 | D01.4 | RN-034 | UX Admin | Card sem dados suficientes | "Dados insuficientes para o período selecionado" (nunca "0" ou "0%") + ícone informação + mantém estrutura visual | S6 |
| REQ-090 | D01.4 | RN-035 | Admin | Configuração threshold confiança | Padrão: 80%; range configurável: 50%-95% | S6 |
| REQ-091 | D01.4 | RN-035 | UX Admin | Erro threshold inválido | Mensagem inline abaixo do campo; valor inválido permanece no campo para correção | S6 |
| REQ-092 | D01.4 | RN-035 | Admin | Toast confirmação + log auditoria | Toast "Nível de supervisão atualizado para [valor]%"; histórico "Alterado de X% para Y% por [Admin] em [data/hora]" | S6 |
| REQ-093 | D01.4 | RN-036 | Disponibilidade | 24/7 com dependência OpenAI | Se API indisponível: Calculadora ativa; FAB: badge amarela status degradado | S3 |
| REQ-094 | D01.4 | RN-037 | Segurança | Gate de isolamento antes de ativar IA | Filtro de escopo + filtro de contexto + pen test 100% bloqueado para dados Cedente | S2 |
| REQ-095 | D01.4 | RN-038 | Segurança | Cobertura cenários de recusa | Instruções cobrem 7 cenários RN-004 + 20 perguntas adversariais testadas antes do lançamento | S9 |
| REQ-096 | D01.4 | RN-039 | Qualidade | Supervisão Admin funcional antes do lançamento | Registro interações + Dashboard métricas + alerta automático confiança + takeover manual | S9 |
| REQ-097 | D01.5 | RN-040 | WhatsApp | Vinculação número WhatsApp | Validação formato DDD+número; máscara (XX) XXXXX-XXXX; validação em tempo real; número já vinculado → mensagem + link suporte | S7 |
| REQ-098 | D01.5 | RN-040 | WhatsApp | OTP SMS enviado | 6 dígitos; prazo 15 minutos; 3 tentativas/hora | S7 |
| REQ-099 | D01.5 | RN-041 | WhatsApp | Validação OTP SMS | Correto → avança; incorreto → limpa campo + tentativas restantes; 3 esgotadas → bloqueio 30 min + contador mm:ss | S7 |
| REQ-100 | D01.5 | RN-041 | WhatsApp | OTP expirado | Mensagem + botão "Reenviar código"; cooldown 60s após reenvio + contador "Reenviar em [ss]s" | S7 |
| REQ-101 | D01.5 | RN-042 | WhatsApp | Segunda etapa vinculação | Mensagem boas-vindas WhatsApp + código confirmação; código correto → Ativo; não respondido em 24h → NaoVinculado | S7 |
| REQ-102 | D01.5 | RN-043 | WhatsApp | Re-verificação periódica | A cada 30 dias; OTP via WhatsApp; 48h para responder; sem resposta → Suspenso; Suspenso 7 dias → Desvinculado | S7 |
| REQ-103 | D01.5 | RN-044 | WhatsApp | Desvinculação | Via plataforma (modal confirmação) + via comando PARAR no WhatsApp (imediato, sem confirmação — LGPD) | S7 |
| REQ-104 | D01.5 | RN-045 | WhatsApp | Bloqueio OTP por 5 falhas consecutivas | 30 minutos; desbloqueio automático | S7 |
| REQ-105 | D01.5 | RN-046 | WhatsApp | Capacidades agente no WhatsApp | Mesmas capacidades do webchat; sem gráficos interativos (tabelas texto); sem botões ação (links plataforma); rate limit: 20 msg/hora | S7 |
| REQ-106 | D01.5 | RN-046 | WhatsApp | Histórico unificado | Webchat + WhatsApp mesmo histórico; rate limits contados separadamente | S7 |
| REQ-107 | D01.5 | RN-047 | Notificações | Alerta nova oportunidade compatível | Disparo: nova oportunidade + perfil compatível + alertas ativos; campos: OPR + Δ estimado + score risco + link; webchat: card + botão "Ver oportunidade"; WhatsApp: opt-in ativo | S7 |
| REQ-108 | D01.5 | RN-048 | Notificações | Alerta prazo Escrow | Disparo: 2 dias úteis antes do vencimento; campos: código negociação + valor + data vencimento + link; não disparar se depósito já feito | S7 |
| REQ-109 | D01.5 | RN-049 | Notificações | Alerta mudança status proposta | Novo status + significado linguagem clara + próximo passo recomendado; webchat + WhatsApp (opt-in) | S7 |
| REQ-110 | D01.5 | RN-050 | Notificações | Gestão opt-in | Meu Perfil > Notificações; 3 tipos configuráveis individualmente; desvinculação WhatsApp cancela todos alertas WhatsApp | S7 |
| REQ-111 | D01.5 | RN-051 | LGPD | Consentimento explícito primeiro uso | Banner antes de qualquer processamento; aceite: registra data+hora+versão + animação slide-up 300ms; recusa: chat bloqueado + campo desabilitado + reaparece próxima abertura | S2 |
| REQ-112 | D01.5 | RN-052 | LGPD | Exclusão dados conta encerrada | Histórico excluído em até 48h; confirmação imediata ao Cessionário; acesso chat bloqueado durante exclusão | S3 |
| REQ-113 | D01.5 | RN-053 | LGPD | Anonimização para métricas | Remove identificadores (nome, CPF, e-mail, WhatsApp); mantém: tipo pergunta, categoria resposta, CSAT médio | S3 |
| REQ-114 | D02 | 2.1 | Stack | Node.js 22.x LTS + NestJS 10.x+ | Framework HTTP único; Express/Fastify/Hono proibidos sem ADR | S1 |
| REQ-115 | D02 | 2.1 | Stack | TypeScript 5.4+ strict:true | Arquivos .js/.jsx proibidos | S1 |
| REQ-116 | D02 | 2.1 | Stack | Prisma 6.x+ | Único ORM aprovado | S1 |
| REQ-117 | D02 | 2.2 | Stack | Supabase PostgreSQL 17+ + pgvector 0.7+ | Único banco relacional aprovado | S1 |
| REQ-118 | D02 | 2.3 | Stack | Redis 7.4+ com TTL definido + estratégia de invalidação | Mudança de prompt invalida cache correspondente | S1 |
| REQ-119 | D02 | 2.3 | Stack | RabbitMQ 4.x+ com DLQ e retry | Toda fila com retry e dead-letter queue | S1 |
| REQ-120 | D02 | 3.1 | Stack IA | GPT-4o via OpenAI SDK 4.x+ | Versão fixada em produção; temperatura 0.1; max_tokens 4096 | S3 |
| REQ-121 | D02 | 3.2 | Stack IA | LangGraph.js 0.2.x+ | Fluxos stateful: análise, comparação, simulação, takeover detection | S3 |
| REQ-122 | D02 | 3.2 | Stack IA | LangChain.js 0.3.x+ | RAG pipeline + tool executor | S3 |
| REQ-123 | D02 | 3.2 | Stack IA | Vercel AI SDK 4.x+ | Streaming SSE; resposta inicial ≤ 5s | S3 |
| REQ-124 | D02 | 3.3 | IA | Function calling via OpenAI | 5 tools: calculate_delta, calculate_roi, calculate_portfolio, get_market_data, get_opportunity_data | S3 |
| REQ-125 | D02 | 3.4 | IA | Prompts como código | 4 arquivos: repasse-ai-system.prompt.ts, repasse-ai-user.prompt.ts, repasse-ai-tools.ts, repasse-ai-guardrails.ts | S3 |
| REQ-126 | D02 | 3.4 | IA | Structured Outputs obrigatório | Para score de risco (RN-012), cálculo de comissão (RN-013), comparação (RN-015); `response_format: { type: "json_schema" }`; parsing manual JSON proibido | S3 |
| REQ-127 | D02 | 3.4 | IA | Guardrails no prompt | 7 cenários de recusa + exemplos adversariais + delimitadores `<user_input>...</user_input>` | S3 |
| REQ-128 | D02 | 3.4 | IA | Few-shot examples | Mínimo 3 exemplos por tipo de consulta (análise, comparação, simulação) | S3 |
| REQ-129 | D02 | 3.5 | Cache | Cache exato Redis | TTL 24h; chave: modelo+versão prompt+hash input; cálculos determinísticos temperature 0 | S3 |
| REQ-130 | D02 | 3.5 | Cache | Cache semântico Redis | TTL 1h; threshold 0.92; consultas suporte operacional | S3 |
| REQ-131 | D02 | 4.1 | RAG | pgvector + OpenAI Embeddings | text-embedding-3-small (1536 dims); chunking 512-1024 tokens, overlap 10-20%; similarity threshold 0.78 | S3 |
| REQ-132 | D02 | 4.1 | RAG | Índice HNSW (padrão) / IVFFlat (>1M vetores) | Via raw SQL (Prisma não suporta vector nativamente) | S3 |
| REQ-133 | D02 | 4.2 | RAG | Pipeline ingestão | RabbitMQ (assíncrono); chunks com metadata (source, page, section, timestamp, opportunity_id); idempotente | S3 |
| REQ-134 | D02 | 4.3 | RAG | 4 fontes de dados RAG | Regras plataforma (sem filtro) + dados empreendimentos (sem filtro) + oportunidades marketplace (filtrado cessionario_id) + histórico conversas (filtrado cessionario_id, 90 dias) | S3 |
| REQ-135 | D02 | 5.4 | Segurança | NestJS Guards | Guard de Cessionário + Guard de Admin; toda request com cessionario_id validado no JWT | S2 |
| REQ-136 | D02 | 6.1 | Infra | Monorepo Turborepo + pnpm 9.x+ | apps/ai/ dentro do monorepo shiftlabs-repasse; pnpm obrigatório (npm/yarn proibidos) | S1 |
| REQ-137 | D02 | 6.2 | Arquitetura | Estrutura de pastas completa | modules/{ai,calculator,chat,notification,supervision,whatsapp}, common/{decorators,filters,guards,interceptors,pipes}, config, jobs | S1 |
| REQ-138 | D02 | 6.3 | Padrões | ESLint + Prettier + Husky + lint-staged | Pre-commit hook; Conventional Commits | S1 |
| REQ-139 | D12 | 3.1 | Banco | `chat_conversations` | 8 colunas; enums channel_enum + conversation_status_enum; 3 índices; 2 constraints; 1 conversa ativa por canal por Cessionário | S1 |
| REQ-140 | D12 | 3.2 | Banco | `chat_messages` | 7 colunas; enums message_role_enum + channel_enum; 1 FK ON DELETE CASCADE; 2 índices; 2 constraints; imutável após criação | S1 |
| REQ-141 | D12 | 3.3 | Banco | `ai_interactions` | 15 colunas; enum query_type_enum + interaction_status_enum; 2 FKs; 5 índices; 3 constraints | S1 |
| REQ-142 | D12 | 3.4 | Banco | `llm_cache_entries` | 10 colunas; unique key composta; sem soft delete | S1 |
| REQ-143 | D12 | 3.5 | Banco | `document_embeddings` | 7 colunas; vector(1536); índice IVFFlat lists=100; sem FK usuários | S1 |
| REQ-144 | D12 | 3.6 | Banco | `whatsapp_bindings` | 11 colunas; enum whatsapp_binding_status_enum; unique phone_number; soft delete | S1 |
| REQ-145 | D12 | 3.7 | Banco | `otp_attempts` | 6 colunas; FK para whatsapp_bindings; append-only; sem updated_at | S1 |
| REQ-146 | D12 | 3.8 | Banco | `notification_preferences` | 6 colunas; unique cessionario_id | S1 |
| REQ-147 | D12 | 3.9 | Banco | `notification_events` | 11 colunas; enums event_type + channel + status | S1 |
| REQ-148 | D12 | 3.10 | Banco | `ai_takeovers` | 8 colunas; FK para ai_interactions; campos: reason + reason_detail + status | S1 |
| REQ-149 | D12 | 3.11 | Banco | `agent_configurations` | 7 colunas; key única; campos: key + value + value_type + description + is_active + updated_by | S1 |
| REQ-150 | D12 | 3.12 | Banco | `agent_status_logs` | 7 colunas; append-only; campos: event_type + status_before + status_after + trigger_reason + error_rate_pct + triggered_by | S1 |
| REQ-151 | D12 | 3.13 | Banco | `lgpd_consents` | 8 colunas; append-only; campos: consent_type + version + granted + ip_address + user_agent | S1 |
| REQ-152 | D12 | 2 | Banco | Ordem produção | migrate deploy → policies.sql (RLS) → indexes.sql → seed (9 configurações) | S1 |
| REQ-153 | D13 | — | Schema Prisma | Middleware soft delete global | delete() nunca executado em tabelas PII; soft delete via middleware | S1 |
| REQ-154 | D13 | — | Schema Prisma | Middleware tenant (RLS) | SET LOCAL para isolamento por cessionario_id | S1 |
| REQ-155 | D13 | — | Schema Prisma | Seed de 9 configurações obrigatórias do agente | agent_configurations seed antes do lançamento | S1 |
| REQ-156 | D14 | — | Arquitetura | 8 containers | NestJS (Railway) + Supabase PostgreSQL + Redis (Upstash) + RabbitMQ (CloudAMQP) + OpenAI + EvolutionAPI (Fase 2) + Langfuse + Supabase Auth | S1 |
| REQ-157 | D14 | — | Arquitetura | ADR-001 a ADR-005 documentados | Backend puro sem frontend (001), LangGraph.js stateful (002), pgvector inline (003), Railway deploy (004), SELECT FOR UPDATE takeover (005) | S1 |
| REQ-158 | D16 | 7.1 | API | `GET /health` | Verifica serviço + dependências (database, redis, rabbitmq, llm); 200 healthy / 503 degraded | S1 |
| REQ-159 | D16 | 7.2 | API | `POST /chat/conversations` | 201 Created; cessionario_id do JWT; max 1 ativa por canal | S3 |
| REQ-160 | D16 | 7.2 | API | `GET /chat/conversations` | Paginação; filtrado por cessionario_id | S3 |
| REQ-161 | D16 | 7.2 | API | `GET /chat/conversations/{id}/messages` | Paginação; lazy-load 20 mensagens; order by created_at desc | S3 |
| REQ-162 | D16 | 7.2 | API | `POST /chat/conversations/{id}/messages` | SSE streaming; rate limit 30/h webchat; timeout stream 120s; início stream 10s | S3 |
| REQ-163 | D16 | 7.2 | API | `DELETE /chat/history` | 204; soft delete todas as conversas; job async 48h para hard delete | S3 |
| REQ-164 | D16 | 7.3 | API | Endpoints calculadora (3) | `POST /calculator/commission`, `POST /calculator/escrow`, `POST /calculator/roi`; rate limit 10 req/min | S4 |
| REQ-165 | D16 | 7.4 | API | Endpoints notificações (3) | `GET /notifications/preferences`, `PATCH /notifications/preferences`, `GET /notifications/events`; rate limit 30 req/min | S7 |
| REQ-166 | D16 | 7.5 | API | Endpoints supervisão Admin (7) | `GET/GET/{id} /supervision/interactions`, `POST /supervision/takeovers`, `GET/PATCH /supervision/takeovers/{id}/release`, `GET /supervision/metrics`, `GET /supervision/alerts`, `GET+PATCH /supervision/agent-config`; rate limit 60 req/min | S6 |
| REQ-167 | D16 | 7.6 | API | Endpoints WhatsApp (3+webhook) | `POST /whatsapp/bindings`, `GET /whatsapp/bindings`, `DELETE /whatsapp/bindings/{id}`, `POST /whatsapp/webhook`; webhook com HMAC validation | S7 |
| REQ-168 | D17 | 2.1 | Integração | OpenAI (P0) | Retry backoff exponencial 3 tentativas, base 1s; kill switch PostHog | S1/S3 |
| REQ-169 | D17 | 2.2 | Integração | Supabase (P0) | RLS habilitado; SLA 99.9% | S1 |
| REQ-170 | D17 | 2.3 | Integração | Upstash Redis (P0) | SLA 99.99%; TTL em todo uso | S1 |
| REQ-171 | D17 | 2.4 | Integração | EvolutionAPI (P1, Fase 2) | Webhook HMAC; mock via FEATURE_WHATSAPP_MOCK=true | S7 |
| REQ-172 | D17 | 2.5 | Integração | CloudAMQP RabbitMQ (P1) | SLA 99.5%; DLQ em todas as filas | S1 |
| REQ-173 | D17 | 2.6 | Integração | Langfuse (P1) | Tracing obrigatório; toda feature IA com trace antes de ir a produção | S1 |
| REQ-174 | D19 | 2.2 | Agente IA | Ciclo de execução LangGraph | START → GUARDRAIL → CACHE_CHECK → INTENT → RAG → TOOL_SELECT → TOOL_EXEC → GENERATE → STREAM → PERSIST → LANGFUSE | S3 |
| REQ-175 | D19 | 2.3 | Agente IA | AgentState schema | 9 campos: messages, cessionario_id, conversation_id, intent, rag_context, tool_results, confidence, cached, error, stream_done | S3 |
| REQ-176 | D19 | 5.1 | Agente IA | Memória curta (sessão) | Redis `chat:history:{conv_id}`; TTL 30 min (renovado a cada mensagem); 20 mensagens | S3 |
| REQ-177 | D19 | 5.1 | Agente IA | Memória longa (histórico) | Supabase; 90 dias; paginada | S3 |
| REQ-178 | D22 | — | Setup | 18 variáveis de ambiente | 9 sensíveis + 9 não sensíveis; .env nunca commitado; Railway Secrets em produção | S1 |
| REQ-179 | D24 | — | CI/CD | Pipeline GitHub Actions | lint + typecheck + testes unitários + integração + build + security audit por PR | S8 |
| REQ-180 | D24 | — | CI/CD | Ambientes | dev (local Docker) + staging (Railway branch staging) + produção (Railway branch main) | S8 |
| REQ-181 | D24 | — | CI/CD | Deploy automático | PR → 2 aprovações → merge main → Railway deploy → health check 5× em 60s → tag Git | S8 |
| REQ-182 | D24 | — | CI/CD | Rollback | Railway "Redeploy previous deployment" < 3 min | S8 |
| REQ-183 | D27 | — | Qualidade | Pirâmide de testes | 70% unitários (Jest + mocks) / 25% integração (Jest + Supertest + Docker) / 5% E2E (Jest + Supertest + full stack) | S8 |
| REQ-184 | D29 | — | Go-Live | Launch checklist T-7/T-3/T-1 | T-7: integrações + secrets Railway produção; T-3: QA + E2E; T-1: freeze + smoke tests + war room Slack #repasse-ai-launch | S9 |
| REQ-185 | D29 | — | Go-Live | Smoke tests + go/no-go | `http_error_rate < 2%` + `GET /health` 200 + circuit breaker CLOSED + sem CRITICAL Sentry; rollback se any threshold cruzado em T+15min | S9 |

---

## Verificação de Cobertura

```
Registro Mestre: 185 requisitos
Atribuídos: 185 (100%)
Sem sprint: 0 (deve ser 0)
```

**Distribuição por Sprint:**
- S1 — Fundação: REQ-007, REQ-114 a REQ-158, REQ-168 a REQ-173, REQ-178 → ~38 itens
- S2 — Auth: REQ-004 a REQ-006, REQ-022 a REQ-023, REQ-028, REQ-071 a REQ-072, REQ-094, REQ-111, REQ-135 → ~12 itens
- S3 — Agente Core: REQ-008 a REQ-021, REQ-024 a REQ-027, REQ-029 a REQ-032, REQ-036, REQ-038 a REQ-039, REQ-062 a REQ-067, REQ-073, REQ-075 a REQ-076, REQ-093, REQ-112 a REQ-113, REQ-120 a REQ-134, REQ-159 a REQ-163, REQ-174 a REQ-177 → ~55 itens (quebrar em S3a + S3b)
- S4 — Calculadora de Comissão: REQ-002, REQ-040 a REQ-045, REQ-068 a REQ-069, REQ-164 → ~10 itens
- S5 — Comparação e Simulação: REQ-046 a REQ-061, REQ-074, REQ-037 → ~18 itens
- S6 — Admin e Supervisão: REQ-070, REQ-077 a REQ-092 → ~17 itens
- S7 — WhatsApp e Notificações: REQ-003, REQ-097 a REQ-110, REQ-165, REQ-167, REQ-171 → ~18 itens
- S8 — Qualidade: REQ-179 a REQ-183 → ~5 itens
- S9 — Go-Live: REQ-095 a REQ-096, REQ-184 a REQ-185 → ~4 itens
- S10 — Sprint Final (Auditoria): todos REQs (verificação cruzada)
- CROSS: REQ-001 (Δ/Delta — definição cross-cutting)
