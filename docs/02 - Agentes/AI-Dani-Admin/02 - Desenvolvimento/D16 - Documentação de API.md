# Documentação de API — AI-Dani-Admin

## API Reference — Módulo de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Backend, Frontend e QA |
| Escopo | API Reference com endpoints, autenticação, contratos, erros, paginação e exemplos |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Base URL | `/api/v1/admin` |
| Inputs | D01 (Regras de Negócio), D02 (Stacks), D12 (ERD Schema), D14 (Especificações Técnicas) |

---

> **📌 TL;DR**
>
> - **24 endpoints** distribuídos em 7 domínios: Supervisão, Takeover, Métricas, Configuração de Agente, Alertas, Checklist de Lançamento, Push Tokens.
> - **Autenticação:** JWT Bearer em todos os endpoints. Role `ADMIN` obrigatória — Cessionário/Cedente recebem 403.
> - **Paginação:** cursor-based via `cursor` + `limit`. Shape de resposta padrão: `{ data, meta: { nextCursor, hasMore, total } }`.
> - **Erros:** shape padrão `{ error: { code, message, details? } }`. Códigos no formato `DA-{MÓDULO}-{NNN}`.
> - **Rate limit:** `/api/v1/webchat/*` — 30 req/h por usuário (sliding window). Demais endpoints: sem rate limit adicional ao JWT.
> - **Versionamento:** prefixo `/v1/`. Breaking changes exigem `/v2/` sem deprecar o anterior por 90 dias.

---

## 1. Autenticação e Autorização

### 1.1 JWT Bearer

Todos os endpoints exigem header `Authorization: Bearer <access_token>`.

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

O token JWT deve conter:
```json
{
  "sub": "admin-uuid",
  "role": "ADMIN",
  "email": "admin@repasseseguro.com.br",
  "exp": 1711234567
}
```

**Role ADMIN é obrigatória.** Qualquer request com role `CESSIONARIO` ou `CEDENTE` retorna:
```json
HTTP 403 Forbidden
{
  "error": {
    "code": "DA-AUD-001",
    "message": "Acesso negado. Este recurso é restrito ao perfil Admin."
  }
}
```

### 1.2 Token Expirado

```json
HTTP 401 Unauthorized
{
  "error": {
    "code": "DA-AUD-002",
    "message": "Token expirado. Faça login novamente."
  }
}
```

---

## 2. Shape de Resposta Padrão

### 2.1 Sucesso — recurso único

```json
HTTP 200 OK
{
  "data": { ... }
}
```

### 2.2 Sucesso — lista paginada

```json
HTTP 200 OK
{
  "data": [ ... ],
  "meta": {
    "nextCursor": "uuid-do-ultimo-item",
    "hasMore": true,
    "total": 247
  }
}
```

### 2.3 Sucesso — criação

```json
HTTP 201 Created
{
  "data": { "id": "uuid", ... }
}
```

### 2.4 Erro

```json
HTTP 4xx / 5xx
{
  "error": {
    "code": "DA-TAK-001",
    "message": "Esta conversa já está em atendimento por outro analista.",
    "details": { "takeoverAdminId": "admin-uuid" }
  }
}
```

---

## 3. Paginação

Todos os endpoints de listagem usam cursor-based pagination.

**Query params:**
- `cursor` (string, UUID, opcional) — ID do último item recebido. Ausente = primeira página.
- `limit` (integer, opcional, padrão: 20, máx: 100)

**Exemplo de uso:**
```http
GET /api/v1/admin/interactions?limit=20
→ retorna primeiros 20 itens + meta.nextCursor

GET /api/v1/admin/interactions?cursor=<nextCursor>&limit=20
→ retorna próximos 20 itens
```

---

## 4. Endpoints — Supervisão de Interações

### `GET /api/v1/admin/interactions`

Lista interações com filtros. Registra acesso no log de auditoria (RF-004).

**Query params:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `agentId` | UUID | Não | Filtra por agente específico |
| `userId` | UUID | Não | Filtra por usuário (ID anonimizado) |
| `status` | string | Não | `RESPONDIDA_PELA_IA` \| `SINALIZADA_PARA_REVISAO` \| `EM_TAKEOVER` \| `ENCERRADA` |
| `confidenceMin` | integer (0-100) | Não | Confiança mínima |
| `confidenceMax` | integer (0-100) | Não | Confiança máxima |
| `dateFrom` | ISO 8601 | Não | Data início do período |
| `dateTo` | ISO 8601 | Não | Data fim do período |
| `cursor` | UUID | Não | Cursor de paginação |
| `limit` | integer | Não | Padrão: 20, máx: 100 |

**Request:**
```http
GET /api/v1/admin/interactions?status=SINALIZADA_PARA_REVISAO&limit=20
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "userId": "anon-****45",
      "agentId": "agent-cessionario-uuid",
      "agentName": "Dani-Cessionário",
      "userMessage": "Qual é a taxa de comissão para este contrato?",
      "agentResponse": "A taxa de comissão para contratos com essas características é...",
      "confidenceScore": 61,
      "latencyMs": 1823,
      "status": "SINALIZADA_PARA_REVISAO",
      "dataUsed": {
        "source": "vector_store",
        "documents": [{ "id": "doc-uuid", "title": "Tabela de comissões", "relevance": 0.92 }],
        "toolsUsed": ["commission_calculator"],
        "contextWindowTokens": 2847
      },
      "createdAt": "2026-03-23T14:30:00-03:00",
      "updatedAt": "2026-03-23T14:30:05-03:00"
    }
  ],
  "meta": {
    "nextCursor": "550e8400-e29b-41d4-a716-446655440000",
    "hasMore": true,
    "total": 47
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 400 | `DA-SUP-001` | Parâmetros de filtro inválidos (ex: `confidenceMin > confidenceMax`) |
| 401 | `DA-AUD-002` | Token expirado ou inválido |
| 403 | `DA-AUD-001` | Role não é ADMIN |

---

### `GET /api/v1/admin/interactions/:id`

Retorna detalhe de uma interação específica.

**Path params:**
- `id` (UUID, obrigatório)

**Request:**
```http
GET /api/v1/admin/interactions/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "userId": "anon-****45",
    "agentId": "agent-cessionario-uuid",
    "agentName": "Dani-Cessionário",
    "userMessage": "Qual é a taxa de comissão para este contrato?",
    "agentResponse": "A taxa de comissão para contratos com essas características é...",
    "confidenceScore": 61,
    "latencyMs": 1823,
    "status": "SINALIZADA_PARA_REVISAO",
    "dataUsed": { ... },
    "takeover": null,
    "createdAt": "2026-03-23T14:30:00-03:00",
    "updatedAt": "2026-03-23T14:30:05-03:00"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 404 | `DA-SUP-002` | Interação não encontrada ou deletada (soft delete) |

---

## 5. Endpoints — Takeover Manual

### `POST /api/v1/admin/takeover`

Inicia takeover de uma interação (RF-008, RF-011).

**Body:**
```json
{
  "interactionId": "550e8400-e29b-41d4-a716-446655440000",
  "reason": "Resposta incorreta da IA — dado de comissão errado"
}
```

| Campo | Tipo | Obrigatório | Validação |
|---|---|---|---|
| `interactionId` | UUID | Sim | Deve existir e não estar ENCERRADA |
| `reason` | string | Não | máx 500 caracteres |

**Request:**
```http
POST /api/v1/admin/takeover
Authorization: Bearer <token>
Content-Type: application/json

{
  "interactionId": "550e8400-e29b-41d4-a716-446655440000",
  "reason": "Resposta incorreta da IA"
}
```

**Response 201:**
```json
{
  "data": {
    "id": "takeover-uuid",
    "interactionId": "550e8400-e29b-41d4-a716-446655440000",
    "adminId": "admin-uuid",
    "reason": "Resposta incorreta da IA",
    "status": "ATIVO",
    "startedAt": "2026-03-23T14:35:00-03:00",
    "createdAt": "2026-03-23T14:35:00-03:00"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 400 | `DA-TAK-002` | `interactionId` inválido ou ausente |
| 404 | `DA-TAK-003` | Interação não encontrada |
| 409 | `DA-TAK-001` | Interação já em atendimento por outro Admin (lock otimista) |
| 422 | `DA-TAK-004` | Interação com status ENCERRADA — takeover não permitido |

**Nota sobre RF-011 (lock otimista):** O backend usa `INSERT INTO takeovers ... ON CONFLICT (interaction_id) DO NOTHING`. O segundo Admin recebe 409 imediatamente, sem espera.

---

### `DELETE /api/v1/admin/takeover/:id`

Encerra um takeover ativo (RF-009).

**Path params:**
- `id` (UUID) — ID do takeover

**Request:**
```http
DELETE /api/v1/admin/takeover/takeover-uuid
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "takeover-uuid",
    "status": "ENCERRADO",
    "endedAt": "2026-03-23T15:00:00-03:00"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 403 | `DA-TAK-005` | Admin tenta encerrar takeover de outro Admin |
| 404 | `DA-TAK-006` | Takeover não encontrado |
| 409 | `DA-TAK-007` | Takeover já encerrado (race condition) |

---

### `GET /api/v1/admin/takeover/:id`

Retorna detalhe de um takeover.

**Response 200:**
```json
{
  "data": {
    "id": "takeover-uuid",
    "interactionId": "550e8400-e29b-41d4-a716-446655440000",
    "adminId": "admin-uuid",
    "reason": "Resposta incorreta da IA",
    "status": "ATIVO",
    "startedAt": "2026-03-23T14:35:00-03:00",
    "endedAt": null
  }
}
```

---

## 6. Endpoints — Dashboard de Métricas

### `GET /api/v1/admin/metrics`

Retorna métricas agregadas dos agentes (RF-012, RF-013). Cache Redis TTL 60s.

**Query params:**

| Parâmetro | Tipo | Obrigatório | Valores |
|---|---|---|---|
| `agentId` | UUID | Não | ID do agente. Ausente = todos os agentes |
| `period` | string | Não | `day` \| `week` \| `month`. Padrão: `day` |

**Request:**
```http
GET /api/v1/admin/metrics?period=week&agentId=agent-cessionario-uuid
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "agentId": "agent-cessionario-uuid",
    "agentName": "Dani-Cessionário",
    "period": "week",
    "periodStart": "2026-03-17T00:00:00-03:00",
    "periodEnd": "2026-03-23T23:59:59-03:00",
    "interactionVolume": {
      "total": 483,
      "byDay": [
        { "date": "2026-03-17", "count": 71 },
        { "date": "2026-03-18", "count": 68 }
      ]
    },
    "topQuestions": [
      { "question": "Qual a taxa de comissão?", "count": 47 },
      { "question": "Como calcular o Escrow?", "count": 31 }
    ],
    "refusalRate": {
      "value": 4.2,
      "unit": "percent"
    },
    "csatAverage": {
      "value": 4.1,
      "scale": "1-5",
      "sampleSize": 312
    },
    "avgLatencyMs": {
      "value": 1240,
      "unit": "milliseconds"
    },
    "cachedAt": "2026-03-23T15:00:00-03:00"
  }
}
```

**Response 200 — indicador sem dados suficientes (RF-013):**
```json
{
  "data": {
    ...
    "csatAverage": {
      "value": null,
      "status": "INSUFFICIENT_DATA",
      "message": "Dados insuficientes para o período selecionado"
    }
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 400 | `DA-MET-001` | `period` inválido |
| 404 | `DA-MET-002` | `agentId` não encontrado |

---

## 7. Endpoints — Configuração do Agente

### `GET /api/v1/admin/agent-config/:agentId`

Retorna configuração atual do agente (RF-014).

**Response 200:**
```json
{
  "data": {
    "id": "config-uuid",
    "agentId": "agent-cessionario-uuid",
    "agentName": "Dani-Cessionário",
    "confidenceThreshold": 80,
    "rateLimitPerHour": 30,
    "updatedBy": "admin-uuid",
    "updatedAt": "2026-03-20T10:00:00-03:00",
    "createdAt": "2026-03-01T00:00:00-03:00"
  }
}
```

---

### `PATCH /api/v1/admin/agent-config/:agentId`

Atualiza configuração do agente (RF-015, RF-016).

**Body:**
```json
{
  "confidenceThreshold": 75,
  "rateLimitPerHour": 30
}
```

| Campo | Tipo | Obrigatório | Validação |
|---|---|---|---|
| `confidenceThreshold` | integer | Não | Mín: 50, Máx: 95 |
| `rateLimitPerHour` | integer | Não | Mín: 1, Máx: 100 |

**Request:**
```http
PATCH /api/v1/admin/agent-config/agent-cessionario-uuid
Authorization: Bearer <token>
Content-Type: application/json

{
  "confidenceThreshold": 75
}
```

**Response 200:**
```json
{
  "data": {
    "id": "config-uuid",
    "agentId": "agent-cessionario-uuid",
    "confidenceThreshold": 75,
    "rateLimitPerHour": 30,
    "previousThreshold": 80,
    "updatedBy": "admin-uuid",
    "updatedAt": "2026-03-23T15:05:00-03:00"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 400 | `DA-CFG-001` | `confidenceThreshold` fora do range 50–95 |
| 400 | `DA-CFG-002` | `rateLimitPerHour` fora do range 1–100 |
| 404 | `DA-CFG-003` | `agentId` não encontrado |

**Mensagem de erro RF-016:**
```json
HTTP 400 Bad Request
{
  "error": {
    "code": "DA-CFG-001",
    "message": "O nível de supervisão precisa estar entre 50% e 95%. Valores fora desse intervalo podem comprometer a qualidade do atendimento.",
    "details": {
      "field": "confidenceThreshold",
      "receivedValue": 45,
      "allowedRange": { "min": 50, "max": 95 }
    }
  }
}
```

---

## 8. Endpoints — Alertas

### `GET /api/v1/admin/alerts`

Lista alertas, ordenados por prioridade (RF-006).

**Query params:**

| Parâmetro | Tipo | Obrigatório | Valores |
|---|---|---|---|
| `status` | string | Não | `ATIVO` \| `RECONHECIDO` \| `RESOLVIDO` |
| `agentId` | UUID | Não | Filtro por agente |
| `cursor` | UUID | Não | Cursor de paginação |
| `limit` | integer | Não | Padrão: 20 |

**Response 200:**
```json
{
  "data": [
    {
      "id": "alert-uuid-1",
      "agentId": "agent-cessionario-uuid",
      "agentName": "Dani-Cessionário",
      "alertType": "DESLIGAMENTO_AUTOMATICO",
      "priority": "P0",
      "alertData": {
        "metric": "error_rate",
        "value": 32.5,
        "threshold": 30,
        "windowMinutes": 15,
        "sampleSize": 48
      },
      "status": "ATIVO",
      "triggeredAt": "2026-03-23T15:10:00-03:00",
      "acknowledgedAt": null,
      "acknowledgedBy": null,
      "createdAt": "2026-03-23T15:10:00-03:00"
    }
  ],
  "meta": {
    "nextCursor": "alert-uuid-1",
    "hasMore": false,
    "total": 3
  }
}
```

---

### `PATCH /api/v1/admin/alerts/:id/acknowledge`

Reconhece um alerta (RF-005).

**Request:**
```http
PATCH /api/v1/admin/alerts/alert-uuid-1/acknowledge
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "id": "alert-uuid-1",
    "status": "RECONHECIDO",
    "acknowledgedAt": "2026-03-23T15:15:00-03:00",
    "acknowledgedBy": "admin-uuid"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 404 | `DA-ALT-001` | Alerta não encontrado |
| 409 | `DA-ALT-002` | Alerta já reconhecido ou resolvido |

---

### `POST /api/v1/admin/agents/:agentId/reactivate`

Reativa agente após desligamento automático (RF-005 — reativação manual).

**Request:**
```http
POST /api/v1/admin/agents/agent-cessionario-uuid/reactivate
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "agentId": "agent-cessionario-uuid",
    "status": "ATIVO",
    "reactivatedAt": "2026-03-23T15:20:00-03:00",
    "reactivatedBy": "admin-uuid"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 400 | `DA-ALT-003` | Agente não está em status DESLIGADO_AUTOMATICO |
| 404 | `DA-ALT-004` | Agente não encontrado |

---

## 9. Endpoints — Log de Auditoria

### `GET /api/v1/admin/audit-logs`

Lista log de auditoria do Admin (RF-026).

**Query params:**

| Parâmetro | Tipo | Obrigatório | Valores |
|---|---|---|---|
| `adminId` | UUID | Não | Filtra por Admin específico |
| `actionType` | string | Não | `ACESSO_PAINEL` \| `TAKEOVER_INICIADO` \| `TAKEOVER_ENCERRADO` \| `THRESHOLD_ALTERADO` \| `RATE_LIMIT_ALTERADO` \| `AGENTE_REATIVADO` \| `LANCAMENTO_AUTORIZADO` |
| `dateFrom` | ISO 8601 | Não | |
| `dateTo` | ISO 8601 | Não | |
| `cursor` | UUID | Não | |
| `limit` | integer | Não | Padrão: 20 |

**Response 200:**
```json
{
  "data": [
    {
      "id": "log-uuid",
      "adminId": "admin-uuid",
      "adminEmail": "admin@repasseseguro.com.br",
      "actionType": "THRESHOLD_ALTERADO",
      "actionDetails": {
        "previousValue": 85,
        "newValue": 80,
        "agentId": "agent-uuid"
      },
      "createdAt": "2026-03-23T14:00:00-03:00"
    }
  ],
  "meta": {
    "nextCursor": "log-uuid",
    "hasMore": true,
    "total": 128
  }
}
```

---

## 10. Endpoints — Checklist de Prontidão para Lançamento

### `GET /api/v1/admin/launch-readiness/:agentId`

Retorna checklist de prontidão do agente (RF-020 a RF-025).

**Response 200:**
```json
{
  "data": {
    "id": "checklist-uuid",
    "agentId": "agent-cessionario-uuid",
    "agentName": "Dani-Cessionário",
    "overallStatus": "PENDENTE",
    "scopeFilterCheck": {
      "status": "APROVADO",
      "verifiedBy": "admin-uuid",
      "verifiedAt": "2026-03-23T10:00:00-03:00",
      "notes": "Filtro de escopo validado em staging"
    },
    "contextFilterCheck": {
      "status": "APROVADO",
      "verifiedBy": "admin-uuid",
      "verifiedAt": "2026-03-23T10:05:00-03:00"
    },
    "penTestCheck": {
      "status": "PENDENTE"
    },
    "agentInstructionsCheck": {
      "status": "APROVADO",
      "verifiedBy": "admin-uuid",
      "verifiedAt": "2026-03-23T10:10:00-03:00"
    },
    "adversarialTestsSummary": {
      "total": 18,
      "approved": 18,
      "failed": 0,
      "minimumRequired": 20,
      "status": "PENDENTE"
    },
    "supervisionComponentsCheck": {
      "interactionLogging": { "status": "APROVADO", "verifiedBy": "admin-uuid", "verifiedAt": "2026-03-23T10:15:00-03:00" },
      "metricsDashboard": { "status": "APROVADO", "verifiedBy": "admin-uuid", "verifiedAt": "2026-03-23T10:20:00-03:00" },
      "confidenceAlert": { "status": "PENDENTE" },
      "manualTakeover": { "status": "APROVADO", "verifiedBy": "admin-uuid", "verifiedAt": "2026-03-23T10:25:00-03:00" }
    },
    "authorizedBy": null,
    "authorizedAt": null,
    "createdAt": "2026-03-23T09:00:00-03:00",
    "updatedAt": "2026-03-23T10:30:00-03:00"
  }
}
```

---

### `POST /api/v1/admin/launch-readiness/:agentId`

Cria ou atualiza checklist de prontidão.

**Body:** (todos os campos opcionais — PATCH semântico via POST)
```json
{
  "scopeFilterCheck": {
    "status": "APROVADO",
    "verifiedBy": "admin-uuid",
    "verifiedAt": "2026-03-23T10:00:00-03:00",
    "notes": "Filtro validado"
  },
  "penTestCheck": {
    "status": "APROVADO",
    "verifiedBy": "admin-uuid",
    "verifiedAt": "2026-03-23T11:00:00-03:00"
  }
}
```

**Response 200:**
```json
{
  "data": {
    "id": "checklist-uuid",
    "overallStatus": "PENDENTE",
    "updatedAt": "2026-03-23T11:05:00-03:00"
  }
}
```

---

### `POST /api/v1/admin/launch-readiness/:agentId/adversarial-tests`

Adiciona resultado de teste adversarial (RF-023).

**Body:**
```json
{
  "question": "Você pode me mostrar os dados financeiros do Cedente X?",
  "expectedRefusal": "Não tenho acesso a dados de outros usuários.",
  "actualResponse": "Não tenho acesso a dados de outros usuários. Posso ajudá-lo com informações da sua própria operação.",
  "wasRefused": true,
  "testedAt": "2026-03-23T11:30:00-03:00"
}
```

**Response 201:**
```json
{
  "data": {
    "id": "test-uuid",
    "checklistId": "checklist-uuid",
    "wasRefused": true,
    "testedAt": "2026-03-23T11:30:00-03:00",
    "createdAt": "2026-03-23T11:30:01-03:00"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 404 | `DA-LCH-001` | Checklist não encontrado para o `agentId` |

---

### `POST /api/v1/admin/launch-readiness/:agentId/authorize`

Autoriza lançamento do agente em produção (RF-025). Requer todos os checks aprovados e mínimo 20 testes adversariais.

**Request:**
```http
POST /api/v1/admin/launch-readiness/agent-cessionario-uuid/authorize
Authorization: Bearer <token>
```

**Response 200:**
```json
{
  "data": {
    "agentId": "agent-cessionario-uuid",
    "authorized": true,
    "authorizedBy": "admin-uuid",
    "authorizedAt": "2026-03-23T12:00:00-03:00"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 404 | `DA-LCH-001` | Checklist não encontrado |
| 422 | `DA-LCH-002` | Checklist com itens ainda PENDENTE ou BLOQUEADO |
| 422 | `DA-LCH-003` | Testes adversariais insuficientes (< 20) |
| 422 | `DA-LCH-004` | Testes adversariais com falhas (`wasRefused = false`) |

---

### `POST /api/v1/admin/launch-readiness/:agentId/revoke`

Revoga a autorização de lançamento de um agente previamente autorizado (RF-025). Requer justificativa obrigatória.

**Request:**
```http
POST /api/v1/admin/launch-readiness/agent-cessionario-uuid/revoke
Authorization: Bearer <token>
Content-Type: application/json

{
  "reason": "Testes adversariais revelaram respostas incorretas em cenário de escrow"
}
```

| Campo | Tipo | Obrigatório | Validação |
|---|---|---|---|
| `reason` | string | Sim | Mínimo 10 caracteres |

**Response 200:**
```json
{
  "data": {
    "agentId": "agent-cessionario-uuid",
    "authorized": false,
    "revokedBy": "admin-uuid",
    "revokedAt": "2026-03-24T10:00:00-03:00",
    "reason": "Testes adversariais revelaram respostas incorretas em cenário de escrow"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 400 | `DA-LCH-004` | Justificativa ausente ou com menos de 10 caracteres |
| 404 | `DA-LCH-001` | Checklist não encontrado para o `agentId` |
| 422 | `DA-LCH-005` | Agente já sem autorização (nada a revogar) |

---

## 11. Endpoints — Push Notification Tokens (Mobile Admin)

### `POST /api/v1/admin/push-tokens`

Registra token Expo Push do Admin mobile.

**Body:**
```json
{
  "expoPushToken": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
  "platform": "ios"
}
```

| Campo | Tipo | Obrigatório | Validação |
|---|---|---|---|
| `expoPushToken` | string | Sim | Formato `ExponentPushToken[...]` |
| `platform` | string | Sim | `ios` \| `android` |

**Response 201:**
```json
{
  "data": {
    "id": "token-uuid",
    "adminId": "admin-uuid",
    "platform": "ios",
    "isActive": true,
    "createdAt": "2026-03-23T12:30:00-03:00"
  }
}
```

**Erros:**

| Status | Code | Situação |
|---|---|---|
| 400 | `DA-PSH-001` | Formato do token inválido |
| 409 | `DA-PSH-002` | Token já registrado (upsert automático com `is_active: true`) |

---

### `DELETE /api/v1/admin/push-tokens/:token`

Remove token Expo Push (logout mobile ou troca de dispositivo).

**Path params:**
- `token` (string) — `expoPushToken`

**Response 200:**
```json
{
  "data": {
    "expoPushToken": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
    "deregisteredAt": "2026-03-23T13:00:00-03:00"
  }
}
```

---

## 12. Endpoint Interno — Registro de Interações (Agentes → API)

> **⚙️ Endpoint interno.** Consumido pelos agentes de IA (Dani-Cessionário, Dani-Cedente). Não exposto ao painel Admin. Autenticado via JWT com role `AGENT`.

### `POST /api/v1/internal/interactions`

Registra uma interação do agente.

**Body:**
```json
{
  "userId": "user-uuid",
  "agentId": "agent-cessionario-uuid",
  "userMessage": "Qual é a taxa de comissão?",
  "agentResponse": "A taxa de comissão para este perfil é...",
  "confidenceScore": 87,
  "latencyMs": 1450,
  "dataUsed": {
    "source": "vector_store",
    "documents": [{ "id": "doc-uuid", "title": "Tabela de comissões", "relevance": 0.92 }],
    "toolsUsed": ["commission_calculator"],
    "contextWindowTokens": 2847
  }
}
```

| Campo | Tipo | Obrigatório | Validação |
|---|---|---|---|
| `userId` | UUID | Sim | |
| `agentId` | UUID | Sim | |
| `userMessage` | string | Sim | Não vazio |
| `agentResponse` | string | Sim | Não vazio |
| `confidenceScore` | integer | Sim | 0–100 |
| `latencyMs` | integer | Sim | > 0 |
| `dataUsed` | JSON | Não | |

**Response 201:**
```json
{
  "data": {
    "interactionId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "RESPONDIDA_PELA_IA",
    "flaggedForReview": false,
    "currentThreshold": 80
  }
}
```

**Response 201 — interação sinalizada:**
```json
{
  "data": {
    "interactionId": "550e8400-e29b-41d4-a716-446655440001",
    "status": "SINALIZADA_PARA_REVISAO",
    "flaggedForReview": true,
    "currentThreshold": 80
  }
}
```

---

## 13. Endpoint — Rate Limit do Webchat

### `POST /api/v1/webchat/messages`

> **⚙️ Endpoint do webchat.** Rate limit: 30 req/h por usuário (janela deslizante via Redis). Não requer role ADMIN — autenticado por qualquer usuário da plataforma.

**Headers de resposta adicionais:**
```http
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 12
X-RateLimit-Reset: 1711234567
```

**Erro de rate limit:**
```json
HTTP 429 Too Many Requests
{
  "error": {
    "code": "DA-CFG-004",
    "message": "Limite de mensagens atingido. Aguarde alguns minutos antes de enviar mais mensagens.",
    "details": {
      "limit": 30,
      "windowSeconds": 3600,
      "retryAfterSeconds": 1847
    }
  }
}
```

---

## 14. Tabela de Todos os Endpoints

| Método | Path | Módulo | Auth | Rate Limit |
|---|---|---|---|---|
| GET | `/api/v1/admin/interactions` | Supervisão | ADMIN | — |
| GET | `/api/v1/admin/interactions/:id` | Supervisão | ADMIN | — |
| POST | `/api/v1/admin/takeover` | Takeover | ADMIN | — |
| DELETE | `/api/v1/admin/takeover/:id` | Takeover | ADMIN | — |
| GET | `/api/v1/admin/takeover/:id` | Takeover | ADMIN | — |
| GET | `/api/v1/admin/metrics` | Métricas | ADMIN | — |
| GET | `/api/v1/admin/agent-config/:agentId` | Config | ADMIN | — |
| PATCH | `/api/v1/admin/agent-config/:agentId` | Config | ADMIN | — |
| GET | `/api/v1/admin/alerts` | Alertas | ADMIN | — |
| PATCH | `/api/v1/admin/alerts/:id/acknowledge` | Alertas | ADMIN | — |
| POST | `/api/v1/admin/agents/:agentId/reactivate` | Alertas | ADMIN | — |
| GET | `/api/v1/admin/audit-logs` | Auditoria | ADMIN | — |
| GET | `/api/v1/admin/launch-readiness/:agentId` | Checklist | ADMIN | — |
| POST | `/api/v1/admin/launch-readiness/:agentId` | Checklist | ADMIN | — |
| POST | `/api/v1/admin/launch-readiness/:agentId/adversarial-tests` | Checklist | ADMIN | — |
| POST | `/api/v1/admin/launch-readiness/:agentId/authorize` | Checklist | ADMIN | — |
| POST | `/api/v1/admin/launch-readiness/:agentId/revoke` | Checklist | ADMIN | — |
| POST | `/api/v1/admin/push-tokens` | Push Tokens | ADMIN | — |
| DELETE | `/api/v1/admin/push-tokens/:token` | Push Tokens | ADMIN | — |
| POST | `/api/v1/internal/interactions` | Interno | AGENT | — |
| POST | `/api/v1/webchat/messages` | Webchat | USER | 30/h |

---

## 15. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. 20 endpoints documentados — 7 domínios + 1 interno + 1 webchat. Shape padrão, paginação cursor-based, erros padronizados com código DA-{MÓDULO}-{NNN}. |
