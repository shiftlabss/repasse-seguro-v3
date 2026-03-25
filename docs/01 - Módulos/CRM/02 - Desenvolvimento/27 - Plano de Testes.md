# 27 - Plano de Testes

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | QA e Engenharia |
| **Escopo** | Estratégia de QA por camada, casos de teste detalhados, cobertura e gates de release |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | 02 Stacks CRM · 01.1 a 01.5 Regras de Negócio |

---

> **TL;DR**
>
> - **Pirâmide:** unitário (Jest/Vitest) → integração (Supertest + banco real) → E2E (Playwright).
> - **Cobertura mínima:** 80% unitário em módulos críticos, 60% integração, fluxos críticos 100% E2E.
> - **Frameworks:** Jest (unitário backend) · Vitest (unitário frontend) · Supertest (integração API) · Playwright (E2E).
> - **Fluxos críticos com E2E obrigatório:** criação de Caso, avanço de pipeline, assinatura ZapSign, geração de relatório, alertas SLA.
> - **Mínimo 20 casos de teste** documentados neste plano.

---

## 1. Estratégia Geral

O CRM do Repasse Seguro é uma ferramenta interna crítica para operação de cessões imobiliárias. Fluxos como avanço de status do pipeline, geração de alertas de SLA, criação de envelopes ZapSign e geração de relatórios têm impacto direto nos negócios. A estratégia de testes prioriza **risco operacional e integridade de dados** acima de cobertura percentual bruta.

**Perfil de risco:**
- **Crítico (E2E obrigatório):** pipeline de Casos (criação, avanço, conclusão), assinatura eletrônica (ZapSign), alertas de SLA, relatório semanal
- **Alto (integração obrigatória):** cálculo de comissão, transições de status, monitor de SLA, notificações
- **Médio (unitário suficiente):** formatação de datas, utilitários, lógica de filtros, mascaramento de dados

---

## 2. Pirâmide de Testes

| Camada | Ferramentas | Cobertura mínima | Quando roda |
|---|---|---|---|
| **Unitário** | Jest (`apps/api`) · Vitest (`apps/web-crm`) | 80% branches em módulos críticos; 60% demais | Todo commit (pre-commit hook) + CI em todo PR |
| **Integração** | Supertest + NestJS Testing + Prisma (banco real Supabase staging) | 60% endpoints críticos documentados | Todo PR para `develop` |
| **E2E** | Playwright | 100% dos 5 fluxos críticos passando | Merge para `develop` e `main` |
| **Contrato** | Zod schema parse | 100% contratos obrigatórios | Todo PR |

---

## 3. Módulos Críticos e Cobertura

| Módulo | Cobertura unitária mínima | Integração obrigatória |
|---|---|---|
| `CasesService` | 80% | Sim — transições de status, criação com snapshot |
| `CommissionsService` | 85% | Sim — cálculo nos 4 cenários (A/B/C/D) |
| `SlaMonitorService` | 80% | Sim — detecção de alertas, disparo de notificações |
| `NegotiationsService` | 75% | Sim — criação de proposta, aceite, contraproposta |
| `ZapSignService` | 75% | Sim — criação de envelope, processamento de webhook |
| `ReportsService` | 70% | Sim — geração de relatório semanal |
| `ContactsService` | 70% | Sim — CRUD completo |
| `ActivitiesService` | 70% | Sim — registro de atividade por tipo |
| Componentes React (lógica) | 60% | N/A |

---

## 4. Setup de Testes

### 4.1 Unitário (Backend — Jest)

```typescript
// Estrutura padrão de teste unitário
// apps/api/modules/crm/cases/cases.service.spec.ts

describe('CasesService', () => {
  let service: CasesService;
  let prisma: DeepMockProxy<PrismaClient>;
  let redis: jest.Mocked<RedisService>;
  let queue: jest.Mocked<RabbitMQService>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        CasesService,
        { provide: PrismaService, useValue: mockDeep<PrismaClient>() },
        { provide: RedisService, useValue: { get: jest.fn(), set: jest.fn(), del: jest.fn() } },
        { provide: RabbitMQService, useValue: { publish: jest.fn() } },
      ],
    }).compile();

    service = module.get(CasesService);
    prisma = module.get(PrismaService);
    redis = module.get(RedisService);
    queue = module.get(RabbitMQService);
  });
});
```

### 4.2 Integração (Supertest)

```typescript
// Setup com banco Supabase staging isolado
// apps/api/test/setup-integration.ts

beforeAll(async () => {
  app = await createTestApp();  // NestJS app com banco real
  await prisma.$executeRaw`TRUNCATE TABLE cases, contacts, activities, negotiations CASCADE`;
  await seedTestData(prisma);
});

afterAll(async () => {
  await prisma.$disconnect();
  await app.close();
});
```

### 4.3 E2E (Playwright)

```typescript
// apps/web-crm/e2e/setup.ts
import { test as base } from '@playwright/test';

export const test = base.extend({
  authenticatedPage: async ({ page }, use) => {
    await page.goto('/login');
    await page.fill('[name=email]', 'analista@crm.dev');
    await page.fill('[name=password]', 'CrmAnalista@123');
    await page.click('[type=submit]');
    await page.waitForURL('/dashboard');
    await use(page);
  },
});
```

---

## 5. Casos de Teste Detalhados

### TC-CRM-001 — Criação de Caso com snapshot de configuração

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-001 |
| **Cenário** | Criar novo Caso e validar que o snapshot de parâmetros do sistema é gerado |
| **Tipo** | Integração |
| **Prioridade** | Crítico |
| **RN** | RN-006, RN-147 |
| **Pré-condições** | Usuário autenticado como ANALISTA_RS; parâmetros do sistema configurados em `system_parameters` |
| **Passos** | 1. POST `/v1/cases` com dados válidos; 2. Verificar resposta; 3. Consultar Caso criado no banco |
| **Resultado esperado** | HTTP 201; campo `config_snapshot` no Caso contém cópia dos parâmetros do sistema vigentes; status inicial = `CADASTRO` |

---

### TC-CRM-002 — Avanço de status válido no pipeline

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-002 |
| **Cenário** | Avançar Caso de CADASTRO para SIMULACAO |
| **Tipo** | Unitário + Integração |
| **Prioridade** | Crítico |
| **RN** | RN-039 a RN-044 |
| **Pré-condições** | Caso existente com status CADASTRO; usuário ANALISTA_RS |
| **Passos** | 1. PATCH `/v1/cases/:id/status` com `{ to_status: "SIMULACAO" }`; 2. Verificar banco |
| **Resultado esperado** | HTTP 200; `case.status = "SIMULACAO"`; histórico de transição registrado em `case_status_history`; `updated_by` = UUID do usuário |

---

### TC-CRM-003 — Transição de status inválida retorna erro

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-003 |
| **Cenário** | Tentar retroceder Caso de MATCH para CADASTRO (transição proibida) |
| **Tipo** | Unitário |
| **Prioridade** | Crítico |
| **RN** | RN-039 a RN-044 |
| **Pré-condições** | Caso com status MATCH |
| **Passos** | 1. Chamar `casesService.advanceStatus(caseId, 'CADASTRO', userId)` |
| **Resultado esperado** | Lança `InvalidStatusTransitionError`; banco não alterado; log WARN emitido |

---

### TC-CRM-004 — Cálculo de comissão — Cenário A (sem parceiro)

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-004 |
| **Cenário** | Calcular comissão do CRM sem parceiro externo envolvido |
| **Tipo** | Unitário |
| **Prioridade** | Crítico |
| **RN** | RN-050 a RN-057 |
| **Pré-condições** | Caso sem parceiro; valor de transação R$ 200.000; percentual de comissão RS = 3% (configurado em `system_parameters`) |
| **Passos** | 1. Chamar `commissionsService.calculate(caseId, transactionValue, 'SCENARIO_A')` |
| **Resultado esperado** | Comissão RS = R$ 6.000,00; comissão parceiro = R$ 0,00; total = R$ 6.000,00; registro salvo em `commissions` |

---

### TC-CRM-005 — Cálculo de comissão — Cenário B (com parceiro)

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-005 |
| **Cenário** | Calcular comissão com parceiro externo — divisão entre RS e parceiro |
| **Tipo** | Unitário |
| **Prioridade** | Crítico |
| **RN** | RN-050 a RN-057 |
| **Pré-condições** | Caso com parceiro externo vinculado; valor de transação R$ 200.000; percentual total = 3%; split RS/parceiro = 60/40 |
| **Passos** | 1. Chamar `commissionsService.calculate(caseId, transactionValue, 'SCENARIO_B')` |
| **Resultado esperado** | Comissão RS = R$ 3.600,00; comissão parceiro = R$ 2.400,00; total = R$ 6.000,00 |

---

### TC-CRM-006 — Monitor de SLA detecta Caso em risco

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-006 |
| **Cenário** | SlaMonitorService detecta Caso com deadline vencido e cria alerta |
| **Tipo** | Unitário |
| **Prioridade** | Crítico |
| **RN** | RN-107, RN-108 |
| **Pré-condições** | Caso com `sla_deadline` = ontem; status != CONCLUIDO; nenhum alerta existente para este Caso |
| **Passos** | 1. Mock `prisma.cases.findMany` retornando caso vencido; 2. Chamar `slaMonitor.checkDeadlines()` |
| **Resultado esperado** | `prisma.slaAlerts.create` chamado com `severity = "HIGH"`; `queue.publish('crm.sla_alerts', ...)` chamado; log INFO emitido |

---

### TC-CRM-007 — Alerta de SLA não duplicado para mesmo Caso

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-007 |
| **Cenário** | SlaMonitorService NÃO cria alerta duplicado se já existe alerta ativo para o Caso |
| **Tipo** | Unitário |
| **Prioridade** | Alto |
| **RN** | RN-107 |
| **Pré-condições** | Alerta ativo já existe em `sla_alerts` para o Caso |
| **Passos** | 1. Mock `prisma.slaAlerts.findFirst` retornando alerta existente; 2. Chamar `slaMonitor.checkDeadlines()` |
| **Resultado esperado** | `prisma.slaAlerts.create` NÃO chamado; `queue.publish` NÃO chamado |

---

### TC-CRM-008 — Criação de envelope ZapSign para formalização

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-008 |
| **Cenário** | Criar envelope de assinatura eletrônica ao avançar para FORMALIZACAO |
| **Tipo** | Integração |
| **Prioridade** | Crítico |
| **RN** | RN-189 |
| **Pré-condições** | Caso em ANUENCIA com todos os critérios cumpridos; mock do ZapSign API configurado |
| **Passos** | 1. PATCH `/v1/cases/:id/status` com `{ to_status: "FORMALIZACAO" }`; 2. Verificar chamada ao ZapSign mock |
| **Resultado esperado** | HTTP 200; ZapSign mock recebe POST para criar envelope com 3 signatários (Cedente, Cessionário, RS); `case.zapsign_envelope_id` preenchido |

---

### TC-CRM-009 — Webhook ZapSign SIGNED atualiza status da formalização

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-009 |
| **Cenário** | Receber webhook de assinatura concluída do ZapSign e atualizar Caso |
| **Tipo** | Integração |
| **Prioridade** | Crítico |
| **RN** | RN-189 |
| **Pré-condições** | Caso em FORMALIZACAO com `zapsign_envelope_id` preenchido; HMAC válido |
| **Passos** | 1. POST `/v1/webhooks/zapsign` com payload `{ status: "SIGNED", token: "..." }` e header `X-Zapsign-Signature: [hmac_valido]` |
| **Resultado esperado** | HTTP 200; campo `all_signed_at` preenchido; critério de assinatura marcado como cumprido; log INFO emitido |

---

### TC-CRM-010 — Webhook ZapSign com HMAC inválido rejeitado

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-010 |
| **Cenário** | Rejeitar webhook ZapSign com assinatura HMAC inválida |
| **Tipo** | Integração |
| **Prioridade** | Crítico |
| **RN** | RN-196 |
| **Pré-condições** | Endpoint de webhook ZapSign ativo |
| **Passos** | 1. POST `/v1/webhooks/zapsign` com HMAC incorreto no header |
| **Resultado esperado** | HTTP 401; banco não alterado; log WARN com `msg: "HMAC inválido"` |

---

### TC-CRM-011 — Registro de atividade de ligação em Caso

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-011 |
| **Cenário** | Analista RS registra ligação realizada em um Caso |
| **Tipo** | Integração |
| **Prioridade** | Alto |
| **RN** | RN-098 a RN-102 |
| **Pré-condições** | Caso existente; usuário ANALISTA_RS autenticado |
| **Passos** | 1. POST `/v1/cases/:id/activities` com `{ type: "CALL", notes: "Contato realizado", duration_minutes: 10 }` |
| **Resultado esperado** | HTTP 201; atividade salva em `activities` com `case_id`, `created_by` e timestamp corretos |

---

### TC-CRM-012 — Analista RS não acessa Casos de outro Analista

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-012 |
| **Cenário** | RBAC — Analista RS não pode ver Casos atribuídos a outro Analista |
| **Tipo** | Integração |
| **Prioridade** | Crítico |
| **RN** | RN-012, RN-193 |
| **Pré-condições** | 2 usuários ANALISTA_RS; Caso A atribuído ao Analista 1; autenticado como Analista 2 |
| **Passos** | 1. GET `/v1/cases` como Analista 2 |
| **Resultado esperado** | HTTP 200; resposta não contém o Caso A do Analista 1; apenas Casos do Analista 2 |

---

### TC-CRM-013 — COORDENADOR_RS acessa todos os Casos

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-013 |
| **Cenário** | Coordenador RS tem visibilidade total de todos os Casos da equipe |
| **Tipo** | Integração |
| **Prioridade** | Alto |
| **RN** | RN-012, RN-193 |
| **Pré-condições** | Casos atribuídos a diferentes Analistas RS; autenticado como COORDENADOR_RS |
| **Passos** | 1. GET `/v1/cases` como COORDENADOR_RS |
| **Resultado esperado** | HTTP 200; resposta inclui todos os Casos (de todos os Analistas) |

---

### TC-CRM-014 — Usuário sem papel adequado não acessa Settings

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-014 |
| **Cenário** | ANALISTA_RS não consegue acessar configurações do sistema (endpoint de Admin) |
| **Tipo** | Integração |
| **Prioridade** | Crítico |
| **RN** | RN-193 |
| **Pré-condições** | Autenticado como ANALISTA_RS |
| **Passos** | 1. GET `/v1/settings/system-parameters` com token de ANALISTA_RS |
| **Resultado esperado** | HTTP 403; mensagem de erro de autorização |

---

### TC-CRM-015 — Busca e filtro de Contatos por tipo

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-015 |
| **Cenário** | Filtrar Contatos por tipo = CESSIONARIO e verificar mascaramento de dados |
| **Tipo** | Integração |
| **Prioridade** | Alto |
| **RN** | RN-093 a RN-097, RN-012 |
| **Pré-condições** | 5 Contatos seedados: 2 CEDENTE, 3 CESSIONARIO |
| **Passos** | 1. GET `/v1/contacts?type=CESSIONARIO` |
| **Resultado esperado** | HTTP 200; 3 Contatos retornados; CPF mascarado (ex: `***.XXX.XXX-**`); e-mail mascarado |

---

### TC-CRM-016 — Geração do relatório semanal

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-016 |
| **Cenário** | Gerar relatório semanal e verificar dados consolidados |
| **Tipo** | Integração |
| **Prioridade** | Alto |
| **RN** | RN-110 |
| **Pré-condições** | Banco com Casos e Atividades do período; usuário ADMIN_RS |
| **Passos** | 1. POST `/v1/admin/reports/weekly/trigger` com `{ period: "2026-03-17/2026-03-23" }` |
| **Resultado esperado** | HTTP 202; job enfileirado em `crm.weekly_reports`; após processamento, relatório disponível em `/v1/reports/weekly/latest` |

---

### TC-CRM-017 — Upload de documento ao Dossiê

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-017 |
| **Cenário** | Fazer upload de documento ao Dossiê de um Caso |
| **Tipo** | Integração |
| **Prioridade** | Alto |
| **RN** | RN-058 a RN-063 |
| **Pré-condições** | Caso existente; usuário ANALISTA_RS com acesso; Supabase Storage configurado |
| **Passos** | 1. POST `/v1/cases/:id/dossier/documents` com `multipart/form-data` contendo PDF |
| **Resultado esperado** | HTTP 201; registro em `dossier_documents` com `storage_path` (UUID v4); `status = "PENDING_REVIEW"` |

---

### TC-CRM-018 — Caso cancelado não permite avanço de status

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-018 |
| **Cenário** | Tentar avançar status de Caso CANCELADO — deve ser bloqueado |
| **Tipo** | Unitário |
| **Prioridade** | Alto |
| **RN** | RN-039 a RN-044 |
| **Pré-condições** | Caso com status = CANCELADO |
| **Passos** | 1. Chamar `casesService.advanceStatus(caseId, 'CADASTRO', userId)` |
| **Resultado esperado** | Lança `CaseAlreadyTerminatedError`; banco não alterado |

---

### TC-CRM-019 — E2E: Criação de Caso via interface do CRM

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-019 |
| **Cenário** | Analista RS cria novo Caso pelo formulário do CRM |
| **Tipo** | E2E (Playwright) |
| **Prioridade** | Crítico |
| **RN** | RN-006 a RN-009 |
| **Pré-condições** | Usuário `analista@crm.dev` autenticado no CRM (staging) |
| **Passos** | 1. Navegar para `/pipeline`; 2. Clicar em "Novo Caso"; 3. Preencher formulário com dados válidos; 4. Submeter |
| **Resultado esperado** | Redirecionamento para `/cases/[novo-id]`; badge de status = "CADASTRO"; toast de sucesso visível |

---

### TC-CRM-020 — E2E: Avanço de Caso no Pipeline Kanban

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-020 |
| **Cenário** | Coordenador RS avança Caso de SIMULACAO para VERIFICACAO via pipeline |
| **Tipo** | E2E (Playwright) |
| **Prioridade** | Crítico |
| **RN** | RN-039 a RN-044 |
| **Pré-condições** | Caso em SIMULACAO visível no pipeline; autenticado como COORDENADOR_RS |
| **Passos** | 1. Navegar para `/pipeline`; 2. Localizar card do Caso em "Simulação"; 3. Clicar em "Avançar para Verificação"; 4. Confirmar no modal |
| **Resultado esperado** | Card se move para coluna "Verificação"; badge atualizado; toast de confirmação; badge de stage no detalhe do Caso atualizado |

---

### TC-CRM-021 — E2E: Geração de relatório semanal via interface

| Campo | Valor |
|---|---|
| **ID** | TC-CRM-021 |
| **Cenário** | Admin RS aciona geração manual do relatório semanal pela interface |
| **Tipo** | E2E (Playwright) |
| **Prioridade** | Alto |
| **RN** | RN-110 |
| **Pré-condições** | Autenticado como ADMIN_RS; dados de Casos na semana configurada |
| **Passos** | 1. Navegar para `/reports`; 2. Clicar em "Gerar Relatório"; 3. Selecionar período; 4. Confirmar |
| **Resultado esperado** | Toast "Relatório em processamento"; após < 30s, relatório disponível na listagem com link de download |

---

## 6. Dados de Teste

### 6.1 Seed de Teste

```typescript
// apps/api/prisma/seed-test.ts
export async function seedCrmTestData(prisma: PrismaClient) {
  // Usuários por papel
  const adminRs = await createUser(prisma, { role: 'ADMIN_RS', email: 'admin@crm.test' });
  const coordenador = await createUser(prisma, { role: 'COORDENADOR_RS', email: 'coord@crm.test' });
  const analista1 = await createUser(prisma, { role: 'ANALISTA_RS', email: 'analista1@crm.test' });
  const analista2 = await createUser(prisma, { role: 'ANALISTA_RS', email: 'analista2@crm.test' });

  // Contatos
  const cedente = await createContact(prisma, { type: 'CEDENTE', assigned_to: analista1.id });
  const cessionario = await createContact(prisma, { type: 'CESSIONARIO' });

  // Casos em diferentes estágios
  const caseEmCadastro = await createCase(prisma, {
    status: 'CADASTRO',
    assigned_analyst_id: analista1.id,
    cedente_id: cedente.id,
  });

  const caseEmMatch = await createCase(prisma, {
    status: 'MATCH',
    assigned_analyst_id: analista1.id,
  });

  const caseEmFormalizacao = await createCase(prisma, {
    status: 'FORMALIZACAO',
    assigned_analyst_id: analista2.id,
    zapsign_envelope_id: 'env-test-uuid',
  });

  const caseSlaVencido = await createCase(prisma, {
    status: 'SIMULACAO',
    assigned_analyst_id: analista1.id,
    sla_deadline: new Date(Date.now() - 24 * 60 * 60 * 1000), // ontem
  });

  // Parâmetros do sistema
  await prisma.systemParameters.upsert({
    where: { key: 'commission_rate_rs' },
    update: {},
    create: { key: 'commission_rate_rs', value: '0.03', description: 'Taxa de comissão RS' },
  });
}
```

### 6.2 Factories

```typescript
// apps/api/test/factories/case.factory.ts
export function buildCase(overrides: Partial<CaseInput> = {}): CaseInput {
  return {
    status: 'CADASTRO',
    transaction_value: '200000.00',
    sla_deadline: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 dias
    config_snapshot: {},
    ...overrides,
  };
}
```

### 6.3 Dados Proibidos em Testes

- Nunca usar CPF reais — usar gerador de CPF válido: `faker.helpers.fromRegExp(/[0-9]{3}\.[0-9]{3}\.[0-9]{3}-[0-9]{2}/)`
- Nunca usar nomes reais de pessoas
- Nunca usar e-mails reais (exceto contas de teste isoladas)
- Nunca usar tokens de API de produção em testes

---

## 7. Gates de Qualidade

### 7.1 Gate de Merge (PR → develop)

- TypeScript compila sem erros: `pnpm typecheck`
- Lint passando: `pnpm lint`
- Testes unitários: todos passando + cobertura ≥ threshold por módulo
- Testes de integração: todos passando
- Testes de contrato: todos passando

### 7.2 Gate de Release (PR → main)

- Todos os gates de `develop`
- Testes E2E dos 5 fluxos críticos passando (TC-CRM-019, TC-CRM-020, TC-CRM-021 + ZapSign webhook + SLA alert)
- Smoke test de staging: endpoints `/health`, `/v1/cases`, `/v1/pipeline`, `/v1/contacts` respondendo em < 500ms
- `pnpm audit` sem vulnerabilidades High/Critical

### 7.3 Cobertura Mínima (CI bloqueia se abaixo)

```json
// jest.config.json (apps/api)
{
  "coverageThreshold": {
    "./src/modules/crm/cases/": { "branches": 80 },
    "./src/modules/crm/commissions/": { "branches": 85 },
    "./src/modules/crm/sla/": { "branches": 80 },
    "./src/modules/crm/": { "branches": 70 }
  }
}
```

---

## 8. Integração com CI/CD

| Evento CI | Suítes que rodam | Tempo máximo |
|---|---|---|
| `push` para feature branch | Unitários apenas | 3 min |
| PR → `develop` | Unitários + Integração + Contrato | 10 min |
| Merge → `develop` | Unitários + Integração + Contrato + E2E | 20 min |
| Merge → `main` | Suite completa + Smoke test staging | 25 min |

---

## 9. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — estratégia de testes por camada (Jest/Vitest/Supertest/Playwright), 21 casos de teste detalhados (TC-CRM-001 a TC-CRM-021), cobertura mínima por módulo, seed de testes, factories, gates de merge e release, integração CI. |
