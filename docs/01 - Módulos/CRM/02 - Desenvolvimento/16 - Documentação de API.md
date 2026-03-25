# 16 - Documentação de API

## Repasse Seguro — CRM API

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend Lead, Frontend Lead, QA |
| **Escopo** | Contratos de endpoints REST do CRM — rotas, métodos, DTOs, respostas e status codes |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | Doc 01.1–01.5 Regras de Negócio CRM · Doc 02 Stacks CRM · Doc 15 Arquitetura de Pastas |

---

> **TL;DR**
>
> - **Base URL:** `https://api.repasseseguro.com.br/api/v1/crm`
> - **Autenticação:** JWT Bearer em todos os endpoints (exceto `/auth/login` e `/auth/forgot-password`). Supabase Auth como provedor de identidade.
> - **Padrão de resposta:** `{ data: T, meta?: { page, total, per_page, total_pages } }` para sucesso; RFC 7807 para erros.
> - **12 grupos de endpoints** documentados: auth, team, cases, contacts, activities, communication, dossier, negotiations, commission, sla, reports, config, webhooks.
> - **RBAC declarado** por endpoint — 4 roles: Admin RS, Coordenador RS, Analista RS, Parceiro Externo.
> - **Rate limiting:** 200 req/min geral (ferramenta interna). Endpoints de auth: 10 req/min por IP.
> - **Error codes** com prefixo `CRM-`.

---

## 1. Convenções Globais

### 1.1 Autenticação

```http
Authorization: Bearer <access_token>
Content-Type: application/json
```

- Access token: JWT emitido pelo Supabase Auth, expira em 60 min (RN-003).
- Refresh: automático via `@supabase/ssr` no frontend — transparente para o usuário.
- Token expirado retorna `401 Unauthorized` com `WWW-Authenticate: Bearer error="invalid_token"`.
- Sessão de inatividade: 60 min. Renovação automática enquanto o usuário está ativo (RN-003).

### 1.2 Roles RBAC

| Role | Descrição |
|---|---|
| `ADMIN_RS` | Acesso total — configurações, equipe, relatórios, todos os Casos |
| `COORDENADOR_RS` | Supervisão — aprovações, redistribuição, relatórios |
| `ANALISTA_RS` | Operação — Casos atribuídos, contatos, atividades, comunicação |
| `PARCEIRO_EXTERNO` | Acesso restrito — visualiza Casos vinculados ao seu Parceiro (somente leitura) |

### 1.3 Padrão de Resposta — Sucesso

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

// Operação sem retorno de dados
{ "data": null }
```

### 1.4 Padrão de Resposta — Erro (RFC 7807)

```json
{
  "type": "https://api.repasseseguro.com.br/errors/crm/validation",
  "title": "Validation Failed",
  "status": 422,
  "detail": "O campo 'contact_type' deve ser um dos valores: CEDENTE, CESSIONARIO, PARCEIRO, INCORPORADORA",
  "instance": "/api/v1/crm/contacts",
  "code": "CRM-014",
  "errors": [
    { "field": "contact_type", "message": "Tipo de contato inválido" }
  ]
}
```

### 1.5 Status Codes Utilizados

| Code | Uso |
|---|---|
| `200 OK` | Leitura ou atualização bem-sucedida |
| `201 Created` | Criação bem-sucedida |
| `202 Accepted` | Operação aceita para processamento assíncrono (relatórios, notificações) |
| `204 No Content` | Deleção ou logout bem-sucedido |
| `400 Bad Request` | Payload mal formado |
| `401 Unauthorized` | Token ausente, inválido ou expirado |
| `403 Forbidden` | Token válido, mas sem permissão RBAC |
| `404 Not Found` | Recurso não encontrado |
| `409 Conflict` | Conflito de estado (Optimistic Lock — version mismatch) |
| `422 Unprocessable Entity` | Regra de negócio violada |
| `423 Locked` | Conta bloqueada por tentativas de login (RN-002) |
| `429 Too Many Requests` | Rate limit excedido |
| `500 Internal Server Error` | Erro interno não esperado |
| `502 Bad Gateway` | Falha na integração externa (ZapSign, Meta, Escrow) |

### 1.6 Paginação

```http
GET /api/v1/crm/cases?page=1&per_page=25&status=NEGOCIACAO&sort=sla_deadline&order=asc
```

Parâmetros padrão: `page=1`, `per_page=25`, máximo `per_page=100`.

### 1.7 Versionamento

Prefixo `/api/v1/` — novas versões são aditivas. Nunca breaking sem deprecation notice de 30 dias.

---

## 2. Endpoints — Autenticação (`/auth`)

> CRM usa login por e-mail/senha sem OTP — ferramenta interna (RN-193). Sem 2FA TOTP no MVP.

### `POST /api/v1/crm/auth/login`

**Role:** Público | **Rate limit:** 10 req/min por IP

```json
// Request
{ "email": "string", "password": "string" }

// Response 200 — login bem-sucedido
{
  "data": {
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "user": {
      "id": "uuid",
      "name": "string",
      "email": "string",
      "role": "ADMIN_RS | COORDENADOR_RS | ANALISTA_RS | PARCEIRO_EXTERNO"
    }
  }
}

// Response 401 — credenciais inválidas: { code: "CRM-001", detail: "E-mail ou senha incorretos" }
// Response 401 — conta inativa: { code: "CRM-002", detail: "Conta desativada. Contate o administrador." }
// Response 423 — conta bloqueada: { code: "CRM-003", locked_until: "ISO8601", detail: "Conta bloqueada por 15 minutos." }
```

### `POST /api/v1/crm/auth/refresh`

**Role:** Público (com refresh_token válido)

```json
// Request
{ "refresh_token": "string" }

// Response 200
{ "data": { "access_token": "eyJ...", "refresh_token": "eyJ..." } }

// Response 401 — token inválido ou expirado: { code: "CRM-004" }
```

### `POST /api/v1/crm/auth/logout`

**Role:** Qualquer autenticado

```
// Response 204 — sessão invalidada
```

### `POST /api/v1/crm/auth/forgot-password`

**Role:** Público | **Rate limit:** 5 req/min por IP

```json
// Request
{ "email": "string" }

// Response 200 — sempre (não revela se e-mail existe)
{ "data": { "message": "Se o e-mail existir, você receberá um link de recuperação." } }
```

### `POST /api/v1/crm/auth/reset-password`

**Role:** Token de reset válido (via Supabase Auth)

```json
// Request
{ "token": "string", "password": "string", "password_confirmation": "string" }

// Response 200
// Response 400 — token inválido: { code: "CRM-005" }
// Response 422 — senhas não coincidem: { code: "CRM-006" }
```

---

## 3. Endpoints — Equipe (`/team`)

> Usuários internos da Repasse Seguro. Não confundir com Contatos (Cedentes/Cessionários).

### `GET /api/v1/crm/team`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`
**Query params:** `page`, `per_page`, `role` (filtro), `is_active` (boolean), `search` (nome/e-mail)

```json
// Response 200
{
  "data": [
    {
      "id": "uuid",
      "name": "string",
      "email": "string",
      "role": "ANALISTA_RS",
      "is_active": true,
      "cases_count": 12,
      "created_at": "ISO8601"
    }
  ],
  "meta": { "page": 1, "per_page": 25, "total": 8, "total_pages": 1 }
}
```

### `POST /api/v1/crm/team`

**Role:** `ADMIN_RS`

```json
// Request
{
  "name": "string",
  "email": "string",
  "role": "COORDENADOR_RS | ANALISTA_RS | PARCEIRO_EXTERNO"
}

// Response 201 — convite enviado por e-mail via Supabase Auth
// Response 409 — e-mail já cadastrado: { code: "CRM-011" }
```

### `GET /api/v1/crm/team/:id`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — inclui Casos atribuídos, SLA médio, última atividade
{
  "data": {
    "id": "uuid",
    "name": "string",
    "email": "string",
    "role": "ANALISTA_RS",
    "is_active": true,
    "cases_count": 12,
    "avg_sla_days": 4.2,
    "last_activity_at": "ISO8601"
  }
}
```

### `PATCH /api/v1/crm/team/:id`

**Role:** `ADMIN_RS`

```json
// Request (campos opcionais + Optimistic Lock)
{ "name": "string?", "role": "CrmRole?", "is_active": "boolean?", "version": "number" }

// Response 200
// Response 409 — version mismatch: { code: "CRM-012" }
// Response 403 — tentativa de editar o próprio papel: { code: "CRM-013" }
```

### `DELETE /api/v1/crm/team/:id`

**Role:** `ADMIN_RS`

```
// Response 204 — soft delete (deleted_at setado)
// Response 422 — usuário tem Casos ativos: redistribua antes: { code: "CRM-014", active_cases: 5 }
```

### `POST /api/v1/crm/team/:id/redistribute`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request — redistribuir Casos de um Analista para outro(s)
{
  "target_analyst_id": "uuid",     // redistribuir todos para um Analista
  "case_assignments": [            // OU distribuição manual por Caso
    { "case_id": "uuid", "analyst_id": "uuid" }
  ]
}

// Response 200 — { redistributed_count: 5 }
// Response 422 — analista destino inativo: { code: "CRM-015" }
```

---

## 4. Endpoints — Casos (`/cases`)

### `GET /api/v1/crm/cases`

**Role:** `ANALISTA_RS` (apenas Casos atribuídos), `COORDENADOR_RS` e `ADMIN_RS` (todos os Casos)
**Query params:** `page`, `per_page`, `status` (enum), `analyst_id`, `partner_id`, `search` (RS-XXXX, empreendimento), `sort` (`created_at | sla_deadline | updated_at`), `order` (`asc | desc`)

```json
// Response 200
{
  "data": [
    {
      "id": "uuid",
      "identifier": "RS-0042",
      "status": "NEGOCIACAO",
      "cedente": { "id": "uuid", "name": "Jo** ****" },
      "empreendimento": "Residencial Park 2",
      "contract_table_value": "280000.00",
      "assigned_analyst": { "id": "uuid", "name": "Ana Lima" },
      "sla": { "status": "ATENCAO", "percent_consumed": 72, "deadline": "ISO8601" },
      "created_at": "ISO8601",
      "updated_at": "ISO8601"
    }
  ],
  "meta": { "page": 1, "per_page": 25, "total": 134, "total_pages": 6 }
}
```

### `POST /api/v1/crm/cases`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{
  "cedente_contact_id": "uuid",
  "empreendimento": "string",
  "scenario": "A | B | C | D",
  "contract_table_value": "decimal string",
  "paid_amount": "decimal string",
  "assigned_analyst_id": "uuid?"    // se omitido, fica como não atribuído
}

// Response 201 — { data: { id, identifier: "RS-0043", status: "CADASTRO", ... } }
// Response 422 — Cedente não encontrado: { code: "CRM-021" }
// Response 422 — Cenário inválido: { code: "CRM-022" }
```

### `GET /api/v1/crm/cases/:id`

**Role:** `ANALISTA_RS` (apenas Casos atribuídos), `COORDENADOR_RS`, `ADMIN_RS`, `PARCEIRO_EXTERNO` (Casos do seu Parceiro)

```json
// Response 200 — Caso completo com relacionamentos
{
  "data": {
    "id": "uuid",
    "identifier": "RS-0042",
    "status": "NEGOCIACAO",
    "scenario": "B",
    "empreendimento": "string",
    "contract_table_value": "280000.00",
    "paid_amount": "180000.00",
    "origin": "PLATAFORMA | MANUAL",
    "cedente": { "id": "uuid", "name": "Jo** ****", "phone_masked": "(85) 9****-5678" },
    "assigned_analyst": { "id": "uuid", "name": "string" },
    "sla": {
      "status": "ATENCAO",
      "percent_consumed": 72,
      "deadline": "ISO8601",
      "current_state_started_at": "ISO8601"
    },
    "dossier_summary": { "total": 8, "approved": 5, "pending": 2, "rejected": 1 },
    "active_proposal_id": "uuid?",
    "zapsign_envelope_id": "string?",
    "created_at": "ISO8601",
    "updated_at": "ISO8601",
    "version": 7
  }
}
// Response 403 — Analista tentando acessar Caso de outro: { code: "CRM-023" }
```

### `PATCH /api/v1/crm/cases/:id`

**Role:** `ANALISTA_RS` (Casos atribuídos), `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{
  "empreendimento": "string?",
  "contract_table_value": "decimal string?",
  "paid_amount": "decimal string?",
  "version": "number"
}

// Response 200
// Response 409 — version mismatch: { code: "CRM-024" }
```

### `PATCH /api/v1/crm/cases/:id/transition`

**Role:** `ANALISTA_RS` (transições regulares), `COORDENADOR_RS` (cancelamento e bloqueio), `ADMIN_RS`

```json
// Request
{
  "to_status": "CrmCaseStatus",
  "reason": "string?",    // obrigatório para cancelamento
  "version": "number"
}

// Response 200 — { data: { id, new_status, previous_status, transitioned_at } }
// Response 422 — transição inválida: { code: "CRM-025", detail: "Transição de CADASTRO para CONCLUIDO não é permitida" }
// Response 422 — critérios não atendidos: { code: "CRM-026", unmet_criteria: ["dossier_pending"] }
// Response 403 — Analista tentando cancelar sem ser Coordenador RS: { code: "CRM-027" }
```

### `PATCH /api/v1/crm/cases/:id/assign`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{ "analyst_id": "uuid", "version": "number" }

// Response 200 — notificação NOT-CRM-01 enviada ao Analista
// Response 422 — Analista inativo: { code: "CRM-028" }
```

### `GET /api/v1/crm/cases/:id/history`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — histórico completo de transições de estado
{
  "data": [
    {
      "from_status": "CADASTRO",
      "to_status": "SIMULACAO",
      "transitioned_by": { "id": "uuid", "name": "string" },
      "reason": "string?",
      "transitioned_at": "ISO8601"
    }
  ]
}
```

---

## 5. Endpoints — Contatos (`/contacts`)

### `GET /api/v1/crm/contacts`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`
**Query params:** `page`, `per_page`, `type` (`CEDENTE | CESSIONARIO | PARCEIRO | INCORPORADORA`), `search` (nome/e-mail mascarado)

> **RN-012:** Dados pessoais mascarados em listagens. `ANALISTA_RS` vê apenas Contatos vinculados aos seus Casos.

```json
// Response 200
{
  "data": [
    {
      "id": "uuid",
      "name": "Jo** ****",
      "email_masked": "jo***@gmail.com",
      "phone_masked": "(85) 9****-5678",
      "type": "CEDENTE",
      "active_cases_count": 1,
      "created_at": "ISO8601"
    }
  ]
}
```

### `POST /api/v1/crm/contacts`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{
  "name": "string",
  "cpf": "string (000.000.000-00) | cnpj?",
  "email": "string",
  "phone": "string?",
  "type": "CEDENTE | CESSIONARIO | PARCEIRO | INCORPORADORA",
  "partner_id": "uuid?"   // obrigatório se type=PARCEIRO
}

// Response 201
// Response 409 — CPF/e-mail já cadastrado: { code: "CRM-031" }
```

### `GET /api/v1/crm/contacts/:id`

**Role:** `ANALISTA_RS` (Contatos dos seus Casos), `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — dados completos (não mascarados) + Casos vinculados
{
  "data": {
    "id": "uuid",
    "name": "João Silva",
    "cpf": "123.456.789-00",
    "email": "joao@gmail.com",
    "phone": "(85) 99999-5678",
    "type": "CEDENTE",
    "linked_cases": [{ "id": "uuid", "identifier": "RS-0042", "status": "NEGOCIACAO" }],
    "opt_out_whatsapp": false,
    "created_at": "ISO8601"
  }
}
```

### `PATCH /api/v1/crm/contacts/:id`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request (campos opcionais + Optimistic Lock)
{ "phone": "string?", "email": "string?", "opt_out_whatsapp": "boolean?", "version": "number" }

// Response 200
// Response 409 — version mismatch: { code: "CRM-032" }
```

### `PATCH /api/v1/crm/contacts/:id/opt-out`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request — registro de opt-out WhatsApp (LGPD / RN-188)
{ "channel": "WHATSAPP", "reason": "string?" }

// Response 200 — opt_out_whatsapp = true, registrado no log de auditoria
```

---

## 6. Endpoints — Atividades (`/cases/:id/activities`)

### `GET /api/v1/crm/cases/:id/activities`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`
**Query params:** `page`, `per_page`, `type` (filtro por tipo), `from`, `to` (range de datas)

```json
// Response 200
{
  "data": [
    {
      "id": "uuid",
      "type": "LIGACAO | REUNIAO | WHATSAPP | NOTA | FOLLOW_UP",
      "summary": "string",
      "notes": "string?",
      "scheduled_at": "ISO8601?",
      "completed_at": "ISO8601?",
      "created_by": { "id": "uuid", "name": "string" },
      "created_at": "ISO8601"
    }
  ]
}
```

### `POST /api/v1/crm/cases/:id/activities`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{
  "type": "LIGACAO | REUNIAO | WHATSAPP | NOTA | FOLLOW_UP",
  "summary": "string (max 200 chars)",
  "notes": "string? (max 2000 chars)",
  "scheduled_at": "ISO8601?",       // obrigatório se type=FOLLOW_UP ou REUNIAO
  "contact_id": "uuid?"
}

// Response 201
// Response 422 — follow_up sem scheduled_at: { code: "CRM-041" }
```

### `PATCH /api/v1/crm/cases/:id/activities/:activity_id`

**Role:** `ANALISTA_RS` (atividades próprias), `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{ "summary": "string?", "notes": "string?", "completed_at": "ISO8601?", "version": "number" }
```

### `DELETE /api/v1/crm/cases/:id/activities/:activity_id`

**Role:** `ANALISTA_RS` (atividades próprias criadas há menos de 1h), `COORDENADOR_RS`, `ADMIN_RS`

```
// Response 204 — soft delete
// Response 422 — atividade com mais de 1h não pode ser excluída por Analista: { code: "CRM-042" }
```

---

## 7. Endpoints — Comunicação WhatsApp (`/cases/:id/messages`)

### `GET /api/v1/crm/cases/:id/messages`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`
**Query params:** `page`, `per_page`

```json
// Response 200 — histórico de mensagens do Caso
{
  "data": [
    {
      "id": "uuid",
      "direction": "OUTBOUND | INBOUND",
      "content": "string",
      "template_id": "string?",
      "status": "SENT | DELIVERED | READ | FAILED",
      "sent_at": "ISO8601",
      "contact": { "id": "uuid", "name": "string" }
    }
  ]
}
```

### `POST /api/v1/crm/cases/:id/messages`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request — mensagem de texto livre (dentro da janela de 24h)
{
  "contact_id": "uuid",
  "content": "string (max 4096 chars)"
}

// Response 201 — { data: { id, status: "SENT", ... } }
// Response 422 — fora da janela de 24h, use template: { code: "CRM-051", window_expired_at: "ISO8601" }
// Response 422 — opt-out registrado: { code: "CRM-052" }
```

### `POST /api/v1/crm/cases/:id/messages/template`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request — template HSM pré-aprovado pela Meta
{
  "contact_id": "uuid",
  "template_name": "string",
  "template_params": { "key": "value" }
}

// Response 201
// Response 422 — template não encontrado ou não aprovado: { code: "CRM-053" }
// Response 422 — opt-out registrado: { code: "CRM-052" }
```

### `GET /api/v1/crm/messages/templates`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — lista de templates aprovados pela Meta
{
  "data": [
    {
      "name": "string",
      "category": "UTILITY | AUTHENTICATION | MARKETING",
      "status": "APPROVED | PENDING | REJECTED",
      "language": "pt_BR",
      "body": "string (preview)"
    }
  ]
}
```

---

## 8. Endpoints — Dossiê (`/cases/:id/dossier`)

### `GET /api/v1/crm/cases/:id/dossier`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — documentos do Dossiê com status e progresso
{
  "data": {
    "summary": { "total": 8, "approved": 5, "pending": 2, "rejected": 1 },
    "documents": [
      {
        "id": "uuid",
        "type": "IDENTIDADE | COMPROVANTE_ENDERECO | EXTRATO_SALDO | CONTRATO | COMPROVANTE_ESCROW | OUTRO",
        "status": "PENDENTE | APROVADO | REJEITADO",
        "origin": "CEDENTE_VIA_PLATAFORMA | UPLOAD_MANUAL",
        "filename": "string",
        "file_size_kb": 245,
        "uploaded_at": "ISO8601",
        "reviewed_at": "ISO8601?",
        "rejection_reason": "string?"
      }
    ]
  }
}
```

### `POST /api/v1/crm/cases/:id/dossier`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`
**Content-Type:** `multipart/form-data`

```
// Form fields:
// - document_type: DocumentType
// - file: File (pdf/jpg/png, máx 10MB — RN-058)

// Response 201 — { data: { id, status: "PENDENTE", signed_url: "https://..." } }
// Response 422 — tipo de arquivo inválido: { code: "CRM-061" }
// Response 422 — tamanho excedido: { code: "CRM-062" }
```

### `PATCH /api/v1/crm/cases/:id/dossier/:doc_id/approve`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — { data: { id, status: "APROVADO", reviewed_at: "ISO8601" } }
// Response 422 — documento já aprovado: { code: "CRM-063" }
```

### `PATCH /api/v1/crm/cases/:id/dossier/:doc_id/reject`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{ "rejection_reason": "string (min 20 chars)" }

// Response 200 — { data: { id, status: "REJEITADO", rejection_reason: "string" } }
```

### `DELETE /api/v1/crm/cases/:id/dossier/:doc_id`

**Role:** `ANALISTA_RS` (documentos não aprovados), `COORDENADOR_RS`, `ADMIN_RS`

```
// Response 204 — soft delete + remoção do Storage
// Response 422 — documento aprovado não pode ser removido: { code: "CRM-064" }
```

### `GET /api/v1/crm/cases/:id/dossier/:doc_id/download`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — signed URL temporária (15 min de validade)
{ "data": { "url": "https://supabase.co/storage/signed/...", "expires_at": "ISO8601" } }
```

---

## 9. Endpoints — Negociações (`/cases/:id/negotiations`)

### `GET /api/v1/crm/cases/:id/negotiations`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — histórico de propostas e contrapropostas
{
  "data": [
    {
      "id": "uuid",
      "status": "PENDENTE | ACEITA | RECUSADA | EXPIRADA | CONTRAPROPOSTA",
      "proposed_value": "280000.00",
      "cessionario": { "id": "uuid", "name": "string" },
      "proposed_by": { "id": "uuid", "name": "string" },
      "expires_at": "ISO8601",
      "created_at": "ISO8601"
    }
  ]
}
```

### `POST /api/v1/crm/cases/:id/negotiations`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request — criar nova proposta
{
  "cessionario_contact_id": "uuid",
  "proposed_value": "decimal string",
  "expires_at": "ISO8601?"     // padrão: 7 dias
}

// Response 201
// Response 422 — Caso não está em estado MATCH ou NEGOCIACAO: { code: "CRM-071" }
```

### `PATCH /api/v1/crm/cases/:id/negotiations/:proposal_id/accept`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — { data: { id, status: "ACEITA", accepted_at: "ISO8601" } }
// Response 422 — proposta expirada: { code: "CRM-072" }
```

### `PATCH /api/v1/crm/cases/:id/negotiations/:proposal_id/reject`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{ "reason": "string?" }

// Response 200 — { data: { id, status: "RECUSADA" } }
```

### `POST /api/v1/crm/cases/:id/negotiations/:proposal_id/counter`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request — criar contraproposta
{ "counter_value": "decimal string", "reason": "string?" }

// Response 201 — { data: { id, status: "PENDENTE", parent_proposal_id: "uuid" } }
```

---

## 10. Endpoints — Comissão (`/cases/:id/commission`)

### `GET /api/v1/crm/cases/:id/commission`

**Role:** `ANALISTA_RS`, `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200
{
  "data": {
    "case_id": "uuid",
    "accepted_value": "280000.00",
    "commission_rate": "0.05",
    "estimated_gross": "14000.00",
    "partner_share": "3500.00",
    "analyst_share": "1400.00",
    "net_company": "9100.00",
    "status": "ESTIMADA | CONFIRMADA",
    "discount_applied": "0.00",
    "discount_reason": "string?",
    "confirmed_at": "ISO8601?"
  }
}
```

### `PATCH /api/v1/crm/cases/:id/commission`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request — ajuste de comissão (ex: desconto aprovado)
{
  "discount_percent": "decimal (0-15)?",    // cap 15% para Casos > R$1M (RN-052)
  "discount_reason": "string (min 20 chars)?",
  "version": "number"
}

// Response 200
// Response 422 — desconto acima do cap permitido: { code: "CRM-081" }
```

---

## 11. Endpoints — SLA (`/sla`)

### `GET /api/v1/crm/sla/alerts`

**Role:** `ANALISTA_RS` (alertas próprios), `COORDENADOR_RS`, `ADMIN_RS`
**Query params:** `status` (`ATENCAO | CRITICO | VENCIDO`), `analyst_id`, `page`, `per_page`

```json
// Response 200
{
  "data": [
    {
      "id": "uuid",
      "case_id": "uuid",
      "case_identifier": "RS-0042",
      "status": "CRITICO",
      "percent_consumed": 87,
      "deadline": "ISO8601",
      "current_state": "NEGOCIACAO",
      "assigned_analyst": { "id": "uuid", "name": "string" }
    }
  ]
}
```

### `GET /api/v1/crm/sla/config`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — prazos por estado do Caso
{
  "data": {
    "CADASTRO": { "warning_hours": 24, "critical_hours": 48 },
    "SIMULACAO": { "warning_hours": 24, "critical_hours": 48 },
    "VERIFICACAO": { "warning_hours": 48, "critical_hours": 96 },
    "NEGOCIACAO": { "warning_hours": 72, "critical_hours": 120 }
  }
}
```

### `PATCH /api/v1/crm/sla/config`

**Role:** `ADMIN_RS`

```json
// Request
{
  "state": "CrmCaseStatus",
  "warning_hours": "number",
  "critical_hours": "number"
}

// Response 200
// Response 422 — critical_hours < warning_hours: { code: "CRM-091" }
```

---

## 12. Endpoints — Relatórios (`/reports`)

### `GET /api/v1/crm/reports/pipeline`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`
**Query params:** `from`, `to`, `analyst_id`, `status`, `scenario`

```json
// Response 200 — funil de Casos por estado
{
  "data": {
    "period": { "from": "ISO8601", "to": "ISO8601" },
    "by_status": [
      { "status": "NEGOCIACAO", "count": 12, "avg_days": 3.4 }
    ],
    "by_analyst": [
      { "analyst_id": "uuid", "name": "string", "cases_count": 8, "avg_sla_days": 3.1 }
    ],
    "total_cases": 134
  }
}
```

### `GET /api/v1/crm/reports/commission`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`
**Query params:** `from`, `to`, `analyst_id`

```json
// Response 200 — resumo de comissões no período
{
  "data": {
    "period": { "from": "ISO8601", "to": "ISO8601" },
    "total_estimated": "145000.00",
    "total_confirmed": "98000.00",
    "by_analyst": [
      { "analyst_id": "uuid", "name": "string", "estimated": "30000.00", "confirmed": "22000.00" }
    ]
  }
}
```

### `POST /api/v1/crm/reports/export`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Request
{
  "type": "pipeline | commission | activity",
  "format": "csv | pdf",
  "from": "ISO8601",
  "to": "ISO8601",
  "filters": {
    "analyst_id": "uuid?",
    "status": "CrmCaseStatus?"
  }
}

// Response 202 — geração assíncrona
{ "data": { "job_id": "uuid", "status": "QUEUED" } }
```

### `GET /api/v1/crm/reports/export/:job_id`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — status do job
{
  "data": {
    "job_id": "uuid",
    "status": "QUEUED | PROCESSING | DONE | FAILED",
    "download_url": "https://...?",   // disponível quando status=DONE
    "expires_at": "ISO8601?"          // URL expira em 1h
  }
}
```

---

## 13. Endpoints — Configurações (`/config`)

### `GET /api/v1/crm/config`

**Role:** `COORDENADOR_RS`, `ADMIN_RS`

```json
// Response 200 — todos os parâmetros configuráveis
{
  "data": [
    {
      "key": "sla.default_warning_percent",
      "value": "70",
      "type": "NUMBER",
      "description": "Percentual de SLA consumido para alerta amarelo",
      "updated_at": "ISO8601",
      "updated_by": { "id": "uuid", "name": "string" }
    }
  ]
}
```

### `PATCH /api/v1/crm/config/:key`

**Role:** `ADMIN_RS`

```json
// Request
{ "value": "string", "update_reason": "string (min 10 chars)" }

// Response 200
// Response 422 — tipo de dado inválido: { code: "CRM-101" }
// Response 422 — chave protegida (read-only): { code: "CRM-102" }
```

---

## 14. Endpoints — Webhooks de Entrada (`/webhooks`)

> Endpoints de webhook validados por assinatura HMAC-SHA256. Sem JWT Bearer.

### `POST /api/v1/crm/webhooks/platform`

**Autenticação:** Header `X-RS-Platform-Signature` (HMAC-SHA256)

```json
// Payload — novo Caso criado na plataforma (RN-183)
{
  "event": "CASE_CREATED | DOCUMENT_SYNCED | INTEREST_REGISTERED | STATE_UPDATE",
  "case_identifier": "RS-XXXX?",
  "payload": { ... }
}

// Response 200 — enfileirado no RabbitMQ
// Response 401 — assinatura inválida: { code: "CRM-111" }
```

### `POST /api/v1/crm/webhooks/whatsapp`

**Autenticação:** Query param `hub.verify_token` (setup) + Header `X-Hub-Signature-256` (eventos)

```json
// Payload — mensagem recebida de Cedente/Cessionário (RN-187)
{
  "object": "whatsapp_business_account",
  "entry": [{ "changes": [{ "value": { "messages": [...] } }] }]
}

// Response 200 — enfileirado para processamento assíncrono
```

### `POST /api/v1/crm/webhooks/zapsign`

**Autenticação:** Header `X-ZapSign-Token`

```json
// Payload — atualização de status de assinatura (RN-189)
{
  "token": "string",
  "event": "SIGNED | REFUSED | EXPIRED",
  "document": { "token": "string", "status": "string" },
  "signer": { "name": "string", "email": "string", "status": "string" }
}

// Response 200 — enfileirado no RabbitMQ
```

### `POST /api/v1/crm/webhooks/escrow`

**Autenticação:** Header `X-Celcoin-Signature` (HMAC-SHA256)

```json
// Payload — confirmação de depósito (RN-191)
{
  "transaction_id": "string",
  "case_identifier": "RS-XXXX",
  "status": "PAID | FAILED | REVERSED",
  "amount": "decimal string",
  "confirmed_at": "ISO8601"
}

// Response 200
```

---

## 15. Matriz de Acesso por Endpoint (Resumo)

| Grupo | Endpoint | ANALISTA_RS | COORDENADOR_RS | ADMIN_RS | PARCEIRO_EXTERNO |
|---|---|---|---|---|---|
| Auth | POST /auth/login | ✅ | ✅ | ✅ | ✅ |
| Team | GET /team | — | ✅ | ✅ | — |
| Team | POST /team | — | — | ✅ | — |
| Team | POST /team/:id/redistribute | — | ✅ | ✅ | — |
| Cases | GET /cases | ✅ próprios | ✅ todos | ✅ todos | ✅ do Parceiro |
| Cases | POST /cases | ✅ | ✅ | ✅ | — |
| Cases | PATCH /cases/:id/transition | ✅ regular | ✅ + cancelar | ✅ | — |
| Cases | PATCH /cases/:id/assign | — | ✅ | ✅ | — |
| Contacts | POST /contacts | ✅ | ✅ | ✅ | — |
| Activities | POST /cases/:id/activities | ✅ | ✅ | ✅ | — |
| Communication | POST /cases/:id/messages | ✅ | ✅ | ✅ | — |
| Dossier | PATCH /dossier/:doc_id/approve | — | ✅ | ✅ | — |
| Negotiations | POST /cases/:id/negotiations | ✅ | ✅ | ✅ | — |
| Commission | PATCH /cases/:id/commission | — | ✅ | ✅ | — |
| SLA | PATCH /sla/config | — | — | ✅ | — |
| Reports | POST /reports/export | — | ✅ | ✅ | — |
| Config | PATCH /config/:key | — | — | ✅ | — |

---

## 16. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 12 grupos, 50+ endpoints, padrão RFC 7807, error codes CRM-XXX, RBAC por endpoint, paginação, Optimistic Lock e webhooks. |
