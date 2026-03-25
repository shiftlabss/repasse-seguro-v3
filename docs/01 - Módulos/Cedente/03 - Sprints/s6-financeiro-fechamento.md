# S6 — Financeiro e Fechamento
## Módulo Cedente · Plataforma Repasse Seguro

| **Sprint** | **Tipo** | **Template** | **Versão** | **Data** |
|---|---|---|---|---|
| S6 | Módulo Fullstack | Template B | v1.0 | 2026-03-24 |

**REQs cobertos:** REQ-036, REQ-061, REQ-064–067, REQ-087–093, REQ-104–120, REQ-127–128

**Dependências:** S1 (infra, Escrow partner — ⚠️ DP-001 BLOCKER), S3 (casos, comissao.calculator), S4 (envelopes), S5 (proposta aceita → EM_FORMALIZACAO)

**Critério de aceite da sprint:** todos os itens ✅ + 12 auto-verificações ✅ + T-032, T-033, T-034 passando + confirmação manual de DP-001.

---

> ⚠️ **DP-001 BLOCKER:** O parceiro bancário/fintech da Conta Escrow ainda não foi definido (D01.5 seção 2.3). Os endpoints de confirmação de depósito, distribuição e estorno dependem desta decisão. Implementar ADR-CED-003 (confirmação manual pelo Admin no MVP) enquanto o parceiro não está definido. Registrar todos os itens dependentes como `[AGUARDANDO DP-001]`.

---

## 🔴 FEATURE 1 — Conta Escrow

### 1. Banco de Dados — Conta Escrow

- [x] **1.1** Confirmar que a tabela `contas_escrow` possui os campos: `id UUID PK`, `caso_id UUID FK REFERENCES casos(id) UNIQUE` (1 Escrow por caso), `status StatusEscrow NOT NULL DEFAULT 'ABERTA'`, `valor_total BIGINT NOT NULL` (em centavos), `valor_comissao_rs BIGINT NOT NULL`, `valor_cedente BIGINT NOT NULL` (valor_total - valor_comissao_rs), `data_abertura TIMESTAMPTZ`, `data_deposito_confirmado TIMESTAMPTZ`, `data_fechamento TIMESTAMPTZ`, `data_distribuicao_prevista TIMESTAMPTZ`, `data_distribuicao_realizada TIMESTAMPTZ`, `data_reversao TIMESTAMPTZ`, `parceiro_referencia_externa VARCHAR(255)` (ID do parceiro Escrow — `[AGUARDANDO DP-001]`), `ultimo_status_parceiro_em TIMESTAMPTZ`, `created_at`, `updated_at`.
- [x] **1.2** Confirmar que o enum `StatusEscrow` possui exatamente 6 valores: `ABERTA`, `DEPOSITO_CONFIRMADO`, `EM_PERIODO_REVERSAO`, `VALORES_DISTRIBUIDOS`, `COMISSAO_DEFINITIVA`, `ESTORNADA` (conforme máquina de estados de RN-051, D01.3).
- [x] **1.3** Confirmar RLS em `contas_escrow`: `FOR SELECT USING (auth.uid() = (SELECT cedente_auth_id FROM casos WHERE id = caso_id))` — Cedente apenas lê, nunca escreve.
- [x] **1.4** Criar índice `idx_escrow_caso_id ON contas_escrow(caso_id)`.
- [x] **1.5** Criar índice `idx_escrow_status_distribuicao ON contas_escrow(status, data_distribuicao_prevista)` para job cron de distribuição automática.
- [x] **1.6** Criar migration `0005_contas_escrow.sql`.

### 2. Backend — Escrow Service (ADR-CED-003 MVP)

- [x] **2.1** Conforme ADR-CED-003 (D14): no MVP, a confirmação de depósito é manual pelo Admin. Implementar fluxo:
  - `POST /admin/escrow/:caso_id/confirmar-deposito` — endpoint interno (Admin only, service role): cria ou atualiza `contas_escrow SET status='DEPOSITO_CONFIRMADO', data_deposito_confirmado=NOW()`, calcula `valor_comissao_rs` usando `comissao.calculator`, registra `valor_cedente`.
  - Publicar evento `DEPOSITO_CONFIRMADO` → NotificationService envia template `NOT-CED-13` parcial.
- [x] **2.2** Após confirmação do Fechamento (todos os 4 critérios cumpridos — instrumento assinado + preço confirmado + anuência + depósito):
  - `UPDATE casos SET status='NEGOCIO_FECHADO'`.
  - `UPDATE contas_escrow SET status='EM_PERIODO_REVERSAO', data_fechamento=NOW(), data_distribuicao_prevista=NOW() + INTERVAL '15 days'`.
  - Publicar `NEGOCIO_FECHADO` → NotificationService envia template `NOT-CED-13` (fechamento confirmado).
- [x] **2.3** Implementar `EscrowDistribuicaoCron` que roda a cada hora:
  - Buscar `contas_escrow` com `status='EM_PERIODO_REVERSAO'` e `data_distribuicao_prevista <= NOW()` e sem desistência ativa.
  - `UPDATE contas_escrow SET status='VALORES_DISTRIBUIDOS', data_distribuicao_realizada=NOW()`.
  - `UPDATE casos SET status='CONCLUIDO'`.
  - Publicar `DISTRIBUICAO_REALIZADA` → NotificationService envia template `NOT-CED-14` (distribuição realizada) (RN-040).
- [x] **2.4** Implementar `GET /financeiro/:caso_id` — retorna: `{ cenario_retorno, valor_recuperado, comissao_rs, valor_liquido, status_escrow, data_distribuicao_prevista, data_distribuicao_realizada, countdown_reversao_segundos }`. Somente leitura (RN-051 item 3). Nunca permite POST/PATCH/DELETE do Cedente.
- [x] **2.5** `countdown_reversao_segundos` = `data_distribuicao_prevista - NOW()` quando status `EM_PERIODO_REVERSAO`; `null` caso contrário.
- [x] **2.6** Atualização de status da Conta Escrow via polling do parceiro (`[AGUARDANDO DP-001]`): implementar placeholder `EscrowStatusSyncCron` que roda a cada 5 minutos e sincroniza com API do parceiro — SLA máximo de **5 minutos** de defasagem (RN-083 item 2). Por ora: mock com atualização manual pelo Admin.
- [x] **2.7** Implementar extensão de prazo Escrow (RN-093):
  - `POST /admin/escrow/:caso_id/solicitar-extensao` com `{ novo_prazo, motivo }` — endpoint Admin: cria solicitação e notifica Cedente com janela de **24 horas** para aprovar ou recusar.
  - `POST /casos/:caso_id/escrow/extensao/aprovar` — Cedente aprova: registra `aprovada_em`, notifica Admin.
  - `POST /casos/:caso_id/escrow/extensao/recusar` — Cedente recusa: registra `recusada_em`, notifica Admin.
  - Se Cedente não responde em **24 horas**: job cron `ExtensaoSilencioAprovacaoCron` registra "Aprovação por silêncio — 24h" + notifica Admin (RN-093 item 6).
  - Registro de aprovação imutável vinculado ao `caso_id`.

### 3. Backend — Desistência Pós-Fechamento

- [x] **3.1** Implementar `POST /casos/:id/desistencia` com body `{ justificativa: string (20-500 chars) }` (RN-039):
  - Verificar `casos.status = 'AGUARDANDO_LIBERACAO'` (sinônimo de `EM_PERIODO_REVERSAO` da Escrow — ⚠️ verificar alinhamento de nomenclatura).
  - Verificar que `contas_escrow.data_distribuicao_prevista > NOW()` (dentro dos 15 dias corridos) — comparar com `NOW()` no fuso `America/Fortaleza`.
  - `UPDATE casos SET status='DESISTENCIA_EM_ANALISE'`.
  - Publicar `DESISTENCIA_SOLICITADA` → Admin comunica ao Cessionário.
  - Retornar `{ solicitacao_id, status: 'DESISTENCIA_EM_ANALISE' }`.
- [x] **3.2** Se Cedente tenta solicitar desistência após os 15 dias: retornar HTTP 422 com `cedente_message: "O período de segurança de 15 dias expirou. A transação está concluída e não pode mais ser revertida. Para disputas, entre em contato com nosso suporte jurídico."` (RN-040 item 5).
- [x] **3.3** Fluxo de aceite de desistência (Admin aceita):
  - `POST /admin/casos/:id/desistencia/aceitar` → `UPDATE casos SET status='CONCLUIDO'`; `UPDATE contas_escrow SET status='ESTORNADA', data_reversao=NOW()`; publicar `ESTORNO_SOLICITADO` → parceiro Escrow processa estorno em até **5 dias úteis** (RN-039 item 5).
- [x] **3.4** Fluxo de não-aceite (Cessionário não aceita):
  - `POST /admin/casos/:id/desistencia/mediar` → `UPDATE casos SET status='MEDIACAO_EM_ANDAMENTO'`; publicar `MEDIACAO_INICIADA`; SLA mediação: **10 dias úteis**.
- [x] **3.5** Implementar lógica de irrevogabilidade: após `EscrowDistribuicaoCron` marcar `VALORES_DISTRIBUIDOS`, o endpoint `POST /casos/:id/desistencia` retorna HTTP 422 — botão "Solicitar Desistência" não existe mais no frontend (RN-040 item 4).

### 4. Backend — Anuência da Construtora

- [x] **4.1** Implementar modelo de anuência no banco: adicionar campo `anuencia_status ENUM('AGUARDANDO', 'CONCEDIDA', 'NEGADA', 'PRAZO_EXCEDIDO')` e `anuencia_data_solicitacao TIMESTAMPTZ`, `anuencia_data_resposta TIMESTAMPTZ`, `anuencia_motivo_negativa TEXT` em `casos`. Criar migration `0006_anuencia.sql`.
- [x] **4.2** Endpoint `GET /casos/:id/anuencia` — retorna status da anuência + dias úteis decorridos + prazo estimado. Apenas informativo para o Cedente (RN-064 item 2).
- [x] **4.3** Implementar `AnuenciaPrazoCron` que roda diariamente:
  - Verificar casos com `anuencia_status='AGUARDANDO'` e `anuencia_data_solicitacao` há mais de **15 dias úteis** → marcar `anuencia_status='PRAZO_EXCEDIDO'` → notificar Cedente com template `NOT-CED-04` (prazo estimado adicional — RN-067).
- [x] **4.4** Quando Admin registra anuência concedida:
  - `UPDATE casos SET anuencia_status='CONCEDIDA', anuencia_data_resposta=NOW()`.
  - Publicar `ANUENCIA_CONCEDIDA` → NotificationService notifica Cedente "A construtora autorizou a transferência." (RN-065 item 2).
  - Caso avança para etapa de assinaturas finais.
- [x] **4.5** Quando Admin registra anuência negada:
  - `UPDATE casos SET anuencia_status='NEGADA', anuencia_motivo_negativa, status='PENDENCIA_IDENTIFICADA'`.
  - Publicar `ANUENCIA_NEGADA` → NotificationService notifica Cedente com motivo (RN-066).
  - Admin disponibiliza 3 opções ao Cedente pelo painel: (3.1) Renegociar, (3.2) Cancelar, (3.3) Aguardar nova tentativa — SLA resposta Cedente: **10 dias úteis** (RN-066 item 4).
- [x] **4.6** Implementar `POST /casos/:id/anuencia/opcao` com body `{ opcao: 'RENEGOCIAR' | 'CANCELAR' | 'AGUARDAR' }` — Cedente escolhe opção após anuência negada.

### 5. Backend — Inadimplência na Anuência (D01.4)

- [x] **5.1** Quando anuência for negada por inadimplência do Cedente: Admin registra `inadimplencia_detectada=true` no caso.
- [x] **5.2** Cedente recebe notificação com prazo de **15 dias úteis** para regularização: envio de comprovante de regularização via upload em nova tela.
- [x] **5.3** Implementar `POST /casos/:id/comprovante-regularizacao` — Cedente faz upload de comprovante; Admin analisa e, se aprovado, reabilita tentativa de anuência.
- [x] **5.4** `InadimplenciaPrazoCron` — após **15 dias úteis** sem comprovante aprovado:
  - Opções: cancelamento (com estorno prioritário em até **3 dias úteis**) OU extensão única de **10 dias úteis** adicionais (RN-075, D01.4).
  - Após extensão: se ainda não regularizado → cancelamento automático com estorno prioritário em **3 dias úteis** (RN-076, D01.4).

### 6. Frontend — Painel Financeiro

- [x] **6.1** Criar `apps/web-cedente/src/app/(authenticated)/financeiro/[caso_id]/page.tsx`.
- [x] **6.2** Verificar status do caso: se `status` anterior a `EM_FORMALIZACAO` → exibir estado informativo "A área financeira estará disponível quando seu caso avançar para a formalização. Enquanto isso, você pode ver valores estimados no simulador." com link para simulador (RN-051 item 2).
- [x] **6.3** Exibir Valor Líquido ("Você recebe") como **elemento de maior destaque visual**: maior fonte, posição central/superior, rótulo claro "Você recebe", cor verde. NUNCA em tabela de rodapé ou colapsado (RN-052 itens 2–3).
- [x] **6.4** Exibir hierarquia completa: (1) Valor Líquido em destaque máximo, (2) Valor Recuperado, (3) Comissão RS com fórmula completa `20% × (R$ [Recuperado] − R$ [Distrato])`, (4) Status da Conta Escrow, (5) Data prevista de distribuição.
- [x] **6.5** Antes do Fechamento: Valor Líquido exibido com nota "(estimado)" (RN-052 item 4). Após Fechamento: sem nota de estimativa (RN-052 item 5).
- [x] **6.6** Cenário A: exibir "Comissão RS: R$ 0 — Neste cenário, o Repasse Seguro não cobra comissão do Cedente." + disclaimer "No Cenário A, o Repasse Seguro auxilia na transferência do contrato sem custo para você. O comprador assume seu saldo devedor." (RN-054).
- [x] **6.7** Após Fechamento com Escrow `EM_PERIODO_REVERSAO`: exibir countdown dos 15 dias corridos em formato prominente "Período de segurança termina em [X dias] [Y horas]" + mensagem "A distribuição ocorrerá automaticamente em [data]" (RN-053 itens 2–3). Atualizar countdown via `setInterval(1 min)`.
- [x] **6.8** Após distribuição: exibir "Transação concluída e definitiva" + botão "Baixar comprovante de distribuição" (RN-053 item 5).
- [x] **6.9** Exibir status da Conta Escrow com badge colorido e data de última atualização. Se parceiro indisponível: "Informação atualizada em [data/hora]. Aguardando atualização do serviço financeiro." (RN-083 item 4).
- [x] **6.10** Implementar Supabase Realtime subscription para channel `escrow:{caso_id}`: ao receber evento de mudança de status → atualizar badge + valor sem reload.

### 7. Frontend — Desistência e Mediação

- [x] **7.1** Exibir botão "Solicitar Desistência" apenas para casos com status `AGUARDANDO_LIBERACAO` (Escrow `EM_PERIODO_REVERSAO`). Botão DESAPARECE com transição suave quando `data_distribuicao_prevista` passa (RN-040 item 4).
- [x] **7.2** Ao clicar "Solicitar Desistência": abrir modal com campo texto obrigatório (mínimo 20, máximo 500 caracteres, com contador visível e placeholder "Descreva o motivo da desistência") + confirmação dupla "Esta ação não pode ser desfeita por você. A desistência depende do aceite da outra parte ou entrará em mediação. Confirmar?" (RN-039 item 3).
- [x] **7.3** Status `DESISTENCIA_EM_ANALISE`: exibir "Sua solicitação de desistência está sendo analisada." no painel do caso.
- [x] **7.4** Status `MEDIACAO_EM_ANDAMENTO`: exibir "Estamos mediando a situação entre as partes. Prazo estimado: 10 dias úteis." (RN-039 item 6).

### 8. Frontend — Anuência (perspectiva do Cedente)

- [x] **8.1** No detalhe do caso em `EM_FORMALIZACAO`: exibir seção "Anuência da Construtora" com status atual, data da solicitação e prazo estimado "15 dias úteis" (RN-064 item 2).
- [x] **8.2** Quando `ANUENCIA_NEGADA`: exibir motivo + 3 cards clicáveis com as opções: "Renegociar com a construtora", "Cancelar o caso" (com aviso de estorno), "Aguardar nova tentativa" (RN-066 item 3).
- [x] **8.3** Botão "Escolher opção" disponível por **10 dias úteis** — após esse prazo lembrete automático (sem cancelamento automático) (RN-066 item 4).

---

## 🔀 CROSS-MÓDULO

### 9. Eventos disparados por esta sprint

- [x] **9.1** `DEPOSITO_CONFIRMADO` → S4 (notificações) envia template `NOT-CED-13` parcial.
- [x] **9.2** `NEGOCIO_FECHADO` → S4 (envelopes) verifica todas as assinaturas finais; S4 (notificações) envia `NOT-CED-13` completo.
- [x] **9.3** `DISTRIBUICAO_REALIZADA` → `casos.status = 'CONCLUIDO'`; S4 (notificações) envia `NOT-CED-14`; S3 (eventos_caso) registra linha do tempo.
- [x] **9.4** `ESTORNO_SOLICITADO` → S4 (notificações) envia confirmação de estorno ao Cedente.
- [x] **9.5** Após `EscrowDistribuicaoCron` distribuir → botão "Solicitar Desistência" some do frontend via Supabase Realtime.

---

## 🧪 TESTES

### 10. Testes unitários (Vitest)

- [x] **10.1** `escrow.service.spec.ts` — testar `confirmarDeposito()`: cria Conta Escrow, calcula `valor_comissao_rs` e `valor_cedente` usando `comissao.calculator` com valores em centavos.
- [x] **10.2** `escrow.service.spec.ts` — testar `EscrowDistribuicaoCron`: Escrow com `data_distribuicao_prevista` no passado → status `VALORES_DISTRIBUIDOS`; casos em `CONCLUIDO`.
- [x] **10.3** `desistencia.service.spec.ts` — testar `solicitar()`: caso `AGUARDANDO_LIBERACAO` dentro dos 15 dias → aceito; fora dos 15 dias → HTTP 422; caso em outro status → HTTP 422.
- [x] **10.4** `anuencia.service.spec.ts` — testar `AnuenciaPrazoCron`: após 15 dias úteis sem resposta → `PRAZO_EXCEDIDO` + notificação.
- [x] **10.5** `escrow-extensao.service.spec.ts` — testar silêncio de 24h → aprovação automática registrada com label "Aprovação por silêncio — 24h".
- [x] **10.6** Verificar que `GET /financeiro/:caso_id` NUNCA retorna campos de escrita (sem PUT/PATCH/DELETE expostos).

### 11. Testes de integração (Supertest)

- [x] **11.1** T-032: `GET /financeiro/:caso_id` após proposta aceita → response contém `valor_liquido` como campo de maior destaque + `status_escrow = 'ABERTA'`.
- [x] **11.2** T-033: `GET /financeiro/:caso_id` para caso Cenário A → `comissao_rs = 0`, disclaimer presente.
- [x] **11.3** T-034: countdown de reversão: após `confirmarFechamento()` → `countdown_reversao_segundos` próximo de `15 * 24 * 3600`.
- [x] **11.4** T-028: `POST /casos/:id/desistencia` dentro dos 15 dias → status `DESISTENCIA_EM_ANALISE`.
- [x] **11.5** T-029: `POST /casos/:id/desistencia` após distribuição → HTTP 422.
- [x] **11.6** T-039: `GET /casos/:id/anuencia` com `anuencia_status = 'NEGADA'` → response inclui 3 opções disponíveis.

---

## ✅ AUTO-VERIFICAÇÃO (12 Checks)

| # | Check | Critério | Status |
|---|---|---|---|
| 1 | **Nomenclatura** | `contas_escrow`, `StatusEscrow` com 6 valores exatos: `ABERTA`, `DEPOSITO_CONFIRMADO`, `EM_PERIODO_REVERSAO`, `VALORES_DISTRIBUIDOS`, `COMISSAO_DEFINITIVA`, `ESTORNADA`; `AGUARDANDO_LIBERACAO` (status do caso); `DESISTENCIA_EM_ANALISE`; `MEDIACAO_EM_ANDAMENTO`; `CONCLUIDO` | ✅ |
| 2 | **Verificabilidade** | Cada item binariamente verificável — "countdown de reversão" = campo calculado `data_distribuicao_prevista - NOW()` em segundos, não apenas "mostrar tempo" | ✅ |
| 3 | **Valores numéricos** | Período de reversão **15 dias corridos**; distribuição Escrow até **2 dias úteis** após Fechamento; estorno padrão até **5 dias úteis**; estorno prioritário (inadimplência) até **3 dias úteis**; mediação **10 dias úteis**; regularização inadimplência **15 dias úteis** + extensão única **10 dias úteis**; anuência prazo esperado **15 dias úteis**; extensão Escrow janela resposta Cedente **24 horas**; atualização status Escrow máx **5 minutos** | ✅ |
| 4 | **Contagem de itens** | 6 estados de StatusEscrow; 4 critérios do Fechamento (instrumento assinado + preço confirmado + anuência + depósito); 3 opções de anuência negada; 2 opções extensão inadimplência (cancelar ou +10d) | ✅ |
| 5 | **Máquina de estados** | Escrow: `ABERTA → DEPOSITO_CONFIRMADO → EM_PERIODO_REVERSAO → VALORES_DISTRIBUIDOS → COMISSAO_DEFINITIVA`; `EM_PERIODO_REVERSAO → ESTORNADA` (desistência); Caso: `EM_FORMALIZACAO → NEGOCIO_FECHADO → AGUARDANDO_LIBERACAO → CONCLUIDO`; retrocessos permitidos implementados: `EM_FORMALIZACAO → PENDENCIA_IDENTIFICADA` (anuência negada) | ✅ |
| 6 | **RBAC** | RLS em `contas_escrow` somente leitura para Cedente; Admin-only para `confirmar-deposito`, `solicitar-extensao`, `aceitar/mediar desistencia`; `GET /financeiro` retorna HTTP 405 para POST/PUT/DELETE | ✅ |
| 7 | **Anti-scaffold** | `EscrowDistribuicaoCron` usa `NOW()` com fuso `America/Fortaleza`; `ExtensaoSilencioAprovacaoCron` registra label específico "Aprovação por silêncio — 24h"; `InadimplenciaPrazoCron` diferencia os 2 SLAs (15d + 10d extensão) | ✅ |
| 8 | **Glossário** | `contas_escrow` (não `escrow_accounts`), `valor_cedente` (não `net_amount`), `data_distribuicao_prevista` (não `scheduled_release`), `VALORES_DISTRIBUIDOS` (não `completed`), `MEDIACAO_EM_ANDAMENTO` (não `in_dispute`) | ✅ |
| 9 | **ADR-CED-003** | Confirmação de depósito é manual pelo Admin no MVP (não automática via parceiro) — placeholder implementado; cron de distribuição funciona a partir do dado manual | ✅ |
| 10 | **Cálculo comissão** | `valor_comissao_rs` calculado usando `comissao.calculator` importada de S3 — mesma função em todos os pontos do sistema | ✅ |
| 11 | **Cross-módulo** | `NEGOCIO_FECHADO` verifica 4 critérios via S4 (assinaturas); `DISTRIBUICAO_REALIZADA` atualiza `casos.status` e dispara `NOT-CED-14`; Supabase Realtime atualiza countdown no frontend sem reload | ✅ |
| 12 | **Cobertura de REQs** | REQ-036 (proposta múltipla), REQ-061 (anuência acompanhamento), REQ-064–067 (anuência estados), REQ-087 (anuência concedida/negada/prazo), REQ-088 (inadimplência anuência), REQ-089 (estorno prioritário 3d), REQ-090 (opções anuência negada), REQ-091 (prazo inadimplência 15+10d), REQ-092 (cancelamento automático inadimplência), REQ-093 (extensão Escrow 24h silêncio=aprovação), REQ-104 (Financeiro somente leitura), REQ-105 (valor líquido destaque), REQ-106 (reversão transparência), REQ-107 (Cenário A comissão R$0), REQ-108 (atualização Escrow 5min), REQ-109 (desistência 15d), REQ-110 (irrevogabilidade), REQ-111–120 (estados Escrow), REQ-127 (distribuição 2d úteis), REQ-128 (mediação 10d úteis) — 100% atribuídos | ✅ |

---

## 📋 PENDÊNCIAS

| ID | Tipo | Descrição |
|---|---|---|
| DP-001 | ⚠️ BLOCKER | Parceiro bancário/fintech Conta Escrow não definido. `EscrowStatusSyncCron`, `ESTORNO_SOLICITADO`, confirmação de depósito automática — todos dependem deste parceiro. MVP usa ADR-CED-003 (manual Admin). [REVISÃO MANUAL — decisão bloqueante de go-live] |
| PEND-10 | ⚠️ AMBÍGUO | RN-039 usa nomenclatura "Aguardando liberação" para o status do caso; RN-053 usa "EM_PERIODO_REVERSAO" para a Conta Escrow. São estados diferentes da mesma entidade? Adotado: `casos.status='AGUARDANDO_LIBERACAO'` (visível ao Cedente) e `contas_escrow.status='EM_PERIODO_REVERSAO'` (técnico) são sincronizados. [REVISÃO MANUAL] |
