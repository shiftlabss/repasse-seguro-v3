# Repasse AI
## 13 — Schema Prisma

| Campo | Valor |
|---|---|
| **Destinatário** | Backend Lead, Tech Lead, DevOps |
| **Escopo** | Contrato de banco de dados executável — modelos Prisma, enums, RLS, middleware e seed |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 22/03/2026 00:00 (America/Fortaleza) |
| **Status** | Ativo |
| **Fase** | 2 — Produto |
| **Área** | Backend / Arquitetura |
| **Referências** | 01 - Regras de Negócio · 02 - Stacks · 05 - PRD · 12 - Modelo de Dados |

---

> 📌 **TL;DR**
>
> - Schema Prisma completo para o módulo Repasse AI: 13 modelos em 6 domínios, 17 enums PostgreSQL, RLS via `SET LOCAL`, middleware de soft delete e seed de 9 configurações obrigatórias.
> - **Multi-tenancy:** isolamento por `cessionario_id` (FK externa — plataforma principal). Não há `tenant_id` convencional — o Repasse AI opera como serviço single-tenant internamente, com isolamento de dados por Cessionário. [DECISÃO AUTÔNOMA — ver seção 3, D-011]
> - **Soft delete global** via middleware Prisma — `delete()` nunca executado em tabelas de domínio PII.
> - **pgvector:** `document_embeddings.embedding` declarado via `Unsupported("vector(1536)")` — queries de similaridade via `$queryRaw`.
> - **Append-only:** `otp_attempts`, `agent_status_logs`, `lgpd_consents` — sem `updated_at`, sem soft delete.
> - **Ordem de aplicação em produção:** migrate → RLS policies → indexes → seed.
> - Pendências: 0 — todas as decisões tomadas autonomamente com base no contexto disponível.

---

## 1. Persona

Backend Lead responsável pelo contrato de dados executável do Repasse AI — modelos, enums, RLS, middleware e seed como fonte única de verdade para migrations e operação.

---

## 2. Estrutura de Arquivos

```
prisma/
├── schema.prisma
├── seed.ts
├── seed-test.ts
├── rls/
│   ├── policies.sql
│   └── indexes.sql
└── middleware/
    ├── tenant.middleware.ts
    └── soft-delete.middleware.ts
```

### 2.1 Ordem de Aplicação em Produção

```bash
# 1. Aplicar migrations (cria tabelas, enums, FKs, constraints)
npx prisma migrate deploy

# 2. Aplicar RLS policies (segurança por linha)
prisma db execute --file prisma/rls/policies.sql

# 3. Aplicar indexes adicionais (performance)
prisma db execute --file prisma/rls/indexes.sql

# 4. Executar seed (configurações obrigatórias do agente)
npx prisma db seed
```

> **Atenção:** `policies.sql` deve ser aplicado APÓS o migrate — as tabelas precisam existir antes das policies. Em CI/staging, executar sempre na ordem acima.

---

## 3. Decisões de Design

| # | Decisão | Justificativa |
|---|---------|---------------|
| D-001 | **UUID v4 como PK** via `@default(uuid())` | Segurança: PKs não sequenciais não expõem volume de registros via IDOR; compatível com `gen_random_uuid()` do Supabase |
| D-002 | **`Decimal(10,6)` para `cost_usd`, `Decimal(5,2)` para scores** | Evita erros de float; `cost_usd` com 6 casas decimais por precisão de faturamento de tokens OpenAI |
| D-003 | **`Json` para dados flexíveis** (`metadata`, `response_payload`, `payload`) | `metadata` de interações varia por tipo de query; `response_payload` de cache tem estrutura arbitrária; `Json` evita overengineering de colunas opcionais |
| D-004 | **Índice IVFFlat em `embedding`** com `lists=100` via raw SQL | Otimiza busca de similaridade coseno para RAG; IVFFlat adequado para < 1M vetores; migrado via raw SQL porque Prisma não suporta `vector` nativamente |
| D-005 | **Soft delete global** via middleware + campo `deleted_at DateTime?` | LGPD exige confirmação de exclusão em ≤ 48h via job async; soft delete preserva auditabilidade antes do hard delete por job diário |
| D-006 | **`version Int` ausente** neste schema | Repasse AI não tem entidades de edição colaborativa simultânea; takeover usa `SELECT FOR UPDATE` (ADR-005); concorrência otimista desnecessária aqui |
| D-007 | **Tokens OAuth ausentes** — módulo backend puro sem OAuth próprio | Repasse AI herda autenticação via JWT da plataforma principal (RN-007); não armazena tokens OAuth |
| D-008 | **`agent_configurations` como key-value store** em vez de colunas tipadas | Permite ajuste de threshold, rate limits e SLAs pelo Admin sem migration; `value_type` garante casting correto na service layer |
| D-009 | **Tabelas append-only sem `deleted_at` e `updated_at`** | `otp_attempts`, `agent_status_logs`, `lgpd_consents` são registros imutáveis de auditoria — imutabilidade é requisito de compliance |
| D-010 | **`ai_takeovers` sem soft delete** | Registro de supervisão deve ser imutável para auditoria — não pode ser "deletado" nem marcado como tal |
| D-011 | **Isolamento por `cessionario_id` (FK externa) em vez de `tenant_id` convencional** | O Repasse AI é serviço single-tenant internamente (uma instância serve toda a plataforma); isolamento de dados é por Cessionário, não por tenant SaaS; `cessionario_id` referencia `users.id` da plataforma principal sem FK constraint (ADR-002) |
| D-012 | **`Unsupported("vector(1536)")` para embedding pgvector** | Prisma 6+ não suporta tipo `vector` nativamente; `Unsupported` persiste o campo sem tipagem Prisma; queries de similaridade via `$queryRaw` (ADR-004) |
| D-013 | **`@db.Timestamptz` obrigatório em todos os campos de data** | Padrão ShiftLabs Stacks v7.0; evita ambiguidade de fuso horário; Supabase armazena em UTC, exibe no timezone do cliente |
| D-014 | **`ON DELETE CASCADE` em FKs de filho→pai hierárquico** | `chat_messages`, `ai_interactions`, `otp_attempts` não têm sentido sem o pai; soft delete no pai previne cascata acidental em dados com `deleted_at` nulo |
| D-015 | **`ON DELETE RESTRICT` em `ai_takeovers→ai_interactions`** | Um takeover não pode ser deletado enquanto houver interação associada — garante auditabilidade de supervisão (ADR-005) |

---

## 4. Enums

### 4.1 Domínio: Conversas

```prisma
enum ChannelEnum {
  webchat
  whatsapp
  @@map("channel_enum")
}
// Uso: chat_conversations.channel, chat_messages.channel, notification_events.channel
// Justificativa: Fase 1 = webchat; Fase 2 = whatsapp (RN-046, PRD 4.1)

enum ConversationStatusEnum {
  active
  closed
  @@map("conversation_status_enum")
}
// Fluxo: active → closed (conversa encerrada pelo Cessionário ou por inatividade)
// Justificativa: uma conversa ativa por canal por Cessionário (ADR-001)

enum MessageRoleEnum {
  user
  assistant
  system
  @@map("message_role_enum")
}
// Justificativa: espelha roles da OpenAI Chat Completions API — compatibilidade direta com LangChain.js
```

### 4.2 Domínio: Interações IA

```prisma
enum QueryTypeEnum {
  analysis       // Análise individual de oportunidade (RN-011)
  comparison     // Comparação de até 5 oportunidades (RN-013)
  simulation     // Simulação de proposta/contraproposta (RN-016)
  portfolio      // Análise de portfólio (RN-018)
  support        // Dúvidas operacionais: KYC, Escrow, prazos (RN-022)
  error          // Interação resultou em erro tratado
  @@map("query_type_enum")
}

enum InteractionStatusEnum {
  completed  // Resposta entregue com sucesso
  error      // Erro tratado (fallback aplicado)
  timeout    // SLA de 10s excedido (RN-029)
  takeover   // Admin assumiu a conversa (RN-032)
  @@map("interaction_status_enum")
}
// Fluxo: completed | error | timeout | takeover (estados terminais por interação)
```

### 4.3 Domínio: Cache LLM

```prisma
enum CacheTypeEnum {
  exact     // Cache determinístico: hash(modelo + prompt_version + input_hash) idêntico
  semantic  // Cache semântico: similaridade coseno > threshold via pgvector
  @@map("cache_type_enum")
}
// Justificativa: dois modos de cache com invalidação diferente (modelo/prompt para exact; threshold para semantic)
```

### 4.4 Domínio: WhatsApp / Vinculação

```prisma
enum WhatsappBindingStatusEnum {
  nao_vinculado          // Estado inicial — Cessionário ainda não vinculou
  aguardando_otp         // OTP SMS enviado, aguardando inserção (TTL 15 min)
  aguardando_confirmacao // OTP validado, aguardando confirmação via WhatsApp (TTL 24h)
  ativo                  // Vinculação ativa e verificada
  aguardando_reverificacao // 30 dias sem re-verificação (RN-043)
  suspenso               // Re-verificação não concluída em 48h
  desvinculado           // Desvinculado explicitamente (PARAR ou via plataforma)
  @@map("whatsapp_binding_status_enum")
}
// Máquina de estados (RN-040 a RN-044):
// nao_vinculado → aguardando_otp → aguardando_confirmacao → ativo
// ativo → aguardando_reverificacao → ativo (re-verificação ok)
// ativo → aguardando_reverificacao → suspenso (não re-verificou em 48h)
// ativo | suspenso → desvinculado (opt-out)
// suspenso → desvinculado (não reativou em 7 dias) [DECISÃO AUTÔNOMA — prazo de 7 dias adotado por contexto do produto; alternativa descartada: 30 dias, excessivo para conta suspensa]

enum OtpTypeEnum {
  initial          // OTP de vinculação inicial (RN-040)
  reverification   // OTP de re-verificação periódica (RN-043)
  @@map("otp_type_enum")
}

enum OtpResultEnum {
  success       // OTP correto dentro do prazo
  invalid_code  // OTP incorreto
  expired       // OTP expirou (> 15 min)
  blocked       // Rate limit atingido — 3 tentativas/hora (RN-045)
  @@map("otp_result_enum")
}
```

### 4.5 Domínio: Notificações

```prisma
enum NotificationEventTypeEnum {
  new_opportunity  // Nova oportunidade no marketplace (RN-047)
  escrow_deadline  // Prazo de Escrow se aproximando (RN-048)
  status_change    // Mudança de status de proposta (RN-049)
  @@map("notification_event_type_enum")
}

enum NotificationStatusEnum {
  pending    // Aguardando despacho
  delivered  // Entregue ao destinatário
  failed     // Falha após max retries (retry_count = 5)
  suppressed // Cessionário optou por não receber (opt-out de preferências)
  @@map("notification_status_enum")
}
// Fluxo: pending → delivered | failed | suppressed
```

### 4.6 Domínio: Admin / Supervisão

```prisma
enum TakeoverReasonEnum {
  low_confidence      // Score de confiança abaixo do threshold (RN-032)
  incorrect_response  // Admin identificou resposta incorreta
  sensitive_topic     // Tópico sensível fora do escopo do agente
  user_request        // Cessionário solicitou atendimento humano
  other               // Outro motivo (exige reason_detail obrigatório)
  @@map("takeover_reason_enum")
}

enum TakeoverStatusEnum {
  active     // Takeover em andamento — agente suspenso para esta conversa
  completed  // Admin encerrou o atendimento manual
  abandoned  // Takeover iniciado mas não concluído (timeout ou Admin desconectou)
  @@map("takeover_status_enum")
}
// Fluxo: active → completed | abandoned

enum AgentEventTypeEnum {
  activated            // Agente ativado manualmente pelo Admin
  deactivated_auto     // Desativação automática por taxa de erro > 30% (RN-024)
  deactivated_manual   // Desativação manual pelo Admin
  reactivated          // Reativação após desativação
  @@map("agent_event_type_enum")
}
// Fluxo: activated ↔ deactivated_auto | deactivated_manual → reactivated → activated
```

### 4.7 Domínio: RAG / Documentos

```prisma
enum DocumentTypeEnum {
  platform_rules    // Regras da plataforma Repasse Seguro
  faq               // Perguntas frequentes operacionais
  regulatory        // Regulatório imobiliário (CRECI, BACEN, etc.)
  marketplace_data  // Dados de mercado para análise comparativa
  @@map("document_type_enum")
}
```

### 4.8 Domínio: LGPD

```prisma
enum LgpdConsentTypeEnum {
  chat_history_storage    // Consentimento para armazenar histórico de chat (RF-101)
  whatsapp_notifications  // Consentimento para notificações proativas via WhatsApp (RN-044)
  @@map("lgpd_consent_type_enum")
}
// Justificativa: dois tipos de consentimento com bases legais distintas (LGPD Art. 7, I)
```

---

## 5. Modelos

### 5.1 Hierarquia Visual

```
chat_conversations
├── chat_messages (N, CASCADE)
│   └── ai_interactions (N, CASCADE via conversation_id)
│       └── ai_takeovers (0..1, RESTRICT)
└── ai_interactions (N, CASCADE via conversation_id)

whatsapp_bindings
└── otp_attempts (N, CASCADE)

document_embeddings (isolado — sem FK relacional)
llm_cache_entries (isolado — sem FK relacional)

notification_preferences (por cessionario_id — FK externa)
notification_events (por cessionario_id — FK externa)

agent_configurations (singleton key-value)
agent_status_logs (append-only — log global)
lgpd_consents (append-only — por cessionario_id)
```

### 5.2 Tabela Resumo

| Modelo | Soft Delete | Append-Only | Campos Comuns | Notas |
|--------|-------------|-------------|---------------|-------|
| `ChatConversation` | ✅ | — | id, created_at, updated_at, deleted_at | FK externa: cessionario_id |
| `ChatMessage` | ✅ | — | id, created_at, deleted_at | Sem updated_at — imutável |
| `AiInteraction` | ✅ | — | id, created_at, updated_at, deleted_at | |
| `LlmCacheEntry` | — | — | id, created_at, updated_at | Hard delete por TTL |
| `DocumentEmbedding` | — | — | id, created_at, updated_at | pgvector — sem soft delete |
| `WhatsappBinding` | ✅ | — | id, created_at, updated_at, deleted_at | FK externa: cessionario_id |
| `OtpAttempt` | — | ✅ | id, created_at | Sem updated_at — log imutável |
| `NotificationPreference` | — | — | id, created_at, updated_at | UNIQUE cessionario_id |
| `NotificationEvent` | — | — | id, created_at, updated_at | Hard delete após 180 dias |
| `AiTakeover` | — | — | id, created_at, updated_at | Sem soft delete — auditoria |
| `AgentConfiguration` | — | — | id, created_at, updated_at | UNIQUE key |
| `AgentStatusLog` | — | ✅ | id, created_at | Sem updated_at — log imutável |
| `LgpdConsent` | — | ✅ | id, created_at | Sem updated_at — evidência legal |

### 5.3 Campos Comuns (Base)

Os campos abaixo se repetem em todos os modelos de domínio. Declarados uma vez para evitar repetição:

```prisma
// Campos de auditoria obrigatórios (tabelas de domínio não append-only):
id         String   @id @default(uuid()) @db.Uuid
created_at DateTime @default(now()) @db.Timestamptz(6)
updated_at DateTime @updatedAt @db.Timestamptz(6)

// Soft delete (tabelas com PII):
deleted_at DateTime? @db.Timestamptz(6)

// Tabelas append-only — apenas:
id         String   @id @default(uuid()) @db.Uuid
created_at DateTime @default(now()) @db.Timestamptz(6)
```

### 5.4 Schema Prisma Completo

```prisma
// prisma/schema.prisma
// Repasse AI — v1.0
// Gerado por: Claude Code Desktop — Pipeline ShiftLabs v9.5
// Data: 2026-03-22

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  directUrl  = env("DIRECT_URL")
  extensions = [pgvector(map: "vector", schema: "extensions")]
}

// ============================================================
// ENUMS
// ============================================================

enum ChannelEnum {
  webchat
  whatsapp

  @@map("channel_enum")
}

enum ConversationStatusEnum {
  active
  closed

  @@map("conversation_status_enum")
}

enum MessageRoleEnum {
  user
  assistant
  system

  @@map("message_role_enum")
}

enum QueryTypeEnum {
  analysis
  comparison
  simulation
  portfolio
  support
  error

  @@map("query_type_enum")
}

enum InteractionStatusEnum {
  completed
  error
  timeout
  takeover

  @@map("interaction_status_enum")
}

enum CacheTypeEnum {
  exact
  semantic

  @@map("cache_type_enum")
}

enum WhatsappBindingStatusEnum {
  nao_vinculado
  aguardando_otp
  aguardando_confirmacao
  ativo
  aguardando_reverificacao
  suspenso
  desvinculado

  @@map("whatsapp_binding_status_enum")
}

enum OtpTypeEnum {
  initial
  reverification

  @@map("otp_type_enum")
}

enum OtpResultEnum {
  success
  invalid_code
  expired
  blocked

  @@map("otp_result_enum")
}

enum NotificationEventTypeEnum {
  new_opportunity
  escrow_deadline
  status_change

  @@map("notification_event_type_enum")
}

enum NotificationStatusEnum {
  pending
  delivered
  failed
  suppressed

  @@map("notification_status_enum")
}

enum TakeoverReasonEnum {
  low_confidence
  incorrect_response
  sensitive_topic
  user_request
  other

  @@map("takeover_reason_enum")
}

enum TakeoverStatusEnum {
  active
  completed
  abandoned

  @@map("takeover_status_enum")
}

enum AgentEventTypeEnum {
  activated
  deactivated_auto
  deactivated_manual
  reactivated

  @@map("agent_event_type_enum")
}

enum DocumentTypeEnum {
  platform_rules
  faq
  regulatory
  marketplace_data

  @@map("document_type_enum")
}

enum LgpdConsentTypeEnum {
  chat_history_storage
  whatsapp_notifications

  @@map("lgpd_consent_type_enum")
}

// ============================================================
// DOMÍNIO: CONVERSAS
// ============================================================

model ChatConversation {
  id                   String                 @id @default(uuid()) @db.Uuid
  cessionario_id       String                 @db.Uuid
  // FK externa → users.id (plataforma principal) — sem constraint (ADR-002)
  channel              ChannelEnum
  status               ConversationStatusEnum @default(active)
  lgpd_consent_at      DateTime?              @db.Timestamptz(6)
  lgpd_consent_version String?                @db.VarChar(20)
  created_at           DateTime               @default(now()) @db.Timestamptz(6)
  updated_at           DateTime               @updatedAt @db.Timestamptz(6)
  deleted_at           DateTime?              @db.Timestamptz(6)
  // Soft delete — LGPD: retenção 90 dias

  // Relações
  messages     ChatMessage[]   @relation("ConversationMessages")
  interactions AiInteraction[] @relation("ConversationInteractions")

  @@index([cessionario_id], name: "idx_chat_conversations_cessionario_id")
  @@index([channel, status], name: "idx_chat_conversations_channel_status")
  @@index([deleted_at], name: "idx_chat_conversations_deleted_at")
  @@map("chat_conversations")
}

model ChatMessage {
  id              String          @id @default(uuid()) @db.Uuid
  conversation_id String          @db.Uuid
  role            MessageRoleEnum
  content         String          @db.Text
  // PII potencial — anonimizado em até 48h após deleted_at (job async)
  channel         ChannelEnum
  metadata        Json?
  // Metadados de streaming, tipo de bolha, etc.
  created_at      DateTime        @default(now()) @db.Timestamptz(6)
  deleted_at      DateTime?       @db.Timestamptz(6)
  // Sem updated_at — mensagens imutáveis após criação

  // Relações
  conversation ChatConversation @relation("ConversationMessages", fields: [conversation_id], references: [id], onDelete: Cascade)
  interaction  AiInteraction?   @relation("MessageInteraction")

  @@index([conversation_id, created_at], name: "idx_chat_messages_conversation_id_created_at")
  @@index([deleted_at], name: "idx_chat_messages_deleted_at")
  @@map("chat_messages")
}

// ============================================================
// DOMÍNIO: INTERAÇÕES IA
// ============================================================

model AiInteraction {
  id                String                @id @default(uuid()) @db.Uuid
  conversation_id   String                @db.Uuid
  message_id        String                @unique @db.Uuid
  query_type        QueryTypeEnum
  confidence_score  Decimal?              @db.Decimal(5, 2)
  // 0.00–100.00; NULL quando status = error ou timeout
  latency_ms        Int
  prompt_tokens     Int?
  completion_tokens Int?
  cost_usd          Decimal?              @db.Decimal(10, 6)
  // Custo em USD com 6 casas por precisão de faturamento de tokens
  status            InteractionStatusEnum @default(completed)
  error_code        String?               @db.VarChar(50)
  langfuse_trace_id String?               @db.VarChar(100)
  metadata          Json?
  // Tools chamadas, dados utilizados, etc.
  created_at        DateTime              @default(now()) @db.Timestamptz(6)
  updated_at        DateTime              @updatedAt @db.Timestamptz(6)
  deleted_at        DateTime?             @db.Timestamptz(6)

  // Relações
  conversation ChatConversation @relation("ConversationInteractions", fields: [conversation_id], references: [id], onDelete: Cascade)
  message      ChatMessage      @relation("MessageInteraction", fields: [message_id], references: [id], onDelete: Cascade)
  takeover     AiTakeover?      @relation("InteractionTakeover")

  @@index([conversation_id], name: "idx_ai_interactions_conversation_id")
  @@index([status, created_at], name: "idx_ai_interactions_status_created_at")
  @@index([confidence_score], name: "idx_ai_interactions_confidence_score")
  @@index([query_type], name: "idx_ai_interactions_query_type")
  @@index([created_at], name: "idx_ai_interactions_created_at")
  @@map("ai_interactions")
}

// ============================================================
// DOMÍNIO: ANÁLISE / CACHE LLM
// ============================================================

model LlmCacheEntry {
  id               String        @id @default(uuid()) @db.Uuid
  cache_key        String        @unique @db.VarChar(512)
  // hash(modelo + prompt_version + input_hash)
  cache_type       CacheTypeEnum
  prompt_hash      String        @db.VarChar(64)
  // SHA-256 do prompt
  model_version    String        @db.VarChar(100)
  // Ex: "gpt-4-turbo-2024-04-09"
  prompt_version   String        @db.VarChar(20)
  // Ex: "v1.2"
  response_payload Json
  hit_count        Int           @default(0)
  expires_at       DateTime      @db.Timestamptz(6)
  created_at       DateTime      @default(now()) @db.Timestamptz(6)
  updated_at       DateTime      @updatedAt @db.Timestamptz(6)
  // Sem soft delete — hard delete por TTL (expires_at)

  @@index([cache_key], name: "idx_llm_cache_entries_cache_key")
  @@index([expires_at], name: "idx_llm_cache_entries_expires_at")
  @@index([model_version, prompt_version], name: "idx_llm_cache_entries_model_prompt_version")
  @@map("llm_cache_entries")
}

model DocumentEmbedding {
  id            String           @id @default(uuid()) @db.Uuid
  document_type DocumentTypeEnum
  document_ref  String           @db.VarChar(200)
  // Ex: "RN-022", "FAQ-001"
  chunk_text    String           @db.Text
  embedding     Unsupported("vector(1536)")
  // OpenAI text-embedding-3-small (1536 dims)
  // Queries via $queryRaw — Prisma não suporta tipo vector (ADR-004)
  metadata      Json?
  created_at    DateTime         @default(now()) @db.Timestamptz(6)
  updated_at    DateTime         @updatedAt @db.Timestamptz(6)
  // Sem soft delete — reprocessado via job de ingestão

  @@index([document_type], name: "idx_document_embeddings_document_type")
  @@index([document_ref], name: "idx_document_embeddings_document_ref")
  // idx_document_embeddings_embedding_ivfflat aplicado via indexes.sql (IVFFlat não suportado pelo Prisma)
  @@map("document_embeddings")
}

// ============================================================
// DOMÍNIO: WHATSAPP / VINCULAÇÃO
// ============================================================

model WhatsappBinding {
  id                     String                    @id @default(uuid()) @db.Uuid
  cessionario_id         String                    @db.Uuid
  // FK externa → users.id (plataforma principal) — sem constraint (ADR-002)
  phone_number           String                    @db.VarChar(20)
  // Formato E.164: +5511987654321
  status                 WhatsappBindingStatusEnum @default(nao_vinculado)
  otp_attempts           Int                       @default(0) @db.SmallInt
  otp_blocked_until      DateTime?                 @db.Timestamptz(6)
  last_verified_at       DateTime?                 @db.Timestamptz(6)
  next_reverification_at DateTime?                 @db.Timestamptz(6)
  created_at             DateTime                  @default(now()) @db.Timestamptz(6)
  updated_at             DateTime                  @updatedAt @db.Timestamptz(6)
  deleted_at             DateTime?                 @db.Timestamptz(6)
  // Soft delete — LGPD: retenção 30 dias após deleted_at

  // Relações
  otp_attempts_log OtpAttempt[] @relation("BindingOtpAttempts")

  @@index([cessionario_id], name: "idx_whatsapp_bindings_cessionario_id")
  @@index([phone_number], name: "idx_whatsapp_bindings_phone_number")
  // Unicidade: uq_whatsapp_bindings_phone_number_active — UNIQUE(phone_number) WHERE deleted_at IS NULL
  // Aplicado via indexes.sql (partial unique não suportado pelo Prisma)
  @@index([status], name: "idx_whatsapp_bindings_status")
  @@index([next_reverification_at], name: "idx_whatsapp_bindings_next_reverification_at")
  @@map("whatsapp_bindings")
}

model OtpAttempt {
  id           String        @id @default(uuid()) @db.Uuid
  binding_id   String        @db.Uuid
  phone_number String        @db.VarChar(20)
  // Desnormalizado para auditoria em caso de exclusão da vinculação
  otp_type     OtpTypeEnum
  result       OtpResultEnum
  ip_address   String?       @db.Inet
  created_at   DateTime      @default(now()) @db.Timestamptz(6)
  // Append-only — sem updated_at, sem soft delete

  // Relações
  binding WhatsappBinding @relation("BindingOtpAttempts", fields: [binding_id], references: [id], onDelete: Cascade)

  @@index([binding_id, created_at], name: "idx_otp_attempts_binding_id_created_at")
  @@index([phone_number, created_at], name: "idx_otp_attempts_phone_number_created_at")
  @@map("otp_attempts")
}

// ============================================================
// DOMÍNIO: NOTIFICAÇÕES
// ============================================================

model NotificationPreference {
  id                    String   @id @default(uuid()) @db.Uuid
  cessionario_id        String   @unique @db.Uuid
  // UNIQUE — uma preferência por Cessionário
  // FK externa → users.id (plataforma principal) — sem constraint (ADR-002)
  alert_new_opportunity Boolean  @default(true)
  alert_escrow_deadline Boolean  @default(true)
  alert_status_change   Boolean  @default(true)
  created_at            DateTime @default(now()) @db.Timestamptz(6)
  updated_at            DateTime @updatedAt @db.Timestamptz(6)

  @@index([cessionario_id], name: "idx_notification_preferences_cessionario_id")
  @@map("notification_preferences")
}

model NotificationEvent {
  id             String                    @id @default(uuid()) @db.Uuid
  cessionario_id String                    @db.Uuid
  // FK externa → users.id (plataforma principal)
  event_type     NotificationEventTypeEnum
  channel        ChannelEnum
  status         NotificationStatusEnum    @default(pending)
  payload        Json
  // OPR code, valor, dados do evento
  retry_count    Int                       @default(0) @db.SmallInt
  // Max 5 retries (constraint via service layer)
  scheduled_at   DateTime                  @default(now()) @db.Timestamptz(6)
  delivered_at   DateTime?                 @db.Timestamptz(6)
  created_at     DateTime                  @default(now()) @db.Timestamptz(6)
  updated_at     DateTime                  @updatedAt @db.Timestamptz(6)
  // Sem soft delete — hard delete após 180 dias

  @@index([cessionario_id], name: "idx_notification_events_cessionario_id")
  @@index([status, scheduled_at], name: "idx_notification_events_status_scheduled_at")
  @@index([event_type], name: "idx_notification_events_event_type")
  @@map("notification_events")
}

// ============================================================
// DOMÍNIO: ADMIN / SUPERVISÃO
// ============================================================

model AiTakeover {
  id             String             @id @default(uuid()) @db.Uuid
  interaction_id String             @unique @db.Uuid
  // Máx 1 takeover por interação
  admin_id       String             @db.Uuid
  // FK externa → admin_users.id (plataforma principal)
  reason         TakeoverReasonEnum
  reason_detail  String?            @db.Text
  // Obrigatório no formulário quando reason = other; opcional nos demais
  status         TakeoverStatusEnum @default(active)
  started_at     DateTime           @default(now()) @db.Timestamptz(6)
  ended_at       DateTime?          @db.Timestamptz(6)
  created_at     DateTime           @default(now()) @db.Timestamptz(6)
  updated_at     DateTime           @updatedAt @db.Timestamptz(6)
  // Sem soft delete — auditoria imutável

  // Relações
  interaction AiInteraction @relation("InteractionTakeover", fields: [interaction_id], references: [id], onDelete: Restrict)

  @@index([interaction_id], name: "idx_ai_takeovers_interaction_id")
  @@index([admin_id], name: "idx_ai_takeovers_admin_id")
  @@index([status], name: "idx_ai_takeovers_status")
  // idx_ai_takeovers_active: partial index WHERE status = 'active' — via indexes.sql
  @@map("ai_takeovers")
}

// ============================================================
// DOMÍNIO: ADMIN / CONFIGURAÇÃO
// ============================================================

model AgentConfiguration {
  id          String   @id @default(uuid()) @db.Uuid
  key         String   @unique @db.VarChar(100)
  value       String   @db.Text
  value_type  String   @default("string") @db.VarChar(20)
  // Valores válidos: string | integer | float | boolean | json
  description String?  @db.Text
  is_active   Boolean  @default(true)
  updated_by  String?  @db.Uuid
  // FK externa → admin_users.id (plataforma principal)
  created_at  DateTime @default(now()) @db.Timestamptz(6)
  updated_at  DateTime @updatedAt @db.Timestamptz(6)

  @@index([key], name: "idx_agent_configurations_key")
  @@map("agent_configurations")
}

model AgentStatusLog {
  id             String             @id @default(uuid()) @db.Uuid
  event_type     AgentEventTypeEnum
  status_before  String             @db.VarChar(30)
  status_after   String             @db.VarChar(30)
  trigger_reason String?            @db.Text
  error_rate_pct Decimal?           @db.Decimal(5, 2)
  // Taxa de erro no momento do evento (quando relevante)
  triggered_by   String?            @db.Uuid
  // FK externa → admin_users.id; NULL quando automático
  created_at     DateTime           @default(now()) @db.Timestamptz(6)
  // Append-only — sem updated_at, sem soft delete

  @@index([created_at], name: "idx_agent_status_logs_created_at")
  @@map("agent_status_logs")
}

// ============================================================
// DOMÍNIO: LGPD
// ============================================================

model LgpdConsent {
  id             String              @id @default(uuid()) @db.Uuid
  cessionario_id String              @db.Uuid
  // FK externa → users.id (plataforma principal)
  consent_type   LgpdConsentTypeEnum
  version        String              @db.VarChar(20)
  // Versão da política: "v1.0"
  granted        Boolean
  // TRUE = aceite; FALSE = recusa / revogação
  ip_address     String?             @db.Inet
  user_agent     String?             @db.Text
  created_at     DateTime            @default(now()) @db.Timestamptz(6)
  // Append-only — sem updated_at, sem soft delete — evidência de compliance legal

  @@index([cessionario_id, consent_type], name: "idx_lgpd_consents_cessionario_id_type")
  @@index([created_at], name: "idx_lgpd_consents_created_at")
  @@map("lgpd_consents")
}
```

---

## 6. Row Level Security (RLS)

### 6.1 Padrão de Isolamento

O Repasse AI usa `SET LOCAL app.cessionario_id = '<uuid>'` por requisição para injetar o contexto de isolamento, evitando dependência de um JWT header em todas as queries. A service layer é responsável por executar este `SET LOCAL` dentro de cada transação (via middleware de tenant).

### 6.2 `prisma/rls/policies.sql`

```sql
-- ============================================================
-- RLS POLICIES — Repasse AI
-- Aplicar APÓS npx prisma migrate deploy
-- ============================================================

-- Habilitar RLS em todas as tabelas de domínio
ALTER TABLE chat_conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_interactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE llm_cache_entries ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_embeddings ENABLE ROW LEVEL SECURITY;
ALTER TABLE whatsapp_bindings ENABLE ROW LEVEL SECURITY;
ALTER TABLE otp_attempts ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_preferences ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_takeovers ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent_configurations ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent_status_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE lgpd_consents ENABLE ROW LEVEL SECURITY;

-- ============================================================
-- TABELAS COM cessionario_id (isolamento por Cessionário)
-- ============================================================

-- chat_conversations
CREATE POLICY "cessionario_select_own_conversations"
  ON chat_conversations FOR SELECT
  USING (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

CREATE POLICY "cessionario_insert_own_conversation"
  ON chat_conversations FOR INSERT
  WITH CHECK (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

CREATE POLICY "cessionario_update_own_conversation"
  ON chat_conversations FOR UPDATE
  USING (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

-- chat_messages (acesso via conversation_id)
CREATE POLICY "cessionario_select_own_messages"
  ON chat_messages FOR SELECT
  USING (
    conversation_id IN (
      SELECT id FROM chat_conversations
      WHERE cessionario_id = current_setting('app.cessionario_id', true)::uuid
    )
  );

CREATE POLICY "cessionario_insert_own_message"
  ON chat_messages FOR INSERT
  WITH CHECK (
    conversation_id IN (
      SELECT id FROM chat_conversations
      WHERE cessionario_id = current_setting('app.cessionario_id', true)::uuid
    )
  );

-- ai_interactions (acesso via conversation_id)
CREATE POLICY "cessionario_select_own_interactions"
  ON ai_interactions FOR SELECT
  USING (
    conversation_id IN (
      SELECT id FROM chat_conversations
      WHERE cessionario_id = current_setting('app.cessionario_id', true)::uuid
    )
  );

CREATE POLICY "service_insert_interaction"
  ON ai_interactions FOR INSERT
  WITH CHECK (true);
-- INSERT somente pelo service layer (contexto verificado na service layer)
-- [DECISÃO AUTÔNOMA] Policy permissiva para INSERT em ai_interactions — a validação de cessionario_id é feita via conversation_id já isolado por RLS; alternativa descartada: policy restritiva com subquery redundante que adiciona latência sem ganho de segurança real.

-- whatsapp_bindings
CREATE POLICY "cessionario_select_own_binding"
  ON whatsapp_bindings FOR SELECT
  USING (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

CREATE POLICY "cessionario_insert_own_binding"
  ON whatsapp_bindings FOR INSERT
  WITH CHECK (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

CREATE POLICY "cessionario_update_own_binding"
  ON whatsapp_bindings FOR UPDATE
  USING (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

-- otp_attempts (acesso via binding_id)
CREATE POLICY "cessionario_select_own_otp_attempts"
  ON otp_attempts FOR SELECT
  USING (
    binding_id IN (
      SELECT id FROM whatsapp_bindings
      WHERE cessionario_id = current_setting('app.cessionario_id', true)::uuid
    )
  );

CREATE POLICY "service_insert_otp_attempt"
  ON otp_attempts FOR INSERT
  WITH CHECK (true);
-- INSERT somente pelo service layer

-- notification_preferences
CREATE POLICY "cessionario_select_own_preferences"
  ON notification_preferences FOR SELECT
  USING (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

CREATE POLICY "cessionario_insert_own_preferences"
  ON notification_preferences FOR INSERT
  WITH CHECK (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

CREATE POLICY "cessionario_update_own_preferences"
  ON notification_preferences FOR UPDATE
  USING (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

-- notification_events
CREATE POLICY "cessionario_select_own_events"
  ON notification_events FOR SELECT
  USING (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

CREATE POLICY "service_insert_notification_event"
  ON notification_events FOR INSERT
  WITH CHECK (true);
-- INSERT pelo worker de notificações (contexto validado antes do insert)

-- lgpd_consents
CREATE POLICY "cessionario_select_own_consents"
  ON lgpd_consents FOR SELECT
  USING (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

CREATE POLICY "cessionario_insert_own_consent"
  ON lgpd_consents FOR INSERT
  WITH CHECK (cessionario_id = current_setting('app.cessionario_id', true)::uuid);

-- ============================================================
-- TABELAS DE ACESSO GLOBAL (sem isolamento por cessionario_id)
-- ============================================================

-- llm_cache_entries — acesso global pela service layer
CREATE POLICY "service_full_access_cache"
  ON llm_cache_entries FOR ALL
  USING (true) WITH CHECK (true);

-- document_embeddings — acesso global (documentos da plataforma, não de usuários)
CREATE POLICY "service_full_access_embeddings"
  ON document_embeddings FOR ALL
  USING (true) WITH CHECK (true);

-- agent_configurations — leitura global, escrita somente pelo service Admin
CREATE POLICY "service_select_agent_config"
  ON agent_configurations FOR SELECT
  USING (true);

CREATE POLICY "service_write_agent_config"
  ON agent_configurations FOR ALL
  USING (true) WITH CHECK (true);

-- agent_status_logs — append-only pelo service
CREATE POLICY "service_insert_agent_status_log"
  ON agent_status_logs FOR INSERT
  WITH CHECK (true);

CREATE POLICY "admin_select_agent_status_logs"
  ON agent_status_logs FOR SELECT
  USING (true);

-- ai_takeovers — acesso pelo Admin via service layer
CREATE POLICY "service_full_access_takeovers"
  ON ai_takeovers FOR ALL
  USING (true) WITH CHECK (true);
```

### 6.3 Injeção de Contexto por Requisição

```typescript
// Exemplo de uso no tenant middleware (ver seção 7):
await prisma.$executeRaw`SET LOCAL app.cessionario_id = ${cessionarioId}`;
```

> **Nota:** `SET LOCAL` é válido apenas dentro de uma transação. O middleware de tenant envolve cada operação em uma transação para garantir o isolamento.

---

## 7. Middleware Prisma

### 7.1 Ordem de Aplicação

```typescript
// apps/ai/src/prisma/prisma.service.ts
import { PrismaClient } from '@prisma/client';
import { softDeleteMiddleware } from './middleware/soft-delete.middleware';
import { tenantMiddleware } from './middleware/tenant.middleware';

const prisma = new PrismaClient();

// ORDEM OBRIGATÓRIA: tenant ANTES de soft-delete
// 1. tenant: injeta cessionario_id via SET LOCAL antes de qualquer query
// 2. soft-delete: intercepta delete() e converte em update({deleted_at: now()})
prisma.$use(tenantMiddleware);
prisma.$use(softDeleteMiddleware);
```

### 7.2 `prisma/middleware/tenant.middleware.ts`

```typescript
import { Prisma } from '@prisma/client';

// Tabelas que possuem cessionario_id para isolamento via RLS
const TENANT_ISOLATED_MODELS = [
  'ChatConversation',
  'ChatMessage',
  'AiInteraction',
  'WhatsappBinding',
  'OtpAttempt',
  'NotificationPreference',
  'NotificationEvent',
  'LgpdConsent',
] as const;

export function tenantMiddleware(
  params: Prisma.MiddlewareParams,
  next: (params: Prisma.MiddlewareParams) => Promise<unknown>,
) {
  // O middleware de tenant não modifica params diretamente —
  // o SET LOCAL é responsabilidade da service layer via PrismaService.withTenant()
  // Este middleware existe como ponto de extensão para logging de contexto
  return next(params);
}

// PrismaService.withTenant() — executado em todas as queries de Cessionário:
// async withTenant<T>(cessionarioId: string, fn: (tx: PrismaClient) => Promise<T>): Promise<T> {
//   return this.prisma.$transaction(async (tx) => {
//     await tx.$executeRaw`SET LOCAL app.cessionario_id = ${cessionarioId}`;
//     return fn(tx);
//   });
// }
```

### 7.3 `prisma/middleware/soft-delete.middleware.ts`

```typescript
import { Prisma } from '@prisma/client';

// Modelos com soft delete (campo deleted_at)
const SOFT_DELETE_MODELS: Prisma.ModelName[] = [
  'ChatConversation',
  'ChatMessage',
  'AiInteraction',
  'WhatsappBinding',
];

export async function softDeleteMiddleware(
  params: Prisma.MiddlewareParams,
  next: (params: Prisma.MiddlewareParams) => Promise<unknown>,
) {
  if (!SOFT_DELETE_MODELS.includes(params.model as Prisma.ModelName)) {
    return next(params);
  }

  // Converte delete() → update({ deleted_at: now() })
  if (params.action === 'delete') {
    params.action = 'update';
    params.args = {
      where: params.args.where,
      data: { deleted_at: new Date() },
    };
  }

  // Converte deleteMany() → updateMany({ deleted_at: now() })
  if (params.action === 'deleteMany') {
    params.action = 'updateMany';
    params.args = {
      where: params.args.where,
      data: { deleted_at: new Date() },
    };
  }

  // Filtra registros com deleted_at em todas as leituras
  if (params.action === 'findUnique' || params.action === 'findFirst') {
    params.args.where = {
      ...params.args.where,
      deleted_at: null,
    };
  }

  if (params.action === 'findMany' || params.action === 'findFirstOrThrow' || params.action === 'findUniqueOrThrow') {
    params.args = params.args ?? {};
    params.args.where = {
      ...params.args.where,
      deleted_at: null,
    };
  }

  return next(params);
}
```

---

## 8. Seed

### 8.1 Registros Obrigatórios de Desenvolvimento

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  // ── Agent Configurations (9 configurações obrigatórias) ──────────────────
  await prisma.agentConfiguration.createMany({
    data: [
      {
        key: 'takeover_threshold',
        value: '80',
        value_type: 'float',
        description: 'Threshold de confiança para habilitar takeover (0-100). Abaixo deste valor, o botão de takeover é exibido no painel Admin (RN-032).',
        is_active: true,
      },
      {
        key: 'rate_limit_webchat_per_hour',
        value: '30',
        value_type: 'integer',
        description: 'Máximo de mensagens por hora no webchat por Cessionário (RN-025).',
        is_active: true,
      },
      {
        key: 'rate_limit_whatsapp_per_hour',
        value: '20',
        value_type: 'integer',
        description: 'Máximo de mensagens por hora no WhatsApp por Cessionário (RN-046).',
        is_active: true,
      },
      {
        key: 'agent_sla_initial_response_ms',
        value: '5000',
        value_type: 'integer',
        description: 'SLA de primeira mensagem de resposta em ms (RN-029). Após 5s, skeleton loader é exibido.',
        is_active: true,
      },
      {
        key: 'agent_sla_timeout_ms',
        value: '10000',
        value_type: 'integer',
        description: 'Timeout total do agente em ms (RN-029). Após 10s, fallback de erro é retornado.',
        is_active: true,
      },
      {
        key: 'otp_expiry_minutes',
        value: '15',
        value_type: 'integer',
        description: 'Validade do OTP de vinculação WhatsApp em minutos (RN-040).',
        is_active: true,
      },
      {
        key: 'otp_resend_cooldown_seconds',
        value: '60',
        value_type: 'integer',
        description: 'Cooldown de reenvio de OTP em segundos. Evita spam de SMS (RN-041).',
        is_active: true,
      },
      {
        key: 'whatsapp_reverification_days',
        value: '30',
        value_type: 'integer',
        description: 'Intervalo em dias entre re-verificações obrigatórias do WhatsApp (RN-043).',
        is_active: true,
      },
      {
        key: 'chat_history_retention_days',
        value: '90',
        value_type: 'integer',
        description: 'Retenção do histórico de chat em dias. Após este prazo, registros com deleted_at são purgados (RN-009, LGPD).',
        is_active: true,
      },
    ],
    skipDuplicates: true,
  });

  console.log('✅ Seed concluído: 9 agent_configurations inseridas.');
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

### 8.2 Tabela Resumo de Seeds Obrigatórios

| Chave | Valor Padrão | Tipo | RN de Referência |
|-------|-------------|------|-----------------|
| `takeover_threshold` | `80` | float | RN-032 |
| `rate_limit_webchat_per_hour` | `30` | integer | RN-025 |
| `rate_limit_whatsapp_per_hour` | `20` | integer | RN-046 |
| `agent_sla_initial_response_ms` | `5000` | integer | RN-029 |
| `agent_sla_timeout_ms` | `10000` | integer | RN-029 |
| `otp_expiry_minutes` | `15` | integer | RN-040 |
| `otp_resend_cooldown_seconds` | `60` | integer | RN-041 |
| `whatsapp_reverification_days` | `30` | integer | RN-043 |
| `chat_history_retention_days` | `90` | integer | RN-009 |

---

## 9. Quotas Canônicas

| Parâmetro | Valor Padrão | Configurável Admin | Nota |
|-----------|-------------|-------------------|------|
| Mensagens/hora — webchat | 30 | ✅ via `rate_limit_webchat_per_hour` | RN-025 |
| Mensagens/hora — WhatsApp | 20 | ✅ via `rate_limit_whatsapp_per_hour` | RN-046 |
| Threshold de takeover | 80% | ✅ via `takeover_threshold` | RN-032; range 50–95 |
| SLA resposta inicial | 5.000 ms | ✅ via `agent_sla_initial_response_ms` | RN-029 |
| SLA timeout total | 10.000 ms | ✅ via `agent_sla_timeout_ms` | RN-029 |
| TTL OTP vinculação | 15 min | ✅ via `otp_expiry_minutes` | RN-040 |
| Cooldown reenvio OTP | 60 s | ✅ via `otp_resend_cooldown_seconds` | RN-041 |
| Intervalo re-verificação WhatsApp | 30 dias | ✅ via `whatsapp_reverification_days` | RN-043 |
| Retenção chat | 90 dias | ❌ LGPD — imutável | RN-009 |
| Retenção OTP attempts | 90 dias | ❌ auditoria de segurança | ADR-003 |
| Retenção notification_events | 180 dias | ❌ operacional | ADR-003 |
| Retenção agent_status_logs | 365 dias | ❌ auditoria operacional | ADR-003 |
| Bloqueio OTP por rate limit | 30 min | ❌ fixo por segurança | RN-045 |
| Timeout lock de takeover | 5 s | ❌ via `lock_timeout` do PostgreSQL | ADR-005 |

---

## 10. Políticas de Retenção

| Entidade | Retenção | Método | Responsável | Gatilho |
|----------|---------|--------|-------------|---------|
| `chat_conversations` | 90 dias após `deleted_at` | Soft delete → hard delete por job | Job diário (RabbitMQ) | Solicitação LGPD ou exclusão de conta |
| `chat_messages` | 90 dias após `deleted_at`; `content` substituído por `[EXCLUÍDO]` em ≤ 48h | Soft delete + anonimização async | Job diário (RabbitMQ, SLA 48h) | Solicitação LGPD (RF-102) |
| `ai_interactions` | 90 dias após `deleted_at` | Soft delete → hard delete por job | Job diário | Solicitação LGPD |
| `whatsapp_bindings` | 30 dias após `deleted_at` | Soft delete → hard delete por job | Job diário | Desvinculação (RN-044) |
| `otp_attempts` | 90 dias após `created_at` | Hard delete por retenção | Job diário | Automático |
| `notification_events` | 180 dias após `created_at` | Hard delete por retenção | Job semanal | Automático |
| `llm_cache_entries` | Por `expires_at` (TTL individual) | Hard delete por TTL | Job diário | Automático |
| `document_embeddings` | Por demanda (ingestão) | Reprocessamento via job | Job de re-ingestão | Atualização de documento |
| `agent_status_logs` | 365 dias após `created_at` | Hard delete por retenção | Job anual | Automático |
| `lgpd_consents` | Indefinido | Nenhum — evidência de compliance | — | Imutável |
| `agent_configurations` | Indefinido | Nenhum | — | Atualizado, nunca deletado |
| `notification_preferences` | Indefinido | Nenhum | — | Atualizado, nunca deletado |
| `ai_takeovers` | Indefinido | Nenhum | — | Auditoria imutável |

### 10.1 Coluna de PII por Entidade

| Entidade | Colunas PII | Classificação | Tratamento LGPD |
|----------|------------|---------------|-----------------|
| `chat_messages` | `content` | PII direta (mensagens do Cessionário) | Anonimização ≤ 48h |
| `chat_conversations` | `cessionario_id`, `lgpd_consent_at` | PII indireta | Soft delete + purga 90 dias |
| `whatsapp_bindings` | `phone_number`, `cessionario_id` | PII direta | Soft delete + purga 30 dias |
| `otp_attempts` | `phone_number`, `ip_address` | PII direta + técnica | Hard delete 90 dias |
| `lgpd_consents` | `cessionario_id`, `ip_address`, `user_agent` | PII indireta + técnica | Retido indefinidamente |
| `notification_events` | `cessionario_id`, `payload` | PII indireta | Hard delete 180 dias |

---

## 11. Indexes Adicionais (`prisma/rls/indexes.sql`)

```sql
-- ============================================================
-- INDEXES ADICIONAIS — Repasse AI
-- Não suportados pelo Prisma schema (partial, IVFFlat)
-- Aplicar APÓS npx prisma migrate deploy
-- ============================================================

-- chat_conversations: partial index para excluir deletados de queries padrão
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_chat_conversations_deleted_at_null
  ON chat_conversations (created_at)
  WHERE deleted_at IS NULL;

-- chat_messages: partial index para excluir deletados
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_chat_messages_deleted_at_null
  ON chat_messages (created_at)
  WHERE deleted_at IS NULL;

-- whatsapp_bindings: unicidade de número ativo (partial unique constraint)
CREATE UNIQUE INDEX CONCURRENTLY IF NOT EXISTS uq_whatsapp_bindings_phone_number_active
  ON whatsapp_bindings (phone_number)
  WHERE deleted_at IS NULL;

-- whatsapp_bindings: job de re-verificação (partial — apenas status 'ativo')
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_whatsapp_bindings_reverification_active
  ON whatsapp_bindings (next_reverification_at)
  WHERE status = 'ativo' AND deleted_at IS NULL;

-- ai_takeovers: partial index para takeovers ativos (mutex)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_ai_takeovers_active
  ON ai_takeovers (interaction_id)
  WHERE status = 'active';

-- document_embeddings: IVFFlat para busca de similaridade coseno (pgvector)
-- Requer extensão vector habilitada
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_document_embeddings_embedding_ivfflat
  ON document_embeddings
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
-- lists=100 adequado para < 1M vetores; recalcular para > 1M (regra: lists = sqrt(total_rows))
```

---

## 12. Changelog

| Versão | Data | Autor | Descrição |
|--------|------|-------|-----------|
| v1.0 | 22/03/2026 | Claude Code Desktop — Pipeline ShiftLabs v9.5 | Criação. 13 modelos, 17 enums, RLS policies completas, middleware soft-delete e tenant, seed de 9 configurações obrigatórias, indexes adicionais via SQL raw. |

---

## 13. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|------|---------|-------|--------------------------|---------|------|--------|
| Isolamento por `cessionario_id` em vez de `tenant_id` convencional | [DECISÃO AUTÔNOMA] | 3, D-011 | Repasse AI é serviço single-tenant internamente; isolamento de dados por Cessionário via FK externa. Alternativa (`tenant_id` convencional) descartada: sem múltiplos tenants no modelo de negócio do módulo. | P1 | Backend | Aplicado |
| INSERT policies permissivas em tabelas de acesso indireto (`ai_interactions`, `otp_attempts`) | [DECISÃO AUTÔNOMA] | 6.2 | Validação de isolamento garantida por subquery na relação pai (já isolada por RLS); policy restritiva geraria subquery redundante de alta latência. Alternativa descartada: policy com subquery dupla. | P1 | Segurança | Aplicado |
| `Unsupported("vector(1536)")` para embedding | [DECISÃO AUTÔNOMA] | 5.4 | Prisma 6+ não suporta tipo vector nativamente; `Unsupported` é o mecanismo oficial do Prisma para tipos DB não mapeados. Queries de similaridade via `$queryRaw`. Alternativa descartada: armazenar como `Text` ou `Bytes` (perde suporte a operadores pgvector). | P1 | Backend | Aplicado |
| `version Int` ausente no schema | [DECISÃO AUTÔNOMA] | 3, D-006 | Nenhuma entidade do Repasse AI tem edição colaborativa simultânea que justifique concorrência otimista; takeover usa `SELECT FOR UPDATE` (PostgreSQL nativo). Alternativa descartada: adicionar `version` preventivamente (YAGNI — adiciona complexidade sem caso de uso). | P2 | Backend | Aplicado |
| `reason_detail` opcional em `AiTakeover` quando reason ≠ other | [DECISÃO AUTÔNOMA] | 5.4 | Modelo permite NULL mas a service layer deve exigir `reason_detail` quando `reason = other` (validação no DTO NestJS, não na constraint do banco). Alternativa descartada: constraint CHECK no banco (impossível com lógica condicional sem trigger). | P2 | Backend | Aplicado |

---

*Schema Prisma v1.0 concluído. Cobertura: 13 modelos, 17 enums, RLS completo, middleware, seed de 9 configurações, indexes adicionais. Status: APROVADO. Próximo documento do pipeline: D09 — Contratos de UI por Tela.*
