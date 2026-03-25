# 07 - Wireframes

## Módulo Cessionário · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Design, Frontend e Produto |
| **Escopo** | Wireframes em texto/ASCII das telas críticas do módulo Cessionário |
| **Módulo** | Cessionário |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 22/03/2026 00:00 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - Este documento descreve a estrutura e layout das telas críticas do módulo Cessionário em formato de wireframe descritivo.
> - Cobertura: 12 telas críticas (de 34 no mapa) com descrição detalhada de layout, hierarquia e componentes.
> - Padrão visual: shadcn/ui + Tailwind CSS + Brand Guide (Inter Variable, radius 14px, azul #0069A8).
> - Layout base: Sidebar fixa (240px) + Main content (flex-1) + Header (64px) — SPA 100% logada.
> - Telas de autenticação: layout centrado sem sidebar.

---

## 1. Layout Base — Telas Autenticadas

```
┌─────────────────────────────────────────────────────────────────┐
│ HEADER (64px altura)                                            │
│ [Logo Repasse Seguro]              [🔔 Badge]  [Avatar + Nome ▼]│
├──────────────┬──────────────────────────────────────────────────┤
│ SIDEBAR      │ MAIN CONTENT AREA                                │
│ (240px)      │ (flex-1, padding 24px)                          │
│              │                                                  │
│ ● Dashboard  │                                                  │
│ ○ Oportunid. │  <Conteúdo específico de cada tela>              │
│ ○ Propostas  │                                                  │
│ ○ Negociações│                                                  │
│ ○ Assinaturas│                                                  │
│ ○ Financeiro │                                                  │
│ ○ Assistente │                                                  │
│ ○ Meu Perfil │                                                  │
│              │                                                  │
│ (bottom)     │                                                  │
│ [Sair]       │                                                  │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 2. T-AUTH-02 — Tela de Cadastro

```
┌─────────────────────────────────────────────────────────────────┐
│                   [Logo Repasse Seguro]                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Criar sua conta                            │   │
│  │                                                         │   │
│  │  Nome completo *                                        │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │                                                 │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  │  E-mail *                                               │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │                                                 │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  │  Senha *                                                │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ ••••••••                               [👁]     │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │  [██░░░] Força da senha: Média                          │   │
│  │                                                         │   │
│  │  ☐ Aceito os Termos de Uso e Política de Privacidade *  │   │
│  │  ☐ Aceito o uso dos dados pelo Analista de IA *         │   │
│  │                                                         │   │
│  │  [████████ Criar conta ████████]  ← btn primary         │   │
│  │                                                         │   │
│  │  ─────────── ou ───────────                             │   │
│  │  [G  Continuar com Google]  ← btn outline               │   │
│  │                                                         │   │
│  │  Já tem conta? Fazer login                              │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Comportamentos:**
- Indicador de força de senha: atualiza em tempo real (fraca=vermelho, média=amarelo, forte=verde)
- Botão "Criar conta" desabilitado até ambos os checkboxes obrigatórios marcados
- Validação on blur em todos os campos
- Links dos termos abrem em nova aba

---

## 3. T-DASH-01 — Dashboard

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  Dashboard                                       │
│              │                                                  │
│ ● Dashboard  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│              │  │Propostas │ │Negociações│ │  Valor em Escrow │ │
│              │  │  Ativas  │ │  Ativas  │ │                  │ │
│              │  │   [3]    │ │   [2]    │ │  R$ 530.000,00   │ │
│              │  │ badge azul│ │  1 ⚠️ 3d │ │                  │ │
│              │  └──────────┘ └──────────┘ └──────────────────┘ │
│              │                                                  │
│              │  ┌───────────────────────────────────────────┐   │
│              │  │  Alertas e Notificações                   │   │
│              │  │  • Proposta aceita — OPR-2026-0042   2min │   │
│              │  │  • Dep. Escrow confirmado — NEG-001  1h  │   │
│              │  │  • KYC aprovado                     3h   │   │
│              │  │  [Ver todas as notificações →]            │   │
│              │  └───────────────────────────────────────────┘   │
│              │                                                  │
│              │  Oportunidades em Destaque                       │
│              │  ┌──────────────┐ ┌──────────────┐ ┌──────────┐ │
│              │  │ OPR-2026-042 │ │ OPR-2026-051 │ │OPR-...  │ │
│              │  │ Fortaleza-CE │ │ São Paulo-SP  │ │         │ │
│              │  │ 3 qts · 75m² │ │ 2 qts · 62m² │ │         │ │
│              │  │ Δ R$ 80k     │ │ Δ R$ 120k    │ │         │ │
│              │  │ Score: 8/10  │ │ Score: 7/10  │ │         │ │
│              │  └──────────────┘ └──────────────┘ └──────────┘ │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 4. T-OPR-01 — Lista de Oportunidades

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  Oportunidades                                   │
│              │                                                  │
│ ○ Oportunid.●│  [Filtros: Estado ▼] [Preço ▼] [Tipologia ▼]   │
│              │  [Score risco ▼] [Delta mínimo ▼]               │
│              │  Ordenar: [Recomendados pela IA ▼]              │
│              │                                                  │
│              │  Resultados: 47 oportunidades                   │
│              │                                                  │
│              │  ┌────────────────────────────────────────────┐ │
│              │  │ OPR-2026-0042          Score: 8/10  🟢     │ │
│              │  │ Fortaleza-CE · Aldeota · Empreend. X       │ │
│              │  │ 3 quartos · 85m² · 1 vaga                  │ │
│              │  │ Tabela Atual: R$ 680.000                    │ │
│              │  │ Tabela Contrato: R$ 580.000                 │ │
│              │  │ Δ: R$ 100.000                               │ │
│              │  │ Pago: 35% da Tabela Contrato                │ │
│              │  │ Publicado: 20/03/2026                       │ │
│              │  │                    [Ver detalhes →]         │ │
│              │  └────────────────────────────────────────────┘ │
│              │                                                  │
│              │  ┌────────────────────────────────────────────┐ │
│              │  │ [Skeleton card animado em carregamento]    │ │
│              │  └────────────────────────────────────────────┘ │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 5. T-OPR-02 — Detalhe de Oportunidade

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  ← Oportunidades / OPR-2026-0042                │
│              │                                                  │
│              │  OPR-2026-0042 · Score de Risco: 8/10           │
│              │  Fortaleza-CE · Aldeota · Empreendimento X      │
│              │                                                  │
│              │  ┌───────────────────┬────────────────────────┐ │
│              │  │ DADOS DA OPERAÇÃO │ SIMULAÇÃO DE CUSTOS     │ │
│              │  │                   │                        │ │
│              │  │ Tipologia: 3Q     │ Preço Repasse:         │ │
│              │  │ Metragem: 85m²    │ (a definir na proposta)│ │
│              │  │ Garagem: 1 vaga   │                        │ │
│              │  │                   │ Comissão Comprador:    │ │
│              │  │ Tabela Atual:     │ 20% × Δ = R$ 20.000   │ │
│              │  │ R$ 680.000        │ (estimativa com Δ=100k)│ │
│              │  │                   │                        │ │
│              │  │ Tabela Contrato:  │ Total estimado:        │ │
│              │  │ R$ 580.000        │ Proposta + R$ 20.000   │ │
│              │  │                   │                        │ │
│              │  │ Δ: R$ 100.000     │                        │ │
│              │  │ Pago: 35%         │                        │ │
│              │  └───────────────────┴────────────────────────┘ │
│              │                                                  │
│              │  [Gráfico de valorização do empreendimento]      │
│              │                                                  │
│              │  Comparativo com similares (gerado pela IA):    │
│              │  [Tabela comparativa: score, delta, localização] │
│              │                                                  │
│              │  ┌──────────────────────────────────────────┐   │
│              │  │ [████ Fazer Proposta ████]               │   │
│              │  │ [Consultar Analista]    ← outline        │   │
│              │  └──────────────────────────────────────────┘   │
│              │  (Botão Fazer Proposta: desabilitado se sem KYC) │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 6. T-OPR-03 — Formulário de Proposta

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  ← OPR-2026-0042 / Fazer Proposta               │
│              │                                                  │
│              │  Nova Proposta para OPR-2026-0042                │
│              │                                                  │
│              │  Oportunidade                                    │
│              │  ┌─────────────────────────────────────────┐    │
│              │  │ OPR-2026-0042 · Fortaleza-CE · 3Q/85m² │    │
│              │  └─────────────────────────────────────────┘    │
│              │  (preenchida automaticamente, readonly)          │
│              │                                                  │
│              │  Valor da Proposta (R$) *                        │
│              │  ┌─────────────────────────────────────────┐    │
│              │  │ R$                                      │    │
│              │  └─────────────────────────────────────────┘    │
│              │                                                  │
│              │  Comissão Comprador estimada:                    │
│              │  20% × Δ R$ 100.000 = R$ 20.000                 │
│              │  (atualizado em tempo real conforme valor)       │
│              │                                                  │
│              │  Mensagem ao Admin (opcional, máx 500 chars)    │
│              │  ┌─────────────────────────────────────────┐    │
│              │  │                               0/500     │    │
│              │  └─────────────────────────────────────────┘    │
│              │                                                  │
│              │  ☑ Estou ciente dos termos e da comissão *      │
│              │                                                  │
│              │  [████████ Enviar Proposta ████████]             │
│              │  [Cancelar]  ← ghost btn                         │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 7. T-PRF-02 — KYC Stepper

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  Meu Perfil / Verificação de Identidade (KYC)   │
│              │                                                  │
│              │  ●━━━━━━○━━━━━━○                                │
│              │  Passo 1   Passo 2   Passo 3                    │
│              │  Documento Comprovante Selfie                   │
│              │                                                  │
│              │  ┌──────────────────────────────────────────┐   │
│              │  │  PASSO 1: Documento de Identidade         │   │
│              │  │                                           │   │
│              │  │  Envie o RG ou CNH (frente e verso)       │   │
│              │  │  Formatos: JPG, PNG, PDF · Máx: 10 MB     │   │
│              │  │                                           │   │
│              │  │  Frente:                                  │   │
│              │  │  ┌─────────────────────────────────────┐  │   │
│              │  │  │  ⬆ Arraste ou clique para enviar    │  │   │
│              │  │  └─────────────────────────────────────┘  │   │
│              │  │                                           │   │
│              │  │  Verso:                                   │   │
│              │  │  ┌─────────────────────────────────────┐  │   │
│              │  │  │  ⬆ Arraste ou clique para enviar    │  │   │
│              │  │  └─────────────────────────────────────┘  │   │
│              │  │                                           │   │
│              │  │  [████ Próximo: Comprovante ████]         │   │
│              │  └──────────────────────────────────────────┘   │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 8. T-NEG-02 — Detalhe de Negociação (+ Chat)

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  ← Negociações / NEG-001                        │
│              │                                                  │
│              │  Negociação — OPR-2026-0042                     │
│              │  Status: 🟡 Aguardando Depósito                  │
│              │  Prazo Escrow: 05/04/2026 (3 dias — ⚠️)         │
│              │                                                  │
│              │  ┌──────────────────────────────────────────┐   │
│              │  │ RESUMO DA NEGOCIAÇÃO                     │   │
│              │  │ Proposta aceita:        R$ 480.000       │   │
│              │  │ Comissão Comprador:     R$ 20.000        │   │
│              │  │ Total Escrow:           R$ 500.000       │   │
│              │  └──────────────────────────────────────────┘   │
│              │                                                  │
│              │  [████ Depositar em Escrow ████]  ← CTA          │
│              │  [Solicitar extensão de prazo]    ← outline      │
│              │                                                  │
│              │  ─────── Chat com Admin ───────                  │
│              │  ┌──────────────────────────────────────────┐   │
│              │  │ Admin (10:32): Proposta aceita! ...      │   │
│              │  │ Você (11:15): Obrigado! Vou depositar... │   │
│              │  │ Admin (14:00): Ok, aguardamos...         │   │
│              │  │                                          │   │
│              │  │ ┌──────────────────────────────────┐    │   │
│              │  │ │ Escreva uma mensagem...       📎 │    │   │
│              │  │ └──────────────────────────────────┘    │   │
│              │  │ [Enviar]                                 │   │
│              │  └──────────────────────────────────────────┘   │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 9. T-NEG-03 — Instrução de Depósito Escrow

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  ← Negociações / Depositar em Escrow             │
│              │                                                  │
│              │  Instrução de Depósito — OPR-2026-0042          │
│              │                                                  │
│              │  ┌──────────────────────────────────────────┐   │
│              │  │  VALOR A DEPOSITAR                       │   │
│              │  │                                          │   │
│              │  │  Preço Repasse:    R$ 480.000,00         │   │
│              │  │  Comissão:       + R$  20.000,00         │   │
│              │  │  ─────────────────────────────────       │   │
│              │  │  Total:          = R$ 500.000,00         │   │
│              │  └──────────────────────────────────────────┘   │
│              │                                                  │
│              │  ┌──────────────────────────────────────────┐   │
│              │  │  DADOS DA CONTA ESCROW                   │   │
│              │  │  Banco: 341 - Itaú                       │   │
│              │  │  Agência: 1234                           │   │
│              │  │  Conta: 56789-0                          │   │
│              │  │  Chave PIX: repasse@repasse.com.br       │   │
│              │  │                                          │   │
│              │  │  [📋 Copiar dados bancários]             │   │
│              │  └──────────────────────────────────────────┘   │
│              │                                                  │
│              │  Prazo: até 05/04/2026 (3 dias restantes ⚠️)    │
│              │                                                  │
│              │  [████ Enviar Comprovante ████]                  │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 10. T-ASS-03 — Transição para ZapSign

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  ← Assinaturas / Assinar Documento               │
│              │                                                  │
│              │  ┌──────────────────────────────────────────┐   │
│              │  │                                          │   │
│              │  │  🔒 Assinatura Digital Segura             │   │
│              │  │                                          │   │
│              │  │  Você será redirecionado para o          │   │
│              │  │  ambiente seguro de assinatura digital   │   │
│              │  │  (ZapSign).                              │   │
│              │  │                                          │   │
│              │  │  Ao retornar, sua assinatura será        │   │
│              │  │  registrada automaticamente.             │   │
│              │  │                                          │   │
│              │  │  [████ Continuar para assinatura ████]   │   │
│              │  │  [← Voltar]                              │   │
│              │  │                                          │   │
│              │  │  Powered by ZapSign                      │   │
│              │  └──────────────────────────────────────────┘   │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 11. T-IA-01 — Assistente IA

```
┌──────────────┬──────────────────────────────────────────────────┐
│ SIDEBAR      │  Analista de Oportunidades                      │
│              │                                                  │
│              │  ┌──────────────────────────────────────────┐   │
│              │  │                                          │   │
│              │  │  Histórico de conversa:                  │   │
│              │  │                                          │   │
│              │  │  ─── Hoje ───                            │   │
│              │  │  Você: Analise o risco da OPR-2026-0042  │   │
│              │  │                                          │   │
│              │  │  Analista:                               │   │
│              │  │  Score de risco: 8/10                    │   │
│              │  │  Fatores positivos:                      │   │
│              │  │  • Empreendimento em Aldeota (valorizando)│   │
│              │  │  • 35% pago pelo Cedente (reduz risco)   │   │
│              │  │  • Δ R$ 100k favorável                   │   │
│              │  │  Fatores de atenção:                     │   │
│              │  │  • Prazo de entrega: 18 meses            │   │
│              │  │  [Comparar com similares ▼]              │   │
│              │  │                                          │   │
│              │  │ ┌──────────────────────────────────────┐ │   │
│              │  │ │ Pergunte ao Analista...          [↗] │ │   │
│              │  │ └──────────────────────────────────────┘ │   │
│              │  └──────────────────────────────────────────┘   │
└──────────────┴──────────────────────────────────────────────────┘
```

---

## 12. Modal de Re-autenticação (Overlay Global)

```
┌─────────────────────────────────────────────────────────────────┐
│ TELA ATUAL (opacidade reduzida)                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  🔒 Confirme sua identidade                              │  │
│  │                                                          │  │
│  │  Por segurança, confirme sua senha para continuar.       │  │
│  │                                                          │  │
│  │  Senha *                                                 │  │
│  │  ┌──────────────────────────────────────────────────┐   │  │
│  │  │ ••••••••                               [👁]      │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  │                                                          │  │
│  │  [████ Confirmar ████]  [Cancelar]                       │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13. Anotações de Design

| **Elemento** | **Especificação** |
|---|---|
| Sidebar | 240px fixo, fundo `--secondary` (#F4F4F5), itens com hover state |
| Header | 64px, fundo `--background` (#FFFFFF), border-bottom 1px `--border` |
| Cards de oportunidade | `--radius-lg` (14px), shadow leve, hover: shadow mais pronunciada |
| Botão primário | Fundo `--primary` (#0069A8), texto `--primary-foreground` (#F0F9FF), `--radius-md` (11.2px) |
| Botão outline | Border `--primary`, texto `--primary` |
| Badges de status | `--radius-sm` (8.4px), cores semânticas |
| Skeleton screens | Via Framer Motion, cor `--muted` (#F4F4F5), animação shimmer |
| Toasts | Canto inferior direito, z-index alto, auto-dismiss 5s |
| Modais | Overlay `background/60` backdrop-blur, `--radius-xl` (19.6px) |

---

## 14. Changelog

| **Data** | **Versão** | **Descrição** |
|---|---|---|
| 22/03/2026 | v1.0 | Criação inicial — Pipeline ShiftLabs v9.5 |
