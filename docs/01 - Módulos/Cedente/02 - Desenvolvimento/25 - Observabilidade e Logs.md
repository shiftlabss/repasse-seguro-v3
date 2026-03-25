# 25 - Observabilidade e Logs

## Módulo Cedente · Plataforma Repasse Seguro

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 25 - Observabilidade e Logs | v1.0 | 2026-03-23 (America/Fortaleza) | Claude Code Desktop | Aprovado |

| Campo | Valor |
|---|---|
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 |

---

> **TL;DR**
>
> - **Sentry:** error tracking em todas as camadas (NestJS, Next.js, Expo). Deploy sem Sentry configurado é proibido.
> - **PostHog:** eventos de produto em snake_case. Wrapper centralizado em `src/services/analytics.ts`. Session replay habilitado.
> - **Langfuse:** observabilidade do Guardião do Retorno — tracing, latência, custo por conversa, qualidade de resposta.
> - **Pino:** logs estruturados em JSON no NestJS. Campos obrigatórios: `requestId`, `userId`, `module`, `action`.
> - **SLAs:** API < 200ms p95, upload < 5s, notificação < 30s, e-mail < 5min. Violações geram alertas automáticos.
> - **Dashboards:** Railway metrics (infra), Supabase Dashboard (banco), Sentry (erros), Langfuse (IA).

---

## 1. Arquitetura de Observabilidade

```
┌───────────────────────────────────────────────────────────────────┐
│                     Módulo Cedente                                │
│                                                                   │
│  Next.js (web)     Expo (mobile)     NestJS (backend)            │
│       │                 │                  │                      │
│       ├─── Sentry ──────┤──────────────────┤  (erros)            │
│       ├─── PostHog ─────┤──────────────────┤  (produto)          │
│       │                 │                  │                      │
│       │                 │         NestJS → Pino → stdout         │
│       │                 │         Railway coleta → Sentry Logs   │
│       │                 │                  │                      │
│       │                 │         NestJS → Langfuse  (IA)        │
│       │                 │                  │                      │
│       └─────────────────┴──────────────────┘                     │
│                                                                   │
│  Dashboards:                                                      │
│  • sentry.io — erros e performance                               │
│  • posthog.com — eventos de produto e funis                      │
│  • langfuse.com — Guardião do Retorno                            │
│  • railway.app — infra (CPU, memória, logs)                      │
│  • app.supabase.com — banco (queries, RLS, storage)             │
└───────────────────────────────────────────────────────────────────┘
```

---

## 2. Sentry — Error Tracking

### 2.1 Configuração por camada

**Backend NestJS (`apps/api`):**

```typescript
// apps/api/src/main.ts
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.npm_package_version,
  // Performance monitoring
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  // Ignorar erros esperados
  ignoreErrors: [
    'UnauthorizedException',
    'NotFoundException',
  ],
});

// Integração com NestJS
import { SentryGlobalFilter } from '@sentry/nestjs/setup';

// No AppModule:
{
  provide: APP_FILTER,
  useClass: SentryGlobalFilter,
}
```

**Frontend Next.js (`apps/web-cedente`):**

```typescript
// apps/web-cedente/sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.NEXT_PUBLIC_APP_VERSION,
  // Performance monitoring
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.05 : 1.0,
  // Session Replay — captura replay em caso de erro
  replaysOnErrorSampleRate: 1.0,
  replaysSessionSampleRate: 0.05,
  integrations: [
    Sentry.replayIntegration({
      // Mascarar dados sensíveis nos replays
      maskAllInputs: true,
      maskAllText: false,
      blockAllMedia: false,
    }),
  ],
});
```

**Mobile Expo (`apps/mobile-cedente`):**

```typescript
// apps/mobile-cedente/src/app/_layout.tsx
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV ?? 'development',
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  attachScreenshot: true, // captura screenshot no momento do erro
  attachViewHierarchy: true,
});
```

### 2.2 Classificação de alertas

| Severidade | Critério | SLA de Resposta | Canal |
|---|---|---|---|
| **P0 — Fatal** | Exception não tratada em fluxo financeiro ou autenticação | 5 minutos | PagerDuty + Slack `#incidentes` |
| **P1 — Error** | Taxa de erros > 1% em qualquer endpoint crítico | 15 minutos | Slack `#incidentes` |
| **P2 — Warning** | Erro isolado não crítico, timeout de integração externa | 30 minutos | Slack `#alertas` |

**Alertas de Sentry configurados:**

| Alerta | Condição | Destino |
|---|---|---|
| P0 — Erro em fluxo financeiro | `module: cedente-financeiro` + nível `fatal` | PagerDuty + Slack `#incidentes` |
| P0 — Erro em autenticação | `module: cedente-auth` + nível `error` + > 5 por minuto | Slack `#incidentes` |
| P1 — Upload com falha recorrente | `module: cedente-documentos` + `action: upload` + > 3 erros/5min | Slack `#alertas` |
| P1 — ZapSign indisponível | `module: cedente-assinaturas` + `integration: zapsign` + `error` | Slack `#incidentes` |
| P1 — Guardião do Retorno falhando | `module: cedente-guardiao` + nível `error` + > 5 por minuto | Slack `#alertas` |
| P2 — Degradação de SLA | Latência P95 > 300ms por 5 minutos | Slack `#alertas` |

### 2.3 Context tags obrigatórios

Toda captura manual de erro deve incluir contexto do Módulo Cedente:

```typescript
// ✅ Captura com contexto rico
try {
  await this.zapsignService.enviarDocumento(envelope);
} catch (error) {
  Sentry.withScope((scope) => {
    scope.setTag('module', 'cedente-assinaturas');
    scope.setTag('integration', 'zapsign');
    scope.setTag('action', 'enviar_documento');
    scope.setUser({ id: cedenteId });
    scope.setExtra('casoId', casoId);
    scope.setExtra('envelopeId', envelope.id);
    Sentry.captureException(error);
  });
  throw error;
}
```

### 2.4 Source maps em produção

Source maps são enviados ao Sentry durante o build via CI (nunca expostos publicamente):

```bash
# No workflow de CI:
pnpm build
npx @sentry/cli sourcemaps inject --org <org> --project cedente-api dist/
npx @sentry/cli sourcemaps upload --org <org> --project cedente-api dist/
```

---

## 3. PostHog — Analytics de Produto

### 3.1 Wrapper centralizado

```typescript
// apps/web-cedente/src/services/analytics.ts
import posthog from 'posthog-js';

// Tipos dos eventos do Módulo Cedente
type CedenteEvent =
  | 'case_created'
  | 'case_draft_saved'
  | 'simulator_viewed'             // RN-021: simulador visualizado
  | 'cenario_selected'             // RN-022: cenário escolhido
  | 'dossie_uploaded'              // RN-043: dossiê enviado
  | 'document_uploaded'
  | 'document_rejected'
  | 'proposal_received'            // RN-030: proposta recebida
  | 'proposal_accepted'            // RN-032: proposta aceita
  | 'proposal_rejected'
  | 'counterproposal_sent'         // RN-035: contraproposta enviada
  | 'proposal_expired'             // RN-031: proposta expirada
  | 'escalation_requested'         // RN-025: escalonamento solicitado
  | 'escalation_completed'
  | 'signing_completed'            // RN-047, RN-048: assinatura ZapSign
  | 'closing_completed'            // Fechamento concluído
  | 'reversion_requested'          // RN-039: desistência pós-Fechamento
  | 'guardiao_conversation_started' // RN-058: conversa com Guardião
  | 'guardiao_escalated_to_human'  // RN-061: Guardião escalou para humano
  | 'lgpd_deletion_requested'      // RN-010: solicitação LGPD
  | 'mobile_camera_upload';        // RN-087: upload via câmera

interface EventProperties {
  cedente_id?: string;
  caso_id?: string;
  cenario?: 'A' | 'B' | 'C' | 'D';
  [key: string]: string | number | boolean | undefined;
}

export const analytics = {
  track(event: CedenteEvent, properties?: EventProperties): void {
    if (typeof window === 'undefined') return; // SSR guard
    posthog.capture(event, {
      ...properties,
      module: 'cedente', // tag sempre presente
    });
  },

  identify(cedenteId: string, properties?: Record<string, unknown>): void {
    posthog.identify(cedenteId, properties);
  },

  reset(): void {
    posthog.reset();
  },
};
```

### 3.2 Eventos obrigatórios por fluxo

**Cadastro de imóvel:**
```typescript
// RN-021: simulador visualizado por pelo menos 10s
analytics.track('simulator_viewed', { caso_id: rascunhoId, cenarios_exibidos: 4 });

// RN-022: cenário selecionado
analytics.track('cenario_selected', { caso_id: rascunhoId, cenario: 'C' });

// Cadastro concluído
analytics.track('case_created', { caso_id: novoCase.id, cenario: novoCase.cenario });
```

**Propostas:**
```typescript
// Proposta recebida
analytics.track('proposal_received', {
  caso_id: casoId,
  proposta_id: proposta.id,
  cenario: caso.cenario,
});

// Aceite com dupla confirmação (RN-032)
analytics.track('proposal_accepted', {
  caso_id: casoId,
  proposta_id: propostaId,
  tempo_resposta_horas: tempoResposta, // métricas para V-02
});
```

**Guardião do Retorno:**
```typescript
// Conversa iniciada (RN-058)
analytics.track('guardiao_conversation_started', {
  caso_id: casoId,
  contexto: 'dashboard', // de onde o Cedente abriu o chat
});

// Escalação para humano (RN-061)
analytics.track('guardiao_escalated_to_human', {
  caso_id: casoId,
  motivo: 'confianca_insuficiente',
});
```

### 3.3 Feature flags do Guardião do Retorno

```typescript
// Verificar se Guardião está habilitado (kill switch)
import { useFeatureFlagEnabled } from 'posthog-js/react';

function GuardiaoChatButton({ casoId }: { casoId: string }) {
  const guardioEnabled = useFeatureFlagEnabled('guardiao-do-retorno');

  if (!guardioEnabled) return null;
  return <button onClick={() => openGuardiaoChat(casoId)}>Falar com Assistente</button>;
}
```

### 3.4 Funis recomendados no PostHog

| Funil | Eventos | Objetivo |
|---|---|---|
| Cadastro completo | `case_draft_saved` → `simulator_viewed` → `cenario_selected` → `case_created` | Taxa de conclusão do wizard |
| Proposta → Aceite | `proposal_received` → `proposal_accepted` | Taxa de conversão de propostas |
| Engajamento com Guardião | `guardiao_conversation_started` → `guardiao_escalated_to_human` | Taxa de resolução autônoma |
| Mobile critical path | `mobile_camera_upload` → `dossie_uploaded` | Sucesso no fluxo mobile (V-05) |

---

## 4. Langfuse — Observabilidade do Guardião do Retorno

### 4.1 Integração no NestJS

```typescript
// apps/api/src/modules/cedente-guardiao/guardiao.service.ts
import { Langfuse } from 'langfuse';
import { ChatOpenAI } from '@langchain/openai';
import { CallbackHandler } from 'langfuse-langchain';

@Injectable()
export class GuardiaoService {
  private readonly langfuse: Langfuse;

  constructor() {
    this.langfuse = new Langfuse({
      secretKey: process.env.LANGFUSE_SECRET_KEY!,
      publicKey: process.env.LANGFUSE_PUBLIC_KEY!,
      baseUrl: process.env.LANGFUSE_BASE_URL!,
    });
  }

  async responder(cedenteId: string, casoId: string, mensagem: string): Promise<string> {
    // Criar trace para rastrear a conversa completa
    const trace = this.langfuse.trace({
      name: 'guardiao-responder',
      userId: cedenteId,
      metadata: { casoId, modulo: 'cedente' },
      tags: ['guardiao', 'cedente', 'producao'],
    });

    const handler = new CallbackHandler({ root: trace });

    const llm = new ChatOpenAI({
      model: process.env.OPENAI_MODEL!,
      temperature: 0.3,
      maxTokens: 800,
      callbacks: [handler],
    });

    // ... pipeline RAG + chain

    // Pontuar confiança da resposta (RN-063)
    trace.score({
      name: 'confianca',
      value: nivelConfianca, // 0.0 a 1.0
      comment: 'Auto-score baseado em threshold do retrieval',
    });

    await this.langfuse.flushAsync();
    return resposta;
  }
}
```

### 4.2 Métricas monitoradas no Langfuse

| Métrica | Descrição | Alerta |
|---|---|---|
| `latencia_p95` | Tempo total de resposta do Guardião (p95) | > 5s → alerta P2 |
| `custo_por_conversa` | Custo OpenAI por sessão | > $0.10 → revisão de prompts |
| `taxa_escalacao_humano` | % conversas escaladas para humano (RN-061) | > 30% → revisar base de conhecimento |
| `confianca_media` | Score médio de confiança das respostas | < 0.7 → revisar prompts e RAG |
| `erros_openai` | Erros de chamada à OpenAI API | > 3/hora → alerta P1 |
| `uso_tokens_input` | Tokens de input (contexto) | > 4000 avg → otimizar RAG |

### 4.3 Evals periódicas

```typescript
// Rodar periodicamente (fora do CI — semanal via cron GitHub Actions)
// Validar com golden dataset de 20-50 exemplos

const evals = [
  {
    input: 'Qual o status do meu caso?',
    expected: /status|análise|proposta|formalização/i,  // deve responder com status real
  },
  {
    input: 'Quando vou receber uma proposta?',
    // RN-060: NÃO deve garantir prazo — deve usar linguagem de estimativa
    forbidden: /vou receber|receberá|garantido|certeza/i,
  },
  {
    input: 'Simule o cenário B para meu imóvel',
    expected: /simulação|estimativa|cenário B/i,
  },
];
```

---

## 5. Pino — Logs Estruturados

### 5.1 Configuração no NestJS

```typescript
// apps/api/src/logger/pino.config.ts
import pino from 'pino';

export const pinoLogger = pino({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  formatters: {
    level: (label) => ({ level: label }),
  },
  // Em produção: JSON puro (Railway coleta e indexa)
  // Em desenvolvimento: pretty print
  transport:
    process.env.NODE_ENV !== 'production'
      ? { target: 'pino-pretty', options: { colorize: true } }
      : undefined,
  // Campos sempre presentes
  base: {
    service: 'cedente-api',
    version: process.env.npm_package_version,
    env: process.env.NODE_ENV,
  },
});
```

### 5.2 Campos obrigatórios em todo log

Todo log do Módulo Cedente deve incluir:

| Campo | Tipo | Descrição | Exemplo |
|---|---|---|---|
| `requestId` | string (UUID) | Identificador único da requisição HTTP | `"550e8400-e29b-41d4-a716-446655440000"` |
| `userId` | string \| null | ID do Cedente autenticado | `"ced_abc123"` |
| `module` | string | Módulo do sistema | `"cedente-propostas"` |
| `action` | string | Ação executada | `"aceitar_proposta"` |
| `level` | string | Nível do log | `"info"`, `"warn"`, `"error"` |
| `timestamp` | ISO8601 | Data/hora UTC | `"2026-03-23T14:30:00.000Z"` |

**Campos adicionais por contexto:**

| Campo | Quando incluir |
|---|---|
| `casoId` | Em qualquer operação de caso |
| `propostaId` | Em operações de proposta |
| `integration` | Em chamadas a integrações externas (`zapsign`, `escrow`, `openai`) |
| `duration_ms` | Em operações lentas (uploads, chamadas externas) |
| `statusCode` | Em respostas HTTP |

### 5.3 Exemplos de logs estruturados

```typescript
// Interceptor de requisição — adiciona requestId
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest<Request>();
    const requestId = crypto.randomUUID();
    const { method, url } = request;

    this.logger.info({
      requestId,
      userId: request.user?.id ?? null,
      module: 'api',
      action: 'request_start',
      method,
      url,
    });

    const start = Date.now();

    return next.handle().pipe(
      tap(() => {
        this.logger.info({
          requestId,
          userId: request.user?.id ?? null,
          module: 'api',
          action: 'request_end',
          method,
          url,
          statusCode: context.switchToHttp().getResponse<Response>().statusCode,
          duration_ms: Date.now() - start,
        });
      }),
    );
  }
}
```

```typescript
// Log de operação de negócio
this.logger.info({
  requestId,
  userId: cedenteId,
  module: 'cedente-propostas',
  action: 'proposta_aceita',
  casoId,
  propostaId,
  cenario: caso.cenario,
});

// Log de integração externa
this.logger.info({
  requestId,
  userId: cedenteId,
  module: 'cedente-assinaturas',
  action: 'zapsign_envelope_criado',
  integration: 'zapsign',
  casoId,
  envelopeId: envelope.id,
  duration_ms: Date.now() - start,
});

// Log de erro com contexto completo
this.logger.error({
  requestId,
  userId: cedenteId,
  module: 'cedente-assinaturas',
  action: 'zapsign_webhook_processamento_falhou',
  integration: 'zapsign',
  casoId,
  error: {
    message: error.message,
    stack: process.env.NODE_ENV !== 'production' ? error.stack : undefined,
  },
});
```

### 5.4 O que NÃO logar

- CPF, CNPJ, dados bancários (PII).
- Senhas, tokens JWT, API keys.
- Conteúdo de mensagens do Guardião (privado ao Cedente — RN-062).
- Dados do Cessionário (anonimato — RN-012).

---

## 6. Métricas de SLA

### 6.1 SLAs do Módulo Cedente

| Métrica | SLA | Medição | Alerta |
|---|---|---|---|
| Latência API (p95) | < 200ms | Railway metrics + Sentry performance | > 200ms por 5min → P1 |
| Latência API (p99) | < 500ms | Railway metrics | > 500ms por 5min → P1 |
| Upload de documentos (p95) | < 5s | Log `duration_ms` + Sentry | > 5s por 3 uploads → P2 |
| Notificação no painel | < 30s | Supabase Realtime lag (RN-057) | > 30s consistente → P2 |
| E-mail transacional | < 5min | Resend dashboard (RN-056) | > 5min em 3 e-mails → P2 |
| Guardião do Retorno (p95) | < 5s | Langfuse latência trace | > 5s → P2 |
| Atualização Dashboard | < 60s | Supabase Realtime (RN-014) | > 60s consistente → P2 |
| Uptime API | > 99.5% | Railway uptime | < 99.5% no mês → P0 |

### 6.2 Alertas de degradação de SLA

Configurar no Sentry (Performance Alerts):

```
Alerta: "API latência P95 acima do SLA"
Condição: p95(transaction.duration) > 200ms
Por: 5 minutos consecutivos
Destino: Slack #alertas
```

```
Alerta: "Taxa de erro acima de 1%"
Condição: (errors/total_requests) > 1%
Por: 5 minutos consecutivos
Destino: Slack #incidentes
```

---

## 7. Dashboards Recomendados

### 7.1 Dashboard de Infra — Railway

URL: `https://railway.app/dashboard`

Métricas a acompanhar diariamente:
- CPU usage (alerta se > 80% por mais de 10 min)
- Memory usage (alerta se > 85%)
- Deploy history — quantos deploys hoje
- Logs em tempo real do serviço

### 7.2 Dashboard de Banco — Supabase

URL: `https://app.supabase.com` → projeto produção

Métricas a acompanhar:
- Queries mais lentas (Query Performance)
- Conexões ativas (< 90% do pool)
- Storage utilizado
- Auth: logins bem-sucedidos e falhos (monitorar RN-005: tentativas bloqueadas)
- RLS policies ativas em todas as tabelas de Cedente

**Query de verificação de RLS:**
```sql
-- Verificar tabelas sem RLS habilitado (rodar semanalmente)
SELECT schemaname, tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public'
  AND rowsecurity = false
  AND tablename NOT IN ('schema_migrations'); -- tabelas que propositalmente não têm RLS
```

### 7.3 Dashboard de Erros — Sentry

URL: `https://sentry.io/{org}/repasse-seguro`

Acompanhar diariamente:
- Issues novas nas últimas 24h
- Taxa de erros por endpoint
- Performance: endpoints mais lentos
- Session Replay de sessões com erro

### 7.4 Dashboard de Produto — PostHog

URL: `https://posthog.com/{org}`

Acompanhar semanalmente:
- Funil de cadastro: taxa de conclusão
- Funil de proposta → aceite
- Retenção de usuários (Cedentes que retornam)
- Feature flags: % de usuários com Guardião habilitado

### 7.5 Dashboard de IA — Langfuse

URL: `https://cloud.langfuse.com`

Acompanhar semanalmente:
- Custo total OpenAI no período
- Latência média e p95 das conversas
- Taxa de escalação para humano
- Traces com score de confiança < 0.5 (revisar manualmente)

---

## 8. Runbook de Observabilidade

### 8.1 Como investigar um spike de erros

```bash
# 1. Verificar Sentry — Issues recentes
# sentry.io → Issues → filter: last 1h, status: new

# 2. Verificar logs Railway
railway logs --service cedente-api-prod --tail 100

# 3. Filtrar logs por módulo com jq
railway logs --service cedente-api-prod | grep '"module":"cedente-assinaturas"' | jq .

# 4. Verificar se é deploy recente causando o problema
railway deployments --service cedente-api-prod

# 5. Verificar dependências externas
curl -s https://status.supabase.com/api/v2/summary.json | jq '.status.description'
```

### 8.2 Como investigar latência elevada

```bash
# 1. Verificar Railway metrics — CPU e memória
# railway.app/dashboard → service → Metrics

# 2. Verificar Supabase Query Performance
# app.supabase.com → projeto → Reports → Query Performance

# 3. Verificar se há queries sem índice
# app.supabase.com → SQL Editor:
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

# 4. Verificar Sentry Performance — endpoints mais lentos
# sentry.io → Performance → filter: module=cedente
```

### 8.3 Alarmes de custo OpenAI

```bash
# Verificar custo atual via Langfuse
# cloud.langfuse.com → Dashboard → Cost

# Se custo diário > $10 (threshold de alerta):
# 1. Verificar Langfuse traces com tokens_input > 3000
# 2. Revisar prompt do Guardião — possível context stuffing
# 3. Verificar se algum usuário está em loop de requests
#    (rate limiting em 30 req/min — Stacks 9.3)
```

---

## 9. Referências

- Doc 02 — Stacks Tecnológicas: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/02 - Stacks.md`
- Doc 26 — Runbook Operacional: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/26 - Runbook Operacional.md`
- Documentação Sentry NestJS: [docs.sentry.io/platforms/node/guides/nestjs](https://docs.sentry.io/platforms/node/guides/nestjs)
- Documentação Sentry Next.js: [docs.sentry.io/platforms/javascript/guides/nextjs](https://docs.sentry.io/platforms/javascript/guides/nextjs)
- Documentação Langfuse: [langfuse.com/docs](https://langfuse.com/docs)
- Documentação PostHog: [posthog.com/docs](https://posthog.com/docs)
- Documentação Pino: [getpino.io](https://getpino.io)
