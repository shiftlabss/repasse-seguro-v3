# 11 - Mobile

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Produto, Mobile e Engenharia | Especificação mobile do AI-Dani-Cessionário — responsividade web mobile, WhatsApp (Fase 2), push notifications, gestos e diferenças de plataforma | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - **Fase 1 — Web Responsivo:** não há app nativo. O chat da Dani é 100% responsivo via browser mobile (breakpoint ≤ 640px). Nenhum React Native no escopo da Fase 1.
> - **Fase 2 — WhatsApp:** canal textual via EvolutionAPI (ADR-001). Sem app nativo — a interação é no próprio WhatsApp do usuário.
> - **Plataformas mobile suportadas na Fase 1:** iOS Safari ≥ 16 e Chrome Android ≥ 110 (browser).
> - **Telas críticas com comportamento offline definido:** chat (T-DC-002 a T-DC-004), Widget Top 3 (T-DC-009).
> - **Features nativas utilizadas:** notificações push web (PWA opcional), clipboard para copiar valores calculados.
> - **Diferenças críticas iOS vs Android:** comportamento do teclado virtual, safe area, back navigation.
> - **Zero seções pendentes por insuficiência de insumo.** Todos os comportamentos decididos com base no D02 (Stacks), D06 (Mapa de Telas) e D09 (Contratos de UI).

---

## 1. Plataformas e Versões Suportadas

### 1.1 Escopo de Plataformas

> ⚙️ **Contexto crítico:** O AI-Dani-Cessionário é um **módulo web responsivo** na Fase 1. Não há app React Native/Expo. O doc D02 (Stacks) define explicitamente: "Mobile refere-se à responsividade da interface web no Fase 1. Na Fase 2, o canal WhatsApp é via EvolutionAPI — não há app nativo."

| Plataforma | Tecnologia | Fase | Versão mínima |
|---|---|---|---|
| iOS — browser | Safari Mobile | Fase 1 | iOS 16 (Safari 16) |
| Android — browser | Chrome Mobile | Fase 1 | Chrome 110 / Android 9 |
| WhatsApp iOS | EvolutionAPI (canal textual) | Fase 2 | WhatsApp ≥ 23.x |
| WhatsApp Android | EvolutionAPI (canal textual) | Fase 2 | WhatsApp ≥ 23.x |
| React Native / Expo | Não aplicável | Fora do escopo | ADR obrigatório antes de iniciar |

### 1.2 Dispositivos de Referência para Testes

| Dispositivo | OS | Prioridade |
|---|---|---|
| iPhone 14 | iOS 16 (Safari) | P0 |
| iPhone SE (3ª geração) | iOS 16 (tela pequena — 375px) | P1 |
| Samsung Galaxy S23 | Android 13 (Chrome) | P0 |
| Xiaomi Redmi Note 12 | Android 12 (Chrome) | P1 |
| iPad Pro 11" | iOS 16 (Safari) | P2 (breakpoint tablet) |

### 1.3 Breakpoints e Densidade

| Breakpoint | Largura | Comportamento do chat |
|---|---|---|
| Mobile | ≤ 640px | Chat expande para 100vw × 100dvh. Radius superior removido. Teclado empurra o input para cima via `dvh` |
| Tablet | 641px–1024px | Chat permanece como overlay 400×600px no canto inferior direito |
| Desktop | > 1024px | Chat overlay padrão 400×600px |

---

## 2. Arquitetura de Navegação (Web Mobile)

> 💡 **Contexto:** Como a Dani é um overlay sobre a plataforma Repasse Seguro, não há navegação autônoma. A navegação mobile é gerenciada pela plataforma principal — a Dani flutua sobre ela.

### 2.1 Padrão de Navegação

- **Plataforma principal:** TanStack Router (browser history API) — gerenciada pela plataforma
- **Chat da Dani:** state machine via Zustand (`isChatOpen`, `chatContext`) — sem rotas próprias
- **FAB:** `position: fixed` sempre visível — não participa da navegação

### 2.2 Mapeamento Tela → Tipo de Navegação

| Tela | Tipo | Transição entrada | Transição saída | Back iOS | Back Android |
|---|---|---|---|---|---|
| T-DC-001 (FAB) | Fixed overlay | Scale + fade-in, 200ms | Scale + fade-out, 150ms | Não aplicável | Não aplicável |
| T-DC-002/003/004 (Chat) | Modal overlay (full-screen mobile) | Slide-up + fade-in, 250ms | Slide-down + fade-out, 200ms | Swipe-down fecha o chat | Botão back fecha o chat |
| T-DC-005 a T-DC-011 (Inline) | Conteúdo dentro do chat | Fade-in, 300ms | Não removidos individualmente | — | — |
| T-DC-012 (WhatsApp) | Seção dentro de Meu Perfil | Fade-in, 250ms | — | Swipe-back retorna ao topo | Back retorna ao menu anterior |

### 2.3 Comportamento do Back Button / Swipe-Back

**iOS (Safari Mobile):**
- Swipe da borda esquerda → navega na história do browser (plataforma principal)
- Chat aberto: swipe-down fecha o chat (gesture `onDismiss` via Framer Motion `drag`)
- Chat aberto: swipe da borda **não** propaga para o browser (stopPropagation no overlay)

**Android (Chrome):**
- Botão back físico/software: se chat está aberto → fecha o chat (interceptado via `popstate` event)
- Chat fechado: botão back navega na história do browser normalmente

### 2.4 Deep Links (Web)

| Ação | URL | Parâmetros | Comportamento |
|---|---|---|---|
| Abrir chat sem contexto | `/cessionario/dashboard?chat=open` | — | Abre T-DC-002 automaticamente |
| Abrir chat com OPR | `/cessionario/oportunidade/{opr_id}?chat=open` | `opr_id` | Abre T-DC-003 com contexto injetado |
| Abrir vinculação WhatsApp | `/cessionario/perfil/whatsapp` | — | Abre seção T-DC-012 |
| Alerta específico | `/cessionario/alertas/{alerta_id}` | `alerta_id` | Abre chat com contexto do alerta |

**Fallback:** Todas as URLs acima são acessíveis pelo browser sem o app — não há fallback necessário (web-only Fase 1).

---

## 3. Gestos e Interações Nativas (Web Mobile)

### 3.1 Tabela de Gestos

| Gesto | Contexto | Ação | Conflito com nativo | Alternativa acessível |
|---|---|---|---|---|
| Tap | FAB | Abre/fecha chat | Nenhum | Botão com `aria-label` e foco via Tab |
| Tap | Bubble / Card de OPR | Seleciona / expande | Nenhum | `role="button"`, `onKeyDown` Enter |
| Tap | Quick action chips | Dispara ação (simular, comparar) | Nenhum | Botão padrão com Tab + Enter |
| Swipe-down (vertical) | Chat aberto (mobile) | Fecha o chat | iOS swipe-back (edge) — não conflita (central vs. edge) | Botão X com `aria-label="Fechar chat"` |
| Pull-to-refresh | Área de mensagens do chat | Carrega mensagens mais antigas (paginação reversa) | Android pull-to-refresh do browser — interceptado dentro do chat | Botão "Carregar mensagens anteriores" no topo da lista |
| Long press | Bubble de mensagem | Menu contextual: copiar texto | Nenhum | Botão de cópia `...` ao lado da bubble |
| Scroll vertical | Área de mensagens | Navega no histórico | Scroll nativo do browser — funciona normalmente | Navegação via teclado virtual/tela |
| Pinch-to-zoom | Não suportado no chat | Bloqueado (`touch-action: none` no chat) | Evita zoom acidental no chat | Aumentar fonte via configurações do sistema (testado) |
| Double-tap | Não suportado | Sem ação | Safari double-tap para zoom — bloqueado via `meta viewport` | — |

### 3.2 Feedback Háptico

> [DECISÃO AUTÔNOMA] Web não suporta haptic feedback via API nativa (Vibration API tem suporte limitado no iOS Safari). Decisão: sem háptico na Fase 1 (web). Alternativa descartada: `navigator.vibrate()` — não suportado no iOS Safari 16. Na Fase 2 (WhatsApp), o háptico é gerenciado pelo próprio WhatsApp — fora do escopo da Dani.

| Ação | Feedback visual substituto |
|---|---|
| Mensagem enviada | Input limpa + botão enviar brief `opacity: 0.7` por 100ms |
| Rate limit atingido | Input desabilita + toast visual |
| OTP correto | Border do OTPInput muda para `--risk-low` (verde) |
| OTP incorreto | Shake animation no OTPInput, 300ms |
| Chat aberto | Slide-up animation com `ease-out` |

---

## 4. Comportamento Offline

### 4.1 Estratégia de Cache Local

**Tecnologia:** TanStack Query `staleTime` + `cacheTime` (in-memory, Fase 1)

> [DECISÃO AUTÔNOMA] Sem Service Worker / offline-first storage na Fase 1 (escopo web responsivo, não PWA). O cache é in-memory via TanStack Query — limpo ao fechar o browser. Alternativa descartada: IndexedDB com sw — adiciona complexidade de Service Worker sem justificativa de negócio na Fase 1 (chat requer conexão para funcionar).

| Recurso | `staleTime` | `cacheTime` | Dados disponíveis offline |
|---|---|---|---|
| Histórico de conversas | 30s | 5 min | Últimas mensagens carregadas em memória |
| Widget Top 3 Dashboard | 60s | 10 min | Cards carregados anteriormente |
| Alertas não lidos (badge FAB) | 30s | 5 min | Contagem em memória |
| Status do agente | 60s | 2 min | Último estado conhecido |

### 4.2 Telas com Suporte Offline

| Tela | Dados em cache | Ações possíveis offline | Ações bloqueadas offline |
|---|---|---|---|
| T-DC-001 (FAB) | Badge count (in-memory) | Exibir badge com último valor conhecido | Atualizar contagem em tempo real |
| T-DC-002/003/004 (Chat) | Histórico carregado em memória | Ler mensagens anteriores | Enviar novas mensagens |
| T-DC-009 (Widget Top 3) | Cards carregados em memória | Visualizar cards | Atualizar recomendações, abrir chat com contexto |
| T-DC-005 a T-DC-011 (Inline) | Respostas renderizadas em memória | Ler análises anteriores | Nova análise, nova simulação |
| T-DC-012 (WhatsApp) | Estado de vinculação (in-memory) | Visualizar estado atual | Iniciar vinculação, verificar OTP |

### 4.3 Indicadores Visuais de Estado de Rede

| Estado | Visual | Posicionamento |
|---|---|---|
| Online → Offline | Toast `Alert variant="destructive"` desliza de cima: "Sem conexão. Algumas funcionalidades podem estar indisponíveis." | Top da viewport, `position: fixed` |
| Offline → Online | Toast `Alert variant="default"` com ícone check: "Conexão restaurada." (auto-dismiss 3s) | Top da viewport |
| Chat offline | Input desabilitado + texto abaixo: "Sem conexão — não é possível enviar mensagens." | Dentro do input bar |

### 4.4 Comportamento ao Reconectar

1. `navigator.onLine` event listener detecta reconexão
2. TanStack Query invalida todas as queries com `staleTime` expirado automaticamente
3. Toast "Conexão restaurada." exibido por 3s
4. Chat input reativado imediatamente
5. Badge FAB atualizado com contagem real de alertas

**Conflitos de sincronização:** Não aplicável na Fase 1 — sem fila de ações offline (chat requer conexão ativa para enviar mensagens).

---

## 5. Push Notifications

### 5.1 Escopo de Push

> ⚙️ **Contexto:** Na Fase 1, push notifications para o chat da Dani são entregues via **alertas proativos no próprio chat** (in-app) e via badge do FAB. Web Push API (PWA) é opcional na Fase 1.

| Canal | Fase | Tecnologia | Cobertura |
|---|---|---|---|
| In-app (chat + badge FAB) | Fase 1 | SSE + TanStack Query | Online somente |
| Web Push (PWA) | Fase 1 — opcional | Web Push API + Service Worker | Background (browser fechado) |
| WhatsApp push | Fase 2 | EvolutionAPI | Nativo do WhatsApp |

### 5.2 Tipos de Notificação

| Tipo | Canal | Trigger | Conteúdo | Deep Link |
|---|---|---|---|---|
| Nova oportunidade relevante | In-app + Web Push (opcional) | `TipoAlerta.NOVA_OPORTUNIDADE` | "Nova oportunidade disponível: [OPR]" | `/cessionario/oportunidade/{opr_id}?chat=open` |
| Oportunidade encerrada | In-app + Web Push (opcional) | `TipoAlerta.OPORTUNIDADE_ENCERRADA` | "A oportunidade [OPR] foi encerrada." | `/cessionario/alertas/{alerta_id}` |
| Negociação atualizada | In-app + Web Push (opcional) | `TipoAlerta.NEGOCIACAO_ATUALIZADA` | "Sua negociação em [OPR] foi atualizada." | `/cessionario/negociacoes/{neg_id}?chat=open` |
| Escrow — prazo | In-app + Web Push (opcional) | `TipoAlerta.ESCROW_PRAZO` | "Prazo de Escrow para [OPR] vence em X dias." | `/cessionario/negociacoes/{neg_id}` |
| ZapSign pendente | In-app + Web Push (opcional) | `TipoAlerta.ZAPSIGN_PENDENTE` | "Documento aguardando sua assinatura." | `/cessionario/negociacoes/{neg_id}` |
| OTP de vinculação WhatsApp | SMS (Fase 2) | Usuário solicita vinculação | "Código Repasse Seguro: [XXXXXX]. Válido por 15 minutos." | — |

### 5.3 Permissões por Plataforma

| Plataforma | Comportamento padrão | Quando solicitar | Tratamento de recusa |
|---|---|---|---|
| iOS Safari (Web Push) | Bloqueado por padrão — usuário deve solicitar explicitamente | Após Cessionário ativar "Ativar alertas" (T-DC-002/009) | Exibir instruções: "Para receber alertas, vá em Configurações do Safari > [site] > Notificações > Permitir" |
| Android Chrome (Web Push) | Solicita permissão via prompt nativo | Após Cessionário ativar "Ativar alertas" | Toast informativo: "Alertas desabilitados. Para receber notificações, habilite em Configurações do Chrome." |
| WhatsApp (Fase 2) | Notificações gerenciadas pelo próprio WhatsApp | — | Se Cessionário silenciar o bot: alertas via in-app somente |

### 5.4 Comportamento por Estado do App

| Estado | iOS Safari | Android Chrome |
|---|---|---|
| App (browser) aberto | Toast in-app + badge FAB atualizado via SSE | Toast in-app + badge FAB atualizado |
| Browser em background | Web Push (se permissão concedida) via Service Worker | Web Push (se permissão concedida) |
| Browser fechado | Web Push (se permissão concedida, iOS 16.4+) | Web Push (se permissão concedida) |

### 5.5 Agrupamento de Notificações

- Múltiplos alertas do mesmo tipo agrupados: "3 oportunidades novas disponíveis"
- Badge do FAB exibe contagem total de não lidos (máximo "99+")
- Notificações push agrupadas por tag `dani-alerts` — substituem as anteriores se não lidas

---

## 6. Deep Links e Universal Links

### 6.1 Schema de URLs (Web)

Base: `https://app.repasseseguro.com.br`

| Rota | Tela | Parâmetros obrigatórios | Parâmetros opcionais |
|---|---|---|---|
| `/cessionario/dashboard` | Dashboard (T-DC-009) | — | `chat=open` |
| `/cessionario/dashboard?chat=open` | Dashboard + Chat aberto (T-DC-002) | — | — |
| `/cessionario/oportunidade/{opr_id}` | Tela de Oportunidade | `opr_id` | `chat=open` |
| `/cessionario/oportunidade/{opr_id}?chat=open` | Chat com contexto OPR (T-DC-003) | `opr_id` | — |
| `/cessionario/negociacoes/{neg_id}` | Tela de Negociação | `neg_id` | `chat=open` |
| `/cessionario/negociacoes/{neg_id}?chat=open` | Chat com contexto de negociação (T-DC-004) | `neg_id` | — |
| `/cessionario/alertas/{alerta_id}` | Chat com alerta aberto | `alerta_id` | — |
| `/cessionario/perfil/whatsapp` | Vinculação WhatsApp (T-DC-012) | — | — |

### 6.2 Tratamento de Deep Link com Autenticação Necessária

1. Usuário clica em link (ex: push notification)
2. TanStack Router verifica `isAuthenticated` via JWT no localStorage
3. **Autenticado:** navega direto para a rota + abre chat se `chat=open`
4. **Não autenticado:** redirect para `/login?redirect={encoded_url}`
5. Após login: redirect automático para a rota original com parâmetros preservados

### 6.3 Fallback

- Todas as rotas são web — não há fallback para app store (app nativo não existe na Fase 1)
- URL inválida ou `opr_id` inexistente: redirect para `/cessionario/dashboard` + toast "Oportunidade não encontrada."

---

## 7. Permissões Nativas

| Permissão | Feature | Quando solicitar | Mensagem de contexto | Fallback se negada |
|---|---|---|---|---|
| Notificações (Web Push) | Alertas proativos em background | Ao clicar "Ativar alertas" em T-DC-002 ou T-DC-009 | "Ative notificações para receber alertas quando novas oportunidades aparecerem." | Alertas somente in-app (chat aberto). Banner persistente: "Notificações desabilitadas — alertas disponíveis somente no chat." |
| Clipboard (copy) | Copiar valores calculados | Ao clicar "Copiar valores" em T-DC-010 | Não exibe prompt (Clipboard API não requer permissão explícita em iOS 13.4+ e Android) | Exibir valores em modal com instrução "Selecione e copie manualmente." |

**Comportamento ao revogar permissão de notificações:**
- Badge FAB continua funcionando (in-app)
- Web Push para de funcionar silenciosamente
- Ao abrir o chat: banner informativo "Notificações desabilitadas. Ative em Configurações para receber alertas." (exibido uma vez por sessão)

---

## 8. Diferenças iOS vs. Android

| Feature | iOS Safari | Android Chrome |
|---|---|---|
| Teclado virtual | Empurra o viewport para cima. Chat usa `100dvh` para compensar — garante que o input bar sempre visível | Redimensiona o viewport (`visual viewport` API). Comportamento similar com `dvh` |
| Safe area (notch/ilha) | `env(safe-area-inset-*)` obrigatório no FAB e input bar | `env(safe-area-inset-*)` em dispositivos com notch (Android 9+) |
| Scroll inercial | Nativo e fluido no iOS | Pode ter jank em dispositivos intermediários. Usar `overflow-y: scroll; -webkit-overflow-scrolling: touch` no chat |
| Swipe-back | Gesto de borda da esquerda — navega browser history | Botão back software/físico — interceptado via `popstate` |
| Web Push | Disponível apenas iOS 16.4+ com PWA adicionado à tela inicial | Disponível sem instalação desde Android 4+ |
| Clipboard API | Disponível sem permissão explícita iOS 13.4+ | Disponível sem permissão Chrome 66+ |
| `100dvh` (viewport height) | Suportado Safari 15.4+ | Suportado Chrome 108+ |
| Input `type="tel"` | Abre teclado numérico com campo telefone nativo | Abre teclado numérico simples |
| Input autofocus | Bloqueado por padrão no Safari mobile (exige interação do usuário) | Funciona normalmente |
| Long press em texto | Exibe menu nativo de seleção de texto | Exibe menu nativo |
| Animações Framer Motion | `transform` e `opacity` acelerados por hardware | `transform` e `opacity` acelerados. Evitar `box-shadow` animado (jank em mid-range) |
| `prefers-reduced-motion` | Respeitado nativamente via Media Queries | Respeitado nativamente via Media Queries |

---

## 9. Performance Mobile

### 9.1 Métricas-Alvo

| Métrica | Alvo | Medição |
|---|---|---|
| First Contentful Paint (FCP) | ≤ 2s em 4G | Lighthouse Mobile |
| Time to Interactive (TTI) | ≤ 3.5s em 4G | Lighthouse Mobile |
| Chat window open → primeiro pixel | ≤ 100ms | Performance.mark() |
| Scroll FPS na área de mensagens | ≥ 60fps | DevTools Performance tab |
| Bundle size (JS — chat module) | ≤ 150KB gzipped | Vite bundle analyzer |
| Latência API (análise individual) | ≤ 5s (RN-DC-029) | Pino log + Langfuse |

### 9.2 Otimização de Listas (Chat)

- Área de mensagens: virtualização via `react-virtual` ou `@tanstack/react-virtual` — renderizar apenas mensagens visíveis
- Paginação reversa: carregar 20 mensagens por vez. Pull-to-refresh carrega mais 20
- Images/avatars: lazy loading com `loading="lazy"` + placeholder `<Skeleton>`
- Framer Motion: usar `will-change: transform` apenas em elementos animados críticos (FAB, chat window) — não globalmente

### 9.3 Otimização de Imagens

- Avatar da Dani: SVG inline (sem request HTTP)
- Ícones: Lucide React (tree-shakable SVG)
- Gráfico de valorização (Chart): renderizado somente quando visível no viewport (`IntersectionObserver`)

### 9.4 Monitoramento

- Sentry Performance: rastrear `transaction` por abertura do chat e por análise de OPR
- Vitals via `web-vitals` package: CLS, LCP, FID reportados ao Sentry
- Alerta automático: FCP > 4s em produção → alerta Sentry P1

---

## 10. Acessibilidade Mobile

### 10.1 VoiceOver (iOS) e TalkBack (Android)

| Componente | VoiceOver (iOS) | TalkBack (Android) |
|---|---|---|
| FAB | `aria-label="Abrir chat com Dani — [N] alertas não lidos"` anunciado ao focar | Mesmo aria-label anunciado |
| Chat window | `role="dialog"`, `aria-modal="true"`, `aria-label="Chat com Dani"` — foco trap ativo | Mesmo comportamento |
| Área de mensagens | `role="log"`, `aria-live="polite"` — novas mensagens anunciadas automaticamente | Mesmo comportamento |
| `RiskScoreBadge` | `aria-label="Score de risco: [N] de 10 — risco [baixo/moderado/alto]"` | Mesmo |
| OTPInput | `role="group"`, `aria-label="Código de verificação — 6 dígitos"` | Mesmo |
| Rate limit counter | `aria-live="assertive"` na bubble de sistema ao atingir limite | Mesmo |

### 10.2 Tamanhos Mínimos de Toque

| Plataforma | Tamanho mínimo | Aplicação no AI-Dani |
|---|---|---|
| iOS | 44×44pt | FAB: 56×56px ✅. Chips de ação: min-height 44px ✅. Botão enviar: 44×44px ✅ |
| Android | 48×48dp | FAB: 56×56px ✅. Chips: min-height 48px ✅. OTP fields: min 48×48dp ✅ |

### 10.3 Contraste e Fontes Dinâmicas

- **Contraste mínimo:** 4.5:1 para texto, 3:1 para elementos de UI — conforme D09 Contratos de UI
- **Dynamic Type (iOS) / Font Size (Android):** componentes do chat usam `rem` (relativo ao font-size do sistema). Testado com fonte do sistema em 200% (sem quebra de layout)
- **Escala de fonte máxima suportada:** até 200% — abaixo disso o layout mantém integridade

### 10.4 Navegação por Teclado Externo (Bluetooth)

- Tab order: FAB → (se aberto) Header do chat → Área de mensagens → Input bar → Botão enviar
- `Escape`: fecha o chat
- `Enter` no input: envia mensagem. `Shift+Enter`: quebra linha
- `Tab` nos OTP fields: avança campo. `Shift+Tab`: volta campo
- Foco trap ativo dentro do chat quando aberto (foco não sai do overlay)

---

## 11. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. Escopo mobile clarificado (web responsivo Fase 1, WhatsApp via EvolutionAPI Fase 2). Breakpoints, navegação, gestos, offline, push, deep links, permissões, diferenças iOS/Android, performance e acessibilidade documentados. |

---

## 12. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Haptic feedback via Vibration API | Decisão Autônoma | §3.2 | Não suportado no iOS Safari — descartado na Fase 1. Reavaliar se houver React Native no futuro | P3 | Mobile Lead | Aberto |
| PWA (Service Worker + Web Push) | Decisão Autônoma | §5 | Opcional na Fase 1 — Web Push requer PWA instalado no iOS 16.4+. Implementar se taxa de uso mobile > 30% do total | P2 | Frontend Lead | Aberto |
| `react-virtual` para área de mensagens | Decisão Autônoma | §9.2 | Virtualização necessária para conversas longas (> 100 mensagens). Implementar na primeira release se histórico médio > 50 mensagens | P2 | Frontend Lead | Aberto |
| React Native / Expo (app nativo) | [DEFINIÇÃO PENDENTE] | §1 | ADR obrigatório antes de qualquer desenvolvimento. Opção A: PWA com Web Push (menor custo). Opção B: React Native + Expo (experiência nativa, maior investimento). Decisão depende de métricas de adoção mobile pós-Fase 2 | P1 | Tech Lead + Product | Aberto |
