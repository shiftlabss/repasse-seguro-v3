# 11 - Mobile

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Produto, Frontend e Design |
| **Escopo** | Estratégia mobile do CRM: decisão de plataforma, requisitos de responsividade, breakpoints, funcionalidades por dispositivo e roadmap pós-MVP |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 (America/Fortaleza) |
| **Dependências** | 06 - Mapa de Telas · 01.1 Regras de Negócio (Fundação e Acessos) · 02 Stacks |

---

> **TL;DR**
>
> - **O CRM NÃO tem app mobile dedicado no MVP.** [DECISÃO AUTÔNOMA — justificada na seção 2]
> - **A aplicação web é responsiva a partir de 768px** (tablet) via Next.js 15 App Router + Tailwind CSS.
> - **Mobile < 640px está fora do escopo do MVP** — sem contrato de responsividade para smartphones.
> - **4 funcionalidades críticas de campo** disponíveis e testadas em tablet (768–1279px).
> - **Roadmap:** app mobile nativo para Fase 2 (pós-validação operacional do CRM web).

---

## 1. Contexto e Perfil de Uso

### 1.1 Quem usa o CRM e em que dispositivo

O CRM da Repasse Seguro é uma **ferramenta de trabalho interno** operada por:

| **Perfil** | **Cenário de uso principal** | **Dispositivo esperado** |
|---|---|---|
| Analista RS | Gestão de casos, registro de atividades, comunicação — fluxo contínuo de trabalho diário | Desktop (escritório) |
| Coordenador RS | Supervisão de equipe, redistribuição de casos, aprovação de dossiês | Desktop (escritório) / tablet (reuniões) |
| Admin RS | Configurações, relatórios, gestão de usuários | Desktop exclusivo |
| Parceiro Externo | Consulta de status de casos — acesso eventual | Desktop ou smartphone (portal externo — fora deste escopo) |

### 1.2 Padrão de uso observado

- Analistas RS operam **sessões longas de trabalho** (3–8h/dia) com múltiplas abas e formulários complexos.
- O fluxo de trabalho exige **entrada de dados extensiva** (formulários de caso, registro de atividades, upload de documentos) — tarefas inadequadas para interface mobile.
- O único caso de uso de campo real é o **Coordenador RS em visitas a Analistas ou reuniões de equipe**, acessando o tablet para acompanhar métricas ou redistribuir casos.

---

## 2. Decisão de Plataforma — MVP

> **[DECISÃO AUTÔNOMA — CRM sem app mobile dedicado no MVP]**
>
> **Decisão:** O CRM Repasse Seguro não terá aplicativo mobile dedicado (React Native, Flutter ou PWA instalável) no MVP.
>
> **Justificativa:**
>
> 1. **Perfil de uso interno em escritório:** Analistas RS e Coordenadores RS trabalham em ambiente de escritório com acesso a desktop. O CRM é uma ferramenta de produtividade profissional com fluxos complexos — criação de caso, upload de dossiê, registro de atividades retroativas — que não se traduzem bem para telas <6 polegadas.
>
> 2. **Custo de desenvolvimento sem retorno proporcional:** desenvolver e manter um app mobile nativo ou PWA para um público interno de ~15–50 usuários no MVP representaria 30–40% do esforço de frontend sem proporção de valor. Esses recursos são melhor alocados em qualidade e cobertura de funcionalidades no web.
>
> 3. **Sem caso de uso de campo crítico no MVP:** a Repasse Seguro opera em modelo assistido com profissionais em escritório. Não há cenário de uso de campo (visita a imóveis, cadastro on-site) no MVP que exija app mobile.
>
> 4. **Tablet como caso de uso secundário real:** Coordenadores RS em reuniões de equipe podem precisar acessar métricas ou redistribuir casos via tablet. Esse cenário é coberto por responsividade web ≥768px sem necessidade de app dedicado.
>
> 5. **Referência de mercado:** CRMs corporativos internos de complexidade equivalente (HubSpot interno, Salesforce para equipes pequenas) priorizam web first e lançam mobile em fases posteriores após validar os fluxos críticos.
>
> **Revisão obrigatória:** a decisão deve ser reavaliada antes do início da Fase 3 (pós-MVP) com base em dados de uso coletados nos primeiros 90 dias de operação. Métricas-gatilho para reavaliação: >20% dos acessos por dispositivo móvel OU pedido formal de mais de 3 Analistas RS de funcionalidade mobile.

---

## 3. Estratégia de Responsividade

### 3.1 Abordagem

**Progressive enhancement + Mobile-first CSS** via Tailwind CSS. O layout base é definido para telas menores e progressivamente expandido com breakpoints (`md:`, `lg:`).

### 3.2 Breakpoints

| **Breakpoint** | **Tailwind prefix** | **Faixa de resolução** | **Comportamento do CRM** |
|---|---|---|---|
| **Mobile** | base (sem prefix) | < 640px | Fora do escopo MVP — sem contrato |
| **Mobile grande** | `sm:` | 640–767px | Fora do escopo MVP — sem contrato |
| **Tablet** | `md:` | 768–1023px | Responsivo — funcionalidades críticas disponíveis |
| **Tablet grande** | `lg:` | 1024–1279px | Responsivo — sidebar colapsada (ícones) |
| **Desktop** | `xl:` | ≥ 1280px | Layout completo — primário |

### 3.3 Comportamento da Sidebar por Breakpoint

| **Breakpoint** | **Estado da sidebar** | **Comportamento** |
|---|---|---|
| ≥ 1280px | Expandida (`w-64`) | Labels + ícones visíveis |
| 1024–1279px | Colapsada (`w-16`) | Apenas ícones, tooltip com label no hover |
| 768–1023px | Colapsada (`w-16`) | Apenas ícones, toque abre tooltip/label |
| < 768px | Oculta | Hamburguer menu (fora do escopo MVP) |

### 3.4 Adaptações de Layout por Breakpoint

| **Componente** | **Desktop ≥1280px** | **Tablet 768–1279px** |
|---|---|---|
| Pipeline Kanban | Scroll horizontal, colunas `min-w-[280px]` | Scroll horizontal via `ScrollArea`, colunas `min-w-[240px]` |
| Tabelas (DataTable) | Todas as colunas visíveis | Colunas secundárias ocultas com `hidden md:table-cell` |
| Formulários | Layout 2 colunas (`grid-cols-2`) | Layout 1 coluna (`grid-cols-1`) |
| Cards de KPI (Dashboard) | Grid 4 colunas | Grid 2 colunas |
| Modal / Dialog | `max-w-xl` centralizado | `w-full mx-4` (quase fullscreen) |
| Topbar | Busca global visível | Busca colapsada em ícone |

---

## 4. Funcionalidades por Dispositivo

### 4.1 Funcionalidades disponíveis em tablet (768px+)

As seguintes funcionalidades são **obrigatoriamente** funcionais e testadas em resolução de tablet (768px):

| **Funcionalidade** | **Tela** | **Prioridade** | **Justificativa** |
|---|---|---|---|
| Registro de Atividade / Follow-up | T-CRM-061 | Crítica | Coordenador RS em reunião pode precisar registrar atividade on-the-go |
| Avanço de estado do Caso | T-CRM-022 | Crítica | Decisão operacional que pode ocorrer fora do desktop |
| Upload de documento no Dossiê | T-CRM-040 | Crítica | Analista RS pode receber documento em campo e fazer upload imediato |
| Envio de mensagem WhatsApp | T-CRM-070 | Crítica | Resposta a contato pode ser urgente |
| Visualização do Pipeline Kanban | T-CRM-020 | Alta | Coordenador RS em reunião de equipe |
| Visualização do SLA Monitor | T-CRM-080 | Alta | Coordenador RS monitorando equipe remotamente |
| Consulta de Perfil do Contato | T-CRM-051 | Média | Verificação rápida antes de uma ligação |
| Reconhecer alerta de SLA | T-CRM-080 | Alta | Ação urgente que não pode esperar desktop |

### 4.2 Funcionalidades desktop-only (não disponíveis em tablet)

As seguintes funcionalidades são **exclusivas de desktop** por complexidade de interação ou raridade de uso em campo:

| **Funcionalidade** | **Tela** | **Motivo** |
|---|---|---|
| Relatórios Financeiro e Inteligência de Mercado | T-CRM-103, T-CRM-104 | Análise profunda requer tela grande; uso exclusivo de Admin RS em escritório |
| Configurações — Parâmetros do Sistema | T-CRM-110 | Alterações de parâmetros críticos requerem atenção e validação — não devem ser feitas em tablet |
| Configurações — Log de Auditoria | T-CRM-112 | Análise forense de logs — uso em desktop exclusivo |
| Gestão de usuários (Convidar / Suspender / Desligar) | T-CRM-093 | Ação de alto impacto — requer confirmação explícita em ambiente controlado |
| Criação de Template de Mensagem | T-CRM-111 | Edição de template exige atenção a formatação — incompatível com tablet |
| Drag-and-drop no Kanban | T-CRM-020 | Interação complexa — roadmap futuro para touch |

### 4.3 Fora do escopo MVP (mobile <768px)

Sem contrato de responsividade para smartphones. Tentar acessar o CRM em smartphone ≤640px exibirá um banner informativo:

> "O CRM Repasse Seguro foi projetado para uso em desktop ou tablet. Para melhor experiência, acesse de um dispositivo com tela ≥ 768px."

O acesso não é bloqueado tecnicamente, mas sem responsividade garantida.

---

## 5. Requisitos de Touch para Tablet

Para garantir usabilidade em tablets com interface touch:

| **Requisito** | **Especificação** | **Componente afetado** |
|---|---|---|
| Tamanho mínimo de área clicável | 44×44px (WCAG 2.1 critério 2.5.5) | Botões, badges, links de tabela |
| Espaçamento entre elementos interativos | Mínimo 8px entre elementos adjacentes | Listas, menus dropdown |
| Scroll horizontal | `overflow-x: auto` com scroll touch | Pipeline Kanban, tabelas largas |
| Formulários com teclado virtual | `scrollIntoView` ao focar campo para não ocultar com teclado | Todos os formulários modais |
| Pinch-to-zoom não bloqueado | `<meta name="viewport" content="width=device-width, initial-scale=1">` — sem `user-scalable=no` | Layout global |

---

## 6. Testes de Responsividade

### 6.1 Dispositivos de referência para testes

| **Dispositivo** | **Resolução** | **Obrigatório para MVP** |
|---|---|---|
| iPad Air (10.9") | 820×1180px (retrato) | Sim |
| iPad Pro (11") | 834×1194px (retrato) | Sim |
| iPad Pro (12.9") | 1024×1366px (paisagem) | Sim |
| MacBook Air 13" | 1280×800px | Sim (desktop mínimo) |
| Desktop 1440px | 1440×900px | Sim (desktop padrão) |

### 6.2 Critérios de aceite para responsividade

1. As 8 funcionalidades críticas de campo (seção 4.1) funcionam sem erros em 820×1180px.
2. Nenhum elemento interativo tem área clicável < 44×44px em viewport 820px.
3. Textos não são truncados de forma que comprometa o significado nas 4 funcionalidades críticas.
4. Formulários modais (T-CRM-061, T-CRM-032, T-CRM-033) são completamente utilizáveis com teclado virtual ativo.
5. Pipeline Kanban rola horizontalmente com scroll touch sem perda de dados visíveis.

---

## 7. Roadmap Mobile Pós-MVP

### 7.1 Fase 2 — App Mobile Nativo (revisão em Q3 2026)

**Gatilho:** >20% dos acessos por dispositivo móvel OU pedido formal de ≥3 Analistas RS.

**Escopo preliminar para avaliação:**

| **Módulo** | **Prioridade estimada** | **Justificativa** |
|---|---|---|
| Dashboard pessoal (Analista RS) | Alta | Visão rápida de casos e follow-ups do dia |
| Registro de Atividade (T-CRM-061) | Alta | Uso frequente de campo |
| Notificações push (SLA, follow-up) | Alta | Substituiria notificações por e-mail |
| Avanço de estado do Caso | Média | Depende do perfil de uso coletado |
| Pipeline (somente leitura) | Média | — |
| Comunicação WhatsApp | Baixa | Conflito com WhatsApp pessoal — avaliar UX |

**Tecnologia sugerida para avaliação:** React Native (Expo) — reutiliza lógica de negócio TypeScript compartilhada via monorepo Turborepo (`packages/shared`).

### 7.2 Fase 2 — PWA (alternativa mais rápida)

Se o app nativo for deferido, uma Progressive Web App (PWA) com funcionalidades de cache offline e notificações push pode ser entregue com menor esforço. Avaliação técnica necessária na Fase 2.

---

## 8. Referência de Implementação

### 8.1 Configuração do Tailwind (`tailwind.config.ts`)

```typescript
// Breakpoints padrão usados no CRM
// md: 768px  → tablet mínimo (funcionalidades críticas)
// lg: 1024px → tablet grande (sidebar colapsada)
// xl: 1280px → desktop padrão
// 2xl: 1536px → desktop grande

theme: {
  screens: {
    sm: '640px',
    md: '768px',
    lg: '1024px',
    xl: '1280px',
    '2xl': '1536px',
  }
}
```

### 8.2 Padrão de ocultação condicional

```tsx
// Coluna de tabela visível apenas em desktop
<TableHead className="hidden xl:table-cell">Empreendimento</TableHead>

// Sidebar expandida apenas em xl+
<aside className="w-16 xl:w-64 transition-all duration-200">

// Grid de KPIs adaptativo
<div className="grid grid-cols-2 xl:grid-cols-4 gap-4">
```

### 8.3 Meta tag obrigatória

```html
<!-- Em layout.tsx raiz -->
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

---

## 9. Controle de Versão

| **Versão** | **Data** | **Responsável** | **Alteração** |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — decisão de plataforma MVP, breakpoints, funcionalidades por dispositivo |
