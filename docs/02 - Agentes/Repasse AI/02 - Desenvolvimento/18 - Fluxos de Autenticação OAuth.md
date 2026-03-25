# Repasse AI
## 18 — Fluxos de Autenticação OAuth

| Campo | Valor |
|---|---|
| **Destinatário** | Backend Lead / Tech Lead |
| **Escopo** | Sessão de usuário, JWT, segurança, refresh e fluxos de credenciais de plataformas integradas |
| **Módulo** | Repasse AI |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Tipos de autenticação cobertos:** 1 tipo — JWT Bearer Token herdado da plataforma Cessionário (Supabase Auth). O Repasse AI **não tem** login próprio, OAuth 2.0 de plataforma ou social login.
> - **Sessão:** access token com lifetime de 1h; refresh gerenciado pelo frontend Cessionário via Supabase Auth. Repasse AI apenas valida — não emite tokens.
> - **Vault de credenciais de serviço:** API Keys de OpenAI, Redis, RabbitMQ, EvolutionAPI armazenadas em Railway Secrets — nunca em código.
> - **WhatsApp Binding:** fluxo OTP não é OAuth 2.0 — é OTP SMS + confirmação WhatsApp. Documentado na Seção 5.
> - **Total de plataformas com OAuth 2.0:** 0 — Repasse AI não integra com Google, Facebook, LinkedIn ou similar.
> - **State CSRF e PKCE:** N/A — sem fluxos OAuth de usuário final no Repasse AI.
> - **Seções pendentes:** 0. 2 decisões autônomas aplicadas (marcadas inline).

---

## 1. Persona e Contexto

> 🎯 **Objetivo:** Este documento define o comportamento completo de autenticação, autorização e gestão de credenciais do módulo Repasse AI. É a fonte de verdade para implementação de guards, validação de tokens e fluxo de credenciais de serviço.

**Quem usa este documento:**
- Backend developers: implementam guards e validação JWT sem inventar regras.
- Security engineers: auditam sem encontrar lacunas de segurança.
- QA: valida segurança com base no checklist da Seção 7.

**Nota arquitetural importante:** O Repasse AI é um **serviço backend puro (PG-03)** que não possui login próprio, cadastro de usuários ou fluxo OAuth de plataforma. A autenticação é **herdada** da plataforma Cessionário (Supabase Auth). O serviço **valida** tokens — nunca os emite.

---

## 2. Tipos de Auth no Repasse AI

| Tipo | Descrição | Emissor | Quem Valida | Escopo |
|---|---|---|---|---|
| **Tipo 1 — JWT Cessionário** | Access token JWT gerado pela plataforma Cessionário ao fazer login | Supabase Auth (plataforma) | `JwtAuthGuard` + `CessionarioGuard` no Repasse AI | Todos os endpoints de `/chat`, `/calculator`, `/notification`, `/whatsapp` |
| **Tipo 2 — JWT Admin** | Access token JWT com `role: ADMIN` gerado pela plataforma Admin | Supabase Auth (plataforma) | `JwtAuthGuard` + `AdminGuard` no Repasse AI | Endpoints de `/supervision`, `/ai/interactions`, `/ai/metrics` |
| **Tipo 3 — API Keys de Serviço** | Credenciais de serviços externos (OpenAI, Redis, RabbitMQ, EvolutionAPI) | Provedores externos | Injeção via Railway Secrets | Backend internamente |
| **Tipo 4 — OTP WhatsApp** | Fluxo de vinculação de número WhatsApp via OTP SMS | Backend Repasse AI | `WhatsappService` | `/whatsapp/bindings` |

> ⚙️ **OAuth 2.0 de plataforma: NÃO APLICÁVEL** — O Repasse AI não implementa Authorization Code Flow, PKCE ou conexão OAuth com Google, Facebook, LinkedIn, Meta ou qualquer provedor de identidade externo. A autenticação do usuário final é 100% gerenciada pela plataforma Cessionário. O Repasse AI não tem estado de OAuth para gerenciar.

---

## 3. Tipo 1 e 2 — JWT da Plataforma Cessionário

### 3.1 Estrutura do JWT

O JWT emitido pela plataforma Cessionário (Supabase Auth) contém:

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",   // user_id no Supabase
  "cessionario_id": "660e8400-e29b-41d4-a716-446655440001",  // ID do Cessionário
  "role": "CESSIONARIO",                           // ou "ADMIN"
  "email": "usuario@empresa.com.br",
  "exp": 1750000000,
  "iat": 1749996400,
  "iss": "supabase"
}
```

> ⚙️ **Separação entre `sub` e `cessionario_id`:** [DECISÃO AUTÔNOMA] — `sub` é o UUID do usuário no Supabase Auth; `cessionario_id` é o UUID da empresa (tenant) no modelo de dados do Repasse Seguro. Um usuário pode pertencer a apenas uma empresa (1:1 no modelo atual). O RLS usa `cessionario_id`, não `sub`. Alternativa descartada: usar `sub` como `cessionario_id` (quebraria multi-usuário por empresa em Fase 3+).

### 3.2 Fluxo de Validação de Token no Repasse AI

```typescript
// Pseudocódigo: JwtAuthGuard.canActivate()

async canActivate(context: ExecutionContext): Promise<boolean> {
  const request = context.switchToHttp().getRequest();
  const authHeader = request.headers['authorization'];

  // 1. Extrair token
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    throw new UnauthorizedException('[HTTP] Token de autenticação não fornecido.');
  }
  const token = authHeader.split(' ')[1];

  // 2. Verificar modo dev (JWT_DEV_MODE=true apenas em NODE_ENV=development)
  if (process.env.JWT_DEV_MODE === 'true' && process.env.NODE_ENV === 'development') {
    const payload = verifyJwt(token, process.env.JWT_DEV_SECRET);
    request.user = payload;
    return true;
  }

  // 3. Validar com Supabase Auth (produção/staging)
  try {
    const { data: { user }, error } = await supabaseAdmin.auth.getUser(token);
    if (error || !user) {
      throw new UnauthorizedException('[HTTP] Token inválido ou expirado.');
    }
    // Montar payload unificado
    request.user = {
      sub: user.id,
      cessionario_id: user.user_metadata.cessionario_id,
      role: user.user_metadata.role,
      email: user.email,
    };
    return true;
  } catch (e) {
    throw new UnauthorizedException('[HTTP] Token inválido ou expirado.');
  }
}
```

### 3.3 Guard de Autorização por Role

```typescript
// Pseudocódigo: CessionarioGuard.canActivate()
async canActivate(context: ExecutionContext): Promise<boolean> {
  const user = context.switchToHttp().getRequest().user;
  if (!user?.cessionario_id) {
    throw new ForbiddenException('[HTTP] cessionario_id ausente no token.');
  }
  if (user.role !== 'CESSIONARIO' && user.role !== 'ADMIN') {
    throw new ForbiddenException('[HTTP] Papel de usuário insuficiente.');
  }
  return true;
}

// Pseudocódigo: AdminGuard.canActivate()
async canActivate(context: ExecutionContext): Promise<boolean> {
  const user = context.switchToHttp().getRequest().user;
  if (user.role !== 'ADMIN') {
    throw new ForbiddenException('[HTTP] Acesso restrito a administradores.');
  }
  return true;
}
```

### 3.4 Tabela de Segurança de Sessão

| Regra | Valor | Responsável |
|---|---|---|
| Access token lifetime | 1 hora | Supabase Auth (plataforma Cessionário) |
| Refresh token lifetime | 7 dias (padrão Supabase) | Supabase Auth (plataforma Cessionário) |
| Refresh token storage | httpOnly cookie (gerenciado pela plataforma Cessionário) | Frontend Cessionário |
| Refresh proativo | Supabase SDK no frontend Cessionário (auto-refresh em `onAuthStateChange`) | Frontend Cessionário |
| Session timeout | 1h (acesso token); 7d (refresh) | Supabase Auth |
| 2FA/MFA | Gerenciado pela plataforma Cessionário — fora do escopo do Repasse AI | Plataforma |
| Brute-force protection | Gerenciado pelo Supabase Auth (built-in rate limiting no endpoint `/auth/v1/token`) | Supabase Auth |
| Token revogação | Via `supabaseAdmin.auth.admin.signOut(userId)` — chamado ao `DELETE /chat/history` | Repasse AI + Plataforma |

### 3.5 Endpoints de Auth no Repasse AI

> ⚙️ O Repasse AI **não expõe** endpoints de login, registro, 2FA ou refresh. Esses endpoints existem na plataforma Cessionário. Os únicos endpoints relacionados à auth no Repasse AI são:

| Método | Rota | Descrição | Guards |
|---|---|---|---|
| `GET` | `/health` | Health check público — sem auth | Nenhum |
| `GET` | `/repasse-ai/v1/*` | Todos os endpoints protegidos | `JwtAuthGuard` + `CessionarioGuard` |
| `GET/POST/DELETE` | `/repasse-ai/v1/supervision/*` | Endpoints Admin | `JwtAuthGuard` + `AdminGuard` |
| `POST` | `/internal/webhooks/whatsapp` | Webhook EvolutionAPI — sem JWT, protegido por IP allowlist + HMAC | `WebhookSignatureGuard` |

---

## 4. Tipo 3 — Credenciais de Serviço (API Keys)

### 4.1 Visão Geral

Credenciais de serviços externos (OpenAI, Redis, RabbitMQ, EvolutionAPI, Langfuse) são **API Keys**, não OAuth. São tratadas como secrets de infraestrutura, não como tokens de usuário.

| Serviço | Tipo de Credencial | Env Var | Escopo |
|---|---|---|---|
| OpenAI | API Key (Bearer) | `OPENAI_API_KEY` | Backend apenas |
| Upstash Redis | Password na URL | `REDIS_URL` | Backend apenas |
| CloudAMQP | Username/Password na URL | `AMQP_URL` | Backend apenas |
| EvolutionAPI | API Key no header | `EVOLUTION_API_KEY` | Backend apenas |
| Langfuse | Public Key + Secret Key | `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` | Backend apenas |
| Supabase Auth (service role) | Service Role Key | `SUPABASE_SERVICE_ROLE_KEY` | Backend apenas — nunca frontend |

### 4.2 Vault de Credenciais de Serviço por Ambiente

| Ambiente | Vault | Acesso |
|---|---|---|
| **Production** | Railway Secrets (criptografado em repouso) | Tech Lead + DevOps |
| **Staging** | Railway Secrets (projeto staging separado) | Tech Lead + DevOps |
| **Development** | `apps/ai/.env` (não commitado, no `.gitignore`) | Dev local |

### 4.3 Regras de Segurança para API Keys

```
1. Nunca expor API Keys em logs, respostas de API, traces Langfuse ou código-fonte.
2. PII masking antes de qualquer chamada OpenAI: substituir CPF, nome, e-mail, telefone por [PII_TYPE].
3. OPENAI_API_KEY, SUPABASE_SERVICE_ROLE_KEY e EVOLUTION_API_KEY: rotação a cada 90 dias.
4. AMQP_URL e REDIS_URL: rotação a cada 90 dias.
5. Após rotação: validar health check antes de revogar credencial antiga.
```

---

## 5. Tipo 4 — OTP WhatsApp (Não é OAuth)

### 5.1 Descrição do Fluxo

O fluxo de vinculação WhatsApp **não é OAuth 2.0**. É um fluxo OTP SMS + confirmação pelo canal WhatsApp. Não envolve Authorization Code, PKCE ou state CSRF.

### 5.2 State Machine do Binding

```
UNBOUND
  │
  │ POST /whatsapp/bindings (phone_number)
  ▼
OTP_SENT                    ← Redis: OTP code (TTL 5min), attempts counter (TTL 15min)
  │
  │ POST /whatsapp/bindings/verify (otp_code correto)
  ▼
PENDING_WHATSAPP             ← aguarda mensagem de confirmação via WhatsApp (timeout 10min)
  │                          └─ Timeout → volta a UNBOUND (exige novo OTP)
  │ Webhook EvolutionAPI recebe "CONFIRMAR"
  ▼
ACTIVE                       ← binding vinculado; notificações WhatsApp ativas
  │
  │ DELETE /whatsapp/bindings/me
  ▼
UNBOUND (soft delete)
```

### 5.3 Pseudocódigo do Fluxo OTP

```typescript
// ETAPA 1: Solicitar vinculação
// POST /whatsapp/bindings
async initiateBinding(cessionarioId: string, phoneNumber: string): Promise<BindingResult> {
  // 1. Validar formato E.164
  if (!isValidE164(phoneNumber)) {
    throw new BadRequestException('[WA] Número de telefone inválido.');
  }

  // 2. Verificar se número já vinculado a outro cessionário
  const existingBinding = await whatsappRepo.findActiveByPhone(phoneNumber);
  if (existingBinding && existingBinding.cessionario_id !== cessionarioId) {
    throw new ConflictException('[WA] Número já vinculado a outro Cessionário.');
  }

  // 3. Verificar rate limit de OTP (max 3 por 15 min)
  const attemptsKey = REDIS_KEYS.WHATSAPP_OTP_ATTEMPTS(cessionarioId, sha256(phoneNumber));
  const attempts = await redis.get(attemptsKey);
  if (attempts && parseInt(attempts) >= 3) {
    throw new TooManyRequestsException('[WA] Muitas tentativas de OTP. Aguarde 15 minutos.');
  }

  // 4. Gerar OTP de 6 dígitos
  const otpCode = generateSecureOtp(6);
  const otpKey = REDIS_KEYS.WHATSAPP_OTP(cessionarioId, sha256(phoneNumber));
  await redis.setex(otpKey, 300, otpCode);      // TTL: 5 minutos
  await redis.incr(attemptsKey);
  await redis.expire(attemptsKey, 900);          // TTL: 15 minutos

  // 5. Enviar OTP via SMS (EvolutionAPI ou provider SMS)
  await smsService.sendOtp(phoneNumber, otpCode);

  // 6. Criar/atualizar binding em estado OTP_SENT
  return whatsappRepo.upsertBinding(cessionarioId, phoneNumber, 'OTP_SENT');
}

// ETAPA 2: Verificar OTP
// POST /whatsapp/bindings/verify
async verifyOtp(cessionarioId: string, phoneNumber: string, otpCode: string): Promise<BindingResult> {
  const otpKey = REDIS_KEYS.WHATSAPP_OTP(cessionarioId, sha256(phoneNumber));
  const storedOtp = await redis.get(otpKey);

  if (!storedOtp) {
    throw new BadRequestException('[WA] Código OTP expirado. Solicite um novo código.');
  }

  if (storedOtp !== otpCode) {
    throw new BadRequestException('[WA] Código OTP inválido.');
  }

  // OTP válido: limpar da Redis + avançar estado
  await redis.del(otpKey);
  return whatsappRepo.updateBindingStatus(cessionarioId, 'PENDING_WHATSAPP', new Date(Date.now() + 600000));
}

// ETAPA 3: Processamento do webhook
// POST /internal/webhooks/whatsapp
async processWebhook(payload: WhatsappWebhookDto): Promise<void> {
  // 1. Validar assinatura HMAC
  const signature = headers['x-evolution-signature'];
  if (!validateHmacSha256(payload, signature, process.env.EVOLUTION_WEBHOOK_SECRET)) {
    throw new UnauthorizedException('[WA] Assinatura de webhook inválida.');
  }

  // 2. Extrair número e mensagem
  const phoneNumber = extractPhoneFromJid(payload.data.key.remoteJid);
  const message = payload.data.message?.conversation?.toUpperCase();

  // 3. Verificar se é confirmação de binding
  if (message !== 'CONFIRMAR') return; // ignorar outras mensagens

  // 4. Buscar binding em PENDING_WHATSAPP
  const binding = await whatsappRepo.findPendingByPhone(phoneNumber);
  if (!binding || binding.status !== 'PENDING_WHATSAPP') return;

  // 5. Verificar se não expirou (timeout 10 min)
  if (new Date() > binding.pending_expires_at) {
    await whatsappRepo.updateBindingStatus(binding.cessionario_id, 'UNBOUND');
    return;
  }

  // 6. Ativar binding
  await whatsappRepo.updateBindingStatus(binding.cessionario_id, 'ACTIVE');
}
```

### 5.4 Segurança do Fluxo OTP

| Regra | Implementação |
|---|---|
| OTP gerado com entropia segura | `crypto.randomInt(100000, 999999)` (Node.js `crypto` module) |
| OTP em Redis com TTL de 5 min | `SETEX {key} 300 {otp}` |
| Max 3 tentativas por 15 min | Contador Redis com `INCR` + `EXPIRE 900` |
| Número hasheado nas chaves Redis | `sha256(phoneNumber)` — número não em plain text na chave |
| Webhook validado com HMAC-SHA256 | `crypto.timingSafeEqual(receivedSig, computedSig)` |
| Timeout de confirmação WhatsApp: 10 min | `pending_expires_at` no banco; verificado no webhook handler |
| OTP deletado após uso | `redis.del(otpKey)` após verificação bem-sucedida |

---

## 6. Middleware de Segurança Global

### 6.1 Ordem de Execução de Guards

```
Request
  │
  ▼
JwtAuthGuard          → valida Bearer Token; extrai user payload
  │
  ▼
CessionarioGuard      → verifica cessionario_id; OU
AdminGuard            → verifica role ADMIN (endpoints /supervision)
  │
  ▼
RateLimitGuard        → verifica limite de requisições por tipo de rota
  │
  ▼
ValidationPipe        → valida DTOs com class-validator
  │
  ▼
Controller
  │
  ▼
TransformInterceptor  → normaliza resposta { data, meta }
  │
  ▼
LoggingInterceptor    → log estruturado Pino
```

### 6.2 Isolamento de Tenant via RLS

Após autenticação bem-sucedida, todo acesso ao banco passa por `withTenant()`:

```typescript
// PrismaService.withTenant() aplica SET LOCAL para RLS
async withTenant<T>(cessionarioId: string, fn: (tx: PrismaClient) => Promise<T>): Promise<T> {
  return this.$transaction(async (tx) => {
    await tx.$executeRaw`SET LOCAL app.cessionario_id = ${cessionarioId}`;
    return fn(tx);
  });
}
```

Todas as 22 policies RLS filtram por `current_setting('app.cessionario_id', true)::uuid`. Sem `withTenant()`, queries retornam 0 linhas (policies impedem cross-tenant silenciosamente).

---

## 7. Checklist de Segurança

| # | Verificação | Implementação | Escopo |
|---|---|---|---|
| S-01 | Tokens JWT nunca em plain text em logs | `LoggingInterceptor` omite header `Authorization` nos logs | Global |
| S-02 | `cessionario_id` sempre verificado antes de acessar dados | `CessionarioGuard` obrigatório em todos os endpoints de Cessionário | Global |
| S-03 | RLS ativo em todas as queries de Cessionário | `withTenant()` obrigatório no `PrismaService` | Todos os módulos |
| S-04 | PII masking antes de chamadas OpenAI e Langfuse | `LlmService.maskPii()` aplicado antes de construir prompt | `AiModule` |
| S-05 | API Keys nunca em código-fonte ou logs | Railway Secrets; env vars apenas; log sem valores sensíveis | Global |
| S-06 | Webhook EvolutionAPI com HMAC-SHA256 | `WebhookSignatureGuard` usando `crypto.timingSafeEqual` | `WhatsappModule` |
| S-07 | IP allowlist para endpoint de webhook | Configuração Railway + middleware de IP validation | `/internal/webhooks` |
| S-08 | OTP gerado com `crypto.randomInt` (entropia segura) | `crypto.randomInt(100000, 999999)` no `WhatsappService` | `WhatsappModule` |
| S-09 | Max 3 tentativas de OTP por 15 min | Contador Redis com TTL 900s | `WhatsappModule` |
| S-10 | Rate limiting por usuário em todos os endpoints de IA | `@nestjs/throttler` por `cessionario_id` (30 msg/h webchat) | `ChatModule`, `AiModule` |
| S-11 | `SUPABASE_SERVICE_ROLE_KEY` exclusiva ao backend | Env var Railway Secrets; nunca no frontend; nunca no cliente | Global |
| S-12 | Soft delete para dados PII com 90 dias de retenção | `deleted_at` + job de anonimização LGPD via RabbitMQ | Todos os módulos com PII |
| S-13 | JWT_DEV_MODE desabilitado em staging e produção | Verificação `NODE_ENV !== 'development'` no `JwtAuthGuard` | Global |
| S-14 | CORS configurado com origem explícita | `CorsOptions` com `origin: process.env.ALLOWED_ORIGINS` | `AppModule` |

---

## 8. Refresh de Token — Responsabilidade

| Responsabilidade | Dono | Mecanismo |
|---|---|---|
| Emissão do access token | Supabase Auth (plataforma Cessionário) | `POST /auth/v1/token` |
| Refresh do access token | Frontend Cessionário | Supabase SDK `onAuthStateChange` + `setSession()` |
| Expiração de refresh token | Supabase Auth (7 dias padrão) | Configuração do Supabase |
| Revogação de token (logout) | Frontend Cessionário + Plataforma | `supabase.auth.signOut()` |
| Detecção de token expirado no Repasse AI | `JwtAuthGuard` | `error.name === 'TokenExpiredError'` → 401 |

> ⚙️ **O Repasse AI não gerencia refresh de access token.** Se o frontend Cessionário não renovar o token antes da expiração de 1h e enviar request com token expirado, o Repasse AI retorna `401 UNAUTHORIZED` e o frontend é responsável por renovar e retentar.

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 22/03/2026 | v1.0 | Versão inicial. Documentação completa de autenticação JWT para serviço backend puro. 4 tipos de auth mapeados. Fluxo OTP WhatsApp com state machine. Checklist de segurança com 14 itens. Isolamento tenant via RLS documentado. |

---

## 10. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Separação entre `sub` e `cessionario_id` no JWT | [DECISÃO AUTÔNOMA] | 3.1 Estrutura do JWT | `sub` = UUID de usuário Supabase; `cessionario_id` = UUID do tenant. Separação necessária para suportar multi-usuário por empresa em fases futuras. Alternativa descartada: usar `sub` como `cessionario_id` (quebraria escalonamento futuro). | P1 | Backend Lead | Decidido |
| `JWT_DEV_MODE` para desenvolvimento local | [DECISÃO AUTÔNOMA] | 3.2 Fluxo de Validação | Em dev local sem Supabase configurado, `JWT_DEV_MODE=true` permite assinar JWTs com `JWT_DEV_SECRET`. Desabilitado automaticamente em `NODE_ENV !== 'development'`. Alternativa descartada: exigir Supabase em dev (bloqueia onboarding). | P1 | Backend Lead | Decidido |

---

*Próximo documento do pipeline: D19 — Criação de Agentes de IA.*
