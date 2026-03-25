# S3 — Supervisão de Interações

## Metadados

| Campo         | Valor                                                                           |
| ------------- | ------------------------------------------------------------------------------- |
| Sprint        | S3                                                                              |
| Nome          | Supervisão de Interações                                                        |
| Template      | B (Módulo Fullstack por Feature)                                                |
| Módulo        | AI-Dani-Admin                                                                   |
| Docs Fonte    | D05 (RF-001 a RF-010, RF-026), D06, D07, D08, D09, D10, D12, D13, D16, D20, D25 |
| REQs Cobertos | REQs de S3 no registro-mestre.md                                                |
| Data          | 2026-03-24                                                                      |

## Objetivo

Implementar o módulo de Supervisão de Interações completo (fullstack): listagem paginada de interações com filtros (RF-001, RF-002), detalhe de interação (RF-003), endpoint interno para criação de interações (RF-010), tela T-001 (Lista de Interações), tela T-002 (Detalhe de Interação), subscription Supabase Realtime para atualizações em tempo real, e todos os testes associados.

---

## FEATURE 1 — Listagem de Interações (RF-001, RF-002)

### BANCO (Feature 1)

- [x] Confirmar que índices `(agent_id, created_at DESC)`, `(status, created_at DESC)`, `(user_id, created_at DESC)` existem em `interactions` (criados em S1)
- [x] Confirmar que soft delete (`deleted_at IS NULL`) está no padrão de todas as queries de listagem

### BACKEND (Feature 1)

- [x] Criar `apps/api/src/modules/supervision/supervision.module.ts` com imports: `PrismaModule`, `AuditLogModule`
- [x] Criar `apps/api/src/modules/supervision/supervision.service.ts`
- [x] Criar `apps/api/src/modules/supervision/supervision.controller.ts`
- [x] Criar `apps/api/src/modules/supervision/dto/list-interactions.dto.ts`:
  - `cursor?: string` — UUID da última interação da página anterior
  - `limit?: number` — default 20, máx 100 (D16)
  - `agentType?: 'DANI_CESSIONARIO' | 'DANI_CEDENTE'`
  - `status?: InteractionStatus` — enum: `SINALIZADA_PARA_REVISAO`, `EM_TAKEOVER`, `RESPONDIDA_PELA_IA`, `ENCERRADA`
  - `startDate?: string` — ISO8601
  - `endDate?: string` — ISO8601
  - `minConfidence?: number` — 0–100
  - `maxConfidence?: number` — 0–100
- [x] Implementar `SupervisionService.listInteractions(dto: ListInteractionsDto)`:
  - Query com cursor-based pagination: `WHERE id > :cursor ORDER BY created_at DESC LIMIT :limit`
  - Aplica todos os filtros do DTO: `agentType`, `status`, `startDate`, `endDate`, `minConfidence`, `maxConfidence`
  - Exclui soft-deleted: `WHERE deleted_at IS NULL`
  - Retorna `{ data: Interaction[], nextCursor: string | null, total: number }`
  - `nextCursor` é o `id` do último item retornado; `null` se não há próxima página
- [x] Implementar `GET /api/v1/admin/interactions` no `SupervisionController`:
  - Guards: `@UseGuards(JwtAuthGuard, RolesGuard)`, `@Roles('ADMIN')`
  - Valida DTO com `class-validator`
  - Retorna `{ data: InteractionSummary[], nextCursor: string | null, total: number }`
  - `InteractionSummary`: `id`, `agentType`, `status`, `confidenceScore`, `latencyMs`, `createdAt`, `userId` (anonimizado — apenas prefixo UUID)
  - Em erro de DB → HTTP 500 com code `DA-SUP-001`

### FRONTEND (Feature 1)

- [x] Criar `apps/web/src/components/app-sidebar.tsx` — AppSidebar (256px) com labels exatos conforme D08 seção 3.3 / REQ-143 — [CORRIGIDO: FINDING-004]:
  - Item "Supervisão IA" → link `/admin/supervisao-ia/interacoes`
  - Item "Interações" → link `/admin/supervisao-ia/interacoes`
  - Item "Métricas" → link `/admin/supervisao-ia/metricas`
  - Item "Configurações" → link `/admin/supervisao-ia/configuracoes`
  - Item "Auditoria" → link `/admin/supervisao-ia/auditoria`
  - Item "Checklist de Lançamento" → link `/admin/supervisao-ia/checklist-lancamento`
  - Rotas frontend alinhadas com D06 (`/admin/supervisao-ia/` como prefixo) — corrige FINDING-007
- [x] Criar `apps/web/src/app/admin/interactions/page.tsx` — tela T-001 (Lista de Interações) conforme D06/D07:
  - Layout: AppSidebar (256px) + conteúdo principal
  - Componentes shadcn/ui: `Table`, `TableHeader`, `TableBody`, `TableRow`, `TableCell` (D09)
  - Coluna "Agente": exibe `DANI_CESSIONARIO` → "Dani Cessionário" / `DANI_CEDENTE` → "Dani Cedente" (D10)
  - Coluna "Status": badge colorido por status (4 variantes conforme D09 — `InteractionRow` styles)
    - `SINALIZADA_PARA_REVISAO` → badge amarelo "Aguardando revisão"
    - `EM_TAKEOVER` → badge azul "Em atendimento humano"
    - `RESPONDIDA_PELA_IA` → badge verde "Respondida pela IA"
    - `ENCERRADA` → badge cinza "Encerrada"
  - Coluna "Confiança": exibe `confidenceScore` + "%" ou "—" se null (D10)
  - Coluna "Latência": converte `latencyMs` para segundos com 1 casa decimal (D10: `latency_ms → latency_seconds`)
  - Coluna "Data": formato `dd/MM/yyyy HH:mm` (timezone America/Fortaleza)
  - Paginação: botão "Carregar mais" que usa `nextCursor`
  - Loading state: skeleton table (shadcn/ui `Skeleton`) — texto "Carregando interações..." (D08)
  - Empty state quando lista vazia: ícone + título "Nenhuma interação encontrada" + subtítulo "Não há interações registradas para os filtros selecionados." (D08)
- [x] Criar `apps/web/src/app/admin/interactions/_components/interaction-filters.tsx`:
  - Filtros: `Select` para agentType, `Select` para status, `DatePicker` para intervalo, inputs numéricos para confiança mín/máx
  - Ao alterar qualquer filtro, refaz a query (reset cursor)
  - Texto do filtro de status usa os termos aprovados do D08: "Aguardando revisão", "Em atendimento humano", "Respondida pela IA", "Encerrada"
- [x] Criar `apps/web/src/app/admin/interactions/_components/interaction-row.tsx`:
  - Clique na row navega para `/admin/interactions/:id`
  - Aplica `InteractionRow` interface do D09: `{ id, agentType, status, confidenceScore, latencyMs, createdAt }`
  - 4 row styles por status (D09): `sinalizada` (amarelo), `emTakeover` (azul), `respondida` (verde), `encerrada` (cinza)

### WIRING (Feature 1)

- [x] Verificar que `SupervisionModule` está importado no `AppModule`
- [x] Verificar que `GET /api/v1/admin/interactions` retorna dados reais do banco (com seed)
- [x] Verificar que filtro `status=SINALIZADA_PARA_REVISAO` filtra corretamente
- [x] Verificar que paginação `cursor` funciona: segunda página retorna itens diferentes da primeira
- [x] Verificar que `apps/web` exibe a tabela corretamente com dados da API

### TESTES (Feature 1)

- [x] Criar `supervision.service.spec.ts` — Vitest:
  - Test: `listInteractions({})` retorna primeiros 20 itens ordenados por `created_at DESC`
  - Test: `listInteractions({ limit: 5 })` retorna no máximo 5 itens
  - Test: `listInteractions({ limit: 200 })` limita a 100 (máximo)
  - Test: `listInteractions({ status: 'SINALIZADA_PARA_REVISAO' })` filtra por status
  - Test: `listInteractions({ cursor: '<uuid>' })` retorna itens após o cursor
  - Test: registros com `deleted_at != null` NÃO aparecem na listagem
  - Test: `nextCursor` é `null` quando não há mais itens

---

## FEATURE 2 — Detalhe de Interação (RF-003)

### BANCO (Feature 2)

- [x] Confirmar que índice em `session_id` existe em `interactions`
- [x] Confirmar que campos `langfuse_trace_id`, `model_used`, `tokens_used`, `cost_usd` são retornados no detalhe

### BACKEND (Feature 2)

- [x] Criar `apps/api/src/modules/supervision/dto/interaction-detail.dto.ts`:
  - Response: todos os campos de `interactions` + lista de `takeovers` associados
- [x] Implementar `SupervisionService.getInteractionById(id: string, adminId: string)`:
  - Busca `interaction` por `id` onde `deleted_at IS NULL`
  - Inclui `takeovers` associados (via `interaction_id`)
  - Verifica que `id` é UUID válido — senão HTTP 400 com code `DA-SUP-002`
  - Se não encontrado → HTTP 404 com code `DA-SUP-003`, mensagem `'Interação não encontrada'`
  - Registra acesso em `admin_access_logs`: `{ action: 'VIEW_INTERACTION', targetType: 'interaction', targetId: id }`
- [x] Implementar `GET /api/v1/admin/interactions/:id` no `SupervisionController`:
  - Guards: `@Roles('ADMIN')`
  - Extrai `adminId` do JWT payload
  - Chama `SupervisionService.getInteractionById(id, adminId)`

### FRONTEND (Feature 2)

- [x] Criar `apps/web/src/app/admin/interactions/[id]/page.tsx` — tela T-002 (Detalhe de Interação) conforme D06/D07:
  - Header: breadcrumb "Interações > Detalhe" + badge de status
  - Card de metadados: agentType, confidenceScore ("Confiança"), latencySeconds ("Latência"), model_used, tokens_used, cost_usd, langfuse_trace_id (link para Langfuse se não null)
  - Chat transcript: mensagens do usuário (alinhadas à direita) e respostas do agente (alinhadas à esquerda)
  - Se `status === 'SINALIZADA_PARA_REVISAO'`: exibe botão "Assumir conversa" (D10 — `takeover` → "Assumir conversa")
  - Se `status === 'EM_TAKEOVER'`: exibe separador "Atendimento humano" (D10 — COMP-002)
  - Histórico de takeovers: lista de registros da tabela `takeovers` com `adminId`, `startedAt`, `endedAt`
  - Loading state: skeleton card — texto "Carregando detalhes..." (D08)
  - Erro 404: mensagem "Interação não encontrada" (D08)
- [x] Criar `apps/web/src/app/admin/interactions/[id]/_components/interaction-metadata-card.tsx`:
  - Exibe todos os campos de metadados com labels corretos (D10: "Confiança", "Latência", etc.)
  - `confidenceScore` null → exibe "—"
  - `langfuse_trace_id` não null → link externo para Langfuse
- [x] Criar `apps/web/src/app/admin/interactions/[id]/_components/chat-transcript.tsx`:
  - Renderiza `user_message` e `agent_response` em bolhas de chat
  - Separador visual "Atendimento humano" (COMP-002) inserido quando `status === 'EM_TAKEOVER'`

### WIRING (Feature 2)

- [x] Verificar que `GET /api/v1/admin/interactions/:id` retorna interação com takeovers
- [x] Verificar que clicar em uma row na T-001 navega para T-002 com os dados corretos
- [x] Verificar que acesso é registrado em `admin_access_logs`

### TESTES (Feature 2)

- [x] Continuar em `supervision.service.spec.ts`:
  - Test: `getInteractionById(id)` retorna interação com array `takeovers`
  - Test: `getInteractionById('invalid-uuid')` lança `BadRequestException` com code `DA-SUP-002`
  - Test: `getInteractionById('<inexistente>')` lança `NotFoundException` com code `DA-SUP-003`
  - Test: acesso registra log em `admin_access_logs` com `action: 'VIEW_INTERACTION'`
  - Test: interação com `deleted_at != null` → NotFoundException (não expõe deletados)

---

## FEATURE 3 — Endpoint Interno de Criação de Interação (RF-010)

### BANCO (Feature 3)

- [x] Confirmar que `InteractionStatus` enum tem valor `SINALIZADA_PARA_REVISAO` como default em `interactions.status`

### BACKEND (Feature 3)

- [x] Criar `apps/api/src/modules/supervision/dto/create-interaction.dto.ts`:
  - `agentId: string` — UUID obrigatório
  - `agentType: AgentType` — obrigatório
  - `userId: string` — UUID obrigatório
  - `sessionId: string` — UUID obrigatório
  - `userMessage: string` — obrigatório
  - `agentResponse?: string`
  - `confidenceScore?: number` — 0–100
  - `latencyMs?: number`
  - `modelUsed?: string`
  - `tokensUsed?: number`
  - `costUsd?: number`
  - `langfuseTraceId?: string`
- [x] Implementar `SupervisionService.createInteraction(dto: CreateInteractionDto)`:
  - Cria registro em `interactions` com `status` calculado:
    - Se `confidenceScore >= threshold` (lê `confidence_threshold` do `agent_configurations` do `agentId`) → `RESPONDIDA_PELA_IA`
    - Se `confidenceScore < threshold` → `SINALIZADA_PARA_REVISAO` (requer revisão humana — RN-DA-032)
    - Se `confidenceScore` null → `SINALIZADA_PARA_REVISAO` (precaução)
  - Retorna interação criada
- [x] Implementar `POST /api/v1/internal/interactions` no `SupervisionController`:
  - Protegido por `InternalApiKeyGuard` (não JWT) — `@Public()` + `@UseGuards(InternalApiKeyGuard)`
  - Valida DTO
  - Retorna `{ id, status, confidenceScore }` — HTTP 201

### FRONTEND (Feature 3)

> Endpoint interno — sem UI de criação. Frontend passivo (recebe via Realtime).

### WIRING (Feature 3)

- [x] Verificar que `POST /api/v1/internal/interactions` com `X-Internal-Api-Key` correto cria interação no banco
- [x] Verificar que `status` é `RESPONDIDA_PELA_IA` quando `confidenceScore=90` e threshold=80
- [x] Verificar que `status` é `SINALIZADA_PARA_REVISAO` quando `confidenceScore=70` e threshold=80

### TESTES (Feature 3)

- [x] Continuar em `supervision.service.spec.ts`:
  - Test: `createInteraction(dto)` com `confidenceScore=90`, `threshold=80` → `status = 'RESPONDIDA_PELA_IA'`
  - Test: `createInteraction(dto)` com `confidenceScore=70`, `threshold=80` → `status = 'SINALIZADA_PARA_REVISAO'`
  - Test: `createInteraction(dto)` sem `confidenceScore` → `status = 'SINALIZADA_PARA_REVISAO'`
  - Test: `agentId` inválido → `BadRequestException`

---

## FEATURE 4 — Supabase Realtime (atualizações em tempo real)

### BACKEND (Feature 4)

- [x] Verificar que Supabase Realtime está habilitado para a tabela `interactions` no projeto Supabase (D02)
- [x] Confirmar que policy de RLS no Supabase permite que o canal Realtime envie eventos INSERT/UPDATE para `interactions`

### FRONTEND (Feature 4)

- [x] Criar `apps/web/src/hooks/use-interactions-realtime.ts`:
  - Usa `@supabase/ssr` para criar subscription ao canal `postgres_changes` na tabela `interactions`
  - Ao receber evento `INSERT` ou `UPDATE`: chama `mutate()` do SWR/React Query para revalidar a lista em T-001
  - Ao receber evento de interação com `status = 'SINALIZADA_PARA_REVISAO'`: exibe toast de alerta "Nova interação aguardando revisão" (D08 — texto aprovado)
  - Cleanup: cancela subscription no `useEffect` cleanup
- [x] Aplicar hook `useInteractionsRealtime()` na página `apps/web/src/app/admin/interactions/page.tsx`

### WIRING (Feature 4)

- [x] Verificar que ao criar interação via `POST /api/v1/internal/interactions`, a T-001 atualiza automaticamente sem refresh
- [x] Verificar que toast aparece quando nova interação `SINALIZADA_PARA_REVISAO` é criada

### TESTES (Feature 4)

- [x] Criar `apps/web/src/hooks/use-interactions-realtime.spec.ts`:
  - Test: hook subscreve ao canal correto da tabela `interactions`
  - Test: ao receber evento INSERT, chama `mutate()` para revalidar lista
  - Test: cleanup cancela subscription ao desmontar

---

## 🔀 Cross-Módulo

- [x] `SupervisionService.createInteraction` lê `confidence_threshold` de `agent_configurations` — dependência de S5 (AgentConfig). Em S3, mockar com valor hardcoded `80` e adicionar TODO para substituir pela query real após S5.
- [x] `AuditLogService.log` chamado em `getInteractionById` — depende do S2 (AuditLogModule). Verificar que `AuditLogModule` está importado em `SupervisionModule`.

---

## AUTO-VERIFICAÇÃO S3

| Check                 | Critério                                                                                                                                                 | Status |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura       | `interactions`, `InteractionStatus`, `SINALIZADA_PARA_REVISAO`, `EM_TAKEOVER`, `RESPONDIDA_PELA_IA`, `ENCERRADA`, `DA-SUP-001/002/003` usados exatamente | ☑      |
| #2 Verificabilidade   | Cada item é binariamente verificável                                                                                                                     | ☑      |
| #3 Valores numéricos  | Paginação default 20, max 100; threshold default 80                                                                                                      | ☑      |
| #4 N itens completos  | 4 features cobertas: listagem, detalhe, criação interna, Realtime                                                                                        | ☑      |
| #5 Máquinas de estado | InteractionStatus: 4 transições documentadas (SINALIZADA→EM_TAKEOVER→ENCERRADA, RESPONDIDA_PELA_IA→ENCERRADA)                                            | ☑      |
| #6 Schedules/TTLs     | N/A em S3 (cleanup de 90 dias implementado em S1)                                                                                                        | ☑      |
| #7 Conflitos          | Nenhum conflito novo em S3                                                                                                                               | ☑      |
| #8 Ambiguidades       | Nenhuma ambiguidade nova em S3                                                                                                                           | ☑      |
| #9 Anti-scaffold      | `SupervisionService.listInteractions` tem lógica completa: cursor, filtros, soft delete, total                                                           | ☑      |
| #10 Cross-módulo      | Dependência em `agent_configurations.confidence_threshold` (S5) documentada com TODO                                                                     | ☑      |
| #11 IDs de referência | RF-001, RF-002, RF-003, RF-010, D06, D07, D08, D09 citados nos itens relevantes                                                                          | ☑      |
| #12 Cobertura REQs S3 | Todos os REQs de supervisão no registro-mestre.md têm ≥ 1 item                                                                                           | ☑      |
