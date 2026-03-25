# 14 - Modelo de Negócios

Fase: 4 — Negócio
Área: Estratégia

# 1. Modelo de Negócios — Repasse Seguro

## 1.1 Metadados do Documento

| Campo | Valor |
| --- | --- |
| Destinatário | Stakeholders de produto, comercial, investidores e liderança do Repasse Seguro |
| Escopo | Modelo de monetização completo e arquitetura de plataforma: racional estratégico, mecânica de receita (2 comissões), perfis de acesso (3 dashboards), unit economics, análise de sensibilidade, fluxo de caixa, breakeven, modelos alternativos rejeitados, matriz de riscos, governança de cálculo, ciclo de vida do caso (estados e transições), SLA operacional por etapa, política de negociação de comissão e FAQ operacional |
| Versão | v3.5 |
| Responsável | Fernando Calado |
| Data da versão | 21/02/2026 17:53 (America/Fortaleza) — v3.5 |

## 1.2 Glossário de Definições-chave

<aside>
📖

**Termos utilizados neste documento**

- **Cedente:** pessoa física ou jurídica que detém o contrato imobiliário e deseja repassá-lo (quem vende/transfere).
- **Comprador (cessionário):** pessoa física ou jurídica que adquire o contrato repassado (quem compra).
- **Valor Recuperado:** montante total que o cedente recebe no fechamento do repasse.
- **Tabela Atual:** preço vigente do imóvel conforme a fonte de referência, na **data do lance do comprador** (ver hierarquia de fontes na seção 8.5).
- **Tabela do Contrato:** valor do imóvel conforme o contrato original firmado entre o cedente e a construtora. Usado como referência para calcular o Δ do comprador.
- **Δ (delta):** diferença econômica entre a Tabela Atual e a Tabela do Contrato. Fórmula: Δ = max(0, Tabela_Atual − Tabela_Contrato).
- **Valor Distrato Referência:** estimativa do valor que o cedente receberia caso optasse pelo distrato, documentada no dossiê. Usado como base de cálculo da comissão do cedente nos Cenários B, C e D.
- **Fechamento:** evento que dispara a cobrança de comissão. Definição completa na seção 6 (3 critérios cumulativos obrigatórios).
- **Termo Comercial:** documento formalizado entre as partes e o Repasse Seguro que define percentuais, pisos, tetos e condições específicas de cada caso.
- **Dossiê:** conjunto de evidências documentais de um caso (tabela, preço, distrato, instrumento assinado, etc.).
- **Cenário A:** cedente apenas transfere o saldo devedor — RS não cobra comissão.
- **Cenários B, C e D:** cedente define meta de retorno (100%, +30% ou +50% do valor pago). RS cobra **20%** sobre (Valor Recuperado − Valor Distrato Referência).
- **Piso de comissão:** valor mínimo de comissão do comprador, quando definido no Termo Comercial.
- **Teto de comissão:** valor máximo de comissão, quando definido no Termo Comercial.
- **Preço do Repasse:** valor efetivamente pago pelo comprador ao cedente pela cessão/transferência do contrato, confirmado por evidência documental no dossiê.
- **Conta Escrow (conta garantia/caução):** mecanismo bancário ou virtual que atua como intermediário neutro em transações de alto risco ou valor, retendo os recursos até que todas as condições contratuais sejam cumpridas — garantindo segurança e mitigando o risco de inadimplência. No Repasse Seguro, a Conta Escrow retém o valor da transação e libera as comissões da RS e o valor líquido ao cedente após o cumprimento das condições de Fechamento (seção 6).
- **Perda evitada:** diferença entre o que o cedente receberia no distrato e o que recebe via repasse — é o motor econômico da tese.
- **Custo incremental por caso:** custo operacional direto para processar um caso individual (triagem, verificação, formalização, suporte).
- **Margem de contribuição:** receita por caso − custo incremental por caso.
</aside>

<aside>
📌

**TL;DR — Monetização do Repasse Seguro**

- O Repasse Seguro tem **apenas 2 receitas**: (1) comissão do **cedente** e (2) comissão do **comprador**. Não há assinatura, mensalidade ou capital próprio.
- O **cedente escolhe um dos 4 cenários de retorno** no cadastro: **A** (assumir saldo devedor, RS sem comissão), **B** (valor pago), **C** (valor pago +30%) ou **D** (valor pago +50%). Cenários B/C/D: RS cobra **20%** sobre (Valor Recuperado − Valor Distrato Referência).
- O **investidor não paga mensalidade**. Comissão: **20%** sobre (Tabela Atual − Tabela do Contrato). Exceção no Cenário A sem valorização: 20% sobre o valor pago pelo cedente.
- **Unit economics do caso-tipo (Base):** ticket combinado de **R$ 50k** (R$ 30k cedente + R$ 20k comprador), custo incremental estimado de **R$ 5k–9k**, margem de contribuição de **R$ 33,5k–40k** (~67–80%).
- **Breakeven operacional:** a partir de **~3 casos/mês** no cenário Base, com burn mensal de R$ 110k.
- **TAM de comissões:** R$ 2,5B–4,5B/ano (50–70 mil casos × R$ 50–65k ticket médio).
- O modelo foi escolhido por **não exigir capital próprio, não assumir risco de balanço e alinhar incentivos**: só cobra quando há resultado econômico real para ambas as partes.
- A comissão da RS é recebida por meio de uma **Conta Escrow** — mecanismo neutro que retém os recursos da transação até o cumprimento das condições de Fechamento, eliminando risco de inadimplência e garantindo segurança para todas as partes.
- A plataforma opera com **3 ambientes independentes**: Dashboard do **Cedente**, Dashboard do **Cessionário** e **Admin** (acesso total) — dados isolados entre as partes externas.
- O **ciclo de vida de um caso** percorre 9 estados principais (Captado → Concluído), com SLAs definidos por etapa e ciclo total estimado de **45–60 dias** (realista). A **política de desconto** permite negociação do percentual de comissão dentro de pisos por faixa de valor, preservando margem e governança.
</aside>

---

## 2. Contexto Estratégico: Por Que Este Modelo

<aside>
🎯

**Objetivo desta seção**

Fundamentar a escolha do modelo de receita com dados de mercado, lógica econômica e análise de alternativas. O modelo não é arbitrário — é a resposta a uma estrutura de mercado específica.

</aside>

### 2.1 O motor econômico da tese: o delta entre distrato e cessão

A Lei 13.786/2018 criou um **delta econômico mensurável** entre distrato e cessão que é o fundamento de todo o modelo:

| **Cenário** | **Valor retido/perdido pelo cedente** | **Base legal** |
| --- | --- | --- |
| Distrato (patrimônio de afetação) | Até **50%** do valor pago | Lei 13.786/2018 |
| Distrato (sem afetação) | Até **25%** do valor pago | Lei 13.786/2018 |
| Cessão via Repasse Seguro | **20%** (comissão + deságio típico) | Cessão de direitos |

**Exemplo concreto (caso didático) — Imóvel R$ 4.000.000 / Total pago pelo cedente: R$ 500.000**

*Distrato referência (50% retido): R$ 250.000*

| **Cenário** | **Retorno esperado** | **Valor Recuperado** | **Base de cálculo** (Recup. − Distrato) | **Comissão RS** (20% × Base) | **Cedente recebe líquido** |
| --- | --- | --- | --- | --- | --- |
| **A** | Apenas saldo devedor | R$ 0 | — | **R$ 0** | **R$ 0** (−R$ 250k vs distrato) |
| **B** | 100% do pago | R$ 500.000 | R$ 250.000 | **R$ 50.000** | **R$ 450.000** (+R$ 200k vs distrato) |
| **C** | 100% + 30% | R$ 650.000 | R$ 400.000 | **R$ 80.000** | **R$ 570.000** (+R$ 320k vs distrato) |
| **D** | 100% + 50% | R$ 750.000 | R$ 500.000 | **R$ 100.000** | **R$ 650.000** (+R$ 400k vs distrato) |

<aside>
💡

**Leitura analítica**

No **Cenário A**, o cedente não recupera nada — sai com **R$ 250k a menos** que no distrato. É a última opção, usada quando nenhum comprador aceita os cenários superiores. Já no **Cenário B** (o mais conservador com comissão), o cedente sai com **R$ 200k a mais** do que no distrato — depois de pagar a comissão. No **Cenário D**, esse ganho chega a **R$ 400k**. A comissão da RS, nos cenários B/C/D, é sempre inferior ao benefício gerado — o que explica a willingness-to-pay elevada e o alinhamento de incentivos do modelo.

</aside>

### 2.2 Dimensão do mercado endereçável (conexão com receita)

| **Métrica** | **Dado** | **Fonte** |
| --- | --- | --- |
| Unidades vendidas (2024) | **400,5 mil** (+20,9% vs 2023) | CBIC/CII |
| Taxa de distrato (média histórica) | **15–30%** das entregas | Fitch Ratings / Senior Index |
| Universo de casos/ano | **50.000–70.000 contratos** passíveis de repasse | Estimativa interna |
| Ticket médio de comissão total | **R$ 50k–65k** por caso fechado | Modelo de Negócios |
| **TAM de comissões** | **R$ 2,5B–4,5B/ano** | Volume × ticket |
| Participação alvo (Ano 2) | **0,5–1%** do TAM | Cenário conservador |

### 2.3 Por que comissão sobre resultado (e não SaaS, spread ou fee fixo)

| **Modelo avaliado** | **Como funciona** | **Por que foi descartado** | **Quem usa** |
| --- | --- | --- | --- |
| SaaS / assinatura | Cobrança mensal recorrente | Não há uso recorrente do cedente. Pessoa usa uma vez para resolver um problema pontual. Churn seria ~100%. | Portais genéricos |
| Spread / deságio | Comprar o contrato com desconto e revender | Exige **capital próprio**, assume **risco de balanço** e limita escala ao capital disponível. Se o mercado virar, o spread vira prejuízo. | Dinheiro na Planta |
| Fee fixo por anúncio | Cobrar para listar o contrato | Não alinha incentivo (cobra mesmo sem resultado). Gera desconfiança e baixa willingness-to-pay em público que já está em apuros financeiros. | Classificados |
| Fee fixo por lead | Cobrar por conexão qualificada | Funciona para volume alto e ticket baixo. Não cabe em mercado de ticket alto e ciclo longo. Risco de "lead frio" que não fecha. | Marketplaces de serviços |
| **Intermediação financeira (escrow próprio)** | Operar conta escrow proprietária com capital e licença próprios | Exige licença regulatória, capital regulatório e compliance pesado. **Inviável na fase inicial.** O RS adota escrow via parceiro bancário/fintech regulado — sem licença própria. | Fintechs |
| **Cobrar % sobre ganho real, apenas no fechamento** | **Alinha incentivo, zero capital próprio, zero risco de balanço, escalável sem capital por caso.** | **Repasse Seguro** |  |

<aside>
✅

**Por que o modelo escolhido é superior para este mercado**

1. **Zero risco de balanço:** o Repasse Seguro não compra contratos nem assume posição financeira.
2. **Alinhamento de incentivos:** só cobra quando há resultado econômico real para o cedente E para o comprador.
3. **Escalabilidade:** não precisa de capital por caso — a plataforma escala com processo, não com dinheiro.
4. **Willingness-to-pay natural:** o cedente está evitando perda de R$ 100k+; pagar R$ 30k de comissão é decisão fácil. O comprador está ganhando desconto de R$ 100k+; pagar R$ 20k é proporcional ao ganho.
5. **Defensibilidade:** o modelo de spread (Dinheiro na Planta) é limitado pelo capital; o modelo de comissão é limitado pela capacidade operacional — que escala mais barato.
</aside>

### 2.4 Willingness-to-pay: a lógica econômica de cada lado

| **Parte** | **Alternativa sem o RepasseSeguro** | **Com o RepasseSeguro** | **Ganho líquido (após comissão)** |
| --- | --- | --- | --- |
| **Cedente** | Distrato: perde R$ 150k (50% de R$ 300k pagos) | Cessão (Cenário B): recupera R$ 300k, paga comissão RS de R$ 30k (20% × R$ 150k base) | **R$ 270k** vs R$ 150k = **+R$ 120k** líquido a mais |
| **Comprador** | Comprar na tabela: R$ 600k (Tabela Atual) | Comprar via repasse: R$ 480k + comissão RS de R$ 20k (20% × Δ de R$ 100k) | Paga **R$ 500k** vs R$ 600k = **economia de R$ 100k** líquida |

<aside>
💡

**Insight: a comissão é "invisível" quando o ganho é grande**

No caso acima, o cedente ganha R$ 120k a mais que no distrato — depois de pagar a comissão. O comprador economiza R$ 100k — depois de pagar a comissão. A comissão total do RepasseSeguro é R$ 50k, que representa **~23% do valor criado** para as partes (R$ 220k total).

O RepasseSeguro captura fração do valor que cria — e ambas as partes saem significativamente melhores do que sem a plataforma.

</aside>

---

## 3. Estrutura do Modelo (2 receitas)

<aside>
🎯

**Princípio central**

- Monetizar apenas quando existe **resultado econômico real e mensurável**.
- Manter o produto simples: comissão de duas pontas, regras objetivas e auditáveis.
- Zero dependência de serviços recorrentes, assinatura ou capital próprio.
</aside>

### 3.1 Visão consolidada

| **Receita** | **Quem paga** | **Base de cálculo** | **Quando paga** |
| --- | --- | --- | --- |
| **Comissão do cedente** | Quem repassa o contrato | Cenários B/C/D: **20%** × (Valor Recuperado − Valor Distrato Referência). Cenário A: R$ 0. | No Fechamento |
| **Comissão do comprador** | Quem compra o repasse | **20%** × Δ = max(0, Tabela Atual − Tabela do Contrato). Exceção (Cenário A + Δ = 0): 20% × Valor Pago pelo Cedente. | No Fechamento |

### 3.2 O que NÃO é receita (limites do modelo)

- **Não há assinatura** ou mensalidade na plataforma.
- **Não há fee por anúncio** ou cadastro.
- **Não há cobrança de lead** ou conexão.
- **Não há spread** — o RepasseSeguro não assume posição financeira.
- **Não há escrow proprietário** — o RS utiliza uma **Conta Escrow via parceiro regulado** como intermediário neutro para retenção e liberação dos valores, garantindo segurança e eliminando risco de inadimplência.

<aside>
🔴

**Alerta de governança (para proteger o modelo)**

- Se as referências de tabela e datas não forem travadas, vira disputa.
- Precisa estar escrito: qual tabela, qual data, qual fonte e como tratar exceções.
- Sem evento de Fechamento inequívoco, não há gatilho de cobrança.
</aside>

---

## 4. Arquitetura da Plataforma — Perfis de Acesso

<aside>
🎯

**Objetivo desta seção**

Definir os três ambientes distintos da plataforma Repasse Seguro, cada um desenhado para atender às necessidades específicas do seu perfil de usuário — sem sobreposição de contexto e sem exposição de dados sensíveis entre as partes.

</aside>

A plataforma opera com **3 dashboards independentes**, cada um com escopo, permissões e experiência próprias:

| **#** | **Dashboard** | **Perfil** | **Escopo de Acesso** |
| --- | --- | --- | --- |
| 1 | **Dashboard do Cedente** | Pessoa física ou jurídica que deseja repassar seu contrato imobiliário | Visão exclusiva dos seus próprios casos: status de triagem, formalização, fechamento e comissão devida. Sem acesso a dados do comprador ou de outros cedentes. |
| 2 | **Dashboard do Cessionário** | Pessoa física ou jurídica interessada em adquirir contratos repassados | Visão dos contratos disponíveis para aquisição, histórico de lances, casos em negociação e comissões. Sem acesso a dados do cedente além do necessário para o fechamento. |
| 3 | **Dashboard Admin (Repasse Seguro)** | Equipe interna do Repasse Seguro | Acesso total: todos os casos, dossiês, comissões, status de formalização, histórico de comunicações, relatórios financeiros e painel de gestão operacional. |

<aside>
🔴

**Princípio de isolamento de dados**

Os dashboards do Cedente e do Cessionário são **completamente isolados entre si**. Nenhuma das partes tem acesso a dados da outra além do estritamente necessário para o fechamento do caso — protegendo a confidencialidade comercial e a integridade do processo.

</aside>

<aside>
⚙️

**Dependência de implementação**

A separação de ambientes exige controle de autenticação e permissões por perfil (RBAC). O Admin tem visão consolidada e ferramentas de gestão que não estão disponíveis nos perfis externos.

</aside>

---

## 5. Receitas futuras (roadmap pós-lançamento)

<aside>
⚙️

**Receitas planejadas para fases futuras (NÃO incluídas no lançamento inicial)**

Estas receitas estão no roadmap, mas **não entram nas projeções do lançamento** e **não devem ser consideradas na análise de viabilidade inicial**. Servem para demonstrar upside ao investidor.

</aside>

| **Receita futura** | **Como funciona** | **Fase estimada** | **Dependência** |
| --- | --- | --- | --- |
| White-label para incorporadoras | Ambiente personalizado de repasse (modelo Passu adaptado), com fee de plataforma | Pós-validação (12–18 meses) | Volume + parcerias com incorporadoras |
| Referral para parceiros | Comissão de indicação para advogados, despachantes, cartórios, crédito | Pós-lançamento (6–12 meses) | Base de parceiros qualificados |
| Selo "Verificado Premium" | Verificação aprofundada com laudo + certificação, fee do cedente | Pós-lançamento (6–12 meses) | Processo de verificação maduro |
| Dados transacionais | "Tabela RepasseSeguro" — referência de preço de cessão por região/incorporadora | Longo prazo (24+ meses) | Volume de transações + base de dados |

---

## 6. Definição Única de "Fechamento" (gatilho de cobrança)

<aside>
⚙️

**Definição objetiva (vale para cedente e comprador)**

Considera-se **Fechamento** quando TODOS os itens abaixo forem verdadeiros:

1. Existe **instrumento assinado** pelas partes (termo de cessão/repasse, aditivo contratual ou documento equivalente).
2. O **preço do repasse** está confirmado por evidência (termo/recibo/condição acordada) e há ciência/aceite das partes.
3. As **condições mínimas de formalização** do caso foram atendidas conforme o termo comercial (ex.: documentação mínima entregue e etapa de formalização atingida).

**Observação:** Fechamento **não** significa "imóvel registrado" ou "aprovado por incorporadora" em 100% dos cenários. Significa o evento inequívoco acordado que dispara a cobrança.

</aside>

<aside>
🔴

**Para evitar disputa**

- Se qualquer um dos 3 itens acima não ocorrer, **não há cobrança de comissão**.
- Exceções precisam estar escritas no termo comercial do caso.
</aside>

<aside>
🖥️

**Responsabilidade por dashboard no Fechamento**

- **Admin:** visualiza e confirma o status de Fechamento de todos os casos, com acesso ao dossiê completo.
- **Cedente:** recebe notificação de Fechamento e assina o instrumento via seu dashboard.
- **Cessionário:** confirma preço e condições via seu dashboard antes do Fechamento ser registrado.
</aside>

---

## 7. Receita 1 — Cenários de Retorno do Cedente

<aside>
💡

Ao cadastrar o imóvel na plataforma, o cedente **escolhe o cenário de retorno desejado** — que define o quanto quer recuperar e qual será a comissão da RS pelo serviço prestado.

Quanto maior o retorno esperado, **mais difícil será encontrar um comprador**. Essa informação é apresentada de forma clara e transparente no momento do cadastro.

Caso não seja possível encontrar comprador para o cenário escolhido, o cedente poderá aceitar um cenário inferior mediante **termos de aceite formais** — preservando a segurança jurídica de todas as partes.

</aside>

### 7.1 Os 4 Cenários de Retorno

<aside>
⚙️

**Definição dos Cenários**

O cedente escolhe **um dos 4 cenários** no momento do cadastro do imóvel. Cada cenário define a meta de retorno e a respectiva comissão da RS.

</aside>

| **Cenário** | **Retorno esperado** | **Comissão da RS** | **Observação** |
| --- | --- | --- | --- |
| **A** | Apenas assumir o saldo devedor | **R$ 0** — RS não cobra comissão | Pior cenário para o cedente. Mais fácil de fechar. O cedente livra-se do contrato sem receber nada além da quitação do saldo. |
| **B** | Valor pago (100% recuperado) | **20%** sobre (Valor Recuperado − Valor Distrato Referência) | O cedente recupera tudo que pagou. Cenário intermediário de dificuldade. |
| **C** | Valor pago + 30% | **20%** sobre (Valor Recuperado − Valor Distrato Referência) | O cedente recupera o valor pago com valorização de 30%. Mais difícil de fechar que B. |
| **D** | Valor pago + 50% | **20%** sobre (Valor Recuperado − Valor Distrato Referência) | Máximo retorno. Maior dificuldade para encontrar comprador. Melhor cenário para o cedente. |

<aside>
🔴

**Dificuldade crescente por cenário**

Quanto maior o retorno desejado, menor a atratividade para o comprador e mais difícil o fechamento. O cedente deve ser informado disso com clareza durante o cadastro.

**Fluxo de escalonamento:** D → C → B → A. Se não fechar no cenário escolhido, o cedente desce para o cenário inferior mediante aceite formal.

</aside>

### 7.2 Fórmula de Comissão da RS (Cenários B, C e D)

<aside>
⚙️

**Fórmula padrão (Cenários B, C e D)**

- Comissão_RS = **20%** × (Valor_Recuperado − Valor_Distrato_Referência)

**Exemplo ilustrativo — Cenário C (valor pago + 30%):**

- Valor pago pelo cedente: R$ 200.000
- Valor Recuperado (100% + 30%): R$ 260.000
- Valor Distrato Referência (ex: 50% retido): R$ 100.000
- Base de cálculo: R$ 260.000 − R$ 100.000 = **R$ 160.000**
- Comissão RS: 20% × R$ 160.000 = **R$ 32.000**

**No Cenário A:** comissão da RS = **R$ 0** em todos os casos.

</aside>

### 7.3 Escalonamento e Termos de Aceite

Ao cadastrar o imóvel, o cedente assina um **termo de aceite de escalonamento** que autoriza a RS a ofertar o imóvel no cenário escolhido e, se não houver comprador dentro de um prazo definido, oferecê-lo no cenário imediatamente inferior.

| **Etapa** | **Ação** | **Condição para avançar** |
| --- | --- | --- |
| 1ª tentativa | Oferta no cenário escolhido pelo cedente | Prazo definido sem comprador → avança com aceite formal do cedente |
| 2ª tentativa | Oferta no cenário imediatamente inferior | Novo prazo sem comprador → avança com novo aceite |
| 3ª tentativa | Próximo cenário inferior | E assim sucessivamente até o Cenário A |
| Cenário A | Apenas assumir saldo devedor | RS não cobra comissão neste cenário |

<aside>
✅

**Governança do escalonamento**

Cada transição de cenário exige **aceite formal do cedente** (assinatura eletrônica via módulo ZapSign — ver seção 8-A). Nenhum cenário é alterado de forma unilateral pela RS.

</aside>

### 7.4 Evento de Cobrança

No **Fechamento**, conforme definido na seção 6.

### 7.5 Política de Desconto e Negociação do Percentual

<aside>
🎯

**Objetivo desta subseção**

Estabelecer uma **régua de referência** para negociações de percentual de comissão, evitando que o time comercial crie precedentes inconsistentes nos primeiros casos. O percentual padrão é 20%, mas pode ser ajustado dentro das faixas abaixo.

</aside>

#### 7.5.1 Tabela Indicativa de Pisos por Faixa de Valor Recuperado

| **Faixa de Valor Recuperado** | **% Padrão** | **% Mínimo (piso)** | **Racional** |
| --- | --- | --- | --- |
| Até R$ 150.000 | **20%** | **20%** | Casos pequenos — comissão já é baixa em valor absoluto. Sem margem para desconto. |
| R$ 150.001 – R$ 300.000 | **20%** | **18%** | Faixa intermediária. Desconto pontual permitido para fechar caso estratégico. |
| R$ 300.001 – R$ 700.000 | **20%** | **15%** | Casos de ticket alto. Comissão em valor absoluto já é significativa mesmo com desconto. |
| Acima de R$ 700.000 | **20%** | **12%** | Casos premium. 20% pode parecer excessivo para o cedente; desconto preserva competitividade sem comprometer margem. |

<aside>
🔴

**Regras de governança da negociação**

1. **Qualquer desconto abaixo do padrão de 20%** deve ser registrado no Termo Comercial com justificativa.
2. **Desconto abaixo do piso da faixa** requer aprovação da liderança (Admin nível gerencial).
3. **Comissão zero** só existe no Cenário A — nunca como desconto comercial nos cenários B/C/D.
4. Esta tabela é uma **referência indicativa** para o lançamento. Deve ser revisada após os primeiros 20 casos fechados com base nos dados reais (ver V11 na seção 17.3).
</aside>

#### 7.5.2 Comissão do Comprador — Mesma Lógica

A mesma régua de negociação se aplica à comissão do comprador (20% sobre Δ), com os seguintes ajustes:

| **Faixa de Δ (valorização)** | **% Padrão** | **% Mínimo (piso)** | **Racional** |
| --- | --- | --- | --- |
| Até R$ 100.000 | **20%** | **20%** | Comissão máxima de R$ 20k — sem margem para desconto. |
| R$ 100.001 – R$ 300.000 | **20%** | **17%** | Economia do comprador já é expressiva. Desconto pontual permitido. |
| Acima de R$ 300.000 | **20%** | **13%** | Casos de alta valorização. Comissão absoluta é alta; desconto mantém atratividade. |

<aside>
💡

**Leitura analítica**

A régua de pisos garante que, mesmo com desconto máximo, o **ticket mínimo por caso não caia abaixo do custo incremental** (R$ 5k–9k). Exemplo: caso com Valor Recuperado de R$ 150k e Δ de R$ 100k, com desconto máximo:

- Comissão cedente: 20% × R$ 150k (base) = R$ 30k (sem desconto nessa faixa)
- Comissão comprador: 20% × R$ 100k = R$ 20k (sem desconto nessa faixa)
- **Ticket total: R$ 50k** — margem de contribuição preservada.

Em casos maiores (Recuperado R$ 500k, Δ R$ 200k), com desconto máximo: cedente 15% × R$ 350k base = R$ 52,5k + comprador 17% × R$ 200k = R$ 34k = **R$ 86,5k** — margem ainda mais alta em valor absoluto.

</aside>

---

## 8. Receita 2 — Comissão do Investidor (Comprador)

<aside>
🎯

**Objetivo**

Tornar o cadastro do investidor completamente gratuito, sem fricção de entrada. A comissão incide apenas quando o investidor fecha um negócio — alinhando o incentivo da RS com o resultado real do comprador.

</aside>

### 8.1 Cadastro Gratuito — Zero Mensalidade

<aside>
✅

**Regra de acesso**

O investidor ao se cadastrar na plataforma **não paga nada**. Nenhuma taxa de cadastro, nenhuma mensalidade, nenhum fee recorrente.

A cobrança ocorre **exclusivamente no Fechamento** de um negócio, sobre a base definida abaixo.

</aside>

### 8.2 Base de Cálculo — Regra Padrão

A comissão do investidor incide sobre a **diferença entre a Tabela Atual e a Tabela do imóvel em contrato** (preço original firmado entre o cedente e a construtora).

<aside>
⚙️

**Fórmula padrão (todos os cenários, exceto a exceção da seção 8.3)**

- Δ = Tabela_Atual − Tabela_Contrato
- Comissão_Investidor = **20%** × Δ

**Onde:**

- **Tabela_Atual:** preço vigente do imóvel na data do lance do comprador (hierarquia de fontes: construtora → oferta pública → avaliação formal).
- **Tabela_Contrato:** valor do imóvel conforme contrato original firmado entre o cedente e a construtora.

**Se Δ ≤ 0:** comissão do investidor = **R$ 0** (não há valorização real a tributar).

</aside>

### 8.3 Exceção — Cenário A sem Valorização

Quando o cedente estiver no **Cenário A** (apenas assumir saldo devedor) **e** não houver valorização do imóvel (Tabela_Atual = Tabela_Contrato, Δ = 0), aplica-se a regra de exceção:

<aside>
⚙️

**Fórmula de exceção (Cenário A + Δ = 0)**

- Comissão_Investidor = **20%** × Valor_Pago_pelo_Cedente

**Justificativa:** nesse cenário, o investidor adquire o imóvel sem desconto de tabela mas se beneficia da estrutura operacional e jurídica da RS. A comissão incide sobre o valor que o cedente investiu no contrato como base de referência do negócio.

</aside>

<aside>
🔴

**Condição de aplicação da exceção — ambos os critérios obrigatórios**

1. O cedente está no **Cenário A** (sem retorno financeiro além da quitação do saldo devedor).
2. Δ = 0 — a tabela atual **não apresenta valorização** em relação à tabela do contrato original.

Se o Δ for positivo, mesmo no Cenário A, aplica-se a regra padrão (20% sobre Δ).

</aside>

### 8.4 Resumo Consolidado por Cenário do Cedente

| **Cenário** | **Situação do Δ** | **Fórmula — Comissão do Investidor** | **Comissão RS sobre o Cedente** |
| --- | --- | --- | --- |
| **A** | Δ = 0 (sem valorização) | 20% × Valor_Pago_pelo_Cedente *(exceção)* | R$ 0 |
| **A** | Δ > 0 (com valorização) | 20% × Δ *(regra padrão)* | R$ 0 |
| **B** | Qualquer Δ | 20% × Δ *(regra padrão)* | 20% × (Valor Recuperado − Valor Distrato Ref.) |
| **C** | Qualquer Δ | 20% × Δ *(regra padrão)* | 20% × (Valor Recuperado − Valor Distrato Ref.) |
| **D** | Qualquer Δ | 20% × Δ *(regra padrão)* | 20% × (Valor Recuperado − Valor Distrato Ref.) |

### 8.5 Hierarquia de Fontes da Tabela Atual

1. **Tabela oficial da construtora** vigente na data do lance do comprador
2. **Oferta pública comprovável** vigente na mesma data
3. **Avaliação formal** (laudo), quando a construtora não publica tabela

<aside>
⚙️

**Evidência obrigatória no dossiê**

O Analista de Formalização deve anexar **PDF/print/link** da tabela do mês + documento do preço do contrato original **antes do fechamento ser liberado**. Sem evidência, o fechamento não é concluído.

</aside>

### 8.6 Evento de Cobrança

No **Fechamento**, conforme definido na seção 6.

---

## 8-A. Módulo Interno — ZapSign (Assinatura Eletrônica)

<aside>
⚙️

**Dependência de implementação**

Este módulo é uma **integração nativa** da plataforma RS com o ZapSign. O objetivo é que todo o fluxo de assinatura eletrônica ocorra **dentro da própria plataforma** — sem redirecionamento externo — mantendo a experiência unificada e o dossiê centralizado.

</aside>

### 8-A.1 Objetivo

Centralizar **todos os documentos que exigem assinatura** no fluxo do Repasse Seguro dentro de um módulo integrado ao ZapSign, eliminando ferramentas externas, processos manuais e risco de dossiê incompleto.

### 8-A.2 Documentos que passarão pelo módulo ZapSign

| **Documento** | **Signatários** | **Momento no fluxo** |
| --- | --- | --- |
| Termo de cadastro do cedente | Cedente | Onboarding do cedente |
| Termos de aceite de escalonamento de cenário | Cedente | Mudança de cenário (D → C → B → A) |
| Instrumento de cessão/repasse | Cedente + Cessionário | Fechamento |
| Termo Comercial (comissões) | Cedente e/ou Cessionário + RS | Pré-fechamento |
| Confirmação de fechamento | Todas as partes | Evento de Fechamento |

<aside>
✅

**Benefício operacional**

Todos os documentos assinados ficam **automaticamente arquivados no dossiê do caso** dentro do Admin. O Analista de Formalização tem visibilidade em tempo real do status de cada assinatura pendente — sem precisar checar ferramentas externas.

</aside>

---

## 8-B. Conta Escrow — Mecanismo de Recebimento e Liquidação

<aside>
🎯

**Objetivo desta seção**

Definir o funcionamento da **Conta Escrow** como mecanismo central de recebimento e liquidação do Repasse Seguro — garantindo segurança, neutralidade e eliminação do risco de inadimplência para todas as partes.

</aside>

### 8-B.1 O que é a Conta Escrow

A Conta Escrow, ou "conta garantia/caução", é um **mecanismo bancário ou virtual que atua como intermediário neutro** em transações de alto risco ou valor, como imóveis ou fusões, **retendo os recursos até que todas as condições contratuais sejam cumpridas** — garantindo segurança e mitigando o risco de inadimplência.

No contexto do Repasse Seguro, a Conta Escrow é o **canal pelo qual a RS recebe suas comissões** — eliminando cobrança direta, reduzindo fricção e criando confiança entre as partes.

<aside>
✅

**Benefícios da Conta Escrow para o modelo RS**

1. **Eliminação do risco de inadimplência** — o valor é retido antes da liberação, não depois.
2. **Neutralidade** — nenhuma das partes controla os recursos até o cumprimento das condições.
3. **Confiança** — cedente e comprador sabem que o dinheiro só é movimentado quando todos os critérios de Fechamento (seção 6) forem cumpridos.
4. **Automatização** — liberação pode ser acionada automaticamente pelo Admin após validação do Fechamento.
5. **Compliance simplificado** — a RS não opera a conta diretamente, utiliza parceiro bancário ou fintech regulado.
</aside>

### 8-B.2 Fluxo Operacional da Conta Escrow

| **Etapa** | **Ação** | **Responsável** |
| --- | --- | --- |
| 1 | Comprador (cessionário) confirma lance e valor do negócio na plataforma | Cessionário |
| 2 | Comprador realiza depósito na **Conta Escrow vinculada ao caso** — valor total do negócio (preço do repasse + comissão do comprador) | Cessionário |
| 3 | Conta Escrow **retém o valor** até confirmação do Fechamento pelo Admin (3 critérios cumulativos da seção 6) | Conta Escrow (automático) |
| 4 | Após Fechamento confirmado: **liberação simultânea** — RS recebe comissão (cedente + comprador), cedente recebe valor líquido acordado | Admin (confirma) → Conta Escrow (executa) |
| 5 | Início do prazo de reversão de **15 dias corridos** (seção 12) | Sistema (automático) |
| 6 | Em caso de reversão dentro dos 15 dias: **estorno automático** da Conta Escrow ao comprador | Admin (confirma) → Conta Escrow (executa) |
| 7 | Após 15 dias sem reversão: comissão **definitiva** e caso Concluído | Sistema (automático) |

<aside>
💡

**Leitura analítica**

A Conta Escrow resolve dois problemas simultâneos: (1) elimina o risco de inadimplência da comissão — o valor já está retido antes da liberação — e (2) reduz a fricção de cobrança pós-fechamento, que no modelo anterior dependia de Pix/TED com prazo de 5 dias úteis e gerava risco de atraso. Com escrow, o **Fechamento e o recebimento da comissão acontecem no mesmo evento**.

</aside>

### 8-B.3 Comissão do Cedente via Escrow

<aside>
⚙️

**Mecânica de retenção**

A comissão do cedente (20% sobre Valor Recuperado − Valor Distrato Referência, nos cenários B/C/D) é **retida automaticamente na Conta Escrow** no momento da liberação do valor ao cedente.

Ou seja: o cedente recebe o **valor líquido** — já descontada a comissão — e a RS recebe a comissão diretamente da conta.

**Exemplo:** Valor Recuperado R$ 300k, Distrato Ref R$ 150k → Comissão RS = R$ 30k → Cedente recebe R$ 270k líquidos.

</aside>

### 8-B.4 Comissão do Comprador via Escrow

<aside>
⚙️

**Mecânica de retenção**

O comprador deposita na Conta Escrow o valor total: **Preço do Repasse + Comissão do Comprador**.

No Fechamento, a Conta Escrow libera:

- **Comissão do comprador** → para a RS
- **Preço do Repasse** (líquido da comissão do cedente) → para o cedente
- **Comissão do cedente** → para a RS

**Exemplo:** Preço do Repasse R$ 300k + Comissão Comprador R$ 20k = Depósito total R$ 320k. No Fechamento: RS recebe R$ 20k (comprador) + R$ 30k (cedente) = R$ 50k; Cedente recebe R$ 270k.

</aside>

### 8-B.5 Regras de Operação

| **Regra** | **Descrição** |
| --- | --- |
| Vinculação por caso | Cada caso ativo gera uma **Conta Escrow exclusiva**, identificada pelo ID do caso na plataforma |
| Depósito obrigatório | O caso só avança para **Em Formalização** (estado 6, seção 8-D) após confirmação do depósito na escrow |
| Liberação condicionada | Valores só são liberados após os **3 critérios de Fechamento** (seção 6) serem confirmados pelo Admin |
| Estorno por reversão | Se houver desistência formal dentro dos 15 dias (seção 12), estorno integral automático ao comprador |
| Prazo de liberação | Até **2 dias úteis** após confirmação do Fechamento pelo Admin |
| Parceiro regulado | A Conta Escrow é operada por **parceiro bancário ou fintech regulado** — a RS não opera a conta diretamente e não necessita de licença própria do Banco Central |

<aside>
🔴

**Pontos de atenção**

1. **Seleção do parceiro escrow:** avaliar provedores regulados (ex: Escrow Inc, Celcoin, bancos digitais com API de escrow) quanto a taxas, SLA de liberação e integração via API.
2. **Custo da escrow:** taxas do parceiro devem ser incorporadas ao custo incremental por caso (seção 9) — impacto estimado de R$ 200–500/caso.
3. **Experiência do comprador:** o depósito antecipado pode gerar fricção. O fluxo deve comunicar claramente que o valor está **protegido e só é liberado após cumprimento das condições**.
</aside>

---

## 8-C. Condição da Construtora — Anuência e Adimplência

<aside>
🔴

**Regra crítica de fluxo**

A construtora **só autoriza a cessão (anuência)** se **todas as parcelas do contrato original estiverem em dia** no momento da solicitação de transferência.

Contratos com parcelas em atraso **não podem ser repassados** enquanto a inadimplência não for regularizada pelo cedente.

</aside>

### 8-C.1 Impacto no fluxo operacional

| **Situação no cadastro** | **Ação da RS** |
| --- | --- |
| Parcelas em dia | Fluxo normal. Caso segue para triagem e oferta ao investidor. |
| Parcelas em atraso | Caso **bloqueado**. Cedente é informado da pendência e orientado sobre regularização antes de prosseguir. |
| Parcelas regularizadas após bloqueio | Cedente reapresenta comprovante. Admin reativa o caso na plataforma. |
| Construtora nega anuência por outro motivo | Registrar no dossiê e escalar para análise jurídica (ver seção 17.2 — V9). |

<aside>
⚙️

**Dependência de implementação**

A triagem inicial do cadastro do imóvel deve incluir um **campo obrigatório de declaração de adimplência** pelo cedente, com upload de comprovante de pagamento das últimas parcelas. O Admin valida a documentação antes de ativar o caso na plataforma.

</aside>

---

## 8-D. Ciclo de Vida do Caso — Estados e Transições

<aside>
🎯

**Objetivo desta seção**

Mapear todos os **estados possíveis** de um caso dentro da plataforma, as **condições de transição** entre eles e o **responsável** por cada mudança de estado. Sem isso, a plataforma não tem máquina de estados para construir — e o time operacional não tem visibilidade do funil real.

</aside>

### 8-D.1 Diagrama de Estados

| **#** | **Estado** | **Descrição** | **Responsável** | **Dashboard** |
| --- | --- | --- | --- | --- |
| 1 | **Captado** | Cedente cadastrou o imóvel na plataforma. Dados iniciais preenchidos, aguardando triagem. | Cedente (self-service) | Cedente |
| 2 | **Em Triagem** | Analista de Formalização valida documentação, adimplência e elegibilidade do contrato (ver seção 8-C). | Analista / Admin | Admin |
| 3 | **Qualificado** | Caso aprovado na triagem. Dossiê mínimo completo. Pronto para oferta ao mercado de compradores. | Analista / Admin | Admin |
| 4 | **Oferta Ativa** | Caso publicado no Dashboard do Cessionário. Compradores podem visualizar e fazer lances. | Admin | Cessionário + Admin |
| 5 | **Em Negociação** | Pelo menos um lance recebido. Comprador e cedente em processo de alinhamento de valor e condições. | Admin (mediação) | Todos |
| 6 | **Em Formalização** | Valor aceito por ambas as partes. Documentação final sendo preparada — Termo Comercial, instrumento de cessão, anuência da construtora. | Analista / Admin | Todos |
| 7 | **Fechamento** | Todos os 3 critérios da seção 6 cumpridos. Comissão cobrada. Caso considerado concluído operacionalmente. | Admin | Todos |
| 8 | **Pós-Fechamento** | Período de 15 dias corridos para reversão (seção 12). Comissão monitorada. Caso sob observação. | Admin | Admin |
| 9 | **Concluído** | Prazo de reversão expirado sem desistência. Comissão definitiva. Caso arquivado. | Sistema (automático) | Admin |

### 8-D.2 Estados Excepcionais

| **Estado** | **Quando ocorre** | **Pode voltar ao fluxo?** |
| --- | --- | --- |
| **Bloqueado** | Inadimplência do cedente detectada na triagem ou pendência documental crítica | Sim — após regularização e revalidação pelo Admin |
| **Escalonado** | Caso não fechou no cenário atual e cedente aceitou cenário inferior (seção 7.3) | Sim — volta para **Oferta Ativa** no novo cenário |
| **Em Reversão** | Desistência formal comunicada dentro dos 15 dias (seção 12) | Depende da mediação — pode ser **Cancelado** ou voltar a **Concluído** |
| **Cancelado** | Cedente desistiu, caso inviável ou reversão confirmada | Não — caso arquivado definitivamente |

### 8-D.3 Regras de Transição

<aside>
⚙️

**Regras obrigatórias de transição**

1. **Captado → Em Triagem:** automático após preenchimento completo do cadastro pelo cedente.
2. **Em Triagem → Qualificado:** requer validação do Analista + dossiê mínimo completo (declaração de adimplência + comprovante de parcelas + contrato original).
3. **Qualificado → Oferta Ativa:** decisão do Admin de publicar o caso. Pode haver fila de priorização.
4. **Oferta Ativa → Em Negociação:** automático ao receber o primeiro lance válido de um comprador.
5. **Em Negociação → Em Formalização:** ambas as partes aceitaram valor e condições. Admin confirma e inicia documentação.
6. **Em Formalização → Fechamento:** 3 critérios cumulativos da seção 6 cumpridos.
7. **Fechamento → Pós-Fechamento:** automático — inicia contagem de 15 dias corridos.
8. **Pós-Fechamento → Concluído:** automático após 15 dias sem desistência formal.
9. **Qualquer estado → Cancelado:** mediante decisão do Admin com justificativa no dossiê.
</aside>

<aside>
🔴

**Princípio de não-regressão**

Um caso **nunca volta para um estado anterior** no fluxo principal (exceto via escalonamento de cenário, que reinicia a partir de Oferta Ativa). Isso garante integridade da trilha de auditoria e evita loops operacionais.

</aside>

---

## 8-F. Superinteligência da Plataforma — Agentes de IA por Perfil

<aside>
🎯

**Objetivo desta seção**

Definir a arquitetura de inteligência artificial da plataforma Repasse Seguro: **dois agentes especializados** — um para o cedente e outro para o cessionário — operando sobre uma **engine única compartilhada**, com objetivos, personalidades e escopos de dados completamente diferentes.

A IA não é uma feature auxiliar. É o **moat real da plataforma** — a camada que absorve custo operacional, gera retenção, acelera decisões e cria defensibilidade baseada em dados proprietários.

</aside>

### 8-F.1 Arquitetura: Uma Engine, Dois Agentes

A plataforma opera com **dois agentes de IA independentes**, cada um acessível via **chat interno 24h** no dashboard do respectivo perfil. Os agentes compartilham a mesma infraestrutura de motor de cálculo e base de conhecimento, mas são **completamente isolados em termos de dados e objetivos**.

| **Dimensão** | **Agente do Cedente** | **Agente do Cessionário** |
| --- | --- | --- |
| **Nome interno** | 🛡️ Guardião do Retorno | 📊 Analista de Oportunidades |
| **Missão** | Maximizar o retorno do cedente e reduzir a ansiedade de quem está em momento de vulnerabilidade financeira | Transformar o investidor em uma máquina de decisão rápida e informada, entregando análises que normalmente exigiriam um assessor financeiro |
| **Tom de comunicação** | Empático, calmo, educativo — guia quem está inseguro | Analítico, direto, orientado a dados — fala a língua do investidor |
| **Dashboard** | Dashboard do Cedente | Dashboard do Cessionário |
| **Disponibilidade** | 24 horas por dia, chat interno | 24 horas por dia, chat interno |

<aside>
🔴

**Princípio de isolamento de dados (seção 4)**

O agente do cedente **nunca** acessa dados do cessionário. O agente do cessionário **nunca** acessa dados sensíveis do cedente (valores pagos, cenário escolhido, termos de negociação). A mesma regra de RBAC dos dashboards se aplica integralmente aos agentes de IA.

</aside>

---

### 8-F.2 Agente do Cedente — 🛡️ Guardião do Retorno

<aside>
🎯

**Objetivo**

Ser o **braço direito do cedente** durante toda a jornada — do cadastro ao fechamento. O cedente está, na maioria dos casos, em momento de pressão financeira. A IA reduz incerteza, acelera decisões e guia o processo com clareza e empatia.

</aside>

| **#** | **Funcionalidade** | **Descrição** | **Benefício** |
| --- | --- | --- | --- |
| 1 | **Simulador de cenários em tempo real** | O cedente informa quanto pagou e qual o imóvel. A IA mostra instantaneamente o que receberia em cada cenário (A/B/C/D) vs. distrato, com linguagem simples, visual e sem jargão. | Decisão de cenário informada, sem depender de atendimento humano |
| 2 | **Cadastro assistido inteligente** | Guia o cedente passo a passo no preenchimento do dossiê: explica por que cada documento é necessário, avisa o que está faltando, valida dados em tempo real e dá estimativa de prazo para conclusão. | Dossiê mais completo na triagem, menos retrabalho para o Analista |
| 3 | **Status inteligente do caso** | Não apenas mostra "Em Triagem" — **explica** o que isso significa, o que está acontecendo por trás, qual o próximo passo e quanto tempo falta (baseado nos SLAs da seção 8-E). | Elimina 80% das mensagens de "qual o status do meu caso?" |
| 4 | **Análise de escalonamento** | Quando o cenário escolhido não atrai comprador, a IA apresenta proposta de descer de cenário com **simulação comparativa clara**: "No cenário C você receberia R$ X. No cenário B, R$ Y. A diferença é R$ Z, mas a chance de fechar sobe de ~30% para ~70%." | Escalonamento mais rápido e consciente, menos atrito emocional |
| 5 | **FAQ contextual e suporte 24h** | Responde dúvidas sobre anuência, prazos, documentação e comissão — tudo **contextualizado ao caso específico** do cedente, não respostas genéricas de FAQ. | Custo de suporte humano próximo de zero |
| 6 | **Notificações proativas inteligentes** | Avisa quando houve lance, quando o caso mudou de estado, quando precisa de ação do cedente. Tom calmo, empático e direto — com o contexto do que significa cada evento. | Cedente sempre informado sem precisar abrir a plataforma |
| 7 | **Educação financeira sob demanda** | Explica o que é cessão de direitos, o que a Lei 13.786 significa na prática, por que o repasse é melhor que o distrato. Conteúdo entregue quando o cedente pedir, no contexto da dúvida. | Cedente mais confiante e menos dependente de validação externa |

<aside>
💡

**Leitura analítica**

O agente do cedente resolve um problema estrutural da operação: **o cedente é o perfil que mais gera custo de suporte** — porque está inseguro, não entende o processo e precisa de validação constante. Uma IA que guia, simula e educa reduz drasticamente o tempo do time operacional gasto com atendimento, ao mesmo tempo que melhora a qualidade do dossiê e a velocidade de decisão do cedente.

</aside>

---

### 8-F.3 Agente do Cessionário — 📊 Analista de Oportunidades

<aside>
🎯

**Objetivo**

Ser o **analista pessoal do investidor** — entregando análises, comparativos e recomendações que normalmente exigiriam um assessor financeiro dedicado. O investidor não quer ser guiado — quer **dados, velocidade e precisão**.

</aside>

| **#** | **Funcionalidade** | **Descrição** | **Benefício** |
| --- | --- | --- | --- |
| 1 | **Radar de oportunidades personalizado** | O investidor define critérios (região, faixa de preço, Δ mínimo, tipo de imóvel) e a IA filtra, ranqueia e entrega os melhores repasses no perfil dele — **proativamente**, sem que precise buscar. | Investidor recebe oportunidades sem esforço de busca |
| 2 | **Briefing executivo por caso** | Ao abrir um repasse, a IA apresenta resumo executivo com: Δ estimado, comissão projetada, tempo estimado de fechamento, riscos identificados e **score de atratividade** — sem expor dados sensíveis do cedente. | Decisão em minutos, não em horas |
| 3 | **Simulador de retorno** | O investidor pergunta: "Se eu comprar esse repasse por R$ X, quanto economizo vs. tabela? Qual meu custo total com comissão?" A IA calcula em tempo real com todas as variáveis do caso. | Autonomia total para modelar cenários financeiros |
| 4 | **Comparativo entre repasses** | "Compare os 3 melhores repasses disponíveis na zona sul de SP com Δ acima de R$ 80k." A IA monta tabela comparativa com métricas-chave: preço, Δ, comissão estimada, região, incorporadora. | Análise de portfólio sem esforço manual |
| 5 | **Análise de valorização por região** | Com base nos dados transacionais da plataforma, a IA mostra tendência de Δ por bairro, incorporadora e tipo de imóvel. Este é o embrião da **"Tabela RepasseSeguro"** (receita futura — seção 5). | Inteligência de mercado proprietária |
| 6 | **Alertas de novos repasses** | Push inteligente quando um repasse que se encaixa no perfil do investidor entra em Oferta Ativa. Timing é tudo nesse mercado — quem vê primeiro, fecha primeiro. | Vantagem competitiva de tempo sobre outros investidores |
| 7 | **Suporte 24h contextualizado** | Dúvidas sobre documentação, prazos, anuência, como funciona o lance, como funciona a comissão — tudo contextualizado ao caso em andamento, não respostas genéricas. | Zero dependência do time da RS para dúvidas operacionais |
| 8 | **Histórico de decisões e aprendizado** | A IA aprende com o que o investidor fechou, rejeitou e demonstrou interesse. Com o tempo, as recomendações ficam cada vez mais afiadas e personalizadas. **Isso é a barreira de saída.** | Experiência progressivamente insubstituível |

<aside>
💡

**Leitura analítica**

O agente do cessionário resolve o problema de **discovery** do investidor. Hoje, quem investe em cessão imobiliária depende de corretores, grupos de WhatsApp e pesquisa manual. Um agente que analisa, filtra e entrega repasses personalizados 24h transforma a plataforma no **canal preferencial de acesso a oportunidades** — e isso é o que gera retenção e recorrência de uso, mesmo num modelo sem assinatura.

</aside>

---

### 8-F.4 Camada Compartilhada — Engine Única

<aside>
⚙️

**Dependência de implementação**

Os dois agentes operam sobre uma **infraestrutura única compartilhada**, com os seguintes componentes:

</aside>

| **#** | **Componente** | **Descrição** | **Usado por** |
| --- | --- | --- | --- |
| 1 | **Base de conhecimento unificada** | Regras do modelo de negócios, FAQ, SLAs, fluxos operacionais, legislação (Lei 13.786), glossário de termos — atualizada centralmente. | Ambos os agentes |
| 2 | **Motor de cálculo** | Fórmulas de comissão (cedente e comprador), cálculo de Δ, simulação de cenários A/B/C/D, projeção de retorno, comparativo com distrato. | Ambos os agentes |
| 3 | **Camada de isolamento de dados (RBAC)** | Filtra automaticamente quais dados cada agente pode acessar. O agente do cedente vê **apenas** dados do cedente e do seu caso. O agente do cessionário vê **apenas** repasses disponíveis e dados públicos. | Infraestrutura |
| 4 | **Motor de aprendizado contínuo** | Quanto mais casos são processados, melhor ficam as estimativas de prazo, conversão por cenário, score de atratividade e recomendações personalizadas. | Ambos os agentes |
| 5 | **Motor de recomendação** | Algoritmo que cruza perfil do investidor (critérios, histórico de decisões) com repasses disponíveis para gerar ranking personalizado de oportunidades. | Agente do Cessionário |

---

### 8-F.5 Impacto Estratégico

<aside>
✅

**Por que a IA é o moat real da plataforma**

1. **Redução brutal de custo operacional** — 80% das interações de suporte (dúvidas de status, simulação, documentação) são absorvidas pela IA. O time da RS foca em mediação e fechamento, não em responder "qual o status do meu caso?".
2. **Efeito de rede de dados** — cada caso que passa pela plataforma alimenta a inteligência dos dois agentes. O investidor que usa há 6 meses tem uma experiência incomparavelmente melhor que um novo. O cedente que preenche o cadastro assistido pela IA tem dossiê mais completo, o que acelera triagem. **Ciclo virtuoso.**
3. **Defensibilidade real** — um concorrente pode copiar o modelo de comissão em 1 dia. Copiar uma IA treinada com dados transacionais reais de cessão imobiliária leva anos.
4. **Escala sem custo linear** — os agentes atendem 10 ou 10.000 usuários com o mesmo custo de infraestrutura. Cada novo caso melhora a IA para todos os usuários.
5. **Barreira de saída** — quanto mais o investidor usa o agente, mais o sistema aprende suas preferências. Quanto mais o cedente interage, mais rápido o dossiê fica completo. Trocar de plataforma significa **perder inteligência acumulada**.
</aside>

---

### 8-F.6 Roadmap de Implementação

| **Fase** | **Agente do Cedente** | **Agente do Cessionário** | **Dependência** |
| --- | --- | --- | --- |
| **MVP (Lançamento)** | Simulador de cenários + cadastro assistido + FAQ contextual + suporte 24h | Chat com FAQ inteligente + briefing de caso + suporte 24h sobre processo | Base de conhecimento + LLM integrado + motor de cálculo |
| **V2 (Pós-lançamento)** | Status inteligente do caso + análise de escalonamento + notificações proativas | Simulador de retorno + comparativo entre repasses + análises financeiras sob demanda | Dados de tabela e Δ estruturados + SLAs monitorados |
| **V3 (Escala)** | Educação financeira sob demanda + integração com ZapSign para guiar assinatura | Radar de oportunidades + alertas proativos + análise de valorização por região + aprendizado de preferências | Motor de recomendação + volume de repasses + dados transacionais |

<aside>
💡

**Leitura analítica**

O MVP já entrega valor real para ambos os perfis: o cedente ganha simulação e assistência no cadastro (que reduz custo de triagem), e o cessionário ganha briefing e suporte (que reduz custo de atendimento). As fases V2 e V3 constroem inteligência proprietária — e é aí que a plataforma se torna insubstituível.

A V3 do agente do cessionário é particularmente estratégica: o **radar de oportunidades com aprendizado** cria o mesmo efeito de lock-in que algoritmos de recomendação criam em plataformas de conteúdo — mas aplicado a investimento imobiliário.

</aside>

---

## 8-E. SLA Operacional por Etapa

<aside>
🎯

**Objetivo desta seção**

Definir **prazos-alvo** para cada etapa do ciclo de vida do caso. Sem SLA, o ciclo pode se estender indefinidamente — comprometendo a projeção de volume, a experiência do cedente e a previsibilidade de receita.

</aside>

### 8-E.1 Tabela de SLAs

| **#** | **Transição** | **SLA-alvo** | **SLA máximo** | **Consequência se exceder** |
| --- | --- | --- | --- | --- |
| 1 | Captado → Em Triagem | Imediato | 24 horas | Alerta automático ao Admin |
| 2 | Em Triagem → Qualificado | **3 dias úteis** | 5 dias úteis | Cedente recebe notificação de status + Admin prioriza |
| 3 | Qualificado → Oferta Ativa | **2 dias úteis** | 3 dias úteis | Caso entra automaticamente na fila de publicação |
| 4 | Oferta Ativa → Em Negociação | **15 dias corridos** | 30 dias corridos | Se não houver lance em 30 dias: avaliação de escalonamento de cenário |
| 5 | Em Negociação → Em Formalização | **10 dias úteis** | 20 dias úteis | Admin aciona mediação ativa entre as partes |
| 6 | Em Formalização → Fechamento | **10 dias úteis** | 20 dias úteis | Inclui prazo de anuência da construtora — se travar, escalar conforme seção 8-C |
| 7 | Fechamento → Pós-Fechamento | Imediato | Imediato | Automático |
| 8 | Pós-Fechamento → Concluído | **15 dias corridos** | 15 dias corridos | Fixo — prazo de reversão (seção 12) |

### 8-E.2 Ciclo Total Estimado (Captação → Conclusão)

| **Cenário** | **Ciclo total estimado** | **Observação** |
| --- | --- | --- |
| Otimista (sem gargalos) | **30–40 dias** | Triagem rápida + lance em 7 dias + formalização sem bloqueio |
| Realista (fluxo típico) | **45–60 dias** | Inclui tempo de negociação e anuência da construtora |
| Pessimista (com escalonamento) | **90–120 dias** | Caso escalonou de cenário + anuência travou |

<aside>
💡

**Impacto na projeção de volume**

Se o ciclo realista é de **45–60 dias**, os casos captados no M1 só geram receita no M2–M3. Isso reforça a **nota de cautela da seção 11.2** sobre defasagem de caixa — e significa que o pipeline precisa estar **2 meses à frente** da meta de fechamento.

Para fechar 12 casos/mês em M7, é preciso ter 60 casos captados em M5–M6.

</aside>

<aside>
⚙️

**Dependência de implementação**

Os SLAs devem ser monitorados via **dashboard do Admin** com alertas automáticos quando o SLA-alvo for ultrapassado. A plataforma deve registrar timestamps de cada transição de estado para alimentar métricas de performance operacional.

</aside>

---

## 9. Unit Economics — Anatomia Financeira de Um Caso

<aside>
🎯

**Objetivo desta seção**

Detalhar a economia de um caso individual — desde a receita bruta até a margem de contribuição — para entender a viabilidade unitária e identificar as variáveis que mais impactam o resultado.

</aside>

### 9.1 Caso-tipo (cenário Base)

| **Componente** | **Valor** | **Notas** |
| --- | --- | --- |
| **Receita bruta — comissão cedente** | **R$ 30.000** | 20% × R$ 150k (Valor Recuperado R$ 300k − Distrato Ref R$ 150k) |
| **Receita bruta — comissão comprador** | **R$ 20.000** | 20% × Δ R$ 100k (Tabela Atual R$ 600k − Tabela Contrato R$ 500k) |
| **Receita bruta total** | **R$ 50.000** | Ticket combinado (cedente + comprador) |
| Impostos estimados (ISS + simples/lucro presumido) | R$ 5.000 – R$ 7.500 | 10–15% da receita bruta (depende do regime) |
| **Receita líquida** | **R$ 42.500 – R$ 45.000** |  |
| Custo de triagem e verificação | R$ 1.500 – R$ 2.500 | Tempo de analista + ferramentas de verificação |
| Custo de formalização assistida | R$ 1.500 – R$ 3.000 | Suporte ao cedente e comprador + checklist + trilha |
| Custo jurídico por caso | R$ 500 – R$ 1.500 | Revisão de termos, compliance, dossiê |
| Custo de aquisição (CAC por caso) | R$ 1.000 – R$ 2.000 | Proporção do marketing/distribuição por caso fechado |
| **Custo incremental total** | **R$ 5.000 – R$ 9.000** |  |
| **Margem de contribuição** | **R$ 33.500 – R$ 40.000** |  |
| **Margem de contribuição %** | **67% – 80%** |  |

<aside>
💡

**Leitura analítica**

- A margem de contribuição por caso é **excepcionalmente alta** (67–80%), o que significa que cada caso adicional gera forte contribuição para cobrir custos fixos.
- O **gargalo não é margem — é volume**. A viabilidade do modelo depende de gerar casos suficientes para cobrir o burn fixo mensal.
- O custo incremental por caso tende a **cair com escala** (automação de verificação, templates de formalização, parceiros recorrentes), melhorando a margem de contribuição ao longo do tempo.
</aside>

### 9.2 Comparativo por cenário

| **Métrica** | **Conservador** | **Base (M7–M12)** | **Agressivo** |
| --- | --- | --- | --- |
| Ticket combinado | R$ 35.000 | R$ 50.000 | R$ 65.000 |
| Custo incremental | R$ 8.000 | R$ 7.000 | R$ 5.500 |
| Margem de contribuição | R$ 27.000 | R$ 43.000 | R$ 59.500 |
| Margem % | 77% | 86% | 92% |
| Casos/mês | 5 | 12 | 20 |
| Contribuição mensal total | R$ 135.000 | R$ 516.000 | R$ 1.190.000 |

---

## 10. Análise de Sensibilidade — O Que Mata ou Multiplica o Modelo

<aside>
🎯

**Objetivo**

Identificar quais variáveis têm maior impacto na receita e na viabilidade, para priorizar o que controlar e o que monitorar.

</aside>

### 10.1 Variáveis críticas e impacto na receita anual (cenário Base)

| **Variável** | **Valor Base** | **Se cair 30%** | **Se subir 30%** | **Impacto na receita anual** |
| --- | --- | --- | --- | --- |
| Volume de casos/mês (média) | 9,5 casos | 6,7 → R$ 4,0M | 12,4 → R$ 7,4M | 🔴 **Crítico** — variável #1 |
| Ticket cedente | R$ 30.000 | R$ 21k → R$ 4,7M | R$ 39k → R$ 6,7M | 🟡 **Alto** |
| Ticket comprador | R$ 20.000 | R$ 14k → R$ 5,0M | R$ 26k → R$ 6,4M | 🟡 **Alto** |
| Taxa de conversão | 20% | 14% → 6,7 casos → R$ 4,0M | 26% → 12,4 casos → R$ 7,4M | 🔴 **Crítico** — equivale a volume |
| % de casos com Δ ≤ 0 | 10% estimado | 7% → receita +1% | 30% → receita −8% | 🟢 **Moderado** |
| Taxa de desistência pós-fechamento | 5% estimado | 3% → +R$ 80k | 15% → −R$ 400k | 🟡 **Alto** se subir muito |

<aside>
🔴

**As 2 variáveis que matam o modelo**

1. **Volume de casos fechados** — se cair 50% (de 9,5 para 4,8/mês), a receita cai para R$ 2,9M e o burn de R$ 110k/mês reduz significativamente a margem.
2. **Taxa de conversão** — se a formalização travar (incorporadoras bloqueando anuência), a conversão cai e o volume desaba. Os dois riscos se retroalimentam.

**Mitigação:** fechar incorporadoras piloto antes de abrir o marketplace + pool de compradores pré-cadastrados.

</aside>

### 10.2 Cenário de stress: o que acontece se o modelo operar no pior caso

| **Premissa de stress** | **Valor** | **Impacto** |
| --- | --- | --- |
| Casos fechados/mês | 3 (vs 9,5 Base) | Receita mensal: R$ 150k (3 × R$ 50k ticket Base) |
| Ticket conservador (cedente-only, Δ ≈ 0) | R$ 30k (vs R$ 50k Base) | Receita mensal: R$ 90k |
| Burn mensal | R$ 110k | Déficit mensal: R$ 20k (pior caso) |
| Com R$ 700k de investimento | Runway pós-receita | **35 meses** antes de acabar (déficit de R$ 20k/mês) |

<aside>
💡

**Leitura do stress test**

Mesmo no **pior cenário** (3 casos/mês a R$ 30k), o déficit mensal é de R$ 20k — o que dá aproximadamente **35 meses de runway** com o investimento de R$ 700k. Isso mostra que o modelo é **altamente resiliente a cenários adversos**, porque a margem de contribuição por caso é alta.

O risco real não é falta de margem — é **falta de casos**.

</aside>

---

## 11. Fluxo de Caixa e Breakeven

### 11.1 Breakeven operacional mensal

| **Métrica** | **Cálculo** | **Resultado** |
| --- | --- | --- |
| Burn fixo mensal estimado | Produto + infra + design + go-to-market + jurídico | **R$ 110.000/mês** |
| Margem de contribuição por caso (Base) | R$ 50k receita − R$ 7k custo incremental | **R$ 43.000/caso** |
| **Breakeven = burn ÷ margem por caso** | R$ 110k ÷ R$ 43k | **2,6 → ~3 casos/mês** |

<aside>
✅

**Breakeven em ~3 casos/mês**

Com margem de contribuição de R$ 43k por caso e burn de R$ 110k/mês, o Repasse Seguro atinge breakeven operacional com **3 casos fechados por mês**. No cenário Base, isso acontece já no **M1** (6 casos/mês).

No cenário Conservador (5 casos/mês), o breakeven também é atingido desde o início.

</aside>

### 11.2 Evolução mensal do caixa (cenário Base com investimento de R$ 700k)

| **Mês** | **Aporte** | **Casos** | **Receita** |
| --- | --- | --- | --- |
| M1 | R$ 116.667 | 6 | R$ 300.000 |
| M2 | R$ 116.667 | 6 | R$ 300.000 |
| M3 | R$ 116.667 | 6 | R$ 300.000 |
| M4 | R$ 116.667 | 8 | R$ 400.000 |
| M5 | R$ 116.667 | 8 | R$ 400.000 |
| M6 | R$ 116.665 | 8 | R$ 400.000 |
| M7–M12 | R$ 0 | 12/mês | R$ 600.000/mês |

<aside>
💡

**Leitura analítica do fluxo**

- O cenário Base é **cash flow positivo desde o M1** (receita de R$ 300k > burn de R$ 110k).
- Ao final de 12 meses, o caixa acumulado chega a **~R$ 5M** (investimento + resultado operacional).
- Isso demonstra que o modelo é **autossustentável** muito cedo, desde que o volume mínimo de casos seja atingido.
- **Nota de cautela:** a defasagem de caixa (tempo entre fechamento e recebimento da comissão) não está modelada aqui. Se a defasagem for de 30 dias, M1 recebe R$ 0 de receita e M2 recebe as comissões de M1. Isso desloca o breakeven para M2 e reduz o caixa acumulado nos primeiros meses.
</aside>

---

## 12. Cláusula de Reversão (cancelamento pós-fechamento)

<aside>
🔴

**Regra de reversão**

- Se o comprador ou cedente **desistir formalmente** em até **15 dias corridos** após a data do fechamento, a comissão é **integralmente estornada**.
- Desistência formal = comunicação por escrito (e-mail, termo assinado ou mensagem registrada) com aceite da outra parte.
- Após 15 dias, a comissão é considerada definitiva e não há estorno.

**Objetivo:** proteger as partes contra arrependimento legítimo e demonstrar seriedade do RepasseSeguro.

</aside>

### 12.1 Desistência unilateral (sem aceite da contraparte)

<aside>
⚠️

**Cenário:** uma das partes comunica desistência por escrito dentro dos 15 dias, mas a outra parte **não aceita ou não responde**.

**Regra:**

- A desistência unilateral **sem aceite** dentro dos 15 dias não gera estorno automático.
- O Repasse Seguro abre um **período de mediação de 10 dias úteis** a partir da comunicação de desistência para buscar resolução.
- Se ao fim da mediação não houver acordo: a comissão permanece **retida em conta** até resolução formal.
- O Repasse Seguro **não arbitra** a disputa — apenas mantém a comissão retida e documenta o histórico no dossiê.
</aside>

### 12.2 Impacto financeiro estimado da reversão

| **Taxa de reversão estimada** | **Impacto na receita anual (Base)** | **Impacto no caixa** |
| --- | --- | --- |
| 5% (otimista) | −R$ 285k (de R$ 5,7M para R$ 5,415M) | Gerenciável — comissões retidas compensam parcialmente |
| 10% (realista) | −R$ 570k (de R$ 5,7M para R$ 5,13M) | Impacto moderado — requer provisão de R$ 47,5k/mês |
| 15% (estresse) | −R$ 855k (de R$ 5,7M para R$ 4,845M) | Impacto alto — requer revisão do processo de qualificação |

---

## 13. Forma de Cobrança

<aside>
💳

**Regras de cobrança**

*(Visibilidade por dashboard: Admin acessa histórico completo de todos os casos; Cedente e Cessionário visualizam apenas as cobranças dos seus próprios casos.)*

**Meio de recebimento:**

- Comissão recebida via **Conta Escrow** vinculada a cada caso (ver seção 8-B).
- O valor é **retido na escrow** no momento do depósito do comprador e **liberado automaticamente** para a RS no Fechamento.

**Prazo de liberação:**

- Liberação em até **2 dias úteis** após confirmação do Fechamento pelo Admin.

**Modalidade:**

- Comissão do **comprador**: retida diretamente do depósito total na escrow (Preço do Repasse + Comissão).
- Comissão do **cedente**: retida na escrow antes da liberação do valor líquido ao cedente.
- Cada parte recebe **detalhamento do cálculo** no seu dashboard antes do Fechamento.

**Inadimplência (não-depósito na escrow):**

- Se o comprador não efetuar o depósito na Conta Escrow dentro do prazo estipulado, o caso **não avança para Formalização** — permanece em **Em Negociação** até regularização ou cancelamento.
- Não há cobrança pós-fechamento — o modelo escrow elimina o risco de inadimplência por design.

**Nota fiscal:**

- NFS-e (serviços) emitida no ato da liberação da comissão pela escrow, com descrição "Serviço de intermediação de cessão/repasse imobiliário".
</aside>

---

## 14. Projeções de Receita (12 meses)

<aside>
✅

**Baseline oficial (planejamento)**

- O cenário **Base (rampa)** é a referência oficial de planejamento.
- A projeção Base usa uma **rampa de volume** (M1–M3: 6 casos/mês; M4–M6: 8 casos/mês; M7–M12: 12 casos/mês).
</aside>

### 14.1 Premissas dos cenários

| Cenário | Casos fechados/mês | Ticket cedente | Ticket comprador | Receita média/mês |
| --- | --- | --- | --- | --- |
| Conservador | 5 casos | R$ 20.000 | R$ 15.000 | R$ 175.000 |
| Base (rampa) | 6 → 8 → 12 (média 9,5) | R$ 30.000 | R$ 20.000 | R$ 475.000 |
| Agressivo | 20 casos | R$ 40.000 | R$ 25.000 | R$ 1.300.000 |

### 14.1.1 Premissa de conversão (funil captação → fechamento)

| Cenário | Conversão | Captados/mês | Fechados/mês | Receita/caso |
| --- | --- | --- | --- | --- |
| Conservador | 20% | 25 | 5 | R$ 35.000 |
| Base (M7–M12) | 20% | 60 | 12 | R$ 50.000 |
| Agressivo | 25% | 80 | 20 | R$ 65.000 |

<aside>
⚠️

**Nota sobre o cenário Agressivo**

O cenário Agressivo (20 fechamentos/mês, R$ 15,6M/ano) assume **condições que vão além do crescimento orgânico**: parcerias ativas com corretoras e incorporadoras, canal de indicação estruturado, time comercial dedicado e possível expansão geográfica. Sem essas alavancas, o cenário Base é a referência mais realista.

</aside>

### 14.2 Receita projetada em 12 meses

| Cenário | Receita cedente | Receita comprador | Receita total 12 meses |
| --- | --- | --- | --- |
| Conservador | R$ 1.200.000 | R$ 900.000 | **R$ 2.100.000** |
| Base (rampa) | R$ 3.420.000 | R$ 2.280.000 | **R$ 5.700.000** |
| Agressivo | R$ 9.600.000 | R$ 6.000.000 | **R$ 15.600.000** |

### 14.3 Evolução mensal (cenário Base — rampa)

| Período | Casos/mês | Receita cedente | Receita comprador | Receita total |
| --- | --- | --- | --- | --- |
| M1–M3 | 6 | R$ 540.000 | R$ 360.000 | R$ 900.000 |
| M4–M6 | 8 | R$ 720.000 | R$ 480.000 | R$ 1.200.000 |
| M7–M12 | 12 | R$ 2.160.000 | R$ 1.440.000 | R$ 3.600.000 |

<aside>
💡

**Leitura rápida (cenário Base — rampa)**

- 114 casos fechados/ano (média de **9,5 casos/mês**), com aceleração a partir de M7.
- Proporção de receita: ~**60% cedente / 40% comprador**.
- Base anual: **R$ 5,7M**, com tickets de R$ 30k (cedente) e R$ 20k (comprador).
</aside>

---

## 15. Matriz de Riscos do Modelo de Receita

| **#** | **Risco** | **Probabilidade** | **Impacto** | **Mitigação** |
| --- | --- | --- | --- | --- |
| R1 | **Disputa sobre Tabela Atual** — partes contestam a fonte ou data da tabela de referência | 🟡 Média | 🟡 Alto | Hierarquia de fontes + regra de data do lance + evidência obrigatória no dossiê |
| R2 | **Não-depósito na Conta Escrow** — comprador não efetua o depósito na escrow dentro do prazo, travando o caso antes da formalização | 🟡 Média | 🟢 Moderado | Caso permanece em "Em Negociação" até regularização ou cancelamento. Sem depósito, não há fechamento — logo não há comissão em risco. Mitigação: prazo claro de depósito + notificações proativas + possibilidade de reoferta a outro comprador |
| R3 | **Alta taxa de reversão** — muitos casos desistem dentro dos 15 dias | 🟢 Baixa | 🟡 Alto | Qualificação rigorosa pré-fechamento + processo de formalização assistido |
| R4 | **Concentração de receita no cedente** — casos com Δ ≤ 0 (sem desconto para comprador) reduzem a receita dupla | 🟡 Média | 🟢 Moderado | Filtro de priorização (ticket mínimo do cedente R$ 10k) + curadoria de casos com Δ positivo |
| R5 | **Questionamento regulatório** — órgão regulador questionar a cobrança como intermediação imobiliária sem registro | 🟢 Baixa | 🔴 Crítico | Posicionamento como plataforma de conexão e formalização + termos claros + compliance proativo |
| R6 | **Commoditização da comissão** — concorrentes oferecerem comissões menores para atrair cedentes | 🟡 Média | 🟢 Moderado | Diferencial em governança, trilha de auditoria e formalização (não compete por preço, compete por confiança) |

---

## 16. FAQ — Perguntas Frequentes

### 16.1 O que dispara a cobrança de comissão?

A comissão só é cobrada no **Fechamento**, que acontece quando: (1) existe instrumento assinado entre as partes, (2) o preço do repasse está comprovado por evidência e (3) as condições mínimas de formalização previstas no termo comercial foram cumpridas.

### 16.2 Quem paga comissão?

Existem **duas fontes de receita**: (1) comissão do **cedente** e (2) comissão do **comprador**, cada uma com base de cálculo própria.

### 16.3 Como é calculada a comissão do cedente?

O cedente escolhe um dos **4 cenários de retorno** (A, B, C ou D). No **Cenário A** (apenas assumir saldo devedor), a RS não cobra comissão. Nos **Cenários B, C e D**, a comissão é **20%** sobre (Valor Recuperado − Valor Distrato Referência). Detalhes na seção 7.

### 16.4 Qual a diferença entre os cenários A, B, C e D?

**Cenário A:** cedente apenas transfere o saldo devedor — RS não cobra comissão. **Cenário B:** cedente recupera 100% do valor pago. **Cenário C:** valor pago +30%. **Cenário D:** valor pago +50%. Nos cenários B/C/D, a RS cobra **20%** sobre (Valor Recuperado − Valor Distrato Referência). Detalhes na seção 7.1.

### 16.5 Como é calculada a comissão do comprador?

Sobre o **delta de valorização**: Δ = max(0, Tabela Atual − Tabela do Contrato). Comissão = **20%** × Δ. Exceção: no Cenário A sem valorização (Δ = 0), a comissão é 20% × Valor Pago pelo Cedente. Quando previsto no Termo Comercial, aplica-se piso mínimo de R$ 5.000.

### 16.6 O que acontece se não houver desconto real para o comprador?

Se Δ ≤ 0, a comissão variável do comprador é **R$ 0**. O caso é aceito se a comissão do cedente justificar o custo operacional (sugestão de piso: R$ 10k).

### 16.7 Qual tabela de preços é usada como referência?

A **tabela vigente na data do lance do comprador**, com hierarquia: (1) construtora, (2) oferta pública, (3) avaliação formal. Evidência obrigatória no dossiê.

### 16.8 A comissão pode ser estornada?

Sim. Se houver desistência formal (por escrito, com aceite da outra parte) em até **15 dias corridos** após o fechamento, a comissão é integralmente estornada. Após esse prazo, é definitiva.

### 16.9 Há assinatura, mensalidade ou outras taxas recorrentes?

Não. O modelo é baseado apenas em **comissão de sucesso nas duas pontas**. Zero assinatura, zero mensalidade.

### 16.10 O piso de R$ 5.000 do comprador é obrigatório?

É regra padrão. Pode ser ajustado por caso, desde que registrado no Termo Comercial. Não se aplica automaticamente quando Δ = 0.

### 16.11 E se a construtora mudar a tabela depois do lance?

A referência é a **tabela vigente na data do lance**. Mudanças posteriores não alteram o cálculo daquele caso.

### 16.12 Podem ser negociados percentuais diferentes por caso?

Sim, desde que registrados no Termo Comercial. O documento define a lógica padrão, mas cada caso pode ter condições específicas formalizadas.

### 16.13 É possível cobrar comissão de apenas uma das pontas?

Sim. O desenho prevê duas receitas, mas é possível estruturar casos com comissão apenas do cedente ou apenas do comprador.

### 16.14 Como o RepasseSeguro garante transparência dos cálculos?

Três pilares: (1) fórmulas simples e objetivas neste documento, (2) evidências anexadas ao dossiê, (3) simulações claras apresentadas às partes com valores de comissão explicitados.

### 16.15 Quantos casos precisa captar para atingir as metas?

Com taxa de conversão de 20%, para fechar 12 casos/mês (Base em M7–M12) são necessários ~60 casos captados/mês. Tabela completa na seção 14.1.1.

### 16.16 Por que não usar modelo de spread como o Dinheiro na Planta?

O spread exige capital próprio, assume risco de balanço e limita escala ao capital disponível. O modelo de comissão escala com processo e capacidade operacional — que é mais barato e defensável. Análise completa na seção 2.3.

### 16.17 Qual o breakeven do modelo?

**~3 casos fechados/mês** com ticket Base de R$ 50k, margem de R$ 43k/caso e burn de R$ 110k/mês. Análise completa na seção 11.

---

## 17. Perguntas de Validação em Aberto

### 17.1 Monetização

---

> **V1 — A explicação dos 4 cenários (A/B/C/D) é clara o suficiente para o cedente escolher no cadastro sem assistência?**
> 

> O cedente escolhe entre 4 cenários de retorno (A = assumir saldo devedor; B = 100% do pago; C = +30%; D = +50%). A dúvida é: a apresentação na plataforma é compreensível sem suporte humano, ou o cedente precisa de atendimento para decidir?
> 

**Como deve ser respondida:** A resposta deve vir de testes de usabilidade com pelo menos 5 cedentes reais ou simulados. O critério de avaliação é: quantos conseguiram escolher o cenário correto sem assistência? Quais cenários geraram mais dúvidas? A resposta deve indicar se o fluxo de cadastro precisa de ajustes (tooltips, comparativo visual, simulador) e quais cenários exigem mais explicação.

---

> **V2 — O comprador aceita pagar comissão zero quando Δ ≤ 0? O piso de R$ 5k é visto como justo ou como barreira?**
> 

> Quando não há desconto real (Preço do Repasse ≥ Tabela Atual), a comissão variável do comprador é R$ 0. Mas o piso de R$ 5k pode ser aplicado quando previsto no Termo Comercial. A dúvida é se o comprador entende e aceita essa lógica — ou se a vê como cobrança injustificada.
> 

**Como deve ser respondida:** Coletar reação de pelo menos 3 compradores potenciais durante a apresentação do modelo. A resposta deve indicar se o piso de R$ 5k é mantido, ajustado ou removido — e com qual justificativa comercial. Se houver resistência ao piso, avaliar se ele deve ser opcional por padrão.

---

> **V3 — Como garantir auditabilidade das evidências (tabela, preço, data do lance) dentro do fluxo da plataforma?**
> 

> O modelo depende de evidências documentais para calcular Δ e disparar comissões. Se o dossiê não for preenchido corretamente ou a plataforma não forçar o upload antes do fechamento, o cálculo vira disputa. A dúvida é: qual o mecanismo técnico e operacional que impede um fechamento sem evidência válida?
> 

**Como deve ser respondida:** A resposta deve descrever o fluxo técnico mínimo: quais campos são obrigatórios antes do status "Fechamento" ser liberado no Admin, quem valida a evidência (Analista de Formalização), e o que acontece se a evidência for contestada depois. Deve incluir uma decisão sobre bloqueio automático vs. validação manual.

---

> **V4 — Qual é a defasagem real entre o fechamento e o recebimento da comissão? O prazo de 5 dias úteis é operacionalmente viável para ambas as partes?**
> 

> O modelo assume cobrança no dia do fechamento com prazo de 5 dias úteis. Mas o cedente pode não ter liquidez imediata — ele recebe do comprador e só então paga a comissão. Se a sequência de pagamentos não estiver clara, o prazo de 5 dias pode ser sistematicamente descumprido.
> 

**Como deve ser respondida:** A resposta deve mapear a sequência real de pagamentos: (1) quem paga quem e em qual ordem, (2) qual o prazo médio real observado nos primeiros casos, (3) se o prazo de 5 dias precisa ser ajustado para 10 ou 15 dias úteis. Deve resultar em uma regra revisada de prazo de cobrança, se necessário.

---

> **V5 — A taxa de reversão nos primeiros casos ficará dentro da estimativa de 5–10%? Quais são os gatilhos reais de desistência?**
> 

> O modelo provisiona reversão de 5–10% sobre a receita. Mas os primeiros casos podem ter padrões diferentes — cedentes mais inseguros, compradores que não conseguem financiamento, incorporadoras que demoram a responder. A dúvida é se a estimativa é calibrada ou subestimada.
> 

**Como deve ser respondida:** A resposta deve ser construída após os primeiros 10 fechamentos. Deve registrar o número de reversões, o motivo de cada uma (desistência do cedente, do comprador, bloqueio externo) e se o padrão observado está acima ou abaixo dos 5–10%. Se acima, indicar qual ajuste de processo ou qualificação pré-fechamento é necessário.

---

> **V6 — Em quantos cenários a incorporadora bloqueia ou atrasa a anuência à cessão? Qual o impacto real na taxa de conversão?**
> 

> A anuência da incorporadora é uma etapa crítica em muitos casos. Se a incorporadora não responde, cobra taxa abusiva ou nega a cessão, o caso não fecha — e o funil trava. A dúvida é: esse bloqueio é exceção ou regra? E o que fazer quando acontece?
> 

**Como deve ser respondida:** A resposta deve vir de dados reais dos primeiros 20 casos captados. Deve indicar: (1) % de casos onde a incorporadora foi acionada, (2) % de bloqueios ou atrasos, (3) tempo médio de resposta. Com base nisso, definir se é necessário um protocolo específico de relação com incorporadoras (canal direto, termo de parceria, etc.) antes de escalar o volume.

---

> **V7 — Quais funcionalidades são exclusivas de cada perfil (Admin, Cedente, Cessionário)? Qual o nível mínimo de dados que cada perfil precisa ver para operar com autonomia?**
> 

> A plataforma opera com 3 dashboards independentes. Mas o escopo exato de cada perfil ainda não foi mapeado em nível funcional. Se o cedente não tiver visibilidade suficiente do status do seu caso, vai ligar ou enviar mensagem — o que aumenta custo operacional. Se tiver visibilidade demais, pode expor dados do comprador.
> 

**Como deve ser respondida:** A resposta deve ser um mapeamento funcional por perfil: lista de telas, campos visíveis, ações disponíveis e dados bloqueados. Deve ser validada com pelo menos 2 usuários de cada perfil (cedente e comprador reais, ou simulados em teste de usabilidade) antes de entrar em desenvolvimento.

---

### 17.2 Jurídico e Operacional

---

> **V8 — Qual CNAE será utilizado para a operação do RepasseSeguro? Ele cobre a atividade de intermediação de cessão imobiliária?**
> 

> A escolha do CNAE define o regime tributário, a alíquota de ISS aplicável e a legalidade da atividade perante a Receita Federal e órgãos reguladores (CRECI, CVM, BACEN). Um CNAE inadequado pode gerar autuação ou obrigação de licença profissional que inviabiliza o modelo.
> 

**Como deve ser respondida:** A resposta deve vir de parecer jurídico formal, com indicação do CNAE principal e, se necessário, secundário. Deve também esclarecer se a atividade exige registro no CRECI (corretagem) ou se pode ser enquadrada como plataforma de tecnologia/intermediação digital. Prazo para resposta: antes da abertura do CNPJ.

---

> **V9 — Como o RepasseSeguro opera legalmente quando a incorporadora não responde ou nega a anuência? Existe protocolo alternativo que preserve o fechamento?**
> 

> Alguns contratos de compra e venda exigem anuência expressa da incorporadora para a cessão ser válida. Se a incorporadora nega ou ignora o pedido, o repasse pode ser considerado inválido juridicamente — expondo cedente, comprador e o RepasseSeguro a riscos.
> 

**Como deve ser respondida:** A resposta deve vir de análise jurídica dos tipos de contrato mais comuns no mercado-alvo. Deve definir: (1) em quais contratos a anuência é obrigatória vs. opcional, (2) qual o protocolo do RepasseSeguro quando a anuência é negada (encerrar o caso, negociar com a incorporadora, acionar via judicial?), (3) se isso deve ser informado ao cedente antes da captação do caso.

---

> **V10 — Em caso de contestação do dossiê após o fechamento, quem responde juridicamente — o Analista de Formalização ou o RepasseSeguro como empresa?**
> 

> Se a evidência da tabela for contestada pelas partes (ex: "essa não era a tabela vigente na data do lance"), a responsabilidade pela validação pode recair sobre a pessoa que assinou o dossiê ou sobre a empresa. Sem clareza, isso vira passivo jurídico.
> 

**Como deve ser respondida:** A resposta deve vir de cláusula específica redigida pelo jurídico no Termo Comercial e no contrato de trabalho/prestação do Analista. Deve definir: (1) a empresa assume responsabilidade pelo processo de validação, (2) o analista é responsável pela execução dentro do protocolo, (3) o Termo Comercial limita a responsabilidade do RepasseSeguro a casos onde o protocolo foi seguido.

---

### 17.3 Precificação e Captação

---

> **V11 — O percentual fixo de 20% é competitivo e sustentável para todos os tamanhos de caso?**
> 

> A comissão do cedente (Cenários B/C/D) é fixa em 20% sobre (Valor Recuperado − Valor Distrato Referência). A dúvida é: esse percentual único funciona bem tanto para casos pequenos (R$ 50k recuperados) quanto para casos grandes (R$ 1M+ recuperados)? Em casos grandes, 20% pode parecer excessivo; em casos pequenos, pode não cobrir o custo operacional.
> 

**Como deve ser respondida:** A resposta deve vir de simulação com pelo menos 5 faixas de valor recuperado (R$ 50k, R$ 150k, R$ 300k, R$ 500k, R$ 1M) e feedback de pelo menos 3 cedentes por faixa. Deve indicar se o percentual de 20% é mantido como único, ou se é necessário criar uma tabela regressiva (ex: 20% até R$ 300k, 15% de R$ 300k a R$ 700k, 10% acima de R$ 700k). O resultado deve ser registrado como política interna de precificação.

---

> **V12 — Qual é o canal principal de captação nos primeiros 90 dias? O modelo assume 25–60 casos captados/mês desde o M1, mas esse canal ainda não foi validado operacionalmente.**
> 

> O breakeven depende de volume de casos. Mas o modelo não define de onde vêm os leads nos primeiros 3 meses — se é tráfego orgânico, indicação de advogados/despachantes, parceria com incorporadoras ou outbound direto. Sem canal definido, a premissa de 25–60 captados/mês é uma hipótese sem fundação.
> 

**Como deve ser respondida:** A resposta deve resultar de um experimento real de captação antes do lançamento: testar ao menos 2 canais (ex: grupo de WhatsApp de cedentes + parceria com 1 despachante imobiliário) e medir CPL (custo por lead) e taxa de qualificação. Com esses dados, definir o canal primário de M1–M3, o canal secundário e a meta revisada de captação.

---

## 18. Documentos Relacionados

| **Documento** | **Escopo** | **Status** |
| --- | --- | --- |
| Análise Geral | Contexto do problema, mercado, atores e hipóteses | v2.2 |
| Pesquisa de Mercado / Benchmark | Benchmark competitivo com scoring, Porter adaptado, mapa de posicionamento | v2.0 |
| Estudo de Viabilidade | Viabilidade de mercado, operacional, jurídica e econômica | v1.1 |
| Veredito Final | GO/NO GO com condições mínimas, roadmap e gates | v1.2 |
| Fundraising | Tese de investimento, valuation e termos | v1.1 |
| Custos Operacionais | Detalhamento de setup, burn, premissas e cenários | v1.9 |