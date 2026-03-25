# S1 — Fundação

| Campo                | Valor                                                                                                                                                                                            |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sprint**           | S1                                                                                                                                                                                               |
| **Nome**             | Fundação                                                                                                                                                                                         |
| **Tipo**             | Fixa — Infraestrutura                                                                                                                                                                            |
| **Módulo**           | Cedente                                                                                                                                                                                          |
| **Template**         | Template A (Infraestrutura)                                                                                                                                                                      |
| **Docs consultados** | D02 (Stacks), D12 (Modelo de Dados), D13 (Schema Prisma), D14 (Specs Técnicas), D15 (Arquitetura de Pastas), D17 (Integrações), D20 (Error Handling), D22 (Setup Local), D23 (Guia Contribuição) |
| **Cross-cutting**    | D10 (Glossário), D02 (Stacks)                                                                                                                                                                    |
| **REQs cobertos**    | REQ-133 a REQ-159, REQ-169(infra), REQ-170-173, REQ-174-180, REQ-194-201, REQ-209                                                                                                                |

> **Critério de conclusão:** projeto roda localmente com `pnpm dev`; banco PostgreSQL existe com esquema completo (11 tabelas + 16 enums + RLS habilitado); layout shell renderiza com sidebar; rotas protegidas por auth guard; error handling global responde com RFC 7807; integrações externas configuradas com health checks; zero lógica de negócio — apenas infraestrutura.

---

## 🗃️ BANCO DE DADOS

### Migrations Prisma — 11 tabelas

- [x] Migration criada para tabela `users`: uuid PK via pgcrypto, email UK, name, provider, email_verified_at Timestamptz, created_at Timestamptz, updated_at Timestamptz, deleted_at Timestamptz (soft delete) — REQ-133
- [x] Migration criada para tabela `cedentes`: uuid PK, user_id FK → users, tipo (enum TipoCedente: PF/PJ), cpf_cnpj UK, telefone, status_conta (enum StatusConta), lgpd_consent boolean, lgpd_consent_at Timestamptz, notification_preferences JSONB, ai_consent boolean, ai_consent_at Timestamptz, created_at, updated_at — REQ-134
- [x] Migration criada para tabela `casos`: uuid PK, cedente_id FK → cedentes, codigo UK, status (enum StatusCaso — 13 valores), status_interno (enum StatusCasoInterno), cenario (enum CenarioRetorno: A/B/C/D), cidade, bairro, estado, nome_empreendimento, endereco_completo, quartos int, area_m2 decimal, tem_garagem boolean, valor_pago_cedente decimal, valor_tabela_contrato decimal, valor_tabela_atual decimal, valor_distrato_referencia decimal, valor_recuperado decimal, comissao_rs decimal, valor_liquido_cedente decimal (generated column), construtora, cessao_livre boolean, data_cadastro, data_aprovacao, data_fechamento, data_distribuicao, ultimo_escalonamento_at, deleted_at — REQ-135
- [x] Migration criada para tabela `dossies`: uuid PK, caso_id FK (relação 1:1 com casos), status (enum StatusDossie), total_documentos int, documentos_verificados int, documentos_pendentes int, documentos_rejeitados int, completo boolean, completo_em Timestamptz, created_at, updated_at — REQ-136
- [x] Migration criada para tabela `documentos_dossie`: uuid PK, dossie_id FK → dossies, caso_id FK → casos, tipo (enum TipoDocumento), nome, status (enum StatusDocumento: PENDENTE/EM_ANALISE/VERIFICADO/REJEITADO), url_arquivo, mime_type, tamanho_bytes int, motivo_rejeicao, verificado_por FK → users, verificado_em Timestamptz, tentativas int default 0, created_at, updated_at — REQ-137
- [x] Migration criada para tabela `propostas`: uuid PK, caso_id FK → casos, cedente_id FK → cedentes, codigo UK, valor_proposto decimal, comissao_comprador decimal, valor_liquido_cedente_estimado decimal, status (enum StatusProposta), mensagem_admin, rodadas_contraproposta int default 0, valor_contraproposta decimal, expires_at Timestamptz, respondido_em Timestamptz, created_at, updated_at, deleted_at — REQ-138
- [x] Migration criada para tabela `envelopes_assinatura`: uuid PK, caso_id FK → casos, cedente_id FK → cedentes, tipo_documento (enum TipoEnvelope: TERMO_CADASTRO/TERMO_ESCALONAMENTO/TERMO_COMERCIAL/INSTRUMENTO_CESSAO), status (enum StatusEnvelope), zapsign_token, zapsign_doc_url, created_at, updated_at — REQ-139
- [x] Migration criada para tabela `contas_escrow`: uuid PK, caso_id FK, status (enum StatusEscrow: ABERTA/DEPOSITO_CONFIRMADO/EM_PERIODO_DE_REVERSAO/VALORES_DISTRIBUIDOS/COMISSAO_DEFINITIVA/ESTORNADA), created_at, updated_at — REQ-140
- [x] Migration criada para tabela `notificacoes`: uuid PK, cedente_id FK → cedentes, tipo (enum TipoNotificacao — 17+ tipos), lida boolean default false, dados JSONB (conteúdo da notificação), redirect_url, created_at — REQ-141
- [x] Migration criada para tabela `eventos_caso`: uuid PK, caso_id FK → casos, tipo (enum TipoEvento), ator (enum TipoAtor), descricao, metadados JSONB, criado_em Timestamptz — REQ-142
- [x] Migration criada para tabela `ai_sessions`: uuid PK, cedente_id FK → cedentes, mensagens JSONB array (role, content, timestamp), anonimizado_em Timestamptz (LGPD — 90 dias), created_at, updated_at — REQ-143

### Enums Prisma — 16 enums

- [x] Todos os 16 enums declarados no schema.prisma: TipoCedente (PF/PJ), StatusConta (PENDENTE_ATIVACAO/ATIVA/BLOQUEADA/ENCERRADA), StatusCaso (13 valores: CADASTRO_REALIZADO/EM_ANALISE/PENDENCIA_IDENTIFICADA/APROVADO_PARA_OFERTA/DISPONIVEL_PARA_COMPRADORES/PROPOSTA_RECEBIDA/EM_FORMALIZACAO/NEGOCIO_FECHADO/AGUARDANDO_LIBERACAO/DESISTENCIA_EM_ANALISE/MEDIACAO_EM_ANDAMENTO/CONCLUIDO/CANCELADO), StatusCasoInterno (13 valores), CenarioRetorno (A/B/C/D), StatusDossie, TipoDocumento, StatusDocumento, TipoProposta, StatusProposta, TipoEnvelope (4 valores), StatusEnvelope, StatusEscrow (6 valores), TipoNotificacao, TipoAtor, TipoEvento — REQ-145

### Convenções Prisma

- [x] Schema Prisma com datasource postgresql, DATABASE_URL e DIRECT_URL via env(); extensão pgcrypto habilitada via extensions = [pgcrypto] — REQ-147
- [x] Convenções aplicadas: UUID v4 via @default(dbgenerated("gen_random_uuid()")), snake_case via @map/@@@map em todos os models, PascalCase para models, soft delete via deletedAt DateTime? em todas as tabelas de domínio principal — REQ-147
- [x] generated column `valor_liquido_cedente` em `casos` definida via @default(dbgenerated(...)) ou view/trigger com cálculo `valor_recuperado - comissao_rs` — REQ-135

### RLS e Segurança de Banco

- [x] RLS habilitado em todas as tabelas com dados de usuários via Supabase; política de filtro por `cedente_id` implementada e testada — REQ-144
- [x] Tabela `propostas` sem campo `cessionario_id` exposto ao frontend do Cedente; anonimização estrutural garantida no model e na API — REQ-144
- [x] Índices criados: `cedentes.user_id`, `cedentes.cpf_cnpj`, `casos.cedente_id`, `casos.codigo`, `documentos_dossie.dossie_id`, `propostas.caso_id`, `notificacoes.cedente_id` — REQ-133 a REQ-140

### Seed

- [x] Seed `prisma/seed/cedente.seed.ts` criado com: 1 Cedente PF ativo, 1 Cedente PJ ativo, 2 casos em diferentes status, dossiês associados com documentos em estados variados — para uso em desenvolvimento e testes — REQ-146

---

## 🏗️ BACKEND (NestJS)

### Estrutura de Pastas

- [x] Estrutura `apps/api/src/modules/` criada com 11 módulos de domínio: `auth/`, `cedentes/`, `casos/`, `dossie/`, `documentos/`, `propostas/`, `escalonamento/`, `assinaturas/`, `financeiro/`, `notificacoes/`, `ai/` — REQ-157
- [x] Cada módulo com padrão: `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `dto/`, `entities/` — REQ-157
- [x] Diretório `apps/api/src/common/` com: `guards/`, `interceptors/`, `filters/`, `pipes/`, `decorators/` — REQ-157
- [x] Diretório `apps/api/src/infrastructure/` com: `database/` (PrismaService), `cache/` (Redis), `queue/` (RabbitMQ), `storage/` (Supabase Storage) — REQ-157

### Error Handling Global

- [x] `GlobalExceptionFilter` implementado no NestJS capturando todas as exceções não tratadas — REQ-194
- [x] Todo erro retornado no formato RFC 7807 Problem Details: `{ type, title, status, detail, instance, code, cedente_message, timestamp, trace_id }` — REQ-195
- [x] Campo `detail` (técnico) e `cedente_message` (amigável e acionável) sempre separados; `cedente_message` nunca revela stack trace ou detalhes internos — REQ-195
- [x] 11 prefixos de erro mapeados: AUTH, CED, CAS, DOC, PRP, ASS, ESC, NOT, AI, ANU, COM — REQ-159

### Circuit Breakers

- [x] Circuit breaker configurado para integração ZapSign: threshold de falhas, timeout, estado half-open — REQ-196
- [x] Circuit breaker configurado para integração Conta Escrow: threshold de falhas, timeout, estado half-open — REQ-196

### Cache Redis

- [x] Conexão Redis configurada via Upstash em produção; Docker em desenvolvimento — REQ-153
- [x] Prefixo `rs:cedente:` em todas as chaves Redis — REQ-158
- [x] 8 chaves documentadas com TTL explícito: sessão (TTL 24h), rascunho wizard (TTL 30 dias), dados do caso (TTL 60s), score proposta, resultado IA (TTL 4h), notificações não lidas, rate limit tokens, CNPJ validado — REQ-153

### Filas RabbitMQ

- [x] Conexão RabbitMQ configurada via CloudAMQP em produção; Docker em desenvolvimento — REQ-154
- [x] 5 exchanges criados (notification.exchange, email.exchange, zapsign.exchange, reminder.exchange, jobs.exchange) — REQ-154
- [x] 8 queues configuradas com DLQ em todas e retry com backoff exponencial (3 tentativas em 30 min: 2min → 10min → 1h) — REQ-154

### Pino Logging

- [x] PinoLogger configurado no NestJS com formato JSON estruturado — REQ-209
- [x] Campos obrigatórios em todo log: `requestId`, `userId`, `module`, `action`, `timestamp` — REQ-209
- [x] Request ID gerado por middleware e propagado via contexto em todo ciclo de vida da request — REQ-209

### Sentry Setup

- [x] Sentry SDK instalado no NestJS com DSN via env var `SENTRY_DSN` — REQ-197
- [x] Alertas configurados por severidade: P0→PagerDuty, P1→Slack #alerts <5min, P2→Slack #warnings <30min — REQ-197

### Validação de DTOs

- [x] `class-validator` + `class-transformer` instalados; ValidationPipe global configurado — REQ-133 a REQ-145
- [x] Erro de validação retorna 400 com `validation_errors[]` por campo (específico, nunca genérico) — REQ-195

### Helmet e Segurança

- [x] Helmet configurado como primeiro middleware na aplicação NestJS — REQ-133

### Rate Limiting Global

- [x] Rate limiter configurado: 100 req/min geral (autenticado); headers `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` em toda resposta — REQ-171
- [x] Rate limits específicos: auth login/refresh 10 req/15min; cadastro 5 req/h; recuperação senha 3 req/h; upload dossiê 20 req/10min; propostas 30 req/5min; IA 20 req/min — REQ-171

### Swagger/OpenAPI

- [x] @nestjs/swagger configurado com documento OpenAPI gerado automaticamente em /api/docs — REQ-133
- [x] Base URL `/api/v1` com versionamento via prefixo de path — REQ-170

---

## 🎨 FRONTEND (Next.js — Layout Shell)

### Estrutura de Pastas Web

- [x] `apps/web-cedente/` com route groups `(public)/` (layout sem auth) e `(authenticated)/` (layout logado com sidebar) — REQ-155
- [x] `src/features/<módulo>/` com barrel exports obrigatórios para cada domínio — REQ-155
- [x] `src/services/api.ts` com wrapper de `fetch` nativo (sem Axios direto) — REQ-155

### Layout Shell Autenticado

- [x] `AppLayout` com sidebar fixa lateral esquerda renderizando: Dashboard, Meus Casos, Cadastrar Imóvel, Documentos, Propostas, Assinaturas, Financeiro, Assistente IA, Meu Perfil — REQ-022 (itens do menu)
- [x] Header com ícone de sino para notificações (badge numérico, aria-label "Notificações — [X] não lidas"), nome do usuário logado e opção de logout — REQ-095
- [x] Sidebar: navegação sem recarregamento de página (client-side navigation); cada item troca a área de trabalho — REQ-022
- [x] Auth guard implementado: rotas autenticadas redirecionam para `/login` se sem token válido — REQ-013

### Design Tokens e Tema

- [x] `packages/design-tokens/` com tokens CSS + JSON para cores, tipografia, espaçamentos e shadows — REQ-148
- [x] Tailwind 4 configurado com tokens do design system; shadcn/ui integrado — REQ-148

### Toast / Notificação Global

- [x] Sistema de toast global implementado (feedback de sucesso/erro/warning) acessível via hook ou contexto — REQ-194

### Error Boundaries

- [x] Error boundary global no layout raiz capturando erros não tratados no React — REQ-194
- [x] Error boundary por feature (cada módulo tem seu boundary) exibindo estado de erro com botão "Tentar novamente" — REQ-194

### Supabase Realtime (Setup)

- [ ] Supabase Realtime client configurado no frontend para subscriptions de notificações do Cedente logado — REQ-095

---

## 📱 MOBILE (Expo — Setup)

### Estrutura de Pastas Mobile

- [ ] `apps/mobile-cedente/` com `src/screens/<módulo>/` e expo-router 4 file-based routing — REQ-156
- [ ] Stack navigator raiz com rota `/login` e rota `/(tabs)/` — REQ-218

### Setup Expo

- [ ] Expo SDK 52+ configurado; React Native 0.76+; expo-router 4 instalado — REQ-216
- [ ] `expo-secure-store` instalado para armazenamento de tokens — REQ-185
- [ ] `@react-native-community/netinfo` instalado para detecção de estado de rede offline — REQ-219

---

## 🔗 INTEGRAÇÕES (Setup)

### Supabase

- [x] Supabase configurado com DATABASE_URL, DIRECT_URL, SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY via env vars — REQ-176
- [x] PrismaService singleton injetável no NestJS com conexão ao Supabase PostgreSQL — REQ-176
- [ ] Supabase Storage bucket `documentos-dossie` criado com políticas de acesso restrito por `cedente_id` — REQ-176

### ZapSign

- [x] ZAPSIGN_API_KEY, ZAPSIGN_WEBHOOK_SECRET, ZAPSIGN_BASE_URL configurados via env vars — REQ-174
- [x] `ZapSignService` criado no NestJS com método de health check e configuração de retry (2min→10min→1h→DLQ) — REQ-174

### Conta Escrow

- [x] ESCROW_API_KEY, ESCROW_WEBHOOK_SECRET, ESCROW_BASE_URL configurados via env vars [⚠️ AMBÍGUO: parceiro Escrow não definido — DP-001; placeholder com mock para desenvolvimento] — REQ-175

### Resend

- [x] RESEND_API_KEY configurado; ResendService criado com SDK TypeScript; templates React Email em `apps/api/src/infrastructure/email/templates/` — REQ-177

### OpenAI

- [x] OPENAI_API_KEY configurado; OpenAI SDK 4.x instalado; modelo fixado `gpt-4-turbo-2024-04-09` — REQ-178

### Receita Federal

- [x] URL pública da API Receita Federal configurada; `ReceitaFederalService` criado sem autenticação; fallback documentado — REQ-179

### Langfuse

- [x] LANGFUSE_SECRET_KEY + LANGFUSE_PUBLIC_KEY configurados; SDK Langfuse 3.x instalado; fire-and-forget com falha silenciosa — REQ-180

### Sentry + PostHog

- [x] SENTRY_DSN, POSTHOG_API_KEY configurados em backend, web e mobile; SDKs instalados — REQ-180

---

## 🔧 MONOREPO E AMBIENTE

### Estrutura Monorepo

- [x] Turborepo configurado na raiz com `turbo.json` definindo tarefas: `build`, `test`, `lint`, `type-check` — REQ-148
- [x] `pnpm-workspace.yaml` declarando todos os workspaces: apps/_, packages/_ — REQ-148
- [x] `packages/shared-types/` com types TypeScript compartilhados entre apps — REQ-148
- [x] `packages/eslint-config/` com config ESLint base (strict: true, no explicit any) — REQ-148
- [x] `packages/tsconfig/` com config TypeScript base (strict: true) — REQ-148
- [x] Um único `pnpm install` na raiz instala todas as dependências — REQ-200

### Setup Local (.env.example)

- [x] `.env.example` criado na raiz com todas as 42 variáveis de ambiente documentadas: backend NestJS, frontend Next.js e mobile Expo — REQ-201
- [x] `docker-compose.yml` criado com serviços: PostgreSQL 17, Redis 7, RabbitMQ 4 — REQ-200
- [x] `supabase/` configurado com Supabase CLI para uso local — REQ-200
- [x] `.nvmrc` na raiz com valor `22` — REQ-200

### Guia de Contribuição (D23)

- [x] Git flow documentado: branches `feature/*`, `fix/*`, `develop`, `main` — REQ-202
- [x] Template de PR criado em `.github/pull_request_template.md` — REQ-202
- [x] commitlint configurado com Conventional Commits — REQ-202
- [x] README.md na raiz com instruções de setup em < 20 minutos — REQ-200

---

## 🔌 WIRING (Integração de camadas)

- [x] Health check endpoint `GET /health` respondendo `{ "status": "ok" }` com HTTP 200 e latência < 500ms — REQ-213
- [x] PrismaService conecta ao banco local e migrations aplicadas automaticamente no startup — REQ-147
- [x] Redis, RabbitMQ e PostgreSQL validados localmente via `docker compose up` + `pnpm dev` — REQ-200
- [x] Layout shell renderiza no browser em `http://localhost:3001` com sidebar e header funcionais — REQ-155
- [x] Auth guard redireciona para `/login` ao acessar rota autenticada sem token — REQ-013

---

## 🧪 TESTES

- [x] Testes unitários do `GlobalExceptionFilter`: erro 4XX retorna RFC 7807 com `cedente_message` correto — REQ-195
- [x] Testes de integração do `PrismaService`: conexão ao banco de dados de test funciona — REQ-147
- [x] Teste de migração: `prisma migrate deploy` executa sem erros em banco limpo — REQ-147
- [ ] Teste de RLS: query com `cedente_id` incorreto retorna 0 registros — REQ-144
- [ ] Cobertura ≥ 80% nos módulos `common/` (guards, filters, interceptors) — REQ-210

---

## ✅ AUTO-VERIFICAÇÃO (12 checks)

- [x] **C1** — Todos os 11 models Prisma estão no schema com campos, tipos, FKs e constraints corretos?
- [x] **C2** — Os 16 enums têm todos os valores listados nos docs?
- [x] **C3** — RLS habilitado e política por `cedente_id` validada com teste real?
- [x] **C4** — GlobalExceptionFilter retorna RFC 7807 em todos os cenários de erro?
- [x] **C5** — 8 chaves Redis com prefixo `rs:cedente:` e TTL explícito?
- [x] **C6** — 5 exchanges + 8 queues RabbitMQ com DLQ e retry configurados?
- [x] **C7** — Layout shell com sidebar, header e auth guard funcionando?
- [x] **C8** — 42 env vars documentadas no .env.example?
- [x] **C9** — Integrações ZapSign, Escrow, Supabase, Resend, OpenAI, Receita Federal, Langfuse, Sentry configuradas (health check)?
- [x] **C10** — Glossário (D10) consultado: `Caso`, `Cedente`, `Dossiê`, `Conta Escrow`, `Guardião do Retorno`, `CenarioRetorno (A/B/C/D)` refletidos nos enums e models?
- [x] **C11** — Pino logger com campos obrigatórios (requestId, userId, module, action) em toda request?
- [x] **C12** — Todos os REQs atribuídos a S1 (REQ-133 a REQ-159, REQ-169 infra, REQ-170-173, REQ-174-180, REQ-194-201, REQ-209) têm pelo menos 1 item de checklist?
