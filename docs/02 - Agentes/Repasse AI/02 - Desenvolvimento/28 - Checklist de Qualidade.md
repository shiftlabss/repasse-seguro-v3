# 28 - Checklist de Qualidade

## Cabeçalho

| **Nome do Documento** | **Versão** | **Data** | **Autor** | **Status** |
| --- | --- | --- | --- | --- |
| 28 - Checklist de Qualidade | v1.0 | 2026-03-22 | Claude Code Desktop (ShiftLabs Pipeline v9.5) | Aprovado |

---

## TL;DR

> 📌 **Gate de qualidade do Repasse AI:**
>
> - **4 partes:** Code Review (backend), Acessibilidade (Webchat PWA), Segurança, Observabilidade e Operação.
> - **Contexto PG-03:** Repasse AI é backend puro — checklist de frontend cobre apenas o Webchat embeddado (componente na plataforma Cessionário) e a experiência mobile PWA, não uma SPA autônoma.
> - **Itens bloqueantes:** 18 itens identificados que impedem merge em `main` ou promoção a produção.
> - **Aplicado em:** PR (code review), auditoria quinzenal, pré-release e pós-incidente.
> - **Ferramentas principais:** ESLint + `@typescript-eslint`, Jest coverage, Sentry, axe-core, `pnpm audit`.

---

## 1. Como Usar o Checklist

| Momento | Quem Preenche | Itens Obrigatórios | Evidência |
| --- | --- | --- | --- |
| **PR (pre-merge)** | Author + Reviewer | Partes 1 e 3 (bloqueantes) | GitHub PR checklist |
| **Auditoria quinzenal** | Tech Lead | Todas as partes | Notion — página de auditoria |
| **Pré-release** | Tech Lead + QA | Todas as partes | Notion — checklist de go-live |
| **Pós-incidente** | DRI do incidente | Parte 3 (segurança) + Parte 4 | Post-mortem (Doc 26) |

**Como registrar evidência:** cada item `[ ]` deve ser marcado `[x]` com comentário de evidência inline no PR ou no documento de auditoria. Itens com `[N/A]` requerem justificativa escrita.

---

## 2. Parte 1 — Code Review (Backend)

### 2.1 Geral

- [ ] **[BLOQUEANTE]** Nenhum `throw new Error()` genérico em módulos de domínio — usar subclasse de `BaseError`
- [ ] **[BLOQUEANTE]** Nenhum dado sensível (JWT, OTP, telefone, CPF) em logs — verificar via `PiiMaskingInterceptor`
- [ ] **[BLOQUEANTE]** `correlation_id` propagado em todos os logs, erros e eventos SSE do PR
- [ ] Código segue nomenclatura da Doc 15 (kebab-case arquivos, PascalCase classes, camelCase vars)
- [ ] Nenhuma query de banco sem `withTenant()` em módulos de domínio
- [ ] `async/await` usado corretamente — sem Promises não tratadas
- [ ] Sem `any` explícito em TypeScript sem justificativa no comentário
- [ ] Sem `console.log` em código de produção — usar `Logger` do NestJS/Pino

### 2.2 Módulo de IA

- [ ] **[BLOQUEANTE]** Tool calls não usam `parallel_tool_calls: true` (sequencial obrigatório — Doc 19)
- [ ] Guardrails pré-LLM chamados antes de invocar LangGraph (`PiiGuard`, `TenantGuard`, etc.)
- [ ] `semantic_cache_service.get()` chamado antes de invocar LLM
- [ ] Circuit breaker verificado antes de chamar LLM — não chamar quando `OPEN`
- [ ] Temperatura LLM `0.1` conforme Doc 19 — sem override não documentado
- [ ] Prompt version registrado em `AgentConfiguration` — não hardcoded

### 2.3 API e Contratos

- [ ] **[BLOQUEANTE]** Todo endpoint novo tem guard de autenticação (`@UseGuards(JwtAuthGuard)`)
- [ ] Todo endpoint administrativo tem guard de role (`@UseGuards(AdminGuard)`)
- [ ] DTOs têm validação class-validator completa — nenhum campo sem decorator
- [ ] Response schema segue padrão da Doc 16 (paginação, error schema, SSE format)
- [ ] Rate limiting aplicado em endpoints de chat e WhatsApp (Doc 16)
- [ ] Novos endpoints documentados em `16 - Documentação de API.md`

### 2.4 Banco de Dados

- [ ] **[BLOQUEANTE]** Nenhuma migration com DROP de coluna em uso sem etapa de migração em dois passos
- [ ] Migration nova é aditiva e backward-compatible
- [ ] Índices criados para novas colunas usadas em filtros/sorts frequentes
- [ ] Soft delete respeitado — sem `DELETE` físico em tabelas de domínio PII
- [ ] `created_at`/`updated_at` presentes em novas entidades

### 2.5 Testes

- [ ] **[BLOQUEANTE]** Cobertura do módulo modificado ≥ threshold definido no `jest.config.ts`
- [ ] Novos caminhos de erro têm testes unitários (não só happy path)
- [ ] Mocks usados corretamente — sem over-mocking de lógica interna
- [ ] Nenhum dado real (email, telefone, CPF) em factories de teste

### 2.6 Notificações

- [ ] Novos templates adicionados na tabela de seeds `AgentConfiguration` — não hardcoded no código
- [ ] Notificações críticas não têm opt-out removível — verificar tabela NT-XXX da Doc 21
- [ ] Envio de notificação usa `NotificationService.sendNotification()` — não direto ao canal

---

## 3. Parte 2 — Acessibilidade (WCAG 2.1 AA — Webchat PWA)

> ⚙️ **Contexto:** PG-03 — Repasse AI é backend puro. O Webchat é um componente embeddado na plataforma Cessionário (Next.js). Checklist de acessibilidade cobre os componentes do Webchat e a experiência mobile PWA. Avaliação de acessibilidade feita pelo time da plataforma Cessionário ao integrar o Webchat; os contratos de acessibilidade são definidos aqui.

### 3.1 Perceptível

- [ ] **[BLOQUEANTE]** Contraste de texto normal ≥ 4.5:1 (WCAG 1.4.3) — verificado com axe-core + Lighthouse
- [ ] **[BLOQUEANTE]** Contraste de texto grande (≥ 18pt ou 14pt bold) ≥ 3:1
- [ ] **[BLOQUEANTE]** Contraste de elementos de UI (botões, inputs, ícones) ≥ 3:1
- [ ] Contraste validado em dark mode também — não apenas light mode
- [ ] Imagens informativas têm `alt` text descritivo; imagens decorativas têm `alt=""`
- [ ] ChatBubble tem indicador visual de estado além de cor (ícone ou texto) — não depende só de cor
- [ ] StatusBadge tem label textual além da cor para cada estado
- [ ] Conteúdo de vídeo ou áudio (se aplicável) tem transcrição ou closed captions

### 3.2 Operável

- [ ] **[BLOQUEANTE]** Todas as funcionalidades do Webchat acessíveis via teclado (Tab, Enter, Escape, setas)
- [ ] Sem keyboard trap — pressionar Escape sai de qualquer modal ou drawer
- [ ] Tab order lógico e sequencial — verificar com inspeção manual via Tab
- [ ] Modais e drawers têm focus trap (Tab circula apenas dentro do overlay enquanto aberto)
- [ ] Focus restoration ao fechar overlay — foco retorna ao elemento que abriu
- [ ] Visible focus ring em todos os elementos interativos — não removido via `outline: none` sem substituto
- [ ] **[BLOQUEANTE]** Touch targets mínimos 44×44px (WCAG 2.5.5) — botões do Webchat e inputs
- [ ] Swipe gestures têm alternativa via teclado ou botão

### 3.3 Compreensível

- [ ] Formulários têm `label` associado a cada `input` via `for`/`id` ou `aria-labelledby`
- [ ] Erros de validação descrevem o problema e como corrigir — não apenas "campo inválido"
- [ ] Mensagens de loading anunciadas por screen reader via `aria-live="polite"`
- [ ] Estados de empty state e error têm texto descritivo (Doc 08 UX Writing)
- [ ] Toast notifications anunciadas via `aria-live="assertive"` (para erros críticos) ou `polite`

### 3.4 Robusto

- [ ] **[BLOQUEANTE]** ARIA roles semânticos corretos: `main`, `nav`, `aside`, `dialog`, `alert`, `status`
- [ ] `aria-label` ou `aria-labelledby` em elementos sem texto visível (ícone-botões)
- [ ] Tabelas com `<th scope>` correto quando aplicável
- [ ] Skip link "Pular para conteúdo principal" visível no primeiro Tab da página
- [ ] `prefers-reduced-motion: reduce` respeitado — animações não essenciais desabilitadas
- [ ] Validação com axe-core (0 violations críticas) e Lighthouse Accessibility ≥ 90

### 3.5 Mobile PWA Específico

- [ ] Safe area `env(safe-area-inset-*)` aplicada em telas fullscreen mobile
- [ ] Teclado virtual não cobre o input de mensagem — `viewport` ou scroll compensado
- [ ] iOS VoiceOver e Android TalkBack testados nos fluxos principais (envio de mensagem, OTP)
- [ ] Push notification com `aria-describedby` apontando para conteúdo relevante quando aplicável

---

## 4. Parte 3 — Segurança

### 4.1 Autenticação e Autorização

- [ ] **[BLOQUEANTE]** Todo endpoint tem guard de autenticação ou é marcado explicitamente como `@Public()`
- [ ] **[BLOQUEANTE]** `withTenant()` chamado em toda query de entidade de domínio — nunca query sem isolamento
- [ ] JWT não logado em nenhum nível (mesmo WARN ou INFO)
- [ ] `JWT_DEV_MODE` desabilitado em staging e produção — verificar Railway env vars
- [ ] Tokens expiram conforme configurado — não há tokens de desenvolvimento sem expiração em produção

### 4.2 Validação de Input

- [ ] **[BLOQUEANTE]** Nenhum input do usuário inserido diretamente em query SQL — usar Prisma parameterizado
- [ ] DTOs validados com `class-validator` antes de chegar ao service
- [ ] `ValidationPipe` global ativo com `whitelist: true` (remove campos extras) e `forbidNonWhitelisted: true`
- [ ] Tamanho máximo de payload configurado no NestJS (`bodyParser` limit)
- [ ] Conteúdo de mensagem do usuário não injetado em templates sem sanitização

### 4.3 Segredos e Dados Sensíveis

- [ ] **[BLOQUEANTE]** Nenhum segredo (API key, JWT secret, OTP) commitado em código — verificar com `git-secrets`
- [ ] `.env` no `.gitignore` — verificar histórico do git
- [ ] Variáveis de ambiente acessadas via `process.env.VAR` — não hardcoded
- [ ] PII mascarado antes de qualquer log (Doc 20 e 25)
- [ ] Push subscriptions (endpoints VAPID) armazenados criptografados

### 4.4 Webhook e Integrações

- [ ] **[BLOQUEANTE]** Webhook EvolutionAPI validado com HMAC-SHA256 antes de processar payload
- [ ] IP allowlist configurada no Railway para webhook do EvolutionAPI (quando possível)
- [ ] Timeout configurado em todas as chamadas a serviços externos (OpenAI, EvolutionAPI, pgvector)
- [ ] Circuit breaker ativo para integração OpenAI — não há chamadas sem proteção

### 4.5 Rate Limiting

- [ ] Rate limit configurado para endpoints de chat (30 msg/h), WhatsApp (20 msg/h), calculator (10 req/min)
- [ ] OTP WhatsApp com max 3 tentativas e TTL de 5 min — verificar `WhatsappService`
- [ ] DDoS básico: Railway tem limite de requisições por IP — verificar configuração

### 4.6 Headers e Transporte

- [ ] HTTPS obrigatório (Railway enforce por padrão)
- [ ] `Strict-Transport-Security` header configurado
- [ ] CORS configurado corretamente — não usar `origin: '*'` em produção
- [ ] `Content-Security-Policy` configurado no Webchat quando embeddado

### 4.7 Dependências

- [ ] **[BLOQUEANTE]** `pnpm audit --audit-level high` passa com 0 vulnerabilidades HIGH/CRITICAL
- [ ] Dependências desatualizadas revisadas mensalmente
- [ ] Nenhuma dependência com licença incompatível (GPL sem exceção)

---

## 5. Parte 4 — Observabilidade e Operação

### 5.1 Logs

- [ ] **[BLOQUEANTE]** Correlation ID presente em todos os logs de erro e warn do PR
- [ ] Logs seguem schema JSON da Doc 25 — sem `console.log` formatado manualmente
- [ ] Nível de log correto: `error` para falhas funcionais, `warn` para anomalias, `info` para fluxo normal
- [ ] Stack trace não exposto em resposta HTTP — apenas em logs internos

### 5.2 Métricas e Alertas

- [ ] Novos módulos ou integrações adicionados ao health check `/health`
- [ ] Erros novos com `error_code` da taxonomia da Doc 20 — não novos códigos ad-hoc
- [ ] Se novo threshold de alerta necessário: adicionado ao Doc 25 (Observabilidade) no mesmo PR

### 5.3 Health Checks Pós-Deploy

- [ ] `GET /health` retorna 200 no ambiente de destino após o deploy
- [ ] `http_error_rate` < 1% por 5 min pós-deploy
- [ ] Sentry sem novos erros CRITICAL nos primeiros 10 min pós-deploy
- [ ] Circuit breaker em estado CLOSED após deploy

### 5.4 Readiness

- [ ] Novas variáveis de ambiente documentadas no `.env.example` da Doc 22
- [ ] Migrations executadas antes do deploy (`pnpm prisma migrate deploy`)
- [ ] Seeds necessários documentados e executados em staging antes de produção

---

## 6. Itens Bloqueantes

Consolidação de todos os itens que impedem merge em `main` ou promoção a produção.

| Categoria | Item | Por Que Bloqueia | Como Verificar | Impacto se Ignorado |
| --- | --- | --- | --- | --- |
| Code Review | `throw new Error()` genérico em módulo de domínio | Não segue contrato de error handling; correlation_id perdido | Buscar `throw new Error` no diff do PR | Erros sem `error_code`; usuário recebe 500 sem referência |
| Code Review | PII em logs | LGPD — dado pessoal exposto em sistemas de logging | `PiiMaskingInterceptor` + revisão manual do PR | Multa LGPD + exposição de dados de usuário |
| Code Review | Endpoint sem guard de autenticação | Endpoint público sem intenção | Verificar `@UseGuards` em todo `@Controller` novo | Acesso não autenticado a dados de cessionário |
| Code Review | Query de banco sem `withTenant()` | Cross-tenant data leak | Buscar `.findMany()`, `.findFirst()` sem `withTenant` | Exposição de dados de outro cessionário (LGPD grave) |
| Code Review | Migration DROP sem dois passos | Perda de dados em produção | Revisar migration — qualquer `DROP COLUMN`/`DROP TABLE` | Dados perdidos irreversivelmente em produção |
| Code Review | Cobertura abaixo do threshold | Risco de regressão em código crítico | `pnpm test --coverage` — ver relatório de coverage | Regressões passam pelo CI sem detecção |
| Code Review | Tool calls com `parallel_tool_calls: true` | Agente pode invocar tools em ordem incorreta | Verificar `AiService` — configuração de `ChatOpenAI` | Cálculo financeiro incorreto (calculate_delta antes de calculate_roi) |
| Acessibilidade | Contraste < 4.5:1 para texto normal | WCAG 1.4.3 AA — potencial obrigação legal | axe-core + Lighthouse Accessibility score | Usuários com baixa visão não conseguem ler conteúdo |
| Acessibilidade | Keyboard trap | WCAG 2.1.2 — funcionalidade inacessível via teclado | Teste manual com Tab + Escape | Usuários de teclado e screen reader bloqueados |
| Acessibilidade | Touch target < 44×44px | WCAG 2.5.5 — itens não acionáveis por motor grosso | Inspeção visual + DevTools device mode | Usuários móveis com dificuldade motora excluídos |
| Acessibilidade | ARIA roles incorretos | Semântica quebrada para screen readers | axe-core + NVDA/VoiceOver | Screen reader anuncia elemento de forma confusa |
| Segurança | Input SQL sem Prisma parameterizado | SQL Injection | Buscar `$queryRawUnsafe` ou template string em queries | Injeção de SQL em produção |
| Segurança | Segredo commitado em código | Exposição de credenciais | `git-secrets` pre-push hook; revisão do diff | Comprometimento de API keys, banco, JWT |
| Segurança | Webhook sem validação HMAC | Injeção de webhook falso | Verificar `WhatsappController` — `validateHmac()` | Atacante injeta mensagens falsas no sistema |
| Segurança | `pnpm audit` com vulnerabilidade HIGH/CRITICAL | Vulnerabilidade exploitável em dependência | `pnpm audit --audit-level high` no CI | Vetor de ataque via supply chain |
| Segurança | `JWT_DEV_MODE=true` em staging/produção | Autenticação bypassada | Verificar Railway env vars — staging e prod | Qualquer request com `dev-token` passa como autenticado |
| Observabilidade | Correlation ID ausente em log de erro | Impossível rastrear incidente | Revisar manualmente logs do PR | Diagnóstico em produção impossível sem rastreabilidade |
| Observabilidade | `GET /health` retornando 503 após deploy | Serviço não inicializou corretamente | `curl https://api.repasseseguro.com.br/health` | Deploy quebrado em produção sem detecção |

---

## 7. Ferramentas de Apoio

| Ferramenta | Categoria | O Que Valida | Quando Usar | Limitação |
| --- | --- | --- | --- | --- |
| ESLint + `@typescript-eslint` | Code Review | Qualidade de código TypeScript, regras de style | A cada commit (pre-commit hook) | Não detecta lógica de negócio incorreta |
| Jest `--coverage` | Code Review | Cobertura de testes por módulo | A cada PR (CI) | Cobertura alta não garante qualidade de testes |
| `pnpm audit` | Segurança | Vulnerabilidades em dependências npm | A cada PR (CI) | Não detecta vulnerabilidades de lógica de aplicação |
| `git-secrets` | Segurança | Segredos commitados (API keys, tokens) | Pre-push hook | Pode ter falsos negativos para padrões customizados |
| axe-core | Acessibilidade | WCAG 2.1 AA — contraste, ARIA, labels | Auditoria quinzenal + pré-release | Detecta ~30-40% dos problemas — não substitui teste manual |
| Lighthouse | Acessibilidade + Performance | Score de acessibilidade, performance, SEO | Auditoria quinzenal | Score alto não garante 100% de acessibilidade |
| NVDA (Windows) / VoiceOver (macOS/iOS) | Acessibilidade | Screen reader — fluxos críticos do Webchat | Pré-release | Requer setup de ambiente dedicado |
| Sentry | Observabilidade | Error tracking — novos erros pós-deploy | Pós-deploy (monitoramento 30 min) | Não detecta problemas de latência ou UX |
| `curl` + `jq` | Observabilidade | Health check manual | Pós-deploy | Não substitui monitoring contínuo |
| Railway Logs | Observabilidade | Logs em tempo real do serviço | Durante e após deploy | Retenção limitada (30 dias) |

---

## 8. Mapeamento por Categoria

| Item do Checklist | Referência Externa | Referência Interna |
| --- | --- | --- |
| Contraste 4.5:1 texto normal | WCAG 1.4.3 AA | Doc 03 Brand Theme Guide |
| Touch target 44×44px | WCAG 2.5.5 | Doc 11 Mobile |
| ARIA roles semânticos | WCAG 4.1.2 | Doc 09 Contratos de UI |
| `prefers-reduced-motion` | WCAG 2.3.3 | Doc 03 Brand Theme Guide |
| SQL Injection via Prisma | OWASP A03:2021 | Doc 12 Modelo de Dados |
| JWT/segredos em logs | OWASP A02:2021 | Doc 20 Error Handling / Doc 25 Observabilidade |
| Rate limiting | OWASP A05:2021 | Doc 16 API / Doc 14 Specs |
| HMAC webhook | OWASP A07:2021 | Doc 17 Integrações Externas |
| Dependências vulneráveis | OWASP A06:2021 | Doc 23 Guia de Contribuição |
| `withTenant()` obrigatório | Isolamento multi-tenant | Doc 18 Auth / Doc 12 Schema |
| Conventional Commits | Padrão interno | Doc 23 Guia de Contribuição |
| Correlation ID | Rastreabilidade | Doc 20 Error Handling / Doc 25 Observabilidade |

---

## 9. Periodicidade de Auditoria

| Frequência | Escopo | Responsável | Output |
| --- | --- | --- | --- |
| **PR (cada PR)** | Partes 1 + 3 bloqueantes | Author + Reviewer | Checklist no PR description |
| **Quinzenal** | Todas as partes | Tech Lead | Notion — página de auditoria com data e resultado |
| **Pré-release** | Todas as partes + Gates de deploy (Doc 27) | Tech Lead + QA | Notion — checklist de go-live |
| **Pós-incidente crítico** | Parte 3 (segurança) + Parte 4 | DRI do incidente | Post-mortem (Doc 26) |
| **Mensal** | Dependências (`pnpm audit` + revisão de versões) | Developer responsável | Issue no GitHub se encontrada vulnerabilidade |

---

## 10. Backlog de Pendências

| ID | Descrição | Prioridade | Observação |
| --- | --- | --- | --- |
| CQ-001 | Configurar `git-secrets` como pre-push hook no monorepo | Alta | Parte do setup inicial |
| CQ-002 | Configurar ESLint rule para detectar `throw new Error()` em módulos de domínio | Alta | Pode ser regra customizada ou plugin |
| CQ-003 | Integrar axe-core nos testes de integração do Webchat quando disponível (componente embeddado) | Média | Depende de acesso ao repositório da plataforma Cessionário |
| CQ-004 | Criar template de auditoria quinzenal no Notion com todos os itens deste checklist | Média | Ação administrativa |

> **Decisões Autônomas Tomadas Neste Documento:**
>
> 1. **[DECISÃO AUTÔNOMA] Threshold contraste WCAG AA (4.5:1) não AAA (7:1):** alternativa AAA descartada por ser mais restritiva que o exigido legalmente e gerar falsos positivos no design system existente. Critério: WCAG 2.1 AA é o padrão legal mínimo e mais amplamente adotado.
> 2. **[DECISÃO AUTÔNOMA] Checklist de acessibilidade orientado ao Webchat embeddado:** alternativa (checklist genérico de SPA) descartada por PG-03 (produto é backend puro sem frontend autônomo). Critério: documentar contratos de acessibilidade que a plataforma Cessionário deve honrar ao integrar o Webchat.
> 3. **[DECISÃO AUTÔNOMA] Auditoria quinzenal (não semanal):** alternativa semanal descartada por overhead excessivo em time pequeno sem degradação significativa de qualidade. Critério: frequência sustentável em early-stage.

---

*Próximo documento do pipeline: D29 — Go-Live Playbook.*
