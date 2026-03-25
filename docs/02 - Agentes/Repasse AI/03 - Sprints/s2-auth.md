# S2 — Auth e Sessão

<!-- Atualizado em 2026-03-24 pela A03 — 5 correções aplicadas (FINDING-004, FINDING-006, FINDING-015, FINDING-017, FINDING-023) -->

| Campo              | Valor                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------- |
| **Sprint**         | S2                                                                                                |
| **Nome**           | Auth e Sessão                                                                                     |
| **Template**       | A — Infraestrutura                                                                                |
| **Docs Fonte**     | 01.1-RN-Fundação, 01.4-RN-Admin, 01.5-RN-LGPD, 14-EspecificaçõesTécnicas, 16-API, 22-Ambiente     |
| **REQs cobertos**  | REQ-004, REQ-005, REQ-006, REQ-022, REQ-023, REQ-028, REQ-071, REQ-072, REQ-094, REQ-111, REQ-135 |
| **Total de itens** | 38                                                                                                |
| **Status**         | ✅ Concluída                                                                                      |

---

## 🎯 Objetivo

Implementar os mecanismos de autenticação (JWT validado externamente), RBAC (3 roles: CESSIONARIO, ADMIN, CEDENTE), isolamento de dados em 3 camadas, criação e gerenciamento de sessão de chat, rate limiting por canal, e consentimento LGPD. Nenhum login próprio — o Repasse AI valida JWT emitido pela plataforma principal.

---

## 📦 BANCO DE DADOS — Auth e Sessão

### 1. Constraint de escopo no `PrismaService` (RLS por software)

- [x] Implementar método `withScope(role: Role, userId: string, organizationId: string)` no `PrismaService` que retorna uma instância Prisma com filtros de escopo automaticamente aplicados
- [x] **Camada 1 — Scope filter:** `CESSIONARIO` → ver apenas seus próprios dados (`cessionario_id = userId`); `ADMIN` → ver todos os dados da organização (`organization_id = organizationId`); `CEDENTE` → acesso negado (sem scope válido para Repasse AI)
- [x] **Camada 2 — Context filter:** queries de `chat_conversations` e `chat_messages` adicionam automaticamente `AND deleted_at IS NULL`
- [x] **Camada 3 — Agent instruction:** não persiste no banco, é injetado no system prompt do agente (implementado em S3)
- [x] Teste: `PrismaService.withScope('CESSIONARIO', userId1)` → query retorna apenas registros onde `cessionario_id = userId1`; `PrismaService.withScope('CEDENTE', ...)` → lançar `ForbiddenException("CEDENTE não possui acesso ao Repasse AI")`

### 2. Tabela `lgpd_consents` — INSERT de consentimento inicial

- [x] Criar método `LgpdService.recordConsent(cessionarioId, consentType, granted, ipAddress, userAgent)` — INSERT imutável em `lgpd_consents`
- [x] Criar método `LgpdService.hasValidConsent(cessionarioId, consentType)` — SELECT mais recente por tipo; retornar `granted` mais recente
- [x] **Retenção LGPD 90 dias:** criar job (cron) `LgpdRetentionJob` — executar diariamente, soft delete em `chat_conversations` onde `created_at < NOW() - INTERVAL '90 days'` E `deleted_at IS NULL`
- [x] Validar que `lgpd_consents` é append-only: `UPDATE` e `DELETE` bloqueados pelo middleware do `PrismaService` (herdado de S1)
- [x] Teste unitário: `hasValidConsent(userId, 'data_processing')` sem registros → retornar `false`; com registro `granted=true` → retornar `true`; com registro mais recente `granted=false` → retornar `false`

---

## 🏗️ BACKEND — Auth, RBAC e Sessão

### 3. Estratégia JWT completa (`JwtStrategy`)

- [x] Completar `JwtStrategy` iniciada em S1: validar assinatura com `JWT_SECRET`, verificar `exp` (expiração), extrair payload `{ sub, role, organizationId, email }`
- [x] Criar interface `AuthenticatedUser` com campos: `userId: string`, `role: 'CESSIONARIO' | 'ADMIN' | 'CEDENTE'`, `organizationId: string`, `email: string`
- [x] Injetar `AuthenticatedUser` via decorator `@CurrentUser()` em qualquer controller
- [x] Validar que `role === 'CEDENTE'` → `403 Forbidden` com mensagem `"CEDENTE não possui acesso ao Repasse AI"` — retornar antes de qualquer processamento de negócio
- [x] **Verificação de sessão ativa (FINDING-015):** autenticação por herança de sessão ocorre via validação do JWT no `JwtAuthGuard` — se o token JWT da plataforma principal está presente e válido (`exp > now`), a sessão é considerada ativa. Não há endpoint de verificação de sessão próprio no Repasse AI. Se token expirado (`exp < now`): `JwtAuthGuard` lança `401 Unauthorized` → frontend exibe banner "Sessão expirada" + botão "Fazer login" + desabilita campo de input (REQ-023). Body de erro: `{ code: 'SESSION_EXPIRED', message: 'Sua sessão expirou. Faça login novamente.' }` — [CORRIGIDO: FINDING-015]
- [x] Teste unitário: token com `role: 'CEDENTE'` → `JwtStrategy.validate()` lança `ForbiddenException`; token expirado → `UnauthorizedException` com `{ code: 'SESSION_EXPIRED' }`; token com assinatura inválida → `UnauthorizedException`; token válido com role `CESSIONARIO` → retorna `AuthenticatedUser`

### 4. `RolesGuard` com decorator `@Roles()`

- [x] Implementar `RolesGuard` que lê metadata via `Reflector` e compara `user.role` contra roles declaradas em `@Roles()`
- [x] Hierarquia: `ADMIN` > `CESSIONARIO` — ADMIN pode acessar endpoints `CESSIONARIO` (mas CESSIONARIO não pode acessar endpoints `ADMIN`)
- [x] `CEDENTE` — nenhum acesso válido em Repasse AI; qualquer endpoint → `403 Forbidden`
- [x] Registrar `RolesGuard` globalmente após `JwtAuthGuard` na ordem de execução
- [x] Teste: endpoint `@Roles('ADMIN')` com token `CESSIONARIO` → `403`; endpoint `@Roles('CESSIONARIO')` com token `ADMIN` → `200`; endpoint `@Roles('CESSIONARIO')` com token `CEDENTE` → `403`

### 5. `ChatModule` — endpoint `POST /chat/conversations` (criar sessão)

- [x] Criar `ChatController` com endpoint `POST /chat/conversations` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `CreateConversationDto`: `{ title?: string, context?: Record<string, unknown> }` — validados com `class-validator`
- [x] `ChatService.createConversation(user: AuthenticatedUser, dto)`:
  1. Verificar `LgpdService.hasValidConsent(user.userId, 'ai_analysis')` → se `false`, lançar `ForbiddenException("Consentimento LGPD necessário para usar o Repasse AI")`
  2. INSERT em `chat_conversations` com `cessionario_id = user.userId`, `status = 'active'`
  3. Retornar `ConversationResponseDto`
- [x] `ConversationResponseDto`: `{ id, cessionarioId, title, status, context, createdAt }`
- [x] Resposta HTTP: `201 Created` com body `ConversationResponseDto`
- [x] Teste de integração: `POST /chat/conversations` com token `CESSIONARIO` válido + consentimento → `201`; sem consentimento LGPD → `403`; token `CEDENTE` → `403`; sem token → `401`

### 6. `ChatModule` — endpoint `GET /chat/conversations` (listar sessões)

- [x] Criar endpoint `GET /chat/conversations` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] Query params: `page` (default 1), `per_page` (default 20, max 100), `status` (optional: `active | archived | deleted`)
- [x] `ChatService.listConversations(user, page, per_page, status)`:
  1. Usar `PrismaService.withScope(user.role, user.userId)` para filtrar automaticamente
  2. Aplicar `status` filter se fornecido
  3. Retornar paginação: `{ data: ConversationResponseDto[], meta: { page, per_page, total, total_pages } }`
- [x] `CESSIONARIO` → vê apenas suas próprias conversas; `ADMIN` → vê todas (sem filtro por `cessionario_id`)
- [x] Teste de integração: `CESSIONARIO` lista conversas → retorna apenas as suas; `ADMIN` lista → retorna todas; `per_page=0` → `400 Bad Request`; `per_page=101` → `400 Bad Request`

### 7. `ChatModule` — endpoint `GET /chat/conversations/:id`

- [x] Criar endpoint `GET /chat/conversations/:id` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] `ChatService.getConversation(user, id)`:
  1. SELECT com scope filter
  2. Se não encontrado (ou `deleted_at` preenchido, ou pertence a outro usuário) → `404 Not Found` com mensagem `"Conversa não encontrada"`
  3. Retornar `ConversationResponseDto` com metadados da conversa (sem mensagens — usar endpoint de histórico)
- [x] **Endpoint de histórico de mensagens (FINDING-017):** `GET /chat/conversations/:id/messages` — retorna mensagens paginadas da conversa (últimas 20, `page` + `per_page`). Nomenclatura consolidada com Registro Mestre (seção 2.4). Endpoint SSE de envio de mensagem: `POST /chat/conversations/:id/messages` com header `Accept: text/event-stream` para streaming (implementado em S3b mas `ChatController` declara a rota). — [CORRIGIDO: FINDING-017]
- [x] Teste de integração: `GET /chat/conversations/:id` de outro cessionário → `404` (não `403` — não revelar existência); GET conversa soft-deleted → `404`; GET conversa válida com token correto → `200`; `GET /chat/conversations/:id/messages` → retorna array de mensagens paginado

### 8. `ChatModule` — endpoint `PATCH /chat/conversations/:id` (arquivar/reativar)

- [x] Criar endpoint `PATCH /chat/conversations/:id` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] DTO `UpdateConversationDto`: `{ status: 'active' | 'archived' }` — apenas estas transições via PATCH
- [x] Máquina de estado `conversation_status` — transições válidas: [CORRIGIDO: FINDING-023 — ⚠️ CONFLITO: Registro Mestre seção 2.7 usa `active → closed`, mas Doc 12 schema Prisma define enum com valores `active`, `archived`, `deleted`. Adotar Doc 12 como fonte normativa — `archived` é o equivalente de `closed` na terminologia de negócio (RN-009).]
  - `active` → `archived` ✅ (equivale a "fechar" a conversa — negócio chama de "closed")
  - `archived` → `active` ✅ (reativação)
  - `active` → `deleted` ❌ (apenas via DELETE endpoint)
  - `deleted` → qualquer → `409 Conflict` com mensagem `"Conversa deletada não pode ser modificada"`
- [x] Transição inválida (e.g., `archived` → `deleted` via PATCH) → `422 Unprocessable Entity`
- [x] Teste: `active → archived` → `200`; `archived → active` → `200`; `deleted → active` → `409`; `active → deleted` via PATCH → `422`

### 9. `ChatModule` — endpoint `DELETE /chat/conversations/:id` (soft delete)

- [x] Criar endpoint `DELETE /chat/conversations/:id` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] `ChatService.deleteConversation(user, id)`:
  1. Verificar escopo (pertence ao usuário ou ADMIN)
  2. Soft delete: UPDATE `deleted_at = now()`, `status = 'deleted'`
  3. **Não deletar fisicamente** — registro permanece no banco
  4. Retornar `204 No Content`
- [x] Transição de estado: qualquer status → `deleted` (via DELETE endpoint)
- [x] Teste: DELETE conversa própria (CESSIONARIO) → `204`; DELETE conversa de outro usuário (CESSIONARIO) → `404`; DELETE conversa inexistente → `404`; DELETE já deletada → `404` (soft-deleted não visível)

### 10. Rate Limiting por canal

- [x] Instalar e configurar `@nestjs/throttler` com armazenamento Redis (`ThrottlerStorageRedisService`)
- [x] Criar `RateLimitGuard` customizado que diferencia por canal:
  - `webchat`: 30 mensagens/hora por `userId` (janela deslizante — sliding window)
  - `whatsapp`: 20 mensagens/hora por `userId` (janela deslizante)
- [x] Identificar canal pelo header `X-Channel: webchat | whatsapp` ou por endpoint (endpoints `/whatsapp/` → rate limit WhatsApp)
- [x] Resposta ao exceder limite: `429 Too Many Requests` com headers `Retry-After` (segundos até reset) e `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- [x] Teste unitário: 30 msgs webchat em 1h → 31ª msg → `429`; 20 msgs WhatsApp → 21ª → `429`; após janela deslizante → contador reset; ADMIN não sujeito a rate limit por mensagem

### 11. `LgpdModule` — endpoint `POST /lgpd/consent` (registrar consentimento)

- [x] Criar `LgpdController` com endpoint `POST /lgpd/consent` (`@Roles('CESSIONARIO')`)
- [x] DTO `RecordConsentDto`: `{ consentType: LgpdConsentType, granted: boolean }` — `consentType` deve ser enum válido
- [x] `LgpdService.recordConsent`: INSERT em `lgpd_consents` com `ip_address` do request e `user_agent` do header
- [x] Resposta: `201 Created` com `{ id, cessionarioId, consentType, granted, createdAt }`
- [x] Teste: POST com `consentType` inválido → `400`; POST com `granted: false` → INSERT com `granted=false` (revogação válida); verificar que INSERT nunca faz UPDATE

### 12. `LgpdModule` — endpoint `GET /lgpd/consent` (verificar consentimentos)

- [x] Criar endpoint `GET /lgpd/consent` (`@Roles('CESSIONARIO', 'ADMIN')`)
- [x] Retornar estado atual de cada tipo de consentimento: `{ data_processing: boolean, ai_analysis: boolean, whatsapp_contact: boolean, marketing: boolean }`
- [x] Lógica: para cada tipo, buscar o registro mais recente e retornar seu `granted`; se não existe registro → `false`
- [x] Teste: sem registros → todos `false`; com `data_processing=true` e `ai_analysis=false` → refletir exatamente; consulta de outro usuário (CESSIONARIO) → `404` (não vê dados de terceiros)

### 13. Endpoint `DELETE /lgpd/data` (exclusão de dados LGPD)

- [x] Criar endpoint `DELETE /lgpd/data` (`@Roles('CESSIONARIO')`)
- [x] `LgpdService.deleteUserData(userId, reason?: string)`:
  1. Soft delete em `chat_conversations` (e cascade para `chat_messages`, `ai_interactions`) onde `cessionario_id = userId`
  2. Soft delete em `whatsapp_bindings` onde `cessionario_id = userId`
  3. INSERT em `lgpd_consents` com `consentType = 'data_processing', granted = false` (revogação)
  4. **Não deletar fisicamente** — manter para auditoria LGPD
  5. **SLA 48h — encerramento de conta (FINDING-004):** se `reason === 'account_closure'` → registrar `deadline: NOW() + INTERVAL '48 hours'` no `lgpd_consents.metadata` (JSONB). O job `LgpdRetentionJob` (cron diário) verifica registros com `metadata->>'account_closure' IS NOT NULL AND deadline < NOW()` e completa anonimização. Teste: INSERT com `reason='account_closure'` → metadata contém `deadline` futuro de 48h — [CORRIGIDO: FINDING-004]
  6. Retornar `202 Accepted` com `{ message: "Solicitação de exclusão registrada. Dados serão anonimizados em até 30 dias." }`
- [x] **Idempotência (FINDING-006):** se `SELECT COUNT(*) WHERE cessionario_id = userId AND deleted_at IS NULL` retornar `0` → retornar `204 No Content` (sem erro — idempotente). Teste: `DELETE data` sem dados pendentes → `204` — [CORRIGIDO: FINDING-006]
- [x] Teste: DELETE data com dados → `202`; verificar que conversas aparecem como soft-deleted; não apagar `lgpd_consents` histórico; DELETE data sem dados → `204`

---

## ✅ TESTES — Auth e Sessão

### 14. Suite de testes de Auth

- [x] Criar `test/unit/common/strategies/jwt.strategy.spec.ts` — cobrir: token válido, token expirado, token inválido, role CEDENTE
- [x] Criar `test/unit/common/guards/roles.guard.spec.ts` — cobrir: hierarquia ADMIN > CESSIONARIO, CEDENTE bloqueado
- [x] Criar `test/integration/chat/conversations.spec.ts` — cobrir todos os endpoints de `ChatController` (itens 5-9)
- [x] Criar `test/integration/lgpd/lgpd.spec.ts` — cobrir consentimento e exclusão de dados
- [x] Criar `test/unit/common/guards/rate-limit.guard.spec.ts` — cobrir limites 30/h webchat e 20/h WhatsApp com janela deslizante
- [x] Cobertura mínima do módulo `common/`: 90% (conforme Doc 27); módulo `chat/`: 80%

---

## 🔀 Cross-Módulo

- [x] **→ S3a (Agente Core):** `JwtStrategy`, `AuthenticatedUser`, `@CurrentUser()`, `@Roles()`, `PrismaService.withScope()` e `LgpdService.hasValidConsent()` são dependências diretas do módulo de agente — confirmar exports do `AuthModule` e `LgpdModule` para uso em S3

---

## ✔️ Auto-Verificação S2 (12 Checks)

| #   | Check                                                                                                                                 | Status |
| --- | ------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                          | ✅     |
| 2   | Nomes exatos: `chat_conversations`, `chat_messages`, `lgpd_consents`, `AuthenticatedUser`, `ConversationStatus`, `LgpdConsentType`    | ✅     |
| 3   | Valores numéricos: 30 msg/h webchat, 20 msg/h WhatsApp, 90 dias retenção, 20 mensagens histórico, 3 roles (CESSIONARIO/ADMIN/CEDENTE) | ✅     |
| 4   | Glossário consultado — RBAC, Takeover, LGPD, Score de Risco, Cedente, Cessionário                                                     | ✅     |
| 5   | Anti-Scaffold R10: cada item tem sub-itens com lógica, validações, errors e testes                                                    | ✅     |
| 6   | Máquina de estado `conversation_status`: `active↔archived`, `*→deleted` (somente via DELETE); `deleted` bloqueado para PATCH          | ✅     |
| 7   | Rate limit sliding window documentado; `Retry-After` header incluído                                                                  | ✅     |
| 8   | Sem conflitos não marcados entre docs                                                                                                 | ✅     |
| 9   | 3 camadas de isolamento documentadas (scope + context + agent instruction)                                                            | ✅     |
| 10  | Template A aplicado: Banco→Backend→Wiring(N/A)→Testes                                                                                 | ✅     |
| 11  | Nenhuma lógica de negócio de IA — apenas auth, sessão, LGPD, rate limit                                                               | ✅     |
| 12  | REQs cobertos: REQ-004, REQ-005, REQ-006, REQ-022, REQ-023, REQ-028, REQ-071, REQ-072, REQ-094, REQ-111, REQ-135 — todos com ≥1 item  | ✅     |
