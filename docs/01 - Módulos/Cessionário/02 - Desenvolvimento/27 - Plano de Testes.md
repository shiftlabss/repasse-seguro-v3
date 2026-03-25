# 27 - Plano de Testes

| Campo | Valor |
|---|---|
| **Nome do Documento** | 27 - Plano de Testes |
| **Versão** | v1.0 |
| **Data** | 2026-03-22 (America/Fortaleza) |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Produto** | Repasse Seguro — Módulo Cessionário |

---

> 📌 **TL;DR — Estratégia de Testes**
>
> - **Pirâmide:** 70% unitário / 20% integração / 8% E2E / 2% manual exploratório.
> - **Ferramentas:** Vitest (unitário + integração backend), Playwright (E2E), Testing Library (frontend), Supertest (API).
> - **Fluxos críticos obrigatórios:** KYC → Proposta → Negociação → Escrow → Formalização → Fechamento.
> - **Gates:** cobertura ≥ 80% em módulos críticos; 100% nos cálculos de comissão e Escrow; E2E verde para todos os fluxos críticos antes de merge em `main`.
> - **Dados:** factories isoladas por suíte; banco de teste resetado antes de cada suíte E2E; nenhum dado de produção em testes.
> - **CI:** unitário + integração rodam em todo PR; E2E roda em PRs para `main` e em staging pré-deploy.
> - **Bloqueio de merge:** qualquer falha em unitário, integração ou lint bloqueia `develop`; qualquer falha em E2E bloqueia `main`.

---

## 1. Estratégia Geral

O Módulo Cessionário é o coração operacional e financeiro da plataforma Repasse Seguro. Ele processa cessões imobiliárias, movimenta recursos financeiros via conta Escrow (Celcoin), integra com plataforma de assinatura digital (ZapSign), verifica identidades (idwall/KYC), envia notificações multi-canal e oferece agente de IA (Analista de Oportunidades).

**Perfil de risco do produto:**

| Área | Criticidade | Razão |
|---|---|---|
| Cálculo de comissão (Comprador) | P0 — CRÍTICO | Impacto financeiro direto. Erro gera perda ou cobrança indevida. |
| Escrow — depósito e liberação | P0 — CRÍTICO | Movimentação real de dinheiro via Celcoin/BCB. |
| KYC — validação de identidade | P0 — CRÍTICO | Fraude tem impacto legal e regulatório. |
| Anonimato do Cedente | P0 — CRÍTICO | Exposição de dados viola LGPD e quebra o modelo de negócio. |
| Formalização (ZapSign) | P1 — ALTO | Falha na assinatura pode atrasar ou bloquear fechamento. |
| Notificações críticas (NOT-CES-05/06) | P1 — ALTO | Alertas de prazo Escrow — falha pode causar perda de oportunidade. |
| Fluxo de proposta e negociação | P1 — ALTO | Core do produto; falha afeta receita. |
| Analista de Oportunidades (IA) | P2 — MÉDIO | Funcionalidade de valor, mas falha não bloqueia operação. |
| Dashboard e visualizações | P2 — MÉDIO | UX degradado, mas sem impacto financeiro direto. |

**Escopo de teste:**

- Backend: NestJS (todos os módulos de negócio)
- Frontend: React + Vite (fluxos críticos e formulários)
- Mobile: React Native + Expo (notificações push, KYC, proposta)
- Integrações: ZapSign, idwall, Celcoin (mocks em unitário/integração; sandbox em E2E)

---

## 2. Pirâmide de Testes

```
           /\
          /  \
         / E2E \        ~8% — Fluxos críticos end-to-end
        /--------\
       /Integração\      ~20% — Endpoints, banco, filas, integrações
      /------------\
     /  Unitário    \    ~70% — Funções, services, cálculos, guards
    /----------------\
   / Manual Explorat. \  ~2% — Casos extremos e novos fluxos
  /--------------------\
```

| Camada | Proporção | Objetivo | Ferramenta |
|---|---|---|---|
| Unitário | 70% | Lógica de negócio, cálculos, transformações, guards, pipes | Vitest |
| Integração | 20% | Endpoints reais, banco de dados, filas RabbitMQ, Redis, workers | Vitest + Supertest |
| E2E | 8% | Fluxos completos do usuário no browser | Playwright |
| Manual exploratório | 2% | Novos fluxos, edge cases, UX crítico pré-release | Checklist manual |

**Justificativa da proporção:**
[DECISÃO AUTÔNOMA — 70/20/8/2. Produto com lógica financeira complexa favorece alta cobertura unitária para proteger cálculos de comissão e regras de Escrow. E2E focado em fluxos críticos evita suíte frágil e lenta. Alternativa descartada: 50/30/20 com mais E2E — aumentaria tempo de CI sem ganho proporcional de confiança dado o isolamento já garantido por unitário e integração.]

---

## 3. Testes Unitários

### 3.1 Objetivo

Verificar a lógica de negócio isolada de dependências externas. Foco em funções puras, services, cálculos financeiros, validações, guards e transformações de dados.

### 3.2 Ferramentas

| Ferramenta | Uso |
|---|---|
| Vitest | Runner, assertions, coverage |
| `vi.mock()` | Mock de dependências (Prisma, Redis, RabbitMQ, HTTP clients) |
| Testing Library (React) | Renderização de componentes frontend |
| `@nestjs/testing` | `Test.createTestingModule()` para providers isolados |

### 3.3 Cobertura Mínima

| Área | Cobertura mínima | Observação |
|---|---|---|
| Cálculo de Comissão Comprador | **100%** | Zero tolerância a erro financeiro |
| Lógica de Escrow (prazos, estados) | **100%** | Impacto financeiro direto |
| Guards de autenticação e RBAC | **95%** | Segurança |
| Services de negócio (proposta, negociação, KYC) | **80%** | Fluxo core |
| Services auxiliares (notificação, log) | **70%** | Suporte |
| Componentes frontend com lógica | **80%** | Formulários, validações |
| Utilitários e helpers | **90%** | Alta reutilização |

### 3.4 Padrão e Naming Convention

```typescript
// Formato: describe('<NomeDoService>') > describe('<nomeDoMetodo>') > it('<cenário>')
describe('CommissionService', () => {
  describe('calculateBuyerCommission', () => {
    it('should return 20% of delta when delta > 0', () => { ... })
    it('should apply 20% of cedente paid value when delta <= 0', () => { ... })
    it('should throw ValidationError when tabela_atual is missing', () => { ... })
  })
})
```

**Regras:**

- Um `describe` por classe/service/component.
- Um `it` por cenário (feliz + alternativo + erro).
- Mocks declarados no topo do arquivo, nunca inline em cada `it`.
- Não testar implementação interna — testar comportamento observável.
- `console.log` proibido em testes.

### 3.5 Quando Não Usar Teste Unitário

- Para lógica que só faz sentido com banco real (usar integração).
- Para fluxos de autenticação end-to-end (usar E2E).
- Para componentes puramente visuais sem lógica (usar snapshot com cuidado).

### 3.6 Exemplo de Cenário Unitário

```typescript
// src/modules/commission/commission.service.spec.ts
describe('CommissionService', () => {
  describe('calculateBuyerCommission', () => {
    it('should calculate 20% of delta when Tabela Atual > Tabela Contrato', () => {
      // Arrange
      const input = {
        tabelaAtual: 500_000,
        tabelaContrato: 400_000,
        valorPagoCedente: 200_000,
      }
      const expected = 20_000 // 20% × (500k - 400k) = 20k

      // Act
      const result = service.calculateBuyerCommission(input)

      // Assert
      expect(result).toBe(expected)
    })

    it('should apply 20% of valorPagoCedente when delta <= 0', () => {
      // Arrange: oportunidade submersa
      const input = {
        tabelaAtual: 380_000,
        tabelaContrato: 400_000, // delta = -20k
        valorPagoCedente: 200_000,
      }
      const expected = 40_000 // 20% × 200k

      // Act + Assert
      expect(service.calculateBuyerCommission(input)).toBe(expected)
    })
  })
})
```

---

## 4. Testes de Integração

### 4.1 Objetivo

Verificar a integração real entre as partes do sistema: endpoints HTTP, banco de dados Prisma, cache Redis, filas RabbitMQ, workers e serviços externos com mock/stub.

### 4.2 Setup

```typescript
// Ambiente de integração:
// - PostgreSQL real (Docker Compose — mesma configuração do D22)
// - Redis real (Docker Compose)
// - RabbitMQ real (Docker Compose)
// - Serviços externos: ZapSign, idwall, Celcoin → SEMPRE mockados via MSW ou vi.mock()
// - Supabase Storage: mockado via vi.mock('@supabase/supabase-js')

// Antes de cada suíte:
beforeAll(async () => {
  await app.get(PrismaService).$executeRaw`TRUNCATE TABLE ... CASCADE`
  await redis.flushDb()
})

afterEach(async () => {
  await prisma.$executeRaw`DELETE FROM proposals WHERE id LIKE 'test-%'`
})
```

### 4.3 Categorias e Critérios

| Categoria | Setup | Dependências | Critério de aprovação |
|---|---|---|---|
| Endpoints REST (Supertest) | App NestJS completo, banco real | Prisma + banco | Status HTTP correto, body da resposta, headers |
| Autenticação e JWT | JWT real, Redis real | Supabase Auth mockado | Token válido, refresh, revogação, rate limit |
| Workers RabbitMQ | Consumer real, banco real | Fila real, serviços externos mockados | Mensagem consumida, estado atualizado, DLQ funciona |
| Migrations Prisma | Banco limpo pré-suíte | PostgreSQL real | Migrations aplicadas sem erro; rollback funcional |
| Cache Redis | Redis real | TTL configurado | Hit/miss correto; expiração funciona |
| KYC Worker | Consumer real, banco real | idwall mockado | Status atualizado corretamente em todos os fluxos |

### 4.4 Exemplo de Cenário de Integração

```typescript
// src/modules/proposal/proposal.controller.integration.spec.ts
describe('POST /api/v1/proposals', () => {
  it('should create proposal and return 201 when cessionario has approved KYC', async () => {
    // Arrange: seed cessionario com KYC aprovado
    const cessionario = await factory.create('cessionario', { kycStatus: 'APPROVED' })
    const opportunity = await factory.create('opportunity', { status: 'DISPONIVEL' })
    const token = generateTestJwt(cessionario.id)

    // Act
    const response = await request(app.getHttpServer())
      .post('/api/v1/proposals')
      .set('Authorization', `Bearer ${token}`)
      .send({ opportunityId: opportunity.id, valor: 450_000 })

    // Assert
    expect(response.status).toBe(201)
    expect(response.body.data.status).toBe('PENDING')
    expect(response.body.data.opportunityId).toBe(opportunity.id)

    // Verificar efeito colateral: notificação enfileirada
    const queued = await rabbitMqTestHelper.getQueueMessages('notification.email')
    expect(queued).toHaveLength(1)
    expect(queued[0].template).toBe('NOT-CES-03')
  })

  it('should return 403 when cessionario KYC is pending', async () => {
    const cessionario = await factory.create('cessionario', { kycStatus: 'PENDING' })
    const token = generateTestJwt(cessionario.id)

    const response = await request(app.getHttpServer())
      .post('/api/v1/proposals')
      .set('Authorization', `Bearer ${token}`)
      .send({ opportunityId: 'opp-1', valor: 450_000 })

    expect(response.status).toBe(403)
    expect(response.body.error.code).toBe('PERM-001')
  })
})
```

---

## 5. Testes de Contrato

### 5.1 Objetivo

Garantir que os contratos de API (schemas de request/response, status codes, headers) não regridem entre versões. Protege integrações externas e o frontend.

### 5.2 Contratos Documentados

[DECISÃO AUTÔNOMA — usar testes de snapshot de schema OpenAPI/Swagger gerado pelo NestJS + validação de contrato via Zod schemas compartilhados em `packages/shared-types/`. Alternativa descartada: Pact.io (consumer-driven contract) — overhead elevado para time IA sem consumidores externos formalizados além do próprio frontend.]

| Endpoint | Schema obrigatório | Status codes esperados |
|---|---|---|
| `POST /api/v1/proposals` | `CreateProposalDto` → `ProposalResponseDto` | 201, 400, 403, 409, 429 |
| `GET /api/v1/opportunities` | — → `OpportunityListResponseDto` | 200, 401 |
| `POST /api/v1/auth/login` | `LoginDto` → `AuthResponseDto` | 200, 401, 429 |
| `POST /api/v1/auth/refresh` | `RefreshDto` → `AuthResponseDto` | 200, 401 |
| `PATCH /api/v1/negotiations/:id/counteroffer` | `CounterOfferDto` → `NegotiationResponseDto` | 200, 400, 403, 404, 409 |
| `POST /api/v1/escrow/deposit` | `EscrowDepositDto` → `EscrowResponseDto` | 201, 400, 403, 409 |
| `POST /api/v1/kyc/submit` | `KycSubmitDto` → `KycStatusDto` | 202, 400, 403 |
| `GET /api/v1/ai/analyze` (SSE) | — → `StreamingChunkDto` | 200, 401, 429 |

### 5.3 Verificações de Contrato

Para cada endpoint listado acima, os testes de contrato devem verificar:

- [ ] Todos os campos obrigatórios presentes na response.
- [ ] Tipos de dados corretos (string, number, UUID, ISO date).
- [ ] `correlation_id` presente em todas as respostas.
- [ ] Schema de erro seguindo `{ "error": { "code", "user_message", "correlation_id", "timestamp" } }`.
- [ ] Headers obrigatórios: `Content-Type: application/json`, `X-Correlation-ID`.
- [ ] Dados do Cedente **ausentes** em qualquer response acessível pelo Cessionário.

### 5.4 Exemplo de Teste de Contrato

```typescript
// Validação automática do schema OpenAPI gerado
it('should match OpenAPI contract for POST /api/v1/proposals', async () => {
  const response = await request(app.getHttpServer())
    .post('/api/v1/proposals')
    .send(validProposalPayload)
    .set('Authorization', `Bearer ${validToken}`)

  // Validar contra schema Zod compartilhado
  const parsed = ProposalResponseSchema.safeParse(response.body.data)
  expect(parsed.success).toBe(true)

  // Garantir ausência de dados do Cedente
  expect(response.body).not.toHaveProperty('cedente')
  expect(response.body).not.toHaveProperty('cedenteCpf')
  expect(response.body).not.toHaveProperty('cedenteName')
})
```

---

## 6. Testes E2E

### 6.1 Objetivo

Validar fluxos completos do usuário no browser, simulando a jornada real da aplicação de ponta a ponta.

### 6.2 Ferramentas

| Ferramenta | Uso |
|---|---|
| Playwright | Browser automation (Chromium + Firefox) |
| MSW (Mock Service Worker) | Mock de integrações externas em nível de rede |
| `@playwright/test` | Fixtures, expect, screenshots on failure |

### 6.3 Ambiente E2E

- URL base: ambiente de staging (`STAGING_URL` configurada no CI).
- Banco: instância de teste isolada, resetada antes de cada suíte.
- Integrações externas: todas mockadas via MSW (ZapSign, idwall, Celcoin sandbox).
- Autenticação: usuários de teste criados via seed antes da suíte.

[DECISÃO AUTÔNOMA — MSW para mock de integrações externas no E2E. Justificativa: intercepta requisições no nível de rede sem modificar código de produção, evita dependência de sandboxes externas (instáveis), mantém suíte determinística. Alternativa descartada: chamar sandboxes reais — risco de flaky test por instabilidade externa e custo de transação por execução.]

### 6.4 Reset e Isolamento

```typescript
// playwright.config.ts
use: {
  baseURL: process.env.E2E_BASE_URL,
  video: 'retain-on-failure',
  screenshot: 'only-on-failure',
}

// Cada teste usa fixture com usuario fresco:
test.beforeEach(async ({ page }) => {
  await seedTestUser({ kycStatus: 'APPROVED', proposals: [] })
  await page.goto('/login')
})

test.afterAll(async () => {
  await cleanupTestUsers()
})
```

### 6.5 Fluxos E2E Obrigatórios

| ID | Fluxo | Prioridade | Módulo |
|---|---|---|---|
| E2E-001 | Cadastro + Login + Google OAuth | P0 | Auth |
| E2E-002 | Onboarding KYC — submissão aprovada | P0 | KYC |
| E2E-003 | Onboarding KYC — submissão rejeitada | P0 | KYC |
| E2E-004 | Marketplace — listar e filtrar oportunidades | P0 | Marketplace |
| E2E-005 | Enviar proposta (Cessionário com KYC aprovado) | P0 | Proposta |
| E2E-006 | Tentativa de proposta sem KYC — bloqueio | P0 | Proposta |
| E2E-007 | Negociação — aceitar contraproposta | P1 | Negociação |
| E2E-008 | Negociação — rejeitar e encerrar | P1 | Negociação |
| E2E-009 | Depósito Escrow — fluxo completo | P0 | Escrow |
| E2E-010 | Formalização — assinar documentos via ZapSign (mock) | P1 | Formalização |
| E2E-011 | Fechamento da operação | P0 | Fechamento |
| E2E-012 | Solicitar reversão (dentro de 15 dias) | P1 | Reversão |
| E2E-013 | Chat com Analista de Oportunidades — streaming | P2 | IA |
| E2E-014 | Verificar que dados do Cedente não aparecem em nenhuma tela | P0 | Segurança/LGPD |
| E2E-015 | Re-autenticação para ação crítica (RN-005) | P0 | Auth |
| E2E-016 | Expiração de sessão por inatividade (30 min) | P1 | Auth |

### 6.6 Exemplo de Cenário E2E

```typescript
// tests/e2e/proposal.spec.ts
test('E2E-005: Cessionário com KYC aprovado envia proposta e recebe confirmação', async ({ page }) => {
  // Arrange: usuário com KYC aprovado e seed de oportunidade
  await seedOpportunity({ id: 'OPR-2026-0042', status: 'DISPONIVEL' })
  await loginAs(page, 'cessionario_kyc_approved@test.com')

  // Act: navegar ao marketplace e enviar proposta
  await page.goto('/oportunidades')
  await page.getByText('OPR-2026-0042').click()
  await page.getByRole('button', { name: 'Fazer Proposta' }).click()
  await page.getByLabel('Valor da proposta').fill('450000')
  await page.getByRole('button', { name: 'Confirmar Proposta' }).click()

  // Assert: feedback de sucesso e proposta aparece em "Minhas Propostas"
  await expect(page.getByText('Proposta enviada com sucesso')).toBeVisible()
  await page.goto('/minhas-propostas')
  await expect(page.getByText('OPR-2026-0042')).toBeVisible()
  await expect(page.getByText('Aguardando análise')).toBeVisible()

  // Garantir anonimato: nome do Cedente não aparece
  await expect(page.getByText(/cedente/i)).not.toBeVisible()
})
```

---

## 7. Cenários por Módulo

### 7.1 Módulo: Autenticação e Sessão

| # | Cenário | Tipo | Prioridade | RN/Contrato | Risco coberto |
|---|---|---|---|---|---|
| 1 | Login com email+senha válidos | Integração | P0 | RN-001 | Acesso não autorizado |
| 2 | Login com senha incorreta — 5ª tentativa → lockout 15min | Integração | P0 | RN-006 | Brute force |
| 3 | Refresh token válido gera novo par de tokens | Integração | P0 | RN-004 | Sessão expirada |
| 4 | Refresh token inválido retorna 401 | Unitário | P0 | RN-004 | Token roubado |
| 5 | Sessão expira após 30min de inatividade | Integração | P1 | RN-004 | Sessão abandonada |
| 6 | Re-autenticação exigida para ação crítica | E2E | P0 | RN-005 | Ação não autorizada |
| 7 | Google OAuth — fluxo PKCE completo | E2E | P1 | RN-003 | OAuth misconfigured |
| 8 | Token de acesso expirado é rejeitado com 401 | Unitário | P0 | RN-004 | Token antigo aceito |
| 9 | Logout invalida refresh token no Redis | Integração | P1 | RN-004 | Token reutilizado após logout |
| 10 | Rate limit de 5 req/min em `/auth/login` | Integração | P0 | RN-006 | DoS no endpoint de login |

### 7.2 Módulo: KYC

| # | Cenário | Tipo | Prioridade | RN/Contrato | Risco coberto |
|---|---|---|---|---|---|
| 1 | Submissão KYC enfileira job no RabbitMQ | Integração | P0 | RN-010 | Job perdido |
| 2 | Worker KYC processa aprovação idwall e atualiza status | Integração | P0 | RN-010 | Status incorreto |
| 3 | Worker KYC processa rejeição e notifica Cessionário | Integração | P0 | RN-011 | Notificação não enviada |
| 4 | KYC manual review quando idwall indisponível | Integração | P1 | RN-010 | idwall instável |
| 5 | Proposta bloqueada com KYC pendente | Unitário | P0 | RN-012 | Bypass de KYC |
| 6 | Proposta bloqueada com KYC rejeitado | Unitário | P0 | RN-012 | Bypass de KYC |
| 7 | Upload de documento via signed URL (não multipart) | Integração | P0 | Stack (D02) | Arquivo exposto publicamente |
| 8 | SLA de 5min para resultado do KYC | Integração | P1 | RN-010 | SLA violado |
| 9 | Notificação NOT-CES-02 enviada após resultado KYC | Integração | P1 | NOT-CES-02 | Notificação não enviada |
| 10 | Dados de selfie e documento não logados | Unitário | P0 | LGPD/D25 | Vazamento de dados |

### 7.3 Módulo: Marketplace e Oportunidades

| # | Cenário | Tipo | Prioridade | RN/Contrato | Risco coberto |
|---|---|---|---|---|---|
| 1 | Listar apenas oportunidades DISPONIVEL e COM_PROPOSTA | Integração | P0 | RN-017 | Oportunidade errada exibida |
| 2 | Dados do Cedente ausentes em todos os campos da response | Integração | P0 | RN-014 | Violação LGPD |
| 3 | Filtro por faixa de preço retorna apenas oportunidades corretas | Unitário | P1 | RN-018 | Filtro incorreto |
| 4 | Delta calculado corretamente (Tabela Atual − Tabela Contrato) | Unitário | P0 | RN-017 | Cálculo errado exposto ao Cessionário |
| 5 | Skeleton cards exibidos durante loading ≤ 3s | E2E | P2 | RN-017 (DEC-006) | UX degradada |
| 6 | Estado vazio exibido quando sem oportunidades | E2E | P2 | RN-052 | Tela quebrada |
| 7 | Ordenação "Recomendados pela IA" como padrão | Integração | P2 | RN-018 | Ordenação incorreta |
| 8 | Supabase Realtime notifica nova oportunidade em tempo real | E2E | P1 | RN-017 | Marketplace desatualizado |
| 9 | Cenário escolhido pelo Cedente não aparece em nenhum endpoint | Integração | P0 | RN-017 | Exposição de contrato do Cedente |
| 10 | RLS filtra oportunidades corretas por Cessionário logado | Integração | P0 | RN-013/RN-068 | Cross-contamination de dados |

### 7.4 Módulo: Propostas

| # | Cenário | Tipo | Prioridade | RN/Contrato | Risco coberto |
|---|---|---|---|---|---|
| 1 | Cessionário com KYC aprovado cria proposta (201) | Integração | P0 | RN-019 | Proposta não criada |
| 2 | Limite de 3 propostas simultâneas — 4ª retorna 409 | Integração | P0 | RN-020 | Spam de propostas |
| 3 | Limite de 10 propostas em 24h | Integração | P0 | RN-020 | Abuso de propostas |
| 4 | Proposta aceita transiciona oportunidade para EM_NEGOCIAÇÃO | Integração | P0 | RN-021 | Estado inconsistente |
| 5 | Proposta expira após prazo sem resposta | Unitário | P1 | RN-023 | Proposta zumbi |
| 6 | Cancelamento de proposta pelo Cessionário (modal confirmação) | E2E | P1 | RN-022 | Cancelamento acidental |
| 7 | Notificação NOT-CES-03 enviada ao criar proposta | Integração | P1 | NOT-CES-03 | Notificação não enviada |
| 8 | Notificação NOT-CES-04 ao receber resposta da proposta | Integração | P1 | NOT-CES-04 | Cessionário sem retorno |
| 9 | Proposta criada com idempotência (retry não cria duplicata) | Integração | P0 | D17 (idempotência) | Duplicata por retry |
| 10 | Valor mínimo da proposta validado no DTO | Unitário | P1 | RN-021 | Proposta com valor inválido |

### 7.5 Módulo: Escrow e Financeiro

| # | Cenário | Tipo | Prioridade | RN/Contrato | Risco coberto |
|---|---|---|---|---|---|
| 1 | Cálculo de Comissão Comprador com delta > 0 | Unitário | P0 | RN-037 | Cobrança incorreta |
| 2 | Cálculo de Comissão Comprador com delta ≤ 0 (submerso) | Unitário | P0 | RN-037 | Cobrança incorreta |
| 3 | Comissão Total = Preço Repasse + Comissão Comprador | Unitário | P0 | RN-037 | Valor total errado |
| 4 | Prazo de depósito: 10 dias úteis a partir da confirmação | Unitário | P0 | RN-028 | Prazo incorreto |
| 5 | Alerta NOT-CES-05 (48h antes do prazo) disparado | Integração | P0 | NOT-CES-05 | Cessionário perde prazo |
| 6 | Alerta NOT-CES-06 (no dia do prazo) disparado | Integração | P0 | NOT-CES-06 | Cessionário perde prazo |
| 7 | NOT-CES-05/06 não podem ser desativados pelo Cessionário | Unitário | P0 | RN-069 | Bypass de notificação crítica |
| 8 | Conta Escrow criada automaticamente ao entrar em Formalização | Integração | P1 | RN-028 | Conta não criada |
| 9 | Extensão de prazo (+5 dias úteis) requer aprovação Admin | Integração | P1 | RN-028 | Extensão não autorizada |
| 10 | Idempotency key evita duplo depósito Celcoin | Integração | P0 | D17 (Celcoin) | Dupla cobrança |

### 7.6 Módulo: Formalização e Fechamento

| # | Cenário | Tipo | Prioridade | RN/Contrato | Risco coberto |
|---|---|---|---|---|---|
| 1 | Webhook ZapSign válido (HMAC-SHA256) processa assinatura | Integração | P0 | D17 (ZapSign) | Webhook forjado |
| 2 | Webhook ZapSign inválido retorna 401 | Integração | P0 | D17 (ZapSign) | Webhook forjado aceito |
| 3 | Retry de webhook ZapSign (3x em 30min) | Integração | P1 | D17 (ZapSign) | Webhook perdido |
| 4 | Fechamento exige 4 critérios simultâneos | Unitário | P0 | RN-035 | Fechamento prematuro |
| 5 | Fechamento com 1 critério faltante é bloqueado | Unitário | P0 | RN-035 | Fechamento inválido |
| 6 | Notificação NOT-CES-11 ao fechar operação | Integração | P1 | NOT-CES-11 | Cessionário sem confirmação |
| 7 | Reversão disponível em até 15 dias corridos após fechamento | Unitário | P1 | RN-039 | Janela de reversão incorreta |
| 8 | Reversão fora do prazo bloqueada com mensagem correta | Unitário | P1 | RN-039 | Reversão indevida |
| 9 | Reembolso via Escrow processado após reversão aprovada | Integração | P0 | RN-040 | Reembolso não processado |
| 10 | Notificação NOT-CES-08 ao enviar documentos para assinatura | Integração | P1 | NOT-CES-08 | Cessionário não notificado |

### 7.7 Módulo: Analista de Oportunidades (IA)

| # | Cenário | Tipo | Prioridade | RN/Contrato | Risco coberto |
|---|---|---|---|---|---|
| 1 | Prompt injection detectado e bloqueado pelo InputSanitizerService | Unitário | P0 | RN-047/D19 | Manipulação do LLM |
| 2 | OutputFilterService bloqueia resposta com linguagem de FOMO | Unitário | P0 | RN-047/D19 | Resposta prejudicial ao Cessionário |
| 3 | Agente não acessa dados do Cedente (query filtrada por cessionario_id) | Integração | P0 | RN-049 | Violação de privacidade |
| 4 | Memória de sessão expira após 30min de inatividade | Unitário | P1 | D19 (LGPD) | Memória persistida indevidamente |
| 5 | Streaming SSE inicia em ≤ 5s (SLA individual) | E2E | P1 | RN-050 | SLA violado |
| 6 | Rate limit de 20 chamadas/min por usuário | Integração | P1 | D02 (ADR-001) | Abuso do endpoint LLM |
| 7 | Langfuse registra trace com userId hasheado (não real) | Unitário | P0 | D25/LGPD | userId exposto em logs de IA |
| 8 | Tool `getOpportunity` filtrado por cessionario_id | Unitário | P0 | RN-049 | Cross-contamination |
| 9 | Modelo fixado em `gpt-4-turbo-2024-04-09` | Unitário | P1 | D02 (Stack) | Modelo errado em produção |
| 10 | Resposta de análise de risco segue RiskScoreSchema (JSON Schema) | Unitário | P1 | D19 | Schema inválido retornado |

---

## 8. Dados de Teste

### 8.1 Factories

[DECISÃO AUTÔNOMA — factories dinâmicas com `@anatine/zod-mock` + `@faker-js/faker` para gerar dados realistas sem fixtures estáticas. Justificativa: fixtures estáticas ficam desatualizadas rapidamente; factories geram dados sempre consistentes com o schema atual. Alternativa descartada: fixtures JSON hardcoded — qualquer mudança de schema quebra múltiplos testes sem indicação clara.]

```typescript
// packages/test-utils/factories/cessionario.factory.ts
export const cessionarioFactory = {
  create: async (overrides: Partial<Cessionario> = {}) => {
    return prisma.cessionario.create({
      data: {
        id: `test-${faker.string.uuid()}`,
        name: faker.person.fullName(),
        email: faker.internet.email({ provider: 'test.repasse.com' }),
        cpf: faker.helpers.fake('###.###.###-##'), // mascarado
        kycStatus: 'PENDING',
        ...overrides,
      }
    })
  }
}
```

### 8.2 Dados Proibidos em Testes

| Dado | Motivo |
|---|---|
| CPF reais (validação de algoritmo) | Risco de cruzamento com dados reais |
| Tokens de produção | Comprometimento de segurança |
| Credenciais de sandbox reais (ZapSign, idwall) | Executar transações reais durante teste |
| Emails de usuários reais | LGPD e spam |
| Chaves de API de produção | Custo e segurança |

### 8.3 Mascaramento de Dados Sensíveis

```typescript
// CPF sempre mascarado nos factories de teste
cpf: '***.***.***-**' // Nunca CPF com dígitos verificáveis reais

// Email sempre com domínio de teste
email: faker.internet.email({ provider: 'test.repasse.com' })

// Nunca usar faker.finance.creditCardNumber() ou similares
```

### 8.4 Limpeza e Isolamento

| Estratégia | Quando usar |
|---|---|
| `TRUNCATE ... CASCADE` antes da suíte | Suítes E2E e integração completa |
| `DELETE WHERE id LIKE 'test-%'` após cada teste | Integração granular |
| `redis.flushDb()` antes da suíte | Testes de sessão e rate limit |
| Banco de dados separado (`test_db`) | Isolamento total |
| Prefixo `test-` em todos os IDs de factory | Identificação e limpeza |

---

## 9. Cobertura e Gates

### 9.1 Métricas de Cobertura

| Módulo | Threshold mínimo | Bloqueia merge |
|---|---|---|
| `CommissionService` (cálculo financeiro) | **100%** statements | ✅ Sim |
| `EscrowService` (prazos, estados) | **100%** statements | ✅ Sim |
| `AuthGuard`, `JwtGuard`, `KycGuard` | **95%** statements | ✅ Sim |
| `ProposalService`, `NegotiationService` | **80%** statements | ✅ Sim |
| `KycWorker`, `NotificationWorker` | **80%** statements | ✅ Sim |
| `InputSanitizerService`, `OutputFilterService` | **95%** statements | ✅ Sim |
| Frontend — componentes com lógica | **80%** branches | ✅ Sim |
| Cobertura geral do projeto | **75%** statements | Aviso (não bloqueia) |

### 9.2 Gates de Merge e Release

| Gate | Condição de bloqueio | Alvo |
|---|---|---|
| **Gate 1** — Merge em `develop` | Qualquer teste unitário ou integração falhando | Todo PR |
| **Gate 2** — Merge em `develop` | Type-check ou lint com erro | Todo PR |
| **Gate 3** — Merge em `develop` | Cobertura abaixo do threshold em módulo crítico | Todo PR com mudança em módulo crítico |
| **Gate 4** — Merge em `main` | Qualquer teste E2E P0 falhando | PRs `develop → main` |
| **Gate 5** — Deploy em staging | Suíte E2E completa verde | Pre-deploy |
| **Gate 6** — Deploy em produção | Smoke tests de produção verdes + aprovação manual | Pre-deploy prod |

### 9.3 Exemplo de Gate com Critério Objetivo

```yaml
# .github/workflows/ci.yml
- name: Run critical coverage check
  run: |
    pnpm test --coverage --coverageThreshold='{"CommissionService":{"statements":100},"EscrowService":{"statements":100}}'
  # Retorna exit code 1 se threshold não atingido → bloqueia merge
```

---

## 10. Testes de Regressão

### 10.1 Quando Rodar

| Trigger | Suíte |
|---|---|
| Todo PR para `develop` | Unitário + integração (obrigatório) |
| Todo PR para `main` | Unitário + integração + E2E (obrigatório) |
| Deploy em staging | Suíte E2E completa |
| Hotfix em `main` | Smoke tests + E2E críticos (P0) |
| Cron diário (staging) | Suíte completa |

### 10.2 Fluxos Sempre na Regressão

Os fluxos abaixo **nunca podem ser removidos** da suíte de regressão, independentemente de mudanças no produto:

1. Login + sessão + logout
2. KYC — submissão e resultado
3. Criação de proposta
4. Cálculo de comissão (todos os cenários de delta)
5. Depósito Escrow + alertas NOT-CES-05/06
6. Assinatura ZapSign (webhook HMAC-SHA256)
7. Fechamento de operação (4 critérios)
8. Anonimato do Cedente em todas as responses
9. Re-autenticação para ação crítica

### 10.3 Manutenção da Suíte

- Teste que falha intermitentemente (flaky) por 3 execuções consecutivas → quarentena imediata + issue obrigatória.
- Teste obsoleto (RN removida ou fluxo descontinuado) → marcar `test.skip()` com comentário e issue para remoção definitiva em até 5 dias.
- Nunca remover teste sem rastrear a RN que o originou e confirmar que a RN foi removida.

---

## 11. Testes Não-Funcionais

### 11.1 Performance

[DECISÃO AUTÔNOMA — k6 para testes de carga. Justificativa: scripting em JavaScript (alinhado com o stack Node.js do time), open-source, integração nativa com CI. Alternativa descartada: Gatling (JVM, curva de aprendizado maior).]

| Endpoint | Usuários simultâneos | p95 esperado | Critério de falha |
|---|---|---|---|
| `GET /api/v1/opportunities` | 100 | < 500ms | > 5% de erros ou p95 > 1s |
| `POST /api/v1/proposals` | 50 | < 300ms | > 1% de erros |
| `GET /api/v1/ai/analyze` (SSE) | 20 | < 5s para primeiro chunk | > 10s para primeiro chunk |

### 11.2 Acessibilidade

- `axe-core` integrado ao Playwright para verificar violações WCAG 2.1 AA nos fluxos críticos.
- Violações de nível A/AA bloqueiam E2E.
- Executar em: marketplace, formulário de proposta, stepper KYC, modal de re-autenticação.

### 11.3 Segurança Básica

| Verificação | Ferramenta | Frequência |
|---|---|---|
| Audit de dependências | `pnpm audit --audit-level=high` | Todo PR |
| OWASP Top 10 básico | Checklist manual | Pré-release |
| Headers de segurança (Helmet) | Supertest (verificar headers) | Integração |
| Dados do Cedente ausentes em responses | Testes de contrato | Todo PR |

---

## 12. Integração com CI/CD

### 12.1 Pipeline de Testes no CI

```
PR aberto (qualquer branch → develop)
    │
    ├── Lint + type-check (< 2min)
    ├── Testes unitários + cobertura (< 5min)
    └── Testes de integração (< 8min)
         │
         └── [todos devem passar para merge em develop]

PR: develop → main
    │
    ├── Todos os checks acima
    └── Testes E2E (Playwright, staging) (< 20min)
         │
         └── [todos E2E P0 devem passar para merge em main]

Deploy em staging
    │
    └── Suíte E2E completa + smoke tests

Deploy em produção
    │
    ├── Smoke tests de produção
    └── Aprovação manual obrigatória
```

### 12.2 Tempos Máximos de CI

| Suíte | Tempo máximo | Ação se exceder |
|---|---|---|
| Lint + type-check | 2 minutos | Investigar e otimizar imports |
| Unitário + cobertura | 5 minutos | Revisar testes pesados, usar `--reporter=dot` |
| Integração | 8 minutos | Paralelizar suítes |
| E2E (Playwright) | 20 minutos | Revisar fluxos lentos, aumentar paralelismo |
| Total PR (develop) | 15 minutos | Aceitável |
| Total PR (main) | 35 minutos | Aceitável |

### 12.3 Publicação de Resultados

- Relatório de cobertura: publicado como artefato e comentado no PR.
- Relatório E2E: screenshots e vídeos de falhas como artefato de CI.
- Histórico de flaky tests: registrado em issue com label `flaky-test`.

---

## 13. Backlog de Pendências

| ID | Item | Tipo | Prioridade |
|---|---|---|---|
| TEST-001 | Definir arquivo `vitest.config.ts` com thresholds de cobertura por módulo | Ação técnica | Alta |
| TEST-002 | Criar `packages/test-utils/` com factories, helpers e fixtures compartilhados | Ação técnica | Alta |
| TEST-003 | Configurar k6 para testes de carga dos endpoints críticos | Ação técnica | Média |
| TEST-004 | Criar seed script para ambiente E2E em staging | Ação técnica | Alta |
| TEST-005 | Verificar disponibilidade de sandbox ZapSign para testes E2E (em vez de mock) | [DEFINIÇÃO PENDENTE] — Opção A: mock MSW (determinístico, sem custo); Opção B: sandbox real ZapSign (mais realista, depende de contrato). Impacto: se webhook real não for testado em E2E, risco residual de falha de assinatura em produção. | Alta |

---

*Documento gerado pelo pipeline ShiftLabs v9.5 — Fase 2. Executor: Claude Code Desktop.*
