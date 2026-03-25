# 16 - Documentação de API

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Backend, Frontend e QA | API Reference do AI-Dani-Cessionário — endpoints, autenticação, contratos, erros, paginação e exemplos | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **18 endpoints documentados** distribuídos em 6 domínios: Agente/Chat, Calculadora, Alertas, WhatsApp (Fase 2), Oportunidades (read-only), Auth.
> - **Base URL:** `https://api.repasseseguro.com.br/api/v1` (prod) · `http://localhost:3000/api/v1` (local)
> - **Autenticação:** Bearer JWT herdado da plataforma Repasse Seguro. Access token 15min, refresh token 7 dias.
> - **Paginação:** `{ data: [...], meta: { page, per_page, total, total_pages } }`. `per_page` máximo: 100.
> - **Padrão de erro:** `{ error: { code: string, message: string, details?: object } }`
> - **Rate limiting:** 30 msgs/hora no chat (RN-DC-025) + 3 tentativas OTP/hora. Headers `X-RateLimit-*` em todos os endpoints.
> - **Zero endpoints pendentes por insuficiência de insumo.**

---

## 1. Visão Geral

### 1.1 Base URL e Ambientes

| Ambiente | Base URL |
|---|---|
| Produção | `https://api.repasseseguro.com.br/api/v1` |
| Staging | `https://api-staging.repasseseguro.com.br/api/v1` |
| Desenvolvimento local | `http://localhost:3000/api/v1` |

### 1.2 Content-Type e Encoding

- Todas as requisições: `Content-Type: application/json; charset=UTF-8`
- Exceção: endpoint SSE usa `Accept: text/event-stream`
- Charset: UTF-8 em todas as respostas

### 1.3 Versionamento

- Prefixo de versão em todos os endpoints: `/api/v1/`
- Breaking changes exigem nova versão (`/api/v2/`) com período de deprecação de 30 dias
- Cabeçalho `Deprecation: date` notifica deprecação antecipada

### 1.4 Convenção de Nomenclatura

- Paths em `kebab-case`: `/dani-chat`, `/dani-alertas`
- Domínio da Dani: prefixo `/dani` em todos os endpoints
- IDs como path params: `/{id}` usando UUID v4
- Ações não-CRUD: verbo no path — `/whatsapp/vincular`, `/whatsapp/verificar-otp`

---

## 2. Autenticação

### 2.1 Fluxo JWT

A Dani **herda a sessão da plataforma Repasse Seguro** — sem login próprio.

```
1. Cessionário faz login na plataforma Repasse Seguro
2. Plataforma retorna access_token (15min) + refresh_token (7 dias)
3. Frontend inclui access_token no header Authorization de todas as chamadas da Dani
4. Ao receber 401 UNAUTHORIZED: frontend usa refresh_token para renovar o access_token
5. Logout na plataforma invalida automaticamente o acesso à Dani
```

### 2.2 Headers Obrigatórios

```
Authorization: Bearer {access_token}
Content-Type: application/json
```

### 2.3 Renovação de Token

```bash
# POST /auth/refresh (plataforma principal — não é endpoint da Dani)
curl -X POST https://api.repasseseguro.com.br/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "eyJ..."}'
```

```json
{
  "data": {
    "access_token": "eyJ...",
    "expires_in": 900
  }
}
```

### 2.4 Tratamento de Token Inválido

| Cenário | HTTP | Error Code | Ação do Frontend |
|---|---|---|---|
| Token ausente | 401 | `AUTH_TOKEN_MISSING` | Redirecionar para login |
| Token expirado | 401 | `AUTH_JWT_EXPIRED` | Chamar `/auth/refresh` com refresh token |
| Token inválido | 401 | `AUTH_JWT_INVALID` | Redirecionar para login |
| Token sem role CESSIONARIO | 403 | `AUTH_INSUFFICIENT_ROLE` | Exibir mensagem de acesso negado |

---

## 3. Padrão de Resposta

### 3.1 Schema de Sucesso

```json
{
  "data": { /* payload da resposta */ }
}
```

Para listagens paginadas:
```json
{
  "data": [ /* array de itens */ ],
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### 3.2 Schema de Erro

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Mensagem legível para o usuário",
    "details": {}
  }
}
```

### 3.3 Campos Padrão

Todo response inclui os headers:
```
X-Request-Id: {uuid}
X-RateLimit-Limit: {limit}
X-RateLimit-Remaining: {remaining}
X-RateLimit-Reset: {epoch_timestamp}
```

---

## 4. Códigos HTTP

| Código | Significado geral | Contexto da Dani |
|---|---|---|
| `200` | OK | Resposta de sucesso (GET, PATCH) |
| `201` | Created | Recurso criado (POST que cria entidade) |
| `204` | No Content | Operação bem-sucedida sem body (DELETE, PATCH silencioso) |
| `400` | Bad Request | Dados de entrada inválidos (validação de DTO) |
| `401` | Unauthorized | Token ausente, expirado ou inválido |
| `403` | Forbidden | Token válido mas sem permissão (role insuficiente, RBAC) |
| `404` | Not Found | Recurso não encontrado ou não pertence ao Cessionário autenticado |
| `409` | Conflict | Estado conflitante (ex: já vinculado ao WhatsApp) |
| `422` | Unprocessable Entity | Dados válidos sintaticamente mas inválidos semanticamente (ex: OPR encerrada) |
| `429` | Too Many Requests | Rate limit atingido |
| `500` | Internal Server Error | Erro inesperado do servidor |
| `502` | Bad Gateway | Serviço externo (OpenAI, EvolutionAPI) indisponível |
| `504` | Gateway Timeout | Timeout no upstream (SSE/OpenAI) — fallback ativado |

---

## 5. Paginação

### 5.1 Query Params Padrão

| Param | Tipo | Default | Máximo | Descrição |
|---|---|---|---|---|
| `page` | integer | 1 | — | Página atual |
| `per_page` | integer | 20 | 100 | Itens por página |
| `sort` | string | `created_at` | — | Campo de ordenação |
| `order` | `asc` \| `desc` | `desc` | — | Direção de ordenação |

### 5.2 Objeto Meta

```json
{
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 45,
    "total_pages": 3
  }
}
```

---

## 6. Rate Limiting

| Categoria | Limite | Janela | Header de Reset |
|---|---|---|---|
| Chat (enviar mensagem) | 30 msgs | 1 hora deslizante | `X-RateLimit-Reset: {epoch}` |
| OTP WhatsApp | 3 tentativas | 1 hora | `X-RateLimit-Reset: {epoch}` |
| Hard block OTP | 5 falhas consecutivas → 30min bloqueado | 30 min | `X-RateLimit-Reset: {epoch}` |
| Demais endpoints | 1000 req | 1 hora por IP | `X-RateLimit-Reset: {epoch}` |

**Response quando limite atingido:**
```json
HTTP 429 Too Many Requests
{
  "error": {
    "code": "AGENTE_RATE_LIMIT_EXCEEDED",
    "message": "Você atingiu o limite de mensagens por hora.",
    "details": {
      "reset_at": "2026-03-23T15:30:00Z",
      "limit": 30,
      "window_seconds": 3600
    }
  }
}
```

---

## 7. Endpoints por Domínio

### 7.1 Domínio: Agente / Chat

**Tabela resumo:**

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/dani/chat` | Envia mensagem ao agente (não-streaming) | Bearer |
| `GET` | `/dani/stream` | Abre canal SSE para streaming de resposta | Bearer |
| `GET` | `/dani/conversas` | Lista conversas do Cessionário | Bearer |
| `GET` | `/dani/conversas/{conversa_id}/mensagens` | Lista mensagens de uma conversa | Bearer |
| `POST` | `/dani/conversas` | Cria nova conversa | Bearer |
| `PATCH` | `/dani/conversas/{conversa_id}/encerrar` | Encerra uma conversa | Bearer |

---

#### `POST /dani/chat`

Envia mensagem ao agente Dani (response completo, sem streaming).

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "mensagem": "Analise a oportunidade OPR-2024-0042",
  "conversa_id": "uuid-conversa",
  "opr_context": {
    "opr_id": "OPR-2024-0042"
  }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `mensagem` | string (1–1000 chars) | ✅ | Texto da mensagem |
| `conversa_id` | string (UUID) | ❌ | Se omitido, cria nova conversa |
| `opr_context.opr_id` | string | ❌ | Injetar contexto de OPR na conversa |

**Response `200`:**
```json
{
  "data": {
    "mensagem_id": "uuid-mensagem",
    "conversa_id": "uuid-conversa",
    "resposta": "Análise da oportunidade OPR-2024-0042: Δ de R$ 80.000,00...",
    "remetente": "DANI",
    "metadata": {
      "tipo_resposta": "analise",
      "opr_ids": ["OPR-2024-0042"],
      "confianca_modelo": 0.92,
      "latency_ms": 2340,
      "fallback_ativo": false
    },
    "created_at": "2026-03-23T14:00:00Z"
  }
}
```

**Erros:**

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `AGENTE_MENSAGEM_VAZIA` | "A mensagem não pode estar vazia." | `mensagem` ausente ou vazia |
| 400 | `AGENTE_MENSAGEM_TOO_LONG` | "Mensagem deve ter no máximo 1000 caracteres." | `mensagem.length > 1000` |
| 404 | `AGENTE_CONVERSA_NOT_FOUND` | "Conversa não encontrada." | `conversa_id` inexistente ou de outro Cessionário |
| 429 | `AGENTE_RATE_LIMIT_EXCEEDED` | "Limite de mensagens atingido." | > 30 msgs/hora |
| 502 | `AGENTE_LLM_UNAVAILABLE` | "Serviço de IA temporariamente indisponível." | OpenAI indisponível (fallback retorna 200 com `fallback_ativo: true`) |

**Exemplo cURL:**
```bash
curl -X POST https://api.repasseseguro.com.br/api/v1/dani/chat \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{"mensagem": "Analise OPR-2024-0042", "conversa_id": "550e8400-e29b-41d4-a716-446655440000"}'
```

---

#### `GET /dani/stream`

Abre canal SSE para streaming token a token da resposta da Dani.

**Headers:**
```
Authorization: Bearer {access_token}
Accept: text/event-stream
```

**Query Params:**

| Param | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `mensagem` | string | ✅ | Texto URL-encoded |
| `conversa_id` | string (UUID) | ❌ | Conversa existente |
| `opr_id` | string | ❌ | Injetar contexto de OPR |

**Response Stream (`200 text/event-stream`):**
```
data: {"type":"token","content":"Anali"}
data: {"type":"token","content":"sando a"}
data: {"type":"token","content":" oportunidade..."}
data: {"type":"metadata","payload":{"tipo_resposta":"analise","latency_ms":2100}}
data: {"type":"done","mensagem_id":"uuid-mensagem","conversa_id":"uuid-conversa"}
```

**Eventos SSE:**

| Tipo | Payload | Quando |
|---|---|---|
| `token` | `{ content: string }` | Cada token recebido do LLM |
| `metadata` | `{ tipo_resposta, opr_ids, confianca_modelo, latency_ms, fallback_ativo }` | Ao finalizar resposta |
| `done` | `{ mensagem_id, conversa_id }` | Stream concluído |
| `error` | `{ code, message }` | Erro durante streaming |
| `fallback` | `{ banner: "FALLBACK" \| "OFFLINE" }` | Quando Calculadora assume |

**Erros:**

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `AGENTE_MENSAGEM_VAZIA` | "mensagem obrigatória." | Query param ausente |
| 429 | `AGENTE_RATE_LIMIT_EXCEEDED` | "Limite de mensagens atingido." | > 30 msgs/hora |
| 504 | `AGENTE_STREAM_TIMEOUT` | "Timeout na resposta." | LLM > 30s (frontend recebe evento `error`) |

---

#### `GET /dani/conversas`

Lista conversas do Cessionário autenticado.

**Query Params:** `page`, `per_page`, `status` (`ATIVA` | `ENCERRADA` | `EXPIRADA`), `canal` (`WEBCHAT` | `WHATSAPP`)

**Response `200`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "canal": "WEBCHAT",
      "status": "ATIVA",
      "expires_at": "2026-06-21T14:00:00Z",
      "ultima_mensagem_at": "2026-03-23T14:00:00Z",
      "created_at": "2026-03-23T10:00:00Z"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 5, "total_pages": 1 }
}
```

---

#### `GET /dani/conversas/{conversa_id}/mensagens`

Lista mensagens de uma conversa. Filtradas por `cessionario_id` automaticamente.

**Query Params:** `page`, `per_page` (max 100), `order` (default: `asc`)

**Response `200`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "conversa_id": "uuid",
      "remetente": "CESSIONARIO",
      "conteudo": "Analise OPR-2024-0042",
      "metadata": null,
      "created_at": "2026-03-23T14:00:00Z"
    },
    {
      "id": "uuid",
      "conversa_id": "uuid",
      "remetente": "DANI",
      "conteudo": "Análise da oportunidade OPR-2024-0042...",
      "metadata": {
        "tipo_resposta": "analise",
        "opr_ids": ["OPR-2024-0042"],
        "confianca_modelo": 0.92,
        "latency_ms": 2340,
        "fallback_ativo": false
      },
      "created_at": "2026-03-23T14:00:02Z"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 2, "total_pages": 1 }
}
```

**Erros:**

| Código | Error Code | Quando ocorre |
|---|---|---|
| 404 | `AGENTE_CONVERSA_NOT_FOUND` | Conversa inexistente ou de outro Cessionário |

---

#### `POST /dani/conversas`

Cria uma nova conversa.

**Request Body:**
```json
{
  "canal": "WEBCHAT",
  "opr_context": { "opr_id": "OPR-2024-0042" }
}
```

**Response `201`:**
```json
{
  "data": {
    "id": "uuid",
    "canal": "WEBCHAT",
    "status": "ATIVA",
    "expires_at": "2026-06-21T14:00:00Z",
    "created_at": "2026-03-23T14:00:00Z"
  }
}
```

---

#### `PATCH /dani/conversas/{conversa_id}/encerrar`

Encerra uma conversa ativa.

**Response `200`:**
```json
{
  "data": { "id": "uuid", "status": "ENCERRADA", "updated_at": "2026-03-23T15:00:00Z" }
}
```

**Erros:**

| Código | Error Code | Quando ocorre |
|---|---|---|
| 404 | `AGENTE_CONVERSA_NOT_FOUND` | Conversa inexistente |
| 422 | `AGENTE_CONVERSA_JA_ENCERRADA` | Conversa já encerrada ou expirada |

---

### 7.2 Domínio: Calculadora

**Tabela resumo:**

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/calculadora/calcular` | Executa cálculo determinístico de comissão/ROI | Bearer |

---

#### `POST /calculadora/calcular`

Executa cálculo determinístico sem depender da IA (Calculadora de Comissão).

**Request Body:**
```json
{
  "opr_id": "OPR-2024-0042",
  "valor_proposta": 350000.00
}
```

**Response `200`:**
```json
{
  "data": {
    "opr_id": "OPR-2024-0042",
    "valor_proposta": 350000.00,
    "delta": 80000.00,
    "comissao": 16000.00,
    "custo_total_escrow": 366000.00,
    "roi": {
      "conservador": { "percentual": 12.5, "valor_estimado": 396000.00 },
      "base":         { "percentual": 21.8, "valor_estimado": 430000.00 },
      "otimista":     { "percentual": 35.0, "valor_estimado": 477000.00 }
    },
    "formula_aplicada": "20_PERCENT_DELTA",
    "source": "computed",
    "disclaimer": "Esses são valores projetados com base nos dados disponíveis. Resultados reais podem variar."
  }
}
```

| Campo `formula_aplicada` | Quando |
|---|---|
| `20_PERCENT_DELTA` | Δ > 0: `comissao = 0.20 * delta` |
| `20_PERCENT_VALOR_PAGO` | Δ ≤ 0: `comissao = 0.20 * valor_pago_cedente` |

**Erros:**

| Código | Error Code | Quando ocorre |
|---|---|---|
| 400 | `CALC_INVALID_VALUE` | `valor_proposta` ≤ 0 ou não-numérico |
| 404 | `OPR_NOT_FOUND` | OPR inexistente no marketplace |
| 422 | `CALC_DADOS_INSUFICIENTES` | OPR sem tabela_atual ou tabela_contrato |

**Exemplo cURL:**
```bash
curl -X POST https://api.repasseseguro.com.br/api/v1/calculadora/calcular \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{"opr_id": "OPR-2024-0042", "valor_proposta": 350000}'
```

---

### 7.3 Domínio: Alertas

**Tabela resumo:**

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/dani/alertas` | Lista alertas do Cessionário | Bearer |
| `GET` | `/dani/alertas/count` | Contagem de alertas não lidos (badge FAB) | Bearer |
| `PATCH` | `/dani/alertas/{alerta_id}/lido` | Marca alerta como lido | Bearer |

---

#### `GET /dani/alertas`

Lista alertas do Cessionário autenticado.

**Query Params:** `page`, `per_page`, `status` (`PENDENTE` | `ENVIADO` | `LIDO` | `FALHA`), `tipo`

**Response `200`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "tipo": "NOVA_OPORTUNIDADE",
      "status": "ENVIADO",
      "payload": {
        "opr_id": "OPR-2024-0043",
        "mensagem": "Nova oportunidade disponível: apartamento 2BR em Fortaleza.",
        "link_acao": "/cessionario/oportunidade/OPR-2024-0043?chat=open"
      },
      "lido_em": null,
      "created_at": "2026-03-23T10:00:00Z"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 3, "total_pages": 1 }
}
```

---

#### `GET /dani/alertas/count`

Contagem de alertas não lidos — usado para badge do FAB.

**Response `200`:**
```json
{
  "data": { "nao_lidos": 3 }
}
```

---

#### `PATCH /dani/alertas/{alerta_id}/lido`

Marca alerta como lido.

**Response `200`:**
```json
{
  "data": {
    "id": "uuid",
    "status": "LIDO",
    "lido_em": "2026-03-23T14:30:00Z"
  }
}
```

**Erros:**

| Código | Error Code | Quando ocorre |
|---|---|---|
| 404 | `ALERTA_NOT_FOUND` | Alerta inexistente ou de outro Cessionário |
| 422 | `ALERTA_JA_LIDO` | Alerta já marcado como lido |

---

### 7.4 Domínio: WhatsApp (Fase 2)

**Tabela resumo:**

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/whatsapp/status` | Status atual de vinculação | Bearer |
| `POST` | `/whatsapp/vincular` | Inicia vinculação — envia OTP SMS | Bearer |
| `POST` | `/whatsapp/verificar-otp` | Verifica OTP e avança estado | Bearer |
| `DELETE` | `/whatsapp/vincular` | Desvincula WhatsApp | Bearer |
| `POST` | `/whatsapp/webhook` | Webhook EvolutionAPI — mensagens recebidas | API Key |

---

#### `GET /whatsapp/status`

Retorna estado atual de vinculação do Cessionário.

**Response `200`:**
```json
{
  "data": {
    "estado": "VINCULADO",
    "numero_sufixo": "4521",
    "vinculado_em": "2026-03-20T10:00:00Z"
  }
}
```

---

#### `POST /whatsapp/vincular`

Inicia o fluxo de vinculação — gera OTP e envia via SMS.

**Request Body:**
```json
{ "numero_telefone": "+5585999994521" }
```

**Response `200`:**
```json
{
  "data": {
    "estado": "OTP_SMS_ENVIADO",
    "expires_at": "2026-03-23T14:15:00Z"
  }
}
```

**Erros:**

| Código | Error Code | Quando ocorre |
|---|---|---|
| 400 | `WA_NUMERO_INVALIDO` | Número não segue formato E.164 |
| 409 | `WA_JA_VINCULADO` | Cessionário já tem número vinculado |
| 429 | `WA_OTP_RATE_LIMIT` | > 3 tentativas OTP/hora |
| 429 | `WA_OTP_HARD_BLOCK` | 5 falhas consecutivas — bloqueado 30min |
| 502 | `WA_SMS_DELIVERY_FAILED` | Falha no envio via EvolutionAPI |

---

#### `POST /whatsapp/verificar-otp`

Verifica o OTP digitado pelo Cessionário.

**Request Body:**
```json
{ "codigo": "123456" }
```

**Response `200`:**
```json
{
  "data": {
    "estado": "AGUARDANDO_CONFIRMACAO_WA",
    "mensagem": "Código correto! Confirme a vinculação no WhatsApp."
  }
}
```

**Erros:**

| Código | Error Code | Quando ocorre |
|---|---|---|
| 400 | `WA_OTP_INVALIDO` | OTP incorreto |
| 400 | `WA_OTP_EXPIRADO` | OTP expirado (> 15 min) |
| 429 | `WA_OTP_HARD_BLOCK` | 5 falhas consecutivas |

---

#### `DELETE /whatsapp/vincular`

Desvincula o WhatsApp do Cessionário.

**Response `200`:**
```json
{
  "data": { "estado": "DESVINCULADO", "updated_at": "2026-03-23T15:00:00Z" }
}
```

---

#### `POST /whatsapp/webhook`

Recebe mensagens do WhatsApp via EvolutionAPI. **Não requer Bearer — usa API Key de webhook.**

**Headers:**
```
x-api-key: {EVOLUTIONAPI_WEBHOOK_SECRET}
Content-Type: application/json
```

**Request Body (EvolutionAPI payload):**
```json
{
  "event": "messages.upsert",
  "data": {
    "key": { "remoteJid": "5585999994521@s.whatsapp.net" },
    "message": { "conversation": "Quais são as melhores oportunidades?" },
    "messageTimestamp": 1711195200
  }
}
```

**Response `200`:**
```json
{ "data": { "received": true } }
```

**Erros:**

| Código | Error Code | Quando ocorre |
|---|---|---|
| 401 | `WA_WEBHOOK_INVALID_KEY` | API Key inválida ou ausente |
| 404 | `WA_CESSIONARIO_NOT_FOUND` | Número não vinculado a nenhum Cessionário |

---

### 7.5 Domínio: Oportunidades (Read-Only)

**Tabela resumo:**

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `GET` | `/oportunidades/{opr_id}` | Dados de uma oportunidade para contexto da Dani | Bearer |

---

#### `GET /oportunidades/{opr_id}`

Retorna dados de uma oportunidade para injeção de contexto na Dani.

**Response `200`:**
```json
{
  "data": {
    "opr_id": "OPR-2024-0042",
    "tabela_atual": 430000.00,
    "tabela_contrato": 350000.00,
    "valor_pago_cedente": 85000.00,
    "preco_repasse": 350000.00,
    "localizacao": {
      "cidade": "Fortaleza",
      "estado": "CE",
      "empreendimento": "Residencial Horizonte"
    },
    "tipologia": "2BR",
    "status_marketplace": "DISPONIVEL",
    "snapshot_at": "2026-03-23T14:00:00Z"
  }
}
```

**Erros:**

| Código | Error Code | Quando ocorre |
|---|---|---|
| 404 | `OPR_NOT_FOUND` | OPR inexistente |
| 403 | `OPR_ACCESS_DENIED` | Oportunidade restrita |

---

### 7.6 Domínio: Sessão / Contexto

**Tabela resumo:**

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/dani/sessao` | Registra sessão da Dani (início de uso) | Bearer |

---

#### `POST /dani/sessao`

Registra uma nova sessão da Dani vinculada ao JWT atual.

**Request Body:**
```json
{
  "conversa_id": "uuid",
  "canal": "WEBCHAT"
}
```

**Response `201`:**
```json
{
  "data": {
    "id": "uuid",
    "estado_agente": "OPERACIONAL",
    "expires_at": "2026-03-23T15:00:00Z"
  }
}
```

---

## 8. Webhooks

### 8.1 Eventos Disponíveis (Fase 2 — EvolutionAPI)

| Evento | Trigger | Payload |
|---|---|---|
| `messages.upsert` | Mensagem recebida no WhatsApp | Ver §7.4 `POST /whatsapp/webhook` |
| `connection.update` | Status da conexão EvolutionAPI | `{ state: "open" \| "close" \| "connecting" }` |

### 8.2 Verificação de Assinatura

```
Header: x-api-key: {EVOLUTIONAPI_WEBHOOK_SECRET}
```

Backend valida `x-api-key` antes de processar qualquer payload. Request sem chave ou com chave inválida: `401`.

### 8.3 Retry Policy

EvolutionAPI faz até 3 retries automáticos em caso de timeout ou `5xx`. O endpoint de webhook é idempotente — múltiplas entregas do mesmo evento não duplicam mensagens (deduplicação via `messageTimestamp` + `remoteJid`).

---

## Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. 18 endpoints em 6 domínios. Padrões de autenticação, erros, paginação e rate limiting. Webhook EvolutionAPI (Fase 2). |

---

## Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Endpoint bulk "marcar todos alertas como lido" | Decisão Autônoma | §7.3 Alertas | Não documentado por ausência de RF específico. Implementar se UX exigir (botão "Marcar todos como lido"). Sugestão: `PATCH /dani/alertas/marcar-todos-lido` | P2 | Product Owner | Aberto |
| Endpoint GET status do agente | Decisão Autônoma | §7.1 | Útil para polling do estado `OPERACIONAL/FALLBACK/DESLIGADO`. Sugestão: `GET /dani/status` retornando `{ estado_agente: string }` | P2 | Backend Lead | Aberto |
| Webhook assinatura HMAC | [DEFINIÇÃO PENDENTE] | §8.2 | Opção A: `x-api-key` simples (atual — mais fácil). Opção B: HMAC-SHA256 do payload (mais seguro, padrão GitHub/Stripe). Trade-off: HMAC requer shared secret e computação no recebimento. Decisão impacta segurança | P1 | Backend Lead + Security | Aberto |
