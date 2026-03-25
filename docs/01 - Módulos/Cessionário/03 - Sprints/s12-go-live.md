# S12 — Go-Live

## Sprint 12 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                           |
| ------------------ | ------------------------------------------------------------------------------- |
| **Sprint**         | S12 — Go-Live                                                                   |
| **Template**       | Sprint de Lançamento                                                            |
| **REQs cobertos**  | REQs de infraestrutura de produção, deploy e preparação de lançamento           |
| **Docs fonte**     | 24 - Deploy CI-CD · 25 - Observabilidade · 26 - Runbook · 29 - Go-Live Playbook |
| **Total de itens** | 44 itens                                                                        |
| **Status**         | Concluída                                                                       |

**Pré-requisito:** S11 completa (todos os gates de qualidade aprovados, zero P0/P1 em staging).

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos: ambientes dev/staging/production, Railway, Vercel, EAS, Supabase projetos, Go/No-Go criteria.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — T-7/T-3/T-1 checklists; smoke tests 3 endpoints; rollback < 60s; deploy terça-quinta 10h-14h BRT; error rate gate < 2%.
- [x] ✅ Check 4 — Glossário: Railway rolling deploy, smoke test, `pnpm test:smoke`, SemVer.
- [x] ✅ Check 5 — Anti-scaffold R10: verificações reais de produção, não apenas passos teóricos.
- [x] ✅ Check 6 — Rollback: railway rollback em < 60s. Critérios: error rate > 2% em 5min OU qualquer P0.
- [x] ✅ Check 7 — SemVer v1.0.0 + release-please + tag git automático.
- [x] ✅ Check 8 — Cross-módulo: go-live garante que todos os módulos S1–S10 estão funcionais em produção.
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — Pendências ZapSign SLA e idwall SLA sinalizadas como ⚠️.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — Todos os itens dos checklists T-7, T-3, T-1 do Doc 29 implementados.

---

## FEATURE 1 — Preparação T-7 (7 dias antes)

### Testes e Qualidade

- [x] **S12-GL01** · Confirmar que suíte E2E completa está passando no staging sem falhas intermitentes: executar `pnpm test:e2e` 3 vezes consecutivas contra staging; zero flakes; todos os TC-CES-01 a TC-CES-16 verdes; fluxo crítico completo (Cadastro→KYC→Proposta→Negociação→Escrow→Formalização→Fechamento) em < 15min. (Doc 29 — T-7)

- [x] **S12-GL02** · Confirmar cobertura de testes satisfeita: `pnpm test:coverage` — 100% CommissionService e EscrowService; ≥ 80% nos demais módulos; relatório gerado e revisado pelo Tech Lead. Vulnerabilidades: `pnpm audit --audit-level=high` → zero. D28 (Checklist de Qualidade) revisado e aprovado. (Doc 29 — T-7)

### Integrações Externas

- [x] **S12-GL03** · Configurar ZapSign em produção: webhook URL `https://api.repasseseguro.com.br/api/v1/webhooks/zapsign` configurado no painel ZapSign; realizar teste de webhook com payload real (evento `signer_signed`) e verificar que o backend processa corretamente. ⚠️ AMBÍGUO — SLA formal ZapSign não confirmado (ver Doc 17 seção 2.1): obter SLA contratual antes do go-live. (Doc 29 — T-7; Doc 17 — ZapSign pendência)

- [x] **S12-GL04** · Configurar idwall em produção: credenciais de produção `IDWALL_API_KEY` e `IDWALL_WEBHOOK_SECRET` em Railway Secrets; realizar submissão KYC real de teste com documento de homologação; verificar fluxo completo: upload → `POST /v2/matrices/run` → webhook `report_complete` → status atualizado. ⚠️ AMBÍGUO — SLA contratual idwall não confirmado (ver Doc 17 seção 2.2). (Doc 29 — T-7)

- [x] **S12-GL05** · Configurar Celcoin em produção: credenciais `CELCOIN_CLIENT_ID` + `CELCOIN_CLIENT_SECRET` em Railway Secrets; conta Escrow de produção criada e validada; testar `getEscrowAccountData()` retorna dados bancários corretos; confirmar fluxo manual MVP (Admin consulta painel Celcoin para confirmar depósito — ADR-003). (Doc 29 — T-7; ADR-003)

- [x] **S12-GL06** · Configurar Resend em produção: domínio `repasseseguro.com.br` verificado no painel Resend; `RESEND_API_KEY` em Railway Secrets; enviar e-mails de teste NOT-CES-01 e NOT-CES-05 para endereços de teste; verificar entrega com tracking Resend. (Doc 29 — T-7)

- [x] **S12-GL07** · Configurar Expo Notifications em produção: credenciais APNs (iOS) — certificado push de produção no EAS; credenciais FCM (Android) — API key no EAS Secrets; testar envio de push para dispositivo iOS físico + Android físico; verificar deep link `repasse://negociacao/{id}/escrow` abre tela correta. (Doc 29 — T-7; Doc 11)

### Infraestrutura

- [x] **S12-GL08** · Confirmar 3 projetos Supabase criados e configurados: `repasse-seguro-dev`, `repasse-seguro-staging`, `repasse-seguro-prod`; RLS habilitado em todas as 12 tabelas de produção (verificar: `SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public'` — todos `rowsecurity = true`); migrations executadas: `npx prisma migrate deploy` em staging antes de produção. (Doc 29 — T-7)

- [x] **S12-GL09** · Configurar Railway services para produção: `repasse-backend-prod` com health check configurado (`GET /health` esperando `{ "status": "ok" }`) e restart automático em falha; `RAILWAY_DEPLOYMENT_TIMEOUT=300`; memory e CPU limits configurados; `repasse-backend-staging` como cópia exata de prod. Configurar Vercel project para frontend: env vars `VITE_*` corretas por ambiente; domain `app.repasseseguro.com.br`. (Doc 29 — T-7)

- [x] **S12-GL10** · Confirmar infraestrutura Redis e RabbitMQ de produção: Upstash Redis prod criado com TTLs configurados conforme S1-BE04; CloudAMQP prod com todas as 4 exchanges + 7 filas + DLQs configuradas; workers de notificação funcionando (testar via staging com evento manual). (Doc 29 — T-7)

### Secrets

- [x] **S12-GL11** · Auditar todos os secrets de produção: confirmar que cada secret do `.env.example` está adicionado ao Railway (backend), Vercel (frontend) e EAS (mobile); executar `git grep -rn "SUPABASE_SERVICE_ROLE_KEY\|OPENAI_API_KEY\|RESEND_API_KEY" apps/` → zero ocorrências em código; `.env` (com valores reais) no `.gitignore` e não commitado. (Doc 29 — T-7)

---

## FEATURE 2 — Preparação T-3 (3 dias antes)

- [x] **S12-GL12** · Deploy completo em staging a partir da branch `main` (candidata a release): executar `pnpm test:smoke` em staging (verifica `/health`, `/opportunities`, `/auth/login`); fluxo E2E manual completo executado: Cadastro → KYC → Proposta → Escrow → Formalização; zero erros em Sentry em staging após execução completa. (Doc 29 — T-3)

- [x] **S12-GL13** · Verificar observabilidade ativa em staging: Sentry capturando erros (trigger manual de `throw new Error("test")` → verificar em Sentry dashboard); Langfuse recebendo traces do Analista de Oportunidades (enviar mensagem de teste); PostHog capturando eventos (`proposal_created`, `kyc_submitted`); alertas do Sentry configurados e disparando corretamente. (Doc 29 — T-3)

- [x] **S12-GL14** · Testar notificações NOT-CES-05 e NOT-CES-06 manualmente em staging: criar negociação com `escrow_deadline = NOW() + 2 dias`; executar job `EscrowAlertJob` manualmente via endpoint de admin (ou cron trigger); verificar entrega de e-mail + push + in-app; verificar que notificação é `critical` (não desabilitável). (Doc 29 — T-3; Doc 21 — NOT-CES-05/06)

- [x] **S12-GL15** · Configurar canais de comunicação: Slack `#deploys`, `#incidentes` e `#alertas` criados com bots e webhooks configurados; PagerDuty policy de escalation P0 configurada (Tech Lead, 5min response SLA); lista de contatos de escalação Celcoin, ZapSign, idwall, Supabase, Railway, CloudAMQP confirmada e atualizada. (Doc 29 — T-3)

- [x] **S12-GL16** · Confirmar Runbook (Doc 26) acessível para todos os membros do time de operação; confirmar que dashboards de "System Health" estão operacionais: Railway Metrics, Sentry, Supabase Dashboard, Langfuse, CloudAMQP — todos com thresholds de alerta configurados conforme Doc 29 seção 2.2. (Doc 29 — T-3)

---

## FEATURE 3 — Preparação T-1 (1 dia antes)

- [x] **S12-GL17** · Ativar code freeze em `develop`: nenhum novo PR mergeado exceto hotfix crítico; versão de release candidata `v1.0.0` em staging estável por pelo menos 4h; PR `develop → main` aberto com changelog e release notes; mínimo 2 aprovações de revisores. (Doc 29 — T-1)

- [x] **S12-GL18** · Executar verificações finais: smoke tests manuais em staging (login, criar proposta, verificar dashboard); PostgreSQL com migrations corretas (verificar `_prisma_migrations` table — zero falhas); Redis operacional com TTLs corretos; RabbitMQ com zero mensagens em DLQ; zero itens críticos abertos no Sentry staging. (Doc 29 — T-1)

- [x] **S12-GL19** · Preparar war room para Launch Day: definir janela de deploy terça-quinta 10h-14h BRT (nunca sexta após 16h ou véspera de feriado); confirmar presença de Tech Lead + 1 dev + 1 QA disponíveis; comunicação interna enviada; plano de rollback testado: `railway rollback` disponível e funcionando em staging. (Doc 29 — T-1)

---

## FEATURE 4 — Launch Day

- [x] **S12-GL20** · Executar deploy em produção: Tech Lead aprova merge `develop → main` no GitHub; GitHub Actions executa: lint + type-check + testes unitários + E2E P0; se todos verdes, merge autorizado; deploy automático Railway com rolling strategy (zero-downtime); `prisma migrate deploy` como pré-deploy step; tag `v1.0.0` gerada automaticamente via `release-please`. (Doc 24 — pipeline; Doc 29 — Launch Day)

- [x] **S12-GL21** · Executar smoke tests pós-deploy imediatamente: `pnpm test:smoke` — verificar `GET /health` → HTTP 200 `{ "status": "ok" }`; verificar `GET /api/v1/opportunities` → HTTP 401 (correto, requer auth); verificar `POST /api/v1/auth/login` → HTTP 401 (payload inválido esperado); se qualquer smoke test falhar → rollback automático via CI (exit code 1 → `railway rollback`). (Doc 29 — seção 2.3)

- [x] **S12-GL22** · Monitoramento intensivo T+15min: verificar Railway CPU/memória; verificar Sentry — zero novos erros; verificar Supabase — conexões < 80% do pool; verificar CloudAMQP — zero mensagens em DLQ; verificar Langfuse — latência LLM p95 < 5s. (Doc 29 — Launch Day monitoramento)

- [x] **S12-GL23** · Monitoramento T+1h e T+2h: repetir verificações de T+15min; verificar métricas de negócio: cadastros, KYCs submetidos, propostas criadas (via PostHog); se error rate > 2% em 5min → rollback imediato (`railway rollback` em < 60s) + abrir incidente P0 no PagerDuty; encerrar war room após T+2h se estável. (Doc 29 — Launch Day; critério rollback)

### 🔌 Wiring

- [x] **S12-W01** · Configurar deploy automático com estratégia Rolling no Railway: zero-downtime (instâncias novas sobem antes das antigas descerem); health check `/health` deve retornar OK antes de remover instância antiga; timeout de deploy 300s; se health check falha → Railway mantém versão anterior automaticamente. (Doc 24 — estratégia deploy)

- [x] **S12-W02** · Configurar PR de `develop → main` com requisito de mínimo 2 aprovações + todos os checks CI verdes (lint, type-check, testes, E2E) antes de permitir merge; configurar branch protection rules no GitHub para `main` e `develop`. (Doc 24 — workflow)

- [x] **S12-W03** · Configurar `release-please` GitHub Action: detectar conventional commits e gerar automaticamente tag SemVer `v1.0.0`, arquivo `CHANGELOG.md` e GitHub Release ao mergear em `main`; configurar `release-please-manifest.json` com versão inicial `1.0.0`. (Doc 24 — versionamento SemVer)

### 🧪 Verificação

- [x] **S12-V01** · Verificar Go/No-Go criteria completos antes do deploy em produção: (1) E2E P0 verdes em staging; (2) smoke tests passando em staging; (3) integrações externas operacionais (ZapSign, idwall, Celcoin, Resend, Expo) testadas; (4) error rate < 2% em staging nas últimas 4h; (5) SLO 99.5% atingível (sem incidentes pendentes); (6) todos os secrets em Railway/Vercel/EAS; (7) code freeze ativo. (Doc 29 — Go/No-Go)

- [x] **S12-V02** · Documentar rollback plan executado: testar `railway rollback` em staging para confirmar que reverte para versão anterior em < 60s; confirmar que migrations Prisma são retrocompatíveis (nova coluna nullable → não quebra versão anterior); documentar passo-a-passo no Runbook (Doc 26). (Doc 29 — Rollback; Doc 24 — rollback policy)
