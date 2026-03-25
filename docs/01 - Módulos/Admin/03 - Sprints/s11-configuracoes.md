# S11 — Configurações

## Repasse Seguro — Módulo Admin

| Campo | Valor |
|---|---|
| **Sprint** | S11 |
| **Nome** | Configurações |
| **Template** | B — Módulo Fullstack |
| **REQs cobertos** | REQ-024, REQ-163, REQ-164, REQ-165, REQ-173, REQ-242, REQ-243, REQ-244, REQ-245, REQ-246, REQ-247 |
| **Docs fonte** | D01.4 §4 (RN-107 a RN-120), D01.5 §2 (RN-124), D06 §2.11 (T-095 a T-100) |
| **Itens totais** | 38 |

---

> **Objetivo da sprint:** Implementar o módulo de Configurações com acesso exclusivo MASTER (RN-109), 5 seções: Comissões e Parâmetros Financeiros (T-096), Prazos e SLAs (T-097), Templates de Notificação (T-098), Integrações ZapSign/Celcoin/Meta/Twilio (T-099), Usuários e Permissões (T-100); audit trail obrigatório em toda alteração (RN-110); confirmação dupla em parâmetros críticos (RN-112); rollback de parâmetros numéricos (RN-113); alterações prospectivas only (RN-111); modo manutenção (RN-119); versionamento de templates ZapSign (RN-124); API keys mascaradas com reveal auditado.

---

## FEATURE 1 — Infraestrutura de Configurações

### Banco de Dados

**[BE-01] Validar schema `global_configs` e `case_config_snapshots`**
- Confirmar tabela `global_configs` com colunas: `id` (UUID PK), `config_key` (VARCHAR UNIQUE), `config_value` (TEXT), `data_type` (ENUM `string | number | boolean | json`), `description`, `update_reason` (obrigatório), `updated_by` (UUID FK), `updated_at`, `previous_value` (TEXT — para rollback), `is_critical` (boolean — confirmação dupla obrigatória)
- Confirmar tabela `case_config_snapshots` com colunas: `id`, `case_id` (UUID FK UNIQUE), `commission_cedente_pct`, `commission_cessionario_pct`, `delta_review_threshold`, `distrato_reference_pct`, `reversal_days`, `snapshot_at` (imutável após criação)
- Confirmar índice `UNIQUE(config_key)` em `global_configs`
- Confirmar que `case_config_snapshots.snapshot_at` é preenchido na criação do caso e jamais atualizado
- Verificação: `prisma migrate deploy` sem erros; alterar `global_configs` não altera `case_config_snapshots` de casos existentes

### Backend

**[BE-02] Implementar `ConfigsService` — CRUD de configurações com audit trail**
- `GET /v1/configs`: role `MASTER`; retorna todos os `global_configs` como `{ key, value, data_type, description, is_critical, updated_at, updated_by_name, previous_value }`
- `GET /v1/configs/:key`: role `MASTER`; retorna parâmetro específico + histórico de alterações (query em `audit.audit_logs` WHERE `action = 'CONFIG_UPDATED'` AND `metadata->>'config_key' = key` ORDER BY `created_at DESC`)
- `PATCH /v1/configs/:key`: role `MASTER`; body: `{ value, update_reason: string (min 20 chars) }`
  - Validar `update_reason` ≥ 20 chars; se < 20 → `422` com message `"O motivo da alteração deve ter pelo menos 20 caracteres."`
  - Salvar `previous_value = current_value` antes de atualizar
  - Registrar em `audit.audit_logs`: `action = 'CONFIG_UPDATED'`, `metadata = { config_key, previous_value, new_value, update_reason, master_id }`
  - **Alterações prospectivas only (RN-111):** jamais recalcular `case_config_snapshots` de casos existentes
  - Invalidar cache Redis `rs:configs:all` ao salvar
  - Retornar `200` com config atualizada
- `POST /v1/configs/:key/rollback`: role `MASTER`; restaurar `config_value = previous_value`; registrar audit `action = 'CONFIG_ROLLED_BACK'`; apenas para parâmetros numéricos (`data_type = 'number'`) — RN-113; templates e integrações → `422 "Rollback não disponível para este tipo de parâmetro."`
- Verificação: PATCH com `update_reason` de 15 chars retorna 422; PATCH válido atualiza `global_configs` + registra audit + invalida cache; rollback de template retorna 422

**[BE-03] Implementar confirmação dupla para parâmetros críticos (RN-112)**
- Parâmetros críticos (`is_critical = true`): percentual de comissão Cedente, percentual de comissão Cessionário, prazo de reversão, modo de distribuição escrow
- Para esses parâmetros: `PATCH /v1/configs/:key` com body incluindo `{ ..., confirm_double: false }` → retorna `202 Accepted` com `{ require_confirmation: true, summary: "Alterar [key] de [old] para [new]. Afeta apenas novos casos." }`; o cliente deve re-enviar com `confirm_double: true`
- Re-envio com `confirm_double: true` → executa a alteração normalmente
- Sem `confirm_double: true` em parâmetro crítico → `409` com message `"Parâmetro crítico requer confirmação dupla."`
- Verificação: PATCH de `commission_cedente_pct` sem `confirm_double: true` → 202 com summary; re-envio com `confirm_double: true` → 200 com atualização

---

## FEATURE 2 — Seção: Comissões e Parâmetros Financeiros (T-096)

### Backend

**[BE-04] Implementar endpoints de Comissões**
- Chaves configuráveis em `global_configs`:
  - `COMMISSION_CEDENTE_PCT` (number, default 20, range 5–50, `is_critical = true`)
  - `COMMISSION_CESSIONARIO_PCT` (number, default 20, range 5–50, `is_critical = true`)
  - `DELTA_REVIEW_THRESHOLD` (number, default 100000 — R$100.000)
  - `DISTRATO_REFERENCE_PCT` (number, default 50 — 50% do valor pago)
  - `COMMISSION_FLOOR_BRL` (number, default 0 — piso de comissão em R$)
  - `SCENARIO_A_CESSIONARIO_EXCEPTION` (boolean, default true — exceção Cenário A)
- Validações: `COMMISSION_CEDENTE_PCT` e `COMMISSION_CESSIONARIO_PCT` devem estar entre 5 e 50; se fora → `422`
- Todos exigem `update_reason ≥ 20 chars`; os 2 percentuais exigem confirmação dupla
- Verificação: PATCH `COMMISSION_CEDENTE_PCT` com valor 3 retorna 422; valor 25 com `confirm_double: true` → 200; `case_config_snapshots` de casos existentes inalterados

---

## FEATURE 3 — Seção: Prazos e SLAs (T-097)

### Backend

**[BE-05] Implementar endpoints de Prazos e SLAs**
- Chaves configuráveis em `global_configs`:
  - `SLA_CAPTADO_TRIAGEM_MAX_HOURS` (number, default 24)
  - `SLA_TRIAGEM_QUALIFICADO_TARGET_DAYS` (number, default 3)
  - `SLA_TRIAGEM_QUALIFICADO_MAX_DAYS` (number, default 5)
  - `SLA_QUALIFICADO_OFERTA_TARGET_DAYS` (number, default 2)
  - `SLA_QUALIFICADO_OFERTA_MAX_DAYS` (number, default 3)
  - `SLA_OFERTA_NEGOCIACAO_TARGET_DAYS` (number, default 15)
  - `SLA_OFERTA_NEGOCIACAO_MAX_DAYS` (number, default 30)
  - `SLA_NEGOCIACAO_FORMALIZACAO_TARGET_DAYS` (number, default 10)
  - `SLA_NEGOCIACAO_FORMALIZACAO_MAX_DAYS` (number, default 20)
  - `SLA_FORMALIZACAO_FECHAMENTO_TARGET_DAYS` (number, default 10)
  - `SLA_FORMALIZACAO_FECHAMENTO_MAX_DAYS` (number, default 20)
  - `REVERSAL_PERIOD_DAYS` (number, default 15, range 7–30, `is_critical = true`)
  - `ESCROW_DEPOSIT_DEADLINE_BUSINESS_DAYS` (number, default 10)
  - `ESCROW_DEPOSIT_EXTENSION_BUSINESS_DAYS` (number, default 5)
  - `AUTO_ESCALATION_DAYS` (number, default 30)
  - `MEDIATION_MAX_BUSINESS_DAYS` (number, default 10)
- `REVERSAL_PERIOD_DAYS` exige confirmação dupla
- `SlaMonitorService` relê configurações do Redis a cada ciclo de monitoramento (não hardcodado)
- Verificação: alterar `SLA_TRIAGEM_QUALIFICADO_MAX_DAYS` para 7 → novos casos usam 7 dias no SlaMonitorService

---

## FEATURE 4 — Seção: Templates de Notificação (T-098) e ZapSign (RN-124)

### Backend

**[BE-06] Implementar CRUD de templates de notificação e versionamento ZapSign (RN-124)**
- Templates de e-mail armazenados em tabela `notification_templates` (separada de `global_configs`): colunas `id`, `template_id` (ex: `rs_caso_captado`), `channel`, `subject`, `body_html`, `footer_html`, `variables` (JSONB), `version` (INTEGER), `activated_at`, `activated_by`, `is_active` (apenas 1 ativo por `template_id`)
- `GET /v1/configs/templates`: role `MASTER`; retorna todos os templates com versão ativa
- `PATCH /v1/configs/templates/:template_id`: role `MASTER`; cria nova versão (INSERT) — nunca altera versão existente (histórico imutável — RN-124); nova versão fica como `is_active = true`, versão anterior `is_active = false`
- `POST /v1/configs/templates/:template_id/test-send`: role `MASTER`; body: `{ recipient_email, variables: object }`; envia e-mail de preview via Resend (modo `dry_run` em desenvolvimento)
- Templates ZapSign (Instrumento de Cessão, Termo Comercial, Termo de Aceite): versão + data de ativação + histórico imutável; PDF gerado persistido (RN-124)
- Alterações de template registradas em `audit.audit_logs` com `action = 'TEMPLATE_UPDATED'`
- Verificação: PATCH de template cria nova versão (version+1), versão anterior permanece em histórico; test-send retorna status do envio

---

## FEATURE 5 — Seção: Integrações (T-099)

### Backend

**[BE-07] Implementar gestão de integrações ZapSign, Celcoin, Meta Cloud API, Twilio**
- Chaves em `global_configs` para cada integração:
  - ZapSign: `ZAPSIGN_API_KEY` (type: string, masked), `ZAPSIGN_WEBHOOK_SECRET` (type: string, masked), `ZAPSIGN_CONTINGENCY_ENABLED` (boolean)
  - Celcoin: `CELCOIN_CLIENT_ID` (masked), `CELCOIN_CLIENT_SECRET` (masked), `CELCOIN_WEBHOOK_SECRET` (masked)
  - Meta Cloud API: `META_WABA_TOKEN` (masked), `META_PHONE_NUMBER_ID` (masked)
  - Twilio: `TWILIO_ACCOUNT_SID` (masked), `TWILIO_AUTH_TOKEN` (masked), `TWILIO_FROM_NUMBER` (masked)
- **API keys mascaradas:** `GET /v1/configs` retorna valor mascarado `"****[últimos 4 chars]"` para chaves com `is_secret = true`; `GET /v1/configs/:key/reveal`: role `MASTER`; retorna valor completo; registra `audit.audit_logs` `action = 'SECRET_REVEALED'`
- `POST /v1/configs/integrations/:name/test`: role `MASTER`; testa conexão com a integração (ping ZapSign, Celcoin health check, Meta API test, Twilio test call); retorna `{ status: 'ok' | 'error', message, latency_ms }`
- Retornar status de conexão: `{ status: 'CONNECTED' | 'ERROR' | 'DISCONNECTED', last_sync_at }`
- Verificação: `GET /v1/configs/ZAPSIGN_API_KEY` retorna valor mascarado; `GET .../reveal` retorna valor completo + registra audit; test connection retorna status correto

---

## FEATURE 6 — Seção: Usuários e Permissões (T-100) e Modo Manutenção (RN-119)

### Backend

**[BE-08] Implementar modo manutenção (RN-119)**
- Chave `MAINTENANCE_MODE_ENABLED` (boolean, default false) em `global_configs`
- `PATCH /v1/configs/MAINTENANCE_MODE_ENABLED` com `{ value: 'true', update_reason: ... }`: ao ativar → invalidar todos os tokens JWT de usuários não-Master (setar `invalidated_before = NOW()` no Redis por perfil); ao desativar → remover flag
- Middleware global `MaintenanceModeGuard`: antes de qualquer request, verificar `MAINTENANCE_MODE_ENABLED` em Redis; se ativo e `req.user.role !== 'MASTER'` → retornar `503 Service Unavailable` com `{ message: "A plataforma está em manutenção programada. Tente novamente em alguns minutos." }`
- Verificação: ativar manutenção → request com token COORDENADOR retorna 503; Master continua com 200; desativar → todos voltam a funcionar

---

## FEATURE 7 — Frontend Configurações (T-095 a T-100)

### Frontend

**[FE-01] Implementar T-095 — Hub de Configurações (`/configuracoes`)**
- Rota: `/configuracoes`; módulo visível na sidebar APENAS para MASTER (oculto para todos os outros — menu inexistente)
- Acesso como não-Master → `403 Forbidden` retornado pelo backend; redirecionar para `/dashboard`
- Grid de 5 cards: Comissões e Parâmetros Financeiros, Prazos e SLAs, Templates de Notificação, Integrações, Usuários e Permissões
- Banner amarelo permanente: "Alterações aqui afetam toda a operação. Tenha cuidado e registre o motivo de cada alteração."
- Verificação: acessar `/configuracoes` como COORDENADOR → redirecionado para `/dashboard`; MASTER vê os 5 cards

**[FE-02] Implementar T-096 — Comissões e Parâmetros Financeiros**
- Campos: percentual comissão Cedente (range input 5–50%, step 1%, default 20%), percentual comissão Cessionário (idem), limiar revisão Delta (input número R$, default 100.000), Valor Distrato Referência (%, default 50%), piso comissão (R$, default 0), toggle exceção Cenário A
- Cada campo: label descritivo, valor atual, campo editável, valor padrão em texto menor, botão "Reverter ao padrão" (ativa POST rollback)
- Parâmetros críticos com ícone cadeado + tooltip "Alteração requer confirmação dupla."
- Campo "Motivo da alteração" obrigatório (mínimo 20 chars) com contador de caracteres — exibido ao editar qualquer campo
- **Fluxo confirmação dupla:** clicar "Salvar" → modal #1 mostra resumo; clicar "Confirmar" → modal #2 com checkbox "Li e entendo o impacto nos novos casos" → clicar "Confirmo a alteração" → salvar; cancelar em qualquer modal → nenhuma alteração aplicada
- Toast após salvar: "Parâmetro [nome] atualizado de [anterior] para [novo]. Afeta apenas novos casos."
- Link "Ver histórico" por parâmetro → drawer lateral com audit log do parâmetro
- Verificação: alterar `COMMISSION_CEDENTE_PCT` → 2 modais aparecem em sequência; cancelar no 2° → nada alterado; confirmar → toast + audit log

**[FE-03] Implementar T-097 — Prazos e SLAs**
- Tabela editável: 7 transições de SLA + prazos adicionais (reversão, depósito, escalonamento, mediação)
- Colunas: etapa, SLA alvo (dias), SLA máximo (dias) — ambos editáveis inline; última alteração (data); alterado por
- Inline edit: clicar no valor → campo de input inline; Tab para próximo; botão "Salvar tudo" ao final da tabela
- `REVERSAL_PERIOD_DAYS` com ícone de cadeado (confirmação dupla)
- Aviso global: "Alterações afetam apenas novos casos e eventos a partir desta data."
- Verificação: editar inline e clicar "Salvar tudo" → PATCH enviado com todos os valores modificados; `REVERSAL_PERIOD_DAYS` exige 2 modais

**[FE-04] Implementar T-098 — Templates de Notificação**
- Lista de 22 templates (conforme D21 §3.1) com colunas: nome, canal (badge), trigger event, pré-visualização do corpo, botão "Editar"
- "Editar" → drawer lateral com editor de texto rico; variáveis disponíveis listadas (ex: `{{case_id}}`, `{{cedente_name}}`); botão "Testar envio" (campo de e-mail destinatário + envio real/preview)
- Badge de versão: "v[N] — ativo desde [data]"; link "Ver versões anteriores" → histórico imutável de versões
- Salvar template: POST nova versão; confirmação: "Nova versão v[N+1] criada. A versão anterior v[N] permanecerá no histórico."
- Verificação: salvar template → versão incrementada; histórico mostra versões anteriores imutáveis; "Testar envio" enfileira e-mail de preview

**[FE-05] Implementar T-099 — Integrações**
- 4 cards: ZapSign, Celcoin, Meta Cloud API, Twilio
- Cada card: nome, status badge (Conectado ✅ / Erro ⚠️ / Desconectado ❌), data/hora última sincronização, botão "Configurar" → drawer
- Drawer de configuração: campos de API key mascarados `"****XXXX"` + botão "Revelar" (abre confirmação "Revelar exporá a chave completa. Confirma?" + registra auditoria); botão "Testar conexão" → spinner → resultado (latência + status)
- Salvar alteração de credencial: exige `update_reason ≥ 20 chars` + registra audit
- Verificação: "Revelar" API key → confirmação → valor completo exibido por 30s + audit registrado; "Testar conexão" ZapSign → exibe `status: ok` ou mensagem de erro

**[FE-06] Implementar T-100 — Usuários e Permissões + Modo Manutenção**
- T-100: Matriz de permissões (4 perfis × 10 módulos) como tabela de checkboxes; read-only para visualização; toggle por célula editável pelo Master; botão "Salvar" exige confirmação: "Esta alteração afetará todos os usuários do perfil [X]."
- ⚠️ AMBÍGUO: RN-118 declara que perfis customizados não existem no MVP — a tela T-100 exibe os 4 perfis fixos sem possibilidade de criar novos; checkboxes de permissão são apenas para visualização de referência (os guards de RBAC são hardcoded no backend)
- Modo manutenção: toggle "Modo Manutenção" na seção Sistema; ao ativar → modal "Ativar modo manutenção encerrará todas as sessões de outros operadores. Confirma?"; ao ativar com sucesso → banner vermelho fixo no topo: "Modo manutenção ativo. Apenas Masters têm acesso."; ao desativar → banner some
- Verificação: ativar modo manutenção → banner vermelho aparece; outro operador logado recebe 503 ao fazer qualquer request

---

## WIRING

**[WIRE-01] Integrar `ConfigsModule` no NestJS**
- `ConfigsController` com todas as rotas; `MASTER` guard em todos os endpoints
- `ConfigsService` injeta `PrismaService`, `RedisService`, `AuditService`, `NotificationService` (para modo manutenção)
- `MaintenanceModeGuard` registrado globalmente como middleware `APP_GUARD` com prioridade antes de `JwtAuthGuard`
- `RedisService.del('rs:configs:all')` chamado após cada PATCH bem-sucedido
- Verificação: `AppModule` importa `ConfigsModule`; `MaintenanceModeGuard` está em `providers` como global guard; alteração de config invalida cache Redis

**[WIRE-02] Integrar frontend Configurações com React Query**
- `useConfigs()`: `useQuery` para `GET /v1/configs`; `staleTime: 0` (configurações sempre frescas)
- `usePatchConfig(key)` mutation: ao confirmar duplo → re-enviar com `confirm_double: true`; após sucesso → invalidar `useConfigs()` + toast
- `useRollbackConfig(key)` mutation: após sucesso → invalidar `useConfigs()` + toast "Parâmetro revertido ao valor anterior."
- Sidebar "Configurações" visível APENAS para MASTER; completamente ausente do DOM para outros perfis
- Verificação: mutation de confirmação dupla implementa fluxo de 2 chamadas sequenciais; invalidação de cache após salvar funciona corretamente

---

## TESTES

**[TEST-01] Testes unitários — `ConfigsService`**
- `patchConfig()`: `update_reason` < 20 chars → `422`; parâmetro crítico sem `confirm_double: true` → `202` com summary; com `confirm_double: true` → `200` + audit registrado + cache invalidado
- `rollbackConfig()`: template → `422`; parâmetro numérico → restaura `previous_value` + audit
- `maintenanceMode ON`: tokens não-Master invalidados; requests não-Master → `503`
- `patchConfig()`: `COMMISSION_CEDENTE_PCT = 3` → `422`; `= 25` → `200`; `case_config_snapshots` existentes inalterados

**[TEST-02] Testes de integração — endpoints de Configurações**
- COORDENADOR → `GET /v1/configs` → 403; MASTER → 200
- `PATCH /v1/configs/COMMISSION_CEDENTE_PCT` sem `confirm_double: true` → 202; com `confirm_double: true` → 200 + audit
- `GET /v1/configs/ZAPSIGN_API_KEY` → valor mascarado; `GET .../reveal` → valor completo + audit registrado
- `POST /v1/configs/MAINTENANCE_MODE_ENABLED` com value=true → request COORDENADOR → 503

**[TEST-03] Testes de componente — Frontend**
- T-095: COORDENADOR não vê item "Configurações" na sidebar; MASTER vê 5 cards
- T-096: fluxo de confirmação dupla renderiza 2 modais sequencialmente; cancelar no 2° → PATCH não enviado
- T-099: "Revelar" API key exibe modal de confirmação antes de revelar; "Testar conexão" exibe spinner

**[TEST-04] Testes E2E — fluxo de alteração crítica**
- Playwright: MASTER → `/configuracoes/comissoes` → alterar `COMMISSION_CEDENTE_PCT` para 22% → motivo → salvar → modal #1 aparece → confirmar → modal #2 aparece com checkbox → marcar checkbox + confirmar → toast "Parâmetro atualizado" → verificar `global_configs` com value=22 → verificar `case_config_snapshots` de caso existente ainda tem 20

---

## AUTO-VERIFICAÇÃO (12 CHECKS)

- [x] **Check #1 — RBAC exclusivo MASTER (RN-109 REQ-024/164):** sidebar oculta para todos os não-Master; `MASTER` guard em todos os endpoints; 403 para qualquer outro perfil
- [x] **Check #2 — Nomes canônicos:** `global_configs`, `case_config_snapshots`, `notification_templates`, `COMMISSION_CEDENTE_PCT`, `COMMISSION_CESSIONARIO_PCT`, `REVERSAL_PERIOD_DAYS`, `MAINTENANCE_MODE_ENABLED` — zero sinônimos
- [x] **Check #3 — Audit trail obrigatório (RN-110 REQ-164):** `update_reason ≥ 20 chars` em todo PATCH; `audit.audit_logs` registrado para cada mutação; histórico por parâmetro disponível
- [x] **Check #4 — Confirmação dupla (RN-112):** 3 parâmetros críticos com `is_critical = true`; fluxo 202 → re-envio com `confirm_double: true` → 200; 2 modais no frontend
- [x] **Check #5 — Rollback numérico (RN-113):** POST rollback disponível para `data_type = 'number'`; templates e integrações retornam 422; audit registrado
- [x] **Check #6 — Alterações prospectivas (RN-111):** PATCH de config NUNCA atualiza `case_config_snapshots` existentes; testado em E2E
- [x] **Check #7 — Versionamento templates ZapSign (RN-124 REQ-173):** INSERT de nova versão, nunca UPDATE; versão anterior permanece imutável; `is_active` toggle; PDF gerado persistido
- [x] **Check #8 — API keys mascaradas:** `is_secret = true` → valor mascarado no GET; reveal → audit `SECRET_REVEALED`; reveal exige confirmação
- [x] **Check #9 — Modo manutenção (RN-119):** ativação invalida tokens não-Master; `MaintenanceModeGuard` global retorna 503; banner vermelho no frontend
- [x] **Check #10 — Cache invalidado:** `rs:configs:all` invalidado após cada PATCH bem-sucedido; agentes e SlaMonitor releem configs em cada ciclo
- [x] **Check #11 — Anti-scaffold R10:** todos os itens têm chaves canônicas de `global_configs`, faixas de valores, condições de erro, audit trail e verificação
- [x] **Check #12 — Cobertura REQs S11:** REQ-024 ✅ REQ-163 ✅ REQ-164 ✅ REQ-165 ✅ REQ-173 ✅ REQ-242 ✅ REQ-243 ✅ REQ-244 ✅ REQ-245 ✅ REQ-246 ✅ REQ-247 ✅

---

*Sprint S11 gerada por Pipeline ShiftLabs v2.3 — 2026-03-24*
