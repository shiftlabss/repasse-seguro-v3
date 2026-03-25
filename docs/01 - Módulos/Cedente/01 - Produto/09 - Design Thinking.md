# 09 - Design Thinking

Fase: 3 — Produto
Área: Produto

<aside>
📋

**Documento Normativo — Repasse Seguro**

Este documento é referência obrigatória para decisões de produto, UX e experiência da Repasse Seguro. Qualquer feature ou fluxo que contradiga o que está aqui deve ser revisado antes de implementação.

</aside>

---

## Design Thinking — Repasse Seguro

### Ideação Centrada no Usuário para a Infraestrutura de Formalização de Cessões

| **Destinatário** | Shift Labs — Produto, UX, Marketing, Comercial, Jurídico, Engenharia, CS, Operações |
| --- | --- |
| **Escopo** | Design Thinking aplicado à Repasse Seguro: personas, dores, "como podemos…?", ideação de features e riscos de UX — tudo centrado no ecossistema de cessões imobiliárias. |
| **Versão** | v2.0 |
| **Responsável** | Fernando Calado |
| **Data da versão** | 25/02/2026 11:41 (America/Fortaleza) |

---

<aside>
📌

**TL;DR**

- **4 personas mapeadas** com dores, necessidades e implicações: cedente PF, cessionário (comprador oportunista/investidor), corretor/advogado e incorporadora.
- Cada persona se conecta ao **propósito triplo**: recuperação patrimonial (cedente), oportunidade verificada (cessionário), inteligência de mercado (ecossistema).
- **14 perguntas "como podemos…?"** estruturadas por stakeholder — o norte verdadeiro do produto.
- **15 ideias de produto** organizadas por persona e validadas contra o propósito triplo.
- **Riscos de UX** mapeados por ator — o que observar antes de escalar.
- Todo o doc alinhado com os **4 Princípios de Voz** (Clareza, Seriedade sem frieza, Transparência radical, Empoderamento sem promessa), **Memorando de Essência v2.0** e os **6 Princípios Inegociáveis**.
</aside>

---

# 1. Objetivo e Enquadramento

## 1.1 Objetivo Central

Aplicar **Design Thinking** à **Repasse Seguro — infraestrutura de formalização de cessões imobiliárias** — para que cada decisão de produto seja **centrada nas pessoas** que participam do ecossistema de cessões:

- **Cedentes** (quem precisa sair do contrato)
- **Cessionários** (quem quer entrar com segurança e desconto)
- **Corretores e Advogados** (parceiros profissionais)
- **Incorporadoras** (gestoras dos contratos originais)

<aside>
💡

**Posicionamento estratégico**

A Repasse Seguro é o **terceiro caminho** — entre o distrato punitivo (perda de 25-50%) e a informalidade arriscada (contrato de gaveta). Toda ideação neste documento parte desse posicionamento: não estamos construindo um marketplace de imóveis, estamos construindo uma **infraestrutura de confiança**.

</aside>

## 1.2 Conexão com o Propósito Triplo

<aside>
🎯

Toda ideia neste documento deve resolver pelo menos **um** dos três propósitos da Repasse Seguro:

1. **Recuperação patrimonial** → para o cedente
2. **Oportunidade verificada** → para o cessionário
3. **Inteligência de mercado** → para o ecossistema

Se não resolve nenhum dos três, provavelmente não é prioridade.

</aside>

## 1.3 Resultado Esperado

- **Experiência clara e acolhedora** para o cedente em momento de vulnerabilidade.
- **Confiança e transparência radical** para o cessionário antes de qualquer proposta.
- **Ferramenta de resolução com comissão justa** para o corretor/advogado.
- **Canal de redução de distratos sem custo** para a incorporadora.
- Coerência com **Memorando de Essência**, **Tom de Voz**, **Modelo de Negócios**, **Proposta de Valor** e **One-Liner e ICPs**.

---

# 2. Empatia: Quem São os Usuários da Repasse Seguro

## 2.1 Grupos Principais de Usuários

- **Cedente PF** (pessoa física que precisa sair do contrato)
- **Cessionário** (comprador oportunista / investidor cauteloso)
- **Corretor / Advogado imobiliário** (parceiro profissional)
- **Incorporadora médio porte** (gestora do contrato original)

<aside>
🎯

**Insight**

Todos eles são, em graus diferentes, **usuários da Repasse Seguro**. Se você desenhar só para o cedente e ignorar o corretor ou a incorporadora, o produto não emplaca no ecossistema real. A formalização depende de **todos os atores** cooperando.

</aside>

---

## 2.2 Persona 1 — Cedente PF

**Nome fictício:** Mariana, 34 anos

**Contexto:** Comprou apartamento na planta há 2 anos. Divorciou-se e não consegue manter as parcelas. Não sabe que existe alternativa ao distrato.

### 2.2.1 Necessidades

- Entender **quanto pode recuperar** vs o distrato.
- Ter clareza sobre **o processo de cessão** — passo a passo.
- Sentir **segurança jurídica** em cada etapa.
- Não ser pressionada a aceitar condições ruins.
- Resolver em **tempo razoável** (45-60 dias).

### 2.2.2 Dores Atuais

- Medo de **perder 25-50% do patrimônio** no distrato.
- **Desconhecimento total** sobre cessão como alternativa.
- Desconfiança de processos informais (**contrato de gaveta**).
- **Vulnerabilidade emocional** — divórcio, perda de renda, mudança de cidade.
- Falta de orientação clara — **não sabe por onde começar**.

### 2.2.3 Implicações para a Repasse Seguro

- Linguagem **acolhedora e educativa** — alinhada ao arquétipo Cuidador. Nunca técnica ou burocrática.
- **Simulação de cenários** (A/B/C/D) visível desde o primeiro contato — empoderamento sem promessa.
- **🛡️ Guardião do Retorno** (IA) como primeiro ponto de orientação — empático, calmo, educativo.
- **Dossiê verificado** que gera confiança antes de qualquer negociação.
- Tom alinhado: **Clareza acima de tudo** + **Empoderamento sem promessa**.

### 2.2.4 Perfis Mais Comuns de Cedente

| **Perfil** | **Gatilho de saída** | **Dor específica** |
| --- | --- | --- |
| Divórcio / separação | Não pode manter as parcelas sozinho | Urgência emocional + pressão financeira simultâneas |
| Perda de renda / desemprego | Inadimplência ou risco de inadimplência | Medo de negativação + perda total do investimento |
| Mudança de cidade | Oportunidade profissional ou familiar em outro estado | Contrato preso sem liquidez, imóvel ainda na planta |
| Investidor que precisa de liquidez | Precisa resgatar capital investido na planta | Distrato punitivo é inaceitável, gaveta é arriscado |
| Arrependimento da compra | Compra por impulso ou mudança de planos | Culpa + desinformação sobre alternativas |

---

## 2.3 Persona 2 — Cessionário (Comprador Oportunista / Investidor Cauteloso)

**Nome fictício:** Rafael, 38 anos

**Contexto:** Investidor cauteloso que busca imóvel abaixo da tabela com segurança. Já perdeu dinheiro em contrato de gaveta. Quer dados, não promessas.

### 2.3.1 Necessidades

- Acessar oportunidades **verificadas** com desconto real (**Δ documentado**).
- Conhecer o **dossiê completo** antes de fazer proposta.
- Ter **segurança jurídica** na formalização — conta escrow, anuência, trilha.
- Comparar oportunidades com **dados objetivos**.

### 2.3.2 Dores Atuais

- Desconfiança de oportunidades **"boas demais"** — sem verificação.
- **Medo de golpe** em repasse informal (contrato de gaveta).
- Falta de verificação documental nos canais existentes (**OLX, WhatsApp, grupos**).
- Não saber avaliar se o **Δ é real** ou inflado.
- **Processo opaco** — sem trilha de auditoria.

### 2.3.3 Implicações para a Repasse Seguro

- **📊 Analista de Oportunidades** (IA) com foco em dados: Δ, tabela, comparações de mercado.
- **Dossiê auditável** disponível antes de qualquer proposta.
- **Conta escrow** como garantia visível de proteção.
- **Ficha de caso** com todos os dados relevantes — zero surpresa.
- Tom alinhado: **Transparência radical** + **Clareza acima de tudo**.

### 2.3.4 Perfis Mais Comuns de Cessionário

| **Perfil** | **Motivação** | **Dor específica** |
| --- | --- | --- |
| Investidor PF de oportunidade | Comprar abaixo da tabela para valorizar | Risco de golpe + falta de verificação documental |
| Comprador de primeira moradia | Acessar imóvel mais barato com segurança | Desconhece o processo de cessão, precisa de orientação total |
| Investidor recorrente | Múltiplas cessões como estratégia patrimonial | Precisa de fluxo eficiente e dados comparativos |

---

## 2.4 Persona 3 — Corretor / Advogado Imobiliário

**Nome fictício:** Paulo, 45 anos

**Contexto:** Corretor com 15 anos de mercado. Recebe clientes com contratos problemáticos e não tem ferramenta para resolver cessão. Hoje, ou perde o cliente ou assume risco tentando intermediar informalmente.

### 2.4.1 Necessidades

- Ter uma **solução para indicar** quando o cliente precisa sair do contrato.
- **Ganhar comissão** sobre indicação sem assumir risco operacional.
- Manter **credibilidade profissional** — não quer indicar algo duvidoso.
- **Acompanhar o caso** indicado sem depender de ligações.

### 2.4.2 Dores Atuais

- **Perde clientes** que precisam sair de contratos — não sabe como resolver.
- **Assume risco** quando tenta intermediar informalmente.
- Falta de **processo estruturado** para cessão.
- Sem **ferramenta digital** para acompanhar o caso indicado.

### 2.4.3 Implicações para a Repasse Seguro

- **Programa de parceria** com comissão clara e transparente.
- **Portal do parceiro** com acompanhamento de casos indicados.
- **Kit de indicação** — material educativo para o parceiro explicar o processo ao cliente.
- Tom alinhado: **Seriedade sem frieza** + **Clareza** (profissional-para-profissional).

<aside>
⚠️

**Desafio**

O corretor/advogado precisa sentir que a RS é uma **extensão do seu serviço**, não uma ameaça ao seu papel. Se o profissional sentir que vai perder o cliente para a plataforma, não indica. O programa de parceria precisa ser transparente e justo — sempre.

</aside>

---

## 2.5 Persona 4 — Incorporadora Médio Porte

**Nome fictício:** Construtora Horizonte, ~500 unidades/ano

**Contexto:** Volume crescente de distratos que consome caixa e gera desgaste jurídico. Cessões informais entre compradores geram risco para a empresa.

### 2.5.1 Necessidades

- **Reduzir volume e custo** de distratos.
- **Manter contratos ativos** com compradores qualificados.
- Ter **compliance** na cessão (anuência documentada, trilha de auditoria).
- **Dados consolidados** sobre cessões do portfólio.

### 2.5.2 Dores Atuais

- Distrato **consome caixa** e gera passivo contábil.
- Cessões informais entre compradores geram **risco jurídico** para a incorporadora.
- Falta de **canal organizado** para encaminhar cessões.
- Sem **dados consolidados** sobre cessões por empreendimento.

### 2.5.3 Implicações para a Repasse Seguro

- **Fluxo digital de anuência** integrado ao processo — mais simples que o atual.
- **Relatórios de cessões** por empreendimento (volume, perfil, tempo, Δ).
- **Zero custo** para a incorporadora — modelo de comissão sobre resultado.
- Tom alinhado: **Transparência radical** + **Seriedade sem frieza** (B2B institucional).

### 2.5.4 Perfis de Incorporadora

| **Perfil** | **Dor específica** | **Argumento-chave** |
| --- | --- | --- |
| Incorporadora com alto volume de distratos | Impacto severo no caixa e desgaste jurídico | "Contrato continua ativo. Sem custo. Sem desgaste." |
| Incorporadora com cessões informais recorrentes | Risco jurídico de cessões sem anuência | "Formalize o que já acontece — com trilha de auditoria." |
| Incorporadora que quer dados do mercado secundário | Zero visibilidade sobre o perfil de cessões | "Dados reais de cessões por empreendimento — para decisão, não para achismo." |

---

# 3. Definição de Problemas: "Como Podemos…?"

A partir da empatia, a etapa de **Definição** transforma dores em perguntas orientadoras.

## 3.1 Para o Cedente

- **Como podemos** ajudar o cedente a **entender que existe alternativa ao distrato** antes que aceite a perda?
- **Como podemos** tornar o processo de cessão **claro e previsível** para quem nunca ouviu falar nisso?
- **Como podemos** proteger o cedente **financeiramente** sem prometer resultado?
- **Como podemos** reduzir a **ansiedade** de quem está em situação de vulnerabilidade patrimonial?

## 3.2 Para o Cessionário

- **Como podemos** garantir que cada oportunidade apresentada é **verificada e segura**?
- **Como podemos** mostrar o **Δ real** com transparência radical — fonte, cálculo, metodologia?
- **Como podemos** eliminar o **medo de golpe** em repasse, diferenciando a RS de qualquer canal informal?
- **Como podemos** tornar a formalização tão simples que o cessionário **prefira a RS** a qualquer alternativa?

## 3.3 Para o Corretor / Advogado

- **Como podemos** transformar o corretor em **agente de solução** para contratos problemáticos?
- **Como podemos** criar um programa de parceria com **comissão justa e processo zero-fricção**?
- **Como podemos** dar **visibilidade ao parceiro** sobre o andamento dos casos indicados?

## 3.4 Para a Incorporadora

- **Como podemos** reduzir distratos **sem custo** para a incorporadora?
- **Como podemos** digitalizar o **fluxo de anuência** para cessões?
- **Como podemos** gerar **dados acionáveis** sobre o perfil de cessões do portfólio?

<aside>
🎯

**Insight**

Essas 14 perguntas são o "norte verdadeiro" do produto. Qualquer feature da Repasse Seguro que não ajude a responder pelo menos uma delas tende a virar complexidade extra, não valor real.

</aside>

---

# 4. Ideação Centrada no Usuário para a Repasse Seguro

## 4.1 Princípios de Ideação

- **Centrado no cedente:** acolhimento, clareza, cenários — nunca pressão, nunca promessa vazia.
- **Centrado no cessionário:** dados, verificação, segurança — nunca opacidade, nunca "boa demais para ser verdade".
- **Centrado no parceiro:** comissão justa, processo claro, credibilidade — nunca risco de perder o cliente.
- **Centrado no propósito triplo:** toda ideia deve aliviar **recuperação, oportunidade ou inteligência**.

---

## 4.2 Ideias para a Experiência do Cedente

### 4.2.1 Simulador de Cenários (A/B/C/D)

O cedente cadastra seu caso e recebe **simulação visual** dos 4 cenários de retorno antes de qualquer conversa.

- Mostra claramente: o que receberia no distrato vs o que pode recuperar com cessão.
- Cenários com fórmula visível, fonte e data — transparência radical.

**Valor:** clareza imediata, empoderamento sem promessa.

**Propósito triplo:** Recuperação patrimonial (cedente).

---

### 4.2.2 Guardião do Retorno — Onboarding Guiado

IA que acompanha o cedente desde o primeiro contato: explica o processo, responde dúvidas, orienta sobre documentação necessária.

- Tom empático, calmo, educativo — arquétipo Cuidador.
- Nunca pressiona. Nunca promete. Orienta e acompanha.

**Valor:** acolhimento em momento vulnerável, redução de abandono.

**Propósito triplo:** Recuperação + Inteligência (dados de dúvidas frequentes).

---

### 4.2.3 Dashboard do Cedente

Painel com **status do caso** (9 estados), próximas etapas, documentos pendentes e histórico de ações.

- Cada mudança de estado vem com explicação clara do que acontece agora.
- Indicador visual de progresso — cedente nunca se sente perdido.

**Valor:** transparência total, previsibilidade, redução de ansiedade.

**Propósito triplo:** Recuperação patrimonial.

---

### 4.2.4 Dossiê Visual — "Seu Caso em 1 Página"

Resumo visual do dossiê verificado: contrato, valores pagos, cenários, status — em linguagem acessível.

- O cedente entende seu próprio caso sem precisar de advogado.
- Pode compartilhar com familiares para tomar decisão informada.

**Valor:** empoderamento com informação, não com pressão.

**Propósito triplo:** Recuperação + Inteligência.

---

### 4.2.5 Notificações de Progresso

Avisos proativos a cada mudança de estado: "Seu caso avançou para Verificação" com explicação do que acontece agora e próximos passos.

- Canais: push, e-mail, WhatsApp (conforme preferência).
- Tom: claro, objetivo, acolhedor. Nunca genérico.

**Valor:** previsibilidade, conexão emocional com o processo.

**Propósito triplo:** Recuperação patrimonial.

---

## 4.3 Ideias para a Experiência do Cessionário

### 4.3.1 Analista de Oportunidades — Curadoria Inteligente

IA que apresenta oportunidades verificadas com **Δ calculado**, tabela de referência e comparações de mercado.

- Tom analítico, objetivo, orientado a dados — nunca empurra, informa.
- Cada recomendação vem com fonte, cálculo e metodologia.

**Valor:** confiança nos dados, eliminação do achismo.

**Propósito triplo:** Oportunidade verificada.

---

### 4.3.2 Ficha do Caso — Dossiê para Decisão

Cada oportunidade tem ficha completa: localização, incorporadora, Δ, documentação verificada, cenário de retorno, estágio da obra.

- Tudo que o cessionário precisa para decidir está em um lugar.
- Zero surpresa — se tem risco, a ficha mostra.

**Valor:** decisão informada, transparência radical.

**Propósito triplo:** Oportunidade + Inteligência.

---

### 4.3.3 Comparador de Oportunidades

Ferramenta para comparar 2-3 casos lado a lado: Δ, localização, incorporadora, estágio da obra, cenário de retorno.

**Valor:** racionalidade na decisão, múltiplos dados em uma tela.

**Propósito triplo:** Oportunidade verificada.

---

### 4.3.4 Alerta de Nova Oportunidade

Notificação quando um caso verificado compatível com o perfil do cessionário entra na plataforma.

- Critérios configuráveis: cidade, faixa de Δ, incorporadora, tipo de imóvel.

**Valor:** acesso antecipado, fidelização.

**Propósito triplo:** Oportunidade + Inteligência.

---

## 4.4 Ideias para Parceiros (Corretor / Advogado)

### 4.4.1 Portal do Parceiro

Dashboard com: casos indicados, status de cada um, comissões acumuladas, material educativo.

- Atualização em tempo real — parceiro nunca precisa ligar para perguntar.
- Cálculo de comissão visível: base, percentual, status de pagamento.

**Valor:** profissionalização, transparência, confiança.

**Propósito triplo:** Inteligência do ecossistema.

---

### 4.4.2 Kit de Indicação

Material pronto para o parceiro apresentar a RS ao cliente: explicação do processo, cenários, comparação distrato vs cessão.

- PDF + apresentação + FAQ — o parceiro não precisa "vender" a RS, apenas indicar com material profissional.

**Valor:** reduz fricção de indicação, mantém credibilidade do profissional.

---

### 4.4.3 Link de Indicação Rastreável

URL única do parceiro para encaminhar clientes. Toda conversão é registrada automaticamente.

- Zero burocracia. O parceiro compartilha o link. A plataforma faz o resto.

**Valor:** rastreabilidade total, comissão garantida.

---

## 4.5 Ideias para Incorporadora e Ecossistema

### 4.5.1 Fluxo Digital de Anuência

Incorporadora recebe solicitação de anuência com dossiê completo via plataforma. Aprova, recusa ou solicita ajuste digitalmente.

**Valor:** reduz tempo de formalização, elimina papel, mantém trilha de auditoria.

**Propósito triplo:** Inteligência do ecossistema.

---

### 4.5.2 Relatório de Cessões por Empreendimento

Dashboard B2B: volume de cessões, perfil de cedentes, tempo médio de ciclo, Δ praticado, taxa de conclusão.

**Valor:** dados para tomada de decisão, redução proativa de distratos.

**Propósito triplo:** Inteligência de mercado.

---

### 4.5.3 Canal Direto de Encaminhamento

Incorporadora encaminha cedentes diretamente para a RS **antes que formalizem o distrato**.

- Integração simples: e-mail, formulário ou API.
- O contrato continua ativo com um comprador qualificado — sem custo para a incorporadora.

**Valor:** preserva fluxo de caixa, mantém contrato ativo.

---

## 4.6 Matriz de Ideias × Propósito Triplo

| **Ideia** | **Persona** | **Recuperação (cedente)** | **Oportunidade (cessionário)** | **Inteligência (ecossistema)** |
| --- | --- | --- | --- | --- |
| Simulador de Cenários | Cedente | ✅ |  | ✅ |
| Guardião do Retorno | Cedente | ✅ |  | ✅ |
| Dashboard do Cedente | Cedente | ✅ |  |  |
| Dossiê Visual | Cedente | ✅ |  | ✅ |
| Notificações de Progresso | Cedente | ✅ |  |  |
| Analista de Oportunidades | Cessionário |  | ✅ | ✅ |
| Ficha do Caso | Cessionário |  | ✅ | ✅ |
| Comparador de Oportunidades | Cessionário |  | ✅ |  |
| Alerta de Nova Oportunidade | Cessionário |  | ✅ | ✅ |
| Portal do Parceiro | Corretor/Advogado |  |  | ✅ |
| Kit de Indicação | Corretor/Advogado | ✅ |  |  |
| Link de Indicação Rastreável | Corretor/Advogado |  |  | ✅ |
| Fluxo Digital de Anuência | Incorporadora | ✅ |  | ✅ |
| Relatório por Empreendimento | Incorporadora |  |  | ✅ |
| Canal Direto de Encaminhamento | Incorporadora | ✅ | ✅ | ✅ |

---

# 5. Riscos de UX a Observar em Testes

## 5.1 Para o Cedente

- Linguagem **técnica ou jurídica** que afasta em vez de acolher — o cedente já está vulnerável.
- Simulador que parece **promessa de resultado** em vez de apresentação de cenários.
- Processo que **não comunica progresso** — cedente se sente abandonado no meio do caminho.
- Guardião do Retorno com **tom frio ou robótico** — deve ser empático, humano e educativo.
- Excesso de **campos obrigatórios** no cadastro inicial — alta fricção = abandono alto.

## 5.2 Para o Cessionário

- Ficha de caso **incompleta** que gera desconfiança — "se está escondendo algo, não confio".
- Δ apresentado **sem fonte ou metodologia** — parece inflado ou irreal.
- Processo de proposta **complexo demais** — deve ser simples e direto.
- Falta de clareza sobre **custos e comissão** — transparência radical é obrigatória.

## 5.3 Para o Corretor / Advogado

- Medo de **perder o cliente** para a plataforma — o parceiro precisa sentir que a RS é extensão do serviço, não competidora.
- Comissão **opaca** ou com regras confusas — confiança quebrada = parceiro perdido.
- Portal do parceiro **sem atualização** — parceiro se sente ignorado.

## 5.4 Para a Incorporadora

- Processo de anuência **mais burocrático** que o atual — deve ser mais simples, não mais complexo.
- Relatórios **genéricos** sem valor acionável — dados precisam gerar decisão.
- Percepção de que a RS **compete com vendas diretas** — o argumento é complementaridade, não substituição.

<aside>
🔴

**Regra de Ouro de UX**

Em qualquer teste, se a experiência da Repasse Seguro **aumentar a desconfiança** para cedente, cessionário, parceiro ou incorporadora, é sinal de que a solução está mais próxima de "mais um classificado" do que de "infraestrutura de confiança". O objetivo é sempre **simplificar a vida e construir confiança** em quem usa.

</aside>

---

# 6. Teste Ácido Aplicado ao Design Thinking

<aside>
🧪

**Antes de aprovar qualquer ideia ou protótipo, aplique o Teste Ácido do Memorando de Essência:**

Se você tirar o nome "Repasse Seguro" do conceito e não der para saber que é sobre nós, o conceito precisa de revisão.

**Sinais de identidade obrigatórios (pelo menos 1):**

- **Terceiro caminho** (não somos distrato nem informalidade)
- **Formalização com trilha de auditoria** (ninguém mais faz isso no mercado de cessões)
- **Comissão sobre resultado** (se não fecha, ninguém paga)
- **Dossiê verificado** (curadoria antes de qualquer conversa)
- **Infraestrutura de confiança** (não marketplace, não classificado)
</aside>

---

# 7. Como Usar Este Documento

## 7.1 Para Produto

- Usar as personas, dores e "como podemos…" como **filtro de priorização** de funcionalidades.
- Validar se cada item do backlog se conecta claramente a: cedente, cessionário, corretor/advogado ou incorporadora.
- Sempre cruzar este doc com **Casos de Uso**, **Service Design**, **Modelo de Negócios** e **Proposta de Valor** ao tomar decisões.
- Usar a **Matriz de Ideias × Propósito Triplo** (Seção 4.6) para priorizar features com maior cobertura de valor.

## 7.2 Para UX / Design

- Usar as ideias desta página como **ponto de partida de protótipos** (não como solução final).
- Criar roteiros de teste com base nas perguntas de definição ("como podemos…").
- Medir sempre: clareza do processo, confiança percebida, tempo para primeira ação e taxa de abandono.
- Toda microcopy e interação deve seguir os **4 Princípios de Voz** (Clareza, Seriedade sem frieza, Transparência radical, Empoderamento sem promessa).

## 7.3 Para Comercial e Estratégia

- Usar as personas e exemplos de jornada para **explicar a RS em discovery** e apresentações ("Antes — distrato/gaveta" vs "Com a RS").
- Conectar este doc com **Proposta de Valor**, **Modelo de Negócios** e **One-Liner e ICPs** para manter narrativa única.
- Adaptar vocabulário ao **ICP** do lead — cedente precisa de acolhimento, cessionário precisa de dados, incorporadora precisa de B2B.

## 7.4 Onboarding de Novos Membros

> **Exercício prático sugerido**
> 

> Peça para a pessoa ler este documento e responder, por escrito:
> 

> "Escolha 2 personas e, para cada uma, identifique: qual é o propósito triplo mais relevante, qual dor principal a RS resolve, e qual ideia da Seção 4 você priorizaria. Justifique."
> 

---

# 8. Conexão com os 4 Princípios de Voz

Cada seção do Design Thinking ativa diferentes Princípios de Voz:

| **Seção** | **Princípio dominante** | **Por quê** |
| --- | --- | --- |
| Empatia (personas) | Seriedade sem frieza + Empoderamento sem promessa | Entender dores reais sem banalizar nem criar expectativa irreal |
| Definição ("como podemos…?") | Clareza acima de tudo | Perguntas diretas focadas em gerar ação |
| Ideação | Transparência radical + Clareza | Features que mostram dados, fontes e trilha — nunca escondem |
| Riscos de UX | Empoderamento sem promessa | Honestidade sobre limitações e cuidados |
| Teste Ácido | Seriedade sem frieza + Transparência radical | Honestidade sobre identidade e coerência — se não passa, refaz |

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
| **Jobs To Be Done** | Que "trabalho" cada ator resolve com a RS | [10 - Jobs To Be Done](10%20-%20Jobs%20To%20Be%20Done%20312d824e597f809cbc23ef6b1a495514.md) |
| **Service Design** | Blueprint da jornada completa (9 estados) | [11 - Service Design](11%20-%20Service%20Design%20312d824e597f80b687f6cea30a3a736e.md) |
| **Casos de Uso** | Cenários práticos — o que a RS faz em cada situação real | [08 - Casos de Uso](08%20-%20Casos%20de%20Uso%20312d824e597f809fa5bbc5e9bd29efd7.md) |
| **UX Writing** | Textos específicos de cada tela e interação | [12 - UX Writing](12%20-%20UX%20Writing%20312d824e597f80f7bbcec784b9830523.md) |
| **Modelo de Negócios** | Comissões, unit economics, ciclo de estados | [14 - Modelo de Negócios](14%20-%20Modelo%20de%20Neg%C3%B3cios%20301d824e597f8003891ac9058bb4f812.md) |
| **Proposta de Valor** | Por que contratar — ROI, cenários, argumentos | [15 - Proposta de Valor - Cedente/Cessionário](15%20-%20Proposta%20de%20Valor%20-%20Cedente%20Cession%C3%A1rio%20303d824e597f80ef8783f56e9efc039a.md) |
| **One-Liner e ICPs** | 5 ICPs, one-liners oficiais, anti-ICP | [03 - One-Liner e ICPs](03%20-%20One-Liner%20e%20ICPs%20301d824e597f8076a76ad0ef11fe3804.md) |

### Mapa do Ecossistema

<aside>
🗺️

[05 - Memorando de Essência](05%20-%20Memorando%20de%20Ess%C3%AAncia%20312d824e597f807fb684d0bd73affebd.md) → "O que somos"

[04 - Manifesto da Marca](04%20-%20Manifesto%20da%20Marca%20303d824e597f8023bc06f5f40b1e40ea.md) → "Por que existimos"

[07 - Tom de Voz e Identidade Verbal](07%20-%20Tom%20de%20Voz%20e%20Identidade%20Verbal%20303d824e597f80c6bb3ff800a72f0c72.md) → "Como falamos"

**Design Thinking** → "Para quem desenhamos" *(você está aqui)*

[10 - Jobs To Be Done](10%20-%20Jobs%20To%20Be%20Done%20312d824e597f809cbc23ef6b1a495514.md) → "Que trabalho resolvemos"

[11 - Service Design](11%20-%20Service%20Design%20312d824e597f80b687f6cea30a3a736e.md) → "Como a jornada funciona"

[08 - Casos de Uso](08%20-%20Casos%20de%20Uso%20312d824e597f809fa5bbc5e9bd29efd7.md) → "O que a RS faz na prática"

</aside>

---

# 10. Changelog

| **#** | **Versão** | **Data** | **Mudanças** |
| --- | --- | --- | --- |
| 1 | **v1.0** | 25/02/2026 | Versão inicial criada para o Menux (perspectiva incorreta). |
| 2 | **v2.0** | 25/02/2026 | **Reescrita completa para Repasse Seguro.** 4 personas adaptadas (cedente PF, cessionário, corretor/advogado, incorporadora) com dores, necessidades e perfis detalhados. 14 perguntas "como podemos…?" recontextualizadas para cessões imobiliárias. 15 ideias de produto (simulador de cenários, Guardião do Retorno, Analista de Oportunidades, portal do parceiro, fluxo de anuência digital, etc.). Matriz de Ideias × Propósito Triplo. Riscos de UX por ator. Teste Ácido adaptado. Conexão com 4 Princípios de Voz. Referências cruzadas atualizadas para ecossistema RS. |