# 06 - Mapa de Telas

## Repasse Seguro — Módulo Admin

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Produto, UX e Frontend |
| **Escopo** | Inventário completo de telas, hierarquia de navegação, estados visuais, responsividade e mapeamento de componentes |
| **Módulo** | Admin |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-22 (America/Fortaleza) |
| **Dependências** | 01.1–01.5 Regras de Negócio · 02 Stacks · 03 Brand Guide · 05.1–05.5 PRD |

---

> 📌 **TL;DR**
>
> - **Aplicação única:** SPA React + Vite — dashboard 100% logada, sem SSR.
> - **10 módulos** mapeados na sidebar: Dashboard, Pipeline, Triagem, Negociação, Formalização, Financeiro, Supervisão IA, Usuários, Relatórios, Configurações.
> - **4 perfis** com variantes de tela explícitas: Analista, Coordenador, Gestor Financeiro, Master.
> - **Total de telas/unidades documentais:** 68 telas (T-001 a T-068), incluindo modais, drawers e overlays com lógica própria.
> - **Plataformas:** Web (SPA — primário) + Mobile (React Native / Expo — secundário, operadores em campo).
> - Telas sem ID neste documento não existem para o pipeline.

---

## 1. Arquitetura de Navegação

### 1.1 Layout Global

O Admin é uma **SPA com layout shell fixo**:

```
┌─────────────────────────────────────────────────────────────┐
│  TOPBAR (h-14, fixo)                                        │
│  Logo | Breadcrumb                    Avatar | Notificações  │
├──────────────┬──────────────────────────────────────────────┤
│  SIDEBAR     │  ÁREA DE TRABALHO (conteúdo dinâmico)        │
│  (w-64,      │                                              │
│   fixo)      │                                              │
│              │                                              │
│  10 módulos  │                                              │
│              │                                              │
└──────────────┴──────────────────────────────────────────────┘
```

- **Topbar:** altura fixa `h-14`. Conteúdo: logo Repasse Seguro (esquerda), breadcrumb de localização (centro-esquerda), ícone de notificações com badge de contagem (direita), avatar do operador com dropdown (direita).
- **Sidebar:** largura fixa `w-64` em desktop. Collapsible para `w-16` (ícones only) em telas `< 1280px`. Cada item do menu: ícone + label. Item ativo com fundo `--accent` e borda esquerda `--primary` (4px). Hover: fundo `--accent`.
- **Área de trabalho:** `flex-1`, scroll vertical independente. Padding padrão `p-6`.

### 1.2 Hierarquia de Módulos (Sidebar)

```
Admin SPA
├── [T-001] Login / Autenticação
├── [T-002] 2FA — Verificação de código
├── [T-003] Recuperação de senha
├── [T-004] Redefinição de senha
│
├── Dashboard [T-010]
│
├── Pipeline
│   ├── [T-020] Pipeline — Visão Kanban
│   ├── [T-021] Pipeline — Visão Lista
│   └── [T-022] Pipeline — Detalhe do Caso
│
├── Triagem
│   ├── [T-030] Triagem — Fila FIFO
│   ├── [T-031] Triagem — Painel do Caso
│   ├── [T-032] Triagem — Aba Dossiê
│   ├── [T-033] Triagem — Aba Adimplência
│   └── [T-034] Modal — Confirmar Bloqueio de Caso
│
├── Negociação
│   ├── [T-040] Negociação — Lista de Casos em Negociação
│   ├── [T-041] Negociação — Painel do Caso
│   ├── [T-042] Negociação — Timeline de Propostas
│   ├── [T-043] Modal — Aceitar Proposta
│   ├── [T-044] Modal — Recusar / Contraproposta
│   └── [T-045] Modal — Aprovar Escalonamento
│
├── Formalização
│   ├── [T-050] Formalização — Lista de Casos
│   ├── [T-051] Formalização — Painel do Caso
│   ├── [T-052] Formalização — Card Assinaturas (ZapSign)
│   ├── [T-053] Formalização — Card Anuência da Construtora
│   ├── [T-054] Formalização — Card Depósito Escrow
│   └── [T-055] Modal — Confirmar Fechamento
│
├── Financeiro
│   ├── [T-060] Financeiro — Contas Escrow (tabela)
│   ├── [T-061] Financeiro — Drawer Detalhe da Conta Escrow
│   ├── [T-062] Financeiro — Painel Comissões e Faturas
│   ├── [T-063] Financeiro — Painel Inadimplência
│   ├── [T-064] Modal — Bloquear Distribuição
│   ├── [T-065] Modal — Iniciar Reversão (Estorno)
│   └── [T-066] Modal — Conciliação Bancária
│
├── Supervisão IA
│   ├── [T-070] Supervisão IA — Painel Principal
│   ├── [T-071] Supervisão IA — Log de Decisões do Agente
│   └── [T-072] Modal — Takeover Manual de Agente
│
├── Usuários
│   ├── [T-075] Usuários — Lista Operadores
│   ├── [T-076] Usuários — Perfil do Operador
│   ├── [T-077] Usuários — Lista Cedentes
│   ├── [T-078] Usuários — Perfil do Cedente
│   ├── [T-079] Usuários — Lista Cessionários
│   ├── [T-080] Usuários — Perfil do Cessionário
│   └── [T-081] Modal — Suspender / Reativar Usuário
│
├── Relatórios
│   ├── [T-085] Relatórios — Hub de Relatórios
│   ├── [T-086] Relatórios — SLA Operacional
│   ├── [T-087] Relatórios — Volume de Casos
│   ├── [T-088] Relatórios — Receita
│   ├── [T-089] Relatórios — Conversão
│   ├── [T-090] Relatórios — Auditoria
│   └── [T-091] Relatórios — Agentes de IA
│
└── Configurações
    ├── [T-095] Configurações — Hub
    ├── [T-096] Configurações — Comissões e Parâmetros Financeiros
    ├── [T-097] Configurações — Prazos e SLAs
    ├── [T-098] Configurações — Templates de Notificação
    ├── [T-099] Configurações — Integrações (ZapSign, Celcoin, Meta, Twilio)
    └── [T-100] Configurações — Usuários e Permissões
```

---

## 2. Inventário Completo de Telas

### 2.1 Bloco: Autenticação

---

**T-001 | Login / Autenticação**
- **Aplicação:** Admin Web + Mobile
- **Plataforma:** Desktop · Mobile · Tablet
- **Perfis que acessam:** Todos (pré-autenticação)
- **Rota:** `/login`
- **Descrição:** Tela de entrada da plataforma. Formulário de login com e-mail e senha. Logo do Repasse Seguro centralizada no topo. Card centralizado com `--radius-xl`, fundo `--card`, sombra suave.
- **Campos:** E-mail (input type=email, placeholder "seu@email.com", obrigatório), Senha (input type=password com toggle show/hide, obrigatório), Botão "Entrar" (primário, full-width, `--primary`).
- **Estados:**
  - `default`: formulário vazio, botão "Entrar" ativo
  - `loading`: ao clicar "Entrar", botão muda para spinner + "Entrando..." + desabilitado
  - `erro credencial`: campo e-mail e senha com borda `--destructive`, mensagem inline abaixo do campo senha: "E-mail ou senha incorretos."
  - `erro bloqueio (5 tentativas)`: banner `--destructive` acima do formulário: "Conta temporariamente bloqueada. Tente novamente em 30 minutos." Formulário desabilitado.
  - `erro rede`: toast vermelho "Não foi possível conectar. Verifique sua conexão."
- **Navegação:** Sucesso → T-002 (se 2FA ativo) ou T-010 (Dashboard)
- **Responsividade:**
  - Desktop: card `max-w-md` centralizado, fundo `--muted`
  - Tablet: igual ao desktop
  - Mobile: card full-width com padding `p-4`, fundo branco
- **Acessibilidade:** `aria-label` em todos os inputs. Focus ring visível. Mensagens de erro com `role="alert"`. Tab order: e-mail → senha → toggle show/hide → botão Entrar.
- **Microinteração:** Shake horizontal (200ms) no card ao erro de credencial. Botão com estado de loading previne duplo clique.

---

**T-002 | 2FA — Verificação de Código**
- **Aplicação:** Admin Web + Mobile
- **Perfis que acessam:** Todos (pós-credencial, pré-acesso)
- **Rota:** `/login/2fa`
- **Descrição:** Tela de verificação do código de autenticação de 2 fatores (app autenticador). Card idêntico ao T-001. Instrução clara: "Digite o código de 6 dígitos do seu aplicativo autenticador."
- **Campos:** 6 inputs individuais (1 dígito cada, type=number, max=1), agrupados como OTP input. Botão "Verificar" (primário, full-width). Link "Usar código de recuperação" (secondary).
- **Estados:**
  - `default`: 6 inputs vazios
  - `preenchendo`: foco avança automaticamente para o próximo input ao digitar
  - `loading`: botão "Verificar" → spinner + "Verificando..." + desabilitado
  - `erro código inválido`: borda vermelha nos 6 inputs + mensagem "Código incorreto. Verifique seu aplicativo."
  - `expirado`: mensagem "Código expirado. Aguarde o próximo código (30s)" com countdown visual
- **Navegação:** Sucesso → T-010 (Dashboard)
- **Responsividade:** Igual T-001.
- **Acessibilidade:** `aria-label="Dígito X de 6"` em cada input. `role="alert"` em mensagens de erro.
- **Microinteração:** Paste automático detecta string de 6 dígitos e distribui nos campos. Shake ao erro.

---

**T-003 | Recuperação de Senha**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Todos (não autenticado)
- **Rota:** `/recuperar-senha`
- **Descrição:** Formulário simples. Campo de e-mail. Botão "Enviar instruções".
- **Estados:**
  - `default`: campo vazio
  - `loading`: botão com spinner
  - `sucesso`: mensagem verde "Instruções enviadas para [email]. Verifique sua caixa de entrada." Campo e botão ocultados.
  - `e-mail não encontrado`: mensagem neutra (segurança): "Se este e-mail estiver cadastrado, você receberá as instruções."
- **Navegação:** Link "Voltar ao login" → T-001
- **Responsividade:** Igual T-001.
- **Acessibilidade:** `aria-live="polite"` na área de feedback.

---

**T-004 | Redefinição de Senha**
- **Aplicação:** Admin Web
- **Rota:** `/redefinir-senha?token=XXX`
- **Campos:** Nova senha (com regras de força visíveis: 8+ chars, maiúscula, número, símbolo), Confirmar nova senha. Botão "Redefinir senha".
- **Estados:**
  - `token inválido/expirado`: página exibe erro com CTA "Solicitar novo link" → T-003
  - `validação`: indicador de força da senha em tempo real (barra colorida: fraca/média/forte)
  - `sucesso`: toast "Senha redefinida com sucesso." Redireciona para T-001 após 3s.
- **Responsividade:** Igual T-001.

---

### 2.2 Bloco: Dashboard

---

**T-010 | Dashboard Principal**
- **Aplicação:** Admin Web + Mobile
- **Perfis que acessam:** Todos (conteúdo filtrado por perfil)
- **Rota:** `/dashboard`
- **Descrição:** Primeira tela após login. Visão consolidada operacional com KPIs, funil de casos, atividades pendentes e alertas de SLA. Conteúdo varia por perfil.
- **Layout (Desktop):** Grid 12 colunas. Zona superior: 4 cards de KPI (3 colunas cada). Zona média: gráfico funil (8 colunas) + ranking da equipe (4 colunas). Zona inferior: tabela de atividades pendentes (8 colunas) + alertas SLA (4 colunas).
- **Variantes por perfil:**
  - *Analista:* KPIs = "Meus casos ativos", "SLAs vencendo hoje", "Casos em triagem", "Casos em negociação". Gráfico = funil dos seus casos. Ranking = sua posição.
  - *Coordenador:* KPIs = "Casos ativos total", "SLAs vencidos", "Analistas ativos", "Casos sem atribuição". Gráfico = funil geral. Ranking = todos os Analistas.
  - *Gestor Financeiro:* KPIs = "Escrows pendentes", "Valor em custódia", "Distribuições esta semana", "Faturas em aberto". Sem gráfico de funil — substitui por gráfico de distribuições por período.
  - *Master:* todos os KPIs disponíveis. Toggle entre visão operacional e financeira.
- **Estados:**
  - `loading`: skeleton em todas as zonas (animação pulse)
  - `loaded`: dados exibidos com fade-in staggered (100ms entre cada zona)
  - `erro parcial`: zona com falha exibe card de erro inline com botão "Tentar novamente" — demais zonas continuam normais
  - `offline`: banner superior "Dados podem estar desatualizados — sem conexão" + dados em cache exibidos com badge "cache"
  - `primeiro acesso`: todas as zonas vazias + card de onboarding central: "Bem-vindo ao Repasse Seguro Admin. Comece cadastrando o primeiro caso." + CTA "Ir para Pipeline"
- **Filtros:** Período (Hoje / Esta semana / Este mês / Customizado) no canto superior direito.
- **Responsividade:**
  - Desktop (≥1280px): layout grid completo conforme descrito
  - Tablet (768–1279px): sidebar collapsa para ícones. KPIs em grid 2x2. Gráfico e ranking em coluna única.
  - Mobile (<768px): KPIs em lista vertical (1 por linha), gráfico simplificado (barra), ranking oculto, atividades limitadas a 5 + botão "Ver todas"
- **Acessibilidade:** Gráficos com `aria-label` descritivo. KPI cards com `role="status"`. Tab order lógico (KPIs → gráfico → atividades → alertas).
- **Microinteração:** KPIs com contador animado ao carregar (easing ease-out, 600ms). Hover em card KPI: elevação sutil (shadow-md). Atualização automática a cada 60s via polling Redis.

---

### 2.3 Bloco: Pipeline

---

**T-020 | Pipeline — Visão Kanban**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Analista (seus casos), Coordenador (todos), Master (todos), Gestor Financeiro (leitura)
- **Rota:** `/pipeline?view=kanban`
- **Descrição:** Board kanban com 14 colunas (1 por estado do ciclo de vida do caso). Header por coluna: nome do estado + contagem de cards. Cards ordenados por data de entrada no estado (mais antigos no topo).
- **Card de caso:** ID do caso (badge), endereço do imóvel (título, truncate 2 linhas), Analista responsável (avatar 24px + nome), cenário (A/B/C/D badge colorido), tempo no estado atual (badge relativo: "Há 3 dias"), indicador SLA (verde/amarelo/vermelho), valor estimado.
- **Estados:**
  - `loading`: skeleton cards por coluna
  - `vazio por coluna`: mensagem "Nenhum caso neste estado" (sem CTA — estado não é ponto de entrada)
  - `vazio total`: mensagem central "Nenhum caso cadastrado. Aguardando cadastro pelo Cedente."
  - `atualização em tempo real`: Supabase Realtime — card move entre colunas com animação de slide horizontal (300ms ease-in-out). Badge de "Atualizado agora" aparece no card por 3s.
- **Interações:**
  - Click no card → T-022 (Detalhe do Caso) abrindo como drawer lateral direito (480px)
  - Drag-and-drop: disponível apenas para Coordenador e Master. Arrasta card entre estados permitidos pelo ciclo de vida. Estados inválidos para drop: coluna fica com opacidade 0.4.
- **Filtros:** barra de filtros acima do board: Analista (multi-select), Cenário (A/B/C/D), SLA (Vencendo hoje / Vencido / Em dia).
- **Toggle:** botão "Kanban / Lista" no canto superior direito → T-021.
- **Responsividade:**
  - Desktop (≥1280px): scroll horizontal do board. Sidebar colapsada recomendada.
  - Tablet (768–1279px): scroll horizontal mantido. Drag-and-drop desabilitado (use ações via tap).
  - Mobile (<768px): visão kanban **não disponível** em mobile. Redireciona automaticamente para T-021 (Lista).
- **Acessibilidade:** Drag-and-drop com alternativa via menu de contexto (botão "Mover para..." em cada card). `aria-label` por coluna com contagem. Cards com `role="listitem"`.
- **Microinteração:** Coluna de destino válida: destaque com borda `--primary` ao arrastar card sobre ela. Snap animation ao soltar.

---

**T-021 | Pipeline — Visão Lista**
- **Aplicação:** Admin Web + Mobile
- **Perfis que acessam:** Analista (seus casos), Coordenador (todos), Master (todos), Gestor Financeiro (leitura)
- **Rota:** `/pipeline?view=list`
- **Descrição:** Tabela paginada de todos os casos. Paginação: 25 por página.
- **Colunas:** ID, Imóvel (endereço truncado), Estado atual (badge colorido), Cenário (A/B/C/D), Analista (avatar + nome), Tempo no estado, SLA (ícone colorido), Valor estimado, Ações.
- **Ações por linha:** botão "Ver detalhe" → T-022. Coordenador/Master: dropdown com "Reatribuir analista", "Cancelar caso".
- **Filtros:** mesmos do kanban + busca por endereço/ID.
- **Estados:** loading (skeleton rows), vazio (ilustração + texto "Nenhum caso encontrado com esses filtros"), erro (inline alert).
- **Responsividade:**
  - Desktop: tabela completa
  - Tablet: colunas "Valor" e "Analista" ocultadas; acessíveis via expansão da linha
  - Mobile: lista de cards (1 por caso), sem tabela. Cada card: ID + endereço + estado + SLA. Tap para T-022.
- **Acessibilidade:** `role="table"`, `scope="col"` em headers, `aria-sort` em colunas ordenáveis.

---

**T-022 | Pipeline — Detalhe do Caso (Drawer)**
- **Aplicação:** Admin Web + Mobile
- **Perfis que acessam:** Todos (conteúdo filtrado por perfil)
- **Descrição:** Drawer lateral direito (480px desktop, fullscreen mobile) que abre sobre a lista/kanban sem navegar. Exibe: cabeçalho do caso (ID, endereço, estado atual como badge, cenário), tabs: Resumo · Dossiê · Histórico · Ações.
- **Tab Resumo:** dados do Cedente, Cessionário (quando houver), valores calculados, Analista responsável, timeline simplificada do ciclo de vida.
- **Tab Dossiê:** lista de documentos do dossiê com status (Pendente/Verificado/Rejeitado), botões de visualizar/baixar.
- **Tab Histórico:** log cronológico de todas as ações no caso (quem fez o quê, quando).
- **Tab Ações:** ações disponíveis para o perfil do operador logado (ex: Analista vê "Iniciar Triagem"; Coordenador vê "Reatribuir", "Cancelar caso").
- **Estados:**
  - `loading`: skeleton nas tabs
  - `erro ao carregar`: mensagem de erro com botão "Tentar novamente"
- **Fechamento:** botão "×" no canto superior direito ou clique fora do drawer (com overlay semitransparente).
- **Responsividade:**
  - Desktop: drawer 480px, overlay escurece o restante
  - Mobile: fullscreen slide-up from bottom (comportamento de sheet)
- **Acessibilidade:** Focus trap dentro do drawer. `aria-modal="true"`. Fechamento via Escape key. `aria-labelledby` no título.

---

### 2.4 Bloco: Triagem

---

**T-030 | Triagem — Fila FIFO**
- **Aplicação:** Admin Web + Mobile
- **Perfis que acessam:** Analista, Coordenador, Master
- **Rota:** `/triagem`
- **Descrição:** Lista ordenada por data de entrada (FIFO). Cada item: ID do caso, endereço do imóvel, data de entrada, tempo na fila (relativo: "Há 2 dias"), progresso do dossiê (ex: "3/6 documentos" como barra de progresso), analista atribuído (se houver).
- **Destaque FIFO:** o caso no topo da fila recebe borda esquerda azul `--primary` + badge "Próximo na fila".
- **Alerta fora de ordem:** ao selecionar caso não-FIFO, banner amarelo acima do painel: "Este caso não é o próximo na fila. O caso [ID] aguarda há mais tempo." + link "Ir para o caso mais antigo".
- **Estados:**
  - `loading`: skeleton list
  - `vazio`: "Nenhum caso aguardando triagem no momento." (sem CTA)
  - `erro`: alert inline com "Tentar novamente"
- **Responsividade:**
  - Desktop/Tablet: lista lateral (1/3 da tela) + painel de trabalho (2/3) — layout master-detail
  - Mobile: lista fullscreen → tap abre T-031 (push navigation)
- **Acessibilidade:** `role="list"`, itens com `role="listitem"`. Foco no primeiro item ao abrir a página.

---

**T-031 | Triagem — Painel do Caso**
- **Aplicação:** Admin Web + Mobile
- **Perfis que acessam:** Analista, Coordenador, Master
- **Rota:** `/triagem/:id`
- **Descrição:** Painel de trabalho do Analista para um caso em triagem. Layout de duas colunas: esquerda (60%) com abas Dossiê e Adimplência; direita (40%) com sidebar de informações do caso, histórico de ações e botões de ação principal.
- **Header do caso:** ID + endereço + estado atual + cenário + Cedente. Botão "Retornar à fila" (link, canto superior esquerdo).
- **Estados:**
  - `caso bloqueado`: banner fixo no topo "Caso Bloqueado — Inadimplência" (fundo `--destructive`, texto branco). Todos os botões de avanço desabilitados. Apenas "Desbloquear" visível (para Analista atribuído e Coordenador).
  - `caso qualificado`: banner verde "Caso Qualificado — Aguardando publicação de oferta pelo Coordenador."
  - `loading`: skeleton nas abas
- **Ações da sidebar direita:**
  - *Analista:* "Bloquear Caso" (vermelho outline), "Qualificar Caso" (primário, habilitado apenas quando todos os critérios atendidos), "Solicitar documentos" (secondary)
  - *Coordenador:* + "Cancelar Caso" (destructive)
  - *Master:* igual ao Coordenador + "Publicar Oferta" diretamente (skip da aprovação do Coordenador)
- **Responsividade:**
  - Desktop: duas colunas conforme descrito
  - Tablet: duas colunas com sidebar reduzida (30%)
  - Mobile: abas fullscreen; sidebar colapsada como accordion abaixo das abas
- **Acessibilidade:** Botões destrutivos com `aria-describedby` apontando para modal de confirmação. Focus management ao abrir modal.

---

**T-032 | Triagem — Aba Dossiê**
- **Aplicação:** Admin Web + Mobile
- **Descrição:** Lista dos 6 documentos obrigatórios do dossiê com status individual.
- **Documentos e status:** cada documento exibe nome, tipo esperado, data de upload, operador que verificou, status (Pendente ⏳ / Verificado ✅ / Rejeitado ❌).
- **Ações por documento:** "Visualizar" (abre PDF em modal/aba nova), "Verificar" (muda status para Verificado, com carimbo de nome + timestamp), "Rejeitar" (abre modal com campo de motivo obrigatório), "Solicitar reenvio" (visível quando Rejeitado).
- **Contador:** "X/6 documentos verificados" como barra de progresso no topo da aba.
- **Upload:** botão "Adicionar documento" para o Analista incluir documentos recebidos fora do sistema (máx 10MB, PDF/JPEG/PNG).
- **Estados:**
  - `dossiê completo`: barra de progresso 100% + banner verde "Dossiê completo. Todos os 6 documentos verificados."
  - `documento rejeitado`: linha do documento com fundo `--destructive/10`, ícone ❌, motivo da rejeição visível inline
  - `aguardando upload`: documento sem arquivo ainda — label "Pendente de envio pelo Cedente" (cinza)
- **Responsividade:** Em mobile, lista de cards em vez de tabela.

---

**T-033 | Triagem — Aba Adimplência**
- **Aplicação:** Admin Web
- **Descrição:** Checklist de verificação de adimplência. Lista das 3 últimas parcelas do contrato com status de pagamento (comprovante anexado / verificado / em atraso). Campo para registro de observações do Analista.
- **Ações:** "Confirmar adimplência" (botão primário — habilita avanço), "Registrar inadimplência" (botão vermelho outline — dispara T-034).
- **Exceção de adimplência:** toggle "Aprovar via confirmação da construtora" — visível apenas para Coordenador. Ao ativar: campo obrigatório "Número do protocolo / referência da confirmação" + upload do documento da construtora.
- **Estados:** `verificado` (banner verde), `inadimplente` (banner vermelho), `aguardando comprovante` (banner amarelo)

---

**T-034 | Modal — Confirmar Bloqueio de Caso**
- **Aplicação:** Admin Web + Mobile
- **Trigger:** Clique em "Bloquear Caso" em T-031
- **Descrição:** Modal de confirmação centralizado (`--radius-xl`, `max-w-sm`). Título: "Confirmar bloqueio por inadimplência". Corpo: "O Cedente será notificado automaticamente com orientações para regularização." Checkbox obrigatório: "Confirmo que as parcelas em atraso foram identificadas no comprovante."
- **Ações:** "Confirmar bloqueio" (primário `--destructive`) | "Cancelar" (secondary)
- **Estados:** `loading` ao confirmar (spinner no botão), `sucesso` (modal fecha + toast "Caso bloqueado. Cedente notificado." + status do caso atualizado na tela)
- **Acessibilidade:** `role="dialog"`, `aria-modal="true"`, focus trap, fechamento via Escape.

---

### 2.5 Bloco: Negociação

---

**T-040 | Negociação — Lista de Casos em Negociação**
- **Aplicação:** Admin Web + Mobile
- **Perfis que acessam:** Analista (seus casos), Coordenador, Master
- **Rota:** `/negociacao`
- **Descrição:** Lista de casos nos estados "Em Negociação" e "Oferta Ativa". Similar ao T-030 (master-detail). Colunas: ID, imóvel, estado, cenário, Cessionário(s) com proposta, valor da melhor proposta, prazo da proposta, SLA.
- **Estados:** loading (skeleton), vazio ("Nenhum caso em negociação no momento."), erro.
- **Responsividade:** Igual T-030.

---

**T-041 | Negociação — Painel do Caso**
- **Aplicação:** Admin Web + Mobile
- **Rota:** `/negociacao/:id`
- **Descrição:** Painel de mediação de propostas. Layout igual ao T-031. Abas: Timeline de Propostas · Informações do Caso · Histórico.
- **Sidebar de ações:**
  - *Analista:* "Aceitar proposta" (CTA primário), "Recusar / Contraproposta" (secondary), "Escalar para Coordenador"
  - *Coordenador:* + "Aprovar Escalonamento de Cenário" (se aplicável), "Cancelar negociação"
- **Alerta de proposta expirada:** banner amarelo "Proposta expirada em [data]. Aguardando nova proposta do Cessionário."
- **Responsividade:** Igual T-031.

---

**T-042 | Negociação — Timeline de Propostas**
- **Aplicação:** Admin Web
- **Descrição:** Histórico cronológico de propostas e contrapropostas em formato de timeline vertical. Cada entrada: tipo (Proposta / Contraproposta / Aceite / Recusa), valor, autor (Cessionário ou Analista/RS), data/hora, status (Ativa / Expirada / Aceita / Recusada).
- **Proposta ativa:** entrada com borda `--primary` + badge "Ativa" + exibição do prazo restante (countdown regressivo se <24h).
- **Responsividade:** Mobile: timeline simplificada (apenas tipo + valor + status).

---

**T-043 | Modal — Aceitar Proposta**
- **Aplicação:** Admin Web + Mobile
- **Trigger:** Clique em "Aceitar proposta" em T-041
- **Descrição:** Modal de confirmação. Exibe resumo: valor da proposta, Cessionário, cenário do Cedente, comissão calculada automaticamente (Cedente e Cessionário). Checkbox: "Confirmo os valores calculados."
- **Ações:** "Confirmar aceite" (primário) | "Cancelar" (secondary)
- **Pós-aceite:** caso muda para "Em Formalização". Toast "Negociação concluída. Caso movido para Formalização." Redireciona para T-050.

---

**T-044 | Modal — Recusar / Contraproposta**
- **Aplicação:** Admin Web + Mobile
- **Trigger:** Clique em "Recusar / Contraproposta" em T-041
- **Descrição:** Modal com toggle: "Recusar" ou "Fazer contraproposta". Se Recusar: campo motivo (select + textarea opcional). Se Contraproposta: campo valor (currency input, validado contra regras de negócio).
- **Validação contraproposta:** valor deve ser maior que a proposta original. Se menor: erro inline "A contraproposta deve ser superior à proposta recebida."
- **Ações:** "Confirmar" (primário) | "Cancelar" (secondary)

---

**T-045 | Modal — Aprovar Escalonamento de Cenário**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Coordenador, Master
- **Trigger:** Clique em "Aprovar Escalonamento" em T-041
- **Descrição:** Modal com exibição do escalonamento proposto (cenário atual → cenário inferior) e implicações financeiras para o Cedente (comparação dos valores antes/depois). Campo de confirmação: o Coordenador deve digitar "CONFIRMAR" para habilitar o botão de aprovação.
- **Ações:** "Aprovar escalonamento" (primário, habilitado apenas após digitar confirmação) | "Cancelar" (secondary)

---

### 2.6 Bloco: Formalização

---

**T-050 | Formalização — Lista de Casos**
- **Aplicação:** Admin Web + Mobile
- **Perfis que acessam:** Analista, Coordenador, Master
- **Rota:** `/formalizacao`
- **Descrição:** Lista de casos no estado "Em Formalização". Colunas: ID, imóvel, critérios cumpridos (badges: Assinaturas ✅/⏳, Anuência ✅/⏳/N/A, Depósito ✅/⏳), dias no estado, SLA.
- **Destaque:** caso com todos os 4 critérios cumpridos recebe badge "Pronto para fechamento" em verde.
- **Responsividade:** Igual T-030.

---

**T-051 | Formalização — Painel do Caso**
- **Aplicação:** Admin Web + Mobile
- **Rota:** `/formalizacao/:id`
- **Descrição:** Painel central de acompanhamento dos 4 critérios obrigatórios de Fechamento. Layout: cabeçalho do caso + 4 cards de critério dispostos em grid 2x2 (desktop) ou lista (mobile).
- **4 cards de critério:** Assinaturas (T-052), Anuência da Construtora (T-053), Depósito Escrow (T-054), Instrumento Assinado (verificação inline no card).
- **Botão "Confirmar Fechamento":** visível na sidebar direita. Habilitado apenas quando todos os 4 critérios mostram status ✅. Ao clicar → T-055 (Modal).
- **Banner de contingência ZapSign (quando ativo):** banner amarelo global no topo: "Serviço de assinaturas temporariamente indisponível." + timestamp + link "Registrar assinatura manual" (Coordenador/Master only).
- **Responsividade:** Desktop: layout 2x2. Tablet: lista de 4 cards em coluna. Mobile: accordion por critério.

---

**T-052 | Formalização — Card Assinaturas (ZapSign)**
- **Aplicação:** Admin Web
- **Descrição:** Card expandido dentro de T-051. Exibe: status de cada signatário (Cedente / Cessionário / Construtora quando exigida) com badge individual (Pendente / Assinado / Recusado / Expirado). Barra de progresso fracionária "X/3 assinaturas".
- **Ações:** "Enviar para assinatura" (primário, initial state), "Reenviar" ao lado de signatário Pendente (após 5 dias) ou Expirado. Contador de reenvios "Reenvio X de 3".
- **Atualização em tempo real:** badge de cada signatário atualiza via webhook ZapSign sem reload.
- **Critério cumprido:** card com fundo verde leve + badge "✅ Assinaturas concluídas".

---

**T-053 | Formalização — Card Anuência da Construtora**
- **Aplicação:** Admin Web
- **Descrição:** Card para registro e confirmação da anuência. Se o contrato exige anuência: campo para anexar documento da construtora + campo de data de emissão + campo de protocolo. Botão "Confirmar anuência recebida". Se anuência não exigida: card mostra "N/A — contrato não exige anuência" + badge cinza.
- **Estados:** Pendente / Confirmada / N/A.
- **Critério cumprido:** card com fundo verde leve.

---

**T-054 | Formalização — Card Depósito Escrow**
- **Aplicação:** Admin Web
- **Descrição:** Card de monitoramento do depósito na Conta Escrow. Exibe: valor esperado, valor depositado (atualizado em tempo real via Celcoin webhook), status (Aguardando / Parcial / Confirmado), data limite de depósito (com countdown se <48h), histórico de movimentações.
- **Alerta prazo:** badge "Prazo próximo" (amarelo, ≤3 dias) ou "Prazo estourado" (vermelho).
- **Ação do Coordenador:** "Prorrogar prazo de depósito" (link, registra novo prazo com justificativa).
- **Critério cumprido:** card com fundo verde leve.

---

**T-055 | Modal — Confirmar Fechamento**
- **Aplicação:** Admin Web
- **Trigger:** Clique em "Confirmar Fechamento" em T-051
- **Descrição:** Modal de confirmação final. Exibe sumário completo: 4 critérios cumpridos com checkmarks, valores calculados (comissão Cedente + comissão Cessionário + valor líquido ao Cedente), data e hora do fechamento.
- **Confirmação em duplo:** campo obrigatório: Analista digita "FECHAR" para habilitar o botão. Se Delta ≥ R$100.000: botão desabilitado com mensagem "Aguardando aprovação do Coordenador." + notificação automática ao Coordenador.
- **Ações:** "Confirmar Fechamento" (primário `--primary`) | "Cancelar" (secondary)
- **Pós-fechamento:** caso muda para "Pós Fechamento". Toast "Fechamento confirmado. Contagem de 15 dias iniciada."

---

### 2.7 Bloco: Financeiro

---

**T-060 | Financeiro — Contas Escrow**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Gestor Financeiro, Coordenador (leitura), Master
- **Rota:** `/financeiro/escrow`
- **Descrição:** Tabela de todas as Contas Escrow ativas e históricas. Colunas: ID do caso, Status da Conta Escrow, Valor Esperado, Valor Depositado, Data do Depósito, Data Prevista de Distribuição, Ações.
- **Filtros:** chips clicáveis acima da tabela por status (Aberta / Depósito Confirmado / Em Distribuição / Distribuída / Congelada / Estornada). Múltiplos filtros combinados.
- **Destaques:** "Prazo próximo" (badge amarelo) e "Prazo estourado" (badge vermelho) com tooltip de dias.
- **Ação por linha:** "Ver detalhe" → T-061 (drawer lateral). Gestor Financeiro: dropdown com "Bloquear distribuição" (→ T-064), "Iniciar reversão" (→ T-065).
- **Atualização:** Supabase Realtime. Badge "Ao vivo" no cabeçalho da tabela.
- **Responsividade:** Desktop: tabela completa. Mobile: cards por conta com status e valor principal.
- **Acessibilidade:** `aria-live="polite"` na tabela para atualizações em tempo real.

---

**T-061 | Financeiro — Drawer Detalhe da Conta Escrow**
- **Aplicação:** Admin Web
- **Trigger:** Clique em "Ver detalhe" em T-060
- **Descrição:** Drawer lateral direito (480px). Seções: Dados do caso (ID, imóvel, Cedente, Cessionário), Valores (breakdown: Cedente % | RS % como barras empilhadas horizontais + valores monetários), Histórico de movimentações (timeline cronológica), Ações disponíveis por perfil.
- **Responsividade:** Mobile: drawer fullscreen slide-up.

---

**T-062 | Financeiro — Comissões e Faturas**
- **Aplicação:** Admin Web
- **Rota:** `/financeiro/comissoes`
- **Descrição:** Tabela de faturas de comissão do RS. Colunas: ID fatura, ID caso, Valor, Status (Pendente / Distribuída / Cancelada), Data de criação, Data de distribuição.
- **Filtros:** Status, período, faixa de valor.
- **Export:** botão "Exportar CSV" / "Exportar PDF" no canto superior direito.
- **Responsividade:** Igual T-060.

---

**T-063 | Financeiro — Inadimplência**
- **Aplicação:** Admin Web
- **Rota:** `/financeiro/inadimplencia`
- **Descrição:** Painel de casos com depósito em atraso. Tabela: ID caso, valor esperado, dias em atraso, ações (por perfil).
- **Ações por perfil:**
  - *Gestor Financeiro:* "Notificar Cessionário" (re-dispara notificação), "Iniciar cancelamento" (→ modal de confirmação)
  - *Coordenador:* "Prorrogar prazo" (com justificativa)
  - *Master:* todas as ações

---

**T-064 | Modal — Bloquear Distribuição**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Gestor Financeiro
- **Trigger:** "Bloquear distribuição" em T-060
- **Descrição:** Modal. Campo justificativa obrigatório (mín. 20 caracteres, contador de caracteres visível). Aviso: "A solicitação será enviada ao Master para aprovação. A distribuição será suspensa imediatamente."
- **Ações:** "Solicitar bloqueio" (primário) | "Cancelar" (secondary)
- **Pós-ação:** notificação automática ao(s) Master(s). Toast "Solicitação de bloqueio enviada."

---

**T-065 | Modal — Iniciar Reversão (Estorno)**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Gestor Financeiro
- **Trigger:** "Iniciar reversão" em T-060
- **Descrição:** Modal. Exibe resumo: valor total depositado, partes envolvidas, implicações (sem comissão para RS). Campo confirmação: "Confirmo o estorno integral de R$ [valor] ao Cessionário [nome]." Checkbox obrigatório.
- **Ações:** "Confirmar reversão" (primário `--destructive`) | "Cancelar" (secondary)
- **Pós-ação:** Conta Escrow congelada. Comprovante gerado automaticamente no dossiê. Toast "Reversão iniciada. Comprovante gerado no dossiê."

---

**T-066 | Modal — Conciliação Bancária**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Gestor Financeiro, Master
- **Descrição:** Modal com upload de extrato bancário (PDF/CSV). Sistema faz matching automático com registros internos. Resultados: ✅ Conciliados / ⚠️ Divergências / ❓ Não identificados.
- **Ações:** "Exportar relatório de conciliação" | "Registrar divergência manualmente" | "Fechar"

---

### 2.8 Bloco: Supervisão IA

---

**T-070 | Supervisão IA — Painel Principal**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Coordenador, Master
- **Rota:** `/supervisao-ia`
- **Descrição:** Painel de monitoramento dos 2 agentes de IA: Guardião do Retorno e Analista de Oportunidades. Para cada agente: card com nome, status (Ativo / Pausado / Takeover Manual), métricas de desempenho (decisões nas últimas 24h, taxa de acerto), botões de ação.
- **Layout:** grid 2 colunas (um card por agente). Abaixo: log de decisões recentes (tabela unificada, últimas 20 entradas). Refresh automático a cada 10s via Supabase Realtime.
- **Estados:**
  - `agente ativo`: card com badge verde "Ativo"
  - `agente em takeover`: card com badge vermelho "Takeover Manual" + nome do operador que assumiu
  - `loading`: skeleton nos cards
- **Ações por agente:** "Ver log completo" → T-071. "Takeover manual" → T-072 (apenas Master).
- **Responsividade:** Desktop: 2 colunas. Mobile: 1 coluna, cards empilhados.

---

**T-071 | Supervisão IA — Log de Decisões do Agente**
- **Aplicação:** Admin Web
- **Rota:** `/supervisao-ia/:agente/log`
- **Descrição:** Tabela paginada de todas as decisões do agente selecionado. Colunas: timestamp, tipo de decisão, caso relacionado, confiança (%), resultado, feedback do operador (quando houver).
- **Filtros:** período, tipo de decisão, faixa de confiança.
- **Ação por linha:** "Revisar" → abre drawer com detalhes completos da decisão.
- **Export:** "Exportar CSV".

---

**T-072 | Modal — Takeover Manual de Agente**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Master
- **Trigger:** Clique em "Takeover manual" em T-070
- **Descrição:** Modal de confirmação. Exibe: nome do agente, impacto (lista de casos em que o agente está ativo). Campo justificativa obrigatório. Toggle "Pausar agente" ou "Assumir controle manualmente".
- **Ações:** "Confirmar takeover" (primário `--destructive`) | "Cancelar"

---

### 2.9 Bloco: Usuários

---

**T-075 | Usuários — Lista de Operadores**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Coordenador (leitura + edição básica), Master (tudo)
- **Rota:** `/usuarios/operadores`
- **Descrição:** Tabela de operadores internos. Colunas: avatar, nome, e-mail, perfil (badge), status (Ativo / Suspenso / Inativo), data de criação, casos ativos, última atividade. Botão "+ Novo operador" (Master only).
- **Ações por linha:** "Ver perfil" → T-076. Master: dropdown com "Editar", "Suspender", "Reatribuir casos".
- **Filtros:** perfil, status, busca por nome/e-mail.
- **Responsividade:** Desktop: tabela. Mobile: cards.

---

**T-076 | Usuários — Perfil do Operador**
- **Aplicação:** Admin Web
- **Rota:** `/usuarios/operadores/:id`
- **Descrição:** Página de perfil completo. Seções: dados pessoais, perfil de acesso, casos atribuídos (lista paginada), log de atividade recente, ações.
- **Ações:** Coordenador: "Atribuir caso". Master: + "Editar perfil", "Alterar perfil de acesso", "Suspender operador" (→ T-081), "Forçar redefinição de senha".

---

**T-077 | Usuários — Lista de Cedentes**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Coordenador, Master
- **Rota:** `/usuarios/cedentes`
- **Descrição:** Similar a T-075 para Cedentes externos. Colunas: nome, CPF (mascarado), e-mail, total de casos, status do usuário, data de cadastro.
- **Ações:** "Ver perfil" → T-078.

---

**T-078 | Usuários — Perfil do Cedente**
- **Aplicação:** Admin Web
- **Rota:** `/usuarios/cedentes/:id`
- **Descrição:** Perfil completo do Cedente. Seções: dados cadastrais (CPF, RG, endereço), casos vinculados, histórico de interações, documentos de identificação. Ações: "Editar dados cadastrais" (Coordenador+), "Suspender conta" → T-081.
- **Mascaramento de dados:** CPF exibido mascarado por padrão. Botão "Mostrar CPF" com registro de auditoria.

---

**T-079 | Usuários — Lista de Cessionários**
- **Aplicação:** Admin Web
- **Rota:** `/usuarios/cessionarios`
- **Descrição:** Igual T-077 para Cessionários.

---

**T-080 | Usuários — Perfil do Cessionário**
- **Aplicação:** Admin Web
- **Rota:** `/usuarios/cessionarios/:id`
- **Descrição:** Igual T-078 para Cessionários. Seção adicional: propostas enviadas (histórico).

---

**T-081 | Modal — Suspender / Reativar Usuário**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Coordenador (Cedentes/Cessionários), Master (todos)
- **Trigger:** Ação "Suspender" em T-076, T-078 ou T-080
- **Descrição:** Modal de confirmação. Exibe: nome do usuário, perfil, impacto (ex: "X casos ativos serão pausados"). Campo motivo obrigatório (select: Fraude suspeita / Inadimplência / Solicitação do usuário / Outro + textarea). Reativação: toggle para reverter.
- **Ações:** "Confirmar suspensão" (primário `--destructive`) | "Cancelar"

---

### 2.10 Bloco: Relatórios

---

**T-085 | Relatórios — Hub**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Coordenador (sem Receita), Gestor Financeiro (Receita + Auditoria), Master (tudo)
- **Rota:** `/relatorios`
- **Descrição:** Grid de cards de relatórios disponíveis. Cada card: ícone, nome do relatório, descrição curta, botão "Abrir". Relatórios sem permissão exibidos com overlay bloqueado e tooltip de permissão requerida.

---

**T-086 | Relatórios — SLA Operacional**
**T-087 | Relatórios — Volume de Casos**
**T-088 | Relatórios — Receita** *(Gestor Financeiro e Master only)*
**T-089 | Relatórios — Conversão**
**T-090 | Relatórios — Auditoria** *(Gestor Financeiro e Master only)*
**T-091 | Relatórios — Agentes de IA**

- **Aplicação:** Admin Web
- **Estrutura padrão de cada relatório:**
  - Filtros no topo: período (date range picker), agrupamento (diário / semanal / mensal), granularidade.
  - Gráfico principal (barras ou linha, tokens de cor `--chart-1` a `--chart-5`).
  - Tabela de dados abaixo do gráfico (paginada, ordenável).
  - Botões "Exportar CSV" e "Exportar PDF" no canto superior direito.
  - Estado `loading`: skeleton no gráfico e tabela. Estado `vazio`: ilustração + "Nenhum dado no período selecionado."
  - Agendamento de relatório: ícone de relógio → drawer de configuração de envio periódico por e-mail.
- **Responsividade:** Desktop: gráfico full-width + tabela. Mobile: gráfico substituído por KPIs numéricos; tabela em cards.

---

### 2.11 Bloco: Configurações

---

**T-095 | Configurações — Hub**
- **Aplicação:** Admin Web
- **Perfis que acessam:** Master only
- **Rota:** `/configuracoes`
- **Descrição:** Página de navegação para sub-seções de configuração. Lista de cards: Comissões e Parâmetros Financeiros, Prazos e SLAs, Templates de Notificação, Integrações, Usuários e Permissões.
- **Aviso de risco:** banner amarelo permanente: "Alterações aqui afetam toda a operação. Tenha cuidado e registre o motivo de cada alteração."

---

**T-096 | Configurações — Comissões e Parâmetros Financeiros**
- **Aplicação:** Admin Web
- **Rota:** `/configuracoes/comissoes`
- **Descrição:** Formulário de configuração. Campos: percentual de comissão do Cedente (range 5–50%, default 20%), percentual de comissão do Cessionário (range 5–50%, default 20%), limiar de revisão do Delta (valor em R$, default R$100.000), Valor Distrato Referência (percentual sobre pagamentos, default 50%).
- **Aviso:** "Alterações afetam apenas novos casos a partir desta data." Cada campo com helper text explicativo.
- **Audit trail:** toda alteração exige campo "Motivo da alteração" (obrigatório, mín. 20 chars) antes de salvar.
- **Ações:** "Salvar" (primário) | "Restaurar padrões" (secondary, com confirmação)

---

**T-097 | Configurações — Prazos e SLAs**
- **Aplicação:** Admin Web
- **Rota:** `/configuracoes/prazos`
- **Descrição:** Tabela de SLAs configuráveis por etapa do ciclo de vida. Colunas: etapa, SLA padrão (dias), SLA atual, última alteração, alterado por.
- **Inline edit:** clique no valor para editar inline. Botão "Salvar tudo" ao final.

---

**T-098 | Configurações — Templates de Notificação**
- **Aplicação:** Admin Web
- **Rota:** `/configuracoes/templates`
- **Descrição:** Lista de templates de notificação por canal (WhatsApp / E-mail / SMS). Para cada template: nome, canal, trigger event, pré-visualização do conteúdo, botão "Editar".
- **Editor de template:** drawer lateral com editor de texto rico (variáveis como `{{nome_cedente}}`, `{{link_acesso}}`). Botão "Testar envio" para envio de preview.

---

**T-099 | Configurações — Integrações**
- **Aplicação:** Admin Web
- **Rota:** `/configuracoes/integracoes`
- **Descrição:** Cards por integração (ZapSign, Celcoin, Meta Cloud API, Twilio). Cada card: nome, status (Conectado ✅ / Erro ⚠️ / Desconectado ❌), última sincronização, botão "Configurar" (abre drawer com campos de API key, webhook URL, etc.).
- **Segurança:** API keys exibidas mascaradas (****). Botão "Revelar" com log de auditoria.

---

**T-100 | Configurações — Usuários e Permissões**
- **Aplicação:** Admin Web
- **Rota:** `/configuracoes/usuarios`
- **Descrição:** Matriz de permissões editável por perfil e módulo. Toggle por célula. Botão "Salvar" exige confirmação: "Esta alteração afetará todos os usuários do perfil [X]."

---

## 3. Fluxos Cross-Módulo

### FLOW-001: Captação → Triagem → Qualificação → Oferta Ativa
```
[Cedente cadastra imóvel externo]
  → Sistema cria caso (estado: Captado)
  → Notificação à equipe de triagem
  → T-030 (Fila FIFO) → Analista seleciona caso
  → T-031 (Painel) + T-032 (Dossiê) + T-033 (Adimplência)
  → T-034 (Modal Bloqueio) [se inadimplente] → caso: Bloqueado
  → [Cedente regulariza] → caso: Em Triagem novamente
  → T-031 ação "Qualificar Caso" → caso: Qualificado
  → Coordenador em T-022 ação "Publicar Oferta" → caso: Oferta Ativa
```

### FLOW-002: Proposta → Negociação → Aceite → Formalização
```
[Cessionário faz proposta — módulo externo]
  → caso muda para Em Negociação
  → T-040 + T-041 + T-042 (Timeline)
  → T-043 (Modal Aceite) → caso: Em Formalização
  → Redirect automático para T-050
```

### FLOW-003: Formalização → Fechamento → Pós-Fechamento
```
T-051 (Painel) → preenche T-052, T-053, T-054
  → 4 critérios ✅ → botão "Confirmar Fechamento" habilitado
  → T-055 (Modal) → caso: Fechamento → Pós-Fechamento
  → Sistema inicia contagem de 15 dias
  → 15 dias sem reversão → caso: Concluído
  → Conta Escrow: distribuição automática
```

### FLOW-004: Reversão dentro dos 15 dias
```
T-060 (Financeiro — Escrow) → T-065 (Modal Reversão)
  → caso: Em Reversão → Em Mediação (se unilateral) / Cancelado
  → Conta Escrow: estorno integral ao Cessionário
```

### FLOW-005: Bloqueio de Distribuição
```
T-060 → T-064 (Modal Bloqueio) → notificação ao Master
  → Master aprova (T-010 alerta ou e-mail) → distribuição suspensa
  → Gestor resolve irregularidade → libera manualmente em T-060
```

---

## 4. Componentes Reutilizáveis

| **ID** | **Nome** | **Descrição** | **Telas que usa** |
|---|---|---|---|
| **C-01** | CaseCard | Card de caso para kanban (T-020) e listas | T-020, T-021 |
| **C-02** | StatusBadge | Badge colorido por estado do caso | Todas as telas com listagem de casos |
| **C-03** | CaseSidebar | Drawer lateral de detalhe do caso | T-022, T-061 |
| **C-04** | DocumentChecklist | Checklist de documentos com status individual | T-032, T-051 |
| **C-05** | TimelineLog | Timeline cronológica de eventos | T-042, T-071, T-022 (Tab Histórico) |
| **C-06** | SLAIndicator | Ícone/badge de SLA (verde/amarelo/vermelho) | T-020, T-021, T-030, T-040, T-050 |
| **C-07** | ConfirmModal | Modal de confirmação genérico reutilizável | T-034, T-043, T-044, T-055, T-064, T-065, T-072, T-081 |
| **C-08** | FilterChips | Grupo de chips filtrantes clicáveis | T-020, T-021, T-040, T-050, T-060 |
| **C-09** | EscrowBreakdownBar | Barras empilhadas de distribuição Cedente/RS | T-061 |
| **C-10** | ReportChart | Gráfico de barras/linha para relatórios | T-086 a T-091 |
| **C-11** | Topbar | Header global com logo, breadcrumb, notificações e avatar | Todas as telas logadas |
| **C-12** | Sidebar | Menu lateral colapsável com 10 módulos | Todas as telas logadas |
| **C-13** | NotificationBell | Ícone de sino com badge e dropdown de notificações | C-11 (Topbar) |
| **C-14** | AvatarDropdown | Avatar do operador com dropdown (perfil, logout) | C-11 (Topbar) |

---

## 5. Estados Globais de Interface

| **Estado** | **Trigger** | **Apresentação** |
|---|---|---|
| **Sessão expirada** | JWT expirado ou revogado | Modal "Sua sessão expirou. Faça login novamente." Redireciona para T-001. |
| **Sem permissão** | Acesso a módulo/ação não permitida pelo perfil | Toast "Você não tem permissão para executar esta ação." OU página bloqueada com overlay. |
| **Manutenção** | Flag de manutenção ativa no backend | Banner fixo no topo "Sistema em manutenção programada. Operações estão pausadas." |
| **Offline** | Perda de conexão detectada | Banner "Sem conexão — dados podem estar desatualizados." Toast quando restaura: "Conexão restabelecida." |
| **Erro 500** | Erro crítico do backend | Página de erro genérica com ID do erro e botão "Voltar ao início" + link para suporte. |

---

## 6. Convenções de Responsividade

| **Breakpoint** | **Valor** | **Comportamento da Sidebar** | **Comportamento do Conteúdo** |
|---|---|---|---|
| **Desktop** | ≥1280px | Expandida `w-64`, sempre visível | Grid completo, tabelas full |
| **Tablet** | 768–1279px | Colapsada `w-16` (ícones), hover para expandir | Grid simplificado, colunas secundárias ocultadas |
| **Mobile** | <768px | Oculta, acessível via botão hamburger → sheet de baixo | Layout em coluna única, tabelas → cards |

---

## 7. Convenções de Acessibilidade (a11y)

- **Focus management:** ao abrir modal/drawer, foco vai para o primeiro elemento interativo. Ao fechar, retorna ao elemento trigger.
- **Focus trap:** obrigatório em todos os modais (C-07 e variantes).
- **ARIA roles:** `role="dialog"` em modais, `role="table"` em tabelas, `role="list"` em listas, `role="status"` em KPIs dinâmicos, `role="alert"` em mensagens de erro.
- **Contraste:** mínimo WCAG AA (4.5:1 texto, 3:1 elementos de UI). `--primary #0069A8` sobre branco = 5.2:1 ✅.
- **Keyboard navigation:** Tab order lógico (top-to-bottom, left-to-right). Escape fecha modais/drawers. Enter/Space ativa botões.
- **Touch targets:** mínimo 44×44px em mobile para todos os elementos interativos.
- **Screen reader:** imagens decorativas com `alt=""`. Ícones de ação com `aria-label` descritivo.

---

## 8. Convenções de Microinteração

- **Transições de estado:** 150ms ease-in-out para mudanças de background/cor. 300ms para entrada/saída de drawers e modais.
- **Loading states:** skeleton loading (pulsante) em todo carregamento de dados. Spinner em botões de ação.
- **Toast notifications:** aparecem no canto inferior direito, duração 4s (erro: 6s, persistente até dismiss). Máximo 3 toasts simultâneos (pilha).
- **Confirmações destrutivas:** sempre via modal com campo de confirmação tipado (quando ação é irreversível).
- **Atualização em tempo real:** dados que mudam via Supabase Realtime usam fade-in sutil (200ms) no novo valor. Badge "Atualizado agora" visível por 3s.

---

## 9. Rastreabilidade RF → Tela

| **RF** | **Tela(s)** | **Módulo** |
|---|---|---|
| RF-001 a RF-003 | T-055 | Formalização — Fechamento |
| RF-004 a RF-010 | T-054, T-060, T-061, T-065 | Financeiro / Escrow |
| RF-011 a RF-020 | T-055, T-060, T-062 | Fechamento / Financeiro |
| RF-021 a RF-030 | T-060, T-061, T-064 | Financeiro |
| RF-031 a RF-044 | T-040 a T-045 | Negociação |
| RF-045 a RF-054 | T-050 a T-055 | Formalização |
| RF-055 a RF-065 | T-030 a T-034 | Triagem |
| RF-066 a RF-074 | T-001, T-002, T-075 a T-081 | Autenticação / Usuários |
| RF-075 a RF-086 | T-060 a T-066 | Financeiro (Gestão) |
| RF-087 a RF-094 | T-070 a T-072 | Supervisão IA |
| RF-095 a RF-108 | T-085 a T-091 | Relatórios |
| RF-109 a RF-120 | T-095 a T-100 | Configurações |
| RF-121 a RF-148 | T-001 a T-004, todas as telas logadas | Autenticação / Transversais |

---

*Documento gerado por Claude Code Desktop — Pipeline ShiftLabs v9.5 — 2026-03-22*
