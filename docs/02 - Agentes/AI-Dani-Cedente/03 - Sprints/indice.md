# Índice de Sprints — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| **Gerado por** | Pipeline de Sprints ShiftLabs v2.3 |
| **Data** | 2026-03-24 |
| **Produto** | AI-Dani-Cedente |
| **Total de sprints** | 11 (S1–S10 + Sprint Final S11) |
| **Total de REQs** | 187 (100% cobertos) |

---

## Tabela de Sprints

| Sprint | Nome | Tipo | Docs Consultados | Qtd Itens (aprox.) | REQs Cobertos | Status |
|---|---|---|---|---|---|---|
| S1 | Fundação | Infraestrutura (Template A) | D02, D12, D13, D14, D15, D16, D20 | ~55 | REQ-085 a REQ-100, REQ-116, REQ-119, REQ-124–REQ-127, REQ-130–REQ-155, REQ-163, REQ-166, REQ-183–REQ-184 | ⬜ Pendente |
| S2 | Auth e Segurança | Infraestrutura (Template A) | D01, D02, D16, D17, D18 | ~45 | REQ-010, REQ-018, REQ-022, REQ-081, REQ-084, REQ-095, REQ-110–REQ-113, REQ-118, REQ-156–REQ-157, REQ-164–REQ-165, REQ-167, REQ-169, REQ-174–REQ-176 | ⬜ Pendente |
| S3 | Agente Core | Módulo Fullstack (Template B) | D01, D02, D05.1, D16, D17, D19 | ~58 | REQ-017, REQ-019–REQ-026, REQ-079–REQ-080, REQ-082, REQ-096, REQ-101–REQ-102, REQ-105–REQ-108, REQ-114–REQ-115, REQ-117, REQ-128, REQ-158–REQ-162, REQ-168, REQ-177–REQ-182 | ⬜ Pendente |
| S4 | Oportunidade e Cenários | Módulo Fullstack (Template B) | D01, D05.1, D05.2, D05.3, D16 | ~52 | REQ-003–REQ-004, REQ-011–REQ-012, REQ-027–REQ-036, REQ-039–REQ-045 | ⬜ Pendente |
| S5 | Dossiê | Módulo Fullstack (Template B) | D01, D05.3, D16, D17 | ~40 | REQ-005, REQ-046–REQ-049, REQ-170 | ⬜ Pendente |
| S6 | Propostas e Negociação | Módulo Fullstack (Template B) | D01, D05.3, D16 | ~45 | REQ-037, REQ-050–REQ-061 | ⬜ Pendente |
| S7 | Escrow e Assinatura | Módulo Fullstack (Template B) | D01, D05.4, D16, D17 | ~48 | REQ-006–REQ-009, REQ-038, REQ-062–REQ-069, REQ-171 | ⬜ Pendente |
| S8 | Notificações e Suporte | Módulo Fullstack (Template B) | D01, D05.4, D16, D17, D21 | ~55 | REQ-070–REQ-073, REQ-083, REQ-097, REQ-185–REQ-187 | ⬜ Pendente |
| S9 | Observabilidade, Qualidade e Fallback | Módulo Fullstack (Template B) | D01, D02, D05.5, D17, D19 | ~58 | REQ-013, REQ-075–REQ-078, REQ-103–REQ-104, REQ-109, REQ-120–REQ-123, REQ-129, REQ-172–REQ-173 | ⬜ Pendente |
| S10 | Go-Live | Infraestrutura de Deploy (Template A) | D02, D24, D25, D26, D29 | ~40 | REQ-131 a REQ-147 (via D12/D13 backfill), plus deploy REQs de D24–D29 | ⬜ Pendente |
| S11 | Sprint Final — Auditoria | Auditoria (Sprint Final obrigatória) | D01–D29 (todos) | ~60 verificações | REQ-001 a REQ-187 (matriz de cobertura) | ⬜ Pendente |

---

## REQs Cross-Cutting (sem sprint dedicada)

| REQ | Descrição | Cobertura |
|---|---|---|
| REQ-001 | Cedente — definição | Via S2 (CedenteIsolationMiddleware) + S3 (system prompt) |
| REQ-002 | Cessionário — definição e isolamento | Via S2 (PII masking) + S3 (dados bloqueados) |
| REQ-014 | Nome exibido "Dani" | Via S3 (persona + chat widget) |
| REQ-015 | Nome interno AI-Dani-Cedente | Via S1 (estrutura de módulos) |
| REQ-016 | Persona da Dani — Guardiã do Retorno | Via S1 (system prompt template) + S3 (agent config) |
| REQ-074 | Prazos operacionais | Via S3 (base de conhecimento RAG) + S7/S8 (implementações) |

---

## Distribuição por Tipo de Sprint

| Tipo | Sprints | Qtd |
|---|---|---|
| Infraestrutura (Template A) | S1, S2, S10 | 3 |
| Módulo Fullstack (Template B) | S3, S4, S5, S6, S7, S8, S9 | 7 |
| Sprint Final (Auditoria) | S11 | 1 |

---

## Dependências entre Sprints

```
S1 (Fundação — banco + estrutura)
  └─ S2 (Auth — JWT + isolamento + rate limit)
       └─ S3 (Agente Core — LLM + RAG + SSE)
            ├─ S4 (Oportunidade — estado + cenários + Δ)
            │    └─ S5 (Dossiê — upload + validação + progresso)
            │         └─ S6 (Propostas — aceitação requer dossiê completo)
            │              └─ S7 (Escrow + ZapSign — depende de proposta aceita)
            └─ S8 (Notificações — depende de S3 SSE + S7 ZapSign régua)
                 └─ S9 (Observabilidade — Langfuse + PostHog + Fallback + Health)
                      └─ S10 (Go-Live — CI/CD + Railway + Runbooks + Smoke Test)
                           └─ S11 (Auditoria — verifica cobertura total S1–S10)
```

---

## Arquivos Gerados

| Arquivo | Descrição |
|---|---|
| `registro-mestre.md` | Fonte da verdade — 187 REQs com sprint assignments |
| `s1-fundacao.md` | Sprint 1 — banco, enums, infraestrutura base |
| `s2-auth.md` | Sprint 2 — autenticação, autorização, rate limit |
| `s3-agente-core.md` | Sprint 3 — agente IA, LLM, RAG, chat SSE |
| `s4-oportunidade-cenarios.md` | Sprint 4 — oportunidade, cenários, Δ, simulação |
| `s5-dossie.md` | Sprint 5 — dossiê, documentos, Supabase Storage |
| `s6-propostas-negociacao.md` | Sprint 6 — propostas, aceitação, contraproposta |
| `s7-escrow-assinatura.md` | Sprint 7 — escrow, ZapSign, régua de assinatura |
| `s8-notificacoes-suporte.md` | Sprint 8 — notificações, templates, CSAT |
| `s9-observabilidade-qualidade-fallback.md` | Sprint 9 — Langfuse, PostHog, fallback, health checks |
| `s10-go-live.md` | Sprint 10 — CI/CD, Railway, runbooks, smoke test |
| `s11-auditoria.md` | Sprint Final — auditoria de cobertura 187 REQs |
| `indice.md` | Este arquivo — índice consolidado |
