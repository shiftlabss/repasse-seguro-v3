# 13 - Schema Prisma

## Módulo Cessionário · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia Backend |
| **Escopo** | Schema Prisma completo · Enums · Models · Relações · Índices |
| **Módulo** | Cessionário |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - Schema Prisma completo para o módulo Cessionário, derivado do ERD (D12).
> - Stack: Prisma 6.x + PostgreSQL 17 (via Supabase). Datasource: `postgresql`.
> - Convenções: UUID v4, snake_case para tabelas e colunas, PascalCase para enums, UPPER_SNAKE_CASE para valores de enum.
> - Soft delete via `deleted_at DateTime? @db.Timestamptz` em todas as tabelas de domínio.
> - `@@map` para nomear tabelas em snake_case plural no banco.
> - Extensões necessárias: `pgcrypto` (UUID), `pgvector` (embeddings do Analista de Oportunidades).

---

## 1. Schema Prisma Completo

```prisma
// schema.prisma
// Módulo Cessionário — Repasse Seguro
// Prisma 6.x + PostgreSQL 17 via Supabase

generator client {
  provider = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  directUrl  = env("DIRECT_URL")
  extensions = [pgcrypto, pgvector(map: "vector")]
}

// ============================================================
// ENUMS
// ============================================================

enum CessionarioStatus {
  CADASTRADA
  KYC_EM_ANALISE
  KYC_APROVADO
  KYC_REPROVADO
  BLOQUEADA_TEMPORARIAMENTE
  ENCERRADA
}

enum KycStatus {
  PENDENTE
  EM_ANALISE
  APROVADO
  REPROVADO
}

enum OpportunityStatus {
  DISPONIVEL
  COM_PROPOSTA
  EM_NEGOCIACAO
  RESERVADA
  CONCLUIDA
  CANCELADA
}

enum ProposalStatus {
  ENVIADA
  EM_ANALISE
  ACEITA
  RECUSADA
  EXPIRADA
  CANCELADA
}

enum NegotiationStatus {
  EM_NEGOCIACAO
  EM_CONTRAPROPOSTA
  AGUARDANDO_DEPOSITO
  DEPOSITO_CONFIRMADO
  ENCERRADA
  CANCELADA
}

enum EscrowStatus {
  AGUARDANDO_DEPOSITO
  DEPOSITO_ENVIADO
  DEPOSITO_CONFIRMADO
  REEMBOLSADO
}

enum FormalizationStatus {
  DOCUMENTOS_DISPONIVEIS
  ASSINATURA_PENDENTE_CESSIONARIO
  ASSINATURA_PENDENTE_CEDENTE
  AGUARDANDO_ANUENCIA
  CONCLUIDA
  CANCELADA
}

enum TransactionType {
  ESCROW_DEPOSIT
  COMMISSION
  REFUND
  OPERATION_COMPLETED
}

enum TransactionStatus {
  PENDENTE
  PROCESSADO
  FALHOU
}

enum AuthProvider {
  EMAIL
  GOOGLE
}

enum ActorType {
  CESSIONARIO
  ADMIN
  SYSTEM
}

// ============================================================
// MODELS
// ============================================================

model User {
  id                String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  email             String    @unique @db.VarChar(255)
  name              String    @db.VarChar(255)
  provider          AuthProvider @default(EMAIL)
  emailVerifiedAt   DateTime? @map("email_verified_at") @db.Timestamptz
  createdAt         DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt         DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt         DateTime? @map("deleted_at") @db.Timestamptz

  // Relações
  cessionario       Cessionario?

  @@map("users")
}

model Cessionario {
  id                      String             @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  userId                  String             @unique @map("user_id") @db.Uuid
  status                  CessionarioStatus  @default(CADASTRADA)
  phone                   String?            @db.VarChar(20)
  phoneVerifiedAt         DateTime?          @map("phone_verified_at") @db.Timestamptz
  bankAccount             Json?              @map("bank_account") @db.JsonB
  bankAccountVerifiedAt   DateTime?          @map("bank_account_verified_at") @db.Timestamptz
  notificationPreferences Json               @default("{\"email\": true, \"push\": true, \"sms\": true}") @map("notification_preferences") @db.JsonB
  aiConsent               Boolean            @default(true) @map("ai_consent")
  aiConsentAt             DateTime?          @map("ai_consent_at") @db.Timestamptz
  investmentPreferences   Json?              @map("investment_preferences") @db.JsonB
  createdAt               DateTime           @default(now()) @map("created_at") @db.Timestamptz
  updatedAt               DateTime           @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  user                    User               @relation(fields: [userId], references: [id], onDelete: Restrict)
  kycDocuments            KycDocument[]
  proposals               Proposal[]
  negotiations            Negotiation[]
  escrowDeposits          EscrowDeposit[]
  formalizations          Formalization[]
  financialTransactions   FinancialTransaction[]
  notifications           Notification[]
  aiSessions              AiSession[]

  @@index([status], name: "idx_cessionarios_status")
  @@map("cessionarios")
}

model KycDocument {
  id                    String       @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  cessionarioId         String       @map("cessionario_id") @db.Uuid
  status                KycStatus    @default(PENDENTE)
  identityDocFrontUrl   String?      @map("identity_doc_front_url") @db.VarChar(500)
  identityDocBackUrl    String?      @map("identity_doc_back_url") @db.VarChar(500)
  addressDocUrl         String?      @map("address_doc_url") @db.VarChar(500)
  selfieUrl             String?      @map("selfie_url") @db.VarChar(500)
  identityStatus        KycStatus    @default(PENDENTE) @map("identity_status")
  addressStatus         KycStatus    @default(PENDENTE) @map("address_status")
  selfieStatus          KycStatus    @default(PENDENTE) @map("selfie_status")
  rejectionReason       String?      @map("rejection_reason") @db.Text
  idwallSessionId       String?      @map("idwall_session_id") @db.VarChar(255)
  attemptCount          Int          @default(0) @map("attempt_count")
  blockedUntil          DateTime?    @map("blocked_until") @db.Timestamptz
  reviewerId            String?      @map("reviewer_id") @db.Uuid
  reviewedAt            DateTime?    @map("reviewed_at") @db.Timestamptz
  createdAt             DateTime     @default(now()) @map("created_at") @db.Timestamptz
  updatedAt             DateTime     @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  cessionario           Cessionario  @relation(fields: [cessionarioId], references: [id], onDelete: Restrict)

  @@index([cessionarioId], name: "idx_kyc_documents_cessionario_id")
  @@map("kyc_documents")
}

model Opportunity {
  id                    String            @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  code                  String            @unique @db.VarChar(20)
  status                OpportunityStatus @default(DISPONIVEL)
  city                  String            @db.VarChar(100)
  neighborhood          String            @db.VarChar(100)
  state                 String            @db.VarChar(2)
  developmentName       String            @map("development_name") @db.VarChar(200)
  bedrooms              Int
  areaSqm               Decimal           @map("area_sqm") @db.Decimal(8, 2)
  hasGarage             Boolean           @map("has_garage")
  currentTableValue     Decimal           @map("current_table_value") @db.Decimal(15, 2)
  contractTableValue    Decimal           @map("contract_table_value") @db.Decimal(15, 2)
  cededentePaidPercentage Decimal         @map("cedente_paid_percentage") @db.Decimal(5, 2)
  cedentePaidValue      Decimal           @map("cedente_paid_value") @db.Decimal(15, 2)
  aiRiskScore           Decimal?          @map("ai_risk_score") @db.Decimal(4, 2)
  appreciationData      Json?             @map("appreciation_data") @db.JsonB
  publishedAt           DateTime?         @map("published_at") @db.Timestamptz
  createdAt             DateTime          @default(now()) @map("created_at") @db.Timestamptz
  updatedAt             DateTime          @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt             DateTime?         @map("deleted_at") @db.Timestamptz

  // Nota: delta_value é calculado como current_table_value - contract_table_value
  // Implementado como propriedade virtual ou coluna gerada no banco via migration

  // Relações
  proposals             Proposal[]
  negotiations          Negotiation[]

  @@index([status], name: "idx_opportunities_status")
  @@index([city, state], name: "idx_opportunities_city_state")
  // idx_opportunities_embedding criado via migration SQL (HNSW pgvector)
  @@map("opportunities")
}

model Proposal {
  id              String         @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  cessionarioId   String         @map("cessionario_id") @db.Uuid
  opportunityId   String         @map("opportunity_id") @db.Uuid
  proposedValue   Decimal        @map("proposed_value") @db.Decimal(15, 2)
  commissionBuyer Decimal        @map("commission_buyer") @db.Decimal(15, 2)
  message         String?        @db.Text
  status          ProposalStatus @default(ENVIADA)
  rejectionReason String?        @map("rejection_reason") @db.Text
  expiresAt       DateTime       @map("expires_at") @db.Timestamptz
  createdAt       DateTime       @default(now()) @map("created_at") @db.Timestamptz
  updatedAt       DateTime       @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt       DateTime?      @map("deleted_at") @db.Timestamptz

  // Relações
  cessionario     Cessionario    @relation(fields: [cessionarioId], references: [id], onDelete: Restrict)
  opportunity     Opportunity    @relation(fields: [opportunityId], references: [id], onDelete: Restrict)
  negotiation     Negotiation?

  @@index([cessionarioId], name: "idx_proposals_cessionario_id")
  @@index([opportunityId], name: "idx_proposals_opportunity_id")
  @@index([status], name: "idx_proposals_status")
  @@index([cessionarioId, status], name: "idx_proposals_cessionario_status")
  @@map("proposals")
}

model Negotiation {
  id                  String            @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  proposalId          String            @unique @map("proposal_id") @db.Uuid
  cessionarioId       String            @map("cessionario_id") @db.Uuid
  opportunityId       String            @map("opportunity_id") @db.Uuid
  agreedValue         Decimal?          @map("agreed_value") @db.Decimal(15, 2)
  commissionBuyer     Decimal?          @map("commission_buyer") @db.Decimal(15, 2)
  totalEscrowValue    Decimal?          @map("total_escrow_value") @db.Decimal(15, 2)
  status              NegotiationStatus @default(EM_NEGOCIACAO)
  counterofferRounds  Int               @default(0) @map("counteroffer_rounds")
  escrowDeadline      DateTime?         @map("escrow_deadline") @db.Timestamptz
  extensionUsed       Boolean           @default(false) @map("extension_used")
  extendedDeadline    DateTime?         @map("extended_deadline") @db.Timestamptz
  createdAt           DateTime          @default(now()) @map("created_at") @db.Timestamptz
  updatedAt           DateTime          @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  proposal            Proposal          @relation(fields: [proposalId], references: [id], onDelete: Restrict)
  cessionario         Cessionario       @relation(fields: [cessionarioId], references: [id], onDelete: Restrict)
  opportunity         Opportunity       @relation(fields: [opportunityId], references: [id], onDelete: Restrict)
  escrowDeposits      EscrowDeposit[]
  formalization       Formalization?
  financialTransactions FinancialTransaction[]
  chatMessages        ChatMessage[]

  @@index([cessionarioId], name: "idx_negotiations_cessionario_id")
  @@index([status], name: "idx_negotiations_status")
  @@map("negotiations")
}

model ChatMessage {
  id            String      @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  negotiationId String      @map("negotiation_id") @db.Uuid
  authorId      String      @map("author_id") @db.Uuid
  authorType    ActorType   @map("author_type")
  content       String      @db.Text
  attachmentUrl String?     @map("attachment_url") @db.VarChar(500)
  createdAt     DateTime    @default(now()) @map("created_at") @db.Timestamptz

  // Relações
  negotiation   Negotiation @relation(fields: [negotiationId], references: [id], onDelete: Cascade)

  @@index([negotiationId], name: "idx_chat_messages_negotiation_id")
  @@map("chat_messages")
}

model EscrowDeposit {
  id            String       @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  negotiationId String       @map("negotiation_id") @db.Uuid
  cessionarioId String       @map("cessionario_id") @db.Uuid
  amount        Decimal      @db.Decimal(15, 2)
  status        EscrowStatus @default(AGUARDANDO_DEPOSITO)
  receiptUrl    String?      @map("receipt_url") @db.VarChar(500)
  submittedAt   DateTime?    @map("submitted_at") @db.Timestamptz
  confirmedAt   DateTime?    @map("confirmed_at") @db.Timestamptz
  confirmedBy   String?      @map("confirmed_by") @db.Uuid
  createdAt     DateTime     @default(now()) @map("created_at") @db.Timestamptz
  updatedAt     DateTime     @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  negotiation   Negotiation  @relation(fields: [negotiationId], references: [id], onDelete: Restrict)
  cessionario   Cessionario  @relation(fields: [cessionarioId], references: [id], onDelete: Restrict)

  @@index([negotiationId], name: "idx_escrow_deposits_negotiation_id")
  @@map("escrow_deposits")
}

model Formalization {
  id                  String              @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  negotiationId       String              @unique @map("negotiation_id") @db.Uuid
  cessionarioId       String              @map("cessionario_id") @db.Uuid
  status              FormalizationStatus @default(DOCUMENTOS_DISPONIVEIS)
  zapsignSessionId    String?             @map("zapsign_session_id") @db.VarChar(255)
  zapsignBuyerSignUrl String?             @map("zapsign_buyer_sign_url") @db.VarChar(500)
  documentUrl         String?             @map("document_url") @db.VarChar(500)
  buyerSignedAt       DateTime?           @map("buyer_signed_at") @db.Timestamptz
  sellerSignedAt      DateTime?           @map("seller_signed_at") @db.Timestamptz
  anuenciaObtainedAt  DateTime?           @map("anuencia_obtained_at") @db.Timestamptz
  createdAt           DateTime            @default(now()) @map("created_at") @db.Timestamptz
  updatedAt           DateTime            @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  negotiation         Negotiation         @relation(fields: [negotiationId], references: [id], onDelete: Restrict)
  cessionario         Cessionario         @relation(fields: [cessionarioId], references: [id], onDelete: Restrict)

  @@index([cessionarioId], name: "idx_formalizations_cessionario_id")
  @@map("formalizations")
}

model FinancialTransaction {
  id            String            @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  cessionarioId String            @map("cessionario_id") @db.Uuid
  negotiationId String?           @map("negotiation_id") @db.Uuid
  type          TransactionType
  amount        Decimal           @db.Decimal(15, 2)
  status        TransactionStatus @default(PENDENTE)
  description   String?           @db.Text
  createdAt     DateTime          @default(now()) @map("created_at") @db.Timestamptz
  updatedAt     DateTime          @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  cessionario   Cessionario       @relation(fields: [cessionarioId], references: [id], onDelete: Restrict)
  negotiation   Negotiation?      @relation(fields: [negotiationId], references: [id], onDelete: SetNull)

  @@index([cessionarioId], name: "idx_financial_transactions_cessionario_id")
  @@map("financial_transactions")
}

model Notification {
  id            String      @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  cessionarioId String      @map("cessionario_id") @db.Uuid
  type          String      @db.VarChar(50)
  title         String      @db.VarChar(200)
  body          String      @db.Text
  metadata      Json?       @db.JsonB
  read          Boolean     @default(false)
  readAt        DateTime?   @map("read_at") @db.Timestamptz
  createdAt     DateTime    @default(now()) @map("created_at") @db.Timestamptz

  // Relações
  cessionario   Cessionario @relation(fields: [cessionarioId], references: [id], onDelete: Cascade)

  @@index([cessionarioId, read], name: "idx_notifications_cessionario_unread", where: "read = false" )
  @@map("notifications")
}

model AiSession {
  id            String      @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  cessionarioId String      @map("cessionario_id") @db.Uuid
  messages      Json        @default("[]") @db.JsonB
  context       Json?       @db.JsonB
  createdAt     DateTime    @default(now()) @map("created_at") @db.Timestamptz
  updatedAt     DateTime    @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  cessionario   Cessionario @relation(fields: [cessionarioId], references: [id], onDelete: Cascade)

  @@index([cessionarioId], name: "idx_ai_sessions_cessionario_id")
  @@map("ai_sessions")
}

model AuditLog {
  id          String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  actorId     String    @map("actor_id") @db.Uuid
  actorType   ActorType @map("actor_type")
  action      String    @db.VarChar(100)
  entityType  String    @map("entity_type") @db.VarChar(50)
  entityId    String?   @map("entity_id") @db.Uuid
  before      Json?     @db.JsonB
  after       Json?     @db.JsonB
  ipAddress   String?   @map("ip_address") @db.VarChar(45)
  createdAt   DateTime  @default(now()) @map("created_at") @db.Timestamptz
  // Sem updatedAt — audit logs são imutáveis

  @@index([actorId], name: "idx_audit_logs_actor_id")
  @@index([entityType, entityId], name: "idx_audit_logs_entity")
  @@map("audit_logs")
}
```

---

## 2. Migrations SQL Adicionais

As seguintes operações exigem SQL raw além do que o Prisma gera automaticamente:

### 2.1 Coluna Gerada delta_value

```sql
-- Migration: add_delta_value_generated_column
ALTER TABLE opportunities
ADD COLUMN delta_value DECIMAL(15,2)
GENERATED ALWAYS AS (current_table_value - contract_table_value) STORED;

CREATE INDEX idx_opportunities_delta_value ON opportunities (delta_value);
```

### 2.2 Coluna de Embedding (pgvector)

```sql
-- Migration: add_embedding_column_opportunities
ALTER TABLE opportunities
ADD COLUMN embedding vector(1536);

CREATE INDEX idx_opportunities_embedding
ON opportunities
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

### 2.3 Habilitar Extensões

```sql
-- Migration: enable_extensions
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "vector";
```

---

## 3. Convenções do Schema

| **Convenção** | **Regra** |
|---|---|
| Primary keys | UUID v4 via `gen_random_uuid()` |
| Timestamps | Sempre `@db.Timestamptz` — nunca sem timezone |
| Soft delete | `deletedAt DateTime? @map("deleted_at") @db.Timestamptz` |
| Auditoria | `createdAt` e `updatedAt` obrigatórios em toda tabela |
| Nomenclatura | Tabelas: `@@map("snake_case_plural")` · Colunas: `@map("snake_case")` |
| Enums | PascalCase no Prisma, UPPER_SNAKE_CASE como valores |
| Foreign keys | Todo FK tem índice correspondente |
| Decimais financeiros | `@db.Decimal(15, 2)` — nunca Float para valores monetários |

---

## 4. Changelog

| **Data** | **Versão** | **Descrição** |
|---|---|---|
| 22/03/2026 | v1.0 | Criação inicial — Pipeline ShiftLabs v9.5 |
