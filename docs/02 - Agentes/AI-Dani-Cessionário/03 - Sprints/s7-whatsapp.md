# S7 — WhatsApp Fase 2: Vinculação, OTP e Canal WhatsApp

| **Sprint**         | S7 — WhatsApp Fase 2: Vinculação, OTP e Canal WhatsApp                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| **Template**       | B — Módulo Fullstack (organizado por Feature com vertical slice Banco→Backend→Frontend→Wiring→Testes)                    |
| **REQs cobertos**  | REQ-036, REQ-037, REQ-038, REQ-039, REQ-082, REQ-083, REQ-084, REQ-085, REQ-086, REQ-128                                 |
| **Docs fonte**     | D01 (RN-DC-040, RN-DC-041, RN-DC-042, RN-DC-044), D09 (T-DC-012), D16 (§7.4), D17 (EvolutionAPI), D21 (§2.2, §2.3, §4.3) |
| **Total de itens** | 47                                                                                                                       |

---

## 🎯 Objetivo

Implementar o canal WhatsApp (Fase 2): fluxo completo de vinculação em 3 etapas (informar número → OTP SMS → confirmação WhatsApp), `WhatsappModule` (NestJS), `WebhookApiKeyGuard`, processamento de mensagens recebidas via EvolutionAPI (incluindo comando `PARAR` LGPD), T-DC-012 (tela de vinculação no perfil do Cessionário), estado `dani_vinculacoes_whatsapp`, fila `dani.whatsapp` e toda a máquina de estados da vinculação.

---

## Feature 1 — `WhatsappModule` e Máquina de Estados da Vinculação

### 🗄️ Banco de Dados

- [x] **[BANCO-WA-001]** Verificar tabela `dani_vinculacoes_whatsapp` e seus campos:
  - Sub-item: Campos: `id UUID PK`, `cessionario_id UUID NOT NULL UNIQUE`, `phone_hash TEXT NOT NULL UNIQUE` (bcrypt hash completo), `phone_suffix TEXT NOT NULL` (últimos 4 dígitos para exibição mascarada `"••••-XXXX"`), `estado EstadoVinculacaoWhatsapp NOT NULL`, `vinculado_em TIMESTAMPTZ`, `desvinculado_em TIMESTAMPTZ`, `confirmation_token TEXT` (token de confirmação etapa 2/3, hash), `confirmation_expires_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`
  - Sub-item: Enum `EstadoVinculacaoWhatsapp`: `NAO_VINCULADO | OTP_SMS_ENVIADO | AGUARDANDO_CONFIRMACAO_WA | VINCULADO`; campo `NAO_VINCULADO` é estado inicial e após desvinculação
  - Sub-item: Índice `@@index([cessionario_id])` e `@@unique([phone_hash])` para verificar duplicidade
  - Sub-item: Número nunca armazenado em texto claro — apenas `phone_hash` (bcrypt cost 12 conforme REQ-096) + `phone_suffix` para exibição (D09 T-DC-012)

- [x] **[BANCO-WA-002]** Verificar Redis keys para OTP:
  - Sub-item: `dani:otp:{phone_hash}` → valor = OTP hash (bcrypt); TTL = 900s (15 min — RN-DC-041)
  - Sub-item: `dani:rate:otp:{phone_hash}` → contador de tentativas; TTL = 3600s (1 hora — janela deslizante — RN-DC-041)
  - Sub-item: `dani:block:otp:{phone_hash}` → flag de hard block; TTL = 1800s (30 min após 5 falhas consecutivas — RN-DC-041)
  - Sub-item: `dani:otp:cooldown:{phone_hash}` → cooldown de reenvio; TTL = 60s (botão "Reenviar código" — RN-DC-041)

---

### 🔧 Backend

- [x] **[BACK-WA-001]** Implementar `WhatsappModule` em `src/whatsapp/whatsapp.module.ts`:
  - Sub-item: Providers: `WhatsappService`, `WhatsappController`, `WebhookApiKeyGuard`
  - Sub-item: Imports: `PrismaModule`, `RedisModule`, `AmqpModule` (para publicar em `dani.whatsapp`)
  - Sub-item: Não importa `AgenteModule` diretamente — mensagens recebidas via WhatsApp são roteadas ao `AgenteService` via injeção ou evento interno

- [x] **[BACK-WA-002]** Implementar `WhatsappService.iniciarVinculacao(numero_telefone, cessionario_id)`:
  - Sub-item: Valida `numero_telefone` no formato E.164 (`+55DDXXXXXXXXX`); se inválido → lança `WhatsappException` código `WA_NUMERO_INVALIDO`, status 400, `user_message: "O número informado não parece ser válido. Verifique o DDD e tente novamente."`
  - Sub-item: Verifica `EXISTS dani:block:otp:{phone_hash}`; se existir → lança `WA_OTP_HARD_BLOCK`, status 429, inclui `reset_at` no erro
  - Sub-item: Verifica `dani_vinculacoes_whatsapp.estado` do `cessionario_id`: se `VINCULADO` → lança `WA_JA_VINCULADO`, status 409
  - Sub-item: Verifica `phone_hash` único: se `phone_hash` já existe com outro `cessionario_id` → lança `WA_NUMERO_JA_ASSOCIADO`, status 409, `user_message: "Este número já está associado a outra conta. Se acredita que há um erro, entre em contato com o suporte."`
  - Sub-item: Verifica rate limit: `INCR dani:rate:otp:{phone_hash}`; se > 3 → lança `WA_OTP_RATE_LIMIT`, status 429
  - Sub-item: Gera OTP de 6 dígitos; armazena hash bcrypt (cost 12) em `dani:otp:{phone_hash}` TTL 900s
  - Sub-item: Publica envio de SMS na fila `dani.notificacoes` com template `TPLF-008` e variáveis `{ otp, expiracao_min: 15 }`; NUNCA envia SMS síncrono
  - Sub-item: Cria/atualiza registro em `dani_vinculacoes_whatsapp` com `estado = 'OTP_SMS_ENVIADO'`
  - Sub-item: Retorna `{ estado: 'OTP_SMS_ENVIADO', expires_at: now + 900s }`

- [x] **[BACK-WA-003]** Implementar `WhatsappService.verificarOtp(codigo, cessionario_id)`:
  - Sub-item: Busca `dani_vinculacoes_whatsapp` do `cessionario_id`; se `estado !== 'OTP_SMS_ENVIADO'` → lança `WA_ESTADO_INVALIDO`, status 422
  - Sub-item: Verifica `EXISTS dani:block:otp:{phone_hash}`; se existir → lança `WA_OTP_HARD_BLOCK`, status 429
  - Sub-item: Busca `dani:otp:{phone_hash}`; se não existe (expirou) → lança `WA_OTP_EXPIRADO`, status 400, `user_message: "O código expirou. Solicite um novo código para continuar."`
  - Sub-item: Compara `bcrypt.compare(codigo, otp_hash)`; se incorreto:
    - Incrementa `dani:rate:otp:{phone_hash}` (TTL 3600s janela deslizante)
    - Incrementa contador de falhas consecutivas; se >= 5 → `SET dani:block:otp:{phone_hash} 1 EX 1800` (hard block 30 min)
    - Se `rate:otp` >= 3 → lança `WA_OTP_RATE_LIMIT`, status 429 com `reset_at`
    - Caso contrário → lança `WA_OTP_INVALIDO`, status 400, `user_message: "O código informado não está correto."`
  - Sub-item: OTP correto: deleta `dani:otp:{phone_hash}` e `dani:rate:otp:{phone_hash}`; reseta contador de falhas consecutivas
  - Sub-item: Gera `confirmation_token` UUID único; armazena hash em `dani_vinculacoes_whatsapp.confirmation_token`; `confirmation_expires_at = now + 24h` (RN-DC-042)
  - Sub-item: Publica mensagem de boas-vindas ao WhatsApp via `dani.notificacoes` / `dani.whatsapp` com `confirmation_token` para o Cessionário confirmar
  - Sub-item: Atualiza `estado = 'AGUARDANDO_CONFIRMACAO_WA'`
  - Sub-item: Retorna `{ estado: 'AGUARDANDO_CONFIRMACAO_WA', mensagem: "Código correto! Confirme a vinculação no WhatsApp." }`

- [x] **[BACK-WA-004]** Implementar `WhatsappService.desvincular(cessionario_id, motivo: 'PLATAFORMA' | 'LGPD_PARAR')`:
  - Sub-item: `motivo = 'PLATAFORMA'`: requer confirmação prévia (modal no frontend — D09 T-DC-012); atualiza `estado = 'NAO_VINCULADO'`, `desvinculado_em = NOW()`; loga `{ cessionario_id_hash: sha256(cessionario_id), tipo: 'desvinculacao_plataforma', timestamp }`
  - Sub-item: `motivo = 'LGPD_PARAR'`: processamento IMEDIATO sem confirmação; atualiza `estado = 'NAO_VINCULADO'` atomicamente; loga `{ cessionario_id_hash, phone_hash, tipo: 'opt_out_lgpd', timestamp }` conforme D21 §4.3; NUNCA enfileira este processamento
  - Sub-item: Após desvinculação: invalida `dani:block:otp:{phone_hash}`, `dani:rate:otp:{phone_hash}`, `dani:otp:{phone_hash}`
  - Sub-item: Desvinculação LGPD deve ser processada em < 1s (não pode estar sob retry da fila)

---

### 🔧 Backend — Webhook e Processamento de Mensagens

- [x] **[BACK-WA-005]** Implementar `WebhookApiKeyGuard` em `src/auth/guards/webhook-api-key.guard.ts`:
  - Sub-item: Verifica header `x-api-key` igual a `process.env.EVOLUTIONAPI_WEBHOOK_SECRET`
  - Sub-item: Chave ausente ou incorreta → lança `AuthException` código `WA_WEBHOOK_INVALID_KEY`, status 401
  - Sub-item: NUNCA usa `JwtAuthGuard` ou `CessionarioOwnerGuard` neste endpoint — é autenticação de sistema, não de usuário (D02 WIRE-AUTH-001)

- [x] **[BACK-WA-006]** Implementar `WhatsappController` com 5 endpoints:
  - Sub-item: `GET /whatsapp/status` — `@UseGuards(JwtAuthGuard, CessionarioOwnerGuard)`; retorna `{ estado, numero_sufixo, vinculado_em }` do `dani_vinculacoes_whatsapp` do `cessionario_id` do JWT
  - Sub-item: `POST /whatsapp/vincular` — `@UseGuards(JwtAuthGuard, CessionarioOwnerGuard)`; DTO `VincularWhatsappDto: { numero_telefone: string }` (formato E.164); chama `WhatsappService.iniciarVinculacao()`
  - Sub-item: `POST /whatsapp/verificar-otp` — `@UseGuards(JwtAuthGuard, CessionarioOwnerGuard)`; DTO `VerificarOtpDto: { codigo: string }` (6 dígitos numéricos exatos); chama `WhatsappService.verificarOtp()`
  - Sub-item: `DELETE /whatsapp/vincular` — `@UseGuards(JwtAuthGuard, CessionarioOwnerGuard)`; chama `WhatsappService.desvincular(cessionario_id, 'PLATAFORMA')`; retorna `{ estado: 'DESVINCULADO', updated_at }`
  - Sub-item: `POST /whatsapp/webhook` — `@UseGuards(WebhookApiKeyGuard)`; DTO `EvolutionApiWebhookDto` validado; processa payload; responde `200 { data: { received: true } }` imediatamente; processamento assíncrono

- [x] **[BACK-WA-007]** Implementar `WhatsappService.processarWebhook(payload: EvolutionApiWebhookDto)`:
  - Sub-item: Extrai `remoteJid` → `phone_number`; hash = `sha256(phoneE164)` para lookup em `dani_vinculacoes_whatsapp` (REQ-096)
  - Sub-item: Se `phone_hash` não encontrado em `dani_vinculacoes_whatsapp` → 404 `WA_CESSIONARIO_NOT_FOUND`
  - Sub-item: Se mensagem = `"PARAR"` (case-insensitive): chama `desvincular(cessionario_id, 'LGPD_PARAR')` IMEDIATAMENTE; responde confirmação de opt-out ao WhatsApp; NÃO enfileira este processamento na fila (LGPD exige < 1s)
  - Sub-item: Para demais mensagens: encontra `cessionario_id` pelo `phone_hash`; encaminha para `AgenteService.chat()` com contexto `canal: 'WHATSAPP'`; resposta publicada em `dani.whatsapp` para envio via EvolutionAPI
  - Sub-item: Timeout EvolutionAPI: 10s; retry 3x com backoff `5s → 15s → 30s`; após 3 falhas → DLQ `dani.whatsapp.dlq`

- [x] **[BACK-WA-008]** Implementar consumer `dani.whatsapp` para envio via EvolutionAPI:
  - Sub-item: `@RabbitSubscribe({ queue: 'dani.whatsapp', exchange: 'dani.whatsapp' })` — subscribe com ack manual
  - Sub-item: Chama `POST {EVOLUTIONAPI_URL}/message/sendText` com `{ number: phone_e164, text: message }` e header `apikey: {EVOLUTIONAPI_KEY}`
  - Sub-item: Timeout 10s; retry 3x (5s/15s/30s); após 3 falhas → DLQ `dani.whatsapp.dlq`
  - Sub-item: Log de envio: `{ event: 'whatsapp_sent', phone_hash: sha256(phone), template_id, latency_ms }` — nunca número real

---

### 🖥️ Frontend

- [x] **[FRONT-WA-001]** Implementar `T-DC-012 — Vinculação WhatsApp (Fase 2)` em `src/features/dani/whatsapp/WhatsappVinculacao.tsx`:
  - Sub-item: **Estado Skeleton** — `<Skeleton className="h-20 w-full rounded-lg" />` enquanto `GET /whatsapp/status` carrega
  - Sub-item: **Estado Empty (NAO_VINCULADO)** — `<WhatsappStatusBadge estado="NAO_VINCULADO" />` + campo `<Input type="tel" inputMode="numeric" placeholder="+55 (XX) XXXXX-XXXX" />` + `<Button variant="default" aria-label="Vincular WhatsApp">Vincular</Button>`
  - Sub-item: **Estado Error** — `<Alert variant="destructive">` específico por tipo: número inválido (`WA_NUMERO_INVALIDO`), OTP incorreto (`WA_OTP_INVALIDO`), hard block (`WA_OTP_HARD_BLOCK` com tempo restante `mm:ss`), número já associado (`WA_NUMERO_JA_ASSOCIADO`)
  - Sub-item: **Estado Populated (VINCULADO)** — `<WhatsappStatusBadge estado="VINCULADO" />` + número mascarado `"•••••-XXXX"` (sufixo 4 dígitos) + `<Button variant="destructive" size="sm">Desvincular</Button>`
  - Sub-item: Indicador de progresso por etapas: `<Progress>` em `--primary`; Etapa 1/3, 2/3, 3/3

- [x] **[FRONT-WA-002]** Implementar Etapa 1/3 — Informar número:
  - Sub-item: Validação em tempo real após `onBlur` do Input: regex `^\+55\d{11}$` ou formato brasileiro com máscara `(XX) XXXXX-XXXX`
  - Sub-item: Erro inline abaixo do Input antes de submit (sem aguardar resposta do servidor)
  - Sub-item: Submit → `POST /whatsapp/vincular`; se sucesso → transiciona para Etapa 2/3 com `AnimatePresence` slide horizontal 250ms
  - Sub-item: `WhatsAppPhaseGuard`: se Fase 2 não habilitada → exibe `<Alert variant="default">` banner "Esta funcionalidade estará disponível em breve." (sem o formulário)

- [x] **[FRONT-WA-003]** Implementar Etapa 2/3 — Inserir OTP:
  - Sub-item: `<OTPInput>` — 6 campos `<Input maxLength={1} />` em linha, `role="group"`, `aria-label="Código de verificação — 6 dígitos"`; cada campo `aria-label="Dígito [N] de 6"` (D09 T-DC-012 acessibilidade)
  - Sub-item: Auto-avanço entre campos ao digitar; `Backspace` retrocede para campo anterior
  - Sub-item: Foco automático no primeiro campo ao entrar na Etapa 2/3
  - Sub-item: Timer countdown visível: "Código válido por [mm:ss]" — calculado client-side com base em `expires_at` retornado pelo backend; atualizado a cada 1s
  - Sub-item: Link "Reenviar código" com cooldown 60s: desabilitado durante cooldown com contador "Reenviar em [ss]s" (RN-DC-041)
  - Sub-item: OTP incorreto (`WA_OTP_INVALIDO`): `<Alert variant="destructive">` + campos OTP limpos + shake animation 300ms (somente se `prefers-reduced-motion` inativo)
  - Sub-item: Hard block (`WA_OTP_HARD_BLOCK`): `<Alert variant="destructive">` com texto "Número de tentativas excedido. Aguarde [mm:ss] para tentar novamente." + formulário desabilitado
  - Sub-item: OTP expirado (`WA_OTP_EXPIRADO`): `<Alert variant="destructive">` + link "Reenviar código" ativo imediatamente
  - Sub-item: Submit OTP → `POST /whatsapp/verificar-otp`; se sucesso → transiciona para Etapa 3/3

- [x] **[FRONT-WA-004]** Implementar Etapa 3/3 — Aguardar confirmação WhatsApp:
  - Sub-item: Loader/spinner (única exceção permitida para spinner — estado de aguardo de evento externo WebSocket/webhook, não de carregamento de dados)
  - Sub-item: Instrução: "Confirme a vinculação no WhatsApp que foi enviado para seu número. Aguardando confirmação..."
  - Sub-item: Timeout 5 min client-side: se não confirmado em 5 min → exibe `<Alert variant="default">` "A confirmação expirou. Você pode tentar novamente." + botão "Recomeçar"
  - Sub-item: Confirmação bem-sucedida (webhook recebido, `estado = 'VINCULADO'`): fade-in 300ms do estado "Vinculado" + `<WhatsappStatusBadge estado="VINCULADO" />`; template `TPLF-009` disparado via `NotificacaoService`
  - Sub-item: Polling `GET /whatsapp/status` a cada 5s durante Etapa 3 para detectar confirmação (alternativa ao WebSocket)

- [x] **[FRONT-WA-005]** Implementar `Dialog` de confirmação de desvinculação:
  - Sub-item: `<Dialog aria-modal="true">` com texto normativo do D08: "Tem certeza que deseja desvincular seu WhatsApp? Você não receberá mais notificações por esse canal."
  - Sub-item: Botão "Confirmar" → `DELETE /whatsapp/vincular`; sucesso → estado `NAO_VINCULADO`
  - Sub-item: Botão "Cancelar" → fecha Dialog sem ação
  - Sub-item: Motion: `useReducedMotion()` desabilita animações de transição entre etapas (D09 T-DC-012)

---

### 🔌 Wiring

- [x] **[WIRE-WA-001]** Integrar `WhatsappVinculacao` à rota `/cessionario/perfil/whatsapp`:
  - Sub-item: `WhatsAppPhaseGuard` aplicado na rota via TanStack Router `beforeLoad` (REQ-131)
  - Sub-item: Deep link `?chat=open` preservado após redirect de autenticação (REQ-135)
  - Sub-item: `queryKey: ['whatsapp', 'status', cessionario_id]` — invalidado após `vincular`, `verificar-otp` e `desvincular`

- [x] **[WIRE-WA-002]** Configurar variáveis de ambiente para EvolutionAPI:
  - Sub-item: `EVOLUTIONAPI_URL`, `EVOLUTIONAPI_KEY`, `EVOLUTIONAPI_WEBHOOK_SECRET` presentes em `.env` (já documentados em S1 como parte das 28 vars)
  - Sub-item: `EVOLUTIONAPI_INSTANCE_NAME` configurado — nome da instância EvolutionAPI v2.x (ADR-001)

---

### ✅ Testes

- [x] **[TEST-WA-001]** Testes unitários de `WhatsappService.iniciarVinculacao`:
  - Sub-item: Número válido `+5585999994521` + não vinculado → `amqpMock.publish` chamado com `TPLF-008`; `dani_vinculacoes_whatsapp.estado = 'OTP_SMS_ENVIADO'`; Redis `dani:otp:{hash}` TTL 900s criado
  - Sub-item: Número inválido `85999` → lança `WA_NUMERO_INVALIDO` 400
  - Sub-item: `dani:block:otp:{hash}` existe → lança `WA_OTP_HARD_BLOCK` 429 com `reset_at`
  - Sub-item: `dani:rate:otp:{hash}` >= 3 → lança `WA_OTP_RATE_LIMIT` 429
  - Sub-item: `phone_hash` já existe em outro `cessionario_id` → lança `WA_NUMERO_JA_ASSOCIADO` 409
  - Sub-item: OTP nunca logado em produção (`NODE_ENV !== 'development'`): spy no logger confirma ausência do valor OTP bruto

- [x] **[TEST-WA-002]** Testes unitários de `WhatsappService.verificarOtp`:
  - Sub-item: OTP correto → `dani:otp` deletado; `estado = 'AGUARDANDO_CONFIRMACAO_WA'`; `TPLF-009` publicado
  - Sub-item: OTP incorreto × 4 → `dani:rate:otp` = 4; lança `WA_OTP_INVALIDO` 400 (ainda não bloqueado)
  - Sub-item: OTP incorreto × 5 (consecutivos) → `dani:block:otp` criado TTL 1800s; lança `WA_OTP_HARD_BLOCK` 429
  - Sub-item: OTP correto após hard block → verificar que `dani:block:otp` é verificado ANTES de qualquer comparação
  - Sub-item: OTP expirado (TTL expirado) → lança `WA_OTP_EXPIRADO` 400

- [x] **[TEST-WA-003]** Testes unitários de `WhatsappService.processarWebhook`:
  - Sub-item: Mensagem `"PARAR"` → `desvincular(cessionario_id, 'LGPD_PARAR')` chamado síncronamente; estado = `NAO_VINCULADO`; log `opt_out_lgpd` gerado
  - Sub-item: Mensagem `"parar"` (lowercase) → mesmo comportamento (case-insensitive)
  - Sub-item: Phone hash não encontrado → 404 `WA_CESSIONARIO_NOT_FOUND`
  - Sub-item: Mensagem normal → `AgenteService.chat()` chamado com `canal: 'WHATSAPP'` e `cessionario_id` correto

- [x] **[TEST-WA-004]** Testes de integração dos endpoints de WhatsApp:
  - Sub-item: `POST /whatsapp/vincular` — JWT válido → 200 com `estado: 'OTP_SMS_ENVIADO'`; JWT de outro Cessionário → 403 (CessionarioOwnerGuard)
  - Sub-item: `POST /whatsapp/verificar-otp` — OTP correto → 200 com `estado: 'AGUARDANDO_CONFIRMACAO_WA'`; OTP incorreto → 400 `WA_OTP_INVALIDO`
  - Sub-item: `DELETE /whatsapp/vincular` — JWT válido → 200 com `estado: 'DESVINCULADO'`
  - Sub-item: `POST /whatsapp/webhook` — sem `x-api-key` → 401 `WA_WEBHOOK_INVALID_KEY`; `x-api-key` correto + mensagem `"PARAR"` → desvinculação imediata, 200
  - Sub-item: `POST /whatsapp/webhook` — `x-api-key` correto + mensagem normal → `AgenteService` chamado; 200 `{ received: true }`

- [x] **[TEST-WA-005]** Testes P0 de isolamento e LGPD:
  - Sub-item: Cessionário A envia `PARAR` → somente `dani_vinculacoes_whatsapp` de A é atualizado; B permanece inalterado
  - Sub-item: `desvincular(LGPD_PARAR)` não pode ser enfileirado — deve ser síncrono (verificar ausência de `amqpMock.publish` neste path)
  - Sub-item: `phone_hash` nunca exposto em logs — spy no logger confirma que nenhum log contém o número em texto claro
  - Sub-item: Após desvinculação: `GET /whatsapp/status` retorna `estado: 'NAO_VINCULADO'`; notificações WhatsApp não são mais enviadas para esse `cessionario_id`

---

## 🔍 Auto-verificação S7 (12 checks)

| #   | Check                                                                                                                                                                                                                                          | Status |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | Todos os itens são binariamente verificáveis                                                                                                                                                                                                   | ✅     |
| 2   | Nomes exatos: `WhatsappModule`, `WhatsappService`, `WhatsappController`, `WebhookApiKeyGuard`, `dani_vinculacoes_whatsapp`, `EstadoVinculacaoWhatsapp`, `OTPInput`, `WhatsappStatusBadge`                                                      | ✅     |
| 3   | Valores numéricos: OTP 6 dígitos, TTL OTP 900s (15min), TTL rate limit 3600s (1h), TTL hard block 1800s (30min), TTL cooldown reenvio 60s, timeout confirmação WA 24h (backend) / 5min (client-side), 3 tentativas/hora, 5 falhas = hard block | ✅     |
| 4   | Máquina de estados documentada: `NAO_VINCULADO → OTP_SMS_ENVIADO → AGUARDANDO_CONFIRMACAO_WA → VINCULADO`; retorno a `NAO_VINCULADO` via desvinculação ou expiração                                                                            | ✅     |
| 5   | LGPD: comando `PARAR` = desvinculação IMEDIATA e SÍNCRONA; nunca enfileirada; log `opt_out_lgpd` obrigatório; phone_hash nunca em logs                                                                                                         | ✅     |
| 6   | Redis keys documentados com TTLs exatos: `dani:otp:{hash}`, `dani:rate:otp:{hash}`, `dani:block:otp:{hash}`, `dani:otp:cooldown:{hash}`                                                                                                        | ✅     |
| 7   | `WebhookApiKeyGuard` usa `x-api-key: {EVOLUTIONAPI_WEBHOOK_SECRET}` — nunca Bearer JWT neste endpoint (WIRE-AUTH-001 S2)                                                                                                                       | ✅     |
| 8   | EvolutionAPI: timeout 10s; retry 3x (5s/15s/30s); DLQ `dani.whatsapp.dlq`; endpoint `/message/sendText` (D17)                                                                                                                                  | ✅     |
| 9   | Nenhuma suposição sem base documental                                                                                                                                                                                                          | ✅     |
| 10  | Nenhum item de scaffold — todos com sub-itens verificáveis (R10)                                                                                                                                                                               | ✅     |
| 11  | `CessionarioOwnerGuard` em endpoints Bearer; `WebhookApiKeyGuard` em `/webhook`; ambos não mockáveis em testes de integração (REQ-149)                                                                                                         | ✅     |
| 12  | REQs REQ-036–039, REQ-082–086, REQ-128 têm ≥1 item de checklist                                                                                                                                                                                | ✅     |

---

## REQs cobertos por esta sprint

REQ-036 (RN-DC-040 — vinculação 3 etapas: número, OTP, confirmação WA), REQ-037 (RN-DC-041 — OTP rate limit 3/h + hard block 5 falhas + cooldown 60s), REQ-038 (RN-DC-042 — confirmação via WhatsApp + expiração 24h), REQ-039 (RN-DC-044 — PARAR = desvinculação LGPD imediata), REQ-082 (`GET /whatsapp/status`), REQ-083 (`POST /whatsapp/vincular`), REQ-084 (`POST /whatsapp/verificar-otp`), REQ-085 (`DELETE /whatsapp/vincular`), REQ-086 (`POST /whatsapp/webhook` com `WebhookApiKeyGuard`), REQ-128 (T-DC-012 — tela de vinculação: 4 estados + 3 etapas + `OTPInput` + `WhatsappStatusBadge` + WCAG 2.1 AA)
