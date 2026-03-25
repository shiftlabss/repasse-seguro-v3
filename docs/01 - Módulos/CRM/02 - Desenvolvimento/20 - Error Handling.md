# 20 - Error Handling

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend Lead, Frontend Lead, QA |
| **Escopo** | Catálogo completo de erros do CRM, estratégia global de tratamento, UX de erro e dead-letter queue |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | Doc 01.1–01.5 Regras de Negócio CRM · Doc 02 Stacks CRM · Doc 16 API CRM · Doc 17 Integrações |

---

> **TL;DR**
>
> - **Error codes com prefixo `CRM-`** — 35 cenários documentados, categorizados por domínio.
> - **Padrão RFC 7807** para todas as respostas de erro da API.
> - **Estratégia global:** filtro de exceções NestJS, Sentry para P0/P1, dead-letter queue para falhas de integração.
> - **UX:** toast para erros transitórios, página de erro para falhas críticas, retry automático para integrações.
> - **Erros esperados vs inesperados** — apenas inesperados vão para o Sentry.

---

## 1. Classificação de Erros

### 1.1 Por Prioridade

| Prioridade | Descrição | Ação obrigatória |
|---|---|---|
| P0 — Crítico | Sistema inoperante, perda de dados, falha de segurança | Alerta imediato Sentry + PagerDuty (Admin RS notificado em < 15 min) |
| P1 — Alto | Funcionalidade crítica degradada (integração falhando, autenticação) | Sentry alert em < 30 min |
| P2 — Médio | Funcionalidade parcialmente degradada (relatório falhou, export falhou) | Sentry log, triagem no próximo dia útil |
| P3 — Baixo | Erros de validação, erros esperados de negócio | Log estruturado (Pino), sem alerta ativo |

### 1.2 Por Origem

| Origem | Tipo | Tratamento |
|---|---|---|
| Erros de validação | Esperados | 400/422, mensagem amigável, sem Sentry |
| Erros de negócio | Esperados | 422, mensagem amigável, sem Sentry |
| Erros de autenticação/autorização | Esperados | 401/403, mensagem amigável, sem Sentry |
| Erros de integração externa | Semi-esperados | 502, retry automático, Sentry se P0/P1 |
| Erros internos (bugs) | Inesperados | 500, mensagem genérica ao usuário, Sentry obrigatório |

---

## 2. Estratégia Global de Error Handling (NestJS)

### 2.1 Filtro de Exceções Global

```typescript
// crm-exception.filter.ts
@Catch()
export class CrmExceptionFilter implements ExceptionFilter {
  constructor(
    private readonly logger: PinoLogger,
    private readonly sentry: SentryService,
  ) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const { status, body } = this.normalize(exception);
    const isUnexpected = status >= 500;

    // Log estruturado sempre
    this.logger.error({
      type: 'http_error',
      status,
      code: body.code,
      path: request.url,
      method: request.method,
      user_id: (request as any).user?.id,
      request_id: request.headers['x-request-id'],
      error: isUnexpected ? exception : undefined,
    });

    // Sentry apenas para P0/P1 (5xx + 502)
    if (isUnexpected || status === 502) {
      this.sentry.captureException(exception, {
        tags: { module: 'crm', path: request.url },
        user: { id: (request as any).user?.id },
        extra: { body: body },
      });
    }

    response.status(status).json(body);
  }

  private normalize(exception: unknown): { status: number; body: RfcErrorBody } {
    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      const exceptionResponse = exception.getResponse() as any;

      return {
        status,
        body: {
          type: `https://api.repasseseguro.com.br/errors/crm`,
          title: this.getTitle(status),
          status,
          detail: exceptionResponse?.detail || exceptionResponse?.message || 'Erro na requisição',
          code: exceptionResponse?.code || `CRM-${status}`,
          errors: exceptionResponse?.errors,
        },
      };
    }

    // Erro inesperado — nunca expor detalhes internos ao cliente
    return {
      status: 500,
      body: {
        type: 'https://api.repasseseguro.com.br/errors/crm/internal',
        title: 'Erro Interno',
        status: 500,
        detail: 'Ocorreu um erro inesperado. Nossa equipe foi notificada.',
        code: 'CRM-500',
      },
    };
  }
}
```

### 2.2 Interceptor de Transformação de Resposta

```typescript
// transform.interceptor.ts — wrap responses bem-sucedidas
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, ApiResponse<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<ApiResponse<T>> {
    return next.handle().pipe(
      map((data) => ({
        data: data?.data ?? data,
        meta: data?.meta ?? undefined,
      }))
    );
  }
}
```

### 2.3 Dead-Letter Queue para Integrações

```typescript
// Padrão para todos os consumers RabbitMQ
@RabbitSubscribe({
  exchange: 'crm.direct',
  routingKey: 'crm.zapsign.events',
  queue: 'crm.zapsign.events',
  queueOptions: {
    durable: true,
    deadLetterExchange: 'crm.dlx',
    deadLetterRoutingKey: 'crm.zapsign.dlq',
  },
})
async handleZapsignEvent(msg: ZapsignEventDto): Promise<Nack | undefined> {
  try {
    await this.zapsignWebhookHandler.handle(msg);
  } catch (error) {
    this.logger.error({ type: 'consumer_error', queue: 'crm.zapsign.events', error });
    // Nack sem requeue → vai para DLQ após max_retries
    if (this.shouldRetry(error)) {
      return new Nack(true); // requeue uma vez
    }
    return new Nack(false); // DLQ
  }
}
```

---

## 3. Catálogo de Erros do CRM

### 3.1 Autenticação e Autorização (CRM-001 a CRM-010)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-001 | 401 | "E-mail ou senha incorretos." | invalid_credentials: email not found or password mismatch | P3 |
| CRM-002 | 401 | "Conta desativada. Entre em contato com o administrador." | account_disabled: is_active=false for user_id={id} | P3 |
| CRM-003 | 423 | "Conta bloqueada por excesso de tentativas. Tente novamente em {X} minutos." | account_locked: failed_attempts>=5, locked_until={ts} for user_id={id} | P3 |
| CRM-004 | 401 | "Sessão expirada. Faça login novamente." | refresh_token_invalid_or_expired: jti={jti} | P3 |
| CRM-005 | 400 | "Link de recuperação inválido ou expirado. Solicite um novo." | reset_token_invalid: token expired or already used | P3 |
| CRM-006 | 422 | "As senhas não coincidem." | password_confirmation_mismatch | P3 |
| CRM-007 | 401 | "Sua sessão foi invalidada. Faça login novamente." | session_blacklisted: user_id={id} | P3 |
| CRM-008 | 403 | "Você não tem permissão para realizar esta ação." | rbac_denied: role={role}, required={required_roles}, path={path} | P3 |
| CRM-009 | 401 | "Acesso negado. Autenticação necessária." | jwt_missing_or_malformed | P3 |
| CRM-010 | 401 | "Token de acesso inválido ou expirado." | jwt_verification_failed: {error_detail} | P3 |

### 3.2 Equipe e Usuários (CRM-011 a CRM-020)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-011 | 409 | "Este e-mail já está cadastrado no sistema." | user_email_conflict: email={email} already exists | P3 |
| CRM-012 | 409 | "Conflito de edição: o registro foi alterado por outro usuário. Recarregue e tente novamente." | optimistic_lock_conflict: team_member_id={id}, expected_version={v}, actual_version={v2} | P3 |
| CRM-013 | 403 | "Você não pode alterar o seu próprio papel no sistema." | self_role_change_forbidden: user_id={id} | P3 |
| CRM-014 | 422 | "Este usuário possui {N} Casos ativos. Redistribua os Casos antes de desativar." | cannot_deactivate_analyst_with_active_cases: user_id={id}, active_cases={N} | P3 |
| CRM-015 | 422 | "O analista de destino está inativo. Escolha um analista ativo." | redistribute_target_inactive: target_analyst_id={id} | P3 |
| CRM-016 | 404 | "Usuário não encontrado." | user_not_found: id={id} | P3 |
| CRM-017 | 422 | "Não é possível redistribuir Casos para o mesmo analista." | redistribute_same_analyst: source_id={id} == target_id={id} | P3 |

### 3.3 Casos (CRM-021 a CRM-030)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-021 | 422 | "Contato de Cedente não encontrado. Cadastre o Cedente antes de criar o Caso." | cedente_contact_not_found: contact_id={id} | P3 |
| CRM-022 | 422 | "Cenário inválido. Os cenários disponíveis são: A, B, C ou D." | invalid_scenario: received={scenario} | P3 |
| CRM-023 | 403 | "Você não tem acesso a este Caso." | case_access_denied: analyst_id={id}, case_id={id} | P3 |
| CRM-024 | 409 | "Conflito de edição: o Caso foi alterado por outro usuário. Recarregue e tente novamente." | optimistic_lock_conflict: case_id={id}, version_mismatch | P3 |
| CRM-025 | 422 | "Transição de estado inválida: não é possível ir de '{from}' para '{to}'." | invalid_case_transition: case_id={id}, from={from}, to={to} | P3 |
| CRM-026 | 422 | "Não é possível avançar o Caso. Critérios pendentes: {lista}" | transition_criteria_not_met: case_id={id}, unmet=[{criterios}] | P3 |
| CRM-027 | 403 | "Apenas Coordenadores RS e Admin RS podem cancelar Casos." | cancel_case_rbac: role={role}, case_id={id} | P3 |
| CRM-028 | 422 | "O analista selecionado está inativo. Escolha um analista ativo." | assign_inactive_analyst: analyst_id={id} | P3 |
| CRM-029 | 404 | "Caso não encontrado." | case_not_found: id={id} | P3 |
| CRM-030 | 422 | "O Caso já está no estado solicitado." | transition_same_state: case_id={id}, status={status} | P3 |

### 3.4 Contatos (CRM-031 a CRM-035)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-031 | 409 | "CPF ou e-mail já cadastrado no sistema." | contact_duplicate: cpf_or_email conflict for email={email} | P3 |
| CRM-032 | 409 | "Conflito de edição no Contato. Recarregue e tente novamente." | optimistic_lock_conflict: contact_id={id} | P3 |
| CRM-033 | 404 | "Contato não encontrado." | contact_not_found: id={id} | P3 |
| CRM-034 | 403 | "Você não tem acesso a este Contato." | contact_access_denied: analyst_id={id}, contact_id={id} | P3 |
| CRM-035 | 422 | "CPF inválido." | invalid_cpf: value={masked} | P3 |

### 3.5 Atividades (CRM-041 a CRM-045)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-041 | 422 | "Follow-up e Reunião exigem data/hora de agendamento." | missing_scheduled_at: type={type} requires scheduled_at | P3 |
| CRM-042 | 422 | "Atividades com mais de 1 hora só podem ser excluídas por Coordenadores RS ou Admin RS." | activity_delete_time_limit: activity_id={id}, created_at={ts} | P3 |
| CRM-043 | 404 | "Atividade não encontrada." | activity_not_found: id={id} | P3 |
| CRM-044 | 403 | "Você só pode editar atividades criadas por você." | activity_edit_ownership: analyst_id={id}, created_by={id2} | P3 |

### 3.6 Comunicação WhatsApp (CRM-051 a CRM-056)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-051 | 422 | "A janela de 24h expirou. Use um template aprovado para contatar este Contato." | whatsapp_window_expired: contact_id={id}, window_expired_at={ts} | P2 |
| CRM-052 | 422 | "Este Contato optou por não receber mensagens pelo WhatsApp." | whatsapp_opt_out: contact_id={id} | P3 |
| CRM-053 | 422 | "Template não encontrado ou não aprovado pela Meta." | whatsapp_template_not_found: template_name={name} | P2 |
| CRM-054 | 502 | "Não foi possível enviar a mensagem. Tente novamente em instantes." | whatsapp_api_error: meta_error_code={code}, detail={detail} | P1 |
| CRM-055 | 422 | "Número de telefone inválido para WhatsApp." | whatsapp_invalid_phone: contact_id={id} | P3 |
| CRM-056 | 429 | "Limite de mensagens atingido. Aguarde alguns minutos." | whatsapp_rate_limit: contact_id={id} | P2 |

### 3.7 Dossiê (CRM-061 a CRM-067)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-061 | 422 | "Tipo de arquivo inválido. Formatos aceitos: PDF, JPG, PNG." | invalid_document_type: mime_type={type} | P3 |
| CRM-062 | 422 | "Arquivo muito grande. Tamanho máximo: 10 MB." | document_size_exceeded: size={size_kb}KB, max=10240KB | P3 |
| CRM-063 | 422 | "Este documento já foi aprovado e não pode ser alterado." | document_already_approved: doc_id={id} | P3 |
| CRM-064 | 422 | "Documentos aprovados não podem ser removidos." | approved_document_delete_forbidden: doc_id={id} | P3 |
| CRM-065 | 404 | "Documento não encontrado no Dossiê." | document_not_found: doc_id={id} | P3 |
| CRM-066 | 422 | "Motivo de rejeição deve ter no mínimo 20 caracteres." | rejection_reason_too_short: length={len} | P3 |
| CRM-067 | 500 | "Erro ao fazer upload do documento. Tente novamente." | storage_upload_error: supabase_error={detail} | P1 |

### 3.8 Negociações (CRM-071 a CRM-076)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-071 | 422 | "Propostas só podem ser criadas quando o Caso está em negociação ou com match registrado." | proposal_invalid_case_state: case_id={id}, status={status} | P3 |
| CRM-072 | 422 | "Esta proposta expirou e não pode mais ser aceita." | proposal_expired: proposal_id={id}, expired_at={ts} | P3 |
| CRM-073 | 422 | "Esta proposta já foi respondida." | proposal_already_resolved: proposal_id={id}, status={status} | P3 |
| CRM-074 | 404 | "Proposta não encontrada." | proposal_not_found: id={id} | P3 |
| CRM-075 | 422 | "Valor da contraproposta deve ser maior que zero." | counterproposal_invalid_value: value={value} | P3 |

### 3.9 Comissão (CRM-081 a CRM-084)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-081 | 422 | "Desconto máximo permitido é {X}% para este Caso." | commission_discount_cap_exceeded: case_id={id}, requested={pct}, cap={cap} | P3 |
| CRM-082 | 422 | "Motivo do desconto deve ter no mínimo 20 caracteres." | discount_reason_too_short: length={len} | P3 |
| CRM-083 | 422 | "Comissão já confirmada não pode ser alterada sem aprovação do Admin RS." | commission_confirmed_readonly: case_id={id} | P3 |
| CRM-084 | 404 | "Registro de comissão não encontrado." | commission_not_found: case_id={id} | P3 |

### 3.10 SLA e Configuração (CRM-091 a CRM-102)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-091 | 422 | "O prazo crítico deve ser maior que o prazo de atenção." | sla_config_invalid: critical_hours ({c}h) < warning_hours ({w}h) | P3 |
| CRM-092 | 404 | "Configuração de SLA não encontrada para o estado especificado." | sla_config_not_found: state={state} | P3 |
| CRM-101 | 422 | "Tipo de dado inválido para esta configuração." | config_type_mismatch: key={key}, expected={type}, received={type2} | P3 |
| CRM-102 | 422 | "Esta configuração é somente leitura e não pode ser alterada via API." | config_readonly_key: key={key} | P3 |

### 3.11 Integrações Externas (CRM-111 a CRM-120)

| Code | HTTP | Mensagem (pt-BR — usuário) | Mensagem técnica (logs) | P |
|---|---|---|---|---|
| CRM-111 | 401 | N/A (webhook — sem frontend) | webhook_signature_invalid: source={source}, received={sig} | P1 |
| CRM-112 | 502 | "Não foi possível enviar o contrato para assinatura. Tente novamente em instantes." | zapsign_api_error: status={status}, body={body} | P1 |
| CRM-113 | 422 | "Aguarde 5 minutos antes de tentar reenviar o contrato." | zapsign_resend_cooldown: envelope_id={id}, last_attempt={ts} | P3 |
| CRM-114 | 422 | "Limite de {N} tentativas de envio atingido. O Admin RS foi notificado." | zapsign_max_retries: envelope_id={id}, attempts={N} | P2 |
| CRM-115 | 502 | "Integração com a Plataforma RS temporariamente indisponível. Dados serão sincronizados automaticamente." | platform_sync_error: direction={dir}, case_id={id}, error={detail} | P1 |
| CRM-116 | 502 | "Confirmação de depósito Escrow não disponível. Notificação enviada ao Coordenador RS." | escrow_webhook_processing_error: case_id={id}, error={detail} | P0 |
| CRM-117 | 500 | "Erro ao processar relatório. Tente novamente em alguns instantes." | report_generation_error: job_id={id}, error={detail} | P2 |
| CRM-118 | 404 | "Job de exportação não encontrado." | export_job_not_found: job_id={id} | P3 |
| CRM-119 | 422 | "URL de download expirada. Gere um novo relatório." | export_url_expired: job_id={id} | P3 |
| CRM-120 | 500 | "Erro ao gerar URL de download. Tente novamente." | storage_signed_url_error: doc_id={id}, error={detail} | P1 |

---

## 4. UX de Erro no Frontend

### 4.1 Toast — Erros Transitórios

Erros recuperáveis exibem toast no canto superior direito. Dismissível automaticamente em 5s para erros de validação, persistente até dismiss manual para erros de integração.

```typescript
// Mapeamento de code → toast config
const ERROR_TOAST_CONFIG: Record<string, ToastConfig> = {
  'CRM-001': { type: 'error', title: 'E-mail ou senha incorretos', duration: 5000 },
  'CRM-003': { type: 'warning', title: 'Conta bloqueada', description: 'Tente novamente em {X} minutos', duration: 'persistent' },
  'CRM-024': { type: 'warning', title: 'Conflito de edição', description: 'Recarregue a página e tente novamente', duration: 'persistent', action: { label: 'Recarregar', onClick: () => window.location.reload() } },
  'CRM-051': { type: 'warning', title: 'Janela de 24h expirada', description: 'Use um template aprovado para este contato', duration: 8000 },
  'CRM-054': { type: 'error', title: 'Falha no envio da mensagem', description: 'Mensagem não entregue — tente novamente', duration: 'persistent' },
  'CRM-112': { type: 'error', title: 'Erro ao enviar contrato', description: 'Aguarde instantes e tente novamente', duration: 'persistent' },
};

// Uso nos hooks do frontend
function useCrmMutation<T>(mutationFn: () => Promise<T>) {
  return useMutation({
    mutationFn,
    onError: (error: CrmApiError) => {
      const config = ERROR_TOAST_CONFIG[error.code] ?? {
        type: 'error',
        title: 'Erro inesperado',
        description: 'Nossa equipe foi notificada.',
        duration: 'persistent',
      };
      toast(config);
    },
  });
}
```

### 4.2 Página de Erro — Falhas Críticas

Para erros 500 e falhas que impedem o uso da tela atual:

```
┌─────────────────────────────────────────────────────┐
│  ⚠️  Ocorreu um erro inesperado                     │
│                                                     │
│  Nossa equipe foi notificada automaticamente.       │
│  Tente recarregar a página ou voltar ao início.     │
│                                                     │
│  [Recarregar página]    [Voltar ao Dashboard]       │
│                                                     │
│  Código de referência: CRM-500-xxxx-yyyy            │
└─────────────────────────────────────────────────────┘
```

### 4.3 Inline — Erros de Formulário

Erros de validação (400/422) exibidos diretamente no campo do formulário:

```typescript
// Integração React Hook Form + Zod + API errors
function CaseForm() {
  const { setError } = useForm<CreateCaseInput>({ resolver: zodResolver(createCaseSchema) });

  const onSubmit = async (data: CreateCaseInput) => {
    try {
      await createCase(data);
    } catch (error) {
      if (error.code === 'CRM-021') {
        setError('cedente_contact_id', { message: 'Contato de Cedente não encontrado' });
      }
      if (error.status === 422 && error.errors) {
        error.errors.forEach(({ field, message }) => setError(field, { message }));
      }
    }
  };
}
```

### 4.4 Retry Automático — Erros de Integração (Frontend)

```typescript
// TanStack Query — retry automático para erros de rede/integração
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error: CrmApiError) => {
        // Não retry para erros de cliente (4xx)
        if (error.status < 500) return false;
        // Máximo 3 retries para erros de servidor/integração (5xx, 502)
        return failureCount < 3;
      },
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 10000), // exp backoff
    },
    mutations: {
      retry: false, // Mutations: nunca retry automático — usuário decide
    },
  },
});
```

---

## 5. Logs Estruturados (Pino)

```typescript
// Formato padrão de log de erro no CRM
{
  "level": "error",
  "time": "2026-03-23T14:32:01.234Z",
  "msg": "case_transition_failed",
  "module": "crm",
  "request_id": "req-abc123",
  "user_id": "uuid",
  "user_role": "ANALISTA_RS",
  "path": "/api/v1/crm/cases/uuid/transition",
  "method": "PATCH",
  "status": 422,
  "code": "CRM-025",
  "detail": "invalid_case_transition: from=NEGOCIACAO, to=PUBLICACAO",
  "duration_ms": 45
}
```

**Campos obrigatórios em todo log de erro:**
- `request_id` — propagado do header `X-Request-ID`
- `user_id` — do JWT (quando autenticado)
- `module: "crm"` — para filtragem no painel do Sentry
- `code` — CRM-XXX

---

## 6. Configuração do Sentry

```typescript
// Sentry configurado em main.ts do NestJS
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new ProfilingIntegration(),
  ],
  beforeSend(event) {
    // Remover dados pessoais dos eventos Sentry
    if (event.request?.data) {
      event.request.data = maskPii(event.request.data);
    }
    return event;
  },
});

// Alertas automáticos configurados no Sentry (via webhook ao Slack #crm-alertas):
// - P0: qualquer CRM-116, CRM-067, erros 500 em produção → alerta imediato
// - P1: CRM-054, CRM-112, CRM-115 → alerta em 30 min
// - P2: > 10 ocorrências de mesmo error code em 1 hora → alerta P2
```

---

## 7. Tabela de Referência Rápida

| Código | HTTP | Prioridade | Domínio |
|---|---|---|---|
| CRM-001 | 401 | P3 | Auth |
| CRM-002 | 401 | P3 | Auth |
| CRM-003 | 423 | P3 | Auth |
| CRM-004 | 401 | P3 | Auth |
| CRM-005 | 400 | P3 | Auth |
| CRM-006 | 422 | P3 | Auth |
| CRM-007 | 401 | P3 | Auth |
| CRM-008 | 403 | P3 | Auth/RBAC |
| CRM-009 | 401 | P3 | Auth |
| CRM-010 | 401 | P3 | Auth |
| CRM-011 | 409 | P3 | Equipe |
| CRM-012 | 409 | P3 | Equipe |
| CRM-013 | 403 | P3 | Equipe |
| CRM-014 | 422 | P3 | Equipe |
| CRM-015 | 422 | P3 | Equipe |
| CRM-016 | 404 | P3 | Equipe |
| CRM-017 | 422 | P3 | Equipe |
| CRM-021 | 422 | P3 | Casos |
| CRM-022 | 422 | P3 | Casos |
| CRM-023 | 403 | P3 | Casos |
| CRM-024 | 409 | P3 | Casos |
| CRM-025 | 422 | P3 | Casos |
| CRM-026 | 422 | P3 | Casos |
| CRM-027 | 403 | P3 | Casos |
| CRM-028 | 422 | P3 | Casos |
| CRM-029 | 404 | P3 | Casos |
| CRM-030 | 422 | P3 | Casos |
| CRM-031 | 409 | P3 | Contatos |
| CRM-032 | 409 | P3 | Contatos |
| CRM-033 | 404 | P3 | Contatos |
| CRM-041 | 422 | P3 | Atividades |
| CRM-042 | 422 | P3 | Atividades |
| CRM-051 | 422 | P2 | WhatsApp |
| CRM-052 | 422 | P3 | WhatsApp |
| CRM-053 | 422 | P2 | WhatsApp |
| CRM-054 | 502 | P1 | WhatsApp |
| CRM-061 | 422 | P3 | Dossiê |
| CRM-062 | 422 | P3 | Dossiê |
| CRM-063 | 422 | P3 | Dossiê |
| CRM-064 | 422 | P3 | Dossiê |
| CRM-067 | 500 | P1 | Dossiê/Storage |
| CRM-071 | 422 | P3 | Negociações |
| CRM-072 | 422 | P3 | Negociações |
| CRM-081 | 422 | P3 | Comissão |
| CRM-091 | 422 | P3 | SLA |
| CRM-101 | 422 | P3 | Config |
| CRM-111 | 401 | P1 | Webhooks |
| CRM-112 | 502 | P1 | ZapSign |
| CRM-113 | 422 | P3 | ZapSign |
| CRM-114 | 422 | P2 | ZapSign |
| CRM-115 | 502 | P1 | Plataforma RS |
| CRM-116 | 502 | P0 | Escrow |
| CRM-117 | 500 | P2 | Relatórios |
| CRM-120 | 500 | P1 | Storage |

---

## 8. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 53 error codes CRM-XXX, filtro global NestJS, Sentry P0/P1, dead-letter queue, UX de erro (toast/página/inline/retry automático). |
