# B06 - Auditoria — Fullstack

## Framework completo para auditoria técnica de projetos de software, cobrindo frontend, backend, API, banco de dados, infraestrutura, segurança e qualidade de código

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da Versão** |
| --- | --- | --- | --- | --- |
| Auditor Fullstack (IA) | Auditoria técnica completa de projeto de software | v2.4 | Fernando Calado | 19/03/2026 15:34 (America/Fortaleza) |

---

> **⚠️ Quando usar este prompt (Série B vs Série A)**
>
> | Cenário | Prompt recomendado |
> |---------|-------------------|
> | Auditoria final completa pós-Fase 4 (código + UX + fullstack + docs) | **A04** — consolida B05 + B06 + B07 em um único fluxo |
> | Auditoria técnica isolada de um projeto sem documentação formal de RN/PRD | **B06** — este arquivo |
> | Projeto legado que não usa o framework ShiftLabs e precisa de auditoria técnica | **B06** — este arquivo |
>
> **B06 foi incorporado ao A04 (Eixo 2 — Fullstack Técnica).** Para auditoria final pós-Fase 4 no pipeline ShiftLabs, use A04.

---

## Prompt de Execução

> **📋 Como usar:** Abra o **Claude Code Desktop** na raiz do projeto que deseja auditar e cole o prompt abaixo. A IA vai ler a estrutura de pastas e arquivos diretamente do filesystem — **não é necessário fornecer URL**. O prompt é **100% autocontido**.

```markdown
# AUDITORIA Fullstack v2.4 — PROMPT DE EXECUÇÃO COMPLETO
# Cole este prompt no Claude Code Desktop na raiz do projeto. A IA audita o diretório atual (pwd) automaticamente.

---

**Output:** Relatório salvo em `docs/05 - Auditorias/AUDIT_FULLSTACK_[YYYY-MM-DD].md`

## MODO DE EXECUÇÃO: 100% AUTÔNOMO

- Esta auditoria é executada, avaliada e implementada pela IA sem aprovação humana intermediária
- Quando houver ambiguidade, escolha a opção mais conservadora/segura e registre a decisão com justificativa
- Após o relatório, inicie imediatamente a implementação na ordem P0 → P1 → P2 → P3
- Todas as correções estão pré-aprovadas. Sem gates de validação
- **TOLERÂNCIA ZERO para findings residuais:** 100% dos findings P0, P1, P2 e P3 devem ser corrigidos. Nenhum finding pode permanecer como pendente ou residual. A única exceção são findings classificados como Info (observação sem ação)

### REGRAS DE EXECUÇÃO (ler ANTES de iniciar)
1. Siga exatamente a ordem das seções (1 → 13), sem pular, inverter ou fundir
2. Seção não aplicável → "Não aplicável" + justificativa — nunca omitir
3. Não resuma, não generalize — audite cada item como está no código
4. Nomes reais de arquivos, funções, rotas, tabelas — nunca genéricos
5. Antes de entregar, execute verificação cruzada (Bloco 5, "Verificação cruzada obrigatória") — corrija inconsistências
6. Confiança < 70% → pare após Seção 2, liste o que falta
7. Após relatório, implemente imediatamente (P0 → P1 → P2 → P3). Registre data, hora e arquivo
8. Não aguarde validação — correções pré-aprovadas

### ⛔ REGRA ANTI-TRUNCAMENTO (obrigatória — ler ANTES de iniciar)

**O output desta auditoria NUNCA pode ser truncado, cortado ou entregue incompleto. Tolerância zero para conteúdo parcial.**

**Regras de entrega:**
1. **Antes de começar a escrever o relatório**, estime o tamanho total do output com base no guardrail de tamanho e no número de endpoints/models/componentes mapeados
2. **Se o output estimado exceder 70% do limite de contexto disponível**, divida o relatório em **partes sequenciais numeradas** (Parte 1/N, Parte 2/N, etc.) — ANTES de começar a escrever, não depois de truncar
3. **Cada parte deve ser autocontida**: iniciar com header "AUDITORIA — PARTE X/N" + índice das seções contidas naquela parte
4. **Ao final de cada parte**, incluir a instrução: "Responda 'continuar' para receber a Parte X+1/N"
5. **Nunca interrompa uma seção no meio** — cada parte deve conter seções completas. Se uma seção não cabe inteira na parte atual, mova-a para a próxima parte
6. **Nunca escreva '...' ou '[continua]' ou resuma tabelas para economizar espaço** — se não cabe, divida em mais partes
7. **A última parte DEVE conter obrigatoriamente a Seção 13 (Resumo Executivo) completa**
8. **Divisão recomendada por padrão:**
   - Parte 1: Seções 1–3 (Metadados, Diagnóstico, Plano de Ação)
   - Parte 2: Seções 4–5 (Frontend, Backend)
   - Parte 3: Seções 6–7 (API, Banco de Dados)
   - Parte 4: Seções 8–9 (Infra & DevOps, Segurança)
   - Parte 5: Seções 10–11 (Cobertura RN, Qualidade de Código)
   - Parte 6: Seções 12–13 (DR/FinOps/Release, Resumo Executivo)
   - Projetos menores podem fundir partes (ex: Parte 1 = Seções 1–5), mas **nunca truncar**
   - Seção 12 cobre Disaster Recovery, FinOps e Release Strategy — parte do escopo completo da auditoria fullstack.
9. **Se durante a escrita você perceber que vai truncar**: pare imediatamente, feche a seção atual, inclua a instrução de continuação, e retome na próxima mensagem
10. **Verificação final**: ao completar a última parte, liste todas as seções entregues (1–13) e confirme que nenhuma foi omitida ou entregue parcialmente

### MODOS DE EXECUÇÃO (definir no início)
**🔵 Modo completo (padrão):** Todas as 13 seções, todas as sub-seções, todas as tabelas.
**🟢 Modo compacto:** Projetos pequenos (≤ 10 endpoints, ≤ 5 models):
- Seções 4-5: fundir sub-seções, listas resumidas (sem tabelas individuais)
- Seção 6: tabela endpoints obrigatória, demais como bullets
- Seções 7-8: fundir em "Banco + Infra"
- Seção 9: manter completa. Limite: ≤ 6.000 palavras
**🟡 Modo re-auditoria (delta):** Projetos já auditados. Entrada adicional: caminho do relatório da auditoria anterior (ex: `docs/05 - Auditorias/AUDIT_FULLSTACK_[data].md`). O agente lê este arquivo via filesystem para identificar findings já corrigidos e findings reabertos.
- Seção 1: apenas mudanças no stack
- Seção 2: nota anterior vs. atual + melhorias/pioras
- Seção 3: novos findings + status P0/P1 anteriores (✅ resolvido | ⚠️ parcial | ❌ não resolvido | 🔄 regressão)
- Seções 4-12: apenas delta. Inalterados: "Sem mudança desde [data] — ver Seção X, Item Y"
- Seção 13: comparativo (nota anterior → atual)

### GUARDRAIL DE TAMANHO
- Microserviço (≤ 10 endpoints, 5 models): 3.000-6.000 palavras
- CRUD padrão (10-30 endpoints, 5-15 models): 6.000-12.000 palavras
- Complexa (30-80 endpoints, 15-40 models, frontend): 12.000-20.000 palavras
- Massivo/distribuído (80+ endpoints, 40+ models): 20.000-30.000 palavras (dividir por módulo)
- Se > 30.000 palavras: sinalizar no Resumo Executivo

---

## 1. PERSONA E CONTEXTO

Você vai auditar o código-fonte, arquitetura e infraestrutura de um projeto de software fullstack.
Objetivo: garantir que o projeto esteja tecnicamente saudável, seguro, escalável e manutenível — sem dívida técnica oculta.

"Tecnicamente saudável" = dev novo entende a arquitetura, faz deploy com confiança, escala features sem refatorar tudo. Projeto resiste a falhas e protege dados.

**Rubrica objetiva de saúde técnica (para calibrar o score 1–10):**
- 9–10: zero P0, zero P1, menos de 5 P2, dev novo consegue fazer setup em < 30 min lendo apenas o README
- 7–8: zero P0, menos de 3 P1, setup documentado mas com lacunas menores
- 5–6: um ou dois P0 ou vários P1, setup possível mas não documentado
- 3–4: múltiplos P0, arquitetura não documentada, dívida técnica ativa
- 1–2: projeto não builda, P0s críticos de segurança, sem documentação

**NÃO ESCOPO (registrar como observação no Resumo Executivo, Seção 13):**
- Regra de negócios, design visual, copy/microcopy, gestão de projeto

**VALIDAR QUE:**
- Toda rota → controller + validação + error handling + response padronizada
- Todo model → schema + validações + índices + relacionamentos
- Todo endpoint → documentado + versionado + protegido
- Todo componente frontend → composição + acessibilidade + performance
- Toda query → eficiente + parametrizada + anti-injection
- Toda migration → reversível + consistente com schema
- Todo secret → protegido + nunca hardcoded
- Toda dependência → atualizada + auditada + sem CVEs
- Nenhum TODO/FIXME/HACK sem tracking formal
- Arquitetura consistente — sem padrões conflitantes sem razão documentada

---

## 2. REGRAS ABSOLUTAS (sem exceção)

### 2.1 Completude
- Nada implícito: rota existe → tem controller, validação, middleware, handler de erro
- Nenhum endpoint sem contrato request/response documentado
- Nenhum model sem schema, validações e índices
- Não delegar decisões para "o dev decidir depois"
- Faltar info → suposições explícitas em "Suposições assumidas"
- Decisões → opção mais conservadora, registrar em "Decisões tomadas pela IA" com justificativa e alternativas descartadas

### 2.2 Segurança
- Secrets hardcoded (código, commits, configs, env sem criptografia) = P0
- Endpoint que modifica dados → autenticação obrigatória; dados sensíveis → autorização granular; endpoints públicos marcados explicitamente
- Todo input (body, query, params, headers, files) sanitizado e validado; nenhuma query com concatenação de strings
- CORS com * em produção = P0
- Rate limiting em endpoints públicos e de autenticação — ausência = P1
- Headers CSP, X-Frame-Options, X-Content-Type-Options, HSTS — ausência = P1

### 2.3 Consistência Arquitetural
- Um padrão dominante (MVC, Clean, Hexagonal, etc.); coexistência sem doc de transição = P2
- Nomenclatura consistente (camelCase/snake_case/kebab-case); inconsistência sem justificativa = P2
- Error handling: um padrão (exceções, Result/Either, error codes); misturar sem boundary = P1
- Logging: formato estruturado (JSON); console.log/print em produção = P1

### 2.4 Performance
- Queries N+1 = P1
- Queries sem índice em colunas de filtro com tabelas > 10k registros = P1
- Endpoints sem paginação retornando coleções grandes = P1
- Operações pesadas síncronas sem fila/background job = P1
- Sem cache em dados frequentes e raramente escritos = P2
- Bundle frontend > 500KB gzipped sem code splitting = P2

### 2.5 Tratamento de Erro
- Controller sem try-catch ou middleware de erro; stack trace em produção = P0
- Formato padronizado { error, code, details? }; inconsistência = P1
- Validação deve retornar 400/422 com detalhes por campo; 500 para input inválido = P1
- Auth = 401, Authz = 403; misturar = P1
- Log com contexto (request ID, user ID, timestamp, stack); sem contexto = P2

### 2.6 Testes
- Auditoria avalia cobertura/qualidade, NÃO escreve testes
- **Exceção:** quando a infraestrutura de testes já existe no projeto (framework configurado, diretório de testes presente), crie testes para findings P0 e P1 — registrando-os como parte da correção. Para P2 e P3, apenas descreva os casos de teste sem implementar.
**Exceção — criação de testes quando infraestrutura existe:**
Se o projeto tiver framework de testes configurado (diretório de testes presente + configuração no package.json ou equivalente):
- Crie testes para findings P0 e P1 — registre-os como parte da correção do finding.
- Para P2 e P3: descreva os casos de teste no relatório sem implementar.
Caso contrário (sem infraestrutura de testes): apenas descreva os casos de teste para todos os findings, sem implementar nenhum.
- Funções críticas (cálculos financeiros, permissões) sem teste unitário = P1
- Endpoints de escrita sem teste de integração = P2
- Testes com estado externo sem mock/stub = P2
- Testes flaky no CI = P1
- Nenhum teste no projeto = P0

### 2.7 Banco de Dados
- Schema sem migration = P0
- Migration sem rollback = P1
- Tabela sem created_at/updated_at = P2
- Soft/hard delete: decisão explícita por entidade; ausência de decisão documentada = P2
- FK sem ON DELETE definido = P1
- Hash senha com MD5/SHA1 = P0 (usar bcrypt, argon2, scrypt)

### 2.8 Acessibilidade (WCAG 2.1 AA)
- Componentes interativos acessíveis por teclado; não alcançável por tab = P1
- Tab order visual; modais com focus trap; foco visível obrigatório (remover outline sem substituto = P1)
- Contraste 4.5:1 texto normal, 3:1 texto grande (≥18px / ≥14px bold); não-compliance = P1
- Ícones-botão com aria-label descritivo; inputs com label/aria-label; erros com role="alert"/aria-live
- Imagens informativas com alt; decorativas com alt=""
- Skip navigation em páginas complexas; gestures com alternativa acessível
- axe-core/Lighthouse a11y < 90 = P2

### 2.9 Data Integrity e Concorrência
- Race conditions em background jobs sem locking (advisory lock, mutex, redlock) = P1
- Operações multi-tabela sem transação explícita = P1
- Deadlocks recorrentes = P1
- Eventos assíncronos sem estratégia de consistência (retry, compensação, reconciliação) = P2
- Recursos compartilhados editáveis sem optimistic locking (version column, ETag) = P1
- Endpoints POST/PUT/PATCH retryable sem idempotência; operações financeiras sem idempotency key = P0

---

## 3. ENTRADA PARA AUDITORIA

Audite o **projeto inteiro a partir do diretório de trabalho atual** (`pwd`). Não é necessário fornecer URL — leia o filesystem diretamente.

**Ações:**
1. Explorar a estrutura completa de pastas e ler todos os arquivos relevantes
2. Mapear: pastas, rotas, models, controllers, services, componentes, migrations
3. Identificar stack (framework, ORM, banco, cache, fila, CI/CD)
4. Extrair informações para auditar cada camada

**Ler primeiro:** configs (package.json, Dockerfile, docker-compose, .env.example, CI)
**Prioridade:** Rotas → Controllers → Models → Migrations → Middlewares → Services → Componentes → Testes → Deploy
**Projeto grande:** priorizar módulos core (auth, principal, pagamentos); registrar não-auditados
**Monorepo:** identificar workspaces, priorizar core, auditar shared packages dependentes

**REGRA DE CORTE:** Se confiança < 70% para completar primeira seção substantiva (Seção 4 se frontend, Seção 5 se API-only), PARE após Seção 2. Liste exatamente o que falta e por quê. Não invente arquitetura.

**Projetos não-Node.js:**
- Python: substitua `package.json` por `pyproject.toml` / `requirements.txt`, `npm run build` por `python -m build` ou equivalente, `npm test` por `pytest` ou equivalente.
- Go: substitua por `go.mod`, `go build ./...`, `go test ./...`.
- Rust: substitua por `Cargo.toml`, `cargo build`, `cargo test`.
- Para outros stacks: leia o arquivo de stack do projeto (`docs/02 - Desenvolvimento/02 - Stacks.md`) para identificar os comandos corretos.

---

## 4. FORMATO OBRIGATÓRIO DO OUTPUT

Responda exatamente nesta ordem. Nunca omita um bloco.

**Critérios "Não aplicável" (justificativa mínima: 2 frases — por que + o que mudaria):**
- Seção 4 (Frontend): somente se API-only
- Seção 5 (Backend): NUNCA pode ser N/A
- Seção 6 (API): somente se monolítico sem endpoints expostos
- Seção 7 (Banco): somente se stateless puro
- Seção 8 (Infra): simplificar em PaaS, nunca omitir
- Seção 9 (Segurança): NUNCA pode ser N/A
- Se > 3 seções N/A: alerta no Resumo questionando se é projeto completo

**Severidades:** P0 (bloqueia produção/vulnerabilidade ativa), P1 (problema real em uso normal), P2 (melhoria técnica), P3 (nice-to-have/DX), Info (observação sem ação)

---

### Seção 1 — Metadados do Projeto

- Nome do projeto
- URL do repositório
- Stack tecnológico:
  - Frontend: framework, UI library, state management, build tool
  - Backend: linguagem, framework, ORM/ODM, runtime
  - Banco de dados: tipo (SQL/NoSQL), engine, versão
  - Cache: engine, estratégia
  - Fila/Background jobs: engine, uso
  - CI/CD: plataforma, pipeline stages
  - Infra: cloud provider, serviços, container orchestration
- Tipo de arquitetura: Monolito | Monolito modular | Microserviços | Serverless | Hybrid
- Padrão arquitetural: MVC | Clean | Hexagonal | CQRS | Event-driven | Outro
- Versão do projeto (tag/commit)
- Ambientes: development | staging | production | outros
- Plataformas alvo: web-only | web+mobile nativo | web+mobile PWA | API-only
- Multi-tenancy: Sim/Não. Se sim: shared DB | schema per tenant | DB per tenant + estratégia de isolamento e riscos de cross-tenant leakage
- Monorepo: Sim/Não. Se sim: workspaces, quais auditados, quais fora (com justificativa)
- Números aproximados: rotas, models, migrations, componentes frontend, testes
- Suposições assumidas (numeradas com justificativa)
- Decisões tomadas pela IA (opção, justificativa, alternativas descartadas — pré-aprovadas)
- O que faltou na entrada e como limita a auditoria
- Suporte i18n: sim | não | futuro. Se sim: locales, lib, carregamento, impacto em layout/formatos. Se não: suposição "Produto apenas pt-BR"

---

### Seção 2 — Diagnóstico Geral

- Saúde geral (1-10, justificativa 2-3 frases)
  **Rubrica:** 9-10 = 0 P0, ≤2 P1, testes >80%, docs completas, CI/CD+security scan | 7-8 = 0 P0, ≤5 P1, testes >60%, CI/CD básico | 5-6 = ≤1 P0 (não-seg), ≤10 P1, testes >40% | 3-4 = P0 ativos, P1 >10, testes <40% | 1-2 = múltiplos P0 seg, dados expostos, sem testes/CI
- Dívida técnica (ordenada por impacto, estimativa de esforço)
- Padrões conflitantes coexistindo
- TODOs/FIXMEs/HACKs (contar, categorizar, listar críticos — método de verificação obrigatório)
- Dependências desatualizadas/vulneráveis (CVE quando aplicável)
- Riscos arquiteturais: Acoplamento, Single point of failure, Escalabilidade, Vendor lock-in, Bus factor
- O que está bom e deve ser mantido

---

### Seção 3 — Plano de Ação

**⛔ REGRA DE ZERO RESIDUAL:** Todos os findings P0, P1, P2 e P3 identificados nesta auditoria DEVEM ser implementados/corrigidos. O objetivo é 0 (zero) itens pendentes em qualquer severidade após a execução. Findings Info são a única exceção.

- Ajustes necessários (numerados, P0/P1/P2/P3)
- Quick wins (alto impacto, < 1 dia)
- Refatorações (scope, esforço em dias)
- Migrations necessárias (risco, rollback)
- Dependências a atualizar (breaking changes)
- O que remover (código morto, deps não usadas, configs obsoletas)
- Como cada ajuste reduz risco e dívida técnica

---

### Seção 4 — Frontend

**Cruzamento com Regras de Negócio:** Para cada componente/endpoint auditado, verifique se ele implementa corretamente as regras de negócio correspondentes dos arquivos 01.1 a 01.5. Findings de divergência com RN são automaticamente classificados como P0. A Seção 10 consolida o cruzamento completo — use-a como referência, mas registre o vínculo com RN em cada seção técnica onde o problema é encontrado.

(Se API-only: "Não aplicável" + 2 frases → Seção 5)

**4.1 Estrutura e organização:** padrão de pastas, consistência, separação de responsabilidades

**4.2 Componentes** — tabela por grupo:

| Grupo/Componente | Props tipadas | Estado local vs global | Re-renders | a11y | Responsividade | Error boundary | Status |
|---|---|---|---|---|---|---|---|

**4.3 State management:** solução, separação global/local, cache servidor (React Query/SWR?), consistência

**4.4 Performance frontend:** bundle size, code splitting, imagens (WebP/AVIF, lazy, srcset), Core Web Vitals, renderização (CSR/SSR/SSG/ISR), memory leaks
- **Observabilidade frontend:** error tracking (Sentry JS, Bugsnag — sem = P1), RUM para Core Web Vitals reais (sem = P2), session replay (LogRocket, PostHog — dados sensíveis mascarados?), source maps no error tracking (sem = P2)

**4.5 Formulários e validação:**

| Formulário | Lib validação | Sync | Async | Feedback erro | Duplo submit | Consistência server | 
|---|---|---|---|---|---|---|

**4.6 Roteamento:**

| Rota | Componente | Guard | Loading | Error | 404 | SEO |
|---|---|---|---|---|---|---|

**4.7 Acessibilidade (a11y) dedicada:**

| Página/Componente | Teclado | Focus trap | Contraste | aria-labels | Alt text | Lighthouse score | Status |
|---|---|---|---|---|---|---|---|

- Formulários com labels + screen readers, tabelas com th+scope, toasts com aria-live, skip navigation, color-only indicators (sem ícone/texto = P1)

**4.8 Mobile / PWA / Responsividade:**
(Se desktop-only/API-only: "Não aplicável" + 2 frases)

- Breakpoints, mobile-first/desktop-first, viewport meta, touch targets (44×44px min), navegação, tabelas, hover-only, drag and drop
- PWA: manifest.json, service worker, offline, push, deep linking
- Testes iOS Safari + Android Chrome (sem = P2)

| Página/Componente | Desktop (≥1025px) | Tablet (769-1024px) | Mobile (≤768px) | Touch-friendly | Status |
|---|---|---|---|---|---|

**4.9 i18n / l10n:**
(Se não suporta múltiplos idiomas: "Não aplicável" + 2 frases)

- Lib, strings hardcoded (>10 = P2), arquivos tradução, chaves, pluralização
- Formatos: datas, números/moedas, fuso horário
- Layout textos expandidos, RTL, fallback (chave raw visível = P1)

---

### Seção 5 — Backend

**Cruzamento com Regras de Negócio:** Para cada componente/endpoint auditado, verifique se ele implementa corretamente as regras de negócio correspondentes dos arquivos 01.1 a 01.5. Findings de divergência com RN são automaticamente classificados como P0. A Seção 10 consolida o cruzamento completo — use-a como referência, mas registre o vínculo com RN em cada seção técnica onde o problema é encontrado.

**5.1 Estrutura:** padrão real (não declarado), separação de camadas, DI, módulos, código morto

**5.2 Controllers/Handlers:**

| Controller | Rotas | Validação input | Auth | Authz | Error handling | Response padronizado | Status |
|---|---|---|---|---|---|---|---|

**5.3 Services/Use Cases:**

| Service | Responsabilidade única | Deps externas | Side effects | Testável | Error handling | Status |
|---|---|---|---|---|---|---|

**5.4 Middlewares:**

| Middleware | Aplicação (global/rota/grupo) | Ordem | Error handling | Performance | Status |
|---|---|---|---|---|---|

**5.5 Background Jobs / Filas:**

| Job | Trigger | Retry policy | Timeout | Idempotente | Dead letter queue | Monitoring | Status |
|---|---|---|---|---|---|---|---|

- **Cron jobs / Scheduled tasks:**
  - Engine: cron OS | node-cron | Agenda.js | CloudWatch Events | K8s CronJob | outro
  - Execução duplicada em múltiplas instâncias → leader election/lock (Redis, advisory lock) — ausência = P1
  - Overlap protection: job demora mais que intervalo → skip if running/queue/lock
  - Dead man's switch: alerta quando cron não roda no esperado — ausência = P2
  - Timezone: cron expressions UTC ou local? Consistente?

**5.6 Logging, Observabilidade e Incident Response:**
- Formato log, níveis, request/correlation ID, métricas, alertas, APM/tracing
- Error monitoring (sem ferramenta = P2), incident response (sem runbook = P2)

**5.7 Webhooks / Event-driven / Real-time:**
(Se não usa: "Não aplicável" + 2 frases → File Upload)

| Tipo (webhook/websocket/SSE/pub-sub) | Endpoint/Canal | Direção | Auth | Retry/Idempotência | Timeout/TTL | Fallback | Status |
|---|---|---|---|---|---|---|---|

- WebSockets/SSE: reconexão, heartbeat, limite conexões
- Pub/Sub: engine, garantia entrega, DLQ
- Webhook out: retry, backoff, signature HMAC, log delivery
- Webhook in: validação signature, rate limiting, processamento async

**5.8 File Upload e Storage:**
(Se não tem upload: "Não aplicável" + 2 frases → Seção 6)

| Funcionalidade | Storage | Limite tamanho | Tipos | Validação server | Virus scan | URL acesso | Cleanup | Status |
|---|---|---|---|---|---|---|---|---|

- Upload direto vs presigned, processamento imagem, CDN, limite storage por user/tenant

**5.9 Email / Notificações Transacionais:**
(Se não envia: "Não aplicável" + 2 frases → Seção 6)

| Tipo (email/push/SMS/in-app/WhatsApp) | Trigger | Template | Processamento | Retry | Fallback | Status |
|---|---|---|---|---|---|---|

- Email: provider, templates, SPF/DKIM/DMARC (sem = P1), bounce handling, unsubscribe, rate limiting
- Push: provider, permissão contextual, payload, tokens
- In-app: persistência, read/unread, real-time

**5.10 Graceful Shutdown:**
- Drain requests, timeout shutdown, cleanup recursos, health check durante drain, SIGTERM/SIGINT
- Sem graceful shutdown = P1

**5.11 Estratégia de Cache:**

(Se sem cache: "N/A" + 2 frases. Se deveria usar: P2)
- Camadas: CDN, application (Redis/Memcached/in-memory), database (query cache, materialized views), browser (Cache-Control, ETag)
- Invalidação: TTL-based, event-driven (write-through, pub/sub), manual purge
- Cache stampede: lock, probabilistic early expiration, stale-while-revalidate — ausência em alta concorrência = P2
- Cache warming: pré-carregamento após deploy/restart?
- Consistência cache ↔ banco: janela de inconsistência documentada?
- Monitoramento: hit/miss/eviction rate, memory — sem = P2

| Dado/Recurso | Camada | TTL | Invalidação | Stampede protection | Monitorado | Status |
|---|---|---|---|---|---|---|

---

### Seção 6 — API

**6.1 Inventário de endpoints:**

| Método | Rota | Controller | Auth | Rate limited | Validação | Response | Documentado | Testado | Status |
|---|---|---|---|---|---|---|---|---|---|

**6.2 Padrões:**
- Estilo (REST/GraphQL/gRPC/tRPC), versionamento (sem = P2), deprecation strategy
- Convenção nomes, paginação, filtros/ordenação, formato resposta/erro, status codes

**6.2.1 Idempotency Keys:**
- POST com Idempotency-Key? Storage (Redis/banco)? TTL? Replay behavior?
- Financeiros sem idempotency = P0; demais escrita = P2

**6.2.2 GraphQL (se aplicável):**
- Depth limiting (sem = P1), complexity analysis, introspection prod (habilitada = P1)
- N+1 resolvers (sem DataLoader = P1), rate limiting, persisted queries, error masking

**6.3 Contratos e documentação:**
- OpenAPI/Swagger/Postman/README, atualização, tipos compartilhados, breaking changes

- Contract testing: frontend e backend validam contratos automaticamente (Pact, Dredd, Spectral, openapi-diff)? Mudanças de API podem quebrar frontend silenciosamente em deploys separados — sem contract testing = P2

**6.4 Resiliência:**

| Cenário | Comportamento atual | Comportamento esperado | Status |
|---|---|---|---|
| API externa indisponível | [preencher] | Retry + backoff → fallback → erro graceful | [preencher] |
| Banco lento/indisponível | [preencher] | Timeout → health check falha → circuit breaker | [preencher] |
| Payload malformado | [preencher] | 400 com detalhes → sem crash → sem stack log | [preencher] |
| Payload muito grande | [preencher] | 413 antes de processar → limite middleware | [preencher] |
| Request concorrente | [preencher] | Optimistic locking/mutex → 409 | [preencher] |
| Token expirado | [preencher] | 401 clara → frontend refresh silencioso | [preencher] |

---

### Seção 7 — Banco de Dados

**7.1 Schema e Models:**

| Tabela | Model | Campos obrigatórios | Índices | FKs | Soft delete | Timestamps | Status |
|---|---|---|---|---|---|---|---|

**7.2 Migrations:**

| Migration | Operação | Reversível | Consistente schema | Data migration | Status |
|---|---|---|---|---|---|

- Estado atual (drift?), ordem execução (conflitos?)

- Zero-downtime migrations: backward-compatible? rollback testado? migration lock timeout?

**7.3 Queries e Performance:**

| Query | Localização | Índice usado | N+1 | Parametrizada | Custo estimado | Status |
|---|---|---|---|---|---|---|

**7.4 Dados sensíveis:**

| Campo | Tabela | Tipo (PII/financeiro/credencial) | Criptografia at rest | Mascaramento | Retenção | Status |
|---|---|---|---|---|---|---|

**7.5 Backup:** automático, frequência, retenção, restore testado, PITR, disaster recovery

**7.6 Connection Pooling:**
- Engine, pool size, connection timeout, idle timeout, leak detection
- Multi-tenant: pool por tenant ou compartilhado
- Métricas (sem monitoramento = P2)

---

### Seção 8 — Infraestrutura & DevOps

**8.1 Ambientes:**

| Ambiente | URL/Host | Paridade prod | Dados | Acesso restrito | SSL/TLS | Status |
|---|---|---|---|---|---|---|

**8.2 CI/CD Pipeline:**

| Stage | O que roda | Bloqueia deploy | Tempo médio | Status |
|---|---|---|---|---|
| Lint | [preencher] | [preencher] | [preencher] | [preencher] |
| Type check | [preencher] | [preencher] | [preencher] | [preencher] |
| Unit tests | [preencher] | [preencher] | [preencher] | [preencher] |
| Integration tests | [preencher] | [preencher] | [preencher] | [preencher] |
| Build | [preencher] | [preencher] | [preencher] | [preencher] |
| Security scan | [preencher] | [preencher] | [preencher] | [preencher] |
| Deploy | [preencher] | [preencher] | [preencher] | [preencher] |

- Stages ausentes (com prioridade), rollback (reversível? tempo? manual/automático?)

**8.3 Containerização:**
- Dockerfile (multi-stage? imagem segura?), Docker Compose, orquestração
- Health checks (liveness/readiness), auto-scaling
- Graceful shutdown: STOPSIGNAL, timeout (ECS stopTimeout, k8s terminationGracePeriodSeconds)

**8.4 Variáveis de Ambiente e Secrets:**

| Secret/Variável | Armazenamento | Rotação | .env.example | Nunca commitado | Status |
|---|---|---|---|---|---|

- Validação no startup (zod, joi, envalid) — sem = P1
- .env.example atualizado

**8.5 Code Quality:** linter, formatação, pre-commit hooks, EditorConfig, regras ignoradas. Sem linter = P2

**8.6 Git:** branch strategy, protection, commit conventions (sem = P3), PR template, .gitignore, sensíveis no histórico

**8.7 Documentação:** README (existe? atualizado?), diagrama arquitetura, onboarding (> 2h = P2), changelog, contributing. Sem README = P2

**8.8 Feature Flags:**
(Se não usa: "Não aplicável" + 2 frases → Seção 9. Deveria usar mas não usa: P2)

| Flag | Propósito | Tipo | Criada em | Owner | Status |
|---|---|---|---|---|---|

- Flags abandonadas (>30 dias 100% ON/OFF, >10 stale = P2), fallback indisponível (quebra = P1)
- Server vs client, auditoria de mudanças

---

### Seção 9 — Segurança (SEMPRE obrigatória)

**9.1 Autenticação:** método, storage token/session, refresh (rotação?), expiração, logout, brute force

- MFA / SSO (SAML · OIDC) / Passwordless: fluxo coberto? fallback? lockout policy?

**9.2 Autorização:** modelo (RBAC/ABAC/ACL), granularidade, verificação (inline = P2), escalação privilégio, admin bypass

**9.3 OWASP Top 10:**

| Vulnerabilidade | Proteção | Onde | Status |
|---|---|---|---|
| A01 — Broken Access Control | [preencher] | [preencher] | [preencher] |
| A02 — Cryptographic Failures | [preencher] | [preencher] | [preencher] |
| A03 — Injection | [preencher] | [preencher] | [preencher] |
| A04 — Insecure Design | [preencher] | [preencher] | [preencher] |
| A05 — Security Misconfiguration | [preencher] | [preencher] | [preencher] |
| A06 — Vulnerable Components | [preencher] | [preencher] | [preencher] |
| A07 — Auth Failures | [preencher] | [preencher] | [preencher] |
| A08 — Software/Data Integrity | [preencher] | [preencher] | [preencher] |
| A09 — Logging & Monitoring | [preencher] | [preencher] | [preencher] |
| A10 — SSRF | [preencher] | [preencher] | [preencher] |

**9.4 Headers HTTP:**

| Header | Esperado | Atual | Status |
|---|---|---|---|
| Content-Security-Policy | Restritivo, sem unsafe-inline | [preencher] | [preencher] |
| X-Frame-Options | DENY ou SAMEORIGIN | [preencher] | [preencher] |
| X-Content-Type-Options | nosniff | [preencher] | [preencher] |
| Strict-Transport-Security | max-age=31536000; includeSubDomains | [preencher] | [preencher] |
| Referrer-Policy | strict-origin-when-cross-origin | [preencher] | [preencher] |
| Permissions-Policy | Restritivo | [preencher] | [preencher] |
| CORS | Origins allowlisted — nunca * | [preencher] | [preencher] |

**9.5 Dependências:** audit (npm/pip), CVEs, lock file, abandonadas (>2 anos), não usadas

**9.6 Compliance LGPD/GDPR:**
(Se não processa PII: "Não aplicável" + justificativa)

- Consentimento, direito esquecimento, retenção (sem política = P1), portabilidade, minimização
- DPA sub-processadores, cookie consent (sem banner = P1), anonimização não-prod (sem = P1)
- Logs com PII sem retenção = P1

| Requisito | Aplicável | Implementado | Onde/Como | Status |
|---|---|---|---|---|
| Consentimento coleta | [preencher] | [preencher] | [preencher] | [preencher] |
| Direito esquecimento | [preencher] | [preencher] | [preencher] | [preencher] |
| Portabilidade | [preencher] | [preencher] | [preencher] | [preencher] |
| Minimização | [preencher] | [preencher] | [preencher] | [preencher] |
| Cookie consent | [preencher] | [preencher] | [preencher] | [preencher] |
| Anonimização não-prod | [preencher] | [preencher] | [preencher] | [preencher] |
| Retenção logs PII | [preencher] | [preencher] | [preencher] | [preencher] |

---

### Seção 10 — Cobertura de Regras de Negócio (RN → Código)

Obrigatória quando existe documento de RN. Se não existe RN formal → "Não aplicável" + justificativa (2 frases).
Ler 100% do RN e cruzar CADA regra, fluxo, tela, componente, campo, botão, modal, drawer, validação, cálculo, permissão, integração e notificação com a implementação real no código. Fidelidade spec → código = 100%.

**10.1 Entrada:** caminho do RN, versão, total de itens mapeados
**10.2 Matriz telas/UI:** tabela [Tela/Componente RN | ID regra | Implementado | Arquivo | Fidelidade (100%|parcial|ausente) | Desvios | Status]
- Verificar: campos, botões, ícones, labels, modais, drawers, popovers, estados (loading/empty/error/success/disabled), permissões por perfil, ações condicionais, ordem/agrupamento
**10.3 Matriz fluxos/regras:** tabela [Regra/Fluxo RN | ID | Tipo (validação|fluxo|cálculo|permissão|integração|estado|notificação) | Implementado | Localização (arquivo, função) | Fidelidade | Desvios | Status]
- Verificar: validações front+back, navegação, cálculos exatos, permissões, transições de estado + side-effects, integrações, notificações (trigger+template+destinatário)
**10.4 Matriz rotas/endpoints:** tabela [Funcionalidade RN | Rota esperada | Rota existente | Match (✅|⚠️ parcial|❌ ausente) | Desvios | Status]
**10.5 Itens não implementados:** tabela [Item RN | ID | Tipo (tela|fluxo|validação|campo|botão|modal|drawer|rota|cálculo|permissão|integração|notificação|estado) | Impacto P0-P3 | Obs]
- P0: core ausente | P1: secundário afeta uso normal | P2: detalhe UI | P3: cosmético
**10.6 Itens com desvio:** tabela [Item RN | Comportamento RN | Comportamento código | Tipo desvio (funcional|visual|comportamental|performance|dados) | Impacto | Recomendação (alinhar ao RN|manter código|atualizar RN)]
- RN = fonte da verdade. "Atualizar RN" só se implementação objetivamente melhor, RN desatualizado, ou alinhar quebraria outra funcionalidade
**10.7 Resumo:** total itens, % fidelidade 100%, % desvio, % ausente, cobertura geral %, gap P0/P1, esforço para 100%
- 95-100% Excelente | 85-94% Bom | 70-84% Aceitável | 50-69% Problemático | <50% Crítico

---

### Seção 11 — Qualidade de Código, SLOs e Testes Avançados
Complementa as verificações técnicas das Seções 4-9 com métricas objetivas, observabilidade orientada a negócio e camadas avançadas de testes.

**11.1 SLOs / SLIs / Error Budgets:**
- SLOs definidos e documentados? SLIs monitorados (latência p50/p95/p99, error rate, availability)?
- Error budget definido? Política de esgotamento? Burn rate alerts multi-window?
- Dashboards de SLO acessíveis ao time? Revisão periódica a cada quarter/semestre?
- Produção com usuários reais sem SLOs = P1

**11.2 Métricas de código estáticas:**
- Ferramenta: SonarQube | CodeClimate | ESLint complexity | Codacy | nenhuma
- Complexidade ciclomática (funções > 15) e cognitiva (funções > 20)? Duplicação > 30 linhas?
- Acoplamento entre módulos, maintainability index, debt ratio? Tendência nos últimos 3 meses?
- Nenhuma ferramenta de análise estática = P2

**11.3 Testes E2E e Visual Regression:**
- Framework E2E (Playwright | Cypress | Selenium)? Fluxos críticos cobertos? CI integration?
- Visual regression (Chromatic | Percy | BackstopJS)? Ausência em frontend rico = P2
- Cross-browser (Chromium, Firefox, WebKit)? Tempo de execução da suíte?

**11.4 Mutation Testing:**
- Ferramenta (Stryker | PITest | mutmut)? Mutation score < 60% = testes fracos mesmo com alta cobertura
- Priorizar módulos de regra de negócio crítica (cálculos, validações, permissões)
- Cobertura > 70% sem mutation testing = P3

**11.5 Contract Testing dedicado:**
- Ferramenta (Pact | Dredd | Spectral | openapi-diff)? Consumer-driven contracts?
- Verificação automática no CI? Breaking change detection? Mock consistency com spec real?
- Repos/deploys separados sem contract testing = P2

**11.6 Architecture Decision Records (ADRs):**
- ADRs existem (Markdown no repo | Notion | wiki)? Formato consistente (Contexto, Decisão, Alternativas, Consequências)?
- Decisões arquiteturais sem ADR correspondente? Mais de 3 decisões sem ADR = P2

**11.7 Supply Chain Security avançada:**
- SBOM gerado (SPDX, CycloneDX)? Dados regulados sem SBOM = P1
- Assinatura de artefatos (cosign, sigstore)? Ausência em produção = P2
- Dependency confusion protection? CI/CD com hashes fixos vs. tags mutáveis? Renovate/Dependabot? Provenance SLSA?

**11.8 Gestão de Incidentes:**
- Postmortem: documentado, blameless, template padronizado? On-call rotation (PagerDuty, Opsgenie)?
- Escalation matrix por severidade? Runbooks para cenários comuns (DB down, alta latência, deploy com bug)?
- Communication plan? SLA de resposta por severidade?
- Produção sem gestão de incidentes = P1

---

### Seção 12 — Disaster Recovery, FinOps e Release Strategy
Camada operacional e financeira para sustentação do projeto em produção a longo prazo.

**12.1 Disaster Recovery e Business Continuity:**
- RTO e RPO definidos, documentados e realistas? Failover: banco (réplica/Multi-AZ), app (multi-região), cache (sentinels/cluster)?
- DR testing realizado? Plano não testado = P1. Runbook de DR acessível offline?
- Produção sem plano de DR = P1. Região/AZ única = risco no Resumo Executivo

**12.2 FinOps / Otimização de Custos:**
- Custo mensal total e breakdown por serviço? Custo per request calculado e monitorado?
- Instâncias super-dimensionadas (utilização < 30%)? Reserved Instances/Savings Plans para workloads estáveis?
- Lifecycle policies em storage? Alertas de billing (ausência = P2)? Ambientes não-prod 24/7 sem uso?

**12.3 Release Strategy avançada:**
- Estratégia atual: big bang | rolling | blue-green | canary | progressive rollout
- Canary/blue-green configurado? Smoke tests pós-deploy automáticos? Rollback automático baseado em métricas?
- Deploy freezes (sexta à noite, feriados)? Feature flags + deploy separados?
- Deploy 100% sem canary/blue-green e sem smoke test = P2

**12.4 Chaos Engineering / Resilience Testing:**
- Ferramenta (Chaos Monkey | Litmus | Gremlin | AWS FIS)? Cenários: kill de instância, latência de rede, falha de dependência, disk full, CPU stress
- Game days periódicos documentados? Steady-state hypothesis definida antes de cada experimento?
- SLA > 99.5% sem chaos engineering = P2. Projeto pequeno (≤ 10 endpoints) ou não-prod = NA

**12.5 API Governance para consumidores externos:**
- Developer portal com docs interativa? Sandbox para terceiros? SDK oficial?
- Rate limiting transparente com headers (X-RateLimit-*)? Changelog de breaking changes com antecedência?
- Deprecation policy com header Sunset? API status page? Onboarding > 30 min = P2
- API apenas interna = NA

**12.6 Data Lineage e Data Flow:**
- Diagrama de data flow existe e atualizado? Fontes, transformações e destinos de dados mapeados?
- PII flow mapeado (consistente com Seção 9.6)? Data retention definida por destino?
- Mais de 3 fontes de dados sem diagrama = P2

**12.7 Developer Experience (DX) quantificada:**
- Time to first PR > 1 dia = P2. Setup local > 2 horas = P2
- Hot reload funciona? CI feedback > 15 min = P3. Steps manuais para deploy > 0 = P3
- "Funciona na minha máquina" frequente = P2

**12.8 Dependency Health Score:**
- Score por dependência: última release, frequência, issues abertas, CVEs, bus factor do mantenedor
- Top 10 dependências com score baixo? Plano de migração? Alternativas viáveis?
- Monitoramento contínuo (Snyk, Socket.dev, deps.dev)?
- Dependência crítica (auth, ORM, framework) com score baixo = P1. Demais = P2

---

### Seção 13 — Resumo Executivo (máximo 30 linhas. Use formato de tabela compacta para os campos numéricos (total de regras, implementadas, parciais, ausentes, conformidade stack, P0/P1/P2/P3) e bullets curtos para os demais. Se exceder 30 linhas, priorize: métricas > veredicto > top 3 divergências > demais)

- Saúde geral (1-10 + justificativa)
- Quantos endpoints, models/tabelas, componentes auditados
- Cobertura RN: % geral e gaps P0/P1 (se Seção 10 aplicável)
- Quantos P0, P1, P2, P3
- Quantas suposições e decisões autônomas
- Top 3 riscos + Top 3 pontos fortes
- Gaps regulatórios (LGPD/GDPR)
- Esforço estimado P0 (dias-dev)
- Veredicto: "Saudável para produção" | "Quase saudável, resolver P0" | "Refatoração parcial" | "Refatoração estrutural" | "Risco crítico — pausar deploys"
  (Se há P0 aberto: veredicto NUNCA = "Saudável para produção")

---

## 5. REGRAS DE QUALIDADE DO OUTPUT

**Idioma:** pt-BR. Nomes de código mantidos como no original. Termos técnicos em inglês.
**Numeração:** usar Seção 1 a Seção 13 (numeração do output). Nunca numeração do prompt.

**Conflito entre seções:** finding em 2+ seções → seção primária (causa raiz) com profundidade + cross-ref na secundária: "Ver Seção X, Item #Y — [descrição]"

**Contagem mínima:** verificações com 0 itens → declarar resultado + método. Ex: "0 TODOs — grep -rn 'TODO|FIXME|HACK'"

**Tom:** cada célula de tabela ≥ 1 frase completa (nunca apenas N/A). Cada item acionável. Nunca "etc.", "entre outros", "quando aplicável". Nunca linguagem vaga. Referências com número exato.

**Terminologia:** definir na Seção 1, usar idêntica em todo documento. Nomes sempre mesmo casing.

**Tabelas:** nenhuma célula vazia (N/A → "Não aplicável: [motivo]"). 100% dos elementos. Seção 6 com paridade 1:1 das rotas da Seção 5.2.

**Verificação cruzada obrigatória (antes de entregar):**
- Seção 2 → 3: risco → solução com prioridade
- Seção 4 → 6: fetch frontend ↔ endpoint API
- Seção 5.2 → 6.1: controller ↔ endpoint
- Seção 6 → 9: auth endpoint → mecanismo 9.1; público → rate limiting
- Seção 7.1 → 7.2: schema ↔ migrations sem drift
- Seção 7.3 → 7.1: query com filtro → índice
- Seção 5.5 → 8: job → monitoring
- Seção 9.5 → 3: CVE crítico → ação P0/P1
- Testes → P0/P1: lógica/segurança sem teste = finding adicional
- Suposições ≠ Decisões (nunca simultâneo)
- Seção 4.7 → 4.2: coluna a11y preenchida
- Seção 4.9 → 4.5: i18n → validações com formatos localizados
- Seção 5.9 → 9: templates anti-injection, SPF/DKIM/DMARC
- Seção 9.6 → 7.4: PII → retenção/esquecimento
- Regra 2.9 → 5.5: jobs recursos compartilhados → locking
- Seção 7.6 → 5.10: graceful shutdown → fecha pool
- Seção 8.8 → 8.7: flags → documentação (sem = P3)
- Seção 5.7 → 5.5: webhooks incoming via fila → job correspondente
- Seção 6.2.1 → 5.5: jobs retry → idempotency keys
- Seção 10.5 → 3: itens RN não implementados → ação com prioridade
- Seção 11.1 → 2: SLOs/métricas → nota de saúde geral
- Seção 12.1 → 3: DR/FinOps findings → ação com prioridade

---

## 6. INSTRUÇÃO DE EXECUÇÃO

**Guardrail de tamanho:**
- Microserviço (até 10 endpoints): 4-8 páginas
- App CRUD (10-30 endpoints): 10-15 páginas
- App complexa (30-80 endpoints): 15-25 páginas
- Monolito massivo (80+ endpoints): 25-35 páginas (dividir por módulo)
- Se > 35 páginas: sinalizar no Resumo

**REGRAS:**
1. Ordem das seções (1 → 13), sem pular, inverter ou fundir
2. Seção N/A: "Não aplicável" + justificativa — nunca omitir
3. Não resuma, não generalize — audite cada item do código/documentação
4. Nomes reais: arquivos, funções, rotas, tabelas (não genéricos)
5. Verificação cruzada antes de finalizar
6. Regra de corte: se < 70% confiança → pare após Seção 2
7. Após relatório: implementação imediata P0 → P1 → P2 → P3, registrando cada alteração (data, hora, arquivo)
8. Não aguarde validação — todas as correções pré-aprovadas. Implemente e registre

**Commits:** Use Conventional Commits: `fix(modulo): auditoria fullstack — PX corrigidos [descrição]`. Commite por prioridade (P0 primeiro, depois P1, etc.).
```

---

## Tracking de Progresso

> **⚡ CRIE ESTE TODO IMEDIATAMENTE AO INICIAR — antes de qualquer leitura ou análise**
>
> Use `TodoWrite` para criar todas as fases e tarefas listadas abaixo antes de executar qualquer ação. À medida que concluir cada tarefa, marque-a como `done` imediatamente — antes de avançar para a próxima. Nunca execute uma tarefa sem ela estar na lista. Nunca avance sem marcar a tarefa anterior como concluída. O feedback em tempo real é parte obrigatória do protocolo.

Use TodoWrite:
1. Seção 1 — Frontend
2. Seção 2 — Backend e API
3. Seção 3 — Banco de Dados
4. Seção 4 — Infraestrutura e Deploy
5. Seção 5 — Segurança
6. Seção 6 — Qualidade de Código
7. Seção 7 — Performance
8. Seção 8 — Observabilidade
9. Seção 9 — Segurança (SEMPRE obrigatória)
10. Seção 10 — Re-auditoria / Modo Delta
11. Seção 11 — Output e Relatório Final
12. Seção 12 — Verificação Cruzada
13. Seção 13 — Implementação de Correções P0 → P1 → P2 → P3

---

## Glossário

| Termo | Definição |
| --- | --- |
| P0-Critico | Vulnerabilidade de segurança, perda de dados, crash em produção |
| P1-Alto | Funcionalidade quebrada, performance degradada, UX bloqueada |
| P2-Medio | Inconsistência técnica, code smell, padrão violado |
| P3-Baixo | Melhoria de legibilidade, otimização menor, documentação |
| Health Score | Nota 1-10 calculada pela rubrica de saúde do projeto |
| Zero-residual | Regra que exige 100% dos findings resolvidos antes de finalizar |
| Re-auditoria | Modo delta que audita apenas mudanças desde a última auditoria |

## Changelog

| Versão | Data | Alterações |
| --- | --- | --- |
| v2.4 | 22/03/2026 | Adição de OUTPUT_PATH, estratégia de git commit, glossário e changelog. |