# S4 — Marketplace e Propostas

## Sprint 4 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S4 — Marketplace e Propostas                                                                                     |
| **Template**       | B — Módulo Fullstack (feature com vertical slice Banco→Backend→Frontend→Wiring→Testes)                           |
| **REQs cobertos**  | S4-001 a S4-034 (34 requisitos do Registro Mestre)                                                               |
| **Docs fonte**     | 01.2 - RN Core e Receita · 06 - Mapa de Telas · 09 - Contratos de UI · 16 - Documentação de API · 10 - Glossário |
| **Total de itens** | 54 itens                                                                                                         |
| **Status**         | Concluída                                                                                                        |

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos: `opportunities`, `proposals`, `OpportunityStatus`, `ProposalStatus`, códigos OPR-XXXX-NNNN, telas T-OPR-01/02/03 e T-PRP-01/02.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — 9 campos exibidos no card de oportunidade; limite 3 propostas simultâneas; janela deslizante 10/24h; comissão 20% Δ; TTL Redis quota 1min.
- [x] ✅ Check 4 — Glossário: `delta_value`, `commissionBuyer`, anonimato do Cedente, `OPR-XXXX-NNNN`, `quotaBar`.
- [x] ✅ Check 5 — Anti-scaffold R10: cálculo de comissão real, validação de quota Redis real, filtros reais.
- [x] ✅ Check 6 — Máquinas de estado: ProposalStatus (ENVIADA→EM_ANALISE→ACEITA/RECUSADA/EXPIRADA/CANCELADA) e OpportunityStatus (DISPONIVEL→COM_PROPOSTA→EM_NEGOCIACAO→...) completamente documentadas.
- [x] ✅ Check 7 — TTL `rs:proposta:quota:{id}` 1min; `rs:oportunidades:lista` 5min; `rs:oportunidade:{id}` 10min.
- [x] ✅ Check 8 — Cross-módulo: proposta ACEITA inicia negociação (S5).
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — Sem ambiguidades.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S4-001 a S4-034 cobertos.

---

## FEATURE 1 — Marketplace de Oportunidades (T-OPR-01, T-OPR-02)

### 🗄️ Banco

- [x] **S4-B01** · Confirmar que `opportunities` table existe com todos os campos incluindo `delta_value DECIMAL(15,2) GENERATED ALWAYS AS (current_table_value - contract_table_value) STORED`, `embedding vector(1536)`, `idx_opportunities_embedding` HNSW, `idx_opportunities_status`, `idx_opportunities_city_state`, `idx_opportunities_delta_value`. Confirmar ausência de `cedente_id` (anonimato estrutural — RN-063). (Doc 13 — model Opportunity)

- [x] **S4-B02** · Confirmar políticas RLS em `opportunities`: leitura permitida para todos os usuários autenticados com `status IN ('DISPONIVEL', 'COM_PROPOSTA')`; escrita restrita a `service_role` (apenas Admin e sistema criam/editam oportunidades). Verificar índice `idx_opportunities_city_state` para filtros por localização. (Doc 12 — ERD; Doc 13)

### ⚙️ Backend

- [x] **S4-BE01** · Implementar `GET /api/v1/opportunities` em `apps/api/src/modules/opportunities/`: guards `JwtAuthGuard` + `KycGuard`; paginação offset-based (`page`, `per_page` max 100 default 20); filtros: `?city=`, `?state=`, `?min_value=`, `?max_value=`, `?bedrooms=`, `?status=DISPONIVEL` (default); ordenação: `?sort=delta_value` (desc default), `?sort=published_at` (desc); retornar array com 9 campos por oportunidade: `code` (OPR-XXXX-NNNN), `city`, `neighborhood`, `state`, `development_name`, `bedrooms`, `area_sqm`, `has_garage`, `current_table_value`, `contract_table_value`, `delta_value`, `ai_risk_score`, `status`, `published_at`; NUNCA retornar `cedente_id` (anonimato absoluto); cache Redis `rs:oportunidades:lista:{hash_params}` TTL 5min; meta de paginação: `{ total, page, per_page, total_pages }`. (Doc 16 — GET /opportunities; Doc 01.2 — RN-018)

- [x] **S4-BE02** · Implementar `GET /api/v1/opportunities/:id` em `apps/api/src/modules/opportunities/`: guards `JwtAuthGuard` + `KycGuard`; retornar todos os campos da oportunidade + campos adicionais: `cedente_paid_percentage`, `cedente_paid_value`, `commission_buyer` (calculado: se `delta_value > 0` → `0.20 × delta_value`; se ≤ 0 → `0.20 × cedente_paid_value`), `appreciation_data`, `ai_risk_score`; NUNCA retornar nome, CPF, contato do Cedente; cache Redis `rs:oportunidade:{id}` TTL 10min; erro `NOT-FOUND-001` se não existe ou `deleted_at IS NOT NULL`. (Doc 16 — GET /opportunities/:id; Doc 01.2 — RN-018, RN-031)

- [x] **S4-BE03** · Implementar validação de anonimato do Cedente em `OpportunitiesService`: método `sanitizeOpportunity(opp)` que garante que os campos `cedente_id`, `cedente_name`, `cedente_cpf`, `cedente_contact` nunca aparecem em nenhum response; aplicar em todos os métodos que retornam oportunidades; cobrir com teste unitário que valida ausência de todos esses campos. (Doc 01.5 — RN-063; Doc 01.1 — RN-063)

### 🎨 Frontend

- [x] **S4-FE01** · Implementar tela `T-OPR-01 — Marketplace` em `apps/web/src/features/marketplace/pages/MarketplacePage.tsx` (rota `/oportunidades`): buscar via `GET /api/v1/opportunities` com TanStack Query; grid de cards `OpportunityCard` com paginação; filtros: cidade/estado (dropdown), faixa de valor (range slider), quartos (select); ordenação: por delta_value (default) ou data de publicação; skeleton screens durante loading; `EmptyState` "Nenhuma oportunidade disponível" com ação "Limpar filtros"; cada card exibe: código OPR-XXXX-NNNN, cidade/bairro, empreendimento, quartos, área, garagem, valor atual da tabela, delta_value com cor (verde se > 0, vermelho se ≤ 0), `AiRiskScoreBadge`; ao clicar no card → T-OPR-02. (Doc 06 — T-OPR-01; Doc 01.2 — RN-018)

- [x] **S4-FE02** · Implementar `OpportunityCard.tsx` em `apps/web/src/components/domain/`: exibir exatamente os 9 campos do marketplace (conforme D09 Contratos de UI): `code`, `city`/`neighborhood`, `development_name`, `bedrooms`, `area_sqm`, `has_garage`, `current_table_value`, `delta_value`, `ai_risk_score`; delta_value com formatação BRL e indicador visual (verde/vermelho/cinza); `AiRiskScoreBadge` exibe score 0-10 com gradiente cor; animação Framer Motion `AnimatePresence` ao entrar/sair da lista. (Doc 09 — contrato T-OPR-01)

- [x] **S4-FE03** · Implementar tela `T-OPR-02 — Detalhe da Oportunidade` em `apps/web/src/features/marketplace/pages/OpportunityDetailPage.tsx` (rota `/oportunidades/:id`): buscar via `GET /api/v1/opportunities/:id`; exibir todos os campos detalhados: além dos 9 do card, exibir `cedente_paid_percentage`, `cedente_paid_value`, `commission_buyer` (formatado em BRL), `appreciation_data` (gráfico se disponível), score de risco; NUNCA exibir dados do Cedente; botão "Fazer Proposta" → T-OPR-03 (bottom sheet); botão "Analisar com IA" → T-IA-01 (passando `opportunity_id`); skeleton durante loading; erro 404 → `EmptyState` "Oportunidade não disponível". (Doc 06 — T-OPR-02)

- [x] **S4-FE04** · Implementar `AiRiskScoreBadge.tsx` em `apps/web/src/components/domain/`: score de 0-10 com gradiente cor (0-3 verde, 4-6 âmbar, 7-10 vermelho); tooltip "Score de risco calculado pelo Analista de IA"; null-safe (exibir "—" se score não disponível). Implementar `QuotaBar.tsx`: barra de progresso mostrando `X de 3 propostas ativas` e `Y de 10 propostas hoje`; cores: verde < 80% do limite, âmbar ≥ 80%, vermelho = 100%. (Doc 01.3 — RN-042 widget; Doc 09)

---

## FEATURE 2 — Propostas (T-PRP-01, T-PRP-02, T-OPR-03)

### 🗄️ Banco

- [x] **S4-B03** · Confirmar que `proposals` table existe com índices: `idx_proposals_cessionario_id`, `idx_proposals_opportunity_id`, `idx_proposals_status`, `idx_proposals_cessionario_status` (composto em `cessionario_id, status`). Confirmar `soft_delete` (`deleted_at`) presente. Confirmar FK `opportunity_id → opportunities.id RESTRICT` e `cessionario_id → cessionarios.id RESTRICT` (nunca Cascade — preservar histórico). (Doc 13 — model Proposal)

- [x] **S4-B04** · Confirmar RLS em `proposals`: cessionário só lê/cria suas próprias propostas (`cessionario_id = auth.uid()`); Admin lê todas (`service_role`); update/delete apenas por `service_role` (sistema controla estados). Verificar índice composto usado em query de quota: `idx_proposals_cessionario_status` para `WHERE cessionario_id = X AND status = 'ENVIADA' AND deleted_at IS NULL`. (Doc 12 — ERD)

### ⚙️ Backend

- [x] **S4-BE04** · Implementar `POST /api/v1/proposals` em `apps/api/src/modules/proposals/`: guards `JwtAuthGuard` + `KycGuard` + `ReAuthGuard` (proposta é ação crítica — RN-007); body `{ opportunity_id: string, proposed_value: number, message?: string }`; validações: (1) `proposed_value` inteiro positivo em centavos (nunca float); (2) verificar quota Redis `rs:proposta:quota:{cessionario_id}` — máx 3 propostas com status `ENVIADA` ou `EM_ANALISE` simultâneas (retornar `RATE-001` se atingido); (3) janela deslizante: máx 10 propostas criadas nas últimas 24h (rastrear via Redis `rs:proposta:diario:{cessionario_id}` com TTL até fim da janela de 24h); (4) verificar que oportunidade está em status `DISPONIVEL`; calcular `commission_buyer` no backend (nunca confiar no frontend); definir `expires_at = NOW() + 72 horas` (SLA de análise); criar `proposals` com status `ENVIADA`; atualizar `opportunities.status = COM_PROPOSTA`; disparar NOT-CES-03 via NotificationService; retornar HTTP 201 com proposta criada. (Doc 16 — POST /proposals; Doc 01.2 — RN-020, RN-021, RN-022)

- [x] **S4-BE05** · Implementar cálculo de `commission_buyer` em `ProposalsService.calculateCommission()`: parâmetros `currentTableValue: number`, `contractTableValue: number`, `cedentePaidValue: number` (todos em centavos); fórmula: `delta = currentTableValue - contractTableValue`; se `delta > 0`: retornar `Math.round(0.20 * delta)`; se `delta <= 0`: retornar `Math.round(0.20 * cedentePaidValue)`; resultado inteiro em centavos; nunca usar `float` (usar inteiros puros); testar com 100% de cobertura. (Doc 01.2 — RN-031)

- [x] **S4-BE06** · Implementar `GET /api/v1/proposals` em `apps/api/src/modules/proposals/`: guards `JwtAuthGuard` + `KycGuard`; retornar propostas do cessionário autenticado; paginação offset-based; filtro `?status=`; ordenação por `created_at DESC`; incluir campos da oportunidade vinculada: `opportunity_code`, `opportunity_city`, `opportunity_state`; incluir `commission_buyer`, `expires_at`, `rejection_reason` se reprovada. (Doc 16 — GET /proposals)

- [x] **S4-BE07** · Implementar `GET /api/v1/proposals/:id` em `apps/api/src/modules/proposals/`: guards `JwtAuthGuard` + `KycGuard`; verificar que proposta pertence ao cessionário autenticado; retornar dados completos da proposta incluindo opportunity (sem dados do Cedente); erro `NOT-FOUND-001` se não existe; erro `PERM-001` se pertence a outro cessionário. (Doc 16 — GET /proposals/:id)

- [x] **S4-BE08** · Implementar `DELETE /api/v1/proposals/:id` (cancelamento) em `apps/api/src/modules/proposals/`: guards `JwtAuthGuard` + `KycGuard` + `ReAuthGuard` (cancelamento é ação crítica); verificar que `status === 'ENVIADA'` (só pode cancelar proposta em análise — `CONFLICT-001` se em outro estado); soft delete (`deleted_at = NOW()`); atualizar `status = CANCELADA`; se não há mais propostas ativas para a oportunidade, atualizar `opportunities.status = DISPONIVEL`; retornar HTTP 204. (Doc 16 — DELETE /proposals/:id; Doc 01.2 — RN-024)

- [x] **S4-BE09** · Implementar job cron de expiração de propostas em `apps/api/src/modules/proposals/jobs/proposal-expiry.job.ts`: cron `*/15 * * * *` (a cada 15min); buscar propostas com `status IN ('ENVIADA', 'EM_ANALISE')` e `expires_at < NOW()`; atualizar `status = EXPIRADA`; verificar e atualizar `opportunities.status` conforme necessário; disparar NOT-CES-03 (com status EXPIRADA) via NotificationService. (Doc 01.2 — RN-023; Doc 01.4 — RN-054: SLA proposta 48h análise, 72h expiração)

- [x] **S4-BE10** · Implementar `GET /api/v1/proposals/quota` em `apps/api/src/modules/proposals/`: guards `JwtAuthGuard` + `KycGuard`; retornar `{ simultaneous: { used: number, limit: 3 }, daily: { used: number, limit: 10, resets_at: ISO8601 } }`; usar cache Redis TTL 1min; usado pelo frontend para exibir `QuotaBar`. (Doc 01.2 — RN-020, RN-022; Doc 16)

### 🎨 Frontend

- [x] **S4-FE05** · Implementar tela `T-OPR-03 — Formulário de Proposta` como bottom sheet em `apps/web/src/features/marketplace/components/ProposalSheet.tsx` (abre sobre T-OPR-02): campos: `proposed_value` (input monetário em reais, convertido para centavos no submit), `message` (textarea opcional max 500 chars); exibir `commission_buyer` calculado em tempo real conforme usuário digita valor (usar `lib/commission.ts`); exibir aviso "Esta ação requer confirmação de identidade" se re-auth necessária; `QuotaBar` com status atual da quota; submit chama `POST /api/v1/proposals`; ao precisar de re-auth, abrir `ReAuthModal` → ao confirmar, prosseguir; sucesso → fechar sheet, toast "Proposta enviada!", atualizar oportunidade para COM_PROPOSTA; erros: `RATE-001` → "Limite de propostas atingido"; `CONFLICT-001` → "Esta oportunidade não está mais disponível". (Doc 06 — T-OPR-03; Doc 01.2 — RN-020, RN-021)

- [x] **S4-FE06** · Implementar tela `T-PRP-01 — Lista de Propostas` em `apps/web/src/features/proposals/pages/ProposalsPage.tsx` (rota `/propostas`): buscar via `GET /api/v1/proposals`; lista com filtro por status (tabs: Ativas, Aceitas, Recusadas, Expiradas); cada item: código OPR-XXXX-NNNN, valor proposto, commission_buyer, status com `ProposalStatusBadge`, countdown para `expires_at` se status=ENVIADA; swipe left (mobile) revela ação "Cancelar"; click → T-PRP-02; `EmptyState` por tab. (Doc 06 — T-PRP-01)

- [x] **S4-FE07** · Implementar tela `T-PRP-02 — Detalhe da Proposta` em `apps/web/src/features/proposals/pages/ProposalDetailPage.tsx` (rota `/propostas/:id`): buscar via `GET /api/v1/proposals/:id`; exibir: dados da oportunidade (sem Cedente), valor proposto, commission_buyer, status com badge, motivo de rejeição se RECUSADA, expires_at countdown se ENVIADA; botão "Cancelar Proposta" (visível apenas se `status === 'ENVIADA'`) → modal de confirmação → `ReAuthModal` → `DELETE /api/v1/proposals/:id`; botão "Ver Negociação" se `status === 'ACEITA'` e negociação existir. (Doc 06 — T-PRP-02)

- [x] **S4-FE08** · Implementar `ProposalStatusBadge.tsx` em `apps/web/src/components/domain/`: mapear todos os 6 estados de `ProposalStatus` com cor e label: ENVIADA (azul, "Enviada"), EM_ANALISE (âmbar, "Em análise"), ACEITA (verde, "Aceita"), RECUSADA (vermelho, "Recusada"), EXPIRADA (cinza, "Expirada"), CANCELADA (cinza escuro, "Cancelada"). Reutilizável em T-PRP-01, T-PRP-02 e Dashboard.

- [x] **S4-FE09** · Configurar subscription Supabase Realtime em `proposals` para atualização em tempo real: `cessionario_id=eq.{id}` filtro (RLS); ao receber UPDATE (ex: proposta aceita pelo Admin), atualizar TanStack Query cache, mostrar toast "Sua proposta foi aceita!" (NOT-CES-03); badge de propostas ativas atualizado no header. Degradar para polling 30s se Realtime indisponível.

### 🔌 Wiring

- [x] **S4-W01** · Configurar validação de quota Redis no `ProposalsService`: chave `rs:proposta:quota:{cessionario_id}` — count de propostas com `status IN ('ENVIADA', 'EM_ANALISE')` e `deleted_at IS NULL`, TTL 1min; chave `rs:proposta:diario:{cessionario_id}` — count de propostas criadas em 24h, TTL = segundos até meia-noite + 24h da criação; ao criar proposta: incrementar ambos; ao cancelar/expirar: decrementar quota simultânea; verificar antes de qualquer criação. (Doc 01.2 — RN-020, RN-022)

- [x] **S4-W02** · Garantir que `commission_buyer` é sempre calculado no backend em `ProposalsService.calculateCommission()` e nunca aceito do frontend; o frontend apenas exibe uma preview calculada localmente para UX — o valor real é o retornado pelo backend no response da proposta criada. Verificar que qualquer discrepância é ignorada (backend é fonte de verdade). (Doc 01.2 — RN-031)

### 🧪 Testes

- [x] **S4-T01** · Testes unitários `ProposalsService` — quota: criar 3 propostas simultâneas → 4ª retorna `RATE-001`; criar 10 propostas em 24h → 11ª retorna `RATE-001`; oportunidade não-DISPONIVEL → `CONFLICT-001`; quota Redis TTL 1min verificado via mock. Cobertura 100% branches.

- [x] **S4-T02** · Testes unitários `calculateCommission`: Δ positivo (500000-400000=100000 → comissão=20000); Δ zero → comissão=0.20×cedentePaidValue; Δ negativo → comissão=0.20×cedentePaidValue; inteiros em centavos sem float; 100% cobertura. (Doc 01.2 — RN-031)

- [x] **S4-T03** · Testes unitários `OpportunitiesService` — anonimato: método `sanitizeOpportunity` nunca retorna `cedente_id`, `cedente_name`, `cedente_cpf`; verificado com 5 cenários de oportunidade com dados fictícios do Cedente. (Doc 01.5 — RN-063)

- [x] **S4-T04** · Testes E2E Playwright — TC-CES-07 "Envio de proposta válida": login KYC_APROVADO → T-OPR-01 → selecionar oportunidade DISPONIVEL → T-OPR-03 → preencher valor → re-auth → submit → verificar status COM_PROPOSTA na oportunidade e ENVIADA na proposta; TC-CES-08 "Cancelamento de proposta": cancelar proposta ENVIADA → re-auth → verificar CANCELADA; TC-CES-09 "Limite de propostas simultâneas": criar 3 propostas → 4ª retorna mensagem de limite. (Doc 01.3 — TC-CES-07, TC-CES-08, TC-CES-09)

---

## 🔀 Cross-Módulo

- [x] **S4-CM01** · **[→ S5]** Quando Admin aceita proposta (via painel Admin — fora do escopo desta sprint), `proposals.status` muda para `ACEITA` e sistema cria registro em `negotiations` com `status = EM_NEGOCIACAO`. Garantir que `OpportunitiesService` atualiza `opportunities.status = EM_NEGOCIACAO` ao criar negociação. Placeholder para implementação em S5.

- [x] **S4-CM02** · **[← S3]** Dashboard widget "Resumo de Propostas" deve ser invalidado no cache Redis quando `proposals.status` muda. Garantir que `ProposalsService` emite evento que invalida `rs:dashboard:{cessionario_id}`.

---

## 📊 COBERTURA DE REQs S4

| REQ ID | Descrição                                    | Item(s)               |
| ------ | -------------------------------------------- | --------------------- |
| S4-001 | GET /opportunities com 9 campos              | S4-BE01, S4-FE01      |
| S4-002 | GET /opportunities/:id com commission        | S4-BE02, S4-FE03      |
| S4-003 | Anonimato Cedente em oportunidades           | S4-BE03, S4-T03       |
| S4-004 | Cache TTL 5min lista, 10min detalhe          | S4-BE01, S4-BE02      |
| S4-005 | POST /proposals com validações               | S4-BE04               |
| S4-006 | Quota 3 simultâneas Redis TTL 1min           | S4-BE04, S4-W01       |
| S4-007 | Janela deslizante 10/24h                     | S4-BE04, S4-W01       |
| S4-008 | commission_buyer calculado no backend        | S4-BE05, S4-W02       |
| S4-009 | expires_at = NOW() + 72h                     | S4-BE04               |
| S4-010 | GET /proposals (lista)                       | S4-BE06               |
| S4-011 | GET /proposals/:id (detalhe)                 | S4-BE07               |
| S4-012 | DELETE /proposals/:id (cancelamento)         | S4-BE08               |
| S4-013 | Job cron expiração 15min                     | S4-BE09               |
| S4-014 | GET /proposals/quota                         | S4-BE10               |
| S4-015 | T-OPR-01 Marketplace                         | S4-FE01               |
| S4-016 | OpportunityCard 9 campos                     | S4-FE02               |
| S4-017 | T-OPR-02 Detalhe Oportunidade                | S4-FE03               |
| S4-018 | AiRiskScoreBadge                             | S4-FE04               |
| S4-019 | QuotaBar                                     | S4-FE04               |
| S4-020 | T-OPR-03 Formulário Proposta                 | S4-FE05               |
| S4-021 | T-PRP-01 Lista Propostas                     | S4-FE06               |
| S4-022 | T-PRP-02 Detalhe Proposta                    | S4-FE07               |
| S4-023 | ProposalStatusBadge                          | S4-FE08               |
| S4-024 | Realtime proposals updates                   | S4-FE09               |
| S4-025 | proposed_value em centavos                   | S4-BE04               |
| S4-026 | re-auth em proposta e cancelamento           | S4-BE04, S4-BE08      |
| S4-027 | NOT-CES-03 em status proposta                | S4-BE04, S4-BE09      |
| S4-028 | Status ENVIADA→EM_ANALISE via Admin          | S4-BE04 (placeholder) |
| S4-029 | OpportunityStatus COM_PROPOSTA               | S4-BE04               |
| S4-030 | OpportunityStatus DISPONIVEL ao cancelar     | S4-BE08               |
| S4-031 | SLA análise 48h, expiração 72h               | S4-BE04, S4-BE09      |
| S4-032 | Filtros marketplace por cidade/valor/quartos | S4-BE01, S4-FE01      |
| S4-033 | TC-CES-07, TC-CES-08, TC-CES-09              | S4-T04                |
| S4-034 | proposed_value nunca float                   | S4-BE04, S4-BE05      |
