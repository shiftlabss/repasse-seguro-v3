# Error Handling — AI-Dani-Admin

## Estratégia de Tratamento de Erros do Módulo de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend |
| Escopo | Estratégia completa de tratamento de erros: classificação, shapes de resposta, filtros globais, logging e comportamento por camada |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D02 (Stacks — NestJS, Pino, Sentry), D14 (Especificações Técnicas), D15 (Arquitetura de Pastas), D16 (Documentação de API) |

---

> **📌 TL;DR**
>
> - **Shape único de erro:** `{ error: { code, message, details? } }` em todas as respostas de erro HTTP.
> - **Códigos no formato** `DA-{MÓDULO}-{NNN}` — 1 prefixo único por módulo.
> - **3 camadas de tratamento:** Controller (ValidationPipe), Service (exceções de negócio), Repository (exceções de banco + Prisma).
> - **HttpExceptionFilter global:** captura e formata todas as exceções não tratadas.
> - **Logging:** Pino com request ID em todo log de erro. PII proibido em logs.
> - **Sentry:** captura exceções de infraestrutura P0 e P1. Langfuse captura erros de LLM.
> - **Erros de usuário (4xx):** nunca logados no Sentry. Apenas registrados em Pino nível `warn`.

---

## 1. Classificação de Erros

| Categoria | Status HTTP | Sentry | Pino Level | Exemplo |
|---|---|---|---|---|
| **Validação de input** | 400 | Não | `warn` | threshold fora de 50-95% |
| **Autenticação** | 401 | Não | `warn` | JWT expirado |
| **Autorização** | 403 | Não | `warn` | Role != ADMIN |
| **Não encontrado** | 404 | Não | `warn` | Interação inexistente |
| **Conflito** | 409 | Não | `warn` | Takeover simultâneo |
| **Regra de negócio** | 422 | Não | `warn` | Autorizar checklist com itens PENDENTE |
| **Rate limit** | 429 | Não | `warn` | Webchat > 30 msg/h |
| **Erro de infraestrutura** | 503 | Sim (P1) | `error` | DB connection timeout |
| **Erro interno inesperado** | 500 | Sim (P0) | `error` | NullPointerException não tratada |
| **Dependência externa** | 502 | Sim (P2) | `error` | OpenAI timeout |

---

## 2. Shape de Resposta de Erro

### 2.1 Formato padrão

```typescript
interface ErrorResponse {
  error: {
    code: string       // DA-{MÓDULO}-{NNN}
    message: string    // Mensagem legível para o usuário/dev
    details?: Record<string, unknown>  // Informações adicionais (opcional)
  }
}
```

### 2.2 Exemplos por tipo

**400 — Validação:**
```json
{
  "error": {
    "code": "DA-CFG-001",
    "message": "O nível de supervisão precisa estar entre 50% e 95%. Valores fora desse intervalo podem comprometer a qualidade do atendimento.",
    "details": {
      "field": "confidenceThreshold",
      "receivedValue": 45,
      "allowedRange": { "min": 50, "max": 95 }
    }
  }
}
```

**401 — Token expirado:**
```json
{
  "error": {
    "code": "DA-AUD-002",
    "message": "Token expirado. Faça login novamente."
  }
}
```

**403 — Role inválida:**
```json
{
  "error": {
    "code": "DA-AUD-001",
    "message": "Acesso negado. Este recurso é restrito ao perfil Admin."
  }
}
```

**404 — Não encontrado:**
```json
{
  "error": {
    "code": "DA-SUP-002",
    "message": "Interação não encontrada.",
    "details": { "interactionId": "uuid-fornecido" }
  }
}
```

**409 — Conflito de takeover:**
```json
{
  "error": {
    "code": "DA-TAK-001",
    "message": "Esta conversa já está em atendimento por outro analista."
  }
}
```

**422 — Regra de negócio:**
```json
{
  "error": {
    "code": "DA-LCH-003",
    "message": "Mínimo de 20 testes adversariais não atingido.",
    "details": {
      "current": 18,
      "required": 20
    }
  }
}
```

**429 — Rate limit:**
```json
{
  "error": {
    "code": "DA-CFG-004",
    "message": "Limite de mensagens atingido. Aguarde alguns minutos antes de enviar mais mensagens.",
    "details": {
      "limit": 30,
      "windowSeconds": 3600,
      "retryAfterSeconds": 1847
    }
  }
}
```

**500 — Erro interno:**
```json
{
  "error": {
    "code": "DA-SYS-500",
    "message": "Erro interno do servidor. Nossa equipe foi notificada."
  }
}
```

**503 — Serviço indisponível (DB/Redis):**
```json
{
  "error": {
    "code": "DA-SYS-503",
    "message": "Serviço temporariamente indisponível. Tente novamente em instantes."
  }
}
```

---

## 3. HttpExceptionFilter Global

```typescript
// src/common/filters/http-exception.filter.ts

import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common'
import * as Sentry from '@sentry/nestjs'

@Catch()
export class GlobalHttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger('ExceptionFilter')

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp()
    const response = ctx.getResponse()
    const request = ctx.getRequest()

    let status = HttpStatus.INTERNAL_SERVER_ERROR
    let errorBody: { error: { code: string; message: string; details?: unknown } }

    if (exception instanceof HttpException) {
      status = exception.getStatus()
      const exceptionResponse = exception.getResponse()

      // Verificar se o exception já tem o shape correto
      if (
        typeof exceptionResponse === 'object' &&
        'error' in exceptionResponse &&
        typeof (exceptionResponse as any).error?.code === 'string'
      ) {
        errorBody = exceptionResponse as typeof errorBody
      } else {
        // Mapear erros NestJS padrão para nosso shape
        errorBody = this.mapNestJsException(exception, status)
      }

      // Erros 4xx: log como warn (não vão para Sentry)
      if (status < 500) {
        this.logger.warn(
          `[${request.method}] ${request.url} → ${status} ${errorBody.error.code}`,
          { requestId: request.headers['x-request-id'] }
        )
      }
    } else {
      // Erro não esperado (500) → Sentry P0
      Sentry.captureException(exception, {
        tags: {
          method: request.method,
          path: request.url,
          requestId: request.headers['x-request-id'],
        },
      })

      this.logger.error(
        `[${request.method}] ${request.url} → 500 INTERNAL_SERVER_ERROR`,
        exception instanceof Error ? exception.stack : String(exception),
        { requestId: request.headers['x-request-id'] }
      )

      errorBody = {
        error: {
          code: 'DA-SYS-500',
          message: 'Erro interno do servidor. Nossa equipe foi notificada.',
        },
      }
    }

    response.status(status).json(errorBody)
  }

  private mapNestJsException(
    exception: HttpException,
    status: number,
  ): { error: { code: string; message: string } } {
    const defaults: Record<number, { code: string; message: string }> = {
      400: { code: 'DA-SYS-400', message: 'Requisição inválida.' },
      401: { code: 'DA-AUD-002', message: 'Não autenticado.' },
      403: { code: 'DA-AUD-001', message: 'Acesso negado.' },
      404: { code: 'DA-SYS-404', message: 'Recurso não encontrado.' },
      409: { code: 'DA-SYS-409', message: 'Conflito de estado.' },
      422: { code: 'DA-SYS-422', message: 'Entidade não processável.' },
      429: { code: 'DA-SYS-429', message: 'Muitas requisições. Tente novamente em instantes.' },
      503: { code: 'DA-SYS-503', message: 'Serviço temporariamente indisponível.' },
    }

    return { error: defaults[status] ?? { code: 'DA-SYS-500', message: 'Erro interno.' } }
  }
}
```

### 3.1 Registro global no AppModule

```typescript
// app.module.ts
import { APP_FILTER } from '@nestjs/core'

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: GlobalHttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

---

## 4. ValidationPipe Global

Erros de validação de DTO são transformados no shape padrão:

```typescript
// main.ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,           // Remove campos não declarados no DTO
    forbidNonWhitelisted: true, // 400 se campos desconhecidos são enviados
    transform: true,           // Transforma tipos (string → number onde aplicável)
    exceptionFactory: (errors) => {
      const details = errors.reduce((acc, error) => {
        acc[error.property] = Object.values(error.constraints ?? {})
        return acc
      }, {} as Record<string, string[]>)

      return new BadRequestException({
        error: {
          code: 'DA-SYS-400',
          message: 'Parâmetros inválidos na requisição.',
          details,
        },
      })
    },
  }),
)
```

**Exemplo de resposta de validação:**
```json
HTTP 400 Bad Request
{
  "error": {
    "code": "DA-SYS-400",
    "message": "Parâmetros inválidos na requisição.",
    "details": {
      "confidenceThreshold": ["confidenceThreshold must not be less than 50"],
      "interactionId": ["interactionId must be a UUID"]
    }
  }
}
```

---

## 5. Tratamento de Erros Prisma

Erros do Prisma são capturados no Repository e relançados como HttpException com código correto:

```typescript
// supervision.repository.ts

import { PrismaClientKnownRequestError } from '@prisma/client/runtime/library'
import { Injectable, NotFoundException, ConflictException, ServiceUnavailableException } from '@nestjs/common'
import * as Sentry from '@sentry/nestjs'

@Injectable()
export class SupervisionRepository {
  async findById(id: string): Promise<Interaction> {
    try {
      const interaction = await this.prisma.interaction.findUnique({
        where: { id, deleted_at: null },
      })

      if (!interaction) {
        throw new NotFoundException({
          error: { code: 'DA-SUP-002', message: 'Interação não encontrada.' }
        })
      }

      return interaction
    } catch (error) {
      if (error instanceof NotFoundException) throw error
      this.handlePrismaError(error, 'DA-SUP')
    }
  }

  private handlePrismaError(error: unknown, prefix: string): never {
    if (error instanceof PrismaClientKnownRequestError) {
      switch (error.code) {
        case 'P2002': // Unique constraint violation
          throw new ConflictException({
            error: { code: `${prefix}-409`, message: 'Conflito de dados únicos.' }
          })
        case 'P2025': // Record not found
          throw new NotFoundException({
            error: { code: `${prefix}-404`, message: 'Registro não encontrado.' }
          })
        case 'P1001': // Can't reach database server
        case 'P1008': // Operations timed out
          Sentry.captureException(error, { tags: { layer: 'repository', prefix } })
          throw new ServiceUnavailableException({
            error: { code: 'DA-SYS-503', message: 'Serviço temporariamente indisponível. Tente novamente em instantes.' }
          })
        default:
          Sentry.captureException(error, { tags: { layer: 'repository', code: error.code } })
          throw new ServiceUnavailableException({
            error: { code: 'DA-SYS-503', message: 'Erro no banco de dados.' }
          })
      }
    }

    // Erro desconhecido
    Sentry.captureException(error)
    throw new ServiceUnavailableException({
      error: { code: 'DA-SYS-503', message: 'Serviço temporariamente indisponível.' }
    })
  }
}
```

---

## 6. Tratamento de Erros em Consumers RabbitMQ

```typescript
// alerts/consumers/slack.consumer.ts

@RabbitSubscribe({
  exchange: 'dani-admin.alerts',
  routingKey: 'slack',
  queue: 'dani-admin.alerts.slack',
  errorHandler: defaultNackErrorHandler,
})
async handleSlackAlert(payload: AlertSlackPayload): Promise<void> {
  const logger = new Logger('SlackConsumer')

  try {
    // Idempotência
    const alreadySent = await this.redis.get(`dani-admin:sent:slack:${payload.alertEventId}`)
    if (alreadySent) {
      logger.log(`Alerta ${payload.alertEventId} já enviado (idempotente)`)
      return
    }

    await this.slackService.sendWebhook(payload)
    await this.redis.set(`dani-admin:sent:slack:${payload.alertEventId}`, '1', 'EX', 86400)

  } catch (error) {
    logger.error(`Falha ao enviar alerta Slack ${payload.alertEventId}`, error)
    Sentry.captureException(error, {
      tags: {
        consumer: 'slack',
        alertType: payload.alertType,
        alertEventId: payload.alertEventId,
      },
      level: payload.alertType === 'DESLIGAMENTO_AUTOMATICO' ? 'fatal' : 'error',
    })

    // Relança para que o RabbitMQ execute retry (NACK sem requeue após máx retries)
    throw error
  }
}
```

**Comportamento de NACK:** após 3 tentativas (configurado no RabbitMQ via `x-max-retry` header), a mensagem é enviada para a DLQ `dani-admin.alerts.slack.dlq`. O Sentry captura o evento de falha final.

---

## 7. Logging de Erros com Pino

### 7.1 Configuração do Pino

```typescript
// main.ts
import { Logger } from 'nestjs-pino'

const app = await NestFactory.create(AppModule, {
  bufferLogs: true,
})

app.useLogger(app.get(Logger))
```

```typescript
// app.module.ts
import { LoggerModule } from 'nestjs-pino'

LoggerModule.forRoot({
  pinoHttp: {
    level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
    transport: process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty', options: { colorize: true } }
      : undefined,
    redact: {
      paths: [
        'req.headers.authorization',   // JWT não aparece em logs
        'req.body.password',
        'req.body.email',              // PII — proibido em logs
        '*.cpf',
        '*.senha',
      ],
      remove: true,
    },
    genReqId: (req) => req.headers['x-request-id'] ?? randomUUID(),
    serializers: {
      req: (req) => ({
        method: req.method,
        url: req.url,
        requestId: req.id,
      }),
      res: (res) => ({
        statusCode: res.statusCode,
      }),
    },
  },
})
```

### 7.2 Uso nos Services

```typescript
@Injectable()
export class TakeoverService {
  private readonly logger = new Logger(TakeoverService.name)

  async startTakeover(adminId: string, dto: StartTakeoverDto): Promise<TakeoverResponse> {
    this.logger.log(`Admin ${adminId} iniciando takeover da interação ${dto.interactionId}`)

    try {
      const result = await this.takeoverRepository.create(adminId, dto)
      this.logger.log(`Takeover ${result.id} criado com sucesso`)
      return result
    } catch (error) {
      if (error instanceof ConflictException) {
        this.logger.warn(`Conflito de takeover: interação ${dto.interactionId} já em atendimento`)
        throw error
      }
      this.logger.error(`Erro ao criar takeover: ${error.message}`, error.stack)
      throw error
    }
  }
}
```

**Regras de logging:**
- Nível `log` (info): operações bem-sucedidas relevantes (takeover criado, threshold atualizado).
- Nível `warn`: erros esperados de negócio (4xx), rate limit, conflitos.
- Nível `error`: erros de infraestrutura (5xx), timeouts, exceções não tratadas.
- **NUNCA** logar: JWT, passwords, CPF, e-mail, nomes completos, dados financeiros de usuários.

---

## 8. Tabela de Todos os Códigos de Erro

| Código | Módulo | HTTP | Situação |
|---|---|---|---|
| `DA-SUP-001` | Supervisão | 400 | Parâmetros de filtro inválidos |
| `DA-SUP-002` | Supervisão | 404 | Interação não encontrada |
| `DA-TAK-001` | Takeover | 409 | Interação já em atendimento por outro Admin |
| `DA-TAK-002` | Takeover | 400 | `interactionId` inválido ou ausente |
| `DA-TAK-003` | Takeover | 404 | Interação não encontrada |
| `DA-TAK-004` | Takeover | 422 | Interação ENCERRADA — takeover não permitido |
| `DA-TAK-005` | Takeover | 403 | Admin tenta encerrar takeover de outro Admin |
| `DA-TAK-006` | Takeover | 404 | Takeover não encontrado |
| `DA-TAK-007` | Takeover | 409 | Takeover já encerrado |
| `DA-MET-001` | Métricas | 400 | Parâmetro `period` inválido |
| `DA-MET-002` | Métricas | 404 | `agentId` não encontrado |
| `DA-CFG-001` | Configuração | 400 | `confidenceThreshold` fora de 50–95 |
| `DA-CFG-002` | Configuração | 400 | `rateLimitPerHour` fora de 1–100 |
| `DA-CFG-003` | Configuração | 404 | `agentId` não encontrado |
| `DA-CFG-004` | Configuração | 429 | Rate limit do webchat excedido |
| `DA-ALT-001` | Alertas | 404 | Alerta não encontrado |
| `DA-ALT-002` | Alertas | 409 | Alerta já reconhecido ou resolvido |
| `DA-ALT-003` | Alertas | 400 | Agente não está em DESLIGADO_AUTOMATICO |
| `DA-ALT-004` | Alertas | 404 | Agente não encontrado |
| `DA-LCH-001` | Checklist | 404 | Checklist não encontrado |
| `DA-LCH-002` | Checklist | 422 | Checklist com itens PENDENTE ou BLOQUEADO |
| `DA-LCH-003` | Checklist | 422 | Testes adversariais insuficientes (< 20) |
| `DA-LCH-004` | Checklist | 422 | Testes adversariais com falhas |
| `DA-PSH-001` | Push Tokens | 400 | Formato do token inválido |
| `DA-PSH-002` | Push Tokens | 409 | Token já registrado |
| `DA-AUD-001` | Auth | 403 | Role != ADMIN |
| `DA-AUD-002` | Auth | 401 | JWT expirado ou inválido |
| `DA-AUD-003` | Auth | 401 | Credenciais de login inválidas |
| `DA-AUD-004` | Auth | 401 | Refresh token expirado |
| `DA-SYS-400` | Sistema | 400 | Requisição inválida genérica |
| `DA-SYS-404` | Sistema | 404 | Recurso não encontrado genérico |
| `DA-SYS-409` | Sistema | 409 | Conflito genérico |
| `DA-SYS-422` | Sistema | 422 | Entidade não processável genérica |
| `DA-SYS-429` | Sistema | 429 | Rate limit genérico |
| `DA-SYS-500` | Sistema | 500 | Erro interno inesperado |
| `DA-SYS-503` | Sistema | 503 | Serviço indisponível (DB, Redis) |

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Shape único de erro, GlobalHttpExceptionFilter, ValidationPipe, tratamento de erros Prisma e RabbitMQ, tabela completa de códigos. |
