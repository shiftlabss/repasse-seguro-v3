# Schema Prisma — AI-Dani-Admin

## Schema Completo do Módulo de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend |
| Escopo | Schema Prisma completo do módulo AI-Dani-Admin — pronto para implementação |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D12 (ERD Schema), D02 (Stacks — Prisma 6.x+, Supabase PostgreSQL 17+) |

---

> **📌 TL;DR**
>
> - Schema Prisma completo com 8 models: `Interaction`, `Takeover`, `AgentConfiguration`, `AdminAccessLog`, `AlertEvent`, `LaunchReadinessChecklist`, `AdversarialTestResult`, `PushNotificationToken`.
> - Referencia models externos: `User`, `Agent` (gerenciados por outros módulos).
> - Todos os enums, índices e convenções ShiftLabs aplicados.
> - Pronto para `prisma migrate dev` após adicionar ao schema consolidado da plataforma.

---

## 1. Schema Completo

```prisma
// ============================================================
// AI-Dani-Admin — Prisma Schema
// Módulo: Supervisão Operacional
// Versão: v1.0 | 2026-03-23
// Fonte normativa: D02 (Stacks), D12 (ERD)
// ============================================================

// Enums

enum InteractionStatus {
  RESPONDIDA_PELA_IA
  SINALIZADA_PARA_REVISAO
  EM_TAKEOVER
  ENCERRADA
}

enum TakeoverStatus {
  ATIVO
  ENCERRADO
}

enum AdminActionType {
  ACESSO_PAINEL
  TAKEOVER_INICIADO
  TAKEOVER_ENCERRADO
  THRESHOLD_ALTERADO
  RATE_LIMIT_ALTERADO
  AGENTE_REATIVADO
  LANCAMENTO_AUTORIZADO
}

enum AlertType {
  DESLIGAMENTO_AUTOMATICO
  LATENCIA_ALTA
  TAXA_ERRO_ELEVADA
  CSAT_DEGRADADO
  TAXA_RECUSA_ALTA
  CONSUMO_PROCESSAMENTO
}

enum AlertStatus {
  ATIVO
  RECONHECIDO
  RESOLVIDO
}

enum ChecklistStatus {
  PENDENTE
  APROVADO
  BLOQUEADO
}

// Models

/// Registra cada interação entre usuário e agente de IA.
/// Tabela central do módulo AI-Dani-Admin.
/// Retenção: soft delete após 90 dias.
model Interaction {
  id               String            @id @default(uuid()) @db.Uuid
  user_id          String            @db.Uuid
  agent_id         String            @db.Uuid
  user_message     String
  agent_response   String
  confidence_score Int               // 0-100
  latency_ms       Int               // latência em milissegundos
  data_used        Json?             // dados utilizados pelo agente para gerar a resposta
  csat_score       Int?              // nota CSAT 1–5 dada pelo usuário ao encerrar; usado em csatAverage (RN-DA-031)
  status           InteractionStatus @default(RESPONDIDA_PELA_IA)
  created_at       DateTime          @default(now()) @db.Timestamptz
  updated_at       DateTime          @updatedAt @db.Timestamptz
  deleted_at       DateTime?         @db.Timestamptz // soft delete — retenção 90 dias

  // Relacionamentos
  takeover         Takeover?

  // Referências externas (não gerenciadas por este módulo)
  // user: User @relation(fields: [user_id], references: [id])
  // agent: Agent @relation(fields: [agent_id], references: [id])

  @@map("interactions")
  @@index([confidence_score, created_at], map: "idx_interactions_confidence_created")
  @@index([agent_id, created_at], map: "idx_interactions_agent_created")
  @@index([user_id, created_at], map: "idx_interactions_user_created")
  @@index([status], map: "idx_interactions_status")
  @@index([created_at], map: "idx_interactions_created_at")
  @@index([deleted_at], map: "idx_interactions_deleted_at")
}

/// Registra cada takeover manual pelo Admin.
/// Uma interação pode ter no máximo 1 takeover ativo por vez.
model Takeover {
  id             String        @id @default(uuid()) @db.Uuid
  interaction_id String        @unique @db.Uuid // UNIQUE: 1 takeover ativo por interação
  admin_id       String        @db.Uuid
  reason         String?       // motivo selecionado pelo Admin (opcional)
  status         TakeoverStatus @default(ATIVO)
  started_at     DateTime      @db.Timestamptz
  ended_at       DateTime?     @db.Timestamptz
  created_at     DateTime      @default(now()) @db.Timestamptz
  updated_at     DateTime      @updatedAt @db.Timestamptz

  // Relacionamentos
  interaction    Interaction   @relation(fields: [interaction_id], references: [id], onDelete: CASCADE)
  // admin: User @relation(fields: [admin_id], references: [id])

  @@map("takeovers")
  @@index([interaction_id], map: "idx_takeovers_interaction")
  @@index([admin_id, started_at], map: "idx_takeovers_admin_started")
  @@index([status], map: "idx_takeovers_status")
}

/// Armazena os parâmetros configuráveis dos agentes.
/// Uma configuração ativa por agente (upsert ao alterar).
model AgentConfiguration {
  id                  String   @id @default(uuid()) @db.Uuid
  agent_id            String   @unique @db.Uuid // UNIQUE: 1 config por agente
  confidence_threshold Int     @default(80)    // 50–95, padrão 80
  rate_limit_per_hour Int      @default(30)    // mensagens por hora por usuário
  updated_by          String   @db.Uuid        // admin_id que alterou
  created_at          DateTime @default(now()) @db.Timestamptz
  updated_at          DateTime @updatedAt @db.Timestamptz

  // Referências externas
  // agent: Agent @relation(fields: [agent_id], references: [id])
  // updatedBy: User @relation(fields: [updated_by], references: [id])

  @@map("agent_configurations")
  @@index([agent_id], map: "idx_agent_configs_agent")
}

/// Log imutável de acessos e ações dos Admins.
/// Hard delete após 365 dias. Sem soft delete — imutável por design.
model AdminAccessLog {
  id             String          @id @default(uuid()) @db.Uuid
  admin_id       String          @db.Uuid
  action_type    AdminActionType
  action_details Json            // detalhes da ação: filtros, valores, IDs
  created_at     DateTime        @default(now()) @db.Timestamptz

  // Referências externas
  // admin: User @relation(fields: [admin_id], references: [id])

  @@map("admin_access_logs")
  @@index([admin_id, created_at], map: "idx_access_logs_admin_created")
  @@index([action_type, created_at], map: "idx_access_logs_type_created")
}

/// Registra cada alerta automático disparado pelo sistema.
/// Hard delete após 365 dias.
model AlertEvent {
  id               String       @id @default(uuid()) @db.Uuid
  agent_id         String?      @db.Uuid // nullable: alertas de consumo não têm agente específico
  alert_type       AlertType
  alert_data       Json         // dados do alerta: valor medido, condição, período
  status           AlertStatus  @default(ATIVO)
  triggered_at     DateTime     @db.Timestamptz
  acknowledged_at  DateTime?    @db.Timestamptz
  acknowledged_by  String?      @db.Uuid // admin_id que reconheceu
  created_at       DateTime     @default(now()) @db.Timestamptz

  // Referências externas
  // agent: Agent? @relation(fields: [agent_id], references: [id])
  // acknowledgedBy: User? @relation(fields: [acknowledged_by], references: [id])

  @@map("alert_events")
  @@index([alert_type, triggered_at], map: "idx_alert_events_type_triggered")
  @@index([status, triggered_at], map: "idx_alert_events_status_triggered")
  @@index([agent_id, triggered_at], map: "idx_alert_events_agent_triggered")
}

/// Registra o estado do checklist de prontidão para lançamento.
/// Sem deleção automática — histórico permanente de lançamentos.
model LaunchReadinessChecklist {
  id                          String          @id @default(uuid()) @db.Uuid
  agent_id                    String          @db.Uuid
  scope_filter_check          Json            // {status, verifiedBy, verifiedAt, notes}
  context_filter_check        Json
  pen_test_check              Json
  agent_instructions_check    Json            // {identidade, dadosBloqueados, exemplosRecusa, formatoResposta}
  adversarial_tests_summary   Json            // {total, approved, failed}
  supervision_components_check Json           // {registro, dashboard, alerta, takeover}
  overall_status              ChecklistStatus @default(PENDENTE)
  authorized_by               String?         @db.Uuid // admin_id que autorizou
  authorized_at               DateTime?       @db.Timestamptz
  created_at                  DateTime        @default(now()) @db.Timestamptz
  updated_at                  DateTime        @updatedAt @db.Timestamptz

  // Relacionamentos
  adversarial_tests           AdversarialTestResult[]

  // Referências externas
  // agent: Agent @relation(fields: [agent_id], references: [id])
  // authorizedBy: User? @relation(fields: [authorized_by], references: [id])

  @@map("launch_readiness_checklists")
  @@index([agent_id], map: "idx_checklists_agent")
  @@index([overall_status], map: "idx_checklists_status")
}

/// Registra cada teste adversarial individual.
/// Sem deleção automática.
model AdversarialTestResult {
  id               String                   @id @default(uuid()) @db.Uuid
  checklist_id     String                   @db.Uuid
  question         String                   // pergunta adversarial testada
  expected_refusal String                   // resposta de recusa esperada
  actual_response  String                   // resposta real do agente
  was_refused      Boolean                  // true se o agente recusou corretamente
  tested_at        DateTime                 @db.Timestamptz
  created_at       DateTime                 @default(now()) @db.Timestamptz

  // Relacionamentos
  checklist        LaunchReadinessChecklist @relation(fields: [checklist_id], references: [id], onDelete: CASCADE)

  @@map("adversarial_test_results")
  @@index([checklist_id], map: "idx_adversarial_checklist")
}

/// Tokens de push notification dos Admins para alertas mobile.
model PushNotificationToken {
  id               String   @id @default(uuid()) @db.Uuid
  admin_id         String   @db.Uuid
  expo_push_token  String   @unique
  platform         String   // "ios" | "android"
  is_active        Boolean  @default(true)
  created_at       DateTime @default(now()) @db.Timestamptz
  updated_at       DateTime @updatedAt @db.Timestamptz

  // Referências externas
  // admin: User @relation(fields: [admin_id], references: [id])

  @@map("push_notification_tokens")
  @@index([admin_id], map: "idx_push_tokens_admin")
  @@unique([expo_push_token], map: "uq_push_tokens_token")
}
```

---

## 2. Migrations — Ordem de Criação

As migrations devem ser criadas nesta ordem para respeitar FKs:

```
1. agents (externo — já existe)
2. users (externo — já existe)
3. interactions
4. takeovers (depende de interactions)
5. agent_configurations
6. admin_access_logs
7. alert_events
8. launch_readiness_checklists
9. adversarial_test_results (depende de launch_readiness_checklists)
10. push_notification_tokens
```

---

## 3. Seeds de Desenvolvimento

```typescript
// prisma/seeds/ai-dani-admin.ts

import { PrismaClient, InteractionStatus, AlertType, AlertStatus } from '@prisma/client'

const prisma = new PrismaClient()

async function seedDaniAdmin() {
  // Seed de configuração padrão para agentes
  const agents = await prisma.agent.findMany({
    where: { name: { in: ['Dani-Cessionário', 'Dani-Cedente'] } }
  })

  for (const agent of agents) {
    await prisma.agentConfiguration.upsert({
      where: { agent_id: agent.id },
      update: {},
      create: {
        agent_id: agent.id,
        confidence_threshold: 80,
        rate_limit_per_hour: 30,
        updated_by: process.env.SEED_ADMIN_ID!,
      }
    })
  }

  // Seed de interações de exemplo para desenvolvimento
  // ... (criado automaticamente por testes de integração)
}

seedDaniAdmin()
  .catch(console.error)
  .finally(() => prisma.$disconnect())
```

---

## 4. Validações no Service Layer

### 4.1 Validação de confidence_threshold

```typescript
// Antes de salvar em AgentConfiguration
if (confidenceThreshold < 50 || confidenceThreshold > 95) {
  throw new BadRequestException(
    'O nível de supervisão precisa estar entre 50% e 95%.'
  )
}
```

### 4.2 Lock otimista para Takeover

```typescript
// Prevenir takeover simultâneo (RF-011)
const result = await prisma.$executeRaw`
  INSERT INTO takeovers (id, interaction_id, admin_id, reason, status, started_at, created_at, updated_at)
  VALUES (
    gen_random_uuid(),
    ${interactionId}::uuid,
    ${adminId}::uuid,
    ${reason},
    'ATIVO',
    NOW(),
    NOW(),
    NOW()
  )
  ON CONFLICT (interaction_id) DO NOTHING
  RETURNING id
`
// result = 0 → conflito → throw ConflictException
if (result === 0) {
  throw new ConflictException('Esta conversa já está em atendimento por outro analista.')
}
```

### 4.3 Query de Interações com Soft Delete

```typescript
// Todo service que consulta interactions deve filtrar deleted_at
const interactions = await prisma.interaction.findMany({
  where: {
    deleted_at: null,  // OBRIGATÓRIO — sempre filtrar soft delete
    // outros filtros...
  }
})
```

---

## 5. Exemplos de Estrutura JSON dos Campos

### 5.1 `interactions.data_used`

```json
{
  "source": "vector_store",
  "documents": [
    { "id": "doc-uuid-1", "title": "Tabela de comissões", "relevance": 0.92 }
  ],
  "tools_used": ["commission_calculator"],
  "context_window_tokens": 2847
}
```

### 5.2 `admin_access_logs.action_details`

Para THRESHOLD_ALTERADO:
```json
{
  "previous_value": 85,
  "new_value": 80,
  "agent_id": "agent-uuid"
}
```

Para TAKEOVER_INICIADO:
```json
{
  "interaction_id": "interaction-uuid",
  "reason": "Resposta incorreta da IA",
  "confidence_score": 61
}
```

### 5.3 `alert_events.alert_data`

```json
{
  "metric": "error_rate",
  "value": 32.5,
  "threshold": 30,
  "window_minutes": 15,
  "agent_name": "Dani-Cessionário",
  "sample_size": 48
}
```

### 5.4 `launch_readiness_checklists.supervision_components_check`

```json
{
  "interaction_logging": {
    "status": "APPROVED",
    "verifiedBy": "admin-uuid",
    "verifiedAt": "2026-03-23T12:00:00Z"
  },
  "metrics_dashboard": {
    "status": "APPROVED",
    "verifiedBy": "admin-uuid",
    "verifiedAt": "2026-03-23T12:00:00Z"
  },
  "confidence_alert": {
    "status": "PENDING"
  },
  "manual_takeover": {
    "status": "APPROVED",
    "verifiedBy": "admin-uuid",
    "verifiedAt": "2026-03-23T12:00:00Z"
  }
}
```

---

## 6. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Schema Prisma completo com 8 models, enums, índices e exemplos de uso. |
