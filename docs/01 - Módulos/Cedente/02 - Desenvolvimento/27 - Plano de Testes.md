# 27 - Plano de Testes

| Campo | Valor |
|---|---|
| **Nome do Documento** | 27 - Plano de Testes |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Produto** | Repasse Seguro — Módulo Cedente |

---

> **TL;DR — Estratégia de Testes do Módulo Cedente**
>
> - **Pirâmide:** 70% unitário / 20% integração / 8% E2E / 2% manual exploratório.
> - **Ferramentas:** Vitest (unitário + integração backend), Playwright (E2E), React Native Testing Library (mobile), Supertest (API).
> - **Fluxos críticos obrigatórios:** Cadastro Completo de Imóvel → Wizard de Cenários → Proposta (aceite/recusa/contraproposta) → Assinatura ZapSign → Escrow → Guardião do Retorno.
> - **Gates:** cobertura ≥ 80% em módulos críticos; 100% nos cálculos de comissão (Cenários B/C/D) e Escrow; E2E verde para todos os fluxos críticos antes de merge em `main`.
> - **Isolamento LGPD:** nenhum dado real de Cedente usado em testes; isolamento por `cedente_id` validado em toda suíte.
> - **CI:** unitário + integração rodam em todo PR; E2E roda em PRs para `main` e em staging pré-deploy.
> - **Bloqueio de merge:** qualquer falha em unitário, integração ou lint bloqueia `develop`; qualquer falha em E2E bloqueia `main`.

---

## 1. Estratégia Geral

O Módulo Cedente é a porta de entrada do negócio do Repasse Seguro. Sem Cedente cadastrado e com dossiê aprovado, não há oferta, não há proposta e não há receita. A criticidade dos testes reflete esse papel central: falhas nos cálculos de comissão ou no fluxo de Escrow têm impacto financeiro direto; falhas no isolamento de dados entre Cedentes constituem violação de LGPD.

**Perfil de risco do módulo:**

| Área | Criticidade | Razão |
|---|---|---|
| Cálculo de comissão — Cenários B, C, D | P0 — CRÍTICO | Impacto financeiro direto. Erro gera cobrança indevida ou prejuízo ao Cedente. |
| Escrow — depósito, reversão, distribuição | P0 — CRÍTICO | Movimentação real de dinheiro. Falha bloqueia o Fechamento. |
| Isolamento LGPD entre Cedentes | P0 — CRÍTICO | Vazamento de dados viola a Lei Geral de Proteção de Dados. |
| Formalização — assinatura ZapSign | P1 — ALTO | Falha bloqueia o Fechamento e a receita do negócio. |
| Guardião do Retorno — simulações e respostas | P1 — ALTO | Falha prejudica a experiência do Cedente no produto. |
| Fluxo de propostas (aceite/recusa/contraproposta) | P1 — ALTO | Core do produto; falha afeta diretamente a receita. |
| Escalonamento de cenário | P1 — ALTO | Falha pode travar o caso em cenário sem demanda. |
| Gestão do dossiê (upload, validação) | P2 — MÉDIO | UX degradado, mas falha não bloqueia dados financeiros. |
| Notificações e lembretes | P2 — MÉDIO | Falha reduz engajamento mas não trava fluxo principal. |
| Dashboard e visualizações | P3 — BAIXO | UX degradado sem impacto financeiro direto. |

**Escopo de teste:**

- Backend: NestJS — todos os módulos de negócio do Cedente
- Frontend: React + Vite — fluxos críticos e formulários (wizard de cadastro, propostas, assinaturas)
- Mobile: React Native + Expo — notificações push, upload via câmera, propostas, Guardião
- Integrações: ZapSign, Escrow (parceiro bancário/fintech) — mocks em unitário/integração; sandbox em E2E

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
| Unitário | 70% | Lógica de negócio, cálculos de comissão, simulação de cenários, escalonamento, guards, pipes | Vitest |
| Integração | 20% | Endpoints reais, banco de dados Prisma, filas, workers, webhooks ZapSign e Escrow com mock | Vitest + Supertest |
| E2E | 8% | Fluxos completos do Cedente no browser e mobile | Playwright + RNTL |
| Manual exploratório | 2% | Novos fluxos, edge cases, UX crítico pré-release | Checklist manual |

**Justificativa da proporção:**

[DECISÃO AUTÔNOMA — 70/20/8/2. O Módulo Cedente possui lógica financeira complexa (4 cenários de comissão, simulação de retorno, período de reversão Escrow). Alta cobertura unitária protege os cálculos sem overhead de E2E. E2E focado em jornadas críticas evita suíte frágil e lenta. Alternativa descartada: 50/30/20 com mais E2E — aumentaria tempo de CI sem ganho proporcional dado o isolamento garantido por unitários.]

---

## 3. Testes Unitários

### 3.1 Objetivo

Verificar a lógica de negócio isolada de dependências externas. Foco em funções puras, services, cálculos de comissão, simulação de cenários de retorno, guards de autenticação e RBAC, validações de upload e transformações de dados.

### 3.2 Ferramentas

| Ferramenta | Uso |
|---|---|
| Vitest | Runner, assertions, coverage |
| `vi.mock()` | Mock de dependências (Prisma, Redis, ZapSign client, Escrow client) |
| Testing Library (React) | Renderização de componentes frontend — wizard, propostas, simulador |
| React Native Testing Library | Componentes mobile — upload câmera, notificações, chat Guardião |
| `@nestjs/testing` | `Test.createTestingModule()` para providers isolados |

### 3.3 Cobertura Mínima

| Área | Cobertura mínima | Observação |
|---|---|---|
| Cálculo de comissão — Cenários B, C, D | **100%** | Zero tolerância a erro financeiro |
| Cálculo Valor Distrato Referência | **100%** | Base do cálculo de comissão |
| Lógica de Escrow (prazos, estados, reversão) | **100%** | Impacto financeiro direto |
| Simulador de Cenários (Etapa 3 do wizard) | **100%** | Decisão do Cedente baseia-se neste cálculo |
| Guards de autenticação e RBAC | **95%** | Segurança e isolamento de dados |
| Services de negócio (propostas, escalonamento, dossiê) | **80%** | Fluxo core |
| Services auxiliares (notificações, logs) | **70%** | Suporte |
| Componentes frontend com lógica (wizard, formulários) | **80%** | Formulários e validações |
| Utilitários e helpers | **90%** | Alta reutilização |

### 3.4 Padrão e Naming Convention

```typescript
// Formato: describe('<NomeDoService>') > describe('<nomeDoMetodo>') > it('<cenário>')
describe('CommissionCedenteService', () => {
  describe('calculateCedenteCommission', () => {
    it('should return 20% of (valorRecuperado - valorDistratoRef) for Cenário B', () => { ... })
    it('should return 20% of (valorRecuperado - valorDistratoRef) for Cenário C', () => { ... })
    it('should return 20% of (valorRecuperado - valorDistratoRef) for Cenário D', () => { ... })
    it('should return R$ 0 for Cenário A', () => { ... })
    it('should throw ValidationError when valorPago is missing', () => { ... })
  })
})
```

**Regras:**

- Um `describe` por classe/service/component.
- Um `it` por cenário (feliz + alternativo + erro).
- Mocks declarados no topo do arquivo, nunca inline em cada `it`.
- Não testar implementação interna — testar comportamento observável.
- `console.log` proibido em testes.
- Dados de Cedente em testes: sempre factories isoladas com dados fictícios.

### 3.5 Casos Críticos — Cálculo de Comissão

```typescript
// src/modules/commission/commission-cedente.service.spec.ts

describe('CommissionCedenteService', () => {

  describe('calculateCedenteCommission', () => {

    // Cenário B: Cedente recupera 100% do valor pago
    it('should calculate 20% of (valorRecuperado - valorDistratoRef) for Cenário B', () => {
      const input = {
        cenario: 'B',
        valorPagoCedente: 200_000,
        valorRecuperado: 200_000,
        // valorDistratoRef = 50% × 200_000 = 100_000
        // comissão = 20% × (200_000 - 100_000) = 20_000
      }
      expect(service.calculateCedenteCommission(input)).toBe(20_000)
    })

    // Cenário C: Cedente recupera 100% + 30%
    it('should calculate 20% of (valorRecuperado - valorDistratoRef) for Cenário C', () => {
      const input = {
        cenario: 'C',
        valorPagoCedente: 200_000,
        valorRecuperado: 260_000, // 200k + 30%
        // valorDistratoRef = 50% × 200_000 = 100_000
        // comissão = 20% × (260_000 - 100_000) = 32_000
      }
      expect(service.calculateCedenteCommission(input)).toBe(32_000)
    })

    // Cenário D: Cedente recupera 100% + 50%
    it('should calculate 20% of (valorRecuperado - valorDistratoRef) for Cenário D', () => {
      const input = {
        cenario: 'D',
        valorPagoCedente: 200_000,
        valorRecuperado: 300_000, // 200k + 50%
        // valorDistratoRef = 50% × 200_000 = 100_000
        // comissão = 20% × (300_000 - 100_000) = 40_000
      }
      expect(service.calculateCedenteCommission(input)).toBe(40_000)
    })

    // Cenário A: sem comissão
    it('should return R$ 0 for Cenário A (sem comissão)', () => {
      const input = {
        cenario: 'A',
        valorPagoCedente: 200_000,
        valorRecuperado: 0,
      }
      expect(service.calculateCedenteCommission(input)).toBe(0)
    })

    // Erro de input
    it('should throw ValidationError when valorPagoCedente is zero or negative', () => {
      const input = { cenario: 'B', valorPagoCedente: 0, valorRecuperado: 100_000 }
      expect(() => service.calculateCedenteCommission(input)).toThrow('valorPagoCedente must be positive')
    })

  })

  describe('calculateValorDistratoReferencia', () => {
    it('should return exactly 50% of valorPagoCedente', () => {
      expect(service.calculateValorDistratoReferencia(200_000)).toBe(100_000)
    })
    it('should handle values with cents correctly', () => {
      expect(service.calculateValorDistratoReferencia(150_500)).toBe(75_250)
    })
  })

})
```

### 3.6 Casos Críticos — Escalonamento de Cenário

```typescript
// src/modules/escalonamento/escalonamento.service.spec.ts

describe('EscalonamentoService', () => {

  describe('validateEscalonamento', () => {
    it('should allow D → C (one step down)', () => {
      expect(service.validateEscalonamento('D', 'C')).toBe(true)
    })
    it('should block D → B (skip not allowed)', () => {
      expect(() => service.validateEscalonamento('D', 'B')).toThrow('Escalonamento não pode pular cenários')
    })
    it('should block D → A (skip not allowed)', () => {
      expect(() => service.validateEscalonamento('D', 'A')).toThrow('Escalonamento não pode pular cenários')
    })
    it('should block escalonamento when cenario atual is A (mínimo)', () => {
      expect(() => service.validateEscalonamento('A', 'A')).toThrow('Cenário A é o mínimo — não é possível escalonar')
    })
  })

  describe('checkCooldown', () => {
    it('should block if last escalonamento was less than 7 days ago', () => {
      const lastEscalonamento = new Date(Date.now() - 3 * 24 * 60 * 60 * 1000) // 3 dias atrás
      expect(service.checkCooldown(lastEscalonamento)).toBe(false)
    })
    it('should allow if last escalonamento was more than 7 days ago', () => {
      const lastEscalonamento = new Date(Date.now() - 8 * 24 * 60 * 60 * 1000) // 8 dias atrás
      expect(service.checkCooldown(lastEscalonamento)).toBe(true)
    })
  })

})
```

### 3.7 Casos Críticos — Escrow

```typescript
// src/modules/escrow/escrow-cedente.service.spec.ts

describe('EscrowCedenteService', () => {

  describe('calculateDistribuicao', () => {
    it('should distribute valorLiquidoCedente after 15 days with no reversal', () => {
      const escrow = { valorTotal: 300_000, comissaoRS: 40_000, dateFechamento: somePastDate }
      expect(service.calculateDistribuicao(escrow).valorLiquidoCedente).toBe(260_000)
    })
    it('should block distribution if still within 15-day reversal period', () => {
      const recentDate = new Date(Date.now() - 10 * 24 * 60 * 60 * 1000) // 10 dias atrás
      expect(() => service.distribuirValores({ dateFechamento: recentDate })).toThrow('Período de reversão ainda ativo')
    })
    it('should return full estorno when desistência is approved within 15 days', () => {
      const result = service.processarDesistencia({ valorTotal: 300_000 })
      expect(result.estornoTotal).toBe(300_000)
      expect(result.valorLiquidoCedente).toBe(0)
    })
  })

})
```

---

## 4. Testes de Integração

### 4.1 Objetivo

Verificar a integração real entre as partes do sistema: endpoints HTTP, banco de dados Prisma, filas de processamento, webhooks ZapSign e Escrow com mock/stub, e workers assíncronos.

### 4.2 Setup

```typescript
// Ambiente de integração:
// - PostgreSQL real (Docker Compose)
// - Redis real (Docker Compose)
// - Serviços externos: ZapSign, Escrow → SEMPRE mockados via MSW ou vi.mock()
// - Supabase Storage: mockado via vi.mock('@supabase/supabase-js')

beforeAll(async () => {
  await app.get(PrismaService).$executeRaw`TRUNCATE TABLE casos, documentos, propostas, envelopes_assinatura, escrow_contas CASCADE`
  await redis.flushDb()
})

afterEach(async () => {
  await prisma.$executeRaw`DELETE FROM casos WHERE id LIKE 'test-%'`
})
```

### 4.3 Endpoints Críticos — Casos

| ID | Endpoint | Método | Cenário testado | Resultado esperado |
|---|---|---|---|---|
| INT-001 | `/api/v1/cedente/casos` | POST | Cadastro de imóvel com dados válidos | HTTP 201, caso criado com status `cadastro_realizado` |
| INT-002 | `/api/v1/cedente/casos` | POST | Duplicidade — mesmo imóvel com caso ativo | HTTP 409, mensagem de bloqueio |
| INT-003 | `/api/v1/cedente/casos/:id` | GET | Cedente acessa seu próprio caso | HTTP 200, dados do caso |
| INT-004 | `/api/v1/cedente/casos/:id` | GET | Cedente tenta acessar caso de outro Cedente | HTTP 403, isolamento LGPD |
| INT-005 | `/api/v1/cedente/casos/:id/cancelar` | POST | Cancelamento antes do Fechamento | HTTP 200, status muda para `cancelado` |
| INT-006 | `/api/v1/cedente/casos/:id/rascunho` | GET | Rascunho disponível dentro de 30 dias | HTTP 200, rascunho retornado |
| INT-007 | `/api/v1/cedente/casos/:id/rascunho` | GET | Rascunho após 30 dias | HTTP 404, rascunho descartado automaticamente |

### 4.4 Endpoints Críticos — Propostas

| ID | Endpoint | Método | Cenário testado | Resultado esperado |
|---|---|---|---|---|
| INT-008 | `/api/v1/cedente/propostas/:id/aceitar` | POST | Aceite com confirmação dupla | HTTP 200, caso muda para `em_formalizacao` |
| INT-009 | `/api/v1/cedente/propostas/:id/aceitar` | POST | Aceite com proposta já expirada | HTTP 409, mensagem de proposta expirada |
| INT-010 | `/api/v1/cedente/propostas/:id/recusar` | POST | Recusa de proposta | HTTP 200, caso retorna para `disponivel_para_compradores` |
| INT-011 | `/api/v1/cedente/propostas/:id/contraproposta` | POST | Contraproposta acima do piso do cenário | HTTP 200, contraproposta enviada ao Admin |
| INT-012 | `/api/v1/cedente/propostas/:id/contraproposta` | POST | Contraproposta abaixo do piso do cenário | HTTP 422, mensagem de valor abaixo do mínimo |
| INT-013 | `/api/v1/cedente/propostas` | GET | Listagem sem dados de identificação do Cessionário | HTTP 200, sem CPF/nome/email do Cessionário |

### 4.5 Endpoints Críticos — Documentos (Dossiê)

| ID | Endpoint | Método | Cenário testado | Resultado esperado |
|---|---|---|---|---|
| INT-014 | `/api/v1/cedente/documentos/:id/upload` | POST | Upload de PDF válido (< 10 MB) | HTTP 200, documento com status `em_analise` |
| INT-015 | `/api/v1/cedente/documentos/:id/upload` | POST | Upload com MIME inválido (ex: `.exe`) | HTTP 422, rejeição com mensagem de formato |
| INT-016 | `/api/v1/cedente/documentos/:id/upload` | POST | Upload de arquivo > 10 MB | HTTP 413, mensagem de limite de tamanho |
| INT-017 | `/api/v1/cedente/documentos/:id/upload` | POST | Tentativa de substituir documento já `verificado` | HTTP 403, bloqueio de imutabilidade |
| INT-018 | `/api/v1/cedente/documentos` | GET | Dossiê completo → caso avança para `em_analise` | HTTP 200, transição de estado confirmada |

### 4.6 Webhooks ZapSign

| ID | Cenário | Payload | Resultado esperado |
|---|---|---|---|
| INT-019 | Webhook de assinatura concluída — Termo de Cadastro | `{ "event": "document.signed", "doc_id": "...", "signer": "cedente" }` | Documento muda para `assinado`; caso avança para `em_analise` se dossiê completo |
| INT-020 | Webhook com HMAC-SHA256 inválido | Payload válido com assinatura incorreta | HTTP 401, webhook rejeitado |
| INT-021 | Webhook duplicado (idempotência) | Mesmo `doc_id` enviado duas vezes | HTTP 200, segundo evento ignorado sem duplicação de estado |
| INT-022 | Webhook de assinatura — Instrumento de Cessão | `{ "event": "document.signed", "doc_id": "...", "signer": "cedente" }` | Verificação de todas as assinaturas obrigatórias antes de avançar |

### 4.7 Webhooks Escrow

| ID | Cenário | Payload | Resultado esperado |
|---|---|---|---|
| INT-023 | Depósito confirmado na Conta Escrow | `{ "event": "escrow.deposit_confirmed", "caso_id": "..." }` | Status da Conta Escrow muda para `deposito_confirmado` |
| INT-024 | Distribuição automática após 15 dias | `{ "event": "escrow.distributed", "caso_id": "..." }` | Caso muda para `concluido`; comprovante gerado |
| INT-025 | Estorno por desistência aprovada | `{ "event": "escrow.reversed", "caso_id": "..." }` | Conta Escrow muda para `estornada`; notificação enviada ao Cedente |
| INT-026 | Webhook Escrow com payload malformado | JSON inválido | HTTP 400, erro tratado sem crash do serviço |

---

## 5. Testes E2E

### 5.1 Objetivo

Verificar os fluxos completos do Cedente no browser e no aplicativo mobile, simulando as ações reais do usuário da interface até a resposta final do sistema.

### 5.2 Ferramentas

| Ferramenta | Uso |
|---|---|
| Playwright | E2E web (Chrome, Firefox) |
| React Native Testing Library | E2E mobile (Expo — React Native) |
| MSW (Mock Service Worker) | Mock de APIs externas em E2E |
| Faker.js | Geração de dados fictícios por suíte |

### 5.3 Configuração

```typescript
// playwright.config.ts
export default defineConfig({
  testDir: './e2e',
  fullyParallel: false, // E2E financeiro: sequencial para evitar conflito de estado
  retries: 1,
  workers: 2,
  reporter: [['html'], ['junit', { outputFile: 'e2e-results.xml' }]],
  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
  },
})

// Antes de cada suíte: reset do banco e seeds de dados de teste
beforeAll(async () => {
  await resetTestDatabase()
  await seedCedenteTestData()
})
```

### 5.4 Suíte E2E-001 — Cadastro Completo de Imóvel

**Pré-condição:** Cedente autenticado sem casos ativos.

| Passo | Ação do usuário | Resultado esperado | RN rastreada |
|---|---|---|---|
| 1 | Acessa "Cadastrar Imóvel" na sidebar | Wizard abre na Etapa 1 (dados do imóvel) | RN-021 |
| 2 | Preenche todos os campos da Etapa 1 e avança | Etapa 2 desbloqueada | RN-023 |
| 3 | Preenche valores na Etapa 2 (valores pagos) | Simulador da Etapa 3 calcula os 4 cenários | RN-021 |
| 4 | Permanece na Etapa 3 por menos de 10 segundos e tenta avançar | Botão "Avançar" desabilitado com timer regressivo | RN-021 |
| 5 | Aguarda 10 segundos na Etapa 3 | Botão "Avançar" habilitado | RN-021 |
| 6 | Avança para Etapa 4 sem selecionar cenário | Mensagem de validação: "Selecione o cenário de retorno desejado para continuar." | RN-022 |
| 7 | Seleciona Cenário C e marca o checkbox de confirmação | Card do cenário com borda de destaque; botão "Avançar" habilitado | RN-022 |
| 8 | Preenche Etapa 5 (confirmação) e clica "Confirmar Cadastro" | Tela de sucesso exibida; Termo de Cadastro gerado no ZapSign | RN-024 |
| 9 | Acessa "Assinaturas" e assina o Termo de Cadastro | Documento muda para "Assinado"; caso avança para "Em análise" ao completar dossiê | RN-024 |

**Validações de isolamento LGPD:**
- Nenhum dado de outro Cedente visível na listagem de casos.
- Tentativa de acessar `/casos/:id` de outro Cedente retorna 403.

### 5.5 Suíte E2E-002 — Fluxo de Propostas

**Pré-condição:** Cedente com caso no status "Disponível para compradores".

| Passo | Ação do usuário | Resultado esperado | RN rastreada |
|---|---|---|---|
| 1 | Sistema envia proposta via Admin (seed de dados) | Notificação no painel; caso muda para "Proposta recebida" | RN-030 |
| 2 | Cedente acessa "Propostas" | Tela exibe valor, simulação financeira, timer regressivo; sem dados do Cessionário | RN-030 |
| 3 | Cedente clica "Aceitar Proposta" | Modal de confirmação com resumo financeiro (valor líquido em destaque) | RN-032 |
| 4 | Cedente confirma aceite | Caso muda para "Em formalização"; outras propostas ativas encerradas como "Superada" | RN-032, RN-033 |

**Suíte E2E-002b — Recusa e Contraproposta:**

| Passo | Ação do usuário | Resultado esperado | RN rastreada |
|---|---|---|---|
| 1 | Cedente clica "Recusar Proposta" e confirma | Caso retorna para "Disponível para compradores" | RN-034 |
| 2 | Nova proposta recebida; Cedente clica "Fazer Contraproposta" | Modal de contraproposta exibido com piso do cenário visível | RN-035 |
| 3 | Cedente informa valor abaixo do piso | Alerta inline; botão "Confirmar" bloqueado | RN-035 |
| 4 | Cedente informa valor válido e confirma | Contraproposta enviada; toast de sucesso; timer reiniciado | RN-035 |

### 5.6 Suíte E2E-003 — Assinatura ZapSign

**Pré-condição:** Caso em "Em formalização" com Termo Comercial pendente.

| Passo | Ação do usuário | Resultado esperado | RN rastreada |
|---|---|---|---|
| 1 | Cedente acessa "Assinaturas" | Lista com Termo Comercial e Instrumento de Cessão pendentes | RN-047 |
| 2 | Cedente clica "Assinar Agora" | ZapSign carregado inline (sem redirecionamento externo) | RN-047 |
| 3 | Cedente fecha o ZapSign antes de assinar | Documento permanece "Pendente"; toast informativo | RN-047 |
| 4 | Cedente assina Termo Comercial | Documento muda para "Assinado"; Instrumento de Cessão ainda pendente | RN-047, RN-048 |
| 5 | Cedente assina Instrumento de Cessão | Verificação de todas as assinaturas obrigatórias; Fechamento progride | RN-048 |

**Edge case:** ZapSign indisponível (mock de timeout) → mensagem de erro com botão "Tentar novamente" e opção "Abrir em nova aba".

### 5.7 Suíte E2E-004 — Escrow (Depósito, Reversão, Distribuição)

**Pré-condição:** Caso em "Negócio fechado"; Conta Escrow aberta.

| Passo | Ação do usuário / evento | Resultado esperado | RN rastreada |
|---|---|---|---|
| 1 | Sistema recebe webhook de depósito confirmado | Painel Financeiro exibe "Conta Escrow: Depósito Confirmado" | RN-051, RN-053 |
| 2 | Cedente acessa painel Financeiro | Valor líquido em destaque (maior fonte, verde); countdown de 15 dias visível | RN-052, RN-053 |
| 3 | Cedente clica "Solicitar Desistência" dentro dos 15 dias | Modal com aviso proeminente; campo de justificativa obrigatório | RN-053 |
| 4 | Cedente confirma desistência | Status da Escrow muda para "Em reversão"; toast de confirmação | — |
| 5 | Simulação: 15 dias sem desistência → distribuição automática | Caso muda para "Concluído"; comprovante disponível para download | RN-053 |

### 5.8 Suíte E2E-005 — Guardião do Retorno

**Pré-condição:** Cedente autenticado com caso ativo.

| Passo | Ação do usuário | Resultado esperado | RN rastreada |
|---|---|---|---|
| 1 | Cedente acessa "Assistente IA" | Mensagem de boas-vindas; 3 sugestões de ação exibidas | — |
| 2 | Cedente digita "Simular Cenário C" | Guardião retorna simulação com valores calculados para o Cenário C | — |
| 3 | Cedente pergunta sobre status do caso | Guardião retorna status atual corretamente via API | — |
| 4 | Cedente solicita cadastro assistido | Modo "Cadastro Assistido" ativado; stepper de 5 etapas exibido | — |
| 5 | Cedente abandona cadastro assistido | Dados descartados; modo desativado sem rascunho criado | — |
| 6 | Resposta do LLM com FOMO detectada | OutputFilterService bloqueia a resposta; Guardião reformula sem manipulação | — |

### 5.9 Suíte E2E-006 — Escalonamento de Cenário

**Pré-condição:** Cedente com caso no Cenário D, status "Disponível para compradores".

| Passo | Ação do usuário | Resultado esperado | RN rastreada |
|---|---|---|---|
| 1 | Cedente clica "Alterar Cenário" | Sistema exibe apenas opção "Escalonar para Cenário C" | RN-025, RN-026 |
| 2 | Cedente confirma escalonamento | Termo de Aceite gerado no ZapSign; botão "Alterar Cenário" bloqueado por 7 dias | RN-025, RN-027 |
| 3 | Cedente tenta escalonar novamente antes de 7 dias | Botão desabilitado com data de liberação exibida | RN-027 |
| 4 | Cedente tenta subir de Cenário C para D | Sistema exibe mensagem de impossibilidade com opção de cancelar o caso | RN-029 |

### 5.10 Testes Mobile (React Native Testing Library + Expo)

| ID | Fluxo | Plataforma | Caso testado |
|---|---|---|---|
| MOB-001 | Upload de documento via câmera | iOS + Android | Câmera abre; captura realizada; upload iniciado; documento aparece como "Em análise" |
| MOB-002 | Upload com permissão de câmera negada | iOS + Android | Tela de bloqueio com instrução; botão "Abrir configurações" |
| MOB-003 | Recebimento de notificação push — nova proposta | iOS + Android | Notificação push chega; ao abrir, navega direto para a proposta recebida |
| MOB-004 | Recebimento de notificação push — documento rejeitado | iOS + Android | Ao abrir, navega para o documento rejeitado com o motivo visível |
| MOB-005 | Chat com Guardião do Retorno (mobile) | iOS + Android | Chat em tela cheia; SSE streaming funciona; resposta exibida com cursor piscante |
| MOB-006 | Proposta recebida no mobile — aceite | iOS + Android | Modal de aceite exibe valor líquido em destaque; confirmação dupla funcional |
| MOB-007 | Offline — tentativa de upload | iOS + Android | Botão "Enviar" desabilitado; tooltip "Você precisa de conexão para enviar documentos." |
| MOB-008 | Bottom tab bar — navegação entre módulos | iOS + Android | Cada aba troca de contexto sem recarregar dados desnecessariamente |

---

## 6. Testes de Segurança

### 6.1 Isolamento LGPD entre Cedentes

| ID | Caso | Resultado esperado |
|---|---|---|
| SEC-001 | Cedente A tenta acessar caso do Cedente B via URL direta | HTTP 403; dados do Cedente B não expostos |
| SEC-002 | Cedente A tenta listar documentos do Cedente B | HTTP 403 |
| SEC-003 | Cedente A tenta acessar proposta do Cedente B | HTTP 403 |
| SEC-004 | Response de listagem de propostas contém dados do Cessionário (nome, CPF, email) | Falha — esses dados não devem aparecer em nenhuma response do módulo Cedente |
| SEC-005 | RLS do Supabase — query sem filtro por `cedente_id` retorna dados de outros Cedentes | Falha — RLS deve bloquear automaticamente |

### 6.2 Rate Limiting

| ID | Caso | Resultado esperado |
|---|---|---|
| SEC-006 | 10 tentativas de login com senha errada em 2 minutos | Após 5 falhas: lockout de 15 minutos; HTTP 429 com mensagem |
| SEC-007 | 50 requisições ao Guardião do Retorno em 1 minuto | Rate limit aplicado; HTTP 429 após exceder threshold |
| SEC-008 | Upload de 20 documentos em paralelo (ataque de carga) | Requisições acima do limite bloqueadas com HTTP 429 |

### 6.3 Validação de Uploads

| ID | Caso | Resultado esperado |
|---|---|---|
| SEC-009 | Upload de arquivo `.php` renomeado para `.pdf` | MIME type real verificado; arquivo rejeitado com HTTP 422 |
| SEC-010 | Upload de arquivo PDF com conteúdo malicioso embedded | Rejeitado na validação de MIME ou na verificação de integridade |
| SEC-011 | Upload via URL assinada expirada | HTTP 403; acesso negado |

---

## 7. Testes de Ambientes

### 7.1 Local (desenvolvimento)

- Todos os testes unitários e de integração rodam localmente com `pnpm test`.
- E2E pode ser executado localmente com `pnpm test:e2e` (requer Docker Compose com Postgres e Redis).
- Serviços externos (ZapSign, Escrow) sempre mockados.

### 7.2 Staging (pré-produção)

- CI executa suíte completa a cada push para `main`.
- E2E usa sandbox do ZapSign e ambiente sandbox do Escrow.
- Smoke tests automatizados pós-deploy: `/health`, `/api/v1/cedente/casos` (401 esperado sem token).
- Performance: todas as rotas críticas testadas com `k6` — SLA: p95 < 200ms.

### 7.3 Produção (smoke tests pós-deploy)

```bash
# Executado automaticamente pelo CI após deploy em produção
pnpm test:smoke:prod
# Verifica: /health, /api/v1/cedente/casos (401 esperado), /api/v1/cedente/propostas (401 esperado)
# Falha: retorna exit code 1 → rollback automático via CI
```

| Smoke test | Endpoint | Resultado esperado |
|---|---|---|
| Health check | `GET /health` | HTTP 200, `{ "status": "ok" }` em < 500ms |
| Auth — sem token | `GET /api/v1/cedente/casos` | HTTP 401 (não 5xx) |
| Auth — login inválido | `POST /api/v1/auth/login` com payload inválido | HTTP 400 (não 5xx) |
| Guardião — sem token | `POST /api/v1/cedente/guardiao/chat` | HTTP 401 (não 5xx) |

---

## 8. Performance

### 8.1 SLAs de Resposta

| Endpoint | SLA p95 | SLA p99 | Observação |
|---|---|---|---|
| `GET /api/v1/cedente/casos` | < 200ms | < 500ms | Lista com paginação |
| `POST /api/v1/cedente/casos` | < 500ms | < 1s | Inclui geração do Termo de Cadastro no ZapSign |
| `POST /api/v1/cedente/propostas/:id/aceitar` | < 300ms | < 800ms | Inclui encerramento de propostas concorrentes |
| `GET /api/v1/cedente/simulador` | < 300ms | < 500ms | Cálculo dos 4 cenários |
| `POST /api/v1/cedente/guardiao/chat` | < 3s (first token) | < 5s | SSE streaming — latência do primeiro token |

### 8.2 Ferramentas de Performance

| Ferramenta | Uso |
|---|---|
| `k6` | Load testing dos endpoints críticos em staging |
| Lighthouse | Performance frontend (Core Web Vitals, bundle size) |
| `pnpm build` + `rollup-plugin-visualizer` | Verificação do bundle size (< 300KB gzipped) |

---

## 9. Critérios de Aceitação por Fase

### 9.1 Gate para Merge em `develop`

- [ ] Todos os testes unitários passando (zero falhas).
- [ ] Todos os testes de integração passando.
- [ ] Cobertura de código igual ou acima do mínimo por área (tabela 3.3).
- [ ] Lint sem erros (`pnpm lint`).
- [ ] TypeScript sem erros de compilação (`pnpm typecheck`).
- [ ] Nenhum `console.log` em código de produção.

### 9.2 Gate para Merge em `main`

- [ ] Todos os critérios de `develop` satisfeitos.
- [ ] Suíte E2E completa passando em staging (zero falhas em P0/P1).
- [ ] Smoke tests de staging passando.
- [ ] Performance p95 < 200ms em endpoints críticos verificada com `k6`.
- [ ] Testes de segurança SEC-001 a SEC-011 passando.
- [ ] D28 (Checklist de Qualidade) aprovado pelo Tech Lead.

### 9.3 Gate para Go-Live

- [ ] Todos os critérios de `main` satisfeitos.
- [ ] E2E executado manualmente no ambiente de staging em modo headful.
- [ ] Integrações ZapSign e Escrow testadas com sandbox de produção.
- [ ] Dados de teste removidos de staging (conformidade LGPD).
- [ ] D29 (Go-Live Playbook) aprovado pelo Tech Lead.

---

## 10. Dados de Teste e LGPD

### 10.1 Princípios

- **Nunca usar dados reais de Cedentes em testes** — violação de LGPD.
- Dados gerados por factories (Faker.js) são descartados ao final de cada suíte.
- Banco de dados de teste completamente isolado do banco de staging e produção.
- Staging não contém dados de usuários reais — apenas dados sintéticos de testes.

### 10.2 Factory de Dados de Teste

```typescript
// src/test/factories/cedente.factory.ts
export const createCedenteFactory = (overrides?: Partial<Cedente>): Cedente => ({
  id: `test-${faker.string.uuid()}`,
  nome: faker.person.fullName(),
  cpf: faker.string.numeric(11),
  email: faker.internet.email(),
  telefone: faker.phone.number(),
  tipo: 'PF',
  ...overrides,
})

export const createCasoFactory = (cedenteId: string, overrides?: Partial<Caso>): Caso => ({
  id: `test-${faker.string.uuid()}`,
  cedente_id: cedenteId,
  status: 'cadastro_realizado',
  cenario: 'C',
  endereco: faker.location.streetAddress(),
  valor_pago_cedente: faker.number.int({ min: 50_000, max: 500_000 }),
  ...overrides,
})
```

---

## 11. CI/CD e Automação

### 11.1 Pipeline CI (GitHub Actions)

```yaml
# .github/workflows/ci.yml
jobs:
  unit-and-integration:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
      - name: Install dependencies (pnpm)
      - name: Lint
      - name: Typecheck
      - name: Unit tests (pnpm test:unit)
      - name: Integration tests (pnpm test:integration) # Postgres + Redis via Docker
      - name: Coverage report (≥ thresholds definidos na tabela 3.3)

  e2e:
    runs-on: ubuntu-latest
    needs: [unit-and-integration]
    if: github.ref == 'refs/heads/main' || github.event_name == 'pull_request'
    steps:
      - name: E2E tests (pnpm test:e2e) # Playwright + staging
      - name: Upload Playwright trace on failure
```

### 11.2 Regras de Bloqueio

| Evento | Bloqueio |
|---|---|
| Push em qualquer branch | Lint + Typecheck + Unitários + Integração |
| PR para `develop` | Todos os acima |
| PR para `main` | Todos os acima + E2E completo + Smoke tests staging |
| Deploy para produção | Smoke tests produção pós-deploy + health check |
