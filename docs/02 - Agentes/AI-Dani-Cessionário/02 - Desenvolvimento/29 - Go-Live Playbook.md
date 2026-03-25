# 29 - Go-Live Playbook
## AI-Dani-Cessionário — Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Produto** | AI-Dani-Cessionário |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Status** | Aprovado — pronto para execução |
| **Bloco** | 7 — Go-Live |

---

> 📌 **TL;DR — Go-Live Operacional**
>
> O lançamento do AI-Dani-Cessionário é uma operação controlada em 4 fases: preparação (T-7 a T-1), launch window (Seg–Sex 09h–12h), estabilização intensiva (T+15min a T+24h), e encerramento do war room. Go/no-go é decidido pelo Tech Lead com base em critérios objetivos (zero P0 open, cobertura ≥80%, E2E verdes, dependências operacionais validadas). Rollback automático se health check falha por >3min; manual em menos de 5min via Railway CLI. On-call obrigatório por 48h após go-live. Nenhum improviso — cada etapa tem dono, horário e critério de sucesso.

---

## 1. Matriz de Escalation

| **Severidade** | **Impacto** | **Canal** | **Pessoas Acionadas** | **SLA de Resposta** | **Critério de Escalação** |
|---|---|---|---|---|---|
| **P0 — Crítico** | Produção totalmente indisponível ou dado financeiro incorreto | `#dani-incidents` + DM on-call + DM CEO tech | On-call, Tech Lead, CEO tech | 5 minutos | Qualquer ocorrência imediata |
| **P1 — Alto** | Funcionalidade crítica degradada (chat, calculadora, OTP) | `#dani-incidents` | On-call, Tech Lead | 15 minutos | Não resolvido em 30min → escalar para CEO tech |
| **P2 — Médio** | Feature não-crítica falhando (notificações, alertas In-App) | `#dani-deployments` | On-call | 1 hora | Não resolvido em 4h → escalar para Tech Lead |
| **P3 — Baixo** | Degradação cosmética, erro isolado | `#dani-dev` | Dev responsável | 4 horas úteis | Não resolvido no sprint → backlog |
| **Go-Live Bloqueado** | Critério de go/no-go não atingido | `#dani-incidents` | Tech Lead + CEO tech | Imediato | Reagendamento obrigatório |

**[DECISÃO AUTÔNOMA]:** On-call rotation com rotação semanal entre membros do time de engenharia — Justificativa: distribui responsabilidade e evita burnout; rotação documentada em `docs/02 - Agentes/.../on-call.md` | Alternativa descartada: Tech Lead sempre on-call — concentração de responsabilidade e ponto único de falha humano.

---

## 2. Health Checks

### 2.1 Endpoints Críticos

| **Endpoint** | **Método** | **Threshold** | **Frequência** | **Responsável** | **Interpretação de Falha** |
|---|---|---|---|---|---|
| `GET /health` | HTTP | 200 em <500ms | A cada 30s (Railway) | Automático | Reiniciar serviço; se persistir por >3min → rollback |
| `POST /api/v1/agente/chat` (smoke) | HTTP | 200 ou 401 em <2s | Pós-deploy manual | On-call | Se 5xx → rollback imediato |
| `POST /api/v1/calculadora/simular` (smoke) | HTTP | 200 ou 401 em <500ms | Pós-deploy manual | On-call | Se 5xx → rollback imediato |
| `POST /api/v1/auth/otp/solicitar` (smoke) | HTTP | 200 ou 429 em <1s | Pós-deploy manual | On-call | Se 5xx → rollback imediato |
| Redis `PING` | CLI | PONG em <100ms | Checklist diário | On-call | Se falha → verificar ALT-004 (D26) |
| RabbitMQ queue depth | Management UI | `dani.notificacoes` < 100 mensagens | Checklist diário | On-call | Se > 100 → verificar consumer; se DLQ > 50 → ALT-005 |

### 2.2 Dashboards Obrigatórios para Launch Day

```
1. Sentry: https://sentry.io/organizations/{ORG}/issues/ — erro rate em tempo real
2. Langfuse: https://cloud.langfuse.com — latência LLM p95, custo por hora
3. Railway: https://railway.app/project/{ID} — CPU/memória API em tempo real
4. RabbitMQ Management: https://rabbitmq.repasse-seguro.com.br:15672 — filas e DLQ
```

### 2.3 Interpretação de Thresholds Críticos

| **Métrica** | **Normal** | **Alerta (P2)** | **Crítico (P1)** | **Rollback (P0)** |
|---|---|---|---|---|
| HTTP 5xx rate | < 0.5% | 1–2% | 2–5% | > 5% por 2min |
| Latência p95 chat | < 3s | 3–5s | 5–8s | > 10s ou sem resposta |
| Redis latência | < 10ms | 10–50ms | 50–200ms | > 200ms ou timeout |
| DLQ mensagens | 0–5 | 5–50 | 50–200 | > 200 |
| Error rate OTP | < 5% | 5–15% | 15–30% | > 30% (possível ataque) |

---

## 3. Pré-Launch

### 3.1 T-7 dias (1 semana antes)

**Responsável:** Tech Lead

```
[ ] Confirmar que todos os 29 documentos do pipeline estão gerados e aprovados
[ ] Confirmar que release candidate (RC) está criado: git tag v{X}.{Y}.{Z}-rc.1
[ ] Deploy do RC em staging executado e estável por 24h
[ ] Executar suite completa de testes em staging: unitário + integração + E2E + contrato
[ ] Verificar que cobertura unitária ≥80% e integração ≥70%
[ ] Todos os 10 fluxos E2E obrigatórios (E2E-001 a E2E-010) verdes
[ ] Auditoria de acessibilidade: zero violações axe-core critical/serious em todas as telas
[ ] `pnpm audit` — zero CVEs critical ou high
[ ] Confirmar que variáveis de ambiente de produção estão configuradas no Railway
[ ] Confirmar que secrets de produção estão no GitHub Actions Secrets
[ ] Confirmar que backups automáticos do Supabase estão habilitados
[ ] Confirmar que canais Slack estão criados: #dani-incidents, #dani-releases, #dani-deployments
[ ] Confirmar que on-call para os 7 dias pós-launch está definido
[ ] Freeze de código: comunicar ao time que nenhum PR será mergeado em develop durante T-1
```

### 3.2 T-3 dias (3 dias antes)

**Responsável:** Tech Lead + DevOps

```
[ ] Executar smoke tests manuais em staging cobrindo todos os fluxos críticos
[ ] Testar fluxo completo de vinculação WhatsApp (solicitar OTP → verificar → confirmar)
[ ] Testar calculadora com valores reais de oportunidade
[ ] Testar isolamento: criar 2 usuários de staging, confirmar que não veem dados um do outro
[ ] Verificar que Langfuse está capturando traces com userId = sha256 (sem UUID raw)
[ ] Verificar que Sentry está configurado com environment = "production" para a versão RC
[ ] Confirmar que URLs de produção estão corretas em smoke-tests.sh
[ ] Executar smoke-tests.sh contra ambiente de staging com URLs de produção (para testar o script)
[ ] Preparar mensagem de lançamento para stakeholders internos
[ ] Confirmar acesso de todos os membros do time on-call a: Railway, Vercel, Supabase, Sentry, Langfuse
[ ] Confirmar que comandos de rollback estão acessíveis e foram testados em staging
```

### 3.3 T-1 dia (véspera)

**Responsável:** Tech Lead

```
[ ] Freeze de código ativo: nenhum PR mergeado após esse ponto sem aprovação do Tech Lead
[ ] Confirmar versão final: git tag vX.Y.Z em main (substituir RC por release final)
[ ] Confirmar que CI de release.yml passou 100% com a tag final
[ ] Briefing do time on-call: revisar D26 (Runbook Operacional), confirmar matriz de escalation
[ ] Confirmar janela de deploy: amanhã {DATA} às 09h00 (America/Fortaleza)
[ ] Notificar stakeholders: "Deploy do AI-Dani-Cessionário amanhã às 09h00"
[ ] Confirmar que rollback é reproduzível: executar dry-run do comando em staging
[ ] Verificar que go/no-go checklist será executado amanhã às 08h30
[ ] Confirmar que nenhum incidente P0/P1 está aberto em staging
```

---

## 4. Launch Day

### 4.1 Cronograma da Janela de Launch

**Janela:** Seg–Sex, 09h00–12h00 (America/Fortaleza).

**[DECISÃO AUTÔNOMA]:** Janela das 09h–12h escolhida — Justificativa: tráfego baixo (cessionários tipicamente acessam após 13h), equipe de engenharia disponível, 3h de margem antes do pico para detectar e corrigir problemas | Alternativa descartada: deploy às 14h — coincide com pico de uso, janela de diagnóstico menor.

| **Horário** | **Atividade** | **Responsável** | **Critério de Sucesso** |
|---|---|---|---|
| **08h30** | Go/no-go meeting: checar todos os pré-requisitos | Tech Lead + DevOps | Todos os itens verdes — ver seção 12 |
| **09h00** | Notificação no Slack: "Deploy iniciando" | On-call | Mensagem enviada |
| **09h05** | Tag de release aplicada: `git tag vX.Y.Z && git push origin vX.Y.Z` | Tech Lead | CI `release.yml` dispara automaticamente |
| **09h10** | Acompanhar CI: aguardar stages 1–5 passarem | On-call | CI verde em todos os stages |
| **09h30** | Deploy automático em produção (Railway + Vercel) | Automático (CI) | Health check retorna 200 |
| **09h35** | Executar smoke tests manuais | On-call | Todos os checks passando |
| **09h40** | Checkpoint 1 — go/no-go pós-deploy | Tech Lead | Zero 5xx; health check ok; dependências ok |
| **09h45** | Publicar mensagem de lançamento | Tech Lead | Mensagem publicada em #dani-releases |
| **10h00** | Monitoramento intensivo T+30min | On-call | Sentry, Langfuse, RabbitMQ normais |
| **11h00** | Checkpoint 2 — T+1h | On-call | Latência p95 < 5s; 5xx < 0.5%; DLQ < 5 |
| **12h00** | Encerramento da janela de launch | Tech Lead | War room reduzido; on-call normal ativado |

### 4.2 Smoke Tests Manuais Pós-Deploy

```bash
# 1. Health check
curl -s https://api.repasse-seguro.com.br/health | jq
# Esperado: { "status": "ok", "version": "vX.Y.Z" }

# 2. Chat retorna 401 sem auth (endpoint protegido)
STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST https://api.repasse-seguro.com.br/api/v1/agente/chat)
echo "Chat sem auth: $STATUS"  # Esperado: 401

# 3. Calculadora retorna 401 sem auth
STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST https://api.repasse-seguro.com.br/api/v1/calculadora/simular)
echo "Calculadora sem auth: $STATUS"  # Esperado: 401

# 4. OTP solicitar retorna 200 (ou 429 se rate limit ativo)
STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST https://api.repasse-seguro.com.br/api/v1/auth/otp/solicitar \
  -H "Content-Type: application/json" -d '{"telefone": "+5511999990001"}')
echo "OTP solicitar: $STATUS"  # Esperado: 200 ou 429

# 5. Frontend carregou
curl -s -o /dev/null -w "%{http_code}" https://app.repasse-seguro.com.br
# Esperado: 200
```

### 4.3 Checkpoint de Go/No-Go Pós-Deploy (09h40)

Avaliar:
- `[ ]` HTTP 5xx rate < 0.5% nos últimos 10min (Sentry)
- `[ ]` Latência p95 chat < 5s (Langfuse)
- `[ ]` Redis `PING` retorna PONG
- `[ ]` RabbitMQ `dani.notificacoes` sem acúmulo anormal
- `[ ]` Nenhum alerta P0/P1 disparado em Sentry
- `[ ]` Smoke tests manuais todos passando

Se algum item falhar → **acionamento de rollback imediato** (seção 8).

---

## 5. Pós-Launch

### 5.1 T+15 minutos

**Responsável:** On-call (ativo no war room)

```
[ ] Sentry: zero erros P0/P1 novos desde deploy
[ ] Langfuse: latência média < 3s; nenhum timeout de LLM
[ ] Railway: CPU/memória dentro do normal (< 70%)
[ ] Redis: resposta < 10ms; sem keys expiradas indevidas
[ ] RabbitMQ: DLQ com 0–5 mensagens
[ ] Frontend (Vercel): 0 erros de JavaScript no console (DevTools)
[ ] Primeiro cessionário real atendido com sucesso (log de sessão confirmado)
```

### 5.2 T+1 hora

**Responsável:** On-call + Tech Lead

```
[ ] Taxa de erro 5xx: < 0.5% na última hora
[ ] Latência p95 chat: < 5s na última hora
[ ] DLQ: zero acúmulo anormal
[ ] Custo LLM na primeira hora: dentro do baseline projetado (Langfuse)
[ ] Nenhum cessionário reportou problema por canal de suporte
[ ] Rate limit de OTP não disparou em volume anormal
[ ] Agente respondendo corretamente (verificar 3 sessões aleatórias no Langfuse)
[ ] Publicar atualização: "T+1h — sistema estável" no #dani-releases
```

### 5.3 T+24 horas

**Responsável:** Tech Lead

```
[ ] Review completo de todos os erros do Sentry nas últimas 24h
[ ] Review de custo LLM: projeção mensal dentro do orçamento?
[ ] Nenhum incidente P0/P1 ocorreu (ou foi resolvido com post-mortem iniciado)
[ ] Feedback dos primeiros cessionários coletado (se disponível)
[ ] DLQ processada completamente; fila normal
[ ] Backup do Supabase das últimas 24h confirmado no Dashboard
[ ] Encerrar war room formal: equipe volta ao modo normal de operação
[ ] Agendar retrospectiva do launch (48h após go-live)
[ ] Publicar: "AI-Dani-Cessionário v{X}.{Y}.{Z} — 24h estável" no #dani-releases
```

---

## 6. Deploy e Rollback

### 6.1 Deploy de Produção — Passo a Passo

```bash
# 1. Confirmar que branch main está atualizada
git checkout main && git pull origin main

# 2. Confirmar que CI de develop → main passou
# Verificar GitHub Actions: release.yml deve estar verde

# 3. Criar tag semver
git tag vX.Y.Z
git push origin vX.Y.Z

# 4. CI `release.yml` dispara automaticamente com a tag
# Aguardar stages 1-5 + aprovação humana (environment: production, 2 reviewers)

# 5. Railway deploy automático após aprovação do CI
# Acompanhar em: https://railway.app/project/{ID} → dani-api → Deployments

# 6. Vercel deploy automático (após merge em main)
# Acompanhar em: https://vercel.com/team/{TEAM}/dani

# 7. Health check automático pelo Railway (3min)
# Se falhar → rollback automático Railway

# 8. Executar smoke tests manuais (seção 4.2)
```

### 6.2 Rollback — Triggers

Acionar rollback **imediatamente** quando:
- Health check falha por >3 minutos.
- HTTP 5xx rate > 5% por 2 minutos.
- Qualquer suspeita de falha de isolamento (ALT-003 do D26).
- Smoke test manual falha pós-deploy.

### 6.3 Rollback — Comandos

```bash
# API (Railway) — tempo estimado: 2–3min
railway rollback --service dani-api

# Frontend (Vercel) — tempo estimado: <2min
vercel rollback https://app.repasse-seguro.com.br

# Verificar versão após rollback
curl -s https://api.repasse-seguro.com.br/health | jq '.version'
# Deve retornar versão anterior

# Smoke tests pós-rollback
bash smoke-tests.sh https://api.repasse-seguro.com.br
```

### 6.4 Validação Pós-Rollback

```
[ ] Health check retorna 200 com versão anterior
[ ] Smoke tests manuais passando
[ ] 5xx rate voltando a < 0.5%
[ ] Comunicar rollback no Slack: "🔴 Rollback executado — versão vX.Y.Z revertida para vX.Y.{Z-1}"
[ ] Abrir incidente e iniciar post-mortem
[ ] Janela de deploy encerrada — próxima tentativa somente após diagnóstico
```

---

## 7. Incidentes por Categoria

| **Categoria** | **Sintoma** | **Triagem** | **Ação Inicial** |
|---|---|---|---|
| **Backend — 5xx massivo** | Todos os endpoints retornando 500 | Verificar railway logs; deploy recente? | Rollback imediato se deploy recente |
| **Banco (Supabase)** | Queries com timeout; `INFRA_DATABASE_UNAVAILABLE` | `curl https://status.supabase.com`; testar `psql $DATABASE_URL -c "SELECT 1"` | Aguardar resolução Supabase; ativar modo manutenção se necessário |
| **Cache (Redis)** | Sessões perdidas; rate limit não funcionando | `redis-cli -u $REDIS_URL ping`; verificar memória | Reiniciar container Redis; degradação controlada (seção 10.1 D26) |
| **Fila (RabbitMQ)** | Notificações não entregues; DLQ acumulando | RabbitMQ Management: consumer count; inspecionar 1 mensagem DLQ | Reiniciar worker; reprocessar DLQ após fix |
| **Autenticação (OTP)** | OTP não enviado; hard block massivo | Verificar EvolutionAPI; `redis-cli KEYS 'dani:block:otp:*' \| wc -l` | Circuit breaker? → aguardar. Ataque? → bloquear IPs no Railway |
| **Integração externa (OpenAI)** | Chat sem resposta; `AGENTE_LLM_UNAVAILABLE` | `curl https://status.openai.com/api/v2/status.json` | Ativar FallbackBanner: `redis-cli SET dani:status:agent "degraded"` |
| **Frontend** | JavaScript error no console; tela branca | Verificar Sentry erros de frontend; versão Vercel correta? | Rollback Vercel se deploy recente; comunicar time |

---

## 8. Backup e Restore

### 8.1 Frequência e Armazenamento

| **Tipo** | **Frequência** | **Retenção** | **Onde** |
|---|---|---|---|
| Supabase PITR | Contínuo (WAL) | 7 dias (free) | Supabase Dashboard → Backups |
| Supabase Snapshot | Diário | 7 snapshots | Supabase Dashboard → Backups |
| Redis RDB | A cada 15min | 3 snapshots | Railway (automático) |

### 8.2 Restore vs. Rollback — Quando Usar Cada Um

| **Situação** | **Ação Correta** |
|---|---|
| Bug de código causando erro em produção | Rollback de código (Railway/Vercel) |
| Dados apagados ou corrompidos por bug | Restore Supabase (PITR ou snapshot) |
| Migration aplicada com erro de schema | Rollback de código + avaliar restore de banco |
| Performance degradada (sem perda de dados) | Rollback de código; sem restore necessário |

### 8.3 Restore do Supabase — Passo a Passo

```
1. Acessar Supabase Dashboard → Database → Backups
2. Selecionar ponto de restauração
   — PITR: escolher horário exato anterior ao incidente
   — Snapshot: escolher snapshot diário
3. Clicar "Restore" → aguardar 10–30min
4. Após restore:
   a. Executar: psql $DATABASE_URL -c "SELECT COUNT(*) FROM oportunidades;"
   b. Verificar contagem vs. baseline conhecido
   c. Testar isolamento com 2 cessionários de staging
5. Comunicar stakeholders com impacto (dados entre X e Y podem estar desatualizados)
```

**Ponto de recuperação:** PITR permite restaurar até o segundo anterior ao incidente — máximo de perda aceitável: 15 minutos de dados.

---

## 9. Comunicação de Incidente e Lançamento

### 9.1 Template — Notificação de Lançamento (T-30min)

```
🚀 AI-Dani-Cessionário — Deploy Iniciando

Versão: vX.Y.Z
Início estimado: {HH:MM} (America/Fortaleza)
Impacto esperado: nenhum (zero downtime)
Janela de observação: {HH:MM}–{HH:MM}
Responsável: {NOME}
Rollback disponível: sim (< 5min)

Acompanhe em: #dani-deployments
```

### 9.2 Template — Atualização Durante Incidente

```
[INCIDENTE EM ANDAMENTO — {HH:MM}]
Severidade: P{N}
Status: Investigando / Contendo / Resolvendo
Impacto atual: {N usuários afetados / funcionalidade X indisponível}
Última ação: {O que foi feito}
Próxima ação: {O que vai ser feito}
ETA: {HORÁRIO ou "Em investigação"}
Responsável: {NOME}
```

### 9.3 Template — Encerramento de Lançamento

```
✅ AI-Dani-Cessionário v{X}.{Y}.{Z} — Go-Live Concluído

Horário: {HH:MM}
Status: Estável (T+{N}min)
Health check: ✅ Passando
Taxa de erro: {X}% (baseline normal)
Latência p95: {X}s

Release Notes: {URL}
Próxima janela de estabilização: T+24h ({DATA} {HH:MM})
On-call ativo: {NOME}
```

### 9.4 Frequência de Comunicação por Severidade

| **Severidade** | **Frequência** | **Canal** |
|---|---|---|
| P0 | A cada 15min até resolução | `#dani-incidents` + DM CEO tech |
| P1 | A cada 30min | `#dani-incidents` |
| P2 | A cada 2h | `#dani-deployments` |
| Launch normal | T-30min, pós-deploy, T+1h, T+24h | `#dani-releases` |

---

## 10. Critério de Go/No-Go

### 10.1 Quem Decide

**Decisor:** Tech Lead.

**Participantes:** Tech Lead + DevOps + (opcional) representante de produto.

**Momento:** reunião às 08h30 do dia do launch (30min antes do deploy).

### 10.2 Critérios Objetivos — Verde = Go, Vermelho = No-Go

| **Critério** | **Gate** | **Verificação** |
|---|---|---|
| Testes unitários | Cobertura ≥80%; zero falhas | CI report do RC |
| Testes de integração | Cobertura ≥70%; zero falhas P0 | CI report do RC |
| E2E obrigatórios | 10/10 fluxos E2E-001 a E2E-010 verdes | Playwright report |
| Acessibilidade | Zero violações axe-core critical/serious | axe-core report do RC |
| Segurança | Zero CVEs critical/high | `pnpm audit` output |
| Dependências operacionais | OpenAI, Supabase, EvolutionAPI operacionais | Status pages verificadas às 08h30 |
| Staging estável | Sem P0/P1 em staging nas últimas 24h | Sentry staging |
| Checklist D28 | Todos os itens bloqueantes `[x]` | Documento preenchido |
| Rollback testado | Dry-run executado em staging (T-1) | Evidência no D24 |
| On-call definido | Pessoa com acesso a Railway, Vercel, Sentry | Confirmação verbal ou no Slack |
| Janela de deploy | Seg–Sex 09h–12h; sem feriados ou eventos críticos | Calendário verificado |

### 10.3 Decisão de No-Go — O que Acontece

Se qualquer critério vermelho:
1. Deploy adiado imediatamente — nenhuma tag criada.
2. Comunicar stakeholders: "Go-live adiado — {motivo em 1 frase}".
3. Resolver o item bloqueante em até 24h.
4. Reagendar para próxima janela disponível (mínimo T+24h).
5. Re-executar todos os checks antes do novo go/no-go.

**[DECISÃO AUTÔNOMA]:** No-go automático se taxa de erro > 2% nos primeiros 15min pós-deploy — Justificativa: threshold conservador que protege UX sem bloquear releases estáveis; produto com dados financeiros exige tolerância menor | Alternativa descartada: threshold de 5% — alta demais para produto com transações de comissão calculada.

### 10.4 Exemplo de Decisão de Go/No-Go

```
✅ Go-Live Autorizado — AI-Dani-Cessionário v1.0.0
Data: 23/03/2026 08h45
Decidido por: Tech Lead (nome)

Checklist:
✅ Cobertura unitária: 84% (≥80%)
✅ Cobertura integração: 73% (≥70%)
✅ E2E: 10/10 verdes
✅ Acessibilidade: 0 violações critical
✅ Segurança: 0 CVEs critical
✅ OpenAI status: operational
✅ Supabase status: operational
✅ Staging: 48h sem P0/P1
✅ Rollback: testado em staging (T-1)
✅ On-call: {NOME} confirmado

Deploy autorizado para 09h00.
```

---

## 11. Backlog de Pendências

| **ID** | **Tipo** | **Descrição** | **Impacto no Go-Live** |
|---|---|---|---|
| P-01 | PENDÊNCIA | Canais Slack não criados — `#dani-incidents`, `#dani-releases`, `#dani-deployments` precisam ser provisionados antes do T-7 | Bloqueante: sem canais, comunicação de incidente não é possível |
| P-02 | PENDÊNCIA | On-call rotation não definida — equipe e ferramenta de alerta (PagerDuty/OpsGenie/manual) precisam ser configuradas antes do T-7 | Bloqueante: SLA P0 de 5min inatingível sem on-call formal |
| P-03 | PENDÊNCIA | URLs de produção não confirmadas — placeholder `repasse-seguro.com.br` usado em todo o playbook | Bloqueante: smoke tests e comunicação usarão URLs incorretas se não confirmadas antes do T-3 |
| P-04 | PENDÊNCIA | Provisionamento de Railway (produção) e Vercel (produção) com team plan não confirmado — tokens de CI precisam ser de service accounts | Bloqueante: CI `release.yml` não pode rodar sem `RAILWAY_TOKEN_PROD` e `VERCEL_TOKEN` |
| P-05 | [DECISÃO AUTÔNOMA] | Status page pública não prevista no produto — omitida do playbook | Cessionários não terão visibilidade pública de incidentes; comunicação via Slack interno apenas |
| P-06 | PENDÊNCIA | Provedor de SMS para OTP (Twilio vs AWS SNS) não definido — afeta P-03 do checklist de pré-launch e smoke test de OTP via WhatsApp | Impacto em D+2 se OTP por SMS não funcionar no launch day |

---

*Documento gerado por Claude Code Desktop — Pipeline ShiftLabs v9.5 — AI-Dani-Cessionário*
