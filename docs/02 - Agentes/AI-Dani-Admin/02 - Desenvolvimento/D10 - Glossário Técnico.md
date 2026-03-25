# Glossário Técnico — AI-Dani-Admin

## Termos de Domínio, Negócio e Tecnologia

| Campo | Valor |
|---|---|
| Destinatário | Produto, Engenharia e QA |
| Escopo | Definições normativas de todos os termos usados no módulo AI-Dani-Admin |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D01 (seção 1 - Glossário de Domínio), D05 (RFs), D08 (vocabulário aprovado) |

---

> **📌 TL;DR**
>
> - 3 categorias de termos: Domínio de Negócio (módulo Admin), Técnicos (infraestrutura e IA) e de Interface (UX Writing).
> - Termos do D01 seção 1 são replicados aqui como fonte canônica para engenharia e QA.
> - Toda divergência entre código, documentação e esta lista deve ser resolvida a favor desta lista.
> - Termos de interface (coluna "Na interface") são os aprovados pelo D08 - UX Writing.

---

## 1. Termos de Domínio de Negócio

| Termo | Definição | Referência |
|---|---|---|
| **Admin** | Equipe operacional da plataforma Repasse Seguro. Usuário-alvo da Dani-Admin. Tem acesso exclusivo ao painel de Supervisão IA. | RN-DA-030, D01 seção 1 |
| **AI-Dani-Admin** | Nome interno do produto para a assistente de supervisão operacional dedicada ao Admin. | D01 |
| **Dani** | Nome exibido na interface para o agente (tanto para usuários finais quanto para o Admin). | D01 seção 2.1 |
| **Dani-Cessionário** | Instância do agente de IA dedicada ao atendimento de usuários Cessionários. Supervisionada pelo módulo AI-Dani-Admin. | D01 |
| **Dani-Cedente** | Instância do agente de IA dedicada ao atendimento de usuários Cedentes. Supervisionada pelo módulo AI-Dani-Admin. | D01 |
| **CSAT** | Customer Satisfaction Score — avaliação de satisfação do usuário com o atendimento da IA, em escala de 1 a 5. Meta operacional: média ≥ 3,5. | RN-DA-031, D01 seção 1 |
| **Nível de confiança** | Percentual de 0 a 100% que indica a certeza do agente sobre a resposta gerada. Também referido como `confidence_score` no código. Na interface: "Confiança". | D01 seção 1 |
| **Painel de Supervisão IA** | Interface do Admin para visualizar, filtrar e intervir em interações dos agentes. Engloba as telas T-001 a T-007 do módulo. | D01 seção 1 |
| **Takeover** | Intervenção manual do Admin em uma conversa ativa do agente. Admin assume o atendimento; agente é pausado. Na interface: "Assumir conversa" / "Atendimento humano". | RN-DA-032, RN-DA-033, D01 seção 1 |
| **Threshold de confiança** | Percentual mínimo de confiança aceito antes de sinalizar automaticamente uma interação para revisão do Admin. Configurável: 50%–95%. Padrão: 80%. Na interface: "Nível de supervisão". | RN-DA-035, D01 seção 1 |
| **Taxa de recusa** | Percentual de respostas em que o agente recusou fornecer dados por restrição de perfil. Alerta se > 20% em 24h. | RN-DA-031, D01 seção 1 |
| **DesligadoAutomatico** | Estado do agente quando a taxa de erro supera 30% em 15 minutos. Requer reativação manual pelo Admin. Na interface: "Desligado automaticamente". | RN-DA-031, D01 seção 1 |
| **FallbackAtivo** | Estado do agente quando a API do modelo de IA está indisponível. Calculadora de Comissão assume os cálculos determinísticos. Na interface: "Modo degradado". | RN-DA-036, D01 seção 1 |
| **SinalizadaParaRevisao** | Estado de uma interação cuja confiança está abaixo do threshold configurado, indicando que requer revisão do Admin. Na interface: "Aguardando revisão". | RN-DA-032 |
| **EmTakeover** | Estado de uma interação onde o Admin assumiu o atendimento manualmente. Agente está pausado para esta sessão. Na interface: "Em atendimento humano". | RN-DA-033 |
| **RespondidaPelaIA** | Estado de uma interação onde o agente respondeu dentro do SLA com confiança acima do threshold. Na interface: "Respondida pela IA". | RN-DA-030 |
| **Encerrada** | Estado final de uma interação, concluída pela IA ou pelo Admin. Não aceita novo takeover. | RN-DA-033 |
| **Critérios de prontidão** | Conjunto de verificações obrigatórias antes do lançamento de qualquer agente em produção: isolamento de acesso (RN-DA-037), cobertura de recusa (RN-DA-038) e supervisão funcional (RN-DA-039). | RN-DA-037 a RN-DA-039 |
| **Teste adversarial** | Pergunta formulada para tentar extrair dados bloqueados do agente por indução de prompt. Mínimo 20 testes adversariais antes de qualquer lançamento. | RN-DA-038 |
| **Filtro de escopo** | Mecanismo que valida, para cada consulta de dados, que o recurso pertence ao usuário autenticado antes de qualquer processamento. Pré-requisito de lançamento. | RN-DA-037 |
| **Filtro de contexto** | Mecanismo que garante que as informações fornecidas ao agente contêm apenas dados autorizados para o perfil do usuário autenticado. Pré-requisito de lançamento. | RN-DA-037 |
| **Prompt injection** | Tentativa de manipular o agente inserindo instruções maliciosas no input do usuário para extrair dados bloqueados ou alterar o comportamento do agente. | RN-DA-038 |
| **Instrução permanente do agente** | Prompt de sistema (system prompt) que define identidade, tom, dados bloqueados e formato de resposta do agente. Deve ser aprovada antes do lançamento. | RN-DA-038 |
| **Rate limit do webchat** | Limite de 30 mensagens por hora por usuário (janela deslizante). Configurável pelo Admin. | RN-DA-036 |
| **FAB global** | Floating Action Button — ícone fixo do chat visível em todas as telas do módulo do usuário. Exibe badge de status degradado quando API do modelo está indisponível. | RN-DA-036 |
| **Log de auditoria** | Registro imutável de todas as ações críticas do Admin: takeovers, alterações de threshold, reativações. Retenção: 1 ano. | RNF-005, D05 |
| **Calculadora de Comissão** | Módulo determinístico da plataforma que permanece ativo independentemente da disponibilidade do agente de IA. | RN-DA-036 |

---

## 2. Termos Técnicos (Infraestrutura e IA)

| Termo | Definição | Onde aparece no código |
|---|---|---|
| **confidence_score** | Valor numérico 0–100 representando o nível de confiança do agente na resposta gerada. Armazenado na tabela `interactions`. Na interface: "Confiança". | D12 (schema), D16 (API) |
| **confidence_threshold** | Valor configurável 50–95 (padrão 80) usado para sinalização automática de interações. Armazenado na tabela `agent_configurations`. Na interface: "Nível de supervisão". | D12, D16 |
| **LLM** | Large Language Model — modelo de linguagem de grande escala. No contexto da plataforma: GPT-4 via OpenAI SDK. | D02, D19 |
| **RAG** | Retrieval-Augmented Generation — técnica de geração aumentada por recuperação. O agente busca documentos relevantes no vector store antes de gerar a resposta. | D19 |
| **SSE** | Server-Sent Events — protocolo de streaming unidirecional (servidor → cliente) usado para transmitir respostas do agente em tempo real no webchat. | D02, D14 |
| **Vector store** | Banco de dados de embeddings vetoriais usado pelo RAG para recuperar documentos relevantes. Implementado via Supabase pgvector. | D02, D12 |
| **Embedding** | Representação vetorial de um texto gerada por um modelo de embedding (OpenAI Embeddings). Permite busca semântica por similaridade. | D19 |
| **Langfuse** | Plataforma de observabilidade de LLM. Registra traces de todas as interações dos agentes: custo, latência, qualidade, confiança. Alimenta métricas do dashboard Admin. | D02, D25 |
| **PostHog** | Plataforma de analytics e feature flags. Usado para rastrear eventos de uso do painel Admin e como kill switch para features de IA. | D02, D25 |
| **Sentry** | Plataforma de error tracking. Monitora erros de infraestrutura com stack traces, alertas P0. | D02, D25 |
| **Redis** | Cache em memória. Usado para: cache de métricas agregadas do dashboard (TTL 60s), rate limiting por usuário. | D02, D12 |
| **RabbitMQ** | Sistema de mensageria e filas. Usado para: disparo assíncrono de alertas (Slack, e-mail), processamento de notificações de takeover. | D02, D12 |
| **Supabase Realtime** | Serviço de WebSocket gerenciado do Supabase. Usado para subscriptions em tempo real no painel Admin (atualizações de interações, métricas). | D02, D12 |
| **Lock otimista** | Técnica de controle de concorrência que verifica a versão de um registro antes de modificá-lo, sem bloquear o banco. Usado no takeover simultâneo (RF-011). | D14 |
| **Dead-letter queue (DLQ)** | Fila que recebe mensagens que não puderam ser processadas após o número máximo de tentativas. Toda fila RabbitMQ deve ter DLQ associada. | D02 |
| **JWT** | JSON Web Token — padrão de autenticação stateless. Access token (curto) + refresh token (longo). Role `ADMIN` verificada em todos os endpoints do módulo. | D18 |
| **LGPD** | Lei Geral de Proteção de Dados — lei brasileira de privacidade. Impacta: anonimização de dados de interação, retenção máxima de 90 dias, acesso controlado a dados pessoais. | D05 RNF-004 |
| **PII** | Personally Identifiable Information — dados pessoais identificáveis. Proibidos em logs. Dados de usuário em interações: sempre anonimizados. | D05 RNF-003 |
| **Soft delete** | Estratégia de exclusão que marca registros como deletados (`deleted_at`) sem removê-los fisicamente. Padrão ShiftLabs para tabelas de domínio. | D02, D13 |
| **UUID v4** | Identificador único universal versão 4 (aleatório). Padrão ShiftLabs para primary keys. Seguro para expor em URLs e APIs. | D02, D13 |
| **Webhook** | Callback HTTP para notificação de eventos externos. Para alertas ao Slack: o sistema usa **incoming webhook** do Slack (URL configurada como variável de ambiente). Alertas são enfileirados via RabbitMQ (exchange `dani-admin.alerts`) e consumidos pelo AlertsConsumer que chama o webhook. [CORRIGIDO: PROBLEMA-014] | D17, D14 |
| **Feature flag** | Configuração em PostHog que permite habilitar/desabilitar features em produção sem deploy. Kill switch para desligar features de IA instantaneamente. | D02 |

---

## 3. Termos de Interface (UX Writing — Vocabulário Aprovado)

| Termo técnico interno | Termo na interface | Contexto de uso |
|---|---|---|
| `confidence_threshold` | "Nível de supervisão" | Label do campo de configuração em T-005 |
| `takeover` | "Assumir conversa" | Botão de ação em T-002 |
| `takeover` | "Atendimento humano" | Separador visual no chat (COMP-002) |
| `SINALIZADA_PARA_REVISAO` | "Aguardando revisão" | Badge de status em T-001 e T-002 |
| `EM_TAKEOVER` | "Em atendimento humano" | Badge de status em T-001 e T-002 |
| `RESPONDIDA_PELA_IA` | "Respondida pela IA" | Badge de status |
| `ENCERRADA` | "Encerrada" | Badge de status |
| `AGENTE_ATIVO` | "Ativo" | Status do agente no painel |
| `DESLIGADO_AUTOMATICO` | "Desligado automaticamente" | Status do agente + alerta |
| `FALLBACK_ATIVO` | "Modo degradado" | Status do agente no FAB |
| `confidence_score` | "Confiança" / "Nível de confiança" | Coluna na tabela, card de detalhes |
| `latency_ms` → convertido para `latency_seconds` | "Latência" | Coluna na tabela, card de detalhes |
| `admin_access_logs` | "Log de auditoria" | Nome da tela T-006 |
| `agent_configurations` | "Configurações de Supervisão" | Nome da tela T-005 |

---

## 4. Convenções de Nomenclatura no Código

| Contexto | Convenção | Exemplo |
|---|---|---|
| Tabelas do banco (PostgreSQL) | `snake_case`, plural | `interactions`, `admin_access_logs`, `agent_configurations` |
| Colunas do banco | `snake_case` | `confidence_score`, `created_at`, `agent_id` |
| Enums no banco | `UPPER_SNAKE_CASE` nos valores | `RESPONDIDA_PELA_IA`, `EM_TAKEOVER` |
| Variáveis TypeScript | `camelCase` | `confidenceScore`, `thresholdValue` |
| Constantes TypeScript | `UPPER_SNAKE_CASE` | `DEFAULT_CONFIDENCE_THRESHOLD`, `MAX_RATE_LIMIT` |
| Arquivos e pastas | `kebab-case` | `supervision-panel.module.ts`, `interaction-service.ts` |
| Módulos NestJS | `PascalCase` | `SupervisionPanelModule`, `TakeoverService` |
| Rotas de API | `kebab-case` com `/api/v1/` | `/api/v1/admin/interactions`, `/api/v1/admin/takeover` |
| Eventos PostHog | `snake_case` | `admin_takeover_initiated`, `agent_auto_disabled` |

---

## 5. Constantes de Negócio

| Constante | Valor | Fonte |
|---|---|---|
| `DEFAULT_CONFIDENCE_THRESHOLD` | 80 (%) | RN-DA-035 |
| `MIN_CONFIDENCE_THRESHOLD` | 50 (%) | RN-DA-035 |
| `MAX_CONFIDENCE_THRESHOLD` | 95 (%) | RN-DA-035 |
| `WEBCHAT_RATE_LIMIT_DEFAULT` | 30 (mensagens/hora) | RN-DA-036 |
| `INTERACTION_HISTORY_RETENTION_DAYS` | 90 (dias) | RN-DA-036 |
| `AUDIT_LOG_RETENTION_DAYS` | 365 (dias = 1 ano) | RNF-004 |
| `AUTO_DISABLE_ERROR_RATE_THRESHOLD` | 30 (%) | RN-DA-031 |
| `AUTO_DISABLE_WINDOW_MINUTES` | 15 (minutos) | RN-DA-031 |
| `HIGH_ERROR_RATE_THRESHOLD` | 10 (%) | RN-DA-031 |
| `HIGH_ERROR_RATE_WINDOW_MINUTES` | 15 (minutos) | RN-DA-031 |
| `DEGRADED_CSAT_THRESHOLD` | 3.5 (escala 1-5) | RN-DA-031 |
| `HIGH_REFUSAL_RATE_THRESHOLD` | 20 (%) | RN-DA-031 |
| `PROCESSING_BUDGET_ALERT_THRESHOLD` | 80 (% do orçamento mensal) | RN-DA-031 |
| `LATENCY_SLA_SECONDS` | [DADO PENDENTE: definir SLA de latência em segundos] | RNF-001 |
| `MIN_ADVERSARIAL_TESTS_BEFORE_LAUNCH` | 20 | RN-DA-038 |
| `DASHBOARD_METRICS_CACHE_TTL_SECONDS` | 60 | D02 seção 2.3 |
| `AGENT_CONFIG_CACHE_TTL_SECONDS` | 300 | D27 seção 2.4 (configuração muda raramente; TTL maior reduz carga sem impactar UX) [CORRIGIDO: PROBLEMA-015] |

---

## 6. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. 25 termos de negócio, 20 termos técnicos, 14 termos de interface + constantes. |
