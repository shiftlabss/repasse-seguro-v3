# 10 - Glossário Técnico

## Módulo Cedente · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia, Produto e QA |
| **Escopo** | Terminologia técnica e de negócio do módulo Cedente |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 (America/Fortaleza) |
| **Referências** | 01.1/01.2 - Regras de Negócio · 02 - Stacks · 03 - Brand Guide · 06 - Mapa de Telas |

---

> 📌 **TL;DR**
>
> - Este glossário é a fonte única de verdade de terminologia para o módulo Cedente.
> - A partir do D12 em diante, toda terminologia usada nos docs subsequentes deve estar alinhada com este glossário.
> - O glossário cobre: entidades de domínio de negócio, estados e enums, integrações externas, termos técnicos de stack, siglas e acrônimos, e convenções de código.
> - Termos específicos do Cedente diferem dos termos do Cessionário mesmo quando referem-se ao mesmo conceito — veja coluna "Perspectiva do Cessionário" onde aplicável.
> - Novos termos identificados durante a geração dos docs de arquitetura devem ser adicionados neste glossário.

---

## 1. Termos de Domínio de Negócio

| **Termo** | **Definição técnica** | **Entidade no DB** | **Perspectiva do Cessionário** | **Origem** |
|---|---|---|---|---|
| **Cedente** | Pessoa física (CPF) ou jurídica (CNPJ) que possui um contrato imobiliário e deseja repassá-lo via plataforma. Usuário principal deste módulo. Representado pela entidade `cedentes` no banco. | `cedentes` | Parte anônima da transação — dados do Cedente são ocultados estruturalmente no frontend do Cessionário | RN-001 (D01.1) |
| **Cessionário** | Parte que adquire o direito do contrato cedido. O Cedente nunca vê dados pessoais do Cessionário na plataforma. Representado como entidade opaca do ponto de vista do módulo Cedente. | `cessionarios` | Usuário principal do módulo Cessionário | RN-012 (D01.1) |
| **Caso** | Unidade de trabalho do Cedente. Cada imóvel cadastrado gera um caso com ciclo de vida próprio. Entidade `cases` no banco. Um Cedente pode ter múltiplos casos; um imóvel só pode ter um caso ativo por vez (RN-090). | `cases` | "Oportunidade" — identificada por código anônimo (ex: OPR-2026-0042) sem revelar o endereço real | RN-019, RN-090 (D01.1) |
| **Dossiê** | Conjunto de documentos obrigatórios vinculados ao caso. Criado automaticamente pelo sistema ao criar o caso. Composto por 6 documentos (Cedente PF) ou 8 documentos (Cedente PJ). Entidade `dossiers` no banco, com relação 1:1 com `cases`. | `dossiers`, `documents` | Não visível — documentos são verificados pelo Admin antes de publicar a oportunidade | RN-040 a RN-044 (D01.1) |
| **Cenário de Retorno** | Uma das 4 opções (A, B, C, D) que define o quanto o Cedente espera receber. Escolhido no wizard de cadastro (Etapa 4). Associado ao caso e imutável — mudanças são sempre descendentes (escalonamento). Coluna `cenario` na tabela `cases`. | `cases.cenario` (enum: `A`, `B`, `C`, `D`) | "Faixa de preço esperada" — visível indiretamente pelo preço da oportunidade | RN-022, RN-025 (D01.2) |
| **Cenário A** | Transferência do saldo devedor sem retorno adicional ao Cedente. Comissão = R$ 0. Maior facilidade para encontrar comprador. | `cases.cenario = 'A'` | Oportunidade com menor preço relativo | RN-021 (D01.2) |
| **Cenário B** | Retorno de 100% do valor pago pelo Cedente. Comissão = 20% × (Valor Recuperado − Valor Distrato Referência). | `cases.cenario = 'B'` | Oportunidade com preço moderado | RN-021 (D01.2) |
| **Cenário C** | Retorno de 100% do valor pago + 30%. Comissão = 20% × (Valor Recuperado − Valor Distrato Referência). | `cases.cenario = 'C'` | Oportunidade com preço acima da média | RN-021 (D01.2) |
| **Cenário D** | Retorno de 100% do valor pago + 50%. Comissão = 20% × (Valor Recuperado − Valor Distrato Referência). Mais difícil de vender. | `cases.cenario = 'D'` | Oportunidade com preço premium | RN-021 (D01.2) |
| **Escalonamento** | Processo de descida de cenário (D→C, C→B, B→A). Sempre descendente, um nível por vez, com cooldown de 7 dias entre escalonamentos. Exige assinatura do Termo de Aceite de Escalonamento. Coluna `historico_escalonamentos` (JSONB) em `cases`. | `cases.historico_escalonamentos` (JSONB array) | N/A — opaco para o Cessionário | RN-025 a RN-029 (D01.2) |
| **Proposta** | Oferta de valor feita por um Cessionário (anônimo) para adquirir o repasse do caso. Apresentada pelo Admin ao Cedente. Entidade `proposals` no banco. | `proposals` | Mesma entidade — perspectiva invertida | RN-030 a RN-036 (D01.2) |
| **Contraproposta** | Nova oferta de valor enviada pelo Cedente em resposta a uma proposta, dentro do piso do cenário ativo. Máx 3 rodadas de negociação. Campo `rodadas_negociacao` (JSONB) em `proposals`. | `proposals.rodadas_negociacao` (JSONB) | Mesmo conceito — perspectiva invertida | RN-035 (D01.2) |
| **Valor Recuperado** | Valor efetivo que o Cedente recebe no fechamento. Coluna `valor_recuperado` em `cases`. | `cases.valor_recuperado` | "Preço Repasse" | RN-021 (D01.2) |
| **Valor Distrato Referência** | Valor que o Cedente receberia no distrato com a construtora = 50% do valor pago. Coluna `valor_distrato_referencia` em `cases`. Calculado na Etapa 2 do wizard. | `cases.valor_distrato_referencia` | N/A — cálculo interno | Glossário D01.1 |
| **Valor Líquido** | O que o Cedente efetivamente recebe = Valor Recuperado − Comissão RS. Coluna calculada `valor_liquido` em `cases`. Sempre exibido em destaque máximo na interface. | `cases.valor_liquido` (generated column) | "Preço Repasse" que o Cessionário paga ao Cedente | D01.1 |
| **Comissão RS** | Taxa cobrada pelo Repasse Seguro do Cedente. Fórmula B/C/D: `20% × (Valor Recuperado − Valor Distrato Referência)`. Cenário A: R$ 0. Coluna `comissao_rs` em `cases`. | `cases.comissao_rs` | "Comissão Vendedor" | RN-021 (D01.2) |
| **Conta Escrow** | Conta-garantia operada por parceiro bancário/fintech regulado. Recebe o depósito do Cessionário, retém por 15 dias corridos após o Fechamento e libera automaticamente ao Cedente. Entidade `escrow_accounts` no banco. | `escrow_accounts` | `escrow_deposits` — perspectiva de quem deposita | RN-037 a RN-039 (D01.2) |
| **Período de Reversão** | 15 dias corridos após o Fechamento durante os quais o Cedente pode solicitar desistência. Coluna `prazo_reversao` em `cases`. | `cases.prazo_reversao` (timestamp) | Mesma entidade, perspectiva do Cessionário | RN-040 (D01.2) |
| **Reversão** | Cancelamento do negócio dentro do Período de Reversão. Estorna valores integralmente da Conta Escrow. | `cases.status = 'EM_REVERSAO'` / `'CANCELADO'` | Mesmo conceito | RN-040 (D01.2) |
| **Fechamento** | Evento que confirma o negócio. Exige 4 critérios simultâneos: (1) instrumento assinado pelas partes; (2) preço confirmado por documento; (3) anuência da construtora; (4) depósito confirmado na Conta Escrow. | `cases.status = 'FECHAMENTO'` | Mesmo evento | Glossário D01.1 |
| **Anuência** | Autorização formal da construtora para transferir o contrato. Sem anuência, o Fechamento não ocorre (exceto contratos com cessão livre). Coluna `anuencia_status` em `cases`. | `cases.anuencia_status` | N/A — gerenciado pelo Admin | RN-033 (D01.2) |
| **Guardião do Retorno** | Agente de inteligência artificial do módulo Cedente. Tom empático, orientado a resultados financeiros. Disponível 24h. Implementado com GPT-4 + LangChain.js + pgvector RAG. Nunca tem acesso a dados do Cessionário. | N/A — serviço externo via API | "Analista de Oportunidades" é o agente equivalente no módulo Cessionário | RN-058 a RN-063 (D01.1) |
| **Termo de Cadastro** | Documento gerado automaticamente ao confirmar o cadastro do imóvel (Etapa 5 do wizard). Inclui declaração de adimplência e autorização para intermediação. Gerado via ZapSign. | `signature_envelopes.tipo = 'TERMO_CADASTRO'` | N/A | RN-024 (D01.2) |
| **Termo de Aceite de Escalonamento** | Documento gerado a cada escalonamento. Registra o aceite do Cedente sobre a descida de cenário. Gerado via ZapSign. | `signature_envelopes.tipo = 'TERMO_ESCALONAMENTO'` | N/A | RN-025 (D01.2) |
| **Termo Comercial** | Documento gerado antes do Fechamento. Define comissões e condições acordadas. | `signature_envelopes.tipo = 'TERMO_COMERCIAL'` | Idem | D01.1 |
| **Instrumento de Cessão** | Documento gerado no Fechamento que formaliza a transferência do contrato. Deve ser assinado por todas as partes. | `signature_envelopes.tipo = 'INSTRUMENTO_CESSAO'` | Idem | Glossário D01.1 |
| **Rascunho** | Cadastro de imóvel iniciado e não concluído. Salvo automaticamente ao final de cada etapa concluída no wizard. Expiração: 30 dias corridos. | `cases.status = 'RASCUNHO'` | N/A | RN-023 (D01.2) |
| **Cedente PF** | Cedente pessoa física, identificado por CPF. Dossiê com 6 documentos obrigatórios. | `cedentes.tipo_pessoa = 'PF'` | N/A | RN-001 (D01.1) |
| **Cedente PJ** | Cedente pessoa jurídica, identificado por CNPJ, com Representante Legal designado que assina documentos e executa ações operacionais. Dossiê com 8 documentos obrigatórios. | `cedentes.tipo_pessoa = 'PJ'` | N/A | RN-001 (D01.1) |
| **Representante Legal** | Pessoa física que representa o Cedente PJ na plataforma. Assina documentos e executa ações operacionais em nome da empresa. | `cedentes.representante_legal` (JSONB) | N/A | D01.1 |

---

## 2. Estados das Entidades

### 2.1 Estados da Conta do Cedente

| **Estado** | **Constante de código** | **Descrição** |
|---|---|---|
| Inexistente | N/A | Conta não criada |
| Pendente de ativação | `PENDENTE_ATIVACAO` | Cadastro realizado, e-mail não confirmado |
| Ativo | `ATIVO` | E-mail confirmado — acesso completo |
| Bloqueado temporariamente | `BLOQUEADO_TEMPORARIAMENTE` | 5 tentativas de login falhas — bloqueio de 15 minutos |
| Suspenso | `SUSPENSO` | Suspenso manualmente pelo Admin |
| Encerrado | `ENCERRADO` | Conta excluída/anonimizada (LGPD) |

### 2.2 Estados do Caso (Visão do Cedente)

| **Estado (UI)** | **Constante de código** | **Estado interno Admin** | **Terminal?** |
|---|---|---|---|
| Cadastro realizado | `CAPTADO` | Captado | Não |
| Em análise | `EM_TRIAGEM` | Em Triagem | Não |
| Pendência identificada | `BLOQUEADO` | Bloqueado | Não |
| Aprovado para oferta | `QUALIFICADO` | Qualificado | Não |
| Disponível para compradores | `OFERTA_ATIVA` | Oferta Ativa | Não |
| Proposta recebida | `EM_NEGOCIACAO` | Em Negociação | Não |
| Em formalização | `EM_FORMALIZACAO` | Em Formalização | Não |
| Negócio fechado | `FECHAMENTO` | Fechamento | Não |
| Aguardando liberação | `POS_FECHAMENTO` | Pós Fechamento | Não |
| Desistência em análise | `EM_REVERSAO` | Em Reversão | Não |
| Mediação em andamento | `EM_MEDIACAO` | Em Mediação | Não |
| Concluído | `CONCLUIDO` | Concluído | **Sim** |
| Cancelado | `CANCELADO` | Cancelado | **Sim** |

### 2.3 Estados do Documento (Dossiê)

| **Estado** | **Constante** | **Ação do Cedente** |
|---|---|---|
| Pendente | `PENDENTE` | Enviar |
| Em análise | `EM_ANALISE` | Aguardar |
| Verificado | `VERIFICADO` | Nenhuma (somente leitura) |
| Rejeitado | `REJEITADO` | Reenviar |

### 2.4 Estados da Proposta

| **Estado** | **Constante** | **Conta no prazo?** |
|---|---|---|
| Aguardando resposta | `AGUARDANDO_RESPOSTA` | Sim — timer de 5 dias úteis |
| Contraproposta enviada | `CONTRAPROPOSTA_ENVIADA` | Sim — timer reiniciado |
| Aceita | `ACEITA` | Não — terminal |
| Recusada | `RECUSADA` | Não — terminal |
| Expirada | `EXPIRADA` | Não — terminal (sem resposta em 5 dias úteis) |
| Superada | `SUPERADA` | Não — terminal (outra proposta aceita) |

### 2.5 Estados da Conta Escrow

| **Estado** | **Constante** | **Ação do Cedente** |
|---|---|---|
| Não iniciada | `NAO_INICIADA` | Nenhuma — aguardar formalização |
| Aguardando depósito | `AGUARDANDO_DEPOSITO` | Nenhuma — depósito é ação do Cessionário |
| Depósito confirmado | `DEPOSITO_CONFIRMADO` | Nenhuma — Fechamento em andamento |
| Período de segurança | `POS_FECHAMENTO` | Solicitar desistência (opcional, dentro de 15 dias) |
| Distribuída | `DISTRIBUIDA` | Nenhuma — valores liberados |
| Em reversão | `EM_REVERSAO` | Aguardar processamento |
| Estornada | `ESTORNADA` | Nenhuma — valores devolvidos |

### 2.6 Estados do Envelope de Assinatura

| **Estado** | **Constante** | **Ação do Cedente** |
|---|---|---|
| Aguardando assinatura | `AGUARDANDO` | Assinar via ZapSign |
| Assinado | `ASSINADO` | Nenhuma (somente leitura) |
| Expirado | `EXPIRADO` | Contatar suporte para novo envelope |

---

## 3. Integrações Externas

| **Serviço** | **Propósito no módulo Cedente** | **Autenticação** | **Timeout** | **Fallback** |
|---|---|---|---|---|
| **ZapSign** | Geração e assinatura de documentos eletrônicos (Termo de Cadastro, Instrumento de Cessão, Termos de Escalonamento, Termo Comercial). Embedado via iframe com postMessage. | API Key no backend (nunca exposta ao frontend) | 10s para carregamento do iframe | Mensagem de indisponibilidade + retry |
| **Supabase Auth** | Cadastro, ativação por e-mail, login, recuperação de senha, bloqueio por tentativas, expiração de sessão. | Service Role Key (backend) / Anon Key (frontend) | Padrão Supabase (30s) | Mensagem de erro genérica |
| **Supabase Storage** | Upload de documentos do dossiê (PDF, JPG, PNG, max 10 MB). Validação de MIME type obrigatória no backend. | Signed URLs geradas no backend | 30s | Retry 1x, mensagem de falha |
| **Supabase Realtime** | Atualização em tempo real: status de documentos, notificações no painel, status do caso. Prazo máximo: 30 segundos (RN-057). | Anon Key com RLS | N/A (WebSocket persistente) | Polling a cada 60s como fallback |
| **OpenAI GPT-4** | LLM do Guardião do Retorno. Streaming via SSE. Contexto: documentação do caso + RAG via pgvector. | API Key no backend | 30s (timeout do streaming) | "Não consegui processar. Tente novamente." |
| **LangChain.js** | Orquestração do Guardião: RAG, memória de conversa, LangGraph.js para modo de cadastro assistido. | N/A (biblioteca) | N/A | N/A |
| **Resend** | E-mails transacionais (ativação, recuperação de senha, notificações, lembretes de rascunho). Prazo de entrega: 5 minutos (RN-056). | API Key no backend | N/A (assíncrono via fila) | Dead-letter queue no RabbitMQ |
| **PostHog** | Analytics de comportamento do usuário. Obrigatório desde o lançamento. Rastreia funil de cadastro, abandono de wizard, aceitação de propostas. | API Key pública (frontend) + API Key privada (backend) | N/A (fire-and-forget) | Silently fail — sem impacto no produto |
| **Sentry** | Monitoramento de erros em todas as camadas (frontend Next.js, backend NestJS, mobile Expo). | DSN público (frontend) + DSN privado (backend) | N/A (fire-and-forget) | Silently fail |
| **Upstash Redis** | Cache de dados críticos: sessões, piso de cenário, status de escalonamento enfileirado. | REST API Key | N/A | Recalcular ou ler do banco |
| **CloudAMQP RabbitMQ** | Filas assíncronas: envio de e-mails, descarte de rascunhos, lembretes de prazo, notificações. | AMQP URL com credenciais | N/A | Dead-letter queue |

---

## 4. Termos Técnicos de Stack

### 4.1 Backend

| **Termo** | **Definição** | **Versão no módulo** |
|---|---|---|
| **NestJS** | Framework Node.js para backend. Arquitetura modular, decorators TypeScript, injeção de dependência nativa. Framework único — Express puro, Fastify e Hono são proibidos sem ADR. | 10.x+ |
| **Prisma** | ORM TypeScript para PostgreSQL. Migrations declarativas, tipagem auto-gerada, integração com Supabase. TypeORM, Sequelize e Drizzle são proibidos. | 6.x+ |
| **Prisma Schema** | Arquivo `.prisma` que define as entidades, relações e migrações do banco. Convenções: UUID v4 como PK, snake_case, `created_at`/`updated_at` em todas as tabelas, soft delete via `deleted_at`. | N/A |
| **DTO (Data Transfer Object)** | Classe TypeScript com decorators `class-validator` usada para validar entrada e saída de dados em controllers NestJS. Todo endpoint tem DTO de entrada. | N/A |
| **Guard (NestJS)** | Mecanismo de autorização de rotas no NestJS. `JwtAuthGuard` valida JWT, `RbacGuard` valida role `CEDENTE`. | N/A |
| **Pipe (NestJS)** | Transformação e validação de dados antes de chegar ao controller. `ValidationPipe` global obrigatório. | N/A |
| **Interceptor (NestJS)** | Middleware de cross-cutting concerns: logging de requests, transformação de resposta, rate limiting. | N/A |
| **Module (NestJS)** | Unidade de organização de código. Um módulo por bounded context: `CedentesModule`, `CasesModule`, `ProposalsModule`, `SignaturesModule`, `EscrowModule`, `GuardiaoModule`, `NotificationsModule`. | N/A |
| **Dead-letter Queue (DLQ)** | Fila de mensagens que falharam após todas as tentativas de retry no RabbitMQ. Mensagens na DLQ geram alerta de monitoramento. | N/A |
| **Soft Delete** | Exclusão lógica via coluna `deleted_at`. Registros com `deleted_at IS NOT NULL` são filtrados automaticamente pelo Prisma com middleware. Obrigatório em todas as tabelas de domínio. | N/A |
| **Row Level Security (RLS)** | Política de segurança do PostgreSQL/Supabase que garante isolamento de dados entre Cedentes. Cada Cedente só acessa suas próprias rows (RN-011). | N/A |
| **pgvector** | Extensão PostgreSQL para armazenamento e busca de vetores de embeddings. Usada pelo Guardião do Retorno para RAG (Retrieval Augmented Generation). | N/A |

### 4.2 Frontend Web

| **Termo** | **Definição** | **Versão no módulo** |
|---|---|---|
| **Next.js App Router** | Sistema de roteamento baseado em sistema de arquivos do Next.js 15. Estrutura de rotas: `app/(public)/` para autenticação, `app/(authenticated)/` para área logada. | 15.x+ |
| **Route Group** | Agrupamento de rotas no App Router sem afetar o path da URL. `(public)` para rotas públicas, `(authenticated)` para rotas protegidas por JWT. | N/A |
| **Server Component** | Componente React renderizado no servidor. Padrão para layouts, páginas de listagem e busca de dados iniciais. | N/A |
| **Client Component** | Componente com `'use client'` — necessário para interatividade (useState, useEffect, eventos). Mínimo possível por página. | N/A |
| **shadcn/ui** | Biblioteca de componentes React baseada em Radix UI + Tailwind CSS. Componentes instalados localmente (não dependência de pacote). Única biblioteca de componentes aprovada. | latest |
| **TanStack Query** | Biblioteca de gerenciamento de estado assíncrono para fetch, cache e sincronização. `useQuery`, `useMutation`, `useInfiniteQuery`. | 5.x |
| **Zustand** | Gerenciador de estado global leve. Stores: `authStore` (token, usuário), `uiStore` (sidebar collapse, modais). | 5.x |
| **Radix UI** | Biblioteca de primitivos acessíveis (base do shadcn/ui). Fornece comportamentos acessíveis para Dialog, DropdownMenu, Tabs, RadioGroup, etc. | N/A |
| **Tailwind CSS** | Framework de CSS utilitário. Configuração com tokens do shadcn/ui via CSS custom properties. | 3.x |
| **SSE (Server-Sent Events)** | Protocolo de streaming unidirecional servidor → cliente. Usado pelo Guardião do Retorno para streaming de tokens em tempo real. Implementado com `EventSource` no frontend + endpoint NestJS com `@Sse()`. | N/A |
| **Supabase Realtime** | WebSocket bidirecional para subscriptions em tempo real. Canal por `caseId` para atualizações de documentos e status. | N/A |
| **expo-router** | Sistema de roteamento file-based para React Native/Expo. Tabela de rotas: `(auth)/` para autenticação, `(tabs)/` para área logada com bottom tab bar. | 4.x |

### 4.3 Mobile (React Native / Expo)

| **Termo** | **Definição** | **Versão no módulo** |
|---|---|---|
| **Expo SDK** | Conjunto de APIs e ferramentas para React Native. Modo: Managed Workflow. | 52+ |
| **expo-camera** | API de câmera nativa para iOS e Android. Usada para captura de documentos do dossiê (T-042). | latest |
| **expo-image-picker** | API de galeria para seleção de imagens. Alternativa ao expo-camera para documentos existentes. | latest |
| **expo-document-picker** | API de seleção de PDFs e documentos do sistema de arquivos. Para Cedentes PJ com documentos em cloud. | latest |
| **expo-secure-store** | Armazenamento seguro e criptografado para refresh tokens. Equivalente ao Keychain (iOS) e Keystore (Android). | latest |
| **expo-local-authentication** | API de biometria nativa (Face ID, Touch ID, Fingerprint). Usado para re-autenticação no app. | latest |
| **expo-haptics** | API de feedback háptico. Usado em ações críticas: aceite de proposta, confirmação de cancelamento. | latest |
| **Expo Notifications** | API de push notifications. Canais: propostas recebidas, pendências do dossiê, prazo de rascunho. | latest |
| **React Native Reanimated** | Biblioteca de animações executadas na UI thread (sem janking). Usada para transições de tela e feedback de gestos. | 3.x |
| **React Native Gesture Handler** | Biblioteca de gestos nativos. Swipe-back (iOS), long press, drag. Requerido pelo expo-router. | 2.x |
| **AsyncStorage** | Armazenamento chave-valor não criptografado para dados não sensíveis. Usado pelo Zustand para persistência de preferências. | N/A |
| **OTA Update** | Over-the-Air update — atualização do JavaScript bundle via Expo sem passar pela app store. Disponível com Expo SDK. | N/A |

### 4.4 Infraestrutura e Deploy

| **Termo** | **Definição** | **Ambiente** |
|---|---|---|
| **Railway** | Plataforma de deploy do backend NestJS. Deployments automáticos via git push. | Produção + Staging |
| **Vercel** | Plataforma de deploy do frontend Next.js. Edge Network global + OG Image generation + serverless functions. | Produção + Preview |
| **Turborepo** | Monorepo manager com caching de builds. O Repasse Seguro opera como monorepo único: `apps/web-cedente/`, `apps/mobile-cedente/`, `apps/api/`, `packages/`. | CI/CD |
| **pnpm** | Package manager do monorepo. Workspace protocol para dependências compartilhadas entre packages. | Dev + CI/CD |
| **Docker** | Containers para ambientes locais: PostgreSQL, Redis, RabbitMQ, pgvector. `docker-compose.yml` na raiz do monorepo. | Development only |

---

## 5. Segurança e Compliance

| **Termo** | **Definição** | **Regra de negócio** |
|---|---|---|
| **JWT (JSON Web Token)** | Token de autenticação sem estado. Emitido pelo Supabase Auth. Formato: `header.payload.signature` (Base64). Expiração: 24 horas de inatividade (RN-006). Armazenado em httpOnly cookie (web) ou `expo-secure-store` (mobile). | RN-006 (D01.1) |
| **Refresh Token** | Token de longa duração que renova o JWT sem nova autenticação. Rotação automática a cada uso. Web: httpOnly cookie. Mobile: `expo-secure-store`. | RN-006 (D01.1) |
| **RLS (Row Level Security)** | Políticas do PostgreSQL que garantem que cada Cedente só acessa suas próprias linhas. Ativado em todas as tabelas de domínio via Supabase. | RN-011 (D01.1) |
| **LGPD** | Lei Geral de Proteção de Dados Pessoais (Lei 13.709/2018). O módulo Cedente exige: direito de exclusão de dados (T-052), consentimento explícito no cadastro, dados mínimos coletados, anonimização de dados do Cessionário para o Cedente. | RN-010 (D01.1) |
| **Anonimato do Cessionário** | O Cedente nunca vê nome, CPF, e-mail ou qualquer dado pessoal do Cessionário. A plataforma exibe apenas "Comprador qualificado" ou identificador sequencial neutro. | RN-012 (D01.1) |
| **Signed URL** | URL temporária gerada pelo Supabase Storage para acesso seguro a arquivos do dossiê. Validade: 15 minutos por padrão. O URL nunca é armazenado no banco — é gerado sob demanda. | RN-042 (D01.1) |
| **Helmet** | Middleware NestJS que configura headers HTTP de segurança: CSP, HSTS, X-Frame-Options, etc. Adicionado desde o primeiro commit. | N/A |
| **Rate Limiting** | Limitação de requisições por IP/usuário. Proteção contra brute force: 5 tentativas de login antes do bloqueio (RN-005). Throttle geral: 100 req/min por IP. | RN-005 (D01.1) |
| **MIME Type Validation** | Validação obrigatória no backend do tipo real do arquivo (não apenas extensão) antes de aceitar upload para o Supabase Storage. Tipos aceitos: `application/pdf`, `image/jpeg`, `image/png`. | RN-042 (D01.1) |
| **httpOnly Cookie** | Cookie que não é acessível via JavaScript (proteção contra XSS). Usado para armazenar JWT e refresh token no frontend web. | RN-006 (D01.1) |

---

## 6. Termos de Qualidade e Monitoramento

| **Termo** | **Definição** | **Ferramenta** |
|---|---|---|
| **Sentry** | Plataforma de monitoramento de erros e performance. Captura exceções no backend NestJS, erros de renderização no Next.js e crashes no app Expo. DSN configurado via variável de ambiente. | Sentry.io |
| **PostHog** | Plataforma de analytics de produto. Rastreia: funil de cadastro (Etapa 1→5 do wizard), taxa de abandono por etapa, taxa de aceitação de propostas, tempo de resposta a propostas, uso do Guardião. | PostHog Cloud |
| **Langfuse** | Plataforma de observabilidade de LLMs. Rastreia: latência do Guardião, tokens consumidos, taxa de escalação para humano, qualidade de respostas via thumbs up/down. | Langfuse Cloud |
| **p95 Latency** | Percentil 95 de latência — o tempo de resposta abaixo do qual 95% das requisições são atendidas. SLA do módulo Cedente: API < 200ms no p95 para endpoints críticos. | Sentry Performance |
| **Skeleton Loading** | Padrão de loading state obrigatório em todo o produto. Usa `<Skeleton>` do shadcn/ui. Proibido usar spinners. | N/A |
| **Toast** | Notificação temporária (3–5s) no canto da tela para feedback de ações. Implementado com `<Sonner>` do shadcn/ui. Tipos: sucesso (verde), erro (vermelho), informativo (neutro). | shadcn/ui Sonner |
| **Dead-letter Queue (DLQ)** | Fila de mensagens que falharam após todos os retries no RabbitMQ. Gera alerta no Sentry quando mensagens chegam na DLQ. | CloudAMQP + Sentry |
| **Auditlog** | Registro imutável de todos os eventos do caso (linha do tempo T-021). Tabela `case_events` no banco. `deleted_at` não aplicável — eventos nunca são removidos. | PostgreSQL |

---

## 7. Convenções de Código

### 7.1 Nomes de Entidades no Banco de Dados

| **Entidade** | **Tabela no banco** | **Modelo Prisma** | **Nome no código (TS)** |
|---|---|---|---|
| Cedente | `cedentes` | `Cedente` | `cedente`, `cedentes` |
| Caso | `cases` | `Case` | `case_`, `cases` (evitar `case` — keyword JS) |
| Dossiê | `dossiers` | `Dossier` | `dossier`, `dossiers` |
| Documento | `documents` | `Document` | `document`, `documents` |
| Proposta | `proposals` | `Proposal` | `proposal`, `proposals` |
| Envelope de Assinatura | `signature_envelopes` | `SignatureEnvelope` | `signatureEnvelope`, `signatureEnvelopes` |
| Conta Escrow | `escrow_accounts` | `EscrowAccount` | `escrowAccount`, `escrowAccounts` |
| Evento do Caso | `case_events` | `CaseEvent` | `caseEvent`, `caseEvents` |
| Rascunho | Campo `status = 'RASCUNHO'` em `cases` | N/A | `isDraft` (booleano derivado) |
| Notificação | `notifications` | `Notification` | `notification`, `notifications` |
| Conversa do Guardião | `guardian_conversations` | `GuardianConversation` | `guardianConversation` |
| Mensagem do Guardião | `guardian_messages` | `GuardianMessage` | `guardianMessage` |

### 7.2 Enums TypeScript

```typescript
// Cedente
enum TipoPessoa { PF = 'PF', PJ = 'PJ' }
enum CedenteStatus {
  PENDENTE_ATIVACAO = 'PENDENTE_ATIVACAO',
  ATIVO = 'ATIVO',
  BLOQUEADO_TEMPORARIAMENTE = 'BLOQUEADO_TEMPORARIAMENTE',
  SUSPENSO = 'SUSPENSO',
  ENCERRADO = 'ENCERRADO',
}

// Caso
enum Cenario { A = 'A', B = 'B', C = 'C', D = 'D' }
enum CaseStatus {
  RASCUNHO = 'RASCUNHO',
  CAPTADO = 'CAPTADO',
  EM_TRIAGEM = 'EM_TRIAGEM',
  BLOQUEADO = 'BLOQUEADO',
  QUALIFICADO = 'QUALIFICADO',
  OFERTA_ATIVA = 'OFERTA_ATIVA',
  EM_NEGOCIACAO = 'EM_NEGOCIACAO',
  EM_FORMALIZACAO = 'EM_FORMALIZACAO',
  FECHAMENTO = 'FECHAMENTO',
  POS_FECHAMENTO = 'POS_FECHAMENTO',
  EM_REVERSAO = 'EM_REVERSAO',
  EM_MEDIACAO = 'EM_MEDIACAO',
  CONCLUIDO = 'CONCLUIDO',
  CANCELADO = 'CANCELADO',
}

// Documento
enum DocumentStatus {
  PENDENTE = 'PENDENTE',
  EM_ANALISE = 'EM_ANALISE',
  VERIFICADO = 'VERIFICADO',
  REJEITADO = 'REJEITADO',
}

// Proposta
enum ProposalStatus {
  AGUARDANDO_RESPOSTA = 'AGUARDANDO_RESPOSTA',
  CONTRAPROPOSTA_ENVIADA = 'CONTRAPROPOSTA_ENVIADA',
  ACEITA = 'ACEITA',
  RECUSADA = 'RECUSADA',
  EXPIRADA = 'EXPIRADA',
  SUPERADA = 'SUPERADA',
}

// Escrow
enum EscrowStatus {
  NAO_INICIADA = 'NAO_INICIADA',
  AGUARDANDO_DEPOSITO = 'AGUARDANDO_DEPOSITO',
  DEPOSITO_CONFIRMADO = 'DEPOSITO_CONFIRMADO',
  POS_FECHAMENTO = 'POS_FECHAMENTO',
  DISTRIBUIDA = 'DISTRIBUIDA',
  EM_REVERSAO = 'EM_REVERSAO',
  ESTORNADA = 'ESTORNADA',
}

// Assinatura
enum SignatureStatus {
  AGUARDANDO = 'AGUARDANDO',
  ASSINADO = 'ASSINADO',
  EXPIRADO = 'EXPIRADO',
}

enum SignatureDocumentType {
  TERMO_CADASTRO = 'TERMO_CADASTRO',
  INSTRUMENTO_CESSAO = 'INSTRUMENTO_CESSAO',
  TERMO_ESCALONAMENTO = 'TERMO_ESCALONAMENTO',
  TERMO_COMERCIAL = 'TERMO_COMERCIAL',
}
```

### 7.3 Convenções de Nomenclatura

| **Camada** | **Convenção** | **Exemplo** |
|---|---|---|
| Tabelas do banco | `snake_case`, plural | `cases`, `signature_envelopes` |
| Colunas do banco | `snake_case` | `valor_liquido`, `created_at` |
| Modelos Prisma | `PascalCase`, singular | `Case`, `SignatureEnvelope` |
| Variáveis TypeScript | `camelCase` | `valorLiquido`, `cedenteId` |
| Constantes | `UPPER_SNAKE_CASE` | `MAX_UPLOAD_SIZE_BYTES` |
| Componentes React | `PascalCase` | `ProposalCard`, `EscrowPanel` |
| Hooks React | `camelCase`, prefixo `use` | `useCase`, `useProposals` |
| Stores Zustand | `camelCase`, sufixo `Store` | `authStore`, `uiStore` |
| Endpoints REST | `kebab-case`, plural, versão `/v1/` | `/v1/cases`, `/v1/cases/:id/proposals` |
| Telas (rotas Next.js) | `kebab-case` | `app/(authenticated)/casos/[id]/page.tsx` |
| Telas (rotas Expo) | `kebab-case` | `app/(tabs)/casos/[id].tsx` |

### 7.4 Colunas de Auditoria Obrigatórias (Todas as Tabelas)

```prisma
created_at  DateTime  @default(now()) @db.Timestamptz
updated_at  DateTime  @updatedAt @db.Timestamptz
deleted_at  DateTime? @db.Timestamptz  // soft delete — não aplicável em case_events
created_by  String?   // UUID do usuário que criou
updated_by  String?   // UUID do usuário que atualizou por último
```

---

## 8. Siglas e Acrônimos

| **Sigla** | **Significado** | **Contexto** |
|---|---|---|
| **RN** | Regra de Negócio | Identificadores de regras nos docs 01.1 e 01.2 (ex: RN-001) |
| **RF** | Requisito Funcional | Identificadores de requisitos no PRD (ex: RF-001) |
| **ADR** | Architectural Decision Record | Registro de decisão arquitetural documentado no doc de Stacks |
| **DTO** | Data Transfer Object | Classes de validação de entrada/saída de dados NestJS |
| **JWT** | JSON Web Token | Token de autenticação sem estado |
| **RLS** | Row Level Security | Políticas de isolamento de dados no PostgreSQL/Supabase |
| **LGPD** | Lei Geral de Proteção de Dados | Lei 13.709/2018 — compliance obrigatório |
| **KYC** | Know Your Customer | Verificação de identidade — aplicável ao Cessionário (não ao Cedente nesta versão) |
| **SSE** | Server-Sent Events | Streaming unidirecional para o Guardião do Retorno |
| **OTA** | Over-the-Air | Atualização remota do app Expo sem passar pela app store |
| **DLQ** | Dead-letter Queue | Fila de mensagens com falha no RabbitMQ |
| **RAG** | Retrieval Augmented Generation | Técnica de IA que busca contexto relevante antes de responder |
| **PK** | Primary Key | Chave primária — UUID v4 em todas as tabelas |
| **FK** | Foreign Key | Chave estrangeira — referência entre tabelas |
| **JSONB** | JSON Binary | Tipo de coluna PostgreSQL para dados semi-estruturados |
| **SPA** | Single Page Application | Aplicação web sem recarregamento de página — App Router Next.js |
| **CTA** | Call to Action | Botão ou link que solicita uma ação do usuário |
| **p95** | Percentil 95 | Latência abaixo da qual 95% das requisições são atendidas |
| **PF** | Pessoa Física | Cedente com CPF |
| **PJ** | Pessoa Jurídica | Cedente com CNPJ e Representante Legal |
| **RS** | Repasse Seguro | Nome abreviado da plataforma (somente em contextos técnicos internos) |
| **LLM** | Large Language Model | Modelo de linguagem de grande escala (GPT-4 no módulo Cedente) |
