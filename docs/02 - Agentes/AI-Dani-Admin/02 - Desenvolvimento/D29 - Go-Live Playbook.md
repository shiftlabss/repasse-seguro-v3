# Go-Live Playbook — AI-Dani-Admin

## Plano de Lançamento, Rollout e Validação de Produção

| Campo | Valor |
|---|---|
| Destinatário | Tech Lead, Engenharia Backend, DevOps, Produto |
| Escopo | Sequência completa de go-live do módulo AI-Dani-Admin em produção — pré-lançamento, rollout progressivo, validação e contingência |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D01-D28 (todos os documentos da Fase 2) |

---

> **📌 TL;DR**
>
> - **Janela de go-live:** terça ou quarta, 10h-12h (horário de Fortaleza). Nunca sextas ou vésperas de feriado.
> - **3 fases:** Pré-Go-Live (D-7 a D-1) → Dia do Go-Live (D0) → Pós-Go-Live (D+1 a D+7).
> - **Rollout progressivo:** 10% → 50% → 100% via feature flags PostHog.
> - **Critérios de abortagem:** CSAT < 3.0, erros 5xx > 2%, confidence médio < 60% nas primeiras 2h.
> - **Kill switch:** `webchat-enabled: false` no PostHog em < 30s, sem deploy.
> - **Go/No-Go:** decisão coletiva — Tech Lead + Produto + DevOps antes de cada fase do rollout.

---

## 1. Cronograma de Go-Live

### Fase 1 — Pré-Go-Live (D-7 a D-1)

| Dia | Atividade | Responsável | Critério de Conclusão |
|---|---|---|---|
| D-7 | Deploy em staging da versão candidata | DevOps | Staging estável, health check ok |
| D-7 | Testes E2E completos em staging | Engenharia | CI verde, cobertura ≥ 80% |
| D-5 | Testes adversariais manuais (prompt injection) | Tech Lead | 100% recusas corretas |
| D-5 | Validação do checklist de launch readiness (agentes) | Produto | Todos os itens completos no painel |
| D-3 | Load test em staging (200 req/min por 30min) | DevOps | Zero erros 5xx, p95 < 500ms |
| D-3 | Revisão do runbook de rollback (D26) | Tech Lead | Runbook validado e atualizado |
| D-2 | Configuração das feature flags em produção | DevOps | `webchat-enabled: false` (desabilitado por padrão) |
| D-2 | Configuração dos alertas Slack P0/P1 | DevOps | Canal #alertas-prod configurado |
| D-1 | Dry-run do processo de rollback em staging | DevOps | Rollback executado em < 10min |
| D-1 | Briefing com toda a equipe | Tech Lead | Todos cientes do plano e seus papéis |
| D-1 | Criar tag `v1.0.0` na branch main | Tech Lead | Tag criada, pipeline CD configurado |

### Fase 2 — Dia do Go-Live (D0)

| Horário | Atividade | Responsável |
|---|---|---|
| 09:00 | Go/No-Go call — confirmação final | Tech Lead + Produto + DevOps |
| 09:30 | Deploy da `v1.0.0` em produção (pipeline CD) | DevOps |
| 09:45 | Verificação de health check pós-deploy | DevOps |
| 09:45 | Verificação de migrations aplicadas | DevOps |
| 10:00 | Habilitar `webchat-enabled: 10%` (rollout inicial) | Tech Lead |
| 10:00–11:00 | **Monitoramento intensivo — janela de 1h** | Equipe completa |
| 11:00 | Go/No-Go para 50% | Tech Lead + Produto |
| 11:15 | Habilitar `webchat-enabled: 50%` | Tech Lead |
| 11:15–12:00 | Monitoramento contínuo | Engenharia + DevOps |
| 12:00 | Go/No-Go para 100% | Tech Lead + Produto |
| 12:15 | Habilitar `webchat-enabled: 100%` | Tech Lead |
| 12:15–14:00 | Monitoramento estendido pós-100% | Engenharia |
| 14:00 | Comunicado para stakeholders: "Go-Live concluído" | Produto |

### Fase 3 — Pós-Go-Live (D+1 a D+7)

| Período | Atividade |
|---|---|
| D+1 | Revisão das métricas das primeiras 24h (CSAT, volume, erros) |
| D+2 | Revisão dos traces Langfuse (tokens, custo, latência de modelos) |
| D+3 | Análise dos prompts adversariais reais recebidos |
| D+7 | Retrospectiva de go-live com toda a equipe |
| D+7 | Atualização da documentação com lições aprendidas |

---

## 2. Critérios de Go/No-Go

### 2.1 Go/No-Go — Pré-Deploy (D0 09:00)

Todos os itens devem estar ✅ para prosseguir:

| Item | Critério | Fonte |
|---|---|---|
| CI verde na `v1.0.0` | Todos os checks passando | GitHub Actions |
| Cobertura ≥ 80% | Report do CI | Coverage report |
| Staging estável ≥ 48h | Sem incidentes desde D-2 | Sentry + health check |
| Launch readiness ✅ (todos os agentes) | Checklist completo | Painel Admin |
| Adversarial tests: 100% recusas | RN-DA-037, RN-DA-038 | Resultados D-5 |
| Runbook de rollback validado | Dry-run bem-sucedido | DevOps D-1 |
| Alertas P0/P1 configurados | Canal Slack #alertas-prod | DevOps |
| Feature flags em produção | `webchat-enabled: false` | PostHog |
| Equipe completa disponível | Tech Lead + 2 Eng + DevOps | Confirmação |

### 2.2 Go/No-Go — Escalada para 50% (D0 11:00)

Verificar métricas dos primeiros 60min no rollout de 10%:

| Métrica | Threshold de Go | Threshold de Abortagem |
|---|---|---|
| Taxa de erros 5xx | < 0.5% | > 2% |
| Latência API p95 | < 500ms | > 1500ms |
| Confidence score médio | > 70% | < 60% |
| CSAT (se disponível) | ≥ 4.0 | < 3.0 |
| Health check | `ok` em todas as dependencies | Qualquer `error` |
| Sentry — novas issues críticas | 0 | Qualquer P0 |

### 2.3 Go/No-Go — Escalada para 100% (D0 12:00)

Verificar métricas dos 45min no rollout de 50% (mesmos critérios da tabela acima).

---

## 3. Procedimento de Rollout Progressivo

### 3.1 Feature Flags no PostHog

```
Rollout via PostHog → Feature Flags → webchat-enabled

Fase 1 — 10%:
  Rollout: 10%
  Tipo: Percentage of users
  Filtro adicional: nenhum (aleatório)

Fase 2 — 50%:
  Rollout: 50%
  (aumentar apenas se Go/No-Go aprovado)

Fase 3 — 100%:
  Rollout: 100%
  (aumentar apenas se Go/No-Go aprovado)
```

### 3.2 Rollout Granular por Agente (se necessário)

Se problemas forem isolados em um agente específico:

```
Desabilitar apenas Dani-Cessionário:
→ PostHog → dani-cessionario-enabled → 0%

Desabilitar apenas Dani-Cedente:
→ PostHog → dani-cedente-enabled → 0%
```

---

## 4. Plano de Rollback

### 4.1 Critérios de Abortagem (rollback imediato)

Qualquer um dos itens abaixo dispara rollback automático sem aprovação:

- Taxa de erros 5xx > 2% por mais de 5 minutos
- Sentry com issue P0 (crash total do serviço)
- Health check `/health` retornando 503
- Vazamento de dados detectado (PII em logs, cross-user data)
- CSAT < 3.0 na primeira hora

### 4.2 Procedimento de Rollback

**Rollback imediato via kill switch (< 30s):**

```
1. PostHog → webchat-enabled → 0% (desabilita imediatamente)
2. Comunicar no Slack #alertas-criticos: "webchat desabilitado — investigando"
3. Iniciar diagnóstico (ver D26 Runbook)
```

**Rollback de código (deploy da versão anterior):**

```bash
# Re-deploy da versão anterior via GitHub
# GitHub → Actions → CD Production → Run workflow
# Selecionar tag anterior: v0.X.X

# Ou via SSH:
cd /app/repasse-api
docker pull ghcr.io/shiftlabs/repasse-api:v0.9.0  # versão anterior
docker-compose -f docker-compose.prod.yml up -d --no-deps api

# Verificar
curl https://api.repasseseguro.com.br/health
```

**Rollback com migration destrutiva:**

```bash
# Aplicar migration de rollback (deve estar preparada — D24, D26)
docker exec repasse-api-prod npx prisma migrate deploy
# (a migration de rollback foi incluída nas migrations antes do deploy)
```

### 4.3 Comunicação de Incidente

```
Canal: Slack #alertas-criticos
Template:

🔴 INCIDENTE — AI-Dani-Admin
Status: ROLLBACK ATIVADO
Horário: [TIMESTAMP]
Impacto: [descrever usuários afetados]
Causa provável: [descrever]
Ações tomadas: kill switch ativado / deploy revertido
Próximos passos: [descrever]
Responsável: [nome]
ETA para resolução: [estimativa]
```

---

## 5. Monitoramento de Go-Live

### 5.1 Dashboard de Monitoramento (D0)

Durante o go-live, manter abertos em paralelo:

| Ferramenta | O que monitorar | Frequência |
|---|---|---|
| Sentry | Novas issues, taxa de erros 5xx | Contínuo |
| Langfuse | Traces de IA, tokens, latência de modelo | A cada 15min |
| PostHog | Volume de eventos, feature flag adoption | A cada 15min |
| RabbitMQ Management UI | Consumer lag, DLQ | A cada 15min |
| `/health` endpoint | Status de todas as dependencies | A cada 5min |
| Logs do container | Erros inesperados, warnings | Contínuo |

### 5.2 Queries de Monitoramento Rápido

```sql
-- Volume de interações por hora (D0)
SELECT
  DATE_TRUNC('hour', created_at) as hora,
  COUNT(*) as total,
  COUNT(*) FILTER (WHERE status = 'RESPONDIDA_PELA_IA') as resolvidas,
  COUNT(*) FILTER (WHERE status = 'SINALIZADA_PARA_REVISAO') as sinalizadas,
  ROUND(AVG(confidence_score)::numeric, 3) as avg_confidence
FROM interactions
WHERE created_at > NOW() - INTERVAL '8 hours'
GROUP BY 1
ORDER BY 1 DESC;

-- Taxa de erros por endpoint (via logs — adaptar para sistema de logs)
-- Monitorar via Sentry diretamente

-- Takeovers ativos (não devem acumular)
SELECT COUNT(*) as takeovers_ativos
FROM takeovers
WHERE ended_at IS NULL;
```

---

## 6. Comunicação de Go-Live

### 6.1 Comunicado Pré-Go-Live (D-1)

```
Para: Time de Engenharia + Produto + Suporte
Canal: Slack #geral

Amanhã, [DATA], realizaremos o go-live do AI-Dani-Admin v1.0.

O que muda:
- Painel Admin agora disponível com monitoramento de interações Dani
- Takeover manual habilitado
- Dashboard de métricas e alertas automáticos ativos

Janela: 10h–14h (horário de Fortaleza)
Rollout progressivo: 10% → 50% → 100%
Kill switch disponível: sem necessidade de deploy para reverter

Em caso de problemas: @on-call no Slack #alertas-criticos
```

### 6.2 Comunicado Pós-Go-Live

```
Para: Stakeholders + Time completo
Canal: Slack #geral

✅ AI-Dani-Admin v1.0 — GO-LIVE CONCLUÍDO

Horário: [TIMESTAMP]
Status: Produção estável — rollout 100%

Primeiras métricas (2h):
- Interações processadas: [N]
- Confidence médio: [X]%
- CSAT médio: [X.X]
- Taxa de erros: [X]%

Documentação: [link para docs]
Próxima revisão: D+7
```

---

## 7. Definição de Pronto — Go-Live Completo

O go-live é considerado completo quando:

- [ ] `webchat-enabled` em 100% para todos os usuários
- [ ] Health check verde por 2h contínuas após rollout 100%
- [ ] Sentry sem novas issues P0/P1
- [ ] Comunicado final enviado para stakeholders
- [ ] Monitoramento D+1 agendado
- [ ] Logs do go-live arquivados em `logs/relatorio-final.md`

---

## 8. Resumo de Documentação de Suporte

| Situação | Documento | Seção |
|---|---|---|
| Configurar ambiente de desenvolvimento | D22 | Seção 2 — Setup em 5 Passos |
| Entender os endpoints disponíveis | D16 | Todos os endpoints |
| Responder a incidente P0 | D26 | Seção 2 — Runbook P0 |
| Usar kill switch | D26 | Seção 6 — Kill Switch de IA |
| Fazer rollback de código | D24 | Seção 7 — Estratégia de Rollback |
| Aplicar migration | D26 | Seção 7 — Migrations em Produção |
| Verificar observabilidade | D25 | Todos |
| Checar qualidade antes do deploy | D28 | Seção 7 — Checklist de Pre-Deploy |
| Entender as regras de negócio | D01 | Todos |

---

## 9. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Cronograma D-7 a D+7, critérios Go/No-Go, rollout progressivo via feature flags, plano de rollback, monitoramento de go-live, comunicações. |
