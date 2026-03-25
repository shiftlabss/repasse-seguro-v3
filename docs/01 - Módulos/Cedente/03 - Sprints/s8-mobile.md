# S8 — Mobile

## Módulo Cedente · Repasse Seguro · Sprint 8

| Campo             | Valor                                                                                                  |
| ----------------- | ------------------------------------------------------------------------------------------------------ |
| **Sprint**        | S8 — Mobile                                                                                            |
| **Tipo**          | Template B — Módulo Fullstack por Feature                                                              |
| **Docs Fonte**    | D11 (Mobile), D18 (Auth), D01.1 (RN-087), D21 (Notificações)                                           |
| **REQs cobertos** | REQ-132, REQ-185, REQ-216, REQ-217, REQ-218, REQ-219, REQ-220                                          |
| **Dependências**  | S1 (infra, env vars EAS), S2 (auth tokens expo-secure-store scaffolded), S3–S7 (APIs já implementadas) |
| **Status**        | Pendente                                                                                               |

---

## Contexto

Esta sprint implementa o aplicativo mobile nativo (React Native 0.76+ / Expo SDK 52+ / expo-router 4) para o módulo Cedente. O web já está funcional (S2–S7); o mobile expõe os mesmos endpoints via clientes HTTP nativos, adicionando câmera, biometria, push notifications, comportamento offline e ZapSign via WebView. Não há novos endpoints de backend nesta sprint — apenas implementação nativa no app `apps/mobile-cedente/`.

---

## FEATURE 1 — Infraestrutura Mobile e Navegação

### Banco de Dados / Infra

- [x] **Tabela `push_tokens` no Prisma** — criar migration `add_push_tokens`:
  - Campos obrigatórios: `id UUID PK DEFAULT uuid_generate_v4()`, `cedente_id UUID NOT NULL FK→cedentes(id) ON DELETE CASCADE`, `token TEXT NOT NULL`, `plataforma TEXT NOT NULL CHECK (plataforma IN ('ios', 'android'))`, `device_id TEXT NOT NULL`, `ativo BOOLEAN NOT NULL DEFAULT TRUE`, `created_at TIMESTAMPTZ DEFAULT NOW()`, `updated_at TIMESTAMPTZ DEFAULT NOW()`
  - Índice: `UNIQUE (cedente_id, device_id)` — um token por dispositivo por Cedente
  - RLS: `CREATE POLICY "push_tokens_cedente" ON push_tokens FOR ALL USING (auth.uid() = (SELECT auth_id FROM cedentes WHERE id = cedente_id))`
  - Validar: `pnpm prisma migrate dev --name add_push_tokens` executa sem erro; `pnpm prisma generate` sem erro

- [x] **Endpoint `POST /cedentes/push-token`** — registrar/atualizar token de dispositivo:
  - Body: `{ token: string, plataforma: 'ios' | 'android', device_id: string }`
  - Lógica: `upsert` por `(cedente_id, device_id)` — atualiza `token` e `ativo=true`; deduplicação automática
  - Auth: `JwtAuthGuard` obrigatório
  - Response 200: `{ message: 'Token registrado' }`
  - Validar: token novo → row criada; token atualizado → row existente atualizada; token sem auth → 401

### Frontend Mobile — Estrutura Base

- [x] **Criar `apps/mobile-cedente/`** com estrutura de pastas conforme D11 seção 2.1:
  - Estrutura exata:
    ```
    app/
    ├── _layout.tsx                    # Root layout — AuthProvider, ThemeProvider, QueryClientProvider
    ├── (auth)/
    │   ├── _layout.tsx                # AuthLayout — sem tab bar
    │   ├── login.tsx                  # T-001
    │   ├── cadastro.tsx               # T-002
    │   ├── ativacao-pendente.tsx      # T-003
    │   ├── ativacao-sucesso.tsx       # T-004
    │   ├── link-expirado.tsx          # T-005
    │   ├── recuperar-senha.tsx        # T-006
    │   ├── recuperar-senha-enviado.tsx# T-007
    │   ├── redefinir-senha.tsx        # T-008
    │   └── conta-bloqueada.tsx        # T-009
    ├── (tabs)/
    │   ├── _layout.tsx                # TabsLayout — bottom tab bar T-012
    │   ├── dashboard.tsx              # T-013 / T-014
    │   ├── casos/
    │   │   ├── index.tsx              # T-015
    │   │   └── [id]/
    │   │       ├── index.tsx          # T-016
    │   │       ├── documentos.tsx     # T-017
    │   │       ├── propostas.tsx      # T-018
    │   │       ├── assinaturas.tsx    # T-019
    │   │       ├── financeiro.tsx     # T-020
    │   │       └── historico.tsx      # T-021
    │   ├── documentos.tsx             # T-040
    │   ├── propostas/
    │   │   ├── index.tsx              # T-035
    │   │   └── [id].tsx               # T-036
    │   └── mais/
    │       ├── index.tsx
    │       ├── guardiao.tsx           # T-048
    │       ├── assinaturas.tsx        # T-043
    │       └── perfil/
    │           ├── dados.tsx          # T-050
    │           ├── notificacoes.tsx   # T-051
    │           └── lgpd.tsx           # T-052
    ├── modals/
    │   ├── camera.tsx                 # T-042 — fullscreen
    │   ├── zapsign.tsx                # T-044 — WebView fullscreen
    │   ├── aceitar-proposta.tsx       # T-037
    │   ├── recusar-proposta.tsx       # T-038
    │   ├── contraproposta.tsx         # T-039
    │   ├── cancelar-caso.tsx          # T-022 / T-023
    │   ├── desistencia.tsx            # T-047
    │   ├── escalonamento.tsx          # T-031 / T-032
    │   └── expiracao-sessao.tsx       # T-010
    └── +not-found.tsx
    ```
  - Validar: `pnpm expo start` no `apps/mobile-cedente/` sem erro; expo-router reconhece todas as rotas

- [x] **`app.json` — configuração Expo SDK 52+**:
  - `"jsEngine": "hermes"` obrigatório (reduz TTI ~20%, memória ~40%)
  - `"scheme": "repasse-seguro"` para deep links
  - iOS: `"associatedDomains": ["applinks:repasse-seguro.com.br"]`
  - Android: `intentFilters` com `autoVerify: true`, `scheme: "https"`, `host: "repasse-seguro.com.br"`
  - Permissões iOS: `NSCameraUsageDescription` = "Para envio de documentos do dossiê"
  - Permissões Android: `CAMERA`, `POST_NOTIFICATIONS` (API 33+)
  - Validar: `pnpm expo prebuild --platform ios` sem erro; `pnpm expo prebuild --platform android` sem erro

- [x] **`constants/layout.ts`** — constantes de layout (nunca hardcoded em StyleSheet):
  - `CASE_CARD_HEIGHT: number` — altura fixa dos CaseCards (para `getItemLayout` FlatList)
  - `PROPOSAL_CARD_HEIGHT: number` — altura fixa dos ProposalCards
  - `DOCUMENT_CARD_HEIGHT: number` — altura fixa dos DocumentCards
  - `MIN_TOUCH_TARGET: 44` — tamanho mínimo de toque (iOS 44pt / Android 48dp, usar 44 como base)
  - Validar: import de `constants/layout.ts` sem erro em todos os componentes que usam FlatList

- [x] **Bottom Tab Bar T-012** — `app/(tabs)/_layout.tsx`:
  - 5 tabs: Dashboard (LayoutDashboard), Meus Casos (FolderOpen), Documentos (FileText), Propostas (Tag), Mais (Grid2x2)
  - `tabBarActiveTintColor: tokens.primary` (cor primária do Brand Guide D03)
  - `tabBarInactiveTintColor: tokens.mutedForeground`
  - `height: 60 + insets.bottom`; `paddingBottom: insets.bottom`
  - Badge Documentos: número de documentos pendentes ou rejeitados (apenas quando > 0; máx exibido "99+")
  - Badge Propostas: número de propostas com `status = AGUARDANDO_RESPOSTA` (apenas quando > 0; máx "99+")
  - Badge Mais: soma de assinaturas pendentes + outras pendências (apenas quando > 0)
  - Validar: navegar entre as 5 tabs sem erro; badges aparecem com valores mock; safe area respeitada em iPhone SE e Galaxy S23

- [x] **`AuthGuard` — proteção de rotas autenticadas**:
  - `app/_layout.tsx`: verificar token em `expo-secure-store` antes de renderizar `(tabs)`
  - Se sem token: redirecionar para `/(auth)/login`
  - Se com token válido: renderizar `(tabs)` normalmente
  - Validar: sem token → redireciona para login; com token válido → renderiza dashboard; token expirado → redireciona para login com `setDeepLinkPendente` preservado

- [x] **`useNetworkStatus` hook** — detecção de rede via `@react-native-community/netinfo`:
  - Exportado de `hooks/useNetworkStatus.ts`
  - Retorna `{ isOnline: boolean }`
  - Banner offline global: quando `isOnline === false`, exibir banner âmbar fixo no topo com texto "Sem conexão. Mostrando dados salvos."
  - Banner desaparece automaticamente ao reconectar
  - Validar: simular modo avião → banner aparece; reativar → banner some; hook disponível globalmente via Provider

---

## FEATURE 2 — Upload por Câmera (T-042)

### Backend (endpoint auxiliar já existe em S4 — apenas validar suporte)

- [x] **Confirmar que `POST /dossies/:dossie_id/documentos/upload-url` retorna URL assinada** acessível do mobile:
  - Validar: chamada do simulador iOS/Android com auth header → retorna `{ uploadUrl: string, documentoId: string }` 200
  - Confirmar que CORS permite origin do Expo Go e EAS builds

### Frontend Mobile — T-042

- [x] **Modal de câmera `app/modals/camera.tsx`** — fullscreen, `presentation: 'modal'`:
  - Import: `import { Camera } from 'expo-camera'`
  - Ao abrir: `Camera.requestCameraPermissionsAsync()` — se `status === 'granted'` → mostrar câmera; se `status === 'denied'` → tela de bloqueio com instrução "Abra Configurações → Privacidade → Câmera e ative o Repasse Seguro" + botão "Abrir Configurações" (`Linking.openSettings()`)
  - UI: frame de guia de recorte overlay "Posicione o documento aqui"; botão de captura 64px centralizado; botão "Usar galeria" à esquerda (`expo-image-picker`); botão "Cancelar" (X) 44×44px canto superior esquerdo
  - Após captura: preview da imagem + botão "Usar esta foto" + botão "Tirar novamente"
  - Compressão: `quality: 0.8` antes do upload; se resultado > 10MB após compressão: exibir erro "Imagem muito grande. Tente em menor resolução."
  - Formatos aceitos: JPEG (câmera), PNG, PDF (galeria/document picker via `expo-document-picker`)
  - Safe area: fullscreen intencional — ignorar insets para maximizar área de visualização
  - Validar: permissão concedida → câmera abre; permissão negada → tela de bloqueio; captura → preview → "Usar esta foto" → inicia upload; T-042 coberto em test suite RNTL

- [x] **Upload progress `T-041`** — barra de progresso durante upload:
  - `axios` ou `fetch` com `onUploadProgress` para XMLHttpRequest mobile
  - Componente: barra de progresso `0→100%`; botão "Cancelar" (cancela via `AbortController`)
  - Após sucesso: `DocumentCard` atualiza para status `EM_ANALISE` via `queryClient.invalidateQueries`
  - Após erro: mensagem de erro + botão "Tentar novamente"
  - Validar: upload com arquivo mock 2MB → barra avança; cancelar → `AbortController.abort()` chamado; upload bem-sucedido → DocumentCard atualiza status

- [x] **Galeria como alternativa — `expo-image-picker`**:
  - `ImagePicker.requestMediaLibraryPermissionsAsync()` antes de abrir
  - Configuração: `{ mediaTypes: 'images', quality: 0.8, allowsEditing: true }`
  - Para PDFs: `expo-document-picker` com `type: ['application/pdf']`
  - Validar: selecionar imagem da galeria → mesmo fluxo de upload do T-041; selecionar PDF → upload direto sem preview de câmera

- [x] **`TocavelMinimo` wrapper** — garantia de área mínima de toque em toda UI mobile:
  - Exportado de `components/TocavelMinimo.tsx`
  - Props: `onPress`, `accessibilityLabel`, `children`
  - Implementação: `Pressable` com `style={{ minWidth: 44, minHeight: 44 }}` + `hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}`
  - Aplicado em: todos os botões de ação (câmera, upload, aceite, recusa, assinatura)
  - Validar: inspeção visual — nenhum botão interativo com área < 44px; RN-087 (D01.5) verificado

---

## FEATURE 3 — Biometria e Segurança (REQ-185)

### Frontend Mobile — Biometria

- [x] **`expo-secure-store` para refresh token** — finalizar implementação scaffolded em S2:
  - `apps/mobile-cedente/src/services/auth-storage.ts`:
    - `saveRefreshToken(token: string)`: `SecureStore.setItemAsync('refresh_token', token, { keychainAccessibility: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY })`
    - `getRefreshToken()`: `SecureStore.getItemAsync('refresh_token')`
    - `removeRefreshToken()`: `SecureStore.deleteItemAsync('refresh_token')`
  - Nunca usar AsyncStorage para tokens (não criptografado, visível em backups)
  - Validar: token salvo → recuperável; token removido no logout → `getRefreshToken()` retorna null

- [x] **`reautenticarComBiometria()` — `hooks/useBiometria.ts`**:
  - Fluxo:
    1. `LocalAuthentication.hasHardwareAsync()` — verificar suporte
    2. `LocalAuthentication.isEnrolledAsync()` — verificar configuração
    3. Se sem biometria: fallback → modal de senha da conta
    4. Se com biometria: `LocalAuthentication.authenticateAsync({ promptMessage: 'Confirme sua identidade para continuar', fallbackLabel: 'Usar senha', cancelLabel: 'Cancelar', disableDeviceFallback: false })`
  - Retorna `Promise<boolean>`
  - Validar: dispositivo com Face ID → autenticação biométrica disparada com `promptMessage` correto; dispositivo sem biometria → modal de senha exibido

- [x] **Contextos que exigem re-autenticação biométrica** (D11 seção 6.1):
  - Abertura do app após 30 minutos em background (AppState 'active', `tempoBackground > 30 * 60 * 1000`)
  - Acesso à aba Financeiro (T-020 / T-046)
  - Aceite de proposta (T-037 modal)
  - Solicitação de desistência (T-047 modal)
  - Cancelamento de caso com Escrow ativo (T-023 modal)
  - Validar: mock de 31min background → re-auth disparada ao retornar; toque em "Aceitar Proposta" → re-auth antes de confirmar; biometria bem-sucedida → ação prossegue; biometria cancelada → ação bloqueada + toast "Identidade não confirmada. Ação cancelada."

- [x] **`AppState` — detecção de background e re-autenticação**:
  - `AppState.addEventListener('change', ...)` em `app/_layout.tsx`
  - `nextAppState === 'background'` → salvar `horaEntradaBackground = Date.now()` em memória
  - `nextAppState === 'active'` → calcular `tempoBackground`; se > 30min → `setRequereReautenticacao(true)`; sempre `queryClient.invalidateQueries({ type: 'active' })` ao retornar
  - Validar: app em foreground → sem re-auth; simular 31min background → re-auth modal ao retornar; re-auth bem-sucedida → `setRequereReautenticacao(false)` + navegação normal

---

## FEATURE 4 — Push Notifications

### Backend (novo endpoint)

- [x] **Endpoint `DELETE /cedentes/push-token`** — revogar token ao fazer logout:
  - Body: `{ device_id: string }`
  - Lógica: `UPDATE push_tokens SET ativo = false WHERE cedente_id = :cedente_id AND device_id = :device_id`
  - Chamado no logout mobile antes de `SecureStore.deleteItemAsync('refresh_token')`
  - Validar: token marcado como `ativo = false` após chamada; token inativo não recebe push

- [x] **`PushNotificationService`** — envio de push via Expo Push API:
  - Localização: `apps/api/src/modules/notifications/push.service.ts`
  - Método `sendPush(cedenteId: string, payload: PushPayload)`:
    - Busca tokens ativos: `SELECT token FROM push_tokens WHERE cedente_id = :cedenteId AND ativo = true`
    - Chama `https://exp.host/--/api/v2/push/send` com array de tokens
    - Payload estrutura obrigatória: `{ title, body, data: { tipo, caseId?, proposalId?, envelopeId?, deepLinkUrl } }`
    - `tipo` valores: `'PROPOSTA' | 'DOCUMENTO' | 'ASSINATURA' | 'FINANCEIRO' | 'LEMBRETE'`
    - Tratamento de erro de token inválido (`DeviceNotRegistered`): marcar `ativo = false` automaticamente
  - Integrado ao `NotificationService` existente (S4): quando canal push é selecionado, chama `PushNotificationService.sendPush`
  - Validar: Cedente com token → push enviado para Expo API; token inválido → marcado inativo; Cedente sem token → silencioso (sem erro)

### Frontend Mobile — Push Notifications

- [x] **Canais de notificação Android (API 26+ / Android 8+)** — configurar no boot do app:
  - 6 canais obrigatórios (D11 seção 4.1):
    - `propostas`: nome "Propostas", importância `HIGH`, vibrationPattern `[0, 250, 250, 250]`, lightColor `#0069A8`
    - `documentos`: nome "Documentos", importância `NORMAL`
    - `assinaturas`: nome "Assinaturas", importância `HIGH`
    - `financeiro`: nome "Financeiro", importância `HIGH`
    - `lembretes`: nome "Lembretes", importância `NORMAL`
    - `sistema`: nome "Sistema", importância `LOW`
  - Configurar em `app/_layout.tsx` via `Notifications.setNotificationChannelAsync(...)` condicionado a `Platform.OS === 'android'`
  - Validar: app em Android → 6 canais aparecem em Configurações → Notificações do app

- [x] **Solicitação de permissão de push** — no primeiro acesso ao Dashboard (T-013/T-014), não no onboarding:
  - Lógica: `const { status } = await Notifications.getPermissionsAsync()`; se não `'granted'` e não `'denied'`: `await Notifications.requestPermissionsAsync()`
  - iOS: popup do sistema solicitado automaticamente
  - Android ≤ API 32: automático, sem solicitação
  - Android ≥ API 33: `requestPermissionsAsync()` solicitado
  - Se negado: não reperguntar; instruir em configurações se usuário tentar habilitar manualmente
  - Validar: primeira abertura do Dashboard → popup de permissão no iOS; permissão concedida → `registrarTokenDispositivo()` chamado; permissão negada → silencioso

- [x] **`registrarTokenDispositivo()`** — após permissão concedida:
  - `Notifications.getExpoPushTokenAsync({ projectId: Constants.expoConfig?.extra?.eas?.projectId })`
  - `api.post('/cedentes/push-token', { token: token.data, plataforma: Platform.OS, device_id: await getUniqueId() })`
  - Token rotacionado: verificar e atualizar a cada boot do app (mesmo token → `upsert` idempotente)
  - Um Cedente pode ter múltiplos tokens (múltiplos dispositivos)
  - Validar: token obtido → `POST /cedentes/push-token` chamado; segundo boot → `upsert` sem duplicar row; múltiplos dispositivos → múltiplas rows distintas por `device_id`

- [x] **Deep link ao toque em notificação** — tratamento em background/fechado (D11 seção 10.3):
  - `Notifications.addNotificationResponseReceivedListener`: se `deepLinkUrl` e `isAuthenticated` → `router.push(deepLinkUrl)`; se não autenticado → `setDeepLinkPendente(deepLinkUrl)` no Zustand + `router.push('/(auth)/login')` + redirecionar após login bem-sucedido
  - Esquema de deep links obrigatórios:
    - `repasse-seguro://casos/{caseId}` → T-016
    - `repasse-seguro://casos/{caseId}/documentos` → T-017
    - `repasse-seguro://propostas/{proposalId}` → T-036
    - `repasse-seguro://assinaturas/{envelopeId}` → T-043 → T-044
    - `repasse-seguro://financeiro/{caseId}` → T-046
    - `repasse-seguro://cadastrar` → T-024 ou T-030 se rascunho
    - `repasse-seguro://guardiao` → T-048
  - Validar: notificação de proposta → toque → navega para T-036 com proposta correta; app fechado → abre → navega direto; usuário não autenticado → login → redirecionado para destino original

---

## FEATURE 5 — ZapSign WebView Mobile (REQ-220)

### Frontend Mobile — T-044

- [x] **Modal ZapSign `app/modals/zapsign.tsx`** — WebView fullscreen via `react-native-webview`:
  - Documentos suportados para assinatura mobile: `TERMO_CADASTRO` e `INSTRUMENTO_CESSAO` apenas
  - Documentos não suportados no mobile (`TERMO_ACEITE_ESCALONAMENTO`, `TERMO_COMERCIAL`): exibir mensagem "Assine pelo computador em repasseseguro.com.br" + botão "Copiar link"
  - Configuração WebView:
    ```typescript
    <WebView
      source={{ uri: zapSignUrl }}
      onMessage={(event) => {
        const data = JSON.parse(event.nativeEvent.data)
        if (data.type === 'zapsign:signed') {
          onAssinado(envelopeId)
          router.back()
        }
      }}
      injectedJavaScript={`
        window.addEventListener('message', (e) => {
          window.ReactNativeWebView.postMessage(JSON.stringify(e.data))
        })
        true;
      `}
      startInLoadingState={true}
      renderLoading={() => <SkeletonFullPage />}
      onError={() => setZapSignError(true)}
    />
    ```
  - Botão "Fechar" flutuante 44×44px, canto superior direito, visível sobre a WebView
  - Timeout de carregamento: 10 segundos → exibir mensagem "Não foi possível carregar o documento. Verifique sua conexão." + botão "Tentar novamente"
  - Ao `zapsign:signed` postMessage: fechar modal + toast "Documento assinado com sucesso!" + invalidar query de envelopes
  - Validar: TERMO_CADASTRO → WebView abre com URL ZapSign; postMessage `zapsign:signed` → toast + modal fecha; timeout 10s → mensagem de erro + retry; TERMO_ACEITE_ESCALONAMENTO → mensagem "Assine pelo computador"

- [x] **Lista de Assinaturas T-043** — `app/(tabs)/mais/assinaturas.tsx`:
  - Listar envelopes por `cedente_id` via `GET /assinaturas`
  - Para cada envelope: mostrar tipo (`TERMO_CADASTRO`, `INSTRUMENTO_CESSAO`, etc.), status (`PENDENTE`/`ASSINADO`/`CANCELADO`), data de criação
  - Para envelopes `PENDENTE` com tipo suportado mobile → botão "Assinar" → `router.push('/modals/zapsign?envelopeId=...')`
  - Para envelopes `PENDENTE` com tipo não suportado mobile → texto "Assine pelo computador"
  - Para envelopes `ASSINADO` → badge verde "Assinado"
  - Validar: lista carrega via API real; TERMO_CADASTRO pendente → botão "Assinar"; TERMO_COMERCIAL pendente → "Assine pelo computador"

---

## FEATURE 6 — Comportamento Offline (REQ-219)

### Frontend Mobile — Cache e Offline

- [x] **TanStack Query — configuração de cache offline** para 6 telas (D11 seção 5.1):
  - `T-013/T-014 Dashboard`: `staleTime: 30 * 60 * 1000`, `gcTime: 24 * 60 * 60 * 1000`, `networkMode: 'offlineFirst'`
  - `T-015 Lista de Casos` (primeira página, 10 casos): `staleTime: 30 * 60 * 1000`, `gcTime: 24h`, `networkMode: 'offlineFirst'`
  - `T-016 Detalhe do Caso`: `staleTime: 15 * 60 * 1000`, `gcTime: 24h`, `networkMode: 'offlineFirst'`
  - `T-035 Lista de Propostas ativas`: `staleTime: 15 * 60 * 1000`, `gcTime: 24h`, `networkMode: 'offlineFirst'`
  - `T-040 Documentos` (checklist + status): `staleTime: 30 * 60 * 1000`, `gcTime: 24h`, `networkMode: 'offlineFirst'`
  - `T-050 Perfil` (dados pessoais): `staleTime: 60 * 60 * 1000`, `gcTime: 24h`, `networkMode: 'offlineFirst'`
  - Validar: carregar Dashboard online → modo avião → reabrir app → dados em cache exibidos; banner âmbar visível

- [x] **Bloqueio de ações de escrita offline**:
  - Upload de documento (T-041/T-042): `useNetworkStatus` → se `!isOnline` → botão câmera desabilitado + tooltip "Sem conexão. Conecte-se para enviar documentos."
  - Aceite/Recusa/Contraproposta (T-037/T-038/T-039): se `!isOnline` → botões desabilitados + mensagem "Sem conexão. Conecte-se para responder."
  - Assinatura ZapSign (T-044): se `!isOnline` → botão "Assinar" desabilitado + tooltip "Sem conexão"
  - Upload bloqueado (T-040): tooltip no DocumentCard "Sem conexão"
  - Validar: modo avião → todas as ações de escrita desabilitadas; mensagens de offline visíveis; reconectar → ações habilitadas automaticamente

- [x] **Sincronização ao reconectar**:
  - `TanStack Query`: ao `isOnline` mudar de `false` para `true` → `queryClient.invalidateQueries({ type: 'active' })` automaticamente
  - Badge "Atualizado às HH:MM" (timestamp do cache): exibido quando dados em cache; removido após sincronização bem-sucedida
  - Não implementar fila de ações de escrita offline nesta versão (v1.0 — DECISÃO AUTÔNOMA conforme D11)
  - Validar: desligar rede → badge timestamp aparece; religar → `invalidateQueries` dispara → dados frescos → badge some

---

## FEATURE 7 — Performance e Gestos Nativos

### Frontend Mobile — Performance

- [x] **FlatList com `getItemLayout`** em listas de Casos, Propostas e Documentos:
  - `CASE_CARD_HEIGHT`, `PROPOSAL_CARD_HEIGHT`, `DOCUMENT_CARD_HEIGHT` de `constants/layout.ts`
  - Configuração obrigatória em todas as FlatLists:
    ```typescript
    getItemLayout={(_, index) => ({ length: CARD_HEIGHT, offset: CARD_HEIGHT * index, index })}
    maxToRenderPerBatch={10}
    windowSize={10}
    removeClippedSubviews={true}
    initialNumToRender={5}
    ```
  - Validar: scroll de lista com 50 casos → 60 FPS constante (medido via Flipper); sem `getItemLayout` → falha na revisão

- [x] **Memoização obrigatória** de componentes de lista:
  - `ProposalCard`: `memo` com comparador `prevProps.proposta.status === nextProps.proposta.status`
  - `DocumentCard`: `memo` com comparador `prevProps.documento.status === nextProps.documento.status`
  - `CaseCard`: `memo` com comparador `prevProps.caso.status === nextProps.caso.status`
  - Validar: adicionar item à lista → apenas o novo item re-renderiza (verificar com React DevTools Profiler)

- [x] **`expo-image`** em vez de `<Image>` do React Native em todos os componentes com imagem:
  - Import: `import { Image } from 'expo-image'`
  - Configurar lazy loading em listas de documentos (thumbnails)
  - Validar: import de `react-native` `Image` → erro de lint (`no-react-native-image` rule); `expo-image` em uso

- [x] **Haptics em ações críticas** (D11 seção 3.3):
  - Aceite de proposta (T-037): `Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success)`
  - Recusa de proposta (T-038): `Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium)`
  - Erro de sistema: `Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error)`
  - Ação destrutiva (cancelamento T-022/T-023, desistência T-047): `Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy)`
  - Validar: aceite → haptic success dispara; cancelamento → haptic heavy dispara; dispositivos sem haptic → silencioso sem erro

- [x] **Gestos nativos** (D11 seção 7.1):
  - Pull-to-refresh em `T-015`, `T-035`, `T-013`, `T-040` via `RefreshControl` com `tintColor: tokens.primary` (iOS) e `colors: [tokens.primary]` (Android)
  - Long press em `DocumentCard` → menu de ações contextuais (re-enviar, cancelar envio)
  - Swipe-back (iOS): configurado automaticamente via expo-router; não interceptar em telas de câmera (T-042) e ZapSign (T-044)
  - Scale press `0.95` em cards e botões via `Animated` ou `Reanimated`
  - Validar: pull-to-refresh em lista de casos → spinner aparece → dados recarregam; long press em DocumentCard → menu aparece; swipe-back em T-016 → volta para T-015

### Frontend Mobile — Safe Areas e Plataformas

- [x] **Safe areas** em todos os layouts (D11 seção 7.3):
  - `useSafeAreaInsets()` em: bottom tab bar (insets.bottom), modals (insets.top), header (insets.top + statusBar)
  - T-042 câmera: fullscreen intencional — sem safe area para maximizar área de visualização do documento
  - `KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : 'height'}` em todos os formulários
  - Validar: iPhone 15 Pro (Dynamic Island) → sem sobreposição de conteúdo; iPhone SE → layout não quebra; Galaxy S23 → navigation bar respeitada

- [x] **Código por plataforma** via `Platform.select` (D11 seção 9.2):
  - Sombras: iOS → `shadowColor/shadowOffset/shadowOpacity/shadowRadius`; Android → `elevation: 4`
  - Modals: iOS → sheets com `presentation: 'modal'` e swipe-to-dismiss; Android → bottom sheets via `react-native-reanimated`
  - Status bar: `expo-status-bar` `style="auto"` em ambas as plataformas
  - Font: Inter Variable customizada + San Francisco fallback (iOS) / Roboto fallback (Android)
  - Validar: iOS → sombras com shadow props; Android → elevation; sem console.log de `Platform.OS` em prod

---

## FEATURE 8 — Telas Principais Mobile

### Frontend Mobile — Telas

- [x] **T-013/T-014 Dashboard mobile** — `app/(tabs)/dashboard.tsx`:
  - Mesmos 4 KPI cards do web (S3): casos ativos, pendências, propostas recebidas, valor líquido estimado
  - Estado vazio (primeiro acesso): boas-vindas + checklist 3 passos + botão "Cadastrar meu primeiro imóvel" + link Guardião
  - Layout single-column (sem split view)
  - Pull-to-refresh configurado
  - Atualização por `staleTime` 30min + invalidação ao `AppState 'active'`
  - Validar: dados carregam via API real; estado vazio exibido sem casos; pull-to-refresh dispara refetch

- [x] **T-015 Lista de Casos mobile** — `app/(tabs)/casos/index.tsx`:
  - FlatList com `getItemLayout` + `CaseCard` memoizado
  - Pull-to-refresh
  - Filtros por status acessíveis via chips scrolláveis horizontalmente
  - Busca por nome/endereço (debounce 300ms)
  - Paginação: carregar mais ao rolar para o final (`onEndReached`)
  - Validar: lista carrega primeiros 10 casos; scroll até final → carrega próximos 10; filtro por status → lista atualiza; busca → debounce funciona

- [x] **T-016 Detalhe do Caso mobile** — `app/(tabs)/casos/[id]/index.tsx`:
  - 5 abas navegáveis: Resumo, Documentos, Propostas, Financeiro, Histórico
  - Abas como TabBar nativa (não web tabs)
  - Aba Financeiro (T-020): exige re-autenticação biométrica antes de exibir
  - Validar: navegação entre 5 abas; aba Financeiro → re-auth disparada; dados em cache offline exibidos corretamente

- [x] **T-035/T-036 Propostas mobile** — `app/(tabs)/propostas/[id].tsx`:
  - Lista de propostas: cards em coluna única (sem split view)
  - Toque em card → T-036 (sem split view lateral)
  - Ações (Aceitar/Recusar/Contraproposta): bottom sheet ao toque via `PropostaAcoesSheet`
  - T-037 Aceitar: re-auth biométrica + confirmação dupla com resumo financeiro
  - T-038 Recusar: confirmação simples
  - T-039 Contraproposta: validação de piso (valor_pago como mínimo provisório — PEND-08)
  - Timer regressivo dinâmico: últimas 24h → exibir horas; última hora → exibir minutos + pulsação visual
  - Cessionário: NUNCA exibir nome, CPF, e-mail — apenas número sequencial (RN-012)
  - Validar: T-037 → re-auth → confirmação → aceite → status ACEITA na lista; T-038 → recusa sem re-auth; T-039 → valor abaixo do piso → alerta bloqueante; cessionário_id ausente em toda UI

- [x] **T-040/T-017 Documentos mobile** — `app/(tabs)/documentos.tsx`:
  - Checklist de documentos com status de cada um
  - Botão câmera inline no DocumentCard → T-042
  - Long press → menu de ações contextuais
  - Upload bloqueado se `!isOnline`
  - Validar: DocumentCard com status PENDENTE → botão câmera visível; DocumentCard VERIFICADO → sem botão câmera; offline → câmera desabilitada

- [x] **T-048 Chat Guardião mobile** — `app/(tabs)/mais/guardiao.tsx`:
  - Chat com Vercel AI SDK (SSE stream) — mesmo endpoint `POST /ai/guardiao/message`
  - 4 botões de ação rápida (mesmos do web S7)
  - Rate limit counter (30 msg/hora) visível
  - Layout adaptado para teclado mobile: `KeyboardAvoidingView`
  - Validar: mensagem enviada → resposta streaming aparece; rate limit atingido → botão desabilitado + contador

---

## FEATURE 9 — Testes Mobile

### Testes

- [x] **Testes RNTL (React Native Testing Library) — casos críticos**:
  - **T-042 Upload câmera**: mock de `expo-camera` + `Camera.requestCameraPermissionsAsync` → `granted` → câmera renderiza; mock `denied` → tela de bloqueio renderiza
  - **T-037 Aceite proposta mobile**: mock `expo-local-authentication.authenticateAsync` → `success: true` → aceite processado; mock `success: false` → aceite bloqueado + toast "Identidade não confirmada"
  - **T-039 Contraproposta mobile**: input com valor abaixo do piso → mensagem de erro bloqueante; valor acima → formulário submit
  - **T-016 Acompanhamento status**: caso em `PROPOSTA_RECEBIDA` → badge e próximo passo corretos exibidos
  - Validar: `pnpm test --filter=mobile-cedente` → 100% dos testes acima passando

- [x] **Testes E2E Detox — 7 fluxos obrigatórios** (D11 seção 12.2):
  1. Cadastro completo: T-002 → T-003 → ativação → T-013
  2. Upload por câmera: T-040 → T-042 → upload → DocumentCard "Em Análise"
  3. Aceitar proposta: T-035 → T-036 → T-037 → toast de sucesso
  4. Recusar proposta: T-035 → T-038 → toast de sucesso
  5. Contraproposta: T-035 → T-039 → validação de piso → envio
  6. Notificação push → deep link: notificação de proposta → toque → T-036
  7. Offline → reconexão: desligar rede → navegar → reativar → sincronização
  - Validar: `pnpm detox test --configuration ios.sim.release` → 7 testes passando; `pnpm detox test --configuration android.emu.release` → 7 testes passando

- [x] **Teste de acessibilidade mobile** (D11 seção 11):
  - Todos os botões de ação têm `accessibilityLabel` descritivo (ex: "Aceitar proposta de R$ 45.000,00")
  - Ícones sem texto têm `accessibilityRole="button"` + `accessibilityLabel`
  - Valores financeiros: `accessibilityLabel="R$ 45 mil e 500 reais"` (legível por TalkBack/VoiceOver)
  - Nenhum elemento interativo com área < 44×44px (`TocavelMinimo` aplicado)
  - Contraste: `#0069A8` sobre `#FFFFFF` ratio 5.1:1 ✅ (conforme D03)
  - Validar: ativar VoiceOver iOS → navegar entre tabs → labels anunciados corretamente; ativar TalkBack Android → mesma verificação

- [x] **Matriz de dispositivos — validação manual pré-release** (D11 seção 1.3):
  - iPhone 15 Pro (iOS 17): câmera, Face ID, Dynamic Island safe area
  - iPhone 13 (iOS 16): versão mínima iOS — sem Dynamic Island
  - iPhone SE 3a gen (iOS 16): tela 375pt — layout compacto
  - Samsung Galaxy S23 (Android 13 / API 33): câmera, permissão push solicitada
  - Samsung Galaxy A53 (Android 12 / API 32): push automático, mid-range performance
  - Moto G Pure (Android 10 / API 29): versão mínima Android — hardware limitado
  - Validar: checklist manual preenchido para cada dispositivo antes de criar build de release

- [x] **EAS Build configurado** — `eas.json`:
  - Profile `development`: `distribution: 'internal'`; Android: `buildType: 'apk'`; iOS: `simulator: true`
  - Profile `preview`: `distribution: 'internal'`; ambas plataformas
  - Profile `production`: `distribution: 'store'`; ambas plataformas
  - `deploy-mobile.yml` GitHub Actions: push para `main` + tag `v*.*.*` → EAS Build production automático
  - Validar: `eas build --profile preview --platform all` executa sem erro; Sentry configurado em todas as builds (D25 — REQ-206)

---

## 🔀 Cross-Módulo

- [x] **Registro de token push no login mobile** — após `POST /auth/login` bem-sucedido no mobile:
  - Chamar `registrarTokenDispositivo()` imediatamente após salvar refresh token em SecureStore
  - Se permissão push ainda não solicitada: `registrarTokenDispositivo()` adiado para primeiro acesso ao Dashboard
  - Validar: login bem-sucedido → `POST /cedentes/push-token` chamado dentro de 2s

- [x] **Revogação de token push no logout mobile** — antes de `DELETE /auth/session`:
  - Chamar `DELETE /cedentes/push-token` com `device_id` atual
  - Remover refresh token do SecureStore
  - Validar: logout → token marcado `ativo = false` no banco; segundo login no mesmo dispositivo → token reativado via upsert

- [x] **NotificationService S4 → PushService S8** — integração bidirecional:
  - Quando `NotificationService.enviar(cedenteId, template, canal: 'push')` é chamado (S4): chamar `PushNotificationService.sendPush(cedenteId, payload)` implementado nesta sprint
  - `deepLinkUrl` no payload deve corresponder ao esquema `repasse-seguro://` definido nesta sprint
  - Validar: nova proposta no sistema → `NOT-CED-04` dispara → push enviado ao Cedente → toque na notificação → T-036 aberto com proposta correta

---

## Auto-Verificação (12 Checks)

- [x] **Check #1 — Nomenclatura exata**: `expo-secure-store`, `expo-camera`, `expo-image-picker`, `expo-document-picker`, `expo-local-authentication`, `expo-haptics`, `@react-native-community/netinfo`, `expo-web-browser`, `react-native-webview` — todos usados com nomes corretos conforme D11 seção 1.2
- [x] **Check #2 — Versões**: React Native 0.76+, Expo SDK 52+, expo-router 4.x, React Native Reanimated 3.x, React Native Gesture Handler 2.x, TanStack Query 5.x, Zustand 5.x — todos especificados em `package.json` sem divergência de D11 seção 1.2
- [x] **Check #3 — Plataformas**: iOS 16+ (API Level —) e Android 10 (API 29+) — nenhum código usa API nativa incompatível com versões mínimas; `Platform.select` usado onde há diferença (D11 seção 9.2)
- [x] **Check #4 — REQ-185 biometria**: `expo-local-authentication` + `expo-secure-store`; 5 contextos de re-auth implementados; fallback para senha da conta quando sem biometria (D18 seção 7)
- [x] **Check #5 — REQ-219 offline**: exatamente 6 telas com cache local (T-013/T-014, T-015, T-016, T-035, T-040, T-050); ações de escrita bloqueadas com mensagem "Sem conexão"; `networkMode: 'offlineFirst'` em todas (D11 seção 5.1)
- [x] **Check #6 — REQ-220 ZapSign**: apenas `TERMO_CADASTRO` e `INSTRUMENTO_CESSAO` suportados mobile; `TERMO_ACEITE_ESCALONAMENTO` e `TERMO_COMERCIAL` redirecionam para web (D11 seção 3.2)
- [x] **Check #7 — RN-087 touch targets**: nenhum elemento interativo com área < 44×44px; `TocavelMinimo` wrapper usado em todos os botões de ação; `hitSlop: { top: 8, bottom: 8, left: 8, right: 8 }` (D01.5)
- [x] **Check #8 — REQ-218 navegação**: Stack raiz → Tab navigator (5 tabs) → Stack modals; expo-router file-based; todas as rotas do D11 seção 2.1 implementadas; `+not-found.tsx` presente (D11 seção 2.1)
- [x] **Check #9 — Anonimato cessionário**: `cessionario_id`, nome, CPF, e-mail, telefone do Cessionário NUNCA aparecem em nenhuma tela mobile (RN-012); apenas número sequencial da proposta (D01.1)
- [x] **Check #10 — Hermes engine**: `"jsEngine": "hermes"` em `app.json`; habilitado em todos os profiles EAS Build (D11 seção 8.3)
- [x] **Check #11 — Deep links**: 7 rotas de deep link implementadas com esquema `repasse-seguro://`; `associatedDomains` iOS + `intentFilters` Android configurados; tratamento de app fechado + usuário não autenticado (D11 seção 10)
- [x] **Check #12 — REQs cobertos**: REQ-132 (RN-087 44px touch targets ✅), REQ-185 (biometria + SecureStore ✅), REQ-216 (iOS 16+/Android 10/RN 0.76+/Expo 52+/expo-router 4 ✅), REQ-217 (câmera T-042/ZapSign T-044/propostas T-037/status T-016 ✅), REQ-218 (5 tabs + Stack modals ✅), REQ-219 (6 telas offline ✅), REQ-220 (ZapSign WebView TERMO_CADASTRO + INSTRUMENTO_CESSAO ✅)

---

## Itens Pendentes de Revisão Manual

- **⚠️ PEND-08 replicado** — `valor_pago` como piso provisório para contraproposta mobile (T-039): usar `valor_pago` até endpoint de piso do Admin definido. `[REVISÃO MANUAL]`
- **⚠️ PEND-12** — Detox E2E para câmera (T-042) requer dispositivo físico ou emulador com câmera simulada. Em CI/CD, substituir por mock de câmera via `jest.mock('expo-camera')`. Fluxo E2E completo apenas em dispositivo físico. `[REVISÃO MANUAL]`
- **⚠️ AMBÍGUO** — D11 seção 12.1 menciona "Detox" para E2E; outros docs do projeto mencionam Playwright+RNTL. Adotado Detox para E2E nativo mobile conforme D11 (doc mais específico para mobile). `[REVISÃO MANUAL]`
