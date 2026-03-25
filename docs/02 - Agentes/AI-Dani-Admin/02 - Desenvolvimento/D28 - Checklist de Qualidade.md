# Checklist de Qualidade — AI-Dani-Admin

## Gates de Qualidade e Critérios de Aceitação Técnica

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Backend, Tech Lead, DevOps |
| Escopo | Checklist completo de qualidade para releases do módulo AI-Dani-Admin — cobertura de testes, segurança, performance, observabilidade e conformidade |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D01 (Regras de Negócio), D14-D27 (todos os docs de desenvolvimento) |

---

> **📌 TL;DR**
>
> - **Gate obrigatório:** todos os itens P0 devem estar ✅ antes de qualquer deploy em produção.
> - **5 categorias:** Código e Testes → Segurança → Performance → Observabilidade → Go-Live.
> - **Itens P0 (bloqueantes):** cobertura ≥ 80%, zero lint errors, filtro de escopo funcionando, zero PII em logs, health check ok, migrations testadas.
> - **Itens P1 (recomendados):** cobertura ≥ 85%, adversarial tests passando, dashboards configurados.
> - **Este checklist é executado a cada release** — não é apenas para o primeiro deploy.

---

## 1. Código e Testes

### 1.1 Cobertura de Testes (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Cobertura total ≥ 80% | `npm run test:cov` | Resultado no CI ≥ 80% | ☐ |
| Cobertura do módulo `takeover` ≥ 85% | Takeover é módulo crítico | Coverage report por módulo | ☐ |
| Zero testes falhando | CI verde | GitHub Actions: todos os checks passando | ☐ |
| Testes E2E rodando | `npm run test:e2e` | Zero falhas em ambiente de teste | ☐ |
| Lock otimista testado (takeover simultâneo) | ADR-001 | Teste `[CRÍTICO]` no takeover.service.spec.ts | ☐ |
| Adversarial tests passando (7 categorias) | RN-DA-037, RN-DA-038 | scope-filter.spec.ts: 100% `wasRefused: true` | ☐ |

### 1.2 Qualidade de Código (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Zero lint errors | `npm run lint` | CI: ESLint sem erros | ☐ |
| Zero `console.log` em código de produção | Proibido por D23 | `grep -r "console.log" apps/api/src` deve retornar vazio | ☐ |
| Zero tipos `any` explícitos em código crítico | Proibido por D23 | TypeScript strict mode sem suppressions | ☐ |
| Zero imports cross-module diretos | D23 | Revisão manual ou lint rule customizada | ☐ |
| Conventional Commits em todos os commits da branch | D23 | Título do squash commit segue formato | ☐ |
| `.env.example` atualizado para novas variáveis | D22 | Diff do `.env.example` inclui novas vars | ☐ |

### 1.3 Qualidade de Código (P1 — Recomendado)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Cobertura total ≥ 85% | Acima do mínimo | Verificação adicional | ☐ |
| Nenhum `TODO` pendente em código commitado | Evitar dívida técnica | `grep -r "TODO\|FIXME" apps/api/src` | ☐ |
| Documentação JSDoc nos Services públicos | Legibilidade | Revisão de código | ☐ |

---

## 2. Segurança

### 2.1 Autenticação e Autorização (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| `AdminAuthGuard` em todos os endpoints `/api/v1/admin/*` | RN-DA-001 | Revisão de cada Controller | ☐ |
| Role `ADMIN` verificada no JWT | D18 | Teste E2E com token de não-admin → 403 | ☐ |
| Endpoint interno `/api/v1/internal/interactions` protegido por API key | D16 | Teste sem chave → 401 | ☐ |
| Refresh token rotation funcionando | D18 | Teste: token expirado → refresh → novo token | ☐ |

### 2.2 Filtro de Escopo e Contexto (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| `ScopeFilter` bloqueando dados de outros usuários | RN-DA-037 | Adversarial test categoria 1 passando | ☐ |
| `ContextFilter` garantindo docs apenas do usuário autenticado | RN-DA-037 | Teste de isolamento de contexto | ☐ |
| Prompt injection bloqueado | RN-DA-038 | Adversarial tests categorias 2-7 passando | ☐ |
| Agente recusa quando fora do escopo | RN-DA-038 | Resposta de recusa genérica, sem vazamento | ☐ |

### 2.3 Proteção de Dados (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Zero PII em logs de aplicação | D20, D25 | Buscar `email`, `cpf`, `phone` nos logs de staging | ☐ |
| `JWT_SECRET` não exposto em logs ou código | D22 | `grep -r "JWT_SECRET" apps/api/src` → vazio | ☐ |
| `SUPABASE_SERVICE_ROLE_KEY` não exposto | D22 | Idem para service role key | ☐ |
| Pino redact configurado corretamente | D25 | Testar log com objeto contendo email → `[REDACTED]` | ☐ |
| Rate limit ativo no webchat | D16 | Teste de rate limit (30 req/hora) → 429 no 31º | ☐ |

### 2.4 Segurança (P1 — Recomendado)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Headers de segurança configurados (Helmet) | Boas práticas | Verificar resposta: X-Content-Type-Options, etc. | ☐ |
| CORS configurado para domínios permitidos | D18 | Requisição de origem não autorizada → 403 | ☐ |
| Secrets rotacionados há < 90 dias | D22 | Verificar data da última rotação no log de auditoria | ☐ |

---

## 3. Performance e Confiabilidade

### 3.1 Performance (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Endpoint mais lento ≤ 500ms (p95, exceto IA) | D14 | Load test básico ou verificação manual | ☐ |
| Cache Redis funcionando para métricas (TTL 60s) | D14 | Requisição repetida → latência < 10ms | ☐ |
| Cache Redis funcionando para agent-config (TTL 300s) | D14 | Verificar hit/miss no log | ☐ |
| Nenhum `await` em loop (N+1 queries) | D23 | Revisão de código + query count no teste de integração | ☐ |

### 3.2 Confiabilidade (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| FallbackAtivo ativo quando OpenAI indisponível | RF-018 | Mock OpenAI → 503 → resposta de fallback | ☐ |
| DLQ configurada para todos os consumers | D14 | Verificar filas no RabbitMQ Management UI | ☐ |
| Retry 3x configurado nos consumers | D14 | Simular falha → verificar tentativas no log | ☐ |
| Migrations rodando sem erro no container | D24 | Verificar `CMD` do Dockerfile no boot | ☐ |

### 3.3 Performance (P1 — Recomendado)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Índices do banco configurados para queries de paginação | D13 | `EXPLAIN ANALYZE` nas queries principais | ☐ |
| Resposta de IA em modo streaming funcionando | D16 | SSE: verificar chunks chegando progressivamente | ☐ |
| Supabase Realtime broadcasting alertas | D21 | Simular alerta → verificar evento no painel | ☐ |

---

## 4. Observabilidade

### 4.1 Logs e Monitoramento (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| `/health` retornando 200 com todas as dependencies `ok` | D25, D26 | `curl https://api.repasseseguro.com.br/health` | ☐ |
| Pino emitindo logs estruturados em JSON | D25 | Verificar output de container em staging | ☐ |
| Sentry capturando erros 5xx | D25 | Forçar erro 5xx → verificar issue no Sentry | ☐ |
| Sentry NÃO capturando erros 4xx | D25 | Erro 404 → Sentry sem nova issue | ☐ |

### 4.2 Observabilidade (P1 — Recomendado)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Langfuse registrando traces de IA | D25 | Request ao webchat → trace no Langfuse | ☐ |
| PostHog registrando eventos de negócio | D25 | takeover_started → evento no PostHog | ☐ |
| Feature flags respondendo corretamente | D25 | `webchat-enabled: false` → 503 no webchat | ☐ |
| Alertas Slack P0/P1 configurados | D25 | Simular falha → Slack #alertas-prod | ☐ |
| `X-Correlation-Id` presente em todas as respostas | D25 | Verificar header nas respostas | ☐ |

---

## 5. Migrations e Banco de Dados

### 5.1 Migrations (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| `prisma migrate status` sem migrations pendentes | D26 | Output: "All migrations are up to date" | ☐ |
| Migrations testadas em staging antes de produção | D24 | Confirmar aplicação bem-sucedida em staging | ☐ |
| Migration de rollback escrita (se migration destrutiva) | D24 | Arquivo de rollback presente em `prisma/migrations/` | ☐ |
| Seed de dados de desenvolvimento funcionando | D22 | `npm run seed` → AgentConfiguration criada | ☐ |

---

## 6. Conformidade com Documentação

### 6.1 Rastreabilidade (P0 — Bloqueante)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Todos os endpoints de D16 implementados | D16 | Testar cada endpoint listado | ☐ |
| Error codes seguindo formato `DA-{MODULE}-{NNN}` | D20 | Verificar todas as exceções no código | ☐ |
| Error shape `{ error: { code, message } }` em todas as exceções | D20 | Testes de error shape (D27) passando | ☐ |
| RN-DA-037 (filtro de escopo) implementada e testada | D01, D19 | Adversarial tests passando | ☐ |
| RF-018 (FallbackAtivo) implementada | D05, D19 | Teste de fallback passando | ☐ |
| RF-011 (lock otimista de takeover) implementado | D05, D14 | Teste de takeover simultâneo passando | ☐ |

### 6.2 Convenções (P1 — Recomendado)

| Item | Critério | Verificação | Status |
|---|---|---|---|
| Nomenclatura de arquivos seguindo D15 | D15 | Revisão da estrutura de pastas | ☐ |
| Redis keys com prefixo `dani-admin:` | D15 | `redis-cli KEYS dani-admin:*` em staging | ☐ |
| Queues RabbitMQ com nomenclatura `dani-admin.*` | D14 | RabbitMQ Management UI → verificar fila | ☐ |

---

## 7. Checklist de Pre-Deploy — Resumo Executivo

Use este checklist condensado imediatamente antes de cada deploy em produção:

```
PRE-DEPLOY CHECKLIST — AI-Dani-Admin
=====================================

TESTES
[ ] CI verde (lint + tests + build)
[ ] Cobertura ≥ 80% (report anexo ao PR)
[ ] E2E passando em staging

SEGURANÇA
[ ] AdminAuthGuard em todos os endpoints admin
[ ] Adversarial tests passando
[ ] Zero PII em logs (verificado em staging)
[ ] Secrets atualizados no ambiente

BANCO DE DADOS
[ ] Migrations testadas em staging
[ ] Migration de rollback preparada (se destrutiva)
[ ] `prisma migrate status` sem pendências

OBSERVABILIDADE
[ ] /health → 200 OK em staging
[ ] Sentry DSN configurado
[ ] Kill switch (webchat-enabled) testado

DEPLOY
[ ] Tag semântica criada (vX.Y.Z)
[ ] Aprovação no GitHub environment
[ ] Notificação da equipe no Slack #deploys

Responsável: _______________
Data: _______________
```

---

## 8. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. 5 categorias de qualidade, itens P0/P1, checklist executivo de pre-deploy, rastreabilidade para D01/D05/D14-D27. |
