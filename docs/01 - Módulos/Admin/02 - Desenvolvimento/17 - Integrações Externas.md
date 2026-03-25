# 17 - Integrações Externas

## Repasse Seguro — Admin

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend Lead, DevOps, QA |
| **Escopo** | Especificação técnica das 4 integrações externas: ZapSign, Celcoin, Meta Cloud API (WhatsApp), Twilio (SMS) |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Dependências** | D01 RN (RN-121 a RN-151) · D02 Stacks · D05 PRD · D14 Especificações Técnicas |

---

> 📌 **TL;DR**
>
> - **4 integrações externas:** ZapSign (assinaturas eletrônicas), Celcoin (Conta Escrow/PIX), Meta Cloud API (WhatsApp), Twilio (SMS).
> - **Comunicação:** todas via REST HTTPS. ZapSign e Celcoin retornam via webhook.
> - **Autenticação de webhook:** HMAC-SHA256 em ZapSign e Celcoin — nunca aceitar payload sem verificação.
> - **Padrão:** enfileirar no RabbitMQ imediatamente ao receber webhook (ack < 200ms), processar assincronamente.
> - **Fallback definido** para cada integração — produto não deve parar por falha externa.
> - **Secrets:** nunca em plain text — variáveis de ambiente via Railway (ver D22).

---

## 1. ZapSign — Assinaturas Eletrônicas

### 1.1 Visão Geral

ZapSign é a plataforma de assinatura eletrônica usada para coletar assinaturas nos documentos do dossiê (Contrato de Cessão, Procuração, Termo de Quitação e Instrumento Particular).

| Atributo | Valor |
|---|---|
| **Tipo de API** | REST HTTPS |
| **Base URL** | `https://api.zapsign.com.br/api/v1` |
| **Autenticação saída** | Header `Authorization: Bearer {ZAPSIGN_API_TOKEN}` |
| **Autenticação webhook** | Header `X-ZapSign-Token` verificado por HMAC-SHA256 |
| **Timeout** | 10s para requests; webhook ack em < 200ms |
| **Secret** | `ZAPSIGN_API_TOKEN`, `ZAPSIGN_WEBHOOK_SECRET` |

### 1.2 Fluxo de Criação de Envelope

```
1. Operador cria envelope via Admin SPA
2. API: POST https://api.zapsign.com.br/api/v1/docs/
   {
     "name": "Contrato de Cessão — {case_id}",
     "url_pdf": "{supabase_storage_url}",
     "signers": [
       { "name": "Nome", "email": "email@...", "cpf": "000.000.000-00" }
     ],
     "due_date": "{expires_at}",
     "auto_reminder": true,
     "reminder_every_n_days": 3
   }
3. ZapSign retorna: { "token": "doc_token", "signers": [...] }
4. API salva: zapsign_envelopes.zapsign_document_token = doc_token
5. ZapSign envia e-mail/link de assinatura para cada signatário
```

### 1.3 Eventos de Webhook ZapSign

| Evento | Descrição | Ação no sistema |
|---|---|---|
| `SIGNER_SIGNED` | Signatário assinou | Atualizar signatário no JSONB `signatories`; verificar se todos assinaram |
| `SIGNER_REFUSED` | Signatário recusou | Atualizar status do envelope para `PARCIALMENTE_ASSINADO` + alertar operador |
| `DOCUMENT_SIGNED` | Todos assinaram | `zapsign_envelopes.status = ASSINADO`; `formalization_criteria.signatures_ok = true` |
| `DOCUMENT_EXPIRED` | Prazo expirado | `status = EXPIRADO`; notificar operador para reenvio |

### 1.4 Verificação de Assinatura do Webhook

```typescript
// zapsign-webhook.handler.ts
import * as crypto from 'crypto';

function verifyZapSignWebhook(payload: string, signature: string, secret: string): boolean {
  const computed = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  return crypto.timingSafeEqual(Buffer.from(computed), Buffer.from(signature));
}

// Controller
@Post('/webhooks/zapsign')
async handleZapSignWebhook(
  @Headers('x-zapsign-token') signature: string,
  @RawBody() rawBody: Buffer,
  @Body() body: ZapSignWebhookDto,
) {
  if (!verifyZapSignWebhook(rawBody.toString(), signature, process.env.ZAPSIGN_WEBHOOK_SECRET)) {
    throw new UnauthorizedException('Invalid webhook signature');
  }
  // Enfileirar imediatamente — não processar de forma síncrona
  await this.rabbitMQ.publish('zapsign', 'webhook.zapsign', body);
  return { received: true }; // ack < 200ms
}
```

### 1.5 Reenvio de Envelope

```typescript
// POST https://api.zapsign.com.br/api/v1/docs/{token}/resend
// Incrementa resend_count no banco
// Rate limit ZapSign: máximo 3 reenvios por envelope
```

### 1.6 Tratamento de Erros ZapSign

| Cenário | Resposta ZapSign | Ação |
|---|---|---|
| CPF inválido | 400 Bad Request | Retornar 422 ao operador com campo inválido |
| Token expirado | 401 Unauthorized | Regenerar token via `ZAPSIGN_API_TOKEN` renovado; alertar DevOps |
| Rate limit | 429 Too Many Requests | Retry exponencial: 30s, 2min, 10min via DLQ |
| Timeout | — | Timeout 10s → retry 2x; marcar envelope como `PENDENTE` + alerta |
| ZapSign indisponível | 5xx | Fallback: operador cria envelope manualmente e upload do PDF assinado diretamente no dossiê |

---

## 2. Celcoin — Conta Escrow e PIX

### 2.1 Visão Geral

Celcoin gerencia a Conta Escrow por caso e executa as transferências PIX de distribuição para Cedente, Cessionário e demais partes.

| Atributo | Valor |
|---|---|
| **Tipo de API** | REST HTTPS |
| **Base URL** | `https://sandbox.openfinance.celcoin.dev` (homolog) / `https://openfinance.celcoin.com.br` (prod) |
| **Autenticação** | OAuth 2.0 Client Credentials — `client_id` + `client_secret` → Bearer token |
| **Token TTL** | 3600s — renovar automaticamente com margem de 60s |
| **Autenticação webhook** | Header `X-Celcoin-Signature` — HMAC-SHA256 |
| **Timeout** | 15s para criação de conta; 10s para consultas |
| **Secrets** | `CELCOIN_CLIENT_ID`, `CELCOIN_CLIENT_SECRET`, `CELCOIN_WEBHOOK_SECRET` |

### 2.2 Autenticação OAuth 2.0

```typescript
// celcoin-auth.service.ts — cache do token no Redis
async getCelcoinToken(): Promise<string> {
  const cached = await redis.get('rs:celcoin:token');
  if (cached) return cached;

  const response = await axios.post(`${CELCOIN_BASE_URL}/v5/token`, {
    client_id: process.env.CELCOIN_CLIENT_ID,
    client_secret: process.env.CELCOIN_CLIENT_SECRET,
    grant_type: 'client_credentials',
  });

  const { access_token, expires_in } = response.data;
  await redis.setex('rs:celcoin:token', expires_in - 60, access_token);
  return access_token;
}
```

### 2.3 Fluxo de Criação de Conta Escrow

```
1. Caso criado → API automaticamente cria Conta Escrow na Celcoin
2. POST {CELCOIN_BASE_URL}/v5/escrow/accounts
   { "external_id": "{case_id}", "holder_name": "Repasse Seguro", ... }
3. Celcoin retorna: { "account_id": "celcoin_account_id" }
4. API salva: escrow_accounts.celcoin_account_id = account_id
5. Status: AGUARDANDO_DEPOSITO
```

### 2.4 Monitoramento de Depósito

Celcoin envia webhook quando depósito é identificado na conta:

```json
// POST /webhooks/celcoin
{
  "event": "DEPOSIT_RECEIVED",
  "account_id": "celcoin_account_id",
  "transaction_id": "celcoin_txn_id",
  "amount": "150000.00",
  "payer": { "name": "...", "document": "..." }
}
```

**Ação no sistema:**
1. Validar HMAC-SHA256.
2. Enfileirar no RabbitMQ `celcoin.webhook`.
3. Consumer: atualizar `escrow_accounts.deposited_amount`, `deposit_confirmed_at`, `status = DEPOSITO_CONFIRMADO`.
4. Registrar em `escrow_transactions` (tipo `DEPOSITO`).
5. Verificar se `deposited_amount >= expected_amount` → atualizar `formalization_criteria.deposit_ok = true`.
6. Notificar operador via WhatsApp.

### 2.5 Fluxo de Distribuição PIX

```typescript
// Para cada recipient em distributions[]
const transfer = await celcoin.post('/v5/pix/transfer', {
  external_id: distribution.id,  // Idempotência
  amount: distribution.amount,
  recipient: {
    key_type: 'CPF',  // ou 'CNPJ', 'EMAIL', 'PHONE', 'EVP'
    key: recipient.cpf,
  },
  description: `Repasse Seguro - Caso ${case_id}`,
});
// Salvar celcoin_transfer_id
```

**Idempotência:** Celcoin usa `external_id` para garantir que o mesmo `distribution.id` não seja processado duas vezes.

### 2.6 Eventos de Webhook Celcoin

| Evento | Ação |
|---|---|
| `DEPOSIT_RECEIVED` | Confirmar depósito, atualizar critério de fechamento |
| `PIX_TRANSFER_COMPLETED` | Atualizar `distributions.status = CONCLUIDO` |
| `PIX_TRANSFER_FAILED` | Atualizar `distributions.status = FALHOU`; retry via DLQ |
| `PIX_TRANSFER_REVERSED` | Registrar estorno em `escrow_transactions` |

### 2.7 Tratamento de Erros Celcoin

| Cenário | Ação |
|---|---|
| OAuth token expirado durante request | Renovar token e repetir imediatamente |
| Saldo insuficiente na conta escrow | Retornar 422 ao operador: "Saldo insuficiente para distribuição" |
| PIX inválido (chave não encontrada) | Atualizar `distributions.status = FALHOU`; notificar Gestor Financeiro |
| Celcoin indisponível (5xx) | Retry exponencial via DLQ (5x); alertar DevOps após 3 falhas |
| Timeout | 15s timeout → enfileirar retry; não processar duplicata |

---

## 3. Meta Cloud API — WhatsApp Business

### 3.1 Visão Geral

WhatsApp Business é o canal principal de notificação para Cedentes e Cessionários.

| Atributo | Valor |
|---|---|
| **Tipo de API** | REST HTTPS (Graph API) |
| **Base URL** | `https://graph.facebook.com/v19.0/{PHONE_NUMBER_ID}/messages` |
| **Autenticação** | Header `Authorization: Bearer {META_API_TOKEN}` |
| **Tipo de mensagem** | Template Messages (pré-aprovados pelo WhatsApp) |
| **Secrets** | `META_API_TOKEN`, `META_PHONE_NUMBER_ID`, `META_WABA_ID` |

### 3.2 Templates Necessários e Nomes

Todos os templates devem ser criados e aprovados no WhatsApp Business Manager antes do go-live.

| Template Name | Categoria | Trigger |
|---|---|---|
| `rs_case_status_update` | UTILITY | Mudança de status do caso |
| `rs_document_rejected` | UTILITY | Documento rejeitado — solicitar reenvio |
| `rs_document_approved` | UTILITY | Documento aprovado |
| `rs_proposal_received` | UTILITY | Nova proposta enviada |
| `rs_proposal_accepted` | UTILITY | Proposta aceita |
| `rs_signature_pending` | UTILITY | Documento para assinar (deep link ZapSign) |
| `rs_closing_completed` | UTILITY | Fechamento concluído |
| `rs_deposit_confirmed` | UTILITY | Depósito na conta escrow confirmado |
| `rs_verification_code` | AUTHENTICATION | Código OTP para autenticação mobile |

### 3.3 Estrutura de Requisição

```typescript
// notifications.service.ts
async sendWhatsApp(phone: string, template: string, params: string[]) {
  return axios.post(
    `https://graph.facebook.com/v19.0/${process.env.META_PHONE_NUMBER_ID}/messages`,
    {
      messaging_product: 'whatsapp',
      to: phone.replace(/\D/g, ''),  // Apenas dígitos, com código do país: 55119...
      type: 'template',
      template: {
        name: template,
        language: { code: 'pt_BR' },
        components: [{
          type: 'body',
          parameters: params.map(p => ({ type: 'text', text: p })),
        }],
      },
    },
    { headers: { Authorization: `Bearer ${process.env.META_API_TOKEN}` } }
  );
}
```

### 3.4 Tratamento de Erros WhatsApp

| Cenário | Código Meta | Ação |
|---|---|---|
| Número inválido / não no WhatsApp | 131026 | Fallback automático para SMS via Twilio |
| Template não aprovado | 132001 | Bloquear envio; alertar DevOps para aprovação |
| Rate limit (1000/dia por template) | 130429 | Queued — enviar no próximo janela; log + alerta |
| Token expirado | 190 | Renovar token; alertar DevOps (token tem validade longa) |
| Optin não dado pelo usuário | 131009 | Log + não retransmitir; notificar operador para verificar optin |

---

## 4. Twilio — SMS Fallback

### 4.1 Visão Geral

Twilio é usado como canal de SMS para: (1) OTP de verificação no app mobile, (2) fallback quando WhatsApp não está disponível.

| Atributo | Valor |
|---|---|
| **Tipo de API** | REST HTTPS |
| **Base URL** | `https://api.twilio.com/2010-04-01/Accounts/{TWILIO_ACCOUNT_SID}/Messages.json` |
| **Autenticação** | HTTP Basic Auth: `TWILIO_ACCOUNT_SID:TWILIO_AUTH_TOKEN` |
| **Número de origem** | `TWILIO_PHONE_NUMBER` (número comprado na Twilio) |
| **Secrets** | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER` |

### 4.2 Casos de Uso

| Uso | Template SMS |
|---|---|
| OTP mobile | `Seu código de verificação Repasse Seguro: {code}. Válido por 5 minutos.` |
| Fallback WhatsApp | Versão resumida do template correspondente |

### 4.3 Estrutura de Requisição

```typescript
async sendSms(to: string, body: string) {
  const sid = process.env.TWILIO_ACCOUNT_SID;
  return axios.post(
    `https://api.twilio.com/2010-04-01/Accounts/${sid}/Messages.json`,
    new URLSearchParams({
      From: process.env.TWILIO_PHONE_NUMBER,
      To: `+55${to.replace(/\D/g, '')}`,
      Body: body,
    }),
    { auth: { username: sid, password: process.env.TWILIO_AUTH_TOKEN } }
  );
}
```

### 4.4 Tratamento de Erros Twilio

| Cenário | Código Twilio | Ação |
|---|---|---|
| Número inválido | 21211 | Log + não retransmitir |
| Número bloqueado para SMS | 21610 | Log + alertar operador |
| Saldo insuficiente | 20003 | Alerta crítico DevOps; fallback para e-mail |
| Rate limit | 429 | Retry exponencial 30s |

---

## 5. Estratégia de Fallback entre Canais

```
Para notificações de eventos de caso:
1. Tenta WhatsApp (Meta Cloud API)
   → Sucesso: registra notification_logs.channel = WHATSAPP, status = ENVIADO
   → Falha (número inválido ou sem WhatsApp): vai para passo 2

2. Tenta SMS (Twilio)
   → Sucesso: registra notification_logs.channel = SMS, status = ENVIADO
   → Falha: registra status = FALHOU, error_message

Para OTP mobile:
1. SMS via Twilio (sempre — sem fallback para WhatsApp em autenticação)
```

---

## 6. Monitoramento de Integrações

| Integração | Health Check | Alerta |
|---|---|---|
| ZapSign | GET `/api/v1/me` a cada 5min | Pino WARN + Slack se 3 falhas consecutivas |
| Celcoin | GET `/v5/status` a cada 5min | Idem |
| Meta (WhatsApp) | GET `/debug_token` a cada 10min | Pino WARN se token < 7 dias para expirar |
| Twilio | GET `/Accounts/{SID}` a cada 10min | Pino WARN se saldo < $10 |

---

## 7. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — ZapSign (assinaturas), Celcoin (escrow/PIX), Meta Cloud (WhatsApp), Twilio (SMS), fluxos completos, tratamento de erros e fallback entre canais. |
