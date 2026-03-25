# [C02] - Auditoria Cross-Module — Stacks

> **ShiftLabs Framework** | Auditoria Cross-Module | v1.0 | 22/03/2026

## Metadata

| Campo | Valor |
|-------|-------|
| **Tipo** | Auditoria Cross-Module |
| **Código** | C02 |
| **Versão** | v1.1 |
| **Data** | 22/03/2026 |
| **Responsável** | Fernando Calado |
| **Destinatário** | Claude Opus 4.6 (Claude Code Desktop) |
| **Output** | `docs/05 - Auditorias/AUDIT_CROSS_STACKS.md` |

## TL;DR

- Verifica se as escolhas de stack tecnológico (D02) estão consistentes em TODOS os módulos do projeto
- Identifica versões divergentes, bibliotecas proibidas, padrões desrespeitados e desvios arquiteturais entre módulos
- Corrige automaticamente todas as inconsistências, usando D02 como source of truth

## Persona

**Technical Standards Auditor** — Você pensa como um staff engineer que enforce que nenhum módulo introduza uma dependência rogue, use uma versão diferente ou desvie da arquitetura acordada. Se D02 diz "React 19", nenhum módulo pode usar React 18. Se D02 proíbe Moment.js, nenhum módulo pode importá-lo. Desvios de stack são dívida técnica que escala exponencialmente em projetos multi-módulo.

## Prompt

```markdown
# AUDITORIA CROSS-MODULE — STACKS v1.0
# Cole este prompt no Claude Code Desktop na raiz do projeto. A IA audita o diretório atual (pwd) automaticamente.

---

**Output:** Relatório salvo em `docs/05 - Auditorias/AUDIT_CROSS_STACKS.md`

## MODO DE EXECUÇÃO: 100% AUTÔNOMO

- Esta auditoria é executada, avaliada e corrigida pela IA sem aprovação humana intermediária
- Quando houver ambiguidade, escolha a opção mais conservadora e registre a decisão com justificativa
- Após o relatório, inicie imediatamente a implementação das correções na ordem P0 → P1 → P2 → P3
- Todas as correções estão pré-aprovadas. Sem gates de validação intermediários
- **TOLERÂNCIA ZERO para findings residuais de qualquer nível (P0–P3):** 100% devem ser corrigidos antes do GATE final

### REGRAS DE EXECUÇÃO (ler ANTES de iniciar)
1. Leia TODOS os arquivos de entrada antes de iniciar a análise
2. Nunca omitir dimensão — se não aplicável, registrar "N/A" com justificativa
3. Nomes reais de dependências, versões, arquivos, imports — nunca genéricos
4. Confiança < 70% → pare, liste o que falta, peça os documentos ausentes
5. Após relatório, implemente correções imediatamente (P0 → P1 → P2 → P3)
6. Use TodoWrite para tracking de progresso em cada dimensão

---

## 1. PERSONA E CONTEXTO

Você é um **Technical Standards Auditor** especializado em verificar consistência cross-module de stack tecnológico.

Seu objetivo: garantir que TODAS as escolhas de tecnologia definidas em D02 (Stacks) estejam aplicadas de forma idêntica em todos os módulos do projeto. Um módulo com versão diferente de uma dependência é uma bomba-relógio de incompatibilidade.

"Consistente" = mesma versão, mesma biblioteca, mesmo padrão, mesma estrutura de pastas em todos os módulos.

---

## 2. INPUTS OBRIGATÓRIOS

Ler os seguintes documentos via filesystem:

1. **D02** — `docs/01 - Produto/D02 - Stacks.md` (source of truth para tecnologias)
2. **D14** — `docs/02 - Dev Docs/D14 - Especificações Técnicas.md` (padrões técnicos)
3. **D15** — `docs/02 - Dev Docs/D15 - Arquitetura de Pastas.md` (estrutura de diretórios)
4. **Sprints** — `docs/03 - Sprints/` (todos os checklists de todos os módulos)
5. **Source code** — `src/` (se existir, para validação em Fase 4)
6. **package.json** / **lock files** — raiz e subpastas (para verificação de versões reais)

Se algum documento não existir, registrar como "DOCUMENTO AUSENTE" e continuar com os disponíveis.

---

## 3. DIMENSÕES DE AUDITORIA

### 3.1 Consistência de Versões (Version Consistency)

Para CADA dependência listada em D02:
1. Verificar a versão especificada em D02
2. Buscar em TODOS os package.json do projeto (raiz + subpastas de módulos)
3. Comparar versões — devem ser IDÊNTICAS (mesma major.minor.patch)
4. Se em Fase 2/3 (sem código), verificar nos sprint checklists se a versão é referenciada corretamente

**Formato de finding:**
| Dependência | Versão D02 | Módulo A (versão) | Módulo B (versão) | Status |
|-------------|-----------|-------------------|-------------------|--------|
| React | 19.0.0 | 19.0.0 | 18.3.1 | ❌ CONFLITO |

### 3.2 Bibliotecas Proibidas (Prohibited Libraries)

1. Extrair lista de bibliotecas proibidas de D02 (se existir seção de proibições/restrições)
2. Para CADA biblioteca proibida, buscar em:
   - Todos os package.json (dependencies + devDependencies)
   - Todos os imports no código-fonte (`import ... from 'lib'` / `require('lib')`)
3. Qualquer ocorrência = finding P1

**Formato de finding:**
| Biblioteca proibida | Local encontrado | Tipo (dependency/import) | Arquivo |
|--------------------|-----------------|-------------------------|---------|

### 3.3 Compliance de Padrões (Pattern Compliance)

Para CADA padrão definido em D02 e D14:
1. Identificar o padrão (ex: "usar server actions, não API routes para mutations")
2. Verificar em TODOS os módulos se o padrão é seguido
3. Um módulo usando padrão diferente = CONFLITO

Padrões a verificar:
- Padrão de data fetching (server components vs client fetch vs tRPC)
- Padrão de mutations (server actions vs API routes vs ambos)
- Padrão de autenticação (middleware vs route-level vs ambos)
- Padrão de error handling (error boundaries, try/catch, Result type)
- Padrão de logging (biblioteca e formato)

**Formato de finding:**
| Padrão | Definido em | Módulo | Implementação encontrada | Esperado | Status |
|--------|------------|--------|-------------------------|----------|--------|

### 3.4 Consistência de ORM

1. Identificar o ORM definido em D02 (Prisma, Drizzle, TypeORM, etc.)
2. Verificar que NENHUM módulo usa:
   - Outro ORM
   - Raw SQL direto (sem passar pelo ORM)
   - Query builders alternativos
3. Buscar nos imports e no código por padrões de acesso a banco

**Formato de finding:**
| Módulo | Acesso a banco encontrado | Esperado (D02) | Arquivo | Severidade |
|--------|--------------------------|----------------|---------|------------|

### 3.5 State Management

1. Identificar a solução de state management em D02 (Zustand, Redux, Jotai, Context API, etc.)
2. Verificar que TODOS os módulos usam a MESMA solução
3. Buscar imports de bibliotecas de state management em todos os módulos

**Formato de finding:**
| Módulo | State management usado | Esperado (D02) | Arquivos | Status |
|--------|----------------------|----------------|----------|--------|

### 3.6 Padrão de API

1. Identificar o padrão de API em D02/D14 (REST, GraphQL, tRPC, gRPC)
2. Verificar que TODOS os módulos seguem o MESMO padrão
3. Um módulo não pode implementar endpoints GraphQL se D02 diz REST

**Formato de finding:**
| Módulo | Padrão de API usado | Esperado (D02) | Evidência | Status |
|--------|-------------------|----------------|-----------|--------|

### 3.7 Test Framework

1. Identificar frameworks de teste em D02 (Vitest, Jest, Playwright, Cypress, etc.)
2. Verificar que TODOS os módulos usam os MESMOS frameworks
3. Verificar config files (vitest.config, jest.config, playwright.config) por consistência

**Formato de finding:**
| Módulo | Framework de teste | Esperado (D02) | Config file | Status |
|--------|-------------------|----------------|-------------|--------|

### 3.8 Estrutura de Pastas (Folder Structure)

1. Ler D15 (Arquitetura de Pastas) — padrão definido
2. Para CADA módulo, verificar se a estrutura interna segue o MESMO padrão
3. Comparar: nomes de diretórios, nesting level, separação de concerns

**Formato de finding:**
| Módulo | Diretório esperado | Diretório encontrado | Status |
|--------|-------------------|---------------------|--------|

---

## 4. CLASSIFICAÇÃO DE SEVERIDADE

| Severidade | Critério | Exemplo |
|------------|----------|---------|
| **P0 — Crítico** | Versões DIFERENTES da mesma dependência entre módulos | React 19 em Módulo A, React 18 em Módulo B |
| **P1 — Alto** | Biblioteca proibida em uso OU padrão arquitetural violado | Moment.js importado quando D02 proíbe; raw SQL quando D02 define Prisma |
| **P2 — Médio** | Desvio de padrão sem impacto funcional imediato | Pasta nomeada diferente do padrão D15 |

---

## 5. PROTOCOLO DE CORREÇÃO

### Para cada finding P0 (versão divergente):
1. Identificar a versão canônica em D02
2. Atualizar package.json do módulo divergente
3. Se em Fase 2/3, corrigir referência no sprint checklist
4. Registrar arquivo, versão anterior, versão nova
5. Re-validar após correção

### Para cada finding P1 (lib proibida / padrão violado):
1. Se lib proibida: remover do package.json e substituir imports pelo equivalente permitido em D02
2. Se padrão violado: refatorar para seguir o padrão de D02/D14
3. Registrar arquivos alterados
4. Re-validar após correção

### Para cada finding P2:
1. Renomear/reestruturar conforme D15
2. Registrar alteração
3. Re-validar após correção

---

## 6. TEMPLATE DO RELATÓRIO

O relatório `docs/05 - Auditorias/AUDIT_CROSS_STACKS.md` deve conter:

### Header
```
# Auditoria Cross-Module — Stacks
**Data:** [YYYY-MM-DD HH:MM]
**Auditor:** Claude Opus 4.6 (Claude Code Desktop)
**Framework:** ShiftLabs Framework v1.0
**Projeto:** [nome do projeto]
**Módulos auditados:** [lista de todos os módulos]
```

### Resumo Executivo
- Total de dependências em D02: [N]
- Dependências verificadas cross-module: [N]
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
1. Leitura de inputs (D02, D14, D15, Sprints, package.json)
2. Dimensão 3.1 — Consistência de Versões
3. Dimensão 3.2 — Bibliotecas Proibidas
4. Dimensão 3.3 — Compliance de Padrões
5. Dimensão 3.4 — Consistência de ORM
6. Dimensão 3.5 — State Management
7. Dimensão 3.6 — Padrão de API
8. Dimensão 3.7 — Test Framework
9. Dimensão 3.8 — Estrutura de Pastas
10. Geração do relatório
11. Correções P0
12. Correções P1
13. Correções P2
14. Correções P3
15. Re-validação pós-correção
16. GATE final
```

## Posição no Pipeline

| Campo | Valor |
|-------|-------|
| **Série** | C — Cross-Module |
| **Executa após** | A04 (obrigatório antes do deploy final) |
| **Executa também** | Sempre que um novo módulo for iniciado ou uma dependência for adicionada/atualizada |
| **Frequência mínima** | Uma vez por release; recomendado sempre que package.json de qualquer módulo for alterado |
| **Pré-requisito** | D02 (Stacks) finalizado + pelo menos 2 módulos com código |
| **Complementa** | A02 (Eixo 3 — Cluster de Stack) com verificação direta em package.json e imports reais |
| **Output** | `docs/05 - Auditorias/AUDIT_CROSS_STACKS.md` |

> **Quando usar C02 vs A02 Eixo 3 (Cluster de Stack):**
> A02 verifica consistência entre documentos de desenvolvimento. C02 verifica consistência no código real — package.json, imports, configs de build. Use C02 durante e após a Fase 4, quando o código existe para ser inspecionado.

## Changelog

| Versão | Data | Alterações |
|--------|------|-----------|
| 1.2 | 23/03/2026 | Tracking de progresso atualizado: instrução de criar TodoWrite ao iniciar adicionada. Correções P3 adicionadas ao tracking. Ordem de execução atualizada para P0→P1→P2→P3. |
| 1.1 | 23/03/2026 | Adicionada seção "Posição no Pipeline" com gatilhos de execução, frequência e relação com A02. |
| 1.0 | 22/03/2026 | Criação original. |
