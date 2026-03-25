# S4b — Triagem

## Módulo Admin — Repasse Seguro

| Campo                | Valor                                                                                                                                                                                                                                    |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**           | S4b                                                                                                                                                                                                                                      |
| **Nome**             | Triagem                                                                                                                                                                                                                                  |
| **Tipo**             | Dinâmica — Módulo Fullstack (sub-sprint de S4)                                                                                                                                                                                           |
| **Template**         | B                                                                                                                                                                                                                                        |
| **Docs Consultados** | D01.1, D01.3, D06, D16                                                                                                                                                                                                                   |
| **Cross-cutting**    | D10 (Glossário), D02 (Stacks)                                                                                                                                                                                                            |
| **REQs cobertos**    | REQ-017, REQ-031, REQ-039, REQ-040, REQ-041, REQ-056, REQ-057, REQ-058, REQ-059, REQ-060, REQ-061, REQ-062, REQ-121, REQ-122, REQ-123, REQ-124, REQ-125, REQ-201, REQ-202, REQ-203, REQ-204, REQ-205, REQ-296, REQ-297, REQ-298, REQ-299 |
| **Total de itens**   | 52                                                                                                                                                                                                                                       |

---

> **Critério de conclusão de S4b:** Fila FIFO funcional com destaque no caso mais antigo; alerta informativo ao selecionar fora de ordem registrado em audit; 6 documentos do dossiê verificáveis com carimbo imutável (nome+timestamp); progresso salvo automaticamente ao navegar; bloqueio por inadimplência funcional (T-034 modal + notificação ao Cedente); desbloqueio exige comprovante atualizado; exceção de adimplência por Coordenador com protocolo; sugestão automática de cancelamento após 60 dias bloqueado; endpoints de dossiê com upload max 10MB pdf/jpg/png.

---

## ⚙️ BACKEND

### Feature: Fila de Triagem e Dossiê

- [x] **[BE-01]** Implementar `GET /v1/triagem` (ANALISTA/COORDENADOR/MASTER): retorna casos com status `EM_TRIAGEM` ou `BLOQUEADO` ordenados por `entered_current_status_at ASC` (FIFO obrigatório — RN-055); cada item inclui: `id`, `property_address`, `entered_current_status_at`, `dossie_progress` (`{ verified: N, total: 6 }`), `assigned_analyst.name`, `sla_status`, `sla_days_remaining`; ANALISTA filtra por `assigned_analyst_id = userId`; COORDENADOR/MASTER veem todos. Verificar: ordem FIFO correta (mais antigo primeiro); ANALISTA não vê casos de outro analista.

- [x] **[BE-02]** Implementar `GET /v1/cases/:id/dossie` (ANALISTA): retorna lista dos 6 documentos do dossiê; cada item: `document_type` (enum: contrato_original, comprovante_parcela_1, comprovante_parcela_2, comprovante_parcela_3, declaracao_adimplencia, identificacao, comprovante_endereco, tabela_contrato — mínimo 6 obrigatórios — RN-002), `file_url`, `upload_at`, `status` (PENDENTE/VERIFICADO/REJEITADO), `verified_by.name`, `verified_at`, `rejection_reason`, `history` (versões anteriores preservadas — RN-148). Verificar: documento rejeitado com `history` preserva versão anterior; campo `verified_by` tem carimbo imutável.

- [x] **[BE-03]** Implementar `POST /v1/cases/:id/dossie` (ANALISTA): `multipart/form-data` com campos `document_type` e `file` (max 10MB; formatos aceitos: pdf, jpg, jpeg, png — REQ-297); se `document_type` já existe com status REJEITADO → novo arquivo substitui na checklist ativa, status muda para PENDENTE, histórico preserva versão anterior (RN-148); se `document_type` novo → adicionar ao dossiê; registrar em `audit.audit_logs`; retorna 201. Verificar: upload de PDF 11MB retorna 422; reenvio de documento rejeitado muda status para PENDENTE e preserva histórico.

- [x] **[BE-04]** Implementar `PATCH /v1/cases/:id/dossie/:doc_id/verify` (ANALISTA): muda status do documento para VERIFICADO; aplica carimbo imutável: `verified_by = userId`, `verified_at = NOW()`; carimbo não pode ser alterado após aplicado (imutabilidade RN-056); registrar em audit. Se todos os 6 documentos obrigatórios estiverem VERIFICADOS → atualizar `dossie_progress.verified = 6`. Verificar: após verificar, `verified_by` e `verified_at` são imutáveis (PATCH subsequente não os altera); 6 documentos verificados atualiza `dossie_progress`.

- [x] **[BE-05]** Implementar `PATCH /v1/cases/:id/dossie/:doc_id/reject` (COORDENADOR — REQ-298/299): campo obrigatório `rejection_reason` com mínimo 20 caracteres; muda status para REJEITADO; registrar em audit com `rejection_reason`. Verificar: `rejection_reason` com 19 chars retorna 422; com 20+ chars salva e registra em audit.

- [x] **[BE-06]** Implementar `POST /v1/cases/:id/dossie/:doc_id/approve` (COORDENADOR — REQ-298): aprova documento diretamente sem upload (para exceções); registrar em audit. Verificar: aprovação por COORDENADOR sem upload aplica carimbo de verificação corretamente.

- [x] **[BE-07]** Implementar lógica de salvamento automático de progresso de triagem (RN-057): ao chamar `PATCH /v1/cases/:id/dossie/:doc_id/verify` ou `/reject` → sistema persiste o estado parcial automaticamente (não é necessário "salvar" explicitamente); ao retornar ao caso → `GET /v1/cases/:id/dossie` reflete exatamente o estado anterior; campo `dossie_progress.in_progress = true` quando `verified < 6`; campo `in_progress_message` = `"Verificação em andamento — {verified} de 6 documentos revisados"`. Verificar: após verificar 3 de 6 e navegar para outra tela, retorno ao caso exibe 3 documentos VERIFICADOS.

- [x] **[BE-08]** Implementar `PATCH /v1/cases/:id/block` (ANALISTA): muda status para BLOQUEADO; `reason` obrigatório; registrar em audit; disparar job de notificação ao Cedente: "Seu caso foi bloqueado por pendência de parcelas. Regularize e envie os comprovantes para retomar." (RN-001 — REQ-058); iniciar timer de 60 dias (job cron diário verifica casos BLOQUEADO há 60 dias → gera alerta ao Coordenador: "O caso {id} está bloqueado há 60 dias sem ação do Cedente. Recomenda-se cancelamento." — REQ-060). Verificar: bloqueio sem `reason` retorna 422; notificação ao Cedente disparada; após 60 dias alerta gerado para Coordenador.

- [x] **[BE-09]** Implementar `PATCH /v1/cases/:id/unblock` (ANALISTA atribuído ou COORDENADOR): muda status de BLOQUEADO para EM_TRIAGEM; valida pré-condição: ao menos 1 comprovante de parcela atualizado no dossiê (RN-058); se não houver comprovante atualizado → 422 "Desbloqueio exige comprovante de parcela atualizado no dossiê."; notificar Cedente: "Seu comprovante foi validado. Seu caso será avaliado em até 3 dias úteis." (REQ-059); registrar em audit. Verificar: desbloqueio sem comprovante atualizado retorna 422; desbloqueio com comprovante move status e notifica.

- [x] **[BE-10]** Implementar `PATCH /v1/cases/:id/qualify` (ANALISTA): muda status para QUALIFICADO; pré-condição: todos os 6 documentos do dossiê com status VERIFICADO (RN-002/REQ-061/REQ-062) — se não → 422 "Todos os 6 documentos devem estar verificados antes de qualificar."; adimplência confirmada no dossiê; registrar em audit; reiniciar timer SLA (Qualificado → Oferta Ativa: alvo 2 dias úteis / máximo 3 dias úteis). Verificar: qualificação com 5 documentos verificados retorna 422; com 6 documentos avança para QUALIFICADO.

- [x] **[BE-11]** Implementar exceção de adimplência por confirmação da construtora (RN-001.a — REQ-057): endpoint `PATCH /v1/cases/:id/approve-exception` (COORDENADOR): campos `protocol_number`, `confirmation_document` (upload); valida que `protocol_number` não é vazio; registrar em audit com `exception_type = "ADIMPLENCIA_CONSTRUTORA"`, `protocol_number`; avança caso de BLOQUEADO para QUALIFICADO. Verificar: endpoint acessível apenas por COORDENADOR/MASTER; ANALISTA retorna 403.

---

## 🖥️ FRONTEND

### Feature: T-030 — Triagem Fila FIFO

- [x] **[FE-01]** Implementar `TriagemPage` (T-030, rota `/triagem`): layout master-detail em desktop/tablet (lista 1/3 + painel trabalho 2/3); mobile: lista fullscreen → tap abre T-031 (push navigation); lista ordenada FIFO; cada item: ID do caso, endereço, data de entrada, tempo na fila relativo ("Há 2 dias"), barra de progresso do dossiê "X/6 documentos", analista atribuído; caso no topo: borda esquerda `--primary` + badge "Próximo na fila"; `role="list"` com `role="listitem"` por item; foco no primeiro item ao abrir. Verificar: caso mais antigo tem borda + badge; mobile exibe lista fullscreen.

- [x] **[FE-02]** Implementar alerta FIFO ao selecionar caso fora de ordem (RN-055): ao clicar em caso não-FIFO (não é o mais antigo da fila) → exibir banner amarelo no topo do painel: "Este caso não é o próximo na fila. O caso {ID} aguarda há mais tempo." + link "Ir para o caso mais antigo"; seleção é **permitida** (não bloqueante — DEC-009); log de auditoria registrado automaticamente pelo backend. Verificar: selecionar segundo caso da fila exibe banner com ID do caso mais antigo; banner tem link funcional.

- [x] **[FE-03]** Implementar estados da fila T-030: `loading` = skeleton list; `vazio` = ícone checklist vazio + "Todos os casos foram triados. Não há pendências no momento." (sem CTA — Analistas não criam casos); `erro` = alert inline com "Tentar novamente". Verificar: lista vazia mostra mensagem sem CTA.

### Feature: T-031 — Triagem Painel do Caso

- [x] **[FE-04]** Implementar `TriagemCasePage` (T-031, rota `/triagem/:id`): layout duas colunas (60% abas + 40% sidebar); header: ID + endereço + estado + cenário + nome Cedente + botão "Retornar à fila" (link, canto superior esquerdo); 2 abas: Dossiê (T-032) e Adimplência (T-033); sidebar: informações do caso, histórico de ações, botões de ação principal. Responsividade: desktop = duas colunas; tablet = sidebar reduzida 30%; mobile = abas fullscreen + sidebar colapsada como accordion. Verificar: botão "Retornar à fila" leva para `/triagem`; layout correto nos 3 breakpoints.

- [x] **[FE-05]** Implementar 3 estados do T-031: `caso bloqueado` = banner fixo no topo "Caso Bloqueado — Inadimplência" (fundo `--destructive`, texto branco), botões de avanço desabilitados, apenas "Desbloquear" visível para ANALISTA atribuído e COORDENADOR; `caso qualificado` = banner verde "Caso Qualificado — Aguardando publicação de oferta pelo Coordenador."; `loading` = skeleton nas abas. Verificar: caso BLOQUEADO não exibe botão "Qualificar Caso"; caso QUALIFICADO exibe banner verde.

- [x] **[FE-06]** Implementar sidebar de ações do T-031 com RBAC (D06 §2.4): **ANALISTA:** botão "Bloquear Caso" (vermelho outline — abre T-034 modal) + botão "Qualificar Caso" (primário, habilitado apenas quando `dossie_progress.verified = 6` AND adimplência confirmada) + botão "Solicitar documentos" (secondary); **COORDENADOR:** mesmos + "Cancelar Caso" (destructive); **MASTER:** igual COORDENADOR + "Publicar Oferta" diretamente. `ActionButton` em todos (spinner + desabilitado durante request). Verificar: "Qualificar Caso" desabilitado quando `dossie_progress.verified < 6`; ANALISTA não vê "Cancelar Caso".

### Feature: T-032 — Triagem Aba Dossiê

- [x] **[FE-07]** Implementar `TriagemDossieTab` (T-032): barra de progresso "X/6 documentos verificados" no topo; lista dos 6 documentos com nome, tipo esperado, data upload, operador que verificou, status (Pendente ⏳ / Verificado ✅ / Rejeitado ❌); 3 estados de linha: `dossiê completo` = barra 100% + banner verde "Dossiê completo. Todos os 6 documentos verificados."; `documento rejeitado` = fundo `--destructive/10` + ícone ❌ + motivo rejeição inline; `aguardando upload` = label "Pendente de envio pelo Cedente" (cinza). Mobile: lista de cards em vez de tabela. Verificar: barra atinge 100% quando 6 documentos verificados; documento rejeitado tem fundo destrutivo com motivo visível.

- [x] **[FE-08]** Implementar ações por documento na `TriagemDossieTab`: "Visualizar" → abre PDF em modal ou aba nova; "Verificar" → `PATCH /v1/cases/:id/dossie/:doc_id/verify` com `ActionButton` (spinner + desabilitado); após verificar → carimbo exibido inline: "Verificado por {nome} em {data/hora}" (imutável — não há botão de reverter — RN-056); "Rejeitar" → abre modal com campo de motivo obrigatório; "Solicitar reenvio" → visível quando status = REJEITADO; botão "Adicionar documento" para upload (máx 10MB, PDF/JPEG/PNG). Verificar: após verificar, botão "Verificar" some e carimbo aparece imutável; rejeição sem motivo não submete.

- [x] **[FE-09]** Implementar indicador de progresso salvo automaticamente (RN-057): ao retornar ao caso após navegar → exibir badge no topo da aba: "Verificação em andamento — {X} de 6 documentos revisados" quando `dossie_progress.in_progress = true`; badge some quando `verified = 6`. Verificar: badge aparece ao retornar a caso com triagem parcial; some quando todos verificados.

### Feature: T-033 — Triagem Aba Adimplência

- [x] **[FE-10]** Implementar `TriagemAdimplenciaTab` (T-033): checklist das 3 últimas parcelas do contrato com status (comprovante anexado / verificado / em atraso); campo de observações do Analista; 3 estados: `verificado` = banner verde "Adimplência confirmada."; `inadimplente` = banner vermelho "Inadimplência detectada. Bloqueie o caso."; `aguardando comprovante` = banner amarelo "Aguardando comprovante das parcelas."; botão "Confirmar adimplência" (primário) + botão "Registrar inadimplência" (vermelho outline — abre T-034). Verificar: 3 estados renderizam banners corretos; "Confirmar adimplência" habilita botão "Qualificar Caso" na sidebar.

- [x] **[FE-11]** Implementar exceção de adimplência para COORDENADOR na T-033 (RN-001.a): toggle "Aprovar via confirmação da construtora" visível **apenas** para COORDENADOR/MASTER; ao ativar: campo obrigatório "Número do protocolo / referência da confirmação" + upload do documento da construtora (máx 10MB); botão "Aplicar exceção" → `PATCH /v1/cases/:id/approve-exception` com spinner; após aprovação → caso avança para QUALIFICADO + toast "Exceção de adimplência aprovada. Caso qualificado.". Verificar: ANALISTA não vê toggle; toggle ativado por COORDENADOR com protocolo vazio não submete.

### Feature: T-034 — Modal Confirmar Bloqueio

- [x] **[FE-12]** Implementar `BlockCaseModal` (T-034): modal centralizado (`--radius-xl`, `max-w-sm`); título "Confirmar bloqueio por inadimplência"; corpo "O Cedente será notificado automaticamente com orientações para regularização."; checkbox obrigatório "Confirmo que as parcelas em atraso foram identificadas no comprovante." (desabilita "Confirmar bloqueio" se não marcado); botão "Confirmar bloqueio" (`--destructive`, primário) + botão "Cancelar" (secondary); `role="dialog"`, `aria-modal="true"`, focus trap, fechamento via Escape; estado `loading` = spinner no botão "Confirmar bloqueio" + desabilitado; estado `sucesso` = modal fecha + toast "Caso bloqueado. Cedente notificado." + status do caso atualizado na tela. Verificar: checkbox não marcado desabilita botão de confirmação; Escape fecha modal; sucesso fecha e exibe toast.

---

## 🔗 WIRING

- [x] **[WIRE-01]** Implementar `triagem.service.ts` no frontend: `getTriagemQueue()` → `GET /v1/triagem`; `getDossie(caseId)` → `GET /v1/cases/:id/dossie`; `uploadDocument(caseId, dto)` → `POST /v1/cases/:id/dossie` (multipart); `verifyDocument(caseId, docId)` → `PATCH /v1/cases/:id/dossie/:doc_id/verify`; `rejectDocument(caseId, docId, reason)` → `PATCH /v1/cases/:id/dossie/:doc_id/reject`; `blockCase(caseId, reason)` → `PATCH /v1/cases/:id/block`; `unblockCase(caseId)` → `PATCH /v1/cases/:id/unblock`; `qualifyCase(caseId)` → `PATCH /v1/cases/:id/qualify`; `approveException(caseId, dto)` → `PATCH /v1/cases/:id/approve-exception`. Verificar: `uploadDocument` usa `multipart/form-data`; `rejectDocument` com `reason` < 20 chars lança erro antes de enviar.

- [x] **[WIRE-02]** Integrar `triagem.service.ts` com componentes: `TriagemPage` usa `useQuery(['triagem'])` para listar fila; `TriagemDossieTab` usa `useQuery(['dossie', caseId])` para dossiê; mutações com `useMutation` + `onSuccess: () => queryClient.invalidateQueries(['dossie', caseId])`; após bloquear → invalidar `['cases', caseId]` e `['triagem']`. Verificar: verificar documento atualiza barra de progresso imediatamente via invalidação de query.

---

## ✅ TESTES

- [x] **[TEST-01]** Testes unitários backend: `POST /v1/cases/:id/dossie` com arquivo >10MB retorna 422; reenvio de documento rejeitado muda status para PENDENTE e preserva histórico; `PATCH /v1/cases/:id/dossie/:doc_id/verify` cria carimbo imutável; `PATCH /v1/cases/:id/qualify` com menos de 6 documentos verificados retorna 422; `unblock` sem comprovante atualizado retorna 422. Cobertura: 80% de branches.

- [x] **[TEST-02]** Testes unitários componentes frontend: `BlockCaseModal` desabilita botão sem checkbox marcado; carimbo de verificação é somente leitura no DOM; barra de progresso atualiza ao verificar documento; alerta FIFO aparece ao selecionar caso não-prioritário.

- [x] **[TEST-03]** Testes de integração: `PATCH /v1/cases/:id/qualify` com 6 documentos VERIFICADOS → caso avança para QUALIFICADO + evento registrado em audit; `PATCH /v1/cases/:id/block` → status muda para BLOQUEADO + job de notificação ao Cedente disparado; `PATCH /v1/cases/:id/approve-exception` (COORDENADOR) com protocolo → caso avança para QUALIFICADO + exceção registrada em audit.

- [x] **[TEST-04]** Testes E2E (Playwright): fluxo Triagem — login ANALISTA → `/triagem` → selecionar caso → aba Dossiê → verificar 6 documentos → aba Adimplência → confirmar adimplência → botão "Qualificar Caso" habilitado → clicar → caso avança para QUALIFICADO + toast.

---

## 🔍 AUTO-VERIFICAÇÃO S4b (12 checks)

- [x] **[CHECK-01]** `GET /v1/triagem` retorna casos em EM_TRIAGEM e BLOQUEADO ordenados por `entered_current_status_at ASC` (FIFO)
- [x] **[CHECK-02]** Carimbo de verificação de documento é imutável: `PATCH /v1/cases/:id/dossie/:doc_id/verify` aplicado → nenhum endpoint permite alterar `verified_by` ou `verified_at`
- [x] **[CHECK-03]** Reenvio de documento rejeitado: status muda para PENDENTE, histórico preserva versão rejeitada
- [x] **[CHECK-04]** Upload de arquivo >10MB retorna 422
- [x] **[CHECK-05]** `PATCH /v1/cases/:id/qualify` com 5 documentos verificados retorna 422 com mensagem "Todos os 6 documentos devem estar verificados"
- [x] **[CHECK-06]** `PATCH /v1/cases/:id/unblock` sem comprovante de parcela atualizado retorna 422
- [x] **[CHECK-07]** Bloqueio registra notificação ao Cedente na fila de mensagens
- [x] **[CHECK-08]** Alerta FIFO: selecionar segundo caso da fila exibe banner amarelo com ID do caso mais antigo
- [x] **[CHECK-09]** Toggle "Aprovar via confirmação da construtora" invisível para ANALISTA, visível para COORDENADOR
- [x] **[CHECK-10]** Job cron de 60 dias: caso BLOQUEADO há 60 dias gera alerta ao Coordenador com mensagem correta
- [x] **[CHECK-11]** `PATCH /v1/cases/:id/dossie/:doc_id/reject` com `rejection_reason` < 20 chars retorna 422
- [x] **[CHECK-12]** Todos os REQs de S4b (REQ-017, 031, 039–041, 056–062, 121–125, 201–205, 296–299) cobertos
