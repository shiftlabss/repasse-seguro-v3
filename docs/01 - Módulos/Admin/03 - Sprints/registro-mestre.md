# Registro Mestre de Requisitos — Módulo Admin

## Repasse Seguro

| Campo | Valor |
|---|---|
| **Gerado em** | 2026-03-24 |
| **Gerado por** | Pipeline Fase 3 v2.3 |
| **Total de requisitos** | 312 |
| **Docs fonte** | D01.1–D01.5, D02, D06, D10, D12, D13, D14, D15, D16, D17, D18, D19, D20, D21, D22, D23, D24, D25, D26, D27, D28, D29 |

---

> Este arquivo é a FONTE DA VERDADE para cobertura de requisitos. Cada REQ deve ter ≥1 item de checklist em sua sprint designada.

---

## Tabela de Requisitos

| ID | Doc Fonte | Seção | Tipo | Descrição | Valor/Detalhe | Sprint |
|----|-----------|-------|------|-----------|---------------|--------|
| REQ-001 | D01.1 | §1 | Glossário | Cedente — definição canônica | Pessoa que tem contrato e quer transferir; nunca usar "vendedor" no código | CROSS |
| REQ-002 | D01.1 | §1 | Glossário | Cessionário — definição canônica | Pessoa que adquire o direito; sinônimo aceito: Comprador, Investidor | CROSS |
| REQ-003 | D01.1 | §1 | Glossário | Caso — definição canônica | Entidade central de trabalho; 1 imóvel = 1 caso ativo por Cedente | CROSS |
| REQ-004 | D01.1 | §1 | Glossário | Dossiê — nunca apagado | Conjunto de documentos; preservado mesmo após conclusão ou cancelamento | CROSS |
| REQ-005 | D01.1 | §1 | Glossário | Cenários de Retorno A, B, C, D | A=só saldo devedor; B=100% pago; C=100%+30% valoriz.; D=100%+50% valoriz. | S4 |
| REQ-006 | D01.1 | §1 | Glossário | Valor Distrato Referência | 50% do valor pago pelo Cedente (percentual configurável pelo Master) | S6 |
| REQ-007 | D01.1 | §1 | Glossário | Conta Escrow — definição | Conta garantia operada por parceiro financeiro regulado (Celcoin) | S6 |
| REQ-008 | D01.1 | §1 | Glossário | Fechamento — 4 critérios simultâneos | Instrumento assinado + preço confirmado + anuência + depósito escrow | S7 |
| REQ-009 | D01.1 | §1 | Glossário | Escalonamento — sempre descendente | D→C→B→A; subir de cenário exige cancelar e recadastrar | S5 |
| REQ-010 | D01.1 | §1 | Glossário | Reversão — 15 dias corridos após Fechamento | Conta Escrow estorna integralmente; RS não recebe comissão | S7 |
| REQ-011 | D01.1 | §1 | Glossário | ZapSign — integração de assinatura eletrônica | Usado para: Instrumento de Cessão, Termo Comercial, Termo de Aceite de Escalonamento | S7 |
| REQ-012 | D01.1 | §1 | Glossário | Guardião do Retorno — agente IA | Valida e protege cenários de retorno do Cedente | S9 |
| REQ-013 | D01.1 | §1 | Glossário | Analista de Oportunidades — agente IA | Identifica e qualifica oportunidades de match | S9 |
| REQ-014 | D01.1 | §2 | Módulo | Sidebar com 10 módulos | Dashboard, Pipeline, Triagem, Negociação, Formalização, Financeiro, Supervisão IA, Usuários, Relatórios, Configurações | S1 |
| REQ-015 | D01.1 | §2 | RBAC | Módulo Dashboard — visibilidade por perfil | Todos (conteúdo filtrado por perfil) | S3 |
| REQ-016 | D01.1 | §2 | RBAC | Módulo Pipeline — visibilidade por perfil | Analista (seus casos), Coordenador/Master (todos), Gestor Financeiro (leitura) | S4 |
| REQ-017 | D01.1 | §2 | RBAC | Módulo Triagem — acesso | Analista, Coordenador, Master | S4b |
| REQ-018 | D01.1 | §2 | RBAC | Módulo Negociação — acesso | Analista, Coordenador, Master | S5 |
| REQ-019 | D01.1 | §2 | RBAC | Módulo Formalização — acesso | Analista, Coordenador, Master | S7 |
| REQ-020 | D01.1 | §2 | RBAC | Módulo Financeiro — acesso | Coordenador (leitura), Gestor Financeiro, Master | S6 |
| REQ-021 | D01.1 | §2 | RBAC | Módulo Supervisão IA — acesso | Coordenador, Master | S9 |
| REQ-022 | D01.1 | §2 | RBAC | Módulo Usuários — acesso | Coordenador (leitura+edição básica), Master (tudo) | S8 |
| REQ-023 | D01.1 | §2 | RBAC | Módulo Relatórios — acesso | Coordenador (sem Receita), Gestor Financeiro (Receita+Auditoria), Master (tudo) | S10 |
| REQ-024 | D01.1 | §2 | RBAC | Módulo Configurações — acesso | Apenas Master | S11 |
| REQ-025 | D01.1 | §3 | Perfil | Analista — função e atribuição | Executa operação diária; atribuído pelo Master; só casos designados | S2 |
| REQ-026 | D01.1 | §3 | Perfil | Coordenador — função e atribuição | Supervisiona, atribui casos, aprova escalonamentos, cancela, resolve escalações | S2 |
| REQ-027 | D01.1 | §3 | Perfil | Gestor Financeiro — função e atribuição | Monitora Escrow, processa estornos, inicia reversões, exporta relatórios financeiros | S2 |
| REQ-028 | D01.1 | §3 | Perfil | Master — função e atribuição | Acesso irrestrito; cria operadores; configura parâmetros; deve haver ≥1 Master ativo | S2 |
| REQ-029 | D01.1 | §3 | RBAC | Matriz completa de permissões (10 módulos × 4 perfis) | Ver tabela §3.2 em D01.1 — implementar exatamente | S2 |
| REQ-030 | D01.1 | §4 | Entidade | Caso — criado pelo Cedente; editável por Analista/Coordenador; terminal em Concluído/Cancelado | — | S4 |
| REQ-031 | D01.1 | §4 | Entidade | Dossiê — criado automaticamente ao criar o caso; nunca apagado | — | S4b |
| REQ-032 | D01.1 | §4 | Entidade | Proposta — criada pelo Cessionário; editável antes do aceite; encerrada em várias condições | — | S5 |
| REQ-033 | D01.1 | §4 | Entidade | Termo Comercial — gerado pelo Analista; imutável após assinatura | — | S7 |
| REQ-034 | D01.1 | §4 | Entidade | Conta Escrow do Caso — criada automaticamente ao entrar em Formalização; não editável | — | S6 |
| REQ-035 | D01.1 | §4 | Entidade | Fatura de Comissão — criada no Fechamento; não editável; paga via escrow | — | S6 |
| REQ-036 | D01.1 | §4 | Entidade | Envelope ZapSign — criado pelo Analista; não editável após envio; encerrado após assinaturas | — | S7 |
| REQ-037 | D01.1 | §4 | Entidade | Alerta de SLA — criado automaticamente; não editável; encerrado quando ação resolvida | — | S4 |
| REQ-038 | D01.1 | §5 | Máquina de estado | Caso: Captado → Em Triagem | Analista inicia triagem | S4 |
| REQ-039 | D01.1 | §5 | Máquina de estado | Caso: Em Triagem → Qualificado | Dossiê completo + adimplência ok | S4b |
| REQ-040 | D01.1 | §5 | Máquina de estado | Caso: Em Triagem → Bloqueado | Parcelas em atraso | S4b |
| REQ-041 | D01.1 | §5 | Máquina de estado | Caso: Bloqueado → Em Triagem | Cedente regulariza | S4b |
| REQ-042 | D01.1 | §5 | Máquina de estado | Caso: Qualificado → Oferta Ativa | Coordenador publica oferta | S4 |
| REQ-043 | D01.1 | §5 | Máquina de estado | Caso: Oferta Ativa → Em Negociação | Cessionário faz proposta | S5 |
| REQ-044 | D01.1 | §5 | Máquina de estado | Caso: Oferta Ativa → Oferta Ativa (escalonamento aceito) | Timer SLA reinicia | S5 |
| REQ-045 | D01.1 | §5 | Máquina de estado | Caso: Em Negociação → Oferta Ativa | Proposta expirada sem fila | S5 |
| REQ-046 | D01.1 | §5 | Máquina de estado | Caso: Em Negociação → Em Formalização | Aceite de negociação | S5 |
| REQ-047 | D01.1 | §5 | Máquina de estado | Caso: Em Formalização → Fechamento | 4 critérios cumpridos simultaneamente | S7 |
| REQ-048 | D01.1 | §5 | Máquina de estado | Caso: Fechamento → Pós Fechamento | Sistema inicia contagem 15 dias | S7 |
| REQ-049 | D01.1 | §5 | Máquina de estado | Caso: Pós Fechamento → Concluído | 15 dias sem reversão | S7 |
| REQ-050 | D01.1 | §5 | Máquina de estado | Caso: Pós Fechamento → Em Reversão | Desistência formal consensual | S7 |
| REQ-051 | D01.1 | §5 | Máquina de estado | Caso: Em Reversão → Cancelado | Reversão confirmada | S7 |
| REQ-052 | D01.1 | §5 | Máquina de estado | Caso: Pós Fechamento → Em Mediação | Desistência unilateral | S7 |
| REQ-053 | D01.1 | §5 | Máquina de estado | Caso: Em Mediação → Cancelado | Acordo na mediação | S7 |
| REQ-054 | D01.1 | §5 | Máquina de estado | Caso: Em Mediação → Disputa Formal | Sem acordo em 10 dias úteis | S7 |
| REQ-055 | D01.1 | §5 | Máquina de estado | Caso: Captado/Bloqueado/EmTriagem/Qualificado/OfertaAtiva/EmNegociação/EmFormalização → Cancelado | Cancelamento possível em qualquer estado pré-Fechamento | S4 |
| REQ-056 | D01.1 | §6 | RN | RN-001: Adimplência obrigatória | Comprovantes das últimas 3 parcelas + declaração assinada; Em Triagem → Bloqueado se inadimplente | S4b |
| REQ-057 | D01.1 | §6 | RN | RN-001.a: Exceção de adimplência por confirmação da construtora | Requer aprovação do Coordenador; Bloqueado → Qualificado | S4b |
| REQ-058 | D01.1 | §6 | RN | RN-001 — mensagem bloqueio (ao Cedente) | "Seu caso foi bloqueado por pendência de parcelas..." | S4b |
| REQ-059 | D01.1 | §6 | RN | RN-001 — mensagem desbloqueio (ao Cedente) | "Seu comprovante foi validado. Avaliado em até 3 dias úteis." | S4b |
| REQ-060 | D01.1 | §6 | RN | RN-001 — edge case: bloqueado >60 dias → sugestão de cancelamento ao Coordenador | — | S4b |
| REQ-061 | D01.1 | §6 | RN | RN-002: Dossiê mínimo de 6 documentos | Contrato original, 3 comprovantes parcelas, declaração adimplência, RG/CNH, comprovante endereço ≤90 dias, Tabela do Contrato | S4b |
| REQ-062 | D01.1 | §6 | RN | RN-002 — sem aprovação condicional | Nenhum perfil avança sem os 6 documentos verificados | S4b |
| REQ-063 | D01.1 | §6 | RN | RN-003: Um caso ativo por imóvel por Cedente | Bloqueia novo cadastro se já existe caso ativo (exceto Concluído/Cancelado) | S4 |
| REQ-064 | D01.1 | §6 | RN | RN-003.a: Cadastro simultâneo por Cedentes diferentes | Permite; gera alerta automático ao Coordenador para verificação | S4 |
| REQ-065 | D01.1 | §7 | RN | RN-004: Perfis de acesso — menus sem permissão ocultados (não desabilitados) | DEC-002: ocultação total | S2 |
| REQ-066 | D01.1 | §7 | RN | RN-004 — mudança de perfil: sidebar atualizada sem logout | — | S2 |
| REQ-067 | D01.1 | §7 | RN | RN-004.a: Tentativa de acesso a área não permitida → tela de acesso negado | ícone cadeado + mensagem + botão "Voltar à Dashboard" | S2 |
| REQ-068 | D01.1 | §7 | RN | RN-005: 2FA obrigatório para operadores Admin | Todos: Analista, Coordenador, Gestor Financeiro, Master | S2 |
| REQ-069 | D01.1 | §7 | RN | RN-005 — bloqueio após 5 tentativas consecutivas | Admin: 30 minutos; Cedentes: 15 minutos; Cessionários: 15 minutos | S2 |
| REQ-070 | D01.1 | §7 | RN | RN-005 — expiração de sessão por inatividade | Operadores Admin: 8 horas; Cedentes: 24 horas; Cessionários: 30 minutos | S2 |
| REQ-071 | D01.1 | §7 | RN | RN-005.a: Requisitos de senha | Mínimo 8 chars, 1 maiúscula, 1 número | S2 |
| REQ-072 | D01.1 | §7 | RN | RN-005 — link reset de senha expira em 1 hora (DEC-003) | — | S2 |
| REQ-073 | D01.1 | §7 | RN | RN-006: Unicidade CPF/CNPJ por tipo de perfil | Bloqueia cadastro duplicado; bloqueia self-dealing (mesmo CPF em dois lados do mesmo caso) | S2 |
| REQ-074 | D01.1 | §7 | RN | RN-007: KYC do Cessionário antes da primeira proposta | RG/CNH + comprovante endereço ≤90 dias + selfie liveness; validação automática ≤5 min; revisão manual ≤24h úteis | S8 |
| REQ-075 | D01.1 | §7 | RN | RN-008: Validação de e-mail antes da primeira ação | Cedente: antes de cadastrar imóvel; Cessionário: antes de submeter proposta | S2 |
| REQ-076 | D01.1 | §7 | RN | RN-008 — link validação e-mail expira em 24 horas (DEC-005) | — | S2 |
| REQ-077 | D01.1 | §8 | RN | RN-009: Visibilidade da Dashboard por perfil | Tabela de 10 componentes × 4 perfis em D01.1 §8 implementada exatamente | S3 |
| REQ-078 | D01.1 | §8 | RN | RN-010: Dashboard como tela inicial obrigatória e somente leitura | Redirect pós-login; nenhuma ação operacional; não configurável | S3 |
| REQ-079 | D01.1 | §8 | RN | RN-011: Polling da Dashboard a cada 60 segundos | Cache TTL; skeleton placeholder ao carregar; timeout 10s por card | S3 |
| REQ-080 | D01.1 | §8 | RN | RN-011 — modo offline na Dashboard | Banner fixo "Você está offline"; dados exibidos = últimos válidos com timestamp | S3 |
| REQ-081 | D01.1 | §8 | RN | RN-012: Estado vazio da Dashboard (onboarding) | Card boas-vindas com links "Criar primeiro operador" e "Configurar parâmetros" | S3 |
| REQ-082 | D01.1 | §8 | RN | RN-139: Priorização de alertas na Dashboard | Ordem: (1) SLA estourado, (2) depósitos pendentes próximos, (3) alertas IA, (4) escalonamentos sugeridos | S3 |
| REQ-083 | D01.1 | §9 | RN | RN-013: Visibilidade e controle de concorrência no Pipeline | Analista: só seus casos; lock otimista na edição de status | S4 |
| REQ-084 | D01.2 | §1 | RN | RN-018: Cálculo comissão Cedente — Cenário A = R$0,00 | — | S6 |
| REQ-085 | D01.2 | §1 | RN | RN-018: Cálculo comissão Cedente — Cenários B/C/D = 20% × (Valor Recuperado − Distrato Ref.) | Distrato Ref. = 50% do valor pago | S6 |
| REQ-086 | D01.2 | §1 | RN | RN-018 — percentuais configuráveis (Distrato Ref.: configurável; comissão 20%: intervalo 5%–50%) | Alterações afetam apenas novos casos | S11 |
| REQ-087 | D01.2 | §1 | RN | RN-019: Cálculo comissão Cessionário = 20% × Δ (Tabela Atual − Tabela do Contrato) | Se Δ ≤ 0: comissão variável = R$0; exceção Cenário A com Δ=0 configurável | S6 |
| REQ-088 | D01.2 | §1 | RN | RN-019 — hierarquia de fontes Tabela Atual | (1) tabela construtora, (2) oferta pública, (3) avaliação formal com laudo | S6 |
| REQ-089 | D01.2 | §1 | RN | RN-019.a: Aprovação 4 olhos para Delta ≥ R$100.000 | Coordenador notificado; confirma ou contesta; limiar configurável | S6 |
| REQ-090 | D01.2 | §1 | Máquina de estado | Conta Escrow: Aberta → Depósito Confirmado → Em Distribuição → Distribuída | — | S6 |
| REQ-091 | D01.2 | §1 | Máquina de estado | Conta Escrow: Depósito Confirmado → Congelada → Em Distribuição ou Estornada | — | S7 |
| REQ-092 | D01.2 | §1 | RN | RN-020: Fluxo completo da Conta Escrow (5 etapas) | Criação → Depósito → Retenção pré-Fechamento → Retenção pós-Fechamento → Distribuição automática | S6 |
| REQ-093 | D01.2 | §1 | RN | RN-020 — distribuição automática após 15 dias corridos sem reversão | Para Cedente: Valor Recuperado − Comissão Cedente; para RS: Comissão Cedente + Comissão Cessionário; crédito residual → Cessionário | S6 |
| REQ-094 | D01.2 | §1 | RN | RN-020.a: Conta Escrow individual por caso (sem compartilhamento) | — | S6 |
| REQ-095 | D01.2 | §1 | RN | RN-020.b: Tolerância de depósito R$0,50 | Acima da tolerância positiva = crédito residual; abaixo = depósito parcial não confirmado | S6 |
| REQ-096 | D01.2 | §1 | RN | RN-020.c: Modalidade fracionada para Cessionários Qualificados (50%+50%) | Habilitada pelo Master; 50% ao entrar em Formalização; 50% restantes ≤3 dias úteis antes do prazo de Fechamento | S6 |
| REQ-097 | D01.2 | §1 | RN | RN-021: Distribuição automática com notificação prévia 24h ao Gestor Financeiro | Se Gestor bloqueia → aprovação do Master em ≤24h; se Master não aprova → distribuição automática | S6 |
| REQ-098 | D01.2 | §1 | RN | RN-022: Taxas de transferência TED/PIX/DOC = responsabilidade do Cessionário | Taxas de distribuição = absorvidas pelo parceiro ou descontadas da comissão do RS | S6 |
| REQ-099 | D01.2 | §1 | RN | RN-141: Estorno sempre integral — sem estorno parcial | Sistema bloqueia tentativa de estorno parcial | S7 |
| REQ-100 | D01.2 | §1 | RN | RN-142: Comissão do RS no Cenário A — distribuição obrigatória mesmo com comissão Cedente zerada | RS recebe comissão do Cessionário | S6 |
| REQ-101 | D01.2 | §2 | RN | RN-023: 4 critérios simultâneos para Fechamento | Instrumento assinado + preço confirmado por evidência + anuência construtora + depósito confirmado | S7 |
| REQ-102 | D01.2 | §2 | RN | RN-023 — registros obrigatórios no Fechamento | Data/hora + nome Analista + screenshot dos 4 critérios + cópias ZapSign + comprovante escrow | S7 |
| REQ-103 | D01.2 | §2 | RN | RN-023 — confirmação de Fechamento exige digitação "CONFIRMAR" (DEC-007) | Barreira de confirmação textual para ação irreversível | S7 |
| REQ-104 | D01.2 | §2 | RN | RN-022.a: Fluxo de obtenção de anuência da construtora | SLA de resposta: 15 dias corridos; alerta no 10º dia; escala no 15º | S7 |
| REQ-105 | D01.2 | §2 | RN | RN-022.a — estados da anuência | Não solicitada → Solicitada → Recebida e verificada | Em atraso | Dispensada | S7 |
| REQ-106 | D01.2 | §2 | RN | RN-022.a — exceção: contrato sem exigência de anuência | Analista marca + justificativa; Coordenador confirma em ≤2 dias úteis | S7 |
| REQ-107 | D01.2 | §2 | RN | RN-022.a — 4 olhos para Valor Recuperado > R$500.000 | Validação adicional do Coordenador obrigatória | S7 |
| REQ-108 | D01.2 | §2 | RN | RN-024: Fechamento irreversível | Única via de reversão = fluxo formal dentro de 15 dias | S7 |
| REQ-109 | D01.2 | §2 | RN | RN-025: Escalonamento de cenário — sequência descendente D→C→B→A | Timer 30 dias sem proposta → alerta de estagnação → simulação comparativa ao Cedente | S5 |
| REQ-110 | D01.2 | §2 | RN | RN-025 — aceite via ZapSign (Termo de Aceite de Escalonamento) | SLA de 30 dias reinicia após escalonamento aceito | S5 |
| REQ-111 | D01.2 | §2 | RN | RN-025.a: Conflito entre escalonamento e negociação ativa | Escalonamento em fila; processado automaticamente se negociação resultar em recusa/expiração | S5 |
| REQ-112 | D01.2 | §3 | RN | RN-026: Fluxo de propostas e contrapropostas com limite de 3 rodadas | Validação piso antes de chegar ao Analista; após 3 rodadas → escala obrigatória ao Coordenador | S5 |
| REQ-113 | D01.2 | §3 | RN | RN-027: Prazo de resposta por rodada | 3 dias úteis → lembrete; 5 dias úteis → encerramento automático com ação diferenciada por parte | S5 |
| REQ-114 | D01.2 | §3 | RN | RN-028: Proposta mínima por cenário | Sistema rejeita automaticamente abaixo do piso; sem teto; alerta informativo >30% acima da Tabela Atual | S5 |
| REQ-115 | D01.2 | §3 | RN | RN-029: Histórico imutável de propostas | Nenhum perfil pode editar/excluir após criação | S5 |
| REQ-116 | D01.2 | §3 | RN | RN-030: Propostas simultâneas — negociações paralelas permitidas | Ao aceitar uma proposta → demais encerradas automaticamente como "Superada" | S5 |
| REQ-117 | D01.2 | §3 | RN | RN-031: Rate limiting de propostas | Limite 3 propostas simultâneas; >10 propostas/24h → bloqueio 6 horas + alerta Coordenador | S5 |
| REQ-118 | D01.2 | §3 | RN | RN-144: Anonimato absoluto Cedente/Cessionário na Negociação | Dados de identidade nunca na mesma visualização; exportações não cruzam dados pessoais | S5 |
| REQ-119 | D01.2 | §3 | RN | RN-145: Escalação obrigatória após 3 rodadas | Sistema bloqueia 4ª rodada pelo Analista automaticamente | S5 |
| REQ-120 | D01.2 | §3 | RN | RN-146: Validação de proposta (piso, sem teto, alerta 30%) | Mensagens específicas ao Cessionário e ao Analista | S5 |
| REQ-121 | D01.3 | §1 | RN | RN-055: Ordem FIFO obrigatória na fila de triagem | Alerta informativo (não bloqueante) ao selecionar fora de ordem; registrado em log | S4b |
| REQ-122 | D01.3 | §1 | RN | RN-056: Carimbo imutável de verificação de documento | Nome Analista + data + hora; nenhum perfil pode alterar | S4b |
| REQ-123 | D01.3 | §1 | RN | RN-148: Reenvio de documento — substitui no checklist; histórico preservado | Status volta para Pendente ao receber novo arquivo | S4b |
| REQ-124 | D01.3 | §1 | RN | RN-057: Salvamento automático de progresso de triagem | Progresso parcial preservado ao navegar; indicador "Verificação em andamento — X de 6" | S4b |
| REQ-125 | D01.3 | §1 | RN | RN-058: Bloqueio por inadimplência + desbloqueio | Sugestão cancelamento após 60 dias sem ação; desbloqueio exige comprovante atualizado | S4b |
| REQ-126 | D01.3 | §2 | RN | RN-059: SLAs por etapa — tabela completa (7 transições) | Captado→EmTriagem: 24h; EmTriagem→Qualificado: 3/5 dias úteis; Qualificado→OfertaAtiva: 2/3 dias úteis; OfertaAtiva→EmNegociação: 15/30 dias corridos; EmNegociação→EmFormalização: 10/20 dias úteis; EmFormalização→Fechamento: 10/20 dias úteis; PósFechamento→Concluído: 15 dias corridos fixo | S4 |
| REQ-127 | D01.3 | §2 | RN | RN-059 — alerta de SLA: verde→amarelo (≤20% restante)→vermelho (estourado) | Amarelo: só painel; Vermelho: e-mail + painel | S4 |
| REQ-128 | D01.3 | §2 | RN | RN-060: Reinício de SLA após escalonamento aceito | Timer de 30 dias reinicia a partir da data do escalonamento | S5 |
| REQ-129 | D01.3 | §2 | RN | RN-061: SLA pausado durante indisponibilidade de integração | ZapSign/Celcoin offline → SLA da Formalização pausado automaticamente | S7 |
| REQ-130 | D01.3 | §3 | RN | RN-062: SLA de entrega de notificações | Painel: ≤30 segundos; E-mail: ≤5 minutos; ZapSign: SLA do fornecedor | S8a |
| REQ-131 | D01.3 | §3 | RN | RN-063: Tabela completa de 21 eventos de notificação | Canal E-mail + Painel para todos (exceto SLA próximo = só painel) | S8a |
| REQ-132 | D01.3 | §3 | RN | RN-064: Falhas de entrega → alerta ao Coordenador; badge "Falhou"; reenvio manual | Log com evento + destinatário + canal + data/hora + status de entrega | S8a |
| REQ-133 | D01.3 | §4 | RN | RN-065: Criação de operador — somente Master | Nome + e-mail + perfil; validação em tempo real; e-mail convite; link primeiro acesso | S8 |
| REQ-134 | D01.3 | §4 | RN | RN-066: Suspensão de usuário | Acesso bloqueado imediatamente; casos congelados (Cedente); propostas canceladas (Cessionário); justificativa ≥20 chars | S8 |
| REQ-135 | D01.3 | §4 | RN | RN-067: Inatividade automática após 180 dias corridos | Status → Inativo; reativação automática por login | S8 |
| REQ-136 | D01.3 | §4 | RN | RN-068: Alteração de perfil de operador — apenas Master | Não pode alterar próprio perfil se único Master ativo | S8 |
| REQ-137 | D01.4 | §1 | RN | RN-087: Monitoramento de Contas Escrow em tempo real | Filtros: status, faixa de valor, data prevista de distribuição; painel lateral com breakdown | S6 |
| REQ-138 | D01.4 | §1 | RN | RN-088: Bloqueio de distribuição com aprovação do Master | Gestor bloqueia → Master aprova em ≤24h; sem aprovação → distribuição retomada automaticamente | S6 |
| REQ-139 | D01.4 | §1 | RN | RN-088.a: Escalonamento de aprovação de bloqueio | 8h → reenvio urgente; 16h → todos Masters ativos; sem Master → e-mail emergência; incidente em log de auditoria | S6 |
| REQ-140 | D01.4 | §1 | RN | RN-089: Processamento de estorno | Modal confirmação; congela escrow; sempre total; comprovante gerado automaticamente | S7 |
| REQ-141 | D01.4 | §1 | RN | RN-090: Painel de inadimplência | Lista depósitos pendentes; alerta amarelo ≤3 dias úteis; alerta vermelho = estourado; prorrogação: +5 dias úteis (só Coordenador, 1x) | S6 |
| REQ-142 | D01.4 | §1 | RN | RN-091: Conciliação bancária | Diferença ≤R$0,50 = Conciliado; >R$0,50 = Divergência detectada; Gestor deve justificar antes de fechar | S6 |
| REQ-143 | D01.4 | §1 | RN | RN-092: Relatório financeiro mensal automático | Gerado no 1º dia útil de cada mês; enviado por e-mail ao Gestor Financeiro e Master; exportável PDF/CSV | S10 |
| REQ-144 | D01.4 | §2 | RN | RN-093: Monitoramento passivo contínuo de agentes | Log atualiza ≤10 segundos; indicadores: ≥80% verde, 60–79% amarelo, <60% vermelho | S9 |
| REQ-145 | D01.4 | §2 | RN | RN-094: Alertas de baixa confiança (padrão: limiar 80%) | Coordenador: Confirmar, Reverter ou Takeover; cada ação exige justificativa; auditoria imutável | S9 |
| REQ-146 | D01.4 | §2 | RN | RN-095: Takeover manual e automático | Manual: pausa agente para 1 caso; Automático: falha crítica; limite 5 takeovers simultâneos; agente não se reativa sozinho | S9 |
| REQ-147 | D01.4 | §2 | RN | RN-095.a: Thresholds Guardião do Retorno | ≥90%: autônomo; 70–89%: age+notifica (Analista confirma ≤4h); 50–69%: sugere+aguarda; <50%: não age | S9 |
| REQ-148 | D01.4 | §2 | RN | RN-095.a: Thresholds Analista de Oportunidades | ≥85%: envia sugestão; 60–84%: registra "Em análise" (Analista ≤24h); <60%: log sem notificação | S9 |
| REQ-149 | D01.4 | §2 | RN | RN-095.a: Guardião nunca age autonomamente em | Escalonamento de cenário, bloqueio por adimplência, alteração de valores financeiros | S9 |
| REQ-150 | D01.4 | §2 | RN | RN-095.a: Analista de Oportunidades nunca contata Cessionário diretamente | Aceite de proposta sempre do Analista ou Cedente humano | S9 |
| REQ-151 | D01.4 | §2 | RN | RN-095.a: Thresholds configuráveis pelo Master (50%–95%) | Alterações entram em vigor imediatamente; registradas em auditoria | S9 |
| REQ-152 | D01.4 | §2 | RN | RN-095.a: Alerta de degradação | Taxa de reversão >30% em 7 dias → notificação ao Master | S9 |
| REQ-153 | D01.4 | §2 | RN | RN-096: Falha crítica — 6 eventos gatilho | Loop (3x em <60s), timeout (30s padrão), erro sistema, dados corrompidos, conflito de versão, violação RN | S9 |
| REQ-154 | D01.4 | §2 | RN | RN-097: Revisão diária obrigatória | 24h sem acesso → alerta Master; 48h → escalação crítica; 72h → incidente de governança | S9 |
| REQ-155 | D01.4 | §2 | RN | RN-098: Escalação automática de alertas não resolvidos em 48h ao Master | Badge "Escalado automaticamente" | S9 |
| REQ-156 | D01.4 | §2 | RN | RN-099: Modo Supervisão Total | Todas ações em fila; 4h → alerta Coordenador; 8h → escala Master; 12h → descarta + flag "Ação pendente manual" | S9 |
| REQ-157 | D01.4 | §2 | RN | RN-100: Histórico de calibração de limiar | Imutável; valor anterior + novo + data + Master + justificativa; limiar padrão 80% | S9 |
| REQ-158 | D01.4 | §3 | RN | RN-101: 6 tipos de relatórios | SLA Operacional, Volume de Casos, Receita, Conversão, Trilha de Auditoria, Agentes de IA | S10 |
| REQ-159 | D01.4 | §3 | RBAC | Relatório Receita — apenas Gestor Financeiro e Master | — | S10 |
| REQ-160 | D01.4 | §3 | RBAC | Relatório Auditoria — apenas Gestor Financeiro e Master | — | S10 |
| REQ-161 | D01.4 | §3 | RN | RN-104: Exportação em PDF e CSV | Todos os relatórios exportáveis; export registrado na trilha de auditoria | S10 |
| REQ-162 | D01.4 | §3 | RN | RN-105: Trilha de Auditoria — filtros obrigatórios | Evitar dump completo; export CSV/JSON apenas para Master | S10 |
| REQ-163 | D01.4 | §4 | RN | RN-107: Hub de Configurações — 5 seções | Comissões e Parâmetros, Prazos e SLAs, Templates de Notificação, Integrações, Usuários e Permissões | S11 |
| REQ-164 | D01.4 | §4 | RN | RN-108: Configurações — acesso exclusivo Master | Qualquer alteração registrada em audit trail com nome e timestamp | S11 |
| REQ-165 | D01.4 | §4 | RN | RN-109: Configurações de Integrações (ZapSign e Celcoin) | API Key, webhook secret, configurações de template; modo de contingência | S11 |
| REQ-166 | D01.4 | §4 | RN | RN-111: Snapshot de configuração por caso (case_config_snapshot) | Congelado no momento da criação do caso; alterações futuras não afetam casos em andamento | S4 |
| REQ-167 | D01.5 | §2 | RN | RN-121: Integração ZapSign — envio de envelopes | POST /api/v1/documents; prazo padrão 30 dias; persiste zapsign_document_token; retenta em 5xx | S7 |
| REQ-168 | D01.5 | §2 | RN | RN-121 — RBAC de envelope ZapSign | Enviar/reenviar: Analista, Coordenador, Master; cancelar: Coordenador, Master; visualizar: todos | S7 |
| REQ-169 | D01.5 | §2 | RN | RN-122: Webhooks ZapSign — 4 eventos | document_signed, document_completed, document_refused, document_expired | S7 |
| REQ-170 | D01.5 | §2 | RN | RN-122 — validação HMAC-SHA256 do webhook | Descarta se assinatura inválida; processa idempotente | S7 |
| REQ-171 | D01.5 | §2 | RN | RN-123: Contingência ZapSign | Ativa após 2 timeouts consecutivos (>30s); SLA pausado (RN-061); registro manual aprovado pelo Coordenador | S7 |
| REQ-172 | D01.5 | §2 | RN | RN-123 — prazo máximo contingência: 48h antes de escalar ao Master | DA-009 | S7 |
| REQ-173 | D01.5 | §2 | RN | RN-124: Gestão de templates ZapSign | Versão + data ativação; histórico imutável; alterações prospectivas; PDF gerado persistido | S11 |
| REQ-174 | D01.5 | §3 | RN | RN-125: Integração Celcoin — abertura de Conta Escrow | Dados enviados: UUID caso, valor esperado, dados Cedente/Cessionário, finalidade | S7 |
| REQ-175 | D01.5 | §3 | RN | RN-126: Confirmação de depósito via webhook Celcoin | Tolerância R$0,50; depósito acima = crédito residual; depósito parcial = não confirmado | S6 |
| REQ-176 | D01.5 | §3 | RN | RN-127: Distribuição e estorno via Celcoin | 3 tentativas com intervalo 1h em caso de falha bancária (DA-011) | S6 |
| REQ-177 | D01.5 | §3 | RN | RN-128: Período de custódia — 15 dias após Fechamento | Sistema bloqueia distribuição automática durante custódia; só Master pode antecipar | S7 |
| REQ-178 | D01.5 | §4 | RN | RN-129: Audit trail imutável | Schema separado `audit`; append-only; estrutura obrigatória: 13 campos; 12 categorias de eventos; retenção 5 anos | S1 |
| REQ-179 | D01.5 | §4 | RN | RN-130: Lock otimista em edição concorrente | Campo `version` em toda entidade editável; HTTP 409 em mismatch; modal de conflito lado a lado | S1 |
| REQ-180 | D01.5 | §4 | RN | RN-131: Isolamento de dados por perfil | Matriz detalhada: CPF mascarado para Analista; dados bancários ocultos para Analista; mascaramento no backend (DA-013) | S2 |
| REQ-181 | D01.5 | §4 | RN | RN-132: Estratégia de revalidação por módulo | Dashboard: polling 60s; Pipeline: polling 15s; Negociação: polling 30s; Financeiro: polling 120s; Supervisão IA: polling 10s | S3 |
| REQ-182 | D01.5 | §4 | RN | RN-133: Proteção contra duplo submit | Botão desabilitado ao clicar + spinner; idempotency_key no backend; timeout 30s | S1 |
| REQ-183 | D02 | §2 | Stack | Backend: Node.js 22+ / NestJS 10+ / TypeScript strict / Prisma 6+ / PostgreSQL 17+ (Supabase) | Versões mínimas obrigatórias | S1 |
| REQ-184 | D02 | §2 | Stack | Backend: Redis 7.4+ (Upstash) / RabbitMQ 4+ (CloudAMQP) / Pino 9+ | TTL obrigatório em toda chave Redis | S1 |
| REQ-185 | D02 | §2 | Stack | Backend: Helmet 8+, @nestjs/throttler, @sentry/nestjs 9+ | Middleware de segurança desde o 1º commit | S1 |
| REQ-186 | D02 | §2 | Stack | Integrações externas: Celcoin (Escrow), Meta Cloud API (WhatsApp), Twilio (SMS) | — | S1 |
| REQ-187 | D02 | §3 | Stack | Frontend: React 19+ / Vite 7+ / TanStack Router 1+ / TanStack Query 5+ / Zustand 5+ / Tailwind CSS 4 / shadcn/ui / Framer Motion 12+ | Next.js proibido | S1 |
| REQ-188 | D02 | §5 | Stack | Mobile: React Native 0.76+ / Expo SDK 52+ / expo-router 4+ / Reanimated 3+ | — | S12 |
| REQ-189 | D02 | §6 | Stack | Testes: Vitest + Supertest + Playwright | Pirâmide 70/20/5/5 | S13 |
| REQ-190 | D02 | §4 | ADR | ADR-001: Supabase Realtime para Pipeline | Subscriptions em tabelas de casos e escrow | S1 |
| REQ-191 | D02 | §4 | ADR | ADR-002: Webhook HMAC-SHA256 para ZapSign e Celcoin | — | S1 |
| REQ-192 | D02 | §4 | ADR | ADR-003: Audit trail em schema separado append-only | — | S1 |
| REQ-193 | D06 | §1 | Tela | T-001: Login / Autenticação | Rota /login; todos os perfis; 5 estados; microinteração shake; responsivo; a11y | S2 |
| REQ-194 | D06 | §1 | Tela | T-002: 2FA — Verificação de código | Rota /login/2fa; OTP 6 inputs; timer 30s; countdown visual | S2 |
| REQ-195 | D06 | §1 | Tela | T-003: Recuperação de Senha | Rota /recuperar-senha; e-mail não confirmado nem negado | S2 |
| REQ-196 | D06 | §1 | Tela | T-004: Redefinição de Senha | Rota /redefinir-senha?token=XXX; validação de requisitos inline | S2 |
| REQ-197 | D06 | §2 | Tela | T-010: Dashboard | Rota /dashboard; todos os perfis (conteúdo filtrado); 10 componentes × 4 perfis | S3 |
| REQ-198 | D06 | §2 | Tela | T-020: Pipeline — Visão Kanban | Rota /pipeline; colunas por status do caso | S4 |
| REQ-199 | D06 | §2 | Tela | T-021: Pipeline — Visão Lista | Rota /pipeline?view=list | S4 |
| REQ-200 | D06 | §2 | Tela | T-022: Pipeline — Detalhe do Caso | Rota /pipeline/:id; histórico de status | S4 |
| REQ-201 | D06 | §2 | Tela | T-030: Triagem — Fila FIFO | Rota /triagem; ordem FIFO; alerta informativo se fora de ordem | S4b |
| REQ-202 | D06 | §2 | Tela | T-031: Triagem — Painel do Caso | Rota /triagem/:id | S4b |
| REQ-203 | D06 | §2 | Tela | T-032: Triagem — Aba Dossiê | Dentro de T-031; checklist 6 docs; barra de progresso | S4b |
| REQ-204 | D06 | §2 | Tela | T-033: Triagem — Aba Adimplência | Dentro de T-031; verificação de parcelas | S4b |
| REQ-205 | D06 | §2 | Tela | T-034: Modal — Confirmar Bloqueio de Caso | Mensagem de confirmação; spinner no botão | S4b |
| REQ-206 | D06 | §2 | Tela | T-040: Negociação — Lista de Casos em Negociação | Rota /negociacao; filtros de status | S5 |
| REQ-207 | D06 | §2 | Tela | T-041: Negociação — Painel do Caso | Rota /negociacao/:id; layout 3 colunas | S5 |
| REQ-208 | D06 | §2 | Tela | T-042: Negociação — Timeline de Propostas | Dentro de T-041; barra de progresso rodadas 3 segmentos | S5 |
| REQ-209 | D06 | §2 | Tela | T-043: Modal — Aceitar Proposta | Confirmação de aceite com resumo | S5 |
| REQ-210 | D06 | §2 | Tela | T-044: Modal — Recusar / Contraproposta | Campo pré-preenchido com valor anterior (DEC-008) | S5 |
| REQ-211 | D06 | §2 | Tela | T-045: Modal — Aprovar Escalonamento | Simulação comparativa lado a lado | S5 |
| REQ-212 | D06 | §2 | Tela | T-050: Formalização — Lista de Casos | Rota /formalizacao | S7 |
| REQ-213 | D06 | §2 | Tela | T-051: Formalização — Painel do Caso | Rota /formalizacao/:id; 4 cards de critérios | S7 |
| REQ-214 | D06 | §2 | Tela | T-052: Formalização — Card Assinaturas (ZapSign) | Barra de progresso X/3 assinaturas; reenvio após 5 dias | S7 |
| REQ-215 | D06 | §2 | Tela | T-053: Formalização — Card Anuência da Construtora | Estados: Não solicitada, Solicitada, Recebida, Em atraso, Dispensada | S7 |
| REQ-216 | D06 | §2 | Tela | T-054: Formalização — Card Depósito Escrow | Ícones de estado por status da conta | S7 |
| REQ-217 | D06 | §2 | Tela | T-055: Modal — Confirmar Fechamento | Grid 2x2 dos 4 critérios; botão pulsante; digitação "CONFIRMAR" | S7 |
| REQ-218 | D06 | §2 | Tela | T-060: Financeiro — Contas Escrow (tabela) | Rota /financeiro; filtros chips; painel lateral (drawer) | S6 |
| REQ-219 | D06 | §2 | Tela | T-061: Financeiro — Drawer Detalhe da Conta Escrow | Breakdown barras empilhadas Cedente vs RS | S6 |
| REQ-220 | D06 | §2 | Tela | T-062: Financeiro — Painel Comissões e Faturas | — | S6 |
| REQ-221 | D06 | §2 | Tela | T-063: Financeiro — Painel Inadimplência | Alertas amarelo/vermelho; prorrogação +5 dias úteis | S6 |
| REQ-222 | D06 | §2 | Tela | T-064: Modal — Bloquear Distribuição | Justificativa ≥20 chars; envio ao Master | S6 |
| REQ-223 | D06 | §2 | Tela | T-065: Modal — Iniciar Reversão (Estorno) | Resumo de impacto; confirmação | S7 |
| REQ-224 | D06 | §2 | Tela | T-066: Modal — Conciliação Bancária | Comparação valor esperado vs confirmado | S6 |
| REQ-225 | D06 | §2 | Tela | T-070: Supervisão IA — Painel Principal | Rota /supervisao-ia; métricas resumidas + log de ações | S9 |
| REQ-226 | D06 | §2 | Tela | T-071: Supervisão IA — Log de Decisões do Agente | Tabela com: timestamp, agente, caso, tipo, confiança, resultado | S9 |
| REQ-227 | D06 | §2 | Tela | T-072: Modal — Takeover Manual de Agente | Campo justificativa; contador X/5; badge "Controle manual ativo" borda laranja | S9 |
| REQ-228 | D06 | §2 | Tela | T-075: Usuários — Lista Operadores | Rota /usuarios; filtros role, is_active | S8 |
| REQ-229 | D06 | §2 | Tela | T-076: Usuários — Perfil do Operador | Alterar role, suspender, reativar | S8 |
| REQ-230 | D06 | §2 | Tela | T-077: Usuários — Lista Cedentes | Rota /usuarios/cedentes | S8 |
| REQ-231 | D06 | §2 | Tela | T-078: Usuários — Perfil do Cedente | CPF mascarado para Analista | S8 |
| REQ-232 | D06 | §2 | Tela | T-079: Usuários — Lista Cessionários | Rota /usuarios/cessionarios | S8 |
| REQ-233 | D06 | §2 | Tela | T-080: Usuários — Perfil do Cessionário | KYC badge; dados mascarados | S8 |
| REQ-234 | D06 | §2 | Tela | T-081: Modal — Suspender / Reativar Usuário | Resumo de impacto (X casos congelados, Y propostas canceladas) | S8 |
| REQ-235 | D06 | §2 | Tela | T-085: Relatórios — Hub de Relatórios | Rota /relatorios | S10 |
| REQ-236 | D06 | §2 | Tela | T-086: Relatórios — SLA Operacional | — | S10 |
| REQ-237 | D06 | §2 | Tela | T-087: Relatórios — Volume de Casos | — | S10 |
| REQ-238 | D06 | §2 | Tela | T-088: Relatórios — Receita (Gestor Financeiro + Master only) | — | S10 |
| REQ-239 | D06 | §2 | Tela | T-089: Relatórios — Conversão | — | S10 |
| REQ-240 | D06 | §2 | Tela | T-090: Relatórios — Auditoria (Gestor Financeiro + Master only) | — | S10 |
| REQ-241 | D06 | §2 | Tela | T-091: Relatórios — Agentes de IA | — | S10 |
| REQ-242 | D06 | §2 | Tela | T-095: Configurações — Hub | Rota /configuracoes; apenas Master | S11 |
| REQ-243 | D06 | §2 | Tela | T-096: Configurações — Comissões e Parâmetros Financeiros | Percentuais, Distrato Ref., limiar Delta | S11 |
| REQ-244 | D06 | §2 | Tela | T-097: Configurações — Prazos e SLAs | SLAs configuráveis por etapa | S11 |
| REQ-245 | D06 | §2 | Tela | T-098: Configurações — Templates de Notificação | Editor de templates; canal E-mail | S11 |
| REQ-246 | D06 | §2 | Tela | T-099: Configurações — Integrações (ZapSign, Celcoin, Meta, Twilio) | API Keys, webhook secrets, teste de conexão | S11 |
| REQ-247 | D06 | §2 | Tela | T-100: Configurações — Usuários e Permissões | Criar/editar operadores via tela de config | S11 |
| REQ-248 | D10 | §4 | Glossário | Nomenclatura canônica entre camadas (18 entidades) | Usar EXATAMENTE os nomes: cases, cedentes, cessionarios, proposals, counterproposals, zapsign_envelopes, formalization_criteria, escrow_accounts, escrow_transactions, commission_invoices, distributions, notification_logs, ai_agent_decisions, global_configs, case_config_snapshots, case_status_history, dossie_documents, audit_logs | CROSS |
| REQ-249 | D10 | §2 | Glossário | 14 sinônimos resolvidos | Usar apenas termos canônicos na codebase (ex: nunca "Backoffice" → sempre "Admin") | CROSS |
| REQ-250 | D12 | §3 | Banco | Tabela `users` — 13 colunas | UUID PK, email UNIQUE, name, role ENUM UserRole, password_hash, is_active, two_fa_enabled, two_fa_secret, failed_login_attempts, locked_until, version, created_at, updated_at, deleted_at; índices: UNIQUE(email), INDEX(role), INDEX(is_active, deleted_at) | S1 |
| REQ-251 | D12 | §3 | Banco | Enum `UserRole` = ANALISTA, COORDENADOR, GESTOR_FINANCEIRO, MASTER | — | S1 |
| REQ-252 | D12 | §3 | Banco | Tabela `cedentes` — cpf UNIQUE, email UNIQUE, status ENUM UserExternalStatus, version, soft delete | — | S1 |
| REQ-253 | D12 | §3 | Banco | Enum `UserExternalStatus` = ATIVO, SUSPENSO, INATIVO | — | S1 |
| REQ-254 | D12 | §3 | Banco | Tabela `cessionarios` — estrutura idêntica a cedentes | — | S1 |
| REQ-255 | D12 | §3 | Banco | Tabela `cases` — 19+ colunas incluindo status, scenario, contract_table_value, current_table_value, delta, paid_amount, distrato_reference_value, version, soft delete | — | S1 |
| REQ-256 | D12 | §3 | Banco | Enum `CaseStatus` = CAPTADO, EM_TRIAGEM, BLOQUEADO, QUALIFICADO, OFERTA_ATIVA, EM_NEGOCIACAO, EM_FORMALIZACAO, FECHAMENTO, POS_FECHAMENTO, EM_REVERSAO, EM_MEDIACAO, DISPUTA_FORMAL, CONCLUIDO, CANCELADO | S1 |
| REQ-257 | D12 | §3 | Banco | Enum `CaseScenario` = A, B, C, D | — | S1 |
| REQ-258 | D12 | §3 | Banco | Tabela `case_config_snapshots` — commission_cedente_pct, commission_cessionario_pct, delta_review_threshold, distrato_reference_pct, snapshot_at | — | S1 |
| REQ-259 | D12 | §3 | Banco | Tabela `case_status_history` — from_status, to_status, changed_by, reason | — | S1 |
| REQ-260 | D12 | §3 | Banco | Tabela `dossie_documents` — document_type, storage_path, status ENUM, rejection_reason, verified_by, verified_at, version, soft delete | — | S1 |
| REQ-261 | D12 | §3 | Banco | Tabela `proposals` — proposed_value, status, expires_at, version, soft delete | — | S1 |
| REQ-262 | D12 | §3 | Banco | Tabela `counterproposals` — counter_value, reason | — | S1 |
| REQ-263 | D12 | §3 | Banco | Tabela `zapsign_envelopes` — document_type, zapsign_document_token, status, signatories (JSONB), expires_at, resend_count, soft delete | — | S1 |
| REQ-264 | D12 | §3 | Banco | Tabela `formalization_criteria` — signatures_ok, annuence_ok, annuence_na, annuence_protocol, annuence_document_path, deposit_ok, instrument_ok, delta_approved, delta_approved_by | — | S1 |
| REQ-265 | D12 | §3 | Banco | Tabela `escrow_accounts` — celcoin_account_id, status, expected_amount, deposited_amount, residual_credit, distribution_due_at, distribution_blocked, block_reason, version | — | S1 |
| REQ-266 | D12 | §3 | Banco | Tabela `escrow_transactions` — transaction_type, amount, celcoin_transaction_id | — | S1 |
| REQ-267 | D12 | §3 | Banco | Tabela `commission_invoices` — party_type, commission_amount, base_amount, commission_pct, status, distributed_at | — | S1 |
| REQ-268 | D12 | §3 | Banco | Tabela `distributions` — recipient_type, recipient_id, amount, status, celcoin_transfer_id | — | S1 |
| REQ-269 | D12 | §3 | Banco | Tabela `notification_logs` — recipient_type, channel, event_type, status, payload (JSONB), error_message | — | S1 |
| REQ-270 | D12 | §3 | Banco | Tabela `ai_agent_decisions` — agent_name, decision_type, decision_data (JSONB), confidence_score, outcome, operator_feedback, reviewed_by | — | S1 |
| REQ-271 | D12 | §3 | Banco | Tabela `global_configs` — config_key, config_value, data_type, description, update_reason | — | S1 |
| REQ-272 | D12 | §1 | Banco | Schema `audit.audit_logs` — 13 campos obrigatórios; append-only; retenção 5 anos | — | S1 |
| REQ-273 | D16 | §2 | Endpoint | POST /v1/auth/login — público; resposta 200/401/423 | — | S2 |
| REQ-274 | D16 | §2 | Endpoint | POST /v1/auth/verify-2fa — sessão intermediária com temp_token | — | S2 |
| REQ-275 | D16 | §2 | Endpoint | POST /v1/auth/refresh — público com refresh_token; rotação | — | S2 |
| REQ-276 | D16 | §2 | Endpoint | POST /v1/auth/logout — invalida token no Redis blacklist; 204 | — | S2 |
| REQ-277 | D16 | §2 | Endpoint | POST /v1/auth/forgot-password — público; resposta sempre 200 (segurança) | — | S2 |
| REQ-278 | D16 | §2 | Endpoint | POST /v1/auth/reset-password — token de reset; 400 se expirado | — | S2 |
| REQ-279 | D16 | §3 | Endpoint | GET /v1/users — COORDENADOR; filtros: page, per_page, role, is_active, search | — | S8 |
| REQ-280 | D16 | §3 | Endpoint | POST /v1/users — MASTER; 409 se e-mail duplicado | — | S8 |
| REQ-281 | D16 | §3 | Endpoint | GET /v1/users/:id — COORDENADOR | — | S8 |
| REQ-282 | D16 | §3 | Endpoint | PATCH /v1/users/:id — MASTER; version no body; 409 conflict | — | S8 |
| REQ-283 | D16 | §3 | Endpoint | DELETE /v1/users/:id — MASTER; soft delete; 403 se próprio usuário | — | S8 |
| REQ-284 | D16 | §4 | Endpoint | GET /v1/cedentes — ANALISTA; filtros page, per_page, status, search | — | S8 |
| REQ-285 | D16 | §4 | Endpoint | POST /v1/cedentes — ANALISTA; 409 se CPF/email duplicado | — | S8 |
| REQ-286 | D16 | §4 | Endpoint | PATCH /v1/cedentes/:id/suspend — COORDENADOR; suspension_reason ≥20 chars | — | S8 |
| REQ-287 | D16 | §5 | Endpoint | /v1/cessionarios — estrutura idêntica a /cedentes | — | S8 |
| REQ-288 | D16 | §6 | Endpoint | GET /v1/cases — ANALISTA; filtros: status, scenario, assigned_analyst_id, sort, search | — | S4 |
| REQ-289 | D16 | §6 | Endpoint | POST /v1/cases — ANALISTA; cria também case_config_snapshot, formalization_criteria, escrow_account | — | S4 |
| REQ-290 | D16 | §6 | Endpoint | GET /v1/cases/:id — ANALISTA; inclui config_snapshot, formalization_criteria, escrow_account summary | — | S4 |
| REQ-291 | D16 | §6 | Endpoint | PATCH /v1/cases/:id — ANALISTA; campos editáveis + version | — | S4 |
| REQ-292 | D16 | §6 | Endpoint | PATCH /v1/cases/:id/status — ANALISTA/COORDENADOR; 422 se transição inválida | — | S4 |
| REQ-293 | D16 | §6 | Endpoint | PATCH /v1/cases/:id/assign — COORDENADOR; atribui analista | — | S4 |
| REQ-294 | D16 | §6 | Endpoint | PATCH /v1/cases/:id/close — ANALISTA; 422 se critérios incompletos | — | S7 |
| REQ-295 | D16 | §6 | Endpoint | GET /v1/cases/:id/status-history — ANALISTA | — | S4 |
| REQ-296 | D16 | §7 | Endpoint | GET /v1/cases/:id/dossie — ANALISTA | — | S4b |
| REQ-297 | D16 | §7 | Endpoint | POST /v1/cases/:id/dossie — ANALISTA; multipart/form-data; max 10MB; pdf/jpg/png | — | S4b |
| REQ-298 | D16 | §7 | Endpoint | PATCH /v1/cases/:id/dossie/:doc_id/approve — COORDENADOR | — | S4b |
| REQ-299 | D16 | §7 | Endpoint | PATCH /v1/cases/:id/dossie/:doc_id/reject — COORDENADOR; rejection_reason ≥20 chars | — | S4b |
| REQ-300 | D16 | §8 | Endpoint | GET/POST /v1/cases/:id/proposals — ANALISTA | — | S5 |
| REQ-301 | D16 | §8 | Endpoint | PATCH /v1/cases/:id/proposals/:id/accept — ANALISTA | — | S5 |
| REQ-302 | D16 | §8 | Endpoint | PATCH /v1/cases/:id/proposals/:id/reject — ANALISTA | — | S5 |
| REQ-303 | D16 | §8 | Endpoint | GET/POST /v1/cases/:id/proposals/:id/counterproposals — ANALISTA | — | S5 |
| REQ-304 | D16 | §9 | Endpoint | GET/PATCH /v1/cases/:id/formalization — ANALISTA | — | S7 |
| REQ-305 | D16 | §9 | Endpoint | GET/POST /v1/cases/:id/zapsign — ANALISTA | — | S7 |
| REQ-306 | D16 | §9 | Endpoint | POST /v1/cases/:id/zapsign/:envelope_id/resend — ANALISTA | — | S7 |
| REQ-307 | D16 | §10 | Endpoint | GET /v1/cases/:id/escrow — GESTOR_FINANCEIRO | — | S6 |
| REQ-308 | D16 | §10 | Endpoint | POST /v1/cases/:id/escrow/distribute — GESTOR_FINANCEIRO; 202 assíncrono | — | S6 |
| REQ-309 | D16 | §10 | Endpoint | PATCH /v1/cases/:id/escrow/block — GESTOR_FINANCEIRO | — | S6 |
| REQ-310 | D16 | §10 | Endpoint | PATCH /v1/cases/:id/escrow/unblock — MASTER | — | S6 |
| REQ-311 | D16 | §11 | Endpoint | GET /v1/ai/decisions — COORDENADOR; filtros agent_name, case_id, outcome, date_range | — | S9 |
| REQ-312 | D16 | §11 | Endpoint | POST /v1/ai/decisions/:id/takeover — COORDENADOR; justificativa obrigatória | — | S9 |

---

## Verificação de Cobertura (Etapa 3)

```
Registro Mestre: 312 requisitos
Atribuídos: 312 (100%)
Sem sprint: 0
```

### Distribuição por Sprint

| Sprint | Qtd REQs |
|--------|----------|
| S1 — Fundação | 42 |
| S2 — Auth e RBAC | 29 |
| S3 — Dashboard | 9 |
| S4 — Pipeline e Casos | 18 |
| S4b — Triagem | 22 |
| S5 — Negociação | 20 |
| S6 — Financeiro | 24 |
| S7 — Formalização | 28 |
| S8 — Usuários | 17 |
| S8a — Notificações | 3 |
| S9 — Supervisão IA | 21 |
| S10 — Relatórios | 10 |
| S11 — Configurações | 10 |
| S12 — Mobile | 1 |
| S13 — Qualidade | 1 |
| CROSS | 7 |
| **Total** | **312** |
