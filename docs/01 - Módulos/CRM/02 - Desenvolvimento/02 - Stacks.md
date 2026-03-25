# Stacks Tecnológicas — Módulo CRM

## Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Tech Lead / CTO |
| **Escopo** | Stack normativa do módulo CRM — Repasse Seguro |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 2026-03-23 (America/Fortaleza) |
| **Fonte primária** | ShiftLabs Stacks v7.0 (Fernando Calado) |
| **Fonte secundária** | Regras de Negócio do CRM v1.0 (01.1 a 01.5) |
| **Caráter** | **Normativo.** Desvios exigem ADR aprovado. |

---

> **TL;DR**
>
> - **Backend:** NestJS 10+ em `apps/api/modules/crm/` · Supabase PostgreSQL 17+ com Prisma 6+ · Redis 7.4+ para cache · RabbitMQ 4+ para filas assíncronas.
> - **Frontend Web:** Next.js 15+ com App Router em `apps/web-crm/` · shadcn/ui + Tailwind CSS 4 · Responsive web apenas (breakpoint mínimo 768px) · Sem app mobile nativo na v1.0.
> - **IA integrada:** Dani-Admin (supervisor do CRM) · GPT-4o + LangChain.js (Fase 2).
> - **Integrações:** WhatsApp Business API (Meta Cloud API) · ZapSign (assinatura eletrônica) · Celcoin (Conta Escrow via webhook HMAC-SHA256) · Supabase Auth (JWT com claims de papel).
> - **Observabilidade:** PostHog (analytics de produto) · Sentry (error tracking) · Pino (logs estruturados).
> - **Hospedagem:** Railway (backend) · Vercel (frontend) · Upstash Redis · CloudAMQP.
> - **Proibições críticas:** Next.js 14 ou inferior · Pages Router · class components React · ORM alternativo ao Prisma · PostgreSQL auto-gerenciado em produção · spinners globais (usar skeleton obrigatório).

---

## 1. Princípios de Governança Tecnológica

1. **Baseline primeiro.** O ShiftLabs Stacks v7.0 é a fonte primária de verdade. Toda tecnologia deste documento herda do baseline, salvo desvio documentado via ADR.
2. **CRM é web-first, não mobile-first.** O CRM serve equipe interna em desktop e tablet. Não há app nativo na v1.0. Breakpoint mínimo: 768px.
3. **Justificativa obrigatória.** Nenhuma tecnologia entra ou sai sem justificativa técnica vinculada a uma RN ou requisito de infraestrutura.
4. **Desenvolvimento 100% por IA.** Todo código é gerado por Claude Code Desktop. Decisões de stack priorizam previsibilidade do output de IA.
5. **Rastreabilidade de desvio.** Todo desvio do baseline gera um ADR numerado com contexto, decisão, alternativas e consequências.
6. **Proibição explícita.** Tecnologias não listadas como aprovadas ou condicionais são proibidas por padrão.

---

## 2. Backend

### 2.1 Localização no Monorepo

```
apps/
  api/
    modules/
      crm/           ← módulo NestJS do CRM
        cases/
        contacts/
        activities/
        negotiations/
        commissions/
        dossier/
        team/
        dashboard/
        reports/
        integrations/
```

### 2.2 Stack Obrigatória

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **Node.js** | 22.x+ (LTS) | ✅ Aprovado | Runtime padrão ShiftLabs. | Usar versão LTS. Atualizar conforme release cycle. |
| **NestJS** | 10.x+ | ✅ Aprovado | Arquitetura modular, guards/pipes nativos, Swagger embutido. Output de IA consistente. | Framework HTTP único. Todo endpoint do CRM usa NestJS. |
| **TypeScript** | 5.4+ | ✅ Aprovado | Tipagem estática obrigatória. `strict: true` em todo tsconfig. | Nenhum arquivo `.js` ou `.jsx` permitido. |
| **Prisma** | 6.x+ | ✅ Aprovado | ORM único aprovado. Tipagem auto-gerada, migrations declarativas. | Único ORM permitido. Queries raw via `$queryRaw` com parâmetros preparados. |
| **PostgreSQL (Supabase)** | 17+ | ✅ Aprovado | Banco relacional único, gerenciado via Supabase. | PostgreSQL auto-gerenciado proibido em produção. |
| **Supabase** | latest | ✅ Aprovado | PostgreSQL + Auth + Storage + Realtime. Elimina infra operacional. | Plataforma padrão para banco, auth, storage e realtime. |
| **Supabase Auth** | latest | ✅ Aprovado | Autenticação de usuários internos (Admin RS, Coordenador RS, Analista RS, Parceiro Externo). JWT com claims de papel validados a cada requisição (RN-193). | Único sistema de autenticação. Papéis definidos como roles no Supabase. |
| **Supabase Storage** | latest | ✅ Aprovado | Armazenamento de documentos do Dossiê (RN-058 a RN-063). PDFs, imagens, comprovantes. | Upload via signed URL. Nomenclatura UUID v4. Bucket separado por tipo de documento. |
| **Supabase Realtime** | latest | ✅ Aprovado | Sincronização de estado de Casos em tempo real (RN-183, RN-186). Alertas de SLA ao vivo (RN-107). | Subscriptions em tabelas `cases`, `dossier_documents`, `sla_alerts`. |
| **Redis** | 7.4+ | ✅ Aprovado | Cache de KPIs do Dashboard Executivo (RN-144, ≤5 min defasagem). Blacklist de tokens revogados. Cache de parâmetros de configuração (RN-147). | TTL obrigatório em toda chave. Docker local, **Upstash** em produção. |
| **RabbitMQ** | 4.x+ | ✅ Aprovado | Filas para notificações assíncronas (RN-195), alertas de SLA (RN-107), relatório semanal (RN-110), retentativas de integração (RN-196). | Retry obrigatório + dead-letter queue em toda fila. Docker local, **CloudAMQP** em produção. |
| **Pino** | 9.x+ | ✅ Aprovado | Logs estruturados obrigatórios. Request ID, timestamp, nível e contexto. | Todo serviço NestJS configura Pino. Deploy sem logs estruturados proibido. |
| **class-validator + class-transformer** | latest | ✅ Aprovado | Validação de DTOs integrada aos pipes do NestJS. | Toda entrada de dados validada via pipes. |
| **@nestjs/swagger** | latest | ✅ Aprovado | Swagger/OpenAPI auto-gerado. | Obrigatório em todo endpoint. |
| **Helmet** | 8.x+ | ✅ Aprovado | Headers de segurança automáticos. | Middleware adicionado desde o primeiro commit. |
| **@nestjs/throttler** | latest | ✅ Aprovado | Rate limiting. Endpoints de auth com limites restritivos (RN-002: 5 tentativas, bloqueio 30 min). | Todo endpoint público com rate limiting. |
| **@sentry/nestjs** | 9.x+ | ✅ Aprovado | Error tracking com stack traces, breadcrumbs e alertas. | Obrigatório em produção. Source maps enviados no build. |

### 2.3 Integrações Externas

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **Meta Cloud API (WhatsApp Business)** | latest | ✅ Aprovado | Canal de comunicação principal do CRM com Cedentes e Cessionários (RN-103, RN-187). Templates pré-aprovados pela Meta. | Webhooks validados via assinatura HMAC. Janela de 24h monitorada (RN-187). Fallback via e-mail. |
| **ZapSign** | API REST latest | ✅ Aprovado | Assinatura eletrônica de contratos de cessão (RN-189). Webhook de status de assinatura em tempo real. | Envelope com signatários: Cedente, Cessionário, representante RS. Lembretes D+2, D+4 (RN-189). |
| **Celcoin (Conta Escrow)** | API REST v2+ | ✅ Aprovado | Confirmação de depósito na Conta Escrow como critério de Fechamento (RN-191). | Webhooks validados via HMAC-SHA256. Fallback manual com aprovação de Coordenador RS (RN-192). |
| **LangChain.js** | 0.3+ | ✅ Aprovado (Fase 2) | Orquestração do agente Dani-Admin — supervisor de IA do CRM. | GPT-4o como LLM base. Fase 2 — não implementar na v1.0. |
| **OpenAI SDK** | 4.x+ | ✅ Aprovado (Fase 2) | GPT-4o para Dani-Admin. | Fase 2 — não implementar na v1.0. |

### 2.4 Convenções de Banco de Dados (Prisma Schema)

Herdadas integralmente do ShiftLabs Stacks v7.0, seção 1.5. Destaques críticos para o CRM:

- **Primary keys:** UUID v4 em todas as tabelas. Auto-increment proibido.
- **Colunas de auditoria:** `created_at`, `updated_at` obrigatórios em toda tabela. `created_by`, `updated_by` obrigatórios em tabelas de domínio (casos, negociações, comissões, dossiê, configurações).
- **Soft delete:** Padrão para tabelas de domínio (casos, contatos, atividades, documentos, usuários). Coluna `deleted_at DateTime? @db.Timestamptz`.
- **Log de auditoria:** Schema separado append-only — imutável (RN-194). Nenhum campo pode ser editado ou excluído. Retenção de 10 anos.
- **Timestamps:** `@db.Timestamptz` obrigatório. `@db.Timestamp` sem timezone proibido. Armazenamento em UTC, exibição em America/Fortaleza (RN-198).
- **Nomenclatura:** Tabelas em `snake_case` plural. Colunas em `snake_case`. Enums em `PascalCase` com valores `UPPER_SNAKE_CASE`.
- **Enum de estados do Caso:** `CADASTRO | SIMULACAO | VERIFICACAO | PUBLICACAO | MATCH | NEGOCIACAO | ANUENCIA | FORMALIZACAO | CONCLUIDO | CANCELADO`.
- **Enum de papéis:** `ADMIN_RS | COORDENADOR_RS | ANALISTA_RS | PARCEIRO_EXTERNO`.

### 2.5 Tabelas Principais (referência)

| Tabela | Descrição | RNs relacionadas |
|---|---|---|
| `cases` | Casos de cessão imobiliária | RN-006 a RN-009, RN-039 a RN-044 |
| `contacts` | Cedentes, Cessionários, Parceiros, Incorporadoras | RN-093 a RN-097 |
| `activities` | Ligações, reuniões, WhatsApp, notas, follow-ups | RN-098 a RN-102 |
| `negotiations` | Propostas, contrapropostas, aceites | RN-045 a RN-049 |
| `commissions` | Comissões estimadas e confirmadas | RN-050 a RN-057 |
| `dossier_documents` | Documentos do Dossiê com versão e status | RN-058 a RN-063 |
| `users` | Usuários internos com papel e estado | RN-139 a RN-142 |
| `sla_alerts` | Alertas gerados pelo monitor de SLA | RN-107, RN-108 |
| `audit_log` | Log imutável de todas as ações (schema separado) | RN-194 |
| `system_parameters` | Parâmetros configuráveis do CRM | RN-147, RN-148 |
| `message_templates` | Templates de mensagem aprovados | RN-104, RN-149 |

---

## 3. Frontend Web

### 3.1 Critério de Decisão

O CRM é um **web app 100% autenticado** para equipe interna. Requer SSR leve para carregamento inicial rápido e App Router para layouts aninhados (ex: detalhe do Caso com abas). **Next.js 15+ com App Router** é obrigatório.

Diferente do módulo Admin (que usa React + Vite por ser SPA pura), o CRM usa Next.js para tirar proveito de Server Components no carregamento de dados iniciais dos Casos.

### 3.2 Localização no Monorepo

```
apps/
  web-crm/              ← Next.js 15 App Router
    app/
      (auth)/           ← login, recuperação de senha
      (dashboard)/      ← layout autenticado
        page.tsx        ← painel pessoal do Analista RS
        pipeline/       ← kanban + lista de Casos
        cases/[id]/     ← detalhe do Caso
        contacts/       ← gestão de Contatos
        activities/     ← agenda e atividades
        reports/        ← relatórios
        team/           ← gestão de equipe (Admin only)
        settings/       ← configurações (Admin only)
    components/
      ui/               ← shadcn/ui customizado
      cases/
      pipeline/
      dossier/
      negotiations/
      commissions/
      communications/
    lib/
      motion-tokens.ts
      api-client.ts
```

### 3.3 Stack Obrigatória

| Tecnologia | Versão mínima | Status | Justificativa | Regra de uso |
|---|---|---|---|---|
| **Next.js** | 15.x+ | ✅ Aprovado | App Router com Server Components para carregamento inicial de Casos. Layouts aninhados para detalhe do Caso com abas. | App Router obrigatório. Pages Router proibido. `use client` apenas em componentes com estado ou event handlers. |
| **React** | 19.x+ | ✅ Aprovado | Biblioteca de UI padrão ShiftLabs. | Componentes funcionais. Class components proibidos. |
| **TypeScript** | 5.4+ | ✅ Aprovado | `strict: true` obrigatório. | Nenhum arquivo `.js` ou `.jsx` permitido. |
| **Tailwind CSS** | 4.x+ | ✅ Aprovado | Estilização utility-first. Design tokens do CRM via `@theme`. | Única abordagem de estilização. CSS Modules e styled-components proibidos. |
| **shadcn/ui** | latest | ✅ Aprovado | Componentes acessíveis baseados em Radix UI. Customizados com tokens do CRM (Brand Guide Doc 03). | Instalar apenas componentes necessários. Não customizar internos do Radix. |
| **Lucide React** | latest | ✅ Aprovado | Biblioteca de ícones padrão do CRM. Consistente com shadcn/ui. | Única biblioteca de ícones aprovada. Ícones SVG inline proibidos. |
| **Framer Motion** | 12.x+ | ✅ Aprovado | Animações e transições conforme Motion Spec (Doc 04). | Usar tokens de duração definidos. Spinners proibidos — usar skeleton. |
| **TanStack Query** | 5.x+ | ✅ Aprovado | Cache e sincronização de dados server-state. | Toda chamada de API via TanStack Query. Fetch manual proibido em componentes. |
| **Zustand** | 5.x+ | ✅ Aprovado | Estado global de UI (ex: filtros do Pipeline, preferências do usuário). | Apenas para estado de UI. Dados de servidor via TanStack Query. |
| **TanStack Table** | 8.x+ | ✅ Aprovado | Tabelas complexas com ordenação, filtro e paginação (lista de Casos, relatórios). | Única solução de tabela aprovada. |
| **React Hook Form** | 7.x+ | ✅ Aprovado | Formulários controlados com validação (criação de Caso, registro de Atividade, etc.). | Integrar com Zod para validação de schema. |
| **Zod** | 3.x+ | ✅ Aprovado | Validação de schemas no frontend. Compartilhar schemas com backend via pacote `packages/shared-types`. | Único validador de schema aprovado. |
| **PostHog** | 1.x+ | ✅ Aprovado | Analytics de produto: funil de uso, features mais utilizadas, identificação de gargalos (RN-199). | Dados anonimizados. Não rastrear dados pessoais de Cedentes/Cessionários. |
| **@sentry/nextjs** | 9.x+ | ✅ Aprovado | Error tracking no frontend com breadcrumbs de sessão. | Obrigatório em produção. Source maps enviados no build. |

### 3.4 Responsividade

| Breakpoint | Resolução mínima | Uso |
|---|---|---|
| `tablet` | 768px | Mínimo suportado. Funcionalidades críticas de campo (RN-199). |
| `desktop-sm` | 1280px | Resolução mínima de desktop declarada (RN-199). |
| `desktop-lg` | 1440px+ | Breakpoint ideal para o Pipeline kanban. |

**Mobile nativo (iOS/Android): não suportado na v1.0** (RN-199 e seção 25 das RNs).

---

## 4. Observabilidade

| Ferramenta | Camada | Uso no CRM |
|---|---|---|
| **PostHog** | Frontend | Rastreamento de eventos de produto: abertura de Caso, avanço de estado, uso de filtros, exportações. |
| **Sentry** | Frontend + Backend | Error tracking com alertas automáticos. SLA de resposta a erros P0: < 15 minutos. |
| **Pino** | Backend | Logs estruturados JSON em produção. Request ID propagado do frontend ao backend. |
| **Supabase Dashboard** | Banco de dados | Métricas de banco: slow queries, conexões ativas, uso de storage do Dossiê. |
| **Railway Metrics** | Infraestrutura | CPU, memória, latência de rede do backend. |

---

## 5. Infraestrutura de Produção

| Componente | Serviço | Justificativa |
|---|---|---|
| **Backend (NestJS)** | Railway | Hospedagem simples, deploy via Git, scaling automático. |
| **Frontend (Next.js)** | Vercel | Integração nativa com Next.js, Edge Network global, deploy automático. |
| **Banco de dados** | Supabase | PostgreSQL gerenciado com backups automáticos, RPO 24h, RTO 4h (RN-200). |
| **Redis** | Upstash | Serverless Redis, pay-per-use, zero ops. |
| **Filas (RabbitMQ)** | CloudAMQP | Gerenciado, free tier para dev, escalável. |
| **Storage de documentos** | Supabase Storage | Documentos do Dossiê com signed URLs e controle de acesso por papel. |

---

## 6. Segurança

| Controle | Implementação | RN Referência |
|---|---|---|
| **Autenticação** | Supabase Auth + JWT com claims de papel | RN-193 |
| **Bloqueio por tentativas** | @nestjs/throttler: 5 tentativas / 15 min, bloqueio 30 min | RN-002 |
| **Expiração de sessão** | 60 min de inatividade, renovação automática de JWT | RN-003 |
| **Controle de acesso** | Guards NestJS validam papel JWT em cada requisição | RN-193 |
| **Mascaramento de dados** | Dados pessoais mascarados em listagens e buscas | RN-012 |
| **Acesso mínimo** | Analistas RS só recebem dados dos seus Casos via filtro no backend | RN-012 |
| **Log imutável** | Schema separado append-only, sem UPDATE/DELETE possíveis via Prisma | RN-194 |
| **HTTPS** | TLS 1.3+ obrigatório. HTTP redirect para HTTPS. | — |
| **Headers de segurança** | Helmet: CSP, HSTS, X-Frame-Options | — |
| **Webhooks externos** | Validação HMAC-SHA256 (Celcoin, Meta, ZapSign) | RN-196 |

---

## 7. ADRs Específicos do CRM

| ADR | Decisão | Contexto |
|---|---|---|
| **CRM-ADR-001** | Next.js 15 App Router em vez de React + Vite | CRM tem layouts aninhados complexos (detalhe do Caso com abas) e beneficia de Server Components para carregamento inicial de dados. |
| **CRM-ADR-002** | Supabase Realtime para alertas de SLA | Alertas de SLA (RN-107) exigem atualização em tempo real no Pipeline sem polling. Supabase Realtime elimina polling manual. |
| **CRM-ADR-003** | Sem app mobile nativo na v1.0 | CRM é ferramenta interna de desktop/tablet. App nativo avaliado para Fase 2 conforme adoção em campo (RN-199). |
| **CRM-ADR-004** | Dani-Admin (IA) adiado para Fase 2 | Validação operacional do CRM sem IA primeiro. Integração LangChain.js + GPT-4o na Fase 2. |

---

## 8. Tecnologias Proibidas

| Tecnologia | Alternativa aprovada | Motivo da proibição |
|---|---|---|
| Next.js 14 ou inferior | Next.js 15+ | App Router estável apenas no 15+. |
| Pages Router | App Router | Arquitetura legada sem Server Components. |
| Class components React | Componentes funcionais + hooks | Padrão ShiftLabs. Output de IA inconsistente com class components. |
| Mongoose / TypeORM / Drizzle | Prisma 6+ | ORM único aprovado pelo baseline. |
| PostgreSQL auto-gerenciado | Supabase | Sem ops de banco em produção. |
| Styled-components / Emotion | Tailwind CSS 4 | CSS-in-JS proibido pelo baseline. |
| Axios | Fetch nativo + TanStack Query | Axios adiciona peso sem benefício perceptível com TanStack Query. |
| React Query v3 ou inferior | TanStack Query v5+ | Breaking changes relevantes. Usar versão atual. |
| Spinner global de página | Skeleton screens | Motion Spec (Doc 04): spinners proibidos, exceto em botões de ação. |
| Moment.js | `date-fns` ou `Temporal API` | Moment.js depreciado. |
| `@supabase/auth-helpers` v1 | `@supabase/ssr` (Next.js 15) | Helper legado incompatível com App Router. |
