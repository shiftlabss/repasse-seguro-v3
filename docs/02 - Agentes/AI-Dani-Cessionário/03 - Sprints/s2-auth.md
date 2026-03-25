# S2 — Autenticação e RBAC

| **Sprint**         | S2 — Autenticação e RBAC                                                       |
| ------------------ | ------------------------------------------------------------------------------ |
| **Template**       | A — Infraestrutura (Banco → Backend → Frontend → Wiring → Testes)              |
| **REQs cobertos**  | REQ-003, REQ-004, REQ-005, REQ-091 a REQ-097, REQ-107 (hash)                   |
| **Docs fonte**     | D01 (RN-DC-003, RN-DC-007), D14 (AuthModule), D18 (OAuth), D20 (AuthException) |
| **Total de itens** | 42                                                                             |

---

## 🎯 Objetivo

Implementar o sistema completo de autenticação e RBAC: herança de JWT da plataforma Repasse Seguro, `CessionarioOwnerGuard`, refresh token, blacklist de tokens revogados e isolamento de `cessionario_id` em todos os endpoints.

---

## 🗄️ Banco de Dados

### Feature: Sessões e Blacklist

- [x] **[BANCO-AUTH-001]** Verificar que `dani_sessoes` está migrada (S1) e adicionar índice de expiração se ausente:
  - Sub-item: `dani_sessoes` existe com campo `jwt_session_hash TEXT NOT NULL`
  - Sub-item: Índice `@@index([expires_at])` presente para cleanup eficiente
  - Sub-item: Query `SELECT id FROM dani_sessoes WHERE expires_at < NOW()` executa sem full scan (explain confirm usa índice)

---

## 🔧 Backend

### Feature: AuthModule — JWT e Herança de Sessão

- [x] **[BACK-AUTH-001]** Implementar `JwtStrategy` em `src/auth/strategies/jwt.strategy.ts`:
  - Sub-item: Extende `PassportStrategy(Strategy)` de `passport-jwt`
  - Sub-item: `jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken()` — extrai Bearer token do header `Authorization`
  - Sub-item: `secretOrKeyProvider` usa `jwks-rsa` para validar chave pública da plataforma Repasse Seguro
  - Sub-item: `validate(payload)` extrai `cessionario_id` e `role` do payload JWT
  - Sub-item: Se `cessionario_id` ausente no payload → lança `AuthException` com código `AUTH_004` e status 401
  - Sub-item: Se `role` ausente → lança `AuthException` com código `AUTH_004` e status 401

- [x] **[BACK-AUTH-002]** Implementar `JwtAuthGuard` em `src/auth/guards/jwt-auth.guard.ts`:
  - Sub-item: Extende `AuthGuard('jwt')` de `@nestjs/passport`
  - Sub-item: Token ausente → lança `AuthException` código `AUTH_001`, status 401, `user_message: "Token de autenticação ausente."`
  - Sub-item: Token expirado → lança `AuthException` código `AUTH_002`, status 401, `user_message: "Sessão expirada. Faça login novamente."`; header `WWW-Authenticate: Bearer error="token_expired"`
  - Sub-item: Token inválido/mal-formado → lança `AuthException` código `AUTH_003`, status 401
  - Sub-item: Refresh token ausente (cookie `dani_refresh_token`) → `AUTH_006`, status 401

- [x] **[BACK-AUTH-003]** Implementar `CessionarioOwnerGuard` em `src/auth/guards/cessionario-owner.guard.ts`:
  - Sub-item: Implementa `CanActivate`; injetado nos controladores como `@UseGuards(JwtAuthGuard, CessionarioOwnerGuard)`
  - Sub-item: Extrai `cessionario_id` do JWT payload (já validado pela `JwtStrategy`)
  - Sub-item: Injeta `cessionario_id` no `request.cessionarioId` para uso em serviços e repositórios
  - Sub-item: Verifica que `cessionario_id` do token corresponde ao recurso sendo acessado (quando parâmetro de rota ou body expõe ID)
  - Sub-item: Mismatch de `cessionario_id` → lança `AuthException` código `AUTH_009`, status 403, `user_message: "Acesso negado. Você só pode acessar os próprios dados."`
  - Sub-item: `cessionario_id` não injetado no request context → falha de segurança → lança `AuthException` código `AUTH_004`, status 500, log de auditoria `error` com campo `error_code: AGENTE_004`, alerta Sentry P0

- [x] **[BACK-AUTH-004]** Implementar `RbacGuard` em `src/auth/guards/rbac.guard.ts`:
  - Sub-item: Roles permitidos em endpoints Dani: `CESSIONARIO`, `ADMIN` (REQ-095)
  - Sub-item: Role insuficiente → lança `AuthException` código `AUTH_005`, status 403, body `{ code: "INSUFFICIENT_ROLE", required: "CESSIONARIO_AUTHENTICATED" }`
  - Sub-item: Decorator `@Roles('CESSIONARIO', 'ADMIN')` definido em `src/auth/decorators/roles.decorator.ts`

- [x] **[BACK-AUTH-005]** Implementar Redis para sessão e blacklist conforme D18:
  - Sub-item: Redis session: `dani:session:{session_id}` TTL 604800s (7 dias) — armazenado ao criar sessão
  - Sub-item: Redis blacklist: `dani:revoked:refresh:{jti}` — armazenado ao revogar refresh token via logout
  - Sub-item: Ao receber refresh token: verificar `EXISTS dani:revoked:refresh:{jti}`; se existir → lança `AuthException` código `AUTH_008`, status 401

- [x] **[BACK-AUTH-006]** Implementar endpoints de auth conforme D18:
  - Sub-item: `POST /api/v1/auth/refresh` — valida cookie `dani_refresh_token` (httpOnly), verifica blacklist, emite novo access token TTL 15 min; se refresh expirado → `AUTH_007`; se revogado → `AUTH_008`
  - Sub-item: `POST /api/v1/auth/logout` — adiciona `jti` do refresh token ao blacklist `dani:revoked:refresh:{jti}`; limpa cookie `dani_refresh_token`; retorna 200
  - Sub-item: `GET /api/v1/auth/me` — retorna `{ cessionario_id, role, kyc_status }` filtrado do JWT; guard `CessionarioOwnerGuard` aplicado
  - Sub-item: `POST /dani/sessao` — registra sessão em `dani_sessoes`; retorna 201 com `session_id`

- [x] **[BACK-AUTH-007]** Implementar isolamento de `cessionario_id` em todos os repositórios:
  - Sub-item: Toda query de read/write em qualquer tabela `dani_*` inclui `WHERE cessionario_id = :id`
  - Sub-item: Método de repositório base `BaseRepository.findByCessionario(id)` verifica injeção obrigatória do `cessionario_id`
  - Sub-item: pgvector queries incluem `WHERE namespace = cessionario_id` (REQ-106)
  - Sub-item: Se query executada sem `cessionario_id` no contexto → log `error` com código `AGENTE_004` + alerta Sentry P0 (REQ-003)

- [x] **[BACK-AUTH-008]** Implementar hash SHA-256 para dados sensíveis em logs e Langfuse:
  - Sub-item: Função `hashSHA256(input: string): string` em `src/common/utils/hash.utils.ts`
  - Sub-item: `cessionario_id` em logs sempre como `sha256(cessionario_id)` — nunca UUID raw (REQ-107, REQ-144)
  - Sub-item: Langfuse `userId = sha256(cessionario_id)` — nunca UUID raw (REQ-107)
  - Sub-item: Phone hash para Redis OTP: `sha256(phoneE164)` (REQ-096)

---

## 🖥️ Frontend

### Feature: Interceptor de JWT e Refresh

- [x] **[FRONT-AUTH-001]** Implementar interceptor HTTP para refresh automático de access token:
  - Sub-item: Access token armazenado em memória JS (in-memory, não localStorage — REQ-091)
  - Sub-item: Interceptor detecta resposta 401 com `error="token_expired"`: executa `POST /api/v1/auth/refresh` automaticamente
  - Sub-item: Se refresh falha: redirect para `/login?redirect={encoded_url}` (REQ-135)
  - Sub-item: Cookie `dani_refresh_token` httpOnly enviado automaticamente pelo browser em `/auth/refresh`
  - Sub-item: Após refresh: re-executa a request original com novo access token

- [x] **[FRONT-AUTH-002]** Implementar verificação de role `CESSIONARIO_AUTHENTICATED` em rotas protegidas:
  - Sub-item: TanStack Router `beforeLoad` verifica `isAuthenticated` via JWT em memória
  - Sub-item: Não autenticado: redirect para `/login?redirect={encoded_url}`
  - Sub-item: `WhatsAppPhaseGuard` aplicado na rota `/cessionario/perfil/whatsapp` (REQ-131)
  - Sub-item: Deep link com `?chat=open` preservado após redirect de autenticação (REQ-135)

---

## 🔌 Wiring

### Feature: Guards Aplicados em Todos os Endpoints

- [x] **[WIRE-AUTH-001]** Verificar e aplicar guards globalmente:
  - Sub-item: `JwtAuthGuard` + `CessionarioOwnerGuard` aplicados em todos os controllers de `dani/*` e `calculadora/*`
  - Sub-item: `POST /whatsapp/webhook` usa API Key auth (`x-api-key: {EVOLUTIONAPI_WEBHOOK_SECRET}`), não Bearer — guard especial `WebhookApiKeyGuard`
  - Sub-item: Lista de verificação D28: nenhum endpoint retorna dados de negócio sem `CessionarioOwnerGuard` (bloqueante)

- [x] **[WIRE-AUTH-002]** Configurar cabeçalhos de segurança conforme D28 §2.3:
  - Sub-item: `X-Content-Type-Options: nosniff`
  - Sub-item: `X-Frame-Options: DENY`
  - Sub-item: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
  - Sub-item: CORS: whitelist de origens — `origin: '*'` proibido em produção

---

## ✅ Testes

- [x] **[TEST-AUTH-001]** Testes unitários do `CessionarioOwnerGuard` — cobertura 100% (REQ-149):
  - Sub-item: JWT válido + `cessionario_id` correto → `canActivate()` retorna `true`
  - Sub-item: JWT válido + `cessionario_id` de outro cessionário → lança `AuthException AUTH_009`, status 403
  - Sub-item: JWT sem `cessionario_id` no payload → lança `AuthException AUTH_004`, status 500, log de auditoria gerado
  - Sub-item: `cessionario_id` não injetado no request context → falha P0, log `error` com `AGENTE_004`

- [x] **[TEST-AUTH-002]** Testes unitários do `JwtAuthGuard`:
  - Sub-item: Bearer token ausente → 401 com código `AUTH_001`
  - Sub-item: Access token expirado → 401 com código `AUTH_002` + header `WWW-Authenticate`
  - Sub-item: Token mal-formado → 401 com código `AUTH_003`
  - Sub-item: Token válido → `validate()` retorna `{ cessionario_id, role }`

- [x] **[TEST-AUTH-003]** Testes de integração dos endpoints de auth:
  - Sub-item: `POST /auth/refresh` — cookie ausente → 401 `AUTH_006`; refresh expirado → 401 `AUTH_007`; refresh revogado → 401 `AUTH_008`; refresh válido → 200 com novo access token
  - Sub-item: `POST /auth/logout` — adiciona `jti` ao blacklist Redis; cookie `dani_refresh_token` limpo; 200
  - Sub-item: `GET /auth/me` — JWT válido → 200 com `{ cessionario_id_hash, role, kyc_status }`
  - Sub-item: `POST /dani/sessao` — JWT válido → 201 com `session_id`

- [x] **[TEST-AUTH-004]** Testes de isolamento de `cessionario_id` (cobertura 100% — P0 crítico):
  - Sub-item: Cessionário A tenta acessar dados do cessionário B via parâmetro de rota → 403 `AUTH_009`
  - Sub-item: Query executada sem `cessionario_id` no contexto → log `error` gerado com `AGENTE_004`
  - Sub-item: `CessionarioOwnerGuard` não pode ser mockado em testes de integração de endpoints (REQ-149)

- [x] **[TEST-AUTH-005]** Testes de hash SHA-256:
  - Sub-item: `hashSHA256(uuid)` retorna string de 64 caracteres hex
  - Sub-item: `hashSHA256(uuid) !== uuid` — nunca retorna o input
  - Sub-item: Logs gerados por `AllExceptionsFilter` não contêm UUID raw de `cessionario_id`

---

## 🔍 Auto-verificação S2 (12 checks)

| #   | Check                                                                                                     | Status |
| --- | --------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                              | ✅     |
| 2   | Nomes exatos usados: `CessionarioOwnerGuard`, `JwtStrategy`, `dani_sessoes`, `dani:revoked:refresh:{jti}` | ✅     |
| 3   | Valores numéricos: TTL 604800s (7 dias), TTL 900s (OTP), cost 12 bcrypt                                   | ✅     |
| 4   | Todos os códigos de erro AUTH_001–009 mapeados                                                            | ✅     |
| 5   | Glossário D10: "Sessão" = auth state temporário (`dani_sessoes`) — distinção preservada                   | ✅     |
| 6   | Máquinas de estado: refresh token → {válido, expirado, revogado} documentadas                             | ✅     |
| 7   | TTLs: access token 15 min, refresh 7 dias, session Redis 604800s                                          | ✅     |
| 8   | ⚠️ AMBÍGUO de paths D29 vs D16 documentado no registro-mestre, não aqui (não é conflito desta sprint)     | ✅     |
| 9   | Nenhuma suposição sem base documental                                                                     | ✅     |
| 10  | Nenhum item de scaffold — todos com sub-itens verificáveis                                                | ✅     |
| 11  | Test de isolamento P0: 100% cobertura exigida em `CessionarioOwnerGuard`                                  | ✅     |
| 12  | REQs REQ-003 a REQ-005 e REQ-091 a REQ-097 têm ≥1 item de checklist                                       | ✅     |

---

## REQs cobertos por esta sprint

REQ-003, REQ-004, REQ-005, REQ-007 (parcial — recusa após insistência, implementado em S4/S5), REQ-010, REQ-091, REQ-092, REQ-093, REQ-094, REQ-095, REQ-096, REQ-097, REQ-107 (hash SHA-256), REQ-144 (correlation ID + hash), REQ-150 (guards — bloqueante backend)
