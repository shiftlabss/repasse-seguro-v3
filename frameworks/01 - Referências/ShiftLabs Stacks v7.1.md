# Stacks

# Stacks Tecnológicas – Shift Labs

## Padrão Técnico Oficial para Todos os Produtos

| Destinatário | Todo o time técnico da Shift Labs (devs, tech leads, produto) |
| --- | --- |
| Escopo | Definição oficial das stacks de Backend, Frontend Web, Mobile, IA/LLM e Infraestrutura que devem ser adotadas em todo produto desenvolvido pela Shift Labs |
| Caráter | **Normativo.** Este documento define o padrão obrigatório. Desvios exigem aprovação do responsável técnico via ADR. |
| Versão | v7.1 |
| Responsável | Fernando Calado |
| Data | 22/03/2026 (America/Fortaleza) |
| Revisão | Trimestral ou a cada mudança significativa de stack |

<aside>
⚠️

**Este documento é o padrão oficial da Shift Labs.** Todo novo projeto, módulo ou serviço deve seguir as definições aqui descritas. Qualquer desvio (uso de outra linguagem, framework, banco de dados ou ferramenta não listada) precisa de justificativa técnica documentada via ADR (Architecture Decision Record) e aprovação do responsável técnico antes da implementação.

</aside>

<aside>
🤖

**Desenvolvimento 100% por IA.** Todo código da Shift Labs é gerado por agentes de IA (Claude Code e GPT Codex). Não há esforço humano no desenvolvimento. Isso significa que **boilerplates, convenções de código, estrutura de pastas e testes automatizados** são os principais contratos de qualidade — eles são o que garante consistência, rastreabilidade e confiabilidade do output gerado. Toda decisão de stack neste documento é desenhada para maximizar a qualidade do output dos agentes de IA.

</aside>

<aside>
🧠

**Duas camadas de IA na Shift Labs.** É fundamental distinguir:

1. **IA para desenvolvimento (seções 1 a 10):** Claude Code e GPT Codex geram 100% do código. São ferramentas de desenvolvimento, não são visíveis ao usuário final dos produtos.
2. **IA no produto (seção 11):** GPT-4, RAG, agentes e pipelines de LLM embarcados nos produtos entregues ao cliente. São features de IA que o usuário final interage.

Toda decisão de stack nas seções 1 a 10 é otimizada para maximizar a qualidade do código gerado por Claude Code e GPT Codex. A seção 11 define como os produtos que entregam IA ao usuário devem ser construídos.

</aside>

<aside>
📌

**TL;DR – Stack oficial Shift Labs (v7.0)**

- **Desenvolvimento (IA para gerar código):** 100% por agentes de IA (Claude Code + GPT Codex). Zero esforço humano no código. Boilerplates, convenções e testes são os contratos de qualidade. Claude Code e GPT Codex são as ferramentas de desenvolvimento — não confundir com a IA embarcada nos produtos (seção 11).
- **IA no produto (IA para o usuário final):** GPT-4 como modelo padrão para features de IA entregues ao usuário (chatbots, agentes, RAG, classificadores). OpenAI SDK + Vercel AI SDK + LangChain.js. Langfuse para observabilidade. Detalhes na seção 11.
- **Backend:** Node.js + NestJS + TypeScript strict + Prisma + Supabase (PostgreSQL). Redis para cache. RabbitMQ para filas. Docker para ambiente local.
- **Frontend Web (público/SEO):** Next.js (App Router) + TypeScript strict + Tailwind CSS v4 + shadcn/ui + Framer Motion. Deploy em Vercel. TanStack Query + Zustand.
- **Frontend Web (interno/logado):** React 19 + Vite + TypeScript strict + Tailwind CSS v4 + shadcn/ui + Framer Motion. TanStack Router como roteador padrão. TanStack Query + Zustand.
- **Mobile:** React Native + Expo + expo-router + TypeScript strict. Reanimated para animações. SecureStore para tokens.
- **Banco de Dados:** Supabase como plataforma padrão (PostgreSQL gerenciado + Auth + Storage + Realtime + Edge Functions).
- **Motion:** Framer Motion na web. React Native Reanimated no mobile. Motion tokens padronizados cross-plataforma.
- **Comunicação:** API REST (JSON) com `fetch` nativo. JWT como padrão de autenticação. SSE para streaming de LLM. Supabase Realtime para subscriptions de dados. WebSocket para bidirecional customizado (via ADR).
- **Segurança:** HTTPS obrigatório, sanitização de inputs, CORS configurado, rate limiting, credentials em variáveis de ambiente. Sentry obrigatório para error tracking.
- **Analytics:** PostHog como ferramenta padrão. Obrigatório em todo produto com usuários. Eventos padronizados em snake_case com wrapper centralizado.
- **Infraestrutura:** Monorepo com Turborepo + pnpm. Docker para ambiente local. Supabase como backend-as-a-service. Vercel para landing pages e frontends Next.js.
- **IA/LLM:** GPT-4 como modelo padrão. OpenAI SDK + Vercel AI SDK + LangChain.js. RAG via Supabase pgvector. Langfuse para observabilidade. Prompts como código. SSE para streaming. Structured Outputs obrigatório para tarefas determinísticas.
</aside>

<aside>
🗺️

**Glossário de Decisões Arquiteturais**

Referência rápida do "porquê" de cada escolha. Para detalhes, consultar a seção indicada.

| **Decisão** | **Justificativa em uma linha** | **Seção** |
| --- | --- | --- |
| NestJS (não Express/Fastify) | Arquitetura modular com decorators, Swagger embutido, guards/pipes nativos. Output de IA mais consistente | 1.1 |
| Prisma (não TypeORM/Drizzle) | Tipagem auto-gerada, migrations declarativas, integração nativa com Supabase. Único ORM aprovado | 1.1 |
| Supabase (não PostgreSQL raw) | PostgreSQL gerenciado + Auth + Storage + Realtime + pgvector em uma plataforma. Elimina infra operacional | 1.2 |
| Redis + RabbitMQ | Redis para cache com TTL. RabbitMQ para filas com retry e dead-letter. Separação de responsabilidades | 1.3 |
| Next.js vs React+Vite | Next.js para face pública/SEO. React+Vite para dashboards/backoffice 100% logado | 2.1 |
| shadcn/ui + Tailwind v4 | Ownership total do código (não dependência). Excelente representação no treinamento dos modelos de IA | 2.3 |
| TanStack Router (Vite) | Type-safe, integração nativa com TanStack Query. Padrão para apps React+Vite | 2.1 |
| TanStack Query + Zustand | TanStack Query para dados do servidor (cache, sync). Zustand para estado global. Mesmo padrão web e mobile | 2.2 |
| Framer Motion / Reanimated | Framer Motion na web (declarativo, AnimatePresence). Reanimated no mobile (UI thread). Motion tokens compartilhados | 2.4 / 3.6 |
| React Native + Expo | Managed workflow, OTA updates, build via EAS. TypeScript strict. Mesmo ecossistema React do web | 3.1 |
| Turborepo + pnpm | Monorepo com cache inteligente. Agente de IA vê o projeto inteiro. Types compartilhados entre apps | 5.1 |
| JWT (não sessions) | Stateless, funciona igual web e mobile. Access token curto + refresh token longo | 4.4 |
| SSE / Supabase Realtime / WebSocket | SSE para streaming de LLM. Supabase Realtime para subscriptions de dados. WebSocket para bidirecional via ADR | 4.1.1 |
| Sentry | Error tracking com stack traces, breadcrumbs e alertas. SDKs para todas as camadas da stack | 8.6 |
| PostHog | Analytics + session replay + feature flags + A/B testing em uma plataforma. Open-source | 8.7 / 9.2 |
| GPT-4 (produto) | Melhor equilíbrio qualidade/function calling/ecossistema. OpenAI SDK bem representado no treinamento dos agentes | 11.1 |
| Langfuse | Observabilidade de LLM: tracing, custo, latência, qualidade. Open-source, integra com OpenAI SDK e LangChain | 11.8 |
| Supabase pgvector (não Pinecone) | Embeddings no mesmo PostgreSQL dos dados. Queries híbridas (vetorial + SQL). Sem infra adicional | 11.3 |

</aside>

---

### 1. Backend

#### 1.1 Stack Obrigatória

- **Runtime:** Node.js (LTS).
- **Framework HTTP:** NestJS — arquitetura modular, decorators TypeScript, Swagger embutido, guards, interceptors e pipes de validação.
- **Linguagem:** TypeScript com `strict: true` obrigatório em todo código backend.
- **ORM:** Prisma como único ORM aprovado — tipagem auto-gerada, migrations declarativas, integração nativa com Supabase/PostgreSQL.

#### 1.2 Banco de Dados

- **Plataforma:** Supabase como padrão para todos os produtos. Supabase fornece PostgreSQL gerenciado, autenticação, storage, realtime e edge functions em uma única plataforma.
- **Banco relacional:** PostgreSQL (via Supabase). PostgreSQL é o único banco relacional aprovado.
- **Supabase Auth:** Pode ser utilizado como camada de autenticação quando adequado ao produto, em complemento ou substituição ao JWT próprio.
- **Supabase Storage:** Usar para armazenamento de arquivos (imagens, documentos, uploads) quando o produto exigir.
- **Supabase Realtime:** Disponível para funcionalidades de tempo real (subscriptions, presence) com justificativa documentada por produto.
- Não utilizar MySQL, MariaDB, SQLite em produção ou outro banco relacional sem aprovação via ADR.

<aside>
💡

**Por que Supabase?** Elimina a necessidade de gerenciar PostgreSQL, auth, storage e realtime separadamente. Reduz complexidade operacional e acelera o setup de novos produtos. O Prisma se conecta ao PostgreSQL do Supabase via connection string padrão.

</aside>

#### 1.3 Cache e Filas

- **Cache:** Redis para cache de dados críticos e melhoria de performance. Em ambiente local, rodar via Docker. Em produção, usar serviço gerenciado (Upstash, Railway, ou equivalente).
- **Filas:** RabbitMQ para mensageria e processamento assíncrono. Em ambiente local, rodar via Docker. Em produção, usar serviço gerenciado (CloudAMQP ou equivalente).

#### 1.4 Requisitos Técnicos Obrigatórios

- **Observabilidade:** Todo serviço deve ter logs estruturados com **Pino**. Logs devem incluir request ID, timestamp, nível e contexto.
- **Filas:** Toda fila RabbitMQ deve ter política de retry configurada e dead-letter queue associada.
- **Cache:** Todo uso de Redis deve ter TTL definido e estratégia de invalidação documentada.
- **Validação:** Usar `class-validator` + `class-transformer` integrados aos DTOs do NestJS. Toda entrada de dados validada via pipes.
- **Documentação de API:** Swagger/OpenAPI auto-gerado via `@nestjs/swagger` é obrigatório em todo serviço.

#### 1.5 Convenções de Banco de Dados (Prisma Schema)

Estas convenções são obrigatórias em todo schema Prisma de todo produto. O objetivo é que qualquer agente de IA gere schemas consistentes entre produtos sem ambiguidade.

**Primary Keys**

- **UUID v4** como padrão para primary keys em todas as tabelas. Nunca usar auto-increment (`@default(autoincrement())`).
- No Prisma: `id String @id @default(uuid()) @db.Uuid`.
- Justificativa: UUIDs são seguros para expor em URLs e APIs, não revelam volume de dados, e funcionam bem em ambientes distribuídos.

**Colunas de Auditoria (obrigatórias em toda tabela)**

- Toda tabela deve ter, no mínimo:

```prisma
created_at DateTime @default(now()) @db.Timestamptz
updated_at DateTime @updatedAt @db.Timestamptz
```

- Tabelas que rastreiam quem criou/modificou devem adicionar:

```prisma
created_by String? @db.Uuid
updated_by String? @db.Uuid
```

- `created_at` e `updated_at` são gerenciados automaticamente pelo Prisma. `created_by` e `updated_by` devem ser preenchidos pelo service com o ID do usuário autenticado.

**Soft Delete vs Hard Delete**

- **Soft delete como padrão** para tabelas de domínio de negócio (usuários, pedidos, contratos, documentos). Implementar com coluna `deleted_at DateTime? @db.Timestamptz`. Registro deletado: `deleted_at` preenchido. Registro ativo: `deleted_at` é `null`.
- **Hard delete permitido** para tabelas auxiliares (logs, tokens expirados, cache, filas processadas) e tabelas de junção sem significado de negócio.
- Todo service que opera em tabelas com soft delete deve filtrar `deleted_at: null` por padrão. Criar um middleware Prisma ou método helper para aplicar o filtro automaticamente.

**Relacionamentos**

- **`onDelete` padrão:** `CASCADE` para tabelas filhas que não fazem sentido sem o pai (ex: items de um pedido). `SET NULL` para referências opcionais (ex: produto sem categoria). `RESTRICT` para evitar exclusão acidental de registros referenciados (ex: usuário com pedidos ativos).
- Documentar a estratégia de `onDelete` escolhida em cada relation do schema com comentário inline.

**Nomenclatura**

- **Tabelas:** `snake_case`, plural. Exemplos: `users`, `order_items`, `contract_transfers`. No Prisma, usar `@@map("nome_tabela")` quando o nome do model for diferente.
- **Colunas:** `snake_case`. Exemplos: `first_name`, `created_at`, `is_active`. No Prisma, usar `@map("nome_coluna")` quando necessário.
- **Enums:** `PascalCase` no Prisma, `UPPER_SNAKE_CASE` nos valores. Exemplo: `enum OrderStatus { PENDING CONFIRMED CANCELLED DELIVERED }`.
- **Índices:** Nomear explicitamente com `@@index([campos], map: "idx_tabela_campos")`. Exemplo: `@@index([user_id, created_at], map: "idx_orders_user_created")`.
- **Unique constraints:** `@@unique([campos], map: "uq_tabela_campos")`.

**Índices**

- Toda foreign key deve ter índice. O Prisma cria automaticamente para `@relation`, mas verificar.
- Colunas usadas em `WHERE` frequente devem ter índice. Colunas usadas em `ORDER BY` frequente devem ter índice.
- Índices compostos: a ordem das colunas importa. Colocar a coluna mais seletiva primeiro.
- Para buscas textuais, avaliar `GIN` index com `pg_trgm` via `$queryRaw` quando necessário.

<aside>
🚫

**Proibido no schema**

- Auto-increment como primary key.
- Tabelas sem `created_at` e `updated_at`.
- `timestamp` sem timezone (`@db.Timestamp`). Usar sempre `@db.Timestamptz`.
- Hard delete em tabelas de domínio de negócio sem justificativa.
- Foreign keys sem índice.
- Nomes de tabelas ou colunas em camelCase no banco (usar `@map`/`@@map` quando necessário).

</aside>

<aside>
🚫

**Proibido no backend**

- Uso de bancos NoSQL (MongoDB, DynamoDB, etc.) sem aprovação via ADR.
- Uso de frameworks alternativos (Express puro, Fastify, Hono, Koa) sem aprovação via ADR.
- TypeORM, Sequelize, Drizzle ou outros ORMs — Prisma é o único ORM aprovado.
- PostgreSQL auto-gerenciado (Docker em produção) — usar Supabase.
- Deploy de serviços sem logs estruturados.
- Filas sem dead-letter queue.
- Credentials, tokens ou secrets no código-fonte (ver seção 8).
- Queries SQL raw sem uso explícito de `$queryRaw` com parâmetros preparados do Prisma.
</aside>

<aside>
📝

**Versões mínimas das dependências**

As versões abaixo devem ser mantidas atualizadas conforme o ambiente de produção. Responsável: Fernando Calado.

**Backend**

- **Node.js:** 22.x+ (LTS)
- **NestJS:** 10.x+
- **TypeScript:** 5.4+
- **Prisma:** 6.x+
- **PostgreSQL (Supabase):** 17+
- **Redis:** 7.4+
- **RabbitMQ:** 4.x+
- **Pino:** 9.x+
- **Helmet:** 8.x+
- **Sentry (`@sentry/nestjs`):** 9.x+

**Frontend Web (Next.js)**

- **Next.js:** 15.x+
- **React:** 19.x+
- **TypeScript:** 5.4+
- **TanStack Query:** 5.x+
- **Zustand:** 5.x+
- **Framer Motion (Motion):** 12.x+
- **Tailwind CSS:** 4.x+
- **shadcn/ui:** latest
- **Radix Primitives:** latest
- **Sentry (`@sentry/nextjs`):** 9.x+
- **PostHog (`posthog-js`):** 1.x+
- **date-fns:** 4.x+
- **date-fns-tz:** 3.x+

**Frontend Web (React + Vite)**

- **React:** 19.x+
- **React DOM:** 19.x+
- **Vite:** 7.x+
- **TypeScript:** 5.4+
- **TanStack Query:** 5.x+
- **TanStack Router:** 1.x+
- **Zustand:** 5.x+
- **Framer Motion (Motion):** 12.x+
- **Tailwind CSS:** 4.x+
- **shadcn/ui:** latest
- **Radix Primitives:** latest
- **Sentry (`@sentry/react`):** 9.x+
- **PostHog (`posthog-js`):** 1.x+
- **date-fns:** 4.x+
- **date-fns-tz:** 3.x+

**Mobile**

- **React Native:** 0.76+
- **Expo SDK:** 52+
- **expo-router:** 4.x+
- **React Native Reanimated:** 3.x+
- **React Native Gesture Handler:** 2.x+
- **Zustand:** 5.x+
- **TanStack Query:** 5.x+
- **Sentry (`@sentry/react-native`):** 6.x+
- **PostHog (`posthog-react-native`):** 3.x+
- **date-fns:** 4.x+
- **date-fns-tz:** 3.x+

**Infraestrutura**

- **Docker Engine:** 27+
- **Docker Compose:** 2.x+
- **Turborepo:** 2.x+
- **pnpm:** 9.x+
</aside>

---

### 2. Frontend Web

#### 2.1 Dois Caminhos — Critério de Decisão

A Shift Labs opera com **dois setups de frontend web**, selecionados conforme o tipo de produto:

| **Critério** | **Next.js (App Router)** | **React + Vite (SPA)** |
| --- | --- | --- |
| Quando usar | Produto com **face pública**, SEO relevante, landing pages, marketplace, onboarding público | Produto **100% logado**: dashboard, admin panel, backoffice, ferramentas internas |
| Rendering | SSR/SSG via App Router com Server Components | SPA puro, client-side rendering |
| Roteamento | File-based (App Router nativo) | **TanStack Router** como padrão (type-safe, integração com TanStack Query). `react-router-dom` como alternativa via ADR |
| Build tool | Next.js built-in (Turbopack) | Vite |
| Deploy | **Vercel** como padrão | Definido por produto (VPS, Railway, etc.) |

<aside>
💡

**Regra simples:** se o produto tem qualquer página que precisa ser indexada pelo Google ou compartilhada em redes sociais com preview, use **Next.js + Vercel**. Se é tudo atrás de login, use **React + Vite**.

</aside>

#### 2.2 Stack Comum (ambos os caminhos)

- **Linguagem:** TypeScript com `strict: true` obrigatório em todo código frontend web.
- **Dados do servidor:** TanStack Query (React Query) como padrão para fetching, caching e sincronização de dados do servidor. Nunca usar `fetch` direto em `useEffect` sem cache.
- **Estado global:** Zustand quando complexidade exigir. Para projetos simples, `useState` e props são aceitos.
- **Proibido para estado:** Redux, MobX, Context API como state manager global.

#### 2.3 UI e Estilização

- **Sistema de componentes:** **shadcn/ui** como padrão. Baseado em **Radix Primitives**, com acessibilidade completa (keyboard nav, ARIA, focus management) inclusa. Os componentes são copiados para o repositório do projeto — ownership total do código, sem dependência externa de runtime. Componentes customizados são permitidos quando shadcn/ui não cobrir o caso de uso.
- **Estilização:** **Tailwind CSS** como framework de estilização padrão em todos os produtos web. Classes utilitárias para estilização de componentes. CSS customizado permitido para casos complexos (animações avançadas, efeitos visuais).
- **Design tokens:** Definidos centralmente no arquivo CSS principal (`global.css`) usando `@theme` directives do Tailwind v4, e expostos automaticamente como variáveis CSS (`var(--token-name)`). Tokens de cores, tipografia, espaçamentos e motion devem ser configurados via `@theme` e consumidos via classes utilitárias ou variáveis CSS.
- **cn() helper:** Usar a função `cn()` (clsx + tailwind-merge) como padrão para composição condicional de classes. Localizada em `src/lib/utils.ts`.

**2.3.1 Design Tokens Obrigatórios**

Todo produto da Shift Labs deve implementar os tokens abaixo no `global.css` via `@theme`. Os valores concretos podem ser ajustados por produto, mas a estrutura e os nomes dos tokens são obrigatórios para consistência cross-produto.

**Cores (estrutura obrigatória, valores por produto)**

```css
@theme {
  --color-background: /* fundo principal */;
  --color-foreground: /* texto principal */;
  --color-primary: /* ação principal, CTAs */;
  --color-primary-foreground: /* texto sobre primary */;
  --color-secondary: /* ação secundária */;
  --color-secondary-foreground: /* texto sobre secondary */;
  --color-muted: /* fundos sutis, placeholders */;
  --color-muted-foreground: /* texto sutil */;
  --color-accent: /* destaques, badges */;
  --color-accent-foreground: /* texto sobre accent */;
  --color-destructive: /* erros, ações destrutivas */;
  --color-destructive-foreground: /* texto sobre destructive */;
  --color-border: /* bordas */;
  --color-input: /* borda de inputs */;
  --color-ring: /* outline de focus */;
  --color-card: /* fundo de cards */;
  --color-card-foreground: /* texto em cards */;
  --color-popover: /* fundo de popovers/dropdowns */;
  --color-popover-foreground: /* texto em popovers */;
}
```

Estes nomes são compatíveis com shadcn/ui e devem ser usados para que os componentes shadcn/ui funcionem corretamente com o tema do produto.

**Escala de Espaçamento (base 4px)**

```css
@theme {
  --spacing-0: 0px;
  --spacing-1: 4px;
  --spacing-2: 8px;
  --spacing-3: 12px;
  --spacing-4: 16px;
  --spacing-5: 20px;
  --spacing-6: 24px;
  --spacing-8: 32px;
  --spacing-10: 40px;
  --spacing-12: 48px;
  --spacing-16: 64px;
  --spacing-20: 80px;
  --spacing-24: 96px;
}
```

Base 4px. Toda medida de espaçamento (padding, margin, gap) deve usar múltiplos de 4px via esses tokens.

**Escala Tipográfica**

```css
@theme {
  --font-sans: 'Inter', system-ui, -apple-system, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;

  --text-xs: 0.75rem;    /* 12px */
  --text-sm: 0.875rem;   /* 14px */
  --text-base: 1rem;     /* 16px */
  --text-lg: 1.125rem;   /* 18px */
  --text-xl: 1.25rem;    /* 20px */
  --text-2xl: 1.5rem;    /* 24px */
  --text-3xl: 1.875rem;  /* 30px */
  --text-4xl: 2.25rem;   /* 36px */

  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;

  --leading-tight: 1.25;
  --leading-normal: 1.5;
  --leading-relaxed: 1.625;
}
```

**Border Radius**

```css
@theme {
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-xl: 16px;
  --radius-full: 9999px;
}
```

**Sombras**

```css
@theme {
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);
  --shadow-xl: 0 20px 25px -5px rgb(0 0 0 / 0.1);
}
```

**Tokens Mobile**

Os mesmos valores devem ser replicados como constantes TypeScript em `packages/shared-utils/tokens.ts` ou `src/constants/tokens.ts` no app mobile. Os nomes devem ser idênticos (em camelCase) para manter paridade visual cross-plataforma.

<aside>
💡

**Por que shadcn/ui + Tailwind?** Em um ambiente onde todo código é gerado por agentes de IA (Claude Code, GPT Codex), a qualidade e consistência do output é significativamente maior com shadcn/ui + Tailwind — ambos são extremamente bem representados no treinamento dos modelos. shadcn/ui entrega acessibilidade completa via Radix Primitives sem esforço adicional, e o código gerado é 100% ownership da Shift Labs.

</aside>

#### 2.4 Motion, Animações e Efeitos Visuais

**2.4.1 Biblioteca de Animação (obrigatória)**

- **Framer Motion (Motion)** como biblioteca padrão de animações para React em todos os produtos web da Shift Labs.
- Cobre transições de tela (`AnimatePresence`), animações de layout (`layoutId`), gestos (drag, tap, hover), animações baseadas em scroll e spring physics.
- API declarativa que se integra naturalmente com componentes React.

**2.4.2 CSS Vanilla para Motion**

- **Transitions e transforms** para micro-interações leves (hover, focus, active states).
- **@keyframes** para animações contínuas (loading spinners, pulsos, shimmers).
- **`backdrop-filter: blur()`** para efeitos de glassmorphism em modais, overlays e cards.
- **Scroll-driven animations** com `animation-timeline: scroll()` para efeitos de parallax e reveal sem JavaScript.
- **Gradientes animados** para backgrounds dinâmicos em seções de destaque.

**2.4.3 Motion Tokens (obrigatórios em todo projeto)**

Todo projeto Shift Labs deve implementar o seguinte sistema de motion tokens como variáveis CSS, integradas ao `@theme` no arquivo CSS principal do Tailwind v4 para uso via classes utilitárias quando aplicável.

- **Duração:** `--duration-instant: 100ms`, `--duration-fast: 150ms`, `--duration-normal: 250ms`, `--duration-slow: 400ms`, `--duration-page: 500ms`.
- **Easing:** `--ease-out: cubic-bezier(0.16, 1, 0.3, 1)`, `--ease-in-out: cubic-bezier(0.65, 0, 0.35, 1)`, `--ease-spring: cubic-bezier(0.34, 1.56, 0.64, 1)`.
- **Escala:** `--scale-press: 0.97`, `--scale-hover: 1.02`.
- **Opacidade:** `--opacity-enter: 0`, `--opacity-visible: 1`.

**2.4.4 Diretrizes Obrigatórias de Motion**

- **Micro-interações em todos os elementos interativos.** Botões, cards, inputs e links devem ter feedback visual em hover, focus e click.
- **Transições de tela obrigatórias.** Toda mudança de rota ou tela deve usar `AnimatePresence` do Framer Motion para transições suaves (fade, slide ou morph).
- **Loading states animados.** Skeleton screens com shimmer effect em vez de spinners genéricos.
- **Princípio de sobriedade.** Animações devem ser percebidas como fluidez, não como distração. Priorizar sutileza sobre espetacularidade.

#### 2.5 Acessibilidade (a11y)

- **HTML semântico obrigatório.** Usar tags corretas (`<button>`, `<nav>`, `<main>`, `<header>`, `<section>`, `<article>`, etc.) em vez de `<div>` para tudo.
- **Navegação por teclado.** Todo elemento interativo deve ser acessível via Tab e acionável via Enter/Space.
- **Atributos ARIA quando necessário.** Usar `aria-label`, `aria-hidden`, `aria-expanded`, `role` etc. em componentes customizados que não têm semântica nativa (modais, dropdowns, tooltips, tabs).
- **Contraste mínimo.** Textos devem ter ratio de contraste de pelo menos 4.5:1 (AA) contra o fundo. Textos grandes (18px+ ou 14px+ bold): 3:1.
- **Foco visível.** Nunca remover `outline` de focus sem substituir por um indicador visual equivalente. Todo estado `:focus-visible` deve ser estético e claro.
- **Textos alternativos.** Toda imagem informativa deve ter `alt` descritivo. Imagens decorativas devem usar `alt=""`.
- **Tamanho mínimo de toque.** Áreas clicáveis devem ter no mínimo 44x44px em interfaces touch.

#### 2.6 Internacionalização (i18n)

- **Estado atual:** Não adotada como padrão. Os produtos da Shift Labs operam em português (pt-BR) neste momento.
- **Biblioteca aprovada (quando necessário):** **i18next** + **react-i18next** como solução padrão para web e mobile. Mesma biblioteca nas duas plataformas, permitindo compartilhar arquivos de tradução.
- **Diretriz:** Todo texto visível ao usuário deve ser extraído para arquivos de tradução no momento em que i18n for adotado no produto. Não hardcodar strings de UI nos componentes.
- **Proibido:** react-intl, LinguiJS ou outras bibliotecas de i18n sem aprovação.

<aside>
🚫

**Proibido no frontend web**

- Bibliotecas de UI de terceiros instaladas como dependência (MUI, Chakra, AntD, Bootstrap, etc.). shadcn/ui é o único sistema de componentes aprovado (ownership de código, não dependência).
- Frameworks de utilidade CSS alternativos ao Tailwind (UnoCSS, Windi CSS, etc.) sem aprovação.
- JavaScript puro (sem TypeScript) em novos projetos — TypeScript strict é obrigatório.
- Three.js/WebGL no produto principal (reservado para landing pages ou marketing).
- GSAP como substituto do Framer Motion (pode ser usado como complemento para casos específicos com aprovação).
- Spinners genéricos como loading state. Usar skeleton screens.
- Remover `outline` de focus sem substituto visual.
- Usar `<div>` ou `<span>` como elemento clicável sem `role` e `tabIndex` apropriados.
- `fetch` direto em `useEffect` sem TanStack Query para dados do servidor.
- Redux, MobX ou Context API como state manager global.
</aside>

#### 2.7 Performance

**2.7.1 Metas de Web Vitals (produtos Next.js)**

- **LCP (Largest Contentful Paint):** < 2.5 segundos. Medir no PageSpeed Insights ou Lighthouse.
- **CLS (Cumulative Layout Shift):** < 0.1. Reservar espaço para imagens com `width`/`height` ou `aspect-ratio`. Evitar injeção de conteúdo acima do fold após o carregamento.
- **INP (Interaction to Next Paint):** < 200ms. Evitar tarefas longas no main thread. Usar `startTransition` para atualizações de estado não urgentes.
- Monitoramento em produção via PostHog ou Web Vitals API. Alertar quando métricas degradarem acima dos thresholds.

**2.7.2 Bundle Size**

- **Budget de bundle:** Definir por produto. Referência: JavaScript total (gzipped) abaixo de 200KB para produtos Next.js com SSR, abaixo de 300KB para SPAs React+Vite.
- **Code splitting:** Obrigatório em todo produto. Next.js faz code splitting automático por rota. Em React+Vite, usar `React.lazy()` + `Suspense` para rotas e componentes pesados.
- **Lazy loading de componentes:** Modais, drawers, gráficos e componentes que não são visíveis no carregamento inicial devem ser lazy loaded.
- **Tree shaking:** Garantido pelo uso de ESM. Importar apenas o necessário (ex: `import { format } from 'date-fns'`, nunca `import * as dateFns`).
- **Análise de bundle:** Usar `@next/bundle-analyzer` (Next.js) ou `rollup-plugin-visualizer` (Vite) para identificar dependências pesadas.

**2.7.3 Imagens (Web)**

- **Formato:** WebP como padrão para imagens fotográficas. SVG para ícones e ilustrações. AVIF como otimização adicional quando suportado.
- **Componente:** `next/image` (Next.js) ou `<img>` com `loading="lazy"` e `decoding="async"` (React+Vite).
- **Tamanho máximo:** Imagens otimizadas antes do upload. Referência: máximo 200KB por imagem para web após compressão. Usar ferramentas de compressão no pipeline de build ou no upload.
- **Responsive images:** Usar `srcSet` e `sizes` para servir tamanhos adequados ao viewport. `next/image` faz isso automaticamente.

**2.7.4 Performance Mobile**

- **Lista longas:** Usar `FlatList` com `getItemLayout` para listas com scroll. Nunca usar `ScrollView` com muitos items.
- **Imagens:** Usar `expo-image` com cache automático e placeholder blur (seção 3.4).
- **Animações:** Todas na UI thread via Reanimated. Nunca animar com `setState` ou `Animated` API legacy.
- **Startup time:** Minimizar imports no entry point. Lazy load de screens secundárias.

---

### 3. Mobile (iOS e Android)

#### 3.1 Stack Obrigatória

- **Framework:** React Native como padrão único para desenvolvimento mobile na Shift Labs.
- **Plataforma de desenvolvimento:** **Expo** (managed workflow) como padrão. Expo simplifica build, deploy, OTA updates e acesso a APIs nativas sem necessidade de configurar Xcode/Android Studio manualmente.
- **Linguagem:** TypeScript com `strict: true` obrigatório no mobile.
- **Eject para bare workflow:** Permitido apenas quando uma funcionalidade nativa específica exigir, com justificativa documentada e aprovação.

#### 3.2 Navegação

- **expo-router** como biblioteca padrão de navegação (file-based routing). Obrigatório em todo projeto mobile.
- Layouts com Stack, Tabs e Drawer configurados via sistema de arquivos.
- Deep linking configurado nativamente pelo expo-router desde o início do projeto.
- Typed routes habilitado para navegação type-safe.

#### 3.3 UI e Estilização

- **Componentes:** Construídos do zero com componentes nativos do React Native (`View`, `Text`, `Pressable`, `ScrollView`, `FlatList`, etc.).
- **Estilização:** `StyleSheet.create()` nativo do React Native. Sem bibliotecas de estilização externas (NativeWind, Styled Components, etc.) sem aprovação.
- **Design tokens:** Os mesmos tokens de cores, tipografia e espaçamentos definidos para web devem ser replicados no mobile via objeto TypeScript constante. Manter consistência visual cross-plataforma.
- **Componentes de UI de terceiros:** Não utilizar React Native Paper, NativeBase, Tamagui ou similares sem aprovação.

#### 3.4 Imagens e Assets

- **Formato preferencial:** SVG para ícones e ilustrações (via `react-native-svg`). PNG apenas para imagens fotográficas ou quando SVG não for viável.
- **Resolução:** Para assets PNG/JPG, fornecer @1x, @2x e @3x seguindo a convenção do React Native (`image.png`, `image@2x.png`, `image@3x.png`).
- **Performance de imagens:** Usar **expo-image** (substituto moderno do `<Image>` nativo) para carregamento otimizado, cache automático e suporte a placeholder blur.
- **Otimização:** Todas as imagens devem ser otimizadas antes de entrar no repositório (comprimir sem perda de qualidade visível). Ferramentas sugeridas: TinyPNG, ImageOptim, ou script automatizado no CI.
- **Proibido:** Usar `<Image>` nativo do React Native para imagens remotas em produção (sem cache). Usar expo-image ou alternativa com cache.

#### 3.5 Gerenciamento de Estado

- **Dados do servidor:** TanStack Query (React Query) — mesmo padrão do web. Fetching, caching e sincronização de dados do servidor.
- **Estado global:** Zustand como solução de estado global (mesmo padrão do web).
- **Estado local:** `useState` e props para estado de componente.
- **Persistência local:** `AsyncStorage` (via `@react-native-async-storage/async-storage`) para dados não-sensíveis que precisam sobreviver ao fechamento do app. Zustand com middleware `persist` usando AsyncStorage para estado global persistente.
- **Proibido:** Redux, MobX ou outras soluções sem aprovação.

#### 3.6 Motion e Animações

- **Biblioteca padrão:** **React Native Reanimated** para animações performantes na UI thread.
- **Gestos:** **React Native Gesture Handler** para interações táteis (swipe, drag, pinch, long press).
- **Transições de tela:** Configurar transições customizadas no expo-router usando Reanimated.
- **Motion tokens:** Os mesmos valores de duração, easing e escala definidos na seção 2.4.3 devem ser replicados como constantes TypeScript no mobile.
- **Diretrizes:** Micro-interações em todos os elementos interativos (feedback tátil em Pressable). Skeleton screens com shimmer para loading states. Princípio de sobriedade (mesmo do web).

#### 3.7 Comunicação com Backend

- Mesma API REST consumida pelo frontend web. Sem backend separado para mobile.
- **Cliente HTTP:** `fetch` nativo do React Native (mesmo padrão do web).
- **Dados do servidor:** TanStack Query — mesmo padrão do web.
- **Autenticação:** JWT com mesma estratégia definida na seção 4.3 (Autenticação). Access token em memória, refresh token armazenado de forma segura via `expo-secure-store`.
- **Offline-first:** Para produtos que exigem funcionalidade offline, usar estratégia de cache local com sincronização. A ser definida por produto quando necessário.

#### 3.8 Acessibilidade Mobile

- **Props de acessibilidade obrigatórias:** `accessibilityLabel`, `accessibilityRole`, `accessibilityState` em todos os elementos interativos.
- **Tamanho mínimo de toque:** 44x44 pontos (mesmo padrão do web, alinhado com as diretrizes da Apple e Google).
- **Suporte a Dynamic Type (iOS) e Font Scale (Android):** Textos devem respeitar as configurações de tamanho de fonte do sistema operacional.
- **Contraste:** Mesmos padrões AA definidos na seção 2.5.
- **Testar com VoiceOver (iOS) e TalkBack (Android)** nos fluxos críticos antes de cada release.

#### 3.9 Build e Distribuição

- **Build:** EAS Build (Expo Application Services) como padrão para gerar builds de iOS e Android.
- **OTA Updates:** EAS Update para atualizações over-the-air de JavaScript sem precisar de novo build nativo.
- **Distribuição interna (testes):** EAS Build com perfil de preview para distribuição interna ao time.
- **Produção:** App Store (iOS) e Google Play (Android). Seguir as guidelines de review de cada store.

#### 3.10 Push Notifications

- **Padrão:** **Expo Notifications** como solução padrão para push notifications em iOS e Android. Expo Notifications abstrai a configuração de APNs (Apple) e FCM (Firebase) e funciona dentro do managed workflow.
- **Registro de tokens:** O push token do device deve ser enviado ao backend e associado ao usuário autenticado. O backend é responsável por armazenar tokens e disparar notificações via Expo Push API.
- **Permissão:** Sempre solicitar permissão de notificação de forma contextual (explicar o valor antes de pedir), nunca no primeiro acesso ao app.
- **Fallback:** Se o produto precisar de funcionalidades avançadas não suportadas pelo Expo Notifications (ex: rich notifications complexas, notification service extensions), avaliar Firebase Cloud Messaging direto com justificativa documentada.
- **Proibido:** OneSignal, Pusher ou outros serviços de push de terceiros sem aprovação.

<aside>
🚫

**Proibido no mobile**

- Flutter, Kotlin Multiplatform, ou desenvolvimento nativo puro (Swift/Kotlin) sem aprovação.
- React Navigation como biblioteca de navegação — usar expo-router.
- Bibliotecas de UI de terceiros (React Native Paper, NativeBase, Tamagui, etc.) sem aprovação.
- NativeWind, Styled Components ou outras libs de estilização externas sem aprovação.
- Armazenar tokens de autenticação em AsyncStorage (usar `expo-secure-store`).
- Publicar builds sem testar acessibilidade com leitor de tela nos fluxos críticos.
- Eject do Expo managed workflow sem justificativa documentada e aprovação.
</aside>

<aside>
💡

**Compartilhamento de código Web ↔ Mobile**

Por usar React em ambas as plataformas, as seguintes camadas podem ser compartilhadas entre web e mobile via packages do monorepo (seção 5.1): lógica de estado (Zustand stores), TanStack Query hooks e configuração, funções utilitárias, services/wrappers de API, validações, constantes e tipos TypeScript. A camada de UI (componentes visuais) não deve ser compartilhada, pois React DOM e React Native têm primitivas diferentes. Extrair lógica compartilhada para `packages/shared-types/`, `packages/shared-utils/` e `packages/shared-services/` no monorepo do produto.

</aside>

---

### 4. Comunicação Frontend/Mobile ↔ Backend

#### 4.1 Protocolo e Padrão de API

- **Protocolo:** API REST (JSON). Obrigatório para toda comunicação entre frontend/mobile e backend.
- **Cliente HTTP:** `fetch` nativo (tanto no browser quanto no React Native). Não utilizar Axios ou outras bibliotecas sem aprovação.
- **Cache e sincronização:** TanStack Query como camada obrigatória entre o `fetch` e os componentes. Nunca fazer fetch direto em componentes sem cache.
- **Documentação:** Swagger/OpenAPI auto-gerado via `@nestjs/swagger`. Todo endpoint documentado.
- **GraphQL, gRPC, WebSockets:** Não aprovados como padrão. Podem ser usados para casos específicos (ex: Supabase Realtime para subscriptions) com justificativa documentada.
- **Streaming de IA (SSE):** Para produtos com funcionalidades de LLM, o streaming de respostas usa SSE como padrão. Ver seção 11.6 para detalhes.

**4.1.1 Critério de Decisão — Comunicação em Tempo Real**

| **Cenário** | **Tecnologia** | **Quando usar** |
| --- | --- | --- |
| Streaming de respostas LLM para o frontend | **SSE (Server-Sent Events)** | Comunicação unidirecional: backend envia tokens incrementalmente ao frontend. Padrão para toda feature de IA com streaming |
| Subscriptions de dados, presence, notificações em tempo real | **Supabase Realtime** | Mudanças em tabelas do banco, presence de usuários online, notificações push em tempo real. Aproveita a infraestrutura Supabase já existente |
| Comunicação bidirecional customizada com lógica de negócio | **WebSocket (via ADR)** | Quando SSE e Supabase Realtime não cobrem o caso de uso. Exemplos: chat peer-to-peer, colaboração em tempo real com edição simultânea, gaming. Exige justificativa documentada via ADR |

<aside>
💡

**Regra simples:** Se é LLM, use SSE. Se é dado do banco em tempo real, use Supabase Realtime. Se é bidirecional e nenhum dos dois resolve, abra um ADR para WebSocket.

</aside>

#### 4.2 Padrões de Requisição

- Endpoints organizados por domínio de negócio (ex: `/api/v1/[dominio]/[recurso]`).
- Padrão RESTful para verbos HTTP: GET (leitura), POST (criação), PUT/PATCH (atualização), DELETE (remoção).
- **API única para web e mobile.** Não criar endpoints separados por plataforma. Se necessário adaptar resposta por plataforma, usar header `X-Platform: web|ios|android`.

#### 4.3 Padrões de Resposta

- **Estrutura padrão de resposta de sucesso:**

```json
{
  "success": true,
  "data": {},
  "meta": {}
}
```

- **Estrutura padrão de resposta de erro:**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Descrição legível para o desenvolvedor",
    "details": {}
  }
}
```

- **Códigos HTTP obrigatórios:** 400 (validação), 401 (não autenticado), 403 (sem permissão), 404 (não encontrado), 409 (conflito), 422 (entidade não processável), 500 (erro interno).
- Nunca retornar stack traces ou detalhes internos do servidor em produção.
- Mensagens de erro para o usuário final devem ser tratadas no frontend/mobile, não no backend.
- No NestJS, usar **Exception Filters** globais para padronizar todas as respostas de erro.

#### 4.4 Autenticação e Autorização

- **Padrão de autenticação:** JWT (JSON Web Tokens) como método padrão para todos os produtos Shift Labs.
- **Alternativa aprovada:** Supabase Auth pode ser utilizado como camada de autenticação quando adequado ao produto, substituindo JWT manual.
- **Access token:** curta duração (15 a 30 minutos). Enviado via header `Authorization: Bearer <token>`.
- **Refresh token (web):** longa duração (7 a 30 dias). Armazenado em httpOnly cookie.
- **Refresh token (mobile):** mesma duração. Armazenado em `expo-secure-store` (nunca em AsyncStorage).
- **Autorização:** NestJS Guards como mecanismo padrão. Cada produto deve implementar controle de acesso baseado em roles ou permissions conforme sua necessidade (RBAC, ABAC, etc.), documentado por produto.
- Outras estratégias de autenticação (sessions, OAuth, etc.) podem ser usadas com justificativa documentada.

**4.4.1 Fluxo de Autenticação End-to-End**

O fluxo abaixo é o padrão obrigatório para produtos com JWT manual. Produtos usando Supabase Auth seguem o fluxo nativo do Supabase.

- **Armazenamento do access token (web):** Em memória (variável JavaScript ou Zustand store). Nunca em localStorage ou sessionStorage. O token é perdido ao recarregar a página, e o refresh token (em httpOnly cookie) o renova automaticamente.
- **Armazenamento do access token (mobile):** Em memória (Zustand store). O refresh token em `expo-secure-store` renova o access token ao reabrir o app.
- **Refresh automático:** O fetch wrapper (em `services/api.ts`) deve interceptar respostas 401 e tentar refresh automaticamente antes de propagar o erro ao componente. Se o refresh falhar (refresh token expirado ou inválido), redirecionar para a tela de login e limpar o estado de autenticação.
- **Logout:** Deve invalidar o refresh token no backend (endpoint `POST /api/v1/auth/logout`), limpar o httpOnly cookie (web) ou o SecureStore (mobile), e resetar o estado de autenticação no Zustand.
- **Interceptor padrão:** Todo produto deve implementar um interceptor no fetch wrapper que: (1) adiciona o access token no header `Authorization`, (2) intercepta 401 e tenta refresh, (3) reenvia a requisição original com o novo token, (4) redireciona para login se o refresh falhar.

```typescript
// Exemplo de estrutura do interceptor (services/api.ts)
// O agente de IA deve gerar a implementação completa seguindo este padrão
export async function fetchWithAuth(url: string, options?: RequestInit) {
  // 1. Adicionar access token ao header
  // 2. Fazer a requisição
  // 3. Se 401, tentar refresh
  // 4. Se refresh OK, reenvar requisição original
  // 5. Se refresh falhar, redirect para login
}
```

#### 4.5 Versionamento de API

- **Padrão:** Versionamento por URL path: `/api/v1/[recurso]`.
- A versão deve ser incrementada quando houver breaking changes.
- Versões anteriores devem ser mantidas por pelo menos 3 meses após deprecação.

#### 4.6 Paginação

- **Padrão:** Cursor-based pagination como padrão para listas ordenadas cronologicamente ou por ID (feeds, históricos, logs, mensagens). Offset-based pagination permitido para listas com ordenação arbitrária (tabelas com sort por colunas, busca com filtros).
- **Estrutura de resposta paginada (cursor-based):**

```json
{
  "success": true,
  "data": {
    "items": [],
    "nextCursor": "eyJpZCI6MTAwfQ==",
    "hasMore": true
  },
  "meta": {
    "limit": 20
  }
}
```

- **Estrutura de resposta paginada (offset-based):**

```json
{
  "success": true,
  "data": {
    "items": [],
    "total": 247,
    "page": 1,
    "pageSize": 20,
    "totalPages": 13
  }
}
```

- **Limite máximo por página:** 100 itens. Padrão: 20 itens. Configurável via query parameter `limit` ou `pageSize`.
- **Query parameters:** Cursor-based: `?cursor=xxx&limit=20`. Offset-based: `?page=1&pageSize=20`.
- O backend deve validar e limitar `limit`/`pageSize` para evitar queries sem bound.

<aside>
💡

**Por que cursor-based como padrão?** Em datasets grandes e com inserções frequentes, offset-based pode pular ou duplicar registros quando novos dados são inseridos entre requisições. Cursor-based garante consistência. Para tabelas administrativas com sort por colunas, offset-based é mais simples e adequado.

</aside>

#### 4.7 Tratamento de Erros no Frontend e Mobile

- **Error Boundary (React):** Todo app web deve ter um Error Boundary global envolvendo o componente raiz. Exibir tela de fallback amigável com opção de "tentar novamente". O Error Boundary deve reportar o erro ao Sentry automaticamente.
- **Error Boundary (React Native):** Mesmo padrão. Usar Error Boundary no componente raiz do app Expo.
- **Tratamento de erros HTTP no TanStack Query:** Configurar `onError` global no `QueryClient` para tratar erros HTTP de forma centralizada. Erros 401 são tratados pelo interceptor de auth (seção 4.4.1). Erros 4xx exibem mensagem ao usuário via toast/notification. Erros 5xx exibem mensagem genérica ("Algo deu errado, tente novamente").
- **Toast/Notification padrão:** Todo produto deve ter um componente de toast/notification centralizado para exibir feedback ao usuário (sucesso, erro, aviso). No web, usar toast do shadcn/ui. No mobile, implementar componente de toast customizado com Reanimated.
- **Retry automático:** TanStack Query com `retry: 3` para erros de rede (5xx, timeout). Sem retry para erros de validação (4xx).
- **Mensagens de erro:** O backend retorna `error.code` (para lógica) e `error.message` (para devs). O frontend mapeia `error.code` para mensagens amigáveis ao usuário em pt-BR. Nunca exibir `error.message` do backend diretamente ao usuário final.

#### 4.8 Upload de Arquivos

- **Plataforma:** Supabase Storage como padrão para armazenamento de arquivos em todos os produtos.
- **Estratégia de upload:** Upload direto do frontend/mobile para o Supabase Storage via **signed URL** gerada pelo backend. O backend gera a signed URL (endpoint `POST /api/v1/uploads/signed-url`), o frontend faz o upload direto para o Supabase Storage, e depois confirma o upload ao backend com o path do arquivo.
- **Nomenclatura de arquivos:** UUID v4 como nome do arquivo. Preservar a extensão original. Exemplo: `a1b2c3d4-e5f6-7890-abcd-ef1234567890.pdf`. Nunca usar o nome original do arquivo como path no storage (risco de colisão e caracteres especiais).
- **Organização no storage:** Buckets organizados por domínio. Paths no formato: `[bucket]/[produto]/[dominio]/[uuid].[ext]`. Exemplo: `documents/menux/contracts/a1b2c3d4.pdf`.
- **Validação (backend, obrigatória):** Tipos MIME permitidos definidos por produto (whitelist, nunca blacklist). Tamanho máximo por tipo de arquivo definido por produto (recomendado: 10MB para imagens, 50MB para documentos). Validação antes de gerar a signed URL.
- **Validação (frontend/mobile):** Validar tipo e tamanho antes de iniciar o upload para feedback imediato ao usuário. Não confiar apenas na validação do frontend.
- **Proibido:** Upload passando pelo backend (multipart/form-data no NestJS) como padrão. Usar signed URL para não sobrecarregar o backend com transferência de arquivos.

#### 4.9 Logging no Frontend e Mobile

- **Princípio:** Em desenvolvimento, logs são livres. Em produção, logs são controlados e direcionados ao Sentry como breadcrumbs.
- **Desenvolvimento:** `console.log`, `console.warn`, `console.error` são permitidos. Removidos automaticamente no build de produção via plugin do bundler.
- **Produção (web):** Nenhum `console.log` em builds de produção. Sentry captura erros automaticamente. Ações relevantes do usuário (navegação, cliques em CTAs, submissões de formulário) são registradas como Sentry breadcrumbs para contexto de debug.
- **Produção (mobile):** Nenhum `console.log` em builds de produção (usar `babel-plugin-transform-remove-console`). Mesma estratégia de breadcrumbs do Sentry.
- **O que nunca logar (nenhum ambiente):** Tokens de autenticação, senhas, dados de pagamento, CPF, ou qualquer PII. Aplica-se a console, Sentry e PostHog.
- **Integração Sentry + PostHog:** PostHog registra eventos de analytics (comportamento do usuário). Sentry registra erros e breadcrumbs (debug). Ambos recebem o mesmo `user_id` para correlação.

#### 4.10 Data, Hora e Timezone

- **Backend e banco de dados:** UTC como padrão obrigatório. Todo timestamp armazenado no PostgreSQL deve ser `timestamptz` (timestamp with time zone) em UTC. Nunca usar `timestamp` sem timezone.
- **Serialização:** ISO 8601 como formato obrigatório para troca de datas entre backend e frontend/mobile. Exemplo: `2026-03-22T14:30:00.000Z`. Nunca usar formatos localizados ou Unix timestamps na API.
- **Frontend e mobile:** Receber datas em UTC (ISO 8601) e converter para o timezone do usuário apenas na camada de apresentação (componentes). Nunca converter timezone em services ou stores.
- **Biblioteca de manipulação:** `date-fns` como padrão para manipulação de datas no frontend e mobile (tree-shakeable, imutável, funcional). `date-fns-tz` para conversão de timezone. Não usar Moment.js (deprecated), Day.js ou Luxon sem aprovação.
- **Locale:** `pt-BR` como locale padrão para formatação de datas exibidas ao usuário. Usar `Intl.DateTimeFormat` ou `date-fns/locale/pt-BR`.
- **Prisma:** Campos de data no schema devem usar `DateTime` (que mapeia para `timestamptz` no PostgreSQL). O Prisma retorna objetos `Date` do JavaScript em UTC.

#### 4.11 Filtros, Ordenação e Busca em APIs de Listagem

Todo endpoint de listagem (`GET /api/v1/[recurso]`) que retorna coleções deve suportar os padrões abaixo. O objetivo é que qualquer agente de IA gere endpoints de listagem consistentes entre produtos.

**Filtros**

- Filtros via query parameters no formato `?campo=valor`. Exemplos: `?status=active`, `?created_after=2026-01-01T00:00:00Z`, `?user_id=uuid`.
- Filtros compostos com múltiplos campos: `?status=active&category=food`.
- Para filtros de intervalo, usar sufixos `_after`, `_before`, `_gte`, `_lte`. Exemplos: `?created_after=2026-01-01T00:00:00Z&created_before=2026-03-01T00:00:00Z`, `?price_gte=100&price_lte=500`.
- O backend deve validar e sanitizar todos os filtros no DTO. Filtros inválidos ou não suportados devem retornar 400.

**Ordenação**

- Ordenação via query parameter `sort` no formato `?sort=campo:direção`. Exemplos: `?sort=created_at:desc`, `?sort=name:asc`.
- Múltiplos campos de ordenação separados por vírgula: `?sort=status:asc,created_at:desc`.
- O backend deve validar que o campo de sort é permitido (whitelist de campos sortáveis). Campos não permitidos devem retornar 400.
- Ordenação padrão (quando `sort` não é fornecido): `created_at:desc`.

**Busca Textual**

- Busca via query parameter `search`: `?search=termo`.
- O backend define quais campos são pesquisáveis por endpoint (ex: `name`, `email`, `description`). Documentar no Swagger.
- Para buscas simples, usar `ILIKE` do PostgreSQL via Prisma (`contains`, mode `insensitive`). Para buscas avançadas com relevância, avaliar `tsvector`/`tsquery` do PostgreSQL via `$queryRaw`.

**Operações em Lote (Bulk)**

- **Bulk create:** `POST /api/v1/[recurso]/bulk` com array de objetos no body. Limite máximo de 100 itens por requisição. Resposta com array de resultados (sucesso/erro por item).
- **Bulk delete:** `DELETE /api/v1/[recurso]/bulk` com array de IDs no body. Limite máximo de 100 IDs. Resposta com contagem de itens deletados.
- **Bulk update:** `PATCH /api/v1/[recurso]/bulk` com array de objetos `{ id, ...campos }`. Mesmo limite.
- Operações em lote devem ser transacionais (tudo ou nada) quando possível. Documentar no Swagger se a operação é transacional ou parcial.

**Exemplo de URL completa**

```
GET /api/v1/orders?status=confirmed&created_after=2026-01-01T00:00:00Z&sort=created_at:desc&search=pizza&cursor=eyJpZCI6MTAwfQ==&limit=20
```

---

### 5. Repositório e Estrutura de Código

#### 5.1 Organização de Repositórios

- **Monorepo com Turborepo como padrão.** Todo produto da Shift Labs é organizado como monorepo usando **Turborepo** para orquestração de builds, cache inteligente e gerenciamento de dependências entre packages.
- **Convenção de nomes do repositório:** `[produto]` (nome do produto em kebab-case). Exemplos: `menux`, `repasse-seguro`, `flow`.
- **Plataforma:** GitHub como padrão. Todo repo deve ter `README.md`, `.env.example`, `.gitignore`, `turbo.json` e `docker-compose.yml` desde o primeiro commit.

**5.1.1 Estrutura do Monorepo**

```
[produto]/
  apps/
    api/                    # NestJS (backend)
    web/                    # Next.js ou React+Vite (frontend web)
    mobile/                 # Expo (mobile, quando aplicável)
  packages/
    shared-types/           # Types TypeScript compartilhados entre apps
    shared-utils/           # Funções utilitárias (validações, formatadores, constantes)
    shared-services/        # Wrappers de API, TanStack Query hooks reutilizáveis
    config-eslint/          # Configuração de lint compartilhada
    config-typescript/      # tsconfig base compartilhado
    ui/                     # Componentes shadcn/ui customizados reutilizáveis (quando aplicável)
  turbo.json                # Configuração do Turborepo (pipelines, cache)
  package.json              # Workspace root (npm/pnpm workspaces)
  docker-compose.yml        # Ambiente local (Redis, RabbitMQ, etc.)
  .env.example
  .gitignore
  README.md
```

**5.1.2 Convenções de Packages**

- **Apps** (`apps/`): Aplicações deployáveis. Cada app tem seu próprio `package.json`, build e deploy independente.
- **Packages** (`packages/`): Código compartilhado entre apps. Importados via alias do workspace (ex: `@shiftlabs/shared-types`, `@shiftlabs/shared-utils`).
- **Nomenclatura de packages:** `@shiftlabs/[nome]` como scope npm. Exemplos: `@shiftlabs/shared-types`, `@shiftlabs/config-eslint`.
- **Regra de ouro:** Lógica de negócio, tipos, validações, constantes e services são compartilhados via packages. Componentes visuais (UI) não devem ser compartilhados entre web e mobile, pois React DOM e React Native têm primitivas diferentes. O package `ui/` é para componentes reutilizáveis dentro da mesma plataforma (ex: entre apps web).

**5.1.3 Turborepo — Configuração Obrigatória**

- **Pipeline de build:** Definir `turbo.json` com pipelines para `build`, `lint`, `type-check`, `test` e `dev`. O Turborepo resolve automaticamente a ordem de execução baseada no grafo de dependências.
- **Cache:** Cache local habilitado por padrão. Turborepo Remote Cache (via Vercel) recomendado para times com múltiplos agentes de IA operando no mesmo repo.
- **Package manager:** **pnpm** como padrão para workspaces. pnpm é mais eficiente em disco, mais rápido que npm/yarn e tem suporte nativo a workspaces via `pnpm-workspace.yaml`.

<aside>
💡

**Por que Monorepo + Turborepo?** Em um ambiente de desenvolvimento 100% por IA, o agente precisa de visibilidade completa do projeto para gerar código consistente. Com monorepo, o agente vê API, web e mobile no mesmo contexto. Mudanças que afetam múltiplas camadas (ex: alteração de tipo na API que impacta o frontend) são feitas em um único commit com validação imediata pelo TypeScript. Turborepo adiciona cache inteligente e builds paralelos, garantindo que o pipeline de CI continue rápido mesmo com múltiplos apps e packages.

</aside>

<aside>
🚫

**Proibido em repositórios**

- Repos separados por camada para o mesmo produto (ex: `menux-api` e `menux-web` como repos distintos). Usar monorepo.
- npm ou yarn como package manager em novos projetos. Usar pnpm.
- Packages no monorepo sem `package.json` próprio e sem alias `@shiftlabs/` configurado.
- Código duplicado entre apps que deveria estar em um package compartilhado.
- Compartilhar componentes visuais (UI) entre web e mobile no mesmo package. React DOM e React Native têm primitivas diferentes.

</aside>

#### 5.2 Estrutura de Pastas (Backend — NestJS)

```jsx
src/
  modules/              # Módulos por domínio de negócio
    [dominio]/
      [dominio].module.ts
      [dominio].controller.ts
      [dominio].service.ts
      [dominio].repository.ts
      dto/
        create-[dominio].dto.ts
        update-[dominio].dto.ts
      tests/
  prisma/
    schema.prisma        # Schema único do Prisma
    migrations/          # Migrations auto-geradas
    seed.ts              # Seed de dados
  common/
    decorators/          # Decorators customizados
    filters/             # Exception filters globais
    guards/              # Auth guards, role guards
    interceptors/        # Logging, transform interceptors
    pipes/               # Validation pipes customizados
  config/                # Configurações (env, supabase, redis, rabbitmq)
  jobs/                  # Workers e consumers de filas
app.module.ts
main.ts
Dockerfile
docker-compose.yml
tsconfig.json
.env.example
.eslintrc.json
.prettierrc
```

- Cada domínio de negócio tem seu próprio módulo NestJS com controller, service, repository e DTOs.
- Testes ficam dentro do módulo correspondente, na pasta `tests/`.
- O schema Prisma fica centralizado em `prisma/schema.prisma`, conectado ao PostgreSQL do Supabase.

#### 5.3 Estrutura de Pastas (Frontend Web — Next.js)

```jsx
src/
  app/                  # App Router (rotas, layouts, pages)
    (public)/           # Rotas públicas (landing, auth)
    (authenticated)/    # Rotas protegidas
    layout.tsx
    page.tsx
  components/           # Componentes reutilizáveis
    ui/                 # Componentes shadcn/ui
    [ComponentName]/
      ComponentName.tsx
  hooks/                # Custom hooks
  utils/                # Funções utilitárias
  services/             # Chamadas à API (fetch wrappers)
  stores/               # Zustand stores
  types/                # Types TypeScript globais
  lib/
    utils.ts            # cn() helper (shadcn/ui)
  styles/               # CSS global
    global.css          # Tailwind directives, @theme (design tokens, motion tokens), reset e estilos base
postcss.config.mjs       # PostCSS config (Tailwind v4 plugin)
components.json          # Configuração shadcn/ui
next.config.ts
tsconfig.json
.eslintrc.json
.prettierrc
```

#### 5.4 Estrutura de Pastas (Frontend Web — React + Vite)

```jsx
src/
  components/           # Componentes reutilizáveis
    ui/                 # Componentes shadcn/ui
    [ComponentName]/
      ComponentName.tsx
  pages/                # Telas/views principais
    [PageName]/
      PageName.tsx
  hooks/                # Custom hooks
  utils/                # Funções utilitárias
  services/             # Chamadas à API (fetch wrappers)
  stores/               # Zustand stores
  types/                # Types TypeScript globais
  assets/               # Imagens, fontes, ícones
  lib/
    utils.ts            # cn() helper (shadcn/ui)
  styles/               # CSS global
    global.css          # Tailwind directives, @theme (design tokens, motion tokens), reset e estilos base
  App.tsx
  main.tsx
postcss.config.mjs       # PostCSS config (Tailwind v4 plugin)
components.json          # Configuração shadcn/ui
vite.config.ts
tsconfig.json
.eslintrc.json
.prettierrc
```

#### 5.5 Estrutura de Pastas (Mobile — Expo + expo-router)

```jsx
app/                    # expo-router (file-based routing)
  (tabs)/               # Tab navigator
    index.tsx
    profile.tsx
    _layout.tsx
  (auth)/               # Auth flow
    login.tsx
    register.tsx
    _layout.tsx
  _layout.tsx            # Root layout
src/
  components/           # Componentes reutilizáveis
    [ComponentName]/
      ComponentName.tsx
      ComponentName.styles.ts
  hooks/                # Custom hooks
  utils/                # Funções utilitárias
  services/             # Chamadas à API (fetch wrappers)
  stores/               # Zustand stores
  constants/            # Tokens, cores, tipografia, espaçamentos
    tokens.ts           # Design tokens e motion tokens
    colors.ts
    typography.ts
  types/                # Types TypeScript globais
  assets/               # Imagens, fontes, ícones
app.json
eas.json
tsconfig.json
.eslintrc.json
.prettierrc
```

- `app/` contém o file-based routing do expo-router. Cada arquivo é uma rota.
- `src/` contém toda a lógica, componentes, serviços e estado.
- `stores/` centraliza os Zustand stores (obrigatório no mobile por complexidade).
- `constants/` contém os design tokens como constantes TypeScript.

#### 5.6 Padrões de Código (obrigatórios)

**Linting e Formatting**

- **ESLint** obrigatório em todo projeto (backend, frontend web e mobile). Configuração base a ser definida no boilerplate oficial.
- **Prettier** obrigatório para formatação automática. Configuração padrão: `semi: true`, `singleQuote: true`, `trailingComma: 'all'`, `printWidth: 100`.
- Lint e formatting devem rodar automaticamente no pre-commit (via **Husky** + **lint-staged**).

**Convenções de Commits**

- **Conventional Commits** obrigatório. Formato: `tipo(escopo): descrição`.
- Tipos permitidos: `feat`, `fix`, `refactor`, `docs`, `style`, `test`, `chore`, `perf`, `ci`.
- Exemplos: `feat(auth): adicionar refresh token`, `fix(cart): corrigir cálculo de total`, `docs(readme): atualizar instruções de setup`.

**Convenções de Nomenclatura**

- **Arquivos backend:** `kebab-case` para módulos NestJS (`user.service.ts`, `order.controller.ts`, `create-order.dto.ts`).
- **Arquivos frontend web:** `PascalCase` para componentes (`CartItem.tsx`, `OrderHistory.tsx`). `camelCase` para hooks e utils (`useCart.ts`, `formatPrice.ts`).
- **Arquivos mobile:** `PascalCase` para componentes (`CartItem.tsx`, `OrderHistory.tsx`). `camelCase` para hooks, utils e stores (`useCart.ts`, `formatPrice.ts`, `cartStore.ts`).
- **Variáveis e funções:** `camelCase`.
- **Classes e componentes React/React Native:** `PascalCase`.
- **Constantes:** `UPPER_SNAKE_CASE`.
- **Tabelas e colunas do banco (Prisma schema):** `snake_case` (usar `@@map` e `@map` no Prisma quando necessário).
- **Endpoints de API:** `kebab-case` para rotas (`/api/v1/order-items`).

**Code Review**

- Todo código gerado por agentes de IA deve passar por **pull request** antes do merge. O PR é o registro de rastreabilidade do que foi gerado, não um gate de aprovação humana.
- PRs devem ter descrição clara do que foi gerado, contexto da instrução dada ao agente e, quando aplicável, screenshots ou GIFs para mudanças visuais.
- O **pipeline de CI** (lint → type-check → testes) é a principal camada de validação. Se o pipeline passa, o código pode ser mergeado.
- Branch principal: `main`. Branches de trabalho: `tipo/descricao` (ex: `feat/refresh-token`, `fix/cart-total`).

<aside>
🚫

**Proibido em código**

- Commits diretos na `main` sem pull request.
- Código sem lint passando.
- Arquivos `.env` commitados.
- `console.log` em produção. No backend, usar Pino. No mobile, desabilitar logs em builds de produção (usar `babel-plugin-transform-remove-console` ou equivalente).
- Comentários de código morto (código comentado). Remover e confiar no histórico do Git.
- Arquivos `.js` ou `.jsx` em novos projetos — usar `.ts` e `.tsx` exclusivamente.
</aside>

#### 5.7 Seed e Fixtures

- **Seed de desenvolvimento:** Todo produto deve ter um script de seed (`prisma/seed.ts`) que popula o banco local com dados mínimos para desenvolvimento. O seed deve criar: pelo menos 1 usuário admin com credenciais documentadas no `.env.example`, dados de exemplo suficientes para navegar pelas telas principais do produto, e dados que exercitem os edge cases mais comuns (listas vazias, listas com muitos itens, estados pendentes/concluídos).
- **Execução do seed:** `pnpm prisma db seed` deve funcionar em um banco limpo sem erros. O seed deve ser idempotente (rodar múltiplas vezes sem duplicar dados). Usar `upsert` do Prisma em vez de `create` para registros fixos.
- **Fixtures para testes:** Testes de integração devem ter suas próprias fixtures (dados de teste) isoladas do seed de desenvolvimento. Fixtures ficam em `src/modules/[dominio]/tests/fixtures/`. Cada teste deve ser responsável por criar e limpar seus próprios dados.
- **Reset do banco local:** Documentar no `README.md` o comando para resetar o banco local: `pnpm prisma migrate reset` (roda migrations do zero + seed).

#### 5.8 Migrations em Produção

- **Execução:** Migrations em produção devem rodar automaticamente no pipeline de CI/CD, antes do deploy do novo código. O step de migration no GitHub Actions deve rodar `pnpm prisma migrate deploy` conectado ao banco de produção do Supabase.
- **Rollback:** Prisma não suporta rollback automático de migrations. Para reverter uma migration em produção, criar uma nova migration que desfaz as alterações. Nunca editar ou deletar uma migration já aplicada.
- **Migrations destrutivas (DROP COLUMN, DROP TABLE, ALTER TYPE):** Devem ser executadas em duas etapas. Primeira etapa: deploy do código que não depende mais da coluna/tabela. Segunda etapa: migration que remove a coluna/tabela. Isso evita downtime se o deploy do código e a migration não forem atômicos.
- **Migrations em tabelas grandes (>1M rows):** Usar `CREATE INDEX CONCURRENTLY` para índices (via `$queryRaw` no Prisma). Para alterações de colunas, avaliar impacto de lock na tabela e executar em horário de baixo tráfego. Documentar no PR quando a migration envolve tabelas grandes.
- **Ambientes:** Testar toda migration no ambiente de desenvolvimento local e no staging (quando disponível) antes de aplicar em produção.

---

### 6. Testes

#### 6.1 Princípio

- Como todo código é gerado por agentes de IA, **testes automatizados são a principal rede de segurança** da Shift Labs. Não há revisão humana linha a linha — os testes são o que valida a correção do output gerado.
- A cobertura de testes deve ser tratada como **requisito de entrega**, não como melhoria futura.
- Todo novo produto deve nascer com testes de integração nos endpoints críticos e testes de componente nas screens críticas.

#### 6.2 Stack de Testes Aprovada

**Backend (NestJS)**

- **Testes unitários:** Vitest para lógica de negócio, services e utils.
- **Testes de integração:** Supertest + NestJS Testing Module para endpoints da API REST.

**Frontend Web**

- **Testes de componente:** Vitest + React Testing Library para componentes críticos.
- **Testes end-to-end:** Playwright para fluxos críticos do usuário.

**Mobile**

- **Testes de componente:** Jest + React Native Testing Library para componentes e screens críticos.
- **Testes end-to-end:** Maestro como ferramenta de E2E para apps mobile (iOS e Android).

#### 6.3 Cobertura Mínima Esperada (meta progressiva)

- **Curto prazo:** Testes de integração nos endpoints críticos de cada produto (pagamentos, autenticação, operações destrutivas).
- **Médio prazo:** Testes unitários para toda lógica de negócio do backend. Testes de componente para screens críticas no mobile.
- **Longo prazo:** Testes E2E para os 5 fluxos mais críticos de cada produto (web via Playwright, mobile via Maestro).

<aside>
🧪

**Prioridade crítica.** Em um ambiente onde todo código é gerado por IA, testes automatizados são a **única barreira confiável** contra regressões e bugs. Todo novo produto deve nascer com testes de integração nos endpoints críticos e testes de componente nas screens principais. Sem testes, não há como validar o output dos agentes.

</aside>

<aside>
🤖

**Testes de IA/LLM.** Módulos que utilizam LLM têm estratégia de testes própria (unit + integration + evals com golden datasets) definida na seção 11.10. A stack de testes desta seção (Vitest, Supertest) continua aplicável para a lógica determinística ao redor do LLM.

</aside>

#### 6.4 O Que Testar por Camada (checklist para agentes de IA)

Esta seção define o que o agente de IA deve gerar como testes para cada tipo de código. O objetivo é eliminar ambiguidade: o agente deve gerar estes cenários sem precisar de instrução adicional.

**Endpoint de criação (POST)**

- Input válido retorna 201 com o objeto criado.
- Input com campo obrigatório faltando retorna 400.
- Input com tipo errado (string onde deveria ser number) retorna 400.
- Input com valor duplicado em campo unique retorna 409.
- Requisição sem autenticação retorna 401.
- Requisição com usuário sem permissão retorna 403.
- Verificar que o registro foi realmente persistido no banco (query de verificação).

**Endpoint de leitura (GET)**

- Retorna 200 com dados corretos para ID válido.
- Retorna 404 para ID inexistente.
- Retorna 401 sem autenticação.
- Listagem retorna array com paginação correta.
- Filtros funcionam corretamente (cada filtro suportado).
- Ordenação funciona corretamente.
- Soft-deleted records não aparecem na listagem (quando aplicável).

**Endpoint de atualização (PATCH/PUT)**

- Input válido retorna 200 com o objeto atualizado.
- Atualização parcial (PATCH) altera apenas os campos enviados.
- Retorna 404 para ID inexistente.
- Retorna 400 para input inválido.
- Retorna 401 sem autenticação.
- Retorna 403 sem permissão.
- Verificar que `updated_at` foi atualizado.

**Endpoint de deleção (DELETE)**

- Retorna 200/204 para deleção bem-sucedida.
- Retorna 404 para ID inexistente.
- Retorna 401 sem autenticação.
- Retorna 403 sem permissão.
- Verificar que soft delete preencheu `deleted_at` (quando aplicável).
- Verificar que hard delete removeu o registro (quando aplicável).

**Lógica de negócio (services)**

- Cenário feliz (happy path) funciona.
- Edge cases documentados nos business rules.
- Valores limites (zero, negativo, máximo permitido).
- Estado inválido (ex: cancelar pedido já entregue).

**Componentes React/React Native (screens críticas)**

- Renderiza sem erros.
- Exibe loading state enquanto dados carregam.
- Exibe estado vazio quando não há dados.
- Exibe dados corretamente quando carregados.
- Exibe erro quando a API falha.
- Elementos interativos respondem a clique/toque.
- Formulários validam campos obrigatórios antes de submeter.

---

### 7. Deploy e CI/CD

#### 7.1 Backend

- **Docker é obrigatório** para ambiente de desenvolvimento local em todos os produtos.
- Todo serviço deve ter `Dockerfile` e `docker-compose.yml` para ambiente local (incluindo Redis e RabbitMQ).
- Imagens devem ser leves (multi-stage build quando aplicável).
- **Produção:** Deploy em serviço gerenciado (Railway, Render, VPS com Docker, ou equivalente). Definido por produto.

#### 7.2 Frontend Web

- **Landing pages e produtos públicos (Next.js):** Deploy em **Vercel** como padrão. Vercel é a plataforma nativa do Next.js, com preview deployments automáticos por PR, edge network global e integração nativa com GitHub.
- **Produtos internos/logados (React + Vite):** Deploy definido por produto (Vercel, Railway, VPS, ou equivalente).

#### 7.3 Banco de Dados

- **Supabase** gerencia PostgreSQL, Auth, Storage e Realtime em produção. Sem necessidade de gerenciar infraestrutura de banco.
- **Ambiente local:** Usar Supabase CLI (`supabase start`) para rodar instância local durante desenvolvimento, ou conectar direto ao projeto Supabase de desenvolvimento. Nota: o comando `supabase start` roda containers Docker internamente (PostgreSQL, GoTrue, Storage, Realtime, etc.). Portanto, Docker instalado é pré-requisito para o Supabase CLI local, e está em conformidade com a seção 7.1 (Docker obrigatório para ambiente local).

#### 7.4 Build e Distribuição (Mobile)

- **EAS Build** como padrão para gerar builds iOS e Android.
- **EAS Update** para OTA updates de JavaScript.
- **Perfis de build:** `development` (simulador/device local), `preview` (distribuição interna), `production` (stores).

#### 7.5 Pipeline de CI/CD

- **GitHub Actions** como padrão de CI/CD para todos os produtos.
- Pipeline mínimo obrigatório: lint → type-check → testes → build → deploy.
- Preview deployments automáticos por PR (Vercel faz nativamente para Next.js).

#### 7.6 Ambientes Obrigatórios

- **Backend:** Mínimo desenvolvimento local (Docker) + produção. Staging recomendado.
- **Frontend Web:** Mínimo desenvolvimento local + produção. Preview por PR obrigatório em projetos Vercel.
- **Mobile:** Mínimo development build (device local) + produção (stores). Preview build (distribuição interna) recomendado.
- **Supabase:** Mínimo 2 projetos — desenvolvimento e produção. Staging recomendado para produtos críticos.

<aside>
🚀

**Pendência:** Definir pipeline completo de GitHub Actions para backend e mobile. Responsável: Fernando Calado. Prazo sugerido: próximo sprint.

</aside>

---

### 8. Segurança

#### 8.1 Comunicação

- **HTTPS obrigatório** em todos os ambientes de produção e staging. Sem exceções.
- **CORS:** Configurado explicitamente em todo serviço backend via NestJS. Nunca usar `Access-Control-Allow-Origin: *` em produção.
- **Certificate pinning (mobile):** Recomendado para apps que lidam com dados financeiros ou sensíveis.

#### 8.2 Proteção de Dados

- **Sanitização de inputs:** Todo dado recebido do frontend/mobile deve ser validado e sanitizado no backend antes de processamento ou persistência. No NestJS, usar `class-validator` pipes. Nunca confiar em validação apenas do cliente.
- **SQL Injection:** O uso de Prisma com query builder tipado é a primeira camada de proteção. `$queryRaw` é permitido apenas com parâmetros preparados.
- **XSS (Cross-Site Scripting):** Nunca renderizar HTML não sanitizado. No React, evitar `dangerouslySetInnerHTML` exceto em casos controlados com sanitização explícita (ex: DOMPurify).
- **CSRF (Cross-Site Request Forgery):** Obrigatório em todo produto web que usa cookies para autenticação (incluindo refresh tokens em httpOnly cookies). Implementar proteção via token CSRF sincronizado. No NestJS, usar `@nestjs/csrf` ou middleware equivalente.
- **Row Level Security (Supabase):** Quando Supabase Auth for utilizado, RLS deve ser habilitado e configurado em todas as tabelas que contêm dados de usuários. Nunca expor a `service_role` key no frontend/mobile.

#### 8.3 Credentials e Secrets

- **Variáveis de ambiente obrigatórias** para toda credential, token, secret ou chave de API. Sem exceções.
- **Proibido:** Credentials, tokens, senhas ou chaves hardcoded no código-fonte, em comentários, ou em arquivos commitados.
- Arquivos `.env` devem estar no `.gitignore`. Um arquivo `.env.example` com as chaves (sem valores) deve existir em todo repositório (na raiz do monorepo e, quando necessário, em cada app).
- **Supabase:** Nunca expor a `service_role` key no frontend ou mobile. Usar apenas a `anon` key no cliente. A `service_role` key deve ser usada exclusivamente no backend.
- **Mobile:** Tokens de autenticação devem ser armazenados em `expo-secure-store`. Nunca em AsyncStorage.

**8.3.1 Nomenclatura de Variáveis de Ambiente**

- **Padrão:** `[SERVICO]_[RECURSO]_[ATRIBUTO]` em `UPPER_SNAKE_CASE`.
- Exemplos obrigatórios para todo produto:

```
# Supabase
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_DB_URL=

# Auth
JWT_SECRET=
JWT_ACCESS_EXPIRATION=
JWT_REFRESH_EXPIRATION=

# Redis
REDIS_URL=

# RabbitMQ
RABBITMQ_URL=

# Observabilidade
SENTRY_DSN=
POSTHOG_API_KEY=
POSTHOG_HOST=

# IA (quando aplicável)
OPENAI_API_KEY=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=

# App
NODE_ENV=
PORT=
APP_URL=
```

- Cada app no monorepo pode ter seu próprio `.env` e `.env.example`. Variáveis compartilhadas entre apps devem ter o mesmo nome e formato.
- Nunca usar nomes genéricos como `KEY`, `SECRET`, `TOKEN` sem prefixo de serviço.

#### 8.4 Rate Limiting

- Todo endpoint público deve ter **rate limiting** configurado para prevenir abuso.
- No NestJS, usar `@nestjs/throttler` como padrão.
- Endpoints de autenticação (login, registro, reset de senha) devem ter rate limiting mais restritivo.

#### 8.5 Headers de Segurança

- **Helmet.js obrigatório** em todo serviço NestJS para configurar headers de segurança automaticamente (X-Content-Type-Options, X-Frame-Options, Strict-Transport-Security, etc.). Deve ser adicionado como middleware desde o início do projeto.

#### 8.6 Error Tracking

- **Sentry** como plataforma obrigatória de error tracking em todos os produtos da Shift Labs.
- Integração obrigatória em todas as camadas: backend (NestJS via `@sentry/nestjs`), frontend web (React via `@sentry/react`, Next.js via `@sentry/nextjs`) e mobile (React Native/Expo via `@sentry/react-native`).
- Todo erro não tratado deve ser capturado automaticamente com stack trace, breadcrumbs (ações do usuário que precederam o erro) e contexto (user ID, rota, plataforma).
- **Source maps** devem ser enviados ao Sentry no build de produção para stack traces legíveis.
- **Performance monitoring** (tracing) habilitado em produção para identificar gargalos de latência.
- **Alertas:** Configurar alertas para novos erros e para spikes de erros acima da baseline.
- Sentry complementa os logs estruturados do Pino (seção 1.4). Pino é para logs operacionais detalhados. Sentry é para captura, agrupamento e alerta de erros em runtime.

<aside>
💡

**Por que Sentry?** Em um ambiente onde todo código é gerado por IA, erros em produção precisam ser detectados e diagnosticados rapidamente. Sentry fornece stack traces completos, breadcrumbs de contexto, agrupamento automático de erros similares e alertas. Tem SDKs nativos para todas as camadas da stack da Shift Labs (NestJS, React, Next.js, Expo).

</aside>

#### 8.7 Feature Flags

- **PostHog Feature Flags** como solução padrão para feature flags em todos os produtos.
- Obrigatório para rollout progressivo de novas features de IA (novos prompts, modelos, agentes).
- Recomendado para A/B testing de prompts e interfaces em produtos com base de usuários ativa.
- Usar como kill switch para features de IA em produção (desligar rapidamente se o modelo gerar outputs problemáticos).
- Integração via PostHog SDK (mesmo SDK do analytics, seção 9), sem dependência adicional.

<aside>
💡

**Por que Feature Flags?** Para sistemas com IA em todas as camadas, o comportamento do produto pode mudar significativamente com alterações de prompt ou modelo. Feature flags permitem rollout gradual, A/B testing e rollback instantâneo sem deploy.

</aside>

<aside>
🤖

**Segurança de IA/LLM.** Produtos com funcionalidades de IA têm requisitos adicionais de segurança: proteção contra prompt injection, rate limiting por usuário em endpoints de LLM, content filtering e masking de PII. Ver seção 11.9 para detalhes.

</aside>

<aside>
🚨

**Proibido em segurança**

- CORS com wildcard (`*`) em produção.
- Credentials no código-fonte ou em arquivos commitados.
- `$queryRaw` do Prisma sem parâmetros preparados.
- `dangerouslySetInnerHTML` sem sanitização.
- Endpoints públicos sem rate limiting.
- HTTP (sem TLS) em produção ou staging.
- Tokens de autenticação em AsyncStorage no mobile.
- Endpoints mutação (POST/PUT/PATCH/DELETE) em apps web com cookies sem proteção CSRF.
- `service_role` key do Supabase exposta no frontend ou mobile.
- Tabelas Supabase sem RLS quando Supabase Auth está em uso.
- Deploy de produto em produção sem Sentry configurado.
- Feature flags de IA sem kill switch configurado.
</aside>

---

### 9. Analytics e Tracking

#### 9.1 Diretriz Geral

- Todo produto da Shift Labs que tenha usuários finais deve ter **analytics implementado desde o lançamento**. Analytics não é opcional.
- O objetivo é ter dados mínimos para medir adoção, retenção e uso das funcionalidades principais.

#### 9.2 Ferramenta Padrão

- **PostHog** como ferramenta oficial de product analytics da Shift Labs.
- PostHog é open-source, pode ser self-hosted ou usado em cloud, e cobre product analytics, session replay, feature flags (seção 8.7) e A/B testing em uma única plataforma.
- SDKs disponíveis para todas as camadas da stack: React (`posthog-js`), Next.js (`posthog-js`), React Native (`posthog-react-native`), Node.js (`posthog-node`).
- **Deploy:** PostHog Cloud como padrão inicial. Self-hosted (via Docker) como opção quando necessário por compliance ou custo.
- **Integração com Sentry:** PostHog se integra nativamente com Sentry para correlacionar erros com comportamento do usuário.

<aside>
💡

**Por que PostHog?** Unifica analytics, session replay, feature flags e A/B testing em uma plataforma. Open-source com opção self-hosted. SDKs para toda a stack da Shift Labs. Elimina a necessidade de contratar ferramentas separadas para cada funcionalidade.

</aside>

#### 9.3 Padrão de Eventos (obrigatório independente da ferramenta)

- **Nomenclatura de eventos:** `snake_case` com formato `[objeto]_[ação]`. Exemplos: `button_clicked`, `order_created`, `screen_viewed`, `signup_completed`, `payment_failed`.
- **Propriedades obrigatórias em todo evento:** `event_name`, `timestamp`, `platform` (web, ios, android), `user_id` (quando autenticado), `screen_name`.
- **Eventos mínimos para todo produto:** `screen_viewed` (toda tela), `signup_completed`, `login_completed`, `core_action_completed` (a ação principal do produto, ex: pedido criado, contrato transferido).
- **Consistência cross-plataforma:** O mesmo evento deve ter o mesmo nome e as mesmas propriedades no web e no mobile. A única diferença deve ser o valor de `platform`.

#### 9.4 Implementação

- **Camada de abstração obrigatória.** Todo tracking deve passar por um service/wrapper centralizado (ex: `analytics.track('order_created', { ... })`). Nunca chamar a SDK da ferramenta diretamente nos componentes. Isso permite trocar de ferramenta sem refatorar toda a aplicação.
- No frontend web, o wrapper fica em `src/services/analytics.ts`.
- No mobile, o wrapper fica em `src/services/analytics.ts`.

#### 9.5 Privacidade

- **Consentimento:** Em produtos que operam sob LGPD, exibir aviso de coleta de dados e permitir opt-out quando exigido por lei.
- **Dados sensíveis:** Nunca enviar para analytics: senhas, tokens, dados de pagamento, CPF, ou qualquer PII (Personally Identifiable Information) além do user_id.

<aside>
🚫

**Proibido em analytics**

- Chamar SDK de analytics diretamente nos componentes (sem wrapper).
- Eventos sem nomenclatura padronizada (snake_case, objeto_ação).
- Enviar PII (CPF, email, senhas, dados de pagamento) para ferramentas de analytics.
- Lançar produto sem analytics mínimo configurado.
</aside>

<aside>
📁

**Pendência:** Configurar projeto PostHog (cloud ou self-hosted) e criar wrapper centralizado de analytics para o boilerplate oficial. Responsável: Fernando Calado.

</aside>

---

### 10. Pontos de Evolução Técnica

Esta seção consolida apenas o que **ainda não é padrão** mas está sendo avaliado como evolução. Itens que já são obrigatórios estão nas seções normativas acima.

<aside>
🚨

**Prioridade zero: Boilerplates e infraestrutura base.** Sem os boilerplates, cada novo projeto é configurado do zero pelo agente de IA, e a consistência entre produtos diverge. Sem CI/CD, o princípio de "testes como rede de segurança" não funciona. Sem Sentry e PostHog configurados, as seções 8.6, 8.7 e 9 são letra morta. Estes são os itens que desbloqueiam todos os outros.

</aside>

#### 10.1 Prioridade Zero — Infraestrutura Base (bloqueia tudo)

| **Item** | **Descrição** | **Responsável** | **Status** |
| --- | --- | --- | --- |
| Boilerplate monorepo | Template Turborepo + pnpm workspaces com `apps/` + `packages/` + configs compartilhadas (ESLint, TypeScript). **Artefato crítico para IA.** Ponto de partida de todo novo produto | Fernando Calado | Pendente |
| Boilerplate backend (app) | Template NestJS + Prisma + Supabase + Redis + RabbitMQ + Pino + Helmet + Sentry + Docker, como app dentro do monorepo. **Artefato crítico para IA** | Fernando Calado | Pendente |
| Boilerplate web Next.js (app) | Template Next.js + TypeScript + Tailwind CSS v4 + shadcn/ui + TanStack Query + Zustand + Framer Motion + PostHog + Sentry + date-fns + tokens + lint, como app dentro do monorepo. **Artefato crítico para IA** | Fernando Calado | Pendente |
| Boilerplate web Vite (app) | Template React + Vite + TypeScript + Tailwind CSS v4 + shadcn/ui + TanStack Router + TanStack Query + Zustand + Framer Motion + PostHog + Sentry + date-fns + tokens + lint, como app dentro do monorepo. **Artefato crítico para IA** | Fernando Calado | Pendente |
| Boilerplate mobile (app) | Template Expo + expo-router + TypeScript + Reanimated + TanStack Query + Zustand + PostHog + Sentry + date-fns + tokens + lint, como app dentro do monorepo. **Artefato crítico para IA** | Fernando Calado | Pendente |
| Projeto Sentry | Criar organização Sentry, projetos por produto, DSN disponível para os boilerplates. Sem isso, a seção 8.6 não funciona | Fernando Calado | Pendente |
| Projeto PostHog | Criar projeto PostHog (cloud ou self-hosted), API key disponível, wrapper centralizado de analytics para os boilerplates. Sem isso, as seções 8.7 e 9 não funcionam | Fernando Calado | Pendente |
| Pipeline CI/CD (GitHub Actions) | Pipeline mínimo: lint → type-check → testes → prisma migrate deploy → build → deploy. Testar com o boilerplate monorepo. Sem isso, o princípio "testes validam" da seção 6 não funciona | Fernando Calado | Pendente |

#### 10.2 Curto Prazo (próximos sprints)

**Frontend Web**

- **Framer Motion (Motion):** Adotar nos produtos ativos que ainda não utilizam.
- **Motion tokens:** Criar arquivo padrão de variáveis CSS via `@theme` conforme seção 2.4.3. Incluir no boilerplate de novos projetos.
- **Design tokens visuais:** Formalizar sistema consistente de tokens (cores, tipografia, espaçamentos) como padrão cross-produto e cross-plataforma.
- **Auditoria de acessibilidade:** Revisar produtos ativos contra as diretrizes da seção 2.5.

**Mobile**

- **Setup do primeiro projeto:** Configurar boilerplate React Native + Expo + expo-router + TypeScript + Reanimated + Zustand + TanStack Query usando o template monorepo.
- **Design tokens mobile:** Replicar tokens do web como constantes TypeScript.

**Backend**

- **Migração de produtos ativos:** Planejar migração de Express → NestJS e TypeORM → Prisma para produtos existentes, priorizando por risco/impacto.

**Tooling**

- **Avaliar Biome como substituto de ESLint + Prettier.** Biome faz lint + format em uma única ferramenta, é ordens de magnitude mais rápido, e tem adoção crescente. Avaliar viabilidade de migração mantendo compatibilidade com as regras existentes. Responsável: Fernando Calado.

#### 10.3 Médio Prazo (próximo trimestre)

**Frontend Web**

- **Efeitos visuais avançados:** Glassmorphism, scroll-driven animations e skeleton screens como padrão de UX.

**Backend**

- **Testes:** Todo produto ativo deve ter testes de integração nos endpoints críticos.

**IA/LLM**

- **Boilerplate do módulo de IA:** Template NestJS + OpenAI SDK + Langfuse + pgvector, como módulo dentro do app backend do monorepo. **Artefato crítico para IA.** Responsável: Fernando Calado.
- **Golden datasets:** Criar primeiros golden datasets de eval para os produtos com IA ativa (Apolo, Atena). Mínimo 20 a 50 exemplos por domínio.
- **Threshold de custo:** Definir threshold de custo diário por produto para alertas de observabilidade (Langfuse).

#### 10.4 Longo Prazo (backlog estratégico)

**Frontend Web**

- **Testes E2E:** Playwright nos fluxos críticos de cada produto web.

**Mobile**

- **Testes E2E:** Maestro nos fluxos críticos de cada produto mobile.
- **Offline-first:** Definir estratégia padrão de cache e sincronização offline quando necessário.

**Infraestrutura**

- **Monitoramento:** Evoluir de logs (Pino) + error tracking (Sentry) para observabilidade completa (métricas, traces, alertas integrados).

---

### 11. IA, LLM e Agentes

Esta seção define o padrão técnico para produtos da Shift Labs que utilizam **inteligência artificial, modelos de linguagem (LLM) e agentes autônomos** como funcionalidade do produto — ou seja, IA como feature entregue ao usuário final, não apenas como ferramenta de desenvolvimento.

<aside>
🧠

**Escopo desta seção.** As seções anteriores definem como a Shift Labs **desenvolve** software (com agentes de IA gerando código). Esta seção define como a Shift Labs **constrói produtos que usam IA** — chatbots, assistentes, agentes autônomos, pipelines de RAG, classificadores, extratores de dados, etc. São camadas complementares: o código do produto de IA é gerado por Claude Code/GPT Codex (seções 1-10), mas a funcionalidade de IA do produto segue os padrões definidos aqui.

</aside>

#### 11.1 Modelo de Linguagem Padrão

- **LLM padrão:** **GPT-4** (OpenAI) como modelo principal para todos os produtos da Shift Labs que utilizam LLM.
- **SDK:** **OpenAI SDK for Node.js** (`openai`) como cliente padrão para comunicação com a API da OpenAI. Integração nativa com TypeScript.
- **Modelos secundários:** Outros modelos (Claude, Gemini, modelos open-source) podem ser adotados por produto com justificativa documentada via ADR. O GPT-4 é o default inicial.
- **Versionamento de modelo:** Sempre fixar a versão do modelo em produção (ex: `gpt-4-turbo-2024-04-09`). Nunca usar aliases genéricos (`gpt-4`) em produção — apenas em desenvolvimento.
- **Fallback:** Todo produto deve ter estratégia de fallback documentada para indisponibilidade da API da OpenAI (retry com backoff exponencial, modelo secundário, ou degradação graceful).
- **Otimização de custo (model tiers):** Para tarefas simples e determinísticas (classificação, extração com schema fixo, sumarização curta), avaliar uso de modelos mais leves (GPT-4o-mini ou equivalente) com justificativa documentada por produto. GPT-4 continua como padrão para tarefas complexas (agentes, geração longa, raciocínio multi-step).

<aside>
💡

**Por que GPT-4?** Melhor equilíbrio entre qualidade de output, suporte a function calling, ecossistema maduro e estabilidade da API. O OpenAI SDK é extremamente bem representado no treinamento dos agentes de IA que geram nosso código, o que maximiza a qualidade do código gerado.

</aside>

#### 11.2 Orquestração de Agentes

**11.2.1 Framework Padrão**

- **Vercel AI SDK** como framework padrão de orquestração para produtos que precisam de interação LLM no frontend (chat, streaming, assistentes). Integração nativa com Next.js, React e Node.js.
- **LangChain.js** como framework aprovado para pipelines complexos de backend: chains, RAG, agentes com ferramentas, roteamento de prompts, memory management.
- **Orquestração customizada via NestJS:** Permitida para agentes simples que não justifiquem a complexidade do LangChain. Implementar como módulo NestJS dedicado (`src/modules/ai/`).

**11.2.2 Critério de Decisão**

| **Cenário** | **Framework** | **Quando usar** |
| --- | --- | --- |
| Chat/assistente com streaming no frontend | **Vercel AI SDK** | Produto com interface de chat, respostas em tempo real, integração React/Next.js |
| Pipeline RAG, agentes com tools, chains complexas | **LangChain.js** | Backend que precisa de retrieval, roteamento, memory, múltiplas tools |
| Chamada direta à API (classificação, extração, sumarização) | **OpenAI SDK direto** | Tarefas simples sem orquestração: um prompt, uma resposta |
| Agente autônomo com lógica de negócio complexa | **LangGraph.js** | Agentes com fluxo de decisão stateful, loops, checkpoints e branching |

#### 11.3 RAG (Retrieval-Augmented Generation)

**11.3.1 Stack de RAG**

- **Vector Store:** **Supabase pgvector** como padrão. Aproveita o PostgreSQL já utilizado pela Shift Labs, sem adicionar infraestrutura externa. A extensão `pgvector` habilita busca por similaridade vetorial diretamente no Supabase.
- **Embeddings:** **OpenAI Embeddings** (`text-embedding-3-small` como default, `text-embedding-3-large` para casos que exigem maior precisão). Modelo definido por produto.
- **Chunking:** Implementar estratégia de chunking por produto, documentada no PRD. Recomendações padrão: chunks de 512-1024 tokens com overlap de 10-20%. Usar separadores semânticos (parágrafos, seções) em vez de corte por tamanho fixo quando possível.
- **Reranking:** Opcional. Quando necessário, usar **Cohere Rerank** ou reranking via LLM como segunda camada de relevância após a busca vetorial.

**11.3.2 Pipeline de Ingestão**

- Documentos devem ser processados via job assíncrono (RabbitMQ) para não bloquear a API.
- Cada chunk armazenado deve conter: `content` (texto), `embedding` (vetor), `metadata` (source, page, section, timestamp).
- Toda ingestão deve ser idempotente — re-processar um documento atualiza os chunks existentes, não duplica.
- Índice vetorial deve usar **`hnsw`** como padrão no pgvector (melhor performance de busca, boa relação custo-benefício). `ivfflat` como alternativa para datasets muito grandes (>1M vetores) onde o tempo de construção do índice é crítico. Configurar conforme volume de dados do produto.

<aside>
💡

**Por que Supabase pgvector?** Elimina a necessidade de um serviço de vector store separado (Pinecone, Qdrant, Weaviate). Os embeddings ficam no mesmo PostgreSQL que o resto dos dados, simplificando queries híbridas (vetorial + filtros SQL), backups e permissões. O Prisma pode coexistir com queries pgvector via `$queryRaw`.

</aside>

#### 11.4 Prompt Engineering

**11.4.1 Gerenciamento de Prompts**

- **Prompts como código.** Todos os prompts de produção devem ser armazenados em arquivos TypeScript no repositório, nunca hardcoded inline nas chamadas à API.
- **Estrutura de pastas:**

```jsx
src/
  modules/
    ai/
      prompts/
        [dominio]/
          [dominio]-system.prompt.ts
          [dominio]-user.prompt.ts
          [dominio]-tools.ts
      ai.module.ts
      ai.service.ts
      ai.config.ts
```

- **Template engine:** Usar template literals do TypeScript com tipagem forte para variáveis de prompt. Cada prompt deve exportar uma função tipada que recebe as variáveis e retorna a string final.
- **Versionamento:** Prompts são versionados junto com o código via Git. Mudanças significativas de prompt devem ter seu próprio commit com descrição clara do que mudou e por quê.

**11.4.2 Boas Práticas Obrigatórias**

- **System prompt separado do user prompt.** Sempre usar a separação `system` / `user` / `assistant` da API.
- **Instruções explícitas de formato.** Todo prompt que espera output estruturado deve incluir instruções claras de formato (JSON schema, exemplos).
- **Few-shot examples.** Para tarefas de classificação, extração ou formatação, incluir 2-3 exemplos no prompt.
- **Guardrails no prompt.** Incluir instruções explícitas sobre o que o modelo NÃO deve fazer (ex: "Não invente informações", "Se não souber, diga que não sabe").
- **Structured Outputs (obrigatório para tarefas determinísticas).** Para toda tarefa que exige output JSON tipado (classificação, extração, formatação estruturada), usar `response_format: { type: "json_schema" }` da API da OpenAI com schema definido. Elimina parsing manual e garante type-safety no output do LLM. Structured Outputs é obrigatório nessas tarefas, não opcional.

#### 11.5 Function Calling e Tools

- **Function calling** (tool use) do GPT-4 como mecanismo padrão para agentes que precisam executar ações (consultar banco, chamar APIs externas, manipular dados).
- **Definição de tools:** Cada tool deve ser definida como um objeto TypeScript tipado com `name`, `description` e `parameters` (JSON Schema). Armazenadas em `src/modules/ai/prompts/[dominio]/[dominio]-tools.ts`.
- **Execução segura:** Toda tool que executa side effects (escrita em banco, chamada a API externa, envio de email) deve ter validação de permissões e rate limiting independente.
- **Logging:** Toda execução de tool deve ser logada com input, output, duração e status (sucesso/erro).

#### 11.6 Streaming

- **Server-Sent Events (SSE)** como padrão para streaming de respostas LLM do backend para o frontend.
- **Vercel AI SDK** gerencia streaming nativamente em produtos Next.js. Para produtos React + Vite, usar a API de streaming do Vercel AI SDK no backend NestJS com SSE endpoint.
- **Indicadores de loading:** No frontend, exibir indicador de "digitando" enquanto o stream está ativo. Renderizar tokens incrementalmente conforme chegam.
- **Timeout:** Configurar timeout de stream por produto (recomendado: 60-120 segundos). Exibir mensagem de fallback se o stream não iniciar em 10 segundos.
- **WebSockets:** Permitidos como alternativa ao SSE quando o produto já usa Supabase Realtime ou quando a comunicação precisa ser bidirecional, com justificativa documentada.

#### 11.7 Cache de LLM

- **Semantic cache:** Recomendado para produtos com alto volume de chamadas repetitivas. Usar Redis + embeddings para identificar queries semanticamente similares e retornar respostas cacheadas.
- **Exact cache:** Obrigatório para chamadas determinísticas (mesma entrada sempre produz mesma saída, ex: classificações com temperature 0). Implementar via Redis com TTL definido por produto.
- **Cache key:** Compor a chave de cache com: modelo, versão do prompt, hash do input e parâmetros relevantes (temperature, max_tokens).
- **Invalidação:** Toda mudança de prompt ou modelo deve invalidar o cache correspondente. Documentar estratégia de invalidação por produto.

#### 11.8 Observabilidade de IA

**11.8.1 Ferramenta Padrão**

- **Langfuse** como plataforma padrão de observabilidade para chamadas LLM. Open-source, self-hosted ou cloud. Tracing completo de chamadas, custo, latência, tokens e qualidade.
- Integração nativa com OpenAI SDK, LangChain.js e Vercel AI SDK.

**11.8.2 O que Monitorar (obrigatório)**

- **Latência:** Tempo de resposta de cada chamada LLM (p50, p95, p99).
- **Custo:** Tokens consumidos (input + output) por chamada, por usuário e por produto. Alerta configurado para gastos acima do threshold definido.
- **Erros:** Taxa de erro da API, timeouts, rate limits atingidos.
- **Qualidade:** Feedback do usuário (thumbs up/down) quando aplicável. Scores de avaliação automática quando configurados.
- **Traces:** Trace completo de pipelines multi-step (RAG: query → embedding → retrieval → reranking → generation).

**11.8.3 Alertas Obrigatórios**

- Custo diário acima de 2x a média dos últimos 7 dias.
- Taxa de erro acima de 5% em janela de 1 hora.
- Latência p95 acima de 30 segundos.

#### 11.9 Segurança de IA

**11.9.1 Prompt Injection**

- **Sanitização de input do usuário obrigatória.** Todo texto fornecido pelo usuário que será inserido em um prompt deve ser tratado como dado não-confiável.
- **Separação clara entre instruções e dados.** Usar delimitadores explícitos (ex: `<user_input>...</user_input>`) para isolar o input do usuário dentro do prompt.
- **Validação de output.** Para agentes com function calling, validar que as tools chamadas e os parâmetros são permitidos antes de executar.

**11.9.2 Limites e Proteção**

- **Rate limiting por usuário:** Todo endpoint que chama LLM deve ter rate limiting por usuário autenticado, independente do rate limiting global. Recomendado: 20-60 chamadas/minuto por usuário (definido por produto).
- **Limite de tokens:** Configurar `max_tokens` em toda chamada à API. Nunca deixar ilimitado.
- **Content filtering:** Usar a moderação da OpenAI (`/v1/moderations`) para filtrar inputs e outputs em produtos voltados ao público geral.
- **PII em prompts:** Nunca enviar dados sensíveis desnecessários (CPF, senhas, dados de pagamento) para a API do LLM. Mascarar ou remover antes de incluir no prompt.

**11.9.3 API Keys**

- API keys da OpenAI (e de qualquer provider de LLM) seguem as mesmas regras da seção 8.3 — variáveis de ambiente, nunca no código, nunca no frontend/mobile.
- **Chamadas ao LLM sempre via backend.** O frontend/mobile nunca deve chamar a API da OpenAI diretamente. Toda chamada passa pelo backend NestJS, que gerencia autenticação, rate limiting e logging.

#### 11.10 Testes de IA

**11.10.1 Desafio**

Outputs de LLM são não-determinísticos por natureza. Testes tradicionais (assert exato) não funcionam para validar respostas de IA. A estratégia de testes deve ser adaptada.

**11.10.2 Estratégia por Camada**

- **Testes unitários (determinísticos):** Testar toda lógica ao redor do LLM — parsing de respostas, construção de prompts, validação de tools, chunking, formatação de contexto. Estes são determinísticos e devem usar Vitest como o restante do backend.
- **Testes de integração (semi-determinísticos):** Testar o pipeline completo (prompt → LLM → parsing → ação) com temperature 0 e seed fixo quando possível. Validar estrutura do output (JSON válido, campos obrigatórios presentes) em vez de conteúdo exato.
- **Evals (avaliação de qualidade):** Para cada produto, manter um **golden dataset** — conjunto de inputs com outputs esperados, revisados manualmente. Rodar evals periodicamente (não no CI) para medir regressão de qualidade após mudanças de prompt ou modelo.

**11.10.3 Ferramentas de Eval**

- **Langfuse Evals** como padrão para avaliação automatizada de qualidade.
- **Métricas de eval recomendadas:** Faithfulness (resposta fiel ao contexto), relevance (resposta relevante à pergunta), correctness (resposta factualmente correta), format compliance (output no formato esperado).
- **Golden datasets:** Mínimo 20-50 exemplos por domínio/feature. Armazenados no repositório em `tests/evals/[dominio]/`.

#### 11.11 Estrutura de Pastas (módulo de IA — NestJS)

```jsx
src/
  modules/
    ai/
      ai.module.ts           # Módulo NestJS de IA
      ai.service.ts          # Service principal (orquestração)
      ai.config.ts           # Configurações (modelos, tokens, timeouts)
      llm/
        llm.service.ts       # Wrapper do OpenAI SDK
        llm.types.ts         # Types para chamadas LLM
      rag/
        rag.service.ts       # Pipeline de RAG
        rag.ingest.ts        # Ingestão e chunking
        rag.retrieve.ts      # Busca vetorial + reranking
      prompts/
        [dominio]/
          [dominio]-system.prompt.ts
          [dominio]-user.prompt.ts
          [dominio]-tools.ts
      agents/
        [agent-name]/
          [agent-name].agent.ts    # Lógica do agente
          [agent-name].tools.ts    # Tools específicas
          [agent-name].types.ts    # Types do agente
      cache/
        ai-cache.service.ts  # Cache de respostas LLM
      observability/
        langfuse.service.ts  # Integração Langfuse
      tests/
        evals/               # Golden datasets e eval scripts
```

#### 11.12 Versões Mínimas das Dependências (IA)

- **OpenAI SDK:** 4.x+
- **LangChain.js:** 0.3.x+
- **LangGraph.js:** 0.2.x+
- **Vercel AI SDK:** 4.x+
- **Langfuse JS SDK:** 3.x+
- **pgvector (Supabase):** 0.7+

<aside>
🚫

**Proibido em IA/LLM**

- Chamadas à API de LLM diretamente do frontend ou mobile. Toda chamada passa pelo backend.
- API keys de LLM no código-fonte, frontend ou mobile.
- Prompts hardcoded inline nas chamadas à API. Usar arquivos de prompt tipados.
- Chamadas ao LLM sem `max_tokens` configurado.
- Envio de PII desnecessária (CPF, senhas, dados de pagamento) para a API do LLM.
- Deploy de features de IA sem observabilidade (Langfuse) configurada.
- Vector stores externos (Pinecone, Qdrant, Weaviate) sem aprovação via ADR — usar Supabase pgvector.
- Frameworks de orquestração Python (LangChain Python, CrewAI, AutoGen) — a stack é 100% TypeScript/Node.js.
- Endpoints de IA sem rate limiting por usuário.
- `temperature` acima de 0 em tarefas de classificação, extração ou formatação estruturada.
- Tarefas determinísticas (classificação, extração, formatação) sem Structured Outputs (`response_format: { type: "json_schema" }`). Parsing manual de JSON é proibido quando Structured Outputs está disponível.
</aside>

<aside>
📁

**Pendências:**

- Definir threshold de custo diário por produto para alertas de observabilidade. Responsável: Fernando Calado.
- Criar boilerplate do módulo de IA (NestJS + OpenAI SDK + Langfuse + pgvector). **Artefato crítico para IA.** Responsável: Fernando Calado.
- Avaliar adoção de **Vercel AI SDK** vs. implementação customizada de streaming para produtos React + Vite. Responsável: Fernando Calado.
</aside>

---

### 12. Governança deste Documento

#### 12.1 Revisão

- Este documento deve ser revisado **trimestralmente** ou sempre que houver mudança significativa de stack.
- O responsável pela manutenção é o líder técnico da Shift Labs.

#### 12.2 Processo de Desvio (ADR)

- Qualquer desvio do padrão definido neste documento exige um **Architecture Decision Record (ADR)**: justificativa técnica escrita, análise de impacto (manutenção, contratação, compatibilidade) e aprovação do responsável técnico.
- ADRs devem ser registrados em `docs/adrs/ADR-XXX-titulo.md` no repositório do produto, revisados pelo responsável técnico em até 48h.
- Desvios aprovados devem ser referenciados neste documento com data e justificativa.

#### 12.3 Checklist de Atualidade da Stack (trimestral)

A cada revisão trimestral, responder estas 3 perguntas para cada item da stack:

1. **Gargalo?** — Este item virou gargalo de contratação, manutenção ou performance?
2. **Risco?** — Este item virou risco (segurança, suporte da comunidade, depreciação)?
3. **Ganho claro?** — Existe ganho mensurável em trocar (tempo de entrega, bugs, custo)?

Se a resposta for "sim" para qualquer pergunta, abrir ADR de avaliação.

#### 12.4 Checklist de Novo Projeto (do zero ao primeiro deploy)

Este checklist é o passo a passo operacional para criar um novo produto da Shift Labs. Deve ser seguido na ordem.

**Fase 1 — Setup do repositório**

- [ ] Criar repositório no GitHub: `[produto]` (nome do produto em kebab-case)
- [ ] Clonar boilerplate monorepo (Turborepo + pnpm + apps/ + packages/)
- [ ] Renomear packages e apps para o produto
- [ ] Configurar `pnpm-workspace.yaml` e `turbo.json`
- [ ] Criar `.env.example` com todas as variáveis obrigatórias (seção 8.3.1)
- [ ] Primeiro commit: `chore(setup): init [produto]`

**Fase 2 — Infraestrutura**

- [ ] Criar projeto Supabase (desenvolvimento + produção)
- [ ] Configurar schema Prisma inicial com colunas de auditoria (seção 1.5)
- [ ] Rodar primeira migration: `pnpm prisma migrate dev`
- [ ] Criar seed inicial com usuário admin e dados de exemplo (seção 5.7)
- [ ] Configurar Redis (local via Docker, produção via Upstash ou equivalente)
- [ ] Configurar RabbitMQ se necessário (local via Docker, produção via CloudAMQP)

**Fase 3 — Observabilidade**

- [ ] Criar projeto no Sentry para cada app (api, web, mobile)
- [ ] Configurar Sentry DSN no `.env` de cada app
- [ ] Criar projeto PostHog (ou usar o existente da ShiftLabs)
- [ ] Configurar PostHog API key no `.env` de cada app frontend/mobile
- [ ] Configurar wrapper de analytics (`src/services/analytics.ts`)
- [ ] Configurar Langfuse se o produto tiver features de IA

**Fase 4 — CI/CD**

- [ ] Configurar GitHub Actions: lint → type-check → testes → build
- [ ] Configurar migration automática em produção no pipeline
- [ ] Configurar preview deploys (Vercel para Next.js, ou equivalente)
- [ ] Configurar deploy de produção (Vercel, Railway, VPS)

**Fase 5 — Primeiro deploy**

- [ ] Rodar pipeline completo localmente: `turbo lint type-check test build`
- [ ] Deploy do backend em produção
- [ ] Deploy do frontend em produção
- [ ] Verificar Sentry recebendo eventos
- [ ] Verificar PostHog recebendo analytics
- [ ] Verificar seed de produção (se aplicável)
- [ ] Smoke test manual nos fluxos críticos

**Fase 6 — Documentação mínima**

- [ ] README.md atualizado com: descrição do produto, como rodar localmente, como rodar testes, variáveis de ambiente necessárias
- [ ] Swagger/OpenAPI acessível na URL do backend
- [ ] ADR-001 documentado se houver qualquer desvio do padrão deste documento

#### 12.5 Backup e Disaster Recovery

**12.5.1 Banco de Dados (Supabase)**

- **Backups automáticos:** Supabase Pro realiza backups diários automáticos com retenção de 7 dias. Verificar que o plano do projeto inclui backups. Projetos no plano Free não têm backups automáticos e não devem ser usados em produção.
- **Point-in-time recovery (PITR):** Disponível no plano Pro. Permite restaurar o banco para qualquer momento nos últimos 7 dias. Ideal para recuperação de dados deletados acidentalmente ou migrations destrutivas.
- **Teste de restauração:** A cada trimestre, testar restauração de backup em um projeto Supabase de teste. Documentar o resultado. Backup que não é testado não é backup.
- **Export periódico:** Realizar `pg_dump` mensal do banco de produção e armazenar em local independente do Supabase (ex: bucket S3, Google Cloud Storage). Isso mitiga o risco de vendor lock-in e de perda de acesso à plataforma.

**12.5.2 Arquivos (Supabase Storage)**

- Supabase Storage armazena arquivos nos buckets configurados. Backups de storage não são cobertos pelo backup de banco.
- Para produtos com arquivos críticos (contratos, documentos legais), configurar replicação para bucket externo (S3 ou equivalente) via job periódico ou trigger de upload.

**12.5.3 Código**

- GitHub é o backup primário do código. Repositórios no GitHub têm redundância geográfica automática.
- Recomendado: habilitar **GitHub Archive** ou mirror em um segundo serviço de Git como medida adicional.

**12.5.4 Plano de Indisponibilidade**

- **Se Supabase ficar fora do ar:** O produto fica indisponível para operações de banco, auth e storage. Não existe failover automático. Comunicar aos usuários via canal secundário (email, redes sociais). Monitorar status em status.supabase.com.
- **Se Vercel ficar fora do ar:** Produtos Next.js ficam indisponíveis. Produtos React+Vite em outro host não são afetados. Monitorar status em vercel.com/status.
- **Se OpenAI ficar fora do ar:** Features de IA ficam indisponíveis. Implementar degradação graceful (seção 11.1): exibir mensagem ao usuário, desabilitar features de IA via PostHog Feature Flags, retornar resposta padrão quando possível.

<aside>
💡

**Sobre vendor lock-in:** A stack da Shift Labs depende fortemente de Supabase. Isso é uma escolha consciente para reduzir complexidade operacional. O risco é mitigado por: (1) o banco é PostgreSQL padrão e pode ser migrado para qualquer host, (2) o Prisma abstrai a camada de acesso a dados, (3) exports periódicos via `pg_dump` mantêm cópia independente. A revisão trimestral (seção 12.3) deve avaliar este risco a cada ciclo.

</aside>

#### 12.6 Histórico de Versões

- **v1.0** (09/01/2026): Versão inicial focada no Menux.
- **v1.1** (23/02/2026): Adicionadas seções de comunicação e versões de dependências.
- **v1.2** (23/02/2026): Adicionada seção de Motion/Animações.
- **v2.0** (23/02/2026): Reposicionamento como padrão oficial da Shift Labs. Adição de seções de testes, deploy, repositório, governança. Regras normativas.
- **v2.1** (23/02/2026): Adição de Segurança, Acessibilidade, padrão JWT, estrutura de erros, versionamento de API.
- **v2.2** (23/02/2026): Seção de Repositório e Estrutura de Código completa.
- **v3.0** (23/02/2026): Adição da seção 3 (Mobile). React Native + Expo como padrão. TypeScript obrigatório no mobile. React Navigation, Reanimated, Gesture Handler, Zustand, expo-secure-store. Estrutura de pastas mobile. Testes mobile. Build e distribuição via EAS. Acessibilidade mobile.
- **v3.1** (23/02/2026): Revisão completa. Correção de typos. Helmet.js obrigatório.
- **v4.0** (23/02/2026): Adição de Analytics e Tracking, Push Notifications, Internacionalização, Imagens e Assets mobile, proteção CSRF.
- **v5.0** (08/03/2026): **Atualização maior da stack.** Backend: Express → NestJS. ORM: TypeORM → Prisma (único aprovado). Banco: PostgreSQL em Docker → Supabase (PostgreSQL gerenciado + Auth + Storage + Realtime). Frontend: dois caminhos — Next.js (público/SEO, deploy em Vercel) e React + Vite (interno/logado). TypeScript strict obrigatório em todas as camadas (inclusive frontend web). Mobile: React Navigation → expo-router (file-based routing). Estado: TanStack Query como padrão obrigatório para dados do servidor + Zustand para estado global. Padrões de resposta de API unificados (success/error). ADR como processo formal de desvio. Checklist trimestral de atualidade da stack. Supabase RLS e service_role key nas regras de segurança. GitHub Actions como CI/CD padrão. Estruturas de pastas atualizadas para NestJS, Next.js e expo-router.
- **v5.1** (12/03/2026): **Princípio de desenvolvimento por IA.** Adicionado princípio fundacional: todo código é gerado por agentes de IA (Claude Code + GPT Codex). Callout 🤖 de desenvolvimento por IA no cabeçalho. TL;DR atualizado. Seção de Code Review recontextualizada para rastreabilidade (não aprovação humana). Testes elevados a prioridade crítica como principal rede de segurança. Boilerplates marcados como artefatos críticos para consistência dos agentes.
- **v5.2** (12/03/2026): **Tailwind CSS + shadcn/ui como padrão.** CSS vanilla substituído por Tailwind CSS. shadcn/ui adotado como sistema de componentes padrão (ownership de código via Radix Primitives). Estruturas de pastas atualizadas com `components/ui/`, `lib/utils.ts`, `tailwind.config.ts` e `components.json`. Versões de dependências atualizadas. Boilerplates web atualizados. Callout de proibição ajustado.
- **v6.0** (15/03/2026): **Seção 11 — IA, LLM e Agentes.** GPT-4 como LLM padrão. OpenAI SDK como cliente. Vercel AI SDK para streaming/chat. LangChain.js e LangGraph.js para orquestração. RAG com Supabase pgvector + OpenAI Embeddings. Prompt engineering como código (arquivos TypeScript tipados). Function calling com validação. SSE para streaming. Cache de LLM (exact + semantic via Redis). Langfuse como observabilidade obrigatória. Segurança de IA (prompt injection, rate limiting por usuário, content filtering, PII masking). Testes de IA (unit + integration + evals com golden datasets). Estrutura de pastas do módulo de IA. Versões mínimas das dependências de IA.
- **v6.1** (15/03/2026): **Auditoria profunda.** Renumeração de seções (IA → 11, Governança → 12). Escopo atualizado para incluir IA/LLM. Histórico de versões reordenado cronologicamente. Model tiers para otimização de custo. Structured Outputs como prática obrigatória. HNSW como índice vetorial padrão. Cross-references entre seções de IA e seções técnicas existentes (testes, segurança, comunicação, evolução). Itens de evolução de IA adicionados à seção 10.
- **v7.1** (22/03/2026): **Nomenclatura de repositórios.** Convenção de nomes de repositório ajustada de `shiftlabs-[produto]` para `[produto]` em kebab-case (ex: `repasse-seguro`, `menux`, `flow`). Atualizado em seção 5.1, estrutura do monorepo, proibições e checklist de novo projeto (12.4).
- **v7.0** (22/03/2026): **Atualização maior. Completude para autonomia total dos agentes de IA.** Callout "Duas camadas de IA" no cabeçalho. Glossário de Decisões Arquiteturais. **Monorepo com Turborepo** — seção 5.1 reescrita com `apps/` + `packages/`, pnpm, Turborepo. Tailwind v4: referências migradas para `@theme`/`postcss.config.mjs`. TanStack Router padrão para React+Vite. Critério SSE vs Supabase Realtime vs WebSocket (4.1.1). Auth end-to-end com interceptor de refresh (4.4.1). Paginação cursor/offset (4.6). Error handling frontend/mobile (4.7). Upload via signed URL (4.8). Logging frontend/mobile (4.9). Data/hora/timezone UTC + date-fns (4.10). Filtros, ordenação, busca e bulk operations em APIs (4.11). Convenções de banco de dados Prisma: UUID, auditoria, soft delete, relacionamentos, nomenclatura, índices (1.5). Design tokens concretos: cores shadcn/ui, espaçamento base 4px, escala tipográfica, border radius, sombras (2.3.1). Performance e Web Vitals: LCP, CLS, INP, bundle budget, code splitting, imagens (2.7). Checklist de testes por camada para agentes de IA (6.4). Seed e fixtures (5.7). Migrations em produção (5.8). Nomenclatura de env vars (8.3.1). Sentry obrigatório (8.6). PostHog Feature Flags (8.7). PostHog analytics oficial (9.2). Seção 10 com Prioridade Zero (boilerplates, Sentry, PostHog, CI/CD). Structured Outputs obrigatório (11.4.2). Checklist de novo projeto do zero ao primeiro deploy (12.4). Backup e disaster recovery (12.5). Biome como avaliação futura. Versões mínimas atualizadas.

---