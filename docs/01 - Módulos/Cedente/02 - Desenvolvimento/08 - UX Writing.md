# 08 - UX Writing

## Módulo Cedente · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Documento** | 08 - UX Writing |
| **Destinatário** | Design, Frontend e Produto |
| **Escopo** | Tom de voz · Padrões de mensagem · Microcopy por componente · Catálogo de strings por tela (T-001 a T-052) |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 (America/Fortaleza) |
| **Referências** | 01.1 - Regras de Negócio · 01.2 - Regras de Negócio · 06 - Mapa de Telas · 07 - Wireframes · 03 - Brand Guide |

---

> 📌 **TL;DR**
>
> - O tom do módulo Cedente é: **empático, orientado ao resultado financeiro, sem jargão técnico**. A plataforma lida com a decisão mais difícil do Cedente — abrir mão de um contrato imobiliário. Cada palavra precisa transmitir segurança e clareza sobre o que ele vai receber.
> - Erros seguem o padrão de 2 frases: (1) o que aconteceu — específico; (2) o que o Cedente deve fazer — ação clara.
> - O Guardião do Retorno fala de forma acolhedora e orientada a resultados. Nunca usa jargão jurídico ou financeiro sem explicar. Nunca cria urgência artificial.
> - Valores financeiros sempre formatados: R$ 87.500,00 — nunca "87500" nem "R$87.500".
> - Prazos sempre com data completa ou dias exatos: "até 28/03/2026" ou "em 5 dias úteis" — nunca "em breve" ou "em alguns dias".
> - Cenários sempre com letra maiúscula: Cenário A, Cenário B, Cenário C, Cenário D.
> - Personas: **Cedente PF** (pessoa física com contrato imobiliário, assustada com burocracia, quer saber quanto vai receber) e **Cedente PJ** (empresa ou investidor com portfólio de contratos).

---

## 1. Princípios de Tom de Voz

### 1.1 Atributos de Tom

| **Atributo** | **Descrição** | **Na prática** |
|---|---|---|
| **Empático** | O Cedente está tomando uma decisão difícil. A plataforma reconhece isso sem dramatizar. | "Seu imóvel está disponível para compradores qualificados." (não: "Aguardando propostas.") |
| **Orientado ao resultado** | O Cedente quer saber quanto vai receber. Valor líquido em destaque sempre. | "Você receberá R$ 45.000,00 líquidos." (não: "Valor após dedução de comissão: R$ 45.000,00") |
| **Claro** | Sem jargão jurídico ou financeiro sem explicação. Termos do domínio são explicados na primeira ocorrência. | "Conta Escrow (conta-garantia): onde o dinheiro fica protegido até a liberação." |
| **Direto** | CTAs acionáveis, sem rodeios. O usuário sabe exatamente o que vai acontecer ao clicar. | "Aceitar proposta" (não: "Prosseguir com a aceitação desta proposta") |
| **Seguro** | A plataforma transmite confiança em cada micro-interação. Nunca cria dúvida desnecessária. | "Imóvel cadastrado com sucesso!" (não: "Seu cadastro foi recebido e será processado.") |
| **Sem drama** | Erros e pendências são parte do fluxo. Trate-os como etapas, não como falhas. | "Um documento precisa de atenção. Veja o que está pendente." (não: "ERRO: Documento rejeitado!") |

### 1.2 O Que Nunca Fazer

> 🚫 **Anti-padrões de escrita:**
>
> - ❌ Urgência e FOMO: "Não perca esta proposta!", "Corra!", "Última chance!"
> - ❌ Erros genéricos: "Algo deu errado. Tente novamente."
> - ❌ Jargão técnico sem contexto: "Erro 422", "Token JWT inválido", "Status KYC"
> - ❌ Promessas vagas: "Em breve", "Em alguns dias úteis", "Logo"
> - ❌ Linguagem passiva: "Sua proposta será analisada pelo nosso time"
> - ❌ Valores sem formatação: "45000" ou "R$45.000"
> - ❌ Datas vagas: "em alguns dias úteis", "em breve"
> - ❌ Jargão jurídico solto: "instrumento de cessão" sem explicar o que é
> - ❌ Cenário em minúsculo: "cenário a", "cenario b"
> - ❌ Cessionário identificado: nunca mostrar nome, CPF ou dados do comprador

### 1.3 Vocabulário Controlado

Estes são os termos canônicos do módulo Cedente. Use exatamente estes nomes em toda a interface:

| **Termo canônico** | **Nunca use** | **Contexto** |
|---|---|---|
| Caso | Processo, imóvel cadastrado, operação | A unidade de trabalho do Cedente |
| Dossiê | Documentação, documentos do processo | Conjunto de documentos do caso |
| Cenário A / B / C / D | Opção A, Plano A, Modalidade A | As quatro opções de retorno |
| Valor líquido | Valor final, líquido total, seu dinheiro | O que o Cedente efetivamente recebe |
| Conta Escrow | Conta garantia, conta bloqueada | Conta que retém o valor por 15 dias |
| Fechamento | Conclusão, finalização, encerramento | Evento que confirma o negócio |
| Reversão | Desistência, cancelamento pós-fechamento | Cancelamento dentro dos 15 dias |
| Anuência | Autorização da construtora, aprovação | Permissão formal da construtora |
| Escalonamento | Ajuste de cenário, mudança de opção | Descida de cenário (D→C→B→A) |
| Proposta | Oferta, lance, pedido de compra | Oferta do comprador (anônimo) |
| Guardião do Retorno | IA, assistente, bot, robô | Agente de IA do módulo |
| Cedente | Vendedor, proprietário, você (no heading) | O usuário principal do módulo |

---

## 2. Padrão de Mensagens por Tipo

### 2.1 Mensagens de Erro (Padrão 2 Frases)

> ⚙️ **Regra:** (1) O que aconteceu — específico, sem "Erro" como primeira palavra. (2) O que fazer — ação clara.

| **Contexto** | **Mensagem** |
|---|---|
| E-mail já cadastrado | "Este e-mail já está em uso. Faça login ou recupere sua senha." |
| CPF/CNPJ já cadastrado | "Este CPF já possui uma conta. Faça login ou recupere sua senha." |
| Senha fraca | "Senha muito simples. Use ao menos 8 caracteres, uma letra maiúscula e um número." |
| Link de ativação expirado | "O link de ativação expirou. Solicite um novo link para ativar sua conta." |
| Sessão expirada | "Sua sessão expirou por inatividade. Faça login novamente para continuar." |
| Conta bloqueada | "Conta bloqueada temporariamente por segurança. Tente novamente a partir das [horário]." |
| Credenciais incorretas | "E-mail ou senha incorretos. Verifique seus dados e tente novamente." |
| Duplicidade de imóvel | "Você já tem um caso ativo para este endereço. Acompanhe o andamento em Meus Casos." |
| Cenário não selecionado | "Selecione um cenário de retorno para continuar o cadastro." |
| Valor abaixo do piso | "Este valor está abaixo do mínimo permitido para o Cenário [X] (R$ [piso]). Ajuste o valor ou solicite um escalonamento." |
| Upload — formato inválido | "Formato não aceito. Envie o documento em PDF, JPG ou PNG." |
| Upload — arquivo muito grande | "Arquivo muito grande. O tamanho máximo é de 10 MB." |
| Upload — falha de conexão | "Envio interrompido. Verifique sua conexão e tente novamente." |
| Aceite de proposta — erro técnico | "Não foi possível registrar seu aceite. Tente novamente." |
| Aceite de proposta — 3 falhas | "Estamos resolvendo um problema técnico. Sua proposta está segura e o prazo foi pausado." |
| Contraproposta abaixo do piso | "Contraproposta abaixo do mínimo do Cenário [X]. Informe um valor acima de R$ [piso] para continuar." |
| Cálculo de cenários falhou | "Não foi possível calcular os cenários. Verifique os valores informados e tente novamente." |
| Erro genérico de API | "Algo inesperado aconteceu. Se o problema persistir, entre em contato com o suporte." |
| Falha de conexão | "Sem conexão. Verifique sua internet e tente novamente." |
| ZapSign indisponível | "Assinatura temporariamente indisponível. Tente novamente em instantes." |
| Rascunho não carregado | "Não foi possível carregar o rascunho. Você pode começar um novo cadastro." |

### 2.2 Mensagens de Sucesso (Toasts)

| **Ação** | **Toast** |
|---|---|
| Conta criada | "Conta criada! Verifique seu e-mail para ativar o acesso." |
| Conta ativada | "Conta ativada com sucesso! Bem-vindo ao Repasse Seguro." |
| Imóvel cadastrado | "Imóvel cadastrado com sucesso!" |
| Dado do perfil salvo | "Dados atualizados com sucesso." |
| Preferências de notificação salvas | "Preferências salvas." |
| Documento enviado | "Documento enviado. Aguardando verificação." |
| Proposta aceita | "Proposta aceita! Aguarde o Contrato de Cessão para assinatura." |
| Proposta recusada | "Proposta recusada. Seu imóvel continua disponível para compradores." |
| Contraproposta enviada | "Contraproposta enviada! Aguardando resposta do comprador." |
| Escalonamento solicitado | "Escalonamento solicitado. O Termo de Aceite será enviado para assinatura." |
| Documento assinado | "Documento assinado com sucesso!" |
| Caso cancelado | "Caso cancelado. Você pode cadastrar o imóvel novamente a qualquer momento." |
| Desistência registrada | "Solicitação de desistência registrada. Entraremos em contato em até 48 horas." |
| Sessão renovada | "Sessão renovada." |
| E-mail de confirmação reenviado | "E-mail reenviado com sucesso!" |
| Alteração de e-mail iniciada | "Confirmação enviada para o novo e-mail." |

### 2.3 Mensagens de Estado Vazio (Empty States)

| **Tela** | **Título** | **Subtítulo** | **CTA** |
|---|---|---|---|
| Dashboard — sem casos | "Bem-vindo ao Repasse Seguro" | "Cadastre seu primeiro imóvel para começar a receber propostas de compradores qualificados." | Cadastrar meu primeiro imóvel |
| Meus Casos — sem casos | "Nenhum caso cadastrado" | "Adicione um imóvel para acompanhar o andamento do repasse." | Cadastrar imóvel |
| Meus Casos — sem resultados com filtros | "Nenhum caso com esses filtros" | "Tente ajustar os filtros para ver mais resultados." | Limpar filtros |
| Propostas — sem propostas ativas | "Nenhuma proposta recebida ainda" | "Seu imóvel está disponível para compradores qualificados. Você será notificado quando receber uma proposta." | — |
| Propostas — histórico vazio | "Nenhuma proposta anterior" | "Seu histórico de propostas aparecerá aqui após as primeiras negociações." | — |
| Documentos — nenhum enviado | "Dossiê vazio" | "Envie os documentos do imóvel para que possamos iniciar a análise." | Enviar documentos |
| Assinaturas — nenhuma pendente | "Você está em dia com as assinaturas" | "Documentos para assinar aparecerão aqui quando necessário." | — |
| Financeiro — pré-formalização | "Conta Escrow ainda não criada" | "O valor será depositado na Conta Escrow após a assinatura do Contrato de Cessão." | Ver simulação |
| Histórico de eventos — vazio | "Nenhum evento registrado" | "As atualizações do caso aparecerão aqui." | — |
| Chat — primeiro acesso | "Olá! Sou o Guardião do Retorno." | "Estou aqui 24h para ajudar com simulações, dúvidas e o andamento do seu caso." | — |

### 2.4 Mensagens de Confirmação (Ações Destrutivas)

> ⚙️ **Regra:** toda ação irreversível exige confirmação com resumo do impacto. Nunca "Tem certeza?" isolado.

| **Ação** | **Título do Modal** | **Corpo** | **Botão Confirmar** |
|---|---|---|---|
| Cancelar caso (sem Escrow) | "Cancelar caso" | "Você está prestes a cancelar o caso para [endereço]. Esta ação é irreversível. Todas as propostas ativas serão encerradas." | Confirmar cancelamento |
| Cancelar caso (com Escrow) | "Cancelar caso com Escrow ativo" | "Há uma Conta Escrow ativa para este caso. O cancelamento iniciará o processo de estorno. O prazo de devolução pode ser de até 15 dias úteis." | Confirmar cancelamento |
| Solicitar desistência | "Solicitar desistência" | "A solicitação de desistência cancela o processo de repasse. O valor em Escrow será estornado. Este processo pode levar até 15 dias úteis." | Confirmar desistência |
| Descarte de rascunho | "Descartar rascunho?" | "Seu rascunho de cadastro será removido permanentemente. Esta ação é irreversível." | Sim, descartar |
| Subida de cenário bloqueada | "Não é possível subir de cenário" | "Para tentar um cenário superior, é necessário cancelar este caso e cadastrá-lo novamente. Atenção: o cancelamento encerra todas as propostas ativas." | Cancelar o caso |
| Exclusão de dados LGPD | "Solicitar exclusão de dados" | "Todos os seus dados pessoais serão excluídos. Casos ativos podem ser afetados. O prazo é de até 30 dias úteis." | Solicitar exclusão |

---

## 3. Microcopy por Componente

### 3.1 Botões — CTAs Principais

| **Componente** | **Estado** | **Label** |
|---|---|---|
| Botão de cadastro | Default | "Criar conta" |
| Botão de cadastro | Loading | "Criando conta..." |
| Botão de login | Default | "Entrar" |
| Botão de login | Loading | "Verificando dados..." |
| Botão de avanço no wizard | Default | "Próxima etapa" |
| Botão de avanço no wizard | Bloqueado (10s simulador) | "Analise os cenários por alguns instantes..." |
| Botão de confirmação do cadastro | Default | "Confirmar cadastro" |
| Botão de confirmação do cadastro | Loading | "Criando caso..." |
| Botão de aceite de proposta | Default | "Aceitar proposta" |
| Botão de aceite de proposta | Loading | "Processando aceite..." |
| Botão de recusa de proposta | Default | "Recusar proposta" |
| Botão de contraproposta | Default | "Enviar contraproposta" |
| Botão de contraproposta | Loading | "Enviando..." |
| Botão de escalonamento | Default | "Solicitar escalonamento" |
| Botão de escalonamento | Loading | "Solicitando..." |
| Botão de upload | Default | "Enviar documento" |
| Botão de upload | Rejeitado | "Reenviar documento" |
| Botão de assinatura | Default | "Assinar" |
| Botão de desistência | Default | "Solicitar desistência" |
| Botão de cancelamento | Default | "Cancelar caso" |

### 3.2 Labels de Campos de Formulário

| **Campo** | **Label** | **Placeholder** | **Helper text** |
|---|---|---|---|
| E-mail de cadastro | "E-mail" | "seu@email.com" | N/A |
| Confirmar e-mail | "Confirme o e-mail" | "seu@email.com" | "Deve ser igual ao e-mail acima." |
| Senha | "Senha" | "••••••••" | "Mínimo 8 caracteres, uma letra maiúscula e um número." |
| Confirmar senha | "Confirme a senha" | "••••••••" | "Deve ser igual à senha acima." |
| CPF | "CPF" | "000.000.000-00" | N/A |
| CNPJ | "CNPJ" | "00.000.000/0000-00" | N/A |
| Endereço do imóvel | "Endereço completo" | "Rua, número, bairro, cidade — ou informe o CEP" | "Digite o CEP para preencher automaticamente." |
| Construtora | "Nome da construtora" | "Ex: MRV, Cyrela, Direcional..." | N/A |
| Valor original do contrato | "Valor original do contrato" | "R$ 0,00" | "Valor total do contrato de compra e venda no momento da assinatura." |
| Total pago até hoje | "Total pago até hoje" | "R$ 0,00" | "Soma de todas as parcelas e valores pagos desde o início, incluindo entrada." |
| Saldo devedor | "Saldo devedor atual" | "R$ 0,00" | "Quanto ainda falta pagar para quitar o contrato com a construtora." |
| Valor da contraproposta | "Seu valor" | "R$ 0,00" | "Piso do Cenário [X]: R$ [piso]." |
| Motivo de cancelamento | "Motivo do cancelamento" | "Selecione um motivo" | N/A |
| Justificativa de desistência | "Descreva o motivo" | "Explique o motivo da sua desistência..." | "Mínimo 20 caracteres. [X]/500" |

### 3.3 StatusBadge — Rótulos de Status do Caso

| **Status interno** | **Label exibido ao Cedente** | **Cor (token)** |
|---|---|---|
| Captado | "Cadastro realizado" | `--muted-foreground` |
| Em Triagem | "Em análise" | `--chart-3` (âmbar) |
| Bloqueado | "Pendência identificada" | `--destructive` |
| Qualificado | "Aprovado para oferta" | `--chart-2` (verde suave) |
| Oferta Ativa | "Disponível para compradores" | `--primary` |
| Em Negociação | "Proposta recebida" | `--primary` com pulsação |
| Em Formalização | "Em formalização" | `--chart-3` (âmbar) |
| Fechamento | "Negócio fechado" | `--chart-2` (verde) |
| Pós Fechamento | "Aguardando liberação" | `--chart-2` (verde suave) |
| Em Reversão | "Desistência em análise" | `--chart-3` (âmbar) |
| Em Mediação | "Mediação em andamento" | `--chart-3` (âmbar) |
| Concluído | "Concluído" | `--chart-2` (verde) |
| Cancelado | "Cancelado" | `--muted-foreground` |

### 3.4 Mensagens Explicativas de Status (Painel do Cedente)

| **Status** | **Mensagem exibida no painel** |
|---|---|
| Cadastro realizado | "Seu imóvel foi cadastrado. Envie os documentos e assine o Termo para iniciarmos a análise." |
| Em análise | "Estamos verificando seus documentos. Prazo estimado: 3 dias úteis." |
| Pendência identificada | "Identificamos uma pendência nos seus documentos. Resolva para continuar." |
| Aprovado para oferta | "Seus documentos foram aprovados! Em breve seu imóvel estará disponível para compradores." |
| Disponível para compradores | "Seu imóvel está disponível para compradores qualificados. Você será notificado quando receber uma proposta." |
| Proposta recebida | "Você recebeu uma proposta! Acesse Propostas para avaliar e responder." |
| Em formalização | "Estamos preparando os documentos finais. Fique atento às assinaturas pendentes." |
| Negócio fechado | "Parabéns! O repasse foi formalizado. Seus valores estão retidos na Conta Escrow pelo período de segurança de 15 dias. A distribuição acontece automaticamente ao final desse período." |
| Aguardando liberação | "Seus valores estão retidos na Conta Escrow. O período de segurança termina em [data]. Após essa data, a distribuição será processada automaticamente." |
| Desistência em análise | "Sua solicitação de desistência está sendo analisada. Entraremos em contato em breve." |
| Mediação em andamento | "Estamos mediando a situação entre as partes. Prazo estimado: 10 dias úteis." |
| Concluído | "Repasse concluído! R$ [valor] foi liberado para sua conta." |
| Cancelado | "Este caso foi cancelado. Você pode cadastrar o imóvel novamente a qualquer momento." |

### 3.5 DocumentCard — Labels de Status do Documento

| **Status** | **Label** | **Cor** | **Ação disponível** |
|---|---|---|---|
| Pendente | "Pendente" | `--muted-foreground` | Enviar documento |
| Em Análise | "Em análise" | `--chart-3` (âmbar) | Nenhuma (aguardar) |
| Verificado | "Verificado" | `--chart-2` (verde) | Nenhuma (somente leitura) |
| Rejeitado | "Rejeitado" | `--destructive` | Reenviar documento |

Documento rejeitado exibe motivo em texto âmbar abaixo do card: "[Motivo da rejeição informado pelo Admin]."

### 3.6 ProposalCard — Microcopy

| **Elemento** | **Texto** |
|---|---|
| Rótulo de valor principal | "Você recebe" |
| Rótulo de valor da proposta | "Valor proposto pelo comprador" |
| Rótulo de comissão | "Comissão Repasse Seguro" |
| Rótulo de prazo | "Prazo para responder" |
| Badge urgente (< 24h) | "Urgente" |
| Status da proposta | "Aguardando sua resposta" / "Contraproposta enviada" / "Aceita" / "Recusada" / "Expirada" / "Superada" |
| Rodada de negociação | "Rodada [N] de negociação" |

### 3.7 EscrowPanel — Microcopy

| **Estado** | **Título** | **Subtítulo** |
|---|---|---|
| Não iniciado (pré-formalização) | "Conta Escrow: Não iniciada" | "O valor será depositado pelo comprador após a assinatura do Contrato de Cessão." |
| Aguardando depósito | "Conta Escrow: Aguardando depósito" | "O comprador ainda não confirmou o depósito." |
| Depósito confirmado | "Conta Escrow: Depósito confirmado" | "O valor está sob custódia. O período de segurança começa após o Fechamento." |
| Pós-fechamento (em espera) | "Conta Escrow: Período de segurança" | "Liberação automática em [X] dias." |
| Distribuída | "Conta Escrow: Valor liberado" | "R$ [valor] foi transferido para sua conta." |
| Em reversão | "Conta Escrow: Em processo de reversão" | "O estorno está sendo processado. Prazo: até 15 dias úteis." |
| Estornada | "Conta Escrow: Estornada" | "O valor foi estornado integralmente." |

### 3.8 GuardiaoChat — Microcopy

| **Elemento** | **Texto** |
|---|---|
| Label do agente | "Guardião do Retorno" |
| Badge de tipo | "IA" |
| Placeholder do input | "Pergunte ao Guardião..." |
| Mensagem de boas-vindas (primeiro acesso) | "Olá! Sou o Guardião do Retorno. Estou aqui 24h para ajudar com simulações, dúvidas sobre o seu caso e acompanhamento do andamento. Como posso ajudar?" |
| Sugestões de primeira mensagem | "Qual o status do meu caso?" · "Simular meus cenários de retorno" · "Preciso de ajuda com os documentos" |
| Loading (streaming) | Cursor piscante no balão do Guardião |
| Erro de resposta | "Não consegui processar sua mensagem. Tente novamente." |
| Sem conexão | "Sem conexão. O Guardião estará disponível quando você reconectar." |
| Escalação para humano | "Mensagem encaminhada à equipe. Prazo de resposta: até 2 dias úteis." |
| Modo de cadastro assistido — banner | "Modo: Cadastro Assistido" |
| Cadastro assistido — conclusão | "Coletei todas as informações! Clique abaixo para revisar e confirmar o cadastro do seu imóvel." |

---

## 4. Catálogo de Strings por Módulo

### 4.1 Módulo de Autenticação (T-001 a T-010)

| **ID da String** | **Contexto** | **Texto** |
|---|---|---|
| AUTH-001 | T-001 — Título da tela | "Acesse sua conta" |
| AUTH-002 | T-001 — Link de cadastro | "Ainda não tem conta? Criar conta" |
| AUTH-003 | T-001 — Link de recuperação | "Esqueci minha senha" |
| AUTH-004 | T-002 — Título da tela | "Crie sua conta gratuita" |
| AUTH-005 | T-002 — Link de login | "Já tenho conta. Entrar" |
| AUTH-006 | T-002 — Toggle de tipo | "Pessoa Física" / "Pessoa Jurídica" |
| AUTH-007 | T-003 — Título | "Confirme seu e-mail" |
| AUTH-008 | T-003 — Instrução | "Enviamos um link para [e-mail mascarado]. Clique no link para ativar sua conta." |
| AUTH-009 | T-003 — Botão de reenvio (cooldown) | "Reenviar em [X]s" |
| AUTH-010 | T-003 — Botão de reenvio (disponível) | "Reenviar e-mail" |
| AUTH-011 | T-004 — Título | "Conta ativada com sucesso!" |
| AUTH-012 | T-004 — Subtítulo | "Você será redirecionado para a plataforma em 3 segundos." |
| AUTH-013 | T-004 — Botão | "Ir agora" |
| AUTH-014 | T-005 — Título | "Link expirado" |
| AUTH-015 | T-005 — Instrução | "O link de ativação expirou. Links são válidos por 48 horas." |
| AUTH-016 | T-006 — Título | "Recupere sua senha" |
| AUTH-017 | T-006 — Instrução | "Informe seu e-mail cadastrado e enviaremos um link para criar uma nova senha." |
| AUTH-018 | T-007 — Título | "Verifique sua caixa de entrada" |
| AUTH-019 | T-007 — Instrução | "Enviamos um link de recuperação para o seu e-mail. O link é válido por 1 hora." |
| AUTH-020 | T-008 — Título | "Crie uma nova senha" |
| AUTH-021 | T-009 — Título | "Conta bloqueada temporariamente" |
| AUTH-022 | T-009 — Mensagem | "Você excedeu o número máximo de tentativas. Tente novamente após:" |
| AUTH-023 | T-010 — Título do modal | "Sua sessão está prestes a expirar" |
| AUTH-024 | T-010 — Mensagem | "Por segurança, sua sessão encerrará em:" |

### 4.2 Módulo de Dashboard (T-013, T-014)

| **ID da String** | **Contexto** | **Texto** |
|---|---|---|
| DASH-001 | T-013 — KPI: casos ativos | "Casos ativos" |
| DASH-002 | T-013 — KPI: pendências | "Pendências" |
| DASH-003 | T-013 — KPI: propostas | "Propostas recebidas" |
| DASH-004 | T-013 — KPI: valor estimado | "Valor estimado total" |
| DASH-005 | T-013 — Seção próximos passos | "O que fazer agora" |
| DASH-006 | T-014 — Título (primeiro acesso) | "Pronto para começar?" |
| DASH-007 | T-014 — Subtítulo | "Cadastre seu imóvel em menos de 5 minutos e comece a receber propostas de compradores qualificados." |
| DASH-008 | T-014 — CTA | "Cadastrar meu primeiro imóvel" |
| DASH-009 | T-014 — Checklist item 1 | "Cadastre o imóvel e os valores do contrato" |
| DASH-010 | T-014 — Checklist item 2 | "Envie os documentos do dossiê" |
| DASH-011 | T-014 — Checklist item 3 | "Escolha seu cenário de retorno" |

### 4.3 Módulo de Cadastro de Imóvel (T-024 a T-030)

| **ID da String** | **Contexto** | **Texto** |
|---|---|---|
| CAD-001 | T-024 — Título da tela | "Cadastrar imóvel" |
| CAD-002 | T-024 — Label do stepper, etapa 1 | "Dados do imóvel" |
| CAD-003 | T-025 — Label do stepper, etapa 2 | "Valores do contrato" |
| CAD-004 | T-026 — Label do stepper, etapa 3 | "Simulador de cenários" |
| CAD-005 | T-026 — Texto abaixo do botão bloqueado | "Analise os cenários por alguns instantes antes de avançar." |
| CAD-006 | T-027 — Label do stepper, etapa 4 | "Escolha do cenário" |
| CAD-007 | T-027 — Checkbox de confirmação | "Confirmo que li e compreendo as condições do cenário escolhido." |
| CAD-008 | T-028 — Label do stepper, etapa 5 | "Revisão e confirmação" |
| CAD-009 | T-028 — Destaque do cenário | "Cenário escolhido: [X] — Você recebe R$ [valor]" |
| CAD-010 | T-029 — Título de sucesso | "Imóvel cadastrado com sucesso!" |
| CAD-011 | T-029 — Próximo passo 1 | "Assine o Termo de Cadastro" |
| CAD-012 | T-029 — Próximo passo 2 | "Envie os documentos do dossiê" |
| CAD-013 | T-030 — Texto do banner de rascunho | "Você tem um rascunho salvo para [endereço]. Deseja continuar?" |

### 4.4 Módulo de Cenários e Escalonamento (T-031 a T-034)

| **ID da String** | **Contexto** | **Texto** |
|---|---|---|
| CEN-001 | T-031 — Título | "Alterar seu cenário de retorno" |
| CEN-002 | T-031 — Subtítulo | "Você pode escalonar para um cenário com maior chance de proposta. Escalonamentos são sempre descendentes (D→C→B→A)." |
| CEN-003 | T-031 — Card atual | "Cenário atual" |
| CEN-004 | T-031 — Card novo | "Cenário proposto" |
| CEN-005 | T-032 — Checkbox | "Confirmo que compreendo que o escalonamento é irreversível neste caso." |
| CEN-006 | T-033 — Título | "Pedido enfileirado" |
| CEN-007 | T-033 — Mensagem | "Você tem uma proposta em análise. Seu pedido de escalonamento foi registrado e será processado assim que a proposta for resolvida." |
| CEN-008 | T-034 — Título | "Não é possível subir de cenário" |
| CEN-009 | T-034 — Mensagem | "Para tentar um cenário superior, é necessário cancelar este caso e cadastrá-lo novamente. Atenção: o cancelamento encerra todas as propostas ativas." |

### 4.5 Módulo de Propostas (T-035 a T-039)

| **ID da String** | **Contexto** | **Texto** |
|---|---|---|
| PROP-001 | T-035 — Título da tela | "Propostas" |
| PROP-002 | T-036 — Rótulo principal | "Você recebe" |
| PROP-003 | T-036 — Seção de comissão (colapsada) | "Ver detalhes da comissão" |
| PROP-004 | T-037 — Título | "Aceitar proposta" |
| PROP-005 | T-037 — Aviso | "Ao aceitar, o caso entra em processo de formalização. Esta ação inicia o Contrato de Cessão." |
| PROP-006 | T-038 — Título | "Recusar proposta" |
| PROP-007 | T-038 — Mensagem | "Confirma a recusa desta proposta? O comprador não será notificado do motivo." |
| PROP-008 | T-039 — Título | "Enviar contraproposta" |

### 4.6 Módulo de Simulador de Cenários (T-026)

> Este módulo é sensível financeiramente. Cada palavra influencia a decisão do Cedente.

| **Cenário** | **Nome exibido** | **Tagline** | **Label de dificuldade** |
|---|---|---|---|
| A | "Cenário A — Transferência rápida" | "Transfira o contrato sem receber valores adicionais." | "Maior chance de proposta" |
| B | "Cenário B — Recupere o investido" | "Receba de volta tudo que você pagou até hoje." | "Boa chance de proposta" |
| C | "Cenário C — Valorização moderada" | "Receba o que pagou mais 30% de valorização." | "Chance moderada de proposta" |
| D | "Cenário D — Retorno máximo" | "Receba o que pagou mais 50% de valorização." | "Menor chance de proposta — ideal para imóveis premium" |

Para cada cenário, o card exibe em ordem hierárquica:
1. "**Você recebe**: R$ [valor]" (verde, destaque máximo)
2. "Comissão Repasse Seguro: R$ [valor]" (segundo nível)
3. "Comissão: R$ 0" (apenas Cenário A)

---

## 5. Tom do Guardião do Retorno

### 5.1 Princípios de Voz do Guardião

O Guardião do Retorno é o agente de IA do Cedente. Seu papel é acolher, orientar e simplificar — não impressionar com vocabulário técnico.

| **Princípio** | **Descrição** | **Exemplo** |
|---|---|---|
| Acolhedor | Reconhece que o Cedente pode estar com dúvidas ou apreensivo | "Entendo que essa decisão é importante. Vou te ajudar a entender cada etapa." |
| Objetivo | Respostas diretas ao ponto, sem enrolação | "Seu caso está em 'Em análise'. O prazo estimado é de 3 dias úteis." |
| Explicativo | Explica termos do domínio sem ser condescendente | "O Escrow é uma conta garantia — funciona como um cofre temporário para o dinheiro da transação." |
| Sem urgência artificial | Nunca cria pressão para agir rápido | (nunca usa "Aproveite!", "Corra!", "Não perca!") |
| Sem jargão jurídico | Evita termos técnicos sem contexto | "Instrumento de Cessão = documento que formaliza a transferência do seu contrato" |

### 5.2 Exemplos de Diálogos do Guardião

**Dúvida sobre cenário:**
> Cedente: "Devo escolher o Cenário C ou D?"
>
> Guardião: "Depende do quanto você quer esperar e do tipo do seu imóvel. O Cenário D oferece 50% a mais que o valor pago, mas é mais difícil encontrar comprador. O Cenário C oferece 30% a mais e tem mais demanda. Se você precisa resolver logo, C é mais seguro. Se seu imóvel é premium e pode aguardar, D pode valer a pena. Quer que eu simule os valores para você?"

**Dúvida sobre pendência no dossiê:**
> Cedente: "O que está faltando no meu dossiê?"
>
> Guardião: "No seu dossiê para o caso [endereço], o documento 'Extrato de Pagamentos' está com status 'Rejeitado'. O motivo informado foi: 'Documento ilegível'. Você precisa reenviar uma versão mais nítida. Quer instruções de como tirar uma foto melhor do documento?"

---

## 6. Notificações Push e E-mail

### 6.1 Push Notifications (Mobile)

| **Evento** | **Título** | **Corpo** |
|---|---|---|
| Proposta recebida | "Nova proposta recebida!" | "Você recebeu uma proposta de R$ [valor] para [endereço mascarado]. Responda em até 5 dias úteis." |
| Proposta expirando | "Proposta expirando!" | "Sua proposta para [endereço] expira amanhã. Responda para não perder esta oportunidade." |
| Pendência no dossiê | "Ação necessária no seu dossiê" | "Um documento do caso [endereço] precisa ser reenviado. Veja o que está pendente." |
| Documento assinado | "Documento assinado!" | "O Termo de Cadastro para [endereço] foi assinado com sucesso." |
| Negócio fechado | "Parabéns! Negócio fechado." | "O repasse de [endereço] foi concluído. Seus valores estão na Conta Escrow." |
| Liberação do Escrow | "Valores liberados!" | "R$ [valor] foram transferidos para sua conta. Verifique seu extrato." |
| Rascunho expirando | "Rascunho de cadastro expirando" | "Seu rascunho de imóvel incompleto expira em 5 dias. Finalize para não perder o progresso." |

### 6.2 E-mails Transacionais (Assunto)

| **Evento** | **Assunto do e-mail** |
|---|---|
| Ativação de conta | "Ative sua conta no Repasse Seguro" |
| Recuperação de senha | "Redefinição de senha — Repasse Seguro" |
| Novo documento para assinar | "Documento aguardando sua assinatura — [tipo do documento]" |
| Proposta recebida | "Você recebeu uma proposta! — [endereço mascarado]" |
| Pendência no dossiê | "Ação necessária no seu dossiê — [endereço]" |
| Negócio fechado | "Repasse concluído — valores em Escrow" |
| Liberação de valores | "Seus valores foram liberados — R$ [valor]" |
| Lembrete de rascunho (7 dias) | "Você tem um cadastro incompleto — complete em 23 dias" |
| Rascunho expirado | "Seu cadastro incompleto foi removido" |

---

## 7. Acessibilidade e Internacionalização

### 7.1 Regras de Acessibilidade para Texto

| **Regra** | **Aplicação** |
|---|---|
| `aria-label` em ícones sem texto | Todos os ícones funcionais têm `aria-label` descritivo |
| `aria-describedby` em tooltips | Campos com helper text usam `aria-describedby` |
| Status de loading anunciado | Mensagens de carregamento usam `aria-live="polite"` |
| Erros inline associados ao campo | `aria-describedby` aponta para o `id` do elemento de erro |
| Valores financeiros por extenso | `aria-label="R$ 45.000 vírgula 00"` em valores críticos |
| Contadores regressivos | `aria-live="polite"` com atualização a cada minuto (não a cada segundo) |
| Modais | `role="dialog"` + `aria-modal="true"` + `aria-labelledby` |

### 7.2 Formatação Canônica de Dados

| **Tipo de dado** | **Formato** | **Exemplo** |
|---|---|---|
| Valor monetário | R$ [valor com 2 casas decimais, separador de milhar] | R$ 87.500,00 |
| Data completa | DD/MM/AAAA | 28/03/2026 |
| Data com hora | DD/MM/AAAA às HH:MM | 28/03/2026 às 14:30 |
| Data relativa (até 7 dias) | "há X dias" / "há X horas" | "há 2 dias" |
| Data relativa (mais de 7 dias) | Data completa | 15/03/2026 |
| Prazo em dias úteis | "[N] dias úteis" | "5 dias úteis" |
| Prazo com data-limite | "até [data]" | "até 28/03/2026" |
| Percentual | [N]% | 20% |
| E-mail mascarado | [prefixo****]@[domínio] | jo****@gmail.com |
| Endereço mascarado | "[Rua...] [número mascarado]" | "Rua das Flores, •••" |
