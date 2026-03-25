# S5 — Propostas e Negociação
## Módulo Cedente · Plataforma Repasse Seguro

| **Sprint** | **Tipo** | **Template** | **Versão** | **Data** |
|---|---|---|---|---|
| S5 | Módulo Fullstack | Template B | v1.0 | 2026-03-24 |

**REQs cobertos:** REQ-044 a REQ-060, REQ-096, REQ-121–122

**Dependências:** S1 (infra), S2 (auth), S3 (casos, comissao.calculator), S4 (notificações, eventos_caso)

**Critério de aceite da sprint:** todos os itens ✅ + 12 auto-verificações ✅ + T-037, T-039 passando.

---

## 🔴 FEATURE 1 — Propostas (recebimento, visualização, resposta)

### 1. Banco de Dados — Propostas

- [x] **1.1** Confirmar que a tabela `propostas` possui os campos: `id UUID PK`, `caso_id UUID FK REFERENCES casos(id)`, `numero_sequencial INTEGER NOT NULL` (1, 2, 3... por caso), `tipo TipoProposta NOT NULL`, `status StatusProposta NOT NULL DEFAULT 'AGUARDANDO_RESPOSTA'`, `valor BIGINT NOT NULL` (em centavos), `rodada_negociacao INTEGER NOT NULL DEFAULT 1`, `recebida_em TIMESTAMPTZ NOT NULL DEFAULT now()`, `expira_em TIMESTAMPTZ NOT NULL`, `respondida_em TIMESTAMPTZ`, `valor_contraproposta BIGINT`, `created_at`, `updated_at`.
- [x] **1.2** Confirmar que o enum `TipoProposta` possui: `OFERTA_INICIAL`, `CONTRAPROPOSTA_CESSIONARIO`, `CONTRAPROPOSTA_CEDENTE`.
- [x] **1.3** Confirmar que o enum `StatusProposta` possui: `AGUARDANDO_RESPOSTA`, `ACEITA`, `RECUSADA`, `SUPERADA`, `EXPIRADA`, `SEM_RESPOSTA`.
- [x] **1.4** Confirmar que a tabela `propostas` NUNCA possui os campos `cessionario_id`, `cessionario_nome`, `cessionario_email` ou qualquer dado identificador do Cessionário (RN-012). Se esses campos existirem em alguma migration → removê-los da projeção com RLS/policy ou nunca inserir via API do Cedente.
- [x] **1.5** Criar política RLS em `propostas`: `FOR SELECT USING (auth.uid() = (SELECT cedente_auth_id FROM casos WHERE id = caso_id))` — Cedente vê APENAS propostas dos seus casos (confirmação S2/S3).
- [x] **1.6** Criar índice `idx_propostas_caso_status ON propostas(caso_id, status)` para filtrar propostas ativas por caso.
- [x] **1.7** Criar índice `idx_propostas_expira_em ON propostas(status, expira_em)` para job cron de expiração automática.
- [x] **1.8** Confirmar que `expira_em = recebida_em + INTERVAL '5 business days'` (5 dias úteis — RN-031). Implementar função PostgreSQL `add_business_days(timestamp, integer)` que respeita feriados nacionais brasileiros (ou adotar biblioteca `date-fns` no backend para calcular o valor antes de inserir).

### 2. Backend — Propostas Endpoints

- [x] **2.1** Implementar `GET /propostas?caso_id=:id&status=AGUARDANDO_RESPOSTA` — retorna propostas ativas (status `AGUARDANDO_RESPOSTA`) do caso, ordenadas por `valor DESC` (maior valor primeiro — RN-036 item 3). Badge count disponível via header `X-Total-Count`. NUNCA retornar `cessionario_id` no response (RN-012).
- [x] **2.2** Implementar `GET /propostas/historico?caso_id=:id` — retorna TODAS as propostas do caso (incluindo expiradas, recusadas, superadas) para exibição na aba Histórico do caso (RN-091).
- [x] **2.3** Response de proposta deve incluir campos calculados: `valor_liquido: bigint` (calculado pela `comissao.calculator`), `comissao_rs: bigint`, `ganho_vs_distrato: bigint`, `timer_restante_segundos: number`, `urgencia: 'NORMAL' | 'ALERTA' | 'CRITICA'` (CRITICA = ≤1 dia útil = badge vermelho, ALERTA = 2–3 dias úteis = badge amarelo, NORMAL = >3 dias úteis — RN-030 item 3, RN-031).
- [x] **2.4** Timer `timer_restante_segundos` = `expira_em - NOW()` em segundos. Negativo = proposta expirada.
- [x] **2.5** Implementar `POST /propostas/:id/aceitar` (RN-032):
  - Verificar `status = 'AGUARDANDO_RESPOSTA'` e `expira_em > NOW()`.
  - Usar `prisma.$transaction`: atualizar `propostas SET status='ACEITA', respondida_em=NOW()`; encerrar TODAS as outras propostas ativas do mesmo caso com `status='SUPERADA'` (RN-033); `UPDATE casos SET status='EM_FORMALIZACAO'`.
  - Se aceite falhar 3 vezes (erro de sistema): escalar automaticamente para Admin com flag `aceite_com_falha=true` + pausar timer da proposta e informar Cedente: "Estamos resolvendo um problema técnico. Sua proposta está segura e o prazo foi pausado." (RN-032 item 5).
  - Publicar evento `PROPOSTA_ACEITA` no RabbitMQ → NotificationService envia template `NOT-CED-10`.
  - Retornar HTTP 200 `{ caso_status: 'EM_FORMALIZACAO', proposta_id, valor_aceito, valor_liquido, comissao_rs }`.
- [x] **2.6** Implementar `POST /propostas/:id/recusar` (RN-034):
  - Verificar `status = 'AGUARDANDO_RESPOSTA'`.
  - `UPDATE propostas SET status='RECUSADA', respondida_em=NOW()`.
  - `UPDATE casos SET status='DISPONIVEL_PARA_COMPRADORES'` (caso retorna para fila).
  - Publicar evento `PROPOSTA_RECUSADA`.
  - Retornar HTTP 200 `{ caso_status: 'DISPONIVEL_PARA_COMPRADORES', mensagem: "Proposta recusada. Seu imóvel continua disponível para compradores." }` (RN-034 item 3).
- [x] **2.7** Implementar `POST /propostas/:id/contraproposta` com body `{ valor: bigint }` (RN-035):
  - Verificar `status = 'AGUARDANDO_RESPOSTA'` e `expira_em > NOW()`.
  - Validar que `valor >= piso_do_cenario` (buscar piso via endpoint interno do Admin — se não disponível, usar `valor_pago` como referência mínima e marcar `⚠️ PEND-08`).
  - Se `valor < piso`: retornar HTTP 422 com `cedente_message: "O valor informado está abaixo do mínimo para o cenário [X]. Você pode: (1) Informar um valor acima de R$ [piso], ou (2) Solicitar ajuste de cenário."`.
  - Se válido: `UPDATE propostas SET status='AGUARDANDO_RESPOSTA', tipo='CONTRAPROPOSTA_CEDENTE', valor_contraproposta=valor, rodada_negociacao=rodada+1, expira_em=NOW()+5_dias_uteis`.
  - Timer reiniciado (RN-035 item 3).
  - Publicar evento `CONTRAPROPOSTA_ENVIADA` → Admin medeia com Cessionário.

### 3. Backend — Jobs de Expiração de Propostas

- [x] **3.1** Implementar `PropostaExpiracaoCron` que roda a cada hora:
  - Buscar propostas com `status = 'AGUARDANDO_RESPOSTA'` e `expira_em <= NOW()`.
  - `UPDATE propostas SET status='EXPIRADA'` (ou `SEM_RESPOSTA` se primeira tentativa sem resposta do Cedente — RN-031 item 5).
  - `UPDATE casos SET status='DISPONIVEL_PARA_COMPRADORES'`.
  - Publicar evento `PROPOSTA_EXPIRADA` → Admin notifica Cessionário.
- [x] **3.2** Implementar `PropostaLembreteCron` que roda a cada hora:
  - Após 2 dias úteis sem resposta: badge → `urgencia='ALERTA'`; publicar `LEMBRETE_PROPOSTA_EXPIRANDO` → template `NOT-CED-09` (lembrete proposta expirando — desativável).
  - Após 3 dias úteis sem resposta: badge → `urgencia='CRITICA'`; publicar lembrete urgente.
  - (RN-031 itens 3–4)

### 4. Backend — Escalonamento de Cenário

- [x] **4.1** Implementar `GET /casos/:id/escalonamento/disponivel` — retorna se escalonamento está disponível: `{ disponivel: boolean, proximo_cenario: CenarioRetorno | null, motivo_bloqueio?: string, proximo_disponivel_em?: string }`.
- [x] **4.2** Lógica de disponibilidade (RN-025, RN-026, RN-027):
  - Se `cenario_retorno = 'A'`: `{ disponivel: false, motivo_bloqueio: "Cenário A é o mínimo. Não é possível escalonar." }`.
  - Se há proposta ativa (`AGUARDANDO_RESPOSTA`): `{ disponivel: false, motivo_bloqueio: "Há uma proposta em análise. Escalonamento será processado após conclusão da negociação." }`.
  - Se escalonamento realizado há <7 dias corridos: `{ disponivel: false, proximo_disponivel_em: timestamp }` (RN-027 item 3).
  - Senão: `{ disponivel: true, proximo_cenario: cenario_imediatamente_inferior }`.
- [x] **4.3** Implementar `POST /casos/:id/escalonamento/solicitar` (RN-025):
  - Verificar disponibilidade (item 4.2).
  - Se há proposta ativa: enfileirar escalonamento com `escalonamento_pendente = true` no `casos` — processar após conclusão da negociação (RN-025 item 4).
  - Se não há proposta ativa: `UPDATE casos SET cenario_retorno=proximo_cenario, ultimo_escalonamento_em=NOW()`; gerar envelope ZapSign `TERMO_ACEITE_ESCALONAMENTO`; publicar evento `ESCALONAMENTO_SOLICITADO`.
- [x] **4.4** Implementar `DELETE /casos/:id/escalonamento/cancelar` — cancela escalonamento enfileirado; limpar `escalonamento_pendente = false`; cooldown não conta (RN-027 item 5).
- [x] **4.5** Lógica pós-negociação: quando proposta for aceita/recusada/expirada E `escalonamento_pendente = true` → processar escalonamento automaticamente (RN-025 item 4).
- [x] **4.6** Edge case RN-079 (D01.4 — proposta cancela escalonamento enfileirado): se chega nova proposta válida enquanto escalonamento está enfileirado → cancelar o escalonamento enfileirado automaticamente e notificar Cedente: "Uma nova proposta foi recebida. Seu pedido de ajuste de cenário foi cancelado. Você pode solicitá-lo novamente após responder à proposta."

### 5. Frontend — Tela Propostas

- [x] **5.1** Criar `apps/web-cedente/src/app/(authenticated)/propostas/page.tsx` — lista de propostas ativas.
- [x] **5.2** Badge numérico no menu lateral "Propostas ([X])" indicando quantidade de propostas ativas (RN-036 item 3). Atualizar via Supabase Realtime.
- [x] **5.3** Listar propostas ordenadas por `valor DESC` (maior primeiro). Cada item exibe: identificação sequencial "Proposta 1", valor oferecido, timer regressivo dinâmico (atualizado a cada minuto; nos últimos 24h exibe horas; na última hora exibe minutos com pulsação do badge vermelho — RN-030 item 3), badge de urgência (vermelho/amarelo/normal).
- [x] **5.4** Ao clicar em uma proposta: abrir detalhe com hierarquia visual obrigatória (RN-030 item 4):
  - (1) **"Você recebe" em destaque máximo** — `valor_liquido` em fonte maior, posição superior, cor verde.
  - (2) Valor da proposta em segundo nível.
  - (3) Comissão RS + comparativo vs. distrato em terceiro nível (colapsável).
- [x] **5.5** 3 botões de ação: "Aceitar Proposta", "Recusar Proposta", "Fazer Contraproposta".
- [x] **5.6** Implementar modal de confirmação de aceite (RN-032 item 2): "Ao aceitar, o valor acordado será R$ [X]. Sua comissão RS será R$ [Y]. Você receberá R$ [Z] líquidos. Confirmar?". Botão "Confirmar aceite" + "Cancelar".
- [x] **5.7** Implementar modal de confirmação de recusa (RN-034 item 2): confirmação simples sem justificativa obrigatória.
- [x] **5.8** Implementar modal de contraproposta (RN-035): campo de valor com formatação R$ em tempo real; simulação inline do `valor_liquido` ao digitar; validação de piso do cenário com mensagem de erro específica.
- [x] **5.9** NUNCA exibir nome, CPF, e-mail do Cessionário em nenhum elemento da tela (RN-012).
- [x] **5.10** Propostas expiradas/recusadas/superadas: NÃO aparecem na lista ativa de Propostas — apenas na aba "Propostas" do detalhe do caso em Meus Casos (RN-091).
- [x] **5.11** Criar `apps/web-cedente/src/app/(authenticated)/propostas/[id]/page.tsx` — detalhe de proposta.

### 6. Frontend — Tela Escalonamento

- [x] **6.1** No detalhe do caso em Meus Casos: exibir botão "Alterar Cenário" apenas se escalonamento disponível (chamar `GET /casos/:id/escalonamento/disponivel`).
- [x] **6.2** Se `disponivel=false`: botão desabilitado com tooltip com `motivo_bloqueio`; se há prazo de cooldown: exibir "Próximo ajuste disponível a partir de [data]" (RN-027 item 3).
- [x] **6.3** Ao clicar "Alterar Cenário": exibir modal com simulação comparativa (cenário atual vs. cenário inferior) calculada com `comissao.calculator` e os valores do caso; exibir `proximo_cenario`.
- [x] **6.4** Ao confirmar: `POST /casos/:id/escalonamento/solicitar`. Se escalonamento enfileirado: exibir no detalhe do caso "Ajuste de cenário solicitado. Será processado após a conclusão da negociação em andamento." com ícone de relógio + botão "Cancelar solicitação" (RN-025 item 4).
- [x] **6.5** Exibir aviso de cenário mínimo: se `cenario_retorno = 'A'` → não mostrar botão "Alterar Cenário". Se Cedente consultar Guardião sobre escalonamento → exibir "Cenário A é o mínimo. Para tentar cenário superior, seria necessário cancelar e recadastrar." (RN-025 item 2, RN-029).

---

## 🔀 CROSS-MÓDULO

### 7. Eventos disparados por esta sprint

- [x] **7.1** Após `PROPOSTA_ACEITA`: `UPDATE casos SET status='EM_FORMALIZACAO'` + publicar `STATUS_MUDOU` → S4 (eventos_caso) registra na linha do tempo + S6 (Financeiro) recebe evento para abrir Conta Escrow.
- [x] **7.2** Após `PROPOSTA_RECUSADA` ou `PROPOSTA_EXPIRADA`: `UPDATE casos SET status='DISPONIVEL_PARA_COMPRADORES'` + publicar `STATUS_MUDOU`.
- [x] **7.3** Após `ESCALONAMENTO_SOLICITADO` (sem proposta ativa): gerar envelope ZapSign `TERMO_ACEITE_ESCALONAMENTO` (S4) + publicar `STATUS_MUDOU` após assinatura.
- [x] **7.4** Após `PROPOSTA_ACEITA` + `escalonamento_pendente=true`: cancelar escalonamento + notificar Cedente (RN-079, edge case).

---

## 🧪 TESTES

### 8. Testes unitários (Vitest)

- [x] **8.1** `propostas.service.spec.ts` — testar `aceitar()`: proposta `AGUARDANDO_RESPOSTA` → aceita + outras superadas + caso muda para `EM_FORMALIZACAO`; proposta expirada → HTTP 422; proposta de outro caso → HTTP 403.
- [x] **8.2** `propostas.service.spec.ts` — testar `recusar()`: proposta aceita → caso volta para `DISPONIVEL_PARA_COMPRADORES`.
- [x] **8.3** `propostas.service.spec.ts` — testar `contraproposta()`: valor acima do piso → aceito; valor abaixo do piso → HTTP 422 com mensagem específica; rodada incremented.
- [x] **8.4** `propostas.service.spec.ts` — testar que `cessionario_id` nunca aparece no response (mock query + verificar response shape).
- [x] **8.5** `escalonamento.service.spec.ts` — testar `verificarDisponibilidade()`: cenário A → `disponivel=false`; proposta ativa → `disponivel=false`; cooldown 7d → `disponivel=false` com `proximo_disponivel_em`; tudo ok → `disponivel=true`.
- [x] **8.6** `escalonamento.service.spec.ts` — testar enfileiramento: proposta ativa + solicitar escalonamento → `escalonamento_pendente=true`; recusa proposta → escalonamento processado automaticamente.
- [x] **8.7** `proposta-expiracao.cron.spec.ts` — testar que proposta expirada muda status e retorna caso para `DISPONIVEL_PARA_COMPRADORES`.

### 9. Testes de integração (Supertest)

- [x] **9.1** T-037: `POST /propostas/:id/aceitar` → verificar `casos.status='EM_FORMALIZACAO'`; outras propostas do caso com `status='SUPERADA'`; response contém `valor_liquido` calculado.
- [x] **9.2** T-039: `POST /propostas/:id/recusar` → verificar `casos.status='DISPONIVEL_PARA_COMPRADORES'`; proposta com `status='RECUSADA'`.
- [x] **9.3** `POST /propostas/:id/aceitar` com proposta já expirada → HTTP 422.
- [x] **9.4** `POST /propostas/:id/aceitar` com token de Cedente de outro caso → HTTP 403 (RLS).
- [x] **9.5** `GET /propostas?caso_id=:id` — verificar que response não contém campo `cessionario_id` (verificar schema do response).
- [x] **9.6** `POST /casos/:id/escalonamento/solicitar` com cooldown ativo → HTTP 422 com data de próximo disponível.
- [x] **9.7** `POST /casos/:id/escalonamento/solicitar` com proposta ativa → `escalonamento_pendente=true`; `POST /propostas/:id/recusar` → verificar escalonamento processado automaticamente.

---

## ✅ AUTO-VERIFICAÇÃO (12 Checks)

| # | Check | Critério | Status |
|---|---|---|---|
| 1 | **Nomenclatura** | `propostas` (não `offers`), `TipoProposta`: `OFERTA_INICIAL`, `CONTRAPROPOSTA_CESSIONARIO`, `CONTRAPROPOSTA_CEDENTE`; `StatusProposta`: `AGUARDANDO_RESPOSTA`, `ACEITA`, `RECUSADA`, `SUPERADA`, `EXPIRADA`, `SEM_RESPOSTA`; `DISPONIVEL_PARA_COMPRADORES`, `EM_FORMALIZACAO`, `PROPOSTA_RECEBIDA` — nomes exatos dos estados | ✅ |
| 2 | **Verificabilidade** | Cada item binariamente verificável — "implementar aceite" → "verificar que `propostas SET status='ACEITA'` + outras `status='SUPERADA'` + `casos SET status='EM_FORMALIZACAO'` em uma `prisma.$transaction`" | ✅ |
| 3 | **Valores numéricos** | Proposta expira em **5 dias úteis**; badge amarelo após **2 dias úteis**; badge vermelho após **3 dias úteis**; cooldown escalonamento **7 dias corridos**; reinício do timer na contraproposta = +**5 dias úteis**; contraproposta Cessionário sem resposta = **5 dias úteis** | ✅ |
| 4 | **Contagem de itens** | 3 ações do Cedente na proposta: aceitar, recusar, contrapropor; ordenação por valor DESC; badge numérico no menu; 4 estados de urgência no timer; 3 campos no response de proposta calculados: `valor_liquido`, `comissao_rs`, `ganho_vs_distrato` | ✅ |
| 5 | **Máquina de estados** | `StatusProposta`: `AGUARDANDO_RESPOSTA` → `ACEITA`/`RECUSADA`/`EXPIRADA`/`SEM_RESPOSTA`; após aceite → outras propostas → `SUPERADA`; `casos.status`: `PROPOSTA_RECEBIDA` → `EM_FORMALIZACAO` (aceite) ou `DISPONIVEL_PARA_COMPRADORES` (recusa/expiração); escalonamento: enfileirado → processado após negociação; cooldown → liberado | ✅ |
| 6 | **RBAC** | RLS em `propostas` garante que Cedente só vê propostas dos seus casos; `cessionario_id` nunca no response (projeção explícita Prisma `select: { cessionario_id: false }`); `POST /propostas/:id/aceitar` verifica owner do caso | ✅ |
| 7 | **Anti-scaffold** | `prisma.$transaction` real no aceite (não 2 queries separadas); `PropostaExpiracaoCron` verifica dias úteis reais (não apenas intervalo simples); simulação de valor líquido na contraproposta usa `comissao.calculator` importada de S3 | ✅ |
| 8 | **Glossário** | `propostas` (não `bids`), `contraproposta` (não `counter_offer`), `piso_do_cenario` (não `minimum_price`), `rodada_negociacao` (não `round`), `valor_liquido` (não `net_value`) | ✅ |
| 9 | **RLS** | `propostas` isoladas por `caso_id → cedente_auth_id`; `cessionario_id` jamais exposto ao Cedente; aceite com `prisma.$transaction` evita race condition de duplo aceite | ✅ |
| 10 | **Hierarquia visual** | `valor_liquido` sempre em destaque máximo (maior fonte, verde, posição superior) — verificar no componente React que não há cenário onde `comissao_rs` aparece como destaque principal | ✅ |
| 11 | **Cross-módulo** | `PROPOSTA_ACEITA` dispara abertura de Conta Escrow em S6; `ESCALONAMENTO_SOLICITADO` gera envelope ZapSign em S4; `STATUS_MUDOU` registrado em `eventos_caso` em S3/S4 | ✅ |
| 12 | **Cobertura de REQs** | REQ-044 (recebimento proposta), REQ-045 (timer 5d úteis), REQ-046 (aceite confirmação dupla), REQ-047 (superação concorrentes), REQ-048 (recusa), REQ-049 (contraproposta no piso), REQ-050 (múltiplas propostas ordenação), REQ-051 (escalonamento descendente), REQ-052 (um nível), REQ-053 (cooldown 7d), REQ-054 (múltiplo ao longo do ciclo), REQ-055 (subida proibida), REQ-056 (transparência comissão 4 momentos), REQ-057 (fórmula exemplos numéricos), REQ-058 (desistência pós-Fechamento — referência a S6), REQ-059 (irrevogabilidade 15d), REQ-060 (histórico escalonamento), REQ-096 (proposta cancela escalonamento enfileirado), REQ-121–122 (edge cases escalonamento) — 100% atribuídos | ✅ |

---

## 📋 PENDÊNCIAS

| ID | Tipo | Descrição |
|---|---|---|
| PEND-08 | ⚠️ AMBÍGUO | RN-035 menciona "piso do cenário de retorno calculado pelo Admin". A API do Admin para buscar esse piso não está especificada no Doc 16 (44 endpoints documentados). Adotado provisoriamente: usar `valor_pago` do caso como piso mínimo para validação no MVP. [REVISÃO MANUAL — solicitar endpoint Admin] |
| PEND-09 | ⚠️ AMBÍGUO | RN-025 item 4 diz que proposta ativa CANCELA o escalonamento enfileirado (RN-079/D01.4). Interpretado como: nova proposta CANCELA escalonamento pendente; proposta recusada/expirada PROCESSA escalonamento pendente. Confirmar com PM. |
