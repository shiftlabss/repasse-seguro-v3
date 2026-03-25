# 10 - Glossário Técnico

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia, Produto e QA |
| **Escopo** | Dicionário de termos de negócio e termos técnicos do CRM — definição precisa, origem e distinções críticas |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 (America/Fortaleza) |
| **Dependências** | 01.1–01.5 Regras de Negócio · 05.1–05.5 PRD · 02 Stacks |

---

> **TL;DR**
>
> - **52 termos** documentados: 24 de negócio e 28 técnicos.
> - **Convenção de categorias:** `[NEGÓCIO]` — entidades, processos e conceitos do domínio imobiliário de cessão; `[TÉCNICO]` — infraestrutura, padrões de código, protocolos e ferramentas.
> - Termos com origem em Regras de Negócio têm referência à RN de origem.
> - Termos técnicos em inglês são mantidos em inglês — definições em português.

---

## 1. Termos de Negócio

### 1.1 Entidades e Papéis

---

**Analista RS** `[NEGÓCIO]`
Profissional interno da Repasse Seguro responsável por conduzir um ou mais Casos do cadastro ao fechamento. Papel operacional principal do CRM — acessa apenas os Casos atribuídos a si. Origem: RN-001, seção 3.

---

**Admin RS** `[NEGÓCIO]`
Administrador do sistema com acesso irrestrito a todos os Casos, Contatos, configurações e logs de auditoria. Pode criar e desativar usuários e alterar parâmetros críticos (com aprovação dupla). Origem: RN-001, seção 3.

---

**Caso** `[NEGÓCIO]`
Unidade operacional central do CRM. Representa um contrato imobiliário em processo de cessão, do cadastro do Cedente até a formalização concluída. Cada Caso tem exatamente 1 Cedente titular e pode ter 1 Cessionário ativo por vez. Identificado pelo formato `RS-YYYY-NNNN`. Origem: RN-006.

> **Distinção crítica:** um Caso não é um simples registro de CRM de vendas. É um processo operacional com 9 estados, condições de saída obrigatórias e SLA monitorado.

---

**Cedente** `[NEGÓCIO]`
Pessoa física ou jurídica que detém o contrato imobiliário e deseja repassá-lo. Cadastrado como Contato com papel "Cedente" no CRM. Escolhe um Cenário (A/B/C/D) na fase de simulação. Dado do Cenário é confidencial — nunca exposto ao Cessionário. Origem: RN-006, RN-012.

---

**Cessionário** `[NEGÓCIO]`
Pessoa física ou jurídica que adquire o contrato repassado. Cadastrado como Contato com papel "Cessionário". Vinculado ao Caso via evento de Match. Origem: RN-014.

---

**Coordenador RS** `[NEGÓCIO]`
Profissional interno que supervisiona a equipe de Analistas RS. Pode redistribuir Casos, aprovar Dossiês, acessar relatórios consolidados e reconhecer alertas de SLA. Não acessa configurações do sistema. Origem: RN-001, seção 3.

---

**Parceiro Externo** `[NEGÓCIO]`
Corretor ou advogado que indicou o Cedente ou o Cessionário. Acesso restrito ao portal externo — apenas o status resumido dos Casos em que tem participação como indicador. Nunca vê dados pessoais de Cedentes ou Cessionários. Origem: RN-004.

---

**Contato** `[NEGÓCIO]`
Registro de uma pessoa física ou jurídica no CRM. Um Contato pode ter papel de Cedente, Cessionário, Corretor, Advogado ou Incorporadora. Um Contato pode ser vinculado a múltiplos Casos ao longo do tempo. Origem: RN-014.

> **Distinção crítica — Contato vs. Usuário:** Contato é uma entidade de negócio (Cedente, Cessionário, Parceiro). Usuário é quem opera o CRM (Analista RS, Coordenador RS, Admin RS). São entidades separadas no banco de dados.

---

### 1.2 Processos e Estados

---

**Ciclo de Vida do Caso** `[NEGÓCIO]`
Os 9 estados pelos quais um Caso passa: Cadastro → Simulação → Verificação → Publicação → Match → Negociação → Anuência → Formalização → Concluído. Cada estado tem SLA esperado e condição de saída obrigatória. Casos podem ser cancelados em qualquer estado anterior a "Concluído". Origem: RN-006 a RN-009, seção 5.

---

**Condição de Saída** `[NEGÓCIO]`
Critério que deve ser atendido para um Caso avançar de um estado para o próximo. O sistema valida a condição antes de permitir o avanço (RN-007). Exemplo: condição de saída do estado "Verificação" = Dossiê completo e aprovado pelo Coordenador RS.

---

**SLA do Caso** `[NEGÓCIO]`
Prazo máximo esperado para cada estado do Caso (ex: "Verificação" = 5 dias úteis). Violação de SLA gera alertas automáticos para Analista RS e Coordenador RS. Ciclo total acima de 60 dias corridos gera alerta para Coordenador RS. Origem: RN-009.

---

**Match** `[NEGÓCIO]`
Evento em que o sistema identifica compatibilidade entre um Caso publicado (Cedente) e um Cessionário interessado. Dispara a transição do estado "Publicação" para "Match". Origem: seção 2, glossário.

---

**Fechamento** `[NEGÓCIO]`
Evento que dispara o registro e cobrança de comissão. Requer 3 critérios cumulativos: (1) instrumento de cessão assinado por todas as partes, (2) anuência da incorporadora confirmada e (3) comprovante de transação financeira na conta escrow. Origem: seção 2, glossário; RN-074 (01.2).

---

**Anuência** `[NEGÓCIO]`
Autorização formal da incorporadora para a transferência do contrato. Condição obrigatória para o Fechamento. Estado específico no ciclo de vida do Caso. Origem: seção 2, glossário.

---

**Dossiê** `[NEGÓCIO]`
Conjunto de documentos verificados de um Caso: Tabela Atual, Tabela Contrato, comprovante de saldo devedor, documentos pessoais do Cedente e instrumento de cessão assinado. Deve ser aprovado pelo Coordenador RS antes de a Publicação ser habilitada. Origem: seção 2, glossário; RN-058 a RN-063 (01.2).

---

**Cenário (A/B/C/D)** `[NEGÓCIO]`
Opção de retorno escolhida pelo Cedente na fase de Simulação. Determina a meta de valor recuperado e a base de cálculo da comissão do Cedente. Dado confidencial — nunca exposto ao Cessionário, ao Parceiro Externo ou a qualquer tela que não seja a do Analista RS responsável pelo Caso. Origem: seção 2, glossário.

---

**Delta (Δ)** `[NEGÓCIO]`
Diferença econômica entre a Tabela Atual e a Tabela do Contrato. Base de cálculo da comissão do Cessionário. Se Δ ≤ 0, a comissão do Cessionário é calculada sobre o Valor Pago pelo Cedente. Origem: seção 2, glossário.

---

**Conta Escrow** `[NEGÓCIO]`
Mecanismo financeiro que retém o valor da transação até o cumprimento das condições de Fechamento. A liberação é registrada no Caso e dispara o cálculo e cobrança das comissões. Origem: seção 2, glossário.

---

**Termo Comercial** `[NEGÓCIO]`
Documento que formaliza as condições específicas de comissão de um Caso: piso, teto e percentual negociado. Define os limites dentro dos quais propostas e contrapropostas são válidas. Origem: seção 2, glossário.

---

**Perda Evitada** `[NEGÓCIO]`
Diferença entre o que o Cedente receberia no distrato e o que recebe via cessão. Métrica de valor gerado pelo Caso — usada em relatórios de inteligência de mercado. Origem: seção 2, glossário.

---

**Follow-up** `[NEGÓCIO]`
Atividade futura agendada com prazo e responsável definidos. Pode ser vinculado a um Caso ou a um Contato. Status: Agendado → Concluído / Vencido. Origem: RN-015, seção 8.2.

---

**Atividade** `[NEGÓCIO]`
Registro de interação vinculada a um Caso ou Contato: ligação, reunião, e-mail, WhatsApp, nota interna. Pode ser uma atividade passada (retroativa até 30 dias para Analista RS) ou um Follow-up futuro. Origem: RN-098 a RN-100 (01.3).

---

**Opt-out** `[NEGÓCIO]`
Status de um Contato que optou por não receber comunicações via WhatsApp ou e-mail. Quando ativo, todos os botões de comunicação são desabilitados e uma mensagem LGPD é exibida. Origem: RN-106 (01.3).

---

**Investidor Recorrente** `[NEGÓCIO]`
Flag aplicada a um Cessionário que já concluiu mais de 1 cessão pela plataforma. Exibida como badge na listagem de Contatos e no perfil. Origem: RN-096 (01.3).

---

## 2. Termos Técnicos

### 2.1 Autenticação e Segurança

---

**JWT (JSON Web Token)** `[TÉCNICO]`
Padrão aberto (RFC 7519) para transmissão segura de informações entre partes como objeto JSON assinado. Usado pelo Supabase Auth para representar a sessão do usuário. O payload contém `user_id`, `role` e `exp` (expiração). Nunca armazenado em `localStorage` — apenas em cookie `HttpOnly`.

---

**RLS (Row-Level Security)** `[TÉCNICO]`
Recurso do PostgreSQL que restringe quais linhas de uma tabela cada usuário pode ver ou modificar com base em políticas definidas no banco. No CRM, o RLS garante que Analistas RS acessem apenas os Casos atribuídos a eles, e Parceiros Externos não acessem dados pessoais. Complementa (não substitui) a validação na camada de aplicação.

---

**Supabase Auth** `[TÉCNICO]`
Serviço de autenticação gerenciado pela Supabase (baseado em GoTrue). Gerencia sessões, refresh tokens, recuperação de senha e convites. Integrado nativamente ao RLS do PostgreSQL via `auth.uid()`.

---

**Refresh Token** `[TÉCNICO]`
Token de longa duração (tipicamente 7–30 dias) usado para obter novos access tokens sem exigir novo login. Armazenado em cookie `HttpOnly`. O Supabase gerencia a rotação automática.

---

**Rate Limiting** `[TÉCNICO]`
Controle de frequência de requisições a um endpoint. No contexto de autenticação (RN-002), limita tentativas de login a 5 falhas em 15 minutos antes do bloqueio. Implementado no NestJS via `@nestjs/throttler` e no Supabase Auth.

---

**Enumeração de usuários** `[TÉCNICO]`
Ataque que explora diferenças nas respostas de um endpoint para descobrir se um e-mail está ou não cadastrado. Mitigado no fluxo de recuperação de senha (RN-005) retornando sempre a mesma mensagem independentemente de o e-mail existir.

---

### 2.2 Banco de Dados e ORM

---

**Prisma** `[TÉCNICO]`
ORM (Object-Relational Mapper) de TypeScript/JavaScript para Node.js. Versão usada: Prisma 6+. Responsável por migrations, type-safety das queries e geração do client. Documentação normativa: Doc 02 (Stacks).

---

**Migration** `[TÉCNICO]`
Arquivo SQL gerado pelo Prisma que descreve uma alteração incremental no schema do banco de dados. Versionado no repositório em `prisma/migrations/`. Aplicado em produção via `prisma migrate deploy`.

---

**Soft Delete** `[TÉCNICO]`
Padrão de deleção lógica: o registro não é removido fisicamente da tabela — apenas o campo `deleted_at` é preenchido com a data/hora da deleção. Registros com `deleted_at IS NOT NULL` são filtrados automaticamente via middleware Prisma. Necessário para LGPD (retenção e auditoria).

---

**Optimistic Locking** `[TÉCNICO]`
Estratégia de controle de concorrência que usa um campo `version Int` nas entidades editáveis. Ao atualizar, o sistema verifica se `version` no banco ainda é igual ao `version` enviado pelo cliente — se divergir, rejeita a operação. Previne sobrescrita silenciosa de dados em edições concorrentes (RN-130 em 01.2).

---

**Decimal(15,2)** `[TÉCNICO]`
Tipo de dado PostgreSQL para valores monetários. Armazena até 15 dígitos totais com 2 casas decimais. Usado em todos os campos de valor financeiro no schema CRM. `Float` é proibido em campos monetários por acumular erros de ponto flutuante.

---

**UUID v4** `[TÉCNICO]`
Identificador universalmente único gerado de forma aleatória (128 bits). Usado como chave primária em todas as tabelas do schema. Vantagens: não enumerável (segurança), geração descentralizada (compatível com Supabase Auth). Desvantagem: índice B-tree levemente menos eficiente que `BIGSERIAL` — aceito para segurança.

---

**Schema (PostgreSQL)** `[TÉCNICO]`
Namespace lógico dentro de um banco de dados PostgreSQL que agrupa tabelas, views e funções. O CRM usa `public` (domínio principal) e `audit` (trilha imutável). Não confundir com "schema Prisma" (arquivo de definição do ORM).

---

**Seed** `[TÉCNICO]`
Script (`prisma/seed.ts`) que popula o banco de dados com dados iniciais para desenvolvimento e testes. Executado via `npx prisma db seed`. Nunca executado em produção.

---

**@db.Timestamptz** `[TÉCNICO]`
Decorator Prisma que mapeia o campo para o tipo `TIMESTAMPTZ` do PostgreSQL (timestamp com fuso horário). Obrigatório em todos os campos de data/hora do schema. Garante que o banco armazene e retorne datas em UTC, e que a conversão para `America/Fortaleza` seja feita na camada de apresentação.

---

### 2.3 API e Comunicação

---

**NestJS** `[TÉCNICO]`
Framework Node.js para construção de APIs server-side. Versão usada: NestJS 10. Responsável pela API REST do CRM (autenticação, casos, contatos, atividades, relatórios). Documentação normativa: Doc 02 (Stacks).

---

**Endpoint** `[TÉCNICO]`
URL específica de uma API que responde a uma requisição HTTP. Formato no CRM: `GET /cases`, `POST /cases/:id/advance`, `PATCH /contacts/:id`. Documentados individualmente no Doc 14 (Especificações Técnicas).

---

**Webhook** `[TÉCNICO]`
Mecanismo de comunicação em que um sistema externo notifica o CRM via requisição HTTP POST quando um evento ocorre. Usado pela integração ZapSign (assinaturas digitais) para notificar o CRM quando um envelope é assinado ou recusado.

---

**RabbitMQ** `[TÉCNICO]`
Message broker AMQP usado para processamento assíncrono de eventos no CRM: disparo de alertas de SLA, notificações de WhatsApp, geração de relatórios pesados. Provedor no ambiente de produção: CloudAMQP. Documentação normativa: Doc 02 (Stacks).

---

**Dead Letter Queue (DLQ)** `[TÉCNICO]`
Fila especial do RabbitMQ que recebe mensagens que falharam após N tentativas de processamento. Evita perda de eventos críticos (ex: alerta de SLA não processado). Monitorada pelo time de engenharia via painel CloudAMQP.

---

**Redis / Upstash** `[TÉCNICO]`
Banco de dados em memória usado para cache de sessões, rate limiting e dados de alta frequência de acesso (ex: parâmetros globais do sistema, contagem de alertas de SLA por Analista RS). Provedor: Upstash (serverless Redis). Documentação normativa: Doc 02 (Stacks).

---

**Server-Sent Events (SSE)** `[TÉCNICO]`
Protocolo HTTP unidirecional do servidor para o cliente. Usado no CRM para push de notificações em tempo real (novos alertas de SLA, follow-ups vencidos) sem polling. Alternativa ao WebSocket para comunicação unidirecional.

---

### 2.4 Frontend e Renderização

---

**Next.js App Router** `[TÉCNICO]`
Sistema de roteamento do Next.js 15 baseado no diretório `app/`. Suporta Server Components (RSC), Client Components (`"use client"`), e layouts aninhados. O CRM usa App Router com autenticação protegida via middleware. Documentação normativa: Doc 02 (Stacks).

---

**Server Component (RSC)** `[TÉCNICO]`
Componente React que é renderizado exclusivamente no servidor. Não tem acesso a estado do browser (`useState`, `useEffect`). Ideal para busca de dados inicial (dashboard, listagens). Reduz o JavaScript enviado ao cliente.

---

**Client Component** `[TÉCNICO]`
Componente React marcado com `"use client"` que é hidratado e executado no browser. Necessário para interatividade (formulários, modais, drag-and-drop). No CRM, usado em componentes com estado local ou efeitos colaterais.

---

**shadcn/ui** `[TÉCNICO]`
Coleção de componentes de UI acessíveis e customizáveis baseados em Radix UI e Tailwind CSS. Diferente de uma biblioteca, os componentes são copiados para o projeto e modificados conforme necessário. Versão de referência: documentada no Doc 02 (Stacks).

---

**TanStack Query (React Query)** `[TÉCNICO]`
Biblioteca de gerenciamento de estado de servidor para React. Usada no CRM para busca, cache e sincronização de dados da API (casos, contatos, atividades). Chaves de cache padronizadas por entidade e ID.

---

**Zod** `[TÉCNICO]`
Biblioteca TypeScript-first de validação e parsing de schemas. Usada em conjunto com React Hook Form para validação de formulários no frontend e com NestJS (class-validator) para validação de DTOs na API.

---

### 2.5 Integrações Externas

---

**ZapSign** `[TÉCNICO]`
Plataforma de assinatura digital usada para o instrumento de cessão e demais documentos do Dossiê. O CRM envia envelopes via API REST da ZapSign e recebe o status de assinatura via webhook. Condição de Fechamento: instrumento assinado por todas as partes.

---

**WhatsApp Business API (Meta)** `[TÉCNICO]`
API oficial da Meta para envio de mensagens WhatsApp por empresas. Exige templates aprovados para iniciar conversas (fora da janela de 24h). O CRM integra via provedor (a definir em Doc 02) para envio de mensagens e recebimento via webhook.

---

**Janela de 24h** `[TÉCNICO]`
Regra da WhatsApp Business API que permite envio de mensagens livres (fora de template) apenas dentro de 24 horas após a última mensagem recebida do contato. Após 24h sem interação, apenas templates aprovados pela Meta podem ser enviados. Origem: RN-103 (01.3).

---

**Supabase Storage** `[TÉCNICO]`
Serviço de armazenamento de objetos do Supabase (baseado em S3). Usado para armazenar documentos do Dossiê (PDFs, imagens). Acesso controlado via RLS + URLs assinadas com expiração.

---

**Turborepo** `[TÉCNICO]`
Sistema de build para monorepos com cache de tarefas e pipeline de builds paralelos. O projeto Repasse Seguro é um monorepo Turborepo com workspaces: `apps/web` (Next.js), `apps/api` (NestJS), `packages/shared` (tipos e utilitários compartilhados). Documentação normativa: Doc 02 (Stacks).

---

## 3. Índice por Categoria

### Termos de Negócio (24)
Analista RS · Admin RS · Anuência · Atividade · Caso · Cedente · Cenário (A/B/C/D) · Cessionário · Ciclo de Vida do Caso · Condição de Saída · Conta Escrow · Contato · Coordenador RS · Delta (Δ) · Dossiê · Fechamento · Follow-up · Investidor Recorrente · Match · Opt-out · Parceiro Externo · Perda Evitada · SLA do Caso · Termo Comercial

### Termos Técnicos (28)
@db.Timestamptz · Dead Letter Queue (DLQ) · Decimal(15,2) · Endpoint · Enumeração de usuários · Janela de 24h · JWT · Migration · NestJS · Next.js App Router · Optimistic Locking · Prisma · RabbitMQ · Rate Limiting · Redis / Upstash · Refresh Token · RLS · Schema (PostgreSQL) · Seed · Server Component (RSC) · Client Component · Server-Sent Events (SSE) · shadcn/ui · Soft Delete · Supabase Auth · Supabase Storage · TanStack Query · Turborepo · UUID v4 · Webhook · WhatsApp Business API · Zod · ZapSign

---

## 4. Controle de Versão

| **Versão** | **Data** | **Responsável** | **Alteração** |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 52 termos (24 negócio + 28 técnico) |
