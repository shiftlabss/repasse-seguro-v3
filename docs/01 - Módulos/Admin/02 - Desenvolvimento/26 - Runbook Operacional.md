# 26 - Runbook Operacional

## Repasse Seguro — Módulo Admin

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Backend, SRE, DevOps e Operação |
| **Escopo** | Resposta a alertas, diagnóstico, recuperação operacional, rollback, restore e post-mortem |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Status** | Aprovado |
| **Dependências** | D05 PRD · D14 Especificações Técnicas · D20 Error Handling · D24 Deploy CI-CD · D25 Observabilidade e Logs |

---

> 📌 **TL;DR**
>
> - **4 severidades:** Critical (SLA 15min), High (SLA 1h), Medium (SLA 4h), Low (SLA 24h).
> - **10 alertas catalogados** com diagnóstico, ação imediata e critério de encerramento.
> - **Rollback Railway:** < 1min via dashboard; **Rollback Vercel:** < 30s via CLI/dashboard.
> - **Backup Supabase:** point-in-time restore (PITR) — ponto de recuperação a cada hora.
> - **5 cenários de incidente** com procedimento completo: banco indisponível, Redis down, fila parada, integração externa instável, erro massivo pós-deploy.
> - **Primeiro passo em qualquer incidente:** verificar Railway Logs + `GET /health` + último deploy.
> - **Post-mortem obrigatório** para todo incidente Critical ou High não resolvido em SLA.

---

## 1. Classificação de Severidade

| Severidade | Definição | SLA Resposta | SLA Resolução | Impacto típico | Canal | Escalation |
|---|---|---|---|---|---|---|
| **Critical** | Produção totalmente indisponível ou perda/corrupção de dados financeiros | 15 min | 2h | Todos os usuários afetados. Operação parada. | PagerDuty + Slack `#alertas-producao` | Tech Lead + CTO |
| **High** | Funcionalidade crítica degradada (escrow, auth, fechamento) ou erro 5xx > 5% | 1h | 4h | Parte dos usuários ou fluxo crítico impactado | Slack `#alertas-producao` | Tech Lead |
| **Medium** | Funcionalidade não-crítica com comportamento inesperado. SLA próximo de estourar. | 4h | 24h | Impacto operacional limitado; workaround disponível | Slack `#eng-geral` | Engenheiro responsável |
| **Low** | Bug cosmético, warning recorrente, degradação mínima | 24h | Próximo sprint | Impacto mínimo | GitHub Issue | Nenhum |

---

## 2. Triagem Inicial

Execute este checklist em **toda** abertura de incidente, antes de qualquer ação corretiva:

1. **Confirmar impacto:** quantos usuários? Todas as funcionalidades ou específicas?
2. **Verificar último deploy:** `Railway → Deployments` — algum deploy nas últimas 2h?
3. **Verificar health check:** `curl https://api.repasseseguro.com.br/health`
4. **Abrir Railway Logs:** filtrar por `level:error` e `correlation_id` do alerta
5. **Verificar Sentry:** confirmar se há new issues no contexto do horário do incidente
6. **Checar dependências externas:** Railway status, Supabase status, Upstash status, CloudAMQP status
7. **Decidir contenção:** problema é novo (causa deploy?) ou antigo (causa ambiente/infra)?
8. **Abrir thread no Slack `#alertas-producao`** com: `[INCIDENTE] {descrição} — severidade {X} — investigando`

---

## 3. Catálogo de Alertas

### 3.1 Alta taxa de erros 5xx

**Condição:** `api.error_rate` > 2% por 5min
**Severidade:** Critical
**Sintomas:** Sentry com múltiplos `INT_001`, Railway Logs com `"level":"error"` crescente
**Hipóteses:** deploy com bug, banco indisponível, Redis down, migration mal aplicada

**Diagnóstico:**
```bash
# 1. Verificar health check
curl https://api.repasseseguro.com.br/health

# 2. Verificar último deploy no Railway
# railway logs --tail 100 --environment production

# 3. Buscar no Sentry por erros recentes com filtro "environment=production"
```

**Ação imediata:**
1. Se deploy recente (< 2h): executar rollback imediato (ver Seção 5)
2. Se banco down: ver Cenário de Incidente 5.1
3. Se causa desconhecida: ativar Contingência Manual (desativar agentes IA, modo supervisor total)

**Encerramento:** `GET /health` retornando `ok` + taxa de erros < 0.5% por 10min consecutivos

---

### 3.2 DLQ com mensagens

**Condição:** `rabbitmq.dlq.count` > 0
**Severidade:** High
**Sintomas:** notificações não chegando a destinatários, jobs de distribuição não executando

**Diagnóstico:**
```bash
# Verificar mensagens na DLQ via CloudAMQP dashboard
# Identificar queue de origem e mensagem de erro no payload
```

**Ação imediata:**
1. Acessar CloudAMQP dashboard → DLQ
2. Inspecionar payload e `x-death` headers (histórico de mortes)
3. Se bug de worker: corrigir + redeploy + mover mensagens da DLQ para fila original via management API
4. Se dados inválidos: analisar caso a caso — não processar automaticamente

**Encerramento:** DLQ vazia + workers consumindo normalmente

---

### 3.3 Latência p95 > 1s

**Condição:** `api.request.latency.p95` > 1000ms por 5min
**Severidade:** High

**Diagnóstico:**
```bash
# Verificar queries lentas no Supabase → Database → Query Performance
# Verificar Redis hit rate (< 70% sugere cache miss)
# Verificar Railway CPU (> 85% sugere gargalo de compute)
```

**Ação imediata:**
1. Identificar endpoint mais lento via Sentry Performance
2. Verificar se há query N+1 em execução
3. Verificar índices ausentes no Supabase Query Planner

**Encerramento:** p95 < 500ms por 10min consecutivos

---

### 3.4 Brute force de login

**Condição:** `api.auth.login.failure_rate` > 20 req/min por mesmo IP
**Severidade:** Critical

**Ação imediata:**
1. Identificar IP atacante nos Railway Logs (campo `ip_hash` → correlacionar no Sentry)
2. Bloquear IP via Railway network settings ou Cloudflare (se configurado)
3. Verificar se alguma conta foi comprometida (verificar `locked_until` nos últimos 30min)
4. Notificar Master via Slack `#alertas-seguranca`

**Encerramento:** ataque cessado + IP bloqueado + nenhuma conta comprometida confirmada

---

### 3.5 Falha crítica de agente IA

**Condição:** qualquer evento `FALHA_CRITICA` em `ai_agent_decisions`
**Severidade:** High

**Ação imediata:**
1. Verificar qual agente e qual caso (campo `case_id` + `failure_type` no log)
2. Confirmar que takeover automático foi ativado (Redis `rs:agent:takeover:{agent}:{case_id}`)
3. Notificar Coordenador responsável pelo caso
4. Se `LOOP_DETECTADO` ou `DADOS_CORROMPIDOS`: investigar prompt version + schema de output

**Encerramento:** agente inspecionado + takeover devolvido + causa raiz registrada

---

### 3.6 Serviço externo degradado (ZapSign / Celcoin / Meta)

**Condição:** `EXT_001` > 5% de chamadas em 5min para qualquer serviço
**Severidade:** High

**Ação imediata:**
1. Verificar status page do serviço:
   - ZapSign: `status.zapsign.com.br`
   - Celcoin: página de status Celcoin
   - Meta (WhatsApp): `developers.facebook.com/status`
2. Se incidente no provedor: notificar operadores que ações de ZapSign/Celcoin podem atrasar
3. Se causa interna (auth expirado, token inválido): renovar credencial e monitorar

**Encerramento:** taxa `EXT_001` < 1% por 10min

---

### 3.7 Memória Railway > 90%

**Condição:** `railway.memory_usage` > 90% por 5min
**Severidade:** High

**Ação imediata:**
```bash
# Reiniciar instância (Railway dashboard → Service → Restart)
# Monitorar após restart — se voltar a crescer em < 30min: memory leak
```

**Encerramento:** memória < 75% por 20min. Se leak confirmado: corrigir no próximo deploy.

---

### 3.8 Redis indisponível

**Condição:** `rs:session:*` GET falhando / `ECONNREFUSED` em logs Redis
**Severidade:** Critical

**Impacto:** todas as sessões JWT inválidas (sem blacklist = stateless degradado), cache miss total

**Ação imediata:**
1. Verificar Upstash console — instância ativa?
2. Verificar `REDIS_URL` no Railway environment variables
3. Se Upstash com incidente: ativar modo degradado (ver Seção 8.1)

**Encerramento:** `redis-cli ping` retornando `PONG` + logs sem `ECONNREFUSED`

---

### 3.9 Banco de dados indisponível

**Condição:** `DATABASE_URL` connection refused / Prisma `P1001` nos logs
**Severidade:** Critical

**Ação imediata:**
1. Verificar Supabase dashboard — instância ativa?
2. Verificar status da pool de conexões (Supabase → Database → Connection Pooling)
3. Se connection pool esgotado: identificar queries pendentes + restart do pool

**Encerramento:** `GET /health` com `"database": "ok"`

---

### 3.10 Fila RabbitMQ parada

**Condição:** `rabbitmq.consumer.lag` > 1000 por 10min
**Severidade:** High

**Diagnóstico:**
```bash
# CloudAMQP → Management UI
# Verificar consumers ativos por queue
# Verificar connections abertas
```

**Ação imediata:**
1. Se zero consumers: workers crasharam — verificar Railway Logs do serviço worker
2. Se consumers ativos mas sem progresso: mensagem com bug bloqueando fila → inspecionar
3. Reiniciar workers via Railway se necessário

**Encerramento:** lag < 50 mensagens por 5min

---

## 4. Diagnóstico por Camada

### 4.1 Aplicação (API NestJS)

```bash
# Ver logs em tempo real
railway logs --tail 200 --environment production

# Verificar versão em deploy
curl https://api.repasseseguro.com.br/health | jq .version

# Buscar correlation_id específico nos logs
# Railway Logs → filtrar por correlation_id=uuid
```

### 4.2 Banco de Dados (Supabase PostgreSQL)

```bash
# Via Supabase dashboard:
# Database → Query Performance (queries lentas)
# Database → Connection Pooling (conexões ativas)
# Database → Backups (último backup disponível)

# Via Prisma Studio em staging (NUNCA em produção diretamente):
pnpm --filter api prisma studio
```

### 4.3 Cache (Upstash Redis)

```bash
# Verificar conexão
redis-cli -u $REDIS_URL ping

# Verificar chave específica
redis-cli -u $REDIS_URL get "rs:session:{user_id}"

# Ver tamanho das chaves
redis-cli -u $REDIS_URL dbsize

# Memory usage
redis-cli -u $REDIS_URL info memory
```

### 4.4 Filas (RabbitMQ / CloudAMQP)

```
# Acessar Management UI: https://[host].cloudamqp.com/management
# Queues → verificar Ready + Unacked + Total
# Consumers → verificar count por queue
# DLQ → verificar mensagens + x-death headers
```

### 4.5 Integrações Externas

```bash
# ZapSign — testar connectivity
curl -H "Authorization: Token $ZAPSIGN_API_TOKEN" \
  https://api.zapsign.com.br/api/v1/docs/ -I

# Celcoin — verificar token OAuth
curl https://sandbox.openfinance.celcoin.dev/v5/token \
  -d "client_id=$CELCOIN_CLIENT_ID&client_secret=$CELCOIN_CLIENT_SECRET&grant_type=client_credentials"

# Meta (WhatsApp)
curl "https://graph.facebook.com/v18.0/$META_PHONE_NUMBER_ID?access_token=$META_API_TOKEN"
```

---

## 5. Rollback

### 5.1 Quando Acionar

- `GET /health` retorna status != `200 ok` por > 2min após deploy
- Taxa de erros 5xx > 5% nos primeiros 10min pós-deploy
- Alerta Critical disparado logo após deploy
- Qualquer alerta de dados financeiros corrompidos

### 5.2 Procedimento Railway (API)

1. Acessar `railway.app` → Projeto → Service `api` → Deployments
2. Localizar último deploy estável (anterior ao atual)
3. Clicar `···` → "Rollback to this deployment"
4. Aguardar < 1min
5. Verificar: `curl https://api.repasseseguro.com.br/health`
6. Confirmar versão: `jq .version` deve ser a versão anterior
7. Notificar: `[ROLLBACK] API revertida para v{versão-anterior}. Motivo: {motivo}`

### 5.3 Procedimento Vercel (SPA)

```bash
# Via CLI
vercel rollback [url-do-deploy-anterior] --token=$VERCEL_TOKEN

# Ou via dashboard Vercel:
# Project → Deployments → selecionar deploy anterior → "Promote to Production"
# Tempo: < 30s
```

### 5.4 Atenção com Migrations

> 🔴 **Migrations aplicadas NÃO são desfeitas automaticamente com rollback.** Se a nova versão aplicou uma migration incompatível com a versão anterior do código, o rollback do app sem desfazer a migration pode causar erros. Procedimento: acionar restore de banco (Seção 6) + rollback de app.

---

## 6. Restore e Recuperação de Dados

### 6.1 Backup Supabase (PITR)

Supabase Pro oferece Point-in-Time Recovery com ponto de recuperação a cada hora.

**Quando acionar:** corrupção de dados financeiros (escrow, distribuição), deleção acidental em produção.

**Procedimento:**
1. Identificar timestamp do último estado consistente
2. Acessar Supabase Dashboard → Database → Backups → Point in Time Recovery
3. Selecionar ponto de restauração
4. Confirmar impacto: dados entre ponto de restauração e momento do restore serão perdidos
5. Executar restore (Supabase cria nova instância — atual fica disponível para comparação)
6. Validar integridade: contar registros críticos (casos, transações escrow, distribuições)
7. Atualizar `DATABASE_URL` no Railway para nova instância
8. Notificar stakeholders: "Restauração de dados executada. Operações entre [hora A] e [hora B] podem precisar ser refeitas."

**Tempo estimado:** 15–30min para PITR de banco pequeno.

---

## 7. Cenários de Incidente

### 7.1 Banco Indisponível

**Impacto:** API 100% inoperante. Todos os usuários afetados.

1. Verificar Supabase Status page
2. Se incidente no Supabase: aguardar restauração + comunicar usuários
3. Se conexão perdida internamente: verificar `DATABASE_URL` + pool settings
4. Fallback: nenhum disponível — banco é dependência crítica sem alternativa
5. **Comunicação:** abrir incidente Critical, notificar todos os operadores ativos

### 7.2 Redis Down

**Impacto:** sessões JWT invalidadas (sem blacklist), cache miss, agentes sem state de takeover.

1. API continua funcionando em **modo degradado**: autenticação stateless sem blacklist
2. Logar alerta: "Redis indisponível — tokens revogados via logout podem ser reutilizados"
3. Acionar Upstash suporte ou aguardar auto-recovery
4. Após Redis voltar: usuários não precisam re-logar (tokens válidos ainda funcionam)

### 7.3 Fila RabbitMQ Parada

**Impacto:** notificações não entregues, distribuições escrow pendentes.

1. Identificar se workers estão rodando (Railway Logs)
2. Verificar CloudAMQP — conexão ativa?
3. Reiniciar workers via Railway
4. Verificar DLQ por mensagens acumuladas
5. Processar DLQ manualmente se necessário (mensagens críticas: distribuições escrow)

### 7.4 Integração Externa Instável (ZapSign ou Celcoin)

**Impacto:** formalização parada (ZapSign), distribuições escrow paradas (Celcoin).

1. Verificar status page do serviço
2. Ativar modo manual para o módulo afetado (ver Seção 8)
3. Comunicar Coordenador: "Módulo {X} operando em modo manual temporariamente"
4. Monitorar recuperação — reassumir automático quando estável por 10min

### 7.5 Erro Massivo Pós-Deploy

**Impacto:** deploy introduziu bug que causa erros em escala.

1. Identificar a partir do Railway Logs quando os erros começaram (antes ou após deploy)
2. Se após deploy: **executar rollback imediatamente** (Seção 5.2)
3. Se antes do deploy: investigar causa independente do deploy
4. Após rollback: criar issue no GitHub com `[REGRESSION]` + correlation_ids dos erros
5. Corrigir na branch hotfix + redeploy

---

## 8. Contingência Manual

### 8.1 Modo Degradado (Redis Down)

- API funciona sem blacklist de tokens — logout não invalida token até expiração natural (1h)
- Avisar operadores: "Sessões logout não são invalidadas imediatamente. Evitar logout como mecanismo de segurança até Redis restaurado."
- Duração máxima: 1h (TTL do access token) sem impacto de segurança significativo

### 8.2 Modo Manual (Agentes IA)

Quando o módulo de IA apresentar falhas repetidas:
1. Master ativa "Modo Supervisão Total" em Configurações → IA
2. Todos os agentes ficam com ações em fila aguardando confirmação humana
3. Coordenador gerencia confirmações manualmente
4. Desativar modo quando agentes estabilizados por 30min

### 8.3 Notificações via E-mail Apenas (WhatsApp/Celcoin down)

Quando Meta Cloud API ou Twilio instáveis:
1. Sistema continua enviando notificações apenas por e-mail (canal mais robusto)
2. Coordenador monitora notificações com status `FALLBACK_SMS` ou `FAILED` no log
3. Não há ação manual necessária — e-mail garante entrega das notificações críticas

---

## 9. Checklists Recorrentes

### 9.1 Checklist Diário (Dev/DevOps — 9h)

- [ ] `GET /health` retorna `{ status: "ok" }` em produção
- [ ] Sentry: nenhum novo erro Critical nas últimas 24h
- [ ] DLQ RabbitMQ: 0 mensagens
- [ ] Railway: nenhum alerta de CPU > 85% ou RAM > 90% nas últimas 24h
- [ ] Notificações com status `FAILED` no log: se > 5, investigar

### 9.2 Checklist Semanal (Tech Lead — Segunda)

- [ ] Revisar Sentry: top 5 erros da semana — algum novo ou crescendo?
- [ ] Taxa de falha de notificações por canal: < 2% para e-mail, < 5% para WhatsApp
- [ ] Revisar Redis hit rate: > 85%
- [ ] Revisar p95 latência: < 500ms
- [ ] Verificar error budget SLO: quanto foi consumido na semana?
- [ ] Verificar takeovers de IA da semana: taxa de reversão humana > 30%?

### 9.3 Checklist Pós-Incidente

- [ ] Post-mortem redigido dentro de 48h (se Critical ou High fora do SLA)
- [ ] Root cause documentado no GitHub Issue
- [ ] Action items com owners e prazos definidos
- [ ] Alerta ou threshold ajustado para detectar antes na próxima vez
- [ ] Runbook atualizado se o procedimento foi diferente do documentado

---

## 10. URLs e Ferramentas Críticas

| Recurso | URL / Comando | Uso | Acesso |
|---|---|---|---|
| Railway Dashboard | `railway.app` | Logs, deploy, rollback | Equipe de Eng |
| Vercel Dashboard | `vercel.com` | Deploy SPA, rollback | Equipe de Eng |
| Supabase Dashboard | `supabase.com/dashboard` | Banco, PITR, Auth | Tech Lead + DBA |
| Upstash Console | `console.upstash.com` | Redis stats, flush | Equipe de Eng |
| CloudAMQP Management | `[host].cloudamqp.com/management` | Filas, DLQ, consumers | Equipe de Eng |
| Sentry | `sentry.io` | Error tracking, traces | Equipe de Eng |
| Health Check | `curl https://api.repasseseguro.com.br/health` | Status imediato | Qualquer |
| ZapSign Status | `status.zapsign.com.br` | Status do serviço | Qualquer |

---

## 11. Comunicação de Incidente

### Template de Abertura

```
[INCIDENTE] {título curto} — Severidade: {Critical/High/Medium}
Impacto: {quem está afetado e o que não funciona}
Início: {hora}
Status: Investigando
Responsável: {nome}
Próxima atualização: em {X}min
```

### Template de Atualização (a cada 30min para Critical, 1h para High)

```
[ATUALIZAÇÃO] {título} — {hora}
Status: {Investigando / Contendo / Recuperando}
Descoberta: {o que foi identificado até agora}
Próxima ação: {o que será feito agora}
ETA de resolução: {estimativa}
```

### Template de Encerramento

```
[RESOLVIDO] {título} — {hora de encerramento}
Duração: {X}min
Causa raiz: {resumo}
Resolução: {o que foi feito}
Impacto: {usuários / funcionalidades / dados afetados}
Post-mortem: {link ou "será publicado em 48h"}
```

---

## 12. Template de Post-Mortem

```markdown
# Post-Mortem — {título do incidente}

**Data:** {data}
**Severidade:** Critical / High
**Duração:** {X}min
**Impacto:** {descrição do impacto real}

## Timeline

| Hora | Evento |
|---|---|
| HH:MM | Alerta disparado |
| HH:MM | Engenheiro on-call notificado |
| HH:MM | Causa raiz identificada |
| HH:MM | Ação de contenção aplicada |
| HH:MM | Serviço restaurado |

## Causa Raiz
{descrição técnica precisa da causa raiz}

## Fatores Contribuintes
{o que permitiu que a causa raiz causasse impacto}

## O que funcionou bem
{o que o time fez certo durante a resposta}

## O que poderia ter sido melhor
{o que atrasou a resposta ou ampliou o impacto}

## Ações Corretivas

| Ação | Owner | Prazo | Status |
|---|---|---|---|
| {ação 1} | {nome} | {data} | Aberto |

## Lições Aprendidas
{aprendizados do incidente}
```

---

## 13. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — 4 severidades, 10 alertas, triagem inicial, 5 cenários de incidente, rollback Railway/Vercel, PITR Supabase, contingência manual, checklists recorrentes, template post-mortem. |

---

## 14. Backlog de Pendências

| Item | Marcador | Seção | Justificativa | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Nenhuma pendência aberta | — | — | Todos os alertas e procedimentos derivados dos documentos de input | — | — | Fechado |
