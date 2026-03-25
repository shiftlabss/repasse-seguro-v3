# 27 - Plano de Testes

## Cabeçalho

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
| --- | --- | --- | --- | --- |
| 27 - Plano de Testes | v1.0 | 2026-03-22 | Claude Code Desktop (ShiftLabs Pipeline v9.5) | Aprovado |

---

## TL;DR

> 📌 **Estratégia de testes do Repasse AI:**
>
> - **Pirâmide:** 70% unitários / 25% integração / 5% E2E — sem testes manuais rotineiros.
> - **Ferramenta única:** Jest + Supertest (NestJS); sem Playwright/Cypress (backend puro, PG-03).
> - **Gates de merge:** lint + typecheck + testes unitários passando; cobertura mínima 80% em módulos críticos (ai, chat, whatsapp).
> - **Fluxos críticos E2E:** mensagem Webchat → agente IA → resposta SSE; WhatsApp OTP → binding; takeover Admin.
> - **Dados de teste:** factories TypeScript por entidade; seed isolado por teste; banco de teste dedicado (Docker PostgreSQL).
> - **Módulos cobertos:** ai, chat, calculator, notification, supervision, whatsapp, common (guards, filters, interceptors).

---

## 1. Estratégia Geral

O Repasse AI é um backend puro (PG-03) — a estratégia de testes é inteiramente focada em NestJS. Não há testes de frontend, React Testing Library, Playwright ou Cypress. O perfil de risco concentra-se em três áreas: (1) pipeline do agente IA (LangGraph.js + OpenAI), onde erros têm impacto direto no negócio; (2) isolamento de tenant via RLS, onde falha expõe dados de outro cessionário; (3) integração WhatsApp, onde falha interrompe comunicação com o lead.

**Princípios:**
- Teste existe para cobrir risco real, não para atingir percentual.
- Fluxos críticos têm cobertura E2E obrigatória; fluxos secundários têm cobertura de integração.
- Mocks são permitidos em unitários; integrações externas (OpenAI, EvolutionAPI) são mockadas em integração e E2E.
- Todo teste deve ser determinístico — sem `sleep`, sem dependência de ordem de execução.

---

## 2. Pirâmide de Testes

```
          ┌───────────────────┐
          │      E2E (5%)     │  ← 10-15 fluxos críticos, banco real Docker
          ├───────────────────┤
          │  Integração (25%) │  ← endpoints, fila, cache, banco, guards
          ├───────────────────┤
          │  Unitários (70%)  │  ← services, repositories, utils, error classes
          └───────────────────┘
```

| Camada | Proporção | Ferramentas | Velocidade | Quando Roda |
| --- | --- | --- | --- | --- |
| Unitários | 70% | Jest + mocks | < 30s | A cada commit (pre-push hook + CI) |
| Integração | 25% | Jest + Supertest + Docker PostgreSQL/Redis | ~2 min | A cada PR (CI) |
| E2E | 5% | Jest + Supertest + Docker full stack | ~5 min | A cada merge em `main` + release |

---

## 3. Testes Unitários

### 3.1 Padrão e Ferramentas

- **Framework:** Jest 29+ com `ts-jest`
- **Localização:** `*.spec.ts` na mesma pasta do arquivo testado
- **Naming:** `describe('[NomeDoServico]') → it('deve [comportamento esperado] quando [condição]')`
- **Isolamento:** todo serviço é testado com dependências mockadas via `jest.fn()`
- **Mocks aceitos:** qualquer dependência externa (PrismaService, RabbitMQService, OpenAI SDK)
- **Mocks proibidos:** lógica interna do próprio módulo (circular mocking)

### 3.2 Cobertura Mínima por Módulo

| Módulo | Coverage mínima (statements) | Justificativa |
| --- | --- | --- |
| `ai` | 85% | Pipeline LLM é o core do produto |
| `chat` | 80% | Fluxo principal de interação |
| `whatsapp` | 80% | Integração crítica de canal |
| `supervision` | 75% | Mutex de takeover — risco de concorrência |
| `notification` | 75% | Entrega assíncrona |
| `calculator` | 75% | Cálculos financeiros |
| `common` (guards, filters) | 90% | Segurança transversal |

### 3.3 Exemplo de Cenário Unitário

```typescript
// apps/ai/src/ai/services/__tests__/ai.service.spec.ts
describe('AiService', () => {
  describe('processMessage', () => {
    it('deve retornar resposta do agente quando LLM responde com sucesso', async () => {
      // Arrange
      mockLangGraphRunner.invoke.mockResolvedValueOnce({
        messages: [{ role: 'assistant', content: 'O ROI do imóvel é 12%.' }],
        stream_done: true,
      });

      // Act
      const result = await aiService.processMessage({
        conversationId: 'conv_test',
        message: 'Qual o ROI deste imóvel?',
        cessionarioId: 'ces_test',
        correlationId: 'req_test',
      });

      // Assert
      expect(result.content).toBe('O ROI do imóvel é 12%.');
      expect(mockLangGraphRunner.invoke).toHaveBeenCalledTimes(1);
    });

    it('deve lançar LlmUnavailableError quando OpenAI retorna 503 após 3 tentativas', async () => {
      mockLangGraphRunner.invoke.mockRejectedValue(new Error('Service Unavailable'));

      await expect(
        aiService.processMessage({ conversationId: 'conv_test', message: 'Teste', cessionarioId: 'ces_test', correlationId: 'req_test' }),
      ).rejects.toBeInstanceOf(LlmUnavailableError);
    });

    it('deve retornar cache hit sem invocar LangGraph quando semantic cache encontrar similaridade > 0.92', async () => {
      mockSemanticCacheService.get.mockResolvedValueOnce({ hit: true, response: 'Resposta cacheada.' });

      const result = await aiService.processMessage({ conversationId: 'conv_test', message: 'Qual o ROI?', cessionarioId: 'ces_test', correlationId: 'req_test' });

      expect(result.content).toBe('Resposta cacheada.');
      expect(mockLangGraphRunner.invoke).not.toHaveBeenCalled();
    });
  });
});
```

---

## 4. Testes de Integração

### 4.1 Setup

- **Banco:** PostgreSQL 17 Docker (`docker-compose.test.yml`) — schema isolado por suite via `beforeAll` migration + `afterAll` rollback
- **Redis:** Redis 7.4 Docker — flushado via `beforeEach`
- **RabbitMQ:** mockado (não Docker) — `jest.mock('@common/rabbitmq')` nos testes de integração
- **OpenAI:** mockado via `jest.mock('@langchain/openai')` — nunca chamada real em CI
- **EvolutionAPI:** mockado — `nock` ou `jest.mock`

### 4.2 Testes de Endpoints (Supertest)

Cobrir para cada endpoint da Doc 16:
- Status code esperado (200, 201, 422, 401, 403, 404, 429)
- Schema da resposta (campo obrigatório `error_code` em erros)
- Header `x-correlation-id` presente na resposta
- Comportamento com JWT inválido → 401
- Comportamento com role insuficiente → 403
- Comportamento com tenant errado → 403 `TENANT_MISMATCH`

### 4.3 Exemplo de Cenário de Integração

```typescript
// apps/ai/src/chat/controllers/__tests__/chat.controller.integration.spec.ts
describe('POST /repasse-ai/v1/chat/conversations/:id/messages', () => {
  it('deve retornar 422 com error_code VALIDATION_ERROR quando message está vazio', async () => {
    const response = await request(app.getHttpServer())
      .post(`/repasse-ai/v1/chat/conversations/${testConversationId}/messages`)
      .set('Authorization', `Bearer ${validJwt}`)
      .send({ message: '' });

    expect(response.status).toBe(422);
    expect(response.body.error.error_code).toBe('VALIDATION_ERROR');
    expect(response.body.error.correlation_id).toBeDefined();
  });

  it('deve retornar 403 com error_code TENANT_MISMATCH quando cessionario_id do JWT não corresponde à conversa', async () => {
    const response = await request(app.getHttpServer())
      .post(`/repasse-ai/v1/chat/conversations/${outroTenantConversationId}/messages`)
      .set('Authorization', `Bearer ${validJwtCes1}`)
      .send({ message: 'Teste de cross-tenant' });

    expect(response.status).toBe(403);
    expect(response.body.error.error_code).toBe('TENANT_MISMATCH');
  });
});
```

### 4.4 Testes de Cache Redis

| Cenário | Setup | Verificação |
| --- | --- | --- |
| Rate limit excedido | Simular 31 mensagens em 1h para mesmo user_id | Resposta 429 com `error_code RATE_LIMIT_EXCEEDED` |
| OTP expira | Criar OTP no Redis com TTL 1s, aguardar 2s | `WHATSAPP_OTP_EXPIRED` ao tentar validar |
| Semantic cache hit | Pre-popular chave de cache com embedding | LangGraph não é invocado; resposta em < 50ms |

### 4.5 Testes de Banco (RLS)

```typescript
it('deve lançar erro ao query sem withTenant() em módulo de domínio', async () => {
  // Tentar acessar tabela com RLS sem SET LOCAL app.cessionario_id
  const query = prismaService.$queryRaw`SELECT * FROM conversations`;
  await expect(query).rejects.toThrow(); // RLS bloqueia
});

it('withTenant() deve isolar dados corretamente entre cessionários', async () => {
  const conv1 = await prismaService.withTenant('ces_1').conversation.findMany();
  const conv2 = await prismaService.withTenant('ces_2').conversation.findMany();
  expect(conv1.every(c => c.cessionario_id === 'ces_1')).toBe(true);
  expect(conv2.every(c => c.cessionario_id === 'ces_2')).toBe(true);
});
```

---

## 5. Testes de Contrato

### 5.1 Contratos de API Internos

Para cada endpoint da Doc 16, manter um contrato de schema que valida:
- Response body segue o schema JSON documentado
- Campos obrigatórios (`error_code`, `correlation_id`, `timestamp`) presentes em respostas de erro
- Paginação (`page`, `per_page`, `meta`) presente em endpoints de listagem
- SSE stream termina com `data: [DONE]`

### 5.2 Contrato com EvolutionAPI

O webhook recebido da EvolutionAPI deve ser validado contra o schema de payload documentado:
- `event`, `instance`, `data.key.remoteJid`, `data.message.conversation`
- HMAC-SHA256 válido (testes com payload intencionalmente corrompido → 401)

### 5.3 Contrato com OpenAI

Mockado via `jest.fn()` — o contrato é validado pelo Langfuse em produção (evals). Em testes: verificar que o prompt enviado ao mock contém `cessionario_id`, `conversation_id` e não contém PII.

---

## 6. Testes E2E

### 6.1 Ambiente

- Docker Compose com PostgreSQL 17 + pgvector, Redis 7.4, RabbitMQ 4
- OpenAI mockado via `nock` interceptor
- EvolutionAPI mockado via `nock` interceptor
- `JWT_DEV_MODE=true` para autenticação sem Supabase

### 6.2 Fluxos Críticos E2E (obrigatórios)

| ID | Fluxo | Módulos Envolvidos | Critério de Aprovação |
| --- | --- | --- | --- |
| E2E-001 | Mensagem Webchat → Agente IA → Resposta SSE | chat, ai, common | SSE retorna `{"type":"done"}` com `content` não vazio em < 10s |
| E2E-002 | WhatsApp OTP → Binding → ACTIVE | whatsapp | `WhatsAppBinding.status = ACTIVE` após OTP válido |
| E2E-003 | Admin solicita Takeover → Agente para → Supervisor responde | supervision, chat, ai | `Conversation.mode = HUMAN`, SSE do supervisor funcionando |
| E2E-004 | Isolamento de tenant: cessionário A não acessa dados do B | chat, common | 403 `TENANT_MISMATCH` em todos os endpoints testados |
| E2E-005 | Rate limit: 31ª mensagem em 1h retorna 429 | chat, common | 429 com `error_code RATE_LIMIT_EXCEEDED` e header `Retry-After` |
| E2E-006 | Circuit breaker: 5 falhas LLM → estado OPEN → rejeição imediata | ai | 503 imediato (< 50ms) na 6ª requisição; circuit breaker OPEN |
| E2E-007 | Notificação push após agente responder | chat, ai, notification | `NotificationLog.event = delivered` após processamento da fila |
| E2E-008 | Webhook EvolutionAPI com HMAC inválido → rejeição | whatsapp | 401 sem processar mensagem |

### 6.3 Exemplo de Cenário E2E

```typescript
// apps/ai/src/e2e/__tests__/chat-agent.e2e.spec.ts
describe('E2E-001: Mensagem Webchat → Agente IA → Resposta SSE', () => {
  beforeAll(async () => {
    await seedDatabase(prisma, { cessionario: testCessionario, conversation: testConversation });
    nock('https://api.openai.com').post('/v1/chat/completions').reply(200, mockOpenAIResponse);
  });

  afterAll(async () => clearDatabase(prisma));

  it('deve retornar stream SSE com resposta do agente em < 10 segundos', async () => {
    const chunks: string[] = [];
    const startTime = Date.now();

    const response = await request(app.getHttpServer())
      .post(`/repasse-ai/v1/chat/conversations/${testConversation.id}/messages`)
      .set('Authorization', `Bearer ${testJwt}`)
      .send({ message: 'Qual o ROI deste imóvel?' })
      .buffer(false);

    response.on('data', (chunk: Buffer) => chunks.push(chunk.toString()));
    await new Promise((resolve) => response.on('end', resolve));

    const elapsed = Date.now() - startTime;
    const fullResponse = chunks.join('');

    expect(elapsed).toBeLessThan(10000);
    expect(fullResponse).toContain('"type":"done"');
    expect(fullResponse).toContain('data: [DONE]');
  });
});
```

---

## 7. Cenários por Módulo

### 7.1 Módulo `ai`

| Cenário | Tipo | Prioridade | Risco Coberto |
| --- | --- | --- | --- |
| processMessage retorna resposta correta | Unit | P0 | Core do produto |
| processMessage com cache semântico hit (> 0.92) | Unit | P0 | Custo OpenAI |
| processMessage com cache miss invoca LangGraph | Unit | P0 | Core do produto |
| LLM 503 lança LlmUnavailableError após 3 tentativas | Unit | P0 | Resiliência |
| Circuit breaker abre após 5 falhas consecutivas | Unit | P0 | Resiliência |
| Circuit breaker em OPEN rejeita imediatamente (< 50ms) | Integration | P0 | Resiliência |
| RAG retrieval retorna contexto com threshold 0.78 | Unit | P1 | Qualidade de resposta |
| RAG retrieval fallback com threshold 0.60 quando 0.78 falha | Unit | P1 | Qualidade de resposta |
| Guardrail bloqueia conteúdo de outro tenant | Unit | P0 | Segurança |
| Tool calculate_delta retorna delta correto | Unit | P1 | Cálculo financeiro |
| Tool calculate_roi retorna ROI correto | Unit | P1 | Cálculo financeiro |

### 7.2 Módulo `chat`

| Cenário | Tipo | Prioridade | Risco Coberto |
| --- | --- | --- | --- |
| POST /messages cria mensagem e retorna 201 | Integration | P0 | Core do produto |
| POST /messages com JWT inválido retorna 401 | Integration | P0 | Segurança |
| POST /messages com message vazio retorna 422 | Integration | P0 | Validação |
| POST /messages cross-tenant retorna 403 TENANT_MISMATCH | E2E | P0 | Isolamento |
| GET /conversations lista apenas do tenant correto | Integration | P0 | Isolamento |
| SSE stream entrega chunks em ordem correta | E2E | P0 | Consistência |
| Rate limit: 31ª mensagem em 1h retorna 429 | E2E | P1 | Proteção de abuse |
| Conversa em modo HUMAN rejeita mensagens do agente | Unit | P0 | Takeover |

### 7.3 Módulo `whatsapp`

| Cenário | Tipo | Prioridade | Risco Coberto |
| --- | --- | --- | --- |
| POST /whatsapp/otp envia OTP via EvolutionAPI | Integration | P0 | OTP flow |
| POST /whatsapp/otp/verify com código correto → ACTIVE | E2E | P0 | Binding |
| POST /whatsapp/otp/verify com OTP expirado → 422 | Integration | P0 | Segurança OTP |
| POST /whatsapp/otp/verify com 4ª tentativa → 429 | Integration | P0 | Proteção brute force |
| Webhook com HMAC válido processa mensagem | Integration | P0 | Integridade webhook |
| Webhook com HMAC inválido retorna 401 | Integration | P0 | Segurança webhook |
| Binding inativo não recebe mensagens do agente | Unit | P0 | Estado de binding |

### 7.4 Módulo `supervision`

| Cenário | Tipo | Prioridade | Risco Coberto |
| --- | --- | --- | --- |
| Admin solicita takeover → Conversation.mode = HUMAN | E2E | P0 | Takeover flow |
| Dois admins tentam takeover simultaneamente → mutex garante exclusividade | Integration | P0 | Concorrência |
| Admin libera takeover → modo volta a AI | E2E | P1 | Retorno ao agente |
| Não-admin tenta takeover → 403 | Integration | P0 | Autorização |

### 7.5 Módulo `notification`

| Cenário | Tipo | Prioridade | Risco Coberto |
| --- | --- | --- | --- |
| Notificação crítica enviada mesmo com opt-out configurado | Unit | P0 | Compliance |
| Notificação não-crítica ignorada com opt-out | Unit | P0 | Preferências |
| Template interpolado corretamente com variáveis | Unit | P1 | Conteúdo |
| DLQ atingida gera tracking failed | Integration | P1 | Rastreabilidade |
| Notificação publicada na fila correta por canal | Unit | P0 | Roteamento |

### 7.6 Módulo `common` (Guards, Filters, Interceptors)

| Cenário | Tipo | Prioridade | Risco Coberto |
| --- | --- | --- | --- |
| JwtAuthGuard rejeita JWT expirado | Unit | P0 | Autenticação |
| JwtAuthGuard aceita JWT_DEV_MODE em dev | Unit | P0 | Ambiente dev |
| AllExceptionsFilter não expõe stack trace ao cliente | Unit | P0 | Segurança |
| AllExceptionsFilter retorna INTERNAL_ERROR para exceções genéricas | Unit | P0 | Error handling |
| PiiMaskingInterceptor remove JWT do log | Unit | P0 | LGPD |
| CorrelationIdMiddleware propaga correlation_id | Unit | P1 | Rastreabilidade |

---

## 8. Dados de Teste

### 8.1 Factories

```typescript
// apps/ai/src/test/factories/cessionario.factory.ts
export const cessionarioFactory = (overrides?: Partial<Cessionario>): Cessionario => ({
  id: `ces_test_${nanoid(8)}`,
  name: 'Cessionário Teste',
  email: 'test@repasseseguro.com.br',
  role: 'CESSIONARIO',
  created_at: new Date(),
  ...overrides,
});

// apps/ai/src/test/factories/conversation.factory.ts
export const conversationFactory = (cessionarioId: string, overrides?: Partial<Conversation>): Conversation => ({
  id: `conv_test_${nanoid(8)}`,
  cessionario_id: cessionarioId,
  mode: 'AI',
  status: 'ACTIVE',
  created_at: new Date(),
  ...overrides,
});
```

### 8.2 Seeds de Teste

```typescript
// apps/ai/src/test/seeds/seed-test-db.ts
export async function seedDatabase(prisma: PrismaClient, opts: SeedOptions) {
  const cessionario = await prisma.cessionario.create({ data: opts.cessionario });
  const conversation = await prisma.conversation.create({ data: { ...opts.conversation, cessionario_id: cessionario.id } });
  return { cessionario, conversation };
}

export async function clearDatabase(prisma: PrismaClient) {
  // Limpar em ordem inversa de FK
  await prisma.message.deleteMany();
  await prisma.conversation.deleteMany();
  await prisma.cessionario.deleteMany();
}
```

### 8.3 Dados Mascarados

- Nenhum telefone real, CPF, CNPJ ou email de usuário real em seeds de teste.
- Usar formato padronizado: `test_user_${nanoid}@test.repasseseguro.com.br`, `+55859000${nanoid(4)}`.
- Seeds de produção nunca importados para testes.

---

## 9. Cobertura e Gates

### 9.1 Thresholds de Cobertura (Jest --coverage)

```json
// jest.config.ts
coverageThreshold: {
  global: {
    statements: 75,
    branches: 70,
    functions: 75,
    lines: 75,
  },
  "./apps/ai/src/ai/": { statements: 85 },
  "./apps/ai/src/chat/": { statements: 80 },
  "./apps/ai/src/whatsapp/": { statements: 80 },
  "./apps/ai/src/common/": { statements: 90 },
}
```

### 9.2 Gates de Merge (PR → main)

| Gate | Threshold | Bloqueia Merge? |
| --- | --- | --- |
| Lint (`pnpm lint`) | 0 erros | Sim |
| Typecheck (`pnpm typecheck`) | 0 erros | Sim |
| Testes unitários (`pnpm test`) | 100% passing | Sim |
| Cobertura global | ≥ 75% statements | Sim |
| Cobertura módulo `ai` | ≥ 85% statements | Sim |
| Cobertura módulo `common` | ≥ 90% statements | Sim |
| Build (`pnpm build`) | Sem erros | Sim |

### 9.3 Gates de Release (merge → deploy produção)

| Gate | Threshold | Bloqueia Deploy? |
| --- | --- | --- |
| Testes E2E (`pnpm test:e2e`) | 100% passing | Sim |
| Testes de integração | 100% passing | Sim |
| `GET /health` pós-deploy | 200 OK | Sim |
| `http_error_rate` 5 min pós-deploy | < 1% | Sim (rollback automático se > 10%) |

---

## 10. Testes de Regressão

### 10.1 Quando Rodar

- A cada merge em `main` (CI automático)
- Antes de todo go-live (obrigatório)
- Após qualquer mudança em `common/guards`, `common/filters`, `common/middleware`

### 10.2 Suíte de Regressão Core

Fluxos E2E-001 a E2E-008 (seção 6.2) são a suíte de regressão imutável. Todo bug corrigido em produção ganha um teste antes do fix ser mergeado.

### 10.3 Remoção de Teste Obsoleto

Teste só pode ser removido com PR dedicado, justificativa no PR description (`[OBSOLETO]: motivo`) e aprovação do Tech Lead. Nunca remover teste que cobre fluxo crítico sem substituto equivalente.

---

## 11. Testes Não-Funcionais

### 11.1 Performance (Manual — pré-go-live)

| Teste | Ferramenta | Critério | Frequência |
| --- | --- | --- | --- |
| Latência p95 processMessage | k6 ou Artillery | p95 < 8000ms (10 usuários simultâneos) | Pré-go-live |
| Throughput de mensagens | k6 | 100 req/min sem degradação | Pré-go-live |
| Conexões SSE simultâneas | k6 | 200 conexões sem memory leak | Pré-go-live |

### 11.2 Segurança Básica (Automatizado no CI)

| Teste | Ferramenta | Critério |
| --- | --- | --- |
| Auditoria de dependências | `pnpm audit --audit-level high` | 0 vulnerabilidades HIGH/CRITICAL |
| Verificação de PII em logs | Teste unitário `PiiMaskingInterceptor` | JWT/OTP/telefone não aparecem |
| Cross-tenant em todos os endpoints | Testes de integração por módulo | 403 `TENANT_MISMATCH` em todos |

[DECISÃO AUTÔNOMA] Testes de carga não incluídos no CI regular: execução em CI aumentaria tempo de pipeline para > 10 min sem benefício em cada PR. Alternativa (incluir no CI) descartada por custo de tempo. Critério: rodar em pré-go-live é suficiente para o perfil de uso atual.

---

## 12. Integração com CI/CD

| Suíte | Etapa CI | Tempo Estimado | Falha Bloqueia |
| --- | --- | --- | --- |
| Lint + Typecheck | PR opened/updated | ~1 min | Merge em `main` |
| Testes unitários + coverage | PR opened/updated | ~2 min | Merge em `main` |
| Testes de integração | PR opened/updated | ~3 min | Merge em `main` |
| Build | PR opened/updated | ~3 min | Merge em `main` |
| Testes E2E | Merge em `main` | ~5 min | Deploy em Railway |
| Security audit | PR opened/updated | ~30s | Merge em `main` |

**Total CI por PR:** ~9 min. **Total antes do deploy:** ~14 min.

---

## 13. Backlog de Pendências

| ID | Descrição | Prioridade | Observação |
| --- | --- | --- | --- |
| TEST-001 | Configurar banco de teste dedicado no `docker-compose.test.yml` | Alta | Necessário antes de escrever primeiros testes de integração |
| TEST-002 | Criar `jest.config.ts` com thresholds de cobertura por módulo (seção 9.1) | Alta | Parte do setup inicial |
| TEST-003 | Avaliar uso de `testcontainers` para PostgreSQL + Redis no CI sem Docker Compose manual | Média | Alternativa mais portável para CI |
| TEST-004 | Definir estratégia de mocking para Langfuse SDK (não deve chamar Langfuse Cloud em testes) | Alta | Evitar custo e latência em CI |

> **Decisões Autônomas Tomadas Neste Documento:**
>
> 1. **[DECISÃO AUTÔNOMA] Sem Playwright/Cypress:** alternativa (E2E com browser) descartada por PG-03 (backend puro). Critério: produto não tem frontend próprio.
> 2. **[DECISÃO AUTÔNOMA] RabbitMQ mockado em testes de integração:** alternativa (RabbitMQ Docker real) descartada por complexidade de setup e flakiness em CI. Critério: comportamento da fila é validado em E2E com Docker full stack.
> 3. **[DECISÃO AUTÔNOMA] Testes de carga apenas em pré-go-live:** alternativa (k6 no CI a cada PR) descartada por overhead de tempo. Critério: YAGNI + tempo de CI aceitável.

---

*Próximo documento do pipeline: D24 — Deploy, CI/CD e Versionamento.*
