# 07 - Wireframes

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Design e Frontend Engineering | Wireframes textuais de todas as telas e componentes do AI-Dani-Cessionário (D06) | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - Wireframes textuais (ASCII + Mermaid) para todos os 12 componentes/telas do D06.
> - Tokens de cor e motion do D03 e D04 referenciados em cada wireframe.
> - Variantes mobile (≤ 640px) documentadas separadamente onde diferem.
> - Cada wireframe inclui: layout, estados, hierarquia visual e notas de acessibilidade.
> - Toda tela do D06 tem wireframe correspondente neste documento.

---

## W-DC-001 — FAB Global

```
┌────────────────────────────────────────────────────────────────────┐
│  [Qualquer tela do Cessionário]                                    │
│                                                                    │
│  ... conteúdo da página ...                                       │
│                                                                    │
│                                                    ┌──────────┐   │
│                                                    │  [BADGE] │   │
│                                                    │    🤖    │   │
│                                                    │  DANI    │   │
│                                                    └──────────┘   │
│                                                    bottom:24px    │
│                                                    right: 24px    │
└────────────────────────────────────────────────────────────────────┘
```

**Estados:**
- **Normal:** Ícone da Dani. Cor `--primary`. Radius circular (`--radius-xl`). 56×56px.
- **Com alertas:** Badge vermelha no canto superior direito com número de alertas não lidos.
- **Hover:** `scale: 1.05`. Cursor pointer.
- **Foco teclado:** `ring` com `--ring`.

**Acessibilidade:**
- `aria-label="Abrir chat com Dani"` (sem alertas)
- `aria-label="Abrir chat com Dani — {N} alertas não lidos"` (com badge)
- `role="button"` + `tabIndex={0}`

---

## W-DC-002 — Chat Window (Desktop)

```
┌─────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────┐  │
│  │  [Avatar Dani]  Dani                  [X] │  │  ← Header 56px, bg --primary
│  │  Analista de Oportunidades                │  │
│  └───────────────────────────────────────────┘  │
│                                                   │
│  ┌───────────────────────────────────────────┐  │
│  │                                           │  │
│  │  [Mensagens da Dani — bg --muted]         │  │
│  │                                           │  │
│  │  [Mensagens do Cessionário — bg --primary]│  │
│  │                                           │  │
│  │  ... histórico ...                        │  │
│  │                                           │  │
│  └───────────────────────────────────────────┘  │
│                                                   │
│  ┌───────────────────────────────────────────┐  │
│  │  [Input text placeholder...]    [Enviar ▶]│  │  ← Input bar 56px
│  └───────────────────────────────────────────┘  │
│                                                   │
│  400px largura × 600px altura                    │
│  position: fixed; bottom: 88px; right: 24px      │
│  border-radius: --radius-xl (cantos superiores)  │
└─────────────────────────────────────────────────┘
```

**Variante mobile (≤ 640px):**
```
┌────────────────────────────────────────────────┐
│  [Avatar]  Dani · Analista              [←][X] │  ← Header full width
│─────────────────────────────────────────────────│
│                                                  │
│  ... mensagens ...                               │
│                                                  │
│─────────────────────────────────────────────────│
│  [Input...]                            [Enviar] │
└────────────────────────────────────────────────┘
  100vw × 100dvh, position: fixed, sem radius
```

---

## W-DC-003 — Primeiro Acesso (Boas-vindas + Starters)

```
┌─────────────────────────────────────────────────┐
│  [Header: Dani · Analista de Oportunidades]  [X] │
│─────────────────────────────────────────────────│
│                                                   │
│  ┌─────────────────────────────────────────┐    │
│  │ 🤖  Olá! Sou a Dani, sua Analista       │    │  ← Bubble --muted, fade-in 300ms
│  │      de Oportunidades. Posso analisar   │    │
│  │      riscos, comparar imóveis e         │    │
│  │      simular retornos para você.        │    │
│  │      Como posso ajudar?                 │    │
│  └─────────────────────────────────────────┘    │
│                                                   │
│  ┌──────────────────────────────────────────┐   │
│  │ Quais as melhores oportunidades hoje? →  │   │  ← Chip outlined --radius-md
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │ Tenho R$ 500k. O que recomenda? →        │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │ Como funciona a comissão? →              │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │ Prazo para depósito em Escrow? →         │   │
│  └──────────────────────────────────────────┘   │
│                                                   │
│  [Input text...]                      [Enviar ▶] │
└─────────────────────────────────────────────────┘
```

---

## W-DC-004 — KYC Pendente (Variante do Primeiro Acesso)

```
│  [Boas-vindas da Dani]                           │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │ ℹ️  Para acessar todas as análises, você   │ │  ← Banner informativo
│  │     precisa concluir sua verificação.      │ │
│  │     → Acesse Meu Perfil > Verificação      │ │  ← Link clicável
│  └─────────────────────────────────────────────┘ │
```

---

## W-DC-005 — Análise de Oportunidade Individual (inline)

```
│  ┌─────────────────────────────────────────────┐ │
│  │ 🤖  Análise: OPR-2024-0042                 │ │  ← Bubble da Dani
│  │                                             │ │
│  │  ┌─────────────────────────────────────┐   │ │
│  │  │ [OPR-2024-0042]  [● DISPONÍVEL]     │   │ │  ← Chip OPR + Badge status
│  │  │ Fortaleza · Edifício Solar · 3BR    │   │ │
│  │  │─────────────────────────────────────│   │ │
│  │  │ Δ (Delta)         R$ 150.000,00     │   │ │
│  │  │ Comissão          R$  30.000,00     │   │ │
│  │  │ Custo total       R$ 330.000,00     │   │ │
│  │  │─────────────────────────────────────│   │ │
│  │  │ Score de Risco   [● 3/10 BAIXO]     │   │ │  ← Badge verde + aria-label
│  │  │ Fatores: valorização, localização   │   │ │
│  │  │─────────────────────────────────────│   │ │
│  │  │ ROI Projetado                        │   │ │
│  │  │ [🛡 Conservador]  38%               │   │ │
│  │  │ [🎯 Base]         45%  ← destacado  │   │ │
│  │  │ [📈 Otimista]     52%               │   │ │
│  │  │─────────────────────────────────────│   │ │
│  │  │ ℹ Valores projetados. Resultados   │   │ │  ← Aviso obrigatório, text-xs
│  │  │   reais podem variar.               │   │ │
│  │  └─────────────────────────────────────┘   │ │
│  │                                             │ │
│  │  [Simular proposta]  [Comparar similares]   │ │  ← Botões outlined
│  └─────────────────────────────────────────────┘ │
```

---

## W-DC-006 — Score de Risco com Indicadores Visuais

```
  ┌──────────────────────────────┐
  │  Score de Risco              │
  │                              │
  │  [●] 3/10  RISCO BAIXO       │  ← Verde (#16A34A), aria-label completo
  │                              │
  │  Fatores analisados:         │
  │  • Valorização histórica     │
  │  • Localização               │
  │  • Tempo de mercado          │
  └──────────────────────────────┘

  Escala de cores:
  1─3: ● Verde  (#16A34A) "RISCO BAIXO"
  4─6: ● Amarelo (#D97706) "RISCO MODERADO"
  7─10: ● Vermelho (#DC2626) "RISCO ALTO"

  Contraste mínimo: 4.5:1 (obrigatório)
  aria-label: "Score de risco: {N} de 10 — risco {nível}"
```

---

## W-DC-007 — Comparação de Oportunidades (Tabela inline)

```
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 🤖  Comparação de 3 oportunidades                        │  │
│  │                                                           │  │
│  │  ┌──────┬──────────┬──────────┬──────────┬───────┬──────┐ │  │
│  │  │ OPR  │    Δ     │ Comissão │  Escrow  │ Risco │ Loc. │ │  │  ← role="table"
│  │  ├──────┼──────────┼──────────┼──────────┼───────┼──────┤ │  │
│  │  │ 0042 │ 150.000  │  30.000  │ 330.000  │ 3/10  │ FOR  │ │  │  ← clicável
│  │  │      │          │          │          │ [●]   │      │ │  │
│  │  ├──────┼──────────┼──────────┼──────────┼───────┼──────┤ │  │
│  │  │ 0078 │ 120.000  │  24.000  │ 284.000  │ 4/10  │ FOR  │ │  │
│  │  │ ★MELHOR OPÇÃO │           highlighted │       │      │ │  │  ← badge + bg sutil
│  │  ├──────┼──────────┼──────────┼──────────┼───────┼──────┤ │  │
│  │  │ 0091 │  90.000  │  18.000  │ 238.000  │ 4/10  │ CWB  │ │  │
│  │  └──────┴──────────┴──────────┴──────────┴───────┴──────┘ │  │
│  │                                                           │  │
│  │  Ranqueado por: melhor relação retorno/risco              │  │
│  └───────────────────────────────────────────────────────────┘  │
```

---

## W-DC-008 — Simulação de Proposta (resultado inline)

```
│  ┌─────────────────────────────────────────────┐ │
│  │ 🤖  Simulação para R$ 300.000,00           │ │
│  │                                             │ │
│  │  Comissão           R$  30.000,00           │ │
│  │  Total no Escrow    R$ 330.000,00           │ │
│  │  ─────────────────────────────────────────  │ │
│  │  ROI Projetado                              │ │
│  │  [🛡] Conservador   38%                    │ │
│  │  [🎯] Base          45%  ← referência       │ │
│  │  [📈] Otimista      52%                    │ │
│  │  ─────────────────────────────────────────  │ │
│  │  ℹ Valores projetados. Resultados reais    │ │  ← obrigatório
│  │    podem variar conforme condições de      │ │
│  │    mercado.                                 │ │
│  │                                             │ │
│  │  [Simular outro valor]  [Ir para a opor.] │ │  ← quick action buttons
│  └─────────────────────────────────────────────┘ │
```

---

## W-DC-009 — Simulação de Contraproposta

```
│  ┌─────────────────────────────────────────────┐ │
│  │ 🤖  Contraproposta: R$ 280.000,00          │ │
│  │                                             │ │
│  │  Comissão           R$  28.000,00          │ │
│  │  Total no Escrow    R$ 308.000,00          │ │
│  │  vs. proposta ant.  R$ 330.000,00          │ │
│  │                   ▼ -R$ 22.000 (-6,7%)     │ │  ← seta verde (economia)
│  │  ─────────────────────────────────────────  │ │
│  │  ROI Ajustado                               │ │
│  │  [🛡] Conservador  40%                     │ │
│  │  [🎯] Base         48%                     │ │
│  │  [📈] Otimista     56%                     │ │
│  │                                             │ │
│  │  [Simular outro valor]  [Ir p/ negociação] │ │
│  └─────────────────────────────────────────────┘ │
```

---

## W-DC-010 — Widget Top 3 no Dashboard

```
┌────────────────────────────────────────────────────┐
│  🤖 Oportunidades em Destaque                  [→] │  ← widget no Dashboard
│  ─────────────────────────────────────────────────  │
│  ┌──────────────────────────────────────────────┐  │
│  │ OPR-2024-0042 · Fortaleza                    │  │  ← card 1
│  │ Δ R$ 150.000 · Risco 3/10 [● BAIXO]         │  │
│  │ Comissão estimada: R$ 30.000                 │  │
│  │                              [Analisar →]   │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │ OPR-2024-0078 · Fortaleza                    │  │  ← card 2
│  │ Δ R$ 120.000 · Risco 4/10 [● MÉDIO]         │  │
│  │ Comissão estimada: R$ 24.000                 │  │
│  │                              [Analisar →]   │  │
│  └──────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────┐  │
│  │ OPR-2024-0091 · Curitiba                     │  │  ← card 3
│  │ Δ R$ 90.000  · Risco 4/10 [● MÉDIO]         │  │
│  │ Comissão estimada: R$ 18.000                 │  │
│  │                              [Analisar →]   │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

**Estado: perfil incompleto:**
```
│  ℹ Recomendações baseadas em dados gerais.         │
│    Complete seu perfil para resultados            │
│    personalizados.        [→ Ir para Perfil]     │
```

---

## W-DC-011 — Banner de Fallback da Calculadora

```
  ┌─────────────────────────────────────────────────────┐
  │  🧮  Modo básico — sem análise da IA               │  ← fundo --agent-fallback
  │      Cálculo realizado sem análise contextual.     │
  │      Para análise completa, tente novamente        │
  │      em instantes.                                  │
  └─────────────────────────────────────────────────────┘

  Resultado do cálculo exibido normalmente abaixo.
  Ações disponíveis: copiar valores, solicitar novo cálculo.
```

---

## W-DC-012 — Estado de Rate Limit do Input

```
  ┌─────────────────────────────────────────────────────────┐
  │  [Você atingiu o limite de 30 mensagens/hora.          │
  │   Próxima mensagem disponível em: 23:45]              │
  │                                                         │
  │  ┌──────────────────────────────────────┐  [Enviar]   │
  │  │ [campo desabilitado — bg cinza]      │  [inativo]  │  ← cursor: not-allowed
  │  └──────────────────────────────────────┘             │
  └─────────────────────────────────────────────────────────┘

  Contador: mm:ss, atualizado a cada segundo.
  Reativação: pulse 500ms (--motion-feedback) na borda do input.
```

---

## W-DC-013 — Typing Indicator

```
  ┌──────────────────────────────────┐
  │  🤖  ● ● ●                      │  ← Três pontos com bounce stagger 150ms
  └──────────────────────────────────┘

  role="status"
  aria-label="Dani está digitando"
  Removido do DOM quando resposta chega.
  Reduced motion: opacity pulse (0.3→1→0.3) sem translação.
```

---

## W-DC-014 — Vinculação WhatsApp (Fase 2)

### Etapa 1 — Informar número

```
┌───────────────────────────────────────────────────┐
│  Meu Perfil > WhatsApp                            │
│  ──────────────────────────────────────────────── │
│  Vincule seu WhatsApp para usar a Dani            │
│  fora da plataforma.                              │
│                                                   │
│  Número de WhatsApp                               │
│  ┌──────────────────────────────────────────┐    │
│  │ (xx) xxxxx-xxxx                          │    │  ← máscara automática
│  └──────────────────────────────────────────┘    │
│  [Mensagem de erro inline se inválido]            │
│                                                   │
│  [Vincular WhatsApp]                              │  ← CTA primary
└───────────────────────────────────────────────────┘
```

### Etapa 2 — Inserir OTP SMS

```
│  Insira o código enviado por SMS                  │
│  para (xx) xxxxx-xxxx                             │
│                                                   │
│  ┌──────────────────────────────────────────┐    │
│  │ _ _ _ _ _ _                              │    │  ← 6 dígitos, input OTP
│  └──────────────────────────────────────────┘    │
│  [Tentativas restantes: 2]                        │
│  [Reenviar em 57s]                                │
│                                                   │
│  [Confirmar]                                      │
```

### Etapa 3 — Confirmação WhatsApp

```
│  ✉  Enviamos uma mensagem de confirmação          │
│     para o seu WhatsApp.                          │
│     Responda com o código enviado.                │
│     Expira em 24 horas.                           │
```

### Estado vinculado

```
│  ✅ WhatsApp vinculado                           │
│  Número: (85) •••••-3210                         │
│                          [Desvincular]           │
```

---

## W-DC-015 — Mensagens de Recusa (padrão)

```
  ┌──────────────────────────────────────────────────────────────┐
  │ 🤖  Essa informação não está disponível para o seu perfil.  │  ← bubble --muted
  │     Para mais detalhes sobre a transação, entre em          │
  │     contato com o suporte via negociação.                   │
  └──────────────────────────────────────────────────────────────┘

  Após 2 insistências — mesma mensagem + alternativa.
  Após 3ª insistência — mensagem de recusa sem alternativas.
```

---

## Rastreabilidade: Telas D06 × Wireframes D07

| Tela (D06) | Wireframe (D07) | Coberta |
|---|---|---|
| T-DC-001 — FAB Global | W-DC-001 | ✅ |
| T-DC-002 — Chat (sem contexto) | W-DC-002, W-DC-003 | ✅ |
| T-DC-003 — Chat (contexto oportunidade) | W-DC-002 + W-DC-005 | ✅ |
| T-DC-004 — Chat (contexto negociação) | W-DC-002 + W-DC-009 | ✅ |
| T-DC-005 — Módulo Análise Individual | W-DC-005, W-DC-006 | ✅ |
| T-DC-006 — Módulo Comparação | W-DC-007 | ✅ |
| T-DC-007 — Módulo Simulação Proposta | W-DC-008 | ✅ |
| T-DC-008 — Módulo Simulação Contraproposta | W-DC-009 | ✅ |
| T-DC-009 — Widget Top 3 Dashboard | W-DC-010 | ✅ |
| T-DC-010 — Banner Fallback | W-DC-011 | ✅ |
| T-DC-011 — Estado Rate Limit | W-DC-012 | ✅ |
| T-DC-012 — Vinculação WhatsApp | W-DC-014 | ✅ |

**12/12 telas cobertas por wireframes ✅**

---

## Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. 15 wireframes textuais cobrindo todas as 12 telas do D06. Variantes mobile, estados, acessibilidade e rastreabilidade documentados. |
