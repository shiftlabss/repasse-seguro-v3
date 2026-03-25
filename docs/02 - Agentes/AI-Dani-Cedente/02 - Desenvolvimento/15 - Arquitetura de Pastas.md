# 15 - Arquitetura de Pastas — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Tech Lead, Arquiteto, DevOps |
| Escopo | Estrutura do monorepo, convenções de nomenclatura, padrões de módulo e mapeamento de erros/cache |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D01 · D02 · D09 · D13 · D14 |

---

> **📌 TL;DR**
>
> - **Monorepo:** single repo `ai-dani-cedente/` — backend-only NestJS service (sem apps/web próprio; o widget é servido pelo frontend da plataforma Repasse Seguro).
> - **Backend feature-first:** `src/modules/<domain>/` com padrão rígido Controller → Service → Repository → DTO → Entity.
> - **12 módulos:** auth, chat, agent, rag, opportunity, proposal, dossier, escrow, notification, simulation, fallback, admin.
> - **Cache Redis:** padrão `dani:{cedente_id}:{dominio}:{id}` — todas as chaves com TTL explícito.
> - **Error Prefixes:** DCE- (Dani-Cedente) por módulo com código numérico único.
> - **Filas RabbitMQ:** 3 queues mapeadas (rag.ingest, notification.send, escrow.events).
> - **Zero seções pendentes** — cobertura 100% dos 12 módulos e todos os padrões de nomenclatura.

---

## 1. Persona

Arquiteto de Software responsável pela estrutura e governança do repositório do AI-Dani-Cedente. Foco em previsibilidade: todo dev sabe onde criar cada arquivo sem perguntar, e todo agente de IA gera código respeitando a árvore de pastas.

---

## 2. Visão Geral do Monorepo

```
ai-dani-cedente/                  # Repositório raiz do serviço
├── src/                          # Código fonte NestJS
│   ├── app.module.ts             # Módulo raiz
│   ├── main.ts                   # Bootstrap do servidor
│   ├── modules/                  # Módulos de domínio (ver Seção 4)
│   ├── common/                   # Código cross-cutting
│   │   ├── middleware/           # PII masking, rate limit, cedente isolation
│   │   ├── guards/               # JwtAuthGuard, RbacGuard, RateLimitGuard
│   │   ├── interceptors/         # LoggingInterceptor, LangfuseInterceptor
│   │   ├── filters/              # GlobalExceptionFilter
│   │   ├── decorators/           # @CurrentCedente(), @RequireRole()
│   │   └── pipes/                # ValidationPipe, ParseUuidPipe
│   ├── infrastructure/           # Adaptadores de infraestrutura
│   │   ├── prisma/               # PrismaService, prisma.module.ts
│   │   ├── redis/                # RedisService, redis.module.ts
│   │   ├── rabbitmq/             # RabbitMQService, publishers, consumers
│   │   ├── openai/               # OpenAIService, streaming adapter
│   │   └── langfuse/             # LangfuseService, trace wrappers
│   └── config/
│       ├── configuration.ts      # Env vars validadas via Zod
│       └── configuration.schema.ts
├── prisma/                       # Schema, migrations, seed, RLS
│   ├── schema.prisma
│   ├── seed.ts
│   ├── rls/
│   │   ├── policies.sql
│   │   └── indexes.sql
│   └── middleware/
│       ├── cedente-isolation.middleware.ts
│       └── soft-delete.middleware.ts
├── test/                         # Testes e2e
│   ├── app.e2e-spec.ts
│   └── jest-e2e.json
├── docs/                         # Documentação técnica (D01-D29)
├── .github/
│   └── workflows/
│       ├── ci.yml                # CI: lint, test, build
│       └── deploy.yml            # CD: deploy em Railway/VPS
├── docker-compose.yml            # Dev: PostgreSQL, Redis, RabbitMQ
├── Dockerfile                    # Produção
├── .env.example                  # Template de env vars
├── tsconfig.json
├── tsconfig.build.json
├── .eslintrc.js
├── .prettierrc
└── package.json
```

---

## 3. Estrutura Interna de Módulos (Backend)

### 3.1 Padrão obrigatório por módulo

```
src/modules/<domain>/
├── <domain>.module.ts            # NestJS @Module() — imports, providers, controllers
├── <domain>.controller.ts        # HTTP endpoints (@Controller, @Get, @Post, etc.)
├── <domain>.service.ts           # Lógica de negócio (@Injectable)
├── <domain>.repository.ts        # Acesso ao Prisma (@Injectable — padrão Repository)
├── dto/
│   ├── create-<domain>.dto.ts    # DTO de criação (@IsString, @IsUUID, etc.)
│   ├── update-<domain>.dto.ts    # DTO de atualização (extends PartialType)
│   └── filter-<domain>.dto.ts    # DTO de filtros para listagem (opcional)
├── entities/
│   └── <domain>.entity.ts        # Classe de entidade (mapeamento Prisma → resposta API)
└── tests/
    ├── <domain>.service.spec.ts  # Testes unitários do Service
    └── <domain>.controller.spec.ts # Testes unitários do Controller
```

### 3.2 Módulos e estrutura detalhada

```
src/modules/
├── auth/                         # Validação JWT, guards, decorators
│   ├── auth.module.ts
│   ├── auth.guard.ts             # JwtAuthGuard — usa Supabase Auth
│   ├── auth.decorator.ts         # @CurrentCedente()
│   └── auth.service.ts           # Verifica JWT, resolve cedente_id
│
├── chat/                         # Sessões de chat, mensagens, SSE streaming
│   ├── chat.module.ts
│   ├── chat.controller.ts        # POST /chat/sessions, GET /chat/sessions/:id/messages, SSE
│   ├── chat.service.ts
│   ├── chat.repository.ts
│   ├── dto/
│   │   ├── create-session.dto.ts
│   │   ├── create-message.dto.ts
│   │   └── csat.dto.ts
│   ├── entities/
│   │   ├── chat-session.entity.ts
│   │   └── chat-message.entity.ts
│   └── tests/
│
├── agent/                        # LangGraph state machine + LangChain tools
│   ├── agent.module.ts
│   ├── agent.service.ts          # Orquestra LangGraph, invoca tools
│   ├── agent.graph.ts            # DaniCedenteGraph — definição do grafo LangGraph
│   ├── tools/
│   │   ├── get-opportunity.tool.ts
│   │   ├── get-proposal.tool.ts
│   │   ├── get-dossier.tool.ts
│   │   ├── get-escrow.tool.ts
│   │   └── simulate-return.tool.ts
│   └── tests/
│
├── rag/                          # RAG pipeline, embeddings, busca semântica
│   ├── rag.module.ts
│   ├── rag.service.ts            # query(text) → chunks relevantes
│   ├── rag.repository.ts         # Acesso a knowledge_embeddings + pgvector
│   ├── rag.ingest.service.ts     # Chunking + embedding de novos conteúdos
│   ├── dto/
│   │   └── ingest-knowledge.dto.ts
│   └── tests/
│
├── opportunity/                  # Oportunidades e cenários (leitura da API RS)
│   ├── opportunity.module.ts
│   ├── opportunity.controller.ts # GET /opportunities/:id
│   ├── opportunity.service.ts
│   ├── opportunity.repository.ts
│   ├── dto/
│   │   └── opportunity-context.dto.ts
│   ├── entities/
│   │   └── opportunity.entity.ts
│   └── tests/
│
├── proposal/                     # Propostas — aceite, recusa, contraproposta
│   ├── proposal.module.ts
│   ├── proposal.controller.ts    # GET /proposals, PATCH /proposals/:id/accept, etc.
│   ├── proposal.service.ts
│   ├── proposal.repository.ts
│   ├── dto/
│   │   ├── accept-proposal.dto.ts
│   │   ├── reject-proposal.dto.ts
│   │   └── create-counter-proposal.dto.ts
│   ├── entities/
│   │   └── proposal.entity.ts
│   └── tests/
│
├── dossier/                      # Documentos do dossiê, upload
│   ├── dossier.module.ts
│   ├── dossier.controller.ts     # GET /dossier, POST /dossier/upload
│   ├── dossier.service.ts
│   ├── dossier.repository.ts
│   ├── dto/
│   │   └── upload-document.dto.ts
│   ├── entities/
│   │   └── dossier-document.entity.ts
│   └── tests/
│
├── escrow/                       # Transações Escrow, extensão, alertas
│   ├── escrow.module.ts
│   ├── escrow.controller.ts      # GET /escrow/:id, POST /escrow/:id/extend
│   ├── escrow.service.ts
│   ├── escrow.repository.ts
│   ├── dto/
│   │   └── extend-escrow.dto.ts
│   ├── entities/
│   │   └── escrow-transaction.entity.ts
│   └── tests/
│
├── notification/                 # Notificações proativas (webchat + email)
│   ├── notification.module.ts
│   ├── notification.service.ts
│   ├── notification.publisher.ts # Publica em RabbitMQ notification.send
│   ├── notification.consumer.ts  # Consome de RabbitMQ notification.send
│   ├── templates/
│   │   ├── nova-proposta.template.ts
│   │   ├── escrow-alerta.template.ts
│   │   └── documento-aprovado.template.ts # + demais 7 templates
│   └── tests/
│
├── simulation/                   # Cálculo de retorno líquido
│   ├── simulation.module.ts
│   ├── simulation.controller.ts  # POST /simulation
│   ├── simulation.service.ts     # Cálculo síncrono: offered_value - outstanding_balance
│   ├── dto/
│   │   └── simulate-return.dto.ts
│   └── tests/
│
├── fallback/                     # Circuit breaker, kill switch, thresholds
│   ├── fallback.module.ts
│   ├── fallback.service.ts       # Monitora thresholds 10%/30% via Redis sliding window
│   ├── fallback.guard.ts         # Bloqueia requests quando agente está DISABLED
│   └── tests/
│
└── admin/                        # Painel admin — takeover, métricas, configurações
    ├── admin.module.ts
    ├── admin.controller.ts       # Endpoints protegidos por role 'admin'
    ├── admin.service.ts
    ├── dto/
    │   └── takeover.dto.ts
    └── tests/
```

---

## 4. Regras de Import

### 4.1 Imports entre módulos

```typescript
// ✅ CORRETO — importar pelo módulo NestJS (injeção de dependência)
@Module({
  imports: [ChatModule, ProposalModule],
  providers: [AgentService],
})
export class AgentModule {}

// ✅ CORRETO — importar tipo/DTO de outro módulo (via exports do módulo)
import { ProposalEntity } from '../proposal/entities/proposal.entity';

// 🔴 PROIBIDO — importar serviço diretamente sem injeção de dependência
import { ProposalService } from '../proposal/proposal.service'; // ❌

// 🔴 PROIBIDO — importar de infrastructure diretamente em controller
import { RedisService } from '../../infrastructure/redis/redis.service'; // ❌ no controller
```

### 4.2 Regra de camadas (direção de dependência)

```
Controller → Service → Repository → PrismaService
Controller → Service → Infrastructure (via módulo)
Guard → Service (quando necessário)
Consumer (RabbitMQ) → Service
```

> ⚙️ **Regra:** controllers nunca acessam repositories diretamente. Infrastructure é sempre injetada via módulo, nunca importada diretamente nos módulos de domínio.

---

## 5. Convenções de Nomenclatura

| Contexto | Convenção | Exemplo |
|---|---|---|
| **Pastas de módulo** | `kebab-case` | `chat-session/`, `rag-ingest/` |
| **Arquivos TypeScript** | `kebab-case.tipo.ts` | `chat.service.ts`, `create-message.dto.ts` |
| **Classes NestJS (Controller, Service, etc.)** | `PascalCase` | `ChatService`, `ProposalController` |
| **DTOs** | `PascalCase + Dto` | `CreateMessageDto`, `AcceptProposalDto` |
| **Entities** | `PascalCase + Entity` | `ChatMessageEntity`, `ProposalEntity` |
| **Interfaces** | `PascalCase + Interface` ou `I + PascalCase` | `ChatServiceInterface` ou `IChatService` |
| **Enums** | `PascalCase` | `ProposalStatus`, `EscrowStatus` |
| **Constantes** | `UPPER_SNAKE_CASE` | `MAX_MESSAGES_PER_HOUR`, `ESCROW_DEPOSIT_DAYS` |
| **Variáveis e parâmetros** | `camelCase` | `cedenteId`, `proposalStatus` |
| **Testes unitários** | `kebab-case.spec.ts` | `chat.service.spec.ts` |
| **Testes e2e** | `kebab-case.e2e-spec.ts` | `chat.e2e-spec.ts` |
| **Arquivos de configuração** | `kebab-case` | `configuration.ts`, `jest.config.ts` |
| **Variáveis de ambiente** | `UPPER_SNAKE_CASE` | `DATABASE_URL`, `OPENAI_API_KEY` |
| **Endpoints de API** | `kebab-case`, plural para coleções | `/chat/sessions`, `/proposals/:id/accept` |
| **Tipos utilitários** | `PascalCase + Type` | `ChatMessageType`, `ProposalType` |

---

## 6. Cache Redis — Convenção de Chaves

Prefixo do produto: `dani` — todas as chaves seguem `dani:{scope}:{id}`.

| Chave (padrão) | TTL | Contexto |
|---|---|---|
| `dani:rate_limit:{cedente_id}:{hour_timestamp}` | 3600s | Contador de mensagens por hora (janela deslizante) |
| `dani:agent_state:{session_id}` | 1800s (30min inatividade) | Estado do grafo LangGraph por sessão |
| `dani:llm_cache:exact:{sha256(prompt)}` | 3600s | Cache exato de resposta LLM para prompts idênticos |
| `dani:llm_cache:semantic:{embedding_hash}` | 1800s | Cache semântico de resposta LLM para prompts similares |
| `dani:opportunity:{cedente_id}:{opportunity_id}` | 300s | Dados da oportunidade ativa do Cedente |
| `dani:error_count:{15min_window_key}` | 900s | Contador de erros na janela de 15 minutos (fallback threshold) |
| `dani:agent_status:global` | Sem TTL (manual) | Estado global do agente (ENABLED / DISABLED) |
| `dani:session:{session_id}:messages_count` | 3600s | Contagem de mensagens na sessão atual |

**Regras obrigatórias:**
- Toda chave começa com `dani:` — nunca sem prefixo
- Chaves com dado pessoal nunca: CPF, nome, email — apenas IDs
- `dani:agent_status:global` é a única chave sem TTL automático (kill switch)

---

## 7. Mapeamento Módulo → Error Prefix e Queues

| Módulo NestJS | Error Prefix | Queue(s) RabbitMQ |
|---|---|---|
| `auth` | `DCE-AUTH-` | — |
| `chat` | `DCE-CHAT-` | — |
| `agent` | `DCE-AGENT-` | — |
| `rag` | `DCE-RAG-` | `rag.ingest` (producer) |
| `opportunity` | `DCE-OPP-` | — |
| `proposal` | `DCE-PROP-` | — |
| `dossier` | `DCE-DOS-` | — |
| `escrow` | `DCE-ESCROW-` | `escrow.events` (producer + consumer) |
| `notification` | `DCE-NOTIF-` | `notification.send` (producer + consumer) |
| `simulation` | `DCE-SIM-` | — |
| `fallback` | `DCE-FALLBACK-` | — |
| `admin` | `DCE-ADMIN-` | — |
| `infrastructure/prisma` | `DCE-DB-` | — |
| `infrastructure/redis` | `DCE-CACHE-` | — |
| `infrastructure/rabbitmq` | `DCE-QUEUE-` | — |
| `infrastructure/openai` | `DCE-LLM-` | — |
| `infrastructure/langfuse` | `DCE-TRACE-` | — |

### 7.1 Padrão de Error Code

```typescript
// Formato: DCE-{MODULE}-{HTTP_STATUS}{SEQUENCE}
// Exemplo:
throw new NotFoundException('DCE-PROP-404001', 'Proposta não encontrada');
throw new UnauthorizedException('DCE-AUTH-401001', 'Token inválido');
throw new TooManyRequestsException('DCE-CHAT-429001', 'Rate limit excedido');
```

---

## 8. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. Monorepo single-repo NestJS backend-only. 12 módulos de domínio com estrutura Controller→Service→Repository→DTO→Entity. Cache Redis 8 chaves com TTL. Error Prefixes 17 módulos/infra. Filas RabbitMQ 3 queues. Convenções de nomenclatura 14 contextos. |

---

## 9. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Separação `rag.ingest.service.ts` em módulo próprio vs. dentro do módulo `rag` | DECISÃO AUTÔNOMA | Seção 3.2 (rag) | Mantido dentro do módulo `rag` para evitar dependência circular com `knowledge_embeddings`. Alternativa: módulo `rag-ingest` separado (mais isolado, mais modules). | P2 | Engenharia | Validar |
| Módulo `admin` — separar controllers por domínio vs. único controller | DECISÃO AUTÔNOMA | Seção 3.2 (admin) | Adotado único controller `admin.controller.ts` com prefixo `/admin` e guards. Alternativa: controllers separados por domínio (`admin-chat.controller.ts`, etc.) — aumenta a granularidade mas fragmenta a camada admin. | P2 | Engenharia | Validar |
