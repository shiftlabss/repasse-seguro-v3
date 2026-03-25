# 11 - Mobile

## Módulo Cessionário · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Produto, Mobile e Engenharia |
| **Escopo** | Especificação mobile — navegação, gestos, offline, push notifications e diferenças de plataforma |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop — Pipeline ShiftLabs v9.5 |
| **Data da versão** | 22/03/2026 00:00 (America/Fortaleza) |
| **Status** | Ativo |
| **Referências** | 01 - Regras de Negócio · 02 - Stacks · 05 - PRD · 06 - Mapa de Telas · 07 - Wireframes · 09 - Contratos de UI por Tela |

---

> 📌 **TL;DR**
>
> - **Plataformas:** iOS 16+ e Android 10 (API 29+). React Native 0.76+ com Expo SDK 52+, expo-router 4.
> - **Navegação:** Stack navigator raiz → Tab navigator (5 tabs) → Stack modals para fluxos de proposta, KYC e formalização. expo-router com file-based routing.
> - **Telas com suporte offline:** 8 telas críticas com dados em cache local (Marketplace, Dashboard, Propostas, Negociações, Histórico Financeiro, Perfil, Notificações, Detalhe de Oportunidade). Ações de escrita são enfileiradas.
> - **Features nativas:** câmera (selfie KYC), galeria/documentos (upload KYC), push notifications (Expo Notifications), biometria (re-autenticação), haptics (feedback em ações críticas), network detection.
> - **Diferenças críticas iOS vs. Android:** swipe-back nativo (iOS) vs. botão hardware (Android); permissão push automática Android 12- vs. solicitada Android 13+; Face ID/Touch ID (iOS) vs. Fingerprint/Face Unlock (Android); bottom tab com safe-area (iOS) vs. navigation bar Android.
> - **Zero seções pendentes** — todos os comportamentos especificados com base nos docs de input.

---

## 1. Plataformas e Versões Suportadas

### 1.1 Versões Mínimas de Sistema

| Plataforma | Versão mínima | API Level | Justificativa |
|---|---|---|---|
| iOS | 16.0 | — | Suporte a Face ID v2, Dynamic Island handling, live activities; base de usuários iOS 16+ representa >90% no Brasil |
| Android | 10 (Q) | API 29 | Suporte a Scoped Storage obrigatório, biometria unificada `BiometricPrompt`, permissões modernizadas. API 28 descartada: deprecated APIs, complexidade extra sem benefício |

### 1.2 Stack Mobile

| Tecnologia | Versão | Papel |
|---|---|---|
| React Native | 0.76+ | Framework mobile |
| Expo SDK | 52+ | Managed workflow, OTA updates |
| expo-router | 4.x | File-based routing + deep links nativos |
| React Native Reanimated | 3.x | Animações na UI thread |
| React Native Gesture Handler | 2.x | Gestos: swipe, long press, drag |
| TanStack Query | 5.x | Cache e sync de dados |
| Zustand | 5.x | Estado global + persistência AsyncStorage |
| expo-secure-store | latest | Refresh tokens (criptografado) |
| expo-camera | latest | Câmera para selfie KYC |
| expo-image-picker | latest | Galeria para documentos KYC |
| expo-document-picker | latest | PDFs para documentos KYC |
| Expo Notifications | latest | Push notifications |
| expo-local-authentication | latest | Biometria para re-auth |
| expo-haptics | latest | Feedback háptico |
| @react-native-community/netinfo | latest | Detecção de estado de rede |

### 1.3 Dispositivos de Referência para Testes

| Dispositivo | Plataforma | Propósito |
|---|---|---|
| iPhone 15 Pro | iOS 17 | Referência primária — Dynamic Island |
| iPhone 13 | iOS 16 | Versão mínima iOS |
| iPhone SE (3a gen) | iOS 16 | Tela menor (375pt) |
| Samsung Galaxy S23 | Android 13 (API 33) | Referência Android moderna |
| Samsung Galaxy A53 | Android 12 (API 32) | Referência mid-range Android |
| Moto G Pure | Android 10 (API 29) | Versão mínima Android |

### 1.4 Resolução e Densidade

| Categoria | Density | Resolução referência | Suporte |
|---|---|---|---|
| mdpi | 1x | 320×480 | Suportado (descontinuado em novas telas) |
| hdpi | 1.5x | 480×800 | Suportado |
| xhdpi | 2x | 720×1280 | Suportado |
| xxhdpi | 3x | 1080×1920 | Primário Android |
| xxxhdpi | 4x | 1440×2560 | Suportado (high-end) |
| @2x / @3x | iOS scale | iPhone padrão / Pro | Primário iOS |

Layout flexível com `Dimensions.get('window')` e constantes de layout. Nunca valores hardcoded em `StyleSheet`.

---

## 2. Arquitetura de Navegação

### 2.1 Padrão Adotado

**expo-router 4** com file-based routing — estrutura de arquivos em `app/` mapeia diretamente para rotas. Grupos de rotas com `(tabs)` para tabs e `(auth)` para autenticação.

```
app/
├── _layout.tsx              # RootLayout — Providers, ThemeProvider, AuthGuard
├── (auth)/
│   ├── _layout.tsx          # AuthLayout — sem tabs, sem header
│   ├── login.tsx            # T-AUTH-01
│   ├── cadastro.tsx         # T-AUTH-02
│   ├── recuperar-senha.tsx  # T-AUTH-03
│   └── nova-senha.tsx       # T-AUTH-04
├── (tabs)/
│   ├── _layout.tsx          # TabLayout — 5 tabs com safe-area
│   ├── dashboard/
│   │   └── index.tsx        # T-DASH-01
│   ├── marketplace/
│   │   ├── index.tsx        # T-OPR-01
│   │   ├── [id]/
│   │   │   ├── index.tsx    # T-OPR-02
│   │   │   └── proposta.tsx # T-OPR-03
│   ├── operacoes/
│   │   ├── propostas/
│   │   │   ├── index.tsx    # T-PRP-01
│   │   │   └── [id].tsx     # T-PRP-02
│   │   ├── negociacoes/
│   │   │   ├── index.tsx    # T-NEG-01
│   │   │   ├── [id]/
│   │   │   │   ├── index.tsx # T-NEG-02 (chat)
│   │   │   │   ├── escrow.tsx # T-NEG-03
│   │   │   │   └── escrow-status.tsx # T-NEG-04
│   │   └── formalizacoes/
│   │       ├── [id]/
│   │       │   ├── documentos.tsx # T-ASS-01
│   │       │   ├── assinatura.tsx # T-ASS-02
│   │       │   └── zapsign.tsx    # T-ASS-03
│   ├── financeiro/
│   │   ├── index.tsx        # T-FIN-01
│   │   ├── [id].tsx         # T-FIN-02
│   │   └── comprovante/[id].tsx # T-FIN-03
│   └── perfil/
│       ├── index.tsx        # T-PRF-01
│       ├── kyc.tsx          # T-PRF-02 (full-screen modal)
│       ├── kyc-status.tsx   # T-PRF-03
│       ├── editar.tsx       # T-PRF-04
│       └── configuracoes.tsx # T-PRF-05
└── analista.tsx             # T-IA-01 (modal full-screen)
```

### 2.2 Hierarquia de Navegadores

```
RootStack (Stack.Navigator)
├── AuthGroup (não autenticado)
│   ├── Login
│   ├── Cadastro
│   ├── RecuperarSenha
│   └── NovaSenha
└── TabNavigator (autenticado, KYC_APROVADO)
    ├── Tab: Dashboard (ícone Home)
    ├── Tab: Marketplace (ícone Search)
    ├── Tab: Operações (ícone FileText) — badge contador
    ├── Tab: Financeiro (ícone DollarSign)
    └── Tab: Perfil (ícone User) — badge KYC
    └── Modal (sobre tabs): Analista IA (botão FAB no Dashboard)
    └── Modal KYC (full-screen): T-PRF-02
    └── Modal ZapSign: T-ASS-03
```

### 2.3 Mapeamento Tela → Tipo de Navegação → Transição

| Tela | Tipo | Transição iOS | Transição Android | Deep Link |
|---|---|---|---|---|
| T-AUTH-01 (Login) | Stack (raiz) | Fade | Fade | `repasse://login` |
| T-AUTH-02 (Cadastro) | Stack push | Slide from right | Fade up | `repasse://cadastro` |
| T-AUTH-03 (Recuperar) | Stack push | Slide from right | Fade up | Não |
| T-AUTH-04 (Nova Senha) | Stack push | Slide from right | Fade up | `repasse://reset-senha?token=X` |
| T-DASH-01 (Dashboard) | Tab root | Nenhuma (tab switch) | Nenhuma | `repasse://dashboard` |
| T-OPR-01 (Marketplace) | Tab root | Nenhuma | Nenhuma | `repasse://marketplace` |
| T-OPR-02 (Detalhe OPR) | Stack push | Slide from right | Fade up | `repasse://oportunidade/{id}` |
| T-OPR-03 (Proposta) | Modal bottom sheet | Slide from bottom | Slide from bottom | Não |
| T-PRP-01 (Propostas) | Tab stack | Nenhuma | Nenhuma | `repasse://propostas` |
| T-PRP-02 (Detalhe prop) | Stack push | Slide from right | Fade up | `repasse://proposta/{id}` |
| T-NEG-01 (Negociações) | Tab stack | Nenhuma | Nenhuma | `repasse://negociacoes` |
| T-NEG-02 (Chat neg) | Stack push | Slide from right | Fade up | `repasse://negociacao/{id}` |
| T-NEG-03 (Escrow) | Stack push | Slide from right | Fade up | `repasse://negociacao/{id}/escrow` |
| T-NEG-04 (Escrow status) | Stack push | Slide from right | Fade up | `repasse://negociacao/{id}/escrow-status` |
| T-ASS-01 (Documentos) | Stack push | Slide from right | Fade up | `repasse://formalizacao/{id}` |
| T-ASS-02 (Assinatura) | Stack push | Slide from right | Fade up | `repasse://formalizacao/{id}/assinar` |
| T-ASS-03 (ZapSign) | Modal full-screen | Slide from bottom | Slide from bottom | Não |
| T-FIN-01 (Financeiro) | Tab root | Nenhuma | Nenhuma | `repasse://financeiro` |
| T-FIN-02 (Transação) | Stack push | Slide from right | Fade up | `repasse://transacao/{id}` |
| T-FIN-03 (Comprovante) | Stack push | Slide from right | Fade up | Não |
| T-IA-01 (Analista IA) | Modal full-screen | Slide from bottom | Slide from bottom | `repasse://analista?opportunity_id={id}` |
| T-PRF-01 (Perfil) | Tab root | Nenhuma | Nenhuma | `repasse://perfil` |
| T-PRF-02 (KYC) | Modal full-screen | Slide from bottom | Slide from bottom | `repasse://kyc` |
| T-PRF-03 (KYC status) | Stack push | Slide from right | Fade up | `repasse://kyc/status` |
| T-PRF-04 (Editar perfil) | Stack push | Slide from right | Fade up | Não |
| T-PRF-05 (Config) | Stack push | Slide from right | Fade up | Não |

### 2.4 Comportamento Botão Voltar por Plataforma

| Situação | iOS | Android |
|---|---|---|
| Stack com tela anterior | Swipe-back desde a borda esquerda; tap no `<` do header | Botão hardware/software back |
| Modal full-screen (KYC, IA, ZapSign) | Arraste para baixo (swipe down to dismiss) | Botão back fecha modal; não permite swipe horizontal |
| Tab raiz (Dashboard, Marketplace) | Nenhum — primeiro item do stack | Botão back minimiza o app (não navega para outra tab) |
| Bottom sheet (T-OPR-03 Proposta) | Swipe down to dismiss | Botão back fecha sheet |
| Dialog/Modal de confirmação | Tap fora ou botão "Cancelar" | Botão back = "Cancelar" |
| Tela de autenticação (Login) | n/a — não há voltar | Botão back fecha o app |

⚙️ **Proteção de navegação:** formulários com dados não salvos (KYC step, proposta) exibem `Alert.alert("Deseja sair?", "Suas informações serão perdidas.")` ao tentar voltar.

---

## 3. Gestos e Interações Nativas

### 3.1 Tabela de Gestos

| Gesto | Contexto | Ação | Conflito nativo | Feedback háptico | Alternativa acessível |
|---|---|---|---|---|---|
| **Swipe-back** (iOS) | Qualquer tela em Stack | Voltar à tela anterior | Nenhum — gesto nativo iOS | Nenhum | Tap no botão `<` do header |
| **Swipe down** | Modais (KYC, IA, ZapSign, Bottom Sheet) | Fechar modal | Nenhum no iOS; no Android conflito com sistema se swipe vertical ativo | `ImpactFeedbackStyle.Light` ao soltar | Botão "Fechar" no topo do modal |
| **Pull-to-refresh** | Listas (Marketplace, Propostas, Negociações, Financeiro) | Recarregar dados | Conflito com scroll se `scrollEnabled` inconsistente — resolver com `RefreshControl` nativo | `ImpactFeedbackStyle.Medium` ao disparar refresh | Botão "Atualizar" no header (ícone `RefreshCw`) |
| **Swipe left** em item de lista | T-PRP-01 (proposta ativa), T-NEG-01 (negociação) | Revelar ação contextual ("Cancelar", "Ver detalhes") | No Android: conflito com swipe-back do sistema — usar apenas swipe left (não right) | `ImpactFeedbackStyle.Medium` ao revelar | Tap no item → menu de ações |
| **Long press** | Card de oportunidade (T-OPR-01), item de proposta | Exibir context menu (compartilhar, ver detalhes) | Nenhum | `ImpactFeedbackStyle.Heavy` | Tap normal → detalhes |
| **Double tap** | Valor monetário (T-OPR-02, T-NEG-02) | Copiar valor para clipboard | Nenhum | `NotificationFeedbackType.Success` | Tap no ícone `Copy` ao lado do valor |
| **Pinch-to-zoom** | Documentos de formalização (T-ASS-01), comprovante de depósito (T-FIN-03) | Zoom no documento | Nenhum | Nenhum | Botão "Ampliar" no toolbar |

### 3.2 Feedback Háptico por Ação de Domínio

| Ação | Háptico |
|---|---|
| Proposta enviada com sucesso | `NotificationFeedbackType.Success` |
| Proposta cancelada | `NotificationFeedbackType.Warning` |
| Erro de validação (quota atingida, KYC pendente) | `NotificationFeedbackType.Error` |
| Contraproposta enviada | `ImpactFeedbackStyle.Medium` |
| Upload de documento concluído | `NotificationFeedbackType.Success` |
| Cópia de PIX/dados bancários | `ImpactFeedbackStyle.Light` |
| Botão de ação primária (long press implícito) | `ImpactFeedbackStyle.Medium` |

🔴 **Proibido:** feedback háptico em ações de listagem/scroll — apenas em ações de negócio com resultado definitivo.

---

## 4. Comportamento Offline

### 4.1 Estratégia de Cache Local

**Tecnologia:** TanStack Query com `AsyncStorage` como `persistQueryClient` (dados não sensíveis). `expo-secure-store` para tokens.

| Parâmetro | Valor | Justificativa |
|---|---|---|
| Cache GC time | 24h | Dados de oportunidade e propostas precisam estar disponíveis por um dia sem rede |
| Stale time | 5min | Dados considerados frescos por 5 min após fetch |
| Máximo de itens em cache | 100 oportunidades + 50 propostas + 20 negociações | Limitado por AsyncStorage (max ~6MB por chave no iOS) |
| Dados NUNCA persistidos localmente | Tokens de autenticação (em `expo-secure-store`), dados KYC, documentos de formalização | Segurança LGPD — documentos sensíveis não ficam no cache offline |

### 4.2 Telas com Suporte Offline

| Tela | Dados em cache | Ações offline disponíveis | Ações offline bloqueadas | Indicador visual |
|---|---|---|---|---|
| T-DASH-01 (Dashboard) | KPIs da última sessão | Visualizar KPIs | Atualizar dados, acessar IA | Banner amarelo "Você está offline" |
| T-OPR-01 (Marketplace) | Últimas 100 oportunidades | Visualizar lista, filtrar localmente | Carregar mais, atualizar | Banner offline + badge "Dados de [hora]" |
| T-OPR-02 (Detalhe OPR) | Dados da oportunidade visitada | Visualizar detalhes, score IA (cache) | Análise IA nova, fazer proposta | Banner offline |
| T-PRP-01 (Propostas) | Todas as propostas ativas | Visualizar lista | Cancelar proposta, criar nova | Banner offline |
| T-PRP-02 (Detalhe prop) | Detalhes da proposta | Visualizar detalhes | Cancelar | Banner offline |
| T-NEG-01 (Negociações) | Todas as negociações | Visualizar lista | Acessar negociação (chat requer rede) | Banner offline |
| T-NEG-02 (Chat negociação) | Histórico de mensagens | Ler histórico | Enviar mensagem, contraproposta | Banner offline + input desabilitado |
| T-FIN-01 (Financeiro) | Últimas 50 transações | Visualizar histórico | Filtrar novo período | Banner offline |
| T-PRF-01 (Perfil) | Dados do perfil, status KYC | Visualizar perfil | Editar perfil, enviar KYC | Banner offline |
| T-PRF-05 (Config) | Preferências de notificação | Visualizar | Alterar preferências | Banner offline |

**Telas sem suporte offline** (exigem rede):
- T-AUTH-* — autenticação sempre online.
- T-OPR-03 — proposta exige validação de quota em tempo real.
- T-NEG-03 / T-NEG-04 — Escrow exige confirmação no servidor.
- T-ASS-* — documentos ZapSign exigem rede.
- T-IA-01 — Analista de IA exige OpenAI online.

### 4.3 Fila de Ações Offline

[DECISÃO AUTÔNOMA — fila persistida em AsyncStorage como array JSON. Alternativa descartada: Redux Offline (overhead desnecessário com TanStack Query). Justificativa: TanStack Query `useMutation` com `retry` e `onSettled` já fornece retry automático; ações críticas (proposta, cancelamento) são enfileiradas via Zustand store persistido.]

| Ação enfileirável | Condição | Sincronização ao reconectar |
|---|---|---|
| Alteração de preferências de notificação | Offline ao salvar | Auto-sync ao reconectar (silencioso) |
| Marcar notificação como lida | Offline ao tap | Auto-sync ao reconectar (silencioso) |

🔴 **Ações não enfileiráveis** (exigem confirmação online imediata):
- Envio de proposta (validação de quota Redis em tempo real).
- Cancelamento de proposta (atualização de status crítica).
- Envio de contraproposta (validação de rodadas).
- Confirmação de depósito Escrow.

### 4.4 Indicadores Visuais de Estado de Rede

- **Banner offline global:** `<OfflineBanner>` com fundo `#F59E0B` (âmbar) + ícone `WifiOff` + texto "Você está offline. Algumas funcionalidades estão indisponíveis." — exibido no topo da screen, abaixo do header, usando `@react-native-community/netinfo`.
- **Dados em cache:** badge `"Atualizado às HH:mm"` em cinza (`--muted-foreground`) abaixo do título da listagem.
- **Botão de ação primária offline:** desabilitado + tooltip "Conecte-se à internet para realizar esta ação."
- **Reconexão:** banner desaparece com `AnimatePresence` (fade out 300ms) + TanStack Query invalida queries automaticamente via `netInfo.addEventListener`.

### 4.5 Resolução de Conflitos

Ao reconectar após ações offline:
1. TanStack Query refaz todas as queries marcadas como stale.
2. Se dados locais diferem dos dados do servidor (ex: status de proposta mudou enquanto offline), os dados do servidor prevalecem sem notificação (servidor é fonte de verdade).
3. Exceção: se ação enfileirada falha no servidor (ex: preferência salva conflita com nova regra), `Alert.alert` informa o usuário com a mensagem de erro do servidor.

---

## 5. Push Notifications

### 5.1 Tipos e Deep Links por Notificação

| Código | Título | Corpo | Deep Link | Prioridade |
|---|---|---|---|---|
| NOT-CES-01 | "KYC Aprovado!" | "Sua identidade foi verificada. Você já pode fazer propostas." | `repasse://marketplace` | Alta |
| NOT-CES-02 | "KYC Pendente" | "Um documento precisou ser reenviado. Verifique sua conta." | `repasse://kyc/status` | Alta |
| NOT-CES-03 | "Proposta Recebida pelo Admin" | "Sua proposta para OPR-XXXX está em análise." | `repasse://proposta/{id}` | Normal |
| NOT-CES-04 | "Proposta Aceita!" | "Sua proposta para OPR-XXXX foi aceita. Negociação iniciada." | `repasse://negociacao/{id}` | Alta |
| NOT-CES-05 | "Depósito Escrow Pendente" | "Realize o depósito de R$ X em X dias úteis." | `repasse://negociacao/{id}/escrow` | Alta |
| NOT-CES-06 | "Depósito Confirmado!" | "Seu depósito foi confirmado. Formalização liberada." | `repasse://formalizacao/{id}` | Alta |
| NOT-CES-07 | "Prazo de Depósito Venceu" | "O prazo de depósito expirou. A operação foi cancelada." | `repasse://negociacoes` | Alta |
| NOT-CES-08 | "Documento para Assinar" | "O instrumento de cessão está disponível para sua assinatura." | `repasse://formalizacao/{id}/assinar` | Alta |
| NOT-CES-09 | "Operação Concluída!" | "Parabéns! Sua cessão foi concluída com sucesso." | `repasse://financeiro` | Alta |
| NOT-CES-10 | "Operação Revertida" | "A reversão da operação foi iniciada." | `repasse://financeiro` | Normal |
| NOT-CES-11 | "Prazo de Reversão" | "Você tem X dias para solicitar reversão desta operação." | `repasse://financeiro` | Normal |

### 5.2 Permissões por Plataforma

| Plataforma | Comportamento padrão | Momento da solicitação | Mensagem de contexto |
|---|---|---|---|
| iOS | Permissão explícita obrigatória | Após conclusão do cadastro (T-AUTH-02), com contexto "Para receber alertas sobre propostas, depósitos e documentos" | `expo-notifications` `requestPermissionsAsync()` |
| Android 12- (API ≤ 31) | Concedida automaticamente (POST_NOTIFICATIONS não existia) | n/a | n/a |
| Android 13+ (API 33+) | Permissão `POST_NOTIFICATIONS` obrigatória | Após cadastro, mesmo fluxo do iOS | `PermissionsAndroid.request(POST_NOTIFICATIONS)` |

**Tratamento de recusa:**
- Exibir `Alert.alert("Notificações desabilitadas", "Sem notificações push, você pode perder alertas críticos de prazo de Escrow. Ative nas configurações.", [{text: "Abrir Configurações", onPress: () => Linking.openSettings()}, {text: "Continuar sem notificações"}])`.
- Fallback obrigatório: e-mail é o canal mínimo garantido (RN-069) — o fluxo continua funcionando sem push.
- Não solicitar novamente na mesma sessão após recusa — respeitar `ios.status !== 'denied'`.

### 5.3 Comportamento por Estado do App

| Estado | Comportamento |
|---|---|
| **Foreground** | Notificação exibida como `in-app banner` (componente customizado no topo, 4s) — NÃO via sistema operacional. Tap no banner navega para deep link. |
| **Background** | Notificação do sistema operacional (standard push). Tap abre app e navega para deep link via `expo-notifications` `addNotificationResponseReceivedListener`. |
| **App killed (terminated)** | Notificação do sistema operacional. Tap abre app na tela do deep link. `getLastNotificationResponseAsync()` no startup para capturar tap em cold start. |

### 5.4 Agrupamento e Prioridade

- **Agrupamento iOS:** todas as notificações do app agrupadas por bundle ID no Centro de Notificações.
- **Agrupamento Android:** canal `repasse_alerts` para notificações de alta prioridade (KYC, Escrow, Assinatura) e `repasse_updates` para normais.
- **Priority Android:** `HIGH` para NOT-CES-01, 04, 05, 06, 07, 08, 09. `DEFAULT` para demais.
- **Badge:** contador no ícone do app — incrementado por notificação recebida, zerado ao abrir o app.

### 5.5 Opt-out e Preferências

Gerenciado em T-PRF-05 (Configurações). Preferências sincronizadas com backend (tabela `cessionarios.notification_preferences`).

| Canal | Pode desabilitar? | Fallback |
|---|---|---|
| Push | Sim | E-mail permanece ativo (canal mínimo) |
| E-mail | Não (RN-069) | E-mail é obrigatório |

---

## 6. Deep Links e Universal Links

### 6.1 Schema de URLs

- **Custom scheme:** `repasse://` (ambas plataformas — fallback universal)
- **Universal Links (iOS):** `https://app.repasseseguro.com.br/*` via `apple-app-site-association`
- **App Links (Android):** `https://app.repasseseguro.com.br/*` via `assetlinks.json`

Expo Router cuida do registro de esquemas e dos handlers automaticamente com o arquivo `app.json`:
```json
{
  "expo": {
    "scheme": "repasse",
    "intentFilters": [...]
  }
}
```

### 6.2 Mapeamento Completo de Deep Links

| Deep Link | Tela | Parâmetros | Autenticação necessária |
|---|---|---|---|
| `repasse://login` | T-AUTH-01 | — | Não |
| `repasse://cadastro` | T-AUTH-02 | — | Não |
| `repasse://reset-senha` | T-AUTH-04 | `?token=JWT_TOKEN` | Não (token na URL) |
| `repasse://dashboard` | T-DASH-01 | — | Sim |
| `repasse://marketplace` | T-OPR-01 | — | Sim |
| `repasse://oportunidade/{id}` | T-OPR-02 | `id: UUID` | Sim |
| `repasse://propostas` | T-PRP-01 | — | Sim |
| `repasse://proposta/{id}` | T-PRP-02 | `id: UUID` | Sim |
| `repasse://negociacoes` | T-NEG-01 | — | Sim |
| `repasse://negociacao/{id}` | T-NEG-02 | `id: UUID` | Sim |
| `repasse://negociacao/{id}/escrow` | T-NEG-03 | `id: UUID` | Sim |
| `repasse://negociacao/{id}/escrow-status` | T-NEG-04 | `id: UUID` | Sim |
| `repasse://formalizacao/{id}` | T-ASS-01 | `id: UUID` | Sim |
| `repasse://formalizacao/{id}/assinar` | T-ASS-02 | `id: UUID` | Sim |
| `repasse://financeiro` | T-FIN-01 | — | Sim |
| `repasse://transacao/{id}` | T-FIN-02 | `id: UUID` | Sim |
| `repasse://analista` | T-IA-01 | `?opportunity_id=UUID` (opcional) | Sim |
| `repasse://kyc` | T-PRF-02 | — | Sim |
| `repasse://kyc/status` | T-PRF-03 | — | Sim |
| `repasse://perfil` | T-PRF-01 | — | Sim |

### 6.3 Tratamento de Deep Link com Autenticação

Quando deep link exige autenticação e usuário não está logado:

1. Salvar URL destino em Zustand: `useAuthStore.setState({ pendingDeepLink: url })`.
2. Redirecionar para T-AUTH-01 (Login).
3. Após login bem-sucedido: verificar `pendingDeepLink` e navegar para destino.
4. Timeout: se login não ocorrer em 5 min, limpar `pendingDeepLink`.

### 6.4 Fallback Web

Se o app não estiver instalado:
- Universal Link / App Link redireciona para `https://app.repasseseguro.com.br/{rota}` — versão web do produto.
- Custom scheme (`repasse://`) exibe erro de sistema "Nenhum app encontrado para abrir este link" — aceitável para links internos.

---

## 7. Permissões Nativas

### 7.1 Tabela de Permissões

| Permissão | Feature | Momento da solicitação | Mensagem de contexto | Fallback se negada |
|---|---|---|---|---|
| `Camera` | Selfie de prova de vida (KYC etapa 3) | Ao iniciar a etapa 3 do KYC stepper | "Para a prova de vida, precisamos acessar sua câmera. Nenhuma foto é armazenada sem sua confirmação." | Oferecer opção de upload da galeria (foto existente); informar que selfie ao vivo é preferível para aprovação |
| `MediaLibrary` / `Photos` | Upload de documentos KYC (identidade, comprovante) | Ao tocar em "Escolher da galeria" na etapa 1 ou 2 do KYC | "Para enviar seu documento de identidade, precisamos acessar sua galeria de fotos." | Oferecer opção de câmera para captura ao vivo |
| `POST_NOTIFICATIONS` (Android 13+) | Push notifications | Após conclusão do cadastro | "Para receber alertas de prazo de Escrow e assinaturas pendentes, ative as notificações." | Canal e-mail permanece ativo; banner permanente no Dashboard informando que notificações estão desabilitadas |
| `LocalAuthentication` (Biometria) | Re-autenticação em ações críticas | Ao primeiro uso de re-auth (proposta, cancelamento) | "Usaremos sua biometria para confirmar sua identidade em ações importantes." | Fallback para senha (campo Input) |

### 7.2 Tratamento de Revogação de Permissão

| Permissão revogada | Comportamento |
|---|---|
| Câmera revogada após concessão | Ao tentar usar câmera no KYC: `Alert` com botão "Abrir Configurações" via `Linking.openSettings()` |
| Fotos revogadas | Ao tentar acessar galeria: mesmo fluxo de câmera |
| Notificações revogadas | Banner persistente no Dashboard: "Notificações desabilitadas. [Ativar]" → `Linking.openSettings()` |
| Biometria revogada | Fallback automático para input de senha sem notificação de erro |

---

## 8. Diferenças iOS vs. Android

### 8.1 Tabela Comparativa

| Feature | iOS | Android |
|---|---|---|
| **Navegação voltar** | Swipe-back desde borda esquerda (nativo) | Botão hardware/software back |
| **Modal dismiss** | Swipe down para fechar | Botão back fecha; sem swipe horizontal |
| **Push permission** | Diálogo explícito obrigatório na primeira vez | Automático Android 12-; `POST_NOTIFICATIONS` Android 13+ |
| **Biometria** | Face ID (iPhone X+) ou Touch ID (SE, iPad) | Fingerprint (maioria) ou Face Unlock (high-end) |
| **Keyboard avoidance** | `KeyboardAvoidingView behavior="padding"` | `behavior="height"` + `windowSoftInputMode="adjustResize"` |
| **Safe area** | `useSafeAreaInsets()` — notch/Dynamic Island top; home indicator bottom | Não há home indicator; status bar altura variável |
| **Bottom tab bar** | Acima do home indicator (safe area) | Acima do sistema de navegação gestual |
| **Status bar** | Controlada via `expo-status-bar` | Translúcida em Android 10+ via `translucent` |
| **Haptics** | `expo-haptics` (iOS apenas; falha silenciosa no Android) | `expo-haptics` com suporte parcial — `impactAsync` nem sempre disponível |
| **Share sheet** | `Share.share()` — sheet nativa iOS | `Share.share()` — intent chooser Android |
| **Camera UI** | `expo-camera` — HUD nativo iOS | `expo-camera` — HUD nativo Android |
| **Fonts** | San Francisco como fallback de sistema | Roboto como fallback de sistema |
| **Scroll bounce** | Bounce efeito nativo no fim do scroll | Sem bounce — efeito de overscroll (glow) |
| **Tooltip** | Long press com UIMenu (iOS 14+) | Tooltip via `TooltipProps` do componente |
| **Clipboard** | `expo-clipboard` — sincroniza via Handoff | `expo-clipboard` — sem Handoff |
| **Background fetch** | Limitado pelo iOS (AppRefreshTask) | Mais permissivo com WorkManager |
| **Deep link handling** | Universal Links com `apple-app-site-association` | App Links com `assetlinks.json` |
| **OTA Updates** | Permitido pela App Store (JS-only changes) | Permitido pela Play Store |
| **Build** | EAS Build (Xcode Cloud ou local Mac) | EAS Build (Linux cloud) |

### 8.2 Comportamentos Específicos de Plataforma

**iOS — Safe Area:**
- `<SafeAreaView>` obrigatório em todas as telas com conteúdo próximo às bordas.
- Dynamic Island (iPhone 14 Pro+): status bar area de 59pt vs. 47pt no notch.
- Home indicator: padding bottom mínimo de 34pt em telas full-screen.

**Android — Navigation Gestures:**
- Em dispositivos com gesture navigation (Android 10+): swipe horizontal pode conflitar com gestos do app.
- Solução: `gestureResponseDistance` de 10px nos Stack navigators para prevenir false positives.
- Botão back Android em Stack push navega para tela anterior; em tab raiz minimiza o app.

**Teclado — campos monetários:**
- iOS: `keyboardType="decimal-pad"` — usa vírgula como separador decimal.
- Android: `keyboardType="number-pad"` — sem vírgula nativa; parse manual com `parseFloat(value.replace(',', '.'))`.
- Formatação de moeda sempre em backend (nunca depende do locale do dispositivo).

---

## 9. Performance Mobile

### 9.1 Métricas-Alvo

| Métrica | Target | Ferramenta de medição |
|---|---|---|
| Cold start (app killed → tela inicial) | < 3s | `expo-startup-time`, Sentry Performance |
| Warm start (background → foreground) | < 800ms | Sentry Performance |
| TTI (Time to Interactive) Dashboard | < 2s | Sentry Performance |
| FPS durante scroll de listas | 60fps (ou 120fps em Pro Motion) | React Native Debugger, Flipper |
| Tamanho do bundle JS (OTA update) | < 2MB gzipped | EAS Build stats |
| Tamanho do app instalado (iOS) | < 80MB | App Store Connect |
| Tamanho do app instalado (Android) | < 50MB | Play Console |
| Latência de fetch de lista (Marketplace) | < 500ms | Sentry Performance |

### 9.2 Otimização de Listas

- **`FlashList`** (Shopify) como substituição do `FlatList` para todas as listas longas (Marketplace, Propostas, Negociações, Financeiro) — estimatedItemSize obrigatório.
- **`getItemLayout`** em listas com item heights fixas.
- **Paginação:** cursor-based com `fetchNextPage` do TanStack Query ao atingir 80% do scroll.
- **Skeleton:** `<SkeletonItem>` com `Reanimated` pulse animation — não spinner.
- **Janela de renderização:** `windowSize={5}` no FlashList para performance em listas longas.

### 9.3 Otimização de Imagens

- **`expo-image`** com `contentFit="cover"` e `placeholder={blurHash}` para todas as imagens remotas.
- Formatos: WebP quando disponível; JPEG como fallback.
- `recyclingKey` nos itens de lista para reutilizar instâncias de imagem.
- **Lazy loading:** imagens fora da viewport não são carregadas (padrão `expo-image`).

### 9.4 Monitoramento

- **Sentry Performance:** traces automáticos em navegação de telas e HTTP requests.
- **PostHog:** eventos de performance (`startup_time`, `screen_load_time`).
- **Hermes JS engine:** obrigatório (default em Expo 52). JSC proibido em produção.
- **React Compiler:** habilitado via `babel-plugin-react-compiler` para auto-memoização.

---

## 10. Acessibilidade Mobile

### 10.1 VoiceOver (iOS) e TalkBack (Android)

| Requisito | Implementação | Critério de aceite |
|---|---|---|
| Labels em botões icônicos | `accessibilityLabel="Enviar proposta"` em todos os `<Pressable>` sem texto visível | VoiceOver/TalkBack anuncia ação correta |
| Live regions | `accessibilityLiveRegion="polite"` em contadores (SLA, quota), `"assertive"` em erros críticos | Screen reader anuncia atualizações |
| Grupos de acessibilidade | `accessible={true} accessibilityRole="button"` em cards compostos (OpportunityCard) | Card inteiro anunciado como unidade |
| Estados | `accessibilityState={{ disabled: true }}` em botões desabilitados (quota atingida, offline) | VoiceOver/TalkBack anuncia estado |
| Ordem de focus | `accessibilityViewIsModal={true}` em modais | Focus não sai do modal |
| Hints | `accessibilityHint="Toque para abrir os detalhes desta oportunidade"` em cards | Instruções de uso disponíveis |

### 10.2 Tamanhos Mínimos de Toque

| Plataforma | Tamanho mínimo | Implementação |
|---|---|---|
| iOS | 44 × 44pt | `minWidth: 44, minHeight: 44` + `hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}` |
| Android | 48 × 48dp | `minWidth: 48, minHeight: 48` + `hitSlop` |

Todos os CTAs principais (Fazer Proposta, Enviar, Confirmar) com `height: 52` para superar o mínimo em ambas as plataformas.

### 10.3 Contraste e Tipografia

- Contraste mínimo 4.5:1 para texto normal, 3:1 para texto grande (>18pt ou 14pt bold).
- **Dynamic Type (iOS):** `Text` sempre com `allowFontScaling={true}` (padrão) — nunca `allowFontScaling={false}`.
- **Font scaling (Android):** respeitar `fontScale` do sistema — layouts flexíveis sem altura fixa em `Text`.
- Tamanho mínimo de fonte: 14sp/14pt para texto de corpo. 12sp/12pt apenas para labels secundários.

### 10.4 Navegação por Teclado Externo (iPad + Android tablets)

- Tab order lógico via `TabIndex` em formulários.
- Atalhos de teclado: `Enter` para submit em formulários, `Esc` para fechar modais.
- Focus visível em todos os elementos interativos (`focusStyle` obrigatório no design system).

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 22/03/2026 | v1.0 | Criação inicial — Pipeline ShiftLabs v9.5. Cobertura completa: 26 telas, plataformas iOS 16+/Android 10+, navegação expo-router, 8 telas offline, 11 tipos de push com deep link, 4 permissões nativas, tabela iOS vs. Android, performance targets, acessibilidade VoiceOver/TalkBack. |

---

## 12. Backlog de Pendências

| Item | Tipo | Seção | Impacto | Justificativa / Decisão | Dono | Status |
|---|---|---|---|---|---|---|
| Fila offline implementada via Zustand + AsyncStorage (sem Redux Offline) | Decisão Autônoma | §4.3 | P1 | TanStack Query + Zustand já fornece retry; Redux Offline é overhead desnecessário no stack Expo; alternativa descartada: WatermelonDB (too heavy para casos de uso simples) | Mobile Lead | Decidido |
| Haptics em Android: `expo-haptics` com suporte parcial | Decisão Autônoma | §3.2 | P2 | API `impactAsync` nem sempre disponível em todos os dispositivos Android; falha silenciosa é aceitável; alternativa descartada: vibração nativa (muito grosseira) | Mobile Lead | Decidido |
| Bottom sheet de proposta (T-OPR-03) como modal, não stack | Decisão Autônoma | §2.3 | P2 | Bottom sheet preserva contexto do detalhe da oportunidade; usuário pode descartar facilmente; alternativa: stack push (menos fluido, perde contexto visual) | Mobile Lead | Decidido |
| `FlashList` (Shopify) no lugar de `FlatList` | Decisão Autônoma | §9.2 | P1 | Performance 10x superior em listas com 100+ items; `FlatList` descartado por janela de renderização ineficiente; FlashList é dependência extra mas amplamente adotado no ecossistema React Native | Mobile Lead | Decidido |
| iOS: oferecer galeria como fallback se câmera recusada no KYC | Decisão Autônoma | §7.2 | P1 | selfie ao vivo é preferível para idwall mas não obrigatória; permitir upload de foto recente reduz abandono no KYC; alternativa: bloquear totalmente (aumenta abandono, inaceitável) | Mobile Lead | Decidido |
