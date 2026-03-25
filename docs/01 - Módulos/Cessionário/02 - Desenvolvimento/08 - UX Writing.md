# 08 - UX Writing

## Módulo Cessionário · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Design, Frontend e Produto |
| **Escopo** | Tom de voz · Padrões de mensagem · Microcopy por componente · Catálogo de strings |
| **Módulo** | Cessionário |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - O tom do módulo Cessionário é: **confiante, claro e sem drama**. A plataforma lida com investimentos reais — cada palavra precisa transmitir segurança sem gerar ansiedade.
> - Erros seguem o padrão de 2 frases: (1) o que aconteceu; (2) o que o usuário deve fazer.
> - O Analista de Oportunidades (IA) fala de forma analítica e orientada a dados. Nunca usa urgência, FOMO ou linguagem emocional.
> - Valores financeiros sempre formatados: R$ 530.000,00 — nunca "530000" nem "R$530.000".
> - Prazos sempre com data completa: "até 05/04/2026" — nunca "em breve" ou "em alguns dias".

---

## 1. Princípios de Tom de Voz

### 1.1 Atributos de Tom

| **Atributo** | **Descrição** | **Na prática** |
|---|---|---|
| **Confiante** | A plataforma sabe o que faz. Não pede desculpas desnecessárias. | "Aguardando confirmação do Admin." (não: "Pedimos desculpas, mas ainda não recebemos...") |
| **Claro** | Sem jargão desnecessário. Termos técnicos de cessão são explicados na primeira vez. | "Escrow (conta-garantia): onde seu dinheiro fica protegido." |
| **Direto** | CTAs acionáveis, sem rodeios. | "Depositar em Escrow" (não: "Clique aqui para prosseguir com o depósito") |
| **Humano** | Fala com pessoas, não com processos. | "Sua verificação foi aprovada!" (não: "Status KYC atualizado para APROVADO") |
| **Sem drama** | Erros são naturais — trate-os como parte do fluxo. | "E-mail já em uso. Tente fazer login ou recupere sua senha." (não: "Erro! Este e-mail já existe no sistema") |

### 1.2 O Que Nunca Fazer

> 🚫 **Anti-padrões de escrita:**
>
> - ❌ Urgência e FOMO: "Não perca!", "Última chance!", "Corra!"
> - ❌ Erros genéricos: "Algo deu errado. Tente novamente."
> - ❌ Jargão técnico sem contexto: "Error 422 Unprocessable Entity"
> - ❌ Promessas de retorno: "Em breve", "Disponível logo", "Em alguns dias"
> - ❌ Linguagem passiva: "Sua proposta será analisada pelo nosso time"
> - ❌ Minúscula em valores: "r$ 530.000,00"
> - ❌ Datas vagas: "em alguns dias úteis"

---

## 2. Padrão de Mensagens por Tipo

### 2.1 Mensagens de Erro (Padrão 2 Frases)

> ⚙️ **Regra:** (1) O que aconteceu — específico, sem "Erro" como primeira palavra. (2) O que fazer — ação clara.

| **Contexto** | **Mensagem** |
|---|---|
| E-mail já cadastrado | "Este e-mail já está em uso. Tente fazer login ou recupere sua senha." |
| Senha fraca | "Senha muito curta. Use ao menos 8 caracteres." |
| KYC — documento ilegível | "Documento não reconhecido. Envie uma foto nítida e com boa iluminação." |
| KYC — selfie não correspondente | "Selfie não correspondeu ao documento. Tire uma nova foto em local bem iluminado." |
| Proposta com valor zero | "O valor da proposta deve ser maior que zero. Informe um valor válido." |
| Oportunidade indisponível | "Esta oportunidade não está mais disponível para novas propostas. Explore outras no marketplace." |
| Limite de propostas simultâneas | "Você atingiu o limite de 3 propostas simultâneas. Aguarde uma resposta antes de enviar uma nova." |
| Rate limit de propostas | "Limite de propostas diárias atingido. Próxima proposta disponível às {horário}." |
| Falha de conexão | "Não foi possível conectar. Verifique sua conexão e tente novamente." |
| Sessão expirada | "Sua sessão expirou por inatividade. Faça login novamente para continuar." |
| Re-autenticação falhou | "Não foi possível confirmar sua identidade. Verifique sua senha e tente novamente." |
| Código de verificação incorreto | "Código incorreto. Verifique e tente novamente." |
| Código expirado | "Alteração não confirmada. O código expirou. Solicite um novo código para salvar." |
| Contraproposta com mesmo valor | "O novo valor deve ser diferente do valor anterior. Altere o valor para enviar a contraproposta." |
| Erro genérico de API | "Algo inesperado aconteceu. Se o problema persistir, entre em contato com o suporte." |

---

### 2.2 Mensagens de Sucesso (Toasts)

| **Ação** | **Toast** |
|---|---|
| Dado do perfil salvo | "Dado atualizado com sucesso." |
| Proposta enviada | "Proposta enviada com sucesso!" |
| Proposta cancelada | "Proposta cancelada." |
| Comprovante de Escrow enviado | "Comprovante enviado. Aguardando confirmação do Admin." |
| Dados bancários copiados | "Dados copiados para a área de transferência." |
| Notificações atualizadas | "Preferências salvas." |

---

### 2.3 Mensagens de Estado Vazio

| **Tela** | **Título** | **Subtítulo** | **CTA** |
|---|---|---|---|
| Marketplace sem oportunidades | "Nenhuma oportunidade no momento" | "Novas oportunidades são publicadas regularmente. Volte em breve." | — |
| Marketplace — sem resultados com filtros | "Nenhuma oportunidade com esses filtros" | "Tente ajustar os filtros para ver mais opções." | Limpar filtros |
| Minhas Propostas — sem propostas | "Você ainda não enviou nenhuma proposta" | "Explore as oportunidades disponíveis e faça sua primeira proposta." | Explorar Oportunidades |
| Negociações — sem negociações | "Nenhuma negociação ativa" | "Suas negociações aparecerão aqui após uma proposta ser aceita." | Explorar Oportunidades |
| Assinaturas — sem documentos | "Nenhum documento pendente" | "Documentos para assinatura aparecerão aqui após o depósito ser confirmado." | Ver Negociações |
| Financeiro — sem transações | "Nenhuma transação registrada" | "Seu histórico financeiro aparecerá aqui após a primeira operação." | Ver Oportunidades |
| Notificações — sem alertas | "Tudo em dia" | "Você será notificado sobre atualizações importantes aqui." | — |

---

### 2.4 Mensagens de Confirmação (Modais)

| **Ação** | **Título do Modal** | **Corpo** | **CTA Primário** | **CTA Secundário** |
|---|---|---|---|---|
| Cancelar proposta | "Cancelar esta proposta?" | "Tem certeza que deseja cancelar sua proposta para {OPR-AAAA-NNNN}? Esta ação não pode ser desfeita." | "Cancelar proposta" (vermelho) | "Manter proposta" |
| Excluir conta | "Excluir sua conta?" | "Esta ação não pode ser desfeita. Seus dados pessoais serão anonimizados em até 48 horas. Dados financeiros serão retidos por 5 anos conforme legislação." | "Confirmar exclusão" (vermelho) | "Cancelar" |
| Revogar consentimento IA | "Revogar uso de dados pela IA?" | "Ao revogar, você deixará de receber recomendações personalizadas. As demais funcionalidades permanecem disponíveis." | "Revogar" | "Manter ativo" |
| Solicitar exportação de dados | "Exportar seus dados?" | "Você receberá um arquivo com todos os seus dados pessoais em até 48 horas, por e-mail e disponível para download na plataforma por 7 dias." | "Exportar dados" | "Cancelar" |

---

### 2.5 Mensagens de Aviso (Banners e Callouts)

| **Contexto** | **Mensagem** |
|---|---|
| KYC não aprovado — tela de oportunidade | "Complete sua verificação para fazer propostas. Acesse Meu Perfil para enviar seus documentos." |
| Prazo de Escrow — 3 dias restantes | "Atenção: prazo de depósito em Escrow vence em {N} dias. Prazo: {data}." |
| Prazo de Escrow — último dia | "Urgente: último dia para depósito em Escrow. Prazo hoje às 23:59." |
| Banner de boas-vindas | "Bem-vindo ao Repasse Seguro! Complete sua verificação para começar a investir." |
| E-mail não pode ser desabilitado | "O e-mail não pode ser desabilitado — é o canal mínimo obrigatório para notificações críticas de prazo." |
| KYC em análise manual | "Sua verificação requer análise adicional. Prazo estimado: até 24 horas úteis." |
| Envio ao ZapSign temporariamente indisponível | "Os documentos estão sendo preparados. Você receberá uma notificação quando estiverem prontos para assinatura." |

---

### 2.6 Labels de Status

| **Status** | **Label ao Usuário** | **Cor** |
|---|---|---|
| Proposta: ENVIADA | "Enviada" | Azul |
| Proposta: EM_ANALISE | "Em análise" | Azul |
| Proposta: ACEITA | "Aceita" | Verde |
| Proposta: RECUSADA | "Recusada" | Vermelho |
| Proposta: EXPIRADA | "Expirada" | Cinza |
| Proposta: CANCELADA | "Cancelada" | Cinza |
| KYC: CADASTRADA | "Verificação pendente" | Amarelo |
| KYC: KYC_EM_ANALISE | "Em análise" | Azul |
| KYC: KYC_APROVADO | "Verificado ✓" | Verde |
| KYC: KYC_REPROVADO | "Reprovado" | Vermelho |
| Negociação: EM_NEGOCIACAO | "Em negociação" | Azul |
| Negociação: AGUARDANDO_DEPOSITO | "Aguardando depósito" | Amarelo |
| Negociação: DEPOSITO_CONFIRMADO | "Depósito confirmado" | Verde |
| Formalização: ASSINATURA_PENDENTE_CESSIONARIO | "Sua assinatura pendente" | Amarelo |
| Formalização: AGUARDANDO_ANUENCIA | "Aguardando anuência" | Azul |
| Formalização: CONCLUIDA | "Concluída ✓" | Verde |
| Operação: CONCLUIDA | "Operação concluída ✓" | Verde |

---

## 3. Microcopy por Componente

### 3.1 Formulário de Proposta

| **Elemento** | **Texto** |
|---|---|
| Label do campo de valor | "Valor da Proposta (R$)" |
| Placeholder | "0,00" |
| Helper text | "Informe o Preço Repasse que deseja pagar pelo imóvel." |
| Label comissão | "Comissão Repasse Seguro (estimativa)" |
| Tooltip comissão | "20% × Δ entre Tabela Atual e Tabela Contrato. Calculada automaticamente sobre o valor informado." |
| Label mensagem | "Mensagem ao Admin (opcional)" |
| Placeholder mensagem | "Justificativas ou perguntas sobre a oportunidade." |
| Contador mensagem | "{N}/500 caracteres" |
| Checkbox confirmação | "Estou ciente dos termos da operação e da comissão calculada." |
| Botão enviar | "Enviar Proposta" |
| Tela de confirmação título | "Proposta enviada com sucesso!" |
| Tela de confirmação subtítulo | "Você será notificado sobre o resultado em até 72 horas úteis." |

---

### 3.2 Instrução de Depósito Escrow

| **Elemento** | **Texto** |
|---|---|
| Título da seção | "Instrução de Depósito" |
| Label breakdown | "Composição do depósito" |
| Label total | "Total a depositar" |
| Seção dados bancários | "Dados da conta Escrow" |
| Nota de segurança | "Estes dados são exclusivos para esta operação. Não compartilhe." |
| Botão copiar | "Copiar dados bancários" |
| Toast após copiar | "Dados copiados para a área de transferência." |
| Label prazo | "Prazo para depósito" |
| Label contador | "Dias restantes" |
| Botão enviar comprovante | "Enviar Comprovante" |
| Label extensão | "Solicitar extensão de prazo" |
| Tooltip extensão | "Extensão de +5 dias úteis. Disponível uma vez por negociação." |

---

### 3.3 KYC Stepper

| **Elemento** | **Texto** |
|---|---|
| Título da tela | "Verificação de Identidade" |
| Subtítulo | "Sua verificação é necessária para fazer propostas. O processo leva menos de 5 minutos." |
| Passo 1 título | "Documento de Identidade" |
| Passo 1 instrução | "Envie o RG ou CNH (frente e verso). O documento deve estar legível e sem reflexos." |
| Passo 2 título | "Comprovante de Endereço" |
| Passo 2 instrução | "Envie uma conta de luz, água ou extrato bancário emitido nos últimos 90 dias." |
| Passo 3 título | "Selfie de Prova de Vida" |
| Passo 3 instrução | "Posicione seu rosto no enquadramento indicado e aguarde a instrução." |
| Botão próximo passo | "Próximo" |
| Botão enviar (último passo) | "Enviar para verificação" |
| Status em análise | "Documentos enviados. Você será notificado sobre o resultado." |
| Tempo estimado automático | "Tempo estimado: até 5 minutos." |
| Tempo estimado manual | "Sua verificação requer análise manual. Prazo: até 24 horas úteis." |

---

### 3.4 Assistente IA — Analista de Oportunidades

| **Elemento** | **Texto** |
|---|---|
| Mensagem de boas-vindas | "Olá! Sou o Analista de Oportunidades. Posso ajudar você a analisar riscos, comparar oportunidades e simular retornos. Como posso ajudar?" |
| Placeholder do campo | "Pergunte ao Analista..." |
| Indicador de carregamento | "Analisando..." |
| Resposta ao perguntar sobre o Cedente | "Não tenho acesso a informações pessoais do Cedente. Posso ajudar com dados da oportunidade, análise de risco e comparativos. O que você gostaria de saber?" |
| Disclaimer (rodapé do chat) | "O Analista de Oportunidades fornece análises baseadas em dados disponíveis. Não constitui recomendação de investimento." |

---

### 3.5 Tela de Transição ZapSign

| **Elemento** | **Texto** |
|---|---|
| Título | "Assinatura Digital Segura" |
| Corpo | "Você será redirecionado para o ambiente seguro de assinatura digital (ZapSign). Ao retornar, sua assinatura será registrada automaticamente." |
| Botão ação | "Continuar para assinatura" |
| Botão voltar | "Voltar" |
| Rodapé | "Powered by ZapSign — assinatura com validade jurídica." |
| Confirmação após retorno | "Assinatura registrada com sucesso." |

---

## 4. Formatação de Dados

| **Tipo** | **Formato Correto** | **Formato Incorreto** |
|---|---|---|
| Valores monetários | R$ 530.000,00 | R$530000 / 530.000 / R$ 530.000 |
| Datas | 05/04/2026 | 05-04-26 / 5 de abril / "em breve" |
| Prazos | "Prazo: 05/04/2026" | "Prazo em 3 dias" |
| Porcentagens | 35% | 35 % / 0.35 |
| Score IA | 8/10 | 8 de 10 / 8,0 |
| Código de oportunidade | OPR-2026-0042 | opr2026-42 / OPR 2026 0042 |
| Tempo de expiração | "Expira às 14:30 de 05/04/2026" | "Expira em breve" |

---

## 5. Glossário de Termos para Usuário Final

| **Termo Técnico** | **Como explicar ao Cessionário** |
|---|---|
| Escrow | "Conta-garantia onde seu pagamento fica protegido até a conclusão da cessão." |
| KYC | "Verificação de identidade — necessária para fazer propostas na plataforma." |
| Δ (Delta) | "Diferença entre o valor atual do imóvel e o valor original do contrato do Cedente." |
| Comissão Comprador | "Taxa da Repasse Seguro: 20% sobre a valorização do imóvel (Δ)." |
| Anuência | "Autorização da construtora para que a cessão seja formalizada." |
| Formalização | "Processo de assinatura digital dos documentos de cessão." |
| Cedente | "A outra parte da operação. Os dados pessoais do Cedente são mantidos anônimos por todo o processo." |
| Contraproposta | "Novo valor proposto por você durante a negociação. Máximo de 3 rodadas." |

---

## 6. Changelog

| **Data** | **Versão** | **Descrição** |
|---|---|---|
| 22/03/2026 | v1.0 | Criação inicial — Pipeline ShiftLabs v9.5 |
