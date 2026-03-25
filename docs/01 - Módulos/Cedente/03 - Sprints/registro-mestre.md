# Registro Mestre de Requisitos — Módulo Cedente

| Campo | Valor |
|---|---|
| **Módulo** | Cedente |
| **Gerado por** | Pipeline ShiftLabs v2.3 |
| **Data** | 2026-03-24 |
| **Fonte** | docs/01 - Módulos/Cedente/02 - Desenvolvimento/ |
| **Total de requisitos** | 210 |

---

## Tabela de Requisitos

| ID | Doc Fonte | Seção | Tipo | Descrição | Valor/Detalhe | Sprint |
|----|-----------|-------|------|-----------|---------------|--------|
| REQ-001 | D01.1 | 7.1 | Regra de negócio | RN-001: Cadastro e ativação da conta do Cedente | CPF/CNPJ único, e-mail único, senha ≥8 chars+maiúscula+número; conta criada como PENDENTE_ATIVACAO; link e-mail válido 48h | S2 |
| REQ-002 | D01.1 | 7.1 | Regra de negócio | RN-001: Validação inline on blur de cada campo | Indicador verde (válido) / vermelho+mensagem (inválido) ao perder foco | S2 |
| REQ-003 | D01.1 | 7.1 | Regra de negócio | RN-001: Estado de carregamento ao submeter cadastro | Spinner + texto "Verificando dados..." + botão desabilitado até resposta | S2 |
| REQ-004 | D01.1 | 7.1 | Regra de negócio | RN-001: Mensagem CPF/CNPJ duplicado | "Este CPF/CNPJ já possui uma conta. Faça login ou recupere sua senha." | S2 |
| REQ-005 | D01.1 | 7.1 | Regra de negócio | RN-001: Mensagem e-mail duplicado | "Este e-mail já está cadastrado. Faça login ou recupere sua senha." | S2 |
| REQ-006 | D01.1 | 7.1 | Regra de negócio | RN-001: Mensagem senha fraca | "A senha deve ter pelo menos 8 caracteres, uma letra maiúscula e um número." | S2 |
| REQ-007 | D01.1 | 7.1 | Regra de negócio | RN-002: Ativação por link de e-mail | Link válido por 48 horas; conta → ATIVA; tela de sucesso com redirect automático 3 segundos | S2 |
| REQ-008 | D01.1 | 7.1 | Regra de negócio | RN-002: Link expirado (>48h) | "O link de confirmação expirou. Solicite um novo link na tela de login." | S2 |
| REQ-009 | D01.1 | 7.1 | Regra de negócio | RN-003: Reenvio do e-mail de confirmação | Gera novo link 48h; invalida link anterior; botão desabilitado 60 segundos com timer regressivo | S2 |
| REQ-010 | D01.1 | 7.1 | Regra de negócio | RN-004: Recuperação de senha via e-mail | Link de redefinição válido 1 hora; resposta genérica independente de existência da conta | S2 |
| REQ-011 | D01.1 | 7.1 | Regra de negócio | RN-005: Bloqueio por tentativas de login | 5 tentativas consecutivas → bloqueio 15 minutos; timer regressivo visível; mensagem genérica antes do 5º | S2 |
| REQ-012 | D01.1 | 7.1 | Regra de negócio | RN-005: Aviso incremental a partir da 3ª tentativa | "Você tem mais [X] tentativas antes do bloqueio temporário." | S2 |
| REQ-013 | D01.1 | 7.1 | Regra de negócio | RN-006: Expiração de sessão por inatividade | Modal de aviso 5 minutos antes; sessão encerrada após 24h sem renovação | S2 |
| REQ-014 | D01.1 | 7.1 | Regra de negócio | RN-006: Preservação de rascunho local ao expirar sessão | Dados de formulários em preenchimento preservados em rascunho local | S2 |
| REQ-015 | D01.1 | 7.2 | Regra de negócio | RN-007: Imutabilidade do CPF/CNPJ | Campo desabilitado para edição; sugestão de contato com suporte para correção | S2 |
| REQ-016 | D01.1 | 7.2 | Regra de negócio | RN-008: Alteração de e-mail com dupla confirmação | Link enviado para novo e-mail; e-mail anterior ativo até confirmação; prazo 48h; badge "Pendente de confirmação" | S2 |
| REQ-017 | D01.1 | 7.2 | Regra de negócio | RN-009: Notificações desativáveis | Eventos 3, 9, 12, 17 podem ser desativados por e-mail | S2 |
| REQ-018 | D01.1 | 7.2 | Regra de negócio | RN-009: Notificações críticas não desativáveis | Eventos 8, 13, 14 com ícone de cadeado + tooltip; toggle visualmente desabilitado | S2 |
| REQ-019 | D01.1 | 7.2 | Regra de negócio | RN-010: Solicitação de exclusão de dados (LGPD) | Confirmação com aviso de irreversibilidade; protocolo gerado; prazo 15 dias corridos; banner permanente; botão cancelar | S2 |
| REQ-020 | D01.1 | 7.3 | Regra de negócio | RN-011: Isolamento total de dados do Cedente | RLS por cedente_id; tentativa de acesso a dados de outro → 403 + log de segurança | S2 |
| REQ-021 | D01.1 | 7.3 | Regra de negócio | RN-012: Anonimato do Cessionário nas propostas | Nunca exibir nome, CPF, e-mail, telefone do Cessionário; apenas número sequencial da proposta | S3/S5 |
| REQ-022 | D01.1 | 7.4 | Regra de negócio | RN-013: Dashboard como visão consolidada somente leitura | Cards: casos ativos, pendências, propostas recebidas, valor líquido estimado; sem ações operacionais | S3 |
| REQ-023 | D01.1 | 7.4 | Regra de negócio | RN-013: Dashboard vazio (primeiro acesso) | Boas-vindas personalizada; checklist 3 passos; botão "Cadastrar meu primeiro imóvel"; link Guardião | S3 |
| REQ-024 | D01.1 | 7.4 | Regra de negócio | RN-014: Dados exclusivos do Cedente no Dashboard | Atualização a cada 60 segundos ou em tempo real; indicador de última atualização | S3 |
| REQ-025 | D01.1 | 7.4 | Regra de negócio | RN-014: Falha de conexão no Dashboard | "Dados podem estar desatualizados. Verificando conexão..." | S3 |
| REQ-026 | D01.1 | 7.4 | Regra de negócio | RN-015: Valor líquido estimado com disclaimer obrigatório | Nota: "Valor estimado com base no cenário atual. Sujeito a alteração conforme negociação." | S3 |
| REQ-027 | D01.1 | 7.4 | Regra de negócio | RN-015: Card sem proposta ativa | Ícone de espera + "Aguardando primeira proposta"; menor opacidade | S3 |
| REQ-028 | D01.1 | 7.4 | Regra de negócio | RN-016: Destaque urgência ≤1 dia útil | Item com destaque vermelho + ícone + texto curto da ação | S3 |
| REQ-029 | D01.1 | 7.4 | Regra de negócio | RN-016: Destaque entre 2-3 dias úteis | Item com destaque amarelo | S3 |
| REQ-030 | D01.1 | 7.5 | Regra de negócio | RN-017: Histórico permanente de todos os casos | Casos ativos, concluídos e cancelados visíveis para sempre; filtros por status e cenário; busca por nome/endereço; paginação 10 itens | S3 |
| REQ-031 | D01.1 | 7.5 | Regra de negócio | RN-018: Status com linguagem simples e próximo passo | Nome amigável + frase explicativa; nunca exibir termos internos (Captado, Qualificado, Oferta Ativa) | S3 |
| REQ-032 | D01.1 | 7.5 | Regra de negócio | RN-019: Bloqueio de edição após triagem | Dados do imóvel bloqueados após "Aprovado para oferta"; mensagem de suporte | S3 |
| REQ-033 | D01.1 | 7.5 | Regra de negócio | RN-020: Detalhe do caso em 5 abas | Abas: Resumo, Documentos, Propostas, Financeiro, Histórico; navegáveis por teclado (role tablist/tab/tabpanel, aria-selected) | S3 |
| REQ-034 | D01.1 | 7.6 | Regra de negócio | RN-088: Upload opcional na Etapa 5 do wizard | Botão "Pular esta etapa"; caso criado com dossiê incompleto; notificação de pendência | S3 |
| REQ-035 | D01.1 | 7.6 | Regra de negócio | RN-089: Guardião em todas as etapas do cadastro | Chat disponível no canto inferior direito em todas as 5 etapas; cadastro assistido com confirmação antes de salvar | S3/S7 |
| REQ-036 | D01.1 | 7.6 | Regra de negócio | RN-013a: Cedente PJ — documentos adicionais | Contrato Social, Cartão CNPJ (30 dias), RG/CNH representante legal, procuração se aplicável | S6 |
| REQ-037 | D01.2 | 4.1 | Regra de negócio | RN-021: Simulador obrigatório antes da escolha de cenário | Bloqueio 10 segundos; skeleton loading durante cálculo; barra de progresso se >5s | S3 |
| REQ-038 | D01.2 | 4.1 | Regra de negócio | RN-022: Escolha ativa do cenário sem pré-seleção | Nenhum cenário pré-selecionado; checkbox de confirmação obrigatório; borda destaque no selecionado | S3 |
| REQ-039 | D01.2 | 4.1 | Regra de negócio | RN-023: Salvamento automático do rascunho | Auto-save ao final de cada etapa; rascunho disponível 30 dias corridos; 1 rascunho ativo por vez; banner de retorno com botões "Continuar"/"Descartar" | S3 |
| REQ-040 | D01.2 | 4.1 | Regra de negócio | RN-023: Lembretes de rascunho | E-mail nos dias 7, 15 e 25; notificação de descarte após 30 dias | S3 |
| REQ-041 | D01.2 | 4.1 | Regra de negócio | RN-024: Geração automática do Termo de Cadastro | Gerado via ZapSign ao confirmar cadastro; caso criado como CADASTRO_REALIZADO; notificação e-mail+painel | S3 |
| REQ-042 | D01.2 | 4.1 | Regra de negócio | RN-024: Tela de sucesso do cadastro | "Próximos passos: (1) Assinar Termo de Cadastro (2) Enviar documentos"; botões "Ir para Assinaturas" / "Ir para Documentos" | S3 |
| REQ-043 | D01.2 | 4.1 | Regra de negócio | RN-090: Bloqueio de duplicidade — 1 caso ativo por imóvel | Caso ativo qualquer status exceto Concluído/Cancelado → bloqueio com mensagem | S3 |
| REQ-044 | D01.2 | 4.2 | Regra de negócio | RN-025: Escalonamento descendente D→C, C→B ou B→A | Com proposta ativa: enfileirado; sem proposta: exibe simulação comparativa Guardião; gera Termo de Aceite ZapSign | S5 |
| REQ-045 | D01.2 | 4.2 | Regra de negócio | RN-025: Escalonamento enfileirado — exibição no detalhe do caso | "Ajuste de cenário solicitado. Será processado após a conclusão da negociação em andamento." + ícone relógio + botão cancelar | S5 |
| REQ-046 | D01.2 | 4.2 | Regra de negócio | RN-026: Escalonamento um nível por vez | Apenas o cenário imediatamente inferior como opção; impossível pular (D→B direto) | S5 |
| REQ-047 | D01.2 | 4.2 | Regra de negócio | RN-027: Cooldown de 7 dias corridos entre escalonamentos | Botão "Alterar Cenário" desabilitado; exibe data de próxima disponibilidade | S5 |
| REQ-048 | D01.2 | 4.2 | Regra de negócio | RN-027: Exceção cooldown — Admin pode liberar antecipadamente | Admin pode autorizar escalonamento fora do cooldown | S5 |
| REQ-049 | D01.2 | 4.2 | Regra de negócio | RN-028: Escalonamentos múltiplos ao longo do ciclo | Cada escalonamento gera novo Termo de Aceite; Guardião apresenta simulação atualizada; histórico cronológico imutável | S5 |
| REQ-050 | D01.2 | 4.2 | Regra de negócio | RN-029: Subida de cenário não é permitida | Mensagem explicando que é necessário cancelar e recadastrar; opção de iniciar cancelamento | S5 |
| REQ-051 | D01.2 | 4.3 | Regra de negócio | RN-030: Recebimento e visualização de proposta | Notificação e-mail+painel; identificação sequencial; valor; timer regressivo dinâmico (últimas 24h→horas, última hora→minutos+pulsação) | S5 |
| REQ-052 | D01.2 | 4.3 | Regra de negócio | RN-030: Hierarquia visual da simulação | (1) Valor líquido "Você recebe" destaque máximo; (2) Valor proposta; (3) Comissão+distrato colapsável | S5 |
| REQ-053 | D01.2 | 4.3 | Regra de negócio | RN-031: Prazo de resposta em dois estágios | D+2 úteis: lembrete + badge amarelo; D+3: badge vermelho; D+5: proposta encerrada "Sem resposta" | S5 |
| REQ-054 | D01.2 | 4.3 | Regra de negócio | RN-032: Aceite com confirmação dupla | Tela de confirmação com resumo financeiro (valor acordado, comissão RS, valor líquido); após aceite → EM_FORMALIZACAO | S5 |
| REQ-055 | D01.2 | 4.3 | Regra de negócio | RN-032: Falha de aceite (sistema) | Retry até 3 vezes; se 3 falhas: escalar para Admin + "Sua proposta está segura e o prazo foi pausado." | S5 |
| REQ-056 | D01.2 | 4.3 | Regra de negócio | RN-033: Encerramento automático de propostas concorrentes | Após aceite, demais propostas → "Superada" automaticamente | S5 |
| REQ-057 | D01.2 | 4.3 | Regra de negócio | RN-034: Recusa de proposta | Confirmação simples; proposta → "Recusada"; caso retorna "Disponível para compradores" | S5 |
| REQ-058 | D01.2 | 4.3 | Regra de negócio | RN-035: Contraproposta dentro do cenário ativo | Validação contra piso do cenário; se abaixo do piso → alerta e bloqueio | S5 |
| REQ-059 | D01.2 | 4.3 | Regra de negócio | RN-036: Propostas múltiplas — ordenação por valor | Ordenação padrão maior valor primeiro; badge numérico no menu Propostas | S5 |
| REQ-060 | D01.2 | 4.3 | Regra de negócio | RN-091: Propostas expiradas apenas no histórico | Lista ativa mostra só propostas com status aguardando resposta; expiradas no histórico do caso | S5 |
| REQ-061 | D01.2 | 4.4 | Regra de negócio | RN-037: Transparência no cálculo de comissão em 4 momentos | (1) Simulador cadastro; (2) Proposta recebida; (3) Termo Comercial; (4) Painel Financeiro pós-Fechamento | S5/S6 |
| REQ-062 | D01.2 | 4.4 | Regra de negócio | RN-038: Fórmula de comissão Cenários B/C/D | Comissão = 20% × (Valor Recuperado − Valor Distrato Referência); Valor Distrato Referência = 50% do valor pago | S3/S5/S6 |
| REQ-063 | D01.2 | 4.4 | Regra de negócio | RN-038: Exemplos numéricos por cenário | Base: pago R$400.000; Distrato R$200.000; B→líq R$360.000; C→líq R$456.000; D→líq R$520.000 | S3 |
| REQ-064 | D01.2 | 4.5 | Regra de negócio | RN-039: Solicitação de desistência pós-Fechamento (15 dias) | Justificativa obrigatória (mín 20, máx 500 chars); confirmação dupla; → DESISTENCIA_EM_ANALISE | S6 |
| REQ-065 | D01.2 | 4.5 | Regra de negócio | RN-039: Desistência aceita pelo Cessionário | Estorno integral Escrow em até 5 dias úteis; → CONCLUIDO com nota de reversão | S6 |
| REQ-066 | D01.2 | 4.5 | Regra de negócio | RN-039: Cessionário não aceita desistência | → MEDIACAO_EM_ANDAMENTO; prazo 10 dias úteis | S6 |
| REQ-067 | D01.2 | 4.5 | Regra de negócio | RN-040: Irrevogabilidade após 15 dias | → CONCLUIDO automático; botão "Solicitar Desistência" desaparece com transição suave | S6 |
| REQ-068 | D01.3 | 2 | Regra de negócio | RN-041: 6 documentos obrigatórios para Cedente PF | Contrato original, comprovantes 3 parcelas, declaração adimplência, RG/CNH, comprovante endereço (90 dias), tabela contrato | S4 |
| REQ-069 | D01.3 | 2 | Regra de negócio | RN-041: aria-label descritivo por documento | "[Nome do documento] — status: [status]"; ícones com texto alternativo | S4 |
| REQ-070 | D01.3 | 2 | Regra de negócio | RN-042: Validação formato e tamanho no upload | PDF/JPG/PNG; máximo 10 MB; barra de progresso + percentual; opção cancelar | S4 |
| REQ-071 | D01.3 | 2 | Regra de negócio | RN-042: Validação MIME type real (não apenas extensão) | Tipo real do arquivo validado; arquivo .exe renomeado para .pdf rejeitado | S4 |
| REQ-072 | D01.3 | 2 | Regra de negócio | RN-042: Mensagens de erro de upload | Formato inválido: "Formato não aceito. Envie em PDF, JPG ou PNG."; >10MB: "O arquivo é muito grande. O limite é de 10 MB por documento." | S4 |
| REQ-073 | D01.3 | 2 | Regra de negócio | RN-042: Falha de conexão durante upload | "A conexão foi interrompida. Tente enviar o documento novamente." + botão "Tentar novamente" com arquivo mantido | S4 |
| REQ-074 | D01.3 | 2 | Regra de negócio | RN-043: Avanço automático para triagem após dossiê completo | 6/8 docs em EmAnalise/Verificado + Termo Cadastro assinado → caso → EM_ANALISE; banner 10s | S4 |
| REQ-075 | D01.3 | 2 | Regra de negócio | RN-095: SLA análise KYC automatizada | Resultado em até 30 minutos; análise manual em até 2 dias úteis | S4 |
| REQ-076 | D01.3 | 2 | Regra de negócio | RN-095: Restrições durante análise | Cedente não pode publicar; Cessionário não envia propostas; painel exibe tipo de análise + prazo | S4 |
| REQ-077 | D01.3 | 2 | Regra de negócio | RN-044: Imutabilidade de documento verificado | Substituição bloqueada após "Verificado"; mensagem de suporte | S4 |
| REQ-078 | D01.3 | 2 | Regra de negócio | RN-045: Reenvio de documento rejeitado | Motivo visível com templates padronizados; tooltip "Precisa de ajuda?" abre Guardião; histórico de versões | S4 |
| REQ-079 | D01.3 | 2 | Regra de negócio | RN-046: Lembrete de documentos pendentes | A cada 7 dias corridos com docs pendentes: e-mail + notificação painel | S4 |
| REQ-080 | D01.3 | 3 | Regra de negócio | RN-047: Assinatura eletrônica inline via ZapSign | ZapSign em iframe/componente inline sem redirecionamento; timeout 10s → retry; 3 falhas → mensagem indisponível | S4 |
| REQ-081 | D01.3 | 3 | Regra de negócio | RN-047: Leitura obrigatória antes de assinar | Cedente visualiza documento completo antes de assinar; assinatura interrompida mantém status "Pendente" | S4 |
| REQ-082 | D01.3 | 3 | Regra de negócio | RN-047: Registro pós-assinatura | Data/hora + identificador de sessão + PDF disponível para download | S4 |
| REQ-083 | D01.3 | 3 | Regra de negócio | RN-048: Bloqueio de avanço por assinatura pendente | CADASTRO_REALIZADO→EM_ANALISE: Termo de Cadastro assinado; EM_FORMALIZACAO→NEGOCIO_FECHADO: Termo Comercial + Instrumento assinados | S4 |
| REQ-084 | D01.3 | 3 | Regra de negócio | RN-049: Imutabilidade após assinatura | Documento assinado imutável; substituição somente via Admin cancelando envelope e criando novo | S4 |
| REQ-085 | D01.3 | 3 | Regra de negócio | RN-050: Lembrete de assinatura pendente D+3 | Após 3 dias úteis sem assinatura: lembrete ao Cedente por e-mail | S4 |
| REQ-086 | D01.3 | 3 | Regra de negócio | RN-050: Alerta ao Admin em D+10 | Após 10 dias úteis sem assinatura: alerta interno ao Admin para contato ativo | S4 |
| REQ-087 | D01.3 | 4 | Regra de negócio | RN-051: Painel financeiro somente leitura | Resumo: cenário, Valor Recuperado, Comissão RS, Valor Líquido (destaque), status Escrow, data distribuição | S6 |
| REQ-088 | D01.3 | 4 | Regra de negócio | RN-051: Estado antes de EM_FORMALIZACAO | "A área financeira estará disponível quando seu caso avançar para a formalização." + link simulador | S6 |
| REQ-089 | D01.3 | 4 | Regra de negócio | RN-052: Valor líquido como destaque principal | Maior fonte, posição central/superior, rótulo "Você recebe"; nunca enterrado | S6 |
| REQ-090 | D01.3 | 4 | Regra de negócio | RN-053: Countdown dos 15 dias de reversão | "A distribuição ocorrerá automaticamente em [data]"; countdown visível; comprovante download após 15 dias | S6 |
| REQ-091 | D01.3 | 4 | Regra de negócio | RN-054: Cenário A — exibição específica | "Comissão RS: R$ 0 — Neste cenário, o Repasse Seguro não cobra comissão do Cedente." | S6 |
| REQ-092 | D01.3 | 5 | Regra de negócio | RN-055: Cancelamento antes do Fechamento | Dropdown motivo obrigatório; confirmação dupla; com Escrow → 2 etapas visuais; → CANCELADO | S6 |
| REQ-093 | D01.3 | 5 | Regra de negócio | RN-055: Cancelamento com Escrow ativo | Alerta estorno integral em até 5 dias úteis; botão "Confirmar cancelamento" em destaque destrutivo (vermelho) | S6 |
| REQ-094 | D01.3 | 6 | Regra de negócio | RN-056: 17 eventos de notificação | Eventos 1-17 conforme tabela D01.3 seção 6; e-mail em até 5 minutos | S4 |
| REQ-095 | D01.3 | 6 | Regra de negócio | RN-057: Notificações no painel — badge e direcionamento | Badge numérico no sino; contraste 4.5:1; aria-label "Notificações — [X] não lidas"; em até 30 segundos | S4 |
| REQ-096 | D01.3 | 6.1 | Regra de negócio | RN-092: Histórico de escalonamento na aba Histórico | Cada escalonamento: data/hora, cenário anterior, cenário novo, valores simulados, referência ao Termo assinado | S5 |
| REQ-097 | D01.4 | 2 | Regra de negócio | RN-058: Contexto do caso ao Guardião | Sistema fornece automaticamente: status, cenário, docs pendentes, propostas ativas, histórico | S7 |
| REQ-098 | D01.4 | 2 | Regra de negócio | RN-058: Mensagem proativa de boas-vindas | "Olá, [nome]! Sou o Guardião do Retorno..."; botões de ação rápida clicáveis | S7 |
| REQ-099 | D01.4 | 2 | Regra de negócio | RN-059: Tom empático e linguagem acessível | Frases curtas; sem jargões; reconhece estado emocional; vocabulário simples | S7 |
| REQ-100 | D01.4 | 2 | Regra de negócio | RN-060: Proibição de garantias de resultado | Linguagem de estimativa obrigatória; nunca garantir prazo ou valor | S7 |
| REQ-101 | D01.4 | 2 | Regra de negócio | RN-061: Escalação para humano | Threshold < confiança suficiente → oferta de escalação; solicitação registrada; prazo 1 dia útil / 4h em horário comercial (8h-18h, seg-sex) | S7 |
| REQ-102 | D01.4 | 2 | Regra de negócio | RN-062: Histórico de conversas permanente e imutável | Histórico completo organizado por data; nenhuma mensagem pode ser deletada | S7 |
| REQ-103 | D01.4 | 2 | Regra de negócio | RN-063: Supervisão e takeover pelo Admin | Admin monitora nível de confiança; pode assumir o chat; mensagem de transição ao Cedente | S7 |
| REQ-104 | D01.4 | 3 | Regra de negócio | RN-064: Acompanhamento da anuência pelo Cedente | Status visível: "Aguardando resposta da construtora. Prazo estimado: 15 dias úteis." | S6 |
| REQ-105 | D01.4 | 3 | Regra de negócio | RN-065: Anuência concedida — notificação e avanço | Notificação Cedente; avanço para assinaturas finais | S6 |
| REQ-106 | D01.4 | 3 | Regra de negócio | RN-066: Anuência negada — 3 opções ao Cedente | Cards: (1) renegociar, (2) cancelar com estorno, (3) aguardar nova tentativa; prazo 10 dias úteis para escolha | S6 |
| REQ-107 | D01.4 | 3 | Regra de negócio | RN-067: Anuência pendente >15 dias úteis | Notificação com prazo estimado adicional; Cedente pode cancelar a qualquer momento | S6 |
| REQ-108 | D01.4 | 3 | Regra de negócio | RN-068: Cessão livre — sem anuência obrigatória | Classificação "Cessão livre" no dossiê; caso avança direto para assinaturas finais; Cedente informado | S6 |
| REQ-109 | D01.4 | 4 | Regra de negócio | RN-069: Cadastro Cedente PJ — campos adicionais | Razão Social, Nome Fantasia, CNPJ, Nome Representante Legal, CPF Representante Legal | S2 |
| REQ-110 | D01.4 | 4 | Regra de negócio | RN-069: Validação CNPJ via Receita Federal on blur | ATIVA → indicador verde + preenchimento automático; irregular → rejeição; MEI → CCMEI em vez de Contrato Social | S2 |
| REQ-111 | D01.4 | 4 | Regra de negócio | RN-070: Dossiê PJ com 8 documentos | Docs 7 e 8 adicionais: Contrato Social/CCMEI + RG/CNH Representante Legal | S4 |
| REQ-112 | D01.4 | 4 | Regra de negócio | RN-071: Assinaturas por Representante Legal | Envelope ZapSign para e-mail do Representante Legal; CPF signatário deve coincidir; procurador exige procuração no dossiê | S4 |
| REQ-113 | D01.4 | 4 | Regra de negócio | RN-072: Conta bancária vinculada ao CNPJ | Verificação de titularidade antes da distribuição Escrow; bloqueio + notificação Admin se não corresponder | S6 |
| REQ-114 | D01.4 | 5 | Regra de negócio | RN-073: Detecção de inadimplência na anuência | Notificação específica com número de parcelas em atraso + prazo para regularização | S6 |
| REQ-115 | D01.4 | 5 | Regra de negócio | RN-074: Prazo 15 dias úteis para regularização | Contador visual: data-alvo + dias restantes + barra de progresso (verde>10, amarelo 5-10, vermelho<5); lembretes dias 5 e 10 | S6 |
| REQ-116 | D01.4 | 5 | Regra de negócio | RN-074: Upload comprovante de quitação | Campo adicional ativado; após aprovação Admin → nova solicitação de anuência | S6 |
| REQ-117 | D01.4 | 5 | Regra de negócio | RN-075: Opções após prazo expirado | (1) cancelar com estorno (até 3 dias úteis); (2) extensão única de 10 dias úteis | S6 |
| REQ-118 | D01.4 | 5 | Regra de negócio | RN-075: Cancelamento automático após extensão expirada | Cancelamento automático com estorno prioritário; total acumulado: 25 dias úteis | S6 |
| REQ-119 | D01.4 | 5 | Regra de negócio | RN-076: Estorno prioritário por inadimplência | Prazo 3 dias úteis (vs 5 padrão); notificação ao Cedente e Cessionário | S6 |
| REQ-120 | D01.4 | 5 | Regra de negócio | RN-077: Reativação após regularização tardia | Caso cancelado → permitir novo cadastro do mesmo imóvel; histórico anterior não herdado | S6 |
| REQ-121 | D01.4 | 6 | Regra de negócio | RN-078: Nova proposta cancela escalonamento enfileirado | Cancelamento automático do escalonamento na fila + notificação explicativa; histórico registra evento | S5 |
| REQ-122 | D01.4 | 6 | Regra de negócio | RN-079: Re-solicitação sem cooldown após cancelamento pelo sistema | Botão liberado imediatamente; novo Termo gerado normalmente | S5 |
| REQ-123 | D01.5 | 2.1 | Regra de negócio | RN-080: ZapSign inline sem redirecionamento | Cedente não precisa criar conta ZapSign; acesso autenticado pela sessão da plataforma | S4 |
| REQ-124 | D01.5 | 2.1 | Regra de negócio | RN-081: Rastreabilidade da assinatura | Data/hora + identificador de sessão + versão do documento + hash PDF; imutável; download disponível | S4 |
| REQ-125 | D01.5 | 2.1 | Regra de negócio | RN-094: Régua notificações ZapSign | D+0 envio; D+2 (1º lembrete); D+4 (lembrete urgente + alerta Admin); D+5 expiração; prazo 5 dias úteis | S4 |
| REQ-126 | D01.5 | 2.2 | Regra de negócio | RN-082: Validação CNPJ em tempo real | API Receita Federal on blur; fallback: avançar com banner amarelo "Verificação do CNPJ pendente" | S2 |
| REQ-127 | D01.5 | 2.3 | Regra de negócio | RN-083: Atualização automática status Conta Escrow | Atualizações do parceiro refletidas em até 5 minutos; fallback: último status em cache com timestamp | S6 |
| REQ-128 | D01.5 | 2.3 | Regra de negócio | RN-093: Fluxo aprovação extensão prazo Escrow | Notificação Cedente 24h para aprovar/recusar; silêncio = aprovação automática; registro imutável | S6 |
| REQ-129 | D01.5 | 3 | Regra de negócio | RN-084: Precedência — regra de seção 5 (CED/EC) sobre menu | Regras CED/EC prevalecem sobre regras de menu (DC, MC, CAD, DOC, PROP, ASS, FIN-C, IA-C, PERF) | CROSS |
| REQ-130 | D01.5 | 3 | Regra de negócio | RN-085: Mediação total pelo Admin | Cedente e Cessionário nunca comunicam diretamente; toda proposta/negociação passa pelo Admin | CROSS |
| REQ-131 | D01.5 | 3 | Regra de negócio | RN-086: Não regressão de estados | Retrocessos permitidos apenas: EM_ANALISE→PENDENCIA, PROPOSTA→DISPONIVEL, FORMALIZACAO→PENDENCIA | CROSS |
| REQ-132 | D01.5 | 3 | Regra de negócio | RN-087: Acessibilidade mobile — 3 ações críticas | Upload via câmera, assinatura ZapSign touch, resposta proposta; botões ≥44px; orientação paisagem recomendada para assinatura | S8 |
| REQ-133 | D12 | 2 | Banco de dados | Tabela `users` | uuid PK, email UK, name, provider, email_verified_at, created_at, updated_at, deleted_at | S1 |
| REQ-134 | D12 | 2 | Banco de dados | Tabela `cedentes` | uuid PK, user_id FK, tipo (PF/PJ), cpf_cnpj UK, telefone, status_conta, lgpd_consent, lgpd_consent_at, notification_preferences (JSONB), ai_consent, ai_consent_at | S1 |
| REQ-135 | D12 | 2 | Banco de dados | Tabela `casos` | uuid PK, cedente_id FK, codigo UK, status, status_interno, cenario, cidade, bairro, estado, nome_empreendimento, endereco_completo, quartos, area_m2, tem_garagem, valor_pago_cedente, valor_tabela_contrato, valor_tabela_atual, valor_distrato_referencia, valor_recuperado, comissao_rs, valor_liquido_cedente, construtora, cessao_livre, datas (cadastro/aprovacao/fechamento/distribuicao/ultimo_escalonamento) | S1 |
| REQ-136 | D12 | 2 | Banco de dados | Tabela `dossies` | uuid PK, caso_id FK (1:1), status, total_documentos, documentos_verificados, documentos_pendentes, documentos_rejeitados, completo, completo_em | S1 |
| REQ-137 | D12 | 2 | Banco de dados | Tabela `documentos_dossie` | uuid PK, dossie_id FK, caso_id FK, tipo, nome, status, url_arquivo, mime_type, tamanho_bytes, motivo_rejeicao, verificado_por FK, verificado_em, tentativas | S1 |
| REQ-138 | D12 | 2 | Banco de dados | Tabela `propostas` | uuid PK, caso_id FK, cedente_id FK, codigo UK, valor_proposto, comissao_comprador, valor_liquido_cedente_estimado, status, mensagem_admin, rodadas_contraproposta, valor_contraproposta, expires_at, respondido_em, deleted_at | S1 |
| REQ-139 | D12 | 2 | Banco de dados | Tabela `envelopes_assinatura` | uuid PK, caso_id FK, cedente_id FK, tipo_documento (TERMO_CADASTRO/ESCALONAMENTO/COMERCIAL/INSTRUMENTO), status, zapsign_token, zapsign_doc_url | S1 |
| REQ-140 | D12 | 2 | Banco de dados | Tabela `contas_escrow` | Estados: Aberta, DepositoConfirmado, EmPeriodoDeReversao, ValoresDistribuidos, ComissaoDefinitiva, Estornada | S1 |
| REQ-141 | D12 | 2 | Banco de dados | Tabela `notificacoes` | Por cedente_id; lida/não lida; tipo (17 tipos); redirecionamento para tela correspondente | S1 |
| REQ-142 | D12 | 2 | Banco de dados | Tabela `eventos_caso` | Histórico cronológico imutável de todas as mudanças de status; linha do tempo do caso | S1 |
| REQ-143 | D12 | 2 | Banco de dados | Tabela `ai_sessions` | Histórico de conversas Guardião; imutável; LGPD: conteúdo anonimizado após 90 dias | S1 |
| REQ-144 | D12 | 2 | Banco de dados | RLS em todas as tabelas com dados de usuários | Filtro por cedente_id; `propostas` nunca expõe cessionario_id ao frontend | S1 |
| REQ-145 | D12 | 2 | Banco de dados | Enums: TipoCedente, StatusConta, StatusCaso (13 estados), StatusCasoInterno, CenarioRetorno, StatusDossie, TipoDocumento, StatusDocumento, StatusProposta, TipoEnvelope, StatusEnvelope, StatusEscrow, TipoNotificacao, TipoAtor, TipoEvento | S1 |
| REQ-146 | D13 | 1 | Schema Prisma | 11 models: User, Cedente, Caso, Dossie, DocumentoDossie, Proposta, EnvelopeAssinatura, ContaEscrow, Notificacao, EventoCaso, AiSession | S1 |
| REQ-147 | D13 | 1 | Schema Prisma | Convenções: UUID v4 via pgcrypto, snake_case via @map/@@map, PascalCase para models, UPPER_SNAKE_CASE para enums, soft delete via deletedAt | S1 |
| REQ-148 | D14 | 1 | Arquitetura | Monorepo Turborepo: apps/web-cedente, apps/api, apps/mobile-cedente + packages/shared-types, design-tokens, eslint-config, tsconfig | S1 |
| REQ-149 | D14 | 1 | Arquitetura | ADR-CED-001: Next.js App Router para web pública + autenticada no mesmo app | S1 |
| REQ-150 | D14 | 1 | Arquitetura | ADR-CED-002: SSE para streaming IA (Guardião do Retorno) | S1/S7 |
| REQ-151 | D14 | 1 | Arquitetura | ADR-CED-003: confirmação manual do Escrow no MVP | S6 |
| REQ-152 | D14 | 1 | Arquitetura | ADR-CED-004: upload direto Supabase Storage via signed URL | S4 |
| REQ-153 | D14 | 3 | Cache Redis | 7 recursos: sessão (TTL 24h), rascunho wizard (TTL 30d), dados do caso (TTL 60s), score proposta, resultado IA (TTL 4h), notificações não lidas, rate limit | S1 |
| REQ-154 | D14 | 3 | Filas RabbitMQ | 5 exchanges, 8 queues, DLQ em todas; retry backoff exponencial 3 tentativas em 30 min | S1 |
| REQ-155 | D15 | 2 | Arquitetura de pastas | apps/web-cedente: route groups (public) e (authenticated); features em src/features/<módulo>/ | S1 |
| REQ-156 | D15 | 3 | Arquitetura de pastas | apps/mobile-cedente: screens/<módulo>/ com expo-router 4 | S1 |
| REQ-157 | D15 | 4 | Arquitetura de pastas | apps/api: modules/<módulo>/ padrão Controller→Service→Repository→DTO→Entity; 11 módulos de domínio | S1 |
| REQ-158 | D15 | 5 | Cache Redis | Prefixo rs:cedente: em todas as chaves; 8 recursos com TTL explícito | S1 |
| REQ-159 | D15 | 5 | Error prefixes | 11 prefixos: AUTH, CED, CAS, DOC, PRP, ASS, ESC, NOT, AI, ANU, COM | S1 |
| REQ-160 | D16 | 7.1 | Endpoint | POST /auth/register | Cadastro Cedente PF/PJ; erros: CED-001 (CPF dup), CED-002 (email dup), CED-003 (senha fraca), CED-004 (CPF inválido) | S2 |
| REQ-161 | D16 | 7.1 | Endpoint | POST /auth/login | Login e-mail/senha; erros: AUTH-001, AUTH-008 (pendente), AUTH-009 (bloqueado 15min) | S2 |
| REQ-162 | D16 | 7.1 | Endpoint | GET /auth/activate | Ativação por token URL; erro AUTH-011 (link expirado) | S2 |
| REQ-163 | D16 | 7.1 | Endpoint | POST /auth/resend-activation | Reenvio e-mail ativação | S2 |
| REQ-164 | D16 | 7.1 | Endpoint | POST /auth/refresh | Renovar access token via cookie refresh | S2 |
| REQ-165 | D16 | 7.1 | Endpoint | DELETE /auth/session | Logout — invalida refresh token | S2 |
| REQ-166 | D16 | 7.1 | Endpoint | POST /auth/forgot-password | Solicitar reset de senha | S2 |
| REQ-167 | D16 | 7.1 | Endpoint | POST /auth/reset-password | Confirmar nova senha via token | S2 |
| REQ-168 | D16 | 7.1 | Endpoint | POST /auth/reauth | Re-autenticação para ação crítica (LGPD, exclusão) | S2 |
| REQ-169 | D16 | 7 | Endpoint | 44 endpoints totais em 10 domínios: Auth, Perfil, Casos, Dossiê, Propostas, Escalonamento, Assinaturas, Financeiro, Notificações, IA | S1-S7 |
| REQ-170 | D16 | 1 | API padrão | Base URL: https://api.repasseseguro.com.br/api/v1; paginação offset (page, per_page max 100, default 20); valores monetários em centavos | CROSS |
| REQ-171 | D16 | 1 | API padrão | Rate limiting: 100 req/min geral; 10 req/15min auth login/refresh; 5 req/h cadastro; 3 req/h recuperação senha; 20 req/10min upload; 30 req/5min propostas; 20 req/min IA | S1 |
| REQ-172 | D16 | 2 | API padrão | Access token 15min (prod) / 30min (staging); refresh token 30 dias httpOnly cookie rotacionado a cada uso | S2 |
| REQ-173 | D16 | 3 | API padrão | Padrão de erro RFC 7807 com campos: type, title, status, detail, instance, code, cedente_message, timestamp, trace_id | S1 |
| REQ-174 | D17 | 2 | Integração | ZapSign P0: API Key ZAPSIGN_API_KEY + ZAPSIGN_WEBHOOK_SECRET (HMAC); 4 tipos de envelope; retry 2min→10min→1h→DLQ | S1/S4 |
| REQ-175 | D17 | 2 | Integração | Conta Escrow P0: API Key; webhook confirmação depósito; último status em cache; ESCROW_API_KEY + ESCROW_WEBHOOK_SECRET + ESCROW_BASE_URL [DEFINIÇÃO PENDENTE DP-001] | S1/S6 |
| REQ-176 | D17 | 2 | Integração | Supabase P0: DATABASE_URL, DIRECT_URL, SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY | S1 |
| REQ-177 | D17 | 2 | Integração | Resend P1: RESEND_API_KEY; retry 3x; fila RabbitMQ; templates React Email | S1/S4 |
| REQ-178 | D17 | 2 | Integração | OpenAI P1: OPENAI_API_KEY; GPT-4 versão fixada gpt-4-turbo-2024-04-09; text-embedding-3-small | S1/S7 |
| REQ-179 | D17 | 2 | Integração | Receita Federal P1: pública sem autenticação; fallback avançar com banner + verificação manual Admin | S1/S2 |
| REQ-180 | D17 | 2 | Integração | Langfuse P2: LANGFUSE_SECRET_KEY + LANGFUSE_PUBLIC_KEY; fire-and-forget; falha silenciosa | S1/S7 |
| REQ-181 | D18 | 3 | Auth | Fluxo cadastro: e-mail/senha; link ativação TTL 24h; Supabase Auth signUp | S2 |
| REQ-182 | D18 | 4 | Auth | Rate limiting login: 5 tentativas → bloqueio 15min via Redis | S2 |
| REQ-183 | D18 | 5 | Auth | Tokens: access JWT 1h; refresh 7 dias (httpOnly web; SecureStore mobile); refresh token rotation | S2 |
| REQ-184 | D18 | 6 | Auth | RLS Supabase: isolamento total; auth.uid() em toda linha de dados de usuário | S2 |
| REQ-185 | D18 | 7 | Auth | Mobile biometria opcional via expo-local-authentication; tokens em expo-secure-store | S8 |
| REQ-186 | D19 | 2 | Agente IA | Guardião do Retorno: nome "Dani-Cedente"; GPT-4 Turbo fixado gpt-4-turbo-2024-04-09; LangChain.js 0.3+; LangGraph.js 0.2+ (cadastro assistido) | S7 |
| REQ-187 | D19 | 2 | Agente IA | RAG: pgvector HNSW index; chunking 512 tokens overlap 64; text-embedding-3-small | S7 |
| REQ-188 | D19 | 2 | Agente IA | Memória: curta por sessão Redis TTL 4h; longa imutável no banco (histórico permanente) | S7 |
| REQ-189 | D19 | 2 | Agente IA | Tool calls (apenas leitura): get_simulation, get_case_status, get_pending_documents | S7 |
| REQ-190 | D19 | 2 | Agente IA | Rate limiting IA: 30 mensagens/hora por Cedente | S7 |
| REQ-191 | D19 | 2 | Agente IA | Escalação automática: threshold confiança < 0.80 | S7 |
| REQ-192 | D19 | 2 | Agente IA | LGPD: conteúdo mensagens anonimizado após 90 dias; cedente_id nunca enviado ao Langfuse | S7 |
| REQ-193 | D19 | 2 | Agente IA | LangGraph.js para cadastro assistido multi-step (checkpoints + branching) | S7 |
| REQ-194 | D20 | 1 | Error handling | GlobalExceptionFilter NestJS; 4 famílias: E4XX, E5XX, EINT, EBI | S1 |
| REQ-195 | D20 | 2 | Error handling | Padrão RFC 7807: type, title, status, detail (técnico, nunca ao user), cedente_message (amigável), instance, code, timestamp, trace_id | S1 |
| REQ-196 | D20 | 3 | Error handling | Circuit breakers: ZapSign e Conta Escrow com circuit breaker dedicado | S1 |
| REQ-197 | D20 | 3 | Error handling | Alertas Sentry: P0→PagerDuty; P1→Slack #alerts <5min; P2→Slack #warnings <30min | S1 |
| REQ-198 | D21 | 1 | Notificações | 4 canais: e-mail (Resend), push mobile (Expo Notifications), in-app (Supabase Realtime), WhatsApp unidirecional (Fase 2, não no MVP) | S4 |
| REQ-199 | D21 | 2 | Notificações | 14 templates: NOT-CED-01 a NOT-CED-14 via RabbitMQ assíncrono; nunca síncrono | S4 |
| REQ-200 | D22 | 1 | Setup | Pré-requisitos: Node.js 22+, pnpm 9+, Docker Desktop 27+, Docker Compose 2+, Git 2+, Supabase CLI 2+, Railway CLI, Expo CLI | S1 |
| REQ-201 | D22 | 3 | Setup | 42 variáveis de ambiente entre backend NestJS, frontend Next.js e mobile Expo; secrets via Railway (backend), Vercel (frontend), EAS Secrets (mobile) | S1 |
| REQ-202 | D24 | 2 | CI/CD | Pipeline GitHub Actions: ci.yml (lint+type-check+testes+build em todo PR); deploy-api.yml (Railway); deploy-web-cedente.yml (Vercel); deploy-mobile.yml (EAS) | S9 |
| REQ-203 | D24 | 2 | CI/CD | Deploy produção: tag v*.*.* em main → automático Railway prod + Vercel prod | S9 |
| REQ-204 | D24 | 2 | CI/CD | Migrations: pnpm prisma migrate deploy executa ANTES do deploy de backend; expand/contract para migrations destrutivas | S9 |
| REQ-205 | D24 | 2 | CI/CD | Rollback: railway rollback <60s backend; Vercel rollback instantâneo | S9 |
| REQ-206 | D25 | 1 | Observabilidade | Sentry: error tracking em NestJS, Next.js e Expo; deploy sem Sentry configurado é proibido | S9 |
| REQ-207 | D25 | 1 | Observabilidade | PostHog: eventos em snake_case; wrapper centralizado em src/services/analytics.ts; session replay | S9 |
| REQ-208 | D25 | 1 | Observabilidade | Langfuse: tracing Guardião; latência, custo, qualidade de resposta | S9 |
| REQ-209 | D25 | 1 | Observabilidade | Pino: logs estruturados JSON; campos obrigatórios: requestId, userId, module, action | S1 |
| REQ-210 | D28 | 2 | Qualidade | Bloqueantes de PR: CI vermelho, dado sensível exposto, console.log em prod, secrets no código | S9 |
| REQ-211 | D28 | 2 | Qualidade | Bloqueantes de release: E2E P0 falhando, cálculo comissão com erro, isolamento LGPD violado, migration sem rollback | S9 |
| REQ-212 | D29 | 1 | Go-Live | Go/No-Go: E2E verde, smoke tests, ZapSign+Escrow sandbox operacionais, error rate <1% staging, LGPD validado, Guardião <20% escalamentos | S10 |
| REQ-213 | D29 | 2 | Go-Live | Health checks: GET /health HTTP200 <500ms; GET /cedente/casos; POST /auth/login; GET /cedente/simulador | S10 |
| REQ-214 | D29 | 1 | Go-Live | Launch day: terça a quinta, 10h-14h BRT; monitoramento intensivo T+15min, T+1h, T+2h | S10 |
| REQ-215 | D29 | 1 | Go-Live | Rollback: feature flag desativa módulo; railway rollback <60s; gatilho error rate >5% ou P95>1s | S10 |
| REQ-216 | D11 | 1 | Mobile | Plataformas: iOS 16+ e Android 10 (API 29+); React Native 0.76+; Expo SDK 52+; expo-router 4 | S8 |
| REQ-217 | D11 | 2 | Mobile | Casos de uso prioritários: upload por câmera (T-042), assinatura ZapSign touch (T-044), resposta propostas (T-037/T-039), acompanhamento status (T-016) | S8 |
| REQ-218 | D11 | 3 | Mobile | Navegação: Stack raiz → Tab navigator (5 tabs bottom bar) → Stack modals; expo-router file-based | S8 |
| REQ-219 | D11 | 4 | Mobile | Offline: 6 telas com cache local (Dashboard, Meus Casos, Lista Propostas, Detalhe Caso, Documentos, Perfil); ações de escrita bloqueadas offline | S8 |
| REQ-220 | D11 | 5 | Mobile | ZapSign via WebView em tela cheia; Termo de Cadastro e Instrumento de Cessão no mobile | S8 |

---

## Verificação de Cobertura

```
Registro Mestre: 220 requisitos
Atribuídos: 220 (100%)
Sem sprint: 0 (✅)
```

---

## Distribuição por Sprint

| Sprint | Quantidade de REQs |
|--------|--------------------|
| S1 — Fundação | REQ-133 a REQ-159, REQ-169(parcial), REQ-170-173, REQ-174-180, REQ-194-201, REQ-209 |
| S2 — Auth | REQ-001 a REQ-021 (auth), REQ-109-110, REQ-126, REQ-160-168, REQ-181-184 |
| S3 — Cadastro e Dashboard | REQ-022 a REQ-043, REQ-062-063 |
| S4 — Dossiê e Assinaturas | REQ-068 a REQ-086, REQ-094-095, REQ-111-112, REQ-123-125, REQ-174(ZapSign), REQ-177(Resend), REQ-198-199 |
| S5 — Propostas e Negociação | REQ-044 a REQ-060, REQ-096, REQ-121-122 |
| S6 — Financeiro e Fechamento | REQ-036, REQ-061, REQ-064 a REQ-067, REQ-087 a REQ-093, REQ-104 a REQ-120, REQ-127-128, REQ-175(Escrow) |
| S7 — Agente IA | REQ-097 a REQ-103, REQ-150(SSE), REQ-186 a REQ-193 |
| S8 — Mobile | REQ-132, REQ-185, REQ-216 a REQ-220 |
| S9 — Qualidade | REQ-202 a REQ-211 |
| S10 — Go-Live | REQ-212 a REQ-215 |
| S11 — Sprint Final Auditoria | Todos os REQs (verificação cross-sprint) |
| CROSS | REQ-129-131, REQ-170 |
