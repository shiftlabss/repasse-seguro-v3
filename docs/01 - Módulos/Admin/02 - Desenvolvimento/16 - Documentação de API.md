# 16 - Documentação de API

## Repasse Seguro — API NestJS

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend Lead, Frontend Lead, QA |
| **Escopo** | Contratos de endpoints REST — rotas, métodos, DTOs, respostas e status codes |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Dependências** | D01 RN · D02 Stacks · D12 ERD · D13 Schema Prisma · D14 Especificações Técnicas |

---

> 📌 **TL;DR**
>
> - **Base URL:** `https://api.repasseseguro.com.br/v1`
> - **Autenticação:** JWT Bearer em todos os endpoints (exceto `/auth/login` e `/auth/forgot-password`).
> - **Padrão de resposta:** `{ data: T, meta?: { page, total, per_page } }` para sucesso; RFC 7807 para erros.
> - **11 grupos de endpoints** documentados: auth, users, cedentes, cessionarios, cases, formalization, escrow, commission, ai-supervision, reports, configs.
> - **RBAC declarado** por endpoint — role mínimo.
> - **Versioning:** prefixo `/v1/` — novas versões são aditivas, nunca breaking sem deprecation notice.

---

## 1. Convenções Globais

### 1.1 Autenticação

```http
Authorization: Bearer <access_token>
Content-Type: application/json
```

- Access token: JWT HS256, expira em 1h.
- Refresh token: rotacionar via `POST /auth/refresh`.
- Token expirado retorna `401 Unauthorized` com `WWW-Authenticate: Bearer error="invalid_token"`.

### 1.2 Padrão de Resposta — Sucesso

```json
// Recurso único
{ "data": { "id": "uuid", ... } }

// Listagem paginada
{
  "data": [...],
  "meta": {
    "page": 1,
    "per_page": 25,
    "total": 120,
    "total_pages": 5
  }
}
```

### 1.3 Padrão de Resposta — Erro (RFC 7807)

```json
{
  "type": "https://api.repasseseguro.com.br/errors/validation",
  "title": "Validation Failed",
  "status": 422,
  "detail": "O campo 'cpf' deve ser um CPF válido",
  "instance": "/v1/cedentes",
  "errors": [{ "field": "cpf", "message": "CPF inválido" }]
}
```

### 1.4 Status Codes Utilizados

| Code | Uso |
|---|---|
| `200 OK` | Leitura ou atualização bem-sucedida |
| `201 Created` | Criação bem-sucedida |
| `202 Accepted` | Operação aceita para processamento assíncrono |
| `204 No Content` | Deleção bem-sucedida (soft delete) |
| `400 Bad Request` | Erro de cliente — payload mal formado |
| `401 Unauthorized` | Token ausente, inválido ou expirado |
| `403 Forbidden` | Token válido mas sem permissão (RBAC) |
| `404 Not Found` | Recurso não encontrado |
| `409 Conflict` | Conflito — Optimistic Lock (version mismatch) |
| `422 Unprocessable Entity` | Dados válidos mas regra de negócio violada |
| `423 Locked` | Conta bloqueada por tentativas de login |
| `429 Too Many Requests` | Rate limit excedido |
| `500 Internal Server Error` | Erro não esperado no servidor |
| `502 Bad Gateway` | Falha na integração externa (ZapSign, Celcoin) |

### 1.5 Paginação

```http
GET /v1/cases?page=1&per_page=25&status=EM_TRIAGEM&sort=created_at&order=desc
```

Parâmetros padrão: `page=1`, `per_page=25`, máximo `per_page=100`.

---

## 2. Endpoints — Autenticação

### `POST /v1/auth/login`
**Role:** Público
```json
// Request
{ "email": "string", "password": "string" }

// Response 200 — 2FA desabilitado
{ "data": { "access_token": "jwt", "refresh_token": "jwt", "user": { "id", "name", "role" } } }

// Response 200 — 2FA habilitado
{ "data": { "requires_2fa": true, "temp_token": "jwt_5min" } }

// Response 401 — credenciais inválidas
// Response 423 — conta bloqueada: { "locked_until": "2026-03-22T10:30:00Z" }
```

### `POST /v1/auth/verify-2fa`
**Role:** Sessão intermediária com `temp_token`
```json
// Request
{ "temp_token": "string", "totp_code": "string (6 dígitos)" }

// Response 200
{ "data": { "access_token": "jwt", "refresh_token": "jwt" } }

// Response 401 — código inválido ou expirado
```

### `POST /v1/auth/refresh`
**Role:** Público (com refresh_token válido)
```json
// Request
{ "refresh_token": "string" }

// Response 200
{ "data": { "access_token": "jwt", "refresh_token": "jwt" } }
```

### `POST /v1/auth/logout`
**Role:** Qualquer autenticado
```
// Response 204 No Content — invalida token no Redis blacklist
```

### `POST /v1/auth/forgot-password`
**Role:** Público
```json
// Request
{ "email": "string" }

// Response 200 — sempre (não revelar se e-mail existe)
{ "data": { "message": "Se o e-mail existir, você receberá um link." } }
```

### `POST /v1/auth/reset-password`
**Role:** Token de reset válido
```json
// Request
{ "token": "string", "password": "string", "password_confirmation": "string" }

// Response 200
// Response 400 — token inválido ou senhas não coincidem
```

---

## 3. Endpoints — Usuários (Operadores)

### `GET /v1/users`
**Role:** COORDENADOR
**Params:** `page`, `per_page`, `role`, `is_active`, `search` (nome/e-mail)
```json
// Response 200 — listagem paginada com RoleBadge data
{ "data": [{ "id", "email", "name", "role", "is_active", "created_at" }], "meta": {} }
```

### `POST /v1/users`
**Role:** MASTER
```json
// Request
{ "email": "string", "name": "string", "role": "ANALISTA|COORDENADOR|GESTOR_FINANCEIRO|MASTER" }

// Response 201
// Response 409 — e-mail já cadastrado
```

### `GET /v1/users/:id`
**Role:** COORDENADOR

### `PATCH /v1/users/:id`
**Role:** MASTER
```json
// Request (campos opcionais)
{ "name": "string?", "role": "UserRole?", "is_active": "boolean?", "version": "number" }

// Response 409 — version mismatch (Optimistic Lock)
```

### `DELETE /v1/users/:id`
**Role:** MASTER
```
// Response 204 — soft delete (is_active = false)
// Response 403 — tentativa de deletar o próprio usuário
```

---

## 4. Endpoints — Cedentes

### `GET /v1/cedentes`
**Role:** ANALISTA
**Params:** `page`, `per_page`, `status`, `search` (nome/CPF)

### `POST /v1/cedentes`
**Role:** ANALISTA
```json
// Request
{ "name": "string", "cpf": "string (000.000.000-00)", "email": "string", "phone": "string?", "address": "string?" }

// Response 201
// Response 409 — CPF ou e-mail já cadastrado
```

### `GET /v1/cedentes/:id`
**Role:** ANALISTA

### `PATCH /v1/cedentes/:id`
**Role:** ANALISTA
```json
// Request inclui "version": number para Optimistic Lock
```

### `PATCH /v1/cedentes/:id/suspend`
**Role:** COORDENADOR
```json
// Request
{ "suspension_reason": "string (min 20 chars)" }
```

---

## 5. Endpoints — Cessionários

Estrutura idêntica a `/cedentes`. Substituir `cedentes` por `cessionarios` em todas as rotas.

---

## 6. Endpoints — Casos

### `GET /v1/cases`
**Role:** ANALISTA
**Params:** `page`, `per_page`, `status`, `scenario`, `assigned_analyst_id`, `sort`, `order`, `search`
```json
// Response — inclui CaseStatusBadge data, cedente.name, assigned_analyst.name
```

### `POST /v1/cases`
**Role:** ANALISTA
```json
// Request
{
  "cedente_id": "uuid",
  "scenario": "A|B|C|D",
  "property_address": "string",
  "contract_table_value": "decimal (string)",
  "paid_amount": "decimal (string)",
  "assigned_analyst_id": "uuid?"
}

// Response 201 — cria também case_config_snapshot, formalization_criteria, escrow_account
```

### `GET /v1/cases/:id`
**Role:** ANALISTA
```json
// Response — case completo com: config_snapshot, formalization_criteria, escrow_account summary
```

### `PATCH /v1/cases/:id`
**Role:** ANALISTA
```json
// Request — campos editáveis + version
{ "property_address"?: "string", "current_table_value"?: "decimal", "version": "number" }
```

### `PATCH /v1/cases/:id/status`
**Role:** ANALISTA (avanços simples) / COORDENADOR (cancelamento, bloqueio)
```json
// Request
{ "to_status": "CaseStatus", "reason": "string?", "version": "number" }

// Response 422 — transição de status inválida
```

### `PATCH /v1/cases/:id/assign`
**Role:** COORDENADOR
```json
// Request
{ "analyst_id": "uuid", "version": "number" }
```

### `PATCH /v1/cases/:id/close`
**Role:** ANALISTA
```
// Response 422 — critérios de fechamento incompletos
// Response 200 — caso avança para FECHAMENTO, jobs assíncronos disparados
```

### `GET /v1/cases/:id/status-history`
**Role:** ANALISTA

### `GET /v1/cases/:id/config-snapshot`
**Role:** ANALISTA

---

## 7. Endpoints — Dossiê e Documentos

### `GET /v1/cases/:id/dossie`
**Role:** ANALISTA

### `POST /v1/cases/:id/dossie`
**Role:** ANALISTA
```json
// Request: multipart/form-data
{ "document_type": "DocumentType", "file": "File (pdf/jpg/png max 10MB)" }
```

### `PATCH /v1/cases/:id/dossie/:doc_id/approve`
**Role:** COORDENADOR

### `PATCH /v1/cases/:id/dossie/:doc_id/reject`
**Role:** COORDENADOR
```json
// Request
{ "rejection_reason": "string (min 20 chars)" }
```

---

## 8. Endpoints — Propostas e Contrapropostas

### `GET /v1/cases/:id/proposals`
**Role:** ANALISTA

### `POST /v1/cases/:id/proposals`
**Role:** ANALISTA
```json
// Request
{ "cessionario_id": "uuid", "proposed_value": "decimal", "expires_at": "ISO8601?" }
```

### `PATCH /v1/cases/:id/proposals/:proposal_id/accept`
**Role:** ANALISTA

### `PATCH /v1/cases/:id/proposals/:proposal_id/reject`
**Role:** ANALISTA

### `GET /v1/cases/:id/proposals/:proposal_id/counterproposals`
**Role:** ANALISTA

### `POST /v1/cases/:id/proposals/:proposal_id/counterproposals`
**Role:** ANALISTA
```json
// Request
{ "counter_value": "decimal", "reason": "string?" }
```

---

## 9. Endpoints — Formalização (ZapSign e Critérios)

### `GET /v1/cases/:id/formalization`
**Role:** ANALISTA

### `PATCH /v1/cases/:id/formalization`
**Role:** ANALISTA
```json
// Request — atualiza critérios editáveis
{
  "annuence_ok": "boolean?",
  "annuence_na": "boolean?",
  "annuence_protocol": "string?",
  "delta_approved": "boolean? (COORDENADOR only)"
}
```

### `GET /v1/cases/:id/zapsign`
**Role:** ANALISTA

### `POST /v1/cases/:id/zapsign`
**Role:** ANALISTA
```json
// Request
{
  "document_type": "EnvelopeDocumentType",
  "signatories": [{ "name": "string", "email": "string", "cpf": "string?" }],
  "expires_at": "ISO8601?"
}
```

### `POST /v1/cases/:id/zapsign/:envelope_id/resend`
**Role:** ANALISTA

---

## 10. Endpoints — Escrow e Financeiro

### `GET /v1/cases/:id/escrow`
**Role:** GESTOR_FINANCEIRO

### `GET /v1/cases/:id/escrow/transactions`
**Role:** GESTOR_FINANCEIRO
**Params:** `page`, `per_page`, `type`

### `POST /v1/cases/:id/escrow/distribute`
**Role:** GESTOR_FINANCEIRO
```
// Response 202 — distribuição iniciada assincronamente
// Response 403 — distribuição bloqueada
// Response 422 — depósito não confirmado
```

### `PATCH /v1/cases/:id/escrow/block`
**Role:** GESTOR_FINANCEIRO
```json
// Request
{ "block_reason": "string (min 20 chars)" }
```

### `PATCH /v1/cases/:id/escrow/unblock`
**Role:** MASTER
```json
// Request — Master aprova desbloqueio
{ "approval_note": "string?" }
```

### `GET /v1/cases/:id/commission`
**Role:** GESTOR_FINANCEIRO

### `GET /v1/cases/:id/distributions`
**Role:** GESTOR_FINANCEIRO

---

## 11. Endpoints — Supervisão IA

### `GET /v1/ai/decisions`
**Role:** COORDENADOR
**Params:** `page`, `per_page`, `agent_name`, `outcome`, `case_id`, `from`, `to`

### `GET /v1/ai/decisions/:id`
**Role:** COORDENADOR

### `PATCH /v1/ai/decisions/:id/review`
**Role:** COORDENADOR
```json
// Request
{ "outcome": "APROVADO_OPERADOR|REJEITADO_OPERADOR", "operator_feedback": "string?" }
```

### `GET /v1/ai/agents`
**Role:** MASTER

### `PATCH /v1/ai/agents/:agent_name`
**Role:** MASTER
```json
// Request
{ "enabled": "boolean", "confidence_threshold": "decimal (0-1)" }
```

---

## 12. Endpoints — Relatórios

### `GET /v1/reports/operational`
**Role:** COORDENADOR
**Params:** `from`, `to`, `analyst_id`, `status`, `scenario`

### `GET /v1/reports/analyst-performance`
**Role:** COORDENADOR
**Params:** `from`, `to`, `analyst_id`

### `POST /v1/reports/export`
**Role:** COORDENADOR
```json
// Request
{ "type": "operational|financial|analyst", "format": "csv|pdf", "from": "ISO8601", "to": "ISO8601" }

// Response 202 — geração assíncrona, retorna job_id
// GET /v1/reports/export/:job_id — consultar status e URL de download
```

---

## 13. Endpoints — Configurações

### `GET /v1/configs`
**Role:** COORDENADOR

### `GET /v1/configs/:key`
**Role:** COORDENADOR

### `PATCH /v1/configs/:key`
**Role:** MASTER
```json
// Request
{ "config_value": "string", "update_reason": "string (min 10 chars)" }

// Response 200
// Response 422 — tipo de dado inválido para a chave
```

---

## 14. Endpoints — Webhooks (Entrada)

> ⚙️ Endpoints de webhook são verificados por assinatura HMAC-SHA256. Sem JWT.

### `POST /v1/webhooks/zapsign`
**Autenticação:** Header `X-ZapSign-Token` verificado contra secret configurado
```json
// Payload ZapSign
{ "token": "string", "event": "SIGNED|REFUSED|EXPIRED", "signer": {...} }

// Response 200 — enfileirado no RabbitMQ para processamento assíncrono
```

### `POST /v1/webhooks/celcoin`
**Autenticação:** Header `X-Celcoin-Signature` verificado por HMAC
```json
// Payload Celcoin
{ "transaction_id": "string", "status": "PAID|FAILED|REVERSED", "amount": "decimal" }
```

---

## 15. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — 11 grupos, 50+ endpoints, padrão RFC 7807, RBAC por endpoint, paginação, Optimistic Lock e webhooks. |
