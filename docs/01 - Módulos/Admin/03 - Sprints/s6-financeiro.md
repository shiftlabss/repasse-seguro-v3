# S6 — Financeiro

## Módulo Admin — Repasse Seguro

| Campo | Valor |
|---|---|
| **Sprint** | S6 |
| **Nome** | Financeiro |
| **Tipo** | Dinâmica — Módulo Fullstack |
| **Template** | B |
| **Docs Consultados** | D01.2, D01.4, D01.5, D06, D16 |
| **Cross-cutting** | D10 (Glossário), D02 (Stacks) |
| **REQs cobertos** | REQ-006, REQ-007, REQ-020, REQ-034, REQ-035, REQ-084, REQ-085, REQ-087 a REQ-100, REQ-137 a REQ-139, REQ-141, REQ-142, REQ-175, REQ-176, REQ-218 a REQ-222, REQ-224, REQ-307 a REQ-310 |
| **Total de itens** | 50 |

---

> **Critério de conclusão de S6:** Tabela de Contas Escrow com filtros chips e Realtime; drawer de detalhe T-061 com breakdown barras empilhadas; bloqueio de distribuição com justificativa ≥20 chars → aprovação Master ≤24h → escalonamento 8h/16h/e-mail emergência; distribuição automática em 15 dias com notificação prévia 24h; webhook Celcoin confirmando depósito com tolerância R$0,50; estorno sempre integral; painel de inadimplência com alertas amarelo/vermelho; conciliação bancária com matching ≤R$0,50; comissões calculadas corretamente por cenário; retentativa de 3x com intervalo 1h em falha bancária.

---

## ⚙️ BACKEND

### Feature: Escrow e Distribuição

- [x] **[BE-01]** Implementar `GET /v1/cases/:id/escrow` (GESTOR_FINANCEIRO/COORDENADOR/MASTER — REQ-307): retorna estado completo da Conta Escrow: `status` (ABERTA/DEPOSITO_CONFIRMADO/EM_DISTRIBUICAO/DISTRIBUIDA/CONGELADA/BLOQUEADA/ESTORNADA), `expected_value`, `deposited_value`, `deposit_date`, `scheduled_distribution_date`, `residual_credit`, `breakdown: { cedente_value, rs_value }`, `transactions: []`. COORDENADOR: somente leitura (sem ações). Verificar: ANALISTA não consegue acessar endpoint (403); COORDENADOR recebe dados sem campos de ação.

- [x] **[BE-02]** Implementar webhook `POST /v1/webhooks/celcoin` (público com HMAC-SHA256 — RN-126/REQ-175): validar assinatura `X-Celcoin-Signature` via HMAC-SHA256 com `webhook_secret` das Configurações; se inválida → descartar + log; se válida → processar idempotente via `X-Idempotency-Key`; para evento `deposit_confirmed`: comparar `confirmed_value` com `expected_value`; se diferença ≤ R$0,50 → status DEPOSITO_CONFIRMADO; excedente registrado como `residual_credit`; se diferença > R$0,50 → status permanece ABERTA + notificar Cessionário "O valor recebido na Conta Escrow está abaixo do esperado. Deposite o valor restante de R$ {diferença} para confirmar."; notificação em tempo real ao painel do Analista (Supabase Realtime). Verificar: depósito com diferença R$0,30 = confirmado; depósito com diferença R$1,00 = não confirmado + notificação.

- [x] **[BE-03]** Implementar `POST /v1/cases/:id/escrow/distribute` (GESTOR_FINANCEIRO — REQ-308): (1) validar que Conta Escrow está em status DEPOSITO_CONFIRMADO; (2) validar que 15 dias corridos pós-Fechamento transcorreram (RN-020); (3) calcular distribuição: Cedente = Valor Recuperado − Comissão Cedente; RS = Comissão Cedente + Comissão Cessionário; crédito residual → Cessionário; (4) muda status para EM_DISTRIBUICAO; (5) chamar Celcoin API para transferências; (6) em caso de falha bancária → retentativa com 3 tentativas a intervalo de 1 hora cada (RN-127/REQ-176); resposta 202 (assíncrono); após sucesso → status DISTRIBUIDA; registrar em audit. Verificar: 202 retornado imediatamente; status atualiza para DISTRIBUIDA após execução assíncrona; falha bancária retenta 3x com 1h de intervalo.

- [x] **[BE-04]** Implementar distribuição automática por job cron (RN-021): verificar diariamente casos em POS_FECHAMENTO há 14 dias sem reversão → enviar notificação prévia 24h ao Gestor Financeiro: "A Conta Escrow do caso {case_id} será distribuída em 24 horas. Revise os valores e bloqueie se necessário."; no 15º dia sem reversão e sem bloqueio ativo → disparar distribuição automaticamente; registrar em audit. Verificar: notificação 24h antes disparada no 14º dia; distribuição automática no 15º dia se não bloqueada.

- [x] **[BE-05]** Implementar `PATCH /v1/cases/:id/escrow/block` (GESTOR_FINANCEIRO — REQ-309): campo `block_reason` (mínimo 20 caracteres); se < 20 chars → 422; muda status para BLOQUEADA; notificar Master imediatamente por e-mail + painel; iniciar timer de 24h para aprovação (RN-088); job de escalonamento: após 8h sem resposta → reenviar com urgência "Distribuição será liberada automaticamente em {X} horas"; após 16h → notificar TODOS os Masters ativos; se nenhum Master ativo → usar e-mail emergência de Configurações → Segurança; se 24h expirarem sem aprovação → status volta para EM_DISTRIBUICAO + retomar distribuição + registrar incidente severidade alta em audit (RN-088.a). Verificar: `block_reason` < 20 chars retorna 422; notificação ao Master disparada imediatamente; após 24h sem aprovação distribuição retoma automaticamente.

- [x] **[BE-06]** Implementar `PATCH /v1/cases/:id/escrow/unblock` (MASTER — REQ-310): aprova desbloqueio da distribuição bloqueada pelo Gestor; campo `approval_note?`; muda status de BLOQUEADA para EM_DISTRIBUICAO; notificar Gestor Financeiro "O bloqueio de distribuição foi aprovado pelo Master. A distribuição permanece suspensa."; registrar em audit. Verificar: endpoint acessível apenas por MASTER; GESTOR_FINANCEIRO retorna 403.

- [x] **[BE-07]** Implementar estorno via Celcoin (RN-089/RN-141): `POST /v1/cases/:id/escrow/reverse` (GESTOR_FINANCEIRO): (1) congela Conta Escrow imediatamente (status CONGELADA); (2) validar que estorno é SEMPRE do valor integral depositado (RN-141) — se algum parâmetro tentar estorno parcial → 422 "O estorno deve ser do valor integral depositado. Não é possível processar estornos parciais."; (3) chamar Celcoin para estorno; (4) gerar comprovante PDF salvo no dossiê automaticamente; (5) status → ESTORNADA; (6) notificar Cedente e Cessionário; (7) 3 retentativas com 1h de intervalo em falha. Verificar: tentativa de estorno parcial retorna 422; comprovante gerado automaticamente no dossiê.

- [x] **[BE-08]** Implementar `GET /v1/cases/:id/commission` (GESTOR_FINANCEIRO — REQ-307): retorna cálculo de comissão por cenário usando `case_config_snapshot`: Cenário A = comissão Cedente R$0,00; Cenários B/C/D = 20% × (Valor Recuperado − Distrato Ref. = 50% do valor pago — RN-018); Comissão Cessionário = 20% × Δ (Tabela Atual − Tabela do Contrato — RN-019); se Δ < 0 → comissão variável R$0,00; se Delta ≥ R$100.000 → criar alerta de aprovação 4 olhos ao Coordenador (RN-019.a — REQ-089); Cenário A com Δ = 0 + exceção ativa → comissão = 20% × valor pago. Verificar: Cenário A retorna comissão_cedente = 0; Cenário B calcula corretamente 20% × (Valor_Recuperado − Distrato_Ref).

- [x] **[BE-09]** Implementar painel de inadimplência (RN-090): `GET /v1/financeiro/inadimplencia` (GESTOR_FINANCEIRO): retorna casos com Conta Escrow em status ABERTA com `deposit_deadline` próximo ou vencido; cada item inclui `days_overdue`, `days_remaining`, `prorrogation_count`, `sla_status` (YELLOW quando ≤3 dias úteis restantes, RED quando vencido); endpoint `POST /v1/cases/:id/escrow/prorrogate` (COORDENADOR): concede prorrogação de +5 dias úteis com `justification` obrigatória; limite: 1 prorrogação por caso (`prorrogation_count = 1` bloqueia nova concessão). Verificar: COORDENADOR com 2ª tentativa de prorrogação no mesmo caso retorna 422; prazo ≤3 dias úteis = status YELLOW.

- [x] **[BE-10]** Implementar conciliação bancária (RN-091): `POST /v1/financeiro/conciliacao` (GESTOR_FINANCEIRO): aceita upload de extrato bancário (PDF/CSV); sistema faz matching automático de movimentações por valor + data ± 1 dia; resultado: `conciliados: []`, `divergencias: []` (diferença > R$0,50), `nao_identificados: []`; divergência detectada → badge "Divergência detectada" + exige justificativa do Gestor antes de fechar (`PATCH /v1/financeiro/conciliacao/:id/close` com `justification` ≥20 chars); gerar relatório exportável. Verificar: diferença de R$0,30 = conciliado; diferença de R$1,00 = divergência; fechar divergência sem justificativa retorna 422.

---

## 🖥️ FRONTEND

### Feature: T-060 — Financeiro Contas Escrow

- [x] **[FE-01]** Implementar `FinanceiroEscrowPage` (T-060, rota `/financeiro/escrow`): tabela de todas as Contas Escrow; colunas: ID do caso, Status da Conta Escrow (badge colorido), Valor Esperado, Valor Depositado, Data do Depósito, Data Prevista de Distribuição, Ações; filtros chips acima da tabela por status: Aberta / Depósito Confirmado / Em Distribuição / Distribuída / Congelada / Estornada (chips clicáveis, chip ativo = preenchido; múltiplos combinados); destaques: "Prazo próximo" (badge amarelo) e "Prazo estourado" (badge vermelho) com tooltip de dias; ação por linha: "Ver detalhe" → T-061 (drawer); GESTOR_FINANCEIRO: dropdown "Bloquear distribuição" (→ T-064), "Iniciar reversão" (→ T-065); Supabase Realtime + badge "Ao vivo" no cabeçalho; `aria-live="polite"` na tabela; mobile: cards por conta. Verificar: chips filtram corretamente; Realtime atualiza status sem reload.

- [x] **[FE-02]** Implementar `EscrowDetailDrawer` (T-061): drawer lateral 480px (fullscreen mobile); seções: Dados do caso (ID, imóvel, Cedente, Cessionário), Valores (barras empilhadas horizontais mostrando Cedente % | RS % + valores monetários), Histórico de movimentações (timeline cronológica), Ações por perfil; 6 estados visuais da Conta Escrow com ícones (D06 §2.7 — Aberta: cofre aberto cinza; Depósito Confirmado: cofre fechado verde; Em Distribuição: setas divergentes azul; Distribuída: check verde escuro; Congelada: cadeado vermelho; Estornada: seta reversa laranja). Verificar: barras empilhadas refletem proporção correta Cedente vs RS; ícones corretos por status.

- [x] **[FE-03]** Implementar `BlockDistributionModal` (T-064): trigger = "Bloquear distribuição"; campo justificativa obrigatório (mín. 20 chars, contador de caracteres visível); aviso "A solicitação será enviada ao Master para aprovação. A distribuição será suspensa imediatamente."; botão "Solicitar bloqueio" habilitado apenas quando `justificativa.length >= 20`; `ActionButton` (spinner + desabilitado durante request); pós-ação: toast "Solicitação de bloqueio enviada."; `role="dialog"`, `aria-modal="true"`, focus trap, Escape fecha. Verificar: botão desabilitado com 19 chars; habilitado com 20+ chars.

- [x] **[FE-04]** Implementar `InitiateReversalModal` (T-065): trigger = "Iniciar reversão"; exibe: valor total depositado, partes envolvidas, aviso "(sem comissão para RS)"; checkbox obrigatório "Confirmo o estorno integral de R$ {valor} ao Cessionário {alias}."; botão "Confirmar reversão" (`--destructive`, habilitado só com checkbox); pós-ação: Conta Escrow congelada + comprovante gerado + toast "Reversão iniciada. Comprovante gerado no dossiê.". Verificar: checkbox desmarcado desabilita botão; pós-ação exibe toast correto.

### Feature: T-062 — Comissões e Faturas

- [x] **[FE-05]** Implementar `FinanceiroComissoesPage` (T-062, rota `/financeiro/comissoes`): tabela de faturas de comissão do RS; colunas: ID fatura, ID caso, Valor, Status (Pendente/Distribuída/Cancelada badge colorido), Data de criação, Data de distribuição; filtros: Status, período, faixa de valor; export: botão "Exportar CSV" + "Exportar PDF"; polling 120s (RN-132). Verificar: export CSV gera arquivo com dados corretos; filtro por período funciona.

### Feature: T-063 — Painel Inadimplência

- [x] **[FE-06]** Implementar `FinanceiroInadimplenciaPage` (T-063, rota `/financeiro/inadimplencia`): tabela de casos com depósito em atraso; colunas: ID caso, Valor Esperado, Dias em atraso, Ações; alertas visuais: badge "Prazo próximo" (amarelo, ≤3 dias úteis restantes) + badge "Prazo estourado" (vermelho); ações por perfil: **GESTOR_FINANCEIRO** = "Notificar Cessionário" (re-dispara notificação) + "Iniciar cancelamento" (→ modal); **COORDENADOR** = "Prorrogar prazo" (com justificativa); **MASTER** = todas; polling 120s. Verificar: ANALISTA não acessa a rota (403/redirect); COORDENADOR vê "Prorrogar prazo" mas não "Iniciar cancelamento".

### Feature: T-066 — Conciliação Bancária

- [x] **[FE-07]** Implementar `ConciliacaoModal` (T-066): upload extrato bancário (PDF/CSV); após upload → exibir resultados em 3 seções: ✅ Conciliados, ⚠️ Divergências (diferença > R$0,50), ❓ Não identificados; botões: "Exportar relatório de conciliação" + "Registrar divergência manualmente" (abre campo justificativa ≥20 chars) + "Fechar". Verificar: upload de CSV com divergência aparece em seção correta; fechar divergência sem justificativa mantém modal aberto.

---

## 🔗 WIRING

- [x] **[WIRE-01]** Implementar `financeiro.service.ts` no frontend: `getEscrowAccounts(filters)` → `GET /v1/financeiro/escrow`; `getEscrowDetail(caseId)` → `GET /v1/cases/:id/escrow`; `blockDistribution(caseId, dto)` → `PATCH /v1/cases/:id/escrow/block`; `initiateReversal(caseId)` → `POST /v1/cases/:id/escrow/reverse`; `distributeEscrow(caseId)` → `POST /v1/cases/:id/escrow/distribute`; `getCommission(caseId)` → `GET /v1/cases/:id/commission`; `prorrogateDeposit(caseId, dto)` → `POST /v1/cases/:id/escrow/prorrogate`. Verificar: `blockDistribution` com `block_reason` < 20 chars captura 422 e exibe mensagem ao usuário.

- [x] **[WIRE-02]** Integrar Supabase Realtime na `FinanceiroEscrowPage`: assinar canal `escrow:all` (GESTOR_FINANCEIRO/MASTER) ou `escrow:assigned` (COORDENADOR); ao receber evento de mudança de status → `queryClient.invalidateQueries(['escrow'])` + atualizar badge na linha correspondente; fallback polling 120s (RN-132). Verificar: confirmação de depósito via webhook Celcoin atualiza status na tabela em tempo real.

---

## ✅ TESTES

- [x] **[TEST-01]** Testes unitários backend: webhook Celcoin com assinatura HMAC-SHA256 inválida → descartado; depósito com diferença R$0,30 = confirmado; depósito com diferença R$1,00 = não confirmado; tentativa de estorno parcial retorna 422; `block_reason` < 20 chars retorna 422; cálculo comissão Cenário B = 20% × (Valor_Recuperado − 50% × valor_pago). Cobertura: 80% de branches.

- [x] **[TEST-02]** Testes unitários frontend: `BlockDistributionModal` desabilita botão com <20 chars; `InitiateReversalModal` desabilita botão sem checkbox; chips de filtro combinados filtram corretamente; barras empilhadas no drawer refletem proporção correta.

- [x] **[TEST-03]** Testes de integração: distribuição automática no 15º dia via job cron; escalonamento de bloqueio: após 8h sem resposta → reenvio urgente; após 24h → distribuição automática + incidente em audit; retentativa de 3x com 1h de intervalo em falha bancária.

- [x] **[TEST-04]** Testes E2E (Playwright): GESTOR_FINANCEIRO acessa `/financeiro/escrow` → filtra por "Depósito Confirmado" → clica "Bloquear distribuição" → preenche justificativa → confirma → verifica toast e status alterado para "Bloqueada".

---

## 🔍 AUTO-VERIFICAÇÃO S6 (12 checks)

- [x] **[CHECK-01]** Webhook Celcoin com HMAC-SHA256 inválida é descartado e registrado em log
- [x] **[CHECK-02]** Depósito com diferença ≤R$0,50: status = DEPOSITO_CONFIRMADO; diferença >R$0,50: status permanece ABERTA + notificação
- [x] **[CHECK-03]** Estorno parcial retorna 422 "O estorno deve ser do valor integral depositado"
- [x] **[CHECK-04]** Bloqueio de distribuição: `block_reason` < 20 chars retorna 422
- [x] **[CHECK-05]** Escalonamento de aprovação: após 24h sem resposta do Master → distribuição automática + incidente em audit
- [x] **[CHECK-06]** Distribuição automática no 15º dia pós-Fechamento sem reversão (notificação 24h antes)
- [x] **[CHECK-07]** Comissão Cenário A: cedente = R$0,00; RS = comissão Cessionário (20% × Δ)
- [x] **[CHECK-08]** Delta ≥ R$100.000 gera alerta de aprovação 4 olhos ao Coordenador
- [x] **[CHECK-09]** Prorrogação de depósito: 2ª tentativa pelo COORDENADOR retorna 422 (limite 1x por caso)
- [x] **[CHECK-10]** Conciliação bancária: diferença R$0,30 = Conciliado; diferença R$1,00 = Divergência detectada
- [x] **[CHECK-11]** Retentativa de distribuição/estorno: 3 tentativas com intervalo de 1h em falha bancária
- [x] **[CHECK-12]** Todos os REQs de S6 (REQ-006, 007, 020, 034, 035, 084, 085, 087–100, 137–139, 141, 142, 175, 176, 218–222, 224, 307–310) cobertos
