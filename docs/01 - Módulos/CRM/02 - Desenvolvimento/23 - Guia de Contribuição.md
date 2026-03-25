# 23 - Guia de Contribuição

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — Backend, Frontend |
| **Escopo** | Gitflow, convenção de commits, PR template, code review checklist, padrões de código e processo de release |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |

---

> **TL;DR**
>
> - **Gitflow:** `main` (produção) / `develop` (integração) / `feature/*` / `hotfix/*`.
> - **Commits:** Conventional Commits obrigatórios. Mensagens em português do Brasil. Tipos: `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`, `ci`.
> - **PR:** mínimo 1 aprovação para `develop`; 2 aprovações para `main`. PR sem testes para nova feature = bloqueado.
> - **Cobertura mínima:** 80% de branches em novos serviços. CI falha se cair abaixo.
> - **Release:** SemVer automático via `semantic-release`. CHANGELOG gerado automaticamente.

---

## 1. Gitflow

### 1.1 Estrutura de Branches

| Branch | Propósito | Deploy automático |
|---|---|---|
| `main` | Código em produção — sempre estável | Vercel produção + Railway produção |
| `develop` | Integração contínua — base de todas as features | Vercel preview (staging) |
| `feature/crm-NNN-descricao` | Desenvolvimento de nova feature | Vercel preview por PR |
| `hotfix/crm-NNN-descricao` | Correção crítica em produção | Via PR direto para `main` (e `develop`) |
| `release/vX.Y.Z` | Preparação de release — apenas bugfixes e docs | Staging de validação |

### 1.2 Fluxo de Feature

```
develop
  │
  ├─ feature/crm-042-pipeline-kanban  (branch de desenvolvimento)
  │      ↓
  │   [commits de desenvolvimento]
  │      ↓
  │   [PR aberto → code review → CI verde → merge]
  │      ↓
  └─ develop  (integrado)
         ↓
      [release/v1.1.0]
         ↓
      main (produção)
```

### 1.3 Fluxo de Hotfix

```
main (produção com bug)
  │
  ├─ hotfix/crm-078-fix-sla-alert
  │      ↓
  │   [commit de correção]
  │      ↓
  │   [PR para main + PR para develop]
  │      ↓
  └─ main + develop (sincronizados)
```

### 1.4 Regras de Branch

- **Nunca** commitar diretamente em `main` ou `develop` — sempre via PR.
- Branch de feature nasce sempre a partir de `develop`.
- Hotfix nasce a partir de `main` e é mergeado tanto em `main` quanto em `develop`.
- Branches de feature devem ser deletadas após merge (GitHub: "Delete branch" automático).
- Nomenclatura: `feature/crm-[numero]-[descricao-curta-em-kebab-case]`.

---

## 2. Convenção de Commits

O módulo CRM adota o padrão **Conventional Commits** (https://www.conventionalcommits.org). Mensagens em **português do Brasil**. Termos técnicos (nomes de funções, variáveis, arquivos) ficam em inglês.

### 2.1 Formato

```
<tipo>(<escopo>): <descrição curta no imperativo>

[corpo opcional — contexto, motivação, detalhes]

[rodapé opcional — referências de issues, breaking changes]
```

### 2.2 Tipos

| Tipo | Quando usar | Exemplo |
|---|---|---|
| `feat` | Nova funcionalidade | `feat(pipeline): adiciona visualização kanban com drag-and-drop` |
| `fix` | Correção de bug | `fix(sla): corrige cálculo de prazo quando data cai em feriado nacional` |
| `chore` | Tarefas técnicas sem impacto no usuário | `chore(deps): atualiza @nestjs/common para v10.4` |
| `docs` | Alteração de documentação | `docs(api): adiciona exemplos de resposta no endpoint de casos` |
| `test` | Adição ou alteração de testes | `test(cases): adiciona cenários de transição de status inválida` |
| `refactor` | Refatoração sem mudança de comportamento | `refactor(commissions): extrai lógica de cálculo para CommissionCalculator` |
| `perf` | Melhoria de performance | `perf(dashboard): adiciona cache Redis para KPIs do painel executivo` |
| `ci` | Alteração em pipelines de CI/CD | `ci: adiciona step de typecheck antes do deploy` |

### 2.3 Escopos do CRM

| Escopo | Descrição |
|---|---|
| `auth` | Autenticação e controle de acesso |
| `cases` | Módulo de Casos |
| `pipeline` | Pipeline kanban e transições |
| `contacts` | Módulo de Contatos |
| `activities` | Módulo de Atividades |
| `negotiations` | Módulo de Negociações |
| `commissions` | Módulo de Comissões |
| `dossier` | Módulo de Dossiê |
| `reports` | Módulo de Relatórios |
| `team` | Gestão de Equipe |
| `settings` | Configurações do sistema |
| `sla` | Monitor de SLA e alertas |
| `notifications` | Notificações e filas |
| `integrations` | Integrações externas (ZapSign, Celcoin, Meta, Resend) |
| `infra` | Infraestrutura, Docker, CI |
| `deps` | Dependências |

### 2.4 Breaking Changes

Breaking changes devem ser indicados com `!` após o tipo/escopo e descritos no rodapé:

```
feat(api)!: altera formato de resposta do endpoint de casos

BREAKING CHANGE: campo `assigned_to` renomeado para `assigned_analyst_id`.
Clientes que consomem este endpoint devem atualizar suas integrações.

Refs: CRM-142
```

### 2.5 Exemplos Corretos

```bash
feat(cases): implementa criação de Caso com snapshot de configuração
fix(dossier): corrige upload de PDF com nome contendo caracteres especiais
test(commissions): adiciona testes unitários para cálculo de comissão nos cenários A/B/C/D
refactor(sla): extrai SlaMonitorService para módulo separado
chore(deps): atualiza prisma para v6.5.0
docs(contribuicao): adiciona exemplos de escopos no guia de commits
perf(reports): substitui query N+1 por join com paginação cursor-based
```

---

## 3. Pull Request Template

Todo PR deve preencher o template abaixo (arquivo `.github/pull_request_template.md`):

```markdown
## Descrição

<!-- O que esta PR faz? Por que é necessária? -->

## Tipo de mudança

- [ ] feat — nova funcionalidade
- [ ] fix — correção de bug
- [ ] refactor — refatoração sem mudança de comportamento
- [ ] perf — melhoria de performance
- [ ] test — adição/alteração de testes
- [ ] docs — documentação
- [ ] chore — manutenção técnica
- [ ] ci — pipeline CI/CD

## Como testar

<!-- Passos detalhados para testar esta PR. Inclua credenciais de teste se necessário. -->

1.
2.
3.

## Checklist

- [ ] Testes adicionados/atualizados para as mudanças
- [ ] Cobertura >= 80% nos módulos alterados (verificado com `pnpm test --coverage`)
- [ ] TypeScript compila sem erros (`pnpm typecheck`)
- [ ] Lint passando (`pnpm lint`)
- [ ] Nenhuma variável de ambiente nova sem documentação no `22 - Guia de Ambiente`
- [ ] Migrations sem breaking changes (verificado em staging)
- [ ] Nenhum `console.log` de debug esquecido
- [ ] `CHANGELOG.md` não precisa de edição manual (gerado automaticamente)

## RN rastreada

<!-- Qual Regra de Negócio ou RF esta PR implementa ou corrige? -->
RN-XXX / RF-XXX

## Screenshots (se aplicável)

<!-- Antes / Depois — especialmente para mudanças de UI -->
```

---

## 4. Code Review Checklist

O revisor deve validar todos os itens abaixo antes de aprovar:

| # | Item | Critério |
|---|---|---|
| 1 | **Correção funcional** | O código faz o que a descrição da PR diz? Testei localmente? |
| 2 | **Testes** | Há testes cobrindo os cenários happy path e edge cases da mudança? Cobertura >= 80% nos módulos tocados? |
| 3 | **TypeScript strict** | Nenhum `any` implícito. Tipos explícitos em assinaturas de funções públicas. `strict: true` respeitado. |
| 4 | **Segurança** | Guards NestJS presentes nos endpoints novos? Papéis corretos validados? Nenhum dado sensível retornado sem necessidade? |
| 5 | **LGPD / PII** | Nenhum CPF, e-mail ou nome de usuário real nos logs de aplicação (Pino)? |
| 6 | **Performance** | Queries Prisma com `select` explícito (sem `findMany` sem limites)? Cache Redis usado onde aplicável (KPIs, parâmetros)? |
| 7 | **Migrations** | Se há migration nova: é aditiva (backward-compatible)? Sem `DROP COLUMN` ou `ALTER` destrutivo sem rollback planejado? |
| 8 | **Convenção de código** | Nomenclatura em `camelCase` (TypeScript) e `snake_case` (banco/Prisma)? Arquivos organizados dentro de `apps/api/modules/crm/` ou `apps/web-crm/`? |
| 9 | **Sem duplicação** | Lógica nova não duplica algo já existente em serviço compartilhado (`packages/`)? |
| 10 | **Sem `console.log`** | Nenhum `console.log`, `console.debug` ou `console.error` esquecido — usar Pino (`this.logger`) no backend. |

---

## 5. Padrões de Código

### 5.1 TypeScript

- `strict: true` obrigatório em todo `tsconfig.json`. Nunca desativar.
- Nenhum arquivo `.js` ou `.jsx` — apenas `.ts` e `.tsx`.
- `any` explícito proibido. Usar `unknown` + type guard quando necessário.
- Tipos de retorno explícitos em funções públicas de serviços NestJS.
- Enums TypeScript apenas para tipos de UI. Para banco, usar enums do Prisma.

### 5.2 ESLint e Prettier

Configurações na raiz do monorepo (`eslint.config.mjs`, `prettier.config.mjs`). Não alterar sem ADR aprovado.

```bash
# Verificar lint
pnpm lint

# Corrigir automaticamente
pnpm lint --fix

# Formatar código
pnpm format
```

### 5.3 NestJS (Backend)

- Todo módulo CRM em `apps/api/modules/crm/[sub-modulo]/`.
- Controller recebe apenas DTOs validados com `class-validator`.
- Service contém toda lógica de negócio — Controllers são apenas roteadores.
- Guards NestJS em todo endpoint autenticado — nunca validar papel no service.
- Logs via `PinoLogger` injetado — nunca `console.log`.
- Prisma queries: sempre usar `select` explícito em `findMany`. Nunca retornar todos os campos de tabelas com dados sensíveis.

### 5.4 Next.js (Frontend)

- App Router obrigatório. Pages Router proibido.
- `use client` apenas em componentes com estado (`useState`, `useEffect`) ou event handlers.
- Server Components para carregamento inicial de dados dos Casos.
- Toda chamada de API via TanStack Query — fetch manual em componentes proibido.
- Skeleton screens obrigatórios durante loading — spinners globais proibidos (exceto em botões de ação).
- Acessibilidade: todo elemento interativo com `aria-label` quando o texto não é descritivo.

### 5.5 Testes Obrigatórios para Novas Features

Toda PR com `feat` DEVE incluir testes:

| Tipo de feature | Testes obrigatórios |
|---|---|
| Novo endpoint NestJS | Teste unitário do service + teste de integração com Supertest |
| Nova lógica de negócio (cálculo, validação) | Testes unitários cobrindo todos os branches |
| Nova tela/componente de UI | Teste de renderização com `@testing-library/react` |
| Novo fluxo crítico (ex: avanço de pipeline) | Teste E2E com Playwright |
| Nova migration Prisma | Teste de integração validando constraints e soft-delete |

**Cobertura mínima:** 80% de branches nos módulos novos ou alterados. Verificar com:

```bash
pnpm --filter api test --coverage
# Checar relatório em coverage/lcov-report/index.html
```

---

## 6. Processo de Release

### 6.1 Versionamento Semântico (SemVer)

O CRM segue **Semantic Versioning 2.0.0** (`MAJOR.MINOR.PATCH`):

| Tipo | Quando incrementar | Exemplo |
|---|---|---|
| `PATCH` | Bug fix sem breaking change | `1.0.0 → 1.0.1` |
| `MINOR` | Nova feature backward-compatible | `1.0.1 → 1.1.0` |
| `MAJOR` | Breaking change | `1.1.0 → 2.0.0` |

### 6.2 Fluxo de Release

```bash
# 1. Criar branch de release a partir de develop
git checkout develop
git pull origin develop
git checkout -b release/v1.1.0

# 2. Validação final (apenas bugfixes nesta branch)
# Executar suite completa de testes:
pnpm test
pnpm typecheck
pnpm lint

# 3. Abrir PR de release/v1.1.0 → main
# PR com título: "release: v1.1.0"

# 4. Após merge em main — semantic-release cria tag automaticamente
# Tag: v1.1.0
# CHANGELOG.md atualizado automaticamente com commits desde v1.0.x

# 5. Fazer merge de main de volta em develop (para sincronizar bugfixes)
git checkout develop
git merge main
git push origin develop
```

### 6.3 CHANGELOG Automático

O `CHANGELOG.md` é gerado automaticamente pelo `semantic-release` com base nos commits Conventional Commits. **Nunca editar manualmente.**

```bash
# semantic-release rodará automaticamente no CI ao fazer merge em main
# Configurado em .releaserc.json na raiz do monorepo
```

### 6.4 Hotfix em Produção

```bash
# 1. Criar branch de hotfix a partir de main
git checkout main
git pull origin main
git checkout -b hotfix/crm-078-fix-sla-alert

# 2. Implementar a correção + testes

# 3. PR para main (revisão obrigatória de Tech Lead)
# 4. Após merge em main: semantic-release cria tag de PATCH

# 5. Sincronizar hotfix em develop
git checkout develop
git merge main
git push origin develop
```

---

## 7. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — Gitflow, Conventional Commits com escopos do CRM, PR template, code review checklist (10 itens), padrões TypeScript/NestJS/Next.js, cobertura mínima, processo de release SemVer + CHANGELOG automático. |
