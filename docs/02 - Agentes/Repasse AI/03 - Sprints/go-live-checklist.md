# Go-Live Checklist — Repasse AI

## T-7 (7 dias antes do lancamento)

- [ ] Todos os 5 smoke tests de staging passam (`pnpm test:smoke`)
- [ ] Cobertura de testes: todos os modulos acima dos thresholds (S8 verificado)
- [ ] Langfuse evals: `answer_correctness >= 0.80` e `faithfulness >= 0.85` no golden dataset de 50 exemplos
- [ ] Zero vulnerabilidades HIGH/CRITICAL em `pnpm audit`
- [ ] Variaveis de producao configuradas no Railway Dashboard
- [ ] Backup do banco de dados testado e restauravel
- [ ] Plano de rollback documentado e testado (ver abaixo)

## T-3 (3 dias antes)

- [ ] Deploy em staging com tag release candidate (ex: `v1.0.0-rc1`)
- [ ] Smoke tests em staging: 5/5 passando
- [ ] Carga simulada: 30 req/min por 5min em staging — sem cold starts, latencia mediana < 3s

## T-1 (1 dia antes)

- [ ] Deploy em staging com tag de release candidate (ex: `v1.0.0-rc1`) sem erros
- [ ] Smoke tests em staging: 5/5 passando
- [ ] Latencia media em staging: <= 5s para analise, <= 10s para comparacao (verificar logs Railway)
- [ ] 2 approvals de merge para `main` confirmados

## Launch Day

- [ ] Deploy para `production` com tag final (ex: `v1.0.0`) -> smoke tests producao: 5/5 passando
- [ ] `GET /health` producao: `200 { status: "ok" }`
- [ ] Canal Slack `#repasse-ai-launch` criado e equipe notificada
- [ ] Sentry producao ativo: verificar via `POST /internal/test-sentry` com header `X-Internal-Test: {INTERNAL_TEST_SECRET}` — Sentry captura exception de teste em < 30s
- [ ] Langfuse producao ativo: request real -> trace registrado
- [ ] `agent_configurations.agent_status = 'online'` em producao (via `POST /supervision/agent/status`)

---

## Plano de Rollback (< 3 minutos)

**Trigger:** smoke test falha em producao OU error rate > 5% em 5 minutos (Sentry)

| Passo | Tempo     | Acao                                                                 |
| ----- | --------- | -------------------------------------------------------------------- |
| 1     | 0-30s     | `railway rollback --service repasse-ai` para versao anterior         |
| 2     | 30s-60s   | Verificar `GET /health` na versao anterior                           |
| 3     | 60s-90s   | Executar 5 smoke tests na versao anterior                            |
| 4     | 90s-120s  | Notificar `#repasse-ai-launch` no Slack com status de rollback       |
| 5     | 120s-180s | Confirmar trafego roteado para versao anterior + fechar incidente P0 |

---

## Escalation Matrix

| Severidade       | Tempo de Resposta | Exemplo                                                                                      | Responsavel         | Acao                                    |
| ---------------- | ----------------- | -------------------------------------------------------------------------------------------- | ------------------- | --------------------------------------- |
| **P0 — Critico** | < 15 min          | Servico fora do ar; vazamento de dados; todos os usuarios impactados                         | CTO + Lead Engineer | Rollback imediato + war room Slack      |
| **P1 — Alto**    | < 1h              | Degradacao de performance (latencia > 30s); takeover nao funciona; rate limit falso positivo | Lead Engineer       | Hotfix em staging -> deploy emergencial |
| **P2 — Medio**   | < 4h              | WhatsApp fora do ar; Langfuse offline; cache indisponivel                                    | Engineer on-call    | Investigar + plano de correcao          |
| **P3 — Baixo**   | Proximo dia util  | Cobertura abaixo do threshold; alertas Admin nao chegando; golden dataset score abaixo       | Equipe de QA        | Ticket no backlog                       |

### Configuracao de alertas Sentry

- **P0/P1:** Notificacao imediata via PagerDuty/Slack
- **P2:** Slack `#repasse-ai-alerts`
- **P3:** GitHub Issue automatico
