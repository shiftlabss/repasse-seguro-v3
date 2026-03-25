# 29 - Go-Live Playbook

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Tech Lead, DevOps, Coordenador RS, Engenharia |
| **Escopo** | Plano completo de go-live do módulo CRM — pré-requisitos, sequência de ativação, critérios de go/no-go, rollback e comunicação |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | 24 Deploy CI-CD · 25 Observabilidade · 26 Runbook · 27 Plano de Testes · 28 Checklist de Qualidade |

---

> **TL;DR**
>
> - **Janela de go-live:** segunda a sexta, 10h–16h (America/Fortaleza). Sem exceções sem aprovação de Tech Lead + Coordenador RS.
> - **Critério go/no-go:** decisão objetiva baseada em sinais de staging — não é opinião.
> - **Rollback:** em ≤ 10 min se error rate > 2% nos primeiros 15 min pós-deploy.
> - **War room:** ativo nas primeiras 48h após go-live.
> - **CRM é ferramenta interna** — impacto limitado à equipe RS, não a usuários finais.

---

## 1. Matriz de Escalada

| Severidade | Impacto | Canal | Pessoas acionadas | SLA de resposta |
|---|---|---|---|---|
| **Critical** | CRM fora do ar — equipe RS sem acesso | Slack `#crm-incidents` @channel + WhatsApp direto | Tech Lead + DevOps + Coordenador RS | 15 min |
| **High** | Feature crítica quebrada (pipeline, ZapSign, relatório) | Slack `#crm-incidents` + menção Tech Lead | Tech Lead + Backend Lead | 1 hora |
| **Medium** | Feature degradada, alertas de SLA atrasados | Slack `#crm-alertas` | Backend Lead | 4 horas |
| **Low** | Problema cosmético, analytics offline | Slack `#crm-dev` | Dev de plantão | 24 horas |

### 1.1 Responsáveis

| Papel | Contato | Backup |
|---|---|---|
| Tech Lead | [DADO PENDENTE] | [DADO PENDENTE] |
| Backend Lead | [DADO PENDENTE] | [DADO PENDENTE] |
| Frontend Lead | [DADO PENDENTE] | [DADO PENDENTE] |
| DevOps | [DADO PENDENTE] | [DADO PENDENTE] |
| Coordenador RS | [DADO PENDENTE] | [DADO PENDENTE] |
| On-call (plantão) | Rotativo semanal — ver escala | Tech Lead |

---

## 2. Pré-Requisitos de Infraestrutura

Antes de iniciar qualquer etapa do go-live, confirmar que toda a infraestrutura está provisionada e configurada:

| # | Pré-requisito | Responsável | Status |
|---|---|---|---|
| INF-01 | Projeto Supabase de produção criado com banco PostgreSQL 17 | DevOps | |
| INF-02 | Projeto Railway criado para `crm-api` com ambiente `production` | DevOps | |
| INF-03 | Projeto Vercel criado para `web-crm` com domínio `crm.repasseseguro.com.br` | DevOps | |
| INF-04 | Instância Upstash Redis de produção criada | DevOps | |
| INF-05 | Instância CloudAMQP de produção criada com filas CRM configuradas | DevOps | |
| INF-06 | Conta Resend de produção com domínio `repasseseguro.com.br` verificado | DevOps | |
| INF-07 | Conta ZapSign de produção com token de API gerado | Tech Lead | |
| INF-08 | Conta Celcoin de produção com HMAC secret configurado | Tech Lead | |
| INF-09 | Meta Cloud API configurada com número WhatsApp Business de produção | Tech Lead | |
| INF-10 | Sentry: projetos `crm-api` e `crm-frontend` criados em produção | DevOps | |
| INF-11 | PostHog: projeto `crm-production` criado | DevOps | |
| INF-12 | Todos os secrets de produção configurados no Railway e Vercel (ver Doc 22) | DevOps | |

---

## 3. Usuários Iniciais

Os usuários Admin RS e Coordenador RS devem ser criados antes do go-live para que a equipe possa acessar o CRM no D-0:

| # | Item | Responsável |
|---|---|---|
| USR-01 | Conta de `ADMIN_RS` criada no Supabase Auth de produção | Tech Lead |
| USR-02 | Conta de `COORDENADOR_RS` criada no Supabase Auth de produção | Tech Lead |
| USR-03 | Contas de `ANALISTA_RS` criadas para todos os Analistas da equipe RS | Coordenador RS |
| USR-04 | Contas de `PARCEIRO_EXTERNO` criadas para parceiros necessários ao go-live | Coordenador RS |
| USR-05 | Credenciais comunicadas individualmente de forma segura (nunca por e-mail não criptografado) | Tech Lead |

---

## 4. Sequência de Ativação (Timeline)

### 4.1 D-7 — Staging Final

**Responsável geral:** Tech Lead

| # | Ação | Dono | Critério de sucesso |
|---|---|---|---|
| 1 | Congelar `develop` — nenhum merge de feature nova; apenas bugfixes | Tech Lead | Branch `develop` com tag `release-candidate-v1.0.0-rc1` |
| 2 | Executar suite completa de testes em staging: unit + integration + E2E | QA | 100% dos fluxos críticos E2E passando; cobertura ≥ 80% em módulos críticos |
| 3 | Validar todas as variáveis de ambiente de produção (Railway + Vercel) | DevOps | Checklist do Doc 22 revisado; zero vars indefinidas |
| 4 | Confirmar limites e quotas das integrações: ZapSign (envelopes/mês), Meta (templates aprovados), Resend (envios/dia), Celcoin | DevOps | Confirmação por escrito no Slack `#crm-go-live` |
| 5 | Executar `pnpm audit` — zero vulnerabilidades High/Critical | Backend Lead | Saída do audit sem itens High ou Critical |
| 6 | Revisar Checklist de Qualidade (Doc 28) completo | Tech Lead + QA | Zero itens Bloqueantes ❌ em aberto |
| 7 | Validar backup: executar restore de teste do backup mais recente do Supabase staging | DevOps | Restore concluído em < 30 min; dados íntegros |
| 8 | Testar todos os webhooks em staging (ZapSign, Celcoin, Meta) | Backend Lead | Webhooks recebidos e processados corretamente |

---

### 4.2 D-3 — Smoke Tests

**Responsável geral:** QA + Backend Lead

| # | Ação | Dono | Critério de sucesso |
|---|---|---|---|
| 1 | Executar smoke test de produção (antes de ter usuários) apontando para infra de produção com dados sintéticos | DevOps | `/health` retorna OK para banco + Redis + RabbitMQ |
| 2 | Verificar conectividade de todas as integrações de produção (sem transações reais) | Backend Lead | ZapSign sandbox → produção: curl no endpoint de listagem de documentos retorna 200 |
| 3 | Confirmar que Sentry de produção recebe eventos (acionar erro de teste manualmente) | DevOps | Evento aparece no Sentry dashboard em < 1 min |
| 4 | Confirmar que PostHog de produção recebe eventos | DevOps | Evento `test_event` aparece no PostHog dashboard |
| 5 | Confirmar que alertas Sentry chegam ao Slack `#crm-incidents` | DevOps | Mensagem recebida no canal após trigger manual |
| 6 | Verificar DNS e certificado TLS de `crm.repasseseguro.com.br` e `api.crm.repasseseguro.com.br` | DevOps | HTTPS funcional sem warnings de certificado |

---

### 4.3 D-1 — Freeze + Backup

**Responsável geral:** Tech Lead + DevOps

| # | Ação | Dono | Critério de sucesso |
|---|---|---|---|
| 1 | **Code freeze total** — nenhum commit após 18h de D-1 | Tech Lead | Branch `main` com tag `v1.0.0-rc-final` |
| 2 | Criar branch `release/v1.0.0` a partir de `main` | Tech Lead | Branch criada e testada |
| 3 | Executar backup manual do Supabase staging (estado de pré-go-live) | DevOps | Backup confirmado com timestamp |
| 4 | Confirmar que migrations de produção estão prontas para `prisma migrate deploy` | Backend Lead | `prisma migrate status` em produção mostra migrations pendentes |
| 5 | Notificar equipe RS sobre o go-live do dia seguinte | Coordenador RS | Mensagem enviada para todos os usuários que terão acesso |
| 6 | Confirmar escalas de plantão para as primeiras 48h pós-launch | Tech Lead | Escala definida e comunicada no `#crm-incidents` |

---

### 4.4 D-0 — Dia do Go-Live

**Janela autorizada:** 10h–16h (America/Fortaleza). **Responsável geral:** Tech Lead.

#### Checklist de Go/No-Go (09h30 — antes de iniciar deploy)

| Critério | Threshold | Status |
|---|---|---|
| Suite de testes `release/v1.0.0` passando | 100% unitários + integração + E2E críticos | |
| Smoke test de produção | `GET /health` = OK | |
| Sentry ativo | Evento de teste recebido | |
| Zero itens Bloqueantes no Checklist de Qualidade | 0 itens ❌ | |
| Infra 100% provisionada | Todos os INF-XX ✅ | |
| Usuários iniciais criados | Todos os USR-XX ✅ | |
| Tech Lead presente | Disponível por 4h | |
| DevOps de plantão | Disponível por 4h | |

**Decisão de Go/No-Go:** Tech Lead declara Go ou No-Go às 09h45 com base nos critérios acima. A decisão é comunicada no `#crm-go-live`.

---

#### Sequência de Deploy (D-0 após Go)

```
10h00 — Início do deploy
         │
         ├── 1. pnpm --filter api prisma migrate deploy (produção)
         │        ↓ Verificar: prisma migrate status mostra 0 migrations pendentes
         │
         ├── 2. Deploy CRM API → Railway production
         │        ↓ Verificar: railway status + curl /health = OK
         │
         ├── 3. Deploy web-crm → Vercel production
         │        ↓ Verificar: https://crm.repasseseguro.com.br carrega tela de login
         │
         ├── 4. Smoke test pós-deploy (T+15min)
         │        ↓ Ver Seção 5 — Smoke Test
         │
         ├── 5. Go/No-Go pós-deploy (T+15min)
         │        ↓ Tech Lead avalia métricas
         │
         ├── 6. Ativação dos usuários (T+30min se Go)
         │        ↓ Comunicar equipe RS que o CRM está ativo
         │
10h45 — CRM em produção (estimado)
```

---

## 5. Smoke Test Pós-Deploy (T+15min)

Executar imediatamente após o deploy, antes de ativar usuários:

```bash
BASE="https://api.crm.repasseseguro.com.br"
TOKEN="[token de usuário de teste de produção]"

# 1. Health check
curl -f $BASE/health
# Esperado: { "status": "ok", "database": "ok", "redis": "ok", "rabbitmq": "ok" }

# 2. Autenticação
curl -f -X POST $BASE/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "email": "admin@crm.dev", "password": "[senha]" }'
# Esperado: HTTP 200 com access_token

# 3. Listagem de Casos (banco acessível)
curl -f $BASE/v1/cases \
  -H "Authorization: Bearer $TOKEN"
# Esperado: HTTP 200 com array (vazio no primeiro acesso)

# 4. Listagem de Contatos
curl -f $BASE/v1/contacts \
  -H "Authorization: Bearer $TOKEN"
# Esperado: HTTP 200

# 5. Frontend carregando
curl -f -I https://crm.repasseseguro.com.br
# Esperado: HTTP 200 com content-type text/html
```

**Critério de aprovação:** todos os 5 checks retornam HTTP 200. Qualquer falha = acionar protocolo de rollback imediato.

---

## 6. Critérios de Go/No-Go Pós-Deploy

Avaliados pelo Tech Lead nos primeiros 15 min após deploy:

| Métrica | Threshold Go | Threshold No-Go (Rollback) |
|---|---|---|
| Smoke test | 5/5 endpoints OK | Qualquer endpoint falhando |
| Taxa de erros 5xx | < 0.5% | > 2% por > 5 min |
| Latência p95 | < 300ms | > 1s por > 5 min |
| Health check | DB + Redis + RabbitMQ = UP | Qualquer dependência DOWN |
| Sentry: novos erros P0 | 0 | ≥ 1 erro P0 novo |

**Decisão:** Tech Lead declara **GO** (ativar usuários) ou **NO-GO** (rollback imediato). Não há meio-termo — a decisão é comunicada no `#crm-incidents`.

---

## 7. Rollback

### 7.1 Quando Acionar

- Qualquer threshold No-Go da Seção 6 atingido nos primeiros 15 min
- Tech Lead identifica bug crítico que impede operação básica do CRM
- Solicitação do Coordenador RS por impacto severo na equipe

### 7.2 Procedimento de Rollback (≤ 10 min)

```bash
# PASSO 1 (1 min) — Comunicar imediatamente
# Slack #crm-incidents: "@channel ROLLBACK INICIADO — CRM voltando para versão anterior."

# PASSO 2 (2 min) — Reverter API (Railway)
# Railway Dashboard → CRM API → Deployments → [deployment anterior] → Redeploy
# OU via CLI:
railway deployments --environment production
# Copiar o deployment ID anterior
# Railway Dashboard → clicar em "Redeploy" no deployment anterior

# PASSO 3 (1 min) — Reverter Frontend (Vercel)
# Vercel Dashboard → web-crm → Deployments → [deployment anterior] → Promote to Production
# OU via CLI:
vercel rollback --scope repasse-seguro

# PASSO 4 (2 min) — Verificar saúde após rollback
curl https://api.crm.repasseseguro.com.br/health
# Esperado: { "status": "ok" }

# PASSO 5 (1 min) — Confirmar no canal
# Slack #crm-incidents: "Rollback concluído. CRM restaurado para versão anterior. Investigando causa."

# PASSO 6 — Análise de causa raiz (após rollback estabilizado)
# Abrir post-mortem no Notion/Confluence
# Identificar causa, criar hotfix, re-testar, agendar novo deploy
```

**Tempo esperado para rollback completo: 7–10 min.**

### 7.3 Rollback de Migration

Se a migration causou o problema:

```
1. NÃO executar rollback automático — Prisma não suporta rollback de migrations
2. Identificar o DDL que causou o problema
3. Criar nova migration de reversão manual
4. Aplicar via pipeline de staging → produção
5. Se dados foram corrompidos: acionar restore de backup do Supabase (ver seção 5.1 do Doc 24)
```

---

## 8. Monitoramento Pós-Launch

### 8.1 Primeiros 15 min (T+0 a T+15)

- Tech Lead e DevOps em war room no Slack `#crm-go-live`
- Monitorar Sentry, Railway Metrics e `/health` a cada 2 min
- Critério: qualquer threshold No-Go = rollback imediato

### 8.2 Primeiras 2 horas (T+15 a T+2h)

| # | O que monitorar | Ferramenta | Threshold de alerta |
|---|---|---|---|
| 1 | Taxa de erros 5xx | Railway Metrics + Sentry | > 1% |
| 2 | Latência p95 | Railway Metrics | > 500ms |
| 3 | Health check | `curl /health` a cada 5 min | Qualquer DOWN |
| 4 | Filas RabbitMQ | CloudAMQP | DLQ > 0 |
| 5 | Login de usuários reais | PostHog `user_logged_in` | Nenhum evento = problema |
| 6 | Novos erros Sentry | Sentry Dashboard | Qualquer novo erro P0/P1 |

### 8.3 Primeiras 24 horas

- Check-in a cada 2 horas pelo on-call (registrar no `#crm-go-live`)
- Monitorar se relatório semanal será gerado corretamente (se go-live for segunda-feira)
- Verificar se alertas de SLA estão disparando corretamente

### 8.4 Primeiras 48 horas — Checklist de Estabilização

| # | Item | Status |
|---|---|---|
| ST-01 | Nenhum erro P0 ou P1 no Sentry nas últimas 24h | |
| ST-02 | Taxa de erros 5xx < 0.1% em média nas últimas 24h | |
| ST-03 | Latência p95 < 300ms em média nas últimas 24h | |
| ST-04 | DLQ de todas as filas = 0 | |
| ST-05 | Todos os usuários conseguiram autenticar sem problemas reportados | |
| ST-06 | Pelo menos 1 Caso criado com sucesso por usuário real | |
| ST-07 | Alertas de SLA funcionando (ao menos 1 alerta gerado se houver casos com deadline próximo) | |
| ST-08 | Sem reclamações críticas da equipe RS registradas no Slack | |

**Critério de encerramento do war room:** todos os ST-XX ✅ e nenhum novo incidente P0/P1 nas últimas 12h.

---

## 9. Comunicação

### 9.1 Template de E-mail para a Equipe no Dia do Lançamento

```
Assunto: CRM Repasse Seguro — Disponível agora em produção 🚀

Olá, equipe,

O CRM do Repasse Seguro acaba de entrar em produção.

--- ACESSO ---
URL: https://crm.repasseseguro.com.br
Suas credenciais foram enviadas individualmente de forma segura.
No primeiro acesso, você será solicitado a alterar sua senha.

--- O QUE ESTÁ DISPONÍVEL ---
• Pipeline de Casos com visualização kanban e lista
• Gestão de Contatos (Cedentes, Cessionários, Parceiros, Incorporadoras)
• Registro de Atividades (ligações, reuniões, WhatsApp, notas)
• Gestão de Negociações e Comissões
• Dossiê de documentos por Caso
• Alertas automáticos de SLA
• Relatório semanal (toda segunda às 08h)

--- SUPORTE ---
Para dúvidas: Slack #crm-suporte
Para problemas técnicos urgentes: Slack #crm-incidents

Bom trabalho a todos!
[Assinatura do Tech Lead / Coordenador RS]
```

### 9.2 Comunicação de Rollback (se necessário)

```
Slack #crm-incidents:
"@channel — O deploy do CRM foi revertido após identificarmos [descrição resumida do problema].
O CRM está indisponível temporariamente enquanto investigamos e corrigimos.
Estimativa de retorno: [horário estimado].
Próxima atualização: [horário do próximo update]."
```

---

## 10. Post-Mortem (se rollback ou incidente P0)

Todo incidente P0 ou rollback deve gerar um post-mortem em até 48h:

| Campo | Conteúdo |
|---|---|
| **Data/hora do incidente** | |
| **Duração** | |
| **Impacto** | Número de usuários afetados, features impactadas |
| **Linha do tempo** | Eventos em ordem cronológica |
| **Causa raiz** | |
| **Ações imediatas** | O que foi feito para resolver |
| **Itens de ação** | Lista de tarefas para prevenir recorrência (com dono e prazo) |
| **O que funcionou bem** | |
| **O que pode melhorar** | |

---

## 11. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — pré-requisitos de infra (12 itens), usuários iniciais, sequência D-7/D-3/D-1/D-0, smoke test pós-deploy (5 endpoints), critérios go/no-go objetivos, rollback em ≤10 min, monitoramento pós-launch 15min/2h/24h/48h, checklist de estabilização, template de comunicação para equipe RS. |
