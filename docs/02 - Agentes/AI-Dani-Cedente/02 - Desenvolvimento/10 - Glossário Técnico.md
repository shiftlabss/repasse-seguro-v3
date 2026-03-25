# 10 - Glossário Técnico — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Produto e Engenharia |
| Escopo | Dicionário central que mapeia linguagem de negócio para linguagem técnica |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D01 — Regras de Negócio · D05 — PRD · D12 — ERD Schema |

---

> **📌 TL;DR**
>
> - **Total de termos catalogados:** 48 entradas no glossário principal.
> - **Siglas e acrônimos:** 12 registrados.
> - **Conceitos mapeados entre camadas:** 9 entidades principais (negócio → frontend → backend → banco → API).
> - **Sinônimos resolvidos:** 21 variações mapeadas para termos canônicos.
> - **Ambiguidades críticas resolvidas:** 4 casos com decisão autônoma documentada.
> - **Termos pendentes:** nenhum — cobertura 100% dos documentos de input.

---

## 1. Como Usar

### 1.1 Consulta

- Glossário em ordem **alfabética rigorosa** — use busca no editor (Ctrl+F / Cmd+F) pelo termo.
- Sinônimos sempre apontam para o termo canônico — se você pesquisou "vendedor" e encontrou "→ Ver **Cedente**", use **Cedente**.
- Mapeamento entre camadas na Seção 4 — use quando precisar do nome correto em frontend, backend, banco ou API.

### 1.2 Contribuição de novos termos

1. Identifique o termo candidato no código, documentação ou conversa de time.
2. Verifique se já existe entrada no glossário (incluindo sinônimos).
3. Se não existir: abra PR com a nova entrada seguindo o formato da Seção 2.
4. **Todo novo termo deve ser aprovado** pelo Tech Lead ou Product Manager antes de entrar na codebase.
5. Sinônimos devem sempre apontar para o canônico — nunca criar dois canônicos para o mesmo conceito.

### 1.3 Regra de ouro

> ⚙️ Se um termo não está neste glossário, **não use na codebase** sem revisão documental. Crie a entrada primeiro.

---

## 2. Glossário Principal

Ordem alfabética rigorosa. Termos canônicos em negrito. Sinônimos apontam para o canônico.

| Termo | Definição | Entidade Técnica | Sinônimos/Variações | Contexto de Uso |
|---|---|---|---|---|
| **Aceite de proposta** | Ação do Cedente que aprova uma proposta recebida, iniciando o processo de Escrow. | `ProposalStatus.ACEITA` / `PATCH /proposals/:id/accept` | Aprovação, confirmação de proposta | Módulo de Propostas — transição de estado para `ACEITA` |
| **Admin** | Usuário interno da Repasse Seguro com acesso ao painel administrativo; pode intervir em conversas, aprovar documentos e gerenciar o sistema. | `role: 'admin'` / perfil no JWT | Administrador, gestor, moderador | Controle de acesso, takeover de conversa, aprovação de dossiê |
| **Agente** | Instância do serviço AI-Dani-Cedente em execução, processando conversas de Cedentes. | `DaniCedenteAgent` / LangGraph state machine | Bot, assistente, IA | Arquitetura do sistema de IA |
| **Backoff exponencial** | Estratégia de retry com espera crescente entre tentativas: 1s, 2s, 4s. | `RetryConfig` | Retry com espera | Fallback de chamadas à API OpenAI |
| **Bolha de chat** | Componente visual que exibe uma mensagem individual no chat, com variante primária (Dani) e secundária (Cedente). | `ChatBubble` (frontend) | Balão de chat, mensagem | Interface do chat — T-04 |
| **Card de análise** | Componente inline exibido no chat pela Dani para apresentar dados estruturados (proposta, dossiê, Escrow, simulação). | `AnalysisCard` (frontend) | Card de proposta, card inline, widget | T-05 — exibido após análise de proposta ou consulta de dossiê |
| **Cedente** | Pessoa física titular do financiamento imobiliário que deseja vender o repasse; usuário principal do AI-Dani-Cedente. | `cedente_profiles` / `CedenteProfile` | Vendedor, proprietário, cliente, titular | Toda a aplicação — isolamento de dados, JWT, RLS |
| **Cenário** | Configuração de repasse (A, B, C ou D) que define as condições de negociação; confidencial para o Cessionário. | `opportunity_scenarios` / `OpportunityScenario` | Modalidade, tipo de repasse, plano | Módulo de Oportunidade — cálculo do Δ |
| **Cessionário** | Pessoa interessada em comprar o repasse do Cedente; não tem acesso ao AI-Dani-Cedente. | Não existe entidade própria no escopo da Dani | Comprador, adquirente, interessado | Referenciado apenas no contexto de propostas e Escrow; dados são bloqueados pela Dani |
| **Chat** | Interface conversacional do AI-Dani-Cedente; o canal principal de comunicação entre Cedente e Dani. | `chat_sessions` / `chat_messages` / `ChatSession` | Webchat, assistente, conversa | Toda a aplicação |
| **Confidence score** | Métrica de confiança da resposta do agente (0–100%), calculada por Langfuse; abaixo de 80% aciona sinalização para takeover. | `confidence_score` (campo de log Langfuse) | Score de confiança, nível de certeza | Módulo de takeover — RF-DCE-030 |
| **Contraproposta** | Proposta alternativa enviada pelo Cedente em resposta a uma proposta do Cessionário, com valor diferente. | `ProposalType.CONTRAPROPOSTA` / `POST /proposals/:id/counter` | Contra-oferta, nova proposta, rebate | Módulo de Propostas — RF-DCE-019 |
| **CSAT** | Customer Satisfaction Score — avaliação de satisfação do Cedente ao encerrar uma conversa, escala 1–5. | `csat_score` (campo em `chat_sessions`) / `POST /chat/sessions/:id/csat` | Avaliação, nota de satisfação, feedback | Módulo de métricas — RF-DCE-032 |
| **Δ (Delta)** | Diferença entre Tabela Atual e Tabela Contrato do financiamento; base de cálculo do valor de repasse. Fallback para Valor Pago se Δ ≤ 0. | `delta` (campo calculado em `opportunity_scenarios`) | Delta, diferença de tabelas, variação | Módulo de Oportunidade — cálculo do cenário |
| **Documento** | Arquivo enviado pelo Cedente para compor o dossiê do processo de repasse. | `dossier_documents` / `DossierDocument` | Arquivo, anexo, comprovante | Módulo de Dossiê — RF-DCE-014 |
| **Dossiê** | Conjunto de documentos obrigatórios que o Cedente precisa enviar para viabilizar o repasse. | `dossier_documents` (coleção) / `DossierService` | Documentação, pasta, arquivos, checklist | Módulo de Dossiê — RF-DCE-014 a RF-DCE-015 |
| **Edge case** | Cenário excepcional ou limite que requer tratamento específico fora do fluxo principal. | Sem entidade técnica própria | Caso extremo, cenário limite | Seção de fallback e RF-DCE-033 |
| **Embedding** | Representação vetorial de texto gerada pelo modelo `text-embedding-3-small` (1536 dimensões) para busca semântica. | `embedding` (coluna `vector(1536)` em `knowledge_embeddings`) | Vetor semântico, representação vetorial | Módulo RAG |
| **Escrow** | Mecanismo de garantia financeira onde o Cessionário deposita o valor acordado antes da assinatura do contrato. | `escrow_transactions` / `EscrowTransaction` | Conta garantia, custódia, depósito garantido | Módulo de Escrow — RF-DCE-022 a RF-DCE-025 |
| **Estado** | Situação atual de uma entidade no ciclo de vida do processo de repasse. | `status` (campo em múltiplas tabelas) / TypeScript enum | Status, situação, fase | Oportunidade, proposta, Escrow, documento |
| **Extensão de prazo** | Prorrogação de +5 dias úteis no prazo do Escrow, solicitada pelo Cessionário e aprovada pelo Cedente (silêncio em 24h = aprovação). | `EscrowExtension` / `POST /escrow/:id/extend` | Prorrogação, renovação de prazo | Módulo de Escrow — RF-DCE-024 |
| **Fallback** | Comportamento do sistema em caso de falha da IA ou da infraestrutura, incluindo desligamento automático por threshold. | `FallbackService` | Contingência, modo degradado | RF-DCE-029 — thresholds 10%/30% |
| **Feature flag** | Mecanismo de ativação/desativação de funcionalidades em produção, via PostHog; usado como kill switch da Dani. | PostHog feature flag `dani_cedente_enabled` | Kill switch, toggle de funcionalidade | Módulo de fallback — RF-DCE-029 |
| **Financiamento** | Contrato de crédito imobiliário que o Cedente está quitando e deseja vender via repasse. | Referenciado em `opportunities.financing_contract_number` | Contrato imobiliário, crédito imobiliário | Módulo de Oportunidade |
| **Guardiã do Retorno** | Identidade da Dani-Cedente: parceira que acompanha o ciclo completo do repasse para maximizar o retorno do Cedente. | Sem entidade técnica | Assistente, agente, bot | Identidade do produto — D03 Brand Guide |
| **Histórico de conversas** | Registro das mensagens trocadas entre Cedente e Dani, retido por 90 dias (configurável). | `chat_messages` / `ChatMessage` | Log de chat, mensagens anteriores, contexto | Módulo de Chat — RF-DCE-007 |
| **Isolamento de dados** | Princípio arquitetural que impede um Cedente de acessar dados de outro Cedente ou de Cessionários. | JWT `cedente_id` + RLS Supabase + PII masking | Separação de dados, data isolation | Toda a aplicação — RN-DCE-001 |
| **JWT** | JSON Web Token — token de autenticação com claim `cedente_id` que identifica e isola o Cedente. | `AuthGuard` / `JwtService` | Token de acesso, auth token, Bearer token | Middleware de autenticação em todos os endpoints |
| **Kill switch** | Mecanismo de desligamento instantâneo da Dani em produção via feature flag PostHog. | PostHog `dani_cedente_enabled = false` | Desligamento forçado, shutdown | RF-DCE-029 |
| **KYC** | Know Your Customer — verificação de identidade obrigatória antes do primeiro uso da Dani. | `cedente_profiles.kyc_status` | Verificação de identidade, cadastro, onboarding | Módulo de Primeiro Uso — RF-DCE-005 |
| **Langfuse** | Plataforma de observabilidade LLM usada para rastrear traces, spans e calcular o confidence score de cada resposta. | `LangfuseService` / SDK `langfuse` | Observabilidade de IA, LLM tracing | Observabilidade obrigatória — D02 Stacks |
| **LangGraph** | Biblioteca JavaScript para construção de agentes stateful com grafo de estados; orquestra os fluxos da Dani. | `DaniCedenteGraph` / LangGraph.js | Orquestrador de agente, máquina de estados | Arquitetura da IA — estados: idle, analyzing_proposal, escrow_monitoring, dossier_guidance, takeover |
| **LangChain** | Biblioteca JavaScript para construção de chains, tools e pipelines RAG integrados ao LLM. | `LangChainService` / `RAGChain` | Framework de IA, pipeline LLM | Módulo RAG e tools da Dani |
| **Marketplace** | Área da plataforma Repasse Seguro onde as oportunidades de repasse são publicadas e visíveis para Cessionários. | `opportunity.status = 'PUBLICADA'` | Plataforma, site, portal, vitrine | Módulo de Oportunidade — publicação e retirada |
| **Mensagem** | Unidade de comunicação no chat, podendo ser do Cedente ou da Dani. | `chat_messages` / `ChatMessage` | Texto, resposta, input | Módulo de Chat |
| **Modal de confirmação** | Componente de UI que solicita confirmação explícita do Cedente antes de ações irreversíveis. | `ConfirmationModal` (frontend) | Dialog, pop-up, overlay de confirmação | T-06 — aceite de proposta, retirada de oportunidade |
| **Notificação proativa** | Mensagem enviada pela Dani ao Cedente por evento do sistema, sem que o Cedente tenha perguntado. | `NotificationService` / `ProactiveNotification` | Alerta, push notification, aviso automático | Módulo de Notificações — RF-DCE-028 (10 eventos) |
| **Oportunidade** | Registro de repasse disponível para negociação no marketplace, criado pelo Cedente. | `opportunities` / `Opportunity` | Anúncio, oferta, listagem, repasse publicado | Módulo de Oportunidade — ciclo de vida completo |
| **PII** | Personally Identifiable Information — dados pessoais que identificam diretamente um indivíduo; mascarados antes de qualquer log ou envio ao LLM. | PII masking middleware | Dados pessoais, informações sensíveis | Isolamento de dados — RN-DCE-001 |
| **Proposta** | Oferta enviada por um Cessionário ao Cedente com o valor que deseja pagar pelo repasse. | `proposals` / `Proposal` | Oferta, lance, bid, proposta de compra | Módulo de Propostas — RF-DCE-016 a RF-DCE-021 |
| **RAG** | Retrieval-Augmented Generation — técnica que recupera documentos relevantes do banco vetorial para enriquecer o contexto do LLM. | `RAGService` / `knowledge_embeddings` | Busca semântica, contexto aumentado, RAG pipeline | Módulo de conhecimento da Dani |
| **Rate limit** | Limite de 30 mensagens por hora por Cedente, controlado via Redis com janela deslizante. | `RateLimitGuard` / Redis counter | Limite de uso, throttle, quota | RF-DCE-007 — RF-DCE-009 |
| **Repasse** | Transferência do financiamento imobiliário do Cedente para um Cessionário, com pagamento do valor acordado. | `opportunities` (entidade principal) | Cessão, venda do financiamento, transferência | Toda a aplicação — produto negociado |
| **Retorno líquido** | Valor que o Cedente efetivamente receberá após o repasse: Valor do Repasse − Saldo Devedor. | `net_return` (campo calculado) | Lucro, ganho líquido, valor final, retorno | Módulo de Simulação — RF-DCE-018 |
| **RLS** | Row Level Security — política de segurança no Supabase/PostgreSQL que garante isolamento de dados por `cedente_id`. | RLS policies em todas as tabelas | Segurança de linha, isolamento de banco | Supabase — todas as tabelas do schema |
| **Saldo devedor** | Valor atual da dívida do financiamento imobiliário do Cedente com o banco. | `outstanding_balance` (campo em `opportunities`) | Dívida, débito, quanto falta pagar | Módulo de Simulação — cálculo do retorno líquido |
| **Sessão de chat** | Período contínuo de interação entre Cedente e Dani, com início, histórico e encerramento. | `chat_sessions` / `ChatSession` | Conversa, sessão, thread de chat | Módulo de Chat — CSAT coletado ao encerrar |
| **Simulação** | Cálculo estimado do retorno líquido do Cedente com base nos dados da proposta e saldo devedor. | `SimulationService` / `POST /simulation` | Cálculo de retorno, projeção, estimativa | Módulo de Simulação — RF-DCE-018 |
| **Takeover** | Intervenção manual do Admin em uma conversa da Dani quando o confidence score cai abaixo de 80%. | `TakeoverEvent` / `admin_takeover_logs` | Intervenção, override, assumir conversa | RF-DCE-030 |
| **Toast** | Notificação visual não-intrusiva exibida na interface do chat para alertar o Cedente sobre eventos do sistema. | `ToastNotification` (frontend) | Pop-up de notificação, snackbar, alerta visual | Componente transversal — 10 eventos proativos |
| **Valor do repasse** | Valor acordado entre Cedente e Cessionário para a transferência do financiamento. | `repasse_value` (campo em `proposals`) | Preço do repasse, valor de venda, valor acordado | Módulo de Propostas e Simulação |
| **ZapSign** | Plataforma de assinatura digital usada para formalizar o contrato de cessão entre Cedente e Cessionário. | `ZapSignService` / `signature_status` | Assinatura digital, contrato eletrônico | Módulo de Assinatura — RF-DCE-026 |

---

## 3. Siglas e Acrônimos

| Sigla | Significado Completo | Contexto |
|---|---|---|
| **Admin** | Administrador da plataforma Repasse Seguro | Controle de acesso, takeover, aprovação de documentos |
| **API** | Application Programming Interface | Endpoints REST do backend NestJS |
| **CSAT** | Customer Satisfaction Score | Avaliação de satisfação pós-conversa — escala 1–5 |
| **DTO** | Data Transfer Object | Padrão de transferência de dados no NestJS (ex: `CreateProposalDto`) |
| **ERD** | Entity Relationship Diagram | Modelo de dados — D12 |
| **JWT** | JSON Web Token | Token de autenticação com claim `cedente_id` |
| **KYC** | Know Your Customer | Verificação de identidade antes do primeiro uso |
| **LGPD** | Lei Geral de Proteção de Dados | Compliance de privacidade — histórico 90 dias, PII masking |
| **LLM** | Large Language Model | Modelo de linguagem (GPT-4) usado pela Dani |
| **PII** | Personally Identifiable Information | Dados pessoais mascarados antes de logs e envio ao LLM |
| **RAG** | Retrieval-Augmented Generation | Técnica de busca semântica para enriquecer contexto do LLM |
| **RLS** | Row Level Security | Política de isolamento de dados no Supabase PostgreSQL |
| **SLA** | Service Level Agreement | Acordo de nível de serviço — latência ≤5s p95 |
| **SSE** | Server-Sent Events | Protocolo de streaming de respostas do LLM para o frontend |

---

## 4. Convenções de Nomenclatura entre Camadas

| Conceito de Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|
| **Cedente** | `CedenteProfile`, `useCedente()` | `CedenteService`, `CedenteProfileDto` | `cedente_profiles` | `GET /cedente/profile` |
| **Oportunidade** | `Opportunity`, `OpportunityCard` | `OpportunityService`, `CreateOpportunityDto` | `opportunities` | `GET /opportunities`, `POST /opportunities` |
| **Cenário** | `OpportunityScenario`, `ScenarioCard` | `ScenarioService`, `OpportunityScenarioDto` | `opportunity_scenarios` | `GET /opportunities/:id/scenarios` |
| **Proposta** | `Proposal`, `ProposalCard`, `ProposalAnalysis` | `ProposalService`, `CreateCounterProposalDto` | `proposals` | `GET /proposals`, `PATCH /proposals/:id/accept` |
| **Dossiê** | `DossierDocument`, `DossierStatus` | `DossierService`, `UploadDocumentDto` | `dossier_documents` | `GET /dossier`, `POST /dossier/upload` |
| **Escrow** | `EscrowStatus`, `EscrowCard` | `EscrowService`, `EscrowTransactionDto` | `escrow_transactions` | `GET /escrow/:id`, `POST /escrow/:id/extend` |
| **Sessão de chat** | `ChatSession`, `useChatSession()` | `ChatService`, `ChatSessionDto` | `chat_sessions` | `POST /chat/sessions`, `GET /chat/sessions/:id` |
| **Mensagem** | `ChatMessage`, `ChatBubble` | `MessageService`, `CreateMessageDto` | `chat_messages` | `POST /chat/sessions/:id/messages` |
| **Conhecimento RAG** | N/A (backend only) | `RAGService`, `EmbeddingDto` | `knowledge_embeddings` | N/A (uso interno) |

---

## 5. Ambiguidades Resolvidas

| Termo ambíguo | Interpretações possíveis | Decisão | Termo canônico escolhido | Justificativa |
|---|---|---|---|---|
| **"Proposta"** | (A) Oferta inicial do Cessionário ao Cedente; (B) Contraproposta enviada pelo Cedente | Unified under `Proposal` com campo `type` discriminante (`INICIAL` / `CONTRAPROPOSTA`) | `Proposal` (com `ProposalType` enum) | D01 usa "proposta" para ambos os sentidos; o campo `type` resolve a distinção sem duplicar entidade. [DECISÃO AUTÔNOMA — alternativa descartada: criar entidade `Counteroffer` separada, o que duplica lógica de estado e aceitação sem benefício.] |
| **"Valor do repasse"** | (A) Valor pedido pelo Cedente na oportunidade; (B) Valor oferecido pelo Cessionário na proposta | São conceitos distintos: `opportunity.asking_price` vs `proposal.offered_value` | `asking_price` (oportunidade) / `offered_value` (proposta) | D01 distingue claramente o valor publicado pelo Cedente do valor ofertado pelo Cessionário. [DECISÃO AUTÔNOMA — termos descritivos em inglês snake_case seguindo convenção do banco.] |
| **"Retorno"** | (A) Retorno líquido calculado (Repasse − Saldo Devedor); (B) Valor bruto do repasse | Retorno = sempre líquido. Valor bruto = "valor do repasse" | `net_return` | D01 e D05 sempre usam "retorno" no sentido líquido. Valor bruto é chamado de "valor do repasse". [DECISÃO AUTÔNOMA — elimina ambiguidade em simulações e cards de análise.] |
| **"Cancelar"** | (A) Cancelar uma ação em andamento na UI (botão secundário); (B) Cancelar/recusar uma proposta como ação de negócio | São contextos distintos: `cancel` (UI action) vs `RECUSADA` (ProposalStatus) | `Cancelar` (UI) / `Recusar` (negócio) | D08 UX Writing define "Recusar" como verbo de negócio e "Cancelar" como ação de UI. [DECISÃO AUTÔNOMA — evita confusão no vocabulário controlado da interface.] |

---

## 6. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. 48 termos catalogados, 14 siglas, 9 entidades mapeadas entre 5 camadas, 21 sinônimos resolvidos, 4 ambiguidades documentadas. Alinhado com D01 (Regras de Negócio), D05 (PRD), D12 (ERD Schema) e D08 (UX Writing — vocabulário controlado). |

---

## 7. Backlog de Pendências

| Item | Marcador | Seção | Justificativa / Trade-off | Impacto | Dono | Status |
|---|---|---|---|---|---|---|
| Nome canônico para "oportunidade retirada temporariamente" vs "oportunidade encerrada/cancelada permanentemente" | DECISÃO AUTÔNOMA | Seção 2 — Oportunidade | Adotado `PAUSADA` para retirada temporária e `CANCELADA` para encerramento permanente, alinhado com os 6 estados do D05.2 (Rascunho, Publicada, Em negociação, Pausada, Expirada, Cancelada). Alternativa descartada: usar `INATIVA` (genérico demais). | P1 | Produto | Validar com time |
| Nome do campo backend para o delta (Δ) entre tabelas | DECISÃO AUTÔNOMA | Seção 4 — Convenções | Adotado `delta` (snake_case minúsculo). Alternativa descartada: `table_delta` (redundante, "tabela" já está implícito no contexto). | P2 | Engenharia | Validar com tech lead |
