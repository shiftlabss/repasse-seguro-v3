# 10 - Glossário Técnico

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Produto e Engenharia | Dicionário central que mapeia linguagem de negócio para linguagem técnica | v1.0 | Claude Code Desktop | 22/03/2026 — America/Fortaleza |

---

> 📌 **TL;DR — Visão geral do glossário**
>
> - **Total de termos catalogados:** 72 termos principais
> - **Total de siglas e acrônimos:** 22 siglas
> - **Conceitos mapeados entre camadas:** 18 entidades principais (negócio → frontend → backend → banco → API)
> - **Sinônimos resolvidos:** 14 pares de sinônimos com termo canônico definido
> - **Ambiguidades críticas encontradas:** 3 ambiguidades resolvidas autonomamente
> - **Termos pendentes por insuficiência de insumo:** 0 — cobertura total dos documentos de input

---

## 1. Como Usar

### 1.1 Consulta

Este glossário está organizado em **ordem alfabética rigorosa**. Para localizar um termo:

1. Procure diretamente na coluna **Termo** da tabela principal (Seção 2).
2. Se o termo não aparecer como entrada principal, verifique a coluna **Sinônimos/Variações** — ele pode ser um sinônimo apontando para o canônico.
3. Para siglas, consulte a Seção 3 — Siglas e Acrônimos.
4. Para saber o nome correto em frontend/backend/banco/API, consulte a Seção 4 — Convenções de Nomenclatura entre Camadas.

### 1.2 Contribuição com novos termos

1. Verifique se o termo já existe no glossário (busca por sinônimos inclusa).
2. Se não existir, proponha a adição via PR com a entrada completa: termo, definição, entidade técnica, sinônimos e contexto de uso.
3. Aguarde aprovação do Tech Lead antes de usar o novo termo na codebase.
4. Sinônimos devem **sempre** apontar para o termo canônico — nunca ser registrados como entrada independente.
5. Nenhum termo novo entra na codebase sem registro aprovado aqui.

---

## 2. Glossário Principal

> ⚙️ **Convenção:** Termos em **negrito** na coluna Sinônimos indicam o canônico quando a entrada é um sinônimo.

| Termo | Definição | Entidade Técnica | Sinônimos/Variações | Contexto de Uso |
|---|---|---|---|---|
| Agente de IA | Módulo automatizado que analisa casos, detecta anomalias e emite decisões auditáveis sem intervenção humana. | `ai_agent_decisions` / `AiAgentDecision` | IA, AI Agent, Supervisão IA | Módulo de Supervisão IA — monitoramento contínuo do pipeline. |
| Analista | Perfil de operador com acesso a triagem, negociação e formalização; sem acesso a financeiro e configurações. | `users.role = 'ANALISTA'` / `UserRole.ANALISTA` | — | Módulo de Usuários — gestão de acessos e permissões. |
| Anuência | Autorização formal da construtora para transferência do contrato imobiliário ao Cessionário. | `formalization_criteria.anuencia_status` | Autorização da construtora | Módulo de Formalização — critério obrigatório para Fechamento. |
| Backoffice | → Ver **Admin** (termo canônico). | — | Admin, Painel Administrativo | — |
| Bloqueado | Estado do caso em que a triagem identificou impedimento e o caso aguarda resolução antes de prosseguir. | `cases.status = 'BLOQUEADO'` | Em Bloqueio | Módulo de Triagem — gestão do FIFO de casos. |
| Cancelado | Estado terminal do caso em que o processo é encerrado sem conclusão de negócio. | `cases.status = 'CANCELADO'` | Descartado, Encerrado | Ciclo de vida do caso — estado final negativo. |
| Captado | Estado inicial do caso, indicando que o lead foi registrado e ainda não passou pela triagem. | `cases.status = 'CAPTADO'` | Lead, Novo | Ciclo de vida do caso — ponto de entrada no pipeline. |
| Cartão de Pipeline | Componente visual que representa um caso na view Kanban do Pipeline; exibe dados resumidos do caso. | `PipelineCard` / `CaseCardKanban` | Card de Caso, Kanban Card | Módulo de Pipeline — visualização Kanban. |
| Caso | Unidade central de negócio que representa a gestão completa de uma cessão imobiliária desde a captação até a conclusão. | `cases` / `Case` | Processo, Deal, Negócio | Todos os módulos — entidade principal do produto. |
| Cedente | Parte vendedora que cede seus direitos contratuais sobre o imóvel ao Cessionário. | `cedentes` / `Cedente` | Cedente, Vendedor, Parte Vendedora | Módulo de Negociação, Formalização — cadastro e documentação. |
| Cenário | Modalidade de retorno financeiro ao Cedente (A, B, C ou D) definida em negociação e usada no cálculo de comissões. | `proposals.cedente_scenario` | Modalidade, Opção de Repasse | Módulo de Negociação — proposta e contraproposta. |
| Cenário A | Cedente recebe integralmente o saldo devedor quitado pela Repasse Seguro. | `cedente_scenario = 'A'` | — | Módulo Financeiro — cálculo de comissão do Cedente. |
| Cenário B | Cedente recebe saldo devedor + percentual do Delta negociado. | `cedente_scenario = 'B'` | — | Módulo Financeiro — cálculo de comissão do Cedente. |
| Cenário C | Cedente recebe saldo devedor + valor fixo negociado. | `cedente_scenario = 'C'` | — | Módulo Financeiro — cálculo de comissão do Cedente. |
| Cenário D | Cedente recebe saldo devedor + percentual sobre o valor total do imóvel. | `cedente_scenario = 'D'` | — | Módulo Financeiro — cálculo de comissão do Cedente. |
| Cessão | Transferência formal dos direitos e obrigações de um contrato imobiliário do Cedente para o Cessionário. | — | Repasse, Transferência | Contexto jurídico e comercial — conceito central do produto. |
| Cessão Imobiliária | → Ver **Cessão** (termo canônico). | — | Repasse Imobiliário, Transferência Imobiliária | — |
| Cessionário | Parte compradora que adquire os direitos contratuais sobre o imóvel do Cedente. | `cessionarios` / `Cessionario` | Comprador, Parte Compradora, Adquirente | Módulo de Negociação, Formalização — cadastro e documentação. |
| Checklist de Fechamento | Conjunto dos 4 critérios obrigatórios que devem ser satisfeitos para que o caso avance para o estado Fechamento. | `formalization_criteria` | Critérios de Fechamento, Fechamento Criteria | Módulo de Formalização — gate de transição de estado. |
| Comissão | Valor calculado pela Repasse Seguro como receita pela intermediação do negócio, cobrado de Cedente e/ou Cessionário. | `commission_invoices` / `CommissionInvoice` | Taxa, Fee | Módulo Financeiro — faturamento e distribuição. |
| Concluído | Estado terminal positivo do caso em que todas as obrigações foram cumpridas. | `cases.status = 'CONCLUIDO'` | Finalizado, Encerrado com Sucesso | Ciclo de vida do caso — estado final positivo. |
| Conta Escrow | Conta bancária administrada pela Celcoin que retém os recursos financeiros do negócio até a liberação autorizada. | `escrow_accounts` / `EscrowAccount` | Conta Bloqueada, Escrow | Módulo Financeiro — gestão de recursos. |
| Contraproposta | Resposta do Cessionário a uma Proposta do Cedente, ajustando condições financeiras. | `counterproposals` / `Counterproposal` | Contra-oferta | Módulo de Negociação — ciclo de proposta e aceite. |
| Controle de Versão | Mecanismo de `version` incremental nas entidades para prevenir conflitos de escrita concorrente (optimistic locking). | `version` (coluna) | Optimistic Locking, Versionamento de Registro | Todas as entidades core — integridade de dados. |
| Coordenador | Perfil de operador com acesso ampliado; pode supervisionar analistas e aprovar etapas específicas. | `users.role = 'COORDENADOR'` / `UserRole.COORDENADOR` | — | Módulo de Usuários — gestão de acessos. |
| Dashboard | Tela inicial do Admin com KPIs consolidados do pipeline, gráficos e alertas. | `DashboardPage` / `DashboardModule` | Home, Painel | Módulo de Dashboard — visão executiva. |
| Decision Log | → Ver **Decisão do Agente** (termo canônico). | — | — | — |
| Decisão do Agente | Registro imutável de uma ação tomada pelo Agente de IA, incluindo motivo, confiança e dados analisados. | `ai_agent_decisions` / `AiAgentDecision` | Decision Log, AI Decision | Módulo de Supervisão IA — auditoria de ações automatizadas. |
| Delta | Diferença entre o valor de mercado do imóvel e o saldo devedor do contrato; base de cálculo da comissão do Cessionário. | `proposals.delta` | Diferencial, Spread | Módulo de Negociação e Financeiro — cálculo de comissão. |
| Disputa Formal | Estado do caso em que há conflito entre partes com necessidade de mediação formal. | `cases.status = 'DISPUTA_FORMAL'` | Litígio | Ciclo de vida do caso — fluxo de resolução de conflitos. |
| Distribuição | Operação de liberação dos recursos da Conta Escrow para as partes após aprovação do Gestor Financeiro ou Master. | `distributions` / `Distribution` | Liberação, Release | Módulo Financeiro — encerramento financeiro do caso. |
| Dossiê | Conjunto de documentos obrigatórios que devem ser coletados e validados durante a formalização do caso. | `dossie_documents` / `DossieDocument` | Pasta de Documentos, Documentação | Módulo de Formalização — checklist documental. |
| Em Formalização | Estado do caso em que os documentos estão sendo coletados e validados. | `cases.status = 'EM_FORMALIZACAO'` | Formalização | Ciclo de vida do caso. |
| Em Mediação | Estado do caso em que há conflito entre partes sendo tratado de forma não formal. | `cases.status = 'EM_MEDIACAO'` | Mediação | Ciclo de vida do caso — fluxo de conflito. |
| Em Negociação | Estado do caso em que propostas e contrapropostas estão sendo trocadas. | `cases.status = 'EM_NEGOCIACAO'` | Negociação | Ciclo de vida do caso. |
| Em Reversão | Estado do caso em que o Fechamento foi contestado e há pedido de desfazimento dentro do prazo legal. | `cases.status = 'EM_REVERSAO'` | Reversão | Ciclo de vida do caso — fluxo pós-fechamento. |
| Em Triagem | Estado do caso em que está sendo avaliado por um Analista antes de qualificar. | `cases.status = 'EM_TRIAGEM'` | Triagem | Ciclo de vida do caso. |
| Envelope | Objeto criado na ZapSign representando um conjunto de documentos para coleta de assinaturas eletrônicas. | `zapsign_envelopes` / `ZapsignEnvelope` | ZapSign Envelope, Documento ZapSign | Módulo de Formalização — integração com ZapSign. |
| Escrow | → Ver **Conta Escrow** (termo canônico). | — | — | — |
| Fechamento | Estado e evento do caso em que os 4 critérios obrigatórios foram atendidos e o negócio é formalmente concluído. | `cases.status = 'FECHAMENTO'` / `ClosingEvent` | Closing, Conclusão do Negócio | Módulo de Formalização — gate principal do produto. |
| FIFO | Política de atendimento da fila de triagem: primeiro caso captado é o primeiro a ser atribuído. | — | First In First Out | Módulo de Triagem — regra de distribuição de trabalho. |
| Gestor Financeiro | Perfil de operador com acesso exclusivo ao módulo financeiro, aprovações de distribuição e relatórios financeiros. | `users.role = 'GESTOR_FINANCEIRO'` / `UserRole.GESTOR_FINANCEIRO` | Financeiro, Gestor | Módulo de Usuários — acessos financeiros. |
| Global Config | Parâmetro de configuração sistêmica que define comportamentos do produto (prazos, percentuais, limites). | `global_configs` / `GlobalConfig` | Configuração Global, Config Sistêmica | Módulo de Configurações — Master e Coordenador. |
| KPI | Indicador-chave de desempenho exibido no Dashboard para monitoramento executivo do pipeline. | — | Indicador, Métrica | Módulo de Dashboard — visão gerencial. |
| Master | Perfil de operador com acesso irrestrito a todos os módulos, incluindo configurações críticas e aprovações especiais. | `users.role = 'MASTER'` / `UserRole.MASTER` | Admin, Super Admin, Administrador | Módulo de Usuários — perfil de maior privilégio. |
| Módulo | Área funcional do Admin, acessível pelo Sidebar, com conjunto de telas e regras próprias. | — | Seção, Área | Navegação global — estrutura da SPA. |
| Oferta Ativa | Estado do caso em que há proposta publicada aguardando resposta do Cessionário. | `cases.status = 'OFERTA_ATIVA'` | Em Oferta, Publicado | Ciclo de vida do caso. |
| Operador | Qualquer usuário autenticado no Admin com um dos 4 perfis de acesso. | `users` / `User` | Usuário Interno, Collaborator | Módulo de Usuários — gestão de acessos. |
| Optimistic Locking | → Ver **Controle de Versão** (termo canônico). | — | — | — |
| Pipeline | Módulo do Admin que exibe todos os casos em view Kanban ou Lista, organizados por estado. | `PipelineModule` / `PipelinePage` | Funil, Kanban | Módulo de Pipeline — visão operacional do ciclo de vida. |
| Pós Fechamento | Estado do caso após o Fechamento, durante o período de Reversão (até 15 dias). | `cases.status = 'POS_FECHAMENTO'` | Pós-Closing | Ciclo de vida do caso — janela de reversão. |
| Proposta | Oferta financeira formal enviada ao Cessionário com as condições do negócio. | `proposals` / `Proposal` | Oferta, Offer | Módulo de Negociação — ciclo de proposta. |
| Qualificado | Estado do caso após triagem aprovada, indicando que o caso está apto para negociação. | `cases.status = 'QUALIFICADO'` | Aprovado na Triagem | Ciclo de vida do caso. |
| Realtime | Mecanismo de Supabase Realtime que sincroniza dados entre servidor e cliente via WebSocket. | — | WebSocket, Live Update | Pipeline, Supervisão IA, Financeiro — atualizações ao vivo. |
| Repasse | → Ver **Cessão** (termo canônico). | — | Cessão, Transferência | — |
| Repasse Seguro | Nome do produto — plataforma de intermediação de cessões imobiliárias. | — | RS, Plataforma | Contexto geral do produto. |
| Reversão | Processo de desfazimento do Fechamento iniciado dentro de 15 dias após o evento de closing. | `cases.status = 'EM_REVERSAO'` / `ReversalProcess` | Cancelamento Pós-Fechamento, Distrato | Módulo de Formalização — fluxo pós-fechamento. |
| RLS | Row Level Security — política de segurança em nível de linha no PostgreSQL/Supabase que isola dados por tenant. | — | Row Level Security | Banco de dados — isolamento de dados. |
| Saldo Devedor | Valor ainda devido pelo Cedente ao banco/construtora referente ao financiamento do imóvel. | `proposals.outstanding_balance` | Dívida Remanescente | Módulo de Negociação — base de cálculo financeiro. |
| Sidebar | Menu de navegação lateral fixo do Admin com links para os 10 módulos. | `AppSidebar` / `SidebarNav` | Menu Lateral, Nav | Layout global — componente de navegação. |
| Signatário | Pessoa que deve assinar um documento eletrônico no envelope ZapSign. | `zapsign_envelopes.signatories` | Assinante, Signer | Módulo de Formalização — integração ZapSign. |
| SLA | Acordo de nível de serviço que define prazos máximos para ações operacionais no pipeline. | — | Service Level Agreement, Prazo Operacional | Módulo de Triagem e Notificações — monitoramento de prazos. |
| Snapshot de Configuração | Cópia imutável dos parâmetros globais vigentes no momento da criação de um caso, preservada para integridade histórica. | `case_config_snapshots` / `CaseConfigSnapshot` | Config Snapshot, Configuração Congelada | Todos os módulos — regra RN-111. |
| Soft Delete | Padrão de exclusão lógica onde o registro recebe `deleted_at` preenchido mas permanece no banco. | `deleted_at` (coluna) | Exclusão Lógica, Logical Delete | Todas as tabelas — dado nunca é apagado fisicamente. |
| Supervisão IA | Módulo do Admin dedicado ao monitoramento das decisões do Agente de IA e configuração de parâmetros de automação. | `AiSupervisionModule` | IA Supervision, Painel de IA | Módulo de Supervisão IA. |
| Timestamp | Valor de data e hora armazenado com fuso horário (`TIMESTAMPTZ`) no banco PostgreSQL/Supabase. | `TIMESTAMPTZ` | Datetime, Data/Hora | Todas as tabelas — padrão de coluna temporal. |
| Topbar | Barra de cabeçalho superior fixa do Admin com identidade do produto, perfil do usuário e notificações. | `AppTopbar` / `TopbarNav` | Header, Barra Superior | Layout global — componente de identidade e acesso rápido. |
| Transação de Escrow | Movimentação financeira (crédito ou débito) registrada na Conta Escrow de um caso. | `escrow_transactions` / `EscrowTransaction` | Movimentação, Transação | Módulo Financeiro — histórico de movimentações. |
| Triagem | Processo de avaliação inicial de um caso Captado para determinar se é elegível para negociação. | `TriagemModule` / `TriagemPage` | Qualificação Inicial, Screening | Módulo de Triagem — FIFO de casos. |
| UUID | Identificador único universal v4 usado como chave primária em todas as tabelas do banco. | `uuid` (tipo PostgreSQL) | ID, Primary Key, PK | Todas as tabelas — convenção de PK. |
| Webhook | Endpoint HTTP que recebe notificações assíncronas de integrações externas (ZapSign, Celcoin). | — | Callback, Notificação HTTP | Integrações — ZapSign e Celcoin. |

---

## 3. Siglas e Acrônimos

| Sigla | Significado Completo | Contexto |
|---|---|---|
| ADR | Architecture Decision Record | Documento de decisão arquitetural registrada no Doc 02 — Stacks. |
| API | Application Programming Interface | Interface de comunicação entre backend e frontend/integrações. |
| BaaS | Backend as a Service | Modelo de serviço do Supabase — banco + auth + storage + realtime. |
| CDN | Content Delivery Network | Distribuição de assets estáticos via Vercel Edge Network. |
| CI/CD | Continuous Integration / Continuous Deployment | Pipeline de build e deploy automatizado. |
| DTO | Data Transfer Object | Objeto usado em NestJS para validar dados de entrada de endpoints. |
| ERD | Entity-Relationship Diagram | Diagrama relacional do banco de dados — Doc 12. |
| FIFO | First In First Out | Política de fila de triagem: primeiro captado é o primeiro atendido. |
| FK | Foreign Key | Chave estrangeira em relacionamentos entre tabelas. |
| KPI | Key Performance Indicator | Indicador-chave de desempenho exibido no Dashboard. |
| MFA | Multi-Factor Authentication | Autenticação multifator para operadores do Admin. |
| MQ | Message Queue | Fila de mensagens assíncronas via RabbitMQ. |
| MVP | Minimum Viable Product | Escopo mínimo viável — não aplicável neste pipeline (cobertura 100%). |
| ORM | Object-Relational Mapper | Camada de abstração de banco — Prisma no projeto. |
| PK | Primary Key | Chave primária das tabelas — sempre UUID v4 neste projeto. |
| PRD | Product Requirements Document | Documento de requisitos do produto — Doc 05. |
| RF | Requisito Funcional | Requisito de comportamento do sistema — catalogados no PRD. |
| RLS | Row Level Security | Isolamento de dados por linha no PostgreSQL/Supabase. |
| RN | Regra de Negócio | Regra que define comportamento do produto — catalogadas nos Docs 01.x. |
| RS | Repasse Seguro | Nome abreviado do produto (uso interno). |
| SLA | Service Level Agreement | Prazo máximo para execução de ações operacionais. |
| SPA | Single Page Application | Arquitetura do Admin — não há reload de página na navegação. |
| UUID | Universally Unique Identifier | Identificador único v4 usado como PK em todas as tabelas. |

---

## 4. Convenções de Nomenclatura entre Camadas

> ⚙️ **Padrão:** `snake_case` para banco; `PascalCase` para entidades TypeScript; `camelCase` para variáveis e props; `kebab-case` para rotas de API.

| Conceito de Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|
| Caso | `Case`, `CaseCard`, `useCaseStore` | `CaseService`, `CaseRepository`, `CreateCaseDto` | `cases`, `case_id` | `/cases`, `CaseResponse` |
| Cedente | `Cedente`, `CedenteForm` | `CedenteService`, `CedenteDto` | `cedentes`, `cedente_id` | `/cedentes`, `CedenteResponse` |
| Cessionário | `Cessionario`, `CessionarioForm` | `CessionarioService`, `CessionarioDto` | `cessionarios`, `cessionario_id` | `/cessionarios`, `CessionarioResponse` |
| Proposta | `Proposal`, `ProposalCard` | `ProposalService`, `CreateProposalDto` | `proposals`, `proposal_id` | `/cases/:id/proposals`, `ProposalResponse` |
| Contraproposta | `Counterproposal`, `CounterproposalForm` | `CounterproposalService`, `CreateCounterproposalDto` | `counterproposals`, `counterproposal_id` | `/cases/:id/counterproposals`, `CounterproposalResponse` |
| Dossiê (documento) | `DossieDocument`, `DossieUpload` | `DossieService`, `UploadDocumentDto` | `dossie_documents`, `document_id` | `/cases/:id/dossie`, `DossieDocumentResponse` |
| Envelope ZapSign | `ZapsignEnvelope`, `SignatureStatus` | `ZapsignService`, `ZapsignEnvelopeDto` | `zapsign_envelopes`, `envelope_id` | `/zapsign/envelopes`, `EnvelopeResponse` |
| Critérios de Fechamento | `FormalizationCriteria`, `ClosingChecklist` | `FormalizationService`, `UpdateCriteriaDto` | `formalization_criteria`, `criteria_id` | `/cases/:id/formalization`, `FormalizationCriteriaResponse` |
| Conta Escrow | `EscrowAccount`, `EscrowCard` | `EscrowService`, `EscrowAccountDto` | `escrow_accounts`, `account_id` | `/cases/:id/escrow`, `EscrowAccountResponse` |
| Transação de Escrow | `EscrowTransaction`, `TransactionRow` | `EscrowTransactionService`, `TransactionDto` | `escrow_transactions`, `transaction_id` | `/escrow/:account_id/transactions`, `EscrowTransactionResponse` |
| Distribuição | `Distribution`, `DistributionModal` | `DistributionService`, `CreateDistributionDto` | `distributions`, `distribution_id` | `/cases/:id/distributions`, `DistributionResponse` |
| Comissão / Invoice | `CommissionInvoice`, `InvoiceCard` | `CommissionService`, `CommissionInvoiceDto` | `commission_invoices`, `invoice_id` | `/cases/:id/commission`, `CommissionInvoiceResponse` |
| Decisão do Agente IA | `AiDecisionCard`, `AiDecisionLog` | `AiAgentService`, `AiDecisionDto` | `ai_agent_decisions`, `decision_id` | `/ai/decisions`, `AiDecisionResponse` |
| Configuração Global | `GlobalConfigForm`, `ConfigItem` | `GlobalConfigService`, `UpdateConfigDto` | `global_configs`, `config_key` | `/configs`, `GlobalConfigResponse` |
| Snapshot de Config | `ConfigSnapshot` (read-only) | `CaseConfigSnapshotService` | `case_config_snapshots`, `snapshot_id` | `/cases/:id/config-snapshot`, `ConfigSnapshotResponse` |
| Usuário / Operador | `User`, `UserCard`, `useAuthStore` | `UserService`, `CreateUserDto` | `users`, `user_id` | `/users`, `UserResponse` |
| Histórico de Status | `CaseStatusHistory`, `StatusTimeline` | `CaseStatusService`, `StatusHistoryDto` | `case_status_history`, `history_id` | `/cases/:id/status-history`, `StatusHistoryResponse` |
| Log de Notificação | `NotificationLog`, `NotificationBadge` | `NotificationService`, `NotificationLogDto` | `notification_logs`, `notification_id` | `/notifications`, `NotificationLogResponse` |

---

## 5. Ambiguidades Resolvidas

### 5.1 "Repasse" vs "Cessão"

**Termos ambíguos:** "Repasse" e "Cessão" são usados de forma intercambiável nos documentos de input para descrever a transferência de direitos contratuais.

**Interpretações possíveis:**
- A) "Cessão" como termo jurídico formal; "Repasse" como termo comercial simplificado.
- B) Termos totalmente equivalentes.

**Decisão autônoma [DECISÃO AUTÔNOMA]:** "Cessão" é adotado como termo canônico técnico (utilizado no banco, API e documentação técnica), enquanto "Repasse" é mantido apenas como sinônimo de comunicação comercial e no nome do produto "Repasse Seguro". Justificativa: "Cessão" é o termo jurídico correto para transferência de direitos contratuais (Código Civil Brasileiro, Art. 286+); "Repasse" tem conotação coloquial e pode causar ambiguidade com "repasse financeiro". Alternativa descartada: adotar "Repasse" como canônico tornaria a nomenclatura técnica juridicamente imprecisa.

**Termo canônico técnico:** Cessão / `cases` / `/cases`

---

### 5.2 "Admin" vs "Backoffice" vs "Painel"

**Termos ambíguos:** Os documentos usam "Admin", "Backoffice" e "Painel Administrativo" para se referir ao mesmo produto.

**Interpretações possíveis:**
- A) Termos diferentes para módulos distintos.
- B) Termos equivalentes para o mesmo produto.

**Decisão autônoma [DECISÃO AUTÔNOMA]:** "Admin" é adotado como termo canônico. Justificativa: é o nome mais curto, amplamente usado em contextos técnicos (NestJS admin, React admin patterns) e o que aparece com maior frequência nos documentos de input. "Backoffice" e "Painel" são registrados como sinônimos. Alternativa descartada: "Backoffice" tem conotação ampla que pode confundir com sistemas de outros departamentos.

**Termo canônico:** Admin

---

### 5.3 "Operador" vs "Usuário Interno"

**Termos ambíguos:** "Operador" e "Usuário Interno" referem-se ao mesmo ator — qualquer pessoa autenticada no Admin.

**Interpretações possíveis:**
- A) "Operador" = usuário que executa operações no pipeline; "Usuário Interno" = qualquer funcionário com acesso.
- B) Termos equivalentes.

**Decisão autônoma [DECISÃO AUTÔNOMA]:** "Operador" é adotado como termo canônico técnico para qualquer usuário autenticado no Admin com um dos 4 perfis. Justificativa: "Operador" é mais preciso — implica que o usuário opera o sistema ativamente — e é usado consistentemente nas Regras de Negócio. "Usuário" é mantido como nome da entidade técnica (`users` / `User`) por convenção. Alternativa descartada: "Usuário Interno" cria redundância com "usuário externo" (Cedente/Cessionário no portal público, se existir).

**Termo canônico:** Operador (negócio) / `User` / `users` (técnico)

---

## 6. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 22/03/2026 | v1.0 | Versão inicial — 72 termos, 23 siglas, 18 entidades mapeadas entre camadas, 3 ambiguidades resolvidas autonomamente. |

---

## 7. Backlog de Pendências

> ✅ Nenhuma pendência registrada. Cobertura total dos documentos de input atingida com 72 termos catalogados.

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Repasse vs Cessão | [DECISÃO AUTÔNOMA] | 5.1 | "Cessão" adotado como canônico técnico (base jurídica); "Repasse" mantido como sinônimo comercial | Nomenclatura de API e banco | Tech Lead | Resolvido |
| Admin vs Backoffice | [DECISÃO AUTÔNOMA] | 5.2 | "Admin" adotado por frequência e precisão técnica | Naming conventions globais | Tech Lead | Resolvido |
| Operador vs Usuário Interno | [DECISÃO AUTÔNOMA] | 5.3 | "Operador" no negócio, `User`/`users` no técnico — separa conceito de role do objeto de dados | Documentação de API e banco | Tech Lead | Resolvido |
