# S4 — Takeover Manual

## Metadados

| Campo | Valor |
|---|---|
| Sprint | S4 |
| Nome | Takeover Manual |
| Template | B (Módulo Fullstack por Feature) |
| Módulo | AI-Dani-Admin |
| Docs Fonte | D05 (RF-004 a RF-009, RF-026), D06, D07, D08, D09, D10, D12, D13, D14, D16, D20, D25 |
| REQs Cobertos | REQs de S4 no registro-mestre.md |
| Data | 2026-03-24 |

## Objetivo

Implementar o módulo de Takeover Manual completo (fullstack): iniciar takeover (RF-004), enviar mensagem em takeover (RF-005), encerrar takeover (RF-006), concorrência simultânea com lock otimista ADR-001 (RF-007), máquina de estados de interação (RF-008), tela T-002 (componentes de takeover), MODAL-001 (confirmação), COMP-002 (separador visual), log de auditoria de takeover, e todos os testes associados.

---

## FEATURE 1 — Iniciar Takeover (RF-004, RF-007)

### BANCO (Feature 1)

- [x] Confirmar que tabela `takeovers` tem constraint `UNIQUE (interaction_id)` (parcial: onde `status != 'ENCERRADO'`) — ADR-001 lock otimista
- [x] Confirmar que índice `(interaction_id)` existe em `takeovers`
- [x] Confirmar que campos `admin_id`, `status`, `started_at`, `ended_at`, `admin_message` existem com tipos corretos

### BACKEND (Feature 1)

- [x] Criar `apps/api/src/modules/takeover/takeover.module.ts` com imports: `PrismaModule`, `AuditLogModule`, `RabbitMQModule`
- [x] Criar `apps/api/src/modules/takeover/takeover.service.ts`
- [x] Criar `apps/api/src/modules/takeover/takeover.controller.ts`
- [x] Criar `apps/api/src/modules/takeover/dto/start-takeover.dto.ts` — [CORRIGIDO: FINDING-002]:
  - `interactionId: string` — UUID obrigatório (extraído do path param)
  - `reason?: string` — opcional, max 500 caracteres; motivo do takeover (D16/REQ-203)
- [x] Implementar `TakeoverService.startTakeover(interactionId: string, adminId: string)`:
  - Verifica que `interaction` existe e `deleted_at IS NULL` — senão 404 `DA-TAK-001`
  - Verifica que `interaction.status` é `SINALIZADA_PARA_REVISAO` — senão 409 `DA-TAK-002` (`'Interação não está disponível para takeover'`)
  - Verifica que `interaction.status` NÃO é `ENCERRADA` — senão 409 `DA-TAK-003` (`'Interação já encerrada'`)
  - Lock otimista (ADR-001): tenta `INSERT INTO takeovers (interaction_id, admin_id, status) VALUES (...) ON CONFLICT (interaction_id) DO NOTHING`
  - Se `ON CONFLICT` ativou (nenhuma row inserida) → 409 `DA-TAK-004` (`'Outro administrador já assumiu esta conversa'`)
  - Se sucesso: atualiza `interactions.status = 'EM_TAKEOVER'`
  - Toda a operação em transaction Prisma (`prisma.$transaction`)
  - Registra em `admin_access_logs`: `{ action: AdminActionType.TAKEOVER_INICIADO, targetType: 'interaction', targetId: interactionId, afterState: { status: 'EM_TAKEOVER' } }` — [CORRIGIDO: FINDING-005]
  - Retorna `{ takeoverId, interactionId, status: 'EM_TAKEOVER', startedAt }`
- [x] Implementar `POST /api/v1/admin/interactions/:id/takeover` no `TakeoverController`:
  - Guards: `@Roles('ADMIN')`
  - Extrai `adminId` do JWT payload
  - HTTP 201 em sucesso
- [x] Implementar `GET /api/v1/admin/interactions/:id/takeover` no `TakeoverController` — [CORRIGIDO: FINDING-009]:
  - Guards: `@Roles('ADMIN')`
  - Retorna takeover ativo da interação: `{ id, interactionId, adminId, status, startedAt, adminMessage }`
  - Se não existe takeover ativo → HTTP 404 com code `DA-TAK-007`

### FRONTEND (Feature 1)

- [x] Criar `apps/web/src/app/admin/interactions/[id]/_components/takeover-button.tsx` — [CORRIGIDO: FINDING-002]:
  - Botão "Assumir conversa" (D10 — texto aprovado)
  - Visível apenas quando `status === 'SINALIZADA_PARA_REVISAO'`
  - Ao clicar: abre MODAL-001 (confirmação) antes de chamar API
  - Estado loading durante chamada API
  - Em sucesso: toast "Takeover iniciado com sucesso. Você agora controla esta conversa." (D08)
  - Em erro 409 `DA-TAK-004`: toast de erro "Outro administrador já assumiu esta conversa." (D08)
  - Em erro 409 `DA-TAK-002`: toast de erro "Interação não está disponível para takeover."
- [x] Criar Select de motivo de takeover com 5 opções exatas (D08 seção 3.4 / REQ-145) — [CORRIGIDO: FINDING-002]:
  - "Resposta incorreta da IA"
  - "Confiança abaixo do aceitável"
  - "Situação complexa para a IA"
  - "Solicitação do usuário"
  - "Outro"
  - Campo incluído no MODAL-001 de confirmação antes de chamar API
- [x] Criar `apps/web/src/app/admin/interactions/[id]/_components/takeover-confirm-modal.tsx` — MODAL-001:
  - Título: "Assumir conversa?"
  - Corpo: "Você irá assumir o controle desta conversa. O agente será pausado."
  - Botões: "Cancelar" (secundário) e "Assumir conversa" (primário, destrutivo) — D08
  - Usa `Dialog` do shadcn/ui (D09)

### WIRING (Feature 1)

- [x] Verificar que `POST /api/v1/admin/interactions/:id/takeover` cria registro em `takeovers` e atualiza `interactions.status`
- [x] Verificar que tentativa simultânea de takeover retorna 409 `DA-TAK-004`
- [x] Verificar que log é criado em `admin_access_logs`
- [x] Verificar que botão "Assumir conversa" aparece na T-002 quando `status = 'SINALIZADA_PARA_REVISAO'`

### TESTES (Feature 1)

- [x] Criar `takeover.service.spec.ts` com Vitest:
  - Test: `startTakeover(interactionId, adminId)` com interação `SINALIZADA_PARA_REVISAO` → cria takeover, atualiza status para `EM_TAKEOVER`
  - Test: `startTakeover` com interação `ENCERRADA` → lança `ConflictException` com code `DA-TAK-003`
  - Test: `startTakeover` com interação inexistente → lança `NotFoundException` com code `DA-TAK-001`
  - Test: `startTakeover` com takeover ativo existente → lança `ConflictException` com code `DA-TAK-004` (ADR-001)
  - Test: toda a operação roda em transaction Prisma (`prisma.$transaction`)
  - Test: log de auditoria criado com `action: 'TAKEOVER_STARTED'`

---

## FEATURE 2 — Enviar Mensagem em Takeover (RF-005)

### BANCO (Feature 2)

- [x] Confirmar que `takeovers.admin_message` é TEXT nullable — armazena última mensagem do admin

### BACKEND (Feature 2)

- [x] Criar `apps/api/src/modules/takeover/dto/send-takeover-message.dto.ts`:
  - `message: string` — obrigatório, min 1 caractere, max 5000 caracteres
- [x] Implementar `TakeoverService.sendMessage(interactionId: string, adminId: string, message: string)`:
  - Verifica que `interaction` tem `status = 'EM_TAKEOVER'` — senão 409 `DA-TAK-005` (`'Conversa não está em takeover'`)
  - Verifica que o takeover ativo pertence ao `adminId` — senão 403 `DA-TAK-006` (`'Você não é o responsável por este takeover'`)
  - Atualiza `takeovers.admin_message = message`
  - Atualiza `interactions.updated_at = now()`
  - Retorna `{ takeoverId, message, sentAt }`
- [x] Implementar `POST /api/v1/admin/interactions/:id/takeover/message` no `TakeoverController`:
  - Guards: `@Roles('ADMIN')`
  - HTTP 200 em sucesso

### FRONTEND (Feature 2)

- [x] Criar `apps/web/src/app/admin/interactions/[id]/_components/takeover-chat.tsx`:
  - Input de texto + botão "Enviar" — visível apenas quando `status === 'EM_TAKEOVER'` e admin é o responsável
  - Ao enviar: chama `POST /api/v1/admin/interactions/:id/takeover/message`
  - Mensagem enviada aparece imediatamente no chat transcript (otimistic update)
  - COMP-002: separador visual "Atendimento humano" acima das mensagens do admin (D10)
  - Estado loading durante envio

### WIRING (Feature 2)

- [x] Verificar que mensagem enviada persiste em `takeovers.admin_message`
- [x] Verificar que admin que não é o responsável recebe 403

### TESTES (Feature 2)

- [x] Continuar em `takeover.service.spec.ts`:
  - Test: `sendMessage` com interação `EM_TAKEOVER` e admin correto → sucesso
  - Test: `sendMessage` com interação `RESPONDIDA_PELA_IA` → 409 `DA-TAK-005`
  - Test: `sendMessage` com admin diferente do responsável → 403 `DA-TAK-006`

---

## FEATURE 3 — Encerrar Takeover (RF-006)

### BANCO (Feature 3)

- [x] Confirmar que `takeovers.ended_at` é TIMESTAMPTZ nullable — preenchido ao encerrar
- [x] Confirmar que `interactions.status` transita para `ENCERRADA` ao encerrar takeover

### BACKEND (Feature 3)

- [x] Implementar `TakeoverService.endTakeover(interactionId: string, adminId: string)`:
  - Verifica que interação tem `status = 'EM_TAKEOVER'` — senão 409 `DA-TAK-005`
  - Verifica que takeover ativo pertence ao `adminId` — senão 403 `DA-TAK-006`
  - Em transaction Prisma:
    - Atualiza `takeovers.status = 'ENCERRADO'`, `takeovers.ended_at = now()`
    - Atualiza `interactions.status = 'ENCERRADA'`
  - Registra em `admin_access_logs`: `{ action: AdminActionType.TAKEOVER_ENCERRADO, targetType: 'interaction', targetId: interactionId, beforeState: { status: 'EM_TAKEOVER' }, afterState: { status: 'ENCERRADA' } }` — [CORRIGIDO: FINDING-005]
  - Retorna `{ takeoverId, endedAt, interactionStatus: 'ENCERRADA' }`
- [x] Implementar `DELETE /api/v1/admin/interactions/:id/takeover` no `TakeoverController`:
  - Guards: `@Roles('ADMIN')`
  - HTTP 200 em sucesso
- [x] Verificar que interação `ENCERRADA` NÃO aceita novo takeover — `startTakeover` deve retornar 409 `DA-TAK-003`

### FRONTEND (Feature 3)

- [x] Criar `apps/web/src/app/admin/interactions/[id]/_components/end-takeover-button.tsx`:
  - Botão "Encerrar atendimento" — visível apenas quando `status === 'EM_TAKEOVER'` e admin é o responsável
  - Ao clicar: abre MODAL-002 (confirmação de encerramento) antes de chamar API
  - Em sucesso: toast "Atendimento encerrado. A interação foi marcada como Encerrada." (D08)
- [x] Criar `apps/web/src/app/admin/interactions/[id]/_components/end-takeover-confirm-modal.tsx` — MODAL-002:
  - Título: "Encerrar atendimento?"
  - Corpo: "A interação será marcada como Encerrada e não poderá receber novo atendimento."
  - Botões: "Cancelar" (secundário) e "Encerrar" (primário, destrutivo)

### WIRING (Feature 3)

- [x] Verificar que `DELETE /api/v1/admin/interactions/:id/takeover` encerra takeover e marca interação como `ENCERRADA`
- [x] Verificar que tentativa de novo takeover em interação `ENCERRADA` retorna 409 `DA-TAK-003`
- [x] Verificar que log `TAKEOVER_ENDED` é criado em `admin_access_logs`

### TESTES (Feature 3)

- [x] Continuar em `takeover.service.spec.ts`:
  - Test: `endTakeover` com interação `EM_TAKEOVER` → atualiza `takeovers.status = 'ENCERRADO'` e `interactions.status = 'ENCERRADA'`
  - Test: `endTakeover` com interação não em takeover → 409 `DA-TAK-005`
  - Test: `endTakeover` com admin diferente → 403 `DA-TAK-006`
  - Test: após `endTakeover`, `startTakeover` na mesma interação → 409 `DA-TAK-003`
  - Test: log de auditoria criado com `action: 'TAKEOVER_ENDED'` e `beforeState/afterState` corretos
  - Test: toda a operação roda em transaction Prisma

---

## FEATURE 4 — Máquina de Estados Completa (RF-008)

### BACKEND (Feature 4)

Documentar e implementar validações para TODAS as transições válidas e inválidas de `InteractionStatus`:

- [x] `SINALIZADA_PARA_REVISAO` → `EM_TAKEOVER`: via `startTakeover` — VÁLIDA
- [x] `SINALIZADA_PARA_REVISAO` → `RESPONDIDA_PELA_IA`: agente gera nova resposta com confiança >= threshold sem intervenção humana — VÁLIDA (REQ-013) — [CORRIGIDO: FINDING-010]
- [x] `SINALIZADA_PARA_REVISAO` → `ENCERRADA`: via `endTakeover` sem takeover ativo → INVÁLIDA (409 `DA-TAK-005`)
- [x] `EM_TAKEOVER` → `ENCERRADA`: via `endTakeover` — VÁLIDA
- [x] `ENCERRADA` → qualquer: INVÁLIDA (409 `DA-TAK-003`)
- [x] `RESPONDIDA_PELA_IA` → `EM_TAKEOVER`: VÁLIDA — Admin pode assumir qualquer conversa ativa por qualquer motivo (REQ-031, REQ-014) — [CORRIGIDO: FINDING-010]
- [x] `RESPONDIDA_PELA_IA` → `ENCERRADA`: via sistema (automático após SLA) — implementado em S7 (agente IA)
- [x] Criar helper `InteractionStateMachine.canTransition(from: InteractionStatus, to: InteractionStatus): boolean`
- [x] Criar testes unitários para o helper com todas as 6 transições acima

### FRONTEND (Feature 4)

- [x] Criar `apps/web/src/lib/interaction-state-machine.ts` — espelha a lógica do backend:
  - `canInitiateTakeover(status: InteractionStatus): boolean` — true apenas para `SINALIZADA_PARA_REVISAO`
  - `canEndTakeover(status: InteractionStatus): boolean` — true apenas para `EM_TAKEOVER`
  - `isTerminal(status: InteractionStatus): boolean` — true para `ENCERRADA`
- [x] Usar helper para controlar visibilidade de botões na T-002 (não confiar apenas no estado da API)

---

## Cross-Módulo

- [x] `TakeoverService.startTakeover` publica evento `admin_takeover_initiated` no PostHog — implementar apenas o hook de tracking (PostHogService injetado). PostHog implementado em S7, mas o hook de evento deve ser inserido aqui com `TODO: conectar PostHogService em S7`.
- [x] `TakeoverService.endTakeover` deve notificar via RabbitMQ (para alertas) — depende de S5 (AlertsModule). Resolvido em S5: AlertsService injetado e evento publicado.

---

## AUTO-VERIFICAÇÃO S4

| Check | Critério | Status |
|---|---|---|
| #1 Nomenclatura | `takeovers`, `TakeoverStatus`, `ATIVO`, `ENCERRADO`, `DA-TAK-001` a `DA-TAK-006`, ADR-001 usados exatamente | PASS |
| #2 Verificabilidade | Cada item binariamente verificável | PASS |
| #3 Valores numéricos | `admin_message` max 5000 chars; transaction Prisma | PASS |
| #4 N itens completos | 4 features: iniciar, mensagem, encerrar, máquina de estados | PASS |
| #5 Máquinas de estado | TODAS as 6 transições de `InteractionStatus` documentadas (válidas e inválidas) | PASS |
| #6 Schedules/TTLs | N/A em S4 | PASS |
| #7 Conflitos | Nenhum conflito novo | PASS |
| #8 Ambiguidades | Nenhuma ambiguidade nova | PASS |
| #9 Anti-scaffold | `TakeoverService.startTakeover` usa INSERT ON CONFLICT real (ADR-001), transaction, audit log | PASS |
| #10 Cross-módulo | PostHog event hook (S7) e RabbitMQ notification (S5) documentados com TODO | PASS |
| #11 IDs de referência | RF-004 a RF-009, ADR-001, D06, D07, D08, MODAL-001/002, COMP-002 citados | PASS |
| #12 Cobertura REQs S4 | Todos os REQs de takeover no registro-mestre.md têm ≥ 1 item | PASS |
