# S5 — Negociação

## Módulo Admin — Repasse Seguro

| Campo                | Valor                                                                               |
| -------------------- | ----------------------------------------------------------------------------------- |
| **Sprint**           | S5                                                                                  |
| **Nome**             | Negociação                                                                          |
| **Tipo**             | Dinâmica — Módulo Fullstack                                                         |
| **Template**         | B                                                                                   |
| **Docs Consultados** | D01.1, D01.2, D01.3, D06, D16                                                       |
| **Cross-cutting**    | D10 (Glossário), D02 (Stacks)                                                       |
| **REQs cobertos**    | REQ-043 a REQ-054, REQ-109 a REQ-120, REQ-128, REQ-206 a REQ-215, REQ-300 a REQ-306 |
| **Total de itens**   | 48                                                                                  |

---

> **Critério de conclusão de S5:** Fluxo completo de proposta funcional (submissão → validação de piso → apresentação ao Cedente → aceite/recusa/contraproposta); limite de 3 rodadas com escalação obrigatória ao Coordenador; prazo de resposta 3/5 dias úteis com expiração automática; propostas de múltiplos Cessionários gerenciadas em paralelo; anonimato absoluto Cedente/Cessionário; escalonamento de cenário D→C→B→A com Termo de Aceite via ZapSign; polling de 30s; histórico de propostas imutável.

---

## ⚙️ BACKEND

### Feature: Propostas e Contrapropostas

- [x] **[BE-01]** Implementar `GET /v1/cases/:id/proposals` (ANALISTA): retorna todas as propostas do caso com campos `cessionario_id`, `proposed_value`, `expires_at`, `round_number`, `status` (SUBMETIDA/APRESENTADA_CEDENTE/ACEITA/RECUSADA/CONTRAPROPOSTA_ATIVA/ESCALADA/EXPIRADA/SUPERADA), `counterproposals`; ordenado por `created_at DESC`; anonimato: retorna `cessionario_id` mas **não** `cessionario.name`, `cessionario.cpf`, `cessionario.phone` para ANALISTA (RN-144 — apenas `cessionario_id` + `alias: "Cessionário Interessado #N"`). Verificar: ANALISTA não recebe dados pessoais do Cessionário em fase de proposta; alias gerado sequencialmente por caso.

- [x] **[BE-02]** Implementar `POST /v1/cases/:id/proposals` (ANALISTA): campos `cessionario_id`, `proposed_value`, `expires_at?`; validações: (1) caso deve estar em status OFERTA_ATIVA; (2) `proposed_value` >= valor mínimo do cenário (RN-028/RN-146) — se abaixo → 422 `"A proposta mínima para este caso é R$ {valor_minimo}. Ajuste sua proposta para continuar."`; (3) verificar rate limiting (RN-031): Cessionário com 3 propostas ativas simultâneas → 422; >10 propostas em 24h → bloquear por 6h + alerta ao Coordenador; (4) se `proposed_value > Tabela Atual × 1.30` → criar alerta informativo ao Analista (não bloqueia); muda status caso para EM_NEGOCIACAO; registrar em audit. Verificar: proposta abaixo do piso retorna 422; Cessionário com 3 propostas ativas retorna 422.

- [x] **[BE-03]** Implementar `PATCH /v1/cases/:id/proposals/:proposal_id/accept` (ANALISTA): valida que proposta está em status APRESENTADA_CEDENTE ou APRESENTADA_CESSIONARIO; registra aceite de negociação; muda status da proposta para ACEITA; muda status do caso para EM_FORMALIZACAO; **encerra automaticamente todas as demais propostas ativas** do caso com status SUPERADA (RN-030) e notifica Cessionários superados: "Sua proposta para o caso {case_id} foi superada por outra proposta aceita pelo Cedente."; registrar em audit; histórico imutável (RN-029). Verificar: aceitar proposta encerra todas as demais com status SUPERADA; Cessionários superados recebem notificação.

- [x] **[BE-04]** Implementar `PATCH /v1/cases/:id/proposals/:proposal_id/reject` (ANALISTA): muda proposta para RECUSADA; se não houver outras propostas ativas → caso retorna para OFERTA_ATIVA; registrar em audit. Verificar: recusa com outras propostas ativas mantém caso em EM_NEGOCIACAO; recusa sem outras propostas move caso para OFERTA_ATIVA.

- [x] **[BE-05]** Implementar `POST /v1/cases/:id/proposals/:proposal_id/counterproposals` (ANALISTA): campos `counter_value`, `reason?`; incrementa `round_number`; se `round_number >= 3` e esta é a 3ª rodada sem acordo → criar nova contraproposta é permitido, mas ao chegar na 3ª rodada → **bloquear início de 4ª rodada** (RN-145/RN-026); se ANALISTA tenta criar contraproposta quando `round_number >= 3` → 422 `"Esta negociação atingiu o limite de 3 rodadas. A escalação ao Coordenador é obrigatória para prosseguir."` + mudar proposta para status ESCALADA; notificar Coordenador: "A negociação do caso {case_id} atingiu o limite de 3 rodadas. O caso foi escalado para mediação ativa."; registrar em audit. Verificar: 4ª contraproposta retorna 422; notificação ao Coordenador disparada na escalação.

- [x] **[BE-06]** Implementar `GET /v1/cases/:id/proposals/:proposal_id/counterproposals` (ANALISTA): retorna histórico de contrapropostas da proposta; imutável após criação. Verificar: nenhum endpoint permite editar/excluir contraproposta existente.

- [x] **[BE-07]** Implementar job de expiração de propostas por rodada (RN-027): cron diário verifica propostas APRESENTADA_CEDENTE ou APRESENTADA_CESSIONARIO; no 3º dia útil sem resposta → enviar lembrete automático à parte que não respondeu; no 5º dia útil sem resposta → encerrar rodada como EXPIRADA e executar ação diferenciada: se Cedente não respondeu → proposta para Cessionário como EXPIRADA + "Sua proposta de R$ {valor} para o caso {case_id} não obteve resposta do Cedente no prazo de 5 dias úteis."; se Cessionário não respondeu → proposta EXPIRADA + notifica Cedente; se não há outras propostas → caso volta para OFERTA_ATIVA; registrar em audit. Verificar: após 5 dias úteis sem resposta do Cedente, proposta muda para EXPIRADA e caso volta para OFERTA_ATIVA.

### Feature: Escalonamento de Cenário

- [x] **[BE-08]** Implementar `POST /v1/cases/:id/escalation` (COORDENADOR — sugestão ao Cedente): (1) gerar simulação comparativa: cenário atual vs. cenário inferior (D→C, C→B, B→A — RN-025); simulação inclui valores de Valor Recuperado estimado para cada cenário; (2) verificar se há proposta ativa em negociação → se sim, enfileirar escalonamento (RN-025.a) e notificar Cedente: "Seu pedido de escalonamento será processado após a conclusão da negociação em andamento."; (3) se não há proposta ativa → enviar simulação ao Cedente imediatamente; (4) se caso já está no Cenário A → 422 "Este caso já está no cenário mais acessível. Não há escalonamento disponível."; registrar em audit. Verificar: escalonamento com proposta ativa em negociação enfileira e não processa imediatamente; caso em Cenário A retorna 422.

- [x] **[BE-09]** Implementar `PATCH /v1/cases/:id/escalation/accept` (ANALISTA — após aceite do Cedente): (1) validar que escalonamento está pendente; (2) criar envelope ZapSign com "Termo de Aceite de Escalonamento" (RN-025); (3) após assinatura via webhook ZapSign → mudar cenário do caso para o inferior; (4) reiniciar timer SLA de 30 dias para OfertaAtiva→EmNegociacao (REQ-128/RN-060); (5) registrar em audit. Verificar: após aceite do Cedente, cenário do caso muda; timer SLA reinicia a partir da data do aceite.

- [x] **[BE-10]** Implementar job de detecção de estagnação (RN-025): cron a cada 12h verifica casos em OFERTA_ATIVA há mais de 30 dias corridos sem proposta recebida → gerar alerta "Estagnação" no pipeline do Coordenador. Verificar: caso em OFERTA_ATIVA há 31 dias aparece com alerta de Estagnação no pipeline.

---

## 🖥️ FRONTEND

### Feature: T-040 — Negociação Lista de Casos

- [x] **[FE-01]** Implementar `NegociacaoPage` (T-040, rota `/negociacao`): layout master-detail igual a T-030; lista casos em status EM_NEGOCIACAO e OFERTA_ATIVA; colunas: ID, imóvel, estado, cenário, Cessionário(s) com proposta (alias anônimo), valor da melhor proposta, prazo da proposta, SLA; ANALISTA: apenas seus casos; polling de 30 segundos (RN-132); estados: `loading` (skeleton), `vazio` ("Nenhum caso em negociação no momento."), `erro` (alert + "Tentar novamente"). Verificar: ANALISTA não vê casos de outros analistas; polling de 30s funciona.

### Feature: T-041 — Negociação Painel do Caso

- [x] **[FE-02]** Implementar `NegociacaoCasePage` (T-041, rota `/negociacao/:id`): layout igual T-031 (60% abas + 40% sidebar); 3 abas: Timeline de Propostas (T-042), Informações do Caso, Histórico; sidebar de ações RBAC: **ANALISTA** = "Aceitar proposta" (CTA primário — abre T-043), "Recusar / Contraproposta" (secondary — abre T-044), "Escalar para Coordenador"; **COORDENADOR** = mesmos + "Aprovar Escalonamento de Cenário" (se aplicável — abre T-045), "Cancelar negociação"; alerta proposta expirada: banner amarelo "Proposta expirada em {data}. Aguardando nova proposta do Cessionário." Verificar: ANALISTA não vê "Aprovar Escalonamento de Cenário"; banner de proposta expirada aparece quando `status = EXPIRADA`.

- [x] **[FE-03]** Implementar indicador de rodadas na sidebar de negociação (RN-026/RN-145): barra de progresso com 3 segmentos; segmento preenchido = rodada concluída; ao atingir 3/3 → barra vermelha + label "Escalação obrigatória"; contador regressivo de resposta: "X dias restantes para resposta." Verificar: após 3 rodadas barra fica vermelha e botão de nova rodada some.

### Feature: T-042 — Negociação Timeline de Propostas

- [x] **[FE-04]** Implementar `NegociacaoTimeline` (T-042): timeline vertical cronológica; cada entrada: tipo (Proposta / Contraproposta / Aceite / Recusa / Expirada / Superada), valor, autor (alias anônimo para Cessionário — RN-144), data/hora, status; proposta ativa: borda `--primary` + badge "Ativa" + countdown regressivo se <24h; propostas superadas: badge "Superada" cinza + linha tachada sobre o valor; campo de contraproposta pré-preenchido com último valor ao abrir (DEC-008); se `counter_value == valor_anterior` → aviso "O valor informado é igual ao anterior. Confirma?". Responsividade: mobile = timeline simplificada (tipo + valor + status). Verificar: dados de identidade do Cessionário nunca aparecem na timeline; contraproposta igual ao anterior exibe aviso.

### Feature: T-043 — Modal Aceitar Proposta

- [x] **[FE-05]** Implementar `AcceptProposalModal` (T-043): trigger = botão "Aceitar proposta" em T-041; exibe resumo: valor da proposta, alias do Cessionário (anônimo — RN-144), cenário do Cedente, comissão calculada automaticamente (Cedente por cenário + Cessionário sobre Delta — RN-018/019); checkbox obrigatório "Confirmo os valores calculados."; botão "Confirmar aceite" (primário) + "Cancelar" (secondary); `role="dialog"`, `aria-modal="true"`, focus trap, Escape fecha; após aceite: caso muda para EM_FORMALIZACAO + toast "Negociação concluída. Caso movido para Formalização." + redireciona para T-050. Verificar: comissão calculada corretamente por cenário; aceite redireciona para `/formalizacao/{id}`.

### Feature: T-044 — Modal Recusar / Contraproposta

- [x] **[FE-06]** Implementar `RejectOrCounterModal` (T-044): trigger = "Recusar / Contraproposta" em T-041; 2 abas no modal: "Recusar" e "Contraproposta"; aba Recusar: botão "Confirmar recusa" → `PATCH /reject`; aba Contraproposta: campo de valor pré-preenchido com último valor (DEC-008) + campo `reason?`; se `counter_value == valor_anterior` → aviso inline "O valor informado é igual ao anterior. Confirma?"; se rodada = 3ª → desabilita aba Contraproposta + label "Limite de rodadas atingido. Use 'Escalar para Coordenador'." Verificar: 3ª rodada desabilita aba de contraproposta; campo pré-preenchido com valor anterior.

### Feature: T-045 — Modal Aprovar Escalonamento

- [x] **[FE-07]** Implementar `ApproveEscalationModal` (T-045): trigger = "Aprovar Escalonamento de Cenário" em T-041 (apenas COORDENADOR/MASTER); exibe tabela comparativa lado a lado: cenário atual vs. cenário inferior com valores em destaque (verde = maior, vermelho = menor); botões "Aceitar Escalonamento" + "Recusar"; se Cedente já está no Cenário A → botão não exibido + mensagem "Este caso já está no cenário mais acessível. Não há escalonamento disponível."; ao aceitar → `POST /v1/cases/:id/escalation/accept` + toast "Escalonamento enviado para assinatura via ZapSign.". Verificar: caso no Cenário A não exibe opção de escalonamento; tabela comparativa mostra valores lado a lado.

---

## 🔗 WIRING

- [x] **[WIRE-01]** Implementar `negociacao.service.ts` no frontend: `getProposals(caseId)` → `GET /v1/cases/:id/proposals`; `createProposal(caseId, dto)` → `POST /v1/cases/:id/proposals`; `acceptProposal(caseId, proposalId)` → `PATCH /accept`; `rejectProposal(caseId, proposalId)` → `PATCH /reject`; `createCounterproposal(caseId, proposalId, dto)` → `POST /counterproposals`; `createEscalation(caseId)` → `POST /escalation`; `acceptEscalation(caseId)` → `PATCH /escalation/accept`. Verificar: `createProposal` com valor abaixo do piso captura erro 422 e exibe mensagem com valor mínimo.

- [x] **[WIRE-02]** Integrar polling de 30s na `NegociacaoPage` (RN-132): `useInterval(30000, () => queryClient.invalidateQueries(['proposals', caseId]))`; exibir timestamp "Atualizado às HH:MM:SS" (RN-132). Verificar: proposta aceita por outro operador atualiza automaticamente em até 30s.

---

## ✅ TESTES

- [x] **[TEST-01]** Testes unitários backend: proposta abaixo do piso retorna 422 com mensagem correta; aceitar proposta encerra todas as demais com status SUPERADA; 4ª contraproposta retorna 422; caso volta para OFERTA_ATIVA após última proposta expirada; rate limiting: 4ª proposta simultânea do Cessionário retorna 422. Cobertura: 80% de branches.

- [x] **[TEST-02]** Testes unitários componentes: `AcceptProposalModal` calcula comissão corretamente para cenário B (20% × (Valor Recuperado − Distrato Ref.)); `NegociacaoTimeline` não exibe nome/CPF do Cessionário; indicador de rodadas fica vermelho após 3 rodadas.

- [x] **[TEST-03]** Testes de integração: fluxo completo proposta → contraproposta (3 rodadas) → escalação ao Coordenador; job de expiração marca proposta como EXPIRADA após 5 dias úteis; caso volta para OFERTA_ATIVA após todas as propostas encerradas; aceite encerra propostas paralelas como SUPERADA e notifica Cessionários.

- [x] **[TEST-04]** Testes E2E (Playwright): fluxo Negociação — ANALISTA cria proposta → COORDENADOR visualiza na tela T-041 → proposta aceita → caso avança para EM_FORMALIZACAO → redirecionado para T-050.

---

## 🔍 AUTO-VERIFICAÇÃO S5 (12 checks)

- [x] **[CHECK-01]** Proposta abaixo do piso retorna 422 com valor mínimo correto por cenário
- [x] **[CHECK-02]** Aceitar proposta encerra automaticamente todas as demais propostas ativas com status SUPERADA e notifica Cessionários superados
- [x] **[CHECK-03]** 4ª contraproposta pelo ANALISTA retorna 422 com mensagem de escalação obrigatória
- [x] **[CHECK-04]** Escalação ao Coordenador: notificação gerada + proposta muda para status ESCALADA
- [x] **[CHECK-05]** Job de expiração: proposta sem resposta do Cedente em 5 dias úteis → EXPIRADA + notificação ao Cessionário
- [x] **[CHECK-06]** Rate limiting: Cessionário com 3 propostas ativas tenta 4ª → 422
- [x] **[CHECK-07]** Anonimato absoluto: dados de identidade do Cessionário (nome, CPF, e-mail) não retornam em `GET /v1/cases/:id/proposals` para ANALISTA
- [x] **[CHECK-08]** Timeline T-042: Cessionário exibido como "Cessionário Interessado #N" sem dados pessoais
- [x] **[CHECK-09]** `AcceptProposalModal` calcula corretamente comissão Cedente = 20% × (Valor Recuperado − Distrato Ref.) para cenários B/C/D
- [x] **[CHECK-10]** Escalonamento em Cenário A retorna 422 "Este caso já está no cenário mais acessível"
- [x] **[CHECK-11]** Polling de 30s na `NegociacaoPage` executa re-fetch das propostas automaticamente
- [x] **[CHECK-12]** Todos os REQs de S5 (REQ-043–054, REQ-109–120, REQ-128, REQ-206–215, REQ-300–306) cobertos
