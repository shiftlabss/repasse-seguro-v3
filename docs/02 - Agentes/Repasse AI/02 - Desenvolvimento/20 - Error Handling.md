# 20 - Error Handling

## Cabeçalho

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
| --- | --- | --- | --- | --- |
| 20 - Error Handling | v1.0 | 2026-03-22 | Claude Code Desktop (ShiftLabs Pipeline v9.5) | Aprovado |

---

## TL;DR

> 📌 **Princípios e cobertura de error handling do Repasse AI:**
>
> - **Contrato único:** backend, logs e UI falam a mesma língua — `error_code` como chave de identidade de todo erro.
> - **Camadas cobertas:** NestJS (backend), Webchat + WhatsApp SSE (canal de entrega), Supabase RLS, LangGraph.js (agente IA), integrações externas (OpenAI, EvolutionAPI, Upstash, RabbitMQ).
> - **10 categorias** de erro mapeadas com status HTTP, mensagem técnica, mensagem de UI e recovery.
> - **Correlation ID obrigatório** em todo log de erro — rastreabilidade end-to-end desde o request até o LLM.
> - **Stack trace nunca exposta** ao usuário — logs internos apenas.
> - **Retry com exponential backoff** para erros de integração externa (P0/P1); não-retryable com fallback imediato ou circuit breaker.
> - **Dados sensíveis (PII, JWT, OTP) excluídos** de todos os logs por `PiiMaskingInterceptor`.

---

## 1. Taxonomia de Erros

Esta seção define as categorias de erro do Repasse AI com status HTTP, mensagem técnica, mensagem de UI e estratégia de recovery. A taxonomia é a fonte de verdade para backend, frontend e observabilidade — nenhuma camada pode criar categorias fora desta tabela.

> 🔴 **Regra inegociável:** toda resposta de erro da API deve conter `error_code` desta taxonomia. Códigos inventados por módulo sem registro aqui são proibidos.

| Código | Categoria | Status HTTP | Mensagem Técnica | Mensagem de UI | Recovery | Logging Level |
| --- | --- | --- | --- | --- | --- | --- |
| `VALIDATION_ERROR` | Validação de input | 422 | Input inválido conforme DTO | "Verifique os dados enviados e tente novamente." | Retornar campo + motivo; não retryable | WARN |
| `AUTH_INVALID_TOKEN` | Autenticação | 401 | JWT inválido ou expirado | "Sua sessão expirou. Faça login novamente." | Redirecionar para re-auth; não retryable | WARN |
| `AUTH_INSUFFICIENT_PERMISSIONS` | Autorização | 403 | Role insuficiente para o recurso | "Você não tem permissão para esta ação." | Exibir mensagem; não retryable | WARN |
| `TENANT_MISMATCH` | Isolamento tenant | 403 | cessionario_id do token não corresponde ao recurso | "Acesso negado." | Exibir mensagem genérica; não retryable | ERROR |
| `NOT_FOUND` | Recurso inexistente | 404 | Entidade não encontrada no banco | "Item não encontrado." | Retornar lista vazia ou breadcrumb; não retryable | INFO |
| `CONFLICT` | Conflito de estado | 409 | Estado atual incompatível com a operação | "Esta ação não pode ser realizada no estado atual." | Recarregar estado atual; não retryable | WARN |
| `RATE_LIMIT_EXCEEDED` | Rate limiting | 429 | Limite de requisições atingido | "Você enviou muitas mensagens. Aguarde alguns instantes." | Retry após `Retry-After` header; retryable | WARN |
| `LLM_UNAVAILABLE` | Integração OpenAI | 503 | OpenAI API timeout ou 5xx | "O assistente está temporariamente indisponível. Tente em instantes." | Exponential backoff 3x; circuit breaker após 5 falhas | ERROR |
| `WHATSAPP_INTEGRATION_ERROR` | Integração EvolutionAPI | 502 | EvolutionAPI não respondeu ou retornou erro | "Não foi possível enviar via WhatsApp. Tente novamente." | Retry 2x; fallback para DLQ RabbitMQ | ERROR |
| `INTERNAL_ERROR` | Erro interno inesperado | 500 | Exceção não tratada no servidor | "Ocorreu um erro inesperado. Nossa equipe foi notificada." | Log + Sentry alert; não retryable pelo usuário | ERROR |
| `RAG_RETRIEVAL_ERROR` | Falha no RAG/pgvector | 503 | pgvector query falhou ou timeout | "O assistente não conseguiu acessar a base de conhecimento. Tente novamente." | Retry 1x; fallback sem RAG context | ERROR |
| `EXTERNAL_SERVICE_ERROR` | Serviços externos genéricos | 502 | Serviço externo (Upstash, CloudAMQP) indisponível | "Serviço temporariamente indisponível." | Retry com backoff; fail-open onde seguro | ERROR |

---

## 2. Schema de Erro (Backend)

Todo erro retornado pela API segue este schema JSON. Campos obrigatórios: `error_code`, `category`, `status`, `message`, `user_message`, `correlation_id`, `timestamp`, `metadata`.

> 🔴 **Regra inegociável:** `stack_trace` nunca aparece no payload de resposta ao cliente — apenas em logs internos. `metadata` não deve conter PII.

### 2.1 Schema Base

```json
{
  "error": {
    "error_code": "VALIDATION_ERROR",
    "category": "validation",
    "status": 422,
    "message": "Campo 'message' é obrigatório e não pode estar vazio.",
    "user_message": "Verifique os dados enviados e tente novamente.",
    "correlation_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD7",
    "timestamp": "2026-03-22T14:30:00.000Z",
    "metadata": {
      "field": "message",
      "constraint": "isNotEmpty",
      "path": "/repasse-ai/v1/chat/conversations/conv_abc123/messages"
    }
  }
}
```

### 2.2 Exemplo: Erro de Autenticação

```json
{
  "error": {
    "error_code": "AUTH_INVALID_TOKEN",
    "category": "authentication",
    "status": 401,
    "message": "JWT expirado: exp 1742600000 < now 1742600120.",
    "user_message": "Sua sessão expirou. Faça login novamente.",
    "correlation_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD8",
    "timestamp": "2026-03-22T14:32:00.000Z",
    "metadata": {
      "token_expired_at": "2026-03-22T14:30:00.000Z"
    }
  }
}
```

### 2.3 Exemplo: Erro de Integração Externa (LLM)

```json
{
  "error": {
    "error_code": "LLM_UNAVAILABLE",
    "category": "external_service",
    "status": 503,
    "message": "OpenAI API retornou 503 após 3 tentativas (último erro: upstream connect error).",
    "user_message": "O assistente está temporariamente indisponível. Tente em instantes.",
    "correlation_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD9",
    "timestamp": "2026-03-22T14:35:00.000Z",
    "metadata": {
      "provider": "openai",
      "model": "gpt-4o",
      "attempts": 3,
      "circuit_breaker_status": "OPEN"
    }
  }
}
```

### 2.4 Exemplo: Erro Interno Inesperado

```json
{
  "error": {
    "error_code": "INTERNAL_ERROR",
    "category": "internal",
    "status": 500,
    "message": "Erro não tratado no AiService.processMessage.",
    "user_message": "Ocorreu um erro inesperado. Nossa equipe foi notificada.",
    "correlation_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD0",
    "timestamp": "2026-03-22T14:36:00.000Z",
    "metadata": {
      "module": "ai",
      "service": "AiService",
      "method": "processMessage"
    }
  }
}
```

---

## 3. Error Classes (Backend)

Hierarquia de classes NestJS para o módulo `common/errors/`. Todas herdam de `BaseError` que estende `HttpException`.

> ⚙️ **Padrão obrigatório:** toda exceção lançada em qualquer módulo deve ser uma subclasse de `BaseError`. `throw new Error('algo')` é proibido nos módulos de domínio.

### 3.1 Hierarquia

```
BaseError (extends HttpException)
├── ValidationError          → 422 | VALIDATION_ERROR       | WARN  | não retryable
├── AuthError                → 401 | AUTH_INVALID_TOKEN      | WARN  | não retryable
├── ForbiddenError           → 403 | AUTH_INSUFFICIENT_PERMISSIONS | WARN | não retryable
├── TenantMismatchError      → 403 | TENANT_MISMATCH         | ERROR | não retryable
├── NotFoundError            → 404 | NOT_FOUND               | INFO  | não retryable
├── ConflictError            → 409 | CONFLICT                | WARN  | não retryable
├── RateLimitError           → 429 | RATE_LIMIT_EXCEEDED     | WARN  | retryable
├── LlmUnavailableError      → 503 | LLM_UNAVAILABLE         | ERROR | retryable (circuit breaker)
├── WhatsappIntegrationError → 502 | WHATSAPP_INTEGRATION_ERROR | ERROR | retryable
├── RagRetrievalError        → 503 | RAG_RETRIEVAL_ERROR     | ERROR | retryable (1x)
├── ExternalServiceError     → 502 | EXTERNAL_SERVICE_ERROR  | ERROR | retryable
└── InternalError            → 500 | INTERNAL_ERROR          | ERROR | não retryable pelo usuário
```

### 3.2 Implementação BaseError

```typescript
// apps/ai/src/common/errors/base.error.ts
import { HttpException } from '@nestjs/common';

export interface ErrorMetadata {
  [key: string]: unknown;
}

export abstract class BaseError extends HttpException {
  readonly errorCode: string;
  readonly category: string;
  readonly userMessage: string;
  readonly isRetryable: boolean;
  readonly correlationId: string;
  readonly metadata: ErrorMetadata;

  constructor(params: {
    errorCode: string;
    category: string;
    status: number;
    message: string;
    userMessage: string;
    isRetryable: boolean;
    correlationId: string;
    metadata?: ErrorMetadata;
  }) {
    super(params.message, params.status);
    this.errorCode = params.errorCode;
    this.category = params.category;
    this.userMessage = params.userMessage;
    this.isRetryable = params.isRetryable;
    this.correlationId = params.correlationId;
    this.metadata = params.metadata ?? {};
  }
}
```

### 3.3 Exemplo: LlmUnavailableError

```typescript
// apps/ai/src/common/errors/llm-unavailable.error.ts
import { BaseError } from './base.error';

export class LlmUnavailableError extends BaseError {
  constructor(correlationId: string, attempts: number, circuitBreakerStatus: string) {
    super({
      errorCode: 'LLM_UNAVAILABLE',
      category: 'external_service',
      status: 503,
      message: `OpenAI API indisponível após ${attempts} tentativas.`,
      userMessage: 'O assistente está temporariamente indisponível. Tente em instantes.',
      isRetryable: true,
      correlationId,
      metadata: { provider: 'openai', model: 'gpt-4o', attempts, circuit_breaker_status: circuitBreakerStatus },
    });
  }
}
```

### 3.4 Global Exception Filter

```typescript
// apps/ai/src/common/filters/all-exceptions.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, Logger } from '@nestjs/common';
import { BaseError } from '../errors/base.error';
import { Request, Response } from 'express';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const request = ctx.getRequest<Request>();
    const response = ctx.getResponse<Response>();
    const correlationId = (request.headers['x-correlation-id'] as string) ?? `req_${Date.now()}`;

    if (exception instanceof BaseError) {
      this.logger.log({
        level: this.getLogLevel(exception.getStatus()),
        error_code: exception.errorCode,
        category: exception.category,
        message: exception.message,
        correlation_id: correlationId,
        path: request.url,
        // stack_trace em logs internos somente — nunca na resposta
        stack: exception.stack,
      });
      return response.status(exception.getStatus()).json({
        error: {
          error_code: exception.errorCode,
          category: exception.category,
          status: exception.getStatus(),
          message: exception.message,
          user_message: exception.userMessage,
          correlation_id: correlationId,
          timestamp: new Date().toISOString(),
          metadata: exception.metadata,
        },
      });
    }

    // Erro não tratado — mascarar detalhes
    this.logger.error({
      error_code: 'INTERNAL_ERROR',
      message: exception instanceof Error ? exception.message : 'Unknown error',
      stack: exception instanceof Error ? exception.stack : undefined,
      correlation_id: correlationId,
      path: request.url,
    });
    return response.status(500).json({
      error: {
        error_code: 'INTERNAL_ERROR',
        category: 'internal',
        status: 500,
        message: 'Erro interno não tratado.',
        user_message: 'Ocorreu um erro inesperado. Nossa equipe foi notificada.',
        correlation_id: correlationId,
        timestamp: new Date().toISOString(),
        metadata: {},
      },
    });
  }

  private getLogLevel(status: number): string {
    if (status >= 500) return 'error';
    if (status >= 400) return 'warn';
    return 'info';
  }
}
```

---

## 4. Error Boundaries (Frontend)

> 📌 **Contexto PG-03:** Repasse AI é backend puro. O "frontend" do Repasse AI é o Webchat embeddado na plataforma Cessionário (Next.js) e a interface WhatsApp (via EvolutionAPI). Error boundaries cobrem o componente Webchat e a integração SSE — não uma SPA autônoma.

Esta seção documenta os três níveis de boundary para o Webchat e os padrões de tratamento de erro no SSE stream.

### 4.1 Níveis de Boundary

| Nível | Escopo | Tipo de Erro Capturado | Fallback UI | Integração Observabilidade | Retry |
| --- | --- | --- | --- | --- | --- |
| **Global** | Toda a aplicação Cessionário | Erro JS não tratado, falha de hydration | Tela de erro com botão "Recarregar página" | Sentry.captureException | Não automático |
| **Route-level** | Rota do Webchat (ex: `/chat`) | Falha no carregamento de dados iniciais, 404 de conversas | Skeleton + mensagem "Não foi possível carregar. Tente novamente." | Sentry + log estruturado | Botão manual |
| **Component-level** | ChatBubble, MessageInput, StatusBadge | Erro de renderização de componente individual | Substituir componente por placeholder com `⚠️ Conteúdo indisponível` | Log local + Sentry | Automático na remontagem |

### 4.2 Tratamento de Erros no SSE Stream

O stream SSE pode ser interrompido por timeout, erro de rede ou erro do agente. O cliente deve tratar:

```typescript
// Webchat SSE error handling
const eventSource = new EventSource(`/repasse-ai/v1/chat/conversations/${id}/messages/stream`, {
  headers: { Authorization: `Bearer ${token}` },
});

eventSource.addEventListener('error', (event) => {
  if (eventSource.readyState === EventSource.CLOSED) {
    // Conexão encerrada — exibir estado de erro e botão de retry
    setChatState({ status: 'error', errorCode: 'SSE_CONNECTION_LOST' });
    trackError('SSE_CONNECTION_LOST', { conversation_id: id });
  }
});

eventSource.addEventListener('message', (event) => {
  const data = JSON.parse(event.data);
  if (data === '[DONE]') {
    eventSource.close();
    return;
  }
  if (data.type === 'error') {
    // Erro estruturado vindo do backend via SSE
    setChatState({ status: 'error', errorCode: data.error_code, userMessage: data.user_message });
    eventSource.close();
  }
});
```

### 4.3 Exemplo Correto vs Anti-Exemplo

✅ **Correto — boundary com fallback contextual:**
```tsx
// ChatArea com error boundary
function ChatArea() {
  const { data, error, isLoading } = useConversationMessages(conversationId);

  if (isLoading) return <ChatSkeleton />;
  if (error) return (
    <ErrorState
      title="Não foi possível carregar as mensagens"
      action={{ label: 'Tentar novamente', onClick: () => refetch() }}
      correlationId={error.correlationId}
    />
  );
  return <MessageList messages={data} />;
}
```

❌ **Anti-exemplo — erro engolido:**
```tsx
// PROIBIDO — erro nunca chega ao usuário
function ChatArea() {
  const { data } = useConversationMessages(conversationId);
  // Se data for undefined por erro, renderiza lista vazia sem feedback
  return <MessageList messages={data ?? []} />;
}
```

---

## 5. Mapeamento API Error → UI

Lookup table canônica de `error_code` → componente de UI → mensagem exibida → ação disponível.

> ⚙️ **Padrão obrigatório:** o Webchat consome apenas `user_message` do schema de erro para exibição. A mensagem técnica (`message`) nunca é exibida ao usuário.

| `error_code` | Componente UI | Mensagem Exibida | Ação Disponível |
| --- | --- | --- | --- |
| `VALIDATION_ERROR` | Toast (warning) | "Verifique os dados e tente novamente." | Destacar campo inválido |
| `AUTH_INVALID_TOKEN` | Modal bloqueante | "Sua sessão expirou. Faça login novamente." | Botão "Fazer login" → redirect |
| `AUTH_INSUFFICIENT_PERMISSIONS` | Toast (error) | "Você não tem permissão para esta ação." | Dismiss |
| `TENANT_MISMATCH` | Toast (error) | "Acesso negado." | Dismiss |
| `NOT_FOUND` | Empty state inline | "Nenhum resultado encontrado." | Botão "Voltar" |
| `RATE_LIMIT_EXCEEDED` | Toast (warning) com countdown | "Muitas mensagens. Aguarde {segundos}s." | Countdown + retry automático |
| `LLM_UNAVAILABLE` | ChatBubble (error variant) | "O assistente está temporariamente indisponível. Tente em instantes." | Botão "Tentar novamente" |
| `WHATSAPP_INTEGRATION_ERROR` | Toast (error) | "Não foi possível enviar via WhatsApp. Tente novamente." | Botão "Tentar novamente" |
| `RAG_RETRIEVAL_ERROR` | ChatBubble (warning variant) | "O assistente respondeu sem acesso à base de conhecimento." | Nenhuma — resposta parcial exibida |
| `INTERNAL_ERROR` | Toast (error) | "Ocorreu um erro inesperado. Nossa equipe foi notificada." | Dismiss + ID do erro para suporte |
| `CONFLICT` | Toast (warning) | "Esta ação não pode ser realizada no estado atual." | Botão "Recarregar" |
| `EXTERNAL_SERVICE_ERROR` | Toast (error) | "Serviço temporariamente indisponível." | Botão "Tentar novamente" |

### 5.1 Exibição do Correlation ID ao Usuário

Para `INTERNAL_ERROR`, exibir o `correlation_id` truncado como referência de suporte:

```
"Ocorreu um erro inesperado. Nossa equipe foi notificada.
Ref: req_01HX4M3...NTD0"
```

Isso permite que o suporte correlacione o relato do usuário ao log interno sem expor detalhes técnicos.

---

## 6. Retry e Recovery

### 6.1 Classificação por Retryable

| `error_code` | Retryable? | Estratégia | Max Retries | Backoff | Jitter | Fallback se Esgotado |
| --- | --- | --- | --- | --- | --- | --- |
| `VALIDATION_ERROR` | Não | — | 0 | — | — | Retornar 422 imediatamente |
| `AUTH_INVALID_TOKEN` | Não | — | 0 | — | — | Redirect para re-auth |
| `AUTH_INSUFFICIENT_PERMISSIONS` | Não | — | 0 | — | — | Exibir mensagem de permissão |
| `TENANT_MISMATCH` | Não | — | 0 | — | — | Exibir "Acesso negado" |
| `NOT_FOUND` | Não | — | 0 | — | — | Exibir empty state |
| `CONFLICT` | Não | — | 0 | — | — | Recarregar estado |
| `RATE_LIMIT_EXCEEDED` | Sim | Fixed delay | — | `Retry-After` header | Não | Exibir countdown |
| `LLM_UNAVAILABLE` | Sim | Exponential backoff | 3 | 1s → 2s → 4s | ±200ms | Circuit breaker → erro ao usuário |
| `WHATSAPP_INTEGRATION_ERROR` | Sim | Exponential backoff | 2 | 2s → 4s | ±500ms | DLQ RabbitMQ |
| `RAG_RETRIEVAL_ERROR` | Sim | Retry simples | 1 | 1s | Não | Responder sem contexto RAG |
| `EXTERNAL_SERVICE_ERROR` | Sim | Exponential backoff | 2 | 1s → 2s | ±300ms | Fail-open onde seguro |
| `INTERNAL_ERROR` | Não | — | 0 | — | — | Log + Sentry alert |

### 6.2 Exponential Backoff com Jitter (Implementação)

```typescript
// apps/ai/src/common/utils/retry.util.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  opts: {
    maxRetries: number;
    baseDelayMs: number;
    jitterMs?: number;
    onRetry?: (attempt: number, error: unknown) => void;
  },
): Promise<T> {
  let lastError: unknown;
  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      if (attempt === opts.maxRetries) break;
      const delay = opts.baseDelayMs * Math.pow(2, attempt);
      const jitter = opts.jitterMs ? Math.random() * opts.jitterMs - opts.jitterMs / 2 : 0;
      opts.onRetry?.(attempt + 1, error);
      await new Promise((r) => setTimeout(r, Math.max(0, delay + jitter)));
    }
  }
  throw lastError;
}
```

### 6.3 Circuit Breaker (LLM)

O circuit breaker do agente IA está documentado no Doc 19 — Criação de Agentes de IA. Parâmetros:
- **Threshold de abertura:** 5 falhas consecutivas em 5 minutos
- **Estado:** `CLOSED` (normal) → `OPEN` (bloqueado) → `HALF_OPEN` (teste)
- **Reset:** automático após 5 minutos em estado OPEN
- **Ao abrir:** lança `LlmUnavailableError` com `circuit_breaker_status: "OPEN"` imediatamente, sem tentar o LLM

---

## 7. Logging de Erros

### 7.1 Campos Obrigatórios em Todo Log de Erro

```typescript
interface ErrorLog {
  level: 'info' | 'warn' | 'error';
  correlation_id: string;        // obrigatório — rastreabilidade end-to-end
  error_code: string;            // taxonomia desta seção
  category: string;              // categoria do erro
  message: string;               // mensagem técnica — não exposta ao usuário
  stack_trace?: string;          // apenas em logs internos — nunca na resposta HTTP
  user_id_hash?: string;         // hash SHA-256 do user_id — nunca o ID em claro
  cessionario_id_hash?: string;  // hash SHA-256 do cessionario_id
  timestamp: string;             // ISO 8601 UTC
  request_path: string;          // ex: /repasse-ai/v1/chat/conversations/.../messages
  request_method?: string;       // GET, POST, etc.
  environment: 'development' | 'staging' | 'production';
  module?: string;               // ai, chat, calculator, notification, supervision, whatsapp
  service?: string;              // nome do Service NestJS
  duration_ms?: number;          // tempo de execução quando relevante
}
```

### 7.2 Exclusões Obrigatórias de Dados Sensíveis

> 🔴 **Regra inegociável:** os campos abaixo nunca aparecem em nenhum log. O `PiiMaskingInterceptor` remove automaticamente antes do log ser escrito.

| Campo Sensível | Tratamento |
| --- | --- |
| `Authorization` header (JWT) | Removido — nunca logado |
| `password`, `otp`, `token` | Removido — nunca logado |
| `cpf`, `cnpj`, `telefone` | Mascarado: `***.***.***-**` |
| `email` | Mascarado: `us**@do***.com` |
| `message` do usuário (conteúdo) | Truncado a 100 chars em WARN/ERROR |
| IDs de usuário em claro | Substituído por `SHA-256(user_id)[:8]` |
| Variáveis de ambiente | Removido — nunca logado |

### 7.3 Exemplo de Log Estruturado (JSON)

```json
{
  "level": "error",
  "correlation_id": "req_01HX4M3P9QKJZ8B2VCWF5NTD9",
  "error_code": "LLM_UNAVAILABLE",
  "category": "external_service",
  "message": "OpenAI API indisponível após 3 tentativas.",
  "user_id_hash": "a3f8c2d1",
  "cessionario_id_hash": "b7e4f9a0",
  "timestamp": "2026-03-22T14:35:00.000Z",
  "request_path": "/repasse-ai/v1/chat/conversations/conv_abc123/messages",
  "request_method": "POST",
  "environment": "production",
  "module": "ai",
  "service": "AiService",
  "duration_ms": 12350,
  "metadata": {
    "provider": "openai",
    "model": "gpt-4o",
    "attempts": 3,
    "circuit_breaker_status": "OPEN"
  }
}
```

### 7.4 Anti-Exemplos de Logging

❌ **Anti-exemplo 1 — Stack trace exposta ao usuário:**
```json
// PROIBIDO — resposta HTTP com stack trace
{
  "statusCode": 500,
  "message": "Error: Cannot read property 'id' of undefined\n  at AiService.processMessage (/apps/ai/src/ai/services/ai.service.ts:45:12)\n  at ...",
  "error": "Internal Server Error"
}
```
✅ **Correto:** retornar `INTERNAL_ERROR` com `user_message` genérica e `correlation_id` para diagnóstico.

---

❌ **Anti-exemplo 2 — Erro engolido sem log:**
```typescript
// PROIBIDO — erro silenciado
async processMessage(dto: CreateMessageDto) {
  try {
    return await this.llmService.invoke(dto);
  } catch (e) {
    return null; // silenciado — usuário não sabe o que aconteceu
  }
}
```
✅ **Correto:** relançar como `LlmUnavailableError` ou `InternalError` com `correlationId`.

---

❌ **Anti-exemplo 3 — Resposta genérica sem código:**
```json
// PROIBIDO — sem error_code, sem correlationId
{
  "error": "Something went wrong"
}
```
✅ **Correto:** schema completo com `error_code`, `user_message` e `correlation_id`.

---

❌ **Anti-exemplo 4 — Padrões de código mistos:**
```typescript
// PROIBIDO — três padrões diferentes no mesmo produto
throw new HttpException('User not found', 404);                    // módulo chat
throw { status: 404, message: 'Opportunity not found' };          // módulo calculator
return res.status(404).json({ error: 'Conversation not found' }); // módulo supervision
```
✅ **Correto:** usar sempre `throw new NotFoundError(correlationId, 'Entidade', id)`.

---

## 8. Observabilidade e Alertas

Esta seção define os padrões de observabilidade para erros — integrações Sentry e Langfuse, métricas e alertas automáticos. O Doc 25 (Observabilidade e Logs) detalha a configuração completa; aqui registramos os contratos de erro.

### 8.1 Integração Sentry

```typescript
// Sentry captura apenas erros 5xx e erros de integração crítica
Sentry.withScope((scope) => {
  scope.setTag('error_code', error.errorCode);
  scope.setTag('module', error.metadata.module);
  scope.setExtra('correlation_id', error.correlationId);
  scope.setExtra('metadata', error.metadata);
  // user_id nunca em claro — apenas hash
  scope.setUser({ id: hashUserId(userId) });
  Sentry.captureException(error);
});
```

**Alertas automáticos por `error_code`:**

| `error_code` | Threshold | Canal | Severidade |
| --- | --- | --- | --- |
| `LLM_UNAVAILABLE` | ≥ 3 em 5 min | PagerDuty + Slack #alerts | P1 |
| `INTERNAL_ERROR` | ≥ 10 em 5 min | PagerDuty + Slack #alerts | P1 |
| `WHATSAPP_INTEGRATION_ERROR` | ≥ 5 em 10 min | Slack #alerts | P2 |
| `TENANT_MISMATCH` | ≥ 1 | Slack #security + Sentry | P1 |
| `RAG_RETRIEVAL_ERROR` | ≥ 10 em 5 min | Slack #alerts | P2 |

### 8.2 Langfuse — Traces de Erro do Agente

Erros no pipeline LangGraph.js são registrados como spans com status `ERROR` no Langfuse. Campos de trace obrigatórios em caso de erro:

```typescript
const span = trace.span({
  name: 'agent.processMessage',
  input: { conversation_id, intent }, // sem conteúdo de mensagem
  metadata: { cessionario_id_hash: hashId(cessionario_id) },
});
span.end({
  output: null,
  level: 'ERROR',
  statusMessage: error.errorCode,
  metadata: { correlation_id: error.correlationId, attempts: error.metadata.attempts },
});
```

### 8.3 Health Check de Erros

O endpoint `GET /health` responde 200 mesmo quando serviços externos estão degradados (fail-safe), mas reporta estado no corpo:

```json
{
  "status": "degraded",
  "services": {
    "openai": { "status": "down", "circuit_breaker": "OPEN", "error_code": "LLM_UNAVAILABLE" },
    "supabase": { "status": "up" },
    "redis": { "status": "up" },
    "rabbitmq": { "status": "up" },
    "evolution_api": { "status": "up" }
  }
}
```

---

## 9. Testes de Error Handling

> ⚙️ **Padrão obrigatório:** todo erro da taxonomia deve ter ao menos um teste unitário (comportamento da classe) e um teste de integração (resposta HTTP). O Plano de Testes (Doc 27) detalha a cobertura completa.

### 9.1 Testes Unitários de Error Classes

```typescript
// apps/ai/src/common/errors/__tests__/llm-unavailable.error.spec.ts
describe('LlmUnavailableError', () => {
  it('deve ter status 503 e error_code LLM_UNAVAILABLE', () => {
    const error = new LlmUnavailableError('req_test', 3, 'OPEN');
    expect(error.getStatus()).toBe(503);
    expect(error.errorCode).toBe('LLM_UNAVAILABLE');
    expect(error.isRetryable).toBe(true);
  });

  it('não deve expor stack trace no payload de resposta', () => {
    const error = new LlmUnavailableError('req_test', 3, 'OPEN');
    const payload = error.getResponse();
    expect(JSON.stringify(payload)).not.toContain('stack');
  });
});
```

### 9.2 Testes de Integração (AllExceptionsFilter)

```typescript
// apps/ai/src/common/filters/__tests__/all-exceptions.filter.spec.ts
describe('AllExceptionsFilter', () => {
  it('deve retornar 401 com error_code AUTH_INVALID_TOKEN', async () => {
    const response = await request(app.getHttpServer())
      .post('/repasse-ai/v1/chat/conversations/test/messages')
      .set('Authorization', 'Bearer token_invalido')
      .send({ message: 'Olá' });

    expect(response.status).toBe(401);
    expect(response.body.error.error_code).toBe('AUTH_INVALID_TOKEN');
    expect(response.body.error.correlation_id).toBeDefined();
    expect(response.body.error).not.toHaveProperty('stack');
  });

  it('erro não tratado deve retornar INTERNAL_ERROR sem stack trace', async () => {
    jest.spyOn(aiService, 'processMessage').mockRejectedValueOnce(new Error('Erro inesperado'));
    const response = await request(app.getHttpServer())
      .post('/repasse-ai/v1/chat/conversations/conv_test/messages')
      .set('Authorization', `Bearer ${validJwt}`)
      .send({ message: 'Teste' });

    expect(response.status).toBe(500);
    expect(response.body.error.error_code).toBe('INTERNAL_ERROR');
    expect(JSON.stringify(response.body)).not.toContain('stack');
  });
});
```

### 9.3 Testes de Retry

```typescript
// apps/ai/src/common/utils/__tests__/retry.util.spec.ts
describe('withRetry', () => {
  it('deve tentar maxRetries+1 vezes e lançar o último erro', async () => {
    const fn = jest.fn().mockRejectedValue(new Error('timeout'));
    await expect(withRetry(fn, { maxRetries: 3, baseDelayMs: 10 })).rejects.toThrow('timeout');
    expect(fn).toHaveBeenCalledTimes(4); // 1 original + 3 retries
  });

  it('deve ter sucesso se a função passar antes de esgotar retries', async () => {
    const fn = jest.fn()
      .mockRejectedValueOnce(new Error('timeout'))
      .mockResolvedValueOnce('ok');
    const result = await withRetry(fn, { maxRetries: 2, baseDelayMs: 10 });
    expect(result).toBe('ok');
    expect(fn).toHaveBeenCalledTimes(2);
  });
});
```

---

## 10. Glossário de Códigos de Erro

Glossário completo de todos os `error_code` usados no Repasse AI, com prefixo de módulo.

| `error_code` | Prefixo de Log | Módulo | Retryable | HTTP | Descrição Curta |
| --- | --- | --- | --- | --- | --- |
| `VALIDATION_ERROR` | `[APP]` | Todos | Não | 422 | Input não passa no DTO |
| `AUTH_INVALID_TOKEN` | `[APP]` | Todos (guard) | Não | 401 | JWT inválido ou expirado |
| `AUTH_INSUFFICIENT_PERMISSIONS` | `[APP]` | Todos (guard) | Não | 403 | Role insuficiente |
| `TENANT_MISMATCH` | `[DB]` | Todos (RLS) | Não | 403 | Cross-tenant detectado |
| `NOT_FOUND` | `[APP]` | Todos | Não | 404 | Entidade não existe |
| `CONFLICT` | `[APP]` | chat, supervision | Não | 409 | Estado incompatível |
| `RATE_LIMIT_EXCEEDED` | `[APP]` | chat, whatsapp | Sim | 429 | Limite de req atingido |
| `LLM_UNAVAILABLE` | `[AI]` | ai | Sim | 503 | OpenAI indisponível |
| `WHATSAPP_INTEGRATION_ERROR` | `[WA]` | whatsapp | Sim | 502 | EvolutionAPI falhou |
| `RAG_RETRIEVAL_ERROR` | `[AI]` | ai | Sim (1x) | 503 | pgvector query falhou |
| `EXTERNAL_SERVICE_ERROR` | `[APP]` | Todos | Sim | 502 | Serviço externo genérico |
| `INTERNAL_ERROR` | `[APP]` | Todos | Não | 500 | Erro interno inesperado |
| `CHAT_MESSAGE_SEND_FAILED` | `[CHAT]` | chat | Sim | 503 | Falha ao persistir mensagem |
| `NOTIFICATION_SEND_FAILED` | `[NOTIF]` | notification | Sim | 503 | Push notification falhou |
| `SUPERVISION_TAKEOVER_CONFLICT` | `[SUPER]` | supervision | Não | 409 | Mutex de takeover ocupado |
| `WHATSAPP_OTP_EXPIRED` | `[WA]` | whatsapp | Não | 422 | OTP expirado ou inválido |
| `WHATSAPP_OTP_MAX_ATTEMPTS` | `[WA]` | whatsapp | Não | 429 | Máximo de tentativas OTP |

---

## 11. Backlog de Pendências

| ID | Descrição | Prioridade | Doc Dependente |
| --- | --- | --- | --- |
| EH-001 | Validar se `correlation_id` será gerado via `nanoid` ou `uuid v7` — impacta ordenação por timestamp nos logs | Média | 25 - Observabilidade |
| EH-002 | Definir threshold exato de alertas para `RATE_LIMIT_EXCEEDED` em ambiente staging (diferente de produção) | Baixa | 25 - Observabilidade |
| EH-003 | Confirmar se `TENANT_MISMATCH` deve gerar alerta imediato de segurança em staging ou apenas em produção | Alta | 25 - Observabilidade |

> **Decisões Autônomas Tomadas Neste Documento:**
>
> 1. **[DECISÃO AUTÔNOMA] Hash de user_id em logs:** SHA-256 truncado a 8 chars hex — opção alternativa (omitir user_id completamente) descartada por dificultar correlação de incidentes sem comprometer privacidade. Critério: rastreabilidade mínima conforme LGPD (pseudonimização).
> 2. **[DECISÃO AUTÔNOMA] RAG_RETRIEVAL_ERROR como retryable 1x com fallback:** alternativa (lançar 503 imediatamente) descartada porque o agente pode responder com qualidade reduzida sem contexto RAG, melhorando a experiência do usuário. Critério: fail-graceful documentado no Doc 19.
> 3. **[DECISÃO AUTÔNOMA] Exibir correlation_id truncado ao usuário em INTERNAL_ERROR:** alternativa (esconder completamente) descartada porque facilita o atendimento de suporte sem expor IDs internos completos. Critério: equilíbrio entre privacidade e operabilidade.

---

*Próximo documento do pipeline: D21 — Notificações, Templates e Implementação.*
