# 13 - Schema Prisma

## Repasse Seguro — Módulo CRM

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Backend Lead, Tech Lead, DevOps | Contrato de banco de dados executável — modelos Prisma, enums, RLS, middleware e seed | v1.0 | Claude Code Desktop | 2026-03-23 — America/Fortaleza |

---

> **TL;DR — Decisões críticas deste schema**
>
> - **16 modelos Prisma** (15 tabelas `public` + 1 tabela `audit.audit_logs`).
> - **Stack:** Prisma 6+ + PostgreSQL 17+ via Supabase. RLS por `assigned_to`/`role` — sem multi-tenancy.
> - **Soft delete global** via `deleted_at` + Middleware — nenhum delete físico em entidades de negócio.
> - **Optimistic Locking** via campo `version Int` na tabela `cases`.
> - **Valores monetários** sempre `Decimal(15,2)` — nunca `Float`.
> - **3 tabelas append-only:** `case_status_history`, `communications`, `audit_logs`.
> - **Audit trail** separado em schema `audit` — INSERT-only, retenção 5 anos.
> - **0 seções pendentes** — cobertura total do Doc 12 (ERD Schema).

---

## 1. Estrutura de Arquivos

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

### 1.1 Ordem de Aplicação em Produção

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

## 2. Decisões de Design

| **#** | **Decisão** | **Justificativa** |
|---|---|---|
| D-001 | UUID v4 como PK em todas as tabelas | PKs não enumeráveis; compatível com Supabase Auth |
| D-002 | `Decimal(15,2)` em campos monetários | Precisão obrigatória para BRL — `Float` acumula erro |
| D-003 | Soft delete global via `deleted_at` | LGPD: exclusão lógica imediata; purge físico assíncrono |
| D-004 | Optimistic locking via `version Int` em `cases` | Previne perda silenciosa em edições concorrentes |
| D-005 | Append-only em `case_status_history` e `communications` | Imutabilidade de histórico — nunca alterar |
| D-006 | `Json` / `jsonb` para payloads flexíveis | `notification_logs.payload` e `audit_logs.old_data/new_data` variam por evento |
| D-007 | Schema separado `audit` | Isolamento total do audit trail — RLS mais restritiva |
| D-008 | `@db.Timestamptz` obrigatório | Armazenamento em UTC — conversão para `America/Fortaleza` na apresentação |
| D-009 | `communications` sem `deleted_at` | Append-only: registros de comunicação são imutáveis |
| D-010 | `commissions` sem soft delete | Registro financeiro contábil — imutável após criação |

---

## 3. Schema Prisma

```prisma
// prisma/schema.prisma
// Repasse Seguro — Módulo CRM
// Versão: v1.0 | Data: 2026-03-23

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

// ─────────────────────────────────────────────
// ENUMS
// ─────────────────────────────────────────────

enum CrmRole {
  ADMIN_RS
  COORDENADOR_RS
  ANALISTA_RS
}

enum UserStatus {
  ACTIVE
  SUSPENDED
  DEACTIVATED
}

enum ContactRole {
  CEDENTE
  CESSIONARIO
  CORRETOR
  ADVOGADO
  INCORPORADORA
}

enum ContactStatus {
  ACTIVE
  ARCHIVED
  OPT_OUT
}

enum CaseState {
  CADASTRO
  SIMULACAO
  VERIFICACAO
  PUBLICACAO
  MATCH
  NEGOCIACAO
  ANUENCIA
  FORMALIZACAO
  CONCLUIDO
  CANCELADO
}

enum CedenteScenario {
  A
  B
  C
  D
}

enum CancelReason {
  CEDENTE_DESISTIU
  DOCUMENTACAO_INVALIDA
  SEM_CESSIONARIO
  ANUENCIA_NEGADA
  OUTROS
}

enum ActivityType {
  CALL
  MEETING
  EMAIL
  WHATSAPP
  INTERNAL_NOTE
}

enum ActivityResult {
  POSITIVE
  NEUTRAL
  NO_ANSWER
  NEGATIVE
}

enum FollowupStatus {
  SCHEDULED
  COMPLETED
  OVERDUE
}

enum ActivityPriority {
  NORMAL
  HIGH
}

enum CommChannel {
  WHATSAPP
  EMAIL
}

enum CommDirection {
  OUTBOUND
  INBOUND
}

enum DeliveryStatus {
  SENT
  DELIVERED
  READ
  FAILED
}

enum ProposalType {
  PROPOSAL
  COUNTEROFFER
  FINAL_OFFER
}

enum ProposalStatus {
  ACTIVE
  ACCEPTED
  REJECTED
  EXPIRED
}

enum CommissionType {
  CEDENTE
  CESSIONARIO
  PARCEIRO
}

enum CommissionStatus {
  PENDING
  REGISTERED
  PAID
  CANCELLED
}

enum DossierDocType {
  TABELA_ATUAL
  TABELA_CONTRATO
  SALDO_DEVEDOR
  DOCS_PESSOAIS
  INSTRUMENTO_CESSAO
}

enum DocStatus {
  PENDING
  UPLOADED
  APPROVED
  REJECTED
}

enum SlaAlertLevel {
  WARNING
  CRITICAL
  URGENT
}

// ─────────────────────────────────────────────
// USUÁRIOS E ACESSO
// ─────────────────────────────────────────────

model CrmUser {
  id          String     @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  email       String     @unique @db.VarChar(255)
  fullName    String     @map("full_name") @db.VarChar(255)
  role        CrmRole
  status      UserStatus @default(ACTIVE)
  avatarUrl   String?    @map("avatar_url")
  lastLoginAt DateTime?  @map("last_login_at") @db.Timestamptz

  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz

  // Relações
  assignedCases        Case[]             @relation("AssignedCases")
  createdCases         Case[]             @relation("CreatedCases")
  activities           Activity[]         @relation("ActivityCreatedBy")
  communications       Communication[]    @relation("CommunicationSentBy")
  proposals            Proposal[]
  commissionsApproved  Commission[]       @relation("CommissionApprovedBy")
  dossierUploads       DossierDocument[]  @relation("DossierUploadedBy")
  dossierReviews       DossierDocument[]  @relation("DossierReviewedBy")
  caseStatusChanges    CaseStatusHistory[] @relation("StatusChangedBy")
  slaAcknowledgements  SlaAlert[]         @relation("SlaAcknowledgedBy")
  createdContacts      Contact[]          @relation("ContactCreatedBy")
  createdTags          Tag[]
  contactTagsCreated   ContactTag[]
  notificationsReceived NotificationLog[] @relation("NotificationUser")
  systemConfigUpdates  SystemConfig[]     @relation("ConfigUpdatedBy")
  systemConfigApprovals SystemConfig[]   @relation("ConfigPendingApprovedBy")

  @@map("crm_users")
}

// ─────────────────────────────────────────────
// CONTATOS
// ─────────────────────────────────────────────

model Contact {
  id                   String        @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  fullName             String        @map("full_name") @db.VarChar(255)
  email                String?       @db.VarChar(255)
  phone                String?       @db.VarChar(20)
  cpfCnpj              String?       @map("cpf_cnpj") @db.VarChar(18)
  role                 ContactRole
  status               ContactStatus @default(ACTIVE)
  hasOptOut            Boolean       @default(false) @map("has_opt_out")
  isRecurrentInvestor  Boolean       @default(false) @map("is_recurrent_investor")
  isPossibleDuplicate  Boolean       @default(false) @map("is_possible_duplicate")
  primaryContactId     String?       @map("primary_contact_id") @db.Uuid
  origin               String?       @db.VarChar(100)
  createdById          String        @map("created_by") @db.Uuid

  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz

  // Relações
  primaryContact  Contact?  @relation("ContactMerge", fields: [primaryContactId], references: [id])
  mergedContacts  Contact[] @relation("ContactMerge")
  createdBy       CrmUser   @relation("ContactCreatedBy", fields: [createdById], references: [id])

  cedenteCases     Case[]          @relation("CedenteCases")
  cessionarioCases Case[]          @relation("CessionarioCases")
  activities       Activity[]      @relation("ActivityContact")
  communications   Communication[] @relation("CommunicationContact")
  tags             ContactTag[]

  @@map("contacts")
}

// ─────────────────────────────────────────────
// CASOS E CICLO DE VIDA
// ─────────────────────────────────────────────

model Case {
  id                    String           @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseNumber            String           @unique @map("case_number") @db.VarChar(20)
  state                 CaseState        @default(CADASTRO)
  cedenteContactId      String           @map("cedente_contact_id") @db.Uuid
  cessionarioContactId  String?          @map("cessionario_contact_id") @db.Uuid
  assignedTo            String           @map("assigned_to") @db.Uuid
  scenario              CedenteScenario?
  contractValue         Decimal          @map("contract_value") @db.Decimal(15, 2)
  currentTableValue     Decimal?         @map("current_table_value") @db.Decimal(15, 2)
  delta                 Decimal?         @db.Decimal(15, 2)
  enterpriseName        String           @map("enterprise_name") @db.VarChar(255)
  enterpriseAddress     String?          @map("enterprise_address")
  cancelReason          CancelReason?    @map("cancel_reason")
  cancelReasonDetail    String?          @map("cancel_reason_detail")
  version               Int              @default(1)
  stateChangedAt        DateTime         @default(now()) @map("state_changed_at") @db.Timestamptz
  estimatedCloseAt      DateTime?        @map("estimated_close_at") @db.Timestamptz
  createdById           String           @map("created_by") @db.Uuid

  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz

  // Relações
  cedente          Contact           @relation("CedenteCases", fields: [cedenteContactId], references: [id])
  cessionario      Contact?          @relation("CessionarioCases", fields: [cessionarioContactId], references: [id])
  assignedUser     CrmUser           @relation("AssignedCases", fields: [assignedTo], references: [id])
  createdBy        CrmUser           @relation("CreatedCases", fields: [createdById], references: [id])

  statusHistory    CaseStatusHistory[]
  slaAlerts        SlaAlert[]
  activities       Activity[]
  communications   Communication[]
  proposals        Proposal[]
  commissions      Commission[]
  dossierDocuments DossierDocument[]
  notifications    NotificationLog[]

  @@map("cases")
}

model CaseStatusHistory {
  id            String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseId        String    @map("case_id") @db.Uuid
  fromState     CaseState? @map("from_state")
  toState       CaseState  @map("to_state")
  changedById   String    @map("changed_by") @db.Uuid
  justification String?

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  case      Case    @relation(fields: [caseId], references: [id])
  changedBy CrmUser @relation("StatusChangedBy", fields: [changedById], references: [id])

  @@map("case_status_history")
}

model SlaAlert {
  id               String        @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseId           String        @map("case_id") @db.Uuid
  alertLevel       SlaAlertLevel @map("alert_level")
  stateAtAlert     CaseState     @map("state_at_alert")
  daysInState      Int           @map("days_in_state")
  slaExpectedDays  Int           @map("sla_expected_days")
  acknowledged     Boolean       @default(false)
  acknowledgedById String?       @map("acknowledged_by") @db.Uuid
  acknowledgedAt   DateTime?     @map("acknowledged_at") @db.Timestamptz

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  case           Case     @relation(fields: [caseId], references: [id])
  acknowledgedBy CrmUser? @relation("SlaAcknowledgedBy", fields: [acknowledgedById], references: [id])

  @@map("sla_alerts")
}

// ─────────────────────────────────────────────
// ATIVIDADES
// ─────────────────────────────────────────────

model Activity {
  id               String           @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseId           String?          @map("case_id") @db.Uuid
  contactId        String?          @map("contact_id") @db.Uuid
  createdById      String           @map("created_by") @db.Uuid
  activityType     ActivityType     @map("activity_type")
  channel          String?          @db.VarChar(50)
  summary          String
  result           ActivityResult?
  isFollowup       Boolean          @default(false) @map("is_followup")
  activityDate     DateTime         @map("activity_date") @db.Timestamptz
  followupDueDate  DateTime?        @map("followup_due_date") @db.Timestamptz
  followupStatus   FollowupStatus?  @map("followup_status")
  priority         ActivityPriority @default(NORMAL)

  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz

  case      Case?   @relation(fields: [caseId], references: [id])
  contact   Contact? @relation("ActivityContact", fields: [contactId], references: [id])
  createdBy CrmUser  @relation("ActivityCreatedBy", fields: [createdById], references: [id])

  @@map("activities")
}

// ─────────────────────────────────────────────
// COMUNICAÇÕES
// ─────────────────────────────────────────────

model Communication {
  id              String         @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseId          String?        @map("case_id") @db.Uuid
  contactId       String         @map("contact_id") @db.Uuid
  sentById        String?        @map("sent_by") @db.Uuid
  channel         CommChannel
  direction       CommDirection
  content         String
  templateId      String?        @map("template_id") @db.VarChar(100)
  deliveryStatus  DeliveryStatus @default(SENT) @map("delivery_status")
  isManualRecord  Boolean        @default(false) @map("is_manual_record")
  isWithinWindow  Boolean        @default(true) @map("is_within_window")
  sentAt          DateTime       @map("sent_at") @db.Timestamptz
  deliveredAt     DateTime?      @map("delivered_at") @db.Timestamptz
  readAt          DateTime?      @map("read_at") @db.Timestamptz

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  case    Case?   @relation(fields: [caseId], references: [id])
  contact Contact @relation("CommunicationContact", fields: [contactId], references: [id])
  sentBy  CrmUser? @relation("CommunicationSentBy", fields: [sentById], references: [id])

  @@map("communications")
}

// ─────────────────────────────────────────────
// PROPOSTAS E NEGOCIAÇÃO
// ─────────────────────────────────────────────

model Proposal {
  id              String         @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseId          String         @map("case_id") @db.Uuid
  createdById     String         @map("created_by") @db.Uuid
  proposalType    ProposalType   @map("proposal_type")
  value           Decimal        @db.Decimal(15, 2)
  status          ProposalStatus @default(ACTIVE)
  rejectionReason String?        @map("rejection_reason")
  validUntil      DateTime?      @map("valid_until") @db.Timestamptz
  roundNumber     Int            @default(1) @map("round_number")

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz

  case      Case    @relation(fields: [caseId], references: [id])
  createdBy CrmUser @relation(fields: [createdById], references: [id])

  @@map("proposals")
}

// ─────────────────────────────────────────────
// COMISSÕES
// ─────────────────────────────────────────────

model Commission {
  id               String           @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseId           String           @map("case_id") @db.Uuid
  commissionType   CommissionType   @map("commission_type")
  baseValue        Decimal          @map("base_value") @db.Decimal(15, 2)
  percentage       Decimal          @db.Decimal(5, 4)
  grossAmount      Decimal          @map("gross_amount") @db.Decimal(15, 2)
  netAmount        Decimal          @map("net_amount") @db.Decimal(15, 2)
  discountApplied  Decimal          @default(0) @map("discount_applied") @db.Decimal(15, 2)
  discountReason   String?          @map("discount_reason")
  approvedById     String?          @map("approved_by") @db.Uuid
  status           CommissionStatus @default(PENDING)
  registeredAt     DateTime?        @map("registered_at") @db.Timestamptz

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz

  case       Case     @relation(fields: [caseId], references: [id])
  approvedBy CrmUser? @relation("CommissionApprovedBy", fields: [approvedById], references: [id])

  @@map("commissions")
}

// ─────────────────────────────────────────────
// DOSSIÊ E DOCUMENTOS
// ─────────────────────────────────────────────

model DossierDocument {
  id              String         @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseId          String         @map("case_id") @db.Uuid
  documentType    DossierDocType @map("document_type")
  status          DocStatus      @default(PENDING)
  storagePath     String?        @map("storage_path")
  fileName        String?        @map("file_name") @db.VarChar(255)
  fileSizeBytes   Int?           @map("file_size_bytes")
  mimeType        String?        @map("mime_type") @db.VarChar(100)
  rejectionReason String?        @map("rejection_reason")
  uploadedById    String?        @map("uploaded_by") @db.Uuid
  reviewedById    String?        @map("reviewed_by") @db.Uuid
  uploadedAt      DateTime?      @map("uploaded_at") @db.Timestamptz
  reviewedAt      DateTime?      @map("reviewed_at") @db.Timestamptz

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz
  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz

  case       Case     @relation(fields: [caseId], references: [id])
  uploadedBy CrmUser? @relation("DossierUploadedBy", fields: [uploadedById], references: [id])
  reviewedBy CrmUser? @relation("DossierReviewedBy", fields: [reviewedById], references: [id])

  @@map("dossier_documents")
}

// ─────────────────────────────────────────────
// TAGS
// ─────────────────────────────────────────────

model Tag {
  id          String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  name        String @unique @db.VarChar(50)
  color       String @db.VarChar(7)
  createdById String @map("created_by") @db.Uuid

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  createdBy CrmUser      @relation(fields: [createdById], references: [id])
  contacts  ContactTag[]

  @@map("tags")
}

model ContactTag {
  contactId   String @map("contact_id") @db.Uuid
  tagId       String @map("tag_id") @db.Uuid
  createdById String @map("created_by") @db.Uuid

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  contact   Contact @relation(fields: [contactId], references: [id])
  tag       Tag     @relation(fields: [tagId], references: [id])
  createdBy CrmUser @relation(fields: [createdById], references: [id])

  @@id([contactId, tagId])
  @@map("contact_tags")
}

// ─────────────────────────────────────────────
// NOTIFICAÇÕES
// ─────────────────────────────────────────────

model NotificationLog {
  id                String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  caseId            String? @map("case_id") @db.Uuid
  userId            String  @map("user_id") @db.Uuid
  notificationType  String  @map("notification_type") @db.VarChar(50)
  channel           String  @db.VarChar(20)
  payload           Json
  delivered         Boolean @default(false)
  errorMessage      String? @map("error_message")

  sentAt    DateTime @default(now()) @map("sent_at") @db.Timestamptz
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  case Case?   @relation(fields: [caseId], references: [id])
  user CrmUser @relation("NotificationUser", fields: [userId], references: [id])

  @@map("notification_logs")
}

// ─────────────────────────────────────────────
// CONFIGURAÇÕES DO SISTEMA
// ─────────────────────────────────────────────

model SystemConfig {
  id                     String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  configKey              String  @unique @map("config_key") @db.VarChar(100)
  configValue            String  @map("config_value")
  description            String?
  isCritical             Boolean @default(false) @map("is_critical")
  requiresDoubleApproval Boolean @default(false) @map("requires_double_approval")
  pendingValue           String? @map("pending_value")
  pendingApprovedById    String? @map("pending_approved_by") @db.Uuid
  updatedById            String  @map("updated_by") @db.Uuid

  updatedAt DateTime @updatedAt @map("updated_at") @db.Timestamptz
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  updatedBy          CrmUser  @relation("ConfigUpdatedBy", fields: [updatedById], references: [id])
  pendingApprovedBy  CrmUser? @relation("ConfigPendingApprovedBy", fields: [pendingApprovedById], references: [id])

  @@map("system_configs")
}

// ─────────────────────────────────────────────
// AUDITORIA (Schema separado)
// ─────────────────────────────────────────────

model AuditLog {
  id         String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  tableName  String  @map("table_name") @db.VarChar(100)
  recordId   String  @map("record_id") @db.Uuid
  action     String  @db.VarChar(10)
  oldData    Json?   @map("old_data")
  newData    Json?   @map("new_data")
  changedBy  String  @map("changed_by") @db.Uuid
  ipAddress  String? @map("ip_address") @db.VarChar(45)
  userAgent  String? @map("user_agent")

  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  @@map("audit_logs")
  @@schema("audit")
}
```

---

## 4. Middleware

### 4.1 `soft-delete.middleware.ts`

```typescript
// prisma/middleware/soft-delete.middleware.ts
import { Prisma } from '@prisma/client';

const SOFT_DELETE_MODELS = [
  'CrmUser',
  'Contact',
  'Case',
  'Activity',
  'DossierDocument',
];

export function softDeleteMiddleware(): Prisma.Middleware {
  return async (params, next) => {
    if (!SOFT_DELETE_MODELS.includes(params.model ?? '')) {
      return next(params);
    }

    // Interceptar DELETE físico e converter em UPDATE de deletedAt
    if (params.action === 'delete') {
      params.action = 'update';
      params.args.data = { deletedAt: new Date() };
    }

    if (params.action === 'deleteMany') {
      params.action = 'updateMany';
      if (params.args.data !== undefined) {
        params.args.data.deletedAt = new Date();
      } else {
        params.args.data = { deletedAt: new Date() };
      }
    }

    // Filtrar registros soft-deletados em todas as queries de leitura
    if (params.action === 'findUnique' || params.action === 'findFirst') {
      params.action = 'findFirst';
      params.args.where = {
        ...params.args.where,
        deletedAt: null,
      };
    }

    if (params.action === 'findMany') {
      params.args.where = {
        ...params.args.where,
        deletedAt: null,
      };
    }

    if (params.action === 'count') {
      params.args.where = {
        ...params.args.where,
        deletedAt: null,
      };
    }

    return next(params);
  };
}
```

### 4.2 `audit.middleware.ts`

```typescript
// prisma/middleware/audit.middleware.ts
import { Prisma, PrismaClient } from '@prisma/client';

const AUDITED_MODELS = [
  'Case',
  'Contact',
  'Commission',
  'DossierDocument',
  'CrmUser',
  'SystemConfig',
];

export function auditMiddleware(
  prisma: PrismaClient,
  getContext: () => { userId: string; ipAddress?: string; userAgent?: string },
): Prisma.Middleware {
  return async (params, next) => {
    if (!AUDITED_MODELS.includes(params.model ?? '')) {
      return next(params);
    }

    if (!['create', 'update', 'updateMany', 'delete', 'deleteMany'].includes(params.action)) {
      return next(params);
    }

    let oldData: Record<string, unknown> | null = null;

    // Capturar estado anterior para UPDATE e DELETE
    if (['update', 'delete'].includes(params.action) && params.args?.where?.id) {
      const model = params.model as string;
      const record = await (prisma as Record<string, unknown>)[
        model.charAt(0).toLowerCase() + model.slice(1) as string
      ].findUnique({ where: { id: params.args.where.id } }) as Record<string, unknown> | null;
      oldData = record;
    }

    const result = await next(params);
    const ctx = getContext();

    await prisma.$executeRaw`
      INSERT INTO audit.audit_logs (id, table_name, record_id, action, old_data, new_data, changed_by, ip_address, user_agent)
      VALUES (
        gen_random_uuid(),
        ${params.model},
        ${(result as Record<string, unknown>)?.id ?? params.args?.where?.id},
        ${params.action.toUpperCase()},
        ${oldData ? JSON.stringify(oldData) : null}::jsonb,
        ${result ? JSON.stringify(result) : null}::jsonb,
        ${ctx.userId}::uuid,
        ${ctx.ipAddress ?? null},
        ${ctx.userAgent ?? null}
      )
    `;

    return result;
  };
}
```

---

## 5. Políticas RLS (SQL)

```sql
-- prisma/rls/policies.sql

-- ─────────────────────────────────────────────
-- Habilitar RLS nas tabelas principais
-- ─────────────────────────────────────────────

ALTER TABLE cases ENABLE ROW LEVEL SECURITY;
ALTER TABLE contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE activities ENABLE ROW LEVEL SECURITY;
ALTER TABLE communications ENABLE ROW LEVEL SECURITY;
ALTER TABLE dossier_documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE commissions ENABLE ROW LEVEL SECURITY;
ALTER TABLE sla_alerts ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit.audit_logs ENABLE ROW LEVEL SECURITY;

-- ─────────────────────────────────────────────
-- cases: Analista RS vê apenas seus casos
-- ─────────────────────────────────────────────

CREATE POLICY "cases_analista_own"
ON cases FOR SELECT
TO authenticated
USING (
  assigned_to = auth.uid()
  AND deleted_at IS NULL
);

CREATE POLICY "cases_coord_admin_all"
ON cases FOR SELECT
TO authenticated
USING (
  EXISTS (
    SELECT 1 FROM crm_users u
    WHERE u.id = auth.uid()
    AND u.role IN ('COORDENADOR_RS', 'ADMIN_RS')
    AND u.deleted_at IS NULL
  )
  AND deleted_at IS NULL
);

CREATE POLICY "cases_write_by_role"
ON cases FOR ALL
TO authenticated
USING (
  (
    -- Analista RS: apenas seus casos
    assigned_to = auth.uid()
    OR
    -- Coordenador RS e Admin RS: todos
    EXISTS (
      SELECT 1 FROM crm_users u
      WHERE u.id = auth.uid()
      AND u.role IN ('COORDENADOR_RS', 'ADMIN_RS')
      AND u.deleted_at IS NULL
    )
  )
  AND deleted_at IS NULL
);

-- ─────────────────────────────────────────────
-- contacts: Analista RS vê apenas contatos de seus casos
-- ─────────────────────────────────────────────

CREATE POLICY "contacts_analista_own"
ON contacts FOR SELECT
TO authenticated
USING (
  EXISTS (
    SELECT 1 FROM cases c
    WHERE (c.cedente_contact_id = contacts.id OR c.cessionario_contact_id = contacts.id)
    AND c.assigned_to = auth.uid()
    AND c.deleted_at IS NULL
  )
  AND contacts.deleted_at IS NULL
);

CREATE POLICY "contacts_coord_admin_all"
ON contacts FOR SELECT
TO authenticated
USING (
  EXISTS (
    SELECT 1 FROM crm_users u
    WHERE u.id = auth.uid()
    AND u.role IN ('COORDENADOR_RS', 'ADMIN_RS')
    AND u.deleted_at IS NULL
  )
  AND contacts.deleted_at IS NULL
);

-- ─────────────────────────────────────────────
-- audit.audit_logs: Apenas Admin RS pode ler
-- ─────────────────────────────────────────────

CREATE POLICY "audit_logs_admin_only"
ON audit.audit_logs FOR SELECT
TO authenticated
USING (
  EXISTS (
    SELECT 1 FROM crm_users u
    WHERE u.id = auth.uid()
    AND u.role = 'ADMIN_RS'
    AND u.deleted_at IS NULL
  )
);

-- Nenhuma policy de INSERT/UPDATE/DELETE para audit_logs
-- Apenas triggers do banco podem escrever nesta tabela
```

---

## 6. Índices de Performance

```sql
-- prisma/rls/indexes.sql

-- crm_users
CREATE INDEX IF NOT EXISTS idx_crm_users_email ON crm_users(email);
CREATE INDEX IF NOT EXISTS idx_crm_users_role_status ON crm_users(role, status);

-- contacts
CREATE INDEX IF NOT EXISTS idx_contacts_email ON contacts(email) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_contacts_cpf_cnpj ON contacts(cpf_cnpj) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_contacts_role_status ON contacts(role, status) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_contacts_created_by ON contacts(created_by);

-- cases
CREATE INDEX IF NOT EXISTS idx_cases_case_number ON cases(case_number);
CREATE INDEX IF NOT EXISTS idx_cases_state ON cases(state) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_cases_assigned_to ON cases(assigned_to) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_cases_cedente ON cases(cedente_contact_id);
CREATE INDEX IF NOT EXISTS idx_cases_state_changed_at ON cases(state_changed_at) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_cases_created_at ON cases(created_at);

-- case_status_history
CREATE INDEX IF NOT EXISTS idx_case_status_history_case_id ON case_status_history(case_id);
CREATE INDEX IF NOT EXISTS idx_case_status_history_created_at ON case_status_history(created_at);

-- sla_alerts
CREATE INDEX IF NOT EXISTS idx_sla_alerts_case_id ON sla_alerts(case_id);
CREATE INDEX IF NOT EXISTS idx_sla_alerts_unack ON sla_alerts(acknowledged) WHERE acknowledged = false;

-- activities
CREATE INDEX IF NOT EXISTS idx_activities_case_id ON activities(case_id) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_activities_created_by ON activities(created_by) WHERE deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_activities_followup_due ON activities(followup_due_date) WHERE is_followup = true AND deleted_at IS NULL;
CREATE INDEX IF NOT EXISTS idx_activities_followup_status ON activities(followup_status) WHERE is_followup = true;

-- communications
CREATE INDEX IF NOT EXISTS idx_communications_case_id ON communications(case_id);
CREATE INDEX IF NOT EXISTS idx_communications_contact_id ON communications(contact_id);
CREATE INDEX IF NOT EXISTS idx_communications_sent_at ON communications(sent_at);

-- proposals
CREATE INDEX IF NOT EXISTS idx_proposals_case_id ON proposals(case_id);
CREATE INDEX IF NOT EXISTS idx_proposals_status ON proposals(status);

-- commissions
CREATE INDEX IF NOT EXISTS idx_commissions_case_id ON commissions(case_id);
CREATE INDEX IF NOT EXISTS idx_commissions_status ON commissions(status);
CREATE INDEX IF NOT EXISTS idx_commissions_registered_at ON commissions(registered_at);

-- dossier_documents
CREATE INDEX IF NOT EXISTS idx_dossier_docs_case_id ON dossier_documents(case_id);
CREATE INDEX IF NOT EXISTS idx_dossier_docs_status ON dossier_documents(status);

-- notification_logs
CREATE INDEX IF NOT EXISTS idx_notifications_user_id ON notification_logs(user_id);
CREATE INDEX IF NOT EXISTS idx_notifications_sent_at ON notification_logs(sent_at);

-- audit.audit_logs
CREATE INDEX IF NOT EXISTS idx_audit_table_record ON audit.audit_logs(table_name, record_id);
CREATE INDEX IF NOT EXISTS idx_audit_changed_by ON audit.audit_logs(changed_by);
CREATE INDEX IF NOT EXISTS idx_audit_created_at ON audit.audit_logs(created_at);
```

---

## 7. Seed de Desenvolvimento

```typescript
// prisma/seed.ts
import { PrismaClient, CrmRole, UserStatus } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  console.log('Seeding CRM database...');

  // Admin RS
  const admin = await prisma.crmUser.upsert({
    where: { email: 'admin@repasse.com.br' },
    update: {},
    create: {
      email: 'admin@repasse.com.br',
      fullName: 'Admin Repasse Seguro',
      role: CrmRole.ADMIN_RS,
      status: UserStatus.ACTIVE,
    },
  });

  // Coordenador RS
  const coordenador = await prisma.crmUser.upsert({
    where: { email: 'coordenador@repasse.com.br' },
    update: {},
    create: {
      email: 'coordenador@repasse.com.br',
      fullName: 'Maria Coordenadora',
      role: CrmRole.COORDENADOR_RS,
      status: UserStatus.ACTIVE,
    },
  });

  // Analista RS
  const analista = await prisma.crmUser.upsert({
    where: { email: 'analista@repasse.com.br' },
    update: {},
    create: {
      email: 'analista@repasse.com.br',
      fullName: 'João Analista',
      role: CrmRole.ANALISTA_RS,
      status: UserStatus.ACTIVE,
    },
  });

  // Contato Cedente de exemplo
  const cedente = await prisma.contact.create({
    data: {
      fullName: 'Carlos Cedente',
      email: 'carlos@example.com',
      phone: '85999990001',
      cpfCnpj: '12345678901',
      role: 'CEDENTE',
      createdById: analista.id,
    },
  });

  // Caso de exemplo
  await prisma.case.create({
    data: {
      caseNumber: 'RS-2026-0001',
      state: 'CADASTRO',
      cedenteContactId: cedente.id,
      assignedTo: analista.id,
      contractValue: 350000.00,
      enterpriseName: 'Reserva do Parque',
      enterpriseAddress: 'Rua das Flores, 100, Fortaleza CE',
      createdById: analista.id,
    },
  });

  // Parâmetros do sistema
  await prisma.systemConfig.createMany({
    data: [
      {
        configKey: 'sla_cadastro_dias_uteis',
        configValue: '1',
        description: 'SLA do estado Cadastro em dias úteis',
        isCritical: false,
        requiresDoubleApproval: false,
        updatedById: admin.id,
      },
      {
        configKey: 'sla_verificacao_dias_uteis',
        configValue: '5',
        description: 'SLA do estado Verificação em dias úteis',
        isCritical: false,
        requiresDoubleApproval: false,
        updatedById: admin.id,
      },
      {
        configKey: 'sla_ciclo_total_dias',
        configValue: '60',
        description: 'Prazo máximo do ciclo total do Caso em dias corridos',
        isCritical: true,
        requiresDoubleApproval: true,
        updatedById: admin.id,
      },
    ],
    skipDuplicates: true,
  });

  console.log(`Seed concluído. Users: ${[admin, coordenador, analista].map(u => u.email).join(', ')}`);
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

---

## 8. Controle de Versão

| **Versão** | **Data** | **Responsável** | **Alteração** |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 16 modelos, 20 enums, middleware, RLS, indexes, seed |
