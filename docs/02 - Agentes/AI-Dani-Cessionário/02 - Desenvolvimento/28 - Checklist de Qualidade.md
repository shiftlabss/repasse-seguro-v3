# 28 - Checklist de Qualidade
## AI-Dani-Cessionário — Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Produto** | AI-Dani-Cessionário |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Status** | Aprovado — pronto para execução |
| **Bloco** | 6 — Qualidade |

---

> 📌 **TL;DR — Gate de Qualidade**
>
> Este checklist é o gate obrigatório para PR, auditoria quinzenal e pré-release do AI-Dani-Cessionário. Cobre code review (frontend, backend, API, dados, performance), acessibilidade WCAG 2.1 AA (18+ itens), segurança contextualizada ao produto (isolamento, OTP, rate limit, secrets) e observabilidade operacional. Itens marcados com **[BLOQUEANTE]** impedem merge ou release imediatamente. Itens sem marcação são recomendações que geram label `needs-attention` no PR. Todo item tem resposta objetiva: sim, não, ou N/A.

---

## 1. Como Usar o Checklist

### 1.1 Quando Aplicar

| **Momento** | **Quem Preenche** | **Partes Obrigatórias** |
|---|---|---|
| **PR para `develop`** | Autor + Revisor | Parte 1 (Code Review) + Parte 3 (Segurança — itens P0) |
| **PR para `main`** | Tech Lead + QA | Todas as partes |
| **Auditoria quinzenal** | Tech Lead | Partes 2, 3 e 4 completas |
| **Pré-release (go-live)** | Tech Lead + DevOps | Todas as partes + Tabela de Bloqueantes |
| **Pós-incidente crítico** | Responsável pelo incidente | Partes 3 e 4 |

### 1.2 Como Registrar Evidência

- `[x]` — Item verificado com evidência.
- `[ ]` — Item ainda não verificado.
- `[N/A]` — Item não se aplica a este PR/release (justificar na descrição do PR).
- **[BLOQUEANTE]** — Se marcado `[ ]` em item bloqueante, merge ou release está impedido.

### 1.3 Evidências Aceitas

- Link para relatório de cobertura no CI.
- Screenshot do axe-core ou Lighthouse sem violações críticas.
- Saída de `pnpm audit` sem CVEs críticas.
- Output de `redis-cli GET dani:status:agent` em produção.
- Hash do artefato de build confirmado.

---

## 2. Parte 1 — Code Review

### 2.1 Frontend (React / TypeScript)

```
[ ] [BLOQUEANTE] Componente implementa todos os 4 estados obrigatórios: Skeleton → Empty → Error → Populated (spinners proibidos)
[ ] [BLOQUEANTE] Nenhum spinner (`<LoadingSpinner>` ou `animate-spin`) em componentes de dados assíncronos — apenas Skeleton
[ ] [BLOQUEANTE] Barrel exports obrigatórios em cada feature: `features/dani-chat/index.ts`, `features/dani-dashboard/index.ts`, `features/dani-whatsapp/index.ts`
[ ] [BLOQUEANTE] Nenhum componente importa diretamente de outro feature sem passar pelo barrel export
[ ] Props tipadas com TypeScript — sem `any` em componentes novos
[ ] Hooks customizados extraídos se a lógica de estado for usada em >1 componente
[ ] `useEffect` com cleanup function quando necessário (listeners, timers, subscriptions)
[ ] Sem `console.log` ou `console.error` em código de produção — apenas Pino no backend
[ ] Componentes de chat usam `role="log"` ou `aria-live="polite"` nas mensagens
[ ] Responsividade verificada nos breakpoints definidos (≤640px para mobile)
[ ] Tokens do Brand Theme Guide usados em vez de valores hardcoded (cores, fontes, espaçamentos)
[ ] `data-testid` adicionado em todos os elementos interagidos por testes (chat input, FAB, botões de ação)
```

### 2.2 Backend (NestJS / TypeScript)

```
[ ] [BLOQUEANTE] `CessionarioOwnerGuard` aplicado em TODOS os endpoints que retornam dados de negócio
[ ] [BLOQUEANTE] Toda query ao banco inclui WHERE `cessionario_id = :id` — sem query cross-tenant
[ ] [BLOQUEANTE] Nenhum dado sensível (phone, CPF, JWT, OTP, API key) presente em logs — usar Pino redact
[ ] [BLOQUEANTE] `CalculadoraModule` não importa de `AgenteModule` e vice-versa (verificar circular dependency)
[ ] DTOs de request e response validados com `class-validator` + `class-transformer`
[ ] Exceções lançam `DaniBaseException` (ou subclasse) — nunca `Error` nativo diretamente
[ ] `AllExceptionsFilter` não expõe stack trace em resposta de produção
[ ] Correlation ID propagado do middleware para todos os logs do fluxo
[ ] Rate limit verificado em endpoints de agente e OTP — `dani:rate:*` keys com TTL correto
[ ] Redis keys seguem nomenclatura: `dani:{namespace}:{identificador}` (sem UUID raw)
[ ] Migrations backward-compatible — sem DROP de coluna sem período de deprecação
[ ] Módulos NestJS registrados corretamente (sem import circular em `AppModule`)
```

### 2.3 API (Endpoints)

```
[ ] [BLOQUEANTE] Endpoint documentado no D16 — nenhum endpoint novo sem documentação
[ ] [BLOQUEANTE] Endpoint com dados de negócio tem `CessionarioOwnerGuard` + `JwtAuthGuard` aplicados
[ ] Status codes corretos: 200 (ok), 201 (criado), 400 (validação), 401 (não autenticado), 403 (proibido), 422 (schema inválido), 429 (rate limit), 500 (erro interno)
[ ] Body de erro segue schema `ErrorResponse` de D20: `{ code, category, message, user_message, correlation_id, timestamp }`
[ ] Headers de segurança presentes: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`
[ ] CORS configurado com whitelist de origens — sem `origin: '*'` em produção
[ ] SSE endpoints retornam `Content-Type: text/event-stream` com `Cache-Control: no-cache`
[ ] Pagination implementada em endpoints de listagem (sem retornar todos os registros sem limite)
```

### 2.4 Dados e Banco

```
[ ] [BLOQUEANTE] Índices adicionados em colunas de filtro frequente: `cessionario_id`, `status`, `created_at`
[ ] Foreign keys com `ON DELETE` explícito (CASCADE ou RESTRICT — nunca implícito)
[ ] Migration testada em ambiente local antes do PR
[ ] Dados sensíveis nunca armazenados em texto plano (OTP como bcrypt, phone como hash em Redis)
[ ] pgvector queries incluem filtro de namespace por `cessionario_id`
[ ] Sem N+1 queries — verificar com `prisma.$queryRaw` explain analyze se necessário
```

### 2.5 Performance

```
[ ] Cache Redis utilizado para resultados de calculadora: `dani:cache:calc:{opr_id}:{val_hash}`
[ ] Componentes React com re-render excessivo verificados com React DevTools
[ ] Bundle size de frontend não cresceu >10% em relação à versão anterior (Vite bundle report)
[ ] Lazy loading aplicado em rotas não-críticas
[ ] Imagens otimizadas (WebP ou AVIF, lazy loading nativo)
```

---

## 3. Parte 2 — Acessibilidade (WCAG 2.1 AA)

### 3.1 ARIA Roles e Landmarks

```
[ ] [BLOQUEANTE] `role="main"` presente na área de conteúdo principal de cada página
[ ] [BLOQUEANTE] `role="dialog"` no painel de chat com `aria-labelledby` apontando para o título
[ ] [BLOQUEANTE] `role="log"` ou `aria-live="polite"` na área de mensagens do chat (anúncio automático para screen readers)
[ ] `role="navigation"` em menus de navegação principal
[ ] `aria-label` em todos os botões sem texto visível (FAB, ícones de ação)
[ ] `aria-labelledby` em formulários complexos associando label ao grupo de campos
[ ] `aria-expanded` atualizado dinamicamente em dropdowns e painéis colapsáveis
[ ] `aria-busy="true"` durante carregamento de dados assíncronos
[ ] `aria-live="assertive"` apenas para erros críticos que exigem atenção imediata
```

**Ferramenta:** axe-core (`@axe-core/playwright`) — zero violações `critical` ou `serious`.

### 3.2 Focus Management

```
[ ] [BLOQUEANTE] Focus trap ativo quando painel de chat está aberto (Tab não sai do painel)
[ ] [BLOQUEANTE] Focus restaurado para o elemento acionador (FAB) quando painel de chat fecha
[ ] Tab order lógico e coerente com layout visual — sem saltos inesperados
[ ] Visible focus ring presente em todos os elementos interativos (não removido via `outline: none` sem substituto)
[ ] Primeiro elemento focável é o link "Pular para conteúdo principal" (skip link)
[ ] Modal/drawer captura focus no primeiro elemento focável ao abrir
[ ] Elementos de overlay não recebem focus quando fechados (`visibility: hidden` ou `display: none`)
```

**Ferramenta:** teste manual com Tab/Shift+Tab; verificação no código por `outline: none` sem `focus-visible`.

### 3.3 Screen Reader Flows

```
[ ] [BLOQUEANTE] Estados de erro anunciados via `aria-live` ou `role="alert"` — não apenas visualmente
[ ] Estado de carregamento (Skeleton) tem texto alternativo para screen reader: `aria-label="Carregando oportunidades"`
[ ] Estado Empty tem texto descritivo visível e acessível: não apenas ícone sem texto
[ ] Tabelas com `<thead>`, `<th scope="col">` e `<th scope="row">` quando aplicável
[ ] Formulários com `<label>` associado por `for`/`id` ou `aria-label` — sem placeholder como único label
[ ] Links descritivos — sem "clique aqui" ou "saiba mais" sem contexto
[ ] Imagens informativas com `alt` descritivo; imagens decorativas com `alt=""`
```

**Ferramenta:** NVDA + Chrome (Windows) ou VoiceOver + Safari (macOS) — testar fluxo de chat completo.

### 3.4 Skip Links

```
[ ] Link "Pular para conteúdo principal" presente como primeiro item do DOM
[ ] Skip link visível ao receber focus (não oculto permanentemente)
[ ] Skip link aponta para `<main>` ou elemento com `id="main-content"`
```

### 3.5 Motion Reduction

```
[ ] [BLOQUEANTE] `@media (prefers-reduced-motion: reduce)` aplicado a todas as animações de transição não essenciais
[ ] Animações do chat (mensagens aparecendo) desabilitadas ou reduzidas com `prefers-reduced-motion`
[ ] Skeleton animation (`animate-pulse`) desabilitada com `prefers-reduced-motion`
[ ] Nenhuma animação que pisca >3 vezes por segundo (risco de fotossensibilidade)
```

**Ferramenta:** DevTools → Emulate CSS media feature `prefers-reduced-motion: reduce`.

### 3.6 Contraste

```
[ ] [BLOQUEANTE] Texto normal: ratio de contraste ≥ 4.5:1 (WCAG AA)
[ ] [BLOQUEANTE] Texto grande (≥18pt ou ≥14pt bold): ratio ≥ 3:1
[ ] Elementos de UI (borders, ícones informativos): ratio ≥ 3:1
[ ] Contraste verificado também em dark mode (se implementado)
[ ] Texto sobre imagem ou gradiente tem overlay com contraste garantido
```

**Ferramenta:** axe-core; Colour Contrast Analyser; DevTools accessibility panel.

**[DECISÃO AUTÔNOMA]:** Threshold AA (4.5:1) adotado como mínimo — Justificativa: padrão legal mínimo verificável via axe-core com baixo índice de falso positivo | Alternativa descartada: AAA (7:1) — restritivo demais para o design system atual e aumenta falsos positivos sem obrigação legal correspondente.

### 3.7 Teclado

```
[ ] [BLOQUEANTE] Todas as funcionalidades do chat acessíveis via teclado (enviar mensagem com Enter, abrir/fechar painel com Esc)
[ ] Sem keyboard traps — Esc sempre fecha modais e painéis
[ ] Atalhos de teclado documentados se existirem (`aria-keyshortcuts` ou documentação de UX)
[ ] Elementos com `onClick` customizado também têm `onKeyDown` para Enter/Space
[ ] Elementos `<div>` ou `<span>` interativos têm `tabIndex="0"` e role apropriado
```

### 3.8 Touch Targets (Mobile)

```
[ ] Todos os elementos interativos têm área mínima de 44×44px (WCAG 2.5.5)
[ ] FAB (botão flutuante do chat) com dimensão ≥ 56×56px
[ ] Botão de envio de mensagem ≥ 44×44px
[ ] Espaçamento entre elementos clicáveis ≥ 8px para evitar clique acidental
[ ] Verificado em viewport ≤ 640px (breakpoint mobile da Fase 1)
```

**Ferramenta:** DevTools → Device Toolbar; Lighthouse "Tap targets" audit.

---

## 4. Parte 3 — Segurança

### 4.1 Autenticação e Autorização

```
[ ] [BLOQUEANTE] JWTs validados com `JwtAuthGuard` em todos os endpoints protegidos
[ ] [BLOQUEANTE] `CessionarioOwnerGuard` ativo em todos os endpoints que retornam dados de cessionário
[ ] [BLOQUEANTE] OTP armazenado como hash bcrypt (custo 12) — nunca em texto plano
[ ] [BLOQUEANTE] Phone armazenado apenas como SHA-256 em chaves Redis — nunca em texto no Redis
[ ] JWT com expiração de 15min (access) e 7 dias (refresh) — conforme D18
[ ] Refresh token revogado via Redis blacklist (`dani:revoked:refresh:{jti}`) ao logout
[ ] Hard block de OTP: 5 falhas → TTL 1800s em `dani:block:otp:{phone_hash}`
[ ] Rate limit de OTP: 3 solicitações/hora por telefone
[ ] Admin bypass documentado e isolado (`role === 'ADMIN'` em `CessionarioOwnerGuard`)
```

### 4.2 Validação de Input

```
[ ] [BLOQUEANTE] Todos os DTOs validados com `class-validator` antes de processar
[ ] [BLOQUEANTE] Payloads de mensagem do chat sanitizados antes de enviar ao LLM (sem injection de system prompt)
[ ] Tamanho máximo de mensagem do chat definido e validado (ex: 2000 caracteres)
[ ] Campos numéricos da calculadora validados: não negativos, não NaN, não Infinity
[ ] Telefone validado no formato E.164 antes de armazenar hash
[ ] UUID validado com regex antes de usar como `cessionario_id` ou `opr_id`
```

### 4.3 Secrets e Variáveis de Ambiente

```
[ ] [BLOQUEANTE] Nenhuma chave de API, senha ou segredo hardcoded no código fonte
[ ] [BLOQUEANTE] GitLeaks passou sem findings no CI
[ ] `.env` listado no `.gitignore` — apenas `.env.example` no repositório
[ ] Variáveis sensíveis classificadas no D22 e armazenadas no GitHub Actions Secrets
[ ] JWT_SECRET com entropia mínima de 256 bits
[ ] Rotação de secrets documentada em D22 com plano de rotação de JWT (15min overlap)
```

### 4.4 Transporte e Headers

```
[ ] HTTPS obrigatório em todos os ambientes (TLS 1.2+)
[ ] `Strict-Transport-Security: max-age=31536000; includeSubDomains` presente
[ ] `X-Content-Type-Options: nosniff` presente
[ ] `X-Frame-Options: DENY` presente
[ ] `Content-Security-Policy` configurado (sem `unsafe-inline` em produção)
[ ] Cookies de session com `httpOnly: true`, `secure: true`, `SameSite: Strict`
```

### 4.5 Dados Sensíveis e Logs

```
[ ] [BLOQUEANTE] Pino redact ativo para: `phone`, `otp`, `jwt`, `cpf`, `password`, `apiKey`, `cessionario_id` raw
[ ] Stack traces não expostos em respostas de produção (apenas `correlation_id`)
[ ] Langfuse: `userId = sha256(cessionario_id)` — UUID raw nunca em traces
[ ] RAG: namespace pgvector por `cessionario_id` — busca semântica não retorna chunks de outro tenant
[ ] Logs de auditoria retidos por 1 ano (LGPD)
```

### 4.6 Dependências

```
[ ] `pnpm audit` sem CVEs `critical` ou `high`
[ ] Dependências atualizadas na última semana (verificar `pnpm outdated`)
[ ] Nenhuma dependência abandonada (último commit > 2 anos) em path crítico
[ ] Licenças verificadas (sem GPL em código proprietário)
```

### 4.7 Rate Limiting e Proteção contra Abuso

```
[ ] Rate limit de webchat: 30 mensagens/hora por `cessionario_id` (Redis sliding window)
[ ] Rate limit de OTP: 3 solicitações/hora por telefone (chave `dani:rate:otp:{phone_hash}`)
[ ] Hard block ativo após 5 falhas de OTP (TTL 1800s)
[ ] CORS whitelist configurada — sem `origin: '*'`
[ ] Payload de request com limite de tamanho (ex: 1MB máx por request)
```

---

## 5. Parte 4 — Observabilidade e Operação

```
[ ] [BLOQUEANTE] `correlation_id` presente em todos os logs do fluxo (request → LLM → response)
[ ] [BLOQUEANTE] `GET /health` retorna 200 com `{ status: "ok", version: "X.Y.Z", dependencies: {...} }`
[ ] Nível de log correto por ambiente: `debug` local, `info` staging/produção
[ ] Sentry configurado com `dsn` de produção; releases e ambiente (`environment: "production"`) configurados
[ ] Langfuse: traces criados por sessão com `userId = sha256(cessionario_id)`, `sessionId`, `correlationId`
[ ] Alertas críticos configurados no Sentry (ALT-001 a ALT-012 de D26)
[ ] RabbitMQ Management UI acessível com credenciais de operação
[ ] Redis: chaves de rate limit e block com TTL correto verificado pós-deploy (`redis-cli TTL`)
[ ] Smoke tests pós-deploy executados e passando (script `smoke-tests.sh` de D24)
[ ] Dashboard de latência p95 atualizado e refletindo versão atual
```

---

## 6. Itens Bloqueantes

| **Categoria** | **Item** | **Por que bloqueia** | **Como verificar** | **Impacto se ignorado** |
|---|---|---|---|---|
| Frontend | 4 estados UI (Skeleton → Empty → Error → Populated) | Spinners proibidos por contrato D09 | Inspecionar DOM/Storybook | UX quebrada; accessibility failure |
| Frontend | Barrel exports obrigatórios por feature | Import direto quebra encapsulamento e tree-shaking | `grep -r "from '../"` entre features | Acoplamento silencioso; refactor caro |
| Backend | `CessionarioOwnerGuard` em todo endpoint de negócio | Vazamento de dados cross-tenant = P0 de segurança | Unit test C-04 + integration test O-09 | LGPD violation; incidente P0 |
| Backend | Toda query com `WHERE cessionario_id = :id` | Query sem filtro retorna dados de todos os cessionários | Integration test de isolamento (A-04) | Falha de isolamento = P0 |
| Backend | Pino redact ativo (sem PII em logs) | Log com phone/CPF/JWT = violação LGPD | Grep em logs locais: `phone`, `cpf`, `jwt` | LGPD; exposição de dado sensível |
| Backend | OTP como bcrypt hash (nunca texto plano) | OTP em texto plano = credential exposure | Unit test O-02 | Autenticação comprometida |
| API | Endpoint documentado em D16 | API não documentada = contrato quebrado para consumidores | Verificar D16 + Pact contrato | Breaking change silencioso |
| API | `CessionarioOwnerGuard` + `JwtAuthGuard` em endpoints protegidos | Endpoint sem guard = acesso anônimo a dados | `curl` sem token deve retornar 401 | Acesso não autorizado |
| Segurança | Nenhuma chave de API hardcoded | Secret em repositório público = comprometimento imediato | GitLeaks no CI | Rotação emergencial; incidente P0 |
| Acessibilidade | `role="dialog"` + focus trap no painel de chat | Screen reader sem role não anuncia diálogo | axe-core + NVDA/VoiceOver manual | WCAG 2.1 AA violation; usuários de TA excluídos |
| Acessibilidade | Contraste texto ≥ 4.5:1 | Requisito legal WCAG 2.1 AA | axe-core no CI | Violação de acessibilidade; exclusão |
| Acessibilidade | `aria-live="polite"` em mensagens do chat | Screen reader não anuncia novas mensagens | axe-core + VoiceOver test | Usuários de TA não recebem respostas da Dani |
| Acessibilidade | `prefers-reduced-motion` respeitado | Animações causam desconforto ou crise para usuários com condições vestibulares | CSS media query presente | Exclusão; possível violação regulatória |
| Observabilidade | `correlation_id` em todos os logs | Sem correlação, diagnóstico de incidente é inviável | Grep em logs de fluxo completo | MTTR alto em incidentes |
| Observabilidade | `GET /health` retorna 200 | Sem health check, Railway não pode detectar falha de deploy | `curl` pós-deploy | Deploy silenciosamente quebrado |

---

## 7. Ferramentas de Apoio

| **Ferramenta** | **Categoria** | **O que valida** | **Quando usar** | **Limitação** |
|---|---|---|---|---|
| `@axe-core/playwright` | Acessibilidade | WCAG 2.1 AA automatizado | PR para main; pré-release | Não detecta erros de fluxo de screen reader que exigem teste manual |
| Lighthouse CI | Acessibilidade + Performance | Score de acessibilidade; tamanho de bundle | PR para main | Falsos negativos em componentes dinâmicos |
| NVDA + Chrome (Windows) | Acessibilidade | Screen reader real — anúncios, focus, live regions | Auditoria quinzenal; pré-release | Requer hardware Windows |
| VoiceOver + Safari (macOS) | Acessibilidade | Screen reader real em macOS/iOS | Auditoria quinzenal | Requer macOS |
| GitLeaks | Segurança | Secrets hardcoded no código | Cada PR (CI automático) | Não detecta secrets em variáveis calculadas |
| `pnpm audit` | Segurança | CVEs em dependências | Semanal + todo PR | Não detecta vulnerabilidades de lógica |
| Vitest coverage | Code quality | Cobertura de linhas, branches, funções | PR (CI automático) | Cobertura alta ≠ testes de qualidade |
| Playwright | E2E + Acessibilidade | Fluxos completos + axe integrado | PR para main + pré-release | Lento para feedback rápido em feature branches |
| Sentry | Observabilidade | Erros em produção; alertas | Pós-deploy; monitoramento contínuo | Não captura erros de lógica sem exceção |
| Langfuse | Observabilidade LLM | Latência, custo, traces do agente | Pós-deploy; custo semanal | Dados com delay de alguns segundos |
| RabbitMQ Management | Observabilidade | Profundidade de filas, DLQ | Checklist diário | Interface web — sem alertas nativos |
| `redis-cli` | Operação | TTLs, rate limits, bloqueios de OTP | Pós-deploy; diagnóstico de incidente | Requer acesso à rede do Redis |

---

## 8. Mapeamento por Categoria

| **Checklist** | **Referência** | **Padrão** |
|---|---|---|
| Isolamento `cessionario_id` | D14 — Especificações Técnicas; D01 — RN-DC-001 | Princípio interno: Isolamento Total |
| OTP e autenticação | D18 — Fluxos de Autenticação OAuth; RN-DC-040–044 | Interno + OWASP A07 (Identification Failures) |
| Contraste e acessibilidade | D09 — Contratos de UI; D08 — UX Writing | WCAG 2.1 AA (ISO/IEC 40500) |
| Secrets e hardcoding | D22 — Guia de Ambiente | OWASP A02 (Cryptographic Failures) |
| Validação de input | D16 — Documentação de API; DTOs NestJS | OWASP A03 (Injection) |
| Headers de segurança | D14 — Especificações Técnicas | OWASP A05 (Security Misconfiguration) |
| Rate limiting | D01 — RN-DC-030; D14 | OWASP A04 (Insecure Design) |
| Observabilidade e logs | D25 — Observabilidade e Logs | Interno + LGPD (dados de auditoria) |
| Conventional commits | D23 — Guia de Contribuição | Interno (Conventional Commits 1.0.0) |
| Dados sensíveis em logs | D25 — Observabilidade; D20 — Error Handling | LGPD Art. 46; OWASP A09 (Logging Failures) |
| Bundle performance | D02 — Stacks; D14 | Interno: SLO web < 3s LCP |
| Testes de contrato | D27 — Plano de Testes | Pact Consumer-Driven Contract Testing |

---

## 9. Periodicidade de Auditoria

### 9.1 Em Todo PR

Partes obrigatórias: Code Review (Parte 1) + Segurança P0 items (Parte 3.1 a 3.3).

Ferramenta automática: GitLeaks + Vitest coverage + axe-core (PR para main).

### 9.2 Auditoria Quinzenal (toda segunda-feira de releases)

Partes obrigatórias: todas. Foco especial:
- Verificar novos CVEs em `pnpm audit`.
- Testar fluxo de screen reader manualmente (NVDA ou VoiceOver).
- Verificar que `prefers-reduced-motion` continua ativo após novos componentes de animação.
- Confirmar que nenhuma chave rotacionada está vencida (D22 — checklist de rotação).

### 9.3 Pré-Release

Partes obrigatórias: todas + Tabela de Bloqueantes totalmente preenchida (nenhum `[ ]` restante).

Requerimento adicional: relatório de axe-core em todas as páginas do fluxo E2E obrigatório.

### 9.4 Pós-Incidente Crítico

Partes obrigatórias: Parte 3 (Segurança) + Parte 4 (Observabilidade).

Objetivo: verificar se o incidente revelou gap no checklist. Se sim, adicionar item novo e atualizar este documento.

---

## 10. Backlog de Pendências

| **ID** | **Tipo** | **Descrição** | **Impacto** |
|---|---|---|---|
| P-01 | PENDÊNCIA | Content-Security-Policy específico do produto não definido — política genérica usada | CSP incorreto pode permitir XSS ou bloquear recursos legítimos |
| P-02 | [DECISÃO AUTÔNOMA] | Teste de VoiceOver definido como auditoria manual quinzenal, não automatizado no CI — Justificativa: não há ferramenta de automação confiável para screen reader real em CI; teste manual é mais fidedigno | Alternativa descartada: Selenium + screen reader virtual — falsos negativos frequentes |
| P-03 | PENDÊNCIA | Checklist de dark mode — produto usa apenas light mode na Fase 1; dark mode não implementado | Seção de contraste em dark mode marcada como N/A até Fase 2 |
| P-04 | PENDÊNCIA | `Content-Security-Policy` para assets externos (OpenAI streaming, Supabase realtime) precisa ser testado — whitelist de origens não definida | CSP bloqueando SSE ou Supabase realtime em produção |

---

*Documento gerado por Claude Code Desktop — Pipeline ShiftLabs v9.5 — AI-Dani-Cessionário*
