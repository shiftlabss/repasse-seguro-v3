# S6 — Atividades

## CRM Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Sprint** | S6 |
| **Nome** | Atividades |
| **Template** | B — Módulo Fullstack |
| **Docs fonte** | 01.3, 05.3, 06, 16, 27 |
| **REQs cobertos** | REQ-015, REQ-068 a REQ-072, REQ-161 a REQ-162, REQ-205 a REQ-207, REQ-260, REQ-283 (parcial), REQ-310 |
| **Objetivo** | Analista RS registra Atividades (ligação, reunião, WhatsApp, e-mail, nota interna, follow-up), agenda Follow-ups com alertas 30 min antes, linha do tempo consolidada por Caso, limite de retroatividade 30 dias. |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — `activities`, 8 tipos exatos, `ActivityStatus`, `ActivityType`, resultados exatos ("Sem resposta", "Contatado — follow-up necessário", "Contatado — Caso avançou", "Encaminhado para Coordenador")
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: Valores exatos — resumo mínimo 20 chars, retroatividade 30 dias para Analista, 3h após agendamento para Vencido, aviso 30 min antes follow-up, 3 dias úteis vencido → notifica Coordenador RS
- [x] C4: Glossário — Atividade, Follow-up, estados (Registrada/Agendada/Vencida/Concluída) cobertos
- [x] C5: Estados da Atividade: Registrada / Agendada → Concluída / Vencida — todas transições cobertas
- [x] C6: Worker de verificação diária 08h00, worker de follow-up vencido (3h após agendamento), aviso 30 min antes
- [x] C7: RBAC — exclusão de Atividade apenas Admin RS; retroatividade > 30 dias apenas Coord/Admin
- [x] C8: T-CRM-060/061 com estados loading/empty
- [x] C9: Integração com linha do tempo do Caso (append-only)
- [x] C10: Glossário — Atividade vs Follow-up distinção clara
- [x] C11: Anti-scaffold — validação resumo, lógica de retroatividade, lógica de prioridade alta (1 por Caso)
- [x] C12: REQ-015, REQ-068 a REQ-072 cobertos

---

## FEATURE 1 — Registro de Atividades

### Banco / Migrations

- [x] **REQ-310** Verificar na tabela `activities` os valores de `status`: `REGISTRADA`, `AGENDADA`, `VENCIDA`, `CONCLUIDA`. Verificar `type` com os 8 valores: `LIGACAO`, `REUNIAO`, `WHATSAPP`, `EMAIL`, `NOTA_INTERNA`, `FOLLOWUP_AGENDADO`, `DOCUMENTO_ENVIADO`, `EVENTO_SISTEMA`. Adicionar index em `activities.scheduled_at` e `activities.case_id`
  - Validar: enums completos; indexes criados; migration aplicada

### Backend

- [x] **REQ-205** `GET /cases/:id/activities` — Roles: todos internos com escopo. Query params: `type`, `status`, `from_date`, `to_date`, `page`, `per_page`. Retornar atividades ordenadas por `activity_date DESC`. Parceiro Externo → retornar apenas `type = EVENTO_SISTEMA` (transições de estado), sem conteúdo de resumo ou resultado
  - Validar: Parceiro Externo → apenas eventos de sistema; filtro `type=FOLLOWUP_AGENDADO` retorna apenas follow-ups; paginação funciona

- [x] **REQ-206 / REQ-069** `POST /cases/:id/activities` — Roles: todos internos. Request body: `{ type, activity_date, duration_minutes?, channel?, summary (mínimo 20 chars), result?, scheduled_at? (para follow-ups), contact_id? }`. Validações:
  - `summary` com menos de 20 caracteres → 422 `CRM-034`
  - `activity_date` no futuro (exceto para `FOLLOWUP_AGENDADO`) → 422 `CRM-035`
  - Retroatividade: `activity_date` mais de 30 dias no passado por Analista RS → 422 `CRM-036`: "Atividades retroativas além de 30 dias devem ser registradas pelo Coordenador RS ou Admin RS."
  - Retroatividade > 30 dias por Coordenador/Admin → exigir justificativa (`justification: string, mínimo 20 chars`) no body
  - `FOLLOWUP_AGENDADO`: `scheduled_at` obrigatório (data futura)
  - Criar atividade com `status = REGISTRADA` (ou `AGENDADA` para follow-ups)
  - Criar `audit_log`
  - Validar: Analista RS com retroatividade de 45 dias → 422 CRM-036; Coord com 45 dias + justificativa → 201; follow-up sem `scheduled_at` → 422; resumo < 20 chars → 422 CRM-034

- [x] **REQ-207** `PATCH /cases/:id/activities/:activityId` — Roles: criador da atividade (para edição dentro de 1h), `COORDENADOR_RS`, `ADMIN_RS`. Apenas `ADMIN_RS` pode excluir (`DELETE`) e apenas em caso de duplicata comprovada + justificativa obrigatória. Campos editáveis: `summary`, `result`, `channel`, `duration_minutes`
  - Validar: Analista RS não consegue editar atividade de outro; DELETE por Analista RS → 403; DELETE por Admin RS sem justificativa → 422; DELETE válido → soft delete + audit_log

- [x] **REQ-072** Implementar validação de retroatividade em `ActivitiesService`:
  - Calcular `days_diff = now() - activity_date` em dias
  - Se `days_diff > 30` e role = `ANALISTA_RS` → lançar `CRM-036`
  - Se `days_diff > 30` e role = `COORDENADOR_RS` ou `ADMIN_RS` → exigir `justification` no body; registrar justificativa na atividade como `notes`
  - Validar: 30 dias exatos → permitido para Analista RS; 31 dias → bloqueado para Analista RS; 31 dias com justificativa por Coord → permitido

---

## FEATURE 2 — Follow-ups e Agenda

### Backend

- [x] **REQ-070 / REQ-100** Implementar criação de Follow-up:
  - `POST /cases/:id/activities` com `type = FOLLOWUP_AGENDADO` e `scheduled_at` (futuro obrigatório)
  - Ao criar: agendar job de notificação push+e-mail para `scheduled_at - 30 minutos` via fila RabbitMQ (`followup_reminder` queue com delay)
  - Validar prioridade alta: se `is_high_priority = true`, verificar que Caso não tem outro Follow-up `is_high_priority = true` pendente → 422 `CRM-037` se já existe
  - `status = AGENDADA`
  - Validar: follow-up criado sem `scheduled_at` → 422; follow-up com `scheduled_at` no passado → 422; 2 follow-ups alta prioridade no mesmo Caso → 422 CRM-037

- [x] **REQ-015** Implementar `FollowupVencidoWorker` como parte do `SlaCheckerWorker` (cron 08h00 America/Fortaleza): buscar `activities` com `type = FOLLOWUP_AGENDADO`, `status = AGENDADA`, `scheduled_at < now()`. Para cada um:
  - Marcar `status = VENCIDA`
  - Exibir badge "Atrasado" no Caso (via Supabase Realtime)
  - Notificar Analista RS: "Você tem [N] follow-up(s) vencido(s). Acesse sua fila para atualizar."
  - Se `scheduled_at < now() - 3 dias úteis`: notificar também Coordenador RS
  - Validar: follow-up com `scheduled_at = ontem` → status VENCIDA após cron; Analista RS notificado; após 3 dias úteis → Coordenador RS notificado

- [x] Implementar endpoint `POST /cases/:id/activities/:activityId/complete` — Marcar Follow-up como `status = CONCLUIDA`. Validar que `type = FOLLOWUP_AGENDADO` e `status = AGENDADA` ou `VENCIDA`. Registrar `completed_at = now()`
  - Validar: marcar atividade não-follow-up como concluída → 422; follow-up AGENDADO → CONCLUIDA + completed_at preenchido

- [x] **REQ-071** Implementar linha do tempo do Caso em `ActivitiesService.getCaseTimeline(caseId)`: consolidar cronologicamente:
  - Transições de estado (`case_status_history`) → `type: EVENTO_SISTEMA`
  - Todas as atividades registradas (todos os 8 tipos)
  - Uploads no Dossiê (`dossier_documents.created_at`)
  - Propostas e contrapropostas (`proposals.created_at`)
  - Alertas de SLA gerados (`sla_alerts.triggered_at`)
  - Retornar como array ordenado por data DESC. Somente leitura — nenhuma modificação possível
  - Validar: linha do tempo inclui entradas dos 5 tipos; Parceiro Externo → apenas transições de estado (type EVENTO_SISTEMA, sem resumo)

### Frontend

- [x] **REQ-161** Implementar `app/(dashboard)/activities/page.tsx` — T-CRM-060 Agenda de Follow-ups:
  - **Visão diária e semanal**: calendário com follow-ups agendados do Analista RS logado
  - Lista de Follow-ups vencidos com badge "Atrasado" + contagem no topo
  - Lista de Follow-ups do dia com horários
  - Botão "Novo Follow-up" → abre T-CRM-061 Modal
  - Botão "Concluir" em cada follow-up vencido/agendado
  - Empty state do dia: "Sem follow-ups para hoje. Ótimo trabalho!"
  - Validar: follow-up vencido aparece na lista de vencidos com badge "Atrasado"; clique em "Concluir" → status CONCLUIDA + remove da lista de pendentes; visão semanal exibe follow-ups distribuídos por dia

- [x] **REQ-162** Implementar `ActivityModal` (T-CRM-061) para registro e agendamento:
  - Modo "Registrar Atividade": campos `type (select com 8 opções)`, `activity_date (date + time, default: agora)`, `duration_minutes (apenas para Ligação e Reunião)`, `channel`, `summary (textarea, contagem de chars, mínimo 20)`, `result (select: "Sem resposta" | "Contatado — follow-up necessário" | "Contatado — Caso avançou" | "Encaminhado para Coordenador")`
  - Modo "Agendar Follow-up": campos `scheduled_at (datepicker com horário)`, `contact_target (select Contato do Caso)`, `description/summary`, `is_high_priority (toggle)`
  - Retroatividade: ao selecionar data > 30 dias atrás por Analista RS → exibir aviso vermelho "Data além do limite de 30 dias. Solicite ao Coordenador RS que registre esta atividade." e bloquear submit
  - Validar: `summary` com < 20 chars → contador em vermelho + botão submit desabilitado; retroatividade de 31 dias para Analista RS → aviso + submit bloqueado; tipo "Nota interna" não tem campo `result` (apenas notas internas não disparam notificação)

- [x] **REQ-071** Implementar linha do tempo visual em `components/cases/CaseTimeline.tsx`:
  - Timeline vertical cronológica com ícones por tipo de evento
  - Cada entrada: ícone tipo, data/hora (formatada America/Fortaleza), usuário responsável, resumo/descrição
  - Filtros: tipo de evento (estado/atividade/documento/proposta/SLA)
  - Scroll infinito (load mais ao chegar no fim da lista)
  - Somente leitura — sem edição possível a partir da timeline
  - Validar: linha do tempo integrada na tab "Histórico" de T-CRM-022; Parceiro Externo não vê resumos de Atividades; entradas de Dossiê exibem nome do documento; alertas SLA exibidos com badge colorido

---

## 🧪 Testes

### Backend — Unitário (Jest)

- [x] **REQ-260** `ActivitiesService` — cobertura 70%:
  - `should reject summary with less than 20 characters`
  - `should block retroactivity beyond 30 days for ANALISTA_RS`
  - `should allow retroactivity beyond 30 days for COORDENADOR_RS with justification`
  - `should set status AGENDADA for follow-up type`
  - `should reject follow-up without scheduled_at`
  - `should reject second high-priority follow-up for same case`
  - `should mark follow-up as VENCIDA after deadline`
  - `should block delete by ANALISTA_RS`

### Backend — Integração

- [x] `POST /cases/:id/activities` com summary < 20 chars → 422 CRM-034
- [x] `POST /cases/:id/activities` com data 45 dias atrás por Analista RS → 422 CRM-036
- [x] `POST /cases/:id/activities/:id/complete` em atividade não-follow-up → 422
- [x] `GET /cases/:id/activities` por Parceiro Externo → apenas eventos de sistema

---

## 🔀 Cross-Módulo

- A linha do tempo (`getCaseTimeline`) consolidada aqui é renderizada em T-CRM-022 (S3) na tab "Histórico". A tab estava em placeholder até S6.
- O worker `FollowupVencidoWorker` (parte do `SlaCheckerWorker` de S3) é atualizado aqui para processar também Follow-ups vencidos.
- Alertas de SLA vencido (S3/S8) aparecem na linha do tempo como entradas `EVENTO_SISTEMA`.
