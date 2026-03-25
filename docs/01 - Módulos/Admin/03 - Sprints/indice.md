# Índice de Sprints — Módulo Admin

## Repasse Seguro

| Campo | Valor |
|---|---|
| **Gerado em** | 2026-03-24 |
| **Total de sprints** | 17 |
| **Total de itens** | 741 |
| **Total de REQs cobertos** | 312 / 312 (100%) |
| **Status Fase 3** | Concluída — aguardando Auditoria A03 |

---

## Tabela de Sprints

| # | Arquivo | Nome | Docs Fonte | REQs | Itens | Status |
|---|---------|------|------------|------|-------|--------|
| S1 | `s1-fundacao.md` | Fundação | D02, D12, D13, D15, D22, D23, D24, D25 | 42 | 58 | Pendente |
| S2 | `s2-auth.md` | Auth e RBAC | D01.1 §7, D14, D16 §2, D18 | 29 | 52 | Pendente |
| S3 | `s3-dashboard.md` | Dashboard | D01.1 §8, D06 §2.1, D01.5 §4 | 9 | 40 | Pendente |
| S4 | `s4-pipeline-casos.md` | Pipeline e Casos | D01.1 §4–5, D06 §2.2, D16 §6 | 18 | 55 | Pendente |
| S4b | `s4b-triagem.md` | Triagem | D01.2, D06 §2.3, D16 §7 | 22 | 52 | Pendente |
| S5 | `s5-negociacao.md` | Negociação | D01.2, D06 §2.4, D16 §8 | 20 | 48 | Pendente |
| S6 | `s6-financeiro.md` | Financeiro | D01.2, D06 §2.6, D16 §10, D17 §Celcoin | 24 | 50 | Pendente |
| S7 | `s7-formalizacao.md` | Formalização | D01.1 §4, D06 §2.5, D16 §9, D17 §ZapSign | 28 | 54 | Pendente |
| S8 | `s8-usuarios.md` | Usuários | D01.3 §4, D06 §2.9, D16 §3–5 | 17 | 42 | Pendente |
| S8a | `s8a-notificacoes.md` | Notificações | D21, D01.3 §5, D06 §Topbar | 3 | 38 | Pendente |
| S9 | `s9-supervisao-ia.md` | Supervisão IA | D01.4, D06 §2.8, D16 §11, D19 | 21 | 46 | Pendente |
| S10 | `s10-relatorios.md` | Relatórios | D01.4, D06 §2.10, D16 §12 | 10 | 36 | Pendente |
| S11 | `s11-configuracoes.md` | Configurações | D01.4, D06 §2.11, D16 §13 | 10 | 38 | Pendente |
| S12 | `s12-mobile.md` | Mobile | D02 §5, D11 | 1 | 36 | Pendente |
| S13 | `s13-qualidade.md` | Qualidade | D27, D28 | 1 | 42 | Pendente |
| SN1 | `sN1-go-live.md` | Go-Live | D29, D28, D24, D26 | CROSS | 44 | Pendente |

---

## Registro Mestre

| Arquivo | Propósito |
|---------|-----------|
| `registro-mestre.md` | Fonte da verdade — 312 REQs de D01.1–D01.5, D02, D06, D10, D12, D13, D14, D15, D16, D17, D18, D19, D20, D21, D22, D23, D24, D25, D26, D27, D28, D29 |

---

## Distribuição de REQs por Sprint

| Sprint | Qtd REQs | % do total |
|--------|----------|------------|
| S1 — Fundação | 42 | 13,5% |
| S2 — Auth e RBAC | 29 | 9,3% |
| S3 — Dashboard | 9 | 2,9% |
| S4 — Pipeline e Casos | 18 | 5,8% |
| S4b — Triagem | 22 | 7,1% |
| S5 — Negociação | 20 | 6,4% |
| S6 — Financeiro | 24 | 7,7% |
| S7 — Formalização | 28 | 9,0% |
| S8 — Usuários | 17 | 5,4% |
| S8a — Notificações | 3 | 1,0% |
| S9 — Supervisão IA | 21 | 6,7% |
| S10 — Relatórios | 10 | 3,2% |
| S11 — Configurações | 10 | 3,2% |
| S12 — Mobile | 1 | 0,3% |
| S13 — Qualidade | 1 | 0,3% |
| CROSS | 7 | 2,2% |
| **Total** | **312** | **100%** |

> REQs CROSS não são alocados a uma sprint específica — permeiam toda a implementação (Glossário D10, nomenclatura canônica D10, ADRs D02).

---

## Ordem de Execução Recomendada

```
S1 Fundação
→ S2 Auth e RBAC
→ S3 Dashboard
→ S4 Pipeline e Casos
→ S4b Triagem
→ S5 Negociação
→ S6 Financeiro
→ S7 Formalização
→ S8 Usuários
→ S8a Notificações
→ S9 Supervisão IA (condicional — produto utiliza)
→ S10 Relatórios
→ S11 Configurações
→ S12 Mobile (condicional — produto utiliza)
→ S13 Qualidade
→ SN1 Go-Live
```

---

## Dependências Críticas

| Sprint | Depende de | Motivo |
|--------|------------|--------|
| S2 | S1 | Schema `users`, Redis, JWT configurados |
| S3 | S1, S2 | Auth obrigatório para dashboard |
| S4 | S1, S2 | Schema `cases`, RBAC por perfil |
| S4b | S4 | Casos em status `CAPTADO` devem existir |
| S5 | S4b | Casos `QUALIFICADO` → `OFERTA_ATIVA` obrigatório |
| S6 | S5 | Casos `EM_FORMALIZACAO` + conta escrow |
| S7 | S6 | ZapSign, escrow account e critérios de formalização |
| S8 | S1, S2 | Schema `users`, `cedentes`, `cessionarios` |
| S8a | S1 | RabbitMQ exchange `rs.notifications`, workers |
| S9 | S8a | Notificações de agente dependem de S8a |
| S10 | S1–S9 | Agrega dados de todas as sprints anteriores |
| S11 | S1, S2 | Schema `global_configs`, MASTER only |
| S12 | S2 | Auth mobile, tokens, push notifications |
| S13 | S1–S12 | Testes cobrem todos os módulos anteriores |
| SN1 | S1–S13 | Go-live apenas após qualidade aprovada |

---

## Glossário de Documentos Fonte

| Código | Arquivo | Conteúdo |
|--------|---------|----------|
| D01.1 | `01.1 - Regras de Negócio — Fundação e Acessos.md` | Glossário, perfis, máquinas de estado, RN-001–RN-008 |
| D01.2 | `01.2 - Regras de Negócio — Módulos Core e Receita.md` | Pipeline, negociação, escrow, comissões, RN-009–RN-090 |
| D01.3 | `01.3 - Regras de Negócio — Módulos Operação e Suporte.md` | Usuários, notificações, RN-055–RN-071 |
| D01.4 | `01.4 - Regras de Negócio — Módulos Administração e Configuração.md` | IA, relatórios, configurações, RN-087–RN-120 |
| D01.5 | `01.5 - Regras de Negócio — Integrações, Transversais e Consolidação.md` | Integrações, LGPD, transversais |
| D02 | `02 - Stacks.md` | Stack tecnológico, ADRs, versões mínimas |
| D06 | `06 - Mapa de Telas.md` | T-001 a T-100, rotas, especificações de telas |
| D10 | `10 - Glossário Técnico.md` | Nomenclatura canônica — 18 entidades, 14 sinônimos resolvidos |
| D11 | `11 - Mobile.md` | Stack mobile, telas M-001 a M-050, push, offline |
| D12 | `12 - Modelo de Dados (ERD Schema).md` | 22 tabelas, enums, índices, schema audit |
| D13 | `13 - Schema Prisma.md` | Schema Prisma completo |
| D14 | `14 - Especificações Técnicas.md` | Auth 2FA, sessions, RBAC detalhado |
| D15 | `15 - Arquitetura de Pastas.md` | Estrutura monorepo, pastas por camada |
| D16 | `16 - Documentação de API.md` | Todos os endpoints, contratos, status codes |
| D17 | `17 - Integrações Externas.md` | ZapSign, Celcoin, Resend, Meta, Twilio |
| D18 | `18 - Fluxos de Autenticação e Autorização.md` | JWT, refresh token, blacklist, RBAC guards |
| D19 | `19 - Criação de Agentes de IA.md` | GuardiaoDoRetorno, AnalistaOportunidades, circuit breaker |
| D20 | `20 - Error Handling.md` | RFC 7807, GlobalExceptionFilter, error codes |
| D21 | `21 - Notificações, Templates e Implementação.md` | 22 templates, 4 canais, retry policy |
| D22 | `22 - Guia de Ambiente, Setup Local e Secrets.md` | .env, setup local, secrets |
| D23 | `23 - Guia de Contribuição.md` | Git flow, PR template, code style |
| D24 | `24 - Deploy, CI-CD e Versionamento.md` | Pipelines, Railway, Vercel, GitHub Actions |
| D25 | `25 - Observabilidade e Logs.md` | Pino, correlation ID, SLOs, dashboards |
| D26 | `26 - Runbook Operacional.md` | Incidentes, on-call, 10 cenários |
| D27 | `27 - Plano de Testes.md` | Pirâmide 70/20/5/5, Vitest, Playwright |
| D28 | `28 - Checklist de Qualidade.md` | Gates B01–B15, WCAG, segurança, observabilidade |
| D29 | `29 - Go-Live Playbook.md` | T-7/T-3/T-1, launch day, rollback, Supervisão Total IA |

---

*Índice gerado pelo pipeline ShiftLabs v2.3 — módulo Admin, Repasse Seguro.*
