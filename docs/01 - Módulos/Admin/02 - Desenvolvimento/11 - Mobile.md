# 11 - Mobile

## Repasse Seguro — App Mobile (Cedente / Cessionário)

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Produto, Mobile e Engenharia |
| **Escopo** | Especificação mobile com navegação, gestos, offline, push, deep links e diferenças de plataforma |
| **Módulo** | Mobile — React Native / Expo |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 — America/Fortaleza |
| **Dependências** | D01 RN · D02 Stacks · D05 PRD · D06 Mapa de Telas · D07 Wireframes · D09 Contratos de UI |

---

> 📌 **TL;DR**
>
> - **Público-alvo do app mobile:** Cedente e Cessionário (usuários externos) — não é uma versão mobile do Admin.
> - **Stack:** React Native 0.76+ · Expo SDK 52+ · expo-router 4+ · Reanimated 3+ · Expo Notifications.
> - **Navegação:** Bottom tab (4 abas) + Stack navegação por contexto. Sem sidebar.
> - **Offline:** Telas de consulta funcionam com cache local. Ações de escrita requerem conexão — nunca falham silenciosamente.
> - **Push Notifications:** Deep link contextual obrigatório. FCM (Android) + APNs (iOS).
> - **Gestos nativos:** Swipe to dismiss em modals, pull-to-refresh em listagens, long press para ações rápidas.
> - **Diferenças iOS/Android** explicitadas por seção.
> - **0 seções pendentes** — cobertura total.

---

## 1. Persona

Mobile Lead / React Native Dev com foco em experiência nativa. O app Repasse Seguro Mobile é uma experiência independente para Cedentes e Cessionários acompanharem o andamento de seus casos, assinar documentos e receber notificações — nunca uma WebView ou adaptação do Admin web.

---

## 2. Público-Alvo e Escopo do App

| Ator | Acesso | Telas Principais |
|---|---|---|
| **Cedente** | Login com CPF + senha | Status do caso, Documentos, Propostas recebidas, Assinatura, Notificações |
| **Cessionário** | Login com CPF + senha | Status do caso, Propostas enviadas/respondidas, Assinatura, Notificações |

> 🔴 **Anti-padrão:** O app mobile NÃO é um Admin mobile. Operadores internos acessam o Admin exclusivamente via browser (SPA). O app é para usuários externos.

---

## 3. Stack Técnica

```
React Native 0.76+
Expo SDK 52+
expo-router 4+                 # File-based routing (App Router pattern)
Reanimated 3+                  # Animações de gestos e transições
Expo Notifications              # Push notifications FCM + APNs
TanStack Query 5+               # Cache e sincronização de dados
Zustand 5+                      # Estado global (sessão, preferências)
expo-secure-store               # Armazenamento seguro (token JWT, biometria)
expo-local-authentication       # Face ID / Fingerprint
@shopify/flash-list             # Listas performáticas (FlatList replacement)
expo-document-picker            # Upload de documentos
expo-file-system                # Acesso ao filesystem local
expo-camera                     # Captura de documentos (câmera)
@gorhom/bottom-sheet            # Bottom sheets nativos
Zod + React Hook Form           # Validação de formulários
```

---

## 4. Estrutura de Navegação

### 4.1 Arquitetura de Rotas (expo-router)

```
app/
├── (auth)/
│   ├── login.tsx               # M-001: Login com CPF
│   ├── verify-otp.tsx          # M-002: Verificação OTP (SMS)
│   └── forgot-password.tsx     # M-003: Recuperação de senha
├── (tabs)/
│   ├── _layout.tsx             # Bottom Tab Navigator
│   ├── index.tsx               # M-010: Home / Meu Caso
│   ├── documents.tsx           # M-020: Documentos
│   ├── notifications.tsx       # M-030: Notificações
│   └── profile.tsx             # M-040: Perfil
├── cases/
│   ├── [id].tsx                # M-011: Detalhe do Caso
│   └── [id]/
│       ├── timeline.tsx        # M-012: Timeline do Caso
│       ├── documents.tsx       # M-021: Documentos do Caso
│       └── sign.tsx            # M-050: Assinar Documento (ZapSign WebView)
└── +not-found.tsx              # M-099: 404
```

### 4.2 Bottom Tab Navigator — 4 Abas

| Aba | Ícone | Tela Principal | Badge |
|---|---|---|---|
| **Início** | `home` | M-010 — Meu Caso | — |
| **Documentos** | `folder` | M-020 — Documentos | Contagem pendentes |
| **Notificações** | `bell` | M-030 — Notificações | Contagem não lidas |
| **Perfil** | `user` | M-040 — Meu Perfil | — |

### 4.3 Hierarquia de Navegação

```
AuthStack
├── Login (M-001)
├── Verificação OTP (M-002)
└── Recuperação de Senha (M-003)

TabNavigator (após autenticação)
├── Início → M-010
│   └── Stack → M-011 (Detalhe) → M-012 (Timeline)
├── Documentos → M-020
│   └── Stack → M-021 (Documentos do Caso) → M-050 (Assinar)
├── Notificações → M-030
└── Perfil → M-040
```

---

## 5. Autenticação Mobile

### 5.1 Login com CPF

| Aspecto | Comportamento |
|---|---|
| **Campo CPF** | Máscara automática `000.000.000-00`. Teclado numérico nativo. |
| **Campo Senha** | Toggle de visibilidade. Teclado padrão. |
| **Biometria** | Face ID (iOS) / Fingerprint (Android) — acionada automaticamente se habilitada nas preferências. Fallback para CPF + senha. |
| **Armazenamento** | JWT armazenado em `expo-secure-store` (criptografado no Keychain iOS / Keystore Android). |
| **Refresh Token** | Refresh automático silencioso. Expiração do access token: 1h. Refresh token: 7 dias. |

**iOS específico:**
- Face ID: `NSFaceIDUsageDescription` obrigatório no Info.plist.
- Keychain Access Group configurado para compartilhar tokens entre extensões.

**Android específico:**
- Fingerprint via `BiometricPrompt` API.
- Keystore via `expo-secure-store` — automaticamente isolado por app.

### 5.2 OTP via SMS

- Código 6 dígitos enviado via Twilio.
- Campo `InputOTP` com auto-preenchimento nativo (iOS: SMS AutoFill / Android: SMS Retriever API).
- TTL do OTP: 5 minutos. Reenvio habilitado após 60 segundos.

### 5.3 Biometria — Fluxo de Habilitação

```
1. Login bem-sucedido com CPF + senha
2. Modal bottom-sheet: "Habilitar login por biometria?"
3. Se aceito → salvar CPF no SecureStore + flag biometria_enabled
4. Próximo login: verificar biometria → se OK → recuperar senha do SecureStore → autenticar silenciosamente
5. Se biometria falhar 3x → fallback automático para CPF + senha
```

---

## 6. Telas Mobile

### M-001 — Login

**Componentes:**
- `TextInput` (CPF com máscara) + `TextInput` (senha)
- `TouchableOpacity` botão login com estado de loading (`ActivityIndicator` nativo — exceção ao padrão Skeleton, aceitável em auth)
- `Pressable` "Esqueci minha senha" → M-003

**Gestos:** Teclado fecha ao tocar fora do input (DismissKeyboardView).

**Offline:** Formulário funciona offline — erro exibido ao tentar submeter sem conexão.

---

### M-010 — Home / Meu Caso

**Componentes:**
- `FlashList` de cards de caso (um ou múltiplos para Cessionário com várias propostas)
- `CaseStatusBadge` (Reanimated animated value para transições de estado)
- `RefreshControl` — pull-to-refresh com haptic feedback leve
- `SkeletonPlaceholder` — loading state (nunca spinner)
- `EmptyState` — "Nenhum caso ativo no momento"

**Gestos:**
- Pull-to-refresh: revalida query TanStack Query + haptic `impactAsync(LIGHT)`
- Swipe right em card: atalho para "Ver Documentos" (iOS somente — Reanimated Swipeable)
- Tap em card: navega para M-011

**Offline:** Exibe dados do cache TanStack Query com banner "Você está offline — dados podem estar desatualizados".

**iOS específico:** Cards com `UIContextMenu` long press (preview + ações rápidas).
**Android específico:** Long press com ripple effect nativo (Material Design).

---

### M-011 — Detalhe do Caso

**Componentes:**
- `ScrollView` com `stickyHeaderIndices` para o CaseStatusBadge
- `Tabs` horizontais (ScrollView horizontal): Resumo / Timeline / Documentos
- `Animated.View` (Reanimated) para collapse/expand de seções financeiras
- Floating Action Button (FAB) contextual: ação principal do estado atual

**Gestos:**
- Swipe back (iOS): navega para M-010 (comportamento nativo da Stack)
- Pull-to-refresh na tab Resumo
- Long press no endereço do imóvel: copiar para clipboard com feedback haptic

---

### M-012 — Timeline do Caso

**Componentes:**
- `FlashList` com itens de timeline (de baixo para cima — mais recente no topo)
- Ícone por tipo de evento (status change, documento, proposta, assinatura)
- `Skeleton` durante carregamento

**Offline:** Timeline exibida do cache. Novos itens carregados ao reconectar.

---

### M-020 — Documentos

**Componentes:**
- `SectionList` agrupada por status: Pendente / Enviado / Aprovado / Rejeitado
- `DocStatusBadge` por documento
- `Pressable` card de documento → expandir com nome, tipo, data
- `TouchableOpacity` botão "Enviar Documento" (upload câmera ou galeria)

**Gestos:**
- Swipe up em card de documento: bottom-sheet com opções (Visualizar, Reenviar, Info)
- Long press: vibração + menu contextual

**iOS específico:** `UIDocumentPickerViewController` via `expo-document-picker`. Preview nativo de PDF via `expo-file-viewer`.
**Android específico:** Intent picker do sistema. Preview PDF via biblioteca externa.

---

### M-021 — Upload de Documento

**Componentes:**
- `@gorhom/bottom-sheet` com 3 opções: Câmera, Galeria, Arquivo
- `expo-camera`: captura com recorte automático de documento (edge detection)
- `expo-document-picker`: seleção de arquivo PDF
- `Progress` animado via Reanimated durante upload
- Compressão automática: imagens reduzidas para máx 2MB antes do upload

**Permissões:**
- Câmera: `expo-camera` → solicitar em runtime. Exibir explicação antes do modal de permissão nativo.
- Galeria: `expo-image-picker` → solicitar em runtime.
- Se negada: exibir bottom-sheet explicativo com link para Configurações do sistema.

**iOS específico:** `NSCameraUsageDescription` e `NSPhotoLibraryUsageDescription` no Info.plist.
**Android específico:** `CAMERA` e `READ_EXTERNAL_STORAGE` permissions no AndroidManifest.

---

### M-030 — Notificações

**Componentes:**
- `FlashList` de notificações (mais recente primeiro)
- `NotificationCard` com ícone por tipo de evento, texto, timestamp relativo ("há 2h")
- Badge de não lida (ponto azul) que desaparece ao tap
- Swipe left para dismissar (marcar como lida) com animação Reanimated

**Gestos:**
- Swipe left: marcar como lida + Haptic feedback light
- Pull-to-refresh
- Tap: navegar para deep link da notificação (M-011, M-021 ou M-050)

---

### M-040 — Perfil

**Componentes:**
- Avatar com iniciais (sem foto de perfil — [DECISÃO AUTÔNOMA]: mantém simplicidade e evita moderação de conteúdo)
- `List.Item` (estilo Settings): Nome, CPF (mascarado), E-mail, Telefone
- Toggle "Biometria" com `expo-local-authentication`
- Botão "Sair" com `AlertDialog` de confirmação
- Link "Política de Privacidade" e "Termos de Uso" (abre in-app WebView)

---

### M-050 — Assinar Documento (ZapSign)

**Implementação:** WebView com a URL de assinatura fornecida pelo ZapSign.

```tsx
<WebView
  source={{ uri: zapsignSignUrl }}
  onMessage={handleZapSignMessage}  // Captura evento de conclusão
  onNavigationStateChange={handleNavChange}  // Detecta redirect de conclusão
  injectedJavaScript={ZAPSIGN_BRIDGE_SCRIPT}
  startInLoadingState={true}
  renderLoading={() => <SkeletonLoading />}
/>
```

**iOS específico:**
- `WKWebView` nativo (padrão do React Native WebView).
- Suporte a Camera via `allowsInlineMediaPlayback={true}`.

**Android específico:**
- `WebView` com `domStorageEnabled={true}` e `javaScriptEnabled={true}`.
- `allowFileAccess={true}` para upload de documentos dentro do ZapSign.

**Fluxo de conclusão:**
1. ZapSign redireciona para URL de callback configurada.
2. `onNavigationStateChange` detecta a URL de callback.
3. App fecha WebView, navega de volta e dispara revalidação da query de envelopes.
4. Supabase Realtime atualiza `formalization_criteria.signatures_ok` em background.

---

## 7. Push Notifications

### 7.1 Configuração

```typescript
// Registro de token
await Notifications.requestPermissionsAsync();
const token = await Notifications.getExpoPushTokenAsync({
  projectId: Constants.expoConfig?.extra?.eas?.projectId,
});
// Enviar token para API → salvo em users (campo: push_token)
await api.post('/users/push-token', { token: token.data });
```

**iOS:** APNs via Expo Push Service. `UNUserNotificationCenter` para foreground handling.
**Android:** FCM via Expo Push Service. Canal de notificação (`notification_channel`) criado no boot do app.

### 7.2 Tipos de Notificação e Deep Links

| Evento | Título | Corpo | Deep Link |
|---|---|---|---|
| Caso avançou de status | "Seu caso foi atualizado" | "Status: {novo_status}" | `/cases/{case_id}` |
| Documento rejeitado | "Documento rejeitado" | "Verifique o motivo e reenvie" | `/cases/{case_id}/documents` |
| Documento aprovado | "Documento aprovado!" | "{tipo_documento} foi aprovado" | `/cases/{case_id}/documents` |
| Proposta recebida | "Nova proposta disponível" | "Uma proposta foi enviada para análise" | `/cases/{case_id}` |
| Proposta aceita | "Proposta aceita!" | "Sua proposta foi aceita" | `/cases/{case_id}` |
| Assinatura pendente | "Documento para assinar" | "Clique para assinar o {tipo_doc}" | `/cases/{case_id}/sign` |
| Fechamento concluído | "Fechamento realizado!" | "Seu caso foi fechado com sucesso" | `/cases/{case_id}` |
| Depósito confirmado | "Depósito confirmado" | "O valor foi depositado na conta escrow" | `/cases/{case_id}` |

### 7.3 Handling Foreground/Background

```typescript
// Foreground: exibir in-app banner
Notifications.addNotificationReceivedListener((notification) => {
  showInAppBanner(notification.request.content);
});

// Background/killed: navegação via deep link ao abrir
Notifications.addNotificationResponseReceivedListener((response) => {
  const { data } = response.notification.request.content;
  router.push(data.deepLink);
});
```

**iOS específico:** `UNNotificationPresentationOptions` com `alert + sound + badge` no foreground.
**Android específico:** Notification Channel com `IMPORTANCE_HIGH` para notificações de ação urgente.

---

## 8. Deep Links

### 8.1 Configuração (app.config.ts)

```typescript
expo: {
  scheme: 'repasseseguro',
  intentFilters: [  // Android
    {
      action: 'VIEW',
      data: [{ scheme: 'https', host: 'app.repasseseguro.com.br' }],
      category: ['DEFAULT', 'BROWSABLE'],
    }
  ],
  // iOS: Associated Domains configurado no Apple Developer
}
```

### 8.2 Mapeamento de Deep Links

| URL | Tela |
|---|---|
| `repasseseguro://cases/{id}` | M-011 — Detalhe do Caso |
| `repasseseguro://cases/{id}/documents` | M-021 — Documentos do Caso |
| `repasseseguro://cases/{id}/sign` | M-050 — Assinar Documento |
| `repasseseguro://notifications` | M-030 — Notificações |
| `https://app.repasseseguro.com.br/cases/{id}` | M-011 (Universal Links iOS / App Links Android) |

### 8.3 Handling

```typescript
// expo-router trata automaticamente deep links via app.config.ts scheme
// Fallback: se usuário não estiver autenticado, redirecionar para login
// e após autenticação navegar para o deep link original (salvo em SecureStore)
```

---

## 9. Comportamento Offline

### 9.1 Estratégia

**TanStack Query** como camada de cache offline:
- `staleTime: 5 * 60 * 1000` (5 minutos) para dados de caso
- `gcTime: 24 * 60 * 60 * 1000` (24h) para retenção do cache
- `NetworkMode: 'offlineFirst'` para telas de consulta

### 9.2 Comportamento por Tela

| Tela | Offline disponível? | Comportamento |
|---|---|---|
| M-001 Login | Não — requer autenticação | Exibir "Sem conexão. Conecte-se para entrar." |
| M-010 Home | Sim — dados do cache | Banner amarelo "Dados podem estar desatualizados" |
| M-011 Detalhe | Sim — dados do cache | Idem |
| M-012 Timeline | Sim — dados do cache | Idem |
| M-020 Documentos | Sim — lista do cache | Botão upload desabilitado |
| M-021 Upload | Não — requer upload | Exibir "Sem conexão. Conecte-se para enviar." |
| M-030 Notificações | Sim — lista do cache | Idem home |
| M-040 Perfil | Sim — dados locais | Edições desabilitadas |
| M-050 Assinatura | Não — WebView requer rede | Exibir "Sem conexão. Conecte-se para assinar." |

### 9.3 Banner de Offline

```tsx
// Componente global — exibido em todas as telas quando offline
<NetworkStatusBanner>
  Você está offline. Algumas informações podem não estar atualizadas.
</NetworkStatusBanner>
```

- Banner não bloqueia navegação nem conteúdo.
- Desaparece automaticamente quando conexão é restaurada.
- Animação: slide-down 300ms ao aparecer, slide-up ao desaparecer.

---

## 10. Gestos

### 10.1 Padrões Globais

| Gesto | Ação | Feedback |
|---|---|---|
| Pull-to-refresh | Revalidar query + atualizar dados | Refresh indicator nativo + haptic light |
| Swipe back | Voltar na stack de navegação | Nativo iOS / Android back gesture |
| Long press em card | Menu contextual (bottom-sheet) | Haptic medium + ripple (Android) |
| Swipe left em notificação | Marcar como lida | Haptic light + animação slide-out |
| Swipe up em modal | Dismiss do bottom-sheet | Haptic light |
| Double tap em valor financeiro | Copiar para clipboard | Haptic light + Toast "Copiado!" |

### 10.2 Conflitos de Gesto Resolvidos

**Problema:** Swipe horizontal em cards dentro de `FlashList` pode conflitar com `ScrollView` horizontal.
**Solução [DECISÃO AUTÔNOMA]:** Swipe em card desabilitado quando dentro de contexto de scroll horizontal. Ações disponíveis apenas via long press. Justificativa: evita friction de gesture conflict; long press tem precedência clara. Alternativa descartada: ativar swipe com `simultaneousHandlers` — maior complexidade e comportamento imprevisível.

### 10.3 Diferenças iOS vs. Android

| Gesto | iOS | Android |
|---|---|---|
| Voltar | Swipe desde a borda esquerda (nativo Stack) | Botão back nativo (físico ou gesto) |
| Drag to dismiss modal | Suportado via `@gorhom/bottom-sheet` | Idem |
| Context menu (long press) | `UIContextMenu` com preview visual | `AlertDialog` ou bottom-sheet |
| Haptic feedback | `UIImpactFeedbackGenerator` (preciso) | `Vibrator` (menos granular) |

---

## 11. Permissões Nativas

| Permissão | Momento de Solicitar | Mensagem de Justificativa | iOS Info.plist Key |
|---|---|---|---|
| Notificações push | No onboarding pós-login | "Ative para receber atualizações do seu caso em tempo real" | `UNUserNotificationCenter` |
| Câmera | Ao tap em "Fotografar Documento" | "Precisamos da câmera para capturar seus documentos" | `NSCameraUsageDescription` |
| Galeria (leitura) | Ao tap em "Escolher da Galeria" | "Precisamos acessar suas fotos para selecionar documentos" | `NSPhotoLibraryUsageDescription` |
| Biometria | Após primeiro login bem-sucedido | "Use Face ID / biometria para entrar mais rápido" | `NSFaceIDUsageDescription` |

**Regra:** Nunca solicitar permissão sem contexto claro. Sempre exibir modal explicativo *antes* do prompt nativo do sistema.

**Permissão negada:** Exibir bottom-sheet com "Para usar este recurso, ative a permissão em Ajustes > [App Name]" com botão "Ir para Ajustes" que abre `Linking.openSettings()`.

---

## 12. Acessibilidade Mobile

| Requisito | iOS | Android |
|---|---|---|
| Screen reader | VoiceOver — `accessibilityLabel` em todos os elementos interativos | TalkBack — idem |
| Tamanho de toque mínimo | 44×44pt | 48×48dp |
| Contraste | WCAG 2.1 AA (ratio 4.5:1 texto) | Idem |
| Fonte dinâmica | `allowFontScaling={true}` em todos os textos | Respeita `fontScale` do sistema |
| Reduzir movimento | `useReducedMotionValue` do Reanimated | `AccessibilityInfo.isReduceMotionEnabled()` |
| Foco em modais | Focus trap em `@gorhom/bottom-sheet` | Idem |

---

## 13. Performance

| Métrica | Alvo | Estratégia |
|---|---|---|
| TTI (Time to Interactive) | < 2s cold start | Expo hermes + bundle splitting por rota |
| FPS em listagens | 60fps estável | `FlashList` vs `FlatList` (40% mais performático) |
| Tamanho do bundle | < 15MB (download) | EAS Build com tree-shaking + asset optimization |
| Refresh em pull-to-refresh | < 1s para atualizar dados | TanStack Query staleTime curto + Realtime |
| Upload de documento | Progress visível < 100ms após start | Upload com eventos de progresso Axios |

---

## 14. Notificações In-App (Foreground)

Quando o app está em foreground e uma notificação chega:

```tsx
// Banner customizado no topo da tela (não usar alert nativo)
<InAppNotificationBanner
  title={notification.title}
  body={notification.body}
  onPress={() => router.push(notification.data.deepLink)}
  duration={4000}  // Auto-dismiss em 4s
/>
```

- Animação: slide-down 250ms + auto-dismiss com fade 150ms.
- Toque no banner: navega para o deep link.
- Swipe up: dismiss manual.

---

## 15. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop | Versão inicial — 12 telas, navegação Bottom Tab + Stack, push com deep links, offline-first, gestos, permissões, acessibilidade e performance. |

---

## 16. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Avatar sem foto de perfil | [DECISÃO AUTÔNOMA] | 6 (M-040) | Simplicidade + evitar moderação de conteúdo de foto. Alternativa (upload de avatar) descartada. | P2 | Product | Resolvido |
| Swipe em card desabilitado em scroll horizontal | [DECISÃO AUTÔNOMA] | 10.2 | Evitar conflito de gestos. Long press como alternativa. | P1 | Mobile Lead | Resolvido |
| OTP via SMS vs. Email | [DECISÃO AUTÔNOMA] | 5.2 | SMS escolhido pois Cedentes/Cessionários têm telefone cadastrado obrigatoriamente. Email como fallback apenas se SMS falhar. | P1 | Backend Lead | Resolvido |
