# 29 — Go-Live Playbook

## Metadados

| Campo       | Valor                                                             |
|-------------|-------------------------------------------------------------------|
| **Nome**    | Go-Live Playbook — Repasse Seguro                                 |
| **Versão**  | 1.0.0                                                             |
| **Data**    | 2025-01-01                                                        |
| **Autor**   | DevOps / SRE / Tech Lead / Product Manager                        |
| **Status**  | Ativo                                                             |
| **Produto** | Repasse Seguro — Plataforma de Cessões Imobiliárias               |

---

📌 **TL;DR — Resumo Operacional**

Este playbook governa o lançamento do Repasse Seguro em produção como operação controlada. Cobre preparação em T-7 / T-3 / T-1, launch day com critério de go/no-go explícito, estabilização em T+15min / T+1h / T+24h, rollback, incidentes por categoria e comunicação de ponta a ponta.

Regras inegociáveis:
- **Go/no-go** é decidido pelo Tech Lead com base em sinais objetivos — não é opinião.
- **Rollback automático** é acionado se error rate > 2% nos primeiros 15 minutos (ver §8).
- **War room** permanece ativo por 24h após go-live; encerramento mediante critério formal.
- **Janela de deploy** autorizada: segunda a sexta, 10h–16h (America/Fortaleza) — sem exceções sem aprovação de Product Manager + Tech Lead.
- **Supervisão Total de IA** ativa nos primeiros 30 dias — todos os agentes requerem aprovação humana.

---

## 3. Matriz de Escalation

| Severidade | Impacto | Canal de Abertura | Pessoas Acionadas | SLA de Resposta | Critério de Escalação |
|------------|---------|-------------------|-------------------|-----------------|----------------------|
| **Critical** | Plataforma indisponível ou perda de dados financeiros | Slack `#rs-incidents` @channel + telefonema direto | Tech Lead + CEO + DevOps | **15 minutos** | Erros 5xx > 5%, DB inacessível, escrow com falha, rollback em curso |
| **High** | Feature principal degradada ou integração externa crítica offline | Slack `#rs-incidents` + menção direta ao Tech Lead | Tech Lead + Backend Lead | **1 hora** | ZapSign offline, Resend com falha, AI agents falhando > 3x consecutivas |
| **Medium** | Feature secundária com problema ou latência degradada | Slack `#rs-alertas` | Backend Lead + Frontend Lead | **4 horas** | p95 > 1s por > 10min, DLQ com mensagens, WhatsApp fallback para SMS |
| **Low** | Problema cosmético, analytics offline, log ausente | Slack `#rs-dev` | Desenvolvedor de plantão | **24 horas** | PostHog offline, log level incorreto, alerta não crítico |

### 3.1 Responsáveis por função

| Papel | Nome | Canal de contato | Backup |
|-------|------|-----------------|--------|
| Tech Lead | [DADO PENDENTE] | Slack + WhatsApp | [DADO PENDENTE] |
| Backend Lead | [DADO PENDENTE] | Slack + WhatsApp | [DADO PENDENTE] |
| Frontend Lead | [DADO PENDENTE] | Slack | [DADO PENDENTE] |
| DevOps / SRE | [DADO PENDENTE] | Slack + WhatsApp | [DADO PENDENTE] |
| Product Manager | [DADO PENDENTE] | Slack + WhatsApp | [DADO PENDENTE] |
| On-call (plantão) | Rotativo semanal | Slack `#rs-incidents` | Tech Lead |

---

## 4. Health Checks

### 4.1 Endpoints críticos

| Endpoint | Método | Frequência | Threshold | Responsável | Interpretação de falha |
|----------|--------|------------|-----------|-------------|----------------------|
| `/health` (API Railway) | GET | A cada 30s (Railway) | HTTP 200 com `status: ok` para DB + Redis + RabbitMQ | DevOps | Qualquer dependência `degraded` → escalação imediata |
| `/health/ready` (API Railway) | GET | A cada 30s | HTTP 200 — aplicação pronta para receber tráfego | DevOps | HTTP 503 → deploy não concluído ou aplicação travada |
| Supabase Dashboard | Dashboard | Contínuo (via alertas) | Connection pool < 80% de utilização | Backend Lead | > 80% utilização → checar queries longas no Supabase |
| Redis (Upstash Console) | Console | A cada 5min pós-launch | Hit rate > 60%, memória < 80% | Backend Lead | Hit rate baixo → snapshot de configs não cacheado |
| RabbitMQ (CloudAMQP UI) | Console | A cada 5min pós-launch | Filas com profundidade < 100, DLQ = 0 | Backend Lead | DLQ > 0 → verificar workers e retry policy |

### 4.2 Dashboards de monitoramento

| Dashboard | URL / Localização | O que monitorar | Responsável |
|-----------|------------------|-----------------|-------------|
| API Health (Railway) | Railway Project → Metrics | CPU, memória, latência, erro rate | DevOps |
| Sentry Issues | Sentry dashboard | Novos erros e volume de erros | Backend Lead |
| PostHog | PostHog dashboard | Eventos de criação de conta, login, criação de caso | Product Manager |
| CloudAMQP | CloudAMQP console | Fila de notificações, DLQ | Backend Lead |
| Upstash Redis | Upstash console | Hits, comandos/s, memória | DevOps |
| Vercel Analytics | Vercel dashboard (SPA) | Web Vitals, erros de frontend | Frontend Lead |

### 4.3 SLOs monitorados no launch day

| SLO | Threshold de alerta | Threshold crítico (no-go / rollback) |
|-----|--------------------|------------------------------------|
| API availability | < 99.5% em 15min | < 95% em qualquer janela de 5min |
| p95 latency | > 500ms | > 2s por > 5min |
| Error rate (5xx) | > 1% | > 2% por > 5min |
| Email delivery | < 95% em 5min | < 80% em 15min |
| DLQ rate | > 0 mensagens | > 5 mensagens não processadas |

---

## 5. Pré-Launch

### 5.1 T-7 dias (Semana anterior ao lançamento)

**Responsável geral:** Tech Lead

| # | Item | Dono | Critério de sucesso | Ação se falhar |
|---|------|------|--------------------|--------------------|
| 1 | Congelar `develop` — nenhum merge de feature nova após esta data; apenas bugfixes e pré-release tasks | Tech Lead | Branch `develop` com tag `release-candidate-v1.0.0-rc1` | Adiar go-live até freeze alcançado |
| 2 | Executar suite completa de testes (unit + integration + E2E) no ambiente de staging | QA / Backend Lead | 100% dos 5 fluxos críticos E2E passando; cobertura ≥ 80% em módulos críticos | Bloquear go-live; abrir issues para falhas |
| 3 | Validar todas as variáveis de ambiente de produção (Railway + Vercel) | DevOps | Checklist de env vars revisado; nenhuma variável indefinida em produção | Preencher vars antes de T-3 |
| 4 | Confirmar quotas e limites dos serviços externos: Resend (envios/dia), Meta Cloud API, Twilio, ZapSign | DevOps | Confirmação por escrito no Slack `#rs-go-live` para cada serviço | Negociar aumento de quota antes de T-3 |
| 5 | Executar `pnpm audit` — zero vulnerabilidades High/Critical | Backend Lead | Saída `pnpm audit` sem itens High ou Critical | Corrigir vulnerabilidades antes de T-3 |
| 6 | Revisar Checklist de Qualidade (D28) completo | Tech Lead + QA | Zero itens 🔴 em aberto | Resolver bloqueantes antes de T-3 |
| 7 | Validar backup de banco de dados em staging: executar restore de teste do backup mais recente | DevOps | Restore concluído com sucesso em < 30min; dados íntegros | Investigar causa; não avançar sem restore validado |
| 8 | Comunicar stakeholders internos sobre data e janela de go-live | Product Manager | Confirmação de recebimento de CEO + time de operações | Reagendar comunicação antes de T-3 |

---

### 5.2 T-3 dias

**Responsável geral:** Tech Lead + DevOps

| # | Item | Dono | Critério de sucesso | Ação se falhar |
|---|------|------|--------------------|--------------------|
| 1 | Validar migrations de produção em staging com dados reais (cópia anonimizada) | Backend Lead | Todas as migrations aplicadas sem erro; zero dados corrompidos | Reverter migration problemática; adiar go-live |
| 2 | Configurar alertas em Railway + Sentry + CloudAMQP para thresholds do launch day | DevOps | Testar cada alerta disparando condição manualmente em staging | Configurar alertas manualmente antes de T-1 |
| 3 | Validar todas as integrações externas em produção com chamadas de teste: ZapSign, Resend, Meta Cloud API, Twilio | DevOps + Backend Lead | Cada integração responde 200 com payload válido | Contatar suporte da integração; não avançar sem validação |
| 4 | Deploy do Release Candidate para staging e executar smoke test manual dos 5 fluxos críticos | QA | Todos os fluxos concluídos sem erro no ambiente de staging | Correção e redeploy em staging antes de T-1 |
| 5 | Validar rollback em staging: executar rollback de versão e confirmar que aplicação volta ao estado anterior | DevOps | Railway rollback < 1min; Vercel rollback < 30s; health check passando após rollback | Investigar processo de rollback antes de T-1 |
| 6 | Definir war room: canal Slack `#rs-war-room` criado, acesso confirmado para todos os participantes | Tech Lead | Canal criado, todos com acesso, horários de plantão definidos | Criar canal e definir plantão antes de T-1 |
| 7 | Confirmar janela de deploy: segunda a sexta, 10h–16h (Fortaleza), sem outros deploys de terceiros nesta janela | Tech Lead + Product Manager | Confirmação no Slack `#rs-go-live` | Renegociar data se conflito existir |

---

### 5.3 T-1 dia (Véspera do lançamento)

**Responsável geral:** Tech Lead

| # | Item | Dono | Critério de sucesso | Ação se falhar |
|---|------|------|--------------------|--------------------|
| 1 | Congelar `main` — nenhuma alteração após este momento sem aprovação explícita do Tech Lead | Tech Lead | Última tag de release criada (`v1.0.0`) no GitHub | Adiar go-live |
| 2 | Confirmar ambientes de produção: Railway API rodando, Vercel SPA publicada, Supabase projeto de produção ativo | DevOps | `curl -s https://api.repasseseguro.com.br/health` retorna 200 | Investigar e resolver antes do launch day |
| 3 | Revisar Runbook Operacional (D26) — todos os participantes do war room devem ter lido os 5 cenários de incidente | Tech Lead | Confirmação por escrito de cada participante no Slack | Agendar leitura rápida no início do launch day |
| 4 | Criar snapshot de configuração (snapshot dos agentes de IA) para uso no launch day | Backend Lead | Snapshot registrado em `ai_config_snapshots` com timestamp e hash | Criar snapshot manualmente antes do deploy |
| 5 | Preparar mensagens de comunicação de lançamento (ver §11) para envio após go-live | Product Manager | Templates prontos e revisados | Preparar textos com PM antes do launch day |
| 6 | Confirmar que modo "Supervisão Total" de IA está ativado — todos os agentes requerem aprovação humana | Tech Lead | Flag de supervisão total ativa em produção; confirmar via tabela `system_configs` | Ativar manualmente via migration ou script |
| 7 | Definir who-is-on-call para as primeiras 24h: nome, contato e horário de cada turno | Tech Lead | Lista publicada no Slack `#rs-war-room` | Tech Lead assume primeira janela manualmente |

---

## 6. Launch Day

### 6.1 Cronograma do Launch Day

**Janela autorizada:** 10h00–16h00 (America/Fortaleza), segunda a sexta.

**Horário de início recomendado:** 10h00 — máxima janela de recuperação antes das 16h.

| Horário | Atividade | Responsável | Duração estimada |
|---------|-----------|-------------|-----------------|
| 09h45 | Verificação final: war room ativo, todos confirmados, health checks de staging OK | Tech Lead | 15 min |
| 10h00 | ✅ **GO/NO-GO inicial** — decisão formal de prosseguir (ver §12) | Tech Lead | 5 min |
| 10h05 | Executar migrations de produção via `deploy.yml` (passo de migration antes do app deploy) | DevOps | 10–15 min |
| 10h20 | Verificar migrations aplicadas: `railway logs \| grep "migration"` — zero erros | Backend Lead | 5 min |
| 10h25 | ✅ **GO/NO-GO pós-migration** — confirmar que dados estão íntegros antes de subir nova versão | Tech Lead + Backend Lead | 5 min |
| 10h30 | Deploy da API no Railway (rolling update automático via tag `v1.0.0`) | DevOps | 5–10 min |
| 10h40 | Deploy da SPA no Vercel (atomic deploy automático) | DevOps | 3–5 min |
| 10h45 | Smoke test manual: Login 2FA, Criação de Caso, consultar caso, verificar dashboard Admin | QA + Product Manager | 15 min |
| 11h00 | ✅ **GO/NO-GO pós-smoke test** — confirmar que fluxos críticos funcionam em produção | Tech Lead + QA | 5 min |
| 11h05 | Liberar acesso para usuários internos (piloto fechado — equipe interna) | Product Manager | — |
| 11h15 | Monitoramento intensivo T+15min (ver §7) | DevOps + Backend Lead | 15 min |
| 11h30 | Monitoramento T+1h começa | DevOps | — |
| 12h30 | Revisão T+1h — check de saúde da plataforma | Tech Lead | 10 min |
| 13h00 | Comunicação pública de lançamento (se go-live estável) | Product Manager | — |

### 6.2 Checklist de Launch Day

| # | Item | Dono | Horário | Critério de sucesso |
|---|------|------|---------|---------------------|
| L01 | War room Slack `#rs-war-room` ativo, todos presentes | Tech Lead | 09h45 | Todos confirmados com emoji ✅ na mensagem |
| L02 | Health checks de staging confirmados (API + DB + Redis + RabbitMQ) | DevOps | 09h50 | Todos os serviços `status: ok` |
| L03 | Variáveis de ambiente de produção verificadas uma última vez | DevOps | 09h55 | Zero variáveis indefinidas ou expiradas |
| L04 | Ponto de restore registrado: nota do commit hash atual em `main` | Backend Lead | 10h00 | Hash registrado no Slack `#rs-war-room` |
| L05 | GO/NO-GO inicial declarado | Tech Lead | 10h00 | Declaração explícita no canal `#rs-war-room` |
| L06 | Migrations executadas sem erro | DevOps | 10h05–10h20 | `railway logs` sem linha de erro de migration |
| L07 | GO/NO-GO pós-migration | Tech Lead | 10h25 | Confirmação explícita ou acionamento de restore |
| L08 | API Railway respondendo após rolling update | DevOps | 10h40 | `/health` retorna 200 com todas as deps `ok` |
| L09 | SPA Vercel publicada e acessível | DevOps | 10h45 | URL de produção carrega sem erro no browser |
| L10 | Smoke test dos 5 fluxos críticos | QA | 10h45–11h00 | 100% dos fluxos concluídos sem erro visível |
| L11 | GO/NO-GO pós-smoke test | Tech Lead | 11h00 | Declaração no `#rs-war-room` |
| L12 | Erro rate monitorado: < 1% nos primeiros 15min | DevOps | 11h00–11h15 | Railway Metrics: error rate abaixo do threshold |
| L13 | Sentry: zero novos erros Critical no primeiro T+15min | Backend Lead | 11h15 | Sentry Issues: zero itens novos com nível Critical |

---

## 7. Pós-Launch

### 7.1 T+15 minutos

**Responsável:** DevOps + Backend Lead

| # | Verificação | Threshold | Ação se falhar |
|---|------------|-----------|----------------|
| 1 | Error rate 5xx | < 1% | Se > 2%: acionar rollback imediato (§8) |
| 2 | Latência p95 | < 500ms | Se > 2s por > 5min: escalar para Tech Lead |
| 3 | DLQ de notificações | 0 mensagens | Se > 0: verificar workers e retry; escalar se > 5 |
| 4 | Health check `/health` | HTTP 200, todos `ok` | Se qualquer dep `degraded`: escalar imediatamente |
| 5 | Sentry novos erros | 0 Critical | Se Critical aparecer: triagem imediata |
| 6 | Fluxo de Login 2FA | Funcional | Se quebrar: no-go, rollback |
| 7 | Notificação de teste (e-mail) | Entregue em < 5min | Se falhar: verificar Resend + DLQ |

### 7.2 T+1 hora

**Responsável:** Tech Lead (revisão) + DevOps (monitoramento)

| # | Verificação | Threshold | Ação se falhar |
|---|------------|-----------|----------------|
| 1 | API availability acumulada | ≥ 99.5% | Abrir incidente High se < 99% |
| 2 | Taxa de erros de login | < 5% | Verificar AuthService; checar rate limit |
| 3 | Taxa de criação de caso com sucesso | ≥ 95% das tentativas | Investigar CasesService; checar snapshot de config |
| 4 | Fila de notificações | Profundidade < 100 | Se acumulando: escalar para Backend Lead |
| 5 | Redis hit rate | > 60% | Se baixo: verificar cache de configs de IA |
| 6 | Supabase connections | < 80% do pool | Se atingir 80%: investigar queries longas |
| 7 | Sentry error volume tendência | Decrescente ou estável | Se crescente: triagem urgente antes de T+2h |

### 7.3 T+24 horas

**Responsável:** Tech Lead + Product Manager

| # | Verificação | Critério |
|---|------------|----------|
| 1 | SLOs de 24h | API availability ≥ 99.5%, p95 ≤ 500ms, email delivery ≥ 95% |
| 2 | Incidentes gerados | Zero Critical; High resolvidos com RCA iniciado |
| 3 | Feedback de usuários internos | Coletado via canal `#rs-feedback` |
| 4 | Agentes de IA em Supervisão Total | Zero decisões autônomas sem aprovação humana nas primeiras 24h |
| 5 | DLQ acumulada | Zero mensagens não processadas ao final das 24h |
| 6 | Decisão de escalar acesso | Product Manager decide baseado nos sinais coletados |

### 7.4 Critério de encerramento do war room

O war room (`#rs-war-room`) é encerrado quando **todos** os critérios abaixo forem atingidos:

- [ ] 24h desde o deploy sem incidente Critical ou High aberto
- [ ] SLOs de disponibilidade e latência atingidos nas últimas 12h
- [ ] DLQ zerada
- [ ] Sentry sem novos erros Critical nas últimas 12h
- [ ] Supervisão Total de IA ativa e sem rejeições manuais pendentes
- [ ] Tech Lead publica mensagem formal de encerramento no `#rs-war-room`

---

## 8. Deploy e Rollback

### 8.1 Procedimento de deploy em produção

```bash
# 1. Criar tag de release no GitHub (trigger do deploy.yml)
git tag -a v1.0.0 -m "Release v1.0.0 — go-live inicial"
git push origin v1.0.0

# 2. Acompanhar GitHub Actions workflow deploy.yml
# Job order: typecheck → lint → test → migrate → deploy-api → deploy-spa → health-check

# 3. Verificar migration aplicada
railway logs --tail | grep -E "migration|error"

# 4. Verificar health check pós-deploy
curl -s https://api.repasseseguro.com.br/health | jq .

# 5. Verificar SPA publicada
curl -I https://app.repasseseguro.com.br | grep "HTTP"
```

**Aprovação obrigatória:** O job `deploy-production` no GitHub Actions requer aprovação do Tech Lead no ambiente `production` (GitHub Environments) antes de executar.

---

### 8.2 Triggers de rollback

Executar rollback **imediatamente** se qualquer uma das condições abaixo for verdadeira nos primeiros 15 minutos após deploy:

| Condição | Threshold | Fonte de observação |
|----------|-----------|---------------------|
| Error rate 5xx | > 2% por > 5min | Railway Metrics |
| Health check falha | Qualquer dep crítica (`db`, `redis`) retorna `degraded` | `/health` endpoint |
| Fluxo crítico quebrado | Login 2FA ou Criação de Caso com erro | Smoke test manual |
| Sentry Critical | Novo erro Critical não existente em staging | Sentry Issues |
| Migration corrompeu dados | Qualquer inconsistência detectada em dados de produção | Backend Lead |

[DECISÃO AUTÔNOMA]: rollback automático acionado com error rate > 2% nos primeiros 15min — Justificativa: threshold conservador que protege UX e transações financeiras sem bloquear releases estáveis | Alternativa descartada: threshold de 5% — tolerância alta demais para produto com dados financeiros e transações de cessão imobiliária.

---

### 8.3 Procedimento de rollback

**Rollback de API (Railway) — estimativa: < 1 minuto**

```bash
# Via Railway Dashboard (preferencial — mais rápido)
# 1. Acessar Railway Dashboard → Projeto → Deployments
# 2. Identificar deploy anterior estável
# 3. Clicar em "Rollback to this deployment"
# 4. Confirmar rollback

# Via Railway CLI (alternativa)
railway rollback
```

Verificação após rollback:
```bash
curl -s https://api.repasseseguro.com.br/health | jq .
# Esperado: {"status":"ok","db":"ok","redis":"ok","rabbitmq":"ok"}
```

**Rollback de SPA (Vercel) — estimativa: < 30 segundos**

```bash
# Via Vercel CLI
vercel rollback

# Via Vercel Dashboard
# 1. Vercel Dashboard → Projeto → Deployments
# 2. Deploy anterior estável → "Promote to Production"
```

**⚠️ CRÍTICO — Migrations não são revertidas automaticamente**

Migrations de banco de dados **não fazem parte** do rollback de aplicação. Se uma migration corrompeu dados:
1. Interromper rollback de código imediatamente
2. Acionar PITR (ver §10) se dados foram destruídos
3. NÃO executar migration de rollback sem aprovação do Tech Lead e análise de impacto

---

### 8.4 Validação pós-rollback

| Verificação | Comando / URL | Critério |
|-------------|---------------|----------|
| API health | `curl https://api.repasseseguro.com.br/health` | HTTP 200, todas deps `ok` |
| SPA acessível | Abrir URL de produção no browser | Carrega sem erro |
| Login funcional | Teste manual de login com usuário de staging | Login + 2FA bem-sucedidos |
| Error rate | Railway Metrics | < 1% após 5min do rollback |

---

## 9. Incidentes por Categoria

### 9.1 Backend (API Railway)

| Sintoma | Triagem imediata | Ação inicial |
|---------|-----------------|--------------|
| Erros 5xx > 2% | `railway logs --tail` + Sentry Issues | Verificar último deploy; se pós-deploy: rollback |
| API não responde | `curl /health` retorna timeout ou 503 | Verificar Railway dashboard status; reiniciar serviço se necessário |
| Latência p95 > 2s | Railway Metrics → verificar CPU/memória | Verificar queries lentas no Supabase; checar Redis hit rate |
| NestJS Exception não tratada | Sentry: erro sem RFC 7807 | Hotfix imediato via branch `hotfix/`; deploy express |

### 9.2 Banco de Dados (Supabase)

| Sintoma | Triagem imediata | Ação inicial |
|---------|-----------------|--------------|
| DB connection timeout | Supabase Dashboard → Connection Pool | Checar queries longas; `SELECT * FROM pg_stat_activity WHERE state = 'active'` |
| Pool esgotado (> 95%) | Supabase Dashboard → Metrics | Fechar conexões ociosas; otimizar queries com N+1 |
| Migration com erro | `railway logs` pós-deploy | Parar deploy imediatamente; acionar Tech Lead; avaliar PITR |
| Dados corrompidos | Verificação manual pelo Backend Lead | Acionar PITR (§10); não prosseguir sem validação |

### 9.3 Cache (Redis / Upstash)

| Sintoma | Triagem imediata | Ação inicial |
|---------|-----------------|--------------|
| Redis indisponível | Upstash console + `/health` retorna `redis: degraded` | Ativar modo degradado — JWT stateless sem session Redis; sem cache de config de IA |
| Hit rate < 20% | Upstash console → Stats | Verificar TTL dos snapshots; re-popular cache manualmente |
| Memória > 90% | Upstash console → Memory | Revisar TTLs; executar `redis-cli FLUSHDB` apenas com aprovação do Tech Lead |

**Modo degradado com Redis offline:** Sistema opera com JWT stateless (sem invalidação por logout), cache desativado, agentes de IA sem cache de contexto. Performance degradada mas funcional. Ver D26 §5.2.

### 9.4 Fila (RabbitMQ / CloudAMQP)

| Sintoma | Triagem imediata | Ação inicial |
|---------|-----------------|--------------|
| DLQ com mensagens | CloudAMQP Management UI → DLQ | Inspecionar mensagem; corrigir causa; reprocessar via `amqp-tools` |
| Workers offline | CloudAMQP → Consumers = 0 | Reiniciar serviço Railway; verificar crash do consumer |
| Fila com profundidade > 500 | CloudAMQP → Queue depth | Verificar se workers estão processando; escalar horizontalmente se necessário |
| Notificações não entregues | `notification_logs` com status `FAILED` | Checar workers + provider externo (Resend/Meta/Twilio) |

### 9.5 Autenticação

| Sintoma | Triagem imediata | Ação inicial |
|---------|-----------------|--------------|
| 100% de falhas de login | Sentry + `railway logs` | Verificar JWT_SECRET em produção; verificar Supabase Auth |
| Refresh token inválido | Logs de `TokenExpiredError` | Verificar rotação de secrets; forçar logout se necessário |
| 2FA com falha | Logs de `AUTH_002` | Verificar integração com TOTP; verificar clock skew entre servidores |
| Brute force detectado | Alerta Sentry (≥ 10 falhas em 60s) | Rate limit ativo; bloquear IP se necessário; checar `fail2ban` config |

### 9.6 Integrações Externas

| Serviço | Sintoma | Triagem | Ação |
|---------|---------|---------|------|
| ZapSign | Erro ao criar documento | `curl -X POST https://app.zapsign.com.br/api/v1/docs` com token | Verificar token; se indisponível: modo sem assinatura (bloqueado por regra de negócio) |
| Resend | E-mails não entregues | Resend dashboard → Delivery | Verificar domínio SPF/DKIM; fallback para log de e-mail pendente |
| Meta Cloud API | WhatsApp não enviado | Meta Business Manager | Fallback automático para SMS (Twilio) |
| Twilio | SMS não enviado | Twilio console | Verificar saldo e número de origem; notificação falha entra em DLQ |

### 9.7 Frontend (SPA Vercel)

| Sintoma | Triagem imediata | Ação inicial |
|---------|-----------------|--------------|
| SPA com tela branca | Vercel logs + DevTools console | Verificar build errors; rollback Vercel se pós-deploy |
| Assets 404 | Vercel deployment details | Verificar build output e rotas de SPA |
| Error boundary disparado | Sentry Issues (frontend) | Identificar componente e rota; hotfix se crítico |
| Web Vitals degradados | Vercel Analytics + Lighthouse | Verificar bundle size; code split se necessário |

---

## 10. Backup e Restore

### 10.1 Estratégia de backup

| Tipo | Frequência | Armazenamento | Retenção |
|------|------------|---------------|----------|
| Backup automático Supabase | Diário | Supabase infraestrutura gerenciada | 7 dias (plano Pro) |
| PITR (Point-in-Time Recovery) | Checkpoint a cada hora | Supabase infraestrutura gerenciada | [DADO PENDENTE — verificar plano Supabase contratado] |
| Export manual pré-deploy | Antes de cada deploy com migration | Storage externo definido pelo DevOps | Indefinido (manter ao menos 3 exports) |

### 10.2 Quando optar por restore versus rollback

| Situação | Decisão | Justificativa |
|----------|---------|---------------|
| Deploy causou bug mas dados estão íntegros | **Rollback de código** (Railway/Vercel) | Mais rápido (< 1min); sem risco de perda de dados recentes |
| Migration corrompeu dados em produção | **PITR + rollback de código** | Dados destruídos não voltam com rollback de código |
| Perda acidental de registros em tabela | **PITR** | Recupera estado exato de um ponto no tempo |
| Tabela dropada por acidente | **PITR** | Única forma de recuperar estrutura e dados |

### 10.3 Procedimento de restore via PITR

```sql
-- 1. Acessar Supabase Dashboard → Settings → Database → Point in Time Recovery
-- 2. Selecionar o ponto de restore (anterior ao incidente)
-- 3. Confirmar restore — ATENÇÃO: operação irreversível, sobrescreve dados atuais

-- Após restore, verificar integridade:
SELECT COUNT(*) FROM cases WHERE created_at > '2025-01-01';
SELECT COUNT(*) FROM escrow_accounts;
SELECT COUNT(*) FROM users;

-- 4. Atualizar DATABASE_URL no Railway se o endpoint do Supabase mudar após restore
-- 5. Reiniciar serviços Railway para reconectar pool de conexões
railway redeploy
```

**Tempo estimado de restore:** 15–30 minutos. Plataforma permanece em manutenção durante o processo.

**Ponto de recuperação:** Último checkpoint PITR antes do incidente. Dados criados entre o checkpoint e o incidente são perdidos.

### 10.4 Comunicação durante restore

- Ativar banner de manutenção na SPA antes de iniciar o restore.
- Postar atualização a cada 10 minutos no canal `#rs-war-room`.
- Comunicar ETA de retorno ao Product Manager para comunicação externa.

---

## 11. Comunicação de Incidente e Lançamento

### 11.1 Comunicação interna — templates

**Abertura de incidente (Slack `#rs-incidents`)**

```
🔴 [INCIDENTE ABERTO] — Severidade: {Critical/High/Medium}
Produto: Repasse Seguro
Início: {HH:mm} (Fortaleza)
Sintoma: {descrição curta}
Impacto: {quem é afetado e o que não funciona}
Responsável: @{nome}
Status: Investigando
War room: #rs-war-room
```

**Atualização de incidente (a cada 30min para Critical, 1h para High)**

```
🔄 [ATUALIZAÇÃO DE INCIDENTE] — {HH:mm}
Status: {Investigando / Mitigando / Monitorando}
Situação atual: {o que foi descoberto}
Próximos passos: {o que está sendo feito}
ETA de resolução: {estimativa ou "indefinido"}
```

**Encerramento de incidente**

```
✅ [INCIDENTE RESOLVIDO] — {HH:mm}
Duração: {X horas Y minutos}
Causa raiz: {descrição técnica resumida}
Ação tomada: {rollback / hotfix / restore / configuração}
Próximos passos: {post-mortem agendado / issue aberta}
Post-mortem: agendado para {data}
```

---

### 11.2 Comunicação de lançamento — templates

**Anúncio interno (Slack `#rs-geral`) — após go-live confirmado**

```
🚀 Repasse Seguro está no ar!

O lançamento foi concluído com sucesso às {HH:mm}.
Acesso: https://app.repasseseguro.com.br

Os fluxos críticos foram validados:
✅ Login com 2FA
✅ Criação de caso
✅ Dashboard Admin
✅ Notificações

Monitoramento intensivo ativo pelas próximas 24h.
Dúvidas ou problemas: #rs-war-room
```

**Comunicação para usuários-piloto** — [DADO PENDENTE: template de e-mail para usuários convidados para o piloto. PM deve preparar com base no tom e voz da marca D03.]

**Comunicação pública (quando definida)** — [DADO PENDENTE: canais e timing de comunicação pública não definidos nos inputs disponíveis. PM deve definir estratégia de lançamento público.]

---

### 11.3 Frequência de comunicação por severidade

| Severidade | Frequência de atualização | Canal | Responsável |
|------------|--------------------------|-------|-------------|
| Critical | A cada 30 minutos | `#rs-incidents` + stakeholders diretos | Tech Lead |
| High | A cada 1 hora | `#rs-incidents` | Backend Lead |
| Medium | A cada 4 horas | `#rs-alertas` | Desenvolvedor de plantão |
| Low | Quando resolvido | `#rs-dev` | Desenvolvedor |

---

## 12. Critério de Go/No-Go

### 12.1 Responsável pela decisão

A decisão de Go/No-Go é do **Tech Lead** com input do DevOps (sinais técnicos) e Product Manager (sinais de negócio). Não é democrática — um veto técnico do Tech Lead é suficiente para No-Go.

### 12.2 Checkpoints de go/no-go

| Checkpoint | Momento | Responsável | O que avalia |
|------------|---------|-------------|-------------|
| **GO/NO-GO Inicial** | 10h00 do launch day | Tech Lead | Todos os itens de T-7, T-3, T-1 verificados; ambientes OK; time presente |
| **GO/NO-GO Pós-Migration** | Após migrations aplicadas (≈10h25) | Tech Lead + Backend Lead | Zero erros de migration; dados íntegros; rollback validado disponível |
| **GO/NO-GO Pós-Smoke Test** | Após smoke test (≈11h00) | Tech Lead + QA | 100% dos 5 fluxos críticos passando em produção |

### 12.3 Critérios objetivos de NO-GO

Qualquer um dos itens abaixo resulta em **NO-GO automático**:

| Condição | Momento | Ação imediata |
|----------|---------|---------------|
| Algum item 🔴 de D28 em aberto | T-1 | Resolver antes de avançar; adiar go-live |
| Migration com erro em produção | Pós-migration | Parar deploy; acionar PITR se necessário |
| Health check com dep `degraded` após deploy | Pós-deploy | Rollback de código imediato |
| Smoke test com falha em fluxo crítico | Pós-smoke test | Rollback de código imediato |
| Error rate > 2% nos primeiros 15min | T+15min | Rollback automático |
| Tech Lead veta por qualquer razão técnica | Qualquer momento | Adiar; comunicar no `#rs-war-room` |

### 12.4 O que acontece em caso de NO-GO

1. Tech Lead declara NO-GO com justificativa explícita no `#rs-war-room`.
2. DevOps executa rollback se já houve deploy (§8.3).
3. Product Manager comunica stakeholders sobre adiamento (sem detalhes técnicos para externos).
4. Tech Lead define próxima janela de tentativa (mínimo T+1 dia após correção e nova validação em staging).
5. Post-mortem de NO-GO: reunião no mesmo dia para analisar causa e plano de correção.

---

## 13. Backlog de Pendências

| # | Item | Categoria | Prioridade | O que falta |
|---|------|-----------|------------|-------------|
| P01 | Definir responsáveis por função com nome e contatos (§3.1) | Escalation | Alta | Nomes e contatos do time não disponíveis nos inputs |
| P02 | Confirmar plano Supabase e disponibilidade de PITR (§10.1) | Backup | Alta | Verificar plano contratado — PITR requer Supabase Pro ou superior |
| P03 | Preparar template de comunicação para usuários-piloto (§11.2) | Comunicação | Média | PM deve preparar e-mail de convite para piloto com identidade visual D03 |
| P04 | Definir estratégia e timing de comunicação pública (§11.2) | Comunicação | Média | PM deve definir canais (redes sociais, site, e-mail) e data de anúncio público |
| P05 | Configurar banner de manutenção na SPA (§10.4) | Infraestrutura | Média | Implementar componente de manutenção no frontend para uso durante restore |
| P06 | Definir rotação de on-call por turno nas primeiras 24h (§3.1 + §7) | Operação | Alta | Nomes e horários de plantão não definidos |
| P07 | Validar endpoint `/health/ready` implementado na API (§4.1) | Infraestrutura | Alta | Verificar se `/health/ready` existe ou apenas `/health` |

---

*Documento gerado pelo pipeline ShiftLabs v9.5 — módulo Admin, Repasse Seguro.*
