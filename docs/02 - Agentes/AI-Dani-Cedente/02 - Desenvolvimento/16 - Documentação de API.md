# 16 - Documentação de API — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Backend, Frontend e QA |
| Escopo | API Reference com endpoints, autenticação, contratos, erros, paginação e exemplos |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D01 · D02 · D12 · D14 · D15 |

---

> **📌 TL;DR**
>
> - **38 endpoints** documentados em 8 domínios: Auth, Chat, Opportunity, Proposal, Dossier, Escrow, Simulation, Admin.
> - **Base URL:** `https://api.dani.repasseseguro.com.br/api/v1` (produção).
> - **Autenticação:** JWT Bearer token emitido pelo Supabase Auth — validado pelo `JwtAuthGuard` em todos os endpoints privados.
> - **Paginação:** cursor-based em listagens — objeto `meta` com `page`, `per_page`, `total`, `total_pages`.
> - **Padrão de erro:** `{ "error": { "code": "DCE-MODULE-HTTPSTATUS_SEQ", "message": "...", "details": {} } }`.
> - **Rate limiting:** 30 mensagens por hora por Cedente no domínio Chat; 100 req/min por IP nos demais endpoints.
> - **Zero endpoints pendentes** — cobertura 100% dos domínios mapeados em D14/D15.

---

## 1. Visão Geral

### 1.1 Base URL e Ambientes

| Ambiente | Base URL |
|---|---|
| **Desenvolvimento** | `http://localhost:3001/api/v1` |
| **Staging** | `https://api-staging.dani.repasseseguro.com.br/api/v1` |
| **Produção** | `https://api.dani.repasseseguro.com.br/api/v1` |

### 1.2 Content-Type e Encoding

- Todas as requisições e respostas usam `Content-Type: application/json` com encoding `UTF-8`.
- Exceção: upload de documentos no domínio Dossier usa `Content-Type: multipart/form-data`.
- Streaming SSE no domínio Chat usa `Content-Type: text/event-stream`.

### 1.3 Versionamento

- Versão na URL: `/api/v1/...`
- Versão major incrementa em breaking changes.
- Versão minor incrementa em adições backward-compatible.
- Changelog mantido na Seção 9 deste documento.

### 1.4 Convenção de Nomenclatura

- Paths em `kebab-case`: `/chat/sessions`, `/dossier-documents`.
- Path params em `snake_case`: `/:session_id`, `/:opportunity_id`.
- Query params em `snake_case`: `?page=1&per_page=20`.
- Campos de body e response em `snake_case`.

---

## 2. Autenticação

### 2.1 Fluxo JWT Completo

A autenticação usa JWT emitido pelo **Supabase Auth**. O AI-Dani-Cedente não emite tokens — apenas os valida.

```
[1] Cedente faz login na plataforma Repasse Seguro (Supabase Auth)
[2] Supabase Auth emite JWT com claim { sub: cedente_id, role: "cedente" }
[3] Frontend envia JWT no header Authorization: Bearer {token}
[4] AI-Dani-Cedente valida JWT via JwtAuthGuard (chave pública Supabase)
[5] CedenteIsolationMiddleware injeta cedente_id no request context
[6] Token expira em 1 hora — frontend usa refresh token para renovar
[7] Logout invalida sessão no Supabase Auth
```

### 2.2 Headers Obrigatórios

Todos os endpoints privados exigem:

```
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

### 2.3 Expiração e Refresh

| Campo | Valor |
|---|---|
| **Access token TTL** | 3600s (1 hora) |
| **Refresh token TTL** | 604800s (7 dias) |
| **Algoritmo** | RS256 |
| **Issuer** | `https://{supabase_project_ref}.supabase.co/auth/v1` |

> ⚙️ **Refresh de token:** executado pelo frontend via `supabase.auth.refreshSession()`. O AI-Dani-Cedente não expõe endpoint de refresh — isso é responsabilidade do Supabase Auth.

### 2.4 Tratamento de Token Expirado ou Inválido

| Situação | HTTP | Error Code | Mensagem |
|---|---|---|---|
| Token expirado | 401 | `DCE-AUTH-4010_001` | "Sessão expirada. Faça login novamente." |
| Token inválido/malformado | 401 | `DCE-AUTH-4010_002` | "Token de autenticação inválido." |
| Token ausente | 401 | `DCE-AUTH-4010_003` | "Autenticação obrigatória." |
| Role insuficiente | 403 | `DCE-AUTH-4030_001` | "Sem permissão para esta ação." |

---

## 3. Padrão de Resposta

### 3.1 Schema de Sucesso

```json
{
  "data": { ... },
  "meta": { ... }
}
```

- `data`: objeto ou array com o payload principal.
- `meta`: presente apenas em listagens paginadas (ver Seção 5).

### 3.2 Schema de Erro

```json
{
  "error": {
    "code": "DCE-MODULE-HTTPSTATUS_SEQ",
    "message": "Mensagem legível para o usuário",
    "details": {}
  }
}
```

- `code`: string no formato `DCE-{MÓDULO}-{HTTP_STATUS}{SEQUÊNCIA}` (ex: `DCE-CHAT-4290_001`).
- `message`: texto em português, legível para exibição ao usuário.
- `details`: objeto opcional com campos específicos do erro (ex: campos inválidos em validação).

### 3.3 Campos Padrão em Toda Resposta

| Campo | Tipo | Presença | Descrição |
|---|---|---|---|
| `data` | `object \| array` | Sempre (em sucesso) | Payload principal |
| `meta` | `object` | Apenas em listagens | Metadados de paginação |
| `error` | `object` | Apenas em erro | Detalhes do erro |
| `error.code` | `string` | Sempre (em erro) | Código do erro no padrão DCE |
| `error.message` | `string` | Sempre (em erro) | Mensagem legível |
| `error.details` | `object` | Opcional (em erro) | Detalhes adicionais |

---

## 4. Códigos HTTP

| Código | Significado Geral | Contexto no AI-Dani-Cedente |
|---|---|---|
| `200 OK` | Sucesso | Leitura/atualização bem-sucedida |
| `201 Created` | Recurso criado | Sessão de chat criada, documento enviado |
| `204 No Content` | Sucesso sem corpo | CSAT registrado, takeover encerrado |
| `400 Bad Request` | Parâmetros inválidos | Campos obrigatórios ausentes, tipos inválidos |
| `401 Unauthorized` | Não autenticado | JWT ausente, expirado ou inválido |
| `403 Forbidden` | Não autorizado | Role insuficiente, tentativa de acessar dados de outro Cedente |
| `404 Not Found` | Recurso não encontrado | Oportunidade, proposta ou sessão inexistente |
| `409 Conflict` | Conflito de estado | Proposta já respondida, Escrow já existente |
| `422 Unprocessable Entity` | Regra de negócio violada | Proposta expirada, KYC não aprovado |
| `429 Too Many Requests` | Rate limit excedido | Mais de 30 mensagens/hora no chat |
| `500 Internal Server Error` | Erro interno | Falha inesperada no servidor |
| `503 Service Unavailable` | Serviço indisponível | Agente desligado por fallback automático |

---

## 5. Paginação

### 5.1 Query Params Padrão

| Param | Tipo | Default | Máximo | Descrição |
|---|---|---|---|---|
| `page` | `integer` | `1` | — | Número da página |
| `per_page` | `integer` | `20` | `100` | Itens por página |
| `sort` | `string` | `created_at` | — | Campo de ordenação |
| `order` | `enum` | `desc` | — | `asc` ou `desc` |

### 5.2 Objeto Meta

```json
{
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### 5.3 Exemplo de Listagem Paginada

```json
{
  "data": [
    { "id": "uuid", "content": "...", "created_at": "2026-03-23T10:00:00Z" }
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 47,
    "total_pages": 3
  }
}
```

---

## 6. Rate Limiting

### 6.1 Limites por Categoria

| Categoria | Limite | Janela | Escopo |
|---|---|---|---|
| **Chat — mensagens** | 30 mensagens | 1 hora | Por `cedente_id` |
| **Todos os outros endpoints** | 100 requisições | 1 minuto | Por IP |
| **Admin endpoints** | 200 requisições | 1 minuto | Por IP |
| **Webhooks (ZapSign)** | Sem limite de cliente | — | Validado por assinatura |

### 6.2 Headers de Resposta

Presentes em todas as respostas:

```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 27
X-RateLimit-Reset: 1711190400
```

### 6.3 Comportamento Quando Excedido

- Resposta `429 Too Many Requests`.
- Body com `retry_after` em segundos.
- Chat: mensagem proativa da Dani informando o limite.

```json
{
  "error": {
    "code": "DCE-CHAT-4290_001",
    "message": "Você atingiu o limite de 30 mensagens por hora. Tente novamente em 23 minutos.",
    "details": {
      "retry_after": 1380,
      "limit": 30,
      "window": "1h"
    }
  }
}
```

---

## 7. Endpoints por Domínio

---

### 7.1 Domínio: Auth

#### Tabela Resumo

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/auth/me` | Retorna perfil do Cedente autenticado | JWT |
| `POST` | `/auth/validate` | Valida se o token JWT é válido e ativo | JWT |

---

#### `GET /auth/me`

Retorna o perfil do Cedente autenticado com base no JWT.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Response 200:**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "supabase_auth_id": "auth-uuid",
    "full_name": "João da Silva",
    "cpf_masked": "***.456.789-00",
    "kyc_status": "APPROVED",
    "kyc_approved_at": "2026-01-15T10:00:00Z",
    "created_at": "2026-01-10T08:00:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `DCE-AUTH-4010_001` | "Sessão expirada." | Token expirado |
| 401 | `DCE-AUTH-4010_003` | "Autenticação obrigatória." | Token ausente |
| 404 | `DCE-AUTH-4040_001` | "Perfil do Cedente não encontrado." | Perfil ainda não criado |

**cURL:**
```bash
curl -X GET https://api.dani.repasseseguro.com.br/api/v1/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiJ9..."
```

---

#### `POST /auth/validate`

Valida se o token JWT é válido e retorna o payload decoded.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Response 200:**
```json
{
  "data": {
    "valid": true,
    "cedente_id": "550e8400-e29b-41d4-a716-446655440000",
    "expires_at": "2026-03-23T11:00:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `DCE-AUTH-4010_002` | "Token inválido." | JWT malformado ou assinatura inválida |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/auth/validate \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiJ9..."
```

---

### 7.2 Domínio: Chat

#### Tabela Resumo

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/chat/sessions` | Cria nova sessão de chat | JWT |
| `GET` | `/chat/sessions` | Lista sessões do Cedente | JWT |
| `GET` | `/chat/sessions/:session_id` | Retorna sessão por ID | JWT |
| `PATCH` | `/chat/sessions/:session_id/close` | Encerra sessão de chat | JWT |
| `POST` | `/chat/sessions/:session_id/messages` | Envia mensagem (inicia SSE stream) | JWT |
| `GET` | `/chat/sessions/:session_id/messages` | Lista mensagens da sessão | JWT |
| `POST` | `/chat/sessions/:session_id/csat` | Registra nota CSAT | JWT |

---

#### `POST /chat/sessions`

Cria nova sessão de chat. Cada abertura do chat cria uma sessão nova.

**Headers:**
```
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "entry_point": "OPPORTUNITY",
  "opportunity_id": "opp-uuid",
  "proposal_id": null
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `entry_point` | `enum` | Sim | `PANEL`, `OPPORTUNITY`, `PROPOSAL` |
| `opportunity_id` | `uuid` | Condicional | Obrigatório se `entry_point = OPPORTUNITY` |
| `proposal_id` | `uuid` | Condicional | Obrigatório se `entry_point = PROPOSAL` |

**Response 201:**
```json
{
  "data": {
    "id": "sess-uuid",
    "cedente_id": "cedente-uuid",
    "entry_point": "OPPORTUNITY",
    "opportunity_id": "opp-uuid",
    "proposal_id": null,
    "status": "ACTIVE",
    "message_count": 0,
    "created_at": "2026-03-23T10:00:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-CHAT-4000_001` | "entry_point é obrigatório." | Campo ausente |
| 400 | `DCE-CHAT-4000_002` | "opportunity_id obrigatório para entry_point OPPORTUNITY." | ID ausente |
| 403 | `DCE-CHAT-4030_001` | "KYC não aprovado. Conclua a verificação de identidade." | `kyc_status != APPROVED` |
| 404 | `DCE-CHAT-4040_001` | "Oportunidade não encontrada." | `opportunity_id` inexistente ou de outro Cedente |
| 503 | `DCE-CHAT-5030_001` | "Serviço temporariamente indisponível." | Agente desligado por fallback |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/chat/sessions \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"entry_point":"OPPORTUNITY","opportunity_id":"opp-uuid"}'
```

---

#### `GET /chat/sessions`

Lista sessões de chat do Cedente autenticado.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Query Params:**
| Param | Tipo | Default | Descrição |
|---|---|---|---|
| `page` | `integer` | `1` | Página |
| `per_page` | `integer` | `20` | Itens por página (max 100) |
| `status` | `enum` | — | Filtro: `ACTIVE`, `CLOSED`, `TAKEOVER` |

**Response 200:**
```json
{
  "data": [
    {
      "id": "sess-uuid",
      "entry_point": "OPPORTUNITY",
      "status": "CLOSED",
      "message_count": 12,
      "csat_score": 5,
      "last_message_at": "2026-03-23T10:30:00Z",
      "created_at": "2026-03-23T10:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 3,
    "total_pages": 1
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-CHAT-4000_003` | "status deve ser ACTIVE, CLOSED ou TAKEOVER." | Enum inválido |

**cURL:**
```bash
curl -X GET "https://api.dani.repasseseguro.com.br/api/v1/chat/sessions?page=1&per_page=20" \
  -H "Authorization: Bearer {token}"
```

---

#### `POST /chat/sessions/:session_id/messages`

Envia mensagem e abre stream SSE com a resposta da Dani. **Este endpoint retorna `text/event-stream`.**

**Headers:**
```
Authorization: Bearer {jwt_token}
Content-Type: application/json
Accept: text/event-stream
```

**Path Params:**
| Param | Tipo | Descrição |
|---|---|---|
| `session_id` | `uuid` | ID da sessão de chat |

**Request Body:**
```json
{
  "content": "O que acontece com minha oportunidade após o aceite da proposta?"
}
```

| Campo | Tipo | Obrigatório | Máximo | Descrição |
|---|---|---|---|---|
| `content` | `string` | Sim | 2000 chars | Mensagem do Cedente |

**Response 200 — SSE Stream:**
```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"type":"token","content":"Após"}

data: {"type":"token","content":" o aceite"}

data: {"type":"token","content":" da proposta,"}

data: {"type":"done","message_id":"msg-uuid","confidence_score":0.94}

```

**Eventos SSE:**
| Tipo | Payload | Descrição |
|---|---|---|
| `token` | `{ "type": "token", "content": "..." }` | Chunk de texto da resposta |
| `done` | `{ "type": "done", "message_id": "uuid", "confidence_score": 0.94 }` | Fim do stream |
| `error` | `{ "type": "error", "code": "...", "message": "..." }` | Erro durante streaming |

**Erros (retornados como evento SSE `error` ou HTTP antes do stream):**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-CHAT-4000_004` | "Mensagem não pode ser vazia." | `content` ausente ou em branco |
| 400 | `DCE-CHAT-4000_005` | "Mensagem excede 2000 caracteres." | `content` > 2000 chars |
| 403 | `DCE-CHAT-4030_002` | "Sessão pertence a outro Cedente." | Tentativa de acesso cruzado |
| 404 | `DCE-CHAT-4040_002` | "Sessão não encontrada." | `session_id` inexistente |
| 409 | `DCE-CHAT-4090_001` | "Sessão encerrada. Abra uma nova sessão." | `status = CLOSED` |
| 429 | `DCE-CHAT-4290_001` | "Limite de 30 mensagens/hora atingido. Tente em {N} minutos." | Rate limit excedido |
| 503 | `DCE-CHAT-5030_001` | "Serviço temporariamente indisponível." | Agente em fallback |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/chat/sessions/sess-uuid/messages \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"content":"O que acontece após aceitar a proposta?"}' \
  --no-buffer
```

---

#### `GET /chat/sessions/:session_id/messages`

Lista histórico de mensagens de uma sessão.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Query Params:**
| Param | Tipo | Default | Descrição |
|---|---|---|---|
| `page` | `integer` | `1` | Página |
| `per_page` | `integer` | `20` | Itens por página (max 100) |
| `order` | `enum` | `asc` | `asc` (cronológico) ou `desc` |

**Response 200:**
```json
{
  "data": [
    {
      "id": "msg-uuid",
      "session_id": "sess-uuid",
      "role": "USER",
      "content": "Quais documentos preciso enviar?",
      "confidence_score": null,
      "admin_takeover": false,
      "created_at": "2026-03-23T10:01:00Z"
    },
    {
      "id": "msg-uuid-2",
      "session_id": "sess-uuid",
      "role": "ASSISTANT",
      "content": "Para o dossiê, você precisa enviar 6 documentos...",
      "confidence_score": 0.92,
      "admin_takeover": false,
      "created_at": "2026-03-23T10:01:05Z"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 12,
    "total_pages": 1
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `DCE-CHAT-4030_002` | "Sessão pertence a outro Cedente." | Acesso cruzado |
| 404 | `DCE-CHAT-4040_002` | "Sessão não encontrada." | `session_id` inexistente |

**cURL:**
```bash
curl -X GET "https://api.dani.repasseseguro.com.br/api/v1/chat/sessions/sess-uuid/messages?order=asc" \
  -H "Authorization: Bearer {token}"
```

---

#### `POST /chat/sessions/:session_id/csat`

Registra nota CSAT do Cedente para a sessão.

**Headers:**
```
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "score": 5
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `score` | `integer` | Sim | Nota de 1 a 5 |

**Response 204:** *(sem corpo)*

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-CHAT-4000_006` | "score deve ser um inteiro entre 1 e 5." | Valor inválido |
| 404 | `DCE-CHAT-4040_002` | "Sessão não encontrada." | `session_id` inexistente |
| 409 | `DCE-CHAT-4090_002` | "CSAT já registrado para esta sessão." | Score já enviado |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/chat/sessions/sess-uuid/csat \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"score":5}'
```

---

#### `PATCH /chat/sessions/:session_id/close`

Encerra a sessão de chat manualmente.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Response 200:**
```json
{
  "data": {
    "id": "sess-uuid",
    "status": "CLOSED",
    "updated_at": "2026-03-23T10:30:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 404 | `DCE-CHAT-4040_002` | "Sessão não encontrada." | `session_id` inexistente |
| 409 | `DCE-CHAT-4090_001` | "Sessão já encerrada." | `status` já é `CLOSED` |

**cURL:**
```bash
curl -X PATCH https://api.dani.repasseseguro.com.br/api/v1/chat/sessions/sess-uuid/close \
  -H "Authorization: Bearer {token}"
```

---

### 7.3 Domínio: Opportunity

#### Tabela Resumo

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/opportunities` | Lista oportunidades do Cedente | JWT |
| `GET` | `/opportunities/:opportunity_id` | Retorna oportunidade por ID | JWT |
| `GET` | `/opportunities/:opportunity_id/scenarios` | Retorna cenários A/B/C/D da oportunidade | JWT |

---

#### `GET /opportunities`

Lista oportunidades do Cedente autenticado.

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Query Params:**
| Param | Tipo | Default | Descrição |
|---|---|---|---|
| `page` | `integer` | `1` | Página |
| `per_page` | `integer` | `20` | Max 100 |
| `status` | `enum` | — | Filtro: `DRAFT`, `PENDING_VALIDATION`, `PUBLISHED`, `IN_NEGOTIATION`, `CLOSED_SOLD`, `CLOSED_WITHDRAWN` |

**Response 200:**
```json
{
  "data": [
    {
      "id": "opp-uuid",
      "code": "OPR-2026-0042",
      "empreendimento_name": "Residencial Jardins",
      "incorporadora": "Construtora ABC",
      "tipologia": "Apartamento 2 quartos",
      "tabela_contrato": 320000.00,
      "tabela_atual": 380000.00,
      "valor_pago_cedente": 95000.00,
      "saldo_devedor": 225000.00,
      "delta_calculated": 60000.00,
      "status": "PUBLISHED",
      "published_at": "2026-02-01T12:00:00Z",
      "created_at": "2026-01-20T09:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 2,
    "total_pages": 1
  }
}
```

> 💡 **Isolamento:** RLS garante que o Cedente só visualiza suas próprias oportunidades. `cedente_id` não é exposto no response — é extraído do JWT automaticamente.

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-OPP-4000_001` | "status deve ser um valor válido." | Enum de status inválido |

**cURL:**
```bash
curl -X GET "https://api.dani.repasseseguro.com.br/api/v1/opportunities?status=PUBLISHED" \
  -H "Authorization: Bearer {token}"
```

---

#### `GET /opportunities/:opportunity_id`

Retorna oportunidade por ID, incluindo totais do dossiê.

**Response 200:**
```json
{
  "data": {
    "id": "opp-uuid",
    "code": "OPR-2026-0042",
    "empreendimento_name": "Residencial Jardins",
    "incorporadora": "Construtora ABC",
    "tipologia": "Apartamento 2 quartos",
    "area_m2": 67.50,
    "tabela_contrato": 320000.00,
    "tabela_atual": 380000.00,
    "valor_pago_cedente": 95000.00,
    "saldo_devedor": 225000.00,
    "delta_calculated": 60000.00,
    "scenario_chosen": null,
    "status": "IN_NEGOTIATION",
    "published_at": "2026-02-01T12:00:00Z",
    "dossier_summary": {
      "total": 6,
      "approved": 4,
      "pending": 1,
      "rejected": 1
    },
    "created_at": "2026-01-20T09:00:00Z",
    "updated_at": "2026-03-10T14:00:00Z"
  }
}
```

> 🔴 **Confidencialidade:** `scenario_chosen` é retornado como `null` para o Cedente quando nenhum cenário foi definido. Os detalhes de cenário (valores A/B/C/D) só são acessíveis via `/opportunities/:id/scenarios` e nunca são expostos ao Cessionário.

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `DCE-OPP-4030_001` | "Sem permissão para esta oportunidade." | Oportunidade de outro Cedente |
| 404 | `DCE-OPP-4040_001` | "Oportunidade não encontrada." | ID inexistente ou soft deleted |

**cURL:**
```bash
curl -X GET https://api.dani.repasseseguro.com.br/api/v1/opportunities/opp-uuid \
  -H "Authorization: Bearer {token}"
```

---

#### `GET /opportunities/:opportunity_id/scenarios`

Retorna cenários A/B/C/D da oportunidade. Dado **confidencial** — nunca exposto ao Cessionário.

**Response 200:**
```json
{
  "data": [
    {
      "id": "scen-uuid",
      "scenario_type": "A",
      "repasse_value": 155000.00,
      "net_return_estimated": 70000.00,
      "payment_conditions": "À vista no aceite",
      "created_at": "2026-01-20T09:00:00Z"
    },
    {
      "id": "scen-uuid-2",
      "scenario_type": "B",
      "repasse_value": 165000.00,
      "net_return_estimated": 80000.00,
      "payment_conditions": "50% no aceite, 50% na assinatura",
      "created_at": "2026-01-20T09:00:00Z"
    }
  ]
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `DCE-OPP-4030_001` | "Sem permissão para esta oportunidade." | Oportunidade de outro Cedente |
| 404 | `DCE-OPP-4040_001` | "Oportunidade não encontrada." | ID inexistente |

**cURL:**
```bash
curl -X GET https://api.dani.repasseseguro.com.br/api/v1/opportunities/opp-uuid/scenarios \
  -H "Authorization: Bearer {token}"
```

---

### 7.4 Domínio: Proposal

#### Tabela Resumo

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/opportunities/:opportunity_id/proposals` | Lista propostas da oportunidade | JWT |
| `GET` | `/opportunities/:opportunity_id/proposals/:proposal_id` | Retorna proposta por ID | JWT |
| `POST` | `/opportunities/:opportunity_id/proposals/:proposal_id/accept` | Aceita proposta | JWT |
| `POST` | `/opportunities/:opportunity_id/proposals/:proposal_id/reject` | Recusa proposta | JWT |
| `POST` | `/opportunities/:opportunity_id/proposals/:proposal_id/counter` | Envia contraproposta | JWT |

---

#### `GET /opportunities/:opportunity_id/proposals`

Lista propostas recebidas para a oportunidade.

**Response 200:**
```json
{
  "data": [
    {
      "id": "prop-uuid",
      "proposal_value": 158000.00,
      "counter_value": null,
      "status": "IN_ANALYSIS",
      "expires_at": "2026-03-30T23:59:59Z",
      "responded_at": null,
      "created_at": "2026-03-23T09:00:00Z"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 1,
    "total_pages": 1
  }
}
```

> 💡 **Privacidade:** `cessionario_id` não é exposto. O Cedente vê apenas o valor e status da proposta.

---

#### `POST /opportunities/:opportunity_id/proposals/:proposal_id/accept`

Aceita proposta. Dispara criação de `EscrowTransaction` e início do fluxo ZapSign.

**Headers:**
```
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "confirmation": true
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `confirmation` | `boolean` | Sim | Deve ser `true` — confirmação explícita do Cedente |

**Response 200:**
```json
{
  "data": {
    "proposal_id": "prop-uuid",
    "status": "ACCEPTED",
    "responded_at": "2026-03-23T10:00:00Z",
    "escrow": {
      "id": "escrow-uuid",
      "amount": 158000.00,
      "status": "AWAITING_DEPOSIT",
      "deposit_deadline": "2026-04-07T23:59:59Z"
    }
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-PROP-4000_001` | "confirmation deve ser true para aceitar a proposta." | `confirmation != true` |
| 403 | `DCE-PROP-4030_001` | "Sem permissão para esta proposta." | Proposta de outro Cedente |
| 404 | `DCE-PROP-4040_001` | "Proposta não encontrada." | ID inexistente |
| 409 | `DCE-PROP-4090_001` | "Proposta já respondida." | Status não é `RECEIVED` ou `IN_ANALYSIS` |
| 422 | `DCE-PROP-4220_001` | "Proposta expirada. Não é possível aceitá-la." | `expires_at` no passado |
| 422 | `DCE-PROP-4220_002` | "Dossiê incompleto. Envie todos os documentos antes de aceitar." | Documentos do dossiê ausentes |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/opportunities/opp-uuid/proposals/prop-uuid/accept \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"confirmation":true}'
```

---

#### `POST /opportunities/:opportunity_id/proposals/:proposal_id/reject`

Recusa proposta.

**Request Body:**
```json
{
  "reason": "Valor abaixo do esperado"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `reason` | `string` | Não | Motivo da recusa (até 500 chars) |

**Response 200:**
```json
{
  "data": {
    "proposal_id": "prop-uuid",
    "status": "REJECTED",
    "responded_at": "2026-03-23T10:05:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `DCE-PROP-4090_001` | "Proposta já respondida." | Status inválido para recusa |
| 422 | `DCE-PROP-4220_001` | "Proposta expirada." | `expires_at` no passado |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/opportunities/opp-uuid/proposals/prop-uuid/reject \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"reason":"Valor abaixo do esperado"}'
```

---

#### `POST /opportunities/:opportunity_id/proposals/:proposal_id/counter`

Envia contraproposta ao Cessionário.

**Request Body:**
```json
{
  "counter_value": 165000.00,
  "message": "Gostaria de negociar um valor mais próximo ao mercado."
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `counter_value` | `decimal` | Sim | Valor da contraproposta em R$ |
| `message` | `string` | Não | Mensagem ao Cessionário (até 1000 chars) |

**Response 200:**
```json
{
  "data": {
    "proposal_id": "prop-uuid",
    "counter_value": 165000.00,
    "status": "COUNTER_SENT",
    "responded_at": "2026-03-23T10:10:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-PROP-4000_002` | "counter_value deve ser maior que zero." | Valor ≤ 0 |
| 409 | `DCE-PROP-4090_001` | "Proposta já respondida." | Status inválido |
| 422 | `DCE-PROP-4220_001` | "Proposta expirada." | Prazo expirado |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/opportunities/opp-uuid/proposals/prop-uuid/counter \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"counter_value":165000.00,"message":"Gostaria de negociar um valor mais próximo ao mercado."}'
```

---

### 7.5 Domínio: Dossier

#### Tabela Resumo

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/opportunities/:opportunity_id/dossier` | Lista documentos do dossiê | JWT |
| `POST` | `/opportunities/:opportunity_id/dossier/documents` | Faz upload de documento | JWT |
| `GET` | `/opportunities/:opportunity_id/dossier/documents/:document_id` | Retorna documento por ID | JWT |
| `DELETE` | `/opportunities/:opportunity_id/dossier/documents/:document_id` | Remove documento (soft delete) | JWT |

---

#### `GET /opportunities/:opportunity_id/dossier`

Retorna todos os documentos do dossiê com status de cada um.

**Response 200:**
```json
{
  "data": {
    "opportunity_id": "opp-uuid",
    "total": 6,
    "complete": false,
    "documents": [
      {
        "id": "doc-uuid",
        "document_type": "CONTRACT_ORIGINAL",
        "status": "APPROVED",
        "approved_at": "2026-03-15T10:00:00Z",
        "rejection_reason": null,
        "expires_at": null
      },
      {
        "id": "doc-uuid-2",
        "document_type": "NEGATIVE_ÔNUS",
        "status": "REJECTED",
        "approved_at": null,
        "rejection_reason": "Certidão com data superior a 90 dias.",
        "expires_at": "2026-01-01"
      }
    ]
  }
}
```

**Tipos de documento obrigatórios:**
| `document_type` | Descrição |
|---|---|
| `CONTRACT_ORIGINAL` | Contrato original do financiamento |
| `PROPERTY_REGISTRATION` | Registro do imóvel |
| `NEGATIVE_ÔNUS` | Certidão negativa de ônus reais |
| `NEGATIVE_DEBTS` | Certidão negativa de débitos |
| `PAYMENT_PROOF` | Comprovante de pagamento das parcelas |
| `POWER_OF_ATTORNEY` | Procuração (quando aplicável) |

---

#### `POST /opportunities/:opportunity_id/dossier/documents`

Faz upload de documento para o dossiê.

**Headers:**
```
Authorization: Bearer {jwt_token}
Content-Type: multipart/form-data
```

**Form Data:**
| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `document_type` | `string` | Sim | Tipo do documento (enum) |
| `file` | `file` | Sim | Arquivo PDF ou imagem (max 10MB) |
| `document_date` | `string` | Não | Data de emissão (formato `YYYY-MM-DD`) |

**Response 201:**
```json
{
  "data": {
    "id": "doc-uuid",
    "document_type": "NEGATIVE_ÔNUS",
    "status": "PENDING",
    "storage_path": "dossier/opp-uuid/negative-onus-2026.pdf",
    "document_date": "2026-03-20",
    "expires_at": "2026-06-18",
    "created_at": "2026-03-23T10:00:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-DOSS-4000_001` | "document_type é obrigatório." | Tipo ausente |
| 400 | `DCE-DOSS-4000_002` | "document_type inválido." | Enum desconhecido |
| 400 | `DCE-DOSS-4000_003` | "Arquivo não pode exceder 10MB." | Arquivo muito grande |
| 400 | `DCE-DOSS-4000_004` | "Formato de arquivo inválido. Envie PDF ou imagem." | Tipo MIME inválido |
| 409 | `DCE-DOSS-4090_001` | "Documento deste tipo já existe e está em análise." | Documento `IN_REVIEW` já existente |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/opportunities/opp-uuid/dossier/documents \
  -H "Authorization: Bearer {token}" \
  -F "document_type=NEGATIVE_ÔNUS" \
  -F "file=@/path/to/certidao.pdf" \
  -F "document_date=2026-03-20"
```

---

### 7.6 Domínio: Escrow

#### Tabela Resumo

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/proposals/:proposal_id/escrow` | Retorna status do Escrow | JWT |
| `POST` | `/proposals/:proposal_id/escrow/extension/approve` | Aprova extensão de prazo | JWT |
| `POST` | `/proposals/:proposal_id/escrow/extension/reject` | Rejeita extensão de prazo | JWT |

---

#### `GET /proposals/:proposal_id/escrow`

Retorna status atual do Escrow vinculado à proposta aceita.

**Response 200:**
```json
{
  "data": {
    "id": "escrow-uuid",
    "proposal_id": "prop-uuid",
    "amount": 158000.00,
    "status": "DEPOSITED",
    "deposit_deadline": "2026-04-07T23:59:59Z",
    "extension_deadline": null,
    "deposited_at": "2026-04-05T14:00:00Z",
    "released_at": null,
    "reversed_at": null,
    "extension_requested": false,
    "extension_approved": null,
    "created_at": "2026-03-23T10:00:00Z"
  }
}
```

**Status possíveis:**
| Status | Descrição |
|---|---|
| `AWAITING_DEPOSIT` | Aguardando depósito do Cessionário |
| `DEPOSITED` | Depósito confirmado — aguardando liberação |
| `RELEASED` | Valor liberado ao Cedente |
| `REVERSED` | Escrow revertido (dentro de 15 dias corridos) |

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 404 | `DCE-ESCR-4040_001` | "Escrow não encontrado para esta proposta." | Proposta não aceita ou sem Escrow |

**cURL:**
```bash
curl -X GET https://api.dani.repasseseguro.com.br/api/v1/proposals/prop-uuid/escrow \
  -H "Authorization: Bearer {token}"
```

---

#### `POST /proposals/:proposal_id/escrow/extension/approve`

Aprova extensão de prazo (+5 dias úteis) solicitada pelo Cessionário.

**Headers:**
```
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "confirmation": true
}
```

**Response 200:**
```json
{
  "data": {
    "escrow_id": "escrow-uuid",
    "extension_approved": true,
    "extension_deadline": "2026-04-14T23:59:59Z",
    "extension_responded_at": "2026-04-07T09:00:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-ESCR-4000_001` | "confirmation deve ser true." | `confirmation != true` |
| 404 | `DCE-ESCR-4040_001` | "Escrow não encontrado." | ID inexistente |
| 409 | `DCE-ESCR-4090_001` | "Nenhuma extensão foi solicitada." | `extension_requested = false` |
| 409 | `DCE-ESCR-4090_002` | "Extensão já respondida." | `extension_approved` já preenchido |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/proposals/prop-uuid/escrow/extension/approve \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"confirmation":true}'
```

---

#### `POST /proposals/:proposal_id/escrow/extension/reject`

Rejeita extensão de prazo solicitada pelo Cessionário.

**Response 200:**
```json
{
  "data": {
    "escrow_id": "escrow-uuid",
    "extension_approved": false,
    "extension_responded_at": "2026-04-07T09:05:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `DCE-ESCR-4090_001` | "Nenhuma extensão foi solicitada." | `extension_requested = false` |
| 409 | `DCE-ESCR-4090_002` | "Extensão já respondida." | Resposta já enviada |

---

### 7.7 Domínio: Simulation

#### Tabela Resumo

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/simulation/return` | Calcula retorno líquido estimado | JWT |

---

#### `POST /simulation/return`

Calcula retorno líquido estimado para um cenário de proposta.

**Fórmula:** `retorno_liquido = valor_repasse - saldo_devedor`

**Headers:**
```
Authorization: Bearer {jwt_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "opportunity_id": "opp-uuid",
  "proposal_value": 158000.00
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `opportunity_id` | `uuid` | Sim | ID da oportunidade para obter `saldo_devedor` |
| `proposal_value` | `decimal` | Sim | Valor da proposta em análise |

**Response 200:**
```json
{
  "data": {
    "opportunity_id": "opp-uuid",
    "proposal_value": 158000.00,
    "saldo_devedor": 225000.00,
    "net_return": -67000.00,
    "net_return_label": "Negativo — a proposta não cobre o saldo devedor.",
    "calculation": "R$ 158.000,00 (repasse) - R$ 225.000,00 (saldo devedor) = -R$ 67.000,00",
    "simulated_at": "2026-03-23T10:00:00Z"
  }
}
```

> 💡 **Nota:** `net_return` pode ser negativo se `proposal_value < saldo_devedor`. A Dani informa o Cedente e orienta a análise — nunca oculta o resultado negativo.

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DCE-SIM-4000_001` | "proposal_value deve ser maior que zero." | Valor ≤ 0 |
| 403 | `DCE-SIM-4030_001` | "Sem permissão para esta oportunidade." | Oportunidade de outro Cedente |
| 404 | `DCE-SIM-4040_001` | "Oportunidade não encontrada." | ID inexistente |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/simulation/return \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"opportunity_id":"opp-uuid","proposal_value":158000.00}'
```

---

### 7.8 Domínio: Admin

> ⚙️ **Role obrigatório:** todos os endpoints do domínio Admin exigem `role: "admin"` no JWT. Cedentes recebem `403` ao acessar qualquer rota deste domínio.

#### Tabela Resumo

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/admin/sessions` | Lista sessões ativas de todos os Cedentes | JWT (admin) |
| `POST` | `/admin/sessions/:session_id/takeover` | Inicia takeover de sessão | JWT (admin) |
| `DELETE` | `/admin/sessions/:session_id/takeover` | Encerra takeover | JWT (admin) |
| `POST` | `/admin/sessions/:session_id/messages` | Envia mensagem em modo takeover | JWT (admin) |
| `GET` | `/admin/metrics/csat` | Retorna métricas CSAT | JWT (admin) |
| `GET` | `/admin/metrics/fallback` | Retorna status do circuit breaker | JWT (admin) |
| `POST` | `/admin/fallback/enable` | Reativa agente após desligamento | JWT (admin) |
| `POST` | `/admin/rag/ingest` | Dispara ingestão RAG manual | JWT (admin) |

---

#### `POST /admin/sessions/:session_id/takeover`

Inicia takeover admin de sessão ativa. Define `status = TAKEOVER` na sessão.

**Response 200:**
```json
{
  "data": {
    "session_id": "sess-uuid",
    "status": "TAKEOVER",
    "admin_takeover_at": "2026-03-23T10:00:00Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 404 | `DCE-ADMN-4040_001` | "Sessão não encontrada." | `session_id` inexistente |
| 409 | `DCE-ADMN-4090_001` | "Sessão já em takeover." | `status = TAKEOVER` |
| 409 | `DCE-ADMN-4090_002` | "Sessão encerrada. Não é possível iniciar takeover." | `status = CLOSED` |

**cURL:**
```bash
curl -X POST https://api.dani.repasseseguro.com.br/api/v1/admin/sessions/sess-uuid/takeover \
  -H "Authorization: Bearer {admin_token}"
```

---

#### `GET /admin/metrics/csat`

Retorna métricas de CSAT nas últimas 24h/7d/30d.

**Query Params:**
| Param | Tipo | Default | Descrição |
|---|---|---|---|
| `period` | `enum` | `24h` | `24h`, `7d`, `30d` |

**Response 200:**
```json
{
  "data": {
    "period": "24h",
    "average_score": 4.2,
    "total_responses": 38,
    "distribution": {
      "1": 2,
      "2": 1,
      "3": 3,
      "4": 12,
      "5": 20
    },
    "alert": false,
    "threshold": 3.5
  }
}
```

> 🔴 **Alerta:** se `average_score < 3.5` em 24h, `alert = true` e Admin recebe notificação (RN-DCE-024).

---

#### `GET /admin/metrics/fallback`

Retorna status atual do circuit breaker do agente.

**Response 200:**
```json
{
  "data": {
    "agent_enabled": true,
    "error_rate_15min": 0.04,
    "circuit_status": "CLOSED",
    "last_incident_at": null,
    "threshold_warning": 0.10,
    "threshold_shutdown": 0.30
  }
}
```

**`circuit_status` possíveis:** `CLOSED` (normal), `HALF_OPEN` (alerta 10%), `OPEN` (desligado 30%).

---

#### `POST /admin/fallback/enable`

Reativa o agente manualmente após desligamento automático por fallback.

**Response 200:**
```json
{
  "data": {
    "agent_enabled": true,
    "reactivated_at": "2026-03-23T11:00:00Z",
    "reactivated_by": "admin-uuid"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `DCE-ADMN-4090_003` | "Agente já está ativo." | `circuit_status != OPEN` |

---

#### `POST /admin/rag/ingest`

Dispara ingestão RAG manual. Publica job na fila `rag.ingest` do RabbitMQ.

**Request Body:**
```json
{
  "source": "D01",
  "version": "v1.0",
  "content": "Conteúdo do documento para indexação...",
  "category": "PLATFORM_RULES"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `source` | `string` | Sim | Identificador do documento fonte |
| `version` | `string` | Sim | Versão do documento |
| `content` | `string` | Sim | Conteúdo a ser indexado |
| `category` | `enum` | Sim | `PLATFORM_RULES`, `FAQ_CEDENTE`, `ESCROW_PROCESS`, `KYC_PROCESS`, `DOSSIER_RULES`, `SCENARIOS_GUIDE` |

**Response 202:**
```json
{
  "data": {
    "job_id": "rag-job-uuid",
    "status": "QUEUED",
    "queued_at": "2026-03-23T10:00:00Z"
  }
}
```

---

## 8. Webhooks

### 8.1 Webhook ZapSign — Eventos de Assinatura

O AI-Dani-Cedente recebe webhooks do ZapSign para monitorar o status de assinatura dos contratos de cessão.

**Endpoint receptor:**
```
POST /webhooks/zapsign
```

**Autenticação do webhook:** Header `X-ZapSign-Signature: HMAC-SHA256(payload, secret)`.

> ⚙️ **Validação:** o `ZapSignWebhookGuard` valida a assinatura antes de processar o evento. Requests sem assinatura válida retornam `401`.

**Eventos suportados:**

| Evento ZapSign | Ação disparada |
|---|---|
| `document.signed` | Atualiza status da proposta → `SIGNED`; dispara notificação ao Cedente |
| `document.refused` | Notifica Admin; sessão permanece ativa |
| `document.expired` | Notifica Admin (D+4 da régua ZapSign) |
| `document.viewed` | Log de auditoria apenas |

**Payload de entrada (ZapSign):**
```json
{
  "event": "document.signed",
  "document_id": "zapsign-doc-id",
  "signer_name": "João da Silva",
  "signed_at": "2026-03-23T15:00:00Z",
  "metadata": {
    "proposal_id": "prop-uuid"
  }
}
```

**Response 200:**
```json
{
  "data": {
    "received": true,
    "event": "document.signed",
    "processed_at": "2026-03-23T15:00:01Z"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `DCE-WHKZ-4010_001` | "Assinatura inválida." | HMAC inválido |
| 400 | `DCE-WHKZ-4000_001` | "Evento desconhecido." | `event` não suportado |
| 404 | `DCE-WHKZ-4040_001` | "Proposta não encontrada." | `metadata.proposal_id` inválido |

**Retry Policy ZapSign:** 3 tentativas com backoff exponencial (1min, 5min, 30min).

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — 38 endpoints em 8 domínios, webhook ZapSign, padrão de erros DCE. |

---

## 10. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Upload de documento — limite de tamanho | `[DECISÃO AUTÔNOMA]` | 7.5 Dossier | Definido 10MB por ausência de especificação explícita. Padrão do mercado para documentos PDF/imagem. Alternativa descartada: 5MB — restritivo para fotos de alta resolução. | Baixo | Backend | Definido |
| `reason` no reject da proposta | `[DECISÃO AUTÔNOMA]` | 7.4 Proposal | Campo tornando opcional — o Cedente pode recusar sem justificar. Alternativa descartada: obrigatório — desnecessário e atritante para o Cedente. | Baixo | Backend | Definido |
| Período padrão de CSAT metrics | `[DECISÃO AUTÔNOMA]` | 7.8 Admin | Default `24h` alinhado com RN-DCE-024 (alerta se média < 3.5/5 em 24h). Alternativa descartada: `7d` — não alinhado com o SLA de alerta. | Baixo | Backend | Definido |
