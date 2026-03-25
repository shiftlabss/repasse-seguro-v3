# 29 - Go-Live Playbook

## Cabeçalho

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
| --- | --- | --- | --- | --- |
| 29 - Go-Live Playbook | v1.0 | 2026-03-22 | Claude Code Desktop (ShiftLabs Pipeline v9.5) | Aprovado |

---

## TL;DR

> 📌 **Playbook de lançamento do Repasse AI:**
>
> - **Janelas de preparação:** T-7 (infraestrutura e integrações), T-3 (QA e testes E2E), T-1 (freeze, smoke tests, war room setup).
> - **Launch Day:** Terça ou Quinta, 14h BRT — deploy via Railway + health checks automáticos + go/no-go explícito em T+15min.
> - **Go/no-go baseado em:** `http_error_rate < 2%` + `GET /health` 200 + circuit breaker CLOSED + sem CRITICAL no Sentry.
> - **No-go automático:** rollback Railway em < 3 min se qualquer threshold cruzado nos primeiros 15 min.
> - **War room:** Slack #repasse-ai-launch (criado T-1), encerrado em T+1h se tudo estável.
> - **Estabilização:** monitoramento intensivo T+15min, T+1h, T+24h com critérios de normalização definidos.
> - **Comunicação:** templates internos por momento + canal externo para cessionários via plataforma Repasse Seguro.

---

## 1. Matriz de Escalation

| Severidade | Impacto | Canal | Pessoas Acionadas | SLA Resposta | Critério de Escalação |
| --- | --- | --- | --- | --- | --- |
| **CRITICAL (P0)** | Serviço totalmente indisponível | PagerDuty + Slack #repasse-ai-launch | Tech Lead + DRI do deploy + on-call backend | 15 min | `http_error_rate > 10%` OU `GET /health` 503 > 30s |
| **HIGH (P1)** | Funcionalidade crítica degradada | Slack #repasse-ai-launch + DM on-call | On-call backend; Tech Lead notificado | 30 min | Circuit breaker OPEN OU `llm_error_rate > 10%` |
| **MEDIUM (P2)** | Funcionalidade degradada com workaround | Slack #repasse-ai-launch | Developer responsável | 1 hora | `notification_delivery_rate < 80%` OU DLQ > 10 |
| **LOW (P3)** | Anomalia sem impacto imediato | Slack #repasse-ai-dev | Developer que detectou | Próximo dia útil | Métricas fora de target mas abaixo de thresholds P2 |

---

## 2. Health Checks

| Sinal | Endpoint / Comando | Frequência | Responsável | Threshold OK | Ação em Falha |
| --- | --- | --- | --- | --- | --- |
| Serviço ativo | `GET /health` | A cada 5 min (T+15min, T+1h) | DRI do deploy | 200 com `status` ≠ `down` | Acionar P0 imediatamente |
| Error rate | Railway Metrics / Sentry `http_error_rate` | Contínuo — alertas automáticos | GitHub Actions / DRI | < 2% nos primeiros 15 min | No-go ou rollback se > 5% |
| LLM disponível | Langfuse traces + `/health.openai` | A cada 5 min | DRI | `circuit_breaker_status: CLOSED` | Acionar CB-001 (Runbook) |
| Fila | RabbitMQ Management → queues | A cada 10 min | DRI | `queue_depth < 50`, `dlq_count = 0` | Acionar NOTIF-001 |
| Custo OpenAI | Langfuse → Cost by trace | A cada hora | Tech Lead | < $10/hora nos primeiros 6h | Verificar volume anômalo |
| Banco | `db_connection_pool_usage` | Contínuo | Railway Metrics | < 70% | Acionar DB-001 (Runbook) |
| Sentry — erros novos | Sentry dashboard | Monitoramento manual T+15min, T+1h | DRI | 0 erros CRITICAL inexistentes antes do deploy | Acionar P0 se CRITICAL novo |

---

## 3. Pré-Launch

### 3.1 T-7 (7 dias antes do go-live)

**Dono:** Tech Lead

- [ ] Todas as integrações P0/P1 testadas em staging com dados reais mascarados:
  - [ ] OpenAI: mensagem de teste retorna resposta em < 8s
  - [ ] Supabase: queries com `withTenant()` funcionando corretamente
  - [ ] Upstash Redis: rate limiting e OTP testados
  - [ ] EvolutionAPI: webhook recebido e validado com HMAC correto
  - [ ] RabbitMQ: notificação publicada e entregue via worker
- [ ] Todos os secrets de produção configurados no Railway Secrets (não staging):
  - [ ] `OPENAI_API_KEY`
  - [ ] `DATABASE_URL` (produção Supabase)
  - [ ] `REDIS_URL` (Upstash produção)
  - [ ] `RABBITMQ_URL` (CloudAMQP produção)
  - [ ] `EVOLUTION_API_URL` e `EVOLUTION_API_KEY`
  - [ ] `EVOLUTION_API_WEBHOOK_SECRET`
  - [ ] `JWT_SECRET` (Supabase)
  - [ ] `LANGFUSE_PUBLIC_KEY` e `LANGFUSE_SECRET_KEY`
  - [ ] `SENTRY_DSN`
- [ ] `FEATURE_WHATSAPP_MOCK=false` em produção
- [ ] `JWT_DEV_MODE=false` em produção
- [ ] `LOG_LEVEL=info` em produção
- [ ] Backup Supabase do ambiente de staging verificado (valida que backups estão funcionando)
- [ ] Domínio `api.repasseseguro.com.br` apontando para Railway produção

### 3.2 T-3 (3 dias antes do go-live)

**Dono:** Tech Lead + QA Lead

- [ ] Testes E2E completos passando em staging (8 fluxos críticos — Doc 27):
  - [ ] E2E-001: Mensagem Webchat → Agente IA → Resposta SSE
  - [ ] E2E-002: WhatsApp OTP → Binding → ACTIVE
  - [ ] E2E-003: Admin Takeover
  - [ ] E2E-004: Isolamento de tenant (TENANT_MISMATCH)
  - [ ] E2E-005: Rate limit
  - [ ] E2E-006: Circuit breaker
  - [ ] E2E-007: Notificação push após resposta do agente
  - [ ] E2E-008: Webhook HMAC inválido → rejeição
- [ ] Checklist de Qualidade (Doc 28) passando — todos os 18 itens bloqueantes verificados
- [ ] Smoke tests automáticos passando em staging (`pnpm test:smoke`)
- [ ] Seeds de produção preparados (`AgentConfiguration` — 9 configs do Doc 13)
- [ ] Doc 22 (Setup Local) revisado — qualquer dev consegue replicar o ambiente
- [ ] War room Slack #repasse-ai-launch criado com participantes confirmados
- [ ] Rollback Railway testado em staging (executar rollback manual e verificar saúde)
- [ ] `pnpm audit --audit-level high` → 0 vulnerabilidades HIGH/CRITICAL

### 3.3 T-1 (1 dia antes do go-live)

**Dono:** Tech Lead

- [ ] **Code freeze decretado** — nenhum PR mergeado em `main` após este ponto sem aprovação explícita do Tech Lead
- [ ] Smoke tests passando em produção atual (endpoint `/health` verificado)
- [ ] Último deployment de staging promovido para produção (sem alterações pendentes)
- [ ] Migrations Prisma executadas em produção: `railway run pnpm prisma migrate deploy`
- [ ] Seeds de produção executados: `railway run pnpm prisma db seed`
- [ ] Sentry configurado e recebendo eventos de produção (`curl` de teste)
- [ ] Langfuse configurado e recebendo traces de produção (trace de teste manual)
- [ ] PagerDuty integration ativa — testar alerta de teste
- [ ] Notificar stakeholders sobre go-live: data, hora, o que muda
- [ ] Identificar DRI do deploy (não pode mudar no dia)
- [ ] Verificar horário de deploy na janela Terça/Quinta 14h-16h BRT
- [ ] Verificar disponibilidade dos responsáveis de escalation (P0 e P1)

---

## 4. Launch Day

**DRI do Deploy:** definido no T-1

### 4.1 Checklist Cronológico

| Horário (BRT) | Ação | Responsável | Critério de Sucesso |
| --- | --- | --- | --- |
| 13h45 | Verificar `/health` em staging — tudo verde? | DRI | 200 com todos os serviços `up` |
| 13h50 | Abrir war room no Slack #repasse-ai-launch | Tech Lead | Canal ativo com participantes online |
| 13h55 | Comunicar início do deploy: "Deploy v1.x.x iniciado — 14h00" | Tech Lead | Mensagem enviada ao #deploys |
| **14h00** | **Merge PR `staging → main` — deploy automático Railway** | DRI | CI passa + Railway deploy iniciado |
| 14h03 | Acompanhar Railway dashboard — deployment status | DRI | Status "Active" sem erros |
| 14h05 | **Ponto de Go/No-Go 1:** `GET /health` → 200? | DRI | Todos os serviços `up` ou `degraded` (não `down`) |
| 14h05 | **Ponto de Go/No-Go 1:** `http_error_rate < 2%` nos primeiros 5 min? | DRI | Sentry + Railway Metrics |
| 14h10 | Enviar mensagem de teste manual (Webchat) | QA Lead | Agente responde em < 10s |
| 14h15 | **Ponto de Go/No-Go 2 (final):** todos os health checks OK? | Tech Lead | Ver seção 5 (Critério de Go/No-Go) |
| 14h15 | Se Go: "Deploy v1.x.x concluído ✅ — em estabilização" | Tech Lead | Mensagem no #deploys |
| 14h15 | Se No-Go: iniciar rollback imediatamente | DRI | Ver seção 6 (Rollback) |
| 14h15 | Tag Git criada automaticamente pelo CD workflow | GitHub Actions | `v1.x.x` visível no GitHub |
| 14h30 | **T+30min:** verificar métricas de estabilização | DRI | Ver seção 5 (Pós-Launch) |
| 15h00 | **T+1h:** verificar normalização — encerrar war room se estável | Tech Lead | Ver critério de normalização |

---

## 5. Pós-Launch

### 5.1 T+15min

**Dono:** DRI do deploy

- [ ] `GET /health` retorna 200 com `status: healthy` ou `degraded`
- [ ] `http_error_rate < 2%`
- [ ] `circuit_breaker_open_count = 0`
- [ ] `rabbitmq_dlq_count = 0`
- [ ] Sentry: 0 novos erros CRITICAL inexistentes antes do deploy
- [ ] Langfuse: traces sendo gerados corretamente
- [ ] Pelo menos 3 mensagens de teste respondidas com sucesso (QA manual)

Se qualquer item falhar → acionar escalation conforme matriz (seção 1).

### 5.2 T+1h

**Dono:** Tech Lead

- [ ] Todos os itens do T+15min ainda passando
- [ ] `semantic_cache_hit_rate` > 0% (cache começando a ser preenchido)
- [ ] `notification_delivery_rate` > 90%
- [ ] Custo OpenAI na primeira hora dentro do esperado (< $10)
- [ ] `db_connection_pool_usage < 60%`
- [ ] War room pode ser encerrado se todos os itens passando: "War room encerrado — sistema estável ✅"
- [ ] Changelog atualizado no repositório
- [ ] Release Notes publicadas (ver template no Doc 24)

### 5.3 T+24h

**Dono:** Tech Lead (revisão no dia seguinte)

- [ ] Revisar checklist diário do Doc 26 (Runbook)
- [ ] Sentry: revisar todos os erros HIGH/CRITICAL do dia
- [ ] Langfuse: verificar latência p95 do agente (target < 8s) e custo do dia (target < $50)
- [ ] `semantic_cache_hit_rate` crescendo (target > 10% após 24h)
- [ ] Nenhum ticket de suporte com problema crítico
- [ ] Post-mortem iniciado se houve qualquer P0/P1 durante o go-live

### 5.4 Critério de Normalização

O go-live é considerado estável quando, por 1 hora consecutiva:
- `http_error_rate < 1%`
- `circuit_breaker_open_count = 0`
- `llm_error_rate < 2%`
- `rabbitmq_dlq_count = 0`
- Sem erros CRITICAL novos no Sentry

---

## 6. Deploy e Rollback

### 6.1 Deploy Produção

```bash
# Passo 1: CI/CD automático via PR merge
# Merge em `main` → Railway CD automático

# Verificar deploy
railway status --service repasse-ai

# Health check manual
curl -s https://api.repasseseguro.com.br/health | jq .
```

### 6.2 Critérios de No-Go / Rollback Automático

Iniciar rollback **imediatamente** se qualquer condição abaixo ocorrer nos primeiros 15 minutos:

| Condição | Threshold | Ação |
| --- | --- | --- |
| `http_error_rate` | > 10% por > 3 min | Rollback automático |
| `GET /health` | 503 por > 30s | Rollback automático |
| Circuit breaker | OPEN nos primeiros 5 min | Rollback (se não era problema pré-existente) |
| Erro CRITICAL novo | ≥ 1 no Sentry (inexistente antes do deploy) | Avaliar + rollback se root cause for o deploy |
| `http_error_rate` | > 2% por > 5 min | No-go manual — Tech Lead decide |

### 6.3 Procedimento de Rollback

```bash
# Via Railway CLI
railway login
railway deployments --service repasse-ai
# Copiar ID do deployment anterior (pré-defeito)
railway redeploy --deployment <DEPLOYMENT_ID_ANTERIOR>

# Aguardar ~3 min
watch -n 5 "curl -s https://api.repasseseguro.com.br/health | jq '.status'"

# Verificar
curl -s https://api.repasseseguro.com.br/health | jq .
```

**Após rollback:**
1. Notificar #deploys: "⚠️ Rollback executado para v1.x.x. Causa: [motivo]. Investigando."
2. Abrir post-mortem (mesmo que breve) no Notion
3. Não re-tentar deploy sem corrigir a causa raiz e passar pelo CI completo novamente

---

## 7. Incidentes por Categoria

| Categoria | Sintoma | Triagem Imediata | Ação Inicial | Runbook |
| --- | --- | --- | --- | --- |
| LLM indisponível | 503 com `LLM_UNAVAILABLE`; Langfuse sem traces | OpenAI status page | Aguardar se OpenAI down; rollback se deploy causou | CB-001 |
| Banco | 500s generalizados; Prisma timeout | Supabase status + `db_connection_pool_usage` | Railway restart se pool esgotado; aguardar se Supabase down | DB-001 |
| Redis | Rate limit não funcionando; OTP não expirando | Upstash status | Fail-open em vigor; monitorar volume | — |
| Fila RabbitMQ | Notificações não entregues; DLQ crescendo | CloudAMQP status + queue depths | Verificar worker + processar DLQ | NOTIF-001 |
| Webhook WhatsApp | Mensagens WhatsApp não processadas | IP de origem do request + HMAC status | Verificar secret; bloquear IP se malicioso | SEC-002 |
| Autenticação | 401s generalizados | Supabase Auth status + `JWT_DEV_MODE` config | Verificar `JWT_DEV_MODE=false` em produção | — |
| Custo anômalo | `llm_cost_usd_per_day > $100` | Langfuse → cost by trace + tenant | Identificar tenant + verificar loop | COST-001 |

---

## 8. Backup e Restore

### 8.1 Frequência e Armazenamento

- **Backup automático Supabase:** diário (plano Pro) — retenção 7 dias.
- **RPO (Recovery Point Objective):** máximo 24 horas de perda de dados.
- **RTO (Recovery Time Objective):** 30-45 minutos para restore completo.

### 8.2 Quando Optar por Restore vs. Rollback

| Situação | Decisão | Motivo |
| --- | --- | --- |
| Bug de código introduzido pelo deploy | Rollback Railway | Mais rápido (< 3 min); não há perda de dados |
| Corrupção de dados por bug de código | Restore + Rollback | Rollback primeiro (para o código); restore depois (para os dados) |
| Dados corrompidos por operação manual incorreta | Restore | Não há versão de código para reverter |
| Supabase indisponível (infra externa) | Aguardar | Restore não resolve indisponibilidade da plataforma |

### 8.3 Procedimento de Restore (ver detalhes no Doc 26 seção 6)

1. Identificar janela de corrupção (timestamp de início).
2. Supabase Dashboard → Settings → Backups → selecionar backup anterior ao incidente.
3. Clicar "Restore" → confirmar.
4. Aguardar 15-30 min.
5. Verificar integridade dos dados.
6. Reiniciar Railway para reconectar pool.
7. Comunicar stakeholders sobre dados perdidos na janela.

---

## 9. Comunicação de Incidente e Lançamento

### 9.1 Templates por Momento

**Antes do deploy (T-1h):**
```
🚀 Repasse AI v1.x.x — Deploy agendado para 14h00 BRT
Features: [lista de 2-3 items principais]
Time: [nomes]
War room: #repasse-ai-launch
```

**Deploy iniciado:**
```
⚙️ Deploy v1.x.x iniciado em produção — 14h00 BRT
DRI: @nome | Acompanhe: Railway Dashboard
```

**Go-live confirmado (T+15min):**
```
✅ Repasse AI v1.x.x em produção — sistema estável
Error rate: X% | Circuit breaker: CLOSED | Health: OK
War room: #repasse-ai-launch (aberto até T+1h)
```

**Rollback acionado:**
```
⚠️ Rollback para v1.x.x-anterior iniciado
Causa: [motivo em 1 frase]
ETA: ~3 min | Responsável: @nome
Próxima atualização: 15 min
```

**Encerramento do war room:**
```
✅ War room encerrado — Repasse AI v1.x.x estável
Monitoramento intensivo encerrado | Sistema em operação normal
Próxima revisão: T+24h (amanhã)
```

**Comunicação a cessionários (via plataforma Repasse Seguro, se necessário):**
```
[Apenas para degradação > 30 min que afeta o cessionário]
"O assistente de análise de repasses está temporariamente indisponível.
Estimativa de resolução: [hora]. Pedimos desculpas pelo inconveniente."
```

### 9.2 Frequência de Updates em Incidente

| Severidade | Frequência | Canal |
| --- | --- | --- |
| P0 | A cada 15 min | #repasse-ai-launch + #alerts |
| P1 | A cada 30 min | #repasse-ai-launch |
| P2 | A cada 1h | #repasse-ai-launch |

---

## 10. Critério de Go/No-Go

> 🔴 **Regra inegociável:** Go/no-go é uma decisão do Tech Lead com base em dados objetivos — nunca subjetiva ou por pressão de tempo.

### 10.1 Sinais de Go (T+15min pós-deploy)

Todos os critérios abaixo devem ser verdadeiros para confirmar Go:

| Sinal | Threshold de Go |
| --- | --- |
| `GET /health` | 200 com todos os serviços `up` ou `degraded` (não `down`) |
| `http_error_rate` | < 2% nos primeiros 15 min |
| `circuit_breaker_open_count` | 0 |
| `rabbitmq_dlq_count` | 0 |
| Sentry novos erros CRITICAL | 0 (inexistentes antes do deploy) |
| Teste manual de mensagem | Agente responde em < 15s |

[DECISÃO AUTÔNOMA] Threshold `http_error_rate < 2%` como critério de Go: alternativa (5%) descartada por ser tolerância alta demais para serviço B2B com análise de crédito imobiliário. Critério: 2% equivale a 1 em cada 50 requests com erro — aceitável para early-stage sem SLA contratual, mas rigoroso o suficiente para detectar regressão real.

### 10.2 Ações em Caso de No-Go

| Situação | Ação Imediata | Responsável |
| --- | --- | --- |
| `http_error_rate > 10%` | Rollback automático imediato | DRI |
| `http_error_rate` entre 2% e 10% | Aguardar 5 min; se não melhorar → rollback | Tech Lead |
| Circuit breaker OPEN (pré-existente) | Go — monitor intensivo; não é causado pelo deploy | Tech Lead |
| Circuit breaker OPEN (novo) | Aguardar 5 min reset automático; se persistir → rollback | Tech Lead |
| Sentry CRITICAL novo | Avaliar severidade; se funcional crítico → rollback | Tech Lead |
| Qualquer dúvida | **No-go** — deploy pode ser re-tentado; dados perdidos nunca | Tech Lead |

---

## 11. Backlog de Pendências

| ID | Descrição | Prioridade | Observação |
| --- | --- | --- | --- |
| GL-001 | Configurar smoke tests automáticos (`pnpm test:smoke`) que rodam pós-deploy em staging e produção | Alta | Necessário antes do primeiro go-live real |
| GL-002 | Configurar PagerDuty com escalation para Tech Lead em P0 (integração com Railway e Sentry) | Alta | Necessário antes do go-live |
| GL-003 | Definir contato externo para cessionários em caso de degradação > 30 min (email ou WhatsApp do suporte) | Média | Dependente de decisão de produto |
| GL-004 | Criar página de status pública (`status.repasseseguro.com.br`) para transparência com cessionários | Baixa | P2 — não necessário no primeiro go-live |

> **Decisões Autônomas Tomadas Neste Documento:**
>
> 1. **[DECISÃO AUTÔNOMA] Janela de go-live Terça/Quinta 14h-16h BRT:** alinhado com Doc 24 (Deploy). Alternativa (qualquer dia útil) descartada por risco de problema sem equipe disponível. Critério: SRE best practice.
> 2. **[DECISÃO AUTÔNOMA] Threshold `http_error_rate < 2%` para Go:** alternativa 5% descartada por tolerância alta. Critério: produto B2B com análise financeira — rigor maior que produto consumer.
> 3. **[DECISÃO AUTÔNOMA] War room encerrado em T+1h se estável:** alternativa (T+4h) descartada por overhead de processo. Critério: 1h é suficiente para detectar problemas não óbvios nos primeiros 15 min.
> 4. **[DECISÃO AUTÔNOMA] Comunicação a cessionários apenas se degradação > 30 min:** alternativa (comunicar qualquer degradação) descartada por gerar ruído em incidentes curtos auto-resolvidos. Critério: usuários B2B toleram downtime breve sem comunicação; comunicar apenas quando há impacto real prolongado.

---

*Fim do pipeline de documentação: D29 — Go-Live Playbook concluído.*
*Pipeline ShiftLabs v9.5 — Fase 2 completa.*
