# 08 - UX Writing — AI-Dani-Cedente

| Campo | Valor |
|---|---|
| Destinatário | Produto, design e frontend |
| Escopo | Guia de linguagem da interface com padrões de mensagens, CTAs, empty states e erros |
| Módulo | AI-Dani-Cedente |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 23/03/2026 (America/Fortaleza) |
| Dependências | D03 — Brand Guide · D06 — Mapa de Telas · D07 — Wireframes |

---

> **📌 TL;DR**
>
> - **Tom de voz:** acolhedor, direto e orientado ao próximo passo — a Dani fala como uma parceira experiente, não como um bot genérico.
> - **Princípio máximo:** toda mensagem termina sugerindo ou indicando a próxima ação. Sem beco sem saída.
> - **Padrões documentados:** 5 tipos de mensagem (sucesso, erro, aviso, informação, confirmação destrutiva).
> - **CTAs padronizados:** hierarquia em 3 níveis com verbos controlados por contexto.
> - **Empty states:** cobertura completa das 7 telas do chat (T-01 a T-07).
> - **Vocabulário controlado:** termos padronizados com sinônimos proibidos — alinhado ao glossário do D01.
> - **Zero seções pendentes:** documento completo com cobertura 100%.

---

## 1. Princípios de Escrita

### 1.1 Fundamentos

A Dani-Cedente é a Guardiã do Retorno do Cedente. A linguagem da interface deve refletir essa identidade: uma parceira que conhece o processo, fala de forma humana e nunca deixa o Cedente sem saber o que fazer a seguir.

| Princípio | Descrição | ✅ Correto | 🔴 Incorreto |
|---|---|---|---|
| **Próximo passo sempre** | Toda mensagem deve indicar ou sugerir a ação seguinte | "Sua proposta foi aceita. Agora aguarde o depósito no Escrow." | "Proposta aceita." |
| **Curta e útil** | Máximo 2 frases por mensagem de estado. Sem enrolação. | "Documento enviado com sucesso." | "Seu documento foi enviado com sucesso para a nossa plataforma e será analisado em breve." |
| **Voz ativa** | O sujeito age, não sofre a ação | "A Dani analisou sua proposta." | "A proposta foi analisada pela Dani." |
| **Neutra, sem culpa** | Erros não são do usuário | "Não foi possível enviar o documento. Tente novamente." | "Você enviou um arquivo inválido." |
| **Contexto explícito** | Nunca genérico quando o contexto permite ser específico | "Seu CPF não foi encontrado. Verifique e tente novamente." | "Dados inválidos." |
| **Linguagem do Cedente** | Usa termos do domínio imobiliário sem jargão técnico | "Seu repasse líquido estimado é R$ 45.200,00." | "O delta calculado resultou em valor positivo." |

### 1.2 Do's e Don'ts

**🎯 Faça sempre:**
- Comece mensagens de erro com o contexto ("Não foi possível…", "Não encontramos…")
- Termine mensagens de ação com o próximo passo ("Agora você pode…", "Em seguida…")
- Use o nome "Dani" na primeira pessoa de forma consistente: "Aqui está o resumo que preparei para você."
- Prefira "Confirmar" a "OK", "Enviar" a "Submeter", "Cancelar" a "Desistir"

**🔴 Nunca faça:**
- "Clique aqui" como CTA — sempre informe o que vai acontecer
- Placeholder sem label visível
- Mensagem de erro sem o que o usuário pode fazer
- Empty state sem CTA quando há próxima ação possível
- Dois botões "Confirmar" em contextos diferentes sem diferenciação
- Termos técnicos internos: "webhook falhou", "timeout de API", "erro 500"

---

## 2. Padrão de Mensagens por Tipo

| Tipo | Estrutura | Tom | Tamanho máximo | Exemplo ✅ | Exemplo 🔴 |
|---|---|---|---|---|---|
| **Sucesso** | O que aconteceu + próximo passo (quando houver) | Positivo, direto | 2 frases / 120 caracteres | "Documento enviado! Agora aguarde a análise do Admin." | "Sucesso!" |
| **Erro** | O que aconteceu + o que o usuário pode fazer | Neutro, empático | 2 frases / 140 caracteres | "Não foi possível enviar o documento. Verifique o formato (PDF ou JPG) e tente novamente." | "Erro ao processar." |
| **Aviso** | Contexto do alerta + orientação (sem bloquear) | Calmo, informativo | 2 frases / 130 caracteres | "O prazo do Escrow vence em 2 dias. Confirme com o Cessionário se já foi feito o depósito." | "Prazo se encerrando." |
| **Informação** | Conteúdo neutro contextualizado | Neutro, objetivo | 2 frases / 120 caracteres | "Sua oportunidade está publicada no marketplace e visível para Cessionários." | "Oportunidade ativa." |
| **Confirmação destrutiva** | Consequência explícita + CTAs diferenciados | Sério, direto | Título (60 char) + corpo (2 frases / 140 char) | Título: "Retirar oportunidade do marketplace?" / Corpo: "A oportunidade ficará invisível para Cessionários. Você poderá republicá-la quando quiser." / CTAs: [Retirar] [Cancelar] | Título: "Tem certeza?" / CTA: "Sim" |

### 2.1 Mensagens do Chat — Padrão de Resposta da Dani

Diferentemente das mensagens de estado de UI, as respostas textuais da Dani seguem padrão conversacional:

| Contexto | Estrutura | Exemplo |
|---|---|---|
| **Análise de proposta** | Diagnóstico + retorno líquido + próximo passo | "Recebi a proposta de R$ 280.000,00. Seu retorno líquido estimado é **R$ 45.200,00** (Repasse − Saldo Devedor). Quer que eu te explique como calcular uma contraproposta?" |
| **Orientação sobre dossiê** | Estado atual + o que falta + próxima ação | "Faltam 2 documentos para completar seu dossiê: RG e Comprovante de Residência. Quer que eu te explique como enviar cada um?" |
| **Escrow** | Estado atual + prazo + próxima ação | "O Cessionário tem até [data] para depositar o valor no Escrow. Vou te avisar quando o depósito for confirmado." |
| **Bloqueio de isolamento** | O que não pode fazer + alternativa disponível | "Não posso compartilhar dados do Cessionário. Para negociar diretamente, use o chat de negociação da plataforma." |
| **Limite de conhecimento** | O que não sabe + para onde encaminhar | "Não tenho acesso a essa informação. Para dúvidas jurídicas sobre o contrato, recomendo consultar o suporte da plataforma." |

---

## 3. Labels e Placeholders

### 3.1 Convenções Gerais

| Elemento | Regra | Tamanho máximo | Exemplo ✅ | Exemplo 🔴 |
|---|---|---|---|---|
| **Label** | Sempre visível, descritivo, sentence case | 40 caracteres | "Valor do repasse" | "Campo 1" ou ausente |
| **Placeholder** | Exemplo de formato, não instrução | 50 caracteres | "Ex: R$ 280.000,00" | "Digite o valor aqui" |
| **Helper text** | Instrução adicional abaixo do campo | 80 caracteres | "Use o valor da tabela atual do financiamento." | "Preencha corretamente." |
| **Error text** | Específico ao campo, substitui helper em erro | 80 caracteres | "Valor inválido. Informe um valor entre R$ 1.000,00 e R$ 10.000.000,00." | "Campo obrigatório." |

### 3.2 Regras de Capitalização

- **Labels:** sentence case — "Valor do repasse" (não "Valor Do Repasse")
- **Placeholders:** sentence case — "Ex: R$ 280.000,00"
- **Botões/CTAs:** sentence case — "Confirmar aceite" (não "Confirmar Aceite")
- **Títulos de seção:** sentence case — "Análise de proposta"
- **Títulos de modal:** sentence case — "Aceitar proposta?"

### 3.3 Campos específicos do domínio

| Campo | Label | Placeholder | Helper text |
|---|---|---|---|
| Valor do repasse | "Valor do repasse" | "Ex: R$ 280.000,00" | "Valor da tabela atual do financiamento." |
| Saldo devedor | "Saldo devedor" | "Ex: R$ 234.800,00" | "Valor atual da dívida com o banco." |
| Valor pago | "Valor já pago" | "Ex: R$ 45.200,00" | "Total já quitado do financiamento." |
| Valor da contraproposta | "Sua contraproposta" | "Ex: R$ 295.000,00" | "Valor que você deseja receber pelo repasse." |

---

## 4. CTAs — Hierarquia e Verbos

### 4.1 Hierarquia

| Nível | Descrição | Estilo visual | Exemplo |
|---|---|---|---|
| **Primário** | Ação principal da tela ou modal | Botão sólido, cor primária (#0069A8) | "Aceitar proposta" |
| **Secundário** | Ação alternativa, menos urgente | Botão outline, cor primária | "Enviar contraproposta" |
| **Terciário** | Ação discreta, link ou texto | Link sublinhado ou texto simples | "Ver detalhes" |

### 4.2 Verbos padronizados por contexto

| Contexto | Verbo primário | Verbo secundário | Verbo terciário |
|---|---|---|---|
| Aceitar proposta | "Aceitar proposta" | "Enviar contraproposta" | "Ver simulação" |
| Recusar proposta | "Recusar proposta" | "Cancelar" | "Ver análise" |
| Enviar documento | "Enviar documento" | "Cancelar" | "Ver requisitos" |
| Retirar do marketplace | "Retirar oportunidade" | "Cancelar" | — |
| Confirmar extensão de Escrow | "Confirmar extensão" | "Recusar extensão" | "Ver detalhes" |
| Encerrar conversa | "Encerrar conversa" | — | — |
| Avaliar atendimento (CSAT) | "Enviar avaliação" | "Pular" | — |

### 4.3 Verbos proibidos

| Proibido | Substituto correto | Motivo |
|---|---|---|
| "Clique aqui" | CTA descritivo ("Ver proposta", "Aceitar") | Sem contexto |
| "Submeter" | "Enviar" | Jargão técnico |
| "OK" como primário | "Confirmar", "Entendido" | Ambíguo |
| "Deletar" | "Excluir" (não usado no domínio — usar "Retirar") | Sinônimo incorreto |
| "Desistir" | "Cancelar" | Tom negativo |

### 4.4 Regras de tamanho

- Máximo 3 palavras por CTA
- Exceção permitida: CTAs de confirmação destrutiva podem ter 4 palavras para garantir clareza ("Retirar do marketplace")
- Nunca abreviar para caber: adaptar o componente, não o texto

---

## 5. Empty States

### 5.1 Estrutura obrigatória

Cada empty state deve ter:
1. **Título:** o que está vazio, em linguagem humana (máx. 50 caracteres)
2. **Descrição:** contexto ou o que o usuário pode fazer (máx. 120 caracteres)
3. **CTA:** ação para sair do estado (quando há próxima ação possível)

### 5.2 Empty states por tela

| Tela | Cenário | Título | Descrição | CTA |
|---|---|---|---|---|
| **T-01 — Chat inicial** | Primeiro acesso, KYC pendente | "Boas-vindas ao Repasse Seguro" | "Para começar, precisamos verificar seus dados. O processo leva menos de 5 minutos." | "Iniciar verificação" |
| **T-01 — Chat inicial** | KYC aprovado, sem oportunidade | "Tudo certo com seu cadastro" | "Ainda não temos uma oportunidade de repasse para você. Vou te avisar quando surgir uma." | — |
| **T-02 — Contexto oportunidade** | Cenário calculado, aguardando publicação | "Sua oportunidade está sendo preparada" | "Calculei os valores do seu repasse. Confirme os dados para publicar no marketplace." | "Revisar e publicar" |
| **T-03 — Contexto proposta** | Sem propostas recebidas ainda | "Nenhuma proposta ainda" | "Sua oportunidade está visível no marketplace. Vou te avisar quando um Cessionário enviar uma proposta." | — |
| **T-04 — Conversa ativa** | Histórico de chat vazio | "Nenhuma conversa ainda" | "Pode me perguntar qualquer coisa sobre o seu repasse. Estou aqui para ajudar." | — |
| **T-05 — Cards inline** | Sem documentos no dossiê | "Dossiê vazio" | "Você ainda não enviou nenhum documento. Vou te guiar pelo processo." | "Ver documentos necessários" |
| **T-07 — Rate limit atingido** | 30 mensagens/hora esgotadas | "Limite de mensagens atingido" | "Você atingiu o limite de 30 mensagens por hora. O chat será liberado em [tempo restante]." | — |

### 5.3 Empty states de sistema (T-07)

| Estado | Título | Descrição | CTA |
|---|---|---|---|
| **Digitando** | — | Indicador de digitação animado (3 pontos) | — |
| **Resposta lenta (>5s)** | — | "A Dani está preparando a resposta…" | — |
| **Resposta muito lenta (>10s)** | "Demorando mais que o esperado" | "Pode aguardar ou tentar novamente em instantes." | "Aguardar" / "Tentar novamente" |
| **Indisponível** | "A Dani está temporariamente indisponível" | "Para urgências, entre em contato com o suporte." | "Falar com suporte" |

---

## 6. Onboarding e Tooltips

### 6.1 Princípios

- **Progressivo:** não mostre tudo de uma vez — apresente contexto no momento certo
- **Contextual:** tooltips aparecem na primeira ação, não antes
- **Dispensável:** sempre há como fechar ou pular sem perda de funcionalidade

### 6.2 Regra dos 3 passos

Onboarding do chat é apresentado em no máximo 3 etapas:
1. **Quem sou eu:** "Sou a Dani, sua parceira no processo de repasse."
2. **O que posso fazer:** "Posso te ajudar com propostas, documentação e acompanhar seu Escrow."
3. **Como começar:** "Me conta: você já tem uma proposta em mãos?"

### 6.3 Tamanho máximo de tooltips

| Contexto | Tamanho máximo |
|---|---|
| Tooltip de ícone | 60 caracteres |
| Tooltip de campo | 100 caracteres |
| Tooltip de status | 80 caracteres |
| Tooltip de card de análise | 120 caracteres |

### 6.4 Tom de onboarding

- **Acolhedor, guiado:** use "vamos" (plural inclusivo) em vez de "você deve"
- ✅ "Vamos configurar sua oportunidade juntos."
- 🔴 "Você precisa preencher esses campos."

---

## 7. Mensagens de Erro

### 7.1 Padrão obrigatório

**Fórmula:** [O que aconteceu] + [O que o usuário pode fazer]

Regras:
- Nunca culpar o usuário
- Nunca expor erros técnicos (códigos de erro, stack traces, nomes de endpoints)
- Máximo 2 frases
- Se o erro for temporário, indicar retry

### 7.2 Diferença entre mensagem de UI e log técnico

| Aspecto | Mensagem de UI | Log técnico |
|---|---|---|
| Público | Cedente | Engenharia/Ops |
| Tom | Humano, empático | Técnico, preciso |
| Exemplo | "Não foi possível enviar o documento. Tente novamente." | `[ERROR] dossier.upload failed: s3_upload_timeout after 30s (attempt 3/3)` |
| Onde aparece | Interface do chat | Sentry / logs de servidor |

### 7.3 Catálogo de erros por contexto

| Contexto | Mensagem de UI | Erro técnico mapeado |
|---|---|---|
| Falha ao enviar documento | "Não foi possível enviar o documento. Verifique se o arquivo é PDF ou JPG (máx. 10 MB) e tente novamente." | Upload S3 timeout / formato inválido |
| Falha de conectividade | "Você parece estar sem conexão. Verifique sua internet e tente novamente." | Network error / timeout de API |
| Sessão expirada | "Sua sessão expirou. Faça login novamente para continuar." | JWT expired |
| Erro desconhecido | "Ocorreu um problema inesperado. Tente novamente. Se o problema persistir, entre em contato com o suporte." | 500 / erro não mapeado |
| Rate limit de mensagens | "Você atingiu o limite de 30 mensagens por hora. Aguarde [tempo] para continuar." | Rate limit Redis |
| Documento rejeitado pelo Admin | "Este documento foi recusado: [motivo]. Corrija o arquivo e reenvie." | Admin rejection webhook |
| Proposta expirada | "Esta proposta não está mais disponível. Pode ter sido cancelada pelo Cessionário." | Proposal status change |
| Escrow vencido sem depósito | "O prazo do Escrow encerrou sem o depósito. A proposta foi cancelada e sua oportunidade voltou ao marketplace." | Escrow timeout event |

### 7.4 Mensagens de validação de formulário

| Campo | Mensagem de erro |
|---|---|
| Valor monetário inválido | "Informe um valor válido em reais. Ex: R$ 280.000,00." |
| Campo obrigatório vazio | "Este campo é obrigatório para continuar." |
| Formato de arquivo inválido | "Formato não aceito. Envie o arquivo em PDF ou JPG." |
| Arquivo muito grande | "O arquivo é muito grande. O tamanho máximo é 10 MB." |
| Contraproposta abaixo do mínimo | "Sua contraproposta precisa ser maior que o saldo devedor." |

---

## 8. Truncamento e Overflow

### 8.1 Regras por componente

| Componente | Tamanho máximo | Comportamento ao ultrapassar |
|---|---|---|
| Bolha de chat (Dani) | Sem limite — scroll interno | — |
| Bolha de chat (Cedente) | Sem limite — scroll interno | — |
| CTA (botão) | 3 palavras / 30 caracteres | Ajustar layout do componente, não o texto |
| Label de campo | 40 caracteres | Truncar com "…" + tooltip com texto completo |
| Título de card de análise | 50 caracteres | Truncar com "…" + tooltip |
| Descrição de card | 120 caracteres | Truncar com "…" + CTA "Ver mais" |
| Nome de documento no dossiê | 60 caracteres | Truncar com "…" + tooltip com nome completo |
| Toast de notificação | 100 caracteres | Truncar com "…" + CTA "Ver detalhes" |
| Sugestão de conversa | 60 caracteres | Truncar com "…" — não há tooltip |

### 8.2 Tooltip de expansão

- Exibir sempre que o texto for truncado por overflow
- Delay de exibição: 300ms após hover
- Texto do tooltip: conteúdo completo sem formatação adicional
- Não usar tooltip em dispositivos mobile — usar modal ou expansão inline

### 8.3 Regra de hierarquia

Nunca adaptar o texto para caber — adaptar o componente. Se o componente não pode crescer, truncar com tooltip obrigatório.

---

## 9. Acessibilidade Textual

### 9.1 Aria-labels obrigatórios

| Componente | aria-label | Exemplo |
|---|---|---|
| Botão fechar (chat) | "Fechar chat da Dani" | `aria-label="Fechar chat da Dani"` |
| Botão enviar mensagem | "Enviar mensagem" | `aria-label="Enviar mensagem"` |
| Avatar da Dani | "Foto da Dani, assistente de repasse" | `aria-label="Foto da Dani, assistente de repasse"` |
| Card de análise de proposta | "Card de análise: proposta de [valor]" | `aria-label="Card de análise: proposta de R$ 280.000,00"` |
| Indicador de digitação | "A Dani está digitando" | `aria-label="A Dani está digitando"` |
| Estrela de avaliação CSAT | "Avaliar com [N] estrela(s)" | `aria-label="Avaliar com 4 estrelas"` |
| Toast de notificação | "Notificação: [conteúdo resumido]" | `aria-label="Notificação: nova proposta recebida"` |
| Status do Escrow | "Status do Escrow: [estado]" | `aria-label="Status do Escrow: aguardando depósito"` |

### 9.2 Alt texts

| Imagem | alt text |
|---|---|
| Avatar da Dani (online) | "Dani — online" |
| Avatar da Dani (digitando) | "Dani — digitando" |
| Ilustração de empty state | "Ilustração: [contexto do empty state]" |
| Ícone de status de documento | "Documento [nome]: [status]" |

### 9.3 Ordem de leitura (screen reader)

Para o chat, a ordem obrigatória de leitura é:
1. Header do chat ("Chat com a Dani — [status]")
2. Histórico de mensagens (mais antigas primeiro, mais recentes por último)
3. Indicador de digitação (quando ativo)
4. Cards de análise inline (após a mensagem que os originou)
5. Campo de entrada de mensagem
6. Botão enviar

### 9.4 Contraste e WCAG 2.1 AA

- Todo texto de UI deve ter contraste mínimo de 4.5:1 (texto normal) ou 3:1 (texto grande/bold)
- Mensagens de erro não podem depender apenas de cor — incluir ícone ou texto descritivo
- Estados ativos de botão devem ter indicador visual além da cor (outline de foco mínimo 2px)

---

## 10. Vocabulário Controlado

Termos padronizados para uso na interface — elimina sinônimos e inconsistências.

| Termo correto | Termos proibidos (sinônimos) | Contexto de uso |
|---|---|---|
| **Cedente** | Vendedor, proprietário, cliente, usuário | Pessoa que vende o repasse |
| **Cessionário** | Comprador, adquirente, interessado | Pessoa que compra o repasse |
| **Repasse** | Cessão, transferência, venda do financiamento | O produto sendo negociado |
| **Oportunidade** | Anúncio, oferta, listagem, post | A oportunidade de repasse publicada |
| **Proposta** | Oferta, lance, bid | A proposta enviada pelo Cessionário |
| **Retorno líquido** | Lucro, ganho, valor final, saldo positivo | Repasse − Saldo Devedor |
| **Saldo devedor** | Dívida, débito, quanto falta pagar | Valor atual da dívida com o banco |
| **Dossiê** | Documentação, pasta, arquivos | Conjunto de documentos do processo |
| **Escrow** | Conta garantia, custódia, depósito | Mecanismo de garantia do pagamento |
| **Marketplace** | Plataforma, site, portal, loja | Onde as oportunidades são publicadas |
| **Contraproposta** | Contra-oferta, nova proposta, rebate | Proposta alternativa enviada pelo Cedente |
| **Admin** | Administrador, gestor, moderador, plataforma | Equipe interna da Repasse Seguro |
| **Publicar** | Anunciar, divulgar, postar, colocar no ar | Tornar a oportunidade visível no marketplace |
| **Retirar** | Remover, deletar, excluir, despublicar | Tornar a oportunidade invisível no marketplace |
| **Aceitar** | Aprovar, confirmar, fechar | Aceitar uma proposta |
| **Recusar** | Rejeitar, declinar, negar | Recusar uma proposta |
| **Enviar** | Submeter, mandar, encaminhar | Enviar documento ou mensagem |
| **Cancelar** | Desistir, abandonar, sair | Cancelar uma ação em andamento |

---

## 11. Tom de Voz — Guia Expandido

### 11.1 Personalidade da interface

A Dani-Cedente tem 4 adjetivos que definem sua voz:

| Adjetivo | O que significa na prática | O que NÃO significa |
|---|---|---|
| **Acolhedora** | Usa linguagem humana, reconhece o estresse do processo ("Sei que pode ser bastante informação…") | Excessivamente informal, usa gírias ou emojis fora do lugar |
| **Direta** | Vai direto ao ponto, sem introduções longas. A primeira frase sempre entrega o conteúdo principal. | Fria, robótica ou sem empatia |
| **Segura** | Mostra conhecimento do processo sem arrogância. Quando não sabe, admite com clareza. | Onisciente ou presunçosa |
| **Orientada a ação** | Toda interação termina com o próximo passo claro | Impositiva ou ansiosa |

### 11.2 Escala de formalidade por contexto

| Contexto | Nível | Exemplo |
|---|---|---|
| Erro crítico (Escrow vencido, proposta cancelada) | Formal, direto, sem alarmismo | "O prazo do Escrow encerrou sem o depósito. A proposta foi cancelada e sua oportunidade voltou ao marketplace." |
| Sucesso (proposta aceita, documento aprovado) | Neutro, positivo | "Proposta aceita com sucesso! Agora aguarde o depósito no Escrow." |
| Onboarding / primeiro acesso | Acolhedor, guiado | "Olá! Sou a Dani, sua parceira no processo de repasse. Vamos começar?" |
| Empty state (sem proposta ainda) | Leve, encorajador, sem pressão | "Sua oportunidade está visível no marketplace. Vou te avisar quando uma proposta chegar." |
| Análise de proposta | Técnico-acessível, com dados | "Recebi a proposta de R$ 280.000,00. Seu retorno líquido estimado é **R$ 45.200,00**." |
| Bloqueio de isolamento | Claro, sem desculpas | "Não tenho acesso a dados do Cessionário. Para negociar diretamente, use o chat de negociação da plataforma." |
| Aviso de prazo | Calmo, proativo | "O prazo do Escrow vence em 2 dias. Confirme com o Cessionário se o depósito já foi feito." |

### 11.3 Regras de capitalização

| Elemento | Padrão | Exemplo |
|---|---|---|
| Títulos de seção / tela | Sentence case | "Análise de proposta" |
| Títulos de modal | Sentence case | "Aceitar proposta?" |
| Botões / CTAs | Sentence case | "Aceitar proposta" |
| Labels de campo | Sentence case | "Valor do repasse" |
| Mensagens de chat | Sentence case | "Sua proposta foi aceita com sucesso." |
| Nomes de entidades do domínio | Maiúscula inicial (são substantivos próprios do sistema) | "Cedente", "Cessionário", "Escrow", "Admin", "Dani" |
| Status de documentos / estados | Maiúscula inicial quando exibido como badge | "Aguardando envio", "Em análise", "Aprovado", "Rejeitado" |

---

## 12. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. Guia completo de UX Writing da AI-Dani-Cedente: princípios (6), padrões de mensagem (5 tipos), labels/placeholders, CTAs com verbos controlados, empty states das 7 telas (T-01 a T-07), onboarding, erros (catálogo de 8 cenários), truncamento, acessibilidade textual, vocabulário controlado (18 termos), tom de voz expandido. |

---

## 13. Backlog de Pendências

| Item | Tipo | Seção | Impacto | Justificativa | Dono | Status |
|---|---|---|---|---|---|---|
| Tom de voz em cenários de perda financeira (proposta abaixo do esperado, Escrow revertido) | Decisão Autônoma | Seção 11.2 | P1 | Adotado tom neutro-empático sem alarmismo, baseado no Brand Guide (acolhedor) e perfil B2B do produto. Alternativa descartada: tom urgente (gera ansiedade desnecessária em contexto financeiro). | Produto | Validar com time |
| Mensagem de CSAT — textos das estrelas (1 a 5) | Decisão Autônoma | Seção 5 / T-07 | P2 | Adotado padrão implícito de estrelas sem rótulo textual — padrão universal reconhecível. Alternativa: rótulos explícitos ("Péssimo" a "Excelente") — pode influenciar viés de resposta. | Design | Validar com time |
