# S2 — Auth e Segurança

| Campo                | Valor                                                                                                                                                                                                                                            |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sprint**           | S2                                                                                                                                                                                                                                               |
| **Nome**             | Auth e Segurança                                                                                                                                                                                                                                 |
| **Tipo**             | Fixa de Infraestrutura                                                                                                                                                                                                                           |
| **Template**         | A (Infraestrutura)                                                                                                                                                                                                                               |
| **Docs Consultados** | D01, D02, D16, D17, D18                                                                                                                                                                                                                          |
| **Cross-cutting**    | D10 (Glossário)                                                                                                                                                                                                                                  |
| **REQs cobertos**    | REQ-010, REQ-018, REQ-022, REQ-081, REQ-084, REQ-095, REQ-110, REQ-111, REQ-112, REQ-113, REQ-118, REQ-156, REQ-157, REQ-164, REQ-165, REQ-167, REQ-169, REQ-174, REQ-175, REQ-176                                                               |
| **Objetivo**         | JwtAuthGuard funcional, CedenteIsolationMiddleware ativo, RbacGuard com roles cedente/admin, rate limiting 30 msg/h, PII masking, prompt injection protection, endpoints `GET /auth/me` e `POST /auth/validate` funcionando com testes completos |

---

## Critério de Conclusão da S2

Ao final desta sprint:

- `POST /auth/validate` com token JWT válido retorna `{ data: { valid: true, cedente_id: "...", expires_at: "..." } }`
- `GET /auth/me` com token válido retorna perfil do Cedente autenticado
- Token expirado retorna 401 com `DCE-AUTH-4010_001`
- Token ausente retorna 401 com `DCE-AUTH-4010_003`
- Token com role `cedente` em endpoint `/admin/*` retorna 403 com `DCE-AUTH-4030_001`
- Rate limit de 30 mensagens/hora por `cedente_id` ativo no Redis
- `cedente_id` é sempre extraído do JWT — nunca de `req.body`
- PII masking ativo: CPF, e-mail, telefone, Authorization nunca em logs

---

## ⚙️ BACKEND

### Módulo `auth`

- [x] **Implementar `JwtAuthGuard`** em `src/modules/auth/guards/jwt-auth.guard.ts`: (1) extrai Bearer token do header `Authorization`; (2) verifica assinatura RS256 localmente com `SUPABASE_JWT_SECRET` (sem request HTTP); (3) extrai `cedente_id` do claim `sub`; (4) extrai `role` do claim; (5) injeta `req.user = { cedenteId, role }` no context; (6) lança `DaniAuthError('DCE-AUTH-4010_001')` se expirado; `DaniAuthError('DCE-AUTH-4010_002')` se inválido/malformado; `DaniAuthError('DCE-AUTH-4010_003')` se ausente
  - Validação: guard aplicado a todos os endpoints de negócio; validação local sem chamada HTTP ao Supabase; testes unitários com 5 cenários (token válido, expirado, inválido, ausente, role incorreto)
- [x] **Implementar `CedenteIsolationMiddleware`** em `src/modules/auth/middleware/cedente-isolation.middleware.ts`: (1) aplica PII masking nos dados do request (cpf, email, telefone, Authorization) antes de qualquer log; (2) injeta `cedente_id` no AsyncLocalStorage para uso em qualquer ponto da cadeia sem passar via parâmetro; (3) rejeita requests onde `cedente_id` do JWT diverge do recurso solicitado
  - Validação: `@CurrentCedente()` decorator extrai o `cedente_id` correto em qualquer controller; testes verificam que Cedente A nunca acessa dados do Cedente B
- [x] **Implementar `RbacGuard`** em `src/modules/auth/guards/rbac.guard.ts`: (1) lê role esperado do metadata configurado via `@RequireRole()`; (2) compara com role do JWT; (3) lança `DaniForbiddenError('DCE-AUTH-4030_001')` se insuficiente
  - Validação: rotas `/admin/*` têm `@RequireRole('admin')` e retornam 403 para token `cedente`; testes com role válido e inválido
- [x] **Implementar `RateLimitGuard`** em `src/modules/auth/guards/rate-limit.guard.ts`: (1) incrementa contador `rate_limit:{cedente_id}` no Redis com janela deslizante de 1 hora; (2) se contador > 30: retorna HTTP 429 com `{ error: { code: "DCE-CHAT-4290_001", message: "...", details: { retry_after: N, limit: 30, window: "1h" } } }` e headers `X-RateLimit-Limit: 30`, `X-RateLimit-Remaining: N`, `X-RateLimit-Reset: epoch`; (3) campo de entrada desabilitado com contador regressivo no frontend (comportamento documentado — implementação frontend na S3)
  - Validação: após 30 requests no mesmo window de 1h para o mesmo `cedente_id`: 31º retorna 429; contador resets após expiração da janela Redis
- [x] **Implementar `PiiMaskingMiddleware`** global em `src/common/middleware/pii-masking.middleware.ts`: (1) substitui valores de campos `cpf`, `email`, `telefone`, `Authorization` por `[REDACTED]` em todos os logs; (2) aplicado antes do `LoggingInterceptor`; (3) nunca mascara os dados no request em si — apenas no log
  - Validação: testes verificam que log de request contendo CPF real emite log com `[REDACTED]`
- [x] **Implementar Prompt Injection Protection** em `src/common/middleware/prompt-sanitization.middleware.ts`: (1) sanitiza inputs do Cedente removendo padrões de injeção (`IGNORE PREVIOUS INSTRUCTIONS`, `</s>`, sequências de escape, etc.); (2) system prompt mantido em arquivo separado e imutável (`src/modules/agent/prompts/system-prompt.v1.0.ts`); (3) instruções do usuário inseridas em tag delimitada no prompt
  - Validação: testes com payloads de injeção conhecidos verificam sanitização; system prompt nunca modificável via input do usuário
- [x] **Implementar `PiiMaskingService`** com método `maskCessionarioData(context: LlmContext): LlmContext`: substitui `cessionario_nome` por `"[Comprador]"` e `cessionario_cpf` por `"[REDACTED]"` antes de qualquer inserção no contexto do LLM
  - Validação: testes unitários verificam mascaramento de todos os campos PII do Cessionário; nenhum dado pessoal do Cessionário enviado ao LLM

### Endpoints Auth

- [x] **Implementar `GET /api/v1/auth/me`** em `AuthController`: (1) extrai `cedente_id` via `@CurrentCedente()`; (2) busca `CedenteProfile` onde `id = cedente_id AND deleted_at IS NULL`; (3) retorna `{ data: { id, supabase_auth_id(=user_id), full_name, cpf_masked(apenas formato ***.456.789-00), kyc_status, kyc_approved_at(=kyc_verified_at), created_at } }`; (4) errors: 401 `DCE-AUTH-4010_001` (expirado), 401 `DCE-AUTH-4010_003` (ausente), 404 `DCE-AUTH-4040_001` (perfil não encontrado)
  - Validação: request com token válido e perfil existente → 200 com `cpf_masked` e nunca CPF completo; sem perfil → 404; sem token → 401
- [x] **Implementar `POST /api/v1/auth/validate`** em `AuthController`: (1) valida token JWT; (2) retorna `{ data: { valid: true, cedente_id: "...", expires_at: "..." } }`; (3) error: 401 `DCE-AUTH-4010_002` (token inválido/malformado)
  - Validação: token válido → 200 com `valid: true`; token com assinatura alterada → 401 `DCE-AUTH-4010_002`

### KYC — Verificação de Identidade

- [x] **Implementar lógica de status KYC no `CedenteProfile`**: (1) `PENDENTE` = cadastro não verificado; (2) `APROVADO` = verificado com sucesso (prazo automatizado ≤30 minutos); (3) `REPROVADO` = verificação falhou; (4) durante análise (qualquer status ≠ APROVADO): Cedente pode navegar mas fica bloqueado de publicar oportunidades (validação no endpoint de publicação na S4)
  - Validação: `GET /auth/me` retorna `kyc_status` correto; teste verifica que usuário PENDENTE não pode publicar oportunidade
- [x] **Criar `src/modules/auth/dto/kyc-status.dto.ts`** com enum `KycStatus` e resposta tipada para `GET /auth/me`
  - Validação: DTO alinhado com o model Prisma `CedenteProfile.kyc_status`

---

## 🧪 TESTES

- [x] **Criar `src/modules/auth/guards/jwt-auth.guard.spec.ts`** com testes unitários cobrindo: (1) token válido com role `cedente` → `canActivate` retorna `true`; (2) token expirado → lança `DaniAuthError(DCE-AUTH-4010_001)`; (3) token inválido/malformado → lança `DaniAuthError(DCE-AUTH-4010_002)`; (4) token ausente → lança `DaniAuthError(DCE-AUTH-4010_003)`; (5) token com role incorreto para rota admin → `RbacGuard` lança `DaniForbiddenError`
  - Validação: 5 cenários cobertos; mocks do JWT sem chamada real ao Supabase
- [x] **Criar `src/modules/auth/middleware/cedente-isolation.middleware.spec.ts`** com testes: (1) `cedente_id` injetado corretamente do JWT; (2) Cedente A não consegue acessar recurso do Cedente B (retorna 403); (3) `req.body.cedenteId` é ignorado — sempre usa JWT
  - Validação: teste de isolamento binário: Cedente A ≠ Cedente B sempre resulta em 403
- [x] **Criar `src/modules/auth/guards/rate-limit.guard.spec.ts`** com testes: (1) 30 requests no mesmo window → todos passam; (2) 31º request → retorna 429 com `retry_after`; (3) após expiração do window Redis → contador reseta e novo request passa
  - Validação: mock do Redis com contador; 3 cenários cobertos
- [x] **Criar teste de integração `test/auth.e2e-spec.ts`** com Supertest: (1) `GET /auth/me` com token válido → 200; (2) `GET /auth/me` sem token → 401; (3) `POST /auth/validate` com token válido → 200 `valid: true`; (4) endpoint `/admin/` com token `cedente` → 403
  - Validação: 4 cenários de integração com banco de teste real (testcontainers ou Supabase local)

---

## 🔀 Cross-Módulo

- A `CedenteIsolationMiddleware` é consumida por TODOS os módulos de negócio (S3–S8) — implementada aqui como ponto de corte global
- O `JwtAuthGuard` é aplicado globalmente via `APP_GUARD` no `app.module.ts` — habilitado aqui, usado em todas as sprints seguintes
- Rate limiting do Redis (`rate_limit:{cedente_id}`) é compartilhado entre `chat` e `notifications` na S8

---

## 🔍 AUTO-VERIFICAÇÃO S2

| Check                | Critério                                                                                                                                                                | Status |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura      | `JwtAuthGuard`, `CedenteIsolationMiddleware`, `RbacGuard`, `RateLimitGuard` — nomes exatos conforme D18                                                                 | [x]    |
| #2 Valores numéricos | Rate limit: 30 msg/h; janela: 1 hora; access token: 3600s; refresh token: 604800s; headers X-RateLimit corretos                                                         | [x]    |
| #3 Error codes       | DCE-AUTH-4010_001 (expirado), DCE-AUTH-4010_002 (inválido), DCE-AUTH-4010_003 (ausente), DCE-AUTH-4030_001 (role), DCE-CHAT-4290_001 (rate limit) — todos implementados | [x]    |
| #4 Isolamento        | `cedente_id` exclusivamente do JWT claim; nunca de `req.body`; AsyncLocalStorage para propagação                                                                        | [x]    |
| #5 PII               | CPF, e-mail, telefone, Authorization → `[REDACTED]` em logs; cessionario_nome → `[Comprador]` no LLM                                                                    | [x]    |
| #6 Roles             | 2 roles: `cedente` e `admin`; sem hierarquia complexa; RLS ativa no banco                                                                                               | [x]    |
| #7 RS256             | Validação local do JWT (sem request HTTP); chave pública via `SUPABASE_JWT_SECRET`                                                                                      | [x]    |
| #8 Anti-scaffold     | Guards, middleware e serviços com lógica real — não apenas stubs; PII masking testado                                                                                   | [x]    |
| #9 KYC               | 3 status (PENDENTE, APROVADO, REPROVADO); prazo automatizado ≤30min; bloqueio de publicação se não APROVADO                                                             | [x]    |
| #10 Glossário        | `cpf_masked` no response (não CPF completo); `user_id` = `supabase_auth_id`; `confidence_score` tracking previsto                                                       | [x]    |
| #11 Cobertura REQs   | REQ-010, REQ-018, REQ-022, REQ-081, REQ-084, REQ-095, REQ-110–113, REQ-118, REQ-156–157, REQ-164–165, REQ-167, REQ-169, REQ-174–176 cobertos                            | [x]    |
| #12 Testes           | Testes unitários de guards (5 cenários JWT, 3 rate limit, 2 isolamento) + E2E de auth passando                                                                          | [x]    |
