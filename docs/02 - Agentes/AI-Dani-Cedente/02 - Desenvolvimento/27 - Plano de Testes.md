# 27 - Plano de Testes — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| **Nome do Documento** | Plano de Testes — AI-Dani-Cedente |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Bloco** | 6 — Qualidade |
| **Dependências** | D01, D02, D05, D12, D14, D16 |

---

> **📌 TL;DR**
>
> - **Pirâmide:** 60% unitário / 30% integração / 8% E2E / 2% manual exploratório.
> - **Módulos críticos** (agent, auth, escrow, fallback): cobertura mínima 90% + cenários E2E obrigatórios.
> - **Gates de merge:** lint + type-check + unit + integração + cobertura ≥ 80% geral / ≥ 90% módulos críticos.
> - **Dados de teste:** factories + seed determinístico + reset após cada suíte E2E. Nunca dados reais de produção.
> - **Fluxos críticos E2E:** isolamento total do Cedente, confirmação de ação financeira, circuit breaker, fallback.
> - **Testes de contrato:** todos os 38 endpoints da API (D16) possuem schema validado com Zod + testes de status code.
> - **Regressão:** suíte de regressão roda em todo PR para `develop` e `main`. Tempo máx: 8 minutos.

---

## 1. Estratégia Geral

O serviço AI-Dani-Cedente é um agente de IA com acesso a dados financeiros sensíveis do Cedente. O perfil de risco é **alto**: falhas de isolamento de dados (`cedenteId` errado) ou de fluxo financeiro (ação executada sem confirmação) têm consequências diretas para usuários reais.

A estratégia de testes reflete esse perfil:

1. **Isolamento de dados do Cedente é inegociável.** Todo test que envolve dados deve verificar que o `cedenteId` correto está sendo usado e que outro Cedente não acessa dados alheios.
2. **Confirmação de ação financeira é testada em todos os níveis.** Aceite de proposta, extensão de Escrow e similares nunca podem ser executados sem o estado `pendingConfirmation` ter sido confirmado.
3. **Circuit breaker e fallback são testados de forma isolada.** Simular degradação de LLM e verificar comportamento esperado.
4. **Cobertura percentual é consequência, não objetivo.** O objetivo é cobrir riscos reais — fluxo feliz, fluxo de erro, edge cases e regressões históricas.

---

## 2. Pirâmide de Testes

```
                    ┌─────────┐
                    │  E2E    │  8% — fluxos críticos completos
                    │  8%     │  Cypress / Playwright (ou supertest E2E)
                   ─┴─────────┴─
              ┌──────────────────────┐
              │   Integração   30%   │  Banco, Redis, RabbitMQ, Langfuse mock
              │   (supertest + Jest) │
             ─┴──────────────────────┴─
        ┌──────────────────────────────────┐
        │     Unitário         60%         │  Services, guards, middleware, tools
        │     (Jest + mocks)               │
       ─┴──────────────────────────────────┴─
   ┌────────────────────────────────────────────┐
   │  Manual exploratório    2%                  │  Novas features, edge cases raros
   └────────────────────────────────────────────┘
```

| Camada | Proporção | Framework | Quando Rodar |
|---|---|---|---|
| **Unitário** | 60% | Jest + ts-jest | Pre-commit (lint-staged) + CI em todo PR |
| **Integração** | 30% | Jest + supertest + testcontainers | CI em todo PR |
| **E2E** | 8% | Jest + supertest (API E2E) | CI em PR para `develop` e `main` |
| **Manual exploratório** | 2% | — | Novas features antes do release |

---

## 3. Testes Unitários

### 3.1 Objetivo e Escopo

Testes unitários verificam a lógica isolada de services, guards, middleware, ferramentas do agente e transformações de dados. Dependências externas (banco, Redis, RabbitMQ, LLM) são sempre mockadas.

### 3.2 Padrão e Ferramentas

| Atributo | Definição |
|---|---|
| **Framework** | Jest + ts-jest |
| **Cobertura mínima geral** | 80% de linhas |
| **Cobertura mínima — módulos críticos** | 90% (agent, auth, escrow, fallback) |
| **Isolamento** | Nenhuma chamada real a banco, Redis, RabbitMQ ou OpenAI |
| **Mocks aceitos** | `jest.mock()`, `jest.fn()`, `jest.spyOn()`. Para NestJS: `@nestjs/testing` TestingModule |
| **Naming convention** | `<NomeDoSujeito>.spec.ts` na mesma pasta do arquivo testado |

### 3.3 Naming de Testes

```
describe('<NomeDaClasse>', () => {
  describe('<nomeDoMetodo>', () => {
    it('deve <comportamento esperado> quando <condição>', () => { ... })
  })
})
```

### 3.4 Quando NÃO Usar Testes Unitários

- Configurações de módulo NestJS (cobertos por integração).
- Mapeamentos de schema do Prisma (cobertos por integração com banco de teste).
- Fluxos completos de autenticação (cobertos por integração).

### 3.5 Exemplo — Teste Unitário de Service

```typescript
// src/modules/agent/agent.service.spec.ts
describe('AgentService', () => {
  describe('handleMessage', () => {
    it('deve retornar estado pendingConfirmation ao processar aceite de proposta', async () => {
      const mockSession = createMockSession({ state: 'analyzing_proposal' });
      const mockLlmResponse = { action: 'accept_proposal', proposal_id: 'PRP-001' };

      jest.spyOn(llmService, 'call').mockResolvedValue(mockLlmResponse);

      const result = await agentService.handleMessage({
        sessionId: 'sess_001',
        message: 'quero aceitar esta proposta',
        cedenteId: 'ced_001',
      });

      expect(result.state).toBe('pendingConfirmation');
      expect(result.pendingAction.type).toBe('accept_proposal');
      expect(result.pendingAction.proposal_id).toBe('PRP-001');
      // Garantir que o endpoint de aceite NÃO foi chamado ainda
      expect(proposalService.accept).not.toHaveBeenCalled();
    });

    it('deve rejeitar mensagem se cedenteId for diferente do contexto', async () => {
      // CedenteIsolationMiddleware deve impedir isso antes de chegar aqui
      // Mas o service também valida como última linha de defesa
      jest.spyOn(cedenteContext, 'getCedenteId').mockReturnValue('ced_001');

      await expect(
        agentService.handleMessage({
          sessionId: 'sess_001',
          message: 'consultar proposta',
          cedenteId: 'ced_999', // ← diferente do contexto
        })
      ).rejects.toThrow(ForbiddenException);
    });
  });
});
```

---

## 4. Testes de Integração

### 4.1 Objetivo e Escopo

Testes de integração verificam que os componentes funcionam corretamente juntos: endpoints HTTP, banco de dados, cache Redis, filas RabbitMQ e workers. Usam instâncias reais de serviços via `testcontainers` (PostgreSQL, Redis, RabbitMQ locais no CI) ou banco de teste.

### 4.2 Setup

```typescript
// test/setup.ts
beforeAll(async () => {
  // Subir containers de teste via testcontainers
  postgresContainer = await new PostgreSqlContainer('supabase/postgres:15.1.0.117').start();
  redisContainer = await new RedisContainer('redis:7.4-alpine').start();
  rabbitmqContainer = await new RabbitMQContainer('rabbitmq:4-management-alpine').start();

  // Aplicar migrations
  await prisma.$executeRaw`...migrations...`;
  // Seed de teste
  await seedTestData();
});

afterEach(async () => {
  // Limpar dados entre testes
  await truncateTestTables();
});

afterAll(async () => {
  await postgresContainer.stop();
  await redisContainer.stop();
  await rabbitmqContainer.stop();
});
```

### 4.3 Categorias de Integração

| Categoria | O que Verifica | Ferramentas |
|---|---|---|
| **Endpoints HTTP** | Status codes, payloads, headers, autenticação | supertest + JWT de teste |
| **Banco de dados** | Queries, RLS, soft delete, constraints | Prisma + PostgreSQL de teste |
| **Cache Redis** | TTL, invalidação, rate limit sliding window | Redis de teste |
| **Fila RabbitMQ** | Publicação, consumo, DLQ, retry | RabbitMQ de teste |
| **Workers** | NotificationWorker, RagWorker processam corretamente | RabbitMQ de teste + mock de provedor |
| **Auth flow** | JWT válido/inválido, RBAC, RLS ativado | JWT RS256 de teste |

### 4.4 Exemplo — Teste de Integração de Endpoint

```typescript
// test/integration/chat.integration.spec.ts
describe('POST /api/v1/chat/sessions/:sessionId/messages', () => {
  it('deve retornar 429 após 30 mensagens na mesma hora', async () => {
    const cedenteJwt = createTestJwt({ cedenteId: 'ced_001', role: 'cedente' });

    // Consumir 30 mensagens
    for (let i = 0; i < 30; i++) {
      await request(app.getHttpServer())
        .post('/api/v1/chat/sessions/sess_001/messages')
        .set('Authorization', `Bearer ${cedenteJwt}`)
        .send({ message: `Mensagem ${i}` })
        .expect(200);
    }

    // 31ª mensagem deve ser bloqueada
    const response = await request(app.getHttpServer())
      .post('/api/v1/chat/sessions/sess_001/messages')
      .set('Authorization', `Bearer ${cedenteJwt}`)
      .send({ message: 'Mensagem 31' })
      .expect(429);

    expect(response.body.error.code).toBe('DCE-CHAT-4290_001');
    expect(response.body.error.user_message).toContain('limite de mensagens');
  });

  it('deve retornar 404 ao acessar sessão de outro Cedente', async () => {
    const outroJwt = createTestJwt({ cedenteId: 'ced_002', role: 'cedente' });

    await request(app.getHttpServer())
      .post('/api/v1/chat/sessions/sess_ced_001/messages')
      .set('Authorization', `Bearer ${outroJwt}`)
      .send({ message: 'Tentativa de acesso cruzado' })
      .expect(404); // 404, não 403 — não revela que a sessão existe
  });
});
```

---

## 5. Testes de Contrato

Os 38 endpoints da API (D16) possuem contratos de schema validados com Zod. Testes de contrato verificam que:

1. Respostas de sucesso correspondem ao schema documentado em D16.
2. Respostas de erro seguem o schema padrão `{ error: { code, message, user_message, correlation_id } }`.
3. Status codes correspondem aos documentados.
4. Headers obrigatórios (`x-correlation-id`, `X-RateLimit-*`) estão presentes.

### 5.1 Contratos por Domínio

| Domínio | Endpoints | Status Codes Testados |
|---|---|---|
| Auth | 2 | 200, 401 |
| Chat | 5 (inclui SSE) | 200, 400, 401, 404, 422, 429, 503 |
| Opportunity | 4 | 200, 400, 401, 403, 404, 409 |
| Proposal | 6 | 200, 400, 401, 404, 409, 422 |
| Dossier | 5 | 200, 400, 401, 404, 422 |
| Escrow | 4 | 200, 400, 401, 404 |
| Simulation | 2 | 200, 400, 401, 404 |
| Admin | 10 | 200, 400, 401, 403, 404 |

### 5.2 Exemplo — Contrato de Schema

```typescript
// test/contracts/proposal.contract.spec.ts
const ProposalSchema = z.object({
  data: z.object({
    id: z.string().uuid(),
    opportunity_id: z.string().uuid(),
    status: z.enum(['PENDENTE', 'ACEITA', 'RECUSADA', 'CONTRAPROPOSTA', 'EXPIRADA']),
    valor: z.number().positive(),
    created_at: z.string().datetime(),
  }),
});

it('GET /api/v1/proposals/:id deve retornar schema correto', async () => {
  const response = await request(app.getHttpServer())
    .get('/api/v1/proposals/prp_001')
    .set('Authorization', `Bearer ${cedenteJwt}`)
    .expect(200);

  const result = ProposalSchema.safeParse(response.body);
  expect(result.success).toBe(true);
});
```

### 5.3 Webhook ZapSign — Contrato de Integração

```typescript
it('POST /api/v1/escrow/webhook deve retornar 400 com HMAC inválido', async () => {
  await request(app.getHttpServer())
    .post('/api/v1/escrow/webhook')
    .set('x-zapsign-hmac', 'hmac_invalido')
    .send({ event: 'document.signed', document_id: 'doc_001' })
    .expect(400);
});

it('POST /api/v1/escrow/webhook deve processar evento com HMAC válido', async () => {
  const payload = { event: 'document.signed', document_id: 'doc_001' };
  const hmac = computeHmac(payload, process.env.ZAPSIGN_WEBHOOK_SECRET);

  await request(app.getHttpServer())
    .post('/api/v1/escrow/webhook')
    .set('x-zapsign-hmac', hmac)
    .send(payload)
    .expect(200);
});
```

---

## 6. Testes E2E

### 6.1 Fluxos Críticos com Cobertura E2E Obrigatória

| Fluxo | Requisitos cobertos | Prioridade |
|---|---|---|
| Isolamento de dados — Cedente não acessa dados de outro | RN-DCE-001, RF-DCE-001 | P0 |
| Confirmação obrigatória antes de aceite de proposta | RF-DCE-018, RF-DCE-019 | P0 |
| Circuit breaker: desligamento automático e reativação | RF-DCE-029, RF-DCE-030 | P0 |
| Fluxo completo de chat: mensagem → agente → resposta SSE | RF-DCE-005, RF-DCE-006 | P0 |
| Rate limit de chat: 30 mensagens/hora por cedente_id | RF-DCE-004 | P1 |
| Notificação proativa de nova proposta | RF-DCE-028 | P1 |
| Análise de proposta com simulação de retorno | RF-DCE-016, RF-DCE-017 | P1 |
| Extensão de Escrow: notificação + silêncio = aprovação automática | RF-DCE-024 | P1 |

### 6.2 Ambiente E2E

| Atributo | Definição |
|---|---|
| **Ambiente** | Containers locais (PostgreSQL, Redis, RabbitMQ) + mock de OpenAI + mock de Langfuse |
| **Autenticação** | JWTs RS256 gerados com chave de teste (não a chave de produção do Supabase) |
| **Seed** | `test/e2e/seeds/` — factories determinísticas com IDs fixos |
| **Reset** | `truncateAllTables()` + `seedE2EBase()` no `beforeEach` de cada teste E2E |
| **Mock LLM** | `jest.mock('@langchain/openai')` com respostas predefinidas por cenário |
| **Tempo máximo** | 8 minutos para toda a suíte E2E |

### 6.3 Exemplo — Teste E2E de Isolamento

```typescript
// test/e2e/isolation.e2e.spec.ts
describe('E2E: Isolamento de dados do Cedente', () => {
  it('Cedente ced_001 não deve ver oportunidades de ced_002', async () => {
    const jwt_ced_001 = createTestJwt({ cedenteId: 'ced_001', role: 'cedente' });

    // Criando oportunidade para ced_002 via seed
    await prisma.opportunity.create({
      data: { id: 'opr_002', cedente_id: 'ced_002', ...opportunityFixture }
    });

    // ced_001 tenta acessar oportunidade de ced_002
    await request(app.getHttpServer())
      .get('/api/v1/opportunities/opr_002')
      .set('Authorization', `Bearer ${jwt_ced_001}`)
      .expect(404); // Não 403 — não revela existência do recurso
  });

  it('Cedente ced_001 não deve ver sessões de chat de ced_002', async () => {
    const jwt_ced_001 = createTestJwt({ cedenteId: 'ced_001', role: 'cedente' });

    const session_ced_002 = await prisma.chatSession.create({
      data: { id: 'sess_ced_002', cedente_id: 'ced_002', ...sessionFixture }
    });

    await request(app.getHttpServer())
      .get(`/api/v1/chat/sessions/${session_ced_002.id}`)
      .set('Authorization', `Bearer ${jwt_ced_001}`)
      .expect(404);
  });
});
```

---

## 7. Cenários por Módulo

### Módulo: Agent (Crítico — cobertura mínima 90%)

| Cenário | Tipo | Prioridade | Requisito | Risco Coberto |
|---|---|---|---|---|
| Transição idle → analyzing_proposal ao receber mensagem sobre proposta | Unitário | P0 | RF-DCE-013 | Máquina de estados incorreta |
| `pendingConfirmation` ativo antes de aceite de proposta | Unitário | P0 | RF-DCE-018 | Ação financeira sem confirmação |
| `cedenteId` de ferramenta extraído do contexto, nunca do LLM input | Unitário | P0 | RN-DCE-001 | Prompt injection — acesso cruzado |
| Timeout de ferramenta após 5s dispara fallback | Unitário | P0 | RF-DCE-030 | Tool hang — Cedente sem resposta |
| Agente retorna para idle após sessão expirar (TTL 1800s) | Unitário | P1 | D19 | Estado corrompido após expiração |
| Confidence score < 80% → takeover para Admin | Unitário | P1 | RF-DCE-031 | Resposta incorreta sem escalation |
| System prompt não expõe dados de outro Cedente | Integração | P0 | RN-DCE-001 | Vazamento de contexto no LLM |
| Agente processa mensagem → tool chamada → resposta SSE | Integração | P0 | RF-DCE-006 | Fluxo completo quebrado |
| Circuit breaker abre após 10% de erros em 15min | Integração | P0 | RF-DCE-029 | Degradação não detectada |
| Fluxo completo: mensagem → confirmação → aceite de proposta | E2E | P0 | RF-DCE-018 | Regressão do fluxo financeiro |
| Após circuit breaker aberto, Admin reativa via POST /admin/fallback/enable | E2E | P0 | RF-DCE-030 | Serviço bloqueado para reativação |

### Módulo: Auth (Crítico — cobertura mínima 90%)

| Cenário | Tipo | Prioridade | Requisito | Risco Coberto |
|---|---|---|---|---|
| JWT válido RS256 → request autorizado | Unitário | P0 | RF-DCE-001 | Acesso negado com JWT correto |
| JWT expirado → 401 | Unitário | P0 | RF-DCE-001 | Token expirado não bloqueado |
| JWT com role `cedente` não acessa rotas Admin | Unitário | P0 | RN-DCE-002 | Escalada de privilégio |
| JWT com role `admin` acessa rotas Admin | Unitário | P0 | RN-DCE-002 | Admin bloqueado em suas rotas |
| CedenteIsolationMiddleware injeta cedente_id no AsyncLocalStorage | Unitário | P0 | RN-DCE-001 | cedenteId não propagado |
| PiiMaskingMiddleware mascara CPF antes do log | Unitário | P0 | LGPD | PII em logs de produção |
| Token sem Bearer prefix → 401 | Integração | P1 | RF-DCE-001 | Formato de token inválido não bloqueado |
| RBAC: cedente não acessa `/admin/*` | Integração | P0 | RN-DCE-002 | Escalada de privilégio em integração |

### Módulo: Chat

| Cenário | Tipo | Prioridade | Requisito | Risco Coberto |
|---|---|---|---|---|
| SSE stream abre e fecha corretamente | Integração | P0 | RF-DCE-006 | Streaming quebrado |
| Rate limit 30 msg/h por cedente_id | Integração | P0 | RF-DCE-004 | Abuso sem bloqueio |
| Mensagem armazenada sem PII do Cedente | Unitário | P0 | LGPD | PII em histórico |
| Histórico limitado às últimas 10 mensagens no contexto LLM | Unitário | P1 | D19 | Contexto excessivo aumentando custo |
| Erro de LLM durante streaming → evento SSE de erro | Unitário | P0 | D20 | Cliente sem feedback de erro |
| GET /sessions retorna apenas sessões do cedente_id autenticado | Integração | P0 | RN-DCE-001 | Listagem cruzada de sessões |

### Módulo: Proposal

| Cenário | Tipo | Prioridade | Requisito | Risco Coberto |
|---|---|---|---|---|
| Aceite de proposta sem confirmação prévia → bloqueado | Integração | P0 | RF-DCE-018 | Aceite não intencional |
| Aceite de proposta com confirmação → cria EscrowTransaction | Integração | P0 | RF-DCE-018, RF-DCE-022 | Escrow não criado após aceite |
| Proposta de outro Cedente → 404 | Integração | P0 | RN-DCE-001 | Acesso cruzado de proposta |
| Simulação de retorno líquido: Repasse − Saldo Devedor | Unitário | P0 | RF-DCE-017 | Cálculo errado de retorno |
| Contraproposta cria nova entrada na tabela proposals | Integração | P1 | RF-DCE-019 | Contraproposta não registrada |

### Módulo: Escrow

| Cenário | Tipo | Prioridade | Requisito | Risco Coberto |
|---|---|---|---|---|
| Alerta proativo 2 dias antes do vencimento publicado na fila | Integração | P0 | RF-DCE-023 | Cedente sem alerta de vencimento |
| Extensão aprovada após silêncio de 24h (job scheduler) | Integração | P0 | RF-DCE-024 | Extensão não aprovada automaticamente |
| Extensão recusada pelo Cedente → Admin alertado | Integração | P0 | RF-DCE-024 | Recusa não notificada ao Admin |
| Webhook ZapSign document.signed → Escrow status → LIBERADO | Integração | P0 | RF-DCE-026 | Liberação não disparada após assinatura |
| HMAC inválido em webhook → 400 sem processamento | Integração | P0 | D18 | Webhook falso processado |

### Módulo: Notification

| Cenário | Tipo | Prioridade | Requisito | Risco Coberto |
|---|---|---|---|---|
| Notificação crítica enviada mesmo com opt-out global ativo | Unitário | P0 | RF-DCE-028 | Cedente não recebe alerta de prazo |
| Notificação não-crítica bloqueada com opt-out ativo | Unitário | P0 | D21 | Envio indesejado / LGPD |
| PII não está no payload da fila RabbitMQ | Unitário | P0 | LGPD | E-mail em fila exposta |
| Retry 3x com backoff → DLQ após 3 falhas | Integração | P0 | D21 | Notificação perdida sem alerta |
| Rate limit de e-mail: 11º e-mail não-crítico bloqueado | Integração | P1 | D21 | Spam ao Cedente |

### Módulo: RAG

| Cenário | Tipo | Prioridade | Requisito | Risco Coberto |
|---|---|---|---|---|
| Ingestão via RabbitMQ cria embeddings no pgvector | Integração | P0 | D14 | Base de conhecimento não atualizada |
| Busca semântica retorna k=5 resultados com score ≥ 0.7 | Unitário | P0 | D19 | Resultados irrelevantes no contexto |
| Ingestão falha → DLQ sem crash do serviço | Integração | P1 | D14 | Crash na ingestão |

---

## 8. Dados de Teste

### 8.1 Factories e Seeds

[DECISÃO AUTÔNOMA]: Factories determinísticas com IDs fixos (UUIDs v4 gerados via seed) — Justificativa: garante reprodutibilidade entre execuções de CI sem dependência de geração aleatória; facilita debugging com IDs rastreáveis | Alternativa descartada: IDs gerados aleatoriamente a cada execução — torna difícil rastrear falhas específicas e viola o isolamento entre suítes.

```typescript
// test/factories/cedente.factory.ts
export const cedenteFactory = {
  ced_001: {
    id: 'ced_001-0000-0000-0000-000000000001',
    user_id: 'usr_001-0000-0000-0000-000000000001',
    kyc_status: 'APROVADO',
    created_at: new Date('2026-01-01T00:00:00Z'),
  },
  ced_002: {
    id: 'ced_002-0000-0000-0000-000000000002',
    user_id: 'usr_002-0000-0000-0000-000000000002',
    kyc_status: 'APROVADO',
    created_at: new Date('2026-01-02T00:00:00Z'),
  },
};

// test/factories/opportunity.factory.ts
export const opportunityFactory = {
  opr_001: {
    id: 'opr_001-0000-0000-0000-000000000001',
    cedente_id: 'ced_001-0000-0000-0000-000000000001',
    status: 'PUBLICADA',
    valor_repasse: 90_000,
    saldo_devedor: 20_000,
    // Nunca dados reais de produção
  },
};
```

### 8.2 JWTs de Teste

```typescript
// test/utils/jwt.ts
export function createTestJwt(payload: { cedenteId: string; role: 'cedente' | 'admin' }) {
  return jwt.sign(
    {
      sub: payload.cedenteId,
      role: payload.role,
      iss: 'test-issuer', // Nunca a chave do Supabase de produção
      exp: Math.floor(Date.now() / 1000) + 3600,
    },
    TEST_JWT_PRIVATE_KEY,
    { algorithm: 'RS256' }
  );
}
```

### 8.3 Regras de Dados de Teste

| Regra | Detalhe |
|---|---|
| **Nunca dados reais** | Proibido importar dump de produção para testes |
| **IDs determinísticos** | UUIDs fixos nas factories para rastreabilidade |
| **Reset entre suítes E2E** | `truncateAllTables()` + `seedE2EBase()` no `beforeEach` |
| **PII mascarada** | Mesmo nos testes, CPF/e-mail de fixtures são fictícios |
| **Isolation** | Cada teste cria seus próprios dados ou usa fixtures isoladas |

---

## 9. Cobertura e Gates

### 9.1 Thresholds de Cobertura

| Camada | Threshold Global | Threshold Módulos Críticos |
|---|---|---|
| Lines | ≥ 80% | ≥ 90% (agent, auth, escrow, fallback) |
| Functions | ≥ 80% | ≥ 90% |
| Branches | ≥ 75% | ≥ 85% |
| Statements | ≥ 80% | ≥ 90% |

Configuração no `jest.config.ts`:
```typescript
coverageThreshold: {
  global: { lines: 80, functions: 80, branches: 75, statements: 80 },
  './src/modules/agent/**/*.ts': { lines: 90, functions: 90 },
  './src/modules/auth/**/*.ts': { lines: 90, functions: 90 },
  './src/modules/escrow/**/*.ts': { lines: 90, functions: 90 },
  './src/modules/fallback/**/*.ts': { lines: 90, functions: 90 },
}
```

### 9.2 Gates de Bloqueio

| Gate | Bloqueia Merge? | Bloqueia Release? |
|---|---|---|
| `pnpm lint` com erros | Sim | Sim |
| `pnpm type-check` com erros | Sim | Sim |
| Qualquer teste unitário falhando | Sim | Sim |
| Qualquer teste de integração falhando | Sim | Sim |
| Cobertura abaixo do threshold | Sim | Sim |
| Qualquer teste E2E de fluxo P0 falhando | Sim | Sim |
| Security scan com vulnerabilidade CRITICAL | Sim | Sim |
| Security scan com vulnerabilidade HIGH | Não (warn) | Sim |

---

## 10. Testes de Regressão

A suíte de regressão garante que fluxos que já funcionaram uma vez não regridam. É composta pelos testes E2E P0 + P1 e roda em todo PR para `develop` e `main`.

### 10.1 Fluxos de Regressão Obrigatórios

1. Isolamento de dados do Cedente (acesso cruzado retorna 404)
2. Fluxo completo chat → agente → resposta SSE
3. Confirmação antes de ação financeira
4. Circuit breaker: abertura e reativação manual
5. Rate limit de chat (30 msg/h)
6. Webhook ZapSign com HMAC válido
7. Notificação crítica não bloqueada por opt-out

### 10.2 Manutenção da Suíte

| Situação | Ação |
|---|---|
| Novo fluxo crítico implementado | Adicionar cenário E2E antes do merge |
| Bug corrigido | Adicionar teste de regressão que reproduz o bug antes do fix |
| Teste obsoleto (feature removida) | Remover teste + comentar no PR com justificativa |
| Teste flaky (falha intermitente) | Marcar com `it.skip` + abrir ticket P1 para investigar |

---

## 11. Testes Não-Funcionais

### 11.1 Testes de Performance (Carga)

[DECISÃO AUTÔNOMA]: k6 para testes de carga — Justificativa: leve, scriptável em JavaScript/TypeScript, integra bem com CI GitHub Actions | Alternativa descartada: JMeter — mais complexo de configurar, não nativo para APIs RESTful modernas.

| Cenário | Configuração | Critério de Aprovação |
|---|---|---|
| Chat message burst | 50 users simultâneos, 60s | p95 ≤ 5.000ms, taxa de erro < 1% |
| RAG search | 20 users simultâneos, 30s | p95 ≤ 500ms |
| Notification delivery | 100 notificações/min | p95 ≤ 2.000ms de enfileiramento |

### 11.2 Testes de Resiliência

| Cenário | Método | Critério |
|---|---|---|
| OpenAI timeout | Mock com delay de 6s | Circuit breaker abre após 10% de timeout em 15min |
| Redis down | Parar container Redis durante teste | Serviço retorna 503 com mensagem amigável |
| RabbitMQ down | Parar container RabbitMQ | Notificações falham silenciosamente — sem crash do serviço |

### 11.3 Testes de Segurança Básica

| Cenário | Ferramenta | Gate |
|---|---|---|
| Dependências com CVE | Snyk / Dependabot | CRITICAL bloqueia merge |
| Injection em parâmetros de query | Testes de integração com payloads maliciosos | 400 retornado, sem query executada |
| Header `x-correlation-id` com payload XSS | Teste de integração | Sanitizado antes de logar |

---

## 12. Integração com CI/CD

| Etapa CI | Suíte | Tempo Máx | Falha Bloqueia? |
|---|---|---|---|
| Pre-commit (local) | Lint + Type-check (lint-staged) | 30s | Sim — commit rejeitado |
| PR check — fase 1 | Lint + Type-check + Unitários | 2min | Sim — merge bloqueado |
| PR check — fase 2 | Integração + Contrato | 4min | Sim — merge bloqueado |
| PR check — fase 3 | E2E + Cobertura | 8min | Sim (fluxos P0) — merge bloqueado |
| Deploy Staging | Regressão completa + Smoke test | 10min | Sim — não promove para staging |
| Deploy Produção | Smoke test pós-deploy | 2min | Sim — rollback automático se falhar |

**Publicação de resultados:**
- Coverage report: publicado como artefato no PR (HTML + JSON).
- Falhas de teste: relatório inline no PR via GitHub Actions summary.
- E2E screenshots/logs: artefatos disponíveis por 7 dias no CI.

---

## 13. Backlog de Pendências

| # | Item | Tipo | Prioridade |
|---|---|---|---|
| P-TST-001 | Configurar testcontainers no CI (Docker-in-Docker no GitHub Actions) para testes de integração com PostgreSQL/Redis/RabbitMQ reais | [SEÇÃO PENDENTE] — requer configuração de infra CI | Alta |
| P-TST-002 | Definir estratégia de mock para OpenAI em integração (nock vs. msw vs. mock de classe) | [DECISÃO AUTÔNOMA: jest.mock de classe] — confirmar antes da implementação | Média |
| P-TST-003 | Testes de carga k6 — configurar ambiente de staging com dados representativos | [SEÇÃO PENDENTE] — depende do ambiente de staging (D24) | Média |
| P-TST-004 | Contratos de schema para webhooks ZapSign — validar contra payloads reais da API ZapSign | [SEÇÃO PENDENTE] — aguarda documentação da API ZapSign | Média |
| P-TST-005 | Implementar mutation testing (Stryker) para módulos críticos — medir qualidade dos testes além da cobertura | [DECISÃO AUTÔNOMA: adiar para pós go-live] — overhead de implementação alto para fase inicial | Baixa |

---

## 14. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — pirâmide 60/30/8/2. 11 módulos com cenários de teste. Thresholds: 80% global / 90% módulos críticos. Factories determinísticas. 7 fluxos de regressão. Testes não-funcionais: k6 + resiliência + segurança. CI gates completos. |
