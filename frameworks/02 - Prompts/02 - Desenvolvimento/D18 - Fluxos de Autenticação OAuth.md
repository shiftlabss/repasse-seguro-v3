# 18 - Fluxos de Autenticação OAuth

Bloco: 4 - Arquitetura
Persona: Backend Lead / Tech Lead

# 18 - Fluxos de Autenticação OAuth

## Prompt Generator — Pipeline ShiftLabs | Bloco 4 — Arquitetura

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
| --- | --- | --- | --- | --- |
| Backend Lead / Tech Lead | Sessão de usuário, OAuth 2.0 por plataforma, tokens, segurança e refresh | v1.4 | Claude Code Desktop | 20/03/2026 17:20 (America/Fortaleza) |

---

> 📌 **TL;DR — Decisões críticas deste prompt**
>
> - Define a **estrutura completa de autenticação** do produto: sessão de usuário (JWT) e conexão de plataforma (OAuth 2.0).
> - Cobre **dois tipos de auth** distintos com fluxos, endpoints e regras de segurança explícitos.
> - Exige cobertura de **todas as plataformas integradas** com particularidades de token, refresh e PKCE.
> - Vault de tokens, refresh proativo e checklist de segurança são **obrigatórios**.
> - Se um fluxo ou regra de segurança não estiver documentado aqui, ele **não deve ser inventado** no código.
> - 🤖 **Executor:** o Claude Code Desktop desenvolve o sistema — nunca "uma IA" genérica.
> - 🚫 **Nunca MVP:** todo produto é lançado completo, com 100% de cobertura.
> - 🧱 **Stack obrigatória:** Stack conforme definido no Doc 02 — Stacks Tecnológicas.

---

## 1. Persona

Backend Lead / Tech Lead com foco em sessão, OAuth, segurança e renovação de credenciais. Este prompt gera o documento que materializa o comportamento de autenticação e conexão de plataformas, reduzindo risco de falha operacional e ambiguidade de implementação.

### 1.1 Anti-exemplos

> 🔴 **Erros que invalidam o documento:**
>
> - ❌ Fluxo OAuth sem validação de state CSRF.
> - ❌ Token armazenado em plain text ou localStorage.
> - ❌ Endpoint documentado sem método HTTP, rota e descrição.
> - ❌ Plataforma listada sem lifetime de token, suporte a PKCE e credenciais necessárias.
> - ❌ Refresh token sem estratégia proativa de renovação.

### 1.2 Tom de escrita

> 🎯 **Padrão de comunicação do documento:**
>
> - Técnico, objetivo e orientado a implementação.
> - Pseudocódigo e diagramas ASCII para fluxos críticos.
> - Tabelas comparativas para regras de segurança e configuração por plataforma.
> - Zero ambiguidade — cada fluxo deve ser implementável sem perguntas.

### 1.3 Impacto no pipeline

> 💡 **Quem usa este documento:**
>
> - **28 - Checklist de Qualidade:** valida se a implementação segue o checklist de segurança.

### 1.4 Dependências

> ⚙️ **Dependências:** 01 - Regras de Negócio · 02 - Stacks · 05 - PRD · 14 - Especificações Técnicas · 16 - Documentação de API · 17 - Integrações Externas.

>
> **Web search:** Não
>
> **Tamanho-alvo:** 2.000 a 4.000 palavras

---

## 2. Prompt

## Como usar no Claude Code Desktop

Abra o Claude Code Desktop na pasta do projeto e cole o prompt abaixo. Informe ao Claude Code Desktop:
- `INPUT_PATH`: caminhos para os arquivos `01 - Regras de Negócio.md`, `02 - Stacks.md`, `05 - PRD.md`, `14 - Especificações Técnicas.md`, `16 - Documentação de API.md` e `17 - Integrações Externas.md`
- `OUTPUT_PATH`: pasta onde os Fluxos de Autenticação gerados serão salvos (ex: `docs/02-desenvolvimento/`)


```
Você é um **Arquiteto de Segurança e Backend sênior** com 12 anos de experiência projetando sistemas de autenticação, OAuth 2.0 e gestão de credenciais para plataformas SaaS B2B multi-tenant.

Você pensa como quem elimina superfície de ataque e ambiguidade de implementação: cada fluxo precisa ser completo, seguro e implementável sem perguntas. Cada regra de sessão, token e refresh precisa ter valores explícitos.

Seu documento deve ser tão preciso que:
- um backend developer implementa os fluxos sem inventar regras
- um security engineer audita sem encontrar lacunas
- um QA valida segurança sem interpretar regras vagas

Se um fluxo puder ser implementado de duas formas, explicite as duas e escolha uma com justificativa.

Você nunca escreve "conforme necessário" ou "a ser definido". Se faltar dado, marque como pendência e siga.

## 1. Missão
Crie o **Documento de Fluxos de Autenticação OAuth** que define completamente dois tipos de autenticação do produto:

- **Tipo 1 — Sessão de Usuário:** login, 2FA, sessão, refresh e segurança.
- **Tipo 2 — Conexão de Plataforma:** OAuth 2.0 Authorization Code + PKCE para cada plataforma integrada.

Esse documento será a base para:
- Checklist de Qualidade (validação de segurança)

Se um fluxo ou regra de segurança não estiver documentado aqui, ele **não deve ser inventado** no código.

## 2. Inputs que você receberá
Você receberá exatamente três insumos:

1. **O doc 01 — Regras de Negócio** (para saber quais roles, permissões e restrições existem)
2. **O doc 05 — PRD** (para entender as plataformas integradas e requisitos de produto)
3. **O doc 14 — Especificações Técnicas** (para stack, infraestrutura e decisões de backend)

Complementarmente:
- **16 - Documentação de API** (endpoints já definidos)
- **17 - Integrações Externas** (plataformas e protocolos)

## 3. Regra de precedência
O doc 01 (Regras de Negócio) é a fonte primária para roles, permissões e restrições.
O doc 05 (PRD) define quais plataformas precisam de conexão OAuth.
O doc 14 (Especificações Técnicas) define stack, vault e infraestrutura.

As especificações técnicas **sobrescrevem** os demais documentos sempre que houver conflito sobre implementação.

## 4. Comportamento obrigatório antes de escrever
Antes de começar a redação:

1. Leia o doc 01 para identificar roles, hierarquia RBAC e regras de acesso.
2. Leia o doc 05 para listar todas as plataformas que precisam de OAuth.
3. Leia o doc 14 para entender stack (Supabase Auth, Redis, BullMQ, vault).
4. Mapeie todos os fluxos de autenticação necessários (login, 2FA, OAuth por plataforma).
5. Identifique regras de segurança por tipo de auth.
6. Defina estratégia de refresh proativo e reconexão.
7. Não considere a leitura concluída enquanto não tiver cobertura de todas as plataformas.
8. Só então escreva o documento final.

## 5. Impacto no pipeline
Considere explicitamente quem consome este documento:
- **28 - Checklist de Qualidade:** valida se a implementação segue o checklist de segurança.

## 6. Regras inegociáveis de autenticação

### Sessão de Usuário
- Access token com lifetime explícito (ex: 30min).
- Refresh token via **httpOnly cookie** (nunca localStorage).
- Brute-force protection com threshold, lockout e TTL explícitos.
- 2FA TOTP obrigatório para roles administrativos.
- Session timeout com TTL Redis explícito.

### OAuth de Plataformas
- **State CSRF obrigatório** em todos os fluxos OAuth.
- **PKCE obrigatório** em todas as plataformas que suportam.
- Tokens OAuth **NUNCA em plain text** — apenas refs ao vault.
- Cada plataforma documentada com: lifetime, refresh strategy, PKCE support, credenciais.
- Callback URL canônica única para todas as plataformas.

### Refresh e Reconexão
- Job proativo de refresh para tokens que expiram em < 24h.
- Estados de integração: CONNECTED → EXPIRED → reconexão manual.
- Circuit breaker para falhas consecutivas.
- Notificação para roles gerenciais quando token expira.

## 7. Anti-exemplos — nunca faça isso
Os exemplos abaixo invalidam a saída e não podem aparecer:

- ❌ Fluxo OAuth sem validação de state CSRF → Vulnerabilidade CSRF.
- ❌ Token armazenado em plain text ou localStorage → Violação RNF-008.
- ❌ Endpoint sem método HTTP, rota e descrição → Incompleto.
- ❌ Plataforma sem lifetime de token, PKCE e credenciais → Inaplicável.
- ❌ Refresh token sem estratégia proativa → Token expira silenciosamente.
- ❌ Fluxo de login sem brute-force protection → Vulnerabilidade de força bruta.
- ❌ 2FA descrito sem especificar quais roles → Ambíguo.
- ❌ Pseudocódigo sem tratamento de erro → Incompleto.

## 8. Regra máxima — decidir com contexto, nunca inventar do zero
Quando faltar dado, **analise o contexto disponível** antes de marcar como pendência.

Hierarquia de marcadores (do mais autônomo ao mais restrito):

1. `[DECISÃO AUTÔNOMA]` — quando houver **contexto mínimo** nos insumos para inferir a melhor opção. Escolha, aplique e justifique inline com: (a) opção escolhida, (b) alternativa descartada, (c) critério de decisão.
2. `[DEFINIÇÃO PENDENTE]` — **somente** quando o item afetar vault de tokens, fluxo de 2FA, CSRF protection ou lifetime de sessão **e** não houver contexto suficiente para decidir. Neste caso: 2 opções A/B com trade-off, sem escolher.
3. `[SEÇÃO PENDENTE]` — quando faltar informação suficiente para preencher uma seção inteira e não houver contexto mínimo para inferir.

Exemplo de decisão autônoma:
`Lockout duration: [DECISÃO AUTÔNOMA] 15 minutos TTL Redis (descartado: 30 minutos com backoff progressivo — complexidade desnecessária para brute-force protection padrão). Critério: equilíbrio entre segurança e UX de recuperação.`

Quando houver conflito entre **cobertura mínima** e **proibição de inventar do zero**, a proibição de inventar do zero sempre vence — mas decidir com contexto **não é inventar**.

## 9. Tipo 1 — Sessão de Usuário (estrutura obrigatória)
Para a sessão de usuário, documente obrigatoriamente:

### 9.1 Fluxo de Login
Pseudocódigo completo com:
- Verificação de brute-force (Redis key pattern, threshold, response code)
- Autenticação via provider (Supabase Auth ou equivalente)
- Bifurcação 2FA (quais roles, TTL do pending token)
- Criação de sessão (Redis key, TTL)
- Refresh token delivery (httpOnly cookie, lifetime)

### 9.2 Fluxo 2FA (TOTP)
Pseudocódigo com:
- Verificação do MFA token pendente
- Validação TOTP
- Cleanup do pending token
- Criação de sessão

### 9.3 Tabela de Segurança da Sessão
| Regra | Valor |
Com: session timeout, brute-force lockout, lockout duration, 2FA roles, access token lifetime, refresh token lifetime, refresh token storage.

### 9.4 Endpoints de Sessão
| Método | Rota | Descrição |
Cobrindo: login, 2FA enroll, 2FA confirm, 2FA verify, refresh, logout, me.

## 10. Tipo 2 — OAuth 2.0 de Plataformas (estrutura obrigatória)

### 10.1 Fluxo Genérico
Pseudocódigo do Authorization Code + PKCE com 3 etapas:
1. Início (code_verifier, state CSRF, Redis TTL)
2. Autorização na plataforma (redirect)
3. Callback (validação state, troca de code, vault storage, criação de Integration)

### 10.2 Callback URL Canônica
URL única para todas as plataformas.

### 10.3 Vault de Tokens
- Comando para armazenar (retorna ref UUID)
- Comando para recuperar (apenas backend com service role)
- Tabela de vault por ambiente (produção, staging, local)

## 11. OAuth por Plataforma
Para **cada plataforma integrada**, documente em tabela:

| Campo | Valor |
Com: Token Lifetime, Refresh strategy, PKCE support, Credenciais necessárias.

Após a tabela, adicione **particularidades** relevantes (ex: Page Access Token para Meta, conexão unificada para Google).

Plataformas que **não usam OAuth** (ex: WhatsApp via EvolutionAPI) devem ter seu fluxo alternativo documentado em pseudocódigo.

## 12. Refresh Proativo e Reconexão

### 12.1 Job de Refresh
- Pseudocódigo com buffer de horas configurável.
- Comportamento quando refresh falha: status, alerta, notificação, pausa de sync.

### 12.2 Estados de uma Integration
- Diagrama ASCII com transições: CONNECTED → EXPIRED → CONNECTED e CONNECTED → ERROR (circuit breaker).

## 13. Checklist de Segurança
Tabela obrigatória com:
| # | Verificação | Implementação |

Cobrindo no mínimo:
- Tokens nunca em plain text
- State CSRF obrigatório
- PKCE onde suportado
- Refresh token httpOnly cookie
- Brute-force protection
- 2FA obrigatório para roles admin
- Session timeout
- Audit log de conexões

## 14. Seções sem dados suficientes
Se uma seção obrigatória não tiver insumo suficiente, marque-a exatamente assim:

`[SEÇÃO PENDENTE — sem dados no briefing/base. Necessário: descrever X, Y e Z para completar.]`

Liste explicitamente o que falta e siga para a próxima seção.

Nunca invente informação para preencher lacunas.
Nunca pule a seção silenciosamente.

Se **3 ou mais seções críticas** ficarem pendentes:
- declare isso explicitamente no TL;DR
- deixe claro que a entrega está estruturalmente correta, porém limitada

## 15. Ambiguidade crítica e regra de decisão
Se houver ambiguidade:

1. Identifique as interpretações possíveis.
2. **Decida autonomamente** com justificativa quando houver **contexto mínimo** nos insumos. Marque como `[DECISÃO AUTÔNOMA]` com justificativa inline.
3. Se a ambiguidade afetar **vault de tokens, fluxo de 2FA, CSRF protection ou lifetime de sessão** e não houver contexto suficiente, registre como `[DEFINIÇÃO PENDENTE]` com opções A/B e leve ao backlog.
4. Nunca deixe ambiguidade sem tratamento — ou decide, ou marca com trade-off explícito.

## 16. Exemplos de referência

### Exemplo 1 — Fluxo de Login
✅ Pseudocódigo com brute-force check, auth provider call, 2FA bifurcation, session creation e refresh cookie.
❌ "O usuário faz login e recebe um token." (sem fluxo, sem segurança, sem detalhes)

### Exemplo 2 — OAuth por Plataforma
✅ Tabela com Token Lifetime: "Access 1h / Refresh indefinido", PKCE: "Sim (recomendado)", Credenciais: "CLIENT_ID, CLIENT_SECRET, DEVELOPER_TOKEN".
❌ "Usar OAuth padrão do Google." (sem valores, sem particularidades)

### Exemplo 3 — Checklist de Segurança
✅ "S-03 | PKCE obrigatório onde suportado | Google, LinkedIn, X, Pinterest"
❌ "Usar PKCE quando possível." (sem lista de plataformas, sem verificação)

## 17. Estrutura obrigatória de saída
Gere o documento final em **Markdown**, já bonito, limpo e estruturado.

Use obrigatoriamente:
- cabeçalho com H1 + H2
- tabela de metadados
- callout de TL;DR com ícone `📌` e
- hierarquia numerada multinível
- pseudocódigo em blocos de código para fluxos críticos
- tabelas comparativas por plataforma
- separadores visuais `---` entre todas as seções principais

Use estes padrões visuais:
- `🎯` para contexto estratégico e decisões de arquitetura
- `💡` para particularidades e boas práticas
- `✅` para configurações validadas
- `🔴` para alertas críticos de segurança e anti-patterns
- `⚙️` para requisitos técnicos e dependências

## 18. Cabeçalho obrigatório
Monte o cabeçalho com:
- **Destinatário:** Backend Lead / Tech Lead
- **Escopo:** Sessão de usuário, OAuth 2.0 por plataforma, tokens, segurança e refresh
- **Versão:** v1.0
- **Responsável:** Claude Code Desktop
- **Data da versão:** data e hora atual no fuso `America/Fortaleza`

## 19. TL;DR obrigatório
Logo após o cabeçalho, insira um callout `📌` com no máximo 7 bullets resumindo:
- tipos de autenticação cobertos
- regras de sessão (lifetime, 2FA)
- regra de vault (nunca plain text)
- total de plataformas OAuth documentadas
- state CSRF e PKCE
- estratégia de refresh proativo
- se existem seções pendentes

## 20. Seções obrigatórias do documento final
Siga exatamente esta ordem.

### 1. Persona
Contextualização do público-alvo e impacto no pipeline.

### 2. Dois Tipos de Auth
Distinção clara entre Tipo 1 (Sessão de Usuário) e Tipo 2 (Conexão de Plataforma) com pseudocódigo.

### 3. Tipo 1 — Sessão de Usuário
Fluxo de login, 2FA TOTP, tabela de segurança e endpoints.

### 4. Tipo 2 — OAuth 2.0 de Plataformas
Fluxo genérico (Authorization Code + PKCE), callback URL, vault de tokens.

### 5. OAuth por Plataforma
Tabela por plataforma com lifetime, refresh, PKCE, credenciais e particularidades.

### 6. Refresh Proativo e Reconexão
Job de refresh, estados de integração, circuit breaker.

### 7. Checklist de Segurança
Tabela com verificação e implementação para cada regra.

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
- **Persona:** público + impacto + dependências.
- **Tipos de Auth:** distinção clara + pseudocódigo resumo.
- **Sessão:** login + 2FA + segurança + endpoints.
- **OAuth genérico:** fluxo 3 etapas + callback + vault.
- **OAuth por plataforma:** todas as plataformas com tabela completa.
- **Refresh:** job proativo + estados + circuit breaker.
- **Checklist:** mínimo 8 verificações de segurança.

## 22. Autoauditoria obrigatória antes da resposta
Antes de responder, execute internamente este ciclo completo:

1. Gere o rascunho completo.
2. Audite contra a estrutura obrigatória e cobertura mínima.
3. Verifique que todo fluxo tem pseudocódigo com tratamento de erro.
4. Verifique que toda plataforma tem tabela com lifetime, PKCE e credenciais.
5. Verifique que o checklist de segurança cobre todos os itens obrigatórios.
6. Verifique que tokens nunca aparecem em plain text.
7. Corrija qualquer ausência, contradição ou lacuna.
8. Só então responda com a versão final.

## 23. Critérios de qualidade obrigatórios
Antes de responder, valide internamente:

**Formato e estrutura**
- há pseudocódigo para todos os fluxos críticos
- cada plataforma tem tabela com campos obrigatórios
- checklist de segurança está completo

**Técnica e segurança**
- tokens nunca em plain text
- state CSRF em todos os fluxos OAuth
- PKCE documentado por plataforma
- refresh proativo com buffer configurável

**Cobertura**
- dois tipos de auth documentados
- todas as plataformas integradas cobertas
- endpoints de sessão completos
- vault documentado por ambiente
- estados de integração com transições
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
| 20/03/2026 | v1.4 | Filosofia de decisão autônoma: hierarquia [DECISÃO AUTÔNOMA] > [DEFINIÇÃO PENDENTE] > [SEÇÃO PENDENTE]. Regra máxima, ambiguidade e backlog atualizados. |
| 20/03/2026 | v1.3 | Reestruturação completa: documento convertido de formato pronto para **Prompt Generator** (padrão Pipeline ShiftLabs). |
| 18/03/2026 | v1.2 | Padronização da abertura do documento para o formato TL;DR → Persona. |
| 17/03/2026 | v1.1 | Redis keys padronizadas com prefixo `portal:`. |
| 17/03/2026 | v1.0 | Criação. Sessão Supabase Auth + 2FA + OAuth 2.0 por plataforma. |