# 27 - Plano de Testes

## Repasse Seguro — Módulo Admin

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | QA e Engenharia |
| **Escopo** | Estratégia de testes por camada, cenários por módulo, dados de teste, gates e cobertura |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Status** | Aprovado |
| **Dependências** | D01 RN · D02 Stacks · D05 PRD · D12 ERD · D14 Especificações Técnicas · D16 Documentação de API |

---

> 📌 **TL;DR**
>
> - **Pirâmide:** 70% unitário, 20% integração, 5% contrato, 5% E2E — com foco em fluxos de negócio críticos.
> - **Frameworks:** Vitest (backend + frontend), Supertest (integração API), Playwright (E2E).
> - **Gates de merge:** 80% cobertura de branches em serviços críticos (auth, cases, escrow); todos os testes passando.
> - **Gate de release:** 90% cenários E2E críticos passando + smoke test de produção.
> - **5 fluxos críticos com E2E obrigatório:** Login 2FA, Criação de Caso, Fechamento, Distribuição Escrow, Assinatura ZapSign.
> - **Seed controlado:** factory + fixtures por camada; reset automático entre testes de integração e E2E.
> - **CI:** unitários + integração em todo PR; E2E apenas em merge para `develop` e `main`.

---

## 1. Estratégia Geral

O Repasse Seguro é uma plataforma financeira com fluxos irreversíveis: uma Distribuição Escrow executada incorretamente não tem desfazimento simples; um Fechamento com critérios incompletos gera risco jurídico. A estratégia de testes prioriza **risco financeiro e de integridade de dados** acima de cobertura percentual.

**Perfil de risco:**
- **Crítico (E2E obrigatório):** fluxos financeiros, fechamento, autenticação, agentes de IA
- **Alto (integração obrigatória):** transições de status, validações de RN, notificações
- **Médio (unitário suficiente):** cálculos de comissão, formatação de UI, utilitários

**Princípio:** cobertura de linha ≠ qualidade. Um serviço com 100% de cobertura em getters e 0% em fluxo de fechamento está mal testado.

---

## 2. Pirâmide de Testes

| Camada | Proporção | Ferramentas | Velocidade | Quando roda |
|---|---|---|---|---|
| **Unitário** | 70% | Vitest | < 30s | Todo commit (pre-commit) + CI |
| **Integração** | 20% | Vitest + Supertest + Prisma + testcontainers | 2–5min | Todo PR |
| **Contrato** | 5% | Vitest + schemas Zod | < 1min | Todo PR |
| **E2E** | 5% | Playwright | 5–15min | Merge para `develop` e `main` |
| **Manual exploratorio** | Complementar | — | Variável | Pré-release e sprint review |

**Justificativa da proporção:** testes unitários são rápidos e isolados — feedback imediato para o dev. Testes de integração validam que as camadas se conectam. E2E são lentos e frágeis — reservados apenas para os 5 fluxos críticos que não podem falhar em produção.

---

## 3. Testes Unitários

### 3.1 Ferramentas e Padrão

- **Backend:** Vitest + `vi.fn()` para mocks + `@nestjs/testing` para módulos NestJS
- **Frontend:** Vitest + `@testing-library/react` + `msw` (mock service worker) para API
- **Mobile:** Vitest + `@testing-library/react-native`

### 3.2 Convenções

- **Naming:** `{subject}.spec.ts` colocado ao lado do arquivo testado (colocation)
- **Estrutura:** `describe('ClassName/function') → it('should {behavior} when {condition}')`
- **Isolamento:** toda dependência externa (banco, Redis, fila) deve ser mockada via `vi.mock()`
- **Mocks permitidos:** `PrismaService`, `RedisService`, `RabbitMQService`, qualquer API externa

### 3.3 Quando NÃO usar testes unitários

- Para lógica que só faz sentido integrada (ex: constraint de unicidade do banco → teste de integração)
- Para fluxos end-to-end → E2E

### 3.4 Cobertura Mínima por Módulo (branches)

| Módulo | Cobertura mínima |
|---|---|
| `AuthService` | 85% |
| `CasesService` | 80% |
| `EscrowService` | 80% |
| `AiAgentService` | 75% |
| Outros serviços | 70% |
| Componentes React (lógica) | 60% |

### 3.5 Exemplo de Cenário Unitário

```typescript
// auth.service.spec.ts
describe('AuthService.login()', () => {
  it('should return temp_token when 2FA is enabled', async () => {
    // Arrange
    const mockUser = { id: 'uuid', two_fa_enabled: true, is_active: true };
    prisma.user.findFirst.mockResolvedValue(mockUser);
    bcrypt.compare.mockResolvedValue(true);

    // Act
    const result = await authService.login('user@test.com', 'password');

    // Assert
    expect(result).toEqual({ requires_2fa: true, temp_token: expect.any(String) });
    expect(redis.setex).toHaveBeenCalledWith(
      `rs:2fa_pending:${mockUser.id}`, 300, 'true'
    );
  });

  it('should throw AccountLockedError after 5 failed attempts', async () => {
    const mockUser = { ...baseUser, failed_login_attempts: 5 };
    prisma.user.findFirst.mockResolvedValue(mockUser);
    bcrypt.compare.mockResolvedValue(false);

    await expect(authService.login('user@test.com', 'wrong')).rejects.toThrow(AccountLockedError);
    expect(prisma.user.update).toHaveBeenCalledWith(expect.objectContaining({
      data: expect.objectContaining({ locked_until: expect.any(Date) }),
    }));
  });
});
```

---

## 4. Testes de Integração

### 4.1 Setup

- Banco: PostgreSQL via `testcontainers` (container isolado por suíte) — sem contaminar banco de desenvolvimento
- Redis: `ioredis-mock` ou Redis via testcontainers
- RabbitMQ: mock de publicação — validar apenas que a mensagem é publicada, não a entrega
- API: `@nestjs/testing` + Supertest para requisições HTTP reais

### 4.2 Categorias e Critérios

| Categoria | O que testa | Critério de aprovação |
|---|---|---|
| Endpoints de auth | Login, refresh, logout, verify-2fa | Status codes corretos, tokens no formato esperado, Redis populado |
| Endpoints de casos | CRUD, transições de status | Status codes, banco atualizado, history registrado |
| Validações de RN | Regras de negócio em service layer | BIZ_001 lançado corretamente, estado não modificado |
| Escrow e distribuição | Fluxo de escrow — criação, depósito, distribuição | EscrowAccount atualizado, EscrowTransaction criada, jobs publicados |
| Webhooks | ZapSign e Celcoin | HMAC-SHA256 verificado, status atualizado, job publicado |
| Prisma e migrações | Constraints, índices, soft-delete middleware | Violação de unicidade retorna `P2002`, soft-delete filtra `deleted_at` |

### 4.3 Exemplo de Cenário de Integração

```typescript
// cases.e2e-spec.ts (integração de API)
describe('PATCH /v1/cases/:id/status', () => {
  it('should return 422 when transition is invalid', async () => {
    const case_ = await factory.createCase({ status: 'CONCLUIDO' });

    const response = await request(app.getHttpServer())
      .patch(`/v1/cases/${case_.id}/status`)
      .set('Authorization', `Bearer ${analistaToken}`)
      .send({ to_status: 'EM_TRIAGEM', version: 1 });

    expect(response.status).toBe(422);
    expect(response.body.type).toContain('BIZ_002');
    // Banco não deve ter sido alterado
    const unchanged = await prisma.case.findUnique({ where: { id: case_.id } });
    expect(unchanged.status).toBe('CONCLUIDO');
  });
});
```

---

## 5. Testes de Contrato

### 5.1 O que é testado

- Resposta de API (schema Zod) — garante que o frontend nunca receba campo inesperado
- Payloads de webhook entrada (ZapSign, Celcoin) — garante que parser não quebre em payload real
- Mensagens RabbitMQ — garante que worker recebe o schema que o publisher envia

### 5.2 Contratos Obrigatórios

| Contrato | Schema | Validação |
|---|---|---|
| `POST /auth/login` response | `LoginResponseSchema` | Zod parse no teste |
| `POST /cases` response | `CaseResponseSchema` | Zod parse no teste |
| `POST /webhooks/zapsign` payload | `ZapsignWebhookSchema` | Zod parse no teste |
| `POST /webhooks/celcoin` payload | `CelcoinWebhookSchema` | Zod parse no teste |
| RabbitMQ job `notifications.*` | `NotificationJobSchema` | Zod parse no teste |

```typescript
// contracts/login.contract.spec.ts
it('POST /auth/login deve retornar schema válido (sem 2FA)', async () => {
  const response = await request(app).post('/v1/auth/login').send({ email, password });

  const parsed = LoginResponseSchema.safeParse(response.body.data);
  expect(parsed.success).toBe(true);
  expect(parsed.data).toMatchObject({
    access_token: expect.any(String),
    refresh_token: expect.any(String),
    user: { id: expect.any(String), role: expect.any(String) },
  });
});
```

---

## 6. Testes E2E

### 6.1 Fluxos Críticos (obrigatórios)

| Fluxo | Prioridade | RN Rastreada |
|---|---|---|
| Login com 2FA (TOTP) | Crítico | RN-001, RN-005 |
| Criação de Caso com config snapshot | Crítico | RN-010, RN-111 |
| Fechamento de Caso (4 critérios) | Crítico | RN-021, RN-023 |
| Distribuição de Conta Escrow | Crítico | RN-085, RN-086 |
| Assinatura via ZapSign (webhook) | Crítico | RN-040 a RN-047 |

### 6.2 Setup E2E

- Ambiente: staging dedicado com banco isolado
- Seed: executar `pnpm --filter api prisma db seed:test` antes de cada suíte
- Reset: truncar tabelas de domínio (não audit) após cada suíte
- Autenticação: login real com usuário de teste (não mock de token)
- ZapSign/Celcoin: mocked via `msw` no nível de rede — sem chamadas reais ao sandbox

### 6.3 Exemplo de Cenário E2E

```typescript
// e2e/case-closing.spec.ts (Playwright)
test('Fechamento de Caso — 4 critérios cumpridos', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[name=email]', 'analista@repasseseguro.com.br');
  await page.fill('[name=password]', 'Test@1234');
  await page.click('[type=submit]');

  // Navegar para caso em formalização
  await page.goto(`/cases/${SEED_CASE_ID}/formalization`);

  // Verificar que botão de fechar está desabilitado (critérios incompletos)
  await expect(page.locator('[data-testid=confirm-close-btn]')).toBeDisabled();

  // Simular cumprimento dos 4 critérios via API seed
  await seedApi.fulfillClosingCriteria(SEED_CASE_ID);
  await page.reload();

  // Verificar que botão está habilitado com animação de pulsação
  await expect(page.locator('[data-testid=confirm-close-btn]')).toBeEnabled();
  await expect(page.locator('[data-testid=confirm-close-btn]')).toHaveClass(/pulse/);

  // Confirmar fechamento
  await page.click('[data-testid=confirm-close-btn]');
  await page.click('[data-testid=modal-confirm]');

  // Verificar redirecionamento e status
  await expect(page).toHaveURL(`/cases/${SEED_CASE_ID}`);
  await expect(page.locator('[data-testid=case-status-badge]')).toContainText('FECHAMENTO');
});
```

---

## 7. Cenários por Módulo

### 7.1 Módulo: Autenticação

| Cenário | Tipo | Prioridade | RN | Risco coberto |
|---|---|---|---|---|
| Login bem-sucedido sem 2FA | Unitário + Integração | Crítico | RN-001 | Acesso não autorizado |
| Login com 2FA — código correto | E2E | Crítico | RN-001, RN-003 | Bypass de 2FA |
| 5 tentativas incorretas → bloqueio | Unitário + Integração | Crítico | RN-005 | Brute force |
| Conta bloqueada — exibe `locked_until` | Integração | Alto | RN-005 | UX de segurança |
| Refresh token rotation | Unitário + Integração | Crítico | RN-134 | Roubo de token |
| Token reusado detectado — invalida sessão | Unitário | Crítico | RN-134 | Replay attack |
| Logout invalida blacklist Redis | Integração | Alto | RN-004 | Sessão fantasma |
| RBAC: ANALISTA acessando rota de MASTER | Integração | Alto | RN-010 | Escalação de privilégio |

### 7.2 Módulo: Casos

| Cenário | Tipo | Prioridade | RN | Risco coberto |
|---|---|---|---|---|
| Criar caso — config snapshot gerado | Integração | Crítico | RN-111 | Config alterada após criação não afeta caso |
| Transição de status válida | Unitário + Integração | Crítico | RN-012 a RN-020 | Estado inconsistente |
| Transição de status inválida → 422 | Unitário | Alto | BIZ_002 | Pipeline corrompido |
| Conflito de versão (Optimistic Lock) → 409 | Integração | Crítico | RN-130 | Dados sobrescritos |
| Fechamento com 4 critérios | E2E | Crítico | RN-023 | Fechamento indevido |
| Fechamento sem critérios → bloqueio | Integração | Crítico | RN-023 | Compliance |
| Histórico de status registrado | Integração | Alto | RN-129 | Auditoria |
| Soft delete — caso não retorna em listagem | Integração | Alto | RN-130 | Dados fantasma |

### 7.3 Módulo: Escrow e Financeiro

| Cenário | Tipo | Prioridade | RN | Risco coberto |
|---|---|---|---|---|
| Criar conta escrow — idempotência | Integração | Crítico | RN-085 | Conta duplicada |
| Depósito confirmado via webhook Celcoin | Integração | Crítico | RN-085 | Depósito não registrado |
| HMAC-SHA256 webhook inválido → rejeitar | Integração | Crítico | D17 | Webhook forjado |
| Distribuição com escrow bloqueado → 403 | Unitário + Integração | Crítico | RN-086 | Distribuição indevida |
| Comissão calculada corretamente (cenário A/B/C/D) | Unitário | Crítico | RN-088 | Valor errado |
| Distribuição — idempotência com `external_id` | Integração | Crítico | RN-086 | Dupla distribuição |
| Desbloqueio de escrow requer MASTER | Integração | Alto | RN-087 | Escalação de privilégio |

### 7.4 Módulo: Formalização (ZapSign)

| Cenário | Tipo | Prioridade | RN | Risco coberto |
|---|---|---|---|---|
| Criar envelope ZapSign | Integração + Contrato | Crítico | RN-040 | Envelope não criado |
| Webhook ZapSign SIGNED — atualiza status | Integração | Crítico | RN-044 | Assinatura não registrada |
| Webhook ZapSign REFUSED | Integração | Alto | RN-044 | Fluxo de recusa |
| Reenvio de envelope | Integração | Normal | RN-047 | Usuário sem link |
| Critério de anuência salvo | Unitário | Alto | RN-033 | Formalização incompleta |

### 7.5 Módulo: Supervisão IA

| Cenário | Tipo | Prioridade | RN | Risco coberto |
|---|---|---|---|---|
| Agente age autonomamente com confiança ≥ 90% | Unitário | Crítico | RN-095.a | Ação não autorizada |
| Agente enfileira para revisão com confiança 70–89% | Unitário | Crítico | RN-095.a | Bypass de Human Gate |
| Takeover manual — agente pausado para o caso | Integração | Crítico | RN-095 | Agente ignorando takeover |
| Takeover automático após 3 ações idênticas em 60s | Unitário | Crítico | RN-096 | Loop não detectado |
| Schema de saída inválido → falha crítica | Unitário | Crítico | RN-096 | Dados corrompidos persistidos |
| Limite de 5 takeovers simultâneos | Integração | Alto | RN-095 | Sobrecarga do operador |

---

## 8. Dados de Teste

### 8.1 Seed de Teste

```typescript
// prisma/seed-test.ts
export async function seedTestData() {
  // Usuários por role
  const master = await createUser({ role: 'MASTER', email: 'master@test.rs' });
  const coordenador = await createUser({ role: 'COORDENADOR', email: 'coord@test.rs' });
  const analista = await createUser({ role: 'ANALISTA', email: 'analista@test.rs' });
  const gestor = await createUser({ role: 'GESTOR_FINANCEIRO', email: 'gestor@test.rs' });

  // Caso em cada status crítico
  const caseEmTriagem = await createCase({ status: 'EM_TRIAGEM', assigned_analyst_id: analista.id });
  const caseEmFormalizacao = await createCase({ status: 'EM_FORMALIZACAO' });
  const caseReadyToClose = await createCase({ status: 'EM_FORMALIZACAO' });
  await fulfillClosingCriteria(caseReadyToClose.id);

  // Conta escrow com depósito confirmado
  const escrow = await createEscrowAccount({ case_id: caseEmFormalizacao.id, status: 'ATIVO' });
}
```

### 8.2 Factories

Cada entidade tem uma factory com valores padrão + overrides:

```typescript
// tests/factories/case.factory.ts
export function createCase(overrides: Partial<CaseInput> = {}): CaseInput {
  return {
    cedente_id: faker.string.uuid(),
    scenario: 'B',
    property_address: 'Rua Teste, 123',
    contract_table_value: '250000.00',
    paid_amount: '50000.00',
    assigned_analyst_id: faker.string.uuid(),
    ...overrides,
  };
}
```

### 8.3 Isolamento e Reset

- **Unitários:** sem banco — 100% mockado
- **Integração:** banco limpo por `beforeAll` + transação revertida por `afterEach` onde possível
- **E2E:** seed completo por `beforeAll` + truncate por `afterAll`
- **Dados sensíveis:** nunca usar CPFs reais — usar `faker.cpf()` via `@faker-js/faker` (extensão brasileira)

---

## 9. Cobertura e Gates

### 9.1 Thresholds por Camada

| Camada | Métrica | Threshold | Bloqueia merge? |
|---|---|---|---|
| Unitário (branches) | Módulos críticos (auth, cases, escrow, ai) | 80% | Sim |
| Unitário (branches) | Outros módulos | 65% | Sim |
| Integração | Endpoints críticos documentados | 100% cobertos | Sim |
| E2E | 5 fluxos críticos | Todos passando | Sim (merge em `main`) |
| Contrato | Schemas obrigatórios | 100% | Sim |

### 9.2 Gates de Merge

**PR → `develop`:**
- Type-check passando (`pnpm typecheck`)
- Lint passando (`pnpm lint`)
- Unitários: todos passando + coverage ≥ threshold
- Testes de integração: todos passando
- Testes de contrato: todos passando

**PR → `main` (release):**
- Todos os gates de `develop`
- Testes E2E dos 5 fluxos críticos: todos passando
- Smoke test de staging: 5 endpoints-chave respondendo dentro do SLO

---

## 10. Testes de Regressão

### 10.1 Quando Rodar

| Evento | Escopo da Regressão |
|---|---|
| Merge de bugfix em `develop` | Módulo afetado + testes de integração relacionados |
| Merge de feature em `develop` | Suite completa de unitários + integração |
| Merge em `main` (release) | Suite completa + E2E dos 5 fluxos críticos |
| Rollback de deploy | Suite completa + análise do diff que causou o rollback |

### 10.2 Manutenção da Suite

- Teste falhando cronicamente (flaky > 3×/semana): abrir issue + marcar com `.skip` temporário + priorizar correção
- Feature removida: remover testes correspondentes no mesmo PR
- Regra de negócio alterada: atualizar teste no mesmo PR da mudança de código
- Novo bug em produção: sempre acompanhado de teste de regressão que o reproduz

---

## 11. Testes Não-Funcionais

### 11.1 Performance (básico)

**`[DECISÃO AUTÔNOMA]`** — Testes de carga mínimos via Autocannon em CI antes de release:

```bash
# Smoke load test — 100 req/s por 30s
npx autocannon -c 100 -d 30 https://staging.repasseseguro.com.br/health
```

Critério: latência p95 < 500ms, zero erros 5xx.

### 11.2 Acessibilidade

- `@axe-core/playwright` em todos os E2E para detectar violações WCAG 2.1 AA automaticamente.
- Critério: zero violações críticas (`critical`) ou sérias (`serious`) nos 5 fluxos E2E.

### 11.3 Segurança Básica

| Check | Ferramenta | Frequência |
|---|---|---|
| Dependências com CVE crítico | `pnpm audit --audit-level=high` | Todo CI |
| Exposição de secrets em código | `detect-secrets` (pre-commit) | Todo commit |
| Headers de segurança | Teste de integração manual | Pré-release |

---

## 12. Integração com CI/CD

| Etapa CI | Suítes que rodam | Tempo máximo | Bloqueia se falhar? |
|---|---|---|---|
| `push` para feature branch | Unitários apenas | 2min | Não (warning) |
| `PR → develop` | Unitários + Integração + Contrato | 8min | Sim |
| `merge → develop` | Unitários + Integração + Contrato + E2E | 20min | Sim (alerta se E2E falhar) |
| `merge → main` | Suite completa + Smoke test staging | 25min | Sim (rollback automático) |

---

## 13. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — pirâmide de testes, 5 fluxos E2E críticos, cenários por módulo (auth, cases, escrow, ZapSign, IA), seed controlado, gates de merge e release, integração CI. |

---

## 14. Backlog de Pendências

| Item | Marcador | Seção | Justificativa | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Testes de carga completos | `[DECISÃO AUTÔNOMA]` | 11.1 | Autocannon para smoke test suficiente na fase inicial. K6 ou Locust completo reavaliar após go-live com volume real. | Performance em escala | DevOps | Decidido — revisão pós go-live |
