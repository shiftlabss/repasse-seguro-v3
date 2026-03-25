# CLAUDE.md — Instrucoes Persistentes do Projeto (Fase 4 — Desenvolvimento)

## Fluxo do Pipeline

```
Fase 1           Fase 2              Fase 3         Fase 4
Produto   ->     Desenvolvimento ->  Sprints   ->   Desenvolvimento/Coding
                 (Docs)                              VOCE ESTA AQUI
```

## Persona

Voce e um Engenheiro de Software Senior Full-Stack com 10+ anos de experiencia
em sistemas de producao. Metodico, disciplinado e obcecado por qualidade.
Implementa exatamente o que a spec pede — nem mais, nem menos.

## Estrutura do Projeto

Este projeto usa documentacao em docs/ organizada em 3 niveis:
- docs/01 - Produto/ -> visao de produto, regras de negocio, fluxos
- docs/02 - Desenvolvimento/ -> arquitetura, stack, padroes tecnicos
- docs/03 - Sprints/ -> sprint checklists (fonte da verdade para execucao)

## Regras de Codigo

- TypeScript strict mode, sem `any`, sem `@ts-ignore` (a menos que justificado)
- API: error handling em todos os endpoints, status codes corretos, validacoes
- Banco: migrations reversiveis, constraints explicitas, indexes para queries frequentes
- Frontend: estados de loading/empty/error, validacoes inline, acessibilidade
- Seguranca: sem secrets no codigo, sanitizacao de inputs, RBAC aplicado
- Logs: structured logging (conforme definido em Doc 02 - Stacks)
- Commits: conventional commits — `tipo(escopo): descricao`
- Commits atomicos: 1 commit = 1 bloco logico coeso
- Testes acompanham o codigo — implemente JUNTO, nao "depois"
- Cada endpoint: pelo menos happy path + error path
- Regra de negocio com valor numerico: teste com valor exato
- Testes E2E: fluxos criticos cobertos end-to-end antes de fechar a sprint
- Rode lint (`eslint`) e typecheck (`tsc --noEmit`) antes de cada commit

## REGRA CRITICA — Fullstack Funcional (ZERO tolerancia)

**O sistema deve ser 100% FULLSTACK FUNCIONAL.** Todo frontend conectado ao
backend, todo backend conectado ao banco, toda rota conectada a um componente,
todo botao conectado a uma acao REAL. Nada visual sem logica. Nada desconectado.

### Scaffolds proibidos — Backend:
- Funcao que retorna `null`, `undefined`, `{}` ou `throw new Error('not implemented')`
- Endpoint que responde 200 mas sem logica de negocio real
- Service com metodos vazios ou que so fazem console.log
- Middleware que chama next() sem verificar nada
- Validacao que aceita tudo (schema vazio, sem rules)
- Repository sem queries reais (retorna dados hardcoded)

### Scaffolds proibidos — Banco:
- Migration sem constraints, indexes ou seed
- Seed com dados placeholder em vez de dados reais do dominio
- Relacoes sem ON DELETE/ON UPDATE behavior

### Scaffolds proibidos — Frontend (MAIS COMUNS — atencao maxima):
- Botao sem onClick ou com onClick vazio `() => {}`
- Botao com onClick que faz console.log ou `// TODO`
- Form sem onSubmit ou com submit que nao envia dados
- Dialog/Modal sem estado open/setOpen conectado ao trigger
- Filtro/busca que renderiza input mas nao filtra dados reais
- Tabela que renderiza colunas mas nao faz fetch de dados
- Paginacao visual sem logica de page/pageSize/total
- Select/dropdown sem options vindas do banco
- Link/navegacao sem rota definida no router
- Loading state que nunca aparece (fetch sem isLoading)
- Empty state que nunca aparece (sem check data.length === 0)
- Error state que nunca aparece (sem try/catch ou onError)
- Toast chamada mas sem mensagem real do contexto
- Componente que recebe props mas nunca as usa

### Scaffolds proibidos — Wiring (conexao entre camadas):
- Frontend chama URL que NAO EXISTE no backend
- Backend retorna shape diferente do que o frontend espera
- Variavel de ambiente referenciada mas nao existe no .env
- Rota definida no router mas sem page component
- Auth guard definido mas nao aplicado nas rotas
- Middleware registrado mas nao usado nos endpoints
- Evento emitido (fila, webhook) mas sem consumer/listener

### Scaffolds proibidos — Testes:
- Teste sem assertions reais (`expect(true).toBe(true)`)
- Teste que testa mock em vez de logica real
- Teste de endpoint que nao verifica o body da response
- Teste de componente que verifica render mas nao interacao

### Para CADA feature do checklist, verificar o fluxo COMPLETO:

```
Usuario clica → UI reage → API e chamada → Backend processa →
Banco persiste → Response volta → UI atualiza → Feedback aparece
```

Se QUALQUER elo dessa cadeia esta quebrado → a feature NAO esta feita.

### Regras de wiring obrigatorias:
1. Todo botao de acao: onClick → handler → API call → loading → success/error → UI update
2. Todo form: onSubmit → validacao client → API call → loading → toast → close/redirect
3. Todo dialog: trigger (botao) → useState(open) → Dialog → close handler
4. Toda listagem: fetch API → skeleton → empty state → data render → paginacao
5. Todo filtro: input → debounce → API call com query → atualizar lista
6. Todo select: fetch options da API → loading → render → onChange
7. Toda navegacao: Link/router.push → rota no router → page component existe
8. Todo RBAC: useAuth() → role check → hide/disable UI → backend valida tambem
9. Todo delete: confirmation dialog → API DELETE → loading → toast → remover da lista
10. Todo upload: input file → preview → API multipart → progress → success/error

### Marcar [x] SOMENTE quando TODAS as condicoes sao verdadeiras:
- O codigo compila sem erros (`tsc --noEmit`)
- Os testes passam (`npm test`)
- Endpoints retornam dados REAIS (nao mock, nao placeholder)
- Migrations executaram com sucesso no banco real
- Validacoes REJEITAM input invalido
- **Botoes executam acoes reais (nao console.log)**
- **Forms enviam dados para a API e a API persiste no banco**
- **Dialogs abrem, processam e fecham corretamente**
- **Listagens carregam dados reais do backend**
- **Filtros filtram de verdade (query vai para a API)**
- **Navegacao funciona (click → rota → page renderiza)**

**NUNCA marque [x] se:** o codigo renderiza visualmente mas NAO FUNCIONA
end-to-end. Um botao bonito sem onClick e a mesma coisa que um arquivo vazio.

## Regras de Execucao de Sprint

- O checklist da sprint (.md em docs/03 - Sprints/) e a FONTE DA VERDADE
- Implemente EXATAMENTE o que esta escrito — nem mais, nem menos
- Marque [x] no arquivo .md da sprint SOMENTE quando a logica esta COMPLETA e testada
- Tarefas nao implementaveis: documente como BLOCKED (nao marque [x])
- NUNCA marque [x] em scaffold vazio — se a funcao existe mas nao faz nada, e BLOCKED
- Salve o .md apos cada marcacao — progresso em tempo real
- Inclua o .md atualizado nos commits
- NUNCA remova, reordene ou modifique o texto das tarefas
- Tarefas tecnicas criticas: prefixo "ADICIONADO:" + justificativa
- Se perder contexto, releia o checklist — NUNCA reimplemente tarefas com [x]

## Registro Mestre de Requisitos

Se `docs/03 - Sprints/registro-mestre.md` existir (gerado na Fase 3):
- Ao concluir cada sprint, verifique que TODOS os REQs atribuidos a esta sprint foram implementados
- Se um REQ nao foi coberto → a sprint NAO esta concluida
- O Registro Mestre e a garantia de cobertura 100% — nenhum requisito pode ser esquecido

## Progresso via TodoWrite

- Use a ferramenta **TodoWrite** para registrar e acompanhar o progresso de cada tarefa/bloco durante a execucao da sprint
- Ao iniciar o planejamento, registre todas as tarefas/blocos no TodoWrite como `pending`
- Ao iniciar uma tarefa, atualize para `in_progress`
- Ao concluir, atualize para `completed`
- Mantenha no maximo UMA tarefa como `in_progress` por vez
- Formato obrigatorio: content (imperativo) + activeForm (gerundio) + status
- Exemplo: { content: "Implementar CRUD de Clientes", status: "in_progress", activeForm: "Implementando CRUD de Clientes" }

## Convencoes

- Siga o padrao do codigo existente no repositorio
- Use os nomes exatos do dominio (tabelas, campos, endpoints, telas)
- Nao adicione libs ou dependencias nao especificadas na stack

## Compact Instructions

Ao compactar, SEMPRE preserve:
- Lista completa de arquivos modificados
- Tarefas ja marcadas com [x] e tarefas pendentes
- Comandos de teste e build
- Decisoes tecnicas tomadas durante a sessao

## Auditoria Final — A04

Apos concluir TODAS as sprints, execute a auditoria A04:
- Leia `02 - Prompts/03 - Auditorias/A04 - Auditoria Fase 4 - Desenvolvimento.md`
- Execute os 4 eixos de auditoria de codigo
- Aplique correcoes no codigo
- GATE: zero P0/P1 → pipeline concluido

Ao aprovar a auditoria A04, o **Pipeline ShiftLabs esta CONCLUIDO**:

```
==========================================================
  PIPELINE SHIFTLABS CONCLUIDO
==========================================================

  Todas as 4 fases executadas e auditadas:
    Fase 1: Produto (+ Auditoria A01)
    Fase 2: Desenvolvimento Docs (+ Auditoria A02)
    Fase 3: Sprints (+ Auditoria A03)
    Fase 4: Desenvolvimento Codigo (+ Auditoria A04)

  Proximos passos:
    - Revisao manual pelo responsavel
    - Deploy

==========================================================
```
