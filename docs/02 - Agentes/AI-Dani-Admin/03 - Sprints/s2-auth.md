# S2 — Autenticação e RBAC

## Metadados

| Campo         | Valor                                               |
| ------------- | --------------------------------------------------- |
| Sprint        | S2                                                  |
| Nome          | Autenticação e RBAC                                 |
| Template      | A (Infraestrutura)                                  |
| Módulo        | AI-Dani-Admin                                       |
| Docs Fonte    | D02, D18, D05 (RF-026, RNF-006 a RNF-008), D16, D20 |
| REQs Cobertos | REQs de S2 no registro-mestre.md                    |
| Data          | 2026-03-24                                          |

## Objetivo

Implementar autenticação JWT herdada da plataforma central (Supabase Auth), validação de role `ADMIN` em todos os endpoints do módulo AI-Dani-Admin, guard `JwtAuthGuard`, guard `RolesGuard` com decorator `@Roles('ADMIN')`, bloqueio de Cessionário/Cedente com 403, endpoint interno protegido por API Key, e log de auditoria de acesso.

---

## BANCO DE DADOS

> S2 não cria novas tabelas. Usa `admin_access_logs` criada em S1 para registrar ações críticas. Verifica que a tabela existe e está acessível.

### 1. Verificação de pré-requisitos de banco

- [x] Confirmar que tabela `admin_access_logs` foi criada pela migration `005_create_admin_access_logs` (S1)
- [x] Confirmar que campos `admin_id`, `action`, `target_type`, `target_id`, `before_state`, `after_state`, `ip_address`, `user_agent`, `created_at` existem com os tipos corretos
- [x] Confirmar que índices `(admin_id, created_at DESC)`, `(action, created_at DESC)`, `(target_type, target_id)` existem

---

## BACKEND

### 2. ConfigModule — variáveis de JWT

- [x] Adicionar ao `ConfigModule.forRoot()` validação das variáveis obrigatórias de auth:
  - `SUPABASE_JWT_SECRET` — usado para verificar tokens emitidos pelo Supabase Auth
  - `JWT_SECRET` — fallback/custom JWT secret
  - `INTERNAL_API_KEY` — para endpoint `/api/v1/internal/interactions`
- [x] Criar `apps/api/src/common/config/auth.config.ts` com `registerAs('auth', () => ({ jwtSecret, internalApiKey }))` extraindo do `process.env`
- [x] Verificar que aplicação falha ao subir se `SUPABASE_JWT_SECRET` não estiver definido (ConfigModule `validationSchema`)

### 3. JwtStrategy (Passport)

- [x] Criar `apps/api/src/common/strategies/jwt.strategy.ts`:
  - Extende `PassportStrategy(Strategy, 'jwt')`
  - `jwtFromRequest`: `ExtractJwt.fromAuthHeaderAsBearerToken()`
  - `secretOrKey`: `configService.get('auth.jwtSecret')` — usa `SUPABASE_JWT_SECRET`
  - `validate(payload)`: retorna objeto com `{ sub: payload.sub, email: payload.email, role: payload.role }` — NÃO busca no banco, apenas decodifica o JWT
  - Rejeita tokens expirados (padrão passport-jwt)
  - Não emite tokens — AI-Dani-Admin apenas valida (D18)

### 4. JwtAuthGuard

- [x] Criar `apps/api/src/common/guards/jwt-auth.guard.ts`:
  - Extende `AuthGuard('jwt')`
  - Em falha de autenticação, retorna erro `DA-AUTH-001` com HTTP 401 e mensagem `'Token inválido ou expirado'` (formato `{ error: { code: 'DA-AUTH-001', message: '...' } }`)
  - Implementar `canActivate` que chama `super.canActivate(context)` e captura `UnauthorizedException`

### 5. RolesGuard e decorator @Roles

- [x] Criar `apps/api/src/common/decorators/roles.decorator.ts`:
  - `export const Roles = (...roles: string[]) => SetMetadata('roles', roles)`
- [x] Criar `apps/api/src/common/guards/roles.guard.ts`:
  - Implementa `CanActivate`
  - Lê `roles` do Reflector via `'roles'` metadata key
  - Extrai `user.role` do request (populado pelo JwtAuthGuard)
  - Se role não está na lista permitida → retorna `ForbiddenException` com erro `DA-AUTH-002` e mensagem `'Acesso restrito a administradores'` (HTTP 403)
  - Role `ADMIN` tem acesso a TODOS os endpoints do módulo (RF-026)
  - Role `CESSIONARIO` ou `CEDENTE` → sempre 403 em qualquer endpoint do módulo AI-Dani-Admin

### 6. InternalApiKeyGuard

- [x] Criar `apps/api/src/common/guards/internal-api-key.guard.ts`:
  - Extrai header `X-Internal-Api-Key` da request
  - Compara com `configService.get('auth.internalApiKey')` usando `timingSafeEqual` (evitar timing attacks)
  - Em falha → HTTP 401 com erro `DA-AUTH-003`, mensagem `'API Key interna inválida'`
  - Usado exclusivamente em `POST /api/v1/internal/interactions` (D19) — não requer JWT

### 7. AuditLogService

- [x] Criar `apps/api/src/common/services/audit-log.service.ts`:
  - Injeta `PrismaService`
  - Método `log(dto: CreateAuditLogDto)`: cria registro em `admin_access_logs` — [CORRIGIDO: FINDING-005]
  - `CreateAuditLogDto`: `{ adminId: string, action: AdminActionType, targetType?: string, targetId?: string, beforeState?: object, afterState?: object, ipAddress?: string, userAgent?: string }`
  - `action` usa enum `AdminActionType` com 7 valores PT-BR (D12/REQ-184): `ACESSO_PAINEL`, `TAKEOVER_INICIADO`, `TAKEOVER_ENCERRADO`, `THRESHOLD_ALTERADO`, `RATE_LIMIT_ALTERADO`, `AGENTE_REATIVADO`, `LANCAMENTO_AUTORIZADO`
  - Operação é `INSERT` apenas — nunca `UPDATE` ou `DELETE` (tabela append-only)
  - Em falha de DB: loga erro via Pino, NÃO lança exceção (log de auditoria não deve bloquear operação principal)
- [x] Criar `apps/api/src/common/modules/audit-log.module.ts`: exporta `AuditLogService`, importa `PrismaModule`

### 8. Registrar guards globalmente no AppModule

- [x] Em `app.module.ts`, registrar `JwtAuthGuard` e `RolesGuard` como providers com `APP_GUARD`:
  ```ts
  { provide: APP_GUARD, useClass: JwtAuthGuard },
  { provide: APP_GUARD, useClass: RolesGuard },
  ```
- [x] Criar decorator `@Public()` para marcar endpoints que não requerem JWT (ex: `GET /health`, `POST /api/v1/internal/interactions`)
- [x] Verificar que `GET /health` funciona sem token (decorator `@Public()` aplicado)
- [x] Verificar que qualquer endpoint sem `@Public()` retorna 401 se request não tiver `Authorization: Bearer <token>`

### 9. RBAC — mapeamento completo de operações (RF-026)

Implementar e documentar que as seguintes operações exigem role `ADMIN`:

- [x] `GET /api/v1/admin/interactions` — listar interações → `@Roles('ADMIN')`
- [x] `GET /api/v1/admin/interactions/:id` — detalhe de interação → `@Roles('ADMIN')`
- [x] `POST /api/v1/admin/interactions/:id/takeover` — iniciar takeover → `@Roles('ADMIN')`
- [x] `DELETE /api/v1/admin/interactions/:id/takeover` — encerrar takeover → `@Roles('ADMIN')`
- [x] `GET /api/v1/admin/metrics/dashboard` — métricas do dashboard → `@Roles('ADMIN')`
- [x] `GET /api/v1/admin/metrics/agents` — métricas por agente → `@Roles('ADMIN')`
- [x] `GET /api/v1/admin/agent-config` — listar configurações → `@Roles('ADMIN')`
- [x] `PATCH /api/v1/admin/agent-config/:id` — atualizar configuração → `@Roles('ADMIN')`
- [x] `POST /api/v1/admin/alerts/test` — testar alerta → `@Roles('ADMIN')`
- [x] Confirmar que `POST /api/v1/internal/interactions` usa `InternalApiKeyGuard` (não JWT) — marcado com `@Public()` para JwtAuthGuard, mas `@UseGuards(InternalApiKeyGuard)` explícito

### 10. Middleware de extração de IP e User-Agent

- [x] Criar `apps/api/src/common/middleware/request-context.middleware.ts`:
  - Extrai `ip` de `req.ip` ou `X-Forwarded-For` header (respeita proxy)
  - Extrai `userAgent` de `req.headers['user-agent']`
  - Popula `req.context = { ip, userAgent }` para uso no `AuditLogService`
- [x] Registrar middleware em `AppModule.configure()` para todas as rotas `/api/v1/admin/*`

---

## FRONTEND

> S2 não implementa telas de auth (auth é da plataforma central). Implementa apenas cliente HTTP autenticado e hook de sessão para uso nas sprints S3+.

### 11. Cliente HTTP autenticado (`apps/web/src/lib/api-client.ts`)

- [x] Implementar `createApiClient(token: string)` que retorna instância do cliente HTTP com:
  - `baseURL: process.env.NEXT_PUBLIC_API_URL`
  - Header padrão `Authorization: Bearer ${token}`
  - Header padrão `Content-Type: application/json`
  - Interceptor de resposta: se HTTP 401, redireciona para `/login` (sessão expirada)
  - Interceptor de resposta: se HTTP 403, exibe toast de erro `'Acesso não autorizado'`

### 12. Hook `useAdminSession` (`apps/web/src/hooks/use-admin-session.ts`)

- [x] Criar hook que:
  - Usa `@supabase/ssr` para obter sessão atual
  - Verifica que `session.user.role === 'ADMIN'` — se não for, redireciona para `/unauthorized`
  - Retorna `{ session, accessToken, isLoading, isAdmin }`
  - Access token tem duração de 15 minutos (D18); refresh automático via Supabase SDK

### 13. Middleware de proteção de rotas Next.js (`apps/web/src/middleware.ts`)

- [x] Criar `middleware.ts` que:
  - Protege todas as rotas `/admin/*` — verifica cookie de sessão Supabase
  - Se não autenticado, redireciona para `/login`
  - Se autenticado mas role ≠ `ADMIN`, redireciona para `/unauthorized`
  - Rota `/health` (do backend) não é afetada pelo middleware Next.js

---

## WIRING

### 14. Wiring de auth no AppModule

- [x] Verificar que `PassportModule.register({ defaultStrategy: 'jwt' })` está importado no `AppModule`
- [x] Verificar que `JwtModule.registerAsync({ ... })` está configurado com `SUPABASE_JWT_SECRET`
- [x] Verificar que `APP_GUARD` providers estão na ordem correta: `JwtAuthGuard` antes de `RolesGuard`
- [x] Verificar que `AuditLogModule` está importado no `AppModule`
- [x] Fazer request manual: `curl -H "Authorization: Bearer <token-admin-valido>" GET /api/v1/admin/interactions` → deve retornar 200 (ou 404 se ainda não implementado, mas não 401/403)
- [x] Fazer request manual: `curl GET /api/v1/admin/interactions` sem token → deve retornar 401 com `{ error: { code: 'DA-AUTH-001', message: 'Token inválido ou expirado' } }`
- [x] Fazer request manual: `curl -H "Authorization: Bearer <token-cessionario>" GET /api/v1/admin/interactions` → deve retornar 403 com `{ error: { code: 'DA-AUTH-002', message: 'Acesso restrito a administradores' } }`

---

## TESTES

### 15. Testes unitários — JwtStrategy

- [x] Criar `apps/api/src/common/strategies/jwt.strategy.spec.ts`:
  - Test: `validate(payload)` retorna `{ sub, email, role }` para payload válido
  - Test: `validate(payload)` NÃO faz query ao banco
  - Test: JWT expirado é rejeitado pelo PassportStrategy (mock de `passport-jwt`)

### 16. Testes unitários — JwtAuthGuard

- [x] Criar `apps/api/src/common/guards/jwt-auth.guard.spec.ts`:
  - Test: token válido → guard passa (`canActivate` retorna `true`)
  - Test: token inválido → lança `UnauthorizedException` com code `DA-AUTH-001`
  - Test: sem token no header → lança `UnauthorizedException` com code `DA-AUTH-001`

### 17. Testes unitários — RolesGuard

- [x] Criar `apps/api/src/common/guards/roles.guard.spec.ts`:
  - Test: user com role `ADMIN` acessando endpoint `@Roles('ADMIN')` → guard passa
  - Test: user com role `CESSIONARIO` acessando endpoint `@Roles('ADMIN')` → lança `ForbiddenException` com code `DA-AUTH-002`
  - Test: user com role `CEDENTE` acessando endpoint `@Roles('ADMIN')` → lança `ForbiddenException` com code `DA-AUTH-002`
  - Test: endpoint sem `@Roles()` decorator → guard passa (sem restrição de role)

### 18. Testes unitários — InternalApiKeyGuard

- [x] Criar `apps/api/src/common/guards/internal-api-key.guard.spec.ts`:
  - Test: header `X-Internal-Api-Key` com valor correto → guard passa
  - Test: header ausente → HTTP 401 com code `DA-AUTH-003`
  - Test: valor incorreto → HTTP 401 com code `DA-AUTH-003`
  - Test: comparação usa `timingSafeEqual` (não `===`)

### 19. Testes unitários — AuditLogService

- [x] Criar `apps/api/src/common/services/audit-log.service.spec.ts`:
  - Test: `log(dto)` cria registro em `admin_access_logs` com os campos corretos
  - Test: `log(dto)` nunca chama `update` ou `delete` — apenas `create`
  - Test: em falha de DB, NÃO lança exceção — apenas loga o erro via Pino

### 20. Testes E2E — RBAC

- [x] Criar `apps/api/test/auth.e2e-spec.ts`:
  - Test: `GET /api/v1/admin/interactions` sem token → 401, `{ error: { code: 'DA-AUTH-001' } }`
  - Test: `GET /api/v1/admin/interactions` com token de Cessionário → 403, `{ error: { code: 'DA-AUTH-002' } }`
  - Test: `GET /api/v1/admin/interactions` com token de Admin válido → 200 (ou 404, não 401/403)
  - Test: `POST /api/v1/internal/interactions` com `X-Internal-Api-Key` correto → sem 401 (resposta de negócio)
  - Test: `POST /api/v1/internal/interactions` sem `X-Internal-Api-Key` → 401, `{ error: { code: 'DA-AUTH-003' } }`
  - Test: `GET /health` sem token → 200

---

## AUTO-VERIFICAÇÃO S2

| Check                 | Critério                                                                                                              | Status |
| --------------------- | --------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura       | Guards usam nomes exatos: `JwtAuthGuard`, `RolesGuard`, `InternalApiKeyGuard`; erros usam codes `DA-AUTH-001/002/003` | ☑      |
| #2 Verificabilidade   | Cada item de teste é binariamente verificável                                                                         | ☑      |
| #3 Valores numéricos  | Access token 15min, refresh token 7 dias, session web 24h, session mobile 7 dias (D18)                                | ☑      |
| #4 N itens completos  | 9 operações do RBAC mapeadas (RF-026): todas listadas no item 9                                                       | ☑      |
| #5 Máquinas de estado | N/A em S2 — sem máquinas de estado novas                                                                              | ☑      |
| #6 Schedules/TTLs     | TTL de access token (15min) e refresh token (7 dias) documentados no hook                                             | ☑      |
| #7 Conflitos          | Nenhum conflito novo em S2                                                                                            | ☑      |
| #8 Ambiguidades       | RM-005 (endpoint `revoke` em D27 mas não em D16): endpoint NÃO implementado em S2 — marcado como ⚠️ AMBÍGUO           | ☑      |
| #9 Anti-scaffold      | JwtStrategy tem método `validate` com lógica real; RolesGuard compara role real do token                              | ☑      |
| #10 Cross-módulo      | `AuditLogService` exportado de `AuditLogModule` — será injetado em S3 (Takeover), S5 (Config), S8 (Launch)            | ☑      |
| #11 IDs de referência | D18 (auth), D05 RF-026 (RBAC), D16 (error codes), D20 (error handling) citados                                        | ☑      |
| #12 Cobertura REQs S2 | Todos os REQs de auth no registro-mestre.md têm ≥ 1 item no checklist                                                 | ☑      |

---

## ⚠️ Flags Pendentes

- **RM-005** `AMBÍGUO — endpoint revoke em D27 mas ausente de D16`: Não implementado em S2. Se confirmado necessário, adicionar em S2b (sub-sprint) ou S9.
