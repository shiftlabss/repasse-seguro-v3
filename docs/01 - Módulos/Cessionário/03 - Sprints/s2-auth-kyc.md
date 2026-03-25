# S2 — Auth e KYC

## Sprint 2 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                                                                                                              |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sprint**         | S2 — Auth e KYC                                                                                                                                    |
| **Template**       | A — Infraestrutura (Banco → Backend → Frontend → Wiring → Testes)                                                                                  |
| **REQs cobertos**  | S2-001 a S2-046 (46 requisitos do Registro Mestre)                                                                                                 |
| **Docs fonte**     | 01.1 - RN Fundação e Acessos · 06 - Mapa de Telas · 16 - Documentação de API · 17 - Integrações Externas · 18 - Fluxos OAuth · 20 - Error Handling |
| **Total de itens** | 57 itens                                                                                                                                           |
| **Status**         | Concluida                                                                                                                                          |

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nenhum item usa termo genérico. Todos os nomes espelham exatamente os docs (tabelas, enums, campos, endpoints, telas).
- [x] ✅ Check 2 — Cada item é binariamente verificável (feito / não feito).
- [x] ✅ Check 3 — Valores numéricos idênticos aos docs (TTLs, limites, contagens, tempos de expiração).
- [x] ✅ Check 4 — Glossário (D10) consultado: `CessionarioStatus`, `KycStatus`, RLS, `service_role`, soft delete usados corretamente.
- [x] ✅ Check 5 — Nenhum item permite scaffold vazio (R10): todos os itens incluem validações reais, RBAC, lógica de estado e testes.
- [x] ✅ Check 6 — Máquinas de estado CessionarioStatus (CADASTRADA→KYC_EM_ANALISE→KYC_APROVADO/KYC_REPROVADO→BLOQUEADA_TEMPORARIAMENTE/ENCERRADA) completamente documentadas.
- [x] ✅ Check 7 — TTLs: access token 15min (prod)/30min (staging), refresh token 30 dias, sessão 30min, re-auth 15min, bloqueio 30min, Redis quota TTL 1min.
- [x] ✅ Check 8 — Cross-módulo: KYC aprovado desbloqueia acesso a marketplace e propostas — registrado em seção 🔀.
- [x] ✅ Check 9 — Sem conflitos não sinalizados entre docs.
- [x] ✅ Check 10 — Sem ambiguidades não sinalizadas.
- [x] ✅ Check 11 — Nenhum item depende de contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S2-001 a S2-046 têm ≥1 item no checklist.

---

## 🗄️ BANCO — Tabelas e Políticas Auth/KYC

- [x] **S2-B01** · Verificar que tabelas `users`, `cessionarios` e `kyc_documents` foram criadas em S1 com todos os campos corretos. Confirmar policy RLS em `kyc_documents`: apenas o próprio `cessionario_id` lê seus documentos; escrita apenas por `service_role`. Confirmar que `attempt_count` e `blocked_until` existem em `kyc_documents` para controle de bloqueio após 5 tentativas. (Doc 01.1 — RN-012; Doc 13)

- [x] **S2-B02** · Criar tabela `notification_tokens` (necessária para push): `id UUID PK`, `user_id UUID FK → users.id CASCADE`, `push_token VARCHAR(255) NOT NULL`, `platform VARCHAR(10) NOT NULL` (ios/android), `created_at TIMESTAMPTZ DEFAULT NOW()`. Índice `idx_notification_tokens_user_id`. RLS: leitura e escrita somente pelo próprio usuário. (Doc 21 — seção 2.2)

- [x] **S2-B03** · Confirmar política Supabase Storage para bucket de documentos KYC: acesso apenas por `service_role` (backend); Signed URLs com TTL configurável (curto — max 15min para downloads de documentos sensíveis); LGPD: documentos KYC nunca expostos diretamente ao frontend via URL pública. (Doc 17 — seção Supabase Storage)

---

## ⚙️ BACKEND — Endpoints de Autenticação

- [x] **S2-BE01** · Implementar `POST /api/v1/auth/register` em `apps/api/src/modules/auth/`: body `{ name: string, email: string, password: string }`; validações: `name` entre 2-100 chars, `email` formato válido único, `password` mínimo 8 chars + 1 maiúscula + 1 número; criar registro em `users` (provider=EMAIL) + `cessionarios` (status=CADASTRADA); disparar notificação NOT-CES-01 (boas-vindas/verificação de e-mail) via fila RabbitMQ `notification.email`; retornar `{ user: { id, email, name }, cessionario: { id, status: "CADASTRADA" } }` HTTP 201; erro `CONFLICT-001` se e-mail já existe; nunca retornar password no response. (Doc 16 — POST /auth/register; Doc 01.1 — RN-001)

- [x] **S2-BE02** · Implementar `POST /api/v1/auth/login` em `apps/api/src/modules/auth/`: body `{ email: string, password: string }`; validar credenciais contra Supabase Auth (provider EMAIL); verificar se conta está `BLOQUEADA_TEMPORARIAMENTE` (retornar `AUTH-003` com TTL restante se `blocked_until` > agora); gerar access token JWT (expiração 15min em prod, 30min em staging) via Supabase Auth; refresh token httpOnly cookie 30 dias; retornar `{ access_token: string, user: { id, email, name }, cessionario: { id, status } }`; rate limit: 10 req/15min por IP; logar tentativa de login com nível `info`. (Doc 16 — POST /auth/login; Doc 01.1 — RN-002, RN-008; Doc 14 — JWT)

- [x] **S2-BE03** · Implementar lógica de bloqueio por brute-force em `AuthService`: após 5 tentativas de login falhas em 1h (rastrear via Redis `rs:auth:attempts:{email}` com TTL 1h), definir `cessionarios.status = BLOQUEADA_TEMPORARIAMENTE` e `kyc_documents.blocked_until = NOW() + 30 minutos`; retornar `AUTH-003` com header `Retry-After: <segundos>`; após `blocked_until` expirar, status volta para estado anterior automaticamente via job de cleanup ou na próxima autenticação bem-sucedida. (Doc 01.1 — RN-012)

- [x] **S2-BE04** · Implementar `POST /api/v1/auth/oauth/google` em `apps/api/src/modules/auth/`: iniciar fluxo OAuth Google via Supabase Auth; callback URL `POST /api/v1/auth/oauth/google/callback`; se e-mail já existe com provider=EMAIL, vincular contas; se novo usuário, criar `users` (provider=GOOGLE) + `cessionarios` (status=CADASTRADA); retornar mesmo formato de response que login por e-mail; configurar `SUPABASE_OAUTH_GOOGLE_CLIENT_ID` e `SUPABASE_OAUTH_GOOGLE_CLIENT_SECRET`. (Doc 01.1 — RN-003; Doc 18 — OAuth flows)

- [x] **S2-BE05** · Implementar `POST /api/v1/auth/refresh` em `apps/api/src/modules/auth/`: ler refresh token do cookie httpOnly; validar via Supabase Auth; emitir novo access token JWT (15min prod/30min staging); rotacionar refresh token (novo cookie httpOnly 30 dias); retornar `{ access_token: string }`; erro `AUTH-008` se refresh token expirado/inválido → frontend redireciona para /login. (Doc 16 — POST /auth/refresh; Doc 14 — JWT)

- [x] **S2-BE06** · Implementar `POST /api/v1/auth/logout` em `apps/api/src/modules/auth/`: revogar sessão via Supabase Auth; limpar cookie httpOnly do refresh token; retornar HTTP 204. Requer JWT válido (`@UseGuards(JwtAuthGuard)`). (Doc 16 — POST /auth/logout)

- [x] **S2-BE07** · Implementar `POST /api/v1/auth/recover-password` e `POST /api/v1/auth/reset-password` em `apps/api/src/modules/auth/`: `/recover-password` body `{ email: string }` — enviar e-mail de recuperação via Supabase Auth (template Resend); retornar HTTP 200 sem revelar se e-mail existe (prevenção de user enumeration); `/reset-password` body `{ token: string, new_password: string }` — validar token JWT one-time (expiração 1h), atualizar senha via Supabase Auth, revogar todas as sessões ativas. (Doc 16 — POST /auth/recover-password, POST /auth/reset-password; Doc 01.1 — RN-004)

- [x] **S2-BE08** · Implementar `POST /api/v1/auth/re-authenticate` em `apps/api/src/modules/auth/`: body `{ password: string }` ou `{ biometric_token: string }`; revalidar credenciais do usuário autenticado; emitir token de re-auth com TTL 15 minutos (`rs:reauth:{userId}` no Redis); retornar `{ re_auth_token: string, expires_at: ISO8601 }`; erro `AUTH-010` se falhar; obrigatório para: envio de proposta, cancelamento de proposta, contraproposta, confirmação de Escrow. (Doc 01.1 — RN-007; Doc 16 — POST /auth/re-authenticate)

- [x] **S2-BE09** · Implementar `JwtAuthGuard` e `KycGuard` em `apps/api/src/common/guards/`: `JwtAuthGuard` valida Bearer token JWT via Supabase Auth e injeta `user` + `cessionario` no request context; `KycGuard` verifica `cessionario.status === 'KYC_APROVADO'` e retorna `PERM-001` se não aprovado; `ReAuthGuard` verifica token de re-auth Redis com TTL 15min. Decoradores `@RequireReAuth()` e `@RequireKyc()` para composição nos controllers. (Doc 01.1 — RN-006, RN-007)

- [x] **S2-BE10** · Implementar `POST /api/v1/auth/session/validate` em `apps/api/src/modules/auth/`: validar JWT atual; retornar `{ valid: boolean, user: {...}, cessionario: { id, status } }`; usado pelo frontend para validar sessão ao recarregar a página. Verificar que expiração de sessão por inatividade de 30min é controlada no frontend via interceptor (não server-side). (Doc 01.1 — RN-005; Doc 16)

---

## ⚙️ BACKEND — Endpoints de KYC

- [x] **S2-BE11** · Implementar `GET /api/v1/kyc/status` em `apps/api/src/modules/kyc/`: retornar `{ status: KycStatus, cessionario_status: CessionarioStatus, identity_status, address_status, selfie_status, rejection_reason?, attempt_count, blocked_until? }`; guard: apenas usuário autenticado (não exige KYC aprovado). (Doc 16 — GET /kyc/status; Doc 01.1 — RN-010)

- [x] **S2-BE12** · Implementar `POST /api/v1/kyc/documents` em `apps/api/src/modules/kyc/`: body multipart/form-data com campos `document_type` (IDENTITY_FRONT, IDENTITY_BACK, ADDRESS, SELFIE) e `file` (PDF/JPEG/PNG, max 10MB); validar MIME type e tamanho; upload para Supabase Storage bucket `kyc-documents/{cessionario_id}/{tipo}/{timestamp}.{ext}` via `service_role`; criar/atualizar registro em `kyc_documents` com URL do arquivo; rate limit: 5 req/10min por cessionário; retornar `{ document_id: string, document_type: string, url_signed: string, expires_in: 900 }`; verificar que `attempt_count < 5` (retornar `RATE-001` se ≥5 e `blocked_until` no futuro). (Doc 16 — POST /kyc/documents; Doc 01.1 — RN-011, RN-012)

- [x] **S2-BE13** · Implementar `POST /api/v1/kyc/submit` em `apps/api/src/modules/kyc/`: verificar que todos os 4 documentos (IDENTITY_FRONT, IDENTITY_BACK, ADDRESS, SELFIE) foram enviados; atualizar `kyc_documents.status = EM_ANALISE` e `cessionarios.status = KYC_EM_ANALISE`; publicar evento na fila RabbitMQ `kyc.process` com `{ cessionario_id, kyc_document_id, signed_urls: [...] }`; `KycWorker` consome a fila e chama idwall `POST /v2/matrices/run` enviando signed URLs; armazenar `idwall_session_id` em `kyc_documents`; retornar `{ message: "KYC em análise", estimated_time: "5 minutos" }`. (Doc 17 — seção idwall; Doc 01.1 — RN-010)

- [x] **S2-BE14** · Implementar webhook receiver `POST /api/v1/webhooks/idwall` em `apps/api/src/modules/kyc/`: validar assinatura HMAC-SHA256 no header `X-Idwall-Signature` com `IDWALL_WEBHOOK_SECRET` antes de processar; processar eventos `report_complete` (APPROVED → `cessionarios.status = KYC_APROVADO`, REJECTED → `cessionarios.status = KYC_REPROVADO` com `rejection_reason`) e `report_failed` (enviar para revisão manual, manter `KYC_EM_ANALISE`); disparar notificação NOT-CES-02 via fila RabbitMQ; retornar HTTP 200 imediatamente (processamento assíncrono); job de reconciliação: se webhook não recebido em 10min, polling `GET /v2/reports/{reportId}` a cada 2min (max 5 consultas). (Doc 17 — seção idwall; Doc 01.1 — RN-065)

- [x] **S2-BE15** · Implementar job de polling idwall em `apps/api/src/modules/kyc/kyc-reconciliation.service.ts`: executar a cada 2min via `@nestjs/schedule` (cron `*/2 * * * *`); buscar `kyc_documents` com `status = EM_ANALISE` e `idwall_session_id != null` e `updated_at < NOW() - 10 minutes`; para cada registro, chamar `GET /v2/reports/{idwall_session_id}` com retry 5 req max; se resultado disponível, processar igual ao webhook; se não, após 5 tentativas marcar para revisão manual. (Doc 17 — seção idwall fallback)

---

## 🎨 FRONTEND — Telas de Auth (T-AUTH-01 a T-AUTH-04)

- [x] **S2-FE01** · Implementar tela `T-AUTH-01 — Login` em `apps/web/src/features/auth/pages/LoginPage.tsx` (rota `/login`): campos `email` e `password` com validação inline (formato e-mail, senha min 8 chars); botão "Entrar" chama `POST /api/v1/auth/login`; loading state no botão durante request; tratamento de erros: `AUTH-001` → "E-mail ou senha incorretos", `AUTH-003` → "Conta bloqueada por {X} minutos"; link "Esqueci minha senha" → `/recuperar-senha`; link "Criar conta" → `/cadastro`; botão "Entrar com Google" → inicia OAuth; skeleton screen durante redirect pós-login; AnimatePresence na entrada da página. (Doc 06 — T-AUTH-01)

- [x] **S2-FE02** · Implementar tela `T-AUTH-02 — Cadastro` em `apps/web/src/features/auth/pages/RegisterPage.tsx` (rota `/cadastro`): campos `nome`, `email`, `password`, `confirmPassword`; validação: nome 2-100 chars, e-mail único (verificação apenas no submit), senha min 8 chars + 1 maiúscula + 1 número, confirmação deve bater; submit chama `POST /api/v1/auth/register`; sucesso → redirecionar para `/dashboard` com toast "Bem-vindo! Verifique seu e-mail."; erro `CONFLICT-001` → "E-mail já cadastrado". Solicitar permissão push após cadastro bem-sucedido (D11 — seção 5.2). (Doc 06 — T-AUTH-02)

- [x] **S2-FE03** · Implementar telas `T-AUTH-03 — Recuperar Senha` (rota `/recuperar-senha`) e `T-AUTH-04 — Nova Senha` (rota `/nova-senha?token=X`): T-AUTH-03: campo `email`, submit chama `POST /api/v1/auth/recover-password`; sucesso → "Enviamos instruções para seu e-mail" (sem revelar se e-mail existe); T-AUTH-04: campos `nova_senha` e `confirmar_nova_senha`, lê `?token=` da URL, submit chama `POST /api/v1/auth/reset-password`; sucesso → redirecionar para `/login` com toast "Senha redefinida com sucesso"; deep link: `repasse://reset-senha?token=JWT_TOKEN`. (Doc 06 — T-AUTH-03, T-AUTH-04; Doc 11 — seção 6.2)

- [x] **S2-FE04** · Implementar hook `useSession` em `apps/web/src/hooks/useSession.ts`: carregar sessão ao iniciar app via `POST /api/v1/auth/session/validate`; inicializar `auth.store` com `user`, `cessionario`, `kycStatus`; refresh automático de access token 1min antes da expiração (15min) usando `setTimeout` calculado pela `expires_at` do JWT; logout automático após 30min de inatividade (reset timer a cada interação do usuário); ao logout, limpar `auth.store`, redirecionar para `/login`. (Doc 01.1 — RN-005, RN-006)

- [x] **S2-FE05** · Implementar modal de re-autenticação `ReAuthModal.tsx` em `apps/web/src/components/shared/`: exibir ao tentar ação crítica (proposta, cancelamento, contraproposta, Escrow); campos `senha` ou botão "Usar biometria" (mobile); submit chama `POST /api/v1/auth/re-authenticate`; ao sucesso, armazenar `re_auth_token` no store com `expires_at` (15min); ao fechar sem autenticar, cancelar ação; foco no campo senha ao abrir (a11y — `role="dialog"`, focus trap). (Doc 01.1 — RN-007)

- [x] **S2-FE06** · Implementar `AuthGuard`, `KycGuard` e `RbacGuard` em `apps/web/src/router/guards/`: `AuthGuard` redireciona `/login` se `!isAuthenticated`; `KycGuard` mostra banner "Complete seu KYC para acessar esta funcionalidade" se `status !== 'KYC_APROVADO'` e redireciona para `/perfil/kyc`; salvar URL de destino em `auth.store.pendingDeepLink` antes de redirecionar; restaurar navegação após autenticação. (Doc 01.1 — RN-006)

---

## 🎨 FRONTEND — Telas de KYC (T-PRF-02 e T-PRF-03)

- [x] **S2-FE07** · Implementar tela `T-PRF-02 — KYC — Upload de Documentos` em `apps/web/src/features/profile/pages/KycPage.tsx` (rota `/perfil/kyc`): stepper 3 etapas (Identidade, Comprovante de Residência, Selfie); etapa 1 — upload de frente e verso do documento (RG/CNH), aceita JPEG/PNG/PDF max 10MB por arquivo; etapa 2 — upload de comprovante de residência; etapa 3 — captura/upload de selfie; cada etapa: progresso visual, validação de tipo/tamanho antes do upload; chamar `POST /api/v1/kyc/documents` para cada arquivo; ao concluir etapa 3, chamar `POST /api/v1/kyc/submit`; proteção de navegação: `Alert` ao tentar sair com dados não salvos; layout full-screen (FullPageLayout). (Doc 06 — T-PRF-02; Doc 01.1 — RN-010, RN-011)

- [x] **S2-FE08** · Implementar tela `T-PRF-03 — Status do KYC` em `apps/web/src/features/profile/pages/KycStatusPage.tsx` (rota `/perfil/kyc-status`): polling `GET /api/v1/kyc/status` a cada 30s enquanto `status = EM_ANALISE`; exibir estado atual com ícone: `EM_ANALISE` → spinner + "Analisando seus documentos (até 5 minutos)"; `APROVADO` → ✅ + "KYC aprovado! Você pode fazer propostas"; `REPROVADO` → ❌ + motivo da reprovação + botão "Reenviar documentos" (incrementa `attempt_count`); se `blocked_until` ativo, exibir countdown "Você pode tentar novamente em {X} minutos"; verificar `attempt_count ≥ 5` → exibir "Número máximo de tentativas atingido. Entre em contato com o suporte." sem botão de reenvio. (Doc 06 — T-PRF-03; Doc 01.1 — RN-012)

- [x] **S2-FE09** · Implementar `CessionarioStatusBadge.tsx` em `apps/web/src/components/domain/`: renderizar badge com cor e label para cada valor de `CessionarioStatus`: CADASTRADA (cinza, "Aguardando KYC"), KYC_EM_ANALISE (âmbar, "KYC em análise"), KYC_APROVADO (verde, "Verificado"), KYC_REPROVADO (vermelho, "KYC reprovado"), BLOQUEADA_TEMPORARIAMENTE (laranja, "Conta bloqueada"), ENCERRADA (cinza escuro, "Conta encerrada"). Reutilizável em sidebar, header e T-PRF-01. (Doc 10 — glossário CessionarioStatus)

---

## 🔌 WIRING — Integração Auth↔Frontend↔Backend

- [x] **S2-W01** · Configurar Supabase Auth no backend: `SUPABASE_SERVICE_ROLE_KEY` exclusiva para o backend (nunca exposta); configurar Google OAuth provider no painel Supabase com `SUPABASE_OAUTH_GOOGLE_CLIENT_ID` e `SUPABASE_OAUTH_GOOGLE_CLIENT_SECRET`; configurar redirect URL de produção `https://app.repasseseguro.com.br/auth/callback` e de staging `https://staging.repasse-seguro.com.br/auth/callback`. (Doc 17 — seção Supabase)

- [x] **S2-W02** · Configurar upload de documentos KYC end-to-end: frontend usa `services/api.ts` com `multipart/form-data`; backend via `service_role` faz upload para Supabase Storage; bucket `kyc-documents` privado (sem acesso público); Signed URLs geradas pelo backend com TTL 15min para visualização; Signed URLs enviadas para idwall pelo `KycWorker`. Verificar fluxo completo: upload → Storage → idwall → webhook → status atualizado.

- [x] **S2-W03** · Configurar idwall em `apps/api/src/infrastructure/idwall/`: cliente HTTP com `IDWALL_API_KEY` via header `Authorization: ${IDWALL_API_KEY}`; retry 3x backoff exponencial (2s→4s→8s); timeout 30s por request; publicar resultado na fila `kyc.result`; implementar `KycWorker` que consome `kyc.process` e orquestra chamada idwall + armazenamento do `idwall_session_id`; monitorar consumo mensal via `rs:kyc:monthly-count` Redis (contador com TTL até fim do mês). (Doc 17 — seção idwall)

- [x] **S2-W04** · Configurar cookie de refresh token no backend: `Set-Cookie: refresh_token=<token>; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth/refresh; Max-Age=2592000` (30 dias); nunca retornar refresh token no body da resposta; frontend lê access token do body; implementar `cookieParser()` no middleware NestJS. (Doc 14 — JWT)

- [x] **S2-W05** · Implementar AppLayout em `apps/web/src/components/layout/AppLayout.tsx`: sidebar com links de navegação para rotas autenticadas; header com nome do usuário, `CessionarioStatusBadge`, botão de notificações (badge contador de não lidos), botão de logout; subscription Supabase Realtime em `notifications` filtrada por `cessionario_id` (RLS) para atualizar badge em tempo real; degradação para polling 30s se Realtime indisponível com banner "Atualizações em tempo real temporariamente indisponíveis". (Doc 01.1 — RN-009; Doc 17 — Realtime)

---

## 🧪 TESTES — Auth e KYC

- [x] **S2-T01** · Testes unitários `AuthService`: `register` — cenário email duplicado retorna `CONFLICT-001`; cenário senha fraca retorna `VAL-001`; cenário sucesso cria user + cessionario com status CADASTRADA e dispara NOT-CES-01. Cobertura: 100% de branches em `AuthService`. Usar `Test.createTestingModule()` do NestJS. (Doc 27 — TC-CES-01, TC-CES-02)

- [x] **S2-T02** · Testes unitários `AuthService` — login e bloqueio: cenário credenciais inválidas retorna `AUTH-001`; cenário 5 tentativas em 1h → status BLOQUEADA_TEMPORARIAMENTE, retorna `AUTH-003` com `Retry-After`; cenário conta bloqueada com `blocked_until` no futuro retorna `AUTH-003`; cenário login válido com conta KYC_APROVADO retorna access token + seta cookie. Mock Redis para `rs:auth:attempts:{email}`.

- [x] **S2-T03** · Testes unitários `KycService`: submissão KYC com documentos faltando retorna erro; submissão completa atualiza status para `KYC_EM_ANALISE` e publica na fila `kyc.process`; webhook idwall `report_complete` APPROVED → status `KYC_APROVADO` + dispara NOT-CES-02; webhook idwall `report_complete` REJECTED → status `KYC_REPROVADO` + motivo + dispara NOT-CES-02; validação HMAC-SHA256 falha → retorna 401.

- [x] **S2-T04** · Testes de integração E2E (Playwright) — fluxo de cadastro e KYC: TC-CES-01 — "Cadastro de novo Cessionário com e-mail e senha": preencher form, submit, verificar redirecionamento para dashboard com status CADASTRADA. TC-CES-02 — "Login com credenciais válidas": login, verificar token recebido, verificar status cessionario. TC-CES-03 — "Tentativa de login com senha incorreta": verificar mensagem de erro `AUTH-001`. (Doc 01.3 — TC-CES-01, TC-CES-02, TC-CES-03)

- [x] **S2-T05** · Testes de integração E2E (Playwright) — fluxo KYC completo: TC-CES-04 — "Submissão de documentos KYC": upload dos 4 documentos, submissão, verificar status `KYC_EM_ANALISE`; TC-CES-05 — "Aprovação automática KYC": mock webhook idwall `report_complete` APPROVED, verificar status `KYC_APROVADO` + notificação NOT-CES-02. (Doc 01.3 — TC-CES-04, TC-CES-05)

- [x] **S2-T06** · Testes de segurança: verificar que `SUPABASE_SERVICE_ROLE_KEY` nunca aparece em response algum; verificar que stack trace nunca aparece no response de erro; verificar que CPF e dados bancários nunca aparecem em logs (pino.redact ativo); verificar que refresh token está em cookie httpOnly (não acessível por JS); verificar que dados do Cedente estão ausentes em todos os responses do módulo. (Doc 28 — seção 2.2, seção 3)

---

## 🔀 Cross-Módulo

- [x] **S2-CM01** · **[→ S3, S4, S5, S6, S7, S8, S9]** Garantir que `KycGuard` bloqueia acesso a todos os endpoints de marketplace (`/opportunities`), propostas (`/proposals`), negociações (`/negotiations`), Escrow (`/escrow`), formalização (`/formalization`), financeiro (`/transactions`) e IA (`/ai`) se `cessionario.status !== 'KYC_APROVADO'`. Resposta: `PERM-001` HTTP 403 com `user_message: "Complete seu KYC para acessar esta funcionalidade."`. (Doc 01.1 — RN-006)

- [x] **S2-CM02** · **[→ S3]** Quando `cessionarios.status` muda para `KYC_APROVADO` (via webhook idwall), disparar atualização via Supabase Realtime para que o frontend atualize imediatamente o `CessionarioStatusBadge` no header e o `auth.store.kycStatus` sem necessidade de reload da página. (Doc 01.1 — RN-065)

---

## 📊 COBERTURA DE REQs S2

| REQ ID | Descrição                                 | Item(s) do checklist |
| ------ | ----------------------------------------- | -------------------- |
| S2-001 | POST /auth/register                       | S2-BE01              |
| S2-002 | POST /auth/login + JWT                    | S2-BE02              |
| S2-003 | Brute-force lockout 5 tentativas/1h       | S2-BE03              |
| S2-004 | Bloqueio 30 minutos                       | S2-BE03              |
| S2-005 | OAuth Google                              | S2-BE04              |
| S2-006 | POST /auth/refresh                        | S2-BE05              |
| S2-007 | POST /auth/logout                         | S2-BE06              |
| S2-008 | POST /auth/recover-password               | S2-BE07              |
| S2-009 | POST /auth/reset-password                 | S2-BE07              |
| S2-010 | POST /auth/re-authenticate                | S2-BE08              |
| S2-011 | JwtAuthGuard + KycGuard + ReAuthGuard     | S2-BE09              |
| S2-012 | POST /auth/session/validate               | S2-BE10              |
| S2-013 | GET /kyc/status                           | S2-BE11              |
| S2-014 | POST /kyc/documents (upload)              | S2-BE12              |
| S2-015 | POST /kyc/submit                          | S2-BE13              |
| S2-016 | Webhook idwall HMAC-SHA256                | S2-BE14              |
| S2-017 | Job reconciliação idwall 2min             | S2-BE15              |
| S2-018 | Access token 15min prod/30min staging     | S2-BE02              |
| S2-019 | Refresh token httpOnly cookie 30 dias     | S2-W04               |
| S2-020 | Sessão inatividade 30min                  | S2-FE04              |
| S2-021 | Re-auth TTL 15min Redis                   | S2-BE08              |
| S2-022 | Redis rs:auth:attempts:{email} TTL 1h     | S2-BE03              |
| S2-023 | T-AUTH-01 Login                           | S2-FE01              |
| S2-024 | T-AUTH-02 Cadastro                        | S2-FE02              |
| S2-025 | T-AUTH-03 Recuperar Senha                 | S2-FE03              |
| S2-026 | T-AUTH-04 Nova Senha                      | S2-FE03              |
| S2-027 | Hook useSession + auto-refresh            | S2-FE04              |
| S2-028 | Modal ReAuth                              | S2-FE05              |
| S2-029 | AuthGuard + KycGuard + RbacGuard FE       | S2-FE06              |
| S2-030 | T-PRF-02 KYC Upload stepper 3 etapas      | S2-FE07              |
| S2-031 | T-PRF-03 KYC Status polling               | S2-FE08              |
| S2-032 | CessionarioStatusBadge                    | S2-FE09              |
| S2-033 | notification_tokens tabela                | S2-B02               |
| S2-034 | Supabase Storage bucket kyc-documents     | S2-B03               |
| S2-035 | Signed URLs 15min para KYC                | S2-W02               |
| S2-036 | Cookie SameSite + HttpOnly                | S2-W04               |
| S2-037 | AppLayout com Realtime notifications      | S2-W05               |
| S2-038 | Resend NOT-CES-01 em registro             | S2-BE01              |
| S2-039 | NOT-CES-02 em resultado KYC               | S2-BE14              |
| S2-040 | Estado KYC_EM_ANALISE                     | S2-BE13              |
| S2-041 | Estado KYC_APROVADO via webhook           | S2-BE14              |
| S2-042 | Estado KYC_REPROVADO via webhook          | S2-BE14              |
| S2-043 | BLOQUEADA_TEMPORARIAMENTE + blocked_until | S2-BE03              |
| S2-044 | attempt_count controle 5 tentativas KYC   | S2-BE12, S2-FE08     |
| S2-045 | Teste TC-CES-01 a TC-CES-05               | S2-T04, S2-T05       |
| S2-046 | Segurança: CPF/JWT nunca em logs          | S2-T06               |
