# 26 - Runbook Operacional

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — Backend, DevOps, On-call |
| **Escopo** | Playbooks de incidente para os 8 cenários críticos do CRM |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | 24 Deploy CI-CD · 25 Observabilidade e Logs |

---

> **TL;DR**
>
> - **8 playbooks** para os cenários de incidente mais comuns do CRM.
> - **P0 (crítico):** API fora do ar, banco indisponível — SLA de resposta: 15 min.
> - **P1 (alto):** RabbitMQ travado, ZapSign indisponível, falha de sincronização — SLA: 1 hora.
> - **P2 (médio):** Relatório não gerado, SLA alert não disparado, usuário bloqueado — SLA: 4 horas.
> - Cada playbook: sintomas → diagnóstico → resolução passo a passo → escalada.

---

## Canais de Comunicação de Incidente

| Canal | Uso |
|---|---|
| `#crm-incidents` (Slack) | Abertura e atualizações de todos os incidentes P0/P1 |
| `#crm-alertas` (Slack) | Alertas automáticos do Sentry e monitoramento |
| `#crm-dev` (Slack) | Incidentes P2/P3 e problemas técnicos não urgentes |
| E-mail `tech-lead@repasseseguro.com.br` | Escalada para P0 se Slack não responder em 15 min |

---

## Playbook 1 — API Fora do Ar

**Severidade:** P0 | **SLA de resposta:** 15 min

### Sintomas
- Endpoint `/health` retorna 5xx ou não responde
- Sentry reporta Fatal com alto volume de erros
- Time informa que o CRM não abre (frontend retorna "Servidor indisponível")
- Railway Metrics: CPU/memória anormais ou instância com 0 requisições

### Diagnóstico

```bash
# 1. Verificar status atual no Railway
railway status --environment production

# 2. Ver logs recentes (últimos 5 min)
railway logs --environment production --tail 200

# 3. Verificar health check manualmente
curl -I https://api.crm.repasseseguro.com.br/health

# 4. Verificar se há deploy recente em andamento
railway deployments --environment production
# Se há deploy em andamento: aguardar conclusão (máx 5 min) antes de agir

# 5. Verificar métricas de CPU/memória no Railway Dashboard
# Railway → CRM API Service → Metrics
```

### Resolução

**Cenário A — Deploy com falha causou o crash:**
```bash
# 1. Identificar o último deploy estável
railway deployments --environment production

# 2. Reverter para o deployment anterior via Railway Dashboard:
#    Railway → CRM API → Deployments → [deployment anterior] → Redeploy

# 3. Aguardar health check (30–60s) e verificar:
curl https://api.crm.repasseseguro.com.br/health
```

**Cenário B — Crash loop por erro de aplicação:**
```bash
# 1. Verificar logs de erro
railway logs --environment production --tail 500 | grep ERROR

# 2. Identificar a exceção não tratada
# 3. Se causa é configuração (env var ausente): corrigir no Railway Dashboard → Variables
# 4. Reiniciar serviço manualmente:
#    Railway → CRM API → Settings → Restart Service

# 5. Se persistir: rollback conforme Cenário A
```

**Cenário C — Supabase banco indisponível (ver Playbook 2):**
- O `/health` retornará `"database": "down"`
- Executar Playbook 2 em paralelo

### Escalada
- Se não resolvido em **15 min**: acionar Tech Lead via Slack @menção + WhatsApp
- Se não resolvido em **30 min**: acionar CEO (interrupção de operação da equipe RS)
- Comunicar no `#crm-incidents`: "API CRM fora do ar desde [hora]. Investigando."

---

## Playbook 2 — Banco de Dados Indisponível

**Severidade:** P0 | **SLA de resposta:** 15 min

### Sintomas
- `/health` retorna `"database": "down"` ou HTTP 503
- Erros `P1001: Can't reach database server` nos logs do Railway
- Prisma lança `PrismaClientInitializationError` em todas as requisições
- Supabase Status Page (status.supabase.com) pode indicar incidente em curso

### Diagnóstico

```bash
# 1. Verificar status do Supabase
# Acessar: https://status.supabase.com
# Verificar se há incidente ativo para a região sa-east-1

# 2. Testar conexão direta ao Supabase via psql (se disponível)
psql $DIRECT_URL -c "SELECT 1;"

# 3. Verificar logs da API para o erro exato
railway logs --environment production --tail 200 | grep -i "database\|prisma\|P1001"

# 4. Verificar connection pool no Supabase Dashboard
# Supabase → Project → Database → Connection Pooling
# Checar: conexões ativas vs. máximo configurado
```

### Resolução

**Cenário A — Incidente no Supabase (fora do nosso controle):**
```
1. Monitorar status.supabase.com a cada 5 min
2. Comunicar no #crm-incidents: "Banco CRM indisponível. Incidente Supabase em andamento."
3. Ativar modo degradado: desabilitar features que exigem escrita (se possível via feature flag)
4. Aguardar resolução do Supabase (RTO contratual: 4h — RN-200)
5. Após resolução: verificar filas RabbitMQ para mensagens represadas (Playbook 3)
```

**Cenário B — Limite de conexões atingido:**
```bash
# 1. Verificar conexões ativas no Supabase Dashboard
# Se connection pool esgotado: reiniciar serviço CRM API para liberar conexões
railway restart --environment production --service crm-api

# 2. Se persistir: verificar se há queries longas travando conexões
# Supabase → SQL Editor:
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '30 seconds'
ORDER BY duration DESC;

# Cancelar queries longas se necessário:
SELECT pg_cancel_backend(pid) FROM pg_stat_activity
WHERE (now() - query_start) > interval '60 seconds';
```

**Cenário C — `DATABASE_URL` incorreta ou rotacionada:**
```bash
# Verificar e corrigir variável no Railway:
railway variables set DATABASE_URL="[nova_url]" --environment production
railway restart --environment production --service crm-api
```

### Escalada
- Se não resolvido em **15 min**: Tech Lead + notificar equipe RS da indisponibilidade
- Se incidente Supabase > **1 hora**: avaliar ativação de backup manual de leitura (se configurado)

---

## Playbook 3 — RabbitMQ com Fila Travada

**Severidade:** P1 | **SLA de resposta:** 1 hora

### Sintomas
- CloudAMQP mostra DLQ com mensagens acumuladas (`crm.*.dlq`)
- Notificações de atividades não estão chegando aos usuários internos
- Alertas de SLA não estão disparando (possível acúmulo em `crm.sla_alerts`)
- Relatório semanal não foi enviado na segunda-feira às 08h
- Logs indicam `ECONNREFUSED` para CloudAMQP URL

### Diagnóstico

```bash
# 1. Verificar status das filas no CloudAMQP Console
# CloudAMQP → Instance → RabbitMQ Management → Queues
# Verificar: profundidade, taxa de consumo, DLQ

# 2. Verificar logs da API para erros de RabbitMQ
railway logs --environment production --tail 300 | grep -i "rabbitmq\|amqp\|queue"

# 3. Verificar se worker está ativo
railway status --environment production

# 4. Testar conexão manualmente (se disponível)
# node -e "require('amqplib').connect(process.env.RABBITMQ_URL).then(c => { console.log('OK'); c.close(); })"
```

### Resolução

**Cenário A — Worker não está consumindo (bug ou crash):**
```bash
# 1. Reiniciar serviço CRM API (reinicia também os workers NestJS)
railway restart --environment production --service crm-api

# 2. Verificar se mensagens começam a ser consumidas:
# CloudAMQP → Queues → [fila afetada] → Rates → should show positive "deliver"

# 3. Verificar DLQ — mensagens que falharam após máx retries:
# CloudAMQP → Queues → crm.notifications.dlq → Get Messages
# Analisar payload e identificar causa da falha
```

**Cenário B — CloudAMQP indisponível:**
```
1. Verificar status do CloudAMQP (cloudamqp.com/status)
2. Comunicar no #crm-incidents: "Filas CRM indisponíveis. Notificações e relatórios atrasados."
3. Aguardar resolução do CloudAMQP
4. Após resolução: mensagens na fila serão processadas automaticamente
5. Verificar DLQ e reprocessar mensagens que falharam durante a indisponibilidade
```

**Cenário C — Reprocessar mensagens da DLQ:**
```bash
# Via RabbitMQ Management UI (CloudAMQP):
# 1. Acessar DLQ: crm.notifications.dlq
# 2. Get Messages → inspecionar payload
# 3. Se mensagens são válidas: usar "Move messages" para fila original
# 4. Se mensagens são inválidas (bug de serialização): deletar + corrigir bug + fazer novo deploy
```

### Escalada
- Se DLQ > 50 mensagens: acionar Tech Lead
- Se relatório semanal não enviado após segunda-feira 10h: acionar Coordenador RS para reenvio manual

---

## Playbook 4 — ZapSign Indisponível — Impacto em Assinaturas

**Severidade:** P1 | **SLA de resposta:** 1 hora

### Sintomas
- Chamadas ao ZapSign retornam 5xx ou timeout (> 10s)
- Casos em estágio FORMALIZACAO não conseguem ter envelope criado
- Logs indicam `ZapSignIntegrationError` ou `ECONNABORTED`
- Sentry alerta P1 para `ZapSignService`

### Diagnóstico

```bash
# 1. Verificar status do ZapSign
# Acessar: https://status.zapsign.com.br (ou verificar painel ZapSign)

# 2. Verificar logs de erro específicos do ZapSign
railway logs --environment production --tail 300 | grep -i "zapsign"

# 3. Testar API ZapSign manualmente (sem dados reais)
curl -X GET https://api.zapsign.com.br/api/v1/docs/ \
  -H "Authorization: Bearer $ZAPSIGN_API_TOKEN"
# Esperado: HTTP 200 com lista de documentos (pode ser vazia)
```

### Resolução

**Cenário A — Instabilidade temporária do ZapSign:**
```
Impacto: Criação de novos envelopes bloqueada.
Assinaturas em andamento (já enviadas): não afetadas.

1. Comunicar no #crm-incidents: "ZapSign instável. Criação de envelopes temporariamente indisponível."
2. Orientar equipe: casos em FORMALIZACAO aguardam normalização — não tentar recriar manualmente
3. As tentativas de criação de envelope ficam na fila crm.integrations.retry com retry automático
4. Monitorar CloudAMQP para ver quando o retry começa a ser bem-sucedido
5. Após normalização: verificar fila crm.integrations.retry e confirmar que mensagens foram processadas
```

**Cenário B — Token ZapSign expirado/revogado:**
```bash
# 1. Acessar painel ZapSign → API → Gerar novo token
# 2. Atualizar secret no Railway:
railway variables set ZAPSIGN_API_TOKEN="[novo_token]" --environment production

# 3. Reiniciar API para recarregar variável
railway restart --environment production --service crm-api

# 4. Verificar se integração voltou:
railway logs --environment production --tail 50 | grep zapsign
```

**Cenário C — ZapSign indisponível por > 4 horas:**
```
1. Notificar Coordenador RS e Tech Lead
2. Casos em FORMALIZACAO: registrar nota de atividade no CRM ("Assinatura pendente — ZapSign indisponível")
3. Avaliar com Coordenador RS se há alternativa temporária (assinatura física) para casos urgentes
4. Assim que ZapSign normalizar: reenviar envelopes dos casos afetados via CRM
```

### Escalada
- > 2 horas sem resolução: notificar suporte ZapSign via e-mail/telefone
- > 4 horas: Coordenador RS decide protocolo de contingência para assinaturas urgentes

---

## Playbook 5 — Falha de Sincronização com Platform RS

**Severidade:** P1 | **SLA de resposta:** 1 hora

### Sintomas
- Casos criados na Platform RS (módulo principal) não aparecem no CRM
- Casos atualizados no CRM não refletem na Platform RS
- Logs indicam erros de webhook ou requisição entre módulos
- Usuários reportam inconsistência de dados entre o CRM e a plataforma principal

### Diagnóstico

```bash
# 1. Verificar logs de sincronização
railway logs --environment production --tail 300 | grep -i "platform\|sync\|webhook"

# 2. Verificar se há mensagens na DLQ de integração
# CloudAMQP → crm.integrations.dlq → Get Messages

# 3. Verificar health check de ambos os serviços
curl https://api.crm.repasseseguro.com.br/health
curl https://api.repasseseguro.com.br/health  # Platform RS API

# 4. Identificar a última sincronização bem-sucedida:
# Supabase SQL Editor:
SELECT id, created_at, status FROM sync_logs
WHERE module = 'platform_rs'
ORDER BY created_at DESC
LIMIT 10;
```

### Resolução

**Cenário A — Webhook de sincronização falhando:**
```bash
# 1. Identificar o webhook específico (entrada ou saída) que está falhando
# 2. Verificar se o HMAC do webhook está correto (chaves podem ter sido rotacionadas)
# 3. Se HMAC inválido: verificar INTERNAL_HMAC_SECRET em ambos os serviços

# 4. Forçar ressincronização manual via endpoint administrativo (se disponível):
curl -X POST https://api.crm.repasseseguro.com.br/v1/admin/sync/platform-rs \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "since": "2026-03-23T00:00:00Z" }'
```

**Cenário B — Dados divergentes após período de sincronização falha:**
```
1. Identificar os casos afetados (IDs + período)
2. Notificar Coordenador RS sobre possível inconsistência
3. Para cada caso divergente: verificar audit_log para determinar versão correta
4. Aplicar correção manualmente (via Prisma Studio em último caso) com registro de atividade no CRM
5. Após correção: testar sincronização com um caso de teste para confirmar resolução
```

### Escalada
- Dados divergentes em casos com transações financeiras (FORMALIZACAO, CONCLUIDO): escalada imediata ao Tech Lead + Coordenador RS

---

## Playbook 6 — Relatório Semanal Não Gerado

**Severidade:** P2 | **SLA de resposta:** 4 horas

### Sintomas
- Segunda-feira após 08h30 sem e-mail de relatório semanal nos inboxes da equipe RS
- Ausência de log `msg: "Relatório semanal gerado"` no Railway
- Fila `crm.weekly_reports` com mensagens acumuladas ou DLQ com mensagem de falha

### Diagnóstico

```bash
# 1. Verificar se o job agendado foi disparado
railway logs --environment production | grep -i "weekly_report\|relatorio_semanal"

# 2. Verificar fila e DLQ do relatório
# CloudAMQP → crm.weekly_reports + crm.weekly_reports.dlq

# 3. Verificar se o cron job está configurado corretamente
# Supabase Edge Functions ou NestJS @Cron — checar configuração de timezone (America/Fortaleza)

# 4. Verificar se Resend está funcionando
curl -X GET https://api.resend.com/emails \
  -H "Authorization: Bearer $RESEND_API_KEY"
```

### Resolução

**Cenário A — Job agendado não disparou (cron misconfiguration):**
```bash
# 1. Acionar geração manual via endpoint administrativo:
curl -X POST https://api.crm.repasseseguro.com.br/v1/admin/reports/weekly/trigger \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{ "period": "2026-03-17/2026-03-23" }'  # período da semana anterior

# 2. Verificar e corrigir configuração do cron no NestJS:
# apps/api/modules/crm/reports/reports.service.ts
# @Cron('0 8 * * 1', { timeZone: 'America/Fortaleza' })  // toda segunda às 08h

# 3. Deploy da correção via pipeline normal
```

**Cenário B — Falha no envio de e-mail via Resend:**
```bash
# 1. Verificar status do Resend: https://resend.com/status
# 2. Verificar quota de envios do Resend (pode ter atingido limite diário)
# 3. Se Resend OK: verificar logs do Resend Dashboard para bounces ou rejeições
# 4. Após corrigir causa: reenviar relatório manualmente (ver Cenário A)
```

**Cenário C — Erro na geração do relatório (bug ou dados faltantes):**
```bash
# 1. Ver mensagem de erro na DLQ: crm.weekly_reports.dlq
# 2. Identificar e corrigir o bug
# 3. Deploy da correção
# 4. Reenviar manualmente
```

### Escalada
- Se não resolvido até segunda-feira 12h: notificar Coordenador RS e enviar relatório em formato manual (CSV exportado do CRM)

---

## Playbook 7 — SLA Alert Não Disparado

**Severidade:** P2 | **SLA de resposta:** 4 horas

### Sintomas
- Caso com prazo vencido não aparece com alerta no pipeline
- Usuário informa que não recebeu notificação de caso em risco de SLA
- Fila `crm.sla_alerts` vazia ou com erros
- Logs sem mensagem `msg: "Alerta de SLA gerado"` para casos que deveriam ter alerta

### Diagnóstico

```bash
# 1. Verificar se o monitor de SLA está rodando
railway logs --environment production | grep -i "sla_monitor\|sla_alert"

# 2. Verificar casos com SLA vencido no banco
# Supabase SQL Editor:
SELECT id, status, sla_deadline, created_at
FROM cases
WHERE sla_deadline < NOW()
  AND status NOT IN ('CONCLUIDO', 'CANCELADO')
  AND deleted_at IS NULL
ORDER BY sla_deadline ASC;

# 3. Verificar alertas já criados para esses casos
SELECT case_id, severity, created_at, notified_at
FROM sla_alerts
WHERE case_id IN ([ids_dos_casos_acima])
ORDER BY created_at DESC;

# 4. Verificar fila de alertas
# CloudAMQP → crm.sla_alerts
```

### Resolução

**Cenário A — Monitor de SLA não está executando:**
```bash
# 1. Verificar configuração do @Cron do SlaMonitorService
# Deve rodar a cada 5 min: @Cron('*/5 * * * *')

# 2. Reiniciar API para reativar cron jobs:
railway restart --environment production --service crm-api

# 3. Aguardar próxima execução e verificar logs
```

**Cenário B — Alertas criados mas notificação não chegou:**
```bash
# 1. Verificar fila crm.sla_alerts e DLQ
# 2. Se mensagens na DLQ: analisar causa do erro de processamento
# 3. Reprocessar mensagens válidas da DLQ (ver Playbook 3)
# 4. Para casos críticos com SLA vencido: criar alerta manualmente via API administrativa
curl -X POST https://api.crm.repasseseguro.com.br/v1/admin/sla/trigger-alert \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{ "caseId": "uuid-do-caso" }'
```

**Cenário C — Bug na lógica de cálculo de SLA:**
```
1. Identificar casos afetados
2. Notificar Analistas RS responsáveis manualmente (Slack)
3. Corrigir bug e fazer deploy
4. Retroativamente gerar alertas para casos que deveriam ter sido alertados
```

### Escalada
- Se casos com SLA vencido há > 24h sem alerta: notificar Coordenador RS imediatamente para tratativa manual

---

## Playbook 8 — Usuário Bloqueado Sem Conseguir Acesso

**Severidade:** P2 | **SLA de resposta:** 4 horas

### Sintomas
- Membro da equipe RS não consegue fazer login no CRM
- Mensagem de erro: "Conta bloqueada" ou "Credenciais inválidas" mesmo com senha correta
- Pode ocorrer após 5 tentativas de login incorretas (RN-002: bloqueio por 30 min)

### Diagnóstico

```bash
# 1. Verificar status do usuário no banco
# Supabase SQL Editor (usar UUID, nunca e-mail nos logs):
SELECT id, role, is_active, failed_login_attempts, locked_until
FROM users
WHERE id = '[uuid_do_usuario]';

# 2. Verificar logs de tentativas de login
railway logs --environment production | grep -i "auth\|login\|blocked" | grep "[uuid_do_usuario]"

# 3. Verificar se o usuário existe no Supabase Auth
# Supabase Dashboard → Authentication → Users → buscar por e-mail
```

### Resolução

**Cenário A — Bloqueio por tentativas incorretas (RN-002 — bloqueio de 30 min):**
```bash
# Aguardar 30 min para desbloqueio automático (preferencial)
# OU desbloquear manualmente via Supabase SQL (somente Admin RS ou Tech Lead):

UPDATE users
SET failed_login_attempts = 0,
    locked_until = NULL,
    updated_at = NOW()
WHERE id = '[uuid_do_usuario]';

# Registrar ação no audit_log:
INSERT INTO audit_log.actions (user_id, action, target_user_id, metadata)
VALUES ('[uuid_admin]', 'MANUAL_UNBLOCK', '[uuid_do_usuario]', '{"reason": "solicitacao_usuario", "ticket": "CRM-XXX"}');
```

**Cenário B — Conta desativada por Admin RS:**
```bash
# 1. Verificar se is_active = false
# 2. Se desativação foi intencional: confirmar com Coordenador RS antes de reativar
# 3. Reativar se autorizado:
UPDATE users SET is_active = true, updated_at = NOW() WHERE id = '[uuid]';
```

**Cenário C — Problema no Supabase Auth (sessão corrompida):**
```bash
# 1. Supabase Dashboard → Authentication → Users → [usuário]
# 2. Invalidar todas as sessões ativas do usuário
# 3. Solicitar que o usuário tente novo login
# 4. Se persistir: resetar senha via "Send magic link" no Supabase Auth
```

**Cenário D — Papel do usuário incorreto (acesso negado mesmo logado):**
```bash
# 1. Verificar role no banco vs. claims do JWT
# 2. Role foi atualizada no banco mas JWT antigo ainda está ativo?
# 3. Solicitar logout + login ao usuário (renova JWT com claims atualizados)
# 4. Se role estiver errada no banco:
UPDATE users SET role = 'ANALISTA_RS', updated_at = NOW() WHERE id = '[uuid]';
# + registrar no audit_log
```

### Escalada
- Usuário bloqueado que impede operação crítica (ex.: único Coordenador RS disponível): escalar para Admin RS imediatamente para desbloqueio manual

---

## 9. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 8 playbooks de incidente: API fora do ar, banco indisponível, RabbitMQ travado, ZapSign indisponível, falha de sincronização com Platform RS, relatório semanal não gerado, SLA alert não disparado, usuário bloqueado. Cada playbook com sintomas, diagnóstico com comandos, resolução por cenário e escalada. |
