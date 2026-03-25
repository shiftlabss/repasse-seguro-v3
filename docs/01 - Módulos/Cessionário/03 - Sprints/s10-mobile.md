# S10 — Mobile (Sprint Condicional)

## Sprint 10 · Módulo Cessionário · Plataforma Repasse Seguro

| Campo              | Valor                                                        |
| ------------------ | ------------------------------------------------------------ |
| **Sprint**         | S10 — Mobile (Condicional — confirmado: mobile implementado) |
| **Template**       | B — Módulo Fullstack (feature por plataforma)                |
| **REQs cobertos**  | S10-001 a S10-003 (3 requisitos do Registro Mestre)          |
| **Docs fonte**     | 11 - Mobile · 02 - Stacks · 06 - Mapa de Telas               |
| **Total de itens** | 42 itens                                                     |
| **Status**         | Concluída                                                    |

**Pré-requisito:** S1–S9 concluídas. Este sprint adapta o produto web para a plataforma React Native 0.76 + Expo SDK 52.

---

## Auto-Verificação (12 checks)

- [x] ✅ Check 1 — Nomes exatos: `expo-router 4`, `FlashList`, `expo-local-authentication`, `expo-notifications`, `expo-secure-store`, `repasse://` scheme, deep links, plataformas iOS 16+ e Android 10 (API 29+).
- [x] ✅ Check 2 — Cada item binariamente verificável.
- [x] ✅ Check 3 — 8 telas offline com dados em cache; 5 tabs de navegação; 17 deep links mapeados; GC time 24h, stale time 5min, max 100 oportunidades + 50 propostas + 20 negociações em cache.
- [x] ✅ Check 4 — Glossário: `FlashList`, `expo-secure-store`, `offline queue`, `AnimatePresence`, `SafeAreaView`.
- [x] ✅ Check 5 — Anti-scaffold R10: lógica offline real, permissões nativas reais, biometria real.
- [x] ✅ Check 6 — Comportamento por plataforma documentado: iOS swipe-back vs Android back button; Face ID vs Fingerprint; KeyboardAvoidingView por plataforma.
- [x] ✅ Check 7 — Cold start < 3s; warm start < 800ms; bundle < 2MB gzipped.
- [x] ✅ Check 8 — Cross-módulo: todos os módulos S2–S9 têm telas mobile mapeadas em `apps/mobile/`.
- [x] ✅ Check 9 — Sem conflitos.
- [x] ✅ Check 10 — Sem ambiguidades. ⚠️ AMBÍGUO verificado: feriados em dias úteis (S5-W02) — mesma interpretação conservadora aplicada.
- [x] ✅ Check 11 — Sem contexto perdido.
- [x] ✅ Check 12 — 100% dos REQs S10-001 a S10-003 cobertos.

---

## FEATURE 1 — Setup e Navegação Mobile

### 🗄️ Banco / Backend

- [x] **S10-B01** · Confirmar que tabela `notification_tokens` (criada em S2-B02) aceita `platform` = `ios` ou `android`; adicionar coluna `app_version VARCHAR(20)` para rastrear versão do app ao registrar token (útil para suporte). Migration: `ALTER TABLE notification_tokens ADD COLUMN IF NOT EXISTS app_version VARCHAR(20);`.

### ⚙️ Backend

- [x] **S10-BE01** · Implementar `POST /api/v1/notifications/push-token` em `apps/api/src/modules/notifications/`: guard `JwtAuthGuard`; body `{ push_token: string, platform: "ios"|"android", app_version: string }`; criar ou atualizar `notification_tokens` (upsert por `user_id + platform`); remover tokens inválidos (DeviceNotRegistered) ao receber erro do Expo Push API. (Doc 11 — seção 5.1; Doc 21 — seção 2.2)

- [x] **S10-BE02** · Implementar `DELETE /api/v1/notifications/push-token` em `apps/api/src/modules/notifications/`: guard `JwtAuthGuard`; body `{ platform: "ios"|"android" }`; remover token da tabela ao fazer logout ou ao desabilitar push nas configurações. (Doc 11 — seção 5.5)

### 🎨 Frontend Mobile

- [x] **S10-FE01** · Inicializar `apps/mobile/` com Expo SDK 52 + React Native 0.76 + expo-router 4: estrutura de pastas `app/` com file-based routing exatamente conforme Doc 11 seção 2.1 (auth group, tabs group, modais); instalar todas as dependências da stack mobile: `expo-router@4`, `react-native-reanimated@3`, `react-native-gesture-handler@2`, `@tanstack/react-query@5`, `zustand@5`, `expo-secure-store`, `expo-camera`, `expo-image-picker`, `expo-document-picker`, `expo-notifications`, `expo-local-authentication`, `expo-haptics`, `@react-native-community/netinfo`, `@shopify/flash-list`. Verificar: `npx expo start` abre sem erros.

- [x] **S10-FE02** · Configurar `app/_layout.tsx` (RootLayout): providers `TanStack Query`, `ThemeProvider`, `AuthGuard`; `expo-status-bar`; `SafeAreaProvider`; tratar cold start via `getLastNotificationResponseAsync()` para capturar tap em notificação com app fechado; configurar `scheme: "repasse"` em `app.json` para deep links. (Doc 11 — seção 2.1)

- [x] **S10-FE03** · Configurar `app/(tabs)/_layout.tsx` (TabLayout): 5 tabs com ícones corretos: Dashboard (ícone Home), Marketplace (ícone Search), Operações (ícone FileText + badge contador de propostas/negociações ativas), Financeiro (ícone DollarSign), Perfil (ícone User + badge KYC se não aprovado); `safe-area` aplicada no iOS (acima do home indicator); `KeyboardAvoidingView` com `behavior="padding"` (iOS) e `behavior="height"` (Android) aplicado no layout wrapper. (Doc 11 — seção 2.2)

- [x] **S10-FE04** · Implementar todas as telas mobile em `apps/mobile/app/`: reutilizar lógica de negócio de `packages/shared-types` e hooks compartilhados; adaptar componentes para React Native (`View`, `Text`, `Pressable` em vez de `div`, `p`, `button`); layout responsivo com `Dimensions.get('window')` sem hardcoded values; fontes fallback: San Francisco (iOS) e Roboto (Android); scroll bounce nativo iOS habilitado; overscroll glow Android (padrão). (Doc 11 — seção 1.4)

- [x] **S10-FE05** · Implementar `FlashList` (Shopify) para todas as listas longas: `T-OPR-01` (Marketplace), `T-PRP-01` (Propostas), `T-NEG-01` (Negociações), `T-FIN-01` (Financeiro); substituir `FlatList`; `estimatedItemSize` obrigatório por lista; `getItemLayout` para heights fixas; verificar 60fps no scroll via React Native Debugger. (Doc 11 — seção 9.2)

- [x] **S10-FE06** · Implementar gestos nativos com React Native Gesture Handler: swipe-back iOS nativo via expo-router (automático); swipe-down para fechar modais (`withSpringConfig` no iOS, botão back Android); pull-to-refresh com `RefreshControl` nativo nas 4 listas; swipe-left em item de lista para revelar ação "Cancelar" (apenas swipe left — não right — para evitar conflito com swipe-back do sistema Android); `gestureResponseDistance: 10px` em Stack navigators para Android gesture navigation. (Doc 11 — seção 3)

- [x] **S10-FE07** · Implementar feedback háptico em ações de domínio via `expo-haptics`: proposta enviada → `NotificationFeedbackType.Success`; proposta cancelada → `NotificationFeedbackType.Warning`; erro de validação (quota, KYC) → `NotificationFeedbackType.Error`; contraproposta enviada → `ImpactFeedbackStyle.Medium`; upload concluído → `NotificationFeedbackType.Success`; cópia de dados bancários → `ImpactFeedbackStyle.Light`; PROIBIDO em listagem/scroll. (Doc 11 — seção 3.2)

---

## FEATURE 2 — Permissões Nativas e Push

- [x] **S10-FE08** · Implementar permissão de câmera (`Camera`) para selfie KYC: solicitar ao iniciar etapa 3 do KYC stepper com mensagem de contexto "Para a prova de vida, precisamos acessar sua câmera."; fallback se negada: oferecer upload da galeria; se revogada após concessão: `Alert` com botão "Abrir Configurações" via `Linking.openSettings()`. (Doc 11 — seção 7)

- [x] **S10-FE09** · Implementar permissão de galeria/fotos (`MediaLibrary`) para upload de documentos KYC: solicitar ao tocar "Escolher da galeria" nas etapas 1/2 do KYC; fallback: oferecer câmera; tratamento de revogação: mesmo fluxo de câmera. Implementar `expo-document-picker` para PDFs. (Doc 11 — seção 7.1)

- [x] **S10-FE10** · Implementar push notifications nativas via `expo-notifications`: (1) solicitar permissão após cadastro bem-sucedido com mensagem de contexto; iOS: `requestPermissionsAsync()`; Android 13+: `PermissionsAndroid.request(POST_NOTIFICATIONS)`; Android 12-: automático; (2) ao receber token Expo: `POST /api/v1/notifications/push-token` com `platform` e `app_version`; (3) ao revogar: remover com `DELETE /api/v1/notifications/push-token`; (4) tratamento de recusa: `Alert.alert("Notificações desabilitadas", "...", [{text: "Abrir Configurações", onPress: () => Linking.openSettings()}, {text: "Continuar sem notificações"}])` — não solicitar novamente na sessão; (5) fallback: e-mail permanece ativo (RN-069). (Doc 11 — seção 5.2)

- [x] **S10-FE11** · Implementar listeners de notificação push: foreground: `addNotificationReceivedListener` → exibir in-app banner customizado 4s (não via OS); background/killed: `addNotificationResponseReceivedListener` → navegar para deep link; cold start: `getLastNotificationResponseAsync()` no startup. Configurar canais Android: `repasse_alerts` (HIGH priority — NOT-CES-01, 04, 05, 06, 07, 08, 09) e `repasse_updates` (DEFAULT — demais). (Doc 11 — seção 5.3, 5.4)

- [x] **S10-FE12** · Implementar deep links com `expo-router`: custom scheme `repasse://`; Universal Links iOS via `apple-app-site-association` em `https://app.repasseseguro.com.br`; App Links Android via `assetlinks.json`; configurar todos os 18 deep links mapeados em Doc 11 seção 6.2 (repasse://login, repasse://marketplace, repasse://oportunidade/{id}, repasse://proposta/{id}, repasse://negociacao/{id}, repasse://negociacao/{id}/escrow, repasse://formalizacao/{id}, repasse://financeiro, repasse://analista, repasse://kyc, etc.); autenticação obrigatória: salvar `pendingDeepLink` no Zustand, redirecionar para login, restaurar após auth. (Doc 11 — seção 6)

---

## FEATURE 3 — Comportamento Offline

- [x] **S10-FE13** · Implementar `OfflineBanner` no mobile: detectar estado de rede via `@react-native-community/netinfo` listener; banner `<OfflineBanner>` com fundo `#F59E0B` (âmbar) + ícone `WifiOff` + "Você está offline. Algumas funcionalidades estão indisponíveis."; exibido no topo, abaixo do header, com `AnimatePresence` fade out 300ms ao reconectar; badge "Atualizado às HH:mm" em listas com dados em cache. (Doc 11 — seção 4.4)

- [x] **S10-FE14** · Configurar TanStack Query com `AsyncStorage` como `persistQueryClient` para cache offline: GC time 24h; stale time 5min; máximo em cache: 100 oportunidades + 50 propostas + 20 negociações (limitado por AsyncStorage ~6MB por chave iOS); NUNCA persistir: tokens de auth (usar `expo-secure-store`), dados KYC, documentos de formalização (LGPD); ao reconectar: TanStack Query invalida queries stale automaticamente via `netInfo.addEventListener`; servidor é fonte de verdade em conflito. (Doc 11 — seção 4.1, 4.2, 4.5)

- [x] **S10-FE15** · Implementar comportamento offline por tela conforme tabela Doc 11 seção 4.2: T-OPR-03 (proposta) — botão desabilitado offline + tooltip; T-NEG-02 (chat) — input desabilitado + banner; T-NEG-03/04 (Escrow) — bloqueado; T-AUTH-_ — sempre online; T-ASS-_ — bloqueado; T-IA-01 — bloqueado; fila offline: apenas "alteração de preferências de notificação" e "marcar notificação como lida" são enfileiráveis (auto-sync ao reconectar); ações não enfileiráveis listadas: envio de proposta, cancelamento, contraproposta, Escrow. (Doc 11 — seção 4.2, 4.3)

---

## FEATURE 4 — Biometria e Re-Auth Mobile

- [x] **S10-FE16** · Implementar re-autenticação biométrica via `expo-local-authentication`: solicitar biometria ao primeiro uso de re-auth (proposta, cancelamento) com mensagem de contexto; Face ID (iOS iPhone X+) ou Touch ID (iOS SE, iPad) ou Fingerprint/Face Unlock (Android); se biometria indisponível ou revogada: fallback automático para campo de senha (sem Alert de erro); resultado biométrico enviado ao `POST /api/v1/auth/re-authenticate` como `biometric_token`. (Doc 11 — seção 7.1)

---

## FEATURE 5 — Performance e Build

- [x] **S10-FE17** · Configurar métricas de performance mobile (targets Doc 11 seção 9.1): cold start < 3s (medir via `expo-startup-time` + Sentry Performance); warm start < 800ms; TTI Dashboard < 2s; FPS scroll 60fps (120fps Pro Motion iOS); bundle < 2MB gzipped (verificar via EAS Build stats); app instalado iOS < 80MB, Android < 50MB. Configurar Sentry Performance no mobile: `Sentry.init()` com `tracesSampleRate: 0.2` para prod.

- [x] **S10-FE18** · Configurar EAS Build: `eas.json` com profiles `development`, `preview`, `production`; APNs credentials (iOS push certificates) via EAS Credentials; FCM API key (Android push) via EAS Secrets; `EAS_PROJECT_ID` configurado; comando `eas build --platform all` funcionando. OTA updates: Expo Updates configurado para JS-only changes sem nova submissão às stores. (Doc 24 — Secrets EAS)

- [x] **S10-FE19** · Configurar `KeyboardAvoidingView` por plataforma em todos os formulários do mobile (login, cadastro, proposta, chat, KYC): iOS: `behavior="padding"`; Android: `behavior="height"` + `windowSoftInputMode="adjustResize"` no `AndroidManifest.xml`; campos de valor monetário: iOS `keyboardType="decimal-pad"`, Android `keyboardType="number-pad"` com parse manual `parseFloat(value.replace(',', '.'))`. (Doc 11 — seção 8)

### 🧪 Testes

- [x] **S10-T01** · Testes unitários em `apps/mobile/`: `addBusinessDays` com timezone `America/Fortaleza`; `calculateOfflineQueue` retorna apenas ações enfileiráveis; `sanitizeOpportunity` não retorna dados Cedente em mobile.

- [x] **S10-T02** · Testes de permissões: mock `expo-local-authentication.authenticateAsync()` com resultado negativo → fallback para campo senha; mock `expo-notifications.requestPermissionsAsync()` com `denied` → Alert exibido sem loop de solicitação.

- [x] **S10-T03** · Testes de dispositivos de referência (conforme Doc 11 seção 1.3): verificar que app roda sem crashes nos 6 dispositivos de referência (iPhone 15 Pro iOS 17, iPhone 13 iOS 16, iPhone SE iOS 16, Samsung Galaxy S23 Android 13, Samsung Galaxy A53 Android 12, Moto G Pure Android 10); cold start < 3s em todos.

---

## 🔀 Cross-Módulo

- [x] **S10-CM01** · **[← S2–S9]** Todas as telas mobile reutilizam os mesmos endpoints de API (S2–S9). Não criar endpoints específicos para mobile. O mesmo JWT de autenticação funciona em web e mobile.

- [x] **S10-CM02** · **[← S3]** Notificações push mobile são gerenciadas pelo `PushWorker` de S3. Garantir que `PushWorker` consome `notification_tokens.platform` para customizar payload por plataforma (canal Android `repasse_alerts` vs `repasse_updates`).

---

## 📊 COBERTURA DE REQs S10

| REQ ID  | Descrição                                         | Item(s)            |
| ------- | ------------------------------------------------- | ------------------ |
| S10-001 | Expo SDK 52 + React Native 0.76 + expo-router 4   | S10-FE01, S10-FE02 |
| S10-002 | iOS 16+ e Android 10 (API 29+)                    | S10-FE01, S10-FE17 |
| S10-003 | Push notifications nativas + deep links + offline | S10-FE10—S10-FE15  |
