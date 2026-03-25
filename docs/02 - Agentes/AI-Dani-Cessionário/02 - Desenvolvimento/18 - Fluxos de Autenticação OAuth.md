# 18 - Fluxos de Autenticação OAuth

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Backend Lead / Tech Lead | Sessão de usuário, herança de sessão da plataforma, OTP WhatsApp, tokens, segurança e refresh | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Tipos de autenticação:** Tipo 1 — Herança de Sessão JWT da plataforma Repasse Seguro (RN-DC-007); Tipo 2 — Vinculação WhatsApp via OTP (RN-DC-040–044)
> - **Sessão:** JWT access token 15min + refresh token 7 dias em httpOnly cookie; herdado da plataforma — sem novo login
> - **OAuth de plataformas externas:** não aplicável — a Dani não integra com provedores OAuth externos (Google, LinkedIn etc.); integrações externas são via API key (OpenAI, Supabase, Langfuse, EvolutionAPI)
> - **OTP WhatsApp:** 6 dígitos, 15min expiry, bcrypt hash; rate limit 3/hora + hard block 5 falhas consecutivas = 30min
> - **Regra de vault:** tokens JWT nunca em localStorage; refresh token em httpOnly cookie; OTP nunca em plain text (bcrypt)
> - **RBAC:** `CessionarioOwnerGuard` obrigatório em todos os endpoints; payload JWT contém `cessionario_id` e `role`
> - **Seções pendentes:** 0 — todas as seções cobertas pelos inputs D01, D02, D14, D16, D17

---

## 1. Persona e Contexto

**Público-alvo:** Backend Lead / Tech Lead responsável pela implementação de autenticação, guards NestJS e fluxos de OTP.

**Impacto no pipeline:**
- **D28 - Checklist de Qualidade:** valida se todos os guards estão presentes, se tokens nunca estão em plain text, se OTP usa bcrypt, se brute-force protection está implementada.

**Dependências:**
- D01 (RN-DC-007, RN-DC-040–044): regras de negócio de autenticação e OTP
- D02 (Stacks): JWT, Supabase Auth, Redis, NestJS guards
- D14 (Especificações Técnicas): módulos NestJS, Redis keys, flows críticos
- D16 (Documentação de API): endpoints de sessão e WhatsApp
- D17 (Integrações Externas): EvolutionAPI (Fase 2)

---

## 2. Dois Tipos de Autenticação

O produto AI-Dani-Cessionário opera com dois mecanismos distintos de autenticação:

| Tipo | Nome | Protocolo | Quando ocorre |
|---|---|---|---|
| **Tipo 1** | Herança de Sessão da Plataforma | JWT Bearer Token | Toda vez que o Cessionário abre o chat ou qualquer endpoint da Dani |
| **Tipo 2** | Vinculação WhatsApp via OTP | OTP 6 dígitos + bcrypt | Quando o Cessionário vincula seu número WhatsApp (Fase 2) |

> 🎯 **Decisão de arquitetura:** a Dani não implementa login próprio. Ela herda a sessão da plataforma Repasse Seguro (RN-DC-007). Isso elimina fricção de onboarding e mantém consistência de sessão entre módulos.

> ⚙️ **OAuth 2.0 externo:** não aplicável a este produto. As integrações externas (OpenAI, Supabase, Langfuse, EvolutionAPI) usam API keys estáticas gerenciadas via variáveis de ambiente (D17). Não há fluxo OAuth Authorization Code com provedores externos.

**Pseudocódigo resumo:**

```
// Tipo 1 — Toda requisição à API Dani
request → AuthGuard (valida JWT da plataforma)
        → CessionarioOwnerGuard (filtra por cessionario_id)
        → handler

// Tipo 2 — Vinculação WhatsApp (Fase 2)
POST /dani/whatsapp/vincular → gera OTP → envia via SMS/EvolutionAPI → armazena hash bcrypt no Redis
POST /dani/whatsapp/verificar-otp → valida OTP → rate limit check → atualiza perfil
```

---

## 3. Tipo 1 — Herança de Sessão da Plataforma

### 3.1 Fluxo de Sessão

```typescript
// Fluxo de acesso a qualquer endpoint da Dani

async function handleRequest(req: Request): Promise<void> {
  // Step 1: Extrai Bearer token do header Authorization
  const token = req.headers.authorization?.split(' ')[1]
  if (!token) {
    throw UnauthorizedException('AUTH_001: Token ausente')
  }

  // Step 2: Verifica JWT (assinado com JWT_ACCESS_SECRET da plataforma)
  let payload: JwtPayload
  try {
    payload = jwtService.verify(token, { secret: process.env.JWT_ACCESS_SECRET })
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      // Access token expirado → cliente deve usar refresh token
      throw UnauthorizedException('AUTH_002: Token expirado. Use o endpoint de refresh.')
    }
    throw UnauthorizedException('AUTH_003: Token inválido')
  }

  // Step 3: Valida estrutura obrigatória do payload
  if (!payload.cessionario_id || !payload.role) {
    throw UnauthorizedException('AUTH_004: Payload incompleto')
  }

  // Step 4: Verifica role (apenas CESSIONARIO e ADMIN permitidos nos endpoints Dani)
  if (!['CESSIONARIO', 'ADMIN'].includes(payload.role)) {
    throw ForbiddenException('AUTH_005: Role não autorizado para este recurso')
  }

  // Step 5: Injeta contexto na request
  req.user = {
    cessionario_id: payload.cessionario_id,
    role: payload.role,
    session_id: payload.session_id
  }

  // Step 6: CessionarioOwnerGuard valida que o recurso pertence ao cessionario_id
  // (executado por guard separado após AuthGuard)
  return next()
}
```

### 3.2 Fluxo de Refresh Token

```typescript
// POST /api/v1/auth/refresh
// Chamado automaticamente pelo frontend quando recebe 401 AUTH_002

async function refreshSession(req: Request): Promise<RefreshResponse> {
  // Step 1: Lê refresh token do httpOnly cookie
  const refreshToken = req.cookies['dani_refresh_token']
  if (!refreshToken) {
    throw UnauthorizedException('AUTH_006: Refresh token ausente')
  }

  // Step 2: Verifica refresh token
  let payload: RefreshPayload
  try {
    payload = jwtService.verify(refreshToken, { secret: process.env.JWT_REFRESH_SECRET })
  } catch {
    // Refresh token expirado ou inválido → forçar re-login na plataforma
    res.clearCookie('dani_refresh_token')
    throw UnauthorizedException('AUTH_007: Sessão expirada. Faça login novamente.')
  }

  // Step 3: Verifica se refresh token não foi revogado (Redis blacklist)
  const isRevoked = await redis.get(`dani:revoked:refresh:${payload.jti}`)
  if (isRevoked) {
    throw UnauthorizedException('AUTH_008: Refresh token revogado')
  }

  // Step 4: Gera novo access token
  const newAccessToken = jwtService.sign(
    {
      cessionario_id: payload.cessionario_id,
      role: payload.role,
      session_id: payload.session_id
    },
    { expiresIn: process.env.JWT_ACCESS_EXPIRY || '15m' }
  )

  // Step 5: Retorna novo access token
  return { access_token: newAccessToken }
}
```

### 3.3 CessionarioOwnerGuard

```typescript
// Guard aplicado em TODOS os endpoints que retornam dados do Cessionário

@Injectable()
export class CessionarioOwnerGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest()
    const user = request.user

    // Admin tem acesso irrestrito
    if (user.role === 'ADMIN') return true

    // Extrai cessionario_id do recurso (param, body ou query)
    const resourceCessionarioId =
      request.params.cessionario_id ||
      request.body?.cessionario_id ||
      request.query?.cessionario_id

    // Se o endpoint não tem cessionario_id explícito, injeta automaticamente do token
    if (!resourceCessionarioId) {
      request.body = { ...request.body, cessionario_id: user.cessionario_id }
      return true
    }

    // Valida que o recurso pertence ao cessionario autenticado
    if (resourceCessionarioId !== user.cessionario_id) {
      throw ForbiddenException('AUTH_009: Acesso negado a recurso de outro Cessionário')
    }

    return true
  }
}
```

### 3.4 Comportamento quando não há sessão ativa (RN-DC-007)

```
IF usuário abre chat SEM sessão ativa:
  → redirect para /login da plataforma Repasse Seguro
  → PRESERVA ponto de entrada original via query param: ?redirect=/oportunidade/OPR-XXXX-XXXX&open_chat=true
  → Após login bem-sucedido: plataforma redireciona de volta e abre o chat automaticamente
```

### 3.5 Tabela de Segurança da Sessão

| Regra | Valor | Referência |
|---|---|---|
| Access token lifetime | 15 minutos | D02, `JWT_ACCESS_EXPIRY=15m` |
| Refresh token lifetime | 7 dias | D02, `JWT_REFRESH_EXPIRY=7d` |
| Refresh token storage | httpOnly cookie (`dani_refresh_token`) | D02, anti-localStorage |
| Payload obrigatório | `cessionario_id`, `role`, `session_id` | D01 RN-DC-007, D02 |
| Roles permitidos nos endpoints Dani | `CESSIONARIO`, `ADMIN` | D01 §3, RBAC |
| Session timeout (Redis) | 7 dias (alinhado ao refresh token) | [DECISÃO AUTÔNOMA] Redis key `dani:session:{session_id}` TTL=604800s. Alternativa descartada: 24h — muito curto para uso diário. Critério: alinhamento com refresh token lifetime |
| Brute-force protection (login) | Gerenciado pela plataforma (não pela Dani) | D01 RN-DC-007: Dani herda sessão |
| 2FA / TOTP | Não aplicável — Dani herda sessão da plataforma | D01 RN-DC-007 |
| Tokens em plain text | Proibido — access token em memória JS; refresh em httpOnly cookie | D02, RNF de segurança |
| Audit log | Registrado via Pino: `cessionario_id`, `endpoint`, `timestamp`, `ip` | D02 §2.7 |

### 3.6 Endpoints de Sessão

| Método | Rota | Guard | Descrição |
|---|---|---|---|
| `POST` | `/api/v1/auth/refresh` | nenhum (usa cookie) | Renova access token usando refresh token do cookie httpOnly |
| `POST` | `/api/v1/auth/logout` | `AuthGuard` | Invalida sessão: revoga refresh token (Redis blacklist) e limpa cookie |
| `GET` | `/api/v1/auth/me` | `AuthGuard` + `CessionarioOwnerGuard` | Retorna dados do Cessionário autenticado |

> ⚙️ **Nota:** Login e registro são endpoints da plataforma Repasse Seguro, não da Dani. A Dani não expõe endpoint `POST /login`.

---

## 4. Tipo 2 — Vinculação WhatsApp via OTP (Fase 2)

> ⚙️ **Escopo:** exclusivo para o canal WhatsApp (Fase 2). Em Fase 1 (webchat), este fluxo não está ativo.

### 4.1 Visão Geral do Fluxo

O fluxo de vinculação WhatsApp não usa OAuth 2.0. Usa OTP (One-Time Password) enviado por SMS para verificar que o número de telefone pertence ao Cessionário autenticado. Após verificação, uma segunda etapa confirma via WhatsApp Business.

```
Fase 1 do OTP — SMS:
[Cessionário autenticado] → informa número → sistema envia OTP SMS
→ Cessionário insere OTP → sistema valida → avança para Fase 2

Fase 2 da Vinculação — WhatsApp:
[Sistema] → envia mensagem WhatsApp para o número → Cessionário confirma
→ sistema registra vinculação → perfil atualizado
```

### 4.2 Fluxo Detalhado — Etapa 1: Solicitar Vinculação

```typescript
// POST /api/v1/dani/whatsapp/vincular
// Requer: AuthGuard + CessionarioOwnerGuard

async function solicitarVinculacao(
  body: { phone: string },
  user: JwtPayload
): Promise<void> {
  // Step 1: Valida formato do número (Brasil: (XX) XXXXX-XXXX)
  const phoneRegex = /^\+55\d{2}9\d{8}$/
  if (!phoneRegex.test(normalizePhone(body.phone))) {
    throw BadRequestException('WA_001: Número de telefone inválido')
  }

  // Step 2: Verifica se número já está vinculado a OUTRO perfil
  const existingBinding = await prisma.whatsappBinding.findUnique({
    where: { phone_hash: hashPhone(body.phone) }
  })
  if (existingBinding && existingBinding.cessionario_id !== user.cessionario_id) {
    throw ConflictException('WA_002: Número já associado a outra conta')
  }

  // Step 3: Gera OTP de 6 dígitos
  const otp = generateSecureOTP(6)  // crypto.randomInt(100000, 999999)
  const otpHash = await bcrypt.hash(otp, 12)

  // Step 4: Armazena hash no Redis (nunca o OTP em plain text)
  // Key: dani:otp:{phone_hash} | TTL: 900s (15 minutos)
  const phoneHash = hashPhone(body.phone)  // SHA-256
  await redis.setex(`dani:otp:${phoneHash}`, 900, otpHash)

  // Step 5: Verifica rate limit (3 tentativas por hora por número)
  const rateLimitKey = `dani:rate:otp:${phoneHash}`
  const attempts = await redis.incr(rateLimitKey)
  if (attempts === 1) {
    await redis.expire(rateLimitKey, 3600)  // janela de 1 hora
  }
  if (attempts > 3) {
    throw TooManyRequestsException('WA_003: Limite de 3 tentativas por hora atingido')
  }

  // Step 6: Envia OTP via SMS (Fase 2: via EvolutionAPI WhatsApp ou SMS provider)
  await smsProvider.send({
    to: body.phone,
    message: `Seu código de verificação Repasse Seguro é: ${otp}. Válido por 15 minutos.`
  })

  // OTP nunca é retornado na response
  return { message: 'Código enviado. Verifique seu SMS.' }
}
```

### 4.3 Fluxo Detalhado — Etapa 2: Verificar OTP

```typescript
// POST /api/v1/dani/whatsapp/verificar-otp
// Requer: AuthGuard + CessionarioOwnerGuard

async function verificarOTP(
  body: { phone: string; otp: string },
  user: JwtPayload
): Promise<void> {
  const phoneHash = hashPhone(body.phone)

  // Step 1: Verifica hard block (5 falhas consecutivas independente da janela)
  const hardBlockKey = `dani:block:otp:${phoneHash}`
  const isBlocked = await redis.get(hardBlockKey)
  if (isBlocked) {
    const ttl = await redis.ttl(hardBlockKey)
    throw TooManyRequestsException(
      `WA_004: Número bloqueado por 30 minutos. Tente novamente em ${Math.ceil(ttl / 60)} minutos.`
    )
  }

  // Step 2: Recupera hash do OTP armazenado
  const otpHash = await redis.get(`dani:otp:${phoneHash}`)
  if (!otpHash) {
    throw BadRequestException('WA_005: Código expirado. Solicite um novo código.')
  }

  // Step 3: Valida OTP contra o hash (bcrypt compare — timing-safe)
  const isValid = await bcrypt.compare(body.otp, otpHash)

  if (!isValid) {
    // Step 3a: Incrementa contador de falhas consecutivas
    const failKey = `dani:otp:fails:${phoneHash}`
    const fails = await redis.incr(failKey)
    await redis.expire(failKey, 1800)  // TTL: 30min (duração do block)

    // Step 3b: Se atingiu 5 falhas consecutivas → hard block de 30min
    if (fails >= 5) {
      await redis.setex(hardBlockKey, 1800, '1')
      await redis.del(failKey)
      await redis.del(`dani:otp:${phoneHash}`)
      throw TooManyRequestsException('WA_006: Bloqueado por 30 minutos após 5 falhas consecutivas.')
    }

    throw BadRequestException(`WA_007: Código incorreto. ${5 - fails} tentativas restantes.`)
  }

  // Step 4: OTP válido — limpa keys de controle
  await redis.del(`dani:otp:${phoneHash}`)
  await redis.del(`dani:otp:fails:${phoneHash}`)
  await redis.del(hardBlockKey)

  // Step 5: Registra vinculação pendente (aguarda confirmação WhatsApp — Etapa 3)
  await prisma.whatsappBinding.upsert({
    where: { cessionario_id: user.cessionario_id },
    create: {
      cessionario_id: user.cessionario_id,
      phone_hash: phoneHash,
      status: 'PENDING_WHATSAPP_CONFIRM',
      otp_verified_at: new Date()
    },
    update: {
      phone_hash: phoneHash,
      status: 'PENDING_WHATSAPP_CONFIRM',
      otp_verified_at: new Date()
    }
  })

  // Step 6: Envia mensagem WhatsApp via EvolutionAPI (Fase 2)
  await evolutionApiService.sendConfirmationMessage(body.phone)

  return { message: 'OTP validado. Confirme pelo WhatsApp para concluir a vinculação.' }
}
```

### 4.4 Fluxo Detalhado — Etapa 3: Confirmação WhatsApp (RN-DC-042)

```typescript
// POST /api/v1/dani/whatsapp/webhook (recebido da EvolutionAPI)
// Processar resposta de confirmação do Cessionário no WhatsApp

async function handleWhatsappConfirmation(webhook: EvolutionWebhookPayload): Promise<void> {
  // Step 1: Valida assinatura do webhook (HMAC-SHA256)
  validateWebhookSignature(webhook, process.env.EVOLUTION_WEBHOOK_SECRET)

  // Step 2: Verifica se é mensagem de confirmação
  const isConfirmation = webhook.message?.toLowerCase().includes('confirmar') ||
                         webhook.message === '1'  // "Confirmar" ou "1"

  if (!isConfirmation) return

  // Step 3: Busca binding pendente pelo número
  const phoneHash = hashPhone(webhook.from)
  const binding = await prisma.whatsappBinding.findFirst({
    where: { phone_hash: phoneHash, status: 'PENDING_WHATSAPP_CONFIRM' }
  })

  if (!binding) return  // Sem binding pendente — ignora

  // Step 4: Finaliza vinculação
  await prisma.whatsappBinding.update({
    where: { id: binding.id },
    data: { status: 'ACTIVE', whatsapp_confirmed_at: new Date() }
  })

  // Step 5: Notifica plataforma web via SSE ou próxima requisição do Cessionário
  await redis.setex(`dani:whatsapp:linked:${binding.cessionario_id}`, 300, '1')
}
```

### 4.5 Tabela de Segurança OTP

| Regra | Valor | Referência |
|---|---|---|
| OTP tamanho | 6 dígitos | D01 Glossário OTP |
| OTP expiração | 15 minutos | D01 RN-DC-040 |
| OTP armazenamento | bcrypt hash (cost 12), nunca plain text | Segurança — prevenção de exposição |
| Rate limit por número | 3 tentativas/hora (janela deslizante) | D01 RN-DC-041 |
| Hard block | 5 falhas consecutivas = 30min bloqueio | D01 RN-DC-041 |
| Hard block TTL Redis | 1800s (30 min) | D02, D14 Redis keys |
| Rate limit key | `dani:rate:otp:{phone_hash}` | D14 Redis keys |
| Hard block key | `dani:block:otp:{phone_hash}` | D14 Redis keys |
| Phone hash | SHA-256 (nunca número em plain text nas keys Redis) | D15 Arquitetura |
| Cooldown reenvio OTP | 60 segundos | D01 RN-DC-041 |
| Webhook signature | HMAC-SHA256 com `EVOLUTION_WEBHOOK_SECRET` | D17 EvolutionAPI |

---

## 5. Plataformas Integradas — Autenticação

> 🎯 **Contexto:** as integrações externas da Dani não usam OAuth 2.0 Authorization Code. São autenticadas via API keys estáticas ou JWT. Não há fluxo de redirect, callback ou PKCE.

| Plataforma | Tipo de Auth | Credencial | Sensível | Rotação | Referência |
|---|---|---|---|---|---|
| **OpenAI GPT-4o** | API Key estática | `OPENAI_API_KEY` em `.env` | Sim | Manualmente (revogar no dashboard OpenAI) | D17 §2 |
| **Supabase** | Service Role JWT | `SUPABASE_SERVICE_ROLE_KEY` em `.env` | Sim (P0) | Nunca expor no frontend; rotação via Supabase dashboard | D17 §3 |
| **Langfuse** | Secret Key + Public Key | `LANGFUSE_SECRET_KEY`, `LANGFUSE_PUBLIC_KEY` | Sim | Via Langfuse dashboard | D17 §4 |
| **Redis / Upstash** | URL com senha embutida | `REDIS_URL=rediss://:senha@host:6380` | Sim | Via Upstash dashboard | D17 §5 |
| **EvolutionAPI** | API Key estática | `EVOLUTION_API_KEY` em `.env` | Sim | Via EvolutionAPI admin | D17 §6 |

> ⚙️ **Regra absoluta:** nenhuma API key pode ter prefixo `VITE_`. Isso exporia a chave ao browser. Todas as chamadas a serviços externos são feitas exclusivamente pelo backend NestJS.

---

## 6. Refresh Proativo e Reconexão

### 6.1 Refresh do Access Token (Frontend)

O frontend implementa interceptor no cliente HTTP para renovar o access token proativamente:

```typescript
// Interceptor TanStack Query / fetch wrapper
async function authInterceptor(request: Request, next: Handler): Promise<Response> {
  const response = await next(request)

  // Se receber 401 com código AUTH_002 (token expirado)
  if (response.status === 401) {
    const body = await response.json()
    if (body.error?.code === 'AUTH_002') {
      // Tenta refresh automático (máx 1 retry)
      const refreshed = await refreshAccessToken()
      if (refreshed) {
        // Retry da requisição original com novo token
        return next(addAuthHeader(request, refreshed.access_token))
      }
      // Refresh falhou → redireciona para login da plataforma
      redirectToPlatformLogin()
    }
  }

  return response
}
```

### 6.2 Estados de Sessão

```
AUTENTICADO ──────────────────────── access token válido (< 15min)
     │
     │ token expirado (15min)
     ▼
REFRESH PENDENTE ─────────────────── POST /auth/refresh com cookie httpOnly
     │
     ├── refresh ok ──────────────── AUTENTICADO (novo access token)
     │
     └── refresh falhou ──────────── NÃO AUTENTICADO → redirect login
           (expirado ou revogado)

NÃO AUTENTICADO ──────────────────── redirect para /login da plataforma
                                      + ?redirect=<ponto de entrada original>
```

### 6.3 Estados da Vinculação WhatsApp

```
NÃO VINCULADO ────────────────────── Cessionário não tem número vinculado
     │
     │ POST /dani/whatsapp/vincular
     ▼
OTP_ENVIADO ──────────────────────── OTP enviado, aguarda verificação (15min TTL)
     │
     ├── OTP expirou (15min) ──────── NÃO VINCULADO (deve reiniciar)
     │
     ├── 5 falhas consecutivas ────── BLOQUEADO (30min) → NÃO VINCULADO após TTL
     │
     │ POST /dani/whatsapp/verificar-otp (OTP correto)
     ▼
PENDING_WHATSAPP_CONFIRM ─────────── OTP validado, aguarda confirmação no WhatsApp
     │
     │ Webhook EvolutionAPI (confirmação)
     ▼
ATIVO ────────────────────────────── Vinculação completa
     │
     │ POST /dani/whatsapp/desvincular
     ▼
NÃO VINCULADO
```

### 6.4 Circuit Breaker — EvolutionAPI (Fase 2)

```typescript
// Circuit breaker para falhas na EvolutionAPI
// Estado: CLOSED (normal) → OPEN (falhou 3x) → HALF-OPEN (teste após 30s)

if (consecutiveFailures >= 3) {
  // OPEN: falha rápida por 30 segundos
  throw ServiceUnavailableException('WA_008: Canal WhatsApp temporariamente indisponível')
}

// Retry policy para cada chamada: 3x com backoff 5s/15s/30s (D17 §6)
const result = await retry(
  () => evolutionApiService.send(payload),
  { times: 3, delays: [5000, 15000, 30000] }
)

// Se esgotou retries → envia para DLQ
if (!result) {
  await rabbitmq.publish('dani.whatsapp.dlq', payload)
}
```

---

## 7. Checklist de Segurança

| # | Verificação | Implementação | Status |
|---|---|---|---|
| S-01 | Tokens JWT nunca em plain text em banco ou Redis | Access token: memória JS; refresh token: httpOnly cookie; sem armazenamento em BD | ✅ Obrigatório |
| S-02 | OTP nunca em plain text | bcrypt hash (cost 12) armazenado no Redis; OTP gerado com `crypto.randomInt` | ✅ Obrigatório |
| S-03 | `CessionarioOwnerGuard` em todos os endpoints de dados | Guard aplicado via decorator em todos os controllers `AgentModule`, `OportunidadeModule`, `AlertaModule`, `WhatsappModule` | ✅ Obrigatório |
| S-04 | Payload JWT contém `cessionario_id` e `role` | Validado em `AuthGuard` antes de qualquer handler | ✅ Obrigatório |
| S-05 | Rate limit OTP (3/hora por número) | Redis `dani:rate:otp:{phone_hash}` com janela deslizante de 3600s | ✅ Obrigatório |
| S-06 | Hard block OTP (5 falhas = 30min) | Redis `dani:block:otp:{phone_hash}` TTL=1800s; verificado ANTES do bcrypt compare | ✅ Obrigatório |
| S-07 | Phone hash em Redis keys (nunca número raw) | SHA-256 do número normalizado antes de qualquer operação Redis | ✅ Obrigatório |
| S-08 | API keys externas nunca com prefixo `VITE_` | Verificação em lint rule + code review | ✅ Obrigatório |
| S-09 | Webhook EvolutionAPI validado por assinatura HMAC-SHA256 | `validateWebhookSignature()` antes de qualquer processamento | ✅ Obrigatório (Fase 2) |
| S-10 | Session timeout alinhado ao refresh token | Redis `dani:session:{session_id}` TTL=604800s (7 dias) | ✅ Obrigatório |
| S-11 | Refresh token revogado no logout | Redis blacklist `dani:revoked:refresh:{jti}` | ✅ Obrigatório |
| S-12 | `SUPABASE_SERVICE_ROLE_KEY` apenas no backend | Nunca no frontend, nunca com prefixo `VITE_`, apenas em variáveis de servidor | ✅ Crítico P0 |
| S-13 | Audit log de acesso com `cessionario_id` | Pino structured log: `cessionario_id`, `endpoint`, `method`, `ip`, `timestamp` | ✅ Obrigatório |

---

## 8. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Criação. Herança de sessão JWT (RN-DC-007) + OTP WhatsApp (RN-DC-040–044). Sem OAuth 2.0 externo (integrações via API key estática). |

---

## 9. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Session timeout: 7 dias alinhado ao refresh | [DECISÃO AUTÔNOMA] Escolhido 7 dias (= refresh token lifetime). Alternativa: 24h — muito curto para uso diário. Critério: consistência com refresh lifetime. | §3.5 | Alinhamento UX vs segurança | P2 | Backend Lead | Concluído |
| OAuth 2.0 externo: não aplicável | [DECISÃO AUTÔNOMA] Documentado como "não aplicável" pois D17 confirma que todas as integrações usam API key estática. Sem redirect URI, sem state CSRF, sem PKCE necessários. | §5 | Ausência de provedores OAuth | P2 | Tech Lead | Concluído |
| SMS Provider para OTP (Fase 1) | [DEFINIÇÃO PENDENTE] Opção A: Twilio SMS (reconhecimento de mercado, SDK Node.js robusto, custo ~$0.0075/SMS). Opção B: AWS SNS SMS (integração com infra AWS, custo ~$0.00645/SMS). Trade-off: Twilio tem melhor DX; AWS tem integração nativa com stack de infra. | §4.2 | Afeta custo operacional e DX | P1 | DevOps | Pendente |
| 2FA TOTP para role ADMIN | [DECISÃO AUTÔNOMA] TOTP não implementado na Dani — gerenciado pela plataforma Repasse Seguro (Dani herda sessão). Alternativa descartada: implementar TOTP próprio — duplica complexidade sem ganho, pois Admin já usa TOTP na plataforma. | §3.1 | Não reinventar auth da plataforma | P2 | Tech Lead | Concluído |
