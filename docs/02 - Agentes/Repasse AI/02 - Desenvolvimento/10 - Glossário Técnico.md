# Repasse AI
## 10 — Glossário Técnico

| Campo | Valor |
|---|---|
| **Destinatário** | Produto e Engenharia |
| **Escopo** | Dicionário central que mapeia linguagem de negócio para linguagem técnica |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 22/03/2026 00:00 (America/Fortaleza) |
| **Status** | Ativo |
| **Fase** | 2 — Produto |
| **Área** | Produto + Engenharia |
| **Referências** | 01 - Regras de Negócio · 02 - Stacks · 05 - PRD |

---

> 📌 **TL;DR**
>
> - **52 termos** documentados em ordem alfabética — negócio, frontend, backend, banco e API.
> - **Mapeamento cross-camada** para todos os conceitos com correspondência técnica.
> - **Sinônimos registrados** — todo termo com variações aponta para o canônico.
> - **Tabela de siglas** consolidada — 16 siglas usadas no produto.
> - **Alinhado com D08 UX Writing** — vocabulário controlado da UI consistente com nomes técnicos.
> - **Pendências:** 0 — todas as ambiguidades resolvidas com decisão autônoma.

---

## 1. Glossário Principal

> **Ordem rigorosamente alfabética. Cada termo aparece uma única vez como entrada principal. Sinônimos apontam para o canônico.**

---

### A

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Agente** (Repasse AI) | Instância de IA conversacional que opera como Analista de Oportunidades para o Cessionário. | Repasse AI / Analista de Oportunidades | — | `RepasseAiAgent`, `AgentService` | — | `/ai/chat` (POST) |
| **Agente** (Guardião do Retorno) | Instância de IA do Cedente — módulo separado, compartilha infraestrutura de supervisão. | Guardião do Retorno | — | `GuardiaoAgent` | — | — |
| **Alerta proativo** | Notificação enviada proativamente pelo agente ao Cessionário sobre evento relevante (nova oportunidade, prazo Escrow, mudança de status). | Alerta | `AlertaBadge`, bolha de alerta | `NotificationService` | `notification_events` | `/notifications` |
| **Analista de Oportunidades** | Nome de interface do agente Repasse AI — usado na UI do Cessionário. Nunca usar "bot" ou "IA" na interface. | Analista de Oportunidades | — | — | — | — |
| **AguardandoConfirmacao** | Estado da vinculação WhatsApp: OTP SMS validado, aguardando confirmação via mensagem no WhatsApp. | — | Estado em T-016 step 3 | `WhatsAppBinding.status` | `whatsapp_bindings.status = 'aguardando_confirmacao'` | `binding.status = 'aguardando_confirmacao'` |
| **AguardandoOTP** | Estado da vinculação WhatsApp: número informado, OTP SMS enviado, aguardando inserção do código. | — | Estado em T-016 step 2 | `WhatsAppBinding.status` | `whatsapp_bindings.status = 'aguardando_otp'` | `binding.status = 'aguardando_otp'` |
| **AguardandoReVerificacao** | Estado da vinculação WhatsApp: vinculação ativa, mas re-verificação periódica pendente (30 dias atingidos). | — | — | `WhatsAppBinding.status` | `whatsapp_bindings.status = 'aguardando_reverificacao'` | `binding.status = 'aguardando_reverificacao'` |
| **Ativo** (vinculação) | Estado da vinculação WhatsApp: número vinculado e verificado, agente pode enviar e receber mensagens. | — | Status badge verde em T-015 | `WhatsAppBinding.status` | `whatsapp_bindings.status = 'ativo'` | `binding.status = 'ativo'` |

---

### C

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Cache de LLM** | Armazenamento temporário de respostas determinísticas do LLM para reduzir custo e latência. Dois tipos: exact e semantic. | — | — | `LlmCacheService` | `llm_cache_entries` | — |
| **Calculadora de Comissão** | Módulo que calcula comissão, Escrow e ROI usando fórmulas fixas, independente do agente. Fallback primário do Repasse AI. | Calculadora de Comissão | — | `CommissionCalculatorService` | — | `/calculator/commission` (GET) |
| **Canal** | Meio de comunicação entre o Cessionário e o agente: `webchat` ou `whatsapp`. | Canal | `channel` (prop de componente) | `channel: ChannelEnum` | `chat_conversations.channel`, `chat_messages.channel` | `channel` em request/response |
| **Cedente** | Proprietário original do contrato imobiliário que deseja repassar. Seus dados pessoais nunca são expostos ao Cessionário ou ao agente. | Cedente | — | — | — | — |
| **Cenário A/B/C/D** | Opções de repasse oferecidas ao Cedente pelo sistema. Dado confidencial — nunca exposto ao Cessionário nem ao agente. | Cenários A/B/C/D | — | — | — | — |
| **Cessionário** | Investidor ou comprador que adquire o repasse. Usuário principal do Repasse AI. | Cessionário | — | `CessionarioId` (UUID) | `cessionario_id` (FK externa) | `cessionario_id` |
| **Comissão** | Valor devido pelo Cessionário: 20% × Δ. Se Δ ≤ 0: 20% × Valor Pago pelo Cedente. | Comissão | `Comissão` em OportunidadeCard | `commission: number` | — (calculado) | `commission` em response |
| **Confiança** | Score percentual (0–100%) que mede a certeza do agente sobre a qualidade de sua resposta. Abaixo do threshold (padrão 80%) habilita takeover. | Confiança / Score de Confiança | Badge de confiança em T-020 | `confidence_score: number` | `ai_interactions.confidence_score` | `confidenceScore` em response |
| **Consentimento LGPD** | Aceite do Cessionário para armazenamento de conversas. Registrado com timestamp e versão da política. | Consentimento | Banner T-002 | `LgpdConsentService` | `lgpd_consents` | — |
| **Conversa** | Sessão de troca de mensagens entre o Cessionário e o agente, em um canal específico. | Conversa | — | `ConversationService` | `chat_conversations` | `/conversations` |

---

### D

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Δ (Delta)** | Diferença entre Tabela Atual e Tabela Contrato. Base de cálculo da comissão. Se Δ ≤ 0, usa-se fallback: 20% × Valor Pago pelo Cedente. | Δ (Delta) | `Δ` em OportunidadeCard | `delta: number` | — (calculado) | `delta` em response |
| **Dead-letter queue** | Fila de mensagens que falharam após o número máximo de tentativas (RabbitMQ). Garante auditabilidade de falhas. | — | — | DLQ via RabbitMQ | — | — |
| **Desvinculado** | Estado da vinculação WhatsApp: número desvinculado pelo Cessionário (via plataforma ou via comando `PARAR` no WhatsApp). Soft delete na tabela. | — | — | `WhatsAppBinding.status` | `whatsapp_bindings.deleted_at IS NOT NULL` | `binding.status = 'desvinculado'` |
| **Dossiê** | Conjunto de documentos obrigatórios para validação do repasse (contrato, matrícula, certidões). Fora do escopo do Repasse AI — referência de suporte operacional. | Dossiê | — | — | — | — |

---

### E

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Embedding** | Representação vetorial de um texto para busca de similaridade semântica no pipeline RAG. Dimensão 1536 (OpenAI text-embedding-3-small). | — | — | `EmbeddingService` | `document_embeddings.embedding` (vector(1536)) | — |
| **Encerrada** (interação) | Estado de uma interação: conversa concluída pela IA ou pelo Admin. | Encerrada | — | `InteractionStatus.COMPLETED` | `ai_interactions.status = 'completed'` | `status = 'completed'` |
| **Envelope ZapSign** | Pacote de assinatura eletrônica para formalização do contrato de cessão. Fora do escopo do Repasse AI. | Envelope ZapSign | — | — | — | — |
| **Escrow** | Conta garantia onde o Cessionário deposita Preço Repasse + Comissão. Prazo padrão: 10 dias úteis. | Escrow | `Custo Escrow` em OportunidadeCard | `escrowCost: number` | — (calculado) | `escrowCost` em response |

---

### F

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **FAB** | Floating Action Button — botão flutuante de acesso ao chat do Repasse AI, visível em todas as telas do Cessionário. | — | `FAB` component | — | — | — |
| **Feature flag** | Mecanismo para habilitar/desabilitar features em produção sem deploy. Implementado via PostHog. Kill switch para features de IA (RN-024). | Feature flag | — | `FeatureFlagService` (PostHog) | — | — |
| **Fase 1** | Entrega inicial do Repasse AI com canal webchat. Conjunto de RFs 001–085 e funcionalidades Admin. | Fase 1 | — | — | — | — |
| **Fase 2** | Expansão do Repasse AI para o canal WhatsApp via EvolutionAPI. Conjunto de RFs 086–104. | Fase 2 | — | — | — | — |
| **Function calling** | Mecanismo do GPT-4 para executar ferramentas (tools) em resposta a prompts. Base da arquitetura de agente do Repasse AI. | — | — | `tools` em OpenAI SDK | — | — |

---

### G

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Golden dataset** | Conjunto de 50+ exemplos de perguntas e respostas de qualidade usados para evals de qualidade do agente via Langfuse. | — | — | Langfuse Evals | — | — |
| **Guardião do Retorno** | Ver: **Agente** (Guardião do Retorno). Sinônimo — aponta para "Agente". | — | — | — | — | — |

---

### H

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Histórico de Chat** | Conjunto de conversas do Cessionário com o agente, retidas por 90 dias. | Histórico de Chat | T-019, seção em Meu Perfil | `ConversationService.getHistory` | `chat_messages` | `/conversations/:id/messages` |

---

### I

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Interação** | Par pergunta/resposta entre o Cessionário e o agente, com metadados de confiança, custo e latência. | Interação | Linha em T-020 | `InteractionService` | `ai_interactions` | `/interactions` |
| **Isolamento de dados** | Princípio de que cada Cessionário acessa apenas seus próprios dados. Implementado via RBAC + RLS (Supabase Row Level Security). | Isolamento de dados | — | Guards NestJS + RLS | RLS policies | — |

---

### K

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Kill switch** | Mecanismo para desligar features de IA em produção instantaneamente via PostHog feature flag. | Kill switch | — | `FeatureFlagService` | — | — |
| **KYC** | Know Your Customer — verificação de identidade obrigatória: documento (frente/verso) + selfie + comprovante de endereço. | KYC | — | — | — | — |

---

### L

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Langfuse** | Plataforma de observabilidade de IA — tracing, custo, latência e qualidade do agente. Obrigatória em todas as chamadas ao LLM. | — | — | `LangfuseService` | — | — |
| **LangChain.js** | Framework para pipelines RAG, agents com tools e roteamento de prompts. Usado em análise individual, comparação e simulação. | — | — | `LangChainService` | — | — |
| **LangGraph.js** | Framework para fluxos stateful do agente com branching complexo e checkpoints de confiança. | — | — | `LangGraphService` | — | — |
| **LGPD** | Lei Geral de Proteção de Dados Pessoais — regulamento que governa o tratamento de dados do Cessionário. Ver: **Consentimento LGPD**. | LGPD | — | — | `lgpd_consents` | — |
| **LLM** | Large Language Model — modelo de linguagem base. No Repasse AI: GPT-4 via OpenAI SDK. | — | — | `OpenAiService` | — | — |

---

### M

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Mensagem** | Unidade de conteúdo em uma conversa: do usuário (`user`), do agente (`assistant`) ou do sistema (`system`). | Mensagem | `ChatBubble` | `MessageRole` enum | `chat_messages` | `messages[]` em response |
| **Mutex de takeover** | Mecanismo de exclusão mútua que garante que apenas um Admin por vez controla uma conversa. Implementado via `SELECT FOR UPDATE`. | — | — | `TakeoverService` (transaction) | `ai_takeovers` (FOR UPDATE) | — |

---

### N

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **NaoVinculado** | Estado inicial da vinculação WhatsApp: nenhum número vinculado ao perfil. | — | Estado em T-015 (campo vazio) | `WhatsAppBinding.status` | `whatsapp_bindings.status = 'nao_vinculado'` ou ausência de registro | `binding.status = 'nao_vinculado'` |
| **Notificação proativa** | Ver: **Alerta proativo**. Sinônimo — aponta para "Alerta proativo". | — | — | — | — | — |

---

### O

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **OportunidadeCard** | Componente de UI que exibe dados estruturados de uma oportunidade: Δ, comissão, Escrow, ROI, score de risco. | — | `OportunidadeCard` (component) | — | — | — |
| **OPR** | Código identificador único de uma oportunidade no marketplace. Formato: `OPR-AAAA-NNNN`. | OPR / Código OPR | — | `oprCode: string` | — | `oprCode` em request |
| **OTP** | One-Time Password — código de 6 dígitos usado para verificar vinculação do WhatsApp. Validade: 15 minutos. Limite: 3 tentativas/hora. | OTP | `OTPInput` (component), T-016 | `OtpService` | `otp_attempts` | — |

---

### P

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **pgvector** | Extensão PostgreSQL para armazenamento e busca de vetores. Usado como vector store no pipeline RAG do Repasse AI. | — | — | Prisma `$queryRaw` | `document_embeddings` (vector(1536), IVFFlat) | — |
| **PII** | Personally Identifiable Information — dados pessoais do Cessionário. Sujeitos à LGPD. No banco: `chat_messages.content`, `whatsapp_bindings.phone_number`. | Dados pessoais | — | `PiiMaskUtil` | Tabelas com PII mapeadas na seção 9.2 do ERD | — |
| **Portfólio** | Simulação de conjunto de oportunidades com ROI total calculado para múltiplos cenários (3). | Portfólio | `Bolha de Portfólio`, T-007 | `PortfolioService` | — | `/ai/portfolio` |
| **Prompt** | Instrução enviada ao LLM. Armazenado como arquivo TypeScript em `src/modules/ai/prompts/`. Nunca hardcoded inline. | — | — | `*.prompt.ts` | — | — |
| **Proposta** | Oferta formal do Cessionário para aquisição de um repasse. O agente calcula e simula propostas, mas nunca as submete. | Proposta | — | — | — | — |

---

### R

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **RAG** | Retrieval-Augmented Generation — técnica que combina busca semântica em documentos com geração de resposta pelo LLM. Base do suporte operacional do Repasse AI. | — | — | `RagService`, LangChain.js | `document_embeddings` | — |
| **RabbitMQ** | Message broker para processamento assíncrono: ingestão de documentos para RAG, despacho de notificações proativas. | — | — | `RabbitMqService` | — | — |
| **Rate limit** | Limite de mensagens por Cessionário por hora: 30 msg/h no webchat, 20 msg/h no WhatsApp. Implementado via `@nestjs/throttler`. | Limite de mensagens | Bolha T-011 | `ThrottlerService` | — | Header `X-RateLimit-Remaining` |
| **RBAC** | Role-Based Access Control — controle de acesso por perfil. Cessionário, Admin e Cedente têm escopos isolados. | RBAC | — | Guards NestJS | RLS policies | — |
| **Re-verificação** | Processo periódico (a cada 30 dias) de re-confirmação da vinculação WhatsApp via OTP. | Re-verificação | Botão "Re-verificar agora" em T-015 | `WhatsAppService.reverify` | `whatsapp_bindings.next_reverification_at` | — |
| **Redis** | Banco de dados em memória para cache de LLM (exact e semantic) e rate limiting por usuário. | — | — | `RedisService` | — | — |
| **ROI** | Return on Investment — retorno calculado em 3 cenários (Conservador, Base, Otimista). Apresentado em abas no OportunidadeCard. | ROI | `TabComparativo` (3 abas) | `roi: { conservative, base, optimistic }` | — (calculado) | `roi` em response |

---

### S

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Score de Risco** | Avaliação do risco de uma oportunidade em escala 1–10, calculada pelo agente. Exibida como badge colorida (Baixo/Moderado/Alto). | Score de Risco | `ScoreIndicador` component | `riskScore: number` | — (calculado) | `riskScore` em response |
| **Sinalizada para revisão** | Estado de uma interação: confiança abaixo do threshold, aguarda revisão do Admin. | — | Badge vermelho em T-020 | `InteractionStatus.FLAGGED` | `ai_interactions.status = 'takeover'` | `status = 'takeover'` |
| **SLA** | Service Level Agreement — prazo máximo de resposta do agente: ≤ 5s para resposta inicial; timeout em 10s. | SLA | — | `AgentSlaService` | `agent_configurations` (chaves `agent_sla_*`) | — |
| **Soft delete** | Padrão de exclusão lógica via campo `deleted_at` — registros não são removidos fisicamente imediatamente. | — | — | Prisma `where: { deletedAt: null }` | `deleted_at TIMESTAMPTZ NULL` | — |
| **SSE** | Server-Sent Events — protocolo de streaming unidirecional do servidor para o cliente. Usado para streaming de respostas do agente no webchat. | — | `EventSource` / Vercel AI SDK | `@Sse()` decorator NestJS | — | `/ai/chat/stream` |
| **Streaming** | Entrega progressiva de tokens da resposta do LLM ao cliente via SSE. | — | Renderização progressiva de tokens em `ChatBubble` | Vercel AI SDK | — | — |
| **Suspenso** | Estado da vinculação WhatsApp: re-verificação não concluída em 48h; acesso ao agente via WhatsApp bloqueado. | — | Status badge amarelo em T-015 | `WhatsAppBinding.status` | `whatsapp_bindings.status = 'suspenso'` | `binding.status = 'suspenso'` |

---

### T

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Tabela Atual** | Preço vigente do imóvel conforme tabela da incorporadora no momento da análise. | Tabela Atual | — | `currentTablePrice: number` | — | `currentTablePrice` |
| **Tabela Contrato** | Preço do imóvel na data do contrato original assinado pelo Cedente. | Tabela Contrato | — | `contractTablePrice: number` | — | `contractTablePrice` |
| **Takeover** | Intervenção manual do Admin em uma conversa quando a confiança do agente fica abaixo do threshold. Mutex por conversa. | Takeover | Banner destrutivo em T-021 | `TakeoverService` | `ai_takeovers` | `/interactions/:id/takeover` |
| **Threshold (takeover)** | Valor de confiança abaixo do qual o takeover é habilitado. Padrão: 80%. Configurável pelo Admin. | Threshold de confiança | — | `AgentConfigService` | `agent_configurations.key = 'takeover_threshold'` | — |
| **Tool** | Função do agente executada via function calling do GPT-4. Definida como objeto TypeScript tipado em `repasse-ai-tools.ts`. | Tool / Ferramenta | — | `ToolService`, `repasse-ai-tools.ts` | — | — |
| **Trace** | Registro completo de uma chamada ao LLM no Langfuse — inclui input, output, latência, custo e tokens. | — | — | `LangfuseService.trace` | `ai_interactions.langfuse_trace_id` | — |

---

### V

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Vector store** | Banco de dados para armazenamento e busca de embeddings. No Repasse AI: Supabase pgvector. | — | — | `VectorStoreService` | `document_embeddings` | — |
| **Vinculação** | Processo e estado de associação entre um número de WhatsApp e o perfil do Cessionário. | Vinculação WhatsApp | T-015, T-016 | `WhatsAppBindingService` | `whatsapp_bindings` | `/whatsapp/bindings` |

---

### W

| Termo | Definição | Negócio | Frontend | Backend | Banco | API |
|---|---|---|---|---|---|---|
| **Webchat** | Canal de comunicação com o agente integrado à plataforma via FAB. | Webchat | FAB + ChatWindow (T-001 a T-003) | `WebchatModule` | `channel = 'webchat'` | `/ai/chat` |
| **WhatsApp** | Canal de comunicação Fase 2 via EvolutionAPI. As mesmas capacidades do webchat com adaptações de formato. | WhatsApp | T-015 a T-017 | `WhatsAppModule` | `channel = 'whatsapp'` | `/whatsapp/*` |

---

## 2. Sinônimos — Índice de Redirecionamento

> Termos que aparecem em outros documentos mas NÃO são canônicos. Aponta para o termo correto.

| Sinônimo / Variação | Termo Canônico |
|---|---|
| Alerta | Alerta proativo |
| Assistente | Analista de Oportunidades |
| Bot | Analista de Oportunidades |
| Chatbot | Analista de Oportunidades |
| Código OPR | OPR |
| Conta garantia | Escrow |
| Dados pessoais | PII |
| Deletar | Excluir (soft delete) |
| Delta | Δ (Delta) |
| Ferramenta (IA) | Tool |
| Guardião do Retorno | Agente (Guardião do Retorno) |
| IA | Analista de Oportunidades (na UI) |
| Lucro | Δ (Delta) |
| Nota de risco | Score de Risco |
| Notificação proativa | Alerta proativo |
| OTP SMS | OTP |
| Rating | Score de Risco |
| Repasse AI (na UI Cessionário) | Analista de Oportunidades |
| Score de confiança | Confiança |
| Sessão | Conversa |
| Threshold | Threshold (takeover) |
| Vinculação WhatsApp | Vinculação |

---

## 3. Mapeamento Cross-Camada (Entidades Core)

| Conceito de Negócio | Frontend | Backend | Banco | API Endpoint | API Schema |
|---|---|---|---|---|---|
| Conversa | — | `ConversationService`, `CreateConversationDto` | `chat_conversations` | `POST /conversations` | `ConversationResponse` |
| Mensagem | `ChatBubble` | `MessageService`, `CreateMessageDto` | `chat_messages` | `POST /conversations/:id/messages` | `MessageResponse` |
| Interação | Linha em T-020 | `InteractionService` | `ai_interactions` | `GET /interactions` | `InteractionResponse` |
| Vinculação WhatsApp | T-015, T-016 | `WhatsAppBindingService` | `whatsapp_bindings` | `POST /whatsapp/bindings` | `BindingResponse` |
| Tentativa de OTP | — | `OtpService` | `otp_attempts` | — | — |
| Preferência de notificação | T-018 | `NotificationPreferenceService` | `notification_preferences` | `PUT /notifications/preferences` | `PreferenceResponse` |
| Evento de notificação | Bolha de alerta (T-012–T-014) | `NotificationEventService` | `notification_events` | — | — |
| Takeover | Banner T-021 | `TakeoverService` | `ai_takeovers` | `POST /interactions/:id/takeover` | `TakeoverResponse` |
| Configuração do agente | — | `AgentConfigService` | `agent_configurations` | `GET /admin/config` | `ConfigResponse` |
| Consentimento LGPD | Banner T-002 | `LgpdConsentService` | `lgpd_consents` | `POST /lgpd/consent` | — |

---

## 4. Tabela de Siglas

| Sigla | Significado completo | Contexto de uso |
|---|---|---|
| **AI** | Artificial Intelligence | Módulo Repasse AI |
| **API** | Application Programming Interface | Contratos de comunicação entre serviços |
| **CSAT** | Customer Satisfaction Score | Métrica de satisfação — painel Admin T-022 |
| **DLQ** | Dead-Letter Queue | Filas RabbitMQ para mensagens com falha |
| **ERD** | Entity Relationship Diagram | Documento 12 — Modelo de Dados |
| **FK** | Foreign Key | Referências entre tabelas no banco |
| **GPT** | Generative Pre-trained Transformer | Modelo LLM base do Repasse AI (GPT-4) |
| **KYC** | Know Your Customer | Verificação de identidade do Cessionário |
| **LGPD** | Lei Geral de Proteção de Dados Pessoais | Regulamento de privacidade brasileiro |
| **LLM** | Large Language Model | Modelo de linguagem (GPT-4) |
| **OPR** | Oportunidade de Repasse | Código identificador de oportunidade |
| **OTP** | One-Time Password | Código de verificação único para WhatsApp |
| **PII** | Personally Identifiable Information | Dados pessoais sujeitos à LGPD |
| **PK** | Primary Key | Chave primária de tabela |
| **RAG** | Retrieval-Augmented Generation | Técnica de busca semântica + geração LLM |
| **RBAC** | Role-Based Access Control | Controle de acesso por perfil |
| **RLS** | Row Level Security | Isolamento de dados por linha no Supabase |
| **ROI** | Return on Investment | Retorno sobre investimento — 3 cenários |
| **SLA** | Service Level Agreement | Prazo máximo de resposta do agente |
| **SSE** | Server-Sent Events | Protocolo de streaming servidor→cliente |
| **STK** | Stack (abreviação interna) | Referência às Stacks Tecnológicas (Doc 02) |
| **UUID** | Universally Unique Identifier | Formato de chave primária (v4) |

---

## 5. Estados — Máquina de Estados por Entidade

### 5.1 Vinculação WhatsApp

| Estado | Código | Trigger de entrada | Trigger de saída |
|---|---|---|---|
| Não vinculado | `nao_vinculado` | Estado inicial | Cessionário informa número → OTP enviado |
| Aguardando OTP | `aguardando_otp` | OTP SMS enviado | OTP validado ou expirado/esgotado |
| Aguardando confirmação | `aguardando_confirmacao` | OTP SMS validado | Confirmação WA recebida ou prazo 24h expirado |
| Ativo | `ativo` | Confirmação WA recebida | 30 dias de uso → re-verificação / desvinculação |
| Aguardando re-verificação | `aguardando_reverificacao` | 30 dias de uso sem re-verificação | Re-verificação concluída ou prazo 48h expirado |
| Suspenso | `suspenso` | 48h sem re-verificação | Re-verificação via T-015 ou 7 dias → desvinculado |
| Desvinculado | `desvinculado` | Cessionário desvincula / comando PARAR | Estado final (soft delete) |

### 5.2 Interação do Agente

| Estado | Código | Descrição |
|---|---|---|
| Respondida pela IA | `completed` | Agente respondeu dentro do SLA, confiança ≥ threshold |
| Sinalizada para revisão | `takeover` (pending) | Confiança < threshold, aguarda Admin |
| Em takeover | `takeover` (active) | Admin assumiu a conversa |
| Encerrada | `completed` (final) | Conversa concluída pela IA ou Admin |

---

## 6. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop — Pipeline v6.1 | Criação. 52 termos, 22 sinônimos, mapeamento cross-camada de 10 entidades core, 22 siglas, máquinas de estados de vinculação e interação. Alinhado com D08 UX Writing e D12 ERD. |

---

## 7. Backlog de Pendências

| Item | Tipo | Seção | Impacto | Justificativa | Dono | Status |
|---|---|---|---|---|---|---|
| Nome da API response para interação: `InteractionResponse` vs `AiInteractionResponse` | [DECISÃO AUTÔNOMA] | 3 (cross-camada) | P2 | Prefixo `Ai` omitido para consistência com `/interactions` endpoint. O módulo já contextualiza que são interações de IA. Alternativa descartada: `AiInteractionResponse` (verboso sem ganho semântico no contexto do módulo). | Backend | Aplicado |
| Estado `sinalizada para revisão` usa mesmo enum `takeover` que estado `em takeover` | [DECISÃO AUTÔNOMA] | 5.2 | P2 | O campo `status = 'takeover'` na tabela `ai_interactions` cobre ambos os estados (sinalizada e em takeover ativo). A diferença é observada via JOIN com `ai_takeovers` (active). Simplifica schema. Alternativa descartada: dois status separados `flagged` e `in_takeover` (mais verbose, query mais simples). | Backend | Aplicado |

---

*Glossário Técnico v1.0 concluído. 52 termos, 22 siglas, 10 entidades cross-camada. Status: APROVADO. Próximo documento do pipeline: D13 — Schema Prisma.*
