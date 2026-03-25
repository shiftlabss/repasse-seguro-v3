# Registro Mestre de Requisitos — CRM Repasse Seguro

| **Campo** | **Valor** |
|---|---|
| **Módulo** | CRM |
| **Versão** | v1.0 |
| **Data** | 2026-03-24 |
| **Responsável** | Pipeline Fase 3 v2.3 |
| **Docs fonte** | 01.1 a 01.5, 02, 03, 04, 05.1 a 05.5, 06, 07, 08, 09, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29 |
| **Total de REQs** | 312 |

---

> **Uso:** Esta tabela é a fonte da verdade para cobertura. A coluna Sprint é preenchida na Etapa 3. Check #12 de cada sprint constrói valida que 100% dos REQs atribuídos estão cobertos.

---

## Legenda de Tipos

| Tipo | Descrição |
|---|---|
| RN | Regra de negócio |
| RF | Requisito funcional |
| DB | Banco de dados / schema |
| API | Endpoint |
| TELA | Tela / rota frontend |
| INT | Integração externa |
| INFRA | Infraestrutura / DevOps |
| QA | Plano de testes / test case |
| UX | Contrato de UI / UX Writing |
| GLOS | Glossário — item que aparece APENAS no glossário |
| TRANSV | Regra transversal (auditoria, notificações, LGPD) |

---

## Tabela de Requisitos

| ID | Doc Fonte | Seção | Tipo | Descrição | Valor/Detalhe | Sprint |
|----|-----------|-------|------|-----------|---------------|--------|
| REQ-001 | D01.1 | RN-001 | RN | Login de usuário interno via Supabase Auth | Redirecionamento por papel: Admin RS → visão geral; Coordenador RS → painel equipe; Analista RS → fila próprios | S2 |
| REQ-002 | D01.1 | RN-002 | RN | Bloqueio por tentativas de login malsucedidas | 5 tentativas / 15 min / bloqueio 30 min / e-mail de notificação | S2 |
| REQ-003 | D01.1 | RN-003 | RN | Expiração de sessão por inatividade | 60 min inatividade / aviso 5 min antes / preservar URL de retorno | S2 |
| REQ-004 | D01.1 | RN-004 | RN | Acesso do Parceiro Externo via convite | Link convite 7 dias / status resumido apenas / sem dados pessoais | S2 |
| REQ-005 | D01.1 | RN-005 | RN | Recuperação de senha | Link validade 2 horas / proteção anti-enumeração / mesma mensagem para e-mail inexistente | S2 |
| REQ-006 | D01.1 | RN-006 | RN | Criação de novo Caso | Campos obrigatórios: nome Cedente, telefone, e-mail, empreendimento, Tabela Contrato, cenário A/B/C/D / ID formato RS-YYYY-NNNN | S3 |
| REQ-007 | D01.1 | RN-007 | RN | Avanço de estado do Caso com validação de condição de saída | Bloquear se condição não atendida / registrar carimbo + responsável no log | S3 |
| REQ-008 | D01.1 | RN-008 | RN | Cancelamento de Caso com motivo obrigatório | 5 motivos: Cedente desistiu; Documentação inválida; Nenhum Cessionário; Anuência negada; Outros (campo aberto) | S3 |
| REQ-009 | D01.1 | RN-009 | RN | Alerta SLA — cron diário 08h00 America/Fortaleza | 80% → alerta Analista RS + Coordenador RS; 100% → urgência + badge vermelho; 60 dias corridos → alerta consolidado | S3 |
| REQ-010 | D01.1 | RN-010 | RN | Criação de conta novo usuário interno | Link ativação 48 horas / e-mail duplicado bloqueado | S2 |
| REQ-011 | D01.1 | RN-011 | RN | Tour interativo de onboarding | 5 etapas: criar Caso, registrar Atividade, avançar estado, Dashboard pessoal, central de ajuda / não bloqueia o sistema | S2 |
| REQ-012 | D01.1 | RN-012 | RN | Acesso mínimo a dados pessoais (LGPD) | Mascarar: "João S." / CPF com 3 dígitos visíveis / log automático de acesso | S2 |
| REQ-013 | D01.1 | RN-013 | RN | Retenção e exclusão de dados pessoais | Cancelados: 5 anos / Concluídos: 10 anos / anonimização automática / prazo resposta titular: 15 dias corridos | S9 |
| REQ-014 | D01.1 | RN-014 | RN | Criação automática de Contato ao criar Caso | Detecção duplicata por e-mail ou CPF/CNPJ / opção vincular ou criar com flag "possível duplicata" | S5 |
| REQ-015 | D01.1 | RN-015 | RN | Alerta de atividade vencida — cron 08h00 | Badge "Atrasado" no Caso / notificação Analista RS / +3 dias úteis → Coordenador RS | S6 |
| REQ-016 | D01.1 | §5.1 | RN | Estado do Caso: Cadastro → SLA 1 dia útil / condição de saída: Cedente cadastrado + cenário | — | S3 |
| REQ-017 | D01.1 | §5.1 | RN | Estado do Caso: Simulação → SLA 2 dias úteis / condição: Cedente confirmou cenário | — | S3 |
| REQ-018 | D01.1 | §5.1 | RN | Estado do Caso: Verificação → SLA 5 dias úteis / condição: Dossiê completo + aprovado Coordenador RS | — | S3 |
| REQ-019 | D01.1 | §5.1 | RN | Estado do Caso: Publicação → SLA 1 dia útil / condição: Caso publicado no marketplace com Dossiê verificado | — | S3 |
| REQ-020 | D01.1 | §5.1 | RN | Estado do Caso: Match → SLA 3 dias úteis / condição: Cessionário identificado e interesse confirmado | — | S3 |
| REQ-021 | D01.1 | §5.1 | RN | Estado do Caso: Negociação → SLA 7 dias úteis / condição: Proposta aceita por ambas as partes | — | S3 |
| REQ-022 | D01.1 | §5.1 | RN | Estado do Caso: Anuência → SLA 10 dias úteis / condição: Anuência emitida pela incorporadora | — | S3 |
| REQ-023 | D01.1 | §5.1 | RN | Estado do Caso: Formalização → SLA 5 dias úteis / condição: Instrumento assinado + Escrow depositado | — | S3 |
| REQ-024 | D01.1 | §5.1 | RN | Estado do Caso: Concluído → condição: Escrow liberado + comissão registrada | — | S3 |
| REQ-025 | D01.1 | §5.2 | RN | Transição Cadastro → Simulação: cenário confirmado | — | S3 |
| REQ-026 | D01.1 | §5.2 | RN | Transição Simulação → Verificação: documentação iniciada | — | S3 |
| REQ-027 | D01.1 | §5.2 | RN | Transição Verificação → Publicação: Dossiê aprovado | — | S3 |
| REQ-028 | D01.1 | §5.2 | RN | Transição Publicação → Match: Cessionário identificado | — | S3 |
| REQ-029 | D01.1 | §5.2 | RN | Transição Match → Negociação: interesse confirmado | — | S3 |
| REQ-030 | D01.1 | §5.2 | RN | Transição Negociação → Anuência: proposta aceita | — | S3 |
| REQ-031 | D01.1 | §5.2 | RN | Transição Anuência → Formalização: anuência emitida | — | S3 |
| REQ-032 | D01.1 | §5.2 | RN | Transição Formalização → Concluído: Escrow liberado | — | S3 |
| REQ-033 | D01.1 | §5.2 | RN | Transição Negociação → Publicação: proposta recusada (retrocesso) | — | S3 |
| REQ-034 | D01.1 | §5.2 | RN | Transição Verificação → Verificação: pendência documental (auto-loop) | — | S3 |
| REQ-035 | D01.1 | §5.2 | RN | Transição Formalização → Anuência: documentação inválida (retrocesso) | — | S3 |
| REQ-036 | D01.1 | §5.2 | RN | Transição qualquer estado → Cancelado (exceto Concluído) | — | S3 |
| REQ-037 | D01.1 | §3.2 | RN | Matriz de permissões: 4 papéis × 18 ações declaradas | ADMIN_RS / COORDENADOR_RS / ANALISTA_RS / PARCEIRO_EXTERNO | S2 |
| REQ-038 | D01.2 | RN-039 | RN | Visualização do Pipeline em kanban e lista | Kanban: colunas = estados / Lista: ordenável por data, SLA, estado, Analista | S3 |
| REQ-039 | D01.2 | RN-040 | RN | Busca e filtros avançados no Pipeline | 7 filtros: estado, Analista, empreendimento, faixa valor, data, SLA, motivo cancelamento / busca livre em: nome Cedente, RS-XXXX, empreendimento | S3 |
| REQ-040 | D01.2 | RN-041 | RN | Redistribuição de Caso entre Analistas RS | Alerta se Analista ≥ 30 Casos abertos / notificação por e-mail ao novo Analista / log de auditoria | S3 |
| REQ-041 | D01.2 | RN-042 | RN | Retrocesso de estado do Caso | Apenas Coordenador RS / Admin RS / justificativa mínimo 20 caracteres / Concluído e Cancelado não podem ser retrocedidos | S3 |
| REQ-042 | D01.2 | RN-043 | RN | Duplicação de Caso para reaproveitamento | Copia: nome, e-mail, telefone, empreendimento, Tabela Contrato / NÃO copia: Cenário, histórico de atividades / novo ID RS-YYYY-NNNN | S3 |
| REQ-043 | D01.2 | RN-044 | RN | Arquivamento automático de Casos cancelados inativos | Verificação mensal / 90 dias sem atividade → arquivar / notificar Analista RS + Coordenador RS | S3 |
| REQ-044 | D01.2 | RN-045 | RN | Registro de Match entre Cedente e Cessionário | 1 Negociação ativa por vez / dados do Cessionário não expostos ao Cedente / registra: data Match, Cessionário, Analista RS | S4 |
| REQ-045 | D01.2 | RN-046 | RN | Registro de proposta do Cessionário | Prazo padrão validade proposta: 48 horas / alerta se abaixo do piso do Cenário / expiração muda estado para "Expirada" | S4 |
| REQ-046 | D01.2 | RN-047 | RN | Registro de contraproposta do Cedente | Prazo de 48h renovado a cada rodada / alerta ao Analista RS se > 5 rodadas | S4 |
| REQ-047 | D01.2 | RN-048 | RN | Registro de aceite de proposta | Avança Caso para "Anuência" / calcula comissão estimada (exibição apenas) / lançamento definitivo apenas no Fechamento | S4 |
| REQ-048 | D01.2 | RN-049 | RN | Encerramento de Negociação por recusa ou inatividade | Status "Recusada" ou "Expirada" / retorna Caso para "Publicação" / notifica Analista RS / flag "Recusou" no Contato | S4 |
| REQ-049 | D01.2 | RN-050 | RN | Cálculo de comissão do Cessionário | Se Δ > 0: 20% × Δ / Se Δ ≤ 0: 20% × Valor Pago pelo Cedente / exibe: Δ, base de cálculo, valor estimado, total Escrow | S4 |
| REQ-050 | D01.2 | RN-051 | RN | Cálculo de comissão do Cedente por Cenário | Cenário A: R$ 0,00 / Cenários B/C/D: 20% × (Valor Recuperado − Valor Distrato Referência) | S4 |
| REQ-051 | D01.2 | RN-052 | RN | Aplicação de desconto de comissão com tabela de limites | Até R$ 200k: 0% / R$ 200k–500k: 5% / R$ 500k–1M: 10% / acima R$ 1M: 15% / apenas Coordenador RS + Admin RS aplicam | S4 |
| REQ-052 | D01.2 | RN-053 | RN | Registro do Termo Comercial | Campos: percentual, piso, teto, PDF assinado / sobrescreve valores padrão quando presente | S4 |
| REQ-053 | D01.2 | RN-054 | RN | 3 critérios obrigatórios de Fechamento (cumulativos) | (1) instrumento cessão assinado (2) anuência incorporadora (3) comprovante Escrow | S4 |
| REQ-054 | D01.2 | RN-055 | RN | Verificação automática dos 3 critérios antes de Fechamento | Bloqueia se qualquer critério pendente / lista pendências | S4 |
| REQ-055 | D01.2 | RN-056 | RN | Registro definitivo de comissão após Fechamento | Inclui: valor cada comissão, base de cálculo, data, Analista RS, referência documentos / gera resumo financeiro: Preço Repasse, Comissão Cedente, Comissão Cessionário, Valor Líquido, Perda Evitada | S4 |
| REQ-056 | D01.2 | RN-057 | RN | Cancelamento de comissão registrada | Apenas Admin RS / justificativa mínimo 100 caracteres / aprovação segundo Admin RS (quatro olhos) / comissão nunca excluída | S4 |
| REQ-057 | D01.2 | RN-058 | RN | Estrutura obrigatória do Dossiê — 11 documentos | Tabela Atual, Tabela Contrato, extrato saldo devedor, comprovante pagamento, identidade Cedente (frente+verso), selfie vivacidade, comprovante endereço (≤90 dias), Valor Distrato Referência, instrumento cessão assinado, anuência incorporadora, comprovante Escrow | S5 |
| REQ-058 | D01.2 | RN-059 | RN | Upload de documento no Dossiê | Formatos: PDF, JPG, PNG / tamanho máximo: 20 MB / erro formato: "Formato não aceito." / erro tamanho: "O arquivo é muito grande. O limite é de 20 MB." | S5 |
| REQ-059 | D01.2 | RN-060 | RN | Validação e aprovação do Dossiê pelo Coordenador RS | SLA aprovação: 2 dias úteis / estados: Em montagem → Completo → Aprovado / Rejeitado (com pendências) / sem limite de tentativas de reenvio | S5 |
| REQ-060 | D01.2 | RN-061 | RN | Controle de versão de documentos do Dossiê | Versões anteriores arquivadas / "Histórico de versões" acessível / documentos aprovados não substituíveis sem justificativa | S5 |
| REQ-061 | D01.2 | RN-062 | RN | Compartilhamento controlado do Dossiê com Cessionário | Link temporário 7 dias / somente leitura / exclui automaticamente documentos pessoais do Cedente / log de acesso (IP, data/hora) | S5 |
| REQ-062 | D01.2 | RN-063 | RN | Integridade documental antes de avançar para Publicação | Dossiê status "Aprovado" + todos documentos presentes + Tabela Atual emissão ≤ 30 dias | S5 |
| REQ-063 | D01.3 | RN-093 | RN | Perfil completo de Contato — 11 campos estruturados | Nome, CPF/CNPJ, e-mail, telefone, papel, indicador, histórico Casos, histórico Atividades, notas, perfil investidor, tipo imóvel preferido | S5 |
| REQ-064 | D01.3 | RN-094 | RN | Perfil do Cessionário como investidor recorrente | Flag "Investidor Recorrente" após 1º Fechamento / critérios de compatibilidade: região, faixa valor, tipologia | S5 |
| REQ-065 | D01.3 | RN-095 | RN | Mesclagem de Contatos duplicados | Mantém o com mais Casos / transfere histórico / Contato secundário marcado "Mesclado" nunca excluído | S5 |
| REQ-066 | D01.3 | RN-096 | RN | Arquivamento de Contato inativo | Verificação mensal / 12 meses sem Caso ativo e sem Atividade → arquivar / reativação via novo Caso | S5 |
| REQ-067 | D01.3 | RN-097 | RN | Vinculação de Parceiro Externo como indicador | Máximo 2 indicadores por Caso (1 Cedente + 1 Cessionário) / não calcula comissão de parceria automaticamente | S5 |
| REQ-068 | D01.3 | RN-098 | RN | 8 tipos de Atividade disponíveis | Ligação, Reunião, WhatsApp, E-mail, Nota interna, Follow-up agendado, Documento enviado, Evento do sistema | S6 |
| REQ-069 | D01.3 | RN-099 | RN | Registro de Atividade passada | Campos: tipo, data/hora, duração, canal, resumo (mínimo 20 chars), resultado (4 opções) / exclusão apenas Admin RS com justificativa | S6 |
| REQ-070 | D01.3 | RN-100 | RN | Criação de Follow-up agendado | Aviso push/e-mail 30 minutos antes / Vencido se não concluído em 3 horas / máx 1 follow-up alta prioridade por Caso | S6 |
| REQ-071 | D01.3 | RN-101 | RN | Linha do tempo do Caso | Consolida: transições estado, Atividades, documentos Dossiê, propostas, alertas SLA / somente leitura / Parceiro Externo vê apenas mudanças de estado | S6 |
| REQ-072 | D01.3 | RN-102 | RN | Limite de Atividades retroativas | Analista RS: máximo 30 dias retroativos / além de 30 dias: apenas Coordenador RS + Admin RS com justificativa | S6 |
| REQ-073 | D01.3 | RN-103 | RN | Canal principal: WhatsApp Business | Mensagem vinculada automaticamente ao Caso / respostas em tempo real / janela 24h monitorada / alerta se Contato sem WhatsApp | S7 |
| REQ-074 | D01.3 | RN-104 | RN | Templates de mensagem padronizados — 10 templates | Boas-vindas; Solicitação documentos; Publicação Caso; Match; Proposta recebida; Prazo vencendo (24h); Escrow depositado; Contrato assinatura; Lembrete assinatura D+2/D+4; Caso Concluído | S7 |
| REQ-075 | D01.3 | RN-105 | RN | Registro de comunicação recebida externamente | Marcação "Registrado manualmente" na linha do tempo | S7 |
| REQ-076 | D01.3 | RN-106 | RN | Opt-out de comunicações (LGPD) | Bloqueia WhatsApp Business + e-mail / flag "Opt-out" visível / revertível apenas por nova solicitação expressa do Contato | S7 |
| REQ-077 | D01.3 | RN-107 | RN | Monitoramento de SLA — cron 08h00 / 4 níveis de alerta | 80%: amarelo → Analista RS / 100%: vermelho → Analista RS + Coordenador RS / 60 dias: crítico → Coordenador RS + Admin RS / 7 dias sem atividade: inatividade → Analista RS + Coordenador RS | S8 |
| REQ-078 | D01.3 | RN-108 | RN | Escalada automática de SLA crítico | Alerta vermelho não reconhecido em 24h → escala Admin RS / 150% do SLA → "Caso em risco de cancelamento" | S8 |
| REQ-079 | D01.3 | RN-109 | RN | Dashboard de SLA consolidado | Defasagem máx 5 min / exibe: Casos por estado/nível SLA, ranking Analistas, histograma 90 dias, previsão Fechamentos 30 dias | S8 |
| REQ-080 | D01.3 | RN-110 | RN | Relatório semanal automático — todo domingo 20h America/Fortaleza | Conteúdo: Casos abertos/avançados/concluídos/cancelados, alertas SLA, atividades por Analista, follow-ups vencidos, previsão próxima semana | S8 |
| REQ-081 | D01.3 | RN-111 | RN | Alerta: Caso em Verificação sem Dossiê após 5 dias úteis | +10 dias úteis: escalada Coordenador RS | S8 |
| REQ-082 | D01.3 | RN-112 | RN | Notificação de inatividade do Parceiro Externo | Verificação mensal / 90 dias sem indicação → notifica Analista RS / sem comunicação direta ao Parceiro | S8 |
| REQ-083 | D01.4 | RN-139 | RN | Ciclo de vida de usuários internos — 3 estados | Ativo → Suspenso → Desligado / transições: Admin RS → Suspenso, Admin RS → Ativo (reativação), Admin RS → Desligado | S9 |
| REQ-084 | D01.4 | RN-140 | RN | Suspensão de usuário interno | Motivos: Férias, Afastamento médico, Investigação interna, Outros / sessões ativas encerradas imediatamente | S9 |
| REQ-085 | D01.4 | RN-141 | RN | Desligamento de usuário interno | Pré-requisito: todos Casos redistribuídos / e-mail reutilizável apenas após 180 dias / dados preservados como "[Nome] (Desligado)" | S9 |
| REQ-086 | D01.4 | RN-142 | RN | Alteração de papel de usuário interno | Promoção Analista RS → Coordenador RS: acesso imediato ao painel consolidado / novo Admin RS: quatro olhos obrigatório | S9 |
| REQ-087 | D01.4 | RN-143 | RN | Relatório de carga da equipe | Por Analista RS: total Casos por estado, Casos SLA em risco/vencido, follow-ups vencidos, tempo médio últimos 10 Casos | S9 |
| REQ-088 | D01.4 | RN-144 | RN | Dashboard Executivo — 7 métricas em tempo real (≤5 min) | Casos abertos, taxa conversão, ciclo médio total, ciclo médio por estado, Casos em risco SLA, previsão Fechamentos, taxa cancelamento | S8 |
| REQ-089 | D01.4 | RN-145 | RN | Métricas financeiras no Dashboard — aba "Financeiro" (Admin RS only) | Receita confirmada, receita estimada, ticket médio, margem de contribuição, descontos concedidos, indicador breakeven (≥3 Casos/mês) | S8 |
| REQ-090 | D01.4 | RN-146 | RN | Análise de funil de conversão | Taxa cancelamento > 20% → "Gargalo identificado" em vermelho / drilling down por estado | S8 |
| REQ-091 | D01.4 | RN-147 | RN | 14 parâmetros configuráveis do CRM | SLA por estado, limite Casos Analista (30), validade proposta (48h), convite Parceiro (7 dias), arquivamento Contato (12 meses), arquivamento Caso (90 dias), tabela descontos, ciclo alerta (60 dias), follow-up aviso (30 min), inatividade sessão (60 min), retenção cancelados (5 anos), retenção concluídos (10 anos), hora relatório semanal (domingo 20h), retroatividade atividades (30 dias) | S9 |
| REQ-092 | D01.4 | RN-148 | RN | Processo de alteração de parâmetro crítico | 4 parâmetros críticos: SLA, limites desconto, retenção dados, limite Casos / quatro olhos obrigatório | S9 |
| REQ-093 | D01.4 | RN-149 | RN | Gestão de templates de mensagem | Versionamento v1.0, v1.1, v2.0 / templates desativados não excluídos | S9 |
| REQ-094 | D01.4 | RN-150 | RN | 10 relatórios disponíveis com controle de papel | Pipeline completo, desempenho Analista, comissões confirmadas, comissões estimadas, funil, contatos, atividades, cancelamentos, log auditoria, exportação LGPD | S9 |
| REQ-095 | D01.4 | RN-151 | RN | Exportação de dados — formatos CSV e XLSX | Com dados pessoais: restrito Admin RS / log: usuário, data/hora, filtros, quantidade registros | S9 |
| REQ-096 | D01.4 | RN-152 | RN | Relatório de inteligência de mercado | Conteúdo: distribuição Δ por empreendimento/região, tempo médio Match, taxa anuência negada, perfil Cessionários recorrentes, sazonalidade / k-anonimidade: agrupamentos < 5 Casos mascarados | S9 |
| REQ-097 | D01.5 | RN-183 | INT | Sincronização Casos Plataforma RS → CRM | Tempo real (máx 2 min) / origem "Plataforma" / Analista de plantão ou vinculado ao Parceiro indicador / sem Analista: "Não atribuído" + alerta Coordenador RS | S7 |
| REQ-098 | D01.5 | RN-184 | INT | Sincronização documentos Dossiê enviados pela plataforma | Retry: 5 min, por 1 hora / se não resolver: notifica Admin RS | S7 |
| REQ-099 | D01.5 | RN-185 | INT | Sincronização interesse Cessionário Plataforma → CRM | Notifica Analista RS / Match NÃO criado automaticamente — requer validação Analista RS | S7 |
| REQ-100 | D01.5 | RN-186 | INT | Atualização de estado do Caso → Plataforma | Tempo real (máx 2 min) / falha: log + notifica Admin RS / CRM permanece correto | S7 |
| REQ-101 | D01.5 | RN-187 | INT | Envio de mensagens via WhatsApp Business API | Janela 24h monitorada / falha: log + notifica Analista RS / canal alternativo sugerido | S7 |
| REQ-102 | D01.5 | RN-188 | INT | Conformidade WhatsApp Business: 3 categorias | Utilitária, Autenticação, Marketing (opt-in explícito obrigatório) / sem envio promocional automático | S7 |
| REQ-103 | D01.5 | RN-189 | INT | Envio de contrato via ZapSign | Signatários: Cedente, Cessionário, representante RS / lembretes: D+2 (1º) + D+4 (2º + alerta Analista RS) + D+5 (expirado + Admin RS) / PDF assinado → Dossiê automaticamente | S7 |
| REQ-104 | D01.5 | RN-190 | INT | Falha na integração ZapSign | Retry manual: 5 min / 3 falhas consecutivas → Admin RS notificado / Caso permanece em "Formalização" | S7 |
| REQ-105 | D01.5 | RN-191 | INT | Confirmação depósito Conta Escrow via webhook | Webhook + comprovante automático no Dossiê / sem webhook em 24h: alerta Analista RS + Coordenador RS | S7 |
| REQ-106 | D01.5 | RN-192 | INT | Registro manual de comprovante de Escrow | Flag "Inserção manual" / aprovação Coordenador RS antes de aceitar como documento de Fechamento | S7 |
| REQ-107 | D01.5 | RN-193 | INT | Autenticação via Supabase Auth | JWT com claims de papel validados a cada requisição / renovação automática de token se sessão ativa | S2 |
| REQ-108 | D01.5 | RN-194 | TRANSV | Log de auditoria imutável | Campos: usuário, papel, data/hora UTC+Fortaleza, IP, ação, objeto, campo, valor anterior, valor novo / retenção 10 anos / apenas Admin RS acessa | S1 |
| REQ-109 | D01.5 | RN-195 | TRANSV | Notificações push e e-mail — hierarquia 4 prioridades | Crítica: Push+E-mail / Alta: Push+E-mail / Média: Push ou E-mail / Baixa: E-mail apenas / Crítica e Alta não desativáveis | S8 |
| REQ-110 | D01.5 | RN-196 | TRANSV | Protocolo de falha de integração | 1ª falha: retry 2 min / 2ª falha: retry 10 min / 3ª: alerta Admin RS + Analista RS / 1h contínua: alerta crítico | S1 |
| REQ-111 | D01.5 | RN-197 | TRANSV | Busca global Ctrl+K / Cmd+K | Retorna: Casos (ID, empreendimento), Contatos (mascarados), Atividades (texto) / respeita permissões / não indexa: Dossiê PDFs, logs, versões templates | S1 |
| REQ-112 | D01.5 | RN-198 | TRANSV | Idioma pt-BR + fuso America/Fortaleza | UTC internamente / exibição America/Fortaleza / valores: R$ X.XXX,XX | S1 |
| REQ-113 | D01.5 | RN-199 | TRANSV | Responsividade mínima 768px / acessibilidade WCAG 2.1 AA | Contraste mínimo 4.5:1 / rótulos textuais + indicadores de status / funcionalidades críticas campo: registro Atividade, avanço estado, upload documento | S1 |
| REQ-114 | D01.5 | RN-200 | TRANSV | Backup e recuperação | RPO: 24 horas / RTO: 4 horas / backup diário 30 dias retenção / Admin RS notificado em falha | S1 |
| REQ-115 | D02 | §2.2 | INFRA | Stack backend obrigatória | Node.js 22.x LTS / NestJS 10+ / TypeScript 5.4+ strict / Prisma 6+ / PostgreSQL 17+ Supabase / Redis 7.4+ Upstash / RabbitMQ 4+ CloudAMQP | S1 |
| REQ-116 | D02 | §3.3 | INFRA | Stack frontend obrigatória | Next.js 15+ App Router / React 19+ / TypeScript 5.4+ / Tailwind CSS 4+ / shadcn/ui / Framer Motion 12+ / TanStack Query 5+ / Zustand 5+ / TanStack Table 8+ / React Hook Form 7+ / Zod 3+ | S1 |
| REQ-117 | D02 | §2.4 | DB | Convenções de banco de dados | UUID v4 PKs / soft delete deleted_at / @db.Timestamptz obrigatório / Decimal(15,2) monetário / audit schema append-only | S1 |
| REQ-118 | D02 | §2.5 | DB | Tabela `cases` — entidade central | case_number UK, state enum, cedente_contact_id FK, cessionario_contact_id FK, assigned_to FK, scenario, contract_value, current_table_value, delta, enterprise_name, enterprise_address, cancel_reason, cancel_reason_detail, version, state_changed_at, estimated_close_at | S1 |
| REQ-119 | D02 | §2.5 | DB | Tabela `contacts` — Cedentes, Cessionários, Parceiros, Incorporadoras | full_name, email, phone, cpf_cnpj, role, status, has_opt_out, is_recurrent_investor, is_possible_duplicate, primary_contact_id FK | S1 |
| REQ-120 | D02 | §2.5 | DB | Tabela `activities` — todos os tipos de interação | — | S1 |
| REQ-121 | D02 | §2.5 | DB | Tabela `communications` — mensagens WhatsApp e e-mail | — | S1 |
| REQ-122 | D02 | §2.5 | DB | Tabela `commissions` — comissões estimadas e confirmadas | — | S1 |
| REQ-123 | D02 | §2.5 | DB | Tabela `dossier_documents` — documentos com versão e status | — | S1 |
| REQ-124 | D02 | §2.5 | DB | Tabela `crm_users` / `users` — usuários internos | email UK, full_name, role enum, status enum, avatar_url, last_login_at | S1 |
| REQ-125 | D02 | §2.5 | DB | Tabela `sla_alerts` — alertas de SLA | — | S1 |
| REQ-126 | D02 | §2.5 | DB | Tabela `audit_log` — schema separado append-only | — | S1 |
| REQ-127 | D02 | §2.5 | DB | Tabela `system_parameters` / `system_configs` — parâmetros configuráveis | — | S1 |
| REQ-128 | D02 | §2.5 | DB | Tabela `message_templates` — templates de mensagem | — | S1 |
| REQ-129 | D02 | §2.5 | DB | Tabela `proposals` — propostas e contrapropostas | — | S1 |
| REQ-130 | D02 | §2.5 | DB | Tabela `case_status_history` — histórico de transições (append-only) | from_state, to_state, changed_by FK, justification | S1 |
| REQ-131 | D02 | §2.5 | DB | Tabela `tags` + `contact_tags` — N:N entre contacts e tags | — | S1 |
| REQ-132 | D02 | §2.5 | DB | Tabela `notification_logs` — log de notificações enviadas | — | S1 |
| REQ-133 | D02 | §2.2 | DB | Enum CaseState | CADASTRO, SIMULACAO, VERIFICACAO, PUBLICACAO, MATCH, NEGOCIACAO, ANUENCIA, FORMALIZACAO, CONCLUIDO, CANCELADO | S1 |
| REQ-134 | D02 | §2.2 | DB | Enum UserRole | ADMIN_RS, COORDENADOR_RS, ANALISTA_RS, PARCEIRO_EXTERNO | S1 |
| REQ-135 | D02 | §2.2 | DB | Enum UserStatus | ATIVO, SUSPENSO, DESLIGADO | S1 |
| REQ-136 | D02 | §2.2 | DB | Enum NegotiationState | EM_ANDAMENTO, PROPOSTA_ENVIADA, CONTRAPROPOSTA_RECEBIDA, ACEITA, RECUSADA, EXPIRADA | S4 |
| REQ-137 | D02 | §2.2 | DB | Enum DossierState | EM_MONTAGEM, COMPLETO, APROVADO, REJEITADO | S5 |
| REQ-138 | D02 | §2.2 | DB | Enum CommissionState | ESTIMADA, PENDENTE_FECHAMENTO, CONFIRMADA, CANCELADA | S4 |
| REQ-139 | D02 | §6 | INFRA | Controles de segurança: TLS 1.3+, Helmet CSP+HSTS, HMAC-SHA256 webhooks, rate limit 200 req/min geral / 10 req/min auth | — | S1 |
| REQ-140 | D02 | §5 | INFRA | Infraestrutura produção: Railway (API) / Vercel (Frontend) / Supabase (Banco) / Upstash (Redis) / CloudAMQP (RabbitMQ) | — | S1 |
| REQ-141 | D06 | §1.1 | TELA | Layout shell: Topbar h-14 (logo, busca global Ctrl+K, notificações, avatar) + Sidebar w-64 (collapsible para w-16 < 1280px) | — | S1 |
| REQ-142 | D06 | §2.1 | TELA | T-CRM-001 — Login / Autenticação | Rota: /login / Estados: padrão, erro credenciais, sessão expirada, usuário suspenso | S2 |
| REQ-143 | D06 | §2.1 | TELA | T-CRM-002 — Recuperação de senha | Rota: /forgot-password | S2 |
| REQ-144 | D06 | §2.1 | TELA | T-CRM-003 — Redefinição de senha | Rota: /reset-password | S2 |
| REQ-145 | D06 | §2.2 | TELA | T-CRM-010 — Dashboard principal | Variantes por papel: Admin RS + Coordenador RS (métricas executivas) / Analista RS (Casos próprios, agenda, alertas SLA) | S8 |
| REQ-146 | D06 | — | TELA | T-CRM-020 — Pipeline — Visão Kanban | Rota: /pipeline/kanban | S3 |
| REQ-147 | D06 | — | TELA | T-CRM-021 — Pipeline — Visão Lista | Rota: /pipeline/list | S3 |
| REQ-148 | D06 | — | TELA | T-CRM-022 — Pipeline — Detalhes do Caso | Rota: /cases/[id] | S3 |
| REQ-149 | D06 | — | TELA | T-CRM-030 — Negociação — Lista de Casos | Rota: /negotiations | S4 |
| REQ-150 | D06 | — | TELA | T-CRM-031 — Negociação — Painel do Caso | Rota: /negotiations/[caseId] | S4 |
| REQ-151 | D06 | — | TELA | T-CRM-032 — Modal — Registrar Proposta | — | S4 |
| REQ-152 | D06 | — | TELA | T-CRM-033 — Modal — Registrar Contraproposta | — | S4 |
| REQ-153 | D06 | — | TELA | T-CRM-034 — Modal — Aceitar Proposta / Encerrar Negociação | — | S4 |
| REQ-154 | D06 | — | TELA | T-CRM-040 — Dossiê — Checklist de Documentos | Rota: /cases/[id]/dossier | S5 |
| REQ-155 | D06 | — | TELA | T-CRM-041 — Dossiê — Detalhe do Documento | Rota: /cases/[id]/dossier/[docId] | S5 |
| REQ-156 | D06 | — | TELA | T-CRM-042 — Modal — Aprovação do Dossiê (Coordenador RS) | — | S5 |
| REQ-157 | D06 | — | TELA | T-CRM-050 — Contatos — Lista | Rota: /contacts | S5 |
| REQ-158 | D06 | — | TELA | T-CRM-051 — Contatos — Perfil do Contato | Rota: /contacts/[id] | S5 |
| REQ-159 | D06 | — | TELA | T-CRM-052 — Contatos — Formulário de Criação/Edição | Rota: /contacts/new, /contacts/[id]/edit | S5 |
| REQ-160 | D06 | — | TELA | T-CRM-053 — Modal — Mesclar Contatos Duplicados | — | S5 |
| REQ-161 | D06 | — | TELA | T-CRM-060 — Atividades — Agenda de Follow-ups | Rota: /activities | S6 |
| REQ-162 | D06 | — | TELA | T-CRM-061 — Modal — Registrar Atividade / Follow-up | — | S6 |
| REQ-163 | D06 | — | TELA | T-CRM-070 — Comunicação — Caixa de Mensagens do Caso | Rota: /cases/[id]/communications | S7 |
| REQ-164 | D06 | — | TELA | T-CRM-071 — Modal — Selecionar Template | — | S7 |
| REQ-165 | D06 | — | TELA | T-CRM-080 — SLA Monitor — Dashboard de SLA | Rota: /sla | S8 |
| REQ-166 | D06 | — | TELA | T-CRM-081 — SLA Monitor — Detalhe do Alerta | Rota: /sla/[alertId] | S8 |
| REQ-167 | D06 | — | TELA | T-CRM-090 — Equipe — Lista de Usuários | Rota: /team | S9 |
| REQ-168 | D06 | — | TELA | T-CRM-091 — Equipe — Perfil do Usuário | Rota: /team/[userId] | S9 |
| REQ-169 | D06 | — | TELA | T-CRM-092 — Equipe — Carga Atual | Rota: /team/workload | S9 |
| REQ-170 | D06 | — | TELA | T-CRM-093 — Modal — Convidar / Suspender / Desligar Usuário | — | S9 |
| REQ-171 | D06 | — | TELA | T-CRM-100 — Relatórios — Hub | Rota: /reports | S9 |
| REQ-172 | D06 | — | TELA | T-CRM-101 — Relatórios — Pipeline e Conversão | Rota: /reports/pipeline | S9 |
| REQ-173 | D06 | — | TELA | T-CRM-102 — Relatórios — Desempenho da Equipe | Rota: /reports/team | S9 |
| REQ-174 | D06 | — | TELA | T-CRM-103 — Relatórios — Financeiro (Admin RS only) | Rota: /reports/financial | S9 |
| REQ-175 | D06 | — | TELA | T-CRM-104 — Relatórios — Inteligência de Mercado | Rota: /reports/market | S9 |
| REQ-176 | D06 | — | TELA | T-CRM-110 — Configurações — Parâmetros do Sistema | Rota: /settings/parameters | S9 |
| REQ-177 | D06 | — | TELA | T-CRM-111 — Configurações — Templates de Mensagem | Rota: /settings/templates | S9 |
| REQ-178 | D06 | — | TELA | T-CRM-112 — Configurações — Log de Auditoria | Rota: /settings/audit-log | S9 |
| REQ-179 | D06 | — | TELA | T-CRM-113 — Configurações — Perfil do Usuário (pessoal) | Rota: /settings/profile | S2 |
| REQ-180 | D16 | §2 | API | Base URL: `https://api.repasseseguro.com.br/api/v1/crm` / padrão resposta `{ data, meta }` / RFC 7807 para erros | Rate limit: 200 req/min geral / 10 req/min auth | S1 |
| REQ-181 | D16 | §3 | API | POST /auth/login | — | S2 |
| REQ-182 | D16 | §3 | API | POST /auth/logout | — | S2 |
| REQ-183 | D16 | §3 | API | POST /auth/forgot-password | — | S2 |
| REQ-184 | D16 | §3 | API | POST /auth/reset-password | — | S2 |
| REQ-185 | D16 | §3 | API | POST /auth/refresh | — | S2 |
| REQ-186 | D16 | §4 | API | GET /team | — | S9 |
| REQ-187 | D16 | §4 | API | POST /team/invite | — | S9 |
| REQ-188 | D16 | §4 | API | PATCH /team/:id | — | S9 |
| REQ-189 | D16 | §4 | API | POST /team/:id/suspend | — | S9 |
| REQ-190 | D16 | §4 | API | POST /team/:id/terminate | — | S9 |
| REQ-191 | D16 | §5 | API | GET /cases | — | S3 |
| REQ-192 | D16 | §5 | API | POST /cases | — | S3 |
| REQ-193 | D16 | §5 | API | GET /cases/:id | — | S3 |
| REQ-194 | D16 | §5 | API | PATCH /cases/:id | — | S3 |
| REQ-195 | D16 | §5 | API | POST /cases/:id/advance | — | S3 |
| REQ-196 | D16 | §5 | API | POST /cases/:id/cancel | — | S3 |
| REQ-197 | D16 | §5 | API | POST /cases/:id/rollback | — | S3 |
| REQ-198 | D16 | §5 | API | POST /cases/:id/duplicate | — | S3 |
| REQ-199 | D16 | §5 | API | POST /cases/:id/assign | — | S3 |
| REQ-200 | D16 | §6 | API | GET /contacts | — | S5 |
| REQ-201 | D16 | §6 | API | POST /contacts | — | S5 |
| REQ-202 | D16 | §6 | API | GET /contacts/:id | — | S5 |
| REQ-203 | D16 | §6 | API | PATCH /contacts/:id | — | S5 |
| REQ-204 | D16 | §6 | API | POST /contacts/:id/merge | — | S5 |
| REQ-205 | D16 | §7 | API | GET /cases/:id/activities | — | S6 |
| REQ-206 | D16 | §7 | API | POST /cases/:id/activities | — | S6 |
| REQ-207 | D16 | §7 | API | PATCH /cases/:id/activities/:activityId | — | S6 |
| REQ-208 | D16 | §8 | API | GET /cases/:id/communications | — | S7 |
| REQ-209 | D16 | §8 | API | POST /cases/:id/communications/whatsapp | — | S7 |
| REQ-210 | D16 | §8 | API | POST /cases/:id/communications/email | — | S7 |
| REQ-211 | D16 | §9 | API | GET /cases/:id/dossier | — | S5 |
| REQ-212 | D16 | §9 | API | POST /cases/:id/dossier/documents | — | S5 |
| REQ-213 | D16 | §9 | API | POST /cases/:id/dossier/approve | — | S5 |
| REQ-214 | D16 | §9 | API | POST /cases/:id/dossier/share-link | — | S5 |
| REQ-215 | D16 | §10 | API | GET /cases/:id/negotiations | — | S4 |
| REQ-216 | D16 | §10 | API | POST /cases/:id/negotiations | — | S4 |
| REQ-217 | D16 | §10 | API | POST /cases/:id/negotiations/:negId/proposal | — | S4 |
| REQ-218 | D16 | §10 | API | POST /cases/:id/negotiations/:negId/accept | — | S4 |
| REQ-219 | D16 | §10 | API | POST /cases/:id/negotiations/:negId/close | — | S4 |
| REQ-220 | D16 | §11 | API | GET /cases/:id/commission | — | S4 |
| REQ-221 | D16 | §11 | API | POST /cases/:id/commission/discount | — | S4 |
| REQ-222 | D16 | §11 | API | POST /cases/:id/commission/cancel | — | S4 |
| REQ-223 | D16 | §12 | API | GET /sla/alerts | — | S8 |
| REQ-224 | D16 | §12 | API | POST /sla/alerts/:id/acknowledge | — | S8 |
| REQ-225 | D16 | §13 | API | GET /reports/:type | — | S9 |
| REQ-226 | D16 | §13 | API | POST /reports/:type/export | — | S9 |
| REQ-227 | D16 | §14 | API | GET /config/parameters | — | S9 |
| REQ-228 | D16 | §14 | API | PATCH /config/parameters | — | S9 |
| REQ-229 | D16 | §14 | API | GET /config/templates | — | S9 |
| REQ-230 | D16 | §14 | API | POST /config/templates | — | S9 |
| REQ-231 | D16 | §15 | API | POST /webhooks/zapsign | — | S7 |
| REQ-232 | D16 | §15 | API | POST /webhooks/escrow | — | S7 |
| REQ-233 | D16 | §15 | API | POST /webhooks/whatsapp | — | S7 |
| REQ-234 | D16 | §15 | API | POST /webhooks/platform | — | S7 |
| REQ-235 | D17 | §1 | INT | Integração Plataforma RS — webhook bidirecional | Env vars: PLATFORM_WEBHOOK_SECRET, PLATFORM_API_URL | S7 |
| REQ-236 | D17 | §2 | INT | Integração Meta Cloud API (WhatsApp Business) | Env vars: META_WABA_TOKEN, META_PHONE_NUMBER_ID, META_WEBHOOK_VERIFY_TOKEN | S7 |
| REQ-237 | D17 | §3 | INT | Integração ZapSign | Env vars: ZAPSIGN_API_TOKEN, ZAPSIGN_WEBHOOK_SECRET | S7 |
| REQ-238 | D17 | §4 | INT | Integração Celcoin (Conta Escrow) | Env vars: CELCOIN_CLIENT_ID, CELCOIN_CLIENT_SECRET, CELCOIN_WEBHOOK_SECRET / HMAC-SHA256 | S7 |
| REQ-239 | D18 | §1 | INFRA | Supabase Auth — 4 roles como claims JWT | ADMIN_RS, COORDENADOR_RS, ANALISTA_RS, PARCEIRO_EXTERNO validados a cada requisição | S2 |
| REQ-240 | D18 | §2 | INFRA | RLS PostgreSQL — políticas por assigned_to e role | Analista RS vê apenas casos atribuídos / Parceiro Externo sem acesso a dados pessoais | S1 |
| REQ-241 | D20 | §2 | INFRA | Filtro de exceções global NestJS — CrmExceptionFilter | RFC 7807 / Sentry para P0/P1 / DLQ para falhas integração | S1 |
| REQ-242 | D20 | §3 | INFRA | 35 error codes com prefixo CRM- documentados | P0: sistema inoperante (<15 min) / P1: alto (<30 min) / P2: médio / P3: baixo | S1 |
| REQ-243 | D21 | §1 | INFRA | Sistema de notificações — RabbitMQ workers | Filas: sla-alerts, follow-up-reminders, weekly-report, integration-events / DLQ obrigatória | S8 |
| REQ-244 | D21 | §2 | INFRA | Workers: sla-checker (cron 08h00) + notification.worker + weekly-report (cron domingo 20h) | — | S8 |
| REQ-245 | D22 | §1 | INFRA | Variáveis de ambiente obrigatórias — setup local | DATABASE_URL, SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_KEY, REDIS_URL, RABBITMQ_URL, META_WABA_TOKEN, ZAPSIGN_API_TOKEN, CELCOIN_CLIENT_ID, SENTRY_DSN | S1 |
| REQ-246 | D23 | §1 | INFRA | Git flow: main + develop + feature/* + hotfix/* | Commits Conventional Commits / PRs obrigatórios para develop | S1 |
| REQ-247 | D24 | §1 | INFRA | 3 ambientes: development / staging / production | development: localhost:3000/3001 / staging: Vercel preview + Railway staging / production: crm.repasseseguro.com.br | S1 |
| REQ-248 | D24 | §2 | INFRA | Pipeline CI/CD GitHub Actions | lint → typecheck → test → build → deploy / prisma migrate deploy antes do deploy API | S1 |
| REQ-249 | D24 | §3 | INFRA | Rollback em ≤ 10 min / feature flags PostHog | — | S1 |
| REQ-250 | D25 | §1 | INFRA | Observabilidade: PostHog + Sentry + Pino + Supabase Dashboard + Railway Metrics | Sentry SLA P0: < 15 min / Request ID propagado frontend → backend | S1 |
| REQ-251 | D26 | §1 | INFRA | Runbook operacional — procedimentos de incidente | — | SFinal |
| REQ-252 | D27 | §2 | QA | Pirâmide de testes: Jest (unitário BE) + Vitest (unitário FE) + Supertest (integração) + Playwright (E2E) | Unitário: 80% módulos críticos / Integração: 60% endpoints / E2E: 100% fluxos críticos | S10 |
| REQ-253 | D27 | §3 | QA | CasesService — cobertura 80% unitário + integração obrigatória | transições de status, criação com snapshot | S3 |
| REQ-254 | D27 | §3 | QA | CommissionsService — cobertura 85% + integração | cálculo nos 4 cenários A/B/C/D | S4 |
| REQ-255 | D27 | §3 | QA | SlaMonitorService — cobertura 80% + integração | detecção alertas, disparo notificações | S8 |
| REQ-256 | D27 | §3 | QA | NegotiationsService — cobertura 75% + integração | criação proposta, aceite, contraproposta | S4 |
| REQ-257 | D27 | §3 | QA | ZapSignService — cobertura 75% + integração | criação envelope, processamento webhook | S7 |
| REQ-258 | D27 | §3 | QA | ReportsService — cobertura 70% + integração | geração relatório semanal | S9 |
| REQ-259 | D27 | §3 | QA | ContactsService — cobertura 70% + integração | CRUD completo | S5 |
| REQ-260 | D27 | §3 | QA | ActivitiesService — cobertura 70% + integração | registro atividade por tipo | S6 |
| REQ-261 | D27 | §4 | QA | 5 fluxos E2E críticos: criação Caso, avanço pipeline, assinatura ZapSign, geração relatório, alertas SLA | — | S10 |
| REQ-262 | D28 | §1 | QA | Checklist de qualidade: A11y (WCAG 2.1 AA), segurança (OWASP Top 10), performance (API < 200ms p95), LGPD | — | S10 |
| REQ-263 | D29 | §1 | QA | Go-Live Playbook — smoke tests pré-launch | Smoke tests: login, criar Caso, avançar estado, upload Dossiê, enviar WhatsApp | SFinal |
| REQ-264 | D15 | §1 | INFRA | Arquitetura de pastas: monorepo Turborepo `apps/api/modules/crm/` + `apps/web-crm/` + `packages/shared` + `packages/database` + `packages/ui` | — | S1 |
| REQ-265 | D13 | §1 | DB | Schema Prisma — migrations versionadas em `prisma/migrations/` / `prisma migrate deploy` em produção | — | S1 |
| REQ-266 | D13 | §2 | DB | Seed: `prisma/seed.ts` — dados iniciais para dev/testes, nunca em produção | — | S1 |
| REQ-267 | D09 | §1 | UX | Contratos de UI por tela — estados obrigatórios: loading (skeleton), empty state, error state, sem permissão (403) | Spinners globais PROIBIDOS — usar skeleton | S1 |
| REQ-268 | D09 | §2 | UX | Componentes shadcn/ui customizados com tokens do Brand Guide (Doc 03) | — | S1 |
| REQ-269 | D08 | §1 | UX | UX Writing — mensagens de sucesso, erro e empty state por módulo | — | S1 |
| REQ-270 | D04 | §1 | UX | Motion Spec — tokens de duração para animações com Framer Motion 12+ | — | S1 |
| REQ-271 | D03 | §1 | INFRA | Brand Guide — cores, tipografia e tokens de design do CRM | — | S1 |
| REQ-272 | D07 | §1 | UX | Wireframes — 54 telas documentadas T-CRM-001 a T-CRM-054 com componentes por tela | — | SFinal |
| REQ-273 | D10 | GLOS | GLOS | Optimistic Locking — campo `version Int` em entidades editáveis / rejitar se version divergir | Previne sobrescrita silenciosa em edições concorrentes | S3 |
| REQ-274 | D10 | GLOS | GLOS | Server-Sent Events (SSE) — push de notificações em tempo real sem polling | Alertas SLA, follow-ups vencidos | S8 |
| REQ-275 | D10 | GLOS | GLOS | JWT armazenado em cookie HttpOnly — nunca em localStorage | — | S2 |
| REQ-276 | D10 | GLOS | GLOS | Refresh Token gerenciado automaticamente pelo Supabase — rotação automática | — | S2 |
| REQ-277 | D10 | GLOS | GLOS | Turborepo — pipeline de builds com cache de tarefas + builds paralelos | — | S1 |
| REQ-278 | D10 | GLOS | GLOS | Soft Delete — middleware Prisma filtra automaticamente registros com `deleted_at IS NOT NULL` | — | S1 |
| REQ-279 | D10 | GLOS | GLOS | k-anonimidade — agrupamentos < 5 Casos mascarados no relatório de inteligência | Origem: RN-152 | S9 |
| REQ-280 | D10 | GLOS | GLOS | Dead Letter Queue (DLQ) — fila para mensagens que falharam após N tentativas / monitorada via CloudAMQP | — | S1 |
| REQ-281 | D05.2 | RF-CRM-016 a RF-CRM-070 | RF | PRD Parte 2: RFs do módulo Pipeline de Casos (RF-CRM-016 a RF-CRM-030) | Mapeados de RN-039 a RN-044 | S3 |
| REQ-282 | D05.3 | RF-CRM-071 a RF-CRM-130 | RF | PRD Parte 3: RFs dos módulos Negociação, Comissão e Dossiê | Mapeados de RN-045 a RN-063 | S4/S5 |
| REQ-283 | D05.4 | RF-CRM-131 a RF-CRM-180 | RF | PRD Parte 4: RFs dos módulos Contatos, Atividades, Comunicação, SLA | Mapeados de RN-093 a RN-112 | S5/S6/S7/S8 |
| REQ-284 | D05.5 | RF-CRM-181 a RF-CRM-214 | RF | PRD Parte 5: RFs dos módulos Equipe, Dashboard, Configurações, Relatórios, Integrações | Mapeados de RN-139 a RN-200 | S7/S8/S9 |
| REQ-285 | D19 | §1 | INT | Integração passiva Dani-Admin — webhook para escalada de Casos para o CRM (MVP) | Sem agente autônomo na v1.0 / Fase 2: LangChain.js + GPT-4o | S7 |
| REQ-286 | D19 | §2 | INT | Integração passiva Dani-Cedente/Dani-Cessionário — webhook de alertas de interesse (MVP) | — | S7 |
| REQ-287 | D11 | §2 | INFRA | CRM sem app mobile nativo na v1.0 — responsivo ≥ 768px via Next.js 15 App Router + Tailwind CSS | Breakpoints: tablet 768px / desktop-sm 1280px / desktop-lg 1440px+ | S1 |
| REQ-288 | D11 | §3 | INFRA | 4 funcionalidades críticas de campo testadas em tablet (768–1279px): registro Atividade, avanço estado, upload documento, consulta pipeline | — | S10 |
| REQ-289 | D14 | §2 | INFRA | SLA de performance API: < 200ms no p95 para endpoints críticos | — | S10 |
| REQ-290 | D14 | §3 | INFRA | 6 fluxos críticos documentados com diagramas de sequência: (1) criação Caso, (2) avanço de estado, (3) upload Dossiê, (4) Fechamento, (5) alerta SLA, (6) envio ZapSign | — | S1 |
| REQ-291 | D14 | §4 | INFRA | 7 ADRs: CRM-ADR-001 (Next.js App Router), CRM-ADR-002 (Supabase Realtime alertas SLA), CRM-ADR-003 (sem mobile MVP), CRM-ADR-004 (Dani-Admin Fase 2) + 3 adicionais | — | S1 |
| REQ-292 | D02 | §2.5 | INFRA | Supabase Realtime — subscriptions em `cases`, `dossier_documents`, `sla_alerts` | — | S8 |
| REQ-293 | D02 | §2.2 | INFRA | Pino 9+ — logs estruturados JSON obrigatórios em todo serviço NestJS | Request ID, timestamp, nível, contexto | S1 |
| REQ-294 | D02 | §2.2 | INFRA | @nestjs/swagger — Swagger/OpenAPI obrigatório em todo endpoint | — | S1 |
| REQ-295 | D02 | §2.2 | INFRA | Helmet 8+ — headers de segurança desde o primeiro commit | CSP, HSTS, X-Frame-Options | S1 |
| REQ-296 | D02 | §2.2 | INFRA | class-validator + class-transformer — validação de DTOs via pipes NestJS | Toda entrada de dados validada via pipes | S1 |
| REQ-297 | D02 | §3.3 | INFRA | PostHog — analytics de produto, dados anonimizados, sem dados pessoais de Cedentes/Cessionários | — | S1 |
| REQ-298 | D02 | §3.3 | INFRA | @sentry/nextjs 9+ — error tracking frontend com breadcrumbs | Source maps obrigatórios em build | S1 |
| REQ-299 | D02 | §3.3 | INFRA | Zustand 5+ — estado global de UI (filtros Pipeline, preferências) / dados de servidor via TanStack Query | — | S1 |
| REQ-300 | D02 | §3.3 | INFRA | TanStack Table 8+ — tabelas complexas com ordenação, filtro e paginação | — | S1 |
| REQ-301 | D02 | §2.3 | INT | Supabase Storage — documentos Dossiê via signed URLs / nomenclatura UUID v4 / bucket separado por tipo documento | — | S5 |
| REQ-302 | D16 | §1.5 | API | Error codes CRM- padronizados por módulo | CRM-001 a CRM-035+ | S1 |
| REQ-303 | D16 | §1.3 | API | Paginação cursor-based nas listagens: meta.page, meta.per_page, meta.total, meta.total_pages | — | S1 |
| REQ-304 | D16 | §1.1 | API | Token JWT expirado → 401 com WWW-Authenticate: Bearer error="invalid_token" | — | S2 |
| REQ-305 | D20 | §4 | INFRA | UX de erro: toast para erros transitórios / página de erro para falhas críticas / retry automático para integrações | — | S1 |
| REQ-306 | D01.4 | RN-146 | RN | Taxa de cancelamento > 20% por estado → "Gargalo identificado" em vermelho | Limiar: 20% | S8 |
| REQ-307 | D01.3 | RN-109 | RN | Dashboard SLA defasagem máxima 5 minutos | — | S8 |
| REQ-308 | D01.4 | RN-144 | RN | Dashboard Executivo defasagem máxima 5 minutos | — | S8 |
| REQ-309 | D01.1 | §8.1 | RN | Estados do Contato: Ativo / Sem Caso Ativo / Arquivado | — | S5 |
| REQ-310 | D01.1 | §8.2 | RN | Estados da Atividade: Registrada / Agendada / Vencida / Concluída | — | S6 |
| REQ-311 | D01.2 | §12 | RN | Estados do Dossiê: Em montagem / Completo / Aprovado / Rejeitado (com pendências) | — | S5 |
| REQ-312 | D01.2 | §11 | RN | Estados da Comissão: Estimada / Pendente de Fechamento / Confirmada / Cancelada | — | S4 |

---

## Resumo por Sprint

| Sprint | Qtd REQs atribuídos | Status |
|--------|---------------------|--------|
| S1 — Fundação | REQ-108 a REQ-115, REQ-117 a REQ-135, REQ-139 a REQ-141, REQ-180, REQ-241 a REQ-242, REQ-245 a REQ-250, REQ-264 a REQ-271, REQ-277 a REQ-280, REQ-287, REQ-290 a REQ-305 | Pendente |
| S2 — Auth | REQ-001 a REQ-005, REQ-010 a REQ-012, REQ-037, REQ-107, REQ-142 a REQ-144, REQ-179, REQ-181 a REQ-185, REQ-239, REQ-275 a REQ-276, REQ-304 | Pendente |
| S3 — Pipeline de Casos | REQ-006 a REQ-009, REQ-016 a REQ-036, REQ-038 a REQ-043, REQ-146 a REQ-148, REQ-191 a REQ-199, REQ-253, REQ-273, REQ-281 | Pendente |
| S4 — Negociação e Comissão | REQ-044 a REQ-056, REQ-136, REQ-138, REQ-149 a REQ-153, REQ-215 a REQ-222, REQ-254, REQ-256, REQ-282 (parcial) | Pendente |
| S5 — Dossiê e Contatos | REQ-013 (parcial), REQ-014, REQ-057 a REQ-067, REQ-137, REQ-154 a REQ-160, REQ-200 a REQ-204, REQ-211 a REQ-214, REQ-259, REQ-282 (parcial), REQ-283 (parcial), REQ-301, REQ-309, REQ-311 | Pendente |
| S6 — Atividades | REQ-015, REQ-068 a REQ-072, REQ-161 a REQ-162, REQ-205 a REQ-207, REQ-260, REQ-283 (parcial), REQ-310 | Pendente |
| S7 — Comunicação e Integrações | REQ-073 a REQ-076, REQ-097 a REQ-106, REQ-163 a REQ-164, REQ-208 a REQ-210, REQ-231 a REQ-238, REQ-257, REQ-283 (parcial), REQ-284 (parcial), REQ-285 a REQ-286 | Pendente |
| S8 — SLA e Dashboard | REQ-077 a REQ-082, REQ-088 a REQ-090, REQ-109, REQ-145, REQ-165 a REQ-166, REQ-223 a REQ-224, REQ-243 a REQ-244, REQ-255, REQ-261 (parcial), REQ-274, REQ-292, REQ-306 a REQ-308 | Pendente |
| S9 — Equipe, Relatórios e Configurações | REQ-083 a REQ-087, REQ-091 a REQ-096, REQ-167 a REQ-179, REQ-186 a REQ-190, REQ-225 a REQ-230, REQ-258, REQ-279, REQ-284 (parcial) | Pendente |
| S10 — Qualidade | REQ-252, REQ-261, REQ-262, REQ-288 a REQ-289 | Pendente |
| SFinal — Auditoria | REQ-251, REQ-263, REQ-272 | Pendente |
