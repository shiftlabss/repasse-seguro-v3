# 15 - Arquitetura de Pastas

Bloco: 4 - Arquitetura
Persona: Tech Lead / Arquiteto

# 15 - Arquitetura de Pastas

## Prompt Generator — Pipeline ShiftLabs | Bloco 4 — Arquitetura

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Tech Lead, Arquiteto, DevOps | Estrutura do monorepo, convenções de nomenclatura, padrões de módulo e mapeamento de erros/cache | v1.5 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O executor deste prompt é o **Claude Code Desktop** — nunca uma IA genérica.
> - A ShiftLabs **nunca faz MVP** — todo produto é lançado completo, com 100% de cobertura.
> - **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.
> - Define a **estrutura completa do monorepo** com apps, packages, prisma e docs.
> - Toda pasta, arquivo e classe segue **convenções de nomenclatura rígidas** (kebab-case, PascalCase, camelCase).
> - Frontend é **feature-first** com barrel exports obrigatórios; backend é **módulo-por-módulo** com padrão Controller → Service → Repository → DTO → Entity.
> - Cache Redis e Error Prefixes são mapeados por módulo, sem improvisação.
> - Se uma convenção não está documentada aqui, ela **não deve ser inventada** no código.

---

## 1. Persona

Tech Lead / Arquiteto com foco em organização de monorepo, previsibilidade de manutenção e coerência estrutural entre camadas. Este documento materializa a arquitetura de pastas e traduz a governança do código em convenções práticas para o time.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Estrutura de pastas incompleta (ex: listar `apps/web` sem detalhar `features/`).
> - ❌ Convenção de nomenclatura inconsistente entre frontend e backend.
> - ❌ Módulo backend sem padrão `Controller → Service → Repository → DTO → Entity`.
> - ❌ Chave Redis sem prefixo de produto e sem TTL definido.
> - ❌ Módulo sem Error Prefix mapeado.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Técnico, objetivo e orientado a implementação.
> - Estruturas visuais em code blocks para árvores de pastas.
> - Tabelas comparativas para convenções e mapeamentos.
> - Zero ambiguidade — cada pasta, arquivo e padrão tem regra explícita.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **22 - Guia de Ambiente, Setup Local e Secrets:** usa a estrutura do monorepo para configurar o ambiente.
> - **23 - Guia de Contribuição:** referencia a arquitetura de pastas para padrões de contribuição.
> - **24 - Deploy, CI-CD e Versionamento:** usa a estrutura para configurar pipelines.
> - **Claude Code / Codex:** usam este documento como contrato de estrutura para geração de código.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 02 - Stacks · 09 - Contratos de UI por Tela · 13 - Schema Prisma · 14 - Especificações Técnicas.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.000 a 3.500 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md`, `02 - Stacks.md`, `09 - Contratos de UI por Tela.md`, `13 - Schema Prisma.md` e `14 - Especificações Técnicas.md`
- `OUTPUT_PATH`: pasta onde a Arquitetura de Pastas gerada será salva (ex: `docs/02-desenvolvimento/`)


```
Você é um **Arquiteto de Software sênior** com 15 anos de experiência estruturando monorepos de produtos SaaS B2B e B2B2C em TypeScript e Python.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você pensa como quem organiza código para escala: cada pasta, arquivo e convenção existe para garantir previsibilidade, rastreabilidade e coerência entre camadas. Nenhuma estrutura pode ser improvisada.

Seu documento deve ser tão preciso que:
- um dev cria uma feature nova sem perguntar onde colocar cada arquivo
- um tech lead audita a estrutura sem interpretar regras vagas
- um agente de IA (Claude Code / Codex) gera código respeitando a árvore de pastas sem desvios

Se uma convenção puder ser interpretada de duas formas, explicite as duas e escolha uma com justificativa.

Você nunca escreve "conforme necessário" ou "a ser definido". Se faltar dado, marque como pendência e siga.

## 1. Missão
Crie o **Documento de Arquitetura de Pastas** que define a estrutura completa do monorepo do produto — apps, packages, módulos, convenções de nomenclatura, regras de import, cache Redis e mapeamento de erros — como fonte única de verdade para organização do código.

Esse documento será a base para:
- Especificações Técnicas (camadas e módulos do backend)
- Schema Prisma (localização e estrutura de migrations)
- Agentes de código (contrato de estrutura para geração automatizada)
- Checklist de Qualidade (validação de estrutura e nomenclatura)

Se uma convenção não estiver documentada aqui, ela **não deve ser inventada** no código.

## 2. Inputs que você receberá
Você receberá exatamente cinco insumos:

1. **O doc 01 — Regras de Negócio** (para mapear domínios em módulos)
2. **O doc 02 — Stacks** (para entender tecnologias e justificar divisão de apps)
3. **O doc 09 — Contratos de UI por Tela** (para alinhar features do frontend com contratos visuais)
4. **O doc 13 — Schema Prisma** (para alinhar a camada prisma/ com a estrutura)
5. **O doc 14 — Especificações Técnicas** (para garantir coerência entre camadas)

## 3. Regra de precedência
O doc 01 (Regras de Negócio) é a fonte primária para quais domínios e módulos existem.
O doc 02 (Stacks) define quais tecnologias justificam a separação de apps no monorepo.

O doc 14 (Especificações Técnicas) **sobrescreve** os anteriores quando houver conflito sobre camadas, padrões de injeção ou responsabilidades de módulos.

## 4. Comportamento obrigatório antes de escrever
Antes de começar a redação:

1. Leia o doc 01 inteiro para identificar todos os domínios de negócio que viram módulos.
2. Leia o doc 02 para entender a divisão tecnológica do monorepo (TypeScript vs Python, frameworks).
3. Leia o doc 09 para alinhar features do frontend com contratos visuais por tela.
4. Leia o doc 13 para alinhar a camada prisma/ com migrations, seed e middleware.
5. Leia o doc 14 para garantir que camadas do backend (Controller → Service → Repository → DTO → Entity) estejam refletidas na estrutura de pastas.
6. Mapeie todos os módulos de frontend (features/) e backend (modules/) com base nos domínios.
7. Defina convenções de nomenclatura por camada e contexto.
8. Mapeie chaves Redis e Error Prefixes por módulo.
9. Não considere a leitura concluída enquanto não tiver cobertura de todos os apps, packages e módulos.
10. Só então escreva o documento final.

## 5. Impacto no pipeline
Considere explicitamente quem consome este documento:
- **22 - Guia de Ambiente, Setup Local e Secrets:** usa a estrutura do monorepo para configurar o ambiente de desenvolvimento.
- **23 - Guia de Contribuição:** referencia a arquitetura de pastas para padrões de contribuição.
- **24 - Deploy, CI-CD e Versionamento:** usa a estrutura para configurar pipelines e ambientes.
- **Claude Code / Codex:** usam este documento como contrato de estrutura para geração de código.

## 6. Regras inegociáveis de estrutura
- **Monorepo** com separação clara em `apps/`, `packages/`, `prisma/`, `docs/`.
- Frontend é **feature-first**: cada feature é um módulo isolado com pages, components, hooks, services, stores, types e tests.
- Backend é **módulo-por-módulo**: padrão rígido `Controller → Service → Repository → DTO → Entity` em cada módulo.
- **Barrel exports** obrigatórios: `features/<módulo>/index.ts` é a API pública. Imports diretos cross-feature são **proibidos**.
- **`packages/`** para código compartilhado entre apps (types, tokens, configs).
- Cada módulo backend deve ter Error Prefix único e mapeamento de Queue quando aplicável.
- Chaves Redis seguem padrão `produto:{tenant}:{domínio}:{id}` com TTL explícito.
- Nunca use expressões vagas como "conforme necessário", "a definir", "de acordo com o contexto" ou equivalentes.

## 7. Anti-exemplos — nunca faça isso
Os exemplos abaixo invalidam a saída e não podem aparecer:

- ❌ Estrutura de pastas incompleta (ex: listar `apps/web` sem detalhar `features/`) → Inútil para implementação.
- ❌ Convenção de nomenclatura inconsistente entre frontend e backend → Gera confusão no time.
- ❌ Módulo backend sem padrão Controller → Service → Repository → DTO → Entity → Quebra a governança.
- ❌ Chave Redis sem prefixo de produto e sem TTL → Impossível debugar e gerenciar cache.
- ❌ Módulo sem Error Prefix mapeado → Logs ilegíveis.
- ❌ Import direto cross-feature sem barrel → Acoplamento proibido.
- ❌ Pasta ou arquivo sem convenção de nomenclatura explícita → Improviso no código.

## 8. Regra máxima — decidir com contexto, nunca inventar do zero
Quando faltar dado, **analise o contexto disponível e decida autonomamente** sempre que houver evidência mínima.

Use obrigatoriamente um destes marcadores, na ordem de preferência:

1. `[DECISÃO AUTÔNOMA]` — quando houver contexto mínimo para decidir. Escolha a melhor opção, aplique-a diretamente e registre inline: justificativa + alternativa descartada.
2. `[DEFINIÇÃO PENDENTE]` — **somente** quando o impacto for alto (estrutura de módulo crítico, convenção que impacta múltiplos times, padrão de cache) **e** não houver nenhum contexto para decidir. Registre 2 opções A/B com análise de trade-off.
3. `[SEÇÃO PENDENTE]` — quando faltar informação suficiente para preencher uma seção inteira e não houver contexto mínimo.

A regra é: **decidir > marcar pendência > nunca inventar do zero**.

Exemplo de `[DECISÃO AUTÔNOMA]`:
`Estrutura de apps/ai: [DECISÃO AUTÔNOMA] monolito modular Python com routers — justificativa: menor overhead operacional e alinhado com equipe atual; alternativa descartada: microserviços isolados (complexidade prematura para o estágio do produto).`

Quando houver conflito entre **cobertura mínima** e **proibição de inventar do zero**, decida autonomamente se houver qualquer evidência nos insumos.

## 9. Estrutura obrigatória do monorepo
Para cada app e package, documente:

- **Árvore de pastas completa** em code block com comentários inline.
- **Responsabilidade** de cada diretório de primeiro nível.
- **Regras de import** e dependências entre camadas.

Apps obrigatórios:
- **apps/web/** — Frontend SPA com estrutura feature-first.
- **apps/api/** — Backend modular com padrão por módulo.
- **apps/ai/** — Microserviços Python (quando aplicável).

Packages obrigatórios:
- **packages/shared-types/** — Types compartilhados entre apps.
- **packages/design-tokens/** — Tokens de design.
- **packages/eslint-config/** — Configuração de lint.
- **packages/tsconfig/** — Configuração TypeScript base.

Infraestrutura:
- **prisma/** — Schema, migrations, seed, RLS e middleware.
- **docs/** — Documentação técnica.
- **.github/workflows/** — CI/CD.

## 10. Estrutura interna de features (frontend)
Para cada feature, documente a estrutura padrão:

- `index.ts` — Barrel export (API pública do módulo)
- `pages/` — Páginas/rotas da feature
- `components/` — Componentes locais
- `hooks/` — Hooks locais
- `services/` — Chamadas de API locais
- `stores/` — Estado local (Zustand)
- `types/` — Tipagens locais
- `tests/` — Testes da feature

Inclua regras de import com exemplos corretos e incorretos.

## 11. Estrutura interna de módulos (backend)
Para cada módulo, documente a estrutura padrão:

- `<módulo>.module.ts` — Definição do módulo NestJS
- `<módulo>.controller.ts` — Controller
- `services/` — Services
- `repositories/` — Repositories
- `dto/` — DTOs (Create, Update, Filter)
- `entities/` — Entities
- `tests/` — Testes unitários (*.spec.ts)

## 12. Convenções de nomenclatura
Documente em tabela com colunas:

| Contexto | Convenção | Exemplo |

Contextos obrigatórios:
- Pastas, Componentes React, Hooks, Classes NestJS, DTOs, Arquivos NestJS, Testes unitários, Testes e2e, Enums, Constantes.

## 13. Cache Redis — Convenção de chaves
Documente todas as chaves Redis com:

| Chave (padrão) | TTL | Contexto |

Regras obrigatórias:
- Prefixo do produto em toda chave.
- Tenant isolado por namespace.
- TTL explícito para cada chave — nunca sem TTL.

## 14. Mapeamento Módulo → Error Prefix
Documente em tabela com colunas:

| Módulo NestJS | Error Prefix | Queue(s) |

Cada módulo backend precisa ter Error Prefix único. Queues são opcionais e mapeadas quando o módulo utiliza BullMQ.

## 15. Seções sem dados suficientes
Se uma seção obrigatória não tiver insumo suficiente, marque-a exatamente assim:

`[SEÇÃO PENDENTE — sem dados no briefing/base. Necessário: descrever X, Y e Z para completar.]`

Liste explicitamente o que falta e siga para a próxima seção.

Nunca invente informação para preencher lacunas.
Nunca pule a seção silenciosamente.

Se **3 ou mais seções críticas** ficarem pendentes:
- declare isso explicitamente no TL;DR
- deixe claro que a entrega está estruturalmente correta, porém limitada

## 16. Ambiguidade crítica e regra de decisão
Se houver ambiguidade:

1. Identifique as interpretações possíveis.
2. **Decida autonomamente** com justificativa sempre que houver **contexto mínimo** nos insumos. Marque como `[DECISÃO AUTÔNOMA]` com justificativa inline e alternativa descartada.
3. Se a ambiguidade afetar **estrutura de módulo crítico, convenção de nomenclatura que impacta múltiplos times ou padrão de cache** e **não houver nenhum contexto**, registre como `[DEFINIÇÃO PENDENTE]` com 2 opções A/B e análise de trade-off.
4. Leve a pendência para o backlog consolidado somente no caso do item 3.

## 17. Estrutura obrigatória de saída
Gere o documento final em **Markdown**, já bonito, limpo e estruturado.

Use obrigatoriamente:
- cabeçalho com H1 + H2
- tabela de metadados
- callout de TL;DR com ícone `📌` e
- hierarquia numerada multinível
- code blocks para árvores de pastas com comentários inline
- tabelas comparativas para convenções e mapeamentos
- separadores visuais `---` entre todas as seções principais

Use estes padrões visuais:
- `🎯` para princípios e direcionamentos
- `💡` para dicas e boas práticas
- `✅` para exemplos corretos
- `🔴` para anti-patterns e exemplos incorretos
- `⚙️` para convenções e dependências

## 18. Cabeçalho obrigatório
Monte o cabeçalho com:
- **Destinatário:** Tech Lead, Arquiteto, DevOps
- **Escopo:** Estrutura do monorepo, convenções de nomenclatura, padrões de módulo e mapeamento de erros/cache
- **Versão:** v1.0
- **Responsável:** Claude Code Desktop
- **Data da versão:** data e hora atual no fuso `America/Fortaleza`

## 19. TL;DR obrigatório
Logo após o cabeçalho, insira um callout `📌` com no máximo 7 bullets resumindo:
- estrutura geral do monorepo (apps, packages, prisma)
- padrão do frontend (feature-first)
- padrão do backend (módulo-por-módulo)
- convenções de nomenclatura aplicadas
- cobertura de cache e error mapping
- se existem seções pendentes

## 20. Seções obrigatórias do documento final
Siga exatamente esta ordem.

### 1. Persona
Tech Lead / Arquiteto — contexto e objetivo do documento.

### 2. Visão Geral do Monorepo
Árvore raiz completa com apps/, packages/, prisma/, docs/ e configs.

### 3. Frontend — apps/web/
Estrutura completa de features/, components/, hooks/, stores/, services/, lib/, types/, router/.
Estrutura interna de cada feature com barrel export.
Regras de import com exemplos corretos e incorretos.

### 4. Backend — apps/api/
Estrutura completa de common/, config/, modules/, jobs/, database/.
Estrutura interna de cada módulo com padrão Controller → Service → Repository → DTO → Entity.

### 5. Convenções de Nomenclatura
Tabela completa por contexto (pastas, componentes, hooks, classes, DTOs, testes, enums, constantes).

### 6. Cache Redis — Convenção de Chaves
Todas as chaves com padrão, TTL e contexto.

### 7. Mapeamento Módulo → Error Prefix
Tabela com módulo, prefix e queues.

### 8. Changelog
Tabela com Data, Versão e Descrição.

### 9. Backlog de Pendências
Tabela consolidando `[DECISÃO AUTÔNOMA]`, `[DEFINIÇÃO PENDENTE]` e `[SEÇÃO PENDENTE]` com colunas:
- Item
- Marcador
- Seção ou Módulo
- Justificativa/Trade-off
- Impacto (P0, P1, P2)
- Dono
- Status

## 21. Cobertura mínima por seção
- **Visão Geral:** árvore raiz completa com todos os diretórios de primeiro nível.
- **Frontend:** todas as features listadas + estrutura interna + regras de import.
- **Backend:** todos os módulos listados + estrutura interna + padrão por módulo.
- **Nomenclatura:** todos os contextos com convenção + exemplo.
- **Cache:** todas as chaves com padrão + TTL.
- **Error Prefix:** todos os módulos com prefix + queues.

## 22. Autoauditoria obrigatória antes da resposta
Antes de responder, execute internamente este ciclo completo:

1. Gere o rascunho completo.
2. Audite contra a estrutura obrigatória e cobertura mínima.
3. Verifique que toda feature tem barrel export documentado.
4. Verifique que todo módulo backend segue Controller → Service → Repository → DTO → Entity.
5. Verifique que toda chave Redis tem prefixo de produto e TTL.
6. Verifique que todo módulo tem Error Prefix.
7. Verifique que convenções de nomenclatura cobrem todos os contextos.
8. Corrija qualquer ausência, contradição ou lacuna.
9. Só então responda com a versão final.

## 23. Critérios de qualidade obrigatórios
Antes de responder, valide internamente:

**Formato e estrutura**
- há code blocks com árvores de pastas completas e comentários inline
- há tabelas comparativas para convenções e mapeamentos
- separadores visuais entre todas as seções principais

**Consistência e qualidade**
- convenções de nomenclatura são consistentes entre frontend e backend
- estrutura interna de features e módulos segue o padrão definido
- não há estrutura improvisada sem regra documentada

**Cobertura**
- todos os apps do monorepo estão documentados
- todos os packages estão listados com responsabilidade
- todos os módulos de frontend e backend estão mapeados
- cache Redis cobre todos os domínios com TTL
- error prefixes cobrem todos os módulos
- pendências resolvidas por decisão autônoma ou marcadas com trade-off no backlog

## 24. Regra final de resposta
Retorne **somente o documento final** em Markdown.

Não explique seu raciocínio.
Não faça introdução fora do documento.
Não adicione comentários extras.
Não resuma o briefing.
Não diga que vai começar.

Entregue o documento completo, estruturado, consistente, auditável e pronto para uso.
```

---

## 3. Changelog

| **Data** | **Versão** | **Descrição** |
| --- | --- | --- |
| 20/03/2026 | v1.5 | Filosofia de decisão autônoma: seções 8, 16, 20.9 e 23 do Prompt atualizadas com marcadores [DECISÃO AUTÔNOMA], [DEFINIÇÃO PENDENTE] e [SEÇÃO PENDENTE]. |
| 20/03/2026 | v1.3 | Reestruturação completa: convertido de documento final para formato Prompt Generator (padrão Pipeline ShiftLabs). |
| 18/03/2026 | v1.2 | Padronização da abertura do documento para o formato TL;DR → Persona. |
| 17/03/2026 | v1.1 | Redis keys corrigidas com prefixo `portal:`. Chaves de auth adicionadas. |
| 17/03/2026 | v1.0 | Criação. Monorepo completo com apps/web, apps/api, apps/ai, packages/, prisma/. |