# Registro Mestre de Requisitos — AI-Dani-Admin

| Campo | Valor |
|---|---|
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Data | 2026-03-24 |
| Total de requisitos | 220 |
| Gerado por | Pipeline Fase 3 v2.3 |

---

> **Fonte da verdade para cobertura 100%.** Cada requisito atômico extraído de cada doc (D01–D29). A coluna Sprint indica a sprint onde o requisito será implementado.

---

## Legenda de Tipos

| Tipo | Descrição |
|---|---|
| RN | Regra de negócio |
| RF | Requisito funcional |
| RNF | Requisito não funcional |
| BANCO | Tabela, campo, enum, índice |
| ENDPOINT | Endpoint da API REST |
| TELA | Tela ou componente de UI |
| INTEG | Integração externa |
| ENV | Variável de ambiente |
| NOTIF | Template de notificação |
| TESTE | Caso de teste |
| INFRA | Infraestrutura, setup, deploy |
| OBS | Observabilidade, log, métrica |
| GLOSS | Item exclusivo do glossário |
| GOLIVЕ | Item do go-live / qualidade |

---

## Tabela de Requisitos

| ID | Doc Fonte | Seção | Tipo | Descrição | Valor/Detalhe | Sprint |
|----|-----------|-------|------|-----------|---------------|--------|
| REQ-001 | D01 | 1 | GLOSS | Glossário: Admin | Equipe operacional da plataforma Repasse Seguro | CROSS |
| REQ-002 | D01 | 1 | GLOSS | Glossário: CSAT | Customer Satisfaction Score, escala 1 a 5 | CROSS |
| REQ-003 | D01 | 1 | GLOSS | Glossário: Nível de confiança | Percentual 0–100% de certeza do agente | CROSS |
| REQ-004 | D01 | 1 | GLOSS | Glossário: Takeover | Intervenção manual do Admin, threshold padrão 80% | CROSS |
| REQ-005 | D01 | 1 | GLOSS | Glossário: Threshold de confiança | Configurável 50%–95%, padrão 80% | CROSS |
| REQ-006 | D01 | 1 | GLOSS | Glossário: Taxa de recusa | % de respostas recusadas por restrição de perfil | CROSS |
| REQ-007 | D01 | 1 | GLOSS | Glossário: DesligadoAutomatico | Estado quando taxa de erro > 30% em 15 min, reativação manual | CROSS |
| REQ-008 | D01 | 1 | GLOSS | Glossário: FallbackAtivo | Estado quando API do modelo indisponível | CROSS |
| REQ-009 | D01 | 3.1 | RN | RBAC: Matriz de permissões do Admin (10 operações) | Admin permite tudo; Cessionário/Cedente bloqueados | S2 |
| REQ-010 | D01 | 4.2 | RN | Máquina de estados de interação — transição [*] → RespondidaPelaIA | Confiança >= threshold | S3 |
| REQ-011 | D01 | 4.2 | RN | Máquina de estados de interação — transição [*] → SinalizadaParaRevisao | Confiança < threshold | S3 |
| REQ-012 | D01 | 4.2 | RN | Máquina de estados de interação — transição SinalizadaParaRevisao → EmTakeover | Admin inicia takeover | S4 |
| REQ-013 | D01 | 4.2 | RN | Máquina de estados de interação — transição SinalizadaParaRevisao → RespondidaPelaIA | Admin não intervém; agente continua | S4 |
| REQ-014 | D01 | 4.2 | RN | Máquina de estados de interação — transição RespondidaPelaIA → EmTakeover | Admin inicia takeover manual | S4 |
| REQ-015 | D01 | 4.2 | RN | Máquina de estados de interação — transição RespondidaPelaIA → Encerrada | Usuário encerra conversa | S3 |
| REQ-016 | D01 | 4.2 | RN | Máquina de estados de interação — transição EmTakeover → Encerrada | Admin encerra takeover | S4 |
| REQ-017 | D01 | 4.2 | RN | Máquina de estados de interação — transição Encerrada → [*] | Estado final | S3 |
| REQ-018 | D01 | RN-DA-030 | RN | Monitoramento: lista com 8 campos por interação (userId anonimizado, timestamp, pergunta, resposta, confiança, dados usados, latência, agente) | Campos obrigatórios exatos | S3 |
| REQ-019 | D01 | RN-DA-030 | RN | Estado vazio: "Nenhuma interação registrada no período selecionado." + sugestão | Texto exato definido | S3 |
| REQ-020 | D01 | RN-DA-030 | RN | Filtros: por data, usuário, nível de confiança, agente | Chips removíveis individualmente + "Limpar filtros" | S3 |
| REQ-021 | D01 | RN-DA-030 | RN | Skeleton loading inline ao filtrar (não substituir tela inteira) | Inline, não full-page | S3 |
| REQ-022 | D01 | RN-DA-030 | RN | Log de acesso ao painel registrado em admin_access_logs | Admin ID + timestamp + filtros | S3 |
| REQ-023 | D01 | RN-DA-031 | RN | Alerta: Latência alta — tempo de resposta acima do SLA por 5 min consecutivos → Slack + painel | ⚠️ AMBÍGUO (RM-001): LATENCY_SLA_SECONDS indefinido | S5 |
| REQ-024 | D01 | RN-DA-031 | RN | Alerta: Taxa de erro elevada — > 10% em 15 min → Slack + e-mail Admin | 10%, 15min | S5 |
| REQ-025 | D01 | RN-DA-031 | RN | Alerta: Desligamento automático — taxa de erro > 30% em 15 min → Slack + e-mail + painel | 30%, 15min | S5 |
| REQ-026 | D01 | RN-DA-031 | RN | Alerta: CSAT degradado — média < 3,5/5 nas últimas 24h → painel + e-mail | 3.5, 24h | S5 |
| REQ-027 | D01 | RN-DA-031 | RN | Alerta: Taxa de recusa alta — > 20% em 24h → painel Admin | 20%, 24h | S5 |
| REQ-028 | D01 | RN-DA-031 | RN | Alerta: Consumo de processamento — > 80% do orçamento mensal → e-mail Admin | 80% | S5 |
| REQ-029 | D01 | RN-DA-031 | RN | Múltiplos alertas simultâneos: prioridade (desligamento primeiro, latência segundo) | Ordenação de prioridade definida | S5 |
| REQ-030 | D01 | RN-DA-032 | RN | Sinalização automática: confiança < threshold → habilitar botão takeover | Threshold default 80% | S3 |
| REQ-031 | D01 | RN-DA-032 | RN | Takeover manual: botão sempre disponível acima do threshold (por qualquer motivo) | Qualquer interação ativa | S4 |
| REQ-032 | D01 | RN-DA-033 | RN | Execução do takeover: registrar com timestamp, motivo e Admin ID | 3 campos obrigatórios | S4 |
| REQ-033 | D01 | RN-DA-033 | RN | Mensagem ao usuário no takeover: "Um analista da equipe Repasse Seguro assumiu essa conversa para ajudá-lo. Como posso ajudar?" | Texto exato | S4 |
| REQ-034 | D01 | RN-DA-033 | RN | Avatar diferenciado + "Equipe Repasse Seguro" + separador "Atendimento humano" no takeover | 3 elementos visuais | S4 |
| REQ-035 | D01 | RN-DA-033 | RN | Agente para de responder automaticamente durante takeover | Agente pausado para a sessão | S4 |
| REQ-036 | D01 | RN-DA-033 | RN | Campo de entrada do usuário permanece ativo durante takeover | Sem interrupção para o usuário | S4 |
| REQ-037 | D01 | RN-DA-033 | RN | Encerramento de takeover: mensagem "Você está novamente em atendimento com [nome do agente]." | Texto exato + separador + avatar | S4 |
| REQ-038 | D01 | RN-DA-033 | RN | Takeover de conversa encerrada: sistema bloqueia com "Conversa encerrada" | Edge case bloqueado | S4 |
| REQ-039 | D01 | RN-DA-033 | RN | Takeover simultâneo: primeiro Admin confirma; segundo recebe "Esta conversa já está em atendimento por outro analista." | Lock otimista | S4 |
| REQ-040 | D01 | RN-DA-034 | RN | Dashboard: 5 indicadores — volume de interações, top 10 perguntas, taxa de recusa, CSAT médio, tempo médio de resposta | 5 indicadores exatos | S6 |
| REQ-041 | D01 | RN-DA-034 | RN | Dashboard: filtro por período (dia, semana, mês) e por agente (Dani-Cessionário, Dani-Cedente) | 2 filtros | S6 |
| REQ-042 | D01 | RN-DA-034 | RN | Dashboard: indicador sem dados → "Dados insuficientes para o período selecionado" com ícone; NUNCA exibir "0" ou "0%" | Regra de display | S6 |
| REQ-043 | D01 | RN-DA-034 | RN | Dashboard: métricas atualizadas em tempo real (Supabase Realtime) | Tempo real | S6 |
| REQ-044 | D01 | RN-DA-035 | RN | Configuração: threshold configurável 50%–95%, padrão 80% | Range exato | S5 |
| REQ-045 | D01 | RN-DA-035 | RN | Validação de threshold: < 50% ou > 95% → erro inline: "O nível de supervisão precisa estar entre 50% e 95%. Valores fora desse intervalo podem comprometer a qualidade do atendimento." | Texto exato | S5 |
| REQ-046 | D01 | RN-DA-035 | RN | Salvar threshold: toast "Nível de supervisão atualizado para [valor]%." | Texto exato | S5 |
| REQ-047 | D01 | RN-DA-035 | RN | Log de auditoria de alteração de threshold: "Alterado de [anterior]% para [novo]% por [Admin] em [data/hora]" | 4 campos | S5 |
| REQ-048 | D01 | RN-DA-035 | RN | Novo threshold entra em vigor imediatamente para novas interações | Vigência imediata | S5 |
| REQ-049 | D01 | RN-DA-036 | RN | Webchat disponível 24/7 dependendo da API do modelo | Sem horário fixo | S7 |
| REQ-050 | D01 | RN-DA-036 | RN | API indisponível: "O agente está temporariamente indisponível. Os cálculos de comissão e Escrow continuam disponíveis. Tente novamente em instantes." | Texto exato | S7 |
| REQ-051 | D01 | RN-DA-036 | RN | FAB global: badge amarelo quando API do modelo indisponível (status degradado visível antes de abrir chat) | Badge amarelo | S7 |
| REQ-052 | D01 | RN-DA-036 | RN | Rate limit webchat: 30 mensagens por hora por usuário, janela deslizante | 30 msg/h | S7 |
| REQ-053 | D01 | RN-DA-036 | RN | Histórico de conversa: persistência por 90 dias | 90 dias | S7 |
| REQ-054 | D01 | RN-DA-037 | RN | Gate de lançamento: filtro de escopo — toda consulta valida recurso pertence ao usuário autenticado | Pré-requisito de ativação | S8 |
| REQ-055 | D01 | RN-DA-037 | RN | Gate de lançamento: filtro de contexto — dados do agente apenas para o perfil autenticado | Pré-requisito de ativação | S8 |
| REQ-056 | D01 | RN-DA-037 | RN | Gate de lançamento: teste de penetração — acesso cruzado bloqueado em 100% dos casos | 100% | S8 |
| REQ-057 | D01 | RN-DA-037 | RN | Ativação bloqueada se qualquer item de isolamento falhar | Gate obrigatório | S8 |
| REQ-058 | D01 | RN-DA-038 | RN | Instrução permanente: 4 itens obrigatórios (identidade/tom, dados bloqueados, exemplos de recusa, formato de resposta) | 4 itens exatos | S7 |
| REQ-059 | D01 | RN-DA-038 | RN | Testes adversariais: mínimo 20 perguntas antes do lançamento | 20 testes | S8 |
| REQ-060 | D01 | RN-DA-039 | RN | Gate supervisão: 4 componentes obrigatórios (registro de interações, dashboard, alerta de confiança, takeover manual) | 4 componentes | S8 |
| REQ-061 | D01 | RN-DA-039 | RN | Lançamento bloqueado se qualquer componente de supervisão ausente | Gate obrigatório | S8 |
| REQ-062 | D01 | 10 | RN | Edge case: threshold 100% → sistema recusa; mensagem de limite máximo 95% | RF-016 | S5 |
| REQ-063 | D01 | 10 | RN | Edge case: takeover de conversa encerrada → sistema bloqueia | RF-010 | S4 |
| REQ-064 | D01 | 10 | RN | Edge case: CSAT + taxa recusa simultâneos → ambos alertas disparados | RF-005, RF-006 | S5 |
| REQ-065 | D01 | 10 | RN | Edge case: dashboard sem dados → "Dados insuficientes" por indicador (não zeros) | RF-013 | S6 |
| REQ-066 | D01 | 10 | RN | Edge case: taxa de erro > 30% em 15 min → desligamento automático + notificação | RN-DA-031 | S5 |
| REQ-067 | D01 | 2.2 | RN | Estado Agente — máquina de estados: [*] → Ativo (ativação autorizada RF-020, RF-024) | Transição de ativação | S8 |
| REQ-068 | D01 | 2.2 | RN | Estado Agente — Ativo → FallbackAtivo (API do modelo indisponível) | Transição fallback | S7 |
| REQ-069 | D01 | 2.2 | RN | Estado Agente — Ativo → DesligadoAutomatico (taxa de erro > 30% em 15 min) | Transição auto-disable | S5 |
| REQ-070 | D01 | 2.2 | RN | Estado Agente — FallbackAtivo → Ativo (API restaurada) | Transição retorno | S7 |
| REQ-071 | D01 | 2.2 | RN | Estado Agente — DesligadoAutomatico → Ativo (Admin reativa manualmente) | Transição reativação manual | S5 |
| REQ-072 | D05 | 2 | RF | RF-001: lista de todas as interações com 8 campos exatos | RF-001 | S3 |
| REQ-073 | D05 | 2 | RF | RF-002: estado vazio com ícone + texto + sugestão | RF-002 | S3 |
| REQ-074 | D05 | 2 | RF | RF-003: filtros (data, usuário, confiança, agente), chips, "Limpar filtros", skeleton inline | RF-003 | S3 |
| REQ-075 | D05 | 2 | RF | RF-004: log de acesso em admin_access_logs (Admin ID, timestamp ISO 8601, filtros) | RF-004 | S3 |
| REQ-076 | D05 | 2 | RF | RF-005: 6 alertas automáticos com condições, canais e SLA 60s | RF-005 | S5 |
| REQ-077 | D05 | 2 | RF | RF-006: prioridade de alertas simultâneos (desligamento primeiro, latência segundo) | RF-006 | S5 |
| REQ-078 | D05 | 2 | RF | RF-007: sinalização automática de interação com confiança < threshold → badge + botão takeover habilitado | RF-007 | S3 |
| REQ-079 | D05 | 2 | RF | RF-008: takeover com 4 efeitos (registrar, mensagem ao usuário, parar IA, manter input ativo) | RF-008 | S4 |
| REQ-080 | D05 | 2 | RF | RF-009: encerramento de takeover → agente retoma → mensagem + separador + avatar padrão | RF-009 | S4 |
| REQ-081 | D05 | 2 | RF | RF-010: bloquear takeover de conversa encerrada → tooltip "Conversa encerrada" | RF-010 | S4 |
| REQ-082 | D05 | 2 | RF | RF-011: lock otimista para takeover simultâneo → segundo Admin recebe mensagem de conflito | RF-011 | S4 |
| REQ-083 | D05 | 2 | RF | RF-012: Dashboard com 5 indicadores, filtros por período e agente, atualização tempo real | RF-012 | S6 |
| REQ-084 | D05 | 2 | RF | RF-013: indicador sem dados → "Dados insuficientes..." + ícone; zero nunca exibido | RF-013 | S6 |
| REQ-085 | D05 | 2 | RF | RF-014: threshold visualizável e ajustável em Configurações > Supervisão IA, padrão 80%, range 50–95% | RF-014 | S5 |
| REQ-086 | D05 | 2 | RF | RF-015: salvar threshold válido → aplicar imediatamente + toast + log de auditoria | RF-015 | S5 |
| REQ-087 | D05 | 2 | RF | RF-016: threshold inválido → erro inline + campo mantido + modal não fecha | RF-016 | S5 |
| REQ-088 | D05 | 2 | RF | RF-017: webchat 24/7 sem restrição de horário quando API disponível | RF-017 | S7 |
| REQ-089 | D05 | 2 | RF | RF-018: API indisponível → mensagem + Calculadora ativa + FAB badge amarelo | RF-018 | S7 |
| REQ-090 | D05 | 2 | RF | RF-019: rate limit 30 msg/h janela deslizante + histórico 90 dias | RF-019 | S7 |
| REQ-091 | D05 | 2 | RF | RF-020: gate de lançamento — 3 itens de isolamento verificados antes da ativação | RF-020 | S8 |
| REQ-092 | D05 | 2 | RF | RF-021: ativação bloqueada se qualquer item de RF-020 falhar | RF-021 | S8 |
| REQ-093 | D05 | 2 | RF | RF-022: instrução permanente com 4 componentes obrigatórios aprovados antes do lançamento | RF-022 | S8 |
| REQ-094 | D05 | 2 | RF | RF-023: 20 testes adversariais antes do lançamento, 100% recusas corretas | RF-023 | S8 |
| REQ-095 | D05 | 2 | RF | RF-024: 4 componentes de supervisão verificados antes do lançamento | RF-024 | S8 |
| REQ-096 | D05 | 2 | RF | RF-025: lançamento bloqueado se qualquer componente de RF-024 ausente | RF-025 | S8 |
| REQ-097 | D05 | 2 | RF | RF-026: RBAC aplicado em todos os endpoints/telas; não-Admin → 403 | RF-026 | S2 |
| REQ-098 | D05 | 3 | RNF | RNF-001: API Supervisão IA < 500ms p95 para até 1.000 registros | 500ms, 1000 registros | S9 |
| REQ-099 | D05 | 3 | RNF | RNF-001: Dashboard carregamento inicial < 1.5s; Realtime latência máxima 2s | 1.5s, 2s | S9 |
| REQ-100 | D05 | 3 | RNF | RNF-001: Takeover confirmação < 300ms | 300ms | S4 |
| REQ-101 | D05 | 3 | RNF | RNF-002: Painel Admin disponibilidade >= 99,5% | 99.5% SLA | S9 |
| REQ-102 | D05 | 3 | RNF | RNF-002: Alertas disparados em <= 60s após detecção | 60s | S5 |
| REQ-103 | D05 | 3 | RNF | RNF-003: Todos os endpoints Admin protegidos com JWT + role ADMIN | Segurança | S2 |
| REQ-104 | D05 | 3 | RNF | RNF-003: Logs de interação com identificadores anonimizados | PII proibido em logs | S1 |
| REQ-105 | D05 | 3 | RNF | RNF-003: Logs de auditoria imutáveis após criação | Sem deleção/edição retroativa | S1 |
| REQ-106 | D05 | 3 | RNF | RNF-004: Retenção de interações 90 dias (LGPD) | 90 dias | S1 |
| REQ-107 | D05 | 3 | RNF | RNF-004: Retenção de logs de auditoria Admin 1 ano | 365 dias | S1 |
| REQ-108 | D05 | 3 | RNF | RNF-005: Toda ação Admin registrada com Admin ID, tipo, timestamp ISO 8601 | 4 campos | S2 |
| REQ-109 | D05 | 3 | RNF | RNF-006: Filas RabbitMQ com retry e DLQ para alertas | DLQ obrigatória | S1 |
| REQ-110 | D05 | 3 | RNF | RNF-006: Cache Redis TTL 60s para métricas | TTL 60s | S1 |
| REQ-111 | D05 | 3 | RNF | RNF-007: Langfuse obrigatório para tracing de interações | Obrigatório em produção | S1 |
| REQ-112 | D05 | 3 | RNF | RNF-007: Sentry para error tracking (P0 alertas) | Sentry obrigatório | S1 |
| REQ-113 | D05 | 3 | RNF | RNF-007: PostHog para analytics + 8 eventos obrigatórios | 8 eventos definidos | S1 |
| REQ-114 | D05 | 3 | RNF | RNF-008: WCAG 2.1 AA — contraste 4.5:1, navegação por teclado, atributos ARIA | A11y obrigatório | S9 |
| REQ-115 | D05 | 3 | RNF | RNF-008: Skeleton obrigatório para loading (spinner proibido como substituto de conteúdo) | D04 Motion Spec | S1 |
| REQ-116 | D05 | 3 | RNF | RNF-009: Rate limit endpoints de configuração: máx 10 alterações de threshold por hora por Admin | 10/h | S5 |
| REQ-117 | D05 | 3 | RNF | RNF-010: Falha no sistema de alertas não impacta operação do agente nem do painel | Resiliência | S5 |
| REQ-118 | D05 | 3 | RNF | RNF-010: Falha no Langfuse não impacta operação do agente (async/não-bloqueante) | Resiliência | S1 |
| REQ-119 | D05 | 3 | RNF | RNF-011: Filtro de escopo testado com 100% sucesso antes de ativação | Pré-lançamento | S8 |
| REQ-120 | D05 | 3 | RNF | RNF-011: 20 perguntas adversariais aprovadas antes do lançamento | 20 testes | S8 |
| REQ-121 | D05 | 3 | RNF | RNF-012: Lock otimista para takeover simultâneo (sem race condition) | Concorrência | S4 |
| REQ-122 | D05 | 3 | RNF | RNF-012: Dashboard suporta até 10 Admins simultâneos sem degradação | 10 Admins | S6 |
| REQ-123 | D06 | 1 | TELA | T-001: Painel Supervisão IA — rota /admin/supervisao-ia, role Admin | T-001, RF-001–RF-004, RF-007, RF-026 | S3 |
| REQ-124 | D06 | 1 | TELA | T-002: Detalhe de Interação — rota /admin/supervisao-ia/interacoes/:id | T-002, RF-001, RF-007, RF-008, RF-010 | S3 |
| REQ-125 | D06 | 1 | TELA | T-003: Chat em Takeover — rota /admin/supervisao-ia/interacoes/:id/takeover | T-003, RF-008, RF-009, RF-011 | S4 |
| REQ-126 | D06 | 1 | TELA | T-004: Dashboard de Métricas — rota /admin/supervisao-ia/metricas | T-004, RF-012, RF-013 | S6 |
| REQ-127 | D06 | 1 | TELA | T-005: Configurações de Supervisão — rota /admin/supervisao-ia/configuracoes | T-005, RF-014–RF-016, RF-019 | S5 |
| REQ-128 | D06 | 1 | TELA | T-006: Log de Auditoria — rota /admin/supervisao-ia/auditoria | T-006, RF-004, RF-005, RNF-005 | S3 |
| REQ-129 | D06 | 1 | TELA | T-007: Checklist de Prontidão para Lançamento — rota /admin/supervisao-ia/checklist-lancamento | T-007, RF-020–RF-025 | S8 |
| REQ-130 | D06 | 1 | TELA | COMP-001: Banner de Alerta Crítico (Global) — trigger: desligamento automático, latência alta, taxa de erro elevada | RF-005, RF-006 | S5 |
| REQ-131 | D06 | 1 | TELA | COMP-002: Separador Visual de Takeover (no chat do usuário) — exibido ao takeover/encerramento | RF-008, RF-009 | S4 |
| REQ-132 | D06 | 1 | TELA | MODAL-001: Modal de Confirmação de Takeover | RF-008 | S4 |
| REQ-133 | D06 | 1 | TELA | MODAL-002: Modal de Encerramento de Takeover | RF-009 | S4 |
| REQ-134 | D06 | 1 | TELA | MODAL-003: Modal de Reativação de Agente | RF-005 | S5 |
| REQ-135 | D06 | 3 | TELA | T-001: 7 estados (loading inicial, loading de filtro, lista populada, vazio, sinalizada, em takeover, alerta crítico) | 7 estados | S3 |
| REQ-136 | D06 | 3 | TELA | T-002: 5 estados (loading, respondida, sinalizada, em takeover, encerrada) | 5 estados | S3 |
| REQ-137 | D06 | 3 | TELA | T-003: 5 estados (takeover ativo, mensagem recebida, mensagem enviada, encerramento, conflito) | 5 estados | S4 |
| REQ-138 | D06 | 3 | TELA | T-004: 4 estados de card (loading, com dados, sem dados, atualização realtime) | 4 estados | S6 |
| REQ-139 | D06 | 3 | TELA | T-005: 5 estados (padrão, edição, validação inválida, salvo, loading de salvar) | 5 estados | S5 |
| REQ-140 | D06 | 3 | TELA | T-006: 4 estados (loading, com dados, vazio, filtro ativo) | 4 estados | S3 |
| REQ-141 | D06 | 3 | TELA | T-007: 3 estados (itens pendentes, todos aprovados, item falhou) | 3 estados | S8 |
| REQ-142 | D07 | 2–10 | TELA | Wireframes textuais para T-001 a T-007 e COMP-001, COMP-002: layout base AppSidebar (256px) + conteúdo | Estrutura de layout | S3 |
| REQ-143 | D08 | 3.3 | TELA | Labels de navegação da Sidebar: "Supervisão IA / Interações / Métricas / Configurações / Auditoria / Checklist de Lançamento" | Textos exatos | S1 |
| REQ-144 | D08 | 3.3 | TELA | Labels de filtros: "Período", "Agente", "Nível de confiança", "Status" com placeholders exatos | Labels e placeholders | S3 |
| REQ-145 | D08 | 3.4 | TELA | Dropdown de motivo de takeover: 5 opções ("Resposta incorreta da IA", "Confiança abaixo do aceitável", "Situação complexa para a IA", "Solicitação do usuário", "Outro") | 5 opções exatas | S4 |
| REQ-146 | D08 | 5 | TELA | Mensagem de erro threshold: texto exato (seção 5.1 D08) + inline + cor warning + campo mantém valor | UX Writing conforme D08 | S5 |
| REQ-147 | D08 | 5.2–5.6 | TELA | 5 mensagens de erro com apresentação (toast, tooltip, inline) e auto-dismiss (6s, 4s, 3s) | Textos e comportamentos exatos | S3 |
| REQ-148 | D08 | 6 | TELA | 6 toasts de sucesso com textos exatos e auto-dismiss definidos | Textos exatos | S3 |
| REQ-149 | D09 | 1.1 | TELA | Componentes base: Table, Card, Badge, Button, Dialog/AlertDialog, Sheet, Toast/Sonner, Slider, Input, Select, Tabs, Separator, Skeleton, Tooltip, Alert/Custom Banner, Ícones Lucide | shadcn/ui completo | S1 |
| REQ-150 | D09 | 1.3 | TELA | Layout base: padding p-6, max-w-7xl, header flex items-center justify-between mb-6 | Tokens exatos | S1 |
| REQ-151 | D09 | 2 | TELA | InteractionTable: props InteractionRow com 7 campos; row styles por status (4 variações de cor) | Contratos de UI | S3 |
| REQ-152 | D09 | 2 | TELA | BadgeStatus: statusConfig com 4 status mapeados para variant (success, warning, takeover, secondary) | Contrato de componente | S3 |
| REQ-153 | D09 | 2 | TELA | ConfidenceDisplay: >= threshold padrão; < threshold com TriangleAlert 12px warning; null → "—" | Contrato de componente | S3 |
| REQ-154 | D09 | 2 | TELA | LatencyDisplay: <= 2s normal; > 2s warning; > SLA com TriangleAlert; null → "—" | Contrato de componente | S3 |
| REQ-155 | D10 | 5 | GLOSS | Constante: DEFAULT_CONFIDENCE_THRESHOLD = 80 | Valor exato | CROSS |
| REQ-156 | D10 | 5 | GLOSS | Constante: MIN_CONFIDENCE_THRESHOLD = 50 | Valor exato | CROSS |
| REQ-157 | D10 | 5 | GLOSS | Constante: MAX_CONFIDENCE_THRESHOLD = 95 | Valor exato | CROSS |
| REQ-158 | D10 | 5 | GLOSS | Constante: WEBCHAT_RATE_LIMIT_DEFAULT = 30 | Valor exato | CROSS |
| REQ-159 | D10 | 5 | GLOSS | Constante: INTERACTION_HISTORY_RETENTION_DAYS = 90 | Valor exato | CROSS |
| REQ-160 | D10 | 5 | GLOSS | Constante: AUDIT_LOG_RETENTION_DAYS = 365 | Valor exato | CROSS |
| REQ-161 | D10 | 5 | GLOSS | Constante: AUTO_DISABLE_ERROR_RATE_THRESHOLD = 30 | Valor exato | CROSS |
| REQ-162 | D10 | 5 | GLOSS | Constante: AUTO_DISABLE_WINDOW_MINUTES = 15 | Valor exato | CROSS |
| REQ-163 | D10 | 5 | GLOSS | Constante: HIGH_ERROR_RATE_THRESHOLD = 10 | Valor exato | CROSS |
| REQ-164 | D10 | 5 | GLOSS | Constante: HIGH_ERROR_RATE_WINDOW_MINUTES = 15 | Valor exato | CROSS |
| REQ-165 | D10 | 5 | GLOSS | Constante: DEGRADED_CSAT_THRESHOLD = 3.5 | Valor exato | CROSS |
| REQ-166 | D10 | 5 | GLOSS | Constante: HIGH_REFUSAL_RATE_THRESHOLD = 20 | Valor exato | CROSS |
| REQ-167 | D10 | 5 | GLOSS | Constante: PROCESSING_BUDGET_ALERT_THRESHOLD = 80 | Valor exato | CROSS |
| REQ-168 | D10 | 5 | GLOSS | Constante: LATENCY_SLA_SECONDS = [DADO PENDENTE] ⚠️ AMBÍGUO (RM-001) | Valor indefinido | CROSS |
| REQ-169 | D10 | 5 | GLOSS | Constante: MIN_ADVERSARIAL_TESTS_BEFORE_LAUNCH = 20 | Valor exato | CROSS |
| REQ-170 | D10 | 5 | GLOSS | Constante: DASHBOARD_METRICS_CACHE_TTL_SECONDS = 60 | Valor exato | CROSS |
| REQ-171 | D10 | 5 | GLOSS | Constante: AGENT_CONFIG_CACHE_TTL_SECONDS = 300 | Valor exato | CROSS |
| REQ-172 | D11 | 1.1 | TELA | Mobile Admin: alertas push, lista de interações sinalizadas, detalhe, takeover (iniciar/enviar/encerrar) — suporte v1.0 | 6 funcionalidades mobile | S7 |
| REQ-173 | D11 | 1.1 | TELA | Mobile: Dashboard completo, configurar threshold, checklist de lançamento — Web-only v1.0 | Escopo web-only documentado | S7 |
| REQ-174 | D11 | 2 | INFRA | Mobile stack: React Native 0.76+, Expo SDK 52+, expo-router 4.x+, React Native Reanimated 3.x+, React Native Gesture Handler 2.x+, TanStack Query 5.x+, Zustand 5.x+, expo-notifications, expo-secure-store, AsyncStorage | Stack mobile exata | S7 |
| REQ-175 | D12 | 2.1 | BANCO | Tabela: interactions — 12 campos com tipos e constraints exatos | UUID PK, FKs, enum status, Timestamptz, soft delete | S1 |
| REQ-176 | D12 | 2.1 | BANCO | Enum InteractionStatus: RESPONDIDA_PELA_IA, SINALIZADA_PARA_REVISAO, EM_TAKEOVER, ENCERRADA | 4 valores exatos | S1 |
| REQ-177 | D12 | 2.1 | BANCO | Índices de interactions: 5 índices explícitos (confidence+created, agent+created, user+created, status, created_at) | 5 índices | S1 |
| REQ-178 | D12 | 2.1 | BANCO | Retenção interactions: job cron diário — soft delete registros com created_at < now() - 90 days | Job cron | S1 |
| REQ-179 | D12 | 2.2 | BANCO | Tabela: takeovers — 9 campos com tipos e constraints exatos | UNIQUE interaction_id, enum status | S1 |
| REQ-180 | D12 | 2.2 | BANCO | Enum TakeoverStatus: ATIVO, ENCERRADO | 2 valores | S1 |
| REQ-181 | D12 | 2.2 | BANCO | Índices de takeovers: idx_takeovers_interaction, idx_takeovers_admin_started, idx_takeovers_status | 3 índices | S1 |
| REQ-182 | D12 | 2.3 | BANCO | Tabela: agent_configurations — 7 campos; UNIQUE agent_id; confidence_threshold DEFAULT 80; rate_limit_per_hour DEFAULT 30 | Defaults e constraints | S1 |
| REQ-183 | D12 | 2.4 | BANCO | Tabela: admin_access_logs — 5 campos; enum AdminActionType (7 valores); hard delete após 365 dias (sem soft delete) | Imutável, 7 valores de enum | S1 |
| REQ-184 | D12 | 2.4 | BANCO | Enum AdminActionType: ACESSO_PAINEL, TAKEOVER_INICIADO, TAKEOVER_ENCERRADO, THRESHOLD_ALTERADO, RATE_LIMIT_ALTERADO, AGENTE_REATIVADO, LANCAMENTO_AUTORIZADO | 7 valores exatos | S1 |
| REQ-185 | D12 | 2.5 | BANCO | Tabela: alert_events — 9 campos; enum AlertType (6 valores); enum AlertStatus (3 valores) | 3 enums | S1 |
| REQ-186 | D12 | 2.5 | BANCO | Enum AlertType: DESLIGAMENTO_AUTOMATICO, LATENCIA_ALTA, TAXA_ERRO_ELEVADA, CSAT_DEGRADADO, TAXA_RECUSA_ALTA, CONSUMO_PROCESSAMENTO | 6 valores exatos | S1 |
| REQ-187 | D12 | 2.5 | BANCO | Enum AlertStatus: ATIVO, RECONHECIDO, RESOLVIDO | 3 valores | S1 |
| REQ-188 | D12 | 2.6 | BANCO | Tabela: launch_readiness_checklists — 12 campos; 6 campos JSON de check; enum ChecklistStatus (3 valores) | Estrutura JSON de check definida | S1 |
| REQ-189 | D12 | 2.6 | BANCO | Enum ChecklistStatus: PENDENTE, APROVADO, BLOQUEADO | 3 valores | S1 |
| REQ-190 | D12 | 2.7 | BANCO | Tabela: adversarial_test_results — 8 campos; FK → launch_readiness_checklists | Resultados individuais | S1 |
| REQ-191 | D12 | 2.8 | BANCO | Tabela: push_notification_tokens — 7 campos; UNIQUE expo_push_token; platform = "ios"|"android" | Tokens mobile Admin | S1 |
| REQ-192 | D12 | 4 | BANCO | Políticas de retenção: interactions 90 dias (cron diário), admin_access_logs 365 dias (cron mensal), alert_events 365 dias (cron mensal), takeovers cascade com interactions | Jobs automáticos | S1 |
| REQ-193 | D13 | 1 | BANCO | Schema Prisma completo: 8 models com todos enums, @@index, @@map, @@unique | Prisma 6.x+ | S1 |
| REQ-194 | D14 | 2 | INFRA | 6 containers: Backend NestJS, PostgreSQL (Supabase), Redis (Upstash prod), RabbitMQ (CloudAMQP prod ⚠️ RM-004), Supabase Realtime, serviços externos | Containers definidos | S1 |
| REQ-195 | D14 | 2.1 | INFRA | Exchange RabbitMQ: dani-admin.alerts; DLQ obrigatória; retry exponencial máx 3 tentativas | Filas exatas | S1 |
| REQ-196 | D14 | 2.3 | INFRA | Cache Redis: chave padrão metrics:{agent_id}:{period}; TTL 60s; fallback para PostgreSQL | Chave e TTL exatos | S1 |
| REQ-197 | D14 | 3 | INFRA | 7 módulos NestJS: SupervisionPanel, Takeover, Metrics, AgentConfig, LaunchReadiness, Alerts, Auth Guard | Módulos exatos | S1 |
| REQ-198 | D14 | 5 | INFRA | Lock otimista para takeover: INSERT ... ON CONFLICT (interaction_id) DO NOTHING; verificar rows affected = 1 | ADR-001 | S4 |
| REQ-199 | D15 | 2 | INFRA | Estrutura de pastas apps/api/src: 7 módulos de domínio + audit + common; padrão Controller→Service→Repository→DTO | Pastas exatas | S1 |
| REQ-200 | D15 | 3 | INFRA | Redis: 4 chaves com prefixo dani-admin: metrics:{agent_id}:{period}, rate-limit:webchat:{user_id}:{window}, agent-config:{agent_id}, alerts:active-count | 4 chaves exatas | S1 |
| REQ-201 | D16 | — | ENDPOINT | GET /api/v1/admin/interactions — query params: agentId, userId, status, confidenceMin, confidenceMax, dateFrom, dateTo, cursor, limit (padrão 20, máx 100) | 9 query params | S3 |
| REQ-202 | D16 | — | ENDPOINT | GET /api/v1/admin/interactions/:id — path param id (UUID) | Detalhe de interação | S3 |
| REQ-203 | D16 | — | ENDPOINT | POST /api/v1/admin/takeover — body: interactionId (UUID, obrigatório), reason (string, máx 500, opcional) | Lock otimista | S4 |
| REQ-204 | D16 | — | ENDPOINT | DELETE /api/v1/admin/takeover/:id — encerrar takeover ativo | Encerramento | S4 |
| REQ-205 | D16 | — | ENDPOINT | GET /api/v1/admin/takeover/:id — detalhe de takeover | Consulta | S4 |
| REQ-206 | D16 | — | ENDPOINT | GET /api/v1/admin/metrics — query: agentId, period (day|week|month) | Cache Redis 60s | S6 |
| REQ-207 | D16 | — | ENDPOINT | GET /api/v1/admin/agent-config/:agentId — configuração atual | Consulta | S5 |
| REQ-208 | D16 | — | ENDPOINT | PATCH /api/v1/admin/agent-config/:agentId — body: confidenceThreshold (50–95), rateLimitPerHour (1–100) | Atualização | S5 |
| REQ-209 | D16 | — | ENDPOINT | GET /api/v1/admin/alerts — query: status, agentId, cursor, limit | Listagem ordenada por prioridade | S5 |
| REQ-210 | D16 | — | ENDPOINT | PATCH /api/v1/admin/alerts/:id/acknowledge — reconhecer alerta | Ação Admin | S5 |
| REQ-211 | D16 | — | ENDPOINT | POST /api/v1/admin/agents/:agentId/reactivate — reativar agente após desligamento | Reativação manual | S5 |
| REQ-212 | D16 | — | ENDPOINT | GET /api/v1/admin/audit-logs — query: adminId, actionType, dateFrom, dateTo, cursor, limit | Log de auditoria | S3 |
| REQ-213 | D16 | — | ENDPOINT | GET /api/v1/admin/launch-readiness/:agentId — checklist de prontidão | Consulta | S8 |
| REQ-214 | D16 | — | ENDPOINT | POST /api/v1/admin/launch-readiness/:agentId — criar/atualizar checklist | Upsert | S8 |
| REQ-215 | D16 | — | ENDPOINT | POST /api/v1/admin/launch-readiness/:agentId/adversarial-tests — adicionar teste adversarial | Resultado individual | S8 |
| REQ-216 | D16 | — | ENDPOINT | POST /api/v1/admin/launch-readiness/:agentId/authorize — autorizar lançamento (gate: 100% checks + ≥ 20 testes) | Autorização | S8 |
| REQ-217 | D16 | — | ENDPOINT | POST /api/v1/admin/push-tokens — registrar token Expo Push | Mobile Admin | S7 |
| REQ-218 | D16 | — | ENDPOINT | DELETE /api/v1/admin/push-tokens/:token — remover token | Logout mobile | S7 |
| REQ-219 | D16 | — | ENDPOINT | ⚠️ AMBÍGUO (RM-005): endpoint revoke presente em D27 mas ausente de D16 | Marcar como pendente | S9 |
| REQ-220 | D16 | 2 | ENDPOINT | Paginação cursor-based: cursor (UUID) + limit (padrão 20, máx 100); response: {data, meta: {nextCursor, hasMore, total}} | Shape padrão | S1 |
| REQ-221 | D16 | 2.4 | ENDPOINT | Shape de erro padrão: {error: {code, message, details?}} com formato código DA-{MÓDULO}-{NNN} | 15+ códigos definidos | S1 |
| REQ-222 | D17 | 1 | INTEG | OpenAI API: modelo GPT-4o (prod), text-embedding-3-small; env: OPENAI_API_KEY, OPENAI_ORG_ID | RM-002: Vitest adotado | S1 |
| REQ-223 | D17 | 2 | INTEG | Langfuse: SDK 3.x+; fire-and-forget; env: LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY, LANGFUSE_HOST; trace com 6 campos | Tracing async | S1 |
| REQ-224 | D17 | 3 | INTEG | Slack: Incoming Webhook; canal #alertas-dani-admin; env: SLACK_WEBHOOK_URL; payload JSON com blocks | Via RabbitMQ | S1 |
| REQ-225 | D17 | 4 | INTEG | SendGrid: env: SENDGRID_API_KEY, SENDGRID_FROM_EMAIL; templates para 3 tipos de alertas (P0, P1/P2, P3/P5) | E-mail alertas | S1 |
| REQ-226 | D17 | 5 | INTEG | Sentry: @sentry/nestjs; env: SENTRY_DSN; captura apenas 5xx; PII proibido | Error tracking | S1 |
| REQ-227 | D17 | 6 | INTEG | PostHog: env: POSTHOG_API_KEY, POSTHOG_HOST; 8 eventos obrigatórios; kill switch webchat-enabled | Feature flags | S1 |
| REQ-228 | D17 | 7 | INTEG | Expo Push Notifications: env: EXPO_ACCESS_TOKEN; TTL por alerta (P0: 3600s, P1: 1800s); chave idempotência Redis | Push mobile | S7 |
| REQ-229 | D18 | 2 | INFRA | JWT: access token 15min; refresh token 7 dias; payload: sub, email, role, iat, exp, iss, aud | Durações exatas | S2 |
| REQ-230 | D18 | 3 | INFRA | AdminAuthGuard: @UseGuards(AdminAuthGuard) em todos os controllers; verifica JWT + role ADMIN | Guard obrigatório | S2 |
| REQ-231 | D18 | 4 | INFRA | Mobile Admin: JWT em expo-secure-store; verificação de validade a cada abertura + a cada request | Segurança mobile | S7 |
| REQ-232 | D19 | 1 | INFRA | Agentes arquitetura: 7 passos (feature flag → filtro escopo → busca RAG → injeta contexto → calcula confidence_score → registra interação → trace Langfuse) | Fluxo completo | S7 |
| REQ-233 | D19 | 2 | INFRA | Instrução permanente Dani-Cessionário: 5 seções (identidade, dados bloqueados, exemplos recusa, formato resposta, limite escopo) | System prompt exato | S7 |
| REQ-234 | D19 | 3 | INFRA | Instrução permanente Dani-Cedente: equivalente à Dani-Cessionário para o perfil Cedente | System prompt Cedente | S7 |
| REQ-235 | D19 | 4 | INFRA | RAG: busca semântica no vector_store via Supabase pgvector + embeddings text-embedding-3-small | RAG obrigatório | S7 |
| REQ-236 | D19 | 5 | INFRA | Endpoint interno: POST /api/v1/internal/interactions — protegido por API key (não JWT) | Endpoint interno | S7 |
| REQ-237 | D20 | 1 | INFRA | Classificação de erros: 10 categorias com HTTP status, Sentry (S/N), Pino level | Tabela de classificação | S1 |
| REQ-238 | D20 | 2 | INFRA | HttpExceptionFilter global: captura e formata todas as exceções não tratadas | Filter global | S1 |
| REQ-239 | D21 | 1 | NOTIF | Mapa de notificações: 10 eventos × 4 canais com configuração exata (P0–P5 e eventos de chat) | Mapa de 10×4 | S5 |
| REQ-240 | D21 | 2 | NOTIF | Templates push: 3 templates (P0 desligamento, P1 latência, P2 taxa erro) com payload TypeScript completo | TTL por template | S5 |
| REQ-241 | D21 | 3 | NOTIF | Templates Slack: 3 templates (P0, P1, P2) com Block Kit JSON completo | Blocos Slack | S5 |
| REQ-242 | D21 | 4 | NOTIF | Templates e-mail SendGrid: 3 templates (P0, P3, P5) com HTML + texto | Templates HTML | S5 |
| REQ-243 | D21 | 5 | NOTIF | Idempotência: chave Redis alertEventId para evitar duplicatas em retry | Redis de idempotência | S5 |
| REQ-244 | D22 | 2 | INFRA | Setup em 5 passos: clone → install → .env → docker-compose up → prisma migrate dev | Passos exatos | S1 |
| REQ-245 | D22 | 3 | ENV | Variáveis de ambiente obrigatórias: DATABASE_URL, DIRECT_URL, REDIS_URL, RABBITMQ_URL, OPENAI_API_KEY, LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY, LANGFUSE_HOST, SLACK_WEBHOOK_URL, SENDGRID_API_KEY, SENDGRID_FROM_EMAIL, SENTRY_DSN, POSTHOG_API_KEY, POSTHOG_HOST, EXPO_ACCESS_TOKEN, JWT_SECRET, ADMIN_EMAIL | 17+ variáveis | S1 |
| REQ-246 | D22 | 4 | INFRA | docker-compose: PostgreSQL 17+, Redis 7.4+, RabbitMQ 4.x+ — portas 5432, 6379, 5672, 15672 | Docker compose local | S1 |
| REQ-247 | D22 | 4 | INFRA | Seed: `npm run seed` → cria AgentConfiguration padrão (threshold=80, rate_limit=30) | Seed obrigatório | S1 |
| REQ-248 | D23 | — | INFRA | Git flow: feature/* → main via PR; squash merge; conventional commits obrigatórios | Git flow | S1 |
| REQ-249 | D24 | 1 | INFRA | 3 ambientes: development (local), staging (auto on merge to main), production (manual via tag v*) | URLs definidas | S9 |
| REQ-250 | D24 | 2 | INFRA | Pipeline CI: lint → type-check → unit tests → integration tests → build (GitHub Actions) | Pipeline mínimo obrigatório | S9 |
| REQ-251 | D24 | 3 | INFRA | Pipeline CD staging: automático ao merge na main; prisma migrate deploy antes do server start | Automação | S9 |
| REQ-252 | D24 | 3 | INFRA | Pipeline CD production: manual via tag semântica v*; gate: aprovação manual Tech Lead + checklist prontidão | Go/No-Go | S9 |
| REQ-253 | D24 | 4 | INFRA | Versionamento SemVer: tags v1.0.0, v1.1.0, v2.0.0; breaking changes → major | SemVer | S9 |
| REQ-254 | D24 | 5 | INFRA | Rollback: reverter tag + re-deploy; migrations destrutivas exigem migration de rollback | Rollback strategy | S9 |
| REQ-255 | D25 | 2 | OBS | Pino: JSON estruturado; nível debug dev / info staging+prod; redact de PII (11 paths) | Configuração Pino | S1 |
| REQ-256 | D25 | 3 | OBS | Langfuse: fire-and-forget; trace por request com 6 campos (id, name, userId anonimizado, agentId, confidenceScore, latencyMs) | Trace obrigatório | S1 |
| REQ-257 | D25 | 4 | OBS | Sentry: GlobalHttpExceptionFilter captura apenas 5xx; alertas P0 configurados | Sem PII | S1 |
| REQ-258 | D25 | 5 | OBS | PostHog: 8 eventos obrigatórios com snake_case; sem PII; IDs anonimizados | 8 eventos | S1 |
| REQ-259 | D25 | 6 | OBS | Health check endpoint /health: liveness + readiness; PostgreSQL + Redis + RabbitMQ verificados | Health check | S1 |
| REQ-260 | D26 | 1 | INFRA | Classificação de incidentes: P0 (< 5 min), P1 (< 15 min), P2 (< 1h), P3 (< 24h) | SLAs de resposta | S9 |
| REQ-261 | D26 | 2 | INFRA | Runbook P0: sequência de diagnóstico (container, logs, health check, banco, Redis, CPU/mem) | Procedimento exato | S9 |
| REQ-262 | D26 | — | INFRA | Kill switch de IA: feature flag webchat-enabled = false no PostHog em < 30s | Kill switch | S9 |
| REQ-263 | D26 | — | INFRA | Takeover travado: UPDATE interactions + DELETE FROM takeovers (SQL exato no runbook) | Procedimento de recovery | S9 |
| REQ-264 | D26 | — | INFRA | DLQ acumulando: inspecionar, corrigir, republicar via RabbitMQ Management UI | DLQ recovery | S9 |
| REQ-265 | D27 | 1 | TESTE | Estratégia 7 camadas: Service (Vitest, 80%), Controller (Vitest, 70%), Repository (Vitest+Prisma), API E2E (Vitest+Supertest), Consumers RabbitMQ (Vitest, 80%), Agentes IA (adversarial), Segurança (manual) | ⚠️ CONFLITO (RM-002): D27 usa Jest; adotar Vitest | S9 |
| REQ-266 | D27 | 2.1 | TESTE | Test cases SupervisionService: 5 cenários (lista paginada, filtro por status, lista vazia, log ACESSO_PAINEL, NotFoundException) | IDs de test cases | S3 |
| REQ-267 | D27 | 2.2 | TESTE | Test cases TakeoverService: lock otimista, encerramento, conversa encerrada, bloqueio por status | Casos críticos | S4 |
| REQ-268 | D27 | 2.4 | TESTE | ConfigurationService: threshold 75% atualizado, 45% rejeitado (400), log de auditoria registrado | Configuração tests | S5 |
| REQ-269 | D27 | 2.5 | TESTE | MetricsService: métricas calculadas corretamente, cache hit (retorna cachedAt), dados insuficientes (null com status) | Metrics tests | S6 |
| REQ-270 | D27 | 2.6 | TESTE | AlertsService: desligamento automático disparado, DLQ sem retry infinito, prioridade de ordenação | Alerts tests | S5 |
| REQ-271 | D27 | 3 | TESTE | E2E flows: fluxo completo supervisão + takeover; fluxo configuração threshold; fluxo alerta desligamento | E2E definidos | S9 |
| REQ-272 | D27 | 4 | TESTE | Testes adversariais: 7 categorias (acesso cruzado, jailbreak, prompt injection, dados pessoais, instrução interna, contorno de escopo, manipulação de formato) → 100% wasRefused: true | 7 categorias | S8 |
| REQ-273 | D28 | 1 | GOLIVE | P0 - Cobertura >= 80%; zero testes falhando; E2E rodando; lock otimista testado; adversarial tests passando | Itens P0 bloqueantes | S9 |
| REQ-274 | D28 | 1.2 | GOLIVE | P0 - Zero lint errors; zero console.log em produção; zero tipos any explícitos; Conventional Commits | Qualidade de código | S9 |
| REQ-275 | D28 | 2.1 | GOLIVE | P0 - AdminAuthGuard em todos os endpoints /admin/*; role ADMIN no JWT; endpoint /internal protegido por API key | Segurança P0 | S9 |
| REQ-276 | D28 | 2.2 | GOLIVE | P0 - ScopeFilter e ContextFilter funcionando; prompt injection bloqueado | Filtros P0 | S9 |
| REQ-277 | D28 | 3 | GOLIVE | P0 - API p95 < 500ms (1000 registros); takeover < 300ms; dashboard < 1.5s | Performance P0 | S9 |
| REQ-278 | D28 | 4 | GOLIVE | P0 - Langfuse configurado com trace por interação; Sentry com alertas P0; PostHog com eventos obrigatórios | Observabilidade P0 | S9 |
| REQ-279 | D29 | 1 | GOLIVE | Cronograma go-live: 3 fases (D-7 a D-1 pré, D0 rollout, D+1 a D+7 pós) | Cronograma exato | S9 |
| REQ-280 | D29 | 2 | GOLIVE | Rollout progressivo: 10% → 50% → 100% via PostHog feature flag webchat-enabled | 3 fases de rollout | S9 |
| REQ-281 | D29 | 2 | GOLIVE | Critérios de abortagem: CSAT < 3.0, erros 5xx > 2%, confidence médio < 60% nas primeiras 2h | Thresholds de abort | S9 |
| REQ-282 | D29 | 3 | GOLIVE | Load test D-3: 200 req/min por 30 min; zero erros 5xx; p95 < 500ms | Load test | S9 |
| REQ-283 | D29 | 3 | GOLIVE | Smoke tests pós-deploy: health check, autenticação, interação simples, takeover, métricas, alerta simulado | 6 smoke tests | S9 |

---

## Resumo de Cobertura por Sprint

| Sprint | REQs Atribuídos | Descrição |
|--------|----------------|-----------|
| S1 — Fundação | REQ-104 a REQ-113, REQ-115, REQ-118, REQ-143, REQ-149, REQ-150, REQ-175 a REQ-200, REQ-220 a REQ-227, REQ-237 a REQ-248, REQ-255 a REQ-259 | Banco completo, infra, observabilidade, docker |
| S2 — Auth | REQ-009, REQ-097, REQ-103, REQ-108, REQ-229 a REQ-231 | JWT, AdminAuthGuard, RBAC |
| S3 — Supervisão de Interações | REQ-010, REQ-011, REQ-015, REQ-017 a REQ-022, REQ-030, REQ-072 a REQ-075, REQ-078, REQ-123, REQ-124, REQ-128, REQ-135, REQ-136, REQ-140, REQ-142 a REQ-144, REQ-147, REQ-148, REQ-151 a REQ-154, REQ-201, REQ-202, REQ-212, REQ-266 | Painel supervisão, listagem, filtros, logs |
| S4 — Takeover Manual | REQ-012 a REQ-014, REQ-016, REQ-031 a REQ-039, REQ-063, REQ-079 a REQ-082, REQ-100, REQ-121, REQ-125, REQ-131 a REQ-133, REQ-137, REQ-145, REQ-198, REQ-203 a REQ-205, REQ-267 | Takeover, lock otimista, T-003 |
| S5 — Alertas e Configuração | REQ-023 a REQ-029, REQ-044 a REQ-048, REQ-062, REQ-064, REQ-066, REQ-069, REQ-071, REQ-076, REQ-077, REQ-085 a REQ-087, REQ-102, REQ-116, REQ-117, REQ-127, REQ-130, REQ-134, REQ-139, REQ-146, REQ-207 a REQ-211, REQ-239 a REQ-243, REQ-268, REQ-270 | Alertas automáticos, threshold, configuração |
| S6 — Dashboard de Métricas | REQ-040 a REQ-043, REQ-065, REQ-083, REQ-084, REQ-122, REQ-126, REQ-138, REQ-206, REQ-269 | Dashboard, métricas, realtime |
| S7 — Webchat e Agentes de IA | REQ-049 a REQ-053, REQ-058, REQ-068, REQ-070, REQ-088 a REQ-090, REQ-172 a REQ-174, REQ-217, REQ-218, REQ-228, REQ-231 a REQ-236 | Webchat, agentes, mobile, fallback |
| S8 — Prontidão para Lançamento | REQ-054 a REQ-057, REQ-059 a REQ-061, REQ-067, REQ-091 a REQ-096, REQ-119, REQ-120, REQ-129, REQ-141, REQ-213 a REQ-216, REQ-272 | Checklist de lançamento, gates de segurança |
| S9 — Qualidade e Go-Live | REQ-098, REQ-099, REQ-101, REQ-114, REQ-219, REQ-249 a REQ-254, REQ-260 a REQ-265, REQ-271, REQ-273 a REQ-283 | CI/CD, qualidade, go-live, runbook |
| CROSS (todas as sprints) | REQ-001 a REQ-008, REQ-155 a REQ-171 | Glossário, constantes cross-cutting |

---

## Verificação de Cobertura (GATE)

```
Registro Mestre: 283 requisitos
Atribuídos: 283 (100%)
Sem sprint: 0 (deve ser 0)
```

**GATE APROVADO — prosseguir para Etapa 4.**
