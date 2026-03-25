# Registro Mestre de Requisitos — Módulo Cessionário

| **Campo** | **Valor** |
|---|---|
| **Módulo** | Cessionário · Plataforma Repasse Seguro |
| **Pipeline** | ShiftLabs v2.3 — Fase 3 |
| **Gerado em** | 2026-03-24 |
| **Total de REQs** | 285 |

---

## Tabela de Requisitos Atômicos

| ID | Doc Fonte | Seção | Tipo | Descrição | Valor/Detalhe | Sprint |
|---|---|---|---|---|---|---|
| REQ-001 | D01.1 | Glossário | Domínio | Cessionário — entidade | Investidor que adquire direitos de cessão. Entidade `cessionarios` no banco | S1 |
| REQ-002 | D01.1 | Glossário | Domínio | Cedente — anonimato estrutural | Dados pessoais nunca chegam ao frontend do Cessionário | CROSS |
| REQ-003 | D01.1 | Glossário | Domínio | Escrow — conta-garantia | Cessionário deposita Preço Repasse + Comissão Comprador | CROSS |
| REQ-004 | D01.1 | Glossário | Regra | Comissão Comprador — fórmula geral | 20% × (Tabela Atual − Tabela Contrato) quando Δ > 0 | S5 |
| REQ-005 | D01.1 | Glossário | Regra | Comissão Comprador — exceção Δ ≤ 0 | 20% × Valor Pago pelo Cedente quando Δ ≤ 0 | S5 |
| REQ-006 | D01.1 | Glossário | Domínio | KYC — composição obrigatória | Documento de identidade (frente e verso) + comprovante de endereço + selfie de prova de vida | S2 |
| REQ-007 | D01.1 | §4.1 | Estado | Conta: Cadastrada → KYC em Análise | Transição ao enviar documentos | S2 |
| REQ-008 | D01.1 | §4.1 | Estado | Conta: KYC em Análise → KYC Aprovado | Transição ao aprovar validação | S2 |
| REQ-009 | D01.1 | §4.1 | Estado | Conta: KYC em Análise → KYC Reprovado | Transição ao reprovar validação | S2 |
| REQ-010 | D01.1 | §4.1 | Estado | Conta: KYC Reprovado → KYC em Análise | Transição ao reenviar documentos | S2 |
| REQ-011 | D01.1 | §4.1 | Estado | Conta: KYC Aprovado → Bloqueada Temporariamente | Transição por restrição Admin | S2 |
| REQ-012 | D01.1 | §4.1 | Estado | Conta: Bloqueada Temporariamente → KYC Aprovado | Transição ao remover bloqueio | S2 |
| REQ-013 | D01.1 | §4.1 | Estado | Conta: KYC Aprovado → Encerrada | Transição ao solicitar exclusão | S2 |
| REQ-014 | D01.1 | §4.1 | Estado | Conta: Cadastrada → Encerrada | Transição ao solicitar exclusão sem KYC | S2 |
| REQ-015 | D01.1 | RN-001 | RF | Cadastro — campos obrigatórios | nome completo, e-mail, senha | S2 |
| REQ-016 | D01.1 | RN-001 | RF | Cadastro — indicador de força de senha em tempo real | fraca / média / forte | S2 |
| REQ-017 | D01.1 | RN-001 | RF | Cadastro — verificação de e-mail duplicado | Exibe MSG: "Este e-mail já está em uso. Tente fazer login ou recupere sua senha." Campo destacado em vermelho | S2 |
| REQ-018 | D01.1 | RN-001 | RF | Cadastro — conta criada com status Cadastrada | Envia e-mail de verificação, redireciona para Dashboard com banner de boas-vindas | S2 |
| REQ-019 | D01.1 | RN-001 | RF | Banner de boas-vindas — CTAs | "Completar KYC" (primário) + "Explorar Oportunidades" (secundário). Persiste até dispensado por "×" | S2 |
| REQ-020 | D01.1 | RN-001 | RF | Cadastro — validação on blur em tempo real | Campos inválidos com mensagem vermelha abaixo | S2 |
| REQ-021 | D01.1 | RN-001 | RF | Cadastro — preservar dados em caso de falha de conexão | MSG: "Não foi possível concluir o cadastro. Verifique sua conexão e tente novamente." | S2 |
| REQ-022 | D01.1 | RN-002 | RF | Login — indicador de carregamento no botão "Entrar" | Botão desabilitado com spinner durante verificação | S2 |
| REQ-023 | D01.1 | RN-002 | RF | Login — mensagem de credenciais incorretas | "E-mail ou senha incorretos. Verifique seus dados ou recupere sua senha." Banner acima do formulário. Senha limpa, e-mail preservado | S2 |
| REQ-024 | D01.1 | RN-002 | RF | Login — conta bloqueada | "Sua conta está temporariamente suspensa. Entre em contato com o suporte para mais informações." | S2 |
| REQ-025 | D01.1 | RN-002 | RF | Login — conta encerrada | "Esta conta foi encerrada. Não é possível fazer login." | S2 |
| REQ-026 | D01.1 | RN-003 | RF | Login com Google OAuth | Fluxo OAuth: conta existente → sessão; conta nova → criar conta + boas-vindas; falha → MSG retorno | S2 |
| REQ-027 | D01.1 | RN-004 | Regra | Sessão — expiração por inatividade em 30 minutos | Prazo autoritativo para perfil Cessionário | S2 |
| REQ-028 | D01.1 | RN-004 | RF | Aviso de sessão expirando — modal 5 min antes | Modal: "Sua sessão expira em 5 minutos. Deseja continuar?" com "Continuar sessão" e "Sair agora" | S2 |
| REQ-029 | D01.1 | RN-004 | RF | Sessão expirada — redirecionamento para login | MSG: "Sua sessão expirou por inatividade. Faça login novamente para continuar." | S2 |
| REQ-030 | D01.1 | RN-005 | Regra | Re-autenticação para ações críticas | Janela de confiança: 15 minutos | S2 |
| REQ-031 | D01.1 | RN-005 | RF | Re-autenticação — ações que exigem | Depósito em Escrow + assinatura de documento + alteração de dados bancários + solicitação de reversão | S2 |
| REQ-032 | D01.1 | RN-005 | RF | Re-autenticação — modal overlay sem perder contexto | Solicita senha ou biometria; se confirmado → ação prossegue automaticamente | S2 |
| REQ-033 | D01.1 | RN-006 | Regra | Bloqueio de KYC por excesso de falhas | 5 tentativas em 1h → bloqueio de 30 minutos | S2 |
| REQ-034 | D01.1 | RN-006 | RF | Contador regressivo visível durante bloqueio de KYC | Ex: "Disponível em 28:34". Botão de envio desabilitado | S2 |
| REQ-035 | D01.1 | RN-007 | RF | Edição de e-mail — verificação por link | Envia link para novo e-mail antes de salvar | S3 |
| REQ-036 | D01.1 | RN-007 | RF | Edição de telefone — verificação por SMS | Envia código SMS para novo número antes de salvar | S3 |
| REQ-037 | D01.1 | RN-007 | RF | Edição de dados bancários — re-autenticação obrigatória | Conforme RN-005 | S3 |
| REQ-038 | D01.1 | RN-007 | RF | Toast de confirmação de edição | "Dado atualizado com sucesso." desaparece em 5 segundos | S3 |
| REQ-039 | D01.1 | RN-007 | RF | Código/link expirado em 10 min | MSG: "Alteração não confirmada. O código expirou. Solicite um novo código para salvar." | S3 |
| REQ-040 | D01.1 | RN-007 | RF | Código incorreto — até 3 tentativas | MSG: "Código incorreto. Verifique e tente novamente." | S3 |
| REQ-041 | D01.1 | RN-008 | RF | Preferências de notificação — canais | e-mail (obrigatório/não desabilitável), push, SMS | S3 |
| REQ-042 | D01.1 | RN-008 | Regra | E-mail como canal mínimo obrigatório | MSG se tentar desabilitar: "O e-mail não pode ser desabilitado — é o canal mínimo obrigatório para notificações críticas de prazo." | S3 |
| REQ-043 | D01.1 | RN-009 | RF | KYC obrigatório antes da primeira proposta | Botão "Fazer Proposta" desabilitado sem KYC_APROVADO + banner persistente | S2 |
| REQ-044 | D01.1 | RN-010 | RF | KYC — stepper 3 passos | Passo 1: Identidade (frente+verso, JPG/PNG/PDF, máx 10MB) · Passo 2: Comprovante endereço (≤90 dias, JPG/PNG/PDF, máx 10MB) · Passo 3: Selfie liveness | S2 |
| REQ-045 | D01.1 | RN-010 | RF | KYC — preview de arquivo antes de avançar | Opção de remover e reenviar antes do próximo passo | S2 |
| REQ-046 | D01.1 | RN-010 | RF | KYC — preservar progresso por 24h | Abandono do fluxo preserva documentos já enviados por 24 horas | S2 |
| REQ-047 | D01.1 | RN-011 | RF | Validação automática de KYC em até 5 minutos | via OCR + liveness check (idwall) | S2 |
| REQ-048 | D01.1 | RN-011 | RF | Badge verde "Verificado" após KYC aprovado | Exibido ao lado do nome no perfil | S2 |
| REQ-049 | D01.1 | RN-011 | RF | KYC reprovado — motivo por documento | Destaque vermelho por documento + botão "Reenviar" apenas para documentos com problema | S2 |
| REQ-050 | D01.1 | RN-011 | RF | KYC inconclusivo → revisão manual | Status "Em análise manual" com MSG: "Sua verificação requer análise adicional. Prazo estimado: até 24 horas úteis." | S2 |
| REQ-051 | D01.1 | RN-012 | RF | Reenvio parcial de documentos KYC | Apenas documentos reprovados precisam ser reenviados | S2 |
| REQ-052 | D01.1 | RN-013 | Regra | RBAC — isolamento de dados por cessionario_id | Filtro obrigatório em todas as queries. Bloqueio retorna "Nenhum resultado encontrado." | CROSS |
| REQ-053 | D01.1 | RN-014 | Regra | Anonimização do Cedente — dados suprimidos | nome, CPF, contato, endereço, identificadores internos nunca trafegam ao frontend | CROSS |
| REQ-054 | D01.1 | RN-015 | RF | Exportação de dados LGPD — prazo 48h | Arquivo JSON ou PDF disponível para download por 7 dias | S3 |
| REQ-055 | D01.1 | RN-015 | RF | Exclusão de conta — confirmação irreversível | Modal com "Esta ação não pode ser desfeita. Dados financeiros retidos por 5 anos." Botão vermelho "Confirmar exclusão" | S3 |
| REQ-056 | D01.1 | RN-016 | RF | Consentimento IA — checkboxes individuais no cadastro | Termos de Uso + Política de Privacidade (obrigatório) + uso de dados pela IA (obrigatório). Links abrem em nova aba | S2 |
| REQ-057 | D01.1 | RN-016 | RF | Revogação de consentimento IA em Meu Perfil | Modal explicativo antes da revogação com MSG sobre perda de recomendações | S3 |
| REQ-058 | D01.2 | §3 | Estado | Oportunidade: Disponível → Com Proposta | Primeira proposta recebida | S4 |
| REQ-059 | D01.2 | §3 | Estado | Oportunidade: Com Proposta → Disponível | Todas propostas recusadas/expiradas | S4 |
| REQ-060 | D01.2 | §3 | Estado | Oportunidade: Com Proposta → Em Negociação | Proposta aceita pelo Admin | S4 |
| REQ-061 | D01.2 | §3 | Estado | Oportunidade: Em Negociação → Disponível | Negociação cancelada sem depósito | S6 |
| REQ-062 | D01.2 | §3 | Estado | Oportunidade: Em Negociação → Reservada | Escrow confirmado | S6 |
| REQ-063 | D01.2 | §3 | Estado | Oportunidade: Reservada → Concluída | 4 critérios de fechamento atendidos | S8 |
| REQ-064 | D01.2 | §3 | Estado | Oportunidade: Reservada → Disponível | Formalização cancelada | S7 |
| REQ-065 | D01.2 | RN-017 | RF | Marketplace — campos exibidos por oportunidade | código (OPR-XXXX-NNNN), cidade/bairro/empreendimento, tipologia (quartos/metragem/vaga), Tabela Atual, Tabela Contrato, Δ, % pago pelo Cedente, score de risco (1-10), data de publicação | S4 |
| REQ-066 | D01.2 | RN-017 | RF | Marketplace — skeleton cards durante carregamento (máx 3s) | Placeholders animados no layout da listagem | S4 |
| REQ-067 | D01.2 | RN-017 | Regra | Marketplace — dados NUNCA exibidos | nome, CPF, contato, endereço do Cedente; cenário A/B/C/D; termos de negociação do Cedente; motivo da cessão | CROSS |
| REQ-068 | D01.2 | RN-018 | RF | Filtros do marketplace — 5 filtros disponíveis | Localização (estado/cidade/bairro) · Faixa de preço da Tabela Atual · Tipologia (quartos/metragem) · Score de risco (min-max 1-10) · Delta mínimo (R$ ou %) | S4 |
| REQ-069 | D01.2 | RN-018 | RF | Ordenações do marketplace — 4 opções | Mais recentes · Maior delta · Menor risco · Recomendados pela IA (padrão) | S4 |
| REQ-070 | D01.2 | RN-018 | RF | Ordenação padrão "Recomendados pela IA" — fallback para "Mais recentes" sem histórico | Automático sem intervenção manual | S4 |
| REQ-071 | D01.2 | RN-019 | RF | Detalhe de oportunidade — campos expandidos | Todos os campos da listagem + gráfico de valorização + comparativo IA + simulação de custos (Preço Repasse + Comissão estimada) | S4 |
| REQ-072 | D01.2 | RN-019 | RF | Botão "Fazer Proposta" — 3 condições simultâneas | KYC_APROVADO + limite simultâneo < 3 + rate limit < 10/24h | S4 |
| REQ-073 | D01.2 | RN-019 | RF | Tooltip por condição não atendida | KYC: "Complete sua verificação..." · Limite simultâneo: "Você atingiu o limite de 3 propostas simultâneas." · Rate limit: "Limite de propostas diárias atingido. Próxima disponível às {horário}." | S4 |
| REQ-074 | D01.2 | RN-019 | RF | Botão "Consultar Analista" — disponível para qualquer KYC | Botão secundário (outline) ao lado do primário "Fazer Proposta" | S4 |
| REQ-075 | D01.2 | §4 | Estado | Proposta: Inexistente → Enviada | Proposta submetida — conta no limite simultâneo | S4 |
| REQ-076 | D01.2 | §4 | Estado | Proposta: Enviada → Em Análise | Admin inicia análise — ainda conta no limite | S4 |
| REQ-077 | D01.2 | §4 | Estado | Proposta: Enviada → Cancelada | Cessionário cancela — libera vaga no limite | S4 |
| REQ-078 | D01.2 | §4 | Estado | Proposta: Enviada → Expirada | 72h úteis sem resposta — libera vaga no limite | S4 |
| REQ-079 | D01.2 | §4 | Estado | Proposta: Em Análise → Aceita | Admin/Cedente aceita — não conta no limite | S4 |
| REQ-080 | D01.2 | §4 | Estado | Proposta: Em Análise → Recusada | Admin/Cedente recusa — não conta no limite | S4 |
| REQ-081 | D01.2 | §4 | Estado | Proposta: Em Análise → Expirada | 72h úteis sem resposta — não conta no limite | S4 |
| REQ-082 | D01.2 | RN-020 | Regra | Limite de propostas — simultâneas máximo 3 | Conta ENVIADA + EM_ANALISE; MSG: "Você atingiu o limite de 3 propostas simultâneas. Aguarde uma resposta antes de enviar uma nova proposta." | S4 |
| REQ-083 | D01.2 | RN-020 | Regra | Limite de propostas — rate limit 10/24h (janela móvel) | MSG com horário de liberação: "Você atingiu o limite de 10 propostas nas últimas 24 horas. Próxima proposta disponível às {horário}." | S4 |
| REQ-084 | D01.2 | RN-021 | RF | Formulário de proposta — campos | Oportunidade (automático) + Valor da proposta em R$ (> 0) + Mensagem ao Admin (max 500 chars, opcional) + Checkbox ciência dos termos (obrigatório) | S4 |
| REQ-085 | D01.2 | RN-021 | RF | Envio de proposta — 5 validações antes de registrar | KYC_APROVADO + limite simultâneo + rate limit + valor > 0 + oportunidade DISPONIVEL | S4 |
| REQ-086 | D01.2 | RN-021 | RF | Confirmação de proposta enviada | Resumo (código, valor, comissão estimada) + "Proposta enviada com sucesso!" + botões "Ver minhas propostas" e "Voltar ao marketplace" | S4 |
| REQ-087 | D01.2 | RN-021 | RF | Oportunidade indisponível durante preenchimento | Modal: "Esta oportunidade não está mais disponível para novas propostas. Explore outras oportunidades no marketplace." | S4 |
| REQ-088 | D01.2 | RN-022 | RF | Cancelamento de proposta — somente status ENVIADA | Propostas EM_ANALISE não canceláveis pelo Cessionário | S4 |
| REQ-089 | D01.2 | RN-022 | RF | Modal de confirmação de cancelamento | "Tem certeza que deseja cancelar esta proposta para {código}? Esta ação não pode ser desfeita." Botão "Cancelar proposta" vermelho | S4 |
| REQ-090 | D01.2 | RN-023 | Regra | Expiração automática em 72h úteis sem resposta | Sistema executa automaticamente; envia NOT-CES-15; libera vaga no limite | S4 |
| REQ-091 | D01.2 | §5 | Estado | Negociação: Inexistente → Em Negociação | Proposta aceita pelo Admin | S5 |
| REQ-092 | D01.2 | §5 | Estado | Negociação: Em Negociação → Em Contraproposta | Cessionário envia contraproposta | S5 |
| REQ-093 | D01.2 | §5 | Estado | Negociação: Em Contraproposta → Em Negociação | Contraproposta aceita | S5 |
| REQ-094 | D01.2 | §5 | Estado | Negociação: Em Contraproposta → Encerrada | 3ª contraproposta recusada | S5 |
| REQ-095 | D01.2 | §5 | Estado | Negociação: Em Negociação → Aguardando Depósito | Valor acordado | S5 |
| REQ-096 | D01.2 | §5 | Estado | Negociação: Aguardando Depósito → Depósito Confirmado | Admin confirma | S6 |
| REQ-097 | D01.2 | §5 | Estado | Negociação: Aguardando Depósito → Cancelada | Prazo expirado sem depósito | S6 |
| REQ-098 | D01.2 | RN-024 | RF | Negociação criada automaticamente ao aceitar proposta | Status EM_NEGOCIACAO; envia NOT-CES-03 (e-mail + push + SMS) | S5 |
| REQ-099 | D01.2 | RN-025 | RF | Dados visíveis ao Cessionário na negociação | código da oportunidade, valor da proposta aceita, Comissão Comprador, total Escrow, prazo, status, histórico de mensagens | S5 |
| REQ-100 | D01.2 | RN-025 | Regra | Dados NUNCA visíveis ao Cessionário na negociação | Identidade do Cedente, termos internos do Cedente, Comissão Vendedor | CROSS |
| REQ-101 | D01.2 | RN-026 | RF | Chat da negociação — recursos | Mensagens de texto + anexo de documentos + histórico completo | S5 |
| REQ-102 | D01.2 | RN-026 | Regra | Chat intermediado pelo Admin | Cessionário nunca tem contato direto com Cedente | CROSS |
| REQ-103 | D01.2 | RN-026 | Notificação | NOT-CES-14 — nova mensagem do Admin | Canal: e-mail + push | S5 |
| REQ-104 | D01.2 | RN-027 | RF | Contraproposta — 3 validações | Valor diferente do anterior + valor > 0 + rodadas < 3 | S5 |
| REQ-105 | D01.2 | RN-027 | Regra | Limite de 3 rodadas de contraproposta | Botão oculto após 3 rodadas | S5 |
| REQ-106 | D01.2 | RN-027 | RF | Recálculo automático da Comissão Comprador | Ao enviar contraproposta com novo valor | S5 |
| REQ-107 | D01.2 | RN-028 | RF | Instrução de depósito — breakdwown visível | "R$ 500.000 + R$ 30.000 de comissão = R$ 530.000" | S6 |
| REQ-108 | D01.2 | RN-028 | RF | Dados da conta Escrow — botão "Copiar dados bancários" | Copia todos os dados formatados para a área de transferência | S6 |
| REQ-109 | D01.2 | RN-028 | Regra | Prazo de depósito — 10 dias úteis | Data-limite formatada (ex: "até 05/04/2026") + contador de dias restantes | S6 |
| REQ-110 | D01.2 | RN-028 | RF | Upload de comprovante — JPG, PNG, PDF, máx 10MB | Com preview antes do envio | S6 |
| REQ-111 | D01.2 | RN-028 | RF | Cancelamento automático por falta de depósito | Após 10 dias úteis sem comprovante (e sem extensão): cancela negociação, retorna oportunidade ao marketplace | S6 |
| REQ-112 | D01.2 | RN-029 | RF | Extensão de prazo Escrow — única por negociação | +5 dias úteis a partir do último dia do prazo original | S6 |
| REQ-113 | D01.2 | RN-029 | RF | Extensão aprovada — NOT-CES-16 | Canal: e-mail + push | S6 |
| REQ-114 | D01.2 | RN-029 | RF | Extensão negada — NOT-CES-17 | Canal: e-mail + push + SMS | S6 |
| REQ-115 | D01.2 | RN-030 | RF | Confirmação de Escrow pelo Admin — status muda para DEPOSITO_CONFIRMADO | NOT-CES-07 ao Cessionário | S6 |
| REQ-116 | D01.2 | RN-031 | Notificação | NOT-CES-05 — 3 dias restantes para Escrow | Canal: e-mail + push + SMS | S6 |
| REQ-117 | D01.2 | RN-031 | Notificação | NOT-CES-06 — último dia para Escrow | Canal: e-mail + push + SMS | S6 |
| REQ-118 | D01.2 | RN-032 | RF | Pré-requisitos para formalização — 3 condições | Escrow confirmado + documentação do imóvel validada + negociação com status DEPOSITO_CONFIRMADO | S7 |
| REQ-119 | D01.2 | RN-033 | RF | Documentos obrigatórios gerados para formalização | 1. Instrumento Particular de Cessão de Direitos e Obrigações · 2. Termo de Ciência e Anuência · 3. Declaração de Origem de Recursos · 4. Comprovante de Depósito em Escrow | S7 |
| REQ-120 | D01.2 | RN-033 | RF | Visualizador embutido de documentos com download PDF | Disponível antes de assinar | S7 |
| REQ-121 | D01.2 | RN-033 | RF | Botão "Reportar divergência" — pausa fluxo de assinatura | Abre chat com contexto pré-carregado indicando documento com problema | S7 |
| REQ-122 | D01.2 | RN-034 | RF | Fluxo de assinatura via ZapSign — 5 etapas | (1) Cessionário assina → (2) status ASSINATURA_PENDENTE_CEDENTE → (3) Cedente assina → (4) AGUARDANDO_ANUENCIA → (5) anuência → CONCLUIDA | S7 |
| REQ-123 | D01.2 | RN-034 | RF | Tela de transição antes de redirecionar ao ZapSign | "Você será redirecionado para o ambiente seguro de assinatura digital (ZapSign)." com botão "Continuar para assinatura" | S7 |
| REQ-124 | D01.2 | RN-034 | RF | Confirmação pós-assinatura | "Assinatura registrada com sucesso." ao retornar do ZapSign | S7 |
| REQ-125 | D01.2 | §6 | Estado | Formalização: Inexistente → Documentos Disponíveis | 3 pré-requisitos atendidos | S7 |
| REQ-126 | D01.2 | §6 | Estado | Formalização: Documentos Disponíveis → Assinatura Pendente Cessionário | Documentos disponibilizados | S7 |
| REQ-127 | D01.2 | §6 | Estado | Formalização: Assinatura Pendente Cessionário → Assinatura Pendente Cedente | Cessionário assina | S7 |
| REQ-128 | D01.2 | §6 | Estado | Formalização: Assinatura Pendente Cedente → Aguardando Anuência | Cedente assina | S7 |
| REQ-129 | D01.2 | §6 | Estado | Formalização: Aguardando Anuência → Concluída | Anuência obtida | S7 |
| REQ-130 | D01.2 | §6 | Estado | Formalização: Aguardando Anuência → Cancelada | Anuência negada | S7 |
| REQ-131 | D01.2 | §6 | Estado | Formalização: qualquer estado pendente → Cancelada | Cancelamento por qualquer parte | S7 |
| REQ-132 | D01.2 | §7 | Estado | Escrow: Aguardando Depósito → Depósito Enviado | Upload de comprovante pelo Cessionário | S6 |
| REQ-133 | D01.2 | §7 | Estado | Escrow: Depósito Enviado → Depósito Confirmado | Admin confirma | S6 |
| REQ-134 | D01.2 | §7 | Estado | Escrow: Depósito Confirmado → Liberado ao Cedente | 4 critérios de fechamento atendidos | S8 |
| REQ-135 | D01.2 | §7 | Estado | Escrow: qualquer → Reembolsado | Cancelamento, reversão aprovada ou falha na formalização | S8 |
| REQ-136 | D01.2 | RN-035 | Regra | Comissão Comprador — fórmula com Δ > 0 | Comissão = 20% × (Tabela Atual − Tabela Contrato). Exibida com fórmula preenchida e tooltip | S5 |
| REQ-137 | D01.2 | RN-036 | Regra | Comissão Comprador — exceção Δ ≤ 0 | Comissão = 20% × Valor Pago pelo Cedente. Aplicado automaticamente | S5 |
| REQ-138 | D01.2 | RN-037 | Regra | Política de desconto — pisos por faixa | Δ até R$100k: 20% mín · Δ R$100.001–R$300k: 17% mín · Δ > R$300k: 13% mín. Cessionário não vê a política | S5 |
| REQ-139 | D01.2 | RN-038 | RF | Painel Financeiro — 4 seções | Depósitos em Escrow + Comissões + Extrato + Comprovantes | S8 |
| REQ-140 | D01.2 | RN-039 | RF | Critérios de fechamento — 4 simultâneos | instrumento assinado + preço confirmado + anuência + Escrow confirmado | S8 |
| REQ-141 | D01.2 | RN-039 | RF | Checklist visual dos 4 critérios | check verde (concluído) / relógio amarelo (pendente) para cada critério | S8 |
| REQ-142 | D01.2 | RN-039 | RF | Período de segurança — 15 dias corridos | Valores retidos na Conta Escrow após fechamento; distribuição automática após 15 dias se sem reversão | S8 |
| REQ-143 | D01.2 | RN-039 | Notificação | NOT-CES-10 — Operação concluída | Canal: e-mail + push + SMS | S8 |
| REQ-144 | D01.2 | RN-040 | RF | Reversão — prazo de 15 dias corridos | Se fora do prazo: "O prazo para solicitação de reversão encerrou em {data}." | S8 |
| REQ-145 | D01.2 | RN-040 | RF | Reversão — campo de motivo obrigatório | Texto livre para fins de registro | S8 |
| REQ-146 | D01.2 | RN-041 | RF | Reversão aprovada — reembolso em até 5 dias úteis | NOT-CES-11 + NOT-CES-12 | S8 |
| REQ-147 | D01.2 | §8 | Estado | Operação: Em Formalização → Operação Concluída | 4 critérios de fechamento | S8 |
| REQ-148 | D01.2 | §8 | Estado | Operação: Operação Concluída → Em Reversão | Solicitação de reversão dentro de 15 dias corridos | S8 |
| REQ-149 | D01.2 | §8 | Estado | Operação: Em Reversão → Reembolso Processado | Reversão aprovada | S8 |
| REQ-150 | D01.2 | §8 | Estado | Operação: Em Reversão → Operação Concluída | Reversão reprovada | S8 |
| REQ-151 | D01.3 | §3 | RF | Dashboard — widget Propostas Ativas | Contagem de ENVIADA + EM_ANALISE; clique redireciona para Minhas Propostas com filtro | S3 |
| REQ-152 | D01.3 | RN-043 | RF | Dashboard — widget Negociações em Andamento | Contagem + indicador de dias restantes; amarelo = 3 dias, vermelho = último dia | S3 |
| REQ-153 | D01.3 | RN-044 | RF | Dashboard — widget Valor em Escrow | Somatório formatado em R$; zero se nenhum; clique → Financeiro com filtro Escrow | S3 |
| REQ-154 | D01.3 | RN-045 | RF | Dashboard — widget Alertas e Notificações | Últimas 5 não lidas; ícone por tipo (prazo=relógio, proposta=documento, financeiro=cifrão); timestamp relativo; lidas com opacidade reduzida | S3 |
| REQ-155 | D01.3 | RN-046 | RF | Dashboard — widget Oportunidades em Destaque | Top 3 recomendadas pela IA; fallback para 3 mais recentes sem histórico; dados anonimizados | S3 |
| REQ-156 | D01.3 | RN-046 | SLA | Dashboard — atualização em até 60 segundos | Após evento (proposta aceita, Escrow confirmado) via polling ou event-driven | S3 |
| REQ-157 | D01.3 | RN-047 | Regra | Analista de Oportunidades — tom obrigatório | Analítico, objetivo, orientado a dados. Proibido: urgência artificial, FOMO, superlativos de venda | S9 |
| REQ-158 | D01.3 | RN-047 | RF | Filtro de output da IA | Linguagem de urgência/FOMO filtrada antes de exibir ao Cessionário | S9 |
| REQ-159 | D01.3 | RN-048 | RF | Capacidades do Analista — 6 permitidas | análise de risco, comparativo de até 5 oportunidades, ROI projetado, score de risco, sugestão de oportunidades, regras da plataforma | S9 |
| REQ-160 | D01.3 | RN-048 | RF | Capacidades do Analista — 5 proibidas | dados do Cedente, cenário A/B/C/D, submeter propostas, dados de outros Cessionários, garantia de resultados | S9 |
| REQ-161 | D01.3 | RN-048 | SLA | SLA de resposta IA — análise individual | Até 5 segundos | S9 |
| REQ-162 | D01.3 | RN-048 | SLA | SLA de resposta IA — comparativo de até 5 | Até 10 segundos | S9 |
| REQ-163 | D01.3 | RN-049 | Regra | Isolamento de dados do Analista | Opera exclusivamente com dados do Cessionário logado. Consultas filtradas por cessionario_id | S9 |
| REQ-164 | D01.3 | RN-050 | RF | Indicador de digitação — 3 pontos pulsantes | Exibido durante processamento da IA | S9 |
| REQ-165 | D01.3 | RN-050 | RF | Timeout IA além do SLA | MSG: "O Analista está processando sua consulta. Isso está demorando um pouco mais que o usual — aguarde mais alguns instantes." | S9 |
| REQ-166 | D01.3 | RN-050 | RF | Botão "Tentar novamente" após 30 segundos | Aparece se ultrapassar 30s | S9 |
| REQ-167 | D01.3 | RN-051 | RF | Histórico de chat com o Analista | Sessão atual e anteriores exibidas; isolado por Cessionário | S9 |
| REQ-168 | D01.3 | RN-051 | RF | Empty state IA — primeiro acesso | MSG: "Olá! Sou o Analista de Oportunidades. Posso analisar riscos, comparar imóveis e simular retornos." CTAs: "Analise a OPR-X" · "Compare as top 3" | S9 |
| REQ-169 | D01.3 | RN-051 | RF | Contexto pré-carregado ao consultar Analista a partir de oportunidade | Botão "Consultar Analista" na tela de detalhe abre chat com oportunidade pré-carregada | S9 |
| REQ-170 | D01.3 | RN-052 | RF | Empty states — 8 telas mapeadas | Dashboard (novo usuário), Oportunidades (sem resultados), Minhas Propostas, Negociações, Assinaturas, Financeiro, Assistente IA, Alertas Dashboard | CROSS |
| REQ-171 | D01.3 | RN-052 | RF | Empty state — estrutura obrigatória | Ilustração contextual + mensagem (máx 2 frases) + CTA | CROSS |
| REQ-172 | D01.3 | RN-052 | RF | Empty states mobile — CTAs full-width | Botões de largura total empilhados verticalmente em mobile | S10 |
| REQ-173 | D01.3 | RN-052 | RF | Empty states — acessibilidade | Alt descritivo em ilustrações + navegação por Tab/Enter + contraste mínimo 4.5:1 | CROSS |
| REQ-174 | D01.4 | RN-053 | SLA | Análise de proposta — SLA de 48h úteis | Alerta interno ao Admin após 48h. Expiração automática em 72h úteis | S4 |
| REQ-175 | D01.4 | RN-054 | SLA | Validação automática KYC — até 5 minutos | Automático via idwall | S2 |
| REQ-176 | D01.4 | RN-054 | SLA | Validação manual KYC — até 24 horas úteis | Caso inconclusivo encaminhado para Admin | S2 |
| REQ-177 | D01.4 | RN-055 | SLA | Confirmação de depósito Escrow — até 24h úteis | Após identificação bancária pelo Admin | S6 |
| REQ-178 | D01.4 | RN-056 | SLA | Geração de documentos de formalização — até 2h úteis | Após confirmação do Escrow | S7 |
| REQ-179 | D01.4 | RN-057 | SLA | Liberação do Escrow pós-fechamento — até 48h úteis | Após fechamento | S8 |
| REQ-180 | D01.4 | RN-058 | SLA | Processamento de reembolso — até 5 dias úteis | Após aprovação de reversão | S8 |
| REQ-181 | D01.4 | RN-059 | SLA | Resposta a mensagem na negociação — até 4h úteis | Alerta interno ao Admin após 4h | S5 |
| REQ-182 | D01.4 | RN-060 | RF | Mapeamento de estados visíveis ao Cessionário | 19 estados internos mapeados para 14 estados visíveis (tabela RN-060) | CROSS |
| REQ-183 | D01.4 | RN-060 | RF | Timestamp da última atualização de estado | Exibido quando estado interno não tem estado visível mapeado | CROSS |
| REQ-184 | D01.4 | RN-061 | RF | Disparo automático de notificações por evento | Todos os canais habilitados pelo Cessionário. E-mail obrigatório mesmo que outros estejam desabilitados | CROSS |
| REQ-185 | D01.4 | RN-061 | RF | Feedback visual in-app — badge numérico + widget Dashboard | Badge decrementado quando notificação visualizada | CROSS |
| REQ-186 | D01.4 | RN-062 | Notificação | NOT-CES-01 — KYC aprovado | Canal: e-mail + push | S2 |
| REQ-187 | D01.4 | RN-062 | Notificação | NOT-CES-02 — KYC reprovado com motivo | Canal: e-mail + push | S2 |
| REQ-188 | D01.4 | RN-062 | Notificação | NOT-CES-03 — proposta aceita | Canal: e-mail + push + SMS | S5 |
| REQ-189 | D01.4 | RN-062 | Notificação | NOT-CES-04 — proposta recusada com motivo | Canal: e-mail + push | S4 |
| REQ-190 | D01.4 | RN-062 | Notificação | NOT-CES-07 — depósito confirmado | Canal: e-mail + push | S6 |
| REQ-191 | D01.4 | RN-062 | Notificação | NOT-CES-08 — documentos prontos | Canal: e-mail + push | S7 |
| REQ-192 | D01.4 | RN-062 | Notificação | NOT-CES-09 — anuência obtida | Canal: e-mail + push | S7 |
| REQ-193 | D01.4 | RN-062 | Notificação | NOT-CES-11 — reversão aprovada | Canal: e-mail + push + SMS | S8 |
| REQ-194 | D01.4 | RN-062 | Notificação | NOT-CES-12 — reembolso processado | Canal: e-mail + push | S8 |
| REQ-195 | D01.4 | RN-062 | Notificação | NOT-CES-13 — nova oportunidade recomendada | Canal: push | S9 |
| REQ-196 | D01.4 | RN-062 | Notificação | NOT-CES-15 — proposta expirada | Canal: e-mail + push | S4 |
| REQ-197 | D01.5 | RN-063 | RF | Integração ZapSign — envio automático de documentos | Retry automático 3x em 30 min; falha oculta ao Cessionário | S7 |
| REQ-198 | D01.5 | RN-063 | RF | ZapSign — webhook de retorno ao assinar | Status avança automaticamente após callback | S7 |
| REQ-199 | D01.5 | RN-064 | RF | Integração Celcoin — confirmação manual no MVP | Confirmação pelo Admin via comprovante | S6 |
| REQ-200 | D01.5 | RN-065 | RF | Integração idwall — OCR + liveness check | Fallback automático para revisão manual Admin se serviço indisponível | S2 |
| REQ-201 | D01.5 | RN-066 | RF | Integração Resend — e-mail transacional | Retry em até 5 min; log de entrega; e-mail nunca desabilitável | S1 |
| REQ-202 | D01.5 | RN-066 | RF | Notificação in-app como fallback universal | Badge e widget sempre exibidos independente de canais externos | CROSS |
| REQ-203 | D01.5 | RN-067 | Regra | Anonimato absoluto — 7 contextos | marketplace, detalhe de oportunidade, negociação, documentos de formalização, respostas IA, notificações, exportações LGPD | CROSS |
| REQ-204 | D01.5 | RN-068 | Regra | Isolamento por cessionario_id — 6 tipos de dados | propostas, negociações, documentos, Escrow, chat Admin, sessões IA | CROSS |
| REQ-205 | D01.5 | RN-069 | Regra | E-mail como canal mínimo obrigatório — transversal | Garante NOT-CES-05/06/17 sempre chegam | CROSS |
| REQ-206 | D01.5 | RN-070 | Regra | Re-autenticação transversal — 4 ações | Envio comprovante Escrow + assinatura documentos + alteração dados bancários + solicitação reversão | CROSS |
| REQ-207 | D01.5 | RN-071 | Regra | Nunca exibir valores negativos | Fallback R$ 0,00 com registro para auditoria interna | CROSS |
| REQ-208 | D01.5 | RN-072 | Regra | Oportunidades sempre com código anônimo | OPR-XXXX-NNNN em marketplace, Dashboard, notificações, recomendações IA | CROSS |
| REQ-209 | D02 | §2.1 | Stack | Backend: Node.js 22.x + NestJS 10.x + TypeScript 5.4 strict | Único runtime/framework aprovado | S1 |
| REQ-210 | D02 | §2.1 | Stack | ORM: Prisma 6.x | TypeORM, Sequelize, Drizzle proibidos | S1 |
| REQ-211 | D02 | §2.1 | Stack | Banco: PostgreSQL 17 via Supabase | MySQL, NoSQL proibidos sem ADR | S1 |
| REQ-212 | D02 | §2.1 | Stack | Cache: Redis 7.4+ (Docker local, Upstash produção) | TTL obrigatório em todo uso; estratégia de invalidação documentada | S1 |
| REQ-213 | D02 | §2.1 | Stack | Filas: RabbitMQ 4.x+ (Docker local, CloudAMQP produção) | Toda fila com retry + DLQ | S1 |
| REQ-214 | D02 | §2.3 | Stack | IA: GPT-4 Turbo (gpt-4-turbo-2024-04-09) + GPT-4o-mini (gpt-4o-mini-2024-07-18) | Versão fixada em produção | S9 |
| REQ-215 | D02 | §2.3 | Stack | IA: LangChain.js 0.3+ + Vercel AI SDK 4+ + Langfuse JS SDK 3+ | Pipeline RAG + streaming SSE + observabilidade | S9 |
| REQ-216 | D02 | §2.3 | Stack | IA: text-embedding-3-small (1536 dim) + pgvector HNSW | Embeddings para RAG do Analista | S9 |
| REQ-217 | D02 | §2.3 | Regra | Rate limiting LLM por usuário | 20–60 chamadas/minuto (ADR-001) | S9 |
| REQ-218 | D02 | §2.3 | Regra | Custo diário OpenAI — alerta US$50, hard limit US$200 | Configurado no Langfuse | S9 |
| REQ-219 | D02 | §3.1 | Stack | Frontend: React 19 + Vite 7 (SPA, 100% logado) | Next.js não se aplica | S1 |
| REQ-220 | D02 | §3.1 | Stack | Frontend routing: TanStack Router 1.x+ | react-router-dom proibido sem ADR | S1 |
| REQ-221 | D02 | §3.1 | Stack | Frontend data fetching: TanStack Query 5.x+ | Nunca usar fetch direto em useEffect | S1 |
| REQ-222 | D02 | §3.1 | Stack | Frontend state: Zustand 5.x+ | Redux, MobX, Context API proibidos | S1 |
| REQ-223 | D02 | §3.1 | Stack | Frontend UI: Tailwind CSS 4.x + shadcn/ui + Radix Primitives | MUI, Chakra, AntD, Bootstrap proibidos | S1 |
| REQ-224 | D02 | §3.1 | Stack | Frontend animações: Framer Motion 12.x+ (Motion) | AnimatePresence em toda troca de rota; skeleton screens; spinners genéricos proibidos | S1 |
| REQ-225 | D02 | §3.2 | Stack | Supabase Realtime — 6 funcionalidades | novas oportunidades, status proposta, chat negociação, status negociação/Escrow, notificações in-app, — (SSE para IA) | S1 |
| REQ-226 | D02 | §4.1 | Stack | Mobile: React Native 0.76+ + Expo SDK 52 + expo-router 4 | iOS + Android | S10 |
| REQ-227 | D02 | §4.1 | Stack | Mobile animações: Reanimated | Gestos nativos | S10 |
| REQ-228 | D02 | §4.2 | Stack | Segurança: CSRF obrigatório em endpoints mutáveis | X-CSRF-Token header | S1 |
| REQ-229 | D02 | §4.2 | Stack | Segurança: RLS habilitado em todas as tabelas com dados de usuários | service_role nunca exposta no frontend | S1 |
| REQ-230 | D02 | §5 | Integração | Integrações externas — 5 serviços | ZapSign (webhook, retry 3x/30min) · idwall (OCR+liveness, fallback manual) · Resend (e-mail, retry 5min) · Expo Notifications (push mobile) · Celcoin (Escrow, manual MVP/automático v2) | S1 |
| REQ-231 | D12 | §2.1 | Banco | Tabela `users` | id UUID PK gen_random_uuid() · email VARCHAR(255) NOT NULL UNIQUE · name VARCHAR(255) NOT NULL · provider VARCHAR(50) DEFAULT 'email' · email_verified_at TIMESTAMPTZ NULL · created_at/updated_at TIMESTAMPTZ NOT NULL · deleted_at TIMESTAMPTZ NULL | S1 |
| REQ-232 | D12 | §2.2 | Banco | Tabela `cessionarios` | id UUID PK · user_id UUID FK UNIQUE · status ENUM (CADASTRADA/KYC_EM_ANALISE/KYC_APROVADO/KYC_REPROVADO/BLOQUEADA_TEMPORARIAMENTE/ENCERRADA) NOT NULL · phone VARCHAR(20) · phone_verified_at TIMESTAMPTZ · bank_account JSONB NULL · bank_account_verified_at TIMESTAMPTZ · notification_preferences JSONB DEFAULT {"email":true,"push":true,"sms":true} · ai_consent BOOLEAN DEFAULT true · ai_consent_at TIMESTAMPTZ · investment_preferences JSONB · created_at/updated_at TIMESTAMPTZ NOT NULL | S1 |
| REQ-233 | D12 | §2.2 | Índice | Índices `cessionarios` | idx_cessionarios_user_id ON user_id · idx_cessionarios_status ON status | S1 |
| REQ-234 | D12 | §2.3 | Banco | Tabela `kyc_documents` | id UUID PK · cessionario_id FK · status ENUM (PENDENTE/EM_ANALISE/APROVADO/REPROVADO) · identity_doc_front_url/back_url/address_doc_url/selfie_url VARCHAR NULL · identity_status/address_status/selfie_status ENUM PENDENTE · rejection_reason TEXT NULL · idwall_session_id VARCHAR NULL · attempt_count INT DEFAULT 0 · blocked_until TIMESTAMPTZ NULL · reviewer_id UUID NULL · reviewed_at TIMESTAMPTZ NULL · created_at/updated_at TIMESTAMPTZ | S1 |
| REQ-235 | D12 | §2.4 | Banco | Tabela `opportunities` | id UUID PK · code VARCHAR(20) UNIQUE · status ENUM (DISPONIVEL/COM_PROPOSTA/EM_NEGOCIACAO/RESERVADA/CONCLUIDA/CANCELADA) · city/neighborhood/state/development_name NOT NULL · bedrooms INT · area_sqm DECIMAL(8,2) · has_garage BOOLEAN · current_table_value/contract_table_value DECIMAL(15,2) · delta_value GENERATED ALWAYS · cedente_paid_percentage DECIMAL(5,2) · cedente_paid_value DECIMAL(15,2) · ai_risk_score DECIMAL(4,2) NULL · embedding vector(1536) NULL · appreciation_data JSONB NULL · published_at TIMESTAMPTZ NULL · created_at/updated_at · deleted_at NULL | S1 |
| REQ-236 | D12 | §2.4 | Índice | Índices `opportunities` | idx_opportunities_status · idx_opportunities_city_state · idx_opportunities_embedding USING hnsw (vector_cosine_ops) | S1 |
| REQ-237 | D12 | §2.5 | Banco | Tabela `proposals` | id UUID PK · cessionario_id FK · opportunity_id FK · proposed_value DECIMAL(15,2) NOT NULL > 0 · commission_buyer DECIMAL(15,2) NOT NULL · message TEXT NULL (max 500 chars) · status ENUM (ENVIADA/EM_ANALISE/ACEITA/RECUSADA/EXPIRADA/CANCELADA) · rejection_reason TEXT NULL · expires_at TIMESTAMPTZ NOT NULL · created_at/updated_at · deleted_at NULL | S1 |
| REQ-238 | D12 | §2.5 | Índice | Índices `proposals` | idx_proposals_cessionario_id · idx_proposals_opportunity_id · idx_proposals_status · idx_proposals_cessionario_status (cessionario_id, status) | S1 |
| REQ-239 | D12 | §2.6 | Banco | Tabela `negotiations` | id UUID PK · proposal_id FK UNIQUE · cessionario_id FK · opportunity_id FK · agreed_value DECIMAL(15,2) NULL · commission_buyer DECIMAL(15,2) NULL · total_escrow_value DECIMAL(15,2) NULL · status ENUM (EM_NEGOCIACAO/EM_CONTRAPROPOSTA/AGUARDANDO_DEPOSITO/DEPOSITO_CONFIRMADO/ENCERRADA/CANCELADA) · counteroffer_rounds INT DEFAULT 0 MAX 3 · escrow_deadline TIMESTAMPTZ NULL · extension_used BOOLEAN DEFAULT false · extended_deadline TIMESTAMPTZ NULL · created_at/updated_at | S1 |
| REQ-240 | D12 | §2.7 | Banco | Tabela `escrow_deposits` | id UUID PK · negotiation_id FK · cessionario_id FK · amount DECIMAL(15,2) NOT NULL · status ENUM (AGUARDANDO_DEPOSITO/DEPOSITO_ENVIADO/DEPOSITO_CONFIRMADO/REEMBOLSADO) · receipt_url VARCHAR NULL · submitted_at/confirmed_at TIMESTAMPTZ NULL · confirmed_by UUID NULL · created_at/updated_at | S1 |
| REQ-241 | D12 | §2.8 | Banco | Tabela `formalizations` | id UUID PK · negotiation_id FK UNIQUE · cessionario_id FK · status ENUM (DOCUMENTOS_DISPONIVEIS/ASSINATURA_PENDENTE_CESSIONARIO/ASSINATURA_PENDENTE_CEDENTE/AGUARDANDO_ANUENCIA/CONCLUIDA/CANCELADA) · zapsign_session_id VARCHAR NULL · zapsign_buyer_sign_url VARCHAR NULL · document_url VARCHAR NULL · buyer_signed_at/seller_signed_at/anuencia_obtained_at TIMESTAMPTZ NULL · created_at/updated_at | S1 |
| REQ-242 | D12 | §2.9 | Banco | Tabela `financial_transactions` | id UUID PK · cessionario_id FK · negotiation_id FK · type VARCHAR (DEPOSITO_ESCROW/COMISSAO/REEMBOLSO) · amount DECIMAL(15,2) · status VARCHAR · description TEXT · created_at/updated_at | S1 |
| REQ-243 | D12 | §2.10 | Banco | Tabela `notifications` | id UUID PK · cessionario_id FK · type VARCHAR · title/body VARCHAR · metadata JSONB · read BOOLEAN DEFAULT false · read_at TIMESTAMPTZ NULL · created_at | S1 |
| REQ-244 | D12 | §2.11 | Banco | Tabela `ai_sessions` | id UUID PK · cessionario_id FK · messages JSONB · context JSONB · created_at/updated_at | S1 |
| REQ-245 | D12 | §2.12 | Banco | Tabela `audit_logs` | id UUID PK · actor_id FK · actor_type VARCHAR · action VARCHAR · entity_type VARCHAR · entity_id UUID · before/after JSONB · created_at | S1 |
| REQ-246 | D16 | §7.1 | Endpoint | POST /auth/login | e-mail + senha → access token (15min prod/30min staging) + refresh token httpOnly cookie (30 dias). Erros: 401 AUTH-001, 403 AUTH-005, 429 AUTH-004 | S2 |
| REQ-247 | D16 | §7.1 | Endpoint | POST /auth/login/google | OAuth Google → access token. Cria conta se nova | S2 |
| REQ-248 | D16 | §7.1 | Endpoint | POST /auth/register | Cadastro com e-mail/senha → conta Cadastrada + e-mail verificação | S2 |
| REQ-249 | D16 | §7.1 | Endpoint | POST /auth/refresh | Renova access token via refresh token cookie. Erro: 401 AUTH-002 | S2 |
| REQ-250 | D16 | §7.1 | Endpoint | DELETE /auth/session | Logout — invalida refresh token | S2 |
| REQ-251 | D16 | §7.1 | Endpoint | POST /auth/password/reset-request | Solicitar reset de senha (público) | S2 |
| REQ-252 | D16 | §7.1 | Endpoint | POST /auth/password/reset | Confirmar nova senha via token URL | S2 |
| REQ-253 | D16 | §7.1 | Endpoint | POST /auth/reauth | Re-autenticação para ação crítica. Response: reauth_token (30s). Erros: 401 AUTH-001, 422 AUTH-006 | S2 |
| REQ-254 | D16 | §7.2 | Endpoint | GET /cessionarios/me | Perfil do Cessionário logado | S3 |
| REQ-255 | D16 | §7.2 | Endpoint | PATCH /cessionarios/me | Atualizar dados do perfil | S3 |
| REQ-256 | D16 | §7.2 | Endpoint | GET /cessionarios/me/kyc | Status do KYC | S2 |
| REQ-257 | D16 | §7.2 | Endpoint | POST /cessionarios/me/kyc/documents | Upload de documento KYC via signed URL. Erros: 400 CES-002/003, 409 CES-004 | S2 |
| REQ-258 | D16 | §7.2 | Endpoint | DELETE /cessionarios/me | Solicitar exclusão de dados (LGPD). Requer Bearer + reauth | S3 |
| REQ-259 | D16 | §7.3 | Endpoint | GET /opportunities | Listar oportunidades. Query params: page, per_page, sort, order, city, state, price_min, price_max, risk_score_min (1-10), status | S4 |
| REQ-260 | D16 | §7.3 | Endpoint | GET /opportunities/{id} | Detalhe de oportunidade | S4 |
| REQ-261 | D16 | §7.3 | Endpoint | GET /opportunities/{id}/analysis | Análise IA — SSE streaming. Erros: 429 AI-001, 503 AI-002 | S9 |
| REQ-262 | D16 | §7.4 | Endpoint | GET /proposals | Listar propostas do Cessionário | S4 |
| REQ-263 | D16 | §7.4 | Endpoint | POST /proposals | Criar proposta. Body: opportunity_id + proposed_value_cents. Erros: 403 CES-001, 409 OPR-001, 422 PRP-001/002/003 | S4 |
| REQ-264 | D16 | §7.4 | Endpoint | GET /proposals/{id} | Detalhe da proposta | S4 |
| REQ-265 | D16 | §7.4 | Endpoint | DELETE /proposals/{id} | Cancelar proposta (somente ENVIADA ou EM_ANALISE). Erros: 409 PRP-004, 404 PRP-005 | S4 |
| REQ-266 | D16 | §7.5 | Endpoint | GET /negotiations | Listar negociações ativas | S5 |
| REQ-267 | D16 | §7.5 | Endpoint | GET /negotiations/{id} | Detalhe da negociação | S5 |
| REQ-268 | D16 | §7.5 | Endpoint | GET /negotiations/{id}/messages | Histórico de mensagens do chat | S5 |
| REQ-269 | D16 | §7.5 | Endpoint | POST /negotiations/{id}/counteroffer | Enviar contraproposta. Body: counteroffer_value_cents + message. Erros: 422 NEG-001/003, 409 NEG-002 | S5 |
| REQ-270 | D16 | §7.5 | Endpoint | POST /negotiations/{id}/escrow/confirm | Confirmar envio do depósito. Body: payment_method + reference_code. Erros: 409 NEG-004, 422 NEG-005 | S6 |
| REQ-271 | D16 | §7.5 | Endpoint | POST /negotiations/{id}/escrow/extend | Solicitar extensão de prazo Escrow | S6 |
| REQ-272 | D16 | §7.6 | Endpoint | GET /formalizations/{id} | Detalhe da formalização | S7 |
| REQ-273 | D16 | §7.6 | Endpoint | GET /formalizations/{id}/documents | Listar documentos disponíveis | S7 |
| REQ-274 | D16 | §7.6 | Endpoint | POST /formalizations/{id}/sign | Iniciar assinatura — gera URL ZapSign. Erros: 409 FRM-001, 503 FRM-002 | S7 |
| REQ-275 | D16 | §7.7 | Endpoint | GET /financial/summary | KPIs financeiros do Cessionário | S8 |
| REQ-276 | D16 | §7.7 | Endpoint | GET /financial/transactions | Histórico de transações. Params: type, date_from, date_to | S8 |
| REQ-277 | D16 | §7.7 | Endpoint | GET /financial/transactions/{id} | Detalhe de transação | S8 |
| REQ-278 | D16 | §7.7 | Endpoint | GET /financial/transactions/{id}/receipt | URL para comprovante PDF | S8 |
| REQ-279 | D16 | §7.8 | Endpoint | GET /notifications | Listar notificações. Param: read (boolean) | S3 |
| REQ-280 | D16 | §7.8 | Endpoint | PATCH /notifications/{id}/read | Marcar notificação como lida | S3 |
| REQ-281 | D16 | §7.8 | Endpoint | PATCH /notifications/read-all | Marcar todas como lidas | S3 |
| REQ-282 | D16 | §7.8 | Endpoint | GET /notifications/preferences | Obter preferências de notificação | S3 |
| REQ-283 | D16 | §7.8 | Endpoint | PATCH /notifications/preferences | Atualizar preferências. email não pode ser false → 422 NOT-001 | S3 |
| REQ-284 | D16 | §7.9 | Endpoint | GET /ai/sessions + GET /ai/sessions/{id}/messages | Listar sessões e mensagens da IA | S9 |
| REQ-285 | D16 | §7.10 | Endpoint | POST /opportunities/{id}/reverse | Solicitar reversão da operação. Requer Bearer + KYC | S8 |

---

## Verificação de Cobertura (Gate Etapa 3)

| Métrica | Valor |
|---|---|
| Total de requisitos no Registro Mestre | 285 |
| Requisitos com sprint atribuída | 285 |
| Requisitos sem sprint | 0 |
| Cobertura | **100%** ✅ |

---

## Distribuição por Sprint

| Sprint | Qtd REQs |
|---|---|
| S1 — Fundação | 37 |
| S2 — Auth e KYC | 46 |
| S3 — Dashboard e Perfil | 18 |
| S4 — Marketplace e Propostas | 34 |
| S5 — Negociações | 18 |
| S6 — Escrow | 14 |
| S7 — Formalização | 14 |
| S8 — Fechamento e Financeiro | 16 |
| S9 — Assistente IA | 18 |
| S10 — Mobile | 3 |
| CROSS (todas as sprints) | 27 |
| **Total** | **285** |
