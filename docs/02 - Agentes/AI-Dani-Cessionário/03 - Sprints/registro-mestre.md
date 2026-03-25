# Registro Mestre de Requisitos — AI-Dani-Cessionário

| **Produto** | AI-Dani-Cessionário |
|---|---|
| **Pipeline** | ShiftLabs v2.3 — Fase 3 |
| **Gerado em** | 24/03/2026 |
| **Total de REQs** | 157 |
| **Cobertura** | 100% dos docs D01–D29 |

---

## 2.1 Regras de Negócio (D01)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-001 | D01 RN-DC-001 | A Dani acessa apenas: tabelas de oportunidade (leitura), histórico de conversas do cessionário autenticado, perfil do cessionário (read-only), histórico de alertas próprios |
| REQ-002 | D01 RN-DC-002 | A Dani NUNCA acessa: dados pessoais do Cedente, cenário do Cedente (A/B/C/D), informações de outros Cessionários, dados de negociações de outros Cessionários, tabelas de sistema (users, admin_logs, financeiro_repasse) |
| REQ-003 | D01 RN-DC-003 — Camada 1 | Isolamento: scope filter — toda query inclui `WHERE cessionario_id = :id` injetado pelo `CessionarioOwnerGuard` |
| REQ-004 | D01 RN-DC-003 — Camada 2 | Isolamento: context filter — system prompt reforça que a Dani só responde sobre dados do Cessionário autenticado |
| REQ-005 | D01 RN-DC-003 — Camada 3 | Isolamento: system prompt reinforcement — proibição explícita de revelar dados de outros Cessionários ou do Cedente |
| REQ-006 | D01 RN-DC-004 | Mensagem padrão para dados bloqueados: "Essa informação não está disponível para o seu perfil. Para mais detalhes sobre a transação, entre em contato com o suporte via negociação." |
| REQ-007 | D01 RN-DC-004 | Após 2 insistências: mesma mensagem de recusa + alternativa. Após 3ª insistência: recusa sem alternativas |
| REQ-008 | D01 RN-DC-005 | Mensagem de boas-vindas no primeiro acesso: "Olá! Sou a Dani, sua Analista de Oportunidades. Posso analisar riscos, comparar imóveis e simular retornos para você. Como posso ajudar?" |
| REQ-009 | D01 RN-DC-006 | 3 pontos de entrada do chat: (1) botão "Consultar Dani" na Tela de Oportunidade; (2) Widget Top 3 no Dashboard; (3) FAB global em todas as telas |
| REQ-010 | D01 RN-DC-007 | Autenticação da Dani por herança de sessão JWT — a Dani não possui autenticação própria; herda o JWT da plataforma Repasse Seguro |
| REQ-011 | D01 RN-DC-008 | 4 conversation starters: (1) "Quais são as melhores oportunidades para mim hoje?"; (2) "Tenho R$ 500.000 para investir. O que recomenda?"; (3) "Me explica como funciona a comissão do comprador."; (4) "Qual o prazo para depósito em Escrow?" |
| REQ-012 | D01 RN-DC-011 | Análise de oportunidade individual inclui: Δ (Tabela Atual − Tabela Contrato), Comissão Comprador, Custo Total no Escrow, Score de Risco (1–10), Comparativo Regional, Gráfico de Valorização |
| REQ-013 | D01 RN-DC-011 — edge case | Se oportunidade sai do marketplace durante simulação: informar na próxima mensagem + oferecer até 3 sugestões de oportunidades semelhantes como chips de ação rápida |
| REQ-014 | D01 RN-DC-012 | Score de risco em escala 1–10: 1–3 = baixo (verde), 4–6 = moderado (amarelo), 7–10 = alto (vermelho). Cor + rótulo obrigatórios (nunca apenas cor) |
| REQ-015 | D01 RN-DC-013 | Cálculo de Comissão Comprador: se Δ > 0 → `20% × Δ`; se Δ ≤ 0 → `20% × Valor Pago pelo Cedente` |
| REQ-016 | D01 RN-DC-013 — edge case | Quando Δ = 0: comissão calculada sobre o Valor Pago pelo Cedente (fallback automático). Dani exibe nota explicativa sobre o critério de fallback |
| REQ-017 | D01 RN-DC-014 | Custo Total de Escrow = Preço do Repasse + Comissão Comprador |
| REQ-018 | D01 RN-DC-015 | Comparação: máximo 5 oportunidades simultaneamente. Critério padrão: melhor relação retorno/risco. Desempate por maior Δ |
| REQ-019 | D01 RN-DC-015 — edge case | Ao solicitar mais de 5 oportunidades: "Consigo comparar até 5 oportunidades de uma vez. Qual grupo você gostaria de analisar primeiro?" + sugestão de subdivisão |
| REQ-020 | D01 RN-DC-015 — edge case | Duas oportunidades com mesmo score de risco: desempate por maior Δ. Dani informa o critério de desempate |
| REQ-021 | D01 RN-DC-016 | Simulação de custos para uma proposta: dado o valor proposto, calcular Comissão + Custo Total + ROI (3 cenários). Aviso de projeção obrigatório |
| REQ-022 | D01 RN-DC-016 — edge case | Proposta com valor maior que a Tabela Atual: ROI seria negativo. Dani aceita simulação, informa perda estimada e inclui aviso de projeção |
| REQ-023 | D01 RN-DC-017 | ROI com 3 cenários: Conservador (−20% da Tabela Atual base), Base, Otimista (+20% da Tabela Atual base). Fórmula: `(Tabela Atual − Custo Total) ÷ Custo Total × 100` |
| REQ-024 | D01 RN-DC-018 | Simulação de contraproposta: dado novo valor proposto em negociação ativa, calcular nova Comissão, novo Escrow e diferença vs. proposta anterior (seta indicativa) |
| REQ-025 | D01 RN-DC-019 | Simulação de portfólio: dado conjunto de oportunidades, mostrar custo total agregado, ROI médio e distribuição de risco |
| REQ-026 | D01 RN-DC-020 | Simulação de impacto de variação: dado % de variação na Tabela Atual, recalcular ROI e comissão |
| REQ-027 | D01 RN-DC-021 | Top 3 de oportunidades em destaque no Dashboard: personalizado por perfil; se perfil incompleto, baseado em dados gerais de mercado + banner informativo |
| REQ-028 | D01 RN-DC-022 | Resposta a perguntas sobre regras da plataforma: Escrow (prazo padrão 10 dias úteis, extensão +5 dias, reversão 15 dias corridos), ZapSign (5 dias úteis, lembretes D+2 e D+4), KYC (automático ≤30 min, manual ≤2 dias úteis) |
| REQ-029 | D01 RN-DC-023 | Calculadora de Comissão como fallback: funciona sem o agente de IA. Módulo determinístico independente do `AgenteModule` |
| REQ-030 | D01 RN-DC-024 | Desligamento automático por taxa de erro: >10% em 15 min → alerta apenas; >30% em 15 min → desligamento automático; reativação somente manual por Admin |
| REQ-031 | D01 RN-DC-025 | Rate limit de mensagens no webchat: 30 mensagens/hora por `cessionario_id`. Redis key `rate_limit:webchat:{cessionario_id}` TTL 3600s. Janela deslizante |
| REQ-032 | D01 RN-DC-026 | Fluxo principal — análise individual: FAB → Chat → Análise automática (se contexto OPR carregado) → Chips de ação rápida |
| REQ-033 | D01 RN-DC-027 | Fluxo de contraproposta em negociação ativa: FAB na tela de negociação → Chat com contexto → Simulação de contraproposta |
| REQ-034 | D01 RN-DC-028 | A Dani recusa submeter propostas em nome do Cessionário. Resposta 1ª vez: instrução para acessar a tela da oportunidade. Insistência: recusa reiterada sem alternativas |
| REQ-035 | D01 RN-DC-029 | SLA de resposta: análise individual ≤5s, comparação ≤10s, simulação ≤5s, resposta operacional ≤5s. Latência acima de 2× SLA: alerta para o Cessionário com opção de aguardar ou tentar novamente |
| REQ-036 | D01 RN-DC-040 | Vinculação do WhatsApp ao perfil (Fase 2): 3 etapas — (1) informar número + validação em tempo real; (2) inserir OTP SMS; (3) confirmar código enviado ao WhatsApp |
| REQ-037 | D01 RN-DC-041 | Validação do OTP: 6 dígitos, TTL 15 min, 3 tentativas/hora. Hard block após 5 falhas consecutivas (TTL 30 min = 1800s) |
| REQ-038 | D01 RN-DC-042 | Segunda etapa — confirmação pelo WhatsApp: código enviado ao WhatsApp, válido por 24 horas |
| REQ-039 | D01 RN-DC-044 | Desvinculação do WhatsApp: via modal de confirmação na plataforma OU via comando PARAR no WhatsApp (LGPD — imediato e irreversível). Audit log com evento `opted_out` |

---

## 2.2 Stack Tecnológico (D02)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-040 | D02 | Monorepo: Turborepo + pnpm workspaces. Raiz: `repasse-seguro-dani/` |
| REQ-041 | D02 | Backend: Node.js ≥20 LTS, NestJS ≥10, TypeScript ≥5.3, Prisma ≥5.x |
| REQ-042 | D02 | Backend ORM/DB: Supabase PostgreSQL, pgvector extension |
| REQ-043 | D02 | Backend Cache: Redis ≥7.x via ioredis ≥5.x |
| REQ-044 | D02 | Backend Queue: RabbitMQ ≥3.12 via @nestjs/microservices + amqplib |
| REQ-045 | D02 | Backend Validators: class-validator ≥0.14, class-transformer ≥0.5 |
| REQ-046 | D02 | Backend Logging: Pino ≥8.x |
| REQ-047 | D02 | Backend Error tracking: Sentry ≥7.x |
| REQ-048 | D02 | Backend LLM observability: Langfuse ≥2.x |
| REQ-049 | D02 | Frontend: React ≥19, Vite ≥5.x |
| REQ-050 | D02 | Frontend Routing: TanStack Router ≥1.x |
| REQ-051 | D02 | Frontend Data: TanStack Query ≥5.x |
| REQ-052 | D02 | Frontend State: Zustand ≥4.x |
| REQ-053 | D02 | Frontend UI: Tailwind CSS ≥4.x + shadcn/ui |
| REQ-054 | D02 | Frontend Animation: Framer Motion ≥11.x |
| REQ-055 | D02 | AI Stack: GPT-4o, OpenAI SDK ≥4.x, Vercel AI SDK ≥3.x, LangChain.js ≥0.2.x |
| REQ-056 | D02 | AI RAG: pgvector — chunk 512 tokens, overlap 64, modelo `text-embedding-3-small`, namespace por `cessionario_id` |
| REQ-057 | D02 Redis keys | `dani:session:{session_id}` TTL 1800s; `dani:rate:webchat:{id}` TTL 3600s; `dani:rate:otp:{phone_hash}` TTL 3600s; `dani:block:otp:{phone_hash}` TTL 1800s; `dani:status:agent` TTL variável; `dani:cache:calc:{opr_id}:{val_hash}` TTL 300s |
| REQ-058 | D02 RabbitMQ | Fila `dani.notificacoes` (fanout, DLQ, 3x retry 5s/15s/30s); fila `dani.whatsapp` (direct, DLQ, mesmo retry); fila `dani.agent_monitor` (direct, DLQ, retry 2s/5s/10s) |
| REQ-059 | D02 ADR-001 | EvolutionAPI v2.x para WhatsApp (Fase 2) — decisão de arquitetura registrada |
| REQ-060 | D02 ADR-002 | SSE (Server-Sent Events) obrigatório para streaming do LLM via Vercel AI SDK |

---

## 2.3 Modelo de Dados (D12 + D13)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-061 | D12/D13 | Tabela `dani_conversas`: `id UUID PK`, `cessionario_id UUID NOT NULL`, `titulo TEXT`, `status StatusConversa`, `canal CanalDani`, `opr_id TEXT`, `created_at`, `updated_at`, `expires_at` |
| REQ-062 | D12/D13 | Tabela `dani_mensagens`: `id UUID PK`, `conversa_id UUID FK`, `cessionario_id UUID NOT NULL`, `remetente RemetenteMensagem`, `conteudo TEXT`, `metadata JSONB`, `created_at`, `expires_at` (join via conversa) |
| REQ-063 | D12/D13 | Tabela `dani_sessoes`: `id UUID PK`, `cessionario_id UUID NOT NULL`, `jwt_session_hash TEXT`, `estado_agente EstadoAgente`, `ip_address`, `user_agent`, `created_at`, `expires_at` |
| REQ-064 | D12/D13 | Tabela `dani_contextos_opr`: `id UUID PK`, `cessionario_id UUID NOT NULL`, `opr_id TEXT NOT NULL`, `dados_oportunidade JSONB`, `created_at`, `expires_at` |
| REQ-065 | D12/D13 | Tabela `dani_alertas`: `id UUID PK`, `cessionario_id UUID NOT NULL`, `tipo TipoAlerta`, `status StatusAlerta`, `titulo TEXT`, `corpo TEXT`, `opr_id TEXT`, `lido_em TIMESTAMP`, `created_at`. Tipos: NOVA_OPORTUNIDADE, OPORTUNIDADE_ENCERRADA, NEGOCIACAO_ATUALIZADA, ESCROW_PRAZO, ZAPSIGN_PENDENTE |
| REQ-066 | D12/D13 | Tabela `dani_vinculacoes_whatsapp`: `id UUID PK`, `cessionario_id UUID UNIQUE NOT NULL`, `numero_hash TEXT`, `numero_sufixo TEXT` (últimos 4 dígitos), `estado EstadoVinculacaoWhatsapp`, `otp_hash TEXT` (bcrypt cost 12), `tentativas_otp INT`, `otp_expires_at`, `created_at`, `updated_at`. Estados: NAO_VINCULADO, OTP_SMS_ENVIADO, AGUARDANDO_CONFIRMACAO_WA, VINCULADO, DESVINCULADO |
| REQ-067 | D12/D13 | Tabela `dani_rate_limits_audit`: `id UUID PK`, `cessionario_id UUID`, `evento EventoRateLimitAudit`, `canal TEXT`, `ip_address`, `created_at` (append-only, sem updated_at). Eventos: LIMITE_ATINGIDO, DESBLOQUEADO, HARD_BLOCK_OTP |
| REQ-068 | D12/D13 | Migration command: `npx prisma migrate dev --name init_dani_module` |
| REQ-069 | D12/D13 | Job de cleanup: `src/agente/jobs/cleanup-conversas.job.ts` — deleta conversas expiradas e mensagens relacionadas diariamente |
| REQ-070 | D12/D13 | Índices obrigatórios: `cessionario_id` em todas as 7 tabelas; `status` em `dani_conversas`; `created_at` em `dani_mensagens`; `lido_em` em `dani_alertas`; `estado` em `dani_vinculacoes_whatsapp` |

---

## 2.4 Endpoints de API (D16)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-071 | D16 | Base URL: `https://api.repasseseguro.com.br/api/v1`. Formato de erro: `{ error: { code, message, details? } }` |
| REQ-072 | D16 | `POST /dani/chat` — enviar mensagem (não-streaming). Auth: Bearer JWT. Guard: CessionarioOwnerGuard |
| REQ-073 | D16 | `GET /dani/stream` — SSE streaming (Accept: text/event-stream). ADR-002 |
| REQ-074 | D16 | `GET /dani/conversas` — listar conversas do Cessionário (paginado, per_page max 100) |
| REQ-075 | D16 | `GET /dani/conversas/{conversa_id}/mensagens` — listar mensagens de uma conversa |
| REQ-076 | D16 | `POST /dani/conversas` — criar nova conversa |
| REQ-077 | D16 | `PATCH /dani/conversas/{conversa_id}/encerrar` — encerrar conversa |
| REQ-078 | D16 | `POST /calculadora/calcular` — cálculo determinístico (sem agente) |
| REQ-079 | D16 | `GET /dani/alertas` — listar alertas do Cessionário |
| REQ-080 | D16 | `GET /dani/alertas/count` — contagem de alertas não lidos (badge FAB) |
| REQ-081 | D16 | `PATCH /dani/alertas/{alerta_id}/lido` — marcar alerta como lido |
| REQ-082 | D16 | `GET /whatsapp/status` — status de vinculação WhatsApp |
| REQ-083 | D16 | `POST /whatsapp/vincular` — iniciar vinculação (enviar OTP SMS) |
| REQ-084 | D16 | `POST /whatsapp/verificar-otp` — verificar OTP |
| REQ-085 | D16 | `DELETE /whatsapp/vincular` — desvincular WhatsApp |
| REQ-086 | D16 | `POST /whatsapp/webhook` — webhook EvolutionAPI (auth: API Key, não Bearer) |
| REQ-087 | D16 | `GET /oportunidades/{opr_id}` — dados da oportunidade (read-only) |
| REQ-088 | D16 | `POST /dani/sessao` — registrar sessão |
| REQ-089 | D16 ⚠️ AMBÍGUO | D29 referencia `POST /api/v1/agente/chat` e `POST /api/v1/calculadora/simular` — paths diferem de D16 (`/dani/chat` e `/calculadora/calcular`). Adotar D16 como fonte normativa de endpoints |
| REQ-090 | D16 | Paginação: `{ data: [...], meta: { page, per_page, total, total_pages } }`. Headers de rate limit: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` |

---

## 2.5 Autenticação OAuth (D18)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-091 | D18 | JWT: access token TTL 15 min (in-memory JS), refresh token TTL 7 dias (httpOnly cookie `dani_refresh_token`) |
| REQ-092 | D18 | Redis session: `dani:session:{session_id}` TTL 604800s (7 dias) |
| REQ-093 | D18 | Redis blacklist: `dani:revoked:refresh:{jti}` |
| REQ-094 | D18 | Endpoints de auth: `POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout`, `GET /api/v1/auth/me` |
| REQ-095 | D18 | Roles permitidos em endpoints Dani: `CESSIONARIO`, `ADMIN` |
| REQ-096 | D18 | OTP: hash bcrypt cost 12. Redis key `dani:otp:{phone_hash}` TTL 900s. Phone hash: SHA-256 |
| REQ-097 | D18 | Checklist de segurança: 13 itens (S-01 a S-13) — incluindo HTTPS obrigatório, no PII em logs, `cessionario_id` nunca em URL pública |

---

## 2.6 Agente de IA (D19)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-098 | D19 | Agente híbrido: informacional + executor. 5 tools: `buscarOportunidades`, `calcularComissao`, `analisarRisco`, `buscarComparativoRegional`, `verificarStatusNegociacao` |
| REQ-099 | D19 | Timeouts por tool: `buscarOportunidades` 5s, `calcularComissao` 2s, `analisarRisco` 10s, `buscarComparativoRegional` 8s, `verificarStatusNegociacao` 5s |
| REQ-100 | D19 | Retry por tool: todas 2x backoff, exceto `calcularComissao` 1x |
| REQ-101 | D19 | GPT-4o: temperature 0.3, max_tokens 2048, timeout 30s, max retries 3 (backoff 1s/2s/4s) |
| REQ-102 | D19 | System prompt: `src/agente/prompts/dani-system-prompt.v1.ts` (versionado — nunca hardcoded) |
| REQ-103 | D19 | Memória de curto prazo: Redis `dani:session:{session_id}` TTL 1800s, max 50 mensagens por sessão (FIFO) |
| REQ-104 | D19 | Memória de longo prazo: pgvector tabela `agent_memory`, TTL 90 dias. Triggers de ingestão: pós-análise, pós-simulação, pós-comparação |
| REQ-105 | D19 ADR-003 | `CalculadoraModule` completamente isolado — sem dependência do `AgenteModule` |
| REQ-106 | D19 ADR-004 | RAG namespace obrigatório por `cessionario_id` — namespace no pgvector separa embeddings por cessionário |
| REQ-107 | D19 | Langfuse: `userId = sha256(cessionario_id)` (nunca UUID raw — LGPD). Campos obrigatórios: `session_id`, `cessionario_id_hash`, `input_tokens`, `output_tokens`, `latency_ms` |
| REQ-108 | D19 | Takeover: confidence_score < 0.80 em 3 turns consecutivos → publicar evento em `dani.agent_monitor` |

---

## 2.7 Notificações (D21)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-109 | D21 | 10 templates (TPLF-001 a TPLF-010) em `src/notificacoes/templates/` |
| REQ-110 | D21 | Templates críticos (sem opt-out): TPLF-004, TPLF-005, TPLF-006, TPLF-007, TPLF-008 |
| REQ-111 | D21 | RabbitMQ `dani.notificacoes`: fanout, DLQ, retry 3x (5s/15s/30s) |
| REQ-112 | D21 | RabbitMQ `dani.whatsapp`: direct, DLQ, retry 3x (5s/15s/30s) |
| REQ-113 | D21 | RabbitMQ `dani.agent_monitor`: direct, DLQ, retry 3x (2s/5s/10s) |
| REQ-114 | D21 | WhatsApp rate limit: 1 mensagem/minuto por número |
| REQ-115 | D21 | LGPD — comando PARAR via WhatsApp: desvinculação imediata, irreversível, audit log com `opted_out` |
| REQ-116 | D21 | Endpoints de preferências: `GET /api/v1/dani/alertas/preferencias`, `PUT /api/v1/dani/alertas/preferencias`, `POST /api/v1/dani/whatsapp/desvincular` |

---

## 2.8 Telas e UX (D06, D07, D08, D09, D11)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-117 | D06 T-DC-001 | FAB Global: `position: fixed; bottom: 24px; right: 24px; z-index: 9999`. 56×56px. Badge `--destructive` com contagem de alertas não lidos |
| REQ-118 | D06 T-DC-002 | Chat sem contexto: 400px × 600px desktop; 100vw × 100dvh mobile (≤640px). Histórico últimos 90 dias |
| REQ-119 | D06 T-DC-003 | Chat com contexto OPR: contexto pré-carregado (código OPR, valores, localização). Dani inicia análise automaticamente |
| REQ-120 | D06 T-DC-004 | Chat com contexto de negociação: dados da negociação ativa pré-carregados. Módulo de contraproposta ativo |
| REQ-121 | D06 T-DC-005 | Análise individual inline: card com Δ, Comissão, Custo Total, Score de Risco (badge colorido), ROI (3 cenários), Comparativo Regional. Ações rápidas: "Simular proposta" / "Comparar com similares" / "Salvar para alertas" |
| REQ-122 | D06 T-DC-006 | Tabela comparativa inline: colunas OPR, Δ, Comissão, Custo Total Escrow, Score de Risco, Localização, Tipologia. Linha "Melhor opção" em `--primary`. Máx 5 linhas |
| REQ-123 | D06 T-DC-007 | Simulação de proposta inline: Comissão, Depósito total no Escrow, ROI 3 cenários, aviso de projeção obrigatório. Ações: "Simular outro valor" / "Ir para a oportunidade" |
| REQ-124 | D06 T-DC-008 | Simulação de contraproposta inline: nova Comissão, novo Escrow, diferença vs. proposta anterior (seta indicativa), ROI ajustado 3 cenários. Ação: "Ir para tela de negociação" |
| REQ-125 | D06 T-DC-009 | Widget Top 3 no Dashboard: seção "Oportunidades em Destaque". 3 cards: Δ, comissão estimada, score de risco, localização. Estado perfil incompleto: banner com link |
| REQ-126 | D06 T-DC-010 | Banner de Fallback: texto "Modo básico — sem análise da IA". Cor `--agent-fallback` ⚠️ AMBÍGUO D09 valor `#0069A8` não confirmado em D03 |
| REQ-127 | D06 T-DC-011 | Estado de Rate Limit: input desabilitado (fundo cinza), botão enviar inativo, contador regressivo mm:ss. Reativação: pulse 500ms |
| REQ-128 | D06 T-DC-012 | Vinculação WhatsApp (Fase 2): seção em "Meu Perfil > WhatsApp". 3 etapas com estados vinculado/não vinculado. Número mascarado `(XX) •••••-XXXX` |
| REQ-129 | D09 | 4 estados obrigatórios por componente: Skeleton → Empty → Error → Populated. Spinners proibidos em todo o produto |
| REQ-130 | D09 | Componentes críticos de domínio: `RiskScoreBadge`, `OprStatusBadge`, `AgentStatusBanner`, `RateLimitCounter`, `CommissionCard`, `ComparisonTable`, `OTPInput`, `ROIDeltaBadge` |
| REQ-131 | D09 | RBAC frontend: todas as telas exigem role `CESSIONARIO_AUTHENTICATED`. T-DC-012 adiciona `WhatsAppPhaseGuard` |
| REQ-132 | D08 | Strings normativas — não podem ser alteradas sem aprovação de produto: mensagem de recusa (7 variantes tabeladas), aviso de projeção, fallback banner, rate limit |
| REQ-133 | D08 | Formatação de dados: monetário `R$ X.XXX,XX`; percentual `XX%`; score `N/10`; OPR `OPR-XXXX-XXXX`; telefone mascarado `(XX) •••••-XXXX`; tempo `mm:ss` |
| REQ-134 | D11 | Plataformas suportadas Fase 1: iOS Safari ≥16, Chrome Android ≥110 (browser). Sem React Native |
| REQ-135 | D11 | Deep links obrigatórios: `?chat=open` em `/cessionario/dashboard` e `/cessionario/oportunidade/{opr_id}`. URL de vinculação WhatsApp: `/cessionario/perfil/whatsapp` |

---

## 2.9 Integrações Externas (D17)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-136 | D17 | OpenAI P0: Tier 4 recomendado (1M tokens/min). Timeout 30s. Retry 3x backoff exponencial (1s/2s/4s). Fallback: Calculadora. Endpoint: `POST https://api.openai.com/v1/chat/completions` |
| REQ-137 | D17 | Supabase P0: Prisma connection pool limit=10 (PgBouncer). Sem fallback para banco indisponível. SLA 99.9% |
| REQ-138 | D17 | Redis P1: timeout 100ms (fail open). Hard block OTP: fail closed. ioredis retry 3x (100ms/500ms/2s) |
| REQ-139 | D17 | EvolutionAPI P1 (Fase 2): endpoint `/message/sendText`. Webhook `POST /api/v1/whatsapp/webhook` com header `x-api-key`. Timeout 10s. Retry 3x (5s/15s/30s). DLQ se falha definitiva |
| REQ-140 | D17 | Langfuse P1: assíncrono, timeout 5s, retry 2x. Falha não bloqueia resposta ao usuário |
| REQ-141 | D17 | Credenciais: `OPENAI_API_KEY`, `DATABASE_URL`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_PUBLIC_KEY`, `REDIS_URL`, `EVOLUTIONAPI_KEY`, `EVOLUTIONAPI_URL`, `EVOLUTIONAPI_WEBHOOK_SECRET`. Rotações definidas por integração |

---

## 2.10 Observabilidade, CI/CD e Operações (D24, D25, D26, D27, D28, D29)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-142 | D25 | Stack: Pino (JSON estruturado) + Langfuse + Sentry + RabbitMQ Management. SLO: latência p95 ≤5s, disponibilidade ≥99.5% |
| REQ-143 | D25 | Retenção de logs: debug 7 dias, info 30 dias, warn/error 90 dias. Langfuse traces: 90 dias |
| REQ-144 | D25 | Correlation ID gerado por request, propagado em todos os logs, traces e filas. `cessionario_id` como SHA-256 em logs |
| REQ-145 | D24 | 4 ambientes: Local (localhost), Preview (Vercel PR), Staging (develop), Produção (main + tag semver) |
| REQ-146 | D24 | CI/CD: GitHub Actions. Pipeline: lint → tsc → secrets scan → unit tests (coverage ≥80%) → integration → build → deploy |
| REQ-147 | D24 | Railway (API + workers), Vercel (frontend web). Rollback em <5 min via Railway CLI |
| REQ-148 | D27 | Frameworks de teste: Vitest (unit + integração), Playwright (E2E), Pact (contrato) |
| REQ-149 | D27 | Cobertura mínima: unitário ≥80%, integração ≥70%. P0 críticos: isolamento `cessionario_id` 100%, cálculo de comissão 100%, OTP/rate limit/hard block 100% |
| REQ-150 | D28 | Bloqueantes de merge: 4 estados por componente, sem spinners, barrel exports, sem import cross-feature, `CessionarioOwnerGuard` em todos os endpoints, `WHERE cessionario_id` em todas as queries, sem PII em logs, sem circular dependency entre CalculadoraModule e AgenteModule |
| REQ-151 | D29 | Go-Live: lançamento controlado em 4 fases (T-7, T-3, T-1, D-day). Launch window: Seg–Sex 09h–12h. On-call obrigatório por 48h pós-go-live |
| REQ-152 | D29 | Smoke tests pós-deploy: `GET /health`, `POST /api/v1/agente/chat` (200 ou 401 em <2s), `POST /api/v1/calculadora/simular` (200 ou 401 em <500ms) ⚠️ AMBÍGUO — paths divergem de D16 |

---

## 2.11 Contribuição, Brand e Ambiente (D03, D04, D22, D23)

| ID | Origem | Requisito Atômico |
|---|---|---|
| REQ-153 | D22 | Pré-requisitos: Node.js ≥20 LTS, pnpm ≥9, Docker Desktop ≥25, Git ≥2.40, Supabase CLI ≥1.x |
| REQ-154 | D22 | 28 variáveis de ambiente (17 sensíveis, 11 não-sensíveis). Setup em <15 min. `pnpm health` valida todos os serviços |
| REQ-155 | D23 | Branching: trunk-based simplificado. Branches: `main` (2 aprovações), `develop` (1 aprovação), `feature/*`, `bugfix/*`, `hotfix/*` |
| REQ-156 | D23 | Commits: Conventional Commits v1.0.0 obrigatório. Hook `commit-msg` via Husky. Escopos aprovados: `chat`, `agente`, `calculadora`, `auth`, `opr`, `alerta`, `whatsapp`, `ux`, `api`, `db`, `infra` |
| REQ-157 | D03 | Brand: tema herdado do Repasse Seguro. Tokens de extensão: `--risk-low/medium/high`, `--risk-*-bg`, `--agent-fallback (#0069A8)` ⚠️ AMBÍGUO sem confirmação em D03 source code, `--status-available/negotiating/closed`. Tipografia: Inter Variable |

---

## Resumo de Cobertura

| Domínio | REQs | % |
|---|---|---|
| Regras de Negócio (D01) | REQ-001 a REQ-039 | 39 |
| Stack Tecnológico (D02) | REQ-040 a REQ-060 | 21 |
| Modelo de Dados (D12/D13) | REQ-061 a REQ-070 | 10 |
| API (D16) | REQ-071 a REQ-090 | 20 |
| Autenticação (D18) | REQ-091 a REQ-097 | 7 |
| Agente IA (D19) | REQ-098 a REQ-108 | 11 |
| Notificações (D21) | REQ-109 a REQ-116 | 8 |
| Telas e UX (D06–D11) | REQ-117 a REQ-135 | 19 |
| Integrações Externas (D17) | REQ-136 a REQ-141 | 6 |
| Operações/CI/CD (D24–D29) | REQ-142 a REQ-152 | 11 |
| Contribuição/Brand/Ambiente (D03/D04/D22/D23) | REQ-153 a REQ-157 | 5 |
| **Total** | **REQ-001 a REQ-157** | **157** |

### Itens ⚠️ AMBÍGUO registrados

| Item | Descrição | Ação |
|---|---|---|
| REQ-089 | D29 smoke tests usam paths `/agente/chat` e `/calculadora/simular`; D16 define `/dani/chat` e `/calculadora/calcular` | Adotar D16 como normativo |
| REQ-126 | Token `--agent-fallback (#0069A8)` em D09/D06 sem confirmação de valor hex em D03 source code | Usar D09 valor `#0069A8` como referência provisória |
| REQ-152 | Mesma divergência de paths D29 vs D16 nos smoke tests de go-live | Adotar D16 como normativo |
