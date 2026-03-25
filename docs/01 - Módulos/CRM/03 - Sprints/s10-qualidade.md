# S10 — Qualidade, Performance e Segurança

## CRM Repasse Seguro

| **Campo**         | **Valor**                                                                                                                                                                                                                                                    |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Sprint**        | S10                                                                                                                                                                                                                                                          |
| **Nome**          | Qualidade, Performance e Segurança                                                                                                                                                                                                                           |
| **Template**      | B — Módulo Fullstack                                                                                                                                                                                                                                         |
| **Docs fonte**    | 02, 14, 20, 27                                                                                                                                                                                                                                               |
| **REQs cobertos** | REQ-252, REQ-261, REQ-262, REQ-285 a REQ-312 (qualidade transversal)                                                                                                                                                                                         |
| **Objetivo**      | Atingir cobertura de testes exigida (80% módulos críticos, 60% integração, 5 fluxos E2E completos), validar performance (API <200ms p95), verificar OWASP Top 10, WCAG 2.1 AA, tablet 768px, e resolver todos os P0/P1 identificados nas sprints anteriores. |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — Jest (backend), Vitest (frontend), Supertest (integração), Playwright (E2E), 5 fluxos E2E exatos, cobertura 80%/60%/100%
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: Valores exatos — 80% cobertura módulos críticos, 60% integração, 100% 5 E2E críticos, <200ms p95, WCAG 2.1 AA, 768px tablet, OWASP Top 10
- [x] C4: Glossário — P0/P1/P2/P3 conforme Doc 20, WCAG, OWASP cobertos
- [x] C5: Sem máquinas de estado novas nesta sprint (apenas validação de completude das existentes)
- [x] C6: Sem workers novos — validação de workers existentes (S3/S6/S8/S9)
- [x] C7: RBAC — testes E2E cobrem todos os 4 roles em pelo menos 1 fluxo cada
- [x] C8: Sem telas novas — testes de regressão visual e acessibilidade nas telas existentes
- [x] C9: Integração — os 5 fluxos E2E cruzam módulos implementados em sprints anteriores
- [x] C10: Glossário — DLQ, Retry Policy, k-anonymity (relatório inteligência), SLA, Optimistic Locking cobertos nos testes
- [x] C11: Anti-scaffold — cada teste E2E tem steps detalhados com asserções reais; testes de performance com thresholds reais; OWASP com payloads reais
- [x] C12: REQ-252, REQ-261, REQ-262, REQ-285 a REQ-312 cobertos

---

## FEATURE 1 — Cobertura de Testes Backend (Jest + Supertest)

### Backend — Cobertura mínima por módulo

- [x] **REQ-261** Atingir e verificar cobertura 80% nos módulos críticos. Executar `jest --coverage` e confirmar:
  - `CasesService` ≥ 80%: focar em transições de estado, rollback (justificativa 20 chars), redistribuição (limite 30 Casos), gate VERIFICACAO→PUBLICACAO
  - `CommissionsService` ≥ 85% (conforme S4): focar em fórmulas Cedente/Cessionário, tabela de desconto, fechamento 3 critérios
  - `ActivitiesService` ≥ 70% (conforme S6): focar em retroatividade 30 dias, validação resumo 20 chars, prioridade alta follow-up
  - `SlaMonitorService` ≥ 70% (conforme S8): focar em 4 níveis de alerta, cálculo de dias, alerta CRITICO 60 dias
  - `ZapSignService` ≥ 75% (conforme S7): focar em criação de envelope, retentativas D+2/D+4/D+5, webhook all_signed
  - `DossierService` ≥ 70% (conforme S5): focar em gate aprovação, validade Tabela Atual, share link
  - `TeamService` ≥ 70% (conforme S9): focar em ban 180 dias, quatro-olhos parâmetros críticos, redistribuição ao desligar
  - Validar: `jest --coverage --coverageReporters=text-summary` exibe cada módulo ≥ threshold; CI/CD falha se abaixo

- [x] **REQ-262** Atingir e verificar cobertura 60% nos testes de integração (Supertest). Confirmar suítes de integração de S2 a S9 executando sem falhas:
  - Auth flow completo (S2): login → token → refresh → logout
  - Case pipeline (S3): criar Caso → avançar para SIMULACAO → verificar SLA alert
  - Negotiation + Commission (S4): proposta → aceitação → fechamento → comissão calculada
  - Dossier + Contacts (S5): upload → aprovação → gate VERIFICACAO→PUBLICACAO
  - Activities + Timeline (S6): atividade registrada → aparece na timeline
  - WhatsApp + ZapSign (S7): envio mensagem → webhook recebido; envelope criado → webhook all_signed → PDF no dossiê
  - Dashboard + Notifications (S8): métricas geradas → alerta SLA → SSE recebido
  - Team + Config (S9): convite → aceitação → suspensão → reativação; parâmetro crítico → quatro-olhos → aprovação
  - Validar: `jest --testPathPattern=integration --coverage` exibe ≥ 60%; todas as suítes passam

---

## FEATURE 2 — 5 Fluxos E2E Críticos (Playwright)

- [x] **REQ-252 / REQ-285** E2E Fluxo 1 — Criação e Avanço de Caso pelo Pipeline completo:
  - **Setup**: usuário Analista RS logado, Parceiro Externo cadastrado com Caso vinculado
  - **Steps**:
    1. Navegar para `/cases/new` → preencher todos os campos obrigatórios → submit
    2. Verificar: Caso aparece no kanban com estado `CADASTRO` e número RS-YYYY-NNNN
    3. Avançar para `SIMULACAO` → preencher justificativa → confirmar
    4. Avançar para `VERIFICACAO` → confirmar
    5. Fazer upload de 3 documentos do Dossiê → verificar estado do Dossiê como `EM_ANALISE`
    6. Aprovar Dossiê como Coordenador RS → verificar estado `APROVADO`
    7. Avançar para `PUBLICACAO` (gate: Dossiê APROVADO + Tabela Atual válida) → confirmar
    8. Avançar para `MATCH` → confirmar
    9. Entrar em `NEGOCIACAO`: criar proposta com valor → confirmar
    10. Aceitar proposta → avançar para `ANUENCIA`
    11. Avançar para `FORMALIZACAO`
    12. Avançar para `CONCLUIDO` (gate: 3 critérios) → verificar comissão gerada
  - **Asserções**: estado correto em cada etapa; número RS gerado; comissão calculada corretamente na conclusão; audit_log com cada transição

- [x] **REQ-286** E2E Fluxo 2 — Assinatura Eletrônica via ZapSign:
  - **Setup**: Caso em `FORMALIZACAO`, usuário Coordenador RS
  - **Steps**:
    1. Navegar para Caso → aba Dossiê → seção "Instrumento de Cessão"
    2. Click "Iniciar assinatura ZapSign" → confirmar 3 signatários (Cedente, Cessionário, RS)
    3. Verificar: envelope criado → status `AGUARDANDO_ASSINATURAS`
    4. Simular webhook ZapSign `all_signed` (via `curl` ou Playwright `page.route`)
    5. Verificar: PDF adicionado ao Dossiê automaticamente; critério de Fechamento 1 satisfeito; status envelope `CONCLUIDO`
  - **Asserções**: envelope no ZapSign criado com 3 signatários; PDF no Dossiê após webhook; `closing_criteria_1 = true` no `negotiations`

- [x] **REQ-287** E2E Fluxo 3 — Geração de Relatório e Export:
  - **Setup**: Admin RS logado, pelo menos 5 Casos com dados históricos (seed de dados)
  - **Steps**:
    1. Navegar para `/reports`
    2. Verificar: 10 cards de relatórios visíveis para Admin RS
    3. Click "Funil de Conversão" → selecionar período últimos 30 dias → click "Gerar"
    4. Verificar: tabela de resultados populated com dados por estado
    5. Click "Exportar XLSX" → verificar download iniciado
    6. Click "Exportar CSV" → verificar download iniciado
    7. Login como Coordenador RS → verificar: relatórios 5 e 10 ausentes
  - **Asserções**: download XLSX com `Content-Disposition` header correto; CSV com BOM UTF-8; Coord RS não vê relatórios exclusivos Admin

- [x] **REQ-288** E2E Fluxo 4 — Ciclo de Alerta SLA e Notificação:
  - **Setup**: Caso em `NEGOCIACAO` criado há 8 dias (seed com data retroativa)
  - **Steps**:
    1. Executar `SlaCheckerWorker` manualmente via endpoint de teste `POST /admin/workers/run/sla-checker`
    2. Verificar: alerta `VERMELHO` inserido em `sla_alerts` para o Caso
    3. Login como Analista RS dono do Caso → verificar badge no sino de notificações (contagem > 0)
    4. Click no sino → Central de Notificações → verificar notificação "Alerta SLA: Caso RS-..." presente
    5. Click na notificação → navegar para o Caso → verificar badge `VERMELHO` no card
    6. Dashboard Executivo (Coord RS) → verificar contagem de "SLA Vencido" aumentou
    7. Resolver alerta → `PATCH /sla/alerts/:id/resolve` → verificar `resolved_at` preenchido
  - **Asserções**: alerta criado após worker; notificação no SSE (simulada); badge por nível de cor correto; resolução funcional

- [x] **REQ-289** E2E Fluxo 5 — LGPD: Opt-out de Comunicações:
  - **Setup**: Contato com `opt_out_communications = false`, histórico de comunicações WhatsApp
  - **Steps**:
    1. Contato recebe mensagem de opt-out (via webhook simulado com `STOP`)
    2. Verificar: `contacts.opt_out_communications = true`; `contacts.opt_out_at = NOW()`
    3. Tentar enviar nova mensagem WhatsApp para o contato → verificar: 422 `CRM-031` "Contato optou por não receber comunicações."
    4. Navegar para perfil do Contato → verificar badge "Opt-out ativo" visível
    5. Solicitar dados (LGPD) via endpoint → verificar resposta dentro de 15 dias (flag no sistema)
  - **Asserções**: opt-out bloqueia envio de mensagem; badge visível na UI; solicitação LGPD registrada em `notification_logs` com prazo

---

## FEATURE 3 — Performance

- [x] **REQ-290** Validar SLA de API < 200ms p95. Executar suite de benchmark com `k6` ou `autocannon`:
  - Endpoints críticos sob carga de 50 req/s por 60 segundos: `GET /cases`, `GET /cases/:id`, `POST /cases/:id/activities`, `GET /dashboard/metrics`, `GET /reports/1/generate`
  - Asserção: p95 < 200ms para cada endpoint; p99 < 500ms; zero erros 5xx sob carga normal
  - Validar: relatório de benchmark gerado; se algum endpoint falhar → criar issue P1 no registro de pendências

- [x] **REQ-291** Validar queries de banco de dados sem N+1. Revisar com `EXPLAIN ANALYZE`:
  - `GET /cases` com 500 Casos: verificar ausência de N+1 em `contacts`, `sla_alerts`, `tags`
  - `GET /cases/:id/activities` com 200 atividades: verificar paginação com OFFSET não ultrapassa 10ms
  - `GET /dashboard/metrics`: verificar que queries usam indexes (não Seq Scan em tabelas > 1.000 rows)
  - Validar: nenhuma query com Seq Scan em `cases`, `activities`, `sla_alerts`; indexes existem e são usados

---

## FEATURE 4 — Segurança (OWASP Top 10)

- [x] **REQ-293** Validar OWASP A01 — Broken Access Control:
  - Analista RS tentando acessar Caso de outro Analista → 403
  - Parceiro Externo tentando acessar endpoint de Coordenador RS → 403
  - IDOR: tentar `GET /cases/:id` com UUID de Caso de outro tenant → 404 (não 403 para não revelar existência)
  - Validar: RLS PostgreSQL testado com queries diretas simulando outro `auth.uid()`

- [x] **REQ-294** Validar OWASP A02 — Cryptographic Failures:
  - Verificar: HttpOnly cookie em todas as respostas de auth (`Set-Cookie: ... HttpOnly; Secure; SameSite=Strict`)
  - JWT sem `alg: none` → verificar header do token decodificado
  - Dados pessoais no banco: CPF, e-mail não armazenados em plain text em logs (verificar Pino config)
  - Validar: tentativa de token `alg: none` → 401; cookie sem HttpOnly → item de correção P0

- [x] **REQ-295** Validar OWASP A03 — Injection:
  - SQL Injection: enviar `'; DROP TABLE cases; --` em query params de `GET /cases?search=` → verificar sanitização do Prisma
  - XSS: enviar `<script>alert(1)</script>` em campos de texto → verificar output em HTML escapado
  - Validar: Prisma usa prepared statements (sem raw queries com concatenação); output HTML escapado pelo React

- [x] **REQ-296** Validar OWASP A05 — Security Misconfiguration:
  - Verificar: Helmet configurado em NestJS (S1) com headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Content-Security-Policy`, `Strict-Transport-Security`
  - Verificar: Swagger UI desativado em `NODE_ENV=production`
  - Validar: curl para endpoint em produção → headers de segurança presentes; Swagger → 404 em produção

- [x] **REQ-297** Validar OWASP A07 — Identification and Authentication Failures:
  - Verificar bloqueio após 5 tentativas (15 min) configurado em S2
  - Verificar sessão inativa expira em 60 min
  - Verificar HMAC-SHA256 em webhooks ZapSign e Celcoin (S7)
  - Validar: 6ª tentativa de login → 429 com `Retry-After` header; sessão expirada → 401

---

## FEATURE 5 — Acessibilidade (WCAG 2.1 AA)

- [x] **REQ-298** Executar auditoria WCAG 2.1 AA com `axe-core` via Playwright em 5 telas críticas: T-CRM-001 (Login), T-CRM-020 (Kanban), T-CRM-022 (Detalhe do Caso), T-CRM-030 (Negociação), T-CRM-080 (Dashboard Executivo). Asserções:
  - Zero violações de nível A ou AA
  - Todos os elementos interativos têm `aria-label` ou texto visível
  - Contraste mínimo 4.5:1 para texto normal, 3:1 para texto grande
  - Navegação por teclado funcional em formulários modais (Tab, Enter, Escape)
  - Validar: relatório `axe` sem violações AA; Tab navega por todos os campos do `ActivityModal` (T-CRM-061)

---

## FEATURE 6 — Tablet 768px (Responsividade)

- [x] **REQ-299** Validar 4 funcionalidades críticas em viewport 768px (Playwright com `page.setViewportSize({width: 768, height: 1024})`):
  1. **Cadastro de Caso**: `POST /cases` a partir de T-CRM-020 → formulário acessível e submetível
  2. **Registro de Atividade**: `ActivityModal` (T-CRM-061) abre corretamente, campos visíveis e usáveis
  3. **Visualização do Pipeline**: Kanban (T-CRM-020) em modo lista (fallback em 768px, sem scroll horizontal)
  4. **Upload de Documento**: Dossiê (T-CRM-040) com input file visível e funcional
  - Sidebar: recolhida por padrão em < 1280px (implementada em S1) — verificar comportamento em 768px
  - Validar: zero overflow horizontal em 768px nas 4 telas; touch targets ≥ 44px; sidebar recolhida

---

## 🧪 Sumário de Cobertura Esperada

| Módulo               | Tipo        | Threshold    | Sprint Origem |
| -------------------- | ----------- | ------------ | ------------- |
| `CasesService`       | Unit        | 80%          | S3            |
| `CommissionsService` | Unit        | 85%          | S4            |
| `DossierService`     | Unit        | 70%          | S5            |
| `ActivitiesService`  | Unit        | 70%          | S6            |
| `ZapSignService`     | Unit        | 75%          | S7            |
| `SlaMonitorService`  | Unit        | 70%          | S8            |
| `TeamService`        | Unit        | 70%          | S9            |
| Integração (todas)   | Integration | 60%          | S2–S9         |
| E2E Fluxos Críticos  | E2E         | 5/5 passando | S10           |

---

## 🔀 Cross-Módulo

- Esta sprint não cria código novo de negócio — apenas valida o que foi construído nas sprints S1–S9.
- Qualquer falha de P0 (sistema inoperante) ou P1 (fluxo crítico quebrado) encontrada aqui DEVE ser corrigida antes da Sprint Final de Auditoria.
- O endpoint `POST /admin/workers/run/sla-checker` usado no E2E Fluxo 4 deve ser criado como endpoint de admin protegido por `ADMIN_RS` para permitir execução manual em testes e homologação.
- Os benchmarks de performance devem ser executados em ambiente staging (não local) para resultados válidos.
