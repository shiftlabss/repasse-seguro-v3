# 29 - Go-Live Playbook — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| **Nome do Documento** | Go-Live Playbook — AI-Dani-Cedente |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Bloco** | 7 — Go-Live |
| **Dependências** | D14, D17, D24, D25, D26 |

---

> **📌 TL;DR**
>
> - **Janelas de preparação:** T-7 (prontidão técnica), T-3 (staging validado), T-1 (checklist final).
> - **Launch Day:** janela recomendada Terça-Quinta, 10h–14h (horário de Fortaleza). Deploy + health checks + 30min de monitoramento intensivo.
> - **Critério de Go:** `GET /health/ready` OK + taxa de erro < 2% + circuit breaker fechado + smoke test manual aprovado.
> - **Critério de No-Go / Rollback automático:** taxa de erro > 2% nos primeiros 15min OU health check falhando OU circuit breaker abrindo.
> - **Rollback:** `railway rollback` em < 2min. Decidido pelo Tech Lead — sem necessidade de consenso em emergência.
> - **Estabilização:** monitoramento intensivo por 24h. War room encerrado após T+1h se métricas OK.
> - **Comunicação:** 5 momentos de comunicação — T-1, Deploy, Go, Incidente (se houver), Estabilização.

---

## 1. Matriz de Escalation

| Severidade | Impacto | Canal | Pessoas Acionadas | SLA de Resposta | Critério de Escalação |
|---|---|---|---|---|---|
| **P0 — Crítico** | Serviço indisponível ou dados de Cedente em risco | PagerDuty + Slack #ops-incidents | Tech Lead + Product Lead | 15 min | Qualquer membro do time pode acionar imediatamente |
| **P1 — Alto** | Funcionalidade crítica degradada (chat, proposta, Escrow) | Slack #ops-incidents | Tech Lead | 1h | Abre após 15min sem resolução em P0 transitório |
| **P2 — Médio** | Funcionalidade não-crítica degradada | Slack #ops-alerts | Dev on-call | 4h | Abre se não resolvido em 2h |
| **P3 — Baixo** | Anomalia monitorada, sem impacto direto ao usuário | Slack #ops-alerts | Dev on-call | 24h | Ticket para próxima sprint |
| **Segurança** | Vazamento de dados de Cedente (qualquer escala) | Slack #ops-security + DPO | Tech Lead + DPO + CEO | Imediato | Primeiro suspeita, antes de confirmação |

[PENDÊNCIA: definir nomes e contatos do on-call. Preencher antes do Go-Live.]

---

## 2. Health Checks

### 2.1 Endpoints de Health

| Endpoint | Frequência de Verificação | Threshold de Aprovação | Ação em Falha |
|---|---|---|---|
| `GET /health/ready` | A cada 30s (automático Railway) | HTTP 200 com todos os checks OK | Rollback automático se falhar 3x seguidas |
| `GET /health/live` | A cada 15s (automático Railway) | HTTP 200 | Reiniciar pod |
| `GET /api/v1/admin/fallback/status` | Manual — a cada 15min no Launch Day | `{ "enabled": true }` | Investigar circuit breaker |

### 2.2 Dashboards Obrigatórios no Launch Day

| Dashboard | Pergunta Guia | Threshold Go | Threshold No-Go |
|---|---|---|---|
| Saúde Geral (Grafana) | Dani está respondendo bem? | Taxa de erro < 2%, p95 ≤ 5s | Taxa de erro > 2% OU p95 > 5s por 5min |
| Agente IA (Grafana/Langfuse) | LLM operando normalmente? | Erro LLM < 5% | Erro LLM > 10% |
| Notificações (Grafana) | Notificações chegando? | DLQ = 0, entrega ≥ 95% | DLQ > 5 ou entrega < 90% |
| PostHog CSAT | Cedentes satisfeitos? | CSAT ≥ 4.0 nas primeiras avaliações | CSAT < 3.0 em ≥ 3 avaliações seguidas |

### 2.3 Smoke Test Manual — Launch Day

Executar manualmente após deploy e health check automático:

1. Abrir chat widget como Cedente de teste → mensagem de saudação recebida? ✅
2. Perguntar sobre oportunidade cadastrada → Dani responde com dados corretos? ✅
3. Simular início de análise de proposta → Dani entra no estado correto? ✅
4. Verificar que outro Cedente não vê dados do primeiro → 404 retornado? ✅
5. Enviar 30 mensagens rápidas → rate limit ativo na 31ª? ✅
6. Verificar notificação de teste → toast exibido no widget? ✅

---

## 3. Pré-Launch

### T-7 (7 dias antes do Go-Live)

| Item | Responsável | Critério de Sucesso |
|---|---|---|
| [ ] Todos os 29 documentos técnicos gerados e revisados | Tech Lead | Nenhum `[PENDÊNCIA CRÍTICA]` sem resolução |
| [ ] Testes E2E de fluxos P0 passando em staging | QA Lead | CI verde + relatório de cobertura ≥ 90% módulos críticos |
| [ ] Integrações externas testadas em staging: OpenAI, Supabase, ZapSign, Langfuse, PostHog | Tech Lead | Cada integração com teste de round-trip bem-sucedido |
| [ ] Checklist de Qualidade (D28) auditado — sem itens bloqueantes pendentes | Tech Lead | Todos os B-001 a B-018 confirmados |
| [ ] Secrets de produção configurados no Railway | DevOps | `railway vars list` sem valores de staging/test |
| [ ] Backup de staging validado (PITR Supabase) | DevOps | Restore de teste bem-sucedido em ambiente isolado |
| [ ] Runbook Operacional (D26) revisado pelo time | Tech Lead + Dev | Todos do time leram e validaram os procedimentos |

### T-3 (3 dias antes do Go-Live)

| Item | Responsável | Critério de Sucesso |
|---|---|---|
| [ ] Deploy em staging com exata versão a ser lançada (`release/X.Y.Z`) | Tech Lead | Health check verde em staging |
| [ ] Smoke test completo em staging (6 itens da seção 2.3) | QA Lead | 6/6 itens passando |
| [ ] Alertas D25 configurados e testados (Grafana + PagerDuty) | DevOps | Alerta de teste disparado e recebido nos canais corretos |
| [ ] Dados de seed de produção validados (Cedentes piloto cadastrados) | Product Lead | [PENDÊNCIA: lista de Cedentes piloto para Go-Live] |
| [ ] Comunicação interna enviada: data, hora, canal de incidente | Product Lead | Time notificado via Slack |
| [ ] Rollback testado em staging (`railway rollback`) | Tech Lead | Rollback executado em < 2min + health check OK |
| [ ] Plano de comunicação externa (se houver) aprovado | Product Lead | Template de mensagem revisado e aprovado |

### T-1 (1 dia antes do Go-Live)

| Item | Responsável | Critério de Sucesso |
|---|---|---|
| [ ] Freeze de deploys exceto o de Go-Live | Tech Lead | Nenhum PR mergeado em `develop` após freeze |
| [ ] PR de release (`release/X.Y.Z`) com 2 aprovações | Tech Lead | PR aprovado, aguardando merge |
| [ ] Health check de produção manual | DevOps | `GET /health/ready` OK em produção (estado pré-Go-Live) |
| [ ] Dashboard Grafana aberto e monitorado por 1h | DevOps | Sem alertas não esperados |
| [ ] Comunicação interna final: horário exato, responsáveis, canais | Product Lead | Todos confirmaram recebimento |
| [ ] On-call confirmado para o dia do Go-Live | Tech Lead | Contato de on-call disponível e confirmado |
| [ ] `NOTIFICATION_MODE=production` configurado | DevOps | Verificar Railway env var antes do Go-Live |

---

## 4. Launch Day

### 4.1 Timeline do Launch Day

| Horário | Atividade | Responsável | Critério de Sucesso |
|---|---|---|---|
| T-60min | Abrir war room no Slack (`#ops-incidents`) | Tech Lead | Channel aberto, todos avisados |
| T-30min | Verificar health check produção + dashboards baseline | DevOps | Tudo verde — registrar screenshot como baseline |
| T-15min | Confirmar on-call disponível + runbook aberto | Tech Lead | [PENDÊNCIA: contato on-call] |
| T-0 | **Go/No-Go decision** | Tech Lead | Todos os critérios de Go confirmados (seção 10) |
| T+0 | Merge do PR `release/X.Y.Z` em `main` | Tech Lead | Squash merge com 2 aprovações |
| T+0 | `git tag vX.Y.Z && git push --tags` | Tech Lead | Tag criada — pipeline de deploy dispara |
| T+5min | Monitorar GitHub Actions `deploy.yml` | DevOps | Workflow em execução sem erros |
| T+10min | Health check automático Railway | Railway | `/health/ready` retorna 200 |
| T+15min | **Go/No-Go pós-deploy** | Tech Lead | Taxa de erro < 2% + p95 ≤ 5s + circuit breaker fechado |
| T+20min | Smoke test manual (6 itens) | QA Lead | 6/6 itens passando |
| T+30min | Comunicação: "Go-Live concluído" | Product Lead | Mensagem enviada nos canais definidos (seção 9) |
| T+60min | Primeira avaliação de estabilização | Tech Lead | Métricas dentro dos thresholds |

### 4.2 Ativação de Funcionalidades

[DECISÃO AUTÔNOMA]: Go-Live completo desde o início, sem feature flags por funcionalidade — Justificativa: a AI-Dani-Cedente é um serviço monolítico sem frontend próprio e o PostHog flag `dani_cedente_enabled` controla a visibilidade na plataforma como um todo | Alternativa descartada: feature flags granulares por módulo — complexidade desnecessária para o perfil do serviço; o circuit breaker já gerencia degradação parcial.

**Passo de ativação:**
```bash
# Verificar que feature flag está ativa na plataforma Repasse Seguro
# PostHog → Feature Flags → dani_cedente_enabled → deve estar ON para % de usuários alvo
```

---

## 5. Pós-Launch

### T+15min

- [ ] Taxa de erro < 2%? (Grafana Dashboard "Saúde Geral")
- [ ] p95 ≤ 5s? (Grafana → `dani_chat_message_duration_ms`)
- [ ] Circuit breaker fechado? (`GET /admin/fallback/status`)
- [ ] DLQ = 0? (Grafana → Dashboard "Notificações")
- [ ] Nenhum alerta P0/P1 disparado?

**Se qualquer item falhar → Rollback imediato (seção 6). Não aguardar.**

### T+1h

- [ ] CSAT de primeiras sessões ≥ 4.0? (PostHog)
- [ ] Taxa de resolução ≥ 80%? (`dani_resolution_rate`)
- [ ] Nenhuma sessão com takeover anômalo (> 30% das sessões)?
- [ ] Notificações críticas entregues com sucesso? (notification_log)
- [ ] Memória RSS estável (sem crescimento linear)? (Grafana)

**Se T+1h OK → Encerrar war room. Manter monitoramento contínuo por 24h.**

### T+24h

- [ ] CSAT médio ≥ 4.0 nas últimas 24h? (PostHog)
- [ ] Sem incidentes P0/P1 nas últimas 24h?
- [ ] Taxa de erro LLM < 2% nas últimas 24h?
- [ ] Custo OpenAI dentro do budget estimado? (Langfuse → cost tracking)
- [ ] Nenhuma notificação na DLQ?
- [ ] Post-mortem de qualquer incidente P1+ agendado?

**Se T+24h OK → Emitir comunicação de estabilização. Go-Live concluído com sucesso.**

---

## 6. Deploy e Rollback

### 6.1 Deploy de Go-Live

```bash
# 1. Merge do PR de release
# GitHub: PR 'release/X.Y.Z' → main. Merge commit (não squash).

# 2. Criar tag de release
git tag vX.Y.Z -m "Go-Live vX.Y.Z — AI-Dani-Cedente Fase 1"
git push origin vX.Y.Z

# 3. GitHub Actions deploy.yml é disparado automaticamente
# Aguardar conclusão no GitHub Actions tab

# 4. Health check (automático pelo workflow)
# Se falhar → Railway faz rollback automático

# 5. Smoke test manual após deploy
# Ver seção 2.3
```

### 6.2 Critérios para Rollback Imediato

| Gatilho | Quando | Ação |
|---|---|---|
| `GET /health/ready` retorna 5xx | A qualquer momento nos primeiros 30min | Rollback imediato |
| Taxa de erro > 2% por > 5min | Nos primeiros 30min | Rollback imediato |
| Circuit breaker abrindo | Nos primeiros 15min | Rollback imediato |
| Smoke test falhando | Após deploy | Rollback imediato |
| Dados de Cedente vazando | A qualquer momento | Rollback + ativar P0 + notificar DPO |

### 6.3 Execução do Rollback

```bash
# 1. Decisão do Tech Lead (não precisa de consenso em emergência)
# Comunicar no #ops-incidents: "[ROLLBACK INICIADO] Taxa de erro > 2%"

# 2. Executar rollback
railway deployments list --service dani-cedente-prod
railway rollback --deployment <deployment-id-anterior>

# 3. Verificar
sleep 30
curl -f https://dani-cedente.railway.app/health/ready && echo "ROLLBACK OK"

# 4. Verificar taxa de erro (aguardar 5min)
# Grafana → Dashboard "Saúde Geral"

# 5. Comunicar encerramento
# "#ops-incidents: [ROLLBACK CONCLUÍDO] Revertido para vX.Y.Z-1. Motivo: <descrição>."
```

**Tempo esperado:** < 2 minutos.

---

## 7. Incidentes por Categoria no Launch Day

| Categoria | Sintoma | Triagem | Ação Inicial |
|---|---|---|---|
| **Backend crash** | `/health/ready` retorna 5xx | `railway logs --tail 50` | Rollback imediato se deploy recente |
| **LLM indisponível** | Erros DCE-AGENT-503, taxa > 10% | `curl status.openai.com` | Aguardar ou acionar circuit breaker manual |
| **Banco lento** | Latência > 5s, timeouts Prisma | `pg_stat_activity` queries > 2s | Matar queries longas, verificar locks |
| **Redis down** | `/health/ready` `redis: error` | `redis-cli ping` | Modo degradado — sem rate limiting; abrir ticket Railway |
| **Fila parada** | DLQ crescendo, workers não consumindo | RabbitMQ Management UI | Reiniciar RabbitMQ addon |
| **Autenticação falhando** | 401 em massa, JWT rejeitado | Verificar `SUPABASE_JWT_SECRET` | Verificar env var de produção vs staging |
| **ZapSign webhook inválido** | HMAC inválido em logs | Verificar `ZAPSIGN_WEBHOOK_SECRET` | Comparar chave prod vs staging |
| **Isolamento de Cedente** | Dados cruzados (B-001) | `correlation_id` + logs | P0 + DPO + rollback imediato |

---

## 8. Backup e Restore

### 8.1 Backup no Go-Live

| Tipo | Frequência | Retenção | Responsável |
|---|---|---|---|
| Supabase PITR (contínuo) | Contínuo | 7 dias | Automático (Supabase) |
| Snapshot manual pré-Go-Live | 1x antes do deploy | 30 dias | DevOps — executar T-1h antes do Go-Live |
| Snapshot manual pós-rollback | 1x após qualquer rollback | 30 dias | DevOps |

**Snapshot manual pré-Go-Live:**
```bash
# Supabase Dashboard → Database → Backups → Create backup
# Nomear como: "pre-golive-vX.Y.Z-YYYYMMDD-HHMM"
```

### 8.2 Restore vs. Rollback

| Situação | Usar Rollback | Usar Restore |
|---|---|---|
| Bug de código pós-deploy | ✅ Rollback (< 2min, sem perda de dados) | ❌ |
| Migration incompatível aplicada | ✅ Rollback + rollback da migration | ✅ Se rollback da migration não for possível |
| Dados corrompidos por bug de lógica | ❌ | ✅ Restore para ponto antes da corrupção |
| Dados deletados acidentalmente | ❌ | ✅ PITR para antes da deleção |

**Tempo de restore Supabase:** 30–60 minutos. Requer comunicação a stakeholders antes de iniciar.

---

## 9. Comunicação de Incidente e Lançamento

### 9.1 Templates por Momento

**T-1 — Comunicação de Go-Live Agendado (Slack #product-releases):**
```
[GO-LIVE AMANHÃ] AI-Dani-Cedente vX.Y.Z
⏰ Horário: {data} às {hora} (Fortaleza)
📦 Escopo: {resumo das features}
👤 Responsável: @{tech-lead}
🔗 Release notes: {link}
Canal de incidente: #ops-incidents
```

**Deploy iniciado (automático GitHub Actions → #ops-incidents):**
```
[DEPLOY INICIADO] vX.Y.Z em produção
🔗 Workflow: {link GitHub Actions}
⏰ {timestamp}
```

**T+30min — Go-Live confirmado (#product-releases):**
```
[GO-LIVE CONCLUÍDO] AI-Dani-Cedente vX.Y.Z ✅
✅ Health check: OK
✅ Smoke test: 6/6 itens
✅ Taxa de erro: < 1%
✅ Circuit breaker: fechado
Monitoramento intensivo por mais 30min.
```

**Incidente durante Go-Live (#ops-incidents):**
```
[INCIDENTE Go-Live] P{N} — {Título}
⏰ {timestamp}
📍 Escopo: {impacto}
🔄 Ação: {rollback/investigando}
👤 Responsável: @{nome}
```

**T+24h — Estabilização confirmada (#product-releases):**
```
[GO-LIVE ESTÁVEL] AI-Dani-Cedente vX.Y.Z — 24h ✅
📊 CSAT: {valor}/5.0
📈 Taxa resolução: {%}
🔒 Incidentes: {0 / N resolvidos}
🎉 Fase 1 lançada com sucesso!
```

### 9.2 Comunicação de Rollback

```
[ROLLBACK] AI-Dani-Cedente vX.Y.Z → vX.Y.Z-1
⏰ {timestamp}
❌ Motivo: {descrição objetiva}
✅ Status: RESTORED ({duração do rollback})
🔍 Investigação: em andamento
📋 Post-mortem: agendado para {data}
```

---

## 10. Critério de Go/No-Go

### 10.1 Quem Decide

**Decisor:** Tech Lead. Sem necessidade de consenso. A decisão é unilateral e imediata.

**Consultores:** Product Lead (impacto de negócio), DevOps (saúde de infra), QA Lead (smoke test).

### 10.2 Critérios de GO ✅

Todos os itens abaixo DEVEM ser verdadeiros para prosseguir:

- [ ] `GET /health/ready` retorna HTTP 200 com todos os checks OK.
- [ ] Smoke test manual: 6/6 itens passando.
- [ ] Taxa de erro < 2% nos últimos 5 minutos pós-deploy.
- [ ] p95 de latência ≤ 5.000ms.
- [ ] Circuit breaker fechado (`dani_circuit_breaker_state = 0`).
- [ ] DLQ = 0.
- [ ] Nenhum alerta P0/P1 ativo.
- [ ] Todos os itens do T-1 checklist confirmados.

### 10.3 Critérios de NO-GO ❌

Qualquer um dos itens abaixo dispara NO-GO e rollback imediato:

- [ ] `GET /health/ready` retornando 5xx.
- [ ] Taxa de erro > 2% por > 5 minutos pós-deploy.
- [ ] p95 > 8.000ms por > 5 minutos pós-deploy.
- [ ] Circuit breaker abrindo nos primeiros 15 minutos.
- [ ] Qualquer item do smoke test falhando.
- [ ] DLQ > 5 mensagens em 30 minutos.
- [ ] Dados de Cedente sendo acessados de forma cruzada (B-001 violado).

### 10.4 Se No-Go

1. Tech Lead decide rollback imediatamente: `railway rollback --deployment <id>`.
2. Comunicar no #ops-incidents com motivo.
3. Agendar post-mortem em 24h.
4. Renegociar data de Go-Live após correção da causa raiz.

---

## 11. Backlog de Pendências

| # | Item | Tipo | Prioridade |
|---|---|---|---|
| P-GOL-001 | Definir lista de Cedentes piloto para o Go-Live (usuários de confiança para primeiros testes reais) | [PENDÊNCIA] — decisão de produto | Alta |
| P-GOL-002 | Definir contato de on-call (nome, Slack, telefone) para o Launch Day | [PENDÊNCIA] — decisão de time | Alta |
| P-GOL-003 | Configurar PagerDuty com escalation para fora do horário comercial | [SEÇÃO PENDENTE] — requer conta PagerDuty | Média |
| P-GOL-004 | Definir política de comunicação externa para Cedentes durante incidente (se SLA < 99.5%) | [DEFINIÇÃO PENDENTE] — afeta comunicação regulatória. Opção A: comunicação automática via e-mail após 30min de downtime. Opção B: comunicação manual pelo Product Lead apenas se solicitado. | Média |
| P-GOL-005 | Criar Runbook de Go-Live específico para Fase 2 (WhatsApp + React Native) | [SEÇÃO PENDENTE] — Fase 2 não especificada | Baixa (Fase 2) |

---

## 12. Changelog do Documento

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — 3 janelas pré-launch (T-7, T-3, T-1), launch day com timeline hora a hora, 3 janelas pós-launch (T+15min, T+1h, T+24h). Critérios objetivos de Go (8 itens) e No-Go (7 itens). Rollback em < 2min. 8 categorias de incidente no Launch Day. Restore vs. Rollback diferenciados. 5 templates de comunicação. |
