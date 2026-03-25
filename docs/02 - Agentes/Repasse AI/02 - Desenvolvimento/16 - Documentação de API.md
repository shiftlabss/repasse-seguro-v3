# Repasse AI
## 16 — Documentação de API

| Campo | Valor |
|---|---|
| **Destinatário** | Backend, Frontend e QA |
| **Escopo** | API Reference com endpoints, autenticação, contratos, erros, paginação e exemplos |
| **Módulo** | Repasse AI |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Total de endpoints documentados:** 22 endpoints em 6 domínios (chat, ai, calculator, notification, supervision, whatsapp) + 1 health check + 1 webhook.
> - **Base URL:** `https://api.repasseseguro.com.br/repasse-ai/v1`. Versionamento via URL path (`/v1`).
> - **Autenticação:** JWT Bearer Token herdado da sessão da plataforma Cessionário. Header: `Authorization: Bearer {token}`. Sem endpoint de login próprio — token gerado pela plataforma.
> - **Paginação:** query params `page` (default: 1) + `per_page` (default: 20, máx: 100). Objeto `meta` em toda listagem.
> - **Padrão de erro:** `{ "error": { "code": "ERROR_CODE", "message": "...", "details": {} } }`.
> - **Rate limiting:** 30 msg/h (webchat), 20 msg/h (WhatsApp), 10 req/min (calculadora), 5 req/min (Admin supervisão).
> - **Seções pendentes:** 0. 3 decisões autônomas aplicadas (marcadas inline).

---

## 1. Visão Geral

### 1.1 Base URL e Ambientes

| Ambiente | Base URL |
|---|---|
| Development (local) | `http://localhost:3000/repasse-ai/v1` |
| Staging | `https://api-staging.repasseseguro.com.br/repasse-ai/v1` |
| Production | `https://api.repasseseguro.com.br/repasse-ai/v1` |

### 1.2 Content-Type e Encoding

- `Content-Type: application/json` obrigatório em todas as requisições com body.
- `Accept: application/json` em todas as requisições.
- Encoding: `UTF-8`.
- SSE (streaming): `Accept: text/event-stream` + `Cache-Control: no-cache`.

### 1.3 Convenção de Versionamento

- Versionamento via URL path: `/repasse-ai/v1/`.
- Breaking changes exigem nova versão (`/v2`). Versão anterior mantida por 90 dias após deprecação.
- Deprecações anunciadas via header `Deprecation: true` + `Sunset: {data}`.

### 1.4 Convenção de Nomenclatura de Endpoints

- Recursos em `kebab-case` plural: `/chat/conversations`, `/ai/interactions`.
- Ações como sub-recursos: `/supervision/takeovers/{id}/release`.
- IDs de recursos sempre UUIDs v4 no path: `/chat/conversations/{conversation_id}`.
- Query params em `snake_case`: `page`, `per_page`, `confidence_min`, `created_after`.

---

## 2. Autenticação

### 2.1 Contexto

O Repasse AI não possui endpoint de login próprio. O JWT é gerado pela autenticação da plataforma Cessionário (Supabase Auth). O serviço Repasse AI **valida** o token e extrai o `cessionario_id` do payload.

### 2.2 Header Obrigatório

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

O token JWT deve conter no payload:

```json
{
  "sub": "uuid-do-usuario",
  "cessionario_id": "uuid-do-cessionario",
  "role": "CESSIONARIO",
  "exp": 1750000000,
  "iat": 1749996400
}
```

Para endpoints Admin, `role` deve ser `ADMIN`.

### 2.3 Expiração e Refresh

- **Expiração do access token:** 1 hora (gerenciado pela plataforma Cessionário).
- **Refresh:** realizado pela plataforma Cessionário via Supabase Auth. O Repasse AI não expõe endpoint de refresh.
- **SSE (streaming):** o token é validado na abertura da conexão SSE. Conexões ativas não são interrompidas ao expirar durante o stream. [DECISÃO AUTÔNOMA] — Interromper SSE por expiração de token durante análise em andamento seria UX destrutiva; validação apenas na abertura é suficiente dado TTL de 1h. Alternativa descartada: heartbeat de re-validação a cada mensagem (overhead desnecessário).

### 2.4 Tratamento de Token Inválido ou Expirado

```json
HTTP/1.1 401 Unauthorized
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Token de autenticação inválido ou expirado.",
    "details": {}
  }
}
```

### 2.5 Endpoint Admin

Endpoints do domínio `/supervision` e `/admin/*` exigem `role: ADMIN` no payload do JWT. Token de Cessionário com `role: CESSIONARIO` retorna `403 FORBIDDEN`.

---

## 3. Padrão de Resposta

### 3.1 Schema de Sucesso

```json
{
  "data": { ... },         // objeto ou array de objetos
  "meta": { ... }          // presente apenas em listagens paginadas
}
```

### 3.2 Schema de Sucesso — Listagem

```json
{
  "data": [ ... ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### 3.3 Schema de Erro

```json
{
  "error": {
    "code": "ERROR_CODE",          // string: identificador único do erro
    "message": "Mensagem legível", // string: mensagem para exibição
    "details": {}                  // object opcional: campos específicos do erro
  }
}
```

### 3.4 Schema de SSE (streaming)

O endpoint de streaming (`GET /chat/stream`) usa Server-Sent Events:

```
data: {"type":"content","delta":"Análi"}

data: {"type":"content","delta":"se: o"}

data: {"type":"tool_call","tool":"calculate_delta","input":{"valor_face":100000,"valor_proposto":75000}}

data: {"type":"tool_result","tool":"calculate_delta","output":{"delta":-25.0}}

data: {"type":"done","interaction_id":"uuid","confidence":0.87,"latency_ms":1240}

data: [DONE]
```

---

## 4. Códigos HTTP

| Código | Significado Geral | Significado no Repasse AI |
|---|---|---|
| `200 OK` | Sucesso | GET ou PATCH com retorno de dados |
| `201 Created` | Recurso criado | POST que cria conversa, mensagem ou vinculação |
| `204 No Content` | Sucesso sem corpo | DELETE de vinculação WhatsApp, release de takeover |
| `400 Bad Request` | Requisição inválida | Campos obrigatórios ausentes, formato inválido, UUID mal formado |
| `401 Unauthorized` | Não autenticado | JWT ausente, inválido ou expirado |
| `403 Forbidden` | Sem permissão | Cessionário tentando acessar dados de outro; role insuficiente para Admin |
| `404 Not Found` | Recurso não encontrado | `conversation_id`, `interaction_id` ou `takeover_id` não existem |
| `409 Conflict` | Conflito de estado | Takeover já ativo para a interação; OTP já utilizado |
| `422 Unprocessable Entity` | Regra de negócio violada | Tentativa de analisar dado fora do escopo; limite de conversas ativas |
| `429 Too Many Requests` | Rate limit excedido | Limite de mensagens por hora atingido |
| `500 Internal Server Error` | Erro interno | Falha de LLM, timeout, erro inesperado de infraestrutura |
| `503 Service Unavailable` | Serviço indisponível | LLM degradado, kill switch ativo |

---

## 5. Paginação

### 5.1 Query Params Padrão

| Param | Tipo | Default | Máximo | Descrição |
|---|---|---|---|---|
| `page` | integer | 1 | — | Página atual (base 1) |
| `per_page` | integer | 20 | 100 | Itens por página |
| `sort` | string | `created_at` | — | Campo de ordenação |
| `order` | enum | `desc` | — | `asc` ou `desc` |

### 5.2 Objeto `meta`

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

---

## 6. Rate Limiting

### 6.1 Limites por Categoria

| Categoria | Limite | Janela | Escopo |
|---|---|---|---|
| Mensagens de chat (webchat) | 30 mensagens | 1 hora | Por `cessionario_id` |
| Mensagens de chat (WhatsApp) | 20 mensagens | 1 hora | Por `cessionario_id` |
| Endpoints de calculadora | 10 requisições | 1 minuto | Por `cessionario_id` |
| Endpoints Admin (supervisão) | 60 requisições | 1 minuto | Por `admin_id` |
| Endpoints de notificação | 30 requisições | 1 minuto | Por `cessionario_id` |
| Health check | 60 requisições | 1 minuto | Por IP |

### 6.2 Headers de Resposta

```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 12
X-RateLimit-Reset: 1749996400
Retry-After: 1234   # presente apenas quando 429
```

### 6.3 Resposta quando Excedido

```json
HTTP/1.1 429 Too Many Requests
Retry-After: 1234

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Limite de mensagens atingido. Aguarde 20 minutos e 34 segundos para continuar.",
    "details": {
      "limit": 30,
      "window_seconds": 3600,
      "reset_at": "2026-03-22T01:34:00-03:00",
      "retry_after_seconds": 1234
    }
  }
}
```

---

## 7. Endpoints por Domínio

### 7.1 Health Check

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/health` | Verifica disponibilidade do serviço | Não |

---

#### `GET /health`

**Descrição:** Verifica status de disponibilidade do serviço e dependências críticas.

**Headers:** Nenhum obrigatório.

**Response 200:**
```json
{
  "data": {
    "status": "healthy",
    "version": "1.0.0",
    "timestamp": "2026-03-22T00:00:00-03:00",
    "dependencies": {
      "database": "healthy",
      "redis": "healthy",
      "rabbitmq": "healthy",
      "llm": "healthy"
    }
  }
}
```

**Response 503:**
```json
{
  "data": {
    "status": "degraded",
    "version": "1.0.0",
    "timestamp": "2026-03-22T00:00:00-03:00",
    "dependencies": {
      "database": "healthy",
      "redis": "healthy",
      "rabbitmq": "healthy",
      "llm": "degraded"
    }
  }
}
```

---

### 7.2 Domínio: Chat

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/chat/conversations` | Inicia nova conversa | CESSIONARIO |
| `GET` | `/chat/conversations` | Lista conversas do Cessionário | CESSIONARIO |
| `GET` | `/chat/conversations/{conversation_id}/messages` | Lista mensagens de uma conversa | CESSIONARIO |
| `POST` | `/chat/conversations/{conversation_id}/messages` | Envia mensagem e recebe resposta via SSE | CESSIONARIO |
| `DELETE` | `/chat/history` | Apaga todo o histórico de conversas | CESSIONARIO |

---

#### `POST /chat/conversations`

**Descrição:** Inicia uma nova conversa. Retorna a conversa criada.

**Headers:**
```
Authorization: Bearer {token}
Content-Type: application/json
```

**Request body:** Nenhum campo obrigatório. Conversa criada automaticamente com `cessionario_id` do JWT.

```json
{}
```

**Response 201:**
```json
{
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "ACTIVE",
    "created_at": "2026-03-22T00:00:00-03:00",
    "updated_at": "2026-03-22T00:00:00-03:00"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `UNAUTHORIZED` | "Token de autenticação inválido ou expirado." | JWT ausente ou inválido |
| 422 | `LGPD_CONSENT_REQUIRED` | "Consentimento LGPD necessário para iniciar conversa." | Cessionário sem consentimento registrado |

**cURL:**
```bash
curl -X POST https://api.repasseseguro.com.br/repasse-ai/v1/chat/conversations \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

#### `GET /chat/conversations`

**Descrição:** Lista conversas do Cessionário autenticado, ordenadas por data de atualização.

**Headers:**
```
Authorization: Bearer {token}
```

**Query params:**
| Param | Tipo | Default | Descrição |
|---|---|---|---|
| `page` | integer | 1 | Página |
| `per_page` | integer | 20 | Itens por página (máx: 50) |
| `status` | enum | — | Filtrar por `ACTIVE`, `CLOSED` |

**Response 200:**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "ACTIVE",
      "message_count": 14,
      "last_message_at": "2026-03-22T00:00:00-03:00",
      "created_at": "2026-03-21T18:00:00-03:00"
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
| 401 | `UNAUTHORIZED` | "Token inválido ou expirado." | JWT ausente ou inválido |

---

#### `POST /chat/conversations/{conversation_id}/messages`

**Descrição:** Envia uma mensagem do Cessionário e recebe a resposta do agente via Server-Sent Events (streaming).

> 🔴 **SSE:** Esta requisição mantém a conexão aberta até o stream terminar. Timeout máximo: 60s. Se o LLM não responder em 30s, retorna T-009 (Bolha de Erro).

**Headers:**
```
Authorization: Bearer {token}
Content-Type: application/json
Accept: text/event-stream
Cache-Control: no-cache
```

**Path params:**
| Param | Tipo | Descrição |
|---|---|---|
| `conversation_id` | UUID | ID da conversa |

**Request body:**
```json
{
  "content": "Analise a oportunidade OPR-2024-001234",
  "role": "USER"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `content` | string | Sim | Texto da mensagem (máx: 2000 chars) |
| `role` | enum | Sim | Sempre `USER` para Cessionários |

**Response 200 (SSE stream):**
```
data: {"type":"content","delta":"Anali"}

data: {"type":"content","delta":"sando"}

data: {"type":"tool_call","tool":"calculate_delta","input":{"valor_face":100000,"valor_proposto":75000}}

data: {"type":"tool_result","tool":"calculate_delta","output":{"delta":-25.0,"comissao":1500.00,"custo_escrow":300.00}}

data: {"type":"content","delta":" O Δ"}

data: {"type":"done","interaction_id":"uuid","confidence":0.87,"latency_ms":1240,"cached":false}

data: [DONE]
```

**Tipos de evento SSE:**
| type | Campos | Descrição |
|---|---|---|
| `content` | `delta: string` | Token de texto da resposta do agente |
| `tool_call` | `tool: string`, `input: object` | Chamada de ferramenta do agente |
| `tool_result` | `tool: string`, `output: object` | Resultado da ferramenta |
| `done` | `interaction_id`, `confidence`, `latency_ms`, `cached` | Finalização do stream |
| `error` | `code`, `message` | Erro durante streaming |

**Erros (HTTP antes do stream):**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `INVALID_MESSAGE` | "Mensagem inválida ou vazia." | `content` ausente ou > 2000 chars |
| 401 | `UNAUTHORIZED` | "Token inválido ou expirado." | JWT ausente ou inválido |
| 403 | `FORBIDDEN` | "Conversa não pertence ao Cessionário." | `conversation_id` de outro cessionário |
| 404 | `CONVERSATION_NOT_FOUND` | "Conversa não encontrada." | `conversation_id` inexistente |
| 422 | `DATA_OUT_OF_SCOPE` | "Dados solicitados fora do escopo permitido." | Tentativa de obter dados de outro Cessionário ou Cedente |
| 429 | `RATE_LIMIT_EXCEEDED` | "Limite de mensagens atingido. Aguarde X minutos." | 30 msg/h excedidas |
| 503 | `LLM_UNAVAILABLE` | "Agente temporariamente indisponível. Tente novamente em instantes." | Kill switch ativo ou LLM degradado |

**cURL:**
```bash
curl -X POST "https://api.repasseseguro.com.br/repasse-ai/v1/chat/conversations/550e8400-e29b-41d4-a716-446655440000/messages" \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -H "Cache-Control: no-cache" \
  --no-buffer \
  -d '{"content":"Analise a oportunidade OPR-2024-001234","role":"USER"}'
```

---

#### `GET /chat/conversations/{conversation_id}/messages`

**Descrição:** Lista mensagens de uma conversa em ordem cronológica.

**Headers:**
```
Authorization: Bearer {token}
```

**Path params:** `conversation_id` (UUID)

**Query params:** `page`, `per_page` (máx: 50), `order` (default: `asc`)

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "role": "USER",
      "content": "Analise a oportunidade OPR-2024-001234",
      "created_at": "2026-03-22T00:00:00-03:00"
    },
    {
      "id": "uuid",
      "role": "AGENT",
      "content": "Análise completa: Δ = -25%...",
      "interaction_id": "uuid",
      "confidence": 0.87,
      "created_at": "2026-03-22T00:00:01-03:00"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 50,
    "total": 14,
    "total_pages": 1
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `FORBIDDEN` | "Conversa não pertence ao Cessionário." | `conversation_id` de outro cessionário |
| 404 | `CONVERSATION_NOT_FOUND` | "Conversa não encontrada." | `conversation_id` inexistente |

---

#### `DELETE /chat/history`

**Descrição:** Apaga todo o histórico de conversas do Cessionário autenticado (soft delete + enfileira anonimização assíncrona via LGPD).

**Headers:**
```
Authorization: Bearer {token}
```

**Request body:** Nenhum.

**Response 204:** Sem corpo.

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `UNAUTHORIZED` | "Token inválido ou expirado." | JWT ausente |

---

### 7.3 Domínio: AI (Admin)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/ai/interactions` | Lista interações do agente | ADMIN |
| `GET` | `/ai/interactions/{interaction_id}` | Detalhe de uma interação | ADMIN |
| `GET` | `/ai/metrics` | Métricas agregadas do agente | ADMIN |

---

#### `GET /ai/interactions`

**Descrição:** Lista interações do agente com dados de supervisão. Cessionários anonimizados.

**Headers:**
```
Authorization: Bearer {token}  // role: ADMIN
```

**Query params:**
| Param | Tipo | Default | Descrição |
|---|---|---|---|
| `page` | integer | 1 | Página |
| `per_page` | integer | 20 | Itens por página (máx: 100) |
| `confidence_min` | float | — | Confiança mínima (0.0 a 1.0) |
| `confidence_max` | float | — | Confiança máxima |
| `channel` | enum | — | `WEBCHAT` ou `WHATSAPP` |
| `created_after` | ISO 8601 | — | Filtro de data inicial |
| `created_before` | ISO 8601 | — | Filtro de data final |
| `sort` | string | `created_at` | Campo de ordenação |
| `order` | enum | `desc` | `asc` ou `desc` |

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "anonymous_id": "CESS-0042",
      "channel": "WEBCHAT",
      "user_message": "Analise a oportunidade OPR-2024-001234",
      "agent_response_summary": "Análise: Δ = -25%, comissão = R$ 1.500...",
      "confidence_score": 0.87,
      "latency_ms": 1240,
      "data_sources": ["market_data", "opportunity_data"],
      "cached": false,
      "takeover_eligible": false,
      "created_at": "2026-03-22T00:00:00-03:00"
    }
  ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 1847,
    "total_pages": 93
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `UNAUTHORIZED` | "Token inválido ou expirado." | JWT ausente |
| 403 | `FORBIDDEN` | "Acesso restrito a administradores." | Role não é ADMIN |

---

#### `GET /ai/interactions/{interaction_id}`

**Descrição:** Detalhe completo de uma interação, incluindo dados de qualidade e elegibilidade para takeover.

**Headers:**
```
Authorization: Bearer {token}  // role: ADMIN
```

**Path params:** `interaction_id` (UUID)

**Response 200:**
```json
{
  "data": {
    "id": "uuid",
    "anonymous_id": "CESS-0042",
    "conversation_id": "uuid",
    "channel": "WEBCHAT",
    "user_message": "Analise a oportunidade OPR-2024-001234",
    "agent_response": "Análise completa: Δ = -25%...",
    "confidence_score": 0.87,
    "latency_ms": 1240,
    "data_sources": ["market_data", "opportunity_data"],
    "tool_calls": [
      {
        "tool": "calculate_delta",
        "input": { "valor_face": 100000, "valor_proposto": 75000 },
        "output": { "delta": -25.0, "comissao": 1500.00 },
        "latency_ms": 120
      }
    ],
    "cached": false,
    "takeover_eligible": false,
    "takeover_threshold": 0.70,
    "active_takeover": null,
    "created_at": "2026-03-22T00:00:00-03:00"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `FORBIDDEN` | "Acesso restrito a administradores." | Role não é ADMIN |
| 404 | `INTERACTION_NOT_FOUND` | "Interação não encontrada." | `interaction_id` inexistente |

---

#### `GET /ai/metrics`

**Descrição:** Métricas agregadas do agente para o Dashboard Admin (T-022).

**Headers:**
```
Authorization: Bearer {token}  // role: ADMIN
```

**Query params:**
| Param | Tipo | Default | Descrição |
|---|---|---|---|
| `period` | enum | `7d` | `1d`, `7d`, `30d`, `90d` |
| `channel` | enum | — | `WEBCHAT`, `WHATSAPP` ou todos |

**Response 200:**
```json
{
  "data": {
    "period": "7d",
    "total_interactions": 1847,
    "avg_confidence": 0.84,
    "avg_latency_ms": 1380,
    "refusal_rate": 0.03,
    "takeover_rate": 0.08,
    "cache_hit_rate": 0.42,
    "csat_score": 4.2,
    "interactions_by_channel": {
      "WEBCHAT": 1502,
      "WHATSAPP": 345
    },
    "interactions_by_day": [
      { "date": "2026-03-22", "count": 312 }
    ]
  }
}
```

---

### 7.4 Domínio: Calculator

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/calculator/delta` | Calcula Δ e comissão de uma oportunidade | CESSIONARIO |
| `POST` | `/calculator/roi` | Calcula ROI em 3 cenários | CESSIONARIO |
| `POST` | `/calculator/portfolio` | Simula portfólio multi-oportunidade | CESSIONARIO |

---

#### `POST /calculator/delta`

**Descrição:** Calcula Δ (desconto percentual), comissão e custo de Escrow de uma oportunidade.

**Headers:**
```
Authorization: Bearer {token}
Content-Type: application/json
```

**Request body:**
```json
{
  "valor_face": 100000.00,
  "valor_proposto": 75000.00
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `valor_face` | Decimal (2 casas) | Sim | Valor de face do crédito (R$) |
| `valor_proposto` | Decimal (2 casas) | Sim | Valor de compra proposto (R$) |

**Response 200:**
```json
{
  "data": {
    "delta_percent": -25.00,
    "comissao": 1500.00,
    "custo_escrow": 300.00,
    "valor_liquido": 73200.00,
    "calculated_at": "2026-03-22T00:00:00-03:00"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `INVALID_VALUES` | "Valores inválidos para cálculo." | `valor_face` ou `valor_proposto` ≤ 0 |
| 422 | `PROPOSAL_EXCEEDS_FACE` | "Valor proposto não pode exceder valor de face." | `valor_proposto` > `valor_face` |
| 429 | `RATE_LIMIT_EXCEEDED` | "Limite de cálculos atingido." | 10 req/min excedidas |

**cURL:**
```bash
curl -X POST https://api.repasseseguro.com.br/repasse-ai/v1/calculator/delta \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{"valor_face":100000.00,"valor_proposto":75000.00}'
```

---

#### `POST /calculator/roi`

**Descrição:** Calcula ROI esperado em 3 cenários de valorização (pessimista, base, otimista).

**Request body:**
```json
{
  "valor_investido": 75000.00,
  "valor_face": 100000.00,
  "prazo_meses": 24
}
```

**Response 200:**
```json
{
  "data": {
    "cenario_pessimista": { "retorno_percent": 18.5, "retorno_reais": 13875.00 },
    "cenario_base":       { "retorno_percent": 25.0, "retorno_reais": 18750.00 },
    "cenario_otimista":   { "retorno_percent": 33.3, "retorno_reais": 24975.00 },
    "prazo_meses": 24,
    "calculated_at": "2026-03-22T00:00:00-03:00"
  }
}
```

---

#### `POST /calculator/portfolio`

**Descrição:** Simula ROI total de portfólio multi-oportunidade com 3 cenários de valorização (±20%).

**Request body:**
```json
{
  "oportunidades": [
    { "valor_face": 100000.00, "valor_proposto": 75000.00, "prazo_meses": 24 },
    { "valor_face": 50000.00, "valor_proposto": 38000.00, "prazo_meses": 18 }
  ]
}
```

**Regras:**
- Mínimo: 2 oportunidades. Máximo: 10 oportunidades. [DECISÃO AUTÔNOMA] — Limite de 10 baseado em latência máxima aceitável para cálculo em tempo real; mais de 10 seria batch assíncrono. Alternativa descartada: sem limite (latência imprevisível).

**Response 200:**
```json
{
  "data": {
    "total_investido": 113000.00,
    "total_valor_face": 150000.00,
    "cenario_pessimista": { "roi_total_percent": 19.2, "roi_total_reais": 21696.00 },
    "cenario_base":       { "roi_total_percent": 26.5, "roi_total_reais": 29945.00 },
    "cenario_otimista":   { "roi_total_percent": 34.8, "roi_total_reais": 39324.00 },
    "oportunidades_count": 2,
    "calculated_at": "2026-03-22T00:00:00-03:00"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `INVALID_PORTFOLIO` | "Portfólio inválido." | < 2 ou > 10 oportunidades; valores inválidos |

---

### 7.5 Domínio: Notification

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/notifications` | Lista notificações do Cessionário | CESSIONARIO |
| `GET` | `/notifications/preferences` | Lê preferências de notificação | CESSIONARIO |
| `PATCH` | `/notifications/preferences` | Atualiza preferência de um tipo | CESSIONARIO |
| `PATCH` | `/notifications/{notification_id}/read` | Marca notificação como lida | CESSIONARIO |

---

#### `GET /notifications`

**Descrição:** Lista notificações proativas do Cessionário (alertas de oportunidade, Escrow, status).

**Query params:** `page`, `per_page`, `status` (`PENDING`, `DELIVERED`, `READ`, `FAILED`), `type` (`OPPORTUNITY`, `ESCROW_DEADLINE`, `STATUS_CHANGE`, `SYSTEM`)

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "type": "OPPORTUNITY",
      "status": "DELIVERED",
      "title": "Nova oportunidade compatível com seu perfil",
      "content": "OPR-2024-005678 — Δ = -22%, Score: Alto",
      "metadata": {
        "opportunity_id": "uuid",
        "deep_link": "/chat?alert=opportunity&opportunity_id=uuid"
      },
      "created_at": "2026-03-22T00:00:00-03:00",
      "read_at": null
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 5, "total_pages": 1 }
}
```

---

#### `GET /notifications/preferences`

**Response 200:**
```json
{
  "data": {
    "opportunity_alerts": true,
    "escrow_alerts": true,
    "status_alerts": true,
    "updated_at": "2026-03-22T00:00:00-03:00"
  }
}
```

---

#### `PATCH /notifications/preferences`

**Request body:**
```json
{
  "opportunity_alerts": false
}
```

Apenas campos a alterar precisam ser enviados. Campos ausentes são mantidos.

**Response 200:**
```json
{
  "data": {
    "opportunity_alerts": false,
    "escrow_alerts": true,
    "status_alerts": true,
    "updated_at": "2026-03-22T00:00:00-03:00"
  }
}
```

---

#### `PATCH /notifications/{notification_id}/read`

**Descrição:** Marca uma notificação como lida.

**Response 200:**
```json
{
  "data": {
    "id": "uuid",
    "status": "READ",
    "read_at": "2026-03-22T00:00:00-03:00"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `FORBIDDEN` | "Notificação não pertence ao Cessionário." | `notification_id` de outro cessionário |
| 404 | `NOTIFICATION_NOT_FOUND` | "Notificação não encontrada." | `notification_id` inexistente |

---

### 7.6 Domínio: Supervision (Admin)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/supervision/takeovers` | Inicia takeover de uma interação | ADMIN |
| `DELETE` | `/supervision/takeovers/{takeover_id}` | Encerra takeover | ADMIN |
| `POST` | `/supervision/takeovers/{takeover_id}/messages` | Envia mensagem em modo Admin (takeover ativo) | ADMIN |

---

#### `POST /supervision/takeovers`

**Descrição:** Inicia takeover de uma interação. Adquire mutex via `SELECT FOR UPDATE`.

**Headers:**
```
Authorization: Bearer {token}  // role: ADMIN
Content-Type: application/json
```

**Request body:**
```json
{
  "interaction_id": "uuid",
  "reason": "Confiança abaixo do threshold. Resposta incorreta sobre regras de KYC."
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `interaction_id` | UUID | Sim | ID da interação a assumir |
| `reason` | string | Sim | Motivo do takeover (mín: 10 chars, máx: 500 chars) |

**Response 201:**
```json
{
  "data": {
    "id": "uuid",
    "interaction_id": "uuid",
    "conversation_id": "uuid",
    "admin_id": "uuid",
    "reason": "Confiança abaixo do threshold...",
    "status": "ACTIVE",
    "started_at": "2026-03-22T00:00:00-03:00"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `INVALID_REASON` | "Motivo do takeover é obrigatório." | `reason` ausente ou < 10 chars |
| 403 | `FORBIDDEN` | "Acesso restrito a administradores." | Role não é ADMIN |
| 404 | `INTERACTION_NOT_FOUND` | "Interação não encontrada." | `interaction_id` inexistente |
| 409 | `TAKEOVER_ALREADY_ACTIVE` | "Esta interação já possui um takeover ativo." | Outro admin já assumiu |
| 422 | `TAKEOVER_NOT_ELIGIBLE` | "Interação não elegível para takeover." | Confiança acima do threshold |

**cURL:**
```bash
curl -X POST https://api.repasseseguro.com.br/repasse-ai/v1/supervision/takeovers \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{"interaction_id":"uuid","reason":"Resposta incorreta sobre KYC."}'
```

---

#### `DELETE /supervision/takeovers/{takeover_id}`

**Descrição:** Encerra o takeover e devolve o controle ao agente.

**Response 204:** Sem corpo.

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `FORBIDDEN` | "Somente o administrador que iniciou o takeover pode encerrá-lo." | Admin diferente do que iniciou |
| 404 | `TAKEOVER_NOT_FOUND` | "Takeover não encontrado." | `takeover_id` inexistente |
| 409 | `TAKEOVER_ALREADY_ENDED` | "Takeover já foi encerrado." | `status` != `ACTIVE` |

---

#### `POST /supervision/takeovers/{takeover_id}/messages`

**Descrição:** Envia mensagem ao Cessionário em modo Admin durante takeover ativo. Resposta via SSE.

**Request body:**
```json
{
  "content": "Olá, estou assumindo a conversa para esclarecer sobre KYC. Precisa de mais R$ 5.000 no Escrow conforme regra XX."
}
```

**Response 200 (SSE):** Mesmo formato do endpoint de chat, com `role: ADMIN` no evento `done`.

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 404 | `TAKEOVER_NOT_FOUND` | "Takeover não encontrado." | `takeover_id` inexistente |
| 409 | `TAKEOVER_NOT_ACTIVE` | "Takeover não está ativo." | `status` != `ACTIVE` |

---

### 7.7 Domínio: WhatsApp

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/whatsapp/bindings` | Inicia vinculação de número WhatsApp | CESSIONARIO |
| `POST` | `/whatsapp/bindings/verify` | Verifica OTP e confirma vinculação | CESSIONARIO |
| `GET` | `/whatsapp/bindings/me` | Retorna vinculação ativa do Cessionário | CESSIONARIO |
| `DELETE` | `/whatsapp/bindings/me` | Desvincula número WhatsApp | CESSIONARIO |

---

#### `POST /whatsapp/bindings`

**Descrição:** Inicia o fluxo de vinculação enviando OTP SMS para o número informado.

**Request body:**
```json
{
  "phone_number": "+5511987654321"
}
```

| Campo | Tipo | Obrigatório | Validação |
|---|---|---|---|
| `phone_number` | string | Sim | E.164 format: `+55{DDD}{número}` (11 dígitos após +55) |

**Response 201:**
```json
{
  "data": {
    "id": "uuid",
    "phone_number": "+5511987654321",
    "status": "OTP_SENT",
    "otp_expires_at": "2026-03-22T00:05:00-03:00",
    "attempts_remaining": 3
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `INVALID_PHONE` | "Número de telefone inválido. Use o formato +55XXXXXXXXXXX." | Formato E.164 incorreto |
| 409 | `PHONE_ALREADY_BOUND` | "Este número já está vinculado a outro Cessionário." | Número em uso |
| 409 | `BINDING_ALREADY_ACTIVE` | "Já existe uma vinculação ativa para este Cessionário." | Cessionário já tem WhatsApp vinculado |
| 429 | `OTP_RATE_LIMIT` | "Muitas tentativas. Aguarde 15 minutos." | > 3 OTPs solicitados em 15 min |

---

#### `POST /whatsapp/bindings/verify`

**Descrição:** Verifica o OTP SMS e avança para confirmação pelo WhatsApp (Step 3).

**Request body:**
```json
{
  "phone_number": "+5511987654321",
  "otp_code": "123456"
}
```

**Response 200:**
```json
{
  "data": {
    "id": "uuid",
    "phone_number": "+5511987654321",
    "status": "PENDING_WHATSAPP",
    "whatsapp_confirmation_timeout": "2026-03-22T00:10:00-03:00"
  }
}
```

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `INVALID_OTP` | "Código OTP inválido." | OTP incorreto |
| 400 | `OTP_EXPIRED` | "Código OTP expirado. Solicite um novo código." | OTP expirado (> 5 min) |
| 409 | `OTP_MAX_ATTEMPTS` | "Número de tentativas esgotado. Aguarde 15 minutos." | 3 tentativas inválidas |

---

#### `GET /whatsapp/bindings/me`

**Response 200:**
```json
{
  "data": {
    "id": "uuid",
    "phone_number": "+5511987654321",
    "status": "ACTIVE",
    "bound_at": "2026-03-21T10:00:00-03:00",
    "updated_at": "2026-03-21T10:00:00-03:00"
  }
}
```

**Response 200 (sem vinculação):**
```json
{
  "data": null
}
```

---

#### `DELETE /whatsapp/bindings/me`

**Descrição:** Remove a vinculação WhatsApp ativa do Cessionário (soft delete).

**Response 204:** Sem corpo.

**Erros:**
| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 404 | `BINDING_NOT_FOUND` | "Nenhuma vinculação ativa encontrada." | Sem vinculação ativa |

---

## 8. Webhooks

### 8.1 Webhook: EvolutionAPI → Repasse AI

O Repasse AI recebe webhooks do EvolutionAPI quando o usuário envia mensagem confirmando vinculação no WhatsApp (Step 3 do fluxo de binding).

**Endpoint receptor (interno, não exposto publicamente):**
```
POST /internal/webhooks/whatsapp
```

> ⚙️ **Segurança:** Este endpoint é protegido por IP allowlist (apenas IPs do EvolutionAPI) + HMAC-SHA256 signature no header `X-Evolution-Signature`.

**Evento: Confirmação de vinculação (`message.upsert`)**

```json
{
  "event": "message.upsert",
  "instance": "repasse-ai-prod",
  "data": {
    "key": {
      "remoteJid": "5511987654321@s.whatsapp.net",
      "fromMe": false,
      "id": "ABCD1234"
    },
    "message": {
      "conversation": "CONFIRMAR"
    },
    "messageTimestamp": 1750000000
  }
}
```

**Processamento:** o serviço valida a assinatura, extrai o `phone_number`, busca a vinculação em estado `PENDING_WHATSAPP` e atualiza para `ACTIVE`.

**Retry policy:** EvolutionAPI faz 3 tentativas com backoff exponencial (1s, 2s, 4s). Após 3 falhas, evento vai para dead-letter.

**Verificação de assinatura:**
```
X-Evolution-Signature: sha256=HMAC_SHA256(payload_body, EVOLUTION_WEBHOOK_SECRET)
```

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 22/03/2026 | v1.0 | Versão inicial. 22 endpoints em 6 domínios + health check + webhook. Contratos completos com request, response, erros e cURL. |

---

## 10. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Validação de token JWT durante SSE ativo | [DECISÃO AUTÔNOMA] | 2.3 Autenticação | Token validado apenas na abertura do SSE, não a cada token de streaming. Interromper SSE por expiração seria UX destrutiva em análises longas. Alternativa descartada: re-validação por heartbeat (overhead sem benefício de segurança real em TTL de 1h). | P1 | Backend Lead | Decidido |
| Limite de 10 oportunidades por portfólio | [DECISÃO AUTÔNOMA] | 7.4 Calculator | Limite de 10 garante latência em tempo real (< 500ms de cálculo). Mais de 10 exigiria processamento batch assíncrono. Alternativa descartada: sem limite (latência imprevisível impacta UX). | P1 | Backend Lead | Decidido |
| Webhook EvolutionAPI como endpoint interno | [DECISÃO AUTÔNOMA] | 8. Webhooks | Endpoint `/internal/webhooks/whatsapp` protegido por IP allowlist + HMAC, sem autenticação JWT. Justificativa: webhooks externos não carregam JWT; proteção por IP + signature é padrão de mercado. Alternativa descartada: autenticação JWT (EvolutionAPI não suporta). | P0 | Backend Lead / DevOps | Decidido |

---

*Próximo documento do pipeline: D17 — Integrações Externas.*
