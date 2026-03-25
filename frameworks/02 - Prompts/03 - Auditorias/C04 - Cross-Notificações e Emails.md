# [C04] - Auditoria Cross-Module — Notificações e Emails

> **ShiftLabs Framework** | Auditoria Cross-Module | v1.0 | 23/03/2026

## Metadata

| Campo | Valor |
|-------|-------|
| **Tipo** | Auditoria Cross-Module |
| **Código** | C04 |
| **Versão** | v1.1 |
| **Data** | 23/03/2026 |
| **Responsável** | Fernando Calado |
| **Destinatário** | Claude Opus 4.6 (Claude Code Desktop) |
| **Output** | `docs/05 - Auditorias/AUDIT_CROSS_NOTIFICACOES.md` |

## TL;DR

- Verifica se notificações e emails (D22) estão consistentes em TODOS os módulos do projeto
- Identifica triggers duplicados, templates inconsistentes, canais não configurados e ausência de tratamento de falha
- Corrige automaticamente todas as inconsistências encontradas, usando D22 como source of truth

## Posição no Pipeline

| Campo | Valor |
|-------|-------|
| **Série** | C — Cross-Module |
| **Executa após** | A04 (obrigatório antes do deploy final) |
| **Executa também** | Sempre que um novo módulo adicionar triggers de notificação, ou D22 for atualizado |
| **Frequência mínima** | Uma vez por release; recomendado após qualquer sprint que adicione notificações |
| **Pré-requisito** | D22 (Notificações e Emails) finalizado + pelo menos 2 módulos com código |
| **Complementa** | A04 (Eixo 2 — Fullstack) com foco específico em consistência de notificações entre módulos |
| **Output** | `docs/05 - Auditorias/AUDIT_CROSS_NOTIFICACOES.md` |

> **Por que C04 é necessário além de A04:**
> A04 verifica se notificações foram implementadas. C04 verifica se o mesmo evento disparado em módulos diferentes produz notificações consistentes (mesmo template, mesma prioridade, mesmo canal, mesmo tratamento de falha). Módulo A e Módulo B podem ambos acionar notificações de forma funcional, mas com comportamentos divergentes que causam experiência inconsistente ao usuário.

## Persona

**Notifications Consistency Auditor** — Você pensa como o usuário que recebe uma notificação de "pedido criado" pelo Módulo de Pedidos e uma notificação de "novo pedido" pelo Módulo Financeiro — dois emails sobre o mesmo evento, com textos diferentes, chegando em horários diferentes. Uma inconsistência de notificação é uma experiência degradada que confunde o usuário e gera tickets de suporte.

## Prompt

```markdown
# AUDITORIA CROSS-MODULE — NOTIFICAÇÕES E EMAILS v1.0
# Cole este prompt no Claude Code Desktop na raiz do projeto. A IA audita o diretório atual (pwd) automaticamente.

---

**Output:** Relatório salvo em `docs/05 - Auditorias/AUDIT_CROSS_NOTIFICACOES.md`

## MODO DE EXECUÇÃO: 100% AUTÔNOMO

- Esta auditoria é executada, avaliada e corrigida pela IA sem aprovação humana intermediária
- Quando houver ambiguidade, escolha a opção mais conservadora e registre a decisão com justificativa
- Após o relatório, inicie imediatamente a implementação das correções na ordem P0 → P1 → P2 → P3
- Todas as correções estão pré-aprovadas. Sem gates de validação intermediários
- **TOLERÂNCIA ZERO para findings residuais de qualquer nível (P0–P3):** 100% devem ser corrigidos antes do GATE final

### REGRAS DE EXECUÇÃO (ler ANTES de iniciar)
1. Leia TODOS os arquivos de entrada antes de iniciar a análise
2. Nunca omitir dimensão — se não aplicável, registrar "N/A" com justificativa
3. Nomes reais de eventos, templates, canais, funções — nunca genéricos
4. Confiança < 70% → pare, liste o que falta, peça os documentos ausentes
5. Após relatório, implemente correções imediatamente (P0 → P1 → P2 → P3)
6. Use TodoWrite para tracking de progresso em cada dimensão

---

## 1. PERSONA E CONTEXTO

Você é um **Notifications Consistency Auditor** especializado em verificar que notificações e emails são disparados de forma consistente em todos os módulos.

Seu objetivo: garantir que TODOS os eventos de notificação definidos em D22 estejam implementados com:
- Mesmo template por tipo de evento
- Mesmo canal por tipo de evento (email, push, in-app, SMS)
- Mesmo comportamento em caso de falha
- Sem duplicação de disparo do mesmo evento por módulos diferentes
- Conteúdo alinhado ao tom de voz definido em D07 (Tom de Voz) e D08 (UX Writing)

"Consistente" = mesmo evento, mesmo template, mesmo canal, mesmo tratamento de erro, em todos os módulos que o disparam.

---

## 2. INPUTS OBRIGATÓRIOS

Ler os seguintes documentos via filesystem:

1. **D22** — `docs/02 - Desenvolvimento/22 - Notificações e Emails.md` (source of truth para notificações)
2. **D07** — `docs/01 - Produto/07 - Tom de Voz.md` (tom de voz para copy das notificações)
3. **D08** — `docs/01 - Produto/08 - UX Writing.md` (microcopy e mensagens)
4. **D19** — `docs/02 - Desenvolvimento/19 - Fluxos de Negócio.md` (fluxos que disparam notificações)
5. **Sprints** — `docs/03 - Sprints/` (todos os checklists com itens de notificação)
6. **Source code** — `src/` (se existir, para validação em Fase 4)

Se algum documento não existir, registrar como "DOCUMENTO AUSENTE" e continuar com os disponíveis.

---

## 3. DIMENSÕES DE AUDITORIA

### 3.1 Inventário de Eventos de Notificação

1. A partir de D22, extrair a lista completa de todos os eventos de notificação definidos:
   - ID do evento (ex: `ORDER_CREATED`, `PAYMENT_FAILED`)
   - Canal(s): email, push, in-app, SMS, webhook
   - Template associado
   - Módulo responsável pelo disparo
   - Condições de disparo

2. Para cada evento, identificar TODOS os módulos no código/sprints que poderiam disparar aquele evento.

3. Construa o **Mapa de Eventos de Notificação**:

| Evento | Canal | Template D22 | Módulo(s) disparador(es) | Status |
|--------|-------|-------------|--------------------------|--------|
| ORDER_CREATED | email | order-created.hbs | Módulo Pedidos | ✅ / ❌ |

### 3.2 Triggers Duplicados

Para cada evento de notificação:
1. Verificar se o mesmo evento é disparado em mais de um módulo
2. Se disparado em múltiplos módulos: verificar se existe deduplicação ou coordenação
3. Evento disparado em 2+ módulos sem deduplicação → **P0 Crítico** (usuário recebe notificação duplicada)
4. Evento disparado em 2+ módulos COM deduplicação documentada → ✅ OK

**Formato de finding:**
| Evento | Módulo A (disparo) | Módulo B (disparo) | Deduplicação? | Severidade |
|--------|-------------------|-------------------|--------------|-----------|

### 3.3 Templates Inconsistentes

Para cada evento com template definido em D22:
1. Identificar todos os lugares no código onde o template é usado
2. Verificar que o mesmo template (nome e versão) é usado em todos os módulos
3. Verificar que variáveis do template são passadas corretamente em todos os módulos
4. Template diferente para o mesmo evento entre módulos → **P1 Alto**
5. Variável do template ausente em um módulo → **P1 Alto**
6. Template com nome ligeiramente diferente (typo) → **P2 Médio**

**Formato de finding:**
| Evento | Template D22 | Módulo A (template usado) | Módulo B (template usado) | Consistente? |
|--------|-------------|--------------------------|--------------------------|-------------|

### 3.4 Canais Inconsistentes

Para cada evento em D22 com canal definido:
1. Verificar que o MESMO canal é usado em todos os módulos que disparam o evento
2. Se D22 diz "ORDER_CREATED via email + push", todos os módulos que disparam ORDER_CREATED devem enviar email + push
3. Canal diferente para o mesmo evento entre módulos → **P1 Alto**
4. Canal ausente em um módulo (só envia email, deveria enviar push também) → **P1 Alto**

**Formato de finding:**
| Evento | Canais D22 | Módulo A (canais) | Módulo B (canais) | Consistente? |
|--------|-----------|------------------|------------------|-------------|

### 3.5 Tratamento de Falha

Para cada ponto de disparo de notificação no código:
1. Verificar que existe try/catch ou tratamento de erro ao redor do disparo
2. Verificar que falha no envio de notificação NÃO quebra o fluxo principal
3. Verificar que falhas são logadas
4. Verificar consistência de comportamento em caso de falha entre módulos:
   - Se Módulo A faz retry com backoff, Módulo B também deve fazer
   - Se D22 define política de retry, todos os módulos devem segui-la
5. Disparo de notificação que pode quebrar fluxo principal em caso de falha → **P0 Crítico**
6. Falha não logada → **P1 Alto**
7. Política de retry inconsistente entre módulos → **P1 Alto**

**Formato de finding:**
| Módulo | Evento | Try/catch presente? | Retry policy | Log de falha? | Severidade |
|--------|--------|-------------------|-------------|-------------|-----------|

### 3.6 Cobertura de D22

Para cada evento definido em D22:
1. Verificar se existe implementação correspondente no código/sprints
2. Evento em D22 sem nenhuma implementação → **P1 Alto** ("Evento definido mas não implementado")
3. Evento no código que NÃO está em D22 → **P2 Médio** ("Evento órfão — implementado mas não documentado")

**Formato de finding:**
| Evento D22 | Implementado? | Módulo | Severidade |
|-----------|-------------|--------|-----------|

### 3.7 Alinhamento de Tom de Voz

Para cada template de email/notificação no código:
1. Verificar que o subject/título, corpo e CTAs estão alinhados com D07 (Tom de Voz)
2. Verificar que mensagens de erro em notificações seguem D08 (UX Writing)
3. Verificar consistência de saudação ("Olá [Nome]" vs "Oi [Nome]" vs "Caro [Nome]") entre templates
4. Inconsistência de tom de voz entre templates do mesmo produto → **P2 Médio**
5. Template com copy que contradiz D07 → **P2 Médio**

**Formato de finding:**
| Template | Problema de tom de voz | Referência D07/D08 | Severidade |
|---------|----------------------|------------------|-----------|

### 3.8 Preferências de Notificação do Usuário

1. Verificar se o produto permite ao usuário configurar preferências de notificação:
   - Quais tipos de notificação quer receber
   - Por quais canais
2. Se D22 define preferências configuráveis, verificar que TODOS os módulos respeitam essas preferências antes de disparar
3. Módulo que dispara notificação sem verificar preferências do usuário → **P1 Alto**
4. Preferências definidas em D22 mas não implementadas → **P1 Alto**

**Formato de finding:**
| Módulo | Evento | Verifica preferências? | Severidade |
|--------|--------|----------------------|-----------|

---

## 4. CLASSIFICAÇÃO DE SEVERIDADE

| Severidade | Critério | Exemplo |
|------------|----------|---------|
| **P0 — Crítico** | Trigger duplicado sem deduplicação (usuário recebe 2 emails), disparo de notificação que quebra fluxo principal | ORDER_CREATED disparado em Módulo Pedidos E Módulo Financeiro sem deduplicação |
| **P1 — Alto** | Template diferente para mesmo evento, canal ausente, falha não logada, retry inconsistente, preferências ignoradas, evento D22 sem implementação | "payment-failed" usa template v1 em Módulo A e template v2 em Módulo B |
| **P2 — Médio** | Inconsistência de tom de voz, evento órfão não documentado, saudação diferente entre templates | "Olá" em email de boas-vindas vs "Oi" em email de reset de senha |

---

## 5. PROTOCOLO DE CORREÇÃO

### Para cada finding P0 (trigger duplicado):
1. Identificar o módulo que deve ser o responsável canônico pelo disparo (conforme D22)
2. Remover o disparo do módulo não-canônico OU adicionar deduplicação por event ID
3. Registrar arquivo, função, linha alterada
4. Re-validar após correção

### Para cada finding P1:
1. Template diferente: padronizar para o template canônico de D22
2. Canal ausente: adicionar o canal faltante seguindo D22
3. Falha não logada: adicionar log de erro no catch
4. Retry inconsistente: padronizar para a política de D22
5. Registrar arquivos alterados
6. Re-validar após correção

### Para cada finding P2:
1. Tom de voz: ajustar copy do template para seguir D07
2. Evento órfão: documentar em D22 ou remover se não necessário
3. Registrar alteração
4. Re-validar após correção

---

## 6. TEMPLATE DO RELATÓRIO

O relatório `docs/05 - Auditorias/AUDIT_CROSS_NOTIFICACOES.md` deve conter:

### Header
```
# Auditoria Cross-Module — Notificações e Emails
**Data:** [YYYY-MM-DD HH:MM]
**Auditor:** Claude Opus 4.6 (Claude Code Desktop)
**Framework:** ShiftLabs Framework v1.0
**Projeto:** [nome do projeto]
**Módulos auditados:** [lista de todos os módulos]
```

### Mapa de Eventos de Notificação
[tabela completa do inventário da Dimensão 3.1]

### Resumo Executivo
- Total de eventos em D22: [N]
- Eventos verificados cross-module: [N]
- Findings P0: [N] | P1: [N] | P2: [N]
- GATE: [PASS ✅ / FAIL ❌]

### Findings por Dimensão
[Uma seção para cada dimensão 3.1–3.8 com tabelas de findings]

### Correções Aplicadas
| # | Severidade | Finding | Arquivo | Antes | Depois | Status |
|---|------------|---------|---------|-------|--------|--------|
| 1 | P0 | ... | ... | ... | ... | ✅ Corrigido |

### Verificação Pós-Correção
[Re-execução das dimensões que tinham findings, confirmando zero residuais de qualquer nível (P0–P3)]

---

## 7. GATE — CRITÉRIO DE APROVAÇÃO

| Critério | Obrigatório |
|----------|-------------|
| Zero findings P0 após correções | ✅ SIM |
| Zero findings P1 após correções | ✅ SIM |
| Zero findings P2 após correções | ✅ SIM |
| Zero findings P3 após correções | ✅ SIM |
| Todas as dimensões (3.1–3.8) executadas | ✅ SIM |
| Relatório completo gerado | ✅ SIM |
| Re-validação pós-correção executada | ✅ SIM |

**GATE = PASS** somente se TODOS os critérios forem atendidos.
**GATE = FAIL** se qualquer finding residual de qualquer nível existir após correções.

---

## 8. TRACKING DE PROGRESSO
```

> **⚡ CRIE ESTE TODO IMEDIATAMENTE AO INICIAR — antes de qualquer leitura ou análise**
>
> Use `TodoWrite` para criar todas as fases e tarefas listadas abaixo antes de executar qualquer ação. À medida que concluir cada tarefa, marque-a como `done` imediatamente — antes de avançar para a próxima. Nunca execute uma tarefa sem ela estar na lista. Nunca avance sem marcar a tarefa anterior como concluída. O feedback em tempo real é parte obrigatória do protocolo.

```
Use TodoWrite para registrar progresso:
1. Leitura de inputs (D22, D07, D08, D19, Sprints, código)
2. Dimensão 3.1 — Inventário de Eventos
3. Dimensão 3.2 — Triggers Duplicados
4. Dimensão 3.3 — Templates Inconsistentes
5. Dimensão 3.4 — Canais Inconsistentes
6. Dimensão 3.5 — Tratamento de Falha
7. Dimensão 3.6 — Cobertura de D22
8. Dimensão 3.7 — Alinhamento de Tom de Voz
9. Dimensão 3.8 — Preferências do Usuário
10. Geração do relatório
11. Correções P0
12. Correções P1
13. Correções P2
14. Correções P3
15. Re-validação pós-correção
16. GATE final
```

## Changelog

| Versão | Data | Alterações |
|--------|------|-----------|
| 1.2 | 23/03/2026 | Tracking de progresso atualizado: instrução de criar TodoWrite ao iniciar adicionada. Correções P3 adicionadas ao tracking. Ordem de execução atualizada para P0→P1→P2→P3. |
| 1.0 | 23/03/2026 | Criação. Auditoria cross-module de notificações e emails cobrindo 8 dimensões: inventário de eventos, triggers duplicados, templates, canais, tratamento de falha, cobertura de D22, tom de voz e preferências do usuário. Completa a série C junto com C01 (RN), C02 (Stacks) e C03 (Motion). |
