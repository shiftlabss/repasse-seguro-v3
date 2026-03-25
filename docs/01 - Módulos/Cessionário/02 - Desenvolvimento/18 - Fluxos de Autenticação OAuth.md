# 18 - Fluxos de Autenticação OAuth

## Módulo Cessionário · Plataforma Repasse Seguro

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Backend Lead / Tech Lead | Sessão de usuário, OAuth 2.0 por plataforma, tokens, segurança e refresh | v1.0 | Claude Code Desktop | 2026-03-22T02:00:00-03:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Dois tipos de autenticação:** Tipo 1 — Sessão de Usuário (e-mail/senha + Google OAuth via Supabase Auth); Tipo 2 — Sem OAuth de plataformas externas além do Google (sem integrações de terceiros que exijam OAuth 2.0 separado — idwall, ZapSign e Celcoin usam API Key).
> - **Sessão:** Access token JWT 15 minutos; refresh token httpOnly cookie 30 dias; inatividade 30 minutos.
> - **Re-autenticação:** janela de confiança de 15 minutos para ações críticas (RN-005).
> - **Vault:** tokens nunca em plain text — `SUPABASE_SERVICE_ROLE_KEY` e JWT secrets apenas em env vars backend; refresh token apenas em httpOnly cookie.
> - **Segurança:** brute-force protection via Redis (5 tentativas → lockout 15min), state CSRF em Google OAuth, PKCE obrigatório via Supabase Auth.
> - **Refresh proativo:** job Redis a cada 5 minutos verifica tokens próximos de expirar.
> - **Sem seções pendentes** — todos os fluxos cobertos com contexto disponível.

---

## 1. Persona e Contexto

Este documento é a referência autoritativa para implementação de autenticação no módulo Cessionário da plataforma Repasse Seguro. Destina-se ao Backend Lead e Tech Lead responsáveis pela implementação de sessão, OAuth e segurança.

**Impacto no pipeline:**
- **28 - Checklist de Qualidade:** valida se a implementação segue o checklist de segurança desta seção 7.
- **20 - Error Handling:** referencia os códigos de erro de autenticação (AUTH-001 a AUTH-006).
- Se um fluxo ou regra de segurança não estiver documentado aqui, ele **não deve ser inventado** no código.

**Dependências:**
- RN-001 a RN-005 (Regras de autenticação — D01.1)
- `02 - Stacks` (Supabase Auth, Redis, JWT, httpOnly cookies)
- `16 - Documentação de API` (endpoints `/auth/*`)
- `17 - Integrações Externas` (Google OAuth via Supabase)

---

## 2. Dois Tipos de Autenticação

O módulo Cessionário opera com dois tipos distintos de autenticação:

| Tipo | Nome | Descrição | Provedor |
|---|---|---|---|
| **Tipo 1** | Sessão de Usuário | Login do Cessionário na plataforma — acessa dados, proposta, Escrow | Supabase Auth (e-mail/senha + Google OAuth) |
| **Tipo 2** | API Keys de Serviços Externos | Autenticação de integrações backend-to-backend | API Keys (ZapSign, idwall, Celcoin, OpenAI, Resend) — ver D17 |

> 🎯 **Nota arquitetural:** O módulo Cessionário **não possui OAuth 2.0 de conexão com plataformas externas** (como Social Media ou CRMs). As integrações externas (ZapSign, idwall, Celcoin) usam API Keys fixas gerenciadas pelo backend — documentadas no D17. O único fluxo OAuth 2.0 do ponto de vista do usuário é o **Login com Google**, gerenciado integralmente pelo Supabase Auth.

---

## 3. Tipo 1 — Sessão de Usuário

### 3.1 Fluxo de Login (E-mail + Senha)

```pseudocode
POST /api/v1/auth/login
  body: { email, password }

1. BRUTE-FORCE CHECK
   key = "rs:auth:attempts:{ip}:{email}"
   attempts = Redis.GET(key)
   IF attempts >= 5:
     remaining_ttl = Redis.TTL(key)
     RETURN 429 { code: "AUTH-003", message: "Muitas tentativas. Tente novamente em {remaining_ttl}s." }

2. AUTHENTICATE via Supabase Auth
   result = await supabase.auth.signInWithPassword({ email, password })

   IF result.error:
     Redis.INCR(key)
     Redis.EXPIRE(key, 900)  // 15 minutos de lockout window
     RETURN 401 { code: "AUTH-001", message: "E-mail ou senha incorretos." }

   // Sucesso — limpar tentativas
   Redis.DEL("rs:auth:attempts:{ip}:{email}")

3. CHECK ACCOUNT STATUS
   cessionario = await db.cessionarios.findUnique(where: { userId: result.user.id })

   IF cessionario.status == "BLOCKED":
     RETURN 403 { code: "AUTH-004", message: "Conta temporariamente suspensa." }
   IF cessionario.status == "CLOSED":
     RETURN 403 { code: "AUTH-005", message: "Esta conta foi encerrada." }

4. CREATE SESSION
   session_key = "rs:session:{userId}"
   Redis.SETEX(session_key, 1800, JSON.stringify({ userId, email, role: "CESSIONARIO", loginAt: now() }))
   // TTL: 1800s = 30 minutos (inatividade timeout — RN-004)

5. GENERATE TOKENS
   access_token = JWT.sign(
     { sub: userId, email, role: "CESSIONARIO", type: "access" },
     JWT_SECRET,
     { expiresIn: "15m" }
   )
   refresh_token = JWT.sign(
     { sub: userId, type: "refresh", jti: uuid() },
     JWT_SECRET,
     { expiresIn: "30d" }
   )

   // Armazenar refresh token hash no Redis para invalidação
   refresh_key = "rs:auth:refresh:{userId}:{jti}"
   Redis.SETEX(refresh_key, 2592000, "valid")  // 30 dias

6. AUDIT LOG
   await db.audit_logs.create({
     actor_id: userId, actor_type: "CESSIONARIO",
     action: "LOGIN", metadata: { ip, userAgent }
   })

7. RESPONSE
   Set-Cookie: refresh_token={refresh_token}; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=2592000
   RETURN 200 { access_token, user: { id, email, name, role, cessionarioStatus } }
```

---

### 3.2 Fluxo de Login com Google OAuth

```pseudocode
// ETAPA 1 — Iniciar fluxo (Frontend)
GET /api/v1/auth/google/authorize

1. GENERATE STATE (CSRF protection)
   state = crypto.randomUUID()
   code_verifier = crypto.randomBytes(32).toString('base64url')
   code_challenge = SHA256(code_verifier).toString('base64url')

   // Armazenar no Redis com TTL de 10 minutos
   Redis.SETEX("rs:oauth:state:{state}", 600, JSON.stringify({ code_verifier }))

2. REDIRECT to Supabase Auth Google OAuth
   // Supabase gerencia PKCE e o redirect para Google automaticamente
   url = supabase.auth.signInWithOAuth({
     provider: 'google',
     options: {
       redirectTo: '{APP_URL}/api/v1/auth/google/callback',
       queryParams: { state, access_type: 'offline', prompt: 'consent' }
     }
   })
   REDIRECT 302 to url

// ETAPA 2 — Callback (Google → Supabase → Plataforma)
GET /api/v1/auth/google/callback?code={code}&state={state}

1. VALIDATE STATE (CSRF)
   stored = Redis.GET("rs:oauth:state:{state}")
   IF not stored:
     RETURN 400 { code: "AUTH-006", message: "State inválido ou expirado." }
   Redis.DEL("rs:oauth:state:{state}")

2. EXCHANGE CODE via Supabase Auth
   session = await supabase.auth.exchangeCodeForSession(code)
   IF session.error:
     RETURN 400 { code: "AUTH-007", message: "Não foi possível autenticar com o Google." }

3. UPSERT USER
   user = session.data.user
   cessionario = await db.cessionarios.upsert({
     where: { userId: user.id },
     create: { userId: user.id, email: user.email, name: user.user_metadata.full_name, status: "REGISTERED" },
     update: {}  // não sobrescreve dados existentes
   })

4. CHECK ACCOUNT STATUS
   (mesmo que fluxo e-mail/senha: BLOCKED → 403, CLOSED → 403)

5. CREATE SESSION + GENERATE TOKENS
   (idêntico ao fluxo e-mail/senha — steps 4 a 7)

6. REDIRECT to Frontend
   REDIRECT 302 to '{WEB_URL}/dashboard?login=success'
```

---

### 3.3 Fluxo de Refresh Token

```pseudocode
POST /api/v1/auth/refresh
  Cookie: refresh_token={token}

1. EXTRACT TOKEN from httpOnly cookie
   refresh_token = request.cookies['refresh_token']
   IF not refresh_token:
     RETURN 401 { code: "AUTH-008", message: "Refresh token ausente." }

2. VERIFY JWT
   payload = JWT.verify(refresh_token, JWT_SECRET)
   IF error OR payload.type != "refresh":
     RETURN 401 { code: "AUTH-008", message: "Token inválido." }

3. CHECK BLACKLIST (Redis)
   refresh_key = "rs:auth:refresh:{payload.sub}:{payload.jti}"
   IF not Redis.EXISTS(refresh_key):
     RETURN 401 { code: "AUTH-009", message: "Sessão expirada. Faça login novamente." }

4. ROTATE REFRESH TOKEN (Token Rotation)
   // Invalidar token atual
   Redis.DEL(refresh_key)

   // Gerar novo access + refresh
   new_access_token = JWT.sign({ sub, email, role, type: "access" }, JWT_SECRET, { expiresIn: "15m" })
   new_jti = uuid()
   new_refresh_token = JWT.sign({ sub, type: "refresh", jti: new_jti }, JWT_SECRET, { expiresIn: "30d" })
   Redis.SETEX("rs:auth:refresh:{sub}:{new_jti}", 2592000, "valid")

5. UPDATE SESSION TTL
   Redis.EXPIRE("rs:session:{sub}", 1800)  // resetar inatividade

6. RESPONSE
   Set-Cookie: refresh_token={new_refresh_token}; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=2592000
   RETURN 200 { access_token: new_access_token }
```

---

### 3.4 Fluxo de Re-autenticação (RN-005)

Para ações críticas (depósito Escrow, assinatura de documento, alteração de dados bancários):

```pseudocode
POST /api/v1/auth/re-authenticate
  headers: { Authorization: "Bearer {access_token}" }
  body: { password }  // ou { biometric_token } em mobile

1. VERIFY ACCESS TOKEN (middleware padrão)

2. CHECK LAST AUTH TIMESTAMP
   session = Redis.GET("rs:session:{userId}")
   last_auth = session.loginAt  // ou último re-auth bem-sucedido

   IF now() - last_auth <= 15 * 60:  // 15 minutos
     RETURN 200 { message: "Re-autenticação não necessária.", trusted: true }

3. VERIFY PASSWORD via Supabase Auth
   result = await supabase.auth.signInWithPassword({ email: session.email, password })
   IF error:
     RETURN 401 { code: "AUTH-010", message: "Senha incorreta. Tente novamente." }

4. UPDATE SESSION TIMESTAMP
   session.loginAt = now()
   Redis.SETEX("rs:session:{userId}", 1800, JSON.stringify(session))

5. RESPONSE
   RETURN 200 { message: "Re-autenticação bem-sucedida.", trusted: true }
```

---

### 3.5 Fluxo de Logout

```pseudocode
POST /api/v1/auth/logout
  headers: { Authorization: "Bearer {access_token}" }
  Cookie: refresh_token={token}

1. INVALIDATE REFRESH TOKEN
   payload = JWT.verify(refresh_token, JWT_SECRET)
   Redis.DEL("rs:auth:refresh:{payload.sub}:{payload.jti}")

2. DELETE SESSION
   Redis.DEL("rs:session:{userId}")

3. SIGN OUT from Supabase Auth
   await supabase.auth.signOut()

4. AUDIT LOG
   await db.audit_logs.create({ action: "LOGOUT", actorId: userId })

5. CLEAR COOKIE + RESPONSE
   Set-Cookie: refresh_token=; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=0
   RETURN 200 { message: "Sessão encerrada." }
```

---

### 3.6 Tabela de Segurança da Sessão

| Regra | Valor |
|---|---|
| Session timeout (inatividade) | 30 minutos (TTL Redis `rs:session:{userId}`) — RN-004 |
| Aviso de inatividade | Modal exibido aos 25 minutos; 5 minutos para responder |
| Brute-force threshold | 5 tentativas falhas no mesmo IP+email |
| Brute-force lockout duration | 15 minutos (`Redis.EXPIRE "rs:auth:attempts:{ip}:{email}" 900`) |
| Janela de confiança re-auth | 15 minutos para ações críticas (RN-005) |
| 2FA TOTP obrigatório | Apenas para role Admin (Cessionário não tem 2FA obrigatório — não tem role admin) |
| Access token lifetime | 15 minutos |
| Refresh token lifetime | 30 dias |
| Refresh token storage | `httpOnly; Secure; SameSite=Strict` cookie — jamais localStorage |
| Token rotation | Sim — a cada refresh, novo refresh token é emitido e o antigo é invalidado |
| State CSRF (Google OAuth) | UUID aleatório com TTL 10 minutos no Redis |
| PKCE | Gerenciado pelo Supabase Auth (obrigatório) |

---

### 3.7 Endpoints de Sessão

| Método | Rota | Descrição | Auth requerida |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Cadastrar novo Cessionário (e-mail + senha) | Não |
| `POST` | `/api/v1/auth/login` | Login com e-mail e senha | Não |
| `GET` | `/api/v1/auth/google/authorize` | Iniciar fluxo OAuth Google | Não |
| `GET` | `/api/v1/auth/google/callback` | Callback após OAuth Google | Não |
| `POST` | `/api/v1/auth/refresh` | Renovar access token via refresh token (cookie) | Não (usa cookie) |
| `POST` | `/api/v1/auth/logout` | Encerrar sessão e invalidar tokens | Sim (Bearer) |
| `GET` | `/api/v1/auth/me` | Retornar dados do usuário autenticado | Sim (Bearer) |
| `POST` | `/api/v1/auth/re-authenticate` | Re-autenticar para ações críticas | Sim (Bearer) |
| `POST` | `/api/v1/auth/forgot-password` | Solicitar recuperação de senha | Não |
| `POST` | `/api/v1/auth/reset-password` | Confirmar nova senha (via token de e-mail) | Não |

---

## 4. Tipo 2 — OAuth 2.0 de Plataformas

> 🎯 **Escopo desta seção:** No contexto do módulo Cessionário, o único OAuth 2.0 de "plataforma" que envolve redirecionamento de usuário é o **Google OAuth para login** — coberto na seção 3.2. As demais integrações (ZapSign, idwall, Celcoin, OpenAI, Resend) são **API Keys backend-to-backend** e não envolvem fluxo OAuth do usuário.

### 4.1 Fluxo Genérico Authorization Code + PKCE

O fluxo abaixo é o padrão para qualquer OAuth 2.0 que venha a ser adicionado no futuro:

```pseudocode
// ETAPA 1 — Iniciar fluxo
1. Gerar code_verifier (32 bytes aleatórios, base64url)
2. Gerar code_challenge = SHA256(code_verifier).toString('base64url')
3. Gerar state = crypto.randomUUID()
4. Redis.SETEX("rs:oauth:state:{state}", 600, JSON.stringify({ code_verifier, userId }))
5. Construir authorization URL:
   {PROVIDER_AUTH_URL}?
     client_id={CLIENT_ID}
     &redirect_uri={CALLBACK_URL}
     &response_type=code
     &scope={SCOPES}
     &state={state}
     &code_challenge={code_challenge}
     &code_challenge_method=S256
6. Redirecionar usuário

// ETAPA 2 — Provider autentica usuário e redireciona para callback

// ETAPA 3 — Callback
GET {CALLBACK_URL}?code={code}&state={state}

1. Validar state no Redis → extrair code_verifier → DELETE key
2. Trocar code por tokens:
   POST {PROVIDER_TOKEN_URL}
     { grant_type: "authorization_code", code, redirect_uri, client_id, client_secret, code_verifier }
3. Armazenar tokens no vault (nunca plain text):
   vault_ref = await VaultService.store({
     userId, provider, access_token, refresh_token, expires_at
   })
   // VaultService: Supabase Vault (Postgres `vault.secrets`) em prod
4. Criar ou atualizar Integration record no banco:
   { userId, provider, status: "CONNECTED", vaultRef: vault_ref }
```

### 4.2 Callback URL Canônica

```
https://api.repasseseguro.com.br/api/v1/auth/{provider}/callback
```

Exemplos:
- Google: `https://api.repasseseguro.com.br/api/v1/auth/google/callback`

> ⚙️ **Dev/Staging:** `http://localhost:3001/api/v1/auth/{provider}/callback` e `https://staging-api.repasseseguro.com.br/api/v1/auth/{provider}/callback`. Ambas URLs devem estar registradas no console do provedor.

### 4.3 Vault de Tokens

| Ambiente | Vault | Implementação |
|---|---|---|
| Produção / Staging | Supabase Vault (`vault.secrets`) | `SELECT vault.create_secret(secret, 'ACCESS_TOKEN_{userId}_{provider}')` |
| Dev local | Tabela `token_vault` no PostgreSQL local com campo encrypted (pgcrypto) | Simula vault com `pgp_sym_encrypt` |

```pseudocode
// Armazenar token (backend service role apenas)
VaultService.store(userId, provider, token_data):
  ref_id = uuid()
  await db.$executeRaw`SELECT vault.create_secret(${JSON.stringify(token_data)}, ${ref_id})`
  RETURN ref_id

// Recuperar token (backend service role apenas — nunca frontend)
VaultService.retrieve(ref_id):
  result = await db.$queryRaw`SELECT vault.decrypted_secret(${ref_id})`
  RETURN JSON.parse(result.decrypted_secret)
```

> 🔴 **Regra crítica:** `VaultService.retrieve()` é chamado **somente por serviços backend** usando `SUPABASE_SERVICE_ROLE_KEY`. Nunca exposto em endpoints públicos. Nunca logado.

---

## 5. OAuth por Plataforma

### 5.1 Google — Login do Usuário

| Campo | Valor |
|---|---|
| **Finalidade** | Login / cadastro do Cessionário na plataforma (RN-003) |
| **Fluxo** | Authorization Code + PKCE (gerenciado pelo Supabase Auth) |
| **Access Token Lifetime** | 1 hora (Google emite; Supabase gerencia automaticamente) |
| **Refresh Token** | Lifetime indefinido; rotação automática via Supabase Auth |
| **PKCE** | Sim — obrigatório (Supabase Auth implementa) |
| **State CSRF** | Sim — UUID v4 com TTL 10min Redis |
| **Credenciais necessárias** | `GOOGLE_CLIENT_ID` (env Supabase), `GOOGLE_CLIENT_SECRET` (env Supabase) — configurados no Supabase Auth Dashboard |
| **Scopes** | `email`, `profile`, `openid` |
| **Dados recebidos** | `email`, `name`, `picture`, `sub` (Google ID) |
| **Gerenciado por** | Supabase Auth — a plataforma não manipula tokens Google diretamente |

**Particularidades:**
- O token Google é gerenciado pelo Supabase Auth internamente. A plataforma não armazena o access_token nem o refresh_token do Google — apenas a sessão Supabase resultante.
- Se o usuário revoga o acesso no Google Account Settings, a sessão Supabase expira na próxima tentativa de refresh.
- `prompt: 'consent'` + `access_type: 'offline'` garantem que o Supabase obtenha refresh_token para manter a sessão.

---

### 5.2 ZapSign — Autenticação da Integração (API Key, não OAuth)

| Campo | Valor |
|---|---|
| **Tipo de autenticação** | API Key (Bearer Token) |
| **Fluxo** | Sem OAuth — API Key fixa gerenciada por env var `ZAPSIGN_API_TOKEN` |
| **Rotação** | Trimestral (manual — ver D22 Secrets Management) |
| **Vault** | Env var Railway / EAS Secrets |

---

### 5.3 idwall — Autenticação da Integração (API Key, não OAuth)

| Campo | Valor |
|---|---|
| **Tipo de autenticação** | API Key |
| **Fluxo** | Sem OAuth — API Key fixa gerenciada por env var `IDWALL_API_KEY` |
| **Rotação** | Trimestral |
| **Vault** | Env var Railway |

---

### 5.4 Celcoin — Autenticação da Integração (OAuth 2.0 Client Credentials)

| Campo | Valor |
|---|---|
| **Tipo de autenticação** | OAuth 2.0 Client Credentials (machine-to-machine — sem interação do usuário) |
| **Fluxo** | `POST /connect/token` com `grant_type=client_credentials` |
| **Access Token Lifetime** | [DECISÃO AUTÔNOMA] Assumido 1 hora (padrão Celcoin). Critério: padrão de OAuth2 machine-to-machine para fintechs BCB. |
| **Refresh Token** | Não se aplica — Client Credentials obtém novo token via re-autenticação quando expirar |
| **PKCE** | Não aplicável (machine-to-machine, sem browser) |
| **Credenciais** | `CELCOIN_CLIENT_ID`, `CELCOIN_CLIENT_SECRET` |
| **Cache do token** | Redis `rs:celcoin:access-token` com TTL = expires_in - 60s (buffer) |

```pseudocode
// CelcoinAuthService — chamado antes de qualquer requisição Celcoin
async getAccessToken():
  cached = await Redis.GET("rs:celcoin:access-token")
  IF cached: RETURN cached

  response = await fetch(`${CELCOIN_API_URL}/connect/token`, {
    method: "POST",
    body: new URLSearchParams({
      grant_type: "client_credentials",
      client_id: CELCOIN_CLIENT_ID,
      client_secret: CELCOIN_CLIENT_SECRET
    })
  })

  token_data = await response.json()
  Redis.SETEX("rs:celcoin:access-token", token_data.expires_in - 60, token_data.access_token)
  RETURN token_data.access_token
```

---

## 6. Refresh Proativo e Reconexão

### 6.1 Job de Refresh de Sessão

O access token expira em 15 minutos. O frontend é responsável por renovar proativamente antes da expiração:

```pseudocode
// Frontend: useAuthRefresh hook (executado a cada 60s)
async function proactiveRefresh():
  token_payload = JWT.decode(current_access_token)  // sem verificar assinatura
  expires_in = token_payload.exp - now()

  IF expires_in < 180:  // < 3 minutos restantes
    new_token = await POST('/api/v1/auth/refresh')
    updateAccessToken(new_token.access_token)
```

### 6.2 Job de Cleanup de Sessões Inativas (Backend)

```pseudocode
// Cron job: a cada 5 minutos (RabbitMQ scheduled ou NestJS @Cron)
async function cleanInactiveSessions():
  // Sessões Redis com TTL expirado são removidas automaticamente
  // Job verifica e limpa refresh tokens de sessões encerradas

  expired_sessions = await Redis.SCAN("rs:session:*")
  FOR session_key IN expired_sessions:
    IF Redis.TTL(session_key) <= 0:
      userId = extractUserId(session_key)
      // Invalidar todos os refresh tokens do usuário
      refresh_keys = await Redis.SCAN("rs:auth:refresh:{userId}:*")
      FOR key IN refresh_keys:
        Redis.DEL(key)
      AUDIT_LOG("SESSION_EXPIRED_INACTIVITY", userId)
```

### 6.3 Estados de Sessão

```
                    [LOGIN BEM-SUCEDIDO]
                           │
                           ▼
                      ┌─────────┐
                      │  ATIVA  │ ◄── Token renovado via /refresh
                      └────┬────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
   [30min inatividade] [Logout]   [Token inválido/expirado]
           │               │               │
           ▼               ▼               ▼
      ┌────────┐      ┌────────┐     ┌──────────┐
      │EXPIRADA│      │EXPIRADA│     │  INVÁLIDA │
      │(timeout)│     │(manual)│     │(revogada) │
      └────────┘      └────────┘     └──────────┘
           │               │               │
           └───────────────┴───────────────┘
                           │
                   [Redirect → /login]
```

### 6.4 Circuit Breaker para Google OAuth

```pseudocode
// Se Google OAuth falha consecutivamente
google_fail_key = "rs:auth:google:failures"
failures = Redis.INCR(google_fail_key)
Redis.EXPIRE(google_fail_key, 300)  // janela de 5 minutos

IF failures >= 3:
  // Desabilitar botão Google OAuth por 5 minutos
  Redis.SETEX("rs:auth:google:circuit_open", 300, "1")
  // Frontend consulta GET /api/v1/auth/status e esconde botão Google se circuit aberto
  ALERT Sentry: "Google OAuth circuit breaker ativado"
```

---

## 7. Checklist de Segurança

| # | Verificação | Implementação |
|---|---|---|
| S-01 | Tokens nunca em plain text no storage | Access token em memória (Zustand store); refresh token em httpOnly cookie; vault refs para integrações externas |
| S-02 | State CSRF obrigatório em OAuth | `crypto.randomUUID()` armazenado em Redis com TTL 10min; validado no callback antes de processar |
| S-03 | PKCE obrigatório onde suportado | Google OAuth via Supabase Auth (implementado); Celcoin Client Credentials (não aplicável — machine-to-machine) |
| S-04 | Refresh token httpOnly cookie | `Set-Cookie: HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth` — jamais localStorage ou sessionStorage |
| S-05 | Brute-force protection | 5 tentativas por IP+email → lockout 15min via Redis `rs:auth:attempts:{ip}:{email}` |
| S-06 | 2FA obrigatório para admins | N/A para Cessionário (role `CESSIONARIO`); obrigatório para role `ADMIN` — implementado no módulo Admin |
| S-07 | Session timeout por inatividade | 30 minutos via Redis TTL `rs:session:{userId}`; aviso aos 25min (RN-004) |
| S-08 | Audit log de autenticações | `audit_logs` registra: LOGIN, LOGOUT, RE_AUTH, OAUTH_CONNECT, SESSION_EXPIRED_INACTIVITY, TOKEN_REFRESH |
| S-09 | Token rotation no refresh | Cada chamada a `/auth/refresh` emite novo refresh token e invalida o anterior (`rs:auth:refresh:{userId}:{jti}`) |
| S-10 | Re-autenticação para ações críticas | Janela de confiança 15min (RN-005); modal overlay sem perder contexto; ações: depósito Escrow, assinatura, dados bancários |
| S-11 | `SUPABASE_SERVICE_ROLE_KEY` nunca exposto | Env var backend only; nunca em logs, respostas de API ou frontend |
| S-12 | CORS restritivo | `CORS_ORIGIN` com lista explícita por ambiente; sem wildcard `*` em produção |
| S-13 | CSRF protection via SameSite cookie | `SameSite=Strict` nos cookies de refresh; `@nestjs/csrf` em endpoints que mutam estado |
| S-14 | PII não logado | `JWT_SECRET`, `SUPABASE_SERVICE_ROLE_KEY`, CPF, senhas nunca em logs Pino; `scrubFields` configurado no Sentry |
| S-15 | Revogação total de sessão disponível | Admin pode revogar todas as sessões de um Cessionário via `Redis.DEL("rs:session:{userId}")` + scan e DEL de todos os refresh tokens |

---

## 8. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-22 | v1.0 | Criação do documento — Tipo 1 (e-mail + Google OAuth), Tipo 2 (API Keys + Celcoin Client Credentials), fluxos completos com pseudocódigo, checklist com 15 verificações. |

---

## 9. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Celcoin token lifetime | `[DECISÃO AUTÔNOMA]` | 5.4 — Celcoin | Assumido 1h (padrão OAuth2 Client Credentials para fintechs). Descartado: solicitar ao Celcoin (demora). Critério: padrão de mercado BCB. | Baixo — se token expirar antes, requisição falha com 401 e retry renova. | Backend Lead | Implementado (com fallback de retry) |
| 2FA Cessionário opcional | `[DECISÃO AUTÔNOMA]` | 3.6 — Tabela Segurança | 2FA não obrigatório para Cessionário no MVP. Descartado: 2FA obrigatório (aumenta fricção no onboarding — RN-001 prioriza conversão). Critério: re-autenticação para ações críticas (RN-005) já mitiga o risco principal. | Médio — pode ser adicionado como feature opcional em v2. | Tech Lead + Produto | Implementado (decisão MVP) |
| Revogação de Google OAuth | `[DECISÃO AUTÔNOMA]` | 3.2 — Google OAuth | Quando usuário revoga acesso no Google, Supabase Auth detecta na próxima renovação e invalida a sessão. Comportamento documentado. Descartado: webhook de revogação Google (complexidade alta, pouco impacto). | Baixo | Backend Lead | Documentado |
