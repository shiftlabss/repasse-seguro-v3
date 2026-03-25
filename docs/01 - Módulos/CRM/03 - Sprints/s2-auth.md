# S2 — Auth

## CRM Repasse Seguro

| **Campo**         | **Valor**                                                                                                                                  |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sprint**        | S2                                                                                                                                         |
| **Nome**          | Auth                                                                                                                                       |
| **Template**      | A — Infraestrutura                                                                                                                         |
| **Docs fonte**    | 01.1, 05.1, 06, 16, 18                                                                                                                     |
| **REQs cobertos** | REQ-001 a REQ-005, REQ-010 a REQ-012, REQ-037, REQ-107, REQ-142 a REQ-144, REQ-179, REQ-181 a REQ-185, REQ-239, REQ-275 a REQ-276, REQ-304 |
| **Objetivo**      | Usuário consegue fazer login, ver dashboard vazio, logout funciona, roles são verificadas, telas protegidas redirecionam.                  |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS`, `PARCEIRO_EXTERNO`, `crm_users`, `Supabase Auth`
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: Valores exatos — 5 tentativas, 15 min, 30 min bloqueio, 60 min inatividade, 7 dias convite Parceiro, 48h convite interno, 2h link reset senha
- [x] C4: Glossário consultado — JWT HttpOnly cookie, Refresh Token, Rate Limiting, Enumeração de usuários incluídos
- [x] C5: Máquinas de estado de sessão cobertas
- [x] C6: TTLs de link (2h reset, 48h convite, 7 dias Parceiro) incluídos
- [x] C7: RBAC — 4 roles, matriz de permissões, redirecionamentos por papel
- [x] C8: T-CRM-001 com 4 estados de tela documentados
- [x] C9: Supabase Auth como provedor de identidade
- [x] C10: Glossário — JWT em HttpOnly cookie, Refresh Token rotação automática cobertos
- [x] C11: Anti-scaffold — validações, RBAC, error handling, testes por item
- [x] C12: 100% dos REQs de S2 com ≥1 item

---

## 🗄️ Banco de Dados

- [x] **REQ-239** Configurar 4 roles no Supabase Auth como custom claims no JWT: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS`, `PARCEIRO_EXTERNO`. Criar função Supabase `set_user_role(user_id, role)` que atualiza `raw_app_meta_data`
  - Validar: JWT de usuário criado contém `app_metadata.role = "ANALISTA_RS"`; chamada à função com UUID inválido → erro claro

- [x] **REQ-275** Configurar Supabase Auth para armazenar JWT em cookie `HttpOnly; Secure; SameSite=Lax` — NUNCA em `localStorage`. Usar `@supabase/ssr` (Next.js 15) — `@supabase/auth-helpers` v1 proibido
  - Validar: inspeção do browser não mostra JWT em `localStorage`; cookie `sb-access-token` tem flags `HttpOnly` e `Secure`

- [x] **REQ-276** Configurar Supabase Auth para rotação automática de Refresh Token. Tempo de expiração do access token: 60 minutos (alinhado com RN-003). Refresh token gerenciado pelo `@supabase/ssr` de forma transparente
  - Validar: access token expira após 60 min sem interação; com interação ativa, token renovado sem novo login

---

## 🔧 Backend — Auth

### Endpoints de Autenticação

- [x] **REQ-181** `POST /auth/login` — implementar com `@nestjs/throttler`: máximo 10 req/min por IP (REQ-139). Request body: `{ email: string, password: string }`. Fluxo:
  - Verificar credenciais via Supabase Auth
  - Sucesso: retornar `{ data: { access_token, refresh_token, user: { id, email, full_name, role } } }` 200
  - Falha: retornar erro genérico `CRM-001` "E-mail ou senha incorretos. Verifique seus dados e tente novamente." (NÃO revelar se e-mail existe — anti-enumeração)
  - Registrar tentativa no log com timestamp e IP
  - Validar: credenciais válidas → 200 com access_token; credenciais inválidas → 401 com `CRM-001`; resposta para e-mail inexistente idêntica à resposta para senha errada

- [x] **REQ-002 / REQ-182** Implementar bloqueio de conta após 5 tentativas incorretas em 15 minutos: usar Redis para contador `login_attempts:{email}` com TTL de 15 minutos. Após 5 falhas: marcar `account_locked:{email}` no Redis por 30 minutos, enviar e-mail de notificação via fila RabbitMQ, retornar `CRM-002`: "Sua conta está temporariamente bloqueada por segurança. Tente novamente em [tempo restante] ou redefina sua senha agora."
  - Validar: 5ª tentativa incorreta → conta bloqueada, Redis `account_locked:{email}` com TTL 1800s; 6ª tentativa → mensagem de bloqueio com countdown; após 30 min → Redis key expirada → login permitido; e-mail de notificação na fila

- [x] **REQ-003** Implementar expiração de sessão por inatividade: middleware NestJS que verifica timestamp da última ação do usuário (armazenado em Redis `session_last_active:{userId}` com TTL 3600s). Renovar TTL a cada requisição autenticada. JWT expira em 60 min via Supabase. Frontend monitora inatividade com `idleTimeout` no client
  - Validar: Redis `session_last_active:{userId}` renovado a cada request autenticado; TTL inicial 3600s; ausência de request por 3600s → Redis key expira

- [x] **REQ-183** `POST /auth/forgot-password` — Request body: `{ email: string }`. Fluxo:
  - Chamar Supabase Auth `resetPasswordForEmail(email)` com validade de 2 horas (REQ-005)
  - SEMPRE retornar a mesma mensagem `CRM-003`: "Um link de redefinição foi enviado, caso esse e-mail esteja cadastrado." — independentemente de o e-mail existir (anti-enumeração de usuários)
  - Sem rate limit diferenciado — usar o mesmo de 10 req/min por IP de auth
  - Validar: e-mail cadastrado → Supabase envia link + mensagem de confirmação; e-mail inexistente → MESMA mensagem de confirmação (sem erro 404); link gerado com expiração de 2h

- [x] **REQ-184** `POST /auth/reset-password` — Request body: `{ token: string, new_password: string }`. Fluxo:
  - Verificar token via Supabase Auth (rejeitar se expirado — após 2h)
  - Token válido: atualizar senha, retornar 200
  - Token expirado/inválido: retornar `CRM-004` 400: "Link inválido ou expirado. Solicite um novo link de redefinição."
  - Validar: token válido → senha atualizada; token expirado → `CRM-004`; token de formato inválido → 400

- [x] **REQ-185** `POST /auth/refresh` — Request body: `{ refresh_token: string }`. Retornar novo `access_token`. Token inválido → 401 com `CRM-005`. Usado pelo `@supabase/ssr` automaticamente
  - Validar: refresh token válido → novo access_token; refresh token inválido → 401

- [x] **REQ-304** Configurar guard NestJS `JwtAuthGuard` que valida Bearer JWT em toda requisição autenticada. Token expirado → 401 com `WWW-Authenticate: Bearer error="invalid_token"`. Extrair `userId` e `role` do payload JWT e injetar no request
  - Validar: endpoint com guard + token expirado → 401 com header WWW-Authenticate correto; endpoint com guard + sem token → 401

- [x] **REQ-037** Criar `RolesGuard` NestJS com decorator `@Roles(UserRole.ADMIN_RS, UserRole.COORDENADOR_RS, ...)`. Verificar `request.user.role` (extraído do JWT claim) em cada requisição. Role insuficiente → 403 `CRM-006`
  - Validar: endpoint `@Roles(ADMIN_RS)` com token de `ANALISTA_RS` → 403; com token de `ADMIN_RS` → 200

- [x] **REQ-107** Implementar middleware `RlsDataScopeMiddleware` que, para usuário com role `ANALISTA_RS`, injeta filtro automático `assigned_to = userId` em queries de `cases`. Para `PARCEIRO_EXTERNO`, injeta filtro de Casos em que é indicador. Para `ADMIN_RS` e `COORDENADOR_RS`, sem filtro adicional
  - Validar: `GET /cases` com token de `ANALISTA_RS` → retorna apenas Casos com `assigned_to = userId`; com token de `ADMIN_RS` → retorna todos

- [x] **REQ-010** `POST /team/invite` (stub — implementado em S9, mas criar rota aqui para convites de usuários internos). Link ativação via Supabase Auth `inviteUserByEmail()` com validade de 48 horas. E-mail duplicado → `CRM-007`
  - Validar: chamada `inviteUserByEmail` com e-mail válido → link enviado; e-mail duplicado → `CRM-007`; link expira após 48h no Supabase

- [x] **REQ-004** Configurar convite de Parceiro Externo: `POST /team/invite-partner` com link de ativação válido por 7 dias. Conta criada com role `PARCEIRO_EXTERNO`. Reenvio via `POST /team/resend-invite/:id`
  - Validar: link de Parceiro Externo expira em 7 dias (diferente dos 48h internos); conta criada com role correto; reenvio cancela link anterior

---

## 🎨 Frontend — Auth

### T-CRM-001 — Login / Autenticação

- [x] **REQ-142** Implementar `app/(auth)/login/page.tsx` (T-CRM-001) com:
  - Estado padrão: formulário com campos `email` (React Hook Form + Zod: `z.string().email()`) e `password` (Zod: `z.string().min(8)`) + botão "Entrar"
  - Estado de erro: mensagem "E-mail ou senha incorretos. Verifique seus dados e tente novamente." abaixo do campo de senha + campo de senha limpo automaticamente
  - Estado de sessão expirada: banner "Sua sessão expirou. Faça login novamente." (detectado via query param `?reason=expired`)
  - Estado de usuário suspenso: "Seu acesso foi temporariamente suspenso. Contate seu gestor." (detectado via `CRM-008` na resposta)
  - Estado de bloqueio: mensagem "Sua conta está temporariamente bloqueada." com countdown regressivo (lida de `retry_after` no response)
  - Validar: formulário com e-mail inválido → erro inline antes de submeter; submit com campos vazios → bloqueado; após erro de credenciais → campo senha limpo; campo senha com `type="password"` + toggle de visibilidade

- [x] **REQ-001** Implementar redirecionamento por papel após login bem-sucedido:
  - `ADMIN_RS` → `/dashboard` (visão geral com métricas executivas)
  - `COORDENADOR_RS` → `/dashboard` (visão painel da equipe)
  - `ANALISTA_RS` → `/pipeline` (fila de Casos próprios)
  - `PARCEIRO_EXTERNO` → `/partner/cases` (portal externo — fora do layout principal)
  - Preservar URL de retorno (`?return=`) se vier de redirect por sessão expirada
  - Validar: login com `ANALISTA_RS` → redireciona para `/pipeline`; login com URL de retorno `?return=/cases/123` → redireciona para `/cases/123` após sucesso

### T-CRM-002 — Recuperação de Senha

- [x] **REQ-143** Implementar `app/(auth)/forgot-password/page.tsx` (T-CRM-002) com:
  - Campo de e-mail com validação Zod `z.string().email()`
  - Após submit: SEMPRE exibir "Um link de redefinição foi enviado, caso esse e-mail esteja cadastrado." (independentemente de o e-mail existir)
  - Loading state com skeleton no botão (sem spinner global)
  - Validar: e-mail válido ou inválido → mesma mensagem de confirmação; nenhuma indicação de existência de conta; botão "Voltar ao login" disponível

### T-CRM-003 — Redefinição de Senha

- [x] **REQ-144** Implementar `app/(auth)/reset-password/page.tsx` (T-CRM-003) com:
  - Campos: `new_password` (mínimo 8 caracteres, 1 maiúscula, 1 número) e `confirm_password` com validação de igualdade via Zod refine
  - Token expirado: exibir mensagem de erro com link para nova solicitação
  - Sucesso: redirecionar para `/login` com mensagem "Senha redefinida com sucesso."
  - Validar: senha fraca → erro inline antes de submeter; senhas divergentes → erro antes de submeter; token expirado → mensagem com CTA para novo link

### T-CRM-113 — Perfil do Usuário (pessoal)

- [x] **REQ-179** Implementar `app/(dashboard)/settings/profile/page.tsx` (T-CRM-113) com:
  - Exibir: nome completo, e-mail (readonly), papel (readonly), avatar (upload opcional com preview)
  - Editar: nome completo, telefone de contato, foto de perfil (upload via Supabase Storage)
  - Alterar senha: formulário separado com `current_password`, `new_password`, `confirm_password`
  - Configurações de notificação: desativar notificações de prioridade Baixa e Média (conforme RN-195 — Crítica e Alta não desativáveis)
  - Validar: e-mail não editável; papel não editável pelo usuário; atualização de nome → refletida no avatar dropdown da topbar imediatamente; notificações Crítica e Alta sem opção de desabilitar na UI

### Middleware de Auth e Proteção de Rotas

- [x] **REQ-012** Implementar `middleware.ts` do Next.js para proteger rotas do grupo `(dashboard)`: verificar cookie de sessão Supabase Auth via `@supabase/ssr`. Sem sessão → redirect para `/login?return=[url]`. Com sessão mas role insuficiente → redirect para `/forbidden`
  - Validar: acessar `/pipeline` sem autenticação → redirect `/login?return=/pipeline`; acessar `/settings/parameters` com role `ANALISTA_RS` → redirect `/forbidden`

- [x] **REQ-003** Implementar lógica de inatividade no frontend em `lib/session-manager.ts`: detectar ausência de eventos de mouse/teclado/toque por 60 minutos, exibir modal "Sua sessão expira em 5 minutos por inatividade. Clique em qualquer lugar para continuar." 5 minutos antes. Após 5 min sem interação → chamar `supabase.auth.signOut()` e redirecionar para `/login?reason=expired` preservando URL de retorno
  - Validar: modal de aviso aparece aos 55 min de inatividade; clique no modal cancela o countdown; aos 60 min → signOut + redirect para login com banner de sessão expirada

- [x] **REQ-011** Implementar tour interativo de onboarding em `components/onboarding/OnboardingTour.tsx`:
  - Exibido automaticamente na primeira sessão após ativação de conta (checar flag `has_completed_onboarding` no perfil do usuário)
  - 5 etapas: (1) criação de Caso com tooltip no botão "Novo Caso", (2) registro de Atividade, (3) avanço de estado, (4) Dashboard pessoal, (5) central de ajuda
  - Botão "Pular" a qualquer momento → registrar `PATCH /team/:id { has_completed_onboarding: true, skipped_onboarding: true }`
  - Tour NÃO bloqueia uso do CRM (modal não-modal, overlay translúcido, CRM interagível por trás)
  - Disponível novamente em "Ajuda → Tour guiado" (link na sidebar)
  - Validar: primeira sessão → tour exibe automaticamente; "Pular" → tour fecha + flag registrada; nova sessão após pular → tour não exibe; "Ajuda → Tour guiado" → tour reinicia

---

## 🔒 LGPD / Mascaramento

- [x] **REQ-012** Implementar `lib/mask-data.ts` com funções:
  - `maskName(fullName: string): string` → retorna "João S." (primeiro nome + inicial sobrenome)
  - `maskCpf(cpf: string): string` → retorna "**_.XXX._**-\*\*" com 3 dígitos centrais visíveis
  - `maskEmail(email: string): string` → retorna "j\*\*\*@gmail.com"
  - Aplicar automaticamente em: listagens de Pipeline, resultados de busca global, dashboards
  - NUNCA mascarar na tela de detalhe do Caso quando o Analista RS responsável está visualizando
  - Validar: `maskName("João da Silva")` → "João S."; `maskCpf("123.456.789-00")` → "**_.456._**-**"; `maskEmail("joao@gmail.com")` → "j\***@gmail.com"

---

## 🧪 Testes

### Backend — Unitário

- [x] **REQ-002** Teste unitário `AuthService` — bloqueio de conta:
  - `should block account after 5 failed attempts within 15 minutes`
  - `should not block account after 4 failed attempts`
  - `should return remaining lockout time in error message`
  - `should unblock account after 30 minutes`
  - Validar: 100% branches do `AuthService.checkLoginAttempts()` cobertos

### Backend — Integração

- [x] **REQ-001** Teste de integração `POST /auth/login`:
  - Happy path: credenciais válidas → 200, JWT no body, cookie HttpOnly
  - Credenciais inválidas: `CRM-001` 401
  - Conta bloqueada: `CRM-002` 401 com `retry_after`
  - Role check: `ANALISTA_RS` logado → redirect correto simulado

- [x] **REQ-005** Teste de integração `POST /auth/forgot-password`:
  - E-mail cadastrado: 200 com mensagem genérica
  - E-mail inexistente: 200 com MESMA mensagem genérica (anti-enumeração)
  - Validar: resposta idêntica para os dois cenários

### Frontend — Unitário (Vitest)

- [x] Teste `maskName`, `maskCpf`, `maskEmail` em `lib/mask-data.test.ts`:
  - Casos: nomes com 1 palavra, 2 palavras, 3 palavras+
  - CPFs com/sem formatação
  - Validar: 100% branches

---

## 🔀 Cross-Módulo

- A lógica de RBAC (`JwtAuthGuard`, `RolesGuard`, `RlsDataScopeMiddleware`) implementada aqui é consumida por TODOS os módulos de S3 em diante. Qualquer módulo que exija permissão específica deve usar `@Roles()` decorator.
- O mascaramento de dados (`maskName`, `maskCpf`) é consumido pelo módulo Pipeline (S3) e Contatos (S5).

---

## ✅ Critério de Conclusão da S2

1. Login com `ANALISTA_RS` → redireciona para `/pipeline`
2. 5 tentativas inválidas em 15 min → conta bloqueada por 30 min
3. Sessão expira após 60 min de inatividade
4. `PARCEIRO_EXTERNO` não acessa `/settings` (403)
5. Link de reset senha expira em 2h
6. Tour de onboarding exibe na 1ª sessão
7. Busca global mascara dados pessoais nos resultados
