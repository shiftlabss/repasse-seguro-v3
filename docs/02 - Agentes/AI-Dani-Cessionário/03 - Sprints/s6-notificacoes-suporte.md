# S6 — Notificações, Alertas e Suporte Operacional

| **Sprint**         | S6 — Notificações, Alertas e Suporte Operacional                                                                                   |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Template**       | B — Módulo Fullstack (organizado por Feature com vertical slice Banco→Backend→Frontend→Wiring→Testes)                              |
| **REQs cobertos**  | REQ-028, REQ-065, REQ-079, REQ-080, REQ-081, REQ-109, REQ-110, REQ-111, REQ-112, REQ-113, REQ-114, REQ-115, REQ-116                |
| **Docs fonte**     | D01 (RN-DC-022, RN-DC-028, RN-DC-029), D09 (T-DC-001 alertas), D16 (§7.3), D21 (templates TPLF-001–010, canais, RabbitMQ, opt-out) |
| **Total de itens** | 48                                                                                                                                 |

---

## 🎯 Objetivo

Implementar o sistema completo de notificações assíncronas via RabbitMQ: `NotificacaoService` com publicação na fila `dani.notificacoes`, `NotificationProcessor` consumer, 10 templates (`TPLF-001` a `TPLF-010`), 3 endpoints de alertas, preferências de opt-out, badge FAB, painel de alertas, Dead Letter Handler, `AlertaModule` completo e suporte operacional (respostas a dúvidas de plataforma, recusa de submissão de proposta, comportamento em latência alta).

---

## Feature 1 — NotificacaoService e Infraestrutura de Filas

### 🗄️ Banco de Dados

- [x] **[BANCO-NOT-001]** Verificar e ajustar tabela `dani_alertas` para suportar todos os campos do sistema:
  - Sub-item: Campos presentes: `id UUID PK`, `cessionario_id UUID NOT NULL`, `tipo AlertaTipo NOT NULL`, `status StatusAlerta NOT NULL` (`PENDENTE | ENVIADO | LIDO | FALHA`), `payload JSONB NOT NULL`, `prioridade TEXT NOT NULL` (`critico | alto | normal | baixo`), `lido_em TIMESTAMPTZ`, `expires_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ`
  - Sub-item: Índice composto `@@index([cessionario_id, status, created_at])` para queries de listagem paginada
  - Sub-item: Índice `@@index([cessionario_id, status])` WHERE `status != 'LIDO'` para `GET /dani/alertas/count` (explain confirma uso)
  - Sub-item: `cessionario_id` tem foreign key para tabela de cessionários da plataforma; sem exposição de UUID raw em logs (REQ-107)

- [x] **[BANCO-NOT-002]** Verificar tabela de tracking de alertas (schema `dani_rate_limits_audit` ou `alerta_tracking`):
  - Sub-item: Campos: `id UUID PK`, `alerta_id UUID FK(dani_alertas.id)`, `cessionario_id UUID NOT NULL`, `evento EventoRateLimitAudit` (estende para tracking: `sent | delivered | read | clicked | failed | opted_out`), `canal TEXT`, `timestamp TIMESTAMPTZ`, `metadata JSONB`
  - Sub-item: Índice em `(alerta_id, evento)` para queries de métricas de entrega

---

### 🔧 Backend

- [x] **[BACK-NOT-001]** Implementar `NotificacaoService` em `src/notificacoes/notificacao.service.ts`:
  - Sub-item: Método `publicar(evento: NotificacaoEvento): Promise<void>` — nunca retorna dado de negócio, apenas lança ou confirma publicação
  - Sub-item: Valida `Templates[evento.template_id]` existe; se não → lança erro com código `ALERTA_001`, mensagem `"ALERTA_001: template ${template_id} não encontrado"`
  - Sub-item: Valida `evento.cessionario_id` presente para todo evento com `tipo !== 'admin_alert'`; se ausente → lança `ALERTA_002`
  - Sub-item: Publica na exchange `dani.notifications` com routing key `notificacao.nova`; `persistent: true`; `expiration: 3600000` (TTL 1h)
  - Sub-item: Log Pino após publicação: `{ event: 'notification_published', template_id, canal, cessionario_id: sha256(cessionario_id) }` — nunca UUID raw (REQ-107)
  - Sub-item: NUNCA chama canal de envio diretamente — apenas publica na fila (anti-pattern: envio síncrono na request principal conforme D21 §5.2)

- [x] **[BACK-NOT-002]** Implementar os 10 templates em `src/notificacoes/templates/index.ts`:
  - Sub-item: `TPLF-001` (Boas-vindas Dani): gatilho = primeiro acesso ao chat KYC aprovado; canal = In-App; variáveis = `{ nome }`; opt-out = Não
  - Sub-item: `TPLF-002` (Nova Oportunidade): gatilho = nova OPR publicada com match no perfil; canal = In-App + WhatsApp (Fase 2); variáveis = `{ opr_id, cidade, delta, risco, deep_link }`; opt-out = Sim (alertas de oportunidade)
  - Sub-item: `TPLF-003` (Oportunidade Voltou): gatilho = OPR retorna ao status DISPONIVEL; canal = In-App + WhatsApp (Fase 2); variáveis = `{ opr_id, cidade, delta, deep_link }`; opt-out = Sim
  - Sub-item: `TPLF-004` (ZapSign D+2): gatilho = D+2 após aceite sem assinatura; canal = In-App + WhatsApp (Fase 2); variáveis = `{ opr_id, prazo_restante, link_assinatura }`; opt-out = Não (crítico)
  - Sub-item: `TPLF-005` (ZapSign D+4): gatilho = D+4 sem assinatura (urgente); canal = In-App + WhatsApp (Fase 2); variáveis = `{ opr_id, link_assinatura, prazo_expiracao }`; opt-out = Não (crítico)
  - Sub-item: `TPLF-006` (ZapSign Expirado): gatilho = D+5 sem assinatura; canal = In-App; variáveis = `{ opr_id, link_suporte }`; opt-out = Não (crítico)
  - Sub-item: `TPLF-007` (Prazo Escrow Crítico): gatilho = 1 dia útil restante para depósito; canal = In-App + WhatsApp (Fase 2); variáveis = `{ opr_id, valor_escrow, prazo, link_escrow }`; opt-out = Não (crítico)
  - Sub-item: `TPLF-008` (OTP Vinculação): gatilho = Cessionário solicita vincular número; canal = SMS; variáveis = `{ otp, expiracao_min }`; opt-out = Não (crítico); conteúdo SMS: "Seu código de verificação Repasse Seguro é: [XXXXXX]. Válido por 15 minutos."
  - Sub-item: `TPLF-009` (WhatsApp Vinculado): gatilho = confirmação da vinculação; canal = In-App + WhatsApp (Fase 2); variáveis = `{ numero_mascarado }`; opt-out = Não
  - Sub-item: `TPLF-010` (Alerta Admin — Dani Desligada): gatilho = taxa de erro > 30% em 15 min (RN-DC-024); canal = Admin Alert (Sentry + Pino error + Slack #alertas-criticos Fase 2); variáveis = `{ taxa_erro, timestamp, environment }`; opt-out = Não
  - Sub-item: Função `canOptOut(template_id: string): boolean` — retorna `false` para `TPLF-004, TPLF-005, TPLF-006, TPLF-007, TPLF-008, TPLF-010`

- [x] **[BACK-NOT-003]** Implementar `NotificationProcessor` consumer em `src/notificacoes/notification.processor.ts`:
  - Sub-item: `@RabbitSubscribe({ queue: 'dani.notificacoes', exchange: 'dani.notifications' })` — subscribe com `ack` manual
  - Sub-item: Roteamento por `evento.canal`: `in_app` → `AlertaModule.salvar()`; `whatsapp` → publica em `dani.whatsapp`; `sms` → `SMSProvider.send()`; `admin_alert` → `sentry.captureMessage()` + Pino `error`
  - Sub-item: Tracking: após processamento bem-sucedido → salva evento `delivered` em `alerta_tracking`
  - Sub-item: Retry policy: `dani.notificacoes` máx 3 tentativas, backoff `5s → 15s → 30s`; após 3 falhas → envia para DLQ `dani.notificacoes.dlq` (D21 §5.1)
  - Sub-item: Notificações com `NODE_ENV === 'development'`: intercepta envios externos; WhatsApp e SMS logados no console via `NotificationInterceptor`; OTP nunca exibido em produção

- [x] **[BACK-NOT-004]** Implementar Dead Letter Handler em `src/notificacoes/dead-letter.handler.ts`:
  - Sub-item: `@RabbitSubscribe({ queue: 'dani.notificacoes.dlq' })` — processa mensagens mortas
  - Sub-item: Log Pino `error`: `{ event: 'notification_dead_letter', template_id, canal, cessionario_id: sha256(cessionario_id), tentativas: retryCount }`
  - Sub-item: Se `message.prioridade === 'critico'` → captura Sentry `captureMessage('DLQ crítica — notificação não entregue', 'error')` com alerta P0 (D21 §5.3)
  - Sub-item: Salva evento `failed` em `alerta_tracking` com `metadata.tentativa = retryCount` e `metadata.erro = error.message` (sem stack trace — REQ-107)
  - Sub-item: DLQ size > 10 → alerta P1 automático (threshold D21 §6.3)

---

### ✅ Testes

- [x] **[TEST-NOT-001]** Testes unitários de `NotificacaoService`:
  - Sub-item: Template válido + `cessionario_id` presente → `amqpMock.publish` chamado 1x com exchange `dani.notifications`, `persistent: true`
  - Sub-item: Template inválido → lança erro com código `ALERTA_001`
  - Sub-item: `tipo !== 'admin_alert'` + `cessionario_id` ausente → lança `ALERTA_002`
  - Sub-item: Log gerado com `cessionario_id` como `sha256()` — nunca UUID raw (validar com spy no logger)
  - Sub-item: `canOptOut('TPLF-004')` → `false`; `canOptOut('TPLF-007')` → `false`; `canOptOut('TPLF-002')` → `true`

- [x] **[TEST-NOT-002]** Testes de integração do `NotificationProcessor`:
  - Sub-item: Evento in-app válido → `AlertaModule.salvar()` chamado; evento `delivered` salvo em tracking
  - Sub-item: Falha no envio (mock rejeita) → retry 3x; após 3 falhas → `dlqMock.publish` chamado 1x
  - Sub-item: Notificação crítica na DLQ → `sentry.captureMessage` chamado
  - Sub-item: `NODE_ENV === 'development'` → `NotificationInterceptor` intercepta; canal externo nunca chamado

---

## Feature 2 — AlertaModule: Endpoints e Badge FAB

### 🔧 Backend

- [x] **[BACK-ALERTA-001]** Implementar `AlertaController` em `src/alerta/alerta.controller.ts` com 3 endpoints:
  - Sub-item: `GET /dani/alertas` — guards `@UseGuards(JwtAuthGuard, CessionarioOwnerGuard)`; query params: `page` (int, default 1), `per_page` (int, default 20, max 100), `status` (`PENDENTE | ENVIADO | LIDO | FALHA`), `tipo`; filtra sempre por `cessionario_id` do JWT (isolamento REQ-003)
  - Sub-item: `GET /dani/alertas/count` — retorna `{ data: { nao_lidos: number } }`; conta registros WHERE `status != 'LIDO' AND cessionario_id = :id`; cache Redis 30s por `cessionario_id` (polling badge FAB a cada 30s — D21 §2.1)
  - Sub-item: `PATCH /dani/alertas/:alerta_id/lido` — valida que alerta pertence ao `cessionario_id` do JWT; se inexistente → 404 `ALERTA_NOT_FOUND`; se já lido → 422 `ALERTA_JA_LIDO`; atualiza `status = 'LIDO'`, `lido_em = NOW()`; salva evento `read` em `alerta_tracking`
  - Sub-item: `GET /dani/alertas/preferencias` — retorna `PreferenciasNotificacao` do Cessionário (modelo completo de D21 §4.1)
  - Sub-item: `PUT /dani/alertas/preferencias` — atualiza `preferencias.canais` e `preferencias.tipos`; validação: não permite desativar tipos críticos (`negociacao_status`, `zapsign_lembrete`, `escrow_prazo`)

- [x] **[BACK-ALERTA-002]** Implementar `AlertaService.salvar(payload: AlertaInApp): Promise<dani_alertas>`:
  - Sub-item: Persiste alerta em `dani_alertas` com `deep_link` obrigatório — alerta sem `deep_link` lança `ALERTA_003` (anti-pattern D21 §2.1)
  - Sub-item: Invalida cache `GET /dani/alertas/count` do `cessionario_id` após salvar
  - Sub-item: Salva evento `sent` em `alerta_tracking` após persistência
  - Sub-item: Alertas de oportunidade (`tipo = 'NOVA_OPORTUNIDADE' | 'OPORTUNIDADE_RETORNO'`): verifica opt-out do Cessionário antes de salvar — se opt-out ativo → não salva e retorna silenciosamente

---

### 🖥️ Frontend

- [x] **[FRONT-ALERTA-001]** Implementar badge de alertas não lidos no FAB (T-DC-001):
  - Sub-item: Hook `useAlertasCount()` usando TanStack Query com `refetchInterval: 30000` (30s polling — D21 §2.1) para `GET /api/v1/dani/alertas/count`
  - Sub-item: Badge exibida quando `nao_lidos > 0`; oculta quando `nao_lidos === 0` (D09 §3.1)
  - Sub-item: Badge máximo exibe "99+" para `nao_lidos > 99`
  - Sub-item: Animação: badge aparece com `scale + bounce`, 200ms, Framer Motion `spring` (D09 §3.1); `useReducedMotion()` desabilita animação
  - Sub-item: Skeleton area: `Skeleton` cobrindo área do badge enquanto `useAlertasCount` está em `isLoading`

- [x] **[FRONT-ALERTA-002]** Implementar painel de alertas (acessível via clique no FAB quando `nao_lidos > 0`):
  - Sub-item: **Estado Skeleton** — 3 linhas de `<Skeleton className="h-16 w-full rounded-lg" />` enquanto `GET /dani/alertas` está loading
  - Sub-item: **Estado Empty** — `<EmptyState>` com texto "Nenhum alerta no momento." + botão "Ativar alertas de oportunidades"
  - Sub-item: **Estado Error** — `<Alert variant="destructive">` + "Não foi possível carregar seus alertas. Tente novamente." + retry
  - Sub-item: **Estado Populated** — lista de alertas paginada; cada item com `tipo` badge, `titulo`, `corpo`, `deep_link` clicável; alertas não lidos com indicador visual (ponto azul `--primary`)
  - Sub-item: Clique em alerta → chama `PATCH /dani/alertas/:id/lido`; navega para `deep_link`; invalida cache `alertas/count`
  - Sub-item: Botão "Marcar todos como lido" — chama PATCH para cada alerta não lido em paralelo (max concurrency 5)

---

### 🔌 Wiring

- [x] **[WIRE-ALERTA-001]** Integrar polling de alertas ao layout global:
  - Sub-item: `useAlertasCount` montado no componente raiz `AppLayout` — ativo em todas as telas do módulo Cessionário
  - Sub-item: Cache TanStack Query `queryKey: ['dani', 'alertas', 'count', cessionario_id]`; `staleTime: 25000` (25s, ligeiramente menor que refetchInterval 30s)
  - Sub-item: Após `PATCH /lido`: `queryClient.invalidateQueries(['dani', 'alertas'])` e `['dani', 'alertas', 'count']`

---

### ✅ Testes

- [x] **[TEST-ALERTA-001]** Testes de integração dos endpoints de alertas:
  - Sub-item: `GET /dani/alertas` — JWT válido → 200 com `meta.total`; JWT de outro Cessionário → 403 `AUTH_009` (CessionarioOwnerGuard não mockado — REQ-149)
  - Sub-item: `GET /dani/alertas/count` — `nao_lidos` correto para Cessionário A; `nao_lidos` de Cessionário B não visível para Cessionário A
  - Sub-item: `PATCH /dani/alertas/:id/lido` — alerta do Cessionário → 200 com `status: 'LIDO'`; alerta de outro Cessionário → 404 `ALERTA_NOT_FOUND`; alerta já lido → 422 `ALERTA_JA_LIDO`
  - Sub-item: Evento `read` salvo em `alerta_tracking` após PATCH bem-sucedido
  - Sub-item: `PUT /dani/alertas/preferencias` com `negociacao_status: false` → 422 (não pode desativar tipo crítico)

---

## Feature 3 — Suporte Operacional (RN-DC-022, RN-DC-028, RN-DC-029)

### 🔧 Backend

- [x] **[BACK-SUP-001]** Implementar lógica de suporte operacional no `AgenteService` (System Prompt + intent handling):
  - Sub-item: **Suporte a regras da plataforma (RN-DC-022)**: tópicos cobertos no System Prompt: KYC (documentos exigidos, prazo ≤30min automatizado / ≤2 dias úteis revisão manual), Escrow (prazo padrão 10 dias úteis, extensão +5 dias úteis consultando Cedente em 24h com silêncio = aprovação automática, reversão 15 dias corridos), ZapSign (Envelope ZapSign, prazo 5 dias úteis, régua D+2 e D+4, expiração D+5), Status de proposta e negociação
  - Sub-item: Para tópicos fora do escopo (jurídico, fiscal, contrato individual específico): Dani exibe mensagem de redirecionamento com link clicável para canal de suporte (string de D01 RN-DC-022 §3)
  - Sub-item: **Recusa de submissão em nome do Cessionário (RN-DC-028)**: detector de intenção `SUBMIT_PROPOSTA_BY_AGENT` no classificador de intent; resposta normativa: "Posso preparar a análise completa, mas a submissão da proposta precisa ser feita por você. Acesse a tela da oportunidade para confirmar." + botão "Ir para a oportunidade"
  - Sub-item: Na segunda insistência de submissão: "Entendo que seria mais prático, mas por segurança, apenas você pode confirmar a proposta na plataforma." — Dani não submete em nenhuma circunstância
  - Sub-item: Prazos operacionais armazenados como constantes imutáveis em `src/agente/constants/prazos-operacionais.ts`: `ESCROW_PADRAO_DIAS_UTEIS = 10`, `ESCROW_EXTENSAO_DIAS_UTEIS = 5`, `ESCROW_REVERSAO_DIAS_CORRIDOS = 15`, `ZAPSIGN_PRAZO_DIAS_UTEIS = 5`, `KYC_AUTOMATIZADO_MIN = 30`, `KYC_MANUAL_DIAS_UTEIS = 2`

- [x] **[BACK-SUP-002]** Implementar comportamento em latência alta (RN-DC-029):
  - Sub-item: Se resposta > SLA (5s para análise individual, 10s para comparação): mantém streaming SSE ativo + typing indicator no frontend
  - Sub-item: Se após 2× SLA (10s análise / 20s comparação) resposta ainda não entregue: emite evento SSE `{ type: 'warning', message: "A Dani está demorando mais que o esperado. Você pode aguardar ou tentar novamente em instantes.", actions: ['AGUARDAR', 'RETENTAR'] }`
  - Sub-item: Se latência alta persistir por 5 minutos consecutivos: publica alerta para Admin em `dani.agent_monitor`; estado do agente pode transicionar para `LATENCIA_ALTA` (REQ-100)
  - Sub-item: `RETENTAR`: reenvia automaticamente a última mensagem (frontend armazena `ultima_mensagem` em estado local)

---

### 🖥️ Frontend

- [x] **[FRONT-SUP-001]** Implementar handling de evento SSE de latência alta:
  - Sub-item: Ao receber evento SSE `type: 'warning'`: exibe bubble de sistema com texto normativo e dois botões `<Button variant="outline" size="sm">Aguardar</Button>` e `<Button variant="default" size="sm">Tentar novamente</Button>`
  - Sub-item: "Aguardar": fecha o warning bubble; mantém typing indicator visível
  - Sub-item: "Tentar novamente": reenvia a `ultima_mensagem` via nova requisição SSE; cancela a requisição SSE anterior com `AbortController`
  - Sub-item: `aria-live="assertive"` na bubble de warning — screen reader anuncia imediatamente

---

### ✅ Testes

- [x] **[TEST-SUP-001]** Testes unitários de lógica de suporte:
  - Sub-item: Intent `SUBMIT_PROPOSTA_BY_AGENT` → response contém `user_message` normativa de RN-DC-028 e botão "Ir para a oportunidade"
  - Sub-item: Segunda insistência de submissão → segunda mensagem normativa; nenhuma proposta submetida; verificar que nenhum endpoint da plataforma foi chamado
  - Sub-item: Pergunta sobre prazo de Escrow → resposta contém `10 dias úteis` e `+5 dias úteis` (constantes imutáveis verificadas)
  - Sub-item: Pergunta sobre KYC → resposta contém `≤ 30 minutos` (automatizado) e `≤ 2 dias úteis` (revisão manual)
  - Sub-item: Pergunta sobre assessoria jurídica → message de redirecionamento; não tenta responder o conteúdo jurídico

---

## Feature 4 — Preferências de Opt-out e LGPD (D21 §4)

### 🔧 Backend

- [x] **[BACK-OPT-001]** Implementar modelo de `PreferenciasNotificacao` e persistência:
  - Sub-item: Interface `PreferenciasNotificacao` conforme D21 §4.1: `canais: { in_app: boolean (default: true), whatsapp: boolean (default: false) }` + `tipos: { oportunidade_nova, oportunidade_retorno (opt-out permitido), negociacao_status, zapsign_lembrete, escrow_prazo (opt-out NÃO permitido) }`
  - Sub-item: Armazenado em `dani_sessoes.preferencias JSONB` ou tabela dedicada; `updated_at` sempre atualizado
  - Sub-item: `PUT /dani/alertas/preferencias` — valida que campos de opt-out não-permitido nunca chegam como `false` no body; se tentativa → 422 com lista dos campos não configuráveis

- [x] **[BACK-OPT-002]** Implementar verificação de opt-out em `NotificationProcessor`:
  - Sub-item: Antes de roteamento de cada evento: verifica preferências do `cessionario_id`
  - Sub-item: Se `tipo = 'OPORTUNIDADE_NOVA'` e `preferencias.tipos.oportunidade_nova === false` → descarta evento (ack sem processar); salva evento `opted_out` em `alerta_tracking`
  - Sub-item: Se `tipo = 'NOVA_OPORTUNIDADE'` e `preferencias.canais.whatsapp === false` → envia apenas in-app; não enfileira em `dani.whatsapp`
  - Sub-item: Notificações críticas (`TPLF-004/005/006/007/008/010`) → sempre processadas, ignorando `preferencias`

---

### ✅ Testes

- [x] **[TEST-OPT-001]** Testes unitários de preferências e opt-out:
  - Sub-item: Opt-out de `oportunidade_nova` → evento `TPLF-002` descartado; `opted_out` salvo em tracking
  - Sub-item: Opt-out de `whatsapp` → `TPLF-002` enviado apenas in-app; `dani.whatsapp` não recebe mensagem
  - Sub-item: Tentativa de opt-out de `zapsign_lembrete` via `PUT /dani/alertas/preferencias` → 422
  - Sub-item: `TPLF-007` (Escrow crítico) com `escrow_prazo: false` ignorado nas prefs → alerta SEMPRE enviado
  - Sub-item: Opt-out `TPLF-002`: log `opted_out` salvo; `dani_alertas` NÃO tem novo registro para este evento

---

## 🔍 Auto-verificação S6 (12 checks)

| #   | Check                                                                                                                                                                                                                                                                                   | Status |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                                                                                                                            | ✅     |
| 2   | Nomes exatos: `NotificacaoService`, `NotificationProcessor`, `AlertaController`, `AlertaService`, `dead-letter.handler.ts`, `NotificationInterceptor`, `PreferenciasNotificacao`, `alerta_tracking`, `dani_alertas`, `dani.notificacoes`, `dani.notificacoes.dlq`, `dani.notifications` | ✅     |
| 3   | Valores numéricos: 10 templates (TPLF-001–010), retry 3x, backoff 5s/15s/30s, TTL mensagem fila 3600000ms (1h), polling badge 30s, cache count 30s, DLQ alert > 10 itens → P1                                                                                                           | ✅     |
| 4   | Templates críticos sem opt-out: TPLF-004, TPLF-005, TPLF-006, TPLF-007, TPLF-008, TPLF-010 documentados explicitamente                                                                                                                                                                  | ✅     |
| 5   | Glossário D10: `AlertaTipo`, `StatusAlerta`, `EventoRateLimitAudit`, `Envelope ZapSign`, `Escrow` — termos exatos                                                                                                                                                                       | ✅     |
| 6   | Prazos operacionais imutáveis: Escrow 10d úteis + extensão 5d úteis + reversão 15d corridos; ZapSign 5d úteis (D+2/D+4/D+5); KYC ≤30min / ≤2d úteis                                                                                                                                     | ✅     |
| 7   | RabbitMQ: exchanges e DLQs nomeados exatamente: `dani.notificacoes`, `dani.whatsapp`, `dani.agent_monitor`, `dani.notifications` (fanout)                                                                                                                                               | ✅     |
| 8   | Nenhuma suposição sem base documental                                                                                                                                                                                                                                                   | ✅     |
| 9   | Anti-pattern documentado: envio síncrono na request principal proibido; `NotificacaoService.publicar()` nunca chama canal diretamente                                                                                                                                                   | ✅     |
| 10  | Nenhum item de scaffold — todos com sub-itens verificáveis (R10)                                                                                                                                                                                                                        | ✅     |
| 11  | `CessionarioOwnerGuard` não mockado em testes de integração (REQ-149)                                                                                                                                                                                                                   | ✅     |
| 12  | REQs REQ-028, REQ-065, REQ-079–081, REQ-109–116 têm ≥1 item de checklist                                                                                                                                                                                                                | ✅     |

---

## REQs cobertos por esta sprint

REQ-028 (recusa de submissão de proposta pela Dani), REQ-065 (`dani_alertas` tabela + campos + índices), REQ-079 (`GET /dani/alertas`), REQ-080 (`GET /dani/alertas/count`), REQ-081 (`PATCH /dani/alertas/:id/lido`), REQ-109 (arquitetura assíncrona RabbitMQ — nunca síncrono), REQ-110 (10 templates TPLF-001–010 com variáveis, gatilhos, canais), REQ-111 (opt-out por tipo e canal; críticos sem opt-out), REQ-112 (Dead Letter Handler + alerta P0 para críticos), REQ-113 (tracking sent/delivered/read/clicked/failed/opted_out), REQ-114 (polling badge FAB 30s), REQ-115 (suporte operacional: KYC, Escrow, ZapSign — prazos exatos), REQ-116 (comportamento em latência alta — RN-DC-029)
