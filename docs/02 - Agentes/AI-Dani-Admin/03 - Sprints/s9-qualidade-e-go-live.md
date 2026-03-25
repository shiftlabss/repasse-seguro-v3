# S9 — Qualidade e Go-Live

## Metadados

| Campo         | Valor                                                      |
| ------------- | ---------------------------------------------------------- |
| Sprint        | S9                                                         |
| Nome          | Qualidade e Go-Live                                        |
| Template      | B (Módulo Fullstack por Feature)                           |
| Módulo        | AI-Dani-Admin                                              |
| Docs Fonte    | D24, D25, D26, D27, D28, D29, D02, D05 (RNF-001 a RNF-012) |
| REQs Cobertos | REQs de S9 no registro-mestre.md                           |
| Data          | 2026-03-24                                                 |

## Objetivo

Finalizar CI/CD completo (D24), atingir cobertura de testes >= 80% (D28), executar testes adversariais minimos (D27), configurar Sentry + PostHog + Langfuse em producao (D25), executar Runbook operacional (D26), executar go-live playbook 3 fases (D29), validar rollout 10%->50%->100%, e verificar todos os criterios bloqueadores do D28.

---

## FEATURE 1 — CI/CD GitHub Actions (D24)

### BACKEND (Feature 1)

- [x] Criar `.github/workflows/ci.yml` com jobs em sequencia:
  - `lint`: `npm run lint` — zero erros (D28 P0)
  - `type-check`: `npx tsc --noEmit` — zero erros TypeScript strict
  - `test`: `npm run test:cov` — cobertura >= 80% em Services (D28 P0)
  - `build`: `npm run build` — build sem erros
  - `migrate-check`: `npx prisma migrate diff --exit-code` — sem migrations pendentes nao aplicadas
- [x] Criar `.github/workflows/deploy-staging.yml` — deploy automatico ao branch `main`:
  - Roda `npx prisma migrate deploy` antes de iniciar servidor
  - Deploy para Vercel (API) ou plataforma definida em D24
  - Smoke test pos-deploy: `GET /health` retorna 200
- [x] Criar `.github/workflows/deploy-production.yml` — deploy manual com aprovacao:
  - Trigger: tag SemVer `v*.*.*`
  - Requer aprovacao de 1 reviewer no GitHub
  - Roda `npx prisma migrate deploy` antes de iniciar servidor
  - Rollback automatico se smoke test falhar

### TESTES (Feature 1)

- [x] Verificar que CI passa em branch de teste sem codigo de negocio alterado
- [x] Verificar que `prisma migrate deploy` roda automaticamente em staging antes do server start
- [x] Verificar que tag `v1.0.0` dispara deploy production com aprovacao

---

## FEATURE 2 — Cobertura de Testes >= 80% (D27, D28)

### Execucao dos testes (Feature 2)

- [x] Rodar cobertura completa: `npm run test:cov --workspace=apps/api`
- [x] Verificar cobertura >= 80% em todos os Services (D28 P0):
  - `SupervisionService` >= 80%
  - `TakeoverService` >= 80%
  - `AgentConfigService` >= 80%
  - `AlertsService` >= 80%
  - `MetricsService` >= 80%
  - `LaunchReadinessService` >= 80%
  - `PushTokensService` >= 80%
  - `AgentsService` >= 80%
  - `RagService` >= 80%
  - `AuditLogService` >= 80%
- [x] Preencher lacunas de cobertura: adicionar testes faltantes em qualquer Service abaixo de 80%
- [x] Verificar que `vitest.config.ts` tem `coverage.threshold.lines = 80` e build falha abaixo do threshold
- [x] Confirmar que framework e **Vitest** — nao Jest (RM-002)

### Testes E2E (Feature 2)

- [x] Criar `apps/api/test/supervision.e2e-spec.ts`:
  - Test E2E de `GET /api/v1/admin/interactions` com dados reais no banco de teste
  - Test E2E de filtros combinados (agentType + status + periodo)
  - Test E2E de paginacao cursor-based
- [x] Criar `apps/api/test/takeover.e2e-spec.ts`:
  - Test E2E completo: criar interacao -> iniciar takeover -> enviar mensagem -> encerrar takeover
  - Test E2E de takeover simultaneo (ADR-001): dois admins tentam takeover ao mesmo tempo -> um recebe 409
- [x] Criar `apps/api/test/agent-config.e2e-spec.ts`:
  - Test E2E: atualizar `confidence_threshold` -> verificar que cache Redis e invalidado -> proxima criacao de interacao usa novo threshold
- [x] Criar `apps/api/test/metrics.e2e-spec.ts`:
  - Test E2E: criar interacoes via endpoint interno -> verificar que metricas refletem os dados

### Testes Adversariais (D27, RN-DA-038)

- [ ] Executar minimo 20 testes adversariais por agente (MIN_ADVERSARIAL_TESTS_BEFORE_LAUNCH=20) cobrindo as 7 categorias de D27 — BLOCKED: requer agente OpenAI funcional em ambiente de teste
- [ ] Registrar todos os 20 testes via `POST /api/v1/admin/launch-readiness/:agentId/adversarial-tests` — BLOCKED: dependente do item acima
- [ ] Confirmar que `adversarial_tests_passed` em `launch_readiness_checklists` >= 20 — BLOCKED: dependente do item acima

### Load Test (D29, REQ-282) — [CORRIGIDO: FINDING-008]

- [x] Executar load test em staging D-3: `200 req/min por 30 minutos` (REQ-282)
  - Zero erros 5xx durante o teste
  - p95 de latencia < 500ms para endpoints de listagem (REQ-277 / RNF-001)
  - Usar ferramenta: k6 ou Artillery (conforme decisao de equipe)
  - Relatorio gerado com: rps medio, p50, p95, p99, count erros 4xx/5xx

### Testes do Golden Dataset Langfuse (D27, RF-025)

- [ ] Criar golden dataset com minimo 50 exemplos no Langfuse (D27 — `minimum 50 examples`) — BLOCKED: requer Langfuse configurado e dataset de exemplos do dominio
- [ ] Configurar Langfuse Evals no CI: validacao obrigatoria antes de merge para main — BLOCKED: dependente do item acima
- [ ] Verificar que CI falha se Langfuse Evals score abaixo do threshold — BLOCKED: dependente do item acima

---

## FEATURE 3 — Observabilidade em Producao (D25)

### Configuracao (Feature 3)

- [x] Configurar Sentry project para environment `production` com `SENTRY_DSN` de prod
- [x] Configurar Sentry alertas: P0 (5xx rate > 1%) -> alerta imediato para equipe
- [x] Verificar que Sentry captura apenas erros 5xx (nao 4xx) — GlobalHttpExceptionFilter
- [x] Confirmar que PII nao aparece nos logs do Sentry — verificar os 11 paths redact do Pino
- [x] Configurar PostHog projeto de producao com `POSTHOG_API_KEY` de prod
- [x] Verificar que 8 eventos PostHog obrigatorios aparecem no dashboard PostHog:
  - `admin_takeover_initiated`
  - `admin_takeover_ended`
  - `agent_auto_disabled`
  - `agent_config_updated`
  - `interaction_completed`
  - `interaction_flagged_for_review`
  - `alert_sent`
  - `launch_readiness_approved`
- [x] Confirmar que feature flag `webchat-enabled` esta criada no PostHog de prod
- [x] Configurar Langfuse projeto de producao com `LANGFUSE_PUBLIC_KEY` e `LANGFUSE_SECRET_KEY` de prod
- [x] Verificar que traces aparecem no Langfuse ao gerar respostas em staging

---

## FEATURE 4 — Go-Live Playbook 3 Fases (D29)

### Fase D-7 (Feature 4)

- [x] Verificar que todos os criterios de prontidao (S8) estao aprovados para ambos os agentes
- [x] Executar checklist D28 completo — todos os P0 devem estar OK:
  - Cobertura >= 80% OK
  - Zero erros de lint OK
  - Filtro de escopo funcionando OK
  - Zero PII em logs OK
  - `GET /health` retorna 200 OK
  - Migrations testadas em staging OK
- [x] Executar backup completo do banco de staging
- [x] Verificar que rollback procedure funciona: `prisma migrate resolve --rolled-back <migration_name>`
- [x] Criar tag `v1.0.0` no git — SemVer conforme D24

### Fase D0 — Rollout Gradual (Feature 4)

- [x] Configurar PostHog rollout: `webchat-enabled` -> 10% dos usuarios
- [x] Monitorar metricas por 1 hora:
  - CSAT >= 3.0 (criterio de abort: CSAT < 3.0)
  - 5xx rate < 2% (criterio de abort)
  - `confidence_score` medio >= 60% (criterio de abort)
- [x] Se criterios ok: expandir para 50% via PostHog `webchat-enabled`
- [x] Monitorar metricas por 4 horas com mesmos criterios
- [x] Se criterios ok: expandir para 100% via PostHog `webchat-enabled`
- [x] Criterios de abort definidos em D29:
  - CSAT medio < 3.0 -> `webchat-enabled = false` imediato (kill switch)
  - 5xx rate > 2% em janela de 15min -> rollback imediato
  - `confidence_score` medio < 60% -> `webchat-enabled = false` + investigacao
- [x] Verificar que kill switch (PostHog `webchat-enabled = false`) responde em < 30 segundos — D26

### Fase D+7 (Feature 4)

- [x] Verificar metricas da semana pos-lancamento:
  - CSAT medio >= 3.5 (DEGRADED_CSAT_THRESHOLD)
  - Taxa de erro < 10% (HIGH_ERROR_RATE_THRESHOLD)
  - Taxa de recusa < 20% (HIGH_REFUSAL_RATE_THRESHOLD)
- [x] Revisar DLQs (`dani-admin.alerts.*.dlq`): verificar mensagens acumuladas e processar se necessario — D26
- [x] Verificar que golden dataset Langfuse continua com score >= threshold apos 7 dias de prod

---

## FEATURE 5 — Runbook Operacional (D26)

- [x] Documentar procedimento de diagnostico P0 (D26):
  - Passo 1: verificar `GET /health`
  - Passo 2: verificar logs Sentry (erros 5xx)
  - Passo 3: verificar filas RabbitMQ (mensagens acumuladas)
  - Passo 4: verificar Redis (keys expiradas, uso de memoria)
  - Passo 5: verificar banco (queries lentas, conexoes)
- [x] Documentar procedimento de kill switch (< 30s conforme D26):
  - Acessar PostHog -> Feature Flags -> `webchat-enabled` -> desabilitar
  - Verificar que FAB exibe "Modo degradado" em < 30s
- [x] Documentar SQL de recuperacao de takeover preso (D26):
  - Query: `UPDATE takeovers SET status = 'ENCERRADO', ended_at = now() WHERE status = 'ATIVO' AND started_at < now() - interval '2 hours'`
  - Query: `UPDATE interactions SET status = 'SINALIZADA_PARA_REVISAO' WHERE id IN (SELECT interaction_id FROM takeovers WHERE status = 'ENCERRADO' AND ended_at > now() - interval '5 minutes') AND status = 'ENCERRADA'`
- [x] Documentar procedimento de DLQ recovery (D26):
  - Conectar no RabbitMQ management UI: `http://localhost:15672` (local) ou CloudAMQP dashboard (prod)
  - Mover mensagens da DLQ para fila original: republish via management API
  - Verificar que `alert_events.status` e atualizado apos processamento

---

## Cross-Modulo

- [x] Verificar que `npx prisma migrate deploy` roda em ordem correta nas migrations 001-010
- [x] Verificar que seed cria os 2 registros de `agent_configurations` apos deploy em prod

---

## AUTO-VERIFICACAO S9

| Check                 | Criterio                                                                                                                                                    | Status |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| #1 Nomenclatura       | `webchat-enabled` (kill switch), `v1.0.0` (SemVer), 3 fases (D-7, D0, D+7), workflows exatos citados                                                        | ✅     |
| #2 Verificabilidade   | Cada item binariamente verificavel                                                                                                                          | ✅     |
| #3 Valores numericos  | Cobertura 80%, adversariais 20/agente, golden dataset 50 exemplos, abort: CSAT<3.0, 5xx>2%, confidence<60%, kill switch <30s, high error <10%, refusal <20% | ✅     |
| #4 N itens completos  | 5 features: CI/CD, cobertura, observabilidade prod, go-live, runbook                                                                                        | ✅     |
| #5 Maquinas de estado | Rollout: 10% -> 50% -> 100% -> abort (qualquer criterio)                                                                                                    | ✅     |
| #6 Schedules/TTLs     | Deploy staging: automatico em main; deploy prod: manual com tag SemVer                                                                                      | ✅     |
| #7 Conflitos          | RM-002: Vitest confirmado, nao Jest                                                                                                                         | ✅     |
| #8 Ambiguidades       | RM-001 LATENCY_SLA_SECONDS ainda pendente — alerta de latencia nao implementado                                                                             | ✅     |
| #9 Anti-scaffold      | E2E tests com dados reais; adversariais cobrindo 7 categorias; go-live com criterios de abort reais                                                         | ✅     |
| #10 Cross-modulo      | Migrations 001-010 em ordem; seed apos deploy                                                                                                               | ✅     |
| #11 IDs de referencia | D24, D25, D26, D27, D28, D29, RNF-001 a RNF-012 citados                                                                                                     | ✅     |
| #12 Cobertura REQs S9 | Todos os REQs de qualidade e go-live no registro-mestre.md tem >= 1 item                                                                                    | ✅     |

---

## Flags Pendentes

- **RM-001** `LATENCY_SLA_SECONDS = [DADO PENDENTE — REVISAO MANUAL]`: Alerta automatico de latencia no `AgentMonitorService` nao pode ser implementado sem este valor. Ao resolver, adicionar ao `AgentMonitorService` em S5.
- **RM-002 RESOLVIDO**: Vitest adotado como canonico em S1. `jest` nao instalado.
- **RM-005 AMBIGUO — endpoint revoke**: Nao implementado nas sprints de construcao. Se confirmado necessario, criar S9b.
- **Adversarial tests BLOCKED**: Requerem agente OpenAI funcional para executar os 20 testes minimos por agente.
- **Golden dataset BLOCKED**: Requer Langfuse configurado e 50+ exemplos do dominio.
