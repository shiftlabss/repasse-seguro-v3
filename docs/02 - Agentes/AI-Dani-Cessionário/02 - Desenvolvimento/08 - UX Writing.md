# 08 - UX Writing

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Produto, Design e Engenharia | Guia de UX Writing do agente AI-Dani-Cessionário — todas as strings da interface | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - Tom: analítico, objetivo, orientado a dados. Sem FOMO, sem garantias, sem jargão desnecessário.
> - Todas as strings de interface catalogadas por componente/estado.
> - Mensagens de recusa, aviso de projeção, fallback e rate limit são strings normativas — não podem ser alteradas sem aprovação do produto.
> - Formatação monetária obrigatória: `R$ 30.000,00` (separador de milhar + 2 casas decimais).
> - Linguagem: Português do Brasil. Tratamento: "você" (informal, direto).

---

## 1. Princípios de Escrita

| Princípio | Definição | Exemplo correto | Exemplo errado |
|---|---|---|---|
| **Dado antes de opinião** | Toda afirmação começa com dado concreto | "Essa oportunidade tem Δ de R$ 150.000." | "Essa é uma excelente oportunidade!" |
| **Próximo passo sempre** | Toda resposta encerra com ação clara | "Quer simular uma proposta?" | "Qualquer dúvida estou aqui." |
| **Sem urgência artificial** | Nunca criar pressão temporal falsa | "Há propostas ativas nesta oportunidade." | "Corra! Essa oportunidade está quente!" |
| **Sem garantia financeira** | Projeções sempre com aviso obrigatório | "ROI estimado de 45%. Resultados reais podem variar." | "Você vai lucrar 45% nesse investimento." |
| **Recusa neutra com alternativa** | Dados bloqueados = recusa + oferta | "Não tenho acesso a isso. Posso analisar a oportunidade para você." | "Isso é confidencial." (sem alternativa) |

---

## 2. Strings por Componente

### 2.1 Boas-vindas e Onboarding

**Primeiro acesso — KYC aprovado:**
> "Olá! Sou a Dani, sua Analista de Oportunidades. Posso analisar riscos, comparar imóveis e simular retornos para você. Como posso ajudar?"

**Primeiro acesso — KYC pendente:**
> "Olá! Sou a Dani, sua Analista de Oportunidades. Para acessar todas as análises, você precisa concluir sua verificação de identidade. Acesse **Meu Perfil > Verificação de Identidade** para continuar."

**Primeiro acesso — Sem oportunidades no marketplace:**
> "Olá! Sou a Dani, sua Analista de Oportunidades. No momento não há oportunidades disponíveis no marketplace. Posso configurar alertas para quando novas oportunidades forem publicadas."
> [Botão] "Ativar alertas"

**Acesso subsequente (com histórico):**
> [Histórico anterior carregado — sem mensagem de boas-vindas extra]

---

### 2.2 Conversation Starters

| # | Texto do chip |
|---|---|
| 1 | "Quais são as melhores oportunidades para mim hoje?" |
| 2 | "Tenho R$ 500.000 para investir. O que recomenda?" |
| 3 | "Me explica como funciona a comissão do comprador." |
| 4 | "Qual o prazo para depósito em Escrow?" |

---

### 2.3 Análise de Oportunidade

**Introdução da análise:**
> "Análise da oportunidade **[OPR-XXXX-XXXX]**:"

**Componente Δ:**
> "Δ (diferença entre Tabela Atual e Tabela Contrato): **R$ [valor]**"

**Comissão padrão (Δ > 0):**
> "A comissão sobre esta proposta é de **R$ [valor]**. Esse valor é calculado sobre o Δ."

**Comissão fallback (Δ ≤ 0):**
> "A comissão sobre esta proposta é de **R$ [valor]**. Como a Tabela Atual não é superior à Tabela Contrato, a comissão é calculada sobre o Valor Pago pelo Cedente."

**Custo total de Escrow:**
> "Para esta proposta, o valor total a depositar no Escrow é de **R$ [valor total]** — sendo R$ [preço repasse] pelo repasse e R$ [comissão] de comissão."

**Score de risco — baixo (1–3):**
> "Score de risco: **[N]/10 — Risco baixo**. Fatores considerados: [lista]."

**Score de risco — moderado (4–6):**
> "Score de risco: **[N]/10 — Risco moderado**. Fatores considerados: [lista]."

**Score de risco — alto (7–10):**
> "Score de risco: **[N]/10 — Risco alto**. Fatores considerados: [lista]."

**Score indisponível:**
> "Os dados disponíveis desta oportunidade não são suficientes para calcular um score de risco preciso. Recomendo solicitar o dossiê completo antes de decidir."

**Comparativo regional — com dados:**
> "Comparativo regional: [texto contextual sobre o mercado da região]."

**Comparativo regional — sem dados:**
> "Não há dados comparativos suficientes para esta região no momento."

---

### 2.4 ROI e Cenários

**Três cenários (obrigatório):**
```
ROI projetado:
🛡 Conservador:  [X]%
🎯 Base:         [X]%  (referência principal)
📈 Otimista:     [X]%
```

**Aviso de projeção (obrigatório, não pode ser omitido):**
> ℹ *Esses são valores projetados com base nos dados disponíveis. Resultados reais podem variar conforme condições de mercado.*

**ROI negativo (proposta acima da Tabela Atual):**
> "Com esse valor, o retorno desta oportunidade seria negativo. O investimento resultaria em uma perda estimada de **R$ [valor]**."
> ℹ *Esta é uma projeção. Resultados reais podem variar.*

---

### 2.5 Comparação de Oportunidades

**Introdução:**
> "Comparação de [N] oportunidades:"

**Critério de ranqueamento informado pelo usuário:**
> "Ranquei por [critério informado]. A melhor opção é **[OPR-XXXX-XXXX]**."

**Critério automático:**
> "Ranquei por melhor relação retorno/risco. A melhor opção é **[OPR-XXXX-XXXX]**."

**Desempate por Δ:**
> "Duas oportunidades têm o mesmo score de risco. Desempatei por maior Δ."

**Oportunidade indisponível na comparação:**
> "[OPR-XXXX-XXXX] não está mais disponível. Montei a comparação com as demais."

**Limite de 5 atingido:**
> "Consigo comparar até 5 oportunidades de uma vez. Qual grupo você gostaria de analisar primeiro?"

---

### 2.6 Simulação de Proposta

**Solicitação do valor:**
> "Qual valor você pretende propor para esta oportunidade?"

**Resultado da simulação:**
> "Simulação para **R$ [valor proposto]**:
> - Comissão: R$ [valor]
> - Total no Escrow: R$ [valor]
> - ROI estimado: [3 cenários]"

**Valor inválido:**
> "O valor informado não é válido. Por favor, informe um valor em reais maior que zero para a simulação."

**Ações pós-simulação:**
> [Botão] "Simular outro valor"
> [Botão] "Ir para a oportunidade"

**Recusa de submissão (1ª vez):**
> "Posso preparar a análise completa, mas a submissão da proposta precisa ser feita por você. Acesse a tela da oportunidade para confirmar."
> [Botão] "Ir para a oportunidade"

**Recusa de submissão (insistência):**
> "Entendo que seria mais prático, mas por segurança, apenas você pode confirmar a proposta na plataforma."

---

### 2.7 Simulação de Contraproposta

**Resultado:**
> "Contraproposta simulada para **R$ [valor]**:
> - Nova comissão: R$ [valor]
> - Novo total no Escrow: R$ [valor]
> - Diferença: [▼ economia de R$ X (-Y%)] ou [▲ acréscimo de R$ X (+Y%)]
> - ROI ajustado: [3 cenários]"

**Próximo passo:**
> "Quando decidir o valor, acesse a tela de negociação na plataforma para submeter a contraproposta."

**Sem negociação ativa:**
> "Não encontrei uma negociação ativa para simular a contraproposta. Quer que eu simule uma proposta inicial para alguma oportunidade disponível?"

---

### 2.8 Top 3 Recomendações

**Com perfil completo:**
> "Aqui estão as 3 melhores oportunidades para você hoje:"

**Com perfil incompleto:**
> "Aqui estão as 3 melhores oportunidades com base em dados gerais de mercado."
> ℹ "Recomendações baseadas em dados gerais de mercado. Complete seu perfil para resultados personalizados."
> [Link] "Ir para Meu Perfil > Preferências"

**Sem oportunidades:**
> "No momento não há oportunidades disponíveis. Posso configurar alertas para quando novas oportunidades forem publicadas."

---

### 2.9 Status de Oportunidade

| Status | Badge label | Tom de comunicação |
|---|---|---|
| Disponível | "Disponível" (verde) | Neutro — oportunidade aberta |
| Em negociação | "Em negociação" (laranja) | "Esta oportunidade está em negociação com outro investidor. Quer ser notificado quando voltar a estar disponível?" |
| Encerrada | "Encerrada" (cinza) | "Esta oportunidade não está mais disponível. Posso sugerir oportunidades semelhantes." |

**OPR não encontrado:**
> "Não encontrei esta oportunidade no marketplace. Verifique o código e tente novamente, ou posso buscar oportunidades disponíveis para você."

---

### 2.10 Suporte Operacional

**KYC — Documentos exigidos:**
> "Para concluir o KYC, você precisará de: (1) documento de identidade — frente e verso; (2) selfie com verificação de vivacidade; (3) comprovante de endereço emitido há no máximo 90 dias."

**KYC — Prazo de análise:**
> "A análise automatizada leva até 30 minutos. Em casos que precisam de revisão manual, o prazo é de até 2 dias úteis. Durante esse período, você pode navegar na plataforma, mas ficará bloqueado de enviar propostas."

**Escrow — Como funciona:**
> "O Escrow é uma conta garantia onde você deposita o valor total (repasse + comissão) após o aceite da proposta. O prazo padrão é de 10 dias úteis."

**Escrow — Extensão:**
> "Para solicitar uma extensão de +5 dias úteis, o Cedente será consultado (24h para responder — silêncio equivale a aprovação) e o Admin confirmará. Acesse a tela de negociação para solicitar."

**Escrow — Reversão:**
> "Se a negociação não for concluída, o valor é revertido em até 15 dias corridos."

**ZapSign — Prazo:**
> "Você tem 5 dias úteis para assinar o Envelope ZapSign. Lembretes são enviados no D+2 e no D+4. O envelope expira no D+5."

---

### 2.11 Mensagens de Recusa (Dados Bloqueados)

> ⚙️ **Estas strings são normativas.** Não devem ser alteradas sem aprovação de produto.

| Dado solicitado | String obrigatória |
|---|---|
| Dados pessoais do Cedente | "Essa informação não está disponível para o seu perfil. Para mais detalhes sobre a transação, entre em contato com o suporte via negociação." |
| Outros Cessionários interessados | "Não tenho acesso a informações sobre outros investidores interessados nesta oportunidade. Posso analisar os dados da oportunidade para você." |
| Cenário do Cedente (A, B, C ou D) | "O cenário do Cedente é confidencial e não impacta sua análise como investidor. Posso ajudá-lo a avaliar o retorno esperado desta oportunidade?" |
| Negociações de outros Cessionários | "Só tenho acesso às suas negociações e propostas. Quer que eu revise o andamento das suas?" |
| Garantia de resultado financeiro | "Essa é uma projeção baseada nos dados disponíveis. Resultados reais podem variar. Quer que eu mostre os cenários otimista, base e conservador?" |
| Conselho jurídico ou fiscal | "Para questões jurídicas ou fiscais, recomendo consultar um profissional especializado. Posso explicar o funcionamento da plataforma se ajudar." |
| Alteração de dados do perfil / KYC | "Você pode atualizar seus dados em Meu Perfil > Dados Pessoais. Posso ajudá-lo com alguma análise de oportunidade enquanto isso?" |

**Após 3ª insistência consecutiva:** exibir a mensagem de recusa sem adicionar alternativas.

---

### 2.12 Fallback da Calculadora

**Durante fallback (banner):**
> "Modo básico — sem análise da IA"

**Explicação ao Cessionário:**
> "Cálculo realizado sem análise contextual da IA. Para análise completa, tente novamente em instantes."

**Dani desligada:**
> "A Analista de Oportunidades está temporariamente indisponível. Os cálculos de comissão e Escrow continuam disponíveis. Tente novamente em alguns instantes."

---

### 2.13 Rate Limit (Webchat)

**Bloqueio atingido:**
> "Você atingiu o limite de 30 mensagens por hora. Você poderá enviar a próxima mensagem em **[mm:ss]**."

*(Campo desabilitado visualmente com contador regressivo em tempo real.)*

---

### 2.14 SLA e Latência

**Digitando (indicador):**
> *(Três pontos pulsando — sem texto)*
> `aria-label="Dani está digitando"`

**Latência acima de 2× SLA:**
> "A Dani está demorando mais que o esperado. Você pode aguardar ou tentar novamente em instantes."
> [Botão] "Aguardar"
> [Botão] "Tentar novamente"

**Recusa total por falha de isolamento:**
> "O serviço de análise está temporariamente indisponível. Tente novamente em instantes."

---

### 2.15 WhatsApp — Vinculação (Fase 2)

**Etapa 1 — Instrução:**
> "Informe o número de WhatsApp que você quer vincular à sua conta."

**Número inválido:**
> "O número informado não parece ser válido. Verifique o DDD e tente novamente."

**Número já vinculado a outra conta:**
> "Este número já está associado a outra conta. Se acredita que há um erro, entre em contato com o suporte."

**OTP enviado:**
> "Enviamos um código de 6 dígitos para o número **[mascarado]**. O código expira em 15 minutos."

**OTP incorreto:**
> "O código informado não está correto."

**OTP expirado:**
> "O código expirou. Solicite um novo código para continuar."
> [Botão] "Reenviar código" (cooldown: "Reenviar em [ss]s")

**Rate limit OTP:**
> "Você esgotou as tentativas disponíveis. Tente novamente em 30 minutos."

**Confirmação WhatsApp — instrução:**
> "Enviamos uma mensagem de confirmação para o seu WhatsApp. Responda com o código para concluir a vinculação."

**Vinculação concluída:**
> "Seu WhatsApp foi vinculado com sucesso. Agora você pode usar a Dani pelo WhatsApp."

**Desvinculação — confirmação:**
> "Deseja desvincular o número **(xx) •••••-[últimos 4 dígitos]**? Você não poderá mais usar a Dani pelo WhatsApp."
> [Botão] "Confirmar desvinculação" | [Botão] "Cancelar"

**Desvinculação concluída:**
> "Seu WhatsApp foi desvinculado com sucesso."

---

## 3. Labels de Interface

| Elemento | Label |
|---|---|
| Botão CTA da tela de oportunidade | "Consultar Dani" |
| Título do chat | "Dani · Analista de Oportunidades" |
| Placeholder do input | "Digite sua mensagem..." |
| Botão enviar | "Enviar" (desktop) / ícone ▶ (mobile) |
| Botão fechar chat | "Fechar" (desktop) / ← (mobile) |
| FAB sem alertas | `aria-label="Abrir chat com Dani"` |
| FAB com alertas | `aria-label="Abrir chat com Dani — {N} alertas não lidos"` |
| Badge "Melhor opção" | "Melhor opção" |
| Seção Dashboard | "Oportunidades em Destaque" |

---

## 4. Formatação de Dados

| Tipo | Formato | Exemplo |
|---|---|---|
| Valor monetário | `R$ X.XXX,XX` | `R$ 30.000,00` |
| Percentual | `XX%` | `45%` |
| Score de risco | `N/10` | `3/10` |
| Código OPR | `OPR-XXXX-XXXX` | `OPR-2024-0042` |
| Telefone mascarado | `(XX) •••••-XXXX` | `(85) •••••-3210` |
| Tempo (contador) | `mm:ss` | `23:45` |

---

## Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. Guia completo de UX Writing: princípios, strings por componente, labels, formatação. Mensagens normativas identificadas. |
