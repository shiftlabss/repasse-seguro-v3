# 26 - Runbook Operacional

## Módulo Cedente · Plataforma Repasse Seguro

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
|---|---|---|---|---|
| 26 - Runbook Operacional | v1.0 | 2026-03-23 (America/Fortaleza) | Claude Code Desktop | Aprovado |

| Campo | Valor |
|---|---|
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 |

---

> **TL;DR — Resumo Operacional**
>
> - **4 severidades:** P0 (crítico/5min), P1 (alto/15min), P2 (médio/30min), P3 (baixo/2h).
> - **Componentes críticos do Cedente:** API NestJS (Railway), PostgreSQL (Supabase), ZapSign (assinaturas), Escrow (financeiro), OpenAI/Guardião (IA), Redis (Upstash), RabbitMQ (CloudAMQP).
> - **Rollback:** `railway rollback` em < 60s; Vercel tem rollback instantâneo no dashboard.
> - **Comunicação:** Slack `#incidentes` (P0/P1) e `#alertas` (P2/P3).
> - **Post-mortem obrigatório** para todo incidente P0 e P1 que afete fluxo de Fechamento, assinatura ZapSign ou Escrow.
> - **Escalation:** Tech Lead → responsável técnico → suporte das plataformas externas.
> - **RN-080:** ZapSign down → exibir mensagem clara ao Cedente. Nunca silenciar o erro.
> - **RN-083:** Escrow down → exibir último status conhecido com timestamp.

---

## 1. Classificação de Severidade

| Severidade | Definição | SLA Resposta | SLA Resolução | Canal | Escalation |
|---|---|---|---|---|---|
| **P0 — Crítico** | Indisponibilidade total ou falha em fluxo de Fechamento/assinatura/Escrow em produção | 5 minutos | 1 hora | Slack `#incidentes` + PagerDuty | Tech Lead imediato |
| **P1 — Alto** | Degradação severa de funcionalidade principal; upload bloqueado; Guardião do Retorno fora | 15 minutos | 2 horas | Slack `#incidentes` | Tech Lead em 15min |
| **P2 — Médio** | Funcionalidade secundária degradada; latência elevada; notificações atrasadas | 30 minutos | 4 horas | Slack `#alertas` | Tech Lead se não resolvido em 2h |
| **P3 — Baixo** | Anomalia não urgente; log anômalo; métrica levemente fora do normal | 2 horas | 1 dia útil | Slack `#alertas` ou issue | Tech Lead no próximo dia útil |

### 1.1 Componentes e criticidade

| Componente | Criticidade | RN Relacionada | Razão |
|---|---|---|---|
| API backend — NestJS (Railway) | P0 | Toda operação | Toda a operação depende |
| PostgreSQL (Supabase) | P0 | RN-011 | Dados da plataforma e isolamento |
| ZapSign — Assinaturas | P1 | RN-080 | Bloqueia formalização e Fechamento |
| Parceiro Escrow | P1 | RN-083 | Movimentação financeira; fallback: exibir último status |
| OpenAI / Guardião do Retorno | P1 | RN-058 a RN-063 | Funcionalidade de IA do produto |
| Supabase Realtime | P1 | RN-057, RN-014 | Notificações e dashboard em tempo real |
| Redis (Upstash) | P1 | RN-005, RN-006 | Cache de sessão; bloqueio por tentativas |
| RabbitMQ (CloudAMQP) | P1 | RN-056 | Notificações por e-mail via Resend |
| Resend — E-mail | P2 | RN-056 | Fallback: notificação no painel sempre ativa |
| Expo Push Notifications | P2 | RN-057 | Fallback: e-mail e painel |
| Receita Federal — CNPJ | P2 | RN-082 | Fallback: banner de pendência + verificação manual pelo Admin |
| Langfuse | P3 | — | Observabilidade; produto funciona sem ele |

---

## 2. Triagem Inicial

Ao receber alerta ou relato de incidente, executar nesta ordem:

```
1. CONFIRMAR IMPACTO
   - O alerta é real ou falso positivo?
   - Quantos Cedentes estão afetados?
   - Qual funcionalidade está impactada?
   - Há fluxo de Fechamento ou assinatura em andamento?

2. ISOLAR ESCOPO
   - O problema é em produção, staging ou ambos?
   - É o Módulo Cedente ou toda a plataforma?
   - Foi precedido por um deploy recente (< 30min)?

3. VERIFICAR DEPLOY RECENTE
   railway deployments --service cedente-api-prod
   - Se há deploy recente: iniciar rollback sem aguardar diagnóstico completo (P0/P1)

4. VALIDAR DEPENDÊNCIAS EXTERNAS
   - ZapSign: verificar painel de administração ZapSign
   - Supabase: https://status.supabase.com
   - OpenAI: https://status.openai.com
   - Upstash (Redis): https://status.upstash.com
   - CloudAMQP (RabbitMQ): https://status.cloudamqp.com
   - Resend: https://status.resend.com
   - Parceiro Escrow: [URL da status page do parceiro — DEFINIÇÃO PENDENTE DP-001]

5. CONSULTAR DASHBOARDS
   - Sentry: https://sentry.io/{org}/repasse-seguro
   - Railway: https://railway.app/dashboard
   - Supabase Dashboard: https://app.supabase.com
   - Langfuse (se Guardião envolvido): https://cloud.langfuse.com
```

---

## 3. Runbooks por Incidente

### INC-01 — Upload de Documentos com Falha

**Sintomas:**
- Cedentes relatam erro ao enviar documentos (PDF, JPG, PNG) no dossiê.
- Sentry: erros em `module: cedente-documentos`, `action: upload`.
- Mensagem de erro: "Não foi possível enviar o arquivo. Tente novamente."

**Severidade:** P1 (bloqueia avanço para triagem — RN-043)

**Diagnóstico:**

```bash
# 1. Verificar logs do backend
railway logs --service cedente-api-prod | grep '"action":"upload"' | tail -50

# 2. Verificar Supabase Storage
# app.supabase.com → Storage → Documents bucket
# Verificar: bucket existe? Policies de RLS corretas?

# 3. Testar upload manualmente
curl -X POST https://api-cedente.up.railway.app/api/v1/cedente/documentos/signed-url \
  -H "Authorization: Bearer <token-teste>" \
  -H "Content-Type: application/json" \
  -d '{"filename": "test.pdf", "mimeType": "application/pdf", "size": 1024}'
# Esperado: { "signedUrl": "https://...", "path": "..." }

# 4. Verificar cota do Supabase Storage
# app.supabase.com → Settings → Usage — verificar storage usado
```

**Causas comuns e resoluções:**

| Causa | Diagnóstico | Resolução |
|---|---|---|
| Supabase Storage down | Status page + erro 503 nos logs | Aguardar recuperação; informar Cedentes via painel |
| Bucket inexistente ou mal configurado | Erro "Bucket not found" nos logs | `supabase storage buckets create documents` ou recriar via dashboard |
| Policy RLS bloqueando upload | Erro 403 nos logs | Revisar policies em `storage.objects` — garantir policy para Cedentes autenticados |
| Arquivo com MIME type inválido | Erro 400 com "MIME type not allowed" nos logs | Comportamento esperado (RN-042); verificar se usuário está tentando enviar tipo não permitido |
| Arquivo > 10 MB | Erro 400 com "File too large" nos logs | Comportamento esperado (RN-042); mensagem de erro adequada |
| Signed URL expirada | Erro 403 após upload | Verificar TTL da signed URL no código (deve ser ≥ 5 minutos) |

**Resolução para Supabase Storage down:**

```bash
# Informar usuários afetados via Supabase Realtime (se Realtime ainda funcionar)
# ou via e-mail transacional (Resend)

# Após recuperação do Supabase:
# 1. Verificar se uploads em fila foram processados
# 2. Notificar Cedentes que podem retomar o upload
```

**Prevenção:**
- Monitorar cota do Supabase Storage semanalmente.
- Manter alerta no Sentry para erros de upload > 3/minuto.
- Implementar retry automático no frontend (3x com backoff) para erros 5xx.

---

### INC-02 — Webhook ZapSign Não Recebido (Assinatura Pendente > 1 Hora)

**Sintomas:**
- Cedente assinou documento no ZapSign mas status não atualizou no sistema.
- Admin alerta que assinatura aparece como "Pendente" apesar de concluída no ZapSign.
- Logs sem entradas de `action: zapsign_webhook_recebido` após a assinatura.

**Severidade:** P1 (bloqueia progresso de caso — RN-047, RN-048)

**Diagnóstico:**

```bash
# 1. Verificar se o webhook chegou ao backend
railway logs --service cedente-api-prod | grep '"integration":"zapsign"' | tail -100

# 2. Verificar fila RabbitMQ — há mensagens acumuladas?
# http://localhost:15672 (dev) ou CloudAMQP dashboard (prod)
# Fila: zapsign.webhooks

# 3. Verificar se o endpoint de webhook está acessível
curl -X POST https://api-cedente.up.railway.app/api/v1/webhooks/zapsign \
  -H "Content-Type: application/json" \
  -H "X-ZapSign-Secret: <webhook-secret>" \
  -d '{"event": "ping"}'
# Esperado: {"received": true}

# 4. Verificar configuração do webhook no painel ZapSign
# - URL configurada: https://api-cedente.up.railway.app/api/v1/webhooks/zapsign
# - Secret configurado corretamente
# - Verificar logs de delivery no painel ZapSign (tentativas falharam?)

# 5. Verificar HMAC do webhook
# O backend valida a assinatura do webhook com ZAPSIGN_WEBHOOK_SECRET
# Erro 401 nos logs → secret incorreto ou rotacionado
```

**Causas comuns e resoluções:**

| Causa | Diagnóstico | Resolução |
|---|---|---|
| URL do webhook incorreta no ZapSign | ZapSign mostra erro 404 nos logs de delivery | Reconfigurar URL no painel ZapSign |
| Secret rotacionado sem atualizar ZapSign | Erro 401 nos logs do backend | Atualizar secret no painel ZapSign e Railway |
| Backend estava down quando webhook chegou | ZapSign mostra erro 5xx | ZapSign faz retry por até 24h; aguardar ou reprocessar manualmente |
| Bug no processamento do webhook | Erro 500 nos logs com stack trace | Investigar e corrigir; ver INC-06 para taxa de erro geral |
| Fila RabbitMQ acumulada | Mensagens na fila sem consumidor | Reiniciar consumer: `railway restart --service cedente-api-prod` |

**Resolução manual (reprocessar webhook perdido):**

```bash
# Buscar dados da assinatura diretamente na API do ZapSign
curl -X GET "https://api.zapsign.com.br/api/v1/docs/<doc-id>/" \
  -H "Authorization: Bearer <zapsign-api-key>"

# Atualizar status manualmente via endpoint admin (operação restrita)
curl -X POST https://api-cedente.up.railway.app/api/v1/admin/assinaturas/sync \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{"casoId": "<caso-id>", "documentoId": "<doc-id>", "zapsignDocId": "<zapsign-doc-id>"}'
```

**Prevenção:**
- Monitorar painel ZapSign semanalmente para erros de delivery.
- Implementar job de reconciliação: diariamente buscar documentos "Pendentes" há mais de 2h e sincronizar com ZapSign.
- Alerta: Sentry + log quando `action: zapsign_webhook_ausente_apos_1h`.

---

### INC-03 — Webhook Escrow Não Recebido (Depósito Não Confirmado)

**Sintomas:**
- Cessionário reporta que fez depósito mas status do caso não avançou para "Depósito confirmado".
- Painel financeiro do Cedente mostra status de Escrow desatualizado.
- Logs sem entradas de `integration: escrow` recentes.

**Severidade:** P1 (bloqueia Fechamento — RN-083)

> **Nota:** O parceiro Escrow ainda não foi definido (DEFINIÇÃO PENDENTE DP-001). Este runbook usa endpoints e comportamentos genéricos. Atualizar com dados reais quando parceiro for escolhido.

**Diagnóstico:**

```bash
# 1. Verificar logs de webhook do Escrow
railway logs --service cedente-api-prod | grep '"integration":"escrow"' | tail -50

# 2. Verificar endpoint de webhook
curl -X POST https://api-cedente.up.railway.app/api/v1/webhooks/escrow \
  -H "Content-Type: application/json" \
  -H "X-Escrow-Secret: <webhook-secret>" \
  -d '{"event": "ping"}'
# Esperado: {"received": true}

# 3. Verificar status page do parceiro Escrow
# [URL da status page — DP-001 pendente]

# 4. Verificar banco — qual o status atual registrado
# app.supabase.com → SQL Editor:
SELECT id, status_escrow, updated_at
FROM cedente_casos
WHERE id = '<caso-id>';
```

**Comportamento esperado conforme RN-083:**
- Se integração Escrow indisponível: sistema exibe último status conhecido com timestamp.
- Interface: "Informação atualizada em [data/hora]. Aguardando atualização do serviço financeiro."
- O Cedente **nunca** deve ver o painel sem nenhuma informação — sempre exibir último estado.

**Resolução:**

```bash
# 1. Se parceiro Escrow retorna online: aguardar reconciliação automática
# O backend deve ter job de polling como fallback ao webhook

# 2. Se precisa atualizar manualmente (via Admin):
# Admin confirma depósito via painel administrativo
# Registrar no log: motivo da atualização manual

# 3. Notificar o Cedente que o status será atualizado assim que o serviço financeiro responder
```

**Prevenção:**
- Implementar job de polling ao Escrow como fallback ao webhook (a cada 5 minutos quando há casos em status "Aguardando depósito").
- Alerta: log de `escrow_webhook_ausente` após 30 minutos sem confirmação de depósito esperado.

---

### INC-04 — Guardião do Retorno Indisponível (OpenAI Outage)

**Sintomas:**
- Cedentes relatam que o chat com o Guardião não responde ou retorna erro.
- Sentry: erros em `module: cedente-guardiao`, `integration: openai`.
- Langfuse: latência > 30s ou erros 5xx nas traces.

**Severidade:** P1

**Diagnóstico:**

```bash
# 1. Verificar status da OpenAI
curl -s https://status.openai.com/api/v2/summary.json | jq '.status.description'
# Se "Degraded Performance" ou "Partial Outage": causa externa confirmada

# 2. Verificar logs do Guardião
railway logs --service cedente-api-prod | grep '"module":"cedente-guardiao"' | tail -50

# 3. Testar diretamente a API OpenAI
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  https://api.openai.com/v1/models
# Esperado: 200

# 4. Verificar Langfuse
# cloud.langfuse.com → Traces → filtrar last 1h → ver erros
```

**Resolução:**

```bash
# 1. Se outage confirmado na OpenAI:
# - Ativar feature flag de kill switch via PostHog:
# posthog.com → Feature Flags → "guardiao-do-retorno" → disable (100% usuários)
# - Exibir mensagem no chat: "O assistente está temporariamente indisponível.
#   Nossa equipe foi notificada. Por favor, tente novamente em alguns minutos."

# 2. Após recovery da OpenAI:
# - Reabilitar feature flag: "guardiao-do-retorno" → enable (100% usuários)
# - Verificar Langfuse: latência voltou ao normal

# 3. Verificar se há conversas que precisam ser retomadas
# (RN-062: histórico permanente e imutável — nenhuma conversa é perdida)
```

**Rollout progressivo (pós-outage):**
```bash
# Reabilitar Guardião para 10% dos usuários primeiro
# posthog.com → Feature Flags → "guardiao-do-retorno" → rollout: 10%
# Monitorar Langfuse por 30 minutos
# Se estável: subir para 50%, depois 100%
```

**Prevenção:**
- Feature flag de kill switch (`guardiao-do-retorno`) obrigatória — já configurada na stack.
- Alerta Sentry: erros em `module: cedente-guardiao` > 5/minuto → ativar kill switch automaticamente (a implementar).
- Rate limiting de 30 req/min por usuário protege contra loops (Stacks 9.3).

---

### INC-05 — Taxa de Erro > 5% nas Últimas 5 Minutos

**Sintomas:**
- Sentry alerta: taxa de erro acima do threshold.
- Usuários relatam comportamento inconsistente (telas não carregam, ações não salvam).
- Railway: CPU ou memória pico anormal.

**Severidade:** P0 se toda a API; P1 se módulo específico.

**Diagnóstico imediato (primeiros 5 minutos):**

```bash
# 1. Verificar se há deploy recente
railway deployments --service cedente-api-prod
# Se deploy < 30min atrás: rollback IMEDIATO

# 2. Verificar a taxa de erro por endpoint no Sentry
# sentry.io → Performance → sort by error rate

# 3. Verificar logs em tempo real
railway logs --service cedente-api-prod --tail

# 4. Verificar saúde do banco
curl https://api-cedente.up.railway.app/health
# Esperado: {"status":"ok","timestamp":"..."}

# Se health check falha:
railway restart --service cedente-api-prod
```

**Rollback de emergência:**

```bash
# Se confirmado que deploy causou o problema:
railway rollback --service cedente-api-prod
# Aguardar 60s e verificar taxa de erro caindo no Sentry

# Para o frontend (se necessário):
vercel rollback <deployment-url>
```

**Diagnóstico pós-estabilização:**

```bash
# Buscar o padrão de erro
railway logs --service cedente-api-prod | grep '"level":"error"' | jq . | head -50

# Identificar endpoint/módulo mais afetado
# Sentry → Issues → sort by frequency, last 1h

# Verificar se é erro de banco
railway logs --service cedente-api-prod | grep 'prisma' | tail -20
```

**Prevenção:**
- CI obrigatório antes de qualquer deploy (lint + type-check + testes).
- Health check configurado no Railway com restart automático.
- Alerta de taxa de erro no Sentry com threshold de 1%.

---

### INC-06 — Banco de Dados Lento (Query > 1s)

**Sintomas:**
- Latência P95 da API acima do SLA de 200ms.
- Railway logs com requests demorando > 1s.
- Sentry performance: transactions lentas no módulo Cedente.

**Severidade:** P2 (degradação de experiência); P1 se latência > 3s.

**Diagnóstico:**

```bash
# 1. Verificar Supabase Query Performance
# app.supabase.com → Reports → Query Performance
# Identificar queries com mean_exec_time > 100ms

# 2. Via SQL direto (executar no Supabase SQL Editor)
SELECT
  query,
  calls,
  total_exec_time,
  mean_exec_time,
  rows
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC
LIMIT 20;

# 3. Verificar conexões ativas
SELECT count(*), state
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state;
# Alerta se total de conexões > 80% do pool

# 4. Verificar queries lentas em execução no momento
SELECT
  pid,
  now() - pg_stat_activity.query_start AS duration,
  query,
  state
FROM pg_stat_activity
WHERE state != 'idle'
  AND (now() - pg_stat_activity.query_start) > interval '1 second';
```

**Causas comuns e resoluções:**

| Causa | Diagnóstico | Resolução |
|---|---|---|
| Query sem índice | `EXPLAIN ANALYZE` mostra Sequential Scan em tabela grande | Criar índice: `CREATE INDEX CONCURRENTLY ...` |
| N+1 queries | Logs com muitas queries similares por request | Adicionar `include` no Prisma para eager loading |
| Pool de conexões esgotado | Conexões > 80% do limite | Verificar vazamentos de conexão; ajustar pool size no Prisma |
| Migration recente alterou índices | Correlacionar com horário do deploy | Analisar migration; adicionar índices ausentes |
| Tabela de logs crescendo | Tabela `logs` ou `notificacoes` sem particionamento | Arquivar dados antigos; adicionar índice em `created_at` |

**Resolver query lenta sem downtime:**

```sql
-- Criar índice sem bloquear a tabela (CONCURRENTLY)
CREATE INDEX CONCURRENTLY idx_cedente_casos_status
ON cedente_casos (cedente_id, status)
WHERE deleted_at IS NULL;

-- Verificar se índice foi criado
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'cedente_casos';
```

**Prevenção:**
- `EXPLAIN ANALYZE` obrigatório para queries em tabelas > 1000 linhas antes de PR.
- Alertas Supabase para queries > 500ms.
- Revisão mensal de `pg_stat_statements` para identificar queries em deterioração.

---

### INC-07 — Autenticação Falhando (JWT / Supabase Auth)

**Sintomas:**
- Cedentes relatam que não conseguem fazer login ou são deslogados inesperadamente.
- Sentry: erros em `module: cedente-auth`.
- Erro 401 recorrente nos logs mesmo com credenciais válidas.

**Severidade:** P0 (nenhum Cedente consegue operar)

**Diagnóstico:**

```bash
# 1. Verificar status do Supabase Auth
# app.supabase.com → projeto → Authentication → Users
# Testar: consegue buscar um usuário específico?

# 2. Verificar se JWT_SECRET foi rotacionado sem aviso
railway variables --service cedente-api-prod | grep JWT_SECRET
# Se foi rotacionado: todos os tokens emitidos com o secret antigo são inválidos

# 3. Verificar expiração de certificados
curl -s https://api-cedente.up.railway.app/health -v 2>&1 | grep -i "ssl\|certificate\|expire"

# 4. Verificar Supabase Auth status
curl -s "https://<supabase-project-id>.supabase.co/auth/v1/health" \
  -H "apikey: <anon-key>"
# Esperado: {"status":"healthy"}
```

**Resolução:**

```bash
# Se JWT_SECRET foi rotacionado acidentalmente:
# ATENÇÃO: Não reverta o secret — isso invalidaria tokens legítimos novamente.
# Solução: comunicar aos Cedentes que precisam fazer novo login.
# Notificar via Resend: "Por segurança, sua sessão foi encerrada. Faça login novamente."

# Se Supabase Auth está com problema:
# 1. Verificar status.supabase.com
# 2. Abrir ticket de suporte Supabase se P0
# 3. Fallback temporário: usar apenas JWT próprio (ajuste no Guard NestJS)
```

**Prevenção:**
- Processo de rotação de `JWT_SECRET` deve incluir período de transição (aceitar ambos os secrets por 1h).
- Monitorar rate de erros 401 no Sentry — spike anormal indica problema de auth.
- RN-005: bloqueio por 5 tentativas — verificar se Redis está funcionando (bloqueios dependem de Redis).

---

### INC-08 — Notificações Não Sendo Entregues

**Sintomas:**
- Cedentes relatam que não receberam e-mail de notificação (proposta, assinatura, etc.).
- Badge de notificações no painel não atualiza em até 30 segundos (RN-057).

**Severidade:** P2 para e-mails; P1 se notificações no painel afetarem decisões críticas (proposta prestes a expirar — RN-031).

**Diagnóstico — E-mail (Resend):**

```bash
# 1. Verificar Resend dashboard
# resend.com → Emails → filtrar por status "failed" ou "bounced"

# 2. Verificar status do Resend
curl -s https://resend.com/api/emails \
  -H "Authorization: Bearer <resend-api-key>" | jq '.data[0]'

# 3. Verificar fila RabbitMQ
# CloudAMQP dashboard → fila email.notifications → mensagens pendentes

# 4. Verificar logs do consumer
railway logs --service cedente-api-prod | grep '"action":"email_enviado"\|"action":"email_falhou"' | tail -20
```

**Diagnóstico — Notificações no painel (Supabase Realtime):**

```bash
# 1. Verificar Supabase Realtime
# app.supabase.com → Realtime → Inspect → verificar se subscriptions estão ativas

# 2. Verificar se tabela de notificações tem Realtime habilitado
# app.supabase.com → Database → Replication → verificar tabela cedente_notificacoes

# 3. Testar conexão Realtime (console do navegador do Cedente):
# supabase.channel('notificacoes').on('postgres_changes', ...).subscribe()
# Verificar se subscription retorna 'SUBSCRIBED'
```

**Resolução:**

```bash
# E-mail — reprocessar fila acumulada
# Se RabbitMQ tem mensagens acumuladas mas consumer está parado:
railway restart --service cedente-api-prod
# Consumer reinicia e processa a fila

# Realtime — forçar atualização
# Se Realtime está com atraso, forçar refresh via polling temporário:
# O frontend deve ter fallback de polling a cada 60s quando Realtime desconecta (RN-014)
```

---

## 4. Procedimentos de Escalamento

### 4.1 Fluxo de escalamento

```
[Alerta automático Sentry/Railway]
        │
        ▼
Tech Lead on-call
  (Slack @tech-lead | PagerDuty)
        │
        ├── P0/P1 não resolvido em 30 min
        │         ▼
        │   Responsável Técnico Principal
        │   (Fernando Calado | +55 xx xxxxx-xxxx [PREENCHER])
        │
        └── P0 com impacto em dados financeiros
                  ▼
            [Protocolo de dados financeiros]
            [Notificar todas as partes envolvidas]
```

### 4.2 Contatos de emergência

| Função | Nome | Contato | Disponibilidade |
|---|---|---|---|
| Tech Lead on-call | [PREENCHER] | Slack `@tech-lead` / [telefone] | 24/7 para P0 |
| Responsável Técnico | Fernando Calado | [PREENCHER] | 24/7 para P0 |
| Suporte Railway | — | [railway.app/help](https://railway.app/help) | Tickets |
| Suporte Supabase | — | [supabase.com/support](https://supabase.com/support) | Tickets (plano Pro: email) |
| Suporte ZapSign | — | [PREENCHER - contato comercial ZapSign] | Horário comercial |
| Suporte OpenAI | — | [help.openai.com](https://help.openai.com) | Tickets |
| Suporte parceiro Escrow | — | [PREENCHER quando DP-001 definido] | [PREENCHER] |

### 4.3 Comunicação durante incidente P0

```
1. Criar thread no Slack #incidentes:
   "[P0] [CEDENTE] <descrição do problema> — <timestamp>"

2. Atualizar a cada 15 minutos:
   "[UPDATE 15min] Status: em investigação | Impacto: <N cedentes> | ETA: <estimativa>"

3. Quando resolvido:
   "[RESOLVIDO] Duração: <X min> | Causa raiz: <resumo> | Post-mortem: <link>"
```

---

## 5. Checklist de Deploy em Produção

Execute este checklist antes de cada deploy. Não pule etapas.

### 5.1 Pré-deploy (D-1 ou D-2h)

- [ ] CI passou (lint, type-check, testes, build) em `main`
- [ ] PR foi revisado e aprovado por pelo menos 1 reviewer
- [ ] Migration testada em staging — dados íntegros, queries sem degradação
- [ ] Novas variáveis de ambiente configuradas no Railway prod e Vercel prod (se aplicável)
- [ ] Sentry DSN configurado para novos projetos (se aplicável)
- [ ] Comunicar deploy no Slack `#deploys`: "Deploy agendado para [hora] — [resumo das mudanças]"
- [ ] Verificar se há casos em formalização ativa (ZapSign + Escrow em andamento) — evitar deploy em horário de pico

### 5.2 Durante o deploy

- [ ] Acompanhar Railway deployment log em tempo real
- [ ] Verificar health check pós-deploy: `curl https://api-cedente.up.railway.app/health`
- [ ] Verificar Vercel deployment completou com sucesso
- [ ] Confirmar no Sentry: zero erros novos nos primeiros 5 minutos

### 5.3 Pós-deploy (30 minutos de observação)

- [ ] Taxa de erro Sentry < 0.1%
- [ ] Latência P95 API < 200ms (verificar Railway metrics)
- [ ] Supabase Dashboard: queries sem degradação
- [ ] Testar manualmente os fluxos críticos:
  - [ ] Login de Cedente (RN-001)
  - [ ] Upload de documento (RN-042)
  - [ ] Visualização de proposta com timer (RN-030)
  - [ ] Chat com Guardião do Retorno (RN-058) — se feature habilitada
  - [ ] Recebimento de notificação no painel (RN-057)
- [ ] Confirmar no Slack `#deploys`: "Deploy concluído ✓"

### 5.4 Rollback imediato se:

- Taxa de erro > 5% nos primeiros 5 minutos.
- Health check falhando.
- Latência P95 > 500ms por mais de 5 minutos.
- Qualquer erro em fluxo de assinatura ZapSign ou Escrow.

```bash
# Rollback backend
railway rollback --service cedente-api-prod

# Rollback frontend (Vercel dashboard → Deployments → promote anterior)
```

---

## 6. Post-Mortem

**Obrigatório para:**
- Todo incidente P0.
- Todo incidente P1 que afete assinatura ZapSign, Escrow ou fluxo de Fechamento.
- Incidentes P2 recorrentes (mesmo problema pela terceira vez).

**Template de post-mortem:**

```markdown
# Post-Mortem — [Título do Incidente]

**Data:** 2026-03-23
**Duração:** Xh Ymin (HH:MM — HH:MM)
**Severidade:** P0 / P1
**Módulo afetado:** Cedente — [submódulo]
**RNs impactadas:** RN-XXX, RN-XXX

## Resumo
[2-3 frases descrevendo o incidente e impacto]

## Timeline
- HH:MM — Alerta detectado por [Sentry / usuário / monitoramento]
- HH:MM — Tech Lead notificado
- HH:MM — Diagnóstico iniciado
- HH:MM — Causa raiz identificada
- HH:MM — Resolução aplicada
- HH:MM — Serviço normalizado
- HH:MM — Monitoramento confirmou estabilidade

## Causa Raiz
[Descrição técnica da causa raiz]

## Impacto
- Usuários afetados: [N Cedentes]
- Funcionalidades indisponíveis: [lista]
- Casos impactados: [N casos]

## O que funcionou bem
- [item 1]

## O que pode melhorar
- [item 1]

## Ações preventivas
| Ação | Responsável | Prazo |
|---|---|---|
| [Ação 1] | [Nome] | [Data] |
```

---

## 7. Referências

- Doc 02 — Stacks Tecnológicas: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/02 - Stacks.md`
- Doc 24 — Deploy, CI/CD e Versionamento: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/24 - Deploy, CI-CD e Versionamento.md`
- Doc 25 — Observabilidade e Logs: `docs/01 - Módulos/Cedente/02 - Desenvolvimento/25 - Observabilidade e Logs.md`
- Regras de Negócio (Integrações): `docs/01 - Módulos/Cedente/02 - Desenvolvimento/01.5 - Regras de Negócio — Integrações, Transversais e Consolidação.md`
- Status Supabase: [status.supabase.com](https://status.supabase.com)
- Status OpenAI: [status.openai.com](https://status.openai.com)
- Status Upstash: [status.upstash.com](https://status.upstash.com)
- Status CloudAMQP: [status.cloudamqp.com](https://status.cloudamqp.com)
- Status Resend: [status.resend.com](https://status.resend.com)
