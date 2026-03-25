# 17 - Integrações Externas

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend Lead, Engenharia, QA |
| **Escopo** | 5 integrações externas do CRM — fluxos, endpoints, retry policy e comportamento em falha |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | Doc 01.5 RN-183 a RN-196 · Doc 02 Stacks CRM · Doc 16 API CRM |

---

> **TL;DR**
>
> - **5 integrações:** Plataforma RS (sync bidirecional de Casos ≤2min) · WhatsApp Business API (templates HSM, janela 24h, opt-out LGPD) · ZapSign (assinatura eletrônica, régua D+0/D+2/D+4/D+5) · Escrow Celcoin (webhook de depósito, fallback 24h) · Supabase Auth (JWT, RLS, validação de role).
> - **Protocolo padrão de falha (RN-196):** retry 2min → retry 10min → alerta Admin RS → alerta crítico após 1h.
> - **Dead-letter queue** em toda fila RabbitMQ — nenhuma falha de integração perde evento.
> - **HMAC-SHA256** para validação de webhooks (ZapSign, Celcoin, Plataforma RS).

---

## 1. Visão Geral das Integrações

| # | Integração | Tipo | Direção | Protocolo | Criticidade |
|---|---|---|---|---|---|
| 1 | Plataforma Repasse Seguro | Sistema interno | Bidirecional | Webhook + API REST | Crítica — P0 |
| 2 | WhatsApp Business API (Meta) | SaaS externo | Bidirecional | REST + Webhook | Alta — P1 |
| 3 | ZapSign | SaaS externo | Saída + Webhook retorno | REST + Webhook | Alta — P1 |
| 4 | Escrow (Celcoin) | Parceiro financeiro | Entrada (webhook) | Webhook HMAC | Crítica — P0 |
| 5 | Supabase Auth | Plataforma interna | Bidirecional | JWT + RLS | Crítica — P0 |

---

## 2. Protocolo Padrão de Falha (RN-196)

Aplicado a todas as integrações externas (exceto Supabase Auth, que tem tratamento próprio):

```
Falha detectada
  └─ Tentativa 1 (imediata) — falhou
       └─ Retry após 2 minutos
            └─ Tentativa 2 — falhou
                 └─ Retry após 10 minutos
                      └─ Tentativa 3 — falhou
                           └─ Alerta ao Admin RS + notificação ao Analista RS do Caso afetado
                                └─ Se 1 hora de falha contínua:
                                     └─ Alerta CRÍTICO ao Admin RS com recomendação de ação manual
```

**Regra invariante:** Nenhuma falha de integração cancela ou retrocede automaticamente um Caso. Eventos ficam em dead-letter queue aguardando resolução manual.

---

## 3. Integração 1 — Plataforma Repasse Seguro

### 3.1 Descrição

Sincronização bidirecional de Casos, Dossiê e interesses de Cessionários entre a plataforma pública (app/web do Cedente) e o CRM (ferramenta interna da equipe RS). Latência máxima: 2 minutos (RN-183, RN-186).

### 3.2 Fluxos

**Fluxo A — Plataforma → CRM (novo Caso)**

```mermaid
sequenceDiagram
    participant P as Plataforma RS
    participant W as CRM Webhook
    participant Q as RabbitMQ
    participant C as CRM Cases Service
    participant DB as Supabase DB
    participant N as Notifications Service

    P->>W: POST /api/v1/crm/webhooks/platform<br/>{event: "CASE_CREATED", payload: {...}}
    W->>W: Validar HMAC-SHA256 (X-RS-Platform-Signature)
    alt Assinatura inválida
        W-->>P: 401 Unauthorized
    else Assinatura válida
        W->>Q: Enfileirar evento (fila: crm.platform.incoming)
        W-->>P: 200 OK
    end

    Q->>C: Consumir evento CASE_CREATED
    C->>DB: INSERT case (status=CADASTRO, origin=PLATAFORMA)
    C->>DB: Buscar Analista de plantão ou vinculado ao Parceiro
    alt Analista disponível
        C->>DB: UPDATE case SET assigned_analyst_id
        C->>N: Enfileirar NOT-CRM-01 (novo Caso atribuído)
    else Sem Analista disponível
        C->>DB: case.assigned_analyst_id = NULL
        C->>N: Alerta ao Coordenador RS para atribuição manual
    end
```

**Fluxo B — Plataforma → CRM (documento sincronizado)**

```mermaid
sequenceDiagram
    participant P as Plataforma RS
    participant W as CRM Webhook
    participant Q as RabbitMQ
    participant D as CRM Dossier Service
    participant S as Supabase Storage

    P->>W: POST /api/v1/crm/webhooks/platform<br/>{event: "DOCUMENT_SYNCED", case_identifier, document_url, type}
    W->>W: Validar HMAC-SHA256
    W->>Q: Enfileirar evento (fila: crm.platform.incoming)
    W-->>P: 200 OK

    Q->>D: Consumir DOCUMENT_SYNCED
    D->>S: Download do documento via URL segura
    D->>S: Re-upload para bucket do Dossiê do CRM
    D->>D: DB INSERT dossier_documents (origin=CEDENTE_VIA_PLATAFORMA, status=PENDENTE)
```

**Fluxo C — CRM → Plataforma (atualização de estado)**

```mermaid
sequenceDiagram
    participant A as Analista RS (CRM)
    participant CS as CRM Cases Service
    participant PS as Platform Sync Service
    participant P as Plataforma RS API
    participant Q as RabbitMQ (DLQ)

    A->>CS: PATCH /cases/:id/transition {to_status: "PUBLICACAO"}
    CS->>CS: Validar e persistir nova transição
    CS->>PS: Enfileirar evento de sync (fila: crm.platform.outgoing)

    PS->>P: POST /platform/api/cases/sync<br/>{case_identifier, new_status, updated_at}
    alt Plataforma disponível
        P-->>PS: 200 OK
    else Plataforma indisponível
        PS->>PS: Retry (protocolo padrão RN-196)
        PS-->>Q: Dead-letter após 3 falhas consecutivas
        PS->>PS: Admin RS notificado
    end
```

### 3.3 Configuração Técnica

| Parâmetro | Valor |
|---|---|
| Autenticação (entrada) | HMAC-SHA256 — Header `X-RS-Platform-Signature` |
| Autenticação (saída) | Bearer token de serviço (`PLATFORM_SERVICE_TOKEN`) |
| Fila de entrada | `crm.platform.incoming` (RabbitMQ) |
| Fila de saída | `crm.platform.outgoing` (RabbitMQ) |
| Dead-letter queue | `crm.platform.dlq` |
| Latência máxima | 2 minutos (RN-183) |
| Retry policy | 3 tentativas (2min → 10min → DLQ) |

### 3.4 Comportamento em Falha

| Cenário | Comportamento |
|---|---|
| Plataforma indisponível (saída) | Retry policy padrão. Estado no CRM permanece correto. Plataforma atualizada na próxima tentativa bem-sucedida. |
| HMAC inválido (entrada) | 401. Evento descartado. Log de segurança registrado. |
| Caso duplicado | Idempotência por `case_identifier`. Segundo evento ignorado. |
| Analista de plantão indisponível | Caso criado como não atribuído. Alerta ao Coordenador RS. |
| Documento sync falha após 1h | Admin RS notificado. Analista RS pode fazer upload manual. |

---

## 4. Integração 2 — WhatsApp Business API (Meta Cloud API)

### 4.1 Descrição

Canal principal de comunicação do CRM com Cedentes e Cessionários. Mensagens são enviadas pelo Analista RS via CRM, que chama a Meta Cloud API. Respostas chegam ao CRM via webhook da Meta.

**Regras críticas:**
- Janela de 24h: mensagem de texto livre só é permitida se o Contato enviou mensagem nas últimas 24h (RN-187).
- Fora da janela: apenas templates HSM pré-aprovados pela Meta (RN-187).
- Opt-out: contatos com opt-out registrado não recebem mensagens (RN-188).
- Opt-in para marketing: obrigatório antes de enviar template de categoria MARKETING (RN-188).

### 4.2 Fluxos

**Fluxo A — CRM envia mensagem ao Contato**

```mermaid
sequenceDiagram
    participant A as Analista RS
    participant CS as Communication Service
    participant WM as Window Monitor Service
    participant META as Meta Cloud API
    participant Q as RabbitMQ

    A->>CS: POST /cases/:id/messages {contact_id, content}
    CS->>WM: Verificar janela 24h do Contato
    alt Fora da janela
        CS-->>A: 422 CRM-051 (use template)
    else Opt-out registrado
        CS-->>A: 422 CRM-052
    else Janela ativa
        CS->>META: POST /messages {to, type: "text", text: {body}}
        alt Entrega bem-sucedida
            META-->>CS: 200 {messages: [{id: "wamid..."}]}
            CS->>CS: Registrar mensagem (status=SENT)
            CS-->>A: 201 OK
        else Falha de entrega
            META-->>CS: Erro (número inválido, bloqueio, timeout)
            CS->>CS: Registrar falha (status=FAILED, error_detail)
            CS->>Q: Enfileirar alerta ao Analista RS
            CS-->>A: 201 OK (com status=FAILED — notificação enviada)
        end
    end
```

**Fluxo B — Contato responde (webhook Meta → CRM)**

```mermaid
sequenceDiagram
    participant META as Meta Cloud API
    participant W as CRM Webhook
    participant Q as RabbitMQ
    participant CS as Communication Service
    participant N as Notifications Service

    META->>W: POST /api/v1/crm/webhooks/whatsapp<br/>{object, entry: [{changes: [{value: {messages}}]}]}
    W->>W: Verificar X-Hub-Signature-256 (HMAC-SHA256)
    W->>Q: Enfileirar mensagem recebida
    W-->>META: 200 OK (< 20s, obrigatório pela Meta)

    Q->>CS: Consumir mensagem recebida
    CS->>CS: Identificar Contato pelo número de telefone
    CS->>CS: Vincular ao Caso ativo do Contato
    CS->>CS: Atualizar janela de 24h (timestamp)
    CS->>N: Notificar Analista RS responsável
```

### 4.3 Configuração Técnica

| Parâmetro | Valor |
|---|---|
| Base URL | `https://graph.facebook.com/v21.0/{phone_number_id}/messages` |
| Autenticação saída | Bearer `META_WHATSAPP_TOKEN` |
| Autenticação entrada webhook | `X-Hub-Signature-256` (HMAC-SHA256) |
| Verify token (setup) | `META_WEBHOOK_VERIFY_TOKEN` |
| Fila de mensagens recebidas | `crm.whatsapp.incoming` |
| Fila de mensagens enviadas | `crm.whatsapp.outgoing` |
| Fila de status updates | `crm.whatsapp.status` |
| Timeout de resposta ao webhook | 20 segundos (SLA obrigatório da Meta) |

### 4.4 Comportamento em Falha

| Cenário | Comportamento |
|---|---|
| API Meta indisponível (envio) | Mensagem marcada como FAILED. Analista RS notificado para contato alternativo (RN-187). |
| Número inválido | Erro registrado na timeline do Caso. Analista RS notificado. |
| Número bloqueou a conta | Erro `131026` registrado. Analista RS notificado. Flag `whatsapp_blocked=true` no Contato. |
| Webhook Meta não respondido em 20s | Meta retentativa automática — sistema deve ser idempotente por `wamid`. |
| Template rejeitado pela Meta | 422 CRM-053. Analista RS orientado a usar texto livre (se dentro da janela). |

---

## 5. Integração 3 — ZapSign (Assinatura Eletrônica)

### 5.1 Descrição

Plataforma de assinatura eletrônica usada para contratos de cessão imobiliária. O CRM cria envelopes ZapSign com os signatários (Cedente, Cessionário, representante RS), monitora o status via webhook e executa régua de lembretes (RN-189).

### 5.2 Régua de Lembretes

| Dia | Ação |
|---|---|
| D+0 | Criação do envelope — link de assinatura enviado por e-mail a cada signatário |
| D+2 | 1º lembrete automático — e-mail enviado pelo ZapSign |
| D+4 | 2º lembrete urgente — e-mail + alerta ao Analista RS no CRM |
| D+5 | Contrato expirado — Admin RS notificado para decisão de reemissão |

### 5.3 Fluxos

**Fluxo A — Criar envelope e enviar contrato**

```mermaid
sequenceDiagram
    participant A as Analista RS
    participant DS as CRM Dossier Service
    participant ZS as ZapSign API
    participant Q as RabbitMQ

    A->>DS: POST /cases/:id/dossier/zapsign<br/>{document_type, signatories}
    DS->>ZS: POST /api/v1/docs/<br/>{name, signers: [{name, email, cpf}], send_automatic_email: true}
    alt ZapSign disponível
        ZS-->>DS: 200 {doc: {token, status: "pending", signers: [...]}}
        DS->>DS: Salvar envelope_token no Caso
        DS->>DS: Registrar estado: PENDENTE_ASSINATURA
        DS-->>A: 201 OK {envelope_id, status: "PENDENTE"}
    else ZapSign indisponível
        ZS-->>DS: Timeout / 5xx
        DS->>Q: Enfileirar retry (protocolo RN-196)
        DS-->>A: 202 Accepted {message: "Envio em processamento"}
    end
```

**Fluxo B — Webhook ZapSign → CRM (atualização de assinatura)**

```mermaid
sequenceDiagram
    participant ZS as ZapSign
    participant W as CRM Webhook
    participant Q as RabbitMQ
    participant DS as CRM Dossier Service
    participant CS as Cases Service
    participant N as Notifications Service

    ZS->>W: POST /api/v1/crm/webhooks/zapsign<br/>{event: "SIGNED | REFUSED | EXPIRED", token, signer}
    W->>W: Verificar X-ZapSign-Token
    W->>Q: Enfileirar evento (fila: crm.zapsign.events)
    W-->>ZS: 200 OK

    Q->>DS: Consumir evento
    alt event = SIGNED (signatário assinou)
        DS->>DS: Atualizar status do signatário
        DS->>DS: Verificar se TODOS assinaram
        alt Todos assinaram
            DS->>DS: Download PDF assinado do ZapSign
            DS->>DS: Upload PDF para Dossiê (document_type=CONTRATO_ASSINADO)
            DS->>N: Enfileirar NOT-CRM-06 (próximo passo)
        end
    else event = REFUSED
        DS->>DS: Marcar envelope como RECUSADO
        DS->>N: Alerta urgente ao Analista RS
    else event = EXPIRED (D+5)
        DS->>DS: Marcar envelope como EXPIRADO
        DS->>N: Alerta ao Admin RS para decisão
    end
```

**Fluxo C — Retry manual (RN-190)**

```mermaid
sequenceDiagram
    participant A as Analista RS
    participant DS as CRM Dossier Service
    participant ZS as ZapSign API

    A->>DS: POST /cases/:id/dossier/zapsign/:envelope_id/resend
    DS->>DS: Verificar cooldown (5 min desde última tentativa)
    DS->>DS: Verificar contador (máx 3 tentativas consecutivas)
    alt Cooldown ativo
        DS-->>A: 429 {retry_after: "ISO8601"}
    else Limite de tentativas atingido
        DS->>DS: Notificar Admin RS para investigação
        DS-->>A: 422 CRM-111 {detail: "Limite de tentativas atingido. Admin RS notificado."}
    else Permitido
        DS->>ZS: POST /api/v1/docs/{token}/resend-email
        ZS-->>DS: 200 OK
        DS-->>A: 200 OK
    end
```

### 5.4 Configuração Técnica

| Parâmetro | Valor |
|---|---|
| Base URL | `https://api.zapsign.com.br/api/v1` |
| Autenticação saída | Bearer `ZAPSIGN_API_TOKEN` |
| Autenticação entrada webhook | Header `X-ZapSign-Token` comparado com `ZAPSIGN_WEBHOOK_SECRET` |
| Fila de eventos | `crm.zapsign.events` |
| Dead-letter queue | `crm.zapsign.dlq` |
| Tentativas de criação | máx 3, cooldown 5min (RN-190) |
| Validade do envelope | D+5 corridos |

### 5.5 Comportamento em Falha

| Cenário | Comportamento |
|---|---|
| ZapSign indisponível na criação | Retry policy padrão. Caso permanece em FORMALIZACAO. Analista RS notificado. |
| 3 falhas consecutivas de criação | Admin RS notificado para investigação (RN-190). |
| Webhook não entregue | ZapSign tem retry próprio. CRM processa com idempotência por `token`. |
| PDF assinado indisponível para download | Retry 3x. Se falhar, Analista RS pode fazer upload manual com aprovação de Coordenador RS. |

---

## 6. Integração 4 — Escrow (Celcoin)

### 6.1 Descrição

Confirmação de depósito na Conta Escrow como critério de Fechamento do Caso. O CRM recebe confirmação via webhook HMAC-SHA256. Fallback manual com aprovação do Coordenador RS (RN-191, RN-192).

### 6.2 Fluxos

**Fluxo A — Webhook de confirmação de depósito**

```mermaid
sequenceDiagram
    participant CEL as Celcoin
    participant W as CRM Webhook
    participant Q as RabbitMQ
    participant ES as CRM Escrow Handler
    participant DS as Dossier Service
    participant N as Notifications Service

    CEL->>W: POST /api/v1/crm/webhooks/escrow<br/>{transaction_id, case_identifier, status: "PAID", amount, confirmed_at}
    W->>W: Verificar X-Celcoin-Signature (HMAC-SHA256)
    alt Assinatura inválida
        W-->>CEL: 401 Unauthorized
    else Válida
        W->>Q: Enfileirar evento (fila: crm.escrow.events)
        W-->>CEL: 200 OK
    end

    Q->>ES: Consumir evento PAID
    ES->>ES: Localizar Caso por case_identifier
    ES->>DS: Registrar comprovante de depósito (origin=WEBHOOK, status=APROVADO)
    ES->>N: Notificar Analista RS: "Depósito confirmado no Caso RS-XXXX"
    ES->>N: NOT-CRM-05 (dossiê aprovado — próximo passo: Fechamento)
```

**Fluxo B — Fallback: depósito sem webhook**

```mermaid
sequenceDiagram
    participant SLA as SLA Monitor Job
    participant N as Notifications Service
    participant A as Analista RS
    participant DS as Dossier Service
    participant COORD as Coordenador RS

    note over SLA: Cron job verifica Casos em FORMALIZACAO<br/>com prazo esperado de depósito ultrapassado em 24h (RN-191)

    SLA->>N: Alerta ao Analista RS e Coordenador RS
    N-->>A: "Depósito esperado não confirmado no Caso RS-XXXX. Verifique manualmente."

    A->>DS: POST /cases/:id/dossier (upload manual do comprovante)<br/>{document_type: "COMPROVANTE_ESCROW", file, insertion_type: "MANUAL"}
    DS->>COORD: Solicitação de aprovação (flag: "Inserção manual")
    COORD->>DS: PATCH /cases/:id/dossier/:doc_id/approve
    DS->>DS: Registrar status=APROVADO
```

### 6.3 Configuração Técnica

| Parâmetro | Valor |
|---|---|
| Autenticação entrada webhook | Header `X-Celcoin-Signature` (HMAC-SHA256) |
| Chave HMAC | `CELCOIN_WEBHOOK_SECRET` |
| Fila de eventos | `crm.escrow.events` |
| Dead-letter queue | `crm.escrow.dlq` |
| Janela de fallback | 24h após prazo esperado (RN-191) |
| Upload manual | Requer aprovação de Coordenador RS (RN-192) |

### 6.4 Comportamento em Falha

| Cenário | Comportamento |
|---|---|
| Webhook não chega em 24h | Alerta ao Analista RS e Coordenador RS. Upload manual habilitado. |
| Webhook status=FAILED | Analista RS notificado. Verificação manual junto ao parceiro financeiro. |
| Webhook status=REVERSED | Alerta crítico ao Coordenador RS. Caso não pode avançar para Fechamento. |
| Comprovante manual sem aprovação | Documento marcado como PENDENTE. Não conta como critério de Fechamento. |

---

## 7. Integração 5 — Supabase Auth

### 7.1 Descrição

Toda autenticação e autorização do CRM é gerenciada pelo Supabase Auth. JWT com claims de papel (`role`) validados em cada requisição à API NestJS. RLS (Row-Level Security) no banco garante isolamento de dados por papel e por Analista (RN-193).

### 7.2 Fluxos

**Fluxo A — Login e emissão de JWT**

```mermaid
sequenceDiagram
    participant C as CRM Frontend
    participant SA as Supabase Auth
    participant API as CRM API (NestJS)
    participant DB as Supabase DB (RLS)

    C->>SA: signInWithPassword({email, password})
    SA->>SA: Validar credenciais
    SA->>SA: Verificar tentativas (contador RN-002)
    alt Credenciais inválidas
        SA->>SA: Incrementar failed_attempts
        alt failed_attempts >= 5
            SA->>SA: locked_until = NOW() + 15min
        end
        SA-->>C: AuthError (invalid_credentials)
    else Conta inativa
        SA-->>C: AuthError (account_disabled)
    else Login bem-sucedido
        SA->>SA: Buscar role em user_metadata
        SA->>SA: Emitir JWT com claims {sub, email, role, aud, exp}
        SA-->>C: {access_token, refresh_token, user}
        C->>API: GET /api/v1/crm/cases (Authorization: Bearer access_token)
        API->>API: JwtStrategy.validate() — verificar claims
        API->>DB: Query com RLS aplicado (ANALISTA_RS vê apenas seus Casos)
        DB-->>API: Dados filtrados por RLS
        API-->>C: 200 { data: [...] }
    end
```

**Fluxo B — Renovação automática de token**

```mermaid
sequenceDiagram
    participant C as CRM Frontend (@supabase/ssr)
    participant SA as Supabase Auth

    note over C: Token expira em 60min (RN-003)
    C->>SA: refreshSession({refresh_token})
    SA->>SA: Validar refresh_token
    alt Refresh válido
        SA-->>C: {access_token (novo), refresh_token (novo)}
        note over C: Renovação transparente — usuário não percebe
    else Refresh expirado ou inválido
        SA-->>C: AuthError
        note over C: Redirecionar para /login
    end
```

### 7.3 Estrutura do JWT

```json
{
  "sub": "user-uuid-supabase",
  "email": "analista@repasseseguro.com.br",
  "role": "authenticated",
  "user_metadata": {
    "crm_role": "ANALISTA_RS",
    "name": "Ana Lima"
  },
  "app_metadata": {
    "provider": "email"
  },
  "aud": "authenticated",
  "iat": 1711094400,
  "exp": 1711097600
}
```

> **Nota:** O campo `crm_role` em `user_metadata` é o claim de papel do CRM. Definido pelo Admin RS ao criar o usuário via `supabase.auth.admin.updateUserById()`. Validado em cada requisição pelo `CrmJwtStrategy`.

### 7.4 Políticas RLS Relevantes

```sql
-- Analistas RS só acessam Casos atribuídos a eles
CREATE POLICY "analistas_see_own_cases" ON cases
  FOR SELECT
  USING (
    auth.uid() = assigned_analyst_id
    OR (auth.jwt()->>'crm_role') IN ('COORDENADOR_RS', 'ADMIN_RS')
  );

-- Parceiros Externos só acessam Casos vinculados ao seu Parceiro
CREATE POLICY "parceiros_see_partner_cases" ON cases
  FOR SELECT
  USING (
    (auth.jwt()->>'crm_role') = 'PARCEIRO_EXTERNO'
    AND partner_id = (
      SELECT partner_id FROM users WHERE id = auth.uid()
    )
  );

-- Log de auditoria: append-only para todos
CREATE POLICY "audit_log_insert_only" ON audit_log
  FOR INSERT WITH CHECK (true);

-- Nenhum UPDATE ou DELETE no audit_log
CREATE POLICY "audit_log_no_update" ON audit_log
  FOR UPDATE USING (false);

CREATE POLICY "audit_log_no_delete" ON audit_log
  FOR DELETE USING (false);
```

### 7.5 Configuração Técnica

| Parâmetro | Valor |
|---|---|
| Provedor | Supabase Auth (`@supabase/ssr` no frontend, validação JWT no backend) |
| Algoritmo JWT | HS256 |
| Expiração do access token | 60 minutos (RN-003) |
| Renovação | Automática via `@supabase/ssr` antes do vencimento |
| Bloqueio por tentativas | 5 tentativas → bloqueio por 15 min (RN-002) |
| Sessão de inatividade | 60 min (RN-003) |
| RLS | Habilitado em todas as tabelas de domínio |

### 7.6 Comportamento em Falha

| Cenário | Comportamento |
|---|---|
| Token expirado durante sessão | `@supabase/ssr` renova automaticamente. Se refresh falhar, redireciona para login. |
| Role alterado enquanto usuário está logado | Próxima validação de JWT (máx 60min) detecta role atualizado. Para aplicar imediatamente: revogar sessão via Admin API. |
| Supabase Auth indisponível | API retorna 503. Frontend exibe mensagem de erro e tenta reconnect. |
| RLS bloqueando query esperada | Erro 403 na API com `code: "CRM-023"`. Log de segurança registrado. |

---

## 8. Resumo de Filas RabbitMQ

| Fila | Tipo | Produtor | Consumidor | DLQ |
|---|---|---|---|---|
| `crm.platform.incoming` | Eventos da Plataforma RS | WebhookController | PlatformWebhookHandler | `crm.platform.dlq` |
| `crm.platform.outgoing` | Sync de estado para Plataforma RS | CasesService | PlatformSyncService | `crm.platform.dlq` |
| `crm.whatsapp.incoming` | Mensagens recebidas | WebhookController | CommunicationService | `crm.whatsapp.dlq` |
| `crm.whatsapp.outgoing` | Mensagens a enviar | CommunicationService | WhatsAppClient | `crm.whatsapp.dlq` |
| `crm.whatsapp.status` | Status de entrega | WebhookController | CommunicationService | `crm.whatsapp.dlq` |
| `crm.zapsign.events` | Eventos de assinatura | WebhookController | ZapsignWebhookHandler | `crm.zapsign.dlq` |
| `crm.escrow.events` | Confirmação de depósito | WebhookController | EscrowWebhookHandler | `crm.escrow.dlq` |
| `crm.notifications` | Todas as notificações do CRM | Vários services | NotificationsConsumer | `crm.notifications.dlq` |

**Configuração padrão de todas as filas:**
- `durable: true` — sobrevivem a restart do broker
- `x-dead-letter-exchange: crm.dlx` — DLX compartilhado
- `x-message-ttl: 86400000` — TTL de 24h para mensagens na DLQ
- Prefetch: 10 mensagens por consumidor

---

## 9. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 5 integrações, diagramas Mermaid de sequência, retry policy padrão, filas RabbitMQ, RLS policies Supabase. |
