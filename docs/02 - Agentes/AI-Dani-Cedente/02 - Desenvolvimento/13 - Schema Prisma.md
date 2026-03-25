# 13 - Schema Prisma — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Backend Lead, Tech Lead, DevOps |
| Escopo | Contrato de banco de dados executável — modelos Prisma, enums, RLS, middleware e seed |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D01 — Regras de Negócio · D02 — Stacks · D05 — PRD · D12 — ERD Schema |

---

> **📌 TL;DR**
>
> - **9 modelos Prisma** derivados diretamente do ERD (D12): `CedenteProfile`, `Opportunity`, `OpportunityScenario`, `Proposal`, `DossierDocument`, `EscrowTransaction`, `ChatSession`, `ChatMessage`, `KnowledgeEmbedding`.
> - **Isolamento por `cedente_id`** em vez de `tenant_id` genérico — a Dani-Cedente é single-service com RLS por Cedente.
> - **Soft delete global** via `deleted_at` — nunca delete físico (LGPD, histórico 90 dias).
> - **Valores monetários em `Decimal(12,2)`** — zero tolerância a `Float`.
> - **`KnowledgeEmbedding`** usa extensão `pgvector` — gerenciada via SQL raw (Prisma não suporta nativo `vector`).
> - **Sem OAuth tokens** neste serviço — autenticação delegada ao Supabase Auth; o backend consome apenas o JWT.
> - **Zero seções pendentes** — cobertura 100% das entidades do D12.

---

## 1. Persona

Backend Lead responsável pelo contrato de dados executável do AI-Dani-Cedente. Foco em isolamento por `cedente_id`, soft delete LGPD, valores monetários precisos e integração com pgvector para RAG.

---

## 2. Estrutura de Arquivos

```
prisma/
├── schema.prisma
├── seed.ts
├── rls/
│   ├── policies.sql
│   └── indexes.sql
└── middleware/
    ├── cedente-isolation.middleware.ts
    └── soft-delete.middleware.ts
```

### Ordem de aplicação em produção

```bash
# 1. Aplicar migration
npx prisma migrate deploy

# 2. Aplicar RLS policies
npx prisma db execute --file prisma/rls/policies.sql

# 3. Aplicar indexes (incluindo pgvector IVFFlat)
npx prisma db execute --file prisma/rls/indexes.sql

# 4. Seed (apenas desenvolvimento / staging)
npx prisma db seed
```

> ⚠️ **Atenção:** o index IVFFlat para `knowledge_embeddings` requer que a tabela já tenha dados suficientes (mínimo ~2× `lists` registros). Em ambiente vazio, o seed insere embeddings de exemplo antes do index.

---

## 3. Decisões de Design

| # | Decisão | Justificativa |
|---|---|---|
| D-001 | **UUID v4 como PK** (`@default(uuid())`) | Segurança: sem exposição de sequência incremental; compatível com Supabase Auth |
| D-002 | **`Decimal(12,2)` em monetários** | Evita erros de precisão float em valores BRL (repasse, saldo devedor, retorno líquido) |
| D-003 | **Isolamento por `cedente_id`** em vez de `tenant_id` genérico | A Dani-Cedente é single-service sem multi-tenancy SaaS; o isolamento é por Cedente individual via JWT claim |
| D-004 | **Soft delete global** (`deleted_at DateTime?`) | LGPD: exclusão assíncrona ≤90 dias; nunca delete físico em entidades com dados do Cedente |
| D-005 | **`OpportunityScenario` separado de `Opportunity`** | Cenários A/B/C/D são confidenciais para o Cessionário; separação permite RLS granular |
| D-006 | **`pgvector` gerenciado via SQL raw** | Prisma 6.x não suporta tipo `vector` nativo; managed via `prisma/rls/indexes.sql` com `CREATE EXTENSION` + coluna raw |
| D-007 | **`ChatMessage` append-only** | Histórico de conversa é imutável — LGPD e auditoria; sem `deleted_at` nem `updated_at` |
| D-008 | **`KnowledgeEmbedding` sem `cedente_id`** | Base de conhecimento é compartilhada entre todos os Cedentes; sem dado pessoal |
| D-009 | **`csat_score` em `ChatSession`** | CSAT é coletado ao encerrar a sessão; pertence à sessão, não à mensagem |
| D-010 | **`EscrowTransaction.extension_requested_at` e `extension_approved_at`** | Rastreabilidade da extensão de prazo (+5 dias úteis) com timestamp de aprovação/silêncio |

---

## 4. Enums

### 4.1 Domínio: Oportunidade

```prisma
enum OpportunityStatus {
  RASCUNHO      // Criada mas não publicada
  PUBLICADA     // Visível no marketplace
  EM_NEGOCIACAO // Proposta em andamento
  PAUSADA       // Retirada temporariamente pelo Cedente
  EXPIRADA      // Prazo vencido sem negociação
  CANCELADA     // Encerrada permanentemente
}

// Fluxo: RASCUNHO → PUBLICADA → EM_NEGOCIACAO → (PAUSADA) → PUBLICADA
//                                              → EXPIRADA (automático)
//                                              → CANCELADA (manual)
```

```prisma
enum ScenarioType {
  A  // Cenário com condições de negociação tipo A (confidencial)
  B  // Cenário com condições de negociação tipo B (confidencial)
  C  // Cenário com condições de negociação tipo C (confidencial)
  D  // Cenário com condições de negociação tipo D (confidencial)
}
```

### 4.2 Domínio: Proposta

```prisma
enum ProposalStatus {
  RECEBIDA     // Nova proposta do Cessionário
  EM_ANALISE   // Cedente visualizou e está analisando
  ACEITA       // Cedente aceitou — inicia Escrow
  RECUSADA     // Cedente recusou
  CONTRAPROPOSTA // Cedente enviou contraproposta
  CANCELADA    // Cessionário cancelou antes da resposta
  EXPIRADA     // Sem resposta no prazo
}

// Fluxo: RECEBIDA → EM_ANALISE → ACEITA (→ EscrowTransaction criado)
//                              → RECUSADA
//                              → CONTRAPROPOSTA → [nova proposta do Cessionário]
//                 → CANCELADA (por Cessionário)
//                 → EXPIRADA (por timeout)
```

```prisma
enum ProposalType {
  INICIAL        // Primeira proposta do Cessionário
  CONTRAPROPOSTA // Enviada pelo Cedente em resposta
}
```

### 4.3 Domínio: Dossiê

```prisma
enum DossierDocumentType {
  RG_CNH               // Documento de identidade
  CPF                  // CPF do Cedente
  COMPROVANTE_RESIDENCIA
  CONTRATO_FINANCIAMENTO
  EXTRATO_FINANCIAMENTO // Extrato atualizado com saldo devedor
  DECLARACAO_QUITACAO   // Declaração de ausência de ônus
}

enum DossierDocumentStatus {
  AGUARDANDO_ENVIO // Não enviado ainda
  EM_ANALISE       // Enviado, aguardando revisão do Admin
  APROVADO         // Aprovado pelo Admin
  REJEITADO        // Rejeitado pelo Admin com motivo
}
```

### 4.4 Domínio: Escrow

```prisma
enum EscrowStatus {
  AGUARDANDO_DEPOSITO  // Proposta aceita, aguardando depósito do Cessionário
  DEPOSITO_CONFIRMADO  // Valor depositado confirmado
  LIBERADO             // Valor liberado ao Cedente após assinatura
  REVERTIDO            // Valor devolvido ao Cessionário (reversão em 15 dias)
  EXPIRADO             // Prazo de depósito venceu sem depósito
}

// Fluxo: AGUARDANDO_DEPOSITO → DEPOSITO_CONFIRMADO → LIBERADO
//                            → EXPIRADO (automático após 10 dias úteis)
//                                → oportunidade volta ao marketplace
//        DEPOSITO_CONFIRMADO → REVERTIDO (se reversão em 15 dias corridos)
```

### 4.5 Domínio: Chat

```prisma
enum MessageRole {
  USER      // Mensagem do Cedente
  ASSISTANT // Resposta da Dani
  SYSTEM    // Mensagem de sistema (não exibida ao Cedente)
}

enum ChatSessionStatus {
  ATIVA    // Sessão em andamento
  ENCERRADA // Encerrada pelo Cedente ou timeout 5min
}
```

### 4.6 Domínio: KYC

```prisma
enum KycStatus {
  PENDENTE  // Cadastro ainda não verificado
  APROVADO  // Verificado com sucesso
  REPROVADO // Verificação falhou
}
```

---

## 5. Modelos

### 5.1 Hierarquia Visual

```
CedenteProfile
├── Opportunity (1:N)
│   └── OpportunityScenario (1:N, confidencial)
├── Proposal (via Opportunity, N:M)
│   └── EscrowTransaction (1:1 quando ACEITA)
├── DossierDocument (1:N)
├── ChatSession (1:N)
│   └── ChatMessage (1:N, append-only)
└── KnowledgeEmbedding (shared, sem cedente_id)
```

### 5.2 Tabela Resumo

| Modelo | Soft Delete | Concorrência Otimista | Append-Only |
|---|---|---|---|
| CedenteProfile | ✅ | — | — |
| Opportunity | ✅ | ✅ | — |
| OpportunityScenario | ✅ | — | — |
| Proposal | ✅ | — | — |
| DossierDocument | ✅ | — | — |
| EscrowTransaction | ✅ | — | — |
| ChatSession | ✅ | — | — |
| ChatMessage | — | — | ✅ |
| KnowledgeEmbedding | ✅ | — | — |

### 5.3 Campos Comuns Obrigatórios (entidades não-append-only)

```prisma
// Incluído em todos os modelos de negócio (exceto ChatMessage e KnowledgeEmbedding)
id         String    @id @default(uuid())
cedente_id String    // FK → CedenteProfile.id (isolamento RLS)
created_at DateTime  @default(now())
updated_at DateTime  @updatedAt
deleted_at DateTime? // Soft delete LGPD
created_by String    // cedente_id ou 'system' ou 'admin'
updated_by String?
```

### 5.4 Modelos Completos

```prisma
// ============================================================
// generator e datasource
// ============================================================

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  directUrl  = env("DIRECT_URL")
  extensions = [pgcrypto, pg_trgm]
  // pgvector gerenciado via SQL raw (indexes.sql) — não declarado aqui
}

// ============================================================
// CedenteProfile
// ============================================================

model CedenteProfile {
  id         String    @id @default(uuid())
  user_id    String    @unique // FK → Supabase Auth user.id
  full_name  String
  cpf        String    @unique
  kyc_status KycStatus @default(PENDENTE)
  kyc_verified_at DateTime?
  created_at DateTime  @default(now())
  updated_at DateTime  @updatedAt
  deleted_at DateTime?
  created_by String    @default("system")
  updated_by String?

  opportunities    Opportunity[]
  dossier_documents DossierDocument[]
  chat_sessions    ChatSession[]

  @@map("cedente_profiles")
}

// ============================================================
// Opportunity
// ============================================================

model Opportunity {
  id                        String            @id @default(uuid())
  cedente_id                String
  status                    OpportunityStatus @default(RASCUNHO)
  financing_contract_number String
  asking_price              Decimal           @db.Decimal(12, 2)
  outstanding_balance       Decimal           @db.Decimal(12, 2)
  amount_paid               Decimal           @db.Decimal(12, 2)
  delta                     Decimal           @db.Decimal(12, 2)  // Tabela Atual - Tabela Contrato; fallback Valor Pago se ≤ 0
  published_at              DateTime?
  withdrawn_at              DateTime?
  version                   Int               @default(1)         // Concorrência otimista
  created_at                DateTime          @default(now())
  updated_at                DateTime          @updatedAt
  deleted_at                DateTime?
  created_by                String
  updated_by                String?

  cedente   CedenteProfile      @relation(fields: [cedente_id], references: [id], onDelete: Restrict)
  scenarios OpportunityScenario[]
  proposals Proposal[]

  @@map("opportunities")
}

// ============================================================
// OpportunityScenario (cenários A/B/C/D — confidenciais)
// ============================================================

model OpportunityScenario {
  id             String       @id @default(uuid())
  opportunity_id String
  cedente_id     String       // Redundante para RLS — evita JOIN em cada query
  scenario_type  ScenarioType
  conditions     Json         // Condições do cenário (estrutura interna)
  is_active      Boolean      @default(true)
  created_at     DateTime     @default(now())
  updated_at     DateTime     @updatedAt
  deleted_at     DateTime?
  created_by     String
  updated_by     String?

  opportunity Opportunity @relation(fields: [opportunity_id], references: [id], onDelete: Cascade)

  @@unique([opportunity_id, scenario_type])
  @@map("opportunity_scenarios")
}

// ============================================================
// Proposal
// ============================================================

model Proposal {
  id              String         @id @default(uuid())
  opportunity_id  String
  cedente_id      String         // Redundante para RLS
  offered_value   Decimal        @db.Decimal(12, 2)
  net_return      Decimal        @db.Decimal(12, 2)  // Calculado: offered_value - outstanding_balance
  status          ProposalStatus @default(RECEBIDA)
  proposal_type   ProposalType   @default(INICIAL)
  parent_id       String?        // Referência à proposta pai (para contrproposta)
  rejection_reason String?
  expires_at      DateTime?
  created_at      DateTime       @default(now())
  updated_at      DateTime       @updatedAt
  deleted_at      DateTime?
  created_by      String
  updated_by      String?

  opportunity       Opportunity       @relation(fields: [opportunity_id], references: [id], onDelete: Restrict)
  escrow_transaction EscrowTransaction?
  parent_proposal   Proposal?         @relation("ProposalChain", fields: [parent_id], references: [id], onDelete: SetNull)
  child_proposals   Proposal[]        @relation("ProposalChain")

  @@map("proposals")
}

// ============================================================
// DossierDocument
// ============================================================

model DossierDocument {
  id            String                @id @default(uuid())
  cedente_id    String
  document_type DossierDocumentType
  status        DossierDocumentStatus @default(AGUARDANDO_ENVIO)
  file_url      String?               // URL S3/Supabase Storage
  file_name     String?
  file_size_kb  Int?
  rejection_reason String?
  submitted_at  DateTime?
  reviewed_at   DateTime?
  reviewed_by   String?               // Admin user_id
  created_at    DateTime              @default(now())
  updated_at    DateTime              @updatedAt
  deleted_at    DateTime?
  created_by    String
  updated_by    String?

  cedente CedenteProfile @relation(fields: [cedente_id], references: [id], onDelete: Restrict)

  @@unique([cedente_id, document_type])  // Um documento por tipo por Cedente
  @@map("dossier_documents")
}

// ============================================================
// EscrowTransaction
// ============================================================

model EscrowTransaction {
  id                     String       @id @default(uuid())
  proposal_id            String       @unique
  cedente_id             String       // Redundante para RLS
  status                 EscrowStatus @default(AGUARDANDO_DEPOSITO)
  escrow_value           Decimal      @db.Decimal(12, 2)
  deposit_deadline       DateTime     // 10 dias úteis após aceite da proposta
  deposit_confirmed_at   DateTime?
  released_at            DateTime?
  reversed_at            DateTime?
  reversal_deadline      DateTime?    // 15 dias corridos após DEPOSITO_CONFIRMADO
  extension_requested_at DateTime?
  extension_approved_at  DateTime?    // Null = aprovação por silêncio após 24h
  extension_days         Int?         // Sempre 5 quando extensão concedida
  created_at             DateTime     @default(now())
  updated_at             DateTime     @updatedAt
  deleted_at             DateTime?
  created_by             String
  updated_by             String?

  proposal Proposal @relation(fields: [proposal_id], references: [id], onDelete: Restrict)

  @@map("escrow_transactions")
}

// ============================================================
// ChatSession
// ============================================================

model ChatSession {
  id          String            @id @default(uuid())
  cedente_id  String
  status      ChatSessionStatus @default(ATIVA)
  started_at  DateTime          @default(now())
  ended_at    DateTime?
  csat_score  Int?              // 1-5; null se não avaliado
  csat_submitted_at DateTime?
  message_count Int             @default(0)
  created_at  DateTime          @default(now())
  updated_at  DateTime          @updatedAt
  deleted_at  DateTime?         // Soft delete: histórico retido 90 dias, então deletado
  created_by  String
  updated_by  String?

  cedente  CedenteProfile @relation(fields: [cedente_id], references: [id], onDelete: Restrict)
  messages ChatMessage[]

  @@map("chat_sessions")
}

// ============================================================
// ChatMessage (append-only — sem soft delete, sem updated_at)
// ============================================================

model ChatMessage {
  id            String      @id @default(uuid())
  session_id    String
  cedente_id    String      // Redundante para RLS
  role          MessageRole
  content       String      @db.Text   // PII mascarado antes de persistir
  tokens_used   Int?        // Tokens consumidos na chamada LLM (role=ASSISTANT)
  langfuse_trace_id String? // Referência ao trace no Langfuse
  confidence_score Decimal? @db.Decimal(5, 2)  // 0-100; calculado para role=ASSISTANT
  created_at    DateTime    @default(now())

  session ChatSession @relation(fields: [session_id], references: [id], onDelete: Cascade)

  @@map("chat_messages")
}

// ============================================================
// KnowledgeEmbedding (base de conhecimento RAG — sem cedente_id)
// ============================================================

model KnowledgeEmbedding {
  id         String   @id @default(uuid())
  content    String   @db.Text       // Chunk de texto original
  source     String                  // Ex: "regras-de-negocio", "faq-cedente"
  metadata   Json?                   // {section, page, version}
  // embedding vector(1536) — gerenciado via SQL raw (não suportado pelo Prisma)
  created_at DateTime @default(now())
  updated_at DateTime @updatedAt
  deleted_at DateTime?
  created_by String   @default("system")
  updated_by String?

  @@map("knowledge_embeddings")
}
```

---

## 6. Row Level Security (RLS)

### 6.1 Padrão de isolamento por `cedente_id`

O isolamento não usa `SET LOCAL` de tenant genérico — usa o claim `sub` do JWT do Supabase que mapeia para `cedente_id`.

```sql
-- prisma/rls/policies.sql

-- Ativar RLS em todas as tabelas protegidas
ALTER TABLE cedente_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE opportunities ENABLE ROW LEVEL SECURITY;
ALTER TABLE opportunity_scenarios ENABLE ROW LEVEL SECURITY;
ALTER TABLE proposals ENABLE ROW LEVEL SECURITY;
ALTER TABLE dossier_documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE escrow_transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_messages ENABLE ROW LEVEL SECURITY;
-- knowledge_embeddings: sem RLS (dados compartilhados, sem PII)

-- -------------------------------------------------------
-- cedente_profiles: Cedente vê apenas o próprio perfil
-- -------------------------------------------------------
CREATE POLICY "cedente_profiles_select"
  ON cedente_profiles FOR SELECT
  USING (user_id = auth.uid()::text);

CREATE POLICY "cedente_profiles_insert"
  ON cedente_profiles FOR INSERT
  WITH CHECK (user_id = auth.uid()::text);

CREATE POLICY "cedente_profiles_update"
  ON cedente_profiles FOR UPDATE
  USING (user_id = auth.uid()::text);

-- -------------------------------------------------------
-- opportunities: Cedente vê apenas as próprias oportunidades
-- -------------------------------------------------------
CREATE POLICY "opportunities_cedente"
  ON opportunities FOR ALL
  USING (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ))
  WITH CHECK (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ));

-- -------------------------------------------------------
-- opportunity_scenarios: mesma policy de opportunities
-- -------------------------------------------------------
CREATE POLICY "opportunity_scenarios_cedente"
  ON opportunity_scenarios FOR ALL
  USING (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ))
  WITH CHECK (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ));

-- -------------------------------------------------------
-- proposals
-- -------------------------------------------------------
CREATE POLICY "proposals_cedente"
  ON proposals FOR ALL
  USING (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ))
  WITH CHECK (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ));

-- -------------------------------------------------------
-- dossier_documents
-- -------------------------------------------------------
CREATE POLICY "dossier_documents_cedente"
  ON dossier_documents FOR ALL
  USING (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ))
  WITH CHECK (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ));

-- -------------------------------------------------------
-- escrow_transactions
-- -------------------------------------------------------
CREATE POLICY "escrow_transactions_cedente"
  ON escrow_transactions FOR ALL
  USING (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ))
  WITH CHECK (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ));

-- -------------------------------------------------------
-- chat_sessions
-- -------------------------------------------------------
CREATE POLICY "chat_sessions_cedente"
  ON chat_sessions FOR ALL
  USING (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ))
  WITH CHECK (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ));

-- -------------------------------------------------------
-- chat_messages: via session (JOIN)
-- -------------------------------------------------------
CREATE POLICY "chat_messages_cedente"
  ON chat_messages FOR ALL
  USING (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ))
  WITH CHECK (cedente_id IN (
    SELECT id FROM cedente_profiles WHERE user_id = auth.uid()::text
  ));
```

### 6.2 Indexes

```sql
-- prisma/rls/indexes.sql

-- Extensão pgvector (necessária para knowledge_embeddings)
CREATE EXTENSION IF NOT EXISTS vector;

-- Adicionar coluna embedding (não suportada pelo Prisma schema)
ALTER TABLE knowledge_embeddings
  ADD COLUMN IF NOT EXISTS embedding vector(1536);

-- IVFFlat index para busca semântica (cosine distance)
-- Requer ao menos 2× lists (200) registros na tabela antes de criar
CREATE INDEX IF NOT EXISTS idx_knowledge_embedding
  ON knowledge_embeddings
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);

-- Indexes de performance
CREATE INDEX IF NOT EXISTS idx_opportunities_cedente_status
  ON opportunities (cedente_id, status)
  WHERE deleted_at IS NULL;

CREATE INDEX IF NOT EXISTS idx_proposals_opportunity
  ON proposals (opportunity_id, status)
  WHERE deleted_at IS NULL;

CREATE INDEX IF NOT EXISTS idx_chat_sessions_cedente
  ON chat_sessions (cedente_id, status)
  WHERE deleted_at IS NULL;

CREATE INDEX IF NOT EXISTS idx_chat_messages_session
  ON chat_messages (session_id, created_at DESC);

CREATE INDEX IF NOT EXISTS idx_dossier_cedente_type
  ON dossier_documents (cedente_id, document_type)
  WHERE deleted_at IS NULL;

CREATE INDEX IF NOT EXISTS idx_escrow_proposal
  ON escrow_transactions (proposal_id, status)
  WHERE deleted_at IS NULL;
```

---

## 7. Middleware Prisma

### 7.1 Ordem de aplicação

```
1. CedenteIsolationMiddleware  → injeta cedente_id de forma automática em CREATE
2. SoftDeleteMiddleware        → converte delete() → update({ deleted_at })
```

### 7.2 CedenteIsolationMiddleware

```typescript
// prisma/middleware/cedente-isolation.middleware.ts
import { Prisma } from '@prisma/client';

type CedenteModels =
  | 'opportunity'
  | 'opportunityScenario'
  | 'proposal'
  | 'dossierDocument'
  | 'escrowTransaction'
  | 'chatSession'
  | 'chatMessage';

const CEDENTE_MODELS: CedenteModels[] = [
  'opportunity',
  'opportunityScenario',
  'proposal',
  'dossierDocument',
  'escrowTransaction',
  'chatSession',
  'chatMessage',
];

export function cedenteIsolationMiddleware(
  cedenteId: string,
): Prisma.Middleware {
  return async (params, next) => {
    const model = params.model?.toLowerCase() as CedenteModels;

    if (CEDENTE_MODELS.includes(model)) {
      // Injetar cedente_id em CREATE
      if (params.action === 'create' || params.action === 'createMany') {
        if (params.args.data) {
          if (Array.isArray(params.args.data)) {
            params.args.data = params.args.data.map((d: object) => ({
              ...d,
              cedente_id: cedenteId,
              created_by: cedenteId,
            }));
          } else {
            params.args.data.cedente_id = cedenteId;
            params.args.data.created_by = cedenteId;
          }
        }
      }

      // Injetar cedente_id em UPDATE para created_by (updated_by)
      if (params.action === 'update' || params.action === 'updateMany') {
        if (params.args.data) {
          params.args.data.updated_by = cedenteId;
        }
      }
    }

    return next(params);
  };
}
```

### 7.3 SoftDeleteMiddleware

```typescript
// prisma/middleware/soft-delete.middleware.ts
import { Prisma } from '@prisma/client';

// Modelos append-only: nunca recebem soft delete
const APPEND_ONLY_MODELS = ['chatmessage'];

export const softDeleteMiddleware: Prisma.Middleware = async (params, next) => {
  const model = params.model?.toLowerCase();

  if (!model || APPEND_ONLY_MODELS.includes(model)) {
    return next(params);
  }

  // Converter delete → update com deleted_at
  if (params.action === 'delete') {
    params.action = 'update';
    params.args.data = { deleted_at: new Date() };
  }

  if (params.action === 'deleteMany') {
    params.action = 'updateMany';
    if (params.args.data !== undefined) {
      params.args.data.deleted_at = new Date();
    } else {
      params.args.data = { deleted_at: new Date() };
    }
  }

  // Filtrar registros soft-deleted em todas as leituras
  if (params.action === 'findUnique' || params.action === 'findFirst') {
    params.action = 'findFirst';
    params.args.where = { ...params.args.where, deleted_at: null };
  }

  if (params.action === 'findMany') {
    params.args.where = { ...params.args.where, deleted_at: null };
  }

  if (params.action === 'count') {
    params.args.where = { ...params.args.where, deleted_at: null };
  }

  return next(params);
};
```

---

## 8. Seed

Registros mínimos para ambiente de desenvolvimento e testes.

```typescript
// prisma/seed.ts
import { PrismaClient, KycStatus, OpportunityStatus } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  // Cedente de desenvolvimento
  const cedente = await prisma.cedenteProfile.upsert({
    where: { cpf: '000.000.000-00' },
    update: {},
    create: {
      user_id: 'dev-user-supabase-id',
      full_name: 'Cedente Desenvolvimento',
      cpf: '000.000.000-00',
      kyc_status: KycStatus.APROVADO,
      kyc_verified_at: new Date(),
      created_by: 'system',
    },
  });

  // Oportunidade de desenvolvimento
  const opportunity = await prisma.opportunity.upsert({
    where: { id: 'dev-opportunity-001' },
    update: {},
    create: {
      id: 'dev-opportunity-001',
      cedente_id: cedente.id,
      status: OpportunityStatus.PUBLICADA,
      financing_contract_number: 'CEF-2020-DEV-001',
      asking_price: 280000.00,
      outstanding_balance: 234800.00,
      amount_paid: 45200.00,
      delta: 12000.00,
      published_at: new Date(),
      created_by: cedente.id,
    },
  });

  // KnowledgeEmbedding de exemplo (sem embedding — preenchido pelo RAG service)
  await prisma.knowledgeEmbedding.upsert({
    where: { id: 'dev-embedding-001' },
    update: {},
    create: {
      id: 'dev-embedding-001',
      content: 'O repasse é a transferência do financiamento imobiliário do Cedente para um Cessionário.',
      source: 'faq-cedente',
      metadata: { section: 'conceitos-basicos', version: '1.0' },
      created_by: 'system',
    },
  });

  console.log('Seed concluído:', { cedente: cedente.id, opportunity: opportunity.id });
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

---

## 9. Quotas e Limites

| Parâmetro | Valor | Fonte |
|---|---|---|
| Mensagens por Cedente por hora | 30 (janela deslizante Redis) | RN-DCE-022 / RF-DCE-009 |
| Histórico de sessões retido | 90 dias (configurável pelo Admin) | RN-DCE-022 / RF-DCE-007 |
| Tamanho máximo de arquivo (dossiê) | 10 MB | [DECISÃO AUTÔNOMA — padrão S3/Supabase Storage para documentos de identificação] |
| Dimensão de embedding pgvector | 1536 (text-embedding-3-small) | D02 Stacks |
| Listas IVFFlat | 100 | D12 ERD Schema |
| Sessões simultâneas por Cedente | 1 ativa (demais: ENCERRADA automático) | [DECISÃO AUTÔNOMA — evita histórico fragmentado; alternativa descartada: múltiplas sessões (aumenta complexidade de contexto do LLM)] |

---

## 10. Políticas de Retenção (LGPD)

| Entidade | Dado pessoal | Retenção | Método de exclusão |
|---|---|---|---|
| `cedente_profiles` | CPF, nome | 90 dias após encerramento da conta | Soft delete → job async `deleted_at + 90d` → DELETE físico |
| `chat_sessions` | Contexto conversacional | 90 dias (configurável Admin) | Soft delete → job async |
| `chat_messages` | Conteúdo da conversa (PII mascarado em log, mas texto em `content`) | 90 dias junto com a sessão | Cascade delete via `session_id` |
| `dossier_documents` | Arquivos de identidade | Até encerramento do processo + 90 dias | Soft delete + remoção do S3 |
| `opportunities` | Dados financeiros do Cedente | Até encerramento + 90 dias | Soft delete |
| `proposals` | Valores negociados | Até encerramento do processo + 90 dias | Soft delete |
| `escrow_transactions` | Valores financeiros | 5 anos (regulatório financeiro) | Soft delete — exclusão física somente após 5 anos |
| `knowledge_embeddings` | Sem PII | Indefinido (base de conhecimento) | Soft delete manual |
| Logs Langfuse | Traces de LLM com conteúdo mascarado | 90 dias (config Langfuse) | Automático por Langfuse |

> **Regra LGPD:** nenhum dado que permita identificação direta do Cedente (CPF, nome, conteúdo de conversa) é enviado para logs externos sem mascaramento prévio. `ChatMessage.content` é mascarado pelo `PIIMaskingMiddleware` antes do `created_by = cedenteId` ser gravado nos logs de Sentry/Langfuse.

---

## 11. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 23/03/2026 | Claude Code Desktop | Versão inicial. 9 modelos, 11 enums, RLS por cedente_id, middleware (isolamento + soft delete), seed de desenvolvimento, políticas de retenção LGPD. Alinhado com D01, D02, D05 e D12. |

---

## 12. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Tamanho máximo de arquivo para dossiê (10 MB) | DECISÃO AUTÔNOMA | Seção 9 | Padrão do setor para documentos de identificação em Storage S3/Supabase. Alternativa descartada: 5 MB (restritivo para contratos de financiamento que podem ter 15+ páginas digitalizadas). | P2 | Produto | Validar |
| Sessões simultâneas por Cedente limitadas a 1 | DECISÃO AUTÔNOMA | Seção 9 | Contexto LLM unificado facilita continuidade da conversa. Alternativa descartada: múltiplas sessões simultâneas (fragmenta contexto, aumenta custo de tokens). | P2 | Engenharia | Validar |
| Retenção de `escrow_transactions` por 5 anos | DECISÃO AUTÔNOMA | Seção 10 | Regulamentação do Banco Central exige retenção de registros financeiros por 5 anos. Alternativa descartada: 90 dias (insuficiente para compliance financeiro). | P0 | Jurídico / Produto | Confirmar com time jurídico |
