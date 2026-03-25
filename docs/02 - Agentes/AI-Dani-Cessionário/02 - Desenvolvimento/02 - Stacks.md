# 02 - Stacks

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Tech Lead / CTO | Stack normativa do agente AI-Dani-Cessionário, derivada das Regras de Negócio (D01) e do padrão ShiftLabs Stacks v7.1 | v1.0 | Claude Code Desktop | 23/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Backend:** Node.js ≥ 20 LTS + NestJS + TypeScript strict + Prisma + Supabase (PostgreSQL). Redis para cache de sessão e rate limiting. RabbitMQ para fila de notificações.
> - **Frontend Web:** React 19 + Vite (módulo interno logado) + TypeScript strict + Tailwind CSS v4 + shadcn/ui + Framer Motion.
> - **Mobile (Fase 2 — WhatsApp):** EvolutionAPI como gateway; sem app mobile próprio no MVP.
> - **IA no produto:** GPT-4 via OpenAI SDK + Vercel AI SDK. SSE para streaming. Langfuse para observabilidade de LLM. Supabase pgvector para RAG.
> - **Autenticação:** JWT (access token curto + refresh token longo) herdado da sessão da plataforma Repasse Seguro.
> - **Tecnologias proibidas:** jQuery, Moment.js, MySQL, TypeORM, GraphQL (sem ADR), WebSocket sem ADR.
> - **Desvios do baseline:** ADR-001 (EvolutionAPI para WhatsApp), ADR-002 (SSE obrigatório para streaming de LLM).

---

## 1. Princípios de Governança Tecnológica

1. **Baseline primeiro.** O padrão ShiftLabs Stacks v7.1 é a fonte primária de verdade. Desvios exigem ADR aprovado antes da implementação.
2. **Decisão rastreável.** Toda escolha de tecnologia rastreia para uma regra de negócio ou requisito técnico do D01. Preferência pessoal não é critério válido.
3. **Isolamento de dados como restrição de arquitetura.** A Dani opera exclusivamente com dados do Cessionário autenticado (RN-DC-001 a RN-DC-003). Toda camada de stack que acessa dados deve implementar filtro por `cessionario_id` antes de qualquer operação.
4. **Calculadora de Comissão como módulo determinístico independente.** A Calculadora deve funcionar sem depender do modelo de IA (RN-DC-023). A stack da Calculadora é separada logicamente da stack do agente.
5. **Sem MVP.** Todo produto ShiftLabs é lançado completo com 100% de cobertura. Nenhuma feature é entregue pela metade.

---

## 2. Backend

### 2.1 Runtime e Linguagem

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| Node.js | ≥ 20 LTS | ✅ Aprovado | Padrão ShiftLabs, suporte LTS ativo, compatível com NestJS e SDKs da OpenAI | Usar apenas versão LTS em produção. Nunca usar versões odd (19, 21) em produção |
| TypeScript | ≥ 5.3 | ✅ Aprovado | Tipagem obrigatória; `strict: true` em todo código backend | `strict: true` obrigatório no `tsconfig.json`. Proibido `any` sem comentário justificado |

### 2.2 Framework HTTP

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| NestJS | ≥ 10 | ✅ Aprovado | Arquitetura modular, decorators TypeScript, Swagger embutido, guards/pipes nativos. Output de IA mais consistente | Usar módulos separados por domínio: `agente`, `calculadora`, `oportunidade`, `cessionario`, `escrow`, `notificacao` |
| Express | — | ❌ Proibido | NestJS já abstrai Express; uso direto aumenta entropia arquitetural | Usar NestJS. Nunca acessar instância Express diretamente |
| Fastify | — | ❌ Proibido | Fora do padrão ShiftLabs; requer ADR aprovado | ADR obrigatório se necessário |

### 2.3 ORM e Banco de Dados

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| Prisma | ≥ 5.x | ✅ Aprovado | Único ORM aprovado ShiftLabs. Tipagem auto-gerada, migrations declarativas, integração nativa com Supabase | `schema.prisma` é a fonte única de verdade do modelo de dados. Migrations via `prisma migrate dev` |
| Supabase (PostgreSQL) | — | ✅ Aprovado | PostgreSQL gerenciado + Auth + Storage + Realtime + pgvector em uma plataforma. Padrão ShiftLabs | Todas as tabelas com UUID v4 como PK. Colunas `created_at` e `updated_at` obrigatórias |
| TypeORM | — | ❌ Proibido | Fora do padrão ShiftLabs; tipagem inferior ao Prisma | Usar Prisma exclusivamente |
| Drizzle | — | ❌ Proibido | Ecossistema menos maduro para o tamanho do projeto | Usar Prisma exclusivamente |
| MySQL | — | ❌ Proibido | Fora do padrão ShiftLabs; padrão é PostgreSQL via Supabase | Usar Supabase/PostgreSQL |

### 2.4 Cache

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| Redis | ≥ 7.x | ✅ Aprovado | Cache de rate limiting (RN-DC-025, RN-DC-041), cache de sessão da Dani, cache de resultados de cálculo determinístico | TTL obrigatório em toda chave. Estratégia de invalidação documentada por chave. Docker em local, serviço gerenciado (Upstash/Railway) em produção |

**Chaves Redis obrigatórias para a Dani:**

| Chave | TTL | Uso |
|---|---|---|
| `dani:rate:webchat:{cessionario_id}` | 3.600s (1h, janela deslizante) | Rate limit de 30 msgs/hora (RN-DC-025) | [CORRIGIDO: PROBLEMA-007]
| `dani:rate:otp:{phone_hash}` | 3.600s | Rate limit de 3 tentativas OTP/hora (RN-DC-041) | [CORRIGIDO: PROBLEMA-007]
| `dani:block:otp:{phone_hash}` | 1.800s (30min) | Hard block após 5 falhas consecutivas (RN-DC-041) | [CORRIGIDO: PROBLEMA-007]
| `dani:cache:calc:{opr_id}:{val_hash}` | 300s (5min) | Cache de resultado determinístico da Calculadora | [CORRIGIDO: PROBLEMA-007]
| `dani:status:agent` | 60s | Status operacional da Dani (Operacional/Fallback/Desligado) | [CORRIGIDO: PROBLEMA-007]

### 2.5 Filas

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| RabbitMQ | ≥ 3.12 | ✅ Aprovado | Fila de notificações (alertas proativos, lembretes ZapSign, status Escrow) | Toda fila com política de retry (3x com backoff exponencial) e dead-letter queue. Docker em local, CloudAMQP em produção |

**Filas obrigatórias:**

| Fila | DLQ | Uso |
|---|---|---|
| `dani.notificacoes` | `dani.notificacoes.dlq` | Envio de alertas proativos de oportunidades |
| `dani.whatsapp` | `dani.whatsapp.dlq` | Mensagens de saída para EvolutionAPI (Fase 2) |
| `dani.agent_monitor` | `dani.agent_monitor.dlq` | Monitoramento de taxa de erro e SLA (RN-DC-024, RN-DC-029) |

### 2.6 Validação e Serialização

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| class-validator | ≥ 0.14 | ✅ Aprovado | Padrão ShiftLabs. Integrado aos DTOs do NestJS | Toda entrada de dados validada via pipes. Nenhum dado bruto sem DTO validado |
| class-transformer | ≥ 0.5 | ✅ Aprovado | Transformação de DTOs, serialização de responses | Usar junto com `class-validator` em todo DTO |
| Zod | — | 🔶 Condicional | Aprovado apenas para validação em Edge Functions ou scripts fora do NestJS | Não usar em controllers ou services NestJS — usar `class-validator` |

### 2.7 Logging e Observabilidade

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| Pino | ≥ 8.x | ✅ Aprovado | Logs estruturados (JSON), alto desempenho, padrão ShiftLabs | Todo log deve incluir: `request_id`, `timestamp`, `level`, `cessionario_id` (quando disponível), `modulo` |
| Sentry | ≥ 7.x | ✅ Aprovado | Error tracking obrigatório. Stack traces, breadcrumbs e alertas | Obrigatório em toda camada (backend, frontend). DSN em variável de ambiente |
| Langfuse | ≥ 2.x | ✅ Aprovado | Observabilidade de LLM: tracing, custo, latência, qualidade da Dani | Todo trace da Dani deve ter: `session_id`, `cessionario_id`, `input_tokens`, `output_tokens`, `latency_ms` |

### 2.8 Tabela Consolidada — Backend

| Tecnologia | Versão mínima | Status |
|---|---|---|
| Node.js | ≥ 20 LTS | ✅ |
| TypeScript | ≥ 5.3 | ✅ |
| NestJS | ≥ 10 | ✅ |
| Prisma | ≥ 5.x | ✅ |
| Supabase / PostgreSQL | — | ✅ |
| Redis | ≥ 7.x | ✅ |
| RabbitMQ | ≥ 3.12 | ✅ |
| class-validator | ≥ 0.14 | ✅ |
| class-transformer | ≥ 0.5 | ✅ |
| Pino | ≥ 8.x | ✅ |
| Sentry (backend) | ≥ 7.x | ✅ |
| Langfuse | ≥ 2.x | ✅ |
| Express | — | ❌ |
| TypeORM | — | ❌ |
| Drizzle | — | ❌ |
| MySQL | — | ❌ |

---

## 3. Frontend Web

> ⚙️ **Contexto:** O módulo do Cessionário é uma área 100% logada (dashboard, oportunidades, negociações, perfil). Não há face pública/SEO. Portanto, o padrão é **React 19 + Vite** (não Next.js).

### 3.1 Framework e Build

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| React | ≥ 19 | ✅ Aprovado | Padrão ShiftLabs para apps logados. Server Components não são necessários para dashboard | Usar React 19 com concurrent features habilitadas |
| Vite | ≥ 5.x | ✅ Aprovado | Bundler e dev server padrão para apps React internos ShiftLabs | Configuração base no `vite.config.ts`. HMR habilitado em dev |
| Next.js | — | ❌ Proibido (neste módulo) | Módulo 100% logado não precisa de SSR/SSG. Next.js reservado para faces públicas/SEO | Usar React + Vite para o módulo Cessionário |

### 3.2 Roteamento

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| TanStack Router | ≥ 1.x | ✅ Aprovado | Type-safe, integração nativa com TanStack Query. Padrão ShiftLabs para apps React+Vite | Todas as rotas definidas como type-safe. Rota protegida por guard de autenticação |

### 3.3 State Management

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| TanStack Query | ≥ 5.x | ✅ Aprovado | Cache e sincronização de dados do servidor. Padrão ShiftLabs | Toda requisição ao backend via TanStack Query. Invalidação de cache explícita após mutações |
| Zustand | ≥ 4.x | ✅ Aprovado | Estado global de UI (status da Dani, contexto do chat aberto, oportunidade carregada) | Apenas estado de UI. Dados do servidor via TanStack Query |
| Redux | — | ❌ Proibido | Boilerplate excessivo; substituído por Zustand + TanStack Query | Usar Zustand para estado global |

### 3.4 UI e Estilo

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| Tailwind CSS | ≥ 4.x | ✅ Aprovado | CSS utility-first. Padrão ShiftLabs. Design tokens via variáveis CSS | Tokens de cor e espaçamento definidos no Brand Guide (D03). Sem valores mágicos inline |
| shadcn/ui | latest | ✅ Aprovado | Componentes com ownership total do código. Excelente representação nos LLMs | Instalar apenas componentes utilizados. Customizar via Tailwind, nunca sobrescrever classes shadcn diretamente |
| CSS-in-JS (styled-components, emotion) | — | ❌ Proibido | Conflita com Tailwind v4; performance inferior | Usar Tailwind exclusivamente |

### 3.5 Animações e Motion

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| Framer Motion | ≥ 11.x | ✅ Aprovado | Padrão ShiftLabs para animações web. AnimatePresence para transições de entrada/saída do chat | Tokens de motion definidos no Motion Spec (D04). Usar `AnimatePresence` para montagem/desmontagem do FAB e chat window |

**Animações obrigatórias (RN-DC-005, RN-DC-025):**

| Componente | Animação | Duração |
|---|---|---|
| Mensagem de boas-vindas da Dani | fade-in | 300ms |
| Campo de entrada após rate limit desbloqueado | pulse sutil | 500ms |
| FAB com badge | scale + bounce | 200ms |
| Chat window (open/close) | slide-up + fade | 250ms |

### 3.6 Acessibilidade

- WCAG 2.1 nível AA obrigatório em toda interface do módulo Cessionário.
- Score de risco (RN-DC-011): indicadores visuais (verde/amarelo/vermelho) devem ter contraste mínimo 4.5:1 e rótulo textual acessível para screen readers (`aria-label`).
- Todo elemento interativo com `aria-label` ou `aria-labelledby`.
- Tabelas comparativas (RN-DC-015) com `role="table"` e cabeçalhos de coluna `scope="col"`.

### 3.7 Tabela Consolidada — Frontend Web

| Tecnologia | Versão mínima | Status |
|---|---|---|
| React | ≥ 19 | ✅ |
| Vite | ≥ 5.x | ✅ |
| TanStack Router | ≥ 1.x | ✅ |
| TanStack Query | ≥ 5.x | ✅ |
| Zustand | ≥ 4.x | ✅ |
| Tailwind CSS | ≥ 4.x | ✅ |
| shadcn/ui | latest | ✅ |
| Framer Motion | ≥ 11.x | ✅ |
| Next.js | — | ❌ (este módulo) |
| Redux | — | ❌ |
| CSS-in-JS | — | ❌ |

---

## 4. Mobile

> ⚙️ **Escopo mobile:** No MVP (Fase 1), o canal é exclusivamente webchat. Mobile refere-se à responsividade da interface web no Fase 1. Na Fase 2, o canal WhatsApp é via EvolutionAPI — não há app nativo. Portanto, não há React Native/Expo no escopo atual.

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| Responsive Web (Tailwind breakpoints) | — | ✅ Aprovado | A interface do Cessionário deve ser totalmente funcional em mobile via browser | Breakpoints: `sm` (640px), `md` (768px), `lg` (1024px). Chat FAB e janela de chat 100% responsivos |
| React Native + Expo | — | 🔶 Condicional | Reservado para app nativo futuro (pós-Fase 2). ADR obrigatório antes de iniciar | Não iniciar desenvolvimento sem ADR e aprovação do Tech Lead |
| EvolutionAPI | ≥ 2.x | ✅ Aprovado (Fase 2) | Gateway WhatsApp Business para o canal Fase 2 (RN-DC-040 a RN-DC-044). Ver ADR-001 | Gerenciada como serviço externo. Integração via HTTP. Webhook para mensagens recebidas |

---

## 5. IA no Produto (Agente Dani)

### 5.1 Modelo e SDK

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| GPT-4 (OpenAI) | `gpt-4o` | ✅ Aprovado | Melhor equilíbrio qualidade/function calling/ecossistema. Padrão ShiftLabs para features de IA | Usar `gpt-4o` como padrão. Downgrade para `gpt-4o-mini` apenas com ADR documentando impacto de qualidade |
| OpenAI SDK | ≥ 4.x (Node.js) | ✅ Aprovado | SDK oficial OpenAI para Node.js. Integração com NestJS via módulo injetável | Chave de API em variável de ambiente `OPENAI_API_KEY`. Nunca hardcoded |
| Vercel AI SDK | ≥ 3.x | ✅ Aprovado | Gerenciamento de streaming SSE de respostas da Dani. Padrão ShiftLabs | Usar para streaming de respostas longas (análise de oportunidade, comparação) |
| LangChain.js | ≥ 0.2.x | ✅ Aprovado | Orquestração de chains multi-step (análise + cálculo + comparação) | Usar apenas onde a orquestração multi-step justificar. Não usar como wrapper simples do OpenAI SDK |

### 5.2 Structured Outputs e Function Calling

- Structured Outputs obrigatório para respostas determinísticas: cálculo de comissão, ROI, score de risco.
- Function calling obrigatório para: buscar oportunidades no marketplace, calcular comissão via Calculadora, verificar status de Escrow.
- Toda resposta estruturada deve ter schema Zod validando o output antes de exibir ao Cessionário.

### 5.3 Memória e RAG

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| Supabase pgvector | — | ✅ Aprovado | Embeddings de histórico de conversas e dados de empreendimentos. Padrão ShiftLabs | Embeddings gerados via `text-embedding-3-small`. Namespace por `cessionario_id` — nunca misturar dados de Cessionários diferentes |
| Pinecone | — | ❌ Proibido | Infra adicional desnecessária. Supabase pgvector cobre o caso de uso | Usar Supabase pgvector |

**Restrição de isolamento em RAG (RN-DC-001 a RN-DC-003):**

```
WHERE namespace = cessionario_id
```

Toda query vetorial deve incluir filtro de namespace. Consulta sem filtro é tratada como falha de segurança.

### 5.4 Streaming

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| SSE (Server-Sent Events) | — | ✅ Aprovado | Streaming de respostas da Dani para o frontend. Ver ADR-002 | Endpoint SSE: `GET /ai-dani/stream`. Content-Type: `text/event-stream`. Timeout: 30s |
| WebSocket | — | 🔶 Condicional | Aprovado apenas para features bidirecionais não cobertas por SSE. Requer ADR | ADR obrigatório. Não usar para streaming de LLM |

### 5.5 Prompts

- Prompts como código: armazenados em `src/agente/prompts/` como arquivos TypeScript tipados.
- Versão de prompt no nome do arquivo: `dani-system-prompt.v2.ts`.
- Nenhum prompt hardcoded em controller ou service.
- System prompt inclui obrigatoriamente: lista de dados bloqueados (RN-DC-002), mensagens padrão de recusa (RN-DC-004), instrução de não submeter propostas (RN-DC-028).

### 5.6 Tabela Consolidada — IA

| Tecnologia | Versão mínima | Status |
|---|---|---|
| GPT-4 (`gpt-4o`) | — | ✅ |
| OpenAI SDK | ≥ 4.x | ✅ |
| Vercel AI SDK | ≥ 3.x | ✅ |
| LangChain.js | ≥ 0.2.x | ✅ |
| Supabase pgvector | — | ✅ |
| Langfuse | ≥ 2.x | ✅ |
| Pinecone | — | ❌ |

---

## 6. Comunicação e API

### 6.1 Padrão de API

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| REST (JSON) | — | ✅ Aprovado | Padrão ShiftLabs. `fetch` nativo no frontend | Content-Type: `application/json`. Charset: UTF-8 |
| GraphQL | — | ❌ Proibido (sem ADR) | Complexidade desnecessária para este escopo | Usar REST. ADR obrigatório para justificar GraphQL |
| gRPC | — | ❌ Proibido (sem ADR) | Fora do padrão ShiftLabs para este produto | Usar REST |

### 6.2 Autenticação e Autorização

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| JWT | — | ✅ Aprovado | Stateless, funciona igual web e mobile. Padrão ShiftLabs | Access token: 15 min. Refresh token: 7 dias. Payload deve incluir `cessionario_id` e `role`. Nunca armazenar dados sensíveis no payload |
| Herança de sessão da plataforma | — | ✅ Aprovado | A Dani herda a sessão da plataforma Repasse Seguro (RN-DC-007). Sem novo login | Guard NestJS valida JWT da plataforma antes de qualquer endpoint da Dani |
| RBAC | — | ✅ Aprovado | Cessionário acessa apenas os próprios dados (RN-DC-001). Admin tem acesso amplo | Guard `CessionarioOwnerGuard` obrigatório em todos os endpoints que retornam dados do Cessionário |

### 6.3 Versionamento de API

- Prefixo de versão: `/api/v1/`. Nunca expor endpoints sem versionamento.
- Breaking changes exigem nova versão (`/api/v2/`) e período de deprecação de 30 dias.
- Swagger/OpenAPI auto-gerado via `@nestjs/swagger` obrigatório em todo endpoint.

### 6.4 Tabela Consolidada — Comunicação

| Protocolo/Tecnologia | Status |
|---|---|
| REST JSON | ✅ |
| SSE (streaming LLM) | ✅ |
| JWT | ✅ |
| GraphQL | ❌ (sem ADR) |
| gRPC | ❌ (sem ADR) |

---

## 7. Repositório e Código

| Aspecto | Definição |
|---|---|
| Estrutura | Monorepo com Turborepo + pnpm. Pacote `apps/cessionario` para o frontend. Pacote `apps/api` para o backend. Pacotes compartilhados em `packages/` |
| Nomenclatura de arquivos | `kebab-case` para arquivos. `PascalCase` para componentes React. `camelCase` para funções e variáveis. `SCREAMING_SNAKE_CASE` para constantes |
| Nomenclatura de banco | `snake_case` para tabelas e colunas. Plural para tabelas (`oportunidades`, `negociacoes`). Prefixo de FK: `{tabela}_id` |
| Linting | ESLint com config ShiftLabs. Prettier para formatação. Ambos obrigatórios no CI |
| Code review | Pull Request obrigatório para toda mudança em `main`. Mínimo 1 aprovação. CI verde antes do merge |

---

## 8. Testes

| Camada | Ferramenta | Cobertura mínima |
|---|---|---|
| Unit (backend) | Jest + ts-jest | ≥ 80% nas regras de cálculo (Calculadora de Comissão, ROI, Δ) |
| Integration (backend) | Jest + Supertest | Todos os endpoints da API da Dani |
| E2E (frontend) | Playwright | Fluxos críticos: análise de oportunidade, simulação, comparação, fallback da Calculadora |
| LLM (agente) | Langfuse Evals | Score de qualidade ≥ 0.8 em respostas de análise. Cobertura de mensagens de recusa (RN-DC-004) |

**Fixtures obrigatórias:**

- Oportunidade com Δ > 0 (caso padrão)
- Oportunidade com Δ ≤ 0 (caso fallback de comissão)
- Oportunidade em negociação (badge e notificação)
- Oportunidade encerrada (sugestões alternativas)
- Cessionário com KYC pendente
- Simulação com valor inválido (zero, negativo, não numérico)

---

## 9. Deploy e CI/CD

| Aspecto | Definição |
|---|---|
| CI/CD | GitHub Actions. Pipeline obrigatório: lint → test → build → deploy |
| Ambientes | `development` (local), `staging` (Vercel preview + Supabase branch), `production` (Vercel + Supabase main) |
| Frontend deploy | Vercel. Preview automático por PR. Produção via merge em `main` |
| Backend deploy | [DECISÃO AUTÔNOMA — Railway ou Render para NestJS containerizado. Railway preferido por integração com PostgreSQL e Redis gerenciados. Alternativa descartada: AWS ECS (complexidade operacional excessiva para o tamanho do projeto)] |
| Promoção para produção | CI verde + aprovação de PR + testes E2E passando em staging |
| Rollback | Revert de commit + redeploy via CI. Rollback de banco via `prisma migrate resolve` |

---

## 10. Segurança

### 10.1 Regras Críticas

| Regra | Implementação |
|---|---|
| HTTPS obrigatório | TLS em todos os ambientes (produção e staging). Vercel e Railway fornecem TLS automático |
| CORS configurado | Permitir apenas origens da plataforma Repasse Seguro. Nenhum wildcard em produção |
| Rate limiting | Implementado via Redis (seção 2.4). Aplicado no NestJS com guard antes dos controllers |
| Sanitização de inputs | `class-validator` em todos os DTOs. Nenhum dado bruto processado pelo agente sem validação |
| Secrets em variáveis de ambiente | `OPENAI_API_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `JWT_SECRET`, `EVOLUTION_API_KEY` em `.env`. Nunca no código-fonte |
| Logs sem PII | Proibido logar CPF, nome completo, dados pessoais do Cedente ou do Cessionário em qualquer nível de log |

### 10.2 Isolamento de Dados (LGPD)

- Todo endpoint que retorna dados do Cessionário deve filtrar por `cessionario_id` extraído do JWT.
- Nenhum dado de Cedente é retornado ao Cessionário (RN-DC-002). Guard `IsolamentoGuard` aplicado nos serviços de oportunidade.
- Opt-out WhatsApp via comando PARAR processa imediatamente sem retenção (RN-DC-044).
- Histórico de conversas retido por 90 dias. Deleção automática via job agendado.

### 10.3 Tecnologias de Segurança

| Tecnologia | Status | Uso |
|---|---|---|
| Helmet (NestJS) | ✅ Obrigatório | Headers HTTP de segurança em todos os endpoints |
| bcrypt | ✅ Obrigatório | Hash de tokens de vinculação WhatsApp (RN-DC-040) |
| `@nestjs/throttler` | ✅ Obrigatório | Rate limiting global. Complementa Redis para rate limit granular |

---

## 11. Analytics e Tracking

| Tecnologia | Versão mínima | Status | Justificativa |
|---|---|---|---|
| PostHog | — | ✅ Aprovado | Analytics + session replay + feature flags. Padrão ShiftLabs |

**Eventos obrigatórios (snake_case):**

| Evento | Propriedades |
|---|---|
| `dani_chat_opened` | `entry_point` (oportunidade/dashboard/fab), `cessionario_id_hash`, `kyc_status` |
| `dani_analise_requested` | `opr_id`, `response_time_ms`, `has_score` |
| `dani_simulacao_completed` | `opr_id`, `valor_proposto`, `roi_projetado` |
| `dani_comparacao_requested` | `count_oprs`, `criterio_ranqueamento` |
| `dani_fallback_activated` | `motivo` (api_error/latency/error_rate), `duracao_segundos` |
| `dani_rate_limit_hit` | `canal` (webchat/whatsapp), `mensagens_na_janela` |
| `dani_takeover_triggered` | `confidence_score`, `modulo` |

---

## 12. Tecnologias Proibidas

| Tecnologia | Motivo | Alternativa aprovada |
|---|---|---|
| jQuery | Redundante com React 19; aumenta bundle sem benefício | Hooks nativos do React |
| Moment.js | Deprecated; bundle pesado (330kB) | `date-fns` ou `Luxon` |
| MySQL / MariaDB | Fora do padrão ShiftLabs; padrão é PostgreSQL via Supabase | Supabase / PostgreSQL |
| TypeORM | Tipagem inferior ao Prisma; fora do padrão ShiftLabs | Prisma |
| Drizzle | Ecossistema menos maduro | Prisma |
| Pinecone | Infra adicional desnecessária; Supabase pgvector cobre o caso | Supabase pgvector |
| Redux | Boilerplate excessivo | Zustand + TanStack Query |
| CSS-in-JS (styled-components, emotion) | Conflita com Tailwind v4; performance inferior | Tailwind CSS v4 |
| GraphQL (sem ADR) | Complexidade desnecessária para este escopo | REST JSON |
| gRPC (sem ADR) | Fora do padrão ShiftLabs para este produto | REST JSON |
| WebSocket (sem ADR) | SSE cobre o caso de streaming de LLM | SSE via Vercel AI SDK |
| `any` TypeScript (sem comentário) | Elimina benefícios do TypeScript strict | Tipagem explícita |
| Secrets hardcoded | Risco de segurança crítico | Variáveis de ambiente |

---

## 13. ADRs — Desvios do Baseline

**ADR-001: EvolutionAPI como gateway WhatsApp Business (Fase 2)**

1. **Contexto:** RN-DC-040 a RN-DC-044 definem canal WhatsApp para a Dani na Fase 2. O padrão ShiftLabs não especifica gateway WhatsApp.
2. **Decisão:** EvolutionAPI v2.x como gateway WhatsApp Business via webhook + HTTP API.
3. **Alternativas consideradas:** Twilio (custo por mensagem mais alto para volume estimado; lock-in de vendor); Meta Cloud API direta (maior complexidade de manutenção de sessão). EvolutionAPI é open-source, auto-hospedado, custo fixo.
4. **Consequências:** Responsabilidade de manutenção da instância EvolutionAPI (Docker ou Railway). Monitoramento obrigatório via health check a cada 60s. Fila `dani.whatsapp` com DLQ para mensagens não entregues.

**ADR-002: SSE obrigatório para streaming de respostas da Dani**

1. **Contexto:** SLA de resposta ≤ 5s para análise de oportunidade (D01 seção 15). Respostas de análise completa podem exceder 5s de processamento. Streaming melhora percepção de latência.
2. **Decisão:** SSE (Server-Sent Events) via Vercel AI SDK para streaming de respostas da Dani. Endpoint `GET /api/v1/ai-dani/stream`.
3. **Alternativas consideradas:** WebSocket (bidirecional desnecessário para este caso; maior complexidade de infraestrutura); polling (péssima experiência de usuário; maior carga no backend).
4. **Consequências:** Frontend deve implementar `EventSource` ou usar Vercel AI SDK `useChat`. Timeout de conexão SSE: 30s. Reconexão automática em caso de queda.

---

## 14. Backlog de Pendências

| Item pendente | Seção | Impacto | Pergunta objetiva | Dono | Status |
|---|---|---|---|---|---|
| Plataforma de hospedagem do backend NestJS | 9. Deploy e CI/CD | P1 | Railway ou Render? Confirmar plano e limites de instância para produção | Tech Lead | Aberto |

---

## Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial do documento. Stack derivada de D01 (Regras de Negócio AI-Dani-Cessionário) + ShiftLabs Stacks v7.1. ADR-001 (EvolutionAPI) e ADR-002 (SSE streaming). |
