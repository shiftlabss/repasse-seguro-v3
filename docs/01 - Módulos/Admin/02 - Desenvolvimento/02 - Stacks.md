# Stacks Tecnológicas — Módulo Admin

## Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Tech Lead / CTO |
| **Escopo** | Stack normativa do módulo Admin — Repasse Seguro |
| **Versão** | v1.1 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 2026-03-22 (America/Fortaleza) |
| **Fonte primária** | ShiftLabs Stacks v7.0 (Fernando Calado) |
| **Fonte secundária** | Regras de Negócio do Admin v1.2 (01.1 a 01.5) |
| **Caráter** | **Normativo.** Desvios exigem ADR aprovado. |

---

> **📌 TL;DR**
>
> - **Backend:** Node.js 22+ / NestJS 10+ / TypeScript strict / Prisma 6+ / Supabase PostgreSQL 17+. Redis 7.4+ para cache. RabbitMQ 4+ para filas. Pino para logs.
> - **Frontend Web:** React 19+ / Vite 7+ / TypeScript strict / TanStack Router 1+ / TanStack Query 5+ / Zustand 5+ / Tailwind CSS 4 / shadcn/ui / Framer Motion 12+. Dashboard 100% logado — Next.js proibido neste módulo.
> - **Mobile:** React Native 0.76+ / Expo SDK 52+ / expo-router 4+ / Reanimated 3+. Mesma API REST do web.
> - **Tecnologias aprovadas:** 47 tecnologias com status definido. **Proibidas:** 28 tecnologias com alternativa documentada.
> - **Desvios do baseline ShiftLabs:** 3 ADRs documentados (Supabase Realtime para Pipeline, webhook HMAC-SHA256 para integrações, audit trail em schema separado).
> - **Dependências externas:** ZapSign (assinatura digital), Celcoin (Conta Escrow), Meta Cloud API (WhatsApp), Twilio (SMS), Supabase (BaaS).
> - **Infraestrutura de produção:** Railway (backend), Upstash (Redis), CloudAMQP (RabbitMQ), Vercel (frontend web).
> - **Seções pendentes:** Nenhuma. Todas as definições resolvidas na v1.1.

---

## 1. Principios de Governanca Tecnologica

1. **Baseline primeiro.** O ShiftLabs Stacks v7.0 e a fonte primaria de verdade. Toda tecnologia deste documento herda do baseline, salvo desvio documentado via ADR.
2. **Justificativa obrigatoria.** Nenhuma tecnologia entra ou sai do projeto sem justificativa tecnica vinculada a uma regra de negocio ou requisito de infraestrutura.
3. **Desenvolvimento 100% por IA.** Todo codigo e gerado por Claude Code Desktop. Boilerplates, convencoes e testes sao os contratos de qualidade. Decisoes de stack priorizam previsibilidade do output de IA.
4. **Rastreabilidade de desvio.** Todo desvio do baseline gera um ADR numerado com contexto, decisao, alternativas e consequencias.
5. **Proibicao explicita.** Tecnologias nao listadas como aprovadas ou condicionais sao proibidas por padrao. Toda proibicao tem alternativa aprovada.

---

## 2. Backend

### 2.1 Stack Obrigatoria

| Tecnologia | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **Node.js** | 22.x+ (LTS) | ✅ Aprovado | Runtime padrao ShiftLabs. Ecossistema maduro para IA geradora de codigo. | Usar versao LTS. Atualizar conforme release cycle do Node. |
| **NestJS** | 10.x+ | ✅ Aprovado | Arquitetura modular com decorators, Swagger embutido, guards/pipes nativos. Output de IA mais consistente. | Framework HTTP unico. Toda API REST do Admin usa NestJS. |
| **TypeScript** | 5.4+ | ✅ Aprovado | Tipagem estatica obrigatoria. `strict: true` em todo tsconfig. | Nenhum arquivo `.js` ou `.jsx` permitido. |
| **Prisma** | 6.x+ | ✅ Aprovado | ORM unico aprovado. Tipagem auto-gerada, migrations declarativas, integracao nativa com Supabase. | Unico ORM permitido. Queries raw via `$queryRaw` com parametros preparados. |
| **PostgreSQL (Supabase)** | 17+ | ✅ Aprovado | Banco relacional unico. Gerenciado via Supabase. | PostgreSQL auto-gerenciado proibido em producao. |
| **Supabase** | latest | ✅ Aprovado | PostgreSQL gerenciado + Auth + Storage + Realtime + Edge Functions. Elimina infra operacional. | Plataforma padrao para banco, auth, storage e realtime. |
| **Supabase Auth** | latest | ✅ Aprovado | Autenticacao de operadores Admin (Analista, Coordenador, Gestor Financeiro, Master). 2FA obrigatorio (RN-005). | Complementa JWT proprio. Suporta 2FA via aplicativo autenticador. |
| **Supabase Storage** | latest | ✅ Aprovado | Armazenamento de documentos do dossie (RN-002), contratos, comprovantes, envelopes ZapSign. | Upload via signed URL. Nomenclatura UUID v4. |
| **Supabase Realtime** | latest | ✅ Aprovado | Pipeline em tempo real ate 5 segundos (RN-151). Monitoramento de Contas Escrow (RN-087). Supervisao IA com refresh de 10s (RN-093). | Ver ADR-001. Subscriptions de dados em tabelas de casos e escrow. |
| **Redis** | 7.4+ | ✅ Aprovado | Cache de KPIs da Dashboard (RN-011, polling 60s), sessoes JWT (blacklist de tokens revogados, RN-134), cache de configuracoes (RN-111). | TTL obrigatorio em toda chave. Estrategia de invalidacao documentada. Docker local, **Upstash** em producao. `[DECISÃO AUTÔNOMA — Upstash serverless Redis. Zero ops, pay-per-use. Alternativa descartada: Railway addon (custo fixo), ElastiCache (over-engineering para o estágio atual).]` |
| **RabbitMQ** | 4.x+ | ✅ Aprovado | Filas para: notificacoes asincronas (RN-062, SLA 5min para e-mail), distribuicao automatica de Conta Escrow (RN-021), relatorios mensais (RN-092), webhooks de saida (RN-138). | Retry obrigatorio + dead-letter queue em toda fila. Docker local, **CloudAMQP** em producao. `[DECISÃO AUTÔNOMA — CloudAMQP gerenciado. Free tier para dev, escalável. Alternativa descartada: Railway addon (menos controle sobre configuração).]` |
| **Pino** | 9.x+ | ✅ Aprovado | Logs estruturados obrigatorios. Request ID, timestamp, nivel e contexto. | Todo servico NestJS configura Pino. Deploy sem logs estruturados proibido. |
| **class-validator + class-transformer** | latest | ✅ Aprovado | Validacao de DTOs integrada aos pipes do NestJS. | Toda entrada de dados validada via pipes. |
| **@nestjs/swagger** | latest | ✅ Aprovado | Swagger/OpenAPI auto-gerado. | Obrigatorio em todo endpoint. Documentacao gerada automaticamente. |
| **Helmet** | 8.x+ | ✅ Aprovado | Headers de seguranca automaticos. | Middleware adicionado desde o primeiro commit. |
| **@nestjs/throttler** | latest | ✅ Aprovado | Rate limiting. Endpoints de auth com limites mais restritivos (RN-005: 5 tentativas, bloqueio 30min). Anti-fraude de propostas (RN-031: 10 propostas/24h). | Todo endpoint publico com rate limiting. |
| **@sentry/nestjs** | 9.x+ | ✅ Aprovado | Error tracking com stack traces, breadcrumbs e alertas. | Obrigatorio em producao. Source maps enviados no build. |

### 2.2 Integracoes Externas

| Tecnologia | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **Celcoin (Conta Escrow)** | API REST v2+ | ✅ Aprovado | Parceiro financeiro para Conta Escrow digital. Fintech regulada pelo Banco Central do Brasil. APIs REST modernas para escrow digital, split de pagamentos e conciliacao automatica. Custo por transacao competitivo. | Integracao via API REST. Webhooks validados via HMAC-SHA256 (ADR-002). `[DECISÃO AUTÔNOMA — Celcoin escolhida por regulação BCB, APIs REST modernas e escrow digital nativo. Alternativa descartada: Transfeera (foco em pagamentos, não escrow).]` |
| **Meta Cloud API (WhatsApp Business)** | latest | ✅ Aprovado | Canal de notificacao primario para Cedentes e Cessionarios no Brasil (RN-063). Integracao direta sem intermediario, custo por conversa. | Canal dominante para o publico-alvo. Templates de mensagem pre-aprovados. `[DECISÃO AUTÔNOMA — WhatsApp via Meta Cloud API + SMS via Twilio no MVP. WhatsApp é canal dominante no Brasil para o público-alvo. Alternativa descartada: implementar apenas em fase posterior (risco de baixo engajamento dos Cedentes no MVP).]` |
| **Twilio (SMS)** | latest | ✅ Aprovado | Canal de notificacao secundario (fallback quando WhatsApp nao disponivel). Cobertura global, API consolidada. | Fallback para SMS quando WhatsApp nao entregue. Mesmo wrapper de notificacoes. |

### 2.3 Convencoes de Banco de Dados (Prisma Schema)

Herdadas integralmente do ShiftLabs Stacks v7.0, secao 1.5. Destaques criticos para o Admin:

- **Primary keys:** UUID v4 em todas as tabelas. Auto-increment proibido.
- **Colunas de auditoria:** `created_at`, `updated_at` obrigatorios em toda tabela. `created_by`, `updated_by` obrigatorios em tabelas de dominio (casos, propostas, dossie, configuracoes).
- **Soft delete:** Padrao para tabelas de dominio (casos, usuarios, propostas, documentos, envelopes ZapSign). Coluna `deleted_at DateTime? @db.Timestamptz`.
- **Versionamento de entidade:** Campo `version Int @default(1)` obrigatorio em toda entidade editavel para lock otimista (RN-130).
- **Snapshot de configuracao:** Cada caso armazena copia dos parametros vigentes no momento de sua criacao (RN-111).
- **Audit trail:** Schema separado append-only para registros de auditoria (RN-129, DA-012). Retencao de 5 anos.
- **Timestamps:** `@db.Timestamptz` obrigatorio. `@db.Timestamp` sem timezone proibido.
- **Nomenclatura:** Tabelas em `snake_case` plural. Colunas em `snake_case`. Enums em `PascalCase` com valores `UPPER_SNAKE_CASE`.

---

## 3. Frontend Web

### 3.1 Criterio de Decisao

O modulo Admin e um **dashboard 100% logado**. Nenhuma pagina precisa de SEO ou indexacao por buscadores. O setup obrigatorio e **React + Vite (SPA)**. Next.js e proibido neste modulo.

### 3.2 Stack Obrigatoria

| Tecnologia | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **React** | 19.x+ | ✅ Aprovado | Biblioteca de UI padrao ShiftLabs. Ecossistema maduro para geracao por IA. | Componentes funcionais. Class components proibidos. |
| **React DOM** | 19.x+ | ✅ Aprovado | Renderer web. | Acompanha versao do React. |
| **Vite** | 7.x+ | ✅ Aprovado | Bundler para SPA logado. HMR rapido, build otimizado. | Bundler unico para o Admin. |
| **TypeScript** | 5.4+ | ✅ Aprovado | `strict: true` obrigatorio. | Nenhum arquivo `.js` ou `.jsx` permitido. |
| **TanStack Router** | 1.x+ | ✅ Aprovado | Type-safe routing. Integracao nativa com TanStack Query. | Roteador unico do Admin. `react-router-dom` proibido sem ADR. |
| **TanStack Query** | 5.x+ | ✅ Aprovado | Fetching, caching e sincronizacao de dados do servidor. Polling configuravel por modulo (RN-132). | Obrigatorio para toda comunicacao com API. `fetch` direto em `useEffect` proibido. |
| **Zustand** | 5.x+ | ✅ Aprovado | Estado global: sidebar ativa, perfil do operador logado, preferencias de UI, takeovers ativos. | Usar para estado de aplicacao. Dados do servidor via TanStack Query. |
| **Tailwind CSS** | 4.x+ | ✅ Aprovado | Framework de estilizacao padrao. Design tokens via `@theme`. | Unico framework CSS. Classes utilitarias. CSS custom para animacoes avancadas. |
| **shadcn/ui** | latest | ✅ Aprovado | Sistema de componentes. Baseado em Radix Primitives. Ownership total do codigo. | Unico sistema de componentes. Componentes copiados para o repo. |
| **Radix Primitives** | latest | ✅ Aprovado | Primitivas de acessibilidade (keyboard nav, ARIA, focus). Base do shadcn/ui. | Usado via shadcn/ui. Importacao direta permitida para componentes nao cobertos. |
| **Framer Motion (Motion)** | 12.x+ | ✅ Aprovado | Transicoes de tela (AnimatePresence), micro-interacoes, skeleton screens. Pipeline com drag-and-drop (RN-147). | Biblioteca de animacao obrigatoria. Motion tokens padronizados. |
| **date-fns** | 4.x+ | ✅ Aprovado | Manipulacao de datas. Tree-shakeable. | Importar apenas funcoes necessarias. |
| **date-fns-tz** | 3.x+ | ✅ Aprovado | Conversao de timezone na camada de apresentacao. Backend armazena UTC (RN-129 timestamps). | Conversao para timezone do operador apenas em componentes. |
| **@sentry/react** | 9.x+ | ✅ Aprovado | Error tracking no frontend. Error Boundary global. | Obrigatorio em producao. Breadcrumbs para contexto de debug. |
| **PostHog (posthog-js)** | 1.x+ | ✅ Aprovado | Analytics, session replay, feature flags. | Obrigatorio. Wrapper centralizado em `services/analytics.ts`. |
| **cn() helper (clsx + tailwind-merge)** | latest | ✅ Aprovado | Composicao condicional de classes. | Localizado em `src/lib/utils.ts`. |

### 3.3 Design Tokens

Herdados integralmente do ShiftLabs Stacks v7.0, secao 2.3.1. Implementados via `@theme` no `global.css`. Tokens obrigatorios: cores (compativel shadcn/ui), espacamento (base 4px), tipografia (Inter/JetBrains Mono), border-radius, sombras e motion tokens.

### 3.4 Motion e Animacoes

Herdadas do ShiftLabs Stacks v7.0, secao 2.4. Destaques para o Admin:

- **Micro-interacoes obrigatorias:** Botoes, cards de caso, inputs, links de sidebar — feedback visual em hover, focus e click.
- **Transicoes de tela:** `AnimatePresence` entre todas as rotas do TanStack Router.
- **Skeleton screens:** Obrigatorios em Dashboard (RN-011), Pipeline, Financeiro, Relatorios. Spinners genericos proibidos.
- **Confirmacao de Fechamento:** Animacao de pulsacao no botao "Confirmar Fechamento" quando 4 criterios cumpridos (RN-023).
- **Pipeline drag-and-drop:** Animacoes de reordenacao com spring physics (RN-147).

### 3.5 Acessibilidade (a11y)

Herdada do ShiftLabs Stacks v7.0, secao 2.5. Requisitos adicionais do Admin:

- Alertas de SLA nao dependem apenas de cor: icone diferenciado + label textual (RN-059).
- Graficos de Relatorios com tabela alternativa acessivel por leitores de tela (RN-108).
- Contraste minimo 4.5:1 (AA) para todas as interfaces.
- Tamanho minimo de toque 44x44px.

### 3.6 Performance

- **Bundle budget:** JavaScript total gzipped abaixo de 300KB (referencia ShiftLabs para SPAs React+Vite).
- **Code splitting:** `React.lazy()` + `Suspense` para rotas e componentes pesados (modais, drawers, graficos de Relatorios).
- **Lazy loading:** Modais de confirmacao de Fechamento, drawer de detalhes de Conta Escrow, graficos de funil.
- **Tree shaking:** ESM obrigatorio. Importar apenas funcoes necessarias.
- **Analise de bundle:** `rollup-plugin-visualizer` no Vite para identificar dependencias pesadas.

---

## 4. Mobile

### 4.1 Stack Obrigatoria

| Tecnologia | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **React Native** | 0.76+ | ✅ Aprovado | Framework mobile unico. Mesmo ecossistema React do web. | Managed workflow via Expo. |
| **Expo SDK** | 52+ | ✅ Aprovado | Build, deploy, OTA updates sem Xcode/Android Studio manual. | Eject para bare workflow via ADR. |
| **expo-router** | 4.x+ | ✅ Aprovado | File-based routing. Deep linking nativo. | Roteador unico. React Navigation proibido. |
| **TypeScript** | 5.4+ | ✅ Aprovado | `strict: true` obrigatorio. | Sem arquivos JS/JSX. |
| **TanStack Query** | 5.x+ | ✅ Aprovado | Mesma camada de dados do web. | Obrigatorio para comunicacao com API. |
| **Zustand** | 5.x+ | ✅ Aprovado | Estado global. Middleware `persist` com AsyncStorage para dados nao sensiveis. | Mesmo padrao do web. |
| **React Native Reanimated** | 3.x+ | ✅ Aprovado | Animacoes na UI thread. | Biblioteca de animacao obrigatoria. `Animated` API legacy proibida. |
| **React Native Gesture Handler** | 2.x+ | ✅ Aprovado | Interacoes tateis (swipe, drag, pinch). | Complementa Reanimated. |
| **expo-secure-store** | latest | ✅ Aprovado | Armazenamento seguro de tokens JWT (RN-134). | Obrigatorio para refresh tokens. AsyncStorage proibido para tokens. |
| **expo-image** | latest | ✅ Aprovado | Carregamento otimizado com cache e placeholder blur. | Substitui `<Image>` nativo para imagens remotas. |
| **Expo Notifications** | latest | ✅ Aprovado | Push notifications iOS/Android via Expo Push API. | Padrao para push. OneSignal/Pusher proibidos. |
| **@sentry/react-native** | 6.x+ | ✅ Aprovado | Error tracking mobile. | Obrigatorio em producao. |
| **PostHog (posthog-react-native)** | 3.x+ | ✅ Aprovado | Analytics mobile. | Wrapper centralizado. |
| **date-fns** | 4.x+ | ✅ Aprovado | Manipulacao de datas. | Tree-shakeable. |
| **date-fns-tz** | 3.x+ | ✅ Aprovado | Conversao de timezone. | Apenas na camada de apresentacao. |

### 4.2 Build e Distribuicao

- **EAS Build:** Perfis `development`, `preview`, `production`.
- **EAS Update:** OTA updates de JavaScript sem rebuild nativo.
- **Acessibilidade:** `accessibilityLabel`, `accessibilityRole`, `accessibilityState` obrigatorios. Teste com VoiceOver e TalkBack antes de cada release.

---

## 5. Comunicacao e API

| Tecnologia / Padrao | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **API REST (JSON)** | — | ✅ Aprovado | Protocolo padrao para toda comunicacao frontend/mobile e backend. | Endpoints por dominio: `/api/v1/[dominio]/[recurso]`. Swagger obrigatorio. |
| **fetch nativo** | — | ✅ Aprovado | Cliente HTTP padrao (browser e React Native). | Axios proibido. Wrapper em `services/api.ts` com interceptor de auth. |
| **JWT (JSON Web Tokens)** | — | ✅ Aprovado | Autenticacao stateless. Access token 15min, refresh token 8h (RN-134). | Access token em memoria (Zustand). Refresh token em httpOnly cookie (web) ou SecureStore (mobile). |
| **Supabase Realtime** | latest | ✅ Aprovado | Pipeline em tempo real (RN-151), monitoramento de Contas Escrow (RN-087), status de assinaturas ZapSign (RN-122). | Ver ADR-001. Subscriptions de dados. Fallback para polling se WebSocket indisponivel. |
| **SSE (Server-Sent Events)** | — | 🔶 Condicional | Streaming unidirecional. Reservado para features de IA (Guardiao do Retorno, Analista de Oportunidades). | Usar para streaming de respostas LLM. Nao usar para dados transacionais. |
| **WebSocket customizado** | — | 🔶 Condicional | Comunicacao bidirecional. Supabase Realtime cobre a maioria dos casos do Admin. | Apenas via ADR se Supabase Realtime nao cobrir o caso de uso. |
| **Versionamento por URL** | — | ✅ Aprovado | `/api/v1/[recurso]`. | Incrementar versao em breaking changes. Manter versao anterior por 3 meses. |
| **Cursor-based pagination** | — | ✅ Aprovado | Padrao para listas cronologicas (feed de atividade, audit trail, logs de IA). | `?cursor=xxx&limit=20`. Limite maximo 100 itens. |
| **Offset-based pagination** | — | ✅ Aprovado | Tabelas com ordenacao arbitraria (lista de usuarios, relatorios). | `?page=1&pageSize=20`. |
| **Idempotency key** | — | ✅ Aprovado | Protecao contra duplo submit (RN-133). UUID no header `Idempotency-Key`. | Obrigatorio em todas as acoes de mutacao: transicoes de estado, propostas, Fechamento, distribuicao escrow. |
| **HMAC-SHA256** | — | ✅ Aprovado | Validacao de webhooks de entrada (ZapSign RN-122, Celcoin escrow RN-126). | Ver ADR-002. Secret configurado em `Configuracoes`. Payload validado antes de processar. |

### 5.1 Padroes de Resposta

Herdados integralmente do ShiftLabs Stacks v7.0, secao 4.3. Resposta de sucesso: `{ success, data, meta }`. Resposta de erro: `{ success, error: { code, message, details } }`. Codigos HTTP obrigatorios: 400, 401, 403, 404, 409, 422, 429, 500.

### 5.2 Upload de Arquivos

Upload direto para Supabase Storage via signed URL gerada pelo backend. Nomenclatura UUID v4. Validacao de MIME type e tamanho no backend e frontend. Relevante para dossie (RN-002: 6 documentos obrigatorios), anuencia da construtora (RN-033), comprovantes de pagamento.

---

## 6. Repositorio e Codigo

| Tecnologia / Padrao | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **Turborepo** | 2.x+ | ✅ Aprovado | Monorepo com cache inteligente. Agente de IA ve o projeto inteiro. | Monorepo obrigatorio. Repos separados por camada proibidos. |
| **pnpm** | 9.x+ | ✅ Aprovado | Package manager para workspaces. Mais eficiente que npm/yarn. | Unico package manager. npm e yarn proibidos em novos projetos. |
| **GitHub** | — | ✅ Aprovado | Plataforma de repositorio. | `README.md`, `.env.example`, `.gitignore`, `turbo.json`, `docker-compose.yml` desde o primeiro commit. |
| **Conventional Commits** | — | ✅ Aprovado | Formato: `tipo(escopo): descricao`. | Tipos: feat, fix, refactor, docs, style, test, chore, perf, ci. |
| **ESLint** | latest | ✅ Aprovado | Linting obrigatorio em todo projeto. | Roda no pre-commit via Husky + lint-staged. |
| **Prettier** | latest | ✅ Aprovado | Formatacao automatica. | `semi: true`, `singleQuote: true`, `trailingComma: 'all'`, `printWidth: 100`. |
| **Husky + lint-staged** | latest | ✅ Aprovado | Pre-commit hooks para lint e formatting. | Obrigatorio em todo repo. |

### 6.1 Estrutura do Monorepo

Boilerplate Turborepo + pnpm pre-configurado (PEND-003 resolvido). Criar como pre-requisito antes do primeiro sprint de desenvolvimento. Inclui `docker-compose.yml` com Redis + RabbitMQ para ambiente local.

```
shiftlabs-repasse/
  apps/
    api/                    # NestJS (backend)
    web/                    # React + Vite (Admin dashboard)
    mobile/                 # Expo (mobile)
  packages/
    shared-types/           # Types TypeScript compartilhados
    shared-utils/           # Validacoes, formatadores, constantes
    shared-services/        # Wrappers de API, TanStack Query hooks
    config-eslint/          # Configuracao de lint compartilhada
    config-typescript/      # tsconfig base compartilhado
    ui/                     # Componentes shadcn/ui reutilizaveis entre apps web
  turbo.json
  package.json
  docker-compose.yml
  .env.example
  .gitignore
```

### 6.2 Estrutura de Pastas

Herdada integralmente do ShiftLabs Stacks v7.0: secao 5.2 (Backend NestJS), secao 5.4 (React + Vite), secao 5.5 (Expo + expo-router). Cada dominio de negocio do Admin (cases, proposals, dossier, escrow, users, notifications, ai-agents, reports, settings) e um modulo NestJS com controller, service, repository e DTOs.

### 6.3 Convencoes de Nomenclatura

- **Arquivos backend:** `kebab-case` (`case.service.ts`, `create-proposal.dto.ts`).
- **Arquivos frontend:** `PascalCase` para componentes (`PipelineBoard.tsx`), `camelCase` para hooks e utils (`useCase.ts`, `formatCurrency.ts`).
- **Variaveis e funcoes:** `camelCase`.
- **Classes e componentes:** `PascalCase`.
- **Constantes:** `UPPER_SNAKE_CASE`.
- **Tabelas e colunas (Prisma):** `snake_case` com `@@map` e `@map`.
- **Endpoints:** `kebab-case` (`/api/v1/ai-agents`, `/api/v1/audit-trail`).
- **Eventos de analytics:** `snake_case` com formato `[objeto]_[acao]` (`case_created`, `proposal_accepted`, `escrow_distributed`).

---

## 7. Testes

| Tecnologia | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **Vitest** | latest | ✅ Aprovado | Testes unitarios e de integracao no backend e frontend web. | Backend: logica de negocios, services, utils. Frontend: componentes criticos. |
| **Supertest** | latest | ✅ Aprovado | Testes de integracao de endpoints da API REST via NestJS Testing Module. | Endpoints criticos: auth, transicoes de estado, propostas, Fechamento, escrow. |
| **React Testing Library** | latest | ✅ Aprovado | Testes de componente web. | Screens criticas: Dashboard, Pipeline, Formalização, Triagem. |
| **Playwright** | latest | ✅ Aprovado | Testes end-to-end web. | 5 fluxos criticos: RN-072 a RN-076 (fluxo feliz, pendencia documental, bloqueio, cancelamento, disputa). |
| **Jest** | latest | ✅ Aprovado | Testes de componente mobile. | React Native Testing Library + Jest. |
| **Maestro** | latest | ✅ Aprovado | Testes E2E mobile (iOS e Android). | Fluxos criticos do app mobile. |

### 7.1 Cobertura Minima

- **Curto prazo (MVP):** Testes de integracao nos endpoints criticos (auth, transicoes de estado de caso, Fechamento, escrow, propostas).
- **Medio prazo:** Testes unitarios para toda logica de negocios (calculo de comissoes RN-018/019, validacao de propostas RN-028, distribuicao automatica RN-021). Testes de componente para screens criticas.
- **Longo prazo:** E2E para os 5 fluxos do Admin (RN-072 a RN-076) via Playwright (web) e Maestro (mobile).

### 7.2 O Que Testar

Checklist herdado do ShiftLabs Stacks v7.0, secao 6.4. Destaques criticos para o Admin:

- **Fechamento (RN-023):** Testar que botao so ativa com 4 criterios simultaneos. Testar cada combinacao de 3/4 criterios.
- **Comissoes (RN-018/019):** Testar calculo por cenario (A, B, C, D). Testar edge case de Delta zero no Cenario A.
- **Lock otimista (RN-130):** Testar HTTP 409 em edicao concorrente. Testar campos criticos que exigem aprovacao do Master.
- **Idempotency (RN-133):** Testar que duplo submit com mesma `idempotency_key` nao gera duplicata.
- **RBAC (RN-004):** Testar cada perfil contra cada endpoint. Analista nao acessa Financeiro. Gestor nao acessa Triagem.

---

## 8. Deploy e CI/CD

| Tecnologia / Padrao | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **Docker Engine** | 27+ | ✅ Aprovado | Ambiente de desenvolvimento local. | `Dockerfile` e `docker-compose.yml` obrigatorios. Redis e RabbitMQ via Docker local. |
| **Docker Compose** | 2.x+ | ✅ Aprovado | Orquestracao de servicos locais. | Inclui API, Redis, RabbitMQ. |
| **Supabase CLI** | latest | ✅ Aprovado | Instancia local de PostgreSQL + Auth + Storage + Realtime. | `supabase start` para ambiente local. Requer Docker. |
| **GitHub Actions** | — | ✅ Aprovado | CI/CD padrao. | Pipeline: lint → type-check → testes → `prisma migrate deploy` → build → deploy. Configurar no primeiro sprint de desenvolvimento (PEND-001 resolvido). |
| **EAS Build** | latest | ✅ Aprovado | Builds iOS e Android. | Perfis: development, preview, production. |
| **EAS Update** | latest | ✅ Aprovado | OTA updates de JavaScript. | Atualizacoes sem rebuild nativo. |

### 8.1 Ambientes Obrigatorios

- **Backend:** Desenvolvimento local (Docker + Supabase CLI) + Producao (**Railway**). Staging recomendado. `[DECISÃO AUTÔNOMA — Railway pela simplicidade operacional, Docker nativo e auto-deploy via GitHub. Alternativa descartada: Render (cold starts no plano gratuito), VPS (overhead operacional desnecessário para o estágio atual).]`
- **Frontend Web:** Desenvolvimento local (Vite dev server) + Producao (Vercel).
- **Mobile:** Development build (device local) + Producao (stores). Preview build recomendado.
- **Supabase:** Minimo 2 projetos (desenvolvimento + producao). Staging recomendado para o Admin por criticidade financeira.

### 8.2 Migrations em Producao

- `pnpm prisma migrate deploy` no pipeline de CI/CD antes do deploy do codigo.
- Rollback via nova migration (nunca editar migration aplicada).
- Migrations destrutivas em 2 etapas (deploy codigo primeiro, remove coluna depois).
- `CREATE INDEX CONCURRENTLY` para indices em tabelas grandes (audit trail, casos).

---

## 9. Seguranca

| Tecnologia / Padrao | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **HTTPS** | TLS 1.2+ | ✅ Aprovado | Obrigatorio em producao e staging. | Sem excecoes. HTTP proibido. |
| **CORS** | — | ✅ Aprovado | Configurado explicitamente no NestJS. | `Access-Control-Allow-Origin: *` proibido em producao. Dominio do Admin explicitamente listado. |
| **Helmet.js** | 8.x+ | ✅ Aprovado | Headers de seguranca automaticos. | Middleware desde o primeiro commit. |
| **@nestjs/throttler** | latest | ✅ Aprovado | Rate limiting em endpoints publicos e de auth. | Login: 5 tentativas / 30min bloqueio (RN-005). Propostas: 10/24h (RN-031). |
| **CSRF protection** | — | ✅ Aprovado | Obrigatorio para cookies httpOnly (refresh token web). | `@nestjs/csrf` ou middleware equivalente. |
| **class-validator pipes** | — | ✅ Aprovado | Sanitizacao de inputs no backend. | Toda entrada validada. Nunca confiar em validacao do cliente. |
| **Sentry** | 9.x+ | ✅ Aprovado | Error tracking em todas as camadas. | Deploy sem Sentry proibido. Source maps em producao. |
| **Supabase RLS** | — | ✅ Aprovado | Row Level Security habilitado em tabelas com dados de usuarios. | `service_role` key proibida no frontend/mobile. |
| **2FA (Autenticador)** | — | ✅ Aprovado | Obrigatorio para todos os operadores Admin (RN-005). | Via aplicativo autenticador (TOTP). |
| **Mascaramento de CPF/CNPJ** | — | ✅ Aprovado | No backend, nao apenas no frontend (DA-013). | Analista ve mascarado. Coordenador+ ve completo (RN-131). |
| **Idempotency key** | — | ✅ Aprovado | Protecao contra duplo submit (RN-133). | Header `Idempotency-Key` em toda acao de mutacao. |
| **Certificate pinning** | — | 🔶 Condicional | Recomendado no mobile por dados financeiros (Conta Escrow, comissoes). | Implementar se o parceiro financeiro exigir. |

### 9.1 LGPD

- Dados pessoais armazenados com base legal documentada (RN-137): identificacao, contato, financeiros, comportamentais, documentos.
- Direitos do titular atendidos em 15 dias corridos (RN-070, RN-137).
- Dados vinculados a casos ativos ou concluidos retidos por obrigacao legal (5 anos).
- Exportacao de usuarios sem CPF/CNPJ (RN-071).
- Consentimento e opt-out implementados conforme LGPD.
- Auditoria de solicitacoes LGPD com timer regressivo visivel para Master (RN-070).

### 9.2 Variaveis de Ambiente

Nomenclatura padrao: `[SERVICO]_[RECURSO]_[ATRIBUTO]` em `UPPER_SNAKE_CASE`. Variaveis obrigatorias para o Admin:

```
# Supabase
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_DB_URL=

# Auth
JWT_SECRET=
JWT_ACCESS_EXPIRATION=15m
JWT_REFRESH_EXPIRATION=8h

# Redis
REDIS_URL=

# RabbitMQ
RABBITMQ_URL=

# Observabilidade
SENTRY_DSN=
POSTHOG_API_KEY=
POSTHOG_HOST=

# Integracoes — ZapSign
ZAPSIGN_API_KEY=
ZAPSIGN_WEBHOOK_SECRET=

# Integracoes — Celcoin (Conta Escrow)
CELCOIN_API_KEY=
CELCOIN_WEBHOOK_SECRET=
CELCOIN_BASE_URL=

# Integracoes — WhatsApp (Meta Cloud API)
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_VERIFY_TOKEN=

# Integracoes — SMS (Twilio)
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=

# Deploy — Railway
RAILWAY_TOKEN=

# App
NODE_ENV=
PORT=
APP_URL=
ADMIN_CORS_ORIGIN=
```

---

## 10. Analytics e Tracking

| Tecnologia | Versao minima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **PostHog** | latest | ✅ Aprovado | Product analytics, session replay, feature flags, A/B testing. | Obrigatorio em todo ambiente com usuarios. PostHog Cloud como padrao inicial. Criar projeto na organizacao ShiftLabs, gerar API key, configurar wrapper centralizado em `src/services/analytics.ts`. Setup no boilerplate (PEND-002 resolvido). |
| **posthog-js** | 1.x+ | ✅ Aprovado | SDK web. | Wrapper centralizado em `src/services/analytics.ts`. |
| **posthog-react-native** | 3.x+ | ✅ Aprovado | SDK mobile. | Wrapper centralizado em `src/services/analytics.ts`. |
| **posthog-node** | latest | 🔶 Condicional | SDK backend para eventos server-side. | Usar para eventos que nao passam pelo frontend (distribuicao automatica, alertas de SLA). |

### 10.1 Eventos Minimos do Admin

| Evento | Descricao | Modulo |
|---|---|---|
| `screen_viewed` | Toda tela acessada. | Global |
| `login_completed` | Login bem-sucedido com 2FA. | Auth |
| `case_state_changed` | Toda transicao de estado do caso. | Pipeline |
| `proposal_submitted` | Proposta criada. | Negociacao |
| `proposal_accepted` | Aceite registrado. | Negociacao |
| `closing_confirmed` | Fechamento confirmado. | Formalizacao |
| `escrow_distributed` | Distribuicao automatica executada. | Financeiro |
| `escrow_reversed` | Estorno executado. | Financeiro |
| `ai_takeover_started` | Takeover de agente iniciado. | Supervisao IA |
| `report_exported` | Relatorio exportado. | Relatorios |
| `setting_changed` | Parametro de configuracao alterado. | Configuracoes |

### 10.2 Politica de Privacidade

- Nunca enviar para analytics: senhas, tokens, CPF, CNPJ, dados bancarios, dados de pagamento.
- `user_id` como unico identificador pessoal enviado ao PostHog.
- Consentimento LGPD implementado com opt-out.

---

## 11. Tecnologias Proibidas

| Tecnologia | Motivo da Proibicao | Alternativa Aprovada |
|---|---|---|
| **Next.js** (neste modulo) | Admin e 100% logado. Nao precisa de SSR/SSG/SEO. | React + Vite |
| **Express puro / Fastify / Hono / Koa** | NestJS e o framework HTTP unico aprovado. | NestJS |
| **TypeORM / Sequelize / Drizzle** | Prisma e o unico ORM aprovado. | Prisma |
| **MongoDB / DynamoDB / NoSQL** | PostgreSQL via Supabase e o unico banco aprovado. | PostgreSQL (Supabase) |
| **MySQL / MariaDB / SQLite (producao)** | PostgreSQL via Supabase e o unico banco relacional. | PostgreSQL (Supabase) |
| **PostgreSQL auto-gerenciado (Docker em producao)** | Usar Supabase como plataforma gerenciada. | Supabase |
| **Axios** | `fetch` nativo e o cliente HTTP padrao. | fetch nativo |
| **react-router-dom** (sem ADR) | TanStack Router e o roteador padrao para React+Vite. | TanStack Router |
| **Redux / MobX / Context API (state manager)** | Zustand + TanStack Query cobrem todos os casos. | Zustand + TanStack Query |
| **MUI / Chakra / AntD / Bootstrap** | shadcn/ui e o unico sistema de componentes (ownership). | shadcn/ui |
| **UnoCSS / Windi CSS** | Tailwind CSS e o unico framework de utilidades CSS. | Tailwind CSS v4 |
| **Moment.js** | Deprecated. Pesado. | date-fns + date-fns-tz |
| **Day.js / Luxon** | date-fns e o padrao. Tree-shakeable. | date-fns |
| **GSAP** (substituto de Framer Motion) | Framer Motion e a biblioteca de animacao obrigatoria. GSAP como complemento via ADR. | Framer Motion |
| **Three.js / WebGL** | Reservado para landing pages/marketing. Proibido no Admin. | — |
| **GraphQL** | API REST e o padrao. | API REST (JSON) |
| **gRPC** | API REST e o padrao. | API REST (JSON) |
| **Flutter / Kotlin Multiplatform / Swift / Kotlin nativo** | React Native + Expo e o framework mobile unico. | React Native + Expo |
| **React Navigation** | expo-router e o roteador mobile unico. | expo-router |
| **React Native Paper / NativeBase / Tamagui** | Componentes UI construidos do zero com primitivas nativas. | StyleSheet.create() nativo |
| **NativeWind / Styled Components (mobile)** | StyleSheet.create() nativo e o padrao de estilizacao mobile. | StyleSheet.create() |
| **OneSignal / Pusher** | Expo Notifications e o padrao para push. | Expo Notifications |
| **npm / yarn** | pnpm e o unico package manager aprovado. | pnpm |
| **Spinners genericos como loading state** | Skeleton screens obrigatorios. | Skeleton screens |
| **`console.log` em producao** | Backend: Pino. Frontend: removidos no build. Mobile: `babel-plugin-transform-remove-console`. | Pino (backend) / Sentry breadcrumbs |
| **Arquivos `.env` commitados** | `.env` no `.gitignore`. `.env.example` com chaves sem valores. | Variaveis de ambiente |
| **`dangerouslySetInnerHTML` sem sanitizacao** | XSS. Usar DOMPurify se necessario. | DOMPurify |
| **`$queryRaw` sem parametros preparados** | SQL Injection. | `$queryRaw` com parametros preparados |

---

## 12. ADRs — Desvios do Baseline

### ADR-001: Supabase Realtime para Pipeline em Tempo Real

1. **Contexto:** RN-151 exige que o Pipeline atualize em ate 5 segundos para todos os operadores logados. RN-087 exige monitoramento de Contas Escrow em tempo real. RN-093 exige refresh de 10 segundos na Supervisao IA. O ShiftLabs Stacks v7.0 lista Supabase Realtime como "disponivel com justificativa documentada por produto".
2. **Decisao:** Supabase Realtime e aprovado como tecnologia primaria de tempo real do Admin para subscriptions de dados em tabelas de casos, Contas Escrow e acoes de agentes de IA. Fallback para polling com intervalo de 5 segundos se WebSocket estiver indisponivel.
3. **Alternativas descartadas:** (a) Polling puro com intervalo de 5s — descartado por gerar carga excessiva no backend com multiplos operadores simultaneos. (b) WebSocket customizado — descartado porque Supabase Realtime ja esta na infraestrutura e cobre o caso de uso sem codigo adicional.
4. **Consequencias:** Dependencia de Supabase Realtime para funcionalidade critica. Se Supabase Realtime ficar indisponivel, o fallback para polling mantem a operacao funcional com latencia de 5 a 15 segundos. Custo de Supabase aumenta conforme numero de conexoes simultaneas.

### ADR-002: HMAC-SHA256 para Validacao de Webhooks

1. **Contexto:** RN-122 (ZapSign) e RN-126 (parceiro escrow) exigem validacao de webhooks de entrada para prevenir payloads maliciosos. O ShiftLabs Stacks v7.0 nao detalha padrao de validacao de webhooks de entrada, apenas menciona SSE/Supabase Realtime/WebSocket para comunicacao.
2. **Decisao:** Todo webhook de entrada (ZapSign, parceiro escrow) deve ser validado via HMAC-SHA256. O secret e configurado em `Configuracoes > [Integracao]` pelo Master. Payload com assinatura invalida e descartado e registrado no audit trail.
3. **Alternativas descartadas:** (a) Certificado mutuo TLS — descartado por complexidade de configuracao e porque ZapSign usa HMAC nativamente. (b) IP whitelist — descartado por nao garantir integridade do payload.
4. **Consequencias:** Exige que cada parceiro de integracao suporte HMAC-SHA256. ZapSign suporta nativamente. Celcoin suporta webhooks com HMAC-SHA256 nativamente (DP-005 resolvido). `[DECISÃO AUTÔNOMA — Celcoin escolhida por regulação BCB, APIs REST modernas e escrow digital nativo. Alternativa descartada: Transfeera (foco em pagamentos, não escrow).]`

### ADR-003: Audit Trail em Schema Separado (Append-Only)

1. **Contexto:** RN-129 exige registros de auditoria imutaveis com retencao de 5 anos. RN-150 exige auditoria financeira imutavel. DA-012 (decisao autonoma) define que o storage de auditoria deve ser separado do banco principal.
2. **Decisao:** O audit trail e armazenado em schema PostgreSQL dedicado (ex: `audit`) dentro do mesmo Supabase, com tabela append-only. Nenhuma operacao de UPDATE ou DELETE e permitida nas tabelas de auditoria. O schema de auditoria usa usuario de banco com permissao apenas de INSERT e SELECT.
3. **Alternativas descartadas:** (a) Servico de auditoria independente (Datadog, ELK) — descartado por adicionar infraestrutura externa e aumentar custo sem beneficio proporcional no MVP. (b) Mesma tabela no schema principal — descartado por risco de falha no banco principal afetar integridade da auditoria.
4. **Consequencias:** Aumento de storage no Supabase. Queries de auditoria nao competem com queries operacionais. Backup e retencao de 5 anos devem ser configurados especificamente para o schema de auditoria.

---

## 13. Changelog

| Versao | Data | Alteracao |
|---|---|---|
| v1.1 | 2026-03-22 | Resolucao de todas as pendencias: parceiro escrow (Celcoin), canais de notificacao (WhatsApp via Meta Cloud API + SMS via Twilio), infraestrutura de deploy (Railway/Upstash/CloudAMQP), CI/CD pipeline definido (PEND-001), PostHog Cloud configurado (PEND-002), boilerplate monorepo definido (PEND-003). |
| v1.0 | 2026-03-22 | Criacao do documento. 47 tecnologias aprovadas, 28 proibidas, 3 ADRs, 2 definicoes pendentes. |

---

## 14. Backlog de Pendencias

> **Todas as pendencias foram resolvidas na v1.1.**

| ID | Descricao | Status | Resolucao |
|---|---|---|---|
| **DP-005** | Parceiro financeiro para Conta Escrow. | ✅ Resolvido | Celcoin — fintech regulada BCB, APIs REST modernas, escrow digital nativo. Ver secao 2.2. |
| **DP-003** | Canais de notificacao SMS e WhatsApp. | ✅ Resolvido | WhatsApp via Meta Cloud API + SMS via Twilio no MVP. Ver secao 2.2. |
| **PEND-001** | Pipeline completo de GitHub Actions. | ✅ Resolvido | Pipeline: lint → type-check → testes → prisma migrate deploy → build → deploy. Prazo: primeiro sprint de desenvolvimento. |
| **PEND-002** | Configurar projeto PostHog e wrapper de analytics. | ✅ Resolvido | PostHog Cloud na org ShiftLabs. Wrapper em `src/services/analytics.ts`. Prazo: setup do boilerplate. |
| **PEND-003** | Boilerplate oficial do monorepo. | ✅ Resolvido | Turborepo + pnpm com `apps/api`, `apps/web`, `packages/shared-types`, `packages/shared-utils`, `packages/shared-services`, `packages/config-eslint`, `packages/config-typescript`. docker-compose.yml com Redis + RabbitMQ. Prazo: pre-requisito antes do primeiro sprint. |
