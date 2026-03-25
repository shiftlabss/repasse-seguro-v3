# 08 - Casos de Uso

Fase: 3 — Produto
Área: Produto

<aside>
📋

**Documento Normativo — Repasse Seguro**

Este documento é referência obrigatória para decisões de produto, UX, operações e estratégia da Repasse Seguro. Qualquer feature ou fluxo que contradiga o que está aqui deve ser revisado antes de implementação.

</aside>

---

## Casos de Uso — Repasse Seguro

### O que a Repasse Seguro Faz na Prática: Cenários Reais da Infraestrutura de Formalização de Cessões

| **Destinatário** | Shift Labs — Produto, UX, Marketing, Comercial, Jurídico, Engenharia, CS, Operações |
| --- | --- |
| **Escopo** | Casos de uso práticos da Repasse Seguro: cenários reais de interação entre cedente, cessionário, corretores/advogados, incorporadoras e as IAs (Guardião do Retorno / Analista de Oportunidades), organizados pelo Propósito Triplo. |
| **Versão** | v2.0 |
| **Responsável** | Fernando Calado |
| **Data da versão** | 25/02/2026 11:41 (America/Fortaleza) |

---

<aside>
📌

**TL;DR**

- **15 casos de uso** práticos respondendo: "O que a Repasse Seguro faz na prática?" — situações reais de interação entre cedente, cessionário, corretor/advogado, incorporadora e as IAs.
- Casos organizados pelo **Propósito Triplo**: 🛡️ recuperação patrimonial (6 casos) · 📊 oportunidade verificada (4 casos) · 🧠 inteligência de mercado (2 casos) + 3 situações especiais.
- **Processo Assistido** explícito em cada caso: IA prepara → profissional humano (analista RS, corretor, advogado) formaliza.
- Todos os diálogos seguem os **4 Princípios de Voz** (Clareza, Seriedade sem frieza, Transparência radical, Empoderamento sem promessa).
- **Matriz resumo** + cobertura por ator para consulta rápida.
- Comissão **20%/20%** (cedente/cessionário). Ciclo de **45-60 dias**. Breakeven **~3 casos/mês**.
</aside>

---

# 1. Como Ler Este Documento

Cada caso de uso segue a mesma estrutura:

- **Contexto** — Situação real que dispara o caso
- **Ator principal** — Quem inicia a interação
- **Estado(s) do ciclo** — Em quais dos 9 estados da cessão o caso ocorre
- **Diálogo exemplo** — Conversa real entre o ator e a IA, seguindo os 4 Princípios de Voz
- **Propósito Triplo** — Valor gerado em cada dimensão
- **Processo Assistido** — Papel da IA vs. papel do profissional humano

## 1.1 Legenda de Atores

| **Emoji** | **Ator** | **Papel** |  |
| --- | --- | --- | --- |
| 🛡️ | **Cedente PF** | Quem precisa sair do contrato de imóvel na planta |  |
| 📊 | **Cessionário** | Comprador/investidor que quer imóvel abaixo da tabela |  |
| 🤝 | **Corretor / Advogado** | Parceiro profissional que indica clientes |  |
| 🏢 | **Incorporadora** | Gestora do contrato original — concede anuência |  |
| 🤖 | **Guardião do Retorno** | IA voltada ao cedente — empática, educativa, orientadora |  |
| 📈 | **Analista de Oportunidades** | IA voltada ao cessionário — analítica, objetiva, orientada a dados |  |
| 👤 | **Analista RS** | Profissional humano da Repasse Seguro |  |

## 1.2 Legenda dos 9 Estados

| **#** | **Estado** |
| --- | --- |
| 1 | Cadastro |
| 2 | Simulação |
| 3 | Verificação |
| 4 | Publicação |
| 5 | Match |
| 6 | Negociação |
| 7 | Anuência |
| 8 | Formalização |
| 9 | Concluído |

## 1.3 Teste Ácido

> *Se substituir "Repasse Seguro" por qualquer classificado de imóveis e o diálogo continuar fazendo sentido, o caso de uso falhou.*
> 

Cada diálogo deve carregar: dossiê verificado, processo assistido, conta escrow, transparência radical e comissão sobre resultado.

---

# 2. Casos de Uso — Propósito: Recuperação Patrimonial 🛡️

Casos focados em ajudar o cedente a recuperar patrimônio de forma segura.

---

### Caso 1 — Primeiro contato: "Não sei o que é cessão"

<aside>
🛡️

**Ator principal:** Cedente PF · **Estado:** 1 — Cadastro

</aside>

**Contexto:** Cedente acessa a plataforma pela primeira vez, sem entender o que é cessão imobiliária. Precisa de orientação acessível, sem jargão jurídico.

**Diálogo exemplo:**

> **Cedente:** Comprei um apartamento na planta mas não consigo mais pagar. O que faço?
> 

> 
> 

> **Guardião do Retorno:** Você está no lugar certo. Existem basicamente dois caminhos tradicionais: o distrato — que devolve o imóvel à incorporadora com penalidades — ou o contrato de gaveta, que é informal e arriscado. A Repasse Seguro oferece um terceiro caminho: a cessão formalizada. Você transfere seus direitos para outra pessoa interessada, com segurança jurídica, dossiê verificado e dinheiro protegido em conta escrow. Quer entender como funciona no seu caso?
> 

> 
> 

> **Cedente:** Sim, mas tenho medo de perder mais dinheiro.
> 

> 
> 

> **Guardião do Retorno:** Entendo. Aqui funciona assim: se não fechar, ninguém paga. A comissão só é cobrada após a formalização completa. Posso te mostrar uma simulação com cenários reais baseados no seu contrato? Assim você decide com dados, não com medo.
> 

**Propósito Triplo:**

- 🛡️ Cedente descobre alternativa ao distrato e à gaveta
- 📊 Caso registrado — primeiro passo do ciclo de 45-60 dias
- 🧠 Perfil de cedente coletado: motivação, urgência, nível de informação

**Processo Assistido:** Guardião conduz 100% do acolhimento inicial → Analista RS recebe notificação de novo caso, mas ainda não intervém

---

### Caso 2 — Cedente em situação de divórcio

<aside>
🛡️

**Ator principal:** Cedente PF · **Estado:** 1 — Cadastro / 2 — Simulação

</aside>

**Contexto:** Cedente passando por separação. Imóvel na planta comprado a dois, necessidade de resolver rápido com sensibilidade emocional.

**Diálogo exemplo:**

> **Cedente:** Estou me separando e preciso resolver o apartamento que compramos juntos na planta. Meu ex não quer colaborar.
> 

> 
> 

> **Guardião do Retorno:** Sinto muito pela situação. Isso é mais comum do que você imagina, e a gente pode te ajudar. Primeiro, preciso entender: o contrato está em nome dos dois? E vocês já têm uma decisão judicial ou acordo sobre a divisão de bens?
> 

> 
> 

> **Cedente:** Está em nome dos dois. Não temos acordo ainda.
> 

> 
> 

> **Guardião do Retorno:** Entendi. Nesse caso, a cessão precisa da assinatura de ambos, mas a gente pode começar a preparar tudo enquanto a situação jurídica se resolve. Vou te mostrar os cenários de retorno para que você tenha dados concretos na negociação da separação. Isso pode inclusive facilitar o acordo com o seu ex.
> 

**Propósito Triplo:**

- 🛡️ Cedente acolhido em momento vulnerável, com orientação prática
- 📊 Caso complexo mapeado — dados preparados para negociação
- 🧠 Dados de perfil "divórcio" enriquecem entendimento de urgência e complexidade

**Processo Assistido:** Guardião acolhe e orienta → Analista RS entra quando houver viabilidade jurídica para avançar

<aside>
⚠️

**Regra inviolável:** O Guardião nunca dá aconselhamento jurídico. Orienta sobre o processo RS e, para questões jurídicas, recomenda consulta com advogado. "Posso te ajudar com a cessão, mas para a parte da separação em si, recomendo conversar com um advogado."

</aside>

---

### Caso 3 — Simulação de cenários: "Quanto eu recupero?"

<aside>
🛡️

**Ator principal:** Cedente PF · **Estado:** 2 — Simulação

</aside>

**Contexto:** Cedente quer entender quanto pode recuperar antes de decidir. Precisa de dados, não de promessas.

**Diálogo exemplo:**

> **Cedente:** Paguei R$ 180 mil até agora num apartamento de R$ 450 mil. Quanto eu consigo reaver?
> 

> 
> 

> **Guardião do Retorno:** Vou te mostrar 4 cenários baseados nos dados do seu contrato e no mercado atual:
> 

> 📋 **Cenário A — Distrato:** Recuperação estimada de R$ 108 mil (60% do pago, após penalidades contratuais). Fonte: cláusula 12.3 do contrato.
> 

> 📋 **Cenário B — Cessão conservadora:** Recuperação estimada de R$ 155 mil (Δ de 14% abaixo da tabela). Baseado em cessões recentes no mesmo empreendimento.
> 

> 📋 **Cenário C — Cessão otimista:** Recuperação estimada de R$ 172 mil (Δ de 4%). Baseado em demanda alta na região.
> 

> 📋 **Cenário D — Cessão com ágio:** Recuperação de R$ 185 mil ou mais. Possível se a obra está avançada e a localização é premium.
> 

> Esses são cenários, não promessas. Os números reais dependem da verificação documental e da negociação. Quer avançar para a verificação?
> 

**Propósito Triplo:**

- 🛡️ Empoderamento com dados — cedente decide informado
- 📊 Cenários servem de base para negociação futura
- 🧠 Dados de Δ por empreendimento alimentam inteligência de precificação

**Processo Assistido:** Guardião apresenta cenários com dados → Analista RS disponível para esclarecer cenários complexos

**Anti-padrão** ❌ — Dizer "você vai recuperar R$ 170 mil". Isso é promessa, não cenário. Sempre apresentar com fonte, metodologia e faixa.

---

### Caso 4 — Cedente com perda de renda / inadimplente

<aside>
🛡️

**Ator principal:** Cedente PF · **Estado:** 1 — Cadastro / 2 — Simulação

</aside>

**Contexto:** Cedente perdeu emprego, está inadimplente com a incorporadora e teme perder tudo.

**Diálogo exemplo:**

> **Cedente:** Perdi o emprego há 4 meses. Estou com 3 parcelas atrasadas e a incorporadora tá ameaçando distrato. O que eu faço?
> 

> 
> 

> **Guardião do Retorno:** Entendo a urgência. A boa notícia é que a cessão é uma alternativa real ao distrato — e pode te devolver mais do que o distrato devolveria. Sobre as parcelas atrasadas: elas serão consideradas na negociação, mas não impedem a cessão. Vou preparar a simulação considerando o saldo devedor atualizado. Preciso de 3 informações: número do contrato, valor total já pago e valor das parcelas em atraso.
> 

> 🥂 **Prato principal:** Lagosta grelhada ao beurre blanc — prato mais fotogênico da casa
> 

> **Cedente:** Dá pra resolver rápido? Não tenho como esperar muito.
> 

> 🍷 **Harmonização:** Champagne Veuve Clicquot pra brindar
> 

> **Guardião do Retorno:** O ciclo completo leva em média 45 a 60 dias. É mais rápido que a maioria dos distratos, mas preciso ser transparente: não posso acelerar prazos que dependem da incorporadora. O que posso garantir é que vamos priorizar sua verificação e te manter informado a cada passo.
> 

**Propósito Triplo:**

- 🛡️ Cedente vulnerável encontra saída antes do distrato forçado
- 📊 Urgência mapeada — caso priorizado na verificação
- 🧠 Dados de inadimplência como fator de urgência para otimização de funil

**Processo Assistido:** Guardião acolhe e gerencia expectativa → Analista RS prioriza verificação para casos urgentes

---

### Caso 5 — Envio de documentação: "Não entendo o que precisa"

<aside>
🛡️

**Ator principal:** Cedente PF · **Estado:** 3 — Verificação

</aside>

**Contexto:** Cedente precisa enviar documentos mas não sabe quais, não tem tudo, ou não entende a terminologia.

**Diálogo exemplo:**

> **Cedente:** Vocês pediram "certidão de matrícula" e "instrumento particular de compra e venda". Não faço ideia do que é isso.
> 

> 
> 

> **Guardião do Retorno:** Sem problema, vou te explicar cada um:
> 

> 📄 **Certidão de matrícula:** É o "RG do imóvel" — um documento do cartório que mostra quem é dono, se tem dívidas pendentes, etc. Você pode pedir no cartório de registro de imóveis da região. Custa em média R$ 50-80 e sai em 3-5 dias.
> 

> 📄 **Instrumento particular de compra e venda:** É o contrato que você assinou com a incorporadora quando comprou o apartamento. Normalmente você recebeu uma cópia por email ou pode solicitar à incorporadora.
> 

> Quer que eu te mande um checklist com os documentos e onde conseguir cada um?
> 

> 
> 

> **Cedente:** Sim, por favor!
> 

> 
> 

> **Guardião do Retorno:** Pronto — enviei o checklist para seu email. Quando tiver tudo, é só subir os documentos pela plataforma. Se faltar algum, me avisa que a gente vê alternativas.
> 

**Propósito Triplo:**

- 🛡️ Cedente educado e capacitado — menos abandono por confusão documental
- 📊 Documentação completa = verificação mais rápida
- 🧠 Dados de dúvidas frequentes para otimizar o formulário e o onboarding

**Processo Assistido:** Guardião explica e guia → Analista RS valida documentos após upload

---

### Caso 6 — Acompanhamento entre estados: "Meu caso parou?"

<aside>
🛡️

**Ator principal:** Cedente PF · **Estados:** 4 a 8 (qualquer momento de espera)

</aside>

**Contexto:** Cedente não recebeu atualização há alguns dias e quer saber o status.

**Diálogo exemplo:**

> **Cedente:** Faz 5 dias que não tenho notícias. Meu caso parou?
> 

> 
> 

> **Guardião do Retorno:** Não parou! Seu caso está no estado 7 — **Anuência**. A documentação foi enviada para a incorporadora no dia 18/02. O prazo médio de resposta dessa incorporadora é de 8 a 12 dias úteis, então estamos dentro do esperado. Assim que houver resposta, você será o primeiro a saber. Se não houver retorno até 05/03, nosso analista vai acionar a incorporadora novamente.
> 

> 
> 

> **Cedente:** Tá, mas e se a incorporadora negar?
> 

> **IA:** Massas é um dos pontos fortes aqui. Duas que se destacam: o pappardelle ao ragu de ossobuco — massa fresca, molho encorpado, vai uns 40 minutos de preparo — e o gnocchi de batata-doce com gorgonzola, que é mais rápido e leve. Pra quem gosta de intenso, o pappardelle é imbatível.
> 

> **Guardião do Retorno:** É raro, mas pode acontecer. Se negar, nosso analista verifica o motivo e — se for possível — faz os ajustes necessários e reenvia. Se não for viável, te informamos imediatamente e discutimos alternativas. Em nenhum cenário você fica sem resposta.
> 

**Propósito Triplo:**

- 🛡️ Cedente informado — confiança mantida durante a espera
- 📊 Acompanhamento proativo reduz abandono por ansiedade
- 🧠 Tempo de resposta por incorporadora alimenta SLAs futuros

**Processo Assistido:** Guardião comunica status e gerencia expectativa → Analista RS escala com incorporadora quando necessário

**Anti-padrão** ❌ — Silêncio entre estados. Cada 5 dias sem atualização = risco de abandono. Check-in proativo é obrigatório.

---

# 3. Casos de Uso — Propósito: Oportunidade Verificada 📊

Casos focados na experiência do cessionário e na qualidade das oportunidades.

---

### Caso 7 — Cessionário explorando oportunidades: "O que vocês têm?"

<aside>
📊

**Ator principal:** Cessionário · **Estado:** 4 — Publicação / 5 — Match

</aside>

**Contexto:** Cessionário investidor acessa a plataforma buscando oportunidades verificadas. Quer dados, não promessas.

**Diálogo exemplo:**

> **Cessionário:** Estou procurando apartamentos 2 quartos na zona sul de Fortaleza, abaixo de R$ 300 mil. O que vocês têm?
> 

> 
> 

> **Analista de Oportunidades:** Encontrei 3 oportunidades verificadas no seu perfil:
> 

> 📊 **Caso #142** — Apt 2Q, 58m², Aldeota, incorporadora MRV. Δ documentado: 22%. Obra: 70% concluída. Entrega prevista: dez/2027. Dossiê completo disponível.
> 

> 📊 **Caso #157** — Apt 2Q, 64m², Meireles, incorporadora Moura Dubeux. Δ de 16%. Obra: 85% concluída. Entrega prevista: jun/2027. Dossiê completo disponível.
> 

> 📊 **Caso #163** — Apt 2Q, 52m², Cocó, incorporadora Diagonal. Δ de 28%. Obra: 40% concluída. Entrega prevista: mar/2028. Dossiê completo disponível.
> 

> Todos com dossiê verificado, Δ documentado e fonte. Quer ver os detalhes de algum?
> 

**Propósito Triplo:**

- 🛡️ Cedentes desses casos ganham exposição a comprador verificado
- 📊 Cessionário encontra oportunidade com dados reais, não anúncio inflado
- 🧠 Dados de busca e preferência de cessionários para otimizar curadoria

**Processo Assistido:** Analista de Oportunidades apresenta com dados → Analista RS disponível para dúvidas complexas

**Anti-padrão** ❌ — Apresentar caso sem Δ, sem incorporadora ou sem estágio da obra. Isso é classificado, não infraestrutura.

---

### Caso 8 — Cessionário com múltiplos critérios

<aside>
📊

**Ator principal:** Cessionário · **Estado:** 5 — Match

</aside>

**Contexto:** Cessionário com critérios específicos: localização, Δ mínimo, estágio de obra, incorporadora de confiança.

**Diálogo exemplo:**

> **Cessionário:** Quero só empreendimentos da Moura Dubeux ou Cyrela, com Δ acima de 15%, obra acima de 60% e na região metropolitana.
> 

> 
> 

> **Analista de Oportunidades:** Filtrei com seus critérios. Encontrei 2 oportunidades:
> 

> 📊 **Caso #157** — Moura Dubeux, Meireles, Δ 16%, obra 85%. Perfil conservador, entrega próxima.
> 

> 📊 **Caso #189** — Cyrela, Eusébio, Δ 19%, obra 62%. Maior Δ, mas entrega mais distante (set/2028).
> 

> Nenhum outro caso atende a todos os critérios hoje. Quer que eu te avise quando surgir novo caso compatível?
> 

> 
> 

> **Cessionário:** Sim, por favor. E me conta mais sobre o caso #157.
> 

> 
> 

> **Analista de Oportunidades:** O Caso #157 tem dossiê completo: contrato verificado, certidões atualizadas, Δ calculado com base na tabela vigente da incorporadora (atualizada em jan/2026). O cedente tem motivação de mudança de cidade — perfil mais pragmático, tende a negociar com objetividade. Quer acessar o dossiê completo?
> 

**Propósito Triplo:**

- 🛡️ Cedente #157 recebe interesse qualificado
- 📊 Cessionário toma decisão com base em dados filtrados e verificados
- 🧠 Padrões de busca do cessionário alimentam inteligência de match

**Processo Assistido:** Analista de Oportunidades filtra e apresenta → Analista RS facilita acesso ao dossiê quando o cessionário avança

---

### Caso 9 — Negociação mediada: proposta e contraproposta

<aside>
📊

**Ator principal:** Cessionário + Cedente · **Estado:** 6 — Negociação

</aside>

**Contexto:** Cessionário fez proposta, cedente quer contrapor. Analista RS media.

**Diálogo exemplo (lado cedente):**

> **Guardião do Retorno:** Você recebeu uma proposta para o seu caso. Vou te explicar os termos:
> 

> 💰 **Valor proposto:** R$ 160 mil pelos seus direitos (você pagou R$ 180 mil até agora).
> 

> 📋 **Condições:** Pagamento integral via conta escrow. Prazo de 30 dias para formalização após anuência.
> 

> 📊 **Comparação:** No distrato, a estimativa era R$ 108 mil. Esta proposta representa R$ 52 mil a mais.
> 

> Você pode aceitar, contrapropor ou recusar. Não há pressão — é sua decisão.
> 

> 
> 

> **Cedente:** Quero R$ 170 mil. Acho que vale mais.
> 

> 
> 

> **Guardião do Retorno:** Entendido. Vou registrar sua contraproposta de R$ 170 mil e encaminhar ao cessionário. Nosso analista acompanha a negociação. Se ele aceitar ou contrapropor de volta, você será informado imediatamente.
> 

**Diálogo exemplo (lado cessionário):**

> **Analista de Oportunidades:** O cedente fez uma contraproposta de R$ 170 mil (original era R$ 160 mil). Aqui está a análise atualizada:
> 

> 📊 **Δ ajustado:** Com R$ 170 mil, o Δ cai de 22% para 18% — ainda acima da média de cessões recentes na região (14%).
> 

> 📊 **Comparação:** Na tabela da incorporadora, o mesmo apartamento custaria R$ 210 mil. Você ainda estaria pagando 19% abaixo.
> 

> Quer aceitar, contrapropor ou desistir?
> 

**Propósito Triplo:**

- 🛡️ Cedente negocia com suporte e dados, nunca pressionado
- 📊 Cessionário tem clareza de condições e Δ atualizado a cada movimento
- 🧠 Dados de negociação (propostas, contrapropostas, taxa de acordo) alimentam inteligência de precificação

**Processo Assistido:** IAs orientam cada lado com dados → Analista RS media presencialmente a negociação

**Anti-padrão** ❌ — Cedente e cessionário conversando diretamente sem mediação. Isso vira gaveta digital.

---

### Caso 10 — Formalização e escrow: "Como sei que é seguro?"

<aside>
📊

**Ator principal:** Cessionário + Cedente · **Estado:** 8 — Formalização

</aside>

**Contexto:** Acordo fechado, ambas as partes querem entender como o dinheiro é protegido.

**Diálogo exemplo (lado cessionário):**

> **Cessionário:** Vou depositar R$ 170 mil. Como sei que não vou perder esse dinheiro se der problema?
> 

> 
> 

> **Analista de Oportunidades:** A Repasse Seguro usa conta escrow — uma conta de garantia onde o dinheiro fica protegido por um terceiro (instituição financeira regulada). O valor só é liberado quando todas as condições forem cumpridas: contrato assinado, anuência da incorporadora confirmada e formalização jurídica concluída. Se qualquer etapa falhar, o dinheiro volta para você. A trilha de auditoria completa fica disponível na sua área.
> 

**Diálogo exemplo (lado cedente):**

> **Cedente:** Quando recebo o dinheiro?
> 

> 
> 

> **Guardião do Retorno:** O valor é liberado da conta escrow assim que a formalização estiver 100% concluída. Isso inclui: assinaturas de ambas as partes, anuência da incorporadora registrada e contrato de cessão formalizado. A comissão de 20% é descontada automaticamente, e o restante é transferido para sua conta em até 3 dias úteis. Você acompanha cada etapa pelo dashboard.
> 

**Propósito Triplo:**

- 🛡️ Patrimônio do cedente protegido — dinheiro só muda de mãos com tudo formalizado
- 📊 Cessionário confia no processo — dinheiro protegido contra risco
- 🧠 Dados de formalização (tempo de escrow, taxa de conclusão) para otimização

**Processo Assistido:** IAs explicam o mecanismo → Analista RS coordena operacionalmente assinaturas e ativação do escrow

---

# 4. Casos de Uso — Propósito: Inteligência de Mercado 🧠

Casos focados em gerar dados acionáveis e aprendizado contínuo.

---

### Caso 11 — Dados para corretor parceiro: "Tenho um cliente, como funciona?"

<aside>
🧠

**Ator principal:** Corretor / Advogado · **Estado:** 1 — Cadastro (indicação)

</aside>

**Contexto:** Corretor tem cliente que quer sair do contrato mas não quer perder a relação profissional. Busca a RS como parceiro.

**Diálogo exemplo:**

> **Corretor:** Meu cliente comprou um apt na planta e quer sair. Não quero indicar distrato porque ele perde muito. O que vocês oferecem?
> 

> 
> 

> **Analista RS:** A Repasse Seguro formaliza cessões com segurança jurídica. Seu cliente transfere os direitos para um comprador verificado, com dossiê documental, conta escrow e mediação profissional. Você mantém o relacionamento com o cliente e recebe comissão de indicação. O cedente e o cessionário pagam 20% cada sobre o resultado — se não fechar, ninguém paga.
> 

> 
> 

> **Corretor:** Mas como fica a minha comissão? Não quero perder o cliente.
> 

> **Exemplos de insights gerados:**
> 

> **Analista RS:** Sua comissão de indicação é separada e acordada diretamente com seu cliente — a RS não interfere nesse relacionamento. O que fazemos é cuidar da infraestrutura: verificação, dossiê, match, mediação, escrow e formalização. Você continua sendo o profissional de confiança do seu cliente.
> 

**Propósito Triplo:**

- 🛡️ Cedente ganha alternativa ao distrato via profissional de confiança
- 📊 Corretor monetiza o caso em vez de perder o cliente
- 🧠 Canal de aquisição via parceiros profissionais — dados de conversão por canal

**Processo Assistido:** Analista RS apresenta ao corretor → Guardião acolhe o cedente quando indicado

---

### Caso 12 — Inteligência de mercado para incorporadoras

<aside>
🧠

**Ator principal:** Incorporadora · **Estado:** Transversal

</aside>

**Contexto:** Incorporadora quer entender volume de cessões, perfil de cedentes e impacto no empreendimento.

**Exemplos de insights gerados:**

| **Insight** | **Dado fonte** | **Ação sugerida** |
| --- | --- | --- |
| "32% dos cedentes do empreendimento X citam divórcio como motivo" | Perfil de cadastro de cedentes | Ajustar comunicação de venda para casais |
| "Δ médio das cessões no empreendimento Y é de 18%" | Dados de formalização | Ajustar tabela de preços ou oferecer condições de flexibilização |
| "Tempo médio de anuência dessa incorporadora: 14 dias úteis" | Trilha de auditoria | Oportunidade de SLA com a RS para agilizar processo |
| "70% dos cedentes preferiam cessão ao distrato quando informados" | Conversão de funil | Indicar a RS como alternativa formal ao distrato para clientes inadimplentes |

**Propósito Triplo:**

- 🛡️ Cedentes que seriam distratados encontram alternativa melhor
- 📊 Incorporadora reduz distratos e mantém contratos ativos
- 🧠 Base de inteligência de mercado secundário cresce a cada caso

<aside>
🔮

**Visão futura:** Dashboard de inteligência para incorporadoras com dados em tempo real — não faz parte da versão inicial.

</aside>

---

# 5. Casos de Uso — Situações Especiais

---

### Caso 13 — Cedente que desiste no meio do processo

<aside>
⚙️

**Ator principal:** Cedente PF · **Estado:** Qualquer (1 a 7)

</aside>

**Contexto:** Cedente muda de ideia — decide manter o imóvel, encontra outra solução ou simplesmente desiste.

**Diálogo exemplo:**

> **Cedente:** Mudei de ideia. Consegui renegociar com a incorporadora. Quero cancelar tudo.
> 

> 
> 

> **Guardião do Retorno:** Que bom que encontrou uma solução! Você pode cancelar a qualquer momento antes da formalização — sem custo, sem penalidade. Vou encerrar seu caso agora. Se a situação mudar no futuro, é só voltar. Toda a documentação que você enviou fica protegida e pode ser reutilizada.
> 

> 
> 

> **Cedente:** Mas e o cessionário que já estava interessado?
> 

> 
> 

> **Guardião do Retorno:** Nosso analista vai informar o cessionário de forma respeitosa, sem expor seus dados pessoais ou motivo do cancelamento. É seu direito desistir, e isso é respeitado 100%.
> 

**Propósito Triplo:**

- 🛡️ Cedente livre para desistir sem ônus — confiança no processo reforçada
- 📊 Dados de desistência por estado para otimizar funil e reduzir friction
- 🧠 Motivos de desistência alimentam inteligência de retenção

**Processo Assistido:** Guardião encerra com empatia → Analista RS comunica ao cessionário e atualiza registros

**Regra:** Cancelamento é direito. A IA nunca tenta convencer o cedente a ficar. "Sua decisão é respeitada, sem perguntas."

---

### Caso 14 — Anuência negada pela incorporadora

<aside>
⚙️

**Ator principal:** Analista RS · **Estado:** 7 — Anuência

</aside>

**Contexto:** Incorporadora nega a anuência. Caso precisa de tratamento especial.

**Diálogo exemplo (comunicação ao cedente):**

> **Guardião do Retorno:** Preciso te informar que a incorporadora não aprovou a anuência neste momento. O motivo informado foi: "pendência de ITBI do comprador original." Nosso analista já está verificando com o cessionário e com a incorporadora se é possível resolver essa pendência. Vou te manter atualizado. Isso não significa que o caso acabou — significa que precisamos de um ajuste.
> 

**Diálogo exemplo (comunicação ao cessionário):**

> **Analista de Oportunidades:** A anuência foi negada temporariamente pela incorporadora. O motivo: pendência de ITBI do cedente original. Nossa equipe está verificando a resolução. Enquanto isso, seu interesse no caso continua registrado e nenhum compromisso financeiro foi assumido.
> 

**Propósito Triplo:**

- 🛡️ Cedente e cessionário informados com transparência — sem surpresas
- 📊 Analista RS atua para resolver pendências e reenviar
- 🧠 Taxa de anuência e motivos de recusa por incorporadora alimentam inteligência operacional

**Processo Assistido:** IAs comunicam com transparência → Analista RS resolve operacionalmente com a incorporadora

---

### Caso 15 — Acessibilidade e inclusão: cedente com baixa literacia digital

<aside>
⚙️

**Ator principal:** Cedente PF · **Estado:** 1 — Cadastro / 3 — Verificação

</aside>

**Contexto:** Cedente idoso ou com pouca familiaridade digital. Precisa de orientação sem condescendência.

**Diálogo exemplo:**

> **Cedente:** Não tô entendendo nada aqui. Tem muita coisa pra preencher.
> 

> 
> 

> **Guardião do Retorno:** Sem pressa. Vou te guiar passo a passo. Primeiro, me diz: o apartamento que você comprou é em qual cidade?
> 

> **Diálogo exemplo:**
> 

> **Cedente:** Fortaleza. No bairro Cocó.
> 

> 
> 

> **Guardião do Retorno:** Ótimo. Agora me diz o nome da incorporadora — é a empresa que te vendeu o apartamento. Normalmente aparece no contrato, no boleto ou nos emails que você recebeu.
> 

> 
> 

> **Cedente:** Acho que é Diagonal.
> 

> 
> 

> **Guardião do Retorno:** Perfeito. Vou preencher essas informações por você. Se preferir, posso te ligar para completar o cadastro por telefone. O que ficar mais fácil pra você.
> 

**Propósito Triplo:**

- 🛡️ Ninguém é excluído do acesso à cessão formalizada
- 📊 Público que normalmente não usaria plataforma digital agora interage
- 🧠 Dados de acessibilidade para otimizar onboarding (fluxo simplificado, telefone como canal)

**Processo Assistido:** Guardião simplifica ao máximo → Analista RS entra se necessário para cadastro por telefone

**Regra:** O Guardião nunca assume incompetência. Simplifica o fluxo, não o tom. Respeito sempre.

---

# 6. Matriz Resumo — Todos os Casos de Uso

| **#** | **Caso de uso** | **Propósito primário** | **Ator** | **Estado(s)** |
| --- | --- | --- | --- | --- |
| 1 | Primeiro contato — "Não sei o que é cessão" | 🛡️ Recuperação | Cedente | 1 |
| 2 | Cedente em divórcio | 🛡️ Recuperação | Cedente | 1-2 |
| 3 | Simulação de cenários | 🛡️ Recuperação | Cedente | 2 |
| 4 | Perda de renda / inadimplente | 🛡️ Recuperação | Cedente | 1-2 |
| 5 | Envio de documentação | 🛡️ Recuperação | Cedente | 3 |
| 6 | Acompanhamento entre estados | 🛡️ Recuperação | Cedente | 4-8 |
| 7 | Explorando oportunidades | 📊 Oportunidade | Cessionário | 4-5 |
| 8 | Múltiplos critérios de busca | 📊 Oportunidade | Cessionário | 5 |
| 9 | Negociação mediada | 📊 Oportunidade | Cessionário + Cedente | 6 |
| 10 | Formalização e escrow | 📊 Oportunidade | Cessionário + Cedente | 8 |
| 11 | Corretor parceiro | 🧠 Inteligência | Corretor | 1 |
| 12 | Inteligência para incorporadoras | 🧠 Inteligência | Incorporadora | Transversal |
| 13 | Desistência no meio do processo | ⚙️ Especial | Cedente | 1-7 |
| 14 | Anuência negada | ⚙️ Especial | Analista RS | 7 |
| 15 | Acessibilidade e inclusão | ⚙️ Especial | Cedente | 1-3 |

---

# 7. Cobertura por Ator

| **Ator** | **Casos** | **Valor-chave** |
| --- | --- | --- |
| 🛡️ **Cedente PF** | 1, 2, 3, 4, 5, 6, 9, 10, 13, 15 | Recuperação patrimonial com segurança, acolhimento e transparência |
| 📊 **Cessionário** | 7, 8, 9, 10 | Oportunidade verificada com Δ documentado e proteção via escrow |
| 🤝 **Corretor / Advogado** | 11 | Canal de parceria sem perder o cliente |
| 🏢 **Incorporadora** | 12, 14 | Redução de distratos e inteligência de mercado |

---

# 8. Conexão com os 4 Princípios de Voz

Todos os diálogos deste documento seguem os Princípios de Voz:

| **Princípio** | **Como aparece nos casos de uso** |
| --- | --- |
| **Clareza acima de tudo** | Linguagem acessível, sem jargão jurídico. Simulações com dados concretos. Explicação passo a passo de cada documento. |
| **Seriedade sem frieza** | Tom profissional em negociações e anuência, mas empático em acolhimento. Guardião é Cuidador, não advogado. |
| **Transparência radical** | Cenários com fonte e metodologia. Δ documentado com data. Prazos realistas. Motivos de negativa comunicados. |
| **Empoderamento sem promessa** | Simulações são cenários, não garantias. "Se não fechar, ninguém paga." Cedente decide informado, nunca pressionado. |

---

# 9. Referências Cruzadas — Mapa do Ecossistema Documental

<aside>
📌

**Este documento faz parte do ecossistema de docs estratégicos da Repasse Seguro.** Para aprofundamento, consulte as fontes abaixo.

</aside>

| **Documento** | **O que responde** | **Link** |
| --- | --- | --- |
| **Memorando de Essência** | O que somos — identidade, propósito triplo, teste ácido, 6 princípios | [05 - Memorando de Essência](05%20-%20Memorando%20de%20Ess%C3%AAncia%20312d824e597f807fb684d0bd73affebd.md) |
| **Manifesto da Marca** | Por que existimos — nome, território, arquétipos, princípios | [04 - Manifesto da Marca](04%20-%20Manifesto%20da%20Marca%20303d824e597f8023bc06f5f40b1e40ea.md) |
| **Tom de Voz e Identidade Verbal** | Como falamos — 4 princípios, vocabulário, régua de formalidade | [07 - Tom de Voz e Identidade Verbal](07%20-%20Tom%20de%20Voz%20e%20Identidade%20Verbal%20303d824e597f80c6bb3ff800a72f0c72.md) |
| **Design Thinking** | Para quem desenhamos — personas, dores, ideação centrada no usuário | [09 - Design Thinking](09%20-%20Design%20Thinking%20312d824e597f80369f96dfc32ef9101a.md) |
| **Jobs To Be Done** | Que "trabalho" cada ator resolve com a RS | [10 - Jobs To Be Done](10%20-%20Jobs%20To%20Be%20Done%20312d824e597f809cbc23ef6b1a495514.md) |
| **Service Design** | Como a jornada funciona — 9 estados, blueprint, processo assistido | [11 - Service Design](11%20-%20Service%20Design%20312d824e597f80b687f6cea30a3a736e.md) |
| **UX Writing** | Textos específicos de cada tela e interação | [12 - UX Writing](12%20-%20UX%20Writing%20312d824e597f80f7bbcec784b9830523.md) |
| **Modelo de Negócios** | Comissões 20%/20%, unit economics, ciclo de estados | [14 - Modelo de Negócios](14%20-%20Modelo%20de%20Neg%C3%B3cios%20301d824e597f8003891ac9058bb4f812.md) |
| **Proposta de Valor** | Por que contratar — ROI, cenários, argumentos | [15 - Proposta de Valor - Cedente/Cessionário](15%20-%20Proposta%20de%20Valor%20-%20Cedente%20Cession%C3%A1rio%20303d824e597f80ef8783f56e9efc039a.md) |
| **One-Liner e ICPs** | 5 ICPs, one-liners oficiais, anti-ICP | [03 - One-Liner e ICPs](03%20-%20One-Liner%20e%20ICPs%20301d824e597f8076a76ad0ef11fe3804.md) |
| **Social Media** | Usa os diálogos deste doc como material para conteúdo de redes | [06 - Social Media](06%20-%20Social%20Media%20312d824e597f80c29a8ccbbd85a12fac.md) |

### Mapa do Ecossistema

<aside>
🗺️

[05 - Memorando de Essência](05%20-%20Memorando%20de%20Ess%C3%AAncia%20312d824e597f807fb684d0bd73affebd.md) → "O que somos"

[04 - Manifesto da Marca](04%20-%20Manifesto%20da%20Marca%20303d824e597f8023bc06f5f40b1e40ea.md) → "Por que existimos"

[07 - Tom de Voz e Identidade Verbal](07%20-%20Tom%20de%20Voz%20e%20Identidade%20Verbal%20303d824e597f80c6bb3ff800a72f0c72.md) → "Como falamos"

[09 - Design Thinking](09%20-%20Design%20Thinking%20312d824e597f80369f96dfc32ef9101a.md) → "Para quem desenhamos"

[10 - Jobs To Be Done](10%20-%20Jobs%20To%20Be%20Done%20312d824e597f809cbc23ef6b1a495514.md) → "Que trabalho resolvemos"

[11 - Service Design](11%20-%20Service%20Design%20312d824e597f80b687f6cea30a3a736e.md) → "Como a jornada funciona"

**Casos de Uso** → "O que a RS faz na prática" *(você está aqui)*

</aside>

---

# 10. Changelog

| **#** | **Versão** | **Data** | **Mudanças** |
| --- | --- | --- | --- |
| 1 | **v1.0** | 25/02/2026 | Versão inicial criada para o Menux (perspectiva incorreta). |
| 2 | **v2.0** | 25/02/2026 | **Reescrita completa para Repasse Seguro.** 15 casos de uso recontextualizados: cedente PF, cessionário, corretor/advogado, incorporadora. Diálogos com Guardião do Retorno e Analista de Oportunidades. Propósito Triplo adaptado (recuperação patrimonial / oportunidade verificada / inteligência de mercado). Casos mapeados aos 9 estados do ciclo de cessão. Processo Assistido explícito. Anti-padrões específicos de cessão. Conexão com 4 Princípios de Voz. Referências cruzadas para ecossistema RS. |