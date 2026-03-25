# 29 - Go-Live Playbook

| Campo | Valor |
|---|---|
| **Nome do Documento** | 29 - Go-Live Playbook |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Data** | 2026-03-23 |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Produto** | Repasse Seguro — Módulo Cedente |

---

> **TL;DR — Resumo Operacional do Lançamento do Módulo Cedente**
>
> - **Preparação:** 3 janelas — T-7 (testes, infraestrutura e integrações), T-3 (staging final com sandbox ZapSign e Escrow), T-1 (war room e code freeze).
> - **Go/No-Go:** decisão do Tech Lead com base em: E2E verde, smoke tests passando, ZapSign e Escrow sandbox operacionais, error rate < 1% em staging, isolamento LGPD validado, Guardião do Retorno com < 20% de escalamentos.
> - **Launch Day:** terça a quinta, 10h-14h BRT. Deploy automático (Railway) + aprovação manual no GitHub.
> - **Monitoramento pós-launch:** intensivo por 2 horas (T+15min, T+1h, T+2h). War room encerra após T+2h se estável.
> - **Rollback:** feature flag desativa Módulo Cedente sem afetar outros módulos. `railway rollback` em < 60s. Gatilho: error rate > 5% ou P95 > 1s ou Escrow não respondendo.
> - **Critérios Go/No-Go específicos do Cedente:** taxa de erro < 1%, P95 < 200ms, ZapSign respondendo, Escrow respondendo, Guardião com < 20% de escalamentos.
> - **Responsáveis:** Tech Lead (decisão final), on-call (execução), suporte plataformas externas (ZapSign, Escrow).

---

## 1. Matriz de Escalation

| Severidade | Impacto no Módulo Cedente | Canal | Pessoas acionadas | SLA de resposta | Critério de escalação |
|---|---|---|---|---|---|
| **P0 — Crítico** | API down, Escrow falhou, ZapSign inacessível, dados de Cedentes expostos, cálculo de comissão errado | PagerDuty + Slack `#incidentes` | Tech Lead + on-call | 5 minutos | Qualquer falha em fluxo financeiro ou violação de LGPD |
| **P1 — Alto** | Funcionalidade principal degradada (aceite de proposta, upload de dossiê, Guardião do Retorno), assinatura ZapSign lenta | Slack `#incidentes` | Tech Lead | 15 minutos | P0 não resolvido em 15min → escalar para responsável técnico |
| **P2 — Médio** | Latência elevada sem bloqueio operacional, webhook ZapSign com atraso mas funcional, notificações atrasadas | Slack `#alertas` | On-call técnico | 30 minutos | P1 não resolvido em 1h → Tech Lead |
| **P3 — Baixo** | Anomalia não urgente (ex: skeleton loading lento, badge de status com atraso) | Slack `#alertas` ou issue | On-call técnico | 2 horas | — |

**Contatos de suporte externo (confirmar antes do go-live):**

| Serviço | Função no Módulo Cedente | Canal de suporte | SLA de resposta contratual |
|---|---|---|---|
| ZapSign | Assinatura eletrônica de Termos de Cadastro, Aceite de Escalonamento, Termo Comercial, Instrumento de Cessão | [DADO PENDENTE — email/telefone de suporte técnico ZapSign] | [DADO PENDENTE] |
| Parceiro Escrow (fintech/banco) | Conta garantia para depósito do Cessionário, retenção por 15 dias e distribuição ao Cedente | [DADO PENDENTE — email/telefone de suporte técnico Escrow] | [DADO PENDENTE] |
| Supabase | Banco de dados, autenticação e storage de documentos do dossiê | https://supabase.com/support + Discord | Plano Pro: 1 dia útil |
| Railway | Backend NestJS em produção | https://railway.app/help | Plano Pro: email |
| Resend | E-mails transacionais (lembretes de dossiê, notificações de proposta, confirmação de aceite) | [DADO PENDENTE — suporte Resend] | [DADO PENDENTE] |
| Expo / EAS | Build e push notifications do app mobile | https://expo.dev/support | Depende do plano |

---

## 2. Health Checks

### 2.1 Endpoints de Health (Verificar a cada 5min durante launch window)

| Endpoint | Método | Owner | Threshold de sucesso | Interpretação de falha |
|---|---|---|---|---|
| `https://app.repasse-seguro.com.br/health` | GET | Automático (CI smoke test) | HTTP 200, `{ "status": "ok" }` em < 500ms | API offline ou instância não-saudável → P0 imediato |
| `https://api.repasse-seguro.com.br/api/v1/cedente/casos` | GET + Bearer token | Tech Lead | HTTP 200, latência < 500ms | Banco ou auth do Cedente com problema → P1 |
| `https://api.repasse-seguro.com.br/api/v1/auth/login` | POST (payload de teste) | Tech Lead | HTTP 200 ou 401 (sem 5xx) | Auth degradado → P1 |
| `https://api.repasse-seguro.com.br/api/v1/cedente/simulador` | GET + Bearer token | Tech Lead | HTTP 200, latência < 300ms | Simulador de cenários indisponível → P1 |

### 2.2 Dashboards Críticos (Monitorar continuamente durante launch window)

| Dashboard | URL | O que observar | Threshold de alerta |
|---|---|---|---|
| Railway Metrics | https://railway.app/dashboard | CPU, memória, instâncias ativas | CPU > 80%, memória > 85% |
| Sentry | https://sentry.io/{org}/repasse-seguro | Taxa de erros, novos issues do módulo Cedente | Novos P0 ou taxa > 1% em 5min |
| Supabase Dashboard | https://app.supabase.com | Conexões, latência de queries, tabelas de Cedentes | Conexões > 80% do pool |
| Langfuse | https://cloud.langfuse.com | Latência do Guardião, taxa de erro LLM, % de escalamentos | p95 > 5s, taxa de erro > 5%, escalamentos > 20% |
| Upstash Redis | https://console.upstash.com | Memória, operações/s, chaves de sessão de Cedentes | Memória > 70%, `ping` sem resposta |

### 2.3 Smoke Tests Automatizados (Executar imediatamente após deploy)

```bash
# Executado automaticamente pelo CI após deploy em produção
pnpm test:smoke:prod
# Verifica:
# - GET /health → 200
# - GET /api/v1/cedente/casos (sem token) → 401 (não 5xx)
# - POST /api/v1/auth/login (payload inválido) → 400 (não 5xx)
# - GET /api/v1/cedente/simulador (sem token) → 401 (não 5xx)
# - POST /api/v1/cedente/guardiao/chat (sem token) → 401 (não 5xx)
# Falha: retorna exit code 1 → rollback automático via CI
```

---

## 3. Pré-Launch

### 3.1 T-7 (7 dias antes do go-live)

**Owner: Tech Lead**

**Testes e qualidade:**
- [ ] Suíte E2E completa passando no staging sem falhas intermitentes.
- [ ] Todos os fluxos E2E P0 (E2E-001 a E2E-006 conforme D27) verdes.
- [ ] Cobertura de testes ≥ 80% em módulos críticos; 100% em `CommissionCedenteService` e lógica de Escrow.
- [ ] `pnpm audit --audit-level=high` sem vulnerabilidades críticas.
- [ ] D28 (Checklist de Qualidade) revisado e aprovado pelo Tech Lead.
- [ ] Testes de segurança SEC-001 a SEC-011 (D27) todos passando.
- [ ] Isolamento LGPD entre Cedentes validado (SEC-001 a SEC-005 do D27).

**Integrações:**
- [ ] ZapSign: URL de webhook de staging configurada no painel ZapSign. Teste de webhook com payload real de assinatura de Termo de Cadastro.
- [ ] ZapSign: validação HMAC-SHA256 testada com payload válido e inválido (INT-020 do D27).
- [ ] Escrow (parceiro): credenciais de sandbox configuradas. Fluxo de depósito + distribuição testado no sandbox.
- [ ] Escrow: webhook de sandbox testado com payload de depósito confirmado, distribuição e estorno.
- [ ] Resend: domínio de produção verificado e e-mails de teste entregues (lembretes de dossiê, confirmação de aceite de proposta).
- [ ] Expo Notifications: credenciais APNs e FCM de staging configuradas. Push notifications testados em dispositivo real.

**Infraestrutura:**
- [ ] 3 projetos Supabase criados e configurados: dev, staging, production.
- [ ] RLS habilitado em todas as tabelas de Cedentes em produção. Verificação: `SELECT * FROM pg_policies WHERE tablename LIKE '%cedente%' OR tablename LIKE '%caso%' OR tablename LIKE '%proposta%' OR tablename LIKE '%documento%'`.
- [ ] Railway service `backend-prod` criado com health check configurado em `/health`.
- [ ] Vercel project configurado para frontend com environment variables corretas.
- [ ] Upstash Redis prod criado e conectado.

**Secrets:**
- [ ] Todos os secrets de produção adicionados ao Railway (backend) e Vercel (frontend):
  - `ZAPSIGN_API_KEY`, `ZAPSIGN_WEBHOOK_SECRET`
  - `ESCROW_API_KEY`, `ESCROW_WEBHOOK_SECRET`
  - `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` (somente backend)
  - `JWT_SECRET`, `REDIS_URL`
  - `OPENAI_API_KEY` (Guardião do Retorno)
  - `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`
  - `POSTHOG_API_KEY`
  - `SENTRY_DSN`
  - `RESEND_API_KEY`
- [ ] EAS Secrets configurados para o app mobile.
- [ ] Nenhum secret no código ou `.env` commitado.
- [ ] `.env.example` atualizado com todas as variáveis (sem valores).

---

### 3.2 T-3 (3 dias antes do go-live)

**Owner: Tech Lead**

**Validação de staging (simula produção):**
- [ ] Deploy completo em staging executado a partir da branch `main` (mesma que produção).
- [ ] Smoke tests de staging passando: `/health`, `/api/v1/cedente/casos`, `/api/v1/cedente/simulador`.
- [ ] Fluxo completo E2E executado manualmente em staging: cadastro de imóvel (wizard 5 etapas) → upload de dossiê → assinatura do Termo de Cadastro (ZapSign sandbox) → recebimento de proposta → aceite → formalização → Escrow sandbox.
- [ ] Sentry configurado e capturando erros no staging (trigger manual de teste: lançar exceção conhecida).
- [ ] Langfuse recebendo traces do Guardião do Retorno em staging. Simular 5 consultas e verificar traces.
- [ ] PostHog capturando todos os eventos obrigatórios (tabela 5.4 do D28) em staging.
- [ ] Notificações de lembrete de dossiê testadas manualmente em staging (7 dias sem upload → e-mail).
- [ ] Notificação de proposta recebida testada (e-mail + push notification mobile).

**Validação de integrações (sandbox → produção):**
- [ ] ZapSign sandbox → produção: credenciais de produção obtidas e testadas. Documento de teste criado e assinado no ambiente de produção ZapSign.
- [ ] Escrow sandbox → produção: credenciais de produção obtidas. Fluxo completo testado: abertura de conta, depósito simulado, confirmação, distribuição simulada.
- [ ] Confirmação de que os webhooks do ZapSign e Escrow estão apontando para a URL de produção correta.

**Comunicação:**
- [ ] Canais Slack `#deploys`, `#incidentes` e `#alertas` criados e testados.
- [ ] PagerDuty configurado para alertas P0 do Módulo Cedente.
- [ ] Lista de contatos de escalation confirmada e atualizada (incluindo suporte ZapSign e Escrow).
- [ ] Janela de deploy aprovada: terça-quinta, 10h-14h BRT (decisão da semana do go-live).

**Observabilidade:**
- [ ] Alertas configurados no Sentry para erros P0/P1 do Módulo Cedente.
- [ ] Alerta Railway: CPU > 80% e memória > 85%.
- [ ] Alerta Langfuse: Guardião com p95 > 5s ou taxa de escalamentos > 20%.
- [ ] Dashboard "Saúde do Módulo Cedente" operacional com métricas em tempo real.

---

### 3.3 T-1 (1 dia antes do go-live)

**Owner: Tech Lead + QA**

**Freeze:**
- [ ] Code freeze ativado em `develop`. Nenhum novo PR mergeado exceto hotfix crítico aprovado pelo Tech Lead.
- [ ] Versão de release candidata (`v1.0.0-cedente`) em staging, estável por pelo menos 4h.

**Verificações finais:**
- [ ] PR `develop → main` aberto com changelog e release notes do Módulo Cedente.
- [ ] 2 aprovações obtidas no PR de release.
- [ ] Migrations Prisma revisadas: todas retrocompatíveis, rollback documentado em cada arquivo SQL.
- [ ] Bundle size frontend verificado: < 300KB gzipped.
- [ ] Dados de teste de staging removidos (conformidade LGPD): `DELETE FROM cedentes WHERE id LIKE 'test-%'`.

**War room:**
- [ ] War room (Slack huddle ou Google Meet) agendado para o horário do launch.
- [ ] Participantes confirmados: Tech Lead, on-call, responsável de produto.
- [ ] D28 (Checklist de Qualidade) e D29 (este playbook) abertos e acessíveis durante o war room.
- [ ] Links dos dashboards críticos organizados em documento compartilhado.

**Comunicação pré-launch:**
- [ ] Mensagem pré-launch enviada no canal `#deploys`:
  ```
  Go-Live Módulo Cedente agendado para amanhã [data] às [hora] BRT.
  Responsável: @[nome] | War room: [link]
  Monitorar: [link dashboards]
  Versão: v1.0.0-cedente
  ```

---

## 4. Launch Day

**Janela de deploy:** terça a quinta, 10h-14h BRT.

[DECISÃO AUTÔNOMA — janela 10h-14h BRT, terça a quinta. Justificativa: evita segundas (ajuste pós-fim de semana), sextas (time reduzido para resposta), manhã cedo (equipe não totalmente disponível) e fim de tarde (pouco tempo para estabilização antes do fim do expediente). Alternativa descartada: qualquer horário — sem janela definida o risco de deploy fora do horário ideal é alto para produto com transações financeiras.]

### 4.1 Checklist de Launch Day

| Horário | Atividade | Owner | Critério de sucesso | Se falhar |
|---|---|---|---|---|
| T-60min | War room iniciado. Dashboards abertos. | Tech Lead | Todos os participantes presentes | Reagendar se Tech Lead indisponível |
| T-60min | Verificar status de integrações: ZapSign (status page), Escrow (status/ping), Supabase, Railway | Tech Lead | Todas operacionais | No-go se P0 ou P1 em integração crítica |
| T-30min | Executar smoke tests em staging pela última vez | QA | Todos passando | No-go se qualquer smoke test falhar |
| T-30min | Executar dry-run do fluxo: cadastro de imóvel → dossiê → proposta → aceite (staging) | QA | Fluxo completo sem erro | No-go se qualquer etapa falhar |
| T-15min | Verificar que PR `develop → main` está pronto para merge | Tech Lead | PR aprovado com 2 revisores | Obter aprovações restantes |
| T-0 | **GO/NO-GO decision** (ver seção 12) | Tech Lead | Todos os critérios Go satisfeitos | No-go → reagendar |
| T-0 | Merge PR `develop → main` | Tech Lead | CI verde após merge | Investigar falha de CI antes de continuar |
| T+2min | CI disparou workflow `deploy-production.yml` | Automático | Workflow iniciado no GitHub Actions | Verificar trigger manual |
| T+5min | **Aprovação manual no GitHub Environment** `production` | Tech Lead | Workflow aguarda aprovação | Aprovar somente se smoke tests staging ok |
| T+8min | Deploy Rolling iniciado no Railway | Automático | Railway mostra "Deploying" | Verificar Railway Dashboard |
| T+12min | Deploy concluído — Railway mostra instância active | Automático | `railway status` → healthy | Rollback imediato se unhealthy |
| T+13min | Smoke tests de produção executados pelo CI | Automático | Todos passando | Rollback automático se falha |
| T+15min | **Verificação manual — Health Check #1** | Tech Lead | `/health` retorna 200; Sentry sem novos erros; Escrow respondendo | Rollback se error rate > 1% |
| T+20min | Mensagem de lançamento no canal `#releases` | Tech Lead | — | — |

---

## 5. Pós-Launch

### 5.1 T+15 minutos

**Owner: Tech Lead (em war room)**

- [ ] `/health` retornando 200 com versão correta.
- [ ] Taxa de erros 5xx < 1% no Railway Metrics.
- [ ] Sentry: nenhum novo issue crítico do Módulo Cedente.
- [ ] Latência p95 < 200ms (endpoints críticos: `/cedente/casos`, `/cedente/simulador`, `/cedente/propostas`).
- [ ] Railway: todas as instâncias com status "Active".
- [ ] Redis: `ping → PONG`; memória < 70%.
- [ ] ZapSign: webhook de teste processado com sucesso.
- [ ] Escrow: webhook de teste processado com sucesso.
- [ ] Langfuse: Guardião recebendo e respondendo consultas (testar 1 consulta manual).

**Critério de go:** todos os itens verdes → continuar monitoramento.
**Critério de no-go:** qualquer item vermelho → avaliar rollback (ver seção 8).

---

### 5.2 T+1 hora

**Owner: Tech Lead (em war room)**

- [ ] Taxa de erros 5xx < 0.5% acumulada desde o deploy.
- [ ] p95 de latência < 200ms sustentado em todos os endpoints críticos.
- [ ] PostHog: eventos `caso_cadastrado`, `cenario_selecionado`, `proposta_aceita` (se aplicável) sendo capturados.
- [ ] Langfuse: latência do Guardião estável (p95 < 5s); taxa de escalamentos < 20%.
- [ ] Custo OpenAI dentro do esperado para 1h de operação do Guardião.
- [ ] Supabase: conexões com pool < 80%; sem queries lentas (p95 < 500ms no Supabase Dashboard).
- [ ] Nenhum alerta P0 ou P1 aberto sem resolução.

**Critério de normalização:** todos os itens verdes → escalar monitoramento para rotina.

---

### 5.3 T+2 horas — Encerramento do War Room

**Owner: Tech Lead**

- [ ] Verificação final de todos os health checks (repetir tabela da seção 2.1).
- [ ] Nenhum incidente aberto.
- [ ] Mensagem de encerramento no canal `#incidentes`:
  ```
  War room encerrado. Lançamento do Módulo Cedente estável.
  Versão: v1.0.0-cedente | Deploy: [hora] BRT
  Próxima verificação: T+24h às [hora] BRT.
  ```
- [ ] Monitoramento passa para modo de rotina.

---

### 5.4 D+1 (24 horas)

**Owner: Tech Lead**

- [ ] Revisar métricas das primeiras 24h: Railway, Sentry, Langfuse, PostHog.
- [ ] Verificar funil de cadastro: quantos Cedentes iniciaram o wizard, quantos concluíram a Etapa 5.
- [ ] Verificar taxa de abandono no simulador de cenários (Etapa 3): abandono > 60% indica problema de UX.
- [ ] Verificar taxa de upload de dossiê: documentos rejeitados com motivos — padrão de rejeição informa necessidade de melhoria.
- [ ] Verificar webhooks ZapSign e Escrow: sem mensagens na DLQ.
- [ ] Criar issue de post-launch com observações para próximo release (se aplicável).

---

### 5.5 D+7 — Relatório de Primeiros 7 Dias

**Owner: Tech Lead + responsável de produto**

**Métricas a coletar (PostHog + Sentry + Langfuse):**

| Métrica | Onde coletar | Meta de 7 dias |
|---|---|---|
| Cedentes cadastrados (wizard concluído) | PostHog — evento `caso_cadastrado` | [DADO PENDENTE — meta do produto] |
| Taxa de conclusão do wizard (Etapa 1 → Etapa 5) | PostHog — funil de cadastro | > 40% |
| Tempo médio no simulador de cenários (Etapa 3) | PostHog — `tempo_visualizacao_simulador_segundos` | > 15s (indica engajamento real) |
| Distribuição de escolha de cenário (A, B, C, D) | PostHog — evento `cenario_selecionado` | Qualitativo |
| Propostas aceitas / propostas recebidas | PostHog — taxa aceite/recebimento | > 30% |
| Taxa de erro global 5xx | Sentry | < 0.5% |
| P95 de latência (endpoints críticos) | Railway Metrics | < 200ms |
| Taxa de escalamentos do Guardião | Langfuse | < 20% |
| Documentos rejeitados por tipo | Supabase query | Qualitativo |
| Tickets de suporte relacionados ao Módulo Cedente | [DADO PENDENTE — ferramenta de suporte] | < 5 P0/P1 |

**Critério de sucesso aos 7 dias:**
- Taxa de erro < 1% (acumulado nos 7 dias).
- P95 < 200ms em todos os endpoints críticos.
- ZapSign operacional: nenhum Termo bloqueado por falha técnica.
- Escrow operacional: nenhum depósito ou distribuição falhado.
- Guardião: taxa de escalamentos < 20%.

---

## 6. Critérios Go/No-Go

### 6.1 Critérios de Go (todos obrigatórios)

| Critério | Threshold | Fonte de verificação |
|---|---|---|
| E2E P0 passando em staging | 100% verde — zero falhas nos fluxos E2E-001 a E2E-006 | Playwright CI |
| Smoke tests de staging | 100% passando | CI |
| Taxa de erro em staging | < 1% em 1 hora de carga | k6 load test |
| P95 de latência em staging | < 200ms em endpoints críticos | k6 load test |
| ZapSign sandbox operacional | Webhook de teste processado e assinatura de Termo de Cadastro concluída | Teste manual |
| Escrow sandbox operacional | Fluxo completo (depósito + distribuição) testado | Teste manual |
| Guardião do Retorno | < 20% de escalamentos em consultas de teste | Langfuse staging |
| Isolamento LGPD | SEC-001 a SEC-005 do D27 passando | Playwright CI |
| RLS no banco de dados | `SELECT * FROM pg_policies` valida todas as tabelas de Cedentes | SQL manual |
| D28 aprovado | Checklist de Qualidade assinado pelo Tech Lead | Documento |
| Dados de teste removidos de staging | Confirmação manual | SQL manual |
| Secrets de produção configurados | Verificação no Railway e Vercel | Manual |

### 6.2 Critérios de No-Go (qualquer um impede o lançamento)

| Critério | Condição | Ação |
|---|---|---|
| E2E falhando | Qualquer fluxo P0 vermelho | Corrigir e re-executar. Adiar go-live. |
| ZapSign inacessível | Status page com incidente ativo | Aguardar resolução. Adiar go-live. |
| Escrow inacessível | Serviço não respondendo | Aguardar resolução. Adiar go-live. |
| Cálculo de comissão errado | Qualquer falha em `CommissionCedenteService` (100% coverage obrigatório) | Corrigir. Re-executar testes. Adiar go-live. |
| Isolamento LGPD falhando | Qualquer SEC-00X vermelho | Correção obrigatória. Não há exceção. |
| RLS ausente em tabela de Cedente | `pg_policies` não cobre tabela crítica | Aplicar migration de RLS. Re-testar. |
| Performance degradada em staging | P95 > 500ms em endpoints críticos | Investigar e corrigir. Adiar go-live. |
| Vulnerabilidade crítica em dependências | `pnpm audit --audit-level=high` com resultado | Atualizar dependência. Re-auditar. |
| Secret exposto | Credencial no código ou git history | Stop total. Rotacionar secret. Auditar. |

---

## 7. Plano de Rollback

### 7.1 Gatilhos de Rollback

| Gatilho | Tipo | Tempo de resposta |
|---|---|---|
| Smoke tests de produção falhando | Automático (CI) | Imediato |
| Taxa de erros 5xx > 5% em 5min pós-deploy | Manual (Tech Lead) | < 2 minutos |
| P95 > 1s em endpoints críticos por mais de 10min | Manual (Tech Lead) | < 5 minutos |
| ZapSign não respondendo (webhooks perdidos > 3) | Manual (Tech Lead) | < 5 minutos |
| Escrow não respondendo (depósito ou distribuição falhando) | Manual (Tech Lead) | Imediato — P0 financeiro |
| Violação de isolamento LGPD detectada | Manual (Tech Lead) | Imediato — P0 legal |
| Qualquer P0 identificado na launch window | Manual (Tech Lead) | Imediato |

[DECISÃO AUTÔNOMA — threshold de rollback em 5% de error rate (mais conservador que o Módulo Cessionário com 2%). Justificativa: o Módulo Cedente concentra a entrada de dados de usuário (dossiê, cenário, proposta) e uma falha aqui pode resultar em perda de rascunhos e frustração no início da jornada do usuário. Alternativa descartada: threshold de 10% — tolerância alta demais para módulo de onboarding.]

### 7.2 Estratégia de Rollback por Feature Flag

O Módulo Cedente usa feature flag para permitir desativação sem afetar outros módulos da plataforma.

```typescript
// Feature flag: CEDENTE_MODULE_ENABLED
// Configurado via PostHog ou variável de ambiente no Railway
// Quando desativado:
// - Rotas /cedente/* retornam HTTP 503 com mensagem de manutenção
// - Sidebar do painel não exibe itens do Módulo Cedente
// - App mobile exibe banner "Módulo temporariamente indisponível"
// - Módulo Cessionário, Admin e outros módulos continuam operando normalmente
```

**Quando usar feature flag vs. rollback completo:**

| Cenário | Estratégia | Justificativa |
|---|---|---|
| Bug isolado no Módulo Cedente sem impacto financeiro | Feature flag | Permite rollback granular sem afetar outros módulos |
| Escrow falhando (impacto financeiro) | Rollback completo do backend | Escrow é compartilhado — rollback total é mais seguro |
| Violação de isolamento LGPD | Feature flag imediato + investigação | Isola o problema enquanto equipe investiga |
| Performance degradada em todas as rotas | Rollback completo do backend | Problema não é isolado no módulo |

### 7.3 Procedimento de Rollback — Backend (Railway)

```bash
# 1. Confirmar que rollback é necessário (verificar gatilhos da seção 7.1)

# 2. Opção A: Feature flag (rollback suave — preferível para bugs isolados)
# Via Railway environment variable:
# CEDENTE_MODULE_ENABLED=false → Railway redeploy automático

# 3. Opção B: Rollback de versão (quando feature flag não é suficiente)
railway rollback --service backend-prod --environment production
# Troca de tráfego em ~30-60 segundos

# 4. Verificar que versão anterior está ativa
curl https://api.repasse-seguro.com.br/health
# Verificar campo "version" — deve ser versão anterior

# 5. Verificar Sentry — confirmar que spike de erros cessou
# Aguardar 2 minutos e verificar taxa de erros no Railway Metrics

# 6. Notificar canal #incidentes
# "[ROLLBACK] Módulo Cedente v1.0.0 revertida para [versão anterior].
# Motivo: [causa]. [hora] BRT. On-call: @[nome]"

# 7. Abrir post-mortem imediato (template em D26)
# 8. Criar hotfix para o problema identificado
```

### 7.4 Procedimento de Rollback — Frontend (Vercel)

```bash
# Via Vercel CLI:
vercel rollback [previous-deployment-url]
# Ou via Vercel Dashboard → Deployments → Selecionar versão anterior → Redeploy
```

### 7.5 Comunicação de Rollback aos Usuários

**Template de comunicação — degradação:**

```
Prezado usuário do Repasse Seguro,

Identificamos uma instabilidade temporária no Módulo Cedente e realizamos uma
intervenção preventiva para garantir a segurança dos seus dados.

O que foi afetado: [cadastro de imóveis / propostas / assinaturas — especificar]
Status: Em manutenção
Previsão de retorno: [hora] BRT

Seus dados estão seguros. Rascunhos salvos não foram perdidos.
Atualizações em: status.repasse-seguro.com.br

Equipe Repasse Seguro
```

**Template de comunicação — normalização:**

```
Prezado usuário do Repasse Seguro,

O Módulo Cedente foi restaurado e está funcionando normalmente.

Retorno ao normal: [hora] BRT
Duração da manutenção: [X] minutos

Caso encontre qualquer inconsistência, entre em contato com nosso suporte.

Equipe Repasse Seguro
```

**Canal de comunicação:** e-mail transacional (Resend) + notificação in-app + push notification mobile.

---

## 8. Incidentes por Categoria

### Backend (NestJS/Railway)

| Sintoma | Triagem | Ação inicial |
|---|---|---|
| 502/503 em todas as rotas do Cedente | Deploy recente? Instância unhealthy? | Rollback imediato se deploy < 30min; verificar logs Railway |
| Erros 500 em endpoint específico (ex: `/cedente/propostas`) | Stack trace no Sentry; migration recente? | Corrigir via hotfix; feature flag se necessário |
| Timeout no simulador de cenários | Cálculo de comissão em loop? Redis lento? | Verificar p95 do endpoint; verificar Redis |
| Wizard de cadastro travado em Etapa 3 | Timer de 10 segundos em loop? Frontend? | Verificar logs do frontend; Sentry Error Boundary |

### Integrações ZapSign

| Sintoma | Triagem | Ação inicial | Contingência |
|---|---|---|---|
| Webhook de assinatura não processado | Verificar HMAC secret; verificar URL no painel ZapSign | Reprocessar webhook manualmente via Admin | Admin registra assinatura manualmente com evidência documental |
| ZapSign inacessível (iframe não carrega em 10s) | Status page ZapSign | Notificar usuários; timeout de iframe já previsto em RN-047 | Botão "Abrir em nova aba" disponível por design |
| Envelope de assinatura não gerado após aceite | Falha no `ZapSignService.createEnvelope()` | Verificar Sentry; criar envelope manualmente via Admin | Admin notifica Cedente por e-mail com link direto |

### Integrações Escrow

| Sintoma | Triagem | Ação inicial | Contingência |
|---|---|---|---|
| Webhook de depósito não processado | Verificar credenciais Escrow; verificar logs do `EscrowWebhookHandler` | Contatar suporte Escrow; verificar DLQ | Admin confirma depósito manualmente (ADR obrigatório) |
| Distribuição automática após 15 dias não executada | Worker de distribuição parado? Redis TTL errado? | Verificar worker no Railway; verificar Redis TTL | Admin executa distribuição manualmente via painel financeiro |
| Estorno de desistência falhando | Escrow com instabilidade? Saldo insuficiente na conta de reserva? | Contatar suporte Escrow; pausar novas desistências | Informar usuários do atraso; processar via suporte |

### Guardião do Retorno

| Sintoma | Triagem | Ação inicial | Contingência |
|---|---|---|---|
| Guardião não responde (SSE timeout) | OpenAI com instabilidade? LangGraph.js em loop? | Verificar Langfuse; verificar status OpenAI | Mensagem automática: "O Guardião está temporariamente indisponível. Tente novamente em instantes." |
| Respostas com conteúdo inadequado (FOMO, manipulação) | `OutputFilterService` falhando? | Verificar logs do filtro; desabilitar Guardião via feature flag | Desabilitar Guardião temporariamente; usuário vê mensagem de manutenção |
| Taxa de escalamentos > 20% | Perguntas fora do escopo? Contexto insuficiente? | Revisar prompts e contexto do LangGraph | Normalizar threshold temporariamente; investigar padrão de consultas |

### Upload de Documentos (Dossiê)

| Sintoma | Triagem | Ação inicial | Contingência |
|---|---|---|---|
| Upload falhando para todos os documentos | Supabase Storage indisponível? Signed URL expirada? | Verificar Supabase Storage status; verificar geração de signed URL | Informar usuários; upload retorna mensagem de "serviço temporariamente indisponível" |
| MIME type validation falhando (arquivos válidos rejeitados) | Bug na validação de MIME? | Verificar `MimeValidationService`; rollback se necessário | Habilitar feature flag de bypass temporário (somente extensão) — monitorar com alerta |

### Autenticação e Sessão

| Sintoma | Triagem | Ação inicial |
|---|---|---|
| Login retornando 500 | Supabase Auth down? JWT_SECRET correto no Railway? | Verificar status Supabase; verificar logs AuthService |
| Refresh token inválido para todos os Cedentes | JWT_SECRET alterado acidentalmente? | Verificar secret no Railway; Cedentes precisarão fazer login novamente |
| Lockout em massa (muitos Cedentes bloqueados) | IP malicioso? Brute force legítimo? | Verificar Redis `rs:auth:attempts`; avaliar liberação manual no Admin |

---

## 9. Backup e Restore

### 9.1 Frequência e Armazenamento

| Tipo | Frequência | Retenção | Armazenamento |
|---|---|---|---|
| Daily backup (Supabase Pro) | Diário 00:00 UTC | 7 dias | Supabase managed |
| PITR (Point-in-Time Recovery) | Contínuo | 7 dias | Supabase managed |
| Backup de documentos do dossiê (Supabase Storage) | Contínuo (versionamento ativo) | Permanente | Supabase Storage |
| Export manual pré-deploy (pré-migration) | Antes de cada migration crítica | 30 dias | [DADO PENDENTE — bucket S3/R2 para backup manual] |

### 9.2 Restauração de Emergência (PITR)

```sql
-- Exemplo: restaurar banco para estado 2 horas antes de migration problemática
-- Executar no Supabase Dashboard → Database → Backups → PITR
-- Selecionar timestamp: [data/hora desejada]
-- ATENÇÃO: PITR restaura o banco completo — não é seletivo por tabela
```

**Quando usar PITR:**
- Migration aplicada com bug que corrompeu dados de casos ou propostas.
- Deleção acidental de registros de Cedentes.
- Qualquer situação onde o estado do banco precisa ser revertido por razões de integridade.

---

## 10. Métricas de Sucesso — 30 Dias Pós-Launch

### 10.1 Métricas de Produto

| Métrica | Meta 30 dias | Fonte | Observação |
|---|---|---|---|
| Cedentes com cadastro concluído (Etapa 5 do wizard) | [DADO PENDENTE — meta do produto] | PostHog — `caso_cadastrado` | Meta definida com equipe de produto |
| Taxa de conclusão do wizard (Etapa 1 → Etapa 5) | > 35% | PostHog — funil | Benchmark inicial — revisar após 30 dias |
| Taxa de dossiê completo (todos os 6 documentos enviados) | > 60% dos casos cadastrados | Supabase query | Indica qualidade do onboarding |
| Distribuição de escolha de cenário | Cenários B e C dominantes | PostHog — `cenario_selecionado` | Cenário D indica imóvel premium; A indica urgência |
| Taxa de aceite de proposta | > 25% das propostas apresentadas | PostHog — `proposta_aceita` | Benchmark do produto |
| Tempo médio do wizard (Etapa 1 → Etapa 5) | < 20 minutos | PostHog — duração da sessão no wizard | Indica fricção no processo |
| Adoção do Guardião do Retorno | > 40% dos Cedentes ativos consultam ao menos 1x | PostHog — `guardiao_consulta` | Indica valor percebido da IA |

### 10.2 Métricas de Qualidade Técnica

| Métrica | Meta 30 dias | Fonte |
|---|---|---|
| Taxa de erro global (5xx) | < 0.5% | Railway Metrics + Sentry |
| P95 de latência — endpoints críticos | < 200ms | k6 (weekly load test em staging) |
| P99 de latência | < 500ms | Railway Metrics |
| Disponibilidade (uptime) | > 99.5% | Railway Metrics |
| Taxa de webhooks ZapSign processados com sucesso | > 99% | Sentry + DLQ |
| Taxa de webhooks Escrow processados com sucesso | > 99% | Sentry + DLQ |
| Taxa de escalamentos do Guardião | < 20% | Langfuse |
| Latência p95 do Guardião (first token) | < 3s | Langfuse |

### 10.3 Revisão 30 Dias

**Agenda da reunião de revisão (30 dias pós-launch):**

1. Apresentação das métricas acima — Tech Lead + Produto (15 min).
2. Análise de funil: onde os Cedentes abandonam o cadastro (10 min).
3. Análise de suporte: tickets mais frequentes → prioridades de UX para próximo sprint (10 min).
4. Análise do Guardião: consultas mais frequentes → gaps de informação na plataforma (10 min).
5. Plano de melhorias para Sprint 2 (15 min).

---

## 11. Responsáveis

| Papel | Responsabilidade | Contato |
|---|---|---|
| Tech Lead | Decisão Go/No-Go, aprovação de rollback, gestão de incidentes P0 | [DADO PENDENTE — nome e contato] |
| On-call técnico | Execução de rollback, monitoramento pós-launch, resposta a alertas | [DADO PENDENTE — nome e contato] |
| Responsável de produto | Comunicação com usuários, aprovação de mensagens de manutenção | [DADO PENDENTE — nome e contato] |
| Suporte ZapSign | Incidentes de assinatura eletrônica | [DADO PENDENTE — confirmar em T-7] |
| Suporte Escrow | Incidentes de conta garantia e distribuição | [DADO PENDENTE — confirmar em T-7] |
| Suporte Supabase | Incidentes de banco e storage | https://supabase.com/support |
| Suporte Railway | Incidentes de infraestrutura backend | https://railway.app/help |

---

## 12. Decisão Go/No-Go — Formulário Final

> Preencher no T-0, antes do merge do PR de release. Tech Lead assina a decisão.

```
DATA: ________________
HORA: ________________ BRT
VERSÃO: v1.0.0-cedente

CRITÉRIOS GO (marcar todos como GO ou NO-GO):

[ ] GO / [ ] NO-GO — E2E P0 verde em staging (fluxos E2E-001 a E2E-006)
[ ] GO / [ ] NO-GO — Smoke tests de staging passando
[ ] GO / [ ] NO-GO — ZapSign sandbox operacional (último teste: ________)
[ ] GO / [ ] NO-GO — Escrow sandbox operacional (último teste: ________)
[ ] GO / [ ] NO-GO — Taxa de erro em staging < 1%
[ ] GO / [ ] NO-GO — P95 < 200ms em endpoints críticos
[ ] GO / [ ] NO-GO — Guardião do Retorno < 20% de escalamentos
[ ] GO / [ ] NO-GO — Isolamento LGPD validado (SEC-001 a SEC-005)
[ ] GO / [ ] NO-GO — RLS em todas as tabelas de Cedentes validado
[ ] GO / [ ] NO-GO — D28 (Checklist de Qualidade) aprovado
[ ] GO / [ ] NO-GO — Secrets de produção configurados
[ ] GO / [ ] NO-GO — Dados de teste removidos de staging

DECISÃO FINAL: [ ] GO  /  [ ] NO-GO

Se NO-GO — motivo: _________________________________
Próxima janela de go-live: ________________

Assinatura Tech Lead: ________________
```
