# Índice de Sprints — AI-Dani-Cessionário

| **Produto** | AI-Dani-Cessionário |
|---|---|
| **Pipeline** | ShiftLabs v2.3 — Fase 3 |
| **Gerado em** | 24/03/2026 |
| **Total de Sprints** | 10 (S1–S9 construção + S10 Sprint Final) |
| **Total de REQs cobertos** | 157/157 (100%) |

---

## Arquivos Gerados

| Sprint | Arquivo | Módulo / Foco | Docs Mapeados | Qtd Itens | Status |
|---|---|---|---|---|---|
| — | `registro-mestre.md` | Registro Mestre de Requisitos (157 REQs) | D01–D29 | 157 REQs | ✅ Gerado |
| S1 | `s1-fundacao.md` | Fundação — Infraestrutura, Banco de Dados, CI/CD base | D02, D12, D13, D22, D23 | 52 | ✅ Gerado |
| S2 | `s2-auth.md` | Autenticação OAuth + JWT + Guards | D18, D02 | 42 | ✅ Gerado |
| S3 | `s3-agente-core.md` | Agente IA Core — Chat, Streaming, Memória, RAG, UI | D01, D06, D07, D08, D09, D11, D17, D19 | 59 | ✅ Gerado |
| S4 | `s4-calculadora-analise.md` | Calculadora Determinística + Análise Individual | D01, D04, D06, D09, D16, D19 | 46 | ✅ Gerado |
| S5 | `s5-comparacao-simulacao.md` | Comparação de Oportunidades + Simulações | D01, D06, D09, D16 | 46 | ✅ Gerado |
| S6 | `s6-notificacoes-suporte.md` | Notificações, Templates, Alertas, Suporte Operacional | D01, D06, D12, D16, D21 | 48 | ✅ Gerado |
| S7 | `s7-whatsapp.md` | WhatsApp Module — Vinculação, OTP, Webhook, LGPD | D01, D06, D09, D16, D17, D21 | 47 | ✅ Gerado |
| S8 | `s8-qualidade.md` | Observabilidade, Testes, CI/CD, Checklist D28 | D25, D27, D28 | 54 | ✅ Gerado |
| S9 | `s9-go-live.md` | Go-Live — Infra Produção, Rollback, Launch Day, Monitoramento | D24, D26, D29 | 42 | ✅ Gerado |
| S10 | `s10-auditoria.md` | Sprint Final — Matriz de Cobertura 157 REQs × S1–S9 | Registro Mestre + D01–D29 | 157 REQs auditados | ✅ Gerado |
| — | `auditoria-a03.md` | Auditoria A03 — GATE Fase 3 | A03 Prompt | — | ✅ Gerado |

---

## Sumário por Sprint

### S1 — Fundação (52 itens)

**Escopo:** Monorepo Turborepo + pnpm workspaces, setup Node.js/NestJS/TypeScript, Supabase PostgreSQL + pgvector, Redis, RabbitMQ, todas as migrations (7 tabelas), índices obrigatórios, job de cleanup, variáveis de ambiente (28), branching, Conventional Commits, `pnpm health`.

**REQs principais:** REQ-040 a REQ-070, REQ-071 (base URL), REQ-090 (paginação), REQ-141 (credenciais), REQ-153 a REQ-157.

---

### S2 — Auth (42 itens)

**Escopo:** JWT (access 15min in-memory + refresh 7d httpOnly cookie `dani_refresh_token`), Redis session (TTL 604800s), Redis blacklist, 3 endpoints de auth, `CessionarioOwnerGuard`, roles `CESSIONARIO`/`ADMIN`, checklist S-01 a S-13, RBAC frontend `CESSIONARIO_AUTHENTICATED`.

**REQs principais:** REQ-091 a REQ-097, REQ-131.

---

### S3 — Agente Core (59 itens)

**Escopo:** `AgenteModule` completo — 5 tools com timeouts/retries, GPT-4o (temperature 0.3, max_tokens 2048), SSE streaming, system prompt versionado, memória curto/longo prazo, RAG pgvector namespace, chat sem/com contexto (T-DC-001 a T-DC-005), Widget Top 3 (T-DC-009), FAB Global, Rate Limit UI (T-DC-011), 4 estados por componente, 8 componentes críticos, strings normativas, desligamento automático >30%.

**REQs principais:** REQ-001 a REQ-014, REQ-027, REQ-030 a REQ-035, REQ-055 a REQ-056, REQ-060, REQ-098 a REQ-108, REQ-117 a REQ-127, REQ-129 a REQ-135.

---

### S4 — Calculadora e Análise (46 itens)

**Escopo:** `CalculadoraModule` isolado (ADR-003), fórmulas determinísticas Comissão Comprador (Δ > 0 → `20% × Δ`; Δ ≤ 0 → `20% × Valor Pago`), edge case Δ = 0, Custo Total Escrow, `POST /calculadora/calcular`, cache Redis TTL 300s, análise individual inline (T-DC-005), ROI 3 cenários.

**REQs principais:** REQ-015 a REQ-017, REQ-021 a REQ-023, REQ-029, REQ-078, REQ-105, REQ-121.

---

### S5 — Comparação e Simulação (46 itens)

**Escopo:** `OportunidadeService.compararOportunidades()` — máx 5 OPRs, desempate maior Δ, `is_melhor_opcao: true` na primeira, tabela comparativa (T-DC-006/RN-DC-015); `AgenteService.simularProposta()` (T-DC-007/RN-DC-016–017); `AgenteService.simularContraproposta()` com `ROIDeltaBadge` (T-DC-008/RN-DC-018); `analisarPortfolio()` status SUFICIENTE/INSUFICIENTE/EXCEDENTE (RN-DC-019); `simularVariacaoValorizacao()` (RN-DC-020).

**REQs principais:** REQ-018 a REQ-026, REQ-122 a REQ-124.

---

### S6 — Notificações e Suporte (48 itens)

**Escopo:** `NotificacaoService.publicar()` — NUNCA síncrono na request principal; 10 templates TPLF-001–010 com gatilhos/variáveis/opt-out; `NotificationProcessor` consumer com retry 3x (5s/15s/30s); `Dead Letter Handler`; `AlertaController` 5 endpoints; badge FAB polling 30s; prazos operacionais exatos (Escrow 10d/+5d/15d; ZapSign 5d/D+2/D+4/D+5; KYC ≤30min/≤2d); preferências opt-out LGPD.

**REQs principais:** REQ-028, REQ-065, REQ-079 a REQ-081, REQ-109 a REQ-116.

---

### S7 — WhatsApp (47 itens)

**Escopo:** `dani_vinculacoes_whatsapp` schema completo; 4 Redis keys (OTP/rate/block/cooldown); `WhatsappService.iniciarVinculacao()`, `.verificarOtp()`, `.desvincular(motivo)`; `WebhookApiKeyGuard`; `WhatsappController` 5 endpoints; `OTPInput` 6 campos com auto-avanço; T-DC-012 3 etapas + `WhatsAppPhaseGuard`; PARAR = desvinculação síncrona imediata (LGPD); ADR-001 EvolutionAPI v2.x.

**REQs principais:** REQ-036 a REQ-039, REQ-059, REQ-066, REQ-082 a REQ-086, REQ-114 a REQ-115, REQ-128, REQ-139.

---

### S8 — Qualidade (54 itens)

**Escopo:** 12 eventos críticos Pino; Langfuse `userId=sha256`; redact 5 campos; Sentry P0 alerts; retenção de logs (7d/30d/90d); Vitest (55% unitário / 25% integração); 10 fluxos E2E obrigatórios (E2E-001–010) com Playwright; Pact contratos C-01/C-02/C-03 com valores exatos; `ci-pr.yml` + `ci-staging.yml` + `release.yml`; coverage gate ≥80%; GitLeaks; `pnpm audit`; axe-core zero violations; bloqueantes D28.

**REQs principais:** REQ-107, REQ-140, REQ-142 a REQ-144, REQ-146, REQ-148 a REQ-150.

---

### S9 — Go-Live (42 itens)

**Escopo:** Railway + Vercel produção; `release.yml` 8 stages + 2 reviewers; `GET /health` com checks banco/redis/rabbitmq; rollback commands (`railway rollback --service dani-api` + `vercel rollback`); checklists T-7/T-3/T-1; go/no-go 08h30; janela 09h–12h; smoke tests 5 checks; T+15min/T+1h/T+24h; on-call 48h; backups PITR/snapshot/RDB; ⚠️ AMBÍGUO REQ-152.

**REQs principais:** REQ-145, REQ-147, REQ-151 a REQ-152.

---

### S10 — Sprint Final: Auditoria (157 REQs auditados)

**Escopo:** Matriz de cobertura completa 157 × S1–S9; 3 itens ⚠️ AMBÍGUO documentados com ação para Fase 4; 10 ADRs/constraints verificados; etapa adversarial (0 gaps); 7 checks SF-01 a SF-07; ações obrigatórias Fase 4.

**Resultado:** 100% cobertura ✅ — Zero P0/P1 pendentes ✅ — Fase 4 liberada ✅

---

## Rastreabilidade: Docs de Desenvolvimento × Sprints

| Doc | Título | Sprints que usam |
|---|---|---|
| D01 | Regras de Negócio | S3, S4, S5, S6, S7 |
| D02 | Stack Tecnológico | S1, S3, S7 |
| D03 | Brand & Design Tokens | S3 |
| D04 | (Contribuição/Produto) | S4 |
| D06 | Contratos de Tela (T-DC-001–012) | S3, S4, S5, S6, S7 |
| D07 | (UX Flows) | S3 |
| D08 | Strings Normativas | S3 |
| D09 | Contratos de UI por Tela | S3, S4, S5, S7 |
| D11 | Mobile/Plataformas | S1, S3 |
| D12 | Modelo de Dados — Schema | S1, S6, S7 |
| D13 | Modelo de Dados — Migrations | S1 |
| D16 | Documentação de API | S2, S3, S4, S5, S6, S7 |
| D17 | Integrações Externas | S1, S3, S7 |
| D18 | Autenticação OAuth | S2 |
| D19 | Agente de IA | S3, S4 |
| D21 | Notificações, Templates | S6, S7 |
| D22 | Setup de Ambiente | S1 |
| D23 | Git & Contribuição | S1 |
| D24 | CI/CD | S9 |
| D25 | Observabilidade e Logs | S8 |
| D26 | Operações/Ambientes | S9 |
| D27 | Plano de Testes | S8 |
| D28 | Checklist de Qualidade | S8 |
| D29 | Go-Live Playbook | S9 |

---

## Itens ⚠️ AMBÍGUO — Consolidado

| REQ | Sprints | Descrição | Ação na Fase 4 |
|---|---|---|---|
| REQ-089 | S9, S10 | D29 paths `/agente/chat`/`/calculadora/simular` divergem de D16 | Adotar D16 como normativo; atualizar smoke tests |
| REQ-126 | S3, S4, S5, S10 | `--agent-fallback: #0069A8` sem confirmação em D03 | Confirmar com Design System antes de merge |
| REQ-152 | S9, S10 | Mesma divergência D29 vs D16 nos smoke tests | Adotar D16; atualizar `smoke-tests.sh` |
