# 20 - Error Handling — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Nome do Documento | Error Handling |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Status | Aprovado |
| Dependências | D01 · D08 · D14 · D16 |

---

> **📌 TL;DR**
>
> - **Princípios:** erro é contrato — backend, frontend e logs falam a mesma língua. Stack trace nunca exposta ao usuário.
> - **Camadas cobertas:** NestJS backend (GlobalExceptionFilter), SSE streaming (evento `error`), frontend (ErrorBoundary, toasts), logs (Sentry + estruturado JSON).
> - **12 categorias de erro** com código no padrão `DCE-{MÓDULO}-{HTTP_STATUS}{SEQ}`.
> - **Correlation ID** obrigatório em todo log e response de erro.
> - **Retry:** retryable (erros de infraestrutura: 500, 502, 503, 429) vs não-retryable (400, 401, 403, 404, 409, 422).
> - **Dados sensíveis** (`cpf`, `email`, `telefone`, `Authorization`) nunca aparecem em logs.
> - **Separação obrigatória** entre `message` (técnico) e `user_message` (UI em português, tom acolhedor).

---

## 1. Taxonomia de Erros

| Categoria | Código HTTP | Prefixo | Logging Level | Retryable | Recovery |
|---|---|---|---|---|---|
| **Validação** | 400 | `DCE-*-4000_*` | `warn` | Não | Corrigir payload e reenviar |
| **Autenticação** | 401 | `DCE-AUTH-4010_*` | `warn` | Não (exceto expirado → refresh) | Refresh token ou relogar |
| **Autorização** | 403 | `DCE-*-4030_*` | `warn` | Não | Sem ação — role insuficiente |
| **Não Encontrado** | 404 | `DCE-*-4040_*` | `info` | Não | Verificar ID e tentar novamente |
| **Conflito de Estado** | 409 | `DCE-*-4090_*` | `warn` | Não | Verificar estado atual do recurso |
| **Regra de Negócio** | 422 | `DCE-*-4220_*` | `warn` | Não | Corrigir conforme a regra violada |
| **Rate Limit** | 429 | `DCE-CHAT-4290_*` | `warn` | Sim (após retry_after) | Aguardar retry_after e reenviar |
| **Erro Interno** | 500 | `DCE-*-5000_*` | `error` | Sim (3x backoff) | Retry automático; persistir → suporte |
| **Serviço Externo** | 502/503 | `DCE-*-5020_*`, `DCE-*-5030_*` | `error` | Sim (3x backoff) | Retry automático; circuit breaker |
| **Agente Indisponível** | 503 | `DCE-CHAT-5030_001` | `error` | Não (manual) | Admin reativa via `/admin/fallback/enable` |
| **Webhook Inválido** | 401 | `DCE-WHKZ-4010_*` | `warn` | Não | Verificar HMAC secret |
| **Erro de Streaming SSE** | — | Evento `error` no stream | `error` | Sim (fechar e reconectar) | Frontend fecha EventSource e retenta |

---

## 2. Schema de Erro (Backend)

### 2.1 Schema Base

```json
{
  "error": {
    "code": "DCE-MODULE-HTTPSTATUS_SEQ",
    "category": "VALIDATION | AUTH | NOT_FOUND | CONFLICT | BUSINESS_RULE | RATE_LIMIT | INTERNAL | EXTERNAL_SERVICE | AGENT_UNAVAILABLE",
    "message": "Mensagem técnica para logs (sem PII)",
    "user_message": "Mensagem em português para exibição ao usuário",
    "correlation_id": "uuid-v4",
    "timestamp": "2026-03-23T10:00:00.000Z",
    "details": {}
  }
}
```

### 2.2 Exemplos por Categoria

**Exemplo 1 — Validação (400):**
```json
{
  "error": {
    "code": "DCE-CHAT-4000_004",
    "category": "VALIDATION",
    "message": "Message content is empty.",
    "user_message": "A mensagem não pode ser vazia. Digite algo para conversar com a Dani.",
    "correlation_id": "7f3e8a1b-9c2d-4e5f-a6b7-8c9d0e1f2a3b",
    "timestamp": "2026-03-23T10:00:00.000Z",
    "details": { "field": "content", "constraint": "minLength:1" }
  }
}
```

**Exemplo 2 — Rate Limit (429):**
```json
{
  "error": {
    "code": "DCE-CHAT-4290_001",
    "category": "RATE_LIMIT",
    "message": "Rate limit exceeded: 30 messages per hour for cedente_id hash:abc123.",
    "user_message": "Você atingiu o limite de 30 mensagens por hora. Tente novamente em 23 minutos.",
    "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "timestamp": "2026-03-23T10:00:00.000Z",
    "details": { "retry_after": 1380, "limit": 30, "window": "1h" }
  }
}
```

**Exemplo 3 — Agente Indisponível (503):**
```json
{
  "error": {
    "code": "DCE-CHAT-5030_001",
    "category": "AGENT_UNAVAILABLE",
    "message": "Agent circuit breaker OPEN. Error rate 32% in last 15 minutes.",
    "user_message": "A Dani está temporariamente indisponível. Nossa equipe já foi notificada. Tente novamente em alguns minutos.",
    "correlation_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "timestamp": "2026-03-23T10:00:00.000Z",
    "details": { "circuit_status": "OPEN", "error_rate": 0.32 }
  }
}
```

**Exemplo 4 — Conflito de Estado (409):**
```json
{
  "error": {
    "code": "DCE-PROP-4090_001",
    "category": "CONFLICT",
    "message": "Proposal prop-uuid already has status ACCEPTED. Cannot accept again.",
    "user_message": "Esta proposta já foi respondida. Veja o status atual na sua oportunidade.",
    "correlation_id": "c3d4e5f6-a7b8-9012-cdef-012345678901",
    "timestamp": "2026-03-23T10:00:00.000Z",
    "details": { "current_status": "ACCEPTED", "proposal_id": "prop-uuid" }
  }
}
```

---

## 3. Error Classes (Backend — NestJS)

### 3.1 Hierarquia

```typescript
// src/common/errors/base.error.ts

export abstract class DaniBaseError extends Error {
  abstract readonly code: string
  abstract readonly httpStatus: number
  abstract readonly category: ErrorCategory
  abstract readonly loggingLevel: 'info' | 'warn' | 'error'
  abstract readonly retryable: boolean
  readonly userMessage: string
  readonly details?: Record<string, unknown>

  constructor(userMessage: string, details?: Record<string, unknown>) {
    super(userMessage)
    this.userMessage = userMessage
    this.details = details
  }
}

// Erros concretos
export class ValidationError extends DaniBaseError {
  httpStatus = 400; category = 'VALIDATION'; loggingLevel = 'warn'; retryable = false
}
export class AuthenticationError extends DaniBaseError {
  httpStatus = 401; category = 'AUTH'; loggingLevel = 'warn'; retryable = false
}
export class AuthorizationError extends DaniBaseError {
  httpStatus = 403; category = 'AUTHORIZATION'; loggingLevel = 'warn'; retryable = false
}
export class NotFoundError extends DaniBaseError {
  httpStatus = 404; category = 'NOT_FOUND'; loggingLevel = 'info'; retryable = false
}
export class ConflictError extends DaniBaseError {
  httpStatus = 409; category = 'CONFLICT'; loggingLevel = 'warn'; retryable = false
}
export class BusinessRuleError extends DaniBaseError {
  httpStatus = 422; category = 'BUSINESS_RULE'; loggingLevel = 'warn'; retryable = false
}
export class RateLimitError extends DaniBaseError {
  httpStatus = 429; category = 'RATE_LIMIT'; loggingLevel = 'warn'; retryable = true
}
export class InternalError extends DaniBaseError {
  httpStatus = 500; category = 'INTERNAL'; loggingLevel = 'error'; retryable = true
}
export class ExternalServiceError extends DaniBaseError {
  httpStatus = 502; category = 'EXTERNAL_SERVICE'; loggingLevel = 'error'; retryable = true
}
export class AgentUnavailableError extends DaniBaseError {
  httpStatus = 503; category = 'AGENT_UNAVAILABLE'; loggingLevel = 'error'; retryable = false
}
```

### 3.2 GlobalExceptionFilter

```typescript
// src/common/filters/global-exception.filter.ts

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp()
    const response = ctx.getResponse<Response>()
    const request = ctx.getRequest<Request>()

    const correlationId = request.headers['x-correlation-id'] as string
                          || crypto.randomUUID()

    let httpStatus = 500
    let errorCode = 'DCE-SYS-5000_001'
    let message = 'Internal server error.'
    let userMessage = 'Ocorreu um erro inesperado. Tente novamente em alguns instantes.'
    let loggingLevel: 'info' | 'warn' | 'error' = 'error'
    let details: Record<string, unknown> | undefined

    if (exception instanceof DaniBaseError) {
      httpStatus = exception.httpStatus
      errorCode = exception.code
      message = exception.message
      userMessage = exception.userMessage
      loggingLevel = exception.loggingLevel
      details = exception.details
    } else if (exception instanceof HttpException) {
      httpStatus = exception.getStatus()
      message = String(exception.message)
      userMessage = 'Erro na requisição. Verifique os dados e tente novamente.'
      loggingLevel = httpStatus >= 500 ? 'error' : 'warn'
    }

    // Log estruturado (sem PII, sem stack trace no body de resposta)
    logger[loggingLevel]({
      correlationId,
      errorCode,
      category: exception instanceof DaniBaseError ? exception.category : 'UNKNOWN',
      message,
      requestPath: request.url,
      requestMethod: request.method,
      userId: request.user?.cedenteIdHash, // hash SHA-256, não UUID raw
      environment: process.env.NODE_ENV,
      // stack apenas em logs internos, nunca no response
      ...(loggingLevel === 'error' && { stack: exception instanceof Error ? exception.stack : undefined })
    })

    // Sentry: apenas erros >= 500
    if (httpStatus >= 500) {
      Sentry.captureException(exception, { extra: { correlationId, errorCode } })
    }

    response.status(httpStatus).json({
      error: {
        code: errorCode,
        category: exception instanceof DaniBaseError ? exception.category : 'INTERNAL',
        message: loggingLevel === 'error' ? 'Internal error.' : message,
        user_message: userMessage,
        correlation_id: correlationId,
        timestamp: new Date().toISOString(),
        ...(details && { details })
      }
    })
  }
}
```

---

## 4. Error Boundaries (Frontend — Chat Widget)

> 💡 **Contexto:** o chat widget é servido pelo frontend da Plataforma Repasse Seguro (Next.js). Os error boundaries abaixo são definidos pelo AI-Dani-Cedente como contrato para o frontend implementar.

### 4.1 Níveis de Boundary

| Nível | Erros capturados | Fallback UI | Retry |
|---|---|---|---|
| **Global (widget)** | Erros de runtime JS não tratados | Tela de erro com "Recarregar chat" | Recarregar widget |
| **Route-level (sessão)** | Falha ao carregar histórico de sessão | Skeleton com "Erro ao carregar conversa" + botão retry | GET /chat/sessions/:id/messages novamente |
| **Component-level (mensagem)** | Falha ao enviar mensagem (rede, timeout) | Toast de erro + botão "Tentar novamente" | POST /chat/sessions/:id/messages novamente |

### 4.2 Tratamento de Erros SSE

```typescript
// Tratamento de eventos SSE no frontend

const eventSource = new EventSource('/api/v1/chat/sessions/:id/messages', {
  headers: { Authorization: `Bearer ${token}` }
})

eventSource.addEventListener('error', (event) => {
  const errorData = JSON.parse(event.data)

  switch (errorData.code) {
    case 'DCE-CHAT-4290_001': // Rate limit
      showToast('warning', errorData.user_message, { duration: errorData.details.retry_after * 1000 })
      break
    case 'DCE-CHAT-5030_001': // Agente indisponível
      showAgentUnavailableState(errorData.user_message)
      eventSource.close()
      break
    default:
      showToast('error', errorData.user_message)
      eventSource.close()
  }
})

eventSource.addEventListener('done', () => {
  eventSource.close()
})
```

---

## 5. Anti-exemplos

> 🔴 **Nunca faça isso:**

**❌ Erro engolido sem feedback nem log:**
```typescript
try {
  await opportunityRepository.findById(id)
} catch (e) {
  // silêncio
}
```
✅ **Correto:**
```typescript
try {
  return await opportunityRepository.findById(id)
} catch (e) {
  logger.error({ message: e.message, correlationId, opportunityId: id })
  throw new InternalError('Falha ao buscar a oportunidade. Tente novamente.', { opportunityId: id })
}
```

**❌ Resposta genérica sem código:**
```json
{ "error": "Something went wrong." }
```
✅ **Correto:**
```json
{
  "error": {
    "code": "DCE-OPP-5000_001",
    "user_message": "Ocorreu um erro ao carregar sua oportunidade. Tente novamente.",
    "correlation_id": "uuid",
    "timestamp": "2026-03-23T10:00:00.000Z"
  }
}
```

**❌ Stack trace exposta ao usuário:**
```json
{ "error": "TypeError: Cannot read properties of undefined (reading 'id')\n    at OpportunityService.findById..." }
```
✅ **Correto:** stack trace apenas em logs internos (Sentry/logger). Response contém apenas `user_message` amigável.

**❌ Padrões de código mistos:**
```
// Módulo chat: "CHAT_ERROR_001"
// Módulo proposal: "PROP-ERR-400"
// Módulo escrow: "escrow_not_found"
```
✅ **Correto:** padrão único `DCE-{MÓDULO}-{HTTP_STATUS}{SEQ}` em todos os módulos.

---

## 6. Mapeamento API Error → UI

| Error Code | Componente UI | Mensagem exibida | Ação disponível |
|---|---|---|---|
| `DCE-AUTH-4010_001` | Toast warning | "Sua sessão expirou. Reconectando..." | Auto-refresh token |
| `DCE-AUTH-4010_003` | Redirect para login | "Autenticação necessária." | Botão "Fazer login" |
| `DCE-CHAT-4000_004` | Inline no input | "A mensagem não pode ser vazia." | Nenhuma — corrigir o campo |
| `DCE-CHAT-4290_001` | Toast warning persistente | "Limite de mensagens atingido. Tente em X min." | Timer countdown |
| `DCE-CHAT-5030_001` | Estado de indisponibilidade no chat | "A Dani está temporariamente indisponível." | Botão "Tentar mais tarde" |
| `DCE-PROP-4090_001` | Toast error | "Esta proposta já foi respondida." | Botão "Ver status" |
| `DCE-PROP-4220_001` | Toast error | "Proposta expirada." | Botão "Ver outras propostas" |
| `DCE-DOSS-4000_003` | Inline no input de upload | "O arquivo excede 10MB." | Botão "Escolher outro arquivo" |
| `DCE-ESCR-4090_001` | Toast info | "Nenhuma extensão foi solicitada." | Nenhuma |
| `DCE-OPP-4030_001` | Toast error | "Sem permissão para esta oportunidade." | Nenhuma (bloquear ação) |
| `DCE-SYS-5000_001` | Toast error genérico | "Erro inesperado. Tente novamente." | Botão "Tentar novamente" |
| `DCE-WHKZ-4010_001` | (sem UI — webhook) | — | Log + alert ao Admin |

---

## 7. Retry e Recovery

### 7.1 Erros Retryable

| Categoria | Código | Estratégia | Max Retries | Backoff | Após todas as tentativas |
|---|---|---|---|---|---|
| Rate limit | 429 | Aguardar `retry_after` | 1 | Fixo (retry_after) | Toast com countdown |
| Erro interno | 500 | Exponencial | 3 | 1s, 2s, 4s | Toast error + Sentry alert |
| Serviço externo indisponível | 502/503 | Exponencial | 3 | 2s, 4s, 8s | Circuit breaker + toast |
| Timeout SSE | — | Fechar e reconectar | 2 | 1s, 3s | Estado de indisponibilidade |

### 7.2 Erros Não-Retryable

| Categoria | Código | Comportamento | Mensagem |
|---|---|---|---|
| Validação | 400 | Corrigir e reenviar | Inline no campo com erro |
| Autenticação | 401 (inválido) | Redirect para login | Toast warning |
| Autorização | 403 | Bloquear ação | Toast error |
| Não encontrado | 404 | Sem ação | Toast info |
| Conflito | 409 | Verificar estado | Toast error + link para recurso |
| Regra de negócio | 422 | Corrigir conforme regra | Toast error com instrução |
| Agente desligado | 503 (circuit open) | Aguardar reativação manual | Estado especial no chat |

---

## 8. Logging de Erros

### 8.1 Campos Obrigatórios em Todo Log

```typescript
interface ErrorLog {
  correlationId: string         // UUID da requisição
  errorCode: string             // DCE-MODULE-STATUS_SEQ
  category: ErrorCategory       // VALIDATION | AUTH | etc.
  message: string               // Mensagem técnica (sem PII)
  requestPath: string           // /api/v1/chat/sessions
  requestMethod: string         // POST
  userId?: string               // SHA-256 do cedente_id (nunca UUID raw)
  environment: string           // development | staging | production
  timestamp: string             // ISO 8601
  stack?: string                // Apenas para loggingLevel === 'error'
}
```

### 8.2 Campos Nunca Incluídos em Logs

| Campo | Motivo |
|---|---|
| CPF completo | LGPD — PII sensível |
| Email | LGPD — PII |
| Telefone | LGPD — PII |
| `Authorization` header | Segurança — token |
| Senha ou hash de senha | Segurança |
| Conteúdo de mensagem de chat | LGPD — dado pessoal |
| `cedente_id` UUID raw | Substituído por SHA-256 hash |

### 8.3 Correlation ID

- Gerado no `GlobalExceptionFilter` se ausente no request.
- Propagado via header `X-Correlation-ID` em toda resposta de erro.
- Frontend deve incluir `X-Correlation-ID` em requests para correlacionar logs.
- Incluído em todo trace do Langfuse.

---

## 9. Observabilidade e Alertas

| Alert | Threshold | Canal | SLA de resposta |
|---|---|---|---|
| Taxa de `500` errors | > 5% em 5min | Sentry + Slack | 15 min |
| Taxa de erros OpenAI (`502`/`503`) | > 10% em 15min | Sentry + Slack | 15 min |
| Circuit breaker OPEN | Qualquer ocorrência | Sentry + Slack `#alertas-criticos` | 5 min |
| Rate limit atingido | > 10% dos Cedentes em 1h | PostHog + Slack | 1h |
| Error code desconhecido | Qualquer ocorrência | Sentry | 1h |

---

## 10. Testes de Error Handling

### 10.1 Cenários Obrigatórios em Testes

```typescript
// test/error-handling/global-exception-filter.spec.ts

describe('GlobalExceptionFilter', () => {
  it('deve retornar 400 com user_message para ValidationError')
  it('deve retornar 401 sem stack trace no body para AuthenticationError')
  it('deve retornar 404 para NotFoundError — sem revelar outros Cedentes')
  it('deve retornar 429 com retry_after para RateLimitError')
  it('deve retornar 503 com user_message amigável para AgentUnavailableError')
  it('deve incluir correlation_id em toda resposta de erro')
  it('deve nunca incluir stack trace no body da resposta')
  it('deve nunca incluir PII (CPF, email) em mensagem de erro')
  it('deve logar com Sentry apenas erros >= 500')
  it('deve mascarar userId como hash SHA-256 nos logs')
})
```

---

## 11. Glossário de Códigos de Erro

| Código | Descrição |
|---|---|
| `DCE-AUTH-4010_001` | Token JWT expirado |
| `DCE-AUTH-4010_002` | Token JWT inválido/malformado |
| `DCE-AUTH-4010_003` | Token JWT ausente |
| `DCE-AUTH-4030_001` | Role insuficiente |
| `DCE-CHAT-4000_001` | `entry_point` ausente em criação de sessão |
| `DCE-CHAT-4000_004` | Mensagem de chat vazia |
| `DCE-CHAT-4000_005` | Mensagem de chat > 2000 chars |
| `DCE-CHAT-4290_001` | Rate limit de 30 msgs/h excedido |
| `DCE-CHAT-5030_001` | Agente indisponível (circuit breaker OPEN) |
| `DCE-OPP-4030_001` | Acesso a oportunidade de outro Cedente |
| `DCE-OPP-4040_001` | Oportunidade não encontrada |
| `DCE-PROP-4000_001` | `confirmation` ausente ou `false` no aceite |
| `DCE-PROP-4090_001` | Proposta já respondida |
| `DCE-PROP-4220_001` | Proposta expirada |
| `DCE-DOSS-4000_003` | Arquivo > 10MB no upload do dossiê |
| `DCE-ESCR-4040_001` | Escrow não encontrado |
| `DCE-ESCR-4090_001` | Extensão não foi solicitada |
| `DCE-WHKZ-4010_001` | Assinatura HMAC do webhook ZapSign inválida |
| `DCE-SYS-5000_001` | Erro interno inesperado |
| `DCE-ADMN-4090_003` | Agente já está ativo (não pode reativar) |

---

## 12. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Error boundary no chat widget | `[DECISÃO AUTÔNOMA]` | 4 | Contrato definido pelo AI-Dani-Cedente — implementação é responsabilidade do frontend da Plataforma Repasse Seguro. Documentado como contrato de integração. | Médio | Frontend Lead | Pendente implementação |
| Correlation ID persistido no banco | `[DECISÃO AUTÔNOMA]` | 8.3 | Não persistido em `chat_messages` para evitar overhead. Rastreável via Langfuse `trace_id` + Sentry. Alternativa descartada: coluna correlation_id em chat_messages — complexidade sem benefício operacional claro. | Baixo | Backend | Decidido |
