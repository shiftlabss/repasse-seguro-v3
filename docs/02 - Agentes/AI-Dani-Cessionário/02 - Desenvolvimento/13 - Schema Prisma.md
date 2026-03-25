# 13 - Schema Prisma

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Engenharia Backend | Schema Prisma completo do módulo AI-Dani-Cessionário | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - Schema Prisma completo para as 7 tabelas do módulo da Dani (D12).
> - UUID v4 como PK em todas as tabelas. `created_at` / `updated_at` obrigatórios.
> - Enums declarados para campos com valores fixos: `CanalDani`, `StatusConversa`, `RemetenteMensagem`, `EstadoAgente`, `TipoAlerta`, `StatusAlerta`, `EstadoVinculacaoWhatsapp`, `EventoRateLimit`.
> - A tabela `Cessionario` é uma referência externa — definida pela plataforma principal.
> - Convenção ShiftLabs v7.1: `snake_case` no banco via `@map`, `PascalCase` no Prisma.

---

## 1. Configuração do Schema

```prisma
// ============================================================
// AI-Dani-Cessionário — Schema Prisma v1.0
// Padrão ShiftLabs v7.1
// ============================================================

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

---

## 2. Enums

```prisma
enum CanalDani {
  WEBCHAT
  WHATSAPP

  @@map("canal_dani")
}

enum StatusConversa {
  ATIVA
  ENCERRADA
  EXPIRADA

  @@map("status_conversa")
}

enum RemetenteMensagem {
  CESSIONARIO
  DANI
  SISTEMA
  CALCULADORA

  @@map("remetente_mensagem")
}

enum EstadoAgente {
  OPERACIONAL
  FALLBACK
  DESLIGADO

  @@map("estado_agente")
}

enum TipoAlerta {
  NOVA_OPORTUNIDADE
  OPORTUNIDADE_ENCERRADA
  NEGOCIACAO_ATUALIZADA
  ESCROW_PRAZO
  ZAPSIGN_PENDENTE

  @@map("tipo_alerta")
}

enum StatusAlerta {
  PENDENTE
  ENVIADO
  LIDO
  FALHA

  @@map("status_alerta")
}

enum EstadoVinculacaoWhatsapp {
  NAO_VINCULADO
  OTP_SMS_ENVIADO
  AGUARDANDO_CONFIRMACAO_WA
  VINCULADO
  DESVINCULADO

  @@map("estado_vinculacao_whatsapp")
}

enum EventoRateLimitAudit {
  LIMITE_ATINGIDO
  DESBLOQUEADO
  HARD_BLOCK_OTP

  @@map("evento_rate_limit_audit")
}
```

---

## 3. Models

### 3.1 Referência ao Cessionário (tabela externa)

```prisma
// Referência à tabela da plataforma principal.
// Este model NÃO é gerenciado pelo módulo da Dani.
// Apenas para tipagem das FKs.
model Cessionario {
  id         String   @id @default(uuid()) @db.Uuid
  nome       String
  email      String   @unique
  kyc_status String

  created_at DateTime @default(now()) @db.Timestamptz
  updated_at DateTime @updatedAt @db.Timestamptz

  // Relações para o módulo da Dani
  dani_conversas            DaniConversa[]
  dani_mensagens            DaniMensagem[]
  dani_sessoes              DaniSessao[]
  dani_contextos_opr        DaniContextoOpr[]
  dani_alertas              DaniAlerta[]
  dani_vinculacao_whatsapp  DaniVinculacaoWhatsapp?
  dani_rate_limits_audit    DaniRateLimitAudit[]

  @@map("cessionarios")
}
```

---

### 3.2 DaniConversa

```prisma
model DaniConversa {
  id              String         @id @default(uuid()) @db.Uuid
  cessionario_id  String         @db.Uuid
  canal           CanalDani
  status          StatusConversa @default(ATIVA)
  expires_at      DateTime       @db.Timestamptz

  created_at      DateTime       @default(now()) @db.Timestamptz
  updated_at      DateTime       @updatedAt @db.Timestamptz

  // Relações
  cessionario     Cessionario      @relation(fields: [cessionario_id], references: [id])
  mensagens       DaniMensagem[]
  sessoes         DaniSessao[]
  contextos_opr   DaniContextoOpr[]

  // Índices
  @@index([cessionario_id])
  @@index([expires_at])
  @@index([cessionario_id, canal])

  @@map("dani_conversas")
}
```

---

### 3.3 DaniMensagem

```prisma
model DaniMensagem {
  id              String             @id @default(uuid()) @db.Uuid
  conversa_id     String             @db.Uuid
  cessionario_id  String             @db.Uuid
  remetente       RemetenteMensagem
  conteudo        String             @db.Text
  metadata        Json?              @db.JsonB
  // metadata shape (documentado para referência):
  // {
  //   tipo_resposta?: "analise" | "simulacao" | "comparacao" | "suporte" | "recusa"
  //   opr_ids?: string[]
  //   valores_calculados?: { delta?: number, comissao?: number, custo_total?: number, roi_base?: number }
  //   confianca_modelo?: number      // 0.0 a 1.0
  //   latency_ms?: number
  //   fallback_ativo?: boolean
  // }

  created_at      DateTime           @default(now()) @db.Timestamptz
  updated_at      DateTime           @updatedAt @db.Timestamptz

  // Relações
  conversa        DaniConversa   @relation(fields: [conversa_id], references: [id])
  cessionario     Cessionario    @relation(fields: [cessionario_id], references: [id])

  // Índices
  @@index([conversa_id])
  @@index([cessionario_id])
  @@index([conversa_id, created_at(sort: Desc)])

  @@map("dani_mensagens")
}
```

---

### 3.4 DaniSessao

```prisma
model DaniSessao {
  id                 String       @id @default(uuid()) @db.Uuid
  cessionario_id     String       @db.Uuid
  conversa_id        String       @db.Uuid
  jwt_session_hash   String
  estado_agente      EstadoAgente @default(OPERACIONAL)
  expires_at         DateTime     @db.Timestamptz

  created_at         DateTime     @default(now()) @db.Timestamptz
  updated_at         DateTime     @updatedAt @db.Timestamptz

  // Relações
  cessionario        Cessionario  @relation(fields: [cessionario_id], references: [id])
  conversa           DaniConversa @relation(fields: [conversa_id], references: [id])

  // Índices
  @@index([cessionario_id])
  @@index([conversa_id])
  @@index([expires_at])

  @@map("dani_sessoes")
}
```

---

### 3.5 DaniContextoOpr

```prisma
model DaniContextoOpr {
  id                   String       @id @default(uuid()) @db.Uuid
  conversa_id          String       @db.Uuid
  cessionario_id       String       @db.Uuid
  opr_id               String
  dados_oportunidade   Json         @db.JsonB
  // dados_oportunidade shape:
  // {
  //   tabela_atual: number
  //   tabela_contrato: number
  //   valor_pago_cedente: number
  //   preco_repasse: number
  //   localizacao: { cidade: string, estado: string, empreendimento: string }
  //   tipologia: string            // ex: "2BR", "3BR"
  //   status_marketplace: "DISPONIVEL" | "EM_NEGOCIACAO" | "ENCERRADA"
  //   snapshot_at: string          // ISO timestamp do snapshot
  // }

  created_at           DateTime     @default(now()) @db.Timestamptz
  updated_at           DateTime     @updatedAt @db.Timestamptz

  // Relações
  conversa             DaniConversa @relation(fields: [conversa_id], references: [id])
  cessionario          Cessionario  @relation(fields: [cessionario_id], references: [id])

  // Índices
  @@index([conversa_id])
  @@index([cessionario_id])
  @@index([opr_id])

  @@map("dani_contextos_opr")
}
```

---

### 3.6 DaniAlerta

```prisma
model DaniAlerta {
  id              String       @id @default(uuid()) @db.Uuid
  cessionario_id  String       @db.Uuid
  tipo            TipoAlerta
  status          StatusAlerta @default(PENDENTE)
  payload         Json         @db.JsonB
  // payload shape:
  // {
  //   opr_id?: string
  //   mensagem: string
  //   link_acao?: string
  //   dados_extras?: Record<string, unknown>
  // }
  lido_em         DateTime?    @db.Timestamptz

  created_at      DateTime     @default(now()) @db.Timestamptz
  updated_at      DateTime     @updatedAt @db.Timestamptz

  // Relações
  cessionario     Cessionario  @relation(fields: [cessionario_id], references: [id])

  // Índices
  @@index([cessionario_id, status])
  @@index([cessionario_id, created_at(sort: Desc)])

  @@map("dani_alertas")
}
```

---

### 3.7 DaniVinculacaoWhatsapp

```prisma
model DaniVinculacaoWhatsapp {
  id                String                    @id @default(uuid()) @db.Uuid
  cessionario_id    String                    @unique @db.Uuid
  numero_telefone   String
  // Armazenar apenas hash bcrypt do número completo.
  // Exibir apenas os últimos 4 dígitos em texto claro (derivado do hash não é possível).
  // Use campo separado numero_telefone_sufixo para exibição.
  numero_sufixo     String                    // Últimos 4 dígitos para exibição: "(XX) •••••-XXXX"
  estado            EstadoVinculacaoWhatsapp  @default(NAO_VINCULADO)
  otp_hash          String?                   // Hash bcrypt do OTP — nunca texto claro
  tentativas_otp    Int                       @default(0)
  otp_expires_at    DateTime?                 @db.Timestamptz
  vinculado_em      DateTime?                 @db.Timestamptz

  created_at        DateTime                  @default(now()) @db.Timestamptz
  updated_at        DateTime                  @updatedAt @db.Timestamptz

  // Relações
  cessionario       Cessionario               @relation(fields: [cessionario_id], references: [id])

  // Índices
  @@index([estado])

  @@map("dani_vinculacoes_whatsapp")
}
```

---

### 3.8 DaniRateLimitAudit

```prisma
model DaniRateLimitAudit {
  id              String                @id @default(uuid()) @db.Uuid
  cessionario_id  String                @db.Uuid
  canal           CanalDani
  evento          EventoRateLimitAudit
  contagem        Int
  janela_inicio   DateTime              @db.Timestamptz

  created_at      DateTime              @default(now()) @db.Timestamptz
  // Nota: sem updated_at — tabela de auditoria (append-only)

  // Relações
  cessionario     Cessionario           @relation(fields: [cessionario_id], references: [id])

  // Índices
  @@index([cessionario_id, created_at(sort: Desc)])

  @@map("dani_rate_limits_audit")
}
```

---

## 4. Migrations

### 4.1 Estratégia

- Migrations declarativas via `prisma migrate dev` (ambiente local) e `prisma migrate deploy` (produção).
- Nunca executar `prisma db push` em produção.
- Toda migration com nome descritivo: `YYYYMMDDHHMMSS_nome_descritivo`.
- Breaking migrations (DROP COLUMN, RENAME TABLE) exigem feature flag e período de transição.

### 4.2 Primeira migration esperada

```bash
npx prisma migrate dev --name init_dani_module
```

Cria as 7 tabelas + enums + índices conforme o schema acima.

### 4.3 Job de limpeza de histórico (90 dias)

```typescript
// Executado diariamente via job agendado
// src/agente/jobs/cleanup-conversas.job.ts

async function limparConversasExpiradas() {
  const agora = new Date()

  // Deleta mensagens de conversas expiradas
  await prisma.daniMensagem.deleteMany({
    where: {
      conversa: {
        expires_at: { lt: agora }
      }
    }
  })

  // Deleta contextos OPR de conversas expiradas
  await prisma.daniContextoOpr.deleteMany({
    where: {
      conversa: {
        expires_at: { lt: agora }
      }
    }
  })

  // Deleta sessões de conversas expiradas
  await prisma.daniSessao.deleteMany({
    where: {
      conversa: {
        expires_at: { lt: agora }
      }
    }
  })

  // Deleta conversas expiradas
  await prisma.daniConversa.deleteMany({
    where: {
      expires_at: { lt: agora }
    }
  })
}
```

---

## 5. Consultas Obrigatórias (com Filtro RBAC)

Todo acesso ao banco de dados no contexto da Dani deve incluir o filtro `cessionario_id`. Exemplos:

```typescript
// CORRETO — sempre filtrar por cessionario_id
const conversas = await prisma.daniConversa.findMany({
  where: {
    cessionario_id: cessionarioId, // extraído do JWT
    status: 'ATIVA'
  },
  orderBy: { created_at: 'desc' }
})

// CORRETO — histórico de mensagens com filtro duplo
const mensagens = await prisma.daniMensagem.findMany({
  where: {
    conversa_id: conversaId,
    cessionario_id: cessionarioId // redundante mas obrigatório
  },
  orderBy: { created_at: 'asc' },
  take: 100 // paginação
})

// ERRADO — sem filtro de cessionario_id (nunca fazer)
// const todasMensagens = await prisma.daniMensagem.findMany()
```

---

## Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. Schema completo com 8 models (1 externo + 7 do módulo), 8 enums, índices, job de limpeza e exemplos de consulta com RBAC. |
