# Repasse AI
## 11 — Mobile

| Campo | Valor |
|---|---|
| **Destinatário** | Produto, Mobile e Engenharia |
| **Escopo** | Especificação mobile com navegação, gestos, offline, push e diferenças de plataforma |
| **Módulo** | Repasse AI |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Repasse AI é um serviço backend puro (PG-03).** Não existe app React Native, Expo ou qualquer cliente mobile nativo próprio. A experiência mobile é 100% via browser (Web) na aplicação Cessionário.
> - **Plataformas suportadas:** iOS 15+ e Android 10+ via browser mobile (Safari, Chrome). Sem app store.
> - **Padrão de navegação adotado:** Webchat em modal full-screen (< 768px). FAB flutuante sempre visível. Navegação interna por mensagens dentro do chat.
> - **Telas com comportamento offline definido:** 4 telas críticas (T-001 FAB, T-003 Chat Ativo, T-009 Bolha de Erro, T-012/T-013/T-014 Alertas Proativos).
> - **Features nativas utilizadas:** Push Notifications via PWA (Web Push API), deep links via URLs universais, sem câmera/galeria/biometria nativa.
> - **Diferenças críticas iOS vs. Android:** comportamento de Push Web, safe area do notch/Dynamic Island, comportamento do teclado virtual ao abrir chat.
> - **Seções pendentes:** nenhuma. 3 decisões autônomas aplicadas (marcadas inline).

---

## 1. Plataformas e Versões Suportadas

### 1.1 Escopo Arquitetural

> ⚙️ **PG-03 — Repasse AI é um serviço backend sem frontend próprio.** Conforme `02 - Stacks.md` (PG-03): *"O módulo expõe API REST e SSE. Não possui frontend web, mobile, landing page ou dashboard. Seções de frontend e mobile do ShiftLabs Stacks v7.0 não se aplicam a este módulo."*

A experiência mobile do Repasse AI é entregue como **Progressive Web App (PWA)** embutida no frontend da plataforma Cessionário. Não existe aplicativo nativo publicado em App Store ou Google Play.

**Implicações práticas:**
- Nenhum código React Native, Expo ou equivalente é necessário para o Repasse AI.
- O time mobile do Cessionário é responsável pela renderização do webchat no contexto mobile.
- Os contratos de UI (D09) e os wireframes (D07) são a fonte de verdade para o que renderiza em mobile.
- Esta especificação define os **comportamentos nativos esperados** dentro do contexto do browser mobile — gestos, safe areas, push, deep links e offline.

### 1.2 Browsers e Sistemas Operacionais Suportados

| Plataforma | Versão Mínima do SO | Browser Suportado | Versão Mínima do Browser |
|---|---|---|---|
| iOS | 15.0+ | Safari (WKWebView) | Safari 15+ |
| iOS | 15.0+ | Chrome para iOS | Chrome 100+ |
| Android | 10 (API 29)+ | Chrome para Android | Chrome 100+ |
| Android | 10 (API 29)+ | Samsung Internet | 15.0+ |
| Android | 10 (API 29)+ | Firefox para Android | 110+ |

> 💡 **Nota:** iOS 15+ cobre >90% dos dispositivos iOS ativos (2026). Android 10+ cobre >85% do parque Android BR. Safari é o único browser com renderização nativa em iOS (todos os outros browsers usam WKWebView sob o capô).

### 1.3 Dispositivos de Referência para Testes

| Categoria | Dispositivo | SO | Prioridade |
|---|---|---|---|
| iPhone flagship | iPhone 15 Pro | iOS 17 | P1 |
| iPhone mid-range | iPhone 13 | iOS 16 | P1 |
| iPhone entry | iPhone SE (3ª geração) | iOS 16 | P2 |
| Android flagship | Samsung Galaxy S24 | Android 14 | P1 |
| Android mid-range | Motorola Moto G84 | Android 13 | P1 |
| Android entry | Samsung Galaxy A15 | Android 14 | P2 |
| Tablet (referência) | iPad Air (5ª geração) | iPadOS 16 | P2 |

### 1.4 Resolução e Density Buckets

| Bucket | Resolução Lógica (px) | DPR | Exemplo de Dispositivo |
|---|---|---|---|
| Small (mobile portrait) | 375 × 812 | 3x | iPhone 12/13/14 |
| Small (mobile portrait) | 390 × 844 | 3x | iPhone 14 Pro |
| Small (mobile landscape) | 812 × 375 | 3x | iPhone 12 landscape |
| Medium (tablet portrait) | 768 × 1024 | 2x | iPad mini |
| Medium (tablet landscape) | 1024 × 768 | 2x | iPad Air |

**Breakpoints críticos para o Repasse AI:**
- `< 768px`: webchat abre em modal full-screen (ocupa 100vw × 100dvh).
- `768px–1024px`: webchat abre em painel lateral fixo (40% da largura).
- `> 1024px`: comportamento desktop padrão.

---

## 2. Arquitetura de Navegação

### 2.1 Padrão Adotado

O Repasse AI utiliza **navegação por URL + modal overlay** — não existe um stack navigator nativo React Navigation. A navegação é gerenciada pelo router da plataforma Cessionário (Next.js App Router).

```
Plataforma Cessionário (Next.js)
└── Qualquer rota da plataforma
    ├── FAB flutuante (T-001) — sempre presente no DOM
    ├── Modal Chat (T-003) — abre sobre o conteúdo atual sem mudar URL
    │   ├── ChatBubbles (T-004 a T-014) — renderizadas dentro do modal
    │   └── Banner LGPD (T-002) — overlay dentro do modal na 1ª abertura
    └── Meu Perfil (/perfil)
        ├── Seção WhatsApp (T-015)
        │   ├── Modal Vinculação WhatsApp (T-016)
        │   └── Modal Confirmação Desvinculação (T-017)
        ├── Seção Notificações (T-018)
        └── Seção Histórico de Chat (T-019)
```

### 2.2 Hierarquia de Navegadores

| Nível | Tipo | Tecnologia | Telas |
|---|---|---|---|
| L1 — Router da Plataforma | URL-based (Next.js) | `useRouter` | Todas as rotas da plataforma |
| L2 — Modal Chat | Overlay full-screen mobile | `<Dialog>` / `position: fixed` | T-002, T-003, T-004 a T-014 |
| L3 — Modais de Configuração | Overlay sobre Perfil | `<Sheet>` (shadcn/ui) | T-016, T-017 |
| L3 — Seções de Perfil | Tabs dentro de /perfil | `<Tabs>` (shadcn/ui) | T-015, T-018, T-019 |

### 2.3 Mapeamento Completo: Tela → Tipo de Navegação → Transição

| ID | Tela | Tipo | Transição Entrada | Transição Saída | Deep Link | Estado Offline |
|---|---|---|---|---|---|---|
| T-001 | FAB do Chat | Fixed overlay (sempre visível) | Nenhuma (persiste) | Nenhuma | Não | Sim (badge offline) |
| T-002 | Banner Consentimento LGPD | Modal sheet (bottom) | Slide-up 300ms | Fade-out 150ms | Não | Não |
| T-003 | Janela do Chat — Ativo | Modal full-screen (mobile) / Painel lateral (tablet+) | Slide-up 300ms (iOS) / Slide-right 250ms (Android) | Slide-down 300ms (iOS) / Slide-left 250ms (Android) | Sim (`/chat`) | Sim (modo degradado) |
| T-004 | Bolha Análise | Renderização interna no chat | Fade-in 200ms + streaming | — | Não | Sim (cached) |
| T-005 | Bolha Comparação | Renderização interna no chat | Fade-in 200ms + streaming | — | Não | Sim (cached) |
| T-006 | Bolha Simulação | Renderização interna no chat | Fade-in 200ms + streaming | — | Não | Sim (cached) |
| T-007 | Bolha Portfólio | Renderização interna no chat | Fade-in 200ms + streaming | — | Não | Sim (cached) |
| T-008 | Bolha Suporte | Renderização interna no chat | Fade-in 200ms + streaming | — | Não | Sim (cached) |
| T-009 | Bolha Erro/Degradação | Renderização interna no chat | Fade-in 150ms | — | Não | Sim (exibida quando offline) |
| T-010 | Bolha Dado Bloqueado | Renderização interna no chat | Fade-in 150ms | — | Não | Sim (cached) |
| T-011 | Bolha Rate Limit | Renderização interna no chat | Fade-in 150ms | — | Não | Não (requer conexão) |
| T-012 | Alerta Proativo — Oportunidade | Renderização interna no chat | Fade-in 200ms | — | Sim (`/chat?alert=opportunity`) | Não |
| T-013 | Alerta Proativo — Escrow | Renderização interna no chat | Fade-in 200ms | — | Sim (`/chat?alert=escrow`) | Não |
| T-014 | Alerta Proativo — Status | Renderização interna no chat | Fade-in 200ms | — | Sim (`/chat?alert=status`) | Não |
| T-015 | Perfil — Seção WhatsApp | Tab dentro de /perfil | Slide horizontal tab | Slide horizontal tab | Sim (`/perfil?tab=whatsapp`) | Não |
| T-016 | Modal Vinculação WhatsApp | Modal sheet (bottom) | Slide-up 300ms | Slide-down 200ms | Não | Não |
| T-017 | Modal Confirmação Desvinculação | Modal alert dialog | Fade-in 150ms | Fade-out 100ms | Não | Não |
| T-018 | Perfil — Seção Notificações | Tab dentro de /perfil | Slide horizontal tab | Slide horizontal tab | Sim (`/perfil?tab=notificacoes`) | Não |
| T-019 | Perfil — Histórico de Chat | Tab dentro de /perfil | Slide horizontal tab | Slide horizontal tab | Sim (`/perfil?tab=historico`) | Não |
| T-020 | Supervisão IA — Listagem | Rota Admin `/admin/supervisao` | Page transition padrão | Page transition padrão | Sim (`/admin/supervisao`) | Não |
| T-021 | Supervisão — Detalhe + Takeover | Rota Admin `/admin/supervisao/[id]` | Page transition padrão | Voltar à listagem | Sim (`/admin/supervisao/:id`) | Não |
| T-022 | Dashboard Admin | Rota Admin `/admin/dashboard` | Page transition padrão | Page transition padrão | Sim (`/admin/dashboard`) | Não |

> ⚙️ **Admin (T-020 a T-022) não é otimizado para mobile.** [DECISÃO AUTÔNOMA] — O painel Admin não tem layout mobile-first. Opera em landscape em tablet (768px+) com sidebar recolhível. Acesso em phones é possível mas não é o caso de uso principal. Justificativa: Admin é operado por equipe interna em desktop ou tablet. Alternativa descartada: admin mobile-first aumentaria escopo sem benefício real para o perfil de uso documentado.

### 2.4 Comportamento do Botão Voltar e Swipe-Back

| Ação | iOS (Swipe-back) | Android (Botão voltar) |
|---|---|---|
| Chat aberto (T-003) → Fechar chat | Swipe-down fecha o modal | Botão voltar fecha o modal |
| Dentro do chat com histórico | Swipe da borda esquerda NÃO navega dentro do chat (conflito prevenido) | Botão voltar fecha o modal (não navega dentro do chat) |
| Modal LGPD (T-002) aberto | Swipe-down fecha o modal SEM aceitar consentimento | Botão voltar fecha o modal SEM aceitar consentimento |
| Modal WhatsApp (T-016) Step > 1 | Swipe-down retorna ao step anterior (não fecha) | Botão voltar retorna ao step anterior (não fecha) |
| Modal WhatsApp (T-016) Step 1 | Swipe-down fecha o modal | Botão voltar fecha o modal |
| Modal Confirmação Desvinculação (T-017) | Swipe-down cancela (equivale a "Cancelar") | Botão voltar cancela |
| Admin (T-020/T-021/T-022) | Swipe-back navega para rota anterior no histórico | Botão voltar navega para rota anterior |

> ⚙️ **Prevenção de swipe-back no chat:** O modal de chat em iOS usa `overscroll-behavior: contain` no container do histórico de mensagens. O swipe-back da borda é desabilitado via `gestureEnabled: false` no contexto do overlay para evitar fechamento acidental durante scroll de mensagens.

---

## 3. Gestos e Interações Nativas

### 3.1 Tabela Completa de Gestos

| Gesto | Contexto (Tela) | Ação Resultante | Conflito com Nativo | Prevenção | Feedback Háptico | Alternativa Acessível |
|---|---|---|---|---|---|---|
| **Swipe-down** | T-003 Chat (área header) | Fecha o modal do chat | Nenhum | — | Leve (impact light) ao fechar | Botão "×" no header do chat |
| **Swipe-down** | T-002 Banner LGPD | Fecha o banner sem aceitar | Nenhum | — | Nenhum | Botão "Fechar" no banner |
| **Swipe-down** | T-016 Modal WhatsApp Step 1 | Fecha o modal | Nenhum | — | Leve (impact light) | Botão "Cancelar" no modal |
| **Swipe-up (scroll)** | T-003 Histórico de mensagens | Scroll para cima no histórico | iOS: activa rubber band; Android: nenhum | `overscroll-behavior: contain` | Nenhum | ScrollBar visível; botão "Ir ao início" |
| **Pull-to-refresh** | T-003 Chat (histórico) | Recarrega últimas mensagens da API | iOS: conflito com rubber band — **não implementar** [DECISÃO AUTÔNOMA] | Pull-to-refresh desabilitado no iOS | N/A | Botão "Atualizar" no header do chat |
| **Pull-to-refresh** | T-020 Listagem Admin | Recarrega lista de interações | Android: nenhum; iOS: não implementar | `overscroll-behavior: none` no iOS | Leve ao completar refresh | Botão "Atualizar" na toolbar |
| **Long-press** | ChatBubble (qualquer) | Exibe menu de contexto: Copiar texto, Reportar resposta | iOS: conflito com selection nativa — usar `onLongPress` com `preventDefault` | `user-select: none` no container | Medium (impact medium) | Botão "..." visível na bolha ao tocar |
| **Double-tap** | ChatBubble (qualquer) | Nenhuma ação [DECISÃO AUTÔNOMA] | iOS: zoom de acessibilidade | `touch-action: pan-y` no container | N/A | N/A |
| **Pinch-to-zoom** | Imagem/gráfico em bolha (se presente) | Zoom na imagem | iOS: zoom nativo do browser | `touch-action: manipulation` no overlay de imagem | Nenhum | Botão "Expandir" na imagem |
| **Tap no FAB (T-001)** | Qualquer tela da plataforma | Abre/fecha o chat | Nenhum | — | Leve (impact light) | Acessível via tab + Enter |
| **Tap no input** | T-003 Campo de texto | Abre teclado virtual | iOS: teclado empurra viewport; Android: resize do viewport | `position: fixed` no chat container + `height: 100dvh` | Nenhum | — |

> 💡 **Pull-to-refresh no chat desabilitado no iOS:** [DECISÃO AUTÔNOMA] — O rubber band de overscroll do iOS interfere com o scroll do histórico de mensagens. Solução: desabilitar pull-to-refresh no iOS via detecção de `navigator.platform`. No Android, pull-to-refresh é habilitado normalmente. Alternativa descartada: implementar para ambas as plataformas com tratamento especial no iOS aumenta complexidade sem ganho de UX no contexto do chat.

### 3.2 Feedback Háptico por Ação

| Ação | Intensidade | API Web | Fallback se não suportado |
|---|---|---|---|
| Fechar chat (swipe-down) | Impact light | `navigator.vibrate(10)` | Nenhum (apenas visual) |
| Enviar mensagem | Impact light | `navigator.vibrate(10)` | Nenhum |
| Long-press em bolha | Impact medium | `navigator.vibrate([50, 30, 50])` | Nenhum |
| Aceitar consentimento LGPD | Impact medium | `navigator.vibrate(20)` | Nenhum |
| Erro de rede (T-009) | Notification warning | `navigator.vibrate([100, 50, 100])` | Nenhum |
| Rate limit (T-011) | Notification error | `navigator.vibrate([200, 100, 200])` | Nenhum |
| OTP enviado (T-016 Step 2) | Notification success | `navigator.vibrate(30)` | Nenhum |

> ⚙️ **Web Vibration API:** Suportada em Chrome Android. Safari iOS não suporta `navigator.vibrate()` — todos os feedbacks hápticos são degradação graciosa (nunca bloqueiam a ação).

### 3.3 Comportamento do Teclado Virtual

| Evento | iOS (Safari) | Android (Chrome) |
|---|---|---|
| Input do chat recebe foco | Teclado sobe; `visualViewport` reduz; chat container mantém `position: fixed` e reajusta com `window.visualViewport.addEventListener('resize')` | Viewport redimensiona (`resize` event); chat container reajusta via `height: 100dvh` |
| Teclado fecha | `visualViewport` restaura; chat container retorna ao tamanho original | Viewport restaura; chat container retorna |
| Botões de ação (Enviar, CTAs) | Ficam acima do teclado via `padding-bottom: env(keyboard-inset-height, 0)` | Ficam acima do teclado via `windowSoftInputMode: adjustResize` |

---

## 4. Comportamento Offline

### 4.1 Estratégia de Cache Local

| Tecnologia | Uso | TTL | Limite de Armazenamento |
|---|---|---|---|
| **Service Worker Cache API** | Assets estáticos (JS, CSS, fontes, ícones do PWA) | Imutável (hash de build) | Determinado pelo browser |
| **localStorage** | Histórico das últimas 50 mensagens do chat | 7 dias | ~5 MB (padrão browser) |
| **sessionStorage** | Estado da sessão de chat ativa (scroll position, input draft) | Sessão do browser | ~5 MB |
| **IndexedDB** | Histórico de conversas completo (para T-019 Histórico) | 30 dias | ~50 MB (configurável) |
| **Memory (React state)** | Mensagens da conversa ativa em streaming | Duração da sessão | Sem limite fixo |

### 4.2 Telas com Suporte Offline

| Tela | Dados Disponíveis em Cache | Ações Possíveis Offline | Ações Bloqueadas Offline | Indicador Visual |
|---|---|---|---|---|
| T-001 FAB | Badge de alertas (último estado conhecido) | Abrir chat (modo degradado) | — | Badge exibe último valor; sem atualização |
| T-003 Chat — Ativo | Últimas 50 mensagens (localStorage) | Ler histórico, redigir mensagem | Enviar mensagem, receber resposta do agente | Banner "Sem conexão — mensagens serão enviadas ao reconectar" |
| T-004 a T-008 ChatBubbles de análise/simulação | Bolhas da sessão atual (memory) | Ler bolhas renderizadas | Solicitar nova análise | Banner no topo do chat |
| T-009 Bolha Erro/Degradação | N/A | — | — | Exibida automaticamente quando offline |
| T-010 Bolha Dado Bloqueado | Bolha da sessão atual (memory) | Ler mensagem | Solicitar novo dado | — |
| T-012/T-013/T-014 Alertas Proativos | Alertas recebidos antes de ficar offline (memory) | Ler alertas | Receber novos alertas | — |
| T-015 Perfil — WhatsApp | Status da vinculação (localStorage) | Visualizar status | Iniciar/cancelar vinculação | Seção desabilitada com badge "Sem conexão" |
| T-018 Perfil — Notificações | Configurações atuais (localStorage) | Visualizar toggles | Alterar preferências | Toggle desabilitado + badge "Sem conexão" |
| T-019 Perfil — Histórico | Últimas 30 conversas (IndexedDB) | Ler histórico em cache | Apagar histórico | Badge "Dados podem estar desatualizados" |
| T-020/T-021/T-022 Admin | Nenhum | Nenhuma | Todas | Tela de erro com botão "Tentar novamente" |

### 4.3 Fila de Ações Offline

Quando o usuário tenta enviar uma mensagem no chat estando offline:

1. A mensagem é armazenada na fila em localStorage com timestamp e estado `queued`.
2. O input exibe a mensagem com badge "Aguardando conexão" e ícone de relógio.
3. Ao reconectar, a fila é drenada em ordem FIFO automaticamente.
4. Cada mensagem na fila passa por validação de rate limit antes do envio.
5. Se o rate limit for atingido ao reconectar, a mensagem é descartada e o usuário é notificado inline.

**Limite da fila:** máximo de 5 mensagens enfileiradas. A partir da 6ª tentativa, exibe: *"Sua conexão está instável. Aguarde reconectar para continuar."*

### 4.4 Reconexão e Sincronização

| Evento | Ação do App |
|---|---|
| `navigator.onLine` passa de `false` para `true` | Drena fila de mensagens pendentes; atualiza badge do FAB; remove banner de offline |
| SSE (streaming) cai durante resposta do agente | Mensagem exibe estado `error` com botão "Tentar novamente"; não exibe resposta parcial |
| SSE reconecta dentro de 30s | Retoma stream da última mensagem processada (via `Last-Event-ID`) |
| SSE não reconecta após 30s | Exibe T-009 (Bolha Erro/Degradação) com botão "Tentar novamente" |

### 4.5 Resolução de Conflitos

O chat é append-only (sem edição de mensagens enviadas). Conflitos de sincronização são impossíveis no fluxo principal. Único caso de conflito: preferências de notificação (T-018) alteradas offline.

**Regra de conflito para T-018:** se a preferência for alterada offline e o servidor tiver um valor diferente no momento da sincronização, **o valor do servidor vence** (last-write-wins server-side). O usuário vê um toast: *"Suas preferências foram atualizadas pelo servidor."*

### 4.6 Indicadores Visuais de Estado de Rede

| Estado | Componente | Texto | Posição |
|---|---|---|---|
| Offline | `OfflineBanner` (sticky top no chat) | "Sem conexão — mensagens não podem ser enviadas" | Topo do modal do chat |
| Offline | `StatusDot` no FAB | Ponto vermelho substituindo badge | FAB flutuante |
| Reconectando | `OfflineBanner` | "Reconectando…" com animação de pulso | Topo do modal do chat |
| Online (restaurado) | `OfflineBanner` | "Conexão restaurada" por 2s, depois some | Topo do modal do chat |
| Mensagem na fila | Badge na mensagem | Ícone relógio + "Aguardando conexão" | Ao lado da mensagem |

---

## 5. Push Notifications

### 5.1 Contexto Arquitetural

O Repasse AI entrega push notifications via **dois canais distintos**:

1. **Web Push (PWA):** para Cessionários com o app web instalado como PWA ou com browser aberto. Usa Web Push API + Service Worker.
2. **WhatsApp:** mensagens ativas pelo canal WhatsApp (Fase 2). Gerenciado pelo módulo WhatsApp do backend via EvolutionAPI. Não passa pelo sistema de push do browser.

Esta seção cobre **Web Push**. WhatsApp é coberto na Seção 8 (Diferenças iOS vs. Android) e na documentação de integrações externas (D17).

### 5.2 Tipos de Notificação

| Tipo | Trigger | Canal | Prioridade | Agrupamento |
|---|---|---|---|---|
| Nova Oportunidade Compatível | Evento `opportunity.match` no backend | Web Push + WhatsApp | Alta | Por tópico `opportunity` |
| Prazo de Escrow (2 dias úteis) | Job agendado no backend | Web Push + WhatsApp | Crítica | Por `negociacao_id` |
| Mudança de Status de Proposta | Evento `proposal.status_changed` | Web Push + WhatsApp | Alta | Por `proposta_id` |
| Sistema (manutenção, degradação) | Manual via Admin | Web Push | Baixa | Por tópico `system` |

### 5.3 Permissões por Plataforma

| Plataforma | Comportamento Padrão | Quando Solicitar | Fluxo de Concessão |
|---|---|---|---|
| iOS 16.4+ (PWA instalada) | Bloqueado por padrão; requer permissão explícita | Na primeira visita após instalação do PWA, ao abrir T-018 (Notificações) | `Notification.requestPermission()` via prompt do sistema |
| iOS < 16.4 | Web Push **não suportado** | Não solicitar — informar limitação | Banner informativo: "Instale o app e atualize o iOS para receber alertas" |
| Android 13+ (Chrome) | Bloqueado por padrão; requer permissão | Na primeira visita, ao abrir T-018 (Notificações) | `Notification.requestPermission()` via prompt do sistema |
| Android < 13 (Chrome) | Concedido automaticamente (legacy behavior) | Solicitar registro do Service Worker apenas | Nenhum prompt necessário |

**Momento de solicitação de permissão:**
- Nunca solicitar na abertura do app sem contexto.
- Solicitar apenas quando o usuário navega para T-018 (Seção Notificações) **e** clica em "Ativar alertas".
- Exibir antes do prompt do sistema um modal contextual explicando o valor: *"Ative alertas para ser notificado sobre oportunidades compatíveis, prazos de Escrow e mudanças em suas propostas."*

**Tratamento de recusa:**
- Se o usuário recusar o prompt do sistema: exibir banner informativo em T-018 com botão "Ativar nas configurações" (deeplink para configurações do browser/sistema).
- Não reapresentar o prompt automático. Aguardar ação explícita do usuário.
- Funcionalidade do app não é bloqueada pela recusa. Notificações ficam disponíveis via webchat quando o app está aberto.

### 5.4 Comportamento em Foreground, Background e App Killed

| Estado do App | iOS (Safari/PWA) | Android (Chrome) |
|---|---|---|
| **Foreground** (chat aberto) | Notificação suprimida; alerta exibido inline como ChatBubble (T-012/T-013/T-014) | Notificação suprimida; alerta exibido inline como ChatBubble |
| **Foreground** (app aberto, chat fechado) | Notificação exibida como Web Notification nativa no browser | Notificação exibida como Web Notification nativa |
| **Background** (app em background) | iOS < 16.4: Web Push não suportado; iOS 16.4+: notificação exibida na Central de Notificações | Notificação entregue pelo Service Worker; exibida na Central de Notificações |
| **App killed / browser fechado** | iOS < 16.4: não entregue; iOS 16.4+ (PWA): entregue via APNs | Entregue via FCM; exibida na Central de Notificações |

### 5.5 Deep Link Contextual por Tipo de Push

| Tipo de Push | Deep Link | Comportamento ao Tocar |
|---|---|---|
| Nova Oportunidade | `/chat?alert=opportunity&opportunity_id={id}` | Abre o chat; exibe T-012 (Bolha Alerta Oportunidade) com o card da oportunidade específica |
| Prazo de Escrow | `/chat?alert=escrow&negociacao_id={id}` | Abre o chat; exibe T-013 (Bolha Alerta Escrow) com dados da negociação |
| Mudança de Status | `/chat?alert=status&proposta_id={id}` | Abre o chat; exibe T-014 (Bolha Alerta Status) com o novo status |
| Sistema | `/chat?alert=system` | Abre o chat; exibe mensagem de sistema (variante `system` do ChatBubble) |

**Regra de autenticação no deep link:** se o usuário não está autenticado, o deep link é preservado em sessionStorage. Após autenticação, o router redireciona para a URL preservada.

### 5.6 Opt-out e Preferências

- Cada tipo de notificação tem toggle individual em T-018 (Seção Notificações).
- O opt-out por tipo envia `PATCH /notifications/preferences` no backend.
- Opt-out de **todos** os tipos não revoga a permissão do browser — apenas para o envio pelo backend.
- Para revogar a permissão do browser, o usuário deve ir às configurações do sistema operacional.
- Badge no FAB (T-001) reflete apenas alertas dos tipos **ativos** nas preferências.

---

## 6. Deep Links e Universal Links

### 6.1 Schema de URLs

O Repasse AI utiliza **URLs universais (HTTPS)** — não existe schema customizado (`repasse://`). Não há app nativo para registrar schemas.

Base URL: `https://app.repasseseguro.com.br`

### 6.2 Mapeamento Completo: Rota → Tela → Parâmetros

| Rota | Tela | Parâmetros Aceitos | Autenticação |
|---|---|---|---|
| `/chat` | T-003 Chat Ativo | `alert={opportunity\|escrow\|status\|system}`, `opportunity_id`, `negociacao_id`, `proposta_id` | Obrigatória |
| `/chat?alert=opportunity&opportunity_id={id}` | T-003 + T-012 | `opportunity_id`: UUID da oportunidade | Obrigatória |
| `/chat?alert=escrow&negociacao_id={id}` | T-003 + T-013 | `negociacao_id`: UUID da negociação | Obrigatória |
| `/chat?alert=status&proposta_id={id}` | T-003 + T-014 | `proposta_id`: UUID da proposta | Obrigatória |
| `/perfil?tab=whatsapp` | T-015 Seção WhatsApp | `tab`: string | Obrigatória |
| `/perfil?tab=notificacoes` | T-018 Seção Notificações | `tab`: string | Obrigatória |
| `/perfil?tab=historico` | T-019 Histórico | `tab`: string | Obrigatória |
| `/admin/supervisao` | T-020 Listagem Admin | Filtros de data/confiança via query string | Obrigatória (Admin) |
| `/admin/supervisao/:id` | T-021 Detalhe + Takeover | `id`: UUID da interação | Obrigatória (Admin) |
| `/admin/dashboard` | T-022 Dashboard Admin | Filtros de período via query string | Obrigatória (Admin) |

### 6.3 Fallback Web

Todos os deep links são URLs HTTPS válidas. Se o app não estiver instalado (PWA não instalada) ou se acessado em browser desktop:

- A URL resolve normalmente no browser.
- O usuário é redirecionado para a tela correspondente na interface web.
- Não existe app nativo para redirecionar — o fallback é sempre a versão web.

### 6.4 Deep Link com Autenticação Necessária

**Fluxo quando o usuário não está autenticado:**

1. Usuário toca na notificação push.
2. Browser abre a URL do deep link (ex: `/chat?alert=opportunity&opportunity_id=abc123`).
3. Middleware de autenticação detecta sessão ausente.
4. URL original é salva em `sessionStorage.setItem('pendingDeepLink', url)`.
5. Usuário é redirecionado para `/login`.
6. Após autenticação bem-sucedida, `useEffect` verifica `pendingDeepLink`, redireciona e limpa o item.

**Caso especial — OTP de WhatsApp:** deep links de OTP de WhatsApp expiram em 5 minutos. Se o usuário demorar para autenticar, o deep link é seguido mas o OTP já expirou. O modal de vinculação (T-016) exibe estado de erro de OTP expirado com botão "Solicitar novo código".

---

## 7. Permissões Nativas

### 7.1 Tabela Completa de Permissões

| Permissão | Feature | Plataforma | Quando Solicitar | Mensagem de Contexto | Fallback se Negada |
|---|---|---|---|---|---|
| **Notifications** | Push Notifications (Web Push) | iOS 16.4+ (PWA), Android | Ao clicar em "Ativar alertas" em T-018 | "Ative para receber alertas de oportunidades, Escrow e propostas diretamente no seu dispositivo." | Banner em T-018 com link para configurações do browser; app funciona normalmente sem push |
| **Vibration** | Feedback háptico via Web Vibration API | Android Chrome | Nenhuma solicitação necessária (não requer permissão explícita) | N/A | Degradação graciosa: ação ocorre sem vibração |
| **Clipboard Write** | Copiar texto de ChatBubble (via long-press) | iOS, Android | Ao confirmar "Copiar" no menu de contexto | N/A (contexto implícito) | Toast: "Não foi possível copiar. Selecione e copie manualmente." |
| **Storage (localStorage/IndexedDB)** | Cache de histórico, fila offline, draft de input | iOS, Android | Implícita (não requer prompt) | N/A | Se bloqueado (modo privado): app funciona sem persistência; nenhuma mensagem é cacheada |

### 7.2 Permissões Não Utilizadas

O Repasse AI **não solicita** as seguintes permissões:

| Permissão | Justificativa |
|---|---|
| Camera | Nenhuma feature de captura de imagem no módulo |
| Microphone | Chat é text-only na Fase 1 e Fase 2 |
| Location | Não utilizada em nenhum fluxo do Repasse AI |
| Contacts | Não aplicável |
| Biometrics | Autenticação é gerenciada pelo módulo de Auth da plataforma Cessionário |
| Calendar | Não aplicável |

### 7.3 Comportamento ao Revogar Permissão nas Configurações do Sistema

| Permissão Revogada | Detecção | Comportamento |
|---|---|---|
| Notifications (revogada nas config do SO) | `Notification.permission === 'denied'` verificado ao abrir T-018 | Banner em T-018: "Alertas desativados nas configurações do sistema" + botão "Reativar nas configurações" (abre config do SO) |
| Storage (modo privado ativado) | Exceção ao tentar escrever em localStorage | App inicia em modo sem persistência; banner informativo no chat: "Modo privado ativo — histórico não será salvo" |

---

## 8. Diferenças iOS vs. Android

### 8.1 Tabela Comparativa por Feature

| Feature | iOS (Safari / PWA) | Android (Chrome) |
|---|---|---|
| **Navegação voltar** | Swipe-back da borda esquerda (Edge Swipe) | Botão físico ou software Back |
| **Fechar modal chat** | Swipe-down ou botão × | Botão voltar ou botão × |
| **Push Notifications** | Requer iOS 16.4+ **e** PWA instalada (Add to Home Screen); usa APNs | Funciona em browser normal (sem instalação); usa FCM; Android 13+ exige permissão explícita |
| **Teclado virtual** | Altera `visualViewport`, não `window.innerHeight`; `env(safe-area-inset-bottom)` obrigatório | Altera `window.innerHeight` via `resize` event; `windowSoftInputMode: adjustResize` |
| **Safe Area / Notch** | `env(safe-area-inset-top/bottom/left/right)` obrigatório para iPhone X+ e Dynamic Island | Variável — usar `padding-top: env(safe-area-inset-top, 24px)` como fallback |
| **Feedback háptico** | `navigator.vibrate()` não suportado; degradação graciosa | `navigator.vibrate()` suportado no Chrome |
| **Web Notifications (foreground)** | Não exibidas em foreground no Safari iOS (sem PWA) | Exibidas como notificação do sistema mesmo em foreground |
| **Clipboard Write** | Requer gesto do usuário (`user-interaction required`); `navigator.clipboard.writeText()` | Requer gesto do usuário; `navigator.clipboard.writeText()` + fallback `execCommand` |
| **Pull-to-refresh** | Rubber band de overscroll interfere; desabilitado no chat | Nativo suportado; habilitado onde aplicável |
| **Service Worker** | Suportado desde Safari 11.1; escopo limitado em iOS 15 | Suporte completo desde Chrome 45+ |
| **IndexedDB** | Limite de armazenamento mais restritivo (Safari pode expirar IDB em modo privado) | Limite maior; mais previsível |
| **Position: fixed** | Comportamento inconsistente com teclado aberto em iOS < 15.4; resolvido em iOS 15.4+ | Comportamento previsível |
| **CSS `dvh` (dynamic viewport height)** | Suportado a partir do Safari 15.4 | Suportado no Chrome 108+ |
| **100dvh para modal full-screen** | Usar `height: 100dvh` — resolve o problema do teclado virtual empurrando o viewport | Usar `height: 100dvh` — funciona corretamente |

### 8.2 Comportamentos Específicos de Plataforma

**iOS — Specific:**

1. **Dynamic Island (iPhone 14 Pro+):** o FAB (T-001) posicionado no canto inferior direito não conflita. O header do chat deve respeitar `env(safe-area-inset-top)` — mínimo de 59px no iPhone 14 Pro.
2. **Home Indicator:** `env(safe-area-inset-bottom)` = 34px no iPhone X/11/12/13/14/15 sem PWA em fullscreen. O input do chat deve ter `padding-bottom: calc(16px + env(safe-area-inset-bottom))`.
3. **Safari Address Bar (recolhível):** o cálculo de altura do chat deve usar `100dvh` para compensar o comportamento da barra de endereço.
4. **Bounce effect no scroll:** `overscroll-behavior: contain` no container do chat previne o bounce do body ao chegar no fim do histórico.

**Android — Specific:**

1. **Back gesture (Android 10+):** gesture navigation nas bordas laterais pode entrar em conflito com swipe-back do modal. Usar `preventDefaultTouchMove` na região de navegação do modal para Android detectado via `navigator.userAgent`.
2. **Samsung Internet:** pode não suportar `dvh` em versões mais antigas. Fallback: `height: calc(100vh - var(--keyboard-height, 0px))` calculado via `window.visualViewport`.
3. **Chrome autofill:** pode redimensionar o layout ao preencher campos de texto automaticamente. Desabilitar autofill em campos de OTP do WhatsApp via `autocomplete="one-time-code"` (iOS também) e `autocomplete="off"` em demais campos do chat se necessário.

### 8.3 Limitações por Plataforma

| Limitação | iOS | Android |
|---|---|---|
| Push sem PWA instalada | Web Push não funciona em iOS < 16.4 e sem "Add to Home Screen" | Funciona sem instalação |
| Háptico via Web Vibration | Não suportado | Suportado |
| Instalação como PWA | Requer "Adicionar à Tela de Início" manualmente | Chrome exibe banner automático de instalação se manifest estiver correto |
| Background sync (offline queue) | Background Sync API não suportada no Safari iOS | Suportada no Chrome Android |

> ⚙️ **Background Sync para fila offline no iOS:** [DECISÃO AUTÔNOMA] — Como Background Sync API não é suportada no Safari iOS, a fila de mensagens offline é drenada apenas quando o usuário retorna ao app em foreground. No Android, Background Sync API é usada para sincronização imediata ao reconectar mesmo em background. Alternativa descartada no iOS: polling via `setInterval` consumiria bateria e não é permitido em background pelo Safari.

---

## 9. Performance Mobile

### 9.1 Métricas-Alvo

| Métrica | Meta | Medição |
|---|---|---|
| **Time to Interactive (TTI)** | < 3s em 4G; < 5s em 3G | Lighthouse Mobile |
| **First Contentful Paint (FCP)** | < 1.5s em 4G | Lighthouse Mobile |
| **Largest Contentful Paint (LCP)** | < 2.5s em 4G | Lighthouse Mobile / Web Vitals |
| **Cumulative Layout Shift (CLS)** | < 0.1 | Lighthouse Mobile |
| **First Input Delay (FID) / INP** | < 100ms | Web Vitals real user monitoring |
| **Bundle size (JS inicial)** | < 200KB gzipped (módulo Repasse AI) | Webpack Bundle Analyzer |
| **Scroll FPS no chat** | 60 FPS em dispositivos P1; 30 FPS mínimo em P2 | Chrome DevTools Performance |
| **Tempo de abertura do chat (FAB → chat visível)** | < 300ms | Performance.mark() custom |
| **TTR (Time to Response do agente)** | p50 < 3s; p95 < 5s (inclui latência do backend) | Langfuse tracing |

### 9.2 Otimização de Listas

- **Histórico de mensagens (T-003):** virtualização com `react-window` ou `react-virtual`. Renderiza apenas as mensagens visíveis + buffer de 10 acima e abaixo.
- **Listagem Admin (T-020):** paginação server-side com `pageSize: 20`. Não renderiza toda a lista.
- **Quando não virtualizar:** se o histórico tiver < 50 mensagens, virtualização não é aplicada (overhead não compensado).

### 9.3 Otimização de Imagens e Gráficos

- Gráficos nas bolhas (ROI, score) são renderizados como SVG inline — sem requests adicionais.
- Ícones: sistema de ícones via sprite SVG ou `lucide-react` com tree-shaking.
- Sem imagens de usuário ou thumbnails no Repasse AI (sem feature de upload na Fase 1 e Fase 2).

### 9.4 Otimização do Bundle

- O módulo Repasse AI deve ser carregado via **lazy import** pela plataforma Cessionário:
  ```typescript
  const RepasseAIChat = lazy(() => import('@/modules/repasse-ai/ChatModal'));
  ```
- O bundle do módulo não deve ultrapassar 200KB gzipped. Inclui: componentes de chat, formatação de bolhas, lógica de SSE, Service Worker registration.
- LangChain.js e LangGraph.js ficam **exclusivamente no backend** — nunca importados no frontend.

### 9.5 Monitoramento e Alertas

| Ferramenta | Uso | Threshold de Alerta |
|---|---|---|
| Sentry (frontend) | Error tracking de JS no browser mobile | Taxa de erro > 1% em 5 min |
| Web Vitals (real user) | LCP, CLS, INP em produção | LCP > 4s P75 |
| Langfuse | TTR do agente por tela e tipo de análise | p95 TTR > 8s |
| Custom Performance API | Tempo de abertura do chat, tempo de 1ª mensagem | Abertura > 500ms P95 |

---

## 10. Acessibilidade Mobile

### 10.1 VoiceOver (iOS) e TalkBack (Android)

| Componente | Label ARIA / Semântica | Comportamento VoiceOver | Comportamento TalkBack |
|---|---|---|---|
| FAB (T-001) | `aria-label="Abrir chat Repasse AI"`, `role="button"` | Lê "Abrir chat Repasse AI, botão" + badge numérica "X alertas não lidos" | Lê "Abrir chat Repasse AI, botão, duplo toque para ativar" |
| ChatBubble — Usuário | `role="article"`, `aria-label="Você: {texto}"` | Lê o conteúdo da mensagem com contexto de remetente | Lê o conteúdo |
| ChatBubble — Agente | `role="article"`, `aria-label="Repasse AI: {texto}"`, `aria-live="polite"` para streaming | Anuncia mensagem ao completar streaming | Anuncia ao completar |
| ChatBubble — Streaming | `aria-live="polite"` + `aria-busy="true"` durante stream | VoiceOver anuncia ao finalizar (não anuncia cada token) | TalkBack anuncia ao finalizar |
| Input do chat | `aria-label="Digite sua mensagem"`, `role="textbox"` | Lê "Digite sua mensagem, campo de texto" | Lê com instrução de foco |
| Botão Enviar | `aria-label="Enviar mensagem"`, `role="button"` | Desabilitado quando input vazio: `aria-disabled="true"` | Igual ao iOS |
| Banner LGPD (T-002) | `role="dialog"`, `aria-modal="true"`, `aria-labelledby` | Foco é capturado no modal ao abrir | Foco é capturado no modal |
| Modal WhatsApp (T-016) | `role="dialog"`, `aria-modal="true"`, `aria-labelledby` | Foco capturado; steps anunciados | Foco capturado |
| StatusBadge | `aria-label="{status_legível}"`, `role="status"` | Lê status em linguagem clara (ex: "Vinculado") | Igual ao iOS |
| Toggle de Notificação (T-018) | `role="switch"`, `aria-checked="true/false"`, `aria-label="{tipo} notificações"` | Lê estado atual e instrução de ação | Igual ao iOS |

### 10.2 Tamanhos Mínimos de Toque

| Plataforma | Tamanho Mínimo | Elemento Mais Crítico | Verificação |
|---|---|---|---|
| iOS (HIG) | 44 × 44pt | FAB (T-001): mínimo 56 × 56pt; Botão Enviar: 44 × 44pt | Xcode Accessibility Inspector |
| Android (Material) | 48 × 48dp | FAB: mínimo 56 × 56dp; Botão Enviar: 48 × 48dp | Android Accessibility Scanner |

**Elementos que exigem atenção especial:**
- Botão × para fechar o chat: mínimo 44pt/44dp (não pode ser menor para fechar modal).
- Botões de ação nas ChatBubbles (ex: "Ver oportunidade", "Aguardar", "Tentar novamente"): mínimo 44pt/44dp de altura.
- Toggles de notificação (T-018): area de toque mínima de 44pt/44dp mesmo se o visual for menor.

### 10.3 Contraste e Fontes Dinâmicas

| Requisito | Padrão | Valor Mínimo |
|---|---|---|
| Contraste texto/fundo (WCAG AA) | ≥ 4.5:1 para texto normal | ChatBubble texto: #1A1A1A sobre #F5F5F5 — ratio 16:1 ✅ |
| Contraste texto grande (≥ 18pt / bold 14pt) | ≥ 3:1 | Headers de seção — verificar com tema dark |
| Dynamic Type (iOS) | Suporte obrigatório ao `accessibilityFontScale` | Texto do chat deve escalar até 200% sem quebrar layout |
| Font Scale (Android) | Suporte obrigatório até 200% | `rem` em CSS ao invés de `px` para textos |
| Tema escuro (Dark Mode) | Suportado via CSS `prefers-color-scheme` | Tokens de cor do Brand Theme Guide v1.0 |

### 10.4 Navegação por Teclado Externo (Bluetooth)

| Contexto | Comportamento |
|---|---|
| Tab navigation no chat | FAB → Input → Botão Enviar → ChatBubbles (em ordem reversa cronológica) |
| Foco em ChatBubble | Enter abre menu de contexto (Copiar / Reportar) |
| Modal aberto (T-002, T-016, T-017) | Tab fica preso dentro do modal (focus trap via `@radix-ui/react-dialog`) |
| Escape no modal | Fecha o modal (exceto T-002 Banner LGPD — Escape não fecha sem consentimento explícito) |
| Input ativo | Enter envia mensagem; Shift+Enter insere quebra de linha |

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 22/03/2026 | v1.0 | Versão inicial. Especificação mobile completa do Repasse AI: 10 seções, 3 decisões autônomas, 0 seções pendentes. Cobre navegação, gestos, offline, push, deep links, permissões, iOS vs. Android, performance e acessibilidade. |

---

## 12. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Pull-to-refresh desabilitado no iOS | [DECISÃO AUTÔNOMA] | 3.1 Gestos | Rubber band do iOS interfere com scroll do histórico. Desabilitado no iOS, habilitado no Android. Alternativa descartada: tratar conflito com CSS aumenta complexidade sem ganho de UX. | Baixo — usuário tem botão "Atualizar" como alternativa | Frontend Cessionário | Decidido |
| Double-tap sem ação no chat | [DECISÃO AUTÔNOMA] | 3.1 Gestos | Nenhum caso de uso identificado para double-tap em ChatBubble. Evita conflito com zoom de acessibilidade do iOS. Alternativa descartada: "curtir mensagem" não faz parte do produto. | Baixo | Frontend Cessionário | Decidido |
| Background Sync no iOS | [DECISÃO AUTÔNOMA] | 8.3 Limitações | Background Sync API não suportada no Safari iOS. Fila drenada apenas em foreground no iOS. Android usa Background Sync nativo. Alternativa descartada: polling via setInterval em background não permitido pelo Safari. | Médio — usuário pode ter delay na sincronização de mensagens após reconexão no iOS | Frontend Cessionário | Decidido — aceitar limitação da plataforma |

---

*Próximo documento do pipeline: D15 — Arquitetura de Pastas.*
