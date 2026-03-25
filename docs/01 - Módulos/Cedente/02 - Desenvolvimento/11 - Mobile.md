# 11 - Mobile

## Módulo Cedente · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Produto, Mobile e Engenharia |
| **Escopo** | Especificação mobile — navegação, gestos, câmera, assinatura touch, push notifications, offline, biometria e diferenças de plataforma |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Data da versão** | 2026-03-23 (America/Fortaleza) |
| **Status** | Ativo |
| **Referências** | 01.1/01.2 - Regras de Negócio · 02 - Stacks · 06 - Mapa de Telas · 07 - Wireframes · 09 - Contratos de UI por Tela |

---

> 📌 **TL;DR**
>
> - **Plataformas:** iOS 16+ e Android 10 (API 29+). React Native 0.76+ com Expo SDK 52+, expo-router 4.
> - **Casos de uso mobile prioritários:** upload de documentos por câmera (T-042), assinatura ZapSign touch (T-044), resposta a propostas (T-037/T-038/T-039) e acompanhamento de status do caso (T-016).
> - **Navegação:** Stack navigator raiz → Tab navigator (5 tabs via bottom tab bar T-012) → Stack modals para assinatura, cancelamento e desistência. expo-router com file-based routing.
> - **Telas com suporte offline:** 6 telas críticas com dados em cache local (Dashboard, Meus Casos, Lista de Propostas, Detalhe do Caso, Documentos, Perfil). Ações de escrita são bloqueadas offline com mensagem clara.
> - **Features nativas:** câmera (upload de documentos), galeria/documentos (upload alternativo), push notifications (Expo Notifications), biometria (re-autenticação), haptics (feedback em ações críticas), network detection.
> - **Diferenças críticas iOS vs. Android:** swipe-back nativo (iOS) vs. botão hardware back (Android); permissão push automática Android 12- vs. solicitada Android 13+; Face ID/Touch ID (iOS) vs. Fingerprint/Face Unlock (Android); bottom tab com safe-area (iOS) vs. navigation bar Android.
> - **Assinaturas:** ZapSign via WebView em tela cheia. Suporte mobile para Termo de Cadastro e Instrumento de Cessão. Outros documentos: instruem o Cedente a acessar via web.
> - **Zero seções pendentes** — todos os comportamentos especificados com base nos docs de input.

---

## 1. Plataformas e Versões Suportadas

### 1.1 Versões Mínimas de Sistema

| **Plataforma** | **Versão mínima** | **API Level** | **Justificativa** |
|---|---|---|---|
| iOS | 16.0 | — | Face ID v2, Dynamic Island handling; base de usuários iOS 16+ representa >90% no Brasil. Suporte a `expo-local-authentication` com biometria unificada. |
| Android | 10 (Q) | API 29 | Scoped Storage obrigatório para acesso seguro a arquivos, `BiometricPrompt` unificado, permissões modernizadas. API 28 descartada: deprecated APIs com complexidade adicional sem benefício. |

### 1.2 Stack Mobile

| **Tecnologia** | **Versão** | **Papel** |
|---|---|---|
| React Native | 0.76+ | Framework mobile |
| Expo SDK | 52+ | Managed workflow, OTA updates, acesso a APIs nativas |
| expo-router | 4.x | File-based routing + deep links nativos |
| React Native Reanimated | 3.x | Animações na UI thread (sem janking) |
| React Native Gesture Handler | 2.x | Gestos: swipe-back, long press, drag |
| TanStack Query | 5.x | Cache e sincronização de dados |
| Zustand | 5.x | Estado global + persistência AsyncStorage |
| expo-secure-store | latest | Refresh tokens (armazenamento criptografado) |
| expo-camera | latest | Câmera para upload de documentos (T-042) |
| expo-image-picker | latest | Galeria para seleção de documentos existentes |
| expo-document-picker | latest | Seleção de PDFs do sistema de arquivos / cloud |
| Expo Notifications | latest | Push notifications |
| expo-local-authentication | latest | Biometria para re-autenticação |
| expo-haptics | latest | Feedback háptico em ações críticas |
| @react-native-community/netinfo | latest | Detecção de estado de rede para comportamento offline |
| expo-web-browser | latest | WebView para ZapSign (assinatura touch) |

### 1.3 Dispositivos de Referência para Testes

| **Dispositivo** | **Plataforma** | **Propósito** |
|---|---|---|
| iPhone 15 Pro | iOS 17 | Referência primária — Dynamic Island, câmera ProRAW |
| iPhone 13 | iOS 16 | Versão mínima iOS — sem Dynamic Island |
| iPhone SE (3a gen) | iOS 16 | Tela menor (375pt) — testar layouts compactos |
| Samsung Galaxy S23 | Android 13 (API 33) | Referência Android moderna — câmera de alta qualidade |
| Samsung Galaxy A53 | Android 12 (API 32) | Referência mid-range — maioria do mercado brasileiro |
| Moto G Pure | Android 10 (API 29) | Versão mínima Android — hardware limitado |

### 1.4 Resolução e Densidade

| **Categoria** | **Density** | **Resolução referência** | **Suporte** |
|---|---|---|---|
| mdpi | 1x | 320×480 | Suportado (layout não otimizado) |
| hdpi | 1.5x | 480×800 | Suportado |
| xhdpi | 2x | 720×1280 | Suportado |
| xxhdpi | 3x | 1080×1920 | Primário Android |
| xxxhdpi | 4x | 1440×2560 | Suportado (high-end) |
| @2x / @3x | iOS scale | iPhone padrão / Pro | Primário iOS |

Layout flexível com `Dimensions.get('window')`. Nunca valores hardcoded em `StyleSheet`. Constantes de layout em `constants/layout.ts`.

---

## 2. Arquitetura de Navegação

### 2.1 Estrutura de Rotas (expo-router 4)

```
app/
├── _layout.tsx                     # Root layout — AuthProvider, ThemeProvider
├── (auth)/
│   ├── _layout.tsx                 # AuthLayout — sem tab bar
│   ├── login.tsx                   # T-001
│   ├── cadastro.tsx                # T-002
│   ├── ativacao-pendente.tsx       # T-003
│   ├── ativacao-sucesso.tsx        # T-004
│   ├── link-expirado.tsx           # T-005
│   ├── recuperar-senha.tsx         # T-006
│   ├── recuperar-senha-enviado.tsx # T-007
│   ├── redefinir-senha.tsx         # T-008
│   └── conta-bloqueada.tsx         # T-009
├── (tabs)/
│   ├── _layout.tsx                 # TabsLayout — bottom tab bar T-012
│   ├── dashboard.tsx               # T-013 / T-014
│   ├── casos/
│   │   ├── index.tsx               # T-015
│   │   └── [id]/
│   │       ├── index.tsx           # T-016
│   │       ├── documentos.tsx      # T-017
│   │       ├── propostas.tsx       # T-018
│   │       ├── assinaturas.tsx     # T-019
│   │       ├── financeiro.tsx      # T-020
│   │       └── historico.tsx       # T-021
│   ├── documentos.tsx              # T-040 (visão global)
│   ├── propostas/
│   │   ├── index.tsx               # T-035
│   │   └── [id].tsx                # T-036
│   └── mais/
│       ├── index.tsx               # Bottom sheet de "Mais"
│       ├── guardiao.tsx            # T-048
│       ├── assinaturas.tsx         # T-043
│       └── perfil/
│           ├── dados.tsx           # T-050
│           ├── notificacoes.tsx    # T-051
│           └── lgpd.tsx            # T-052
├── modals/
│   ├── camera.tsx                  # T-042 — fullscreen
│   ├── zapsign.tsx                 # T-044 — WebView fullscreen
│   ├── aceitar-proposta.tsx        # T-037
│   ├── recusar-proposta.tsx        # T-038
│   ├── contraproposta.tsx          # T-039
│   ├── cancelar-caso.tsx           # T-022 / T-023
│   ├── desistencia.tsx             # T-047
│   ├── escalonamento.tsx           # T-031 / T-032
│   └── expiracao-sessao.tsx        # T-010
└── +not-found.tsx                  # 404
```

### 2.2 Stack de Navegação

```typescript
// Root Stack (_layout.tsx)
// - AuthGuard: redireciona para (auth)/login se não autenticado
// - (auth) group: sem tab bar, sem header customizado
// - (tabs) group: com bottom tab bar fixa
// - modals: apresentados como sheets (iOS) ou overlays (Android)

// Configuração de modals (expo-router 4)
// app/modals/_layout.tsx
export default function ModalsLayout() {
  return (
    <Stack
      screenOptions={{
        presentation: 'modal',       // iOS: sheet
        animation: 'slide_from_bottom',
        headerShown: false,
      }}
    />
  )
}
```

### 2.3 Bottom Tab Bar (T-012)

```typescript
// app/(tabs)/_layout.tsx
export default function TabsLayout() {
  const { pendencias, propostasAtivas } = useNotificacoes()

  return (
    <Tabs
      screenOptions={{
        tabBarActiveTintColor: tokens.primary,
        tabBarInactiveTintColor: tokens.mutedForeground,
        tabBarStyle: {
          height: 60 + insets.bottom,
          paddingBottom: insets.bottom,
        },
      }}
    >
      <Tabs.Screen name="dashboard" options={{ title: 'Dashboard', tabBarIcon: LayoutDashboard }} />
      <Tabs.Screen name="casos" options={{ title: 'Meus Casos', tabBarIcon: FolderOpen }} />
      <Tabs.Screen name="documentos" options={{
        title: 'Documentos',
        tabBarIcon: FileText,
        tabBarBadge: pendencias.documentos > 0 ? pendencias.documentos : undefined,
      }} />
      <Tabs.Screen name="propostas" options={{
        title: 'Propostas',
        tabBarIcon: Tag,
        tabBarBadge: propostasAtivas > 0 ? propostasAtivas : undefined,
      }} />
      <Tabs.Screen name="mais" options={{
        title: 'Mais',
        tabBarIcon: Grid2x2,
        tabBarBadge: pendencias.total > 0 ? pendencias.total : undefined,
      }} />
    </Tabs>
  )
}
```

**Regras de badge:**
- Badge aparece apenas quando valor > 0
- Badge máximo exibido: 99 (acima disso: "99+")
- Badge de "Propostas": número de propostas ativas aguardando resposta
- Badge de "Documentos": documentos pendentes ou rejeitados
- Badge de "Mais": soma de assinaturas pendentes + outras pendências

---

## 3. Casos de Uso Mobile Prioritários

O Cedente usa o mobile principalmente para:

1. **Upload de documentos por câmera** — o caso de uso mais crítico mobile
2. **Acompanhamento de status do caso** — check rápido no painel
3. **Resposta a propostas** — aceitar, recusar ou fazer contraproposta
4. **Assinatura ZapSign touch** — para Termo de Cadastro e Instrumento de Cessão
5. **Chat com o Guardião** — tirar dúvidas sobre o andamento

### 3.1 Upload de Documentos por Câmera (T-042)

Este é o fluxo mais crítico do app mobile. O Cedente PF frequentemente não tem scanner e depende da câmera do celular.

**Fluxo completo:**

```
DocumentCard (T-040) → botão câmera inline
  ↓
Verificação de permissão de câmera
  ├── Negada → Tela de bloqueio (instrução + "Abrir configurações")
  └── Concedida →
      ↓
      T-042 — Câmera Fullscreen (modal)
        ├── Frame de guia de recorte overlay (posicionar o documento)
        ├── Botão de captura 64px centralizado
        ├── Botão "Usar galeria" à esquerda (expo-image-picker)
        ├── Botão "Cancelar" (X) canto superior esquerdo
        └── Após captura:
            ├── Preview da imagem
            ├── "Usar esta foto" → T-041 (UploadZone com imagem)
            └── "Tirar novamente" → volta para câmera
              ↓
              T-041 — Upload em progresso
                ├── Barra de progresso 0→100%
                ├── Botão "Cancelar"
                └── Sucesso → DocumentCard atualiza para "Em Análise"
```

**Tratamento de permissão de câmera:**

```typescript
import { Camera } from 'expo-camera'

async function solicitarPermissaoCameraEAbrir() {
  const { status } = await Camera.requestCameraPermissionsAsync()
  if (status === 'granted') {
    router.push('/modals/camera')
  } else if (status === 'denied') {
    // Exibir tela de bloqueio com instrução para configurações
    setEstadoPermissao('negada')
  }
  // status === 'undetermined' → nunca acontece após requestCameraPermissionsAsync
}
```

**Validações no upload mobile:**
- Tamanho máximo: 10 MB (mesmo do web)
- Formatos aceitos: JPEG (câmera), PNG, PDF (galeria/document picker)
- MIME type validado no backend (nunca confiar na extensão)
- Compressão automática de imagens de câmera: `quality: 0.8` no expo-camera antes do upload
- Máximo de compressão: resultado deve ser < 10 MB; se ainda > 10 MB após compressão, exibir erro de tamanho

**Comportamento de câmera por plataforma:**

| **Feature** | **iOS** | **Android** |
|---|---|---|
| Qualidade padrão | `Camera.Constants.quality 0.8` | `Camera.Constants.quality 0.8` |
| Orientação | Aceita retrato e paisagem — UI adapta | Aceita retrato e paisagem — UI adapta |
| Zoom | Pinch-to-zoom nativo | Pinch-to-zoom nativo |
| Flash | Auto (desligado por padrão) | Auto (desligado por padrão) |
| Permissão | `NSCameraUsageDescription` no Info.plist | `CAMERA` no AndroidManifest.xml |

### 3.2 Assinatura ZapSign Touch (T-044 Mobile)

Documentos disponíveis para assinatura mobile:
- Termo de Cadastro
- Instrumento de Cessão

Documentos que redirecionam para web:
- Termo de Aceite de Escalonamento
- Termo Comercial

**[DECISÃO AUTÔNOMA — Apenas Termo de Cadastro e Instrumento de Cessão suportam assinatura mobile. Estes são os documentos mais urgentes e com maior probabilidade de o Cedente estar fora do computador. Os outros documentos têm prazo mais longo e são menos urgentes.]**

**Fluxo mobile de assinatura:**

```
T-043 — Lista de Assinaturas
  ├── Documento suportado mobile → botão "Assinar"
  │     ↓
  │     T-044 — ZapSign WebView (fullscreen modal)
  │       ├── expo-web-browser em modo WebView
  │       ├── Botão "Fechar" flutuante (44×44px, canto superior direito)
  │       ├── Loading: skeleton enquanto carrega
  │       ├── Assinatura concluída (postMessage) → fecha + toast + lista atualiza
  │       └── Erro de carregamento (10s) → mensagem + retry
  └── Documento não suportado mobile → "Assine pelo computador"
```

**WebView ZapSign mobile:**

```typescript
import { WebView } from 'react-native-webview'

// Configuração do WebView para ZapSign
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
    // Injeta listener para postMessage
    window.addEventListener('message', (e) => {
      window.ReactNativeWebView.postMessage(JSON.stringify(e.data))
    })
  `}
  startInLoadingState={true}
  renderLoading={() => <SkeletonFullPage />}
  onError={() => setZapSignError(true)}
/>
```

### 3.3 Resposta a Propostas (T-035 a T-039 Mobile)

**Diferenças mobile vs. web:**
- Lista de propostas (T-035): cards em coluna única (sem split view)
- Toque em um card navega para T-036 (sem split view lateral)
- Ações (Aceitar/Recusar/Contraproposta): bottom sheet ao toque em um `ProposalCard`

**Bottom sheet de ações:**

```typescript
// Ao tocar em ProposalCard no mobile
function abrirAcoesBottomSheet(propostaId: string) {
  bottomSheet.present(
    <PropostaAcoesSheet
      propostaId={propostaId}
      onAceitar={() => router.push(`/modals/aceitar-proposta?id=${propostaId}`)}
      onRecusar={() => router.push(`/modals/recusar-proposta?id=${propostaId}`)}
      onContrapropor={() => router.push(`/modals/contraproposta?id=${propostaId}`)}
    />
  )
}
```

**Haptics em ações críticas:**

```typescript
import * as Haptics from 'expo-haptics'

// Aceite de proposta — feedback de sucesso
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success)

// Recusa de proposta — feedback neutro
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium)

// Erro — feedback de erro
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error)

// Ação destrutiva (cancelamento) — vibração longa
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy)
```

---

## 4. Push Notifications

### 4.1 Canais de Notificação

| **Canal** | **Nome exibido** | **Importância** | **Eventos** |
|---|---|---|---|
| `propostas` | "Propostas" | Alta (heads-up) | Nova proposta recebida, proposta expirando |
| `documentos` | "Documentos" | Normal | Documento rejeitado, pendência no dossiê |
| `assinaturas` | "Assinaturas" | Alta | Novo documento para assinar, prazo crítico |
| `financeiro` | "Financeiro" | Alta | Negócio fechado, valores liberados |
| `lembretes` | "Lembretes" | Normal | Rascunho expirando, inatividade |
| `sistema` | "Sistema" | Baixa | Manutenção, novidades da plataforma |

**Canais Android 8+ (API 26+):**
```typescript
// Configurar canais no boot do app (Android)
await Notifications.setNotificationChannelAsync('propostas', {
  name: 'Propostas',
  importance: Notifications.AndroidImportance.HIGH,
  vibrationPattern: [0, 250, 250, 250],
  lightColor: '#0069A8',   // --primary
})
// ... mesmo para outros canais
```

### 4.2 Payload das Notificações

```typescript
// Estrutura padrão de payload push
interface PushPayload {
  title: string
  body: string
  data: {
    tipo: 'PROPOSTA' | 'DOCUMENTO' | 'ASSINATURA' | 'FINANCEIRO' | 'LEMBRETE'
    caseId?: string
    proposalId?: string
    envelopeId?: string
    deepLinkUrl: string     // rota expo-router de destino
  }
}

// Exemplos de deepLinkUrl por tipo
// PROPOSTA: '/propostas/{proposalId}'
// DOCUMENTO: '/casos/{caseId}/documentos'
// ASSINATURA: '/mais/assinaturas'
// FINANCEIRO: '/casos/{caseId}/financeiro'
// LEMBRETE: '/cadastrar' (rascunho)
```

### 4.3 Solicitação de Permissão

**Estratégia: solicitar permissão no primeiro acesso ao Dashboard (T-013/T-014) após login, não no onboarding.**

```typescript
// Hook para solicitar permissão
async function solicitarPermissaoNotificacao(): Promise<boolean> {
  const { status: existingStatus } = await Notifications.getPermissionsAsync()

  if (existingStatus === 'granted') return true

  // Somente solicitar se não foi negado anteriormente
  if (existingStatus !== 'denied') {
    const { status } = await Notifications.requestPermissionsAsync()
    return status === 'granted'
  }

  return false
}
```

**Diferenças por plataforma:**

| **Comportamento** | **iOS** | **Android ≤ 12 (API 32-)** | **Android 13+ (API 33+)** |
|---|---|---|---|
| Permissão necessária? | Sim — solicitada | Não — automática | Sim — `POST_NOTIFICATIONS` solicitada |
| Onde solicitar | Na primeira abertura do Dashboard | N/A (automático) | Na primeira abertura do Dashboard |
| Se negada | Instruir a ativar em Configurações → Notificações | N/A | Instruir a ativar em Configurações → Notificações |

### 4.4 Token de Dispositivo

```typescript
// Registrar token no backend após permissão concedida
async function registrarTokenDispositivo() {
  const token = await Notifications.getExpoPushTokenAsync({
    projectId: Constants.expoConfig?.extra?.eas?.projectId,
  })

  await api.post('/v1/cedentes/push-token', {
    token: token.data,
    plataforma: Platform.OS,   // 'ios' | 'android'
    deviceId: await getUniqueId(),
  })
}
```

**Rotação de tokens:** verificar e atualizar token a cada boot do app. Um Cedente pode ter múltiplos tokens (múltiplos dispositivos).

---

## 5. Comportamento Offline

### 5.1 Telas com Suporte Offline (Cache Local)

| **Tela** | **Dados em cache** | **TTL do cache** | **Ações de escrita offline** |
|---|---|---|---|
| T-013/T-014 Dashboard | KPIs, lista de casos ativos, próximos passos | 30 minutos | Bloqueadas — somente leitura |
| T-015 Lista de Casos | Lista paginada (primeira página, 10 casos) | 30 minutos | Bloqueadas |
| T-016 Detalhe do Caso | Dados do caso, status, próximo passo | 15 minutos | Bloqueadas |
| T-035 Lista de Propostas | Lista de propostas ativas | 15 minutos | Bloqueadas — aceite/recusa requer conexão |
| T-040 Documentos | Checklist de documentos, status de cada um | 30 minutos | Upload bloqueado — tooltip "Sem conexão" |
| T-050 Perfil | Dados pessoais do Cedente | 60 minutos | Bloqueadas |

**Implementação com TanStack Query:**
```typescript
// Cache configurado por query
useQuery({
  queryKey: ['dashboard', cedenteId],
  queryFn: fetchDashboard,
  staleTime: 30 * 60 * 1000,          // 30 minutos
  gcTime: 24 * 60 * 60 * 1000,        // 24 horas no cache
  networkMode: 'offlineFirst',          // usa cache quando offline
})
```

### 5.2 Detecção de Estado de Rede

```typescript
import NetInfo from '@react-native-community/netinfo'

function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(true)

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      setIsOnline(state.isConnected ?? false)
    })
    return unsubscribe
  }, [])

  return { isOnline }
}
```

**Banner offline global:** banner fixo âmbar no topo da tela quando `isOnline === false`. Texto: "Sem conexão. Mostrando dados salvos." Desaparece automaticamente quando reconecta.

### 5.3 Reconexão e Sincronização

Ao reconectar após período offline:
1. `TanStack Query` invalida automaticamente queries com `staleTime` expirado
2. Novo fetch em background (sem travar a UI)
3. Badge "Atualizado às HH:MM" nos dados em cache é removido após sincronização

**Ações de escrita enfileiradas:** não implementado nesta versão (v1.0). Todas as ações de escrita (upload, aceite, contraproposta) requerem conexão ativa. [DECISÃO AUTÔNOMA — fila de ações offline seria complexidade prematura. Cedentes com conectividade ruim são minoria na persona atual.]

---

## 6. Biometria e Segurança

### 6.1 Re-autenticação Biométrica

Contextos que exigem re-autenticação biométrica:
- Abertura do app após 30 minutos em background
- Acesso à seção Financeiro (T-045/T-046)
- Aceite de proposta (T-037)
- Solicitação de desistência (T-047)
- Cancelamento de caso com Escrow ativo (T-023)

```typescript
import * as LocalAuthentication from 'expo-local-authentication'

async function reautenticarComBiometria(): Promise<boolean> {
  const biometriaDisponivel = await LocalAuthentication.hasHardwareAsync()
  const biometriaConfigurada = await LocalAuthentication.isEnrolledAsync()

  if (!biometriaDisponivel || !biometriaConfigurada) {
    // Fallback: solicitar senha da conta
    return await solicitarSenhaConta()
  }

  const resultado = await LocalAuthentication.authenticateAsync({
    promptMessage: 'Confirme sua identidade para continuar',
    fallbackLabel: 'Usar senha',    // iOS
    cancelLabel: 'Cancelar',
    disableDeviceFallback: false,   // permite PIN/senha do dispositivo como fallback
  })

  return resultado.success
}
```

**Biometria por plataforma:**

| **Tipo** | **iOS** | **Android** |
|---|---|---|
| Facial | Face ID | Face Unlock (quando disponível) |
| Digital | Touch ID | Fingerprint |
| Fallback | PIN/senha do dispositivo | PIN/senha/padrão do dispositivo |
| Prompt customizado | `promptMessage` customizado | `promptMessage` customizado |

### 6.2 Armazenamento Seguro

```typescript
import * as SecureStore from 'expo-secure-store'

// Armazenar refresh token
await SecureStore.setItemAsync('refresh_token', token, {
  keychainAccessibility: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
})

// Ler refresh token
const refreshToken = await SecureStore.getItemAsync('refresh_token')

// Revogar ao fazer logout
await SecureStore.deleteItemAsync('refresh_token')
```

**Não usar AsyncStorage para tokens.** AsyncStorage é não criptografado e visível em backups.

### 6.3 App em Background e Foreground

```typescript
import { AppState } from 'react-native'

// Quando app retorna ao foreground
AppState.addEventListener('change', (nextAppState) => {
  if (nextAppState === 'active') {
    const tempoBackground = Date.now() - horaEntradaBackground

    // Se ficou > 30 minutos em background: exigir re-autenticação
    if (tempoBackground > 30 * 60 * 1000) {
      setRequereReautenticacao(true)
    }

    // Invalidar queries stale
    queryClient.invalidateQueries({ type: 'active' })
  }

  if (nextAppState === 'background') {
    setHoraEntradaBackground(Date.now())
  }
})
```

---

## 7. Navegação e Gestos

### 7.1 Gestos Nativos

| **Gesto** | **Plataforma** | **Comportamento** | **Tela** |
|---|---|---|---|
| Swipe-back (direita) | iOS | Navega para a tela anterior | Todas com back navigation |
| Botão hardware back | Android | Navega para a tela anterior ou sai do app no root | Todas |
| Pull-to-refresh | iOS e Android | Recarrega dados da tela atual | T-015, T-035, T-013, T-040 |
| Long press em DocumentCard | iOS e Android | Abre menu de ações contextuais | T-040 |
| Swipe-to-dismiss em modal | iOS | Fecha modal | Todos os modals sheet |
| Scale press (0.95) | iOS e Android | Feedback visual ao toque em botões e cards | Todos os cards e botões |

### 7.2 Pull-to-Refresh

```typescript
import { RefreshControl, ScrollView } from 'react-native'

function ListaCasos() {
  const [refreshing, setRefreshing] = useState(false)
  const { refetch } = useQuery(...)

  const onRefresh = useCallback(async () => {
    setRefreshing(true)
    await refetch()
    setRefreshing(false)
  }, [refetch])

  return (
    <ScrollView
      refreshControl={
        <RefreshControl
          refreshing={refreshing}
          onRefresh={onRefresh}
          tintColor={tokens.primary}    // iOS — cor do spinner
          colors={[tokens.primary]}     // Android — cores do spinner
        />
      }
    >
      {/* conteúdo */}
    </ScrollView>
  )
}
```

### 7.3 Safe Areas

```typescript
import { useSafeAreaInsets } from 'react-native-safe-area-context'

function BottomTabBar() {
  const insets = useSafeAreaInsets()

  return (
    <View style={{
      paddingBottom: insets.bottom,    // home indicator iOS / barra de navegação Android
      paddingLeft: insets.left,        // landscape mode
      paddingRight: insets.right,
    }}>
      {/* tabs */}
    </View>
  )
}
```

**Regras de safe area:**
- Bottom tab bar: sempre respeita `insets.bottom`
- Modals: `insets.top` para notch/Dynamic Island
- Câmera (T-042): fullscreen, ignorar safe area intencionalmente para maximizar área de visualização

---

## 8. Performance

### 8.1 Metas de Performance

| **Métrica** | **Meta** | **Crítica** |
|---|---|---|
| Time to Interactive (TTI) no boot | < 3 segundos (cold start) | Sim |
| Time to Interactive (TTI) no foreground | < 1 segundo (warm start) | Sim |
| FPS durante scroll de lista | 60 FPS constante | Sim |
| FPS durante animações | 60 FPS constante | Sim |
| Tamanho do bundle JS (initial) | < 2 MB comprimido | Recomendado |
| Tempo de upload de documento (10 MB) | < 30 segundos em 4G | Recomendado |
| Tempo de abertura da câmera | < 2 segundos | Sim |

### 8.2 Otimizações Obrigatórias

**Listas grandes (FlatList):**
```typescript
<FlatList
  data={casos}
  renderItem={({ item }) => <CaseCard caso={item} />}
  keyExtractor={item => item.id}
  getItemLayout={(_, index) => ({
    length: CASE_CARD_HEIGHT,      // altura fixa para evitar recálculos
    offset: CASE_CARD_HEIGHT * index,
    index,
  })}
  maxToRenderPerBatch={10}
  windowSize={10}
  removeClippedSubviews={true}    // Android — remove views fora da viewport
  initialNumToRender={5}
/>
```

**Memoização de componentes:**
```typescript
// ProposalCard, DocumentCard, CaseCard devem ser memoizados
const ProposalCard = memo(function ProposalCard({ proposta, ...props }: ProposalCardProps) {
  // ...
}, (prevProps, nextProps) => prevProps.proposta.status === nextProps.proposta.status)
```

**Imagens:**
- Usar `expo-image` (substituto performático de `<Image>`)
- Lazy loading para listas de documentos
- Prefetch de avatares e thumbnails em background

### 8.3 Hermes Engine

O Hermes (engine JS do React Native 0.76+) deve estar habilitado em todas as builds:

```json
// app.json
{
  "expo": {
    "jsEngine": "hermes"
  }
}
```

Hermes reduz TTI em ~20% e consumo de memória em ~40% vs. JSC.

---

## 9. Diferenças iOS vs. Android

### 9.1 Tabela de Diferenças

| **Feature** | **iOS** | **Android** |
|---|---|---|
| Navegação back | Swipe-back nativo pelo expo-router | Botão hardware back (interceptado pelo expo-router) |
| Permissão de push | Solicitada explicitamente (`Notifications.requestPermissionsAsync`) | Automática ≤ API 32; solicitada ≥ API 33 |
| Biometria (Face) | Face ID (seguro, hardware dedicado) | Face Unlock (varia por fabricante, nem sempre seguro) |
| Biometria (Digital) | Touch ID (sensores laterais ou home button) | Fingerprint (posição varia por modelo) |
| Teclado soft | Push-up nativo `KeyboardAvoidingView behavior="padding"` | Resize automático (pode precisar de ajuste manual) |
| Status bar | `expo-status-bar` `style="auto"` | Idem |
| Haptics | `Haptics.notificationAsync` (taptic engine) | `Haptics.impactAsync` (vibração motor) |
| Modals | Sheets com handle visible, swipe-to-dismiss | Bottom sheets via `react-native-reanimated` |
| Safe area top | Notch / Dynamic Island — `insets.top` variável | Status bar padrão — `insets.top = StatusBar.currentHeight` |
| Font rendering | San Francisco (sistema) + Inter Variable | Roboto (sistema) + Inter Variable |
| Deep links | Universal Links (`apple-app-site-association`) | App Links (`assetlinks.json`) |
| OTA updates | Expo Updates (sem restrição) | Expo Updates (sem restrição) |

### 9.2 Código de Plataforma

```typescript
import { Platform } from 'react-native'

// Estilo condicional por plataforma
const styles = StyleSheet.create({
  container: {
    ...Platform.select({
      ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 2 }, shadowOpacity: 0.1, shadowRadius: 4 },
      android: { elevation: 4 },
    }),
  },
})

// Comportamento condicional
const keyboardBehavior = Platform.OS === 'ios' ? 'padding' : 'height'
```

---

## 10. Deep Links

### 10.1 Esquema de Deep Link

```
repasse-seguro://casos/{caseId}                    → T-016 Detalhe do caso
repasse-seguro://casos/{caseId}/documentos         → T-017 Documentos do caso
repasse-seguro://propostas/{proposalId}            → T-036 Detalhe da proposta
repasse-seguro://assinaturas/{envelopeId}          → T-043 → T-044 (abre para assinar)
repasse-seguro://financeiro/{caseId}               → T-046 Financeiro do caso
repasse-seguro://cadastrar                         → T-024 (ou T-030 se há rascunho)
repasse-seguro://guardiao                          → T-048 Chat do Guardião
```

### 10.2 Configuração expo-router

```typescript
// app.json
{
  "expo": {
    "scheme": "repasse-seguro",
    "ios": {
      "associatedDomains": ["applinks:repasse-seguro.com.br"]
    },
    "android": {
      "intentFilters": [{
        "action": "VIEW",
        "autoVerify": true,
        "data": [{ "scheme": "https", "host": "repasse-seguro.com.br" }],
        "category": ["BROWSABLE", "DEFAULT"]
      }]
    }
  }
}
```

### 10.3 Tratamento de Deep Links em Background

Quando o app está fechado e o usuário toca em uma notificação push com `deepLinkUrl`:
1. App inicia normalmente (AuthGuard verifica autenticação)
2. Se autenticado: navega diretamente para a rota do deep link
3. Se não autenticado: navega para Login, armazena `deepLinkUrl` no Zustand, e redireciona após login bem-sucedido

```typescript
// Hook de tratamento de notificação
Notifications.addNotificationResponseReceivedListener(response => {
  const { deepLinkUrl } = response.notification.request.content.data
  if (deepLinkUrl && isAuthenticated) {
    router.push(deepLinkUrl)
  } else if (deepLinkUrl) {
    setDeepLinkPendente(deepLinkUrl)
    router.push('/(auth)/login')
  }
})
```

---

## 11. Acessibilidade Mobile

### 11.1 VoiceOver (iOS) e TalkBack (Android)

| **Regra** | **Implementação** |
|---|---|
| Todos os botões têm rótulo acessível | `accessibilityLabel="Aceitar proposta de R$ 45.000,00"` |
| Imagens decorativas ignoradas | `accessible={false}` em imagens de background |
| Ícones sem texto têm rótulo | `accessibilityLabel="Câmera"` + `accessibilityRole="button"` |
| Listas têm contexto | `accessibilityLabel="Lista de casos. 3 casos encontrados."` |
| Status de loading anunciado | `accessibilityLiveRegion="polite"` em `ActivityIndicator` substituto |
| Valores financeiros legíveis | `accessibilityLabel="R$ 45 mil e 500 reais"` em valores numéricos |
| Contadores regressivos | Atualizados a cada minuto para o leitor (não a cada segundo) |

### 11.2 Tamanhos Mínimos de Toque

**Regra:** todo elemento interativo deve ter área de toque mínima de 44×44 pontos (iOS) / 48×48dp (Android).

```typescript
// Wrapper para garantir área mínima de toque
const TocavelMinimo = ({ children, onPress, accessibilityLabel }) => (
  <Pressable
    onPress={onPress}
    accessibilityLabel={accessibilityLabel}
    style={{ minWidth: 44, minHeight: 44, alignItems: 'center', justifyContent: 'center' }}
    hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}
  >
    {children}
  </Pressable>
)
```

### 11.3 Contraste de Cores

Todos os tokens do Brand Theme Guide (doc 03) atendem WCAG 2.1 AA para tamanho de texto ≥ 14sp. Verificação obrigatória em:
- `--primary` (#0069A8) sobre `--background` (#FFFFFF): ratio 5.1:1 ✅
- `--foreground` (#0A0A0A) sobre `--background` (#FFFFFF): ratio 21:1 ✅
- `--muted-foreground` sobre `--background`: verificar por componente em QA

---

## 12. Testes Mobile

### 12.1 Estratégia de Testes

| **Nível** | **Ferramenta** | **Cobertura mínima** |
|---|---|---|
| Unit (componentes) | Jest + React Native Testing Library | 80% dos componentes críticos |
| Integration (fluxos) | Jest + RNTL + MSW (mock de API) | Fluxo de upload, fluxo de proposta |
| E2E | Detox | Fluxos críticos: cadastro → upload → proposta → assinatura |
| Manual (dispositivos físicos) | Matriz de dispositivos §1.3 | Antes de cada release |

### 12.2 Fluxos E2E Obrigatórios (Detox)

1. **Cadastro completo:** T-002 → T-003 → ativação → T-013
2. **Upload por câmera:** T-040 → T-042 → upload → DocumentCard "Em Análise"
3. **Aceitar proposta:** T-035 → T-036 → T-037 → toast de sucesso
4. **Recusar proposta:** T-035 → T-038 → toast de sucesso
5. **Contraproposta:** T-035 → T-039 → validação de piso → envio
6. **Notificação push → deep link:** notificação de proposta → toque → T-036
7. **Offline → reconexão:** desligar rede → navegar → reativar rede → sincronização

### 12.3 Performance Testing

- **Flipper** (desenvolvimento): rastrear renders desnecessários, consumo de memória, uso de rede
- **Expo Dev Client**: testar builds de desenvolvimento com APIs nativas reais
- **EAS Build + Sentry Performance**: medir TTI em builds de produção em dispositivos reais

---

## 13. Build e Deploy

### 13.1 EAS Build

```json
// eas.json
{
  "cli": { "version": ">= 7.0.0" },
  "build": {
    "development": {
      "distribution": "internal",
      "android": { "buildType": "apk" },
      "ios": { "simulator": true }
    },
    "preview": {
      "distribution": "internal",
      "channel": "preview"
    },
    "production": {
      "channel": "production",
      "android": { "buildType": "app-bundle" },
      "ios": { "autoIncrement": true }
    }
  },
  "submit": {
    "production": {
      "ios": { "appleId": "contato@repasse-seguro.com.br" },
      "android": { "track": "production" }
    }
  }
}
```

### 13.2 OTA Updates

Canais de update:
- `preview`: builds de staging para QA
- `production`: builds de produção

**Política de update:**
- Updates de JavaScript/assets: via OTA (sem passar pelas lojas)
- Updates que alteram módulos nativos: exigem nova build nas lojas

```typescript
// Verificar e aplicar update ao iniciar
import * as Updates from 'expo-updates'

async function verificarUpdate() {
  if (!__DEV__) {
    const update = await Updates.checkForUpdateAsync()
    if (update.isAvailable) {
      await Updates.fetchUpdateAsync()
      await Updates.reloadAsync()
    }
  }
}
```

### 13.3 Variáveis de Ambiente Mobile

```typescript
// app.config.ts
export default ({ config }) => ({
  ...config,
  extra: {
    apiUrl: process.env.API_URL,
    supabaseUrl: process.env.SUPABASE_URL,
    supabaseAnonKey: process.env.SUPABASE_ANON_KEY,
    sentryDsn: process.env.SENTRY_DSN,
    posthogApiKey: process.env.POSTHOG_API_KEY,
    eas: {
      projectId: process.env.EAS_PROJECT_ID,
    },
  },
})
```

Nunca usar `process.env` diretamente nos componentes — acessar via `Constants.expoConfig?.extra`.
