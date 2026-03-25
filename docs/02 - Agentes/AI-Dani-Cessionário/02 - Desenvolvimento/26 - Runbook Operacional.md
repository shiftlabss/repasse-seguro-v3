# 26 - Runbook Operacional
## AI-Dani-Cessionário — Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Produto** | AI-Dani-Cessionário |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Status** | Aprovado — pronto para execução |
| **Bloco** | 6 — Qualidade |

---

> 📌 **TL;DR — Runbook Operacional**
>
> Este documento é o manual de resposta a incidentes do AI-Dani-Cessionário. Cobre 4 severidades, 12 alertas críticos, 6 cenários de incidente e fluxos explícitos de rollback e restore. Toda ação está ordenada, com comando, tempo estimado e validação de encerramento. Em caso de dúvida: (1) isole o escopo, (2) verifique deploy recente, (3) execute diagnóstico por camada, (4) rollback se necessário. O on-call tem autonomia para acionar rollback sem aprovação prévia em P0/P1.

---

## 1. Classificação de Severidade

| **Severidade** | **Definição** | **SLA Resposta** | **SLA Resolução** | **Impacto Típico** | **Canal** | **Escalation** |
|---|---|---|---|---|---|---|
| **P0 — Crítico** | Produção totalmente indisponível ou dado financeiro incorreto sendo servido | 5 minutos | 1 hora | Todos os cessionários afetados; perda financeira potencial | `#dani-incidents` + DM on-call + CEO tech | On-call → Tech Lead → CEO tech em 15min sem resposta |
| **P1 — Alto** | Funcionalidade crítica degradada (chat sem resposta, calculadora indisponível, OTP falha) mas sistema parcialmente operacional | 15 minutos | 4 horas | Maioria dos cessionários afetados; fluxo principal bloqueado | `#dani-incidents` | On-call → Tech Lead em 30min sem resolução |
| **P2 — Médio** | Funcionalidade não-crítica com falha (notificações atrasadas, alertas In-App sem entrega, dashboard lento) | 1 hora | 8 horas úteis | Subgrupo de cessionários afetados; workaround disponível | `#dani-deployments` | On-call → Tech Lead em 2h sem resolução |
| **P3 — Baixo** | Degradação cosmética, log de erro isolado, falha em feature não-core | 4 horas úteis | Próximo sprint | Impacto mínimo; cessionário pode continuar operando | `#dani-dev` | Tech Lead durante horário comercial |

---

## 2. Triagem Inicial

Execute esta sequência a cada incidente, independentemente da severidade reportada.

1. **Confirmar impacto:** "Quantos cessionários estão afetados? Qual funcionalidade?" — verificar Sentry dashboard e Pino logs.
2. **Verificar deploy recente:** `git log --oneline -5` no repositório; verificar Railway "Deployments" — deploy nas últimas 2h = rollback como primeira hipótese.
3. **Checar dependências externas:**
   - OpenAI: `curl https://status.openai.com/api/v2/status.json | jq '.status.indicator'`
   - Supabase: verificar `https://status.supabase.com`
   - Redis: `redis-cli -u $REDIS_URL ping`
   - RabbitMQ: acessar Management UI (seção 12)
4. **Isolar escopo:** "Afeta todos os cessionários ou subgrupo específico?" — se subgrupo, testar com `cessionario_id` isolado.
5. **Consultar dashboards:** Sentry (erros), Langfuse (LLM), RabbitMQ Management (filas), Railway (CPU/memória).
6. **Decidir contenção:** se dados incorretos estão sendo servidos → rollback imediato (P0). Se degradação → diagnosticar antes de rollback.

---

## 3. Catálogo de Alertas

### ALT-001 — Latência p95 Chat > 5s

| **Campo** | **Valor** |
|---|---|
| **Condição** | p95 do endpoint `POST /api/v1/agente/chat` > 5000ms por 3min |
| **Severidade** | P1 |
| **Sintomas** | Usuário vê cursor piscando sem resposta; SSE stream não inicia |
| **Hipóteses** | OpenAI com latência alta; RAG (pgvector) lento; Redis timeout |
| **Comandos de diagnóstico** | `curl -w "@curl-format.txt" -s -o /dev/null https://api.../agente/chat` |
| **Ação imediata** | 1. Verificar status OpenAI. 2. Checar `dani:status:agent` no Redis. 3. Se OpenAI degradado → ativar FallbackBanner (seção 10.1) |
| **Critério de escalação** | Persistir por >15min sem melhora |
| **Condição de encerramento** | p95 < 5000ms por 5min consecutivos |

### ALT-002 — Taxa de Erro HTTP 5xx > 5%

| **Campo** | **Valor** |
|---|---|
| **Condição** | Taxa de respostas 5xx > 5% por 2min |
| **Severidade** | P0 |
| **Sintomas** | Erros `AGENTE_LLM_UNAVAILABLE` ou `INFRA_*` em Sentry; usuários sem resposta |
| **Hipóteses** | Deploy com bug; migração de banco mal-aplicada; memória esgotada no Railway |
| **Comandos de diagnóstico** | `railway logs --service dani-api --tail 100` |
| **Ação imediata** | Rollback imediato se deploy recente (seção 7.3). Se não houver deploy → investigar Railway metrics (CPU/memória) |
| **Critério de escalação** | > 10% por qualquer tempo ou > 5% por >5min |
| **Condição de encerramento** | Taxa 5xx < 1% por 10min |

### ALT-003 — Falha de Isolamento (cessionario_id vazamento)

| **Campo** | **Valor** |
|---|---|
| **Condição** | Log com `WARN` de `CessionarioOwnerGuard` retornando dados de outro `cessionario_id` **OU** alerta manual |
| **Severidade** | P0 — CRÍTICO DE SEGURANÇA |
| **Sintomas** | Cessionário A vê dados do cessionário B; Sentry captura erro de isolamento |
| **Hipóteses** | Bug em query sem filtro `cessionario_id`; middleware desativado; cache cross-tenant |
| **Comandos de diagnóstico** | Verificar Sentry tag `cessionario_id` em erros recentes; revisar logs Pino com `grep "isolamento_falha"` |
| **Ação imediata** | 1. **ROLLBACK IMEDIATO** sem aguardar diagnóstico completo. 2. Abrir incidente P0. 3. Notificar LGPD officer. 4. Investigar post-mortem |
| **Critério de escalação** | Imediato — qualquer ocorrência = P0 automático |
| **Condição de encerramento** | Rollback confirmado + auditoria de logs por 30min sem nova ocorrência |

### ALT-004 — Redis Indisponível

| **Campo** | **Valor** |
|---|---|
| **Condição** | `redis-cli ping` timeout por 30s |
| **Severidade** | P1 |
| **Sintomas** | Rate limit não funciona; cache de calculadora perdido; sessões de agente perdidas |
| **Hipóteses** | Container Redis caído; memory exhausted; rede entre Railway e Redis |
| **Comandos de diagnóstico** | `redis-cli -u $REDIS_URL info memory`; verificar Railway metrics do Redis |
| **Ação imediata** | 1. Reiniciar container Redis. 2. Se memória esgotada: `redis-cli flushdb` apenas no keyspace `dani:cache:*` (não flush total). 3. Ativar degradação controlada (seção 10.2) |
| **Critério de escalação** | Não restaurado em 15min |
| **Condição de encerramento** | `redis-cli ping` retorna `PONG`; cache reconstruído em 5min de uso normal |

### ALT-005 — RabbitMQ Fila Parada (DLQ acúmulo)

| **Campo** | **Valor** |
|---|---|
| **Condição** | Fila `dani.notificacoes.dlq` ou `dani.whatsapp.dlq` com > 50 mensagens |
| **Severidade** | P1 |
| **Sintomas** | Notificações não entregues; alertas In-App atrasados; ZapSign reminders não disparados |
| **Hipóteses** | Consumer com bug de processamento; EvolutionAPI indisponível; schema de mensagem inválido |
| **Comandos de diagnóstico** | Acessar RabbitMQ Management → `dani.notificacoes.dlq` → inspecionar mensagem |
| **Ação imediata** | 1. Inspecionar 1 mensagem da DLQ. 2. Se EvolutionAPI offline: aguardar circuit breaker se fechar. 3. Se bug de consumer: rollback do worker. 4. Reprocessar DLQ após correção |
| **Critério de escalação** | > 200 mensagens em DLQ ou tempo > 2h |
| **Condição de encerramento** | DLQ vazia; consumer processando normalmente |

### ALT-006 — OpenAI Rate Limit ou Quota

| **Campo** | **Valor** |
|---|---|
| **Condição** | HTTP 429 de OpenAI por >5min consecutivos |
| **Severidade** | P1 |
| **Sintomas** | Chat retorna `AGENTE_LLM_RATE_LIMIT`; FallbackBanner exibido para todos |
| **Hipóteses** | Quota mensal esgotada; pico de uso; tier incorreto |
| **Comandos de diagnóstico** | Verificar OpenAI dashboard (`platform.openai.com`) → Usage |
| **Ação imediata** | 1. Ativar FallbackBanner global (seção 10.1). 2. Verificar quota disponível. 3. Reduzir `max_tokens` temporariamente se próximo do limite de RPM |
| **Critério de escalação** | Quota esgotada = P0; upgrade de tier necessário |
| **Condição de encerramento** | Requests sem 429 por 5min |

### ALT-007 — OTP Hard Block Massivo (> 10 usuários em 30min)

| **Campo** | **Valor** |
|---|---|
| **Condição** | > 10 `cessionario_id` distintos com `dani:block:otp:*` em 30min |
| **Severidade** | P1 (possível ataque de força bruta ou bug) |
| **Sintomas** | Pico de `AUTH_OTP_HARD_BLOCK` no Sentry; usuários legítimos sendo bloqueados |
| **Hipóteses** | Ataque coordenado de brute force; bug no contador de falhas; clock skew entre servidores |
| **Comandos de diagnóstico** | `redis-cli --scan --pattern 'dani:block:otp:*' \| wc -l` |
| **Ação imediata** | 1. Verificar IPs de origem (logs Pino). 2. Se ataque: bloquear IPs no WAF/Railway. 3. Se bug: rollback do OTP service |
| **Critério de escalação** | > 50 bloqueios em 30min = P0 |
| **Condição de encerramento** | Taxa de novos bloqueios voltando ao baseline (< 2/hora) |

### ALT-008 — Supabase Indisponível

| **Campo** | **Valor** |
|---|---|
| **Condição** | Queries ao Supabase retornam timeout ou 503 por >1min |
| **Severidade** | P0 |
| **Sintomas** | Todas as operações de leitura/escrita falham; `INFRA_DATABASE_UNAVAILABLE` no Sentry |
| **Hipóteses** | Incidente na plataforma Supabase; rede entre Railway e Supabase; connection pool esgotado |
| **Comandos de diagnóstico** | `curl -f https://[project].supabase.co/rest/v1/`; verificar `https://status.supabase.com` |
| **Ação imediata** | 1. Verificar status page Supabase. 2. Se connection pool: aumentar `SUPABASE_POOL_SIZE` e reiniciar API. 3. Se incidente Supabase: ativar modo somente-leitura se possível (seção 10.3) |
| **Critério de escalação** | > 5min = incidente P0 ativo |
| **Condição de encerramento** | Queries retornando < 200ms por 5min |

### ALT-009 — EvolutionAPI Circuit Breaker Aberto

| **Campo** | **Valor** |
|---|---|
| **Condição** | Circuit breaker de EvolutionAPI em estado "open" (3 falhas consecutivas) |
| **Severidade** | P2 |
| **Sintomas** | OTP WhatsApp não enviado; notificações WhatsApp na DLQ; `WA_CIRCUIT_OPEN` nos logs |
| **Hipóteses** | EvolutionAPI offline; instância WhatsApp Business desconectada; quota de mensagens |
| **Comandos de diagnóstico** | Verificar instância EvolutionAPI Management; testar `POST /message/sendText` com payload mínimo |
| **Ação imediata** | 1. Verificar status EvolutionAPI. 2. Reconectar instância WhatsApp se necessário. 3. Reprocessar DLQ após circuit breaker fechar |
| **Critério de escalação** | > 4h sem resolução = P1 (WhatsApp OTP bloqueado) |
| **Condição de encerramento** | Circuit breaker fechado; teste de envio bem-sucedido |

### ALT-010 — Agente Dani Desativada (Admin)

| **Campo** | **Valor** |
|---|---|
| **Condição** | `dani:status:agent` no Redis = `"disabled"` |
| **Severidade** | P2 (desativação deliberada) ou P1 (desativação acidental) |
| **Sintomas** | Todos os cessionários recebem `AGENTE_DISABLED`; FallbackBanner com mensagem de manutenção |
| **Hipóteses** | Ação administrativa intencional; bug que gravou valor incorreto no Redis |
| **Comandos de diagnóstico** | `redis-cli -u $REDIS_URL GET dani:status:agent` |
| **Ação imediata** | Se intencional: aguardar. Se acidental: `redis-cli SET dani:status:agent "active"` |
| **Critério de escalação** | Se acidental e > 15min sem correção = P1 |
| **Condição de encerramento** | `GET dani:status:agent` retorna `"active"`; smoke test de chat ok |

### ALT-011 — Custo LLM Anormal (> 2x baseline)

| **Campo** | **Valor** |
|---|---|
| **Condição** | Custo OpenAI em 24h > 2x a média dos 7 dias anteriores (Langfuse dashboard) |
| **Severidade** | P2 |
| **Sintomas** | Langfuse alerta de custo; `max_tokens` possivelmente não sendo respeitado |
| **Hipóteses** | Loop de retry excessivo; contexto de conversa crescendo sem limite; abuse de usuário |
| **Comandos de diagnóstico** | Langfuse → Usage → filtrar por `userId` com maior consumo |
| **Ação imediata** | 1. Identificar `cessionario_id` com consumo anormal. 2. Verificar histórico de conversa no Redis. 3. Se necessário: limpar sessão do cessionário |
| **Critério de escalação** | > 3x baseline = P1 |
| **Condição de encerramento** | Custo normalizado por 2h |

### ALT-012 — Health Check Falha Pós-Deploy

| **Campo** | **Valor** |
|---|---|
| **Condição** | `GET /health` retorna não-200 por >3min após deploy |
| **Severidade** | P0 |
| **Sintomas** | Railway mostra deployment "unhealthy"; tráfego mantido na versão anterior |
| **Hipóteses** | Variável de ambiente ausente; migration não aplicada; bug de startup |
| **Comandos de diagnóstico** | `railway logs --service dani-api --tail 50` (erros de startup) |
| **Ação imediata** | Railway faz rollback automático. Se não fizer: `railway rollback --service dani-api` |
| **Critério de escalação** | Rollback automático falha = P0 imediato |
| **Condição de encerramento** | Health check retornando 200; versão anterior ativa |

---

## 4. Diagnóstico por Camada

### 4.1 Aplicação (NestJS API)

```bash
# Logs em tempo real
railway logs --service dani-api --tail 100

# Verificar versão ativa
curl -s https://api.repasse-seguro.com.br/health | jq

# CPU/memória
# Railway Dashboard → dani-api → Metrics

# Erros recentes (últimas 2h)
# Sentry → Issues → filtre por "last 2 hours"
```

**Sinais de problema:** Erros 500 em volume; startup crashes; memory OOM; timeout em todas as requisições.

**Próximo passo:** Se erros pós-deploy → rollback. Se memória → reiniciar serviço. Se timeout externo → verificar dependências.

### 4.2 Banco (Supabase/PostgreSQL)

```bash
# Status da conexão
curl -f "https://[PROJECT_ID].supabase.co/rest/v1/?apikey=$SUPABASE_ANON_KEY"

# Queries lentas (via Supabase Dashboard → Logs → Database)
# Filtrar queries com duration > 1000ms

# Connection pool
# Supabase Dashboard → Database → Connection Pooling → verificar "Active Connections"
```

**Sinais de problema:** Timeout de query; `ECONNREFUSED`; "too many connections".

**Próximo passo:** Aumentar pool se < max. Se query lenta específica → adicionar índice no próximo deploy.

### 4.3 Cache (Redis)

```bash
# Conectividade
redis-cli -u $REDIS_URL ping

# Memória
redis-cli -u $REDIS_URL info memory | grep used_memory_human

# Contagem de chaves por namespace
redis-cli -u $REDIS_URL --scan --pattern 'dani:*' | wc -l
redis-cli -u $REDIS_URL --scan --pattern 'dani:rate:*' | wc -l
redis-cli -u $REDIS_URL --scan --pattern 'dani:block:*' | wc -l

# Verificar se OTP hard block está ativo para um usuário
redis-cli -u $REDIS_URL GET "dani:block:otp:{PHONE_HASH}"

# Verificar status do agente
redis-cli -u $REDIS_URL GET "dani:status:agent"
```

**Sinais de problema:** `PONG` timeout; memória > 80% do limite; chaves de block em volume anormal.

**Próximo passo:** Se memória alta → `FLUSHDB` somente keyspace `dani:cache:*`. Se conectividade → reiniciar container Redis.

### 4.4 Filas (RabbitMQ)

```bash
# Listar filas e profundidade
# RabbitMQ Management UI: http://rabbitmq.repasse-seguro.com.br:15672
# Credenciais: RABBITMQ_USER / RABBITMQ_PASS (de .env)

# Via CLI:
rabbitmqctl list_queues name messages consumers

# Inspecionar mensagem na DLQ (sem consumir)
# RabbitMQ Management → Queue → dani.notificacoes.dlq → Get messages
```

**Sinais de problema:** DLQ > 50 mensagens; consumer count = 0; mensagens acumulando em filas normais.

**Próximo passo:** Consumer morto → reiniciar worker. DLQ com mensagens malformadas → purge após análise.

### 4.5 Integrações Externas

```bash
# OpenAI
curl -s https://status.openai.com/api/v2/status.json | jq '.status.indicator'

# Supabase
curl -s https://status.supabase.com | grep -i "operational"

# EvolutionAPI (instância)
curl -f "http://evolution-api.repasse-seguro.com.br/instance/connectionState/default"
```

### 4.6 Autenticação

```bash
# Verificar se JWT está sendo gerado corretamente
curl -X POST https://api.repasse-seguro.com.br/api/v1/auth/otp/solicitar \
  -H "Content-Type: application/json" \
  -d '{"telefone": "+5511999990001"}'
# Esperado: 200 (não 500)

# Verificar se JWTs revogados estão no Redis
redis-cli -u $REDIS_URL --scan --pattern 'dani:revoked:refresh:*' | wc -l
```

### 4.7 Infraestrutura

```bash
# Railway — status do serviço
railway status --service dani-api

# Vercel — status do deploy
vercel ls --prod

# Verificar variáveis de ambiente críticas (sem expor valores)
railway variables --service dani-api | grep -E "DATABASE_URL|OPENAI|REDIS" | awk -F= '{print $1}'
```

---

## 5. Rollback

Ver procedimento completo em D24 — Deploy, CI-CD e Versionamento, Seção 7.

### 5.1 Critério de Acionamento

Acionar rollback IMEDIATAMENTE quando:
- ALT-002: Taxa 5xx > 5% por 2min.
- ALT-003: Qualquer suspeita de falha de isolamento.
- ALT-012: Health check falha pós-deploy.

Diagnosticar antes de rollback quando:
- Falha em dependência externa (OpenAI, EvolutionAPI) — rollback não resolve.
- Degradação de performance sem erros 5xx.

### 5.2 Comandos Rápidos

```bash
# Rollback API (Railway)
railway rollback --service dani-api

# Rollback Frontend (Vercel)
vercel rollback https://app.repasse-seguro.com.br

# Verificar versão ativa após rollback
curl -s https://api.repasse-seguro.com.br/health | jq '.version'
```

**Tempo estimado:** API 2–3min; Frontend < 2min.

---

## 6. Restore e Recuperação de Dados

### 6.1 Backups Disponíveis

| **Tipo** | **Frequência** | **Retenção** | **Responsável** | **Observação** |
|---|---|---|---|---|
| Supabase Point-in-Time Recovery (PITR) | Contínuo (WAL) | 7 dias (free) / 30 dias (Pro) | Supabase (automático) | Disponível no dashboard Supabase |
| Supabase Scheduled Backup | Diário | 7 backups | Supabase (automático) | Snapshot completo |
| Redis snapshot (RDB) | A cada 15min | 3 snapshots | Railway (automático) | Estado do cache, não dados críticos |

**[DECISÃO AUTÔNOMA]:** Não há dados críticos exclusivos no Redis — toda informação de negócio está no Supabase | Justificativa: Redis contém apenas cache, rate limit e sessões temporárias; perda de Redis = reprocessamento sem perda de dados | Alternativa descartada: Redis como fonte de verdade — viola princípio de isolamento definido em D02.

### 6.2 Restore do Supabase

**Quando usar:** perda de dados por bug de aplicação, exclusão acidental, corrupção.

```bash
# 1. Acessar Supabase Dashboard → projeto → Database → Backups
# 2. Selecionar ponto de restauração (PITR ou snapshot)
# 3. Clicar "Restore to this point"
# Tempo estimado: 10–30 minutos dependendo do tamanho

# 4. Após restore, verificar integridade:
# Contar registros em tabelas críticas
psql $DATABASE_URL -c "SELECT COUNT(*) FROM oportunidades;"
psql $DATABASE_URL -c "SELECT COUNT(*) FROM sessoes;"
psql $DATABASE_URL -c "SELECT COUNT(*) FROM alertas;"

# 5. Verificar que nenhum cessionario_id está com dados faltando
psql $DATABASE_URL -c "SELECT cessionario_id, COUNT(*) FROM oportunidades GROUP BY 1 ORDER BY 2;"
```

### 6.3 Comunicação a Stakeholders em Caso de Restore

```
🔴 AVISO OPERACIONAL — AI-Dani-Cessionário

Tipo: Recuperação de dados
Horário: {HH:MM}
Escopo: {DESCREVER: ex. "oportunidades criadas entre 14h e 15h30 de 23/03"}
Impacto: {DESCREVER: ex. "dados podem estar desatualizados; revalidar informações"}
Status: Em andamento / Concluído
ETA: {HORÁRIO DE CONCLUSÃO}

Responsável: {NOME}
```

### 6.4 Validação de Integridade Pós-Restore

1. Confirmar contagem de registros vs. backup anterior.
2. Testar fluxo de cálculo de comissão com oportunidade de produção.
3. Confirmar que isolamento por `cessionario_id` está íntegro.
4. Confirmar que sessões ativas não têm tokens inválidos.
5. Monitorar Sentry por 30min — zero erros de schema.

---

## 7. Cenários de Incidente

### INC-001 — Indisponibilidade Total do Banco

**Sintoma:** Todos os endpoints retornam 503; `INFRA_DATABASE_UNAVAILABLE` em volume.

**Triagem:**
1. Verificar status Supabase (`https://status.supabase.com`).
2. Testar conexão direta: `psql $DATABASE_URL -c "SELECT 1"`.
3. Verificar connection pool no Dashboard Supabase.

**Contenção:**
- Se incidente Supabase: aguardar resolução; não há alternativa (Supabase é a única DB).
- Ativar página de manutenção no frontend: `vercel env add NEXT_PUBLIC_MAINTENANCE=true`.
- Publicar alerta no Slack `#dani-incidents` e comunicar cessionários via painel de status.

**Resolução:** restauração pelo Supabase ou restore manual (seção 6.2).

**Encerramento:** `GET /health` retornando 200 com `database: "ok"` por 5min.

---

### INC-002 — Falha de Redis (Cache/Session)

**Sintoma:** Sessões do agente perdidas; rate limit não funciona; `INFRA_REDIS_UNAVAILABLE`.

**Triagem:**
1. `redis-cli -u $REDIS_URL ping`.
2. Verificar Railway metrics do Redis (OOM?).
3. Verificar container Railway.

**Contenção:**
- Reiniciar container Redis: Railway Dashboard → Redis → Restart.
- Se OOM: `redis-cli flushdb` apenas namespace `dani:cache:*` (preservar `dani:block:*` e `dani:rate:*` se possível).
- Com Redis indisponível: rate limit desativado temporariamente → monitorar abuse manual.

**Resolução:** Redis reiniciado e respondendo `PONG`.

**Encerramento:** Sessões reconstruídas naturalmente com uso; cache miss aceitável por 5min.

---

### INC-003 — Fila RabbitMQ Parada

**Sintoma:** Notificações não entregues; DLQ acumulando mensagens.

**Triagem:**
1. RabbitMQ Management → verificar consumer count em `dani.notificacoes`.
2. Inspecionar 1 mensagem da DLQ: é erro de processamento ou de schema?
3. Verificar logs do worker de notificações.

**Contenção:**
- Se consumer morto: reiniciar worker (`railway restart --service dani-worker`).
- Se EvolutionAPI offline: aguardar circuit breaker fechar automaticamente.
- Notificações críticas (ZapSign, Escrow) em DLQ: processar manualmente após resolução.

**Resolução:** Consumer ativo; `dani.notificacoes.dlq` retornando ao baseline (< 5 mensagens).

**Encerramento:** Consumer count ≥1; mensagens sendo processadas; DLQ vazia.

---

### INC-004 — Integração Externa Instável (OpenAI Degradado)

**Sintoma:** Chat lento (p95 > 5s) ou sem resposta; `AGENTE_LLM_UNAVAILABLE` em Sentry.

**Triagem:**
1. `curl -s https://status.openai.com/api/v2/status.json | jq '.status.indicator'` — se não `"none"`, incidente no lado OpenAI.
2. Verificar Langfuse — latência média dos últimos 30min.
3. Verificar se retry está em loop (logs `AGENTE_LLM_RETRY_N`).

**Contenção:**
- Ativar FallbackBanner global: `redis-cli SET dani:status:agent "degraded"`.
- Exibir mensagem "Dani está temporariamente indisponível. Use a Calculadora abaixo." (TPLF-009).
- Calculadora permanece funcional — cessionários podem continuar simulações.

**Resolução:** OpenAI normalizado; `redis-cli SET dani:status:agent "active"`.

**Encerramento:** p95 < 5s por 5min; FallbackBanner desativado.

---

### INC-005 — Erro Massivo Pós-Deploy

**Sintoma:** Taxa 5xx > 5% surgindo imediatamente após deploy; correlação temporal clara.

**Triagem:**
1. Confirmar timestamp do deploy vs. início dos erros.
2. Verificar logs de startup da nova versão.
3. Identificar padrão do erro (mesmo `error_code`? mesma rota?).

**Contenção:**
- **Rollback imediato** (não aguardar diagnóstico completo):
  ```bash
  railway rollback --service dani-api
  vercel rollback https://app.repasse-seguro.com.br
  ```
- Smoke tests após rollback (seção 5.3 do D24).

**Resolução:** Rollback confirmado; versão anterior ativa; 5xx retornando ao baseline.

**Encerramento:** Taxa 5xx < 0.5% por 10min; smoke tests verdes.

---

### INC-006 — Quota ou Rate Limit Operacional (OpenAI/EvolutionAPI)

**Sintoma:** HTTP 429 contínuo para todos os usuários; `RATE_*` em volume.

**Triagem:**
1. OpenAI: verificar dashboard de uso (`platform.openai.com` → Usage).
2. EvolutionAPI: verificar quota de mensagens WhatsApp Business.

**Contenção:**
- OpenAI rate limit (RPM): reduzir `max_tokens` temporariamente de 2048 para 1024: `redis-cli SET dani:config:max_tokens 1024`.
- OpenAI quota mensal esgotada: upgrade imediato de tier — P0.
- EvolutionAPI quota WhatsApp: aguardar reset (normalmente 24h) ou provisionar nova instância.

**Resolução:** Quota ampliada ou reset natural.

**Encerramento:** Requisições sem 429 por 5min; latência normalizada.

---

## 8. Contingência Manual

### 8.1 FallbackBanner Global (LLM Indisponível)

**Quando ativar:** ALT-006 (OpenAI quota/rate), ALT-001 persistente por >30min, manutenção programada do LLM.

```bash
# Ativar degradação controlada
redis-cli -u $REDIS_URL SET dani:status:agent "degraded" EX 3600
# EX 3600 = expira em 1 hora (renovar se necessário)

# Desativar após resolução
redis-cli -u $REDIS_URL SET dani:status:agent "active"
```

**Risco operacional remanescente:** Cessionários sem acesso ao chat. Calculadora permanece disponível. Dashboard de oportunidades permanece disponível.

**Duração máxima:** 4h antes de escalar para P0.

### 8.2 Desativar Agente Completamente

**Quando ativar:** Falha de segurança confirmada; dados incorretos sendo servidos pelo LLM; manutenção emergencial.

```bash
redis-cli -u $REDIS_URL SET dani:status:agent "disabled"
```

Todos os cessionários verão mensagem de indisponibilidade (TPLF-009).

### 8.3 Modo Somente-Leitura (Banco Degradado)

**[DECISÃO AUTÔNOMA]:** Se Supabase tiver indisponibilidade de escrita mas leitura funcionar, ativar banner "modo somente leitura" — cessionários podem consultar oportunidades mas não acionar negociações | Alternativa descartada: desativar completamente — perda de valor maior do que degradação parcial.

**Quando ativar:** Supabase em modo read-only por incidente da plataforma.

**Duração máxima:** 2h antes de comunicar cessionários formalmente.

---

## 9. Checklists Recorrentes

### 9.1 Checklist Diário (On-Call — 09h e 17h)

```
[ ] Sentry: zero erros P0/P1 abertos
[ ] Langfuse: custo LLM nas últimas 24h dentro de 1.5x baseline
[ ] RabbitMQ: DLQ de dani.notificacoes e dani.whatsapp < 10 mensagens
[ ] Redis: memória < 80% do limite
[ ] Railway: CPU/memória API dentro do normal (< 70%)
[ ] Supabase: connection pool < 80% de uso
[ ] Health check manual: curl https://api.repasse-seguro.com.br/health
```

### 9.2 Checklist Semanal (Tech Lead — toda segunda-feira)

```
[ ] Revisar todas as entradas da DLQ da semana anterior — padrão?
[ ] Revisar top 10 erros do Sentry — algum novo ou crescendo?
[ ] Verificar custo OpenAI semanal vs. orçamento
[ ] Confirmar que backups Supabase estão sendo gerados (Dashboard → Backups)
[ ] Revisar alertas de acessibilidade da última semana
[ ] Verificar se há dependências desatualizadas com CVE: pnpm audit
[ ] Confirmar que logs de auditoria (LGPD) estão com retenção correta (1 ano)
[ ] Revisar e fechar incidentes P2/P3 abertos há > 7 dias
```

### 9.3 Checklist Pós-Incidente (até 48h após resolução)

```
[ ] Preencher template de post-mortem (seção 14)
[ ] Identificar 1 ação corretiva imediata (máx 3 dias para implementar)
[ ] Identificar 1 ação preventiva de longo prazo (máx 2 sprints)
[ ] Atualizar este Runbook se o incidente revelou gap
[ ] Atualizar D27 (Plano de Testes) se o bug não estava coberto por testes
[ ] Comunicar stakeholders com resumo do incidente
[ ] Criar issue no GitHub com label "post-mortem" e linkar ao incidente
```

---

## 10. URLs e Ferramentas Críticas

| **Recurso** | **URL / Comando** | **Uso** | **Credencial** | **Observação** |
|---|---|---|---|---|
| Railway Dashboard | `https://railway.app/project/{ID}` | Deploy, logs, rollback, métricas | Railway token | [PENDÊNCIA: definir PROJECT_ID de produção] |
| Vercel Dashboard | `https://vercel.com/team/{TEAM}/dani` | Deploy frontend, rollback, preview | Vercel token | — |
| Supabase Dashboard | `https://app.supabase.com/project/{ID}` | Banco, backups, query | Supabase access token | — |
| RabbitMQ Management | `https://rabbitmq.repasse-seguro.com.br:15672` | Filas, DLQ, consumers | `RABBITMQ_USER` / `RABBITMQ_PASS` | — |
| Sentry | `https://sentry.io/organizations/{ORG}/issues/` | Erros, alertas, performance | Sentry auth token | — |
| Langfuse | `https://cloud.langfuse.com/project/{ID}` | LLM traces, custo, latência | Langfuse API key | — |
| OpenAI Platform | `https://platform.openai.com/usage` | Quota, custo, rate limit | OpenAI account | — |
| Health Check | `GET https://api.repasse-seguro.com.br/health` | Validação rápida de status | Nenhuma | Retorna versão e status de dependências |
| Railway CLI rollback | `railway rollback --service dani-api` | Rollback de API | `RAILWAY_TOKEN` | Instalar: `npm i -g @railway/cli` |
| Vercel CLI rollback | `vercel rollback https://app.repasse-seguro.com.br` | Rollback de frontend | `VERCEL_TOKEN` | Instalar: `npm i -g vercel` |
| Redis CLI | `redis-cli -u $REDIS_URL` | Diagnóstico de cache, rate limit | `REDIS_URL` do .env | — |

---

## 11. Comunicação de Incidente

### 11.1 Atualização Interna

**Frequência mínima:**
- P0: a cada 15 minutos até resolução.
- P1: a cada 30 minutos.
- P2/P3: a cada 2h.

**Template de atualização:**

```
[ATUALIZAÇÃO — HH:MM]
Incidente: {TÍTULO}
Severidade: P{N}
Status: Em investigação / Contenção ativa / Resolvido
Impacto atual: {DESCREVER}
Última ação executada: {DESCREVER}
Próxima ação: {DESCREVER}
ETA de resolução: {HORÁRIO ou "Em investigação"}
Responsável: {NOME}
```

### 11.2 Donos por Severidade

| **Severidade** | **Comunicador** | **Canal** | **Frequência** |
|---|---|---|---|
| P0 | On-call | `#dani-incidents` + DM CEO tech | 15min |
| P1 | On-call | `#dani-incidents` | 30min |
| P2 | On-call ou Tech Lead | `#dani-deployments` | 2h |
| P3 | Dev responsável | `#dani-dev` | 1x por dia útil |

### 11.3 Mensagem de Encerramento

```
✅ INCIDENTE ENCERRADO — {TÍTULO}

Horário de abertura: {HH:MM}
Horário de encerramento: {HH:MM}
Duração total: {N} minutos
Causa raiz: {SUMÁRIO EM 1 FRASE}
Resolução aplicada: {ROLLBACK / CORREÇÃO / AJUSTE DE CONFIG}
Impacto confirmado: {N cessionários afetados por X minutos}
Post-mortem: {URL do documento ou "Agendado para {DATA}"}

Responsável: {NOME}
```

---

## 12. Template de Post-Mortem

```markdown
# Post-Mortem — {TÍTULO DO INCIDENTE}

**Data do incidente:** {DATA}
**Severidade:** P{N}
**Duração:** {N} minutos
**Autor:** {NOME}
**Status:** Rascunho / Em revisão / Aprovado

## Resumo Executivo
{2-3 frases descrevendo o incidente, impacto e resolução}

## Timeline

| **Horário** | **Evento** |
|---|---|
| HH:MM | Alerta disparado / primeiro relato |
| HH:MM | On-call notificado |
| HH:MM | Diagnóstico concluído |
| HH:MM | Contenção aplicada |
| HH:MM | Resolução confirmada |
| HH:MM | Incidente encerrado |

## Causa Raiz
{DESCRIÇÃO TÉCNICA DETALHADA}
{Por que aconteceu? Por que não foi detectado antes?}

## Fator Contribuinte
{O que tornou o impacto pior ou a detecção mais lenta?}

## Mitigação Aplicada
{O que foi feito para resolver imediatamente}

## Impacto
- Cessionários afetados: {N}
- Funcionalidade afetada: {NOME}
- Duração de impacto: {N} minutos
- Dados afetados: {SIM/NÃO — DESCREVER}

## Ações Corretivas

| **Ação** | **Owner** | **Prazo** | **Status** |
|---|---|---|---|
| {AÇÃO IMEDIATA} | {NOME} | {DATA} | Pendente |
| {AÇÃO PREVENTIVA} | {NOME} | {DATA} | Pendente |

## Lições Aprendidas
{O que funcionou bem na resposta? O que poderia ter sido mais rápido?}

## Referências
- Sentry issue: {URL}
- GitHub issue: {URL}
- Logs relevantes: {Comandos ou links}
```

---

## 13. Backlog de Pendências

| **ID** | **Tipo** | **Descrição** | **Impacto Operacional** |
|---|---|---|---|
| P-01 | PENDÊNCIA | Canais Slack (`#dani-incidents`, `#dani-deployments`, `#dani-releases`, `#dani-dev`) precisam ser criados antes do go-live | Comunicação de incidente não pode ocorrer sem canais estabelecidos |
| P-02 | PENDÊNCIA | On-call rotation não definida — quem é o on-call e qual ferramenta (PagerDuty, OpsGenie, rotação manual)? | SLA P0 de 5min inatingível sem on-call definido |
| P-03 | [DECISÃO AUTÔNOMA] | Página de status pública (ex: Statuspage.io) não prevista em D02 — omitida do runbook por ausência de source | Impacto em transparência com cessionários durante incidentes |
| P-04 | PENDÊNCIA | URL de produção final não confirmada — `repasse-seguro.com.br` usado como placeholder | Todos os comandos com URL de produção devem ser atualizados |
| P-05 | [DECISÃO AUTÔNOMA] | Retenção de logs de auditoria definida em 1 ano (D25) — Justificativa: padrão LGPD para dados de acesso | Confirmar com equipe jurídica antes do go-live |

---

*Documento gerado por Claude Code Desktop — Pipeline ShiftLabs v9.5 — AI-Dani-Cessionário*
