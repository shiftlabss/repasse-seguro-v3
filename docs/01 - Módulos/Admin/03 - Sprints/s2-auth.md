# S2 — Auth e RBAC

## Módulo Admin — Repasse Seguro

| Campo                | Valor                                                                               |
| -------------------- | ----------------------------------------------------------------------------------- |
| **Sprint**           | S2                                                                                  |
| **Nome**             | Auth e RBAC                                                                         |
| **Tipo**             | Fixa — Auth (fullstack)                                                             |
| **Template**         | A                                                                                   |
| **Docs Consultados** | D01.1, D01.5, D06, D16, D18                                                         |
| **Cross-cutting**    | D10 (Glossário), D02 (Stacks)                                                       |
| **REQs cobertos**    | REQ-025 a REQ-029, REQ-065 a REQ-076, REQ-180, REQ-193 a REQ-196, REQ-273 a REQ-278 |
| **Total de itens**   | 52                                                                                  |

---

> **Critério de conclusão de S2:** usuário consegue fazer login com 2FA; ver Dashboard vazia; logout funciona; roles são verificadas no backend e frontend; telas protegidas redirecionam; isolamento de dados por perfil ativo.

---

## ⚙️ BACKEND

### Auth Service

- [x] **[BE-01]** Implementar `POST /v1/auth/login`: busca usuário por email ignorando `deleted_at IS NOT NULL`; verifica `is_active = true`; verifica `locked_until > NOW()` → 423 `{ locked_until: ISO8601 }`; compara senha com `bcrypt.compare`; se inválida → incrementa `failed_login_attempts`; se `failed_login_attempts >= 5` → `locked_until = NOW() + 30min`; retorna 401 em ambos os casos sem revelar qual campo está errado. Se válida → zera `failed_login_attempts` e `locked_until`. Verificar: 5 tentativas falhas seguidas bloqueiam a conta; tentativa com conta bloqueada retorna 423.

- [x] **[BE-02]** Implementar fluxo 2FA no login: se `two_fa_enabled = true` → gera `temp_token` JWT (HS256, TTL 5 minutos, payload `{ sub, role, type: '2fa_pending' }`); armazena `rs:2fa_pending:{user_id} = true` no Redis com TTL 5min; retorna `{ requires_2fa: true, temp_token }`. Se `two_fa_enabled = false` → gera `access_token` (TTL 1h) + `refresh_token` (TTL 7d); armazena `rs:session:{user_id}` (TTL 24h) e `rs:refresh:{jti} = user_id` (TTL 7d). Verificar: usuário com 2FA ativo recebe `temp_token`; sem 2FA recebe tokens diretamente.

- [x] **[BE-03]** Implementar `POST /v1/auth/verify-2fa`: valida `temp_token` (tipo `2fa_pending`); valida código TOTP 6 dígitos com `otpauth`/`speakeasy` (RFC 6238, janela ±30s); se inválido → 401; se válido → gera `access_token` + `refresh_token` e apaga chave Redis `rs:2fa_pending:{user_id}`. Verificar: código correto retorna tokens; código expirado (>30s) retorna 401.

- [x] **[BE-04]** Implementar `POST /v1/auth/refresh`: valida `refresh_token`; busca `rs:refresh:{jti}` no Redis; se não encontrado → 401 (rotação invalidou); se encontrado → apaga chave antiga no Redis; gera novo par de tokens; armazena nova chave `rs:refresh:{new_jti}`. Verificar: usar refresh_token uma vez funciona; usar o mesmo refresh_token duas vezes falha na segunda.

- [x] **[BE-05]** Implementar `POST /v1/auth/logout`: extrai `jti` do access_token; armazena em blacklist Redis `rs:blacklist:{jti}` com TTL igual ao tempo restante de validade do token; retorna 204. Verificar: após logout, usar o mesmo access_token retorna 401.

- [x] **[BE-06]** Implementar `POST /v1/auth/forgot-password`: busca email; **sempre retorna 200 com a mesma mensagem** (não revela se email existe — RN-005/segurança); se email existe → gera token de reset (UUID v4); armazena `rs:reset:{token} = user_id` no Redis com TTL 3600 segundos (1 hora — DEC-003); envia e-mail via fila `notifications`. Verificar: email existente e email inexistente retornam resposta idêntica.

- [x] **[BE-07]** Implementar `POST /v1/auth/reset-password`: valida token via Redis `rs:reset:{token}`; se não encontrado → 400 (expirado); verifica `password == password_confirmation`; valida requisitos: ≥8 chars, ≥1 maiúscula, ≥1 número (RN-005.a); atualiza `password_hash` com bcrypt; apaga chave Redis de reset. Verificar: token de 1h expirado retorna 400; senha sem maiúscula retorna 422.

- [x] **[BE-08]** Implementar `JwtAuthGuard` e `RolesGuard` para RBAC: decorator `@Roles(...roles: UserRole[])` nos controllers; guard verifica role do token contra roles permitidas; não autorizado → 403 com `{ title: "Forbidden", detail: "Role insuficiente" }`. Implementar hierarquia: MASTER > COORDENADOR > GESTOR_FINANCEIRO > ANALISTA. Verificar: endpoint com `@Roles('COORDENADOR')` rejeita ANALISTA com 403; aceita COORDENADOR, GESTOR_FINANCEIRO e MASTER.

- [x] **[BE-09]** Implementar isolamento de dados por perfil — mascaramento de CPF/CNPJ no backend (DA-013/RN-131): `CpfMaskInterceptor` que, para requisições de ANALISTA, substitui `cpf` por `***.***.***-**` e oculta campos `phone`, dados bancários. Verificar: GET /v1/cedentes com token ANALISTA retorna CPF mascarado; com COORDENADOR retorna CPF completo.

- [x] **[BE-10]** Implementar `ThrottlerGuard` específico para autenticação: endpoints `/v1/auth/login` e `/v1/auth/verify-2fa` com limite de 10 req/min por IP (complementar ao bloqueio de conta de 5 tentativas). Verificar: 11ª requisição em 1 minuto retorna 429.

- [x] **[BE-11]** Registrar todos os eventos de autenticação no `AuditService`: LOGIN_SUCCESS, LOGIN_FAILED, LOGIN_BLOCKED, TWO_FA_SUCCESS, TWO_FA_FAILED, LOGOUT, PASSWORD_RESET_REQUESTED, PASSWORD_RESET_COMPLETED. Verificar: após cada evento, `SELECT event_type FROM audit.audit_logs ORDER BY timestamp DESC LIMIT 1` retorna o evento correto.

### Configuração 2FA Setup para Operadores

- [x] **[BE-12]** Implementar `GET /v1/auth/2fa/setup`: gera secret TOTP e QR code URI para o operador configurar o app autenticador; armazena secret temporário no Redis (TTL 5min) sem salvar no banco ainda. Verificar: retorna `{ qr_code_uri, secret }` para o frontend exibir.

- [x] **[BE-13]** Implementar `POST /v1/auth/2fa/enable`: valida código TOTP com o secret temporário; se válido → salva `two_fa_secret` criptografado no banco e `two_fa_enabled = true`. Verificar: após habilitar, próximo login exige 2FA.

---

## 🖥️ FRONTEND

### Telas de Auth (T-001 a T-004)

- [x] **[FE-01]** Implementar `LoginPage` (T-001, rota `/login`): formulário com inputs email (`type=email`) e senha (`type=password` com toggle show/hide); botão "Entrar" com 5 estados: default (ativo), loading (spinner+"Entrando..."+desabilitado), `erro credencial` (borda `--destructive` + mensagem inline "E-mail ou senha incorretos."), `erro bloqueio` (banner `--destructive` "Conta temporariamente bloqueada. Tente novamente em 30 minutos." + formulário desabilitado), `erro rede` (toast vermelho). Microinteração: shake horizontal 200ms no card ao erro de credencial. `aria-label` em todos os inputs; `role="alert"` em mensagens de erro; tab order: email→senha→toggle→botão. Verificar: 5 estados renderizam corretamente; shake ocorre em erro de credencial.

- [x] **[FE-02]** Implementar `TwoFaPage` (T-002, rota `/login/2fa`): 6 inputs individuais (type=number, max=1) como OTP input com foco automático avançando para o próximo ao digitar; paste automático distribui string de 6 dígitos; botão "Verificar"; link "Usar código de recuperação"; estados: default, preenchendo (foco avança), loading, `erro código inválido` (borda vermelha + mensagem "Código incorreto."), `expirado` (countdown 30s visual). Verificar: paste de "123456" distribui nos 6 inputs; shake ao erro.

- [x] **[FE-03]** Implementar `ForgotPasswordPage` (T-003, rota `/recuperar-senha`): campo email; botão "Enviar instruções"; estados: default, loading, sucesso (mensagem verde "Instruções enviadas..."), `e-mail não encontrado` (mensagem neutra segura). Link "Voltar ao login" → T-001. `aria-live="polite"` na área de feedback. Verificar: mensagem de email não encontrado é idêntica à mensagem neutra.

- [x] **[FE-04]** Implementar `ResetPasswordPage` (T-004, rota `/redefinir-senha?token=XXX`): extrai token da query string; campos nova senha + confirmação; validação inline em tempo real dos requisitos: ≥8 chars, ≥1 maiúscula, ≥1 número; `PasswordStrengthIndicator` mostrando critérios atendidos; estados: default, loading, sucesso (redirect para /login + toast "Senha redefinida com sucesso"), token inválido/expirado (mensagem "Link expirado. Solicite uma nova redefinição."). Verificar: senha sem maiúscula exibe erro inline antes de submeter.

- [x] **[FE-05]** Implementar `auth.store.ts` (Zustand): estado `{ user: User | null, accessToken: string | null }`. Access token armazenado em `sessionStorage` (tab-scoped, vulnerável a XSS evitado — RN-018 doc 18). Refresh token em `httpOnly cookie` via response header. Ação `logout()` limpa `sessionStorage`, chama `POST /v1/auth/logout` e redireciona para `/login`. Verificar: fechar aba descarta token (sessionStorage); refresh não descarta.

- [x] **[FE-06]** Implementar `AuthGuard` como componente wrapper: se `accessToken` nulo → redireciona para `/login`; se `accessToken` expirado (verificar `exp` do JWT) → tenta refresh via `POST /v1/auth/refresh`; se refresh falha → logout e redirect. Verificar: acessar `/dashboard` sem token redireciona; acessar com token expirado tenta refresh antes de redirecionar.

- [x] **[FE-07]** Implementar tela de acesso negado (para URL direta bloqueada — RN-004.a): ícone de cadeado + título "Acesso Negado" + mensagem "Você não tem permissão para acessar esta área. Contate o Master para alterar seu perfil." + botão "Voltar à Dashboard". Verificar: ANALISTA tentando acessar `/financeiro` via URL vê esta tela.

- [x] **[FE-08]** Implementar sidebar com RBAC dinâmico: ao receber notificação de mudança de perfil (polling a cada 60s ou evento WebSocket) → atualizar menus sem necessidade de logout/login (RN-004); exibir notificação no painel: "Seu perfil foi atualizado para [novo perfil]. A sidebar foi atualizada." Verificar: após alterar perfil de ANALISTA para COORDENADOR, os menus de Usuários e Supervisão IA aparecem na sidebar sem logout.

---

## 🔗 WIRING

- [x] **[WIRE-01]** Implementar `auth.service.ts` no frontend: métodos `login()`, `verify2fa()`, `refresh()`, `logout()`, `forgotPassword()`, `resetPassword()`. Todas as chamadas com `axios` e header `Content-Type: application/json`. Verificar: `login()` com credenciais válidas retorna tokens; com inválidas lança erro com mensagem correta.

- [x] **[WIRE-02]** Implementar contador de tentativas no frontend: após 3ª tentativa falha, exibir "X de 5 tentativas restantes" em amarelo; após bloqueio, exibir timer regressivo de 30 minutos. Verificar: contador exibido a partir da 3ª tentativa; timer conta regressivo visualmente.

---

## ✅ TESTES

- [x] **[TEST-01]** Testes unitários `AuthService`: happy path login sem 2FA; happy path login com 2FA (dois passos); 5 falhas consecutivas bloqueiam conta; conta bloqueada retorna 423; senha incorreta não revela existência do email; reset de senha expira em 1h. Cobertura: 85%.

- [x] **[TEST-02]** Testes unitários `JwtAuthGuard` + `RolesGuard`: hierarquia MASTER>COORDENADOR>GESTOR_FINANCEIRO>ANALISTA; endpoint protegido com COORDENADOR aceita MASTER mas rejeita ANALISTA; token expirado retorna 401; token blacklisted retorna 401.

- [x] **[TEST-03]** Testes de integração `POST /v1/auth/login`: banco real (testcontainers); login bem-sucedido retorna access + refresh token; 5 falhas consecutivas gravam `locked_until` no banco; 6ª tentativa retorna 423.

- [x] **[TEST-04]** Testes E2E (Playwright): fluxo completo Login com 2FA — T-001 (credenciais) → T-002 (código TOTP) → T-010 (Dashboard); logout limpa sessão e redireciona para /login; acesso direto a rota protegida sem autenticação redireciona. Este é um dos 5 fluxos críticos E2E obrigatórios.

- [x] **[TEST-05]** Teste de isolamento de dados: ANALISTA busca cedente via API → CPF mascarado; COORDENADOR busca mesmo cedente → CPF completo. Verificar sem dependência do frontend (chamada direta à API com tokens respectivos).

---

## 🔍 AUTO-VERIFICAÇÃO S2 (12 checks)

- [x] **[CHECK-01]** Login sem 2FA funciona end-to-end (formulário → API → token → redirect para Dashboard)
- [x] **[CHECK-02]** Login com 2FA funciona em dois passos (temp_token → código TOTP → tokens definitivos)
- [x] **[CHECK-03]** 5 falhas consecutivas bloqueiam conta por 30 minutos
- [x] **[CHECK-04]** Logout invalida token no Redis blacklist
- [x] **[CHECK-05]** RBAC backend: ANALISTA em endpoint COORDENADOR retorna 403
- [x] **[CHECK-06]** CPF mascarado na resposta da API para ANALISTA (mascaramento no backend)
- [x] **[CHECK-07]** Menus sem permissão ocultos na sidebar (não desabilitados)
- [x] **[CHECK-08]** Access token em sessionStorage; refresh token em httpOnly cookie
- [x] **[CHECK-09]** Todos os eventos de auth registrados em audit.audit_logs
- [x] **[CHECK-10]** Glossário consultado: role enum usa ANALISTA, COORDENADOR, GESTOR_FINANCEIRO, MASTER (não aliases)
- [x] **[CHECK-11]** Todos os REQs de S2 (REQ-025–029, REQ-065–076, REQ-180, REQ-193–196, REQ-273–278) cobertos
- [x] **[CHECK-12]** Testes unitários AuthService com ≥85% cobertura de branches
