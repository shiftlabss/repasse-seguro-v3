# 23 - Guia de Contribuição

## Módulo Cedente · Plataforma Repasse Seguro

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 23 - Guia de Contribuição | v1.0 | 2026-03-23 (America/Fortaleza) | Claude Code Desktop | Aprovado |

| Campo | Valor |
|---|---|
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 |

---

> **TL;DR**
>
> - **Branches:** `main` (produção), `develop` (integração), `feature/*`, `fix/*`, `hotfix/*`.
> - **Commits:** Conventional Commits obrigatório. Validado via Husky + commitlint no pre-commit.
> - **PRs:** Template obrigatório. Aprovação mínima de 1 reviewer. CI deve passar (lint + type-check + testes) antes do merge.
> - **TypeScript:** `strict: true` obrigatório em todos os workspaces. Arquivos `.js`/`.jsx` proibidos.
> - **Convenções do Módulo Cedente:** IDs de RN nos comentários de lógica de negócio, prefixo `Cedente` em componentes específicos do módulo, estrutura de teste espelhando a estrutura de source.
> - **Desenvolvimento 100% por IA:** O código é gerado por agentes de IA (Claude Code, GPT Codex). Este guia é o contrato de qualidade que todo código gerado deve seguir.

---

## 1. Modelo de Branches

### 1.1 Estrutura de branches

| Branch | Propósito | Origem | Destino de merge |
|---|---|---|---|
| `main` | Código em produção. Protegida. | — | — |
| `develop` | Integração de features. Espelha staging. | `main` | `main` (via release) |
| `feature/<escopo>/<descricao>` | Nova funcionalidade | `develop` | `develop` |
| `fix/<escopo>/<descricao>` | Correção de bug não crítico | `develop` | `develop` |
| `hotfix/<descricao>` | Correção crítica em produção | `main` | `main` + `develop` |
| `release/<versao>` | Preparação de release | `develop` | `main` + `develop` |

### 1.2 Convenção de nomenclatura de branches

```
feature/cedente-cadastro/wizard-simulador
feature/cedente-guardiao/rag-base-conhecimento
fix/cedente-propostas/timer-regressivo
fix/cedente-assinaturas/zapsign-webhook
hotfix/cedente-auth/bloqueio-tentativas-login
```

**Regras:**
- Letras minúsculas, hífens para espaços.
- Escopo sempre referencia o módulo ou submódulo: `cedente-cadastro`, `cedente-propostas`, `cedente-documentos`, `cedente-assinaturas`, `cedente-financeiro`, `cedente-guardiao`, `cedente-auth`.
- Sem caracteres especiais além de `/` e `-`.
- Máximo 60 caracteres.

### 1.3 Fluxo visual

```
main ──────────────────────────────────────────────── (produção)
  │
  └── develop ────────────────────────────────────── (staging)
        │
        ├── feature/cedente-cadastro/wizard ──── PR → develop
        ├── feature/cedente-guardiao/rag ──────── PR → develop
        └── fix/cedente-assinaturas/zapsign ───── PR → develop

main ── hotfix/cedente-auth/bloqueio ─── PR → main → develop
```

### 1.4 Regras de proteção de branch

- `main` e `develop`: push direto proibido. Apenas via pull request.
- `main`: requer aprovação de 1 reviewer + status checks do CI passando.
- `develop`: requer status checks do CI passando. Aprovação recomendada mas não obrigatória para features pequenas.
- Commits de merge devem usar **Squash and Merge** para `develop` e **Merge Commit** para `main` (preservar histórico de releases).

---

## 2. Conventional Commits

### 2.1 Formato obrigatório

```
<tipo>(<escopo>): <descrição>

[corpo opcional]

[rodapé opcional — BREAKING CHANGE, Closes #issue]
```

**Exemplos válidos:**
```
feat(cedente-cadastro): adicionar timer anti-skip no simulador de cenários (RN-021)
fix(cedente-assinaturas): corrigir webhook ZapSign com status 200 duplicado
feat(cedente-guardiao): implementar pipeline RAG para base de conhecimento
refactor(cedente-auth): extrair lógica de bloqueio por tentativas para guard
test(cedente-propostas): adicionar testes para timer regressivo de 5 dias úteis
docs(cedente-stacks): atualizar versão do Langfuse para 3.x
chore(deps): atualizar pnpm para 9.x
```

### 2.2 Tipos permitidos

| Tipo | Quando usar |
|---|---|
| `feat` | Nova funcionalidade que implementa uma RN |
| `fix` | Correção de bug ou comportamento incorreto |
| `docs` | Alterações em documentação apenas |
| `test` | Adição ou correção de testes |
| `refactor` | Refatoração sem mudança de comportamento |
| `perf` | Otimização de performance |
| `chore` | Tarefas de manutenção (deps, config, build) |
| `ci` | Alterações nos pipelines de CI/CD |
| `style` | Formatação de código (sem mudança lógica) |
| `revert` | Reversão de commit anterior |

### 2.3 Escopos do Módulo Cedente

Use sempre o escopo mais específico possível:

| Escopo | Cobertura |
|---|---|
| `cedente-auth` | Autenticação, JWT, sessão, bloqueio de login |
| `cedente-cadastro` | Wizard de cadastro de imóvel, simulador, cenários |
| `cedente-documentos` | Upload de dossiê, validação de documentos |
| `cedente-propostas` | Recebimento, aceite, recusa, contraproposta, timer |
| `cedente-assinaturas` | Integração ZapSign, webhook, rastreabilidade |
| `cedente-financeiro` | Painel financeiro, Escrow, comissões, período de reversão |
| `cedente-guardiao` | Guardião do Retorno, RAG, OpenAI, Langfuse |
| `cedente-notificacoes` | E-mail (Resend), painel (Realtime), push (Expo) |
| `cedente-mobile` | App Expo, camera, upload mobile |
| `cedente-escalonamento` | Escalonamento de cenário, cooldown, fila |
| `cedente-anuencia` | Fluxo de anuência da construtora, inadimplência |
| `cedente-lgpd` | Exclusão de dados, privacidade |
| `shared-types` | Pacote de tipos compartilhados |
| `shared-utils` | Utilitários compartilhados |
| `api` | Backend NestJS genérico |
| `web` | Frontend Next.js genérico |
| `mobile` | App Expo genérico |
| `deps` | Atualização de dependências |
| `ci` | Pipeline de CI/CD |
| `infra` | Configurações de infra, Docker, Railway |

### 2.4 Validação automática de commits

O Husky valida o formato de cada commit via commitlint:

```bash
# Configuração em commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'scope-enum': [2, 'always', [
      'cedente-auth', 'cedente-cadastro', 'cedente-documentos',
      'cedente-propostas', 'cedente-assinaturas', 'cedente-financeiro',
      'cedente-guardiao', 'cedente-notificacoes', 'cedente-mobile',
      'cedente-escalonamento', 'cedente-anuencia', 'cedente-lgpd',
      'shared-types', 'shared-utils', 'api', 'web', 'mobile',
      'deps', 'ci', 'infra'
    ]],
    'subject-max-length': [2, 'always', 100],
  },
};
```

Commits que não seguirem o formato serão rejeitados pelo hook `commit-msg`.

---

## 3. Pull Requests

### 3.1 Template de PR

Todo PR deve usar o template abaixo (`.github/pull_request_template.md`):

```markdown
## Descrição

<!-- Descreva o que foi implementado ou corrigido. -->

## RNs Implementadas / Corrigidas

<!-- Liste as Regras de Negócio cobertas por este PR. -->
- [ ] RN-XXX — Descrição resumida
- [ ] RN-XXX — Descrição resumida

## Tipo de mudança

- [ ] Nova feature (feat)
- [ ] Correção de bug (fix)
- [ ] Hotfix crítico (hotfix)
- [ ] Refatoração (refactor)
- [ ] Testes (test)
- [ ] Documentação (docs)
- [ ] Infra / CI (chore/ci)

## Checklist

### Código
- [ ] TypeScript `strict: true` — sem erros de tipo
- [ ] ESLint e Prettier sem erros
- [ ] Sem `console.log` ou `debugger` no código
- [ ] Sem dados hardcoded (senhas, chaves, URLs de produção)
- [ ] Código não deixa dados de um Cedente acessíveis a outro (RN-011)

### Testes
- [ ] Testes unitários adicionados/atualizados para a lógica de negócio
- [ ] Testes de integração adicionados/atualizados para endpoints novos
- [ ] `pnpm test` passa localmente

### Migrations
- [ ] Nenhuma migration neste PR
- [ ] Migration incluída — verificado que é não-destrutiva ou segue processo de 2 etapas

### Segurança
- [ ] Nenhuma variável de ambiente ou secret no código
- [ ] Inputs validados via class-validator (backend)
- [ ] RLS do Supabase configurado para novas tabelas (se aplicável)

### Documentação
- [ ] Swagger/OpenAPI atualizado para endpoints novos (se aplicável)
- [ ] `.env.example` atualizado para novas variáveis (se aplicável)

## Como testar

<!-- Passos para o reviewer testar as mudanças localmente. -->

1.
2.
3.

## Screenshots / Gravações (opcional)

<!-- Para mudanças de UI. -->
```

### 3.2 Processo de review

1. Abrir PR de `feature/*` para `develop`.
2. Título do PR deve seguir o formato: `feat(cedente-propostas): implementar aceite de proposta com dupla confirmação`.
3. Descrever as RNs cobertas.
4. CI executa automaticamente: lint, type-check, testes.
5. Mínimo 1 reviewer deve aprovar. Para mudanças de segurança (auth, isolamento de dados, RLS), exigir 2 reviewers.
6. Resolver todos os comentários antes do merge.
7. Merge somente com CI verde e aprovação.

### 3.3 Tamanho recomendado de PR

- **Ideal:** menos de 400 linhas de mudança.
- **Aceitável:** até 800 linhas com justificativa (feature completa com testes).
- **Evitar:** PRs com mais de 1000 linhas. Dividir em PRs menores por camada ou subfeature.

### 3.4 Revisão de código — pontos de atenção específicos do Módulo Cedente

Durante o review, verificar especificamente:

| Ponto | Verificação | RN |
|---|---|---|
| Isolamento de dados | Toda query inclui filtro `cedente_id = user.id`? | RN-011 |
| Anonimato do Cessionário | DTOs de resposta expõem dados do Cessionário? | RN-012 |
| Imutabilidade de CPF/CNPJ | Endpoint de update inclui CPF/CNPJ no body? | RN-007 |
| Validação MIME type | Upload valida MIME real, não só extensão? | RN-042 |
| Documentos assinados | Endpoint permite alterar doc com status "Assinado"? | RN-049 |
| Rate limiting do Guardião | Endpoint `/ai/guardiao` tem throttler 30 req/min? | Seção 9.3 de Stacks |
| PII no LLM | CPF/dados financeiros mascarados antes de enviar à OpenAI? | RN-011, Stacks 9.3 |
| Estado do caso | Transição de estado segue máquina de estados definida? | RN-086 |

---

## 4. Padrões de Código TypeScript

### 4.1 Regras gerais

```typescript
// ✅ CORRETO — strict: true, sem any, sem non-null assertion desnecessário
interface CedenteCase {
  id: string;
  cedenteId: string;
  status: CaseStatus;
  cenario: Cenario;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null; // soft delete
}

// ❌ PROIBIDO — any e non-null assertion sem justificativa
const response: any = await api.get('/casos');
const id = response!.data.id;
```

**Configuração obrigatória em todos os `tsconfig.json`:**
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### 4.2 Nomenclatura

| Contexto | Convenção | Exemplo |
|---|---|---|
| Componentes React (web) | PascalCase | `CedentePropostaCard`, `GuardiaoChat` |
| Componentes React Native (mobile) | PascalCase | `CedenteDocumentoUpload`, `CedenteAssinaturaView` |
| Hooks personalizados | camelCase com prefixo `use` | `useCedentePropostas`, `useGuardiaoChat` |
| Services (NestJS) | PascalCase com sufixo `Service` | `CedentePropostasService`, `ZapSignService` |
| Controllers (NestJS) | PascalCase com sufixo `Controller` | `CedenteCasosController` |
| DTOs | PascalCase com sufixo `Dto` | `CreateCedenteDto`, `AceitarPropostaDto` |
| Constantes | SCREAMING_SNAKE_CASE | `MAX_UPLOAD_SIZE_MB`, `PROPOSTA_TIMEOUT_DIAS` |
| Arquivos de componente | kebab-case | `cedente-proposta-card.tsx` |
| Arquivos de service/module | kebab-case | `cedente-propostas.service.ts` |
| Arquivos de teste | espelham o arquivo testado | `cedente-propostas.service.spec.ts` |
| Tabelas do banco | snake_case | `cedente_casos`, `cedente_documentos` |
| Colunas do banco | snake_case | `cedente_id`, `status_proposta`, `created_at` |
| Endpoints de API | kebab-case | `GET /api/v1/cedente/meus-casos` |
| Eventos PostHog | snake_case | `case_created`, `proposal_accepted` |

### 4.3 ESLint — regras obrigatórias

Configuração compartilhada em `packages/config-eslint/`:

```javascript
// packages/config-eslint/index.js
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/strict-type-checked',
    'prettier',
  ],
  rules: {
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-non-null-assertion': 'warn',
    'no-console': ['error', { allow: ['warn', 'error'] }],
    '@typescript-eslint/explicit-function-return-type': 'warn',
    'no-unused-vars': 'off',
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
  },
};
```

### 4.4 Prettier — configuração obrigatória

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always"
}
```

### 4.5 Estrutura de módulo NestJS — padrão do Cedente

```
apps/api/src/modules/cedente-propostas/
  cedente-propostas.module.ts
  cedente-propostas.controller.ts
  cedente-propostas.service.ts
  cedente-propostas.service.spec.ts
  dto/
    aceitar-proposta.dto.ts
    contraproposta.dto.ts
    listar-propostas-query.dto.ts
  entities/
    proposta.entity.ts
  guards/
    proposta-owner.guard.ts
  types/
    proposta-status.enum.ts
```

### 4.6 Estrutura de componente Next.js — padrão do Cedente

```
apps/web-cedente/src/components/propostas/
  CedentePropostaCard/
    CedentePropostaCard.tsx
    CedentePropostaCard.test.tsx
    index.ts           # re-export: export { CedentePropostaCard } from './CedentePropostaCard'
```

---

## 5. Convenções Específicas do Módulo Cedente

### 5.1 Referência a Regras de Negócio nos comentários

Todo trecho de código que implementa uma regra de negócio específica deve ter um comentário inline com o ID da RN:

```typescript
// ✅ CORRETO — Referência clara à regra de negócio
async aceitarProposta(cedenteId: string, propostaId: string): Promise<void> {
  // RN-032: Aceite exige dupla confirmação — validar token de confirmação
  await this.validarTokenConfirmacao(propostaId, cedenteId);

  // RN-033: Aceitar proposta cancela todas as propostas concorrentes do caso
  const caso = await this.casoRepository.findByCedenteAndProposta(cedenteId, propostaId);
  await this.cancelarPropostasConcorrentes(caso.id, propostaId);
}

// RN-031: Timer regressivo — proposta expira em 5 dias úteis sem resposta
const PROPOSTA_TIMEOUT_DIAS_UTEIS = 5;
```

```typescript
// ✅ CORRETO — Validação de estado com referência à RN
// RN-086: Estados avançam sempre para frente. Regressões permitidas: lista explícita.
function validarTransicaoDeEstado(estadoAtual: CaseStatus, novoEstado: CaseStatus): void {
  const transicoesBloqueadas: [CaseStatus, CaseStatus][] = [
    ['aprovado_para_oferta', 'em_analise'],
    ['negocio_fechado', 'proposta_recebida'],
  ];
  // ...
}
```

### 5.2 Prefixo de componentes específicos do Cedente

Componentes React que são exclusivos do Módulo Cedente devem ter o prefixo `Cedente`:

```typescript
// ✅ Componentes exclusivos do Cedente
export function CedentePropostaCard({ proposta }: Props) { ... }
export function CedenteDocumentoDossie({ caso }: Props) { ... }
export function GuardiaoChat({ cedenteId }: Props) { ... }
export function CedenteFinanceiroPanel({ caso }: Props) { ... }
export function CedenteEscalonamentoModal({ caso }: Props) { ... }
```

Componentes do pacote `packages/ui/` (shadcn/ui customizados) não precisam do prefixo — são genéricos.

### 5.3 Estrutura de testes

**Regra:** A estrutura de arquivos de teste espelha a estrutura de source.

```
apps/api/src/modules/cedente-propostas/
  cedente-propostas.service.ts
  cedente-propostas.service.spec.ts   ← testes unitários do service

apps/api/test/
  cedente-propostas.e2e-spec.ts       ← testes de integração/E2E do módulo

apps/web-cedente/src/components/propostas/
  CedentePropostaCard/
    CedentePropostaCard.tsx
    CedentePropostaCard.test.tsx      ← testes de componente
```

**Template de teste unitário para service NestJS:**

```typescript
// cedente-propostas.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { CedentePropostasService } from './cedente-propostas.service';

describe('CedentePropostasService', () => {
  let service: CedentePropostasService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CedentePropostasService,
        // mocks de repositórios e dependências
      ],
    }).compile();

    service = module.get<CedentePropostasService>(CedentePropostasService);
  });

  // RN-031: Timer de proposta — 5 dias úteis
  describe('verificarExpiracaoProposta', () => {
    it('deve encerrar proposta após 5 dias úteis sem resposta', () => {
      // arrange
      // act
      // assert
    });

    it('deve manter proposta ativa antes de 5 dias úteis', () => {
      // arrange
      // act
      // assert
    });
  });

  // RN-033: Aceite cancela propostas concorrentes
  describe('aceitarProposta', () => {
    it('deve cancelar todas as propostas concorrentes ao aceitar', async () => {
      // arrange
      // act
      // assert
    });
  });
});
```

**Template de teste de componente React:**

```typescript
// CedentePropostaCard.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { CedentePropostaCard } from './CedentePropostaCard';
import { mockProposta } from '@/tests/fixtures/propostas';

describe('CedentePropostaCard', () => {
  // RN-030: Timer regressivo com badge vermelho nas últimas 24h
  it('exibe badge vermelho quando proposta está nas últimas 24 horas', () => {
    const propostaProxDeExpirar = {
      ...mockProposta,
      expiresAt: new Date(Date.now() + 20 * 60 * 60 * 1000), // 20h restantes
    };
    render(<CedentePropostaCard proposta={propostaProxDeExpirar} />);
    expect(screen.getByRole('status', { name: /urgente/i })).toBeInTheDocument();
  });
});
```

### 5.4 Hooks de dados — padrão TanStack Query

```typescript
// apps/web-cedente/src/hooks/cedente/useMinhasCasos.ts

import { useQuery } from '@tanstack/react-query';
import { cedenteApi } from '@/services/api';
import type { CasoPaginado } from '@shared-types/cedente';

// Chaves de cache centralizadas
export const cedenteQueryKeys = {
  all: ['cedente'] as const,
  casos: () => [...cedenteQueryKeys.all, 'casos'] as const,
  casoPorId: (id: string) => [...cedenteQueryKeys.casos(), id] as const,
  propostas: (casoId: string) => [...cedenteQueryKeys.all, 'propostas', casoId] as const,
};

export function useMeusCasos(pagina = 1) {
  return useQuery({
    queryKey: cedenteQueryKeys.casos(),
    queryFn: () => cedenteApi.getMeusCasos({ pagina, itensPorPagina: 10 }),
    staleTime: 60_000, // RN-014: dashboard atualiza a cada 60s
    retry: 3,
  });
}
```

### 5.5 Segurança — isolamento de dados

**Toda query de banco que envolva dados de Cedente deve incluir o filtro `cedente_id`:**

```typescript
// ✅ CORRETO — RN-011: isolamento total
async findMeusCasos(cedenteId: string, pagina: number): Promise<Caso[]> {
  return this.prisma.caso.findMany({
    where: {
      cedente_id: cedenteId,  // ← obrigatório
      deleted_at: null,
    },
    take: 10,
    skip: (pagina - 1) * 10,
    orderBy: { created_at: 'desc' },
  });
}

// ❌ PROIBIDO — sem filtro de Cedente
async findAllCasos(): Promise<Caso[]> {
  return this.prisma.caso.findMany(); // expõe dados de todos os Cedentes!
}
```

---

## 6. Fluxo de Desenvolvimento

### 6.1 Criar uma nova feature

```bash
# 1. Atualizar develop
git checkout develop
git pull origin develop

# 2. Criar branch
git checkout -b feature/cedente-propostas/aceite-dupla-confirmacao

# 3. Desenvolver com commits atômicos
git add apps/api/src/modules/cedente-propostas/
git commit -m "feat(cedente-propostas): implementar aceite de proposta (RN-032)"

git add apps/api/src/modules/cedente-propostas/
git commit -m "test(cedente-propostas): adicionar testes para aceite de proposta"

git add apps/web-cedente/src/components/propostas/
git commit -m "feat(cedente-propostas): adicionar modal de dupla confirmação de aceite (RN-032)"

# 4. Push e abrir PR
git push -u origin feature/cedente-propostas/aceite-dupla-confirmacao
gh pr create --base develop --title "feat(cedente-propostas): aceite de proposta com dupla confirmação (RN-032)"
```

### 6.2 Corrigir um bug

```bash
# 1. Criar branch de fix
git checkout develop
git pull origin develop
git checkout -b fix/cedente-assinaturas/zapsign-webhook-duplicado

# 2. Corrigir e testar
# 3. Commit
git commit -m "fix(cedente-assinaturas): evitar processamento duplicado de webhook ZapSign"

# 4. PR
git push -u origin fix/cedente-assinaturas/zapsign-webhook-duplicado
gh pr create --base develop
```

### 6.3 Hotfix em produção

```bash
# 1. Criar hotfix a partir de main
git checkout main
git pull origin main
git checkout -b hotfix/cedente-auth-bloqueio-login

# 2. Corrigir
git commit -m "fix(cedente-auth): corrigir contador de tentativas de login não resetando (RN-005)"

# 3. PR para main
gh pr create --base main --title "hotfix(cedente-auth): corrigir contador de tentativas de login"

# 4. Após merge em main, fazer merge também em develop
git checkout develop
git merge --no-ff main
git push origin develop
```

---

## 7. Qualidade e Gates

### 7.1 Pre-commit hooks (automático via Husky)

A cada `git commit`, o Husky executa:

1. **lint-staged:** ESLint + Prettier nos arquivos staged.
2. **type-check:** `pnpm type-check` no workspace do arquivo alterado.
3. **commitlint:** Valida o formato da mensagem de commit.

```bash
# Se o pre-commit falhar, ver o erro e corrigir:
pnpm lint --filter=api       # corrigir lint no backend
pnpm type-check --filter=web-cedente  # corrigir types no frontend
```

### 7.2 CI em PRs (GitHub Actions)

Todo PR executa automaticamente:

| Step | Comando | Tempo estimado |
|---|---|---|
| Install | `pnpm install --frozen-lockfile` | ~30s |
| Lint | `pnpm lint` | ~20s |
| Type-check | `pnpm type-check` | ~30s |
| Testes unitários | `pnpm test` | ~1-2min |
| Build | `pnpm build` | ~2-3min |

PR só pode ser mergeado com todos os steps verdes.

### 7.3 Cobertura mínima de testes

| Camada | Cobertura mínima | Ferramenta |
|---|---|---|
| Backend — lógica de negócio (services) | 80% | Vitest |
| Backend — endpoints críticos | 100% dos happy paths | Supertest |
| Frontend — componentes críticos | 70% | Vitest + RTL |
| Mobile — screens críticas | 70% | Jest + RNTL |

Componentes críticos do Cedente: wizard de cadastro, simulador, aceite de proposta, upload de documento, painel financeiro, Guardião do Retorno.

---

## 8. Referências

- Doc 02 — Stacks Tecnológicas: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/02 - Stacks.md`
- Doc 24 — Deploy, CI/CD e Versionamento: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/24 - Deploy, CI-CD e Versionamento.md`
- Conventional Commits: [conventionalcommits.org](https://www.conventionalcommits.org)
- commitlint: [commitlint.js.org](https://commitlint.js.org)
- Turborepo Pipelines: [turbo.build/repo/docs/core-concepts/pipelines](https://turbo.build/repo/docs/core-concepts/pipelines)
