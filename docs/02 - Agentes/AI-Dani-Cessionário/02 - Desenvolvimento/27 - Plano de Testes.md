# 27 - Plano de Testes
## AI-Dani-Cessionário — Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Produto** | AI-Dani-Cessionário |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Status** | Aprovado — pronto para execução |
| **Bloco** | 6 — Qualidade |

---

> 📌 **TL;DR — Estratégia de Testes**
>
> O AI-Dani-Cessionário opera sobre três contratos críticos que não admitem falha: **isolamento de dados por cessionário** (`cessionario_id` em toda query), **cálculo determinístico de comissão** (`20% × Δ`) e **autenticação via OTP WhatsApp com rate limit**. O plano cobre quatro camadas (unitário → integração → contrato → E2E) com cobertura mínima de 80% em unitário, 70% em integração e 100% nos fluxos críticos de segurança e negócio. Ferramentas: Vitest (unitário + integração), Playwright (E2E), Pact (contrato). Gates: qualquer falha em teste de isolamento, cálculo ou autenticação bloqueia merge imediatamente — sem exceção.

---

## 1. Estratégia Geral

### 1.1 Objetivo do Plano

O plano de testes do AI-Dani-Cessionário traduz os requisitos e contratos do produto em cenários verificáveis, rastreáveis e automatizados. Qualidade aqui não é métrica de linha — é garantia de que:

- Nenhum cessionário acessa dados de outro (isolamento por `cessionario_id`).
- O cálculo de comissão produz sempre o resultado correto para qualquer combinação de Δ positivo, negativo ou zero.
- A autenticação via OTP responde com segurança a tentativas de abuso.
- O agente de IA opera dentro dos guardrails definidos (recusa, fallback, rate limit).

### 1.2 Perfil de Risco do Produto

| **Área de Risco** | **Criticidade** | **Risco Principal** | **Cobertura Exigida** |
|---|---|---|---|
| Isolamento `cessionario_id` | P0 — Crítico | Vazamento de dados entre cessionários | 100% (unitário + integração + E2E) |
| Cálculo de comissão | P0 — Crítico | Resultado financeiro incorreto | 100% (unitário + integração) |
| OTP / Rate Limit / Hard Block | P0 — Crítico | Acesso não autorizado ou bloqueio indevido | 100% (unitário + integração) |
| SSE Streaming (respostas do agente) | P1 — Alto | Perda de chunks, conexão pendurada | ≥80% (integração + E2E) |
| RabbitMQ — filas de notificação | P1 — Alto | Perda silenciosa de mensagem | ≥80% (integração) |
| Langfuse tracing | P2 — Médio | Dado sensível em trace | ≥70% (integração) |
| Frontend — 4 estados UI | P2 — Médio | Estado Error ou Empty omitido | ≥70% (E2E / visual) |

### 1.3 Escopo

- **Dentro do escopo:** `AgenteModule`, `CalculadoraModule`, `OportunidadeModule`, `CessionarioModule`, `AlertaModule`, `WhatsappModule`, `AuthModule`; frontend `features/dani-chat/`, `features/dani-dashboard/`, `features/dani-whatsapp/`; 18 endpoints da API; integrações externas via mock (OpenAI, Supabase, EvolutionAPI, Redis, Langfuse).
- **Fora do escopo:** EvolutionAPI em modo real (sandbox indisponível na Fase 1); React Native (Mobile Fase 2); testes de penetração profundo (responsabilidade de equipe especializada).

### 1.4 Relação com Merge e Release

```
feature/* → PR → [lint + unitário + integração + contrato] → merge develop
develop → PR → [suite completa + E2E] → merge main
main → release → [regressão completa + não-funcional] → go-live
```

---

## 2. Pirâmide de Testes

```
          /\
         /E2E\          ≈10% — fluxos críticos completos (Playwright)
        /------\
       /Contrato\       ≈10% — schemas API e integrações externas (Pact)
      /----------\
     / Integração \     ≈25% — endpoints, banco, Redis, RabbitMQ (Vitest + Supertest)
    /--------------\
   /   Unitários    \   ≈55% — lógica pura, guards, calculadora, formatters (Vitest)
  /------------------\
```

**Justificativa por camada:**

| **Camada** | **Proporção** | **Justificativa** |
|---|---|---|
| Unitário | 55% | Calculadora, guards, formatters têm lógica determinística — cobertura barata e rápida de feedback |
| Integração | 25% | Endpoints com banco real (Supabase local), Redis real e RabbitMQ real revelam contratos e race conditions |
| Contrato | 10% | OpenAI e EvolutionAPI têm schemas que podem mudar — contratos detectam breaking changes sem E2E |
| E2E | 10% | Fluxos de negócio completos: chat → cálculo → alerta; autenticação ponta a ponta |
| Manual Exploratório | Ad-hoc | Novos fluxos de agente, comportamentos de LLM, UX de acessibilidade |

---

## 3. Testes Unitários

### 3.1 Padrão e Ferramentas

- **Framework:** Vitest (`vitest@^1.x`) — compatível com o monorepo pnpm + Turborepo.
- **Ambiente:** `jsdom` para componentes React; `node` para NestJS/TypeScript.
- **Cobertura:** `@vitest/coverage-v8` com relatório LCOV para CI.
- **Naming convention:** `[arquivo].spec.ts` no mesmo diretório do arquivo testado.
- **Pattern:** AAA (Arrange → Act → Assert). Um `describe` por módulo, um `it` por cenário.

### 3.2 Cobertura Mínima

| **Módulo** | **Cobertura Mínima** | **Justificativa** |
|---|---|---|
| `CalculadoraService` | 100% | Lógica financeira — toda ramificação Δ>0, Δ≤0, Δ=0 exige cobertura total |
| `CessionarioOwnerGuard` | 100% | Controle de acesso — toda ramificação deve ser coberta |
| `OtpService` | 100% | Segurança — geração, hash, verificação, contador de falhas |
| `RateLimitService` | 100% | Proteção contra abuso — janela deslizante, hard block |
| `AgenteService` (lógica pura) | ≥80% | Decisões de roteamento, confidence_score, fallback |
| Formatters / Parsers | ≥80% | Funções puras sem efeito colateral |
| Componentes React (lógica) | ≥70% | Hooks, estado, condições de render |

### 3.3 Isolamento e Mocks Permitidos

- **Permitido:** mock de `PrismaClient`, `Redis`, `OpenAI`, `EvolutionAPI`, `Langfuse`, `RabbitMQ`.
- **Proibido:** mock de `CalculadoraService` em testes do próprio `CalculadoraService` — a lógica deve ser testada com implementação real.
- **Proibido:** mock de `CessionarioOwnerGuard` em testes de integração de endpoints — o guard deve executar real.

### 3.4 Quando NÃO usar teste unitário

- Fluxos que envolvem banco real (usar integração).
- Comportamento de LLM (não determinístico — usar asserção de schema e guardrails).
- Fluxos que dependem de SSE real (usar integração ou E2E).

### 3.5 Exemplo de Cenário Unitário

```typescript
// apps/api/src/calculadora/calculadora.service.spec.ts

describe('CalculadoraService', () => {
  describe('calcularComissao', () => {
    it('retorna 20% do Delta quando Delta > 0', () => {
      // Arrange
      const input = {
        tabelaAtual: 85000,
        tabelaContrato: 70000,
        valorPagoCedente: 65000,
      };
      // Delta = 85000 - 70000 = 15000
      // Comissão = 20% × 15000 = 3000

      // Act
      const resultado = calculadoraService.calcularComissao(input);

      // Assert
      expect(resultado.delta).toBe(15000);
      expect(resultado.comissao).toBe(3000);
      expect(resultado.baseCalculo).toBe('delta');
    });

    it('retorna 20% do valorPagoCedente quando Delta <= 0', () => {
      // Arrange
      const input = {
        tabelaAtual: 60000,
        tabelaContrato: 70000,
        valorPagoCedente: 65000,
      };
      // Delta = 60000 - 70000 = -10000 (≤0)
      // Comissão = 20% × 65000 = 13000

      // Act
      const resultado = calculadoraService.calcularComissao(input);

      // Assert
      expect(resultado.delta).toBe(-10000);
      expect(resultado.comissao).toBe(13000);
      expect(resultado.baseCalculo).toBe('valorPagoCedente');
    });

    it('retorna 20% do valorPagoCedente quando Delta = 0', () => {
      const input = { tabelaAtual: 70000, tabelaContrato: 70000, valorPagoCedente: 65000 };
      const resultado = calculadoraService.calcularComissao(input);
      expect(resultado.delta).toBe(0);
      expect(resultado.comissao).toBe(13000);
      expect(resultado.baseCalculo).toBe('valorPagoCedente');
    });
  });
});
```

**Rastreabilidade:** RN-DC-013 (cálculo de comissão), RF-DC-017 (calculadora determinística), ADR-003 (isolamento do módulo). [CORRIGIDO: PROBLEMA-003 + PROBLEMA-008]

---

## 4. Testes de Integração

### 4.1 Ferramentas e Setup

- **Framework:** Vitest + Supertest para endpoints HTTP.
- **Banco:** Supabase local via `supabase start` (PostgreSQL 15 real, não mock).
- **Redis:** Redis 7 local via Docker (real, não mock).
- **RabbitMQ:** RabbitMQ 3.12 local via Docker (real, não mock).
- **Setup/Teardown:** `beforeAll` cria seeds isolados por `cessionario_id`; `afterEach` limpa estado mutável (Redis keys, mensagens de fila); `afterAll` executa `DELETE` nas tabelas de teste usando `cessionario_id` de teste.

### 4.2 Endpoints — Critérios por Categoria

**Categoria: Agente/Chat**

| **Endpoint** | **Setup** | **Critério de Aprovação** |
|---|---|---|
| `POST /api/v1/dani/chat` | JWT válido, seed de oportunidade, Redis limpo | Status 200; stream SSE iniciado; `correlation_id` presente no header | [CORRIGIDO: PROBLEMA-004]
| `POST /api/v1/dani/chat` (sem JWT) | Sem Authorization header | Status 401; body `{ code: "AUTH_TOKEN_MISSING" }` | [CORRIGIDO: PROBLEMA-004]
| `POST /api/v1/dani/chat` (rate limit) | 31 requisições em 1 hora para o mesmo `cessionario_id` | Status 429; body com `retry_after` em segundos | [CORRIGIDO: PROBLEMA-004]
| `GET /api/v1/dani/conversas/{conversa_id}` | JWT de cessionário A, `conversa_id` pertencente ao cessionário B | Status 403; body `{ code: "AUTH_FORBIDDEN" }` | Endpoint corrigido para `/dani/conversas/{conversa_id}` conforme D16 §7 |

**Categoria: Calculadora**

| **Endpoint** | **Setup** | **Critério de Aprovação** |
|---|---|---|
| `POST /api/v1/calculadora/calcular` | JWT válido, `opr_id` válido do mesmo cessionário | Status 200; body com `delta`, `comissao`, `baseCalculo` corretos | [CORRIGIDO: PROBLEMA-005]
| `POST /api/v1/calculadora/calcular` | JWT válido, `opr_id` de outro cessionário | Status 403; body `{ code: "AUTH_FORBIDDEN" }` | [CORRIGIDO: PROBLEMA-005]
| `POST /api/v1/calculadora/calcular` | Requisição com valores inválidos (tabelaAtual negativo) | Status 422; body `{ code: "VALID_SCHEMA_ERROR" }` | [CORRIGIDO: PROBLEMA-005]

**Categoria: Autenticação / OTP**

| **Endpoint** | **Setup** | **Critério de Aprovação** |
|---|---|---|
| `POST /api/v1/auth/otp/solicitar` | Telefone válido, nenhum OTP pendente | Status 200; Redis contém chave `dani:rate:otp:{phone_hash}` |
| `POST /api/v1/auth/otp/solicitar` | 3 tentativas em 1 hora | Status 429; body com `retry_after` |
| `POST /api/v1/auth/otp/verificar` | OTP correto dentro do prazo | Status 200; JWT retornado |
| `POST /api/v1/auth/otp/verificar` | OTP incorreto 5 vezes | Status 429; chave `dani:block:otp:{phone_hash}` presente no Redis com TTL 1800s |
| `POST /api/v1/auth/otp/verificar` | OTP expirado (>15min) | Status 400; body `{ code: "AUTH_OTP_EXPIRED" }` |

### 4.3 Banco (Supabase)

- Queries de leitura DEVEM incluir `WHERE cessionario_id = :id` — teste verifica que registros de outro cessionário não são retornados.
- Seed de isolamento: criar `cessionario_A` e `cessionario_B` com oportunidades distintas; verificar que query de A retorna apenas dados de A.
- pgvector: teste de busca semântica verifica que namespace `cessionario_id` filtra corretamente — query de A não retorna chunks de B.

### 4.4 Redis

- `dani:rate:webchat:{id}` — verificar que contador incrementa e expira após 3600s.
- `dani:cache:calc:{opr_id}:{val_hash}` — verificar que cache hit retorna resultado idêntico sem re-execução da lógica.
- `dani:block:otp:{phone_hash}` — verificar que TTL é 1800s após 5 falhas consecutivas.
- `dani:revoked:refresh:{jti}` — verificar que refresh token revogado não é aceito.

### 4.5 RabbitMQ

- Publicação em `dani.notificacoes` — consumer consome e grava evento de tracking no Supabase.
- Retry com backoff — simular falha do consumer 1x; verificar reentrega após 5s.
- DLQ — simular 3 falhas consecutivas; verificar que mensagem vai para Dead Letter Queue.

### 4.6 Langfuse Tracing

- Verificar que `userId` no trace é SHA-256 do `cessionario_id` — nunca o UUID raw.
- Verificar que campos redactados (`phone`, `otp`, `jwt`, `cpf`) não aparecem nos traces.

### 4.7 Exemplo de Cenário de Integração

```typescript
// apps/api/test/integration/isolamento.spec.ts

describe('Isolamento por cessionario_id — Oportunidades', () => {
  beforeAll(async () => {
    await seed.criar({
      cessionarioA: { id: 'cess-aaa', oportunidades: ['opr-001', 'opr-002'] },
      cessionarioB: { id: 'cess-bbb', oportunidades: ['opr-003'] },
    });
  });

  it('GET /oportunidades retorna apenas oportunidades do cessionário autenticado', async () => {
    const token = await gerarJwt('cess-aaa');
    const res = await request(app.getHttpServer())
      .get('/api/v1/oportunidades')
      .set('Authorization', `Bearer ${token}`);

    expect(res.status).toBe(200);
    const ids = res.body.data.map((o: any) => o.id);
    expect(ids).toContain('opr-001');
    expect(ids).toContain('opr-002');
    expect(ids).not.toContain('opr-003'); // dado de outro cessionário — NUNCA deve aparecer
  });

  afterAll(async () => {
    await seed.limpar(['cess-aaa', 'cess-bbb']);
  });
});
```

**Rastreabilidade:** RN-DC-001 (isolamento), RF-DC-001 (CessionarioOwnerGuard), ADR-001 (isolamento como princípio).

---

## 5. Testes de Contrato

### 5.1 Ferramentas

- **Framework:** Pact (consumer-driven contract testing).
- **Escopo:** contratos entre `apps/api` e integrações externas (OpenAI, EvolutionAPI).
- **[DECISÃO AUTÔNOMA]:** Pact escolhido sobre testes de schema ad-hoc — Justificativa: Pact produz arquivo `.pact.json` versionado que detecta breaking changes automaticamente no CI, sem necessidade de sandbox real | Alternativa descartada: validação de schema com Zod manual — não detecta breaking changes de forma automatizada e não tem versionamento de contrato.

### 5.2 Contrato: OpenAI GPT-4o

| **Campo** | **Valor esperado** |
|---|---|
| Endpoint | `POST https://api.openai.com/v1/chat/completions` |
| Request: `model` | `gpt-4o` |
| Request: `stream` | `true` |
| Request: `temperature` | `0.3` |
| Request: `max_tokens` | `2048` |
| Response: status | `200` |
| Response: `choices[0].delta.content` | `string` (pode ser vazio no último chunk) |
| Response: `choices[0].finish_reason` | `"stop"` no chunk final |

Contrato de falha:
- HTTP 429 (rate limit OpenAI) → `AGENTE_LLM_RATE_LIMIT` — fallback: FallbackBanner + Calculadora.
- HTTP 503 → `AGENTE_LLM_UNAVAILABLE` — retry 3x com backoff 5s/15s/30s.

### 5.3 Contrato: EvolutionAPI (WhatsApp)

| **Campo** | **Valor esperado** |
|---|---|
| Endpoint | `POST /message/sendText/{instanceName}` |
| Request: `number` | string E.164 |
| Request: `text` | string não-vazia |
| Response: status | `200` ou `201` |
| Response: `key.id` | string (ID da mensagem) |

Contrato de falha:
- HTTP 503 por 3 tentativas consecutivas → circuit breaker aberto → `WA_CIRCUIT_OPEN`.

### 5.4 Contrato: Supabase REST API

- Schema das tabelas principais (`oportunidades`, `sessoes`, `alertas`) validado contra tipos gerados por `supabase gen types typescript`.
- Breaking change em coluna obrigatória → falha de contrato no CI.

### 5.5 Exemplo de Cenário de Contrato

```typescript
// apps/api/test/contract/openai.pact.spec.ts

const openaiMock = new Pact({ consumer: 'dani-api', provider: 'openai-gpt4o' });

describe('Contrato OpenAI — streaming chat', () => {
  it('responde com stream SSE quando model=gpt-4o e stream=true', async () => {
    await openaiMock.addInteraction({
      state: 'OpenAI disponível',
      uponReceiving: 'requisição de chat com stream',
      withRequest: {
        method: 'POST',
        path: '/v1/chat/completions',
        body: { model: 'gpt-4o', stream: true, temperature: 0.3 },
      },
      willRespondWith: {
        status: 200,
        headers: { 'Content-Type': 'text/event-stream' },
        body: Pact.eachLike({ choices: [{ delta: { content: string() } }] }),
      },
    });
    // executa chamada real ao mock e valida
  });
});
```

---

## 6. Testes E2E

### 6.1 Ferramentas e Ambiente

- **Framework:** Playwright (`@playwright/test@^1.x`) com Chromium.
- **Ambiente:** instância local completa — `pnpm dev` com banco Supabase local, Redis local, RabbitMQ local, EvolutionAPI mockada via `msw` (Mock Service Worker).
- **[DECISÃO AUTÔNOMA]:** MSW escolhido para mock de EvolutionAPI no E2E — Justificativa: MSW intercepta no nível de rede sem modificar código de produção; EvolutionAPI real não está disponível na Fase 1 | Alternativa descartada: servidor fake Express — maior manutenção e risco de divergência de comportamento.

### 6.2 Seed e Reset

- **Seed E2E:** arquivo `test/e2e/fixtures/seed-e2e.ts` com 1 cessionário de teste, 3 oportunidades, sessão ativa.
- **Reset entre testes:** `beforeEach` executa `DELETE FROM sessoes WHERE cessionario_id = 'cess-e2e-test'`; limpa Redis keys `dani:*:cess-e2e-test*`.
- **Dados controlados:** nunca usar dados reais de produção. `cessionario_id` de teste: `'00000000-0000-0000-0000-000000000001'`.
- **Autenticação E2E:** `storageState` do Playwright com JWT pré-gerado para o cessionário de teste. Refresh automático desativado para testes de sessão.

### 6.3 Fluxos Críticos E2E (obrigatórios)

| **ID** | **Fluxo** | **Prioridade** | **Requisito** |
|---|---|---|---|
| E2E-001 | Login via plataforma → acesso ao dashboard Dani | P0 | RN-DC-007 |
| E2E-002 | Chat com Dani: pergunta sobre oportunidade → resposta SSE completa | P0 | RF-DC-009 |
| E2E-003 | Cálculo de comissão via chat: Δ>0 e Δ≤0 | P0 | RN-DC-013 | [CORRIGIDO: PROBLEMA-003]
| E2E-004 | Rate limit no chat: 31ª mensagem recebe banner de limite | P0 | RN-DC-025 | [CORRIGIDO: RN-DC-030 inexistente → RN-DC-025]
| E2E-005 | Vinculação WhatsApp: solicitar OTP → OTP correto → confirmação | P0 | RN-DC-040, RN-DC-041, RN-DC-042, RN-DC-044 |
| E2E-006 | Bloqueio OTP: 5 tentativas incorretas → hard block 30min | P0 | RN-DC-041 | [CORRIGIDO: RN-DC-043 inexistente → RN-DC-041]
| E2E-007 | Isolamento: cessionário A não vê oportunidades do cessionário B | P0 | RN-DC-001 |
| E2E-008 | Estado Empty: nenhuma oportunidade → componente EmptyState exibido | P1 | D09 — 4 estados |
| E2E-009 | Estado Error: LLM indisponível → FallbackBanner + Calculadora acessível | P1 | RN-DC-031 |
| E2E-010 | Acessibilidade: navegação por teclado no chat, `aria-live` em mensagens | P1 | WCAG 2.1 AA |

### 6.4 Condições de Flaky Test e Mitigações

| **Causa de Flakiness** | **Mitigação** |
|---|---|
| SSE timeout antes de chunk final | `waitForResponse` com timeout 15s; asserção em evento `finish_reason: stop` |
| Race condition no Redis após reset | `afterEach` aguarda `DEL` com confirmação antes de próximo teste |
| LLM mock com delay variável | MSW com `delay: 100ms` fixo para todos os handlers |
| Banco com dados residuais | Seed usa `cessionario_id` UUID único por run; cleanup garantido no `afterAll` |

### 6.5 Exemplo de Cenário E2E

```typescript
// test/e2e/chat-calculo.spec.ts

test('E2E-003: calcular comissão com Delta positivo via chat', async ({ page }) => {
  await page.goto('/dashboard');
  await page.waitForSelector('[data-testid="dani-chat-input"]');

  // Abre chat
  await page.click('[data-testid="dani-fab"]');
  await page.fill('[data-testid="dani-chat-input"]', 'Qual a comissão da OPR-001?');
  await page.press('[data-testid="dani-chat-input"]', 'Enter');

  // Aguarda streaming completo
  await page.waitForSelector('[data-testid="dani-message-complete"]', { timeout: 15000 });

  // Verifica que resposta contém o valor calculado
  const mensagem = await page.textContent('[data-testid="dani-last-message"]');
  expect(mensagem).toContain('R$ 3.000,00'); // 20% × Δ15000

  // Verifica que não há spinner (proibido por design)
  const spinner = await page.$('[data-testid="loading-spinner"]');
  expect(spinner).toBeNull();
});
```

---

## 7. Cenários por Módulo

### 7.1 AgenteModule

| **#** | **Cenário** | **Tipo** | **Prioridade** | **Requisito** | **Risco Coberto** | **Status Esperado** |
|---|---|---|---|---|---|---|
| A-01 | Mensagem válida → resposta SSE completa | Integração | P0 | RF-DC-009 | LLM não responde | Pass: stream completo |
| A-02 | Rate limit 30 msgs/hora — mensagem 31 | Unitário + Integração | P0 | RN-DC-030 | Abuso da API | Pass: 429 + `retry_after` |
| A-03 | `confidence_score < 0.80` por 3 turnos consecutivos | Unitário | P1 | RN-DC-035 | Agente sem resposta útil | Pass: takeover enfileirado |
| A-04 | Tool `buscarOportunidades` com `cessionario_id` correto | Unitário | P0 | RN-DC-001 | Vazamento de dados | Pass: query filtrada |
| A-05 | Tool `calcularComissao` retorna resultado em cache | Unitário | P1 | RF-DC-015 | Cálculo duplo inconsistente | Pass: cache hit, resultado idêntico |
| A-06 | LLM indisponível → FallbackBanner sem spinner | E2E | P0 | RN-DC-031 | UX degradada | Pass: FallbackBanner visível |
| A-07 | Mensagem de usuário contém PII (CPF) → redacted no log | Integração | P0 | RN-DC-050 | Dado sensível em log | Pass: `[REDACTED]` no campo |
| A-08 | Session Redis expira (1800s) → nova sessão criada | Integração | P1 | RN-DC-025 | Perda de contexto | Pass: nova `session_id` gerada |
| A-09 | Guardrail: pergunta fora do escopo (ex: receita de bolo) | Unitário | P1 | RN-DC-036 | Uso indevido do agente | Pass: resposta de recusa padrão |
| A-10 | Langfuse trace com `userId = sha256(cessionario_id)` | Integração | P0 | RN-DC-055 | UUID raw em trace | Pass: SHA-256 presente, UUID ausente |

### 7.2 CalculadoraModule

| **#** | **Cenário** | **Tipo** | **Prioridade** | **Requisito** | **Risco Coberto** | **Status Esperado** |
|---|---|---|---|---|---|---|
| C-01 | Δ > 0: comissão = 20% × Δ | Unitário | P0 | RN-DC-013 | Cálculo financeiro errado | Pass: valor correto | [CORRIGIDO: PROBLEMA-003]
| C-02 | Δ ≤ 0: comissão = 20% × valorPagoCedente | Unitário | P0 | RN-DC-013 | Cálculo financeiro errado | Pass: valor correto | [CORRIGIDO: PROBLEMA-003]
| C-03 | Δ = 0: comissão = 20% × valorPagoCedente | Unitário | P0 | RN-DC-013 | Edge case ignorado | Pass: valor correto | [CORRIGIDO: PROBLEMA-003]
| C-04 | `opr_id` de outro cessionário → 403 | Integração | P0 | RN-DC-001 | Dados cruzados | Pass: 403 `AUTH_FORBIDDEN` |
| C-05 | Cache hit: segunda requisição idêntica retorna cache | Integração | P1 | RF-DC-031 | Cálculo duplo | Pass: `X-Cache: HIT` | [CORRIGIDO: RF-DC-015 é Score de Risco → RF-DC-031 é Calculadora fallback]
| C-06 | Payload inválido (tabelaAtual negativo) → 422 | Integração | P1 | RF-DC-017 | Dado inválido processado | Pass: 422 `VALID_SCHEMA_ERROR` | [CORRIGIDO: RF-DC-030 inexistente → RF-DC-017]
| C-07 | Resultado armazenado em `dani:cache:calc:{opr_id}:{val_hash}` | Integração | P1 | RF-DC-031 | Cache com key errada | Pass: chave Redis correta | [CORRIGIDO: RF-DC-015 → RF-DC-031]
| C-08 | Precisão: valores com centavos (ex: R$ 85.450,75) | Unitário | P0 | RN-DC-013 | Arredondamento incorreto | Pass: centavos preservados | [CORRIGIDO: PROBLEMA-003]
| C-09 | `CalculadoraModule` isolado de `AgenteModule` | Unitário | P0 | ADR-003 | Acoplamento indevido | Pass: sem import circular |
| C-10 | Cálculo com `valorPagoCedente = 0` | Unitário | P1 | RN-DC-013 | Divisão por zero ou erro | Pass: comissão = 0 | [CORRIGIDO: PROBLEMA-003]

### 7.3 AuthModule / OtpService

| **#** | **Cenário** | **Tipo** | **Prioridade** | **Requisito** | **Risco Coberto** | **Status Esperado** |
|---|---|---|---|---|---|---|
| O-01 | OTP gerado: 6 dígitos, TTL 15min | Unitário | P0 | RN-DC-040 | OTP inválido ou eterno | Pass: formato e TTL corretos |
| O-02 | OTP armazenado como bcrypt hash (cost 12) | Unitário | P0 | RN-DC-041 | OTP em texto plano | Pass: `bcrypt.compare` válido |
| O-03 | OTP correto → JWT retornado | Integração | P0 | RN-DC-041 | Autenticação falha | Pass: JWT válido |
| O-04 | OTP incorreto 1x → contador incrementado | Unitário | P0 | RN-DC-043 | Contador não cresce | Pass: Redis `INCR` + TTL |
| O-05 | OTP incorreto 5x → hard block 1800s | Integração | P0 | RN-DC-043 | Brute force possível | Pass: 429 + TTL 1800s |
| O-06 | OTP expirado (>15min) → 400 | Integração | P0 | RN-DC-040 | OTP fora do prazo aceito | Pass: 400 `AUTH_OTP_EXPIRED` |
| O-07 | 3 solicitações/hora → 4ª bloqueada | Integração | P0 | RN-DC-042 | Spam de SMS | Pass: 429 `RATE_OTP_LIMIT` |
| O-08 | Phone armazenado como SHA-256 na chave Redis | Unitário | P0 | RN-DC-041 | Phone em texto no Redis | Pass: `dani:rate:otp:{sha256}` |
| O-09 | Session inheritance: JWT da plataforma aceito | Integração | P0 | RN-DC-007 | Re-autenticação desnecessária | Pass: acesso sem OTP |
| O-10 | `CessionarioOwnerGuard` — ADMIN bypass | Unitário | P1 | RN-DC-007 | Admin bloqueado | Pass: role ADMIN não verificado |

### 7.4 WhatsappModule

| **#** | **Cenário** | **Tipo** | **Prioridade** | **Requisito** | **Risco Coberto** | **Status Esperado** |
|---|---|---|---|---|---|---|
| W-01 | Envio OTP via EvolutionAPI — sucesso | Integração | P0 | RN-DC-040 | OTP não entregue | Pass: `WA_MESSAGE_SENT` |
| W-02 | EvolutionAPI 503 — retry 3x com backoff | Integração | P0 | RN-DC-040 | OTP perdido em falha | Pass: 3 tentativas + DLQ |
| W-03 | Circuit breaker abre após 3 falhas consecutivas | Unitário | P0 | ADR-001 | Cascata de falhas | Pass: `WA_CIRCUIT_OPEN` |
| W-04 | PARAR recebido → opt-out imediato gravado | Integração | P0 | RN-DC-044 | LGPD — opt-out ignorado | Pass: campo `whatsapp_optout = true` |
| W-05 | Notificação ZapSign D+2 publicada na fila correta | Integração | P1 | RN-DC-060 | Notificação perdida | Pass: `dani.whatsapp` queue |

### 7.5 AlertaModule

| **#** | **Cenário** | **Tipo** | **Prioridade** | **Requisito** | **Risco Coberto** | **Status Esperado** |
|---|---|---|---|---|---|---|
| AL-01 | Alerta In-App exibido no FAB com badge | E2E | P1 | RF-DC-020 | Alerta ignorado | Pass: badge visível |
| AL-02 | Alerta persistido com `cessionario_id` correto | Integração | P0 | RN-DC-001 | Isolamento quebrado | Pass: query filtrada |
| AL-03 | Polling 30s: novo alerta aparece sem reload | E2E | P1 | RF-DC-020 | Alerta atrasado | Pass: badge atualizado em ≤35s |
| AL-04 | DLQ: falha de entrega 3x → admin notificado | Integração | P0 | RN-DC-065 | Alerta crítico perdido | Pass: `dani.agent_monitor` publicado |

---

## 8. Dados de Teste

### 8.1 Factories

```typescript
// test/factories/cessionario.factory.ts
export const cessionarioFactory = {
  create: (overrides?: Partial<Cessionario>): Cessionario => ({
    id: '00000000-0000-0000-0000-000000000001',
    nome: 'Cessionário Teste A',
    email: 'teste-a@dani.local',
    telefone: '+5511999990001',
    whatsapp_vinculado: false,
    ...overrides,
  }),
};

// test/factories/oportunidade.factory.ts
export const oportunidadeFactory = {
  create: (cessionarioId: string, overrides?: Partial<Oportunidade>): Oportunidade => ({
    id: `opr-${uuid()}`,
    cessionario_id: cessionarioId,
    tabela_atual: 85000,
    tabela_contrato: 70000,
    valor_pago_cedente: 65000,
    status: 'DISPONIVEL',
    ...overrides,
  }),
};
```

### 8.2 Seeds

| **Seed** | **Propósito** | **Arquivo** |
|---|---|---|
| `seed-unitario.ts` | Dados estáticos para testes unitários sem banco | `test/seeds/seed-unitario.ts` |
| `seed-integracao.ts` | 2 cessionários + 5 oportunidades cada para isolamento | `test/seeds/seed-integracao.ts` |
| `seed-e2e.ts` | 1 cessionário + 3 oportunidades + sessão ativa | `test/e2e/fixtures/seed-e2e.ts` |
| `seed-regressao.ts` | Casos históricos de bugs: Δ=0, OPR com centavos, OTP expirado | `test/seeds/seed-regressao.ts` |

### 8.3 Mocks Externos

| **Serviço** | **Mock** | **Arquivo** |
|---|---|---|
| OpenAI GPT-4o | `vi.mock` (unitário); MSW handler (E2E) | `test/mocks/openai.mock.ts` |
| EvolutionAPI | MSW handler (integração + E2E) | `test/mocks/evolutionapi.mock.ts` |
| Langfuse | `vi.mock` (unitário); real local (integração) | `test/mocks/langfuse.mock.ts` |
| SMS Provider | `vi.mock` sempre (sem sandbox real) | `test/mocks/sms.mock.ts` |

### 8.4 Dados Mascarados

- Nunca usar CPF, telefone ou e-mail reais em seeds.
- Padrão para telefone de teste: `+5511999990001` a `+5511999990099`.
- Padrão para CPF de teste: `000.000.000-01` a `000.000.000-99` (inválidos por design).
- E-mail de teste: `teste-{n}@dani.local` (domínio inexistente).

### 8.5 Limpeza e Isolamento

- `afterEach`: limpa Redis keys com prefixo `dani:*:cess-test-*`.
- `afterAll`: `DELETE FROM tabela WHERE cessionario_id IN ('...ids de teste...')`.
- Testes de integração NUNCA compartilham `cessionario_id` com testes E2E.
- Seeds são idempotentes: `INSERT ... ON CONFLICT DO NOTHING`.

---

## 9. Cobertura e Gates

### 9.1 Thresholds por Camada

| **Camada** | **Threshold Geral** | **Threshold Fluxos Críticos** | **Ferramenta** |
|---|---|---|---|
| Unitário | ≥80% statements | 100% (Calculadora, Guard, OTP) | `@vitest/coverage-v8` |
| Integração | ≥70% endpoints cobertos | 100% (isolamento, auth, rate limit) | Vitest + Supertest |
| E2E | 10 fluxos obrigatórios | 100% (E2E-001 a E2E-007) | Playwright |
| Contrato | 100% dos contratos definidos | 100% | Pact |

### 9.2 Critérios de Bloqueio de Merge

| **Condição** | **Bloqueia** | **Nível** |
|---|---|---|
| Qualquer teste de isolamento falha | SIM — imediato | `feature/*` → `develop` e `develop` → `main` |
| Qualquer teste de cálculo de comissão falha | SIM — imediato | `feature/*` → `develop` e `develop` → `main` |
| Qualquer teste de OTP / auth falha | SIM — imediato | `feature/*` → `develop` e `develop` → `main` |
| Cobertura unitária < 80% | SIM | `feature/*` → `develop` |
| Cobertura integração < 70% | SIM | `develop` → `main` |
| Qualquer fluxo E2E obrigatório falha | SIM | `develop` → `main` |
| Falha em teste de contrato (Pact) | SIM | `develop` → `main` |
| Teste de acessibilidade (WCAG) falha | SIM | `develop` → `main` |
| Falha em teste não-crítico (aviso) | NÃO — PR marcado com label `needs-attention` | — |

### 9.3 Critérios de Bloqueio de Release (Go-Live)

- Regressão completa sem nenhuma falha.
- Cobertura unitária ≥80%, integração ≥70%.
- Todos os 10 fluxos E2E obrigatórios verde.
- Performance: p95 ≤5s para chat, ≤10s para comparativo (medido em ambiente staging).
- Acessibilidade: 0 violações críticas no relatório Axe.
- Nenhum `[PENDÊNCIA]` crítico sem resolução documentada.

### 9.4 Exemplo de Gate com Critério Objetivo

```yaml
# .github/workflows/ci.yml — gate de cobertura

- name: Verificar cobertura mínima
  run: |
    COVERAGE=$(vitest run --coverage --reporter=json | jq '.total.statements.pct')
    if (( $(echo "$COVERAGE < 80" | bc -l) )); then
      echo "❌ Cobertura abaixo do mínimo: $COVERAGE% (mínimo: 80%)"
      exit 1
    fi
    echo "✅ Cobertura: $COVERAGE%"
```

---

## 10. Testes de Regressão

### 10.1 Quando Executar

| **Evento** | **Suite de Regressão** |
|---|---|
| PR para `main` | Suite completa (unitário + integração + contrato + E2E) |
| Release candidate | Suite completa + não-funcional |
| Hotfix aplicado | Suite crítica (isolamento + auth + calculadora) — máx 10min |
| Atualização de dependência major | Suite completa + contrato |
| Mudança em system prompt do agente | Testes de guardrail + regressão de prompt |

### 10.2 Fluxos Fixos de Regressão

Estes cenários NUNCA saem da suíte de regressão, independentemente de mudanças no produto:

1. Isolamento por `cessionario_id` (A-04, C-04, O-09, AL-02).
2. Cálculo de comissão: Δ>0, Δ≤0, Δ=0, centavos (C-01, C-02, C-03, C-08).
3. OTP: geração, verificação, expiração, hard block (O-01 a O-06).
4. Rate limit: webchat e OTP (A-02, O-07).
5. SSE streaming completo (A-01, E2E-002).
6. FallbackBanner quando LLM indisponível (A-06, E2E-009).

### 10.3 Manutenção da Suíte

- Teste obsoleto: abrir issue `chore: remover teste obsoleto [ID]` com rastreabilidade do requisito que deixou de existir. Aprovação de 2 revisores antes de remover.
- Teste flaky (3 falhas não-determinísticas em 7 dias): mover para `test/quarantine/` com issue aberta. Nunca deletar silenciosamente.
- Novos fluxos de agente: adicionar cenário unitário de guardrail antes de merge.

### 10.4 Regressão de Prompt

Cada versão do system prompt (`dani-system-prompt.v{N}.ts`) exige:
- Mínimo 5 casos de teste de resposta (within scope, out of scope, cálculo, risco, comparativo).
- Comparação com versão anterior: nenhum comportamento previamente aprovado deve regredir.
- Arquivo de golden outputs: `test/regression/prompts/v{N}-golden.json`.

---

## 11. Testes Não-Funcionais

### 11.1 Performance e Carga

**Ferramenta:** [DECISÃO AUTÔNOMA]: k6 escolhido para testes de carga — Justificativa: k6 é scriptável em JavaScript/TypeScript (alinhado ao stack), tem integração nativa com CI e suporte a SLO assertions | Alternativa descartada: Apache JMeter — mais pesado, configuração em XML, integração CI mais complexa.

| **Cenário** | **Carga** | **SLO** | **Critério de Falha** |
|---|---|---|---|
| Chat: 50 usuários simultâneos | 50 VUs × 5min | p95 ≤5s | >5% requisições acima de 5s |
| Calculadora: 100 req/s | 100 RPS × 2min | p95 ≤500ms | >1% erros ou p95 >500ms |
| Comparativo regional: 20 VUs | 20 VUs × 5min | p95 ≤10s | >5% requisições acima de 10s |
| OTP: 10 req/s | 10 RPS × 3min | p95 ≤2s | >1% erros ou timeout |

### 11.2 Resiliência

| **Cenário** | **Injeção de Falha** | **Critério de Aprovação** |
|---|---|---|
| Redis indisponível | Parar container Redis por 30s | API retorna 503 `INFRA_REDIS_UNAVAILABLE`; sem crash |
| RabbitMQ indisponível | Parar container por 30s | Notificações bufferizadas; entregues quando RabbitMQ volta |
| OpenAI HTTP 503 | MSW retorna 503 | FallbackBanner exibido; Calculadora disponível; sem loop de retry infinito |
| EvolutionAPI timeout | MSW `delay: 60s` | Circuit breaker abre após 3 tentativas; `WA_CIRCUIT_OPEN` retornado |

### 11.3 Acessibilidade

- **Ferramenta:** `@axe-core/playwright` integrado aos testes E2E.
- **Critério:** 0 violações de nível `critical` ou `serious` (WCAG 2.1 AA).
- **Escopo:** todas as páginas do fluxo E2E obrigatório.
- Cenários específicos:
  - Navegação por teclado no chat: Tab → input → Enter → mensagem enviada.
  - `aria-live="polite"` em `[data-testid="dani-last-message"]` — verificado via axe.
  - `role="dialog"` no painel de chat com focus trap ativo.
  - `aria-label` em todos os botões de ação (FAB, enviar, fechar).

### 11.4 Segurança Básica

- SQL injection: payload `'; DROP TABLE oportunidades; --` em todos os campos de texto → 422 ou 400 sem execução.
- XSS: payload `<script>alert(1)</script>` em mensagem do chat → sanitizado antes de renderizar.
- JWT manipulado (payload alterado, assinatura inválida) → 401 `AUTH_TOKEN_INVALID`.
- CORS: requisição de origem não autorizada → 403.
- Headers de segurança: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security` presentes em todas as respostas.

---

## 12. Integração com CI/CD

### 12.1 Pipeline de CI

```
Push → PR aberto
  │
  ├── Stage 1: Fast Feedback (≤5min)
  │     ├── lint (ESLint + Prettier)
  │     ├── TypeScript check (tsc --noEmit)
  │     ├── Conventional Commits check
  │     └── Secrets scan (GitLeaks)
  │
  ├── Stage 2: Unitário (≤8min)
  │     └── vitest run --coverage
  │
  ├── Stage 3: Integração (≤15min)
  │     ├── docker-compose up (postgres, redis, rabbitmq)
  │     ├── supabase db push
  │     ├── vitest run --config vitest.integration.config.ts
  │     └── docker-compose down
  │
  ├── Stage 4: Contrato (≤5min) — apenas em PR para main
  │     └── pact verify
  │
  └── Stage 5: E2E (≤20min) — apenas em PR para main
        ├── pnpm dev (background)
        ├── playwright test
        └── axe-core accessibility report
```

### 12.2 Tempos Máximos por Stage

| **Stage** | **Tempo Máximo** | **Se Exceder** |
|---|---|---|
| Fast Feedback | 5min | Falha de CI — revisar regras de lint |
| Unitário | 8min | Falha de CI — identificar teste lento com `--reporter=verbose` |
| Integração | 15min | Falha de CI — revisar setup/teardown de banco |
| Contrato | 5min | Falha de CI — verificar disponibilidade do Pact Broker |
| E2E | 20min | Falha de CI — revisar flaky tests em quarentena |

### 12.3 Resultados Publicados

| **Output** | **Destino** | **Quando** |
|---|---|---|
| Coverage report (LCOV) | GitHub Actions Artifacts + PR comment | A cada PR |
| Pact contracts | Pact Broker (self-hosted ou PactFlow) | PR para main |
| Playwright report (HTML) | GitHub Actions Artifacts | PR para main |
| Axe violations report | PR comment com contagem | PR para main |
| k6 performance report | GitHub Actions Artifacts | Release candidate |

### 12.4 Falhas que Bloqueiam Promoção

| **Falha** | **Bloqueia `feature → develop`** | **Bloqueia `develop → main`** |
|---|---|---|
| Lint ou TypeScript error | SIM | SIM |
| Secrets detectados (GitLeaks) | SIM | SIM |
| Qualquer teste unitário P0 | SIM | SIM |
| Cobertura < 80% | SIM | SIM |
| Qualquer teste integração P0 | NÃO* | SIM |
| Falha E2E obrigatória | NÃO* | SIM |
| Falha de contrato (Pact) | NÃO* | SIM |
| Violação de acessibilidade crítica | NÃO* | SIM |

*Integração, E2E e contrato rodam apenas em PR para `main` por padrão. Para `develop`, rodam em modo `--passWithNoTests` com aviso no PR comment. [DECISÃO AUTÔNOMA]: esta separação acelera o feedback loop em `develop` sem reduzir a qualidade do gate final | Alternativa descartada: rodar suite completa em todo PR — tempo médio de CI 48min, inviável para iteração rápida.

---

## 13. Backlog de Pendências

| **ID** | **Tipo** | **Descrição** | **Impacto** | **Responsável** |
|---|---|---|---|---|
| P-01 | [DEFINIÇÃO PENDENTE] | Provedor de SMS para OTP não definido (Twilio vs AWS SNS) — impacta mock de integração e seed de número de teste | Gate de release pode bloquear se SMS mock não simular comportamento correto do provedor escolhido | Produto + Infra |
| P-02 | [DEFINIÇÃO PENDENTE] | Pact Broker: self-hosted vs PactFlow — impacta Stage 4 do CI e publicação de contratos | Se self-hosted não estiver disponível em staging, Stage 4 não roda | Infra |
| P-03 | [DECISÃO AUTÔNOMA] | Ambiente de staging para E2E de release: Railway com variáveis de staging | Staging isolado de produção; Railway alinhado com D02/D22 | DevOps |
| P-04 | PENDÊNCIA | Cenários de teste para comparativo regional (`buscarComparativoRegional`) — dados de mercado regional não definidos em D01/D05 | 3 cenários E2E não cobertos até que dados sejam definidos | Produto |
| P-05 | PENDÊNCIA | Testes de penetração aprofundados (OWASP Top 10 completo) — fora do escopo deste plano | Recomendado antes do go-live; equipe especializada | Segurança |

---

*Documento gerado por Claude Code Desktop — Pipeline ShiftLabs v9.5 — AI-Dani-Cessionário*
