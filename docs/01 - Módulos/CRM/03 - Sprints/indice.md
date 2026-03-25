# Índice de Sprints — CRM Repasse Seguro

## Pipeline v2.3 — Fase 3: Documentação das Sprints

| **Sprint** | **Nome** | **Arquivo** | **Template** | **Docs Fonte** | **REQs Cobertos** | **Itens Checklist** | **Status** |
|---|---|---|---|---|---|---|---|
| S1 | Fundação | `s1-fundacao.md` | A — Infraestrutura | 02, 12, 14 | REQ-001–014, REQ-191–204 | ~55 | Gerado |
| S2 | Auth | `s2-auth.md` | A — Infraestrutura | 01.1, 02, 05.1, 06 | REQ-015–030 | ~45 | Gerado |
| S3 | Pipeline de Casos | `s3-pipeline-casos.md` | B — Módulo Fullstack | 01.1, 01.2, 05.1, 06, 12, 16, 27 | REQ-031–057 | ~58 | Gerado |
| S4 | Negociação e Comissão | `s4-negociacao-comissao.md` | B — Módulo Fullstack | 01.2, 05.1, 06, 12, 16, 27 | REQ-058–068 | ~52 | Gerado |
| S5 | Dossiê e Contatos | `s5-dossie-contatos.md` | B — Módulo Fullstack | 01.2, 01.3, 06, 12, 16, 27 | REQ-098–138 | ~60 | Gerado |
| S6 | Atividades | `s6-atividades.md` | B — Módulo Fullstack | 01.3, 05.3, 06, 16, 27 | REQ-068–072, REQ-161–162, REQ-205–207, REQ-260, REQ-283, REQ-310 | ~40 | Gerado |
| S7 | Comunicação e Integrações | `s7-comunicacao-integracoes.md` | B — Módulo Fullstack | 01.3, 01.5, 06, 16, 27 | REQ-073–076, REQ-097–107, REQ-139–144 | ~55 | Gerado |
| S8 | SLA, Dashboard e Notificações | `s8-sla-dashboard.md` | B — Módulo Fullstack | 01.3, 01.4, 05.1, 06, 14, 16, 27 | REQ-077–082, REQ-088–090, REQ-109, REQ-145, REQ-165–166, REQ-223–224, REQ-243–244, REQ-255, REQ-274, REQ-292, REQ-306–308 | ~42 | Gerado |
| S9 | Equipe, Relatórios e Configurações | `s9-equipe-relatorios-configuracoes.md` | B — Módulo Fullstack | 01.4, 05.1, 06, 16, 27 | REQ-083–087, REQ-091–096, REQ-167–179, REQ-186–190, REQ-225–230, REQ-258, REQ-279, REQ-284 | ~52 | Gerado |
| S10 | Qualidade, Performance e Segurança | `s10-qualidade.md` | B — Módulo Fullstack | 02, 14, 20, 27 | REQ-252, REQ-261–262, REQ-285–299 | ~35 | Gerado |
| S-Final | Auditoria e Sprint de Correções | `sfinal-auditoria.md` | Sprint Final (7 checks) | Todos | REQ-001–312 (verificação) | ~35 | Gerado |

---

## Sprints Condicionais — Não Geradas (MVP)

| **Sprint Condicional** | **Motivo de Exclusão** | **Doc Fonte** |
|---|---|---|
| Agentes de IA | Dani-Admin é webhook passivo no MVP. Sem agente autônomo (LangChain.js + GPT-4o apenas na Fase 2) | Doc 19 |
| Mobile | Sem app nativo no MVP. Responsivo web a partir de 768px apenas | Doc 11 |

---

## Resumo de Cobertura

| **Faixa de REQs** | **Total** | **Sprints** |
|---|---|---|
| REQ-001 a REQ-030 | 30 | S1, S2 |
| REQ-031 a REQ-072 | 42 | S3, S4, S6 |
| REQ-073 a REQ-144 | 72 | S5, S6, S7, S8 |
| REQ-145 a REQ-204 | 60 | S8, S9, S1 |
| REQ-205 a REQ-260 | 56 | S6, S7, S8, S9, S10 |
| REQ-261 a REQ-312 | 52 | S10, S8, S9, S-Final |
| **TOTAL** | **312** | **S1–S10 + S-Final** |

---

## Dependências entre Sprints (Gates)

| **Gate** | **Sprint que Depende** | **Sprint Pré-requisito** |
|---|---|---|
| VERIFICACAO→PUBLICACAO (Dossiê APROVADO + Tabela Atual ≤30 dias) | S3 (gate da transição) | S5 (implementa Dossiê) |
| Fechamento com 3 critérios cumulativos | S4 (CommissionsService.confirmFechamento) | S5 (Dossiê: critério 1 PDF assinado), S7 (ZapSign: critério 1; Celcoin: critério 3) |
| Linha do tempo completa (getCaseTimeline) | S6 (implementação) | S3 (placeholder criado), S4, S5, S7, S8 (dados de entrada) |
| Follow-up notifications via SSE | S6 (agenda job RabbitMQ) | S8 (SSE implementado, NotificationProvider) |
| Alertas SLA na timeline | S8 (sla_alerts criados) | S6 (getCaseTimeline consome sla_alerts) |
| WeeklyReportWorker com dados completos | S8 (worker) | S3, S4, S5, S6, S7 (dados de entrada) |
| 14 parâmetros sistema lidos pelos workers | S9 (PATCH /system-configs) | S3, S4, S5, S6 (leem parâmetros no runtime) |
| 5 fluxos E2E cruzam múltiplos módulos | S10 | S1–S9 implementados |

---

## Arquivos Gerados

```
docs/01 - Módulos/CRM/03 - Sprints/
├── registro-mestre.md          ← Registro Mestre (312 REQs)
├── s1-fundacao.md
├── s2-auth.md
├── s3-pipeline-casos.md
├── s4-negociacao-comissao.md
├── s5-dossie-contatos.md
├── s6-atividades.md
├── s7-comunicacao-integracoes.md
├── s8-sla-dashboard.md
├── s9-equipe-relatorios-configuracoes.md
├── s10-qualidade.md
├── sfinal-auditoria.md
├── indice.md                   ← Este arquivo
└── auditoria-a03.md            ← Gerado após execução da Auditoria A03
```

---

## Métricas do Pipeline

| **Métrica** | **Valor** |
|---|---|
| Total de sprints geradas | 11 (S1–S10 + S-Final) |
| Total de REQs mapeados | 312 |
| Total estimado de itens de checklist | ~529 |
| Sprints condicionais excluídas (MVP) | 2 (IA, Mobile) |
| Documentos fonte lidos | 15 de 29 (críticos) |
| Cobertura de REQs | 312/312 = 100% |
| Template A (Infraestrutura) | S1, S2 |
| Template B (Módulo Fullstack) | S3 a S10 |
| Sprint Final | S-Final |
