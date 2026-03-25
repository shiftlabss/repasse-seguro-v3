# S2 — Auth

## Módulo Cedente · Plataforma Repasse Seguro

| **Sprint** | **Tipo**              | **Template** | **Versão** | **Data**   |
| ---------- | --------------------- | ------------ | ---------- | ---------- |
| S2         | Infraestrutura (Auth) | Template A   | v1.0       | 2026-03-24 |

**REQs cobertos:** REQ-001 a REQ-021, REQ-109–110, REQ-126, REQ-160–168, REQ-181–184

**Dependências:** S1 concluída (banco, NestJS scaffold, Supabase configurado, Redis operacional, Resend configurado)

**Critério de aceite da sprint:** todos os itens ✅ + 12 auto-verificações ✅ + T-038 (isolamento RLS) passando.

---

## 🗄️ BANCO DE DADOS

### 1. Tabela `users` — extensão para autenticação

- [x] **1.1** Confirmar que a tabela `users` criada na S1 possui os campos: `id UUID PK DEFAULT gen_random_uuid()`, `auth_id UUID UNIQUE NOT NULL` (referência ao `auth.users.id` do Supabase), `email VARCHAR(255) UNIQUE NOT NULL`, `nome VARCHAR(255) NOT NULL`, `tipo_cedente TipoCedente NOT NULL` (`PF` | `PJ`), `status StatusConta NOT NULL DEFAULT 'PENDING_ACTIVATION'`, `email_verificado BOOLEAN NOT NULL DEFAULT false`, `ultimo_login TIMESTAMPTZ`, `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ DEFAULT now()`.
- [x] **1.2** Confirmar que a tabela `cedentes` criada na S1 possui FK `user_id UUID REFERENCES users(id) ON DELETE CASCADE` e campo `cpf VARCHAR(14) UNIQUE` e `cnpj VARCHAR(18) UNIQUE` (apenas um preenchido por linha, conforme `tipo_cedente`).
- [x] **1.3** Confirmar que o enum `StatusConta` possui exatamente os valores: `PENDING_ACTIVATION`, `ACTIVE`, `BLOCKED`, `EXPIRED`, `DELETED` — sem adicionar ou remover valores.
- [x] **1.4** Confirmar que o enum `TipoCedente` possui exatamente os valores: `PF`, `PJ`.
- [x] **1.5** Criar índice `idx_users_auth_id ON users(auth_id)` para lookup rápido de sessão Supabase → usuário da aplicação.
- [x] **1.6** Criar índice `idx_users_email ON users(email)` para validação de unicidade e rate limit de login.
- [x] **1.7** Confirmar que RLS está habilitada na tabela `users`: `ALTER TABLE users ENABLE ROW LEVEL SECURITY`.
- [x] **1.8** Criar política RLS: `CREATE POLICY "cedente_ver_proprio_perfil" ON users FOR SELECT USING (auth.uid() = auth_id)` — Cedente só vê seu próprio registro.
- [x] **1.9** Criar política RLS: `CREATE POLICY "cedente_atualizar_proprio_perfil" ON users FOR UPDATE USING (auth.uid() = auth_id) WITH CHECK (auth.uid() = auth_id)` — Cedente só atualiza seu próprio registro.
- [x] **1.10** Garantir que `cpf` na tabela `cedentes` possui restrição UNIQUE — RN-001 (CPF único na plataforma).
- [x] **1.11** Garantir que `cnpj` na tabela `cedentes` possui restrição UNIQUE — RN-013.a (CNPJ único na plataforma).
- [x] **1.12** Criar migration `0002_auth_indexes_rls.sql` aplicando todos os itens 1.5–1.11 acima que não estejam na migration inicial.
- [x] **1.13** Verificar que a migration é idempotente: rodar `prisma migrate deploy` duas vezes não gera erro.

### 2. Redis — chaves de autenticação

- [x] **2.1** Confirmar que o prefixo Redis para auth é `rs:cedente:auth:` conforme definido em `15 - Arquitetura de Pastas.md`.
- [x] **2.2** Implementar chave `rs:cedente:auth:rate_limit:login:{email}` com TTL de 15 minutos — contador de tentativas de login falhas (RN-005, D18 seção 4.2).
- [x] **2.3** Implementar chave `rs:cedente:auth:session:inactivity:{user_id}` com TTL de 30 minutos — timestamp de última atividade para expiração por inatividade (D18 seção 6.4).
- [x] **2.4** Implementar chave `rs:cedente:auth:email_change_pending:{user_id}` com TTL de 48 horas — armazena `{ new_email, token }` durante troca de e-mail pendente (RN-008).
- [x] **2.5** Confirmar que todas as operações Redis utilizam o cliente Upstash configurado na S1 (`@upstash/redis`).

---

## ⚙️ BACKEND — Módulo Auth (NestJS)

### 3. Estrutura de pastas (conforme D15)

- [x] **3.1** Criar `apps/api/src/modules/auth/auth.module.ts` com importações: `SupabaseModule`, `PrismaModule`, `RedisModule`, `PassportModule`, `JwtModule`.
- [x] **3.2** Criar `apps/api/src/modules/auth/auth.controller.ts` com rotas: `POST /auth/cadastro`, `GET /auth/ativar-conta`, `POST /auth/reenviar-ativacao`, `POST /auth/login`, `POST /auth/logout`, `POST /auth/refresh`, `POST /auth/recuperar-senha`, `POST /auth/redefinir-senha`, `POST /auth/reauth` — totalizando 9 endpoints conforme D16 domínio Auth.
- [x] **3.3** Criar `apps/api/src/modules/auth/auth.service.ts` contendo toda a lógica de negócio de autenticação (sem lógica no controller).
- [x] **3.4** Criar `apps/api/src/modules/auth/guards/jwt-auth.guard.ts` — extrai e valida JWT Bearer em todas as rotas protegidas; retorna `AUTH_004` se token inválido ou expirado.
- [x] **3.5** Criar `apps/api/src/modules/auth/guards/rate-limit.guard.ts` — aplica limite de 5 tentativas em `POST /auth/login` via Redis; retorna HTTP 429 com código `AUTH_005` após 5 falhas; inclui `Retry-After` header com segundos restantes.
- [x] **3.6** Criar `apps/api/src/modules/auth/strategies/jwt.strategy.ts` — Passport JWT strategy que valida `alg: HS256` (nunca `alg: none`), verifica `exp`, extrai `sub` como `auth_id`.
- [x] **3.7** Criar `apps/api/src/modules/auth/dto/cadastro.dto.ts` com validações via `class-validator`:
  - `nome: string` — `@IsString() @MinLength(3) @MaxLength(255)`
  - `email: string` — `@IsEmail()`
  - `senha: string` — `@IsString() @MinLength(8) @Matches(/^(?=.*[A-Z])(?=.*\d)/, { message: 'CADASTRO_005: Senha fraca' })`
  - `cpf?: string` — `@IsOptional() @IsCPF()` (quando `tipo_cedente = PF`)
  - `cnpj?: string` — `@IsOptional() @IsCNPJ()` (quando `tipo_cedente = PJ`)
  - `tipo_cedente: TipoCedente` — `@IsEnum(TipoCedente)`
  - `telefone: string` — `@IsString() @Matches(/^\+55\d{10,11}$/)`
- [x] **3.8** Criar `apps/api/src/modules/auth/dto/login.dto.ts` com `email: string @IsEmail()` e `senha: string @IsString()`.
- [x] **3.9** Criar `apps/api/src/modules/auth/dto/recuperar-senha.dto.ts` com `email: string @IsEmail()`.
- [x] **3.10** Criar `apps/api/src/modules/auth/dto/redefinir-senha.dto.ts` com `token: string @IsString()`, `nova_senha: string @IsString() @MinLength(8) @Matches(/^(?=.*[A-Z])(?=.*\d)/)`, `confirmacao_senha: string @IsString()` + validação custom que `nova_senha === confirmacao_senha`.

### 4. Endpoint POST /auth/cadastro

- [x] **4.1** Validar CPF (dígitos verificadores) e unicidade na tabela `cedentes` — retornar `CADASTRO_003: CPF inválido` ou `CADASTRO_004: CPF já cadastrado` (RN-001).
- [x] **4.2** Validar CNPJ (formato + dígitos) e unicidade na tabela `cedentes` quando `tipo_cedente = PJ` — retornar `CADASTRO_006: CNPJ inválido ou irregular` (RN-013.a).
- [x] **4.3** Para Cedente PJ: chamar Receita Federal API (`RECEITA_FEDERAL_API_URL`) para validar CNPJ ativo; se a API falhar, usar fallback de validação de dígitos verificadores localmente e marcar `⚠️ validação RF pendente` (REQ-126, D17).
- [x] **4.4** Validar e-mail formato RFC 5322 e unicidade na tabela `users` — retornar `CADASTRO_001: E-mail inválido` ou `CADASTRO_002: E-mail já cadastrado`.
- [x] **4.5** Chamar `supabase.auth.signUp({ email, password })` para criar usuário no Supabase Auth.
- [x] **4.6** Em caso de sucesso no Supabase Auth: fazer `INSERT` em `users` com `auth_id = supabase_user.id`, `status = PENDING_ACTIVATION`, `email_verificado = false`; fazer `INSERT` em `cedentes` com `user_id`, CPF ou CNPJ e demais dados.
- [x] **4.7** Em caso de falha no Supabase Auth após INSERT local: realizar rollback da transação (usar `prisma.$transaction`).
- [x] **4.8** Enviar e-mail via Resend usando template `NOT-CED-01` (cedente_boas_vindas) com link de ativação de validade **48 horas** (RN-001 item 3, RN-002).
- [x] **4.9** Retornar HTTP 201 `{ success: true, message: "Verifique seu e-mail para ativar a conta" }` — nunca retornar dados sensíveis no response.
- [x] **4.10** Retornar erros no formato RFC 7807: `{ type, title, status, detail, cedente_message, instance, code, timestamp, trace_id }` — `cedente_message` é o texto amigável exibido na UI, `detail` é técnico (nunca exibido ao usuário).

### 5. Endpoint GET /auth/ativar-conta

- [x] **5.1** Receber `token` via query param; chamar `supabase.auth.verifyOtp({ token, type: 'signup' })`.
- [x] **5.2** Em caso de token válido: atualizar `users SET status = ACTIVE, email_verificado = true, updated_at = now()` (RN-002 item 3).
- [x] **5.3** Em caso de token expirado (>48h — RN-002 item 4): retornar HTTP 400 com código `AUTH_007: Link de ativação expirado` e `cedente_message: "O link de ativação expirou. Solicite um novo."`.
- [x] **5.4** Redirecionar para `NEXT_PUBLIC_APP_URL/dashboard` após ativação bem-sucedida (ou retornar JSON para o frontend redirecionar).
- [x] **5.5** Verificar que conta com `status = EXPIRED` (>30 dias sem ativação) permite novo cadastro com o mesmo e-mail — implementar lógica de cleanup em job cron (REQ-003, D18 seção 3.3).

### 6. Endpoint POST /auth/reenviar-ativacao

- [x] **6.1** Receber `{ email }` no body; verificar se existe `users` com `email` e `status = PENDING_ACTIVATION`.
- [x] **6.2** Se conta existe: chamar `supabase.auth.resend({ type: 'signup', email })`; gerar novo link (TTL 48h); invalidar link anterior.
- [x] **6.3** Aplicar cooldown de 60 segundos entre reenvios: verificar chave Redis `rs:cedente:auth:resend_cooldown:{email}` — retornar HTTP 429 se chave existir, com `Retry-After` header indicando segundos restantes (RN-003 item 3).
- [x] **6.4** Se conta não existe ou já está ativa: retornar HTTP 200 com mensagem genérica `"Não encontramos uma conta pendente com este e-mail."` — não revelar existência ou status da conta.
- [x] **6.5** Retornar HTTP 200 `{ success: true }` em caso de reenvio bem-sucedido.

### 7. Endpoint POST /auth/login

- [x] **7.1** Verificar chave Redis `rs:cedente:auth:rate_limit:login:{email}` antes de chamar Supabase Auth — se valor ≥ 5, retornar HTTP 429 com `AUTH_005`, tempo restante de bloqueio e `Retry-After` header (RN-005).
- [x] **7.2** Chamar `supabase.auth.signInWithPassword({ email, password })`.
- [x] **7.3** Em caso de credenciais incorretas: `INCR rs:cedente:auth:rate_limit:login:{email}` com TTL 15min; retornar HTTP 401 com `AUTH_001` e campo `tentativas_restantes: 5 - contador_atual`; a partir da 3ª tentativa incluir `warning: "Você tem mais [X] tentativas antes do bloqueio temporário."` (RN-005 item 4).
- [x] **7.4** Verificar `users.status`: se `PENDING_ACTIVATION` retornar `AUTH_002`; se `BLOCKED` retornar `AUTH_003`; se `EXPIRED` retornar HTTP 403 com mensagem orientando recadastro.
- [x] **7.5** Em caso de login bem-sucedido: `DEL rs:cedente:auth:rate_limit:login:{email}` (limpar contador); `UPDATE users SET ultimo_login = now()`.
- [x] **7.6** Retornar access token JWT (válido **1 hora**) no body: `{ access_token, expires_in: 3600, token_type: 'Bearer', cedente: { id, nome, email, tipo_pessoa, status: 'ACTIVE' } }`.
- [x] **7.7** Definir cookie `refresh_token` com atributos: `httpOnly: true`, `Secure: true`, `SameSite: Strict`, `Max-Age: 604800` (7 dias) — token nunca no response body, apenas no cookie (D18 seção 4.3).
- [x] **7.8** Access token NUNCA armazenado em localStorage ou sessionStorage no frontend — deve ser armazenado apenas em memória (Zustand store) — documentar isso no README do módulo auth (D18 seção 6.1).

### 8. Endpoint POST /auth/logout

- [x] **8.1** Exigir Bearer token válido no header `Authorization`.
- [x] **8.2** Chamar `supabase.auth.signOut()` — invalida sessão no Supabase server-side.
- [x] **8.3** Remover cookie `refresh_token` setando `Max-Age: 0`.
- [x] **8.4** Deletar chave Redis `rs:cedente:auth:session:inactivity:{user_id}`.
- [x] **8.5** Retornar HTTP 204 No Content.
- [x] **8.6** Suportar logout forçado por Admin: endpoint interno `POST /auth/logout-force/:user_id` (sem bearer do usuário, com service role) que invalida todos os tokens do usuário via `supabase.auth.admin.signOut(user_id)`.

### 9. Endpoint POST /auth/refresh

- [x] **9.1** Ler `refresh_token` do httpOnly cookie (web) — NÃO do body para o endpoint web.
- [x] **9.2** Criar endpoint separado `POST /auth/refresh-mobile` que aceita `{ refresh_token }` no body — exclusivo para o app mobile (D18 seção 8.2).
- [x] **9.3** Chamar Supabase Auth para renovar o refresh token (sliding window — novo token a cada refresh).
- [x] **9.4** Em caso de refresh token expirado (>7 dias): retornar HTTP 401 com `AUTH_006`; não definir novo cookie.
- [x] **9.5** Em caso de sucesso: retornar `{ access_token, expires_in: 3600, token_type: 'Bearer' }` + novo cookie `refresh_token` com `Max-Age: 604800`.
- [x] **9.6** Implementar interceptor no cliente (`apps/api/src/common/interceptors/token-refresh.interceptor.ts`) que, ao receber HTTP 401 com `AUTH_004`, tenta refresh automático uma vez antes de propagar o erro.

### 10. Endpoint POST /auth/recuperar-senha

- [x] **10.1** Receber `{ email }` no body.
- [x] **10.2** Chamar `supabase.auth.resetPasswordForEmail(email, { redirectTo: NEXT_PUBLIC_APP_URL/auth/redefinir-senha })`.
- [x] **10.3** SEMPRE retornar HTTP 200 `{ message: "Se o e-mail estiver cadastrado, você receberá as instruções." }` — resposta genérica que não revela se e-mail existe ou não (OWASP A01, RN-004 item 4).
- [x] **10.4** Link de recuperação com TTL de **1 hora** (configurado no Supabase Auth Dashboard — confirmar configuração).
- [x] **10.5** Aplicar rate limit de 3 solicitações por hora por e-mail via Redis: chave `rs:cedente:auth:pw_reset:{email}` TTL 3600s.

### 11. Endpoint POST /auth/redefinir-senha

- [x] **11.1** Receber `{ token, nova_senha, confirmacao_senha }` no body.
- [x] **11.2** Chamar `supabase.auth.verifyOtp({ token, type: 'recovery' })` para obter sessão.
- [x] **11.3** Em caso de token expirado: retornar HTTP 400 com `AUTH_008: Link de recuperação expirado`.
- [x] **11.4** Em caso de token válido: chamar `supabase.auth.updateUser({ password: nova_senha })`.
- [x] **11.5** Retornar HTTP 200 `{ success: true, message: "Senha redefinida com sucesso. Faça login com sua nova senha." }`.

### 12. Endpoint POST /auth/reauth

- [x] **12.1** Receber `{ senha }` no body (reautenticação por senha) ou `{ biometric: true }` (reautenticação por biometria — apenas mobile).
- [x] **12.2** Validar senha atual via Supabase Auth sem criar nova sessão (`signInWithPassword` + verificação apenas).
- [x] **12.3** Retornar HTTP 200 `{ success: true, reauth_token: string }` — token de curta duração (5 minutos) para confirmar ações sensíveis.
- [x] **12.4** Usar reauth para ações que exijam confirmação explícita: solicitação de exclusão de dados LGPD (RN-010), troca de e-mail (RN-008).

### 13. Gestão de Sessão e Inatividade

- [x] **13.1** Implementar middleware NestJS `InactivityMiddleware` que, em cada requisição autenticada bem-sucedida, atualiza `rs:cedente:auth:session:inactivity:{user_id}` com timestamp atual e TTL de **30 minutos** (D18 seção 6.4).
- [x] **13.2** Implementar job cron `InactivityCheckCron` que roda a cada 5 minutos, verifica tokens com `rs:cedente:auth:session:inactivity` prestes a expirar (entre 25 e 30 minutos) e publica evento `SESSION_EXPIRING_SOON` no RabbitMQ exchange `cedente.notifications` para o frontend exibir o modal de aviso (D18 seção 6.3 e 6.4).
- [x] **13.3** Implementar lembrete automático de ativação: job cron que roda diariamente às 10h BRT verificando `users` com `status = PENDING_ACTIVATION` criados há 3 e 7 dias — enviar template `NOT-CED-01` (lembrete); após 30 dias marcar `status = EXPIRED` (D18 seção 3.3).

### 14. Perfil — Edição e Dados Sensíveis

- [x] **14.1** Implementar `PATCH /cedentes/me/perfil` no módulo `cedentes` (não no módulo `auth`) com os campos editáveis: `nome`, `telefone`, `preferencias_notificacao`.
- [x] **14.2** Garantir que `cpf` e `cnpj` são campos `readonly` na DTO de atualização — qualquer tentativa de atualizar CPF/CNPJ retorna HTTP 403 com mensagem `"CPF/CNPJ não pode ser alterado diretamente. Entre em contato com nosso suporte para correções."` (RN-007).
- [x] **14.3** Implementar fluxo de troca de e-mail em duas etapas (RN-008):
  - `PATCH /cedentes/me/email` inicia a troca: valida novo e-mail (formato + unicidade), armazena `{ new_email, token }` em Redis com TTL 48h (`rs:cedente:auth:email_change_pending:{user_id}`), envia link de confirmação para o novo e-mail.
  - `GET /cedentes/me/email/confirmar?token=xxx` confirma: valida token Redis, chama `supabase.auth.updateUser({ email: new_email })`, atualiza `users.email`, limpa chave Redis.
  - Enquanto troca pendente: exibir na UI `{ email_atual, email_novo_pendente, expira_em }`.
  - Endpoint `DELETE /cedentes/me/email/cancelar` cancela troca pendente (limpa Redis, mantém e-mail atual).
- [x] **14.4** Implementar `POST /cedentes/me/lgpd/solicitar-exclusao` (RN-010):
  - Exigir reauth token válido (5min) no header `X-Reauth-Token`.
  - Registrar solicitação com `data_solicitacao`, `protocolo` (UUID), status `PENDING`.
  - Retornar `{ protocolo, prazo_ate: data+15dias }`.
  - SLA: processamento em até **15 dias corridos**.
  - Endpoint `DELETE /cedentes/me/lgpd/cancelar-exclusao` para cancelar durante o prazo.

### 15. RLS — Políticas de Isolamento

- [x] **15.1** Criar política RLS em `casos`: `CREATE POLICY "cedente_ver_proprios_casos" ON casos FOR SELECT USING (auth.uid() = cedente_auth_id)` (D18 seção 7.2).
- [x] **15.2** Criar política RLS em `casos`: `CREATE POLICY "cedente_criar_caso" ON casos FOR INSERT WITH CHECK (auth.uid() = cedente_auth_id)`.
- [x] **15.3** NÃO criar políticas de UPDATE/DELETE em `casos` para o Cedente — apenas via API com validações de negócio (D18 seção 7.2).
- [x] **15.4** Criar política RLS em `documentos_dossie`: `FOR SELECT USING (auth.uid() = (SELECT cedente_auth_id FROM casos WHERE id = caso_id))`.
- [x] **15.5** Criar política RLS em `documentos_dossie`: `FOR INSERT WITH CHECK (auth.uid() = (SELECT cedente_auth_id FROM casos WHERE id = caso_id))`.
- [x] **15.6** Criar política RLS em `propostas`: `FOR SELECT USING (auth.uid() = (SELECT cedente_auth_id FROM casos WHERE id = caso_id))` — Cedente nunca vê `cessionario_id` (RN-012).
- [x] **15.7** Criar política RLS em `notificacoes`: `FOR ALL USING (auth.uid() = destinatario_auth_id)`.
- [x] **15.8** Criar política RLS em `ai_sessions`: `FOR ALL USING (auth.uid() = cedente_auth_id)`.
- [x] **15.9** Criar migration `0003_rls_policies.sql` com todas as políticas RLS dos itens 15.1–15.8.

---

## 🖥️ FRONTEND — Módulo Auth (Next.js App Router)

### 16. Estrutura de arquivos auth (conforme D15)

- [x] **16.1** Criar `apps/web-cedente/src/app/(public)/cadastro/page.tsx` — tela de cadastro.
- [x] **16.2** Criar `apps/web-cedente/src/app/(public)/login/page.tsx` — tela de login.
- [x] **16.3** Criar `apps/web-cedente/src/app/(public)/ativar-conta/page.tsx` — tela de ativação (recebe token da URL).
- [x] **16.4** Criar `apps/web-cedente/src/app/(public)/recuperar-senha/page.tsx` — tela "Esqueci minha senha".
- [x] **16.5** Criar `apps/web-cedente/src/app/(public)/redefinir-senha/page.tsx` — tela "Criar nova senha".
- [x] **16.6** Criar `apps/web-cedente/src/services/api.ts` — fetch wrapper com interceptor de token refresh automático: ao receber HTTP 401, tenta `POST /auth/refresh` uma vez; se falhar, realiza logout forçado (limpa Zustand + redireciona para `/login`) (D18 seção 6.2).
- [x] **16.7** Criar `apps/web-cedente/src/services/auth.ts` — wrappers: `cadastrar()`, `login()`, `logout()`, `refresh()`, `recuperarSenha()`, `redefinirSenha()`.
- [x] **16.8** Criar `apps/web-cedente/src/stores/auth.store.ts` — Zustand store com: `access_token: string | null`, `expires_at: number | null`, `cedente: { id, nome, email, tipo_pessoa } | null`; actions: `setSession()`, `clearSession()`.
- [x] **16.9** Criar `apps/web-cedente/src/hooks/useAuth.ts` — hook que expõe `isAuthenticated`, `cedente`, `login()`, `logout()`, `isLoading`.
- [x] **16.10** Criar `apps/web-cedente/src/middleware.ts` — Next.js middleware protegendo rotas `(authenticated)`: verifica cookie `refresh_token`; se ausente, redireciona para `/login` com `?redirect=` original (D18 seção 13.3).

### 17. Formulário de Cadastro

- [x] **17.1** Implementar formulário com campos: nome completo, tipo_cedente (toggle PF/PJ), CPF (se PF) ou CNPJ (se PJ), e-mail, telefone, senha, confirmação de senha.
- [x] **17.2** Validação inline on-blur em cada campo: indicador verde (válido) ou vermelho com mensagem específica por campo (RN-001 item 5).
- [x] **17.3** Ao submeter: botão "Cadastrar" exibe spinner com texto "Verificando dados..." e fica desabilitado até resposta (RN-001 item 6).
- [x] **17.4** Em caso de sucesso (HTTP 201): exibir tela intermediária "Verifique seu e-mail" com e-mail mascarado (`u***@dominio.com`) e instrução de prazo de 48h.
- [x] **17.5** Em caso de erro do servidor: exibir `cedente_message` do response RFC 7807 no campo específico ou em toast se for erro geral.
- [x] **17.6** Para Cedente PJ: ao digitar CNPJ completo, chamar `GET /cedentes/cnpj/:cnpj/validar` para pré-validação inline com feedback visual.
- [x] **17.7** Botão "Reenviar e-mail" na tela de aguardo: desabilitado por 60 segundos com timer regressivo visível "Reenviar disponível em [X]s" (RN-003 item 3).

### 18. Formulário de Login

- [x] **18.1** Implementar formulário com campos: e-mail, senha, checkbox "Lembrar por 7 dias".
- [x] **18.2** Em caso de erro HTTP 401 `AUTH_001`: exibir `cedente_message`; a partir da 3ª tentativa exibir alerta amarelo com `tentativas_restantes` e aviso de bloqueio iminente (RN-005 item 4).
- [x] **18.3** Em caso de HTTP 429 `AUTH_005`: exibir timer regressivo visual em vermelho contando os minutos até o desbloqueio (usando `Retry-After` do header) (RN-005 item 3).
- [x] **18.4** Em caso de HTTP 403 `AUTH_002` (conta não ativada): exibir link "Reenviar e-mail de ativação" com ação `POST /auth/reenviar-ativacao`.
- [x] **18.5** Após login bem-sucedido: armazenar `access_token` + `expires_at` no Zustand store (NUNCA em localStorage) e redirecionar para `/dashboard` (ou para `redirect` query param).

### 19. Modal de Aviso de Expiração de Sessão

- [x] **19.1** Implementar `SessionExpiryModal` em `apps/web-cedente/src/components/common/SessionExpiryModal.tsx` — componente não-bloqueante.
- [x] **19.2** Disparar modal quando Supabase Realtime receber evento `SESSION_EXPIRING_SOON` para o `user_id` da sessão atual (ou quando o timer calculado chegar a 5 minutos antes do `expires_at` do access token) (D18 seção 6.3).
- [x] **19.3** Texto do modal: `"Sua sessão está prestes a expirar em 5 minutos. Salve qualquer alteração em andamento ou clique em 'Continuar' para manter a sessão ativa."`.
- [x] **19.4** Botão "Continuar": chama `POST /auth/refresh`, atualiza Zustand store com novo `access_token` e `expires_at`, fecha modal.
- [x] **19.5** Se Cedente ignorar modal: deixar expirar normalmente — próxima requisição aciona refresh automático pelo interceptor.

### 20. Tela Meu Perfil — Dados de Conta

- [x] **20.1** Criar `apps/web-cedente/src/app/(authenticated)/perfil/page.tsx`.
- [x] **20.2** Exibir CPF/CNPJ em campo `readonly` com ícone de cadeado e tooltip `"CPF/CNPJ não pode ser alterado diretamente. Entre em contato com nosso suporte para correções."` (RN-007).
- [x] **20.3** Exibir seção de troca de e-mail: e-mail atual com badge "Ativo"; se há troca pendente, exibir e-mail novo com badge "Pendente de confirmação" + contador de expiração 48h + botão "Cancelar troca" (RN-008).
- [x] **20.4** Seção "Notificações": listar categorias desativáveis (documentos pendentes, proposta expirando, assinatura pendente, rascunho expirando) com toggle; categorias obrigatórias (nova proposta recebida, fechamento confirmado, distribuição realizada) com toggle cinza desabilitado + ícone cadeado + tooltip `"Esta notificação é obrigatória"` (RN-009).
- [x] **20.5** Seção "Exclusão de dados (LGPD)": botão "Solicitar exclusão dos meus dados" abre modal de confirmação com aviso de irreversibilidade; exige reautenticação (senha) antes de confirmar (RN-010).
- [x] **20.6** Se há solicitação LGPD em andamento: exibir banner `"Sua solicitação de exclusão está em processamento (protocolo [número]). Prazo: até [data]."` com botão "Cancelar solicitação".

---

## 📱 MOBILE — Módulo Auth (Expo)

### 21. Autenticação Mobile

- [ ] **21.1** BLOCKED: apps/mobile-cedente não existe no monorepo atual Criar `apps/mobile-cedente/src/screens/auth/LoginScreen.tsx` com formulário e-mail/senha.
- [ ] **21.2** BLOCKED: apps/mobile-cedente não existe no monorepo atual Criar `apps/mobile-cedente/src/screens/auth/CadastroScreen.tsx` com formulário completo PF/PJ.
- [ ] **21.3** BLOCKED: apps/mobile-cedente não existe no monorepo atual Após login bem-sucedido: armazenar `refresh_token` em `expo-secure-store` (`SecureStore.setItemAsync('refresh_token', value)`) — nunca em AsyncStorage (D18 seção 8.1).
- [ ] **21.4** BLOCKED: apps/mobile-cedente não existe no monorepo atual Armazenar `access_token` + `expires_at` em Zustand store (memória volátil — apagado ao fechar o app).
- [ ] **21.5** BLOCKED: apps/mobile-cedente não existe no monorepo atual Implementar interceptor mobile que, ao receber HTTP 401, chama `POST /auth/refresh-mobile` com `{ refresh_token: await SecureStore.getItemAsync('refresh_token') }` no body (D18 seção 8.2).
- [ ] **21.6** BLOCKED: apps/mobile-cedente não existe no monorepo atual Configurar deep link `repasseseguro://auth/callback` para receber token de ativação de conta via e-mail (D18 seção 8.1).

### 22. Biometria (expo-local-authentication) — RN-087

- [ ] **22.1** BLOCKED: apps/mobile-cedente não existe no monorepo atual Implementar tela de configuração em "Perfil > Segurança > Autenticação biométrica" com toggle para habilitar/desabilitar.
- [ ] **22.2** BLOCKED: apps/mobile-cedente não existe no monorepo atual Ao habilitar biometria: chamar `LocalAuthentication.hasHardwareAsync()` e `LocalAuthentication.isEnrolledAsync()` — se não disponível, exibir toast `"Biometria não disponível neste dispositivo."`.
- [ ] **22.3** BLOCKED: apps/mobile-cedente não existe no monorepo atual Persistir preferência em `expo-secure-store`: `SecureStore.setItemAsync('biometric_enabled', 'true')`.
- [ ] **22.4** BLOCKED: apps/mobile-cedente não existe no monorepo atual Ao abrir o app após inatividade >30 minutos (sessão ativa mas inativa): verificar `biometric_enabled`; se habilitado, chamar `LocalAuthentication.authenticateAsync({ promptMessage: 'Confirme sua identidade para acessar o Repasse Seguro', fallbackLabel: 'Usar senha', cancelLabel: 'Cancelar', disableDeviceFallback: false })`.
- [ ] **22.5** BLOCKED: apps/mobile-cedente não existe no monorepo atual Se biometria falhar: redirecionar para tela de login (não para reauth por senha — nova sessão necessária).
- [ ] **22.6** BLOCKED: apps/mobile-cedente não existe no monorepo atual Biometria NUNCA substitui autenticação inicial — é usada APENAS para reautenticação após inatividade (D18 seção 8.3).

---

## 🔀 WIRING

### 23. Integração Backend ↔ Supabase Auth

- [x] **23.1** Confirmar que `SUPABASE_SERVICE_ROLE_KEY` está disponível apenas no backend (`apps/api/.env`) — NUNCA exposta no frontend ou no bundle do mobile (D18 seção 11).
- [x] **23.2** Confirmar que `SUPABASE_ANON_KEY` (chave pública) é a única usada no frontend.
- [x] **23.3** Testar fluxo completo cadastro → ativação → login → refresh → logout em ambiente de desenvolvimento.
- [x] **23.4** Confirmar que Supabase Auth está configurado com `Site URL` = `NEXT_PUBLIC_APP_URL` e redirect URLs autorizadas incluem `/auth/callback` e `/auth/redefinir-senha`.

### 24. Integração Backend ↔ Resend (templates de auth)

- [x] **24.1** Confirmar configuração do template `NOT-CED-01` no Resend para e-mail de boas-vindas + ativação (D21).
- [x] **24.2** Confirmar que e-mail de recuperação de senha usa template Resend customizado (não o e-mail padrão do Supabase).
- [x] **24.3** Testar envio de e-mail em ambiente de desenvolvimento com endereço de teste Resend.

### 25. Frontend ↔ Backend — Endpoints auth

- [x] **25.1** Confirmar que todas as 9 rotas auth do frontend apontam para `https://api.repasseseguro.com.br/api/v1/auth` (ou `localhost:3001/api/v1/auth` em desenvolvimento).
- [x] **25.2** Confirmar que o interceptor de refresh no `api.ts` funciona: ao receber 401, faz refresh e reenvia a requisição original com novo token — sem loop infinito (máx 1 retry por requisição).
- [x] **25.3** Confirmar que cookies `httpOnly` são enviados automaticamente pelo browser em todas as requisições para o backend (CORS `credentials: 'include'`).
- [x] **25.4** Confirmar que `Access-Control-Allow-Credentials: true` e `Access-Control-Allow-Origin: NEXT_PUBLIC_APP_URL` estão configurados no NestJS CORS.

---

## 🧪 TESTES

### 26. Testes unitários (Vitest) — meta ≥80% cobertura no módulo auth

- [x] **26.1** `auth.service.spec.ts` — testar `cadastrar()`: CPF válido/inválido, CPF duplicado, CNPJ validação Receita Federal (mock), e-mail duplicado, senha fraca, rollback em falha Supabase.
- [x] **26.2** `auth.service.spec.ts` — testar `login()`: credenciais corretas, rate limit (1–4 tentativas, 5ª tentativa bloqueio), conta não ativada, conta bloqueada.
- [x] **26.3** `auth.service.spec.ts` — testar `refresh()`: token válido emite novo, token expirado retorna `AUTH_006`.
- [x] **26.4** `auth.service.spec.ts` — testar `recuperarSenha()`: resposta sempre genérica (200) independente de o e-mail existir.
- [x] **26.5** `rate-limit.guard.spec.ts` — testar que 5ª tentativa dispara HTTP 429 com `Retry-After` correto.
- [x] **26.6** `jwt.strategy.spec.ts` — testar que token com `alg: none` é rejeitado.
- [x] **26.7** `inactivity-check.cron.spec.ts` — testar que job publica evento `SESSION_EXPIRING_SOON` quando inatividade > 25min.

### 27. Testes de integração (Vitest + Supertest)

- [x] **27.1** `auth.e2e-spec.ts` — fluxo completo: `POST /auth/cadastro` → verificar `users` criado com `status=PENDING_ACTIVATION` → `GET /auth/ativar-conta?token=xxx` → verificar `status=ACTIVE` → `POST /auth/login` → verificar cookie `refresh_token` set + access_token no body.
- [x] **27.2** `auth.e2e-spec.ts` — fluxo rate limit: 5 tentativas de login com senha errada → 6ª retorna 429 com `AUTH_005`.
- [x] **27.3** `auth.e2e-spec.ts` — fluxo refresh: `POST /auth/refresh` com cookie válido → novo access_token + novo refresh cookie; com cookie expirado → 401 `AUTH_006`.
- [x] **27.4** `auth.e2e-spec.ts` — fluxo recuperação de senha: `POST /auth/recuperar-senha` com e-mail existente e com e-mail inexistente → ambos retornam HTTP 200 com mesma mensagem genérica.

### 28. Teste de Isolamento RLS (T-038 — obrigatório antes de cada release)

- [x] **28.1** Criar `tests/rls/isolamento.spec.ts` implementando exatamente o cenário descrito em D18 seção 7.3:
  - Cedente A não acessa casos do Cedente B via API: `GET /casos/:casoB_id` com token do Cedente A → HTTP 403.
  - Cedente A não acessa casos do Cedente B via Supabase direto: query com `supabaseClientA.from('casos').select('*').eq('id', casoB_id)` → `data` com length 0.
- [x] **28.2** Executar T-038 em CI (deve estar no `ci.yml` como step obrigatório antes de merge em main).

### 29. Testes de segurança

- [x] **29.1** Verificar que access token nunca aparece em localStorage em nenhum cenário (Playwright headless: `localStorage.getItem('access_token')` deve ser `null` após login).
- [x] **29.2** Verificar que `SUPABASE_SERVICE_ROLE_KEY` não aparece em nenhum bundle do frontend (grep no build output).
- [x] **29.3** Verificar que JWT não aceita `alg: none` (testar com token forjado).

---

## ✅ AUTO-VERIFICAÇÃO (12 Checks)

| #   | Check                  | Critério                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Status |
| --- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | **Nomenclatura**       | Todos os itens usam nomes exatos: `users`, `cedentes`, `StatusConta`, `TipoCedente`, `auth_id`, `PENDING_ACTIVATION`, `ACTIVE`; endpoints exatos `/auth/cadastro`, `/auth/login`, `/auth/logout`, `/auth/refresh`, `/auth/refresh-mobile`; códigos de erro exatos `AUTH_001`–`AUTH_008`, `CADASTRO_001`–`CADASTRO_006`                                                                                                                                                                                                                                           | ✅     |
| 2   | **Verificabilidade**   | Cada item de checklist é binariamente verificável (feito/não feito) — nenhum item genérico como "implementar autenticação"                                                                                                                                                                                                                                                                                                                                                                                                                                       | ✅     |
| 3   | **Valores numéricos**  | TTL access token **1 hora** (3600s), refresh token **7 dias** (604800s), ativação **48 horas**, recuperação **1 hora**, bloqueio login **15 minutos** após **5 tentativas**, inatividade **30 minutos**, cooldown reenvio **60 segundos**, LGPD **15 dias corridos**                                                                                                                                                                                                                                                                                             | ✅     |
| 4   | **Contagem de itens**  | 9 endpoints auth documentados: `POST /auth/cadastro`, `GET /auth/ativar-conta`, `POST /auth/reenviar-ativacao`, `POST /auth/login`, `POST /auth/logout`, `POST /auth/refresh`, `POST /auth/refresh-mobile`, `POST /auth/recuperar-senha`, `POST /auth/redefinir-senha`, `POST /auth/reauth`                                                                                                                                                                                                                                                                      | ✅     |
| 5   | **Máquina de estados** | Estados de conta cobertos: Inexistente → `PENDING_ACTIVATION` → `ACTIVE` → `BLOCKED` → desbloqueado; `PENDING_ACTIVATION` → `EXPIRED` (30 dias); sessão: `Anonimo` → `PendentaAtivacao` → `Ativo` → `SessaoAberta` → `TokenRenovado` / `SessaoExpirada` / `Anonimo`; `Ativo` → `Bloqueado` (Admin) → `Ativo`                                                                                                                                                                                                                                                     | ✅     |
| 6   | **RBAC**               | `JwtAuthGuard` aplicado em todas as rotas autenticadas; `RateLimitGuard` em `POST /auth/login`; `SUPABASE_SERVICE_ROLE_KEY` nunca exposta no frontend; reauth obrigatório para LGPD e troca de e-mail                                                                                                                                                                                                                                                                                                                                                            | ✅     |
| 7   | **Anti-scaffold**      | `auth.service.ts` contém lógica real (RLS check, rate limit Redis, cookie set, rollback Prisma transaction) — não apenas chamadas stub                                                                                                                                                                                                                                                                                                                                                                                                                           | ✅     |
| 8   | **Glossário**          | `cedentes` (não `customers`), `casos` (não `cases`), `users` (não `profiles`), `StatusConta` (não `AccountStatus`), `PENDING_ACTIVATION` (não `pending`)                                                                                                                                                                                                                                                                                                                                                                                                         | ✅     |
| 9   | **RLS**                | Políticas RLS criadas para: `users`, `casos`, `documentos_dossie`, `propostas`, `notificacoes`, `ai_sessions`; isolamento T-038 testado                                                                                                                                                                                                                                                                                                                                                                                                                          | ✅     |
| 10  | **Segurança**          | 12 itens do checklist de segurança D18 seção 11 todos cobertos; JWT sem `alg: none`; resposta genérica em `POST /auth/recuperar-senha`                                                                                                                                                                                                                                                                                                                                                                                                                           | ✅     |
| 11  | **Cross-módulo**       | S1 concluída com Supabase, Redis, Resend, RabbitMQ configurados — S2 consome esses serviços sem reconfigurar; RabbitMQ exchange `cedente.notifications` usado para `SESSION_EXPIRING_SOON`                                                                                                                                                                                                                                                                                                                                                                       | ✅     |
| 12  | **Cobertura de REQs**  | REQ-001 (cadastro PF), REQ-002 (ativação 48h), REQ-003 (reenvio + expiração 30d), REQ-004 (recuperação 1h), REQ-005 (bloqueio 5 tentativas/15min), REQ-006 (inatividade 30min/modal 5min), REQ-007 (CPF imutável), REQ-008 (troca e-mail dupla confirmação), REQ-009 (notificações configuráveis), REQ-010 (LGPD 15d), REQ-011 (isolamento RLS), REQ-012 (anonimato Cessionário), REQ-013–021 (dashboard/casos regras), REQ-109–110 (PJ campos), REQ-126 (CNPJ Receita Federal), REQ-160–168 (9 endpoints auth), REQ-181–184 (fluxos auth D18) — 100% atribuídos | ✅     |

---

## 📋 PENDÊNCIAS E DECISÕES

| ID      | Tipo        | Descrição                                                                                                                                                                                                | Ação             |
| ------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| PEND-02 | ⚠️ CONFLITO | D18 seção 3.3 diz TTL ativação = **24 horas**; RN-001 (D01.1) diz TTL ativação = **48 horas**. Adotado **48 horas** por ser o mais específico (RN-001 no doc de regras de negócio). Marcar para revisão. | [REVISÃO MANUAL] |
| PEND-03 | ⚠️ AMBÍGUO  | RN-006 diz expiração por inatividade após **24 horas**; D18 seção 6.4 diz **30 minutos**. Adotado **30 minutos** (mais seguro, valor específico do doc técnico D18).                                     | [REVISÃO MANUAL] |
| DP-001  | ⚠️ BLOCKER  | Conta Escrow (parceiro TBD) — não impacta S2 diretamente. Bloqueio registrado para S6.                                                                                                                   | Pendente S6      |
