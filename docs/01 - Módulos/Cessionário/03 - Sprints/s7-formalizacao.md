# S7 — Formalização

## Sprint 7 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S7 — Formalização                                                                                                         |
| **Template**       | B — Módulo Fullstack (feature com vertical slice Banco→Backend→Frontend→Wiring→Testes)                                    |
| **REQs cobertos**  | S7-001 a S7-014 (14 requisitos do Registro Mestre)                                                                        |
| **Docs fonte**     | 01.2 - RN Core e Receita · 01.5 - Integrações · 06 - Mapa de Telas · 16 - Documentação de API · 17 - Integrações Externas |
| **Total de itens** | 43 itens                                                                                                                  |
| **Status**         | Concluída                                                                                                                 |

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos: `formalizations`, `FormalizationStatus`, ZapSign, telas T-ASS-01/02/03.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — 4 documentos obrigatórios; FormalizationStatus 6 estados; ZapSign retry 3x/30min; webhook HMAC-SHA256; SLA geração 2h.
- [x] ✅ Check 4 — Glossário: `ASSINATURA_PENDENTE_CESSIONARIO`, `AGUARDANDO_ANUENCIA`, `zapsign_session_id`.
- [x] ✅ Check 5 — Anti-scaffold R10: fluxo de assinatura digital real, validação HMAC, transições de estado reais.
- [x] ✅ Check 6 — Máquina de estado FormalizationStatus: DOCUMENTOS_DISPONIVEIS→ASSINATURA_PENDENTE_CESSIONARIO→ASSINATURA_PENDENTE_CEDENTE→AGUARDANDO_ANUENCIA→CONCLUIDA; CANCELADA como terminal. Completo.
- [x] ✅ Check 7 — SLA geração documento 2h (RN-054); ZapSign retry 3x backoff 1s→2s→4s; reconciliação 30min.
- [x] ✅ Check 8 — Cross-módulo: CONCLUIDA → aciona S8 (Fechamento). ZapSign failure → Admin notificado silenciosamente.
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — RN-063 (ZapSign hidden failure) sinalizado.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S7-001 a S7-014 cobertos.

---

## FEATURE — Formalização (Documentos, Assinatura ZapSign)

### 🗄️ Banco

- [x] **S7-B01** · Confirmar `formalizations` table: `id UUID PK`, `negotiation_id UUID UNIQUE FK → negotiations.id RESTRICT`, `cessionario_id UUID FK → cessionarios.id RESTRICT`, `status FormalizationStatus DEFAULT DOCUMENTOS_DISPONIVEIS`, `zapsign_session_id VARCHAR(255)`, `zapsign_buyer_sign_url VARCHAR(500)`, `document_url VARCHAR(500)`, `buyer_signed_at TIMESTAMPTZ`, `seller_signed_at TIMESTAMPTZ`, `anuencia_obtained_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`. Índice `idx_formalizations_cessionario_id`. RLS: cessionário lê apenas suas formalizações; update apenas por `service_role`. (Doc 13 — model Formalization)

- [x] **S7-B02** · Confirmar bucket Supabase Storage `formalization-docs` privado: documentos de formalização gerados pelo sistema; Signed URLs com TTL configurável; acesso somente via backend (`service_role`); ZapSign recebe URL do documento via signed URL gerada pelo backend. (Doc 17 — Supabase Storage)

### ⚙️ Backend

- [x] **S7-BE01** · Implementar `GET /api/v1/formalizations` em `apps/api/src/modules/formalization/`: guards `JwtAuthGuard` + `KycGuard`; retornar formalizações do cessionário; paginação offset-based; incluir: `negotiation_id`, `opportunity_code`, `status` com `FormalizationStatus`, `zapsign_buyer_sign_url` se disponível, `document_url_signed` (TTL 15min), `buyer_signed_at`, `seller_signed_at`; ordenação `created_at DESC`. (Doc 16 — GET /formalizations)

- [x] **S7-BE02** · Implementar `GET /api/v1/formalizations/:id` em `apps/api/src/modules/formalization/`: guards `JwtAuthGuard` + `KycGuard`; verificar pertinência ao cessionário; retornar dados completos da formalização + todos os campos; `document_url_signed` com TTL 15min via Supabase Storage signed URL; incluir dados da negociação + oportunidade (sem Cedente); flag `can_sign: boolean` (true se `status = ASSINATURA_PENDENTE_CESSIONARIO` e `zapsign_buyer_sign_url` disponível). (Doc 16 — GET /formalizations/:id)

- [x] **S7-BE03** · Implementar serviço de geração de documentos em `FormalizationService.generateDocument()`: acionado quando `formalizations.status = DOCUMENTOS_DISPONIVEIS`; gerar PDF do instrumento de cessão com dados da negociação (sem identificação do Cedente — apenas código OPR); upload do PDF para Supabase Storage `formalization-docs/`; SLA: 2h (RN-054); se falhar, logar `error` e notificar Admin internamente (sem notificar o cessionário — falha silenciosa conforme RN-063). (Doc 01.4 — RN-054; Doc 01.5 — RN-063)

- [x] **S7-BE04** · Implementar integração ZapSign em `apps/api/src/infrastructure/zapsign/`: cliente HTTP com `Authorization: Bearer ${ZAPSIGN_API_TOKEN}`; método `createSigningSession(documentUrl, signers)` — `POST /docs/` com PDF URL, signatários (Cessionário: nome, e-mail; Cedente: dados mascarados — apenas código) e redirect URL pós-assinatura; armazenar `zapsign_session_id` e `zapsign_buyer_sign_url` em `formalizations`; atualizar `status = ASSINATURA_PENDENTE_CESSIONARIO`; disparar NOT-CES-08 (documento pronto para assinatura) via NotificationService; retry 3x backoff exponencial (1s→2s→4s), timeout 15s; DLQ `zapsign.dead-letter`; se falhar após 3 tentativas em 30min (RN-063), Admin notificado internamente, `status` permanece `DOCUMENTOS_DISPONIVEIS` (não alterar para erro). (Doc 17 — seção ZapSign; Doc 01.5 — RN-063)

- [x] **S7-BE05** · Implementar webhook receiver `POST /api/v1/webhooks/zapsign` em `apps/api/src/modules/formalization/`: validar HMAC-SHA256 no header `X-ZapSign-Signature` com `ZAPSIGN_WEBHOOK_SECRET`; processar 3 eventos: (1) `signer_signed` — identificar se Cessionário (→ `buyer_signed_at = NOW()`, `status = ASSINATURA_PENDENTE_CEDENTE`) ou Cedente (→ `seller_signed_at = NOW()`); (2) `doc_complete` — ambos assinaram → `status = AGUARDANDO_ANUENCIA`, `anuencia_obtained_at` campo preparado; (3) `signer_refused` — Admin notificado, status mantido para investigação; retornar HTTP 200 imediatamente (processamento assíncrono). (Doc 17 — seção ZapSign webhooks; Doc 01.5 — RN-063)

- [x] **S7-BE06** · Implementar job de reconciliação ZapSign em `apps/api/src/modules/formalization/jobs/zapsign-reconciliation.job.ts`: cron `*/30 * * * *` (a cada 30min); buscar `formalizations` com `status IN ('ASSINATURA_PENDENTE_CESSIONARIO', 'ASSINATURA_PENDENTE_CEDENTE')` e `zapsign_session_id != null` e `updated_at < NOW() - 2h` (webhook não recebido em 2h); chamar `GET /docs/{zapsign_session_id}/` para verificar status; se documento completo, processar igual ao webhook; se não, continuar aguardando. (Doc 17 — seção ZapSign fallback)

- [x] **S7-BE07** · Implementar lógica de conclusão da formalização em `FormalizationService.completeFormalization()`: chamado quando `status = AGUARDANDO_ANUENCIA` e anuência obtida (Admin confirma via painel); atualizar `status = CONCLUIDA`, `anuencia_obtained_at = NOW()`; atualizar `negotiations.status = ENCERRADA`; atualizar `opportunities.status = CONCLUIDA`; acionar S8 (criação de `financial_transactions` para comissão e conclusão de operação); disparar NOT-CES-09 (operação concluída). (Doc 01.2 — RN-033, RN-034, RN-036)

### 🎨 Frontend

- [x] **S7-FE01** · Implementar tela `T-ASS-01 — Documentos de Formalização` em `apps/web/src/features/formalizations/pages/FormalizationDocsPage.tsx` (rota `/formalizacao/:id`): buscar via `GET /api/v1/formalizations/:id`; exibir status com `FormalizationStatusBadge`; lista de documentos: "Instrumento de Cessão" com botão "Visualizar" (abre PDF via Signed URL em nova aba) e botão "Baixar"; exibir signatários: Cessionário (você) — status de assinatura; Cedente (anônimo) — status de assinatura; botão "Assinar digitalmente" visível apenas se `can_sign = true` → redireciona para T-ASS-02; timeline do processo com datas. (Doc 06 — T-ASS-01)

- [x] **S7-FE02** · Implementar tela `T-ASS-02 — Assinatura` em `apps/web/src/features/formalizations/pages/FormalizationSignPage.tsx` (rota `/formalizacao/:id/assinar`): exibir dados da cessão (valor, comissão, oportunidade); aviso "Ao assinar, você confirma ciência das condições."; botão "Assinar via ZapSign" → redireciona para `zapsign_buyer_sign_url` (abre T-ASS-03 — modal WebView/iframe); após retorno do ZapSign (redirect URL configurada), verificar status via `GET /api/v1/formalizations/:id`; se `buyer_signed_at` preenchido → toast "Documento assinado! Aguardando assinatura do cedente." + atualizar status. (Doc 06 — T-ASS-02)

- [x] **S7-FE03** · Implementar tela `T-ASS-03 — ZapSign WebView` em `apps/web/src/features/formalizations/pages/ZapSignWebViewPage.tsx` (modal full-screen): abrir `zapsign_buyer_sign_url` em `<iframe>` (web) ou `WebView` (mobile via expo-router modal); ao receber `postMessage` do ZapSign de conclusão OU após redirect de retorno: fechar modal, voltar para T-ASS-02, re-buscar status; botão "Fechar" no topo → Alert "Deseja sair? Você precisará acessar o link de assinatura novamente."; em mobile: `swipe down to dismiss` com proteção. (Doc 06 — T-ASS-03; Doc 11 — modal full-screen)

- [x] **S7-FE04** · Implementar `FormalizationStatusBadge.tsx` em `apps/web/src/components/domain/`: mapear 6 estados de `FormalizationStatus`: DOCUMENTOS_DISPONIVEIS (azul, "Documentos disponíveis"), ASSINATURA_PENDENTE_CESSIONARIO (âmbar, "Aguardando sua assinatura"), ASSINATURA_PENDENTE_CEDENTE (azul-claro, "Aguardando assinatura do cedente"), AGUARDANDO_ANUENCIA (laranja, "Aguardando anuência"), CONCLUIDA (verde, "Formalização concluída"), CANCELADA (cinza, "Cancelada").

### 🔌 Wiring

- [x] **S7-W01** · Configurar Realtime subscription em `formalizations` no frontend: ao receber UPDATE com `status = ASSINATURA_PENDENTE_CESSIONARIO` → toast "Documento pronto para assinatura!" + badge no sidebar; ao receber `status = CONCLUIDA` → toast de alta prioridade "Parabéns! Operação concluída!"; degradação para polling 30s. (Doc 21 — NOT-CES-08, NOT-CES-09)

- [x] **S7-W02** · Configurar redirect URL pós-assinatura ZapSign: URL de retorno para T-ASS-02 (`https://app.repasseseguro.com.br/formalizacao/{id}/assinar?signed=true`) deve ser configurada no payload de criação da sessão ZapSign `POST /docs/`; ao retornar com `?signed=true`, o frontend re-busca `GET /api/v1/formalizations/:id` e verifica `buyer_signed_at`. (Doc 17 — ZapSign; Doc 16)

### 🧪 Testes

- [x] **S7-T01** · Testes unitários `FormalizationService`: webhook `signer_signed` (Cessionário) → `buyer_signed_at` preenchido + `ASSINATURA_PENDENTE_CEDENTE`; webhook `doc_complete` → `AGUARDANDO_ANUENCIA`; webhook com HMAC inválido → 401; ZapSign retry 3x falha → status permanece `DOCUMENTOS_DISPONIVEIS` (não propagação de erro para usuário). Cobertura 100% branches.

- [x] **S7-T02** · Testes unitários `ZapSignReconciliationJob`: formalização com `updated_at < NOW() - 2h` e `zapsign_session_id` → chamada GET ZapSign; resultado completo → processa como webhook; sem resultado → aguarda próximo ciclo.

- [x] **S7-T03** · Testes E2E Playwright — TC-CES-14 "Acesso ao documento para assinatura": login, navegar para formalização `ASSINATURA_PENDENTE_CESSIONARIO`, verificar `can_sign = true`, verificar botão "Assinar via ZapSign" visível; TC-CES-15 "Conclusão de formalização" (mock webhook): disparar `doc_complete` webhook → verificar `AGUARDANDO_ANUENCIA`. (Doc 01.3 — TC-CES-14, TC-CES-15)

---

## 🔀 Cross-Módulo

- [x] **S7-CM01** · **[← S6]** `formalizations` record com `status = DOCUMENTOS_DISPONIVEIS` criado atomicamente em S6 (quando Escrow confirmado). `FormalizationService.generateDocument()` monitora records neste status para gerar PDF e iniciar fluxo ZapSign.

- [x] **S7-CM02** · **[→ S8]** `FormalizationService.completeFormalization()` deve chamar `FinancialService.recordCommission(negotiationId)` para criar `financial_transactions` (type: COMMISSION) e acionar o fluxo de fechamento de S8.

---

## 📊 COBERTURA DE REQs S7

| REQ ID | Descrição                                            | Item(s) |
| ------ | ---------------------------------------------------- | ------- |
| S7-001 | GET /formalizations                                  | S7-BE01 |
| S7-002 | GET /formalizations/:id                              | S7-BE02 |
| S7-003 | Geração de documento PDF SLA 2h                      | S7-BE03 |
| S7-004 | Integração ZapSign criação sessão                    | S7-BE04 |
| S7-005 | ZapSign retry 3x/30min                               | S7-BE04 |
| S7-006 | Falha ZapSign silenciosa (RN-063)                    | S7-BE04 |
| S7-007 | Webhook ZapSign HMAC-SHA256                          | S7-BE05 |
| S7-008 | Transições via webhook (signer_signed, doc_complete) | S7-BE05 |
| S7-009 | Job reconciliação ZapSign 30min                      | S7-BE06 |
| S7-010 | completeFormalization → ENCERRADA + CONCLUIDA        | S7-BE07 |
| S7-011 | T-ASS-01 Documentos                                  | S7-FE01 |
| S7-012 | T-ASS-02 Assinatura                                  | S7-FE02 |
| S7-013 | T-ASS-03 ZapSign WebView modal                       | S7-FE03 |
| S7-014 | FormalizationStatusBadge 6 estados                   | S7-FE04 |
