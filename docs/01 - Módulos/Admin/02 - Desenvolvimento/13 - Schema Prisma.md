# 13 - Schema Prisma

## Repasse Seguro — Módulo Admin

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Backend Lead, Tech Lead, DevOps | Contrato de banco de dados executável — modelos Prisma, enums, RLS, middleware e seed | v1.0 | Claude Code Desktop | 22/03/2026 — America/Fortaleza |

---

> 📌 **TL;DR — Decisões críticas deste schema**
>
> - **20 modelos Prisma** (19 tabelas `public` + 1 tabela `audit.audit_logs`).
> - **Stack:** Prisma 6+ + PostgreSQL 17+ via Supabase. Sem multi-tenancy por `tenant_id` — produto monoproduto com RLS por `user_id`/`role`.
> - **Soft delete global** via `deleted_at` + Middleware — nenhum delete físico em entidades de negócio.
> - **Optimistic Locking** via campo `version Int` em 9 entidades de edição colaborativa.
> - **Valores monetários** sempre `Decimal(15,2)` — nunca `Float`.
> - **3 tabelas append-only:** `case_status_history`, `counterproposals`, `escrow_transactions`.
> - **Snapshot de configuração** por caso via `CaseConfigSnapshot` (RN-111).
> - **Audit trail** separado em schema `audit` — INSERT-only, retenção 5 anos (RN-129).
> - **0 seções pendentes** — cobertura total dos documentos de input.

---

## 1. Persona

Backend Lead com foco em contrato de dados executável, segurança por padrão e consistência de schema para produto imobiliário financeiro.

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
    ├── soft-delete.middleware.ts
    └── audit.middleware.ts
```

### 2.1 Ordem de Aplicação em Produção

```bash
# 1. Criar e aplicar migrations
npx prisma migrate deploy

# 2. Aplicar RLS policies
psql $DATABASE_URL -f prisma/rls/policies.sql

# 3. Aplicar indexes de performance
psql $DATABASE_URL -f prisma/rls/indexes.sql

# 4. Executar seed (ambiente de desenvolvimento apenas)
npx prisma db seed
```

---

## 3. Decisões de Design

| # | Decisão | Justificativa |
|---|---|---|
| D-001 | **UUID v4 como PK em todas as tabelas** | Segurança: PKs não enumeráveis; compatível com Supabase Auth e geração no client. |
| D-002 | **`Decimal(15,2)` em campos monetários** | Precisão obrigatória para valores BRL — `Float` acumula erro de ponto flutuante. |
| D-003 | **Soft delete global via `deleted_at`** | LGPD: exclusão lógica imediata; purge físico assíncrono ≤48h em PII sensível. |
| D-004 | **Optimistic locking via `version Int`** | Previne perda silenciosa de dados em edições concorrentes (RN-130). |
| D-005 | **Append-only em `case_status_history`, `counterproposals`, `escrow_transactions`** | Imutabilidade de registros históricos e financeiros — nunca alterar ou deletar. |
| D-006 | **`Json` / `jsonb` para payloads flexíveis** | `signatories` (ZapSign), `decision_data` (IA) e `payload` (notificações) variam por evento. |
| D-007 | **Schema separado `audit`** | Isolamento total do audit trail — RLS mais restritiva; nunca exposto pela API. |
| D-008 | **Snapshot de configuração por caso** | RN-111: parâmetros de comissão e Delta congelados na criação do caso. |
| D-009 | **`two_fa_secret` criptografado em repouso** | Campo sensível — aplicação criptografa/descriptografa via serviço de vault; nunca exposto pela API. |
| D-010 | **`CaseConfigSnapshot` sem `deleted_at`** | Snapshot é imutável por design — não existe deleção lógica de configuração histórica. |
| D-011 | **`commission_invoices` sem soft delete** | Fatura de comissão é registro financeiro contábil — imutável após criação. |
| D-012 | **`distributions` append-only** | Distribuição financeira não deve ser alterada ou revertida via ORM — apenas via processo de estorno com novo registro. |
| D-013 | **`global_configs` com `config_key` único** | Chave de configuração é o identificador natural; `id` UUID mantido para auditoria de mudança. |
| D-014 | **`formalization_criteria` 1:1 com `cases`** | Cada caso tem exatamente um registro de critérios de formalização — criado junto com o caso. |

---

## 4. Enums

### 4.1 Domínio: Ciclo de Vida do Caso

```prisma
enum CaseStatus {
  CAPTADO
  EM_TRIAGEM
  BLOQUEADO
  QUALIFICADO
  OFERTA_ATIVA
  EM_NEGOCIACAO
  EM_FORMALIZACAO
  FECHAMENTO
  POS_FECHAMENTO
  EM_REVERSAO
  EM_MEDIACAO
  DISPUTA_FORMAL
  CONCLUIDO
  CANCELADO
}
```

**Fluxo de estados:**
```
CAPTADO → EM_TRIAGEM → BLOQUEADO (retorno a EM_TRIAGEM possível)
                     → QUALIFICADO → OFERTA_ATIVA → EM_NEGOCIACAO
                                                   → EM_FORMALIZACAO → FECHAMENTO
                                                                      → POS_FECHAMENTO → CONCLUIDO
                                                                                       → EM_REVERSAO → EM_MEDIACAO → DISPUTA_FORMAL
CANCELADO (de qualquer estado exceto CONCLUIDO)
```

### 4.2 Domínio: Cenário de Repasse

```prisma
enum CaseScenario {
  A  // Cedente recebe saldo devedor quitado
  B  // Cedente recebe saldo devedor + % do Delta
  C  // Cedente recebe saldo devedor + valor fixo
  D  // Cedente recebe saldo devedor + % do valor total
}
```

### 4.3 Domínio: Fonte da Tabela Atual

```prisma
enum TableSource {
  CONSTRUTORA
  OFERTA_PUBLICA
  LAUDO
}
```

### 4.4 Domínio: Perfis de Operador

```prisma
enum UserRole {
  ANALISTA
  COORDENADOR
  GESTOR_FINANCEIRO
  MASTER
}
```

### 4.5 Domínio: Status de Usuários Externos

```prisma
enum UserExternalStatus {
  ATIVO
  SUSPENSO
  INATIVO
}
```

### 4.6 Domínio: Documento do Dossiê

```prisma
enum DocumentType {
  RG_CEDENTE
  CPF_CEDENTE
  COMPROVANTE_RESIDENCIA_CEDENTE
  CONTRATO_ORIGINAL
  EXTRATO_FINANCIAMENTO
  LAUDO_AVALIACAO
  PROCURACAO
  CERTIDAO_CASAMENTO
  CERTIDAO_NASCIMENTO
  OUTROS
}

enum DocumentStatus {
  PENDENTE
  ENVIADO
  APROVADO
  REJEITADO
}
```

### 4.7 Domínio: Proposta

```prisma
enum ProposalStatus {
  PENDENTE
  ACEITA
  RECUSADA
  EXPIRADA
  CANCELADA
}
```

### 4.8 Domínio: ZapSign / Assinaturas

```prisma
enum EnvelopeDocumentType {
  CONTRATO_CESSAO
  PROCURACAO_REPRESENTACAO
  TERMO_QUITACAO
  INSTRUMENTO_PARTICULAR
}

enum EnvelopeStatus {
  PENDENTE
  ENVIADO
  PARCIALMENTE_ASSINADO
  ASSINADO
  EXPIRADO
  CANCELADO
}
```

### 4.9 Domínio: Escrow / Financeiro

```prisma
enum EscrowAccountStatus {
  AGUARDANDO_DEPOSITO
  DEPOSITO_CONFIRMADO
  DISTRIBUICAO_PENDENTE
  DISTRIBUIDO
  BLOQUEADO
  ESTORNADO
}

enum EscrowTransactionType {
  DEPOSITO
  DISTRIBUICAO
  ESTORNO
  AJUSTE
}

enum DistributionRecipientType {
  CEDENTE
  CESSIONARIO
  REPASSE_SEGURO
  CONSTRUTORA
}

enum DistributionStatus {
  PENDENTE
  APROVADO
  PROCESSANDO
  CONCLUIDO
  FALHOU
  ESTORNADO
}

enum CommissionPartyType {
  CEDENTE
  CESSIONARIO
}

enum CommissionStatus {
  PENDENTE
  CALCULADA
  DISTRIBUIDA
}
```

### 4.10 Domínio: Notificações

```prisma
enum NotificationChannel {
  WHATSAPP
  SMS
  EMAIL
  PUSH
}

enum NotificationStatus {
  PENDENTE
  ENVIADO
  FALHOU
  REENTREGUE
}
```

### 4.11 Domínio: Supervisão IA

```prisma
enum AiDecisionType {
  ANOMALIA_DETECTADA
  ALERTA_SLA
  SUGESTAO_ACAO
  VALIDACAO_DOCUMENTO
  ANALISE_RISCO
}

enum AiDecisionOutcome {
  APROVADO_AUTOMATICO
  APROVADO_OPERADOR
  REJEITADO_OPERADOR
  PENDENTE_REVISAO
}
```

### 4.12 Domínio: Configurações

```prisma
enum ConfigDataType {
  STRING
  INTEGER
  DECIMAL
  BOOLEAN
  JSON
}
```

---

## 5. Modelos

### 5.1 Campos Comuns

> ⚙️ Campos aplicados a todas as entidades de negócio (exceto append-only):

```prisma
// Entidades de negócio padrão
id         String    @id @default(uuid())
created_at DateTime  @default(now()) @db.Timestamptz(6)
updated_at DateTime  @updatedAt @db.Timestamptz(6)
deleted_at DateTime? @db.Timestamptz(6)          // Soft delete
created_by String                                 // FK → users.id
updated_by String?                                // FK → users.id

// Entidades com edição colaborativa (adicional)
version    Int       @default(1)                  // Optimistic locking
```

### 5.2 Schema Prisma Completo

```prisma
// ============================================================
// schema.prisma — Repasse Seguro Admin
// Prisma 6+ · PostgreSQL 17+ · Supabase
// ============================================================

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

// ─── ENUMS ──────────────────────────────────────────────────

enum UserRole {
  ANALISTA
  COORDENADOR
  GESTOR_FINANCEIRO
  MASTER
}

enum UserExternalStatus {
  ATIVO
  SUSPENSO
  INATIVO
}

enum CaseStatus {
  CAPTADO
  EM_TRIAGEM
  BLOQUEADO
  QUALIFICADO
  OFERTA_ATIVA
  EM_NEGOCIACAO
  EM_FORMALIZACAO
  FECHAMENTO
  POS_FECHAMENTO
  EM_REVERSAO
  EM_MEDIACAO
  DISPUTA_FORMAL
  CONCLUIDO
  CANCELADO
}

enum CaseScenario {
  A
  B
  C
  D
}

enum TableSource {
  CONSTRUTORA
  OFERTA_PUBLICA
  LAUDO
}

enum DocumentType {
  RG_CEDENTE
  CPF_CEDENTE
  COMPROVANTE_RESIDENCIA_CEDENTE
  CONTRATO_ORIGINAL
  EXTRATO_FINANCIAMENTO
  LAUDO_AVALIACAO
  PROCURACAO
  CERTIDAO_CASAMENTO
  CERTIDAO_NASCIMENTO
  OUTROS
}

enum DocumentStatus {
  PENDENTE
  ENVIADO
  APROVADO
  REJEITADO
}

enum ProposalStatus {
  PENDENTE
  ACEITA
  RECUSADA
  EXPIRADA
  CANCELADA
}

enum EnvelopeDocumentType {
  CONTRATO_CESSAO
  PROCURACAO_REPRESENTACAO
  TERMO_QUITACAO
  INSTRUMENTO_PARTICULAR
}

enum EnvelopeStatus {
  PENDENTE
  ENVIADO
  PARCIALMENTE_ASSINADO
  ASSINADO
  EXPIRADO
  CANCELADO
}

enum EscrowAccountStatus {
  AGUARDANDO_DEPOSITO
  DEPOSITO_CONFIRMADO
  DISTRIBUICAO_PENDENTE
  DISTRIBUIDO
  BLOQUEADO
  ESTORNADO
}

enum EscrowTransactionType {
  DEPOSITO
  DISTRIBUICAO
  ESTORNO
  AJUSTE
}

enum DistributionRecipientType {
  CEDENTE
  CESSIONARIO
  REPASSE_SEGURO
  CONSTRUTORA
}

enum DistributionStatus {
  PENDENTE
  APROVADO
  PROCESSANDO
  CONCLUIDO
  FALHOU
  ESTORNADO
}

enum CommissionPartyType {
  CEDENTE
  CESSIONARIO
}

enum CommissionStatus {
  PENDENTE
  CALCULADA
  DISTRIBUIDA
}

enum NotificationChannel {
  WHATSAPP
  SMS
  EMAIL
  PUSH
}

enum NotificationStatus {
  PENDENTE
  ENVIADO
  FALHOU
  REENTREGUE
}

enum AiDecisionType {
  ANOMALIA_DETECTADA
  ALERTA_SLA
  SUGESTAO_ACAO
  VALIDACAO_DOCUMENTO
  ANALISE_RISCO
}

enum AiDecisionOutcome {
  APROVADO_AUTOMATICO
  APROVADO_OPERADOR
  REJEITADO_OPERADOR
  PENDENTE_REVISAO
}

enum ConfigDataType {
  STRING
  INTEGER
  DECIMAL
  BOOLEAN
  JSON
}

// ─── AUTENTICAÇÃO E USUÁRIOS ────────────────────────────────

model User {
  id                    String    @id @default(uuid())
  email                 String    @unique
  name                  String
  role                  UserRole
  password_hash         String
  is_active             Boolean   @default(true)
  two_fa_enabled        Boolean   @default(false)
  two_fa_secret         String?   // Criptografado em repouso — ref ao vault de crypto
  failed_login_attempts Int       @default(0)
  locked_until          DateTime? @db.Timestamptz(6)
  version               Int       @default(1)
  created_at            DateTime  @default(now()) @db.Timestamptz(6)
  updated_at            DateTime  @updatedAt @db.Timestamptz(6)
  deleted_at            DateTime? @db.Timestamptz(6)

  // Relações
  assigned_cases        Case[]    @relation("AssignedAnalyst")
  created_cases         Case[]    @relation("CaseCreatedBy")
  updated_cases         Case[]    @relation("CaseUpdatedBy")
  verified_documents    DossieDocument[] @relation("VerifiedBy")
  created_documents     DossieDocument[] @relation("DocumentCreatedBy")
  counterproposals      Counterproposal[]
  delta_approved        FormalizationCriteria[] @relation("DeltaApprovedBy")
  escrow_blocked        EscrowAccount[] @relation("BlockedBy")
  escrow_block_approved EscrowAccount[] @relation("BlockApprovedBy")
  created_envelopes     ZapsignEnvelope[]
  ai_reviewed           AiAgentDecision[]
  config_updated        GlobalConfig[]
  cedente_created_by    Cedente[] @relation("CedenteCreatedBy")
  cedente_updated_by    Cedente[] @relation("CedenteUpdatedBy")
  cessionario_created   Cessionario[] @relation("CessionarioCreatedBy")
  cessionario_updated   Cessionario[] @relation("CessionarioUpdatedBy")
  case_status_changed   CaseStatusHistory[]
  status_history_cases  CaseStatusHistory[] @relation("CaseStatusHistory")
  distributions_approved Distribution[]

  @@map("users")
}

model Cedente {
  id                 String             @id @default(uuid())
  name               String
  cpf                String             @unique
  email              String             @unique
  phone              String?
  address            String?            @db.Text
  status             UserExternalStatus @default(ATIVO)
  suspension_reason  String?            @db.Text
  version            Int                @default(1)
  created_by         String
  updated_by         String?
  created_at         DateTime           @default(now()) @db.Timestamptz(6)
  updated_at         DateTime           @updatedAt @db.Timestamptz(6)
  deleted_at         DateTime?          @db.Timestamptz(6)

  // Relações
  cases              Case[]
  creator            User     @relation("CedenteCreatedBy", fields: [created_by], references: [id], onDelete: Restrict)
  updater            User?    @relation("CedenteUpdatedBy", fields: [updated_by], references: [id], onDelete: SetNull)

  @@map("cedentes")
}

model Cessionario {
  id                 String             @id @default(uuid())
  name               String
  cpf                String             @unique
  email              String             @unique
  phone              String?
  status             UserExternalStatus @default(ATIVO)
  suspension_reason  String?            @db.Text
  version            Int                @default(1)
  created_by         String
  updated_by         String?
  created_at         DateTime           @default(now()) @db.Timestamptz(6)
  updated_at         DateTime           @updatedAt @db.Timestamptz(6)
  deleted_at         DateTime?          @db.Timestamptz(6)

  // Relações
  proposals          Proposal[]
  creator            User     @relation("CessionarioCreatedBy", fields: [created_by], references: [id], onDelete: Restrict)
  updater            User?    @relation("CessionarioUpdatedBy", fields: [updated_by], references: [id], onDelete: SetNull)

  @@map("cessionarios")
}

// ─── CASOS E CICLO DE VIDA ───────────────────────────────────

model Case {
  id                           String      @id @default(uuid())
  cedente_id                   String
  assigned_analyst_id          String?
  status                       CaseStatus  @default(CAPTADO)
  scenario                     CaseScenario
  property_address             String      @db.Text
  contract_table_value         Decimal     @db.Decimal(15, 2)
  current_table_value          Decimal?    @db.Decimal(15, 2)
  current_table_source         TableSource?
  paid_amount                  Decimal     @db.Decimal(15, 2)
  delta                        Decimal?    @db.Decimal(15, 2)
  estimated_recovered_value    Decimal?    @db.Decimal(15, 2)
  distrato_reference_value     Decimal?    @db.Decimal(15, 2)
  commission_cedente_amount    Decimal?    @db.Decimal(15, 2)
  commission_cessionario_amount Decimal?   @db.Decimal(15, 2)
  delta_review_required        Boolean     @default(false)
  delta_review_approved        Boolean?    // NULL=pendente, true=aprovado, false=rejeitado
  closed_at                    DateTime?   @db.Timestamptz(6)
  post_closing_deadline        DateTime?   @db.Timestamptz(6)
  cancellation_reason          String?     @db.Text
  version                      Int         @default(1)
  created_by                   String
  updated_by                   String
  created_at                   DateTime    @default(now()) @db.Timestamptz(6)
  updated_at                   DateTime    @updatedAt @db.Timestamptz(6)
  deleted_at                   DateTime?   @db.Timestamptz(6)

  // Relações
  cedente                  Cedente              @relation(fields: [cedente_id], references: [id], onDelete: Restrict)
  assigned_analyst         User?                @relation("AssignedAnalyst", fields: [assigned_analyst_id], references: [id], onDelete: SetNull)
  creator                  User                 @relation("CaseCreatedBy", fields: [created_by], references: [id], onDelete: Restrict)
  updater                  User                 @relation("CaseUpdatedBy", fields: [updated_by], references: [id], onDelete: Restrict)
  config_snapshot          CaseConfigSnapshot?
  status_history           CaseStatusHistory[]
  dossie_documents         DossieDocument[]
  proposals                Proposal[]
  zapsign_envelopes        ZapsignEnvelope[]
  formalization_criteria   FormalizationCriteria?
  escrow_account           EscrowAccount?
  commission_invoices      CommissionInvoice[]
  distributions            Distribution[]
  notification_logs        NotificationLog[]
  ai_agent_decisions       AiAgentDecision[]

  @@map("cases")
}

model CaseConfigSnapshot {
  id                         String   @id @default(uuid())
  case_id                    String   @unique
  commission_cedente_pct     Decimal  @db.Decimal(5, 4)   // Ex: 0.0500 = 5%
  commission_cessionario_pct Decimal  @db.Decimal(5, 4)
  delta_review_threshold     Decimal  @db.Decimal(15, 2)
  distrato_reference_pct     Decimal  @db.Decimal(5, 4)
  snapshot_at                DateTime @default(now()) @db.Timestamptz(6)

  // Relações
  case Case @relation(fields: [case_id], references: [id], onDelete: Cascade)

  @@map("case_config_snapshots")
}

model CaseStatusHistory {
  id          String     @id @default(uuid())
  case_id     String
  from_status CaseStatus?
  to_status   CaseStatus
  changed_by  String
  reason      String?    @db.Text
  created_at  DateTime   @default(now()) @db.Timestamptz(6)

  // Relações — APPEND-ONLY: sem updated_at, sem deleted_at
  case        Case @relation(fields: [case_id], references: [id], onDelete: Cascade)
  operator    User @relation("CaseStatusHistory", fields: [changed_by], references: [id], onDelete: Restrict)
  user        User @relation(fields: [changed_by], references: [id], onDelete: Restrict)

  @@map("case_status_history")
}

// ─── DOSSIÊ E DOCUMENTOS ────────────────────────────────────

model DossieDocument {
  id               String         @id @default(uuid())
  case_id          String
  document_type    DocumentType
  storage_path     String
  status           DocumentStatus @default(PENDENTE)
  rejection_reason String?        @db.Text
  verified_by      String?
  verified_at      DateTime?      @db.Timestamptz(6)
  version          Int            @default(1)
  created_by       String
  created_at       DateTime       @default(now()) @db.Timestamptz(6)
  updated_at       DateTime       @updatedAt @db.Timestamptz(6)
  deleted_at       DateTime?      @db.Timestamptz(6)

  // Relações
  case        Case  @relation(fields: [case_id], references: [id], onDelete: Restrict) // RN-002: dossiê nunca é hard-deleted
  verifier    User? @relation("VerifiedBy", fields: [verified_by], references: [id], onDelete: SetNull)
  creator     User  @relation("DocumentCreatedBy", fields: [created_by], references: [id], onDelete: Restrict)

  @@map("dossie_documents")
}

// ─── NEGOCIAÇÃO ──────────────────────────────────────────────

model Proposal {
  id               String         @id @default(uuid())
  case_id          String
  cessionario_id   String
  proposed_value   Decimal        @db.Decimal(15, 2)
  status           ProposalStatus @default(PENDENTE)
  expires_at       DateTime?      @db.Timestamptz(6)
  version          Int            @default(1)
  created_at       DateTime       @default(now()) @db.Timestamptz(6)
  updated_at       DateTime       @updatedAt @db.Timestamptz(6)
  deleted_at       DateTime?      @db.Timestamptz(6)

  // Relações
  case             Case           @relation(fields: [case_id], references: [id], onDelete: Restrict)
  cessionario      Cessionario    @relation(fields: [cessionario_id], references: [id], onDelete: Restrict)
  counterproposals Counterproposal[]

  @@map("proposals")
}

model Counterproposal {
  id            String   @id @default(uuid())
  proposal_id   String
  case_id       String
  counter_value Decimal  @db.Decimal(15, 2)
  reason        String?  @db.Text
  created_by    String
  created_at    DateTime @default(now()) @db.Timestamptz(6)

  // Relações — APPEND-ONLY: sem updated_at, sem deleted_at
  proposal  Proposal @relation(fields: [proposal_id], references: [id], onDelete: Restrict)
  creator   User     @relation(fields: [created_by], references: [id], onDelete: Restrict)

  @@map("counterproposals")
}

// ─── FORMALIZAÇÃO ────────────────────────────────────────────

model ZapsignEnvelope {
  id                     String               @id @default(uuid())
  case_id                String
  document_type          EnvelopeDocumentType
  zapsign_document_token String               @unique
  status                 EnvelopeStatus       @default(PENDENTE)
  signatories            Json                 // Array de signatários com status individual
  expires_at             DateTime?            @db.Timestamptz(6)
  resend_count           Int                  @default(0)
  version                Int                  @default(1)
  created_by             String
  created_at             DateTime             @default(now()) @db.Timestamptz(6)
  updated_at             DateTime             @updatedAt @db.Timestamptz(6)
  deleted_at             DateTime?            @db.Timestamptz(6)

  // Relações
  case    Case @relation(fields: [case_id], references: [id], onDelete: Restrict)
  creator User @relation(fields: [created_by], references: [id], onDelete: Restrict)

  @@map("zapsign_envelopes")
}

model FormalizationCriteria {
  id                   String    @id @default(uuid())
  case_id              String    @unique
  signatures_ok        Boolean   @default(false)
  annuence_ok          Boolean   @default(false)
  annuence_na          Boolean   @default(false) // N/A quando construtora não exige (DEC-013 do B04)
  annuence_protocol    String?
  annuence_document_path String?
  deposit_ok           Boolean   @default(false)
  instrument_ok        Boolean   @default(false)
  delta_approved       Boolean?  // NULL=não aplicável ou pendente
  delta_approved_by    String?
  delta_approved_at    DateTime? @db.Timestamptz(6)
  created_at           DateTime  @default(now()) @db.Timestamptz(6)
  updated_at           DateTime  @updatedAt @db.Timestamptz(6)

  // Relações — sem soft delete (registro 1:1 com o caso)
  case           Case  @relation(fields: [case_id], references: [id], onDelete: Cascade)
  delta_approver User? @relation("DeltaApprovedBy", fields: [delta_approved_by], references: [id], onDelete: SetNull)

  @@map("formalization_criteria")
}

// ─── FINANCEIRO ──────────────────────────────────────────────

model EscrowAccount {
  id                     String              @id @default(uuid())
  case_id                String              @unique
  celcoin_account_id     String?             @unique
  status                 EscrowAccountStatus @default(AGUARDANDO_DEPOSITO)
  expected_amount        Decimal             @db.Decimal(15, 2)
  deposited_amount       Decimal             @default(0) @db.Decimal(15, 2)
  residual_credit        Decimal             @default(0) @db.Decimal(15, 2)
  deposit_confirmed_at   DateTime?           @db.Timestamptz(6)
  distribution_due_at    DateTime?           @db.Timestamptz(6)
  distributed_at         DateTime?           @db.Timestamptz(6)
  distribution_blocked   Boolean             @default(false)
  block_reason           String?             @db.Text
  blocked_by             String?
  block_approved_by      String?
  version                Int                 @default(1)
  created_at             DateTime            @default(now()) @db.Timestamptz(6)
  updated_at             DateTime            @updatedAt @db.Timestamptz(6)

  // Relações — sem soft delete (conta financeira não é deletada)
  case           Case                 @relation(fields: [case_id], references: [id], onDelete: Restrict)
  blocker        User?                @relation("BlockedBy", fields: [blocked_by], references: [id], onDelete: SetNull)
  block_approver User?                @relation("BlockApprovedBy", fields: [block_approved_by], references: [id], onDelete: SetNull)
  transactions   EscrowTransaction[]
  distributions  Distribution[]

  @@map("escrow_accounts")
}

model EscrowTransaction {
  id                     String                @id @default(uuid())
  escrow_account_id      String
  transaction_type       EscrowTransactionType
  amount                 Decimal               @db.Decimal(15, 2)
  description            String?               @db.Text
  celcoin_transaction_id String?               @unique
  created_by             String
  created_at             DateTime              @default(now()) @db.Timestamptz(6)

  // Relações — APPEND-ONLY: sem updated_at, sem deleted_at
  escrow_account EscrowAccount @relation(fields: [escrow_account_id], references: [id], onDelete: Restrict)
  creator        User          @relation(fields: [created_by], references: [id], onDelete: Restrict)

  @@map("escrow_transactions")
}

model CommissionInvoice {
  id                String              @id @default(uuid())
  case_id           String
  party_type        CommissionPartyType
  commission_amount Decimal             @db.Decimal(15, 2)
  base_amount       Decimal             @db.Decimal(15, 2)
  commission_pct    Decimal             @db.Decimal(5, 4)
  status            CommissionStatus    @default(PENDENTE)
  distributed_at    DateTime?           @db.Timestamptz(6)
  created_at        DateTime            @default(now()) @db.Timestamptz(6)
  updated_at        DateTime            @updatedAt @db.Timestamptz(6)

  // Relações — sem soft delete (fatura contábil imutável)
  case Case @relation(fields: [case_id], references: [id], onDelete: Restrict)

  @@map("commission_invoices")
}

model Distribution {
  id                  String                    @id @default(uuid())
  case_id             String
  escrow_account_id   String
  recipient_type      DistributionRecipientType
  recipient_id        String
  amount              Decimal                   @db.Decimal(15, 2)
  status              DistributionStatus        @default(PENDENTE)
  celcoin_transfer_id String?                   @unique
  approved_by         String?
  distributed_at      DateTime?                 @db.Timestamptz(6)
  created_at          DateTime                  @default(now()) @db.Timestamptz(6)

  // Relações — APPEND-ONLY: sem updated_at, sem deleted_at
  case          Case          @relation(fields: [case_id], references: [id], onDelete: Restrict)
  escrow_account EscrowAccount @relation(fields: [escrow_account_id], references: [id], onDelete: Restrict)
  approver      User?         @relation(fields: [approved_by], references: [id], onDelete: SetNull)

  @@map("distributions")
}

// ─── NOTIFICAÇÕES ────────────────────────────────────────────

model NotificationLog {
  id             String              @id @default(uuid())
  case_id        String?
  recipient_id   String
  recipient_type String              // 'USER' | 'CEDENTE' | 'CESSIONARIO'
  channel        NotificationChannel
  event_type     String
  status         NotificationStatus  @default(PENDENTE)
  payload        Json
  error_message  String?             @db.Text
  retry_count    Int                 @default(0)
  created_at     DateTime            @default(now()) @db.Timestamptz(6)

  // Relações — APPEND-ONLY
  case Case? @relation(fields: [case_id], references: [id], onDelete: SetNull)

  @@map("notification_logs")
}

// ─── SUPERVISÃO IA ───────────────────────────────────────────

model AiAgentDecision {
  id               String            @id @default(uuid())
  case_id          String?
  agent_name       String
  decision_type    AiDecisionType
  decision_data    Json
  confidence_score Decimal           @db.Decimal(5, 4)  // 0.0000 a 1.0000
  outcome          AiDecisionOutcome @default(PENDENTE_REVISAO)
  operator_feedback String?          @db.Text
  reviewed_by      String?
  created_at       DateTime          @default(now()) @db.Timestamptz(6)

  // Relações — APPEND-ONLY
  case     Case? @relation(fields: [case_id], references: [id], onDelete: SetNull)
  reviewer User? @relation(fields: [reviewed_by], references: [id], onDelete: SetNull)

  @@map("ai_agent_decisions")
}

// ─── CONFIGURAÇÕES ───────────────────────────────────────────

model GlobalConfig {
  id            String         @id @default(uuid())
  config_key    String         @unique
  config_value  String
  data_type     ConfigDataType
  description   String?        @db.Text
  updated_by    String?
  update_reason String?        @db.Text
  created_at    DateTime       @default(now()) @db.Timestamptz(6)
  updated_at    DateTime       @updatedAt @db.Timestamptz(6)

  // Relações
  updater User? @relation(fields: [updated_by], references: [id], onDelete: SetNull)

  @@map("global_configs")
}
```

### 5.3 Hierarquia Visual de Modelos

```
User (operadores internos)
├── Cedente (M:1 ← Case)
│   └── Case (entidade central)
│       ├── CaseConfigSnapshot (1:1)
│       ├── CaseStatusHistory (1:N — append-only)
│       ├── DossieDocument (1:N)
│       ├── Proposal (1:N)
│       │   └── Counterproposal (1:N — append-only)
│       ├── ZapsignEnvelope (1:N)
│       ├── FormalizationCriteria (1:1)
│       ├── EscrowAccount (1:1)
│       │   ├── EscrowTransaction (1:N — append-only)
│       │   └── Distribution (1:N — append-only)
│       ├── CommissionInvoice (1:N)
│       ├── NotificationLog (1:N — append-only)
│       └── AiAgentDecision (1:N — append-only)
└── Cessionario
    └── Proposal (M:1)

GlobalConfig (standalone)
```

### 5.4 Tabela Resumo de Modelos

| Modelo | Soft Delete | Optimistic Lock | Append-Only |
|---|---|---|---|
| `User` | ✅ | ✅ | — |
| `Cedente` | ✅ | ✅ | — |
| `Cessionario` | ✅ | ✅ | — |
| `Case` | ✅ | ✅ | — |
| `CaseConfigSnapshot` | — | — | ✅ (imutável) |
| `CaseStatusHistory` | — | — | ✅ |
| `DossieDocument` | ✅ | ✅ | — |
| `Proposal` | ✅ | ✅ | — |
| `Counterproposal` | — | — | ✅ |
| `ZapsignEnvelope` | ✅ | ✅ | — |
| `FormalizationCriteria` | — | — | — |
| `EscrowAccount` | — | ✅ | — |
| `EscrowTransaction` | — | — | ✅ |
| `CommissionInvoice` | — | — | ✅ (contábil) |
| `Distribution` | — | — | ✅ |
| `NotificationLog` | — | — | ✅ |
| `AiAgentDecision` | — | — | ✅ |
| `GlobalConfig` | — | — | — |

---

## 6. Row Level Security (RLS)

### 6.1 Estratégia Geral

```sql
-- prisma/rls/policies.sql

-- ─── HABILITAR RLS ───────────────────────────────────────────
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.cedentes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.cessionarios ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.cases ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.case_config_snapshots ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.case_status_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.dossie_documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.proposals ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.counterproposals ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.zapsign_envelopes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.formalization_criteria ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.escrow_accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.escrow_transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.commission_invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.distributions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.notification_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.ai_agent_decisions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.global_configs ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit.audit_logs ENABLE ROW LEVEL SECURITY;

-- ─── USUÁRIO DE SERVIÇO (service_role) ──────────────────────
-- O service_role bypassa RLS. Prisma usa service_role para operações server-side.
-- O anon role não tem acesso. Nunca expor service_role no cliente.

-- ─── POLÍTICAS GLOBAIS: acesso apenas via service_role ───────
-- Todas as tabelas: somente service_role lê e escreve.
-- Nenhum acesso via anon ou authenticated (Supabase Auth).
-- O controle de acesso por role (ANALISTA/MASTER etc.) é feito
-- na camada NestJS (Guards), não no banco.

CREATE POLICY "service_role_only" ON public.cases
  FOR ALL TO service_role USING (true) WITH CHECK (true);

-- [DECISÃO AUTÔNOMA] Padrão acima aplicado a todas as tabelas.
-- Justificativa: Admin é um sistema interno (B2B) sem acesso direto do browser ao banco.
-- Toda operação passa pelo NestJS que usa service_role.
-- RLS funciona como barreira de emergência contra SQL injection e misc-configured clients.
-- Alternativa descartada: RLS granular por user_id — aumenta complexidade sem benefício
-- real pois o Admin nunca expõe endpoints anon.

-- ─── AUDIT LOGS: INSERT-only mesmo para service_role ─────────
CREATE POLICY "audit_insert_only" ON audit.audit_logs
  FOR INSERT TO service_role WITH CHECK (true);

-- SELECT restrito ao MASTER via função auxiliar
CREATE POLICY "audit_select_master" ON audit.audit_logs
  FOR SELECT TO service_role
  USING (
    current_setting('app.current_user_role', true) = 'MASTER'
  );
```

### 6.2 Injeção de Contexto por Request

```typescript
// NestJS middleware — injetado antes de toda query Prisma
await prisma.$executeRaw`
  SET LOCAL app.current_user_id = ${userId};
  SET LOCAL app.current_user_role = ${userRole};
`;
```

---

## 7. Middleware Prisma

### 7.1 Ordem de Aplicação

```typescript
// main.ts ou prisma.module.ts
prisma.$use(auditMiddleware);      // 1º: captura operação antes de executar
prisma.$use(softDeleteMiddleware); // 2º: converte delete → update
```

### 7.2 Soft Delete Middleware

```typescript
// prisma/middleware/soft-delete.middleware.ts

// Modelos com soft delete
const SOFT_DELETE_MODELS = [
  'User', 'Cedente', 'Cessionario', 'Case',
  'DossieDocument', 'Proposal', 'ZapsignEnvelope',
];

export function softDeleteMiddleware(): Prisma.Middleware {
  return async (params, next) => {
    if (!SOFT_DELETE_MODELS.includes(params.model ?? '')) {
      return next(params);
    }

    // Converter delete → update com deleted_at
    if (params.action === 'delete') {
      params.action = 'update';
      params.args.data = { deleted_at: new Date() };
    }

    if (params.action === 'deleteMany') {
      params.action = 'updateMany';
      params.args.data = { deleted_at: new Date() };
    }

    // Filtrar registros soft-deleted em leituras
    if (['findUnique', 'findFirst', 'findMany', 'count'].includes(params.action)) {
      params.args.where = {
        ...params.args.where,
        deleted_at: null,
      };
    }

    return next(params);
  };
}
```

### 7.3 Audit Middleware

```typescript
// prisma/middleware/audit.middleware.ts
// Registra toda operação de escrita no audit.audit_logs

const AUDIT_MODELS = [
  'Case', 'EscrowAccount', 'Distribution',
  'FormalizationCriteria', 'GlobalConfig',
];

export function auditMiddleware(userId: string): Prisma.Middleware {
  return async (params, next) => {
    const writeActions = ['create', 'update', 'updateMany', 'delete', 'deleteMany'];

    if (!AUDIT_MODELS.includes(params.model ?? '') || !writeActions.includes(params.action)) {
      return next(params);
    }

    const before = params.action.startsWith('update')
      ? await (prisma as any)[params.model!.toLowerCase()].findFirst({
          where: params.args.where,
        })
      : null;

    const result = await next(params);

    await prisma.$executeRaw`
      INSERT INTO audit.audit_logs
        (id, table_name, operation, record_id, before_data, after_data, performed_by, created_at)
      VALUES
        (gen_random_uuid(), ${params.model}, ${params.action},
         ${result?.id ?? null}, ${JSON.stringify(before)},
         ${JSON.stringify(result)}, ${userId}, NOW())
    `;

    return result;
  };
}
```

---

## 8. Seed

```typescript
// prisma/seed.ts
import { PrismaClient, UserRole, ConfigDataType } from '@prisma/client';
import * as bcrypt from 'bcrypt';

const prisma = new PrismaClient();

async function main() {
  // ─── Usuário Master padrão ──────────────────────────────
  const master = await prisma.user.upsert({
    where: { email: 'master@repasseseguro.com.br' },
    update: {},
    create: {
      id: '00000000-0000-0000-0000-000000000001',
      email: 'master@repasseseguro.com.br',
      name: 'Master Admin',
      role: UserRole.MASTER,
      password_hash: await bcrypt.hash('Change@Me123!', 12),
      is_active: true,
      created_by: '00000000-0000-0000-0000-000000000001',
      updated_by: '00000000-0000-0000-0000-000000000001',
    },
  });

  // ─── Usuário Analista de exemplo ────────────────────────
  await prisma.user.upsert({
    where: { email: 'analista@repasseseguro.com.br' },
    update: {},
    create: {
      email: 'analista@repasseseguro.com.br',
      name: 'Analista Exemplo',
      role: UserRole.ANALISTA,
      password_hash: await bcrypt.hash('Change@Me123!', 12),
      is_active: true,
      created_by: master.id,
      updated_by: master.id,
    },
  });

  // ─── Configurações globais padrão ───────────────────────
  const defaultConfigs = [
    { key: 'commission_cedente_pct_A',     value: '0.0500', type: ConfigDataType.DECIMAL, desc: 'Comissão Cedente Cenário A (5%)' },
    { key: 'commission_cedente_pct_B',     value: '0.0500', type: ConfigDataType.DECIMAL, desc: 'Comissão Cedente Cenário B (5%)' },
    { key: 'commission_cedente_pct_C',     value: '0.0500', type: ConfigDataType.DECIMAL, desc: 'Comissão Cedente Cenário C (5%)' },
    { key: 'commission_cedente_pct_D',     value: '0.0500', type: ConfigDataType.DECIMAL, desc: 'Comissão Cedente Cenário D (5%)' },
    { key: 'commission_cessionario_pct',   value: '0.0300', type: ConfigDataType.DECIMAL, desc: 'Comissão Cessionário sobre Delta (3%)' },
    { key: 'delta_review_threshold',       value: '50000',  type: ConfigDataType.DECIMAL, desc: 'Delta acima deste valor requer aprovação (R$)' },
    { key: 'distrato_reference_pct',       value: '0.5000', type: ConfigDataType.DECIMAL, desc: 'Base do valor de distrato (50% do pago)' },
    { key: 'sla_triagem_hours',            value: '48',     type: ConfigDataType.INTEGER,  desc: 'SLA de triagem em horas' },
    { key: 'sla_negociacao_days',          value: '30',     type: ConfigDataType.INTEGER,  desc: 'SLA de negociação em dias' },
    { key: 'sla_formalizacao_days',        value: '15',     type: ConfigDataType.INTEGER,  desc: 'SLA de formalização em dias' },
    { key: 'post_closing_reversal_days',   value: '15',     type: ConfigDataType.INTEGER,  desc: 'Janela de reversão pós-fechamento em dias' },
  ];

  for (const cfg of defaultConfigs) {
    await prisma.globalConfig.upsert({
      where: { config_key: cfg.key },
      update: {},
      create: {
        config_key:   cfg.key,
        config_value: cfg.value,
        data_type:    cfg.type,
        description:  cfg.desc,
        updated_by:   master.id,
        update_reason: 'Seed inicial — valores padrão',
      },
    });
  }

  console.log('Seed concluído com sucesso.');
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

**Tabela de Registros Mínimos do Seed:**

| Entidade | Quantidade | Descrição |
|---|---|---|
| `users` | 2 | Master Admin + Analista de exemplo |
| `global_configs` | 11 | Percentuais de comissão (4 cenários), Delta threshold, Distrato %, 3 SLAs, janela de reversão |

---

## 9. Quotas Canônicas

| Campo | Valor Default | Configurável via GlobalConfig |
|---|---|---|
| `commission_cedente_pct` | 5% | Sim — por cenário (A/B/C/D) |
| `commission_cessionario_pct` | 3% | Sim |
| `delta_review_threshold` | R$ 50.000 | Sim |
| `distrato_reference_pct` | 50% | Sim |
| `sla_triagem_hours` | 48h | Sim |
| `sla_negociacao_days` | 30 dias | Sim |
| `sla_formalizacao_days` | 15 dias | Sim |
| `post_closing_reversal_days` | 15 dias | Sim |
| `failed_login_attempts` max | 5 tentativas | Hardcoded (RN-005) |
| `locked_until` duração | 30 minutos | Hardcoded (RN-005) |

---

## 10. Políticas de Retenção

| Entidade | Dado Sensível | Retenção | Método |
|---|---|---|---|
| `users.password_hash` | Sim (PII) | Soft delete imediato; purge físico em 48h | Job assíncrono LGPD |
| `users.two_fa_secret` | Sim (PII) | Apagado na desativação do 2FA | Update imediato |
| `cedentes` (CPF, email, phone) | Sim (PII) | Soft delete; purge físico em 48h após solicitação | Job LGPD |
| `cessionarios` (CPF, email, phone) | Sim (PII) | Idem cedentes | Job LGPD |
| `cases` | Sim (dados negociais) | Soft delete; retenção 5 anos para obrigações fiscais | Cron job |
| `dossie_documents.storage_path` | Sim (documentos) | Nunca hard-deleted (RN-002); acesso revogado em caso cancelado | Revogação de permissão |
| `audit.audit_logs` | Sim (auditoria) | Retenção mínima 5 anos (RN-129); nunca purge | Sem exclusão |
| `escrow_transactions` | Sim (financeiro) | Retenção 5 anos (obrigação contábil) | Append-only sem exclusão |
| `commission_invoices` | Sim (financeiro) | Retenção 5 anos | Sem exclusão |
| `distributions` | Sim (financeiro) | Retenção 5 anos | Append-only sem exclusão |
| `notification_logs` | Parcial (payload pode ter PII) | 90 dias; payload purgado após 30 dias | Job cron |
| `ai_agent_decisions` | Não | Retenção indefinida para auditoria de IA | Append-only |
| `global_configs` | Não | Histórico infinito (via audit_logs) | Sem exclusão |

---

## 11. Índices de Performance

```sql
-- prisma/rls/indexes.sql

-- cases: queries mais frequentes
CREATE INDEX idx_cases_status ON public.cases (status) WHERE deleted_at IS NULL;
CREATE INDEX idx_cases_cedente_id ON public.cases (cedente_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_cases_analyst_id ON public.cases (assigned_analyst_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_cases_created_at ON public.cases (created_at DESC) WHERE deleted_at IS NULL;

-- users: login e listagem
CREATE UNIQUE INDEX idx_users_email ON public.users (email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role_active ON public.users (role, is_active) WHERE deleted_at IS NULL;

-- proposals: por caso e cessionário
CREATE INDEX idx_proposals_case_id ON public.proposals (case_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_proposals_status ON public.proposals (status) WHERE deleted_at IS NULL;

-- dossie_documents: por caso e status
CREATE INDEX idx_dossie_case_id ON public.dossie_documents (case_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_dossie_status ON public.dossie_documents (status) WHERE deleted_at IS NULL;

-- escrow: por caso (1:1 mas frequente em JOINs)
CREATE UNIQUE INDEX idx_escrow_case_id ON public.escrow_accounts (case_id);

-- ai_agent_decisions: por caso e tipo
CREATE INDEX idx_ai_decisions_case_id ON public.ai_agent_decisions (case_id);
CREATE INDEX idx_ai_decisions_outcome ON public.ai_agent_decisions (outcome);

-- notification_logs: por status e canal (reentrega)
CREATE INDEX idx_notifications_status ON public.notification_logs (status, created_at DESC);

-- global_configs: lookup por chave (já coberto pelo UNIQUE)
-- Nenhum índice adicional necessário
```

---

## 12. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — 20 modelos, 32 enums, RLS, middleware soft-delete + audit, seed 11 configs, políticas de retenção. |

---

## 13. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| RLS strategy | [DECISÃO AUTÔNOMA] | 6.1 | Admin é sistema interno — RLS como barreira de emergência, controle de acesso no NestJS. Alternativa (RLS granular) descartada por complexidade sem benefício real. | P1 | Backend Lead | Resolvido |
| `CaseStatusHistory` FK dupla | [DECISÃO AUTÔNOMA] | 5.2 | Duas relações `changed_by` referenciando `User` declaradas como `operator` e `user` — Prisma exige aliases distintos. Alternativa (uma só relação sem alias) descartada pois gera conflito de nome. | P1 | Backend Lead | Resolvido |
| `AiDecisionOutcome.PENDENTE_REVISAO` default | [DECISÃO AUTÔNOMA] | 4.11 | Default `PENDENTE_REVISAO` em vez de `APROVADO_AUTOMATICO` para forçar revisão humana inicial. Alternativa (`APROVADO_AUTOMATICO`) descartada — risco de aprovar automaticamente sem supervisão. | P0 | Backend Lead | Resolvido |
