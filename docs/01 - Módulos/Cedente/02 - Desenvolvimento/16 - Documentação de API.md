# 16 - Documentação de API

## Módulo Cedente · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend, Frontend e QA |
| **Escopo** | API Reference — endpoints, autenticação, contratos, erros, paginação e exemplos |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Data da versão** | 23/03/2026 (America/Fortaleza) |
| **Status** | Ativo |
| **Referências** | 01.1 a 01.2 - Regras de Negócio · 02 - Stacks · 12 - Modelo de Dados · 14 - Especificações Técnicas |

---

> **TL;DR**
>
> - **44 endpoints documentados** em 10 domínios: Auth, Perfil do Cedente, Casos/Imóveis, Dossiê (upload e documentos), Propostas (recebimento e resposta), Escalonamento/Cenário, Assinaturas (ZapSign), Financeiro/Escrow, Notificações, IA (Guardião do Retorno).
> - **Base URL:** `https://api.repasseseguro.com.br/api/v1` — versionamento via prefixo de path.
> - **Autenticação:** JWT Bearer (access token 15-30min + refresh token httpOnly cookie). Header `Authorization: Bearer {token}` obrigatório em todos os endpoints protegidos.
> - **Paginação:** offset-based — `page`, `per_page` (max 100, default 20) + objeto `meta` em toda listagem.
> - **Padrão de erro:** `{ "error": { "code": "PREFIX-NNN", "message": "...", "details": {} } }` em todos os endpoints.
> - **Rate limiting:** 100 req/min geral; 10 req/15min em auth; 20 req/10min upload dossiê; 20 req/min Guardião IA.
> - **Isolamento absoluto:** RLS garante que um Cedente jamais acessa dados de outro. `cessionario_id` nunca exposto (RN-012, RN-085).

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
- Upload de documentos do dossiê: não via multipart direto — o backend gera uma signed URL e o cliente faz upload diretamente para o Supabase Storage (ADR-CED-004). Ver §7.3.
- Encoding: `UTF-8`.
- Datas: `ISO 8601` com timezone (`2026-03-23T00:00:00-03:00`).
- Valores monetários: inteiros em **centavos** (ex: `R$ 100.000,00` → `10000000`). [DECISÃO AUTÔNOMA — centavos eliminam erros de ponto flutuante em cálculos de comissão; alternativa descartada: float com 2 casas decimais (risco de imprecisão em multiplicações de 20% sobre Valor Recuperado − Valor Distrato Referência)]

### 1.3 Convenção de Versionamento

- Versão no path: `/api/v1/`, `/api/v2/`.
- Breaking changes incrementam a versão. Versões anteriores mantidas por 90 dias após deprecação.
- Versões deprecated: header `Deprecation: {date}` nas respostas.

### 1.4 Convenção de Nomenclatura

- Paths: `kebab-case`, plural para recursos (`/casos`, `/propostas`, `/assinaturas`).
- Sub-recursos: `/{id}/sub-recurso` (ex: `POST /propostas/{id}/aceitar`).
- Ações sem correspondência CRUD: verbos no path (`POST /{id}/aceitar`, `POST /{id}/recusar`, `POST /{id}/contrapropor`).

---

## 2. Autenticação

### 2.1 Fluxo JWT Completo

```
Cadastro → e-mail de ativação (link 48h) → Conta ativa
     ↓
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
| `X-CSRF-Token` | Token CSRF | Endpoints mutáveis (POST/PUT/PATCH/DELETE) |

### 2.3 Expiração e Refresh

- **Access token:** 15 minutos (produção) / 30 minutos (staging).
- **Refresh token:** 30 dias. Rotacionado a cada uso (refresh token rotation).
- **Inatividade:** sessão encerrada automaticamente após 24 horas de inatividade contínua (RN-006). Aviso exibido 5 minutos antes da expiração.

### 2.4 Tratamento de Token Expirado/Inválido

| Código HTTP | Error Code | Quando ocorre |
|---|---|---|
| 401 | `AUTH-001` | Token ausente, malformado ou expirado |
| 401 | `AUTH-002` | Refresh token inválido/expirado → redirecionar para login |
| 403 | `AUTH-003` | Token válido mas sem permissão para o recurso (RBAC) |
| 403 | `AUTH-008` | Conta pendente de ativação (e-mail não confirmado) |
| 403 | `AUTH-009` | Conta bloqueada temporariamente — 5 tentativas falhas, 15 min (RN-005) |

---

## 3. Padrão de Resposta

### 3.1 Schema de Sucesso

```json
{
  "data": { ... }
}
```

Para listagens:

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

### 3.2 Schema de Erro

```json
{
  "error": {
    "code": "CAS-010",
    "message": "Já existe um caso ativo para este imóvel.",
    "details": {
      "caso_id": "880e8400-e29b-41d4-a716-446655440001",
      "status": "DISPONIVEL_PARA_COMPRADORES"
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
| 202 | Accepted | POST assíncrono (upload dossiê — signed URL gerada) |
| 204 | No Content | DELETE com sucesso |
| 400 | Bad Request | Validação de DTO falhou |
| 401 | Unauthorized | Token ausente, inválido ou expirado |
| 403 | Forbidden | RBAC, conta bloqueada ou pendente de ativação |
| 404 | Not Found | Recurso não encontrado ou não pertence ao Cedente |
| 409 | Conflict | Estado incompatível (caso duplicado, aceite já registrado) |
| 422 | Unprocessable Entity | Regra de negócio violada (cooldown, limite de rodadas, prazo expirado) |
| 429 | Too Many Requests | Rate limit excedido |
| 500 | Internal Server Error | Erro não tratado — Sentry alertado |
| 503 | Service Unavailable | Dependência externa indisponível (ZapSign, Escrow, OpenAI) |

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
| Auth — login, refresh | 10 req | 15 min | `X-RateLimit-Limit: 10` |
| Auth — cadastro | 5 req | 1 h | `X-RateLimit-Limit: 5` |
| Auth — recuperação de senha | 3 req | 1 h | `X-RateLimit-Limit: 3` |
| Upload dossiê (signed URL) | 20 req | 10 min | `X-RateLimit-Limit: 20` |
| Propostas (ações) | 30 req | 5 min | `X-RateLimit-Limit: 30` |
| Guardião IA | 20 req | 1 min | `X-RateLimit-Limit: 20` |

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
    "code": "AUTH-010",
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
| POST | `/auth/register` | Cadastro com nome, CPF/CNPJ, e-mail, telefone e senha | Público |
| POST | `/auth/login` | Login com e-mail/senha | Público |
| GET | `/auth/activate` | Ativação da conta por link de e-mail | Token URL |
| POST | `/auth/resend-activation` | Reenvio do e-mail de ativação | Público |
| POST | `/auth/refresh` | Renovar access token | Cookie refresh |
| DELETE | `/auth/session` | Logout | Bearer |
| POST | `/auth/forgot-password` | Solicitar reset de senha | Público |
| POST | `/auth/reset-password` | Confirmar nova senha via token | Token URL |
| POST | `/auth/reauth` | Re-autenticação para ação crítica (LGPD, exclusão) | Bearer |

---

**POST /auth/register**

Cadastra novo Cedente. Valida CPF/CNPJ (algoritmo de dígito verificador + unicidade), e-mail único, senha mínima. Cria conta com status `PENDENTE_ATIVACAO` e envia e-mail de ativação via fila RabbitMQ → Resend. (RN-001)

```bash
curl -X POST https://api.repasseseguro.com.br/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Ana Martins",
    "cpf_cnpj": "123.456.789-09",
    "email": "ana@email.com",
    "phone": "+5585999999999",
    "password": "Senha@123"
  }'
```

**Response 201:**
```json
{
  "data": {
    "message": "Verifique seu e-mail para ativar sua conta.",
    "email": "ana@email.com"
  }
}
```

**Erros:**

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `CED-001` | "Este CPF/CNPJ já possui uma conta." | CPF/CNPJ duplicado |
| 400 | `CED-002` | "Este e-mail já está cadastrado." | E-mail duplicado |
| 400 | `CED-003` | "A senha deve ter pelo menos 8 caracteres, uma letra maiúscula e um número." | Senha fraca |
| 400 | `CED-004` | "CPF/CNPJ inválido." | Dígito verificador inválido |
| 429 | `AUTH-010` | "Limite de tentativas de cadastro atingido." | 5 req/h por IP |

---

**POST /auth/login**

Login com e-mail e senha. Retorna access token + set-cookie com refresh token httpOnly. (RN-005)

```bash
curl -X POST https://api.repasseseguro.com.br/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ana@email.com","password":"Senha@123"}'
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
      "email": "ana@email.com",
      "name": "Ana Martins",
      "cedente_id": "660e8400-e29b-41d4-a716-446655440001",
      "tipo": "PF",
      "status_conta": "ATIVA"
    }
  }
}
```
*Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict; Max-Age=2592000*

**Erros:**

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 401 | `AUTH-001` | "E-mail ou senha incorretos." | Credenciais inválidas |
| 403 | `AUTH-008` | "Conta pendente de ativação. Verifique seu e-mail." | E-mail não confirmado |
| 403 | `AUTH-009` | "Conta bloqueada temporariamente. Tente novamente às [horário]." | 5 tentativas falhas — bloqueio 15 min (RN-005) |
| 429 | `AUTH-010` | "Limite de tentativas atingido." | 10 req/15min por IP |

---

**GET /auth/activate**

Ativa conta do Cedente via token de URL enviado por e-mail. (RN-002)

**Query params:** `token` (string, obrigatório)

**Response 200:**
```json
{
  "data": {
    "message": "Conta ativada com sucesso!",
    "redirect_url": "/dashboard"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `AUTH-011` | "Link de ativação inválido ou expirado." | Token inválido ou após 48h (RN-002) |

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

Valida senha do usuário para re-autenticação antes de ação crítica (ex: solicitação de exclusão LGPD).

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
| 422 | `AUTH-012` | "Máximo de tentativas de re-autenticação atingido." | 3 falhas → logout |

---

### 7.2 Perfil do Cedente (`/cedentes`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/cedentes/me` | Perfil do Cedente logado | Bearer |
| PATCH | `/cedentes/me` | Atualizar dados do perfil (exceto CPF/CNPJ) | Bearer |
| GET | `/cedentes/me/dados-pessoais` | Exportar dados pessoais (LGPD — direito de acesso) | Bearer |
| GET | `/cedentes/me/exportar` | Exportar portabilidade de dados (JSON/CSV) | Bearer |
| POST | `/cedentes/me/solicitacao-exclusao` | Solicitar exclusão de dados (LGPD) | Bearer + reauth |
| PATCH | `/cedentes/me/ai-consent` | Atualizar consentimento de uso de IA | Bearer |

---

**GET /cedentes/me**

```bash
curl https://api.repasseseguro.com.br/api/v1/cedentes/me \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Ana Martins",
    "email": "ana@email.com",
    "phone": "+5585999999999",
    "cpf_cnpj": "***.***.789-09",
    "tipo": "PF",
    "status_conta": "ATIVA",
    "ai_consent": true,
    "created_at": "2026-01-15T10:00:00-03:00"
  }
}
```

> **Nota de segurança:** CPF/CNPJ retornado mascarado. Campo imutável pelo Cedente (RN-007).

---

**PATCH /cedentes/me**

Atualiza nome, telefone ou preferências. CPF/CNPJ imutável — retorna 422 se enviado. (RN-007)

**Headers:** `Authorization`, `Content-Type`, `X-CSRF-Token`

**Body:**
```json
{
  "name": "Ana Cristina Martins",
  "phone": "+5585988888888"
}
```

**Response 200:**
```json
{
  "data": {
    "id": "660e8400-e29b-41d4-a716-446655440001",
    "name": "Ana Cristina Martins",
    "phone": "+5585988888888",
    "updated_at": "2026-03-23T10:00:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `CED-005` | "CPF/CNPJ não pode ser alterado." | Campo `cpf_cnpj` presente no body (RN-007) |

---

**POST /cedentes/me/solicitacao-exclusao**

Solicita exclusão de dados (LGPD). Exige re-autenticação. Processado pelo Admin em até 15 dias. (RN-010)

**Headers:** `Authorization`, `Content-Type`, `X-CSRF-Token`, `X-Reauth-Token: {reauth_token}`

**Body:**
```json
{ "reason": "Não desejo mais utilizar a plataforma." }
```

**Response 202:**
```json
{
  "data": {
    "solicitacao_id": "770e8400-...",
    "status": "PENDENTE",
    "prazo_dias": 15,
    "message": "Sua solicitação foi recebida e será processada em até 15 dias corridos."
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 403 | `AUTH-013` | "Re-autenticação necessária." | `X-Reauth-Token` ausente ou expirado |
| 409 | `CED-006` | "Solicitação de exclusão já em andamento." | Solicitação anterior ainda pendente |

---

### 7.3 Casos / Imóveis (`/casos`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/casos` | Listar casos do Cedente | Bearer |
| POST | `/casos/draft` | Iniciar rascunho de cadastro (Etapa 1 do wizard) | Bearer |
| GET | `/casos/{id}` | Detalhe de um caso | Bearer |
| PATCH | `/casos/{id}/draft` | Atualizar rascunho (salvar etapa parcial) | Bearer |
| PATCH | `/casos/{id}/simulador-visualizado` | Confirmar visualização do simulador (Etapa 3) | Bearer |
| PATCH | `/casos/{id}/cenario` | Escolher/confirmar cenário de retorno (Etapa 4) | Bearer |
| POST | `/casos/{id}/confirmar` | Confirmar cadastro e concluir wizard (Etapa 5) | Bearer |
| POST | `/casos/{id}/cancelar` | Cancelar caso ativo | Bearer |
| GET | `/casos/{id}/timeline` | Linha do tempo de eventos do caso | Bearer |

---

**GET /casos**

**Query params:** `page`, `per_page`, `sort`, `order`, `status`

```bash
curl "https://api.repasseseguro.com.br/api/v1/casos?page=1&per_page=20" \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440002",
      "code": "CAS-2026-0015",
      "status": "DISPONIVEL_PARA_COMPRADORES",
      "status_label": "Disponível para compradores",
      "cenario": "C",
      "imovel": {
        "endereco": "Rua das Palmeiras, 123 — Apt 45",
        "cidade": "Fortaleza",
        "estado": "CE",
        "tipo": "Apartamento"
      },
      "created_at": "2026-02-01T08:00:00-03:00",
      "updated_at": "2026-03-10T14:00:00-03:00"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 3, "total_pages": 1 }
}
```

---

**POST /casos/draft**

Inicia rascunho de cadastro com dados básicos do imóvel (Etapa 1 do wizard). Verifica duplicidade de imóvel ativo (RN-090). (RN-023)

**Headers:** `Authorization`, `Content-Type`, `X-CSRF-Token`

**Body:**
```json
{
  "imovel": {
    "endereco": "Rua das Palmeiras, 123",
    "numero": "Apt 45",
    "bairro": "Meireles",
    "cidade": "Fortaleza",
    "estado": "CE",
    "cep": "60165-121",
    "tipo": "Apartamento",
    "construtora": "Construtora ABC",
    "numero_contrato": "CONTR-2022-00456"
  }
}
```

**Response 201:**
```json
{
  "data": {
    "caso_id": "880e8400-e29b-41d4-a716-446655440002",
    "rascunho": true,
    "expira_em": "2026-04-22T10:00:00-03:00",
    "etapa_atual": 1
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `CAS-010` | "Já existe um caso ativo para este imóvel." | Imóvel com caso em andamento (RN-090) |
| 409 | `CAS-011` | "Você já tem um rascunho em andamento." | Rascunho ativo existente (RN-023) |

---

**PATCH /casos/{id}/cenario**

Confirma a escolha do cenário de retorno (Etapa 4 do wizard). Requer que o simulador tenha sido visualizado por ao menos 10 segundos (RN-021, RN-022).

**Body:**
```json
{
  "cenario": "C",
  "confirmado": true
}
```

**Response 200:**
```json
{
  "data": {
    "caso_id": "880e8400-...",
    "cenario": "C",
    "cenario_ativo_desde": "2026-03-23T10:05:00-03:00",
    "simulacao": {
      "valor_pago_cedente_cents": 15000000,
      "valor_distrato_referencia_cents": 7500000,
      "retorno_esperado_cents": 19500000,
      "comissao_estimada_cents": 2400000,
      "valor_liquido_cedente_cents": 17100000
    }
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `CAS-012` | "Simulador não foi visualizado. Aguarde 10 segundos antes de escolher." | `simulador_visualizado = false` (RN-021) |
| 422 | `CAS-013` | "Cenário inválido. Opções: A, B, C, D." | Valor diferente de A/B/C/D |
| 422 | `CAS-014` | "Confirmação obrigatória ao escolher cenário." | `confirmado: false` (RN-022) |

---

**POST /casos/{id}/confirmar**

Conclui o wizard, cria o caso definitivamente com status `CADASTRO_REALIZADO`, gera Termo de Cadastro via ZapSign e inicia checklist do dossiê. (RN-024)

**Response 200:**
```json
{
  "data": {
    "caso_id": "880e8400-...",
    "code": "CAS-2026-0015",
    "status": "CADASTRO_REALIZADO",
    "rascunho": false,
    "envelope_assinatura": {
      "id": "990e8400-...",
      "tipo": "TERMO_CADASTRO",
      "status": "PENDENTE",
      "zapsign_doc_url": "https://app.zapsign.com.br/verificar/TOKEN_TERMO"
    },
    "dossie": {
      "id": "aaa0e8400-...",
      "total_documentos": 6,
      "documentos_pendentes": 6
    }
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `CAS-015` | "Cenário não selecionado. Conclua a Etapa 4 antes de confirmar." | Sem cenário definido |
| 503 | `ZAP-001` | "Serviço de assinatura temporariamente indisponível." | ZapSign offline |

---

**POST /casos/{id}/cancelar**

Cancela caso ativo. Permitido em qualquer status anterior ao Fechamento. (RN-049)

**Headers:** `Authorization`, `Content-Type`, `X-CSRF-Token`

**Body:**
```json
{ "motivo": "Decidi não realizar o repasse neste momento." }
```

**Response 200:**
```json
{
  "data": {
    "caso_id": "880e8400-...",
    "status": "CANCELADO",
    "cancelado_em": "2026-03-23T11:00:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `CAS-016` | "Caso não pode ser cancelado no status atual." | Status `NEGOCIO_FECHADO` ou posterior |

---

### 7.4 Dossiê — Upload e Documentos (`/dossies`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/dossies/{id}` | Checklist do dossiê com status de cada documento | Bearer |
| POST | `/dossies/{id}/documentos/{tipo}/upload-url` | Gerar signed URL para upload direto ao Supabase Storage | Bearer |
| POST | `/dossies/{id}/documentos/{tipo}/confirmar` | Confirmar upload concluído e registrar no banco | Bearer |
| GET | `/dossies/{id}/documentos` | Listar documentos do dossiê | Bearer |
| DELETE | `/dossies/{id}/documentos/{tipo}` | Remover documento rejeitado para reenvio | Bearer |

---

**GET /dossies/{id}**

```bash
curl https://api.repasseseguro.com.br/api/v1/dossies/aaa0e8400-... \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": {
    "id": "aaa0e8400-...",
    "caso_id": "880e8400-...",
    "total_documentos": 6,
    "documentos_pendentes": 2,
    "documentos_em_analise": 3,
    "documentos_verificados": 1,
    "documentos_rejeitados": 0,
    "completo": false,
    "documentos": [
      {
        "tipo": "CONTRATO_ORIGINAL",
        "label": "Contrato original com a construtora",
        "status": "VERIFICADO",
        "obrigatorio": true,
        "enviado_em": "2026-03-10T09:00:00-03:00"
      },
      {
        "tipo": "COMPROVANTE_PAGAMENTOS",
        "label": "Comprovante de pagamentos (últimos 12 meses)",
        "status": "PENDENTE",
        "obrigatorio": true,
        "enviado_em": null
      }
    ]
  }
}
```

---

**POST /dossies/{id}/documentos/{tipo}/upload-url**

Valida MIME type real (magic bytes — não extensão declarada, RN-042) e tamanho (≤ 10 MB). Retorna signed URL para upload direto ao Supabase Storage. (ADR-CED-004)

**Headers:** `Authorization`, `Content-Type`, `X-CSRF-Token`

**Body:**
```json
{
  "file_name": "contrato_original.pdf",
  "mime_type": "application/pdf",
  "file_size_bytes": 2097152
}
```

**Response 202:**
```json
{
  "data": {
    "documento_id": "bbb0e8400-...",
    "upload_url": "https://storage.supabase.co/object/sign/dossie/...",
    "upload_url_expires_at": "2026-03-23T10:15:00-03:00",
    "storage_path": "dossies/aaa0e8400.../CONTRATO_ORIGINAL/bbb0e8400-....pdf"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DOS-001` | "Formato não aceito. Use PDF, JPG ou PNG." | MIME inválido (RN-042) |
| 400 | `DOS-002` | "Arquivo muito grande. Máximo 10 MB." | `file_size_bytes > 10485760` (RN-042) |
| 400 | `DOS-003` | "Tipo de documento inválido." | `{tipo}` não reconhecido |
| 429 | `DOS-010` | "Limite de uploads atingido. Aguarde antes de tentar novamente." | 20 req/10min |

---

**POST /dossies/{id}/documentos/{tipo}/confirmar**

Confirma que o upload foi concluído com sucesso no Supabase Storage. Atualiza status do documento para `EM_ANALISE`. Se o dossiê estiver completo e o Termo de Cadastro assinado, avança o caso para `EM_ANALISE`. (RN-043)

**Body:**
```json
{
  "storage_path": "dossies/aaa0e8400.../CONTRATO_ORIGINAL/bbb0e8400-....pdf"
}
```

**Response 200:**
```json
{
  "data": {
    "documento_id": "bbb0e8400-...",
    "tipo": "CONTRATO_ORIGINAL",
    "status": "EM_ANALISE",
    "dossie_completo": false,
    "caso_status": "CADASTRO_REALIZADO"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `DOS-004` | "storage_path inválido ou não encontrado." | Path não existe no Storage |
| 409 | `DOS-005` | "Documento já verificado. Não é possível substituir." | Status `VERIFICADO` |

---

### 7.5 Propostas (`/propostas`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/propostas` | Listar propostas do Cedente | Bearer |
| GET | `/propostas/{id}` | Detalhe de uma proposta com simulação financeira | Bearer |
| POST | `/propostas/{id}/aceitar` | Aceitar proposta (confirmação dupla — RN-032) | Bearer |
| POST | `/propostas/{id}/recusar` | Recusar proposta | Bearer |
| POST | `/propostas/{id}/contrapropor` | Enviar contraproposta | Bearer |

---

**GET /propostas**

**Query params:** `page`, `per_page`, `caso_id`, `status`

```bash
curl "https://api.repasseseguro.com.br/api/v1/propostas?caso_id=880e8400-...&status=RECEBIDA" \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "ccc0e8400-...",
      "caso_id": "880e8400-...",
      "caso_code": "CAS-2026-0015",
      "status": "RECEBIDA",
      "rodada": 1,
      "valor_proposta_cents": 16000000,
      "simulacao": {
        "valor_liquido_cedente_cents": 14800000,
        "comissao_rs_cents": 1200000,
        "valor_distrato_referencia_cents": 7500000
      },
      "expires_at": "2026-03-28T08:00:00-03:00",
      "criada_em": "2026-03-23T08:00:00-03:00"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 1, "total_pages": 1 }
}
```

> **Nota de privacidade:** nenhum campo identificador do Cessionário retornado (RN-012, RN-085). `cessionario_id` nunca presente na resposta.

---

**POST /propostas/{id}/aceitar**

Aceita a proposta após confirmação dupla do Cedente. Encerra automaticamente outras propostas ativas do mesmo caso como `SUPERADA` (RN-033). Caso avança para `EM_FORMALIZACAO`. (RN-032)

**Headers:** `Authorization`, `Content-Type`, `X-CSRF-Token`

**Body:**
```json
{
  "confirmado": true
}
```

**Response 200:**
```json
{
  "data": {
    "proposta_id": "ccc0e8400-...",
    "status": "ACEITA",
    "caso_status": "EM_FORMALIZACAO",
    "valor_aceito_cents": 16000000,
    "valor_liquido_cedente_cents": 14800000,
    "comissao_rs_cents": 1200000,
    "aceito_em": "2026-03-23T10:00:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 400 | `PRP-001` | "Confirmação obrigatória para aceitar proposta." | `confirmado: false` (RN-032) |
| 409 | `PRP-002` | "Proposta não está disponível para aceite." | Status != `RECEBIDA` |
| 422 | `PRP-003` | "Prazo de resposta expirado." | Após 5 dias úteis (RN-031) |
| 404 | `PRP-004` | "Proposta não encontrada." | ID inexistente ou de outro Cedente |

---

**POST /propostas/{id}/recusar**

Recusa a proposta. Caso retorna para `DISPONIVEL_PARA_COMPRADORES`. (RN-034)

**Body:**
```json
{ "motivo": "Valor abaixo do esperado para o cenário." }
```

**Response 200:**
```json
{
  "data": {
    "proposta_id": "ccc0e8400-...",
    "status": "RECUSADA",
    "caso_status": "DISPONIVEL_PARA_COMPRADORES",
    "recusado_em": "2026-03-23T10:05:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `PRP-002` | "Proposta não está disponível para recusa." | Status != `RECEBIDA` |
| 422 | `PRP-003` | "Prazo de resposta expirado." | Após 5 dias úteis (RN-031) |

---

**POST /propostas/{id}/contrapropor**

Envia contraproposta. Valor deve ser ≥ piso do cenário atual (RN-035). (RN-036)

**Body:**
```json
{
  "valor_cents": 17500000,
  "mensagem": "Considerando a valorização da região, proponho este valor."
}
```

**Response 200:**
```json
{
  "data": {
    "proposta_id": "ccc0e8400-...",
    "status": "CONTRAPROPOSTA_ENVIADA",
    "rodada": 2,
    "valor_contraproposta_cents": 17500000,
    "enviado_em": "2026-03-23T10:10:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `PRP-005` | "Valor abaixo do piso do cenário atual." | Valor < mínimo permitido (RN-035) |
| 422 | `PRP-006` | "Máximo de rodadas de negociação atingido." | Limite de rodadas atingido |
| 409 | `PRP-002` | "Proposta não está em estado de negociação." | Status incorreto |

---

### 7.6 Escalonamento de Cenário (`/casos/{id}/escalonamento`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/casos/{id}/escalonamento/opcoes` | Verificar disponibilidade e opções de escalonamento | Bearer |
| POST | `/casos/{id}/escalonamento` | Solicitar escalonamento de cenário | Bearer |
| DELETE | `/casos/{id}/escalonamento/fila` | Cancelar escalonamento enfileirado | Bearer |

---

**GET /casos/{id}/escalonamento/opcoes**

Verifica cooldown (RN-027), proposta ativa em negociação (RN-025) e retorna próximo cenário disponível. (RN-026)

```bash
curl https://api.repasseseguro.com.br/api/v1/casos/880e8400-.../escalonamento/opcoes \
  -H "Authorization: Bearer {token}"
```

**Response 200 — Escalonamento disponível:**
```json
{
  "data": {
    "cenario_atual": "C",
    "cenario_disponivel": "B",
    "disponivel": true,
    "escalonamento_enfileirado": false,
    "simulacao": {
      "cenario_b": {
        "retorno_esperado_cents": 15000000,
        "comissao_estimada_cents": 1500000,
        "valor_liquido_cents": 13500000
      }
    }
  }
}
```

**Response 200 — Cooldown ativo:**
```json
{
  "data": {
    "cenario_atual": "C",
    "disponivel": false,
    "cooldown_ate": "2026-03-30T10:00:00-03:00",
    "message": "Próximo ajuste disponível a partir de 30/03/2026."
  }
}
```

**Response 200 — Proposta ativa (enfileiramento):**
```json
{
  "data": {
    "cenario_atual": "C",
    "disponivel": false,
    "escalonamento_enfileirado": true,
    "message": "Você tem uma proposta em análise. O pedido será processado após a conclusão da negociação."
  }
}
```

---

**POST /casos/{id}/escalonamento**

Solicita escalonamento. Gera Termo de Aceite de Escalonamento via ZapSign. Cenário só é atualizado após assinatura do Termo. (RN-025, RN-026, RN-027)

**Body:**
```json
{ "cenario_novo": "B" }
```

**Response 200:**
```json
{
  "data": {
    "envelope_assinatura": {
      "id": "ddd0e8400-...",
      "tipo": "TERMO_ACEITE_ESCALONAMENTO",
      "status": "PENDENTE",
      "zapsign_doc_url": "https://app.zapsign.com.br/verificar/TOKEN_ESC"
    },
    "message": "Assine o Termo de Aceite para confirmar o escalonamento para o Cenário B."
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `CAS-027` | "Cooldown ativo. Próximo ajuste disponível em [data]." | Menos de 7 dias desde último escalonamento (RN-027) |
| 422 | `CAS-028` | "Cenário A é o mínimo. Não é possível escalonar mais." | Tentativa de escalonar além do Cenário A |
| 422 | `CAS-029` | "Subida de cenário não é permitida." | `cenario_novo` superior ao atual (RN-029) |
| 422 | `CAS-030` | "Escalonamento deve ser para o próximo cenário imediatamente inferior." | Pulo de cenário (ex: C → A) (RN-026) |
| 503 | `ZAP-001` | "Serviço de assinatura temporariamente indisponível." | ZapSign offline |

---

### 7.7 Assinaturas ZapSign (`/assinaturas`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/assinaturas` | Listar envelopes de assinatura do Cedente | Bearer |
| GET | `/assinaturas/{id}` | Detalhe de envelope de assinatura | Bearer |
| GET | `/assinaturas/{id}/url-assinatura` | Obter URL de assinatura (ZapSign inline) | Bearer |

---

**GET /assinaturas**

**Query params:** `caso_id`, `status` (`PENDENTE|ASSINADO_CEDENTE|CONCLUIDO`), `page`, `per_page`

```bash
curl "https://api.repasseseguro.com.br/api/v1/assinaturas?caso_id=880e8400-...&status=PENDENTE" \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "990e8400-...",
      "caso_id": "880e8400-...",
      "caso_code": "CAS-2026-0015",
      "tipo": "TERMO_CADASTRO",
      "tipo_label": "Termo de Cadastro",
      "status": "PENDENTE",
      "expires_at": "2026-03-30T10:00:00-03:00",
      "criado_em": "2026-03-23T10:00:00-03:00"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 1, "total_pages": 1 }
}
```

---

**GET /assinaturas/{id}/url-assinatura**

Obtém URL autenticada do ZapSign para assinatura inline (iframe no painel). URL válida por sessão. (RN-047, RN-080)

```bash
curl https://api.repasseseguro.com.br/api/v1/assinaturas/990e8400-.../url-assinatura \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": {
    "signing_url": "https://app.zapsign.com.br/verificar/TOKEN123?auth=cedente_session",
    "expires_at": "2026-03-23T11:00:00-03:00",
    "envelope_status": "PENDENTE"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `ZAP-002` | "Envelope já assinado pelo Cedente." | Status != `PENDENTE` |
| 422 | `ZAP-003` | "Prazo de assinatura expirado." | Após `expires_at` (RN-050) |
| 503 | `ZAP-001` | "Serviço de assinatura temporariamente indisponível." | ZapSign offline |

---

### 7.8 Financeiro / Escrow (`/financeiro`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/financeiro/resumo` | KPIs financeiros do Cedente | Bearer |
| GET | `/financeiro/escrow/{caso_id}` | Status da Conta Escrow de um caso | Bearer |
| GET | `/financeiro/transacoes` | Histórico de transações | Bearer |
| GET | `/financeiro/transacoes/{id}` | Detalhe de transação | Bearer |
| POST | `/financeiro/reversao/{caso_id}` | Solicitar reversão pós-Fechamento | Bearer |

---

**GET /financeiro/resumo**

```bash
curl https://api.repasseseguro.com.br/api/v1/financeiro/resumo \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": {
    "casos_concluidos": 1,
    "valor_total_recebido_cents": 14800000,
    "valor_em_escrow_cents": 0,
    "comissoes_pagas_rs_cents": 1200000,
    "casos_aguardando_liberacao": 0
  }
}
```

---

**GET /financeiro/escrow/{caso_id}**

Retorna status atual da Conta Escrow para o caso. Somente leitura para o Cedente (RN-083). Último status cacheado exibido com timestamp se parceiro indisponível.

```bash
curl https://api.repasseseguro.com.br/api/v1/financeiro/escrow/880e8400-... \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": {
    "caso_id": "880e8400-...",
    "caso_code": "CAS-2026-0015",
    "status": "AGUARDANDO_DEPOSITO",
    "valor_total_cents": 16000000,
    "valor_liquido_cedente_cents": 14800000,
    "comissao_rs_cents": 1200000,
    "periodo_reversao_termina_em": null,
    "atualizado_em": "2026-03-23T11:00:00-03:00",
    "dados_cached": false
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 404 | `FIN-001` | "Conta Escrow não encontrada para este caso." | Caso ainda não chegou à fase de Fechamento |

---

**POST /financeiro/reversao/{caso_id}**

Solicita reversão do negócio dentro dos 15 dias corridos após o Fechamento. (RN-040, RN-083)

**Body:**
```json
{ "motivo": "Mudança nas condições pessoais — não consigo prosseguir com o repasse." }
```

**Response 200:**
```json
{
  "data": {
    "caso_id": "880e8400-...",
    "status": "DESISTENCIA_EM_ANALISE",
    "solicitado_em": "2026-03-25T10:00:00-03:00",
    "prazo_reversao_expira_em": "2026-04-07T10:00:00-03:00",
    "message": "Sua solicitação de desistência está sendo analisada."
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `FIN-002` | "Prazo de 15 dias para reversão expirado." | Após 15 dias corridos do Fechamento (RN-040) |
| 409 | `FIN-003` | "Reversão já em andamento para este caso." | Solicitação duplicada |
| 409 | `FIN-004` | "Caso não está em período de reversão." | Status != `AGUARDANDO_LIBERACAO` |

---

### 7.9 Notificações (`/notificacoes`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/notificacoes` | Listar notificações do Cedente | Bearer |
| PATCH | `/notificacoes/{id}/lida` | Marcar notificação como lida | Bearer |
| PATCH | `/notificacoes/todas-lidas` | Marcar todas como lidas | Bearer |
| GET | `/notificacoes/preferencias` | Preferências de notificação | Bearer |
| PATCH | `/notificacoes/preferencias` | Atualizar preferências de notificação | Bearer |

---

**GET /notificacoes**

**Query params:** `page`, `per_page`, `lida` (boolean), `caso_id`

```bash
curl "https://api.repasseseguro.com.br/api/v1/notificacoes?lida=false" \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "eee0e8400-...",
      "tipo": "NOT-CED-06",
      "titulo": "Nova proposta recebida!",
      "corpo": "Você recebeu uma proposta de R$ 160.000,00 para o imóvel CAS-2026-0015. Avalie no painel.",
      "lida": false,
      "deep_link": "/propostas/ccc0e8400-...",
      "caso_id": "880e8400-...",
      "criada_em": "2026-03-23T08:00:00-03:00"
    }
  ],
  "meta": { "page": 1, "per_page": 20, "total": 3, "total_pages": 1 }
}
```

---

**PATCH /notificacoes/preferencias**

Atualiza preferências de canal. Notificações por e-mail não podem ser desabilitadas (RN-069).

**Body:**
```json
{
  "email": true,
  "push": true,
  "in_app": true
}
```

**Response 200:**
```json
{
  "data": {
    "email": true,
    "push": true,
    "in_app": true,
    "atualizado_em": "2026-03-23T10:00:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 422 | `NOT-001` | "Notificações por e-mail não podem ser desabilitadas." | `email: false` (RN-069) |

---

### 7.10 IA — Guardião do Retorno (`/guardiao`)

| Método | Path | Descrição | Auth |
|---|---|---|---|
| GET | `/guardiao/session` | Obter ou criar sessão de chat | Bearer |
| POST | `/guardiao/message` | Enviar mensagem e receber resposta via SSE streaming | Bearer |
| GET | `/guardiao/sessions` | Listar sessões de chat do Cedente | Bearer |
| GET | `/guardiao/sessions/{id}/messages` | Histórico de mensagens de uma sessão | Bearer |
| POST | `/guardiao/escalacao` | Solicitar escalação para atendimento humano | Bearer |

---

**GET /guardiao/session**

Obtém a sessão ativa ou cria nova sessão de chat para o caso. Carrega contexto do caso (status, cenário, propostas, valores). (RN-058)

**Query params:** `caso_id` (UUID, opcional)

```bash
curl "https://api.repasseseguro.com.br/api/v1/guardiao/session?caso_id=880e8400-..." \
  -H "Authorization: Bearer {token}"
```

**Response 200:**
```json
{
  "data": {
    "session_id": "fff0e8400-...",
    "caso_id": "880e8400-...",
    "messages_history": [
      {
        "role": "assistant",
        "content": "Olá, Ana! Sou o Guardião do Retorno. Como posso ajudar com seu imóvel na Rua das Palmeiras?",
        "created_at": "2026-03-23T09:00:00-03:00"
      }
    ],
    "context": {
      "caso_status": "DISPONIVEL_PARA_COMPRADORES",
      "cenario": "C"
    }
  }
}
```

---

**POST /guardiao/message** — SSE Streaming

Envia mensagem ao Guardião do Retorno. Retorna resposta via SSE streaming (ADR-CED-002). Se confiança < threshold, sinaliza escalação para humano (RN-061). (RN-058 a RN-063)

**Headers:** `Authorization`, `Content-Type`, `Accept: text/event-stream`, `X-CSRF-Token`

**Body:**
```json
{
  "session_id": "fff0e8400-...",
  "message": "Qual é o melhor momento para escalonar para o Cenário B?"
}
```

**Response — SSE stream:**
```
data: {"token": "Com"}

data: {"token": " base"}

data: {"token": " no seu cenário C atual..."}

event: done
data: {"full_response": "Com base no seu cenário C atual, o melhor momento para escalonar seria...", "tokens_used": 287, "session_id": "fff0e8400-..."}
```

**Sinalização de escalação (RN-061):**
```
event: escalation
data: {"type": "escalation", "message": "Esta dúvida pode precisar de um especialista. Deseja que eu conecte você com nossa equipe?"}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 429 | `AI-010` | "Limite de consultas ao Guardião atingido." | 20 req/min por usuário |
| 403 | `AI-011` | "Consentimento de uso de IA não concedido." | `ai_consent = false` (RN-059) |
| 503 | `AI-012` | "Guardião temporariamente indisponível." | OpenAI timeout ou erro (RN-062) |

---

**POST /guardiao/escalacao**

Solicita escalação da conversa para atendimento humano (Admin). Cria notificação para o Admin com o histórico da sessão. (RN-061)

**Body:**
```json
{ "session_id": "fff0e8400-..." }
```

**Response 200:**
```json
{
  "data": {
    "escalacao_id": "ggg0e8400-...",
    "session_id": "fff0e8400-...",
    "status": "ESCALACAO_SOLICITADA",
    "message": "Um especialista entrará em contato em breve.",
    "criado_em": "2026-03-23T09:15:00-03:00"
  }
}
```

| Código | Error Code | Mensagem | Quando ocorre |
|---|---|---|---|
| 409 | `AI-013` | "Escalação já solicitada para esta sessão." | Escalação duplicada |

---

## 8. Webhooks

### 8.1 Webhook ZapSign (`POST /webhooks/zapsign`)

Recebido do ZapSign ao assinar ou concluir envelope de assinatura.

**Verificação de assinatura:** header `X-Zapsign-Signature: HMAC-SHA256({secret}, body)` — verificado antes de processar (RN-080).

**Payload (evento `signer_signed` — Cedente assinou):**
```json
{
  "event": "signer_signed",
  "doc_token": "TOKEN123",
  "signer": {
    "email": "ana@email.com",
    "role": "cedente",
    "signed_at": "2026-03-23T10:30:00Z"
  }
}
```

**Payload (evento `doc_complete` — todas as partes assinaram):**
```json
{
  "event": "doc_complete",
  "doc_token": "TOKEN123",
  "completed_at": "2026-03-23T11:00:00Z",
  "pdf_url": "https://api.zapsign.com.br/docs/TOKEN123/pdf"
}
```

**Resposta esperada:** `200 OK` — qualquer outro código causa retry pelo ZapSign.

**Retry policy ZapSign:** 3 tentativas em 30 min (5min, 15min, 30min).

**Efeitos após `doc_complete`:**
- `TERMO_CADASTRO`: caso avança para `EM_ANALISE` (se dossiê completo — RN-043).
- `TERMO_ACEITE_ESCALONAMENTO`: cenário é atualizado e caso retorna para `DISPONIVEL_PARA_COMPRADORES`.
- `TERMO_COMERCIAL`: caso pronto para Fechamento.
- `INSTRUMENTO_CESSAO`: Fechamento confirmado, Conta Escrow aberta.

---

### 8.2 Webhook Escrow (`POST /webhooks/escrow`)

Recebido do parceiro Escrow ao confirmar eventos de depósito ou distribuição.

**Verificação de assinatura:** header `X-Escrow-Signature: HMAC-SHA256({secret}, body)` (RN-083).

**Payload (evento `deposit_confirmed`):**
```json
{
  "event": "deposit_confirmed",
  "reference_id": "caso_id",
  "amount_cents": 16000000,
  "confirmed_at": "2026-03-23T14:30:00Z"
}
```

**Payload (evento `distribution_completed`):**
```json
{
  "event": "distribution_completed",
  "reference_id": "caso_id",
  "cedente_amount_cents": 14800000,
  "rs_commission_cents": 1200000,
  "distributed_at": "2026-04-07T10:00:00Z"
}
```

**Payload (evento `reversal_processed`):**
```json
{
  "event": "reversal_processed",
  "reference_id": "caso_id",
  "refunded_amount_cents": 16000000,
  "processed_at": "2026-04-05T10:00:00Z"
}
```

**Resposta esperada:** `200 OK`.

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Criação inicial — Pipeline ShiftLabs v9.5. 44 endpoints em 10 domínios. Autenticação JWT completa com ativação por e-mail. Wizard de cadastro com salvamento de rascunho. Upload de dossiê via signed URL. Propostas com confirmação dupla e anonimato do Cessionário. Escalonamento descendente com cooldown. Assinaturas ZapSign inline. Financeiro/Escrow somente leitura. Guardião do Retorno com SSE streaming. Webhooks ZapSign e Escrow. |

---

## 10. Backlog de Pendências

| Item | Tipo | Seção | Impacto | Justificativa / Decisão | Dono | Status |
|---|---|---|---|---|---|---|
| Valores monetários em centavos (integer) | Decisão Autônoma | §1.2 | P0 | Elimina erros de ponto flutuante no cálculo da fórmula de comissão (20% × (Valor Recuperado − Valor Distrato Referência)); frontend formata para exibição; alternativa descartada: float 2 casas (risco de imprecisão em multiplicações encadeadas) | Backend Lead | Decidido |
| Upload via Signed URL (não multipart direto) | Decisão Autônoma | §7.3 | P1 | Upload direto ao Supabase Storage elimina tráfego de arquivo pelo NestJS — confirmado em ADR-CED-004; API gera signed URL e cliente sobe diretamente; alternativa descartada: multipart no NestJS (bottleneck, latência, custo de egress) | Backend Lead | Decidido |
| SSE para streaming do Guardião (não WebSocket) | Decisão Autônoma | §7.10 | P1 | Confirmado em ADR-CED-002 — SSE é unidirecional server→client, adequado para streaming de tokens LLM; WebSocket descartado por complexidade desnecessária para este caso de uso | Backend Lead | Decidido |
| Paginação offset-based (não cursor) | Decisão Autônoma | §5 | P1 | Listagens do Cedente permitem ordenação por múltiplos campos — offset permite sort por `status`, `created_at`, `valor`; cursor-based requer campo único; alternativa descartada: cursor (limitaria ordenação) | Backend Lead | Decidido |
| Parceiro Escrow (DP-001) | Dado Pendente | §7.8, §8.2 | P0 | Parceiro bancário/fintech regulado para Conta Escrow ainda não definido. Endpoints documentados com contrato genérico. Quando parceiro for escolhido: atualizar paths de webhook, headers de assinatura e payload específico | Produto | Pendente |
| Endpoint de validação CNPJ em tempo real (Receita Federal) | Dado Pendente | §7.2 | P2 | RN-082 define validação de CNPJ em tempo real via API Receita Federal — endpoint de consulta do lado do frontend (on blur) não mapeado neste documento (chamada direta da Receita Federal no frontend). Avaliar se backend deve proxy ou frontend chama diretamente | Backend Lead | Pendente |
