# 00 - Mapa do Pipeline de Auditorias

> **ShiftLabs Framework** | Referência do Sistema de Auditorias | v1.0 | 23/03/2026

---

## TL;DR

- Este documento é o **ponto de entrada** do sistema de auditorias — leia antes de qualquer outra coisa
- O pipeline tem **4 fases** com auditorias obrigatórias (Série A) e opcionais/pontuais (Séries B e C)
- **Série A** = auditorias de fase (rodar uma vez, pós-fase, para liberar o pipeline para a próxima etapa)
- **Série B** = auditorias pontuais (rodar sob demanda, para verificações isoladas ou inline durante uma fase)
- **Série C** = auditorias cross-module (rodar após A04 e/ou ao final de sprints com novos módulos)
- **A05** = re-auditoria incremental (rodar quando um GATE reprova e correções são aplicadas)

---

## Visão Geral do Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PIPELINE SHIFTLABS                               │
│                                                                     │
│  FASE 1            FASE 2            FASE 3            FASE 4       │
│  Produto           Desenvolvimento   Sprints           Código       │
│  (P01-P16)         (D01-D29)         (Checklists)      (src/)       │
│      │                  │                 │                │        │
│      ▼                  ▼                 ▼                ▼        │
│   [A01]             [A02]             [A03]            [A04]        │
│   GATE              GATE              GATE         GATE FINAL       │
│   Fase 1→2          Fase 2→3          Fase 3→4     → Produção       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Série A — Auditorias de Fase (obrigatórias)

| Arquivo | Executa quando | Escopo | GATE |
|---------|---------------|--------|------|
| **A01** | Após Fase 1, antes de iniciar Fase 2 | 16 docs de Produto (P01-P16) | Score ≥ 95%, zero P0–P3 |
| **A02** | Após Fase 2, antes de iniciar Fase 3 | 29 docs de Desenvolvimento (D01-D29) | Zero P0–P3, rastreabilidade ≥ 95% |
| **A03** | Após Fase 3, antes de iniciar Fase 4 | Sprint checklists + Registro Mestre | Zero P0–P3, cobertura 100% |
| **A04** | Após Fase 4, antes de deploy | Código-fonte completo (src/) | Zero P0–P3 |
| **A05** | Após qualquer GATE reprovado + correções aplicadas | Apenas findings corrigidos (incremental) | Confirma zero P0–P3 nos itens re-auditados |
| **A06** | Após Fase 4 ou sob demanda (UX crítica) | Produto implementado — todas as telas, componentes, estados, copy, acessibilidade (src/) | Zero P0–P3 |

**Regra:** Nenhuma fase pode começar sem o GATE da fase anterior estar aprovado.

---

## Série B — Auditorias Pontuais (sob demanda)

Prompts B podem ser usados de **duas formas**:

1. **Inline durante uma fase** — após gerar um documento específico, antes de continuar
2. **Verificação isolada** — quando um documento específico é alterado e você quer re-verificar apenas aquele par

| Arquivo | Foco | Inline em qual fase | Incorporado em |
|---------|------|---------------------|---------------|
| **B01** | Produto × Regras de Negócio | Fase 1 (após gerar RN) | A01 Eixo 3 |
| **B02** | UX × Regras de Negócio | Fase 2 (após gerar D01) | A02 Eixo 2 |
| **B03** | PRD × Regras de Negócio | Fase 2 (após gerar D05) | A02 Eixo 1 |
| **B04** | UX × Mapa de Telas | Fase 2 (após gerar D06) | A02 Eixo 2 |
| **B05** | Código vs Documentação | Fase 4 (por módulo) | A04 Eixo 1 |
| **B06** | Fullstack técnica | Fase 4 (por módulo) | A04 Eixo 2 |
| **B07** | UI/UX visual (profundo) | Fase 4 (por módulo) | A04 Eixo 3 |
| **B08** | Segurança (OWASP) | Fase 4 ou pré-deploy | Complementa A04 |
| **B09** | Performance | Fase 4 ou pré-deploy | Complementa A04 |
| **B10** | LGPD e Compliance | Fase 4 ou pré-deploy | Standalone |
| **B11** | Testes (coverage, qualidade, E2E) | Fase 4 ou pré-deploy | Complementa A04 |
| **B12** | CI/CD (pipeline, deploy, rollback) | Fase 4 ou pré-deploy | Complementa A04 |

**Regra:** Se você está executando o pipeline completo, use as auditorias A. Use B apenas para verificações pontuais ou quando um documento específico for alterado após a auditoria A ter sido executada.

---

## Série C — Auditorias Cross-Module (consistência entre módulos)

Prompts C verificam que o mesmo padrão foi aplicado em **todos os módulos** do projeto.

| Arquivo | Foco | Source of Truth | Executa quando |
|---------|------|----------------|----------------|
| **C01** | Regras de Negócio entre módulos | D01 | Após A04 (obrigatório) + ao final de cada sprint S3+ |
| **C02** | Stack tecnológico entre módulos | D02 | Após A04 (obrigatório) + sempre que package.json for alterado |
| **C03** | Motion/animações entre módulos | D04 | Após A04 (obrigatório) + após sprints com novos componentes animados |
| **C04** | Notificações e emails entre módulos | D22 | Após A04 (obrigatório) + sempre que triggers de notificação forem alterados |

**Regra:** C01-C04 são **obrigatórios antes do deploy final**, após A04 ter sido aprovado.

---

## Relação entre Séries: B supersede / A consolida

```
B01 ──────────────────────────────────────┐
B02 ──────────────────────────┐           │
B03 ──────────────────────────┤           │
B04 ──────────────────────────┘           │
                              A02         A01
B05 ──────────────────────────┐
B06 ──────────────────────────┤  A04
B07 ──────────────────────────┘
B08 ─── Complementa A04 (Segurança)
B09 ─── Complementa A04 (Performance)
B10 ─── Standalone (LGPD)
B11 ─── Complementa A04 (Testes)
B12 ─── Complementa A04 (CI/CD)
```

---

## Fluxo de decisão: qual prompt usar?

```
Quero auditar o projeto...
│
├── ...ao final de uma fase inteira?
│   └── Use a Série A correspondente (A01, A02, A03, A04)
│
├── ...de forma incremental após correções num GATE reprovado?
│   └── Use A05
│
├── ...um documento específico que foi alterado após a auditoria A?
│   └── Use o prompt B correspondente ao documento
│
├── ...a consistência entre módulos no código?
│   └── Use Série C (C01-C04)
│
└── ...um projeto legado sem o framework ShiftLabs?
    └── Use B06 (Fullstack) + B07 (UI/UX) + B08 (Segurança) + B09 (Performance)
```

---

## Fonte de Verdade por Domínio

| Domínio | Fonte de Verdade | Nunca corrigida por |
|---------|-----------------|---------------------|
| Produto / Estratégia | Docs P01-P16 | A01 corrige os docs; Briefing (P00) é a origem |
| Regras de Negócio | D01 (01.1-01.5) | Qualquer auditoria — D01 é sagrado |
| Stack tecnológico | D02 | C02 corrige os módulos divergentes |
| Motion / Animação | D04 | C03 corrige os módulos divergentes |
| PRD / Requisitos | D05 | A02 corrige D05 se necessário |
| Mapa de Telas | D06 | A02/B04 corrigem D06 |
| Glossário | D10 | Usado como referência em todas as auditorias |

---

## Convenções globais do sistema

| Convenção | Valor |
|-----------|-------|
| Severidades | P0 Crítico, P1 Alto, P2 Médio, P3 Baixo |
| GATE de aprovação | Zero P0–P3 após correções |
| Correções autônomas | Marcadas com `[CORRIGIDO: FINDING-XXX]` |
| Decisões autônomas | Marcadas com `[DECISÃO APLICADA: DEC-XXX — justificativa]` |
| Gestão de contexto | `/compact` preservando scorecard + findings abertos |
| Output de relatórios | `docs/05 - Auditorias/` |

---

## Changelog

| Versão | Data | Alterações |
|--------|------|-----------|
| 1.1 | 23/03/2026 | Adicionados B11 (Testes) e B12 (CI/CD) à Série B. GATE global atualizado para zero P0–P3. |
| 1.0 | 23/03/2026 | Criação. Mapa centralizado de todos os prompts de auditoria, relação entre séries A/B/C, fluxo de decisão, convenções globais. |
