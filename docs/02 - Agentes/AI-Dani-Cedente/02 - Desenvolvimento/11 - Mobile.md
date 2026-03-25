# 11 - Mobile — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Produto, Mobile e Engenharia |
| Escopo | Especificação mobile com navegação, gestos, offline, push e diferenças de plataforma |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D01 · D02 · D05 · D06 · D07 · D09 |

---

> **📌 TL;DR**
>
> - **Fase 1:** Chat da Dani é um widget web responsivo embutido na plataforma Repasse Seguro — não é app standalone. Mobile usa `ChatFullscreenLayout` (100vw × 100dvh) via PWA ou WebView no app nativo.
> - **Fase 2 planejada:** Integração do chat como componente React Native nativo no app Repasse Seguro (Expo + React Native Reanimated 3.x+).
> - **Plataformas:** iOS 16+ e Android 10+ (para PWA); para integração nativa futura: iOS 16+ / Android 10+.
> - **Offline:** Chat requer conexão ativa (SSE streaming). Histórico de sessão disponível via cache local (últimas 20 mensagens).
> - **Push notifications:** 10 eventos proativos entregues via Web Push (Fase 1) e React Native Push (Fase 2).
> - **Animações mobile:** React Native Reanimated 3.x+ como equivalente do Framer Motion para Fase 2 (conforme D02 e D04).
> - **Zero seções pendentes** — cobertura completa com distinção clara Fase 1 / Fase 2.

---

## 1. Plataformas e Versões Suportadas

### 1.1 Fase 1 — Web responsiva (PWA/WebView)

| Plataforma | Versão mínima | Observação |
|---|---|---|
| iOS (Safari) | 16.0+ | `100dvh` suportado a partir do iOS 15.4; usar `100dvh` conforme D07 |
| Android (Chrome) | 10+ (Chrome 90+) | `100dvh` e SSE suportados |
| PWA | Qualquer plataforma com browser moderno | Instalável via "Adicionar à tela inicial" |

### 1.2 Fase 2 — React Native nativo (planejado)

| Plataforma | Versão mínima | SDK | Observação |
|---|---|---|---|
| iOS | 16.0+ | React Native 0.73+ / Expo SDK 50+ | Integração como componente nativo no app Repasse Seguro |
| Android | 10+ (API 29+) | React Native 0.73+ / Expo SDK 50+ | Animações via Reanimated 3.x+ |

### 1.3 Dispositivos de referência para testes

| Dispositivo | Plataforma | Resolução | Prioridade |
|---|---|---|---|
| iPhone 15 (Safari) | iOS 17 | 390×844pt | P0 |
| iPhone SE 3ª gen (Safari) | iOS 16 | 375×667pt | P1 — menor tela suportada |
| Samsung Galaxy A54 (Chrome) | Android 13 | 393×851dp | P0 |
| Pixel 7 (Chrome) | Android 13 | 412×915dp | P1 |

---

## 2. Arquitetura de Navegação

### 2.1 Fase 1 — Web

O chat não tem navegação própria — é um widget overlay. A navegação é da plataforma Repasse Seguro hospedeira.

| Componente | Tipo | Transição |
|---|---|---|
| Widget fechado → aberto | Modal/overlay | `slideUp` + `fadeIn` 250ms (D04 `normal`) |
| Widget aberto → fechado | Dismiss | `slideDown` + `fadeOut` 200ms |
| Chat → Modal de confirmação (T-06) | Modal sobre modal | `scale` 0.95→1.0 + `fadeIn` 250ms |
| Chat → Card de análise inline (T-05) | Expansão inline | `scaleY` 0→1, 400ms (D04 `analysisCardVariant`) |

**Comportamento do botão voltar (dispositivos mobile — Fase 1):**
- **Android (Chrome):** botão voltar do browser fecha o chat se aberto como overlay; volta para a página anterior se widget fechado
- **iOS (Safari):** swipe-back nativo fecha modais internos do chat se aberto; caso contrário navega no histórico do browser

### 2.2 Fase 2 — React Native

```
AppNavigator (Stack)
└── RepasseSeguroApp
    ├── TabNavigator
    │   ├── Dashboard
    │   ├── Oportunidades
    │   ├── Propostas
    │   └── Perfil
    └── DaniCedenteChat (Modal Stack — fullscreen)
        ├── ChatScreen          → T-01/T-02/T-03/T-04
        ├── AnalysisCardScreen  → T-05 (expandido em fullscreen)
        └── ConfirmationScreen  → T-06 (modal sobre modal)
```

| Tela | Tipo de navegação | Transição iOS | Transição Android | Deep Link |
|---|---|---|---|---|
| DaniCedenteChat (abertura) | Modal fullscreen | Slide from bottom | Slide from bottom | `repasse://chat` |
| AnalysisCardScreen | Push no modal stack | Slide from right | Slide from right | `repasse://chat/analysis/:type` |
| ConfirmationScreen | Apresentação modal | Scale + fade | Fade | Não (acionado por ação) |

**Botão voltar:**
- **iOS:** swipe-back (edge) fecha a tela anterior dentro do modal stack. Swipe-down fecha o modal fullscreen do chat.
- **Android:** botão hardware/software back fecha a tela atual. Se na raiz do modal stack, fecha o chat.

---

## 3. Gestos e Interações Nativas

### 3.1 Fase 1 — Web mobile

| Gesto | Contexto | Ação | Conflito | Alternativa acessível |
|---|---|---|---|---|
| Swipe up | Widget fechado (botão flutuante) | Abre o chat | Nenhum | Toque no botão Dani |
| Swipe down | Chat aberto (header) | Fecha o chat | Scroll da página se widget não estiver focado | Botão X no header |
| Scroll vertical | Histórico de mensagens | Navega no histórico | Nenhum (scroll isolado dentro do widget) | Teclas de seta (teclado externo) |
| Tap | Sugestões de resposta, CTAs, cards | Seleciona/ativa | Nenhum | Teclado (Enter em foco) |
| Long press | Mensagem | [DECISÃO AUTÔNOMA] Copiar mensagem — tap para selecionar texto nativo do browser. Alternativa: menu contextual custom. | Nenhum | Seleção de texto nativa |

### 3.2 Fase 2 — React Native

| Gesto | Contexto | Ação | Conflito nativo | Feedback | Alternativa acessível |
|---|---|---|---|---|---|
| Swipe down | Header do chat (PanResponder) | Fecha o modal | Scroll vertical do chat — resolvido: gesture zone apenas no header (primeiros 48dp) | Haptic `impactLight` (iOS) | Botão X no header |
| Scroll vertical | FlatList de mensagens | Navega no histórico | Nenhum (FlatList nativa com scroll isolado) | Visual (scrollbar) | VoiceOver/TalkBack scroll |
| Swipe-back (iOS) | Modal stack interno | Volta para tela anterior | Nenhum | — | Botão voltar |
| Tap | Sugestões, cards, modais | Ativa ação | Nenhum | Haptic `selectionChanged` (iOS) | AccessibilityAction |
| Long press | Mensagem da Dani | Copiar texto | Nenhum | Haptic `impactMedium` | AccessibilityAction "Copiar" |

**Feedback háptico (Fase 2 — iOS):**
- Envio de mensagem: `impactLight`
- Aceite de proposta (confirmação destrutiva): `notificationSuccess`
- Erro: `notificationError`
- Android: Vibração de 50ms via `Vibration.vibrate(50)`

---

## 4. Comportamento Offline

### 4.1 Contexto

O chat da Dani-Cedente **requer conexão ativa** para:
- Enviar mensagens (SSE streaming)
- Receber respostas do LLM
- Carregar dados de oportunidade/proposta em tempo real

### 4.2 Cache local

| Dado | TTL cache | Tamanho máximo | Tecnologia |
|---|---|---|---|
| Últimas 20 mensagens da sessão ativa | Até encerramento do app | 50KB por sessão | `AsyncStorage` (RN) / `localStorage` (web) |
| Dados da oportunidade atual | 5 minutos | 5KB | `AsyncStorage` / `sessionStorage` |
| Dados da proposta atual | 5 minutos | 5KB | `AsyncStorage` / `sessionStorage` |
| Token de autenticação (JWT) | Até expiração do token | < 1KB | Secure storage (`expo-secure-store` RN) / `httpOnly cookie` (web) |

### 4.3 Telas e comportamento offline

| Tela | Dados disponíveis offline | Ações possíveis | Ações bloqueadas |
|---|---|---|---|
| T-01 (chat inicial) | Nenhum | Ver mensagem de status offline | Iniciar conversa, carregar sugestões |
| T-02 (contexto oportunidade) | Oportunidade cacheada (5min TTL) | Visualizar dados cacheados | Atualizar status em tempo real |
| T-03 (contexto proposta) | Proposta cacheada (5min TTL) | Visualizar dados cacheados | Aceitar/recusar proposta |
| T-04 (conversa ativa) | Últimas 20 mensagens da sessão | Ler histórico | Enviar mensagem, receber resposta |
| T-05 (cards análise) | Dados do card se cacheado | Visualizar análise anterior | Atualizar com novos dados |
| T-06 (modais) | Nenhum | Nenhuma | Todas as ações de confirmação |
| T-07 (estados sistema) | — | Ver indicador offline | — |

### 4.4 Indicadores visuais de estado de rede

| Estado | Componente | Texto |
|---|---|---|
| Offline detectado | `Alert variant="warning"` no topo do chat | "Sem conexão — algumas funcionalidades estão indisponíveis." |
| Reconexão em andamento | `Skeleton` pulse no header | — |
| Reconectado | Toast 3s | "Conexão restabelecida." |
| Tentativa de envio offline | Campo de input desabilitado + `Alert` | "Sem conexão. Aguarde reconexão para enviar mensagens." |

### 4.5 Sincronização ao reconectar

- Sessão de chat: reconecta SSE automaticamente (sem recarregar a tela)
- Dados de oportunidade/proposta: revalida cache automaticamente
- Ações pendentes: nenhuma ação é enfileirada offline — o Cedente deve retentar após reconexão (confirmações destrutivas requerem ação consciente)

---

## 5. Push Notifications

### 5.1 Tipos de notificação

Os 10 eventos proativos do RF-DCE-028 geram push quando o Cedente está fora da plataforma:

| Evento | Tipo | Prioridade | Deep Link |
|---|---|---|---|
| Nova proposta recebida | Transacional | Alta | `repasse://chat?context=proposal&id={id}` |
| Proposta aceita (confirmação) | Transacional | Alta | `repasse://chat?context=escrow&id={id}` |
| Escrow: depósito confirmado | Transacional | Alta | `repasse://chat?context=escrow&id={id}` |
| Escrow: alerta de prazo (2 dias úteis) | Transacional urgente | Crítica | `repasse://chat?context=escrow&id={id}` |
| Escrow: extensão solicitada | Transacional | Alta | `repasse://chat?context=escrow&id={id}` |
| Escrow: expirado | Transacional urgente | Crítica | `repasse://chat?context=opportunity&id={id}` |
| Documento aprovado | Informativo | Normal | `repasse://chat?context=dossier` |
| Documento rejeitado | Transacional | Alta | `repasse://chat?context=dossier` |
| ZapSign: lembrete de assinatura | Transacional | Alta | `repasse://chat?context=signature` |
| Proposta recusada | Informativo | Normal | `repasse://chat?context=opportunity&id={id}` |

### 5.2 Permissões por plataforma

| Plataforma | Comportamento padrão | Quando solicitar | Fallback se negado |
|---|---|---|---|
| **iOS** | Bloqueado por padrão — requer permissão explícita | Primeira vez que Cedente ativa notificações nas configurações da plataforma | Mostrar banner in-app "Ative notificações para não perder propostas" com link para Configurações |
| **Android 12-** | Concedido por padrão | Não solicita — já disponível | — |
| **Android 13+** | Bloqueado por padrão — `POST_NOTIFICATIONS` permission | Mesma estratégia do iOS | Mostrar banner in-app |
| **Web Push** | Bloqueado por padrão — requer permissão explícita | Após Cedente aceitar primeira proposta ou após ação importante | Mostrar modal educativo antes de solicitar permissão nativa |

### 5.3 Comportamento por estado do app

| Estado | iOS | Android |
|---|---|---|
| **Foreground** | Exibe Toast in-app (`ProactiveToast`). Sem push nativo. | Exibe Toast in-app. Sem push nativo. |
| **Background** | Push nativo exibido no Centro de Notificações. Toque → deep link. | Push nativo exibido na barra de notificações. Toque → deep link. |
| **App killed** | Push nativo exibido. Toque → abre app no deep link. | Push nativo exibido. Toque → abre app no deep link. |

### 5.4 Agrupamento e prioridade

- Notificações do mesmo tipo agrupadas (ex: 3 documentos aprovados → "3 documentos aprovados")
- Notificações urgentes (Escrow crítico) nunca agrupadas — exibidas individualmente
- TTL de push: 24h para informativas, 48h para urgentes (a partir do evento)

---

## 6. Deep Links e Universal Links

### 6.1 Schema

- **Fase 2 (React Native):** `repasse://`
- **Fase 1 (Web):** URL da plataforma + query param `?dani_context=` (sem deep link nativo)

### 6.2 Mapeamento completo

| Rota | Tela aberta | Parâmetros | Requer auth |
|---|---|---|---|
| `repasse://chat` | T-01 (chat inicial) | — | Sim |
| `repasse://chat?context=opportunity&id={id}` | T-02 (contexto oportunidade) | `id`: opportunity UUID | Sim |
| `repasse://chat?context=proposal&id={id}` | T-03 (contexto proposta) | `id`: proposal UUID | Sim |
| `repasse://chat?context=escrow&id={id}` | T-04 + card Escrow (T-05) | `id`: escrow_transaction UUID | Sim |
| `repasse://chat?context=dossier` | T-04 + card Dossiê (T-05) | — | Sim |
| `repasse://chat?context=signature` | T-04 (orientação ZapSign) | — | Sim |

### 6.3 Fallback

- Deep link com `id` inválido ou de outro Cedente → redireciona para T-01 (chat inicial) com mensagem "Não encontrei o conteúdo que você procurava."
- Deep link com Cedente não autenticado → redireciona para tela de login da plataforma; após login, retoma o deep link original (link intent preservation)
- Deep link no browser sem app instalado (Fase 2) → abre URL da plataforma web correspondente

---

## 7. Permissões Nativas

| Permissão | Feature que usa | Momento da solicitação | Mensagem de contexto | Fallback se negada |
|---|---|---|---|---|
| **Push Notifications** | Notificações proativas (10 eventos) | Após Cedente receber primeira proposta | "Ative notificações para ser avisado assim que uma proposta chegar." | Notificações in-app via `ProactiveToast` apenas quando o app está aberto |
| **Câmera** | Upload de documentos do dossiê (foto do documento) | Ao tocar em "Tirar foto" no upload de dossiê | "Precisamos de acesso à câmera para fotografar seus documentos." | Usar galeria como alternativa |
| **Galeria/Fotos** | Upload de documentos do dossiê | Ao tocar em "Escolher da galeria" | "Precisamos de acesso à galeria para selecionar seus documentos." | Usar câmera como alternativa; se ambas negadas: upload de arquivo via seletor de arquivos do sistema |

**Comportamento ao revogar permissão nas configurações do sistema:**
- Push: `ProactiveToast` continua funcionando quando app está em foreground; banner in-app orienta re-habilitar nas configurações
- Câmera/Galeria: botões de upload redirecionam para seletor de arquivos nativo (sem câmera/galeria)

---

## 8. Diferenças iOS vs. Android

| Feature | iOS | Android |
|---|---|---|
| Swipe para fechar chat | Swipe-down no header (Fase 2) | Botão hardware/software back |
| Teclado virtual | Empurra o chat para cima (`KeyboardAvoidingView` behavior: "padding") | Redimensiona a view (`windowSoftInputMode: adjustResize`) |
| Status bar | Transparente + light/dark conforme tema | Colorida com cor primária `#0069A8` |
| Push permission | Solicita explicitamente (NSUserNotificationPermission) | Android 12-: automático; Android 13+: `POST_NOTIFICATIONS` explícito |
| Notificação em foreground | Sem banner nativo — usar Toast in-app | Pode exibir banner nativo em foreground (comportamento configurável) |
| Haptic feedback | `ReactNativeHapticFeedback` com padrões iOS (impactLight, notificationSuccess) | `Vibration.vibrate()` com duração fixa |
| Secure storage | `expo-secure-store` (Keychain) | `expo-secure-store` (Keystore) |
| `100dvh` (Fase 1 web) | Requer polyfill em iOS < 15.4 | Chrome 90+: suportado nativamente |
| Safe area | `SafeAreaView` necessário (notch, Dynamic Island) | `SafeAreaView` necessário para devices com notch |
| Animações | Reanimated 3.x+ com worklets na UI thread | Mesmo — comportamento idêntico |

---

## 9. Performance Mobile

### 9.1 Métricas-alvo

| Métrica | Alvo | Medição |
|---|---|---|
| Tempo de abertura do chat (widget web) | ≤ 500ms até primeiro conteúdo visível | Lighthouse Mobile |
| Tempo de abertura do chat (RN nativo, Fase 2) | ≤ 300ms até primeira frame | React DevTools Profiler |
| Time to Interactive (TTI) do chat | ≤ 1.5s | Lighthouse Mobile |
| FPS durante scroll do histórico | 60 FPS (sem jank) | Flipper / Android Profiler |
| Tamanho do bundle do widget (web, gzipped) | ≤ 150KB | Webpack Bundle Analyzer |
| Latência de primeira token (SSE) | ≤ 5s p95 | Langfuse traces |

### 9.2 Otimizações obrigatórias

- **FlatList virtualizada (Fase 2):** `FlashList` (Shopify) ou `FlatList` com `windowSize: 5`, `removeClippedSubviews: true` para histórico longo
- **Lazy loading de avatares:** Avatar da Dani carregado uma vez e cacheado em memória
- **Streaming SSE:** tokens exibidos progressivamente — não espera resposta completa para renderizar
- **`prefers-reduced-motion`:** via hook `useReducedMotion()` (Framer Motion web) e `AccessibilityInfo.isReduceMotionEnabled()` (RN)

---

## 10. Acessibilidade Mobile

### 10.1 VoiceOver (iOS) e TalkBack (Android)

| Elemento | `accessibilityLabel` | `accessibilityRole` | `accessibilityHint` |
|---|---|---|---|
| Botão abrir chat | "Abrir chat com a Dani" | `button` | "Duplo toque para abrir o chat da Dani" |
| Bolha de mensagem (Dani) | "Dani diz: [conteúdo]" | `text` | — |
| Bolha de mensagem (Cedente) | "Você disse: [conteúdo]" | `text` | — |
| Campo de entrada | "Mensagem para a Dani" | `none` (`textInput` nativo) | "Digite sua mensagem" |
| Botão enviar | "Enviar mensagem" | `button` | — |
| Typing indicator | "A Dani está digitando" | `text` | — |
| Estrela CSAT (N) | "Avaliar com [N] estrela(s)" | `radio` | "Duplo toque para selecionar" |
| CTA primário modal | "[Texto do botão]" | `button` | "Duplo toque para confirmar" |

### 10.2 Tamanhos mínimos de toque

| Plataforma | Mínimo | Aplicação |
|---|---|---|
| iOS | 44×44pt | Botões, sugestões, estrelas CSAT |
| Android | 48×48dp | Idem |
| Web mobile | 48×48px | Idem (seguindo WCAG 2.1 AA target size) |

### 10.3 Fontes dinâmicas

- **iOS:** respeita `Dynamic Type` — textos escalam com configuração do sistema
- **Android:** respeita escala de fonte do sistema (`fontScale`)
- Layout do chat adaptável: usa `flexWrap` e `maxWidth` relativo para acomodar textos maiores

### 10.4 Contraste

- Todos os textos: contraste mínimo 4.5:1 (texto normal) e 3:1 (texto grande/bold)
- Bolha assistant (`--rs-primary/10` background): validado para `--rs-foreground` (texto) em ambos light e dark mode
- StatusBadge: todas as combinações de cor validadas para 3:1 (texto bold em badge)

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. Distinção clara Fase 1 (web responsiva/PWA) vs. Fase 2 (React Native nativo). Navegação, gestos, offline, push (10 eventos), deep links, permissões (câmera, galeria, push), diferenças iOS/Android, performance, acessibilidade VoiceOver/TalkBack. |

---

## 12. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Política de agrupamento de notificações push (Android) | DECISÃO AUTÔNOMA | Seção 5.4 | Adotado agrupamento por tipo com exceção para notificações urgentes de Escrow. Alternativa descartada: sem agrupamento (spam na barra de notificações). | P2 | Mobile | Validar |
| Timing da solicitação de permissão de push (iOS/Android 13+) | DECISÃO AUTÔNOMA | Seção 5.2 | Adotado: após primeira proposta recebida (momento de alto engajamento). Alternativa descartada: onboarding (usuário ainda sem proposta, pouca motivação). | P1 | Produto | Validar |
| `FlashList` vs `FlatList` para histórico de mensagens (Fase 2) | DECISÃO AUTÔNOMA | Seção 9.2 | Adotado `FlashList` (melhor performance com listas longas). Alternativa: `FlatList` com otimizações manuais (mais controle, mais código). | P2 | Mobile | Validar na implementação |
| Estratégia de upload de documentos offline (Fase 2) | DEFINIÇÃO PENDENTE | Seção 4.5 | (A) Enfileirar upload offline, sincronizar ao reconectar; (B) Bloquear upload sem conexão e exibir mensagem clara. Trade-off: (A) melhor UX mas complexidade de fila local; (B) mais simples mas bloqueia ação importante. | P1 | Produto / Mobile | Decidir antes de implementar Fase 2 |
