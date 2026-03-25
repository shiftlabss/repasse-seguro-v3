# S7 — Formalização

## Módulo Admin — Repasse Seguro

| Campo | Valor |
|---|---|
| **Sprint** | S7 |
| **Nome** | Formalização |
| **Tipo** | Dinâmica — Módulo Fullstack |
| **Template** | B |
| **Docs Consultados** | D01.1, D01.2, D01.5, D06, D16 |
| **Cross-cutting** | D10 (Glossário), D02 (Stacks) |
| **REQs cobertos** | REQ-008, REQ-010, REQ-011, REQ-019, REQ-033, REQ-036, REQ-047 a REQ-054, REQ-091, REQ-099, REQ-101 a REQ-109, REQ-129, REQ-140, REQ-167 a REQ-172, REQ-174, REQ-177, REQ-212 a REQ-217, REQ-223, REQ-294, REQ-304 a REQ-306 |
| **Total de itens** | 54 |

---

> **Critério de conclusão de S7:** Painel T-051 com 4 cards de critérios; envelope ZapSign enviado via `POST /api/v1/documents` com prazo 30 dias; webhook ZapSign validado via HMAC-SHA256 com 4 eventos processados idempotente; contingência ZapSign ativa após 2 timeouts + banner amarelo + SLA pausado; anuência da construtora com SLA 15 dias e fluxo de exceção; depósito Escrow com countdown e prorrogação 1x pelo Coordenador; Fechamento irreversível com digitação "CONFIRMAR"; fluxo de reversão em 15 dias; Celcoin abertura de escrow; período de custódia 15 dias bloqueando distribuição automática.

---

## ⚙️ BACKEND

### Feature: Formalização e 4 Critérios de Fechamento

- [x] **[BE-01]** Implementar `GET /v1/cases/:id/formalization` (ANALISTA — REQ-304): retorna estado dos 4 critérios obrigatórios de Fechamento (RN-023): (1) `instrument_signed: boolean` — envelope ZapSign com `document_completed`; (2) `price_confirmed: boolean` — Tabela Atual com fonte identificada no dossiê; (3) `annuence_ok: boolean | null` — anuência obtida ou dispensada; (4) `deposit_confirmed: boolean` — Conta Escrow em DEPOSITO_CONFIRMADO; campos adicionais: `annuence_na: boolean` (dispensada), `annuence_protocol: string?`, `delta_approved: boolean` (aprovação 4 olhos), `zapsign_status`, `escrow_status`, `deposit_deadline`, `prorrogation_count`; campo computed `all_criteria_met: boolean` (todos os 4 simultaneamente true). Verificar: `all_criteria_met` = true somente quando TODOS os 4 critérios individuais são true.

- [x] **[BE-02]** Implementar `PATCH /v1/cases/:id/formalization` (ANALISTA — REQ-304): campos editáveis: `annuence_ok?` (true = anuência recebida + upload obrigatório do documento), `annuence_na?` (true = contrato não exige + `justification` obrigatória), `annuence_protocol?`, `delta_approved?` (COORDENADOR only — RN-019.a); se `annuence_na = true` → exige confirmação do COORDENADOR em ≤2 dias úteis (RN-022.a); se `delta_approved = true` por não-COORDENADOR → 403; carimbo imutável ao confirmar anuência: nome Analista + data + hora; se Valor Recuperado > R$500.000 → exige validação adicional COORDENADOR independente do Analista (RN-022.a). Verificar: ANALISTA não consegue definir `delta_approved = true` (403); confirmação de anuência aplica carimbo imutável.

- [x] **[BE-03]** Implementar integração ZapSign — envio de envelopes (RN-121 — REQ-167/168): `POST /v1/cases/:id/zapsign` (ANALISTA/COORDENADOR/MASTER): campos `document_type` (enum: INSTRUMENTO_CESSAO/TERMO_COMERCIAL/TERMO_ACEITE_ESCALONAMENTO), `signatories: [{ name, email, cpf? }]`, `expires_at?` (default 30 dias corridos); chamar `POST https://api.zapsign.com.br/api/v1/documents` com API Key das Configurações; persiste `zapsign_document_token`, URLs de assinatura, status ENVIADO; em 5xx ou timeout → registrar falha em audit + ativar modo contingência após 2 falhas consecutivas (RN-123); retenta em 5xx (RN-121); mensagem ao operador em erro: "Não foi possível conectar ao serviço de assinaturas no momento. O caso permanece em Formalização e o envio pode ser tentado novamente."; registrar `DOCUMENTO_ENVIADO_ZAPSIGN` em audit. Verificar: 2 timeouts consecutivos ativam modo contingência; API Key inválida retorna mensagem específica ao operador.

- [x] **[BE-04]** Implementar `POST /v1/cases/:id/zapsign/:envelope_id/resend` (ANALISTA — REQ-306): reenviar envelope a signatário específico; limite máximo 3 reenvios por signatário (RN-032 — se > 3 → 422 "Limite de 3 reenvios atingido. Coordenador deve avaliar cancelamento."); apenas se signatário em status PENDENTE ou EXPIRADO; registrar em audit `DOCUMENTO_REENVIADO_ZAPSIGN`. Verificar: 4º reenvio retorna 422; reenvio só disponível para ANALISTA/COORDENADOR/MASTER.

- [x] **[BE-05]** Implementar webhook ZapSign `POST /v1/webhooks/zapsign` (público — RN-122 — REQ-169/170): validar `X-Zapsign-Signature` via HMAC-SHA256 com `webhook_secret` das Configurações; se inválida → descartar + log em audit (possível ataque); processar idempotente (mesmo `document_token` + mesmo evento não duplica); responder HTTP 200 em até 5 segundos; processar 4 eventos: (1) `document_signed` → atualizar status individual do signatário; notificação em tempo real ao painel; (2) `document_completed` → marcar critério `instrument_signed = true`; avaliar se `all_criteria_met = true`; notificar Analista; (3) `document_refused` → registrar recusa com motivo; manter caso em EM_FORMALIZACAO; notificação urgente ao Analista: "O signatário {Nome} recusou a assinatura. Verifique o motivo informado."; (4) `document_expired` → desmarca critério `instrument_signed = false`; notificação urgente ao Analista: "O prazo de assinatura expirou. Reenvie o envelope para reabrir o prazo." Verificar: `document_completed` marca critério e avalia `all_criteria_met`; webhook com assinatura inválida retorna 200 mas é descartado e logado.

- [x] **[BE-06]** Implementar modo de contingência ZapSign (RN-123 — REQ-171/172): ativação após 2 timeouts consecutivos (>30s); campo `zapsign_contingency_mode: boolean` nas Configurações; quando ativo: SLA da etapa Em Formalização → Fechamento é pausado automaticamente (RN-061 — REQ-129); banner ativo para frontend via endpoint `GET /v1/status/integrations`; Coordenador pode registrar assinatura manual (upload de comprovante + justificativa) via `POST /v1/cases/:id/formalization/manual-signature`; após 48h em contingência → escalar ao Master com badge "Escalado automaticamente" (DA-009 — REQ-172); ao restaurar → banner desaparece + SLA retoma + reprocessar envelopes pendentes. Verificar: SLA pausado durante contingência; Analista não consegue registrar assinatura manual (apenas COORDENADOR/MASTER).

- [x] **[BE-07]** Implementar job de SLA de anuência da construtora (RN-022.a — REQ-104): para casos com `annuence_ok = false` e `annuence_na = false` e `annuence_requested_at IS NOT NULL`: no 10º dia corridos → criar alerta para Analista e Coordenador; no 15º dia → escalar ao Coordenador com sugestão de ação (reenvio, contato diretoria, exceção); no 90º dia corridos → Coordenador decide entre cancelar ou escalação institucional; registrar em audit. Verificar: no 10º dia sem anuência → alerta gerado; no 15º → escalação ao Coordenador.

- [x] **[BE-08]** Implementar Celcoin — abertura de Conta Escrow (RN-125 — REQ-174): ao criar caso (ou ao entrar em EM_FORMALIZACAO) → chamar API Celcoin para abrir Conta Escrow com dados: `case_uuid`, `expected_value`, `cedente_data`, `cessionario_data`, `finalidade`; persistir `celcoin_escrow_id`; período de custódia de 15 dias pós-Fechamento: `POST /v1/cases/:id/escrow/distribute` retorna 403 se `days_since_fechamento < 15` (RN-128 — REQ-177); apenas MASTER pode antecipar distribuição com justificativa. Verificar: distribuição antes de 15 dias retorna 403; MASTER com justificativa consegue antecipar.

- [x] **[BE-09]** Implementar `PATCH /v1/cases/:id/close` (ANALISTA — REQ-294): (1) verificar `all_criteria_met = true`; se false → 422 com lista dos critérios pendentes: "Pendência: {critério faltante}. O fechamento só é liberado quando todos os 4 critérios estiverem cumpridos."; (2) exigir `confirmation_text = "CONFIRMAR"` no payload (DEC-007 — REQ-103); se texto incorreto → 422; (3) registrar obrigatoriamente: data/hora, nome Analista, screenshot JSON dos 4 critérios no momento, referências dos documentos ZapSign, comprovante escrow (REQ-102); (4) muda status do caso para FECHAMENTO → POS_FECHAMENTO; (5) iniciar contagem de 15 dias de custódia; (6) notificar Cedente e Cessionário: "O fechamento do caso {case_id} foi confirmado. O período de 15 dias para reversão começou hoje."; (7) Fechamento é irreversível (RN-024 — REQ-108): não existe endpoint de desfazer; única via = fluxo de reversão formal. Verificar: `confirmation_text != "CONFIRMAR"` retorna 422; Fechamento com critério pendente retorna 422 com critério específico.

### Feature: Reversão e Estados Pós-Fechamento

- [x] **[BE-10]** Implementar fluxo de reversão (REQ-050/051/052/053/054): `POST /v1/cases/:id/reversal` (GESTOR_FINANCEIRO): (1) validar que caso está em POS_FECHAMENTO e `days_since_fechamento <= 15`; se > 15 dias → 422; (2) `reversal_type`: CONSENSUAL → status EM_REVERSAO; UNILATERAL → status EM_MEDIACAO; (3) congela Conta Escrow imediatamente (status CONGELADA); (4) RS não recebe comissão em reversão consensual; (5) sem acordo em mediação em 10 dias úteis → status DISPUTA_FORMAL; acordo na mediação → CANCELADO; reversão confirmada → CANCELADO + estorno integral via Celcoin; notificar todas as partes; registrar em audit. Verificar: reversão após 15 dias retorna 422; EM_MEDIACAO sem acordo em 10 dias úteis → DISPUTA_FORMAL automaticamente.

---

## 🖥️ FRONTEND

### Feature: T-050 — Formalização Lista de Casos

- [x] **[FE-01]** Implementar `FormalizacaoPage` (T-050, rota `/formalizacao`): lista de casos em EM_FORMALIZACAO; colunas: ID, imóvel, critérios cumpridos (badges: Assinaturas ✅/⏳, Anuência ✅/⏳/N/A, Depósito ✅/⏳), dias no estado, SLA; caso com todos os 4 critérios = badge "Pronto para fechamento" (verde); layout master-detail; polling 60s + webhook push ZapSign/Celcoin (RN-132); `aria-live="polite"` para atualizações. Verificar: badge "Pronto para fechamento" aparece apenas quando todos os 4 critérios estão ✅.

### Feature: T-051 — Formalização Painel do Caso

- [x] **[FE-02]** Implementar `FormalizacaoCasePage` (T-051, rota `/formalizacao/:id`): painel central com cabeçalho + 4 cards de critério em grid 2x2 (desktop) ou lista (mobile) ou accordion (mobile); cada card mostra: nome do critério, status (Pendente ⏳ / Cumprido ✅), data de cumprimento, link para evidência; quando todos 4 = ✅: cards ficam verdes + botão "Confirmar Fechamento" na sidebar direita ativa com animação pulsação 3s para chamar atenção; botão desabilitado com tooltip listando critérios faltantes; banner de contingência ZapSign: banner amarelo "Serviço de assinaturas temporariamente indisponível." + timestamp + link "Registrar assinatura manual" (COORDENADOR/MASTER only). Verificar: botão desabilitado com tooltip dos critérios faltantes; animação pulsação ao ativar.

### Feature: T-052 — Card Assinaturas ZapSign

- [x] **[FE-03]** Implementar `ZapSignCard` (T-052): exibe status de cada signatário (Cedente / Cessionário / Construtora quando aplicável) com badge individual (Pendente / Assinado / Recusado / Expirado); barra de progresso fracionária "X/3 assinaturas" (DEC-012 — não percentual); botão "Enviar para assinatura" (estado inicial); botão "Reenviar" ao lado de signatários Pendente (após 5 dias) ou Expirado com contador "Reenvio X de 3"; atualização em tempo real via webhook ZapSign (Supabase Realtime); estado `critério cumprido`: card com fundo verde leve + badge "✅ Assinaturas concluídas". RBAC: GESTOR_FINANCEIRO pode apenas visualizar (sem ações). Verificar: "Reenvio 4 de 3" não aparece no DOM (limite atingido); badge atualiza via Realtime sem reload.

### Feature: T-053 — Card Anuência da Construtora

- [x] **[FE-04]** Implementar `AnuenciaCard` (T-053): se contrato exige anuência: campo upload documento + campo data emissão + campo protocolo + botão "Confirmar anuência recebida"; se não exige: card mostra "N/A — contrato não exige anuência" + badge cinza + COORDENADOR pode confirmar exceção; estados: Não solicitada → Solicitada → Recebida e verificada (critério cumprido) | Em atraso (badge vermelho + alerta) | Dispensada (exceção aprovada); para Valor Recuperado > R$500.000: campo "Aprovação do Coordenador obrigatória" com status. Verificar: card N/A aparece quando `annuence_na = true`; upload obrigatório ao confirmar anuência.

### Feature: T-054 — Card Depósito Escrow

- [x] **[FE-05]** Implementar `DepositoEscrowCard` (T-054): exibe valor esperado, valor depositado (atualizado via Celcoin webhook/Realtime), status (Aguardando / Parcial / Confirmado), data limite de depósito com countdown se <48h; badge "Prazo próximo" (amarelo, ≤3 dias) ou "Prazo estourado" (vermelho); ação COORDENADOR: link "Prorrogar prazo de depósito" (registra novo prazo com justificativa — máx 1x); estado critério cumprido: card verde leve. Verificar: countdown aparece quando deadline < 48h; link de prorrogação aparece apenas para COORDENADOR/MASTER.

### Feature: T-055 — Modal Confirmar Fechamento

- [x] **[FE-06]** Implementar `ConfirmClosureModal` (T-055): trigger = botão "Confirmar Fechamento" em T-051; exibe sumário: 4 critérios ✅ com checkmarks, valores calculados (comissão Cedente + comissão Cessionário + valor líquido ao Cedente), data/hora do fechamento; campo obrigatório: Analista digita "CONFIRMAR" para habilitar o botão (DEC-007); se Delta ≥ R$100.000 → botão desabilitado com mensagem "Aguardando aprovação do Coordenador." + notificação automática ao Coordenador; botão "Confirmar Fechamento" (primário `--primary`) + "Cancelar" (secondary); `role="dialog"`, `aria-modal="true"`, focus trap, Escape fecha; pós-fechamento: toast "Fechamento confirmado. Contagem de 15 dias iniciada." + status do caso atualizado. Verificar: campo vazio ou texto diferente de "CONFIRMAR" desabilita botão; Delta ≥ R$100k desabilita botão com mensagem.

---

## 🔗 WIRING

- [x] **[WIRE-01]** Implementar `formalizacao.service.ts` no frontend: `getFormalization(caseId)` → `GET /v1/cases/:id/formalization`; `updateFormalization(caseId, dto)` → `PATCH /v1/cases/:id/formalization`; `sendZapSign(caseId, dto)` → `POST /v1/cases/:id/zapsign`; `resendZapSign(caseId, envelopeId, signatory)` → `POST /v1/cases/:id/zapsign/:envelope_id/resend`; `closeCase(caseId, confirmationText)` → `PATCH /v1/cases/:id/close`; `initiateReversal(caseId, dto)` → `POST /v1/cases/:id/reversal`. Verificar: `closeCase` com `confirmationText != "CONFIRMAR"` captura 422 e exibe mensagem ao usuário.

- [x] **[WIRE-02]** Implementar listener Supabase Realtime na `FormalizacaoCasePage`: assinar canal `formalization:{caseId}`; ao receber evento ZapSign → atualizar badge do signatário; ao receber evento Celcoin deposit → atualizar `DepositoEscrowCard`; ao receber evento `all_criteria_met = true` → habilitar botão "Confirmar Fechamento" com animação; fallback polling 60s (RN-132). Verificar: `document_completed` de ZapSign atualiza card sem reload e habilita botão se todos os 4 critérios OK.

---

## ✅ TESTES

- [x] **[TEST-01]** Testes unitários backend: `PATCH /v1/cases/:id/close` com `confirmation_text != "CONFIRMAR"` retorna 422; Fechamento com critério pendente retorna 422 com critério específico; webhook ZapSign com HMAC inválida descartado e logado; `document_completed` marca `instrument_signed = true`; 2 timeouts ZapSign ativam modo contingência; reversão após 15 dias retorna 422. Cobertura: 80% de branches.

- [x] **[TEST-02]** Testes unitários frontend: `ConfirmClosureModal` desabilita botão com texto diferente de "CONFIRMAR"; `ZapSignCard` exibe "Reenvio 3 de 3" e esconde botão "Reenviar" ao atingir limite; animação pulsação ativa quando `all_criteria_met = true`.

- [x] **[TEST-03]** Testes de integração: fluxo completo de Fechamento — 4 critérios cumpridos → `PATCH /close` → status POS_FECHAMENTO → notificações disparadas; webhook `document_completed` → `instrument_signed = true` → `all_criteria_met` avaliado; modo contingência ZapSign → SLA pausado → restauração → SLA retomado.

- [x] **[TEST-04]** Testes E2E (Playwright): ANALISTA acessa `/formalizacao/:id` → envia envelope ZapSign → simula webhook `document_completed` → card Assinaturas fica verde → completa os outros 3 critérios → botão "Confirmar Fechamento" ativa → digita "CONFIRMAR" → confirma → toast exibido → caso em POS_FECHAMENTO.

---

## 🔍 AUTO-VERIFICAÇÃO S7 (12 checks)

- [x] **[CHECK-01]** `PATCH /v1/cases/:id/close` com `confirmation_text != "CONFIRMAR"` retorna 422
- [x] **[CHECK-02]** `PATCH /v1/cases/:id/close` com critério pendente retorna 422 com critério específico nomeado
- [x] **[CHECK-03]** Webhook ZapSign com HMAC-SHA256 inválida: descartado + registrado em audit (não retorna 4xx)
- [x] **[CHECK-04]** `document_completed` marca `instrument_signed = true` e avalia `all_criteria_met`
- [x] **[CHECK-05]** 2 timeouts consecutivos ZapSign (>30s) ativam modo contingência + SLA pausado
- [x] **[CHECK-06]** Contingência ZapSign: banner amarelo no frontend; link "Registrar assinatura manual" visível apenas para COORDENADOR/MASTER
- [x] **[CHECK-07]** Distribuição da Conta Escrow antes de 15 dias pós-Fechamento retorna 403
- [x] **[CHECK-08]** Reversão após 15 dias corridos pós-Fechamento retorna 422
- [x] **[CHECK-09]** EM_MEDIACAO sem acordo em 10 dias úteis → status DISPUTA_FORMAL automaticamente (job cron)
- [x] **[CHECK-10]** `ConfirmClosureModal`: botão desabilitado com tooltip dos critérios faltantes; ativo com animação pulsação quando todos ✅
- [x] **[CHECK-11]** Reenvio ZapSign: 4ª tentativa retorna 422 "Limite de 3 reenvios atingido"
- [x] **[CHECK-12]** Todos os REQs de S7 (REQ-008, 010, 011, 019, 033, 036, 047–054, 091, 099, 101–109, 129, 140, 167–172, 174, 177, 212–217, 223, 294, 304–306) cobertos
