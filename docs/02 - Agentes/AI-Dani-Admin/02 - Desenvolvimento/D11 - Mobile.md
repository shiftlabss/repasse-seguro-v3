# Mobile — AI-Dani-Admin

## Guia de Adaptação Mobile do Módulo de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Engenharia Frontend Mobile |
| Escopo | Adaptação do módulo AI-Dani-Admin para interfaces mobile (React Native + Expo) |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D02 (Stacks — React Native), D06 (Telas), D09 (Contratos de UI) |

---

> **📌 TL;DR**
>
> - **Contexto:** O painel Admin (AI-Dani-Admin) é primariamente uma interface web (dashboard interno). Mobile é suporte secundário para alertas e supervisão em campo.
> - **Escopo mobile:** Alertas push, visualização da lista de interações e takeover básico. Dashboard completo de métricas é web-only na v1.0.
> - **Stack:** React Native + Expo SDK 52+ + expo-router 4+. Mesma API REST do web.
> - **Funcionalidades mobile v1.0:** Receber alertas push, ver lista de interações, ver detalhe de interação, iniciar/encerrar takeover. Dashboard de métricas e checklist de lançamento: web-only.
> - **Webchat (para usuários finais):** Totalmente suportado no mobile via React Native — recebe mensagens de takeover e retorno ao agente normalmente.

---

## 1. Escopo Mobile — AI-Dani-Admin

### 1.1 Funcionalidades no Mobile (Admin)

| Funcionalidade | Suporte Mobile | Observação |
|---|---|---|
| Receber alertas push (desligamento automático, taxa de erro) | ✅ v1.0 | Via Expo Push Notifications |
| Ver lista de interações sinalizadas | ✅ v1.0 | Filtro padrão: sinalizadas para revisão |
| Ver detalhe de interação | ✅ v1.0 | |
| Iniciar takeover | ✅ v1.0 | |
| Enviar mensagens durante takeover | ✅ v1.0 | |
| Encerrar takeover | ✅ v1.0 | |
| Dashboard de métricas (completo) | ❌ Web-only v1.0 | Dados sensíveis, tela pequena inadequada |
| Configurar threshold | ❌ Web-only v1.0 | Ação de configuração — requer confirmação cuidadosa |
| Checklist de prontidão para lançamento | ❌ Web-only v1.0 | Ação crítica de lançamento |
| Log de auditoria | 🔶 v1.1 | Planejado para versão futura |

### 1.2 Funcionalidades do Webchat (Usuário Final) no Mobile

| Funcionalidade | Suporte Mobile | Observação |
|---|---|---|
| Abrir chat com agente | ✅ Completo | Via FAB global na interface mobile |
| Receber mensagem de takeover do Admin | ✅ Completo | Separador COMP-002 exibido |
| Continuar conversa com Admin durante takeover | ✅ Completo | |
| Receber mensagem de retorno ao agente | ✅ Completo | |
| Receber mensagem de indisponibilidade da API | ✅ Completo | Badge degradado no FAB |

---

## 2. Stack Mobile

Conforme D02 - Stacks (seção 4 — Mobile):

| Tecnologia | Versão | Uso |
|---|---|---|
| React Native | 0.76+ | Framework base |
| Expo SDK | 52+ | Plataforma managed workflow |
| expo-router | 4.x+ | Navegação file-based |
| React Native Reanimated | 3.x+ | Animações na UI thread |
| React Native Gesture Handler | 2.x+ | Gestos |
| TanStack Query | 5.x+ | Fetching e cache de dados |
| Zustand | 5.x+ | Estado global |
| expo-notifications | latest | Push notifications para alertas Admin |
| expo-secure-store | latest | Armazenamento seguro do JWT Admin |
| AsyncStorage | latest | Persistência de preferências não-sensíveis |

---

## 3. Navegação Mobile (Admin)

```
app/
├── (auth)/
│   └── login.tsx              # tela de login (herdada da plataforma)
└── (admin)/
    ├── _layout.tsx             # layout com tabs Admin
    ├── supervisao/
    │   ├── index.tsx           # T-001 mobile: lista de interações sinalizadas
    │   ├── [id].tsx            # T-002 mobile: detalhe de interação
    │   └── [id]/takeover.tsx   # T-003 mobile: chat em takeover
    └── alertas/
        └── index.tsx           # lista de alertas ativos
```

**Tabs de navegação mobile:**
```
[🔔 Alertas]  [💬 Interações]  [👤 Perfil]
```

---

## 4. Adaptações de Layout para Mobile

### 4.1 T-001 Mobile — Lista de Interações

**Diferenças da versão web:**
- Filtro padrão: "Sinalizadas para revisão" (não todas as interações). Admin mobile está em alerta, não em monitoramento passivo.
- Tabela substituída por `FlatList` com cards de interação.
- Cada card exibe: usuário anonimizado, agente, confiança, status badge, timestamp.
- Swipe to action: deslizar item para direita revela botão "Assumir conversa" (apenas para interações não encerradas).

```typescript
// Card de interação mobile
interface InteractionCardProps {
  interaction: InteractionRow
  onTakeover: () => void
}

// Layout do card:
// ┌─────────────────────────────────┐
// │ [Badge status]     [timestamp]  │
// │ ID-****45 · Dani-Cessionário    │
// │ Confiança: 61% ⚠               │
// │ [Assumir conversa →]            │ ← apenas se sinalizada
// └─────────────────────────────────┘
```

**FlatList config:**
```typescript
<FlatList
  data={interactions}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <InteractionCard interaction={item} />}
  getItemLayout={(data, index) => ({ length: 96, offset: 96 * index, index })}
  onEndReached={loadMoreInteractions}
  onEndReachedThreshold={0.5}
  ListEmptyComponent={<EmptyState />}
/>
```

### 4.2 T-002 Mobile — Detalhe de Interação

**Diferenças da versão web:**
- Layout de coluna única (não grid 3-colunas).
- Métricas (confiança, latência, agente, data) em cards horizontais com scroll.
- Histórico do chat em `ScrollView` com `ref` para auto-scroll ao bottom.
- Botão "Assumir conversa" fixo no bottom (sticky footer).

```typescript
// Sticky bottom action
<View style={styles.stickyFooter}>
  <TouchableOpacity
    style={[styles.button, status === 'ENCERRADA' && styles.buttonDisabled]}
    disabled={status === 'ENCERRADA'}
    onPress={handleTakeover}
  >
    <Text>Assumir conversa</Text>
  </TouchableOpacity>
</View>
```

### 4.3 T-003 Mobile — Chat em Takeover

**Componentes:**
- `KeyboardAvoidingView` para comportamento correto do teclado virtual.
- `FlatList` invertida para o histórico do chat (scroll natural do fundo).
- Input fixo no fundo com `Platform.OS` para ajuste de comportamento por plataforma.

```typescript
<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
  style={{ flex: 1 }}
>
  <TakeoverHeader />
  <FlatList
    data={messages}
    inverted
    keyExtractor={(m) => m.id}
    renderItem={({ item }) => <ChatMessage message={item} />}
  />
  <MessageInput onSend={handleSend} />
</KeyboardAvoidingView>
```

---

## 5. Push Notifications — Alertas Admin

### 5.1 Configuração

```typescript
// app/(admin)/_layout.tsx

import * as Notifications from 'expo-notifications'
import { registerForPushNotificationsAsync } from '@/lib/notifications'

// Configurar handler de notificações
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,  // alertas críticos devem fazer som
    shouldSetBadge: true,
  }),
})
```

### 5.2 Tipos de Notificação Push

| Tipo | Título | Corpo | Ação ao tap |
|---|---|---|---|
| Desligamento automático | "Agente desligado" | "Dani-[X] desligado. Taxa de erro: [X]% (15min)" | Navega para lista de alertas |
| Latência alta | "Latência alta" | "Tempo de resposta acima do SLA há [X] min" | Navega para lista de interações |
| Taxa de erro elevada | "Taxa de erro elevada" | "[X]% das respostas com erro (15min)" | Navega para lista de interações |
| CSAT degradado | "CSAT abaixo da meta" | "Média: [X]/5 nas últimas 24h" | Navega para lista de interações |
| Nova interação sinalizada | "Interação aguardando" | "Confiança: [X]% — requer revisão" | Navega para detalhe da interação |

### 5.3 Registro do Token

```typescript
// lib/notifications.ts

export async function registerForPushNotificationsAsync(): Promise<string | null> {
  const { status: existingStatus } = await Notifications.getPermissionsAsync()
  let finalStatus = existingStatus

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync()
    finalStatus = status
  }

  if (finalStatus !== 'granted') return null

  const token = (await Notifications.getExpoPushTokenAsync()).data
  // Registrar token na API: POST /api/v1/admin/push-tokens
  return token
}
```

---

## 6. Design Tokens Mobile

Os tokens de cores são os mesmos do D03 - Brand Guide, implementados como constantes TypeScript:

```typescript
// src/constants/tokens.ts

export const colors = {
  background: '#FFFFFF',
  foreground: '#0A0A0A',
  primary: '#0069A8',
  primaryForeground: '#F0F9FF',
  destructive: '#E7000B',
  muted: '#F5F5F5',
  mutedForeground: '#737373',
  border: '#E5E5E5',
  // Status operacional
  success: '#16A34A',
  successSubtle: '#F0FDF4',
  warning: '#CA8A04',
  warningSubtle: '#FFFBEB',
  takeover: '#0069A8',
  takeoverSubtle: '#EFF6FF',
}

export const spacing = {
  0: 0, 1: 4, 2: 8, 3: 12, 4: 16,
  5: 20, 6: 24, 8: 32, 10: 40, 12: 48,
}

export const radius = {
  sm: 8.4, md: 11.2, lg: 14, xl: 19.6,
}

export const typography = {
  fontFamily: 'Inter_400Regular',
  fontFamilyMedium: 'Inter_500Medium',
  fontFamilySemibold: 'Inter_600SemiBold',
  fontFamilyBold: 'Inter_700Bold',
}
```

---

## 7. Animações Mobile

Conforme D04 - Motion Spec, adaptado para React Native Reanimated:

```typescript
// Equivalentes mobile para os tokens de duração do D04
export const duration = {
  instant: 100,
  fast: 150,
  normal: 250,
  slow: 400,
  page: 500,
}

// Skeleton shimmer no mobile
import Animated, { useSharedValue, withRepeat, withTiming } from 'react-native-reanimated'

function SkeletonLoader({ width, height }: { width: number; height: number }) {
  const opacity = useSharedValue(0.4)

  useEffect(() => {
    opacity.value = withRepeat(
      withTiming(1, { duration: 700 }),
      -1,
      true
    )
  }, [])

  return (
    <Animated.View
      style={[
        { width, height, borderRadius: radius.sm, backgroundColor: colors.muted },
        { opacity }
      ]}
    />
  )
}
```

**prefers-reduced-motion no mobile:** Usar `AccessibilityInfo.isReduceMotionEnabled()` para desabilitar animações quando o usuário ativou "Reduzir movimento" nas acessibilidades do dispositivo.

---

## 8. Segurança Mobile

| Requisito | Implementação |
|---|---|
| Token JWT Admin | `expo-secure-store` — não usar AsyncStorage para tokens |
| Expiração de sessão | Verificar validade do JWT a cada abertura do app e a cada request |
| Screenshot protection | `expo-screen-capture` — considerar para telas com dados operacionais sensíveis |
| Autenticação biométrica (opcional) | `expo-local-authentication` para desbloqueio rápido do app Admin |
| Deep link de notificação | Validar que o deep link de notificação aponta para uma interação que pertence ao Admin antes de navegar |

---

## 9. Webchat no Mobile (Usuário Final)

O webchat para usuários finais (Cessionário/Cedente) no mobile segue o padrão da plataforma Repasse Seguro. Pontos específicos do AI-Dani-Admin:

### 9.1 FAB Global com Badge de Status

```typescript
// Componente FAB no layout principal da plataforma mobile
interface ChatFABProps {
  agentStatus: 'ATIVO' | 'FALLBACK_ATIVO' | 'DESLIGADO'
}

// agentStatus === 'FALLBACK_ATIVO':
//   Badge amarelo sobre o FAB
//   Tooltip/accessibility: "Agente temporariamente indisponível"
```

### 9.2 Separador de Takeover no Chat Mobile

O COMP-002 (separador visual de takeover) é exibido na interface de chat do usuário mobile:

```typescript
// Componente de separador no chat mobile
function TakeoverSeparator({ type }: { type: 'START' | 'END' }) {
  return (
    <View style={styles.separatorContainer}>
      <View style={styles.separatorLine} />
      <Text style={styles.separatorText}>
        {type === 'START' ? 'Atendimento humano' : 'Retorno ao atendimento automatizado'}
      </Text>
      <View style={styles.separatorLine} />
    </View>
  )
}
```

---

## 10. Backlog Mobile v1.1+

| Item | Justificativa |
|---|---|
| Dashboard de métricas mobile (versão simplificada) | Cards de métricas principais para supervisão em campo |
| Log de auditoria mobile | Consulta rápida de ações recentes |
| Configuração de threshold mobile | Com confirmação extra por ser ação de configuração crítica |
| Dark mode automático | `useColorScheme()` para adaptar tokens ao esquema do sistema |

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. Escopo mobile: alertas push + lista de interações + takeover. Web-only: dashboard e checklist de lançamento. |
