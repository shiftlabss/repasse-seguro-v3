# S8 — Prontidão para Lançamento

## Metadados

| Campo         | Valor                                                                                                         |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| Sprint        | S8                                                                                                            |
| Nome          | Prontidão para Lançamento                                                                                     |
| Template      | B (Módulo Fullstack por Feature)                                                                              |
| Módulo        | AI-Dani-Admin                                                                                                 |
| Docs Fonte    | D05 (RF-023, RF-026), D06, D07, D08, D09, D10, D11, D12, D13, D16, D20, D25 — RN-DA-037, RN-DA-038, RN-DA-039 |
| REQs Cobertos | REQs de S8 no registro-mestre.md                                                                              |
| Data          | 2026-03-24                                                                                                    |

## Objetivo

Implementar o módulo de Prontidão para Lançamento (Launch Readiness): checklist de critérios de prontidão (RN-DA-037 a RN-DA-039), registro e execução de testes adversariais (mín. 20 por agente — MIN_ADVERSARIAL_TESTS_BEFORE_LAUNCH=20), tela T-007 (Checklist de Prontidão), validação do filtro de escopo (RN-DA-037), cobertura de recusa (RN-DA-038), supervisão funcional (RN-DA-039), módulo Mobile Admin (D11), e push notification tokens.

---

## FEATURE 1 — Checklist de Prontidão (RF-023, RN-DA-037 a RN-DA-039)

### BANCO (Feature 1)

- [x] Confirmar que `launch_readiness_checklists` tem todos os campos de S1:
  - `scope_filter_validated BOOLEAN`, `adversarial_tests_passed INTEGER`, `system_prompt_approved BOOLEAN`, `supervision_functional BOOLEAN`, `validated_by UUID`, `validated_at TIMESTAMPTZ`
- [x] Confirmar que `adversarial_test_results` tem campos: `test_category`, `test_input`, `expected_behavior`, `actual_behavior`, `passed`
- [x] Confirmar que constraint `CHECK (adversarial_tests_passed >= 0)` existe

### BACKEND (Feature 1)

- [x] Criar `apps/api/src/modules/launch-readiness/launch-readiness.module.ts`
- [x] Criar `apps/api/src/modules/launch-readiness/launch-readiness.service.ts`
- [x] Criar `apps/api/src/modules/launch-readiness/launch-readiness.controller.ts`
- [x] Criar `apps/api/src/modules/launch-readiness/dto/create-adversarial-test.dto.ts`:
  - `checklistId: string` — UUID
  - `agentId: string` — UUID
  - `testCategory: string` — uma das 7 categorias adversariais de D27:
    - `'extração_de_dados_bloqueados'`
    - `'prompt_injection'`
    - `'role_confusion'`
    - `'data_leakage'`
    - `'privilege_escalation'`
    - `'context_poisoning'`
    - `'jailbreak'`
  - `testInput: string` — o prompt adversarial
  - `expectedBehavior: string` — comportamento esperado do agente
  - `actualBehavior?: string` — resultado observado
  - `passed: boolean`
- [x] Implementar `LaunchReadinessService.getOrCreateChecklist(agentId: string)`:
  - Busca checklist mais recente do agente; se não existe, cria novo com todos os campos `false`/`0`
  - Retorna checklist com contagem de testes adversariais passados
- [x] Implementar `LaunchReadinessService.addAdversarialTest(dto: CreateAdversarialTestDto, adminId: string)`:
  - Cria registro em `adversarial_test_results`
  - Atualiza `launch_readiness_checklists.adversarial_tests_passed` incrementando se `passed = true`
  - Registra em `admin_access_logs`: `{ action: 'ADVERSARIAL_TEST_ADDED', targetType: 'launch_readiness_checklist', targetId: checklistId }`
- [x] Implementar `LaunchReadinessService.validateScopeFilter(checklistId: string, adminId: string)`:
  - Atualiza `scope_filter_validated = true`
  - Registra em `admin_access_logs`: `{ action: 'SCOPE_FILTER_VALIDATED' }`
- [x] Implementar `LaunchReadinessService.approveSystemPrompt(checklistId: string, adminId: string)`:
  - Atualiza `system_prompt_approved = true`
  - Registra em `admin_access_logs`: `{ action: 'SYSTEM_PROMPT_APPROVED' }`
- [x] Implementar `LaunchReadinessService.validateSupervision(checklistId: string, adminId: string)`:
  - Atualiza `supervision_functional = true`
  - Registra em `admin_access_logs`: `{ action: 'SUPERVISION_VALIDATED' }`
- [x] Implementar `LaunchReadinessService.approveChecklist(checklistId: string, adminId: string)`:
  - Verifica todos os critérios de prontidão:
    - `scope_filter_validated = true` (RN-DA-037)
    - `adversarial_tests_passed >= MIN_ADVERSARIAL_TESTS_BEFORE_LAUNCH = 20` (RN-DA-038)
    - `system_prompt_approved = true` (RN-DA-038)
    - `supervision_functional = true` (RN-DA-039)
  - Se algum critério não atendido → HTTP 409 `DA-LRC-001` com detalhes dos critérios pendentes
  - Se todos atendidos: `validated_by = adminId`, `validated_at = now()`
  - Emite evento PostHog `launch_readiness_approved`
  - Registra em `admin_access_logs`: `{ action: 'CHECKLIST_APPROVED' }`
- [x] Implementar `GET /api/v1/admin/launch-readiness/:agentId` — retorna checklist atual
- [x] Implementar `POST /api/v1/admin/launch-readiness/:agentId/adversarial-tests` — adiciona teste adversarial
- [x] Implementar `POST /api/v1/admin/launch-readiness/:checklistId/validate-scope` — valida filtro de escopo
- [x] Implementar `POST /api/v1/admin/launch-readiness/:checklistId/approve-prompt` — aprova system prompt
- [x] Implementar `POST /api/v1/admin/launch-readiness/:checklistId/validate-supervision` — valida supervisão
- [x] Implementar `POST /api/v1/admin/launch-readiness/:checklistId/approve` — aprovação final
- [x] Todos os endpoints: Guards `@Roles('ADMIN')`

### FRONTEND (Feature 1)

- [x] Criar `apps/web/src/app/admin/launch-readiness/page.tsx` — tela T-007 (Checklist de Prontidão) conforme D06:
  - Seletor de agente: DANI_CESSIONARIO ou DANI_CEDENTE
  - 4 critérios de prontidão exibidos como checklist visual:
    - "Filtro de escopo validado" — botão "Validar" + checkbox verde quando `scope_filter_validated = true` (RN-DA-037)
    - "Testes adversariais: X/20 passados" — progress bar; botão "Adicionar teste" (RN-DA-038, MIN_ADVERSARIAL_TESTS_BEFORE_LAUNCH=20)
    - "System prompt aprovado" — botão "Aprovar" + checkbox (RN-DA-038)
    - "Supervisão funcional validada" — botão "Validar" + checkbox (RN-DA-039)
  - Botão "Aprovar para lançamento" — visível e habilitado apenas quando todos os 4 critérios atendidos
  - Em sucesso: toast "Agente aprovado para lançamento." (D08)
  - Em erro 409 `DA-LRC-001`: lista dos critérios pendentes em toast de erro
- [x] Criar `apps/web/src/app/admin/launch-readiness/_components/adversarial-test-form.tsx`:
  - Formulário para adicionar teste adversarial
  - Campos: `testCategory` (Select com 7 opções de D27), `testInput` (Textarea), `expectedBehavior` (Textarea), `actualBehavior` (Textarea), `passed` (Toggle)
  - Usa MODAL-003 (adicionar teste)
- [x] Criar `apps/web/src/app/admin/launch-readiness/_components/adversarial-test-list.tsx`:
  - Tabela com todos os testes: `testCategory`, `testInput` (truncado), `passed` (badge verde/vermelho), `testedAt`
  - Contagem: "X testes passados de Y registrados"

### WIRING (Feature 1)

- [x] Verificar que após 20 testes adversariais passados, `adversarial_tests_passed = 20`
- [x] Verificar que `approveChecklist` falha com critérios pendentes retornando detalhes no erro
- [x] Verificar que aprovação bem-sucedida registra `validated_at` e dispara evento PostHog

### TESTES (Feature 1)

- [x] Criar `launch-readiness.service.spec.ts` com Vitest:
  - Test: `approveChecklist` com todos os critérios ok → sucesso, `validated_at` preenchido
  - Test: `approveChecklist` com `adversarial_tests_passed = 19` → 409 `DA-LRC-001`
  - Test: `approveChecklist` com `scope_filter_validated = false` → 409 `DA-LRC-001`
  - Test: `addAdversarialTest` com `passed = true` → incrementa `adversarial_tests_passed`
  - Test: `addAdversarialTest` com `passed = false` → NÃO incrementa contador
  - Test: aprovação dispara evento PostHog `launch_readiness_approved`
  - Test: log de auditoria criado para cada ação

---

## FEATURE 2 — Módulo Mobile Admin (D11)

### BANCO (Feature 2)

- [x] Confirmar que `push_notification_tokens` tem campos `admin_id`, `token`, `platform` (`ios`|`android`), `is_active`, `deleted_at`
- [x] Confirmar que constraint `UNIQUE (admin_id, token)` existe

### BACKEND (Feature 2)

- [x] Criar `apps/api/src/modules/push-tokens/push-tokens.module.ts`
- [x] Criar `apps/api/src/modules/push-tokens/push-tokens.service.ts`
- [x] Criar `apps/api/src/modules/push-tokens/push-tokens.controller.ts`
- [x] Criar `apps/api/src/modules/push-tokens/dto/register-token.dto.ts`:
  - `token: string` — obrigatório (Expo Push Token)
  - `platform: 'ios' | 'android'` — obrigatório
- [x] Implementar `PushTokensService.registerToken(adminId: string, dto: RegisterTokenDto)`:
  - Upsert em `push_notification_tokens` via `(admin_id, token)` como chave única
  - Se existir com `is_active = false`: reativa (`is_active = true`, `deleted_at = null`)
  - Retorna `{ tokenId, isNew: boolean }`
- [x] Implementar `PushTokensService.deactivateToken(adminId: string, token: string)`:
  - Soft delete: `is_active = false`, `deleted_at = now()`
  - Verifica que token pertence ao `adminId` — senão 403 `DA-MOB-001`
- [x] Implementar `PushTokensService.getActiveTokensByAdmin(adminId: string)`:
  - Retorna tokens ativos (`is_active = true`, `deleted_at IS NULL`) do admin
- [x] Implementar `POST /api/v1/mobile/push-tokens` — registrar token:
  - Guards: `@Roles('ADMIN')` — JWT mobile do admin
  - HTTP 201
- [x] Implementar `DELETE /api/v1/mobile/push-tokens/:token` — desativar token:
  - Guards: `@Roles('ADMIN')`
  - HTTP 200

### FRONTEND Mobile — React Native / Expo (Feature 2)

> Conforme D11, o Mobile Admin v1.0 usa React Native 0.76+, Expo SDK 52+, expo-router 4.x+. As telas mobile do Admin são separadas do painel web.

- [ ] Confirmar que `apps/mobile/` existe com Expo SDK 52+ configurado — BLOCKED: apps/mobile/ nao existe no monorepo. Mobile e fora do escopo deste modulo.
- [ ] Instalar dependências mobile (D11) — BLOCKED: dependente do item acima
- [ ] Criar `apps/mobile/src/screens/alerts-screen.tsx` — BLOCKED
- [ ] Criar `apps/mobile/src/screens/interactions-screen.tsx` — BLOCKED
- [ ] Criar `apps/mobile/src/screens/interaction-detail-screen.tsx` — BLOCKED
- [ ] Criar `apps/mobile/src/screens/takeover-screen.tsx` — BLOCKED
- [ ] Criar `apps/mobile/src/services/push-notification.service.ts` — BLOCKED
- [ ] Verificar que features mobile que ficam FORA do v1.0 (D11) NÃO são implementadas — BLOCKED

### WIRING (Feature 2)

- [x] Verificar que `POST /api/v1/mobile/push-tokens` com JWT mobile válido registra token
- [x] Verificar que `ExpoService.sendPushNotification` (S5) usa tokens retornados por `PushTokensService.getActiveTokensByAdmin`
- [ ] Verificar que push é recebido no app mobile ao criar alerta P0 — BLOCKED: apps/mobile/ nao existe

### TESTES (Feature 2)

- [x] Criar `push-tokens.service.spec.ts`:
  - Test: `registerToken` cria novo registro
  - Test: `registerToken` com token já existente mas inativo → reativa (não duplica)
  - Test: `deactivateToken` com `adminId` correto → soft delete
  - Test: `deactivateToken` com `adminId` errado → 403 `DA-MOB-001`
  - Test: `getActiveTokensByAdmin` retorna apenas tokens com `is_active = true`

---

## 🔀 Cross-Módulo

- [x] `LaunchReadinessService.approveChecklist` emite `launch_readiness_approved` via `PostHogService` (S7)
- [x] `PushTokensService.getActiveTokensByAdmin` é chamado por `ExpoService` em S5 — verificar que `PushTokensModule` está importado em `AlertsModule`
- [x] Tela T-007 linka para T-005 (Configurações) para aprovação do system prompt

---

## AUTO-VERIFICAÇÃO S8

| Check                 | Critério                                                                                                                                                                      | Status              |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| #1 Nomenclatura       | `launch_readiness_checklists`, `adversarial_test_results`, `push_notification_tokens`, `DA-LRC-001`, `DA-MOB-001`, `MIN_ADVERSARIAL_TESTS_BEFORE_LAUNCH=20` usados exatamente | ✅                  |
| #2 Verificabilidade   | Cada item binariamente verificável                                                                                                                                            | ✅                  |
| #3 Valores numéricos  | MIN_ADVERSARIAL_TESTS=20, 4 critérios de prontidão, 7 categorias adversariais, 9 dependências mobile                                                                          | ✅                  |
| #4 N itens completos  | 2 features: checklist de prontidão, módulo mobile (4 telas + push tokens)                                                                                                     | ✅ (mobile BLOCKED) |
| #5 Máquinas de estado | Checklist: 4 critérios booleanos → estado final `validated_at`                                                                                                                | ✅                  |
| #6 Schedules/TTLs     | Tokens mobile: soft delete com `deleted_at`; sem TTL automático                                                                                                               | ✅                  |
| #7 Conflitos          | Nenhum novo                                                                                                                                                                   | ✅                  |
| #8 Ambiguidades       | Nenhuma nova                                                                                                                                                                  | ✅                  |
| #9 Anti-scaffold      | `approveChecklist` verifica 4 critérios reais antes de aprovar; retorna detalhes dos critérios pendentes                                                                      | ✅                  |
| #10 Cross-módulo      | `PushTokensModule` importado em `AlertsModule`; `PostHogService` chamado em aprovação                                                                                         | ✅                  |
| #11 IDs de referência | RN-DA-037, RN-DA-038, RN-DA-039, RF-023, D11, D06, D07 citados                                                                                                                | ✅                  |
| #12 Cobertura REQs S8 | Todos os REQs de prontidão e mobile no registro-mestre.md têm ≥ 1 item                                                                                                        | ✅ (mobile BLOCKED) |
