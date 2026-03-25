# Índice de Sprints — AI-Dani-Admin

## Metadados

| Campo | Valor |
|---|---|
| Módulo | AI-Dani-Admin |
| Total de Sprints | 10 (S1–S9 construção + S10 auditoria) |
| Total de REQs | 283 (registro-mestre.md) |
| Cobertura | 100% — 283/283 REQs atribuídos |
| Data de Geração | 2026-03-24 |
| Responsável | Claude Code Desktop |

---

## Tabela de Sprints

| Sprint | Arquivo | Nome | Template | Docs Fonte | Itens Checklist (aprox.) | Status |
|---|---|---|---|---|---|---|
| S1 | `s1-fundacao.md` | Fundação | A (Infraestrutura) | D02, D12, D13, D14, D15, D20, D22, D25 | 31 | ☐ Pendente |
| S2 | `s2-auth.md` | Autenticação e RBAC | A (Infraestrutura) | D02, D18, D05 RF-026, D16, D20 | 20 | ☐ Pendente |
| S3 | `s3-supervisao-de-interacoes.md` | Supervisão de Interações | B (Fullstack) | D05 RF-001–010, D06, D07, D08, D09, D12, D16 | 30 | ☐ Pendente |
| S4 | `s4-takeover-manual.md` | Takeover Manual | B (Fullstack) | D05 RF-004–009, D14, D16, ADR-001 | 30 | ☐ Pendente |
| S5 | `s5-alertas-e-configuracao.md` | Alertas e Configuração | B (Fullstack) | D05 RF-011–016, D17, D21, D25 | 40 | ☐ Pendente |
| S6 | `s6-dashboard-de-metricas.md` | Dashboard de Métricas | B (Fullstack) | D05 RF-017–022, D06, D09, D25 | 30 | ☐ Pendente |
| S7 | `s7-webchat-e-agentes-de-ia.md` | Webchat e Agentes de IA | B (Fullstack) — Condicional IA | D19, D25, RN-DA-036, RN-DA-037 | 30 | ☐ Pendente |
| S8 | `s8-prontidao-para-lancamento.md` | Prontidão para Lançamento | B (Fullstack) | RN-DA-037–039, D11, RF-023 | 30 | ☐ Pendente |
| S9 | `s9-qualidade-e-go-live.md` | Qualidade e Go-Live | B (Fullstack) | D24, D27, D28, D29, RNF-001–012 | 30 | ☐ Pendente |
| S10 | `s10-auditoria.md` | Auditoria Final | Sprint Final | TODOS D01–D29 + registro-mestre.md | 70 | ☐ Pendente |

---

## Mapa de Docs → Sprints

| Doc | Nome | Sprint(s) Principal(ais) |
|---|---|---|
| D02 | Stacks | S1, S7, S9 |
| D05 | PRD (RF + RNF) | S2 a S9 |
| D06 | Mapa de Telas | S3 a S8 |
| D07 | Wireframes | S3, S4, S6 |
| D08 | UX Writing | S3 a S8 |
| D09 | Contratos de UI | S3 a S8 |
| D10 | Glossário Técnico | TODAS (cross-cutting) |
| D11 | Mobile | S8 |
| D12 | Modelo de Dados (ERD) | S1 |
| D13 | Schema Prisma | S1 |
| D14 | Especificações Técnicas | S1, S4 |
| D15 | Arquitetura de Pastas | S1 |
| D16 | Documentação de API | S3 a S8 |
| D17 | Integrações Externas | S5 |
| D18 | Autenticação OAuth | S2 |
| D19 | Criação de Agentes de IA | S7 |
| D20 | Error Handling | S1 a S9 |
| D21 | Notificações | S5 |
| D22 | Guia de Ambiente | S1 |
| D24 | Deploy CI/CD | S9 |
| D25 | Observabilidade | S1, S7, S9 |
| D26 | Runbook Operacional | S9 |
| D27 | Plano de Testes | S9 |
| D28 | Checklist de Qualidade | S9 |
| D29 | Go-Live Playbook | S9 |
| RN-DA-030–039 | Regras de Negócio | S3 a S9 |

---

## Mapa de REQs por Sprint

| Sprint | Faixa de REQs (aprox.) | Tipo Principal |
|---|---|---|
| S1 | REQ-001 a REQ-060 | Banco, Infra, Docker |
| S2 | REQ-061 a REQ-090 | Auth, RBAC, AuditLog |
| S3 | REQ-091 a REQ-120 | Supervisão, Realtime |
| S4 | REQ-121 a REQ-150 | Takeover, Estado |
| S5 | REQ-151 a REQ-180 | Alertas, Config, Monitor |
| S6 | REQ-181 a REQ-210 | Métricas, Dashboard |
| S7 | REQ-211 a REQ-230 | IA, RAG, PostHog |
| S8 | REQ-231 a REQ-250 | Prontidão, Mobile |
| S9 | REQ-251 a REQ-283 | Qualidade, CI/CD, Go-Live |
| CROSS | — | Deps entre módulos |

---

## Itens [REVISÃO MANUAL] por Sprint

| ID | Sprint | Severidade | Descrição |
|---|---|---|---|
| RM-001 | S7, S9 | P1 | `LATENCY_SLA_SECONDS` não definido — alerta de latência não implementável |
| RM-002 | S1 | RESOLVIDO | Vitest adotado como canônico (não Jest) |
| RM-003 | S1 | RESOLVIDO | Redis prod = Upstash |
| RM-004 | S1 | RESOLVIDO | RabbitMQ prod = CloudAMQP |
| RM-005 | S9 | P1 | endpoint `revoke` em D27 mas ausente de D16 — não implementado |
| RM-006 | S6 | P1 | Campo `csat_score` ausente nas migrations — `csatAverage` não implementável |
| RM-007 | S10 | P1 | RN-DA-034 não localizado — verificar D01 |
| RM-008 | S10 | P1 | RF-009 não localizado — verificar D05 |
| RM-009 | S10 | P1 | RNF-012 não localizado — verificar D05 |

**Total P1 abertos: 6 | Total Resolvidos: 3 | Total P0: 0**

---

## Status de GATE

| Gate | Critério | Resultado |
|---|---|---|
| Cobertura 100% REQs | 283/283 REQs têm sprint atribuída | ✅ APROVADO |
| Zero P0 | Nenhum item P0 aberto | ✅ APROVADO |
| P1 documentados | 6 itens P1 com [REVISÃO MANUAL] | ⚠️ 6 pendentes |
| Sprint Final gerada | s10-auditoria.md gerado | ✅ APROVADO |
| Índice gerado | indice.md gerado | ✅ APROVADO |
