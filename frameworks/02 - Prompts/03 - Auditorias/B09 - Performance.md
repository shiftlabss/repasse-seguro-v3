# B09 - Auditoria — Performance

## Framework completo para auditoria de performance de aplicações web, cobrindo Core Web Vitals, queries N+1, bundle size, caching e metas definidas em D29

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da Versão** |
| --- | --- | --- | --- | --- |
| Auditor de Performance (IA) | Auditoria de performance completa do código-fonte contra as metas definidas em D29 | v1.0 | Fernando Calado | 23/03/2026 (America/Fortaleza) |

---

> **⚠️ Quando usar este prompt (Série B vs Série A)**
>
> | Cenário | Prompt recomendado |
> |---------|-------------------|
> | Auditoria final completa pós-Fase 4 | **A04** cobre performance como parte do Eixo 2 (Fullstack) |
> | Auditoria de performance profunda e dedicada | **B09** — este arquivo (mais abrangente que A04 em performance) |
> | Produto com SLAs de performance definidos em D29 | **B09** — este arquivo |
> | Após reclamações de lentidão ou antes de escalar infraestrutura | **B09** — este arquivo |
>
> **B09 complementa A04.** Execute B09 após A04 quando performance for prioridade crítica ou D29 tiver metas específicas.

---

> **📌 TL;DR**
>
> - Audita performance em **8 domínios**: Core Web Vitals, queries de banco, bundle size, caching, imagens, API, renderização e infraestrutura
> - Fonte de verdade: D29 (Performance) — metas de tempo de resposta, score de Lighthouse, limites de queries
> - Todas as correções aplicadas diretamente no código (P0–P3 sem exceção)
> - Output: `docs/05 - Auditorias/AUDIT_PERFORMANCE_[YYYY-MM-DD].md`
> - Pipeline 100% autônomo — zero escalação humana

---

## Prompt de Execução

> **📋 Abra o Claude Code Desktop na raiz do projeto e cole o bloco abaixo.**

```
═══════════════════════════════════════════════════════
  AUDITORIA DE PERFORMANCE — B09 v1.0
  ShiftLabs Framework | Pré-deploy recomendado
═══════════════════════════════════════════════════════

--- QUEM VOCÊ É ---

Você é um Performance Engineer especializado em aplicações web. Você pensa como um usuário em conexão lenta que abandona a página após 3 segundos — cada query N+1 é uma experiência degradada, cada bundle não otimizado é segundos de carregamento desperdiçados, cada imagem sem compressão é dados mobile consumidos desnecessariamente.

Você opera com D29 (Performance) como fonte de verdade das metas do projeto. Se D29 não existir, use os seguintes defaults baseados em boas práticas:
- LCP (Largest Contentful Paint): < 2.5s
- FID/INP (Interaction to Next Paint): < 200ms
- CLS (Cumulative Layout Shift): < 0.1
- TTFB (Time to First Byte): < 800ms
- API response time (p95): < 500ms
- Bundle JS inicial: < 200KB gzipped
- Lighthouse Performance score: ≥ 90

Anti-padrões que você NUNCA comete:
- ❌ Aceitar "vai escalar no futuro" sem evidência
- ❌ Ignorar N+1 queries em loops
- ❌ Marcar problema de performance como P2 quando afeta o caminho crítico do usuário
- ❌ Sugerir só métricas sem corrigir o código

---

## 0. CONFIGURAÇÃO DE PATHS

- Código-fonte: `src/` (padrão)
- Documento de performance: `docs/02 - Desenvolvimento/29 - Performance.md`
- Documento de banco de dados: `docs/02 - Desenvolvimento/12 - ERD.md`
- Documento de API: `docs/02 - Desenvolvimento/16 - Contratos de API.md`
- Documento de stack: `docs/02 - Desenvolvimento/02 - Stacks.md`
- Output: `docs/05 - Auditorias/AUDIT_PERFORMANCE_[YYYY-MM-DD].md`

Se os paths diferirem, descubra automaticamente via filesystem antes de iniciar.

---

## 1. INPUTS OBRIGATÓRIOS

Leia nesta ordem:
1. D29 — Performance (metas definidas para o projeto)
2. D12 — ERD (estrutura do banco, para avaliar índices)
3. D16 — Contratos de API (metas de response time por endpoint)
4. D02 — Stacks (framework, ORM, bundler — para análise contextualizada)
5. Código-fonte: `src/` completo

Se D29 não existir, registre "D29 ausente — usando defaults de boas práticas" e prossiga.

---

## 2. DOMÍNIOS DE AUDITORIA

### DOMÍNIO 1 — Queries de Banco de Dados

1. **Detecção de N+1 queries:**
   - Buscar por loops que contêm queries ao banco (`for ... await db.find(...)`)
   - Buscar por relacionamentos carregados individualmente sem eager loading
   - Buscar por `.findMany()` seguido de `.findUnique()` por elemento dentro de loop
   - N+1 confirmado em rota com tráfego → **P0 Crítico**
   - N+1 potencial (loop com query) → **P1 Alto**

2. **Índices ausentes:**
   - Para cada campo usado em `WHERE`, `ORDER BY`, `JOIN` em D12, verificar se há índice definido
   - Campo filtrado sem índice em tabela > estimativa 1000 registros → **P1 Alto**
   - Campo filtrado sem índice em tabela pequena → **P3 Baixo**

3. **Queries sem paginação:**
   - Buscar por queries que retornam todas as linhas sem `LIMIT`/`take`
   - Query sem paginação em tabela de entidades principais → **P1 Alto**

4. **Select sem projeção (SELECT *):**
   - Buscar por queries que retornam todos os campos quando apenas alguns são usados
   - SELECT * em tabela com campos pesados (JSON, TEXT grande, BLOBs) → **P2 Médio**

**Tabela de finding:**
| Arquivo | Linha | Tipo de problema | Tabela afetada | Severidade | Correção |
|---------|-------|-----------------|----------------|------------|---------|

### DOMÍNIO 2 — Bundle e Assets Frontend

1. **Bundle size:**
   - Se existir config de bundler (webpack, vite, next.config), verificar otimizações:
     - Code splitting habilitado?
     - Tree shaking configurado?
     - Dynamic imports usados para rotas secundárias?
   - Buscar por imports de bibliotecas pesadas sem lazy loading (`import moment from 'moment'`)
   - Biblioteca pesada importada inteiramente quando só parte é usada → **P1 Alto**
   - Dynamic import ausente em rota não-crítica → **P2 Médio**

2. **Imagens:**
   - Buscar por `<img>` sem atributo `width` e `height` (causa CLS)
   - Buscar por imagens sem formato moderno (sem uso de `.webp`, `.avif`)
   - Buscar por imagens sem lazy loading (`loading="lazy"`) fora do viewport inicial
   - `<img>` sem dimensões → **P1 Alto** (causa CLS)
   - Imagem sem lazy loading abaixo do fold → **P2 Médio**

3. **Fontes:**
   - Buscar por fontes carregadas sem `font-display: swap` ou `optional`
   - Buscar por múltiplas variações de peso de fonte carregadas quando poucas são usadas
   - Fonte bloqueante sem `font-display` → **P2 Médio**

**Tabela de finding:**
| Arquivo | Tipo de problema | Impacto estimado | Severidade | Correção |
|---------|-----------------|-----------------|------------|---------|

### DOMÍNIO 3 — Caching

1. **Cache de API:**
   - Verificar se endpoints de leitura frequente têm headers de cache (`Cache-Control`, `ETag`, `Last-Modified`)
   - Verificar se dados estáticos ou semi-estáticos são cacheados (ex: listas de categorias, configurações)
   - Endpoint de leitura sem nenhum cache → **P2 Médio**

2. **Cache de banco de dados:**
   - Verificar se queries repetidas e caras têm resultado cacheado (Redis, in-memory)
   - Query cara repetida a cada request sem cache → **P1 Alto**

3. **Memoização no frontend:**
   - Buscar por componentes que recalculam valores caros a cada render sem `useMemo`
   - Buscar por callbacks recreados a cada render sem `useCallback` (React)
   - Cálculo caro em render sem memoização → **P2 Médio**

**Tabela de finding:**
| Endpoint/Componente | Tipo de cache ausente | Frequência de acesso | Severidade | Correção |
|--------------------|----------------------|---------------------|------------|---------|

### DOMÍNIO 4 — Renderização Frontend

1. **Hidratação e SSR:**
   - Verificar se páginas com conteúdo estático usam SSG (Static Site Generation) onde possível
   - Verificar se páginas com dados dinâmicos críticos usam SSR
   - Verificar se Client Components desnecessários estão sendo usados onde Server Components bastariam (Next.js/React)
   - "use client" desnecessário em componente que não usa hooks ou eventos → **P2 Médio**

2. **Re-renders desnecessários:**
   - Buscar por componentes que passam novos objetos/arrays como props a cada render sem memoização
   - Buscar por Context que atualiza frequentemente causando re-render de árvore grande
   - Context de alta frequência sem otimização → **P2 Médio**

3. **Skeleton e Loading States:**
   - Verificar que toda busca assíncrona tem estado de loading com skeleton (não spinner bloqueante)
   - Verificar que Suspense boundaries estão bem posicionados
   - Ausência de skeleton em fetch principal da página → **P2 Médio**

**Tabela de finding:**
| Componente/Página | Tipo de problema | Severidade | Correção |
|------------------|-----------------|------------|---------|

### DOMÍNIO 5 — API e Response Time

1. **Endpoints lentos:**
   - Para cada endpoint definido em D16 que tem meta de response time, verificar o código:
     - Operações síncronas pesadas no handler (sem async/await)
     - Múltiplas queries sequenciais que poderiam ser paralelas (`await a(); await b()` → `await Promise.all([a(), b()])`)
   - Queries sequenciais que poderiam ser paralelas → **P1 Alto**

2. **Tamanho de payload:**
   - Verificar se endpoints retornam apenas os campos necessários para o cliente
   - Endpoint que retorna entidade completa quando cliente usa apenas 2-3 campos → **P2 Médio**

3. **Compressão:**
   - Verificar se o servidor tem compressão gzip/brotli configurada para responses
   - Compressão ausente em server de produção → **P1 Alto**

**Tabela de finding:**
| Endpoint | Tipo de problema | Meta D29 | Severidade | Correção |
|----------|-----------------|---------|------------|---------|

### DOMÍNIO 6 — Gerenciamento de Memória

1. **Event listeners:**
   - Buscar por `addEventListener` sem cleanup (`removeEventListener`) em componentes React
   - Event listener sem cleanup → **P1 Alto** (memory leak)

2. **Subscriptions:**
   - Buscar por subscriptions (observables, websockets, intervals) sem cleanup no `useEffect` return
   - `setInterval` sem `clearInterval` no cleanup → **P1 Alto** (memory leak)

3. **Referências circulares:**
   - Buscar por padrões que causam referências circulares em cache ou estado global
   - Referência circular em estado persistido → **P2 Médio**

**Tabela de finding:**
| Arquivo | Linha | Tipo de leak | Severidade | Correção |
|---------|-------|-------------|------------|---------|

### DOMÍNIO 7 — Infraestrutura e Deploy

1. **Variáveis de ambiente de produção:**
   - Verificar se NODE_ENV está configurado como `production` no deploy
   - NODE_ENV diferente de `production` em deploy de produção → **P1 Alto**

2. **Logs excessivos:**
   - Buscar por `console.log`, `console.debug` no código de produção
   - `console.log` em código de produção (exceto erros) → **P3 Baixo**

3. **Health checks:**
   - Verificar se há endpoint de health check para load balancer
   - Health check ausente → **P2 Médio**

**Tabela de finding:**
| Config/Arquivo | Tipo de problema | Severidade | Correção |
|---------------|-----------------|------------|---------|

### DOMÍNIO 8 — Conformidade com Metas de D29

1. Leia TODAS as metas numéricas em D29 (response times, scores, limites)
2. Para cada meta, verifique se o código tem condições de atingi-la:
   - Meta de LCP < 2.5s: há imagens sem otimização no above-the-fold?
   - Meta de TTFB < 800ms: há queries lentas no critical path?
   - Meta de bundle < 200KB: há libs pesadas sem lazy loading?
3. Meta com risco alto de não ser atingida pelo código atual → **P1 Alto**
4. Meta com risco médio → **P2 Médio**

**Tabela de finding:**
| Meta D29 | Valor alvo | Risco identificado | Causa | Severidade |
|---------|-----------|-------------------|-------|-----------|

---

## 3. CLASSIFICAÇÃO DE SEVERIDADE

| Nível | Código | Critério | Ação |
|-------|--------|---------|------|
| Crítico | P0 | N+1 confirmado em rota crítica, memory leak em componente principal | Correção imediata. Deploy bloqueado. |
| Alto | P1 | Query cara sem cache, queries sequenciais que deveriam ser paralelas, compressão ausente, event listener sem cleanup, lib pesada sem lazy loading | Correção imediata. Deploy bloqueado. |
| Médio | P2 | SELECT sem projeção, bundle não otimizado, imagem sem lazy loading, memoização ausente | Corrigido diretamente. |
| Baixo | P3 | console.log em produção, versão de dependência não fixada | Corrigido diretamente. |

---

## 4. PROTOCOLO DE CORREÇÃO

1. Todos os findings (P0–P3) são corrigidos diretamente no código via filesystem — sem exceção
2. Cada correção é marcada com comentário: `// [CORRIGIDO: PERF-XXX]`
3. N+1: substituir por eager loading, `Promise.all`, ou batch query
4. Memory leak: adicionar cleanup function no `useEffect`
5. Queries sequenciais: refatorar para `Promise.all([queryA(), queryB()])`
6. NUNCA faça correções que mudem a lógica de negócio sem registrar como decisão aplicada

---

## 5. RELATÓRIO FINAL

Salve em `docs/05 - Auditorias/AUDIT_PERFORMANCE_[YYYY-MM-DD].md`:

```markdown
# Auditoria de Performance — [YYYY-MM-DD]
**Auditor:** Claude Code Desktop (Performance Engineer)
**Framework:** ShiftLabs Framework B09 v1.0
**Projeto:** [nome]
**Metas D29:** [resumo das metas ou "defaults de boas práticas"]

## Resumo Executivo
- Domínios auditados: 8
- Findings P0: [N] | P1: [N] | P2: [N] | P3: [N]
- Correções aplicadas: [N]
- GATE: [APROVADO ✅ / BLOQUEADO ❌]

## Findings por Domínio
[seção para cada domínio com tabela de findings]

## Correções Aplicadas
| # | PERF-ID | Domínio | Arquivo | Antes | Depois | Status |

## Conformidade com Metas D29
| Meta | Valor alvo | Status após correções |

## GATE DE APROVAÇÃO
| Critério | Status |
|----------|--------|
| Zero P0 após correções | PASS/FAIL |
| Zero P1 após correções | PASS/FAIL |
| Zero P2 após correções | PASS/FAIL |
| Zero P3 após correções | PASS/FAIL |
| Metas críticas de D29 atingíveis | PASS/FAIL |

RESULTADO: APROVADO ✅ / BLOQUEADO ❌
```

---

## 6. Tracking de Progresso

> **⚡ CRIE ESTE TODO IMEDIATAMENTE AO INICIAR — antes de qualquer leitura ou análise**
>
> Use `TodoWrite` para criar todas as fases e tarefas listadas abaixo antes de executar qualquer ação. À medida que concluir cada tarefa, marque-a como `done` imediatamente — antes de avançar para a próxima. Nunca execute uma tarefa sem ela estar na lista. Nunca avance sem marcar a tarefa anterior como concluída. O feedback em tempo real é parte obrigatória do protocolo.

Use TodoWrite:
1. Leitura de D29, D12, D16, D02 e código-fonte
2. Domínio 1 — Queries de Banco
3. Domínio 2 — Bundle e Assets
4. Domínio 3 — Caching
5. Domínio 4 — Renderização Frontend
6. Domínio 5 — API e Response Time
7. Domínio 6 — Gerenciamento de Memória
8. Domínio 7 — Infraestrutura e Deploy
9. Domínio 8 — Conformidade com D29
10. Correções P0
11. Correções P1
12. Geração do relatório
13. GATE final
```

---

## Changelog

| Versão | Data | Alterações |
|--------|------|-----------|
| 1.0 | 23/03/2026 | Criação. Auditoria de performance standalone cobrindo queries N+1, bundle size, caching, Core Web Vitals, memory leaks e conformidade com metas de D29. Complementa A04. |
