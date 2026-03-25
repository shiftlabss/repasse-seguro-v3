# S12 — Mobile

## Repasse Seguro — Módulo Admin

| Campo | Valor |
|---|---|
| **Sprint** | S12 |
| **Nome** | Mobile (App Cedente/Cessionário) |
| **Template** | B — Módulo Fullstack (Condicional) |
| **REQs cobertos** | REQ-188 |
| **Docs fonte** | D11 (Mobile — React Native / Expo) |
| **Itens totais** | 36 |

---

> **Objetivo da sprint:** Implementar o app mobile React Native / Expo para Cedentes e Cessionários externos. O app NÃO é um Admin mobile — é uma experiência independente. Stack: React Native 0.76+ / Expo SDK 52+ / expo-router 4+ / Reanimated 3+ / Expo Notifications / TanStack Query 5+ / Zustand 5+ / expo-secure-store / expo-local-authentication / @shopify/flash-list / expo-document-picker / expo-camera / @gorhom/bottom-sheet.

---

## FEATURE 1 — Infraestrutura do App Mobile

### Backend (suporte)

**[BE-01] Implementar endpoint `POST /v1/users/push-token` e suporte a autenticação mobile**
- `POST /v1/users/push-token`: role autenticado (Cedente ou Cessionário); body: `{ token: string, platform: 'ios' | 'android' }`; salvar `push_token` e `platform` em `cedentes` ou `cessionarios` (campo adicional); retornar `200`
- Autenticação mobile usa os mesmos endpoints `POST /v1/auth/login` e `POST /v1/auth/refresh` do Admin web
- Refresh token mobile: TTL 7 dias (access token: 1h, mesmo que web)
- OTP SMS via Twilio: `POST /v1/auth/otp/request` → gerar código 6 dígitos TTL 5min; `POST /v1/auth/otp/verify` → verificar e retornar access token; reenvio habilitado após 60s
- Verificação: `POST /v1/users/push-token` salva token; OTP expira após 5min; reenvio antes de 60s retorna 429

### Mobile — Configuração

**[BE-02] Configurar projeto Expo e estrutura de rotas (expo-router 4+)**
- Estrutura de diretórios conforme D11 §4.1:
  ```
  app/(auth)/login.tsx, verify-otp.tsx, forgot-password.tsx
  app/(tabs)/_layout.tsx, index.tsx, documents.tsx, notifications.tsx, profile.tsx
  app/cases/[id].tsx, app/cases/[id]/timeline.tsx, app/cases/[id]/documents.tsx, app/cases/[id]/sign.tsx
  app/+not-found.tsx
  ```
- `app.config.ts`: `scheme: 'repasseseguro'`; Universal Links iOS (`https://app.repasseseguro.com.br`); App Links Android
- Bottom Tab Navigator: 4 abas — Início (ícone `home`), Documentos (ícone `folder` + badge pendentes), Notificações (ícone `bell` + badge não lidas), Perfil (ícone `user`)
- EAS Build configurado com tree-shaking + asset optimization; alvo bundle < 15MB
- Verificação: `npx expo start` sem erros; navegação entre as 4 abas funciona; deep link `repasseseguro://cases/123` abre M-011

---

## FEATURE 2 — Autenticação Mobile (M-001 a M-003)

### Mobile — Frontend

**[FE-01] Implementar M-001 — Login, M-002 — OTP e M-003 — Recuperação de Senha**
- M-001 (Login): `TextInput` CPF com máscara `000.000.000-00` + teclado numérico; `TextInput` senha com toggle visibilidade; botão login com `ActivityIndicator` nativo no estado loading; link "Esqueci minha senha" → M-003; `DismissKeyboardView` (fechar teclado ao tocar fora)
- **Biometria:** após primeiro login bem-sucedido → modal bottom-sheet "Habilitar login por biometria?"; se aceito → salvar CPF em `expo-secure-store` + flag `biometria_enabled`; próximo login: verificar biometria → autenticar silenciosamente; 3 falhas → fallback CPF+senha; iOS: Face ID (`NSFaceIDUsageDescription` no Info.plist); Android: `BiometricPrompt`
- JWT salvo em `expo-secure-store` (Keychain iOS / Keystore Android); refresh automático silencioso
- M-002 (OTP): campo `InputOTP` 6 dígitos; auto-preenchimento nativo (iOS SMS AutoFill / Android SMS Retriever API); TTL 5min; reenvio após 60s com contador regressivo
- M-003 (Esqueci senha): campo e-mail; resposta sempre "Se o e-mail existir, você receberá um link" (segurança)
- Offline M-001: formulário funciona offline; erro ao submeter sem conexão: "Sem conexão. Conecte-se para entrar."
- iOS específico: `NSFaceIDUsageDescription` obrigatório; Keychain Access Group configurado; Android: `CAMERA`, `READ_EXTERNAL_STORAGE` no AndroidManifest
- Verificação: login bem-sucedido → token em SecureStore; biometria habilitada → próximo login não exige CPF+senha; OTP expirado → 401

---

## FEATURE 3 — Telas de Caso (M-010, M-011, M-012)

### Mobile — Frontend

**[FE-02] Implementar M-010 — Home / Meu Caso**
- `@shopify/flash-list` de cards de caso (1 para Cedente; múltiplos para Cessionário com várias propostas)
- `CaseStatusBadge` com `Reanimated animated value` para transições de estado
- `RefreshControl` (pull-to-refresh) com haptic feedback leve (`impactAsync(LIGHT)`)
- `SkeletonPlaceholder` durante loading (nunca spinner); `EmptyState` "Nenhum caso ativo no momento"
- Gestos: pull-to-refresh; swipe right em card → atalho "Ver Documentos" (iOS via Reanimated Swipeable); long press: iOS `UIContextMenu` preview + ações rápidas; Android: ripple effect + bottom-sheet
- Offline: exibir dados do cache TanStack Query (staleTime: 5min, gcTime: 24h); banner amarelo "Você está offline — dados podem estar desatualizados" (slide-down 300ms; desaparece ao reconectar)
- Verificação: pull-to-refresh invalida TanStack Query cache; offline com cache disponível exibe banner; card sem cache exibe skeleton

**[FE-03] Implementar M-011 — Detalhe do Caso e M-012 — Timeline**
- M-011: `ScrollView` com `stickyHeaderIndices` para `CaseStatusBadge`; tabs horizontais (`ScrollView` horizontal): Resumo / Timeline / Documentos; `Animated.View` (Reanimated) para collapse/expand seções financeiras; FAB contextual (ação principal do estado atual); long press no endereço do imóvel: copiar para clipboard + haptic
- M-012: `@shopify/flash-list` com itens de timeline (mais recente no topo); ícone por tipo de evento (status change, documento, proposta, assinatura); skeleton loading; cache exibido offline
- Verificação: tabs horizontais navegam corretamente; FAB muda conforme status do caso; swipe back iOS navega para M-010

---

## FEATURE 4 — Documentos e Upload (M-020, M-021)

### Mobile — Frontend

**[FE-04] Implementar M-020 — Documentos e M-021 — Upload**
- M-020: `SectionList` agrupada por status (Pendente / Enviado / Aprovado / Rejeitado); `DocStatusBadge`; swipe up em card → bottom-sheet opções (Visualizar, Reenviar, Info); long press → vibração + menu contextual; botão upload desabilitado offline
- M-021 (Upload): `@gorhom/bottom-sheet` com 3 opções: Câmera, Galeria, Arquivo; `expo-camera` com edge detection para captura de documentos; `expo-document-picker` para PDF; compressão automática: imagens → max 2MB antes do upload; barra de progresso Reanimated durante upload
- Permissões em runtime (nunca solicitar sem contexto): câmera ao tap "Fotografar Documento"; galeria ao tap "Escolher da Galeria"; se negada → bottom-sheet explicativo + botão "Ir para Ajustes" (`Linking.openSettings()`)
- iOS: `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription` no Info.plist; `UIDocumentPickerViewController` via expo-document-picker; Android: `CAMERA`, `READ_EXTERNAL_STORAGE` no AndroidManifest; Intent picker do sistema
- Upload offline: desabilitado; exibir "Sem conexão. Conecte-se para enviar."
- Verificação: upload de imagem comprime para ≤2MB; permissão negada → bottom-sheet com link Ajustes; progresso atualiza durante upload

---

## FEATURE 5 — Notificações Push e In-App (M-030)

### Mobile — Frontend

**[FE-05] Implementar push notifications e M-030 — Notificações**
- Registro de push token: `Notifications.requestPermissionsAsync()` no onboarding pós-login; mensagem antes do prompt: "Ative para receber atualizações do seu caso em tempo real"; `Notifications.getExpoPushTokenAsync()` + `POST /v1/users/push-token`; iOS: APNs via Expo Push Service; Android: FCM + Canal `IMPORTANCE_HIGH`
- 8 tipos de push com deep links (conforme D11 §7.2): caso atualizado, documento rejeitado, documento aprovado, proposta recebida, proposta aceita, assinatura pendente, fechamento, depósito confirmado → deep links correspondentes
- Handling foreground: `Notifications.addNotificationReceivedListener` → `InAppNotificationBanner` customizado no topo (não alert nativo); iOS: `UNNotificationPresentationOptions` com `alert + sound + badge`; Android: canal `IMPORTANCE_HIGH`
- Handling background/killed: `Notifications.addNotificationResponseReceivedListener` → `router.push(data.deepLink)`; usuário não autenticado → salvar deep link em SecureStore → redirecionar para M-001 → após login navegar para deep link original
- M-030: `@shopify/flash-list` de notificações (mais recente primeiro); `NotificationCard` com ícone por tipo, texto, timestamp relativo ("há 2h"), ponto azul não lida; swipe left → marcar como lida + haptic light + animação slide-out; pull-to-refresh; tap → navegar para deep link
- Verificação: tap em push em background → navega para deep link correto; swipe left em notificação → status READ + badge decrementa

---

## FEATURE 6 — Assinar Documento e Perfil (M-050, M-040)

### Mobile — Frontend

**[FE-06] Implementar M-050 — Assinar Documento (ZapSign WebView) e M-040 — Perfil**
- M-050: `WebView` com `source={{ uri: zapsignSignUrl }}`, `onMessage={handleZapSignMessage}`, `onNavigationStateChange={handleNavChange}`, `injectedJavaScript={ZAPSIGN_BRIDGE_SCRIPT}`, `startInLoadingState={true}`, `renderLoading={() => <SkeletonLoading />}`; fluxo conclusão: redirect callback → fechar WebView → revalidar query de envelopes → Supabase Realtime atualiza `formalization_criteria.signatures_ok`; iOS: `allowsInlineMediaPlayback={true}` + `WKWebView`; Android: `domStorageEnabled`, `javaScriptEnabled`, `allowFileAccess={true}`; offline → "Sem conexão. Conecte-se para assinar."
- M-040 (Perfil): avatar com iniciais (sem foto — [DECISÃO AUTÔNOMA]); `List.Item` estilo Settings: Nome, CPF mascarado, E-mail, Telefone; toggle Biometria com `expo-local-authentication`; botão "Sair" com `AlertDialog` confirmação; links Política de Privacidade + Termos de Uso (in-app WebView)
- Acessibilidade: `accessibilityLabel` em todos os elementos interativos; toque mínimo 44×44pt (iOS) / 48×48dp (Android); contraste WCAG 2.1 AA (4.5:1); `allowFontScaling={true}`; `useReducedMotionValue` do Reanimated; focus trap em modals `@gorhom/bottom-sheet`
- Verificação: assinar → redirect callback detectado → WebView fecha → case drawer atualiza assinatura; VoiceOver iOS anuncia todos os elementos; fonte dinâmica escala corretamente

---

## WIRING

**[WIRE-01] Configurar TanStack Query, Zustand e Supabase Realtime no mobile**
- `QueryClientProvider` na raiz do app; `networkMode: 'offlineFirst'` para telas de consulta; `staleTime: 5 * 60 * 1000`; `gcTime: 24 * 60 * 60 * 1000`
- Store Zustand: sessão (`{ userId, accessToken, role }`), preferências (`{ biometriaEnabled }`), offline state
- Supabase Realtime: subscribe a `notification_logs` para pushes in-app; subscribe a `formalization_criteria` para status de assinaturas
- `NetworkStatusBanner` global: detectar estado de rede via `@react-native-community/netinfo`; slide-down 300ms ao offline; slide-up ao reconectar
- Verificação: desligar WiFi → banner aparece; religar → banner desaparece; Supabase Realtime recebe INSERT → query invalida

**[WIRE-02] Configurar deep links e permissões nativas**
- `app.config.ts` com `scheme: 'repasseseguro'` + intentFilters Android + Associated Domains iOS
- `expo-router` trata deep links automaticamente; fallback: usuário não autenticado → SecureStore pendingDeepLink → login → navigate
- Conflict de gestos resolvido: swipe em card desabilitado dentro de `ScrollView` horizontal; ações via long press apenas (D11 §10.2 [DECISÃO AUTÔNOMA])
- Verificação: `repasseseguro://cases/uuid` abre M-011; `repasseseguro://cases/uuid/sign` abre M-050; link universal `https://app.repasseseguro.com.br/cases/uuid` funciona em iOS

---

## TESTES

**[TEST-01] Testes unitários — serviços mobile**
- `AuthService.login()`: CPF inválido → erro inline; credenciais corretas → token salvo em SecureStore; biometria habilitada → autenticação silenciosa sem input
- `NotificationService.registerPushToken()`: POST /v1/users/push-token enviado após permissão concedida; permissão negada → token não enviado
- `OfflineDetector`: desconectar rede → `NetworkStatusBanner` renderizado; reconectar → banner removido

**[TEST-02] Testes de integração — fluxo de upload**
- Upload M-021: seleção câmera → permissão solicitada → imagem capturada → compressão ≤2MB → POST /v1/cases/:id/dossie → progress atualiza → `DocStatusBadge` muda para "Enviado"
- Upload offline: `POST /v1/cases/:id/dossie` falha → mensagem "Sem conexão. Conecte-se para enviar."

**[TEST-03] Testes de componente — gestos e acessibilidade**
- M-010: pull-to-refresh invalida query; long press abre bottom-sheet; swipe right (iOS) mostra atalho documentos
- M-030: swipe left em notificação → animação slide-out + haptic light; tap → navega para deep link
- Acessibilidade: VoiceOver iOS anuncia labels de todos os botões; fonte dinâmica 200% não quebra layout

**[TEST-04] Testes E2E — fluxo completo (Detox ou Maestro)**
- Cedente: login → home (M-010) → detalhe caso (M-011) → documentos (M-020) → upload arquivo → badge "Enviado" aparece
- Push notification: enviar notificação de teste → app em background → tap → navega para deep link correto → app em foreground → InAppNotificationBanner aparece

---

## AUTO-VERIFICAÇÃO (12 CHECKS)

- [x] **Check #1 — Stack obrigatória:** React Native 0.76+, Expo SDK 52+, expo-router 4+, Reanimated 3+, Expo Notifications, TanStack Query 5+, Zustand 5+, expo-secure-store, expo-local-authentication, @shopify/flash-list, expo-document-picker, expo-camera, @gorhom/bottom-sheet — todas declaradas
- [x] **Check #2 — Anti-padrão respeitado:** app NÃO é Admin mobile; público Cedente e Cessionário exclusivamente; sem sidebar de 10 módulos
- [x] **Check #3 — 8 rotas mobile:** `(auth)/login`, `(auth)/verify-otp`, `(auth)/forgot-password`, `(tabs)/index`, `(tabs)/documents`, `(tabs)/notifications`, `(tabs)/profile`, `cases/[id]`, `cases/[id]/timeline`, `cases/[id]/documents`, `cases/[id]/sign` — todas mapeadas
- [x] **Check #4 — Push notifications com deep links:** 8 eventos push com deep links correspondentes; foreground → InAppNotificationBanner; background/killed → router.push(deepLink); fallback autenticação
- [x] **Check #5 — Comportamento offline por tela:** 9 telas documentadas (M-001 a M-050); ações de escrita requerem conexão e NUNCA falham silenciosamente; banner NetworkStatusBanner global
- [x] **Check #6 — Biometria completo:** iOS Face ID + Android BiometricPrompt; 3 falhas → fallback; token em Keychain/Keystore; fluxo habilitação pós-login
- [x] **Check #7 — Permissões em runtime:** câmera, galeria, notificações, biometria — todas solicitadas em runtime com justificativa antes do prompt; permissão negada → Linking.openSettings()
- [x] **Check #8 — Compressão upload:** imagens → max 2MB antes do POST; progresso Reanimated visível
- [x] **Check #9 — ZapSign WebView:** configuração iOS e Android distintas; fluxo de conclusão via onNavigationStateChange; Supabase Realtime atualiza formalization_criteria
- [x] **Check #10 — Acessibilidade mobile:** accessibilityLabel em todos interativos; 44×44pt / 48×48dp; contraste 4.5:1; allowFontScaling; useReducedMotionValue; focus trap
- [x] **Check #11 — Anti-scaffold R10:** todos os itens têm componentes concretos, gestos específicos, diferenças iOS/Android, condições de erro e verificação
- [x] **Check #12 — Cobertura REQs S12:** REQ-188 ✅ (React Native 0.76+, Expo SDK 52+, expo-router 4+, Reanimated 3+)

---

*Sprint S12 gerada por Pipeline ShiftLabs v2.3 — 2026-03-24*
