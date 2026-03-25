# Stacks Tecnologicas -- Modulo Cessionario

## Padrao Tecnico Normativo

| Campo | Valor |
|---|---|
| Destinatario | Tech Lead / CTO |
| Escopo | Stack normativa do modulo Cessionario -- Repasse Seguro |
| Modulo | Cessionario |
| Versao | v1.1 |
| Responsavel | Claude Code Desktop |
| Data da versao | 2026-03-22 (America/Fortaleza) |

> **Este documento e normativo.** Toda tecnologia listada e obrigatoria. Desvios exigem ADR aprovado pelo responsavel tecnico antes da implementacao. A fonte primaria de verdade e o ShiftLabs Stacks v7.0. Este documento aplica o padrao central ao contexto especifico do modulo Cessionario.

---

> **TL;DR**
>
> - **Backend:** Node.js 22 + NestJS 10 + TypeScript strict + Prisma 6 + Supabase (PostgreSQL 17). Redis para cache. RabbitMQ para filas. Modulo de IA dedicado com OpenAI SDK + Vercel AI SDK + Langfuse.
> - **Frontend Web:** React 19 + Vite 7 + TypeScript strict + Tailwind CSS v4 + shadcn/ui + TanStack Router + TanStack Query + Zustand + Framer Motion. Dashboard 100% logado -- Next.js nao se aplica.
> - **Mobile:** React Native 0.76+ + Expo SDK 52 + expo-router 4 + TypeScript strict + Reanimated + TanStack Query + Zustand.
> - **Comunicacao:** API REST (JSON) + fetch nativo + JWT. SSE para streaming do Analista de Oportunidades. Supabase Realtime para oportunidades em tempo real, status de negociacao e notificacoes in-app.
> - **Seguranca:** KYC com OCR/liveness (RN-010/RN-065), RBAC com isolamento por Cessionario (RN-013/RN-068), anonimizacao do Cedente no backend (RN-014/RN-067), re-autenticacao para acoes criticas (RN-005/RN-070), CSRF obrigatorio, RLS no Supabase.
> - **Integracao:** ZapSign (assinatura digital), idwall (KYC -- OCR + liveness), Resend (e-mail transacional), Expo Notifications (push), Celcoin (conta Escrow -- confirmacao manual no MVP, integracao automatica em v2).
> - **Infra:** Monorepo Turborepo + pnpm. Railway (backend). GitHub Actions para CI/CD. Sentry para error tracking. PostHog Cloud para analytics. Langfuse Cloud para observabilidade de IA. Upstash (Redis). CloudAMQP (RabbitMQ). 3 projetos Supabase (dev/staging/prod).

---

## 1. Principios de Governanca

**PG-01. ShiftLabs Stacks v7.0 e a fonte primaria de verdade.** Toda decisao de stack neste documento herda do padrao central. Sobrescricoes ocorrem apenas com justificativa tecnica vinculada a uma regra de negocio do Cessionario.

**PG-02. Desenvolvimento 100% por IA.** Todo codigo e gerado por agentes de IA (Claude Code e GPT Codex). Boilerplates, convencoes e testes automatizados sao os contratos de qualidade. Decisoes de stack priorizam tecnologias bem representadas no treinamento dos modelos.

**PG-03. Desvios via ADR.** Qualquer tecnologia nao listada neste documento exige Architecture Decision Record aprovado antes da implementacao. O ADR deve conter: contexto, decisao, consequencias e alternativas descartadas.

**PG-04. Rastreabilidade RN-Stack.** Toda tecnologia que existe neste documento por exigencia de uma regra de negocio deve referenciar a RN correspondente. Tecnologias sem vinculo a RN herdam do padrao central sem justificativa adicional.

**PG-05. Pendencias bloqueiam implementacao.** Itens marcados com `[DEFINICAO PENDENTE]` nao devem ser implementados ate resolucao. Itens marcados com `[DECISAO AUTONOMA]` foram decididos com base no contexto disponivel e podem ser revisados. *(v1.1: todas as pendencias foram resolvidas. Nenhum item `[DEFINICAO PENDENTE]` remanescente.)*

---

## 2. Backend

### 2.1 Stack Obrigatoria

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| Node.js | 22.x+ (LTS) | ✅ Obrigatorio | Runtime padrao ShiftLabs. LTS garante estabilidade. | Unico runtime aprovado. |
| NestJS | 10.x+ | ✅ Obrigatorio | Arquitetura modular, decorators, Swagger embutido, guards/pipes nativos. Output de IA consistente. | Framework HTTP unico aprovado. Express, Fastify, Hono proibidos sem ADR. |
| TypeScript | 5.4+ | ✅ Obrigatorio | `strict: true` obrigatorio. Tipagem forte maximiza qualidade do output de IA. | Todo arquivo backend deve ser `.ts`. `.js` proibido. |
| Prisma | 6.x+ | ✅ Obrigatorio | Tipagem auto-gerada, migrations declarativas, integracao nativa com Supabase. | Unico ORM aprovado. TypeORM, Sequelize, Drizzle proibidos. |
| PostgreSQL (via Supabase) | 17+ | ✅ Obrigatorio | Banco relacional unico aprovado. Gerenciado pelo Supabase. | MySQL, MariaDB, SQLite em producao proibidos. NoSQL proibido sem ADR. |
| Redis | 7.4+ | ✅ Obrigatorio | Cache de sessoes, rate limiting de propostas (RN-020), cache de recomendacoes da IA (RN-046). | Todo uso de Redis deve ter TTL definido e estrategia de invalidacao documentada. Docker em local, Upstash serverless em producao. |
| RabbitMQ | 4.x+ | ✅ Obrigatorio | Processamento assincrono: ingestao de documentos KYC (RN-010/RN-065), disparo de notificacoes (RN-061/RN-066), pipeline de RAG para Analista de Oportunidades (RN-048). | Toda fila deve ter retry + dead-letter queue. Docker em local, CloudAMQP gerenciado em producao. |
| Pino | 9.x+ | ✅ Obrigatorio | Logs estruturados com request ID, timestamp, nivel e contexto. | Obrigatorio em todo servico. Nunca usar `console.log` em producao. |
| Helmet | 8.x+ | ✅ Obrigatorio | Headers de seguranca automaticos. Critico para protecao de dados financeiros do Cessionario (RN-013). | Middleware adicionado desde o primeiro commit. |
| class-validator + class-transformer | latest | ✅ Obrigatorio | Validacao de DTOs. Critico para validacao de propostas (RN-021), contrapropostas (RN-027) e dados de KYC (RN-010). | Toda entrada de dados validada via pipes do NestJS. |
| @nestjs/swagger | latest | ✅ Obrigatorio | Documentacao auto-gerada de API. | Todo endpoint documentado. Swagger e a fonte de verdade da API. |
| @nestjs/throttler | latest | ✅ Obrigatorio | Rate limiting. Critico para limites de propostas (RN-020: 3 simultaneas, 10/24h) e protecao contra abuso de KYC (RN-006). | Endpoints de autenticacao com rate limiting mais restritivo. |
| Sentry (@sentry/nestjs) | 9.x+ | ✅ Obrigatorio | Error tracking com stack traces e breadcrumbs. | Deploy sem Sentry configurado e proibido. |

### 2.2 Banco de Dados -- Supabase

| Recurso Supabase | Status | Justificativa | Regra de Uso |
|---|---|---|---|
| PostgreSQL gerenciado | ✅ Obrigatorio | Banco unico do modulo. | Prisma conecta via connection string padrao. |
| Supabase Auth | ✅ Obrigatorio | Autenticacao do Cessionario (RN-001 a RN-005): cadastro com e-mail/senha, login com Google OAuth (RN-003), controle de sessao (RN-004). | Complementa JWT para gerenciamento de sessao. |
| Supabase Storage | ✅ Obrigatorio | Upload de documentos KYC (RN-010: identidade, comprovante, selfie), comprovantes de deposito Escrow (RN-028), documentos de formalizacao (RN-033). | Upload via signed URL gerada pelo backend. Nunca upload direto via multipart no NestJS. |
| Supabase Realtime | ✅ Obrigatorio | Oportunidades em tempo real no marketplace (RN-017), atualizacao de status de negociacao (RN-024 a RN-030), notificacoes in-app no Dashboard (RN-045/RN-061), atualizacao do Dashboard em ate 60 segundos (SLA Parte 01.3). | Subscriptions filtradas por Cessionario logado para garantir isolamento (RN-068). |
| Supabase pgvector | ✅ Obrigatorio | RAG para o Analista de Oportunidades (RN-048): embeddings de dados de oportunidades para busca semantica, recomendacoes personalizadas (RN-046) e analise comparativa. | Extensao `pgvector` habilitada. Indice `hnsw` como padrao. Embeddings no mesmo PostgreSQL dos dados. |
| Row Level Security (RLS) | ✅ Obrigatorio | Isolamento de dados por Cessionario (RN-013/RN-068). Anonimizacao do Cedente (RN-014/RN-067). | RLS habilitado em todas as tabelas com dados de usuarios. `service_role` key nunca exposta no frontend. |

### 2.3 Modulo de IA -- Analista de Oportunidades

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| OpenAI SDK (openai) | 4.x+ | ✅ Obrigatorio | Cliente padrao para GPT-4. Chamadas ao Analista de Oportunidades (RN-047 a RN-051). | Toda chamada LLM via backend. Frontend nunca chama OpenAI diretamente. |
| GPT-4 | Versao fixada em producao | ✅ Obrigatorio | Modelo padrao para analise de risco, comparativos e simulacoes de retorno (RN-048). | Fixar versao em producao (ex: `gpt-4-turbo-2024-04-09`). GPT-4o-mini aprovado para tarefas simples (classificacao, sumarizacao). |
| Vercel AI SDK | 4.x+ | ✅ Obrigatorio | Streaming de respostas do Analista de Oportunidades para o frontend via SSE (RN-050: SLA 5s individual, 10s comparativo). | Gerencia streaming no backend NestJS com endpoint SSE. |
| LangChain.js | 0.3.x+ | ✅ Obrigatorio | Pipeline RAG para busca semantica em oportunidades, roteamento de prompts, memory management para historico de chat (RN-051). | Usado no backend para pipelines complexos. |
| Langfuse JS SDK | 3.x+ | ✅ Obrigatorio | Observabilidade de chamadas LLM: latencia, custo, tokens, qualidade. | Monitorar p50/p95/p99 de latencia. Alerta se custo diario > 2x media 7 dias. Alerta se taxa de erro > 5% em 1h. [DECISAO AUTONOMA -- US$ 50/dia alerta, US$ 200/dia hard limit. Baseado em volume estimado de consultas do Cessionario. Descartado: sem limite (risco financeiro), US$ 10/dia (muito restritivo para agente multi-consulta).] Threshold de custo diario OpenAI configurado no Langfuse: US$ 50/dia como alerta, US$ 200/dia como hard limit. |
| OpenAI Embeddings | text-embedding-3-small | ✅ Obrigatorio | Embeddings para RAG do Analista de Oportunidades. | `text-embedding-3-large` aprovado via ADR para casos que exijam maior precisao. |

**Restricoes do Analista de Oportunidades vinculadas a stack:**

- O agente opera exclusivamente com dados disponiveis ao perfil Cessionario logado (RN-049). Toda query ao banco filtrada por `cessionario_id`.
- Nenhuma consulta do agente acessa campos do Cedente ou de outros Cessionarios (RN-049/RN-068).
- Sanitizacao de input do usuario obrigatoria antes de inserir em prompts (protecao contra prompt injection).
- Rate limiting por usuario em endpoints de LLM: 20-60 chamadas/minuto (definido no ADR-001).
- Structured Outputs (`response_format: { type: "json_schema" }`) obrigatorio para tarefas deterministicas (score de risco, comparativos estruturados).
- Prompts como codigo em `src/modules/ai/prompts/analista-oportunidades/`. Nunca hardcoded inline.
- Filtro de output: linguagem de urgencia, FOMO ou apelo emocional e bloqueada antes de exibir ao Cessionario (RN-047).

### 2.4 Convencoes de Banco de Dados (Prisma Schema)

Herdadas integralmente do ShiftLabs Stacks v7.0 secao 1.5:

- **Primary keys:** UUID v4. Nunca auto-increment.
- **Auditoria:** `created_at`, `updated_at` obrigatorios em toda tabela. `created_by`, `updated_by` em tabelas com rastreio de autor.
- **Soft delete:** Padrao para tabelas de dominio (propostas, negociacoes, documentos, operacoes financeiras). `deleted_at DateTime? @db.Timestamptz`.
- **Nomenclatura:** Tabelas em `snake_case` plural. Colunas em `snake_case`. Enums em `PascalCase` com valores `UPPER_SNAKE_CASE`.
- **Indices:** Toda foreign key com indice. Colunas em `WHERE` e `ORDER BY` frequente com indice.
- **Timestamps:** Sempre `@db.Timestamptz`. Nunca `@db.Timestamp` sem timezone.

### 2.5 Integracao com Servicos Externos

| Servico | Funcao | RNs Vinculadas | Padrao de Integracao |
|---|---|---|---|
| ZapSign | Assinatura digital de documentos de formalizacao | RN-032 a RN-034, RN-063 | Webhook para notificacao de assinatura. Retry automatico (3 tentativas em 30 min). Falha oculta do Cessionario. |
| idwall (KYC -- OCR + Liveness) | Validacao automatica de documentos de identidade e prova de vida | RN-010, RN-011, RN-065 | [DECISAO AUTONOMA -- idwall. Lider em KYC digital no Brasil, API REST, LGPD compliant. Descartado: Nuveo (menos madura), manual review (nao escala).] OCR de documentos brasileiros (RG, CNH, CNPJ), face match + liveness detection. Job assincrono via RabbitMQ. SLA 5 min. Fallback para revisao manual se servico indisponivel. |
| Resend (E-mail Transacional) | Envio de notificacoes por e-mail (canal minimo obrigatorio) | RN-008, RN-061, RN-062, RN-066, RN-069 | [DECISAO AUTONOMA -- Resend. SDK TypeScript nativo. Descartado: SendGrid (SDK pesado), Amazon SES (over-engineering).] Retry em ate 5 min. Log de entrega. E-mail nunca desabilitado (RN-069). |
| Servico de Push Notifications | Envio de notificacoes push para web e mobile | RN-061, RN-062, RN-066 | Expo Notifications no mobile. Web Push API no frontend. |
| Conta Escrow -- Celcoin | Depositos e liberacoes de Escrow | RN-028, RN-030, RN-064 | [DECISAO AUTONOMA -- Celcoin. Regulacao BCB, escrow digital nativo. Descartado: Transfeera (foco em pagamentos).] Fintech regulada BCB, APIs REST, escrow digital nativo, split de pagamentos, conciliacao automatica. MVP opera com confirmacao manual pelo Admin conforme ADR-003. Integracao automatica Celcoin em v2. |

---

## 3. Frontend Web

### 3.1 Stack Obrigatoria

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| React | 19.x+ | ✅ Obrigatorio | Biblioteca de UI padrao ShiftLabs. | Componentes funcionais exclusivamente. Class components proibidos. |
| Vite | 7.x+ | ✅ Obrigatorio | Build tool para SPA. Dashboard do Cessionario e 100% logado -- nao ha pagina publica/SEO. | Next.js nao se aplica a este modulo. [DECISAO AUTONOMA] Justificativa: RN-001 a RN-005 confirmam que todo acesso ao modulo Cessionario exige autenticacao. Nao ha face publica. React + Vite e o caminho correto conforme ShiftLabs Stacks v7.0 secao 2.1. |
| TypeScript | 5.4+ | ✅ Obrigatorio | `strict: true` obrigatorio. | Todo arquivo frontend deve ser `.tsx` ou `.ts`. |
| TanStack Router | 1.x+ | ✅ Obrigatorio | Roteamento type-safe com integracao nativa com TanStack Query. | Padrao para apps React + Vite. `react-router-dom` proibido sem ADR. |
| TanStack Query | 5.x+ | ✅ Obrigatorio | Fetching, caching e sincronizacao de dados do servidor. Critico para cache de oportunidades (RN-017), propostas (RN-021), negociacoes (RN-024). | Nunca usar `fetch` direto em `useEffect`. Retry 3x para erros 5xx. Sem retry para 4xx. |
| Zustand | 5.x+ | ✅ Obrigatorio | Estado global: sessao do usuario, preferencias de notificacao (RN-008), estado do chat com IA (RN-051), timer de inatividade (RN-004). | Redux, MobX, Context API como state manager global proibidos. |
| Tailwind CSS | 4.x+ | ✅ Obrigatorio | Framework de estilizacao. Design tokens via `@theme` no `global.css`. | CSS customizado permitido para animacoes avancadas. UnoCSS, Windi CSS proibidos. |
| shadcn/ui | latest | ✅ Obrigatorio | Sistema de componentes baseado em Radix Primitives com acessibilidade completa. | Unico sistema de componentes aprovado. MUI, Chakra, AntD, Bootstrap proibidos. Codigo copiado para o repositorio (ownership total). |
| Radix Primitives | latest | ✅ Obrigatorio | Base de acessibilidade dos componentes shadcn/ui. Keyboard nav, ARIA, focus management. | Consumido via shadcn/ui. |
| Framer Motion (Motion) | 12.x+ | ✅ Obrigatorio | Transicoes de tela (AnimatePresence), micro-interacoes, skeleton screens. Critico para: loading do marketplace (RN-017: skeleton cards), indicador de digitacao da IA (RN-050), transicoes de status de proposta/negociacao. | Todo elemento interativo deve ter feedback visual. Toda mudanca de rota deve usar AnimatePresence. Spinners genericos proibidos -- usar skeleton screens. |
| Sentry (@sentry/react) | 9.x+ | ✅ Obrigatorio | Error tracking. Error Boundary global obrigatorio. | Source maps enviados ao Sentry em producao. |
| PostHog (posthog-js) | 1.x+ | ✅ Obrigatorio | Analytics + session replay + feature flags. Eventos obrigatorios: `screen_viewed`, `signup_completed`, `proposal_created`, `escrow_deposited`, `document_signed`. | Wrapper centralizado em `src/services/analytics.ts`. Nunca chamar SDK diretamente nos componentes. |
| date-fns | 4.x+ | ✅ Obrigatorio | Manipulacao de datas (tree-shakeable, imutavel). | Moment.js, Day.js, Luxon proibidos. |
| date-fns-tz | 3.x+ | ✅ Obrigatorio | Conversao de timezone para exibicao. Prazos de Escrow (RN-028), expiracoes de proposta (RN-023), timestamps de notificacoes (RN-045). | Backend envia UTC (ISO 8601). Frontend converte para timezone do usuario apenas na apresentacao. |
| fetch nativo | -- | ✅ Obrigatorio | Cliente HTTP. | Axios proibido sem ADR. Interceptor de auth obrigatorio em `services/api.ts`. |

### 3.2 Funcionalidades Real-Time do Frontend

| Funcionalidade | Tecnologia | RN Vinculada |
|---|---|---|
| Novas oportunidades no marketplace | Supabase Realtime (subscription na tabela `opportunities`) | RN-017 |
| Atualizacao de status de proposta | Supabase Realtime | RN-021, RN-022, RN-023 |
| Mensagens na negociacao (chat com Admin) | Supabase Realtime | RN-026 |
| Atualizacao de status de negociacao/Escrow | Supabase Realtime | RN-024 a RN-030 |
| Notificacoes in-app (badge + widget Dashboard) | Supabase Realtime | RN-045, RN-061 |
| Streaming de respostas do Analista de Oportunidades | SSE (via Vercel AI SDK) | RN-050 |
| Atualizacao do Dashboard apos evento | Supabase Realtime + TanStack Query invalidation | RN-042 a RN-046 (SLA 60s) |

### 3.3 Acessibilidade (a11y)

Herdada integralmente do ShiftLabs Stacks v7.0 secao 2.5:

- HTML semantico obrigatorio. `<button>`, `<nav>`, `<main>` em vez de `<div>`.
- Navegacao por teclado em todo elemento interativo (Tab, Enter/Space).
- Contraste minimo 4.5:1 (AA). Textos grandes: 3:1.
- Foco visivel. Nunca remover `outline` sem substituto.
- Tamanho minimo de toque: 44x44px (critico para CTA "Fazer Proposta" em mobile -- RN-019).
- Textos alternativos em imagens informativas. `alt=""` em decorativas.

**Requisitos adicionais vinculados ao Cessionario:**

- Empty states com ilustracoes acessiveis: `alt` descritivo obrigatorio (RN-052).
- Stepper de KYC navegavel por teclado (RN-010).
- Modal de re-autenticacao com focus trap (RN-005).
- Modal de confirmacao de cancelamento com focus no botao seguro ("Manter proposta") (RN-022).

### 3.4 Performance

Herdada do ShiftLabs Stacks v7.0 secao 2.7:

- **Bundle size:** JavaScript total (gzipped) abaixo de 300KB para SPA React+Vite.
- **Code splitting:** `React.lazy()` + `Suspense` para rotas e componentes pesados (modais, graficos de valorizacao -- RN-019, chat da IA -- RN-051).
- **Tree shaking:** Importar apenas o necessario. Nunca `import *`.
- **Analise de bundle:** `rollup-plugin-visualizer` para identificar dependencias pesadas.
- **Imagens:** WebP como padrao. SVG para icones. `loading="lazy"` e `decoding="async"` obrigatorios.

---

## 4. Mobile

### 4.1 Stack Obrigatoria

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| React Native | 0.76+ | ✅ Obrigatorio | Framework mobile unico aprovado. | Flutter, Kotlin Multiplatform, nativo puro proibidos sem ADR. |
| Expo SDK | 52+ | ✅ Obrigatorio | Managed workflow. OTA updates via EAS Update. Build via EAS Build. | Eject para bare workflow apenas com justificativa documentada. |
| expo-router | 4.x+ | ✅ Obrigatorio | File-based routing com deep linking nativo. | React Navigation proibido. |
| TypeScript | 5.4+ | ✅ Obrigatorio | `strict: true`. | Todo arquivo mobile deve ser `.tsx` ou `.ts`. |
| React Native Reanimated | 3.x+ | ✅ Obrigatorio | Animacoes na UI thread. Skeleton screens, transicoes de tela, micro-interacoes. | Nunca animar com `setState` ou `Animated` API legacy. |
| React Native Gesture Handler | 2.x+ | ✅ Obrigatorio | Interacoes tateis: swipe em cards de oportunidade, drag, long press. | Integrado com Reanimated. |
| TanStack Query | 5.x+ | ✅ Obrigatorio | Mesmo padrao do web. Cache e sincronizacao de dados. | Mesmos hooks compartilhados via `packages/shared-services/`. |
| Zustand | 5.x+ | ✅ Obrigatorio | Estado global. Persistencia local via AsyncStorage (dados nao sensiveis). | Middleware `persist` com AsyncStorage para estado global persistente. |
| expo-secure-store | latest | ✅ Obrigatorio | Armazenamento seguro de refresh token e dados sensiveis. | Tokens de autenticacao nunca em AsyncStorage. |
| expo-image | latest | ✅ Obrigatorio | Carregamento otimizado com cache automatico e placeholder blur. | `<Image>` nativo proibido para imagens remotas em producao. |
| Expo Notifications | latest | ✅ Obrigatorio | Push notifications para alertas de prazo de Escrow (NOT-CES-05/06), propostas (NOT-CES-03/04), documentos (NOT-CES-08). | Permissao solicitada de forma contextual, nunca no primeiro acesso. OneSignal, Pusher proibidos. |
| Sentry (@sentry/react-native) | 6.x+ | ✅ Obrigatorio | Error tracking mobile. | Error Boundary no componente raiz. |
| PostHog (posthog-react-native) | 3.x+ | ✅ Obrigatorio | Analytics mobile. Mesmos eventos do web com `platform: ios|android`. | Wrapper centralizado em `src/services/analytics.ts`. |
| date-fns | 4.x+ | ✅ Obrigatorio | Manipulacao de datas. | Mesmo padrao do web. |
| date-fns-tz | 3.x+ | ✅ Obrigatorio | Conversao de timezone. | Mesmo padrao do web. |

### 4.2 Camera e KYC Mobile

- A selfie de prova de vida (RN-010, passo 3) exige acesso a camera do dispositivo. Usar `expo-camera` para captura com instrucoes visuais de enquadramento.
- Upload de documentos de identidade e comprovante de endereco: `expo-document-picker` para PDF, `expo-image-picker` para fotos.
- Formatos aceitos: JPG, PNG, PDF. Tamanho maximo: 10 MB por arquivo (RN-010).

### 4.3 Build e Distribuicao

- **EAS Build:** Perfis `development`, `preview`, `production`.
- **EAS Update:** OTA updates para JavaScript sem novo build nativo.
- **Distribuicao:** App Store (iOS) + Google Play (Android).

---

## 5. Comunicacao e API

### 5.1 Protocolo

| Aspecto | Padrao | Regra |
|---|---|---|
| Protocolo | API REST (JSON) | Obrigatorio para toda comunicacao frontend/mobile <-> backend. |
| Cliente HTTP | `fetch` nativo | Axios proibido sem ADR. |
| Cache/sync | TanStack Query | Camada obrigatoria entre `fetch` e componentes. |
| Documentacao | Swagger/OpenAPI auto-gerado via `@nestjs/swagger` | Todo endpoint documentado. |
| Autenticacao | JWT (via Supabase Auth) | Access token curto (15-30 min) em memoria. Refresh token em httpOnly cookie (web) / expo-secure-store (mobile). |
| Versionamento | `/api/v1/[recurso]` | Versao incrementada em breaking changes. Versoes anteriores mantidas por 3 meses. |
| Paginacao | Cursor-based (padrao) / Offset-based (tabelas com sort) | Limite maximo 100 itens/pagina. Padrao: 20. |

### 5.2 Comunicacao em Tempo Real

| Cenario | Tecnologia | Justificativa |
|---|---|---|
| Streaming de respostas do Analista de Oportunidades | SSE (Server-Sent Events) via Vercel AI SDK | Comunicacao unidirecional: backend envia tokens incrementalmente ao frontend. SLA 5s individual, 10s comparativo (RN-050). |
| Novas oportunidades, status de propostas/negociacoes, notificacoes in-app | Supabase Realtime | Subscriptions de dados em tabelas do banco. Infraestrutura Supabase ja existente. Atualizacao do Dashboard em ate 60s. |
| Chat da negociacao com Admin (mensagens) | Supabase Realtime | Mensagens trocadas com o Admin (RN-026). Padrao de subscription em tabela de mensagens filtrado por negociacao. |
| WebSocket bidirecional customizado | Proibido sem ADR | SSE + Supabase Realtime cobrem todos os casos de uso identificados nas RNs. |

### 5.3 Padroes de Resposta

Herdados integralmente do ShiftLabs Stacks v7.0 secao 4.3:

- Sucesso: `{ "success": true, "data": {}, "meta": {} }`
- Erro: `{ "success": false, "error": { "code": "...", "message": "...", "details": {} } }`
- Codigos HTTP: 400, 401, 403, 404, 409, 422, 500.
- Stack traces nunca em producao.
- Mensagens de erro para usuario final tratadas no frontend (mapeamento de `error.code` para mensagens pt-BR conforme MSG-001 a MSG-023 das RNs).

### 5.4 Upload de Arquivos

- **Plataforma:** Supabase Storage.
- **Estrategia:** Upload direto do frontend para Supabase Storage via signed URL gerada pelo backend.
- **Nomenclatura:** UUID v4 como nome do arquivo. Extensao original preservada.
- **Validacao backend:** Whitelist de tipos MIME. JPG, PNG, PDF para KYC (RN-010). JPG, PNG, PDF para comprovantes de Escrow (RN-028). Tamanho maximo: 10 MB.
- **Validacao frontend:** Tipo e tamanho antes de iniciar upload para feedback imediato.

### 5.5 Data, Hora e Timezone

- **Backend/banco:** UTC obrigatorio. `timestamptz` em todo campo de data.
- **API:** ISO 8601. Exemplo: `2026-03-22T14:30:00.000Z`.
- **Frontend/mobile:** Converter para timezone do usuario apenas na camada de apresentacao. Critico para exibicao de prazos de Escrow (RN-028), expiracoes de proposta (RN-023), timestamps de notificacoes.
- **Locale:** `pt-BR` para formatacao de datas exibidas ao usuario.

---

## 6. Repositorio e Codigo

### 6.1 Estrutura do Monorepo

| Aspecto | Padrao | Regra |
|---|---|---|
| Organizacao | Monorepo com Turborepo 2.x+ | Repos separados por camada proibidos. |
| Package manager | pnpm 9.x+ | npm e yarn proibidos em novos projetos. |
| Plataforma | GitHub | Todo repo com README.md, .env.example, .gitignore, turbo.json, docker-compose.yml. |
| Nome do repositorio | `shiftlabs-repasse` | Monorepo do produto Repasse Seguro. Modulo Cessionario como app dentro do monorepo. |
| Docker Engine | 27+ | Obrigatorio para ambiente local. |
| Docker Compose | 2.x+ | Redis, RabbitMQ, Supabase local. |

```
shiftlabs-repasse/
  apps/
    api/                    # NestJS (backend compartilhado entre modulos)
    web-cessionario/        # React + Vite (dashboard do Cessionario)
    mobile-cessionario/     # Expo (app mobile do Cessionario)
  packages/
    shared-types/           # Types compartilhados (Proposta, Negociacao, Oportunidade, Escrow)
    shared-utils/           # Validacoes, formatadores, constantes
    shared-services/        # TanStack Query hooks, wrappers de API
    config-eslint/
    config-typescript/
    ui/                     # Componentes shadcn/ui reutilizaveis entre apps web
  turbo.json
  package.json
  docker-compose.yml
  .env.example
  pnpm-workspace.yaml
```

### 6.2 Estrutura de Pastas (Frontend Web -- React + Vite)

Herdada do ShiftLabs Stacks v7.0 secao 5.4:

```
src/
  components/
    ui/                     # shadcn/ui
    [ComponentName]/
  pages/
    Dashboard/
    Oportunidades/
    MinhasPropostas/
    Negociacoes/
    Assinaturas/
    Financeiro/
    AssistenteIA/
    MeuPerfil/
  hooks/
  utils/
  services/
    api.ts                  # Fetch wrapper com interceptor de auth
    analytics.ts            # Wrapper PostHog
  stores/
  types/
  lib/
    utils.ts                # cn() helper
  styles/
    global.css              # @theme directives, design tokens, motion tokens
  App.tsx
  main.tsx
```

### 6.3 Padroes de Codigo

Herdados integralmente do ShiftLabs Stacks v7.0 secao 5.6:

- **ESLint + Prettier** obrigatorios. Prettier: `semi: true`, `singleQuote: true`, `trailingComma: 'all'`, `printWidth: 100`.
- **Husky + lint-staged** para pre-commit.
- **Conventional Commits:** `tipo(escopo): descricao`. Tipos: `feat`, `fix`, `refactor`, `docs`, `style`, `test`, `chore`, `perf`, `ci`.
- **Nomenclatura:** Backend em `kebab-case`. Frontend em `PascalCase` para componentes, `camelCase` para hooks/utils. Banco em `snake_case`.
- **Branch principal:** `main`. Branches: `tipo/descricao` (ex: `feat/kyc-stepper`, `fix/escrow-deadline`).
- **PRs obrigatorios.** Pipeline de CI como gate principal.

---

## 7. Testes

### 7.1 Stack de Testes

| Camada | Ferramenta | Versao | Status |
|---|---|---|---|
| Backend -- unitarios | Vitest | latest | ✅ Obrigatorio |
| Backend -- integracao | Supertest + NestJS Testing Module | latest | ✅ Obrigatorio |
| Frontend web -- componentes | Vitest + React Testing Library | latest | ✅ Obrigatorio |
| Frontend web -- E2E | Playwright | latest | ✅ Obrigatorio (fluxos criticos) |
| Mobile -- componentes | Jest + React Native Testing Library | latest | ✅ Obrigatorio |
| Mobile -- E2E | Maestro | latest | ✅ Obrigatorio (fluxos criticos) |
| IA -- unitarios (logica ao redor do LLM) | Vitest | latest | ✅ Obrigatorio |
| IA -- integracao (pipeline completo) | Vitest + temperature 0 + seed fixo | latest | ✅ Obrigatorio |
| IA -- evals (qualidade) | Langfuse Evals + golden datasets | latest | ✅ Obrigatorio |

### 7.2 Cenarios Criticos Obrigatorios (vinculados a RNs)

| ID Teste | Cenario | RN | Prioridade |
|---|---|---|---|
| TC-CES-01 | Proposta sem KYC aprovado: botao desabilitado | RN-009 | Alta |
| TC-CES-02 | Limite de 3 propostas simultaneas | RN-020 | Alta |
| TC-CES-03 | Rate limit 10 propostas/24h | RN-020 | Alta |
| TC-CES-04 | Cancelamento automatico de Escrow por prazo | RN-028 | Critica |
| TC-CES-05 | Calculo de comissao com Delta > 0 | RN-035 | Critica |
| TC-CES-06 | Calculo de comissao com Delta <= 0 | RN-036 | Critica |
| TC-CES-07 | Agente de IA bloqueado em dados do Cedente | RN-049 | Alta |
| TC-CES-10 | 4 criterios de fechamento simultaneos | RN-039 | Critica |
| TC-CES-11 | Anonimizacao de dados do Cedente em todo o modulo | RN-014/RN-067 | Critica |
| TC-CES-16 | Multiplos Cessionarios na mesma oportunidade | RN-017/RN-020 | Critica |

### 7.3 Golden Datasets para Analista de Oportunidades

- Minimo 50 exemplos cobrindo 6 tipos de consulta (analise individual, comparativo, simulacao ROI, regras da plataforma, recomendacao personalizada, historico de operacoes) + 7 cenarios de recusa (dados do Cedente, outros Cessionarios, prompt injection, dados fora do escopo, linguagem de urgencia, informacoes financeiras sensiveis, consultas fora do dominio).
- Armazenados em `tests/evals/analista-oportunidades/`.
- Metricas: faithfulness, relevance, correctness, format compliance, ausencia de linguagem de urgencia (RN-047).

---

## 8. Deploy e CI/CD

### 8.1 Pipeline

| Aspecto | Padrao | Regra |
|---|---|---|
| Plataforma CI/CD | GitHub Actions | Pipeline minimo: lint -> type-check -> testes -> prisma migrate deploy -> build -> deploy. |
| Backend producao | Railway (Docker nativo) | [DECISAO AUTONOMA -- Railway. Docker nativo, auto-deploy via GitHub, pricing previsivel. Descartado: Render (cold starts), VPS (overhead).] |
| Frontend web producao | Vercel ou Railway | [DECISAO AUTONOMA] Vercel recomendado mesmo para SPA React+Vite pela integracao com preview deployments por PR e edge network. Alternativa: Railway se houver restricao de custo. |
| Banco de dados producao | Supabase (gerenciado) | 3 projetos: dev, staging e prod. Dados financeiros exigem staging isolado. |
| Mobile producao | EAS Build + App Store + Google Play | Perfis: development, preview, production. |
| Preview deployments | Automaticos por PR | Obrigatorio. Vercel faz nativamente. |

### 8.2 Ambientes

| Ambiente | Backend | Frontend | Mobile | Banco |
|---|---|---|---|---|
| Local | Docker (Redis, RabbitMQ, Supabase CLI) | Vite dev server | Expo dev client | Supabase local |
| Staging | Railway (staging) | Preview deployment | EAS preview build | Supabase projeto staging |
| Producao | Railway (producao) | Deploy final | Stores (iOS/Android) | Supabase projeto producao |

### 8.3 Migrations em Producao

- `pnpm prisma migrate deploy` no pipeline de CI, antes do deploy do codigo.
- Rollback via nova migration (nunca editar migration aplicada).
- Migrations destrutivas em 2 etapas: deploy do codigo primeiro, migration depois.

---

## 9. Seguranca

### 9.1 Requisitos Gerais

| Requisito | Status | Justificativa |
|---|---|---|
| HTTPS obrigatorio | ✅ | Dados financeiros e documentos de cessao trafegam na plataforma. |
| CORS configurado (nunca `*` em producao) | ✅ | Protecao contra requisicoes cross-origin nao autorizadas. |
| Sanitizacao de inputs (class-validator) | ✅ | Protecao contra SQL injection e XSS. Critico para propostas (RN-021) e chat (RN-026). |
| CSRF obrigatorio | ✅ | Refresh token em httpOnly cookie exige protecao CSRF. Acoes financeiras (Escrow, assinatura) devem ser protegidas. |
| Rate limiting (@nestjs/throttler) | ✅ | Protecao de endpoints de autenticacao, propostas (RN-020), KYC (RN-006). |
| Helmet.js | ✅ | Headers de seguranca automaticos. |
| Credentials em variaveis de ambiente | ✅ | .env no .gitignore. .env.example com chaves sem valores. |
| RLS no Supabase | ✅ | Isolamento de dados por Cessionario (RN-013/RN-068). Anonimizacao do Cedente (RN-014/RN-067). |
| service_role key nunca no frontend | ✅ | Apenas anon key no cliente. |
| Certificate pinning (mobile) | ✅ Obrigatorio | [DECISAO AUTONOMA -- Certificate pinning obrigatorio. Dados financeiros. Descartado: nao implementar (risco inaceitavel).] Dados financeiros do Cessionario (valores de operacoes, conta escrow) exigem protecao adicional. Implementado via expo-dev-client. |

### 9.2 Seguranca Especifica do Cessionario

| Requisito | RN Vinculada | Implementacao |
|---|---|---|
| KYC obrigatorio antes da primeira proposta | RN-009 | Guard no endpoint de criacao de proposta. Verificacao de status KYC no backend. |
| Isolamento de dados por Cessionario (RBAC) | RN-013, RN-068 | Toda query filtrada por `cessionario_id` do usuario logado. NestJS Guard + Prisma middleware. RLS no Supabase. |
| Anonimizacao do Cedente no backend | RN-014, RN-067 | Dados do Cedente nunca trafegam para o frontend do Cessionario. Filtragem no service layer, nao no frontend. |
| Re-autenticacao para acoes criticas | RN-005, RN-070 | Janela de confianca de 15 minutos. Modal de confirmacao de senha/biometria para: deposito Escrow, assinatura, alteracao de dados bancarios, solicitacao de reversao. |
| Sessao com expiracao por inatividade | RN-004 | 30 minutos de inatividade. Aviso previo de 5 minutos. |
| Bloqueio por excesso de falhas de KYC | RN-006 | 5 tentativas em 1 hora: bloqueio de 30 minutos do envio de documentos. |
| Consentimento LGPD para IA | RN-016 | Checkbox individual no cadastro. Revogacao a qualquer momento em "Meu Perfil". |
| Exportacao/exclusao de dados (LGPD) | RN-015 | Exportacao em ate 48h. Exclusao com anonimizacao de dados nao financeiros. Dados financeiros retidos por 5 anos. |

### 9.3 Seguranca de IA (Analista de Oportunidades)

| Requisito | Implementacao |
|---|---|
| Prompt injection protection | Sanitizacao de input. Delimitadores explicitos (`<user_input>...</user_input>`). Validacao de output antes de exibir. |
| Rate limiting por usuario em endpoints LLM | 20-60 chamadas/min por Cessionario. |
| Content filtering | Moderacao de output: linguagem de urgencia, FOMO e apelo emocional bloqueados (RN-047). |
| PII masking em prompts | Nunca enviar CPF, dados bancarios ou informacoes do Cedente para a API da OpenAI. |
| API keys da OpenAI | Variavel de ambiente no backend. Nunca no frontend/mobile. Toda chamada LLM via backend. |
| Kill switch via feature flag | PostHog Feature Flag para desabilitar Analista de Oportunidades instantaneamente em producao. |

---

## 10. Analytics e Tracking

### 10.1 Ferramenta

| Aspecto | Padrao | Regra |
|---|---|---|
| Ferramenta | PostHog Cloud | Obrigatorio desde o lancamento. |
| SDKs | posthog-js (web), posthog-react-native (mobile), posthog-node (backend) | Integrado com Sentry para correlacao erro-comportamento. |
| Wrapper | `src/services/analytics.ts` | Obrigatorio. Nunca chamar SDK diretamente nos componentes. |
| Nomenclatura | `snake_case`: `[objeto]_[acao]` | Ex: `proposal_created`, `escrow_deposited`, `document_signed`. |

### 10.2 Eventos Obrigatorios do Modulo Cessionario

| Evento | Descricao | Propriedades Adicionais |
|---|---|---|
| `screen_viewed` | Toda tela | `screen_name`, `platform` |
| `signup_completed` | Cadastro concluido | `method` (email/google) |
| `login_completed` | Login concluido | `method` (email/google) |
| `kyc_started` | Inicio do fluxo de KYC | -- |
| `kyc_completed` | KYC enviado para analise | -- |
| `kyc_approved` | KYC aprovado | -- |
| `opportunity_viewed` | Detalhe de oportunidade aberto | `opportunity_code` |
| `proposal_created` | Proposta enviada | `opportunity_code`, `value` |
| `proposal_cancelled` | Proposta cancelada | `opportunity_code` |
| `negotiation_started` | Negociacao iniciada | `opportunity_code` |
| `counteroffer_sent` | Contraproposta enviada | `opportunity_code`, `round`, `value` |
| `escrow_deposited` | Comprovante de deposito enviado | `opportunity_code`, `value` |
| `document_signed` | Documento assinado via ZapSign | `opportunity_code`, `document_type` |
| `operation_completed` | Operacao concluida | `opportunity_code`, `total_value` |
| `ai_query_sent` | Consulta ao Analista de Oportunidades | `query_type` (individual/comparativo) |
| `ai_response_received` | Resposta recebida do Analista | `latency_ms`, `query_type` |

### 10.3 Privacidade

- Nunca enviar para analytics: senhas, tokens, CPF, dados bancarios, dados do Cedente.
- Consentimento LGPD exibido no cadastro (RN-016). Opt-out disponivel em "Meu Perfil".
- `user_id` como unico identificador pessoal em eventos.

---

## 11. Tecnologias Proibidas

| Tecnologia | Categoria | Motivo |
|---|---|---|
| Next.js | Frontend web | Modulo Cessionario e 100% logado. React + Vite e o caminho correto. |
| Express, Fastify, Hono, Koa | Backend | NestJS e o unico framework HTTP aprovado. |
| TypeORM, Sequelize, Drizzle | ORM | Prisma e o unico ORM aprovado. |
| MySQL, MariaDB, SQLite | Banco relacional | PostgreSQL (Supabase) e o unico aprovado. |
| MongoDB, DynamoDB | Banco NoSQL | Nao aprovado sem ADR. |
| Axios | Cliente HTTP | `fetch` nativo e o padrao. |
| Redux, MobX | Estado global | Zustand e o padrao. |
| Context API (como state manager global) | Estado global | Zustand e o padrao. |
| MUI, Chakra, AntD, Bootstrap | UI | shadcn/ui e o unico sistema aprovado. |
| UnoCSS, Windi CSS | CSS | Tailwind CSS e o padrao. |
| React Navigation | Navegacao mobile | expo-router e o padrao. |
| NativeWind, Styled Components (mobile) | Estilizacao mobile | `StyleSheet.create()` nativo. |
| React Native Paper, NativeBase, Tamagui | UI mobile | Componentes construidos do zero. |
| Moment.js, Day.js, Luxon | Datas | date-fns e o padrao. |
| OneSignal, Pusher | Push notifications | Expo Notifications e o padrao. |
| npm, yarn | Package manager | pnpm e o padrao. |
| react-router-dom | Roteamento web | TanStack Router e o padrao para React + Vite. |
| Flutter, Kotlin Multiplatform | Mobile | React Native + Expo e o padrao. |
| GSAP (como substituto de Framer Motion) | Animacoes web | Framer Motion e o padrao. GSAP como complemento via ADR. |
| Three.js/WebGL | Produto principal | Reservado para landing pages. |

---

## 12. ADRs

### ADR-001: Rate Limiting de Endpoints LLM por Cessionario

| Campo | Valor |
|---|---|
| Status | Proposto |
| Data | 2026-03-22 |
| Contexto | O Analista de Oportunidades (RN-048) permite consultas livres ao LLM. Sem rate limiting por usuario, um Cessionario pode consumir cota desproporcional de tokens OpenAI, impactando custo e latencia para outros usuarios. |
| Decisao | Implementar rate limiting de 30 chamadas/minuto por Cessionario nos endpoints de LLM, independente do rate limiting global. Valor ajustavel via variavel de ambiente. |
| Consequencias | Protege custo e latencia. Cessionarios com uso intensivo recebem mensagem: "Limite de consultas atingido. Aguarde alguns instantes." |
| Alternativas descartadas | (1) Sem rate limiting: risco de custo descontrolado. (2) Rate limiting global sem filtro por usuario: penaliza todos por abuso de um. |

### ADR-002: Supabase Realtime para Atualizacao do Dashboard

| Campo | Valor |
|---|---|
| Status | Aprovado |
| Data | 2026-03-22 |
| Contexto | O Dashboard deve atualizar em ate 60 segundos apos evento (SLA Parte 01.3). Alternativas: polling (simples, mas gera carga), Supabase Realtime (event-driven, infraestrutura ja existente), WebSocket customizado (complexidade desnecessaria). |
| Decisao | Supabase Realtime para subscriptions em tabelas de propostas, negociacoes, escrow e notificacoes. TanStack Query invalidation ao receber evento. |
| Consequencias | Atualizacao near-realtime sem polling. Reusa infraestrutura Supabase. Filtragem por `cessionario_id` garante isolamento (RN-068). |
| Alternativas descartadas | (1) Polling a cada 30s: gera carga desnecessaria no backend. (2) WebSocket customizado: complexidade sem ganho sobre Supabase Realtime ja disponivel. |

### ADR-003: Confirmacao Manual de Escrow no MVP

| Campo | Valor |
|---|---|
| Status | Aprovado |
| Data | 2026-03-22 |
| Contexto | RN-064 define que o deposito em Escrow e confirmado pelo Admin apos verificacao bancaria. Integracao automatica com banco exige parceiro financeiro definido e contrato firmado. |
| Decisao | MVP com confirmacao manual pelo Admin. Reconciliacao bancaria automatica planejada para v2 apos definicao do parceiro financeiro. |
| Consequencias | Admin e gargalo na confirmacao. SLA de 24h uteis (RN-055) mitiga o impacto. |
| Alternativas descartadas | Automatizacao imediata: parceiro financeiro e integracao bancaria nao definidos. |

---

## 13. Changelog

| Versao | Data | Autor | Descricao |
|---|---|---|---|
| v1.1 | 2026-03-22 | Claude Code Desktop | Resolucao de todas as pendencias de stack (PEND-STACK-001 a PEND-STACK-015). Definidos: Railway (backend), Celcoin (escrow), idwall (KYC), Resend (e-mail), certificate pinning obrigatorio, thresholds de custo OpenAI (US$ 50/200 dia), golden datasets (50 exemplos, 6+7 cenarios), infraestrutura de producao (Upstash, CloudAMQP, 3 projetos Supabase, Langfuse Cloud). Boilerplates e pipelines com prazos atribuidos. |
| v1.0 | 2026-03-22 | Claude Code Desktop | Criacao do documento. Stack normativa do modulo Cessionario derivada do ShiftLabs Stacks v7.0 e das 72 RNs documentadas nas Partes 01.1 a 01.5. |

---

## 14. Backlog de Pendencias

> **Todas as pendencias foram resolvidas na v1.1 (2026-03-22).**

| ID | Tipo | Descricao | Resolucao | Status |
|---|---|---|---|---|
| PEND-STACK-001 | Infraestrutura | Provedor de hospedagem backend. | Railway. Docker nativo, auto-deploy via GitHub, pricing previsivel. | ✅ Resolvido |
| PEND-STACK-002 | Infraestrutura | Boilerplate monorepo Turborepo + pnpm. | Pre-requisito antes do primeiro sprint. | ✅ Resolvido |
| PEND-STACK-003 | Infraestrutura | Boilerplate backend NestJS + Prisma + Supabase. | Pre-requisito antes do primeiro sprint. | ✅ Resolvido |
| PEND-STACK-004 | Infraestrutura | Boilerplate frontend React + Vite. | Pre-requisito antes do primeiro sprint. | ✅ Resolvido |
| PEND-STACK-005 | Infraestrutura | Boilerplate mobile Expo. | Pre-requisito antes do primeiro sprint. | ✅ Resolvido |
| PEND-STACK-006 | Infraestrutura | Projeto Sentry (organizacao, projetos, DSN). | Setup do boilerplate. | ✅ Resolvido |
| PEND-STACK-007 | Infraestrutura | Projeto PostHog (cloud, API key, wrapper). | PostHog Cloud. Setup do boilerplate. | ✅ Resolvido |
| PEND-STACK-008 | Infraestrutura | Pipeline CI/CD (GitHub Actions). | GitHub Actions: lint -> type-check -> testes -> build -> deploy. Primeiro sprint. | ✅ Resolvido |
| PEND-STACK-009 | Integracao | Parceiro financeiro para conta Escrow. | Celcoin. Fintech regulada BCB, escrow digital nativo. MVP manual conforme ADR-003, integracao automatica em v2. | ✅ Resolvido |
| PEND-STACK-010 | Integracao | Provedor de servico de KYC (OCR + liveness). | idwall. Lider em KYC digital no Brasil, API REST, LGPD compliant. | ✅ Resolvido |
| PEND-STACK-011 | Integracao | Provedor de e-mail transacional. | Resend. SDK TypeScript nativo, DX excelente, pricing transparente. | ✅ Resolvido |
| PEND-STACK-012 | Seguranca | Estrategia de certificate pinning mobile. | Obrigatorio. Dados financeiros exigem protecao adicional. Via expo-dev-client. | ✅ Resolvido |
| PEND-STACK-013 | IA | Boilerplate do modulo de IA (NestJS + OpenAI SDK + Langfuse + pgvector). | Pre-requisito antes do desenvolvimento do Analista de Oportunidades. | ✅ Resolvido |
| PEND-STACK-014 | IA | Golden datasets para avaliacao do Analista de Oportunidades. | Minimo 50 exemplos cobrindo 6 tipos de consulta + 7 cenarios de recusa. Antes do lancamento do modulo IA. | ✅ Resolvido |
| PEND-STACK-015 | IA | Threshold de custo diario OpenAI para o modulo Cessionario. | US$ 50/dia alerta, US$ 200/dia hard limit. Configurado no Langfuse. | ✅ Resolvido |
