# Stacks Tecnologicas — Modulo Cedente

## Padrao Tecnico Normativo

| Campo | Valor |
|---|---|
| Destinatario | Tech Lead / CTO |
| Escopo | Stack normativa do modulo Cedente — Repasse Seguro |
| Modulo | Cedente |
| Versao | v1.1 |
| Responsavel | Claude Code Desktop |
| Data da versao | 2026-03-22 (America/Fortaleza) |

> **Carater normativo.** Este documento define as tecnologias obrigatorias para o modulo Cedente do Repasse Seguro. A fonte primaria de verdade e o ShiftLabs Stacks v7.0. Desvios sao permitidos exclusivamente quando uma regra de negocio do Cedente (RN-001 a RN-092) justificar necessidade tecnica incompativel com o padrao central, documentados via ADR neste mesmo documento.

---

**TL;DR**

- **Backend:** Node.js 22+ / NestJS 10+ / Prisma 6+ / Supabase (PostgreSQL 17+) / Redis (Upstash) / RabbitMQ (CloudAMQP) / Resend (e-mail transacional) — padrao ShiftLabs v7.0 sem desvio. Deploy em Railway.
- **Frontend Web:** Next.js 15+ (App Router) com rotas publicas (landing, cadastro, onboarding) e area logada (dashboard, propostas, documentos, financeiro) no mesmo app. Deploy em Vercel.
- **Mobile:** React Native 0.76+ / Expo SDK 52+ / expo-router 4+ — o Cedente acessa via mobile para upload de documentos por camera, assinatura ZapSign touch e resposta a propostas (RN-087).
- **IA no produto:** GPT-4 via OpenAI SDK + Vercel AI SDK + LangChain.js para o Guardiao do Retorno (RN-058 a RN-063). Langfuse para observabilidade. Supabase pgvector para RAG.
- **Comunicacao:** API REST unica para web e mobile. SSE para streaming do Guardiao do Retorno. Supabase Realtime para notificacoes no painel (RN-057: badge atualizado em ate 30 segundos).
- **Seguranca:** JWT + Supabase Auth. LGPD obrigatoria (RN-010). Isolamento total de dados entre Cedentes (RN-011). Anonimato do Cessionario (RN-012).
- **Analytics:** PostHog obrigatorio desde o lancamento. Sentry obrigatorio em todas as camadas.

---

## 1. Principios de Governanca

**PG-01. ShiftLabs Stacks v7.0 e a lei.** Toda decisao de stack deste documento herda do padrao central. Este documento nao repete o que o padrao central ja define — apenas referencia, especializa ou desvia com justificativa.

**PG-02. RN sobrescreve padrao central somente com ADR.** Se uma regra de negocio do Cedente exigir tecnologia nao prevista no ShiftLabs Stacks v7.0, o desvio e documentado como ADR neste documento com contexto, decisao, alternativas descartadas e consequencias.

**PG-03. Desenvolvimento 100% por IA.** Todo codigo do modulo Cedente e gerado por agentes de IA (Claude Code e GPT Codex). Boilerplates, convencoes de codigo, estrutura de pastas e testes automatizados sao os principais contratos de qualidade.

**PG-04. Monorepo unico por produto.** O Repasse Seguro opera como monorepo Turborepo + pnpm. O modulo Cedente e um conjunto de apps e packages dentro desse monorepo: `apps/web-cedente/` (Next.js), `apps/mobile-cedente/` (Expo) e modulos NestJS dentro de `apps/api/`.

**PG-05. Zero ambiguidade em pendencias.** Itens sem definicao suficiente sao marcados `[DEFINICAO PENDENTE]` com opcoes e impacto. Itens decididos autonomamente sao marcados `[DECISAO AUTONOMA]` com justificativa.

---

## 2. Backend

O backend do modulo Cedente segue integralmente o ShiftLabs Stacks v7.0 secao 1. Nenhum desvio identificado.

### 2.1 Stack Obrigatoria

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| Node.js | 22.x+ (LTS) | ✅ Obrigatorio | Runtime padrao ShiftLabs. | Todo servico backend roda em Node.js LTS. |
| NestJS | 10.x+ | ✅ Obrigatorio | Arquitetura modular, decorators, Swagger embutido, guards/pipes nativos. Output de IA mais consistente. | Framework unico para HTTP. Express puro, Fastify, Hono e Koa sao proibidos sem ADR. |
| TypeScript | 5.4+ | ✅ Obrigatorio | Tipagem forte, strict mode. | `strict: true` obrigatorio em todo codigo backend. Arquivos `.js` proibidos. |
| Prisma | 6.x+ | ✅ Obrigatorio | Unico ORM aprovado. Tipagem auto-gerada, migrations declarativas, integracao Supabase. | TypeORM, Sequelize, Drizzle e qualquer outro ORM sao proibidos. |
| Pino | 9.x+ | ✅ Obrigatorio | Logs estruturados. | Todo servico deve ter logs com request ID, timestamp, nivel e contexto. |
| Helmet | 8.x+ | ✅ Obrigatorio | Headers de seguranca. | Middleware adicionado desde o primeiro commit. |
| class-validator + class-transformer | latest | ✅ Obrigatorio | Validacao de DTOs. | Toda entrada de dados validada via pipes NestJS. |
| @nestjs/swagger | latest | ✅ Obrigatorio | Documentacao auto-gerada. | Todo endpoint documentado via Swagger/OpenAPI. |

### 2.2 Banco de Dados

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| Supabase | latest | ✅ Obrigatorio | PostgreSQL gerenciado + Auth + Storage + Realtime + Edge Functions + pgvector. Elimina infra operacional. | Plataforma unica para banco, auth, storage e realtime. |
| PostgreSQL (via Supabase) | 17+ | ✅ Obrigatorio | Unico banco relacional aprovado. | MySQL, MariaDB, SQLite em producao sao proibidos. |
| Supabase Auth | latest | ✅ Obrigatorio | RN-001 a RN-006 exigem cadastro, ativacao por e-mail, recuperacao de senha, bloqueio por tentativas e expiracao de sessao. Supabase Auth cobre esses fluxos nativamente em complemento ao JWT. | Camada de autenticacao do Cedente. |
| Supabase Storage | latest | ✅ Obrigatorio | RN-041/RN-042 exigem upload de documentos (PDF, JPG, PNG, max 10 MB) com validacao de MIME type. | Upload via signed URL. Validacao de tipo MIME no backend obrigatoria (RN-042). |
| Supabase Realtime | latest | ✅ Obrigatorio | RN-057 exige notificacoes no painel em ate 30 segundos. RN-014 exige atualizacao do Dashboard a cada 60 segundos ou tempo real. | Subscriptions de dados para notificacoes e atualizacao de status. |

**Convencoes de Prisma Schema:** Aplicar integralmente as convencoes do ShiftLabs Stacks v7.0 secao 1.5 — UUID v4 como PK, colunas de auditoria obrigatorias, soft delete para tabelas de dominio, snake_case, timestamps com timezone.

### 2.3 Cache e Filas

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| Redis | 7.4+ | ✅ Obrigatorio | Cache de dados criticos. | TTL definido e estrategia de invalidacao documentada para todo uso. Ambiente local via Docker. Producao em Upstash (serverless, pay-per-use). |
| RabbitMQ | 4.x+ | ✅ Obrigatorio | Processamento assincrono. RN-056 exige envio de e-mail em ate 5 minutos. RN-046 exige lembretes periodicos. RN-023 exige descarte de rascunhos apos 30 dias. | Toda fila com retry configurado e dead-letter queue. Ambiente local via Docker. Producao em CloudAMQP (gerenciado). |

### 2.4 E-mail Transacional

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| Resend | latest | ✅ Obrigatorio | `[DECISAO AUTONOMA — Resend. SDK TypeScript nativo, DX excelente. Descartado: SendGrid (SDK pesado, DX inferior), Amazon SES (over-engineering para o estagio).]` | Provedor unico para e-mails transacionais (ativacao de conta, notificacoes, lembretes — RN-056). SDK TypeScript nativo. Templates em React Email. |

### 2.5 Modulo IA — Guardiao do Retorno

O Guardiao do Retorno (RN-058 a RN-063) e um agente de IA embarcado no produto. Segue integralmente o ShiftLabs Stacks v7.0 secao 11.

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| GPT-4 (OpenAI) | gpt-4-turbo (versao fixada em producao) | ✅ Obrigatorio | Modelo padrao para IA no produto. Melhor equilibrio qualidade/function calling. | Versao fixada em producao. Aliases genericos (`gpt-4`) apenas em desenvolvimento. |
| OpenAI SDK | 4.x+ | ✅ Obrigatorio | Cliente padrao para API OpenAI. | Chamadas ao LLM exclusivamente via backend NestJS. Frontend/mobile nunca chama a API diretamente. |
| Vercel AI SDK | 4.x+ | ✅ Obrigatorio | Streaming de respostas do Guardiao via SSE no frontend Next.js. | Integracao nativa com App Router para chat streaming. |
| LangChain.js | 0.3.x+ | ✅ Obrigatorio | Pipeline RAG para base de conhecimento do Guardiao (FAQ, processo, Lei 13.786). | Chains, retrieval, memory management. |
| LangGraph.js | 0.2.x+ | 🔶 Aprovado | Agentes com fluxo stateful (cadastro assistido com multi-step — RN-089). | Usar quando o Guardiao precisar de fluxo de decisao com checkpoints e branching. |
| Supabase pgvector | 0.7+ | ✅ Obrigatorio | Vector store para RAG. Embeddings no mesmo PostgreSQL. | Indice hnsw como padrao. Queries hibridas (vetorial + SQL). |
| OpenAI Embeddings | text-embedding-3-small | ✅ Obrigatorio | Embeddings para pipeline RAG. | `text-embedding-3-large` via ADR se precisao insuficiente. |
| Langfuse | 3.x+ | ✅ Obrigatorio | Observabilidade de LLM: tracing, custo, latencia, qualidade. | Monitorar latencia, custo, erros e qualidade. Alertas obrigatorios conforme ShiftLabs v7.0 secao 11.8.3. |

**Restricoes do Guardiao derivadas das RNs:**
- RN-058: Acesso ao contexto dos casos do Cedente logado. Nunca acessa dados de outros Cedentes ou Cessionarios.
- RN-060: Proibido garantir resultados financeiros ou prazos. Usar termos de estimativa.
- RN-061: Escalacao para humano quando confianca insuficiente. Criar solicitacao no Admin.
- RN-063: Supervisao e takeover pelo Admin. Registrar nivel de confianca de cada resposta.
- RN-062: Historico de conversas permanente e imutavel.

**Prompts como codigo:** Armazenados em `src/modules/ai/prompts/guardiao/`. System prompt, user prompt e tools em arquivos TypeScript tipados. Versionados via Git.

---

## 3. Frontend Web

### 3.1 Criterio de Decisao: Next.js (App Router)

O modulo Cedente tem face publica (landing page, cadastro, onboarding) que exige SEO e meta tags para compartilhamento social, e area logada (dashboard, propostas, documentos, financeiro, assistente IA, perfil). Conforme ShiftLabs Stacks v7.0 secao 2.1: produto com face publica usa Next.js (App Router) + Vercel.

**[DECISAO AUTONOMA]** O modulo Cedente usa um unico app Next.js com App Router, dividido em route groups: `(public)/` para rotas publicas e `(authenticated)/` para area logada. Justificativa: evita manter dois apps separados (Next.js + React+Vite) para um unico modulo; o App Router suporta ambos os cenarios nativamente com Server Components para SEO e Client Components para interatividade. Alternativa descartada: React+Vite para a area logada — adicionaria complexidade de deploy, duplicacao de configuracao e fragmentacao de shared state.

### 3.2 Stack Obrigatoria

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| Next.js | 15.x+ | ✅ Obrigatorio | App Router com Server Components para landing/SEO e Client Components para dashboard/interatividade. | Deploy em Vercel. File-based routing. Route groups `(public)` e `(authenticated)`. |
| React | 19.x+ | ✅ Obrigatorio | Biblioteca de UI padrao. | Componentes funcionais exclusivamente. Class components proibidos. |
| TypeScript | 5.4+ | ✅ Obrigatorio | Tipagem forte. | `strict: true` obrigatorio. Arquivos `.js`/`.jsx` proibidos. |
| Tailwind CSS | 4.x+ | ✅ Obrigatorio | Framework de estilizacao padrao. | Design tokens via `@theme` no `global.css`. |
| shadcn/ui | latest | ✅ Obrigatorio | Sistema de componentes com ownership total do codigo. Baseado em Radix Primitives com acessibilidade completa. | Componentes copiados para o repositorio. MUI, Chakra, AntD, Bootstrap proibidos. |
| Radix Primitives | latest | ✅ Obrigatorio | Acessibilidade embutida (keyboard nav, ARIA, focus management). | Base do shadcn/ui. |
| TanStack Query | 5.x+ | ✅ Obrigatorio | Fetching, caching e sincronizacao de dados do servidor. | Nunca usar `fetch` direto em `useEffect` sem cache. Retry 3x para erros 5xx. |
| Zustand | 5.x+ | ✅ Obrigatorio | Estado global. RN-006 exige preservacao de rascunhos em formularios. | Redux, MobX, Context API como state manager global proibidos. |
| Framer Motion (Motion) | 12.x+ | ✅ Obrigatorio | Animacoes e transicoes. RN-021 exige animacao de desbloqueio do botao avancar. RN-022 exige expansao animada do card de cenario. RN-040 exige transicao suave do botao "Solicitar Desistencia". | AnimatePresence para transicoes de tela. Skeleton screens para loading (spinners genericos proibidos). |
| Sentry (`@sentry/nextjs`) | 9.x+ | ✅ Obrigatorio | Error tracking. | Source maps em producao. Performance monitoring habilitado. |
| PostHog (`posthog-js`) | 1.x+ | ✅ Obrigatorio | Analytics + session replay + feature flags. | Wrapper centralizado em `src/services/analytics.ts`. Eventos em snake_case. |
| date-fns | 4.x+ | ✅ Obrigatorio | Manipulacao de datas. | Locale pt-BR. Nunca Moment.js, Day.js ou Luxon. |
| date-fns-tz | 3.x+ | ✅ Obrigatorio | Conversao de timezone na camada de apresentacao. | Backend envia UTC. Frontend converte para timezone do usuario. |

### 3.3 Design Tokens

Aplicar integralmente o sistema de design tokens do ShiftLabs Stacks v7.0 secao 2.3.1: cores (compativel com shadcn/ui), espacamento (base 4px), tipografia (Inter + JetBrains Mono), border radius, sombras e motion tokens (duracao, easing, escala, opacidade).

Valores concretos de cores definidos pelo Brand Guide do Cedente (`03 - Brand Guide.md`).

### 3.4 Motion — Requisitos Especificos do Cedente

Alem das diretrizes obrigatorias do ShiftLabs v7.0 secao 2.4, o modulo Cedente exige:

- **Wizard de cadastro (5 etapas):** transicoes entre etapas via AnimatePresence. Barra de progresso animada. Timer de 10 segundos com barra de progresso fina no simulador (RN-021).
- **Cards de cenario:** expansao animada ao selecionar (RN-022). Checkbox com destaque pulsante (RN-022).
- **Timer regressivo de propostas:** atualizacao a cada minuto com pulsacao do badge vermelho nas ultimas 24 horas (RN-030).
- **Skeleton loading:** obrigatorio no simulador de cenarios (RN-021), no Dashboard (RN-013) e na lista de propostas (RN-036).
- **Botao "Solicitar Desistencia":** desaparecimento com transicao suave ao final do periodo de reversao (RN-040).

### 3.5 Acessibilidade (a11y)

Aplicar integralmente as diretrizes do ShiftLabs v7.0 secao 2.5. Requisitos adicionais do Cedente:

- RN-020: Abas do detalhe do caso navegaveis por teclado (setas esquerda/direita) com `role="tablist"`, `role="tab"`, `role="tabpanel"`, `aria-selected`.
- RN-041: Cada documento do dossie com `aria-label` descritivo: "[Nome do documento] — status: [status]".
- RN-057: Badge de notificacoes com contraste WCAG AA (4.5:1) e `aria-label` no sino: "Notificacoes — [X] nao lidas".
- Tamanho minimo de toque 44x44px em interfaces touch (RN-087).

### 3.6 Performance

Metas do ShiftLabs v7.0 secao 2.7: LCP < 2.5s, CLS < 0.1, INP < 200ms. Bundle JS gzipped < 200KB. Code splitting automatico por rota (App Router). Lazy loading de modais, drawers e graficos. Imagens em WebP com `next/image`.

### 3.7 Internacionalizacao (i18n)

Nao adotada neste momento. Produto opera em portugues (pt-BR). Se necessario no futuro: i18next + react-i18next conforme ShiftLabs v7.0 secao 2.6.

---

## 4. Mobile

### 4.1 Justificativa para App Mobile

**[DECISAO AUTONOMA]** O modulo Cedente requer app mobile nativo (React Native + Expo). Justificativa: RN-087 define 3 acoes criticas no mobile — (1) upload de documento via camera do celular, (2) assinatura ZapSign touch e (3) resposta a proposta. O web responsivo cobre o MVP, mas a experiencia de camera nativa, push notifications e SecureStore exigem app dedicado. Alternativa descartada: apenas web responsivo — a experiencia de upload por camera e assinatura touch e significativamente inferior no browser mobile.

### 4.2 Stack Obrigatoria

| Tecnologia | Versao Minima | Status | Justificativa | Regra de Uso |
|---|---|---|---|---|
| React Native | 0.76+ | ✅ Obrigatorio | Framework unico para mobile. | Flutter, Kotlin Multiplatform e nativo puro proibidos sem ADR. |
| Expo (managed workflow) | SDK 52+ | ✅ Obrigatorio | Build simplificado, OTA updates, APIs nativas. | Eject para bare workflow apenas com justificativa e aprovacao. |
| expo-router | 4.x+ | ✅ Obrigatorio | File-based routing. Deep linking nativo. | React Navigation proibido. Typed routes habilitado. |
| TypeScript | 5.4+ | ✅ Obrigatorio | Tipagem forte. | `strict: true` obrigatorio. |
| React Native Reanimated | 3.x+ | ✅ Obrigatorio | Animacoes performantes na UI thread. | Animar com `setState` ou `Animated` API legacy proibido. |
| React Native Gesture Handler | 2.x+ | ✅ Obrigatorio | Interacoes tateis (swipe, drag, pinch). | Obrigatorio para experiencia de assinatura touch (RN-087). |
| TanStack Query | 5.x+ | ✅ Obrigatorio | Fetching, caching e sincronizacao. Mesmo padrao do web. | `fetch` nativo como cliente HTTP. |
| Zustand | 5.x+ | ✅ Obrigatorio | Estado global. Mesmo padrao do web. | Middleware `persist` com AsyncStorage para estado nao sensivel. |
| expo-secure-store | latest | ✅ Obrigatorio | Armazenamento seguro de tokens. | Refresh token obrigatoriamente em SecureStore. Nunca em AsyncStorage. |
| expo-image | latest | ✅ Obrigatorio | Carregamento otimizado de imagens com cache. | Substitui `<Image>` nativo para imagens remotas. |
| expo-camera | latest | ✅ Obrigatorio | Captura de documentos via camera (RN-087). | Upload aceita orientacao retrato e paisagem (RN-087). |
| Expo Notifications | latest | ✅ Obrigatorio | Push notifications. RN-056/RN-057 exigem notificacoes em tempo real. | Solicitar permissao de forma contextual, nunca no primeiro acesso. |
| Sentry (`@sentry/react-native`) | 6.x+ | ✅ Obrigatorio | Error tracking mobile. | Testar com VoiceOver (iOS) e TalkBack (Android) antes de cada release. |
| PostHog (`posthog-react-native`) | 3.x+ | ✅ Obrigatorio | Analytics mobile. | Mesmo wrapper centralizado e mesmos nomes de eventos do web. |
| date-fns | 4.x+ | ✅ Obrigatorio | Manipulacao de datas. | Locale pt-BR. |
| date-fns-tz | 3.x+ | ✅ Obrigatorio | Conversao de timezone. | Mesmo padrao do web. |

### 4.3 UI e Estilizacao

- Componentes construidos do zero com primitivas React Native (`View`, `Text`, `Pressable`, `FlatList`).
- `StyleSheet.create()` nativo. NativeWind, Styled Components proibidos sem aprovacao.
- Design tokens replicados como constantes TypeScript em `src/constants/tokens.ts`. Nomes identicos ao web (em camelCase).
- React Native Paper, NativeBase, Tamagui proibidos.

### 4.4 Build e Distribuicao

- **EAS Build** para gerar builds iOS e Android.
- **EAS Update** para OTA updates de JavaScript.
- **Perfis:** `development` (simulador/device), `preview` (distribuicao interna), `production` (stores).

---

## 5. Comunicacao e API

Segue integralmente o ShiftLabs Stacks v7.0 secao 4. Especializacoes para o Cedente:

### 5.1 Protocolo e Cliente HTTP

| Tecnologia | Status | Justificativa | Regra de Uso |
|---|---|---|---|
| API REST (JSON) | ✅ Obrigatorio | Protocolo padrao. | API unica para web e mobile. Header `X-Platform: web\|ios\|android` quando necessario. |
| `fetch` nativo | ✅ Obrigatorio | Cliente HTTP padrao (browser e React Native). | Axios proibido sem aprovacao. Wrapper com interceptor de auth em `services/api.ts`. |
| Swagger/OpenAPI | ✅ Obrigatorio | Documentacao auto-gerada via `@nestjs/swagger`. | Todo endpoint documentado. |
| GraphQL, gRPC | ❌ Proibido | Nao aprovados como padrao. | Uso somente via ADR com justificativa. |

### 5.2 Comunicacao em Tempo Real

| Cenario | Tecnologia | Justificativa |
|---|---|---|
| Streaming de respostas do Guardiao do Retorno (RN-058) | **SSE (Server-Sent Events)** | Comunicacao unidirecional: backend envia tokens incrementalmente ao frontend. Padrao para toda feature de IA com streaming. |
| Notificacoes no painel em ate 30 segundos (RN-057), atualizacao do Dashboard a cada 60 segundos (RN-014), status da Conta Escrow (RN-083) | **Supabase Realtime** | Subscriptions de dados em tabelas do banco. Aproveita infraestrutura Supabase existente. |
| WebSocket bidirecional | ❌ Via ADR | Somente se SSE e Supabase Realtime nao cobrirem o caso de uso. |

### 5.3 Autenticacao e Autorizacao

| Tecnologia | Status | Justificativa | Regra de Uso |
|---|---|---|---|
| JWT | ✅ Obrigatorio | Stateless, funciona igual web e mobile. | Access token: 15-30 min, em memoria (Zustand). Refresh token (web): httpOnly cookie, 7-30 dias. Refresh token (mobile): expo-secure-store. |
| Supabase Auth | ✅ Obrigatorio | RN-001 a RN-006 exigem fluxos de auth complexos (ativacao por e-mail, bloqueio por tentativas, expiracao de sessao). | Em complemento ao JWT. Row Level Security habilitado e configurado em todas as tabelas com dados de Cedente (RN-011). |
| NestJS Guards | ✅ Obrigatorio | Controle de acesso por role (Cedente PF, Cedente PJ, Admin). | RBAC conforme Matriz de Permissoes (01.1 secao 4.2). |

**Fluxo de auth end-to-end:** Conforme ShiftLabs v7.0 secao 4.4.1. Interceptor no fetch wrapper: adiciona access token, intercepta 401, tenta refresh, reenvia requisicao original, redireciona para login se refresh falhar.

### 5.4 Upload de Arquivos

- **Plataforma:** Supabase Storage.
- **Estrategia:** Upload direto via signed URL gerada pelo backend. Backend valida MIME type (whitelist: `application/pdf`, `image/jpeg`, `image/png`) e tamanho (max 10 MB por arquivo — RN-042) antes de gerar a signed URL.
- **Nomenclatura:** UUID v4 como nome do arquivo. Preservar extensao original.
- **Organizacao:** `documents/repasse/cedente/[caso_id]/[uuid].[ext]`.
- **Validacao de MIME type real:** Obrigatoria no backend (RN-042). Nao confiar apenas na extensao declarada.

### 5.5 Padroes de Resposta, Paginacao, Filtros e Data/Hora

Aplicar integralmente ShiftLabs v7.0 secoes 4.3, 4.6, 4.10, 4.11. Destaques para o Cedente:

- **Paginacao de Meus Casos:** Offset-based com 10 itens por pagina (RN-017). Filtros por status e cenario. Busca por nome do imovel.
- **Paginacao de propostas:** Cursor-based, ordenacao por valor (maior primeiro — RN-036).
- **Timezone:** Backend e banco em UTC. Frontend converte para timezone do usuario na camada de apresentacao. Locale pt-BR para formatacao.

---

## 6. Repositorio e Codigo

Segue integralmente o ShiftLabs Stacks v7.0 secao 5.

### 6.1 Estrutura do Monorepo

```
shiftlabs-repasse/
  apps/
    api/                        # NestJS (backend — todos os modulos)
    web-cedente/                # Next.js (frontend web do Cedente)
    mobile-cedente/             # Expo (mobile do Cedente)
  packages/
    shared-types/               # Types TypeScript compartilhados
    shared-utils/               # Funcoes utilitarias, validacoes, constantes
    shared-services/            # Wrappers de API, TanStack Query hooks
    config-eslint/              # Configuracao de lint compartilhada
    config-typescript/          # tsconfig base compartilhado
    ui/                         # Componentes shadcn/ui customizados reutilizaveis (web)
  turbo.json
  package.json
  pnpm-workspace.yaml
  docker-compose.yml            # Redis, RabbitMQ, Supabase CLI local
  .env.example
  .gitignore
```

### 6.2 Infraestrutura de Monorepo

| Tecnologia | Versao Minima | Status | Regra de Uso |
|---|---|---|---|
| Turborepo | 2.x+ | ✅ Obrigatorio | Pipelines para build, lint, type-check, test, dev. Cache local habilitado. Remote Cache via Vercel recomendado. |
| pnpm | 9.x+ | ✅ Obrigatorio | Package manager unico. npm e yarn proibidos em novos projetos. |
| Docker Engine | 27+ | ✅ Obrigatorio | Ambiente de desenvolvimento local. |
| Docker Compose | 2.x+ | ✅ Obrigatorio | Orquestracao de servicos locais (Redis, RabbitMQ). |

### 6.3 Padroes de Codigo

Aplicar integralmente ShiftLabs v7.0 secoes 5.6, 5.7, 5.8:

- **Linting:** ESLint obrigatorio. Prettier obrigatorio (`semi: true`, `singleQuote: true`, `trailingComma: 'all'`, `printWidth: 100`). Husky + lint-staged para pre-commit.
- **Commits:** Conventional Commits obrigatorio. Formato: `tipo(escopo): descricao`.
- **Nomenclatura:** Backend kebab-case. Frontend/Mobile PascalCase para componentes, camelCase para hooks/utils. Banco snake_case. API kebab-case.
- **Code Review:** Todo codigo via pull request. Pipeline de CI e a principal camada de validacao. Branch principal: `main`.
- **Seed:** `prisma/seed.ts` idempotente com dados minimos para desenvolvimento.
- **Migrations em producao:** `pnpm prisma migrate deploy` no pipeline de CI antes do deploy. Migrations destrutivas em duas etapas.

---

## 7. Testes

Segue integralmente o ShiftLabs Stacks v7.0 secao 6.

### 7.1 Stack de Testes

| Camada | Ferramenta | Status | Regra de Uso |
|---|---|---|---|
| Backend — unitarios | Vitest | ✅ Obrigatorio | Logica de negocios, services, utils. |
| Backend — integracao | Supertest + NestJS Testing Module | ✅ Obrigatorio | Endpoints da API REST. |
| Frontend Web — componentes | Vitest + React Testing Library | ✅ Obrigatorio | Componentes e screens criticas. |
| Frontend Web — E2E | Playwright | ✅ Obrigatorio (medio prazo) | Fluxos criticos: cadastro, proposta, assinatura, financeiro. |
| Mobile — componentes | Jest + React Native Testing Library | ✅ Obrigatorio | Componentes e screens criticas. |
| Mobile — E2E | Maestro | ✅ Obrigatorio (longo prazo) | Fluxos criticos: upload camera, assinatura touch, resposta a proposta. |
| IA/LLM — unitarios | Vitest | ✅ Obrigatorio | Logica deterministica ao redor do LLM (parsing, construcao de prompts, validacao de tools). |
| IA/LLM — integracao | Vitest + temperature 0 | ✅ Obrigatorio | Pipeline completo (prompt > LLM > parsing > acao). Validar estrutura do output. |
| IA/LLM — evals | Langfuse Evals | ✅ Obrigatorio | Golden datasets com 20-50 exemplos por dominio. Rodar periodicamente fora do CI. |

### 7.2 Cenarios Criticos de Teste do Cedente

Conforme plano de testes consolidado (01.5 secao 5): 45 cenarios de teste (T-01 a T-45) cobrindo todas as 92 RNs. Os testes de integracao e E2E devem cobrir esses cenarios progressivamente.

---

## 8. Deploy e CI/CD

### 8.1 Deploy por Camada

| Camada | Plataforma | Status | Regra de Uso |
|---|---|---|---|
| Frontend Web (Next.js) | **Vercel** | ✅ Obrigatorio | Preview deployments automaticos por PR. Edge network global. SSR/SSG nativo. |
| Backend (NestJS) | **Railway** | ✅ Definido | Docker obrigatorio. `[DECISAO AUTONOMA — Railway. Simplicidade, Docker nativo, auto-deploy. Descartado: Render (cold starts), VPS (overhead operacional).]` |
| Banco de Dados | **Supabase** | ✅ Obrigatorio | PostgreSQL gerenciado. 3 projetos: desenvolvimento, staging e producao. `[DECISAO AUTONOMA — 3 projetos Supabase (dev/staging/prod). Dados financeiros exigem staging isolado. Descartado: Branch database (feature ainda em beta, risco para dados financeiros).]` |
| Mobile | **EAS Build + EAS Update** | ✅ Obrigatorio | Perfis: development, preview, production. App Store (iOS) e Google Play (Android). |
| Cache (Redis) | **Upstash** | ✅ Definido | `[DECISAO AUTONOMA — Upstash serverless Redis. Zero ops, pay-per-use. Descartado: Railway addon (custo fixo), ElastiCache (over-engineering).]` |
| Filas (RabbitMQ) | **CloudAMQP** | ✅ Definido | `[DECISAO AUTONOMA — CloudAMQP gerenciado. Descartado: Railway addon (menos controle).]` |
| E-mail Transacional | **Resend** | ✅ Definido | `[DECISAO AUTONOMA — Resend. SDK TypeScript nativo, DX excelente. Descartado: SendGrid (SDK pesado, DX inferior), Amazon SES (over-engineering para o estagio).]` |

### 8.2 Pipeline de CI/CD

| Tecnologia | Status | Regra de Uso |
|---|---|---|
| GitHub Actions | ✅ Obrigatorio | Pipeline minimo: lint > type-check > testes > prisma migrate deploy > build > deploy. |
| GitHub | ✅ Obrigatorio | Plataforma de repositorio. README.md, .env.example, .gitignore, turbo.json e docker-compose.yml desde o primeiro commit. |

### 8.3 Ambientes

- **Backend:** Desenvolvimento local (Docker) + staging (Railway) + producao (Railway).
- **Frontend Web:** Desenvolvimento local + producao. Preview por PR obrigatorio (Vercel nativo).
- **Mobile:** Development build (device) + producao (stores). Preview build recomendado.
- **Supabase:** 3 projetos obrigatorios (dev + staging + prod). Staging isolado para testar migrations e integracoes sem afetar producao.

---

## 9. Seguranca

Segue integralmente o ShiftLabs Stacks v7.0 secao 8. Requisitos adicionais derivados das RNs do Cedente:

### 9.1 Requisitos de Seguranca Especificos

| Requisito | RN Origem | Implementacao |
|---|---|---|
| Isolamento total de dados entre Cedentes | RN-011 | RLS (Row Level Security) do Supabase habilitado em todas as tabelas. Toda query filtrada pelo ID do Cedente logado. Tentativa de acesso a dados de outro usuario: bloqueio + log de seguranca. |
| Anonimato do Cessionario | RN-012 | Nenhuma API do modulo Cedente retorna dados pessoais de Cessionarios. Nenhuma view, DTO ou response schema inclui campos de identificacao do Cessionario. |
| Imutabilidade de CPF/CNPJ | RN-007 | Campo desabilitado para edicao no frontend. Validacao no backend: rejeitar qualquer request de update que inclua cpf/cnpj. |
| Bloqueio por tentativas de login | RN-005 | 5 tentativas consecutivas > bloqueio de 15 minutos. Implementar via `@nestjs/throttler` com store Redis. |
| Expiracao de sessao | RN-006 | 24 horas de inatividade. Modal de aviso 5 minutos antes. Preservar rascunhos locais. |
| LGPD — exclusao de dados | RN-010 | Endpoint `POST /api/v1/cedente/lgpd/exclusion-request`. Protocolo gerado. Admin processa em ate 15 dias corridos. |
| Validacao de MIME type no upload | RN-042 | Validacao pelo tipo real do arquivo (MIME type), nao pela extensao. Whitelist: PDF, JPG, PNG. |
| Documentos assinados imutaveis | RN-049 | Apos assinatura via ZapSign, nenhuma API permite alteracao do documento. Hash do PDF armazenado. |

### 9.2 Stack de Seguranca

| Tecnologia | Status | Regra de Uso |
|---|---|---|
| HTTPS | ✅ Obrigatorio | Todos os ambientes de producao e staging. |
| CORS | ✅ Obrigatorio | Configurado explicitamente no NestJS. Wildcard (`*`) proibido em producao. |
| Helmet.js | ✅ Obrigatorio | Headers de seguranca automaticos. |
| `@nestjs/throttler` | ✅ Obrigatorio | Rate limiting em endpoints publicos. Restritivo em auth (login, registro, reset de senha). |
| `@nestjs/csrf` | ✅ Obrigatorio | Protecao CSRF em app web com cookies (refresh token em httpOnly cookie). |
| class-validator pipes | ✅ Obrigatorio | Sanitizacao de todos os inputs no backend. |
| Prisma (query builder tipado) | ✅ Obrigatorio | Protecao contra SQL injection. `$queryRaw` apenas com parametros preparados. |
| Supabase RLS | ✅ Obrigatorio | Row Level Security em todas as tabelas com dados de Cedente. `service_role` key nunca exposta no frontend/mobile. |
| Sentry | ✅ Obrigatorio | Error tracking em todas as camadas. Deploy sem Sentry configurado e proibido. |
| PostHog Feature Flags | ✅ Obrigatorio | Kill switch para features de IA (Guardiao do Retorno). Rollout progressivo. |
| Certificate pinning (mobile) | ✅ Obrigatorio | O modulo Cedente lida com dados financeiros sensiveis (RN-051 a RN-054). `[DECISAO AUTONOMA — Certificate pinning obrigatorio para dados financeiros. Implementar via expo-dev-client. Descartado: nao implementar (risco inaceitavel para dados financeiros).]` Pinning configurado para dominios da API e Supabase. |

### 9.3 Seguranca de IA

Aplicar integralmente ShiftLabs v7.0 secao 11.9:

- Sanitizacao de input do usuario obrigatoria antes de inserir em prompts.
- Separacao clara entre instrucoes e dados (delimitadores `<user_input>`).
- Validacao de output para function calling.
- Rate limiting por usuario em endpoints do Guardiao: 30 chamadas/minuto por usuario. `[DECISAO AUTONOMA — 30 chamadas/min por usuario. Baseado em padrao de uso de chatbot assistente. Descartado: 20/min (muito restritivo para fluxo conversacional), 60/min (risco de abuso).]` Ajustavel apos testes de carga.
- `max_tokens` configurado em toda chamada.
- Moderacao da OpenAI (`/v1/moderations`) para filtrar inputs do Cedente.
- PII (CPF, dados financeiros) mascarada antes de enviar ao LLM.

---

## 10. Analytics e Tracking

Segue integralmente o ShiftLabs Stacks v7.0 secao 9.

### 10.1 Ferramenta e Implementacao

| Tecnologia | Status | Regra de Uso |
|---|---|---|
| PostHog | ✅ Obrigatorio | Product analytics + session replay + feature flags + A/B testing. |
| Wrapper centralizado | ✅ Obrigatorio | `src/services/analytics.ts` no web e mobile. Nunca chamar SDK diretamente nos componentes. |

### 10.2 Eventos Minimos do Cedente

| Evento | Quando | Propriedades Adicionais |
|---|---|---|
| `screen_viewed` | Toda tela | `screen_name` |
| `signup_completed` | RN-001 confirmado | `user_type: pf\|pj` |
| `login_completed` | Login bem-sucedido | — |
| `case_created` | RN-024 cadastro confirmado | `scenario: A\|B\|C\|D` |
| `document_uploaded` | RN-041/RN-042 upload concluido | `document_type`, `file_format` |
| `proposal_received` | RN-030 proposta apresentada | `proposal_value` |
| `proposal_accepted` | RN-032 aceite confirmado | `proposal_value`, `scenario` |
| `proposal_rejected` | RN-034 recusa confirmada | — |
| `counterproposal_sent` | RN-035 contraproposta enviada | `counter_value` |
| `scenario_escalated` | RN-025 escalonamento concluido | `from_scenario`, `to_scenario` |
| `document_signed` | RN-047 assinatura ZapSign concluida | `document_type` |
| `case_cancelled` | RN-055 cancelamento confirmado | `cancel_reason` |
| `guardian_message_sent` | RN-058 mensagem ao Guardiao | — |
| `guardian_escalated` | RN-061 escalacao para humano | — |

### 10.3 Privacidade

- LGPD: aviso de coleta de dados e opt-out conforme exigido por lei.
- Nunca enviar PII (CPF, e-mail, senhas, dados financeiros) para PostHog.
- `user_id` como unico identificador de usuario no analytics.

---

## 11. Tecnologias Proibidas

Consolidacao das proibicoes do ShiftLabs Stacks v7.0 aplicaveis ao modulo Cedente:

| Categoria | Proibido | Motivo |
|---|---|---|
| Backend — Framework | Express puro, Fastify, Hono, Koa | NestJS e o unico framework aprovado. |
| Backend — ORM | TypeORM, Sequelize, Drizzle | Prisma e o unico ORM aprovado. |
| Backend — Banco | MongoDB, DynamoDB, MySQL, MariaDB, SQLite em producao | PostgreSQL (Supabase) e o unico banco aprovado. |
| Frontend — UI libs | MUI, Chakra, AntD, Bootstrap | shadcn/ui e o unico sistema de componentes. |
| Frontend — CSS | UnoCSS, Windi CSS | Tailwind CSS e o unico framework CSS. |
| Frontend — State | Redux, MobX, Context API (como global state) | Zustand e o padrao. |
| Frontend — Fetch | Axios | `fetch` nativo + TanStack Query. |
| Frontend — Animacao | GSAP como substituto do Framer Motion | Framer Motion e o padrao. GSAP como complemento via ADR. |
| Frontend — Loading | Spinners genericos | Skeleton screens obrigatorios. |
| Mobile — Framework | Flutter, Kotlin Multiplatform, nativo puro | React Native + Expo e o padrao. |
| Mobile — Navegacao | React Navigation | expo-router e o padrao. |
| Mobile — UI libs | React Native Paper, NativeBase, Tamagui | Componentes nativos do zero. |
| Mobile — Estilizacao | NativeWind, Styled Components | `StyleSheet.create()` nativo. |
| Mobile — Token storage | AsyncStorage (para tokens de auth) | expo-secure-store obrigatorio. |
| IA — Orquestracao Python | LangChain Python, CrewAI, AutoGen | Stack 100% TypeScript/Node.js. |
| IA — Vector store externo | Pinecone, Qdrant, Weaviate | Supabase pgvector e o padrao. |
| IA — Chamada direta do frontend | API OpenAI chamada do frontend/mobile | Toda chamada LLM via backend NestJS. |
| Infra — Package manager | npm, yarn | pnpm e o padrao. |
| Infra — Repos separados | Repos por camada para o mesmo produto | Monorepo com Turborepo. |
| Geral — JavaScript puro | Arquivos .js/.jsx em novos projetos | TypeScript strict exclusivamente. |
| Geral — Push notifications | OneSignal, Pusher | Expo Notifications e o padrao. |
| Geral — i18n | react-intl, LinguiJS | i18next + react-i18next quando necessario. |
| Geral — Datas | Moment.js, Day.js, Luxon | date-fns + date-fns-tz. |

---

## 12. ADRs (Architecture Decision Records)

### ADR-001: Next.js unico para face publica e area logada do Cedente

| Campo | Valor |
|---|---|
| **Contexto** | O ShiftLabs Stacks v7.0 define dois caminhos: Next.js para face publica/SEO e React+Vite para dashboards 100% logados. O modulo Cedente tem ambos: landing page, cadastro e onboarding (publico) + dashboard, propostas, documentos, financeiro (logado). |
| **Decisao** | Usar um unico app Next.js (App Router) com route groups `(public)/` e `(authenticated)/`. Server Components para rotas publicas (SEO). Client Components para area logada (interatividade). |
| **Alternativas descartadas** | (A) Dois apps separados (Next.js + React+Vite) — complexidade de deploy, duplicacao de config, fragmentacao de shared state. (B) React+Vite unico sem SSR — perde SEO na landing page. |
| **Consequencias** | Bundle ligeiramente maior que SPA puro na area logada. Mitigado por code splitting automatico do App Router. Ganho: setup simplificado, deploy unico, shared state sem fragmentacao, experiencia unificada. |

### ADR-002: App mobile nativo (React Native + Expo) alem do web responsivo

| Campo | Valor |
|---|---|
| **Contexto** | RN-087 define 3 acoes criticas no mobile: upload via camera, assinatura touch e resposta a proposta. O web responsivo cobre funcionalidade basica, mas a experiencia de camera nativa, push notifications nativas e SecureStore exigem app dedicado. |
| **Decisao** | Desenvolver app mobile com React Native + Expo (managed workflow) como complemento ao web responsivo. Prioridade para as 3 acoes criticas de RN-087. |
| **Alternativas descartadas** | (A) Apenas web responsivo — experiencia inferior para camera e assinatura. (B) PWA — push notifications limitados no iOS, sem SecureStore. |
| **Consequencias** | Custo adicional de manutencao de um app mobile. Mitigado pelo compartilhamento de logica via monorepo (TanStack Query hooks, Zustand stores, types, services). |

### ADR-003: ZapSign como servico de assinatura eletronica

| Campo | Valor |
|---|---|
| **Contexto** | RN-047, RN-080 e RN-081 exigem assinatura eletronica com validade juridica, integracao inline no painel (iframe), rastreabilidade e imutabilidade. O ShiftLabs Stacks v7.0 nao define servico de assinatura. |
| **Decisao** | ZapSign como servico de assinatura eletronica. Integracao via API para geracao de envelopes e webhooks para confirmacao de assinatura. Carregamento inline no painel via iframe. |
| **Alternativas descartadas** | (A) DocuSign — custo mais alto, overkill para o volume inicial. (B) Assinatura propria — complexidade juridica e regulatoria inviavel. |
| **Consequencias** | Dependencia de servico externo. Mitigado por: fallback com abertura em nova aba (RN-047), mensagem de indisponibilidade (RN-080) e retry (3 tentativas antes de escalar). `[DECISAO AUTONOMA — ZapSign confirmado. Orcamento R$ 2-5/envelope com desconto por volume. Descartado: Clicksign (custo por envelope ~2x maior para o volume estimado).]` API REST, SDK JavaScript, webhooks para status de assinatura. |

### ADR-004: Consulta a Receita Federal para validacao de CNPJ

| Campo | Valor |
|---|---|
| **Contexto** | RN-069 e RN-082 exigem validacao de CNPJ em tempo real no cadastro de Cedente PJ. O ShiftLabs Stacks v7.0 nao define integracao com Receita Federal. |
| **Decisao** | Integrar com ReceitaWS (consulta.ws) para validacao de situacao cadastral do CNPJ. Consulta on blur no campo CNPJ. Fallback: cadastro permitido com aviso, verificacao manual pelo Admin na triagem (RN-082). `[DECISAO AUTONOMA — ReceitaWS. API confiavel, sem certificado digital, pricing acessivel. Descartado: API oficial da Receita (exige certificado digital e-CNPJ), CNPJ.ws (menos documentacao).]` |
| **Alternativas descartadas** | (A) Validacao apenas pelo Admin na triagem — atrasa feedback ao Cedente PJ. (B) Sem validacao — risco de empresa irregular operar na plataforma. (C) API oficial da Receita — exige certificado digital e-CNPJ. (D) CNPJ.ws — menos documentacao. |
| **Consequencias** | Dependencia de API externa com disponibilidade variavel. Mitigado pelo fallback documentado em RN-082. ReceitaWS oferece plano gratuito (ate 3 consultas/min) e plano pago para producao. |

### ADR-005: Parceiro Escrow para conta garantia

| Campo | Valor |
|---|---|
| **Contexto** | RN-051 a RN-054 e RN-083 exigem integracao com parceiro Escrow para gestao de conta garantia: deposito confirmado, distribuicao, estorno e reversao. O ShiftLabs Stacks v7.0 nao define parceiro Escrow. |
| **Decisao** | Integrar com Celcoin como parceiro Escrow. Fintech regulada pelo BCB, APIs REST modernas, escrow digital nativo, split de pagamentos e conciliacao automatica. O Cedente visualiza status no painel Financeiro (somente leitura). Atualizacao de status via webhooks do parceiro (ate 5 minutos — RN-083). `[DECISAO AUTONOMA — Celcoin. Regulacao BCB, APIs REST, escrow digital nativo. Descartado: Transfeera (foco em pagamentos, nao escrow).]` |
| **Alternativas descartadas** | (A) Conta propria — exige autorizacao do Banco Central, inviavel no curto prazo. (B) Sem Escrow — sem garantia financeira, inviavel para o modelo de negocio. (C) Transfeera — foco em pagamentos, nao escrow. |
| **Consequencias** | Dependencia critica de servico externo. Fallback: exibir ultimo status conhecido com nota de horario (RN-083). Celcoin regulada pelo BCB oferece garantia de compliance. |

---

## 13. Changelog

| Versao | Data | Autor | Alteracoes |
|---|---|---|---|
| v1.0 | 2026-03-22 | Claude Code Desktop | Criacao do documento normativo de stacks do modulo Cedente. Baseado no ShiftLabs Stacks v7.0 e nas Regras de Negocio D01 (01.1 a 01.5, v1.2). 5 ADRs documentados. 3 integracoes externas mapeadas (ZapSign, Receita Federal, Escrow). |
| v1.1 | 2026-03-22 | Claude Code Desktop | Resolucao de todas as pendencias de stack (STACK-PEND-01 a STACK-PEND-09). Deploy backend: Railway. Redis: Upstash. RabbitMQ: CloudAMQP. CNPJ: ReceitaWS. Escrow: Celcoin. ZapSign: contrato confirmado R$ 2-5/envelope. Rate limiting IA: 30/min. Certificate pinning: obrigatorio. Staging Supabase: 3 projetos. Adicionado Resend como e-mail transacional. Backlog de infraestrutura (boilerplates) documentado com prazos. |

---

## 14. Backlog de Pendencias

| ID | Descricao | Tipo | Status | Resolucao |
|---|---|---|---|---|
| STACK-PEND-01 | Servico gerenciado para deploy do backend NestJS em producao | Custo / Lock-in | ✅ Resolvido | Railway. Docker nativo, auto-deploy via GitHub, pricing previsivel. |
| STACK-PEND-02 | Servico gerenciado para Redis em producao | Custo | ✅ Resolvido | Upstash serverless Redis. Zero ops, pay-per-use. |
| STACK-PEND-03 | Servico gerenciado para RabbitMQ em producao | Custo | ✅ Resolvido | CloudAMQP gerenciado. |
| STACK-PEND-04 | Provedor de API para validacao de CNPJ (Receita Federal) | Custo / Compliance | ✅ Resolvido | ReceitaWS (consulta.ws). API confiavel, sem certificado digital, pricing acessivel. |
| STACK-PEND-05 | Parceiro Escrow (banco ou fintech regulado) | Custo / Compliance / Seguranca | ✅ Resolvido | Celcoin. Regulacao BCB, APIs REST, escrow digital nativo. |
| STACK-PEND-06 | Contrato e custo por envelope ZapSign | Custo | ✅ Resolvido | ZapSign confirmado. R$ 2-5/envelope com desconto por volume. API REST, SDK JavaScript, webhooks. |
| STACK-PEND-07 | Rate limiting por usuario no endpoint do Guardiao do Retorno | Seguranca | ✅ Resolvido | 30 chamadas/min por usuario. Ajustavel apos testes de carga. |
| STACK-PEND-08 | Certificate pinning no app mobile | Seguranca | ✅ Resolvido | Obrigatorio. Implementar via expo-dev-client para dominios da API e Supabase. |
| STACK-PEND-09 | Ambiente de staging para Supabase | Compliance | ✅ Resolvido | 3 projetos Supabase (dev/staging/prod). |

### 14.1 Backlog de Infraestrutura (Boilerplates)

| Item | Status | Prazo |
|---|---|---|
| Boilerplate monorepo (Turborepo + pnpm) | 🔶 Pendente | Pre-requisito antes do primeiro sprint |
| Boilerplate backend NestJS | 🔶 Pendente | Pre-requisito antes do primeiro sprint |
| Boilerplate Next.js (web-cedente) | 🔶 Pendente | Pre-requisito antes do primeiro sprint |
| Boilerplate mobile Expo (mobile-cedente) | 🔶 Pendente | Pre-requisito antes do primeiro sprint |
| Sentry (setup em todas as camadas) | 🔶 Pendente | Setup do boilerplate |
| PostHog (setup em todas as camadas) | 🔶 Pendente | Setup do boilerplate |
| CI/CD GitHub Actions | 🔶 Pendente | Primeiro sprint |

---

*Documento normativo — Modulo Cedente — Repasse Seguro — v1.1*
