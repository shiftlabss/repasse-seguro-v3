# S3 — Dashboard

## Módulo Admin — Repasse Seguro

| Campo                | Valor                               |
| -------------------- | ----------------------------------- |
| **Sprint**           | S3                                  |
| **Nome**             | Dashboard                           |
| **Tipo**             | Dinâmica — Módulo Fullstack         |
| **Template**         | B                                   |
| **Docs Consultados** | D01.1, D01.5, D06, D16              |
| **Cross-cutting**    | D10 (Glossário), D02 (Stacks)       |
| **REQs cobertos**    | REQ-077 a REQ-082, REQ-181, REQ-197 |
| **Total de itens**   | 40                                  |

---

> **Critério de conclusão de S3:** `GET /v1/dashboard` retorna KPIs corretos por perfil; Dashboard é a primeira tela após login; polling de 60s atualiza os dados automaticamente; modo offline exibe banner e timestamp dos dados em cache; estado vazio exibe card de onboarding; alertas ordenados por criticidade (SLA estourado → depósitos → IA → escalonamentos); nenhuma ação operacional é executável diretamente na Dashboard.

---

## ⚙️ BACKEND

### Feature: Endpoint de Dashboard

- [x] **[BE-01]** Implementar `GET /v1/dashboard` com RBAC por perfil (RN-009): retorna payload filtrado conforme o `role` do token; estrutura: `{ kpis: {...}, funnel: {...}, team_ranking: [...], activity_feed: [...], sla_alerts: [...] }`. Regras de filtragem:
  - **ANALISTA:** `kpis.my_active_cases`, `kpis.sla_expiring_today`, `kpis.cases_in_triagem`, `kpis.cases_in_negociacao`; `funnel` = funil dos casos atribuídos ao analista; `team_ranking` = posição do próprio analista; sem KPIs financeiros.
  - **COORDENADOR:** `kpis.total_active_cases`, `kpis.sla_overdue`, `kpis.active_analysts`, `kpis.cases_without_assignment`; `funnel` = funil geral; `team_ranking` = todos os analistas.
  - **GESTOR_FINANCEIRO:** `kpis.pending_escrows`, `kpis.value_in_custody`, `kpis.distributions_this_week`, `kpis.open_invoices`; sem funil de casos; `funnel` substituído por `distributions_chart`; sem `team_ranking`.
  - **MASTER:** todos os KPIs disponíveis; `funnel` geral; `team_ranking` completo.
  - Verificar: token ANALISTA não recebe `kpis.pending_escrows`; token GESTOR_FINANCEIRO não recebe `kpis.my_active_cases`.

- [x] **[BE-02]** Implementar `DashboardService.getKpis(userId, role)`: cada KPI calculado via query SQL otimizada; queries com timeout individual de 10 segundos (RN-011); se uma query ultrapassa 10s → retorna `{ error: true, message: "timeout" }` naquele KPI sem afetar os demais; KPIs gerais consultam tabelas `cases`, `escrow_accounts`, `commissions`; KPIs do ANALISTA filtram por `assigned_analyst_id = userId`. Verificar: falha de um KPI não derruba o endpoint; os demais KPIs retornam normalmente.

- [x] **[BE-03]** Implementar `DashboardService.getSlaAlerts(userId, role)`: busca alertas de SLA pendentes ordenados obrigatoriamente pela criticidade definida na RN-139: (1) `sla_overdue` (casos com `sla_deadline < NOW()`), (2) `pending_deposits` (escrows com prazo próximo), (3) `ai_alerts` (logs de agentes com `requires_action = true`), (4) `escalation_suggestions` (casos com escalonamento sugerido); omite categorias sem alertas ativos; cada alerta inclui `case_id` e `redirect_module` para o frontend navegar. Verificar: ordem obrigatória respeitada mesmo quando existe alerta de escalonamento mas não de SLA estourado.

- [x] **[BE-04]** Implementar `GET /v1/dashboard/activity-feed`: retorna feed cronológico de atividades recentes filtrado por perfil; ANALISTA: apenas eventos de seus casos; COORDENADOR/MASTER: todos os eventos; GESTOR_FINANCEIRO: apenas eventos financeiros; paginação cursor-based (`after=timestamp`); máximo 50 eventos por request. Verificar: ANALISTA não vê eventos de casos de outro analista.

- [x] **[BE-05]** Implementar cache Redis para Dashboard (RN-011/RN-132): chave `rs:dashboard:{userId}` com TTL de 60 segundos; na requisição → checar cache primeiro; se hit → retornar com header `X-Cache: HIT` e `X-Data-Age: {segundos}`; se miss → calcular, armazenar, retornar com `X-Cache: MISS`; invalidar cache manualmente via evento interno quando há mudança de estado de caso do usuário. Verificar: segunda requisição dentro de 60s retorna `X-Cache: HIT`; após 60s retorna `X-Cache: MISS`.

---

## 🖥️ FRONTEND

### Feature: T-010 — Dashboard Principal

- [x] **[FE-01]** Implementar `DashboardPage` (T-010, rota `/dashboard`) com layout grid 12 colunas (RN-009, D06 §2.2):
  - Zona superior: 4 cards de KPI (3 colunas cada em desktop).
  - Zona média: gráfico funil/distribuições (8 colunas) + ranking da equipe (4 colunas).
  - Zona inferior: tabela de atividades pendentes (8 colunas) + alertas SLA (4 colunas).
  - Responsividade: `≥1280px` = grid completo; `768–1279px` = sidebar colapsada, KPIs em grid 2x2, gráfico e ranking em coluna única; `<768px` = KPIs em lista vertical (1 por linha), gráfico simplificado (barra), ranking oculto, atividades limitadas a 5 + botão "Ver todas".
  - Verificar: layout correto nos 3 breakpoints; sidebar colapsa automaticamente em tablet.

- [x] **[FE-02]** Implementar variantes de KPIs por perfil na `DashboardPage` (RN-009, D06 §2.2 — tabela de 10 componentes × 4 perfis):
  - **ANALISTA:** `KpiCard` para "Meus casos ativos", "SLAs vencendo hoje", "Casos em triagem", "Casos em negociação"; gráfico = funil dos seus casos; ranking = sua posição.
  - **COORDENADOR:** `KpiCard` para "Casos ativos total", "SLAs vencidos", "Analistas ativos", "Casos sem atribuição"; gráfico = funil geral; ranking = todos os analistas.
  - **GESTOR_FINANCEIRO:** `KpiCard` para "Escrows pendentes", "Valor em custódia", "Distribuições esta semana", "Faturas em aberto"; sem gráfico de funil; `DistributionsChart` substituindo funil.
  - **MASTER:** todos os KPIs + toggle entre visão operacional e financeira.
  - Verificar: GESTOR_FINANCEIRO não renderiza funil de casos; MASTER renderiza toggle de visão.

- [x] **[FE-03]** Implementar 5 estados visuais da `DashboardPage` (D06 §2.2):
  - `loading`: skeleton em todas as zonas (animação `pulse`).
  - `loaded`: dados exibidos com `fade-in staggered` (100ms entre cada zona).
  - `erro parcial`: zona com falha exibe card de erro inline com botão "Tentar novamente" — demais zonas continuam normais; mensagem: "Não foi possível carregar este indicador. Tente novamente." (RN-011).
  - `offline`: banner superior fixo "Você está offline. Os dados exibidos podem estar desatualizados." + dados em cache exibidos com badge "cache" + timestamp de `X-Data-Age` (RN-011).
  - `primeiro acesso` (estado vazio — RN-012): todas as zonas com KPIs zerados em opacidade reduzida + gráficos com "Dados insuficientes" + card de boas-vindas central: "Bem-vindo ao Repasse Seguro Admin. Comece cadastrando o primeiro operador." + CTA "Criar primeiro operador" → `/usuarios` + CTA "Configurar parâmetros" → `/configuracoes`; card desaparece automaticamente quando o primeiro caso é captado.
  - Verificar: 5 estados renderizam corretamente; card de onboarding some ao primeiro caso.

- [x] **[FE-04]** Implementar `KpiCard` component: label + valor principal (tamanho grande) + variação percentual vs. mês anterior (seta verde ↑ ou vermelha ↓); `role="status"`; microinteração: contador animado ao carregar (easing `ease-out`, 600ms); hover: elevação sutil (`shadow-md`). Verificar: counter anima de 0 até o valor ao montar; acessibilidade `role="status"` presente.

- [x] **[FE-05]** Implementar `AlertsSlaPanel` component com priorização RN-139: renderiza alertas em ordem obrigatória: (1) casos com SLA estourado (badge vermelho), (2) depósitos pendentes próximos do prazo, (3) alertas de agentes de IA não resolvidos, (4) escalonamentos sugeridos; cada alerta clicável navega para: SLA estourado → `/pipeline`, depósito pendente → `/formalizacao`, alerta IA → `/supervisao-ia`, escalonamento → `/pipeline`; categorias sem alertas ativos são omitidas do painel. Verificar: clique em alerta de depósito navega para `/formalizacao`; categorias sem alertas não aparecem no DOM.

- [x] **[FE-06]** Implementar polling automático de 60 segundos na Dashboard (RN-011, RN-132): `useInterval` com intervalo 60000ms chamando `queryClient.invalidateQueries(['dashboard'])`; exibir `timestamp` da última atualização bem-sucedida no canto superior direito: "Atualizado às HH:MM:SS" (RN-132); filtro de período (Hoje / Esta semana / Este mês / Customizado) no canto superior direito passa como query param `period` para `GET /v1/dashboard`. Verificar: após 60s os dados são re-buscados automaticamente; timestamp atualiza a cada ciclo.

- [x] **[FE-07]** Implementar detecção de modo offline e banner (RN-011): listener `window.addEventListener('offline')` → exibe banner fixo no topo "Você está offline. Os dados exibidos podem estar desatualizados."; `window.addEventListener('online')` → remove banner e invalida queries para re-buscar automaticamente; dados em cache continuam exibidos com badge "cache" enquanto offline. Verificar: desconectar rede exibe banner; reconectar remove banner e re-busca dados.

- [x] **[FE-08]** Implementar Dashboard como tela inicial obrigatória (RN-010): `AuthGuard` redireciona para `/dashboard` após login bem-sucedido (não para outra rota); nenhuma ação operacional disponível na Dashboard — todos os CTAs redirecionam para módulos específicos; nenhuma configuração de usuário permite alterar a tela inicial. Verificar: login bem-sucedido sempre redireciona para `/dashboard`; nenhum botão de ação operacional (aprovar, cancelar, fechar) existe no DOM da Dashboard.

- [x] **[FE-09]** Implementar acessibilidade da Dashboard (D06 §2.2): gráficos com `aria-label` descritivo; KPI cards com `role="status"`; tab order lógico: KPIs → gráfico → atividades pendentes → alertas SLA; AlertsSlaPanel com `aria-live="polite"` para atualizações automáticas. Verificar: navegação por teclado segue a ordem definida; screen reader anuncia mudanças no painel de alertas.

---

## 🔗 WIRING

- [x] **[WIRE-01]** Implementar `dashboard.service.ts` no frontend: `getDashboardData(period: string)` → `GET /v1/dashboard?period={period}`; `getActivityFeed(after?: string)` → `GET /v1/dashboard/activity-feed?after={after}`; TanStack Query com `staleTime: 55000` (55s, ligeiramente abaixo do TTL de 60s do Redis); `queryKey: ['dashboard', period, userId]`. Verificar: mudança de período re-executa a query; cache TanStack não serve dados expirados.

- [x] **[WIRE-02]** Integrar `dashboard.service.ts` com `DashboardPage`: `useQuery(['dashboard', period])` no componente; `isLoading` → estado `loading` (skeleton); `isError` → estado `erro parcial` por zona; `data` → renderiza KPIs e gráficos; `isOffline` (via `navigator.onLine`) → estado `offline` com dados do `queryClient.getQueryData`. Verificar: desconexão de rede não quebra a página; dados anteriores continuam visíveis com badge "cache".

---

## ✅ TESTES

- [x] **[TEST-01]** Testes unitários `DashboardService` (backend): `getKpis` para cada um dos 4 perfis retorna apenas os KPIs permitidos; timeout de 10s em query individual não derruba os demais KPIs; `getSlaAlerts` respeita a ordem de criticidade da RN-139; ANALISTA não recebe dados de outros analistas. Cobertura: 80% de branches.

- [x] **[TEST-02]** Testes unitários componentes frontend: `KpiCard` anima contador de 0 ao valor; `AlertsSlaPanel` ordena alertas na ordem RN-139; estado `offline` renderiza banner; estado `primeiro acesso` renderiza card de onboarding com links corretos.

- [x] **[TEST-03]** Testes de integração `GET /v1/dashboard`: banco real (testcontainers); token ANALISTA retorna apenas KPIs de ANALISTA; token GESTOR_FINANCEIRO retorna KPIs financeiros e `DistributionsChart`; segundo request em 60s retorna `X-Cache: HIT`; request após 60s retorna `X-Cache: MISS`.

- [x] **[TEST-04]** Teste de isolamento de dados Dashboard: ANALISTA com 3 casos atribuídos vê `my_active_cases = 3`; ANALISTA não vê os 5 casos de outro analista; COORDENADOR vê todos os 8 casos em `total_active_cases`.

---

## 🔍 AUTO-VERIFICAÇÃO S3 (12 checks)

- [x] **[CHECK-01]** `GET /v1/dashboard` com token ANALISTA retorna KPIs de ANALISTA e não retorna `pending_escrows` nem `value_in_custody`
- [x] **[CHECK-02]** `GET /v1/dashboard` com token GESTOR_FINANCEIRO retorna KPIs financeiros e não retorna funil de casos nem `my_active_cases`
- [x] **[CHECK-03]** `GET /v1/dashboard` com token MASTER retorna todos os KPIs e toggle entre visão operacional e financeira
- [x] **[CHECK-04]** Polling automático de 60 segundos re-busca dados na Dashboard (verificar via Network DevTools)
- [x] **[CHECK-05]** Cache Redis `rs:dashboard:{userId}` com TTL 60s funciona: segunda request em 60s retorna `X-Cache: HIT`
- [x] **[CHECK-06]** Modo offline: desconectar rede exibe banner "Você está offline" e dados em cache com badge "cache"
- [x] **[CHECK-07]** Estado vazio (onboarding): sem casos cadastrados, card "Bem-vindo" aparece com CTAs "Criar primeiro operador" e "Configurar parâmetros"
- [x] **[CHECK-08]** Alertas na ordem RN-139: SLA estourado primeiro, depois depósitos, depois alertas IA, depois escalonamentos
- [x] **[CHECK-09]** Nenhuma ação operacional disponível na Dashboard (verificar ausência de botões "Aprovar", "Cancelar", "Fechar" no DOM)
- [x] **[CHECK-10]** Login bem-sucedido sempre redireciona para `/dashboard` independente do perfil
- [x] **[CHECK-11]** Falha individual de um KPI (timeout 10s) não derruba os demais KPIs nem o endpoint
- [x] **[CHECK-12]** Todos os REQs de S3 (REQ-077–082, REQ-181, REQ-197) cobertos por pelo menos 1 item de checklist
