# 10 - Glossário Técnico

## Módulo Cessionário · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia, Produto e QA |
| **Escopo** | Terminologia técnica e de negócio do módulo Cessionário |
| **Módulo** | Cessionário |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - Este glossário é a fonte única de verdade de terminologia para o módulo Cessionário.
> - A partir do D12 em diante, toda terminologia usada nos docs subsequentes deve estar alinhada com este glossário.
> - O glossário cobre: entidades de domínio, estados, integrações externas, termos técnicos de stack, siglas e acrônimos.
> - Novos termos identificados durante a geração dos docs de arquitetura devem ser adicionados neste doc.

---

## 1. Termos de Domínio de Negócio

| **Termo** | **Definição Técnica** | **Origem** |
|---|---|---|
| **Cessionário** | Investidor ou comprador que adquire direitos de um contrato de cessão imobiliária na plataforma Repasse Seguro. Representado pela entidade `cessionarios` no banco. | RN-001 (D01.1) |
| **Cedente** | Parte que transfere seus direitos contratuais ao Cessionário. Dados pessoais do Cedente são anonimizados estruturalmente — nunca chegam ao frontend do Cessionário. | RN-014 (D01.1) |
| **Oportunidade** | Contrato de cessão imobiliária disponível no marketplace, identificado por código anônimo (ex.: OPR-2026-0042). Entidade `opportunities` no banco. | RN-017 (D01.2) |
| **Proposta** | Oferta de compra enviada pelo Cessionário para uma oportunidade. Entidade `proposals` no banco. Máximo 3 simultâneas, 10 por 24h. | RN-020, RN-021 (D01.2) |
| **Negociação** | Processo iniciado automaticamente quando o Admin aceita uma proposta. Entidade `negotiations` no banco. Máximo 3 rodadas de contraproposta. | RN-024 (D01.2) |
| **Contraproposta** | Nova oferta de valor enviada pelo Cessionário durante uma negociação ativa. Armazenada como campo `counteroffer_rounds` na negociação. | RN-027 (D01.2) |
| **Escrow** | Conta-garantia onde o Cessionário deposita o valor total (Preço Repasse + Comissão Comprador) antes da formalização. Entidade `escrow_deposits` no banco. | RN-028 (D01.2) |
| **Formalização** | Processo de assinatura digital do instrumento de cessão via ZapSign. Entidade `formalizations` no banco. | RN-032 (D01.2) |
| **Fechamento** | Conclusão da operação após 4 critérios simultâneos: instrumento assinado + preço confirmado + anuência obtida + Escrow confirmado. | RN-035 (D01.2) |
| **Reversão** | Cancelamento de operação já concluída, disponível em até 15 dias corridos após o fechamento. | RN-036 (D01.2) |
| **Anuência** | Autorização formal da construtora/incorporadora para efetivação da cessão de direitos. | RN-034 (D01.2) |
| **Tabela Atual** | Valor atualizado do imóvel conforme tabela vigente da construtora. Coluna `current_table_value` na tabela `opportunities`. | Glossário D01.1 |
| **Tabela Contrato** | Valor original do imóvel conforme contrato de compra e venda do Cedente. Coluna `contract_table_value`. | Glossário D01.1 |
| **Δ (Delta)** | Diferença entre Tabela Atual e Tabela Contrato. Coluna gerada `delta_value = current_table_value - contract_table_value`. Base da Comissão Comprador. | Glossário D01.1 |
| **Comissão Comprador** | Taxa cobrada do Cessionário: 20% × Δ. Fallback quando Δ ≤ 0: 20% × Valor Pago pelo Cedente. Coluna `commission_buyer` em `proposals` e `negotiations`. | RN-038 (D01.2) |
| **Preço Repasse** | Valor negociado da cessão que será transferido ao Cedente após o fechamento. Coluna `proposed_value` em `proposals`, `agreed_value` em `negotiations`. | Glossário D01.1 |
| **KYC** | Know Your Customer — verificação de identidade obrigatória antes da primeira proposta. Entidade `kyc_documents`. Composta por: documento de identidade (frente e verso), comprovante de endereço e selfie de prova de vida. | RN-009 a RN-012 (D01.1) |
| **Analista de Oportunidades** | Agente de IA do módulo Cessionário. Tom analítico, orientado a dados. Sem acesso a dados do Cedente. Implementado com GPT-4 + LangChain.js + pgvector RAG. | RN-047 a RN-051 (D01.3) |
| **Score de Risco** | Avaliação de 1 a 10 gerada pela IA para cada oportunidade. 10 = menor risco. Coluna `ai_risk_score` em `opportunities`. | RN-048 (D01.3) |

---

## 2. Estados das Entidades

### 2.1 Estados do Cessionário (conta)

| **Estado** | **Constante de Código** | **Descrição** |
|---|---|---|
| Cadastrada | `CADASTRADA` | Conta criada, KYC não iniciado |
| KYC em Análise | `KYC_EM_ANALISE` | Documentos enviados, aguardando validação |
| KYC Aprovado | `KYC_APROVADO` | Identidade verificada — acesso completo |
| KYC Reprovado | `KYC_REPROVADO` | Documentos não aprovados |
| Bloqueada Temporariamente | `BLOQUEADA_TEMPORARIAMENTE` | Suspensa por Admin ou excesso de falhas |
| Encerrada | `ENCERRADA` | Conta excluída/anonimizada |

### 2.2 Estados da Oportunidade

| **Estado** | **Constante** | **Descrição** |
|---|---|---|
| Disponível | `DISPONIVEL` | Visível no marketplace |
| Com Proposta | `COM_PROPOSTA` | Ao menos uma proposta ativa |
| Em Negociação | `EM_NEGOCIACAO` | Proposta aceita, negociação em andamento |
| Reservada | `RESERVADA` | Escrow confirmado, formalização em curso |
| Concluída | `CONCLUIDA` | Fechamento realizado |
| Cancelada | `CANCELADA` | Operação cancelada — retorna ao marketplace |

### 2.3 Estados da Proposta

| **Estado** | **Constante** | **Conta no limite?** |
|---|---|---|
| Enviada | `ENVIADA` | Sim |
| Em Análise | `EM_ANALISE` | Sim |
| Aceita | `ACEITA` | Não |
| Recusada | `RECUSADA` | Não |
| Expirada | `EXPIRADA` | Não |
| Cancelada | `CANCELADA` | Não |

### 2.4 Estados da Negociação

| **Estado** | **Constante** |
|---|---|
| Em Negociação | `EM_NEGOCIACAO` |
| Em Contraproposta | `EM_CONTRAPROPOSTA` |
| Aguardando Depósito | `AGUARDANDO_DEPOSITO` |
| Depósito Confirmado | `DEPOSITO_CONFIRMADO` |
| Encerrada | `ENCERRADA` |
| Cancelada | `CANCELADA` |

### 2.5 Estados do Escrow

| **Estado** | **Constante** |
|---|---|
| Aguardando Depósito | `AGUARDANDO_DEPOSITO` |
| Depósito Enviado | `DEPOSITO_ENVIADO` |
| Depósito Confirmado | `DEPOSITO_CONFIRMADO` |
| Reembolsado | `REEMBOLSADO` |

### 2.6 Estados da Formalização

| **Estado** | **Constante** |
|---|---|
| Documentos Disponíveis | `DOCUMENTOS_DISPONIVEIS` |
| Assinatura Pendente (Cessionário) | `ASSINATURA_PENDENTE_CESSIONARIO` |
| Assinatura Pendente (Cedente) | `ASSINATURA_PENDENTE_CEDENTE` |
| Aguardando Anuência | `AGUARDANDO_ANUENCIA` |
| Concluída | `CONCLUIDA` |
| Cancelada | `CANCELADA` |

---

## 3. Termos Técnicos de Stack

| **Termo** | **Definição** | **Contexto** |
|---|---|---|
| **RLS (Row Level Security)** | Política do PostgreSQL/Supabase que filtra acesso a linhas de uma tabela com base em predicados por sessão. Usado para isolamento de dados por Cessionário. | RNF-003 (D05.5) |
| **service_role** | Chave do Supabase com privilégios elevados — ignora RLS. Usada exclusivamente no backend NestJS. Nunca exposta ao frontend. | Stacks D02 |
| **RAG (Retrieval Augmented Generation)** | Técnica de IA que combina busca vetorial (pgvector) com geração de texto (LLM) para respostas contextuais. Usada no Analista de Oportunidades. | Stacks D02 |
| **pgvector** | Extensão do PostgreSQL para armazenamento e busca de vetores. Índice HNSW como padrão. Embeddings: `text-embedding-3-small` (1536 dimensões). | Stacks D02 |
| **SSE (Server-Sent Events)** | Protocolo de streaming unidirecional servidor → cliente via HTTP. Usado para streaming de respostas do Analista de Oportunidades. | Stacks D02 |
| **Signed URL** | URL temporária e assinada gerada pelo Supabase Storage para acesso seguro a arquivos privados (documentos KYC, comprovantes). Expira após uso ou timeout. | Stacks D02 |
| **Soft delete** | Padrão de exclusão que não remove fisicamente o registro, mas seta `deleted_at`. Usado em todas as tabelas de domínio. | Stacks D02 |
| **UUID v4** | Formato de identificador único universal. PK padrão de todas as tabelas. Nunca auto-increment. | Stacks D02 |
| **AnimatePresence** | Componente do Framer Motion que gerencia ciclo de vida de animações de entrada e saída. Usado em todas as trocas de rota. | D04, Stacks D02 |
| **Skeleton Screen** | Placeholder animado exibido durante carregamento de conteúdo. Substitui spinners genéricos na plataforma. | D04, RF-052 |
| **Structured Output** | Modo da OpenAI API que força resposta em JSON schema válido. Obrigatório para score de risco e comparativos do Analista. | Stacks D02 |
| **Dead Letter Queue (DLQ)** | Fila de mensagens que falharam após todos os retentativas. Todo worker RabbitMQ deve ter DLQ configurado. | Stacks D02 |
| **Turborepo** | Ferramenta de build para monorepos. Cache inteligente de tasks. `turbo build` executa todos os workspaces. | Stacks D02 |
| **TanStack Query** | Biblioteca de data fetching com caching, sincronização e invalidação automática. Nunca usar `fetch` direto em `useEffect`. | Stacks D02 |

---

## 4. Siglas e Acrônimos

| **Sigla** | **Significado** | **Contexto** |
|---|---|---|
| RN | Regra de Negócio | Numeração: RN-001 a RN-072 no D01 |
| RF | Requisito Funcional | Numeração: RF-001 a RF-073 no D05 |
| RNF | Requisito Não Funcional | Numeração: RNF-001 a RNF-015 no D05.5 |
| SLA | Service Level Agreement | Compromisso de tempo para execução de processos |
| RACI | Responsible, Accountable, Consulted, Informed | Matriz de responsabilidades |
| LGPD | Lei Geral de Proteção de Dados (Lei 13.709/2018) | Regulação de dados pessoais no Brasil |
| KYC | Know Your Customer | Verificação de identidade |
| OCR | Optical Character Recognition | Leitura automática de documentos do KYC |
| FOMO | Fear Of Missing Out | Urgência artificial — proibida no UX Writing (RN-047) |
| RBAC | Role-Based Access Control | Controle de acesso por perfil |
| JWT | JSON Web Token | Token de autenticação |
| PIX | Pagamento Instantâneo | Sistema de pagamento instantâneo do BCB |
| TED | Transferência Eletrônica Disponível | Modalidade de transferência bancária |
| ADR | Architecture Decision Record | Documento de decisão técnica |
| SPA | Single Page Application | Tipo de aplicação web |
| SSE | Server-Sent Events | Streaming unidirecional servidor→cliente |
| DLQ | Dead Letter Queue | Fila de mensagens com falha |
| RLS | Row Level Security | Segurança em nível de linha no banco |
| p50/p95/p99 | Percentis de latência | Métricas de performance de API |
| MVP | Minimum Viable Product | Versão inicial do produto |
| BCB | Banco Central do Brasil | Regulador do sistema financeiro brasileiro |
| NOT-CES-XX | Notificação do Cessionário | Código de notificação (ex.: NOT-CES-01 = KYC aprovado) |

---

## 5. Integrações Externas

| **Serviço** | **Papel** | **SDK/Protocolo** | **Referência** |
|---|---|---|---|
| **Supabase** | Banco (PostgreSQL 17), Auth, Storage, Realtime, pgvector | Supabase JS Client, Prisma | Stacks D02 |
| **ZapSign** | Assinatura digital de documentos de formalização | REST API + Webhook | RN-063 (D01.5) |
| **idwall** | KYC — OCR de documentos brasileiros + liveness detection | REST API + Job assíncrono via RabbitMQ | RN-065 (D01.5) |
| **Resend** | E-mail transacional | SDK TypeScript nativo | RN-066 (D01.5) |
| **Celcoin** | Conta Escrow digital (MVP: confirmação manual; v2: automático) | REST API | RN-064 (D01.5) |
| **OpenAI** | LLM (GPT-4) + Embeddings (text-embedding-3-small) | openai SDK 4.x | Stacks D02 |
| **Langfuse** | Observabilidade de IA: latência, custo, tokens | Langfuse JS SDK 3.x | Stacks D02 |
| **Sentry** | Error tracking (backend NestJS + frontend React) | @sentry/nestjs, @sentry/react | Stacks D02 |
| **PostHog** | Analytics, session replay, feature flags | posthog-js | Stacks D02 |
| **Railway** | Hosting e auto-scaling do backend NestJS | Railway CLI | Stacks D02 |
| **Upstash** | Redis serverless (produção) | Upstash Redis SDK | Stacks D02 |
| **CloudAMQP** | RabbitMQ gerenciado (produção) | amqplib | Stacks D02 |

---

## 6. Changelog

| **Data** | **Versão** | **Descrição** |
|---|---|---|
| 22/03/2026 | v1.0 | Criação inicial — Pipeline ShiftLabs v9.5 |
