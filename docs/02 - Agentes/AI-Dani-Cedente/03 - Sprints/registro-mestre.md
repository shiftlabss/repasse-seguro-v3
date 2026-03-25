# Registro Mestre de Requisitos — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| **Gerado por** | Pipeline de Sprints ShiftLabs v2.3 |
| **Data** | 2026-03-24 |
| **Total de requisitos** | 187 |
| **Fonte** | D01–D29 em `docs/02 - Agentes/AI-Dani-Cedente/02 - Desenvolvimento/` |

---

> Este arquivo é a FONTE DA VERDADE para cobertura. Todo requisito atômico extraído dos 29 documentos de especificação está registrado aqui. A coluna Sprint é preenchida na Etapa 3.

---

## Tabela de Requisitos

| ID | Doc Fonte | Seção | Tipo | Descrição | Valor/Detalhe | Sprint |
|---|---|---|---|---|---|---|
| REQ-001 | D01 | 1 | Glossário/Negócio | Cedente — definição | Proprietário original do contrato imobiliário que deseja repassar | CROSS |
| REQ-002 | D01 | 1 | Glossário/Negócio | Cessionário — definição e isolamento | Investidor/comprador; Cedente não vê dados pessoais do Cessionário | CROSS |
| REQ-003 | D01 | 1 | Regra de negócio | Cenários A/B/C/D — confidencialidade | Nunca revelado ao Cessionário nem à Dani-Cessionário | S4 |
| REQ-004 | D01 | 1 | Regra de negócio | Δ (Delta) — fórmula de cálculo | Δ = Tabela Atual − Tabela Contrato; se Δ ≤ 0 → fallback = Valor Pago pelo Cedente | S4 |
| REQ-005 | D01 | 1 | Regra de negócio | Dossiê — documentos obrigatórios | 5 documentos obrigatórios + 1 condicional (ver D01 seção 6) | S5 |
| REQ-006 | D01 | 1 | Glossário/Negócio | Envelope ZapSign — definição | Pacote de assinatura eletrônica para formalização do contrato de cessão | S7 |
| REQ-007 | D01 | 1 | Regra de negócio | Escrow — prazo padrão de depósito | 10 dias úteis após aceite da proposta | S7 |
| REQ-008 | D01 | 1 | Regra de negócio | Escrow — extensão de prazo | +5 dias úteis; Cedente consultado (24h para responder; silêncio = aprovação automática); Admin confirma | S7 |
| REQ-009 | D01 | 1 | Regra de negócio | Escrow — reversão | 15 dias corridos se negociação não concluída | S7 |
| REQ-010 | D01 | 1 | Regra de negócio | KYC — documentos exigidos | Documento de identidade (frente e verso) + selfie com vivacidade + comprovante de endereço ≤90 dias | S2 |
| REQ-011 | D01 | 1 | Glossário/Negócio | OPR-XXXX-XXXX | Código identificador único de oportunidade | S4 |
| REQ-012 | D01 | 1 | Glossário/Negócio | Tabela Atual / Tabela Contrato | Base de cálculo do Δ | S4 |
| REQ-013 | D01 | 1 | Regra de negócio | Takeover — threshold de confiança | Threshold padrão: 80%; RN-DA-033 referência | S9 |
| REQ-014 | D01 | 2.1 | Identidade | Nome exibido na interface | "Dani" | CROSS |
| REQ-015 | D01 | 2.1 | Identidade | Nome interno do produto | AI-Dani-Cedente | CROSS |
| REQ-016 | D01 | 2.1 | Identidade | Persona da Dani | Guardiã do Retorno — consultora imobiliária experiente, empática, orientada a resultado | CROSS |
| REQ-017 | D01 | 2.2 | Regra de negócio | Dados que a Dani usa | Oportunidade, cenários, Tabela Atual/Contrato, Valor Pago, propostas (valor/status sem identidade), histórico, dossiê, SLAs | S3 |
| REQ-018 | D01 | 2.3 | Regra de negócio | Dados que a Dani NÃO usa | Dados pessoais/financeiros de Cessionários, dados de outros Cedentes, decisões Admin, aconselhamento jurídico/fiscal | S2 |
| REQ-019 | D01 | 2.4 | Regra de negócio | Padrão de resposta — fundamentação | Toda resposta fundamentada nos dados reais do contrato e oportunidade do Cedente | S3 |
| REQ-020 | D01 | 2.4 | Regra de negócio | Padrão de resposta — próximo passo | Toda resposta encerra com próximo passo claro | S3 |
| REQ-021 | D01 | 2.4 | Regra de negócio | Padrão de resposta — linguagem acessível | Vocabulário adaptado ao perfil do Cedente | S3 |
| REQ-022 | D01 | 3.1 | Regra de negócio | RN-DCE-001 — Escopo de dados (fluxo completo) | Middleware valida cedente_id do JWT antes de qualquer chamada ao LLM; violação = LGPD e perda de confiança | S2 |
| REQ-023 | D01 | 3.2 | Regra de negócio | RN-DCE-002 — Mensagens padrão para dados bloqueados | 5 tipos de dado bloqueado com mensagem exata definida | S3 |
| REQ-024 | D01 | 4 | Regra de negócio | RN-DCE-005 — Boas-vindas no primeiro acesso (3 estados) | Estado 1: KYC aprovado+oportunidade ativa; Estado 2: KYC pendente; Estado 3: sem oportunidade | S3 |
| REQ-025 | D01 | 4 | Regra de negócio | RN-DCE-006 — 3 pontos de entrada do chat | PE-1 Painel Cedente; PE-2 Tela Oportunidade (carrega contexto); PE-3 Tela Negociação (carrega contexto de proposta) | S3 |
| REQ-026 | D01 | 4 | Regra de negócio | RN-DCE-008 — Sugestões de conversa (4 starters) | "Qual o retorno esperado?" / "Tenho proposta. Vale aceitar?" / "O que falta no dossiê?" / "Quanto tempo demora?" | S3 |
| REQ-027 | D01 | 5.1 | Máquina de estados | OpportunityStatus — Rascunho | Oportunidade cadastrada mas não publicada | S4 |
| REQ-028 | D01 | 5.1 | Máquina de estados | OpportunityStatus — AguardandoValidacao | Dossiê enviado para análise pelo Admin | S4 |
| REQ-029 | D01 | 5.1 | Máquina de estados | OpportunityStatus — Publicada | Visível no marketplace para Cessionários | S4 |
| REQ-030 | D01 | 5.1 | Máquina de estados | OpportunityStatus — EmNegociacao | Oportunidade com proposta ativa de um Cessionário | S4 |
| REQ-031 | D01 | 5.1 | Máquina de estados | OpportunityStatus — EncerradaVendida | Negociação concluída com sucesso + Escrow liberado | S4 |
| REQ-032 | D01 | 5.1 | Máquina de estados | OpportunityStatus — EncerradaRetirada | Cedente retirou a oportunidade do marketplace | S4 |
| REQ-033 | D01 | 5.1 | Transição de estado | Transição Rascunho → AguardandoValidacao | Dossiê enviado + cenário escolhido | S4 |
| REQ-034 | D01 | 5.1 | Transição de estado | Transição AguardandoValidacao → Rascunho | Admin rejeita com pendências | S4 |
| REQ-035 | D01 | 5.1 | Transição de estado | Transição AguardandoValidacao → Publicada | Admin aprova | S4 |
| REQ-036 | D01 | 5.1 | Transição de estado | Transição Publicada → EmNegociacao | Proposta ativa recebida | S4 |
| REQ-037 | D01 | 5.1 | Transição de estado | Transição EmNegociacao → Publicada | Proposta recusada ou expirada | S6 |
| REQ-038 | D01 | 5.1 | Transição de estado | Transição EmNegociacao → EncerradaVendida | Negociação fechada e Escrow liberado | S7 |
| REQ-039 | D01 | 5.1 | Transição de estado | Transição Publicada → EncerradaRetirada | Cedente retira a oportunidade | S4 |
| REQ-040 | D01 | 5 | Regra de negócio | RN-DCE-010 — Cadastro da oportunidade (campos obrigatórios) | Empreendimento (nome, incorporadora, endereço, tipologia, área) + financeiro (Tabela Contrato, Valor Pago, parcelas restantes, Tabela Atual) | S4 |
| REQ-041 | D01 | 5 | Regra de negócio | RN-DCE-010 — Cálculo do Δ e exibição de resumo | Após todos os dados: Dani calcula Δ e exibe resumo antes de avançar | S4 |
| REQ-042 | D01 | 5 | Regra de negócio | RN-DCE-011 — Escolha de cenário (apresentação) | Cada cenário: valor repasse sugerido + retorno líquido estimado + condições de pagamento | S4 |
| REQ-043 | D01 | 5 | Regra de negócio | RN-DCE-011 — Cenários confidenciais | Dani nunca revela cenários ao Cessionário ou a outro agente | S4 |
| REQ-044 | D01 | 5 | Regra de negócio | RN-DCE-011 — Alteração de cenário pós-publicação | Requer contato com suporte (Admin); não pode ser feito diretamente | S4 |
| REQ-045 | D01 | 5 | Regra de negócio | RN-DCE-012 — Retirada da oportunidade do marketplace | Modal de confirmação se Publicada; bloqueia se Em negociação | S4 |
| REQ-046 | D01 | 6.1 | Regra de negócio | Documentos obrigatórios do dossiê — lista | Contrato original, matrícula do imóvel, certidão neg. ônus reais (90d), certidão neg. débitos CPF/CNPJ (90d), comprovante de pagamentos + procuração (condicional) | S5 |
| REQ-047 | D01 | 6 | Regra de negócio | RN-DCE-013 — Status de cada documento do dossiê | 4 estados: Aprovado ✅, Em análise ⏳, Rejeitado ❌, Pendente 📎 | S5 |
| REQ-048 | D01 | 6 | Regra de negócio | RN-DCE-013 — Percentual de conclusão do dossiê | (docs aprovados / total obrigatórios) × 100 | S5 |
| REQ-049 | D01 | 6 | Regra de negócio | RN-DCE-013 — Dani não valida tecnicamente | Apenas informa status retornado pelo sistema Admin | S5 |
| REQ-050 | D01 | 7.1 | Máquina de estados | ProposalStatus — Recebida | Proposta enviada pelo Cessionário, aguardando resposta | S6 |
| REQ-051 | D01 | 7.1 | Máquina de estados | ProposalStatus — Em análise | Cedente está avaliando a proposta | S6 |
| REQ-052 | D01 | 7.1 | Máquina de estados | ProposalStatus — Aceita | Cedente aceitou; negociação inicia | S6 |
| REQ-053 | D01 | 7.1 | Máquina de estados | ProposalStatus — Recusada | Cedente recusou; oportunidade volta a Publicada | S6 |
| REQ-054 | D01 | 7.1 | Máquina de estados | ProposalStatus — Contraproposta enviada | Cedente enviou contraproposta ao Cessionário | S6 |
| REQ-055 | D01 | 7.1 | Máquina de estados | ProposalStatus — Expirada | Proposta não respondida dentro do prazo | S6 |
| REQ-056 | D01 | 7 | Regra de negócio | RN-DCE-014 — Análise de proposta (campos exibidos) | Valor proposto + comparação com tabela (% variação) + retorno líquido estimado + prazo de resposta (SLA) | S6 |
| REQ-057 | D01 | 7 | Regra de negócio | RN-DCE-014 — Tabela comparativa de múltiplas propostas | Colunas: valor proposto, retorno líquido estimado, prazo de resposta | S6 |
| REQ-058 | D01 | 7 | Regra de negócio | RN-DCE-015 — Simulação do retorno líquido — fórmula | Retorno líquido = Valor Repasse − Saldo Devedor | S6 |
| REQ-059 | D01 | 7 | Regra de negócio | RN-DCE-015 — Aviso obrigatório de estimativa | "Este é um valor estimado... Deduções adicionais (impostos, taxas notariais) podem variar. Consulte um especialista." | S6 |
| REQ-060 | D01 | 7 | Regra de negócio | RN-DCE-016 — Envio de contraproposta | Dani não envia por conta própria; orienta para confirmar na tela de negociação | S6 |
| REQ-061 | D01 | 7 | Regra de negócio | RN-DCE-017 — Aceitação de proposta | Modal com: valor aceito + retorno líquido estimado + próximos passos (Escrow); Dani não aceita em nome do Cedente | S6 |
| REQ-062 | D01 | 8.1 | Máquina de estados | EscrowStatus — Aguardando depósito | Proposta aceita; Cessionário tem 10 dias úteis | S7 |
| REQ-063 | D01 | 8.1 | Máquina de estados | EscrowStatus — Depositado | Cessionário realizou o depósito; avança para assinatura | S7 |
| REQ-064 | D01 | 8.1 | Máquina de estados | EscrowStatus — Liberado ao Cedente | Processo de assinatura concluído | S7 |
| REQ-065 | D01 | 8.1 | Máquina de estados | EscrowStatus — Revertido | Negociação não concluída | S7 |
| REQ-066 | D01 | 8 | Regra de negócio | RN-DCE-018 — Alerta proativo escrow próximo do vencimento | Alerta com 2 dias úteis restantes | S7 |
| REQ-067 | D01 | 8 | Regra de negócio | RN-DCE-018 — Extensão de escrow (fluxo 4 estados) | Notificação ao Cedente → resposta 'Sim' (24h) → Admin formaliza; 'Não' (24h) → Admin alertado; silêncio 24h → aprovação automática | S7 |
| REQ-068 | D01 | 9 | Regra de negócio | RN-DCE-019 — Régua de lembretes ZapSign | D+0 notificação inicial; D+2 1º lembrete; D+4 2º lembrete urgente + alerta Admin; D+5 contrato expirado → Admin decide | S7 |
| REQ-069 | D01 | 9 | Regra de negócio | RN-DCE-019 — Prazo máximo de assinatura | 5 dias úteis | S7 |
| REQ-070 | D01 | 10 | Notificação | RN-DCE-020 — Notificações proativas — 10 eventos | Nova proposta; proposta próx. vencimento (24h); extensão Escrow; depósito confirmado; prazo Escrow (2 dias); ZapSign enviado; lembretes D+2/D+4; documento rejeitado; oportunidade aprovada/publicada; negociação concluída | S8 |
| REQ-071 | D01 | 10 | Notificação | RN-DCE-020 — Canais Fase 1 | Webchat + e-mail | S8 |
| REQ-072 | D01 | 10 | Notificação | RN-DCE-020 — WhatsApp Fase 2 (planejado) | Notificações unidirecionais; opt-out via "PARAR" ou configurações de perfil (imediato, sem confirmação, LGPD) | S8 |
| REQ-073 | D01 | 11 | Regra de negócio | RN-DCE-021 — Suporte operacional (tópicos cobertos) | KYC, dossiê, cenários A/B/C/D, Escrow, ZapSign, marketplace, propostas, encerramento | S8 |
| REQ-074 | D01 | 11 | Regra de negócio | Prazos operacionais conhecidos | Escrow: 10 dias úteis; extensão: +5 dias úteis; reversão: 15 dias corridos; ZapSign: 5 dias úteis; KYC automatizado ≤30min; KYC manual ≤2 dias úteis; dossiê ≤2 dias úteis | CROSS |
| REQ-075 | D01 | 12 | Regra de negócio | RN-DCE-023 — Fallback (API LLM falha) | Exibe: "A Dani está temporariamente indisponível. Tente novamente em instantes." | S9 |
| REQ-076 | D01 | 12 | Regra de negócio | RN-DCE-023 — Threshold 10% erros/15min | Alerta automático ao Admin; Dani continua | S9 |
| REQ-077 | D01 | 12 | Regra de negócio | RN-DCE-023 — Threshold 30% erros/15min | Desligamento automático da Dani-Cedente; reativação manual pelo Admin | S9 |
| REQ-078 | D01 | 12 | Regra de negócio | RN-DCE-023 — Takeover do Admin | Intervém quando confiança < 80%; Cedente não percebe interrupção | S9 |
| REQ-079 | D01 | 13 | SLA | SLAs de tempo de resposta | Consulta status, análise proposta, simulação retorno, dúvida operacional: ≤5 segundos cada | S3 |
| REQ-080 | D01 | 13 | Regra de negócio | Persistência do histórico | 90 dias (parâmetro configurável pelo Admin) | S3 |
| REQ-081 | D01 | 13 | Regra de negócio | Rate limit | 30 mensagens por hora por Cedente (janela deslizante); campo de entrada desabilitado com contador regressivo | S2 |
| REQ-082 | D01 | 13 | Regra de negócio | RN-DCE-022 — Comportamento em latência acima do SLA | >SLA: indicador "digitando" (3 pontos pulsando); >2×SLA (>10s): mensagem + botões "Aguardar"/"Tentar novamente"; >5min latência persistente → alerta Admin | S3 |
| REQ-083 | D01 | 13 | Regra de negócio | RN-DCE-024 — CSAT do Cedente | Escala 1–5; encerramento por inatividade 5min; CSAT médio < 3.5/5 em 24h → alerta Admin | S8 |
| REQ-084 | D01 | 14 | Permissão | Matriz de permissões — Cedente | Cadastrar própria oportunidade; escolher cenário; ver cenário próprio; publicar/retirar própria (com confirmação); ver propostas (sem identidade Cessionário); aceitar/recusar própria (ação direta); enviar contraproposta; acompanhar Escrow e dossiê próprios; assinar contratos; receber notificações | S2 |
| REQ-085 | D02 | 2.1 | Stack | Node.js — versão | 22.x+ (LTS) | S1 |
| REQ-086 | D02 | 2.1 | Stack | NestJS — versão | 10.x+ | S1 |
| REQ-087 | D02 | 2.1 | Stack | TypeScript — versão e config | 5.4+; `strict: true` obrigatório; arquivos .js/.jsx proibidos em novos módulos | S1 |
| REQ-088 | D02 | 2.1 | Stack | Prisma — versão | 6.x+; único ORM aprovado; TypeORM/Sequelize/Drizzle proibidos | S1 |
| REQ-089 | D02 | 2.1 | Stack | Pino — versão | 9.x+; `console.log` proibido em produção | S1 |
| REQ-090 | D02 | 2.1 | Stack | Helmet — versão | 8.x+; middleware adicionado desde o início | S1 |
| REQ-091 | D02 | 2.1 | Stack | class-validator + class-transformer | latest; toda entrada validada via pipes | S1 |
| REQ-092 | D02 | 2.1 | Stack | @nestjs/swagger | latest; todo endpoint documentado | S1 |
| REQ-093 | D02 | 2.1 | Stack | @nestjs/throttler | latest; rate limiting em todos os endpoints públicos | S1 |
| REQ-094 | D02 | 2.2 | Stack | PostgreSQL via Supabase — versão | 17+; único banco aprovado; MySQL/MariaDB/SQLite/MongoDB proibidos | S1 |
| REQ-095 | D02 | 2.2 | Stack | Supabase Auth | latest; complementa JWT da plataforma principal | S2 |
| REQ-096 | D02 | 2.2 | Stack | Supabase pgvector | latest; queries via `$queryRaw` com parâmetros preparados | S3 |
| REQ-097 | D02 | 2.2 | Stack | Supabase Realtime | Condicional; para notificações proativas (RN-DCE-020) | S8 |
| REQ-098 | D02 | 2.2 | Stack | Convenções banco | UUID v4 como PK; `created_at`/`updated_at` obrigatórios; soft delete padrão; `snake_case`; `@db.Timestamptz` obrigatório | S1 |
| REQ-099 | D02 | 2.3 | Stack | Redis — versão | 7.4+; todo uso com TTL e estratégia de invalidação; mudança de prompt invalida cache | S1 |
| REQ-100 | D02 | 2.3 | Stack | RabbitMQ — versão | 4.x+; toda fila com retry e dead-letter queue | S1 |
| REQ-101 | D02 | 3.1 | Stack | GPT-4 via OpenAI SDK | versão fixada em produção (ex: gpt-4-turbo-2024-04-09); alias genérico apenas em dev | S3 |
| REQ-102 | D02 | 3.1 | Stack | GPT-4o-mini — condicional | Temperature 0 obrigatório; classificação de intenção e extração com schema fixo | S3 |
| REQ-103 | D02 | 3.1 | Stack | Retry LLM com backoff exponencial | 3 tentativas, base 1s | S9 |
| REQ-104 | D02 | 3.1 | Stack | Feature flag PostHog como kill switch | desliga features de IA em produção instantaneamente | S9 |
| REQ-105 | D02 | 3.2 | Stack | Vercel AI SDK — versão | 4.x+; streaming SSE; SLA resposta inicial ≤5s | S3 |
| REQ-106 | D02 | 3.2 | Stack | LangChain.js — versão | 0.2.x+; RAG com pgvector; tool calling para consultas de status e cálculo | S3 |
| REQ-107 | D02 | 3.2 | Stack | LangGraph.js — versão | latest stable; estados: idle, analyzing_proposal, escrow_monitoring, dossier_guidance, takeover | S3 |
| REQ-108 | D02 | 3.3 | Stack | OpenAI Embeddings | text-embedding-3-small; re-embedding obrigatório ao alterar base de conhecimento | S3 |
| REQ-109 | D02 | 3.4 | Stack | Langfuse — versão | 3.x+; toda chamada LLM com trace; dataset de evals mínimo 50 exemplos por fluxo | S9 |
| REQ-110 | D02 | 3.5 | Segurança | Isolamento de dados por Cedente | Middleware valida cedente_id antes de qualquer chamada ao LLM | S2 |
| REQ-111 | D02 | 3.5 | Segurança | PII Masking obrigatório | cessionario_nome → "[Comprador]"; cessionario_cpf → "[REDACTED]"; nunca entra no LLM | S2 |
| REQ-112 | D02 | 3.5 | Segurança | Prompt Injection Protection | Sanitização de inputs; system prompt separado e imutável; instruções em tag delimitada | S2 |
| REQ-113 | D02 | 3.5 | Segurança | Rate Limiting por Cedente | 30 mensagens/hora (janela deslizante via Redis); campo desabilitado com contador regressivo | S2 |
| REQ-114 | D02 | 3.5 | Segurança | Content Filtering | OpenAI Moderation API para inputs do Cedente; respostas validadas contra RN-DCE-002 | S3 |
| REQ-115 | D02 | 3.5 | Segurança | Structured Outputs | Toda resposta determinística usa schema JSON fixo (cálculo retorno, status dossiê, status Escrow) | S3 |
| REQ-116 | D02 | 4.1 | API | REST JSON | Content-Type: application/json | S1 |
| REQ-117 | D02 | 4.1 | API | SSE — Server-Sent Events | Exclusivo para `/chat/stream`; WebSocket proibido sem ADR | S3 |
| REQ-118 | D02 | 4.2 | Autenticação | JWT Bearer Token | Access token 15min; refresh 7 dias; `JwtAuthGuard` em todos os endpoints protegidos | S2 |
| REQ-119 | D02 | 4.3 | API | Versionamento | Prefixo `/api/v1/` obrigatório; breaking changes → `/api/v2/`; versão anterior mantida ≥90 dias após deprecação | S1 |
| REQ-120 | D02 | 5.1 | Testes | Cobertura mínima geral | 80% de linhas; módulos críticos: 90% | S9 |
| REQ-121 | D02 | 5.1 | Testes | Evals de IA via Langfuse Evals | Mínimo 50 exemplos por fluxo: análise proposta, orientação dossiê, simulação retorno | S9 |
| REQ-122 | D02 | 5.1 | Testes | Testes de isolamento obrigatórios | Cedente A nunca acessa dados do Cedente B; antes de qualquer deploy em produção | S9 |
| REQ-123 | D02 | 5.1 | Testes | Mocks do LLM obrigatórios | Nunca chamar API real OpenAI em testes automatizados | S9 |
| REQ-124 | D02 | 6.1 | Repositório | Monorepo Turborepo + pnpm | Módulo em `apps/ai-dani-cedente/`; types em `packages/types/` | S1 |
| REQ-125 | D02 | 8.1 | Segurança | HTTPS obrigatório em todos os ambientes | — | S1 |
| REQ-126 | D02 | 8.1 | Segurança | CORS configurado | Apenas origens da plataforma Repasse Seguro | S1 |
| REQ-127 | D02 | 8.1 | Segurança | Supabase RLS ativo | Em todas as tabelas de dados do Cedente | S1 |
| REQ-128 | D02 | 8.3 | Compliance | Histórico de conversas LGPD | 90 dias; CSAT anonimizado antes de analytics | S3 |
| REQ-129 | D02 | 9 | Analytics | PostHog eventos obrigatórios | 9 eventos mínimos: `dani_cedente_chat_opened`, `dani_cedente_message_sent`, `dani_cedente_proposal_analysis_viewed`, `dani_cedente_simulation_requested`, `dani_cedente_dossier_status_viewed`, `dani_cedente_escrow_status_viewed`, `dani_cedente_csat_submitted`, `dani_cedente_rate_limit_reached`, `dani_cedente_takeover_triggered` | S9 |
| REQ-130 | D12 | 2 | Banco | Tabela `cedente_profiles` | id(PK UUID), supabase_auth_id(FK), full_name, cpf_masked, kyc_status(enum KycStatus), kyc_approved_at, created_at, updated_at, deleted_at | S1 |
| REQ-131 | D12 | 2 | Banco | Tabela `opportunities` | id(PK), cedente_id(FK), code, empreendimento_name, incorporadora, tipologia, tabela_contrato(Decimal), tabela_atual(Decimal), valor_pago_cedente(Decimal), delta_calculated(Decimal), scenario_chosen, status(enum OpportunityStatus), published_at, closed_at, created_at, updated_at, deleted_at | S1 |
| REQ-132 | D12 | 2 | Banco | Tabela `opportunity_scenarios` | id(PK), opportunity_id(FK), scenario_type(enum ScenarioType A/B/C/D), repasse_value(Decimal), net_return_estimated(Decimal), payment_conditions, created_at, updated_at | S1 |
| REQ-133 | D12 | 2 | Banco | Tabela `proposals` | id(PK), opportunity_id(FK), proposal_value(Decimal), status(enum ProposalStatus), expires_at, responded_at, created_at, updated_at, deleted_at | S1 |
| REQ-134 | D12 | 2 | Banco | Tabela `dossier_documents` | id(PK), opportunity_id(FK), document_type, status(enum DossierDocumentStatus), rejection_reason, storage_path, expires_at, approved_at, created_at, updated_at, deleted_at | S1 |
| REQ-135 | D12 | 2 | Banco | Tabela `escrow_transactions` | id(PK), proposal_id(FK), amount(Decimal), status(enum EscrowStatus), deposit_deadline, deposited_at, released_at, reversed_at, extension_requested(bool), extension_approved(bool), extension_approved_at, created_at, updated_at | S1 |
| REQ-136 | D12 | 2 | Banco | Tabela `chat_sessions` | id(PK), cedente_id(FK), opportunity_id(FK nullable), proposal_id(FK nullable), entry_point, status(enum ChatSessionStatus), message_count, csat_score, last_message_at, created_at, updated_at, deleted_at | S1 |
| REQ-137 | D12 | 2 | Banco | Tabela `chat_messages` | id(PK), session_id(FK), role(enum MessageRole), content(text), langfuse_trace_id, confidence_score(float), admin_takeover(bool), created_at (append-only — sem updated_at/deleted_at) | S1 |
| REQ-138 | D12 | 2 | Banco | Tabela `knowledge_embeddings` | id(PK), title, content(text), embedding(vector 1536), category, created_at, updated_at (sem cedente_id — conhecimento compartilhado) | S1 |
| REQ-139 | D13 | 4 | Enum | Enum `OpportunityStatus` | RASCUNHO, PUBLICADA, EM_NEGOCIACAO, PAUSADA, EXPIRADA, CANCELADA | S1 |
| REQ-140 | D13 | 4 | Enum | Enum `ScenarioType` | A, B, C, D | S1 |
| REQ-141 | D13 | 4 | Enum | Enum `ProposalStatus` | RECEBIDA, EM_ANALISE, ACEITA, RECUSADA, CONTRAPROPOSTA, CANCELADA, EXPIRADA | S1 |
| REQ-142 | D13 | 4 | Enum | Enum `ProposalType` | INICIAL, CONTRAPROPOSTA | S1 |
| REQ-143 | D13 | 4 | Enum | Enum `DossierDocumentType` | RG_CNH, CPF, COMPROVANTE_RESIDENCIA, CONTRATO_FINANCIAMENTO, EXTRATO_FINANCIAMENTO, DECLARACAO_QUITACAO | S1 |
| REQ-144 | D13 | 4 | Enum | Enum `DossierDocumentStatus` | AGUARDANDO_ENVIO, EM_ANALISE, APROVADO, REJEITADO | S1 |
| REQ-145 | D13 | 4 | Enum | Enum `EscrowStatus` | AGUARDANDO_DEPOSITO, DEPOSITO_CONFIRMADO, LIBERADO, REVERTIDO, EXPIRADO | S1 |
| REQ-146 | D13 | 4 | Enum | Enum `MessageRole` | USER, ASSISTANT, SYSTEM | S1 |
| REQ-147 | D13 | 4 | Enum | Enum `ChatSessionStatus` | ATIVA, ENCERRADA | S1 |
| REQ-148 | D13 | 4 | Enum | Enum `KycStatus` | PENDENTE, APROVADO, REPROVADO | S1 |
| REQ-149 | D13 | 5 | Banco | `Opportunity.version` para concorrência otimista | Campo `version Int @default(1)` | S1 |
| REQ-150 | D13 | 2 | Banco | Estrutura de arquivos Prisma | schema.prisma, seed.ts, rls/policies.sql, rls/indexes.sql, middleware/cedente-isolation.middleware.ts, middleware/soft-delete.middleware.ts | S1 |
| REQ-151 | D13 | 2 | Banco | Ordem de aplicação em produção | 1. prisma migrate deploy; 2. prisma db execute --file rls/policies.sql; 3. prisma db execute --file rls/indexes.sql; 4. prisma db seed (dev/staging only) | S1 |
| REQ-152 | D14 | 3 | Arquitetura | 12 módulos NestJS | auth, chat, agent, rag, opportunity, proposal, dossier, escrow, notification, simulation, fallback, admin | S1 |
| REQ-153 | D14 | 4 | Arquitetura | Filas RabbitMQ — 3 queues | rag.ingest, notification.send, escrow.events — todas com DLQ e retry exponencial | S1 |
| REQ-154 | D14 | 4 | Arquitetura | Cache Redis — 5 chaves com TTL | sessão agente: `dani:agent_state:{sessionId}` TTL 1800s; rate limit: `rate_limit:{cedente_id}`; cache LLM; estado agente; contexto oportunidade | S1 |
| REQ-155 | D15 | 2 | Arquitetura de pastas | Estrutura src/modules, src/common, src/infrastructure, src/config | Padrão Controller → Service → Repository → DTO → Entity por módulo | S1 |
| REQ-156 | D16 | 7.1 | Endpoint | `GET /auth/me` | Retorna perfil do Cedente autenticado; errors: 401 DCE-AUTH-4010_001/003, 404 DCE-AUTH-4040_001 | S2 |
| REQ-157 | D16 | 7.1 | Endpoint | `POST /auth/validate` | Valida token JWT; errors: 401 DCE-AUTH-4010_002 | S2 |
| REQ-158 | D16 | 7.2 | Endpoint | `POST /chat/sessions` | Cria nova sessão de chat | S3 |
| REQ-159 | D16 | 7.2 | Endpoint | `GET /chat/sessions` | Lista sessões do Cedente (paginado) | S3 |
| REQ-160 | D16 | 7.2 | Endpoint | `GET /chat/sessions/:session_id` | Retorna sessão por ID | S3 |
| REQ-161 | D16 | 7.2 | Endpoint | `PATCH /chat/sessions/:session_id/close` | Encerra sessão de chat | S3 |
| REQ-162 | D16 | — | Endpoint | 38 endpoints totais em 8 domínios | Auth(2), Chat(N), Opportunity(N), Proposal(N), Dossier(N), Escrow(N), Simulation(N), Admin(N) — ver D16 seção 7 | S3–S9 |
| REQ-163 | D16 | — | API | Padrão de erro DCE | `{ "error": { "code": "DCE-MODULE-HTTPSTATUS_SEQ", "message": "...", "details": {} } }` | S1 |
| REQ-164 | D16 | 2.3 | API | Access token TTL | 3600s (1 hora) via Supabase Auth | S2 |
| REQ-165 | D16 | 2.3 | API | Refresh token TTL | 604800s (7 dias) via Supabase Auth | S2 |
| REQ-166 | D16 | 5 | API | Paginação | page, per_page(max 100), sort(default created_at), order(asc/desc); objeto meta: page, per_page, total, total_pages | S1 |
| REQ-167 | D16 | 6 | API | Rate limiting — chat | 30 mensagens/hora por cedente_id; outros endpoints 100 req/min por IP; Admin endpoints 200 req/min por IP | S2 |
| REQ-168 | D17 | 2.1 | Integração | OpenAI API — P0 crítico | API key via `OPENAI_API_KEY`; retry backoff 3x base 1s; kill switch PostHog | S3 |
| REQ-169 | D17 | 2.2 | Integração | Supabase Auth — P0 crítico | JWT RS256 validation; `SUPABASE_JWT_SECRET` | S2 |
| REQ-170 | D17 | 2.3 | Integração | Supabase Storage — P1 alto | upload de documentos do dossiê; `SUPABASE_STORAGE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | S5 |
| REQ-171 | D17 | 2.4 | Integração | ZapSign — P1 alto | Assinatura digital; webhook de eventos; HMAC validation obrigatória | S7 |
| REQ-172 | D17 | 2.5 | Integração | Langfuse — P1 alto | `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`; obrigatório antes do deploy | S9 |
| REQ-173 | D17 | 2.6 | Integração | PostHog — P2 médio | `POSTHOG_API_KEY`; kill switch `dani_cedente_enabled` | S9 |
| REQ-174 | D18 | 3 | Auth | JwtAuthGuard — fluxo de validação | Extrai Bearer token → verifica RS256 local → extrai cedente_id → injeta no context; sem request HTTP por validação | S2 |
| REQ-175 | D18 | 4 | Auth | CedenteIsolationMiddleware | Injeta cedente_id em todo request; PII masking antes de logs/LLM | S2 |
| REQ-176 | D18 | — | Auth | RbacGuard — 2 roles | cedente e admin; guards verificam role no JWT | S2 |
| REQ-177 | D19 | 2.2 | Agente IA | LangGraph estados do agente | idle → analyzing_proposal, dossier_guidance, escrow_monitoring, takeover; transições completas documentadas | S3 |
| REQ-178 | D19 | 3.1 | Agente IA | 5 tools LangChain documentadas | get-opportunity, get-proposal, get-dossier, get-escrow, simulate-return — timeout 5s, retries 2, fallback definido | S3 |
| REQ-179 | D19 | 4.1 | Agente IA | Parâmetros GPT-4 | model: gpt-4-turbo, temperature: 0.2, max_tokens: 1024, stream: true, tool_choice: auto | S3 |
| REQ-180 | D19 | 5.1 | Agente IA | Memória de sessão Redis | TTL 1800s por session_id; últimas 10 mensagens; sem memória de longo prazo na Fase 1 | S3 |
| REQ-181 | D19 | 6 | Agente IA | Pipeline RAG | text-embedding-3-small; 1536 dims; IVFFlat cosine k=5; chunks ~800 tokens overlap 100; RecursiveCharacterTextSplitter | S3 |
| REQ-182 | D19 | 6.1 | Agente IA | Categorias da base de conhecimento | PLATFORM_RULES (~50 chunks), FAQ_CEDENTE (~30), ESCROW_PROCESS (~20), KYC_PROCESS (~15), DOSSIER_RULES (~15), SCENARIOS_GUIDE (~10) | S3 |
| REQ-183 | D20 | 1 | Error Handling | 12 categorias de erro | Validação(400), Auth(401), Autorização(403), Not Found(404), Conflito(409), Regra negócio(422), Rate Limit(429), Interno(500), Serviço externo(502/503), Agente indisponível(503), Webhook inválido(401), Erro SSE | S1 |
| REQ-184 | D20 | 2.1 | Error Handling | Schema de erro com correlation_id | `{ error: { code, category, message, user_message, correlation_id, timestamp, details } }` | S1 |
| REQ-185 | D21 | 1 | Notificação | 12 templates de notificação | Cobre 10 eventos RN-DCE-020 + régua ZapSign + extensão Escrow | S8 |
| REQ-186 | D21 | 1.1 | Notificação | Arquitetura event-driven assíncrona | NotificationService.publishNotification() → RabbitMQ exchange `notifications.topic` → filas canal; nunca síncrono no request principal | S8 |
| REQ-187 | D21 | 2.1 | Notificação | Webchat SSE — payload e expiração | Notificações não lidas expiram em 7 dias; fallback offline: persiste como PENDING no banco; deep link obrigatório (action_url) | S8 |

---

## Cobertura por Doc

| Doc | Qtd REQs extraídos | Atribuídos |
|---|---|---|
| D01 | 84 | 84 |
| D02 | 48 | 48 |
| D03 | (tokens de design — coberto via D06/D09 cross) | — |
| D05.1–05.5 | (PRDs cobertos via D01 cross-referência) | — |
| D06 | (7 telas cobertas via D09 cross) | — |
| D07 | (wireframes — via D09) | — |
| D08 | (UX Writing — via D03) | — |
| D09 | (UI contracts — via D06) | — |
| D10 | (glossário cross-cutting) | CROSS |
| D11 | (Mobile Fase 2 — out of scope Fase 1) | — |
| D12 | 10 | 10 |
| D13 | 13 | 13 |
| D14 | 4 | 4 |
| D15 | 1 | 1 |
| D16 | 12 | 12 |
| D17 | 6 | 6 |
| D18 | 3 | 3 |
| D19 | 6 | 6 |
| D20 | 2 | 2 |
| D21 | 3 | 3 |
| D22–D29 | (setup, CI/CD, qualidade — cobertos via sprints S9/S10) | S9/S10 |

---

## Verificação de Cobertura

```
Registro Mestre: 187 requisitos
Atribuídos a sprint: 187 (100%)
Sem sprint: 0 (meta: 0)
```

> **Sprints condicionais:** Mobile (D11) está fora do escopo da Fase 1 — o D11 especifica chat como widget web responsivo na Fase 1, com app React Native nativo planejado para Fase 2. Sprint Mobile NÃO é gerada.
> Agentes de IA (D19): sim, o produto utiliza agentes de IA — Sprint de Agentes de IA está incluída como S3 (Agente Core).
