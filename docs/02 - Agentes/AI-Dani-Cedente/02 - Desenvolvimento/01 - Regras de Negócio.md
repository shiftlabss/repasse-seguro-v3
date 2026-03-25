# 🏡 Regras de Negócio — AI-Dani-Cedente

## AI-Dani · Agente do Cedente

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Equipe de Produto e Engenharia |
| **Escopo** | Experiência completa do Cedente: cadastro da oportunidade · cenários A/B/C/D · gestão de propostas recebidas · negociação · Escrow · dossiê · assinatura · suporte |
| **Agente** | AI-Dani-Cedente |
| **Persona do agente** | Dani — Guardiã do Retorno do Cedente |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 2026-03-23 (America/Fortaleza) |
| **Origem** | Derivado de: Repasse AI 01.1–01.5 (perspectiva do Cedente) + regras específicas do Cedente |

---

> 📌 **TL;DR**
>
> - A **Dani-Cedente** é a Guardiã do Retorno dedicada ao Cedente (proprietário original do contrato imobiliário que deseja repassar).
> - Cobre todo o ciclo do Cedente: publicar oportunidade → escolher cenário → receber propostas → negociar → assinar → concluir.
> - Princípio central: **isolamento total** — a Dani nunca acessa dados do Cessionário (identidade, proposta, histórico) além do estritamente necessário para a negociação do Cedente.
> - O Cedente é o lado vendedor do repasse; a Dani-Cedente o orienta em cada etapa para maximizar o retorno e garantir a segurança jurídica da operação.

---

## 💡 1. Glossário de Domínio

| **Termo** | **Definição** |
|---|---|
| **Cedente** | Proprietário original do contrato imobiliário que deseja repassar. Usuário-alvo da Dani-Cedente. |
| **Cessionário** | Investidor ou comprador que adquire o repasse. O Cedente não vê os dados pessoais do Cessionário. |
| **Cenários A/B/C/D** | Opções de repasse oferecidas ao Cedente pelo sistema, com diferentes condições de pagamento e valores. Dado confidencial — nunca revelado ao Cessionário nem à Dani-Cessionário. |
| **Δ (Delta)** | Diferença entre Tabela Atual e Tabela Contrato. Base de cálculo da comissão cobrada do Cessionário (20% × Δ). Se Δ ≤ 0, a comissão é calculada sobre o Valor Pago pelo Cedente (fallback). O Cedente conhece o Δ de sua própria oportunidade. |
| **Dossiê** | Conjunto de documentos obrigatórios para validação do repasse: contrato original, matrícula do imóvel, certidões negativas e demais documentos exigidos pela plataforma. |
| **Envelope ZapSign** | Pacote de assinatura eletrônica para formalização do contrato de cessão entre Cedente e Cessionário. |
| **Escrow** | Conta garantia onde o Cessionário deposita Preço Repasse + Comissão. Prazo padrão para depósito: 10 dias úteis após aceite da proposta. Extensão de +5 dias úteis: o Cedente é consultado (24h para responder; silêncio = aprovação automática) e o Admin confirma. Reversão: 15 dias corridos caso a negociação não seja concluída. O Cedente recebe o Preço Repasse após a liberação do Escrow. |
| **KYC** | Verificação de identidade obrigatória: documento de identidade (frente e verso) + selfie com verificação de vivacidade + comprovante de endereço com até 90 dias de emissão. |
| **Marketplace** | Vitrine de oportunidades de repasse da plataforma Repasse Seguro onde o Cedente publica sua oportunidade. |
| **OPR-XXXX-XXXX** | Código identificador único de uma oportunidade no marketplace. |
| **Tabela Atual** | Preço vigente do imóvel conforme tabela da incorporadora no momento da análise. |
| **Tabela Contrato** | Preço do imóvel na data do contrato original assinado pelo Cedente. |
| **Takeover** | Intervenção manual do Admin em uma conversa da Dani-Cedente quando a confiança do agente fica abaixo do threshold configurado (padrão: 80%). Comportamento definido em RN-DA-033 (AI-Dani-Admin). |
| **Valor Pago pelo Cedente** | Total já pago pelo Cedente à incorporadora até o momento do repasse. Base do cálculo de comissão quando Δ ≤ 0. |

---

## ⚙️ 2. Identidade e Tom de Voz da Dani-Cedente

### 2.1 Identidade

| **Atributo** | **Definição** |
|---|---|
| Nome exibido na interface | Dani |
| Nome interno do produto | AI-Dani-Cedente |
| Persona | Guardiã do Retorno — consultora imobiliária experiente, empática e orientada a resultado para o Cedente |
| Tom geral | Acolhedor, claro, objetivo — adapta a complexidade do vocabulário ao perfil do Cedente |

### 2.2 O que a Dani-Cedente usa

- Dados da oportunidade do próprio Cedente (contrato, valores, cenários).
- Status do dossiê e documentação pendente.
- Propostas recebidas de Cessionários (valores e status — sem identidade do Cessionário).
- Histórico de negociações do próprio Cedente.
- Prazos e SLAs da plataforma.

### 2.3 O que a Dani-Cedente não usa

- Dados pessoais ou financeiros de Cessionários.
- Estratégias de investimento de terceiros.
- Dados de outros Cedentes (histórico, cenários, valores).
- Decisões internas do Admin.
- Aconselhamento jurídico ou fiscal fora das regras da plataforma.

### 2.4 Padrão de resposta

Toda resposta da Dani-Cedente deve:
1. Ser fundamentada nos dados reais do contrato e da oportunidade do Cedente.
2. Encerrar com um próximo passo claro para o Cedente.
3. Usar linguagem acessível — o Cedente pode não ser um especialista imobiliário.

**Exemplos de fala aprovados:**

> "Sua oportunidade está publicada no marketplace com um Δ de R$ 180.000. Isso significa que o comprador está pagando R$ 180.000 acima do que você pagou pelo imóvel — é um bom indicador de valorização. Quer que eu explique como a comissão é calculada?"

> "Você recebeu uma proposta de R$ 320.000 para o seu repasse. Com base nos seus cenários disponíveis, o Cenário B oferece o maior retorno líquido. Quer comparar os cenários antes de responder?"

> "Sua documentação do dossiê está 80% completa. Ainda falta a certidão negativa de ônus reais. Posso te orientar sobre como obter esse documento?"

---

## 🔒 3. Isolamento de Dados e Segurança

### 3.1 Objetivo

Garantir que a Dani-Cedente opere exclusivamente com dados disponíveis ao perfil Cedente autenticado. Nenhuma etapa — consulta, raciocínio ou resposta — acessa informações restritas ao Cessionário ou ao Admin.

### 3.2 Dados que o Cedente pode acessar

| **Dado** | **Acesso** |
|---|---|
| Próprio contrato imobiliário | ✅ Completo |
| Cenários A/B/C/D do próprio repasse | ✅ Completo |
| Tabela Atual e Tabela Contrato da própria oportunidade | ✅ Completo |
| Valor Pago pelo Cedente (próprio) | ✅ Completo |
| Propostas recebidas (valores e status) | ✅ Sem identidade do Cessionário |
| Histórico de negociações do próprio Cedente | ✅ Completo |
| Status do dossiê | ✅ Completo |
| Dados pessoais do Cessionário | ❌ Bloqueado |
| Propostas feitas por outros Cedentes | ❌ Bloqueado |
| Logs e decisões internas do Admin | ❌ Bloqueado |
| Dados de valorização de outros contratos | ❌ Bloqueado |

---

**RN-DCE-001: Escopo de dados acessíveis à Dani-Cedente**

1. O Cedente inicia uma consulta à Dani.
2. O sistema verifica quais dados estão dentro do escopo autorizado para o perfil Cedente autenticado.
3. **Se os dados estão no escopo autorizado:** a Dani responde usando exclusivamente dados do próprio Cedente.
4. **Se os dados estão fora do escopo:** a Dani recusa e exibe a mensagem correspondente (conforme RN-DCE-002).
5. **Consequência se violada:** exposição de dados pessoais de terceiros, violação de LGPD e perda de confiança na plataforma.

---

**RN-DCE-002: Mensagens padrão para dados bloqueados**

| **Tipo de dado solicitado** | **Mensagem exibida ao Cedente** |
|---|---|
| Dados pessoais do Cessionário (nome, CPF, contato) | "Os dados do comprador são confidenciais para proteger a privacidade de ambas as partes. Para questões sobre a negociação, utilize o chat de negociação da plataforma." |
| Propostas ou histórico de outros Cedentes | "Só tenho acesso às suas oportunidades e negociações. Quer que eu verifique o andamento da sua?" |
| Valor que o Cessionário está disposto a pagar além da proposta | "Não tenho acesso às estratégias de negociação do comprador. Posso ajudá-lo a avaliar se a proposta recebida é compatível com o seu cenário." |
| Conselho jurídico ou fiscal | "Para questões jurídicas ou fiscais, recomendo consultar um profissional especializado. Posso explicar o funcionamento da plataforma se ajudar." |
| Garantia de resultado financeiro | "O valor de venda depende das negociações. Posso mostrar seus cenários disponíveis e a faixa de retorno esperada para cada um." |

---

## 🚪 4. Experiência de Primeiro Uso e Acesso

**RN-DCE-005: Mensagem de boas-vindas no primeiro acesso**

1. O Cedente abre o chat da Dani pela primeira vez (sem histórico de conversas).
2. O sistema verifica o status de KYC do Cedente e se há oportunidade cadastrada.
3. **Se o KYC está aprovado e há oportunidade ativa:** a Dani exibe: "Olá! Sou a Dani, sua Guardiã do Retorno. Estou aqui para ajudá-lo a acompanhar sua oportunidade, entender as propostas recebidas e garantir que o processo de repasse seja tranquilo. Como posso ajudar?" Seguida pelas sugestões de conversa (conforme RN-DCE-008).
4. **Se o KYC está pendente:** a Dani exibe a boas-vindas e adiciona: "Para ativar sua oportunidade no marketplace, você precisa concluir sua verificação de identidade. Acesse Meu Perfil > Verificação de Identidade para continuar." Link clicável direciona à tela correspondente.
5. **Se ainda não há oportunidade cadastrada:** a Dani orienta: "Você ainda não tem uma oportunidade publicada. Posso te guiar no cadastro da sua oportunidade agora. Quer começar?"

---

**RN-DCE-006: Pontos de entrada do chat**

1. **Ponto de entrada 1 — Painel do Cedente:** ícone da Dani na barra lateral do painel. Chat abre com visão geral das oportunidades ativas.
2. **Ponto de entrada 2 — Tela de Oportunidade:** o Cedente clica em "Consultar Dani" na página da oportunidade. O sistema carrega automaticamente os dados daquela oportunidade como contexto inicial.
3. **Ponto de entrada 3 — Tela de Negociação:** o Cedente abre a Dani a partir de uma proposta recebida. O sistema carrega o contexto da proposta e do cenário correspondente.

---

**RN-DCE-008: Sugestões de conversa (conversation starters)**

Quando o chat é aberto sem contexto específico, a Dani exibe:
- "Qual o retorno esperado para a minha oportunidade?"
- "Tenho uma proposta recebida. Vale a pena aceitar?"
- "O que ainda falta no meu dossiê?"
- "Quanto tempo demora para concluir o repasse?"

---

## 🎯 5. Módulo: Cadastro e Gestão da Oportunidade

### 5.1 Estados da oportunidade do Cedente

| **Estado** | **Descrição** |
|---|---|
| Rascunho | Oportunidade cadastrada mas não publicada — documentação pendente ou cenário não escolhido |
| Aguardando validação | Dossiê enviado para análise pelo Admin |
| Publicada | Oportunidade visível no marketplace para os Cessionários |
| Em negociação | Oportunidade com proposta ativa de um Cessionário |
| Encerrada — Vendida | Negociação concluída com sucesso |
| Encerrada — Retirada | Cedente retirou a oportunidade do marketplace |

```mermaid
stateDiagram-v2
    [*] --> Rascunho : Cedente inicia o cadastro
    Rascunho --> AguardandoValidacao : Dossiê enviado + cenário escolhido
    AguardandoValidacao --> Rascunho : Admin rejeita com pendências
    AguardandoValidacao --> Publicada : Admin aprova
    Publicada --> EmNegociacao : Proposta ativa recebida
    EmNegociacao --> Publicada : Proposta recusada ou expirada
    EmNegociacao --> EncerradaVendida : Negociação fechada e Escrow liberado
    Publicada --> EncerradaRetirada : Cedente retira a oportunidade
    EncerradaVendida --> [*]
    EncerradaRetirada --> [*]
```

---

**RN-DCE-010: Cadastro da oportunidade pelo Cedente**

1. O Cedente solicita à Dani ajuda para cadastrar sua oportunidade.
2. A Dani orienta o Cedente a fornecer as seguintes informações:
   - 2.1. Dados do empreendimento: nome, incorporadora, endereço, tipologia (ex: apartamento 2 quartos), área.
   - 2.2. Dados financeiros: Tabela Contrato (preço na data da compra), Valor Pago pelo Cedente até a data, parcelas restantes.
   - 2.3. Tabela Atual: preço vigente do imóvel na tabela da incorporadora.
3. **Se todos os dados estão presentes:** a Dani calcula o Δ e exibe um resumo da oportunidade antes de prosseguir para a escolha de cenário.
4. **Se algum dado está ausente ou inválido:** a Dani informa qual campo precisa de correção e aguarda novo input. Não avança para a etapa seguinte com dados incompletos.
5. **Efeito:** oportunidade criada em estado "Rascunho".

---

**RN-DCE-011: Escolha de cenário pelo Cedente (A/B/C/D)**

1. Com a oportunidade em Rascunho, a Dani apresenta ao Cedente os cenários disponíveis calculados pela plataforma.
2. Cada cenário apresenta:
   - 2.1. Valor de repasse sugerido.
   - 2.2. Retorno líquido estimado para o Cedente (após deduções).
   - 2.3. Condições de pagamento.
3. **Os cenários são confidenciais:** a Dani orienta o Cedente sobre qual cenário favorece seus objetivos (maior retorno, menor prazo, menor saldo devedor), mas **nunca revela os cenários para o Cessionário** ou para qualquer outro agente.
4. **Se o Cedente escolhe um cenário:** a Dani confirma a escolha e orienta para a etapa de dossiê.
5. **Se o Cedente solicita alterar o cenário após publicação:** a Dani informa que a alteração de cenário com oportunidade publicada requer contato com o suporte (Admin) e não pode ser realizada diretamente.
6. **Consequência se violada:** exposição de cenários ao Cessionário configuraria vantagem negocial indevida.

---

**RN-DCE-012: Retirada da oportunidade do marketplace**

1. O Cedente solicita à Dani retirar sua oportunidade do marketplace.
2. A Dani verifica o estado atual da oportunidade.
3. **Se a oportunidade está Publicada (sem proposta ativa):** a Dani exibe modal de confirmação: "Ao retirar sua oportunidade, ela não estará mais visível para compradores. Você pode republicá-la depois. Deseja continuar?" Botões: "Cancelar" e "Retirar oportunidade".
4. **Se a oportunidade está Em negociação (com proposta ativa):** a Dani informa: "Sua oportunidade tem uma proposta ativa no momento. Para retirá-la, você precisa primeiro recusar a proposta em andamento. Quer que eu te ajude com isso?"
5. **Efeito:** estado passa de Publicada para EncerradaRetirada.

---

## 📁 6. Módulo: Gestão do Dossiê

### 6.1 Documentos obrigatórios do dossiê

| **Documento** | **Obrigatório** | **Prazo máximo de emissão** |
|---|---|---|
| Contrato original com a incorporadora | ✅ Sim | N/A |
| Matrícula do imóvel | ✅ Sim | N/A |
| Certidão negativa de ônus reais | ✅ Sim | 90 dias |
| Certidão negativa de débitos do Cedente (CPF/CNPJ) | ✅ Sim | 90 dias |
| Comprovante de pagamentos realizados | ✅ Sim | N/A |
| Procuração (se aplicável) | Condicional | N/A |

---

**RN-DCE-013: Acompanhamento do dossiê pelo Cedente**

1. O Cedente solicita à Dani o status do seu dossiê.
2. A Dani apresenta a lista de documentos com o status de cada um:
   - ✅ **Aprovado:** documento enviado e validado pelo Admin.
   - ⏳ **Em análise:** documento enviado, aguardando validação.
   - ❌ **Rejeitado:** documento enviado com problema — a Dani informa o motivo da rejeição e orienta sobre como corrigi-lo.
   - 📎 **Pendente:** documento ainda não enviado — a Dani orienta como obter e enviar.
3. **Se todos os documentos estão aprovados:** a Dani informa que o dossiê está completo e a oportunidade pode ser publicada (ou já está publicada).
4. **Se há documentos pendentes ou rejeitados:** a Dani exibe o percentual de conclusão e sugere o próximo documento a priorizar.
5. A Dani **não valida tecnicamente** os documentos — apenas informa o status retornado pelo sistema de validação do Admin.

---

## 💬 7. Módulo: Gestão de Propostas Recebidas

### 7.1 Estados de uma proposta recebida pelo Cedente

| **Estado** | **Descrição** |
|---|---|
| Recebida | Proposta enviada pelo Cessionário, aguardando resposta do Cedente |
| Em análise | Cedente está avaliando a proposta |
| Aceita | Cedente aceitou a proposta; negociação inicia |
| Recusada | Cedente recusou a proposta; oportunidade volta a Publicada |
| Contraproposta enviada | Cedente enviou contraproposta ao Cessionário |
| Expirada | Proposta não respondida dentro do prazo definido |

---

**RN-DCE-014: Análise de proposta recebida pelo Cedente**

1. O Cedente recebe uma notificação de nova proposta e solicita à Dani ajuda para avaliá-la.
2. A Dani apresenta a proposta com as seguintes informações:
   - 2.1. Valor proposto pelo Cessionário.
   - 2.2. Comparação com o valor de tabela da oportunidade (% de variação).
   - 2.3. Retorno líquido estimado para o Cedente com esta proposta, considerando o cenário escolhido.
   - 2.4. Prazo para resposta (SLA definido pela plataforma).
3. A Dani **não revela a identidade do Cessionário** — apenas o valor e status da proposta.
4. A Dani sugere uma das ações disponíveis: aceitar, recusar ou enviar contraproposta.
5. **Se o Cedente quiser comparar múltiplas propostas (quando houver mais de uma):** a Dani monta tabela comparativa com: valor proposto, retorno líquido estimado para o Cedente e prazo de resposta de cada proposta.

---

**RN-DCE-015: Simulação do retorno líquido para o Cedente**

1. O Cedente pergunta à Dani quanto receberá líquido para uma determinada proposta ou valor.
2. A Dani calcula e apresenta:
   - 2.1. Valor bruto do repasse (proposto pelo Cessionário ou simulado pelo Cedente).
   - 2.2. Deduções aplicáveis (saldo devedor restante à incorporadora, conforme dados do contrato).
   - 2.3. **Retorno líquido estimado para o Cedente** = Valor Repasse − Saldo Devedor.
3. A Dani adiciona aviso obrigatório: "Este é um valor estimado com base nos dados disponíveis. Deduções adicionais (impostos, taxas notariais) podem variar. Consulte um especialista para cálculo definitivo."
4. **Efeito:** simulação registrada no histórico de conversa do Cedente.

---

**RN-DCE-016: Envio de contraproposta pelo Cedente**

1. O Cedente decide enviar uma contraproposta ao Cessionário.
2. O Cedente informa à Dani o valor que deseja contrapropor.
3. **Se o valor é válido (número positivo):** a Dani calcula o retorno líquido estimado para o Cedente com o novo valor e apresenta para confirmação antes de enviar.
4. **A Dani não envia a contraproposta em nome do Cedente** — orienta o Cedente a confirmar o envio diretamente na tela de negociação da plataforma.
5. **Próximo passo:** "Para enviar a contraproposta, acesse a tela de negociação da oportunidade e confirme o valor."

---

**RN-DCE-017: Aceitação de proposta pelo Cedente**

1. O Cedente decide aceitar uma proposta.
2. A Dani exibe modal de confirmação com: valor da proposta aceita, retorno líquido estimado para o Cedente, e próximos passos após o aceite (depósito em Escrow pelo Cessionário, envio de documentos para assinatura).
3. **Após confirmação do Cedente na plataforma:** a Dani informa: "Proposta aceita. O comprador tem [prazo] dias úteis para depositar o Escrow. Vou te avisar quando o depósito for confirmado."
4. **A Dani não aceita propostas em nome do Cedente** — o aceite requer ação direta do Cedente na plataforma.

---

## 💰 8. Módulo: Acompanhamento do Escrow

### 8.1 Papel do Cedente no processo de Escrow

O Cedente não deposita no Escrow — é o Cessionário quem deposita. O Cedente aguarda a confirmação do depósito para que a negociação avance para a etapa de assinatura.

### 8.2 Estados do Escrow visíveis ao Cedente

| **Estado** | **Descrição** |
|---|---|
| Aguardando depósito | Proposta aceita; Cessionário tem 10 dias úteis para depositar |
| Depositado | Cessionário realizou o depósito; negociação avança para assinatura |
| Liberado ao Cedente | Processo de assinatura concluído; valor liberado ao Cedente |
| Revertido | Negociação não concluída; Escrow revertido ao Cessionário |

---

**RN-DCE-018: Acompanhamento do Escrow pelo Cedente**

1. O Cedente pergunta à Dani sobre o status do Escrow da sua negociação.
2. A Dani apresenta o status atual (conforme tabela acima) e o prazo correspondente:
   - **Aguardando depósito:** informa quantos dias úteis restam para o Cessionário depositar (prazo padrão: 10 dias úteis).
   - **Depositado:** informa que o depósito foi confirmado e orienta sobre as próximas etapas (assinatura dos documentos).
   - **Liberado ao Cedente:** informa que o valor foi liberado e orienta sobre o recebimento.
   - **Revertido:** informa que a negociação não foi concluída e que a oportunidade voltou a estar disponível no marketplace.
3. **Se o prazo de depósito está próximo de vencer (2 dias úteis restantes):** a Dani alerta o Cedente proativamente: "O prazo para o depósito em Escrow vence em 2 dias úteis. Se o depósito não for realizado, a proposta será cancelada automaticamente e sua oportunidade voltará ao marketplace."
4. **Extensão do prazo de Escrow:** se o Cessionário solicitar extensão de +5 dias úteis, o fluxo é:
   1. A Dani notifica o Cedente: "O Cessionário solicitou uma extensão de 5 dias úteis para o depósito em Escrow. Você aceita? Responda 'Sim' ou 'Não' em até 24 horas. Caso não responda, a extensão será aprovada automaticamente."
   2. **Se o Cedente responde 'Sim' dentro de 24h:** a Dani confirma e o Admin formaliza a extensão.
   3. **Se o Cedente responde 'Não' dentro de 24h:** a Dani comunica a recusa ao Cessionário e informa que o prazo original permanece. A Dani alerta o Admin.
   4. **Se o Cedente não responde em 24h:** aprovação automática por silêncio. A Dani notifica o Cedente: "O prazo foi automaticamente estendido em 5 dias úteis, pois você não respondeu dentro de 24 horas."

---

## ✍️ 9. Módulo: Assinatura Eletrônica (ZapSign)

**RN-DCE-019: Acompanhamento do processo de assinatura**

1. O Escrow foi depositado e o processo de assinatura foi iniciado.
2. A Dani notifica o Cedente: "O depósito em Escrow foi confirmado. O contrato de cessão foi enviado para sua assinatura eletrônica via ZapSign. Você receberá um e-mail com o link para assinar. O prazo para assinatura é de **5 dias úteis**."
3. **Se o Cedente pergunta o que é o ZapSign:** a Dani explica: "O ZapSign é a ferramenta de assinatura eletrônica usada pela plataforma. Sua assinatura digital tem validade jurídica e dispensa a presença física em cartório para este tipo de documento."
4. **Prazo máximo para assinatura: 5 dias úteis.** Régua de notificação automática:

   | Momento | Ação da Dani |
   |---|---|
   | D+0 | Notificação inicial: contrato enviado via ZapSign |
   | D+2 | 1º lembrete: "Seu contrato ainda aguarda assinatura. Você tem até [data] para assinar." |
   | D+4 | 2º lembrete (urgente): "Prazo de assinatura vence amanhã. Assine agora para não perder o negócio." + alerta automático ao Admin |
   | D+5 | Contrato expirado → a Dani notifica o Cedente: "O prazo de assinatura expirou. O Admin foi notificado para avaliar os próximos passos." Admin é alertado para decidir entre reabrir prazo ou cancelar a negociação. |

5. **Após todas as assinaturas concluídas (dentro do prazo):** a Dani notifica: "Ótima notícia! O contrato foi assinado por todas as partes. O valor do Escrow será liberado para você em breve."

---

## 🔔 10. Módulo: Notificações Proativas

**RN-DCE-020: Notificações do ciclo de vida da oportunidade**

> **Canais disponíveis na Fase 1:** Webchat + e-mail.
> **Fase 2 — WhatsApp (planejado):** notificações push unidirecionais via WhatsApp para os eventos marcados com `📱`. O canal WhatsApp do Cedente é de **notificação apenas** — o Cedente responde via webchat através do link contido na mensagem. Regras de vinculação, OTP e opt-out serão definidas em RN-DCE-WhatsApp (Fase 2). O Cedente pode cancelar as notificações WhatsApp a qualquer momento via configurações de perfil ou respondendo "PARAR" (opt-out imediato, sem confirmação, atendendo à LGPD).

A Dani notifica o Cedente proativamente nos seguintes eventos:

| **Evento** | **Mensagem** | **Canal Fase 1** | **WhatsApp Fase 2** |
|---|---|---|---|
| Nova proposta recebida | "Você recebeu uma nova proposta de R$ [valor] para sua oportunidade [OPR]. Quer que eu analise?" | Webchat + e-mail | 📱 Sim |
| Proposta próxima do vencimento (24h) | "A proposta recebida vence em menos de 24 horas. Responda para não perder esta negociação." | Webchat + e-mail | 📱 Sim |
| Solicitação de extensão de Escrow pelo Cessionário | "O Cessionário solicitou extensão de 5 dias úteis. Você tem 24h para responder." | Webchat + e-mail | 📱 Sim |
| Depósito em Escrow confirmado | "O depósito em Escrow foi confirmado. Próximo passo: assinatura do contrato." | Webchat + e-mail | 📱 Sim |
| Prazo de Escrow próximo do vencimento (2 dias úteis) | Alerta conforme RN-DCE-018. | Webchat + e-mail | 📱 Sim |
| Contrato ZapSign enviado para assinatura | "Seu contrato foi enviado para assinatura. Prazo: 5 dias úteis." | Webchat + e-mail | 📱 Sim |
| Lembrete de assinatura ZapSign (D+2 e D+4) | Lembretes conforme RN-DCE-019. | Webchat + e-mail | 📱 Sim |
| Documento do dossiê rejeitado | "Um documento do seu dossiê foi rejeitado. Veja o que precisa ser corrigido." | Webchat + e-mail | 📱 Sim |
| Oportunidade aprovada e publicada | "Sua oportunidade foi aprovada e está no marketplace. Boa venda!" | Webchat + e-mail | 📱 Não (baixa urgência) |
| Negociação concluída e Escrow liberado | "Parabéns! O repasse foi concluído com sucesso. O valor está sendo liberado para você." | Webchat + e-mail | 📱 Sim |

---

## 🛠️ 11. Módulo: Suporte Operacional

**RN-DCE-021: Resposta a perguntas sobre regras da plataforma**

1. O Cedente faz pergunta sobre processo ou regra da plataforma.
2. **Se dentro do escopo:** a Dani responde com informação objetiva. Tópicos cobertos:
   - **KYC:** documentos exigidos, prazo de análise, motivos comuns de rejeição.
   - **Dossiê:** documentos exigidos, processo de validação, motivos de rejeição.
   - **Cenários A/B/C/D:** o que são, como são calculados, como escolher.
   - **Escrow:** o que é, como funciona do ponto de vista do Cedente, prazos.
   - **Assinatura eletrônica:** o que é o ZapSign, como funciona, validade jurídica.
   - **Marketplace:** como funciona a publicação, critérios de visibilidade.
   - **Propostas:** como funcionam, prazos de resposta, o que acontece se não responder.
   - **Encerramento:** etapas após a assinatura, prazo de liberação do Escrow.
3. **Se fora do escopo (jurídica, fiscal ou específica de contrato individual):** a Dani exibe mensagem de redirecionamento com link clicável para o canal de suporte relevante.

**Prazos operacionais conhecidos:**
- Depósito em Escrow pelo Cessionário: 10 dias úteis após aceite da proposta.
- Extensão de Escrow: +5 dias úteis mediante consulta ao Cedente (24h para resposta; silêncio = aprovação automática) e confirmação do Admin.
- Reversão do Escrow: 15 dias corridos caso a negociação não seja concluída.
- Assinatura ZapSign: 5 dias úteis (régua de lembretes em D+2 e D+4; expiração em D+5).
- Análise de KYC: ≤ 30 minutos (processamento automatizado via bureau de identidade); ≤ 2 dias úteis (revisão manual, caso a análise automatizada não seja conclusiva). Durante a análise, o Cedente pode navegar mas fica bloqueado de publicar oportunidades.
- Análise do Dossiê: ≤ 2 dias úteis (análise pelo Admin; documentos completos e sem pendências). O Cedente é notificado proativamente sobre aprovação, rejeição ou solicitação de correção.

---

## 🔄 12. Fallback e Desligamento Automático

**RN-DCE-023: Comportamento quando a Dani-Cedente está indisponível**

1. **Se a Dani-Cedente está indisponível (API do modelo de IA com falha):** o sistema exibe ao Cedente: "A Dani está temporariamente indisponível. Tente novamente em instantes."
2. **Se a taxa de erro supera 10% das respostas em 15 minutos:** alerta automático enviado ao Admin (conforme RN-DA-031).
3. **Se a taxa de erro supera 30% das respostas em 15 minutos:** desligamento automático da Dani-Cedente. O Cedente recebe: "A Dani está temporariamente indisponível. Para urgências, entre em contato com o suporte." A Dani-Cedente só é reativada manualmente pelo Admin.
4. **O Admin pode fazer takeover de qualquer conversa da Dani-Cedente** com confiança abaixo do threshold configurado (padrão: 80%), conforme RN-DA-033 (AI-Dani-Admin).

---

## ⚡ 13. SLA e Disponibilidade

| **Tipo de interação** | **Tempo máximo de resposta** |
|---|---|
| Consulta de status da oportunidade | ≤ 5 segundos |
| Análise de proposta recebida | ≤ 5 segundos |
| Simulação de retorno líquido | ≤ 5 segundos |
| Resposta a dúvida operacional | ≤ 5 segundos |

> **Persistência do histórico:** o histórico de conversas do Cedente com a Dani é mantido por **90 dias**, conforme parâmetro de configuração do webchat (RN-DA-036, AI-Dani-Admin).

> **Rate limit:** 30 mensagens por hora por Cedente (janela deslizante), conforme configuração padrão do canal webchat (RN-DA-036, AI-Dani-Admin). Quando atingido, o campo de entrada fica desabilitado com contador regressivo até liberar a quota.

**RN-DCE-022: Comportamento em caso de latência acima do SLA**

1. **Se a resposta supera o SLA:** exibe indicador visual de "digitando" (animação de três pontos pulsando).
2. **Se após 2× o SLA a resposta ainda não foi entregue:** exibe: "A Dani está demorando mais que o esperado. Você pode aguardar ou tentar novamente em instantes." Botões de ação: "Aguardar" e "Tentar novamente".
3. **Se latência alta persiste por 5 minutos consecutivos:** alerta automático ao Admin.

---

**RN-DCE-024: Coleta de CSAT do Cedente**

1. Ao encerrar uma conversa com a Dani-Cedente, o sistema exibe ao Cedente uma avaliação de satisfação (escala de 1 a 5).
2. A resposta do Cedente é registrada e alimenta o Dashboard de métricas do Admin (RN-DA-034), junto com o CSAT do Cessionário.
3. O CSAT médio do Cedente é monitorado como indicador de qualidade separado do Cessionário no painel Admin.
4. **Se o CSAT médio do Cedente cair abaixo de 3,5 / 5 nas últimas 24 horas:** alerta automático ao Admin (conforme RN-DA-031).

---

## 🔴 13. Edge Cases Consolidados

| **Cenário** | **Comportamento esperado** | **RN de referência** |
|---|---|---|
| Cedente tenta ver a identidade do Cessionário | Dani recusa com mensagem de privacidade e orienta a usar o chat de negociação | RN-DC-002 |
| Cedente recebe proposta abaixo do valor esperado | Dani analisa o retorno líquido e apresenta a opção de contraproposta | RN-DCE-014 |
| Cedente solicita alterar cenário com oportunidade publicada | Dani informa que a alteração requer contato com o suporte | RN-DCE-011 |
| Prazo de Escrow vence sem depósito do Cessionário | Dani notifica o Cedente e informa que a proposta foi cancelada, oportunidade volta ao marketplace | RN-DCE-018 |
| Cedente pede à Dani para aceitar proposta por ele | Dani recusa: a confirmação precisa ser feita pelo Cedente diretamente na plataforma | RN-DCE-017 |
| Documento do dossiê rejeitado pelo Admin | Dani informa o motivo e orienta sobre como corrigir | RN-DCE-013 |
| Cedente pergunta quanto o Cessionário pagou de comissão | Dani pode informar o valor da comissão cobrada (dado público da oportunidade) mas não os dados pessoais do Cessionário | RN-DC-001 |
| Cedente quer retirar oportunidade que está Em negociação | Dani informa que precisa primeiro recusar a proposta ativa antes de retirar | RN-DCE-012 |

---

## 📊 14. Matriz de Permissões

| **Operação** | **Cedente** | **Admin** | **Cessionário** |
|---|---|---|---|
| Cadastrar oportunidade | ✅ Própria | ✅ Qualquer | ❌ Não se aplica |
| Escolher cenário (A/B/C/D) | ✅ Próprio cenário | ✅ Todos | ❌ Bloqueado |
| Ver cenário escolhido | ✅ Próprio | ✅ Todos | ❌ Bloqueado |
| Publicar/retirar oportunidade | ✅ Própria (com confirmação) | ✅ Qualquer | ❌ Não se aplica |
| Ver propostas recebidas | ✅ Próprias (sem identidade do Cessionário) | ✅ Todas | ❌ Não se aplica |
| Aceitar/recusar proposta | ✅ Própria (ação direta na plataforma) | ✅ Qualquer | ❌ Não se aplica |
| Enviar contraproposta | ✅ Própria negociação | ✅ Qualquer | ❌ Não se aplica |
| Acompanhar status do Escrow | ✅ Próprias negociações | ✅ Todas | ✅ Próprias negociações |
| Acompanhar dossiê | ✅ Próprio | ✅ Qualquer | ❌ Não se aplica |
| Assinar contrato via ZapSign | ✅ Próprios contratos | ✅ Supervisão | ✅ Próprios contratos |
| Ver dados pessoais do Cessionário | ❌ Bloqueado | ✅ Permitido | ❌ Próprios dados apenas |
| Receber notificações proativas | ✅ Próprias oportunidades | ✅ Via painel | ❌ Não se aplica |
