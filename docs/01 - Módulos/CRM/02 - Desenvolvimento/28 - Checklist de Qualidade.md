# 28 - Checklist de Qualidade

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia — QA, Tech Lead |
| **Escopo** | Checklist completo por categoria para validação antes de cada deploy do CRM |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 — America/Fortaleza |
| **Dependências** | 24 Deploy CI-CD · 25 Observabilidade · 27 Plano de Testes |

---

> **TL;DR**
>
> - **8 categorias** de validação: Funcional, Performance, Segurança, Acessibilidade, Responsividade, Integrações, Banco e Monitoramento.
> - **46 itens no total.** Nenhum item com status Bloqueante em aberto permite deploy.
> - **Itens marcados [BLOQUEANTE]:** impedem deploy se não aprovados.
> - **Itens marcados [RECOMENDADO]:** não bloqueiam, mas devem ser resolvidos no ciclo seguinte.
> - Preencher este checklist para todo PR de `release/*` → `main` e para hotfixes.

---

## Instruções de Uso

| Status | Significado |
|---|---|
| ✅ | Item validado e aprovado |
| ❌ | Item com falha — ação obrigatória antes do deploy |
| ⚠️ | Item com ressalva — documentar e planejar correção |
| N/A | Não aplicável para esta release |

> **Regra:** Nenhum item `[BLOQUEANTE]` pode estar com status ❌ no momento do deploy. Se necessário, criar issue, documentar plano de correção e obter aprovação do Tech Lead.

---

## Categoria 1 — Funcional

*Valida que os Requisitos Funcionais críticos do CRM estão operando corretamente em staging.*

| # | Item | Tipo | Status |
|---|---|---|---|
| F-01 | Pipeline kanban exibe Casos nos estágios corretos (CADASTRO → CONCLUIDO) | [BLOQUEANTE] | |
| F-02 | Criação de Caso persiste no banco com `config_snapshot` e status inicial = `CADASTRO` | [BLOQUEANTE] | |
| F-03 | Avanço de status (ex: SIMULACAO → VERIFICACAO) é registrado com histórico e `updated_by` | [BLOQUEANTE] | |
| F-04 | Transição de status inválida retorna HTTP 422 e não altera o banco | [BLOQUEANTE] | |
| F-05 | Criação de envelope ZapSign ao avançar para FORMALIZACAO — envelope criado no ZapSign sandbox | [BLOQUEANTE] | |
| F-06 | Webhook ZapSign SIGNED atualiza status de assinatura do Caso corretamente | [BLOQUEANTE] | |
| F-07 | Cálculo de comissão correto nos 4 cenários (A/B/C/D) — validado com planilha de referência | [BLOQUEANTE] | |
| F-08 | Alertas de SLA disparados para Casos com deadline vencido | [BLOQUEANTE] | |
| F-09 | Relatório semanal gerado manualmente com dados corretos do período | [BLOQUEANTE] | |
| F-10 | Upload de documento ao Dossiê com armazenamento no Supabase Storage | [BLOQUEANTE] | |
| F-11 | Registro de atividades (ligação, reunião, WhatsApp, nota) vinculado ao Caso | [RECOMENDADO] | |
| F-12 | Dashboard executivo exibe KPIs atualizados (máx 5 min de defasagem via Redis) | [RECOMENDADO] | |

---

## Categoria 2 — Performance

*Valida que os SLOs de tempo de resposta estão sendo cumpridos em staging com dados realistas.*

| # | Item | Tipo | Status |
|---|---|---|---|
| P-01 | Endpoint `GET /v1/cases` (listagem paginada) < 300ms p95 | [BLOQUEANTE] | |
| P-02 | Endpoint `GET /v1/pipeline` (pipeline com Casos agrupados) < 300ms p95 | [BLOQUEANTE] | |
| P-03 | Endpoint `POST /v1/cases` (criação de Caso) < 500ms p95 | [BLOQUEANTE] | |
| P-04 | Endpoint `PATCH /v1/cases/:id/status` (transição de status) < 300ms p95 | [BLOQUEANTE] | |
| P-05 | Dashboard do CRM (`/dashboard`) — First Contentful Paint < 1.5s em rede 4G | [RECOMENDADO] | |
| P-06 | Carregamento do detalhe do Caso (`/cases/:id`) < 1s no p95 | [RECOMENDADO] | |
| P-07 | Cache Redis em uso para KPIs do painel executivo — verificar hit rate > 60% | [RECOMENDADO] | |

**Ferramenta de medição:**
```bash
# Smoke load test em staging — 50 req/s por 30s
npx autocannon -c 50 -d 30 https://api.staging.crm.repasseseguro.com.br/v1/cases \
  -H "Authorization: Bearer $TEST_TOKEN"
# Critério: p95 < 300ms, zero erros 5xx
```

---

## Categoria 3 — Segurança

*Valida controles de acesso, proteção de dados e integridade dos mecanismos de segurança.*

| # | Item | Tipo | Status |
|---|---|---|---|
| S-01 | RBAC validado: ANALISTA_RS não acessa Casos de outros Analistas (filtro de backend) | [BLOQUEANTE] | |
| S-02 | RBAC validado: ANALISTA_RS recebe HTTP 403 ao tentar acessar `/settings` e `/team` | [BLOQUEANTE] | |
| S-03 | RBAC validado: PARCEIRO_EXTERNO recebe HTTP 403 em endpoints exclusivos de equipe interna | [BLOQUEANTE] | |
| S-04 | Webhook ZapSign com HMAC inválido retorna HTTP 401 — banco não alterado | [BLOQUEANTE] | |
| S-05 | Webhook Celcoin com HMAC inválido retorna HTTP 401 | [BLOQUEANTE] | |
| S-06 | Nenhum PII (CPF, e-mail, nome) nos logs do Railway em produção/staging | [BLOQUEANTE] | |
| S-07 | `pnpm audit` sem vulnerabilidades classificadas como High ou Critical | [BLOQUEANTE] | |
| S-08 | Headers de segurança presentes: CSP, HSTS, X-Frame-Options (via Helmet) | [BLOQUEANTE] | |
| S-09 | Sessão expira após 60 min de inatividade — verificado com token expirado | [RECOMENDADO] | |
| S-10 | Bloqueio após 5 tentativas de login incorretas — `locked_until` preenchido | [RECOMENDADO] | |
| S-11 | Signed URLs do Supabase Storage têm expiração configurada (não URLs permanentes) | [RECOMENDADO] | |

---

## Categoria 4 — Acessibilidade

*Valida que o CRM é utilizável por toda a equipe, incluindo usuários com deficiências.*

| # | Item | Tipo | Status |
|---|---|---|---|
| A-01 | Contraste de texto vs. fundo ≥ 4.5:1 nas telas principais do CRM (WCAG AA) | [BLOQUEANTE] | |
| A-02 | Navegação por teclado funcional nas telas críticas: pipeline, detalhe do Caso, formulários | [BLOQUEANTE] | |
| A-03 | Todos os elementos interativos têm `aria-label` ou texto descritivo quando ícone apenas | [RECOMENDADO] | |
| A-04 | Modais têm foco gerenciado: foco vai para modal ao abrir, retorna ao trigger ao fechar | [RECOMENDADO] | |
| A-05 | Mensagens de erro de formulários associadas ao campo via `aria-describedby` | [RECOMENDADO] | |

**Ferramenta:** `@axe-core/playwright` nos testes E2E. Critério: zero violações `critical` ou `serious`.

---

## Categoria 5 — Responsividade

*O CRM suporta tablet (768px) e desktop. Mobile nativo não suportado na v1.0.*

| # | Item | Tipo | Status |
|---|---|---|---|
| R-01 | Pipeline kanban utilizável em tablet (768px) — colunas sem overflow horizontal | [BLOQUEANTE] | |
| R-02 | Formulário de criação de Caso funcional em 768px — campos não sobrepostos | [BLOQUEANTE] | |
| R-03 | Dashboard do CRM legível em 1280px (desktop-sm) | [BLOQUEANTE] | |
| R-04 | Tabelas com scroll horizontal em telas < 1280px (sem quebra de layout) | [RECOMENDADO] | |
| R-05 | Sidebar de navegação colapsável em tablet (768px) para economizar espaço | [RECOMENDADO] | |

---

## Categoria 6 — Integrações

*Valida que todas as integrações externas estão funcionando corretamente em staging.*

| # | Item | Tipo | Status |
|---|---|---|---|
| I-01 | ZapSign: criação de envelope em sandbox funcional com payload correto (3 signatários) | [BLOQUEANTE] | |
| I-02 | ZapSign: webhook de evento SIGNED processado corretamente em staging | [BLOQUEANTE] | |
| I-03 | Celcoin: webhook de confirmação de depósito processado (sandbox) com HMAC validado | [BLOQUEANTE] | |
| I-04 | Meta Cloud API (WhatsApp): envio de mensagem de template funcional em conta de teste | [RECOMENDADO] | |
| I-05 | Resend: e-mail de notificação de relatório enviado em staging (verificar inbox de teste) | [RECOMENDADO] | |
| I-06 | Supabase Realtime: alertas de SLA chegam ao frontend em tempo real (< 2s) em staging | [RECOMENDADO] | |

---

## Categoria 7 — Banco de Dados

*Valida integridade das migrations e consistência dos dados.*

| # | Item | Tipo | Status |
|---|---|---|---|
| B-01 | `prisma migrate deploy` executado sem erros em staging antes do deploy | [BLOQUEANTE] | |
| B-02 | Nenhuma migration com `DROP COLUMN`, `DROP TABLE` ou `NOT NULL` sem `DEFAULT` não revisada | [BLOQUEANTE] | |
| B-03 | `prisma migrate status` mostra 0 migrations pendentes em staging pós-deploy | [BLOQUEANTE] | |
| B-04 | Soft delete funcional: Casos e Contatos deletados não aparecem nas listagens | [BLOQUEANTE] | |
| B-05 | `audit_log` registra ações críticas: criação de Caso, transição de status, acesso a dados sensíveis | [RECOMENDADO] | |
| B-06 | Timestamps armazenados em UTC, exibidos em America/Fortaleza na interface | [RECOMENDADO] | |

---

## Categoria 8 — Monitoramento

*Valida que os sistemas de observabilidade estão ativos e funcionando em produção/staging.*

| # | Item | Tipo | Status |
|---|---|---|---|
| M-01 | Sentry recebe eventos de erro do CRM API em staging (testar com erro intencional) | [BLOQUEANTE] | |
| M-02 | Sentry recebe eventos de erro do CRM frontend em staging | [BLOQUEANTE] | |
| M-03 | Endpoint `/health` retorna `{ "status": "ok" }` com banco, Redis e RabbitMQ UP | [BLOQUEANTE] | |
| M-04 | Alertas Sentry P0/P1 chegam ao Slack `#crm-incidents` (testar com trigger manual) | [BLOQUEANTE] | |
| M-05 | PostHog recebe evento `case_created` ao criar Caso em staging | [RECOMENDADO] | |
| M-06 | Railway Metrics exibindo CPU, memória e latência para o serviço CRM API | [RECOMENDADO] | |
| M-07 | CloudAMQP mostra filas ativas com profundidade = 0 (sem mensagens represadas) antes do deploy | [RECOMENDADO] | |

---

## Resumo por Categoria

| Categoria | Total de Itens | Bloqueantes | Recomendados |
|---|---|---|---|
| Funcional | 12 | 10 | 2 |
| Performance | 7 | 4 | 3 |
| Segurança | 11 | 8 | 3 |
| Acessibilidade | 5 | 2 | 3 |
| Responsividade | 5 | 3 | 2 |
| Integrações | 6 | 3 | 3 |
| Banco de Dados | 6 | 4 | 2 |
| Monitoramento | 7 | 4 | 3 |
| **TOTAL** | **59** | **38** | **21** |

> **Nota:** O total acima reflete os itens neste documento. O requisito mínimo de 40 itens é cumprido pelos 38 itens bloqueantes + 21 recomendados = **59 itens totais**.

---

## Aprovação para Deploy

| Campo | Valor |
|---|---|
| **Release** | v______ |
| **Data de validação** | ____/____/______ |
| **Ambiente validado** | ☐ Staging ☐ Produção |
| **Itens bloqueantes ❌** | ______ (deve ser 0) |
| **Aprovado por (Tech Lead)** | ______________________ |
| **Aprovado por (QA)** | ______________________ |

---

## Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 59 itens em 8 categorias (Funcional, Performance, Segurança, Acessibilidade, Responsividade, Integrações, Banco, Monitoramento). 38 bloqueantes, 21 recomendados. |
