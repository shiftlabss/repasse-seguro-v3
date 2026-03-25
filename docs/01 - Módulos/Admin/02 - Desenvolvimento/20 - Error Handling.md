# 20 - Error Handling

## Repasse Seguro — Módulo Admin

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend, Frontend e Arquitetura |
| **Escopo** | Contrato unificado de tratamento de erros entre API, UI, logs e recovery |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Status** | Aprovado |
| **Dependências** | D01 RN · D08 UX Writing · D14 Especificações Técnicas · D16 Documentação de API |

---

> 📌 **TL;DR**
>
> - **Contrato único** entre backend (NestJS), frontend (React 19/SPA) e logs (Pino/Sentry): toda camada fala a mesma língua de erro.
> - **12 categorias de erro** com código interno, status HTTP, mensagem técnica, mensagem de UI e recovery definidos.
> - **RFC 7807** como padrão de resposta de erro na API: `type`, `title`, `status`, `detail`, `instance`, `correlation_id`.
> - **Stack trace nunca exposta** ao usuário ou em respostas de API; apenas em logs internos.
> - **Retry automático** apenas para erros de rede e serviços externos (backoff exponencial: 1s, 3s, 9s — máximo 3 tentativas).
> - **3 níveis de Error Boundary** no frontend: global (app), route-level e component-level.
> - **Correlation ID** obrigatório em todo log e toda resposta de erro para rastreabilidade end-to-end.

---

## 1. Taxonomia de Erros

Toda categoria de erro no Repasse Seguro obedece ao seguinte contrato:

| Código | Categoria | Status HTTP | Mensagem Técnica | Mensagem de UI | Recovery | Log Level |
|---|---|---|---|---|---|---|
| `VAL_001` | Validação de input | 422 | `Validation failed: {field} — {rule}` | "Verifique os dados e tente novamente." | Nenhum retry — corrigir input | `warn` |
| `AUTH_001` | Token ausente | 401 | `Authorization header missing` | "Sua sessão expirou. Faça login novamente." | Redirecionar para login | `warn` |
| `AUTH_002` | Token inválido ou expirado | 401 | `JWT verification failed: {reason}` | "Sua sessão expirou. Faça login novamente." | Tentar refresh; se falhar, redirecionar para login | `warn` |
| `AUTH_003` | Acesso negado (RBAC) | 403 | `Role {role} not authorized for {resource}` | "Você não tem permissão para realizar esta ação." | Nenhum retry | `warn` |
| `AUTH_004` | Conta bloqueada | 423 | `Account locked until {locked_until}` | "Conta bloqueada. Tente novamente em {X} minutos." | Aguardar expiração | `warn` |
| `RES_001` | Recurso não encontrado | 404 | `{entity} with id {id} not found` | "Este registro não foi encontrado." | Nenhum retry | `info` |
| `CON_001` | Conflito de versão (Optimistic Lock) | 409 | `Version mismatch: expected {expected}, got {actual}` | "Este registro foi alterado por outra pessoa. Recarregue e tente novamente." | Recarregar dados; retry manual | `warn` |
| `CON_002` | Conflito de unicidade | 409 | `Unique constraint violation: {field}` | "Este {campo} já está cadastrado." | Nenhum retry — corrigir input | `warn` |
| `BIZ_001` | Regra de negócio violada | 422 | `Business rule violation: {rule_code}` | Mensagem específica da regra (ver UX Writing) | Nenhum retry — corrigir estado | `warn` |
| `BIZ_002` | Transição de status inválida | 422 | `Invalid status transition: {from} → {to}` | "Esta transição não é permitida no estado atual do caso." | Nenhum retry | `warn` |
| `EXT_001` | Serviço externo indisponível | 502 | `External service {service} returned {status}` | "Houve um problema com um serviço externo. Tentando novamente…" | Retry automático (3×, backoff exponencial) | `error` |
| `INT_001` | Erro interno não esperado | 500 | `Unhandled exception: {error_class}` | "Algo inesperado aconteceu. Se o problema persistir, contate o suporte." | Retry manual 1× após 30s | `error` |

> ⚙️ **Regra inegociável:** Todo erro retornado ao cliente DEVE conter `correlation_id`. Sem `correlation_id`, o erro não é rastreável. Logs sem `correlation_id` são rejeitados pelo pipeline de observabilidade.

---

## 2. Schema de Erro (Backend)

### 2.1 Schema Padrão (RFC 7807 estendido)

```json
{
  "type": "https://api.repasseseguro.com.br/errors/{error_code}",
  "title": "string — título legível da categoria",
  "status": 422,
  "detail": "string — descrição técnica detalhada (sem stack trace)",
  "instance": "/v1/cases/uuid",
  "correlation_id": "uuid-v4",
  "timestamp": "2026-03-22T10:00:00.000Z",
  "errors": [
    {
      "field": "cpf",
      "message": "CPF inválido"
    }
  ]
}
```

**Campos obrigatórios em toda resposta de erro:**

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `type` | URL string | Sim | URI de referência do erro |
| `title` | string | Sim | Título da categoria |
| `status` | number | Sim | HTTP status code |
| `detail` | string | Sim | Descrição técnica sem dados internos |
| `instance` | string | Sim | Path da requisição que gerou o erro |
| `correlation_id` | UUID v4 | Sim | ID de rastreamento end-to-end |
| `timestamp` | ISO 8601 | Sim | Data/hora UTC do erro |
| `errors` | array | Não | Detalhes por campo (apenas em VAL_001) |

### 2.2 Exemplos por Categoria

**VAL_001 — Validação:**
```json
{
  "type": "https://api.repasseseguro.com.br/errors/VAL_001",
  "title": "Validation Failed",
  "status": 422,
  "detail": "One or more fields failed validation",
  "instance": "/v1/cedentes",
  "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2026-03-22T10:00:00.000Z",
  "errors": [
    { "field": "cpf", "message": "CPF inválido" },
    { "field": "email", "message": "E-mail inválido" }
  ]
}
```

**AUTH_004 — Conta bloqueada:**
```json
{
  "type": "https://api.repasseseguro.com.br/errors/AUTH_004",
  "title": "Account Locked",
  "status": 423,
  "detail": "Account locked after 5 failed login attempts",
  "instance": "/v1/auth/login",
  "correlation_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "timestamp": "2026-03-22T10:05:00.000Z",
  "locked_until": "2026-03-22T10:35:00.000Z"
}
```

**CON_001 — Conflito de versão:**
```json
{
  "type": "https://api.repasseseguro.com.br/errors/CON_001",
  "title": "Optimistic Lock Conflict",
  "status": 409,
  "detail": "Version mismatch: expected 3, got 2",
  "instance": "/v1/cases/uuid/status",
  "correlation_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "timestamp": "2026-03-22T10:10:00.000Z"
}
```

**EXT_001 — Serviço externo:**
```json
{
  "type": "https://api.repasseseguro.com.br/errors/EXT_001",
  "title": "External Service Error",
  "status": 502,
  "detail": "ZapSign API returned 503 Service Unavailable",
  "instance": "/v1/cases/uuid/zapsign",
  "correlation_id": "d4e5f6a7-b8c9-0123-defg-234567890123",
  "timestamp": "2026-03-22T10:15:00.000Z"
}
```

---

## 3. Error Classes (Backend)

### 3.1 Hierarquia de Classes NestJS

```typescript
// base/errors/base.error.ts
export class BaseAppError extends Error {
  constructor(
    public readonly errorCode: string,
    public readonly statusCode: number,
    public readonly detail: string,
    public readonly logLevel: 'info' | 'warn' | 'error',
    public readonly isRetryable: boolean,
    public readonly metadata?: Record<string, unknown>,
  ) {
    super(detail);
    this.name = this.constructor.name;
  }
}

// auth.errors.ts
export class TokenExpiredError extends BaseAppError {
  constructor() {
    super('AUTH_002', 401, 'JWT verification failed: token expired', 'warn', false);
  }
}

export class AccountLockedError extends BaseAppError {
  constructor(lockedUntil: Date) {
    super('AUTH_004', 423, `Account locked until ${lockedUntil.toISOString()}`, 'warn', false, { lockedUntil });
  }
}

export class ForbiddenError extends BaseAppError {
  constructor(role: string, resource: string) {
    super('AUTH_003', 403, `Role ${role} not authorized for ${resource}`, 'warn', false);
  }
}

// validation.errors.ts
export class ValidationAppError extends BaseAppError {
  constructor(errors: { field: string; message: string }[]) {
    super('VAL_001', 422, 'One or more fields failed validation', 'warn', false, { errors });
  }
}

// resource.errors.ts
export class NotFoundError extends BaseAppError {
  constructor(entity: string, id: string) {
    super('RES_001', 404, `${entity} with id ${id} not found`, 'info', false, { entity, id });
  }
}

// conflict.errors.ts
export class OptimisticLockError extends BaseAppError {
  constructor(expected: number, got: number) {
    super('CON_001', 409, `Version mismatch: expected ${expected}, got ${got}`, 'warn', false, { expected, got });
  }
}

// business.errors.ts
export class BusinessRuleError extends BaseAppError {
  constructor(ruleCode: string, description: string) {
    super('BIZ_001', 422, `Business rule violation: ${ruleCode} — ${description}`, 'warn', false, { ruleCode });
  }
}

export class InvalidStatusTransitionError extends BaseAppError {
  constructor(from: string, to: string) {
    super('BIZ_002', 422, `Invalid status transition: ${from} → ${to}`, 'warn', false, { from, to });
  }
}

// external.errors.ts
export class ExternalServiceError extends BaseAppError {
  constructor(service: string, upstreamStatus: number) {
    super('EXT_001', 502, `External service ${service} returned ${upstreamStatus}`, 'error', true, { service, upstreamStatus });
  }
}
```

### 3.2 Global Exception Filter

```typescript
// filters/global-exception.filter.ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const correlationId = request.headers['x-correlation-id'] as string ?? randomUUID();

    if (exception instanceof BaseAppError) {
      // Erro esperado — log no nível correto
      logger[exception.logLevel]({
        correlation_id: correlationId,
        error_code: exception.errorCode,
        detail: exception.detail,
        path: request.path,
        metadata: exception.metadata,
      });

      return response.status(exception.statusCode).json({
        type: `https://api.repasseseguro.com.br/errors/${exception.errorCode}`,
        title: ERROR_TITLES[exception.errorCode],
        status: exception.statusCode,
        detail: exception.detail,
        instance: request.path,
        correlation_id: correlationId,
        timestamp: new Date().toISOString(),
        ...(exception.metadata?.errors && { errors: exception.metadata.errors }),
        ...(exception.metadata?.lockedUntil && { locked_until: exception.metadata.lockedUntil }),
      });
    }

    // Erro não esperado (INT_001)
    // NUNCA expor exception.message ou stack ao cliente
    logger.error({
      correlation_id: correlationId,
      error_code: 'INT_001',
      error_class: exception instanceof Error ? exception.constructor.name : 'Unknown',
      stack: exception instanceof Error ? exception.stack : undefined, // apenas no log interno
      path: request.path,
    });

    return response.status(500).json({
      type: 'https://api.repasseseguro.com.br/errors/INT_001',
      title: 'Internal Server Error',
      status: 500,
      detail: 'An unexpected error occurred',
      instance: request.path,
      correlation_id: correlationId,
      timestamp: new Date().toISOString(),
    });
  }
}
```

---

## 4. Error Boundaries (Frontend)

### 4.1 Três Níveis de Boundary

| Nível | Escopo | Erros capturados | Fallback UI | Retry |
|---|---|---|---|---|
| **Global** | App inteiro | Erros não capturados por boundaries inferiores; crashes críticos | Tela de erro global com botão "Recarregar" | Recarregar página |
| **Route-level** | Página/módulo (ex: Pipeline, Financeiro) | Erros de carregamento de rota, falha de fetch crítico | Área de erro inline com botão "Tentar novamente" | Retry da query (TanStack Query) |
| **Component-level** | Widget isolado (ex: EscrowGauge, KpiCard) | Erros de render de componente secundário | Placeholder com ícone de alerta + tooltip | Polling automático ou refresh manual |

### 4.2 Implementação — Global Boundary

```typescript
// components/error-boundary/GlobalErrorBoundary.tsx
export class GlobalErrorBoundary extends React.Component<
  { children: ReactNode },
  { hasError: boolean; correlationId?: string }
> {
  state = { hasError: false, correlationId: undefined };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    const correlationId = randomUUID();
    Sentry.captureException(error, {
      extra: { componentStack: info.componentStack, correlationId },
    });
    this.setState({ correlationId });
  }

  render() {
    if (this.state.hasError) {
      return (
        <ErrorLayout
          title="Algo inesperado aconteceu"
          description="Se o problema persistir, contate o suporte."
          correlationId={this.state.correlationId}
          action={{ label: 'Recarregar', onClick: () => window.location.reload() }}
        />
      );
    }
    return this.props.children;
  }
}
```

### 4.3 Implementação — Route-level (TanStack Router)

```typescript
// router.tsx — erro de carregamento de rota
const routeWithError = createRoute({
  errorComponent: ({ error }) => {
    const correlationId = (error as ApiError)?.correlation_id;
    return (
      <RouteErrorFallback
        error={error}
        correlationId={correlationId}
        onRetry={() => router.invalidate()}
      />
    );
  },
});
```

### 4.4 Tratamento de Erros HTTP (TanStack Query)

```typescript
// hooks/useApiError.ts
export function useApiError(error: unknown): UIErrorState {
  if (!error) return null;

  const apiError = error as ApiError;
  const errorCode = apiError?.error_code ?? 'INT_001';

  return {
    code: errorCode,
    userMessage: ERROR_MESSAGES_UI[errorCode] ?? 'Algo inesperado aconteceu.',
    correlationId: apiError?.correlation_id,
    isRetryable: RETRYABLE_ERRORS.includes(errorCode),
    action: ERROR_ACTIONS[errorCode],
  };
}
```

---

## 5. Mapeamento API Error → UI

| Código de Erro | Status | Componente de UI | Mensagem Exibida | Ação Disponível |
|---|---|---|---|---|
| `VAL_001` | 422 | Toast vermelho + highlight nos campos inválidos | "Verifique os campos destacados." | Corrigir e submeter novamente |
| `AUTH_001` / `AUTH_002` | 401 | Modal de sessão expirada (bloqueante) | "Sua sessão expirou. Faça login novamente." | Botão "Ir para o login" |
| `AUTH_003` | 403 | Toast vermelho | "Você não tem permissão para realizar esta ação." | Fechar |
| `AUTH_004` | 423 | Alert box com timer regressivo | "Conta bloqueada. Disponível em {X} minutos." | Aguardar (sem ação) |
| `RES_001` | 404 | Empty state inline | "Este registro não foi encontrado." | Botão "Voltar" |
| `CON_001` | 409 | Toast laranja | "Registro alterado por outra pessoa. Recarregue os dados." | Botão "Recarregar" |
| `BIZ_001` / `BIZ_002` | 422 | Toast laranja com detalhe da regra | Mensagem específica da regra (ver UX Writing D08) | Corrigir estado ou cancelar |
| `EXT_001` | 502 | Toast amarelo com spinner de retry | "Serviço externo temporariamente indisponível. Tentando novamente…" | Retry automático — botão "Cancelar" |
| `INT_001` | 500 | Toast vermelho + correlation_id exibido discretamente | "Algo inesperado aconteceu. Se persistir, contate o suporte." | Botão "Tentar novamente" |

> ⚙️ **Regra:** `correlation_id` DEVE ser exibido para o usuário em erros `INT_001` — campo pequeno e discreto (ex: "Código de suporte: a1b2c3…"). Isso permite ao suporte rastrear o erro nos logs sem expor detalhes internos.

---

## 6. Retry e Recovery

### 6.1 Classificação por Retryabilidade

| Categoria | Retryable | Estratégia | Detalhe |
|---|---|---|---|
| `VAL_001` | Não | Corrigir input | Usuário deve corrigir os dados — nenhum retry automático |
| `AUTH_001` | Não | Redirecionar para login | Token ausente = sessão nova necessária |
| `AUTH_002` | Parcial | Tentar refresh uma vez | Se refresh falhar → redirecionar para login |
| `AUTH_003` | Não | Nenhuma ação | Role insuficiente — não muda com retry |
| `AUTH_004` | Não | Aguardar expiração | Bloqueio por tentativas — aguardar `locked_until` |
| `RES_001` | Não | Nenhuma ação | Recurso inexistente — retry não resolve |
| `CON_001` | Sim (manual) | Recarregar + retry manual | Usuário recarrega dados e tenta novamente |
| `CON_002` | Não | Corrigir input | Unicidade violada — alterar o dado conflitante |
| `BIZ_001` / `BIZ_002` | Não | Corrigir estado | Regra de negócio — mudar o contexto, não o timing |
| `EXT_001` | Sim (automático) | Backoff exponencial | 3 tentativas: 1s, 3s, 9s; após 3 falhas → `INT_001` |
| `INT_001` | Sim (manual) | Retry manual 1× | Botão "Tentar novamente" disponível por 30s; após timeout → instruir suporte |

### 6.2 Implementação do Retry (TanStack Query)

```typescript
// lib/queryClient.ts
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        const apiError = error as ApiError;
        if (!apiError?.error_code) return failureCount < 1; // erros de rede
        return RETRYABLE_ERRORS.includes(apiError.error_code) && failureCount < 3;
      },
      retryDelay: (attempt) => Math.min(1000 * 3 ** attempt, 9000), // 1s, 3s, 9s
    },
    mutations: {
      retry: 0, // mutations nunca retried automaticamente
    },
  },
});
```

### 6.3 Retry de Serviços Externos (Backend)

```typescript
// utils/retry.util.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  options = { maxAttempts: 3, baseDelay: 1000, jitter: true }
): Promise<T> {
  for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === options.maxAttempts) throw error;
      const delay = options.baseDelay * 3 ** (attempt - 1);
      const jitter = options.jitter ? Math.random() * 200 : 0;
      await sleep(delay + jitter);
    }
  }
  throw new Error('Max retries reached'); // never reached, satisfies TypeScript
}
```

---

## 7. Logging de Erros

### 7.1 Campos Obrigatórios em Todo Log de Erro

```json
{
  "level": "error",
  "timestamp": "2026-03-22T10:00:00.000Z",
  "correlation_id": "uuid-v4",
  "error_code": "EXT_001",
  "category": "external_service",
  "message": "ZapSign API returned 503",
  "stack_trace": "Error: ...\n  at ZapsignService...",
  "user_id_hash": "sha256:abc123...",
  "request_path": "/v1/cases/uuid/zapsign",
  "request_method": "POST",
  "environment": "production",
  "service": "api",
  "version": "1.0.0"
}
```

### 7.2 Regras de Exclusão de Dados Sensíveis

| Dado | Ação |
|---|---|
| `password`, `token`, `secret`, `api_key` | Remover do log; nunca logar |
| `cpf`, `email`, `phone` | Substituir por hash SHA-256 truncado (8 chars) |
| JWT payload | Logar apenas `jti` e `sub`; nunca o token inteiro |
| Stack trace | Apenas em logs internos (Pino); nunca na resposta ao cliente |
| `credit_card`, `pix_key` | Mascarar: `****1234` |

> 🔴 **Regra inegociável:** Nenhum dado sensível em logs. Implementar middleware de sanitização via `pino-serializers` configurado globalmente. Logs com CPF, senha ou token em texto puro devem ser tratados como incidente de segurança.

### 7.3 Configuração do Pino com Sanitização

```typescript
// logger/pino.config.ts
export const pinoConfig = {
  serializers: {
    req: (req) => ({
      id: req.id,
      method: req.method,
      url: req.url,
      // NUNCA serializar headers de autorização
    }),
    err: (err) => ({
      type: err.constructor.name,
      message: err.message,
      // stack apenas em development
      ...(process.env.NODE_ENV !== 'production' && { stack: err.stack }),
    }),
  },
  redact: {
    paths: ['req.headers.authorization', '*.password', '*.token', '*.secret', '*.cpf', '*.email'],
    censor: '[REDACTED]',
  },
};
```

---

## 8. Observabilidade e Alertas

### 8.1 Integração com Sentry

```typescript
// main.ts
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1, // 10% das transações em produção
  beforeSend: (event) => {
    // Remover dados sensíveis antes de enviar ao Sentry
    delete event.request?.headers?.authorization;
    return event;
  },
});
```

- Todos os erros `INT_001` e `EXT_001` enviados ao Sentry automaticamente via `GlobalExceptionFilter`.
- Erros `AUTH_*`, `VAL_*` e `BIZ_*` logados apenas no Pino (não enviam ao Sentry — são erros esperados).

### 8.2 Alertas de Error Rate

| Métrica | Threshold | Ação |
|---|---|---|
| Taxa de `INT_001` > 1% das requisições em 5 min | Alerta crítico | Pager Duty → engenharia on-call |
| Taxa de `EXT_001` > 5% em 5 min (qualquer serviço externo) | Alerta alto | Slack `#alertas-producao` |
| Taxa de `AUTH_002` > 10 req/min por IP | Alerta de segurança | Bloquear IP temporariamente + Slack `#alertas-seguranca` |
| Taxa de `CON_001` > 2% das mutations em 5 min | Alerta médio | Revisão de concorrência |

---

## 9. Testes de Error Handling

### 9.1 Cobertura Obrigatória

Cada categoria de erro deve ter pelo menos 1 teste de unidade e 1 teste de integração cobrindo:

- Resposta HTTP correta (status + corpo RFC 7807)
- Presença de `correlation_id`
- Ausência de stack trace na resposta
- Ausência de dados sensíveis na resposta
- Log gerado com campos obrigatórios

```typescript
// Exemplo de teste de integração — VAL_001
it('POST /cedentes com CPF inválido retorna 422 com schema correto', async () => {
  const response = await request(app.getHttpServer())
    .post('/v1/cedentes')
    .set('Authorization', `Bearer ${validToken}`)
    .send({ cpf: 'invalido', email: 'test@test.com', name: 'Test' });

  expect(response.status).toBe(422);
  expect(response.body).toMatchObject({
    type: expect.stringContaining('VAL_001'),
    status: 422,
    correlation_id: expect.any(String),
    errors: expect.arrayContaining([
      expect.objectContaining({ field: 'cpf' }),
    ]),
  });
  expect(response.body).not.toHaveProperty('stack');
  expect(JSON.stringify(response.body)).not.toContain('password');
});
```

---

## 10. Anti-Patterns

### ❌ Anti-exemplo 1 — Erro engolido sem feedback

```typescript
// ERRADO — erro ignorado silenciosamente
async function updateCase(id: string) {
  try {
    await prisma.case.update({ where: { id }, data: {} });
  } catch (e) {
    // nada aqui — erro desaparece
  }
}

// CORRETO
async function updateCase(id: string) {
  try {
    await prisma.case.update({ where: { id }, data: {} });
  } catch (e) {
    if (e instanceof PrismaClientKnownRequestError && e.code === 'P2034') {
      throw new OptimisticLockError(expected, actual);
    }
    throw e; // propagar para GlobalExceptionFilter
  }
}
```

### ❌ Anti-exemplo 2 — Mensagem genérica sem código

```json
// ERRADO
{ "message": "Algo deu errado" }

// CORRETO (RFC 7807 com código)
{
  "type": "https://api.repasseseguro.com.br/errors/INT_001",
  "title": "Internal Server Error",
  "status": 500,
  "correlation_id": "uuid",
  "timestamp": "..."
}
```

### ❌ Anti-exemplo 3 — Stack trace exposta ao cliente

```json
// ERRADO
{
  "status": 500,
  "message": "Cannot read properties of undefined",
  "stack": "TypeError: Cannot read...\n  at CaseService.updateCase (/app/src/...)"
}

// CORRETO — stack apenas no log interno (Pino)
{
  "type": "https://api.repasseseguro.com.br/errors/INT_001",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "An unexpected error occurred",
  "correlation_id": "uuid"
}
```

### ❌ Anti-exemplo 4 — Codes de erro inconsistentes entre camadas

```typescript
// ERRADO — cada módulo inventando seu próprio padrão
// auth.service: throw new Error('TOKEN_INVALID')
// cases.service: throw new HttpException('not_found', 404)
// cedentes.service: throw { code: 404, msg: 'Not found' }

// CORRETO — hierarquia única de Error Classes
throw new NotFoundError('Cedente', id);       // → código RES_001, status 404
throw new TokenExpiredError();                 // → código AUTH_002, status 401
throw new OptimisticLockError(expected, got); // → código CON_001, status 409
```

---

## 11. Glossário de Códigos de Erro

| Prefixo | Domínio | Faixa |
|---|---|---|
| `VAL_` | Validação de input | VAL_001 – VAL_099 |
| `AUTH_` | Autenticação e autorização | AUTH_001 – AUTH_099 |
| `RES_` | Recursos não encontrados | RES_001 – RES_099 |
| `CON_` | Conflitos (lock, unicidade) | CON_001 – CON_099 |
| `BIZ_` | Regras de negócio | BIZ_001 – BIZ_099 |
| `EXT_` | Serviços externos | EXT_001 – EXT_099 |
| `INT_` | Erros internos não esperados | INT_001 – INT_099 |

---

## 12. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — 12 categorias de erro, RFC 7807, hierarquia de Error Classes, 3 níveis de Error Boundary, retry com backoff exponencial, sanitização de dados sensíveis, integração Sentry, anti-patterns. |

---

## 13. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Nenhuma pendência aberta | — | — | Todas as categorias resolvidas com contexto disponível nos documentos de input | — | — | Fechado |
