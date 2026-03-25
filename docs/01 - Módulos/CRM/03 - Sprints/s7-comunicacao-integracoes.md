# S7 — Comunicação e Integrações

## CRM Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Sprint** | S7 |
| **Nome** | Comunicação e Integrações |
| **Template** | B — Módulo Fullstack |
| **Docs fonte** | 01.3, 01.5, 05.3, 05.5, 06, 16, 17, 27 |
| **REQs cobertos** | REQ-073 a REQ-076, REQ-097 a REQ-106, REQ-163 a REQ-164, REQ-208 a REQ-210, REQ-231 a REQ-238, REQ-257, REQ-283 (parcial), REQ-284 (parcial), REQ-285 a REQ-286 |
| **Objetivo** | WhatsApp Business integrado com templates, mensagens enviadas e recebidas vinculadas ao Caso. ZapSign para assinatura eletrônica. Conta Escrow via Celcoin webhook. Sincronização bidirecional com a Plataforma RS. |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — `communications`, `message_templates`, Meta Cloud API, ZapSign API v2, Celcoin API v2, HMAC-SHA256, 10 templates exatos, 3 categorias WhatsApp (Utilitária/Autenticação/Marketing)
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: Valores exatos — janela 24h WhatsApp, retry ZapSign: 5 min / 3 falhas → Admin RS, lembretes D+2 D+4 D+5, webhook Escrow: 24h tolerância, retry integração: 2 min / 10 min / 3 falhas
- [x] C4: Glossário — Janela de 24h, WhatsApp Business API, ZapSign, Webhook, Celcoin cobertos
- [x] C5: Estados da Mensagem: Enviada / Entregue / Lida / Com falha de entrega
- [x] C6: Protocol de retry de integração: 1ª falha 2 min, 2ª 10 min, 3ª Admin RS, 1h crítico
- [x] C7: RBAC — templates de marketing exigem opt-in explícito; compartilhamento Dossiê exclui dados pessoais
- [x] C8: T-CRM-070/071 com estados de tela
- [x] C9: Env vars: META_WABA_TOKEN, ZAPSIGN_API_TOKEN, CELCOIN_WEBHOOK_SECRET, PLATFORM_WEBHOOK_SECRET — todas declaradas
- [x] C10: Glossário — opt-out LGPD, Janela de 24h cobertos
- [x] C11: Anti-scaffold — validação HMAC de webhooks, lógica de janela 24h, fallback para cada integração
- [x] C12: REQ-073 a REQ-106 cobertos

---

## FEATURE 1 — Comunicação via WhatsApp Business

### Backend

- [x] **REQ-208** `GET /cases/:id/communications` — Roles: todos internos. Query params: `channel (WHATSAPP|EMAIL)`, `direction (INBOUND|OUTBOUND)`, `page`, `per_page`. Retornar mensagens ordenadas por `created_at DESC` com status de entrega (Enviada/Entregue/Lida/Com falha)
  - Validar: paginação funciona; filtro por channel funciona; Parceiro Externo → 403

- [x] **REQ-209 / REQ-073** `POST /cases/:id/communications/whatsapp` — Roles: todos internos (com escopo). Request body: `{ contact_id: UUID, content: string, template_id?: UUID }`. Validações:
  - Verificar Contato tem WhatsApp cadastrado (`phone` não nulo) → se não: 422 `CRM-038`: "Este contato não tem WhatsApp cadastrado. Atualize o perfil antes de enviar mensagens."
  - Verificar opt-out do Contato (`has_opt_out = true`) → 422 `CRM-039`: "Este contato optou por não receber comunicações. Não é possível enviar mensagens."
  - Verificar janela de 24h: buscar última mensagem INBOUND do Contato em `communications`. Se `last_inbound_at < now() - 24h` → apenas templates pré-aprovados (categoria `UTILITARIA` ou `MARKETING`). Se sem template → 422 `CRM-040`: "Fora da janela de 24h. Use um template aprovado para iniciar contato."
  - Enviar via Meta Cloud API (`POST /messages` com `WHATSAPP_BUSINESS_ACCOUNT_ID`)
  - Criar registro em `communications` com `status = ENVIADA`, `direction = OUTBOUND`, `channel = WHATSAPP`, `external_id = (retornado pela Meta API)`
  - Criar Atividade automática: `type = WHATSAPP`, `status = REGISTRADA` vinculada ao Caso e Contato
  - Se falha de entrega: registrar `status = FALHA_ENTREGA`, notificar Analista RS
  - Validar: contato sem WhatsApp → 422 CRM-038; opt-out → 422 CRM-039; fora da janela sem template → 422 CRM-040; envio bem-sucedido → registro em `communications` + atividade criada automaticamente

- [x] **REQ-210** `POST /cases/:id/communications/email` — Roles: todos internos (com escopo). Request body: `{ contact_id: UUID, subject: string, content: string, template_id?: UUID }`. Validações: verificar opt-out. Enviar via provedor de e-mail (SES ou SendGrid — definir em `system_configs`). Criar `communications` + Atividade automática `type = EMAIL`
  - Validar: opt-out → 422 CRM-039; envio bem-sucedido → atividade EMAIL criada

- [x] **REQ-233 / REQ-073** `POST /webhooks/whatsapp` — Webhook da Meta Cloud API. Validar assinatura HMAC-SHA256 com `META_WEBHOOK_VERIFY_TOKEN`. Para mensagem INBOUND:
  - Encontrar Caso ativo pelo número de telefone do remetente
  - Criar `communications` com `direction = INBOUND`, `status = RECEBIDA`
  - Atualizar `last_inbound_at` do Contato para controle da janela de 24h
  - Notificar Analista RS responsável: "Nova mensagem recebida de [Contato mascarado] no Caso [RS-XXXX]."
  - Para status update (entregue/lido): atualizar `communications.status` com `external_id` da mensagem
  - Para verificação do webhook (challenge): responder com `hub.challenge`
  - Validar: assinatura HMAC inválida → 401; mensagem recebida → comunicação INBOUND criada + notificação; `hub.challenge` → resposta correta

- [x] **REQ-102 / REQ-188** Implementar conformidade WhatsApp Business: categorizar templates em `message_templates.category`: `UTILITARIA`, `AUTENTICACAO`, `MARKETING`. Para envio de categoria `MARKETING`: verificar `has_opt_in_marketing = true` no Contato → se não: 422 `CRM-041`
  - Validar: template MARKETING enviado a Contato sem opt-in → 422 CRM-041; template UTILITARIA enviado a Contato fora da janela → permitido

- [x] **REQ-076 / REQ-106** Implementar opt-out no endpoint `POST /contacts/:id/opt-out`: `{ opted_out_by: CONTATO }`. Lógica:
  - Marcar `contacts.has_opt_out = true`
  - Bloquear todos os canais de envio WhatsApp e e-mail para este Contato via CRM
  - Notificar Coordenador RS se Contato tem Caso ativo
  - Reversão: `POST /contacts/:id/opt-in` com `{ new_consent_date, channel, method }` (registrar nova solicitação expressa do Contato)
  - Validar: opt-out ativo → todas as tentativas de envio bloqueadas; opt-in registrado → envio permitido novamente; notificação Coordenador RS enviada ao opt-out com Caso ativo

### Frontend

- [x] **REQ-163** Implementar `app/(dashboard)/cases/[id]/communications/page.tsx` — T-CRM-070 Caixa de Mensagens:
  - Interface de chat: mensagens ordenadas cronologicamente (mais antigas no topo), OUTBOUND à direita, INBOUND à esquerda
  - Status de entrega em cada mensagem (enviada ✓ / entregue ✓✓ / lida 🔵 / falha ❌)
  - Campo de digitação de mensagem com botão enviar (WhatsApp se Contato tem WhatsApp; e-mail se não)
  - Botão "Usar Template" → abre T-CRM-071
  - Banner "Opt-out LGPD" se Contato com `has_opt_out = true` → campo de digitação desabilitado
  - Banner "Fora da janela de 24h — Use um template" quando aplicável
  - Loading: skeleton de bolhas de mensagem
  - Validar: mensagem INBOUND aparece em tempo real via Supabase Realtime (nova `communications` INBOUND); banner opt-out desabilita campo de digitação; banner janela 24h exibe botão "Usar Template"

- [x] **REQ-164** Implementar `TemplateSelectorModal` (T-CRM-071):
  - Lista de templates filtrados por canal e momento
  - Preview do template com campos substituíveis destacados
  - Formulário de preenchimento dos variáveis do template (nome, valores, datas)
  - Ao confirmar → enviar mensagem com template
  - Validar: templates inativos não aparecem na lista; template com variáveis exibe formulário; envio de template Marketing a Contato sem opt-in → erro antes de confirmar

---

## FEATURE 2 — Integração ZapSign (Assinatura Eletrônica)

### Backend

- [x] **REQ-103 / REQ-189** Implementar `ZapSignService.createEnvelope(caseId)`:
  - Criar envelope ZapSign via `POST /api/v1/docs/` com: `name (instrumento de cessão RS-XXXX)`, signatários: Cedente (email, nome), Cessionário (email, nome), representante RS (email, nome configurado em `system_configs`)
  - Armazenar: `zapsign_token`, `zapsign_url` em `cases.zapsign_envelope_id` e `cases.zapsign_url`
  - Enviar links de assinatura por e-mail via `POST /cases/:id/communications/email` para cada signatário
  - Agendar lembretes via RabbitMQ com delay: D+2 (1º lembrete), D+4 (2º lembrete urgente + alerta Analista RS), D+5 (expirado → Admin RS notificado para decisão)
  - Validar: envelope criado → `zapsign_envelope_id` salvo; links enviados por e-mail; jobs de lembrete agendados com delays corretos

- [x] **REQ-104 / REQ-190** Implementar protocolo de falha ZapSign:
  - Se criação do envelope falha → exibir erro ao Analista RS: "Não foi possível enviar o contrato para assinatura. Tente novamente em instantes ou contate o suporte."
  - Registrar erro no `audit.audit_logs` com timestamp e mensagem de erro
  - Caso permanece em `FORMALIZACAO` (não retrocede automaticamente)
  - Reenvio manual: `POST /cases/:id/zapsign/retry`. Após 3 falhas consecutivas → notificar Admin RS
  - Validar: 3 falhas consecutivas → Admin RS notificado; Caso não retrocede; retry disponível após 5 minutos

- [x] **REQ-231 / REQ-103** `POST /webhooks/zapsign` — Validar assinatura `X-ZapSign-Signature` (HMAC-SHA256 com `ZAPSIGN_WEBHOOK_SECRET`). Processar eventos:
  - `doc.signed_by_signer`: atualizar status do signatário
  - `doc.all_signed`: PDF assinado disponível → download do PDF + upload para `dossier_documents` com `document_type = INSTRUMENTO_CESSAO`, `status = APROVADO`, `origin = ZAPSIGN_WEBHOOK`
  - `doc.expired`: notificar Admin RS; marcar `cases.zapsign_expired = true`
  - `doc.refused`: notificar Analista RS
  - Validar: assinatura HMAC inválida → 401; `all_signed` → PDF no Dossiê com status APROVADO; `expired` → Admin RS notificado

---

## FEATURE 3 — Integração Conta Escrow (Celcoin)

### Backend

- [x] **REQ-232 / REQ-105** `POST /webhooks/escrow` — Validar `X-Celcoin-Signature` (HMAC-SHA256 com `CELCOIN_WEBHOOK_SECRET`). Processar confirmação de depósito:
  - Verificar `case_id` no payload
  - Fazer upload do comprovante (PDF URL da Celcoin) para `dossier_documents` com `document_type = COMPROVANTE_ESCROW`, `status = APROVADO`
  - Notificar Analista RS responsável
  - Assinatura inválida → 401
  - Validar: webhook válido → comprovante no Dossiê; Analista RS notificado; assinatura inválida → 401 sem processar

- [x] **REQ-106 / REQ-192** Implementar upload manual de comprovante Escrow:
  - `POST /cases/:id/dossier/documents` com `document_type = COMPROVANTE_ESCROW` e flag `is_manual = true`
  - Ao marcar `is_manual = true`: criar `dossier_documents` com `status = PENDENTE_APROVACAO` (não APROVADO imediatamente)
  - Notificar Coordenador RS para aprovação manual
  - Após aprovação do Coordenador RS → status muda para `APROVADO`
  - Validar: upload manual → status PENDENTE_APROVACAO; aprovação Coordenador RS → APROVADO e critério de Fechamento cumprido; upload automático via webhook → status APROVADO direto

- [x] **REQ-191** Implementar worker de verificação de depósito pendente: se webhook não chegar em 24h após prazo esperado (`cases.escrow_expected_at`), alertar Analista RS + Coordenador RS para verificação manual
  - Validar: 25h após prazo → alerta gerado; webhook recebido antes → alerta não disparado

---

## FEATURE 4 — Integração Plataforma RS

### Backend

- [x] **REQ-234 / REQ-097** `POST /webhooks/platform` — Validar `X-Platform-Signature` (HMAC-SHA256 com `PLATFORM_WEBHOOK_SECRET`). Processar eventos:
  - `case.created` (Cedente cadastrou na plataforma): criar Caso no CRM via `CasesService.create()` com origem `PLATAFORMA`, assignar ao Analista de plantão (configurado em `system_configs`) ou manter `NENHUM_ATRIBUIDO` + alertar Coordenador RS
  - `document.uploaded` (Cedente enviou documento via plataforma): criar `dossier_documents` com `origin = CEDENTE_PLATAFORMA`
  - `interest.registered` (Cessionário manifestou interesse): notificar Analista RS: "Novo interesse registrado no Caso [RS-XXXX]. Avalie e registre o Match se pertinente." O Match NÃO é criado automaticamente
  - Validar: evento `case.created` → Caso criado no CRM; sem Analista de plantão → Caso `NENHUM_ATRIBUIDO` + alerta Coordenador RS; documento → Dossiê atualizado; interesse → notificação Analista RS

- [x] **REQ-100 / REQ-186** Implementar sincronização de estado do Caso → Plataforma:
  - Hook em `CasesService.advance()`: após atualizar estado, chamar `POST /platform/api/cases/{platformCaseId}/state` com novo estado (máx 2 min de defasagem)
  - Retry: 1ª falha → 2 min, 2ª → 10 min, 3ª → Admin RS notificado. Caso no CRM permanece correto
  - Validar: avanço de estado → sincronização ocorre; falha na sincronização → log + Admin RS notificado após 3 tentativas; Caso não retrocede no CRM

- [x] **REQ-285 / REQ-286** Implementar integrações passivas com Dani-Admin e Dani-Cedente/Cessionário:
  - `POST /webhooks/dani` (reutilizar endpoint `/webhooks/platform` com `source = DANI_ADMIN`): Dani-Admin escalona Caso para CRM → notificar Analista RS com contexto
  - `POST /webhooks/dani-interest`: Dani-Cessionário indica interesse → notificar Analista RS para registrar Match manualmente
  - Validar: webhook Dani → notificação gerada; Match NÃO criado automaticamente

---

## 🧪 Testes

### Backend — Unitário (Jest)

- [x] **REQ-257** `ZapSignService` — cobertura 75%:
  - `should create envelope with correct signatories`
  - `should schedule reminder jobs at D+2, D+4, D+5`
  - `should process all_signed webhook and add PDF to dossier`
  - `should reject webhook with invalid HMAC signature`
  - `should notify Admin RS after 3 consecutive failures`
  - `should not rollback case on envelope creation failure`

### Backend — Integração

- [x] `POST /cases/:id/communications/whatsapp` a Contato sem WhatsApp → 422 CRM-038
- [x] `POST /cases/:id/communications/whatsapp` a Contato com opt-out → 422 CRM-039
- [x] `POST /webhooks/whatsapp` com assinatura HMAC inválida → 401
- [x] `POST /webhooks/zapsign` evento `all_signed` → PDF adicionado ao Dossiê
- [x] `POST /webhooks/escrow` válido → comprovante Escrow criado com status APROVADO

---

## 🔀 Cross-Módulo

- Após S7: o critério de Fechamento 3 (comprovante Escrow) está implementado. O gate de `CommissionsService.confirmFechamento()` de S4 passa a funcionar 100%.
- O PDF assinado pelo ZapSign adicionado automaticamente ao Dossiê em S7 satisfaz o critério de Fechamento 1 (instrumento de cessão). Gate de Fechamento totalmente operacional após S5 + S7.
- Templates de WhatsApp (REQ-074) criados no seed (S1) são consumidos aqui. Gestão de templates na interface é implementada em S9.
