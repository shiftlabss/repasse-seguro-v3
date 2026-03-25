# 13 - Schema Prisma

## Módulo Cedente · Plataforma Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Destinatário** | Engenharia Backend |
| **Escopo** | Schema Prisma completo · Enums · Models · Relações · Índices |
| **Módulo** | Cedente |
| **Versão** | v1.0 |
| **Responsável** | Claude Code Desktop |
| **Data** | 2026-03-23 (America/Fortaleza) |

---

> **TL;DR**
>
> - Schema Prisma completo para o módulo Cedente, derivado do ERD (D12).
> - Stack: Prisma 6.x + PostgreSQL 17 (via Supabase). Datasource: `postgresql`.
> - Convenções: UUID v4, snake_case para tabelas e colunas via `@map`/`@@map`, PascalCase para models, UPPER_SNAKE_CASE para valores de enum.
> - Soft delete via `deletedAt DateTime? @map("deleted_at") @db.Timestamptz` nas tabelas de domínio principal.
> - Extensão `pgcrypto` para geração de UUID. Sem `pgvector` neste módulo (RAG do Guardião usa base de conhecimento de produto, não embeddings de entidades do Cedente).
> - 11 models mapeados: User, Cedente, Caso, Dossie, DocumentoDossie, Proposta, EnvelopeAssinatura, ContaEscrow, Notificacao, EventoCaso, AiSession.
> - Enums: TipoCedente, StatusConta, StatusCaso, StatusCasoInterno, CenarioRetorno, StatusDossie, TipoDocumento, StatusDocumento, TipoProposta, StatusProposta, TipoEnvelope, StatusEnvelope, StatusEscrow, TipoNotificacao, TipoAtor, TipoEvento.

---

## 1. Schema Prisma Completo

```prisma
// schema.prisma — cedente
// Módulo Cedente — Repasse Seguro
// Prisma 6.x + PostgreSQL 17 via Supabase
// Gerado por: Claude Code Desktop — Pipeline ShiftLabs v9.5
// Data: 2026-03-23

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  directUrl  = env("DIRECT_URL")
  extensions = [pgcrypto]
}

// ============================================================
// ENUMS
// ============================================================

/// Tipo de pessoa do Cedente (RN-001)
enum TipoCedente {
  PF
  PJ
}

/// Estado da conta do Cedente (RN-001 a RN-010)
enum StatusConta {
  PENDENTE_ATIVACAO
  ATIVA
  BLOQUEADA
  ENCERRADA
}

/// Status visível ao Cedente — 13 estados (D01.1, seção 6)
enum StatusCaso {
  CADASTRO_REALIZADO
  EM_ANALISE
  PENDENCIA_IDENTIFICADA
  APROVADO_PARA_OFERTA
  DISPONIVEL_PARA_COMPRADORES
  PROPOSTA_RECEBIDA
  EM_FORMALIZACAO
  NEGOCIO_FECHADO
  AGUARDANDO_LIBERACAO
  DESISTENCIA_EM_ANALISE
  MEDIACAO_EM_ANDAMENTO
  CONCLUIDO
  CANCELADO
}

/// Status interno do Admin — espelha a visão operacional
enum StatusCasoInterno {
  CAPTADO
  EM_TRIAGEM
  BLOQUEADO
  QUALIFICADO
  OFERTA_ATIVA
  EM_NEGOCIACAO
  EM_FORMALIZACAO
  FECHAMENTO
  POS_FECHAMENTO
  EM_REVERSAO
  EM_MEDIACAO
  CONCLUIDO
  CANCELADO
}

/// Cenário de retorno escolhido pelo Cedente (RN-022, D01.2 seção 2)
enum CenarioRetorno {
  A
  B
  C
  D
}

/// Status agregado do dossiê
enum StatusDossie {
  INCOMPLETO
  COMPLETO
  EM_ANALISE
}

/// Tipo de documento obrigatório do dossiê (RN-041, RN-070)
enum TipoDocumento {
  CONTRATO_ORIGINAL
  COMPROVANTE_PAGAMENTO_1
  COMPROVANTE_PAGAMENTO_2
  COMPROVANTE_PAGAMENTO_3
  DECLARACAO_ADIMPLENCIA
  DOCUMENTO_IDENTIDADE
  COMPROVANTE_ENDERECO
  TABELA_CONTRATO
  CONTRATO_SOCIAL     // Somente PJ (RN-070)
  PROCURACAO          // Somente PJ (RN-070)
}

/// Status individual de cada documento do dossiê (RN-041 a RN-045)
enum StatusDocumento {
  PENDENTE
  EM_ANALISE
  VERIFICADO
  REJEITADO
}

/// Status das propostas recebidas (RN-030 a RN-033)
enum StatusProposta {
  RECEBIDA
  VISUALIZADA
  EM_NEGOCIACAO
  ACEITA
  RECUSADA
  CONTRAPROPOSTA_ENVIADA
  EXPIRADA
  SUPERADA
  CANCELADA
}

/// Tipo de documento de assinatura (RN-024, RN-025, RN-047, RN-048)
enum TipoEnvelope {
  TERMO_CADASTRO
  TERMO_ACEITE_ESCALONAMENTO
  TERMO_COMERCIAL
  INSTRUMENTO_CESSAO
}

/// Status do envelope de assinatura
enum StatusEnvelope {
  PENDENTE
  ASSINADO_CEDENTE
  ASSINADO_ADMIN
  CONCLUIDO
  CANCELADO
}

/// Status da conta escrow (RN-083, D01.3 seção 4)
enum StatusEscrow {
  ABERTA
  DEPOSITO_CONFIRMADO
  EM_PERIODO_REVERSAO
  VALORES_DISTRIBUIDOS
  ESTORNADA
}

/// Tipos de notificação para o Cedente (RN-056)
enum TipoNotificacao {
  CONTA_ATIVADA
  CASO_EM_ANALISE
  PENDENCIA_IDENTIFICADA
  APROVADO_OFERTA
  PROPOSTA_RECEBIDA
  PROPOSTA_EXPIRADA
  PROPOSTA_ACEITA
  EM_FORMALIZACAO
  ASSINATURA_PENDENTE
  NEGOCIO_FECHADO
  AGUARDANDO_LIBERACAO
  CONCLUIDO
  CANCELADO
  DOCUMENTO_REJEITADO
  LEMBRETE_DOCUMENTO
  ESCALONAMENTO_CONCLUIDO
  ESTORNO_PROCESSADO
}

/// Tipo de ator responsável pelo evento
enum TipoAtor {
  CEDENTE
  ADMIN
  SISTEMA
}

/// Tipo de evento no log imutável do caso
enum TipoEvento {
  STATUS_MUDADO
  CENARIO_ALTERADO
  DOCUMENTO_ENVIADO
  DOCUMENTO_VERIFICADO
  DOCUMENTO_REJEITADO
  PROPOSTA_RECEBIDA
  PROPOSTA_ACEITA
  PROPOSTA_RECUSADA
  PROPOSTA_EXPIRADA
  CONTRAPROPOSTA_ENVIADA
  ESCALONAMENTO_SOLICITADO
  ESCALONAMENTO_CONCLUIDO
  ENVELOPE_GERADO
  ENVELOPE_ASSINADO
  ESCROW_ABERTA
  DEPOSITO_CONFIRMADO
  FECHAMENTO_CONFIRMADO
  DISTRIBUICAO_REALIZADA
  DESISTENCIA_SOLICITADA
  MEDIACAO_INICIADA
  CASO_CANCELADO
  CASO_CONCLUIDO
}

/// Provedor de autenticação
enum AuthProvider {
  EMAIL
}

// ============================================================
// MODELS
// ============================================================

/// Tabela base gerenciada pelo Supabase Auth
model User {
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  email           String    @unique @db.VarChar(255)
  name            String    @db.VarChar(255)
  provider        AuthProvider @default(EMAIL) @map("provider")
  emailVerifiedAt DateTime? @map("email_verified_at") @db.Timestamptz
  createdAt       DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt       DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt       DateTime? @map("deleted_at") @db.Timestamptz

  // Relações
  cedente         Cedente?
  documentosVerificados DocumentoDossie[] @relation("VerificadoPor")
  eventosComoAtor EventoCaso[] @relation("AtorEvento")

  @@map("users")
}

/// Perfil estendido do Cedente — PF (CPF) ou PJ (CNPJ)
model Cedente {
  id                      String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  userId                  String    @unique @map("user_id") @db.Uuid
  tipo                    TipoCedente
  cpfCnpj                 String    @unique @map("cpf_cnpj") @db.VarChar(18)
  telefone                String?   @db.VarChar(20)
  telefoneVerificadoEm    DateTime? @map("telefone_verificado_em") @db.Timestamptz
  statusConta             StatusConta @default(PENDENTE_ATIVACAO) @map("status_conta")
  bloqueadaAte            DateTime? @map("bloqueada_ate") @db.Timestamptz
  tentativasLogin         Int       @default(0) @map("tentativas_login")
  representanteLegalNome  String?   @map("representante_legal_nome") @db.VarChar(255)
  representanteLegalCpf   String?   @map("representante_legal_cpf") @db.VarChar(11)
  lgpdConsent             Boolean   @default(false) @map("lgpd_consent")
  lgpdConsentAt           DateTime? @map("lgpd_consent_at") @db.Timestamptz
  aiConsent               Boolean   @default(true) @map("ai_consent")
  aiConsentAt             DateTime? @map("ai_consent_at") @db.Timestamptz
  notificationPreferences Json      @default("{\"email\": true, \"push\": true, \"sms\": false}") @map("notification_preferences")
  createdAt               DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt               DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  user            User      @relation(fields: [userId], references: [id])
  casos           Caso[]
  propostas       Proposta[]
  envelopes       EnvelopeAssinatura[]
  contasEscrow    ContaEscrow[]
  notificacoes    Notificacao[]
  eventosCaso     EventoCaso[] @relation("CedenteDoCaso")
  aiSessions      AiSession[]

  @@index([userId], name: "idx_cedentes_user_id")
  @@index([cpfCnpj], name: "idx_cedentes_cpf_cnpj")
  @@index([statusConta], name: "idx_cedentes_status_conta")
  @@map("cedentes")
}

/// Entidade central — cada imóvel cadastrado pelo Cedente
model Caso {
  id                      String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  cedenteId               String    @map("cedente_id") @db.Uuid
  codigo                  String    @unique @db.VarChar(20)
  status                  StatusCaso @default(CADASTRO_REALIZADO)
  statusInterno           StatusCasoInterno @default(CAPTADO) @map("status_interno")
  cenario                 CenarioRetorno?
  cenarioAtivoDesde       DateTime? @map("cenario_ativo_desde") @db.Timestamptz
  cidade                  String    @db.VarChar(100)
  bairro                  String    @db.VarChar(100)
  estado                  String    @db.VarChar(2)
  nomeEmpreendimento      String    @map("nome_empreendimento") @db.VarChar(200)
  enderecoCompleto        String    @map("endereco_completo") @db.VarChar(500)
  quartos                 Int
  areaM2                  Decimal   @map("area_m2") @db.Decimal(8, 2)
  temGaragem              Boolean   @default(false) @map("tem_garagem")
  construtora             String?   @db.VarChar(200)
  cessaoLivre             Boolean   @default(false) @map("cessao_livre")
  valorPagoCedente        Decimal?  @map("valor_pago_cedente") @db.Decimal(15, 2)
  valorTabelaContrato     Decimal?  @map("valor_tabela_contrato") @db.Decimal(15, 2)
  valorTabelaAtual        Decimal?  @map("valor_tabela_atual") @db.Decimal(15, 2)
  valorDistratoReferencia Decimal?  @map("valor_distrato_referencia") @db.Decimal(15, 2)
  valorRecuperado         Decimal?  @map("valor_recuperado") @db.Decimal(15, 2)
  comissaoRs              Decimal?  @map("comissao_rs") @db.Decimal(15, 2)
  valorLiquidoCedente     Decimal?  @map("valor_liquido_cedente") @db.Decimal(15, 2)
  simuladorVisualizado    Boolean   @default(false) @map("simulador_visualizado")
  simuladorVisualizadoEm  DateTime? @map("simulador_visualizado_em") @db.Timestamptz
  rascunho                Boolean   @default(true)
  rascunhoExpiraEm        DateTime? @map("rascunho_expira_em") @db.Timestamptz
  ultimoEscalonamentoAt   DateTime? @map("ultimo_escalonamento_at") @db.Timestamptz
  escalonamentoEnfileirado Boolean  @default(false) @map("escalonamento_enfileirado")
  dataCadastro            DateTime? @map("data_cadastro") @db.Timestamptz
  dataAprovacao           DateTime? @map("data_aprovacao") @db.Timestamptz
  dataFechamento          DateTime? @map("data_fechamento") @db.Timestamptz
  distribuicaoPrevistaEm  DateTime? @map("distribuicao_prevista_em") @db.Timestamptz
  dataDistribuicao        DateTime? @map("data_distribuicao") @db.Timestamptz
  desistenciaSolicitadaEm DateTime? @map("desistencia_solicitada_em") @db.Timestamptz
  motivoCancelamento      String?   @map("motivo_cancelamento")
  createdAt               DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt               DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt               DateTime? @map("deleted_at") @db.Timestamptz

  // Relações
  cedente         Cedente   @relation(fields: [cedenteId], references: [id])
  dossie          Dossie?
  propostas       Proposta[]
  envelopes       EnvelopeAssinatura[]
  contaEscrow     ContaEscrow?
  notificacoes    Notificacao[]
  eventos         EventoCaso[]
  aiSessions      AiSession[]

  @@index([cedenteId], name: "idx_casos_cedente_id")
  @@index([status], name: "idx_casos_status")
  @@index([codigo], name: "idx_casos_codigo")
  @@index([enderecoCompleto], name: "idx_casos_endereco_completo")
  @@index([statusInterno], name: "idx_casos_status_interno")
  @@map("casos")
}

/// Dossiê de documentos vinculado ao Caso (criado automaticamente)
model Dossie {
  id                    String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  casoId                String    @unique @map("caso_id") @db.Uuid
  status                StatusDossie @default(INCOMPLETO)
  totalDocumentos       Int       @map("total_documentos")
  documentosVerificados Int       @default(0) @map("documentos_verificados")
  documentosPendentes   Int       @map("documentos_pendentes")
  documentosEmAnalise   Int       @default(0) @map("documentos_em_analise")
  documentosRejeitados  Int       @default(0) @map("documentos_rejeitados")
  completo              Boolean   @default(false)
  completoEm            DateTime? @map("completo_em") @db.Timestamptz
  createdAt             DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt             DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  caso          Caso      @relation(fields: [casoId], references: [id])
  documentos    DocumentoDossie[]

  @@index([casoId], name: "idx_dossies_caso_id")
  @@index([status], name: "idx_dossies_status")
  @@map("dossies")
}

/// Documento individual do dossiê — ciclo: Pendente → Em Análise → Verificado | Rejeitado
model DocumentoDossie {
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  dossieId        String    @map("dossie_id") @db.Uuid
  casoId          String    @map("caso_id") @db.Uuid
  tipo            TipoDocumento
  nome            String    @db.VarChar(255)
  status          StatusDocumento @default(PENDENTE)
  urlArquivo      String?   @map("url_arquivo") @db.VarChar(1000)
  storagePath     String?   @map("storage_path") @db.VarChar(500)
  mimeType        String?   @map("mime_type") @db.VarChar(100)
  tamanhoByte     Int?      @map("tamanho_bytes")
  motivoRejeicao  String?   @map("motivo_rejeicao")
  verificadoPor   String?   @map("verificado_por") @db.Uuid
  verificadoEm    DateTime? @map("verificado_em") @db.Timestamptz
  tentativas      Int       @default(0)
  imutavel        Boolean   @default(false)
  createdAt       DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt       DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  dossie          Dossie    @relation(fields: [dossieId], references: [id])
  verificador     User?     @relation("VerificadoPor", fields: [verificadoPor], references: [id])

  @@index([dossieId], name: "idx_documentos_dossie_dossie_id")
  @@index([casoId], name: "idx_documentos_dossie_caso_id")
  @@index([status], name: "idx_documentos_dossie_status")
  @@index([tipo], name: "idx_documentos_dossie_tipo")
  @@map("documentos_dossie")
}

/// Proposta recebida de Cessionário (mediada pelo Admin)
/// ATENÇÃO: cessionario_id NÃO existe neste model — anonimização estrutural (RN-012, RN-085)
model Proposta {
  id                          String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  casoId                      String    @map("caso_id") @db.Uuid
  cedenteId                   String    @map("cedente_id") @db.Uuid
  codigo                      String    @unique @db.VarChar(20)
  valorProposto               Decimal   @map("valor_proposto") @db.Decimal(15, 2)
  comissaoComprador           Decimal   @map("comissao_comprador") @db.Decimal(15, 2)
  valorLiquidoCedenteEstimado Decimal   @map("valor_liquido_cedente_estimado") @db.Decimal(15, 2)
  status                      StatusProposta @default(RECEBIDA)
  mensagemAdmin               String?   @map("mensagem_admin")
  rodadasContraproposta       Int       @default(0) @map("rodadas_contraproposta")
  valorContraproposta         Decimal?  @map("valor_contraproposta") @db.Decimal(15, 2)
  motivoRecusa                String?   @map("motivo_recusa")
  expiresAt                   DateTime? @map("expires_at") @db.Timestamptz
  respondidoEm                DateTime? @map("respondido_em") @db.Timestamptz
  createdAt                   DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt                   DateTime  @updatedAt @map("updated_at") @db.Timestamptz
  deletedAt                   DateTime? @map("deleted_at") @db.Timestamptz

  // Relações
  caso    Caso    @relation(fields: [casoId], references: [id])
  cedente Cedente @relation(fields: [cedenteId], references: [id])

  @@index([casoId], name: "idx_propostas_caso_id")
  @@index([cedenteId], name: "idx_propostas_cedente_id")
  @@index([status], name: "idx_propostas_status")
  @@map("propostas")
}

/// Envelope de assinatura eletrônica via ZapSign
model EnvelopeAssinatura {
  id                        String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  casoId                    String    @map("caso_id") @db.Uuid
  cedenteId                 String    @map("cedente_id") @db.Uuid
  tipoDocumento             TipoEnvelope @map("tipo_documento")
  status                    StatusEnvelope @default(PENDENTE)
  zapsignToken              String?   @map("zapsign_token") @db.VarChar(255)
  zapsignDocUrl             String?   @map("zapsign_doc_url") @db.VarChar(1000)
  urlPdfAssinado            String?   @map("url_pdf_assinado") @db.VarChar(1000)
  hashPdf                   String?   @map("hash_pdf") @db.VarChar(64)
  versaoDocumento           Int       @default(1) @map("versao_documento")
  cedenteAsssinouEm         DateTime? @map("cedente_assinou_em") @db.Timestamptz
  adminAsssinouEm           DateTime? @map("admin_assinou_em") @db.Timestamptz
  lembrete3duEnviado        Boolean   @default(false) @map("lembrete_3du_enviado")
  alertaAdmin10duEnviado    Boolean   @default(false) @map("alerta_admin_10du_enviado")
  expiresAt                 DateTime? @map("expires_at") @db.Timestamptz
  createdAt                 DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt                 DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  caso    Caso    @relation(fields: [casoId], references: [id])
  cedente Cedente @relation(fields: [cedenteId], references: [id])

  @@index([casoId], name: "idx_envelopes_caso_id")
  @@index([cedenteId], name: "idx_envelopes_cedente_id")
  @@index([zapsignToken], name: "idx_envelopes_zapsign_token")
  @@map("envelopes_assinatura")
}

/// Conta garantia (Escrow) vinculada ao caso — somente leitura para o Cedente
model ContaEscrow {
  id                      String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  casoId                  String    @unique @map("caso_id") @db.Uuid
  cedenteId               String    @map("cedente_id") @db.Uuid
  status                  StatusEscrow @default(ABERTA)
  valorTotalDeposito      Decimal?  @map("valor_total_deposito") @db.Decimal(15, 2)
  valorLiquidoCedente     Decimal?  @map("valor_liquido_cedente") @db.Decimal(15, 2)
  valorComissaoRs         Decimal?  @map("valor_comissao_rs") @db.Decimal(15, 2)
  parceiroEscrow          String?   @map("parceiro_escrow") @db.VarChar(100)
  identificadorExterno    String?   @map("identificador_externo") @db.VarChar(255)
  depositoConfirmadoEm    DateTime? @map("deposito_confirmado_em") @db.Timestamptz
  fechamentoConfirmadoEm  DateTime? @map("fechamento_confirmado_em") @db.Timestamptz
  distribuicaoPrevistaEm  DateTime? @map("distribuicao_prevista_em") @db.Timestamptz
  distribuicaoRealizadaEm DateTime? @map("distribuicao_realizada_em") @db.Timestamptz
  estornoRealizadoEm      DateTime? @map("estorno_realizado_em") @db.Timestamptz
  ultimoStatusExternoEm   DateTime? @map("ultimo_status_externo_em") @db.Timestamptz
  createdAt               DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt               DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  caso    Caso    @relation(fields: [casoId], references: [id])
  cedente Cedente @relation(fields: [cedenteId], references: [id])

  @@index([cedenteId], name: "idx_contas_escrow_cedente_id")
  @@index([status], name: "idx_contas_escrow_status")
  @@map("contas_escrow")
}

/// Notificações ao Cedente via e-mail e painel (RN-056, RN-057)
model Notificacao {
  id                 String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  cedenteId          String    @map("cedente_id") @db.Uuid
  casoId             String?   @map("caso_id") @db.Uuid
  tipo               TipoNotificacao
  titulo             String    @db.VarChar(255)
  corpo              String
  metadata           Json?
  canalEmailEnviado  Boolean   @default(false) @map("canal_email_enviado")
  canalPushEnviado   Boolean   @default(false) @map("canal_push_enviado")
  lida               Boolean   @default(false)
  lidaEm             DateTime? @map("lida_em") @db.Timestamptz
  createdAt          DateTime  @default(now()) @map("created_at") @db.Timestamptz

  // Relações
  cedente Cedente @relation(fields: [cedenteId], references: [id])
  caso    Caso?   @relation(fields: [casoId], references: [id])

  @@index([cedenteId], name: "idx_notificacoes_cedente_id")
  @@index([casoId], name: "idx_notificacoes_caso_id")
  @@index([lida], name: "idx_notificacoes_lida")
  @@map("notificacoes")
}

/// Log imutável de eventos do caso — nunca atualizar, nunca deletar
model EventoCaso {
  id               String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  casoId           String    @map("caso_id") @db.Uuid
  cedenteId        String    @map("cedente_id") @db.Uuid
  atorId           String?   @map("ator_id") @db.Uuid
  tipoAtor         TipoAtor  @map("tipo_ator")
  tipoEvento       TipoEvento @map("tipo_evento")
  statusAnterior   String?   @map("status_anterior") @db.VarChar(100)
  statusNovo       String?   @map("status_novo") @db.VarChar(100)
  cenarioAnterior  String?   @map("cenario_anterior") @db.VarChar(1)
  cenarioNovo      String?   @map("cenario_novo") @db.VarChar(1)
  dadosAdicionais  Json?     @map("dados_adicionais")
  createdAt        DateTime  @default(now()) @map("created_at") @db.Timestamptz

  // Relações
  caso    Caso    @relation(fields: [casoId], references: [id])
  cedente Cedente @relation("CedenteDoCaso", fields: [cedenteId], references: [id])
  ator    User?   @relation("AtorEvento", fields: [atorId], references: [id])

  @@index([casoId], name: "idx_eventos_caso_id")
  @@index([cedenteId], name: "idx_eventos_cedente_id")
  @@index([tipoEvento], name: "idx_eventos_tipo_evento")
  @@index([createdAt(sort: Desc)], name: "idx_eventos_created_at_desc")
  @@map("eventos_caso")
}

/// Sessões de chat com o Guardião do Retorno (RN-058 a RN-063)
model AiSession {
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  cedenteId       String    @map("cedente_id") @db.Uuid
  casoId          String?   @map("caso_id") @db.Uuid
  messages        Json      @default("[]")
  context         Json?
  totalTokens     Int?      @map("total_tokens")
  langfuseTraceId String?   @map("langfuse_trace_id") @db.VarChar(255)
  createdAt       DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt       DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  // Relações
  cedente Cedente @relation(fields: [cedenteId], references: [id])
  caso    Caso?   @relation(fields: [casoId], references: [id])

  @@index([cedenteId], name: "idx_ai_sessions_cedente_id")
  @@index([casoId], name: "idx_ai_sessions_caso_id")
  @@map("ai_sessions")
}
```

---

## 2. Convenções Aplicadas

### 2.1 Nomenclatura

| **Elemento** | **Convenção** | **Exemplo** |
|---|---|---|
| Models Prisma | PascalCase singular | `Caso`, `DocumentoDossie` |
| Campos no model | camelCase | `cedenteId`, `valorPagoCedente` |
| Tabelas no banco | snake_case plural via `@@map` | `casos`, `documentos_dossie` |
| Colunas no banco | snake_case via `@map` | `cedente_id`, `valor_pago_cedente` |
| Enums (nome) | PascalCase | `StatusCaso`, `TipoDocumento` |
| Valores de enum | UPPER_SNAKE_CASE | `CADASTRO_REALIZADO`, `EM_ANALISE` |

### 2.2 Campos de Auditoria

Todos os models de domínio contêm:

```prisma
createdAt  DateTime  @default(now()) @map("created_at") @db.Timestamptz
updatedAt  DateTime  @updatedAt @map("updated_at") @db.Timestamptz
```

Tabelas principais (`User`, `Caso`, `Proposta`) incluem soft delete:

```prisma
deletedAt  DateTime?  @map("deleted_at") @db.Timestamptz
```

`EventoCaso` é exceção: não tem `updatedAt` — log imutável por design.

### 2.3 Chaves Primárias

Todas as PKs são UUID v4 gerados pelo PostgreSQL via `pgcrypto`:

```prisma
id  String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
```

### 2.4 Valores Monetários

Todos os campos monetários usam `Decimal` com precisão `@db.Decimal(15, 2)` — evita erros de ponto flutuante em cálculos de comissão. Na camada de API, os valores são transportados em centavos (inteiros) e convertidos na entrada/saída.

### 2.5 Timestamps

Todos os campos de data/hora usam `@db.Timestamptz` (timestamp with timezone). O backend armazena sempre em UTC. O frontend converte para o timezone do usuário via `date-fns-tz`.

---

## 3. Mapeamento Enum → Regra de Negócio

| **Enum** | **Valores** | **Regras de Negócio** |
|---|---|---|
| `StatusCaso` | 13 valores (CADASTRO_REALIZADO a CANCELADO) | D01.1 seção 6 — estados visíveis ao Cedente |
| `StatusCasoInterno` | 13 valores (CAPTADO a CANCELADO) | D01.1 seção 6 — estados internos do Admin |
| `CenarioRetorno` | A, B, C, D | RN-021, RN-022, RN-025 a RN-029 |
| `TipoDocumento` | 10 tipos (6 PF + 2 adicionais PJ + reservas) | RN-041, RN-070 |
| `StatusDocumento` | PENDENTE, EM_ANALISE, VERIFICADO, REJEITADO | RN-041 a RN-045 |
| `StatusProposta` | 9 estados | RN-030 a RN-033, RN-035 a RN-037 |
| `TipoEnvelope` | 4 tipos de documento assinável | RN-024, RN-025, RN-047, RN-048 |
| `StatusEscrow` | 5 estados | RN-083, D01.3 seção 4 |
| `TipoNotificacao` | 17 tipos | RN-056 |
| `TipoEvento` | 22 tipos de evento | Logs de auditoria — imutáveis |

---

## 4. Seed Inicial (Referência)

```typescript
// prisma/seed/cedente.seed.ts
// Seed mínimo para desenvolvimento

import { PrismaClient, TipoCedente, StatusConta, StatusCaso, CenarioRetorno } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  // Cedente PF de teste
  const user = await prisma.user.create({
    data: {
      id: '00000000-0000-0000-0000-000000000001',
      email: 'cedente.teste@repasse.dev',
      name: 'Carlos Cedente Teste',
      provider: 'EMAIL',
      emailVerifiedAt: new Date(),
    },
  });

  const cedente = await prisma.cedente.create({
    data: {
      userId: user.id,
      tipo: TipoCedente.PF,
      cpfCnpj: '000.000.000-00',
      statusConta: StatusConta.ATIVA,
      lgpdConsent: true,
      lgpdConsentAt: new Date(),
    },
  });

  // Caso de teste em rascunho
  await prisma.caso.create({
    data: {
      cedenteId: cedente.id,
      codigo: 'CAS-2026-0001',
      status: StatusCaso.CADASTRO_REALIZADO,
      statusInterno: 'CAPTADO',
      cidade: 'Fortaleza',
      bairro: 'Meireles',
      estado: 'CE',
      nomeEmpreendimento: 'Edifício Teste',
      enderecoCompleto: 'Rua Teste, 123, Meireles, Fortaleza, CE',
      quartos: 2,
      areaM2: 65.0,
      temGaragem: true,
      cenario: CenarioRetorno.B,
      valorPagoCedente: 120000.00,
      rascunho: false,
      dataCadastro: new Date(),
    },
  });

  console.log('Seed do módulo Cedente concluído.');
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

---

## 5. Middleware de Soft Delete

```typescript
// prisma/middleware/soft-delete.middleware.ts
// Filtra automaticamente registros com deletedAt preenchido

import { Prisma } from '@prisma/client';

export const softDeleteMiddleware: Prisma.Middleware = async (params, next) => {
  const modelsComSoftDelete = ['User', 'Caso', 'Proposta'];

  if (modelsComSoftDelete.includes(params.model ?? '')) {
    if (params.action === 'findUnique' || params.action === 'findFirst') {
      params.action = 'findFirst';
      params.args.where = { ...params.args.where, deletedAt: null };
    }

    if (params.action === 'findMany') {
      if (params.args.where) {
        if (params.args.where.deletedAt === undefined) {
          params.args.where.deletedAt = null;
        }
      } else {
        params.args.where = { deletedAt: null };
      }
    }
  }

  return next(params);
};
```
