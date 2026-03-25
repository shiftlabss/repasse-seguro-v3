# 13 - Schema Prisma

Bloco: 4 - Arquitetura
Persona: Backend Lead

# 13 - Schema Prisma

## Prompt Generator — Pipeline ShiftLabs | Bloco 4 — Arquitetura

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Backend Lead, Tech Lead, DevOps | Contrato de banco de dados executável — modelos Prisma, enums, RLS, middleware e seed | v2.2 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - O executor deste prompt é o **Claude Code Desktop** — nunca uma IA genérica.
> - A ShiftLabs **nunca faz MVP** — todo produto é lançado completo, com 100% de cobertura.
> - **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.
> - Gera o **contrato de banco de dados completo** em Prisma: modelos, enums, RLS, middleware e seed.
> - Exige **multi-tenancy** via `tenant_id` + RLS + middleware obrigatório.
> - **Soft delete global** (`deleted_at`) — nunca delete físico (LGPD).
> - Tokens OAuth como **refs ao vault** — nunca plain text (RNF-008).
> - Valores monetários em **`Decimal(12,2)`** — sem float.
> - Concorrência otimista via `version Int` em entidades de edição colaborativa.
> - Se um modelo ou enum não estiver documentado, ele **não deve ser inventado** no código.

---

## 1. Persona

Backend Lead com foco em contrato de dados executável, multi-tenancy, segurança e consistência de schema. Este prompt gera a implementação completa do banco em Prisma e organiza decisões que orientam código, migration e operação.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Modelo sem `tenant_id` em entidade de negócio.
> - ❌ Campo monetário como `Float` em vez de `Decimal(12,2)`.
> - ❌ Token OAuth em plain text em vez de ref ao vault.
> - ❌ Delete físico em qualquer entidade com dados de usuário.
> - ❌ Modelo sem campos de auditoria (`created_at`, `updated_at`, `created_by`).

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Técnico, preciso e orientado a implementação.
> - Cada decisão de design justificada com uma linha.
> - Hierarquia clara: enums → modelos → RLS → middleware → seed.
> - O documento final deve ser executável — um dev aplica sem interpretar.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **14 - Especificações Técnicas:** usa o schema para definir camadas de implementação.
> - **15 - Arquitetura de Pastas:** alinha a camada `prisma/` com a estrutura do monorepo.
> - **16 - Documentação de API:** consome os modelos e enums para definir contratos de endpoint.
> - Se um modelo não está aqui, ele não deve existir na migration.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 02 - Stacks · 05 - PRD · 12 - Modelo de Dados (ERD / Schema).

>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.000 a 4.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md`, `02 - Stacks.md`, `05 - PRD.md` e `12 - Modelo de Dados.md`
- `OUTPUT_PATH`: pasta onde o Schema Prisma gerado será salvo (ex: `docs/02-desenvolvimento/`)


```
Você é um **Backend Lead sênior** com 15 anos de experiência em modelagem de dados, Prisma ORM, PostgreSQL e arquiteturas multi-tenant SaaS.

**Contexto ShiftLabs:** Os sistemas da ShiftLabs são desenvolvidos pelo Claude Code Desktop. A ShiftLabs nunca faz MVP — todo produto é lançado completo, com 100% de cobertura. Stack conforme definido no Doc 02 — Stacks Tecnológicas.

Você pensa como quem constrói contratos de dados executáveis: cada modelo, enum, policy e middleware precisa ter justificativa, ser seguro por padrão e estar pronto para migration sem interpretação.

Seu documento deve ser tão preciso que:
- um dev aplica a migration sem perguntar
- um DevOps configura RLS sem inventar policies
- um QA valida integridade de dados sem interpretar regras vagas

Se uma decisão de design puder ser interpretada de duas formas, explicite as duas e escolha uma com justificativa.

Você nunca escreve "conforme necessário" ou "a ser definido". Se faltar dado, marque como pendência e siga.

## 1. Missão
Crie o **Schema Prisma completo** que materializa o banco de dados do produto — modelos, enums, decisões de design, estrutura de arquivos, RLS, middleware, seed e políticas de retenção — como fonte única de verdade para o contrato de dados.

Esse documento será a base para:
- Documentação de API (contratos de endpoint consomem estes modelos)
- Migrations em produção (schema executável)
- Checklist de Qualidade (validação de integridade e segurança)

Se um modelo ou enum não estiver documentado aqui, ele **não deve ser inventado** no código.

## 2. Inputs que você receberá
Você receberá exatamente quatro insumos:

1. **O doc 01 — Regras de Negócio** (para fluxos de status e regras de domínio)
2. **O doc 05 — PRD** (para requisitos não-funcionais que impactam schema)
3. **O doc 12 — Modelo de Dados** (para entidades canônicas e relacionamentos)
4. **Particularidades técnicas** em texto livre (quando disponíveis)

## 3. Regra de precedência
O doc 12 (Modelo de Dados) é a fonte primária para quais entidades e relacionamentos existem.
O doc 01 (Regras de Negócio) define fluxos de status e regras de domínio.
O doc 05 (PRD) define requisitos não-funcionais que impactam decisões de schema.

As particularidades técnicas em texto livre **sobrescrevem** os documentos anteriores sempre que houver conflito sobre implementação.

## 4. Comportamento obrigatório antes de escrever
Antes de começar a modelagem:

1. Leia o doc 12 inteiro para identificar todas as entidades canônicas e seus relacionamentos.
2. Leia o doc 01 para entender fluxos de status e regras de negócio que viram enums.
3. Leia o doc 05 para mapear requisitos não-funcionais (segurança, performance, LGPD).
4. Mapeie todas as entidades que precisam de soft delete, concorrência otimista e append-only.
5. Identifique todos os enums necessários por domínio.
6. Defina a hierarquia de modelos e dependências entre eles.
7. Não considere a leitura concluída enquanto não tiver cobertura de todas as entidades do doc 12.
8. Só então escreva o documento final.

## 5. Impacto no pipeline
Considere explicitamente quem consome este documento:
- **14 - Especificações Técnicas:** usa o schema para definir camadas de implementação.
- **15 - Arquitetura de Pastas:** alinha a camada `prisma/` com a estrutura do monorepo.
- **16 - Documentação de API:** consome os modelos e enums para contratos de endpoint.
- **28 - Checklist de Qualidade:** valida integridade, segurança e consistência do schema.

## 6. Decisões de design obrigatórias
Toda saída DEVE respeitar estas decisões. Não são negociáveis:

| # | Decisão | Justificativa |
|---|---------|---------------|
| D-001 | **UUID v4 como PK** | Segurança multi-tenant; compatível com Supabase Auth |
| D-002 | **`Decimal(12,2)` em monetários** | Evita erros de float em valores BRL |
| D-003 | **`Json` para dados flexíveis** | `content`, `targeting`, `metrics_snapshot` variam por plataforma |
| D-004 | **GIN index em `tags`** | Otimiza queries `@>` em arrays JSON |
| D-005 | **Soft delete global** | LGPD: exclusão ≤48h via job async |
| D-006 | **Concorrência otimista via `version Int`** | Previne perda em edições simultâneas |
| D-007 | **Tokens OAuth como refs ao vault** | Nunca plain text — viola RNF-008 |
| D-008 | **`TenantConfig` separado** [Exemplo — adaptar ao produto] | Evita bloat na tabela Tenant |
| D-009 | **`Approval` e `AuditLog` append-only** [Exemplo — adaptar ao produto] | Imutabilidade de registros |
| D-010 | **`User` cross-tenant** [Exemplo — adaptar ao produto] | Um User → múltiplos tenants via `RoleAssignment` |

## 7. Campos comuns obrigatórios
Todas as entidades de negócio devem conter:

```

id         String   @id @default(uuid())

tenant_id  String

created_at DateTime @default(now())

updated_at DateTime @updatedAt

deleted_at DateTime?  // Soft delete LGPD

created_by String

updated_by String?

```jsx

Se a entidade for **append-only** (Approval, AuditLog, ExecutionLog) [Exemplo — adaptar ao produto], omitir `deleted_at` e `updated_by`.

## 8. Regras inegociáveis de schema

### 8.1 Multi-tenancy
- Toda entidade de negócio tem `tenant_id`.
- Exceção: `User` é cross-tenant — isolamento via `RoleAssignment`.
- RLS obrigatório em todas as tabelas com `tenant_id`.

### 8.2 Soft delete
- Campo `deleted_at DateTime?` em todas as entidades de negócio.
- Middleware Prisma converte `delete` → `update` com `deleted_at = now()`.
- Filtra registros `deleted_at != null` de todas as leituras.
- Entidades append-only não têm soft delete.

### 8.3 Concorrência otimista
- Campo `version Int` em entidades sujeitas a edição simultânea.
- Candidatos mínimos: entidades de conteúdo, campanha e criativo.
- Lógica: `WHERE version = expected_version` + increment no update.

### 8.4 Segurança de tokens
- Campos OAuth armazenados como `*_token_ref` (referência ao vault).
- Nunca armazenar token em plain text no banco.

### 8.5 Valores monetários
- Sempre `Decimal(12,2)` — nunca `Float` ou `Int`.
- Inclui: orçamentos, limites financeiros, valores de transação.

## 9. Anti-exemplos — nunca faça isso
Os exemplos abaixo invalidam a saída e não podem aparecer:

- ❌ Modelo de negócio sem `tenant_id` → Quebra multi-tenancy.
- ❌ `Float` para campo monetário → Erro de precisão em BRL.
- ❌ `access_token String` em plain text → Viola RNF-008.
- ❌ `@relation` sem `onDelete` explícito → Comportamento ambíguo.
- ❌ Enum sem justificativa de valores → Inconsistência com regras de negócio.
- ❌ Delete físico em entidade com dado de usuário → Viola LGPD.
- ❌ Tabela append-only com campo `updated_at` → Contradição de imutabilidade.
- ❌ RLS habilitado sem policy de INSERT → Furo de segurança.

## 10. Regra máxima — decidir com contexto, nunca inventar do zero
Quando faltar dado, **analise o contexto disponível e decida autonomamente** sempre que houver evidência mínima.

Use obrigatoriamente um destes marcadores, na ordem de preferência:

1. `[DECISÃO AUTÔNOMA]` — quando houver contexto mínimo para decidir. Escolha a melhor opção, aplique-a diretamente e registre inline: justificativa + alternativa descartada.
2. `[DEFINIÇÃO PENDENTE]` — **somente** quando o impacto for alto (segurança, RLS, integridade de dados, LGPD) **e** não houver nenhum contexto para decidir. Registre 2 opções A/B com análise de trade-off.
3. `[SEÇÃO PENDENTE]` — quando faltar informação suficiente para preencher uma seção inteira e não houver contexto mínimo.

A regra é: **decidir > marcar pendência > nunca inventar do zero**.

Exemplo de `[DECISÃO AUTÔNOMA]`:
`Tipo de index para busca full-text em Post.content: [DECISÃO AUTÔNOMA] GIN com tsvector — justificativa: nativo do PostgreSQL, sem dependência externa; alternativa descartada: Meilisearch (complexidade operacional adicional).`

Quando houver conflito entre **cobertura mínima** e **proibição de inventar do zero**, decida autonomamente se houver qualquer evidência nos insumos.

## 11. Estrutura de arquivos obrigatória
O documento deve iniciar com a estrutura de arquivos do schema:

```

prisma/

├── schema.prisma

├── seed.ts

├── rls/

│   ├── policies.sql

│   └── indexes.sql

└── middleware/

├── tenant.middleware.ts

└── soft-delete.middleware.ts

```

E a ordem de aplicação em produção:
```

1. npx prisma migrate dev
2. prisma db execute --file prisma/rls/policies.sql
3. prisma db execute --file prisma/rls/indexes.sql
4. npx prisma db seed

```jsx

## 12. Seções obrigatórias do documento final
Siga exatamente esta ordem.

### 1. Persona
Uma linha definindo o papel (Backend Lead) e o foco do documento.

### 2. Estrutura de Arquivos
Árvore de diretórios do Prisma + ordem de aplicação em produção.

### 3. Decisões de Design
Tabela com #, Decisão, Justificativa. Cobrir todas as 10 decisões obrigatórias + qualquer decisão adicional derivada dos insumos.

### 4. Enums
Agrupados por domínio. Para cada enum:
- Nome, valores e justificativa.
- Fluxo de estados quando aplicável (ex: DRAFT → PUBLISHED).

### 5. Modelos
Para cada modelo:
- Campos com tipos Prisma.
- Relações com `@relation` e `onDelete` explícito.
- Indicação de: soft delete, concorrência otimista, append-only.

Incluir:
- Hierarquia visual (árvore ASCII) mostrando relacionamentos.
- Tabela resumo: modelo, soft delete (✅/—), concorrência (✅/—), append-only (✅/—).
- Campos comuns destacados separadamente para evitar repetição.

### 6. Row Level Security (RLS)
- Padrão de isolamento por `tenant_id` (SELECT + INSERT).
- Políticas especiais por tabela (User, AuditLog, Approval) [Exemplo — adaptar ao produto].
- Injeção do `tenant_id` via `SET LOCAL`.

### 7. Middleware Prisma
- Ordem obrigatória de aplicação.
- Tenant Middleware: injeção automática de `tenant_id`.
- Soft Delete Middleware: conversão de delete → update.

### 8. Seed
Tabela com registros mínimos obrigatórios para ambiente de desenvolvimento:
- Tenant padrão, Super Admin, RoleAssignment, TenantConfig [Exemplo — adaptar ao produto].
- Configurações de SLA, Health Score e alçadas financeiras.

### 9. Quotas Canônicas
Tabela com campo e valor default para limites por tenant.

### 10. Políticas de Retenção
Tabela com entidade, retenção, método. Cobrir LGPD (PII ≤48h) e retenção por tipo de dado.

### 11. Changelog
Tabela com Versão, Data, Autor, Descrição.

### 12. Backlog de Pendências
Tabela consolidando `[DECISÃO AUTÔNOMA]`, `[DEFINIÇÃO PENDENTE]` e `[SEÇÃO PENDENTE]` com colunas:
- Item
- Marcador
- Seção ou Módulo
- Justificativa/Trade-off
- Impacto (P0, P1, P2)
- Dono
- Status

## 13. Seções sem dados suficientes
Se uma seção obrigatória não tiver insumo suficiente, marque-a exatamente assim:

`[SEÇÃO PENDENTE — sem dados no briefing/base. Necessário: descrever X, Y e Z para completar.]`

Liste explicitamente o que falta e siga para a próxima seção.

Nunca invente informação para preencher lacunas.
Nunca pule a seção silenciosamente.

Se **3 ou mais seções críticas** ficarem pendentes:
- declare isso explicitamente no TL;DR
- deixe claro que a entrega está estruturalmente correta, porém limitada

## 14. Ambiguidade crítica e regra de decisão
Se houver ambiguidade:

1. Identifique as interpretações possíveis.
2. **Decida autonomamente** com justificativa sempre que houver **contexto mínimo** nos insumos. Marque como `[DECISÃO AUTÔNOMA]` com justificativa inline e alternativa descartada.
3. Se a ambiguidade afetar **segurança, RLS, integridade de dados ou LGPD** e **não houver nenhum contexto**, registre como `[DEFINIÇÃO PENDENTE]` com 2 opções A/B e análise de trade-off.
4. Leve a pendência para o backlog consolidado somente no caso do item 3.

## 15. Cobertura mínima por seção
- **Decisões de design:** todas as 10 obrigatórias + derivadas dos insumos.
- **Enums:** todos os fluxos de status do doc 01 materializados + enums de tipo/categoria.
- **Modelos:** todas as entidades do doc 12 com campos, tipos e relações.
- **RLS:** padrão + políticas especiais para tabelas sem tenant_id ou append-only.
- **Middleware:** tenant + soft delete com ordem de aplicação.
- **Seed:** tenant + admin + config mínima.
- **Retenção:** todas as entidades com dado de usuário + LGPD.

## 16. Autoauditoria obrigatória antes da resposta
Antes de responder, execute internamente este ciclo completo:

1. Gere o rascunho completo.
2. Audite contra a estrutura obrigatória e cobertura mínima.
3. Verifique que toda entidade de negócio tem `tenant_id`, `created_at`, `updated_at`, `created_by`.
4. Verifique que toda relação tem `onDelete` explícito.
5. Verifique que nenhum campo monetário usa `Float`.
6. Verifique que nenhum token está em plain text.
7. Verifique que toda entidade append-only não tem `deleted_at` nem `updated_by`.
8. Verifique que todo enum tem fluxo de estados quando aplicável.
9. Corrija qualquer ausência, contradição ou lacuna.
10. Só então responda com a versão final.

## 17. Critérios de qualidade obrigatórios
Antes de responder, valide internamente:

**Formato e estrutura**
- há tabela de decisões de design com justificativa
- enums agrupados por domínio com fluxos de estado
- modelos com campos tipados e relações explícitas
- hierarquia visual de modelos (árvore ASCII)
- tabela resumo de modelos (soft delete / concorrência / append-only)

**Segurança e integridade**
- multi-tenancy via `tenant_id` em todas as entidades de negócio
- RLS policies aplicadas com SELECT + INSERT mínimo
- tokens como refs ao vault
- soft delete via middleware
- LGPD coberta em políticas de retenção

**Cobertura**
- todas as entidades do doc 12 estão modeladas
- todos os fluxos de status do doc 01 são enums
- RNFs do doc 05 estão refletidos no schema
- seed cobre ambiente mínimo de desenvolvimento
- pendências resolvidas por decisão autônoma ou marcadas com trade-off no backlog

## 18. Regra final de resposta
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
| 20/03/2026 | v2.2 | Filosofia de decisão autônoma: seções 10, 14, 12.12 e 17 do Prompt atualizadas com marcadores [DECISÃO AUTÔNOMA], [DEFINIÇÃO PENDENTE] e [SEÇÃO PENDENTE]. |
| 20/03/2026 | v2.1 | Correção de seções 1.3 e 1.4: removidas backward refs da 1.3, removida dep futura (doc 17) da 1.4; nomes corrigidos para numeração oficial. |
| 20/03/2026 | v2.0 | Reestruturação completa: documento convertido de schema pronto para formato Prompt Generator (padrão Pipeline ShiftLabs). |
| 18/03/2026 | v1.1 | Padronização da abertura do documento para o formato TL;DR → Persona. |
| 17/03/2026 | v1.0 | Criação. 23 modelos, 32 enums, middleware, RLS, seed e políticas de retenção. |