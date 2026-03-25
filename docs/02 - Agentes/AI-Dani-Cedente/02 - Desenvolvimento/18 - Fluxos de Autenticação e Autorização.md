# 18 - Fluxos de Autenticação e Autorização — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Backend Lead / Tech Lead |
| Escopo | Sessão de usuário JWT, autorização RBAC, isolamento por cedente_id, segurança e rate limiting |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D01 · D02 · D05 · D14 · D16 · D17 |

---

> **📌 TL;DR**
>
> - **Tipos de auth:** 1 — Sessão JWT Supabase Auth (Cedente e Admin). Sem OAuth social — autenticação delegada à Plataforma Repasse Seguro.
> - **Sessão:** JWT RS256, access token TTL 1h, refresh gerenciado pelo frontend via Supabase SDK.
> - **Isolamento de dados:** `CedenteIsolationMiddleware` injeta `cedente_id` em todo request — PII masking obrigatório antes de logs/LLM.
> - **Roles:** `cedente` e `admin`. RBAC simples — 2 roles, sem hierarquia complexa.
> - **JWT validado localmente:** `JwtAuthGuard` usa chave pública RSA do Supabase — sem request HTTP por validação.
> - **2FA:** não aplicável no AI-Dani-Cedente — autenticação 2FA gerenciada pela Plataforma Repasse Seguro (upstream).
> - **Zero seções pendentes** — cobertura 100% dos fluxos aplicáveis a este serviço.

---

> **🎯 Nota de arquitetura:**
>
> O AI-Dani-Cedente é um **serviço backend-only**. Ele **não autentica usuários diretamente** — a autenticação (login, 2FA, refresh de token) é responsabilidade da **Plataforma Repasse Seguro** (sistema upstream) via Supabase Auth. O AI-Dani-Cedente **apenas valida JWTs** já emitidos pela plataforma.
>
> Portanto, este documento cobre:
> - Validação de JWT incoming (JwtAuthGuard)
> - Extração e injeção de `cedente_id` (CedenteIsolationMiddleware)
> - Autorização por role (RbacGuard)
> - Isolamento de dados por Cedente (RLS + middleware)
> - Segurança de requests e proteção contra abuso

---

## 1. Persona e Impacto no Pipeline

**Público:** Backend developers implementando guards, middleware e interceptors de autenticação e autorização.

**Impacto:**
- **28 - Checklist de Qualidade:** valida se a implementação segue o checklist de segurança definido neste documento.
- **22 - Guia de Ambiente:** deriva variáveis `SUPABASE_JWT_SECRET` e `SUPABASE_URL`.
- Se um fluxo de auth não estiver documentado aqui, não deve ser inventado no código.

---

## 2. Tipos de Auth no AI-Dani-Cedente

| Tipo | Descrição | Responsável |
|---|---|---|
| **Tipo 1 — Emissão de JWT** | Login, 2FA, refresh | Plataforma Repasse Seguro (upstream — fora do escopo do AI-Dani) |
| **Tipo 2 — Validação de JWT** | Verificar token incoming em cada request | AI-Dani-Cedente (`JwtAuthGuard`) |
| **Tipo 3 — Autorização RBAC** | Verificar role (`cedente` / `admin`) | AI-Dani-Cedente (`RbacGuard`) |
| **Tipo 4 — Isolamento por Cedente** | Injetar e enforçar `cedente_id` em todos os dados | AI-Dani-Cedente (`CedenteIsolationMiddleware` + RLS) |

---

## 3. Tipo 2 — Validação de JWT (JwtAuthGuard)

### 3.1 Fluxo de Validação

```typescript
// Pseudocódigo: JwtAuthGuard.canActivate()

async canActivate(context: ExecutionContext): Promise<boolean> {
  // 1. Extrair token do header Authorization: Bearer {token}
  const token = extractBearerToken(request.headers.authorization)
  if (!token) {
    throw new UnauthorizedException('DCE-AUTH-4010_003: Token ausente.')
  }

  // 2. Verificar assinatura e expiração (validação LOCAL com chave pública RSA)
  // Sem request HTTP — usa SUPABASE_JWT_SECRET (chave pública)
  try {
    const payload = jwt.verify(token, SUPABASE_JWT_SECRET, {
      algorithms: ['RS256'],
      issuer: `https://${SUPABASE_PROJECT_REF}.supabase.co/auth/v1`
    })
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      throw new UnauthorizedException('DCE-AUTH-4010_001: Sessão expirada.')
    }
    throw new UnauthorizedException('DCE-AUTH-4010_002: Token inválido.')
  }

  // 3. Extrair claims obrigatórios
  const cedenteId = payload.sub       // UUID do Cedente
  const role = payload.role           // 'cedente' | 'admin'
  const expiresAt = payload.exp       // Unix timestamp

  if (!cedenteId || !role) {
    throw new UnauthorizedException('DCE-AUTH-4010_004: Claims obrigatórios ausentes.')
  }

  // 4. Injetar no request context para uso downstream
  request.user = { cedenteId, role, expiresAt }
  return true
}
```

### 3.2 Claims JWT Esperados

| Claim | Tipo | Valor esperado | Uso |
|---|---|---|---|
| `sub` | `string (UUID)` | UUID do Cedente na plataforma | `cedente_id` em todos os dados |
| `role` | `string` | `cedente` ou `admin` | Autorização RBAC |
| `exp` | `number` | Unix timestamp | Validação de expiração |
| `iss` | `string` | URL do Supabase Auth | Validação de issuer |
| `aud` | `string` | `authenticated` | Supabase padrão |

### 3.3 Tabela de Segurança da Sessão

| Regra | Valor |
|---|---|
| Algoritmo de assinatura | RS256 (chave pública Supabase) |
| Access token TTL | 3600s (1 hora) |
| Refresh token TTL | 604800s (7 dias) |
| Validação local vs remota | **Local** — sem request HTTP por validação |
| Refresh de token | Responsabilidade do frontend (Supabase SDK) |
| 2FA | Gerenciado pela Plataforma Repasse Seguro — não validado neste serviço |
| Brute-force protection | Rate limiting via Redis (100 req/min por IP) — não é brute-force de senha |
| Session timeout | Expiração implícita no JWT (`exp` claim) |
| Token storage | httpOnly cookie ou memory — responsabilidade do frontend |
| Logout | Frontend chama `supabase.auth.signOut()` — AI-Dani não tem endpoint de logout |

### 3.4 Endpoints de Auth no AI-Dani-Cedente

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| `GET` | `/api/v1/auth/me` | JWT | Retorna perfil do Cedente autenticado |
| `POST` | `/api/v1/auth/validate` | JWT | Valida se o JWT é válido e retorna payload |

> 💡 **Nota:** não há endpoints de login, refresh ou logout no AI-Dani-Cedente. Esses fluxos são gerenciados pela Plataforma Repasse Seguro.

---

## 4. Tipo 3 — Autorização RBAC (RbacGuard)

### 4.1 Roles Definidos

| Role | Descrição | Permissões |
|---|---|---|
| `cedente` | Cedente (proprietário do financiamento) | Acesso apenas aos próprios dados — todos os endpoints públicos |
| `admin` | Equipe interna Repasse Seguro | Acesso a todos os dados + endpoints `/admin/*` |

### 4.2 Fluxo de Autorização

```typescript
// Pseudocódigo: RbacGuard.canActivate()

async canActivate(context: ExecutionContext): Promise<boolean> {
  // 1. Obter roles permitidos via decorator @RequireRole('admin')
  const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler())
  if (!requiredRoles || requiredRoles.length === 0) {
    return true // Rota sem role restriction — apenas JWT válido necessário
  }

  // 2. Verificar role do usuário (injetado pelo JwtAuthGuard)
  const { role } = request.user
  if (!requiredRoles.includes(role)) {
    throw new ForbiddenException('DCE-AUTH-4030_001: Sem permissão para esta ação.')
  }

  return true
}
```

### 4.3 Uso do Decorator

```typescript
// Em controllers que requerem role admin
@Get('metrics/csat')
@UseGuards(JwtAuthGuard, RbacGuard)
@RequireRole('admin')
async getCsatMetrics() { ... }

// Em controllers que aceitam qualquer role autenticado
@Post('sessions')
@UseGuards(JwtAuthGuard)
async createSession() { ... }
```

### 4.4 Matriz de Permissões por Domínio

| Domínio | Role `cedente` | Role `admin` | Observações |
|---|---|---|---|
| `GET /auth/me` | ✅ Próprio perfil | ✅ | — |
| `POST /chat/sessions` | ✅ | ✅ | Cedente cria sessões para si mesmo |
| `GET /chat/sessions` | ✅ Apenas suas sessões | ✅ Todas | RLS + middleware |
| `POST /chat/sessions/:id/messages` | ✅ | ✅ Via takeover | — |
| `GET /opportunities` | ✅ Apenas suas | ✅ Todas | RLS |
| `GET /opportunities/:id/scenarios` | ✅ Apenas suas | ✅ Todas | Dados confidenciais — RLS |
| `POST /proposals/:id/accept` | ✅ Próprias propostas | ✅ | Confirmação explícita obrigatória |
| `GET /proposals/:id/escrow` | ✅ Próprias | ✅ Todas | — |
| `GET /admin/*` | 🚫 403 | ✅ | Exclusivo admin |
| `POST /admin/sessions/:id/takeover` | 🚫 403 | ✅ | — |
| `GET /admin/metrics/csat` | 🚫 403 | ✅ | — |
| `POST /admin/fallback/enable` | 🚫 403 | ✅ | — |
| `POST /admin/rag/ingest` | 🚫 403 | ✅ | — |

---

## 5. Tipo 4 — Isolamento por Cedente (CedenteIsolationMiddleware)

### 5.1 Conceito

O isolamento por `cedente_id` é o **princípio de segurança central** do AI-Dani-Cedente (RN-DCE-001). Todo dado pertence a um Cedente e nenhum Cedente deve acessar dados de outro.

A responsabilidade é dupla:
1. **Middleware (aplicação):** injeta e valida `cedente_id` em todo request.
2. **RLS (banco de dados):** políticas Row Level Security no Supabase PostgreSQL garantem isolamento mesmo se o middleware falhar.

### 5.2 Fluxo do CedenteIsolationMiddleware

```typescript
// Pseudocódigo: CedenteIsolationMiddleware

async use(req: Request, res: Response, next: NextFunction): Promise<void> {
  // 1. Extrair cedente_id do JWT (já validado pelo JwtAuthGuard)
  const cedenteId = req.user?.cedenteId
  if (!cedenteId) {
    throw new UnauthorizedException('DCE-AUTH-4010_005: cedente_id ausente no token.')
  }

  // 2. Injetar cedente_id no request context para uso no Prisma service
  req.cedenteId = cedenteId

  // 3. Injetar no PrismaService via AsyncLocalStorage
  // Todos os queries subsequentes incluem WHERE cedente_id = cedenteId automaticamente
  prismaContext.run({ cedenteId }, next)
}
```

### 5.3 Validação Cross-Cedente em Queries

```typescript
// Pseudocódigo: Exemplo de como o Repository pattern enforça isolamento

async findOpportunityById(id: string, cedenteId: string) {
  const opportunity = await this.prisma.opportunity.findFirst({
    where: {
      id,
      cedente_id: cedenteId,  // SEMPRE inclui cedente_id — nunca busca por id sozinho
      deleted_at: null
    }
  })

  if (!opportunity) {
    // Retorna 404 (não 403) para não revelar existência de dados de outros Cedentes
    throw new NotFoundException('DCE-OPP-4040_001: Oportunidade não encontrada.')
  }

  return opportunity
}
```

> 🔴 **Regra de segurança crítica:** retornar `404` (não `403`) quando um Cedente tenta acessar recurso de outro Cedente. `403` revelaria a existência do recurso. `404` é a resposta correta para qualquer recurso não pertencente ao Cedente autenticado.

### 5.4 RLS Policies (Supabase PostgreSQL)

```sql
-- Exemplo de policy RLS para a tabela opportunities
-- Aplicada em todas as tabelas de domínio do Cedente

CREATE POLICY "cedente_isolation" ON opportunities
  FOR ALL
  USING (cedente_id = auth.uid());

-- auth.uid() retorna o cedente_id do JWT atual (extraído pelo Supabase Auth)
-- Garante isolamento mesmo se o middleware da aplicação falhar
```

**Tabelas com RLS ativa:**

| Tabela | Policy name | Condição |
|---|---|---|
| `opportunities` | `cedente_isolation` | `cedente_id = auth.uid()` |
| `opportunity_scenarios` | `cedente_isolation` (via opportunity) | JOIN com opportunities |
| `proposals` | `cedente_isolation` (via opportunity) | JOIN com opportunities |
| `dossier_documents` | `cedente_isolation` (via opportunity) | JOIN com opportunities |
| `escrow_transactions` | `cedente_isolation` (via proposal) | JOIN com proposals + opportunities |
| `chat_sessions` | `cedente_isolation` | `cedente_id = auth.uid()` |
| `chat_messages` | `cedente_isolation` (via session) | JOIN com chat_sessions |

---

## 6. PII Masking

### 6.1 Conceito

Antes de qualquer envio para sistemas externos (OpenAI API, Langfuse, logs) e antes de armazenar mensagens no banco, o conteúdo deve ter PII mascarado.

### 6.2 Fluxo do PiiMaskingMiddleware

```typescript
// Pseudocódigo: PiiMaskingMiddleware

function maskPii(content: string): string {
  // CPF: ***.***.***-XX
  content = content.replace(/\d{3}\.\d{3}\.\d{3}-\d{2}/g, '***.***.***-**')

  // Telefone: (XX) XXXXX-XXXX → (XX) *****-****
  content = content.replace(/\(\d{2}\)\s?\d{4,5}-\d{4}/g, '(**) *****-****')

  // Email: j***@d***.com
  content = content.replace(/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g,
    (email) => maskEmail(email))

  // RG: XXXXXXXXX → *********
  content = content.replace(/\b\d{7,9}\b/g, '*'.repeat(9))

  return content
}

// Aplicado em:
// 1. ChatMessage.content antes de INSERT no banco
// 2. LLM request antes de enviar à OpenAI API
// 3. Langfuse trace antes de ingerir
// 4. Logs do LoggingInterceptor (headers Authorization redacted)
```

### 6.3 Campos nunca armazenados

| Campo | Motivo |
|---|---|
| CPF completo | Apenas `cpf_masked` (`***.***.***-XX`) |
| `cessionario_id` | Não armazenado no AI-Dani-Cedente |
| Credenciais/passwords | Nunca trafegam pelo serviço |
| Tokens OAuth de terceiros | Não aplicável — sem OAuth de plataformas externas |

---

## 7. Rate Limiting e Proteção contra Abuso

### 7.1 Fluxo RateLimitGuard (Chat)

```typescript
// Pseudocódigo: RateLimitGuard para endpoint de mensagens

async canActivate(context: ExecutionContext): Promise<boolean> {
  const { cedenteId } = request.user
  const hourTimestamp = Math.floor(Date.now() / 3600000) // truncar para hora atual

  const key = `dani:rate_limit:${cedenteId}:${hourTimestamp}`

  // Sliding window via Redis INCR
  const count = await redis.incr(key)
  if (count === 1) {
    await redis.expire(key, 3600) // TTL 1 hora
  }

  if (count > RATE_LIMIT_CHAT_PER_HOUR) { // default: 30
    const ttl = await redis.ttl(key)
    const retryAfter = ttl

    throw new TooManyRequestsException({
      code: 'DCE-CHAT-4290_001',
      message: `Limite de ${RATE_LIMIT_CHAT_PER_HOUR} mensagens/hora atingido.`,
      details: { retry_after: retryAfter, limit: RATE_LIMIT_CHAT_PER_HOUR, window: '1h' }
    })
  }

  return true
}
```

### 7.2 Redis Keys de Rate Limiting

| Key | TTL | Escopo | Limite |
|---|---|---|---|
| `dani:rate_limit:{cedente_id}:{hour_ts}` | 3600s | Por Cedente, por hora | 30 msg/h |
| (Rate limit global) | 60s | Por IP | 100 req/min |

### 7.3 Headers de Resposta

```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 27
X-RateLimit-Reset: 1711190400
```

---

## 8. Refresh Proativo e Estado da Sessão

> 💡 **Responsabilidade do frontend:** o AI-Dani-Cedente não implementa refresh proativo de token — isso é responsabilidade do frontend da Plataforma Repasse Seguro via Supabase SDK.

O AI-Dani-Cedente apenas responde a tokens expirados com `401` e o código de erro adequado para o frontend agir:

| Situação | Código HTTP | Error Code | Ação do Frontend |
|---|---|---|---|
| Token expirado | 401 | `DCE-AUTH-4010_001` | Chama `supabase.auth.refreshSession()` e retenta |
| Token inválido | 401 | `DCE-AUTH-4010_002` | Redireciona para login |
| Token ausente | 401 | `DCE-AUTH-4010_003` | Redireciona para login |
| Refresh falhou (sessão expirada) | 401 | `DCE-AUTH-4010_001` | Redireciona para login |

---

## 9. Checklist de Segurança

| # | Verificação | Implementação | Onde |
|---|---|---|---|
| S-01 | JWT nunca em plain text em logs | `LoggingInterceptor` redact `Authorization` header | `src/common/interceptors/logging.interceptor.ts` |
| S-02 | `cedente_id` extraído do JWT, nunca do body | `JwtAuthGuard` + `CedenteIsolationMiddleware` | `src/common/guards/jwt-auth.guard.ts` |
| S-03 | RLS ativa em todas as tabelas de domínio | Policies em `prisma/rls/policies.sql` | PostgreSQL |
| S-04 | 404 (não 403) para recursos de outro Cedente | Padrão em todos os repositories | `src/modules/*/**.repository.ts` |
| S-05 | PII mascarado antes de logs e LLM | `PiiMaskingMiddleware` | `src/common/middleware/pii-masking.middleware.ts` |
| S-06 | `cedente_id` como hash SHA-256 em analytics | `PostHogService.hashId()` | `src/infrastructure/posthog/posthog.service.ts` |
| S-07 | Rate limiting 30 msg/h por Cedente | `RateLimitGuard` + Redis sliding window | `src/common/guards/rate-limit.guard.ts` |
| S-08 | Webhook ZapSign validado por HMAC-SHA256 | `ZapSignWebhookGuard` | `src/modules/proposal/zapsign-webhook.guard.ts` |
| S-09 | Endpoints admin protegidos por `@RequireRole('admin')` | `RbacGuard` | `src/modules/admin/*.controller.ts` |
| S-10 | Secrets apenas como env vars — nunca hardcoded | `ConfigurationService` com Joi/zod | `src/config/configuration.ts` |
| S-11 | Headers HTTPS-only (sem mixed content) | Configuração de CORS no NestJS | `src/main.ts` (CORS config) |
| S-12 | `SUPABASE_SERVICE_ROLE_KEY` nunca exposta ao frontend | Usada apenas em Storage backend | `src/infrastructure/supabase/storage.service.ts` |

---

## 10. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — JWT validation, RBAC 2 roles, CedenteIsolationMiddleware, PII masking, rate limiting, checklist de segurança 12 itens. |

---

## 11. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| 2FA para role admin | `[DECISÃO AUTÔNOMA]` | 3.3 | 2FA gerenciado pela Plataforma Repasse Seguro (upstream). AI-Dani-Cedente não valida 2FA — assume que o JWT emitido pós-2FA já é seguro. Alternativa descartada: 2FA próprio no AI-Dani — duplicação desnecessária. Critério: single source of truth para auth. | Médio | Auth Lead | Decidido |
| Token blacklist para logout | `[DECISÃO AUTÔNOMA]` | 3.3 | Sem blacklist de tokens no AI-Dani-Cedente. Tokens expiram naturalmente (TTL 1h). Logout invalida no Supabase Auth (upstream). Alternativa descartada: blacklist Redis — overhead desnecessário dado TTL curto. | Baixo | Backend | Decidido |
| Audit log de conexões e takeovers | `[DECISÃO AUTÔNOMA]` | Seção 9 | Logs de takeover registrados em `chat_sessions.admin_takeover_at` + Langfuse trace. Audit log formal centralizado não implementado na Fase 1. Revisar para Fase 2 se requisito regulatório emergir. | Médio | DevOps | Fase 2 |
