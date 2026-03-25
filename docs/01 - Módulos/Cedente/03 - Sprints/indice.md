# Índice de Sprints — Módulo Cedente
## Plataforma Repasse Seguro · Pipeline ShiftLabs v2.3

| Campo | Valor |
|---|---|
| **Módulo** | Cedente |
| **Gerado por** | Pipeline ShiftLabs v2.3 |
| **Data** | 2026-03-24 |
| **Total de sprints** | 11 |
| **Total de REQs cobertos** | 220 |
| **Cobertura** | 100% |

---

## Visão Geral

| Sprint | Arquivo | Docs Fonte | REQs Cobertos | Qtd Itens | Status |
|--------|---------|------------|---------------|-----------|--------|
| S1 — Fundação | `s1-fundacao.md` | D12, D13, D14, D15, D16, D17, D20, D22, D24, D25 | REQ-133–180, REQ-194–201, REQ-209 (45 REQs) | ~50 | Pendente |
| S2 — Auth | `s2-auth.md` | D01.1, D01.4, D01.5, D18 | REQ-001–020, REQ-109–110, REQ-126, REQ-160–168, REQ-172, REQ-181–185 (30 REQs) | 29 seções | Pendente |
| S3 — Cadastro e Dashboard | `s3-cadastro-dashboard.md` | D01.1, D01.2 | REQ-021–043, REQ-062–063 (24 REQs) | 4 features | Pendente |
| S4 — Dossiê e Assinaturas | `s4-dossie-assinaturas.md` | D01.3, D01.5, D17, D21 | REQ-068–095, REQ-111–112, REQ-123–125, REQ-198–199 (30 REQs) | 4 features | Pendente |
| S5 — Propostas e Negociação | `s5-propostas-negociacao.md` | D01.2, D01.4 | REQ-044–061, REQ-096, REQ-121–122 (20 REQs) | 3 features | Pendente |
| S6 — Financeiro e Fechamento | `s6-financeiro-fechamento.md` | D01.2, D01.3, D01.4, D01.5 | REQ-036, REQ-064–067, REQ-087–093, REQ-104–108, REQ-113–120, REQ-127–128 (22 REQs) | 4 features | Pendente |
| S7 — Agente IA | `s7-agente-ia.md` | D01.4, D19 | REQ-097–103, REQ-186–193 (15 REQs) | 3 features | Pendente |
| S8 — Mobile | `s8-mobile.md` | D11, D18, D01.5 | REQ-132, REQ-185, REQ-216–220 (7 REQs) | 9 features | Pendente |
| S9 — Qualidade | `s9-qualidade.md` | D24, D25, D27, D28 | REQ-202–211 (9 REQs) | 5 features | Pendente |
| S10 — Go-Live | `s10-go-live.md` | D29, D24, D25, D28 | REQ-212–215 (4 REQs) | 7 features | Pendente |
| S11 — Sprint Final: Auditoria | `s11-auditoria.md` | TODOS D01.1–D29 | 220 REQs auditados | 15 seções | Pendente |

---

## Mapa de Dependências

```
S1 (Fundação — infra, banco, auth base)
  └── S2 (Auth — login, cadastro, tokens)
        └── S3 (Cadastro e Dashboard — wizard, simulador, dashboard)
              └── S4 (Dossiê e Assinaturas — docs, ZapSign, notificações)
                    ├── S5 (Propostas e Negociação — propostas, escalonamento)
                    │     └── S6 (Financeiro e Fechamento — Escrow, anuência, inadimplência)
                    │           └── S7 (Agente IA — Guardião do Retorno, RAG)
                    │                 └── S8 (Mobile — RN, Expo, câmera, biometria, push)
                    │                       └── S9 (Qualidade — CI/CD, testes, observabilidade)
                    │                             └── S10 (Go-Live — launch, rollback)
                    │                                   └── S11 (Auditoria — cobertura 100%)
                    └── (S7 pode iniciar em paralelo com S5 após S4)
```

---

## Distribuição de REQs por Sprint

| Sprint | REQs Primários | REQs Compartilhados | Total Únicos |
|--------|---------------|---------------------|-------------|
| S1 | 45 | REQ-150 (com S7), REQ-174 (com S4) | 45 |
| S2 | 28 | REQ-185 (com S8) | 30 |
| S3 | 22 | REQ-021 (com S5), REQ-062 (com S5/S6) | 24 |
| S4 | 30 | — | 30 |
| S5 | 20 | — | 20 |
| S6 | 22 | — | 22 |
| S7 | 15 | REQ-150 (com S1) | 15 |
| S8 | 6 | REQ-185 (com S2) | 7 |
| S9 | 9 | — | 9 |
| S10 | 4 | — | 4 |
| CROSS | 4 | — | 4 |
| **Total únicos** | — | — | **220** |

---

## Itens REVISÃO MANUAL Consolidados

| ID | Sprint | Descrição Resumida |
|----|--------|--------------------|
| PEND-02 | S2 | TTL ativação: 24h (D18) vs 48h (D01.1). Adotado 48h. |
| PEND-03 | S2 | Inatividade: 24h (RN-006) vs 30min (D18). Adotado 30min. |
| PEND-06 | S4 | Docs PF: 6 ou 7-8? Adotado 6 com TABELA_CONTRATO como 6º. |
| PEND-08 | S5 | Piso cenário não tem endpoint Admin em D16. Provisório: valor_pago. |
| PEND-10 | S6 | "Aguardando liberação" (caso) vs "EM_PERIODO_REVERSAO" (Escrow). |
| PEND-11 | S7 | D19: "5 tools + 1 escalation" = 6. Adotado 6. |
| PEND-12 | S8 | Detox câmera requer device físico. CI usa mock. |
| PEND-13 | S10 | Contatos suporte ZapSign/Escrow/Resend são [DADO PENDENTE] em D29. |
| DP-001 | S6/S10 | Parceiro Escrow TBD. Blocker para automação. |
| PEND-ENUM | S11 | StatusProposta "SEM_RESPOSTA" vs "EXPIRADA" — verificar D12. |
| PEND-KB | S11 | guardiao_knowledge_base: confirmar migration S7 separa de ai_sessions S1. |

---

## Gates de Qualidade

| Gate | Critério | Sprint |
|------|---------|--------|
| Cobertura financeira 100% | CommissionCedenteService + EscrowCedenteService | S9 |
| T-038 RLS isolation | Isolamento entre Cedentes obrigatório pré-release | S9 |
| E2E P0 verde | E2E-001 a E2E-007 todos passando | S9 |
| WCAG 2.1 AA | Lighthouse A11y ≥ 95; zero violações axe-core | S9 |
| Go/No-Go 12 critérios | Todos satisfeitos antes do merge em main | S10 |
| Smoke tests prod | 5 testes sem 5xx | S10 |
| Auditoria cobertura 100% | 220 REQs auditados | S11 |

---

## Arquivos de Suporte

| Arquivo | Propósito |
|---------|-----------|
| `registro-mestre.md` | 220 REQs com sprint atribuída — fonte da verdade |
| `s1-fundacao.md` a `s10-go-live.md` | Sprint checklists executáveis |
| `s11-auditoria.md` | Auditoria de cobertura final |
| `indice.md` | Este arquivo |
