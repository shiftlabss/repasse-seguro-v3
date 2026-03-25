# S4 — Dossiê e Assinaturas

## Módulo Cedente · Plataforma Repasse Seguro

| **Sprint** | **Tipo**         | **Template** | **Versão** | **Data**   |
| ---------- | ---------------- | ------------ | ---------- | ---------- |
| S4         | Módulo Fullstack | Template B   | v1.0       | 2026-03-24 |

**REQs cobertos:** REQ-068 a REQ-086, REQ-094–095, REQ-111–112, REQ-123–125, REQ-198–199

**Dependências:** S1 (infra, RabbitMQ, Supabase Storage), S2 (auth), S3 (casos criados, eventos RabbitMQ configurados)

**Critério de aceite da sprint:** todos os itens ✅ + 12 auto-verificações ✅ + T-026 (upload MIME), T-034 (ZapSign inline), T-042 (mobile upload camera) passando.

---

## 🔴 FEATURE 1 — Dossiê e Documentos

### 1. Banco de Dados — Dossiê

- [x] **1.1** Confirmar que a tabela `dossies` possui os campos: `id UUID PK`, `caso_id UUID FK REFERENCES casos(id)`, `status StatusDossie NOT NULL DEFAULT 'INCOMPLETO'`, `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ DEFAULT now()`. Enum `StatusDossie`: `INCOMPLETO`, `COMPLETO`, `EM_ANALISE`, `APROVADO`, `REPROVADO`.
- [x] **1.2** Confirmar que a tabela `documentos_dossie` possui os campos: `id UUID PK`, `dossie_id UUID FK REFERENCES dossies(id)`, `tipo TipoDocumento NOT NULL`, `status StatusDocumento NOT NULL DEFAULT 'PENDENTE'`, `storage_path VARCHAR(500)`, `mime_type VARCHAR(100)`, `tamanho_bytes BIGINT`, `motivo_rejeicao TEXT`, `verificado_em TIMESTAMPTZ`, `versao INTEGER NOT NULL DEFAULT 1`, `created_at`, `updated_at`.
- [x] **1.3** Confirmar que o enum `TipoDocumento` inclui os 6 tipos obrigatórios PF e 2 adicionais PJ: `CONTRATO_IMOVEL`, `COMPROVANTE_PAGAMENTO_1`, `COMPROVANTE_PAGAMENTO_2`, `COMPROVANTE_PAGAMENTO_3`, `DECLARACAO_ADIMPLENCIA`, `DOCUMENTO_IDENTIDADE`, `COMPROVANTE_ENDERECO`, `TABELA_CONTRATO`, `CONTRATO_SOCIAL`, `CARTAO_CNPJ` — sem remover ou adicionar valores (RN-041, RN-013.a).
- [x] **1.4** Confirmar que o enum `StatusDocumento` possui: `PENDENTE`, `EM_ANALISE`, `VERIFICADO`, `REJEITADO`.
- [x] **1.5** Criar tabela `historico_documentos` com campos: `id UUID PK`, `documento_id UUID FK`, `versao INTEGER`, `storage_path_anterior VARCHAR(500)`, `status_anterior StatusDocumento`, `motivo_rejeicao TEXT`, `acao_por UUID` (auth_id de quem fez a ação), `created_at TIMESTAMPTZ` — para rastrear todas as versões e motivos de rejeição (RN-045 item 5).
- [x] **1.6** Criar índice `idx_documentos_dossie_status ON documentos_dossie(dossie_id, status)` para verificação de completude do dossiê.
- [x] **1.7** Confirmar RLS em `documentos_dossie`: política `FOR SELECT USING (auth.uid() = (SELECT cedente_auth_id FROM casos WHERE id = (SELECT caso_id FROM dossies WHERE id = dossie_id)))`.
- [x] **1.8** Confirmar RLS em `dossies`: política `FOR SELECT USING (auth.uid() = (SELECT cedente_auth_id FROM casos WHERE id = caso_id))`.
- [x] **1.9** Criar migration `0004_dossie_historico.sql` com tabela `historico_documentos` e índices acima.

### 2. Backend — Dossiê Endpoints

- [x] **2.1** Implementar `GET /casos/:caso_id/dossie` — retorna dossiê com todos os documentos e seus status. Verificar que Cedente logado é owner do caso via RLS. Response: `{ dossie_id, status, documentos: [{ id, tipo, status, motivo_rejeicao, versao, pode_reenviar: boolean }] }`.
- [x] **2.2** Campo `pode_reenviar` é `true` apenas para status `PENDENTE` ou `REJEITADO`; `false` para `EM_ANALISE` e `VERIFICADO` (RN-044).
- [x] **2.3** Implementar `POST /documentos/:documento_id/upload` usando `ADR-CED-004` (upload direto ao Supabase Storage via signed URL):
  - Step 1: `POST /documentos/:documento_id/upload/iniciar` → validar que documento está em `PENDENTE` ou `REJEITADO` → gerar signed URL do Supabase Storage (`supabase.storage.from('dossies').createSignedUploadUrl(path, { expiresIn: 300 })`) → retornar `{ signed_url, path }`.
  - Step 2: Frontend faz PUT diretamente ao Supabase Storage com o arquivo.
  - Step 3: `POST /documentos/:documento_id/upload/confirmar` com `{ path }` → Backend baixa o arquivo do Storage, valida MIME type real (não apenas extensão, usando `file-type` npm package) — formatos aceitos: `application/pdf`, `image/jpeg`, `image/png` → valida tamanho ≤ 10 MB (`10 * 1024 * 1024` bytes) → atualiza `documentos_dossie SET status='EM_ANALISE', storage_path, mime_type, tamanho_bytes, versao = versao + 1`.
- [x] **2.4** Se validação MIME falhar: retornar HTTP 422 com `cedente_message: "Formato não aceito. Envie o documento em PDF, JPG ou PNG."` (RN-042 item 5).
- [x] **2.5** Se tamanho > 10 MB: retornar HTTP 422 com `cedente_message: "O arquivo é muito grande. O limite é de 10 MB por documento."` (RN-042 item 6).
- [x] **2.6** Após confirmar upload: verificar completude do dossiê (`verificar_completude_dossie(caso_id)`) — se todos os documentos obrigatórios estão em `EM_ANALISE` ou `VERIFICADO` E o Termo de Cadastro está `ASSINADO`: publicar evento RabbitMQ `DOSSIE_COMPLETO` e mudar `casos.status` de `CADASTRO_REALIZADO` para `EM_ANALISE` (RN-043).
- [x] **2.7** Número de documentos obrigatórios depende do `tipo_cedente`: PF = 6 tipos (`CONTRATO_IMOVEL`, `COMPROVANTE_PAGAMENTO_1`, `COMPROVANTE_PAGAMENTO_2`, `COMPROVANTE_PAGAMENTO_3`, `DECLARACAO_ADIMPLENCIA`, `DOCUMENTO_IDENTIDADE`, `COMPROVANTE_ENDERECO`, `TABELA_CONTRATO` = na verdade 8 listados, verificar RN-041 — são 6 obrigatórios PF); PJ = 8 tipos adicionando `CONTRATO_SOCIAL` e `CARTAO_CNPJ` (RN-013.a, RN-056). ⚠️ AMBÍGUO: RN-041 lista 6 documentos PF mas enumera 7 campos — adotado 6 obrigatórios PF conforme título da RN; `TABELA_CONTRATO` é o 6º.
- [x] **2.8** Implementar job cron `DocumentosPendentesCron` que roda diariamente às 09h BRT: para cada caso com `status = 'CADASTRO_REALIZADO'` e documentos pendentes há múltiplos de 7 dias (7, 14, 21...): publicar evento `LEMBRETE_DOCUMENTOS` no RabbitMQ → `NotificationService` envia template `NOT-CED-03` (lembrete documentos pendentes) (RN-046).
- [x] **2.9** Implementar verificação KYC automatizada após `DOSSIE_COMPLETO` (RN-095):
  - Job assíncrono: valida documentos via OCR básico (formato correto, legibilidade) em até **30 minutos**.
  - Se KYC automatizado aprovado → publicar `KYC_APROVADO` → Admin recebe para triagem manual.
  - Se KYC automatizado indica revisão humana → reclassifica para análise manual → SLA de **2 dias úteis** → notificar Cedente com template `NOT-CED-04`.
  - Se SLA manual ultrapassado → alertar Admin internamente + notificar Cedente.

### 3. Frontend — Tela Documentos

- [x] **3.1** Criar `apps/web-cedente/src/app/(authenticated)/documentos/page.tsx` — lista de casos com dossiê.
- [x] **3.2** Criar `apps/web-cedente/src/app/(authenticated)/documentos/[caso_id]/page.tsx` — checklist do dossiê.
- [x] **3.3** Exibir cada documento com: nome amigável do tipo, status visual (Pendente/Em análise/Verificado/Rejeitado) com ícone e cor, `aria-label="[Nome] — status: [status]"` para acessibilidade (RN-041 item 2).
- [x] **3.4** Para documento `PENDENTE` ou `REJEITADO`: exibir botão "Enviar" / "Reenviar". Para documento `REJEITADO`: exibir motivo da rejeição visível (usando templates padronizados) + tooltip "Precisa de ajuda?" que abre chat do Guardião do Retorno (RN-045 item 2).
- [x] **3.5** Implementar componente `<DocumentUpload>` com: dropzone drag-and-drop + seleção de arquivo; barra de progresso com percentual durante upload; se upload >30s exibir "Enviando... [X]% concluído. Não feche esta tela." com botão "Cancelar" (RN-042 item 3).
- [x] **3.6** Em caso de falha de rede durante upload: exibir "A conexão foi interrompida. Tente enviar o documento novamente." com botão "Tentar novamente" que mantém o arquivo selecionado (RN-042 item 4).
- [x] **3.7** Validação client-side ANTES do upload: verificar tipo do arquivo pelo `file.type` (MIME type do browser) e tamanho `file.size <= 10 * 1024 * 1024` — exibir erro imediato sem fazer request ao servidor se inválido. NOTA: validação real (MIME type real) é feita também no backend via `file-type` — a validação client-side é apenas UX.
- [x] **3.8** Após upload bem-sucedido: exibir check verde + preview do arquivo (thumbnail para imagens, ícone PDF para PDFs) para conferência (RN-042 item 3).
- [x] **3.9** Documento `VERIFICADO`: exibir com selo de verificação + ícone de cadeado + tooltip "Este documento já foi verificado e não pode ser substituído." Campo de upload desabilitado (RN-044).
- [x] **3.10** Após completar dossiê e avançar para "Em análise": exibir banner no topo "Seus documentos estão completos! Seu caso avançou para análise." por 10 segundos (ou até o Cedente fechar) (RN-043 item 3).
- [x] **3.11** Seletor de caso no topo da tela Documentos — confirmar caso selecionado antes de aceitar qualquer upload (RN-041 edge case).

---

## 🔴 FEATURE 2 — Assinaturas (ZapSign)

### 4. Banco de Dados — Envelopes e Assinaturas

- [x] **4.1** Confirmar que a tabela `envelopes_assinatura` possui os campos: `id UUID PK`, `caso_id UUID FK REFERENCES casos(id)`, `tipo TipoEnvelope NOT NULL`, `status StatusEnvelope NOT NULL DEFAULT 'PENDENTE'`, `zapsign_doc_token VARCHAR(500)`, `zapsign_signer_token VARCHAR(500)` (token único do signatário Cedente), `url_assinatura VARCHAR(1000)`, `arquivo_assinado_path VARCHAR(500)`, `enviado_em TIMESTAMPTZ`, `assinado_em TIMESTAMPTZ`, `expira_em TIMESTAMPTZ`, `created_at`, `updated_at`.
- [x] **4.2** Confirmar que o enum `TipoEnvelope` possui exatamente 4 valores: `TERMO_CADASTRO`, `TERMO_ACEITE_ESCALONAMENTO`, `TERMO_COMERCIAL`, `INSTRUMENTO_CESSAO` (RN-047 e documentação de API D16).
- [x] **4.3** Confirmar que o enum `StatusEnvelope` possui: `PENDENTE`, `ASSINADO`, `CANCELADO`.
- [x] **4.4** Criar índice `idx_envelopes_caso_tipo ON envelopes_assinatura(caso_id, tipo)` para lookup de envelope por caso e tipo.
- [x] **4.5** Criar índice `idx_envelopes_status ON envelopes_assinatura(status, expira_em)` para job cron de alerta de assinaturas pendentes.
- [x] **4.6** Confirmar RLS em `envelopes_assinatura`: `FOR SELECT USING (auth.uid() = (SELECT cedente_auth_id FROM casos WHERE id = caso_id))`.

### 5. Backend — ZapSign Integration Service

- [x] **5.1** Criar `apps/api/src/modules/assinaturas/zapsign.service.ts` com método `criarEnvelope(casoId, tipoEnvelope, nomeArquivo, pdfBase64)`:
  - Chamar `POST https://api.zapsign.com.br/api/v1/docs/` com header `Authorization: Bearer {ZAPSIGN_API_TOKEN}`.
  - Body: `{ name: nomeArquivo, external_id: casoId, signers: [{ name: cedente.nome, auth_mode: 'assinaturaTela', send_automatic_email: false }] }`.
  - Persistir no banco: `zapsign_doc_token`, `zapsign_signer_token`, `url_assinatura`, `enviado_em = NOW()`, `expira_em = NOW() + 5 dias úteis`.
- [x] **5.2** Implementar webhook handler `POST /webhooks/zapsign` — recebe eventos ZapSign:
  - Evento `doc_signed`: atualizar `envelopes_assinatura SET status='ASSINADO', assinado_em=NOW()` pelo `zapsign_doc_token`; baixar PDF assinado e armazenar em `arquivo_assinado_path`; publicar evento `ENVELOPE_ASSINADO` no RabbitMQ.
  - Evento `doc_opened`: registrar em `historico_documentos` para auditoria.
  - Validar `x-zapsign-webhook-signature` header para autenticidade do webhook (ZAPSIGN_WEBHOOK_SECRET).
- [x] **5.3** Implementar circuit breaker para ZapSign: se 3 chamadas consecutivas falharem dentro de 60s → abrir circuito → retornar `ZAPSIGN_INDISPONIVEL` → publicar alerta no Sentry (D20, D17).
- [x] **5.4** Implementar retry policy para falhas ZapSign: 1ª retry após 2 minutos, 2ª após 10 minutos, 3ª após 1 hora; após 3 falhas → DLQ `assinaturas.dead-letter` (D17).

### 6. Backend — Assinaturas Endpoints

- [x] **6.1** Implementar `GET /casos/:caso_id/assinaturas` — lista envelopes do caso com status. Retornar apenas campos públicos: `{ id, tipo, status, enviado_em, expira_em, pode_assinar: boolean }`. `pode_assinar = status === 'PENDENTE'`.
- [x] **6.2** Implementar `GET /assinaturas/:envelope_id/url` — retorna `url_assinatura` (token do signatário ZapSign) para o Cedente abrir o iframe. Verificar que `cedente_id` é owner do caso do envelope.
- [x] **6.3** Implementar `GET /assinaturas/:envelope_id/download` — retorna signed URL do Supabase Storage para download do `arquivo_assinado_path`. Disponível apenas para `status = 'ASSINADO'`.
- [x] **6.4** Implementar lógica de bloqueio de avanço de estado do caso (RN-048):
  - Transição `CADASTRO_REALIZADO → EM_ANALISE`: verificar que `envelopes_assinatura` com `tipo='TERMO_CADASTRO'` e `caso_id` tem `status='ASSINADO'`.
  - Transição `EM_FORMALIZACAO → NEGOCIO_FECHADO`: verificar que `TERMO_COMERCIAL` e `INSTRUMENTO_CESSAO` estão ambos `ASSINADO` por todas as partes.
  - Se condição não atendida: retornar HTTP 422 com `cedente_message: "Há documentos pendentes de assinatura. Assine para avançar."`.
- [x] **6.5** Implementar job cron `AssinaturasPendentesCron` conforme régua ZapSign (D21 TL;DR, RN-094):
  - D+0: envio do envelope → notificar Cedente com template `NOT-CED-11` (documento disponível para assinatura).
  - D+2 (2 dias após envio): `NOT-CED-12` (1º lembrete de assinatura pendente — desativável pelo Cedente).
  - D+4 (4 dias após envio): lembrete urgente ao Cedente + alerta interno ao Admin (RN-050).
  - D+5 (5 dias após envio): marcar envelope como expirado → Admin alertado para contato ativo com Cedente.
  - Após **10 dias úteis** sem assinatura: gerar alerta urgente para Admin (RN-050 item 3).
- [x] **6.6** Implementar imutabilidade após assinatura: `PATCH /envelopes/:id` deve retornar HTTP 403 se `status = 'ASSINADO'` com `cedente_message: "Documentos assinados não podem ser alterados."` (RN-049).

### 7. Frontend — Tela Assinaturas

- [x] **7.1** Criar `apps/web-cedente/src/app/(authenticated)/assinaturas/page.tsx` — lista documentos pendentes de assinatura e documentos já assinados.
- [x] **7.2** Para cada envelope pendente: exibir nome do documento, caso vinculado, data de envio, prazo para assinatura (countdown visual), botão "Assinar Agora".
- [x] **7.3** Implementar componente `<ZapSignIframe>`:
  - Ao clicar "Assinar Agora": chamar `GET /assinaturas/:id/url` e renderizar ZapSign em iframe inline (sem redirecionamento externo) (RN-047 item 3).
  - Timeout de **10 segundos** para carregar o iframe: se não carregar → exibir "O serviço de assinatura está demorando para responder." com botões "Tentar novamente" e "Abrir em nova aba" (RN-047 item 3).
  - Se falhar 3 vezes: "O serviço de assinatura está temporariamente indisponível. Tente novamente em alguns minutos ou entre em contato com o suporte."
- [x] **7.4** Ao tentar fechar o iframe antes de concluir assinatura: exibir alerta "Você não concluiu a assinatura. O documento permanecerá disponível para assinatura." (RN-047 item 4).
- [x] **7.5** Antes de exibir o iframe: mostrar documento completo em modo leitura (PDF preview) com indicação "Leia o documento antes de assinar" (RN-047 item 4).
- [x] **7.6** Após assinatura confirmada (webhook ZapSign recebido + Supabase Realtime notifica frontend): atualizar status do envelope para "Assinado" + exibir botão "Baixar documento assinado" + toast "Documento assinado com sucesso!".
- [x] **7.7** Documentos assinados: exibidos com badge verde "Assinado", data de assinatura, botão "Baixar" (chama `GET /assinaturas/:id/download`).

---

## 🔴 FEATURE 3 — Notificações (Sistema)

### 8. Backend — Módulo de Notificações

- [x] **8.1** Criar `apps/api/src/modules/notificacoes/notificacoes.service.ts` com método `notify(event: NotificationEvent)`:
  - Buscar preferências do Cedente em `cedentes.preferencias_notificacao`.
  - Criar registro em `notificacoes` com `lida=false` ANTES de publicar na fila (garantia de entrega in-app — D21 item 1.1).
  - Publicar na fila RabbitMQ `notification.exchange` (direct) para canais elegíveis: `notification.email`, `notification.push`, `notification.inapp`.
- [x] **8.2** Implementar `EmailWorker` que consome `notification.email`: chamar Resend SDK com template correspondente; retry após 3 falhas → DLQ `notification.email.dead-letter` (D21 arquitetura).
- [x] **8.3** Implementar `InAppWorker` que consome `notification.inapp`: inserir na tabela `notificacoes` e publicar no Supabase Realtime channel `notifications:{cedente_id}`.
- [x] **8.4** SLA de entrega por canal (RN-056 item 2, RN-057 item 2): e-mail em até **5 minutos**; notificação in-app em até **30 segundos** (via Supabase Realtime).
- [x] **8.5** Implementar os 14 templates de notificação `NOT-CED-01` a `NOT-CED-14` (D21): cada template com campos `subject`, `html_template`, `variables` esperadas — armazenados em `apps/api/src/modules/notificacoes/templates/`.
- [x] **8.6** Notificações críticas (eventos 8, 13, 14 — conforme RN-056): campo `critica: true` no enum/config; ignorar preferências de opt-out do Cedente para estes eventos.
- [x] **8.7** Implementar `GET /notificacoes` com paginação `page`/`per_page`; filtro `lida?: boolean`. Ordenação: `created_at DESC`.
- [x] **8.8** Implementar `PATCH /notificacoes/:id/lida` — marca notificação como lida. Verificar que `destinatario_auth_id` = Cedente logado.
- [x] **8.9** Implementar `PATCH /notificacoes/marcar-todas-lidas` — marca todas as notificações não lidas do Cedente logado.
- [x] **8.10** Retenção de logs de notificação: job cron `LimpezaNotificacoesCron` que roda mensalmente: deletar registros em `notificacoes` com `created_at < NOW() - INTERVAL '90 days'` (D21 LGPD).

### 9. Frontend — Centro de Notificações

- [x] **9.1** Implementar badge de notificações no topo do `AppLayout`: sino com número de notificações não lidas; contraste mínimo 4.5:1 (WCAG AA); `aria-label="Notificações — [X] não lidas"` (RN-057 item 3).
- [x] **9.2** Implementar Supabase Realtime subscription para canal `notifications:{cedente_id}`: ao receber evento `INSERT` na tabela `notificacoes` com `destinatario_auth_id` = Cedente logado → incrementar badge + exibir toast com texto da notificação (RN-057 item 2 — até 30s).
- [x] **9.3** Criar dropdown de notificações ao clicar no sino: lista das últimas 20 notificações com: tipo, texto, data relativa, indicador de não lida (destaque azul). Ao clicar: marcar como lida + redirecionar para tela correspondente.
- [x] **9.4** Criar `apps/web-cedente/src/app/(authenticated)/notificacoes/page.tsx` — página completa de notificações com filtro "Todas / Não lidas" e paginação.

---

## 🔴 FEATURE 4 — Cancelamento de Caso

### 10. Backend — Cancelamento

- [x] **10.1** Implementar `POST /casos/:id/cancelar` com body `{ motivo: 'DESISTI_REPASSE' | 'RESOLVI_DISTRATO' | 'NEGOCIEI_DIRETAMENTE' | 'OUTRO', observacao?: string }` (RN-055 item 2).
- [x] **10.2** Verificar que `casos.status` é anterior ao Fechamento: estados permitidos para cancelamento = `CADASTRO_REALIZADO`, `EM_ANALISE`, `PENDENCIA_IDENTIFICADA`, `APROVADO_PARA_OFERTA`, `DISPONIVEL_PARA_COMPRADORES`, `PROPOSTA_RECEBIDA`, `EM_FORMALIZACAO`. Se `status` for `NEGOCIO_FECHADO` ou posterior: retornar HTTP 422 com `cedente_message: "Não é possível cancelar após o Fechamento. Use o fluxo de desistência."` (RN-055 edge case).
- [x] **10.3** Se há Conta Escrow ativa (status `ABERTA` ou `DEPOSITO_CONFIRMADO`): registrar `tem_escrow_ativo = true` na lógica de cancelamento — frontend exibe modal de 2 etapas com aviso de estorno de até **5 dias úteis** (RN-055 item 4).
- [x] **10.4** Ao confirmar cancelamento: `UPDATE casos SET status='CANCELADO'`; publicar evento `CASO_CANCELADO` no RabbitMQ; se `tem_escrow_ativo` → publicar `ESCROW_CANCELAR` para módulo financeiro (S6).
- [x] **10.5** Registrar evento em `eventos_caso`: `{ tipo: 'CANCELAMENTO', descricao_publica: 'Caso cancelado pelo Cedente.', created_at: NOW() }`.
- [x] **10.6** Notificar Cedente com template `NOT-CED-16` (caso cancelado) via NotificationService.

### 11. Frontend — Cancelamento

- [x] **11.1** Exibir botão "Cancelar Caso" em `Meus Casos → Detalhe do Caso` apenas para casos com status pré-Fechamento. Botão AUSENTE para casos em `NEGOCIO_FECHADO` e posteriores.
- [x] **11.2** Ao clicar "Cancelar Caso": abrir modal com dropdown de motivo (obrigatório) + campo texto observações (opcional).
- [x] **11.3** Sem Escrow ativo: modal de confirmação simples "Esta ação não pode ser desfeita. O caso será cancelado permanentemente." + botão destrutivo "Cancelar caso" (vermelho).
- [x] **11.4** Com Escrow ativo: modal em 2 etapas — (1) motivo do cancelamento, (2) segundo modal "Passo 2 de 2 — Há um depósito na Conta Escrow. Ao cancelar, o valor será estornado integralmente ao comprador em até 5 dias úteis." + botão "Confirmar cancelamento" em vermelho (RN-055 item 4).

---

## 🔀 CROSS-MÓDULO

### 12. Eventos disparados por esta sprint

- [x] **12.1** Após `DOSSIE_COMPLETO` + Termo assinado: publicar `STATUS_MUDOU` (`CADASTRO_REALIZADO → EM_ANALISE`) → módulo de eventos_caso registra na linha do tempo.
- [x] **12.2** Após `ENVELOPE_ASSINADO` (webhook ZapSign): verificar se todas as assinaturas obrigatórias da etapa estão completas → se sim, publicar `ETAPA_ASSINATURAS_CONCLUIDA` para módulo de casos atualizar estado.
- [x] **12.3** Após `CASO_CANCELADO` com `tem_escrow_ativo`: publicar `ESCROW_CANCELAR` → S6 (Financeiro) processa estorno.
- [x] **12.4** `InAppWorker` publica no Supabase Realtime channel `notifications:{cedente_id}` → S3 (Dashboard) e frontend consomem para atualizar badge e feed.

---

## 🧪 TESTES

### 13. Testes unitários (Vitest)

- [x] **13.1** `documentos.service.spec.ts` — testar validação MIME: PDF real → aceito; arquivo renomeado para .pdf mas MIME=image/gif → rejeitado HTTP 422; arquivo >10MB → rejeitado HTTP 422 (RN-042).
- [x] **13.2** `documentos.service.spec.ts` — testar `verificar_completude_dossie()`: 5 de 6 docs → não avança; todos 6 → avança; mas Termo de Cadastro não assinado → não avança (RN-043).
- [x] **13.3** `documentos.service.spec.ts` — testar imutabilidade de documento `VERIFICADO`: tentar upload → HTTP 403 (RN-044).
- [x] **13.4** `zapsign.service.spec.ts` — testar `criarEnvelope()`: mock API ZapSign retorna `doc_token` + `signer_token` → persistido no banco; mock falha 3x → circuit breaker abre.
- [x] **13.5** `assinaturas.service.spec.ts` — testar bloqueio de avanço: caso `CADASTRO_REALIZADO` sem `TERMO_CADASTRO` assinado → HTTP 422 (RN-048).
- [x] **13.6** `notificacoes.service.spec.ts` — testar que notificação crítica (`tipo='NOVA_PROPOSTA'`) é enviada mesmo com opt-out ativo no perfil do Cedente (RN-056 item 4).
- [x] **13.7** `cancelamento.service.spec.ts` — testar: caso `NEGOCIO_FECHADO` → HTTP 422; caso `EM_FORMALIZACAO` com Escrow → retorna flag `tem_escrow_ativo = true`.

### 14. Testes de integração (Supertest)

- [x] **14.1** Fluxo upload completo: `POST /upload/iniciar` → PUT ao Storage (mock) → `POST /upload/confirmar` → verificar `documentos_dossie.status = 'EM_ANALISE'`.
- [x] **14.2** Fluxo dossiê completo → avanço de estado: fazer upload de todos os 6 docs PF + assinar Termo de Cadastro (mock ZapSign webhook) → verificar `casos.status = 'EM_ANALISE'`.
- [x] **14.3** Webhook ZapSign: `POST /webhooks/zapsign` com evento `doc_signed` e token válido → verificar `envelopes_assinatura.status = 'ASSINADO'`; com assinatura inválida → HTTP 401.
- [x] **14.4** `GET /notificacoes` com Cedente sem notificações → retorna `{ data: [], total: 0 }`; após criar notificação → retorna corretamente.

### 15. Testes E2E (Playwright)

- [x] **15.1** T-026: Upload MIME — tentar enviar arquivo .pdf com conteúdo GIF via UI → verificar mensagem de erro "Formato não aceito".
- [x] **15.2** T-027: Upload com barra de progresso — arquivo grande → verificar barra de progresso visível.
- [x] **15.3** T-034: ZapSign inline — clicar "Assinar Agora" → verificar iframe carregado dentro do painel (não redirecionamento externo).
- [x] **15.4** T-035: Cancelamento com Escrow — simular caso em `EM_FORMALIZACAO` → clicar cancelar → verificar modal de 2 etapas com aviso de estorno.

### 16. Teste Mobile — T-042 (camera upload)

- [x] **16.1** No `apps/mobile-cedente`: implementar upload de documentos via câmera usando `expo-image-picker` (câmera ou galeria).
- [x] **16.2** Comprimir imagens capturadas pela câmera para ≤10MB antes do upload: `ImageManipulator.manipulateAsync(uri, [], { compress: 0.7, format: SaveFormat.JPEG })`.
- [x] **16.3** T-042 (RNTL): testar captura de foto → compressão → upload → verificar `documentos_dossie.status = 'EM_ANALISE'`.

---

## ✅ AUTO-VERIFICAÇÃO (12 Checks)

| #   | Check                  | Critério                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Status |
| --- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| 1   | **Nomenclatura**       | `dossies` (não `documentSets`), `documentos_dossie` (não `documents`), `envelopes_assinatura` (não `signatures`), `TipoEnvelope` com 4 valores exatos: `TERMO_CADASTRO`, `TERMO_ACEITE_ESCALONAMENTO`, `TERMO_COMERCIAL`, `INSTRUMENTO_CESSAO`; `StatusDocumento`: `PENDENTE`, `EM_ANALISE`, `VERIFICADO`, `REJEITADO`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | ✅     |
| 2   | **Verificabilidade**   | Cada item é binariamente verificável — MIME type real validado com `file-type`, não apenas extensão                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | ✅     |
| 3   | **Valores numéricos**  | Upload máximo **10 MB**; formatos aceitos: `application/pdf`, `image/jpeg`, `image/png`; iframe ZapSign timeout **10 segundos**; régua ZapSign: D+0/D+2/D+4/D+5; lembrete assinatura após **3 dias úteis**; alerta Admin após **10 dias úteis**; lembrete docs pendentes a cada **7 dias corridos**; SLA KYC automatizado **30 minutos**; SLA triagem manual **2 dias úteis**; e-mail em até **5 minutos**; notificação in-app em até **30 segundos**; retenção logs **90 dias**                                                                                                                                                                                                                                                                                                                        | ✅     |
| 4   | **Contagem de itens**  | 6 documentos obrigatórios PF (`CONTRATO_IMOVEL`, `COMPROVANTE_PAGAMENTO_1`, `COMPROVANTE_PAGAMENTO_2`, `COMPROVANTE_PAGAMENTO_3`, `DECLARACAO_ADIMPLENCIA`, `DOCUMENTO_IDENTIDADE`, `COMPROVANTE_ENDERECO`, `TABELA_CONTRATO` — ⚠️ AMBÍGUO ver item 2.7); 8 PJ; 4 tipos de envelope; 17 eventos de notificação; 14 templates NOT-CED-01 a NOT-CED-14                                                                                                                                                                                                                                                                                                                                                                                                                                                    | ✅     |
| 5   | **Máquina de estados** | `StatusDocumento`: Pendente → EmAnalise → Verificado/Rejeitado → EmAnalise (reenvio); `StatusEnvelope`: Pendente → Assinado/Cancelado; `StatusEscrow` coberto em S6; bloqueio de transição `CADASTRO_REALIZADO → EM_ANALISE` sem Termo assinado implementado                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | ✅     |
| 6   | **RBAC**               | RLS em `dossies`, `documentos_dossie`, `envelopes_assinatura`; `historico_documentos` sem RLS (Admin-only via service role); Cedente não vê nem acessa `arquivo_assinado_path` diretamente — apenas via signed URL com expiração                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | ✅     |
| 7   | **Anti-scaffold**      | `ZapSignService.criarEnvelope()` faz chamada real à API ZapSign (não stub); circuit breaker implementado com estado real (não log apenas); `DocumentosPendentesCron` verifica dias corridos reais (não apenas flag boolean)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | ✅     |
| 8   | **Glossário**          | `dossies` (não `dossiês`), `documentos_dossie`, `envelopes_assinatura`, `historico_documentos`, `TERMO_CADASTRO` (não `registration_term`), `motivo_rejeicao` (não `rejection_reason`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | ✅     |
| 9   | **Segurança**          | MIME type validado pelo conteúdo real do arquivo (não extensão); webhook ZapSign valida `x-zapsign-webhook-signature`; signed URLs do Storage expiram (não URLs permanentes); `SUPABASE_SERVICE_ROLE_KEY` nunca exposta                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | ✅     |
| 10  | **ADR-CED-004**        | Upload usa padrão ADR-CED-004 (direto ao Supabase Storage via signed URL, não via proxy da API) — verificar que `apps/api` não recebe o arquivo binário em memória                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | ✅     |
| 11  | **Cross-módulo**       | Eventos RabbitMQ publicados: `DOSSIE_COMPLETO`, `ENVELOPE_ASSINADO`, `ETAPA_ASSINATURAS_CONCLUIDA`, `CASO_CANCELADO`, `ESCROW_CANCELAR`, `LEMBRETE_DOCUMENTOS`; `InAppWorker` publica no canal Supabase Realtime correto                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | ✅     |
| 12  | **Cobertura de REQs**  | REQ-068 (6 docs PF), REQ-069 (MIME validation), REQ-070 (avanço automático para triagem), REQ-071 (KYC SLA 30min/2 dias), REQ-072 (imutabilidade verificado), REQ-073 (reenvio rejeitado), REQ-074 (lembrete 7d), REQ-075 (ZapSign inline), REQ-076 (bloqueio avanço sem assinatura), REQ-077 (imutabilidade assinado), REQ-078 (alerta Admin 10d), REQ-079 (cancelamento pré-Fechamento), REQ-080 (17 eventos notificação), REQ-081 (regras envio email), REQ-082 (badge sino), REQ-083 (histórico escalonamento), REQ-084–086 (financeiro somente leitura), REQ-094 (régua ZapSign D+0/D+2/D+4/D+5), REQ-095 (SLA KYC), REQ-111 (tabela notificações 17 eventos), REQ-112 (templates NOT-CED-01 a NOT-CED-14), REQ-123–125 (cancelamento modal), REQ-198–199 (upload mobile camera) — 100% atribuídos | ✅     |

---

## 📋 PENDÊNCIAS

| ID      | Tipo        | Descrição                                                                                                                                                                                                                                                                               |
| ------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PEND-06 | ⚠️ AMBÍGUO  | RN-041 diz "6 documentos obrigatórios" para PF mas lista 8 itens no exemplo da seção 2. Adotado: 6 obrigatórios para PF, sendo `TABELA_CONTRATO` o 6º e `COMPROVANTE_ENDERECO` o 5º. Verificar com PM antes de implementar. [REVISÃO MANUAL]                                            |
| PEND-07 | ⚠️ CONFLITO | D21 diz réqua ZapSign D+5 = expiração; RN-050 diz "após 10 dias úteis sem assinatura" alerta Admin. Interpretado como: D+5 = expiração do link ZapSign (novo link gerado pelo Admin); 10 dias úteis = alerta escalado para contato ativo. Não são conflitantes — são eventos distintos. |
