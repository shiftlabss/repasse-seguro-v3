# 20 - Error Handling

## Módulo Cessionário · Plataforma Repasse Seguro

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 20 - Error Handling | v1.0 | 2026-03-22 (America/Fortaleza) | Claude Code Desktop | Aprovado |

---

> 📌 **TL;DR**
>
> - **Princípios:** contrato único de erro entre backend, frontend e logs — stack trace nunca exposta ao usuário, mensagem técnica sempre separada da mensagem de UI.
> - **Camadas cobertas:** NestJS backend (exception filters), React frontend (Error Boundaries), mobile (React Native Error Boundary), Sentry (observabilidade global).
> - **13 categorias de erro** com código, status HTTP, logging level e recovery definidos.
> - **Prefixos de código por domínio:** AUTH, USR, CES, OPR, PRP, NEG, FRM, FIN, NOT, AI, DB, CACHE, STG.
> - **Retry:** exponential backoff (1s → 2s → 4s) apenas para erros 5xx e rate limiting (429). Erros 4xx nunca fazem retry automático.
> - **Correlation ID obrigatório** em todo log de erro e em toda resposta de erro da API.
> - **Dados sensíveis (CPF, JWT, senhas):** nunca aparecem em logs ou respostas de erro.

---

## 1. Taxonomia de Erros

| Código | Categoria | Status HTTP | Mensagem Técnica | Mensagem de UI | Recovery | Logging Level |
|---|---|---|---|---|---|---|
| `AUTH-001` | Credenciais inválidas | 401 | Invalid credentials | "E-mail ou senha incorretos. Verifique seus dados ou recupere sua senha." | Usuário corrige credenciais | `warn` |
| `AUTH-003` | Brute-force lockout | 429 | Too many login attempts | "Muitas tentativas. Aguarde {X} minutos antes de tentar novamente." | Aguardar TTL | `warn` |
| `AUTH-008` | Token expirado/inválido | 401 | Token expired or invalid | Redirect automático para /login | Frontend tenta refresh; se falhar, redirect | `info` |
| `AUTH-010` | Re-autenticação falhou | 401 | Re-authentication failed | "Não foi possível confirmar sua identidade. Verifique sua senha." | Usuário tenta novamente | `warn` |
| `VAL-001` | Validação de input | 422 | Validation failed: {field} {rule} | Mensagem por campo (UX Writing D08) | Usuário corrige o campo | `info` |
| `NOT-FOUND-001` | Recurso não encontrado | 404 | Resource {type} not found: {id} | "Este conteúdo não está mais disponível." | Redirect para listagem | `info` |
| `CONFLICT-001` | Conflito de estado | 409 | State conflict: {current} → {expected} | "Esta ação não pode ser realizada no estado atual. {instrução}" | Usuário vê estado atual e toma ação | `warn` |
| `RATE-001` | Rate limit de negócio | 429 | Business rate limit exceeded: {rule} | "Você atingiu o limite de {X} {ação} por {período}." | Aguardar janela | `warn` |
| `PERM-001` | Sem permissão | 403 | Insufficient permissions for {action} | "Você não tem permissão para realizar esta ação." | KYC ou suporte | `warn` |
| `EXT-001` | Serviço externo indisponível | 502 | External service unavailable: {service} | "Estamos com uma instabilidade temporária. Tente novamente em instantes." | Retry automático 3x | `error` |
| `EXT-002` | Timeout de serviço externo | 504 | External service timeout: {service} after {ms}ms | "A operação demorou mais que o esperado. Tente novamente." | Retry automático 2x | `error` |
| `DB-001` | Erro de banco de dados | 500 | Database error: {operation} | "Estamos com uma instabilidade interna. Nosso time foi notificado." | Retry automático (Prisma retry) | `error` |
| `INT-001` | Erro interno inesperado | 500 | Unexpected internal error | "Ocorreu um erro inesperado. Nosso time foi notificado automaticamente." | Suporte se persistir | `error` (Sentry alert) |

---

## 2. Schema de Erro (Backend)

> ⚙️ **Regra:** Todo response de erro da API deve seguir exatamente este schema. Nunca retornar JSON diferente para erros. Nunca incluir stack trace no response.

```typescript
// Schema padrão de erro — todos os endpoints seguem este contrato
interface ErrorResponse {
  error: {
    code: string;          // Ex: "AUTH-001", "VAL-001", "EXT-001"
    message: string;       // Mensagem técnica para logs (NÃO exibir ao usuário)
    user_message: string;  // Mensagem para exibir ao usuário (UX Writing D08)
    correlation_id: string; // UUID v4 — rastreabilidade entre camadas
    timestamp: string;     // ISO 8601 UTC
    metadata?: Record<string, unknown>; // Contexto adicional (ex: campos com erro)
  }
}
```

### 2.1 Exemplos de Schema

**Erro de validação (`VAL-001`):**
```json
{
  "error": {
    "code": "VAL-001",
    "message": "Validation failed: proposed_value must be a positive integer",
    "user_message": "O valor da proposta deve ser um número positivo em centavos.",
    "correlation_id": "a3f8c2d1-4e5b-4a6c-8f7d-9e0b1c2d3e4f",
    "timestamp": "2026-03-22T02:00:00.000Z",
    "metadata": {
      "fields": [
        { "field": "proposed_value", "rule": "min", "received": -100 }
      ]
    }
  }
}
```

**Erro de serviço externo (`EXT-001`):**
```json
{
  "error": {
    "code": "EXT-001",
    "message": "External service unavailable: zapsign",
    "user_message": "Estamos com uma instabilidade temporária na geração de documentos. Tente novamente em instantes.",
    "correlation_id": "b4e9d3e2-5f6c-5b7d-9e0f-0a1c2d3e4f5a",
    "timestamp": "2026-03-22T02:05:00.000Z",
    "metadata": {
      "service": "zapsign",
      "attempt": 3,
      "retries_exhausted": true
    }
  }
}
```

**Erro de conflito de estado (`CONFLICT-001`):**
```json
{
  "error": {
    "code": "CONFLICT-001",
    "message": "State conflict: proposal status EXPIRED → cannot be accepted",
    "user_message": "Esta proposta expirou e não pode mais ser aceita. Verifique suas propostas ativas.",
    "correlation_id": "c5f0e4f3-6a7d-6c8e-0f1a-1b2c3d4e5f6b",
    "timestamp": "2026-03-22T02:10:00.000Z",
    "metadata": {
      "entity": "proposal",
      "current_status": "EXPIRED",
      "attempted_transition": "ACCEPT"
    }
  }
}
```

---

## 3. Error Classes (Backend — NestJS)

### 3.1 Hierarquia de Classes

```typescript
// apps/api/src/common/errors/base.error.ts
export abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly errorCode: string;
  abstract readonly loggingLevel: 'info' | 'warn' | 'error';
  abstract readonly isRetryable: boolean;

  constructor(
    public readonly message: string,         // técnica
    public readonly userMessage: string,     // para o usuário
    public readonly metadata?: Record<string, unknown>
  ) {
    super(message)
    this.name = this.constructor.name
  }
}

// Subclasses
export class ValidationError extends AppError {
  readonly statusCode = 422;
  readonly errorCode = 'VAL-001';
  readonly loggingLevel = 'info' as const;
  readonly isRetryable = false;
}

export class AuthError extends AppError {
  readonly statusCode = 401;
  readonly errorCode = 'AUTH-001';
  readonly loggingLevel = 'warn' as const;
  readonly isRetryable = false;
}

export class ForbiddenError extends AppError {
  readonly statusCode = 403;
  readonly errorCode = 'PERM-001';
  readonly loggingLevel = 'warn' as const;
  readonly isRetryable = false;
}

export class NotFoundError extends AppError {
  readonly statusCode = 404;
  readonly errorCode = 'NOT-FOUND-001';
  readonly loggingLevel = 'info' as const;
  readonly isRetryable = false;
}

export class ConflictError extends AppError {
  readonly statusCode = 409;
  readonly errorCode = 'CONFLICT-001';
  readonly loggingLevel = 'warn' as const;
  readonly isRetryable = false;
}

export class RateLimitError extends AppError {
  readonly statusCode = 429;
  readonly errorCode = 'RATE-001';
  readonly loggingLevel = 'warn' as const;
  readonly isRetryable = true;  // após TTL
}

export class ExternalServiceError extends AppError {
  readonly statusCode = 502;
  readonly errorCode = 'EXT-001';
  readonly loggingLevel = 'error' as const;
  readonly isRetryable = true;
}

export class ExternalServiceTimeoutError extends AppError {
  readonly statusCode = 504;
  readonly errorCode = 'EXT-002';
  readonly loggingLevel = 'error' as const;
  readonly isRetryable = true;
}

export class InternalError extends AppError {
  readonly statusCode = 500;
  readonly errorCode = 'INT-001';
  readonly loggingLevel = 'error' as const;
  readonly isRetryable = false;
}
```

### 3.2 Global Exception Filter (NestJS)

```typescript
// apps/api/src/common/filters/global-exception.filter.ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  constructor(private readonly logger: Logger) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp()
    const response = ctx.getResponse<Response>()
    const request = ctx.getRequest<Request>()
    const correlationId = request.headers['x-correlation-id'] as string || randomUUID()

    if (exception instanceof AppError) {
      this.logger[exception.loggingLevel]({
        correlation_id: correlationId,
        error_code: exception.errorCode,
        message: exception.message,
        path: request.url,
        user_id: hashUserId(request.user?.id),  // nunca o ID real
        metadata: exception.metadata
      })

      if (exception.loggingLevel === 'error') {
        Sentry.captureException(exception, { extra: { correlationId } })
      }

      response.status(exception.statusCode).json({
        error: {
          code: exception.errorCode,
          message: exception.message,
          user_message: exception.userMessage,
          correlation_id: correlationId,
          timestamp: new Date().toISOString(),
          metadata: exception.metadata
        }
      })
      return
    }

    // Erro não mapeado — InternalError
    this.logger.error({
      correlation_id: correlationId,
      error_code: 'INT-001',
      message: 'Unhandled exception',
      stack: exception instanceof Error ? exception.stack : String(exception),
      path: request.url
    })
    Sentry.captureException(exception, { extra: { correlationId } })

    response.status(500).json({
      error: {
        code: 'INT-001',
        message: 'Unhandled exception',
        user_message: 'Ocorreu um erro inesperado. Nosso time foi notificado automaticamente.',
        correlation_id: correlationId,
        timestamp: new Date().toISOString()
      }
    })
  }
}
```

---

## 4. Error Boundaries (Frontend)

### 4.1 Níveis de Boundary

| Nível | Componente | Tipo de erro capturado | Fallback UI | Observabilidade |
|---|---|---|---|---|
| **Global** | `RootErrorBoundary` (wraps toda a app) | Erros não capturados em qualquer rota | Página de erro global com botão "Recarregar" | Sentry.captureException |
| **Route-level** | `RouteErrorBoundary` (por rota/feature) | Erros de render de uma rota inteira | Empty state de erro da rota com link para Dashboard | Sentry.captureException com rota |
| **Component-level** | `ComponentErrorBoundary` (componentes críticos: KpiCard, NegociationChat, AiAnalyst) | Erros de render de um componente | Skeleton ou placeholder neutro | Sentry.captureException com componente |

### 4.2 Implementação Global Error Boundary

```tsx
// apps/web/src/components/errors/RootErrorBoundary.tsx
import * as Sentry from '@sentry/react'

export const RootErrorBoundary = Sentry.withErrorBoundary(
  ({ children }) => <>{children}</>,
  {
    fallback: ({ error, resetError }) => (
      <FullPageLayout>
        <div className="flex flex-col items-center gap-6 p-8">
          <h1 className="text-2xl font-semibold text-foreground">
            Algo inesperado aconteceu
          </h1>
          <p className="text-muted-foreground text-center max-w-md">
            Nosso time foi notificado automaticamente. Tente recarregar a página.
          </p>
          <button onClick={resetError} className="btn-primary">
            Recarregar
          </button>
        </div>
      </FullPageLayout>
    ),
    onError: (error, info) => {
      Sentry.captureException(error, {
        extra: { componentStack: info.componentStack }
      })
    }
  }
)
```

### 4.3 Tratamento de Erros HTTP no Frontend

```typescript
// apps/web/src/services/api.ts — interceptor de resposta
async function handleApiError(response: Response): Promise<never> {
  const body = await response.json().catch(() => ({}))
  const errorCode = body?.error?.code ?? 'INT-001'
  const correlationId = response.headers.get('x-correlation-id') ?? 'unknown'

  // Log estruturado no frontend (enviado ao Sentry via breadcrumb)
  Sentry.addBreadcrumb({
    category: 'api.error',
    message: `API error: ${errorCode}`,
    level: response.status >= 500 ? 'error' : 'warning',
    data: { errorCode, correlationId, status: response.status, path: response.url }
  })

  throw new ApiError(errorCode, body?.error?.user_message, correlationId, response.status)
}

// Interceptor global no fetch wrapper
export async function apiFetch<T>(url: string, options?: RequestInit): Promise<T> {
  const response = await fetch(`${import.meta.env.VITE_API_URL}${url}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'X-Correlation-ID': crypto.randomUUID(),
      Authorization: `Bearer ${getAccessToken()}`,
      ...options?.headers
    }
  })

  if (!response.ok) {
    await handleApiError(response)
  }

  return response.json()
}
```

---

## 5. Mapeamento API Error → UI

| Código | Componente de UI | Mensagem exibida | Ação disponível |
|---|---|---|---|
| `AUTH-001` | Banner de erro no formulário de login | "E-mail ou senha incorretos. Verifique seus dados ou recupere sua senha." | Link "Recuperar senha" |
| `AUTH-003` | Banner de erro com contador | "Muitas tentativas. Aguarde {X} minutos." | Contador regressivo visível |
| `AUTH-008` | Redirect automático para /login | — (redirect silencioso) | Nenhuma — redirect |
| `VAL-001` | Mensagem inline abaixo do campo | Mensagem específica por campo (UX Writing D08) | Foco no campo com erro |
| `PERM-001` | Toast de erro | "Você precisa completar o KYC para realizar esta ação." | Botão "Completar KYC" |
| `RATE-001` (3 propostas) | Toast de erro | "Você atingiu o limite de 3 propostas simultâneas." | Nenhuma ação — informativo |
| `NOT-FOUND-001` | Página 404 ou inline empty state | "Este conteúdo não está mais disponível." | Botão "Voltar" |
| `CONFLICT-001` | Toast de aviso | Mensagem contextual por entidade | Botão de ação contextual |
| `EXT-001` | Toast de erro + retry automático | "Estamos com uma instabilidade temporária. Tente novamente em instantes." | Botão "Tentar novamente" |
| `EXT-002` | Toast de erro + retry | "A operação demorou mais que o esperado. Tente novamente." | Botão "Tentar novamente" |
| `DB-001` | Toast de erro | "Estamos com uma instabilidade interna. Nosso time foi notificado." | Botão "Tentar novamente mais tarde" |
| `INT-001` | Toast de erro crítico | "Ocorreu um erro inesperado. Nosso time foi notificado automaticamente." | Botão "Contatar suporte" com `correlation_id` |

> ⚙️ **Correlation ID ao usuário:** Para erros `INT-001`, exibir discretamente o `correlation_id` na UI para facilitar o suporte: "Código do erro: {correlation_id}". Isso permite que o suporte encontre o log exato.

---

## 6. Retry e Recovery

### 6.1 Classificação: Retryable vs Não-Retryable

| Categoria | Retryable | Estratégia |
|---|---|---|
| `AUTH-001`, `AUTH-010` | ❌ Não | Usuário corrige credenciais |
| `AUTH-003` (brute-force) | ✅ Após TTL | Aguardar TTL Redis; frontend exibe contador |
| `AUTH-008` (token expirado) | ✅ Automático | Frontend tenta `/auth/refresh` automaticamente 1x |
| `VAL-001` | ❌ Não | Usuário corrige input |
| `PERM-001` | ❌ Não | Usuário completa KYC ou contacta suporte |
| `NOT-FOUND-001` | ❌ Não | Redirect para listagem |
| `CONFLICT-001` | ❌ Não | Usuário vê estado atual e age |
| `RATE-001` | ✅ Após janela | Aguardar janela de rate limit |
| `EXT-001` | ✅ 3 tentativas | Exponential backoff: 1s → 2s → 4s |
| `EXT-002` | ✅ 2 tentativas | Backoff: 2s → 4s |
| `DB-001` | ✅ 2 tentativas | Prisma retry nativo + 1 retry manual |
| `INT-001` | ❌ Não | Suporte com correlation_id |

### 6.2 Estratégia de Retry (Backend — serviços externos)

```typescript
// apps/api/src/common/utils/retry.util.ts
async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxAttempts: number;
    backoffBase: number;   // ms
    jitter: boolean;       // reduz thundering herd
    onRetry?: (attempt: number, error: Error) => void;
  }
): Promise<T> {
  let lastError: Error

  for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error as Error
      if (attempt < options.maxAttempts) {
        const baseDelay = options.backoffBase * Math.pow(2, attempt - 1)  // 1s, 2s, 4s
        const jitter = options.jitter ? Math.random() * baseDelay * 0.1 : 0
        await sleep(baseDelay + jitter)
        options.onRetry?.(attempt, lastError)
      }
    }
  }

  throw lastError!
}

// Uso para serviços externos (ZapSign, idwall, etc.)
const result = await withRetry(
  () => zapSignService.sendDocument(formalizationId),
  { maxAttempts: 3, backoffBase: 1000, jitter: true,
    onRetry: (attempt, err) => logger.warn(`ZapSign retry ${attempt}: ${err.message}`) }
)
```

### 6.3 Retry no Frontend (TanStack Query)

```typescript
// Configuração global do TanStack Query
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        // Erros 4xx: nunca retry (exceção: 429 com retryAfter)
        if (error instanceof ApiError && error.status >= 400 && error.status < 500) {
          if (error.status === 429) return failureCount < 2  // retry 429
          return false
        }
        // Erros 5xx: retry até 3x
        return failureCount < 3
      },
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000)  // exp backoff, max 30s
    },
    mutations: {
      retry: 0  // mutations nunca fazem retry automático
    }
  }
})
```

---

## 7. Logging de Erros

### 7.1 Campos Obrigatórios em Todo Log de Erro

```typescript
interface ErrorLog {
  level: 'info' | 'warn' | 'error';
  correlation_id: string;      // UUID — rastreio entre backend, frontend, Sentry
  error_code: string;          // Ex: "EXT-001"
  category: string;            // Ex: "ExternalServiceError"
  message: string;             // Mensagem técnica
  stack_trace?: string;        // Apenas em logs internos (nível error) — NUNCA no response
  user_id?: string;            // Hash do user_id — NUNCA o ID real ou CPF
  request_path: string;        // Ex: "/api/v1/proposals"
  request_method: string;      // Ex: "POST"
  environment: string;         // Ex: "production"
  timestamp: string;           // ISO 8601 UTC
  metadata?: Record<string, unknown>;  // Contexto adicional sem PII
}
```

### 7.2 Regras de Exclusão de Dados Sensíveis

> 🔴 **Regra inegociável:** Os seguintes dados **nunca aparecem em logs, respostas de erro ou breadcrumbs do Sentry:**

| Dado proibido | Razão |
|---|---|
| Senhas (password, plaintext) | Segurança básica |
| JWT tokens (access ou refresh) | Permite impersonação |
| `SUPABASE_SERVICE_ROLE_KEY` | Bypass de RLS |
| CPF, RG, número de identidade | LGPD — dado sensível |
| Dados bancários (conta, agência) | PCI — dado financeiro sensível |
| E-mail em logs de 5xx | Minimização de PII |
| Stack trace no response HTTP | Exposição de arquitetura interna |

```typescript
// Configuração do scrubFields no Pino
const logger = pino({
  redact: {
    paths: [
      'password', 'senha', 'token', 'access_token', 'refresh_token',
      'authorization', 'cpf', 'rg', 'document_number',
      'bank_account', 'bank_agency',
      'SUPABASE_SERVICE_ROLE_KEY'
    ],
    censor: '[REDACTED]'
  }
})
```

### 7.3 Anti-exemplos de Logging

> ❌ **Anti-exemplo 1 — Erro engolido:**
> ```typescript
> try {
>   await zapSignService.sendDocument(id)
> } catch (e) {
>   // silêncio
> }
> ```
> ✅ **Correto:** Sempre logar + propagar ou registrar em DLQ.

> ❌ **Anti-exemplo 2 — Stack trace no response:**
> ```json
> { "error": "Error: Cannot read property 'id' of undefined\n  at ProposalService.create ..." }
> ```
> ✅ **Correto:** Stack trace apenas em logs internos. Response apenas com `code`, `user_message` e `correlation_id`.

> ❌ **Anti-exemplo 3 — Código de erro inconsistente:**
> ```json
> // Endpoint 1: { "error": "INVALID_TOKEN" }
> // Endpoint 2: { "message": "token invalid", "statusCode": 401 }
> ```
> ✅ **Correto:** Schema único em todos os endpoints: `{ "error": { "code": "AUTH-008", ... } }`.

> ❌ **Anti-exemplo 4 — Mensagem genérica sem código:**
> ```json
> { "message": "Algo deu errado." }
> ```
> ✅ **Correto:** Sempre com código + user_message + correlation_id para rastreabilidade.

---

## 8. Observabilidade e Alertas

### 8.1 Integração com Sentry

| Nível de erro | Capturado pelo Sentry | Alerta configurado |
|---|---|---|
| `info` | ❌ Não | Não |
| `warn` | ❌ Não (apenas breadcrumb) | Não |
| `error` (5xx, exceções não mapeadas) | ✅ Sim | Sim — novo tipo de erro ou spike |

**Configuração obrigatória:**
- `beforeSend`: scrub de PII antes de enviar ao Sentry (ver seção 7.2)
- `tracesSampleRate`: `0.1` (10% em produção) para performance tracing
- `integrations`: `HttpInstrumentation`, `PrismaInstrumentation`

### 8.2 Alertas por Categoria

| Trigger | Ação |
|---|---|
| Novo tipo de erro nunca visto antes | Sentry alert → Slack #incidents |
| Taxa de `INT-001` > 5% em 5min | PagerDuty P0 |
| Taxa de `EXT-001` (ZapSign) > 10% em 1h | Slack #ops |
| Taxa de `DB-001` > 1% em 5min | PagerDuty P1 |
| Taxa de `AUTH-003` (brute-force) > 50/min | Slack #security |

---

## 9. Testes de Error Handling

### 9.1 Casos de Teste Obrigatórios

| Cenário | Tipo de teste | Expectativa |
|---|---|---|
| Login com credenciais erradas | E2E + unit | `AUTH-001`, status 401, `user_message` correto |
| 6 tentativas de login seguidas | Integration | 5ª retorna 401, 6ª retorna `AUTH-003` 429 |
| Token expirado em requisição autenticada | Integration | `AUTH-008` 401; frontend tenta refresh |
| Proposta com `proposed_value` negativo | Unit | `VAL-001` 422 com `metadata.fields` |
| 4ª proposta simultânea (limite: 3) | Integration | `RATE-001` 429 |
| ZapSign retorna 503 | Unit (mock) | `EXT-001` 502, retry 3x documentado |
| Erro não mapeado (throw new Error) | Unit | `INT-001` 500, correlation_id presente, stack NOT no response |
| Acesso a recurso de outro Cessionário | Integration | `PERM-001` 403 (via RLS) |

### 9.2 Validação do Schema de Erro

```typescript
// Teste de schema — aplicado a todo endpoint
function assertErrorSchema(response: ErrorResponse): void {
  expect(response.error.code).toMatch(/^[A-Z]+-\d{3}$/)  // formato CODE-NNN
  expect(response.error.user_message).toBeDefined()
  expect(response.error.correlation_id).toMatch(UUID_REGEX)
  expect(response.error.timestamp).toMatch(ISO8601_REGEX)
  expect(response.error).not.toHaveProperty('stack')      // stack NUNCA no response
  expect(response.error).not.toHaveProperty('stackTrace')
}
```

---

## 10. Glossário de Códigos de Erro

| Prefixo | Domínio | Exemplos |
|---|---|---|
| `AUTH-` | Autenticação e sessão | AUTH-001 (credenciais), AUTH-003 (brute-force), AUTH-008 (token) |
| `USR-` | Usuário (conta, perfil) | USR-001 (e-mail duplicado), USR-002 (conta bloqueada) |
| `CES-` | Cessionário (KYC, dados) | CES-001 (KYC não aprovado), CES-002 (status inválido) |
| `OPR-` | Oportunidades | OPR-001 (não encontrada), OPR-002 (fora do ar) |
| `PRP-` | Propostas | PRP-001 (limite simultâneas), PRP-002 (expirada), PRP-003 (estado inválido) |
| `NEG-` | Negociações | NEG-001 (prazo Escrow), NEG-002 (contraproposta inválida) |
| `FRM-` | Formalização | FRM-001 (ZapSign falhou), FRM-002 (documento pendente) |
| `FIN-` | Financeiro | FIN-001 (depósito não confirmado), FIN-002 (comissão inválida) |
| `NOT-` | Notificações | NOT-001 (e-mail falhou), NOT-002 (push falhou) |
| `AI-` | Analista de Oportunidades | AI-001 (prompt injection), AI-002 (LLM timeout), AI-003 (rate limit IA) |
| `DB-` | Banco de dados | DB-001 (erro de query), DB-002 (timeout de conexão) |
| `CACHE-` | Redis/cache | CACHE-001 (conexão Redis), CACHE-002 (miss crítico) |
| `STG-` | Storage (Supabase Storage) | STG-001 (upload falhou), STG-002 (signed URL expirada) |
| `VAL-` | Validação de input | VAL-001 (campo inválido), VAL-002 (schema inválido) |
| `NOT-FOUND-` | Recurso não encontrado | NOT-FOUND-001 (genérico) |
| `CONFLICT-` | Conflito de estado | CONFLICT-001 (transição inválida de estado) |
| `PERM-` | Permissão negada | PERM-001 (ação não permitida) |
| `RATE-` | Rate limit de negócio | RATE-001 (limite de propostas), RATE-002 (limite de KYC) |
| `EXT-` | Serviço externo | EXT-001 (indisponível), EXT-002 (timeout) |
| `INT-` | Erro interno | INT-001 (não mapeado) |

---

## 11. Backlog de Pendências

| Item | Marcador | Seção | Justificativa | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Correlação de erros frontend-backend | `[DECISÃO AUTÔNOMA]` | 5 — Error Boundaries | Frontend gera o `X-Correlation-ID` e o envia no header de cada request. Backend repete o ID no response. Descartado: backend gera (não há request ainda quando o erro ocorre no cliente). | Alto — rastreio fim-a-fim | Backend Lead | Implementado |
| Nível de detalhe do `metadata` em erros de validação | `[DECISÃO AUTÔNOMA]` | 2 — Schema | `metadata.fields` com lista de campos com erro. Descartado: apenas mensagem genérica (dificulta UX inline). Critério: UX Writing D08 exige mensagem por campo. | Médio | Frontend Lead | Implementado |
| Exibição do correlation_id ao usuário | `[DECISÃO AUTÔNOMA]` | 5 — Mapeamento | Exibir correlation_id apenas para INT-001. Descartado: exibir em todos os erros (ruído para o usuário). Critério: apenas erros sem recovery do usuário justificam expor o ID para suporte. | Baixo | Frontend Lead | Implementado |
