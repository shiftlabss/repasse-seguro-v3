# S3 — Cadastro e Dashboard

## Módulo Cedente · Plataforma Repasse Seguro

| **Sprint** | **Tipo**         | **Template** | **Versão** | **Data**   |
| ---------- | ---------------- | ------------ | ---------- | ---------- |
| S3         | Módulo Fullstack | Template B   | v1.0       | 2026-03-24 |

**REQs cobertos:** REQ-022 a REQ-043, REQ-062–063, REQ-090–091

**Dependências:** S1 (infra), S2 (auth completa — `JwtAuthGuard`, `users`, `cedentes`, RLS)

**Critério de aceite da sprint:** todos os itens ✅ + 12 auto-verificações ✅ + wizard E2E passando (T-001 a T-005).

---

## 🔴 FEATURE 1 — Dashboard

### 1. Banco de Dados — Dashboard

- [x] **1.1** Confirmar que a tabela `casos` possui os campos necessários para o Dashboard: `id`, `cedente_id`, `status StatusCaso NOT NULL`, `cenario_retorno CenarioRetorno`, `nome_imovel VARCHAR(255)`, `valor_pago BIGINT` (em centavos), `created_at`, `updated_at`.
- [x] **1.2** Confirmar que a tabela `notificacoes` possui campo `lida BOOLEAN NOT NULL DEFAULT false` e `destinatario_auth_id UUID` para query de notificações não lidas (badge).
- [x] **1.3** Confirmar que a tabela `eventos_caso` possui campos `caso_id`, `tipo TipoEvento`, `descricao_publica TEXT`, `created_at` para o feed de eventos do Dashboard.
- [x] **1.4** Criar índice `idx_casos_cedente_status ON casos(cedente_id, status)` para query eficiente de casos ativos por Cedente.
- [x] **1.5** Criar índice `idx_eventos_caso_criado ON eventos_caso(caso_id, created_at DESC)` para feed dos últimos 10 eventos.

### 2. Backend — Dashboard Endpoints

- [x] **2.1** Implementar `GET /dashboard/resumo` — retorna: `{ casos_ativos: number, pendencias: number, propostas_recebidas: number, valor_liquido_estimado: number | null }`. Requer `JwtAuthGuard`. Dados filtrados por `cedente_id` via RLS.
- [x] **2.2** No cálculo de `valor_liquido_estimado`: se houver proposta ativa com status `AGUARDANDO_RESPOSTA`, calcular `valor_proposta - comissao_estimada`; se não houver, retornar `null`. Fórmula de comissão: `20% × (valor_proposta − (valor_pago × 50%))` para cenários B/C/D; `0` para cenário A (RN-038).
- [x] **2.3** Implementar `GET /dashboard/proximos-passos` — retorna lista de ações pendentes por urgência: `{ tipo: 'ASSINAR' | 'ENVIAR_DOCUMENTO' | 'RESPONDER_PROPOSTA', caso_id, nome_imovel, prazo_em_dias_uteis: number, urgencia: 'CRITICA' | 'ALTA' | 'NORMAL' }`. Urgência CRITICA = prazo ≤ 1 dia útil (vermelho); ALTA = 2–3 dias úteis (amarelo); NORMAL = >3 dias úteis (RN-016).
- [x] **2.4** Implementar `GET /dashboard/feed-eventos?limit=10` — retorna últimos 10 eventos de todos os casos do Cedente ordenados por `created_at DESC`. Apenas campos públicos: `tipo`, `descricao_publica`, `created_at`, `nome_imovel`.
- [x] **2.5** Todos os endpoints do Dashboard aplicam cache Redis `rs:cedente:dashboard:{user_id}` com TTL de **60 segundos** (RN-014 item 4). Invalidar cache ao receber evento de mudança de estado via RabbitMQ.
- [x] **2.6** Dashboard nunca permite ações operacionais (POST/PATCH/DELETE) — apenas leitura (RN-013 item 4). Retornar HTTP 405 se alguém tentar.

### 3. Frontend — Dashboard

- [x] **3.1** Criar `apps/web-cedente/src/app/(authenticated)/dashboard/page.tsx` — Server Component que faz `fetch` das 3 queries do Dashboard.
- [x] **3.2** Implementar 4 cards de resumo conforme RN-013 item 3:
  - Card "Casos Ativos": número de casos com status não-terminal.
  - Card "Pendências": número de ações urgentes aguardando o Cedente.
  - Card "Propostas Recebidas": número de propostas com status `AGUARDANDO_RESPOSTA`.
  - Card "Valor Líquido Estimado": valor em reais formatado; se `null` exibir ícone de espera com texto "Aguardando primeira proposta" em opacidade reduzida (RN-015 item 4). Sempre com nota: "Valor estimado com base no cenário atual. Sujeito a alteração conforme negociação." (RN-015 item 3).
- [x] **3.3** Implementar painel "Próximos Passos": itens com prazo ≤ 1 dia útil em vermelho com ícone específico (documento/caneta/moeda); prazo 2–3 dias úteis em amarelo; prazo >3 dias úteis sem destaque (RN-016).
- [x] **3.4** Implementar feed dos últimos 10 eventos com timestamp relativo ("há X minutos/horas").
- [x] **3.5** Implementar indicador de última atualização no rodapé: "Atualizado há [X] segundos". Polling a cada **60 segundos** via `setInterval` com `revalidateTag` ou `router.refresh()`. Em caso de falha de rede: exibir "Dados podem estar desatualizados. Verificando conexão..." (RN-014 item 4).
- [x] **3.6** Implementar estado vazio (primeiro acesso, sem casos): exibir (a) mensagem de boas-vindas personalizada com `cedente.nome`, (b) checklist visual de 3 passos ("Cadastre seu imóvel", "Envie os documentos", "Receba propostas"), (c) botão "Cadastrar meu primeiro imóvel" em destaque, (d) link "Fale com nosso assistente" para o chat do Guardião do Retorno (RN-013 item 5).
- [x] **3.7** Dashboard é somente leitura — NENHUM botão de ação operacional (cancelar, aceitar proposta, upload) está presente (RN-013 item 4).
- [x] **3.8** Configurar Supabase Realtime para receber eventos de mudança em `casos` e `notificacoes` do Cedente logado e atualizar os cards sem reload completo da página.

---

## 🔴 FEATURE 2 — Meus Casos

### 4. Backend — Meus Casos Endpoints

- [x] **4.1** Implementar `GET /casos` com paginação offset: `page` (default 1), `per_page` (default 20, máx 100); filtros opcionais: `status`, `cenario_retorno`; campo de busca `q` (busca em `nome_imovel` e `endereco`). Retorna `{ data: Caso[], total: number, page: number, per_page: number }`. Ordenação: `created_at DESC` (RN-017 item 2).
- [x] **4.2** Implementar `GET /casos/:id` — retorna detalhe completo do caso incluindo campos de todas as abas (Resumo, status, valores, cenário). Requer que `cedente_id` corresponda ao Cedente logado via RLS; retorna HTTP 403 se não corresponder (RN-011).
- [x] **4.3** No response de `GET /casos/:id`: incluir `historico` como array de `eventos_caso` do caso ordenados por `created_at ASC` (linha do tempo imutável — RN-020 item 4).
- [x] **4.4** Retornar `nome_amigavel_status` mapeado no backend para cada caso (nunca retornar termos internos como "Captado", "Qualificado", "Oferta Ativa" — apenas os 13 nomes amigáveis da seção 6 de RN-001.1) (RN-018).
- [x] **4.5** Implementar `PATCH /casos/:id/dados-imovel` — bloquear edição se `status` não for `CADASTRO_REALIZADO` ou `EM_ANALISE`: retornar HTTP 403 com `cedente_message: "Os dados do imóvel não podem ser alterados após a aprovação do dossiê."` (RN-019).
- [x] **4.6** Dados de `propostas` retornados no GET do caso NUNCA incluem `cessionario_id`, `cessionario_nome`, `cessionario_email` ou qualquer dado identificador do Cessionário (RN-012). Implementar projeção explícita na query Prisma: `select: { cessionario_id: false, ... }`.

### 5. Frontend — Meus Casos

- [x] **5.1** Criar `apps/web-cedente/src/app/(authenticated)/casos/page.tsx` — lista paginada com filtros.
- [x] **5.2** Implementar filtros: dropdown de status, dropdown de cenário (A/B/C/D), campo de busca por nome do imóvel/endereço.
- [x] **5.3** Implementar paginação numérica no rodapé: setas "Anterior/Próximo", total "Exibindo [X]–[Y] de [Z] casos" (RN-017 item 4).
- [x] **5.4** Cada card de caso exibe: nome do imóvel, status com nome amigável, badge de cenário (A/B/C/D), data de criação, indicador de urgência se houver pendência (RN-018).
- [x] **5.5** Criar `apps/web-cedente/src/app/(authenticated)/casos/[id]/page.tsx` — detalhe do caso.
- [x] **5.6** Implementar abas navegáveis com acessibilidade: `role="tablist"`, `role="tab"`, `role="tabpanel"`, `aria-selected`, navegação por setas esquerda/direita (RN-020 item 2):
  - Aba "Resumo": dados do imóvel, cenário, valor pago, Valor Distrato Referência, simulação dos 4 cenários, status atual com explicação.
  - Aba "Documentos": redireciona para tela de documentos filtrada pelo `caso_id`.
  - Aba "Propostas": redireciona para tela de propostas filtrada pelo `caso_id`.
  - Aba "Financeiro": redireciona para tela financeira filtrada pelo `caso_id`.
  - Aba "Histórico": linha do tempo imutável com todos os eventos do caso.
- [x] **5.7** Aba "Resumo": exibir a simulação dos 4 cenários calculada a partir de `valor_pago` do caso (usando a fórmula de RN-038). Para cada cenário: valor recuperado estimado, comissão RS, valor líquido, comparativo "+R$X vs. distrato".
- [x] **5.8** Aba "Histórico": eventos em ordem cronológica (mais antigo no topo), cada evento com data/hora, tipo e `descricao_publica`. Nunca exibir termos internos de status (RN-018).

---

## 🔴 FEATURE 3 — Wizard de Cadastro de Imóvel

### 6. Banco de Dados — Wizard

- [x] **6.1** Confirmar que a tabela `casos` possui campo `rascunho BOOLEAN NOT NULL DEFAULT false` para diferenciar rascunhos de casos ativos.
- [x] **6.2** Confirmar que a tabela `casos` possui campo `wizard_step_atual INTEGER NOT NULL DEFAULT 1` para retomar rascunho na etapa correta.
- [x] **6.3** Criar índice `idx_casos_cedente_rascunho ON casos(cedente_id, rascunho)` para verificação de rascunho ativo por Cedente.
- [x] **6.4** Confirmar que `casos` possui campo `simulador_visualizado_em TIMESTAMPTZ` para registrar quando o Cedente visualizou o simulador na Etapa 3 (RN-021 item 3).
- [x] **6.5** Criar índice `idx_casos_rascunho_expira ON casos(rascunho, updated_at)` para job cron de descarte de rascunhos expirados.

### 7. Backend — Wizard Endpoints

- [x] **7.1** Implementar `POST /casos/rascunho` — cria rascunho com `rascunho=true`, `wizard_step_atual=1`; verificar que não há outro rascunho ativo para o `cedente_id` (apenas 1 rascunho por vez — RN-023 item 3); se já há rascunho, retornar HTTP 409 com `{ rascunho_id, nome_imovel, ultimo_step, updated_at }` para o frontend oferecer "Continuar" ou "Descartar".
- [x] **7.2** Implementar `PATCH /casos/rascunho/:id/etapa/:step` — salva dados da etapa atual e atualiza `wizard_step_atual`. Validações específicas por etapa (ver itens 7.3–7.7).
- [x] **7.3** Validações da Etapa 1 (Dados do Imóvel): `endereco VARCHAR NOT NULL`, `cidade VARCHAR NOT NULL`, `estado CHAR(2) NOT NULL`, `construtora VARCHAR NOT NULL`, `numero_contrato VARCHAR NOT NULL`. Verificar unicidade de imóvel ativo (RN-090): se já existe `caso` com mesmo `numero_contrato` e `status NOT IN ('CONCLUIDO', 'CANCELADO')`, retornar HTTP 409 com `cedente_message: "Já existe um caso ativo para este imóvel. Acompanhe o andamento em Meus Casos."`.
- [x] **7.4** Validações da Etapa 2 (Dados Financeiros): `valor_pago BIGINT NOT NULL > 0` (em centavos), `valor_contrato BIGINT NOT NULL > 0`, `data_inicio_contrato DATE NOT NULL`. Calcular e retornar `valor_distrato_referencia = valor_pago * 0.5` no response.
- [x] **7.5** Lógica da Etapa 3 (Simulador): ao salvar step 3, persistir `simulador_visualizado_em = NOW()`. Retornar os 4 cenários calculados: para cada `cenario IN (A, B, C, D)` calcular `{ valor_recuperado, comissao_rs, valor_liquido, ganho_vs_distrato }` usando a fórmula de RN-038 (valores em centavos). Calcular cálculo síncrono — se demorar >5s retornar resposta parcial com flag `calculando: true` para o frontend exibir skeleton (RN-021 item 2).
- [x] **7.6** Validações da Etapa 4 (Escolha do Cenário): `cenario_retorno CenarioRetorno NOT NULL`; o campo é obrigatório, sem pré-seleção; verificar que `simulador_visualizado_em` existe e que `NOW() - simulador_visualizado_em >= INTERVAL '10 seconds'` — se não, retornar HTTP 400 com `cedente_message: "Analise os cenários antes de confirmar sua escolha."` (RN-021 item 3).
- [x] **7.7** Etapa 5 (Upload de Documentos — opcional): salvar os documentos enviados no dossiê do caso; se o Cedente pular, retornar `{ pulado: true }` e avançar (RN-088). O caso NÃO avança para `EM_ANALISE` automaticamente — aguarda dossiê completo + Termo de Cadastro assinado.
- [x] **7.8** Implementar `POST /casos/rascunho/:id/confirmar` — converte rascunho em caso real: `UPDATE casos SET rascunho=false, status='CADASTRO_REALIZADO', wizard_step_atual=NULL`; gerar `dossie` associado ao caso; disparar geração do Termo de Cadastro via ZapSign (RN-024). Retornar `{ caso_id, envelope_id }`.
- [x] **7.9** Implementar `DELETE /casos/rascunho/:id` — descarta rascunho; Cedente deve ser o owner (`cedente_id` corresponde ao logado via RLS).
- [x] **7.10** Implementar job cron `RascunhoExpiracaoCron` que roda diariamente às 03h BRT:
  - Enviar lembrete e-mail (template `NOT-CED-05` — rascunho expirando) nos dias 7, 15 e 25 a partir de `created_at` do rascunho (RN-023 item 4).
  - Após 30 dias corridos desde `created_at`: marcar `rascunho=false, status='CANCELADO'` e enviar notificação final (RN-023 item 5).

### 8. Lógica de Comissão — calculadora central

- [x] **8.1** Criar `apps/api/src/modules/casos/comissao.calculator.ts` com função `calcularComissao(valorPago: bigint, valorRecuperado: bigint, cenario: CenarioRetorno): { comissao: bigint, valorLiquido: bigint, ganhoVsDistrato: bigint }`.
- [x] **8.2** Implementar: `valorDistrato = valorPago * 50n / 100n`; se `cenario === 'A'` retornar `comissao=0n`; senão `comissao = (valorRecuperado - valorDistrato) * 20n / 100n`; `valorLiquido = valorRecuperado - comissao`; `ganhoVsDistrato = valorLiquido - valorDistrato`.
- [x] **8.3** Todos os valores em centavos (bigint) — NUNCA usar floats para cálculo financeiro.
- [x] **8.4** Cobertura de teste **100%** desta calculadora: cenários A, B, C, D com os valores do exemplo de RN-038 (`valorPago = 40000000n`): verificar `comissao`, `valorLiquido`, `ganhoVsDistrato` para cada cenário.
- [x] **8.5** A `comissao.calculator.ts` é a ÚNICA fonte de cálculo de comissão em todo o sistema — todos os outros módulos (simulador, proposta, financeiro) importam desta função.

### 9. Frontend — Wizard

- [x] **9.1** Criar `apps/web-cedente/src/app/(authenticated)/cadastrar-imovel/page.tsx` — ponto de entrada do wizard.
- [x] **9.2** Ao entrar em "Cadastrar Imóvel": verificar `GET /casos/rascunho/ativo`; se existir rascunho, exibir banner "Você tem um cadastro em andamento: [nome_imovel ou 'Sem endereço']. Última edição: [data]." com botões "Continuar" e "Descartar e começar novo" (RN-023 item 3).
- [x] **9.3** Implementar componente `<WizardStepper>` com indicador de progresso (5 etapas numeradas), navegação "Voltar" e "Avançar", auto-save ao concluir cada etapa.
- [x] **9.4** Etapa 1 — Dados do Imóvel: formulário com campos endereço, cidade, estado (select), construtora, número do contrato. Validação inline on-blur. Verificação de duplicidade ao submeter (RN-090).
- [x] **9.5** Etapa 2 — Dados Financeiros: campos `valor_pago`, `valor_contrato`, `data_inicio_contrato`. Formatação de moeda em tempo real (R$ X.XXX,XX). Exibir `valor_distrato_referencia` calculado ao sair do campo `valor_pago`.
- [x] **9.6** Etapa 3 — Simulador: exibir 4 cards de cenário com valores calculados. Exibir skeleton loader enquanto calcula (RN-021 item 2). Botão "Avançar" desabilitado pelos primeiros **10 segundos** com barra de progresso fina e texto discreto "Analise os cenários por alguns segundos antes de avançar." (RN-021 item 3). Após 10s: transição visual do botão (cor muda, animação de desbloqueio).
- [x] **9.7** Etapa 4 — Escolha do Cenário: 4 cards sem pré-seleção (RN-022 item 3). Ao selecionar: borda de destaque, resumo expandido com animação suave, checkbox "Confirmo minha escolha..." aparece com destaque pulsante. Bloquear avanço sem checkbox marcado com tooltip "Confirme sua escolha para continuar." (RN-022 item 5).
- [x] **9.8** Etapa 5 — Upload de Documentos: checklist dos 6 documentos obrigatórios PF (ou 8 para PJ) com dropzone de upload. Botão "Pular esta etapa" visível (RN-088). Não é a última etapa — há botão "Confirmar Cadastro" separado.
- [x] **9.9** Tela de Confirmação (após Etapa 5): botão "Confirmar Cadastro" chama `POST /casos/rascunho/:id/confirmar`; em caso de sucesso exibir tela de sucesso: "Seu imóvel foi cadastrado com sucesso! Próximos passos: (1) Assinar o Termo de Cadastro (2) Enviar os documentos pendentes." com botões "Ir para Assinaturas" e "Ir para Documentos" (RN-024 item 2).
- [x] **9.10** Chat do Guardião do Retorno visível no canto inferior direito em TODAS as etapas do wizard sem exceção — componente persistente no layout do wizard (RN-089).

---

## 🔀 CROSS-MÓDULO

### 10. Eventos disparados por esta sprint para outros módulos

- [x] **10.1** Após `POST /casos/rascunho/:id/confirmar` (caso criado): publicar no RabbitMQ exchange `cedente.notifications` a mensagem `{ tipo: 'CASO_CRIADO', caso_id, cedente_id }` → módulo de Notificações processa e envia template `NOT-CED-02` ("Dossiê: documentos pendentes") (S4).
- [x] **10.2** Após criação do caso: publicar `{ tipo: 'ENVELOPE_CRIAR', caso_id, tipo_envelope: 'TERMO_CADASTRO', cedente_id }` para o módulo de Assinaturas gerar envelope ZapSign (S4).
- [x] **10.3** Após `PATCH /casos/:id` mudar status para qualquer dos 13 estados: publicar evento `{ tipo: 'STATUS_MUDOU', caso_id, status_novo, status_anterior }` para o módulo de eventos_caso registrar entrada imutável na linha do tempo.
- [x] **10.4** Após `RascunhoExpiracaoCron` identificar rascunho nos dias 7, 15, 25: publicar `{ tipo: 'RASCUNHO_LEMBRETE', caso_id, cedente_id, dia: 7|15|25 }` → módulo de Notificações envia template `NOT-CED-05`.

---

## 🧪 TESTES

### 11. Testes unitários (Vitest)

- [x] **11.1** `comissao.calculator.spec.ts` — 100% cobertura: todos os 4 cenários com valores de RN-038 (`valorPago = 40000000n`); verificar que `cenario A` retorna `comissao=0n`; verificar que cálculo usa bigint sem perda de precisão.
- [x] **11.2** `casos.service.spec.ts` — testar `verificarDuplicidade()`: caso ativo existente → HTTP 409; caso concluído existente → permite novo cadastro (RN-090).
- [x] **11.3** `casos.service.spec.ts` — testar `calcularSimulador()`: retorna 4 cenários calculados corretamente; `simulador_visualizado_em` é persistido.
- [x] **11.4** `casos.service.spec.ts` — testar bloqueio de avanço de cenário sem 10s de simulador visualizado: retorna HTTP 400.
- [x] **11.5** `rascunho-expiracao.cron.spec.ts` — testar que lembretes são enviados nos dias 7, 15, 25; que rascunho é descartado após 30 dias.
- [x] **11.6** `dashboard.service.spec.ts` — testar que `valor_liquido_estimado` é `null` quando não há proposta ativa; testar urgência CRITICA/ALTA/NORMAL conforme dias úteis.

### 12. Testes de integração (Supertest)

- [x] **12.1** Fluxo wizard completo: `POST /casos/rascunho` → `PATCH etapa/1` → `PATCH etapa/2` → `PATCH etapa/3` (verificar `simulador_visualizado_em` salvo) → `PATCH etapa/4` com cenário → `PATCH etapa/5` com documentos → `POST /confirmar` → verificar `casos.status = 'CADASTRO_REALIZADO'` e `rascunho = false`.
- [x] **12.2** Tentar avançar para Etapa 4 em <10s após salvar Etapa 3 → HTTP 400.
- [x] **12.3** Cadastrar imóvel com `numero_contrato` já existente em caso ativo → HTTP 409 (RN-090).
- [x] **12.4** `GET /casos` com filtro `status=EM_ANALISE` → retorna apenas casos com esse status; com busca `q=nome` → retorna casos com nome correspondente; paginação: `per_page=2&page=2` retorna os corretos.
- [x] **12.5** `GET /casos/:id` com token de Cedente B tentando acessar caso do Cedente A → HTTP 403 (RLS).
- [x] **12.6** `PATCH /casos/:id/dados-imovel` com caso em status `APROVADO_PARA_OFERTA` → HTTP 403 (RN-019).

### 13. Testes E2E (Playwright)

- [x] **13.1** T-001: Wizard completo PF — navegação pelas 5 etapas, verificação dos 10s no simulador, seleção de cenário D, confirmação de cadastro, verificação da tela de sucesso.
- [x] **13.2** T-002: Rascunho — iniciar wizard, sair no passo 2, retornar e verificar banner "Você tem um cadastro em andamento".
- [x] **13.3** T-003: Duplicidade — tentar cadastrar imóvel com numero_contrato de caso ativo → verificar mensagem de bloqueio.
- [x] **13.4** T-004: Dashboard estado vazio — primeiro acesso sem casos → verificar boas-vindas, checklist 3 passos, botão CTA.
- [x] **13.5** T-005: Dashboard com dados — verificar 4 cards, painel de próximos passos com cores corretas (vermelho/amarelo), feed de eventos.

---

## ✅ AUTO-VERIFICAÇÃO (12 Checks)

| #   | Check                  | Critério                                                                                                                                                                                                                                                                                                                                                                                                                                           | Status |
| --- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | **Nomenclatura**       | `casos` (não `imóveis`), `rascunho` (não `draft`), `CenarioRetorno` (não `ScenarioType`), `CADASTRO_REALIZADO` (não `created`), `DISPONIVEL_PARA_COMPRADORES` (não `available`), `wizard_step_atual`, `simulador_visualizado_em`, `valor_pago`, `valor_distrato_referencia`                                                                                                                                                                        | ✅     |
| 2   | **Verificabilidade**   | Cada item é binariamente verificável — "implementar wizard" não existe, existe "implementar validação da Etapa 4 bloqueando avanço sem 10s"                                                                                                                                                                                                                                                                                                        | ✅     |
| 3   | **Valores numéricos**  | Simulador mínimo **10 segundos**; rascunho lembretes dias **7, 15, 25**; rascunho descarte **30 dias corridos**; proposta badge amarelo **2 dias úteis**, vermelho **3 dias úteis**, encerramento **5 dias úteis**; cooldown escalonamento **7 dias corridos**; comissão fórmula `20% × (Valor Recuperado − 50% × Valor Pago)`                                                                                                                     | ✅     |
| 4   | **Contagem de itens**  | 5 etapas do wizard cobertas (Etapas 1–5); 4 cards do Dashboard; 5 abas do Detalhe do Caso (Resumo, Documentos, Propostas, Financeiro, Histórico); 13 estados do caso mapeados para nomes amigáveis                                                                                                                                                                                                                                                 | ✅     |
| 5   | **Máquina de estados** | Transições cobertas: Rascunho → CADASTRO_REALIZADO (após confirmar); CADASTRO_REALIZADO → EM_ANALISE (dossiê completo + Termo assinado — bloqueio implementado); EM_ANALISE → PENDENCIA_IDENTIFICADA → EM_ANALISE; bloqueio de edição de dados do imóvel após APROVADO_PARA_OFERTA                                                                                                                                                                 | ✅     |
| 6   | **RBAC**               | `JwtAuthGuard` em todos os endpoints de `/casos` e `/dashboard`; RLS aplicada; `cessionario_id` nunca retornado no response de `/casos`; Admin não pode cadastrar caso pelo painel do Cedente                                                                                                                                                                                                                                                      | ✅     |
| 7   | **Anti-scaffold**      | `comissao.calculator.ts` implementa fórmula real com bigint; `RascunhoExpiracaoCron` implementa lógica real de dias 7/15/25/30; `GET /dashboard/proximos-passos` calcula urgência por dias úteis reais                                                                                                                                                                                                                                             | ✅     |
| 8   | **Glossário**          | `casos` (não `cases`), `dossiê` criado automaticamente ao confirmar, `cenario_retorno` (não `return_scenario`), `valor_pago` em centavos (bigint), `nome_amigavel_status` nunca expõe termos internos                                                                                                                                                                                                                                              | ✅     |
| 9   | **RLS**                | RLS em `casos` garante que Cedente A não acessa casos do Cedente B; projeção Prisma explícita excluindo `cessionario_id` das queries de propostas                                                                                                                                                                                                                                                                                                  | ✅     |
| 10  | **Cache**              | Cache Redis `rs:cedente:dashboard:{user_id}` TTL 60s; invalidação ao receber evento de mudança de estado via RabbitMQ; polling 60s no frontend                                                                                                                                                                                                                                                                                                     | ✅     |
| 11  | **Cross-módulo**       | Eventos publicados no RabbitMQ após criação de caso (para S4): `CASO_CRIADO`, `ENVELOPE_CRIAR`, `STATUS_MUDOU`; evento `RASCUNHO_LEMBRETE` para notificações                                                                                                                                                                                                                                                                                       | ✅     |
| 12  | **Cobertura de REQs**  | REQ-022 (simulador 10s), REQ-023 (escolha ativa cenário), REQ-024 (rascunho 30d), REQ-025 (Termo de Cadastro), REQ-026 (escalonamento descendente), REQ-027 (um nível por vez), REQ-028 (cooldown 7d), REQ-029 (subida proibida), REQ-030 (recebimento proposta), REQ-031–043 (propostas/negociação/comissão/desistência), REQ-062–063 (eventos dashboard), REQ-090 (duplicidade), REQ-091 (propostas expiradas só no histórico) — 100% atribuídos | ✅     |

---

## 📋 PENDÊNCIAS

| ID      | Tipo       | Descrição                                                                                                                                                                                                           |
| ------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PEND-04 | ⚠️ AMBÍGUO | RN-021 diz "se o cálculo exceder 5 segundos" exibir animação, mas não define timeout máximo. Adotado: timeout de 10s no backend, após isso retornar simulação com valores 0 e flag `calculando: true` para re-poll. |
| PEND-05 | ⚠️ AMBÍGUO | RN-023 diz "ao final de cada etapa concluída" — interpretado como: ao clicar em "Avançar" com sucesso na etapa, não ao digitar cada campo. Auto-save é por etapa, não por campo.                                    |
