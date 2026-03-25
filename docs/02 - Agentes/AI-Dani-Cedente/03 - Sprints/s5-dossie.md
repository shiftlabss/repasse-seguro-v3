# S5 — Dossiê

| Campo | Valor |
|---|---|
| **Sprint** | S5 |
| **Nome** | Dossiê |
| **Tipo** | Módulo Fullstack (Template B) |
| **Template** | B (Módulo Fullstack: por Feature, vertical slice Banco→Backend→Frontend→Wiring→Testes) |
| **Docs Consultados** | D01 (RN-DCE-013), D05.3 (RF-DCE-014, RF-DCE-015), D12, D13, D16 (domínio Dossier), D17 (Supabase Storage) |
| **Cross-cutting** | D10 (Glossário) |
| **REQs cobertos** | REQ-046, REQ-047, REQ-048, REQ-049, REQ-050, REQ-051, REQ-052, REQ-053, REQ-054, REQ-055, REQ-056, REQ-057, REQ-058, REQ-059, REQ-060 |
| **Objetivo** | Upload de documentos via Supabase Storage (`multipart/form-data`, max 10MB, PDF/imagem), 6 tipos de documento (`DossierDocumentType`), 4 estados por documento (PENDING/IN_REVIEW/APPROVED/REJECTED), percentual de conclusão, motivo de rejeição, endpoints `GET /opportunities/:id/dossier`, `POST /dossier/documents`, `GET /dossier/documents/:id`, `DELETE /dossier/documents/:id`, testes unitários e E2E |

---

## Critério de Conclusão da S5

Ao final desta sprint:
- `GET /opportunities/:opportunity_id/dossier` retorna todos os documentos com `status`, `rejection_reason` e `expires_at`
- `POST /opportunities/:opportunity_id/dossier/documents` aceita `multipart/form-data` com `document_type`, `file` (max 10MB, PDF ou imagem) e `document_date` opcional — faz upload ao Supabase Storage e persiste registro com status `PENDING`
- `DELETE /opportunities/:opportunity_id/dossier/documents/:document_id` executa soft delete (`deleted_at`)
- Documento `IN_REVIEW` bloqueia novo upload do mesmo tipo → 409 `DCE-DOSS-4090_001`
- `dossier_summary` calculado: `(documentos APPROVED / total obrigatórios) × 100` — presente em `GET /opportunities/:id`
- Tipos de documento mapeados: `CONTRACT_ORIGINAL`, `PROPERTY_REGISTRATION`, `NEGATIVE_ÔNUS`, `NEGATIVE_DEBTS`, `PAYMENT_PROOF`, `POWER_OF_ATTORNEY`
- Documentos com prazo: `NEGATIVE_ÔNUS` e `NEGATIVE_DEBTS` têm `expires_at` calculado como `document_date + 90 dias`; `expires_at` nulo para demais tipos
- Arquivo nunca armazenado localmente — exclusivamente via Supabase Storage

---

## ⚙️ FEATURE 1 — Armazenamento e Modelo de Dados

### Banco

- [x] **Verificar que migration da tabela `dossier_documents`** já existe no S1 com todos os campos: `id UUID PK`, `opportunity_id UUID FK → opportunities.id`, `document_type DossierDocumentType NOT NULL`, `status DocumentStatus NOT NULL DEFAULT 'PENDING'`, `storage_path TEXT NOT NULL`, `document_date DATE`, `expires_at DATE`, `rejection_reason TEXT`, `approved_at TIMESTAMPTZ`, `deleted_at TIMESTAMPTZ`, `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ`
  - Validação: `\d dossier_documents` mostra todos os campos; `storage_path` é NOT NULL (arquivo obrigatório para registro existir); `expires_at` pode ser NULL (documentos sem validade); `rejection_reason` pode ser NULL (documentos não rejeitados)
- [x] **Verificar enum `DossierDocumentType`** já definido no S1 com exatamente os 6 valores: `CONTRACT_ORIGINAL`, `PROPERTY_REGISTRATION`, `NEGATIVE_ÔNUS`, `NEGATIVE_DEBTS`, `PAYMENT_PROOF`, `POWER_OF_ATTORNEY`
  - Validação: `SELECT enum_range(NULL::DossierDocumentType)` retorna os 6 valores exatos; nenhum valor extra; nenhum valor faltando
- [x] **Verificar enum `DocumentStatus`** já definido no S1 com exatamente os 4 valores: `PENDING`, `IN_REVIEW`, `APPROVED`, `REJECTED`
  - Validação: `SELECT enum_range(NULL::DocumentStatus)` retorna os 4 valores; transição APPROVED → REJECTED impossível sem passar por IN_REVIEW (constraint ou guard de estado)
- [x] **Verificar RLS em `dossier_documents`** via JOIN com `opportunities`: policy `USING (opportunity_id IN (SELECT id FROM opportunities WHERE cedente_id = current_setting('app.current_cedente_id')::uuid AND deleted_at IS NULL))`
  - Validação: Cedente A não visualiza documentos da oportunidade do Cedente B; query cruzada retorna zero linhas; policy ativa para SELECT, INSERT, UPDATE, DELETE

### Backend — Integração Supabase Storage

- [x] **Implementar `StorageService`** em `src/modules/dossier/storage.service.ts`: (1) injeta `@supabase/supabase-js` client configurado com `SUPABASE_URL` e `SUPABASE_SERVICE_ROLE_KEY`; (2) método `uploadDocument(file: Express.Multer.File, opportunityId: string, documentType: string): Promise<string>`: (a) gera path `dossier/{opportunity_id}/{document_type}-{timestamp}.{ext}`; (b) chama `supabase.storage.from('dossier-documents').upload(path, buffer, { contentType: file.mimetype })`; (c) retorna `storage_path` string; (d) se upload falhar → lança `DaniExternalError('DCE-DOSS-5020_001', 'Falha ao enviar arquivo. Tente novamente.')` com HTTP 502; (3) método `deleteDocument(storagePath: string): Promise<void>`: remove arquivo do bucket ao fazer soft delete do registro
  - Validação: path gerado segue formato `dossier/{uuid}/{type}-{epoch}.pdf`; upload de arquivo de 1MB retorna `storage_path` válido; mock do Supabase storage em testes unitários; `SUPABASE_SERVICE_ROLE_KEY` usada apenas no backend — nunca exposta ao frontend

---

## ⚙️ FEATURE 2 — CRUD de Documentos do Dossiê

### Backend — DossierService

- [x] **Implementar `DossierService.getDossier(opportunityId: string, cedenteId: string)`** em `src/modules/dossier/dossier.service.ts`: (1) verifica ownership da oportunidade; (2) busca todos os `dossier_documents WHERE opportunity_id = opportunityId AND deleted_at IS NULL`; (3) calcula `dossier_summary`: `total_obrigatorios = 5` (excluindo `POWER_OF_ATTORNEY` que é condicional), `approved = count(status = 'APPROVED')`, `complete = approved === 5`, `completion_percentage = (approved / 5) × 100`; (4) retorna `{ opportunity_id, total: count, complete, completion_percentage, documents: [...] }`
  - Validação: 5 documentos aprovados → `complete: true`, `completion_percentage: 100`; 0 documentos → `completion_percentage: 0`; `POWER_OF_ATTORNEY` não conta no denominador do percentual; `deleted_at IS NOT NULL` excluído da listagem

- [x] **Implementar `DossierService.uploadDocument(dto: UploadDocumentDto, cedenteId: string)`**: (1) verifica ownership da oportunidade; (2) verifica se já existe documento com mesmo `document_type` e `status IN ('PENDING', 'IN_REVIEW')` → se existir lança `DaniConflictError('DCE-DOSS-4090_001', 'Documento deste tipo já existe e está em análise.')` com HTTP 409; (3) chama `StorageService.uploadDocument(file, opportunityId, document_type)`; (4) calcula `expires_at`: se `document_type IN ('NEGATIVE_ÔNUS', 'NEGATIVE_DEBTS')` E `document_date` fornecido → `expires_at = document_date + 90 dias`; senão `expires_at = null`; (5) persiste `DossierDocument` com `status = 'PENDING'`, `storage_path`, `expires_at` calculado; (6) retorna documento criado
  - Validação: upload de `NEGATIVE_ÔNUS` com `document_date = '2026-03-20'` → `expires_at = '2026-06-18'`; upload de `CONTRACT_ORIGINAL` → `expires_at = null`; documento `IN_REVIEW` existente → 409 `DCE-DOSS-4090_001`; documento `APPROVED` existente permite novo upload (substituição)

- [x] **Implementar `DossierService.getDocument(documentId: string, opportunityId: string, cedenteId: string)`**: (1) verifica ownership da oportunidade; (2) busca `dossier_document WHERE id = documentId AND opportunity_id = opportunityId AND deleted_at IS NULL`; (3) lança `DaniNotFoundError('DCE-DOSS-4040_001')` se não encontrado; (4) retorna documento completo com todos os campos incluindo `rejection_reason` e `storage_path`
  - Validação: ID de documento de outra oportunidade → 404; ID com `deleted_at` preenchido → 404; documento com `rejection_reason` → campo presente no response

- [x] **Implementar `DossierService.deleteDocument(documentId: string, opportunityId: string, cedenteId: string)`**: (1) verifica ownership; (2) busca documento; (3) se `status = 'IN_REVIEW'` → lança `DaniBusinessError('DCE-DOSS-4220_001', 'Não é possível remover documento em análise.')` com HTTP 422; (4) atualiza `deleted_at = now()` (soft delete); (5) chama `StorageService.deleteDocument(storage_path)` para remover do Supabase Storage; (6) retorna `{ message: 'Documento removido com sucesso.' }`
  - Validação: documento com `status = 'IN_REVIEW'` → 422 `DCE-DOSS-4220_001`; soft delete não apaga o arquivo do banco; `storage_path` removido do Supabase Storage após soft delete; documento removido não aparece em `GET /dossier`

### Backend — DossierController

- [x] **Implementar `GET /api/v1/opportunities/:opportunity_id/dossier`** em `DossierController`: (1) aplica `JwtAuthGuard` + `CedenteIsolationMiddleware`; (2) extrai `cedente_id` via `@CurrentCedente()`; (3) chama `DossierService.getDossier(opportunityId, cedenteId)`; (4) retorna `{ data: { opportunity_id, total, complete, completion_percentage, documents: [{ id, document_type, status, approved_at, rejection_reason, expires_at }] } }`
  - Validação: token válido + oportunidade própria → 200 com `documents` array; oportunidade de outro Cedente → 403; sem token → 401; `storage_path` **não retornado** no response (dado interno)

- [x] **Implementar `POST /api/v1/opportunities/:opportunity_id/dossier/documents`** em `DossierController`: (1) aplica `JwtAuthGuard` + `CedenteIsolationMiddleware`; (2) usa `@UseInterceptors(FileInterceptor('file'))` do Multer; (3) valida `document_type` é enum válido de `DossierDocumentType`; (4) valida `file.size ≤ 10 * 1024 * 1024` (10MB) → se exceder lança `DCE-DOSS-4000_003`; (5) valida `file.mimetype IN ('application/pdf', 'image/jpeg', 'image/png', 'image/webp')` → se inválido lança `DCE-DOSS-4000_004`; (6) chama `DossierService.uploadDocument(dto, cedenteId)`; (7) retorna `{ data: { id, document_type, status, storage_path, document_date, expires_at, created_at } }` com HTTP 201
  - Validação: arquivo de 11MB → 400 `DCE-DOSS-4000_003`; arquivo `.xlsx` → 400 `DCE-DOSS-4000_004`; `document_type` ausente → 400 `DCE-DOSS-4000_001`; `document_type = 'INVALID'` → 400 `DCE-DOSS-4000_002`; upload válido → 201 com `storage_path`

- [x] **Implementar `GET /api/v1/opportunities/:opportunity_id/dossier/documents/:document_id`** em `DossierController`: (1) aplica guards; (2) chama `DossierService.getDocument(documentId, opportunityId, cedenteId)`; (3) retorna `{ data: { id, document_type, status, document_date, expires_at, rejection_reason, approved_at, created_at, updated_at } }`
  - Validação: documento com `status = 'REJECTED'` tem `rejection_reason` preenchido no response; `storage_path` ausente no response; ID inexistente → 404 `DCE-DOSS-4040_001`

- [x] **Implementar `DELETE /api/v1/opportunities/:opportunity_id/dossier/documents/:document_id`** em `DossierController`: (1) aplica guards; (2) chama `DossierService.deleteDocument(documentId, opportunityId, cedenteId)`; (3) retorna HTTP 200 com `{ message: 'Documento removido com sucesso.' }`
  - Validação: delete de documento `IN_REVIEW` → 422 `DCE-DOSS-4220_001`; delete bem-sucedido → 200; documento não aparece em `GET /dossier` após delete; arquivo removido do Storage

- [x] **Criar `src/modules/dossier/dto/upload-document.dto.ts`**: (1) `document_type: DossierDocumentType` com `@IsEnum(DossierDocumentType)` e `@IsNotEmpty()`; (2) `document_date?: string` com `@IsOptional()` e `@IsDateString()`; validação de formato `YYYY-MM-DD`
  - Validação: `document_type = 'INVALID'` → ValidationError com campo `document_type`; `document_date = '23/03/2026'` (formato incorreto) → ValidationError; body vazio → ValidationError em `document_type`

---

## ⚙️ FEATURE 3 — Estados e Transições de Documentos

### Backend — DocumentStatusGuard

- [x] **Implementar `DossierService.updateDocumentStatus(documentId, newStatus, rejectionReason?)`** (chamado apenas por Admin via webhook/painel — nunca diretamente pelo Cedente): (1) define transições válidas: `PENDING → IN_REVIEW`, `IN_REVIEW → APPROVED`, `IN_REVIEW → REJECTED`, `REJECTED → PENDING` (reenvio); (2) se transição inválida → lança `DaniBusinessError('DCE-DOSS-4220_002', 'Transição de status inválida.')` com HTTP 422; (3) se `newStatus = 'REJECTED'` e `rejectionReason` ausente → lança `DaniBusinessError('DCE-DOSS-4000_005', 'Motivo de rejeição obrigatório ao rejeitar documento.')` com HTTP 400; (4) atualiza `status`, `rejection_reason` (se rejeição), `approved_at` (se aprovação)
  - Validação: `PENDING → APPROVED` direto → 422 `DCE-DOSS-4220_002`; `IN_REVIEW → REJECTED` sem `rejectionReason` → 400 `DCE-DOSS-4000_005`; `IN_REVIEW → REJECTED` com motivo → status atualizado e `rejection_reason` persistido; `IN_REVIEW → APPROVED` → `approved_at = now()`

- [x] **Verificar que `GET /opportunities/:id/dossier` exibe `rejection_reason` por documento**: (1) se `status = 'REJECTED'` → `rejection_reason` retornado como string não nula; (2) se `status != 'REJECTED'` → `rejection_reason = null`; (3) orientação da Dani via contexto do agente: quando documento está `REJECTED`, ferramenta `get-dossier` retorna o `rejection_reason` para que o agente oriente o Cedente
  - Validação: documento REJECTED com motivo "Certidão com data superior a 90 dias." → campo `rejection_reason` com texto exato; documento APPROVED → `rejection_reason: null`; tool `get-dossier` do agente (S3) recebe `rejection_reason` no payload

---

## 🖥️ FRONTEND (Widget Chat — Dossiê)

### Componente DossierStatusCard

- [x] **Implementar `DossierStatusCard`** em `src/components/chat/DossierStatusCard.tsx`: (1) exibe lista de documentos com ícone por status: `APPROVED → ✅`, `IN_REVIEW → ⏳`, `REJECTED → ❌`, `PENDING → 📎`; (2) exibe `completion_percentage` como barra de progresso (0–100%); (3) se `complete = true`: exibe mensagem "Seu dossiê está completo. Sua oportunidade pode ser publicada."; (4) se `complete = false`: exibe percentual + sugere próximo documento (o primeiro `PENDING` ou `REJECTED` da lista); (5) estado **Skeleton**: 6 linhas de loading (uma por documento); (6) estado **Empty**: "Nenhum documento enviado ainda. Clique para iniciar." com botão de ação; (7) estado **Error**: "Não foi possível carregar o dossiê. [Tentar novamente]"; (8) estado **Populated**: lista completa com ícones e status
  - Validação: 5 documentos APPROVED → barra 100% + mensagem de conclusão; 0 documentos → barra 0% + mensagem Empty; 1 documento REJECTED → ícone ❌ + `rejection_reason` visível; 4 estados implementados e renderizados corretamente

### Componente DocumentStatusBadge

- [x] **Implementar `DocumentStatusBadge`** em `src/components/chat/DocumentStatusBadge.tsx`: (1) mapeamento de `DocumentStatus` → label PT-BR → cor: `PENDING → "Pendente" → cinza`, `IN_REVIEW → "Em análise" → amarelo`, `APPROVED → "Aprovado" → verde`, `REJECTED → "Rejeitado" → vermelho`; (2) props: `status: DocumentStatus`; (3) se `status = 'REJECTED'` e `rejectionReason` fornecido: exibe tooltip com o motivo de rejeição ao hover; (4) tipagem estrita — nenhum `status` retorna label vazio
  - Validação: 4 status mapeados com labels PT-BR exatos conforme D01/D05.3; tooltip com `rejection_reason` visível ao hover em status REJECTED; snapshot test cobre 4 variantes

### Componente DossierProgressBar

- [x] **Implementar `DossierProgressBar`** em `src/components/chat/DossierProgressBar.tsx`: (1) props: `completion_percentage: number` (0–100), `approved: number`, `total_obrigatorios: number`; (2) barra visual com fill proporcional; (3) label: `"{approved} de {total_obrigatorios} documentos aprovados ({completion_percentage}%)`; (4) cor da barra: vermelho se `< 50%`, amarelo se `50–99%`, verde se `100%`; (5) acessibilidade: `role="progressbar"`, `aria-valuenow={completion_percentage}`, `aria-valuemin={0}`, `aria-valuemax={100}`
  - Validação: `completion_percentage = 0` → barra vermelha vazia; `completion_percentage = 60` → barra amarela; `completion_percentage = 100` → barra verde cheia; label exibido com valores corretos; atributos ARIA presentes

---

## 🔀 Cross-Módulo

- `DossierService.getDossier()` é chamado pela ferramenta `get-dossier` do agente (S3) para apresentar status ao Cedente via chat — `DossierModule` deve exportar `DossierService` para injeção em `AgentModule`
- `dossier_summary` calculado em `OpportunityService.findOne()` (S4) usa `DossierService.getDossier()` internamente — garantir que não há N+1 query (incluir em `include` do Prisma ou sub-query otimizada)
- Transição `AWAITING_VALIDATION → DRAFT` (Admin rejeita dossiê — S4) dispara notificação "Documento do dossiê rejeitado" via canal de notificações (S8) — registrar como pendência no módulo de notificações
- Bloqueio de publicação (`kyc_status != APPROVED`) já implementado em S4 — não duplicar validação aqui

---

## 🧪 TESTES

- [x] **Criar `src/modules/dossier/dossier.service.spec.ts`** com testes unitários: (1) `getDossier` retorna `completion_percentage = 0` para 0 documentos APPROVED; (2) `getDossier` retorna `completion_percentage = 100` para 5 documentos APPROVED (excluindo POWER_OF_ATTORNEY no denominador); (3) `uploadDocument` com documento `IN_REVIEW` existente → lança `DaniConflictError('DCE-DOSS-4090_001')`; (4) `uploadDocument` de `NEGATIVE_ÔNUS` com `document_date` → `expires_at = document_date + 90 dias`; (5) `uploadDocument` de `CONTRACT_ORIGINAL` → `expires_at = null`; (6) `deleteDocument` com `status = IN_REVIEW` → lança `DaniBusinessError('DCE-DOSS-4220_001')`; (7) `deleteDocument` com `status = PENDING` → soft delete + `StorageService.deleteDocument()` chamado; (8) `updateDocumentStatus` com transição `PENDING → APPROVED` → lança `DCE-DOSS-4220_002`
  - Validação: 8 cenários; mock do Prisma e mock do `StorageService`; nenhuma chamada real ao Supabase Storage

- [x] **Criar `src/modules/dossier/dossier.controller.spec.ts`** com testes de integração do controller: (1) upload com arquivo de 11MB → 400 `DCE-DOSS-4000_003`; (2) upload com mimetype `text/plain` → 400 `DCE-DOSS-4000_004`; (3) upload sem `document_type` → 400 `DCE-DOSS-4000_001`; (4) upload com `document_type = 'INVALID_TYPE'` → 400 `DCE-DOSS-4000_002`
  - Validação: 4 cenários de validação de input no controller; mock do `DossierService`; nenhuma chamada real ao banco

- [x] **Criar `test/dossier.e2e-spec.ts`** com Supertest: (1) `GET /opportunities/:id/dossier` com token válido → 200 com `documents` array; (2) `GET /dossier` sem token → 401 `DCE-AUTH-4010_003`; (3) `GET /dossier` de oportunidade de outro Cedente → 403; (4) `POST /dossier/documents` com PDF válido < 10MB → 201 com `status: 'PENDING'`; (5) `POST /dossier/documents` com `document_type` duplicado `IN_REVIEW` → 409 `DCE-DOSS-4090_001`; (6) `DELETE /dossier/documents/:id` de documento `IN_REVIEW` → 422 `DCE-DOSS-4220_001`; (7) `DELETE /dossier/documents/:id` de documento `PENDING` → 200 com mensagem de sucesso
  - Validação: 7 cenários E2E com banco de teste; Supabase Storage mockado via `nock` ou jest mock; RLS verificado no cenário de isolamento

---

## 🔍 AUTO-VERIFICAÇÃO S5

| Check | Critério | Status |
|---|---|---|
| #1 Nomenclatura | `DossierService`, `DossierController`, `StorageService` — nomes exatos; tabela `dossier_documents`; enums `DossierDocumentType` e `DocumentStatus` conforme D12/D13/D16 | [x] |
| #2 Tipos de documento | 6 tipos exatos: `CONTRACT_ORIGINAL`, `PROPERTY_REGISTRATION`, `NEGATIVE_ÔNUS`, `NEGATIVE_DEBTS`, `PAYMENT_PROOF`, `POWER_OF_ATTORNEY` — nenhum tipo extra ou faltando | [x] |
| #3 Estados | 4 estados de documento: `PENDING`, `IN_REVIEW`, `APPROVED`, `REJECTED`; transições válidas implementadas; transição inválida → 422 `DCE-DOSS-4220_002` | [x] |
| #4 Expiração | `expires_at = document_date + 90 dias` para `NEGATIVE_ÔNUS` e `NEGATIVE_DEBTS`; `expires_at = null` para os demais 4 tipos | [x] |
| #5 Upload | `multipart/form-data`; max 10MB (`DCE-DOSS-4000_003`); tipos MIME: `application/pdf`, `image/jpeg`, `image/png`, `image/webp`; armazenamento exclusivo no Supabase Storage | [x] |
| #6 Percentual | `completion_percentage = (APPROVED / 5) × 100`; denominador = 5 (excluindo `POWER_OF_ATTORNEY`); `POWER_OF_ATTORNEY` conta se enviado mas não no denominador | [x] |
| #7 Rejeição | `rejection_reason` obrigatório ao mudar para `REJECTED` (`DCE-DOSS-4000_005`); motivo retornado no response quando `status = REJECTED` | [x] |
| #8 Isolamento | RLS via JOIN com `opportunities`; Cedente A nunca vê documentos de Cedente B; `storage_path` não exposto em responses públicos | [x] |
| #9 Soft Delete | `deleted_at` em `dossier_documents`; documento deletado não aparece em listagem; arquivo removido do Supabase Storage após soft delete | [x] |
| #10 Conflito | Novo upload do mesmo `document_type` com status `PENDING` ou `IN_REVIEW` → 409 `DCE-DOSS-4090_001`; upload permitido quando `APPROVED` ou `REJECTED` | [x] |
| #11 Anti-scaffold | `StorageService` com upload real ao Supabase Storage; `getDossier` com cálculo real de percentual; nenhum stub que retorna dados fixos | [x] |
| #12 Cobertura REQs | REQ-046 a REQ-060 — todos com ≥1 item no checklist | [x] |
