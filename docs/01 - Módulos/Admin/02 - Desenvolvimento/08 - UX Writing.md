# 08 - UX Writing

## Repasse Seguro — Módulo Admin

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Design, Frontend e Produto |
| **Escopo** | Inventário completo de textos da interface: labels, mensagens de erro, feedbacks, estados vazios, confirmações, tooltips e notificações inline |
| **Módulo** | Admin |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-22 (America/Fortaleza) |
| **Dependências** | 06 - Mapa de Telas · 07 - Wireframes · 03 Brand Guide · 01.1–01.5 RN |

---

> 📌 **TL;DR**
>
> - **Tom de voz:** direto, respeitoso, operacional. Sem jargões excessivos, sem linguagem informal exagerada. Operadores internos são profissionais — o sistema fala com eles como tal.
> - **Princípio central:** toda mensagem tem no máximo 2 frases: o que aconteceu + o que o usuário pode fazer.
> - **Estrutura do documento:** Princípios → Autenticação → Dashboard → Pipeline → Triagem → Negociação → Formalização → Financeiro → Supervisão IA → Usuários → Relatórios → Configurações → Estados Globais → Notificações.
> - **IDs de string:** formato `UX-[MÓDULO]-[SEQUENCIAL]` para rastreabilidade.

---

## 1. Princípios de UX Writing

### 1.1 Tom de Voz

| **Atributo** | **Guia** | **Exemplo** |
|---|---|---|
| **Direto** | Verbo no imperativo afirmativo quando for instrução. Evite gerúndio passivo. | "Envie o comprovante" → não "O comprovante deve ser enviado" |
| **Respeitoso** | Operadores são profissionais. Não infantilize. | "Confirme o fechamento" → não "Clique aqui para terminar!" |
| **Preciso** | Valores, datas e nomes reais sempre que disponíveis. Evite "alguns" ou "vários". | "R$ 32.000" → não "um valor" |
| **Empático em erros** | Erros não são culpa do usuário — a menos que sejam. Separe os casos. | Erro de rede: "Não foi possível conectar." Erro de dado: "O e-mail informado não é válido." |
| **Acionável** | Toda mensagem de erro tem próximo passo. | "Verifique sua conexão e tente novamente." |
| **Sem jargão técnico** | API, UUID, stack trace → não aparecem para o operador. | "Algo deu errado" → não "Erro 500 — Internal Server Error" |

### 1.2 Regras Obrigatórias

1. **Máximo 2 frases** por mensagem de feedback. Exceção: lista de critérios (checklist).
2. **Maiúscula** apenas no início da frase e em substantivos próprios (nomes de entidades do produto: Caso, Cedente, Conta Escrow). Evitar ALL CAPS em textos (apenas em badges/labels).
3. **Pontuação:** ponto final em frases completas. Sem ponto em labels, placeholders e títulos de botão.
4. **Números:** moeda sempre formatada (R$ 32.000,00 — ponto para milhar, vírgula para decimal). Datas no formato DD/MM/AAAA.
5. **Verbos de ação em botões:** "Confirmar", "Cancelar", "Salvar", "Enviar". Nunca "OK" sozinho — exceto em modais informativos sem ação destrutiva.
6. **Labels de campo:** substantivo, sem "Digite o" antes. "E-mail", não "Digite seu e-mail".
7. **Placeholder:** dá exemplo, não repete o label. `Nome completo` → placeholder "Ex: João da Silva".

### 1.3 Hierarquia de Mensagens

| **Nível** | **Quando usar** | **Componente** | **Duração** |
|---|---|---|---|
| **Toast Sucesso** | Ação concluída com êxito | Toast verde, canto inferior direito | 4 segundos |
| **Toast Erro** | Erro de ação não-crítico | Toast vermelho | 6 segundos |
| **Toast Informativo** | Feedback neutro ou alerta leve | Toast azul | 4 segundos |
| **Banner** | Situação persistente que afeta todo o módulo | Banner fixo no topo | Até ser resolvido |
| **Inline Error** | Erro de validação de campo | Texto abaixo do campo | Até corrigir |
| **Modal** | Confirmação de ação destrutiva ou irreversível | Modal centralizado | Até usuário agir |
| **Empty State** | Ausência de dados na tela | Área central da listagem | — |

---

## 2. Autenticação (T-001 a T-004)

### 2.1 Login (T-001)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-AUTH-001 | Label campo e-mail | E-mail |
| UX-AUTH-002 | Placeholder e-mail | seu@email.com |
| UX-AUTH-003 | Label campo senha | Senha |
| UX-AUTH-004 | Placeholder senha | ••••••••• |
| UX-AUTH-005 | Botão primário | Entrar |
| UX-AUTH-006 | Link recuperação | Esqueci minha senha |
| UX-AUTH-007 | Botão loading | Entrando... |
| UX-AUTH-008 | Erro credencial inválida | E-mail ou senha incorretos. Verifique os dados e tente novamente. |
| UX-AUTH-009 | Erro bloqueio 5 tentativas | Conta bloqueada temporariamente. Tente novamente em 30 minutos ou redefina sua senha. |
| UX-AUTH-010 | Erro de rede | Não foi possível conectar. Verifique sua conexão e tente novamente. |
| UX-AUTH-011 | Aria label botão show/hide senha | Mostrar senha |

### 2.2 2FA (T-002)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-AUTH-020 | Título da tela | Verificação em duas etapas |
| UX-AUTH-021 | Instrução | Digite o código de 6 dígitos do seu aplicativo autenticador. |
| UX-AUTH-022 | Botão verificar | Verificar |
| UX-AUTH-023 | Botão loading | Verificando... |
| UX-AUTH-024 | Erro código inválido | Código incorreto. Verifique seu aplicativo autenticador. |
| UX-AUTH-025 | Erro código expirado | Código expirado. Aguarde o próximo código (30 segundos). |
| UX-AUTH-026 | Link código de recuperação | Usar código de recuperação |

### 2.3 Recuperação de Senha (T-003)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-AUTH-030 | Título da tela | Recuperar senha |
| UX-AUTH-031 | Instrução | Informe seu e-mail cadastrado. Enviaremos as instruções de redefinição. |
| UX-AUTH-032 | Label campo e-mail | E-mail cadastrado |
| UX-AUTH-033 | Botão | Enviar instruções |
| UX-AUTH-034 | Feedback sucesso | Instruções enviadas para [email]. Verifique sua caixa de entrada e a pasta de spam. |
| UX-AUTH-035 | Feedback e-mail não encontrado (mensagem segura) | Se este e-mail estiver cadastrado, você receberá as instruções em breve. |
| UX-AUTH-036 | Link retorno | Voltar ao login |

### 2.4 Redefinição de Senha (T-004)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-AUTH-040 | Título da tela | Redefinir senha |
| UX-AUTH-041 | Label nova senha | Nova senha |
| UX-AUTH-042 | Label confirmar senha | Confirmar nova senha |
| UX-AUTH-043 | Critério: comprimento | 8 ou mais caracteres |
| UX-AUTH-044 | Critério: maiúscula | 1 letra maiúscula |
| UX-AUTH-045 | Critério: minúscula | 1 letra minúscula |
| UX-AUTH-046 | Critério: número | 1 número |
| UX-AUTH-047 | Critério: especial | 1 caractere especial (!@#$%...) |
| UX-AUTH-048 | Erro senhas não coincidem | As senhas informadas não coincidem. |
| UX-AUTH-049 | Erro senha fraca (submit) | Sua nova senha precisa atender a todos os critérios acima. |
| UX-AUTH-050 | Erro token expirado | O link de redefinição expirou. Solicite um novo link. |
| UX-AUTH-051 | Botão | Redefinir senha |
| UX-AUTH-052 | Sucesso | Senha redefinida com sucesso. Você será redirecionado para o login. |
| UX-AUTH-053 | Link token expirado | Solicitar novo link |

---

## 3. Dashboard (T-010)

### 3.1 KPIs e Zonas

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-DASH-001 | Título KPI Analista: casos ativos | Meus casos ativos |
| UX-DASH-002 | Título KPI Analista: SLA | SLAs vencendo hoje |
| UX-DASH-003 | Título KPI Coordenador: total | Casos ativos |
| UX-DASH-004 | Título KPI Coordenador: vencidos | SLAs vencidos |
| UX-DASH-005 | Título KPI Financeiro: escrows | Escrows pendentes |
| UX-DASH-006 | Título KPI Financeiro: custódia | Valor em custódia |
| UX-DASH-007 | Título gráfico funil | Funil de casos |
| UX-DASH-008 | Título ranking | Desempenho da equipe |
| UX-DASH-009 | Título atividades pendentes | Atividades pendentes |
| UX-DASH-010 | Título alertas SLA | Alertas de prazo |

### 3.2 Estados do Dashboard

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-DASH-020 | Empty state — primeiro acesso | Bem-vindo ao Repasse Seguro Admin. Nenhum caso cadastrado ainda. |
| UX-DASH-021 | CTA empty state | Ver Pipeline |
| UX-DASH-022 | Erro parcial de zona | Não foi possível carregar esta seção. Tente novamente. |
| UX-DASH-023 | Botão retry erro parcial | Tentar novamente |
| UX-DASH-024 | Banner offline | Dados podem estar desatualizados — sem conexão. |
| UX-DASH-025 | Banner SLA crítico | [X] caso(s) com prazo vencido. |
| UX-DASH-026 | Link banner SLA | Ver casos |
| UX-DASH-027 | Aria label KPI | [Nome do KPI]: [valor]. [Variação, se disponível] |
| UX-DASH-028 | Label período — Hoje | Hoje |
| UX-DASH-029 | Label período — Esta semana | Esta semana |
| UX-DASH-030 | Label período — Este mês | Este mês |
| UX-DASH-031 | Label período — Customizado | Período personalizado |
| UX-DASH-032 | Timestamp última atualização | Atualizado às [HH:MM] |

---

## 4. Pipeline (T-020 a T-022)

### 4.1 Kanban e Lista

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-PIPE-001 | Título da seção | Pipeline |
| UX-PIPE-002 | Toggle kanban | Kanban |
| UX-PIPE-003 | Toggle lista | Lista |
| UX-PIPE-004 | Placeholder filtro analista | Filtrar por analista |
| UX-PIPE-005 | Placeholder busca | Buscar por endereço ou ID |
| UX-PIPE-006 | Label coluna — contagem | [Estado] ([X]) |
| UX-PIPE-007 | Empty state coluna kanban | Nenhum caso neste estado. |
| UX-PIPE-008 | Empty state lista (sem filtro) | Nenhum caso encontrado. |
| UX-PIPE-009 | Empty state lista (com filtro) | Nenhum caso corresponde aos filtros aplicados. Tente outros critérios. |
| UX-PIPE-010 | Tooltip SLA verde | Dentro do prazo |
| UX-PIPE-011 | Tooltip SLA amarelo | Prazo vencendo em [X] dia(s) |
| UX-PIPE-012 | Tooltip SLA vermelho | Prazo vencido há [X] dia(s) |
| UX-PIPE-013 | Badge atualização realtime | Atualizado agora |
| UX-PIPE-014 | Sugestão visão lista (coluna extensa) | Coluna com muitos casos. Considere a Visão Lista para melhor controle. |
| UX-PIPE-015 | Botão "Ver mais" paginação kanban | Ver mais ([X] casos) |

### 4.2 Drawer Detalhe do Caso (T-022)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-PIPE-020 | Título drawer | Caso [ID] |
| UX-PIPE-021 | Tab resumo | Resumo |
| UX-PIPE-022 | Tab dossiê | Dossiê |
| UX-PIPE-023 | Tab histórico | Histórico |
| UX-PIPE-024 | Tab ações | Ações |
| UX-PIPE-025 | Gestor Financeiro — tab ações | Visualização financeira |
| UX-PIPE-026 | Gestor Financeiro — link | Ver no Financeiro |
| UX-PIPE-027 | Aria label fechar drawer | Fechar painel do caso |

---

## 5. Triagem (T-030 a T-034)

### 5.1 Fila FIFO (T-030)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-TRIA-001 | Título da seção | Triagem |
| UX-TRIA-002 | Badge próximo na fila | Próximo na fila |
| UX-TRIA-003 | Label tempo na fila | Há [X] dia(s) |
| UX-TRIA-004 | Label progresso dossiê | [X]/6 documentos |
| UX-TRIA-005 | Empty state fila — sem casos | Nenhum caso aguardando triagem no momento. |
| UX-TRIA-006 | Empty state — só bloqueados | Todos os casos aguardam regularização pelo Cedente. Nenhuma ação disponível no momento. |
| UX-TRIA-007 | Alerta fora de ordem FIFO | Este caso não é o próximo na fila. O caso [ID] aguarda há mais tempo. |
| UX-TRIA-008 | Link alerta FIFO | Ir para o caso mais antigo |

### 5.2 Painel do Caso (T-031)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-TRIA-010 | Botão voltar à fila | Retornar à fila |
| UX-TRIA-011 | Botão bloquear caso | Bloquear caso |
| UX-TRIA-012 | Botão qualificar caso (habilitado) | Qualificar caso |
| UX-TRIA-013 | Botão qualificar caso (desabilitado — tooltip) | Conclua a verificação de todos os documentos e a adimplência para qualificar. |
| UX-TRIA-014 | Botão solicitar documentos | Solicitar documentos |
| UX-TRIA-015 | Banner caso bloqueado | Caso bloqueado — Inadimplência. Aguardando regularização pelo Cedente. |
| UX-TRIA-016 | Banner caso qualificado | Caso qualificado. Aguardando publicação de oferta pelo Coordenador. |
| UX-TRIA-017 | Botão desbloquear (Analista/Coord.) | Desbloquear caso |

### 5.3 Aba Dossiê (T-032)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-TRIA-020 | Label progresso dossiê completo | Dossiê completo — 6/6 documentos verificados. |
| UX-TRIA-021 | Status documento: pendente | Pendente de envio |
| UX-TRIA-022 | Status documento: verificado | Verificado em [data] por [nome] |
| UX-TRIA-023 | Status documento: rejeitado | Rejeitado — [motivo] |
| UX-TRIA-024 | Botão visualizar documento | Visualizar |
| UX-TRIA-025 | Botão verificar documento | Verificar |
| UX-TRIA-026 | Botão rejeitar documento | Rejeitar |
| UX-TRIA-027 | Botão solicitar reenvio | Solicitar reenvio |
| UX-TRIA-028 | Label upload | Adicionar documento |
| UX-TRIA-029 | Erro visualização PDF corrompido | Não foi possível exibir este documento. O arquivo pode estar corrompido ou em formato inválido. |
| UX-TRIA-030 | Botão após erro PDF | Baixar arquivo |
| UX-TRIA-031 | Placeholder motivo rejeição | Descreva o motivo da rejeição... |
| UX-TRIA-032 | Label motivo rejeição | Motivo da rejeição |

### 5.4 Aba Adimplência (T-033)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-TRIA-040 | Label parcela em dia | Parcela em dia — comprovante verificado |
| UX-TRIA-041 | Label parcela pendente | Comprovante pendente de verificação |
| UX-TRIA-042 | Label parcela em atraso | Parcela em atraso — [mês/ano] |
| UX-TRIA-043 | Botão confirmar adimplência | Confirmar adimplência |
| UX-TRIA-044 | Botão registrar inadimplência | Registrar inadimplência |
| UX-TRIA-045 | Toggle exceção construtora | Aprovar via confirmação da construtora |
| UX-TRIA-046 | Label protocolo construtora | Número do protocolo ou referência |
| UX-TRIA-047 | Banner adimplência confirmada | Adimplência confirmada. |
| UX-TRIA-048 | Banner inadimplência | Inadimplência identificada em [X] parcela(s). |

### 5.5 Modal Confirmar Bloqueio (T-034)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-TRIA-050 | Título modal | Bloquear caso por inadimplência |
| UX-TRIA-051 | Corpo do modal | O Cedente será notificado automaticamente com orientações para regularização. |
| UX-TRIA-052 | Checkbox confirmação | Confirmo que as parcelas em atraso foram identificadas no comprovante. |
| UX-TRIA-053 | Botão confirmar bloqueio | Confirmar bloqueio |
| UX-TRIA-054 | Botão cancelar | Cancelar |
| UX-TRIA-055 | Toast sucesso após bloqueio | Caso bloqueado. O Cedente foi notificado. |

### 5.6 Notificações ao Cedente (geradas pelo sistema)

| **ID** | **Contexto** | **Texto (canal WhatsApp/E-mail)** |
|---|---|---|
| UX-TRIA-060 | Notificação de bloqueio ao Cedente | Seu caso foi bloqueado por pendência de parcelas. Regularize as parcelas indicadas e envie o comprovante de pagamento para desbloquear. |
| UX-TRIA-061 | Notificação de desbloqueio ao Cedente | Seu comprovante foi validado. Seu caso voltou para análise e será avaliado em até 3 dias úteis. |
| UX-TRIA-062 | Toast sucesso qualificação | Caso qualificado com sucesso. |

---

## 6. Negociação (T-040 a T-045)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-NEG-001 | Título da seção | Negociação |
| UX-NEG-002 | Empty state lista | Nenhum caso em negociação no momento. |
| UX-NEG-003 | Badge proposta ativa | Ativa |
| UX-NEG-004 | Label countdown proposta | [Xd Yh] restantes |
| UX-NEG-005 | Label proposta expirada | Proposta expirada em [data] |
| UX-NEG-006 | Banner proposta expirada | Proposta expirada. Aguardando nova proposta do Cessionário. |
| UX-NEG-007 | Badge contagem propostas | [X] proposta(s) ativa(s) |
| UX-NEG-008 | Tooltip badge propostas | [Lista: Cessionário — R$ valor] |
| UX-NEG-009 | Skip link timeline (a11y) | Ir para proposta ativa |
| UX-NEG-010 | Botão aceitar proposta | Aceitar proposta |
| UX-NEG-011 | Botão recusar/contraproposta | Recusar / Contraproposta |
| UX-NEG-012 | Botão escalar coordenador | Escalar para coordenador |
| UX-NEG-013 | Título modal aceite | Confirmar aceite da proposta |
| UX-NEG-014 | Checkbox aceite | Confirmo os valores calculados acima. |
| UX-NEG-015 | Botão confirmar aceite | Confirmar aceite |
| UX-NEG-016 | Toast sucesso aceite | Negociação concluída. Caso movido para Formalização. |
| UX-NEG-017 | Toggle modal recusa: recusar | Recusar proposta |
| UX-NEG-018 | Toggle modal recusa: contraproposta | Fazer contraproposta |
| UX-NEG-019 | Label valor contraproposta | Valor da contraproposta (R$) |
| UX-NEG-020 | Erro contraproposta valor menor | A contraproposta deve ser superior ao valor proposto. |
| UX-NEG-021 | Label motivo recusa (select) | Motivo da recusa |
| UX-NEG-022 | Toast sucesso recusa | Proposta recusada. O Cessionário foi notificado. |
| UX-NEG-023 | Toast sucesso contraproposta | Contraproposta enviada. |
| UX-NEG-024 | Título modal escalonamento | Aprovar escalonamento de cenário |
| UX-NEG-025 | Instrução escalonamento | Digite CONFIRMAR para aprovar o escalonamento. |
| UX-NEG-026 | Placeholder confirmação escalonamento | CONFIRMAR |
| UX-NEG-027 | Botão aprovar escalonamento | Aprovar escalonamento |
| UX-NEG-028 | Toast sucesso escalonamento | Escalonamento aprovado. Cenário alterado de [X] para [Y]. |

---

## 7. Formalização (T-050 a T-055)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-FORM-001 | Título da seção | Formalização |
| UX-FORM-002 | Badge caso pronto para fechamento | Pronto para fechamento |
| UX-FORM-003 | Label critérios (completo, anuência N/A) | [X]/[X] critérios cumpridos |
| UX-FORM-004 | Banner contingência ZapSign | Serviço de assinaturas temporariamente indisponível. Operações de assinatura estão pausadas. |
| UX-FORM-005 | Timestamp banner ZapSign | Indisponível desde [hora] |
| UX-FORM-006 | Link banner ZapSign (Coord/Master) | Registrar assinatura manual |
| UX-FORM-007 | Toast restauração ZapSign | Serviço de assinaturas restaurado. Envelopes pendentes sendo reenviados. |
| UX-FORM-008 | Label card assinaturas | Assinaturas |
| UX-FORM-009 | Progresso assinaturas | [X]/[Y] assinaturas |
| UX-FORM-010 | Status signatário: pendente | Pendente |
| UX-FORM-011 | Status signatário: assinado | Assinado em [data] |
| UX-FORM-012 | Status signatário: recusado | Recusado |
| UX-FORM-013 | Status signatário: expirado | Expirado |
| UX-FORM-014 | Botão enviar assinaturas | Enviar para assinatura |
| UX-FORM-015 | Botão reenviar | Reenviar (reenvio [X] de 3) |
| UX-FORM-016 | Critério assinaturas cumprido | Assinaturas concluídas |
| UX-FORM-017 | Label card anuência | Anuência da construtora |
| UX-FORM-018 | Status anuência N/A | N/A — Não exigido pelo contrato |
| UX-FORM-019 | Tooltip anuência N/A | Anuência não obrigatória para este contrato. Confirmado em [data]. |
| UX-FORM-020 | Botão confirmar anuência | Confirmar anuência recebida |
| UX-FORM-021 | Label card depósito escrow | Depósito na Conta Escrow |
| UX-FORM-022 | Status depósito: aguardando | Aguardando depósito do Cessionário |
| UX-FORM-023 | Status depósito: parcial | Depósito parcial recebido — aguardando complemento de R$ [valor] |
| UX-FORM-024 | Status depósito: confirmado | Depósito confirmado — R$ [valor] |
| UX-FORM-025 | Badge prazo próximo | Prazo em [X] dia(s) |
| UX-FORM-026 | Badge prazo estourado | Prazo vencido há [X] dia(s) |
| UX-FORM-027 | Botão prorrogar prazo (Coord.) | Prorrogar prazo |
| UX-FORM-028 | Botão confirmar fechamento (habilitado) | Confirmar fechamento |
| UX-FORM-029 | Botão confirmar fechamento (desabilitado — tooltip) | Conclua todos os critérios obrigatórios para confirmar o fechamento. |
| UX-FORM-030 | Título modal fechamento | Confirmar fechamento |
| UX-FORM-031 | Instrução campo de confirmação | Para confirmar, digite FECHAR no campo abaixo. |
| UX-FORM-032 | Placeholder campo confirmação | FECHAR |
| UX-FORM-033 | Botão confirmar no modal | Confirmar fechamento |
| UX-FORM-034 | Toast sucesso fechamento | Fechamento confirmado. Contagem de 15 dias iniciada. |
| UX-FORM-035 | Título modal Delta alto — aguardando | Aprovação pendente do Coordenador |
| UX-FORM-036 | Instrução Delta alto | O Delta calculado está acima do limiar de revisão (R$ [threshold]). O Coordenador foi notificado e deve aprovar antes do fechamento. |
| UX-FORM-037 | Botão fechar modal Delta alto | Entendido |
| UX-FORM-038 | Toast aprovação do Coordenador recebida | Fechamento aprovado pelo Coordenador. Confirme o processo. |
| UX-FORM-039 | Toast rejeição pelo Coordenador | Fechamento contestado pelo Coordenador. Motivo: [texto]. Revise os dados e tente novamente. |

---

## 8. Financeiro (T-060 a T-066)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-FIN-001 | Título da seção | Financeiro |
| UX-FIN-002 | Label ao vivo | Ao vivo |
| UX-FIN-003 | Badge prazo próximo (tabela) | Prazo em [X] dia(s) |
| UX-FIN-004 | Badge prazo estourado (tabela) | Prazo vencido — [X] dia(s) |
| UX-FIN-005 | Tooltip badge prazo | [X] dia(s) restantes para o prazo de depósito. |
| UX-FIN-006 | Tooltip badge vencido | Prazo vencido há [X] dia(s). |
| UX-FIN-007 | Status escrow: aberta | Aguardando depósito |
| UX-FIN-008 | Status escrow: depósito confirmado | Depósito confirmado |
| UX-FIN-009 | Status escrow: em distribuição | Em distribuição |
| UX-FIN-010 | Status escrow: distribuída | Distribuída |
| UX-FIN-011 | Status escrow: congelada | Congelada — reversão em andamento |
| UX-FIN-012 | Status escrow: bloqueada | Bloqueada — aguardando aprovação do Master |
| UX-FIN-013 | Status escrow: estornada | Estornada |
| UX-FIN-014 | Label seção distribuição | Distribuição prevista |
| UX-FIN-015 | Label Cedente na distribuição | Cedente |
| UX-FIN-016 | Label RS na distribuição | Repasse Seguro (comissão) |
| UX-FIN-017 | Banner estornada no drawer | Conta Escrow Estornada |
| UX-FIN-018 | Label download comprovante estorno | Baixar comprovante de estorno |
| UX-FIN-019 | Título modal bloquear distribuição | Bloquear distribuição |
| UX-FIN-020 | Instrução modal bloqueio | Descreva o motivo da irregularidade. A solicitação será enviada ao Master para aprovação. A distribuição será suspensa imediatamente. |
| UX-FIN-021 | Label motivo bloqueio | Motivo (mínimo 20 caracteres) |
| UX-FIN-022 | Botão solicitar bloqueio | Solicitar bloqueio |
| UX-FIN-023 | Toast sucesso bloqueio | Solicitação de bloqueio enviada ao Master. |
| UX-FIN-024 | Título modal reversão | Iniciar reversão |
| UX-FIN-025 | Instrução modal reversão | Esta ação estornará integralmente o depósito ao Cessionário. O Repasse Seguro não receberá comissão nesta operação. |
| UX-FIN-026 | Checkbox reversão | Confirmo o estorno integral de R$ [valor] ao Cessionário [nome]. |
| UX-FIN-027 | Botão confirmar reversão | Confirmar reversão |
| UX-FIN-028 | Toast sucesso reversão | Reversão iniciada. Comprovante gerado no dossiê do caso. |
| UX-FIN-029 | Ação notificar cessionário | Notificar Cessionário |
| UX-FIN-030 | Estado botão notificação enviada | Notificado ✓ |
| UX-FIN-031 | Tooltip botão notificação (cooldown) | Aguarde [X] min para reenviar. |
| UX-FIN-032 | Toast erro notificação | Não foi possível enviar a notificação. Tente novamente. |

---

## 9. Supervisão IA (T-070 a T-072)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-IA-001 | Título da seção | Supervisão IA |
| UX-IA-002 | Timestamp atualização | Atualizado: [X] segundo(s) atrás |
| UX-IA-003 | Status agente: ativo | Ativo |
| UX-IA-004 | Status agente: takeover manual | Takeover manual |
| UX-IA-005 | Status agente: falha técnica | Falha técnica |
| UX-IA-006 | Descrição falha técnica | Sem resposta há [X] minuto(s). |
| UX-IA-007 | Botão reiniciar agente (Coord.) | Reiniciar agente |
| UX-IA-008 | Botão takeover manual (Master) | Assumir controle |
| UX-IA-009 | Toast sucesso reinício | Agente reiniciado. |
| UX-IA-010 | Alerta falha para Master | O agente [nome] está com falha técnica. Ação necessária. |
| UX-IA-011 | Título modal takeover | Assumir controle do agente |
| UX-IA-012 | Instrução modal takeover | Ao assumir o controle, o agente para de tomar decisões automáticas. As decisões serão manuais até você liberar. |
| UX-IA-013 | Label motivo takeover | Motivo (obrigatório) |
| UX-IA-014 | Botão confirmar takeover | Confirmar takeover |
| UX-IA-015 | Toast sucesso takeover | Controle manual ativado para [nome do agente]. |

---

## 10. Usuários (T-075 a T-081)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-USR-001 | Título — Operadores | Operadores |
| UX-USR-002 | Título — Cedentes | Cedentes |
| UX-USR-003 | Título — Cessionários | Cessionários |
| UX-USR-004 | Botão novo operador (Master) | Novo operador |
| UX-USR-005 | Status operador: ativo | Ativo |
| UX-USR-006 | Status operador: suspenso | Suspenso |
| UX-USR-007 | Status operador: inativo | Inativo |
| UX-USR-008 | Label CPF mascarado | CPF: •••.•••.•••-••  [Mostrar] |
| UX-USR-009 | Tooltip mostrar CPF | Visualizar CPF registra um log de auditoria. |
| UX-USR-010 | Título modal suspender | Suspender conta |
| UX-USR-011 | Instrução modal suspender | Informe o motivo. O usuário perderá o acesso imediatamente. |
| UX-USR-012 | Opções motivo suspensão | Fraude suspeita / Inadimplência / Solicitação do usuário / Outro |
| UX-USR-013 | Botão confirmar suspensão | Confirmar suspensão |
| UX-USR-014 | Toast sucesso suspensão | Conta suspensa. O acesso foi revogado. |
| UX-USR-015 | Botão reativar | Reativar conta |
| UX-USR-016 | Toast sucesso reativação | Conta reativada. O usuário já pode acessar a plataforma. |

---

## 11. Relatórios (T-085 a T-091)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-REL-001 | Título da seção | Relatórios |
| UX-REL-002 | Label relatório SLA | SLA Operacional |
| UX-REL-003 | Label relatório volume | Volume de Casos |
| UX-REL-004 | Label relatório receita | Receita |
| UX-REL-005 | Label relatório conversão | Conversão |
| UX-REL-006 | Label relatório auditoria | Auditoria |
| UX-REL-007 | Label relatório IA | Agentes de IA |
| UX-REL-008 | Overlay relatório sem permissão | Disponível para [Perfil] |
| UX-REL-009 | Empty state gráfico | Nenhum dado para o período selecionado. |
| UX-REL-010 | Botão exportar CSV | Exportar CSV |
| UX-REL-011 | Botão exportar PDF | Exportar PDF |
| UX-REL-012 | Ícone agendamento ativo — tooltip | Agendamento ativo: [frequência]. Próximo envio: [data]. |
| UX-REL-013 | Toast agendamento configurado | Agendamento configurado. Próximo envio: [data]. |
| UX-REL-014 | Label data range picker | Período |

---

## 12. Configurações (T-095 a T-100)

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-CONF-001 | Título da seção | Configurações |
| UX-CONF-002 | Banner aviso de risco | Alterações aqui afetam toda a operação. Registre o motivo de cada alteração. |
| UX-CONF-003 | Helper comissão cedente | Aplica-se aos cenários B, C e D. Alterações afetam apenas novos casos. |
| UX-CONF-004 | Helper limiar Delta | Acima deste valor, o fechamento requer aprovação do Coordenador. |
| UX-CONF-005 | Helper Distrato Referência | Percentual do valor pago pelo Cedente usado como base de cálculo da comissão. |
| UX-CONF-006 | Label motivo da alteração | Motivo da alteração |
| UX-CONF-007 | Placeholder motivo | Descreva o motivo desta alteração... |
| UX-CONF-008 | Erro motivo muito curto | O motivo deve ter pelo menos 20 caracteres. |
| UX-CONF-009 | Botão salvar configurações | Salvar configurações |
| UX-CONF-010 | Botão restaurar padrões | Restaurar padrões |
| UX-CONF-011 | Modal confirmar restauração | Restaurar os valores padrão reverterá todas as configurações. Novos casos usarão os valores originais. |
| UX-CONF-012 | Toast sucesso salvar | Configurações salvas. |
| UX-CONF-013 | Toast sucesso restaurar | Configurações restauradas para os valores padrão. |
| UX-CONF-014 | Status integração: conectado | Conectado |
| UX-CONF-015 | Status integração: erro | Erro de conexão |
| UX-CONF-016 | Status integração: desconectado | Desconectado |
| UX-CONF-017 | API key mascarada | ••••••••••••••••  [Revelar] |
| UX-CONF-018 | Tooltip revelar API key | Revelar a chave registra um log de auditoria. |
| UX-CONF-019 | Label última sincronização | Última sincronização: [data e hora] |
| UX-CONF-020 | Variável válida no template | {{[nome_variavel]}} |
| UX-CONF-021 | Tooltip variável inválida | Variável não reconhecida. Verifique a lista de variáveis disponíveis. |
| UX-CONF-022 | Botão testar envio template | Testar envio |
| UX-CONF-023 | Toast template enviado | Template enviado para [e-mail de teste]. |
| UX-CONF-024 | Modal permissões — aviso | Esta alteração afetará todos os usuários com o perfil [X]. |

---

## 13. Estados Globais de Interface

| **ID** | **Contexto** | **Texto** |
|---|---|---|
| UX-GLOB-001 | Título sessão expirada | Sessão expirada |
| UX-GLOB-002 | Corpo sessão expirada | Sua sessão foi encerrada por inatividade. Faça login novamente para continuar. |
| UX-GLOB-003 | Botão sessão expirada | Fazer login |
| UX-GLOB-004 | Toast sem permissão | Você não tem permissão para executar esta ação. |
| UX-GLOB-005 | Banner manutenção | Sistema em manutenção programada. Operações estão pausadas. Previsão de retorno: [hora]. |
| UX-GLOB-006 | Banner offline | Sem conexão — dados podem estar desatualizados. |
| UX-GLOB-007 | Toast reconexão | Conexão restabelecida. |
| UX-GLOB-008 | Título erro 500 | Algo deu errado |
| UX-GLOB-009 | Corpo erro 500 | Não foi possível processar sua solicitação. Nossa equipe foi notificada. |
| UX-GLOB-010 | ID do erro (erro 500) | Código do erro: [ERR-ID] |
| UX-GLOB-011 | Botão erro 500 | Voltar ao início |
| UX-GLOB-012 | Link suporte erro 500 | Contatar suporte |
| UX-GLOB-013 | Aria label fechar modal genérico | Fechar |
| UX-GLOB-014 | Label loading genérico | Carregando... |
| UX-GLOB-015 | Label paginação | [X]–[Y] de [Total] |
| UX-GLOB-016 | Botão página anterior | Anterior |
| UX-GLOB-017 | Botão próxima página | Próxima |

---

## 14. Padrões de Validação de Formulários

### 14.1 Mensagens de validação padrão por tipo

| **Tipo de erro** | **Formato** | **Exemplo** |
|---|---|---|
| Campo obrigatório vazio | "O campo [Nome] é obrigatório." | "O campo E-mail é obrigatório." |
| Formato inválido | "Informe um [nome] válido." | "Informe um e-mail válido." |
| Valor abaixo do mínimo | "O valor mínimo é [X]." | "O valor mínimo é R$ 5,00." |
| Valor acima do máximo | "O valor máximo é [X]." | "O valor máximo é 50%." |
| Texto muito curto | "Mínimo de [X] caracteres." | "Mínimo de 20 caracteres." |
| Texto muito longo | "Máximo de [X] caracteres." | "Máximo de 2.000 caracteres." |
| Datas em ordem errada | "A data final deve ser posterior à data inicial." | — |
| Upload — tipo inválido | "Formato não suportado. Envie um arquivo PDF, JPEG ou PNG." | — |
| Upload — tamanho excedido | "O arquivo excede o limite de [X] MB." | — |

### 14.2 Exibição de erros inline

- Mensagem aparece **abaixo do campo** com borda `--destructive`.
- Exibição **em tempo real** para campos com formatação (e-mail, CPF, valor monetário).
- Para campos de texto livre: validação **ao sair do campo** (blur) e **ao submeter**.
- **Não bloquear** o campo enquanto o usuário digita — aguardar o blur.

---

## 15. Terminologia Padronizada

| **Termo preferido** | **Evitar** | **Motivo** |
|---|---|---|
| Caso | Processo, solicitação, ticket | Terminologia interna do produto |
| Cedente | Vendedor, proprietário | Terminologia contratual |
| Cessionário | Comprador, adquirente | Terminologia contratual |
| Conta Escrow | Conta garantia, depósito | Identidade do produto |
| Fechamento | Conclusão, finalização | Evento específico com critérios |
| Dossiê | Pasta, documentação | Terminologia interna |
| Cenário | Opção, modalidade | Terminologia definida no RN |
| Escalonamento | Downgrade, mudança | Terminologia interna |
| Anuência | Aprovação da construtora, autorização | Termo técnico jurídico correto |
| Reversão | Cancelamento pós-fechamento, desistência | Evento específico (RN-028) |
| Analista | Operador, agente | Perfil específico do produto |

---

*Documento gerado por Claude Code Desktop — Pipeline ShiftLabs v9.5 — 2026-03-22*
