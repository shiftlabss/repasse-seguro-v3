# Repasse AI
## 08 — UX Writing

| Campo | Valor |
|---|---|
| **Destinatário** | Produto, Design e Frontend |
| **Escopo** | Guia de linguagem da interface com padrões de mensagens, CTAs, empty states e erros |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data da versão** | 22/03/2026 00:00 (America/Fortaleza) |
| **Status** | Ativo |
| **Fase** | 2 — Produto |
| **Área** | Design + Frontend |
| **Referências** | 03 - Brand Theme Guide · 06 - Mapa de Telas · 07 - Wireframes |

---

> 📌 **TL;DR**
>
> - **5 princípios de escrita** — voz direta, orientada a ação, empática sem floreio.
> - **Tom de voz:** profissional-acessível. O agente é um analista especialista, não um chatbot genérico.
> - **5 tipos de mensagem** documentados — sucesso, erro, aviso, informação, confirmação destrutiva.
> - **38 CTAs padronizados** por contexto e hierarquia.
> - **Empty states** cobertos para todas as 22 telas do Mapa de Telas.
> - **Vocabulário controlado:** 18 termos padronizados com sinônimos proibidos.
> - **Seções pendentes:** 0 — cobertura 100%.

---

## 1. Princípios de Escrita

🎯 **Estes princípios se aplicam a toda mensagem da interface — do FAB ao toast de erro.**

### 1.1 Os 5 Princípios

**P1 — Direta, sem enrolação**
Cada mensagem faz uma coisa. Sem introduções, sem justificativas longas. O usuário está no fluxo de trabalho.
- ✅ "Código expirado. Solicite um novo."
- 🔴 "Infelizmente o código que você inseriu no campo de verificação acabou de expirar. Para continuar o processo, você precisará solicitar o envio de um novo código."

**P2 — Orientada ao próximo passo**
Toda mensagem que não é puramente informativa deve indicar o que o usuário pode fazer.
- ✅ "WhatsApp desvinculado. Você pode vincular outro número a qualquer momento."
- 🔴 "Operação concluída."

**P3 — Empática sem culpar**
Erros são do sistema, nunca do usuário. Nenhuma mensagem usa "você errou", "dados inválidos" ou equivalentes.
- ✅ "O código não está correto. Verifique o SMS e tente novamente."
- 🔴 "Você inseriu o código errado."

**P4 — Específica, nunca genérica**
Se o contexto é conhecido, use-o. Nunca use "Erro ao processar." quando o contexto é disponível.
- ✅ "Não foi possível enviar o código para +55 (11) 98765-4321. Verifique o número e tente novamente."
- 🔴 "Erro ao processar."

**P5 — Consistente**
O mesmo conceito tem o mesmo nome em todo o produto. Se o documento diz "Vincular WhatsApp", o botão diz "Vincular WhatsApp" — não "Conectar número" ou "Ativar WhatsApp".

---

## 2. Padrão de Mensagens por Tipo

| Tipo | Estrutura | Tom | Tamanho máximo | Exemplo ✅ | Exemplo 🔴 |
|---|---|---|---|---|---|
| **Sucesso** | Confirmação + próximo passo (quando aplicável) | Neutro, positivo | 80 caracteres | "WhatsApp vinculado com sucesso." | "Operação realizada." |
| **Erro** | O que aconteceu + o que fazer. Máximo 2 frases. | Neutro, empático | 140 caracteres | "Não foi possível carregar o histórico. Tente novamente." | "Erro ao carregar." |
| **Aviso** | Alerta com contexto + implicação (sem bloqueio) | Neutro, informativo | 120 caracteres | "Código expira em 2 minutos. Insira antes que expire." | "Atenção." |
| **Informação** | Dado contextual relevante, sem ação obrigatória | Neutro | 100 caracteres | "Suas mensagens ficam salvas por 90 dias." | "Informação sobre dados." |
| **Confirmação destrutiva** | Consequência explícita + CTA confirmação/cancelamento | Neutro, preciso | 160 caracteres (corpo) | "Ao desvincular, você não receberá mais alertas pelo WhatsApp. Esta ação pode ser desfeita." | "Tem certeza?" |

### 2.1 Padrão de Erro (detalhe)

Estrutura obrigatória: **[O que aconteceu] + [O que o usuário pode fazer]**

```
✅ "Não foi possível [ação]. [Sugestão de próximo passo]."
✅ "O [elemento] não está [estado esperado]. [Próximo passo]."
```

Sempre em primeira pessoa do sistema ("não foi possível", "não conseguimos") — nunca em segunda pessoa acusatória.

---

## 3. Labels e Placeholders

### 3.1 Regras Gerais

| Elemento | Regra | Tamanho máximo | Exemplo ✅ | Exemplo 🔴 |
|---|---|---|---|---|
| **Label** | Sempre visível, descritivo, conciso. Sentence case. | 40 caracteres | "Número do WhatsApp" | "Campo 1" |
| **Placeholder** | Exemplo de formato, nunca instrução. Desaparece ao digitar. | 50 caracteres | "+55 (11) 98765-4321" | "Digite seu número aqui" |
| **Helper text** | Instrução adicional abaixo do campo, sempre visível. | 80 caracteres | "Use o número com DDD. Ex: (11) 98765-4321" | "Preencha corretamente." |
| **Error text** | Específico ao campo. Substitui helper text em erro. | 100 caracteres | "O número não parece ser válido. Verifique o DDD." | "Número inválido." |

### 3.2 Formulários por Tela

**T-015 — Seção WhatsApp (campo de número)**
- Label: `Número do WhatsApp`
- Placeholder: `+55 (11) 98765-4321`
- Helper text: `Com DDD. Exemplo: (11) 98765-4321`
- Error (formato inválido): `O número não parece ser válido. Verifique o DDD e tente novamente.`
- Error (número já vinculado): `Este número já está associado a outra conta. Entre em contato com o suporte se acredita que há um erro.`

**T-016 — OTP SMS (campos de 6 dígitos)**
- Label: `Código de verificação`
- Placeholder: individual por dígito: `—` (traço)
- Helper text: `Enviamos um código de 6 dígitos para o número informado.`
- Error (código incorreto): `O código não está correto. Verifique o SMS e tente novamente.`
- Error (código expirado): `O código expirou. Solicite um novo para continuar.`
- Error (bloqueio): `Muitas tentativas. Aguarde [mm:ss] para tentar novamente.`

**T-021 — Campo de motivo de takeover**
- Label: `Motivo do takeover`
- Placeholder: `Descreva o motivo da intervenção...`
- Helper text: `Obrigatório. Registrado no histórico da interação.`
- Error: `O motivo é obrigatório para iniciar o takeover.`

---

## 4. CTAs

### 4.1 Hierarquia

| Hierarquia | Tipo visual | Regra | Limite de palavras |
|---|---|---|---|
| **Primário** | Button default (`--primary`) | Ação principal da tela. Verbo no infinitivo. Sempre presente. | Máximo 3 palavras |
| **Secundário** | Button outline | Ação alternativa. Verbo no infinitivo. | Máximo 3 palavras |
| **Terciário** | Link ou button ghost | Ação discreta, navegação de suporte. | Máximo 4 palavras |

### 4.2 Verbos Padronizados por Contexto

| Contexto | CTA Primário | CTA Secundário | CTA Terciário |
|---|---|---|---|
| Salvar configuração | Salvar | Cancelar | — |
| Vinculação WhatsApp step 1 | Enviar código | Cancelar | — |
| Vinculação WhatsApp step 2 | Verificar | Reenviar código | — |
| Confirmação de takeover | Confirmar takeover | Cancelar | — |
| Encerrar takeover | Encerrar takeover | — | — |
| Desvincular WhatsApp | Desvincular | Cancelar | — |
| Apagar histórico de chat | Confirmar exclusão | Cancelar | — |
| Tentar novamente (erro genérico) | Tentar novamente | Aguardar | — |
| Ver detalhes de oportunidade | Ver oportunidade | — | — |
| Re-verificar WhatsApp | Re-verificar agora | — | — |
| Navegação de volta | Voltar | — | — |
| Exportar dados Admin | Exportar | — | — |
| Carregar mais mensagens | Carregar mais | — | — |
| Fechar modal | — | Fechar | — |
| Ver política de privacidade | — | Ver política | — |
| Aceitar consentimento LGPD | Aceitar | Ver política | — |
| Abrir chat | Abrir chat | — | — |
| Ver documentação (Admin) | Ver documentação | — | — |

### 4.3 Verbos Proibidos como CTA Primário

- ❌ "Clique aqui" — sem contexto
- ❌ "Submeter" — anglicismo desnecessário (usar "Enviar")
- ❌ "OK" — sem contexto de ação
- ❌ "Continuar" sem objeto — ambíguo (usar "Enviar código", "Verificar", etc.)
- ❌ "Sim" / "Não" — sem contexto da ação (usar "Confirmar exclusão" / "Cancelar")

---

## 5. Empty States

### 5.1 Estrutura Obrigatória

1. **Título:** o que está vazio, em linguagem humana. Máximo 6 palavras.
2. **Descrição:** por que está vazio ou o que o usuário pode fazer. Máximo 2 frases.
3. **CTA:** ação para sair do empty state (quando houver próximo passo disponível).
4. **Tom:** leve, encorajador para estados de primeiro uso; neutro para filtros sem resultado.

### 5.2 Empty States por Tela

**T-003 — Chat (sem histórico)**
- Título: `Olá! Sou seu Analista de Oportunidades.`
- Descrição: `Posso analisar oportunidades, comparar repasses e calcular simulações. Como posso ajudar?`
- CTA: chips de sugestão clicáveis: `[Analisar uma OPR]` `[Comparar oportunidades]` `[Simular proposta]`

**T-019 — Histórico de Chat (sem conversas)**
- Título: `Nenhuma conversa registrada ainda.`
- Descrição: `Suas conversas com o Analista de Oportunidades aparecerão aqui.`
- CTA: `Abrir chat`

**T-020 — Supervisão IA (sem interações)**
- Título: `Nenhuma interação registrada.`
- Descrição: `Tente ajustar o período ou os filtros aplicados.`
- CTA: sem CTA — painel passivo.

**T-022 — Dashboard Admin (sem dados no período)**
- Título: `Sem dados para o período selecionado.`
- Descrição: `Tente um período diferente ou aguarde novas interações.`
- CTA: sem CTA.

**T-022 — Dashboard Admin (first-use — zero interações)**
- Título: `Ainda não há interações registradas.`
- Descrição: `As métricas aparecerão aqui após os primeiros usos do Repasse AI.`
- CTA: `Ver documentação`

**T-015 — Seção WhatsApp (NaoVinculado)**
- Título: `WhatsApp não vinculado.`
- Descrição: `Vincule seu número para receber alertas de oportunidades, prazos e status pelo WhatsApp.`
- CTA: `Vincular WhatsApp`

**T-018 — Seção Notificações (estado padrão)**
- Não tem empty state — sempre 3 toggles visíveis.

**T-003 — Chat (histórico com "Carregar mais")**
- Texto do botão: `Carregar conversas anteriores`
- Feedback durante carregamento: skeleton de 3 linhas (sem texto)

**T-004 / T-005 / T-006 / T-007 / T-008 — Bolhas**
- Não têm empty state próprio — bolhas só aparecem quando há resposta.

---

## 6. Onboarding e Tooltips

### 6.1 Tom de Onboarding

Tom: acolhedor, guiado, sem intimidar. Máximo 2 frases por step.

**Mensagem de boas-vindas (T-003 — primeiro acesso após LGPD)**:
> "Olá! Sou seu Analista de Oportunidades. Posso analisar repasses, comparar propostas e calcular retornos. Como posso começar?"

**Mensagem de boas-vindas (WhatsApp — primeira mensagem após vinculação)**:
> "Olá! Seu WhatsApp foi vinculado ao Repasse AI. A partir de agora você pode me consultar por aqui. Envie o código de uma oportunidade para começar."

### 6.2 Tooltips

- Tamanho máximo: 100 caracteres.
- Tom: informativo e neutro.
- Sempre justificam por que uma ação está desabilitada.

| Elemento | Tooltip |
|---|---|
| FAB offline | `Sem conexão. O chat voltará quando a internet for restabelecida.` |
| Botão "Vincular WhatsApp" offline | `Sem conexão. A vinculação requer internet.` |
| Botão "Assumir conversa" desabilitado (confiança acima do threshold) | `Confiança acima do limite configurado para takeover.` |
| Toggle de notificação offline | `Sem conexão. Tente novamente quando a internet voltar.` |
| Botão "Reenviar código" (cooldown) | `Aguarde [ss]s para reenviar.` |

### 6.3 Step Progress Labels (T-016)

| Step | Label ativo | Label concluído |
|---|---|---|
| Step 1 | `Inserir número` | `Número confirmado ✓` |
| Step 2 | `Código SMS` | `Código verificado ✓` |
| Step 3 | `Confirmar no WhatsApp` | `WhatsApp confirmado ✓` |

---

## 7. Mensagens de Erro

### 7.1 Padrão Universal

```
Estrutura: [O que aconteceu, em voz passiva/neutra] + [O que o usuário pode fazer]
Tom: neutro, empático, sem culpar
Comprimento: máximo 140 caracteres
```

### 7.2 Diferença entre Mensagem de UI e Log Técnico

| Tipo | Audiência | Exemplo |
|---|---|---|
| **Mensagem de UI** | Usuário final — clara, acionável, sem jargão | "Não foi possível carregar suas mensagens. Tente novamente." |
| **Log técnico** | Dev/Sentry/Langfuse — detalhe técnico completo | `[ERROR] ChatService.getHistory: Connection timeout after 5000ms (conv_id: uuid, user_id: uuid)` |

**Regra:** nunca mostrar logs técnicos na interface. Nunca mostrar stack traces. Códigos de erro só em tooltip de suporte técnico.

### 7.3 Catálogo de Mensagens de Erro

**Conectividade:**
- Offline genérico: `Você está offline. Verifique sua conexão e tente novamente.`
- Timeout de rede: `A conexão demorou mais que o esperado. Tente novamente.`

**Chat — Repasse AI:**
- Agente indisponível (timeout): `O Analista está demorando mais que o esperado. Você pode aguardar ou tentar novamente.`
- Stream interrompido: `Resposta incompleta. Tente novamente para receber a análise completa.`
- Dado bloqueado (restrito de negócio): `Não tenho acesso a essas informações. Posso ajudar com análise de oportunidades disponíveis no marketplace.`
- Rate limit: `Você atingiu o limite de mensagens por hora. Você poderá enviar novas mensagens em [hh:mm].`
- OPR não encontrado: `Não encontrei a oportunidade [OPR-XXXX]. Verifique o código e tente novamente.`

**Vinculação WhatsApp:**
- Número inválido: `O número não parece ser válido. Verifique o DDD e tente novamente.`
- Número já vinculado: `Este número já está associado a outra conta. Entre em contato com o suporte se acredita que há um erro.`
- OTP incorreto: `O código não está correto. Verifique o SMS e tente novamente.`
- OTP expirado: `O código expirou. Solicite um novo para continuar.`
- OTP bloqueado: `Muitas tentativas. Aguarde [mm:ss] para tentar novamente.`
- Confirmação WhatsApp expirada: `A confirmação pelo WhatsApp não foi concluída. Inicie o processo novamente quando quiser vincular seu número.`
- Erro ao desvincular: `Não foi possível desvincular o número. Tente novamente.`

**Painel Admin:**
- Erro ao carregar interações: `Não foi possível carregar as interações. Tente novamente.`
- Erro ao iniciar takeover: `Não foi possível assumir a conversa. Tente novamente.`
- Takeover já ativo (concorrência): `Esta conversa já está sendo controlada por outro membro da equipe.`

**Configurações:**
- Erro ao salvar preferência: `Não foi possível salvar sua preferência. Tente novamente.`
- Erro ao carregar histórico: `Não foi possível carregar o histórico. Tente novamente.`
- Erro ao excluir histórico: `Não foi possível solicitar a exclusão. Tente novamente.`

**LGPD:**
- Erro ao registrar consentimento: `Não foi possível registrar seu consentimento. Tente novamente.`

---

## 8. Truncamento e Overflow

### 8.1 Regras por Componente

| Componente | Regra | Limite | Tratamento |
|---|---|---|---|
| Pergunta do Cessionário na listagem Admin (T-020) | Truncar com ellipsis | 80 caracteres | Tooltip ao hover com texto completo |
| Primeira mensagem na lista de histórico (T-019) | Truncar com ellipsis | 60 caracteres | Nenhum — item clicável abre conversa |
| Badge numérica do FAB (T-001) | Truncar com símbolo | > 9 → "9+" | Sem tooltip |
| Nome em UserAvatar (sidebar) | Truncar com ellipsis | 24 caracteres | Tooltip ao hover |
| Texto de OportunidadeCard (T-012 a T-014) | Truncar com ellipsis | 40 caracteres (título) | Tooltip ao hover |
| Label de configuração Admin (T-022) | Sem truncamento | — | Quebra de linha |
| Mensagens do chat (T-003) | Sem truncamento | — | Scroll vertical |

### 8.2 Overflow em Tabelas (T-005, T-020)

- Células longas: ellipsis + tooltip ao hover.
- Números: sem ellipsis — se não cabe, reduzir tamanho da fonte (mín 12px).
- Datas: sempre formatadas como `dd/mm/aaaa hh:mm`.

---

## 9. Acessibilidade Textual

### 9.1 Aria-Labels

| Elemento | Aria-label |
|---|---|
| FAB do Chat | `Abrir Analista de Oportunidades` |
| Badge numérica do FAB | `[N] alertas não lidos` |
| Botão fechar modal | `Fechar` |
| Toggle de notificação ativo | `[Nome da notificação] ativada. Toque para desativar.` |
| Toggle de notificação inativo | `[Nome da notificação] desativada. Toque para ativar.` |
| Score de risco | `Score de risco: [Baixo / Moderado / Alto]` |
| Badge de confiança Admin | `Confiança: [valor]% — [Alta / Média / Baixa]` |
| Campos OTP | `Dígito [N] de 6 do código de verificação` |
| Input de mensagem do chat | `Escreva sua mensagem para o Analista de Oportunidades` |
| Área de histórico do chat | `Histórico da conversa com o Analista de Oportunidades` |

### 9.2 Alt Texts

| Elemento visual | Alt text |
|---|---|
| Ícone do FAB | `Chat com o Analista de Oportunidades` |
| Ícone do WhatsApp (T-016 step 3) | `WhatsApp aguardando confirmação` |
| Ícone de chat vazio (T-019 empty) | `Nenhuma conversa ainda` |
| Ícone de score baixo (verde) | `Score de risco baixo` |
| Ícone de score moderado (amarelo) | `Score de risco moderado` |
| Ícone de score alto (vermelho) | `Score de risco alto` |

### 9.3 Screen Reader

- `aria-live="polite"` — novos badges no FAB, novas mensagens no chat, atualizações de toggle.
- `aria-live="assertive"` — bloqueios de OTP, erros críticos de segurança.
- `role="log"` — área de histórico do chat.
- `role="dialog"` + `aria-modal="true"` — todos os modais (T-002, T-016, T-017, T-019 confirmação).
- `role="status"` — indicadores de score de risco (T-004), indicador "digitando" (T-003).

### 9.4 Ordem de Leitura

- Sempre: cabeçalho → conteúdo principal → ações (botões).
- Modais: título → conteúdo → ação primária → ação secundária.
- Formulários: label → input → helper text → error text.
- Chat: histórico → input → botão de envio → sugestões.

---

## 10. Vocabulário Controlado

🎯 **Um termo, um significado. Se o documento usa "Vincular", o código usa "Vincular" — nunca "Conectar", "Ativar" ou "Associar".**

| Termo correto | Termos proibidos (sinônimos) | Contexto de uso |
|---|---|---|
| **Analista de Oportunidades** | "Bot", "IA", "Assistente", "Chatbot", "Repasse AI" (na UI) | Nome do agente na interface do Cessionário |
| **Oportunidade** | "Imóvel", "Repasse", "Negócio", "Item" | Ativo disponível no marketplace |
| **Δ (Delta)** | "Lucro", "Ganho", "Margem", "Diferença" | Diferença Tabela Atual − Tabela Contrato |
| **Score de Risco** | "Nota de risco", "Avaliação", "Rating", "Pontuação" | Avaliação 1-10 da oportunidade pelo agente |
| **Vincular** | "Conectar", "Ativar", "Associar", "Cadastrar" | Ação de associar WhatsApp ao perfil |
| **Desvincular** | "Desconectar", "Remover número", "Cancelar WhatsApp" | Ação de remover vinculação WhatsApp |
| **Takeover** | "Assumir controle", "Intervenção manual", "Moderação" | Intervenção do Admin na conversa (uso interno/Admin) |
| **Assumir conversa** | "Fazer takeover", "Intervir", "Controlar" | Label do CTA de takeover na interface do Admin |
| **Escrow** | "Garantia", "Depósito", "Conta garantia" | Mecanismo de pagamento seguro |
| **Comissão** | "Taxa", "Fee", "Honorário" | Valor da comissão do Cessionário |
| **Consentimento** | "Aceite", "Permissão", "Autorização" | Consentimento LGPD |
| **Histórico de Chat** | "Conversas", "Mensagens salvas", "Log de chat" | Seção de histórico em Meu Perfil |
| **Excluir** | "Deletar", "Remover", "Apagar" | Ação destrutiva de remoção (usar "Apagar" como exceção no chat: "Apagar histórico") |
| **Alerta** | "Notificação", "Aviso", "Lembrete" | Mensagem proativa do agente |
| **Verificação** | "Autenticação", "Confirmação de identidade", "Check" | Processo de OTP |
| **Cessionário** | "Usuário", "Cliente", "Investidor" | Perfil comprador (interno — não aparece na UI do Cessionário) |
| **Admin** | "Administrador", "Moderador", "Equipe" | Perfil administrativo |
| **Re-verificar** | "Revalidar", "Re-autenticar", "Confirmar novamente" | Processo de re-verificação periódica do WhatsApp |

**Nota sobre "Analista de Oportunidades" vs "Repasse AI":**
- Na **interface do Cessionário**: sempre "Analista de Oportunidades" ou "seu Analista". Nunca "Repasse AI" — nome técnico/interno.
- Na **interface do Admin**: "Repasse AI" é aceito como nome do módulo.
- No **WhatsApp**: "Analista de Oportunidades" ou simplesmente "o Analista".

---

## 11. Tom de Voz — Guia Expandido

### 11.1 Personalidade da Interface

**5 adjetivos que definem a voz do Repasse AI:**

| Adjetivo | O que significa na prática | O que NÃO significa |
|---|---|---|
| **Especialista** | Usa os termos corretos do domínio (Δ, Escrow, OPR). Não simplifica ao ponto de ser vago. | Não usa jargão desnecessário ou termos técnicos sem contexto. Não é pedante. |
| **Direto** | Uma frase por ideia. Sem introdução. Sem "Prezado usuário". | Não é rude. Direto ≠ ríspido. |
| **Confiável** | As informações estão corretas e são apresentadas com precisão. Incertezas são explicitadas. | Não é inflexível. Admite quando não tem dados suficientes. |
| **Empático** | Reconhece que erros e bloqueios são frustrantes. Sempre indica o próximo passo. | Não é condescendente. Não usa emojis excessivos ou linguagem infantilizada. |
| **Profissional-acessível** | Tom de analista financeiro que fala com clareza para o público leigo. Nem robótico, nem informal. | Não usa gírias. Não usa "Oi!" com exclamação excessiva. Não é frio como sistema bancário. |

### 11.2 Escala de Formalidade por Contexto

| Contexto | Nível | Exemplo |
|---|---|---|
| Erro crítico / bloqueio de segurança | Formal, direto | `Muitas tentativas incorretas. O processo foi bloqueado por segurança. Aguarde 30 minutos.` |
| Erro técnico | Neutro, informativo | `Não foi possível carregar as interações. Tente novamente.` |
| Sucesso de ação destrutiva | Neutro, confirmativo | `Exclusão solicitada. Você receberá confirmação em até 48 horas.` |
| Sucesso de ação padrão | Neutro, positivo | `Preferência salva.` |
| Onboarding / primeiro acesso | Acolhedor, guiado | `Olá! Sou seu Analista de Oportunidades. Como posso ajudar?` |
| Empty state neutro | Leve, informativo | `Nenhuma interação registrada. Ajuste os filtros para ver resultados.` |
| Empty state de primeiro uso | Leve, encorajador | `Ainda não há interações. As métricas aparecerão após os primeiros usos.` |
| Alerta proativo | Direto, relevante | `Nova oportunidade compatível com seu perfil!` |
| Confirmação destrutiva | Preciso, calmo | `Ao desvincular, você não receberá mais alertas pelo WhatsApp.` |

### 11.3 Regras de Capitalização

| Elemento | Regra | Exemplo |
|---|---|---|
| Títulos de seção / tela | Sentence case | `Meu Perfil — Configurações` → `Meu perfil — configurações` ❌ → `Meu perfil` ✅ |
| Títulos de modal | Sentence case | `Vincular WhatsApp` ✅ · `Vincular whatsapp` 🔴 (nomes próprios mantêm maiúscula) |
| Labels de campo | Sentence case | `Número do WhatsApp` ✅ |
| Texto de botão (CTA) | Sentence case | `Enviar código` ✅ · `Enviar Código` 🔴 |
| Mensagens de erro e sucesso | Sentence case | `Código expirado. Solicite um novo.` ✅ |
| Nomes de entidades (OPR, Escrow, WhatsApp) | Mantêm capitalização de marca | `Escrow` ✅ · `escrow` 🔴 · `WhatsApp` ✅ · `Whatsapp` 🔴 |
| Δ (Delta) | Sempre com maiúscula e símbolo | `Δ (Delta)` na primeira menção; `Δ` nas subsequentes |

---

## 12. Changelog

| Versão | Data | Autor | Descrição |
|---|---|---|---|
| v1.0 | 22/03/2026 | Claude Code Desktop — Pipeline v6.1 | Criação. 5 princípios, 5 tipos de mensagem, 38 CTAs, 9 empty states, vocabulário controlado com 18 termos, tom de voz expandido, acessibilidade textual completa. |

---

## 13. Backlog de Pendências

| Item | Tipo | Seção | Impacto | Justificativa | Dono | Status |
|---|---|---|---|---|---|---|
| Tom do agente no WhatsApp vs webchat — distinção de nível de formalidade | [DECISÃO AUTÔNOMA] | 11.2 | P2 | WhatsApp tem caráter mais conversacional; webchat é contexto de trabalho. Decisão: mesmo tom em ambos os canais — consistência de marca prevalece sobre adaptação por canal. Alternativa descartada: tom mais informal no WhatsApp (risco de inconsistência percebida pelo usuário que usa os dois canais). | UX Writing | Aplicado |
| Emoji em mensagens proativas de alerta | [DECISÃO AUTÔNOMA] | 5.2, 7.3 | P2 | Alertas proativos (T-012 a T-014) usam ícones no wireframe. UX Writing opta por texto sem emoji inline nas mensagens — emojis são responsabilidade do componente visual, não do texto. Alternativa descartada: emoji inline nas mensagens (conflito com tom profissional-acessível definido). | UX Writing | Aplicado |
| "Apagar" vs "Excluir" no histórico de chat | [DECISÃO AUTÔNOMA] | 10 | P2 | "Apagar tudo" em T-019 (wireframe) usa linguagem mais acessível. Vocabulário controlado padroniza "Excluir" para ações destrutivas gerais, mas aceita "Apagar" como exceção no contexto de histórico de chat (linguagem mais próxima do usuário). Ambos proibidos de coexistir na mesma tela. | UX Writing | Aplicado |

---

*UX Writing v1.0 concluído. Cobertura: 11 seções, 5 tipos de mensagem, 38 CTAs, 18 termos controlados. Status: APROVADO. Próximo documento do pipeline: D10 — Glossário Técnico.*
