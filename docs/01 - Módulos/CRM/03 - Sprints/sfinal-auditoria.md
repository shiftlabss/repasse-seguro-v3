# S-Final — Auditoria e Sprint de Correções

## CRM Repasse Seguro

| **Campo**         | **Valor**                                                                                                                                                                                                                                                                                              |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sprint**        | S-Final                                                                                                                                                                                                                                                                                                |
| **Nome**          | Auditoria e Sprint de Correções                                                                                                                                                                                                                                                                        |
| **Template**      | Sprint Final (7 checks)                                                                                                                                                                                                                                                                                |
| **Docs fonte**    | Todos (01.1 a 01.5, 02, 05.1, 06, 10, 12, 14, 16, 19, 20, 27)                                                                                                                                                                                                                                          |
| **REQs cobertos** | Todos os REQ-001 a REQ-312 (verificação de cobertura completa)                                                                                                                                                                                                                                         |
| **Objetivo**      | Cruzar o Registro Mestre completo contra a implementação das sprints S1–S10, identificar gaps, verificar completude do Glossário, validar matriz RBAC, confirmar máquinas de estado, verificar fallbacks de integração, e validar todos os requisitos LGPD. GATE: cobertura 100% + zero P0/P1 abertos. |

---

## Check 1 — Cobertura 100% do Registro Mestre

### Verificação por sprint

- [x] **S1 — Fundação**: Verificar que todos os REQs atribuídos à S1 têm pelo menos 1 item implementado e verificável:
  - REQ-001 a REQ-010 (DB schema, enums, RLS, migrations)
  - REQ-011 a REQ-014 (Supabase Storage, seed, configurações iniciais)
  - REQ-191 a REQ-204 (NestJS setup, Next.js setup, layout shell, CI/CD)
  - Validar: 16 tabelas criadas com campos exatos; 6 enums completos; seed com 1 Admin RS + 14 parâmetros + 10 templates; layout shell h-14 + w-64 operacional

- [x] **S2 — Auth**: Verificar REQ-015 (parcial), REQ-016 a REQ-030 cobertos:
  - 5 endpoints de auth implementados (`/auth/login`, `/auth/logout`, `/auth/forgot-password`, `/auth/reset-password`, `/auth/refresh`)
  - Bloqueio 5 tentativas/15 min/30 min
  - Sessão 60 min + aviso 5 min
  - 4 roles como custom claims JWT
  - HttpOnly cookie
  - Guards `JwtAuthGuard` + `RolesGuard` + `RlsDataScopeMiddleware`
  - T-CRM-001 (4 estados), T-CRM-002, T-CRM-003, T-CRM-113 (placeholder)

- [x] **S3 — Pipeline de Casos**: Verificar REQ-031 a REQ-057 cobertos:
  - Função `generate_case_number(year)` → formato RS-YYYY-NNNN
  - 13 transições de estado documentadas e implementadas
  - SlaCheckerWorker cron 08h00 America/Fortaleza (4 níveis base)
  - CaseArchivingWorker mensal (90 dias)
  - Redistribuição com limite 30 Casos
  - Rollback com justificativa 20 chars
  - T-CRM-020/021/022 com kanban/lista/detalhe
  - Optimistic Locking com campo `version`

- [x] **S4 — Negociação e Comissão**: Verificar REQ-058 a REQ-068 cobertos:
  - Enums `NegotiationState` (6 valores) e `CommissionState` (4 valores)
  - Fórmula Cessionário: 20%×Δ (se Δ>0) ou 20%×Valor Pago (se Δ≤0)
  - Fórmula Cedente: 0 para Cenário A; 20%×(recuperado−distrato) para B/C/D
  - Tabela de desconto: ≤R$200k:0%, R$200k–500k:5%, R$500k–1M:10%, >R$1M:15%
  - 3 critérios de fechamento (cumulativos)
  - Cancelamento comissão quatro-olhos + justificativa 100 chars
  - ProposalExpirationWorker (cron 5 min, expiração 48h)
  - T-CRM-030/031/032/033/034

- [x] **S5 — Dossiê e Contatos**: Verificar REQ-098 a REQ-138 cobertos:
  - 11 documentos obrigatórios do Dossiê (nomes exatos)
  - Validação 20MB/PDF+JPG+PNG
  - Gate VERIFICACAO→PUBLICACAO (Dossiê APROVADO + Tabela Atual ≤30 dias)
  - Share link 7 dias (sem dados pessoais do Cedente)
  - 5 roles de Contato exatos
  - Merge de Contatos (primário = mais Casos; secundário marcado MESCLADO, nunca deletado)
  - Arquivamento mensal (12 meses inativo)
  - T-CRM-040 a T-CRM-053

- [x] **S6 — Atividades**: Verificar REQ-068 a REQ-072, REQ-161/162, REQ-205 a REQ-207, REQ-260, REQ-310 cobertos:
  - 8 tipos de atividade exatos
  - Validação resumo mínimo 20 chars
  - Retroatividade 30 dias para Analista RS; justificativa para Coord/Admin
  - Follow-up com aviso 30 min antes (RabbitMQ + delay)
  - FollowupVencidoWorker integrado ao SlaCheckerWorker
  - Escalonamento 3 dias úteis para Coordenador RS
  - Linha do tempo consolidada (5 tipos de entrada)
  - T-CRM-060/061

- [x] **S7 — Comunicação e Integrações**: Verificar REQ-073 a REQ-076, REQ-097 a REQ-107, REQ-139 a REQ-144 cobertos:
  - WhatsApp 24h window enforcement
  - Opt-out LGPD bloqueando envio
  - HMAC-SHA256 em webhooks WhatsApp e ZapSign
  - 10 templates com 3 categorias (Utilitária/Autenticação/Marketing)
  - Opt-in obrigatório para Marketing
  - ZapSign: 3 signatários, lembretes D+2/D+4/D+5, falha 3× → alerta Admin RS
  - Celcoin: webhook HMAC-SHA256 + aprovação manual Coord RS
  - Platform RS: defasagem máx 2 min, retry protocol
  - T-CRM-070/071

- [x] **S8 — SLA, Dashboard e Notificações**: Verificar REQ-077 a REQ-082, REQ-088 a REQ-090, REQ-109, REQ-145, REQ-165/166, REQ-223/224, REQ-243/244, REQ-255, REQ-274, REQ-292, REQ-306/307/308 cobertos:
  - 4 níveis de alerta SLA (AMARELO/LARANJA/VERMELHO/CRITICO)
  - 7 métricas do Dashboard Executivo
  - Aba Financeiro exclusiva Admin RS
  - WeeklyReportWorker domingo 20h00
  - SSE com heartbeat 30s e reconexão exponential backoff
  - Supabase Realtime subscriptions por canal de usuário
  - Defasagem ≤5 min no Dashboard (polling 5 min)
  - T-CRM-080/081

- [x] **S9 — Equipe, Relatórios e Configurações**: Verificar REQ-083 a REQ-096, REQ-167 a REQ-179, REQ-186 a REQ-190, REQ-225 a REQ-230, REQ-258, REQ-279, REQ-284 cobertos:
  - Ciclo de vida do usuário (ATIVO/SUSPENSO/DESLIGADO) com transições corretas
  - Ban 180 dias para reuso de e-mail após desligamento
  - Convite interno 48h; parceiro 7 dias
  - Quatro-olhos para 4 parâmetros críticos
  - 14 parâmetros de sistema com valores/range exatos
  - 10 relatórios com RBAC correto (relatórios 5 e 10 exclusivos Admin RS)
  - Export CSV (BOM UTF-8) e XLSX (`exceljs`)
  - T-CRM-090, T-CRM-100, T-CRM-110, T-CRM-112, T-CRM-113

- [x] **S10 — Qualidade**: Verificar REQ-252, REQ-261/262, REQ-285 a REQ-299 cobertos:
  - 5 fluxos E2E implementados e passando
  - Cobertura unit ≥ threshold por módulo
  - Cobertura integração ≥ 60%
  - API < 200ms p95
  - WCAG 2.1 AA zero violações AA em 5 telas
  - Tablet 768px: 4 funcionalidades críticas
  - OWASP Top 10 validado (A01, A02, A03, A05, A07)

---

## Check 2 — Completude do Glossário (Doc 10)

- [x] Verificar que os 52 termos do Glossário Técnico (Doc 10) estão implementados ou explicitamente documentados no código. Termos de risco (mais propensos a gap):
  - **`version Int` (Optimistic Locking)**: campo `version` presente em `cases`, `negotiations`, `commissions` → incrementado a cada PATCH → 409 `CRM-023` se versão divergir
  - **SSE (Server-Sent Events)**: `GET /notifications/stream` implementado com `EventSource` no frontend e reconexão automática
  - **JWT em HttpOnly cookie**: verificar que `Set-Cookie` tem flags `HttpOnly; Secure; SameSite=Strict` em `POST /auth/login`
  - **Refresh Token auto-rotation**: `POST /auth/refresh` rotaciona token + invalida token anterior
  - **Turborepo**: `turbo.json` com pipelines `build`, `test`, `lint`; workspaces `apps/api`, `apps/web-crm`, `packages/*`
  - **Soft Delete middleware**: Prisma middleware ou extension que seta `deleted_at = NOW()` em vez de `DELETE` para entidades com `deleted_at`
  - **k-anonymity**: relatório de inteligência (Dani-Admin/Dani-Cessionário) aplica k-anonymity ≥ 5 antes de expor dados agregados
  - **DLQ (Dead Letter Queue)**: filas RabbitMQ com DLQ configurada para jobs de follow-up, ZapSign, Celcoin, Platform RS
  - **Parceiro Externo**: `GET /cases/:id/activities` filtra para apenas `type = EVENTO_SISTEMA` sem resumo; `GET /cases/:id/timeline` sem resumos de atividades
  - **MESCLADO**: Contato secundário em merge marcado com status `MESCLADO` e nunca deletado do banco
  - Validar: cada termo acima tem implementação verificável no código ou checklist marcado; ausências marcadas como ⚠️ GAP

---

## Check 3 — Matriz RBAC Completa

- [x] Verificar matrix RBAC completa cruzando todos os endpoints com os 4 roles. Gaps identificados devem ser corrigidos antes de avançar:

  | Endpoint / Ação                                 | ADMIN_RS                         | COORD_RS           | ANALISTA_RS    | PARCEIRO_EXT         |
  | ----------------------------------------------- | -------------------------------- | ------------------ | -------------- | -------------------- |
  | `GET /cases`                                    | ✅ todos                         | ✅ todos da equipe | ✅ atribuídos  | ✅ vinculados        |
  | `POST /cases`                                   | ✅                               | ✅                 | ✅             | ❌ 403               |
  | `DELETE activity`                               | ✅ (soft delete + justificativa) | ❌ 403             | ❌ 403         | ❌ 403               |
  | `GET /dashboard/metrics`                        | ✅                               | ✅                 | ❌ 403         | ❌ 403               |
  | `GET /dashboard/financial`                      | ✅                               | ❌ 403             | ❌ 403         | ❌ 403               |
  | `GET /team`                                     | ✅                               | ❌ 403             | ❌ 403         | ❌ 403               |
  | `PATCH /system-configs`                         | ✅                               | ❌ 403             | ❌ 403         | ❌ 403               |
  | `POST /reports/5/generate` (Comissões/Analista) | ✅                               | ❌ 403             | ❌ 403         | ❌ 403               |
  | `POST /reports/10/generate` (LGPD)              | ✅                               | ❌ 403             | ❌ 403         | ❌ 403               |
  | `GET /cases/:id/activities`                     | ✅                               | ✅                 | ✅ (atribuído) | ✅ só EVENTO_SISTEMA |
  | `DELETE /cases/:id` (rollback arquivamento)     | ✅                               | ✅                 | ❌ 403         | ❌ 403               |
  | `PATCH /commissions/:id/cancel`                 | ✅ (quatro-olhos)                | ✅ (quatro-olhos)  | ❌ 403         | ❌ 403               |
  | `POST /audit/sla-alerts` resolve                | ✅                               | ✅                 | ❌ 403         | ❌ 403               |
  | `GET /audit.audit_logs`                         | ✅                               | ❌ 403             | ❌ 403         | ❌ 403               |
  - Validar: cada célula "❌ 403" testada em integração retorna 403; RLS PostgreSQL reflete o mesmo controle de acesso

---

## Check 4 — Máquinas de Estado Completas

- [x] Verificar completude de todas as máquinas de estado do sistema:

  **CaseState** (10 valores):
  - CADASTRO → SIMULACAO (Analista RS, Coord RS, Admin RS)
  - SIMULACAO → VERIFICACAO (Analista RS, Coord RS, Admin RS)
  - VERIFICACAO → PUBLICACAO (gate: Dossiê APROVADO + Tabela Atual ≤30 dias) (Coord RS, Admin RS)
  - PUBLICACAO → MATCH (automático via Platform RS sync)
  - MATCH → NEGOCIACAO (Analista RS, Coord RS, Admin RS)
  - NEGOCIACAO → ANUENCIA (ao aceitar proposta)
  - ANUENCIA → FORMALIZACAO (Coord RS, Admin RS)
  - FORMALIZACAO → CONCLUIDO (gate: 3 critérios cumulativos)
  - Qualquer estado ativo → CANCELADO (Admin RS + justificativa)
  - Qualquer estado ativo → ARQUIVADO (Admin RS + justificativa; reversível apenas por Admin RS)
  - ARQUIVADO → estado anterior (Admin RS, rollback com justificativa 20 chars)
  - Validar: `case_status_history` tem entrada para cada transição; transições inválidas retornam 422 `CRM-020`

  **NegotiationState** (6 valores):
  - EM_ANDAMENTO → PROPOSTA_ENVIADA → PROPOSTA_ACEITA → FECHADO
  - PROPOSTA_ENVIADA → PROPOSTA_REJEITADA → EM_ANDAMENTO (nova rodada; aviso após 5ª rodada)
  - EM_ANDAMENTO → CANCELADO (Admin RS)
  - Validar: após 5ª rodada de rejeição → aviso disparado; PROPOSTA_ENVIADA com 48h expirada → PROPOSTA_REJEITADA automático (ProposalExpirationWorker)

  **DossierState** (4 valores):
  - INCOMPLETO → EM_ANALISE (ao submeter para revisão)
  - EM_ANALISE → APROVADO (Coord RS, Admin RS, prazo 2 dias)
  - EM_ANALISE → REJEITADO (Coord RS, Admin RS + justificativa)
  - REJEITADO → EM_ANALISE (nova submissão após correção)
  - Validar: Dossiê APROVADO é gate para VERIFICACAO→PUBLICACAO; transição REJEITADO→INCOMPLETO não existe (volta para EM_ANALISE na resubmissão)

  **ActivityStatus** (4 valores):
  - REGISTRADA (criação de atividade não-follow-up)
  - AGENDADA → CONCLUIDA (via `POST /activities/:id/complete`)
  - AGENDADA → VENCIDA (3h após `scheduled_at`, via worker)
  - VENCIDA → CONCLUIDA (via `POST /activities/:id/complete`)
  - Validar: REGISTRADA não pode ir para CONCLUIDA (apenas follow-ups podem ser concluídos); CONCLUIDA é estado final

  **UserStatus** (3 valores):
  - PENDENTE_CONVITE → ATIVO (aceitação do convite)
  - PENDENTE_CONVITE → EXPIRADO (48h/7 dias sem aceitar)
  - ATIVO → SUSPENSO (Admin RS + justificativa)
  - SUSPENSO → ATIVO (Admin RS, reativação)
  - ATIVO/SUSPENSO → DESLIGADO (Admin RS + justificativa; irreversível)
  - Validar: DESLIGADO → ATIVO é impossível; e-mail banido por 180 dias após DESLIGADO

  **CommissionState** (4 valores):
  - PENDENTE → APROVADA (Coord RS ou Admin RS, quatro-olhos)
  - PENDENTE → CANCELADA (Admin RS + justificativa 100 chars, quatro-olhos)
  - APROVADA → PAGO (Admin RS após confirmação de pagamento)
  - APROVADA → CANCELADA (Admin RS + justificativa 100 chars, quatro-olhos)
  - Validar: PAGO → estado final; CANCELADA → estado final

---

## Check 5 — Fallbacks de Integração

- [x] Verificar que cada integração externa tem fallback e DLQ configurados:

  **WhatsApp Business (Meta Cloud API)**:
  - Falha no envio de mensagem → retry 2 min → retry 10 min → DLQ `whatsapp_failed` + notificação ao Analista RS
  - Webhook não validado (HMAC-SHA256 inválido) → 401, sem processar
  - Timeout de 30s na resposta da API Meta → 503 com mensagem ao usuário
  - Validar: DLQ configurada no CloudAMQP; falha após 3 tentativas → `notification_logs` com erro

  **ZapSign (Assinatura Eletrônica)**:
  - Falha na criação do envelope → retry 2 min → retry 10 min → 422 ao usuário com instrução manual
  - Lembretes automáticos D+2/D+4/D+5: falha no envio → DLQ; 3 falhas consecutivas → notificação Admin RS
  - Webhook inválido (HMAC-SHA256) → 401, sem processar
  - Validar: 3 falhas nos lembretes → `notification_logs` para Admin RS com `priority: P1`

  **Celcoin (Escrow)**:
  - Webhook Celcoin com HMAC inválido → 401, sem processar; registrar tentativa em `audit_logs`
  - Aprovação manual pelo Coord RS: notificação por e-mail ao Coord RS; sem resposta em 48h → re-notificação
  - Validar: webhook válido processado idempotentemente (mesmo evento recebido 2x → sem duplicata)

  **Platform RS (Sincronização Bidirecional)**:
  - Falha na sincronização → retry 2 min → retry 10 min → alerta `CRITICO` ao Admin RS
  - Defasagem > 2 min detectada → badge "Sincronização atrasada" no Dashboard
  - Conflito de dados (mesmo Caso modificado em ambos os lados) → registrar conflito em `audit_logs`, aplicar mais recente por `updated_at`
  - Validar: defasagem > 2 min → badge visível; conflito resolvido por `updated_at` + audit_log

  **Supabase Auth**:
  - `signOut(userId)` falha ao suspender/desligar usuário → retry 1x; se falhar → marcar `crm_users.force_logout = true` verificado no `JwtAuthGuard`
  - Validar: token válido de usuário SUSPENSO → 401 `CRM-019`

---

## Check 6 — Requisitos LGPD

- [x] Verificar conformidade completa com LGPD (RN-133 a RN-138, Doc 01.3):
  - **Retenção de dados**:
    - Casos `CANCELADO`: dados pessoais anonimizados após 5 anos → worker de anonimização anual verificado
    - Casos `CONCLUIDO`: dados pessoais anonimizados após 10 anos → worker de anonimização anual verificado
    - Validar: tabela `system_configs` tem parâmetro de data de retenção ou worker com cron anual configurado

  - **Opt-out de comunicações**:
    - Contato com `opt_out_communications = true`: bloqueio em `CommunicationsService.send()` → 422 `CRM-031`
    - Resposta `STOP` via webhook WhatsApp → seta `opt_out_communications = true` automaticamente
    - Badge "Opt-out ativo" visível no perfil do Contato
    - Validar: tentativa de enviar WhatsApp para opt-out → 422 CRM-031; webhook STOP processado

  - **Solicitações de titular (DSAR)**:
    - Endpoint `POST /lgpd/requests` para registrar solicitação de dados
    - Prazo de resposta 15 dias rastreado em `notification_logs`
    - Relatório LGPD (relatório 10) exclusivo para Admin RS
    - Validar: solicitação criada → `notification_logs` com `due_at = NOW() + 15 days`; alerta se prazo se aproximar

  - **Mascaramento de dados em listagens**:
    - `GET /cases` para Parceiro Externo → CPF e e-mail do Cedente/Cessionário mascarados
    - `GET /team` → e-mail mascarado para non-Admin RS
    - Validar: response de listagem para Parceiro Externo não contém CPF/e-mail em plain text

  - **Auditoria imutável**:
    - `audit.audit_logs` com `INSERT` only (sem UPDATE/DELETE) — verificar RLS PostgreSQL
    - Retenção 10 anos → sem mecanismo de purge automático
    - Validar: tentativa de `UPDATE audit.audit_logs` via Prisma → erro de permissão PostgreSQL

---

## Check 7 — Etapa Adversarial (Busca Ativa de Gaps)

- [x] **Gap Adversarial 1 — Dani-Admin webhook**: Verificar que o endpoint `POST /webhooks/dani-admin` existe, está autenticado (HMAC ou Bearer token), e processa payload de inteligência passiva (sem ação autônoma). Confirmar que Dani-Admin não tem capacidade de criar/editar Casos diretamente — apenas recebe notificações
  - Se ausente: marcar ⚠️ GAP — implementar em S-Final

- [x] **Gap Adversarial 2 — Notificação de redistribuição automática**: Ao desligar usuário com Casos → redistribuição automática ou notificação ao Coord RS se sem Analista disponível. Verificar que existe lógica de redistribuição (não apenas setando `assigned_to = NULL`)
  - Se ausente: marcar ⚠️ GAP — implementar em S-Final

- [x] **Gap Adversarial 3 — Rate limiting por endpoint**: Verificar que rate limit de 200 req/min (geral) e 10 req/min (auth) estão configurados via Throttler NestJS. Verificar header `Retry-After` em respostas 429
  - Se ausente: marcar ⚠️ GAP — implementar em S-Final

- [x] **Gap Adversarial 4 — Versionamento de API**: Verificar que URL base `https://api.repasseseguro.com.br/api/v1/crm` está configurada (prefixo `/api/v1` no NestJS). Nenhum endpoint sem o prefixo de versão
  - Se ausente: marcar ⚠️ GAP — implementar em S-Final

- [x] **Gap Adversarial 5 — Idempotency keys**: Verificar endpoints de criação críticos (POST /cases, POST /commissions, POST /communications/send) com header `Idempotency-Key` opcional → se mesmo key enviado 2x → retornar resposta cacheada (Redis TTL 24h) sem criar duplicata
  - ⚠️ AMBÍGUO: Doc 16 menciona idempotência para webhooks mas não especifica Idempotency-Key para endpoints REST. Adotar interpretação conservadora: verificar apenas para webhooks (Celcoin + ZapSign)

- [x] **Gap Adversarial 6 — Campos de `audit_logs` completos**: Verificar que cada evento registrado em `audit.audit_logs` inclui: `user_id`, `action`, `entity_type`, `entity_id`, `ip_address`, `user_agent`, `old_values` (JSON), `new_values` (JSON), `created_at`. Confirmar que o middleware de auditoria está ativo para todas as mutações críticas (state changes, commission changes, config changes, user changes)
  - Se ausente: marcar ⚠️ GAP — implementar em S-Final

- [x] **Gap Adversarial 7 — Schema validation em webhooks**: Verificar que todos os webhooks recebidos (WhatsApp, ZapSign, Celcoin, Platform RS) passam por validação Zod antes de processar. Payload inválido → 422 sem processar, sem lançar exceção não tratada
  - Se ausente: marcar ⚠️ GAP — implementar em S-Final

---

## Critério de Conclusão da Sprint Final

- [x] **GATE 1**: Cobertura 100% — TODOS os REQ-001 a REQ-312 têm pelo menos 1 item de checklist marcado em alguma sprint S1–S10
- [x] **GATE 2**: Zero P0 abertos — nenhum item classificado como P0 (sistema inoperante) pendente
- [x] **GATE 3**: Zero P1 abertos — nenhum item classificado como P1 (fluxo crítico quebrado) pendente
- [x] **GATE 4**: Glossário 52/52 — todos os 52 termos do Glossário Técnico têm implementação verificável
- [x] **GATE 5**: Matriz RBAC completa — 4 roles × todos os endpoints críticos testados
- [x] **GATE 6**: 5 fluxos E2E passando — todos os Playwright E2E da S10 executam sem falhas
- [x] **GATE 7**: Máquinas de estado documentadas com TODAS as transições válidas

**Se todos os 7 GATES passam**: Fase 3 concluída. Executar Auditoria A03 e depois `cp claude-fase-4.md CLAUDE.md`.

**Se algum GATE falha**: Corrigir na sprint corrente (máx 1 iteração). Se não resolve → marcar `[PENDENTE — REVISÃO MANUAL]` e registrar como dívida técnica P1 no `indice.md`.
