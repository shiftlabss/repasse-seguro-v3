# S5 — Dossiê e Contatos

## CRM Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Sprint** | S5 |
| **Nome** | Dossiê e Contatos |
| **Template** | B — Módulo Fullstack |
| **Docs fonte** | 01.2, 01.3, 05.2, 05.3, 06, 16, 17, 27 |
| **REQs cobertos** | REQ-014, REQ-057 a REQ-067, REQ-137, REQ-154 a REQ-160, REQ-200 a REQ-204, REQ-211 a REQ-214, REQ-259, REQ-283 (parcial), REQ-301, REQ-309, REQ-311 |
| **Objetivo** | Analista RS faz upload de documentos no Dossiê, Coordenador RS aprova, link de compartilhamento controlado gerado. Contatos criados, perfil completo, mesclagem de duplicatas, arquivamento automático. |

---

## Auto-verificação (12 checks)

- [x] C1: Nomes exatos — `dossier_documents`, `contacts`, `DossierState`, `document_type` exatos, 11 documentos obrigatórios nomeados
- [x] C2: Todos os itens binariamente verificáveis
- [x] C3: Valores exatos — 20 MB tamanho máximo, PDF/JPG/PNG formatos, Tabela Atual ≤ 30 dias, link compartilhamento 7 dias, Comprovante endereço ≤ 90 dias, SLA aprovação 2 dias úteis, 12 meses arquivamento Contato
- [x] C4: Glossário — Dossiê, Contato vs Usuário (distinção crítica), Investidor Recorrente cobertos
- [x] C5: TODAS as transições DossierState: Em montagem → Completo → Aprovado / Rejeitado → Completo (loop)
- [x] C6: Arquivamento mensal de Contatos inativos (12 meses), SLA de aprovação Dossiê (2 dias úteis)
- [x] C7: RBAC — aprovação Dossiê apenas Coordenador RS / Admin RS; upload todos os usuários internos; mesclagem apenas Coordenador RS
- [x] C8: T-CRM-040/041/042/050/051/052/053 com estados
- [x] C9: Supabase Storage para documentos (signed URLs, UUID v4 nome)
- [x] C10: Glossário — distinção Contato vs Usuário, estados Contato (Ativo/Sem Caso Ativo/Arquivado)
- [x] C11: Anti-scaffold — validação de tipo/tamanho arquivo, controle de versão docs, LGPD no compartilhamento
- [x] C12: REQ-014 e REQ-057 a REQ-067 cobertos

---

## FEATURE 1 — Dossiê

### Banco / Migrations

- [x] **REQ-137** Verificar enum `DossierState` no schema: `EM_MONTAGEM`, `COMPLETO`, `APROVADO`, `REJEITADO`. Verificar tabela `dossier_documents` criada em S1 com campos: `document_type` (string com valores dos 11 tipos obrigatórios), `file_url` (Supabase Storage URL), `file_name`, `file_size_bytes`, `mime_type`, `status (PENDENTE|APROVADO|REJEITADO)`, `version Int @default(1)`, `rejection_reason String?`, `approved_by UUID? FK`, `origin (ANALISTA_RS|CEDENTE_PLATAFORMA)`
  - Validar: enum completo; campo `version` para controle de versões; `origin` distingue quem enviou

- [x] Adicionar coluna `dossier_state DossierState @default(EM_MONTAGEM)` na tabela `cases`. Adicionar migration
  - Validar: Casos existentes têm `dossier_state = EM_MONTAGEM`

### Backend

- [x] **REQ-211** `GET /cases/:id/dossier` — Retornar estado do Dossiê + lista de documentos com versão atual de cada tipo + metadados (quem enviou, data, status). Roles: todos internos; Parceiro Externo: apenas documentos marcados como compartilháveis (sem dados pessoais do Cedente)
  - Validar: retorna 11 slots de documentos com status (presente/ausente) para cada tipo; Parceiro Externo não vê identidade/CPF/endereço do Cedente

- [x] **REQ-212 / REQ-058 / REQ-059** `POST /cases/:id/dossier/documents` — multipart/form-data: `file (obrigatório)`, `document_type (obrigatório)`, `origin (default: ANALISTA_RS)`. Validações:
  - Formato: apenas `application/pdf`, `image/jpeg`, `image/png` → outros → 422 `CRM-026`: "Formato não aceito. Envie arquivos em PDF, JPG ou PNG."
  - Tamanho: máximo 20.971.520 bytes (20 MB) → maior → 422 `CRM-027`: "O arquivo é muito grande. O limite é de 20 MB. Comprima o arquivo e tente novamente."
  - Upload via Supabase Storage: bucket `dossier/{caseId}/{document_type}/`, nome do arquivo = `{UUID-v4}.{ext}`, URL assinada armazenada em `file_url`
  - Se documento deste tipo já existe: incrementar `version` + arquivar versão anterior (manter em `dossier_documents` com `deleted_at` preenchido)
  - Criar `audit_log` com `action = DOCUMENT_UPLOADED`
  - Validar: arquivo PNG > 20MB → 422 CRM-027; arquivo PDF válido → criado com file_url Supabase; reenvio de mesmo tipo → version incrementado; versão anterior arquivada

- [x] **REQ-060 / REQ-059** Implementar lógica de aprovação do Dossiê em `DossierService`:
  - `requestApproval(caseId)`: verifica se todos os 11 documentos obrigatórios estão presentes (baseado no estado do Caso + tabela RN-058). Para Cenário A: Valor Distrato Referência não obrigatório. Para estado anterior à Formalização: instrumento cessão, anuência e comprovante Escrow não obrigatórios. Atualiza `cases.dossier_state = COMPLETO`. Notifica Coordenador RS para revisão (fila RabbitMQ). SLA: 2 dias úteis para aprovação.
  - Validar: requestApproval sem todos os documentos obrigatórios → 422 com lista de faltantes; com todos → dossier_state COMPLETO + notificação Coordenador RS

- [x] **REQ-213 / REQ-060** `POST /cases/:id/dossier/approve` — Roles: `ADMIN_RS`, `COORDENADOR_RS`. Request body: `{ action: APPROVE|REJECT, rejection_notes?: string[] }`. Lógica:
  - `APPROVE`: atualizar `cases.dossier_state = APROVADO`, `approved_by = userId`, `audit_log`. Habilita avanço para `PUBLICACAO`
  - `REJECT`: atualizar `cases.dossier_state = REJEITADO`. `rejection_notes` obrigatório: lista de documentos com pendência e motivo. Notificar Analista RS: "Dossiê rejeitado com pendências: [lista]."
  - Analista RS → 403 `CRM-028`
  - Validar: aprovação por Analista RS → 403; rejeição sem `rejection_notes` → 422; aprovação → `dossier_state = APROVADO`; rejeição → notificação Analista RS com pendências

- [x] **REQ-062 / REQ-063** Implementar gate de integridade documental em `CasesService.advance()` para transição `VERIFICACAO → PUBLICACAO`:
  - Verificar: `dossier_state = APROVADO`
  - Verificar: todos os documentos obrigatórios para publicação presentes e com status `APROVADO`
  - Verificar: Tabela Atual com `created_at` ≤ 30 dias da data atual
  - Se Tabela Atual expirada (> 30 dias): 422 `CRM-029`: "A Tabela Atual está com mais de 30 dias. Reenvie uma tabela atualizada antes de publicar."
  - Validar: Caso com Dossiê APROVADO mas Tabela Atual de 45 dias atrás → 422 CRM-029; Dossiê APROVADO + Tabela Atual de 20 dias → transição permitida

- [x] **REQ-061** Implementar controle de versão de documentos: ao aprovar um documento e receber novo upload do mesmo tipo:
  - Se `dossier_state = APROVADO`: exigir justificativa para substituição (`{ justification: string (mínimo 20 chars) }`) + nova aprovação pelo Coordenador RS
  - Se `dossier_state` ≠ `APROVADO`: permitir substituição diretamente (incrementando version)
  - `GET /cases/:id/dossier/documents/:docId/history`: retornar todas as versões do documento
  - Validar: substituição de doc em Dossiê APROVADO sem justificativa → 422; com justificativa → versão criada + dossier_state volta para EM_MONTAGEM; histórico de versões retorna versões anteriores

- [x] **REQ-214 / REQ-062** `POST /cases/:id/dossier/share-link` — Roles: `ADMIN_RS`, `COORDENADOR_RS`, `ANALISTA_RS`. Regras:
  - Dossiê deve estar `APROVADO` → senão 422 `CRM-030`
  - Gerar link temporário via Supabase Storage Signed URL com expiração em 7 dias (604800 segundos)
  - O link inclui APENAS: Tabela Atual, Tabela do Contrato, extrato saldo devedor, instrumento de cessão (após assinatura). NUNCA inclui: identidade, CPF, endereço do Cedente (LGPD)
  - Registrar acesso no `audit.audit_logs` quando link é gerado (e idealmente quando acessado via webhook Supabase Storage)
  - Validar: gerar link com Dossiê não aprovado → 422 CRM-030; link gerado tem expiração correta de 7 dias; documentos pessoais do Cedente NÃO incluídos nas URLs geradas

---

## FEATURE 2 — Contatos

### Banco / Migrations

- [x] **REQ-309** Verificar na tabela `contacts` os estados via campo `status`: `ATIVO`, `SEM_CASO_ATIVO`, `ARQUIVADO`. Criar index em `contacts.cpf_cnpj` e `contacts.email` para detecção de duplicatas
  - Validar: index criado; `contacts.status` com enum correto; query por CPF/CNPJ usa index

### Backend

- [x] **REQ-200** `GET /contacts` — Roles: todos internos (escopo por role). Query params: `role (CEDENTE|CESSIONARIO|PARCEIRO|INCORPORADORA)`, `status (ATIVO|SEM_CASO_ATIVO|ARQUIVADO)`, `search`, `is_possible_duplicate`, `is_recurrent_investor`, `page`, `per_page`. Dados pessoais mascarados em listagens conforme REQ-012. Archived contacts somente com filtro `status=ARQUIVADO`
  - Validar: `ANALISTA_RS` → apenas Contatos de seus Casos; listagem sem filtro não inclui arquivados; dados mascarados (nome "João S.")

- [x] **REQ-201 / REQ-014** `POST /contacts` — Roles: todos internos. Request body: `{ full_name, email, phone, cpf_cnpj?, role, case_id? }`. Lógica:
  - Verificar duplicata por `email` ou `cpf_cnpj` em Contatos não deletados
  - Se duplicata encontrada: retornar 409 `CRM-031` com dados do Contato existente + `{ action_required: "LINK_OR_CREATE" }`
  - Se `action = LINK`: associar `case_id` ao Contato existente (via `cases.cedente_contact_id` ou `cessionario_contact_id`)
  - Se `action = CREATE`: criar novo Contato com `is_possible_duplicate = true`
  - Criar `audit_log`
  - Validar: CPF duplicado → 409 com CRM-031 e dados do existente; criar com flag duplicata → `is_possible_duplicate = true`; vincular ao existente → histórico preservado

- [x] **REQ-202** `GET /contacts/:id` — Dados completos (sem mascaramento) para Admin RS, Coordenador RS, e Analista RS responsável por Caso vinculado ao Contato. Mascarado para outros. Parceiro Externo → 403. Incluir: perfil, histórico de Casos, histórico de Atividades, flag opt-out, flag duplicata
  - Validar: Analista RS não responsável pelo Caso → dados mascarados; Admin RS → dados completos; log de acesso registrado

- [x] **REQ-203** `PATCH /contacts/:id` — Roles: todos internos (com escopo). Campos editáveis: `full_name`, `phone`, `email`, `cpf_cnpj`, `notas_internas`, `perfil_investidor`, `tipo_imovel_preferido`, `indicador_id (Parceiro)`. Não editável pelo Analista RS: `role`, `is_recurrent_investor` (apenas Analista pode marcar via endpoint específico)
  - Validar: Analista RS não consegue alterar `role` do Contato; `email` alterado → verificar duplicata novamente; audit_log criado

- [x] **REQ-204 / REQ-065** `POST /contacts/:id/merge` — Roles: `ADMIN_RS`, `COORDENADOR_RS`. Request body: `{ secondary_contact_id: UUID }`. Lógica:
  - O Contato com mais Casos vinculados é o principal; o outro é o secundário
  - Transferir todos os Casos, Atividades e histórico do Contato secundário para o principal
  - Marcar Contato secundário com `status = MESCLADO` (novo enum value) e `primary_contact_id = principal.id`. NUNCA excluir — preservar para auditoria
  - Se Analista RS tentar → 403
  - Validar: após mesclagem, GET no secundário retorna dados com flag MESCLADO; Casos do secundário aparecem em GET no principal; secundário não retorna em buscas padrão

- [x] **REQ-064 / REQ-094** `POST /contacts/:id/mark-recurrent` — Roles: `ANALISTA_RS` (apenas Casos próprios), `COORDENADOR_RS`, `ADMIN_RS`. Requisito: pelo menos 1 Fechamento concluído para este Cessionário. Marcar `is_recurrent_investor = true`. Atualizar campos de preferências: `investment_profile`, `preferred_property_type`, `preferred_region`, `value_range_min`, `value_range_max`
  - Validar: marcar como recorrente sem Fechamento concluído → 422 `CRM-032`; com Fechamento → flag ativada + badge na listagem

- [x] **REQ-066 / REQ-096** Implementar `ContactArchivingWorker` (cron mensal, mesmo job do `CaseArchivingWorker`): buscar Contatos com `status ≠ ARQUIVADO`, sem Caso ativo (`cases.state NOT IN (CADASTRO,...,FORMALIZACAO)` ou sem Casos), sem Atividade registrada há 12 meses. Arquivar: `status = ARQUIVADO`. Notificar Analista RS da última interação
  - Validar: Contato sem Atividade há 13 meses → arquivado; Contato com Caso ativo → não arquivado mesmo após 12 meses; reativação manual via criação de novo Caso → status volta para ATIVO

- [x] **REQ-067 / REQ-097** Validar na criação/edição de Caso: campo `indicador_cedente_id` e `indicador_cessionario_id` com máximo 2 indicadores (1 para Cedente + 1 para Cessionário). Ao vincular indicador: verificar que é `Parceiro Externo` com conta ativa. NÃO calcular comissão de parceria automaticamente
  - Validar: tentar adicionar 2 indicadores para o Cedente → 422 CRM-033; Parceiro Externo indicador recebe acesso restrito ao status do Caso

### Frontend

- [x] **REQ-154** Implementar `app/(dashboard)/cases/[id]/dossier/page.tsx` — T-CRM-040 Checklist de Documentos:
  - Checklist com 11 slots de documentos (nomes exatos conforme RN-058)
  - Cada slot: ícone de status (⬜ ausente / 🕐 pendente / ✅ aprovado / ❌ rejeitado), nome do documento, responsável pelo envio, data do upload, botão upload
  - Botão "Solicitar Aprovação" (habilitado apenas quando todos os obrigatórios estão presentes)
  - Botão "Aprovar Dossiê" / "Rejeitar Dossiê" (visível apenas Coord/Admin)
  - Indicador de SLA de aprovação (2 dias úteis)
  - Validar: todos os 11 documentos listados com nomes exatos; botão "Solicitar Aprovação" desabilitado se faltam documentos; Dossiê APROVADO mostra badge verde

- [x] **REQ-155** Implementar `app/(dashboard)/cases/[id]/dossier/[docId]/page.tsx` — T-CRM-041 Detalhe do Documento:
  - Preview do documento (PDF via iframe, imagens via `<img>`)
  - Metadados: data upload, quem enviou (origem), versão atual, status
  - Botão "Ver Histórico de Versões" → lista versões anteriores com data e quem enviou
  - Botão "Substituir documento" (cria nova versão; se Dossiê APROVADO → exige justificativa)
  - Validar: PDF renderiza no iframe; histórico de versões exibe versões em ordem decrescente; substituição com Dossiê APROVADO abre campo de justificativa obrigatório

- [x] **REQ-156** Implementar `DossierApprovalModal` (T-CRM-042) para Coordenador RS / Admin RS:
  - Revisão de cada documento com botões "Aprovar" / "Rejeitar com feedback"
  - Campo de texto por documento rejeitado: motivo da rejeição
  - Botão final "Aprovar Dossiê Completo" / "Rejeitar Dossiê com Pendências"
  - Validar: rejeição sem motivo → campo de texto obrigatório; aprovação → toast "Dossiê aprovado. Caso pode avançar para Publicação."

- [x] **REQ-157** Implementar `app/(dashboard)/contacts/page.tsx` — T-CRM-050 Lista de Contatos:
  - TanStack Table: nome (mascarado), papel, status, Casos vinculados (contagem), data última atividade
  - Filtros: papel, status, possível duplicata, investidor recorrente
  - Badge "Investidor Recorrente" nos Cessionários qualificados
  - Badge "Possível Duplicata" nos Contatos flagados
  - Empty state: "Nenhum contato encontrado para os filtros selecionados."
  - Validar: Analista RS vê apenas Contatos de seus Casos; filtro "Possível Duplicata" mostra apenas flagados; Contatos arquivados ocultos por padrão

- [x] **REQ-158** Implementar `app/(dashboard)/contacts/[id]/page.tsx` — T-CRM-051 Perfil do Contato:
  - Dados completos (CPF/email visíveis para responsável/Coord/Admin, mascarados para outros)
  - Abas: Casos vinculados, Atividades, Comunicações, Notas internas
  - Indicador de opt-out com flag LGPD visível se ativo
  - Botão "Mesclar com duplicata" (apenas Coord/Admin)
  - Validar: opt-out ativo → banner LGPD vermelho no topo; dados pessoais mascarados para Analista não responsável

- [x] **REQ-159** Implementar `app/(dashboard)/contacts/new/page.tsx` e `/contacts/[id]/edit/page.tsx` — T-CRM-052:
  - React Hook Form + Zod: `full_name`, `email`, `phone`, `cpf_cnpj (formatação automática)`, `role (select)`, campos opcionais por papel
  - Detecção de duplicata em tempo real: ao blur do campo `email` ou `cpf_cnpj` → fazer `GET /contacts?email=X` → se duplicata → exibir alerta inline antes de submeter
  - Validar: CPF com formatação automática (XXX.XXX.XXX-XX); e-mail duplicado detectado antes de submeter; campos opcionais de Cessionário aparecem apenas se `role = CESSIONARIO`

- [x] **REQ-160** Implementar `MergeContactsModal` (T-CRM-053) para Coordenador RS / Admin RS:
  - Exibir lado a lado: Contato principal (com mais Casos) vs Contato secundário
  - Lista de Casos, Atividades que serão transferidos
  - Botão "Confirmar Mesclagem" com aviso: "Esta ação não pode ser desfeita. O Contato secundário ficará inacessível para novos vínculos."
  - Botão "Não mesclar (Falso positivo)": descarta o alerta e remove a flag `is_possible_duplicate`
  - Validar: confirmação de mesclagem → toast "Contatos mesclados com sucesso."; "Não mesclar" → flag removida; mesclagem de Contato sem Casos → secundário com 0 Casos é o secundário

---

## 🧪 Testes

### Backend — Unitário (Jest)

- [x] **REQ-259** `ContactsService` — cobertura 70%:
  - `should detect duplicate contact by email`
  - `should detect duplicate contact by cpf_cnpj`
  - `should mark new contact as possible_duplicate`
  - `should transfer cases and activities on merge`
  - `should archive contact after 12 months of inactivity`
  - `should block merge by ANALISTA_RS`

### Backend — Integração

- [x] `POST /cases/:id/dossier/documents` com arquivo > 20 MB → 422 CRM-027
- [x] `POST /cases/:id/dossier/documents` com `.docx` → 422 CRM-026
- [x] `POST /cases/:id/dossier/share-link` com Dossiê não aprovado → 422 CRM-030
- [x] `POST /contacts/:id/merge` por Analista RS → 403

---

## 🔀 Cross-Módulo

- Aprovação do Dossiê (`dossier_state = APROVADO`) desbloqueia o gate em `CasesService.advance()` para `VERIFICACAO → PUBLICACAO` (implementado em S3 como placeholder).
- Os documentos `INSTRUMENTO_CESSAO`, `ANUENCIA` e `COMPROVANTE_ESCROW` são verificados em `CommissionsService.confirmFechamento()` (implementado em S4). Após S5, o gate de Fechamento está funcional.
- Supabase Storage (REQ-301) configurado aqui é consumido pela integração ZapSign (S7) para armazenar PDFs assinados automaticamente.
