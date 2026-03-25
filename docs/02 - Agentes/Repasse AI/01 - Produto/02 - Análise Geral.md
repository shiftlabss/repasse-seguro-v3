# 02 - Análise Geral

Fase: 1 — Discovery
Área: Company Discovery

# 1. Análise Geral — RepasseSeguro

## 1.1 Metadados do Documento

| Campo | Valor |
| --- | --- |
| Destinatário | Stakeholders de produto, operação e liderança da Shift Labs |
| Escopo | Análise geral do problema de repasse/distrato no mercado imobiliário brasileiro, mapeamento de atores, concorrência, hipóteses estratégicas, limites do MVP e alinhamento com o modelo de monetização do RepasseSeguro |
| Versão | v2.4 |
| Responsável | Fernando Calado |
| Data da versão | 24/02/2026 22:00 (America/Fortaleza) |

<aside>
📌

**TL;DR — visão rápida da análise geral**

- Volume relevante de imóveis na planta (~45% das 400,5 mil unidades vendidas em 2024, segundo CBIC/CII) e alta taxa de distratos (~13–15% das vendas segundo Abrainc/Fipe, chegando a 15–30% das entregas segundo Fitch Ratings) criam um estoque recorrente de contratos problemáticos — potencialmente **>50 mil unidades anuais** distratáveis.
- Multas contratuais de 20–25% (chegando a 50% com patrimônio de afetação) geram forte desequilíbrio para o comprador, amparado pela Lei 13.786/2018 e pela jurisprudência do STJ.
- O mercado atual opera majoritariamente via soluções informais e pouco seguras (contratos de gaveta, anúncios dispersos em OLX/Marketplace, corretores sem processo).
- Concorrentes diretos (Dinheiro na Planta, Desfinancia) validam a demanda, mas nenhum atua como **infraestrutura neutra, jurídica-first e focada em formalização** — há espaço claro para o RepasseSeguro.
- O modelo de monetização é baseado em **2 comissões de sucesso** (cedente + comprador), cobradas apenas no Fechamento: o cedente escolhe entre **4 cenários de retorno** (A/B/C/D), com comissão de **20%** sobre (Valor Recuperado − Valor Distrato Referência) nos cenários B/C/D; comprador paga **20%** sobre Δ (Tabela Atual − Tabela do Contrato). Ticket combinado médio: R$ 50k. Sem assinatura, mensalidade ou capital próprio.
</aside>

---

## 2. Problema que a plataforma resolve e contexto no mercado imobiliário brasileiro

### 2.1 Contexto de mercado

O **RepasseSeguro** surge para resolver um problema recorrente no mercado imobiliário brasileiro: a dificuldade de **revender ou transferir contratos de compra de imóveis**, especialmente unidades **adquiridas na planta ou financiadas**, de forma **segura e justa**.

Esse problema ocorre em um contexto onde uma parcela significativa das vendas é de imóveis **na planta** (cerca de 45% das 400,5 mil unidades vendidas em 2024, segundo a CBIC/CII) e muitos compradores enfrentam mudanças de condição ou arrependimento antes da entrega das chaves. Quando esses compradores decidem desistir do negócio — o chamado **distrato imobiliário** — eles se deparam com **penalidades contratuais severas** e poucas alternativas viáveis.

<aside>
🎯

**Por que esse problema importa agora:**

- A combinação de juros elevados, contratos firmados em condições favoráveis (2020–2021) e entrega de chaves em cenário macroeconômico diferente criou um **estoque de contratos inviáveis** sem precedentes recentes.
- A Lei 13.786/2018, ao consolidar o distrato com regras objetivas, tornou evidente o **tamanho do custo de saída** — e, ao mesmo tempo, validou a existência de um **mercado secundário organizado** ao prever a figura do comprador substituto.
- Dados atualizados reforçam o timing: o mercado residencial brasileiro vale **US$ 106,97 bilhões** (2025), com projeção de **US$ 138,70 bilhões** em 2030 (CAGR 5,99%, Mordor Intelligence). A intenção de compra atingiu **50% no Q4/2025** (+5 p.p. vs Q4/2024, Brain), e a valorização de preços (FipeZap) foi de **+6,22%** em 2025, acima da inflação (5,17%).
</aside>

### 2.2 Desequilíbrio contratual e perdas financeiras

Atualmente, os **distratos** impõem perdas financeiras elevadas ao comprador original. A prática comum das incorporadoras é reter até **50% de tudo que foi pago** caso o cliente desista da compra.

#### 2.2.1 Base legal

A **Lei 13.786/2018** (Lei do Distrato) define as regras:

- **Sem patrimônio de afetação:** pena convencional de até **25% do valor pago**, com devolução em até 180 dias.
- **Com patrimônio de afetação:** retenção pode chegar a **50% do valor pago**, com devolução em até 30 dias após o habite-se.

#### 2.2.2 Jurisprudência

A jurisprudência recente do Superior Tribunal de Justiça (STJ) tem limitado essas multas a **20%–25% do valor pago**, considerando cláusulas de 50% como abusivas. Ou seja, há um **desequilíbrio contratual** histórico: cláusulas punitivas contra o comprador (multas altas, devoluções demoradas) contrastam com proteções bem menores ao consumidor.

Esse cenário gerou insegurança jurídica e inúmeros processos — por exemplo, um escritório especializado relata **4,5 mil ações** contra construtoras, grande parte sobre distratos.

#### 2.2.3 Ponto-chave para o RepasseSeguro

A própria Lei 13.786/2018 prevê que **não incide a cláusula penal** se o adquirente encontra um **comprador substituto** que o sub-rogue nas obrigações, desde que haja anuência do incorporador e aprovação do cadastro e capacidade financeira do substituto. Na prática, isso **valida legalmente** a existência de uma plataforma como o RepasseSeguro.

<aside>
💡

**Insight estratégico:** a base legal não apenas permite — ela *favorece* — a substituição de comprador como alternativa ao distrato. O RepasseSeguro transforma essa previsão legal em **infraestrutura operacional**.

</aside>

### 2.3 Impacto macroeconômico recente

O **contexto econômico recente agravou o problema** de forma estrutural:

- Muitos contratos assinados em **2020–2021**, quando os juros habitacionais estavam em torno de **7% a.a.**, tornaram-se inviáveis com a alta da Selic para **~10% a.a.** em 2023.
- Compradores que planejavam financiar na entrega das chaves descobriram que as parcelas ficaram muito mais caras.
- Isso gerou uma **onda de distratos** não por arrependimento, mas por **impossibilidade financeira** — perfil completamente diferente do distrato de conveniência.

#### 2.3.1 Impacto prático

| Indicador | Período favorável (2020–2021) | Período de pressão (2023–2024) | Variação |
| --- | --- | --- | --- |
| Taxa Selic | ~2% a.a. | ~13,75% a.a. (pico 2023) / >13% a.a. (fev/2026) | +587% |
| Juros habitacionais médios | ~7% a.a. | ~10% a.a. | +43% |
| Taxa de distratos (Abrainc/Fipe) | ~8–10% | ~13–15% | +50–60% |
| Parcela média em financiamento de R$ 300k (30 anos) | ~R$ 2.100/mês | ~R$ 2.800/mês | +R$ 700/mês |

### 2.4 Dados de distratos, volume e falta de liquidez

De acordo com a Associação Brasileira de Incorporadoras (Abrainc/Fipe), a taxa de distratos subiu para cerca de **15% das vendas no fim de 2023** (chegando a 13,7% no segmento de médio/alto padrão).

#### 2.4.1 Volume estimado

| Métrica | Dado | Fonte |
| --- | --- | --- |
| Unidades vendidas (2024) | **400.500** (+20,9% vs 2023) | CBIC/CII |
| % na planta | ~45% | CBIC/CII |
| Unidades na planta vendidas | ~180.000 | Cálculo (45% × 400,5k) |
| Taxa de distratos (% das vendas, pico) | ~13–15% | Abrainc/Fipe |
| Taxa de distratos (% das entregas, média histórica) | **15–30%** | Fitch Ratings / Senior Index |
| **Unidades potencialmente distratáveis/ano** | **~24.000–28.000** | Cálculo conservador (só na planta) |
| Incluindo financiados em dificuldade | **>50.000 unidades/ano** | Estimativa |

#### 2.4.2 Variação regional

Em 2024, vários estados registraram aumento de dois dígitos nos distratos:

- **Minas Gerais:** +23% em comparação a 2023
- **São Paulo:** +21% em comparação a 2023

Esses números evidenciam a **falta de liquidez** para quem deseja repassar seu contrato: milhares de famílias e investidores ficaram sem opção a não ser arcar com prejuízos ou enfrentar batalhas judiciais.

### 2.5 Ineficiências de mercado

Além das perdas financeiras, o modelo atual gera **ineficiências de mercado** que afetam todas as partes da cadeia:

#### 2.5.1 Para a incorporadora

Quando um comprador desiste, a unidade volta ao estoque da construtora, que frequentemente demora para revender — gerando **vacância e custos de carregamento**. Muitas incorporadoras **não provisionam adequadamente** devoluções, acumulando riscos no balanço.

#### 2.5.2 Para o mercado

Existiriam interessados em assumir esses contratos (muitas vezes com valorização já incorporada), mas **faltam canais seguros** para conectar essas partes. Hoje esse **gap** é preenchido de forma precária, via anúncios dispersos e acordos informais, expondo todos a riscos de fraude, calote e disputas futuras.

#### 2.5.3 Para o ecossistema

Corretores tentam intermediar informalmente, sem padrão. Bancos operam no escuro. Advogados lidam com volume crescente de disputas. O custo transacional total (incluindo judicial) é muito maior do que seria necessário se houvesse uma **infraestrutura de formalização**.

<aside>
🎯

**Síntese do problema:** o RepasseSeguro endereça a **falta de um mecanismo estruturado, transparente e equitativo** para lidar com distratos e repasses (cessões de contrato) de imóveis — problema que afeta um volume de **>50 mil contratos/ano**, agravado por desequilíbrios contratuais, volatilidade econômica e ausência de canais formalizados.

</aside>

---

## 3. Pessoas afetadas atualmente por esse problema

### 3.1 Visão geral dos atores

Diversos **atores do mercado imobiliário** são impactados pela ausência de soluções adequadas para distratos e repasses. Os principais públicos afetados são mapeados abaixo com suas dores, motivações e o que buscam em uma solução.

### 3.2 Compradores finais (mutuários originais)

#### 3.2.1 Perfil

Famílias ou indivíduos que adquiriram um imóvel na planta ou financiaram um imóvel, mas por algum motivo precisam desistir — mudança de cidade, perda de renda, separação, aumento inesperado de juros, ou simplesmente arrependimento.

#### 3.2.2 Dores específicas

- Enfrentam perdas significativas: podem **perder quase metade do que pagaram** em distrato formal.
- Muitos usaram economias de anos como entrada e correm o risco de não recuperar quase nada.
- Ficam **"presos"** ao contrato por falta de quem assuma a dívida.
- Devolução da incorporadora pode demorar até **180 dias** ou mais.

#### 3.2.3 Comportamento atual

- Tentam negociar diretamente com a construtora (posição de fraqueza).
- Recorrem a anúncios informais ou corretores de confiança.
- Sofrem **ansiedade e insegurança financeira**, pois a alternativa formal (distrato) é ruinosa e a alternativa informal é arriscada.

<aside>
💡

**Insight:** esse público tem a **dor mais aguda** e a **maior disposição para pagar** por uma solução que reduza a perda. É o *core target* do lado da oferta do RepasseSeguro.

</aside>

### 3.3 Vendedores de contratos (cedentes)

#### 3.3.1 Perfil

Inclui tanto os compradores finais acima, no papel de vendedores do contrato, quanto **investidores pessoa física** que compraram unidades visando revender antes da entrega das chaves.

#### 3.3.2 Dores específicas

- Desejam **repassar seus contratos** a terceiros, muitas vezes para evitar prejuízo ou realizar lucro.
- Atuam em um **mercado informal e desorganizado**, com dificuldade de encontrar compradores confiáveis e de precificar o contrato adequadamente.
- Ficam sujeitos a aceitar deságios elevados com a construtora ou negociar no escuro com desconhecidos.

#### 3.3.3 O que buscam

- Canal com compradores qualificados e pré-triados.
- Transparência sobre valor justo e condições do repasse.
- Processo que dê **segurança jurídica** e rastreabilidade.

### 3.4 Compradores interessados em repasses (cessionários)

#### 3.4.1 Perfil

Investidores e compradores oportunistas que buscam **oportunidades abaixo do valor de mercado** — compram contratos com desconto significativo em relação à tabela vigente da incorporadora.

#### 3.4.2 Dores específicas

- Garimpam anúncios dispersos (OLX, Facebook Marketplace, redes de corretores) geralmente **sem transparência contratual**.
- Não têm visibilidade sobre saldo devedor real, reajustes acumulados, cronograma da obra, nem se a cessão é permitida pelo contrato.
- Enfrentam **desconfiança mútua** e **baixa segurança jurídica** em acordos de gaveta.

#### 3.4.3 O que buscam

- Dados padronizados e comparáveis entre oportunidades.
- Verificação mínima da autenticidade do contrato e dos documentos.
- Trilha de formalização que reduza risco de golpe ou disputa futura.

### 3.5 Incorporadoras (construtoras)

#### 3.5.1 Perfil

Empresas que sofrem diretamente com a instabilidade de distratos — afetando fluxo de caixa, provisões contábeis e planejamento de novos lançamentos.

#### 3.5.2 Dores específicas

- Alta taxa de distratos desestabiliza fluxo de caixa e planejamento de empreendimentos.
- Muitas **não provisionam adequadamente** devoluções, acumulando riscos no balanço.
- Precisam **revender unidades canceladas** em cenários possivelmente piores de mercado, com desconto.
- Enfrentam desgaste de imagem e custo administrativo com disputas judiciais crescentes.

#### 3.5.3 Oportunidade com o RepasseSeguro

- Uma plataforma que organize substituições de comprador **reduz o volume de distratos** sem custo para a incorporadora.
- A anuência exigida por lei mantém a construtora no controle, mas sem o custo operacional de organizar o repasse.

### 3.6 Corretores e parceiros do ecossistema

#### 3.6.1 Perfil

**Corretores imobiliários** tentam ajudar clientes a revender contratos informalmente, mas sem sistema padronizado. **Advogados imobiliários** lidam com volume crescente de disputas de distrato.

#### 3.6.2 Dores específicas

- Dificuldade para cobrar comissão e garantir pagamento em operações informais.
- Sem padrão de documentação, cada repasse é um processo manual e diferente.
- Risco reputacional ao intermediar operações que podem dar errado.

#### 3.6.3 Oportunidade com o RepasseSeguro

- Corretores: canal de distribuição natural — já têm o cliente na mão e conhecem quem quer sair.
- Advogados: parceiros de formalização com demanda existente.
- Plataformas emergentes (como **Dinheiro na Planta**) já miram esse público, conectando corretores a clientes que querem repassar contratos.
- Outros parceiros incluem **bancos financiadores** e **investidores institucionais** que se beneficiariam de um processo seguro, mas hoje operam com alta incerteza.

### 3.7 Mapa consolidado dos atores

| Ator | Dor principal | O que busca | Disposição para pagar |
| --- | --- | --- | --- |
| Comprador original / Cedente | Perda de 20–50% no distrato | Saída com menor prejuízo | Alta (evita perda maior) |
| Comprador de repasse / Cessionário | Risco de golpe e falta de dados | Oportunidade verificada com desconto | Média (proporcional ao desconto) |
| Incorporadora | Fluxo de caixa e reputação | Menos distratos e menos litígios | Baixa no MVP (canal futuro) |
| Corretor / Advogado | Informalidade e risco | Processo padronizado e comissão garantida | Indireta (via comissão/parceria) |

<aside>
🎯

O RepasseSeguro tem oportunidade de se posicionar como **infraestrutura neutra** que organiza esses atores, reduzindo desconfiança e dando previsibilidade às transações — capturando valor das duas pontas (cedente e comprador) apenas quando há resultado econômico real e formalizado.

</aside>

---

## 4. Soluções atuais para o problema e concorrentes no Brasil

### 4.1 Distrato convencional com a incorporadora

Diante do problema de distratos e necessidade de repasse, os envolvidos têm recorrido a **soluções improvisadas e alternativas informais**, bem como a alguns serviços emergentes.

#### 4.1.1 Como funciona

É a solução formal prevista em contrato quando o comprador desiste. Envolve a rescisão do contrato de compra e venda diretamente com a construtora. Como mencionado, essa opção acarreta multa pesada — tipicamente **25% a 50%** do valor já pago fica retido pela vendedora.

#### 4.1.2 Problemas

- Devolução do restante ao cliente pode demorar **até 180 dias** após a rescisão ou mesmo após a conclusão da obra.
- Apesar de formal, esse caminho é **oneroso para o comprador** e também indesejado pela construtora (que volta a ter uma unidade em estoque e precisa revender em cenário possivelmente pior).

### 4.2 Repasse informal ("contrato de gaveta")

#### 4.2.1 Como funciona

Acordos privados para transferir imóvel financiado ou na planta a outra pessoa, **sem anuência formal** do banco ou construtora. O novo adquirente paga parcelas ao vendedor original, mas a titularidade oficial não muda.

#### 4.2.2 Problemas

- Prática **sem respaldo legal claro** e altamente arriscada para ambas as partes.
- Cria um **mercado paralelo** vulnerável a fraudes, calotes e disputas futuras.
- Se o vendedor original para de pagar, o imóvel é retomado — e o comprador de gaveta perde tudo.
- Se o comprador de gaveta não paga, o vendedor original fica com a dívida e sem recurso fácil.

### 4.3 Portais generalistas e redes sociais

#### 4.3.1 Como funciona

Uso de **OLX, Zap Imóveis, Facebook Marketplace** e redes informais (grupos de WhatsApp, Telegram, Instagram) para anunciar repasses.

#### 4.3.2 Problemas

- Plataformas funcionam apenas como vitrine, **sem suporte processual ou verificação** de documentos, saldo devedor, ou legitimidade do anúncio.
- Usuários precisam negociar tudo por conta própria (preço, saldo devedor, transferência, anuência).
- A falta de transparência reforça a desconfiança e os riscos de golpe.
- Sem padronização, cada anúncio tem informações diferentes, impossibilitando comparação.

### 4.4 Cessão formal via banco ou construtora

#### 4.4.1 Como funciona

Quando prevista em contrato, é possível fazer **cessão oficial de direitos**. Envolve análise de crédito do novo comprador, avaliação do imóvel e formalização de novo contrato ou termo de cessão.

#### 4.4.2 Problemas

- É a forma **mais segura**, porém **burocrática e lenta**, com várias etapas cartorárias e tributárias (ITBI, registro, averbação).
- Depende da anuência de bancos e incorporadoras, que podem resistir em cenários de valorização do imóvel (preferem revender a tabela cheia).
- Não existe um processo padronizado — cada caso é tratado ad hoc.

### 4.5 Soluções concorrentes especializadas

O benchmark de mercado revela que o espaço de repasses e cessões já é explorado por nichos específicos, mas **nenhum atua como infraestrutura neutra, jurídica-first e focada em repasse imobiliário formalizado**.

#### 4.5.1 Dinheiro na Planta

| Dimensão | Detalhe |
| --- | --- |
| **Segmento** | Imóveis na planta (contratos de compra e venda) |
| **Modelo** | Marketplace que conecta titulares de contratos a investidores pré-cadastrados |
| **Monetização** | Deságio/spread entre compra do cedente e revenda ao investidor (assume posição no deal — **risco de balanço**) |
| **Narrativa** | "Liquidez e deságio menor que o distrato" |
| **Ponto forte** | Rede de investidores pré-cadastrados (liquidez real), foco exclusivo em nicho imobiliário, cobre antes e depois das chaves |
| **Ponto fraco** | Risco de balanço (assume posição financeira), dependência de capital para operar, precificação opaca para o cedente, sem trilha de auditoria (score 1/5 no benchmark v2.0), concentração em médio/alto padrão |
| **Risco de convergência** | 🔴 **ALTO** — maior sobreposição com RepasseSeguro. Defesa: ser infra neutra (não tomar posição no deal) + governança superior |

#### 4.5.2 Desfinancia

| Dimensão | Detalhe |
| --- | --- |
| **Segmento** | Financiamentos em andamento (imóveis e veículos) |
| **Modelo** | Plataforma de repasse de financiamentos, evitando leilões e retomadas |
| **Monetização** | Fee por transação (valor não divulgado), modelo de receita em consolidação |
| **Narrativa** | "Venda sem prejuízo, evite o leilão" |
| **Ponto forte** | Pioneira em repasses de financiamento — valida a existência da demanda. Abrange veículos e imóveis, narrativa emocional forte |
| **Ponto fraco** | Escopo amplo demais dilui especialização, sem trilha de formalização documental (score 1/5 no benchmark v2.0), base limitada com baixa liquidez regional, dependência total do banco credor como gatekeeper |
| **Risco de convergência** | 🟡 **MÉDIO** — se verticalizar para imóveis na planta e adicionar governança. Escopo amplo e fragilidade de processo tornam pouco provável no curto prazo |

#### 4.5.3 Players adjacentes relevantes

- **Imobiliárias digitais** (QuintoAndar, Loft, etc.) e **fundos de investimento** (como Jive e Strategi) compõem o ecossistema mais amplo, atuando em transações e ativos estressados — mas nenhum foca em repasse de contrato como core.
- **Players automotivos** (Auto Avaliar, Motorez, B2B Auto, Passu) mostram modelos de marketplace e SaaS white-label para repasses que servem de referência operacional, mesmo sendo de outro setor.
- **Consig360** (crédito consignado) é a melhor referência conceitual de infraestrutura neutra — fez no consignado exatamente o que o RepasseSeguro quer fazer no imobiliário (hub multi-banco, compliance automatizado, módulo de cessão).
- **Levi Crédito e Credituz** atuam no repasse bancário de incorporadoras (financiamento ao banco na entrega), não na cessão de contrato entre pessoas — sobreposição baixa.

<aside>
💡

**Insight do benchmark v2.0: a competição real é com a informalidade**

O maior "concorrente" do RepasseSeguro não é uma empresa — é a **informalidade** (contrato de gaveta, WhatsApp, corretor por fora). O mercado endereçável está **90%+ desorganizado**. A matriz de scoring (7 critérios, escala 1–5) mostra que nenhum player com alta aderência ao problema imobiliário (≥4/5) tem score alto em governança ou trilha de auditoria. O quadrante superior direito do mapa de posicionamento (alta formalização + receita sobre resultado real) está **vazio no imobiliário**.

Análise completa com scoring, Porter adaptado e mapa de posicionamento: ver Pesquisa de Mercado e Benchmark v2.0.

</aside>

### 4.6 Tabela comparativa: posicionamento do RepasseSeguro vs. alternativas

| Dimensão | Distrato formal | Contrato de gaveta | Portais generalistas | Dinheiro na Planta | Desfinancia | **RepasseSeguro** |
| --- | --- | --- | --- | --- | --- | --- |
| **Segurança jurídica** | Alta (mas caro) | Muito baixa | Nenhuma | Média | Média | **Alta (trilha + curadoria)** |
| **Custo para o vendedor** | 20–50% do pago | Risco de perda total | Variável | Fee/spread | Fee/spread | **Comissão sobre valor recuperado** |
| **Padronização de dados** | Baixa | Nenhuma | Nenhuma | Parcial | Parcial | **Alta (checklist + verificação)** |
| **Governança / auditoria** | Interna da construtora | Nenhuma | Nenhuma | Baixa | Baixa | **Alta (trilha de evidências)** |
| **Modelo de receita** | N/A | N/A | Anúncio | Fee/spread | Fee/spread | **Comissão dupla no fechamento** |

<aside>
💡

**O que o benchmark valida para o RepasseSeguro:**

- Existe precedente forte de mercado para **processo + curadoria + padronização** (não só vitrine).
- Os modelos de monetização mais aderentes ao MVP tendem a ser **fee por caso / comissão no fechamento**, evitando promessas de liquidez.
- **Diferencial defensável:** infra neutra + jurídica-first + trilha de formalização + regras claras de cálculo — nenhum concorrente ocupa essa posição hoje.
</aside>

---

## 5. Principais hipóteses por trás da ideia RepasseSeguro

### 5.1 Hipóteses de mercado e demanda

Para que a plataforma **RepasseSeguro** seja bem-sucedida, algumas premissas fundamentais precisam se confirmar verdadeiras no contexto brasileiro. São **hipóteses estratégicas** sobre mercado, comportamento e viabilidade.

#### 5.1.1 Existência de demanda real e volumosa

- Supõe-se um universo relevante de pessoas buscando repassar contratos e outras dispostas a comprá-los.
- Dados de distratos (~13–15% das vendas, sugerindo **>50 mil unidades anuais** potencialmente distratáveis) indicam um estoque de **"vendedores em apuros"**.
- Sinais de interesse de investidores e compradores oportunistas reforçam a existência da **demanda latente**.

#### 5.1.2 TAM estimado

| Métrica | Estimativa | Base |
| --- | --- | --- |
| Casos potenciais de repasse/ano | 50.000–70.000 | Volume de distratos + financiamentos em dificuldade |
| Ticket médio de comissão total (cedente + comprador) | R$ 50k–65k por caso | Modelo de Negócios v3.4 |
| TAM de comissões (receita potencial do mercado) | R$ 2,5B–4,5B/ano | Cálculo sobre volume × ticket |
| Participação alvo (Ano 2) | 0,5–1% do TAM | Cenário conservador |

### 5.2 Hipóteses de comportamento das partes

#### 5.2.1 Disposição para negociar

- **Vendedores** aceitam perder menos do que no distrato tradicional — a plataforma oferece uma saída com prejuízo menor.
- **Compradores** buscam desconto real em relação ao imóvel novo — aceitam pagar comissão porque o ganho é proporcional.
- Hipótese de que, em ambiente seguro e com dados padronizados, **ambos os lados preferem formalidade** a informalidade.

#### 5.2.2 Adesão ao processo

- O vendedor está disposto a fornecer documentação mínima (contrato, boletos, identidade) em troca de visibilidade e credibilidade.
- O comprador está disposto a passar por triagem (capacidade financeira, documentos) em troca de acesso a oportunidades verificadas.

### 5.3 Hipóteses jurídicas e institucionais

#### 5.3.1 Maturidade jurídica

- **Maturidade jurídica e operacional** suficiente para suportar repasses online formalizados.
- Suposição de que bancos e incorporadoras estão dispostos a **aceitar transferências** desde que informados e com documentação adequada — a Lei 13.786/2018 já prevê essa possibilidade.

#### 5.3.2 Apoio do Judiciário

- Apoio indireto da postura do Judiciário em **proteger o consumidor** contra retenções abusivas — limitando multas de distrato a 20–25%.
- Isso pressiona incorporadoras a serem mais flexíveis com alternativas ao distrato, como a cessão.

#### 5.3.3 Hipótese de compliance

- É possível operar como **plataforma informacional e de conexão** sem configurar intermediação imobiliária formal no MVP, desde que os limites de atuação estejam claramente definidos nos termos de uso.

<aside>
🔴

**Risco associado:** se o RepasseSeguro encostar em **intermediação imobiliária de fato**, será necessário operar com responsabilidade jurídica clara, registros e trilhas de auditoria, e políticas mínimas de compliance (incluindo prevenção à lavagem de dinheiro — Lei 9.613/1998 e Resolução COFECI 1.336/2014).

</aside>

### 5.4 Hipóteses de timing de mercado

#### 5.4.1 Momento favorável

- Momento de **pós-alta de juros** com legado de contratos problemáticos — o estoque de "vendedores em apuros" já existe.
- Expectativa de melhora gradual das condições econômicas, atraindo novos compradores dispostos a assumir contratos com desconto.
- Setor imobiliário em **transformação digital**, com usuários mais maduros para soluções online e proptechs ganhando espaço.

#### 5.4.2 Janela de oportunidade

- O problema é mais agudo **agora** (contratos de 2020–2021 com entrega de chaves em 2023–2025), mas tende a se tornar **estrutural** — sempre haverá distratos e repasses.
- A janela competitiva está aberta: nenhum player consolidado ocupa a posição de **infraestrutura neutra de formalização**.

<aside>
✅

**Se essas hipóteses se confirmarem** — especialmente em demanda volumosa, adesão institucional mínima e timing favorável — o RepasseSeguro tem base concreta para tração, com um TAM de comissões estimado entre **R$ 2,5B e R$ 4,5B/ano**.

</aside>

---

## 6. Limites claros do escopo inicial (o que o MVP não deve tentar resolver agora)

### 6.1 Escopo operacional do MVP

Para assegurar foco e viabilidade, o **MVP (Produto Mínimo Viável)** do RepasseSeguro terá um escopo bem delimitado. Há várias complexidades inerentes ao mercado imobiliário que **não serão abordadas nas primeiras versões da plataforma**.

<aside>
🎯

**Definição do MVP:** curadoria fechada + fluxo assistido + trilha de auditoria + termos + playbook comercial. O RepasseSeguro é uma **operação jurídica e comercial com tecnologia de suporte**, não um app que resolve tudo sozinho.

</aside>

#### 6.1.1 O que o MVP faz

| Capacidade | Descrição |
| --- | --- |
| Intake do caso | Dados do contrato, incorporadora, status de pagamento, urgência |
| Curadoria e qualificação | Critérios claros de aceitação e rejeição de casos |
| Match guiado | Conexão qualificada entre cedente e cessionário (sem "classificado aberto") |
| Formalização assistida | Checklist documental + termos + registro de consentimento por cenário (na planta vs financiado) |
| Trilha de auditoria | Logs de aceite, uploads, histórico de decisões e responsáveis |
| Níveis de verificação | Básico vs verificado, com regras objetivas de publicação |

### 6.2 O que o MVP NÃO faz

#### 6.2.1 Intermediação financeira

- O RepasseSeguro **não será financeira nem escrow** no início.
- Pagamentos, quitações e repasses monetários ocorrerão diretamente entre as partes ou via agentes existentes (bancos, incorporadoras).
- A comissão é cobrada separadamente via Pix ou TED, não retida do valor transacionado.

#### 6.2.2 Garantias de liquidez

- O modelo é de **marketplace estruturado com curadoria**, não de *iBuyer*.
- **Não haverá garantia de venda** em X dias, nem recompra pela própria plataforma.
- O risco de não venda permanece com o usuário.
- A narrativa é **"organizamos e formalizamos quando faz sentido"**, não "garantimos saída e preço superior".

#### 6.2.3 Integrações com cartórios

- Nada de automação profunda de escrituras, ITBI e averbações no primeiro momento.
- A plataforma foca em **orientar** e apoiar com checklists por cenário, não em substituir o cartório.

#### 6.2.4 Integrações com bancos e incorporadoras

- A atuação será **paralela** aos sistemas legados, usando fluxos já existentes offline.
- Integrações futuras (cartórios, bancos, incorporadoras) devem ser **planejadas**, mas sem depender delas para o MVP funcionar.

#### 6.2.5 Serviços adjacentes fora do MVP

- Garantias de adimplência futura (seguros, fianças, etc.).
- Serviços jurídicos personalizados de alta complexidade (advocacia plena).
- Integrações com bases de dados governamentais em tempo real.
- Marketplace aberto sem curadoria.
- Automação sem revisão humana em pontos de risco.

#### 6.2.6 Limites de atuação jurídica no MVP

| ✅ O que fazemos | ❌ O que NÃO fazemos |
| --- | --- |
| Organização documental e checklist de formalização | Intermediação imobiliária formal (não somos corretores) |
| Conexão guiada entre cedente e cessionário | Assessoria jurídica (não substituímos advogado) |
| Registro de consentimentos, termos e trilha de auditoria | Garantia de aprovação pela incorporadora |
| Suporte operacional ao fluxo de substituição | Custódia de valores ou intermediação financeira |

<aside>
⚙️

**O MVP será, portanto, uma plataforma de conexão, curadoria e formalização assistida**, focada em matching qualificado, orientação por cenário e trilha de evidências — sem assumir papéis de fintech, cartório digital, garantidor de liquidez ou intermediário imobiliário formal.

</aside>

---

## 7. Riscos críticos e fatores que podem matar a ideia

### 7.1 Tripé crítico: Confiança, Distribuição e Compliance

O que mata o RepasseSeguro **não é tecnologia**. São três pilares:

#### 7.1.1 Confiança

Se a plataforma parecer só mais um classificado disfarçado, o usuário migra para o que já conhece (WhatsApp, OLX, corretores informais). A confiança é construída com **verificação, curadoria e transparência de processo** — não com promessas de resultado.

#### 7.1.2 Distribuição

Sem canal forte de aquisição de oferta e demanda, o projeto cai no clássico problema de **ovo e galinha** de marketplace. Canais prioritários identificados:

1. **Advogados imobiliários** — já lidam com distratos e têm o cliente na mão.
2. **Corretores de nicho** (imóveis na planta, investidores) — conhecem quem quer sair.
3. **Comunidades de investidores imobiliários** (grupos, fóruns, influenciadores do setor).
4. **Incorporadoras** — podem indicar casos de distrato para reduzir atrito e custo.

#### 7.1.3 Compliance

Sem governança mínima, o risco jurídico e reputacional explode rápido — especialmente em setor com obrigações de **prevenção à lavagem de dinheiro** (Lei 9.613/1998 e Resolução COFECI 1.336/2014).

<aside>
🔴

**Se esses três pontos não forem tratados desde o início**, a plataforma tende a virar: (1) mais um classificado com risco jurídico, (2) um funil caro de leads que não convertem, ou (3) um pipeline de negociações que morrem na última etapa.

</aside>

### 7.2 Matriz de riscos detalhada

| Risco | Impacto | Probabilidade | Sinais de alerta | Mitigação no MVP |
| --- | --- | --- | --- | --- |
| Fraude / golpe | Muito alto | Média | Anúncios sem documento, inconsistências, usuários sem verificação | Checklist + níveis de verificação + regras de publicação + trilha de auditoria |
| Baixa formalização (negócios "travando" na anuência) | Alto | Média/alta | Taxa de conversão match→fechamento muito baixa | Trilhas por cenário + orientação documental + parceiros (advogados, despachantes) |
| Oferta sem qualidade (anúncios incomparáveis, preço irreal) | Médio/alto | Média | Cadastros incompletos, desistência no checklist | Padrão mínimo de dados + validações de cadastro + curadoria de aceitação |
| Demanda fria (baixa conversão para contato) | Alto | Baixa/média | Poucos leads, baixo engajamento, alto bounce | Segmentação (região/obra) + transparência de dados + verificação + canais de distribuição |
| Operação cara (muito suporte por transação) | Alto | Alta | Custo por caso > receita por caso | Assistido no piloto + automação progressiva do checklist + gates de viabilidade |
| Marketplace aberto sem curadoria (risco reputacional) | Muito alto | Baixa (se controlado) | Pressão para escalar antes de estabilizar | Curadoria fechada obrigatória no MVP + posicionamento claro de infra, não vitrine |
| Compliance / lavagem de dinheiro | Muito alto | Baixa | Operações de alto valor sem trilha, valores incompatíveis com perfil | Termos claros + KYC mínimo + logs + política de dados + assessoria jurídica |

### 7.3 Condição de NO GO

<aside>
🔴

**Se o caminho escolhido for um marketplace aberto**, sem curadoria e sem governança, com promessa de liquidez e preço "melhor" sem controle, o veredito muda para **NO GO**, porque o risco reputacional e jurídico supera o ganho potencial. A narrativa precisa ser institucional e conservadora.

</aside>

---

## 8. Síntese da Análise Geral

### 8.1 Visão consolidada

Em resumo, o **MVP do RepasseSeguro** se ancora em:

- Um **problema estrutural** do mercado: distratos e repasses afetando >50 mil contratos/ano, com perdas de 20–50% para o comprador.
- Um **conjunto claro de atores** afetados, com dores relevantes e mensuráveis — e disposição para pagar por uma solução.
- Um **ecossistema de soluções existentes** ainda fragmentado e incompleto — nenhum player ocupa a posição de infraestrutura neutra e jurídica-first.
- **Hipóteses razoáveis** de demanda (TAM de R$ 2,5B–4,5B/ano em comissões), disposição para negociar, suporte jurídico (Lei 13.786/2018) e bom timing (pós-alta de juros).
- Um **escopo inicial propositalmente contido**, que evita dispersão regulatória e operacional.

### 8.2 Hipóteses críticas a validar

Antes de investir pesado, 3 hipóteses precisam ser validadas:

1. **Oferta:** conseguimos captar anúncios com documentação mínima em quantidade suficiente?
2. **Demanda:** compradores aceitam negociar via plataforma quando há transparência e verificação?
3. **Formalização:** em quais cenários a cessão/transferência é praticável sem integrações, e qual é o tempo médio?

> Em quantos casos a incorporadora/banco efetivamente permite a cessão?
> 

> Quais campos e documentos são indispensáveis para reduzir desconfiança?
> 

> Qual é a taxa de fechamento aceitável para sustentar o modelo?
> 

<aside>
✅

**A Análise Geral indica que há espaço real para uma solução como o RepasseSeguro**, desde que o MVP se mantenha disciplinado no escopo (curadoria fechada + formalização assistida), ataque o problema de confiança e formalização como prioridade zero, e respeite as limitações regulatórias e operacionais do mercado imobiliário brasileiro.

</aside>

---

## 9. Alinhamento com o Modelo de Negócios (monetização)

### 9.1 Visão resumida do modelo de receita

O RepasseSeguro monetiza com **2 receitas principais**, definidas no documento "Modelo de Negócios":

#### 9.1.1 Receita 1 — Comissão do cedente (quem repassa)

- O cedente escolhe **um dos 4 cenários de retorno** no cadastro: **A** (assumir saldo devedor, RS sem comissão), **B** (100% do pago), **C** (+30%) ou **D** (+50%).
- Cenários B/C/D: RS cobra **20%** sobre (Valor Recuperado − Valor Distrato Referência).
- Cenário A: comissão da RS = **R$ 0** em todos os casos.
- Se o cenário escolhido não atrair comprador, o cedente pode descer de cenário mediante aceite formal (D → C → B → A).

#### 9.1.2 Receita 2 — Comissão do comprador (quem compra o repasse)

- Cobrada sobre a **valorização real**: Δ = max(0, Tabela Atual − Tabela do Contrato).
- Comissão = **20%** × Δ.
- Comissão **R$ 0** quando Δ ≤ 0, salvo aplicação de piso mínimo (R$ 5.000) previsto em Termo Comercial.
- Exceção (Cenário A + Δ = 0): comissão = 20% × Valor Pago pelo Cedente.

#### 9.1.3 O que NÃO há

- Não há assinatura, mensalidade ou cobrança recorrente — apenas comissão de sucesso nas pontas.

### 9.2 Regras críticas de governança conectadas ao problema

| Regra | Detalhe | Por que importa |
| --- | --- | --- |
| **Tabela de referência** | Sempre a tabela vigente na **data do lance do comprador**, quando houver lance registrado, com fonte e evidência anexadas ao dossiê | Evita disputa sobre base de cálculo |
| **Fontes aceitas** | Construtora → oferta pública → avaliação formal | Hierarquia clara de evidência |
| **Δ ≤ 0** | Comissão do comprador = R$ 0, salvo piso mínimo | Só cobra quando há ganho real |
| **Evento de Fechamento** | Instrumento assinado + preço comprovado + condições mínimas atendidas | Gatilho único e inequívoco de cobrança |
| **Prazo de pagamento** | 5 dias úteis após emissão da cobrança no Fechamento | Previsibilidade de caixa |
| **Desistência pós-fechamento** | Até 15 dias corridos: estorno integral. Após: mediação de 10 dias úteis, sem estorno automático | Protege o modelo sem criar arbitragem |

### 9.3 Como isso conversa com a Análise Geral

- O modelo de monetização reforça a tese central da Análise Geral: **capturar valor apenas quando há resultado econômico real e formalizado**, e não em promessas de liquidez ou volume.
- Os riscos de confiança, compliance e disputa de narrativa (distrato vs repasse) são mitigados com **regras objetivas** de cálculo, datas, fontes de tabela e evento único de Fechamento.
- Sem aderência às fórmulas, datas e regras de cálculo do Modelo de Negócios, o risco de disputa e ruído comercial aumenta — e a viabilidade do projeto diminui proporcionalmente.

### 9.4 Projeções de referência (cenário Base)

| Período | Casos/mês | Casos (período) | Receita cedente (R$ 30k/caso) | Receita comprador (R$ 20k/caso) |
| --- | --- | --- | --- | --- |
| M1–M3 (início) | 6 | 18 | R$ 540.000 | R$ 360.000 |
| M4–M6 (rampa) | 8 | 24 | R$ 720.000 | R$ 480.000 |
| M7–M12 (aceleração) | 12 | 72 | R$ 2.160.000 | R$ 1.440.000 |
| **Ano 1 total** | **Média 9,5** | **114** | **R$ 3.420.000** | **R$ 2.280.000** |

<aside>
💡

**Leitura rápida:** 114 casos fechados/ano com ticket médio total de R$ 50k geram **R$ 5,7M** de receita. A proporção mantém ~**60% cedente / 40% comprador**. Com taxa de conversão estimada de 20%, são necessários ~60 casos captados/mês para atingir 12 fechamentos/mês na fase de aceleração.

</aside>

---

## 10. Próximos passos e documentos relacionados

### 10.1 Documentos complementares

| Documento | Escopo | Status |
| --- | --- | --- |
| Modelo de Negócios | Detalhamento completo das 2 receitas, 4 cenários de retorno, fórmulas, ciclo de vida, SLAs, agentes de IA e FAQ | v3.4 |
| Estudo de Viabilidade | Viabilidade de mercado, operacional, jurídica e econômica do MVP | v1.0 |
| Veredito Final | GO/NO GO com condições mínimas, roadmap e gates mensuráveis | v1.2 |
| Pesquisa de Mercado / Benchmark | Análise de mercado endereçável, contexto macro, framework regulatório, benchmark competitivo com scoring multidimensional, análise de forças setoriais (Porter adaptado), mapa de posicionamento e matriz de riscos/oportunidades | **v2.0** |
| Fundraising | Tese de investimento, valuation e estrutura de rodada | Em construção |

### 10.2 Perguntas de validação em aberto

> Em quantos casos a incorporadora/banco efetivamente permite a cessão no cenário atual?
> 

> Qual canal gera o primeiro caso formalizado com menor custo e maior qualidade?
> 

> Qual dos 4 cenários (A/B/C/D) é mais fácil de fechar nos primeiros 10 casos? O escalonamento funciona conforme previsto?
> 

> Como ocupar o espaço de "infraestrutura neutra e jurídica-first" para repasses imobiliários, aprendendo com as estratégias de liquidez e confiança dos concorrentes sem assumir riscos desnecessários de marketplace aberto?
>