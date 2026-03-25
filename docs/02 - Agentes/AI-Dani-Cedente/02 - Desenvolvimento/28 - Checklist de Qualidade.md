# 28 - Checklist de Qualidade — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| **Nome do Documento** | Checklist de Qualidade — AI-Dani-Cedente |
| **Versão** | v1.0 |
| **Data** | 23/03/2026 |
| **Autor** | Claude Code Desktop |
| **Status** | Rascunho |
| **Bloco** | 6 — Qualidade |
| **Dependências** | D02, D03, D06, D08, D09, D11, D12, D14, D16, D23, D25 |

---

> **📌 TL;DR**
>
> - **Gate de PR:** Partes 1 (Code Review) + 3 (Segurança) + 4 (Observabilidade) são verificadas em todo PR.
> - **Gate de Release:** Todas as 4 partes + tabela de bloqueantes + verificação pós-deploy.
> - **Itens bloqueantes:** 18 itens que impedem merge ou release se não confirmados.
> - **Segurança crítica para este produto:** `cedenteId` do contexto (nunca do LLM/request body), PII mascarado, JWT validado localmente.
> - **Acessibilidade:** WCAG 2.1 AA obrigatório (chat widget web + mobile). Verificado com axe-core + VoiceOver/TalkBack.
> - **Observabilidade:** Correlation ID em todo log, nenhum PII em logs, circuit breaker monitorado.

---

## 1. Como Usar o Checklist

| Momento | Partes Aplicáveis | Quem Preenche | Evidência Necessária |
|---|---|---|---|
| **Code Review (PR)** | Parte 1 + Parte 3 + Parte 4 | Revisor do PR | Check no template de PR |
| **Auditoria quinzenal** | Todas as partes | Tech Lead | Planilha de auditoria |
| **Pré-release (release/X.Y.Z)** | Todas as partes + Itens Bloqueantes | Tech Lead + QA | Checklist assinado no PR de release |
| **Pós-incidente P0/P1** | Parte 3 + Parte 4 | Tech Lead | Item no post-mortem |

**Como registrar:** No template de PR (D23), a seção "Checklist" referencia os itens bloqueantes. Para auditoria e release, usar este documento como formulário com data, responsável e evidência inline.

**Legenda:**
- `[ ]` Item pendente de verificação
- `[x]` Item verificado e aprovado
- `[N/A]` Não se aplica a este PR/release

---

## 2. Parte 1 — Code Review

### 2.1 Backend / NestJS

- [ ] **Sem `cedenteId` extraído de `req.body` ou input do LLM** — extraído exclusivamente do AsyncLocalStorage via `CedenteIsolationMiddleware`. _Evidência: grep por `req.body.cedenteId` ou `input.cedenteId` no diff._
- [ ] **Sem `any` TypeScript sem justificativa** — comentário explicando o motivo obrigatório se usado. _Evidência: `pnpm type-check` sem warnings de `any`._
- [ ] **Sem `console.log` esquecido** — apenas `logger` injetado do NestJS. _Evidência: grep `console\.log` no diff._
- [ ] **Error handling com `GlobalExceptionFilter`** — nenhuma exceção não tratada. Erros lançam classes derivadas de `DaniBaseError` (D20). _Evidência: handlers catch usam `throw new DaniXxxError()`._
- [ ] **Correlation ID propagado** — todo log inclui `correlation_id`. _Evidência: novo código que loga usa `logger.info({correlation_id: ...})`._
- [ ] **Nenhuma chamada síncrona de e-mail/notificação na request principal** — `NotificationService.publishNotification()` é fire-and-forget. _Evidência: sem `await notificationService` dentro de handlers HTTP._
- [ ] **Migrations backward-compatible** — nova coluna com `DEFAULT` ou nullable. Não renomear/remover coluna sem dois deploys. _Evidência: revisar arquivo de migration._
- [ ] **Rate limiting presente em novos endpoints** — endpoints de chat/agent usam `@RateLimit()` decorator.
- [ ] **RLS ativa para novas tabelas** — `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` e política por `cedente_id`. _Evidência: migration inclui `CREATE POLICY`._
- [ ] **Sem credenciais hardcoded** — verificar strings de conexão, tokens, chaves no diff. _Evidência: `git secrets --scan` ou grep por padrões de token._

### 2.2 Agente IA

- [ ] **`cedenteId` injetado pelo contexto nas ferramentas LangChain** — nunca pelo input do LLM. _Evidência: tools do agente usam `cedenteIsolationContext.getCedenteId()`._
- [ ] **Confirmação de ação financeira verificada** — accept/reject/counter de proposta e extensão de Escrow passam pelo estado `pendingConfirmation`. _Evidência: novo flow de ação financeira tem `state = 'pendingConfirmation'` antes de chamar endpoint._
- [ ] **System prompt não contém dados de Cedente hardcoded** — apenas templates com variáveis injetadas em runtime.
- [ ] **Timeout de ferramenta configurado** — toda nova ferramenta LangChain tem `timeout: 5000` e fallback documentado. _Evidência: definição da ferramenta no código._

### 2.3 API

- [ ] **Schema de resposta consistente com D16** — status codes, payload e headers conforme documentação de API. _Evidência: contrato Zod atualizado se interface mudou._
- [ ] **Paginação com objeto `meta`** — endpoints de listagem retornam `{ data: [], meta: { total, page, per_page } }`.
- [ ] **Endpoints Admin com RBAC correto** — rotas `/admin/*` têm `@RequireRole('admin')`. _Evidência: novo endpoint admin usa decorator._
- [ ] **Webhook ZapSign valida HMAC antes de processar** — `x-zapsign-hmac` validado antes de qualquer lógica de negócio.

### 2.4 Banco de Dados

- [ ] **Soft delete implementado** — entidades deletáveis usam `deleted_at` com `@SoftDelete()`. Nunca DELETE físico em entidades de negócio.
- [ ] **Índices para novas queries frequentes** — novas queries em colunas não indexadas têm índice na migration.
- [ ] **Sem query N+1** — uso de `include` ou `select` explícito no Prisma para relações. _Evidência: Prisma debug logs verificados localmente._

### 2.5 Performance

- [ ] **Cache Redis para dados consultados frequentemente** — lista de oportunidades, status de Escrow têm TTL configurado.
- [ ] **Sem processing síncrono pesado na request** — ingestão RAG, envio de e-mail e operações batch são async.

---

## 3. Parte 2 — Acessibilidade (WCAG 2.1 AA)

O chat widget da Dani-Cedente é embarcado na plataforma Repasse Seguro. Acessibilidade afeta diretamente Cedentes com deficiência visual ou motora.

### 3.1 Perceptível

- [ ] **Contraste de texto ≥ 4.5:1 (texto normal) e ≥ 3:1 (texto grande e elementos UI)** — verificado em modo claro e escuro. _Ferramenta: axe-core, Colour Contrast Analyser._
- [ ] **Imagens e ícones têm `alt` descritivo ou `aria-hidden="true"` se decorativo.** _Evidência: inspecionar elementos `<img>` no diff._
- [ ] **Cores não são o único meio de transmitir informação** — status de proposta tem texto além de cor.
- [ ] **Toast proativo anunciado via `aria-live="polite"` ou `aria-live="assertive"` (urgente)** — Cedente com screen reader recebe o aviso.

### 3.2 Operável

- [ ] **Todas as funcionalidades acessíveis via teclado** — Tab, Enter, Escape, setas funcionam no widget. _Evidência: teste manual com teclado._
- [ ] **Sem keyboard traps** — foco não fica preso em nenhum componente sem ação de saída.
- [ ] **Focus trap em modais de confirmação** — foco circula dentro do modal enquanto aberto. _Evidência: Modal de confirmação (T-06) testado com Tab._
- [ ] **Focus restoration ao fechar modal** — foco retorna ao elemento que abriu o modal.
- [ ] **Focus ring visível** — foco de teclado tem outline visível (não apenas `:focus-visible` com outline 0).
- [ ] **Tab order lógico** — ordem de tabulação segue fluxo visual da tela.
- [ ] **Skip link "Pular para conteúdo"** — presente e visível no primeiro Tab no widget.
- [ ] **`prefers-reduced-motion: reduce` respeitado** — animações de chat (bolhas, typing, cards) desabilitadas quando preferência ativa. _Referência: D04 Motion Spec._

### 3.3 Compreensível

- [ ] **Labels associados a inputs** — `<label for>` ou `aria-labelledby`. Sem labels flutuantes sem associação.
- [ ] **Mensagens de erro identificam o campo** — "CPF inválido" específico, não "Campo inválido".
- [ ] **Idioma do documento declarado** — `<html lang="pt-BR">`.

### 3.4 Robusto (ARIA)

- [ ] **Landmarks ARIA presentes** — `<main>`, `<nav>`, `<aside>` no widget onde aplicável.
- [ ] **`aria-label` ou `aria-labelledby` em regiões sem título visível.**
- [ ] **Estados dinâmicos anunciados** — loading, error, success em componentes interativos usam `aria-busy`, `aria-invalid` ou `role="status"`.
- [ ] **Tabelas têm `<th>` com `scope`** — tabelas de análise de proposta nos cards inline.

### 3.5 Mobile

- [ ] **Touch targets ≥ 44×44px** — botões de CTA, ícones e elementos interativos. _Referência: D11, WCAG 2.5.5._
- [ ] **VoiceOver (iOS) e TalkBack (Android) testados** — fluxo principal do chat funciona com screen reader. _Evidência: teste manual._
- [ ] **`accessibilityLabel` definido em componentes React Native** — para Fase 2, todo componente interativo tem label.

---

## 4. Parte 3 — Segurança

### 4.1 Autenticação e Autorização

- [ ] **JWT validado localmente com RS256** — sem chamada HTTP ao Supabase por validação. _Evidência: `JwtAuthGuard` usa `jwt.verify()` com chave pública local._
- [ ] **`JwtAuthGuard` presente em todos os endpoints autenticados** — sem endpoint sem `@UseGuards(JwtAuthGuard)`. _Evidência: grep por `@Controller` sem `@UseGuards`._
- [ ] **RBAC com `@RequireRole()`** — rotas Admin têm `@RequireRole('admin')`, rotas Cedente têm `@RequireRole('cedente')`.
- [ ] **404 (não 403) para acesso cruzado entre Cedentes** — verificar `CedenteIsolationMiddleware` retorna 404. _Evidência: teste de integração de isolamento._
- [ ] **`cedenteId` extraído do JWT via AsyncLocalStorage** — nunca de query param, body ou input do LLM.

### 4.2 PII e LGPD

- [ ] **`PiiMaskingMiddleware` ativo em todas as rotas** — CPF, e-mail, telefone mascarados antes de qualquer log ou resposta.
- [ ] **Nenhum PII em payload de fila RabbitMQ** — e-mail do Cedente não trafega em mensagens de notificação.
- [ ] **Nenhum PII em logs** — novo código não inclui email, cpf, telefone, nome completo em `metadata` de log.
- [ ] **Traces Langfuse sem PII** — conteúdo de chat e dados de Cedente mascarados antes de enviar ao Langfuse.

### 4.3 Validação de Input

- [ ] **DTOs com class-validator** — todo body de request tem DTO com decorators de validação. _Evidência: novo endpoint tem `@Body() dto: CreateXxxDto` com `@IsString()`, `@IsUUID()`, etc._
- [ ] **Sem SQL raw com interpolação de variável não sanitizada** — queries Prisma usam parâmetros. Raw queries usam `Prisma.sql` template.
- [ ] **Webhook HMAC validado antes de processar** — ZapSign webhook rejeita com 400 antes de executar qualquer lógica.

### 4.4 Secrets e Transporte

- [ ] **Nenhuma credencial hardcoded no código** — verificar `process.env.` para todos os valores sensíveis.
- [ ] **`.env` não commitado** — `.gitignore` inclui `.env*`. _Evidência: `git status --porcelain | grep .env`._
- [ ] **HTTPS em todos os endpoints de produção** — Railway configura TLS automaticamente, verificar.
- [ ] **Headers de segurança configurados** — `Helmet` ativo no NestJS: `X-Frame-Options`, `X-Content-Type-Options`, `Strict-Transport-Security`.

### 4.5 Rate Limiting

- [ ] **Rate limit no chat: 30 msg/h por `cedente_id`** — Redis sliding window. _Evidência: teste de integração RB-001 em D27._
- [ ] **Rate limit na API: 100 req/min por IP** — `@nestjs/throttler` configurado.

### 4.6 Dependências

- [ ] **Sem vulnerabilidades CRITICAL nas dependências** — `pnpm audit` ou Snyk limpo. _Evidência: relatório do CI._
- [ ] **Dependências desatualizadas com HIGH CVE sem nota de aceitação de risco** — ticket aberto se não for corrigido imediatamente.

---

## 5. Parte 4 — Observabilidade e Operação

- [ ] **Correlation ID em todos os logs do novo código** — `logger.info({ correlation_id: ctx.correlationId, ... })`.
- [ ] **Nenhum `console.log` ou `console.error`** — apenas `logger` injetado. _Evidência: grep._
- [ ] **Stack trace presente em erros ERROR mas não na resposta HTTP** — `GlobalExceptionFilter` não expõe stack ao cliente.
- [ ] **Novo evento de negócio relevante logado em INFO** — ex: proposta aceita, notificação crítica enviada.
- [ ] **Nova métrica registrada para novos fluxos críticos** — counter ou histogram no Prometheus para novos fluxos adicionados.
- [ ] **Health check atualizado** — se nova dependência crítica adicionada, incluída no `/health/ready`.
- [ ] **Alerta em D25 atualizado** — se novo threshold de erro foi definido para o fluxo adicionado.
- [ ] **Smoke test pós-deploy cobre o novo fluxo** — se é funcionalidade crítica, checklist de smoke test do D24 é atualizado.

---

## 6. Itens Bloqueantes

Estes itens impedem merge de PR ou release se não forem confirmados:

| # | Categoria | Item | Por que Bloqueia | Como Verificar | Impacto se Ignorado |
|---|---|---|---|---|---|
| B-001 | Segurança | `cedenteId` extraído do AsyncLocalStorage, não do request/LLM | Acesso cruzado entre Cedentes | Grep + teste de integração de isolamento | Cedente A acessa dados de Cedente B |
| B-002 | Segurança | JWT validado localmente com RS256 | Autenticação contornada | Verificar `JwtAuthGuard.canActivate()` | Acesso sem autenticação |
| B-003 | Segurança | Sem credencial hardcoded | Credencial em repositório público | `git secrets --scan` ou grep | Comprometimento de conta/API |
| B-004 | Segurança | Sem PII em logs | Violação LGPD | Grep por campos PII no diff | Multa ANPD, dano à reputação |
| B-005 | Segurança | HMAC ZapSign validado antes de processar | Processamento de evento forjado | Teste de integração com HMAC inválido | Manipulação de Escrow |
| B-006 | Agente IA | Confirmação obrigatória antes de ação financeira | Ação executada sem consentimento do Cedente | Teste E2E de confirmação | Aceite de proposta sem intenção do Cedente |
| B-007 | Backend | Migrations backward-compatible | Crash durante rolling deploy | Revisar migration manualmente | Downtime em produção |
| B-008 | Backend | DTOs com validação em todos os body inputs | Injeção de dados inválidos | `pnpm type-check` + revisão de DTO | Erros 500 imprevistos |
| B-009 | Testes | Testes para módulo crítico (agent, auth, escrow, fallback) | Regressão não detectada | Cobertura ≥ 90% via coverage report | Bug em funcionalidade crítica |
| B-010 | Testes | CI verde (lint + type-check + testes + build) | Pipeline quebrado | GitHub Actions check | Código com bugs em produção |
| B-011 | Observabilidade | Correlation ID em logs de novo código | Incidente sem rastreabilidade | Revisar chamadas de logger | Diagnóstico impossível em produção |
| B-012 | Acessibilidade | Contraste ≥ 4.5:1 em novo componente de UI | Violação WCAG 2.1 AA | axe-core no diff | Inacessível para usuários com baixa visão |
| B-013 | Acessibilidade | Focus trap em novo modal/overlay | Keyboard trap — inacessível via teclado | Teste manual com Tab | Usuário preso em overlay sem saída |
| B-014 | API | Schema de resposta consistente com D16 | Contrato quebrado | Teste de contrato Zod | Cliente quebrando em produção |
| B-015 | Banco | RLS ativa em nova tabela com dados de Cedente | Cedente acessa dados alheios | Verificar `pg_tables.rowsecurity` | Violação de isolamento |
| B-016 | Segurança | Sem vulnerabilidade CRITICAL nas dependências | Dependência vulnerável em produção | `pnpm audit` no CI | Exploração de CVE |
| B-017 | Notificações | Notificação assíncrona (fire-and-forget via fila) | Envio síncrono bloqueia request | Verificar que `publishNotification` não é awaited no handler | Latência de API aumentada |
| B-018 | Segurança | Endpoints Admin com `@RequireRole('admin')` | Escalada de privilégio | Grep por `@Controller` de admin sem RBAC | Cedente acessa Admin |

---

## 7. Ferramentas de Apoio

| Ferramenta | Categoria | O que Valida | Quando Usar | Limitação |
|---|---|---|---|---|
| **ESLint + Prettier** | Code Review | Estilo, padrões de código, regras customizadas | Pre-commit + CI | Não detecta lógica de negócio incorreta |
| **TypeScript strict** | Code Review | Tipagem, null-safety | Pre-commit + CI | Não detecta bugs de runtime |
| **Jest + ts-jest** | Testes | Cobertura de código | CI todo PR | Não garante qualidade de teste |
| **axe-core** | Acessibilidade | Violations WCAG automatizáveis (30% do total) | PR + auditoria | Não substitui teste manual com screen reader |
| **Lighthouse** | Acessibilidade + Performance | Score geral, contraste, performance | Auditoria periódica | Ambiente de teste pode divergir de produção |
| **VoiceOver (iOS)** | Acessibilidade | Screen reader flow completo | Pre-release | Requer dispositivo Apple |
| **TalkBack (Android)** | Acessibilidade | Screen reader Android | Pre-release | Requer dispositivo Android |
| **Pino redact** | Segurança + Observabilidade | PII mascarado em logs | Automático — configurado no logger | Campos novos precisam ser adicionados manualmente |
| **Snyk / `pnpm audit`** | Segurança | CVEs em dependências | CI todo PR + semanal | Falsos positivos em CVEs de baixa severidade |
| **Postman / Insomnia** | API | Schema de resposta manual | Pre-release + debug | Manual — não substitui testes de contrato |
| **Colour Contrast Analyser** | Acessibilidade | Ratio de contraste exato | Design review | Apenas cores, não contexto de UX |

---

## 8. Mapeamento por Categoria

| Categoria | Referência Interna | Padrão Externo |
|---|---|---|
| Isolamento de Cedente | D01 RN-DCE-001, D18 | OWASP A01 (Broken Access Control) |
| Autenticação JWT | D18, D14 | OWASP A07 (Authentication Failures) |
| Validação de Input | D16, D20 | OWASP A03 (Injection) |
| PII e LGPD | D18, D21, D25 | LGPD Art. 46, OWASP A02 |
| Rate Limiting | D14, D16 | OWASP A04 (Insecure Design) |
| Acessibilidade Web | D03, D06, D07, D09 | WCAG 2.1 AA |
| Acessibilidade Mobile | D11 | WCAG 2.5.5, iOS HIG, Android Accessibility |
| Observabilidade | D25 | DORA Metrics, SRE Golden Signals |
| Testes | D27 | Test Pyramid (Fowler) |
| Commits e PR | D23 | Conventional Commits v1.0 |
| Deploy e CI | D24 | DORA (Change Failure Rate, Lead Time) |

---

## 9. Periodicidade de Auditoria

| Frequência | O que Revisar | Onde Olhar | Ação se Não OK |
|---|---|---|---|
| **Todo PR** | Itens Bloqueantes B-001 a B-018 | Template de PR preenchido | Bloquear merge |
| **Quinzenal** | Todas as 4 partes | Este checklist + logs de auditoria anteriores | Abrir ticket P2 por item fora do padrão |
| **Pré-release** | Todas as 4 partes + Itens Bloqueantes | Checar cada item com evidência | Bloquear release se qualquer item bloqueante não confirmado |
| **Pós-incidente P0/P1** | Parte 3 (Segurança) + Parte 4 (Observabilidade) | Correlation IDs do incidente + logs | Ação corretiva no post-mortem |

---

## 10. Backlog de Pendências

| # | Item | Tipo | Prioridade |
|---|---|---|---|
| P-CHK-001 | Configurar axe-core como teste de acessibilidade automatizado no CI para componentes do chat widget | [SEÇÃO PENDENTE] — requer setup de ambiente de testes de UI | Alta |
| P-CHK-002 | Criar ambiente de smoke test de acessibilidade com VoiceOver/TalkBack antes de cada release | [SEÇÃO PENDENTE] — requer dispositivos físicos ou emuladores no pipeline | Média |
| P-CHK-003 | Adicionar `git secrets` ou similar ao pre-commit hook para detectar credenciais hardcoded | [DECISÃO AUTÔNOMA: usar detect-secrets da Yelp] — configurar antes do Go-Live | Alta |
| P-CHK-004 | Formalizar planilha de auditoria quinzenal com histórico de itens aprovados/rejeitados | [SEÇÃO PENDENTE] — depende de acordo do time sobre ferramenta (Notion, GitHub Issues) | Baixa |

---

## 11. Changelog do Documento

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial — 4 partes (code review 22 itens, acessibilidade 18 itens, segurança 18 itens, observabilidade 8 itens). 18 itens bloqueantes tabulados com evidência e impacto. 11 ferramentas de apoio. Mapeamento para OWASP, WCAG, LGPD. Periodicidade de auditoria por frequência. |
