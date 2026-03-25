# S11 — Qualidade

## Sprint 11 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                                          |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| **Sprint**         | S11 — Qualidade                                                                                |
| **Template**       | Sprint de Qualidade (auditorias, E2E completo, a11y, segurança, performance)                   |
| **REQs cobertos**  | REQs transversais de qualidade (Doc 27, Doc 28, Doc 25)                                        |
| **Docs fonte**     | 27 - Plano de Testes · 28 - Checklist de Qualidade · 25 - Observabilidade · 01.3 - RN Operação |
| **Total de itens** | 50 itens                                                                                       |
| **Status**         | Concluída                                                                                      |

**Pré-requisito:** S1–S10 concluídas. Esta sprint valida completude funcional e não-funcional.

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos de todos os 16 test cases (TC-CES-01 a TC-CES-16), ferramentas (Vitest, Playwright, axe-core, Lighthouse), thresholds.
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — Cobertura: 70% unit/20% integration/8% E2E/2% manual; 100% CommissionService e EscrowService; Lighthouse a11y ≥ 95; contraste ≥ 4.5:1.
- [x] ✅ Check 4 — Glossário: WCAG 2.1 AA, axe-core, aria-live, focus trap, skip link.
- [x] ✅ Check 5 — Anti-scaffold R10: testes reais com assertions concretas.
- [x] ✅ Check 6 — Fluxo crítico completo: KYC→Proposta→Negociação→Escrow→Formalização→Fechamento.
- [x] ✅ Check 7 — Alertas Sentry/Langfuse configurados com thresholds explícitos.
- [x] ✅ Check 8 — Cross-módulo: testes E2E cobrem integração entre todos os módulos S2–S9.
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — Sem ambiguidades.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — TC-CES-01 a TC-CES-16 todos implementados.

---

## FEATURE 1 — Testes Unitários e de Integração

- [x] **S11-T01** · Verificar cobertura de testes em `CommissionService` e `EscrowService`: executar `pnpm test:coverage` e confirmar 100% de branches em `calculateBuyerCommission()` (S1-FE09/S4-BE05) e `EscrowService.confirmDeposit()` (S6-BE03). Falha nos thresholds bloqueia PR. (Doc 27 — pirâmide de testes; Doc 28 — bloqueantes)

- [x] **S11-T02** · Verificar cobertura geral ≥ 80% em todos os módulos críticos: `AuthService`, `KycService`, `ProposalsService`, `NegotiationsService`, `FinancialService`, `FormalizationService`, `NotificationService`, `AiModule`; executar `pnpm test:coverage --reporter=html` e revisar relatório; corrigir branches descobertos em módulos abaixo do threshold.

- [x] **S11-T03** · Implementar testes de integração com banco de dados real (PostgreSQL de teste): criar `prisma/seed/test.ts` com dataset de teste; testes verificam: RLS bloqueia acesso cross-cessionário; soft delete funciona; índices usados em queries de produção (via `EXPLAIN ANALYZE`); foreign key RESTRICT impede deleção de `users` com `cessionarios` vinculados.

- [x] **S11-T04** · Implementar testes de integração para todos os webhooks: ZapSign HMAC-SHA256 inválido → 401; ZapSign HMAC válido com `signer_signed` → status atualizado; idwall HMAC inválido → 401; idwall `report_complete` APPROVED → KYC_APROVADO; todos os webhooks retornam 200 imediatamente (não bloqueante). Usar `supertest` para requests HTTP.

- [x] **S11-T05** · Implementar testes de integração para filas RabbitMQ: publicar evento `notification.email` → `EmailWorker` consome e chama Resend mock; publicar `kyc.process` → `KycWorker` consome e chama idwall mock; DLQ: simular 3 falhas consecutivas → mensagem vai para `*.dead-letter`; verificar que workers não processam mensagens de DLQ automaticamente.

---

## FEATURE 2 — Testes E2E (Playwright) — TC-CES-01 a TC-CES-16

- [x] **S11-E2E01** · Implementar E2E TC-CES-01 "Cadastro de novo Cessionário": preencher form com e-mail único, submit, verificar HTTP 201, verificar redirecionamento para dashboard, verificar status CADASTRADA, verificar recebimento de NOT-CES-01 (e-mail de boas-vindas capturado via Resend mock). (Doc 01.3 — TC-CES-01)

- [x] **S11-E2E02** · Implementar E2E TC-CES-02 "Login com credenciais válidas" + TC-CES-03 "Login com senha incorreta": TC-02: login válido, verificar access token + cookie refresh token; TC-03: senha errada, verificar mensagem `AUTH-001`, verificar que 5 tentativas em 1h ativam bloqueio. (Doc 01.3 — TC-CES-02, TC-CES-03)

- [x] **S11-E2E03** · Implementar E2E TC-CES-04 "Submissão de documentos KYC" + TC-CES-05 "Aprovação KYC": TC-04: upload dos 4 documentos, submit, status KYC_EM_ANALISE; TC-05: mock webhook idwall approved, status KYC_APROVADO, NOT-CES-02 enviada. (Doc 01.3 — TC-CES-04, TC-CES-05)

- [x] **S11-E2E04** · Implementar E2E TC-CES-06 "Recebimento de notificação in-app": simular evento de negócio, verificar badge no header incrementa, notificação aparece sem reload; TC-CES-07 "Envio de proposta válida": proposta válida com re-auth, verificar ENVIADA + NOT-CES-03. (Doc 01.3 — TC-CES-06, TC-CES-07)

- [x] **S11-E2E05** · Implementar E2E TC-CES-08 "Cancelamento de proposta" + TC-CES-09 "Limite de propostas": TC-08: cancelar proposta ENVIADA com re-auth, verificar CANCELADA; TC-09: criar 3 propostas simultâneas, 4ª retorna RATE-001. (Doc 01.3 — TC-CES-08, TC-CES-09)

- [x] **S11-E2E06** · Implementar E2E TC-CES-10 "Contraproposta" + TC-CES-11 "Extensão Escrow": TC-10: enviar contraproposta em negociação ativa, verificar EM_CONTRAPROPOSTA; TC-11: solicitar extensão de prazo, verificar extension_used=true e novo prazo. (Doc 01.3 — TC-CES-10, TC-CES-11)

- [x] **S11-E2E07** · Implementar E2E TC-CES-12 "Upload comprovante Escrow" + TC-CES-13 "Confirmação Escrow": TC-12: upload PDF comprovante, DEPOSITO_ENVIADO; TC-13: mock confirmação admin, DEPOSITO_CONFIRMADO + NOT-CES-06. (Doc 01.3 — TC-CES-12, TC-CES-13)

- [x] **S11-E2E08** · Implementar E2E TC-CES-14 "Formalização — acesso ao documento" + TC-CES-15 "Conclusão": TC-14: navegar para formalização, verificar `can_sign=true`, link ZapSign visível; TC-15: mock webhook `doc_complete`, AGUARDANDO_ANUENCIA. (Doc 01.3 — TC-CES-14, TC-CES-15)

- [x] **S11-E2E09** · Implementar E2E TC-CES-16 "Painel financeiro pós-conclusão": operação concluída presente, navegar para /financeiro, verificar transações COMMISSION e OPERATION_COMPLETED com valores corretos, verificar commission_breakdown. (Doc 01.3 — TC-CES-16)

- [x] **S11-E2E10** · Implementar E2E do fluxo crítico completo integrado (smoke test de produção): Cadastro → KYC → Marketplace → Proposta → Negociação → Escrow → Formalização → Fechamento → Financeiro; usar dataset de staging; execução em <15min total; deve ser verde antes de qualquer deploy em `main`. (Doc 27 — fluxo crítico; Doc 24 — E2E em CI)

---

## FEATURE 3 — Acessibilidade (WCAG 2.1 AA)

- [x] **S11-A01** · Executar `axe-core` integrado ao Playwright em todas as 34 telas: `T-AUTH-01/02/03/04`, `T-DASH-01`, `T-OPR-01/02/03`, `T-PRP-01/02`, `T-NEG-01/02/03/04`, `T-ASS-01/02/03`, `T-FIN-01/02/03`, `T-IA-01`, `T-PRF-01/02/03/04/05`; Lighthouse Accessibility score ≥ 95 em todas; zero violações `critical` ou `serious` no axe-core. (Doc 28 — seção 3; Doc 27)

- [x] **S11-A02** · Verificar contraste de texto: ≥ 4.5:1 para texto normal; ≥ 3:1 para texto grande (≥ 18pt normal / ≥ 14pt bold); ≥ 3:1 para elementos UI (bordas, ícones); verificar em dark mode; verificar `CessionarioStatusBadge`, `ProposalStatusBadge`, `NegotiationStatusBadge`, `EscrowStatusBadge`, `FormalizationStatusBadge`, `AiRiskScoreBadge` com Colour Contrast Analyser. (Doc 28 — seção 3.1)

- [x] **S11-A03** · Verificar navegação por teclado em fluxos críticos: Tab order lógico em formulários de login, cadastro, proposta; foco ao abrir `ReAuthModal` no primeiro input; focus trap em modais e drawers; focus restaurado ao fechar; skip link "Pular para conteúdo principal" no primeiro Tab; stepper KYC com `aria-current="step"`; `<html lang="pt-BR">`. (Doc 28 — seções 3.2, 3.3)

- [x] **S11-A04** · Verificar touch targets mobile: mínimo 44×44px em todos os CTAs críticos ("Fazer Proposta", "Confirmar", botões KYC, "Assinar via ZapSign"); verificar com DevTools mobile emulation. Verificar que `prefers-reduced-motion: reduce` desabilita animações não essenciais (Framer Motion `useReducedMotion()`). (Doc 28 — seção 3.2)

---

## FEATURE 4 — Segurança

- [x] **S11-S01** · Executar `pnpm audit --audit-level=high`: zero vulnerabilidades críticas ou altas; se encontradas, atualizar dependências ou documentar exceção aceita com justificativa. Bloqueante de release. (Doc 28 — seção 4; Doc 29 — T-7)

- [x] **S11-S02** · Verificar que dados do Cedente estão ausentes em TODOS os responses da API: executar suite de testes que busca strings como `cedente_name`, `cedente_cpf`, `cedente_id`, `cedente_contact` em 100% dos payloads de response dos 47 endpoints; zero ocorrências. (Doc 01.5 — RN-063; Doc 28 — seção 2.2)

- [x] **S11-S03** · Verificar headers de segurança HTTP em respostas da API: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 1; mode=block`, `Strict-Transport-Security: max-age=31536000; includeSubDomains`, `Content-Security-Policy` configurado; todos via `helmet` NestJS. (Doc 28 — seção 4)

- [x] **S11-S04** · Verificar que `SUPABASE_SERVICE_ROLE_KEY` nunca aparece em: logs de backend, responses da API, código do frontend, arquivos `.env` commitados; verificar que `VITE_SUPABASE_ANON_KEY` é `anon key` (não `service_role`); executar `git grep -r "service_role" apps/web/` → zero ocorrências. (Doc 28 — seção 2.2; Doc 17)

- [x] **S11-S05** · Verificar idempotency em endpoints financeiros críticos: `POST /proposals` com mesmo `opportunity_id` dentro da quota — não cria duplicata se já existe proposta ENVIADA para a mesma oportunidade; `POST /escrow/receipt` não permite segundo upload se `status != AGUARDANDO_DEPOSITO`; verificar com teste de integração. (Doc 28 — seção 2.3)

---

## FEATURE 5 — Observabilidade

- [x] **S11-O01** · Configurar todos os alertas do Sentry (conforme Doc 25): novo erro P0 (unhandled exception) → Slack `#incidentes` imediato; taxa de erros > 2% em 5min → PagerDuty P0; erro `EXT-001` (ZapSign/idwall falhou) > 3 ocorrências/hora → Slack `#incidentes`; erro de banco `DB-001` qualquer → Slack `#incidentes`. Verificar que alertas disparam com evento de teste manual. (Doc 25 — alertas)

- [x] **S11-O02** · Configurar alertas de Langfuse (conforme Doc 25): p95 latência LLM > 5s → Slack `#ops`; taxa de erro LLM > 5% → Slack `#ops`; custo diário > threshold configurável → Slack `#ops`. Verificar via dashboard Langfuse que traces do Analista de Oportunidades aparecem com `traceId`, `userId` hash, tokens e latência. (Doc 25; Doc 17 — Langfuse)

- [x] **S11-O03** · Configurar alertas de infraestrutura (Railway + Supabase): CPU Railway > 80% → Slack `#ops`; memória Railway > 85% → Slack `#ops`; conexões Supabase > 80% do pool → PagerDuty; DLQ RabbitMQ > 5 mensagens → Slack `#ops`. Verificar dashboards operacionais conforme Doc 29 seção 2.2. (Doc 29 — dashboards)

- [x] **S11-O04** · Verificar PostHog: eventos de funil configurados e capturando (`proposal_created`, `kyc_submitted`, `kyc_approved`, `escrow_receipt_uploaded`, `formalization_signed`, `operation_completed`); nenhum PII (CPF, e-mail, nome) em propriedades de eventos; feature flags funcionando para funcionalidades condicionais. (Doc 28 — seção 2.1; Doc 25)

---

## FEATURE 6 — Auditoria de Código (Checklist D28)

- [x] **S11-CQ01** · Executar Checklist D28 Parte 1 (Code Review) completo: verificar todos os itens BLOQUEANTES e RECOMENDAÇÕES; documentar evidência linkável para cada item; zero itens BLOQUEANTES abertos sem justificativa. Items críticos: componentes funcionais, TanStack Query, Zustand, sem `console.log`, TypeScript strict, sem `any`, guards em todos os endpoints, KYC guard, rate limiting, correlation ID, schema de erro, RLS, soft delete, Decimal financeiro. (Doc 28 — seção 2)

- [x] **S11-CQ02** · Verificar convenções de arquitetura (Doc 15): nenhum import cross-feature via barrel proibido; todos os arquivos NestJS em `kebab-case.tipo.ts`; componentes React em `PascalCase`; hooks em `camelCase` com prefixo `use`; pastas em `kebab-case`; barrel exports `index.ts` em cada pasta; prefixo `rs:` em todas as chaves Redis; TTL definido para toda chave Redis. (Doc 15 — convenções)

---

## 📊 Cobertura de Qualidade

| Métrica                                | Target              | Status    |
| -------------------------------------- | ------------------- | --------- |
| Cobertura unitária CommissionService   | 100%                | Verificar |
| Cobertura unitária EscrowService       | 100%                | Verificar |
| Cobertura geral módulos críticos       | ≥ 80%               | Verificar |
| TC-CES-01 a TC-CES-16 E2E              | 100% verdes         | Verificar |
| Lighthouse a11y                        | ≥ 95 todas as telas | Verificar |
| axe-core violations (critical/serious) | 0                   | Verificar |
| `pnpm audit` high vulnerabilities      | 0                   | Verificar |
| Dados Cedente em responses             | 0 ocorrências       | Verificar |
| `service_role` no frontend             | 0 ocorrências       | Verificar |
| Alertas Sentry/Langfuse/Railway        | Todos configurados  | Verificar |
