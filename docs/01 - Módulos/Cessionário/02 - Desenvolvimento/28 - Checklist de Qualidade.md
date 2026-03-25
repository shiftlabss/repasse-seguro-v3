# 28 - Checklist de Qualidade

| Campo | Valor |
|---|---|
| **Nome do Documento** | 28 - Checklist de Qualidade |
| **Versão** | v1.0 |
| **Data** | 2026-03-22 (America/Fortaleza) |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Produto** | Repasse Seguro — Módulo Cessionário |

---

> 📌 **TL;DR — Gate de Qualidade**
>
> - **Bloqueantes de PR:** CI vermelho, testes faltando para nova lógica, dado sensível exposto, `console.log` em produção, secrets no código, violações de segurança P0.
> - **Bloqueantes de release:** E2E P0 falhando, cálculo de comissão com erro, dado do Cedente exposto, migration sem rollback documentado, observabilidade não configurada.
> - **4 partes:** Code Review (frontend, backend, API, dados) → Acessibilidade (WCAG 2.1 AA, mobile) → Segurança (auth, dados, transporte, headers) → Observabilidade (logs, alertas, health).
> - **Aplicar em:** todo PR, auditoria quinzenal, pré-release e pós-incidente.
> - **Ferramenta de validação chave:** `axe-core` (a11y), `pnpm audit` (deps), Sentry (erros), Lighthouse (performance/a11y), commitlint (commits).

---

## 1. Como Usar o Checklist

### 1.1 Quando Aplicar

| Contexto | Partes obrigatórias | Quem preenche |
|---|---|---|
| **Code Review (PR)** | Parte 1 completa + itens de segurança relevantes | Revisor do PR |
| **Auditoria quinzenal** | Partes 1, 2, 3 e 4 | Tech Lead |
| **Pré-release (develop → main)** | Todas as partes + tabela de bloqueantes | Tech Lead + QA |
| **Pós-incidente P0/P1** | Parte 3 (segurança) + Parte 4 (observabilidade) | On-call + Tech Lead |

### 1.2 Como Registrar Evidência

- Cada item marcado `[x]` deve ter evidência linkável: link de PR, saída de CLI, screenshot ou comentário inline no code review.
- Item marcado `[ ]` com justificativa "não se aplica" deve ter o motivo registrado.
- Item marcado `[ ]` sem justificativa bloqueia o PR se for item bloqueante.

### 1.3 Sinalização

- **🔴 BLOQUEANTE** — não pode ser ignorado. Impede merge ou deploy.
- **💡 RECOMENDAÇÃO** — melhora qualidade mas não bloqueia se justificado.

---

## 2. Parte 1 — Code Review

### 2.1 Frontend (React + Vite)

- [ ] 🔴 Componentes funcionais exclusivamente. Class components ausentes.
- [ ] 🔴 Nenhuma chamada `fetch` direta em `useEffect`. Usar TanStack Query.
- [ ] 🔴 Estado global gerenciado por Zustand. Redux, MobX, Context API como state manager global ausentes.
- [ ] 🔴 `console.log` ausente em código de produção.
- [ ] 🔴 TypeScript `strict: true`. Nenhum `any` implícito.
- [ ] 🔴 Axios ausente sem ADR aprovado. Usar `fetch` nativo com wrapper em `services/api.ts`.
- [ ] 🔴 PostHog não chamado diretamente nos componentes. Apenas via `src/services/analytics.ts`.
- [ ] 🔴 Error Boundary presente em nível global e por rota crítica.
- [ ] 💡 `React.lazy()` + `Suspense` em rotas e componentes pesados (modais, gráficos, chat IA).
- [ ] 💡 Skeleton screens em vez de spinners genéricos para estados de loading.
- [ ] 💡 Framer Motion `AnimatePresence` em mudanças de rota.
- [ ] 💡 Imagens com `loading="lazy"` e `decoding="async"`. WebP como padrão.
- [ ] 💡 Bundle size gzipped < 300KB verificado com `rollup-plugin-visualizer`.

### 2.2 Backend (NestJS)

- [ ] 🔴 Todo endpoint tem DTO com `class-validator`. Inputs não validados são proibidos.
- [ ] 🔴 Todo endpoint documentado com `@ApiOperation` e `@ApiResponse` no Swagger.
- [ ] 🔴 `console.log` ausente. Usar Pino via `LoggerService`.
- [ ] 🔴 TypeScript `strict: true`. Nenhum `any` implícito.
- [ ] 🔴 Dados do Cedente (nome, CPF, contato) ausentes em qualquer response acessível ao Cessionário.
- [ ] 🔴 Guards de autenticação (`@UseGuards(JwtAuthGuard)`) presentes em todos os endpoints protegidos.
- [ ] 🔴 KYC Guard presente em endpoints que exigem KYC aprovado.
- [ ] 🔴 Rate limiting configurado via `@nestjs/throttler` em endpoints de auth e LLM.
- [ ] 🔴 Correlation ID propagado em todos os logs e responses.
- [ ] 💡 Services testados com `Test.createTestingModule()`. Não testar controllers diretamente.
- [ ] 💡 Prisma queries sem N+1. `include` ou `select` explícito.
- [ ] 💡 TTL definido para toda chave criada no Redis.

### 2.3 API e Contratos

- [ ] 🔴 Schema de erro seguindo `{ "error": { "code", "user_message", "correlation_id", "timestamp" } }`.
- [ ] 🔴 Novos endpoints versionados em `/api/v1/`.
- [ ] 🔴 Status HTTP corretos: 201 (criação), 200 (leitura/atualização), 204 (deleção), 400 (validação), 401 (não autenticado), 403 (não autorizado), 404 (não encontrado), 409 (conflito), 429 (rate limit), 5xx (erro interno).
- [ ] 🔴 Idempotency key em endpoints de criação financeira (Escrow, propostas).
- [ ] 💡 Paginação com cursor ou offset em listagens com > 100 registros esperados.

### 2.4 Banco de Dados (Prisma)

- [ ] 🔴 Primary keys em UUID v4. Auto-increment ausente.
- [ ] 🔴 Timestamps `@db.Timestamptz` em todas as colunas de data. `@db.Timestamp` sem timezone proibido.
- [ ] 🔴 Soft delete em tabelas de domínio (`deleted_at DateTime? @db.Timestamptz`).
- [ ] 🔴 Toda nova migration tem rollback documentado como comentário no arquivo SQL.
- [ ] 🔴 Migrations retrocompatíveis: nova coluna `nullable` antes de ser `NOT NULL`.
- [ ] 🔴 RLS habilitado em todas as tabelas com dados de usuários.
- [ ] 💡 Índice criado para toda nova coluna usada em `WHERE` ou `ORDER BY` frequente.
- [ ] 💡 `created_at` e `updated_at` presentes em toda tabela nova.

### 2.5 Performance

- [ ] 💡 Nenhuma query sem índice em coluna de alta cardinalidade.
- [ ] 💡 Queries com `EXPLAIN ANALYZE` para operações em tabelas com > 10k registros esperados.
- [ ] 💡 Cache Redis para dados de alta leitura e baixa mutação (ex: score de risco de oportunidade).

---

## 3. Parte 2 — Acessibilidade (WCAG 2.1 AA)

### 3.1 Princípio 1: Perceptível

- [ ] 🔴 Contraste de texto normal ≥ 4.5:1 (verificar com axe-core ou Colour Contrast Analyser).
- [ ] 🔴 Contraste de texto grande (≥ 18pt normal / ≥ 14pt bold) ≥ 3:1.
- [ ] 🔴 Contraste de elementos de UI (bordas, ícones, estados) ≥ 3:1.
- [ ] 🔴 Contraste verificado também em dark mode.
- [ ] 🔴 Imagens informativas com `alt` descritivo. Imagens decorativas com `alt=""`.
- [ ] 🔴 HTML semântico: `<button>`, `<nav>`, `<main>`, `<header>`, `<footer>`, `<aside>` em vez de `<div>` genérico.
- [ ] 💡 Vídeos e áudios com legendas ou transcrição (se aplicável).

**Ferramenta:** axe-core integrado ao Playwright; Lighthouse Accessibility score ≥ 95.

### 3.2 Princípio 2: Operável

- [ ] 🔴 Toda funcionalidade acessível por teclado (Tab, Enter, Space, Esc, setas).
- [ ] 🔴 Nenhuma keyboard trap. Foco pode entrar e sair de qualquer componente via teclado.
- [ ] 🔴 Focus ring visível em todos os elementos interativos. `outline: none` proibido sem substituto.
- [ ] 🔴 Tab order lógico seguindo ordem visual da interface.
- [ ] 🔴 Focus trap ativo em modais e drawers enquanto abertos.
- [ ] 🔴 Focus restaurado para o elemento que abriu o overlay ao fechar modal/drawer.
- [ ] 🔴 Skip link "Pular para conteúdo principal" visível no primeiro Tab. Oculto visualmente quando não focado.
- [ ] 🔴 Touch targets (mobile) mínimo 44×44px. Crítico para CTAs "Fazer Proposta", "Confirmar" e botões de KYC.
- [ ] 💡 Animações não essenciais respeitam `prefers-reduced-motion: reduce`. Verificar com media query no CSS e `useReducedMotion()` no Framer Motion.

**Ferramenta:** teste manual com Tab; NVDA (Windows) / VoiceOver (macOS/iOS) para screen reader.

### 3.3 Princípio 3: Compreensível

- [ ] 🔴 Labels associados a inputs via `htmlFor`/`for` ou `aria-label`. Inputs sem label visível: `aria-label` obrigatório.
- [ ] 🔴 Mensagens de erro explicativas e associadas ao campo via `aria-describedby`.
- [ ] 🔴 Stepper de KYC navegável por teclado com estado atual anunciado via `aria-current="step"`.
- [ ] 🔴 Modal de re-autenticação com foco no primeiro elemento interativo ao abrir.
- [ ] 🔴 Linguagem da página declarada: `<html lang="pt-BR">`.
- [ ] 💡 Estados de loading anunciados via `aria-live="polite"` (não urgente) ou `aria-live="assertive"` (crítico).
- [ ] 💡 Empty states com mensagem clara e sugestão de ação. Alt descritivo em ilustrações.

### 3.4 Princípio 4: Robusto

- [ ] 🔴 ARIA roles semânticos corretos: `role="dialog"` em modais, `role="alert"` em mensagens de erro urgentes, `role="status"` em loading não urgente.
- [ ] 🔴 Landmarks corretos: `<main>` único por página, `<nav>` com `aria-label` distintos se múltiplos.
- [ ] 🔴 Tabelas com `<th scope>` correto. Tabelas de dados nunca usadas para layout.
- [ ] 🔴 Formulários com `fieldset` e `legend` para grupos de inputs relacionados (ex: endereço, KYC).
- [ ] 🔴 Componentes shadcn/ui utilizados via cópia no repositório (ownership total) — não via pacote.
- [ ] 💡 Região `aria-live` para anúncios de sucesso/erro em submissões de formulário.
- [ ] 💡 Screen reader flow testado para: login, submissão KYC, criação de proposta, chat com IA.

---

## 4. Parte 3 — Segurança

### 4.1 Autenticação e Sessão

- [ ] 🔴 JWT access token com validade de 15 minutos. Sem token de longa duração.
- [ ] 🔴 Refresh token em cookie `httpOnly; Secure; SameSite=Strict`. Nunca em `localStorage`.
- [ ] 🔴 Token rotation: novo par gerado a cada refresh; token anterior invalidado no Redis.
- [ ] 🔴 Brute force protection: 5 tentativas → lockout 15min via Redis `rs:auth:attempts:{ip}:{email}`.
- [ ] 🔴 Re-autenticação exigida para ações críticas (confirmação de Escrow, assinatura, cancelamento).
- [ ] 🔴 Sessão expirada após 30min de inatividade (Redis TTL `rs:session:{userId}`).
- [ ] 🔴 PKCE implementado no fluxo Google OAuth. State CSRF verificado.

### 4.2 Autorização e Isolamento de Dados

- [ ] 🔴 RLS habilitado em todas as tabelas com dados de usuários no Supabase.
- [ ] 🔴 `service_role` key do Supabase nunca exposta no frontend.
- [ ] 🔴 Toda query ao banco filtrada por `cessionario_id` do usuário logado.
- [ ] 🔴 Dados do Cedente (nome, CPF, email, telefone, cenário escolhido) ausentes em qualquer response do módulo Cessionário.
- [ ] 🔴 RBAC verificado em todos os endpoints com `@UseGuards()` apropriados.
- [ ] 🔴 Agente de IA (Analista de Oportunidades) sem acesso a dados de outros Cessionários.

### 4.3 Validação de Input e Output

- [ ] 🔴 Todo input de usuário validado via `class-validator` no DTO antes de processar.
- [ ] 🔴 Sanitização de prompt do usuário antes de inserir no LLM (`InputSanitizerService`).
- [ ] 🔴 Filtro de output da IA antes de exibir ao usuário (`OutputFilterService` — bloqueia FOMO/urgência).
- [ ] 🔴 Webhook ZapSign validado via HMAC-SHA256 antes de processar.
- [ ] 💡 Upload de arquivos (KYC, comprovantes) via signed URL — nunca multipart direto no NestJS.

### 4.4 Secrets e Configuração

- [ ] 🔴 Nenhuma credencial, token ou API key no código-fonte ou git history.
- [ ] 🔴 Todas as variáveis de ambiente documentadas no `.env.example` sem valores reais.
- [ ] 🔴 Secrets em Railway (backend), Vercel (frontend) e EAS (mobile). Nunca em `.env` commitado.
- [ ] 🔴 `pnpm audit --audit-level=high` sem vulnerabilidades críticas.
- [ ] 💡 Variáveis sensíveis com prefixo `_KEY`, `_SECRET`, `_TOKEN` conforme D22.

### 4.5 Transporte e Headers

- [ ] 🔴 Helmet configurado desde o primeiro commit (headers de segurança automáticos).
- [ ] 🔴 HTTPS obrigatório em todos os ambientes (staging e produção).
- [ ] 🔴 `Content-Security-Policy` configurado para bloquear inline scripts não autorizados.
- [ ] 🔴 `X-Correlation-ID` presente em todas as responses da API.
- [ ] 💡 CORS restrito a origins conhecidos (não `*` em produção).

### 4.6 Logs e Dados Sensíveis

- [ ] 🔴 CPF, senha, token, `access_token`, `SUPABASE_SERVICE_ROLE_KEY`, `OPENAI_API_KEY` presentes no array `redact` do Pino.
- [ ] 🔴 `userId` hasheado antes de enviar ao Langfuse (não usar ID real).
- [ ] 🔴 Nenhum dado pessoal do Cedente em logs de qualquer nível.
- [ ] 🔴 Nenhum dado financeiro (número de conta Escrow, chave PIX) em logs.

---

## 5. Parte 4 — Observabilidade e Operação

- [ ] 🔴 Pino configurado com `redact` completo (ver D25) em todos os serviços.
- [ ] 🔴 Nível de log correto: `error` para falhas, `warn` para degradação, `info` para eventos de negócio, `debug` apenas local.
- [ ] 🔴 Correlation ID propagado em: HTTP request → service → RabbitMQ payload → Sentry → Langfuse.
- [ ] 🔴 Sentry configurado e capturando erros antes do deploy. Deploy sem Sentry é proibido.
- [ ] 🔴 `/health` endpoint retornando estado de saúde do serviço.
- [ ] 🔴 PostHog com eventos obrigatórios instrumentados: `screen_viewed`, `signup_completed`, `proposal_created`, `escrow_deposited`, `document_signed`.
- [ ] 🔴 Langfuse configurado para todas as chamadas LLM com: `traceId`, `sessionId`, `userId` (hasheado), `usage` (tokens + custo), `latência`.
- [ ] 💡 Alertas do D25 configurados para os 9 cenários documentados (api_error_rate, latency_p95, etc.).
- [ ] 💡 Smoke tests passando pós-deploy antes de considerar release bem-sucedido.

---

## 6. Itens Bloqueantes

Tabela consolidada dos itens que impedem merge ou deploy quando não atendidos:

| # | Categoria | Item | Por que bloqueia | Como verificar | Impacto se ignorado |
|---|---|---|---|---|---|
| B-01 | CI | Qualquer check do CI falhando | Garante que código não regride funcionalidade existente | GitHub Actions — todos verdes | Regressão entra em produção |
| B-02 | Testes | Lógica crítica sem cobertura de teste | Sem testes, bugs silenciosos em cálculos financeiros | `pnpm test:coverage` ≥ 80%; 100% em CommissionService | Erro financeiro em produção |
| B-03 | Segurança | Credencial ou secret no código-fonte | Exposição de chave compromete toda a plataforma | `git log` + `grep` por padrões de token; GitGuardian | Comprometimento de segurança |
| B-04 | Segurança | Dado do Cedente em response do Cessionário | Viola LGPD e quebra o modelo de negócio | Testes de contrato; revisão manual da response | Violação regulatória + dano reputacional |
| B-05 | Segurança | Input de usuário sem validação via DTO | Permite injeção de dados maliciosos | Code review — confirmar `@IsString()`, `@IsUUID()` etc. | Injeção de dados, corrupção de estado |
| B-06 | Logs | Dado sensível (CPF, token) em log | Violação LGPD e exposição de credenciais | Revisar Pino `redact` config; testar em staging | Vazamento de dados pessoais |
| B-07 | Banco | Migration sem retrocompatibilidade | Pode derrubar a API durante rolling deploy | Code review da migration + teste em staging | Downtime durante deploy |
| B-08 | Banco | Migration sem rollback documentado | Impossibilita reversão em caso de erro | Comentário de rollback no arquivo SQL | Perda de dados irrecuperável |
| B-09 | Auth | Refresh token em localStorage | Exposto a XSS; compromete todas as sessões | Code review; verificar armazenamento do token | Sequestro de sessão massivo |
| B-10 | Auth | Rate limit ausente em endpoint de login | Permite brute force ilimitado | `@UseGuards(ThrottlerGuard)` presente; testar 6ª tentativa | Comprometimento de contas |
| B-11 | API | Endpoint sem guard de autenticação | Acesso não autenticado a dados privados | Code review; verificar `@UseGuards(JwtAuthGuard)` | Vazamento de dados de todos os usuários |
| B-12 | E2E | Qualquer teste E2E P0 falhando | Fluxo crítico de negócio comprometido | GitHub Actions E2E job verde | Bug em formalização, Escrow ou KYC em produção |
| B-13 | Acessibilidade | Keyboard trap em componente | Usuários de teclado incapazes de sair | Teste manual com Tab; axe-core | Violação WCAG, inacessibilidade total |
| B-14 | Performance | Bundle JS gzipped > 300KB | SLA de carregamento comprometido | `rollup-plugin-visualizer` + `size-limit` | Performance degradada em conexões lentas |
| B-15 | Observabilidade | Sentry não configurado no ambiente | Erros de produção silenciosos | Teste de Sentry: trigger manual de erro | Incidentes sem rastreabilidade |

---

## 7. Ferramentas de Apoio

| Ferramenta | Categoria | O que valida | Quando usar | Limitação |
|---|---|---|---|---|
| axe-core (Playwright) | Acessibilidade | Violações WCAG automatizadas | Todo E2E; auditoria quinzenal | Não detecta erros de fluxo de screen reader |
| Lighthouse | Performance / A11y | Score de performance, a11y, SEO | Pré-release; auditoria quinzenal | Score varia por ambiente e plugins de browser |
| NVDA (Windows) | Acessibilidade | Screen reader flow real | Auditoria manual pré-release | Requer ambiente Windows |
| VoiceOver (macOS/iOS) | Acessibilidade | Screen reader flow real em macOS/iOS | Auditoria manual pré-release; mobile | Comportamento pode variar por versão |
| `pnpm audit` | Segurança | Vulnerabilidades em dependências | Todo PR; CI automático | Não detecta vulnerabilidades 0-day |
| commitlint | Contribuição | Conventional Commits | Todo commit (pre-commit hook) | Não valida conteúdo semântico |
| Vitest coverage | Testes | Cobertura de código | Todo PR com mudança de lógica | Cobertura de linhas ≠ cobertura de comportamento |
| Sentry | Observabilidade | Erros em produção com stack trace | Pós-deploy; monitoramento contínuo | Dados de produção — não usar em dev |
| Langfuse | IA | Tokens, custo, latência de LLM | Monitoramento contínuo de IA | Requer instrumentação manual nas chamadas LLM |
| PostHog | Analytics | Eventos de produto | Pós-release; análise de funil | Não rastreia erros técnicos |
| Colour Contrast Analyser | Acessibilidade | Ratio de contraste exato | Design review; auditoria | Manual; não integrado ao CI |
| `rollup-plugin-visualizer` | Performance | Bundle size por dependência | Pré-release; quando bundle > 200KB | Apenas frontend Vite/Rollup |

---

## 8. Mapeamento por Categoria

| Checklist interno | Padrão externo | Referência |
|---|---|---|
| B-01 a B-15 (bloqueantes) | — | D23 (Guia de Contribuição) |
| Auth (seção 4.1) | OWASP A01, A07 | D18 (Fluxos de Autenticação) |
| Isolamento de dados (seção 4.2) | OWASP A01; LGPD Art. 46 | RN-013, RN-014, RN-068 |
| Validação de input (seção 4.3) | OWASP A03 | D20 (Error Handling), D19 (IA) |
| Secrets (seção 4.4) | OWASP A02 | D22 (Setup e Secrets) |
| Headers (seção 4.5) | OWASP A05; HSTS | D17 (Integrações) |
| Logs de dados sensíveis (seção 4.6) | LGPD Art. 47; OWASP A09 | D25 (Observabilidade) |
| Acessibilidade — contraste | WCAG 2.1 AA 1.4.3, 1.4.11 | D02 (Stacks — a11y) |
| Acessibilidade — teclado | WCAG 2.1 AA 2.1.1, 2.1.2 | D02 (Stacks — a11y) |
| Acessibilidade — ARIA | WCAG 2.1 AA 4.1.2 | D02 (Stacks — a11y) |
| Touch targets 44x44px | WCAG 2.5.5 | RN-019 (mobile) |
| Schema de erro | — | D20 (Error Handling) |
| Correlation ID | — | D25 (Observabilidade) |
| Testes de comissão (100% cobertura) | — | D27 (Plano de Testes) |

---

## 9. Periodicidade de Auditoria

### 9.1 Em Todo PR

Partes obrigatórias pelo revisor:

- [ ] Parte 1 — Code Review completa.
- [ ] Parte 3 — Segurança (itens B-03 a B-11).
- [ ] Parte 4 — Observabilidade (logs, correlation ID, Sentry).
- [ ] Tabela de bloqueantes — verificar se algum item B-01 a B-15 se aplica.

### 9.2 Auditoria Quinzenal

- [ ] Parte 2 — Acessibilidade completa (axe-core em todos os fluxos críticos).
- [ ] `pnpm audit --audit-level=high` — nenhuma vulnerabilidade high/critical.
- [ ] Lighthouse score em produção ≥ 90 (performance) e ≥ 95 (acessibilidade).
- [ ] Verificar se há secrets novos não documentados no `.env.example`.
- [ ] Revisar alertas do Sentry: nenhum erro recorrente sem issue criada.

### 9.3 Pré-Release (develop → main)

- [ ] Todas as partes (1, 2, 3, 4) revisadas.
- [ ] Tabela de bloqueantes: todos os itens verificados e marcados.
- [ ] E2E Playwright P0 passando no staging.
- [ ] Smoke tests de produção prontos.
- [ ] CHANGELOG.md atualizado via `release-please`.
- [ ] Release notes redigidas (template do D24).
- [ ] Migrations com rollback documentado.
- [ ] Sentry configurado na nova versão.
- [ ] Langfuse com alertas de custo configurados.

### 9.4 Pós-Incidente P0/P1

- [ ] Causa raiz identificada e documentada.
- [ ] Item de checklist atualizado se o incidente revelou gap.
- [ ] Teste adicionado para cobrir o cenário do incidente.
- [ ] Post-mortem redigido (template do D26).
- [ ] Alerta novo configurado se o incidente não foi detectado antes.

---

## 10. Backlog de Pendências

| ID | Item | Tipo | Prioridade |
|---|---|---|---|
| QA-001 | Configurar axe-core integrado ao Playwright para executar em todos os fluxos E2E críticos | Ação técnica | Alta |
| QA-002 | Adicionar `size-limit` ao CI para bloquear bundle regressões > 5% | Ação técnica | Média |
| QA-003 | Realizar auditoria inicial de acessibilidade com VoiceOver nos fluxos críticos (login, KYC, proposta) | Ação técnica | Alta |
| QA-004 | Documentar Brand Theme Guide (D03) quando disponível — itens de contraste e design tokens dependem deste documento | [DADO PENDENTE] — D03 não disponível nesta pipeline | Alta |

---

*Documento gerado pelo pipeline ShiftLabs v9.5 — Fase 2. Executor: Claude Code Desktop.*
