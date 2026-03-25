# Guia de Contribuição — AI-Dani-Admin

## Padrões de Desenvolvimento e Fluxo de Contribuição

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend |
| Escopo | Padrões de código, fluxo de contribuição, convenções de commit e processo de code review para o módulo AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D02 (Stacks), D15 (Arquitetura de Pastas), D22 (Setup Local) |

---

> **📌 TL;DR**
>
> - **Fluxo:** feature branch → PR → review obrigatório → squash merge. Nenhum commit direto na `main`.
> - **Commits:** Conventional Commits (`feat:`, `fix:`, `refactor:`, `test:`, `docs:`). Mensagens em português do Brasil.
> - **Code review:** toda PR requer 1 aprovação. PRs com mudanças em módulos de segurança (auth, scope-filter, context-filter) exigem 2 aprovações.
> - **Testes:** cobertura mínima 80% por módulo. PRs sem testes nos Services são rejeitadas.
> - **Linting:** ESLint + Prettier obrigatórios. CI rejeita PRs com lint errors.
> - **Geração por IA:** código gerado por Claude Code ou Codex deve passar pelas mesmas verificações de qualidade. Revisor humano obrigatório.

---

## 1. Fluxo de Trabalho

### 1.1 Criação de Branch

```bash
# Nomenclatura: <tipo>/<numero-da-issue>-<descricao-curta>
# Tipos: feat, fix, refactor, test, docs, chore

git checkout main
git pull origin main
git checkout -b feat/123-takeover-concorrente

# Exemplos válidos:
# feat/124-dashboard-metricas-cache
# fix/125-rate-limit-sliding-window
# refactor/126-supervision-repository-soft-delete
# test/127-takeover-lock-otimista
# docs/128-atualizar-d16-endpoints
```

### 1.2 Fluxo de Commit

```bash
# Adicionar apenas arquivos relevantes (nunca git add .)
git add src/takeover/takeover.service.ts
git add src/takeover/tests/takeover.service.spec.ts

# Commit com Conventional Commits em português
git commit -m "feat(takeover): implementar lock otimista para takeover simultâneo"
```

### 1.3 Pull Request

```bash
# Abrir PR apontando para main
git push origin feat/123-takeover-concorrente
# Abrir PR via GitHub UI ou CLI

# Título da PR: mesmo formato do commit
# Body: preencher template (ver seção 3)
```

---

## 2. Conventional Commits

Formato: `<tipo>(<escopo>): <descrição>`

| Tipo | Quando usar | Exemplo |
|---|---|---|
| `feat` | Nova funcionalidade | `feat(alerts): implementar scheduler de monitoramento de taxa de erro` |
| `fix` | Correção de bug | `fix(takeover): corrigir race condition no encerramento de takeover` |
| `refactor` | Refatoração sem mudança de comportamento | `refactor(supervision): extrair filtro de soft delete para repositório` |
| `test` | Adição ou correção de testes | `test(agent-config): adicionar teste de validação de threshold fora do range` |
| `docs` | Documentação | `docs: atualizar D16 com endpoint de push-tokens` |
| `chore` | Manutenção (deps, configs) | `chore: atualizar prisma para 6.2.0` |
| `perf` | Melhoria de performance | `perf(metrics): adicionar índice em interactions.confidence_score` |
| `ci` | Pipeline de CI/CD | `ci: adicionar step de lint no workflow de PR` |

**Regras:**
- Escopo é o nome do módulo: `takeover`, `supervision`, `metrics`, `alerts`, `agent-config`, `launch-readiness`, `audit`, `common`.
- Descrição em português, minúscula, sem ponto final.
- Breaking changes: adicionar `!` após o tipo/escopo e footer `BREAKING CHANGE:`.
- Máximo 72 caracteres na primeira linha.

---

## 3. Template de Pull Request

```markdown
## Descrição
<!-- Explique o que esta PR faz e por que é necessária -->

## Tipo de mudança
- [ ] feat — nova funcionalidade
- [ ] fix — correção de bug
- [ ] refactor — refatoração
- [ ] test — testes
- [ ] docs — documentação
- [ ] chore — manutenção

## Issues relacionadas
Closes #<número>

## Checklist de implementação
- [ ] Código segue as convenções do D15 (Arquitetura de Pastas)
- [ ] Testes unitários adicionados/atualizados para o Service
- [ ] Nenhum `console.log` em produção (apenas Pino)
- [ ] Nenhum PII em logs
- [ ] DTOs com validações class-validator
- [ ] Errors usando shape padrão `{ error: { code, message } }` (D20)
- [ ] Sem import direto cross-module (apenas via exports do módulo)
- [ ] `.env.example` atualizado se novas variáveis foram adicionadas

## Checklist de segurança (obrigatório para auth, scope-filter, context-filter)
- [ ] Filtro de escopo mantido (RN-DA-037)
- [ ] Filtro de contexto mantido (RN-DA-037)
- [ ] Role ADMIN verificada em endpoints novos
- [ ] Nenhum dado de usuário em logs

## Como testar
<!-- Passos para o revisor reproduzir e validar a mudança -->
1.
2.
3.

## Screenshots (se UI)
<!-- N/A para este módulo backend — apenas se houver mudança em resposta da API -->
```

---

## 4. Padrões de Código

### 4.1 Regras Gerais

```typescript
// ✅ Correto: DI via construtor
@Injectable()
export class TakeoverService {
  constructor(
    private readonly takeoverRepository: TakeoverRepository,
    private readonly auditService: AuditService,
  ) {}
}

// ❌ Incorreto: instanciação manual de dependências
export class TakeoverService {
  private repo = new TakeoverRepository()  // ❌
}
```

```typescript
// ✅ Correto: async/await com tipos explícitos
async findInteraction(id: string): Promise<Interaction | null> {
  return this.prisma.interaction.findUnique({ where: { id } })
}

// ❌ Incorreto: any implícito
async findInteraction(id) {   // ❌ sem tipo
  return this.prisma.interaction.findUnique({ where: { id } })
}
```

```typescript
// ✅ Correto: HttpException com shape padrão
throw new NotFoundException({
  error: {
    code: 'DA-SUP-002',
    message: 'Interação não encontrada.',
  }
})

// ❌ Incorreto: mensagem de texto sem código
throw new NotFoundException('Not found')  // ❌
```

### 4.2 Proibições Absolutas

```typescript
// ❌ PROIBIDO: console.log em qualquer arquivo
console.log('debug')  // ❌ usar logger.log()

// ❌ PROIBIDO: any explícito em tipos críticos
const data: any = ...  // ❌ usar tipo específico

// ❌ PROIBIDO: import direto cross-module
import { TakeoverService } from '../takeover/takeover.service'  // ❌ de dentro de outro módulo

// ❌ PROIBIDO: PII em logs
logger.log(`Admin ${adminEmail} fez takeover`)  // ❌ e-mail é PII
logger.log(`Usuário ${userId} enviou mensagem`)  // ✅ UUID é seguro

// ❌ PROIBIDO: await em loop (N+1)
for (const interaction of interactions) {
  await this.sendNotification(interaction)  // ❌ use Promise.all()
}

// ✅ Correto
await Promise.all(interactions.map(i => this.sendNotification(i)))
```

### 4.3 Estrutura de Testes

```typescript
// supervision.service.spec.ts — estrutura obrigatória

describe('SupervisionService', () => {
  let service: SupervisionService
  let repository: jest.Mocked<SupervisionRepository>
  let auditService: jest.Mocked<AuditService>

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        SupervisionService,
        {
          provide: SupervisionRepository,
          useValue: {
            findAll: jest.fn(),
            findById: jest.fn(),
          },
        },
        {
          provide: AuditService,
          useValue: { log: jest.fn() },
        },
      ],
    }).compile()

    service = module.get(SupervisionService)
    repository = module.get(SupervisionRepository)
    auditService = module.get(AuditService)
  })

  describe('getInteractions', () => {
    it('deve retornar lista paginada de interações', async () => {
      // Arrange
      repository.findAll.mockResolvedValue({ data: [mockInteraction], meta: { total: 1 } })

      // Act
      const result = await service.getInteractions('admin-id', mockFilter)

      // Assert
      expect(result.data).toHaveLength(1)
      expect(auditService.log).toHaveBeenCalledWith('ACESSO_PAINEL', expect.any(Object), 'admin-id')
    })

    it('deve retornar lista vazia quando não há interações', async () => {
      repository.findAll.mockResolvedValue({ data: [], meta: { total: 0 } })
      const result = await service.getInteractions('admin-id', mockFilter)
      expect(result.data).toHaveLength(0)
    })
  })
})
```

---

## 5. Code Review

### 5.1 Critérios de Aprovação

| Critério | Obrigatório | Descrição |
|---|---|---|
| Testes no Service | Sim | Todo Service modificado tem testes cobrindo happy path + cenário de erro |
| Shape de erro correto | Sim | Todas as exceções usam `{ error: { code, message } }` |
| PII em logs | Sim (bloqueante) | Nenhum log com e-mail, CPF, nome ou dado pessoal |
| Convenções de nomenclatura | Sim | Conforme D15 — arquivos, classes, variáveis |
| Import cross-module | Sim (bloqueante) | Nenhum import direto entre módulos (só via exports) |
| Cobertura mínima 80% | Sim | `npm run test:cov` deve mostrar ≥ 80% por módulo modificado |
| Lint sem erros | Sim (bloqueante) | `npm run lint` deve passar sem erros |
| `.env.example` atualizado | Se aplicável | Novas variáveis documentadas no template |

### 5.2 Aprovações Necessárias

| Tipo de mudança | Aprovações |
|---|---|
| Feature comum (CRUD, métricas, alertas) | 1 aprovação |
| Mudança em auth, filtro de escopo, filtro de contexto | 2 aprovações |
| Mudança em schema Prisma (migrations) | 1 aprovação + confirmação do Tech Lead |
| Mudança em Docker, CI/CD | 1 aprovação do DevOps |

### 5.3 Merge Strategy

**Squash merge** para todas as PRs. O histórico do `main` deve ter 1 commit por PR, com mensagem Conventional Commit.

```bash
# Squash merge via GitHub UI ou:
git checkout main
git merge --squash feat/123-takeover-concorrente
git commit -m "feat(takeover): implementar lock otimista para takeover simultâneo"
```

---

## 6. Padrão de Testes por Camada

| Camada | Tipo de Teste | Ferramenta | Cobertura Mínima |
|---|---|---|---|
| Service | Unitário (mocks de Repository e deps externas) | Jest | 80% |
| Controller | Unitário (mocks de Service) | Jest | 70% |
| Repository | Integração (banco real em CI) | Jest + Prisma test db | N/A |
| API (e2e) | Integração/E2E (servidor completo) | Jest + Supertest | Fluxos críticos |
| Consumers RabbitMQ | Unitário (mock do amqplib) | Jest | 80% |

---

## 7. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Fluxo de contribuição, Conventional Commits, template de PR, padrões de código, critérios de code review. |
