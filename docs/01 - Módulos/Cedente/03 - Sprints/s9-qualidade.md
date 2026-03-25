# S9 — Qualidade

## Módulo Cedente · Repasse Seguro · Sprint 9

| Campo             | Valor                                                                                   |
| ----------------- | --------------------------------------------------------------------------------------- |
| **Sprint**        | S9 — Qualidade                                                                          |
| **Tipo**          | Template B — Módulo Fullstack por Feature                                               |
| **Docs Fonte**    | D27 (Plano de Testes), D28 (Checklist de Qualidade), D24 (CI/CD), D25 (Observabilidade) |
| **REQs cobertos** | REQ-202, REQ-203, REQ-204, REQ-205, REQ-206, REQ-207, REQ-208, REQ-210, REQ-211         |
| **Dependências**  | S1–S8 todos implementados e funcionais                                                  |
| **Status**        | Pendente                                                                                |

---

## Contexto

Esta sprint finaliza a infraestrutura de qualidade do módulo Cedente. Não adiciona lógica de negócio nova. Foco em: pipeline CI/CD completo, cobertura de testes mínima por camada (70% unit / 20% integração / 8% E2E / 2% manual), observabilidade (Sentry, PostHog, Langfuse, Pino), gates de PR e release, e acessibilidade WCAG 2.1 AA.

---

## FEATURE 1 — Pipeline CI/CD (REQ-202, REQ-203, REQ-204, REQ-205)

### CI/CD — GitHub Actions

- [x] **`ci.yml`** — workflow de PR check em todo PR para `develop` e `main`:
  - Trigger: `on: pull_request: branches: [develop, main]`
  - Steps obrigatórios na ordem:
    1. `actions/checkout@v4`
    2. `pnpm/action-setup@v4` versão `9`
    3. `actions/setup-node@v4` com `node-version: '22'` e `cache: 'pnpm'`
    4. `pnpm install --frozen-lockfile`
    5. `pnpm lint`
    6. `pnpm type-check`
    7. `pnpm test` com `DATABASE_URL: postgresql://postgres:postgres@localhost:5432/repasse_test`
    8. `pnpm build`
  - `timeout-minutes: 15`
  - Merge bloqueado se qualquer step falhar
  - Validar: abrir PR para `develop` → workflow `ci.yml` dispara; `pnpm lint` com `console.log` → CI vermelho; `pnpm type-check` com `any` implícito → CI vermelho

- [x] **`staging.yml`** — deploy automático ao fazer merge em `develop`:
  - Trigger: `on: push: branches: [develop]`
  - Job `deploy-backend-staging`:
    1. `pnpm prisma migrate deploy` com `DATABASE_URL: ${{ secrets.SUPABASE_STAGING_DATABASE_URL }}` executado ANTES do deploy Railway
    2. `bervProject/railway-deploy@main` com `service: cedente-api-staging`
  - Job `deploy-frontend-staging`:
    1. `amondnet/vercel-action@v25` com `vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID_CEDENTE }}` (preview)
  - Validar: merge em `develop` → migration executa → Railway staging deploy → Vercel preview deploy

- [x] **`production.yml`** — deploy automático ao criar tag `v*.*.*` em `main`:
  - Trigger: `on: push: tags: ['v[0-9]+.[0-9]+.[0-9]+']`
  - Job `deploy-backend-production` com `environment: production` (requer aprovação manual no GitHub)
  - Steps: `pnpm prisma migrate deploy` com `SUPABASE_PROD_DATABASE_URL` → `railway-deploy` `service: cedente-api-prod`
  - Job `deploy-frontend-production`: `needs: deploy-backend-production` → `vercel-action` com `--prod`
  - Validar: tag `v1.0.0` em `main` → aprovação manual → migration prod → Railway prod → Vercel prod

- [x] **`deploy-mobile.yml`** — EAS Build ao criar tag `v*.*.*` em `main`:
  - Trigger: `on: push: tags: ['v[0-9]+.[0-9]+.[0-9]+']`
  - Steps: `eas build --profile production --platform all --non-interactive`
  - `EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}`
  - Validar: tag `v1.0.0` → EAS Build disparado para iOS e Android

- [x] **Secrets GitHub configurados** (D24 seção 2.5) — 7 secrets obrigatórios:
  - `RAILWAY_TOKEN`, `SUPABASE_STAGING_DATABASE_URL`, `SUPABASE_PROD_DATABASE_URL`, `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID_CEDENTE`, `SENTRY_AUTH_TOKEN`
  - Validar: `CI.yml` acessa `DATABASE_URL` via secret; nenhum secret exposto em logs de CI

- [x] **Processo de release Semantic Versioning** (D24 seção 4):
  - Formato: `MAJOR.MINOR.PATCH`
  - `CHANGELOG.md` na raiz do monorepo com formato Keep a Changelog
  - Entrada obrigatória por release: `## [X.Y.Z] - YYYY-MM-DD` com seções `### Adicionado`, `### Corrigido`, `### Alterado`
  - Validar: `CHANGELOG.md` presente na raiz; primeira entrada documenta S1–S8

- [x] **Rollback documentado** (D24 seção 6):
  - Backend: `railway rollback --service cedente-api-prod` < 60s (REQ-205)
  - Frontend: Vercel `Promote to Production` para deploy anterior (instantâneo)
  - Mobile: `eas update --channel production --message "revert: ..."` para bundle anterior
  - Trigger de rollback: error rate > 5% em 5min OU latência P95 > 500ms (D24 seção 6.4)
  - Validar: Runbook de rollback documentado em `docs/runbook-rollback.md` referenciando os 3 comandos

- [x] **Migrations Prisma — processo expand/contract** para migrations destrutivas (D24 seção 5.3):
  - Toda migration destrutiva (rename/remove coluna, change type) exige 2 deploys separados
  - Etapa 1 (Expand): adicionar nova coluna nullable; código lê/escreve nas duas
  - Etapa 2 (Contract): após 24h de monitoramento prod → remover coluna antiga; tornar nova NOT NULL
  - Cada arquivo SQL de migration deve conter rollback como comentário: `-- ROLLBACK: ALTER TABLE ...`
  - Validar: revisão de `prisma/migrations/` — cada arquivo tem comentário `-- ROLLBACK` documentado (D28 seção 2.5)

- [x] **`commitlint`** — configurar `@commitlint/config-conventional`:
  - `.commitlintrc.json` na raiz com `{ "extends": ["@commitlint/config-conventional"] }`
  - Formato aceito: `<type>(<scope>): <message>` — `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`
  - Pre-commit hook via `husky`: `commitlint --edit`
  - Validar: commit com mensagem "mudança" → hook rejeita; commit com "fix(auth): corrigir bloqueio de login" → hook aprova

---

## FEATURE 2 — Cobertura de Testes (REQ-202, REQ-210, REQ-211)

### Testes Unitários (Vitest) — Meta 70%

- [x] **Configuração Vitest** — `vitest.config.ts` com thresholds por módulo:
  - `CommissionCedenteService`: cobertura 100% (branch + line + function) — P0 financeiro
  - `EscrowCedenteService`: cobertura 100% — P0 financeiro
  - `SimuladorCenariosService`: cobertura 100% — decisão do Cedente
  - `GuardsAuth + RBAC`: cobertura 95%
  - `PropostasService`, `EscalonamentoService`, `DossieService`: cobertura 80%
  - `NotificacoesService`: cobertura 70%
  - Componentes frontend com lógica (wizard, formulários): cobertura 80%
  - Validar: `pnpm test --coverage` → CI falha se qualquer threshold não atingido

- [x] **`CommissionCedenteService` — 100% coverage** (D27 seção 3.5):
  - `it('should return 20% × (200000 − 100000) = 20000 for Cenário B with valorPago=200000, valorRecuperado=200000')`
  - `it('should return 20% × (260000 − 100000) = 32000 for Cenário C with valorPago=200000, valorRecuperado=260000')`
  - `it('should return 20% × (300000 − 100000) = 40000 for Cenário D with valorPago=200000, valorRecuperado=300000')`
  - `it('should return 0 for Cenário A')`
  - `it('should throw ValidationError when valorPagoCedente is 0 or negative')`
  - `it('should return exactly 50% of valorPagoCedente for calculateValorDistratoReferencia(200000) → 100000')`
  - `it('should handle cents: calculateValorDistratoReferencia(150500) → 75250')`
  - Todos os cálculos em BigInt (nunca floats)
  - Validar: `pnpm test CommissionCedenteService` → 100% passando; resultado para B com valorPago=400000 → comissão=40000

- [x] **`EscalonamentoService` — 80% coverage** (D27 seção 3.6):
  - `it('should allow D → C (one step down)')`
  - `it('should block D → B (skip): throw Escalonamento não pode pular cenários')`
  - `it('should block D → A (skip): throw Escalonamento não pode pular cenários')`
  - `it('should block when cenario atual is A: throw Cenário A é o mínimo')`
  - `it('should block if last escalonamento < 7 days ago')`
  - `it('should allow if last escalonamento > 7 days ago')`
  - Validar: `pnpm test EscalonamentoService` → todos passando

- [x] **`EscrowCedenteService` — 100% coverage** (D27 seção 3.7):
  - `it('should distribute valorLiquidoCedente after 15 days with no reversal: valorTotal=300000, comissaoRS=40000 → valorLiquido=260000')`
  - `it('should block distribution if within 15-day reversal period (10 dias após fechamento → throw Período de reversão ainda ativo)')`
  - `it('should return full estorno 300000 when desistência approved within 15 days, valorLiquidoCedente=0')`
  - Validar: `pnpm test EscrowCedenteService` → 100% passando

- [x] **T-038 (RLS isolation test)** — obrigatório antes de cada release (D01.5):
  - Criar `cedente_A` e `cedente_B` com dados isolados
  - Tentar `GET /cedentes/:id_de_A/casos` autenticado como `cedente_B` → 403
  - Verificar que resposta não vaza nenhum dado de `cedente_A`
  - Log de segurança gerado na tentativa de acesso cruzado
  - Validar: `pnpm test security.spec.ts` → T-038 passando; execução obrigatória pré-release confirmada em checklist D29

### Testes de Integração (Vitest + Supertest) — Meta 20%

- [x] **Setup de integração** (D27 seção 4.2):
  - PostgreSQL real via Docker Compose (não mock)
  - Redis real via Docker Compose
  - ZapSign: mockado via `vi.mock()` ou MSW
  - Escrow: mockado via `vi.mock()`
  - Supabase Storage: mockado via `vi.mock('@supabase/supabase-js')`
  - `beforeAll`: `TRUNCATE TABLE casos, documentos, propostas, envelopes_assinatura, contas_escrow CASCADE`
  - `afterEach`: `DELETE FROM casos WHERE id LIKE 'test-%'`
  - Validar: `pnpm test:integration` → setup/teardown sem erro; banco limpo entre testes

- [x] **INT-020 — Webhook ZapSign com validação HMAC-SHA256** (D28 seção 6.2):
  - `POST /webhooks/zapsign` com assinatura HMAC-SHA256 válida → 200
  - `POST /webhooks/zapsign` sem assinatura → 401
  - `POST /webhooks/zapsign` com assinatura inválida → 401
  - Validar: `pnpm test:integration INT-020` → 3 casos passando; nenhum webhook sem HMAC é processado

- [x] **SEC-001 a SEC-005 — Isolamento LGPD** (D27, D28 seção 6.2):
  - SEC-001: dados de `cedente_A` não acessíveis autenticado como `cedente_B` (T-038)
  - SEC-002: `cessionario_id` / nome / CPF / e-mail do Cessionário ausentes em qualquer response do módulo Cedente
  - SEC-003: CPF / e-mail / telefone do Cedente redactados nos logs Pino
  - SEC-004: `cedente_id` nunca enviado ao Langfuse
  - SEC-005: tokens em `expo-secure-store` — AsyncStorage não contém refresh token
  - Validar: `pnpm test:integration security/` → SEC-001 a SEC-005 passando

### Testes E2E (Playwright + RNTL) — Meta 8%

- [x] **Configuração Playwright** — `playwright.config.ts`:
  - Browsers: Chromium, Firefox, WebKit (desktop)
  - Viewport: 1280×720 padrão
  - `baseURL: process.env.PLAYWRIGHT_BASE_URL` (staging para CI, localhost para local)
  - `axe-core` integrado via `@axe-core/playwright`: rodar em toda página antes de assertions
  - Validar: `pnpm e2e` em staging → Playwright abre em 3 browsers sem erro de configuração

- [x] **E2E P0 — fluxos críticos do Cedente** (D28 seção 6.2 — gate de release):
  - E2E-001: Cadastro completo → wizard 5 etapas → Termo de Cadastro gerado → status `CADASTRO_REALIZADO`
  - E2E-002: Upload de documento → MIME validation → status `EM_ANALISE`
  - E2E-003: Receber proposta → aceitar → status `EM_FORMALIZACAO` → `contas_escrow` criada
  - E2E-004: Receber proposta → recusar → status `DISPONIVEL_PARA_COMPRADORES`
  - E2E-005: Solicitar escalonamento D→C → cooldown bloqueado nos 7 dias seguintes
  - E2E-006: Desistência pós-Fechamento → dentro de 15 dias corridos → status `DESISTENCIA_EM_ANALISE`
  - E2E-007: Guardião — enviar mensagem → resposta SSE streaming → rate limit após 30 mensagens/hora
  - Validar: `pnpm e2e --project=chromium` em staging → E2E-001 a E2E-007 todos verdes; falha em qualquer um → PR para `main` bloqueado

- [x] **Acessibilidade WCAG 2.1 AA — axe-core via Playwright**:
  - Executar `axe-core` em: `/login`, `/cadastro`, `/dashboard`, `/casos/[id]`, `/propostas`, `/assinaturas`, `/guardiao`, `/perfil`
  - Threshold: Lighthouse Accessibility score ≥ 95 em todas as páginas
  - Itens WCAG obrigatórios (D28 seção 3):
    - Contraste texto normal ≥ 4.5:1 (`#0069A8` sobre `#FFFFFF` → 5.1:1 ✅)
    - Contraste texto grande ≥ 3:1
    - Toda funcionalidade acessível por teclado (Tab, Enter, Space, Esc, setas)
    - Focus ring visível em todos os elementos interativos; `outline: none` proibido sem substituto
    - Focus trap em modais (aceite proposta, recusa, contraproposta, desistência, cancelamento)
    - Focus restaurado para elemento que abriu overlay ao fechar
    - Skip link "Pular para conteúdo principal" presente e visível no primeiro Tab
    - Labels associados a inputs via `htmlFor` ou `aria-label`
    - `role="dialog"` em modais, `role="alert"` em erros urgentes, `role="status"` em loading
    - `<html lang="pt-BR">`
    - `role="log"` + `aria-live="polite"` no chat do Guardião
    - Stepper wizard: `aria-current="step"` na etapa ativa
  - Validar: `pnpm e2e:a11y` → zero violações axe-core em todas as 8 páginas; Lighthouse A11y ≥ 95

---

## FEATURE 3 — Observabilidade (REQ-206, REQ-207, REQ-208)

### Sentry — Error Tracking (REQ-206)

- [x] **Sentry NestJS** — `apps/api/src/main.ts`:
  - `Sentry.init({ dsn: process.env.SENTRY_DSN, environment: process.env.NODE_ENV, release: process.env.npm_package_version, tracesSampleRate: 0.1 em prod / 1.0 em dev })`
  - `ignoreErrors: ['UnauthorizedException', 'NotFoundException']`
  - `SentryGlobalFilter` no `AppModule` via `APP_FILTER`
  - `beforeSend`: redactar CPF, e-mail, telefone do Cedente antes de enviar ao Sentry
  - Validar: deploy sem `SENTRY_DSN` → erro na inicialização (BLOQUEANTE conforme D25); erro 500 real → aparece no Sentry dashboard com `release` correto

- [x] **Sentry Next.js** — `apps/web-cedente/sentry.client.config.ts` e `sentry.server.config.ts`:
  - `Sentry.init({ dsn: process.env.NEXT_PUBLIC_SENTRY_DSN, environment, release: process.env.NEXT_PUBLIC_APP_VERSION })`
  - Session replay configurado (D25 TL;DR)
  - `SENTRY_AUTH_TOKEN` em Vercel para upload de source maps
  - Validar: erro JavaScript no browser → capturado no Sentry com session replay; source map disponível no Sentry

- [x] **Sentry Expo (mobile)** — `apps/mobile-cedente/app/_layout.tsx`:
  - `@sentry/react-native` inicializado com DSN correto
  - `release` configurado via `expo-constants` e `manifest.version`
  - Validar: crash nativo simulado → evento capturado no Sentry com stack trace

- [x] **Alertas Sentry** (D28 seção 5.2):
  - Alerta: novos erros P0 → PagerDuty (D20 REQ-197)
  - Alerta: aumento de taxa de erro > 2% em 5min → Sentry Alert Rule configurado
  - Validar: criar `AlertRule` no Sentry projeto → threshold 2% em 5min → notificação configurada

### PostHog — Analytics (REQ-207)

- [x] **`src/services/analytics.ts`** — wrapper centralizado PostHog (D28 seção 2.1):
  - Exportar função `trackEvent(event: string, properties?: Record<string, unknown>)`
  - Exportar função `identifyUser(cedenteId: string, properties?: Record<string, unknown>)`
  - Nunca chamar `posthog.capture()` diretamente em componentes — apenas via `analytics.ts`
  - Session replay habilitado em produção
  - Validar: chamada direta de `posthog.capture()` em componente → ESLint `no-direct-posthog` rule → erro de lint

- [x] **10 eventos PostHog obrigatórios** — verificados em staging (D28 seção 5.4):
  - `caso_cadastrado`: propriedades `cenario_escolhido`, `tipo_cedente` (PF/PJ) — dispara ao confirmar wizard Etapa 5
  - `cenario_selecionado`: propriedades `cenario`, `tempo_visualizacao_simulador_segundos` — dispara na Etapa 4 ao selecionar cenário
  - `proposta_aceita`: propriedades `valor_proposta`, `valor_liquido_cedente`, `cenario` — dispara ao confirmar aceite
  - `proposta_recusada`: propriedades `motivo_interno` (se fornecido) — dispara ao confirmar recusa
  - `contraproposta_enviada`: propriedades `valor_contraproposta`, `cenario` — dispara ao enviar
  - `escalonamento_solicitado`: propriedades `cenario_anterior`, `cenario_novo` — dispara ao confirmar
  - `documento_enviado`: propriedades `tipo_documento`, `tentativas` — dispara ao upload bem-sucedido
  - `assinatura_concluida`: propriedades `tipo_documento_assinado` — dispara ao receber webhook ZapSign
  - `guardiao_consulta`: propriedades `modo` (`normal` | `cadastro_assistido`) — dispara a cada mensagem enviada
  - `desistencia_solicitada`: propriedades `dias_apos_fechamento` — dispara ao submeter formulário
  - Validar: `pnpm e2e:analytics` em staging → PostHog debug mode confirma 10 eventos disparando com properties corretas

### Langfuse — Observabilidade IA (REQ-208)

- [x] **Langfuse tracing** — já implementado em S7; validar em staging (D28 seção 5.3):
  - Todo trace inclui: `cedente_id` anonimizado (hash SHA-256), `session_id`, latência, tokens consumidos, modelo `gpt-4-turbo-2024-04-09`
  - `cedente_id` NUNCA enviado em texto limpo ao Langfuse (SEC-004)
  - Alerta: latência Guardião p95 > 5s → alerta configurado no Langfuse
  - Alerta: taxa de erro do Guardião > 5% → alerta configurado
  - Validar: conversa com Guardião em staging → trace aparece no Langfuse dashboard com campos corretos; `cedente_id` hash (não plain text) no trace

### Pino — Logs Estruturados (D25, REQ-209 — já base S1)

- [x] **Validação dos logs Pino** — garantir conformidade dos campos obrigatórios (D25):
  - Campos obrigatórios em TODOS os logs: `requestId`, `userId`, `module`, `action`
  - Dados sensíveis redactados: CPF → `[REDACTED_CPF]`, e-mail → `[REDACTED_EMAIL]`, telefone → `[REDACTED_PHONE]`
  - Logs de texto livre proibidos; apenas JSON estruturado
  - Eventos de negócio em nível `info`: aceite proposta, escalonamento, assinatura documento, abertura/distribuição Escrow
  - Erros de integração ZapSign e Escrow: nível `error` com payload de entrada + resposta + correlation_id
  - Nível por ambiente: `debug` dev, `info` staging, `warn` produção
  - Validar: `curl /api/v1/casos` → log de request com `requestId`, `userId`, `module=casos`, `action=listCasos`; CPF no log → aparece como `[REDACTED_CPF]`

---

## FEATURE 4 — Gates de PR e Release

### Gates de PR (REQ-210)

- [x] **ESLint rules** para gates de PR (D28 seção 6.1):
  - `no-console`: erro em todos os arquivos de produção (não em testes)
  - `no-explicit-any`: erro em todo TypeScript (strict mode)
  - `no-direct-posthog`: rule customizada bloqueando `posthog.capture()` fora de `analytics.ts`
  - `no-react-native-image`: rule customizada bloqueando `import { Image } from 'react-native'` (usar `expo-image`)
  - Validar: `pnpm lint` com `console.log` → erro; `pnpm lint` com `any` implícito → erro

- [x] **Verificação de secrets no código** — bloqueante de PR (D28 seção 4.4):
  - `pnpm audit --audit-level=high` sem vulnerabilidades críticas
  - GitHub Secret Scanning habilitado no repositório
  - `.env.example` presente sem valores reais; todas as 42 variáveis documentadas (D22 REQ-201)
  - Validar: tentar commitar `API_KEY=sk-xxx` → GitHub Secret Scanning alerta; `pnpm audit` em CI → falha com vulnerabilidade crítica

- [x] **Checklist de PR automatizado** — template em `.github/pull_request_template.md`:
  - Seções obrigatórias:
    - `### Código`: TypeScript sem any, console.log ausente, queries filtradas por `cedente_id`, dados Cessionário ausentes
    - `### Banco de dados`: RLS em nova tabela, migration com rollback no SQL, sem breaking changes retroativos
    - `### API`: erro RFC 7807, endpoint documentado no Swagger (`@ApiOperation` + `@ApiResponse`)
    - `### Regras de negócio`: comissão fórmula correta, escalonamento server-side, MIME real, HMAC-SHA256 ZapSign
    - `### Observabilidade`: eventos PostHog, Sentry para novos tipos de erro
  - Validar: PR aberto → template preenchido automaticamente; revisor marca todos os checkboxes antes do merge

### Gates de Release (REQ-211)

- [x] **Gates de release documentados** — `docs/checklist-release.md`:
  - E2E P0 (E2E-001 a E2E-007) todos verdes em staging
  - `CommissionCedenteService` 100% coverage
  - `EscrowCedenteService` 100% coverage
  - T-038 (RLS isolation) passando
  - SEC-001 a SEC-005 passando
  - INT-020 (webhook HMAC) passando
  - Sentry inicializado e recebendo eventos de staging
  - PostHog capturando os 10 eventos obrigatórios em staging
  - Langfuse recebendo traces do Guardião em staging
  - `pnpm audit --audit-level=high` sem vulnerabilidades críticas
  - Latência P95 < 200ms em endpoints críticos (k6 load test em staging)
  - Dados de teste removidos de staging
  - Validar: `docs/checklist-release.md` presente; cada item tem responsável e evidência esperada

- [x] **Health check endpoint** — `GET /health` (D28 seção 5.5, D24 seção 3.1):
  - Resposta: `{ "status": "ok", "timestamp": string }` HTTP 200 em < 500ms
  - Inclui verificação de: conexão Postgres, Redis, ping ZapSign, ping Escrow (D28 seção 5.5)
  - Railway health check configurado: path `/health`, timeout 30s, interval 10s, reinicializar após 3 falhas consecutivas
  - Validar: `curl https://api-cedente-staging.up.railway.app/health` → HTTP 200 com `status: ok`; Postgres down → health retorna `status: degraded` e Railway reinicia instância

- [x] **Validação RLS pré-deploy** (D28 seção 2.5):
  - Comando: `SELECT * FROM pg_policies` em staging → verificar todas as tabelas com dados de Cedente têm RLS habilitado
  - Tabelas obrigatórias: `users`, `cedentes`, `casos`, `dossies`, `documentos_dossie`, `propostas`, `envelopes_assinatura`, `contas_escrow`, `notificacoes`, `eventos_caso`, `ai_sessions`, `push_tokens`
  - Validar: `pnpm db:check-rls` → script que executa `pg_policies` query e retorna erro se alguma tabela sem policy

---

## FEATURE 5 — Acessibilidade e Performance (REQ-210)

### Acessibilidade Web

- [x] **Error Boundary** em rotas críticas (D28 seção 2.1):
  - `ErrorBoundary` global em `app/layout.tsx`
  - `ErrorBoundary` por rota crítica: wizard de cadastro, propostas, assinaturas, Guardião
  - Validar: componente filho lança erro → ErrorBoundary renderiza fallback; erro capturado pelo Sentry

- [x] **`React.lazy()` + `Suspense`** em rotas pesadas (D28 seção 2.1):
  - Wizard de cadastro, modais de Escrow, chat Guardião
  - Validar: `rollup-plugin-visualizer` → bundle inicial sem módulos lazy (lazy separado); chunk do wizard < 150KB gzip

- [x] **Skeleton screens** em vez de spinners genéricos (D28 seção 2.1):
  - Cards de cenário no simulador (Etapa 3): skeleton durante os 10s
  - Lista de propostas: skeleton durante carregamento
  - Painel financeiro: skeleton
  - Validar: simular latência 3s na API → skeleton aparece; spinner genérico proibido em listas de dados

- [x] **Bundle size** — verificação com `rollup-plugin-visualizer` (D28 seção 2.1):
  - Bundle gzipped < 300KB para initial load
  - Validar: `pnpm build:analyze` → relatório visual mostra bundle < 300KB

### Performance API

- [x] **k6 load test** — validação de SLA (D28 seção 6.2):
  - Endpoints críticos: `POST /auth/login`, `GET /cedentes/casos`, `POST /ai/guardiao/message`, `GET /health`
  - Meta: P95 < 200ms para todos (D25 TL;DR)
  - Script: `k6 run scripts/load-test.k6.js` com 50 virtual users, 5min
  - Validar: `k6` em staging → relatório mostra P95 < 200ms em todos os endpoints críticos; se P95 > 200ms → gate de release bloqueado

- [x] **Suporte `prefers-reduced-motion`** (D28 seção 3.2):
  - Wizard: transições de etapa respeitam `prefers-reduced-motion: reduce` via `useReducedMotion()` do Framer Motion
  - Mobile: animações com `Reanimated` reduzidas se `AccessibilityInfo.isReduceMotionEnabled()`
  - Validar: `prefers-reduced-motion: reduce` ativo no OS → animações desabilitadas/reduzidas; axe-core não detecta violação

---

## Auto-Verificação (12 Checks)

- [x] **Check #1 — CI pipeline**: `ci.yml` com 8 steps (install, lint, type-check, test, build) + `staging.yml` (migrate→Railway→Vercel) + `production.yml` (tag v*.*.\* + aprovação manual + migrate→Railway→Vercel) + `deploy-mobile.yml` (EAS Build) — todos os 4 workflows presentes e funcionais (REQ-202, REQ-203, REQ-204)
- [x] **Check #2 — Rollback**: `railway rollback` < 60s backend; Vercel instantâneo; `eas update` para mobile; trigger error rate > 5% OU P95 > 500ms (REQ-205)
- [x] **Check #3 — Cobertura financeira 100%**: `CommissionCedenteService` 100% (7 casos incluindo Cenário A R$0 e erro de input), `EscrowCedenteService` 100% (3 casos incluindo período de reversão), `SimuladorCenariosService` 100% (REQ-210, REQ-211)
- [x] **Check #4 — Sentry em todas as camadas**: NestJS, Next.js, Expo — 3 inicializações; `beforeSend` redacta CPF/e-mail/telefone; `release` configurado em cada deploy; deploy sem Sentry → BLOQUEANTE (REQ-206)
- [x] **Check #5 — PostHog 10 eventos**: `caso_cadastrado`, `cenario_selecionado`, `proposta_aceita`, `proposta_recusada`, `contraproposta_enviada`, `escalonamento_solicitado`, `documento_enviado`, `assinatura_concluida`, `guardiao_consulta`, `desistencia_solicitada` — todos via `analytics.ts` wrapper centralizado; chamada direta → lint error (REQ-207)
- [x] **Check #6 — Langfuse traces**: `cedente_id` hashed (nunca plain text); campos `session_id`, latência, tokens, modelo `gpt-4-turbo-2024-04-09`; alerta p95 > 5s e error rate > 5% configurados (REQ-208)
- [x] **Check #7 — T-038 RLS isolation**: isolamento entre cedentes testado; tentativa de acesso cruzado → 403 + log de segurança; execução obrigatória pré-release (D01.5, D28)
- [x] **Check #8 — SEC-001 a SEC-005**: 5 testes de segurança LGPD passando em CI; `cessionario_id` ausente em todas as responses do módulo Cedente; CPF/e-mail redactados nos logs
- [x] **Check #9 — axe-core WCAG 2.1 AA**: 8 páginas sem violação; Lighthouse A11y ≥ 95; focus trap em modais; skip link presente; `<html lang="pt-BR">`; `role="log"` no Guardião
- [x] **Check #10 — Gates de release documentados**: `docs/checklist-release.md` com todos os 13 gates listados; nenhum gate sem responsável e evidência esperada; REQ-211
- [x] **Check #11 — Migrations com rollback**: todos os arquivos SQL em `prisma/migrations/` com comentário `-- ROLLBACK:`; processo expand/contract documentado para destrutivas; `pnpm prisma migrate deploy` em CI antes de cada deploy backend
- [x] **Check #12 — REQs cobertos**: REQ-202 (ci.yml, staging.yml, production.yml, deploy-mobile.yml ✅), REQ-203 (tag v*.*.\* → Railway prod + Vercel prod ✅), REQ-204 (pnpm prisma migrate deploy ANTES do deploy ✅), REQ-205 (rollback < 60s Railway + Vercel instantâneo ✅), REQ-206 (Sentry NestJS + Next.js + Expo ✅), REQ-207 (PostHog analytics.ts ✅), REQ-208 (Langfuse Guardião ✅), REQ-210 (bloqueantes PR ✅), REQ-211 (bloqueantes release ✅)
