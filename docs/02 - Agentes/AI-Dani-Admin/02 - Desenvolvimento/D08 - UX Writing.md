# UX Writing — AI-Dani-Admin

## Guia de Escrita da Interface de Supervisão Operacional

| Campo | Valor |
|---|---|
| Destinatário | Produto, Design e Engenharia Frontend |
| Escopo | Todos os textos da interface do módulo AI-Dani-Admin: labels, mensagens de erro, confirmações, toasts, estados vazios, alertas e copy de takeover |
| Módulo | AI-Dani-Admin |
| Versão | v1.0 |
| Responsável | Claude Code Desktop |
| Data da versão | 2026-03-23 (America/Fortaleza) |
| Inputs | D03 (Brand Guide — tom de voz), D05 (RFs), D06 (Telas), D07 (Wireframes) |

---

> **📌 TL;DR**
>
> - **Tom:** Direto, técnico, orientado a dados e ação. O Admin é um profissional operacional — sem paternalismo, sem fofura, sem drama desnecessário.
> - **Urgência é proporcional à severidade.** Desligamento automático usa linguagem direta e imediata. Filtro de data usa linguagem neutra.
> - **Sempre com próximo passo.** Toda mensagem de erro e alerta termina com uma ação clara.
> - **Zero ambiguidade em ações destrutivas.** Takeover, reativação de agente e encerramento de takeover têm copy explícito sobre o que vai acontecer.
> - **Linguagem do usuário, não do sistema.** "Assumir conversa" — não "Iniciar takeover". "Nível de supervisão" — não "confidence_threshold".

---

## 1. Princípios de UX Writing

**P1. Direto e sem adorno.** O Admin está em modo operacional. Eliminar qualquer palavra que não acrescente informação ou ação. Sem "Por favor,", sem "Obrigado por usar..."

**P2. Dados substituem adjetivos.** "Taxa de erro: 32% nos últimos 15 min" — não "Taxa de erro muito alta". "CSAT: 3.2/5" — não "CSAT ruim".

**P3. Urgência é proporcional.** Desligamento automático: voz direta e imediata. Sinalização de revisão: voz informativa. Filtro sem resultado: voz neutra e útil.

**P4. Próximo passo sempre presente.** Toda mensagem de erro, alerta ou estado vazio termina com o que o Admin deve fazer a seguir.

**P5. Linguagem do Admin, não do código.** Usar vocabulário do domínio operacional (D01 - Glossário), não nomes de variáveis ou estados internos do sistema.

**P6. Ações críticas são explícitas.** Antes de qualquer ação com consequência irreversível ou de alto impacto (assumir conversa, reativar agente), o copy deixa claro o que vai acontecer.

---

## 2. Vocabulário Aprovado

### 2.1 Termos aprovados para uso na interface

| Termo na interface | Termo técnico interno | Quando usar |
|---|---|---|
| "Nível de supervisão" | confidence_threshold | Toda referência ao threshold de confiança |
| "Assumir conversa" | initiate_takeover | Botão e modal de takeover |
| "Encerrar atendimento humano" | end_takeover | Botão de encerramento de takeover |
| "Atendimento humano" | takeover | Separador visual no chat |
| "Aguardando revisão" | SINALIZADA_PARA_REVISAO | Badge de status na lista |
| "Em atendimento humano" | EM_TAKEOVER | Badge de status |
| "Respondida pela IA" | RESPONDIDA_PELA_IA | Badge de status |
| "Encerrada" | ENCERRADA | Badge de status |
| "Ativo" | AGENTE_ATIVO | Status do agente |
| "Desligado automaticamente" | DESLIGADO_AUTOMATICO | Status do agente em alerta |
| "Modo degradado" | FALLBACK_ATIVO | Status do agente em fallback |
| "Confiança" | confidence_score | Coluna e card de nível de confiança |
| "Dani" | ai_agent_display_name | Nome do agente na interface do usuário |

### 2.2 Termos proibidos na interface

| Proibido | Use em vez disso |
|---|---|
| "threshold" (exposto ao Admin) | "Nível de supervisão" |
| "takeover" (exposto ao Admin) | "Assumir conversa" / "Atendimento humano" |
| "confidence_score" | "Nível de confiança" / "Confiança" |
| "N/A" em campos de dados | "Dados insuficientes" ou deixar o card vazio com mensagem |
| "0" ou "0%" para métricas sem dados | "Dados insuficientes para o período selecionado" |
| "Erro" sozinho (sem contexto) | Descrição do erro + ação |
| "Sucesso!" sozinho | Confirmação específica: "Threshold atualizado para 75%." |
| "Indisponível" (sem contexto) | Mensagem completa com contexto e próximo passo |

---

## 3. Copy por Componente

### 3.1 Títulos e Headers

| Tela | H1 | Subtítulo (se aplicável) |
|---|---|---|
| T-001 | "Supervisão IA" | — |
| T-002 | "Interação #[ID]" | — |
| T-003 | "Atendimento humano ativo" | — |
| T-004 | "Dashboard de Métricas" | — |
| T-005 | "Configurações de Supervisão" | — |
| T-006 | "Log de Auditoria" | — |
| T-007 | "Checklist de Prontidão para Lançamento" | — |

### 3.2 Labels de Navegação (Sidebar)

```
Supervisão IA
  ∟ Interações
  ∟ Métricas
  ∟ Configurações
  ∟ Auditoria
  ∟ Checklist de Lançamento
```

### 3.3 Labels de Filtros e Campos

| Campo/Filtro | Label | Placeholder |
|---|---|---|
| Filtro de data | "Período" | "Selecionar período" |
| Filtro de agente | "Agente" | "Todos os agentes" |
| Filtro de confiança | "Nível de confiança" | "Qualquer nível" |
| Filtro de status | "Status" | "Todos os status" |
| Campo de threshold | "Nível de supervisão (%)" | "Ex: 80" |
| Campo de rate limit | "Limite de mensagens por hora" | "Ex: 30" |
| Campo de motivo do takeover | "Motivo (opcional)" | "Selecionar motivo" |

### 3.4 Opções do Dropdown de Motivo de Takeover

```
- Resposta incorreta da IA
- Confiança abaixo do aceitável
- Situação complexa para a IA
- Solicitação do usuário
- Outro
```

---

## 4. Estados Vazios

### 4.1 T-001 — Sem interações no período

```
[Ícone: chat sem mensagem]

"Nenhuma interação registrada no período selecionado."

"Tente ajustar o período ou os filtros aplicados."
```

**Regras:**
- Nunca exibir "Não há dados." — sem próximo passo.
- Nunca exibir apenas a ilustração sem texto.
- O texto de sugestão deve indicar ação concreta.

### 4.2 T-004 — Card de métrica sem dados

```
ℹ "Dados insuficientes para o período selecionado."
```

**Regra:** Nunca exibir "0", "0%", ou "-" como valor de métrica quando não há dados. A ausência de dado é diferente de zero interações.

### 4.3 T-006 — Sem eventos no log

```
[Ícone: documento vazio]

"Nenhum evento registrado neste período."
```

---

## 5. Mensagens de Erro

### 5.1 Validação de Threshold (RF-016)

```
⚠ "O nível de supervisão precisa estar entre 50% e 95%.
   Valores fora desse intervalo podem comprometer a qualidade do atendimento."
```

**Regras:**
- Exibida inline abaixo do campo, em cor de alerta (`--color-warning`).
- Não fecha o modal/drawer.
- Mantém o valor inválido no campo para correção sem redigitar.
- Não usa "Erro:" como prefixo — o contexto já está claro.

### 5.2 Falha ao Salvar Threshold

```
"Não foi possível salvar o nível de supervisão. Tente novamente."
```

**Apresentação:** Toast de erro (cor destrutiva). Auto-dismiss 6 segundos. Botão "Tentar novamente" no toast.

### 5.3 Tentativa de Takeover em Conversa Encerrada (RF-010)

```
"Esta conversa já foi encerrada e não pode receber atendimento humano."
```

**Apresentação:** Tooltip no botão desabilitado "Assumir conversa". Sem modal, sem toast.

### 5.4 Conflito de Takeover — Segundo Admin (RF-011)

```
"Esta conversa já está em atendimento por outro analista."
```

**Apresentação:** Toast de informação (cor neutro-azul). Auto-dismiss 4 segundos.

### 5.5 Falha no Envio de Mensagem durante Takeover

```
"Não foi possível enviar a mensagem. Tente novamente."
```

**Apresentação:** Inline abaixo do campo de digitação. Botão "Tentar novamente" na mensagem.

### 5.6 Agente Não Pôde Ser Reativado

```
"Não foi possível reativar o agente. Verifique se o problema foi resolvido e tente novamente."
```

**Apresentação:** Toast de erro. Auto-dismiss 6 segundos.

---

## 6. Confirmações e Toasts de Sucesso

### 6.1 Toast: Threshold Atualizado (RF-015)

```
✓ "Nível de supervisão atualizado para [valor]%."
```

**Regras:**
- Auto-dismiss 4 segundos.
- Sempre inclui o novo valor para confirmação visual.
- Nunca "Salvo com sucesso!" — sem o valor, não é confirmação útil.

### 6.2 Toast: Rate Limit Atualizado

```
✓ "Limite de mensagens atualizado para [valor] mensagens por hora."
```

### 6.3 Toast: Takeover Iniciado

```
✓ "Você assumiu essa conversa."
```

**Regras:** Auto-dismiss 3 segundos (o Admin já está na tela de chat, confirmação rápida é suficiente).

### 6.4 Toast: Takeover Encerrado

```
✓ "Atendimento encerrado. O agente retomou a conversa."
```

### 6.5 Toast: Agente Reativado

```
✓ "Dani-[Cessionário/Cedente] reativado com sucesso."
```

---

## 7. Mensagens de Alerta (COMP-001)

### 7.1 Desligamento Automático

**Título:** "Agente desligado automaticamente"

**Corpo:** "Dani-[Cessionário/Cedente] foi desligado. Taxa de erro: [X]% nos últimos 15 minutos. Reative somente após identificar e resolver a causa."

**CTA primário:** "Reativar agente"
**CTA secundário:** "Ver detalhes"

**Regras:**
- Máxima urgência — aparecer instantaneamente.
- Não auto-dismiss — persiste até ação.
- Usar números reais do incidente (não "taxa elevada" sem valor).
- Incluir aviso sobre verificação prévia à reativação.

### 7.2 Latência Alta

**Título:** "Latência alta detectada"

**Corpo:** "O tempo de resposta está acima do SLA há [X] minutos. Verifique a disponibilidade da API do modelo."

**CTA:** "Ver métricas"

### 7.3 Taxa de Erro Elevada

**Título:** "Taxa de erro elevada"

**Corpo:** "[X]% das respostas com erro nos últimos 15 minutos. Monitore se a taxa continua subindo."

**CTA:** "Ver interações com erro"

### 7.4 CSAT Degradado

**Título:** "CSAT abaixo da meta"

**Corpo:** "Média: [X]/5 nas últimas 24 horas (meta: 3,5/5). Revise as interações recentes para identificar padrões."

**CTA:** "Ver interações recentes"

### 7.5 Taxa de Recusa Alta

**Título:** "Taxa de recusa elevada"

**Corpo:** "[X]% das respostas com recusa nas últimas 24 horas. Verifique tentativas de manipulação ou problema de experiência."

**CTA:** "Ver interações com recusa"

### 7.6 Consumo de Processamento

**Título:** "Consumo de processamento elevado"

**Corpo:** "[X]% do orçamento mensal consumido. Avalie otimizações no volume de dados por interação."

**CTA:** "Ver dashboard de custos"

---

## 8. Copy de Takeover (COMP-002 — Mensagens para o Usuário)

### 8.1 Início do Takeover (mensagem exibida ao usuário)

```
"Um analista da equipe Repasse Seguro assumiu essa conversa para ajudá-lo. Como posso ajudar?"
```

**Regras:**
- Tom acolhedor mas profissional — é um ser humano falando.
- Não menciona "IA" ou que houve algum problema com o agente.
- Inclui convite para o usuário continuar (próximo passo).
- Exibida com avatar de pessoa (ícone humano, não IA) + nome "Equipe Repasse Seguro".

### 8.2 Separador Visual no Chat

```
──── Atendimento humano ────
```

**Regras:**
- Texto em `--muted-foreground`, font-size pequeno (`text-xs`).
- Linha divisória visual antes e depois do texto.
- Identifica claramente a mudança de remetente para o usuário.

### 8.3 Retorno ao Agente (mensagem exibida ao usuário após encerramento do takeover)

```
"Você está novamente em atendimento com Dani."
```

**Regras:**
- Usa o nome do agente (não "IA" ou "sistema automatizado").
- Tom neutro — não "infelizmente voltamos ao bot".
- Exibida com avatar padrão do agente.

### 8.4 Separador Visual de Retorno

```
──── Retorno ao atendimento automatizado ────
```

---

## 9. Copy de Checklist de Prontidão (T-007)

### 9.1 Status do Checklist

| Estado | Copy |
|---|---|
| Todos aprovados | ✅ "Pronto para lançamento" |
| Itens pendentes | ❌ "Bloqueado — [X] itens pendentes" |
| Item aprovado individual | ✅ [texto do item] |
| Item reprovado individual | ❌ [texto do item] |
| Item pendente de verificação | ○ [texto do item] |

### 9.2 Textos dos Itens do Checklist

**Grupo 1 — Isolamento de Acesso:**
- "Filtro de escopo implementado: toda consulta valida que o recurso pertence ao usuário autenticado."
- "Filtro de contexto implementado: agente recebe apenas dados autorizados para o perfil do usuário."
- "Teste de penetração: acesso cruzado de dados bloqueado em 100% dos cenários testados."

**Grupo 2 — Cobertura de Recusa:**
- "Identidade e tom do agente definidos nas instruções permanentes."
- "Lista de dados bloqueados com exemplos de recusa documentada."
- "Mínimo 20 perguntas adversariais testadas com 100% de recusa correta."
- "Formato de resposta: toda resposta encerra com próximo passo claro."

**Grupo 3 — Supervisão Admin Funcional:**
- "Registro de interações: pergunta, resposta, confiança e latência gravados por interação."
- "Dashboard de métricas operacional: volume, CSAT, recusa e latência visíveis."
- "Alerta automático de confiança: interações abaixo do threshold sinalizadas automaticamente."
- "Takeover manual testado: Admin assumiu e encerrou conversa com registro de motivo."

### 9.3 Botão de Autorização

| Estado | Copy do Botão |
|---|---|
| Checklist incompleto | "Autorizar lançamento" (desabilitado, cinza, cursor not-allowed) |
| Checklist completo | "Autorizar lançamento" (habilitado, primário) |

### 9.4 Modal de Confirmação de Lançamento

```
"Autorizar lançamento do agente"

Você está autorizando o lançamento do [Dani-Cessionário/Dani-Cedente] em produção.
Todos os critérios de prontidão foram verificados e aprovados.

Esta ação será registrada no log de auditoria.

[Cancelar]    [Autorizar lançamento]
```

---

## 10. Copy de Indisponibilidade do Agente (Webchat — RF-018)

### 10.1 Mensagem no chat quando API indisponível

```
"O agente está temporariamente indisponível. Os cálculos de comissão e Escrow continuam disponíveis. Tente novamente em instantes."
```

**Regras:**
- Não culpa o usuário.
- Comunica o que ainda funciona (Calculadora de Comissão).
- Inclui próximo passo implícito ("em instantes" = tentar de novo).
- Não usa palavras técnicas como "API" ou "timeout".

### 10.2 Badge de status degradado no FAB

```
[Tooltip do badge amarelo]: "Agente temporariamente indisponível"
```

---

## 11. Microcopy — Elementos Recorrentes

### 11.1 Paginação

```
Exibindo [X]–[Y] de [total] interações
< Anterior  1  2  3  ...  Próxima >
```

### 11.2 Timestamp na lista de interações

Formato: `DD/MM HH:mm` (sem segundos, sem ano quando é o mesmo ano).
Exemplo: `23/03 14:32`

Tooltip no hover do timestamp: data e hora completa com timezone.
Exemplo: `23/03/2026 14:32:05 (America/Fortaleza)`

### 11.3 Nível de confiança na lista

| Valor | Exibição |
|---|---|
| >= threshold | `94%` (texto padrão, cor foreground) |
| < threshold | `61% ⚠` (com ícone de aviso, cor warning) |
| Não disponível | `—` (traço, apenas quando takeover está ativo e IA não respondeu) |

### 11.4 Latência na lista

| Valor | Exibição |
|---|---|
| <= 2s | `1.2s` (texto padrão) |
| > 2s e <= SLA | `2.8s` (texto em warning) |
| > SLA | `4.1s ⚠` (com ícone, cor warning) |

### 11.5 Skeleton durante loading de filtro

```
[skeleton] [skeleton]  [skeleton] [skeleton]  [████]
```
5 linhas skeleton inline. Não substitui a tabela inteira.

---

## 12. Acessibilidade de Texto

- Todos os ícones acompanhados de label de texto visível OU `aria-label` descritivo.
- Badges de status têm `aria-label` que descreve o estado completo: `aria-label="Status: Aguardando revisão"`.
- Botões desabilitados têm `title` ou `aria-describedby` explicando por que estão desabilitados.
- Mensagens de erro associadas ao campo por `aria-describedby`.
- Alertas críticos (COMP-001) usam `role="alert"` para ser anunciados por leitores de tela.

---

## 13. Changelog

| Data | Versão | Descrição |
|---|---|---|
| 2026-03-23 | v1.0 | Versão inicial. UX Writing completo para o módulo AI-Dani-Admin. |
