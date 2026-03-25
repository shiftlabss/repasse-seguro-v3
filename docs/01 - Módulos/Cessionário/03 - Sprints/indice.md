# Índice de Sprints — Módulo Cessionário

## Sprint Index · Módulo Cessionário · Plataforma Repasse Seguro

| Campo | Valor |
|---|---|
| **Módulo** | Cessionário · Plataforma Repasse Seguro |
| **Pipeline** | ShiftLabs v2.3 — Fase 3 |
| **Gerado em** | 2026-03-24 |
| **Total de Sprints** | 13 |
| **Total de Itens** | 521 |
| **Total de REQs** | 285 |
| **Cobertura** | 100% ✅ |

---

## Tabela Consolidada

| Sprint | Nome | Template | Docs Fonte | REQs Cobertos | Qtd Itens | Status |
|---|---|---|---|---|---|---|
| **S1** | Fundação | A — Infraestrutura | D02, D12, D13, D15, D20, D24, D25 | S1-001 a S1-037 (37 REQs) + CROSS | 55 | Pendente |
| **S2** | Auth e KYC | A — Infraestrutura | D01.1, D02, D16, D17 | S2-001 a S2-046 (46 REQs) | 57 | Pendente |
| **S3** | Dashboard e Perfil | B — Módulo Fullstack | D01.3, D16, D21 | S3-001 a S3-018 (18 REQs) | 46 | Pendente |
| **S4** | Marketplace e Propostas | B — Módulo Fullstack | D01.2, D16, D17, D21 | S4-001 a S4-034 (34 REQs) | 54 | Pendente |
| **S5** | Negociações | B — Módulo Fullstack | D01.2, D16, D17, D21 | S5-001 a S5-018 (18 REQs) | 49 | Pendente |
| **S6** | Escrow | B — Módulo Fullstack | D01.2, D16, D17, D21 | S6-001 a S6-014 (14 REQs) | 42 | Pendente |
| **S7** | Formalização | B — Módulo Fullstack | D01.2, D17, D16, D21 | S7-001 a S7-014 (14 REQs) | 43 | Pendente |
| **S8** | Fechamento e Financeiro | B — Módulo Fullstack | D01.2, D01.4, D16, D21 | S8-001 a S8-016 (16 REQs) | 46 | Pendente |
| **S9** | Assistente IA | B — Módulo Fullstack (Condicional) | D02, D16, D17, D25 | S9-001 a S9-018 (18 REQs) | 52 | Pendente |
| **S10** | Mobile | B — Módulo Fullstack (Condicional) | D02, D11, D16 | S10-001 a S10-003 (3 REQs) | 42 | Pendente |
| **S11** | Qualidade | Sprint de Qualidade | D25, D27, D28 | Transversais (todos os módulos) | 50 | Pendente |
| **S12** | Go-Live | Sprint de Lançamento | D24, D25, D26, D29 | Infraestrutura de produção | 44 | Pendente |
| **S13** | Auditoria (Sprint Final) | Sprint Final | Todos | 285/285 REQs | 41 | Pendente |

---

## Distribuição de REQs por Sprint

| Sprint | REQs Primários | REQs CROSS cobertos | Total |
|---|---|---|---|
| S1 — Fundação | 37 | 27 | 37 + CROSS |
| S2 — Auth e KYC | 46 | — | 46 |
| S3 — Dashboard e Perfil | 18 | — | 18 |
| S4 — Marketplace e Propostas | 34 | — | 34 |
| S5 — Negociações | 18 | — | 18 |
| S6 — Escrow | 14 | — | 14 |
| S7 — Formalização | 14 | — | 14 |
| S8 — Fechamento e Financeiro | 16 | — | 16 |
| S9 — Assistente IA | 18 | — | 18 |
| S10 — Mobile | 3 | — | 3 |
| CROSS (resolvidos em S1) | — | 27 | 27 |
| **Total** | **285** | | **285** |

---

## Dependências entre Sprints

```
S1 (Fundação)
  └── S2 (Auth e KYC)
        └── S3 (Dashboard e Perfil)
              └── S4 (Marketplace e Propostas)
                    └── S5 (Negociações)
                          └── S6 (Escrow)
                                └── S7 (Formalização)
                                      └── S8 (Fechamento e Financeiro)
                                            ├── S9 (Assistente IA) [Condicional]
                                            ├── S10 (Mobile) [Condicional — pós S1-S9]
                                            ├── S11 (Qualidade — pré-requisito: S1-S10)
                                            └── S12 (Go-Live — pré-requisito: S11)
                                                  └── S13 (Sprint Final — Auditoria)
```

---

## Arquivos de Sprint

| Arquivo | Caminho |
|---|---|
| `registro-mestre.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/registro-mestre.md` |
| `s1-fundacao.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s1-fundacao.md` |
| `s2-auth-kyc.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s2-auth-kyc.md` |
| `s3-dashboard-perfil.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s3-dashboard-perfil.md` |
| `s4-marketplace-propostas.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s4-marketplace-propostas.md` |
| `s5-negociacoes.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s5-negociacoes.md` |
| `s6-escrow.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s6-escrow.md` |
| `s7-formalizacao.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s7-formalizacao.md` |
| `s8-fechamento-financeiro.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s8-fechamento-financeiro.md` |
| `s9-assistente-ia.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s9-assistente-ia.md` |
| `s10-mobile.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s10-mobile.md` |
| `s11-qualidade.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s11-qualidade.md` |
| `s12-go-live.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s12-go-live.md` |
| `s13-auditoria.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/s13-auditoria.md` |
| `indice.md` | `docs/01 - Módulos/Cessionário/03 - Sprints/indice.md` |

---

## Legenda de Templates

| Template | Quando usar | Organização |
|---|---|---|
| **A — Infraestrutura** | S1 (Fundação) e S2 (Auth) | Banco → Backend → Frontend → Wiring → Testes |
| **B — Módulo Fullstack** | S3 a S10 | Por FEATURE com vertical slice: Banco → Backend → Frontend → Wiring → Testes |
| **Sprint de Qualidade** | S11 | Por tipo de qualidade: Unit/Integration/E2E/A11y/Segurança/Observabilidade |
| **Sprint de Lançamento** | S12 | Por timeline: T-7 → T-3 → T-1 → Launch Day |
| **Sprint Final** | S13 | Auditoria: Verificação → Matriz de Cobertura → Findings adversariais → GATE |
