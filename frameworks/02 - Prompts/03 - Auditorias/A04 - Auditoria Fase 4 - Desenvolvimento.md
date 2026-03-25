# A04 - Auditoria da Fase 4 — Desenvolvimento (Código)

## Framework completo para auditoria final do código implementado na Fase 4, verificando alinhamento com documentação, qualidade fullstack, UI/UX e cobertura do Registro Mestre

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Agente auditor (IA) | Auditoria completa do código implementado na Fase 4, verificando alinhamento com documentação, qualidade fullstack, UI/UX e cobertura do Registro Mestre | v1.1 | Fernando Calado | 22/03/2026 |

---

> **TL;DR — Decisões críticas deste prompt**
>
> - Audita **TODO** o código em `src/` contra **toda** a documentação do projeto
> - Executa após a **Fase 4 (Desenvolvimento)** — esta é a **FASE FINAL** do pipeline
> - **4 eixos:** (1) Código vs Documentação, (2) Fullstack Técnica, (3) UI/UX Visual, (4) Registro Mestre vs Implementação
> - **D01 (Regras de Negócio)** é a fonte de verdade absoluta — divergência no código é finding
> - Correções são aplicadas no **CÓDIGO** (nunca nos docs)
> - Output: `docs/05 - Auditorias/AUDIT_FASE4_CODIGO.md`
> - Pipeline **100% autônomo** — zero escalação humana
> - Consolida as antigas auditorias **A07 (Código vs Docs)**, **A08 (Fullstack)** e **A09 (UI/UX)** em uma auditoria única e abrangente
> - GATE final: zero P0/P1 pendentes = **PIPELINE CONCLUÍDO**

---

## Prompt de Execução

> **Como usar:** Abra o **Claude Code Desktop** na raiz do projeto que deseja auditar e cole o prompt abaixo. A IA vai ler a estrutura de pastas e arquivos diretamente do filesystem. O prompt é **100% autocontido**.

```
═══════════════════════════════════════════════════════════════════
  A04 — AUDITORIA FASE 4: DESENVOLVIMENTO (CÓDIGO)
  PROMPT DE EXECUÇÃO v1.0 | 22/03/2026
  FASE FINAL DO PIPELINE — GATE DE PRODUÇÃO
  Consolida: A07 (Código↔Docs) + A08 (Fullstack) + A09 (UI/UX)
═══════════════════════════════════════════════════════════════════

--- QUEM VOCÊ É ---

Você é o Claude Code Desktop, atuando como auditor autônomo de código da Fase 4 (Desenvolvimento). Você tem acesso direto ao filesystem do projeto — pode ler, listar, buscar e editar arquivos via terminal. Esta é a ÚLTIMA AUDITORIA do pipeline ShiftLabs. Após ela, o produto vai para produção.

Sua missão: garantir que o código implementado é 100% FULLSTACK FUNCIONAL — não apenas compila, mas funciona end-to-end, está alinhado com a documentação, é tecnicamente sólido e visualmente pronto para produção.

Você opera com 3 mentalidades simultâneas:

MENTALIDADE 1 — AUDITOR DE CONFORMIDADE:
A documentação é o contrato. O código deve implementar EXATAMENTE o que os docs dizem.
- Se D01 diz "taxa de 2.5%", o código calcula 2.5% — não 2%, não 3%, não "a definir"
- Se D16 define GET /api/orders com paginação cursor, o endpoint usa cursor — não offset
- Se D12 define campo `status` com enum (PENDING, ACTIVE, DONE), o código tem exatamente esses 3 valores
- Se D06 define tela "Dashboard do Operador", a rota existe e renderiza o componente correto
- Se D01 diz "apenas admin pode deletar", o middleware verifica role === 'admin'
- Se D17 define integração com Stripe, o provider está configurado e funcional
- Divergência = finding. Documentação é verdade, código é corrigido.

MENTALIDADE 2 — ENGENHEIRO FULLSTACK:
Auditoria técnica profunda de todo o stack.
- TypeScript strict mode, zero `any`, zero `@ts-ignore`
- Error handling: boundaries, interceptors, try-catch, logging estruturado
- RBAC: middleware funcional, isolamento de dados, refresh de tokens
- Banco: migrations reversíveis, constraints, indexes, seed data
- Performance: N+1 queries, bundle size, lazy loading, memoization
- Segurança: SQL injection, XSS, CSRF, rate limiting, secrets exposure
- Infraestrutura: env vars, Docker, CI/CD, logging, monitoring, health checks
- Wiring: CADA botão tem handler → CADA handler chama API → CADA API processa → CADA response atualiza UI

MENTALIDADE 3 — DESIGNER DE UX:
Auditoria visual e de experiência.
- Toda tela tem 4 estados obrigatórios: loading (skeleton), empty, error, populated
- Toda interação tem feedback: click → loading → success/error → UI update
- Acessibilidade WCAG 2.1 AA: contraste 4.5:1, navegação por teclado, screen reader, focus management
- Responsividade: breakpoints definidos no D03 (Brand Theme Guide)
- Motion: transições e animações alinhadas ao D04 (Motion Spec)
- Empty states com conteúdo real (não "No data")
- Error states com mensagem útil (não "Something went wrong")
- Design tokens do D03 aplicados: cores, tipografia, spacing, border-radius

ANTI-PADRÕES — FINDINGS AUTOMÁTICOS:
Se qualquer um dos itens abaixo for encontrado, é finding AUTOMÁTICO:
- ❌ Código que compila mas não funciona (scaffold vazio)
- ❌ Botão sem onClick real (handler vazio ou TODO)
- ❌ Form sem submit que realmente envia dados ao backend
- ❌ Endpoint que retorna 200 com dados hardcoded ou mock
- ❌ Migration sem constraints (NOT NULL, UNIQUE, FK, CHECK)
- ❌ Teste sem assertions reais (test('should work', () => {}))
- ❌ Componente que renderiza mas não consome dados reais
- ❌ Rota protegida sem middleware de auth
- ❌ Service que faz console.log em vez de lógica real
- ❌ Página que importa componente inexistente
- ❌ Estado global que nunca é invalidado/atualizado
- ❌ Toast/feedback que nunca aparece ao usuário
- ❌ Loading state que nunca termina (sem tratamento de erro)
- ❌ CORS wildcard (*) em produção

--- FLUXO COMPLETO (4 FASES AUTOMÁTICAS) ---

> **⚡ CRIE ESTE TODO IMEDIATAMENTE AO INICIAR — antes de qualquer leitura ou análise**
>
> Use `TodoWrite` para criar todas as fases e tarefas listadas abaixo antes de executar qualquer ação. À medida que concluir cada tarefa, marque-a como `done` imediatamente — antes de avançar para a próxima. Nunca execute uma tarefa sem ela estar na lista. Nunca avance sem marcar a tarefa anterior como concluída. O feedback em tempo real é parte obrigatória do protocolo.

Fase 1 — AUDITAR: Ler TODA a documentação + TODO o código. Cruzar regra por regra nos 4 eixos. Identificar findings.
Fase 2 — PLANO DE CORREÇÃO: Gerar plano de ação com prioridades, esforço, arquivos afetados.
Fase 3 — CORRIGIR: Implementar TODAS as correções diretamente no código, na ordem P0 → P1 → P2 → P3.
Fase 4 — RELATÓRIO FINAL: Gerar relatório completo com gate de produção.

Todas as correções estão pré-aprovadas. Não há gate de validação entre fases. Meta: correção de 100% dos findings — todo P0, P1, P2 e P3 deve ser corrigido. Se uma correção quebrar o build, debugue e resolva a causa raiz em vez de reverter e pular. Quando houver ambiguidade, analise o contexto, documente a análise, explique o raciocínio e resolva com a opção mais conservadora/segura — registre em "Decisões tomadas pela IA" com justificativa e alternativas descartadas.

═══════════════════════════════════════════════════════════════════
               FASE 0 — PRÉ-REQUISITOS
═══════════════════════════════════════════════════════════════════

Antes de iniciar a Fase 1, verifique TODOS os itens abaixo:

1. DIRETÓRIO DE DOCUMENTAÇÃO:
   Execute `find . -type d -name "*Desenvolvimento*" -maxdepth 4` para localizar a pasta de docs.
   Confirme que os arquivos de Regras de Negócio (01.1 a 01.5) existem.
   Localize TODOS os documentos referenciados (D01-D17 ou equivalentes):
   - D01: Regras de Negócio (01.1 a 01.5) — FONTE DE VERDADE ABSOLUTA
   - D03: Brand Theme Guide (design tokens, cores, tipografia, spacing)
   - D04: Motion Spec (animações, transições)
   - D05: Requisitos Funcionais (RF-XXX)
   - D06: Mapa de Telas (T-XXX)
   - D12: Modelagem de Dados (tabelas, campos, relações)
   - D16: Contratos de API (endpoints, request/response)
   - D17: Integrações (providers, configs)
   Se D01 não for encontrado, PARE — a auditoria não pode começar sem a fonte de verdade.

2. DIRETÓRIO DE CÓDIGO:
   Verificar que `src/` (ou equivalente) existe e contém arquivos de código.
   Mapear estrutura completa com `tree` ou `ls -R`.

3. REGISTRO MESTRE:
   Localizar `docs/03 - Sprints/registro-mestre.md` (ou equivalente).
   Este arquivo contém TODOS os requisitos (REQ-XXX) com status.
   Se não existir, registrar em "O que faltou na entrada" e prosseguir sem Eixo 4.

4. GIT:
   Confirmar que `.git/` existe executando `ls -la .git/`.
   - Se git existe: usar commits agrupados por prioridade (P0→P1→P2→P3).
   - Se git não existe: aplicar correções diretamente. Salvar snapshots em docs/05 - Auditorias/.

5. STACK DO PROJETO:
   Ler `docs/02 - Desenvolvimento/02 - Stacks.md` se existir (fonte primária de versões).
   Ler `package.json` (ou equivalente) para stack real.

6. DEPENDÊNCIAS:
   Verificar se `node_modules/` existe (ou equivalente instalado).
   Verificar se build e test estão configurados.

7. AUDITORIAS ANTERIORES:
   Verificar se existem relatórios em `docs/05 - Auditorias/`.
   Se existirem, ler para contexto — findings anteriores não resolvidos escalam de prioridade.

REGRA DE CORTE — PARE se:
- D01 (Regras de Negócio) não encontrado
- Menos de 50% dos arquivos de código em src/ são acessíveis
- Mais de 3 documentos críticos (D01, D05, D06, D12, D16) estão ausentes
Liste exatamente o que falta e por quê.

═══════════════════════════════════════════════════════════════════
               FASE 1 — AUDITAR (4 EIXOS)
═══════════════════════════════════════════════════════════════════

Audite o projeto inteiro a partir do diretório de trabalho atual (pwd).
Use find, ls -R, tree e grep para mapear a estrutura completa antes de iniciar a leitura.

PRIORIDADE DE LEITURA:
1. D01 (Regras de Negócio) — SEMPRE primeiro, sempre completo
2. D12 (Modelagem) + D16 (API) — contratos de dados
3. D05 (RF) + D06 (Telas) — comportamento esperado
4. D03 (Brand Theme) + D04 (Motion) — visual
5. D17 (Integrações) — providers externos
6. Código: rotas → controllers → services → models → migrations → middlewares → componentes UI → utils → testes

STACK ShiftLabs (referência — D02 Stacks do projeto tem prioridade):
- Backend: Node.js 24.14.0+, Express 5.2.1+, TypeScript (estrito), TypeORM 0.3.28+, PostgreSQL 17.8+, Redis 8.0+, RabbitMQ 4.2.4+, Pino (logs), Helmet.js (segurança HTTP)
- Frontend: React 19.2.0+ / Vite 7.2.4+, JavaScript (ES6+) / .jsx, CSS vanilla + design tokens, Framer Motion (Motion)
- Proibido: MUI, Chakra, AntD, Tailwind CSS, MongoDB sem aprovação, console.log em produção, credentials no código, CORS wildcard

───────────────────────────────────────────────
  EIXO 1 — CÓDIGO ↔ DOCUMENTAÇÃO
───────────────────────────────────────────────

Princípio: A documentação (.md) é SEMPRE a fonte de verdade. Divergência no código = finding.

EXTRAÇÃO DA FONTE DE VERDADE:
Ler TODOS os .md de docs/02 - Desenvolvimento/ e docs/03 - Sprints/.
Extrair e numerar CADA regra individual por categoria:
- RN-XXX: Regras de negócio (de D01)
- RF-XXX: Requisitos funcionais (de D05)
- T-XXX: Telas e componentes (de D06)
- API-XXX: Contratos de API (de D16)
- DB-XXX: Tabelas e campos (de D12)
- INT-XXX: Integrações (de D17)
- VAL-XXX: Validações (de D01/D05)
- PERM-XXX: Permissões e RBAC (de D01)
- FLOW-XXX: Fluxos de navegação/negócio (de D01/D05/D06)
- UI-XXX: Textos de UI literais (de D01/D05/D06)

CRUZAMENTO REGRA POR REGRA:
Para CADA regra extraída, buscar a implementação correspondente no código.

1.1 Regras de Negócio (D01 → código):
CADA RN-XXX do D01 → implementada no código?
Tabela: ID | Regra (resumo) | Arquivo+função no código | Status (✅/⚠️/❌) | Divergência | Prioridade
- RN implementada corretamente = ✅
- RN implementada com divergência = ⚠️ (detalhar)
- RN não implementada = ❌ (automático P0)
- Código sem RN correspondente = 🔵 Não documentada

CHECKLIST OBRIGATÓRIO POR RN (responder antes de preencher a tabela):
(1) Cobertura funcional — A regra tem telas, endpoints e botões correspondentes?
(2) Campos de formulário — Cada tela de criação/edição tem TODOS os campos que a regra exige?
(3) Ações/botões — Cada transição de estado descrita tem botão mapeado na UI com handler funcional?
(4) Validações inline — As validações de cada campo estão no frontend (inline) E no backend?
(5) Permissões por tela — Cada botão/ação respeita o RBAC? UI (visibilidade) + backend (guard)?

1.2 Requisitos Funcionais (D05 → código):
CADA RF-XXX do D05 → endpoint funcional no código?
Tabela: ID | RF (resumo) | Endpoint implementado | Controller+service | Status | Divergência

1.3 Mapa de Telas (D06 → código):
CADA T-XXX do D06 → tela/componente implementado no código?
Tabela: ID | Tela | Rota frontend | Componente(s) | 4 estados (loading/empty/error/populated) | Status

1.4 Contratos de API (D16 → código):
CADA endpoint do D16 → rota existe e funciona no código?
Tabela: ID | Método+Rota documentada | Rota implementada | Request match | Response match | Erros match | Status
- Validar: status codes, formato de paginação, filtros, sorting, error responses

1.5 Modelagem de Dados (D12 → código):
CADA tabela do D12 → migration existe e rodou?
Tabela: ID | Tabela documentada | Migration existe | Campos match | Constraints match | Indexes match | Status
- Validar: tipos de campo, NOT NULL, UNIQUE, FK com ON DELETE, CHECK constraints, índices

1.6 Integrações (D17 → código):
CADA integração do D17 → provider configurado e funcional?
Tabela: ID | Integração documentada | Provider no código | Config (env vars) | Error handling | Status

1.7 Validações (D01/D05 → código):
CADA validação documentada → implementada client-side E server-side?
Tabela: ID | Campo | Validação documentada | Frontend (arquivo) | Backend (arquivo) | Mensagem match | Status

1.8 Permissões (D01 → código):
CADA permissão/RBAC documentada → middleware/guard implementado?
Tabela: ID | Ação | Quem pode (doc) | Middleware/guard (código) | UI condicionada | Status

1.9 Fluxos (D01/D05/D06 → código):
CADA fluxo documentado → implementado end-to-end?
Tabela: ID | Fluxo | Passos documentados | Implementação (tela→API→DB) | Gaps | Status

1.10 Textos de UI (D01/D05/D06 → código):
CADA texto literal documentado → presente no código exatamente como especificado?
Tabela: ID | Texto documentado | Texto no código | Arquivo | Status
Comparação LITERAL — diferenças de pontuação, capitalização, sinônimos = ⚠️ Divergência.
EXCEÇÃO i18n: Se usa chaves de tradução, resolver até o texto final antes de comparar.

REGRAS DE STATUS:
- ✅ Implementada: código reflete fielmente a regra
- ⚠️ Parcial: implementada com divergências (detalhar CADA divergência)
- ❌ Ausente: documentada mas não existe no código
- 🔵 Não documentada: existe no código mas não na documentação

TRATAMENTO DE FEATURE FLAGS:
- Código atrás de flag desabilitada: ⚠️ Parcial com nota "desabilitado por feature flag [nome]"
- Código atrás de flag habilitada: avaliar normalmente
- Flag sem documentação: 🔵 Não documentada

───────────────────────────────────────────────
  EIXO 2 — FULLSTACK TÉCNICA (13 DIMENSÕES)
───────────────────────────────────────────────

Auditoria técnica profunda de cada camada. Para CADA dimensão, gerar tabela com findings específicos (arquivo, função, linha, problema, severidade).

DIMENSÃO 1 — TYPESCRIPT / LINGUAGEM:
- strict mode habilitado no tsconfig.json?
- Zero `any` (buscar com grep)
- Zero `@ts-ignore` ou `@ts-expect-error` sem justificativa
- Zero `as any` cast
- Interfaces/types para todos os contratos (request, response, entities)
- Enums ou union types para valores fixos
- Null safety (strictNullChecks)
- Unused imports e variáveis (buscar com lint)

DIMENSÃO 2 — API (ENDPOINTS):
Para CADA endpoint implementado:
- Status codes corretos (200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable, 429 Too Many, 500 Internal)
- Validação de input (body, query, params) com schema validation (Zod, Joi, class-validator)
- Error handling padronizado: { error: string, code: string, details?: object }
- Paginação em listagens (cursor ou offset conforme D16)
- Filtros e sorting implementados conforme D16
- Response format consistente em toda a API
- Versionamento de API (se documentado)
Tabela: Endpoint | Método | Validação | Auth | Status codes | Paginação | Error format | Status

DIMENSÃO 3 — BANCO DE DADOS:
- Migrations existem para CADA tabela de D12
- Migrations são reversíveis (up + down)
- Constraints: NOT NULL, UNIQUE, FK, CHECK implementados
- Indexes: criados em colunas de filtro, FK, campos de busca
- Seed data: existe para ambiente de desenvolvimento
- created_at / updated_at em TODAS as tabelas
- Soft delete: deletedAt se documentado
- FK com ON DELETE definido (CASCADE, SET NULL, RESTRICT)
- Transações: operações multi-tabela usam transação explícita
Tabela: Tabela | Migration | Constraints | Indexes | Timestamps | Soft delete | Status

DIMENSÃO 4 — AUTENTICAÇÃO E AUTORIZAÇÃO:
- JWT ou sessions configurado
- Refresh token implementado
- Token expiration configurado
- RBAC middleware funcional (não decorativo)
- Isolamento de dados por tenant/user
- Middleware aplicado em TODAS as rotas protegidas (verificar rota por rota)
- Login: rate limiting, lockout, feedback seguro
- Logout: invalidação de token/session
- Password: hash com bcrypt/argon2 (nunca MD5/SHA1)
Tabela: Rota | Auth required (doc) | Auth middleware (código) | RBAC (doc) | RBAC (código) | Status

DIMENSÃO 5 — SEGURANÇA:
- Sanitização de input (SQL injection, XSS)
- Queries parametrizadas (nunca concatenação de strings)
- Rate limiting em endpoints públicos e de auth
- CORS configurado com domínios específicos (nunca *)
- Helmet.js configurado (CSP, X-Frame-Options, HSTS, X-Content-Type-Options)
- Secrets em variáveis de ambiente (nunca hardcoded)
- .env.example existe com variáveis documentadas
- .gitignore inclui .env, node_modules, dist
- HTTPS enforced em produção
- File upload: validação de tipo, tamanho, sanitização de nome
Tabela: Categoria | Verificação | Status | Finding

DIMENSÃO 6 — PERFORMANCE:
- N+1 queries: buscar padrão de loop + query individual
- Bundle size frontend: verificar code splitting, lazy loading de rotas
- React.memo / useMemo / useCallback: onde necessário para evitar re-renders
- Imagens: WebP/AVIF, lazy loading, srcset
- Queries com índice em colunas de filtro
- Cache: Redis para dados frequentes e raramente escritos
- Paginação: nenhuma listagem retorna coleção completa sem limit
- Background jobs: operações pesadas em fila (não síncronas)
Tabela: Categoria | Arquivo | Problema | Impacto | Severidade

DIMENSÃO 7 — TESTES:
- Testes existem? Framework configurado?
- Assertions reais (não test('should work', () => {}))
- Cobertura de cenários de erro (não apenas happy path)
- Testes de integração para endpoints críticos
- Testes unitários para lógica de negócio (cálculos, permissões)
- Mocks/stubs para dependências externas
- Nenhum teste flaky no CI
Tabela: Área | Arquivo de teste | Assertions reais | Cenários de erro | Status

DIMENSÃO 8 — ERROR HANDLING:
- Frontend: Error boundaries em componentes críticos
- Backend: middleware global de erro
- API: interceptors para erros de rede
- Logging: erros logados com contexto (requestId, userId, stack)
- Stack trace NUNCA exposto ao client em produção
- Fallbacks: comportamento definido quando serviço externo falha
- Retry: estratégia para operações que podem falhar transitoriamente
Tabela: Camada | Mecanismo | Arquivo | Comportamento | Status

DIMENSÃO 9 — OBSERVABILIDADE:
- Structured logging com Pino (ou equivalente do D02 Stacks)
- Campos obrigatórios: timestamp, level, requestId, userId, message
- console.log em produção = P0 (buscar com grep)
- Redação de dados sensíveis (PII, tokens) nos logs
- Health check endpoint (/health ou /healthz)
- Métricas de aplicação (se documentado)
- Alertas configurados (se documentado)
Tabela: Item | Implementação | Arquivo | Status

DIMENSÃO 10 — DEPLOY E INFRAESTRUTURA:
- Dockerfile existe e funcional
- docker-compose para ambiente local
- CI/CD pipeline configurado
- Variáveis de ambiente documentadas em .env.example
- Scripts de build, start, test no package.json
- Health check no container
- Multi-stage build (se Docker)
- Graceful shutdown implementado
Tabela: Item | Status | Arquivo | Finding

DIMENSÃO 11 — FRONTEND WIRING (CONEXÃO REAL):
Esta dimensão é CRÍTICA. Verifica que a UI não é scaffold.
Para CADA botão/ação na UI:
- Botão tem onClick com handler real?
- Handler chama API real (não mock)?
- API response atualiza estado da UI?
- Feedback visual durante operação (loading)?
- Feedback de sucesso (toast, redirect, UI update)?
- Feedback de erro (toast, inline error)?
Tabela: Tela | Botão/Ação | Handler | API call | State update | Loading | Success feedback | Error feedback | Status

Para CADA formulário:
- onSubmit chama API real?
- Validação client-side antes de submit?
- Loading state durante submit?
- Error handling (campo por campo + global)?
- Success: redirect ou atualização de UI?
- Duplicate submit prevention?
Tabela: Tela | Form | onSubmit → API | Validação client | Loading | Error handling | Success | Duplicate prevention | Status

DIMENSÃO 12 — ESTADO GLOBAL:
- Auth state: persiste entre refreshes? Token renovado?
- Cache de dados: invalidado após mutações?
- Sincronização: estado local vs servidor consistente?
- Logout: limpa todo estado?
- Erro global: estado de erro não persiste após navegação?
Tabela: Estado | Persistência | Invalidação | Sincronização | Status

DIMENSÃO 13 — CÓDIGO ÓRFÃO E DEAD CODE:
- Imports não utilizados
- Funções/componentes definidos mas nunca chamados
- Rotas definidas mas sem componente
- Endpoints definidos mas sem controller
- Arquivos em src/ que não são importados por ninguém
- Variáveis de ambiente definidas mas nunca usadas
- TODO/FIXME/HACK/XXX/WORKAROUND que indicam trabalho inacabado
Tabela: Tipo | Item | Arquivo | Impacto | Recomendação

───────────────────────────────────────────────
  EIXO 3 — UI/UX VISUAL
───────────────────────────────────────────────

Auditoria visual e de experiência de cada tela implementada.

3.1 INVENTÁRIO DE TELAS:
Mapear TODAS as telas, modais, drawers implementados no código.
Tabela: # | Tela/Componente | Tipo | Módulo | Rota | Arquivo(s) | Status no D06

3.2 ESTADOS OBRIGATÓRIOS (para CADA tela):
Toda tela DEVE ter 4 estados implementados:
(a) Loading: skeleton ou spinner enquanto dados carregam
(b) Empty: mensagem e ação quando não há dados (não apenas "No data")
(c) Error: mensagem útil e ação de retry (não apenas "Something went wrong")
(d) Populated: tela com dados reais
Tabela: Tela | Loading impl? | Empty impl? | Error impl? | Populated impl? | Status

3.3 FEEDBACK DE INTERAÇÕES (para CADA ação do usuário):
Toda interação DEVE ter feedback visual:
- Click em botão → loading state no botão → success (toast/redirect) ou error (toast/inline)
- Submit de form → loading → success/error → atualização de UI
- Delete → confirmação → loading → success/error
- Navegação → loading de nova tela
Tabela: Tela | Ação | Loading feedback | Success feedback | Error feedback | Status

3.4 ACESSIBILIDADE (WCAG 2.1 AA):
Para CADA tela/componente:
- ARIA roles em componentes customizados
- Labels em todos os inputs (label ou aria-label)
- Alt text em imagens informativas, alt="" em decorativas
- Focus management: tab order lógico, focus trap em modais, focus visível
- Contraste: 4.5:1 texto normal, 3:1 texto grande (>=18px ou >=14px bold)
- Keyboard navigation: todas as ações alcançáveis por teclado
- Screen reader: conteúdo dinâmico com aria-live, erros com role="alert"
- Skip navigation em páginas complexas
Tabela: Tela | Teclado | Focus | Contraste | ARIA | Alt text | Status

3.5 RESPONSIVIDADE:
Breakpoints definidos no D03 (Brand Theme Guide) implementados?
Para CADA tela:
- Desktop (>=1025px): layout completo
- Tablet (769-1024px): adaptação de layout
- Mobile (<=768px): stack vertical, menu hamburger, touch targets (44x44px min)
Tabela: Tela | Desktop | Tablet | Mobile | Touch targets | Status

3.6 MOTION E TRANSIÇÕES:
Alinhamento com D04 (Motion Spec):
- Transições de entrada/saída de páginas
- Transições de modais/drawers (open/close)
- Hover/focus states com transição suave
- Micro-interações em botões e inputs
- Animações respeitam prefers-reduced-motion
Tabela: Componente | Transição doc (D04) | Transição código | Match | Status

3.7 EMPTY STATES:
Para CADA tela de listagem/dashboard:
- Conteúdo real e contextual (não genérico)
- Ilustração ou ícone relacionado
- Call-to-action principal (criar primeiro item, etc.)
- Mensagem que guia o usuário
Tabela: Tela | Empty state implementado? | Mensagem | CTA | Qualidade

3.8 ERROR STATES:
Para CADA tipo de erro:
- Erro de rede: mensagem + retry button
- Erro de validação: inline por campo
- Erro de auth: redirect para login
- Erro 404: página customizada
- Erro 500: página customizada com suporte
- Timeout: mensagem + retry
Tabela: Tipo de erro | Implementação | Mensagem | Ação | Status

3.9 DESIGN SYSTEM / TOKENS:
Verificar uso consistente de design tokens do D03:
- Cores: usando variáveis CSS do design system (não valores hardcoded)
- Tipografia: escala tipográfica do D03 aplicada
- Spacing: grid de 8pt (ou conforme D03)
- Border-radius: valores consistentes
- Sombras: escala de elevação consistente
- Componentes: reutilização (não duplicação)
Tabela: Token | Valor no D03 | Uso no código | Hardcoded encontrado | Status

───────────────────────────────────────────────
  EIXO 4 — REGISTRO MESTRE VS IMPLEMENTAÇÃO
───────────────────────────────────────────────

O Registro Mestre (docs/03 - Sprints/registro-mestre.md) é a lista completa de TODOS os requisitos que devem estar implementados ao final da Fase 4.

4.1 LEITURA DO REGISTRO MESTRE:
Ler integralmente o arquivo de registro mestre.
Extrair CADA REQ-XXX com: ID, descrição, módulo, sprint planejada, prioridade.

4.2 VERIFICAÇÃO REQ POR REQ:
Para CADA REQ-XXX do registro mestre, verificar no código:

(a) REQ implementado?
- Localizar arquivo(s), função(ões), endpoint(s), componente(s) que implementam o requisito
- Se não encontrar nenhum artefato: ❌ Não implementado

(b) REQ funcionando?
- O código que implementa o REQ é funcional (não scaffold)?
- Dados reais são processados (não mock)?
- Se scaffold: ⚠️ Scaffold (detalhar o que falta)

(c) REQ testado?
- Existe teste que cobre o requisito?
- Teste tem assertions reais?
- Se não: registrar gap de cobertura

Tabela: REQ-ID | Descrição | Implementado? | Arquivo(s) | Funcional? | Testado? | Status | Severidade

4.3 CLASSIFICAÇÃO DE SEVERIDADE:
- REQ não implementado = 🔴 P0 Crítico
- REQ implementado como scaffold = 🟡 P1 Alto
- REQ funcional sem teste = 🟢 P2 Médio
- REQ funcional, testado, com divergência menor = ⚪ P3 Baixo

4.4 MATRIZ DE COBERTURA:
Ao final, gerar matriz consolidada:
- Total de REQs no registro mestre
- REQs 100% implementados e funcionais: quantidade e %
- REQs scaffold: quantidade e %
- REQs não implementados: quantidade e %
- REQs sem teste: quantidade e %
- Gap total para 100%

═══════════════════════════════════════════════════════════════════
               NÃO ESCOPO
═══════════════════════════════════════════════════════════════════

Esta auditoria NÃO cobre:
- Qualidade da documentação em si (isso foi auditado nas Fases 1-3)
- Gestão de projeto (sprints, estimativas, velocity)
- Decisões de produto (personas, pricing, posicionamento)
- Infraestrutura cloud (AWS, GCP — apenas Docker e CI/CD locais)

Se identificar problemas nessas áreas, registre como observação no Resumo Executivo.

═══════════════════════════════════════════════════════════════════
               FASE 2 — PLANO DE CORREÇÃO
═══════════════════════════════════════════════════════════════════

Após completar os 4 eixos, gerar plano de ação consolidado.

SEVERIDADES:
🔴 P0 — Crítico (bloqueia produção):
- RN do D01 não implementada
- Endpoint fake (retorna 200 sem lógica)
- Scaffold vazio (compila mas não funciona)
- Vulnerabilidade de segurança ativa (SQL injection, CORS *, secrets exposed)
- Item proibido pela stack ShiftLabs encontrado
- REQ do Registro Mestre não implementado

🟡 P1 — Alto (problema real em uso normal):
- Inconsistência de valor entre doc e código (taxa 2.5% vs 3%)
- RBAC implementado com role errado
- Wiring quebrado (botão → handler → API desconectado)
- Falta de error handling em fluxo crítico
- Migration sem constraint obrigatória
- REQ implementado como scaffold

🟢 P2 — Médio (melhoria técnica):
- Falta de teste para funcionalidade existente
- Ambiguidade de implementação (código funciona mas pode falhar em edge case)
- Performance (N+1, bundle grande, falta lazy loading)
- Acessibilidade (contraste, keyboard nav)
- Empty/error states genéricos

⚪ P3 — Baixo (polish):
- Formatação de código, naming conventions
- Código órfão que não afeta funcionalidade
- Otimizações menores
- Textos de UI com divergência cosmética

FORMATO DO PLANO:
Tabela: # | Eixo | ID finding | Descrição | Arquivo(s) afetado(s) | Prioridade | Esforço estimado | Correção planejada

Escala de esforço: <0.5d, 0.5d, 1d, 2d, 3d, 5d, 8d, 13d (1 dia-dev = 6h produtivas).

═══════════════════════════════════════════════════════════════════
               FASE 3 — CORRIGIR
═══════════════════════════════════════════════════════════════════

Implementar TODAS as correções na ordem P0 → P1 → P2 → P3.

REGRAS DE CORREÇÃO:
1. Correções aplicadas no CÓDIGO — nunca na documentação
2. Se dois docs definem a mesma regra de forma diferente, aplicar a versão mais restritiva:
   - Prazos/timeouts: o valor menor
   - Validações: a regra mais rigorosa
   - Permissões: a mais restritiva
   - Limites numéricos: o menor limite superior ou maior limite inferior
3. Após cada bloco de prioridade (todos P0, depois todos P1, etc.):
   - Se git existe: criar commit com mensagem "fix(audit): P[N] corrections — [resumo]"
   - Verificar que build continua passando
4. Se correção quebrar o build: debugar e resolver, não reverter
5. Se correção requer novo arquivo: criar seguindo a estrutura existente do projeto
6. Se correção requer nova dependência: documentar no relatório

VERIFICAÇÃO PÓS-CORREÇÃO:
Após implementar todas as correções:
1. Executar build: `npm run build` (ou equivalente)
2. Executar testes: `npm test` (ou equivalente)
3. Verificar que nenhum finding P0/P1 permanece
4. Se testes falham: investigar e corrigir
5. Se build falha: investigar e corrigir

═══════════════════════════════════════════════════════════════════
               FASE 4 — RELATÓRIO FINAL
═══════════════════════════════════════════════════════════════════

Gerar relatório completo em: docs/05 - Auditorias/AUDIT_FASE4_CODIGO.md

Se a pasta docs/05 - Auditorias/ não existir, criar.

FORMATO DO RELATÓRIO:

SEÇÃO 1 — METADADOS:
- Nome do projeto
- Stack detectada (frontend + backend + banco + infra)
- Conformidade com stack ShiftLabs: Sim | Parcial | Não
- Número de arquivos analisados
- Número de regras extraídas (por categoria)
- Data e hora da auditoria
- Branch auditado
- Suposições assumidas (numeradas, com justificativa)
- Decisões tomadas pela IA (com justificativa e alternativas descartadas)
- O que faltou na entrada e como limita a auditoria
- Auditorias anteriores consultadas (se existirem)

SEÇÃO 2 — SUMÁRIO EXECUTIVO:
- Score geral (1-10) com justificativa de 2-3 frases
- Rubrica: 9-10 = zero P0/P1, dev novo faz setup em 30min | 7-8 = zero P0, <=3 P1 | 5-6 = 1-2 P0 | 3-4 = múltiplos P0 | 1-2 = projeto não builda
- Total de findings por prioridade (P0, P1, P2, P3)
- Total de findings por eixo
- Top 5 findings mais críticos
- Esforço total estimado para todas as correções

SEÇÃO 3 — MATRIZ DE COBERTURA (REGISTRO MESTRE):
Tabela consolidada do Eixo 4:
- Total de REQs
- % implementados e funcionais
- % scaffold
- % não implementados
- % sem teste
- Lista de REQs P0 (não implementados) com detalhes

SEÇÃO 4 — FINDINGS POR EIXO:

4.1 Eixo 1 — Código ↔ Documentação:
- Total de regras extraídas
- ✅ Implementadas: quantidade e %
- ⚠️ Parciais: quantidade e %
- ❌ Ausentes: quantidade e %
- 🔵 Não documentadas: quantidade
- Lista completa de findings com: ID | Tipo | Descrição | Arquivo | Prioridade | Corrigido?

4.2 Eixo 2 — Fullstack Técnica:
- Findings por dimensão (1-13)
- Lista completa com: Dimensão | ID | Descrição | Arquivo | Prioridade | Corrigido?

4.3 Eixo 3 — UI/UX Visual:
- Findings por subcategoria (estados, feedback, a11y, responsividade, motion, tokens)
- Lista completa com: Categoria | ID | Descrição | Tela/Componente | Prioridade | Corrigido?

4.4 Eixo 4 — Registro Mestre:
- Matriz de cobertura completa
- Lista de REQs com problemas: REQ-ID | Problema | Prioridade | Corrigido?

SEÇÃO 5 — CORREÇÕES APLICADAS:
Para CADA correção implementada:
- ID do finding
- Arquivo alterado
- O que foi mudado (resumo)
- Regra atendida
- Commit (se git disponível)
Tabela: # | Finding ID | Arquivo | Mudança | Regra | Commit

SEÇÃO 6 — CONFORMIDADE COM STACK:
- Stack oficial vs stack do projeto
- Itens proibidos encontrados (sempre P0)
- Itens obrigatórios ausentes
- Convenções de código: naming, estrutura de pastas, commits

SEÇÃO 7 — GATE FINAL DE PRODUÇÃO:

CRITÉRIOS DE APROVAÇÃO:
- Zero P0 pendentes: SIM / NÃO
- Zero P1 pendentes: SIM / NÃO
- Build passa: SIM / NÃO
- Testes passam: SIM / NÃO
- Todos os REQs do Registro Mestre implementados: SIM / NÃO

VEREDICTO:
Se TODOS os critérios = SIM:
  ✅ GATE APROVADO — PIPELINE SHIFTLABS CONCLUÍDO
  O projeto está pronto para produção.
  Mensagem: "Pipeline ShiftLabs concluído com sucesso. Todos os requisitos implementados, testados e auditados. O projeto está pronto para deploy em produção."

Se QUALQUER critério = NÃO:
  ❌ GATE REPROVADO — CORREÇÕES NECESSÁRIAS
  Listar exatamente o que falta para aprovação.
  O pipeline não está concluído até que todos os critérios sejam atendidos.

SEÇÃO 8 — OBSERVAÇÕES E RECOMENDAÇÕES:
- Observações que não são findings mas merecem atenção
- Recomendações para melhorias futuras (pós-produção)
- Riscos identificados que não são bloqueantes
- Dívida técnica aceita conscientemente

═══════════════════════════════════════════════════════════════════
         REGRAS DE QUALIDADE DO OUTPUT
═══════════════════════════════════════════════════════════════════

IDIOMA: Português brasileiro (pt-BR). Nomes de arquivos, funções, variáveis, rotas, tabelas, colunas e configs mantidos exatamente como no código/doc. Termos técnicos em inglês.

NUMERAÇÃO: Seções 1-8 no relatório. Referências cruzadas: "Eixo X, Item Y" (ex: "Eixo 1, RN-003").

QUALIDADE:
- Cada célula de tabela: no mínimo uma frase completa — célula vazia proibida
- Campo "Divergência": específico — nunca apenas "diverge" ou "incompleto"
- Referências ao código: arquivo + função/linha
- Referências à doc: arquivo .md + seção/heading
- Nunca use "etc.", "entre outros", "quando aplicável"
- Nenhuma tabela pode ter célula vazia — se não se aplica, escrever "Não aplicável: [motivo]"

VERIFICAÇÃO CRUZADA OBRIGATÓRIA (antes de entregar o relatório):
- Eixo 1: Toda regra extraída tem linha no cruzamento? Toda ❌/⚠️ tem finding no Plano?
- Eixo 2: Toda dimensão foi auditada? Todo finding tem arquivo+função?
- Eixo 3: Toda tela do inventário foi verificada? Todo estado ausente é finding?
- Eixo 4: Todo REQ do Registro Mestre foi verificado? Todo REQ ausente é P0?
- Plano → Relatório: Todo finding tem correção? Toda correção tem registro?
- Números do Sumário Executivo são exatos e verificáveis
- ✅/⚠️/❌ somam 100% das regras documentadas

IDEMPOTÊNCIA: Mesma entrada = substantivamente mesmo output. Mesmos IDs, status, prioridades, veredicto. Teste mental: "Se eu rodasse de novo, mudaria algo?" Se sim, corrija.

═══════════════════════════════════════════════════════════════════
         PROTOCOLO ANTI-TRUNCAMENTO (OBRIGATÓRIO)
═══════════════════════════════════════════════════════════════════

Este prompt gera outputs extensos. Truncar, cortar, resumir ou omitir qualquer parte é PROIBIDO.

1. DETECÇÃO PROATIVA:
   Antes de iniciar cada seção, estimar tamanho restante disponível.
   Se estimar que o output total ultrapassará 80% do limite de tokens, ativar divisão.

2. DIVISÃO EM PARTES:
   Quando o limite for atingido, PARE no final da seção atual.
   Escreva: "--- CONTINUAÇÃO: PARTE [N+1] ---" + "Seções restantes: [listar]"
   Continue AUTOMATICAMENTE na próxima mensagem.
   Cada parte começa com: "--- PARTE [N] DE [TOTAL] — CONTINUAÇÃO ---"

3. REGRAS DE DIVISÃO:
   - Nunca divida no meio de uma tabela
   - Nunca divida no meio de uma seção
   - O Gate Final (Seção 7) SEMPRE na última parte
   - Se uma seção é muito grande, divida por subseções

4. VERIFICAÇÃO FINAL:
   Ao completar a última parte, listar todas as seções entregues e confirmar que nenhuma foi omitida.

GUARDRAIL DE TAMANHO:
- Projeto pequeno (<=30 regras, <=10 telas): 8.000-15.000 palavras
- Projeto médio (30-100 regras, 10-30 telas): 15.000-25.000 palavras
- Projeto grande (100+ regras, 30+ telas): 25.000-40.000 palavras (dividir por módulo)
- Se >40.000 palavras: sinalizar no Sumário Executivo

═══════════════════════════════════════════════════════════════════
         EXECUTAR AGORA
═══════════════════════════════════════════════════════════════════

Inicie IMEDIATAMENTE a Fase 0 (pré-requisitos).
Após validar pré-requisitos, execute as 4 fases sequencialmente sem pausa.
Não peça confirmação. Não escale. Não pause.
Todas as correções estão pré-aprovadas.
O objetivo é: zero P0, zero P1, gate aprovado, pipeline concluído.

Esta é a ÚLTIMA AUDITORIA do pipeline ShiftLabs.
Após aprovação do gate, o produto vai para produção.
Trate cada finding como se impedisse o lançamento — porque impede.
```

---

## Changelog

| Versão | Data | Descrição |
| --- | --- | --- |
| 1.1 | 23/03/2026 | Protocolo de correção confirmado como P0–P3 sem exceção (já era o comportamento; changelog adicionado para rastreabilidade). A06 adicionada como auditoria complementar de UX profunda. |
| 1.0 | 22/03/2026 | Versão inicial. Consolida antigas A07 (Código↔Docs) + A08 (Fullstack) + A09 (UI/UX) em auditoria única de 4 eixos. Fase final do pipeline ShiftLabs. Gate de produção. |
