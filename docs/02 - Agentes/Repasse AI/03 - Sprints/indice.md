# Índice de Sprints — Repasse AI

| Campo | Valor |
|---|---|
| **Versão** | v1.0 |
| **Data** | 2026-03-24 |
| **Total de sprints** | 10 (S1, S2, S3a, S3b, S4, S5, S6, S7, S8, S9, S10) |
| **Total de REQs** | 185 |
| **Cobertura** | 185/185 (100%) |

---

## Tabela de Sprints

| Sprint | Nome | Template | Docs Fonte | Itens | REQs Cobertos | Status |
|---|---|---|---|---|---|---|
| S1 | Fundação | A | 02, 12, 13, 14, 17, 22 | 27 | REQ-007, REQ-114 a REQ-158, REQ-168 a REQ-173, REQ-178 | ⬜ |
| S2 | Auth e Sessão | A | 01.1, 01.4, 01.5, 14, 16, 22 | 14 | REQ-004 a REQ-006, REQ-022, REQ-023, REQ-028, REQ-071, REQ-072, REQ-094, REQ-111, REQ-135 | ⬜ |
| S3a | Agente Core — Parte 1 | B | 01.2, 01.3, 14, 16, 19 | 14 | REQ-008 a REQ-013, REQ-024 a REQ-027, REQ-029 a REQ-032, REQ-036, REQ-062 a REQ-067, REQ-073, REQ-075, REQ-076 | ⬜ |
| S3b | Agente Core — Parte 2 | B | 01.1, 01.3, 14, 16, 19 | 12 | REQ-014 a REQ-021, REQ-038, REQ-039, REQ-093, REQ-112, REQ-113, REQ-120 a REQ-134, REQ-159 a REQ-163, REQ-174 a REQ-177 | ⬜ |
| S4 | Calculadora de Comissão | B | 01.2, 01.3, 16, 19 | 8 | REQ-022, REQ-023, REQ-033 a REQ-035, REQ-068 a REQ-070, REQ-074, REQ-095 a REQ-097 | ⬜ |
| S5 | Comparação e Simulação | B | 01.2, 14, 16, 19 | 11 | REQ-040 a REQ-058 | ⬜ |
| S6 | Admin e Supervisão | B | 01.4, 14, 16 | 15 | REQ-059 a REQ-061, REQ-077 a REQ-091, REQ-098 a REQ-103 | ⬜ |
| S7 | WhatsApp e Notificações | B | 01.5, 14, 16, 17 | 15 | REQ-001 a REQ-003, REQ-104 a REQ-110, REQ-136 a REQ-158, REQ-164 a REQ-167 | ⬜ |
| S8 | Qualidade e Cobertura | B | 24, 27 | 15 | REQ-178 a REQ-185 | ⬜ |
| S9 | Go-Live | B | 24, 29 | 9 | REQ-115 a REQ-119 | ⬜ |
| S10 | Sprint Final — Auditoria | Final | Todos | N/A | 185 REQs (verificação) | ⬜ |

---

## Distribuição de REQs por Sprint

| Sprint | Qtd REQs | % do Total |
|---|---|---|
| S1 | 47 | 25.4% |
| S2 | 11 | 5.9% |
| S3a | 25 | 13.5% |
| S3b | 37 | 20.0% |
| S4 | 12 | 6.5% |
| S5 | 19 | 10.3% |
| S6 | 24 | 13.0% |
| S7 | 37 | 20.0% |
| S8 | 8 | 4.3% |
| S9 | 5 | 2.7% |
| **Total único** | **185** | **100%** |

*Nota: alguns REQs aparecem em múltiplas sprints (ex: REQ-022, REQ-023 em S2 e S4); a contagem acima refere-se a REQs únicos por sprint primária.*

---

## Arquivos Gerados

| Arquivo | Sprint | Caminho |
|---|---|---|
| `registro-mestre.md` | — | `docs/02 - Agentes/Repasse AI/03 - Sprints/registro-mestre.md` |
| `s1-fundacao.md` | S1 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s1-fundacao.md` |
| `s2-auth.md` | S2 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s2-auth.md` |
| `s3a-agente-core-parte1.md` | S3a | `docs/02 - Agentes/Repasse AI/03 - Sprints/s3a-agente-core-parte1.md` |
| `s3b-agente-core-parte2.md` | S3b | `docs/02 - Agentes/Repasse AI/03 - Sprints/s3b-agente-core-parte2.md` |
| `s4-calculadora-comissao.md` | S4 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s4-calculadora-comissao.md` |
| `s5-comparacao-simulacao.md` | S5 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s5-comparacao-simulacao.md` |
| `s6-admin-supervisao.md` | S6 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s6-admin-supervisao.md` |
| `s7-whatsapp-notificacoes.md` | S7 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s7-whatsapp-notificacoes.md` |
| `s8-qualidade.md` | S8 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s8-qualidade.md` |
| `s9-go-live.md` | S9 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s9-go-live.md` |
| `s10-auditoria.md` | S10 | `docs/02 - Agentes/Repasse AI/03 - Sprints/s10-auditoria.md` |
| `indice.md` | — | `docs/02 - Agentes/Repasse AI/03 - Sprints/indice.md` |
| `auditoria-a03.md` | — | `docs/02 - Agentes/Repasse AI/03 - Sprints/auditoria-a03.md` |

---

## Dependências entre Sprints

```
S1 (Fundação)
  └── S2 (Auth)
        ├── S3a (Agente Core P1)
        │     └── S3b (Agente Core P2)
        │           ├── S4 (Calculadora) ← S3a (tools)
        │           ├── S5 (Comparação/Simulação) ← S4
        │           └── S6 (Admin) ← S3b
        └── S7 (WhatsApp) ← S3b
              └── S8 (Qualidade) ← S1-S7
                    └── S9 (Go-Live) ← S8
                          └── S10 (Auditoria Final) ← S1-S9
```

---

## Sprints Condicionais

| Tipo | Decisão | Justificativa |
|---|---|---|
| Mobile (Fase 2) | NÃO APLICÁVEL | Repasse AI é backend puro (PG-03); sem frontend próprio; sem Playwright/Cypress |
| Agentes de IA | INCORPORADO em S3a/S3b | O produto inteiro É o agente de IA — não é sprint separada |
| WhatsApp | S7 (Fase 2) | Documentado em 01.5 como Fase 2 condicional; implementado em sprint dedicada |
