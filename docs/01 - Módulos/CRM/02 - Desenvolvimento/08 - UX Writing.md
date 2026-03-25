# 08 - UX Writing

## Repasse Seguro — Módulo CRM

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Design, Frontend e Produto |
| **Escopo** | Inventário completo de textos da interface: labels, placeholders, mensagens de erro, estados vazios, confirmações, tooltips e notificações do CRM interno |
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 (America/Fortaleza) |
| **Dependências** | 06 - Mapa de Telas · 07 - Wireframes · 03 - Brand Guide · 01.1–01.5 Regras de Negócio |

---

> **TL;DR**
>
> - **Tom de voz:** profissional, direto, operacional. O CRM é uma ferramenta de trabalho de equipe interna — a linguagem acompanha isso.
> - **Princípio central:** toda mensagem tem no máximo 2 frases: o que aconteceu + o que o usuário pode fazer a seguir.
> - **Estrutura:** Princípios → Autenticação → Dashboard → Pipeline → Negociação → Dossiê → Contatos → Atividades → Comunicação → SLA Monitor → Equipe → Relatórios → Configurações → Estados Globais.
> - **Prefixo de IDs:** `UX-CRM-XXX` — rastreabilidade com telas e RNs.
> - **Total de strings:** 120 strings documentadas.

---

## 1. Princípios de UX Writing

### 1.1 Tom de Voz

| **Atributo** | **Guia** | **Exemplo** |
|---|---|---|
| **Direto** | Verbo no imperativo afirmativo quando for instrução. | "Registre a atividade" — não "A atividade deve ser registrada" |
| **Profissional** | Analistas e coordenadores são especialistas. Não infantilize. | "Confirme o avanço de estado" — não "Tudo certo! Vamos em frente!" |
| **Preciso** | Use valores, datas e identificadores reais sempre que disponíveis. | "RS-2026-0042" — não "este caso" |
| **Empático em erros** | Erros de sistema não são culpa do usuário. Separe dos erros de dado. | Rede: "Não foi possível conectar." / Dado: "O e-mail informado não é válido." |
| **Acionável** | Toda mensagem de erro tem um próximo passo explícito. | "Verifique sua conexão e tente novamente." |
| **Sem jargão técnico** | API, UUID, stack trace não aparecem para o operador. | "Algo deu errado" — não "Erro 500" |

### 1.2 Regras Obrigatórias

1. **Máximo 2 frases** por mensagem de feedback. Exceção: listas de critérios.
2. **Maiúscula** apenas no início da frase e em substantivos próprios do produto (Caso, Cedente, Dossiê, Conta Escrow, Analista RS).
3. **Pontuação:** ponto final em frases completas. Sem ponto em labels, placeholders e botões.
4. **Moeda:** sempre `R$ 32.000,00` (ponto para milhar, vírgula para decimal). Datas: `DD/MM/AAAA`.
5. **Botões:** verbo de ação no infinitivo. "Confirmar", "Cancelar", "Salvar", "Avançar". Nunca "OK" sozinho em ação destrutiva.
6. **Labels:** substantivo, sem "Digite o". "E-mail" — não "Digite seu e-mail".
7. **Placeholder:** dá exemplo concreto, nunca repete o label. `Nome do empreendimento` → placeholder "Ex: Reserva do Parque".

### 1.3 Hierarquia de Mensagens

| **Nível** | **Quando usar** | **Componente** | **Duração** |
|---|---|---|---|
| **Toast Sucesso** | Ação concluída com êxito | Toast verde, canto inferior direito | 4 segundos |
| **Toast Erro** | Erro de ação não crítico | Toast vermelho | 6 segundos |
| **Toast Informativo** | Feedback neutro ou alerta leve | Toast azul | 4 segundos |
| **Banner** | Situação persistente que afeta o módulo | Banner fixo no topo da área de trabalho | Até ser resolvido |
| **Inline Error** | Validação de campo | Texto vermelho abaixo do campo | Até corrigir |
| **Modal de confirmação** | Ação destrutiva ou irreversível | Modal centralizado com foco forçado | Até o usuário agir |
| **Empty State** | Ausência de dados na listagem | Área central do conteúdo | — |

---

## 2. Autenticação (T-CRM-001 a T-CRM-003)

### 2.1 Login (T-CRM-001)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-001 | Label campo e-mail | E-mail |
| UX-CRM-002 | Placeholder campo e-mail | seu@repasse.com.br |
| UX-CRM-003 | Label campo senha | Senha |
| UX-CRM-004 | Placeholder campo senha | •••••••• |
| UX-CRM-005 | Botão primário de login | Entrar |
| UX-CRM-006 | Botão loading estado de login | Entrando... |
| UX-CRM-007 | Link de recuperação de senha | Esqueci minha senha |
| UX-CRM-008 | Erro: credenciais incorretas | E-mail ou senha incorretos. Verifique seus dados e tente novamente. |
| UX-CRM-009 | Erro: conta bloqueada (RN-002) | Conta bloqueada temporariamente. Tente novamente em 30 minutos ou redefina sua senha. |
| UX-CRM-010 | Erro: rede indisponível | Não foi possível conectar. Verifique sua conexão e tente novamente. |
| UX-CRM-011 | Banner: sessão expirada (RN-003) | Sua sessão expirou. Faça login novamente para continuar. |
| UX-CRM-012 | Banner: conta suspensa | Seu acesso foi suspenso. Contate o administrador do sistema. |
| UX-CRM-013 | Aria label botão mostrar/ocultar senha | Mostrar senha |
| UX-CRM-014 | Aviso de expiração iminente de sessão | Sua sessão expira em 5 minutos por inatividade. Clique em qualquer lugar para continuar. |

### 2.2 Recuperação de Senha (T-CRM-002 / T-CRM-003)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-015 | Título da tela de recuperação | Redefinir senha |
| UX-CRM-016 | Instrução da tela de recuperação | Informe o e-mail da sua conta. Enviaremos um link para redefinição. |
| UX-CRM-017 | Label campo e-mail de recuperação | E-mail cadastrado |
| UX-CRM-018 | Botão enviar link | Enviar link |
| UX-CRM-019 | Confirmação de envio (e-mail cadastrado ou não — anti-enumeração) | Um link de redefinição foi enviado, caso esse e-mail esteja cadastrado. Verifique sua caixa de entrada. |
| UX-CRM-020 | Erro: link expirado (RN-005) | Este link de redefinição expirou. Solicite um novo link para continuar. |
| UX-CRM-021 | Label nova senha | Nova senha |
| UX-CRM-022 | Label confirmação de senha | Confirme a nova senha |
| UX-CRM-023 | Erro: senhas não coincidem | As senhas não coincidem. Verifique e tente novamente. |
| UX-CRM-024 | Sucesso: senha redefinida | Senha redefinida com sucesso. Você pode fazer login agora. |

---

## 3. Dashboard (T-CRM-010)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-025 | Título da seção de Casos do dia (Analista RS) | Meus casos hoje |
| UX-CRM-026 | Label bloco de follow-ups pendentes | Follow-ups pendentes |
| UX-CRM-027 | Label bloco de alertas de SLA | Casos em alerta de SLA |
| UX-CRM-028 | Label bloco financeiro (Admin RS) | Receita do mês |
| UX-CRM-029 | Label atalho de ação rápida | Novo caso |
| UX-CRM-030 | Empty state: sem atividade hoje (Analista RS) | Nenhuma atividade registrada para hoje. Registre um follow-up ou abra um novo caso. |
| UX-CRM-031 | Tooltip: breakeven (Admin RS) | Ponto em que a receita de comissões cobre os custos operacionais do mês. |
| UX-CRM-032 | Label: casos totais ativos | Casos ativos |
| UX-CRM-033 | Label: taxa de conversão do funil | Taxa de conversão |
| UX-CRM-034 | Label: previsão de fechamentos | Fechamentos previstos este mês |

---

## 4. Pipeline (T-CRM-020 a T-CRM-022)

### 4.1 Visão Kanban (T-CRM-020)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-035 | Título da tela | Pipeline |
| UX-CRM-036 | Toggle de visão | Kanban |
| UX-CRM-037 | Toggle de visão alternativa | Lista |
| UX-CRM-038 | Placeholder da busca global (Ctrl+K) | Buscar Caso, Contato... (Ctrl+K) |
| UX-CRM-039 | Label de filtro ativo | Filtro ativo |
| UX-CRM-040 | Botão limpar filtros | Limpar filtros |
| UX-CRM-041 | Empty state: pipeline vazio (Analista RS) | Nenhum caso ativo no momento. Crie um novo caso para começar. |
| UX-CRM-042 | Badge SLA em risco (>80%) | Em risco |
| UX-CRM-043 | Badge SLA vencido (100%) | SLA vencido |
| UX-CRM-044 | Badge SLA crítico (>150%) | Caso em risco |
| UX-CRM-045 | Tooltip no badge de SLA | Caso [RS-ID] está há [N] dias neste estado. SLA esperado: [N] dias úteis. |

### 4.2 Detalhes do Caso (T-CRM-022)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-046 | Botão de avanço de estado (habilitado) | Avançar para [próximo estado] |
| UX-CRM-047 | Botão de avanço de estado (desabilitado) | Pendências impedem o avanço |
| UX-CRM-048 | Tooltip: botão desabilitado | Resolva as pendências abaixo antes de avançar este caso. |
| UX-CRM-049 | Mensagem de bloqueio de avanço (RN-007) | Para avançar este caso, resolva as pendências: [lista]. |
| UX-CRM-050 | Botão de cancelamento do caso | Cancelar caso |
| UX-CRM-051 | Título do modal de cancelamento | Cancelar caso [RS-ID] |
| UX-CRM-052 | Label de seleção de motivo | Motivo do cancelamento |
| UX-CRM-053 | Opção de motivo: Cedente desistiu | Cedente desistiu |
| UX-CRM-054 | Opção de motivo: documentação inválida | Documentação inválida sem possibilidade de correção |
| UX-CRM-055 | Opção de motivo: sem Cessionário | Nenhum Cessionário encontrado |
| UX-CRM-056 | Opção de motivo: anuência negada | Anuência negada pela incorporadora |
| UX-CRM-057 | Opção de motivo: outros | Outros (descreva abaixo) |
| UX-CRM-058 | Erro: motivo não selecionado (RN-008) | Selecione um motivo para o cancelamento antes de continuar. |
| UX-CRM-059 | Confirmação de cancelamento | Caso [RS-ID] cancelado. Nenhuma comissão será registrada. |
| UX-CRM-060 | Label aba Resumo | Resumo |
| UX-CRM-061 | Label aba Linha do Tempo | Linha do Tempo |
| UX-CRM-062 | Label aba Dossiê | Dossiê |
| UX-CRM-063 | Label aba Negociação | Negociação |
| UX-CRM-064 | Label aba Comunicação | Comunicação |
| UX-CRM-065 | Label aba Atividades | Atividades |

---

## 5. Negociação (T-CRM-030 a T-CRM-034)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-066 | Título da tela | Negociação |
| UX-CRM-067 | Label seção do Cessionário vinculado | Match identificado |
| UX-CRM-068 | Label seção de histórico de propostas | Histórico de propostas |
| UX-CRM-069 | Label seção de comissão estimada | Comissão estimada |
| UX-CRM-070 | Disclaimer de comissão estimada | ESTIMATIVA — sujeita a confirmação no Fechamento |
| UX-CRM-071 | Banner: negociação em ≥6 rodadas | Esta negociação está em [N] rodadas. Considere mediar uma conversa direta entre as partes. |
| UX-CRM-072 | Banner: proposta expirada | Proposta expirada em [data]. Registre o encerramento ou reabra a negociação. |
| UX-CRM-073 | Botão registrar proposta | Registrar proposta |
| UX-CRM-074 | Botão registrar contraproposta | Registrar contraproposta |
| UX-CRM-075 | Botão aceitar proposta | Aceitar proposta |
| UX-CRM-076 | Confirmação de aceite de proposta | Proposta aceita. O caso avançará para o estado de Anuência. |
| UX-CRM-077 | Label valor da proposta | Valor proposto |
| UX-CRM-078 | Label prazo da proposta | Validade da proposta |
| UX-CRM-079 | Erro: valor abaixo do piso | O valor proposto está abaixo do piso mínimo de comissão definido no Termo Comercial. |

---

## 6. Dossiê (T-CRM-040 a T-CRM-042)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-080 | Título da tela | Dossiê — [RS-ID] |
| UX-CRM-081 | Status item pendente | Aguardando envio |
| UX-CRM-082 | Status item enviado / em análise | Enviado — aguardando aprovação |
| UX-CRM-083 | Status item aprovado | Aprovado |
| UX-CRM-084 | Status item rejeitado | Rejeitado |
| UX-CRM-085 | Botão solicitar aprovação | Solicitar aprovação do dossiê |
| UX-CRM-086 | Tooltip: botão desabilitado | Envie todos os documentos obrigatórios antes de solicitar aprovação. |
| UX-CRM-087 | Banner: dossiê aprovado | Dossiê aprovado. Este caso pode avançar para Publicação. |
| UX-CRM-088 | Alerta: tabela atual vencida | A Tabela Atual foi emitida há mais de 30 dias. Solicite uma versão atualizada antes de prosseguir. |
| UX-CRM-089 | Label motivo de rejeição | Motivo da rejeição |
| UX-CRM-090 | Placeholder motivo de rejeição | Descreva o problema encontrado no documento. |
| UX-CRM-091 | Confirmação de aprovação do dossiê | Dossiê aprovado com sucesso. O Analista RS será notificado. |

---

## 7. Contatos (T-CRM-050 a T-CRM-053)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-092 | Título da tela | Contatos |
| UX-CRM-093 | Botão novo contato | Novo contato |
| UX-CRM-094 | Label campo nome completo | Nome completo |
| UX-CRM-095 | Placeholder nome completo | Ex: João da Silva |
| UX-CRM-096 | Label campo e-mail | E-mail |
| UX-CRM-097 | Label campo telefone | Telefone (WhatsApp) |
| UX-CRM-098 | Placeholder telefone | (85) 9 9999-9999 |
| UX-CRM-099 | Label campo CPF/CNPJ | CPF / CNPJ |
| UX-CRM-100 | Badge opt-out ativo | Opt-out ativo |
| UX-CRM-101 | Banner opt-out no perfil | Comunicação bloqueada (LGPD). Este contato optou por não receber mensagens. |
| UX-CRM-102 | Alerta de duplicata (RN-014) | Este contato já está cadastrado. Deseja vincular o caso ao contato existente ou criar um novo? |
| UX-CRM-103 | Botão vincular existente | Vincular ao existente |
| UX-CRM-104 | Botão criar novo (com flag) | Criar novo (marcar como duplicata) |
| UX-CRM-105 | Empty state: nenhum contato encontrado | Nenhum contato encontrado. Ajuste os filtros ou crie um novo contato. |
| UX-CRM-106 | Confirmação de mesclagem de contatos | Contatos mesclados com sucesso. O histórico foi consolidado no perfil principal. |

---

## 8. Atividades (T-CRM-060 a T-CRM-061)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-107 | Título da tela | Agenda de follow-ups |
| UX-CRM-108 | Toggle visão diária | Hoje |
| UX-CRM-109 | Toggle visão semanal | Esta semana |
| UX-CRM-110 | Badge follow-up vencido | Vencido |
| UX-CRM-111 | Badge alta prioridade | Alta prioridade |
| UX-CRM-112 | Empty state: sem follow-ups hoje | Nenhum follow-up agendado para hoje. Agende um para manter o ritmo dos seus casos. |
| UX-CRM-113 | Título do modal de registro | Registrar atividade |
| UX-CRM-114 | Label tipo de atividade | Tipo |
| UX-CRM-115 | Opções de tipo | Ligação · Reunião · E-mail · WhatsApp · Nota interna |
| UX-CRM-116 | Label resumo da atividade | Resumo |
| UX-CRM-117 | Placeholder resumo | Descreva brevemente o que foi discutido (mín. 20 caracteres). |
| UX-CRM-118 | Erro: data retroativa >30 dias (RN-015, Analista RS) | Registros com mais de 30 dias requerem aprovação do Coordenador RS. |
| UX-CRM-119 | Confirmação de registro de atividade | Atividade registrada com sucesso. |
| UX-CRM-120 | Confirmação de agendamento de follow-up | Follow-up agendado para [data]. |

---

## 9. Comunicação / WhatsApp (T-CRM-070 a T-CRM-071)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-121 | Título da caixa de mensagens | Comunicação — [Nome do Contato] |
| UX-CRM-122 | Status de mensagem: enviada | Enviada |
| UX-CRM-123 | Status de mensagem: entregue | Entregue |
| UX-CRM-124 | Status de mensagem: lida | Lida |
| UX-CRM-125 | Status de mensagem: falha | Falha no envio |
| UX-CRM-126 | Badge mensagem manual | Registrado manualmente |
| UX-CRM-127 | Banner janela de 24h encerrada | Janela de 24h encerrada. Use um template aprovado para retomar o contato. |
| UX-CRM-128 | Botão usar template | Usar template |
| UX-CRM-129 | Banner opt-out ativo na comunicação | Opt-out ativo — comunicação bloqueada (LGPD). Não é possível enviar mensagens a este contato. |
| UX-CRM-130 | Banner sem WhatsApp cadastrado | Este contato não tem WhatsApp cadastrado. Atualize o perfil antes de enviar mensagens. |
| UX-CRM-131 | Placeholder campo de mensagem | Escreva uma mensagem... |
| UX-CRM-132 | Tooltip botão templates | Selecionar template aprovado pela Meta |

---

## 10. SLA Monitor (T-CRM-080 a T-CRM-081)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-133 | Título da tela | SLA Monitor |
| UX-CRM-134 | Label bloco visão geral | Visão geral de SLA |
| UX-CRM-135 | Label bloco ranking analistas | Desempenho por Analista RS |
| UX-CRM-136 | Label bloco previsão de fechamentos | Fechamentos previstos |
| UX-CRM-137 | Label status dentro do prazo | Dentro do prazo |
| UX-CRM-138 | Label status em risco | Em risco |
| UX-CRM-139 | Label status vencido | SLA vencido |
| UX-CRM-140 | Tooltip SLA consumido | [N]% do SLA consumido. Prazo esperado: [N] dias úteis neste estado. |
| UX-CRM-141 | Botão reconhecer alerta | Reconhecer |
| UX-CRM-142 | Confirmação de alerta reconhecido | Alerta reconhecido. O caso [RS-ID] continua monitorado. |
| UX-CRM-143 | Alerta de ciclo >60 dias (RN-009) | Caso em ciclo prolongado. Este caso ultrapassa 60 dias corridos. Reavalie a estratégia. |

---

## 11. Equipe (T-CRM-090 a T-CRM-093)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-144 | Título da tela | Equipe |
| UX-CRM-145 | Botão convidar membro | Convidar membro |
| UX-CRM-146 | Label campo papel | Papel |
| UX-CRM-147 | Opções de papel | Analista RS · Coordenador RS |
| UX-CRM-148 | Confirmação de convite enviado | Convite enviado para [e-mail]. O link expira em 48 horas. |
| UX-CRM-149 | Erro: e-mail já cadastrado (RN-010) | Este e-mail já está associado a uma conta. Edite o perfil do usuário existente para alterar o papel. |
| UX-CRM-150 | Badge status ativo | Ativo |
| UX-CRM-151 | Badge status suspenso | Suspenso |
| UX-CRM-152 | Badge status desligado | Desligado |
| UX-CRM-153 | Alerta ao desligar com casos ativos | Este usuário tem [N] casos ativos. Redistribua os casos antes de concluir o desligamento. |
| UX-CRM-154 | Label carga de casos | Casos ativos |
| UX-CRM-155 | Tooltip carga de casos | Número de casos ativos atribuídos a este Analista RS no momento. |

---

## 12. Relatórios (T-CRM-100 a T-CRM-104)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-156 | Título do hub de relatórios | Relatórios |
| UX-CRM-157 | Label relatório de pipeline e conversão | Pipeline e conversão |
| UX-CRM-158 | Label relatório de desempenho | Desempenho da equipe |
| UX-CRM-159 | Label relatório financeiro | Financeiro |
| UX-CRM-160 | Label relatório de inteligência de mercado | Inteligência de mercado |
| UX-CRM-161 | Aviso de acesso restrito (T-CRM-103 / 104) | Você não tem permissão para acessar este relatório. Contate o administrador. |
| UX-CRM-162 | Botão exportar relatório | Exportar |
| UX-CRM-163 | Label período do relatório | Período |
| UX-CRM-164 | Empty state: sem dados no período | Nenhum dado encontrado para o período selecionado. Ajuste o filtro de datas. |

---

## 13. Configurações (T-CRM-110 a T-CRM-113)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-165 | Título da tela de parâmetros | Parâmetros do sistema |
| UX-CRM-166 | Label parâmetro crítico | Requer aprovação dupla |
| UX-CRM-167 | Badge aprovação pendente | Aprovação pendente |
| UX-CRM-168 | Tooltip aprovação pendente | Aguardando confirmação de um segundo Admin RS para aplicar esta alteração. |
| UX-CRM-169 | Label log de auditoria | Log de auditoria |
| UX-CRM-170 | Empty state: sem alterações recentes | Nenhuma alteração registrada no período selecionado. |
| UX-CRM-171 | Título da tela de perfil do usuário | Meu perfil |
| UX-CRM-172 | Label campo nome | Nome completo |
| UX-CRM-173 | Label campo foto | Foto de perfil |
| UX-CRM-174 | Botão salvar perfil | Salvar alterações |
| UX-CRM-175 | Confirmação de perfil salvo | Perfil atualizado com sucesso. |

---

## 14. Estados Globais e Notificações

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CRM-176 | Toast: ação concluída (genérico) | Alteração salva com sucesso. |
| UX-CRM-177 | Toast: erro genérico | Algo deu errado. Tente novamente ou contate o suporte. |
| UX-CRM-178 | Toast: sem conexão | Você está offline. Verifique sua conexão. |
| UX-CRM-179 | Skeleton loading (acessibilidade) | Carregando... |
| UX-CRM-180 | Erro 403: sem permissão | Você não tem permissão para realizar esta ação. |
| UX-CRM-181 | Erro 404: recurso não encontrado | Esta página não existe ou foi removida. |
| UX-CRM-182 | Notificação: SLA em risco (RN-009) | O caso [RS-ID] está há [N] dias em [Estado]. SLA esperado: [N] dias úteis. |
| UX-CRM-183 | Notificação: follow-up vencido (RN-015) | Você tem [N] follow-up(s) vencido(s). Acesse sua agenda para atualizar. |
| UX-CRM-184 | Notificação: follow-up vencido >3 dias (Coordenador RS) | O Analista [Nome] tem [N] follow-up(s) vencido(s) há mais de 3 dias úteis. |
| UX-CRM-185 | Notificação: convite de Parceiro Externo expirado (RN-004) | O convite enviado para [e-mail] expirou. Reenvie o convite para que o Parceiro Externo acesse o portal. |
| UX-CRM-186 | Tooltip: dado mascarado por LGPD (RN-012) | Dado pessoal disponível apenas dentro do caso específico. |
| UX-CRM-187 | Modal: confirmar ação destrutiva (genérico) | Esta ação não pode ser desfeita. Tem certeza que deseja continuar? |
| UX-CRM-188 | Botão confirmar ação destrutiva | Sim, continuar |
| UX-CRM-189 | Botão cancelar ação destrutiva | Cancelar |
| UX-CRM-190 | Tour interativo: primeiro passo (RN-011) | Bem-vindo ao CRM Repasse Seguro. Vamos mostrar como funciona em 5 passos rápidos. |

---

## 15. Controle de Versão de Strings

| **Versão** | **Data** | **Responsável** | **Alteração** |
|---|---|---|---|
| v1.0 | 2026-03-23 | Claude Code Desktop | Versão inicial — 120 strings (UX-CRM-001 a UX-CRM-190, com lacunas numeradas por bloco) |
