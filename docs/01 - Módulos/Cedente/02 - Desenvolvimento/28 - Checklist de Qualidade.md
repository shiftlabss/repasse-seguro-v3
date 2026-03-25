# 28 - Checklist de Qualidade

| Campo | Valor |
|---|---|
| **Nome do Documento** | 28 - Checklist de Qualidade |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Produto** | Repasse Seguro — Módulo Cedente |

---

> **TL;DR — Gate de Qualidade**
>
> - **Bloqueantes de PR:** CI vermelho, testes faltando para nova lógica, dado sensível de Cedente exposto, `console.log` em produção, secrets no código, violações de segurança P0.
> - **Bloqueantes de release:** E2E P0 falhando, cálculo de comissão com erro, isolamento LGPD violado, migration sem rollback documentado, observabilidade não configurada.
> - **4 partes:** Code Review (frontend, backend, API, dados) → Acessibilidade (WCAG 2.1 AA, mobile) → Segurança (auth, dados, transporte, headers) → Observabilidade (logs, alertas, health).
> - **Aplicar em:** todo PR, auditoria quinzenal, pré-release e pós-incidente.
> - **Ferramenta de validação chave:** `axe-core` (a11y), `pnpm audit` (deps), Sentry (erros), Lighthouse (performance/a11y), commitlint (commits).
> - **Especificidades do Módulo Cedente:** isolamento LGPD entre Cedentes, cálculo de comissão por cenário, RLS por `cedente_id`, webhook ZapSign via HMAC-SHA256, upload via signed URL.

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

- **BLOQUEANTE** — não pode ser ignorado. Impede merge ou deploy.
- **RECOMENDACAO** — melhora qualidade mas não bloqueia se justificado.

---

## 2. Parte 1 — Code Review

### 2.1 Frontend (React + Vite)

- [ ] BLOQUEANTE — Componentes funcionais exclusivamente. Class components ausentes.
- [ ] BLOQUEANTE — Nenhuma chamada `fetch` direta em `useEffect`. Usar TanStack Query.
- [ ] BLOQUEANTE — Estado global gerenciado por Zustand. Redux, MobX, Context API como state manager global ausentes.
- [ ] BLOQUEANTE — `console.log` ausente em código de produção.
- [ ] BLOQUEANTE — TypeScript `strict: true`. Nenhum `any` implícito.
- [ ] BLOQUEANTE — Axios ausente sem ADR aprovado. Usar `fetch` nativo com wrapper em `services/api.ts`.
- [ ] BLOQUEANTE — PostHog não chamado diretamente nos componentes. Apenas via `src/services/analytics.ts`.
- [ ] BLOQUEANTE — Error Boundary presente em nível global e por rota crítica (wizard de cadastro, propostas, assinaturas, Guardião).
- [ ] BLOQUEANTE — Simulador de Cenários (Etapa 3 do wizard) não exibe resultados calculados antes de receber resposta da API — skeleton loading obrigatório.
- [ ] BLOQUEANTE — Timer de 10 segundos na Etapa 3 do wizard implementado conforme RN-021. Botão "Avançar" não pode ser habilitado antes do tempo.
- [ ] BLOQUEANTE — Formulário de contraproposta bloqueia envio quando valor < piso do cenário (RN-035). Validação de piso obrigatória.
- [ ] BLOQUEANTE — Modal de aceite de proposta exibe confirmação dupla (RN-032). Aceite direto sem modal é proibido.
- [ ] RECOMENDACAO — `React.lazy()` + `Suspense` em rotas e componentes pesados (wizard de cadastro, modais de Escrow, chat Guardião).
- [ ] RECOMENDACAO — Skeleton screens em vez de spinners genéricos para estados de loading (cards de cenário, lista de propostas, painel financeiro).
- [ ] RECOMENDACAO — Framer Motion `AnimatePresence` em mudanças de rota e abertura de modais.
- [ ] RECOMENDACAO — Imagens com `loading="lazy"` e `decoding="async"`. WebP como padrão.
- [ ] RECOMENDACAO — Bundle size gzipped < 300KB verificado com `rollup-plugin-visualizer`.

### 2.2 Backend (NestJS)

- [ ] BLOQUEANTE — Todo endpoint tem DTO com `class-validator`. Inputs não validados são proibidos.
- [ ] BLOQUEANTE — Todo endpoint documentado com `@ApiOperation` e `@ApiResponse` no Swagger.
- [ ] BLOQUEANTE — `console.log` ausente. Usar Pino via `LoggerService`.
- [ ] BLOQUEANTE — TypeScript `strict: true`. Nenhum `any` implícito.
- [ ] BLOQUEANTE — Dados do Cessionário (nome, CPF, email, contato) ausentes em qualquer response acessível ao Cedente (anonimato bidirecional — RN-012 da 01.1).
- [ ] BLOQUEANTE — Toda query ao banco filtrada por `cedente_id` do usuário logado. Nunca retornar dados de outros Cedentes.
- [ ] BLOQUEANTE — Guards de autenticação (`@UseGuards(JwtAuthGuard)`) presentes em todos os endpoints protegidos.
- [ ] BLOQUEANTE — Guard de RBAC por perfil de Cedente PF vs. PJ presente onde aplicável (ex: ações de Representante Legal — RN-004 da 01.1).
- [ ] BLOQUEANTE — Rate limiting configurado via `@nestjs/throttler` em endpoints de auth e Guardião do Retorno (LLM).
- [ ] BLOQUEANTE — Correlation ID propagado em todos os logs e responses.
- [ ] BLOQUEANTE — Lógica de escalonamento respeita cooldown de 7 dias (RN-027). Validação server-side obrigatória — não confiar apenas no frontend.
- [ ] BLOQUEANTE — Verificação de duplicidade de caso por imóvel implementada no backend (RN-090). Validação client-side não é suficiente.
- [ ] BLOQUEANTE — Cálculo de comissão dos Cenários B, C, D usa fórmula exata: `20% × (valorRecuperado − valorDistratoReferencia)`, onde `valorDistratoReferencia = 50% × valorPagoCedente`.
- [ ] RECOMENDACAO — Services testados com `Test.createTestingModule()`. Não testar controllers diretamente.
- [ ] RECOMENDACAO — Prisma queries sem N+1. `include` ou `select` explícito.
- [ ] RECOMENDACAO — TTL definido para toda chave criada no Redis.

### 2.3 Mobile (React Native + Expo)

- [ ] BLOQUEANTE — Upload de documentos via câmera usa API do Expo (`expo-camera`) com validação de MIME type antes do upload (RN-042).
- [ ] BLOQUEANTE — Permissão de câmera solicitada via `expo-permissions`. Fluxo de permissão negada implementado com instrução de abertura de configurações.
- [ ] BLOQUEANTE — Notificações push configuradas via Expo Notifications com credenciais APNs (iOS) e FCM (Android) corretas.
- [ ] BLOQUEANTE — Chat do Guardião do Retorno usa SSE (Server-Sent Events) com suporte a streaming de tokens. Sem polling.
- [ ] BLOQUEANTE — Bottom tab bar com 5 itens: Dashboard, Meus Casos, Documentos, Propostas, Mais. Itens adicionais em menu "Mais".
- [ ] BLOQUEANTE — Touch targets mínimo 44×44px em todos os CTAs críticos (Aceitar Proposta, Confirmar, Assinar, Enviar documento).
- [ ] RECOMENDACAO — Expo SDK na versão definida no D02 (Stack Técnico). Não atualizar sem ADR.
- [ ] RECOMENDACAO — `React.memo()` em componentes de lista (lista de casos, lista de propostas) para evitar re-renders desnecessários.

### 2.4 API e Contratos

- [ ] BLOQUEANTE — Schema de erro seguindo `{ "error": { "code", "user_message", "correlation_id", "timestamp" } }`.
- [ ] BLOQUEANTE — Novos endpoints versionados em `/api/v1/`.
- [ ] BLOQUEANTE — Status HTTP corretos: 201 (criação), 200 (leitura/atualização), 204 (deleção), 400 (validação), 401 (não autenticado), 403 (não autorizado), 404 (não encontrado), 409 (conflito — ex: caso duplicado), 429 (rate limit), 5xx (erro interno).
- [ ] BLOQUEANTE — Idempotency key em endpoints de aceite de proposta e operações de Escrow.
- [ ] BLOQUEANTE — Webhook ZapSign validado via HMAC-SHA256 antes de processar (RN-047). Requests sem assinatura válida rejeitados com HTTP 401.
- [ ] RECOMENDACAO — Paginação com cursor ou offset em listagens de casos e propostas com > 100 registros esperados.

### 2.5 Banco de Dados (Prisma)

- [ ] BLOQUEANTE — Primary keys em UUID v4. Auto-increment ausente.
- [ ] BLOQUEANTE — Timestamps `@db.Timestamptz` em todas as colunas de data. `@db.Timestamp` sem timezone proibido.
- [ ] BLOQUEANTE — Soft delete em tabelas de domínio (`deleted_at DateTime? @db.Timestamptz`). Casos e documentos nunca deletados fisicamente.
- [ ] BLOQUEANTE — Toda nova migration tem rollback documentado como comentário no arquivo SQL.
- [ ] BLOQUEANTE — Migrations retrocompatíveis: nova coluna `nullable` antes de ser `NOT NULL`.
- [ ] BLOQUEANTE — RLS (Row-Level Security) habilitado em todas as tabelas com dados de Cedentes no Supabase. Verificar com `SELECT * FROM pg_policies` em staging.
- [ ] BLOQUEANTE — Coluna `cedente_id` indexada em todas as tabelas que fazem join com a tabela de Cedentes.
- [ ] RECOMENDACAO — Índice criado para toda nova coluna usada em `WHERE` ou `ORDER BY` frequente (ex: `status`, `created_at`, `cenario`).
- [ ] RECOMENDACAO — `created_at` e `updated_at` presentes em toda tabela nova.

### 2.6 Performance

- [ ] RECOMENDACAO — Nenhuma query sem índice em coluna de alta cardinalidade (ex: busca por `status` sem índice em tabela `casos`).
- [ ] RECOMENDACAO — Queries com `EXPLAIN ANALYZE` para operações em tabelas com > 10k registros esperados.
- [ ] RECOMENDACAO — Cache Redis para dados de alta leitura e baixa mutação (ex: lista de casos com status estável, simulação de cenário recente).
- [ ] RECOMENDACAO — Simulador de Cenários: resposta em < 300ms p95. Se cálculo exceder 5s, exibir indicador de progresso conforme RN-021.

---

## 3. Parte 2 — Acessibilidade (WCAG 2.1 AA)

### 3.1 Princípio 1: Perceptível

- [ ] BLOQUEANTE — Contraste de texto normal ≥ 4.5:1 (verificar com axe-core ou Colour Contrast Analyser).
- [ ] BLOQUEANTE — Contraste de texto grande (≥ 18pt normal / ≥ 14pt bold) ≥ 3:1.
- [ ] BLOQUEANTE — Contraste de elementos de UI (bordas, ícones, badges de status de proposta, badges de status de documento) ≥ 3:1.
- [ ] BLOQUEANTE — Contraste verificado também em dark mode.
- [ ] BLOQUEANTE — Imagens informativas com `alt` descritivo. Imagens decorativas com `alt=""`.
- [ ] BLOQUEANTE — HTML semântico: `<button>`, `<nav>`, `<main>`, `<header>`, `<aside>` (sidebar) em vez de `<div>` genérico.
- [ ] BLOQUEANTE — Documentos do dossiê: cada DocumentCard com `aria-label="[Nome do documento] — status: [status]"` conforme RN-041.
- [ ] RECOMENDACAO — Status de badges (Pendente, Em Análise, Verificado, Rejeitado) comunicados também por texto ou ícone com `alt`, não apenas por cor.

**Ferramenta:** axe-core integrado ao Playwright; Lighthouse Accessibility score ≥ 95.

### 3.2 Princípio 2: Operável

- [ ] BLOQUEANTE — Toda funcionalidade acessível por teclado (Tab, Enter, Space, Esc, setas).
- [ ] BLOQUEANTE — Nenhuma keyboard trap. Foco pode entrar e sair de qualquer componente via teclado.
- [ ] BLOQUEANTE — Focus ring visível em todos os elementos interativos. `outline: none` proibido sem substituto.
- [ ] BLOQUEANTE — Tab order lógico seguindo a ordem visual da interface (sidebar → conteúdo → ações).
- [ ] BLOQUEANTE — Focus trap ativo em modais (aceite de proposta, recusa, contraproposta, solicitação de desistência, cancelamento de caso) enquanto abertos.
- [ ] BLOQUEANTE — Focus restaurado para o elemento que abriu o overlay ao fechar modal/drawer.
- [ ] BLOQUEANTE — Skip link "Pular para conteúdo principal" visível no primeiro Tab. Oculto visualmente quando não focado.
- [ ] BLOQUEANTE — Touch targets (mobile) mínimo 44×44px. Crítico para CTAs: "Aceitar Proposta", "Assinar Agora", "Enviar Documento", "Confirmar Cadastro".
- [ ] RECOMENDACAO — Animações do wizard (transição entre etapas) respeitam `prefers-reduced-motion: reduce`. Verificar com `useReducedMotion()` no Framer Motion.

**Ferramenta:** teste manual com Tab; VoiceOver (macOS/iOS) para screen reader.

### 3.3 Princípio 3: Compreensível

- [ ] BLOQUEANTE — Labels associados a inputs via `htmlFor`/`for` ou `aria-label`. Inputs sem label visível: `aria-label` obrigatório.
- [ ] BLOQUEANTE — Mensagens de erro explicativas e associadas ao campo via `aria-describedby`. Ex: "O valor está abaixo do mínimo permitido para este cenário (R$ X.XXX)."
- [ ] BLOQUEANTE — Stepper do wizard de cadastro (5 etapas) navegável por teclado com estado atual anunciado via `aria-current="step"`.
- [ ] BLOQUEANTE — Modal de aceite de proposta com foco no primeiro elemento interativo ao abrir.
- [ ] BLOQUEANTE — Linguagem da página declarada: `<html lang="pt-BR">`.
- [ ] BLOQUEANTE — Mensagens de erro de upload de documento seguem padrão de UX Writing (RN-042, RN-045): templates padronizados, não mensagens genéricas.
- [ ] RECOMENDACAO — Estados de loading anunciados via `aria-live="polite"` (simulador de cenários) ou `aria-live="assertive"` (erros críticos de aceite de proposta).
- [ ] RECOMENDACAO — Empty states com mensagem clara e sugestão de ação. Ex: "Nenhuma proposta recebida ainda. Seu imóvel está disponível para compradores."

### 3.4 Princípio 4: Robusto

- [ ] BLOQUEANTE — ARIA roles semânticos corretos: `role="dialog"` em modais de proposta/desistência, `role="alert"` em erros urgentes, `role="status"` em loading não urgente.
- [ ] BLOQUEANTE — Landmarks corretos: `<main>` único por página; sidebar com `role="navigation"` e `aria-label="Menu principal"`.
- [ ] BLOQUEANTE — Tabelas de propostas e documentos com `<th scope>` correto. Tabelas de dados nunca usadas para layout.
- [ ] BLOQUEANTE — Formulários do wizard com `fieldset` e `legend` para grupos de inputs relacionados (ex: dados do imóvel, dados financeiros, escolha de cenário).
- [ ] BLOQUEANTE — Componentes shadcn/ui utilizados via cópia no repositório (ownership total) — não via pacote.
- [ ] BLOQUEANTE — Chat do Guardião do Retorno: `role="log"` na área de mensagens, `aria-live="polite"` — novas mensagens anunciadas ao completar streaming (não token a token).
- [ ] RECOMENDACAO — Região `aria-live` para anúncios de sucesso/erro em submissões do wizard e aceite de proposta.
- [ ] RECOMENDACAO — Screen reader flow testado para: login, wizard de cadastro (5 etapas), aceite de proposta, upload de documento, assinatura ZapSign, chat Guardião.

---

## 4. Parte 3 — Segurança

### 4.1 Autenticação e Sessão

- [ ] BLOQUEANTE — JWT access token com validade de 15 minutos. Sem token de longa duração.
- [ ] BLOQUEANTE — Refresh token em cookie `httpOnly; Secure; SameSite=Strict`. Nunca em `localStorage`.
- [ ] BLOQUEANTE — Token rotation: novo par gerado a cada refresh; token anterior invalidado no Redis.
- [ ] BLOQUEANTE — Brute force protection: 5 tentativas → lockout 15min via Redis `rs:auth:attempts:{ip}:{email}`.
- [ ] BLOQUEANTE — Re-autenticação exigida para ações críticas: aceite de proposta, assinatura de documentos, cancelamento de caso, solicitação de desistência pós-Fechamento.
- [ ] BLOQUEANTE — Sessão expirada após 30min de inatividade (Redis TTL `rs:session:{cedente_id}`).
- [ ] BLOQUEANTE — PKCE implementado no fluxo Google OAuth. State CSRF verificado.

### 4.2 Autorização e Isolamento de Dados (LGPD)

- [ ] BLOQUEANTE — RLS habilitado em todas as tabelas com dados de Cedentes no Supabase. Validar com `SELECT * FROM pg_policies` antes de cada deploy.
- [ ] BLOQUEANTE — `service_role` key do Supabase nunca exposta no frontend.
- [ ] BLOQUEANTE — Toda query ao banco filtrada por `cedente_id` do usuário logado. Sem acesso cruzado entre Cedentes.
- [ ] BLOQUEANTE — Dados do Cessionário (nome, CPF, email, telefone, valor do lance) ausentes em qualquer response do módulo Cedente. O Cedente vê apenas o valor da proposta (RN-030 da 01.2).
- [ ] BLOQUEANTE — RBAC verificado em todos os endpoints com `@UseGuards()` apropriados. Cedente PJ: ações restritas ao Representante Legal designado.
- [ ] BLOQUEANTE — Guardião do Retorno sem acesso a dados de outros Cedentes. Contexto do LLM isolado por sessão de usuário.
- [ ] BLOQUEANTE — Solicitação de exclusão de dados LGPD (T-052 do Mapa de Telas) requer re-autenticação e gera protocolo auditável.

### 4.3 Validação de Input e Output

- [ ] BLOQUEANTE — Todo input de usuário validado via `class-validator` no DTO antes de processar.
- [ ] BLOQUEANTE — Sanitização de prompt do usuário antes de inserir no LLM (`InputSanitizerService`). Prevenção de prompt injection no Guardião do Retorno.
- [ ] BLOQUEANTE — Filtro de output da IA antes de exibir ao Cedente (`OutputFilterService` — bloqueia respostas com manipulação emocional, FOMO ou urgência artificial).
- [ ] BLOQUEANTE — Webhook ZapSign validado via HMAC-SHA256 antes de processar (RN-047). Requests sem assinatura válida rejeitados.
- [ ] BLOQUEANTE — Upload de documentos do dossiê via signed URL do Supabase Storage. Nunca multipart direto no NestJS.
- [ ] BLOQUEANTE — Validação de MIME type real (não apenas extensão) nos uploads (RN-042). Arquivos renomeados detectados.
- [ ] BLOQUEANTE — Contraproposta bloqueada server-side quando valor < piso do cenário ativo (RN-035). Validação apenas client-side é insuficiente.

### 4.4 Secrets e Configuração

- [ ] BLOQUEANTE — Nenhuma credencial, token ou API key no código-fonte ou git history.
- [ ] BLOQUEANTE — Todas as variáveis de ambiente documentadas no `.env.example` sem valores reais.
- [ ] BLOQUEANTE — Secrets em Railway (backend), Vercel (frontend) e EAS (mobile). Nunca em `.env` commitado.
- [ ] BLOQUEANTE — `pnpm audit --audit-level=high` sem vulnerabilidades críticas.
- [ ] RECOMENDACAO — Variáveis sensíveis com prefixo `_KEY`, `_SECRET`, `_TOKEN` conforme D22 (Stack Técnico).

### 4.5 Transporte e Headers

- [ ] BLOQUEANTE — Helmet configurado desde o primeiro commit (headers de segurança automáticos).
- [ ] BLOQUEANTE — HTTPS obrigatório em todos os ambientes (staging e produção).
- [ ] BLOQUEANTE — `Content-Security-Policy` configurado para bloquear inline scripts não autorizados.
- [ ] BLOQUEANTE — `X-Frame-Options: DENY` para prevenir clickjacking (exceto para o iframe do ZapSign — allowlist explícita).
- [ ] BLOQUEANTE — `Referrer-Policy: strict-origin-when-cross-origin`.
- [ ] RECOMENDACAO — `Strict-Transport-Security` (HSTS) com `max-age ≥ 31536000` em produção.

### 4.6 Específico do Módulo Cedente

- [ ] BLOQUEANTE — Escalonamento de cenário: servidor valida que o escalonamento é descendente (D→C, C→B, B→A) e não pula níveis (RN-026 da 01.2).
- [ ] BLOQUEANTE — Cooldown de 7 dias entre escalonamentos validado no servidor (RN-027 da 01.2). Não confiar no estado do frontend.
- [ ] BLOQUEANTE — Aceite de proposta verificado server-side contra timer de expiração (5 dias úteis — RN-031 da 01.2).
- [ ] BLOQUEANTE — Documento verificado (`status: 'verificado'`) não pode ser substituído pelo Cedente (RN-044 da 01.3). Backend rejeita novo upload para documento neste status.
- [ ] BLOQUEANTE — Solicitação de desistência disponível apenas dentro dos 15 dias corridos após o Fechamento (RN-053 da 01.3). Backend valida o período ativo.
- [ ] BLOQUEANTE — Cenário A exibe comissão R$ 0 e disclaimer explicativo na tela Financeiro (RN-054 da 01.3). Nunca exibir R$ 0 sem explicação.

---

## 5. Parte 4 — Observabilidade

### 5.1 Logs Estruturados (Pino)

- [ ] BLOQUEANTE — Todos os logs em formato JSON estruturado via Pino. Logs de texto livre proibidos.
- [ ] BLOQUEANTE — Correlation ID presente em todos os logs de request/response.
- [ ] BLOQUEANTE — Eventos de negócio críticos logados com nível `info`: aceite de proposta, escalonamento de cenário, assinatura de documento, abertura/distribuição de Escrow.
- [ ] BLOQUEANTE — Erros de integração ZapSign e Escrow logados com nível `error` e contexto completo (payload de entrada, resposta recebida, correlation_id).
- [ ] BLOQUEANTE — Dados sensíveis (CPF, email, telefone do Cedente) redactados nos logs. Nunca logar em texto limpo.
- [ ] RECOMENDACAO — Nível de log por ambiente: `debug` em desenvolvimento, `info` em staging, `warn` em produção.

### 5.2 Rastreamento de Erros (Sentry)

- [ ] BLOQUEANTE — Sentry inicializado com DSN correto antes do primeiro deploy em staging.
- [ ] BLOQUEANTE — Sentry configurado para capturar: erros não tratados, exceções de DTO, falhas de integração (ZapSign, Escrow).
- [ ] BLOQUEANTE — `release` configurado no Sentry para cada deploy (permite rastrear quando erro foi introduzido).
- [ ] BLOQUEANTE — Dados pessoais do Cedente não enviados ao Sentry. `beforeSend` configurado para redactar CPF, email e telefone.
- [ ] RECOMENDACAO — Alertas Sentry configurados para: novos erros P0, aumento de taxa de erro > 2% em 5min.

### 5.3 Rastreamento do Guardião do Retorno (Langfuse)

- [ ] BLOQUEANTE — Langfuse inicializado e recebendo traces de todas as chamadas ao LLM do Guardião.
- [ ] BLOQUEANTE — Cada trace inclui: `cedente_id` anonimizado (hash), `session_id`, latência, tokens consumidos, modelo usado.
- [ ] BLOQUEANTE — Latência do Guardião monitorada: alerta se p95 > 5s ou taxa de erro > 5%.
- [ ] RECOMENDACAO — Feedback de qualidade das respostas do Guardião rastreado no Langfuse para melhoria contínua.

### 5.4 Analytics (PostHog)

- [ ] BLOQUEANTE — PostHog inicializado com Project Key correto antes do primeiro deploy em staging.
- [ ] BLOQUEANTE — Eventos obrigatórios implementados e verificados em staging:

| Evento PostHog | Quando disparar | Propriedades obrigatórias |
|---|---|---|
| `caso_cadastrado` | Confirmação do wizard (Etapa 5) | `cenario_escolhido`, `tipo_cedente` (PF/PJ) |
| `cenario_selecionado` | Etapa 4 do wizard — seleção do cenário | `cenario`, `tempo_visualizacao_simulador_segundos` |
| `proposta_aceita` | Confirmação do aceite de proposta | `valor_proposta`, `valor_liquido_cedente`, `cenario` |
| `proposta_recusada` | Confirmação da recusa | `motivo_interno` (se fornecido) |
| `contraproposta_enviada` | Envio de contraproposta | `valor_contraproposta`, `cenario` |
| `escalonamento_solicitado` | Solicitação de escalonamento | `cenario_anterior`, `cenario_novo` |
| `documento_enviado` | Upload de documento concluído | `tipo_documento`, `tentativas` |
| `assinatura_concluida` | ZapSign reporta assinatura | `tipo_documento_assinado` |
| `guardiao_consulta` | Mensagem enviada ao Guardião | `modo` (normal/cadastro_assistido) |
| `desistencia_solicitada` | Solicitação dentro dos 15 dias | `dias_apos_fechamento` |

- [ ] RECOMENDACAO — Feature flags do PostHog utilizadas para rollout gradual de funcionalidades novas (ex: cadastro assistido via Guardião).

### 5.5 Health Checks

- [ ] BLOQUEANTE — Endpoint `GET /health` retornando `{ "status": "ok" }` com HTTP 200 em < 500ms.
- [ ] BLOQUEANTE — Health check inclui verificação de: conexão com Postgres, conexão com Redis, conectividade com ZapSign (ping), conectividade com Escrow (ping).
- [ ] BLOQUEANTE — Railway health check configurado para reinicializar instância se `/health` falhar por 3 tentativas consecutivas.

---

## 6. Tabela de Bloqueantes por Tipo de Evento

### 6.1 Bloqueantes de PR (impede merge em qualquer branch)

| Item | Verificação | Ferramenta |
|---|---|---|
| CI vermelho (lint, typecheck, testes) | Workflow GitHub Actions vermelho | GitHub Actions |
| `any` implícito no TypeScript | `pnpm typecheck` com erro | TypeScript compiler |
| `console.log` em código de produção | `pnpm lint` detecta `no-console` | ESLint |
| Teste faltando para nova lógica de comissão | Cobertura abaixo do threshold (tabela 3.3 do D27) | Vitest coverage |
| Dado pessoal do Cedente exposto em response | Revisão manual de response schemas | Code review |
| Dados do Cessionário expostos ao Cedente | Revisão manual de response schemas | Code review |
| Secret ou credencial no código | `git diff` + `pnpm audit` | GitHub Secrets scanning |
| Migration sem rollback documentado | Revisão manual do arquivo SQL | Code review |
| RLS ausente em nova tabela de Cedente | `SELECT * FROM pg_policies` em staging | SQL manual |

### 6.2 Bloqueantes de Release (impede deploy para produção)

| Item | Verificação | Ferramenta |
|---|---|---|
| E2E P0 falhando em staging | Suíte E2E completa verde | Playwright CI |
| Cálculo de comissão com resultado incorreto | 100% de cobertura em `CommissionCedenteService` | Vitest |
| Isolamento LGPD violado (SEC-001 a SEC-005 do D27) | Testes de segurança passando | Playwright |
| Webhook ZapSign sem validação HMAC-SHA256 | Teste INT-020 do D27 passando | Vitest integração |
| Performance p95 > 200ms em endpoints críticos | `k6` load test em staging | k6 |
| Sentry sem inicialização em produção | Verificação manual pós-deploy | Sentry dashboard |
| PostHog sem captura de eventos obrigatórios | Verificação em staging com PostHog debug | PostHog dashboard |
| Langfuse sem traces do Guardião | Verificação em staging | Langfuse dashboard |
| Dados de teste não removidos de staging | Checklist pré-release (D29) | Manual |
| `pnpm audit` com vulnerabilidade crítica | `pnpm audit --audit-level=high` | CLI |

---

## 7. Checklist de Revisão de PR — Versão Rápida

> Use este checklist resumido no corpo de cada PR. O checklist completo acima é para auditorias e pré-release.

```markdown
## Checklist de PR — Módulo Cedente

### Código
- [ ] Testes escritos para toda lógica nova (unitário obrigatório para lógica de comissão/escalonamento)
- [ ] TypeScript sem `any` implícito
- [ ] `console.log` ausente em produção
- [ ] Nenhum dado de Cessionário exposto ao Cedente (e vice-versa)
- [ ] Todas as queries filtradas por `cedente_id`

### Banco de dados
- [ ] Nova entidade com RLS configurado
- [ ] Migration com rollback documentado no SQL
- [ ] Sem breaking changes retroativos

### API
- [ ] Mensagens de erro seguem padrão: `{ "error": { "code", "user_message", "correlation_id", "timestamp" } }`
- [ ] Endpoint documentado no Swagger

### Regras de negócio específicas
- [ ] Cálculo de comissão usa fórmula correta (20% × (valorRecuperado − valorDistratoRef))
- [ ] Escalonamento valida: descendente, 1 nível por vez, cooldown de 7 dias (server-side)
- [ ] Upload valida MIME real (não apenas extensão)
- [ ] Webhook ZapSign com HMAC-SHA256

### Observabilidade
- [ ] Eventos PostHog implementados para nova funcionalidade
- [ ] Sentry capturando novos tipos de erro se aplicável
```
