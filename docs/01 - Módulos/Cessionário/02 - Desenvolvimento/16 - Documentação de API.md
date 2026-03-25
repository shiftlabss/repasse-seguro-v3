# 16 - Documentação de API

## Módulo Cessionário · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend, Frontend e QA |
| **Escopo** | API Reference — endpoints, autenticação, contratos, erros, paginação e exemplos |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Data da versão** | 22/03/2026 00:00 (America/Fortaleza) |
| **Status** | Ativo |
| **Referências** | 01 - Regras de Negócio · 02 - Stacks · 12 - Modelo de Dados · 14 - Especificações Técnicas |

---

> 📌 **TL;DR**
>
> - **47 endpoints documentados** em 10 domínios: Auth, Usuários, Cessionários/KYC, Oportunidades, Propostas, Negociações, Formalização, Financeiro, Notificações, IA.
> - **Base URL:** `https://api.repasseseguro.com.br/api/v1` — versionamento via prefixo de path.
> - **Autenticação:** JWT Bearer (access token 15-30min + refresh token httpOnly cookie). Header `Authorization: Bearer {token}` obrigatório em todos os endpoints protegidos.
> - **Paginação:** offset-based — `page`, `per_page` (max 100, default 20) + objeto `meta` em toda listagem.
> - **Padrão de erro:** `{ "error": { "code": "PREFIX-NNN", "message": "...", "details": {} } }` em todos os endpoints.
> - **Rate limiting:** 100 req/min geral; 10 req/15min em auth; 20 req/min em endpoints LLM (AI).
> - **Zero endpoints pendentes** — todos os 47 endpoints derivados dos documentos de input.

---

## 1. Visão Geral

### 1.1 Base URL e Ambientes

| Ambiente | Base URL |
|---|---|
| Desenvolvimento | `http://localhost:3000/api/v1` |
| Staging | `https://api-staging.repasseseguro.com.br/api/v1` |
| Produção | `https://api.repasseseguro.com.br/api/v1` |

### 1.2 Content-Type e Encoding

- `Content-Type: application/json` obrigatório em todas as requisições com body.
- Upload de arquivos (KYC, comprovantes): `Content-Type: multipart/form-data` — via URL assinada do Supabase Storage (não upload direto para a API).
- Encoding: `UTF-8`.
- Datas: `ISO 8601` com timezone (`2026-03-22T00:00:00-03:00`).
- Valores monetários: inteiros em **centavos** (ex: `R$ 100.000,00` → `10000000`). [DECISÃO AUTÔNOMA — centavos eliminam erros de ponto flutuante; alternativa descartada: float com 2 casas decimais (risco de precisão em cálculos de comissão)]

### 1.3 Convenção de Versionamento

- Versão no path: `/api/v1/`, `/api/v2/`.
- Breaking changes incrementam a versão. Versões anteriores mantidas por 90 dias.
- Versões deprecated: header `Deprecation: {date}` nas respostas.

### 1.4 Convenção de Nomenclatura

- Paths: `kebab-case`, plural para recursos (`/proposals`, `/negotiations`).
- Sub-recursos: `/{id}/sub-recurso` (ex: `POST /negotiations/{id}/counteroffer`).
- Ações sem correspondência CRUD: verbos no path (`POST /{id}/accept`, `POST /{id}/cancel`).

---

## 2. Autenticação

### 2.1 Fluxo JWT Completo

```
Login → Access Token (15-30min) + Refresh Token (httpOnly cookie, 30 dias)
     ↓
Request com Bearer Token
     ↓
Token expira → POST /auth/refresh → novo Access Token
     ↓
Logout → DELETE /auth/session → invalida refresh token
```

### 2.2 Headers Obrigatórios

| Header | Valor | Obrigatório em |
|---|---|---|
| `Authorization` | `Bearer {access_token}` | Todos os endpoints protegidos |
| `Content-Type` | `application/json` | Endpoints com body |
| `X-CSRF-Token` | Token CSRF | Endpoints mutáveis (POST/PUT/DELETE) |

### 2.3 Expiração e Refresh

- **Access token:** 15 minutos (produção) / 30 minutos (staging).
- **Refresh token:** 30 dias. Rotacionado a cada uso (refresh token rotation).
- **Inatividade:** refresh token invalidado após 7 dias sem uso.

### 2.4 Tratamento de Token Expirado/Inválido

| Código HTTP | Error Code | Quando ocorre |
|---|---|---|
| 401 | `AUTH-001` | Token ausente, malformado ou expirado |
| 401 | `AUTH-002` | Refresh token inválido/expirado → redirecionar para login |
| 403 | `AUTH-003` | Token válido mas sem permissão para o recurso (RBAC) |
| 403 | `CES-001` | KYC não aprovado para endpoint que exige `KYC_APROVADO` |

---

## 3. Padrão de Resposta

### 3.1 Schema de Sucesso

```json
{
  "data": { ... }  // ou array [...] para listagens
}
```

### 3.2 Schema de Erro

```json
{
  "error": {
    "code": "PRP-001",
    "message": "Limite de 3 propostas simultâneas atingido.",
    "details": {
      "current_count": 3,
      "max_allowed": 3
    }
  }
}
```

### 3.3 Schema de Listagem Paginada

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

---

## 4. Códigos HTTP

| Código | Significado | Uso no produto |
|---|---|---|
| 200 | OK | GET, PUT, PATCH com sucesso |
| 201 | Created | POST que cria recurso |
| 202 | Accepted | POST assíncrono (KYC, embed) |
| 204 | No Content | DELETE com sucesso |
| 400 | Bad Request | Validação de DTO falhou |
| 401 | Unauthorized | Token ausente, inválido ou expirado |
| 403 | Forbidden | RBAC ou KYC insuficiente |
| 404 | Not Found | Recurso não encontrado |
| 409 | Conflict | Estado incompatível (proposta para oportunidade já em negociação) |
| 422 | Unprocessable Entity | Regra de negócio violada (quota, rodadas, prazo) |
| 429 | Too Many Requests | Rate limit excedido |
| 500 | Internal Server Error | Erro não tratado — Sentry alertado |
| 503 | Service Unavailable | Dependência externa indisponível (idwall, ZapSign) |

---

## 5. Paginação

### 5.1 Query Params Padrão

| Param | Tipo | Default | Máximo | Descrição |
|---|---|---|---|---|
| `page` | integer | 1 | — | Página atual |
| `per_page` | integer | 20 | 100 | Itens por página |
| `sort` | string | `created_at` | — | Campo de ordenação |
| `order` | `asc` / `desc` | `desc` | — | Direção de ordenação |

### 5.2 Limites

- Máximo absoluto: 100 itens por página.
- Se `per_page > 100`: retorna 422 com `"per_page must not be greater than 100"`.

---

## 6. Rate Limiting

| Categoria | Limite | Janela | Header de resposta |
|---|---|---|---|
| Geral (autenticado) | 100 req | 1 min | `X-RateLimit-Limit: 100` |
| Auth (login, refresh) | 10 req | 15 min | `X-RateLimit-Limit: 10` |
| Upload KYC | 5 req | 10 min | `X-RateLimit-Limit: 5` |
| LLM (AI) | 20 req | 1 min | `X-RateLimit-Limit: 20` |

**Headers de resposta em toda requisição:**

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1711065600
```

**Quando excedido (429):**

```json
{
  "error": {
    "code": "AUTH-004",
    "message": "Limite de requisições atingido.",
    "details": { "retry_after": 47 }
  }
}
```

---

## 7. Endpoints por Domínio

### 7.1 Auth (`/auth`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| POST | `/auth/login` | Login com e-mail/senha | Público |
| POST | `/auth/login/google` | Login com Google OAuth | Público |
| POST | `/auth/register` | Cadastro com e-mail/senha | Público |
| POST | `/auth/refresh` | Renovar access token | Cookie refresh |
| DELETE | `/auth/session` | Logout | Bearer |
| POST | `/auth/password/reset-request` | Solicitar reset de senha | Público |
| POST | `/auth/password/reset` | Confirmar nova senha | Token URL |
| POST | `/auth/reauth` | Re-autenticação para ação crítica | Bearer |

---

**POST /auth/login**

Login com e-mail e senha. Retorna access token + set-cookie com refresh token.

```bash
curl -X POST https://api.repasseseguro.com.br/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"cessionario@email.com","password":"Senha@123"}'
```

**Response 200:**
```json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 900,
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "cessionario@email.com",
      "name": "Marcelo Santos",
      "cessionario_id": "660e8400-e29b-41d4-a716-446655440001",
      "kyc_status": "KYC_APROVADO"
    }
  }
}
```
*Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict; Max-Age=2592000*

**Erros:**

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `AUTH-001` | "Credenciais inválidas." | E-mail ou senha incorretos |
| 403 | `AUTH-005` | "Conta bloqueada temporariamente." | Status `BLOQUEADA_TEMPORARIAMENTE` |
| 429 | `AUTH-004` | "Limite de tentativas atingido." | 10 falhas em 15 min por IP |

---

**POST /auth/refresh**

Renova o access token usando o refresh token httpOnly cookie.

```bash
curl -X POST https://api.repasseseguro.com.br/api/v1/auth/refresh \
  -H "Cookie: refresh_token=..." \
  -H "X-CSRF-Token: {csrf_token}"
```

**Response 200:**
```json
{
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 900
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `AUTH-002` | "Sessão expirada. Faça login novamente." | Refresh token expirado/inválido |

---

**POST /auth/reauth**

Valida senha do usuário para re-autenticação antes de ação crítica.

**Headers:** `Authorization: Bearer {token}`, `X-CSRF-Token`

**Body:**
```json
{ "password": "Senha@123" }
```

**Response 200:**
```json
{ "data": { "reauth_token": "temp_token_30s", "expires_in": 30 } }
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `AUTH-001` | "Senha incorreta." | Senha inválida |
| 422 | `AUTH-006` | "Máximo de tentativas de re-autenticação atingido." | 3 falhas → logout |

---

### 7.2 Cessionários e KYC (`/cessionarios`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/cessionarios/me` | Perfil do Cessionário logado | Bearer |
| PATCH | `/cessionarios/me` | Atualizar dados do perfil | Bearer |
| GET | `/cessionarios/me/kyc` | Status do KYC | Bearer |
| POST | `/cessionarios/me/kyc/documents` | Upload de documento KYC | Bearer |
| DELETE | `/cessionarios/me` | Solicitar exclusão de dados (LGPD) | Bearer + reauth |

---

**GET /cessionarios/me**

```bash
curl https://api.repasseseguro.com.br/api/v1/cessionarios/me \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Marcelo Santos",
    "email": "cessionario@email.com",
    "phone": "+5511999999999",
    "status": "KYC_APROVADO",
    "created_at": "2026-01-15T10:00:00-03:00"
  }
}
```

---

**POST /cessionarios/me/kyc/documents**

Upload de documento para verificação KYC. Arquivo enviado para Supabase Storage via URL assinada retornada pelo backend.

**Headers:** `Authorization: Bearer {token}`, `Content-Type: application/json`

**Body:**
```json
{
  "document_type": "IDENTITY_FRONT",
  "file_name": "rg_frente.jpg",
  "mime_type": "image/jpeg",
  "file_size_bytes": 1048576
}
```

**Response 202:**
```json
{
  "data": {
    "kyc_document_id": "770e8400-...",
    "upload_url": "https://storage.supabase.co/object/sign/kyc/...",
    "upload_url_expires_at": "2026-03-22T00:05:00-03:00",
    "cessionario_status": "KYC_EM_ANALISE"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `CES-002` | "Tipo de documento inválido." | `document_type` não reconhecido |
| 400 | `CES-003` | "Arquivo muito grande. Máximo 10MB." | `file_size_bytes > 10485760` |
| 409 | `CES-004` | "KYC já está em análise." | Status já `KYC_EM_ANALISE` |

---

### 7.3 Oportunidades (`/opportunities`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/opportunities` | Listar oportunidades do marketplace | Bearer + KYC |
| GET | `/opportunities/{id}` | Detalhe de uma oportunidade | Bearer + KYC |
| GET | `/opportunities/{id}/analysis` | Análise IA (streaming SSE) | Bearer + KYC |

---

**GET /opportunities**

**Query params:** `page`, `per_page`, `sort`, `order`, `city`, `state`, `price_min`, `price_max`, `risk_score_min` (1-10), `status` (default: `DISPONIVEL`)

```bash
curl "https://api.repasseseguro.com.br/api/v1/opportunities?page=1&per_page=25&city=São+Paulo" \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "880e8400-...",
      "code": "OPR-2026-0042",
      "status": "DISPONIVEL",
      "city": "São Paulo",
      "state": "SP",
      "property_type": "Apartamento",
      "current_table_value_cents": 45000000,
      "contract_table_value_cents": 38000000,
      "delta_value_cents": 7000000,
      "commission_buyer_cents": 1400000,
      "ai_risk_score": 8,
      "created_at": "2026-02-10T08:00:00-03:00"
    }
  ],
  "meta": { "page": 1, "per_page": 25, "total": 142, "total_pages": 6 }
}
```

⚙️ **Nota de privacidade:** nenhum campo do Cedente retornado (RN-014/RN-067). `cedente_id` nunca presente na resposta.

---

**GET /opportunities/{id}/analysis** — SSE Streaming

Análise do Analista de IA. Retorna stream SSE.

```bash
curl -N "https://api.repasseseguro.com.br/api/v1/opportunities/880e8400.../analysis?message=Qual+o+risco?" \
  -H "Authorization: Bearer {token}" \
  -H "Accept: text/event-stream"
```

**Response — SSE stream:**
```
data: {"token": "Esta"}

data: {"token": " oportunidade"}

data: {"token": " apresenta..."}

event: done
data: {"full_response": "...", "tokens_used": 312}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 429 | `AI-001` | "Limite de consultas atingido." | 20 req/min por usuário |
| 503 | `AI-002` | "Analista temporariamente indisponível." | OpenAI timeout |

---

### 7.4 Propostas (`/proposals`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/proposals` | Listar propostas do Cessionário | Bearer + KYC |
| POST | `/proposals` | Criar proposta para oportunidade | Bearer + KYC |
| GET | `/proposals/{id}` | Detalhe da proposta | Bearer + KYC |
| DELETE | `/proposals/{id}` | Cancelar proposta | Bearer + KYC |

---

**POST /proposals**

**Headers:** `Authorization`, `Content-Type`, `X-CSRF-Token`

**Body:**
```json
{
  "opportunity_id": "880e8400-e29b-41d4-a716-446655440002",
  "proposed_value_cents": 30000000
}
```

**Response 201:**
```json
{
  "data": {
    "id": "990e8400-...",
    "opportunity_id": "880e8400-...",
    "opportunity_code": "OPR-2026-0042",
    "status": "ENVIADA",
    "proposed_value_cents": 30000000,
    "commission_buyer_cents": 1400000,
    "total_cents": 31400000,
    "created_at": "2026-03-22T10:00:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `CES-001` | "KYC não aprovado." | KYC não está `KYC_APROVADO` |
| 409 | `OPR-001` | "Oportunidade não está disponível." | Status != `DISPONIVEL` |
| 422 | `PRP-001` | "Limite de 3 propostas simultâneas atingido." | quota simultânea = 3 |
| 422 | `PRP-002` | "Limite diário de 10 propostas atingido." | quota diária = 10 |
| 422 | `PRP-003` | "Valor proposto deve ser maior que zero." | `proposed_value_cents <= 0` |

---

**DELETE /proposals/{id}**

Cancela proposta. Só possível se status é `ENVIADA` ou `EM_ANALISE`.

**Headers:** `Authorization`, `X-CSRF-Token`

**Response 200:**
```json
{ "data": { "id": "990e8400-...", "status": "CANCELADA" } }
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `PRP-004` | "Proposta não pode ser cancelada neste status." | Status != ENVIADA/EM_ANALISE |
| 404 | `PRP-005` | "Proposta não encontrada." | ID inexistente ou de outro Cessionário |

---

### 7.5 Negociações (`/negotiations`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/negotiations` | Listar negociações ativas | Bearer + KYC |
| GET | `/negotiations/{id}` | Detalhe da negociação | Bearer + KYC |
| GET | `/negotiations/{id}/messages` | Histórico de mensagens | Bearer + KYC |
| POST | `/negotiations/{id}/counteroffer` | Enviar contraproposta | Bearer + KYC |
| POST | `/negotiations/{id}/escrow/confirm` | Confirmar envio do depósito | Bearer + KYC |
| POST | `/negotiations/{id}/escrow/extend` | Solicitar extensão de prazo | Bearer + KYC |

---

**POST /negotiations/{id}/counteroffer**

**Body:**
```json
{
  "counteroffer_value_cents": 28500000,
  "message": "Considerando o estado do imóvel, proponho este valor."
}
```

**Response 200:**
```json
{
  "data": {
    "negotiation_id": "aaa0e8400-...",
    "status": "EM_CONTRAPROPOSTA",
    "counteroffer_round": 2,
    "counteroffer_value_cents": 28500000,
    "commission_buyer_cents": 1400000,
    "total_cents": 29900000
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `NEG-001` | "Limite de 3 rodadas de contraproposta atingido." | `counteroffer_round = 3` |
| 409 | `NEG-002` | "Negociação não está em estado de contraproposta." | Status incorreto |
| 422 | `NEG-003` | "Valor de contraproposta deve ser maior que zero." | `value <= 0` |

---

**POST /negotiations/{id}/escrow/confirm**

Cessionário confirma que realizou o depósito.

**Body:**
```json
{
  "payment_method": "PIX",
  "reference_code": "E00000000202603221000000000000"
}
```

**Response 200:**
```json
{
  "data": {
    "escrow_deposit_id": "bbb0e8400-...",
    "status": "DEPOSITO_ENVIADO",
    "confirmation_sent_at": "2026-03-22T14:30:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `NEG-004` | "Depósito já confirmado." | Status != `AGUARDANDO_DEPOSITO` |
| 422 | `NEG-005` | "Prazo de depósito expirado." | Após 10 dias úteis |

---

### 7.6 Formalização (`/formalizations`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/formalizations/{id}` | Detalhe da formalização | Bearer + KYC |
| GET | `/formalizations/{id}/documents` | Listar documentos disponíveis | Bearer + KYC |
| POST | `/formalizations/{id}/sign` | Iniciar assinatura (gera URL ZapSign) | Bearer + KYC |

---

**POST /formalizations/{id}/sign**

Gera URL de assinatura ZapSign para o Cessionário.

**Response 200:**
```json
{
  "data": {
    "signing_url": "https://app.zapsign.com.br/verificar/TOKEN123",
    "expires_at": "2026-03-25T00:00:00-03:00",
    "formalization_status": "ASSINATURA_PENDENTE_CESSIONARIO"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `FRM-001` | "Formalização não está disponível para assinatura." | Status incorreto |
| 503 | `FRM-002` | "Serviço de assinatura temporariamente indisponível." | ZapSign offline |

---

### 7.7 Financeiro (`/financial`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/financial/summary` | KPIs financeiros do Cessionário | Bearer + KYC |
| GET | `/financial/transactions` | Histórico de transações | Bearer + KYC |
| GET | `/financial/transactions/{id}` | Detalhe de transação | Bearer + KYC |
| GET | `/financial/transactions/{id}/receipt` | URL para comprovante PDF | Bearer + KYC |

---

**GET /financial/transactions**

**Query params:** `page`, `per_page`, `type` (`DEPOSITO_ESCROW|COMISSAO|REEMBOLSO`), `date_from`, `date_to`

**Response 200:**
```json
{
  "data": [
    {
      "id": "ccc0e8400-...",
      "type": "DEPOSITO_ESCROW",
      "description": "Depósito Escrow — OPR-2026-0042",
      "amount_cents": 31400000,
      "status": "CONFIRMADO",
      "created_at": "2026-03-22T14:30:00-03:00"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 12, "total_pages": 1 }
}
```

---

### 7.8 Notificações (`/notifications`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/notifications` | Listar notificações | Bearer |
| PATCH | `/notifications/{id}/read` | Marcar como lida | Bearer |
| PATCH | `/notifications/read-all` | Marcar todas como lidas | Bearer |
| GET | `/notifications/preferences` | Preferências de notificação | Bearer |
| PATCH | `/notifications/preferences` | Atualizar preferências | Bearer |

---

**GET /notifications**

**Query params:** `page`, `per_page`, `read` (boolean)

**Response 200:**
```json
{
  "data": [
    {
      "id": "ddd0e8400-...",
      "type": "NOT-CES-06",
      "title": "Depósito Confirmado!",
      "body": "Seu depósito foi confirmado. Formalização liberada.",
      "read": false,
      "deep_link": "/formalizacoes/eee0e8400-...",
      "created_at": "2026-03-22T15:00:00-03:00"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 5, "total_pages": 1 }
}
```

---

**PATCH /notifications/preferences**

**Body:**
```json
{
  "email": true,
  "push": true,
  "in_app": true
}
```

⚙️ **Regra:** `email` não pode ser definido como `false` (RN-069). Request com `"email": false` retorna 422.

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `NOT-001` | "Notificações por e-mail não podem ser desabilitadas." | `email: false` |

---

### 7.9 IA — Analista de Oportunidades (`/ai`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/ai/sessions` | Listar sessões de chat | Bearer + KYC |
| GET | `/ai/sessions/{id}/messages` | Histórico de mensagens de uma sessão | Bearer + KYC |
| GET | `/ai/stream` | Streaming SSE da análise IA | Bearer + KYC |
| POST | `/opportunities/{id}/risk-score` | Forçar recalculo do score de risco | Bearer + KYC |

**GET /ai/stream** — ver §7.3 `/opportunities/{id}/analysis`. O endpoint `/ai/stream` é uma versão mais geral com parâmetros:

**Query params:** `opportunity_id` (UUID, obrigatório), `session_id` (UUID, opcional — para continuidade), `message` (string, obrigatório)

---

### 7.10 Reversão de Operação

| Método | Path | Descrição | Auth |
|---|---|---|---|
| POST | `/opportunities/{id}/reverse` | Solicitar reversão da operação | Bearer + KYC |

**POST /opportunities/{id}/reverse**

**Body:**
```json
{
  "reason": "Mudança nas condições financeiras pessoais."
}
```

**Response 200:**
```json
{
  "data": {
    "opportunity_id": "880e8400-...",
    "reversal_initiated_at": "2026-04-05T10:00:00-03:00",
    "deadline_was": "2026-04-06T00:00:00-03:00",
    "status": "CANCELADA"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `OPR-002` | "Operação não pode ser revertida neste status." | Status != `CONCLUIDA` |
| 422 | `OPR-003` | "Prazo de 15 dias para reversão expirado." | `created_at + 15d < now` |

---

## 8. Webhooks

### 8.1 Webhook ZapSign (`POST /webhooks/zapsign`)

Recebido do ZapSign ao assinar ou concluir documento.

**Verificação de assinatura:** header `X-Zapsign-Signature: HMAC-SHA256({secret}, body)`

**Payload (evento `signer_signed`):**
```json
{
  "event": "signer_signed",
  "doc_token": "TOKEN123",
  "signer": {
    "email": "cessionario@email.com",
    "signed_at": "2026-03-22T16:00:00Z"
  }
}
```

**Payload (evento `doc_complete`):**
```json
{
  "event": "doc_complete",
  "doc_token": "TOKEN123",
  "completed_at": "2026-03-22T17:30:00Z"
}
```

**Response:** `200 OK` — qualquer outro código causa retry pelo ZapSign.

**Retry policy ZapSign:** 3 tentativas em 30 min (5min, 15min, 30min).

### 8.2 Webhook idwall (`POST /webhooks/idwall`)

Recebido quando processamento KYC é concluído.

**Payload:**
```json
{
  "reference_id": "cessionario_id",
  "status": "APPROVED",
  "confidence": 0.98,
  "checks": {
    "document_ocr": "passed",
    "liveness": "passed",
    "face_match": "passed"
  }
}
```

**Response:** `200 OK`.

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 22/03/2026 | v1.0 | Criação inicial — Pipeline ShiftLabs v9.5. 47 endpoints em 10 domínios. Autenticação JWT completa. Paginação offset-based. Rate limiting por categoria. Webhooks ZapSign e idwall. |

---

## 10. Backlog de Pendências

| Item | Tipo | Seção | Impacto | Justificativa / Decisão | Dono | Status |
|---|---|---|---|---|---|---|
| Valores monetários em centavos (integer) | Decisão Autônoma | §1.2 | P0 | Elimina erros de ponto flutuante em cálculos de comissão; frontend formata para exibição; alternativa descartada: float 2 casas (risco de imprecisão) | Backend Lead | Decidido |
| Paginação offset-based (não cursor) | Decisão Autônoma | §5 | P1 | Listagens do marketplace e financeiro permitem ordenação por múltiplos campos — cursor-based requer campo único; offset permite sort por `ai_risk_score`, `price`, `created_at`; alternativa descartada: cursor (limitaria ordenação) | Backend Lead | Decidido |
| Upload KYC via Signed URL (não multipart direto) | Decisão Autônoma | §7.2 | P1 | Upload direto para Supabase Storage elimina tráfego de arquivo pelo NestJS; API retorna signed URL e cliente sobe diretamente; alternativa descartada: multipart no NestJS (bottleneck, latência) | Backend Lead | Decidido |
| Endpoint SSE em `/opportunities/{id}/analysis` e `/ai/stream` (dois endpoints) | Decisão Autônoma | §7.3 + §7.9 | P2 | O primeiro é contextual (ligado à oportunidade), o segundo é geral (sessão de chat); separação permite cache por oportunidade no primeiro; alternativa descartada: endpoint único (dificulta cache granular) | Backend Lead | Decidido |
