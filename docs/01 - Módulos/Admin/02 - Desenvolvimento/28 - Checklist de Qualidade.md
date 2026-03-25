# 28 — Checklist de Qualidade

## Metadados

| Campo       | Valor                                      |
|-------------|--------------------------------------------|
| **Nome**    | Checklist de Qualidade — Gate de Engenharia |
| **Versão**  | 1.0.0                                      |
| **Data**    | 2025-01-01                                 |
| **Autor**   | Frontend Lead / Backend Lead / Security Engineer |
| **Status**  | Ativo                                      |
| **Módulo**  | Admin — Repasse Seguro                     |

---

📌 **TL;DR**

Este documento é o gate de qualidade obrigatório do Repasse Seguro. Ele deve ser aplicado em cada Pull Request (checklist rápido), antes de releases (auditoria completa) e após incidentes críticos (revisão de segurança e observabilidade). Cobre **Code Review** (frontend, backend, API, dados, performance), **Acessibilidade WCAG 2.1 AA**, **Segurança** contextualizada ao produto (autenticação, autorização, dados sensíveis, dependências) e **Observabilidade/Operação** (logs, métricas, alertas, health checks). Itens marcados com 🔴 **BLOQUEIA PR** e não podem ser ignorados. Demais itens são recomendações com impacto na qualidade do produto.

---

## 3. Como usar o checklist

### 3.1 Quando aplicar

| Contexto              | Quem aplica                        | Cobertura mínima                                                 |
|-----------------------|------------------------------------|------------------------------------------------------------------|
| **PR para `develop`** | Autor + Revisor                    | Parte 1 completa + itens críticos de Segurança                   |
| **PR para `main`**    | Autor + Tech Lead                  | Partes 1–4 + Tabela de Bloqueantes                              |
| **Auditoria quinzenal** | Security Engineer + Backend Lead | Partes 2, 3 e 4 com evidências documentadas                     |
| **Pré-release**       | Tech Lead + QA                     | Documento completo; todos os bloqueantes verificados             |
| **Pós-incidente crítico** | Engenheiro on-call + Tech Lead | Parte 3 (Segurança) + Parte 4 (Observabilidade) — revisão imediata |

### 3.2 Como registrar evidência

- **PR**: adicionar comentário ou seção no template do PR com `[x]` para cada item coberto.
- **Auditoria**: criar issue no repositório com label `quality-audit` e link para a revisão.
- **Pré-release**: incluir no Release Notes (D24) a seção "Checklist de Qualidade — aprovado em DD/MM/AAAA".
- **Pós-incidente**: incluir no post-mortem (D26) a seção "Verificações de qualidade executadas".

### 3.3 Semântica dos itens

- 🔴 **BLOQUEIA PR**: impede merge até resolução. Evidência obrigatória.
- 💡 **RECOMENDAÇÃO**: melhoria desejável. Não bloqueia PR, mas deve ser registrado como issue.
- `[ ]`: item ainda não verificado.
- `[x]`: item verificado e aprovado.
- `[N/A]`: item não aplicável a este contexto (justificar no comentário).

---

## 4. Parte 1 — Code Review

### 4.1 Frontend (React / TypeScript / TanStack)

- [ ] 🔴 Nenhum `any` implícito ou explícito sem justificativa documentada em comentário inline (`// justificativa: ...`)
- [ ] 🔴 Formulários validados com schema Zod no cliente antes de envio à API
- [ ] 🔴 Erros de API capturados e exibidos ao usuário via `GlobalErrorBoundary` ou error boundary de rota (TanStack Router `errorComponent`)
- [ ] 🔴 Dados sensíveis (CPF, PIX, token) nunca exibidos em console.log ou em atributos DOM inspecionáveis
- [ ] 💡 Componentes com responsabilidade única — sem componentes com mais de 200 linhas sem justificativa
- [ ] 💡 Props opcionais com valor padrão definido; props obrigatórias sem valor default
- [ ] 💡 Imagens com `alt` descritivo ou `alt=""` se decorativa; nunca `alt` ausente
- [ ] 💡 Loading states implementados para todas as requisições assíncronas (skeleton ou spinner)
- [ ] 💡 Sem uso de `useEffect` para derivar estado — usar `useMemo` ou seletores do store
- [ ] 💡 Eventos de analytics (PostHog) disparados nos fluxos críticos: login, criação de caso, fechamento, distribuição escrow

**Ferramenta de verificação:** ESLint (`@typescript-eslint/no-explicit-any`), TypeScript strict mode, React DevTools

---

### 4.2 Backend (NestJS / TypeScript)

- [ ] 🔴 Nenhum endpoint sem decorador de autorização (`@Roles`, `@UseGuards`) — proibido endpoint anônimo em produção
- [ ] 🔴 Toda escrita no banco dentro de transação explícita com rollback em caso de erro (`prisma.$transaction`)
- [ ] 🔴 Nenhuma query sem `where` clause em operações de atualização ou exclusão (risco de full-table update)
- [ ] 🔴 Inputs validados com class-validator / Zod no DTO antes de chegar ao service
- [ ] 🔴 Segredos nunca hardcoded — todas as credenciais via variável de ambiente com fallback explícito que lança erro
- [ ] 💡 Services sem dependência direta de infraestrutura — repositórios injetados via interface
- [ ] 💡 Logs de audit emitidos para toda ação que altera estado de caso, escrow ou usuário
- [ ] 💡 Correlation ID propagado em todos os logs (`logger.info({ correlationId, ... })`)
- [ ] 💡 Sem `console.log` — usar exclusivamente o logger Pino injetado
- [ ] 💡 Handlers de exceção não expõem stack trace ao cliente (validar `GlobalExceptionFilter`)

**Ferramenta de verificação:** ESLint, NestJS `ValidationPipe` global, Sentry para erros não tratados

---

### 4.3 API (Contratos e Documentação)

- [ ] 🔴 Nenhuma mudança de contrato de API sem versionamento (`/v1/`, `/v2/`) ou deprecation notice no header
- [ ] 🔴 Schemas de request e response documentados com Zod e validados contra o contrato (D16 — API Contracts)
- [ ] 🔴 Respostas de erro seguem RFC 7807 com campos obrigatórios: `type`, `title`, `status`, `detail`, `instance`, `correlation_id`, `timestamp`
- [ ] 💡 Rate limiting configurado por rota sensível (auth, submissions, AI)
- [ ] 💡 Paginação implementada em todos os endpoints de listagem (evitar full-table scans)
- [ ] 💡 Headers de segurança presentes: `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`

**Ferramenta de verificação:** Postman / Thunder Client contra contrato D16; Zod parse manual

---

### 4.4 Dados e Banco de Dados

- [ ] 🔴 Migrações sempre aditivas — proibido `DROP COLUMN` em coluna em uso sem ciclo de deprecação documentado
- [ ] 🔴 Nenhuma migration que altere dados em produção sem script de rollback (ou justificativa de irreversibilidade aceita pelo Tech Lead)
- [ ] 🔴 Campos CPF, PIX, dados financeiros armazenados com criptografia de aplicação ou via extensão `pgsodium` do Supabase
- [ ] 💡 Índices criados para colunas usadas em `WHERE`, `ORDER BY` ou `JOIN` em queries de produção
- [ ] 💡 Queries N+1 identificadas e eliminadas (usar `include` do Prisma ou DataLoader)
- [ ] 💡 Seeds de teste usam dados sintéticos — nenhum CPF ou dado real

**Ferramenta de verificação:** `EXPLAIN ANALYZE` no Supabase; Prisma Studio para inspecionar schema; `prisma validate`

---

### 4.5 Performance

- [ ] 🔴 Nenhum bundle de página acima de 500 KB gzipped sem justificativa (code splitting obrigatório)
- [ ] 💡 Imagens otimizadas (WebP/AVIF, lazy loading, dimensões explícitas)
- [ ] 💡 Requisições em paralelo quando sem dependência (`Promise.all`) — sem waterfall desnecessário
- [ ] 💡 Cache de Redis utilizado para dados de configuração (snapshots) e listas frequentes
- [ ] 💡 Web Vitals monitorados: LCP < 2.5s, FID < 100ms, CLS < 0.1

**Ferramenta de verificação:** Lighthouse CI, Webpack Bundle Analyzer, Railway Metrics

---

## 5. Parte 2 — Acessibilidade (WCAG 2.1 AA)

### 5.1 Perceptível

- [ ] 🔴 Contraste de texto normal: ratio mínimo 4.5:1 (WCAG AA) — validar em modo claro e modo escuro
  - [DECISÃO AUTÔNOMA]: bloquear PR se ratio < 4.5:1 — Justificativa: padrão legal mínimo e verificável via axe-core | Alternativa descartada: ratio < 7:1 (WCAG AAA) — restritivo demais para MVP de design system
- [ ] 🔴 Contraste de texto grande (≥18pt/14pt bold) e elementos UI: ratio mínimo 3:1
- [ ] 🔴 Todas as imagens informativas com `alt` descritivo; imagens decorativas com `alt=""`
- [ ] 💡 Vídeos com legenda ou transcrição (quando aplicável)
- [ ] 💡 Cores nunca usadas como único meio de transmitir informação (ex: ícone ou texto de acompanhamento)

**Ferramenta:** axe-core, Lighthouse Accessibility, Colour Contrast Analyser

---

### 5.2 Operável

- [ ] 🔴 Todas as funcionalidades acessíveis via teclado — nenhuma ação exige mouse
- [ ] 🔴 Sem keyboard traps — usuário sempre consegue sair de qualquer componente com Tab ou Escape
- [ ] 🔴 Focus trap em modais e drawers — foco deve circular internamente até fechar
- [ ] 🔴 Focus restoration ao fechar modal/drawer — foco retorna ao elemento que acionou a abertura
- [ ] 🔴 Visible focus ring — todos os elementos interativos com indicador de foco visível (não remover `outline` sem substituto)
- [ ] 🔴 Touch targets mobile: mínimo 44×44px (WCAG 2.5.5)
- [ ] 💡 Tab order lógico — segue a ordem visual esperada do layout
- [ ] 💡 Atalhos de teclado documentados quando existirem

**Ferramenta:** Teclado manual (Tab, Shift+Tab, Enter, Space, Escape, Arrow keys), axe-core

---

### 5.3 Compreensível

- [ ] 🔴 Formulários com `<label>` associado a cada campo via `for`/`id` ou `aria-labelledby`
- [ ] 🔴 Mensagens de erro de formulário associadas ao campo via `aria-describedby`
- [ ] 💡 Idioma do documento declarado (`lang="pt-BR"` no `<html>`)
- [ ] 💡 Inputs com `autocomplete` configurado onde aplicável (nome, e-mail, CPF)
- [ ] 💡 Textos de botão descritivos — evitar "Clique aqui", "Saiba mais" sem contexto

**Ferramenta:** NVDA (Windows) / VoiceOver (macOS/iOS) para validação de screen reader

---

### 5.4 Robusto

- [ ] 🔴 HTML semanticamente correto — sem `<div>` onde deveria ser `<button>`, `<a>`, `<nav>`, etc.
- [ ] 🔴 ARIA roles de landmark presentes: `<main>`, `<nav>`, `<aside>`, `<header>`, `<footer>`
- [ ] 🔴 Skip link "Pular para conteúdo principal" como primeiro elemento focável — visível ao receber foco via teclado
- [ ] 🔴 Live regions com `aria-live` para anúncios de status (loading, error, success) — usando `aria-live="polite"` para não-urgentes e `aria-live="assertive"` para erros críticos
- [ ] 🔴 Estados dinâmicos anunciados: loading (`aria-busy="true"`), disabled (`aria-disabled`), expanded (`aria-expanded`), selected (`aria-selected`)
- [ ] 🔴 Tabelas com `<thead>`, `<th scope="col/row">` — sem tabelas de layout
- [ ] 🔴 `prefers-reduced-motion: reduce` — todas as animações não essenciais desabilitadas quando usuário prefere menos movimento
- [ ] 💡 `aria-label` em ícones sem texto visível (`<button aria-label="Fechar modal">`)
- [ ] 💡 Componentes personalizados (dropdown, combobox, date picker) seguem padrões WAI-ARIA Authoring Practices

**Ferramenta:** axe-core (integrado ao Playwright em E2E), NVDA, VoiceOver, Lighthouse Accessibility

---

## 6. Parte 3 — Segurança

### 6.1 Autenticação e Autorização

- [ ] 🔴 JWT validado em todo endpoint protegido — sem bypass de assinatura
- [ ] 🔴 Refresh token com rotação — token antigo invalidado após uso (previne reuse)
- [ ] 🔴 2FA habilitado para roles Master e Coordenador (obrigatório conforme D14)
- [ ] 🔴 Sessão expirada tratada adequadamente — usuário redirecionado ao login sem exposição de dados
- [ ] 🔴 RBAC aplicado no backend — frontend apenas oculta UI, nunca é a única barreira de autorização
- [ ] 🔴 Contas bloqueadas após N tentativas de login — `AccountLockedError` (AUTH_003) com rate limit por IP

**Ferramenta:** Testes de integração de auth (D27); revisão manual do guard NestJS

---

### 6.2 Validação de Input e Injeção

- [ ] 🔴 Todos os inputs do usuário validados no backend com class-validator / Zod — nunca confiar apenas no frontend
- [ ] 🔴 Queries Prisma com parâmetros tipados — sem concatenação de string em SQL
- [ ] 🔴 Upload de arquivos: tipo MIME verificado no backend, tamanho máximo definido, path traversal prevenido
- [ ] 🔴 Saída HTML renderizada com escape correto — sem `dangerouslySetInnerHTML` sem sanitização
- [ ] 💡 Campos de busca com limite de comprimento e caracteres especiais tratados

**Ferramenta:** ESLint (`no-sql-injection` plugin), análise manual de DTOs

---

### 6.3 Dados Sensíveis

- [ ] 🔴 CPF, chave PIX, dados bancários nunca aparecem em logs (verificar `pino.redact` ativo)
- [ ] 🔴 Tokens JWT nunca logados ou expostos em resposta de erro
- [ ] 🔴 Senhas nunca armazenadas em texto plano — hash bcrypt (custo mínimo 12)
- [ ] 🔴 Secrets de serviços externos (ZapSign, Resend, Twilio, Meta Cloud API) em variáveis de ambiente — nunca em código
- [ ] 💡 Dados de PII mascarados em logs de debug: CPF `***.***.***-**`, token truncado

**Ferramenta:** Revisão do `pino.redact` em D25; grep por `console.log` com campos sensíveis

---

### 6.4 Transporte e Headers

- [ ] 🔴 HTTPS obrigatório em todos os ambientes exceto localhost
- [ ] 🔴 Headers de segurança configurados: `Strict-Transport-Security`, `Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`
- [ ] 💡 CORS configurado com lista explícita de origens permitidas — sem `*` em produção
- [ ] 💡 Cookie de sessão com flags `HttpOnly`, `Secure`, `SameSite=Strict`

**Ferramenta:** securityheaders.com, Helmet.js (NestJS middleware)

---

### 6.5 Dependências e Supply Chain

- [ ] 🔴 `pnpm audit --audit-level=high` sem vulnerabilidades High/Critical sem mitigação documentada
- [ ] 💡 Dependências desatualizadas revisadas mensalmente (`pnpm outdated`)
- [ ] 💡 Dependências de desenvolvimento não incluídas no bundle de produção

**Ferramenta:** `pnpm audit`, GitHub Dependabot alerts

---

### 6.6 Rate Limiting e Auditoria

- [ ] 🔴 Rate limiting em endpoints de auth (login, reset password, 2FA): máximo 10 req/min por IP
- [ ] 🔴 Toda ação financeira (escrow, distribuição) registrada na tabela de auditoria com usuário, timestamp e correlation_id
- [ ] 🔴 Toda decisão de agente de IA registrada em `ai_decisions` com output completo e confidence score
- [ ] 💡 Alertas de brute force configurados no Sentry/Railway (≥10 falhas de login em 60s)

**Ferramenta:** Teste manual de rate limit; revisão de triggers de auditoria no Prisma schema

---

## 7. Parte 4 — Observabilidade e Operação

### 7.1 Logs

- [ ] 🔴 Correlation ID presente em todos os logs da requisição (`correlationId` no contexto Pino)
- [ ] 🔴 Logs estruturados em JSON — sem `console.log` em código de produção
- [ ] 🔴 `pino.redact` ativo com paths: `authorization`, `password`, `token`, `secret`, `cpf`, `pix_key`, `credit_card`, `api_key`
- [ ] 💡 Log level correto: `debug` apenas em desenvolvimento, `info` para fluxos normais, `warn` para degradação, `error` para falhas

**Ferramenta:** Revisão do logger Pino (`D25`); Railway Logs em staging

---

### 7.2 Métricas e SLOs

- [ ] 🔴 Endpoint `/health` retornando 200 com status de dependências (DB, Redis, RabbitMQ)
- [ ] 🔴 SLOs configurados e monitorados: API availability ≥99.5%, p95 latency ≤500ms, DLQ rate ≤1%, email delivery ≥95% em 5min
- [ ] 💡 Métricas customizadas de negócio instrumentadas: casos criados/dia, taxa de fechamento, falhas de agente IA

**Ferramenta:** Railway Metrics, Upstash console, dashboard "API Health" (D25)

---

### 7.3 Alertas e On-call

- [ ] 🔴 Alertas configurados para os 10 cenários catalogados no D26 (Runbook Operacional)
- [ ] 🔴 DLQ com mensagens notifica Slack `#rs-alertas` em menos de 5 minutos após threshold
- [ ] 💡 Sentry com fingerprinting correto — sem grupos de erros excessivamente agregados

**Ferramenta:** CloudAMQP Management UI; Sentry Issues dashboard

---

### 7.4 Pós-deploy

- [ ] 🔴 Health check pós-deploy executado antes de considerar release estável (Railway health check endpoint)
- [ ] 🔴 Monitoramento ativo nas primeiras 2h após deploy de feature em produção
- [ ] 💡 Rollback testado em staging antes do deploy de releases com migrations

**Ferramenta:** Railway deploy dashboard; `railway logs --tail` após deploy

---

## 8. Itens Bloqueantes

Tabela consolidada dos itens que impedem merge. Todos exigem evidência de resolução antes do approve.

| # | Categoria | Item | Por que bloqueia | Como verificar | Impacto se ignorado |
|---|-----------|------|-----------------|----------------|---------------------|
| B01 | TypeScript | `any` sem justificativa documentada | Erros de runtime silenciosos em fluxos críticos | ESLint `no-explicit-any` zero warnings | Bugs difíceis de rastrear em produção |
| B02 | Backend | Endpoint sem guard de autorização | Acesso não autorizado a dados de casos e escrow | Revisar todos os controllers no diff do PR | Vazamento de dados de clientes (LGPD) |
| B03 | Backend | Escrita no banco sem transação | Dados corrompidos em falha parcial | Code review manual + test de integração | Inconsistência de estado de caso/escrow |
| B04 | API | Resposta de erro sem RFC 7807 | Clientes não conseguem tratar erros programaticamente | Teste manual dos endpoints alterados | Degradação silenciosa na UI; erros ocultos |
| B05 | Dados | `DROP COLUMN` sem ciclo de deprecação | Quebra de aplicação ao fazer rollback | Revisar migration files no PR | Downtime imediato em caso de rollback |
| B06 | Segurança | Secrets hardcoded | Exposição de credenciais no repositório | `git diff` + grep por padrões de token | Comprometimento de serviços externos |
| B07 | Segurança | Hash de senha < bcrypt custo 12 | Senhas quebráveis por força bruta | Revisar AuthService hashing config | Comprometimento de contas de usuários |
| B08 | Segurança | Dados sensíveis em logs (CPF, token) | Violação de LGPD e exposição em Railway Logs | Revisar `pino.redact` + testar com dado de CPF | Multa ANPD + incidente de segurança |
| B09 | Acessibilidade | Contraste < 4.5:1 em texto normal | Inacessível para usuários com baixa visão (WCAG AA) | axe-core no PR ou Lighthouse CI | Exclusão de usuários + risco de auditoria legal |
| B10 | Acessibilidade | Keyboard trap sem saída via Escape | Usuário de teclado preso no componente | Teste manual: Tab através do fluxo | Inacessibilidade completa para usuários sem mouse |
| B11 | Acessibilidade | Skip link ausente ou não funcional | Screen reader users sem atalho para conteúdo | Testar com VoiceOver/NVDA: primeiro Tab | Navegação degradada para leitores de tela |
| B12 | Acessibilidade | `prefers-reduced-motion` ignorado | Gatilho de convulsão/desorientação para usuários afetados | CSS/JS revisado para `@media (prefers-reduced-motion)` | Risco de saúde + violação WCAG 2.3.3 |
| B13 | Observabilidade | Correlation ID ausente nos logs | Rastreabilidade impossível em incidentes | Grep em logs de staging após deploy | Triagem de incidentes levando horas extras |
| B14 | Observabilidade | Health check retornando 200 com DB offline | Deploy considerado saudável com infraestrutura quebrada | Simular queda de DB e verificar `/health` | Alertas não disparados durante incidente real |
| B15 | Performance | Bundle de página > 500 KB gzipped | Tempo de carregamento inaceitável em mobile | Webpack Bundle Analyzer no CI | Abandono de usuário + SLO de LCP violado |

---

## 9. Ferramentas de Apoio

| Ferramenta | Categoria | O que valida | Quando usar | Limitação |
|------------|-----------|-------------|-------------|-----------|
| **ESLint** (`@typescript-eslint`) | TypeScript / Code Quality | `any` implícito, imports proibidos, regras de estilo | Todo PR — executado no CI | Não captura erros de lógica de negócio |
| **TypeScript** (`tsc --noEmit`) | Type Safety | Compatibilidade de tipos, interfaces, contratos | Todo PR — CI typecheck job | Não detecta erros de runtime de dados externos |
| **Vitest + Supertest** | Testes unitários e integração | Comportamento de services, handlers, repositories | Todo PR + pre-commit | Não substitui testes E2E para fluxos críticos |
| **Playwright** | E2E | 5 fluxos críticos de negócio | PR para `main` + pré-release | Lento (~20min); não cobre edge cases raros |
| **axe-core** | Acessibilidade | WCAG 2.1 AA automatizável (~30% dos critérios) | Integrado ao Playwright; rodar em PR | Não substitui teste manual com screen reader |
| **Lighthouse CI** | Performance + Acessibilidade | Web Vitals, bundle size, acessibilidade básica | Todo PR para `main` | Resultados variam por ambiente de CI |
| **NVDA / VoiceOver** | Screen reader | Fluxos de navegação por teclado e leitura | Auditoria quinzenal + pré-release | Requer tester treinado; não é automatizável |
| **Colour Contrast Analyser** | Acessibilidade | Ratio de contraste de cores (texto e UI) | Design review + acessibilidade PR | Manual; não cobre estados hover/focus dinamicamente |
| **pnpm audit** | Segurança de dependências | CVEs em dependências npm | Todo PR — CI security job | Atraso entre CVE publicado e entrada no banco |
| **securityheaders.com** | Segurança de transporte | Headers HTTP de segurança | Auditoria quinzenal + pré-release | Apenas verifica headers, não conteúdo da resposta |
| **Sentry** | Erros / Observabilidade | Erros de produção com stack trace e breadcrumbs | Monitoramento contínuo + pós-deploy | Ruído sem fingerprinting bem configurado |
| **PostHog** | Analytics / Observabilidade | Funis de usuário, eventos de negócio | Auditoria de produto mensal | Depende de instrumentação manual correta |
| **Railway Metrics** | Infra / Observabilidade | CPU, memória, latência do API server | Monitoramento contínuo | Granularidade limitada vs. Prometheus/Grafana |
| **CloudAMQP UI** | Filas / Observabilidade | Profundidade de fila, DLQ, mensagens mortas | Após deploy + auditoria operacional | Interface manual; sem alertas automáticos nativos |

---

## 10. Mapeamento por Categoria

### 10.1 Segurança × OWASP Top 10

| OWASP 2021 | Item no checklist | Seção |
|------------|-------------------|-------|
| A01 — Broken Access Control | B02 (endpoint sem guard) | 6.1 |
| A02 — Cryptographic Failures | B07 (hash fraco), B08 (dados em logs) | 6.3 |
| A03 — Injection | SQL injection via Prisma tipado, upload path traversal | 6.2 |
| A05 — Security Misconfiguration | Headers de segurança, CORS com `*` | 6.4 |
| A06 — Vulnerable Components | `pnpm audit` obrigatório | 6.5 |
| A07 — Auth Failures | 2FA obrigatório, session expiry, rate limit | 6.1, 6.6 |
| A09 — Security Logging Failures | Correlation ID, auditoria de ações financeiras | 7.1, 6.6 |

### 10.2 Acessibilidade × WCAG 2.1 AA

| Critério WCAG | Nível | Item no checklist | Seção |
|--------------|-------|-------------------|-------|
| 1.4.3 Contrast (Minimum) | AA | Contraste 4.5:1 texto | 5.1 |
| 1.4.11 Non-text Contrast | AA | Contraste 3:1 UI elements | 5.1 |
| 2.1.1 Keyboard | A | Todas funcionalidades via teclado | 5.2 |
| 2.1.2 No Keyboard Trap | A | Sem keyboard trap | 5.2 |
| 2.4.3 Focus Order | A | Tab order lógico | 5.2 |
| 2.4.7 Focus Visible | AA | Visible focus ring | 5.2 |
| 2.5.5 Target Size | AAA | Touch targets 44×44px | 5.2 |
| 1.3.1 Info and Relationships | A | HTML semântico, landmarks | 5.4 |
| 4.1.2 Name, Role, Value | A | ARIA roles e estados | 5.4 |
| 2.4.1 Bypass Blocks | A | Skip link | 5.4 |
| 3.3.2 Labels or Instructions | A | Labels associados a campos | 5.3 |
| 2.3.3 Animation from Interactions | AAA | `prefers-reduced-motion` | 5.4 |

### 10.3 Code Review × Design System (D03 Brand / D09 UI Contracts)

| Critério de Design | Verificação | Item relacionado |
|-------------------|-------------|-----------------|
| Tokens de cor do Brand | Sem cores hardcoded (`#hex`) fora de `tokens.ts` | 4.1 (frontend) |
| Tipografia padronizada | Sem `font-size` inline não mapeado ao design system | 4.1 (frontend) |
| Componentes reutilizáveis | Novos componentes verificados contra D09 UI Contracts | 4.1 (frontend) |

### 10.4 Observabilidade × SLOs Internos (D25)

| SLO | Threshold | Alerta associado | Item checklist |
|-----|-----------|-----------------|----------------|
| API availability | ≥99.5% | Error rate alert (Railway) | 7.2 |
| p95 latency | ≤500ms | Latency alert (Railway) | 7.2 |
| Email delivery | ≥95% em 5min | DLQ alert (CloudAMQP) | 7.2 |
| DLQ rate | ≤1% | DLQ threshold alert | 7.3 |

---

## 11. Periodicidade de Auditoria

### 11.1 Em cada PR

**Quem:** Autor (auto-revisão) + Revisor designado

**Foco:**
- Parte 1 completa (Code Review)
- Itens 🔴 das Partes 2, 3 e 4
- Tabela de bloqueantes: todos os B01–B15 verificados para o escopo do PR

**Evidência:** Comentário no PR com `[x]` nos itens cobertos.

---

### 11.2 Semanal (Segunda-feira)

**Quem:** Backend Lead

**Foco:**
- Revisar DLQ no CloudAMQP — há mensagens não processadas?
- Checar Sentry Issues novas na semana
- Revisar `pnpm audit` — alguma nova vulnerabilidade High/Critical?
- Checar SLOs da semana anterior no Railway Metrics

**Evidência:** Registro no canal `#rs-operacional` no Slack.

---

### 11.3 Quinzenal (Auditoria de Qualidade)

**Quem:** Security Engineer + Frontend Lead

**Foco:**
- Parte 2 (Acessibilidade) — auditoria com axe-core + VoiceOver nos fluxos críticos
- Parte 3 (Segurança) — revisar headers, rate limits, dependências
- Contraste em modo escuro e claro nos componentes novos do período
- Touch targets em mobile dos últimos componentes entregues

**Evidência:** Issue criada com label `quality-audit` + link ao relatório Lighthouse.

---

### 11.4 Pré-release

**Quem:** Tech Lead + QA

**Foco:** Documento completo. Nenhum item 🔴 em aberto. E2E fluxos críticos passando (D27). CHANGELOG atualizado (D24).

**Evidência:** Seção "Checklist de Qualidade aprovado" no Release Notes.

---

### 11.5 Pós-incidente Crítico

**Quem:** Engenheiro on-call + Tech Lead

**Foco:**
- Parte 3 (Segurança): verificar se o incidente expôs dado sensível
- Parte 4 (Observabilidade): o alerta disparou corretamente? O correlation ID estava presente?
- Item B13 (correlation ID) e B14 (health check) — falha nesses itens durante o incidente?

**Evidência:** Seção "Verificações de qualidade" no post-mortem (D26).

---

## 12. Backlog de Pendências

| # | Item | Categoria | Prioridade | Motivo |
|---|------|-----------|------------|--------|
| P01 | Integrar axe-core ao pipeline de Playwright (E2E) para auditoria automatizada de acessibilidade em CI | Acessibilidade | Alta | Atualmente apenas manual; aumenta cobertura sem custo de tempo |
| P02 | Configurar Lighthouse CI no workflow `ci.yml` para bloquear PR com score de acessibilidade < 90 | Performance / Acessibilidade | Alta | Garante regressão automática de Web Vitals e acessibilidade |
| P03 | Criar teste de integração de screen reader (NVDA/VoiceOver scripting) para Login 2FA e Criação de Caso | Acessibilidade | Média | Cobertura de screen reader hoje é exclusivamente manual |
| P04 | Documentar inventário de componentes com avaliação de conformidade WCAG por componente | Acessibilidade | Média | Facilita auditoria quinzenal com baseline conhecido |
| P05 | Configurar Dependabot ou `pnpm audit` automático com abertura de issue no GitHub para CVE High+ | Segurança | Alta | Atualmente manual; atraso de descoberta de vulnerabilidades |
| P06 | Revisar fingerprinting do Sentry para reduzir grupos duplicados de erros de integração externa | Observabilidade | Baixa | Melhora sinal/ruído no dashboard de erros |

---

*Documento gerado pelo pipeline ShiftLabs v9.5 — módulo Admin, Repasse Seguro.*
